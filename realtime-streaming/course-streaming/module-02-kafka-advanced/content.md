# Module 2: Kafka Advanced — EOS, Schema Registry & Connect

> **Scenario Throughout**: ShopStream is growing. You need exactly-once payment processing, schema-governed events across 15 microservices, CDC from your PostgreSQL inventory database, and a monitoring stack that pages you before customers notice.

---

## Screen 1: Exactly-Once Semantics (EOS)

The holy grail of stream processing: every record is processed **exactly once** — no duplicates, no data loss. Kafka achieves this through three layered mechanisms.

**Layer 1 — Idempotent Producer** (within a single partition):

```
Without idempotence:
  Producer sends r5 → Broker writes r5 → ACK lost
  Producer retries  → Broker writes r5 AGAIN → duplicate!

With idempotence (enable.idempotence=true):
  Each producer gets a Producer ID (PID)
  Each record gets a sequence number per partition
  Broker deduplicates: "PID=7, seq=5 already written → skip"

  Producer → [PID=7, seq=5, "order-123 paid"] → Broker
  Retry    → [PID=7, seq=5, "order-123 paid"] → Broker: "already have seq=5, ACK"
```

**Layer 2 — Transactions** (across multiple partitions/topics):

Transactions let you atomically write to multiple topics and commit consumer offsets together. This is essential for "consume-transform-produce" patterns.

```
┌─────────────────────────────────────────────────┐
│              TRANSACTION BOUNDARY                │
│                                                  │
│  1. Read from "order-events" (offset 42)         │
│  2. Process: enrich order with user profile       │
│  3. Write to "enriched-orders" topic             │
│  4. Write to "notification-events" topic         │
│  5. Commit consumer offset (42 → 43)             │
│                                                  │
│  ALL succeed together, or ALL roll back          │
└─────────────────────────────────────────────────┘
```

```python
from confluent_kafka import Producer

producer = Producer({
    'bootstrap.servers': 'kafka:9092',
    'transactional.id': 'order-enrichment-txn-1',  # Unique, stable ID
    'enable.idempotence': True,  # Required for transactions
})
producer.init_transactions()

try:
    producer.begin_transaction()
    producer.produce('enriched-orders', key='order-123', value=enriched_data)
    producer.produce('notification-events', key='order-123', value=notification)
    producer.send_offsets_to_transaction(
        consumer.position(consumer.assignment()),
        consumer.consumer_group_metadata()
    )
    producer.commit_transaction()
except Exception:
    producer.abort_transaction()
```

**Layer 3 — `read_committed` consumers:**

By default, consumers see ALL records — even those from aborted transactions. Setting `isolation.level=read_committed` makes consumers only see records from **committed transactions**:

```
Timeline of partition:
  [r1:committed] [r2:txn-A] [r3:txn-A] [r4:committed] [r5:txn-A]

  read_uncommitted: sees r1, r2, r3, r4, r5 (all records)
  read_committed:   sees r1, then waits... txn-A commits → sees r2, r3, r4, r5
                    if txn-A aborts → sees r1, r4 (r2, r3, r5 filtered out)
```

**EOS end-to-end cost:**

| Component | Overhead |
|---|---|
| Idempotent producer | ~1-3% latency increase |
| Transactions | ~5-10% throughput decrease |
| `read_committed` | Slight delay (waits for txn) |
| Transaction timeout | Default 60s, tune to workload |

**ShopStream decision**: Use full EOS for the payment processing pipeline (order-events → payment-results → inventory-deduct). Use idempotent-only (no transactions) for clickstream enrichment — some duplicates in analytics are acceptable.

### 💡 Interview Insight

