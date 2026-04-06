# DE System Design Problems

> **Module 3** — Real-world system design problems you will face in Data Engineering interviews. Each screen walks through a complete design, from requirements to architecture to tradeoffs, exactly the way you should present in an interview.

---

## Screen 1: Uber Surge Pricing Pipeline

### The Problem

Design a real-time data pipeline that powers Uber's surge pricing. The system must continuously calculate price multipliers for every geographic zone based on real-time supply and demand, blended with historical demand patterns.

### Demand & Supply Signals

The pricing engine needs two categories of real-time signals:

**Demand signals (ride requests):**
- Ride requests per geographic area per minute
- Destination hotspots (concerts, airports, stadiums)
- App opens without booking (latent demand)
- Scheduled ride requests for upcoming hours

**Supply signals (available drivers):**
- Active drivers per area (GPS pings every 4 seconds)
- Driver ETA to the zone (drivers in adjacent zones)
- Driver shift patterns (predicted supply drop-off)
- Ride completions about to free up a driver

### Geo-Spatial Partitioning with H3 Hexagons

You cannot compute supply/demand for arbitrary lat/long points — you need a spatial index. Uber uses **H3**, a hierarchical hexagonal grid system developed internally and open-sourced.

**Why H3?**
- Hexagons tile the earth's surface uniformly (unlike square grids, which distort near poles)
- Each hexagon has **6 equidistant neighbors** (squares have 4 edge + 4 corner neighbors at different distances)
- Hierarchical: resolution 7 hexagons (~5.16 km² each) nest cleanly inside resolution 6 hexagons (~36.13 km²)
- A single `h3_index` integer encodes the location — perfect for `GROUP BY` aggregations

For surge pricing, **resolution 7** (~1.2 km edge length) is the sweet spot: granular enough to detect a stadium emptying, broad enough to have meaningful supply counts.

### Architecture: Streaming + Batch Merge

```
┌─────────────────── REAL-TIME PATH ───────────────────┐
│                                                       │
│  Ride Requests ──►  Kafka  ──► Flink Streaming Job   │
│  Driver GPS    ──►  Kafka  ──►  (tumbling windows    │
│  App Events    ──►  Kafka  ──►   60s per H3 hex)     │
│                                       │               │
│                                       ▼               │
│                              Real-Time Features       │
│                          (demand/supply per hex)      │
│                                       │               │
└───────────────────────────────────────┼───────────────┘
                                        │
                                        ▼
                                ┌───────────────┐
                                │   Pricing     │
                                │   Service     │◄──── gRPC API
                                │  (merge +     │      (rider app
                                │   compute     │       queries)
                                │   multiplier) │
                                └───────┬───────┘
                                        ▲
                                        │
┌───────────────────────────────────────┼───────────────┐
│                                       │               │
│                              Baseline Pricing +       │
│                              Demand Predictions       │
│                                       ▲               │
│                                       │               │
│  Historical  ──► Spark Batch ──► ML Model ──► Feature │
│  Ride Data       (daily)         (Prophet/   Store     │
│  Weather API                      XGBoost)  (Redis)   │
│  Events Cal                                           │
│                                                       │
└─────────────────── BATCH PATH ────────────────────────┘
```

**Flink streaming job details:**
- **Tumbling windows** of 60 seconds, keyed by `h3_index`
- Computes: `request_count`, `active_drivers`, `supply_demand_ratio`
- Emits per-hex features to the pricing service via internal Kafka topic or direct gRPC push

**Batch pipeline details:**
- Daily Spark job trains/updates an ML model predicting baseline demand per hex per hour
- Incorporates: day-of-week seasonality, holidays, weather, local events
- Predictions written to a feature store (Redis) for sub-millisecond lookup by the pricing service

**Pricing service merge logic:**
```
surge_multiplier = f(
    real_time_supply_demand_ratio,   # from Flink (weight: 0.7)
    predicted_baseline_demand,        # from batch ML (weight: 0.2)
    hard_caps_and_regulatory_rules    # business rules (weight: 0.1)
)
```

### Key Tradeoffs

| Tradeoff | Option A | Option B | Uber's Choice |
|---|---|---|---|
| Window size | 30s (responsive) | 120s (stable) | 60s (balanced) |
| Geo resolution | Res 8 (tiny hexes) | Res 6 (large) | Res 7 (city-block) |
| Pricing staleness | Sub-second update | 1-min batched | 1-min (stability) |
| Cold-start areas | Default multiplier 1.0 | Nearest-neighbor | Nearest-neighbor avg |

**Cold-start problem:** New areas with no historical data use the weighted average surge of the 6 neighboring H3 hexagons, falling back to city-wide baseline.

