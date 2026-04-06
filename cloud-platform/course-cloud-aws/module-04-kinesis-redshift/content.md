# Module 4: Kinesis & Redshift — Streaming & Warehousing

> **Scenario**: RetailMart's batch pipelines (Modules 1–3) process data overnight, but the business demands **real-time** dashboards for store performance, fraud detection within seconds, and sub-minute inventory updates. Meanwhile, the analytics team needs a proper **data warehouse** for complex joins across billions of rows. Enter Kinesis for streaming and Redshift for warehousing.

---

## Screen 1: Kinesis Data Streams — Real-Time Ingestion

Kinesis Data Streams (KDS) is a **real-time data streaming service** that captures gigabytes per second from hundreds of thousands of sources — POS terminals, clickstream, IoT sensors, application logs.

### Core Concepts

```
┌──────────────────────────────────────────────────────────────────┐
│                     Kinesis Data Stream                          │
│                                                                  │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐           │
│  │ Shard 0 │  │ Shard 1 │  │ Shard 2 │  │ Shard 3 │           │
│  │         │  │         │  │         │  │         │           │
│  │ 1 MB/s  │  │ 1 MB/s  │  │ 1 MB/s  │  │ 1 MB/s  │  IN      │
│  │  in     │  │  in     │  │  in     │  │  in     │           │
│  │         │  │         │  │         │  │         │           │
│  │ 2 MB/s  │  │ 2 MB/s  │  │ 2 MB/s  │  │ 2 MB/s  │  OUT     │
│  │  out    │  │  out    │  │  out    │  │  out    │           │
│  │         │  │         │  │         │  │         │           │
│  │ 1,000   │  │ 1,000   │  │ 1,000   │  │ 1,000   │  Records │
│  │ rec/s   │  │ rec/s   │  │ rec/s   │  │ rec/s   │  /sec    │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘           │
│                                                                  │
│  Total Capacity: 4 MB/s in, 8 MB/s out, 4,000 records/sec      │
│  Retention: 24 hours (default) → up to 365 days                 │
│  Ordering: Guaranteed WITHIN a shard (by partition key)         │
└──────────────────────────────────────────────────────────────────┘
```

### Per-Shard Limits — Memorize These

| Metric | Limit |
|---|---|
| Write throughput | 1 MB/sec or 1,000 records/sec |
| Read throughput (shared) | 2 MB/sec per shard |
| Read throughput (enhanced fan-out) | 2 MB/sec per consumer per shard |
| Record size max | 1 MB |
| Retention default | 24 hours |
| Retention max | 365 days |
| Shards per stream (default) | 200 (soft limit, request increase) |

### RetailMart: POS Event Streaming Pipeline

```python
import boto3
import json
import hashlib
from datetime import datetime

kinesis = boto3.client("kinesis", region_name="us-east-1")

def send_pos_event(event: dict) -> dict:
    """Send a POS transaction to Kinesis Data Streams.
    
    Partition key = store_id ensures all events from the same 
    store land in the same shard → ordered processing per store.
    """
    partition_key = str(event["store_id"])
    
    response = kinesis.put_record(
        StreamName="retailmart-pos-events",
        Data=json.dumps(event).encode("utf-8"),
        PartitionKey=partition_key,
    )
    return response

# High-throughput: batch up to 500 records per call
def send_pos_batch(events: list[dict]) -> dict:
    """Batch put for high-throughput ingestion."""
    records = [
        {
            "Data": json.dumps(event).encode("utf-8"),
            "PartitionKey": str(event["store_id"]),
        }
        for event in events
    ]
    
    response = kinesis.put_records(
        StreamName="retailmart-pos-events",
        Records=records,  # max 500 records or 5 MB per call
    )
    
    # Handle partial failures (critical!)
    if response["FailedRecordCount"] > 0:
        failed = [
            (i, r) for i, r in enumerate(response["Records"])
            if "ErrorCode" in r
        ]
        print(f"Failed records: {len(failed)} — retrying...")
        # Implement exponential backoff retry for failed records
    
    return response

# Example event from a POS terminal
event = {
    "transaction_id": "txn-2026-04-05-0042-99381",
    "store_id": 42,
    "register_id": 7,
    "items": [
        {"sku": "ELEC-TV-55-SAM", "qty": 1, "price": 499.99},
        {"sku": "ACC-HDMI-CBL", "qty": 2, "price": 12.99},
    ],
    "total_amount": 525.97,
    "payment_method": "credit_card",
    "timestamp": datetime.utcnow().isoformat(),
}

send_pos_event(event)
```

### Partition Key Strategy

The partition key determines which shard a record lands in. A **hot partition key** causes one shard to receive disproportionate traffic.

```
❌ BAD — Using a constant partition key:
   PartitionKey = "all-stores"
   → ALL records go to 1 shard → 1 MB/s bottleneck

❌ MEDIOCRE — Using store_id with skewed distribution:
   Store #1 (flagship) = 40% of all events
   → Shard 0 overloaded, shards 1-3 idle

✅ GOOD — Composite key for even distribution:
   PartitionKey = f"{store_id}-{register_id}"
   → 4,000 stores × 15 registers = 60,000 unique keys
   → Evenly distributed across shards

✅ BEST — When order doesn't matter, use random:
   PartitionKey = str(uuid4())
   → Perfect distribution (but no per-key ordering)
```

### 💡 Interview Insight

> **Q: "How do you determine the number of shards for a Kinesis stream?"**
>
> Calculate from throughput requirements: **Shards = max(incoming_MB/s ÷ 1, outgoing_MB/s ÷ 2)**. RetailMart's 4,000 stores generate ~8 MB/s of POS events at peak → 8 shards minimum for ingestion. If 3 consumers each read at 2 MB/s, you need 8 MB/s × 3 ÷ 2 = 12 shards for fan-out (or use Enhanced Fan-Out to decouple consumer throughput). Always add 20% headroom for spikes. Use **On-Demand mode** if traffic is unpredictable — it auto-scales up to 200 MB/s.

