# Real-Time Analytics & CDC Pipelines

---

## Screen 1: Designing a Real-Time Analytics Pipeline — Overview

When an interviewer says *"Design a real-time analytics system,"* they are testing three things simultaneously: your ability to decompose an ambiguous problem into concrete requirements, your knowledge of modern streaming architectures, and your judgment about tradeoffs. This screen gives you the framework to nail all three.

### Use Case Examples

Real-time analytics shows up everywhere in industry. Grounding your answer in a concrete scenario immediately signals experience:

- **Real-time dashboards (Uber trip metrics):** Dispatchers and city ops teams watch live ride counts, ETAs, and surge pricing per geo-zone. Data must refresh every 1-5 seconds across millions of concurrent trips.
- **Live monitoring & fraud detection:** A payments platform scores every transaction in under 100 ms. A single late alert means money lost. The system ingests 50k+ events/sec and must maintain stateful feature aggregations (e.g., "transactions from this card in the last 10 minutes").
- **Operational analytics (inventory tracking):** A retailer tracks stock-level changes across 5,000 stores. Each POS scan, receiving dock event, and online order triggers an inventory delta. Buyers need sub-minute visibility to prevent stock-outs.

### Key Requirements to Clarify

Before drawing a single box on the whiteboard, pin down the requirements:

| Requirement | Question to Ask | Typical Range |
|---|---|---|
| **Latency** | How fresh must the data be? | Sub-100 ms → seconds → minutes |
| **Throughput** | How many events per second? | Thousands → millions |
| **Query pattern** | Pre-defined aggregations or ad-hoc? | Dashboards vs. exploratory |
| **Retention** | How far back do queries need to reach? | Hours → years |
| **Consistency** | Exactly-once or at-least-once acceptable? | Depends on use case |
| **Concurrency** | How many simultaneous consumers/queries? | Tens → thousands |

### Architecture Overview

Almost every real-time analytics system follows this five-stage spine:

```
Sources → Ingestion (Kafka) → Stream Processing (Flink/Spark) → Serving (OLAP DB) → Consumption (Dashboards / APIs)
```

1. **Sources** — Application databases, clickstream, IoT sensors, mobile SDKs, third-party webhooks.
2. **Ingestion** — Apache Kafka acts as the durable, replayable event backbone. All events land here first.
3. **Stream Processing** — Apache Flink or Spark Structured Streaming performs filtering, enrichment, windowed aggregations, and joins against reference data.
4. **Serving** — A specialized OLAabase (Druid, Pinot, ClickHouse) stores pre-aggregated or raw data optimized for sub-second queries at high concurrency.
5. **Consumption** — Dashboards (Grafana, Superset, Tableau), REST/GraphQL APIs, or downstream microservices query the serving layer.

This is your *default blueprint*. Every subsequent screen zooms into one of these boxes.

### 💡 Interview Insight

> **Start by clarifying latency requirements — "real-time" means different things to different people.** Sub-100 ms is *operational real-time* (fraud scoring, bidding). Seconds is *dashboard real-time* (Uber city view). Minutes is *near-real-time analytics* (inventory reconciliation). Naming the tier explicitly shows the interviewer you've lived through production systems where this distinction caused real arguments.

---

## Screen 2: Kafka as the Ingestion Backbone

Kafka is the gravitational center of every modern streaming architecture. Understanding it deeply — not just "it's a message queue" — separates senior candidates from junior ones.

### Why Kafka?

Kafka provides five properties that no traditional message broker delivers simultaneously:

1. **Durability** — Messages are written to disk and replicated across brokers.
2. **Ordering** — Messages within a partition are strictly ordered by offset — critical for CDC where UPDATE must follow INSERT.
3. **Replayability** — Consumers can seek to any offset and reprocess data, enabling backfills and bug fixes.
4. **Horizontal scalability** — Add partitions and brokers to scale throughput linearly.
5. **Decoupling** — Producers and consumers are fully independent.

### Kafka Architecture for RT Analytics

```
Producers → Topics (partitioned) → Consumer Groups → Stream Processors
```

- **Topics** are organized by domain: `rides.events`, `payments.transactions`, `inventory.updates`. One topic per event type keeps schemas clean and access control simple.
- **Partitions** determine parallelism and ordering. A topic with 64 partitions can have up to 64 consumers processing in parallel within a consumer group. The *partition key* determines which partition a message lands on — all messages with the same key go to the same partition, preserving order for that key.
- **Consumer Groups** enable independent processing. The `fraud-detection` group and the `analytics-dashboard` group both consume from `payments.transactions` at their own pace, with their own offsets.
- **Retention** — Hot retention of 7 days on broker disk is standard. With Kafka's Tiered Storage (KIP-405, GA in Kafka 3.6+), older segments are offloaded to S3/GCS, enabling weeks or months of replayability at low cost.