### 💡 Interview Insight

> This problem tests your ability to combine **streaming + batch + geo-spatial**. Show the interviewer you understand H3 hexagons for geo-partitioning — mention "hierarchical hexagonal grid, resolution 7, each hex has 6 equidistant neighbors." Then clearly draw the two data paths (real-time Flink, batch Spark/ML) merging at the pricing service. Most candidates only design one path. You design both.

---

## Screen 2: E-Commerce Data Warehouse

### The Problem

Design a data warehouse for an e-commerce company (think Amazon/Shopify scale). The warehouse must support daily business reporting, customer analytics, and real-time inventory visibility.

### Source Systems

| Source System | Type | Volume | Ingestion |
|---|---|---|---|
| Orders DB (PostgreSQL) | OLTP | 5M orders/day | CDC (Debezium) |
| Product Catalog | Microservice API | 10M SKUs | Nightly full sync |
| Customer CRM | SaaS (Salesforce) | 50M customers | CDC + API pulls |
| Inventory WMS | Real-time DB | 500K updates/day | CDC (Debezium) |
| Clickstream | Event stream | 2B events/day | Kafka direct |

### Dimensional Model — Star Schema

**The most critical decision: define the GRAIN.**

`fct_orders` grain: **one row per order line item.** Each row represents one product in one order.

```
                          ┌──────────────┐
                          │  dim_date    │
                          │──────────────│
                          │ date_key (PK)│
                          │ full_date    │
                          │ day_of_week  │
                          │ month        │
                          │ quarter      │
                          │ fiscal_year  │
                          │ is_holiday   │
                          └──────┬───────┘
                                 │
┌──────────────┐    ┌────────────┴───────────┐    ┌──────────────────┐
│ dim_customer │    │      fct_orders        │    │   dim_product    │
│──────────────│    │────────────────────────│    │──────────────────│
│ customer_key │◄───│ customer_key (FK)      │───►│ product_key (PK) │
│ customer_id  │    │ product_key  (FK)      │    │ product_id       │
│ name         │    │ date_key     (FK)      │    │ product_name     │
│ email        │    │ geo_key      (FK)      │    │ category         │
│ segment      │    │ order_id               │    │ subcategory      │
│ city         │    │ order_line_id          │    │ brand            │
│ state        │    │ quantity               │    │ unit_price       │
│ valid_from   │    │ unit_price             │    │ valid_from       │
│ valid_to     │    │ gross_amount           │    │ valid_to         │
│ is_current   │    │ discount_amount        │    │ is_current       │
└──────────────┘    │ net_amount             │    └──────────────────┘
                    │ shipping_cost          │
                    │ tax_amount             │    ┌──────────────────┐
                    │ payment_method         │    │  dim_geography   │
                    │ order_status           │───►│──────────────────│
                    └────────────────────────┘    │ geo_key (PK)     │
                                                  │ city             │
                                                  │ state            │
                                                  │ country          │
                                                  │ region           │
                                                  └──────────────────┘
```

### SCD Type 2 — Tracking History

**Slowly Changing Dimension Type 2** keeps a full history of changes by adding `valid_from`, `valid_to`, and `is_current` columns.

**Example: customer moves from Dallas to Austin:**

| customer_key | customer_id | name | city | valid_from | valid_to | is_current |
|---|---|---|---|---|---|---|
| 1001 | C-555 | Jane Doe | Dallas | 2023-01-15 | 2024-06-30 | false |
| 1002 | C-555 | Jane Doe | Austin | 2024-07-01 | 9999-12-31 | true |

The `customer_key` (surrogate key) is what `fct_orders` references. An order placed in March 2024 joins to key `1001` (Dallas). An order placed in August 2024 joins to key `1002` (Austin). This lets you answer: "What was the customer's city **at the time of purchase**?"

Apply SCD Type 2 to `dim_product` as well — track price changes, category reclassifications, and brand acquisitions.

### Aggregation Marts

Built on top of the star schema using dbt:

- **`mart_daily_revenue`**: date, channel, category → total_revenue, order_count, avg_order_value
- **`mart_category_performance`**: category, month → revenue, units_sold, return_rate, margin
- **`mart_customer_cohort`**: signup_month, months_since_signup → active_customers, revenue, retention_rate

### Real-Time Inventory Layer

Alongside the batch warehouse, a **real-time inventory view** for the website:

```
Inventory DB ──► Debezium CDC ──► Kafka ──► Flink ──► Materialized View (Redis)
                                                        │
                                              "SKU-12345: 47 units in stock"
                                              (sub-second freshness)
```

