# Designing a Data Lake / Lakehouse

> **Module 01 — System Design for Data Engineering Interview Prep**
>
> Master the end-to-end architecture of a modern data platform. This module walks you through every layer — from raw ingestion to governed serving — using the same mental framework top candidates use in system design interviews.

---

## Screen 1: Requirements Gathering for a Data Platform

Every great system design answer starts the same way: **not drawing boxes**. It starts with questions. When an interviewer says *"Design a data lake"* or *"Design a data platform for an e-commerce company,"* the worst thing you can do is jump straight to S3 + Spark + Airflow. The best thing you can do is spend the first 3–5 minutes clarifying scope.

### The Interview Framework

Use this four-step loop for any data platform design question:

1. **Clarify** — Ask about data sources, consumers, scale, latency, and constraints.
2. **High-Level Design** — Draw the major subsystems (ingestion, storage, processing, serving, governance).
3. **Deep Dive** — Pick 1–2 areas the interviewer cares about most and go deep.
4. **Tradeoffs** — Articulate what you chose, what you rejected, and why.

### Functional Requirements

Start by enumerating what the platform must *do*:

| Category | Examples |
|---|---|
| Data Sources | OLTP databases (Postgres, MySQL), APIs, application logs, clickstream events, flat files (CSV/Excel from vendors) |
| Data Consumers | BI analysts (dashboards), data scientists (notebooks, ML training), ML engineers (feature serving), operational teams (alerts, reports) |
| Query Patterns | Ad-hoc SQL exploration, scheduled dashboard refreshes, large-scale ML training reads, low-latency feature lookups |

### Non-Functional Requirements

These determine your architecture more than anything else:

| Dimension | Clarifying Question | Impact |
|---|---|---|
| **Scale** | How much data per day? How much total? | TB/day ingestion → streaming + distributed compute; PB total → object storage, not warehouse-only |
| **Latency** | Batch (hours), near-real-time (minutes), real-time (seconds)? | Drives choice of Spark batch vs Structured Streaming vs Flink |
| **Reliability** | What's the SLA for data freshness? Any data loss tolerance? | Exactly-once semantics, replication, dead letter queues |
| **Cost** | Fixed budget or elastic cloud spend? | Spot instances, storage tiering, auto-scaling clusters |
| **Compliance** | PII handling? GDPR/CCPA? HIPAA? | Encryption, column masking, audit logging, retention policies |

### Putting It Together: An Example Scoping Conversation

> **You:** *"Before I design anything — how many data sources are we talking about, and what's the daily data volume?"*
>
> **Interviewer:** *"About 50 source systems. Mix of Postgres databases, Kafka event streams, and third-party API feeds. Roughly 2 TB/day ingestion, growing 30% year-over-year. We have about 500 TB of historical data."*
>
> **You:** *"Got it. And who are the primary consumers? BI dashboards, ML models, or both?"*
>
> **Interviewer:** *"Mostly BI today — Tableau dashboards refreshed every 15 minutes. But the ML team is growing and wants a feature store."*

Now you know: moderate scale, near-real-time BI, batch ML, multi-consumer. That changes everything compared to a pure-batch or pure-streaming design.

### 💡 Interview Insight

> **Always start by asking about the Four V's — Volume, Velocity, Variety, and access patterns — before drawing any architecture.** Interviewers explicitly watch for this. A candidate who spends 3 minutes asking smart questions before touching the whiteboard signals seniority. A candidate who immediately draws Kafka → Spark → S3 signals they have one memorized architecture.

---

## Screen 2: Storage Layer — Object Storage + Table Formats

The storage layer is the gravitational center of any modern data platform. Every other decision — compute engines, query tools, governance — orbits around how and where you store data.

### Why Object Storage Is the Foundation

Cloud object storage (S3, GCS, Azure Data Lake Storage) won the storage wars for data lakes. Here's why:

| Property | Value |
|---|---|
| **Durability** | 99.999999999% (11 nines) — you'd lose 1 object per 10 million every 10,000 years |
| **Cost** | ~$0.023/GB/month (S3 Standard), ~$0.004/GB/month (Glacier) |
| **Scalability** | Virtually infinite — no capacity planning |
| **Decoupled Compute** | Any engine (Spark, Trino, Flink, DuckDB) reads from the same data |