---

## Screen 2: Kinesis Data Firehose — Managed Delivery

Firehose is a **fully managed delivery stream** that captures, transforms, and loads data into destinations — S3, Redshift, OpenSearch, Splunk, HTTP endpoints. Zero administration, auto-scaling, exactly what you want for the "last mile."

### Firehose vs. Data Streams

| Feature | Kinesis Data Streams | Kinesis Data Firehose |
|---|---|---|
| Provisioning | You manage shards | Fully managed, auto-scales |
| Latency | ~200ms (real-time) | 60-900 sec buffer (near-real-time) |
| Consumers | You build (KCL, Lambda) | Built-in destinations |
| Data retention | 24h–365d | None (pass-through) |
| Transformations | Consumer-side | Built-in Lambda transform |
| Replay | Yes (re-read from shard) | No (one-shot delivery) |
| Pricing | Per shard-hour | Per GB ingested |
| Use case | Real-time processing | Loading into data stores |

### RetailMart Firehose: POS Events → S3 (Parquet)

```
POS Terminals ──► Kinesis Data Stream ──► Firehose ──► S3 (Parquet)
                                            │
                                            ├──► Lambda (enrich with store metadata)
                                            ├──► Convert to Parquet (built-in)
                                            ├──► Buffer: 128 MB or 300 seconds
                                            └──► Partition: year/month/day/hour
```

```bash
# Create a Firehose delivery stream: KDS → S3 (Parquet)
aws firehose create-delivery-stream \
  --delivery-stream-name retailmart-pos-to-s3 \
  --delivery-stream-type KinesisStreamAsSource \
  --kinesis-stream-source-configuration '{
    "KinesisStreamARN": "arn:aws:kinesis:us-east-1:123456789:stream/retailmart-pos-events",
    "RoleARN": "arn:aws:iam::123456789:role/FirehoseKinesisRole"
  }' \
  --extended-s3-destination-configuration '{
    "BucketARN": "arn:aws:s3:::retailmart-data-lake-prod",
    "Prefix": "streaming/pos-events/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/hour=!{timestamp:HH}/",
    "ErrorOutputPrefix": "streaming/errors/pos-events/!{firehose:error-output-type}/year=!{timestamp:yyyy}/",
    "RoleARN": "arn:aws:iam::123456789:role/FirehoseS3Role",
    "BufferingHints": {
      "SizeInMBs": 128,
      "IntervalInSeconds": 300
    },
    "CompressionType": "UNCOMPRESSED",
    "DataFormatConversionConfiguration": {
      "Enabled": true,
      "InputFormatConfiguration": {
        "Deserializer": {
          "OpenXJsonSerDe": {}
        }
      },
      "OutputFormatConfiguration": {
        "Serializer": {
          "ParquetSerDe": {
            "Compression": "SNAPPY"
          }
        }
      },
      "SchemaConfiguration": {
        "DatabaseName": "streaming_zone",
        "TableName": "pos_events",
        "RoleARN": "arn:aws:iam::123456789:role/FirehoseGlueRole",
        "Region": "us-east-1",
        "CatalogId": "123456789012"
      }
    }
  }'
```

### Dynamic Partitioning with Firehose

Firehose can partition output based on record content (not just timestamp), enabling Hive-compatible partition layouts:

```
Record: {"store_id": 42, "region": "northeast", ...}

Output path:
  s3://retailmart-data-lake-prod/streaming/pos-events/
    region=northeast/store_id=42/year=2026/month=04/day=05/hour=14/
    firehose-batch-00001.parquet
```

### 💡 Interview Insight

> **Q: "When do you use Firehose alone vs. Data Streams + Firehose?"**
>
> **Firehose alone** (Direct PUT): When you just need to load data into S3/Redshift/OpenSearch with minimal processing. Producers call `PutRecord` directly to Firehose. Simple, cheap, no shard management.
>
> **Data Streams + Firehose**: When you need (1) **multiple consumers** — KDS supports many readers; Firehose is one of them. (2) **Real-time processing** alongside loading — a Lambda consumer does fraud detection in <200ms while Firehose buffers to S3. (3) **Replay** — KDS retains data for re-read; Firehose doesn't. At RetailMart, we use KDS as the "spine" with three consumers: Firehose (→ S3), Lambda (→ fraud alerts), and a Flink app (→ real-time dashboards).

---

## Screen 3: Kinesis vs. Kafka (Amazon MSK)

This is one of the most common interview questions for data engineering roles. Know the trade-offs cold.

### Head-to-Head Comparison

| Dimension | Kinesis Data Streams | Amazon MSK (Kafka) |
|---|---|---|
| **Model** | Shards (AWS-managed) | Partitions (you manage brokers) |
| **Throughput per unit** | 1 MB/s in, 2 MB/s out per shard | No per-partition limit (broker-level) |
| **Ordering** | Per-shard (partition key) | Per-partition (message key) |
| **Retention** | 24h–365d | Unlimited (disk-based) |
| **Replay** | Yes (shard iterator) | Yes (consumer offset reset) |
| **Protocols** | AWS SDK, KPL, KCL | Kafka protocol (open standard) |
| **Ecosystem** | AWS-native | Kafka Connect, KSQL, Schema Registry |
| **Consumer groups** | KCL applications | Native consumer groups |
| **Scaling** | Shard split/merge (minutes) | Add partitions (fast, no rebalance) |
| **Operations** | Zero ops | Broker patching, disk, ZooKeeper/KRaft |
| **Cost model** | Per shard-hour + per GB | Per broker-hour + storage + throughput |
| **Cross-region** | Manual (replicate with Lambda) | MirrorMaker 2 (built-in) |
| **Best for** | AWS-native, small-medium streams | High-throughput, multi-cloud, open-source |

### When to Choose Kinesis