The batch warehouse gets daily inventory snapshots for trend analysis. The real-time path powers the "Only 3 left!" badge on the product page.

### Partitioning Strategy

- `fct_orders`: partition by `order_date` (daily or monthly partitions depending on volume)
- Dimension tables: no partitioning needed (small lookup tables, full scan is fast)
- `mart_daily_revenue`: partition by `date` for efficient time-range queries
- Clickstream raw: partition by `event_date` and `event_hour` (high volume)

### 💡 Interview Insight

> **Always define the GRAIN of your fact table first.** This is the most common mistake candidates make — they jump to schema columns without stating the grain. Say it out loud: "The grain of fct_orders is one row per order line item." Then explain why: it lets you aggregate to order-level, customer-level, or product-level. The grain determines everything.

---

## Screen 3: Recommendation System Data Pipeline

### The Problem

Design the end-to-end data pipeline that powers a recommendation engine like Netflix's "Top Picks for You" or Amazon's "Customers who bought this also bought."

The pipeline is **not** about the ML model itself — it is about the **data infrastructure** that feeds features to training and serves features to inference.

### Feature Engineering

Features fall into three categories:

| Category | Examples | Compute | Freshness |
|---|---|---|---|
| User behavior | Total views, purchase history, avg rating, genre preferences | Batch (daily) | 24h stale OK |
| Item attributes | Category, price tier, popularity score, release recency | Batch (daily) | 24h stale OK |
| Session/real-time | Last 5 clicks, current cart, time-on-page, scroll depth | Streaming | Must be live |
| Interaction | Co-purchase matrix, collaborative filtering embeddings | Batch (weekly) | 7d stale OK |

### The Dual Feature Store Architecture

This is the centerpiece of the design. You need **two stores** because training and serving have fundamentally different requirements:

```
┌──────────────────────── TRAINING PATH ────────────────────────────┐
│                                                                    │
│  Raw Events ──► Spark Batch ──► Feature Computation ──► OFFLINE   │
│  (S3/Delta)     (daily DAG)     (user embeddings,       STORE     │
│                                  item popularity,      (Delta on  │
│                                  co-occurrence)         S3/GCS)   │
│                                                           │       │
│                                                           ▼       │
│                                                    Training Job   │
│                                                    (pull labeled  │
│                                                     feature sets) │
└───────────────────────────────────────────────────────────────────┘

┌──────────────────── SERVING PATH ─────────────────────────────────┐
│                                                                    │
│  Click Events ──► Kafka ──► Flink ──► Session Features ──► ONLINE │
│  Cart Events                (real-time                     STORE   │
│  Page Views                  aggregation)                 (Redis/  │
│                                                          DynamoDB) │
│                                         ▲                    │     │
│            Batch features sync ─────────┘                    ▼     │
│            (daily: copy user/item features               Model     │
│             from offline → online store)                 Serving   │
│                                                         (< 50ms)  │
└───────────────────────────────────────────────────────────────────┘
```

**Offline feature store (for training):**
- Storage: Parquet/Delta Lake on S3
- Access pattern: large batch reads for training datasets
- Time-travel: retrieve features **as they were** at a historical point in time (critical for avoiding label leakage)
- Example: "Give me user_123's features as of 2024-01-15 to pair with their Jan 16 purchase label"

**Online feature store (for serving):**
- Storage: Redis or DynamoDB
- Access pattern: key-value lookup by `user_id` or `item_id`, < 10ms p99
- Updated: real-time features via Flink, batch features synced daily from offline store
- Example: "User_123 just opened the app → fetch their feature vector in 5ms → run inference → return recommendations"

**Feature registry (metadata layer):**
- Schema definitions for every feature (name, type, description, owner)
- Versioning: feature v1 vs v2 with backward compatibility
- Lineage: which raw data sources feed which features
- Monitoring: data quality checks, drift detection, freshness SLAs

### The Train-Serve Skew Problem

The most insidious bug in ML systems: if the features used during **training** differ from those used during **serving**, the model performs poorly in production despite great offline metrics.

**Causes:** different code paths compute the same feature differently (e.g., Spark SQL vs Python), different data freshness (training on T-1 data, serving on real-time data), schema drift.

**Solution:** compute features **once** in a shared feature pipeline and store them in the feature store. Both training and serving read from the **same** store, same code, same definitions.

### End-to-End Serving Flow

```
User opens app
    → API Gateway
    → Recommendation Service
        → Fetch user features from online store (Redis)     ~5ms
        → Fetch candidate item features from online store    ~5ms
        → Run model inference (TensorFlow Serving)          ~20ms
        → Apply business rules (filter out-of-stock, etc.)  ~2ms
        → Return top 20 recommendations
    ← Response to client                              Total: ~35ms
```