The killer feature is **compute-storage separation**. In a traditional data warehouse, compute and storage are bundled — you pay for both even when your cluster is idle. With object storage, you spin up compute only when you need it and tear it down when you're done.

### The Table Format Revolution

Raw Parquet files on S3 get you 80% of the way there, but they can't give you:

- **ACID transactions** — what happens when two jobs write to the same table simultaneously?
- **Time travel** — can you query last Tuesday's version of a table?
- **Schema evolution** — can you add a column without rewriting every file?
- **Efficient upserts** — can you update 1,000 rows without rewriting a 100 GB partition?

This is exactly the gap that **Delta Lake**, **Apache Iceberg**, and **Apache Hudi** fill. They add a metadata layer on top of object storage that turns a pile of Parquet files into a proper table.

### Comparison: Delta Lake vs Iceberg vs Hudi

| Feature | Delta Lake | Apache Iceberg | Apache Hudi |
|---|---|---|---|
| ACID Transactions | ✅ Yes | ✅ Yes | ✅ Yes |
| Time Travel | ✅ Yes (version #) | ✅ Yes (snapshot ID/timestamp) | ✅ Yes (timeline) |
| Schema Evolution | ✅ Add/rename columns | ✅ Full (add, rename, drop, reorder) | ✅ Add/rename columns |
| Partition Evolution | ❌ Requires rewrite | ✅ Hidden partitioning, no rewrite | ❌ Requires rewrite |
| Engine Support | Spark, Trino, Flink (growing) | Spark, Trino, Flink, Dremio, Starrocks | Spark, Trino, Flink |
| Governance | Databricks Unity Catalog | Apache Polaris (open), AWS Glue | Limited |
| Community / Backing | Databricks (open source + commercial) | Netflix/Apple/LinkedIn → Apache Foundation | Uber → Apache Foundation |
| Best For | Databricks-centric shops | Multi-engine, vendor-neutral | CDC-heavy, record-level upserts |

### Delta Lake Deep Dive

Delta Lake stores a **transaction log** in a `_delta_log/` directory alongside your Parquet files. Every write operation (insert, update, delete, merge) creates a new JSON commit file. The log records which files were added and which were removed.

Key operations:
- **OPTIMIZE** — compacts small files into larger ones (target ~1 GB files for best read performance).
- **Z-ORDER** — co-locates related data within files for multi-dimensional filtering. If you frequently filter by `region` and `date`, Z-ordering on both clusters similar values together, reducing the number of files scanned.
- **VACUUM** — removes old files no longer referenced by the transaction log. Default retention is 7 days.

### Iceberg Deep Dive

Iceberg uses a tree of **metadata files → manifest lists → manifest files → data files**. This architecture enables:

- **Hidden partitioning** — you define partition transforms (e.g., `month(event_date)`) and Iceberg handles the rest. Consumers query `WHERE event_date = '2026-03-15'` without needing to know partitioning details.
- **Partition evolution** — change from monthly to daily partitioning *without rewriting any data*. Old data stays in monthly partitions; new data lands in daily ones. Iceberg's planning layer handles both seamlessly.
- **Snapshot isolation** — each query runs against a consistent snapshot, even if writes are happening concurrently.

### When to Choose Which

- **Delta Lake** if you're a Databricks shop or heavily invested in the Spark ecosystem. The integration is deepest and most mature.
- **Apache Iceberg** if you need multi-engine support (Spark + Trino + Flink + Dremio) or want vendor neutrality. The partition evolution story is best-in-class.
- **Apache Hudi** if your primary workload is CDC-heavy with frequent record-level upserts and you need Merge-on-Read for fast ingestion.

### 💡 Interview Insight

> **Mention that table formats solve the "small files problem" and enable ACID on object storage — this shows you understand WHY they exist, not just WHAT they are.** Bonus points: explain that the real innovation is the metadata layer (transaction log or manifest tree) that turns a collection of immutable files into a mutable, queryable table.

---

## Screen 3: Medallion Architecture (Bronze → Silver → Gold)

The medallion architecture is the most common organizational pattern for a modern lakehouse. Think of it as **progressive data refinement** — each layer transforms raw chaos into trusted, business-ready data.

### Bronze Layer — Raw Ingestion

The Bronze layer is your **system of record**. It stores data exactly as it arrived, warts and all.

| Property | Detail |
|---|---|
| Purpose | Preserve raw data with full fidelity |
| Schema | Schema-on-read — store as JSON, Avro, or raw Parquet without transformation |
| Write Pattern | Append-only — never update or delete |
| Partitioning | By ingestion date (`_ingested_date=2026-04-06`) |
| Retention | Long (years) — this is your reprocessing safety net |
| Metadata | Add `_source_system`, `_ingested_at`, `_batch_id` columns |

**Example:** Raw JSON events from Kafka land in `s3://datalake/bronze/clickstream/` partitioned by hour. Each record keeps the original payload, including malformed records and duplicates.

### Silver Layer — Cleaned & Conformed

The Silver layer is where data becomes **trustworthy**.

| Property | Detail |
|---|---|
| Purpose | Clean, deduplicate, conform, apply business keys |
| Schema | Schema-on-write — strong typing, enforced schema evolution |
| Write Pattern | Upsert/merge — idempotent, handles late-arriving data |
| Partitioning | By business key or event date |
| Transformations | Parse nested JSON, cast types, deduplicate by event ID, join reference data, handle SCD Type 2 |

**Example:** Bronze clickstream events are parsed into a structured `silver_clickstream` table. Duplicates are removed by `(event_id, event_timestamp)`. User IDs are resolved against a master customer table. Null `page_url` values are flagged and quarantined.

### Gold Layer — Business-Level Aggregates

The Gold layer is **consumption-ready**. It speaks the language of the business.

| Property | Detail |
|---|---|
| Purpose | Business-level aggregates, dimensional models, feature tables |
| Schema | Star schema, wide denormalized tables, or pre-aggregated metrics |
| Write Pattern | Overwrite partition or incremental aggregate |
| Optimized For | Dashboard queries, ML feature reads, API serving |

**Example:** `gold_daily_revenue` contains `date`, `product_category`, `region`, `total_revenue`, `order_count`, `avg_order_value`. Built from Silver order and product tables. Refreshed daily at 5 AM UTC.

### Data Flow Example (End to End)

```
Raw JSON events (Kafka)
    → Bronze: append raw events, partition by hour
        → Silver: parse, deduplicate, resolve user IDs, validate schema
            → Gold: aggregate daily revenue by product category and region
```

### Why Three Layers?

1. **Reprocessability** — if your Silver logic has a bug, reprocess from Bronze. No need to re-ingest from source systems.
2. **Debugging** — compare Bronze (raw) to Silver (cleaned) to trace data quality issues.
3. **Separation of concerns** — Bronze owners care about ingestion reliability; Gold owners care about business logic correctness.
4. **Different SLAs** — Bronze can lag by an hour; Gold dashboards must be fresh by 6 AM.

### Anti-Patterns to Avoid

- **Too many layers (5+):** Platinum, Diamond, Uranium — each extra layer adds latency, complexity, and confusion. Three is the sweet spot.
- **Skipping Silver:** Going Bronze → Gold directly means your Gold tables embed parsing, cleaning, and aggregation logic. When something breaks, you can't tell if it's a data quality issue or a business logic issue.
- **Gold that's just Silver renamed:** If your Gold layer is just `SELECT * FROM silver_table`, you're not adding value. Gold should reshape data for specific consumption patterns — star schemas for BI, feature vectors for ML.

### 💡 Interview Insight

> **Explain medallion as "progressive data refinement" — each layer adds value and each can be independently reprocessed from the layer below.** This framing shows you understand the operational benefits (reprocessing, debugging) rather than just the conceptual labels.

---

## Screen 4: Ingestion — Batch + Streaming

Ingestion is how data enters your platform. Getting it wrong creates problems that cascade through every downstream layer.

### Batch Ingestion Patterns

| Pattern | How It Works | When to Use |
|---|---|---|
| **Full Load** | Extract entire table every run | Small reference tables (<1 GB), source has no reliable change tracking |
| **Incremental — CDC** | Capture INSERT/UPDATE/DELETE from DB transaction log | Large transactional tables, need deletes, near-real-time |
| **Incremental — Watermark** | `WHERE updated_at > last_run_timestamp` | Source has reliable `updated_at` column, batch is acceptable |
| **Incremental — Audit Columns** | `WHERE batch_id > last_batch_id` | Source system provides batch markers |
| **File-Based** | Poll S3 landing zone or SFTP for new files | Third-party vendor data drops, CSV/Excel feeds |
| **Database Dump** | `pg_dump`, `mysqldump` → object storage | Migration, disaster recovery, dev environment seeding |

### Streaming Ingestion

For data that can't wait:

- **Kafka** — the industry standard. Producers publish events to topics; consumers (Spark, Flink) read from offsets. Kafka retains events for configurable periods (7 days default, infinite with tiered storage).
- **Change Data Capture (Debezium)** — reads the database's WAL (write-ahead log) and publishes row-level changes to Kafka. No polling, no `updated_at` column needed, captures deletes.
- **Kinesis / Pub/Sub** — managed streaming services from AWS/GCP. Less operational burden than self-managed Kafka, but less ecosystem support.

### Lambda vs Kappa Architecture

These are the two canonical architectures for systems that need both batch and streaming.

**Lambda Architecture:**

```
                    ┌─── Batch Layer (Spark) ──── Batch Views ───┐
Source ─── Queue ───┤                                            ├─── Serving Layer
                    └─── Speed Layer (Flink) ──── Real-time Views┘
```

The batch layer recomputes everything periodically for correctness. The speed layer provides low-latency approximate results. The serving layer merges both.

**Kappa Architecture:**

```
Source ─── Stream Log (Kafka) ─── Stream Processor (Flink) ─── Serving Layer
```

Everything is a stream. Reprocessing means replaying from the log with updated logic. No separate batch layer.

| Dimension | Lambda | Kappa |
|---|---|---|
| Complexity | High — two codebases (batch + stream) | Lower — single codebase |
| Latency | Speed layer: seconds; Batch: hours | Seconds to minutes |
| Reprocessing | Re-run batch job | Replay from Kafka log |
| Consistency | Eventual — batch corrects speed layer | Strong — single source of truth |
| Best For | Mixed latency needs, team has batch expertise | Event-driven, team has streaming expertise |
| Risk | Code drift between batch and speed layers | Kafka retention cost, complex stateful processing |

### Ingestion Best Practices

- **Idempotent writes** — re-running an ingestion job should not create duplicates. Use `MERGE` / upsert or deduplicate on primary key + timestamp.
- **Schema registry** — use Confluent Schema Registry or AWS Glue Schema Registry to enforce Avro/Protobuf schemas on producers. Catch breaking changes before they poison your lake.
- **Dead letter queues (DLQ)** — route un-parseable or schema-violating events to a DLQ topic. Don't let one bad record kill your pipeline.
- **Exactly-once delivery** — use Kafka transactions + Flink checkpointing or Spark's idempotent sink for end-to-end exactly-once.

### 💡 Interview Insight

> **Most companies use a hybrid approach — streaming for time-sensitive data (clickstream, fraud detection), batch for historical backfills and slowly-changing reference data. Don't be dogmatic about Lambda vs Kappa.** In an interview, say: *"I'd use Kappa for our event-driven core and layer in batch ingestion for vendor data drops that only arrive daily."*

---

## Screen 5: Processing — Spark, Flink, and Transformation

Processing is where raw data becomes valuable. This is often the deepest part of a system design interview because it touches performance, correctness, and cost.

### Batch Processing with Apache Spark

Spark is the workhorse of batch data processing. Key performance levers:

- **Partitioned reads** — read only the partitions you need. `WHERE event_date = '2026-04-06'` should scan one partition, not the entire table. Table formats (Delta/Iceberg) handle this via metadata pruning.
- **Broadcast joins** — when joining a large fact table (billions of rows) to a small dimension table (<100 MB), broadcast the small table to all executors. Eliminates shuffle.
- **Adaptive Query Execution (AQE)** — Spark 3.x feature that dynamically adjusts shuffle partitions, converts sort-merge joins to broadcast joins at runtime, and optimizes skewed joins. Enable it: `spark.sql.adaptive.enabled = true`.
- **Dynamic partition pruning** — in star schema joins, Spark prunes fact table partitions based on dimension filter results. Reduces I/O by 10–100x on well-modeled data.

### Stream Processing: Spark Structured Streaming vs Apache Flink

| Dimension | Spark Structured Streaming | Apache Flink |
|---|---|---|
| Processing Model | Micro-batch (default ~100ms trigger), continuous mode (experimental) | True record-at-a-time, event-driven |
| Latency | Seconds (micro-batch) | Milliseconds |
| State Management | State stored in checkpoint (HDFS/S3); limited queryable state | RocksDB-backed state; queryable state, incremental checkpoints |
| Windowing | Tumbling, sliding, session (via watermarks) | Tumbling, sliding, session, custom, processing-time and event-time |
| Exactly-Once | ✅ End-to-end with idempotent sinks | ✅ End-to-end with two-phase commit (Kafka, JDBC) |
| Ecosystem | Deeply integrated with Spark SQL, MLlib, Delta Lake | Growing; Flink SQL, CDC connectors, Iceberg sink |
| Best For | Unified batch + streaming on Spark; moderate latency OK | True low-latency, complex event processing, large stateful workloads |

### Transformation with dbt

dbt (data build tool) has become the standard for SQL-based transformations in the Gold layer (and increasingly Silver). It brings **software engineering practices** to data transformation:

- **Modularity** — break transforms into composable models (`stg_orders.sql → int_orders_joined.sql → fct_daily_revenue.sql`).
- **Testing** — built-in tests for `not_null`, `unique`, `accepted_values`, `relationships` (referential integrity). Custom tests for business rules.
- **Documentation** — auto-generated data docs with column descriptions and lineage graphs.
- **Incremental models** — process only new/changed data: `WHERE event_date > (SELECT MAX(event_date) FROM {{ this }})`.
- **Snapshots** — SCD Type 2 tracking with `dbt snapshot`. Automatically adds `dbt_valid_from` and `dbt_valid_to` columns.

dbt works in the lakehouse via adapters like `dbt-spark`, `dbt-databricks`, or `dbt-trino`.

### Processing Optimization Checklist

1. **Partition pruning** — ensure queries filter on partition columns; verify via `EXPLAIN` plans.
2. **Predicate pushdown** — columnar formats (Parquet, ORC) push filters into the scan. Don't read everything and filter in memory.
3. **Columnar formats** — Parquet is the default. Use `ZSTD` compression for best ratio-to-speed balance.
4. **File sizing** — target 128 MB–1 GB per file. Too small = excessive metadata overhead; too large = poor parallelism.
5. **Caching hot tables** — `spark.catalog.cacheTable()` or Delta caching for frequently-accessed dimension tables.
6. **Skew handling** — salted keys, AQE skew join optimization, or pre-aggregation to handle hot keys.

### 💡 Interview Insight

> **When discussing processing, mention both the WHAT (Spark/Flink/dbt) and the HOW (partitioning strategy, join optimization, incremental processing).** Saying *"I'd use Spark"* is a C-level answer. Saying *"I'd use Spark with AQE enabled, broadcast the 50 MB dimension table, partition the fact table by day, and run incremental processing with a high watermark"* is an A-level answer.

---

## Screen 6: Orchestration & Serving Layer

You've built ingestion, storage, and processing. Now: how do you make it all run reliably on schedule, and how do consumers actually access the data?

### Orchestration with Apache Airflow

Airflow is the most widely-adopted workflow orchestrator. Core concepts:

- **DAG (Directed Acyclic Graph)** — defines a pipeline as a set of tasks with dependencies.
- **Operators** — units of work: `SparkSubmitOperator`, `KubernetesPodOperator`, `BigQueryInsertJobOperator`, `PythonOperator`.
- **Sensors** — wait for a condition: `S3KeySensor` (file lands), `ExternalTaskSensor` (upstream DAG completes).
- **Retries + SLAs** — configure `retries=3`, `retry_delay=timedelta(minutes=5)`, `sla=timedelta(hours=2)`.
- **Pools** — limit concurrency: only 5 Spark jobs at once to avoid overwhelming the cluster.

### Airflow Patterns for Data Platforms

- **Task groups** — organize related tasks visually (e.g., `ingest_postgres_sources` group with 20 tasks).
- **Dynamic DAGs** — generate one task per source table from a config file or database. Don't write 50 nearly-identical DAGs.
- **XComs** — pass metadata between tasks: row counts, file paths, partition values. Keep payloads small (<48 KB).
- **Idempotent tasks** — every task should be safe to re-run. Use `MERGE` instead of `INSERT`, overwrite partitions instead of appending.

### Orchestrator Alternatives

| Tool | Philosophy | Differentiator |
|---|---|---|
| **Dagster** | Software-defined assets | Define what you want to exist (assets), not what to run (tasks). Built-in data lineage and type checking. |
| **Prefect** | Dynamic, code-first workflows | No DAG limitations; dynamic task graphs at runtime. Hybrid execution model. |
| **Mage** | Modern DE tool | Notebook-style UI, built-in data quality, easier learning curve than Airflow. |

### Serving Layer — Where Consumers Meet Data

This is the part most candidates forget. You need to **match the serving technology to the access pattern**:

**For BI / SQL Analytics:**

A cloud data warehouse (Snowflake, BigQuery, Redshift) or a lakehouse query engine (Databricks SQL, Trino, Starburst).

- **Why not just query the lake directly?** You *can*, but warehouses add: columnar caching, MPP query optimization, result caching, concurrency scaling, and fine-grained access control. For dashboards with 200 concurrent users, a warehouse serving layer avoids hammering your lake.
- **Pattern:** Gold layer tables are synced (or directly queried) from the warehouse. dbt builds star schemas in the warehouse.

**For ML / Feature Serving:**

A feature store (Feast, Tecton, Databricks Feature Store):

- **Offline store** — Gold-layer feature tables in Parquet/Delta for batch training. Features computed by Spark/dbt.
- **Online store** — low-latency key-value store (Redis, DynamoDB) for real-time inference. Features materialized from offline store.
- **Feature registry** — central catalog of feature definitions, owners, freshness SLAs, and lineage.

**For APIs / Real-Time Lookups:**

- **Materialized views** in Postgres or a warehouse for medium-latency (~100ms) queries.
- **Redis / DynamoDB** for <10ms lookups by key (user profile, recommendation scores, fraud signals).
- **Reverse ETL** (Census, Hightouch) — sync Gold data back into SaaS tools (Salesforce, HubSpot, Braze).

### 💡 Interview Insight

> **Don't forget the serving layer — many candidates design beautiful pipelines but forget to explain how downstream consumers actually ACCESS the data.** When you reach this part of the design, say: *"Now let me talk about how the three consumer personas — BI analysts, ML engineers, and application developers — would each access this data differently."*

---

## Screen 7: Governance, Monitoring & Multi-Tenancy

This is the section that separates senior candidates from everyone else. A data lake without governance is a data swamp. A data platform without monitoring is a liability.

### Data Governance

**Data Catalog:**

A catalog makes data discoverable. Without it, analysts waste hours asking *"Where is the revenue table?"* or *"What does this column mean?"*

| Tool | Key Feature |
|---|---|
| **Databricks Unity Catalog** | Unified governance for Delta Lake: access control, lineage, audit logs, data sharing |
| **DataHub** (LinkedIn) | Open-source metadata platform with push/pull ingestion, search, lineage, data quality integration |
| **Apache Amundsen** (Lyft) | Discovery-focused: search, table/column descriptions, owners, usage stats |
| **OpenMetadata** | Open-source, built-in data quality, lineage, policies, glossary |

**Data Lineage:**

Lineage answers: *"Where did this data come from, and what downstream tables does it feed?"*

- **OpenLineage** — open standard for lineage metadata. Emits events from Spark, Airflow, dbt.
- **Marquez** — lineage server that stores OpenLineage events and serves a lineage graph API.
- **Use case:** A bug is found in `gold_daily_revenue`. Lineage shows it depends on `silver_orders` → `bronze_orders_raw` → Postgres `orders` table. You trace the bug to a schema change in Postgres.

**Access Control:**

- **RBAC** — roles like `data_analyst`, `data_engineer`, `data_scientist` with different table/schema permissions.
- **Column-level security** — mask PII columns (`email`, `ssn`) for analysts but allow data engineers to see them for debugging.
- **Row-level filtering** — regional managers see only their region's data.

### Data Quality Monitoring

Bad data is worse than no data — because people make decisions on it.

| Tool | Approach |
|---|---|
| **Great Expectations** | Define "expectations" in code: `expect_column_values_to_not_be_null("order_id")`, `expect_column_values_to_be_between("price", 0, 100000)`. Run as pipeline validation step. |
| **Soda** | Data contracts as YAML. `checks for orders: row_count > 0`, `freshness < 2h`. SodaCL language. |
| **dbt tests** | Built into your transformation layer: `unique`, `not_null`, `accepted_values`, `relationships`, plus custom SQL tests. |
| **Monte Carlo / Bigeye** | ML-powered anomaly detection: automatically learns normal patterns and alerts on deviation. Commercial. |

### Pipeline Monitoring: SLIs, SLOs, and Alerting

Define concrete, measurable indicators:

- **SLI: Freshness** — when was the table last updated? Measure: `MAX(updated_at)` vs `NOW()`.
- **SLI: Completeness** — is today's partition complete? Measure: today's row count vs 7-day average. Alert if <80%.
- **SLI: Volume** — did we ingest the expected number of records? Measure: actual vs expected (± 2 standard deviations).
- **SLO example:** *"The `gold_daily_revenue` table is refreshed by 6:00 AM UTC, with >99.5% completeness, 99% of the time."*

**Alerting channels:** PagerDuty for critical SLO breaches, Slack for warnings, email for daily summaries. Use both **threshold-based** (row count < 1000) and **anomaly-based** (3σ deviation from trailing 30-day average) alerts.

### Multi-Tenant Data Platform

When your data platform serves multiple teams (or multiple business units):

| Concern | Strategy |
|---|---|
| **Isolation** | Separate schemas per team (light isolation) vs separate cloud accounts per BU (strong isolation). Most companies start with schemas and graduate to accounts for regulated data. |
| **RBAC with Inheritance** | Platform roles → team roles → individual grants. `team_marketing.analyst` inherits `platform.reader` permissions. |
| **Cost Allocation** | Tag every resource (Spark cluster, storage bucket, query) with `team`, `project`, `cost_center`. Build chargeback dashboards showing compute and storage cost per team per month. |
| **Self-Service** | Governed data catalog with access request workflows: analyst finds a table → clicks "Request Access" → team owner approves → RBAC grant is automated. Reduces bottleneck on the platform team. |

### 💡 Interview Insight

> **Governance and monitoring are what separate a senior answer from a mid-level one.** After you finish the core architecture, say: *"Now let me layer on governance. I'd deploy a data catalog like DataHub for discovery, OpenLineage for end-to-end lineage tracking, Great Expectations as a quality gate between Bronze and Silver, and dbt tests between Silver and Gold. Each Gold table would have an SLO — for example, daily revenue ready by 6 AM with 99.5% completeness."* This signals operational maturity.

---

## Screen 8: Quiz — Test Your Data Platform Design Knowledge

Test your understanding of the concepts covered in this module.

---

**Q1: In the medallion architecture, which layer should you reprocess from if you discover a bug in your data cleaning logic?**
- A) Reprocess Gold from the serving layer
- B) Reprocess Silver from Bronze ✅
- C) Re-ingest from source systems
- D) Reprocess Gold from Silver