> **Q: "How does Kafka achieve exactly-once semantics?"**
> "Through three layers: (1) **Idempotent producers** assign sequence numbers per partition so the broker deduplicates retries — this gives exactly-once within a single partition. (2) **Transactions** extend this across partitions and topics — you can atomically produce to multiple topics and commit consumer offsets in one operation. A stable `transactional.id` survives producer restarts. (3) **`read_committed` consumers** only see records from committed transactions, filtering out aborted writes. Together, these enable end-to-end exactly-once in consume-transform-produce pipelines."

---

## Screen 2: Schema Registry — Governing Your Data Contracts

When 15 microservices produce and consume events, you need **data contracts**. Without schema enforcement, a producer can change a field name and silently break every downstream consumer.

**How Schema Registry works:**

```
┌──────────┐     ┌─────────────────┐     ┌──────────┐
│ Producer │ ──→ │ Schema Registry │ ←── │ Consumer │
│          │     │                 │     │          │
│ 1. Register    │  Stores:        │     │ 3. Fetch schema
│    schema      │  • Schema ID=1  │     │    by ID
│ 2. Send msg    │  • Schema def   │     │ 4. Deserialize
│    with ID=1   │  • Versions     │     │    with schema
└──────────┘     │  • Compat rules │     └──────────┘
                 └─────────────────┘

Wire format of a message:
┌──────┬───────────┬──────────────────────────┐
│ 0x00 │ Schema ID │ Avro/Protobuf payload    │
│ 1B   │ 4 bytes   │ variable                 │
└──────┴───────────┴──────────────────────────┘
  Magic byte  Schema ID   Serialized data (no schema embedded!)
```

The schema is **not embedded** in every message — only a 4-byte ID. This saves enormous bandwidth. The consumer fetches and caches the schema from the registry.

**Avro vs Protobuf vs JSON Schema:**

| Feature | Avro | Protobuf | JSON Schema |
|---|---|---|---|
| Encoding | Binary, compact | Binary, compact | Text, verbose |
| Schema in payload | No (registry ID) | No (registry ID) | Inline or ref |
| Schema evolution | Excellent | Good | Limited |
| Code generation | Optional | Required | N/A |
| Kafka ecosystem | Best supported | Growing | Basic |
| Size (1000 orders) | ~45 KB | ~42 KB | ~180 KB |
| Best for | Data pipelines | gRPC + streaming | REST APIs |

**ShopStream Avro schema example:**

```json
{
  "type": "record",
  "name": "OrderEvent",
  "namespace": "com.shopstream.events",
  "fields": [
    {"name": "order_id", "type": "string"},
    {"name": "event_type", "type": {"type": "enum", "name": "EventType",
      "symbols": ["CREATED", "PAID", "SHIPPED", "DELIVERED", "CANCELLED"]}},
    {"name": "amount", "type": "double"},
    {"name": "currency", "type": "string", "default": "USD"},
    {"name": "timestamp", "type": "long"},
    {"name": "customer_id", "type": "string"},
    {"name": "items", "type": {"type": "array", "items": {
      "type": "record", "name": "OrderItem", "fields": [
        {"name": "sku", "type": "string"},
        {"name": "quantity", "type": "int"},
        {"name": "price", "type": "double"}
      ]
    }}}
  ]
}
```

### 💡 Interview Insight

> **Q: "Why use Schema Registry instead of just JSON?"**
> "Three reasons: (1) **Contract enforcement** — Schema Registry validates every message against a registered schema, preventing producers from breaking downstream consumers. (2) **Size efficiency** — Instead of embedding schema in every message, only a 4-byte ID is sent. For Avro binary encoding, this cuts message sizes by 70-80% versus JSON. (3) **Compatibility checking** — The registry enforces evolution rules (backward, forward, full) so schema changes are guaranteed non-breaking. In a microservices architecture with 15 teams, this prevents 'schema surprise' production incidents."

---

## Screen 3: Schema Evolution & Compatibility Modes

Schemas change — that's inevitable. The question is: **how do you change them without breaking running consumers?**

**Compatibility modes:**