### Schema Management

Without schema governance, Kafka topics become the Wild West. The Confluent Schema Registry is the industry standard:

- **Avro or Protobuf schemas** are registered per topic (subject = `<topic>-value`).
- **Compatibility modes** prevent breaking changes:
  - *BACKWARD* — new schema can read old data (safe for consumers upgrading first).
  - *FORWARD* — old schema can read new data (safe for producers upgrading first).
  - *FULL* — both directions compatible (safest, most restrictive).
- **Schema evolution strategy** — Add nullable fields with defaults. Never remove or rename fields without a migration plan. Use schema IDs embedded in message headers for efficient deserialization.

### Kafka Connect

Kafka Connect provides a plug-and-play framework for moving data in and out of Kafka without writing custom code:

- **Source connectors:** JDBC Source (poll tables), Debezium (CDC from database WAL/binlog), S3 Source (file ingestion).
- **Sink connectors:** Elasticsearch Sink (search indexing), S3 Sink (data lake landing), JDBC Sink (write to OLTP/OLAP).
- **Exactly-once semantics:** Combine idempotent producers (`enable.idempotence=true`) with transactional consumers (`isolation.level=read_committed`) to achieve end-to-end exactly-once. Kafka Connect supports this natively with `exactly.once.source.support=enabled`.

### Kafka Streams vs. External Processing

- **Kafka Streams** — Lightweight Java library running inside your app process. Perfect for simple transforms: filtering, mapping, repartitioning.
- **External processing (Flink/Spark)** — Required for complex aggregations, multi-stream joins, windowed computations, and ML feature engineering.

Rule of thumb: if your logic fits in a `KStream.filter().mapValues().to()` chain, use Kafka Streams. If you need windowed joins with watermarks, reach for Flink.

### 💡 Interview Insight

> **Mention partition key design early — it determines ordering guarantees and parallelism.** For a ride-sharing analytics system, partitioning by `driver_id` guarantees all events for a driver are ordered and processed by the same consumer. But if 5% of drivers generate 50% of rides, you get hot partitions. A composite key like `city_id + driver_id` can spread load more evenly. Interviewers love hearing you reason through this tradeoff.

---

## Screen 3: Stream Processing — Flink vs Spark Structured Streaming

The stream processor is the brain of your pipeline. It takes raw events and transforms them into meaningful, queryable aggregations. The two dominant frameworks — Apache Flink and Spark Structured Streaming — take fundamentally different approaches.

### Apache Flink Deep Dive

Flink was built from the ground up as a streaming engine. Batch is a special case of streaming in Flink's worldview, not the other way around.

- **True event-time processing** — Flink natively distinguishes between *event time* (when it happened), *ingestion time* (when Kafka received it), and *processing time* (when Flink processed it). Event-time processing with watermarks is first-class, not bolted on.
- **Per-record processing** — Each event is processed individually as it arrives. There is no micro-batch boundary. This yields true millisecond-level latency for stateful computations.
- **RocksDB state backend** — Flink stores operator state in an embedded RocksDB instance on local SSD. This enables terabytes of state per operator without blowing up JVM heap. State is asynchronously checkpointed to S3/HDFS for fault tolerance.
- **Savepoints for upgrades** — Before deploying a new version of your Flink job, you trigger a savepoint — a consistent snapshot of all operator state. The new job restarts from the savepoint with zero data loss. This is essential for production operations.
- **Exactly-once via checkpointing** — Flink's Chandy-Lamport distributed snapshot algorithm periodically checkpoints operator state and Kafka offsets atomically. On failure, the job restarts from the last checkpoint, replays events from Kafka, and produces no duplicates (when combined with idempotent sinks or two-phase commit).

### Flink Windowing

Flink's windowing is the richest in the streaming ecosystem:

- **Tumbling windows** — Fixed-size, non-overlapping. Example: count rides per 1-minute window.
- **Sliding windows** — Fixed-size, overlapping. Example: count rides in the last 5 minutes, updated every 30 seconds.
- **Session windows** — Dynamic, gap-based. A session window closes after N minutes of inactivity. Example: group a user's clickstream into browsing sessions.
- **Late data handling** — `allowedLateness(Duration.ofMinutes(5))` keeps the window open for stragglers. Events arriving even later are routed to a *side output* for separate handling (e.g., corrections, reconciliation).

### Spark Structured Streaming Deep Dive

Spark Structured Streaming builds streaming on top of the Spark SQL engine. Its philosophy: use the same DataFrame/Dataset API for both batch and streaming, and the engine handles the complexity.

