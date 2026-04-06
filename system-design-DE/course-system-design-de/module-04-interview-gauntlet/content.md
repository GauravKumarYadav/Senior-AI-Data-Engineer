# The Interview Gauntlet 🔥

> **Module 04 — System Design for Data Engineering Interview Prep**
>
> This is the final module. No more hand-holding. No more theory first. This is pure interview combat — system design questions, trade-off debates, failure scenarios, and rapid-fire rounds. You've built the foundation in Modules 1–3. Now prove you can wield it under pressure. Welcome to the Gauntlet.

---

## Screen 1: Architecture Design — End-to-End System Design Questions

These eight questions are the kind you'll face in a 45–60 minute system design round. For each one, we present a structured answer following the framework: **Requirements → Architecture → Data Model → Deep Dive → Tradeoffs**. Study the *shape* of the answers — not just the content. Interviewers want to see your thinking process, not a memorized diagram.

---

### Q1 🔴 HARD — "Design a real-time fraud detection system for a payment processor handling 50K transactions/sec"

<details>
<summary><strong>Question → Answer</strong></summary>

**Requirements Clarification:**
- 50,000 transactions/second peak, ~2B transactions/day.
- End-to-end latency budget: **<200ms** from transaction received to approve/decline decision.
- Must compute real-time features (velocity, geo-distance, amount deviation) and score against an ML model.
- False positive rate must be <1% — blocking legitimate transactions costs revenue.
- Historical feature lookups needed (user's average spend over 30 days, typical login locations).

**High-Level Architecture:**

```
Payment Gateway
    → Kafka (transactions topic, 128 partitions, keyed by card_id)
        → Flink Stream Processor
            ├── Feature Computation (velocity, geo, amount z-score)
            ├── Feature Store Lookup (Redis — 30-day user profile)
            ├── ML Model Scoring (embedded ONNX model in Flink)
            └── Decision: approve / decline / review
                ├── approve → response back to gateway (<200ms)
                ├── decline → response + alert to fraud ops
                └── review → case management queue (Kafka → Postgres)
```

**Data Model:**

- **Transaction Event:** `{txn_id, card_id, merchant_id, amount, currency, geo_lat, geo_lon, timestamp, channel, device_fingerprint}`
- **User Profile (Redis):** `card_id → {avg_30d_amount, txn_count_24h, last_5_locations[], typical_merchants[], risk_score_baseline}`
- **Feature Vector (per txn):** `{velocity_1h, velocity_24h, geo_distance_from_last, amount_zscore, is_new_merchant, device_trust_score, time_since_last_txn}`

**Deep Dive — Feature Computation in Flink:**

1. **Velocity features** — Flink keyed state per `card_id`. Tumbling windows (1h, 24h) count transactions. Stored in RocksDB-backed state with incremental checkpoints every 30 seconds.
2. **Geo-distance** — compare current `(lat, lon)` to last known location from Redis. Haversine distance. Flag if >500km in <2 hours ("impossible travel").
3. **Amount z-score** — `(current_amount - avg_30d) / stddev_30d` from Redis profile. Flag if >3σ.
4. **Model scoring** — ONNX Runtime embedded in Flink operator. Model loaded at startup, hot-reloaded on version change (feature flag). Inference <5ms per transaction.

**Latency Budget Breakdown:**
| Stage | Budget |
|---|---|
| Kafka produce + consume | 10ms |
| Feature computation (Flink) | 20ms |
| Redis feature store lookup | 5ms |
| ML model inference | 5ms |
| Decision + response | 10ms |
| **Total** | **~50ms typical, <200ms p99** |

**Tradeoffs:**
- **Embedded model vs external model service:** Embedded (ONNX in Flink) eliminates network hop, keeping latency <5ms. External model service adds 20–50ms per call but allows independent scaling and A/B testing of models. At 50K TPS, the latency win of embedded is worth the deployment coupling.
- **Flink stateful processing vs external state store only:** Flink keyed state for short-window features (1h, 24h velocity) avoids a Redis round-trip. Longer-window features (30-day profile) live in Redis because Flink state for 30 days × 100M cards would be enormous.
- **False positive tuning:** A secondary "review" path prevents hard-blocking borderline cases. Human reviewers handle ~2% of volume, keeping automation rate >98%.

</details>

### 💡 Interview Insight

> **Always state your latency budget upfront and break it down by component.** This shows you understand that system design isn't about drawing boxes — it's about making each box fast enough to hit the end-to-end SLA. Interviewers love seeing a latency budget table.

---

### Q2 🟡 SENIOR — "Design a data pipeline for a social media feed (Twitter/Instagram scale)"

<details>
<summary><strong>Question → Answer</strong></summary>

**Requirements:**
- 500M monthly active users, 10K new posts/sec, 300K feed reads/sec.
- Feed must show posts from followed accounts, ranked by relevance, within 2 seconds.
- Events: posts, likes, follows, shares, comments — all need to be captured for analytics.

**Architecture — Fan-Out Decision:**

The critical design choice is **fan-out-on-write vs fan-out-on-read:**

| Approach | How It Works | Pros | Cons |
|---|---|---|---|
| **Fan-out-on-write** | When user A posts, push to every follower's timeline cache | Read is O(1) — just fetch from cache | Celebrity problem: user with 50M followers = 50M writes per post |
| **Fan-out-on-read** | When user B opens feed, query all followed users' posts at read time | Write is O(1) — just store the post | Read is O(N) where N = following count; slow for heavy followers |
| **Hybrid (Twitter's approach)** | Fan-out-on-write for regular users (<10K followers); fan-out-on-read for celebrities (>10K followers). Merge at read time. | Balances write amplification and read latency | More complex; two code paths |

**Data Flow:**

```
Post Created → Kafka (posts topic)
    → Fan-out Service (checks follower count)
        ├── <10K followers: write to Redis timeline caches (sorted set per user)
        └── >10K followers: store in Celebrity Posts table
    → Analytics Pipeline (Kafka → Flink → Data Lake)

Feed Read → Timeline Service
    ├── Fetch user's Redis timeline cache (pre-computed)
    ├── Fetch celebrity posts (query Celebrity Posts table for followed celebrities)
    ├── Merge + Rank (lightweight ML ranker: engagement prediction)
    └── Return top 50 posts
```

**Data Model:**
- **Post:** `{post_id, author_id, content, media_urls[], created_at, type}`
- **Timeline Cache (Redis Sorted Set):** key = `timeline:{user_id}`, members = `post_id`, score = `timestamp` (or ranking score). Keep last 800 posts per user.
- **Follow Graph:** `{follower_id, followee_id, created_at}` — stored in a graph database or sharded Postgres.
- **Analytics Events (Kafka → Lake):** `{event_type, user_id, post_id, timestamp, metadata}`

**Tradeoffs:**
- Hybrid fan-out adds complexity but is necessary at scale — pure fan-out-on-write would mean a single celebrity post triggers 50M Redis writes.
- Redis timeline caches consume massive memory: 500M users × 800 posts × 8 bytes/post_id ≈ 3.2 TB of Redis. Use Redis Cluster with consistent hashing.
- Ranking model runs at read time but must be lightweight (<50ms). Heavy ML features are pre-computed offline and stored as user-post affinity scores.

</details>

---

### Q3 🔴 HARD — "Design a real-time inventory tracking system for Walmart (4,700 stores, millions of SKUs)"

<details>
<summary><strong>Question → Answer</strong></summary>

**Requirements:**
- 4,700 stores, ~15M unique SKUs, average ~100K SKUs per store.
- Events: POS sales, returns, receiving (truck deliveries), adjustments (shrinkage, damages), store-to-store transfers.
- Real-time inventory count per store/SKU with <30 second latency.
- Trigger automatic replenishment orders when inventory drops below threshold.
- Handle: Black Friday spikes (10x normal volume), network outages at stores.

**Architecture:**

```
Store POS / Receiving / Adjustments
    → Store Edge Gateway (local buffer if network down)
        → Kafka (inventory-events topic, partitioned by store_id)
            → Flink (keyed by store_id + sku_id)
                ├── Aggregate: running inventory count per store/SKU
                ├── State Store (RocksDB): current_qty, last_updated, pending_transfers
                ├── Replenishment Check: if qty < reorder_point → emit replenishment event
                └── Sink: write to Inventory State DB (Cassandra, partitioned by store_id)
                    → Serving API (read from Cassandra)
                    → Analytics Pipeline (Kafka → S3 → Spark → Gold tables)
```

**Data Model:**

- **Inventory Event:** `{event_id, store_id, sku_id, event_type (sale|return|receive|adjust|transfer), quantity_delta, timestamp, source_system}`
- **Inventory State (Cassandra):** `PRIMARY KEY ((store_id), sku_id)` → `{current_qty, last_updated, reorder_point, reorder_qty, pending_transfer_qty}`
- **Replenishment Order:** `{order_id, store_id, sku_id, order_qty, triggered_at, status}`

**Deep Dive — Edge Buffering:**
Stores lose network connectivity. The edge gateway (lightweight service at each store) buffers events locally (SQLite or embedded Kafka) and replays them when connectivity returns. Events are idempotent — keyed by `event_id`, so replaying doesn't double-count. Flink deduplicates on `event_id` using a keyed state TTL window.

**Tradeoffs:**
- **Cassandra vs Postgres for state store:** Cassandra handles the write volume (millions of updates/sec across all stores) and partitions naturally by `store_id`. Postgres would require heavy sharding. Trade: Cassandra has weaker consistency, but eventual consistency is acceptable for inventory counts (exact-once isn't possible in the physical world anyway — shrinkage exists).
- **Flink state vs external DB for running count:** Flink state gives sub-second computation but is ephemeral (recovered from checkpoints on failure). Cassandra is the durable source of truth. Both are updated — Flink state for speed, Cassandra for durability.
- **Replenishment threshold logic in-stream vs batch:** In-stream triggers are faster (react within seconds) but can over-trigger during flash sales. Add a cooldown window: don't re-trigger replenishment for the same SKU within 4 hours.

</details>

---

### Q4 🟡 SENIOR — "Design an A/B testing analytics platform"

<details>
<summary><strong>Question → Answer</strong></summary>

**Requirements:**
- Support 50+ concurrent experiments across web and mobile.
- Assign users to variants deterministically (same user always sees same variant).
- Compute metrics: conversion rate, revenue per user, engagement. Detect statistical significance.
- Guard against metric degradation (guardrail metrics).

**Architecture:**

```
User Request → Experiment Assignment Service (hash(user_id + experiment_id) % 100)
    → Feature flags / variant config returned to client
    → User interactions → Event Collection (Kafka)
        → Flink: enrich events with experiment assignments
            → Data Lake (Bronze → Silver → Gold)
                → Metric Computation Engine (Spark / dbt)
                    → Statistical Analysis (Python: scipy, statsmodels)
                        → Results Dashboard (with confidence intervals)
```

**Data Model:**
- **Experiment Config:** `{experiment_id, name, variants[], traffic_allocation, start_date, end_date, target_metrics[], guardrail_metrics[], min_sample_size}`
- **Assignment:** `{user_id, experiment_id, variant, assigned_at}` — deterministic hash, no randomness needed at query time.
- **Event:** `{event_id, user_id, event_type, properties{}, timestamp}`
- **Metric Result:** `{experiment_id, variant, metric_name, sample_size, mean, stddev, ci_lower, ci_upper, p_value, is_significant}`

**Deep Dive — Statistical Engine:**
- Use **Welch's t-test** for continuous metrics (revenue per user), **chi-squared test** for proportions (conversion rate).
- Compute **confidence intervals** (95%) and **p-values**. Significance threshold: p < 0.05.
- **Sequential testing** (always-valid p-values) to allow peeking without inflating false positive rates.
- **Guardrail metrics:** latency p99, error rate, crash rate. If any guardrail degrades significantly, auto-alert and recommend stopping the experiment.
- **Sample size calculator:** given baseline rate, minimum detectable effect (MDE), and desired power (80%), compute required users per variant. Display progress bar on dashboard.

**Tradeoffs:**
- **Pre-computed metrics (batch) vs on-demand:** Batch (daily Spark jobs) is cheaper but results are 24h stale. On-demand (query warehouse at dashboard load) is fresher but expensive at 50+ experiments × 10+ metrics each. Hybrid: batch compute daily, allow ad-hoc drill-downs.
- **Bayesian vs Frequentist:** Frequentist (p-values) is industry standard and easier to explain. Bayesian gives probability of being better ("92% chance variant B is better") which is more intuitive. Choose based on team's statistical literacy.

</details>

---

### Q5 🟢 COMMON — "Design a Customer 360 platform"

<details>
<summary><strong>Question → Answer</strong></summary>

**Requirements:**
- Unify customer data from: CRM (Salesforce), orders (Postgres), clickstream (Kafka), support tickets (Zendesk), email engagement (Marketo), social (Twitter API).
- Build a single unified customer profile.
- Serve: segmentation (marketing), churn prediction (data science), personalization (product), support context (customer service).

**Architecture:**

```
Sources (6+)
    → CDC / API Extraction → Kafka → Bronze Layer (raw per source)
        → Identity Resolution Engine (Silver)
            ├── Deterministic matching: email, phone, loyalty_id
            ├── Probabilistic matching: name + address fuzzy match (Jaro-Winkler, >0.85 threshold)
            └── Output: unified_customer_id mapping table
        → Profile Builder (Silver → Gold)
            ├── Merge all source data on unified_customer_id
            ├── Conflict resolution: most-recent-wins for address, source-priority for name
            └── Compute derived fields: lifetime_value, churn_risk_score, preferred_channel
                → Gold: customer_360 table
                    ├── Warehouse (BI / segmentation queries)
                    ├── Redis (real-time lookup for support agents)
                    └── Reverse ETL → Salesforce, Marketo (activation)
```

**Data Model — Gold `customer_360`:**
```
unified_customer_id | first_name | last_name | email | phone
| address | ltv | order_count | last_order_date | avg_order_value
| first_purchase_date | preferred_channel | churn_risk_score
| support_ticket_count | csat_avg | last_support_date
| segments[] | source_ids{crm_id, order_id, clickstream_id, ...}
```

**Deep Dive — Identity Resolution:**
This is the hardest part. Customers use different emails, change addresses, share devices.
- **Step 1: Deterministic keys** — match on `email`, `phone`, `loyalty_card_id` across sources. This resolves ~70% of matches.
- **Step 2: Probabilistic matching** — for remaining records, use Jaro-Winkler string similarity on `(first_name, last_name, address)`. Threshold >0.85 = match. Use blocking (only compare records in the same zip code) to avoid O(n²) comparisons.
- **Step 3: Graph-based resolution** — build a graph where nodes are identifiers and edges are matches. Connected components = same person. Handles transitive matches (email A matched to phone B, phone B matched to email C → all same person).

**Tradeoffs:**
- **Aggressive vs conservative matching:** Too aggressive = merge two different people (false positive). Too conservative = same person has two profiles (false negative). Start conservative (high threshold), let business users manually merge edge cases via a UI.
- **Batch vs real-time resolution:** Batch (nightly Spark job) is simpler and handles probabilistic matching well. Real-time (Flink) updates profiles within seconds of a new event but probabilistic fuzzy matching in a stream is complex. Most companies do batch resolution + real-time append of events to the latest resolved profile.

</details>

---

### Q6 🔴 HARD — "Design a data quality platform for a large organization"

<details>
<summary><strong>Question → Answer</strong></summary>

**Requirements:**
- 500+ tables across 30+ pipelines (Airflow, dbt, Spark).
- Define, execute, and monitor quality rules (completeness, freshness, schema conformance, custom business rules).
- Quality scores per table, lineage-aware alerting (if upstream fails, don't alert on all downstream tables separately).
- Remediation workflows: assign, track, resolve quality incidents.

**Architecture:**

```
Pipeline Execution (Airflow / dbt / Spark)
    → OpenLineage events → Metadata Collector
    → Quality Rule Engine
        ├── Scheduled checks (Great Expectations / Soda)
        ├── Inline checks (dbt tests, Spark validation)
        └── ML anomaly detection (volume, distribution drift)
    → Quality Score Calculator
        ├── Per-table score: weighted average of checks (freshness 30%, completeness 30%, validity 20%, uniqueness 20%)
        └── Per-pipeline score: min(table scores in pipeline)
    → Lineage-Aware Alert Router
        ├── If root cause table identified → alert root cause owner only
        ├── Downstream tables → suppress alerts, show "upstream dependency failure"
        └── Channels: PagerDuty (critical), Slack (warning), email (digest)
    → Incident Management
        ├── Auto-create incidents in Jira/ServiceNow
        ├── Track: MTTD (mean time to detect), MTTR (mean time to resolve)
        └── Remediation playbooks per failure type
    → Quality Dashboard
        ├── Organization-wide quality scorecard
        ├── Trend lines (is quality improving over time?)
        └── Drill-down by team, pipeline, table
```

**Deep Dive — Lineage-Aware Alerting:**
The naive approach: alert on every table that fails a quality check. If `bronze_orders` has a freshness issue, you get alerts for `bronze_orders`, `silver_orders`, `gold_daily_revenue`, `gold_customer_360`, and every downstream dashboard. That's 15 alerts for one root cause.

The smart approach: use the lineage graph to identify the **root cause table** (the furthest-upstream table with a failure). Alert only the owner of that table. For all downstream tables, show status as "degraded — upstream dependency failure on `bronze_orders`" without paging anyone.

**Tradeoffs:**
- **Centralized quality platform vs embedded in each pipeline:** Centralized gives a single pane of glass and consistent scoring. But pipeline teams resist external tools. Best hybrid: embed lightweight checks in pipelines (dbt tests, GE suites) and aggregate results into the central platform via API.
- **ML anomaly detection vs threshold rules:** ML catches unknown-unknowns (distribution shifts, gradual degradation) but produces false positives. Threshold rules are precise but require manual maintenance. Layer both: thresholds for known invariants, ML for everything else.

</details>

---

### Q7 🟡 SENIOR — "Design a cost optimization system for cloud data infrastructure"

<details>
<summary><strong>Question → Answer</strong></summary>

**Requirements:**
- Track and optimize costs across: Spark/Databricks clusters, S3/GCS storage, Snowflake/BigQuery query costs, Kafka clusters, Airflow infrastructure.
- Attribute costs to teams, projects, and pipelines.
- Detect anomalies (sudden cost spikes), recommend optimizations, automate savings.

**Architecture:**

```
Cloud Billing APIs (AWS Cost Explorer, GCP Billing, Databricks API)
    → Cost Collector (daily batch)
        → Cost Attribution Engine
            ├── Join billing line items with resource tags (team, project, pipeline)
            ├── Untagged resources → heuristic attribution (cluster → Airflow DAG → team)
            └── Shared resources → proportional allocation (Kafka by topic bytes, S3 by prefix)
        → Anomaly Detector
            ├── Z-score on daily cost per team/service (alert if >3σ)
            ├── Week-over-week comparison (alert if >50% increase)
            └── New resource detection (new cluster/table appeared)
        → Recommendation Engine
            ├── Storage: suggest S3 lifecycle policies (IA after 30d, Glacier after 90d)
            ├── Compute: identify idle clusters, right-size over-provisioned instances
            ├── Queries: flag full-table scans on large tables, suggest partitioning/clustering
            ├── Spot/preemptible: identify fault-tolerant workloads that can use spot instances
            └── Zombie resources: unused tables (no reads in 90d), idle endpoints
        → Dashboard + Alerts
            ├── Cost by team, by service, by pipeline (drillable)
            ├── Month-over-month trends, forecasting
            └── Savings tracker (recommendations implemented → dollars saved)
```

**Tradeoffs:**
- **Tag-based vs heuristic attribution:** Tags are accurate but require discipline (every resource tagged). Heuristic (infer team from Airflow DAG ownership) fills gaps but can misattribute. Mandate tagging in Terraform/IaC and use heuristics as fallback.
- **Automated vs recommended optimization:** Auto-scaling down idle clusters saves money but risks breaking ad-hoc workloads. Start with recommendations + approval workflow; graduate to automation for well-understood patterns (e.g., auto-hibernate dev clusters at 7pm).

</details>

---

### Q8 🟢 COMMON — "Design a data catalog and discovery platform"

<details>
<summary><strong>Question → Answer</strong></summary>

**Requirements:**
- Make data discoverable across 1,000+ tables in a lakehouse, 200+ Airflow DAGs, 300+ dbt models.
- Search by table name, column name, description, owner, tags.
- Show lineage (where does this table come from? what feeds into it?).
- Track popularity (most-queried tables), freshness, and quality scores.

**Architecture:**

```
Metadata Sources
    ├── Schema Crawler (Hive Metastore / Glue Catalog / Unity Catalog) → table/column metadata
    ├── Airflow API → DAG definitions, task dependencies, run history
    ├── dbt manifest.json → model descriptions, tests, column docs
    ├── Query Logs (Spark/Trino/Snowflake) → usage stats, popularity, user access patterns
    └── OpenLineage events → runtime lineage (which jobs read/write which tables)
→ Metadata Ingestion Service
    → Metadata Store (Postgres + Elasticsearch)
        ├── Postgres: structured metadata (tables, columns, owners, tags, quality scores)
        ├── Elasticsearch: full-text search index (descriptions, column names, tags)
        └── Graph DB (Neo4j) or lineage table: lineage edges (table A → job B → table C)
    → Search API + Lineage API + Popularity API
        → Catalog UI
            ├── Search: "revenue" → shows gold_daily_revenue, silver_orders, dbt model fct_revenue
            ├── Table detail page: schema, description, owner, tags, freshness, quality score, sample data
            ├── Lineage graph: interactive upstream/downstream visualization
            ├── Popularity: "most queried this week", "trending tables"
            └── Access request: "Request access" → approval workflow → auto-grant RBAC
```

**Tradeoffs:**
- **Build vs buy:** DataHub (open source) or Atlan/Alation (commercial) vs custom-built. DataHub covers 80% of needs with lower effort. Custom gives full control but requires a dedicated team. Most companies start with DataHub and extend.
- **Push vs pull metadata ingestion:** Pull (scheduled crawlers) is simpler but stale. Push (OpenLineage emitted at runtime) is real-time but requires instrumentation in every pipeline. Best: push for lineage (real-time), pull for schema (daily crawler).

</details>

---

## Screen 2: Trade-off Discussions

Every senior-level interview includes a trade-off discussion. The interviewer doesn't want the "right" answer — they want to see you **hold two opposing ideas in your head and articulate when each wins**. Here are the eight trade-offs you must master.

---

### 1. Batch vs Streaming Processing

| | Batch | Streaming |
|---|---|---|
| **Definition** | Process data in discrete chunks on a schedule (hourly/daily) | Process data continuously as it arrives |
| **Pros** | Simple to build and debug; cheaper compute (spot instances); easier exactly-once semantics; mature tooling (Spark, dbt) | Low latency (seconds); real-time dashboards and alerts; natural fit for event-driven architectures |
| **Cons** | Data is stale until next run; large catch-up jobs after failures; doesn't handle time-sensitive use cases | Complex state management; harder debugging; higher operational cost; exactly-once is harder |

**Choose Batch when:** data consumers tolerate hourly/daily freshness, workloads are transformations over large historical datasets, team lacks streaming expertise, or cost is a primary concern.

**Choose Streaming when:** latency requirements are <5 minutes, use cases are event-driven (fraud, alerts, real-time personalization), or data naturally arrives as a stream (clickstream, IoT sensors).

### 💡 Interview Insight
> *"Most production systems are hybrid — streaming for time-sensitive paths, batch for historical aggregations and backfills. Say this in an interview to show pragmatism over dogma."*

---

### 2. Data Lake vs Data Warehouse

| | Data Lake | Data Warehouse |
|---|---|---|
| **Definition** | Store raw data in open formats (Parquet/JSON) on object storage | Store structured, modeled data in a proprietary columnar engine |
| **Pros** | Cheap storage (~$0.023/GB/mo); schema flexibility; multi-engine access; supports ML and unstructured data | Fast query performance; built-in governance; SQL-native; concurrency scaling; great for BI |
| **Cons** | "Data swamp" risk without governance; query performance depends on engine choice; requires more engineering | Expensive at scale ($20–40/TB/mo); vendor lock-in; poor for unstructured data; rigid schema |

**Choose Data Lake when:** you have diverse data types (logs, images, JSON), need multi-engine processing (Spark + Flink + Trino), or cost sensitivity is high.

**Choose Data Warehouse when:** your primary consumers are BI analysts running SQL, you need strong governance out of the box, or you want managed performance without tuning.

> 💡 *The modern answer is "Lakehouse" — lake storage + warehouse-like query performance via table formats (Iceberg/Delta) and engines (Databricks SQL, Trino). Mention this to show you're current.*

---

### 3. Schema-on-Read vs Schema-on-Write

| | Schema-on-Read | Schema-on-Write |
|---|---|---|
| **Definition** | Store data as-is; apply schema when querying | Validate and enforce schema at write time |
| **Pros** | Fast ingestion; preserves raw data fidelity; flexible for exploration; accommodates schema changes easily | Catches errors early; downstream consumers trust the data; query performance is better (typed columns) |
| **Cons** | Garbage in, garbage out — bad data flows downstream; slow queries on unstructured data; schema drift goes undetected | Ingestion failures on schema mismatch; rigid — schema changes require pipeline updates; slower ingestion |

**Choose Schema-on-Read when:** ingesting from unreliable or rapidly-changing sources, in the Bronze layer, or when you're still exploring the data.

**Choose Schema-on-Write when:** writing to Silver/Gold layers, when downstream consumers depend on consistent schemas, or when data quality is critical.

> 💡 *The medallion architecture naturally blends both: Schema-on-Read for Bronze (preserve everything), Schema-on-Write for Silver/Gold (enforce contracts).*

---

### 4. Denormalized vs Normalized Data Models

| | Denormalized | Normalized |
|---|---|---|
| **Definition** | Wide tables with redundant data; fewer joins needed | Related data split across tables linked by foreign keys; no redundancy |
| **Pros** | Faster reads (no joins); simpler queries; great for BI/dashboards; columnar formats compress well | No data redundancy; easier updates (change in one place); enforces data integrity; smaller storage |
| **Cons** | Data redundancy; update anomalies; larger storage; harder to maintain consistency | Slow reads (many joins); complex queries; poor for analytical workloads at scale |

**Choose Denormalized when:** building Gold-layer analytical tables, serving BI dashboards, query performance matters more than storage cost.

**Choose Normalized when:** building operational/transactional systems, source-of-truth Silver-layer tables, or when data changes frequently and consistency matters.

> 💡 *"In a data warehouse, I'd use a star schema — fact tables are denormalized for query speed, dimension tables are lightly normalized. This is the Kimball approach and still dominates BI workloads."*

---

### 5. Push vs Pull Ingestion

| | Push (Source → Pipeline) | Pull (Pipeline → Source) |
|---|---|---|
| **Definition** | Source systems push data to the pipeline (webhooks, Kafka producers, API calls) | Pipeline pulls data from sources on a schedule (JDBC queries, API polling, file scanning) |
| **Pros** | Real-time; source controls emission timing; natural for event-driven systems | Pipeline controls timing and rate; simpler for sources (no producer to build); easier backfill |
| **Cons** | Pipeline must handle variable load; source must implement a producer; harder to backfill historical data | Stale data between pulls; can overload source with queries; misses deletes unless CDC is used |

**Choose Push when:** source naturally produces events (application logs, user clicks, IoT sensors), low latency is needed, or the source team owns producing.

**Choose Pull when:** ingesting from databases (JDBC), third-party APIs with rate limits, vendor file drops, or when the source team can't build a producer.

> 💡 *"CDC (Debezium) is the best of both worlds for databases — it reads the WAL (push from DB's perspective) but the CDC connector pulls from the WAL. You get real-time without polling the source tables."*

---

### 6. Monolithic vs Microservice Data Pipelines

| | Monolithic (Central Team) | Microservice / Domain-Owned |
|---|---|---|
| **Definition** | One central data team owns all pipelines in a single Airflow instance | Each domain team owns their data pipelines and publishes data products |
| **Pros** | Consistent standards; single orchestrator; easy cross-domain joins; simpler governance | Teams move independently; domain expertise embedded in pipelines; scales with org size; data mesh aligned |
| **Cons** | Central team is bottleneck; slow iteration; doesn't scale past ~10 data engineers; monolithic DAG complexity | Inconsistent quality; duplication of effort; cross-domain queries harder; governance overhead |

**Choose Monolithic when:** small team (<10 DEs), early-stage data platform, need tight control over quality and cost.

**Choose Microservice/Domain when:** organization is large (>50 DEs), domains have distinct data needs, data mesh principles are adopted, or the central team is a bottleneck.

> 💡 *"Data mesh is the buzzword, but the reality is most companies are somewhere in between — a central platform team provides the infrastructure (Kafka, Spark, Airflow, catalog) and domain teams build pipelines on top. Call this 'federated with a platform.'"*

---

### 7. SQL-First (dbt) vs Code-First (Spark/Python)

| | SQL-First (dbt) | Code-First (Spark/Python) |
|---|---|---|
| **Definition** | Transformations written in SQL, managed by dbt | Transformations written in Python/Scala with PySpark or Pandas |
| **Pros** | Accessible to analysts; built-in testing + docs; version-controlled; declarative; fast iteration | Handle complex logic (ML, APIs, fuzzy matching); full programming language flexibility; better for unstructured data |
| **Cons** | Complex logic in SQL is painful (recursive CTEs, UDFs); limited to tabular transformations; depends on warehouse SQL dialect | Higher learning curve; more boilerplate; testing requires more setup; harder for analysts to contribute |

**Choose SQL/dbt when:** transformations are joins, aggregations, and filters; the team includes analytics engineers; operating within a warehouse/lakehouse with good SQL support.

**Choose Spark/Python when:** transformations involve ML models, API calls, complex string parsing, graph algorithms, or processing non-tabular data (images, logs).

> 💡 *"The best teams use both — dbt for Silver→Gold transformations (joins, aggregations, business logic) and Spark for Bronze→Silver (parsing, deduplication, heavy ETL). Say this in interviews."*

---

### 8. Managed Services vs Self-Hosted

| | Managed (Snowflake, BigQuery, Confluent Cloud) | Self-Hosted (Spark on K8s, Apache Kafka, Trino) |
|---|---|---|
| **Definition** | Vendor operates the infrastructure; you consume via API/SQL | You deploy, configure, scale, and maintain the infrastructure yourself |
| **Pros** | Zero ops burden; auto-scaling; built-in security and compliance; faster time-to-value | Full control; no vendor lock-in; potentially cheaper at massive scale; customizable |
| **Cons** | Expensive at scale; vendor lock-in; limited customization; data egress costs | Requires dedicated platform team; operational burden (upgrades, security patches, scaling); slower to start |

**Choose Managed when:** team is small, speed-to-market matters, operational expertise is limited, or workloads are variable (auto-scaling saves money).

**Choose Self-Hosted when:** scale is massive (>PB), you have a dedicated platform team (>5 infra engineers), workloads are predictable (reserved capacity is cheaper), or vendor lock-in is unacceptable.

> 💡 *"My default recommendation is managed services unless there's a compelling reason to self-host. The total cost of ownership of self-managed Kafka (including engineering time for upgrades, monitoring, security) almost always exceeds Confluent Cloud for teams under 10 engineers."*

---

## Screen 3: Failure Scenarios — "What Happens When X Fails?"

Interviewers love failure scenarios because they reveal **operational maturity**. Anyone can design the happy path. Senior engineers design for the failure path. For each scenario: blast radius → triage → fix → prevention.

---

### 1. 🔴 HARD — "Kafka broker goes down mid-stream — what happens to your pipeline?"

<details>
<summary><strong>Scenario → Answer</strong></summary>

**What Breaks:**
- If one broker dies in a 3-broker cluster, partitions led by that broker become temporarily unavailable.
- Producers get `NotLeaderForPartitionException` and retry to the new leader (if replication factor ≥ 3 and `min.insync.replicas=2`, no data loss).
- Consumers rebalance — consumer group coordinator reassigns partitions. During rebalance (~10–30 seconds), those partitions aren't consumed. Lag grows.
- If replication factor is 1 (terrible idea), **data is lost** for partitions on that broker.

**Immediate Triage:**
1. Check broker health in Kafka admin UI or `kafka-broker-api-versions.sh`.
2. Verify ISR (in-sync replicas) count — if ISR < `min.insync.replicas`, producers are blocked.
3. Monitor consumer lag via Burrow or Kafka consumer group CLI — confirm lag is growing but bounded.

**Fix:**
1. If hardware failure: replace the broker. Kafka automatically reassigns partitions via controller.
2. If software crash: restart the broker process. It rejoins the cluster and catches up from replicas.
3. Verify consumers have resumed and lag is decreasing.

**Prevention:**
- **Replication factor ≥ 3** with `min.insync.replicas=2` — survive one broker loss with zero data loss.
- **Rack-aware replication** — replicas on different racks/AZs to survive rack-level failures.
- **Consumer idempotency** — consumers handle rebalance-triggered redelivery without duplicates (use `MERGE`/upsert on sink).
- **Monitoring:** alert on ISR shrink, consumer lag >threshold, under-replicated partitions.

</details>

---

### 2. 🔴 HARD — "Your Spark job fails halfway through a 4-hour batch run. How do you recover without reprocessing everything?"

<details>
<summary><strong>Scenario → Answer</strong></summary>

**What Breaks:**
- 4 hours of compute wasted. If the job writes to a non-transactional sink (raw Parquet on S3), you might have partial data written — some partitions complete, others missing or corrupt.
- Downstream DAGs that depend on this job may time out or run on stale data.

**Immediate Triage:**
1. Check Spark UI / driver logs for the root cause — OOM? Data skew? Source unavailable? Schema change?
2. Determine which partitions/stages completed successfully.

**Fix — Design for Restartability:**
1. **Partition-level granularity:** Write output partitioned by date or key. Track which partitions are complete in a checkpoint table (`pipeline_checkpoints: {job_name, partition_key, status, completed_at}`). On restart, skip completed partitions.
2. **Delta Lake / Iceberg transactions:** If using table formats, a failed write is automatically rolled back (uncommitted files are cleaned by `VACUUM`). Simply re-run — the ACID transaction ensures no partial state.
3. **Spark checkpointing (Structured Streaming):** For streaming jobs, Spark resumes from the last checkpoint offset automatically.
4. **Idempotent writes:** Use `MERGE` or `INSERT OVERWRITE PARTITION` so re-running the entire job produces the same result without duplicates.

**Prevention:**
- **Break 4-hour jobs into smaller increments** — process one hour at a time. If hour 3 fails, you only re-run hour 3.
- **Dynamic resource allocation:** enable `spark.dynamicAllocation.enabled=true` to avoid OOM from fixed executor counts.
- **Data skew handling:** salted keys, AQE skew join optimization, or pre-filter known hot keys.
- **Retry in Airflow:** `retries=2, retry_delay=timedelta(minutes=10)` with idempotent task design.

</details>

---

### 3. 🟡 SENIOR — "A source database schema changes without warning (column renamed). What breaks and how do you handle it?"

<details>
<summary><strong>Scenario → Answer</strong></summary>

**What Breaks:**
- Ingestion pipeline fails with a column-not-found error (if strict schema) or silently produces NULLs for the renamed column (if permissive).
- Downstream Silver/Gold tables that reference the old column name fail or produce incorrect results.
- Dashboards show blank or wrong values. Business users lose trust.

**Immediate Triage:**
1. Alert fires (ideally from a schema change detector or dbt test failure).
2. Identify: which column changed? Was it renamed, dropped, or type-changed?
3. Check blast radius: which Silver/Gold tables and dashboards depend on this column?

**Fix:**
1. Update the ingestion pipeline to map the new column name → old column name (backward-compatible alias in the Bronze→Silver transform).
2. Re-run the pipeline for the affected time window.
3. Validate downstream tables with data quality checks.

**Prevention:**
- **Schema Registry:** Enforce schemas on Kafka topics (Avro/Protobuf). Producers can't change schemas without backward-compatibility checks.
- **Data Contracts:** Formal agreements between source teams and data teams. Schema changes require a PR-like review process with a deprecation period.
- **Schema change detection:** Nightly job compares current source schema to last-known schema. Alert on any diff *before* the pipeline runs.
- **Defensive ingestion:** Bronze layer stores raw data as-is. Schema enforcement happens in Silver. This means a source schema change doesn't break Bronze — it breaks Silver, which is easier to fix and reprocess.

</details>

---

### 4. 🟡 SENIOR — "Your data pipeline produces duplicate records. How do you detect and fix this?"

<details>
<summary><strong>Scenario → Answer</strong></summary>

**What Breaks:**
- Aggregations are inflated (revenue doubled, user counts wrong).
- Joins produce fan-outs (one order appears twice, each line item joins to both).
- Business decisions made on wrong numbers.

**Detection:**
1. **dbt test:** `unique` test on primary key columns — cheapest and fastest.
2. **SQL check:** `SELECT primary_key, COUNT(*) FROM table GROUP BY primary_key HAVING COUNT(*) > 1` — run as a data quality gate.
3. **Volume anomaly:** row count suddenly doubles compared to historical baseline.
4. **Business user reports:** "Revenue is 2x what it should be" — the worst way to find out.

**Fix:**
1. **Immediate:** Deduplicate in Silver layer — `ROW_NUMBER() OVER (PARTITION BY primary_key ORDER BY updated_at DESC) = 1`.
2. **Root cause:** Why are duplicates appearing?
   - Kafka consumer rebalance reprocessed messages? → Make consumer idempotent.
   - Pipeline retry created duplicate writes? → Use `MERGE`/upsert instead of `INSERT`.
   - Source system sent duplicates? → Deduplicate in Bronze→Silver.
   - Multiple instances of the same pipeline ran simultaneously? → Add Airflow `max_active_runs=1` and concurrency locks.

**Prevention:**
- **Idempotent pipelines:** Every write uses `MERGE` on primary key or `INSERT OVERWRITE PARTITION`.
- **Exactly-once semantics:** Kafka transactions + Flink checkpointing for streaming.
- **dbt unique tests:** On every table, every run. Fail the pipeline if duplicates exist.
- **Primary key enforcement:** Delta Lake and Iceberg don't enforce PKs natively — use dbt tests or Great Expectations as the enforcement layer.

</details>

---

### 5. 🔴 HARD — "The data lake has corrupt data in Silver that propagated to Gold and was served to dashboards. What's your incident response?"

<details>
<summary><strong>Scenario → Answer</strong></summary>

**Blast Radius:**
- Gold tables have incorrect data. Dashboards are showing wrong metrics. Business decisions may have already been made on bad data.
- Depending on how long the corruption went undetected, hours/days of data may be affected.

**Incident Response (structured):**

**1. Detect & Alert (MTTD):**
- Data quality checks (Great Expectations, dbt tests) should have caught this. If they didn't, the first gap is in your quality coverage.
- Business user or analyst reports the anomaly.

**2. Triage (first 15 minutes):**
- Identify the affected time range and tables.
- Use lineage graph to trace corruption from Gold → Silver → Bronze → source.
- Determine: is the corruption from a bad source, a bug in Silver transformation logic, or infrastructure failure?

**3. Contain:**
- **Option A: Time travel rollback.** Use Delta Lake / Iceberg time travel to restore Gold tables to the last known good version: `RESTORE TABLE gold_daily_revenue TO VERSION AS OF 42`.
- **Option B: Flag dashboards.** Add a banner to affected dashboards: "Data under review — numbers may be inaccurate for [date range]."
- Notify stakeholders with affected scope and ETA for fix.

**4. Fix:**
- Fix the Silver transformation logic or source data issue.
- Reprocess Silver from Bronze for the affected time range.
- Reprocess Gold from corrected Silver.
- Validate with quality checks before re-serving.

**5. Post-Incident:**
- Blameless post-mortem: timeline, root cause, blast radius, MTTD, MTTR.
- Add quality checks that would have caught this earlier (e.g., "daily revenue should be within 2σ of 30-day average").
- Update runbooks with this scenario.

**Prevention:**
- **Quality gates between layers:** Bronze→Silver and Silver→Gold transitions should have automated quality checks. Pipeline halts if checks fail.
- **Time travel retention:** Keep ≥7 days of Delta/Iceberg snapshots for rollback capability.
- **Data observability:** ML-based anomaly detection (Monte Carlo, Bigeye) that catches distribution shifts and volume anomalies automatically.

</details>

---

### 6. 🟡 SENIOR — "Kafka consumer lag is growing — consumers can't keep up with producers. What do you do?"

<details>
<summary><strong>Scenario → Answer</strong></summary>

**What Breaks:**
- Data processing falls behind. Real-time dashboards become stale. Latency increases from seconds to minutes or hours.
- If lag exceeds Kafka's retention period, **data loss** occurs — oldest messages are deleted before consumers read them.

**Immediate Triage:**
1. Check consumer lag per partition (Burrow, Kafka consumer group CLI, Grafana).
2. Identify: is lag uniform across partitions (consumer is slow) or concentrated on specific partitions (data skew / hot partition)?
3. Check consumer health: are consumers alive? Are they stuck on a slow operation (e.g., blocking DB write)?

**Fix (ordered by speed):**
1. **Scale consumers:** Add more consumer instances (up to the number of partitions — one consumer per partition max in a consumer group).
2. **Fix hot partitions:** If one partition has 10x the data, re-key the topic or add salt to the partition key.
3. **Optimize consumer processing:** batch writes to the sink, reduce per-record processing time, use async I/O.
4. **Increase Kafka retention:** Buy time by extending retention from 7 days to 14 days while you fix the root cause.
5. **Backpressure:** If using Flink, backpressure is automatic. If using a custom consumer, implement rate limiting on the producer side temporarily.

**Prevention:**
- **Auto-scaling consumers:** Kubernetes HPA based on consumer lag metric.
- **Monitoring:** alert when lag exceeds 5 minutes (warning) or 30 minutes (critical).
- **Load testing:** test consumer throughput at 2x expected peak before going to production.
- **Partition count planning:** ensure enough partitions for parallel consumption. Rule of thumb: partitions ≥ expected peak consumer count.

</details>

---

### 7. 🟢 COMMON — "An Airflow DAG fails at 3am. Walk me through your alerting and recovery process."

<details>
<summary><strong>Scenario → Answer</strong></summary>

**Alerting Chain:**
1. Airflow task fails → built-in retry (`retries=3, retry_delay=5min`).
2. All retries exhausted → Airflow `on_failure_callback` fires.
3. Callback sends alert to: PagerDuty (pages on-call engineer), Slack #data-alerts channel (visibility for the team).
4. Alert includes: DAG name, task name, execution date, error log snippet, link to Airflow UI.

**On-Call Engineer Response:**
1. **Acknowledge** the page within 15 minutes (SLA).
2. **Read the logs** in Airflow UI — identify error type (OOM, source unavailable, permission denied, data issue).
3. **Assess blast radius:** what downstream DAGs/tables depend on this one? Use `ExternalTaskSensor` dependencies or lineage graph.
4. **Fix or escalate:**
   - Transient error (network timeout, spot instance preemption): clear the failed task and re-trigger.
   - Data issue (source schema change, empty source): fix the data issue, then re-trigger.
   - Code bug: fix in a feature branch, deploy, re-trigger.
5. **Verify:** confirm the task succeeded and downstream DAGs completed. Check data quality.
6. **Document:** log incident in the on-call handoff doc with root cause and resolution.

**Prevention:**
- **SLA monitoring:** Airflow SLA misses trigger alerts when a DAG doesn't complete by its deadline (e.g., `gold_daily_revenue` must complete by 6 AM).
- **Idempotent tasks:** every task is safe to re-run — no duplicates, no side effects.
- **Dependency checks:** `ExternalTaskSensor` or data freshness sensors to avoid running if upstream isn't ready.
- **Runbooks:** per-DAG documentation covering common failure modes and resolution steps.

</details>

---

### 8. 🔴 HARD — "You discover that a critical dimension table hasn't been updated for 3 days but nobody noticed. How do you prevent this?"

<details>
<summary><strong>Scenario → Answer</strong></summary>

**Why It Went Undetected:**
- The pipeline may have "succeeded" — it ran but processed zero records (source was empty or query returned no results).
- No freshness monitoring. The Airflow task status was "success" (it ran without errors), but the data was stale.
- Downstream tables that JOIN to this dimension didn't fail — they just used stale dimension values.

**Immediate Fix:**
1. Investigate why the source stopped providing updates — API credential expired? Source table was dropped? CDC connector died?
2. Backfill the dimension table for the missing 3 days.
3. Reprocess any Gold tables that used the stale dimension (they may have incorrect dimension attributes on fact records).

**Prevention — Multi-Layer Freshness Monitoring:**
1. **Table-level freshness SLI:** `SELECT MAX(updated_at) FROM dim_product` — alert if >24 hours stale. Run this check every hour via a monitoring DAG or Great Expectations / Soda.
2. **Row count checks:** alert if a pipeline succeeds but processes 0 rows. A successful DAG run with zero output is a silent failure.
3. **Freshness SLO:** *"dim_product must be updated within 24 hours of the source, 99.5% of the time."* Track compliance on a dashboard.
4. **Data quality dashboard:** a single pane showing freshness, completeness, and volume for all critical tables. Red/yellow/green status. Review daily.
5. **Anomaly detection on update patterns:** dim_product updates daily at ~2 AM. If no update by 6 AM, auto-alert.

### 💡 Interview Insight

> **Silent failures are more dangerous than loud ones.** A pipeline that crashes gets immediate attention. A pipeline that runs successfully but produces zero or stale results can go undetected for days. Always monitor the *data*, not just the *pipeline status*. The key phrase: *"I monitor data freshness and volume, not just Airflow task status."*

</details>

---

## Screen 4: Rapid Fire — 20 Quick Q&A

No preamble. Short questions. Tight answers. This is the speed round.

---

**1. What is backpressure in streaming?**
When a downstream consumer can't process data as fast as the upstream producer sends it. Flink handles this automatically by slowing down source reads. Without backpressure handling, you get OOM errors or data loss.

**2. Explain CAP theorem in 15 seconds.**
In a distributed system, you can only guarantee two of three: **Consistency** (every read sees the latest write), **Availability** (every request gets a response), and **Partition tolerance** (system works despite network splits). Since network partitions are inevitable, you're really choosing between CP (e.g., HBase) and AP (e.g., Cassandra).

**3. ACID vs BASE?**
ACID (Atomicity, Consistency, Isolation, Durability) guarantees strict transactional integrity — used in relational databases. BASE (Basically Available, Soft state, Eventually consistent) relaxes consistency for availability and scale — used in distributed NoSQL systems.

**4. What's the difference between SLA, SLO, and SLI?**
**SLI** = metric you measure (e.g., data freshness = 2 hours). **SLO** = target for that metric (e.g., freshness < 2 hours, 99% of the time). **SLA** = contractual commitment with consequences if the SLO is breached (e.g., service credits).

**5. What is a dead letter queue?**
A holding area for messages that failed processing — malformed records, schema violations, or unhandled exceptions. Instead of blocking the pipeline, bad records are routed to the DLQ for later inspection and remediation.

**6. Explain event sourcing in one sentence.**
Instead of storing current state, store every state change as an immutable event — the current state is derived by replaying all events in order.

**7. What is partition pruning?**
A query optimization where the engine skips reading partitions that don't match the query's filter. If you query `WHERE event_date = '2026-04-06'` on a table partitioned by `event_date`, only that one partition is scanned instead of the entire table.

**8. Delta Lake vs Iceberg — one sentence each.**
**Delta Lake:** Databricks-native table format using a JSON transaction log, best integrated with Spark and Databricks ecosystem. **Iceberg:** Vendor-neutral Apache table format with a manifest-tree metadata structure, best for multi-engine environments (Spark + Trino + Flink).

**9. What is data skew and how do you fix it?**
Data skew is when one partition or key has disproportionately more data than others, causing one task to run 100x longer than the rest. Fix with salted keys (append a random suffix to the hot key), Spark AQE skew join optimization, or pre-filtering/pre-aggregating hot keys.

**10. Explain watermarks in stream processing.**
A watermark is a timestamp that declares *"I believe all events with timestamps before this point have arrived."* It tells the stream processor when it's safe to close a window and emit results. Late events (after the watermark) are either dropped or sent to a side output.

**11. What is Z-ordering?**
A multi-dimensional data layout technique that co-locates related data within files. If you Z-order a table by `(region, product_id)`, queries filtering on either column will scan fewer files. It interleaves the bits of multiple columns to create a space-filling curve.

**12. Star schema vs snowflake schema?**
**Star schema:** central fact table surrounded by denormalized dimension tables — one join to reach any dimension. **Snowflake schema:** dimensions are normalized into sub-dimensions (e.g., `product → category → department`) — more tables, more joins, less redundancy. Star schema is preferred for BI query performance.

**13. What is a slowly changing dimension?**
A dimension whose attributes change over time (e.g., a customer moves to a new city). **Type 1:** overwrite the old value (lose history). **Type 2:** add a new row with `valid_from/valid_to` dates (preserve history). **Type 3:** add a column for the previous value (limited history).

**14. Explain idempotency in data pipelines.**
An idempotent pipeline produces the same result whether you run it once or ten times. Achieved via `MERGE`/upsert instead of `INSERT`, partition overwrite instead of append, or deduplication on a natural key. Critical for safe retries after failures.

**15. What is predicate pushdown?**
Pushing filter conditions down to the storage layer so only matching data is read. Instead of reading all 100 columns and filtering in memory, the engine tells the file reader "only give me rows where `status = 'active'`." Columnar formats (Parquet) support this via min/max stats in row group metadata.

**16. Hot partition — what is it and how to fix it?**
A partition that receives disproportionately more reads or writes than others, creating a bottleneck. Common in Kafka (one key gets most events) or databases (one shard gets most queries). Fix: add a salt/random suffix to the partition key, use composite keys, or pre-split partitions.

**17. What is a compacted topic in Kafka?**
A topic where Kafka retains only the **latest value per key**, deleting older records with the same key. Used for changelog / state topics — e.g., user profile updates. Unlike time-based retention, compaction is key-based: every key is guaranteed to have at least its latest value.

**18. Event-time vs processing-time?**
**Event-time:** when the event actually occurred (embedded timestamp in the record). **Processing-time:** when the system processes the event. Event-time is correct for analytics (handles late/out-of-order data) but requires watermarks. Processing-time is simpler but inaccurate when events arrive late.

**19. What is a feature store?**
A centralized system for managing ML features. It has two components: an **offline store** (data lake — for batch training) and an **online store** (Redis/DynamoDB — for low-latency inference). It ensures training and serving use the same feature definitions, preventing train-serve skew.

**20. What is data lineage and why does it matter?**
Data lineage tracks how data flows from source to destination — which tables feed which tables, through which transformations. It matters for: debugging (trace a bug to its root cause), impact analysis (what breaks if I change this table?), compliance (prove where PII flows), and trust (analysts can verify data provenance).

---

## Screen 5: Key Takeaways & The System Design Framework

### The 6-Step Framework for Any Data Engineering System Design Interview

Memorize this. Internalize it. Use it for every question. The steps should feel as natural as breathing.

---

**Step 1: Clarify Requirements (3–5 minutes)**

Never start drawing. Start asking.

- **Functional:** What data sources? What consumers? What queries/reports/ML models?
- **Non-Functional:** What's the latency requirement (batch/near-real-time/real-time)? What's the data volume (GB/day or TB/day)? What's the SLA for freshness? Any compliance requirements (PII, GDPR)?
- **Scale:** How many users/events/transactions per second? What's the growth rate?

*Signal to the interviewer: "Before I design anything, let me clarify a few things about scale and latency requirements."*

**Step 2: High-Level Architecture (5–7 minutes)**

Draw the major components and data flow arrows. Use the canonical layers:

```
Sources → Ingestion → Storage → Processing → Serving → Consumers
                         ↕
                    Orchestration
                         ↕
                    Governance & Monitoring
```

Name specific technologies for each layer, but explain *why* you chose them. "I'd use Kafka for ingestion because we need real-time and it handles 50K events/sec easily" is better than just writing "Kafka."

**Step 3: Data Model (3–5 minutes)**

Show the key schemas. For a data platform question:
- What does the Bronze/raw event look like?
- What does the Silver/cleaned table look like?
- What does the Gold/serving table look like?

For a system design question:
- What's the primary entity? What are the key attributes?
- How is it partitioned? What's the partition key?
- What file format and why? (Parquet for analytics, Avro for streaming, JSON for flexibility)

**Step 4: Deep Dive (10–15 minutes)**

Pick 2–3 critical components and go deep. The interviewer will often guide you here. Common deep-dive areas:

- **Ingestion:** CDC vs batch, exactly-once semantics, schema evolution
- **Processing:** Spark optimizations, Flink stateful processing, join strategies
- **Data Quality:** validation rules, freshness monitoring, anomaly detection
- **Serving:** query performance, caching, concurrency scaling

This is where you differentiate yourself. Show you understand the *internals*, not just the APIs.

**Step 5: Failure Handling (3–5 minutes)**

Proactively address: *"Here's what happens when things go wrong."*

- What if the source goes down? (Kafka retention buffers, retry logic)
- What if the processing job fails? (Idempotent writes, checkpoint recovery)
- What if bad data enters the system? (Quality gates, DLQ, time travel rollback)
- How do you monitor? (SLIs: freshness, volume, completeness. SLOs with alerting.)

**Step 6: Cost & Scale (2–3 minutes)**

Show you think about the real world:

- How does this scale to 10x volume? (Horizontal scaling, partitioning strategy)
- What are the cost drivers? (Storage vs compute vs network egress)
- Where would you optimize first? (Spot instances, storage tiering, query optimization)
- What's the growth plan? (Start with managed services, graduate to self-hosted at scale)

---

### 10 Golden Rules for DE System Design Interviews

1. **Ask before you draw.** Spend 3–5 minutes on requirements. This is the #1 differentiator between mid and senior candidates.

2. **Name your technologies and justify them.** "Kafka because we need ordered, durable, replayable event streaming at 50K TPS" — not just "a message queue."

3. **State your latency budget.** Whether it's 200ms for fraud detection or 6 AM deadline for a dashboard refresh — make the constraint explicit and design around it.

4. **Show the data model.** Interviewers want to see schemas, partition keys, and format choices. A system design without a data model is a system design without substance.

5. **Talk about failure before they ask.** Proactively say: "Let me address what happens when this component fails." This signals operational maturity.

6. **Quantify everything.** "50K events/sec × 1KB = 50MB/sec throughput → Kafka with 32 partitions handles this easily." Numbers build credibility.

7. **Discuss tradeoffs, not just choices.** "I chose Flink over Spark Structured Streaming because we need sub-second latency. The tradeoff is operational complexity — Flink requires more expertise to manage state and checkpoints."

8. **Layer in governance.** After the core architecture, spend 60 seconds on: data quality checks, access control, lineage, and monitoring. This separates senior from mid-level.

9. **Don't over-engineer.** If the requirement is daily batch with 10GB/day, don't design a Flink cluster with exactly-once semantics. Match complexity to requirements.

10. **Practice the whiteboard.** In real interviews, you're drawing and talking simultaneously. Practice explaining your design out loud while sketching on paper or a whiteboard tool. The framework above should take 35–40 minutes, leaving 5–10 minutes for Q&A.

---

### The Final Word

You've survived the Gauntlet. You've designed fraud detection systems, debugged Kafka broker failures at 3 AM, argued both sides of batch vs streaming, and answered 20 rapid-fire questions without flinching.

Here's the truth: **no one expects you to have every answer memorized.** What interviewers are looking for is your *framework* — how you break down ambiguity, how you reason about tradeoffs, and how you design for the real world where things fail, data is messy, and budgets are finite.

The six-step framework is your skeleton key. Requirements → Architecture → Data Model → Deep Dive → Failure Handling → Cost & Scale. Use it for every question. Adapt it to every scenario. Make it second nature.

Now close this document, open a blank whiteboard, and design a system from scratch. Time yourself. Talk out loud. That's how you get ready.

**You've got this.** 🔥