```
BACKWARD (default): New schema can READ data written with old schema
  → Safe to deploy new CONSUMERS first, then new PRODUCERS

FORWARD: Old schema can READ data written with new schema
  → Safe to deploy new PRODUCERS first, then new CONSUMERS

FULL: Both BACKWARD + FORWARD
  → Deploy in any order — safest, but most restrictive

NONE: No compatibility checking
  → Chaos mode. Don't do this in production!

TRANSITIVE variants: Check against ALL previous versions, not just last
  → BACKWARD_TRANSITIVE, FORWARD_TRANSITIVE, FULL_TRANSITIVE
```

**What changes are safe under each mode?**

| Change | BACKWARD | FORWARD | FULL |
|---|---|---|---|
| Add field with default | ✅ | ✅ | ✅ |
| Add field without default | ❌ | ✅ | ❌ |
| Remove field with default | ✅ | ❌ | ❌ |
| Remove field without default | ❌ | ❌ | ❌ |
| Rename field | ❌ | ❌ | ❌ |
| Change field type | ❌ | ❌ | ❌ |

**ShopStream evolution example:**

Version 1 (current):
```json
{"name": "order_id", "type": "string"},
{"name": "amount", "type": "double"},
{"name": "customer_id", "type": "string"}
```

Version 2 (add optional `shipping_address`):
```json
{"name": "order_id", "type": "string"},
{"name": "amount", "type": "double"},
{"name": "customer_id", "type": "string"},
{"name": "shipping_address", "type": ["null", "string"], "default": null}
```