- **Micro-batch processing** — By default, Spark polls Kafka every 100 ms (configurable trigger interval), pulls a batch of new records, processes them as a DataFrame, and writes results. This is simple, reliable, and leverages Spark's mature SQL optimizer — but it introduces at least one trigger-interval of latency.
- **Continuous processing mode** — Experimental since Spark 2.3, this mode processes records one-at-a-time with ~1 ms latency. However, it supports only a subset of operations (map-like transforms, no aggregations) and lacks production maturity. Most teams stick with micro-batch.
- **Watermarks for late data** — `withWatermark("event_time", "10 minutes")` tells Spark to drop events older than 10 minutes past the current watermark. This bounds state size but means late events are silently dropped — no side output like Flink.
- **Stateful processing** — `mapGroupsWithState` and `flatMapGroupsWithState` enable arbitrary stateful logic per key group, with timeout support for session-like patterns.

### Comparison Table

| Aspect | Flink | Spark Structured Streaming |
|---|---|---|
| **Processing Model** | Per-record (true streaming) | Micro-batch (default) |
| **Latency** | Milliseconds | Seconds (micro-batch trigger) |
| **State Management** | RocksDB, incremental checkpoints | In-memory + HDFS/S3 state store |
| **State Size** | Terabytes (disk-backed) | Gigabytes (memory-bound) |
| **Exactly-Once** | Native distributed snapshots | Micro-batch atomicity + WAL |
| **Windowing** | Tumbling, sliding, session, custom | Tumbling, sliding |
| **Late Data** | Allowed lateness + side outputs | Watermarks only (drop late) |
| **Ecosystem** | Growing; strong in pure streaming | Mature; unified batch + stream |
| **Operational Complexity** | Higher (dedicated cluster, savepoints) | Lower (leverages existing Spark) |
| **When to Use** | Ultra-low latency, complex event processing, large state | Unified batch + streaming workloads, existing Spark ecosystem |

### 💡 Interview Insight

> **Don't just say "Flink is better for streaming" — explain WHY.** Per-record processing eliminates the micro-batch latency floor. RocksDB state backend handles larger state than Spark's in-memory state store (critical for high-cardinality aggregations like per-user metrics across millions of users). And Flink's side outputs for late data give you more control than Spark's drop-or-keep binary. But also acknowledge Spark's strengths: if the team already runs Spark for batch ETL and latency requirements are seconds (not milliseconds), Structured Streaming avoids operating a second framework.

---

## Screen 4: Serving Layer — OLAP Databases for Real-Time Queries

Your stream processor produces aggregations. Now those aggregations need to be queryable — by dashboards, APIs, and analysts — with sub-second latency at potentially thousands of concurrent queries. This is where specialized OLAP databases come in.

### Why Not Just Use Snowflake or BigQuery?

Traditional cloud warehouses are phenomenal for analytical queries on historical data, but they have characteristics that make them suboptimal for real-time serving:

- **Query compilation overhead** — Snowflake compiles each query into an execution plan, spins up virtual warehouse compute, and executes. This adds 500 ms–2 s of baseline latency even for trivial queries.
- **Not optimized for high-concurrency point lookups** — A dashboard with 200 panels, each auto-refreshing every 5 seconds, generates 2,400 queries/minute. Warehouses throttle or charge exorbitantly at this concurrency.
- **Ingestion latency** — Loading data into Snowflake via COPY or Snowpipe takes seconds to minutes. For a 1-second dashboard, this is too slow.

Real-time OLAP databases are purpose-built for this: ingest continuously, query instantly, serve thousands of concurrent users.

### Apache Druid

Druid is a column-oriented, distributed OLAP database designed for sub-second queries on event data.

- **Pre-aggregation at ingestion (rollup)** — Roll up raw events at ingest time. 1 billion clicks becomes manageable counts per (page, country, hour), reducing storage by 100x.
- **Segment-based storage** — Immutable segments partitioned by time enable efficient time-based pruning.
- **Real-time + historical nodes** — Real-time nodes ingest from Kafka with sub-second latency; historical nodes serve persisted segments from S3. Queries merge results from both.
- **Sub-second queries on billions of rows** — Column-oriented format, bitmap indexes, and pre-aggregation enable P99 < 500 ms.

### Apache Pinot

Pinot was built at LinkedIn for user-facing analytics at massive scale (think: "Who viewed your profile" counts across 900M+ members).

- **Star-tree index** — Pinot's signature feature. A pre-materialized tree of aggregations across dimension combinations. Queries that hit the star-tree return in single-digit milliseconds regardless of data volume.
- **Upsert support** — Unlike Druid (append-only by default), Pinot supports row-level upserts keyed by primary key. This is critical for CDC-fed tables where you need the latest state, not a changelog.
- **User-facing analytics** — Pinot is optimized for the "1000 queries/sec, each touching a different user's data" pattern. Low tail latency is a first-class design goal.

### ClickHouse