```
✅ Kinesis when:
  • Team is small, no Kafka expertise
  • Throughput < 200 MB/s
  • Tight AWS integration needed (Firehose, Lambda, Analytics)
  • You want zero operational overhead
  • Budget predictability matters (per-shard pricing)

✅ MSK (Kafka) when:
  • Existing Kafka ecosystem / expertise on team
  • Throughput > 200 MB/s or need for massive fan-out
  • Multi-cloud or hybrid strategy
  • Need Kafka Connect for 100+ pre-built connectors
  • Need KSQL / Kafka Streams for stream processing
  • Log compaction required (keep latest per key)
```

### Architecture Decision: RetailMart Chose Kinesis

```
Decision factors for RetailMart:
  • 4,000 stores × ~2 KB/event × 50 events/sec peak = ~400 MB/s
  • Wait — that's > 200 MB/s. Why Kinesis?
  
  Answer: On-Demand mode scales to 200 MB/s per stream.
  We split into 3 streams:
    • retailmart-pos-events       (~150 MB/s peak)
    • retailmart-clickstream      (~200 MB/s peak)
    • retailmart-inventory-updates (~50 MB/s peak)
  
  Total cost with On-Demand: ~$3,200/month
  MSK equivalent (m5.2xlarge × 6 brokers): ~$4,800/month + ops
  
  Decision: Kinesis — 33% cheaper, zero ops, 3-person team.
```

### 💡 Interview Insight

> **Q: "A company is migrating from on-prem Kafka to AWS. Should they use Kinesis or MSK?"**
>
> **MSK** in most cases. If they have existing Kafka producers/consumers, Kafka Connect pipelines, and team expertise — MSK is a drop-in managed replacement. The migration is straightforward: MirrorMaker 2 replicates topics from on-prem to MSK, then cut over producers. Kinesis would require rewriting every producer/consumer to use the AWS SDK. The only exception: if they want to eliminate Kafka entirely and go "all-in AWS serverless" with Lambda consumers and Firehose delivery — then Kinesis makes sense as a clean break.

---

## Screen 4: Enhanced Fan-Out & Kinesis Data Analytics

### Enhanced Fan-Out (EFO)

By default, all consumers of a shard **share** the 2 MB/s read throughput. With Enhanced Fan-Out, each registered consumer gets a **dedicated** 2 MB/s pipe via HTTP/2 push (SubscribeToShard).

```
Without Enhanced Fan-Out (shared):
┌──────────┐
│  Shard 0 │──── 2 MB/s total ────┬── Consumer A (gets ~0.67 MB/s)
│          │                      ├── Consumer B (gets ~0.67 MB/s)
│          │                      └── Consumer C (gets ~0.67 MB/s)
└──────────┘

With Enhanced Fan-Out (dedicated):
┌──────────┐
│  Shard 0 │──── 2 MB/s ──► Consumer A (dedicated pipe, HTTP/2 push)
│          │──── 2 MB/s ──► Consumer B (dedicated pipe, HTTP/2 push)
│          │──── 2 MB/s ──► Consumer C (dedicated pipe, HTTP/2 push)
└──────────┘
  Total out: 6 MB/s (3 consumers × 2 MB/s each)
```

### When to Use Enhanced Fan-Out

| Scenario | Standard | Enhanced Fan-Out |
|---|---|---|
| 1-2 consumers | ✅ Sufficient | Overkill (cost) |
| 3+ consumers | ⚠️ Throughput split | ✅ Dedicated throughput |
| Latency requirement < 200ms | ⚠️ Polling adds latency | ✅ Push-based, ~70ms |
| Cost sensitivity | ✅ Included in shard price | $$$ per consumer-shard-hour |

```bash
# Register a consumer for Enhanced Fan-Out
aws kinesis register-stream-consumer \
  --stream-arn "arn:aws:kinesis:us-east-1:123456789:stream/retailmart-pos-events" \
  --consumer-name "fraud-detection-consumer"
```

### Kinesis Data Analytics (Managed Apache Flink)

Kinesis Data Analytics lets you run **Apache Flink** applications on streaming data — SQL or Java/Python — fully managed.

```
┌──────────────┐    ┌──────────────────────┐    ┌──────────────┐
│ Kinesis Data │    │  Kinesis Data         │    │ Destinations: │
│ Stream       │───►│  Analytics (Flink)    │───►│ • Kinesis     │
│              │    │                      │    │ • Firehose    │
│ OR           │    │  • SQL queries        │    │ • Lambda      │
│ MSK (Kafka)  │    │  • Windowed aggs     │    │ • Custom sink │
│              │    │  • Pattern detection  │    │               │
└──────────────┘    └──────────────────────┘    └──────────────┘
```

### RetailMart: Real-Time Fraud Detection with Flink SQL

```sql
-- Detect potential fraud: >3 transactions at different stores within 5 minutes
CREATE TABLE pos_events (
    transaction_id STRING,
    store_id       INT,
    customer_id    STRING,
    total_amount   DECIMAL(10,2),
    event_time     TIMESTAMP(3),
    WATERMARK FOR event_time AS event_time - INTERVAL '10' SECOND
) WITH (
    'connector' = 'kinesis',
    'stream'    = 'retailmart-pos-events',
    'aws.region' = 'us-east-1',
    'format'    = 'json',
    'scan.stream.initpos' = 'LATEST'
);

CREATE TABLE fraud_alerts (
    customer_id     STRING,
    distinct_stores INT,
    total_spent     DECIMAL(10,2),
    window_start    TIMESTAMP(3),
    window_end      TIMESTAMP(3)
) WITH (
    'connector' = 'kinesis',
    'stream'    = 'retailmart-fraud-alerts',
    'aws.region' = 'us-east-1',
    'format'    = 'json'
);

-- Tumbling window: flag customers with purchases at 3+ stores in 5 minutes
INSERT INTO fraud_alerts
SELECT
    customer_id,
    COUNT(DISTINCT store_id)   AS distinct_stores,
    SUM(total_amount)          AS total_spent,
    TUMBLE_START(event_time, INTERVAL '5' MINUTE) AS window_start,
    TUMBLE_END(event_time, INTERVAL '5' MINUTE)   AS window_end
FROM pos_events
GROUP BY
    customer_id,
    TUMBLE(event_time, INTERVAL '5' MINUTE)
HAVING COUNT(DISTINCT store_id) >= 3;
```