This is **BACKWARD compatible** — new consumers reading old records will see `shipping_address=null`. Old consumers reading new records will simply ignore the unknown field (Avro's behavior).

**Breaking change example** — renaming `amount` to `total_amount`:

```
DON'T: Rename the field → breaks all existing consumers

DO: Add new field, deprecate old:
  v2: add "total_amount" with default, keep "amount"
  v3: (months later, all consumers migrated) remove "amount"
```

**Subject naming strategies:**
- `TopicNameStrategy` (default): `order-events-value`, `order-events-key`
- `RecordNameStrategy`: `com.shopstream.events.OrderEvent` — shared across topics
- `TopicRecordNameStrategy`: `order-events-com.shopstream.events.OrderEvent`

### 💡 Interview Insight

> **Q: "How do you handle a breaking schema change in production?"**
> "You never make a breaking change directly. Instead, use a multi-step approach: (1) Add the new field with a default value (backward compatible). (2) Deploy all consumers to handle both old and new format. (3) Deploy producers to write the new field. (4) After a full retention period, optionally remove the deprecated field. For truly incompatible changes, create a new topic (e.g., `order-events-v2`) and run both in parallel during migration. The Schema Registry's compatibility checks act as a CI gate — they reject breaking changes before they reach production."

---

## Screen 4: Kafka Connect — Bridging the Data World

Kafka Connect is a framework for streaming data between Kafka and external systems without writing custom producer/consumer code. It's the **integration backbone** of ShopStream.

```
                    Kafka Connect Cluster
                    ┌────────────────────────┐
                    │    Connect Workers     │
                    │  ┌──────────────────┐  │
External Systems    │  │  Source Connectors│  │    Kafka Topics
                    │  │                  │  │
  PostgreSQL   ────►│  │  Debezium PG     │──┼──► inventory-cdc
  (inventory DB)    │  │                  │  │
                    │  ├──────────────────┤  │
  MySQL        ────►│  │  Debezium MySQL  │──┼──► orders-cdc
  (orders DB)       │  │                  │  │
                    │  ├──────────────────┤  │
                    │  │  Sink Connectors  │  │
                    │  │                  │  │
  Elasticsearch◄───┤  │  ES Sink         │◄─┼──  order-events
                    │  │                  │  │
  S3 (archive) ◄───┤  │  S3 Sink         │◄─┼──  clickstream
                    │  └──────────────────┘  │
                    └────────────────────────┘
```

**Source Connectors** read from external systems → write to Kafka topics.
**Sink Connectors** read from Kafka topics → write to external systems.

**Debezium CDC (Change Data Capture)** — the star source connector:

Debezium reads the PostgreSQL WAL (write-ahead log) to capture every INSERT, UPDATE, DELETE as a Kafka event. No polling, no application changes, no triggers.

```
PostgreSQL WAL → Debezium → Kafka Topic

  Before:  inventory table: {sku: "A100", qty: 50}
  UPDATE:  SET qty = 45 WHERE sku = 'A100'

  Debezium emits:
  {
    "before": {"sku": "A100", "qty": 50},
    "after":  {"sku": "A100", "qty": 45},
    "op": "u",
    "ts_ms": 1712345678000,
    "source": {"table": "inventory", "lsn": 123456}
  }
```

**Single Message Transforms (SMTs)** — lightweight transformations within Connect:

```json
{
  "transforms": "extractAfter,addTimestamp,maskPII",
  "transforms.extractAfter.type": "io.debezium.transforms.ExtractNewRecordState",
  "transforms.addTimestamp.type": "org.apache.kafka.connect.transforms.InsertField$Value",
  "transforms.addTimestamp.timestamp.field": "kafka_timestamp",
  "transforms.maskPII.type": "org.apache.kafka.connect.transforms.MaskField$Value",
  "transforms.maskPII.fields": "email,phone",
  "transforms.maskPII.replacement": "***"
}
```

**Dead Letter Queue (DLQ)** — handling poison pills:

```json
{
  "errors.tolerance": "all",
  "errors.deadletterqueue.topic.name": "dlq-inventory-sink",
  "errors.deadletterqueue.topic.replication.factor": 3,
  "errors.deadletterqueue.context.headers.enable": true
}
```

Records that can't be processed (deserialization errors, schema mismatches) go to the DLQ topic instead of crashing the connector. Headers contain error details for debugging.

**Connector deployment modes:**

| Mode | Workers | Use Case |
|---|---|---|
| Standalone | 1 worker | Dev/test, simple pipelines |
| Distributed | N workers | Production, HA, auto-rebalancing |

### 💡 Interview Insight

> **Q: "When would you use Kafka Connect vs writing a custom consumer?"**
> "Kafka Connect for standard integrations — databases (via Debezium CDC), cloud storage (S3 sink), search engines (Elasticsearch sink), JDBC sources. It handles offset management, parallelism, fault tolerance, and schema translation out of the box. I'd write a custom consumer only when I need complex business logic, custom error handling, or the target system has no existing connector. Connect also supports SMTs for lightweight transformations and DLQs for poison pill handling, which would be tedious to implement manually."

---

## Screen 5: Kafka Operations & Monitoring

Running Kafka in production requires mastering CLI tools, understanding consumer lag, and building observability.

**Essential CLI operations:**

```bash
# Topic management
kafka-topics.sh --create --topic order-events \
  --partitions 12 --replication-factor 3 \
  --config retention.ms=7776000000 \     # 90 days
  --config min.insync.replicas=2

kafka-topics.sh --describe --topic order-events
# Shows partition leaders, ISR, replicas per partition

# Consumer group inspection
kafka-consumer-groups.sh --describe --group order-processing
# GROUP        TOPIC         PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# order-proc   order-events  0          145023          145030          7
# order-proc   order-events  1          98210           98210           0
# order-proc   order-events  2          112044          112200          156

# Reset offsets (careful!)
kafka-consumer-groups.sh --reset-offsets \
  --group order-processing --topic order-events \
  --to-datetime 2025-01-15T00:00:00.000 --execute

# Performance testing
kafka-producer-perf-test.sh --topic test \
  --num-records 1000000 --record-size 500 \
  --throughput -1 --producer-props bootstrap.servers=kafka:9092
```

**Consumer lag** — the #1 metric to monitor:

```
Consumer Lag = Log End Offset - Consumer's Committed Offset

  Partition 0: |=====[consumed]======|---[lag: 156]---|→ (new records arriving)
                                     ↑                ↑
                              committed offset    log-end offset

  Lag growing → consumer can't keep up → investigate!
  Lag stable  → consumer is healthy
  Lag = 0     → consumer is caught up (real-time)
```

**Monitoring stack:**

```
┌──────────┐    ┌─────────────────┐    ┌────────────┐    ┌─────────┐
│  Kafka   │    │ Kafka Exporter  │    │ Prometheus │    │ Grafana │
│  Brokers │───►│ (JMX → metrics) │───►│ (scrape)   │───►│ (dash)  │
│  (JMX)   │    │ or JMX Exporter │    │            │    │         │
└──────────┘    └─────────────────┘    └────────────┘    └─────────┘
```

**Critical metrics to alert on:**

| Metric | Warning | Critical | Why |
|---|---|---|---|
| Consumer lag (records) | >1000 | >10000 | Consumer falling behind |
| Under-replicated partitions | >0 | >0 for 5min | Replica not in sync |
| Offline partitions | >0 | >0 | Data unavailable |
| ISR shrink rate | >0 | Sustained | Brokers struggling |
| Request latency p99 | >100ms | >500ms | Performance degradation |
| Disk usage % | >70% | >85% | Retention pressure |
| Network throughput | >70% NIC | >85% NIC | Bandwidth saturation |

**JMX metrics deep dive:**

```
# Broker metrics
kafka.server:type=BrokerTopicMetrics,name=MessagesInPerSec
kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec
kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions

# Producer metrics
kafka.producer:type=producer-metrics,client-id=*,name=record-send-rate
kafka.producer:type=producer-metrics,client-id=*,name=record-error-rate

# Consumer metrics
kafka.consumer:type=consumer-fetch-manager-metrics,name=records-lag-max
kafka.consumer:type=consumer-coordinator-metrics,name=rebalance-latency-avg
```

### 💡 Interview Insight

> **Q: "How do you detect and handle consumer lag in production?"**
> "I monitor consumer lag via Kafka Exporter + Prometheus + Grafana. Alert at two thresholds: warning if lag >1000 records (investigate), critical if lag >10000 (page on-call). Root causes: (1) slow processing — profile consumer logic, increase parallelism. (2) producer burst — auto-scale consumers. (3) rebalancing storm — check for flapping consumers. (4) GC pauses — tune JVM. Immediate mitigation: scale consumers up to partition count. Long-term: increase partitions if needed, optimize processing logic, consider async processing with a local buffer."

---

## Screen 6: Managed Kafka — Cloud Offerings

Running your own Kafka cluster is operationally complex. Managed services handle the infrastructure so you focus on building.

```
┌─────────────────┬──────────────────┬──────────────────┬──────────────────┐
│ Feature         │ Confluent Cloud  │ AWS MSK          │ Azure Event Hubs │
├─────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Protocol        │ Kafka native     │ Kafka native     │ Kafka protocol   │
│                 │                  │                  │ (compatible, not │
│                 │                  │                  │  actual Kafka)   │
├─────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Schema Registry │ Built-in         │ Glue Schema Reg  │ Azure Schema Reg │
├─────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Kafka Connect   │ Managed          │ MSK Connect      │ No native support│
├─────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Stream          │ ksqlDB           │ Flink (managed)  │ Stream Analytics │
│ Processing      │                  │                  │                  │
├─────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Pricing Model   │ Per CKU (usage)  │ Per broker-hour  │ Per throughput   │
│                 │                  │                  │ unit (TU)        │
├─────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Multi-cloud     │ AWS, GCP, Azure  │ AWS only         │ Azure only       │
├─────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Serverless      │ Yes              │ Serverless mode  │ Yes              │
├─────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Max partitions  │ Unlimited*       │ Depends on broker│ Limited by TUs   │
│                 │                  │ count            │                  │
├─────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Best for        │ Full Kafka       │ AWS-native teams │ Azure shops,     │
│                 │ ecosystem,       │ cost-conscious   │ simple streaming │
│                 │ multi-cloud      │                  │                  │
└─────────────────┴──────────────────┴──────────────────┴──────────────────┘
```

**ShopStream scenario**: If you're multi-cloud or want the richest Kafka ecosystem (Schema Registry, ksqlDB, managed connectors), Confluent Cloud. If you're all-in on AWS and cost-sensitive, MSK Serverless. If you're an Azure shop and just need basic event streaming, Event Hubs with Kafka protocol.

### 💡 Interview Insight

> **Q: "How would you choose between self-managed Kafka and a managed service?"**
> "Self-managed makes sense when you need full control over configuration, have a dedicated platform team, or have strict data residency requirements. Managed services (Confluent Cloud, MSK) are better for most teams — they handle broker provisioning, patching, replication, and monitoring. I'd evaluate: (1) team Kafka expertise, (2) operational budget for running infrastructure, (3) feature needs (Schema Registry, Connect, ksqlDB), and (4) cloud lock-in tolerance. At Walmart's scale, the operational cost of self-managing 100+ brokers often exceeds the managed service premium."

---

## Screen 7: Pub/Sub Technology Comparison

**When to use what:**

```
┌────────────────┬───────────┬──────────┬────────────┬───────────┬──────────┐
│ Feature        │ Kafka     │ Kinesis  │ GCP PubSub │ SQS/SNS   │ Pulsar   │
├────────────────┼───────────┼──────────┼────────────┼───────────┼──────────┤
│ Model          │ Log-based │ Log-based│ True pubsub│ Queue/    │ Log-based│
│                │           │          │            │ fanout    │ + queue  │
├────────────────┼───────────┼──────────┼────────────┼───────────┼──────────┤
│ Ordering       │ Per-      │ Per-shard│ Per-key    │ FIFO SQS  │ Per-     │
│                │ partition │          │ (ordering  │ only      │ partition│
│                │           │          │  key)      │           │          │
├────────────────┼───────────┼──────────┼────────────┼───────────┼──────────┤
│ Retention      │ Unlimited │ 7 days   │ 31 days    │ 14 days   │ Unlimited│
│                │ (config)  │ max      │ max        │ max       │ (tiered) │
├────────────────┼───────────┼──────────┼────────────┼───────────┼──────────┤
│ Replay         │ Full      │ Full     │ Seek to    │ No replay │ Full     │
│                │ (offset)  │ (seq#)   │ timestamp  │           │          │
├────────────────┼───────────┼──────────┼────────────┼───────────┼──────────┤
│ Throughput     │ Very high │ 1MB/s    │ High       │ Moderate  │ Very high│
│                │ (scale    │ per shard│ (auto-     │           │ (segment │
│                │ with      │          │  scale)    │           │  storage)│
│                │ partitions│          │            │           │          │
├────────────────┼───────────┼──────────┼────────────┼───────────┼──────────┤
│ Exactly-once   │ Yes       │ No (at-  │ At-least-  │ At-least- │ Yes      │
│                │ (native)  │ least    │ once       │ once      │ (txn)    │
│                │           │ once)    │            │ (FIFO     │          │
│                │           │          │            │  dedup)   │          │
├────────────────┼───────────┼──────────┼────────────┼───────────┼──────────┤
│ Ecosystem      │ Huge      │ AWS only │ GCP only   │ AWS only  │ Growing  │
│                │ (Connect, │          │            │           │ (IO      │
│                │  Schema   │          │            │           │  connect)│
│                │  Reg, etc)│          │            │           │          │
├────────────────┼───────────┼──────────┼────────────┼───────────┼──────────┤
│ Ops complexity │ High      │ Low      │ Low        │ Very low  │ High     │
│ (self-managed) │           │ (managed)│ (managed)  │ (managed) │          │
├────────────────┼───────────┼──────────┼────────────┼───────────┼──────────┤
│ Best for       │ Event     │ AWS log/ │ GCP event  │ Task      │ Multi-   │
│                │ streaming,│ event    │ routing,   │ queues,   │ tenant,  │
│                │ CDC, event│ ingest   │ serverless │ decoupling│ geo-rep, │
│                │ sourcing  │          │ triggers   │           │ tiered   │
│                │           │          │            │           │ storage  │
└────────────────┴───────────┴──────────┴────────────┴───────────┴──────────┘
```

**Decision framework for ShopStream:**

```
Need replay / event sourcing?     → Kafka or Pulsar
AWS-only, simple ingest?          → Kinesis
GCP, serverless triggers?         → Google Pub/Sub
Task queue / work distribution?   → SQS
Fan-out notifications?            → SNS → SQS
Multi-tenant / geo-replication?   → Pulsar
Full ecosystem (CDC, schemas)?    → Kafka
```

**Pulsar vs Kafka** — the emerging debate:

Pulsar separates compute (brokers) from storage (BookKeeper), enabling independent scaling. It has built-in multi-tenancy, geo-replication, and tiered storage. However, Kafka's ecosystem (Connect, Schema Registry, ksqlDB, Flink integration) is vastly larger, and KRaft + tiered storage (KIP-405) is closing the architecture gap.

### 💡 Interview Insight

> **Q: "When would you choose Kinesis over Kafka?"**
> "Kinesis when you're fully AWS-native, want zero operational overhead, and your use case is straightforward event/log ingestion with 7-day retention. Kinesis auto-scales shards and integrates natively with Lambda, Firehose, and Analytics. I'd choose Kafka when I need: longer retention, log compaction, exactly-once semantics, Schema Registry for data governance, Kafka Connect for 200+ integrations, or multi-cloud portability. Kafka is the choice for event-driven architectures; Kinesis is great for AWS-native data pipelines."

---

## Screen 8: Advanced Producer & Consumer Patterns

**Pattern 1 — Outbox Pattern** (for transactional consistency):

```
Problem: Service writes to DB AND produces to Kafka.
         If Kafka fails after DB commit → inconsistency

Solution: Outbox Pattern
  1. Write business data + outbox event to DB in ONE transaction
  2. Debezium CDC reads outbox table → publishes to Kafka
  3. DB is the source of truth; Kafka follows

┌─────────────────────────────┐
│ PostgreSQL                  │
│ ┌─────────────┬───────────┐ │     ┌─────────┐
│ │ orders      │ outbox    │ │────►│ Debezium│───► Kafka
│ │ (business)  │ (events)  │ │ CDC │         │
│ └─────────────┴───────────┘ │     └─────────┘
│   BEGIN;                    │
│   INSERT INTO orders(...);  │
│   INSERT INTO outbox(...);  │
│   COMMIT;                   │
└─────────────────────────────┘
```

**Pattern 2 — Dead Letter Topic + Retry:**

```
Main Topic ──► Consumer ──► Process
                  │ (failure)
                  ▼
              Retry-1 Topic ──► (delay 1 min) ──► Consumer ──► Process
                                                      │ (failure)
                                                      ▼
                                                  Retry-2 Topic ──► (delay 5 min)
                                                                        │ (failure)
                                                                        ▼
                                                                    DLQ Topic
                                                                    (manual review)
```

**Pattern 3 — Event Sourcing with Kafka:**

```
Command: "Place Order #123"
  │
  ▼
┌──────────────────────────────────────────────────────────┐
│ order-events (compacted, key=order_id)                   │
│ [order-123:Created] [order-123:Paid] [order-123:Shipped] │
└──────────────────────────────────────────────────────────┘
  │
  ▼  Materialize (consumer rebuilds state)
┌──────────────────────┐
│ Order #123           │
│ Status: Shipped      │
│ Amount: $99.50       │
│ History: full audit  │
└──────────────────────┘
```

### 💡 Interview Insight

> **Q: "How do you ensure consistency between a database write and a Kafka publish?"**
> "The Outbox Pattern. Instead of writing to the database and Kafka separately (dual-write problem), I write the business data and an outbox event in a single database transaction. Then Debezium CDC captures the outbox table changes and publishes them to Kafka. This guarantees that if the DB transaction commits, the event will eventually reach Kafka. It's eventual consistency with guaranteed delivery — no dual-write, no distributed transaction."

---

## Quiz: Module 2

**Q1:** An idempotent producer retries sending a record. How does the broker detect the duplicate?

A) By checking the message hash
B) By matching the Producer ID (PID) and sequence number
C) By comparing timestamps
D) By checking consumer offsets