ClickHouse is a column-oriented MPP database originally built at Yandex for web analytics (Yandex.Metrica processes billions of events daily).

- **MergeTree engine** — Data is stored in sorted "parts" that are periodically merged in the background (like LSM trees). This enables fast inserts and efficient range scans.
- **Materialized views** — ClickHouse materialized views act as insert triggers: when data lands in a source table, a transformation runs and results are inserted into a target table. This enables real-time pre-aggregation without a separate stream processor.
- **Strong SQL support** — ClickHouse supports a rich SQL dialect including window functions, arrays, maps, and JSON operations. Analysts can query it directly without learning a custom query language.
- **Log analytics sweet spot** — ClickHouse excels at log/trace/metric analytics where data is append-heavy, queries scan large time ranges, and approximate results (e.g., `uniqHLL`) are acceptable.

### Comparison Table

| Aspect | Druid | Pinot | ClickHouse |
|---|---|---|---|
| **Query Latency** | Sub-second (P99 < 500 ms) | Sub-second (star-tree: < 10 ms) | Sub-second to seconds |
| **Ingestion** | Kafka native (real-time nodes) | Kafka native (real-time servers) | Kafka via engine or connector |
| **SQL Support** | Druid SQL (limited joins) | Multi-stage query engine (joins) | Rich SQL dialect |
| **Upserts** | Limited (append-only preferred) | Native upsert support | ReplacingMergeTree (eventual) |
| **Best For** | Time-series dashboards, metrics | User-facing analytics, CDC | Log analytics, ad-hoc queries |
| **Managed Options** | Imply Cloud, AWS via marketplace | StarTree Cloud | ClickHouse Cloud |

### 💡 Interview Insight

> **Pick ONE of these and know it well.** Most interviewers care that you understand *why* you'd use a specialized OLAP DB over a general-purpose warehouse, not that you can recite specs for all three. A strong answer: "We'd use Pinot here because we need upserts from our CDC pipeline and sub-10 ms P99 for user-facing queries. Snowflake can't deliver that concurrency at that latency." That reasoning is worth more than memorizing the ClickHouse MergeTree variants.

---

## Screen 5: The Dual-Write Pattern & Lambda Considerations

Production real-time analytics systems almost never have a single output. They write to at least two destinations — and managing the consistency between them is one of the hardest problems in data engineering.

### Dual-Write Architecture

```
                        ┌──→ Real-time OLAP (Druid/Pinot) ──→ Live Dashboards
Kafka → Flink/Spark ────┤
                        └──→ Data Lake (S3/Delta Lake)    ──→ dbt → Warehouse → BI Tools
```

The same stream processor writes to both a real-time serving layer and a data lake. This is the **dual-write pattern** (sometimes called the "Kappa + batch" hybrid).

### Why Dual-Write?

Each destination serves a fundamentally different access pattern:

- **Real-time OLAP (Druid/Pinot)** — Serves operational dashboards with sub-second latency. Retains 7–30 days of data. Optimized for pre-defined queries (known dimensions, known metrics).
- **Data Lake (S3 + Delta Lake)** — Serves historical analytics, data science, ad-hoc exploration. Retains months or years. Optimized for full scans, joins across datasets, and ML feature engineering.

Trying to serve both patterns from a single system always results in painful compromises. The OLAP DB can't handle full table scans across 2 years of data. The data lake can't serve 1,000 concurrent dashboard queries at 200 ms P99.

### Consistency Challenges

When you write to two systems, they can (and will) diverge:

- **Eventual consistency** — The real-time path may process an event at T+1s while the lake path processes it at T+5min. Any query comparing the two during that window will see different numbers.
- **Failure modes** — The Flink job might successfully write to Druid but fail writing to S3 (or vice versa). Without transactional writes across both sinks, you get data in one but not the other.
- **Reconciliation strategies** — Run periodic (hourly/daily) reconciliation comparing counts between the OLAP DB and the lake. For critical metrics, the lake is the source of truth; the OLAP DB is the "fast but approximate" path.

### Backfill Scenarios

Eventually, you will need to reprocess historical data — a bug fix, a new aggregation, a schema change. Two approaches:

- **Kafka replay** — If Kafka retention covers the period, reset consumer offsets and replay. Simple but slow for large ranges.
- **Batch backfill** — Read historical data from the data lake, process with the same logic (Flink batch mode or Spark batch), and overwrite affected partitions. Faster, but you must ensure batch logic matches streaming logic exactly.

### Cost Considerations

Running two output paths costs more. Justify each by mapping it to business value: real-time OLAP prevents stock-outs and catches fraud (revenue impact); the data lake enables quarterly reviews, ML models, and regulatory reporting (analytical value). If the business only needs one path, don't build both.

### 💡 Interview Insight