### Flink Window Types

```
TUMBLING WINDOW (fixed, non-overlapping):
  |---- 5 min ----|---- 5 min ----|---- 5 min ----|
  |  events here  |  events here  |  events here  |

SLIDING WINDOW (overlapping):
  |---- 5 min ----|
       |---- 5 min ----|
            |---- 5 min ----|
  Slide every 1 minute → 5 overlapping windows active

SESSION WINDOW (gap-based):
  |--events--| 10min gap |--events--events--| 10min gap |--events--|
  |  session 1 |          |    session 2      |          | session 3|
```

### 💡 Interview Insight

> **Q: "How does Kinesis Data Analytics handle late-arriving events?"**
>
> Flink uses **watermarks** — a timestamp threshold below which all data is considered "arrived." The `WATERMARK FOR event_time AS event_time - INTERVAL '10' SECOND` means events up to 10 seconds late are still included in the correct window. Events arriving after the watermark passes are dropped by default, or you can configure **allowed lateness** to emit updated results. For RetailMart's fraud detection, we set a 30-second watermark — POS events rarely arrive later than that. For clickstream (which buffers client-side), we use a 2-minute watermark.

---

## Screen 5: Redshift Architecture — Columnar MPP Warehouse

Amazon Redshift is a **petabyte-scale, columnar, massively parallel processing (MPP)** data warehouse. It's purpose-built for OLAP queries that scan billions of rows.

### Architecture Deep Dive

```
┌──────────────────────────────────────────────────────────────────┐
│                     Redshift Cluster                             │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    Leader Node                            │   │
│  │  • Parses SQL, builds query plan                         │   │
│  │  • Distributes work to compute nodes                     │   │
│  │  • Aggregates results from compute nodes                 │   │
│  │  • Manages metadata, catalog                             │   │
│  └────────────────┬─────────────────────────────────────────┘   │
│                   │                                              │
│       ┌───────────┼───────────┐                                  │
│       ▼           ▼           ▼                                  │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐                           │
│  │Compute  │ │Compute  │ │Compute  │     Compute Nodes          │
│  │Node 0   │ │Node 1   │ │Node 2   │     • Execute query        │
│  │         │ │         │ │         │       fragments in parallel │
│  │ Slice 0 │ │ Slice 0 │ │ Slice 0 │     • Each node has        │
│  │ Slice 1 │ │ Slice 1 │ │ Slice 1 │       multiple slices      │
│  │         │ │         │ │         │     • Each slice = 1 CPU    │
│  │ Local   │ │ Local   │ │ Local   │       + memory + disk      │
│  │ Storage │ │ Storage │ │ Storage │                             │
│  └─────────┘ └─────────┘ └─────────┘                           │
│                                                                  │
│  Managed Storage (RA3):                                         │
│  Hot data cached on local SSD, warm data in S3 (RMS)            │
└──────────────────────────────────────────────────────────────────┘
```

### Columnar Storage — Why It's Fast

```
Row-oriented (PostgreSQL, MySQL):
  Row 1: [txn_id=1, store=42, amount=99.99, method=credit, ts=2026-04-05]
  Row 2: [txn_id=2, store=17, amount=24.50, method=debit,  ts=2026-04-05]
  Row 3: [txn_id=3, store=42, amount=150.00, method=cash,  ts=2026-04-05]
  
  SELECT SUM(amount) → Must read ALL columns of ALL rows

Columnar (Redshift):
  Column "txn_id":  [1, 2, 3, ...]          ← Skip
  Column "store":   [42, 17, 42, ...]        ← Skip
  Column "amount":  [99.99, 24.50, 150.00]   ← READ THIS
  Column "method":  [credit, debit, cash]    ← Skip
  Column "ts":      [2026-04-05, ...]        ← Skip
  
  SELECT SUM(amount) → Reads ONLY the amount column (1/5 of data)
```

**Additional benefits:**
- **Compression**: Same-type values in a column compress 5-10x better than mixed-type rows
- **Zone maps**: Min/max metadata per 1 MB block → skip blocks that can't match the filter
- **Vectorized execution**: Process columns as arrays, leverage CPU SIMD instructions

### Distribution Styles — How Data is Spread Across Nodes

| Style | Behavior | Best For | Avoid When |
|---|---|---|---|
| **KEY** | Rows with same key value → same node | Large fact tables joined on that key | Key is heavily skewed |
| **EVEN** | Round-robin across all nodes | Tables not joined, or no clear key | Frequent joins (causes redistribution) |
| **ALL** | Full copy on every node | Small dimension tables (< 5M rows) | Large tables (wastes storage + slow loads) |
| **AUTO** | Redshift decides (starts EVEN, may change) | Default — let Redshift optimize | When you know the join pattern well |

### RetailMart Redshift Schema Design