> **Answer: B** — Silver contains the cleaning/deduplication/conforming logic. If that logic has a bug, you reprocess Silver from Bronze (which stores raw data with full fidelity). You don't need to re-ingest from source systems because Bronze is your system of record. Reprocessing Gold from Silver would perpetuate the bug since Silver itself is wrong.

---

**Q2: Which table format is the best choice for a company that uses Spark for batch processing, Trino for interactive queries, and Flink for stream processing — and wants to avoid vendor lock-in?**
- A) Delta Lake
- B) Apache Hudi
- C) Apache Iceberg ✅
- D) Raw Parquet files

> **Answer: C** — Apache Iceberg has the broadest multi-engine support (Spark, Trino, Flink, Dremio, Starrocks) and is governed by the Apache Foundation, making it the most vendor-neutral choice. Delta Lake is excellent but most deeply integrated with Databricks. Raw Parquet files lack ACID transactions, time travel, and schema evolution.

---

**Q3: A streaming pipeline sometimes produces duplicate records when a Kafka consumer restarts after a failure. Which combination of techniques best prevents duplicates in the Silver layer?**
- A) Increase Kafka retention period and add more partitions
- B) Use Kafka transactions, idempotent producers, and MERGE/upsert on the sink ✅
- C) Switch from Kafka to a batch ingestion approach
- D) Add a deduplication step in the Gold layer only