> **Mentioning the dual-write pattern shows architectural maturity.** Explain that real-time serves *operational* needs (decisions in seconds) while the lake serves *analytical* needs (decisions in days). Then proactively address consistency: "We accept eventual consistency between the two paths and run a daily reconciliation job. For the first 5 minutes, the OLAP DB may show slightly different numbers than the lake, and we document this SLA for stakeholders." That level of pragmatism is exactly what senior interviewers look for.

---

## Screen 6: Designing a CDC Pipeline — End to End

Change Data Capture is how you get data out of transactional databases and into your analytics ecosystem without destroying the source database's performance or missing intermediate state changes.

### What Is CDC?

CDC captures row-level changes — every INSERT, UPDATE, and DELETE — from a source database's transaction log. Instead of periodically scanning the entire table (`SELECT * FROM orders WHERE updated_at > last_run`), CDC reads the database's internal changelog (WAL in Postgres, binlog in MySQL) and emits each change as a structured event.

### Why CDC Over Batch Extraction?

| Aspect | Batch Extraction (Query-Based) | CDC (Log-Based) |
|---|---|---|
| **Latency** | Minutes to hours (cron schedule) | Seconds (continuous streaming) |
| **Source DB Load** | Heavy (full table scans or indexed queries) | Minimal (reads transaction log) |
| **Missed Changes** | Yes — rapid UPDATE→UPDATE loses intermediate state | No — every change is captured |
| **Deletes** | Hard to detect (row is gone) | Captured as DELETE event |
| **Schema** | Whatever the current table looks like | Before/after snapshots per change |

### End-to-End Architecture

```
OLTP DB (Postgres) → Debezium → Kafka → Stream Processor (Flink/Spark) → Delta Lake (MERGE) → dbt → Warehouse → Dashboards
```

Let's trace a single record through this entire pipeline.

### Debezium Deep Dive

Debezium is the de facto open-source CDC platform. It runs as a Kafka Connect source connector:

1. **Initial snapshot** — On first startup, Debezium snapshots the existing table data, bootstrapping the Kafka topic with current state.
2. **Continuous streaming** — After snapshot, reads new changes from the transaction log (logical replication slots in Postgres, binlog in MySQL).
3. **Change event structure:**

```json
{
  "op": "u",
  "before": { "id": 42, "status": "pending", "amount": 99.99 },
  "after":  { "id": 42, "status": "shipped", "amount": 99.99 },
  "source": {
    "connector": "postgresql",
    "db": "orders_db",
    "table": "orders",
    "lsn": 234567890,
    "ts_ms": 1712345678000
  },
  "ts_ms": 1712345678050
}
```

- `op` — Operation type: `c` (create/insert), `u` (update), `d` (delete), `r` (read/snapshot).
- `before` — Row state before the change (null for inserts).
- `after` — Row state after the change (null for deletes).
- `source.lsn` — Log Sequence Number in Postgres; the exact position in the WAL. This is your ordering guarantee.

4. **Supported databases** — Postgres (logical replication), MySQL (binlog), MongoDB (oplog), SQL Server (CT/CDC tables), Oracle (LogMiner/XStream).

### Kafka Topic Design for CDC

- **One topic per table** — `cdc.orders_db.public.orders`, `cdc.orders_db.public.customers`. Naming convention: `<prefix>.<database>.<schema>.<table>`.
- **Partition by primary key** — Set the message key to the table's primary key. This guarantees all changes for a given row are processed in order by a single consumer partition.
- **Log compaction** — Enable log compaction on CDC topics. Kafka retains only the latest message per key, giving you a "snapshot" of the current state of every row. New consumers can bootstrap from the compacted topic without replaying the full history.

### MERGE into Delta Lake

The stream processor (Flink or Spark) consumes CDC events from Kafka and applies them to a Delta Lake table using the MERGE operation:

```sql
MERGE INTO orders_bronze AS target
USING cdc_changes AS changes
ON target.id = changes.id
WHEN MATCHED AND changes.op = 'd' THEN DELETE
WHEN MATCHED AND changes.ts_ms > target.ts_ms THEN UPDATE SET *
WHEN NOT MATCHED AND changes.op != 'd' THEN INSERT *
```

Key details:
- The `ts_ms > target.ts_ms` condition handles out-of-order events by ensuring only newer changes overwrite older ones (last-writer-wins with timestamp comparison).
- Deletes are handled explicitly via the `op = 'd'` condition.
- The Bronze layer contains a near-replica of the source table with minimal transformation.

### 💡 Interview Insight