### 💡 Interview Insight

> The key insight is the **DUAL feature store** — offline for training, online for serving. Show you understand the **train-serve skew** problem: "If training and serving compute features differently, model accuracy degrades silently in production. The feature store ensures both paths use identical feature definitions." This answer separates senior candidates from juniors.

---

## Screen 4: Log Analytics System at Scale

### The Problem

Design a log analytics system that handles **1M+ events per second** from thousands of microservices. Engineers need to search logs for debugging, dashboards need real-time error rate metrics, and compliance requires long-term retention.

### Collection Layer

**Log agents** run on every application server:

- **Fluent Bit** (lightweight, C-based) on containers and edge nodes
- **Fluentd** (Ruby, plugin-rich) on heavier application servers
- Structured logging enforced: every log line is JSON with mandatory fields

```json
{
  "timestamp": "2025-03-15T14:23:01.456Z",
  "service": "payment-service",
  "level": "ERROR",
  "trace_id": "abc-123-def",
  "message": "Payment gateway timeout",
  "duration_ms": 30012,
  "customer_id": "C-9988",
  "error_code": "PGW_TIMEOUT"
}
```

Structured logging is non-negotiable — unstructured text logs are impossible to aggregate at scale.

### Transport Layer — Kafka as a Buffer

Kafka sits between collection and processing for three critical reasons:

1. **Decoupling:** log agents don't need to know about downstream consumers
2. **Spike absorption:** a deploy-triggered error storm won't overwhelm Elasticsearch
3. **Replay:** re-process logs if you discover a parsing bug

**Topic strategy:** one topic per log level (`logs-error`, `logs-warn`, `logs-info`) or one unified topic with consumer-side filtering. The per-level approach lets you give ERROR logs priority processing and separate retention policies.

### Processing Layer

**Flink streaming jobs:**
- **Enrichment:** add geo-IP from source IP, parse user-agent strings, resolve service names from container IDs
- **Aggregation:** compute error rates per service per minute (for alerting dashboards)
- **Anomaly detection:** compare current error rate to rolling 7-day baseline, fire alert if > 3 standard deviations

### Storage Tier Design — Hot / Warm / Cold

This is the most important architectural decision. Not all logs deserve the same storage tier:

| Tier | Storage | Retention | Query Latency | Use Case | Cost |
|---|---|---|---|---|---|
| **Hot** | Elasticsearch/OpenSearch | 7 days | Sub-second | Live debugging, search | $$$$  |
| **Warm** | S3 + Parquet | 30–90 days | 5–30 seconds | Investigation, trends | $$ |
| **Cold** | S3 Glacier | 1–7 years | Minutes–hours | Compliance, audit | $ |

**Hot tier (Elasticsearch):**
- Daily index rotation: `logs-payment-2025.03.15`
- Shard sizing: target **30–50 GB per shard** (too small = overhead, too large = slow recovery)
- Replica count: 1 replica for durability (2 copies total)
- ILM (Index Lifecycle Management): auto-rollover at 50GB or 1 day, delete after 7 days

**Warm tier (S3 + Parquet):**
- Flink writes Parquet files partitioned by `date/service/level`
- Queryable via **Athena** (serverless SQL) or **Trino** (interactive cluster)
- Schema-on-read: add new fields without migration
- Compressed: ~10:1 compression ratio vs raw JSON

**Cold tier (S3 Glacier):**
- Lifecycle rule: move from S3 Standard to Glacier after 90 days
- Used only for compliance and legal hold
- Retrieval: bulk retrieval in 5–12 hours (acceptable for audits)

### Scale Considerations

At 1M events/sec:
- **Kafka:** 20 partitions per topic, 3 brokers minimum, 100MB/s throughput per broker
- **Elasticsearch:** 15–20 data nodes, SSDs for hot tier, r5.xlarge instances
- **Spike handling:** Kafka absorbs the spike (it can buffer hours of data); Elasticsearch consumers auto-scale or slow down via consumer lag monitoring
- **Cost optimization:** only index fields you search on in Elasticsearch; store the full event in S3

### 💡 Interview Insight

> Show **tiered storage thinking** — not all logs need the same latency. Say: "I'd keep 7 days hot in Elasticsearch for interactive debugging, 90 days warm in Parquet on S3 queryable via Athena for investigations, and 7 years cold in Glacier for compliance." Then mention shard sizing (30–50 GB target) and daily index rotation. This shows you've operated Elasticsearch at scale, not just read about it.

---

## Screen 5: Event-Driven Fintech Architecture