> **Answer: B** — End-to-end exactly-once requires: (1) idempotent producers to prevent duplicate publishes, (2) Kafka transactions to atomically commit consumer offsets with processing, and (3) an idempotent sink (MERGE/upsert by primary key) so re-processing the same records doesn't create duplicates. Deduplicating only in Gold means Silver is still dirty, violating medallion principles.

---

**Q4: Which SLI (Service Level Indicator) would best detect if a daily batch pipeline silently processed zero records due to an empty source table?**
- A) Freshness — time since last table update
- B) Latency — how long the pipeline took to run
- C) Volume — row count compared to expected baseline ✅
- D) Schema — number of columns in the output table

> **Answer: C** — A volume check compares today's row count against a historical baseline (e.g., trailing 7-day average). If the pipeline runs successfully but processes zero records, freshness would still show the table was "updated" (with zero rows), and latency might even be shorter than usual. Only a volume anomaly check catches this silent failure.

---

**Q5: In a multi-tenant data platform, what is the most scalable way to manage data access permissions across 50+ teams?**
- A) Create individual user grants for every table
- B) Use RBAC with role inheritance: platform roles → team roles → individual grants ✅
- C) Give all teams admin access and rely on trust
- D) Create separate data lakes for each team

> **Answer: B** — RBAC with inheritance is the standard approach. Platform-level roles (e.g., `platform.reader`) define baseline permissions, team roles inherit and extend them (e.g., `team_marketing.analyst` adds access to marketing schemas), and individual grants handle exceptions. Individual user grants (A) don't scale to 50+ teams. Separate data lakes (D) eliminates cross-team data sharing. Admin access for everyone (C) is a security nightmare.