```sql
-- Fact table: KEY distribution on store_id (most joins filter by store)
CREATE TABLE warehouse.fact_daily_sales (
    transaction_id  VARCHAR(50)    ENCODE zstd,
    store_id        INTEGER        ENCODE az64       DISTKEY,
    product_sku     VARCHAR(30)    ENCODE zstd,
    quantity        SMALLINT       ENCODE az64,
    total_amount    DECIMAL(10,2)  ENCODE az64,
    payment_method  VARCHAR(20)    ENCODE bytedict,
    transaction_ts  TIMESTAMP      ENCODE az64       SORTKEY
)
DISTSTYLE KEY
COMPOUND SORTKEY (transaction_ts, store_id);

-- Dimension table: ALL distribution (small, joined frequently)
CREATE TABLE warehouse.dim_store (
    store_id        INTEGER        PRIMARY KEY   ENCODE az64,
    store_name      VARCHAR(100)   ENCODE zstd,
    region          VARCHAR(30)    ENCODE bytedict,
    state           VARCHAR(2)     ENCODE bytedict,
    city            VARCHAR(50)    ENCODE zstd,
    format          VARCHAR(20)    ENCODE bytedict,
    open_date       DATE           ENCODE az64,
    sq_footage      INTEGER        ENCODE az64
)
DISTSTYLE ALL;

-- Dimension table: ALL distribution (product catalog)
CREATE TABLE warehouse.dim_product (
    product_sku     VARCHAR(30)    PRIMARY KEY   ENCODE zstd,
    product_name    VARCHAR(200)   ENCODE zstd,
    category        VARCHAR(50)    ENCODE bytedict,
    subcategory     VARCHAR(50)    ENCODE bytedict,
    brand           VARCHAR(50)    ENCODE zstd,
    unit_cost       DECIMAL(10,2)  ENCODE az64
)
DISTSTYLE ALL;
```

### Sort Keys — Zone Map Efficiency

```
COMPOUND SORTKEY (transaction_ts, store_id):

  Block 1:  ts = 2026-04-01 to 2026-04-01, store = 1-500
  Block 2:  ts = 2026-04-01 to 2026-04-01, store = 501-1000
  Block 3:  ts = 2026-04-02 to 2026-04-02, store = 1-500
  ...

  WHERE transaction_ts = '2026-04-05'  → Skips blocks 1-N (zone maps)
  WHERE store_id = 42                   → Only effective after ts filter

INTERLEAVED SORTKEY (transaction_ts, store_id):
  
  Both columns have equal weight in zone maps.
  Better for ad-hoc queries filtering on either column independently.
  BUT: VACUUM REINDEX is expensive (avoid for > 100M rows).
```

### 💡 Interview Insight

> **Q: "How do you choose between KEY, EVEN, and ALL distribution?"**
>
> Decision tree: (1) Is the table small (< 5M rows) and joined frequently? → **ALL** (replicated to every node, joins are local). (2) Is the table large and joined on a specific column? → **KEY** on that join column (co-locates matching rows). (3) Is the table large with no clear join pattern? → **EVEN** (balanced storage). Common mistake: using KEY distribution on a skewed column (e.g., `customer_id` where 1% of customers generate 50% of transactions) — one node drowns while others idle. Check distribution with `SELECT "table", skew_rows FROM svv_table_info`.

---

## Screen 6: Redshift Spectrum & Redshift Serverless

### Redshift Spectrum — Query S3 Without Loading

Spectrum extends Redshift SQL to query data **directly in S3** using the Glue Data Catalog. No data loading, no ETL — just point and query.

```
┌────────────────────────────────────────────────────────────────┐
│                        Redshift Cluster                        │
│                                                                │
│  SELECT s.store_id, d.region, SUM(s.total_amount)             │
│  FROM warehouse.fact_daily_sales s        ← Local (Redshift)  │
│  JOIN spectrum_schema.historical_sales h  ← S3 (Spectrum)     │
│    ON s.store_id = h.store_id                                  │
│  JOIN warehouse.dim_store d               ← Local (Redshift)  │
│    ON s.store_id = d.store_id                                  │
│  GROUP BY s.store_id, d.region;                                │
│                                                                │
│  ┌───────────────────┐    ┌─────────────────────────────────┐ │
│  │ Redshift Compute  │    │ Spectrum Layer (shared pool)     │ │
│  │ Nodes             │    │ Thousands of nodes scan S3       │ │
│  │ (local tables)    │◄──►│ Filter/project/aggregate         │ │
│  └───────────────────┘    │ Return only matching rows        │ │
│                           └─────────────────────────────────┘ │
│                                    │                           │
│                                    ▼                           │
│                           ┌─────────────────┐                 │
│                           │  S3 Data Lake    │                 │
│                           │  (Parquet/ORC)   │                 │
│                           │  Glue Catalog    │                 │
│                           └─────────────────┘                 │
└────────────────────────────────────────────────────────────────┘
```

```sql
-- Create an external schema pointing to Glue Data Catalog
CREATE EXTERNAL SCHEMA spectrum_schema
FROM DATA CATALOG
DATABASE 'curated'
IAM_ROLE 'arn:aws:iam::123456789:role/RedshiftSpectrumRole'
CREATE EXTERNAL DATABASE IF NOT EXISTS;

-- Query S3 data as if it were a local table
SELECT
    region,
    COUNT(*) AS txn_count,
    SUM(total_amount) AS revenue
FROM spectrum_schema.daily_sales
WHERE year = '2025'  -- Partition pruning on S3
GROUP BY region
ORDER BY revenue DESC;

-- Join local Redshift tables with S3 Spectrum tables
SELECT
    d.region,
    d.store_name,
    SUM(CASE WHEN s.year = '2026' THEN s.total_amount END) AS revenue_2026,
    SUM(CASE WHEN h.year = '2025' THEN h.total_amount END) AS revenue_2025,
    ROUND(
        (SUM(CASE WHEN s.year = '2026' THEN s.total_amount END) /
         NULLIF(SUM(CASE WHEN h.year = '2025' THEN h.total_amount END), 0) - 1) * 100,
        2
    ) AS yoy_growth_pct
FROM warehouse.fact_daily_sales s
FULL OUTER JOIN spectrum_schema.daily_sales h
    ON s.store_id = h.store_id AND s.product_sku = h.product_sku
JOIN warehouse.dim_store d ON COALESCE(s.store_id, h.store_id) = d.store_id
GROUP BY d.region, d.store_name
ORDER BY yoy_growth_pct DESC;
```