> **Walk through the FULL pipeline end-to-end.** Interviewers love when you can trace a single record: "A customer updates their address in the app → Postgres writes the UPDATE to the WAL → Debezium reads the WAL entry and produces a change event with before/after snapshots → the event lands in Kafka topic `cdc.users_db.public.customers` partitioned by `customer_id` → Spark Structured Streaming consumes the event in a micro-batch → executes a MERGE INTO the Delta Lake `customers_bronze` table → dbt transforms it into `customers_gold` with business logic → the BI dashboard shows the updated address." This end-to-end fluency is the hallmark of a senior data engineer.

---

## Screen 7: CDC Challenges — Schema Evolution, Deletes, Late Events

Building the initial CDC pipeline is the easy part. Operating it for months and years — as source schemas change, as records are deleted, as events arrive late — is where the real engineering happens.

### Schema Evolution

Source databases evolve constantly. Product teams add columns, change types, rename fields. Your CDC pipeline must handle this gracefully.

**Types of schema changes and their impact:**

- **Adding a nullable column** — Backward compatible. Old events (without the column) can still be deserialized; the new field defaults to null. This is the safe path.
- **Removing a column** — Breaking change. Downstream consumers that reference the column will fail. Requires coordination: update consumers first, then remove the column.
- **Renaming a column** — Effectively a remove + add. The Schema Registry sees it as a new field. You need a mapping layer (in the stream processor or dbt) to bridge old and new names during the transition period.
- **Changing a column type** — Potentially breaking. `INT → BIGINT` is usually safe (widening). `STRING → INT` is not (narrowing). The Schema Registry will reject incompatible type changes under BACKWARD or FULL compatibility.

**Schema evolution strategy (multi-layer defense):**

1. **Schema Registry** — Enforce BACKWARD compatibility mode on all CDC topics. This prevents producers (Debezium) from registering schemas that would break existing consumers.
2. **Delta Lake** — Enable `mergeSchema = true` on write operations. Delta Lake automatically adds new columns to the table schema when they appear in incoming data.
3. **dbt** — In the transformation layer, new columns arrive as nullable. dbt models reference columns explicitly, so a new column in Bronze doesn't break Gold models until you intentionally add it.

### Handling Deletes

Deletes in CDC pipelines are deceptively complex:

- **Soft deletes (preferred)** — When a DELETE event arrives, set `is_deleted = true` and `deleted_at = timestamp` on the target row. The row remains in the table for historical analysis. Filter `WHERE is_deleted = false` in the Gold/serving layer.
- **Hard deletes** — Use the MERGE statement's DELETE clause to physically remove the row. Simpler, but you lose audit trail and can't answer "what was this customer's address before they deleted their account?"
- **Tombstone events in Kafka** — Debezium produces a tombstone (null value with the key) after a DELETE event. If log compaction is enabled, the tombstone eventually causes the key to be removed from the compacted topic, keeping the "snapshot" accurate.

**Recommendation:** Use soft deletes in your lakehouse (Bronze and Silver layers) and hard deletes only in the Gold/serving layer if business rules require it. This preserves lineage while keeping downstream tables clean.

### Late-Arriving Events

In distributed systems, events arrive out of order. A network partition, a slow consumer, or a backlogged Debezium connector can delay events by seconds, minutes, or even hours.

- **Watermarks** — Define how long the stream processor waits for late events before closing a window. A 10-minute watermark means you tolerate up to 10 minutes of delay. Events arriving later are dropped (Spark) or routed to a side output (Flink).
- **Reprocessing** — For events that arrive after the watermark, reprocess affected partitions in the data lake. A nightly "reconciliation" job compares source row counts to lake row counts and patches gaps.
- **Idempotent MERGE** — The MERGE statement with `ts_ms > target.ts_ms` is inherently idempotent. Processing the same event twice doesn't corrupt data — the second MERGE is a no-op because the timestamp hasn't advanced.

### Ordering Challenges

Two rapid updates to the same row (`status = 'pending'` → `status = 'shipped'` → `status = 'delivered'`) may arrive out of order if the stream processor has multiple partitions or retries. Solutions:

- **Partition by primary key** — Ensures all changes for a row go to the same partition and are processed in order.
- **Last-writer-wins with timestamp** — The MERGE condition `changes.ts_ms > target.ts_ms` ensures only the latest change is applied, regardless of processing order.
- **Version tracking** — If the source table has a monotonically increasing version column, use it instead of timestamp for even stronger ordering guarantees.

### 💡 Interview Insight

> **Schema evolution is the #1 operational challenge in CDC pipelines.** When discussing CDC in an interview, proactively mention: "We enforce BACKWARD compatibility in the Schema Registry to prevent breaking changes from propagating. For additive changes like new nullable columns, Delta Lake's `mergeSchema` handles it automatically. For breaking changes like column renames, we coordinate a migration: deploy the consumer update first, then the producer change, with a mapping layer in between." This shows you've dealt with real-world CDC operations, not just built a demo.

---