### The Problem

Design a transaction processing and analytics platform for a fintech company (think Stripe or Square). The system must handle payments with financial-grade correctness, provide real-time fraud detection, and maintain an immutable audit trail for regulators.

### Event Sourcing — The Foundation

In fintech, you **never update or delete** a record. Every state change is an **immutable event** appended to a log:

```
Event 1: TransactionCreated    { txn_id: T-100, amount: $50.00, merchant: "Coffee Shop", ts: 14:00:01 }
Event 2: TransactionAuthorized { txn_id: T-100, auth_code: "A-999", ts: 14:00:02 }
Event 3: TransactionSettled    { txn_id: T-100, settled_amount: $50.00, ts: 14:01:30 }
---
Event 4: TransactionCreated    { txn_id: T-101, amount: $500.00, merchant: "Electronics", ts: 14:02:00 }
Event 5: TransactionDeclined   { txn_id: T-101, reason: "FRAUD_SUSPECT", ts: 14:02:00 }
```

The current account balance is a **derived view** — replay all events from the beginning and you reconstruct it perfectly. This is why banks can always reconcile.

### CQRS — Separating Reads from Writes

**Command Query Responsibility Segregation** uses different models for writes and reads:

```
                WRITE SIDE                          READ SIDE
                                                    
  API Request ──► Validate ──► Append to    ──►  Projection ──► Materialized
  (command)       business     Event Store        (Flink/       Views:
                  rules        (Kafka +           consumer)     • Account Balance
                               append-only                      • Transaction History
                               DB)                              • Monthly Statement
                                                                • Analytics Cube
```

**Write model:** optimized for appending events. Kafka topics partitioned by `account_id` ensure ordering per account.

**Read model:** materialized views optimized for specific query patterns. Account balance is a simple running sum. Transaction history is a time-sorted list. Analytics cubes aggregate across accounts.

### Kafka as the Event Backbone

- **Partitioning by `account_id`:** guarantees all events for one account are in the same partition → ordered processing per account
- **Compacted topics:** for latest-state views (e.g., account profile), Kafka keeps only the latest event per key
- **Retention:** infinite retention for the event store topic (this IS the source of truth)
- **Replication factor 3:** financial data cannot be lost

### Real-Time Fraud Detection

```
Transaction Event ──► Flink Streaming Job ──► Fraud Score ──► Decision
                      │                                        │
                      ├─ Velocity check: > 5 txns in 1 min?   ├─ ALLOW
                      ├─ Geo anomaly: NYC → Tokyo in 30 min?  ├─ REVIEW
                      ├─ Amount deviation: 10x avg purchase?   └─ BLOCK
                      └─ Device fingerprint: new device?
                      
Latency budget: < 100ms from event to decision
```

**Feature computation for fraud:**
- **Velocity:** count of transactions per account in sliding 1-min, 5-min, 1-hour windows
- **Geo-anomaly:** distance between current and last transaction location ÷ time elapsed
- **Amount deviation:** current amount vs 30-day rolling average for this account
- **Device/IP signals:** new device, VPN detected, mismatched billing country

### Regulatory Compliance

| Requirement | Implementation |
|---|---|
| Immutable audit trail | Event store (append-only, no deletes) |
| Data retention (7 yrs) | Kafka infinite retention + S3 archival |
| Encryption at rest | AES-256 on all storage tiers |
| Encryption in transit | TLS 1.3 for all service communication |
| PII masking in analytics | Hash/tokenize PII before writing to warehouse |
| Access logging | Every data access logged with user, timestamp, query |

### Exactly-Once Processing

Financial systems **cannot** double-charge or lose a transaction. End-to-end exactly-once requires:

1. **Idempotency keys:** every API request includes a client-generated UUID. If retried, the system detects the duplicate and returns the original result.
2. **Kafka transactions:** producer writes to multiple topics/partitions atomically. Consumer commits offsets in the same transaction as the database write.
3. **Database upserts:** `INSERT ... ON CONFLICT (txn_id) DO NOTHING` — safe to replay.

### 💡 Interview Insight

> **Event sourcing + CQRS is the gold standard answer for fintech.** Say: "I'd use event sourcing because financial regulations require an immutable audit trail — every state change is an append-only event. CQRS separates the write path (optimized for appending events) from the read path (materialized views optimized for queries like account balance)." Then emphasize **exactly-once processing** with idempotency keys. This shows you understand that financial correctness is non-negotiable.

---

## Screen 6: Data Mesh & SCD Handler at Scale

### Data Mesh Architecture

Data Mesh is an **organizational paradigm**, not a technology. It addresses the failure mode of centralized data teams becoming bottlenecks.