---

## Screen 9: Key Takeaways

1. **Always start with requirements.** Spend the first 3–5 minutes of any data platform design interview clarifying volume, velocity, variety, consumers, and latency needs. This shapes every architectural decision downstream.

2. **Object storage is the foundation.** S3/GCS/ADLS gives you 11-nines durability, infinite scale, and compute-storage separation at ~$0.023/GB/month. Build everything on top of it.

3. **Table formats are non-negotiable.** Delta Lake, Iceberg, or Hudi add ACID transactions, time travel, and schema evolution to your lake. Choose Iceberg for multi-engine neutrality, Delta for Databricks shops, Hudi for CDC-heavy workloads.

4. **Medallion architecture provides progressive refinement.** Bronze (raw, append-only) → Silver (clean, deduplicated, conformed) → Gold (business aggregates, star schemas, feature tables). Each layer can be independently reprocessed from the one below.

5. **Match ingestion to the use case.** Streaming (Kafka + CDC) for time-sensitive data; batch (incremental with watermarks) for historical loads. Most real-world platforms are hybrid — don't be dogmatic about Lambda vs Kappa.

6. **Processing is about the HOW, not just the WHAT.** Mention specific optimizations: broadcast joins for small dimensions, AQE for adaptive execution, partition pruning for I/O reduction, and incremental processing with high watermarks.

7. **The serving layer must match the access pattern.** Warehouses for BI (concurrency, caching), feature stores for ML (offline + online), Redis/DynamoDB for real-time API lookups. One size does not fit all consumers.

8. **Governance is what makes you senior.** Catalog (DataHub / Unity Catalog) for discovery, lineage (OpenLineage) for traceability, access control (RBAC + column masking) for security, and data quality (Great Expectations / dbt tests) for trust.

9. **Define SLOs, not just SLIs.** Don't just monitor freshness — commit to *"Gold revenue table ready by 6 AM UTC with >99.5% completeness, 99% of the time."* This creates accountability and drives alerting thresholds.

10. **Design for reprocessability.** Every layer, every pipeline, every table should be safe to reprocess. Idempotent writes, append-only Bronze, MERGE-based Silver, and partition-overwrite Gold make your platform resilient to bugs, late-arriving data, and schema changes.