## Screen 8: Data Quality & Observability in Streaming Systems

A pipeline that produces wrong data on time is worse than one that produces right data late. Data quality is not an afterthought — it's a design requirement.

### Quality Checks at Every Layer

```
Ingestion Layer          Transformation Layer         Serving Layer
├─ Schema validation     ├─ Business rule checks      ├─ Freshness monitoring
├─ Null / type checks    ├─ Referential integrity     ├─ Completeness checks
├─ Deduplication         ├─ Range / distribution      ├─ Query latency SLAs
└─ Volume anomalies      └─ Cross-source consistency  └─ Anomaly detection
```

**Shift-left philosophy:** Catch bad data at ingestion before it propagates downstream. A malformed event caught at the Kafka consumer level is a log entry. The same bad data caught in a BI dashboard is a trust-destroying incident.

### Key Data Quality Metrics

| Metric | Definition | Example SLA |
|---|---|---|
| **Completeness** | % of expected records present | ≥ 99.5% of source rows in lake |
| **Freshness** | Time since most recent data point | Dashboard data ≤ 30 seconds old |
| **Accuracy** | Validation against source of truth | Aggregations match within 0.1% |
| **Volume** | Row count within expected range | Daily count within ±2σ of 30-day avg |
| **Uniqueness** | No duplicate primary keys | 0 duplicates in Gold tables |

### Data Quality Tools

**Great Expectations:** Define expectations as code (`expect_column_values_to_not_be_null("order_id")`), run checkpoints at pipeline stages, generate HTML data docs, and integrate with Airflow to halt pipelines on failure.

**Soda:** Data contracts in SodaCL (YAML-based): `row_count > 0`, `freshness(created_at) < 1h`. Monitoring dashboard tracks check results over time.

**dbt Tests:** Schema tests (`not_null`, `unique`, `accepted_values`, `relationships`), custom SQL tests (any query returning rows = failure), and freshness tests with warn/error thresholds.

### Data Lineage

When something breaks, you need two answers instantly: **root cause** (trace upstream) and **blast radius** (trace downstream).

- **Column-level lineage** — Track which source columns feed which targets through every transformation.
- **Impact analysis** — Before changing a column, see every downstream dependency. Tools: DataHub, OpenLineage (Marquez standard), dbt's built-in lineage graph.

### Alerting Strategy

1. **Threshold-based** — Row count drops below X, freshness exceeds Y. Deterministic, low false-positives. Route to Slack.
2. **Anomaly detection** — Statistical deviation from baseline (3σ below 30-day rolling average). Route to Slack with higher urgency.
3. **SLA breaches** — Pipeline hasn't completed by deadline. Route to PagerDuty.
4. **Escalation** — Slack (info) → Slack (warning) → PagerDuty (critical) → Incident channel.

### 💡 Interview Insight

> **Frame data quality as "shift-left testing" — the same concept from software engineering applied to data.** Mention specific, quantifiable metrics: "We monitor four dimensions — freshness (SLA: < 30 seconds for the real-time path, < 1 hour for the batch path), completeness (≥ 99.5% of source rows), volume (within 2σ of the 30-day mean), and uniqueness (zero duplicate primary keys in Gold). We use Great Expectations checkpoints at ingestion and dbt tests at transformation, with Slack alerts for warnings and PagerDuty for SLA breaches." Concrete numbers and tool names signal hands-on experience.

---

## Screen 9: Quiz

Test your understanding of real-time analytics and CDC pipeline concepts.

---

**Q1: You are designing a Kafka topic for a CDC pipeline from a `customers` table. Which partition key strategy ensures correct ordering of changes per customer while avoiding hot partitions?**

- A) Random partition assignment for even distribution
- B) Partition by `customer_id` (primary key) ✅
- C) Partition by `updated_at` timestamp
- D) Single partition for global ordering

> **Answer: B** — Partitioning by primary key (`customer_id`) ensures all changes for a given customer are sent to the same partition, preserving order (INSERT before UPDATE before DELETE). Random partitioning (A) breaks ordering. Timestamp partitioning (C) doesn't guarantee per-customer ordering. Single partition (D) preserves global order but eliminates all parallelism and creates a throughput bottleneck.

---

**Q2: Your team needs a stream processing engine for a fraud detection system that requires per-transaction scoring with < 50 ms latency and must maintain 500 GB of stateful feature aggregations. Which is the best choice?**

- A) Spark Structured Streaming with default micro-batch mode
- B) Apache Flink with RocksDB state backend ✅
- C) Kafka Streams embedded in microservices
- D) Spark Structured Streaming with continuous processing mode

> **Answer: B** — Flink with RocksDB handles sub-50 ms per-record processing and supports terabytes of state on local SSD. Spark micro-batch (A) has a latency floor of the trigger interval (typically 100 ms+). Kafka Streams (C) stores state in-memory/RocksDB but isn't designed for 500 GB of state across distributed operators. Spark continuous mode (D) is experimental and doesn't support aggregations.