**Answer:** B) Each idempotent producer has a unique PID. Records are tagged with a monotonic sequence number per partition. The broker deduplicates by checking if it already has a record with that PID + sequence number.

---

**Q2:** Your Avro schema has a field `amount: double`. You want to add `currency: string`. Under BACKWARD compatibility, which is correct?

A) `{"name": "currency", "type": "string"}`
B) `{"name": "currency", "type": "string", "default": "USD"}`
C) Either works
D) You must create a new topic

**Answer:** B) BACKWARD compatibility requires new fields to have defaults so that new consumers can read old records (which lack the field) without error.

---

**Q3:** A Debezium connector captures a database DELETE. What does the Kafka record look like?

A) The record is deleted from Kafka
B) A tombstone record (key = PK, value = null) is published
C) A record with `"op": "d"` and the `before` state is published
D) Both B and C — Debezium publishes the delete event AND a tombstone

**Answer:** D) Debezium publishes the delete event with `"op": "d"` and the before-state, and also publishes a tombstone (null value) for log compaction to eventually remove the key.

---

**Q4:** You set `isolation.level=read_committed` on a consumer. An in-progress transaction has written records to the partition. What does the consumer see?

A) All records including uncommitted
B) Only committed records; uncommitted are invisible until commit/abort
C) An error
D) Uncommitted records marked with a flag