### Redshift Serverless

No clusters to manage. Define a **workgroup** and a **namespace**, and Redshift auto-provisions and scales compute measured in **RPUs** (Redshift Processing Units).

```
┌────────────────────────────────────────────────┐
│           Redshift Serverless                   │
│                                                 │
│  Namespace: retailmart-analytics                │
│    • Database: warehouse                        │
│    • Schemas, tables, users                     │
│    • Encryption (KMS)                           │
│                                                 │
│  Workgroup: analyst-workgroup                   │
│    • Base capacity: 32 RPU                      │
│    • Max capacity: 512 RPU                      │
│    • Auto-scales based on query complexity      │
│    • Cost: per RPU-second (billed by the second)│
│    • Idle: scales to 0 (no charge)              │
│                                                 │
│  Workgroup: etl-workgroup                       │
│    • Base capacity: 128 RPU                     │
│    • For nightly batch loads                    │
│    • Usage limits: max 2 hours/day              │
│                                                 │
└────────────────────────────────────────────────┘
```

```bash
# Create a Redshift Serverless namespace
aws redshift-serverless create-namespace \
  --namespace-name retailmart-analytics \
  --admin-username admin \
  --admin-user-password "${REDSHIFT_ADMIN_PASS}" \
  --db-name warehouse \
  --kms-key-id "arn:aws:kms:us-east-1:123456789:key/mrk-abcdef"

# Create a workgroup with auto-scaling
aws redshift-serverless create-workgroup \
  --workgroup-name analyst-workgroup \
  --namespace-name retailmart-analytics \
  --base-capacity 32 \
  --max-capacity 512 \
  --config-parameters '[
    {"parameterKey": "max_query_execution_time", "parameterValue": "3600"},
    {"parameterKey": "enable_result_cache_for_session", "parameterValue": "true"}
  ]'
```

### Redshift Provisioned vs. Serverless

| Feature | Provisioned | Serverless |
|---|---|---|
| Pricing | Per-node-hour (always on) | Per RPU-second (pay for use) |
| Scaling | Resize cluster (minutes) | Auto-scale (seconds) |
| Idle cost | Full cluster cost | $0 (scales to zero) |
| Concurrency | Concurrency Scaling (extra cost) | Auto-managed |
| Best for | Steady, predictable workloads | Variable, bursty workloads |
| Reserved instances | Yes (up to 75% savings) | No |
| Node types | RA3.xlplus, RA3.4xl, RA3.16xl | N/A (RPUs) |

### 💡 Interview Insight

> **Q: "When would you use Redshift Spectrum vs. just using Athena?"**
>
> Use Spectrum when: (1) You already have a Redshift cluster and need to join local warehouse tables with S3 data. (2) You need Redshift-specific features (materialized views, stored procedures) applied to S3 data. (3) Your query involves both hot (Redshift) and cold (S3) data in one join.
>
> Use Athena when: (1) You don't have/need a Redshift cluster. (2) Queries are purely against S3. (3) You want serverless, per-query pricing without a running cluster. Spectrum requires a running Redshift cluster, so there's always a baseline cost.

---

## Screen 7: Data Sharing & Cross-Account Patterns

### Redshift Data Sharing

Share live, up-to-date data across Redshift clusters/serverless workgroups — even cross-account — **without copying data**.

```
┌───────────────────────────┐     Data Share     ┌───────────────────────────┐
│   Producer Cluster        │ ──────────────────► │   Consumer Cluster        │
│   (retailmart-central)    │   No data movement  │   (retailmart-analytics)  │
│                           │   Real-time access   │                           │
│   Namespace: prod-ns      │                     │   Account: 987654321      │
│   Database: warehouse     │                     │   Region: us-east-1       │
│                           │                     │   (or cross-region!)      │
│   Shared objects:         │                     │   Sees:                   │
│   • fact_daily_sales      │                     │   • Read-only access      │
│   • dim_store             │                     │   • Own compute resources │
│   • dim_product           │                     │   • No storage cost       │
└───────────────────────────┘                     └───────────────────────────┘
```

```sql
-- On the PRODUCER cluster:
CREATE DATASHARE retailmart_share MANAGEDBY NAMESPACE 'prod-ns-id';

ALTER DATASHARE retailmart_share ADD SCHEMA warehouse;
ALTER DATASHARE retailmart_share ADD TABLE warehouse.fact_daily_sales;
ALTER DATASHARE retailmart_share ADD TABLE warehouse.dim_store;
ALTER DATASHARE retailmart_share ADD TABLE warehouse.dim_product;

-- Grant to consumer namespace (same account)
GRANT USAGE ON DATASHARE retailmart_share
TO NAMESPACE 'consumer-ns-id';

-- Grant to consumer account (cross-account)
GRANT USAGE ON DATASHARE retailmart_share
TO ACCOUNT '987654321012';

-- On the CONSUMER cluster:
CREATE DATABASE shared_warehouse
FROM DATASHARE retailmart_share
OF NAMESPACE 'prod-ns-id';

-- Query as if local (but compute is consumer-side)
SELECT region, SUM(total_amount)
FROM shared_warehouse.warehouse.fact_daily_sales f
JOIN shared_warehouse.warehouse.dim_store s ON f.store_id = s.store_id
GROUP BY region;
```

### 💡 Interview Insight

> **Q: "How is Redshift data sharing different from cross-account S3 access?"**
>
> Key differences: (1) **No data movement** — data stays in the producer's managed storage. S3 cross-account requires bucket policies and often data duplication. (2) **Real-time** — consumers see the latest data the instant it's committed. S3-based sharing has pipeline lag. (3) **Compute isolation** — consumers use their own RPUs/cluster, so producer performance isn't affected. (4) **Governance** — producer controls exactly which schemas/tables/views are shared. (5) **Cost** — consumer pays only for compute, not storage. It's conceptually similar to BigQuery's dataset sharing.

---