---

**Q3: In a Delta Lake MERGE operation for CDC events, what does the condition `changes.ts_ms > target.ts_ms` in the WHEN MATCHED clause accomplish?**

- A) It filters out DELETE operations
- B) It enables schema evolution on the target table
- C) It ensures only newer changes overwrite existing rows, handling out-of-order events ✅
- D) It triggers compaction of the Delta Lake table

> **Answer: C** — The timestamp comparison implements a "last-writer-wins" strategy. If an older event arrives after a newer one has already been applied (out-of-order processing), the MERGE becomes a no-op because `changes.ts_ms` is not greater than `target.ts_ms`. This makes the MERGE idempotent and order-tolerant.

---

**Q4: Your CDC pipeline uses the Confluent Schema Registry with BACKWARD compatibility mode. A source team wants to add a new required (non-nullable) column to a Postgres table. What happens?**

- A) The Schema Registry accepts the new schema, and all consumers immediately see the new column
- B) The Schema Registry rejects the new schema because adding a required field is not backward compatible ✅
- C) Debezium automatically adds a default value for the new column
- D) The pipeline continues working but silently drops the new column

> **Answer: B** — BACKWARD compatibility means new schemas must be readable by consumers using the *previous* schema. A new required field without a default value means old consumers can't deserialize new messages (they don't know about the field and can't provide a value). The Schema Registry rejects the registration. The correct approach is to add the column as nullable with a default value, which is backward compatible.

---

**Q5: Which data quality metric would BEST detect a scenario where a CDC pipeline's Debezium connector silently stopped capturing changes from the source database?**

- A) Accuracy (validation against source values)
- B) Uniqueness (duplicate primary key detection)
- C) Freshness (time since most recent data point) ✅
- D) Schema conformance (column type validation)

> **Answer: C** — If Debezium stops producing events, no new data arrives in the pipeline. A freshness check (e.g., "most recent `ts_ms` in the target table is more than 5 minutes old") would immediately detect the stall. Accuracy (A) requires active comparison with the source and wouldn't fire until the next scheduled check. Uniqueness (B) and schema conformance (D) don't detect missing data at all.

---

## Screen 10: Key Takeaways

1. **Always clarify latency tiers first.** "Real-time" is ambiguous. Sub-100 ms (operational), seconds (dashboards), and minutes (near-real-time) each imply radically different architectures and cost profiles. Pin this down before drawing a single box.

2. **Kafka is the universal ingestion backbone.** Its durability, ordering, replayability, and decoupling properties make it the default choice for streaming architectures. Invest in partition key design — it determines both ordering correctness and throughput scalability.

3. **Choose Flink for low-latency, stateful streaming; Spark Structured Streaming for unified batch + stream workloads.** Flink's per-record model and RocksDB state backend handle use cases that Spark's micro-batch model cannot. But Spark's ecosystem maturity and batch/stream unification reduce operational overhead when seconds-level latency is acceptable.

4. **Specialized OLAP databases (Druid, Pinot, ClickHouse) exist because warehouses can't serve sub-second, high-concurrency queries.** Know one well and be able to articulate *why* it's needed — the interviewer cares about your reasoning, not your ability to recite three product specs.

5. **The dual-write pattern is an architectural necessity, not a hack.** Writing to both a real-time OLAP DB and a data lake serves fundamentally different access patterns. Accept eventual consistency between the two paths and implement reconciliation.

6. **CDC via Debezium reads the database transaction log, not the table itself.** This is why it's lower latency, lower load on the source, and captures every intermediate change including deletes. Be able to trace a single record change through the full pipeline end-to-end.

7. **Schema evolution is the #1 operational challenge in CDC.** Enforce compatibility modes in the Schema Registry, use Delta Lake's `mergeSchema` for additive changes, and coordinate migrations for breaking changes. This is what separates demo pipelines from production ones.

8. **Handle deletes with soft deletes in Bronze/Silver and hard deletes only in Gold.** This preserves audit trail and lineage while keeping downstream consumers clean. Use Kafka tombstones for topic compaction.

9. **Data quality is shift-left testing applied to data.** Check quality at ingestion (schema validation, volume anomalies), transformation (business rules, referential integrity), and serving (freshness SLAs, completeness). Use concrete metrics: freshness < 30s, completeness ≥ 99.5%, volume within 2σ.

10. **End-to-end traceability wins interviews.** The ability to trace a record from source database change → WAL → Debezium event → Kafka message → stream processor → Delta Lake MERGE → dbt model → dashboard demonstrates the fluency that distinguishes senior data engineers from those who've only read the documentation.