**Four principles:**

**1. Domain-Oriented Ownership**
Instead of one central data team owning all pipelines, each business domain team (Payments, Inventory, Customer) owns their data end-to-end — from ingestion to transformation to serving. The Payments team builds and operates the `payments` data product because they understand payment business logic better than any central team ever could.

**2. Data as a Product**
Each domain publishes data products with product-level quality:
- **Discoverable:** listed in a central data catalog with description, schema, SLAs
- **Addressable:** standard URIs (`s3://data-products/payments/fct_transactions/v2/`)
- **Self-describing:** schema registry, column descriptions, data quality metrics
- **Interoperable:** standard formats (Parquet, Delta), standard access protocols (SQL, API)
- **Trustworthy:** published SLAs (freshness, completeness, accuracy)

**3. Self-Serve Data Platform**
A dedicated platform team provides centralized tooling as a service:
- Compute: Spark, Flink clusters (shared infrastructure)
- Orchestration: Airflow as a managed service
- Transformation: dbt with shared macros and packages
- Storage: S3/GCS with standardized layouts
- Observability: data quality monitoring, lineage tracking

Domain teams use these tools without managing infrastructure.

**4. Federated Computational Governance**
Global policies enforced automatically, local autonomy for everything else:
- **Global:** PII classification and handling rules, naming conventions, retention policies, access control patterns
- **Local:** teams choose their own scheduling, transformation logic, internal data models

**Challenges (discuss these in interviews — shows maturity):**
- Organizational change management is harder than any technology migration
- Data duplication across domains (Payments and Customer both store customer data)
- Cross-domain joins require contract negotiation between teams
- Platform team sizing: too small → teams are blocked; too large → becomes the new centralized team

### SCD Handler at Scale

**Requirements:** Track 500M dimension records (e.g., product catalog) with daily changes affecting ~2% of records (10M changes/day).

**Approach: Spark + Delta Lake MERGE**

```sql
MERGE INTO dim_product_scd2 AS target
USING staged_changes AS source
ON target.product_id = source.product_id AND target.is_current = true

-- Close the old record
WHEN MATCHED AND (target.price != source.price OR target.category != source.category)
THEN UPDATE SET
    valid_to = source.change_date,
    is_current = false

-- Insert the new version
WHEN NOT MATCHED
THEN INSERT (product_key, product_id, name, price, category, valid_from, valid_to, is_current)
VALUES (generate_surrogate_key(), source.product_id, source.name, source.price, source.category,
        source.change_date, '9999-12-31', true);
```

**Note:** Delta Lake's MERGE executes the matched update and the insert for new version rows in one atomic operation. For the "insert new current version" of changed rows, a second INSERT statement runs for records that were just closed.

**Optimizations:**
- **Partition by hash of `product_id`** (modulo 256): ensures even distribution regardless of product ID patterns; avoids hot partitions from sequential IDs
- **Incremental processing:** CDC from the source system delivers only changed records; the MERGE only touches affected partitions
- **Z-ordering on `product_id`:** co-locates data for the same product across files, making lookups fast
- **Delta Lake file compaction:** periodic `OPTIMIZE` to merge small files from daily MERGEs

**Serving time-travel queries:**
```sql
-- What was the product's price on Black Friday 2024?
SELECT * FROM dim_product_scd2
WHERE product_id = 'SKU-12345'
  AND valid_from <= '2024-11-29'
  AND valid_to   >  '2024-11-29';

-- Or with Delta Lake time travel:
SELECT * FROM dim_product_scd2 VERSION AS OF '2024-11-29'
WHERE product_id = 'SKU-12345' AND is_current = true;
```

### 💡 Interview Insight

> **Data Mesh is a HOT topic.** Frame it as an organizational pattern, not a technology: "Data Mesh solves the bottleneck of centralized data teams by giving domain teams ownership of their data products, supported by a self-serve platform and federated governance." Then discuss the real challenge: "The hardest part isn't Spark or dbt — it's convincing the Payments team to treat their data with the same rigor as their APIs." For SCD, always mention **incremental processing** — nobody should full-scan 500M records daily.

---

## Screen 7: Cross-Cutting Concepts — The Building Blocks

These concepts appear in **every** system design interview. Master them once, apply them everywhere.

### Exactly-Once Processing

The holy grail of distributed data processing. True end-to-end exactly-once requires **all three**:

1. **Idempotent producers:** assign a unique ID to every message; the broker deduplicates on retry
2. **Transactional consumers:** commit the consumer offset AND the downstream write in one atomic transaction
3. **Idempotent sinks:** use UPSERT/MERGE at the destination so replayed messages don't create duplicates