## Screen 8: Redshift vs. Athena vs. BigQuery

This three-way comparison comes up frequently when interviewers test cloud-agnostic thinking.

### Comprehensive Comparison

| Dimension | Redshift | Athena | BigQuery |
|---|---|---|---|
| **Type** | MPP warehouse | Serverless query engine | Serverless warehouse |
| **Engine** | Custom PostgreSQL fork | Trino (Presto) | Dremel/Capacitor |
| **Storage** | Managed (RA3) or local | S3 (external) | BigQuery Storage |
| **Compute model** | Cluster or RPUs | Per-query | Slots (auto or reserved) |
| **Pricing** | Node-hours or RPU-seconds | $5/TB scanned | $6.25/TB scanned (on-demand) |
| **Idle cost** | Cluster: yes. Serverless: no | $0 | $0 (on-demand) |
| **Concurrency** | High (with Concurrency Scaling) | Very high (serverless) | Very high (serverless) |
| **Latency** | Sub-second (cached) | 2-30 seconds | 1-30 seconds |
| **Joins** | Excellent (local + Spectrum) | Good (federation) | Excellent (native) |
| **ACID** | Yes | Via Iceberg | Yes (native) |
| **ML integration** | Redshift ML (SageMaker) | N/A (use SageMaker) | BigQuery ML (native) |
| **Streaming ingest** | Streaming ingestion | N/A (query at rest) | BigQuery Storage Write API |
| **Semi-structured** | SUPER type (JSON) | Native JSON support | STRUCT, ARRAY, JSON |
| **Best for** | Known workloads, complex DW | Ad-hoc exploration, S3 | All-purpose, serverless DW |

### Decision Framework

```
┌─────────────────────────────────────────────────────┐
│         "Where should I run this query?"             │
├─────────────────────────────────────────────────────┤
│                                                      │
│  Is it ad-hoc exploration of S3 data?                │
│    YES → Athena ($5/TB, no infra)                    │
│                                                      │
│  Is it a complex, recurring dashboard query           │
│  joining multiple large tables?                       │
│    YES → Redshift (co-located data, sub-second)      │
│                                                      │
│  Is it a Google Cloud / multi-cloud environment?      │
│    YES → BigQuery (serverless, no ops)               │
│                                                      │
│  Do you need sub-second latency for BI dashboards?   │
│    YES → Redshift (materialized views, result cache)  │
│                                                      │
│  Is usage unpredictable with long idle periods?       │
│    YES → Athena or Redshift Serverless                │
│                                                      │
│  Do you need ML on warehouse data?                   │
│    → Redshift ML or BigQuery ML                      │
│                                                      │
└─────────────────────────────────────────────────────┘
```

### 💡 Interview Insight

> **Q: "RetailMart is evaluating Redshift vs. Athena for their analytics layer. How do you decide?"**
>
> It's not either/or — they complement each other. Use **Athena** for: data exploration, ad-hoc queries by analysts, data quality checks, one-off reports, querying raw data in S3. Use **Redshift** for: curated data warehouse with complex star-schema joins, BI dashboard backends (Tableau, QuickSight), materialized views for pre-aggregated metrics, workloads that need sub-second response. At RetailMart, we use Athena for exploration (raw/curated zones) and Redshift for the production analytics warehouse that powers executive dashboards. Spectrum bridges both — Redshift queries can reach into S3 for historical data.

---

## Screen 9: Streaming-to-Warehouse End-to-End Architecture

### RetailMart Complete Real-Time Pipeline

```
┌─────────┐    ┌──────────────┐    ┌──────────────────────────────────┐
│ 4,000   │    │ Kinesis Data  │    │          Consumers                │
│ POS     │───►│ Streams       │───►│                                  │
│ Stores  │    │ (8 shards)    │    │  ┌────────────────────────────┐  │
└─────────┘    └──────────────┘    │  │ Consumer 1: Firehose       │  │
                                    │  │ → S3 Parquet (5-min buffer)│  │
┌─────────┐    ┌──────────────┐    │  └────────────────────────────┘  │
│ Web /   │    │ Kinesis Data  │    │                                  │
│ Mobile  │───►│ Streams       │    │  ┌────────────────────────────┐  │
│ Click   │    │ (12 shards)   │    │  │ Consumer 2: Flink          │  │
└─────────┘    └──────────────┘    │  │ → Real-time dashboards     │  │
                                    │  │ → Fraud detection          │  │
┌─────────┐    ┌──────────────┐    │  └────────────────────────────┘  │
│ IoT     │    │ Kinesis Data  │    │                                  │
│ Sensors │───►│ Streams       │    │  ┌────────────────────────────┐  │
│ (HVAC)  │    │ (4 shards)    │    │  │ Consumer 3: Lambda         │  │
└─────────┘    └──────────────┘    │  │ → Anomaly alerts (SNS)     │  │
                                    │  └────────────────────────────┘  │
                                    └──────────────────────────────────┘
                                                │
                                                ▼
                              ┌──────────────────────────────┐
                              │  S3 Data Lake (Parquet)       │
                              │  streaming/pos-events/        │
                              │  streaming/clickstream/       │
                              │  streaming/iot-telemetry/     │
                              └─────────────┬────────────────┘
                                            │
                                    ┌───────┼────────┐
                                    ▼       ▼        ▼
                              ┌──────┐ ┌──────┐ ┌──────────┐
                              │Athena│ │Glue  │ │Redshift  │
                              │(ad   │ │ETL   │ │(load via │
                              │ hoc) │ │(join, │ │ COPY or  │
                              │      │ │ agg)  │ │Streaming │
                              │      │ │       │ │Ingestion)│
                              └──────┘ └──────┘ └──────────┘
```

### Loading S3 Data into Redshift via COPY