**Answer:** B) `read_committed` consumers only see records from committed transactions. Uncommitted records are buffered and become visible only after the transaction commits (or filtered if it aborts).

---

**Q5:** When should you prefer the Outbox Pattern over direct Kafka produce?

A) When you need to atomically update a database AND publish an event
B) When you want higher Kafka throughput
C) When you don't have Schema Registry
D) When using log compaction

**Answer:** A) The Outbox Pattern solves the dual-write problem — it ensures database changes and Kafka events are consistent by making the database the single source of truth with CDC forwarding events to Kafka.

---

## Key Takeaways

1. **Exactly-Once** requires three layers: idempotent producer + transactions + `read_committed` consumers.
2. **Schema Registry** is essential governance — use Avro for data pipelines, enforce BACKWARD compatibility.
3. **Schema evolution** must be planned: always add fields with defaults, never rename or remove fields directly.
4. **Kafka Connect** handles standard integrations; **Debezium CDC** is the gold standard for database change capture.
5. **Dead Letter Queues** in Connect prevent poison pills from crashing connectors.
6. **Consumer lag** is the #1 production metric — monitor, alert, and have a scaling playbook.
7. **Managed Kafka** (Confluent, MSK) reduces ops burden; choose based on ecosystem needs and cloud strategy.
8. **Kafka vs alternatives**: Kafka for event streaming + replay + ecosystem; Kinesis/Pub/Sub for cloud-native simplicity; SQS for task queues.
9. **Outbox Pattern** solves the dual-write problem between databases and Kafka.
10. **Transactions** have overhead (~5-10% throughput) — use them where exactly-once matters (payments), not everywhere (clickstream).