**Reality check:** end-to-end exactly-once across system boundaries is extremely hard. In practice, most systems achieve **effectively once** through idempotency — it's safe to process a message multiple times because the result is the same.

### Data Partitioning Strategies

| Strategy | Best For | Watch Out |
|---|---|---|
| **Hash partitioning** | Even distribution | Hot keys (celebrity accounts) |
| **Range partitioning** | Time-series data | Recent partitions get all writes |
| **Composite keys** | Multi-tenant systems | Complexity in query routing |
| **Salting** | Fixing hot partitions | Complicates reads (must fan-out) |

**Hot partition fix with salting:**
```python
# Problem: account "uber_eats" generates 50% of all events
# Solution: salt the partition key
partition_key = f"{account_id}_{random.randint(0, 9)}"
# Now "uber_eats" spreads across 10 partitions
# Read query must fan out to all 10 and merge results
```

### Backfill Strategies

When you discover a bug or add a new feature, you need to reprocess historical data:

- **Full reprocess:** re-run the entire pipeline from raw data. Simple but expensive. Use for schema changes or logic rewrites.
- **Incremental backfill:** reprocess only affected partitions. Use partition-level reruns in Airflow (`airflow dags backfill -s 2024-01-01 -e 2024-03-01`).
- **Kafka replay:** reset consumer group offset to a past timestamp. Reprocess streaming data without re-ingesting.
- **Blue-green tables:** write backfilled data to a new table, validate, then swap. Zero downtime, easy rollback.

### Idempotent Pipelines

**Same input → same output, no matter how many times you run it.**

Techniques:
- **MERGE/UPSERT:** `INSERT ON CONFLICT UPDATE` — safe to replay
- **Partition overwrite:** `INSERT OVERWRITE PARTITION (date='2024-01-15')` — replaces the entire partition
- **Deduplication:** `SELECT DISTINCT` on `(event_id, timestamp)` before writing
- **Deterministic processing:** avoid `now()`, `random()`, or non-deterministic UDFs in transforms

### Late-Arriving Data

Data arrives after the processing window has closed. Three approaches:

1. **Watermarks:** define how long to wait (e.g., "process window closes 5 minutes after wall clock passes window end"). Events arriving within the watermark are included normally.
2. **Allowed lateness:** events arriving after the watermark but within an "allowed lateness" period are processed and emitted as updated results. Flag them as late for downstream awareness.
3. **Reprocessing:** for data arriving days late, trigger a re-run of the affected partition. The idempotent pipeline overwrites the old result.

### Schema Evolution at Scale

| Compatibility | Rule | Use Case |
|---|---|---|
| **Backward** | New schema reads old data | Adding optional columns |
| **Forward** | Old schema reads new data | Consumers update slowly |
| **Full** | Both directions | Maximum flexibility |

**Best practices:**
- Schema registry (Confluent, AWS Glue) enforces compatibility on every write
- New columns always have defaults (never break existing readers)
- Never rename or delete columns — add new ones, deprecate old ones
- Migration scripts for breaking changes (rare, planned, communicated)

### The Interview Design Framework

Use this 6-step framework for **every** system design question:

1. **Clarify requirements:** functional (what does it do?), non-functional (scale, latency, durability), constraints (budget, team size, existing infra)
2. **High-level architecture:** boxes and arrows showing data flow from source to serving
3. **Data model:** schema design, grain definition, partitioning strategy, file format
4. **Deep-dive critical components:** pick 2-3 components the interviewer cares most about and go deep
5. **Failure handling:** what breaks? how do you recover? exactly-once? idempotency? alerting?
6. **Cost and scale:** back-of-envelope math (events/sec × event size × retention = storage), cost optimization levers

### 💡 Interview Insight

> These concepts come up in **EVERY** system design interview. When the interviewer asks "What if a message is processed twice?" — you say "idempotent pipeline: UPSERT at the sink, dedup on event_id." When they ask "What about late data?" — you say "watermarks with allowed lateness, plus partition-level reprocessing for very late arrivals." Drill these patterns until they're reflexive.

---

## Screen 8: Quiz — Test Your Design Thinking

**Q1: In Uber's surge pricing pipeline, why are H3 hexagons preferred over square grid cells for geo-spatial partitioning?**

- A) Hexagons use less storage than squares
- B) Hexagons have 6 equidistant neighbors, providing uniform adjacency analysis ✅
- C) Hexagons are natively supported by Kafka partitioning
- D) Hexagons are required by Flink's geo-spatial library