```sql
-- COPY is the fastest way to load data into Redshift
COPY warehouse.fact_daily_sales
FROM 's3://retailmart-data-lake-prod/curated/daily_sales/year=2026/month=04/'
IAM_ROLE 'arn:aws:iam::123456789:role/RedshiftCopyRole'
FORMAT AS PARQUET
COMPUPDATE ON
STATUPDATE ON;

-- For streaming: Redshift Streaming Ingestion (direct from Kinesis)
CREATE EXTERNAL SCHEMA kinesis_schema
FROM KINESIS
IAM_ROLE 'arn:aws:iam::123456789:role/RedshiftStreamRole';

-- Create a materialized view that auto-refreshes from the stream
CREATE MATERIALIZED VIEW warehouse.mv_realtime_sales AS
SELECT
    approximate_arrival_timestamp AS event_time,
    JSON_PARSE(kinesis_data) AS payload,
    JSON_EXTRACT_PATH_TEXT(kinesis_data, 'store_id')::INT AS store_id,
    JSON_EXTRACT_PATH_TEXT(kinesis_data, 'total_amount')::DECIMAL(10,2) AS amount
FROM kinesis_schema."retailmart-pos-events"
WHERE is_valid_json(kinesis_data);

-- Refresh to pull latest records
REFRESH MATERIALIZED VIEW warehouse.mv_realtime_sales;
```

### 💡 Interview Insight

> **Q: "How do you get near-real-time data into Redshift?"**
>
> Three options ranked by latency: (1) **Redshift Streaming Ingestion** — materialized views read directly from Kinesis/MSK, refresh in seconds. Newest option, lowest latency. (2) **Firehose → S3 → COPY** — 1-5 minute latency (buffer interval), most common pattern. (3) **Micro-batch with Glue/Lambda** — custom COPY every N minutes. For RetailMart's real-time dashboard, we use Streaming Ingestion. For the nightly warehouse refresh, we use COPY from S3.

---

## Screen 10: Module 4 Quiz & Key Takeaways

### Quiz

**Q1.** RetailMart's Kinesis stream has 4 shards and 5 consumer applications reading in shared mode. What's the per-consumer read throughput?

- A) 2 MB/s per consumer
- B) 8 MB/s per consumer
- C) 1.6 MB/s per consumer
- D) 0.4 MB/s per consumer

**Answer: C.** Total stream read throughput = 4 shards × 2 MB/s = 8 MB/s. Shared among 5 consumers = 8 / 5 = 1.6 MB/s per consumer. With Enhanced Fan-Out, each consumer would get 4 × 2 = 8 MB/s dedicated.

---

**Q2.** You need to load streaming data into S3 as Parquet with minimal code. Which service?

- A) Kinesis Data Streams with Lambda consumer
- B) Kinesis Data Firehose with Parquet conversion
- C) Kinesis Data Analytics
- D) AWS DMS

**Answer: B.** Firehose has built-in Parquet/ORC conversion using the Glue Data Catalog schema. Zero custom code required — configure the delivery stream and it handles batching, conversion, partitioning, and delivery.

---

**Q3.** RetailMart's Redshift table `fact_daily_sales` is distributed by `store_id` (KEY). Queries joining on `product_sku` are slow. Why?

- A) The table needs more sort keys
- B) Rows with matching `product_sku` are scattered across nodes, requiring redistribution
- C) The table is too large
- D) Columnar storage doesn't support joins

**Answer: B.** KEY distribution on `store_id` co-locates rows by store, but a join on `product_sku` requires **redistributing** rows across the network so matching SKUs land on the same node. Solutions: (1) If `dim_product` is small, use DISTSTYLE ALL. (2) Create a materialized view pre-joined on `product_sku`. (3) Evaluate if `product_sku` is a better DISTKEY for the dominant query pattern.

---

**Q4.** Which Redshift feature lets a separate AWS account query your warehouse data without copying it?

- A) Redshift Spectrum
- B) Cross-region snapshots
- C) Data Sharing
- D) Federated queries

**Answer: C.** Data Sharing enables cross-cluster and cross-account access to live data with zero data movement. The consumer uses their own compute, sees real-time data, and pays no storage costs.

---

**Q5.** A Kinesis Flink application needs to detect patterns over the last 10 minutes of data, updating results every 1 minute. Which window type?

- A) Tumbling window of 10 minutes
- B) Sliding window of 10 minutes with 1-minute slide
- C) Session window with 10-minute gap
- D) Global window

**Answer: B.** A sliding (hopping) window of 10 minutes with a 1-minute slide produces an updated result every minute, each covering the preceding 10 minutes of data. A tumbling window would only produce results every 10 minutes with no overlap.

---

### 🔑 Key Takeaways

1. **Kinesis Data Streams** = real-time ingestion backbone. Know the per-shard limits: 1 MB/s in, 2 MB/s out, 1,000 records/sec.
2. **Firehose** = managed delivery to S3/Redshift with built-in Parquet conversion. Near-real-time (60-900s buffer), not real-time.
3. **Kinesis vs. Kafka (MSK)**: Kinesis for AWS-native simplicity; MSK for existing Kafka ecosystems and extreme throughput.
4. **Enhanced Fan-Out**: Dedicated 2 MB/s per consumer via HTTP/2 push. Use when you have 3+ consumers or need < 200ms latency.
5. **Redshift is columnar + MPP**: Understand DISTKEY (KEY, EVEN, ALL) and SORTKEY for query performance.
6. **Spectrum** queries S3 from Redshift — bridge your warehouse and data lake in a single SQL statement.
7. **Redshift Serverless** scales to zero, bills per RPU-second — ideal for variable workloads.
8. **Data Sharing** enables cross-account analytics without data duplication — the Redshift equivalent of BigQuery dataset sharing.
9. **Streaming Ingestion** reads directly from Kinesis into Redshift materialized views — lowest latency option for real-time warehouse updates.
10. **Redshift vs. Athena**: Redshift for complex DW workloads and BI dashboards; Athena for ad-hoc S3 exploration. They complement, not compete.