> **Answer: B** — The key advantage of hexagons is that all 6 neighbors are equidistant from the center. Square grids have 4 edge neighbors and 4 corner neighbors at different distances (1.0 vs 1.41x), which creates bias in spatial aggregation. This uniformity makes demand smoothing and neighbor-averaging calculations more accurate.

---

**Q2: When designing an e-commerce data warehouse, what should you define FIRST about the fact table?**

- A) The column data types
- B) The partitioning strategy
- C) The grain (what one row represents) ✅
- D) The surrogate key generation method

> **Answer: C** — The grain is the foundational decision. "One row per order line item" vs "one row per order" completely changes the schema, the dimensions you can attach, and the aggregations you can compute. Every other decision (columns, partitioning, keys) flows from the grain.

---

**Q3: What problem does a dual feature store (offline + online) solve in a recommendation system?**

- A) It reduces storage costs by compressing features
- B) It prevents train-serve skew by ensuring consistent feature definitions ✅
- C) It eliminates the need for real-time feature computation
- D) It replaces the need for a model registry

> **Answer: B** — Train-serve skew occurs when training and serving compute features differently (different code, different freshness, different logic). The dual feature store ensures both paths use the same feature definitions: offline store for training data, online store for serving, both populated by the same feature pipelines.

---

**Q4: In a fintech event-sourcing architecture, how is the current account balance derived?**

- A) Stored as a mutable field in the accounts table, updated on each transaction
- B) Computed by replaying all immutable events for that account from the event store ✅
- C) Cached in Redis and never recomputed
- D) Calculated nightly by a batch job from the data warehouse

> **Answer: B** — In event sourcing, the event store (append-only log of all state changes) is the source of truth. The current balance is a derived materialized view computed by replaying events. In practice, you maintain a materialized view that updates incrementally, but you can always rebuild it from scratch by replaying the event log — this is what makes event sourcing auditable and recoverable.

---

**Q5: Which combination achieves end-to-end exactly-once processing in a Kafka-based pipeline?**

- A) Consumer retries + at-least-once delivery
- B) Idempotent producer + transactional consumer + idempotent sink (UPSERT) ✅
- C) Increasing Kafka replication factor to 5
- D) Using a single Kafka partition to avoid ordering issues

> **Answer: B** — End-to-end exactly-once requires all three layers: the producer deduplicates retried messages (idempotent producer), the consumer commits offsets atomically with the downstream write (transactional consumer), and the sink handles replayed messages gracefully (UPSERT/MERGE). Missing any one layer breaks the guarantee.

---

## Screen 9: Key Takeaways

1. **Streaming + batch merge is the standard pattern** for real-time systems. The streaming path provides low-latency signals; the batch path provides historical context and ML predictions. Design both paths and show how they merge at the serving layer.

2. **H3 hexagonal grids** are the industry standard for geo-spatial partitioning. Remember: 6 equidistant neighbors, hierarchical resolutions, and a single integer index for GROUP BY. Resolution 7 (~5 km²) is the common starting point.

3. **Define the grain first, always.** Before drawing any schema, state: "The grain of this fact table is one row per ___." This single sentence drives every downstream decision about dimensions, measures, and aggregations.

4. **SCD Type 2 preserves history** through `valid_from`, `valid_to`, and `is_current` columns. At scale (500M+ records), use Delta Lake MERGE with incremental CDC processing — never full-scan when only 2% of records change.

5. **The dual feature store solves train-serve skew.** Offline store (S3/Delta) for training, online store (Redis/DynamoDB) for serving, both fed by the same feature pipelines. This is the #1 architecture pattern for production ML systems.

6. **Tiered storage (hot/warm/cold) optimizes cost vs. query speed.** Not all data deserves sub-second access. 7 days hot in Elasticsearch, 90 days warm in Parquet+Athena, years cold in Glacier. Always present this hierarchy in log analytics designs.

7. **Event sourcing + CQRS is the gold standard for fintech.** Immutable events provide the audit trail regulators require. CQRS separates the append-only write model from query-optimized read models. Exactly-once via idempotency keys + transactional consumers + UPSERT sinks.

8. **Data Mesh is an organizational pattern, not a technology.** Domain ownership, data as a product, self-serve platform, federated governance. The hardest challenge is people and culture, not tooling.

9. **Idempotent pipelines are non-negotiable.** Every pipeline you design should produce the same output regardless of how many times it runs. Techniques: MERGE/UPSERT, partition overwrite, deduplication on event_id, deterministic transformations.

10. **Use the 6-step design framework** in every interview: Clarify requirements → High-level architecture → Data model → Deep-dive components → Failure handling → Cost and scale. This structure keeps you organized and ensures you cover what interviewers look for.
