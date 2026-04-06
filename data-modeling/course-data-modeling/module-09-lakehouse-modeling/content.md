# Module 9: Modeling for the Lakehouse

---

## Screen 1: Medallion Architecture — The Layered Foundation

### Why Layers Matter

Every modern data platform eventually confronts the same chaos: raw data floods in from dozens of sources — Kafka streams, API dumps, CDC feeds, flat files — each with different schemas, quality levels, and arrival times. Without structure, you get a data swamp. The **Medallion Architecture** (popularized by Databricks, but universally applicable) solves this by organizing data into three progressive layers of quality: **Bronze**, **Silver**, and **Gold**.

The genius of Medallion isn't the concept of layers — data warehouses have had staging, integration, and presentation layers for decades. The genius is that it adapts these ideas for **lakehouse reality**: schema evolution, massive scale, mixed batch-and-streaming workloads, and open file formats. Each layer has a clear contract, and data gets progressively richer and more trustworthy as it flows through.

### The Full Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     MEDALLION ARCHITECTURE                              │
│                                                                         │
│  SOURCES              BRONZE              SILVER              GOLD      │
│  ───────              ──────              ──────              ────      │
│                                                                         │
│  ┌─────────┐     ┌──────────────┐   ┌──────────────┐   ┌────────────┐  │
│  │  Kafka  │────▶│  raw_orders  │──▶│  clean_orders│──▶│daily_sales │  │
│  │ Streams │     │  (append)    │   │  (deduped,   │   │ _summary   │  │
│  └─────────┘     │              │   │   typed)     │   └────────────┘  │
│                  └──────────────┘   └──────────────┘                    │
│  ┌─────────┐     ┌──────────────┐   ┌──────────────┐   ┌────────────┐  │
│  │  APIs   │────▶│ raw_customers│──▶│clean_customer│──▶│ customer   │  │
│  │  (JSON) │     │  (raw JSON)  │   │  (one row    │   │ _lifetime  │  │
│  └─────────┘     │              │   │   per cust)  │   │  _value    │  │
│                  └──────────────┘   └──────────────┘   └────────────┘  │
│  ┌─────────┐     ┌──────────────┐   ┌──────────────┐   ┌────────────┐  │
│  │   CDC   │────▶│ raw_products │──▶│clean_products│──▶│ product    │  │
│  │  Feeds  │     │  (all ops)   │   │  (current +  │   │ _perf_dash │  │
│  └─────────┘     │              │   │   history)   │   └──────┘  │
│                  └──────────────┘   └──────────────┘                    │
│                                                                         │
│  Schema-on-read ─────▶ Schema enforced ──────▶ Business-optimized      │
│  Append-only           Deduplicated            Star schemas / OBTs      │
│  Raw fidelity          Conformed types         Pre-aggregated           │
└─────────────────────────────────────────────────────────────────────────┘
```

### What Happens at Each Layer

**Bronze (Raw Ingestion)**
- **Schema:** Schema-on-read. Store data as-is from the source — JSON blobs, raw CSV rows, CDC payloads. Add only metadata columns: `_ingested_at`, `_source_file`, `_source_system`.
- **Write pattern:** Append-only. Never update, never delete. This is your immutable audit trail.
- **Grain:** Whatever the source gives you — one row per event, one row per CDC operation, one row per API response.
- **Retention:** Keep indefinitely (or as long as compliance requires). This is your time machine.

**Silver (Cleaned & Conformed)**
- **Schema:** Schema-on-write. Enforce data types, NOT NULL constraints, and valid value ranges. This is where you catch data quality problems.
- **Write pattern:** MERGE/upsert. Deduplicate, apply SCD logic, resolve late-arriving data.
- **Grain:** One row per business entity per version. `clean_customers` has one row per customer (or one per customer per version for SCD2). `clean_orders` has one row per order.
- **Transformations:** Type casting, deduplication, null handling, business key resolution, conforming codes (e.g., "US" vs "USA" vs "United States" → "US").

**Gold (Business-Level)**
- **Schema:** Purpose-built. Star schemas, One Big Tables (OBTs), or pre-aggregated summaries optimized for specific business use cases.
- **Write pattern:** Rebuild or incremental, depending on volume.
- **Grain:** Business-specific. `daily_sales_summary` has one row per store per day. `customer_lifetime_value` has one row per customer.
- **Consumers:** BI dashboards, ML feature stores, executive reports, ad-hoc analysts.

### Example Folder/Table Structures

```
lakehouse/
├── bronze/
│   ├── ecommerce/
│   │   ├── raw_orders/          -- partitioned by _ingested_date
│   │   ├── raw_customers/       -- partitioned by _ingested_date
│   │   └── raw_clickstream/     -- partitioned by _ingested_date
│   └── erp/
│       ├── raw_invoices/
│       └── raw_inventory/
├── silver/
│   ├── clean_orders/            -- partitioned by order_date
│   ├── clean_customers/         -- no partition (small dim)
│   ├── clean_products/          -- no partition (small dim)
│   └── clean_inventory/         -- partitioned by snapshot_date
└── gold/
    ├── sales/
    │   ├── daily_sales_summary/ -- partitioned by sale_date
    │   └── monthly_revenue/     -- partitioned by month
    ├── customer/
    │   ├── customer_360/        -- one big table
    │   └── customer_ltv/
    └── inventory/
        └── stock_alerts/
```

### Why This Works for Batch AND Streaming

The Medallion pattern is pipeline-agnostic. In batch mode, Bronze loads via scheduled ingestion jobs. In streaming mode, Bronze appends from Kafka via Spark Structured Streaming or Flink. Silver and Gold transformations can run as either micro-batch or continuous streaming jobs — the contract between layers doesn't change. Delta Lake's ACID transactions guarantee that downstream consumers never see partial writes regardless of whether the pipeline is batch or streaming.

> **Analogy:** Medallion is like a water treatment plant. Bronze is the raw intake — you pump in whatever comes from the river (mud, fish, debris). Silver is the filtration and treatment stage — remove impurities, standardize pH, test for contaminants. Gold is the tap water delivered to homes — clean, safe, and optimized for the specific use case (drinking, irrigation, industrial). You never throw away the raw intake logs because if someone gets sick, you need to trace back to the source.

### 💡 Interview Insight

> When asked *"Explain the Medallion Architecture"*, respond: **"Medallion organizes lakehouse data into three layers of increasing quality. Bronze is the raw, append-only ingestion layer — schema-on-read, full fidelity, our audit trail and time machine. Silver is the cleaned, conformed layer — schema-enforced, deduplicated, entity-level grain. This is where data quality rules live. Gold is the business-optimized layer — star schemas, OBTs, or pre-aggregations designed for specific consumers. The power of this pattern is that each layer has a clear contract: Bronze guarantees completeness, Silver guarantees correctness, and Gold guarantees performance. It works for both batch and streaming because the layer contracts are independent of the execution engine."**

---

## Screen 2: Schema-on-Read vs Schema-on-Write

### The Oldest Debate in Data Engineering

The question of *when* to enforce schema isn't just academic — it determines where your team spends its debugging time, how fast you can onboard new data sources, and whether your downstream consumers trust the data they're querying. Understanding this tradeoff is essential for any lakehouse modeling interview.

### Schema-on-Write: The Traditional Warehouse Approach

In a **schema-on-write** system, data must conform to a predefined schema *before* it's written. If a record doesn't match the expected types, constraints, or structure, it's rejected at load time.

```
┌──────────────────────────────────────────────────────────┐
│                 SCHEMA-ON-WRITE                           │
│                                                          │
│   Source Data ──▶ [ VALIDATE SCHEMA ] ──▶ Write to Table │
│                         │                                │
│                    Reject if                              │
│                    invalid ──▶ ❌ Error log               │
│                                                          │
│   Example: INSERT INTO orders (order_id INT, ...)        │
│            fails if order_id = 'ABC'                     │
└──────────────────────────────────────────────────────────┘
```

**Advantages:**
- Errors caught immediately — no garbage in the warehouse
- Consumers trust the data because the schema is a contract
- Query performance is predictable — engine knows exact types and layout
- Simpler downstream logic — no defensive casting or null-checking everywhere

**Disadvantages:**
- Rigid — adding a column requires ALTER TABLE and ETL changes
- Slow onboarding — every new source needs schema mapping before any data loads
- Fragile — upstream schema changes break the pipeline immediately
- Data loss risk — rejected records may be dropped silently if error handling is poor

### Schema-on-Read: The Data Lake Approach

In a **schema-on-read** system, data is stored as-is (raw files, JSON, Avro). Schema is applied only when querying — the reader decides how to interpret the data.

```
┌──────────────────────────────────────────────────────────┐
│                 SCHEMA-ON-READ                            │
│                                                          │
│   Source Data ──▶ Write raw to storage (no validation)   │
│                                                          │
│   Query time: SELECT CAST(col1 AS INT) FROM raw_data     │
│               ──▶ ❌ Runtime error if col1 = 'ABC'       │
│                                                          │
│   The READER defines the schema, not the writer          │
└──────────────────────────────────────────────────────────┘
```

**Advantages:**
- Fast ingestion — no upfront schema design needed
- Flexible — store anything, decide meaning later
- No data loss — every record is preserved, even malformed ones
- Handles semi-structured data naturally (JSON, nested arrays)

**Disadvantages:**
- Errors shift downstream — consumers discover bad data at query time
- Every reader reimplements parsing logic (duplicated, inconsistent)
- Performance suffers — engine can't optimize without known types
- Data swamp risk — without discipline, the lake becomes unusable

### The Lakehouse Hybrid: Best of Both Worlds

The lakehouse pattern resolves this tension by applying **schema-on-read at Bronze** and **schema-on-write at Silver**. This is not a compromise — it's a deliberate architecture choice.

```
┌──────────────────────────────────────────────────────────────────┐
│                 LAKEHOUSE HYBRID APPROACH                         │
│                                                                  │
│   BRONZE (Schema-on-Read)          SILVER (Schema-on-Write)      │
│   ─────────────────────            ────────────────────────      │
│   • Store raw as-is               • Enforce types & constraints  │
│   • Preserve full fidelity        • Reject/quarantine bad rows   │
│   • Add metadata columns          • Deduplicate & conform        │
│   • Any format accepted           • Delta/Iceberg enforced       │
│                                                                  │
│   "Accept everything"      ──▶    "Trust but verify"             │
│                                                                  │
│   Raw JSON blob:                  Typed, clean table:            │
│   {"price": "29.99",              price     DECIMAL(10,2)        │
│    "qty": "3",                    quantity  INT                   │
│    "ts": "2024-01-15"}            order_ts  TIMESTAMP            │
└──────────────────────────────────────────────────────────────────┘
```

### Practical Example: Handling Schema Inconsistency

Suppose a source API changes `customer_id` from integer to string in v2. In a pure schema-on-write warehouse, this breaks the pipeline immediately. In the lakehouse hybrid:

```sql
-- Bronze: store raw, both versions coexist peacefully
-- v1 records: {"customer_id": 12345, "name": "Alice"}
-- v2 records: {"customer_id": "CUST-12345", "name": "Alice"}

-- Silver: transformation handles both versions
INSERT INTO silver.clean_customers
SELECT
    CASE
        WHEN typeof(raw:customer_id) = 'INTEGER'
        THEN CONCAT('CUST-', CAST(raw:customer_id AS STRING))
        ELSE raw:customer_id::STRING
    END AS customer_id,
    raw:name::STRING AS customer_name,
    _ingested_at AS first_seen_at
FROM bronze.raw_customers
WHERE raw:customer_id IS NOT NULL;
```

### When to Enforce Strictly vs When to Be Lenient

| Scenario | Strategy | Why |
|----------|----------|-----|
| Financial/regulatory data | Strict schema-on-write | Compliance requires data integrity guarantees |
| Exploratory/new data sources | Schema-on-read at Bronze | Don't block ingestion while exploring |
| ML feature stores | Strict at Silver, flexible at Bronze | Features need type consistency for training |
| Multi-tenant SaaS data | Schema-on-read per tenant, enforce at Silver | Tenants have different schemas |
| Event/clickstream data | Schema-on-read at Bronze | High volume, evolving schema, can't afford rejections |
| Master data (customer, product) | Strict schema-on-write at Silver | Core entities need single source of truth |

> **Analogy:** Schema-on-write is like airport security — check everything before boarding, safer but slower and sometimes you reject passengers who have valid tickets in the wrong format. Schema-on-read is like a train station — anyone can board, but you might find someone without a ticket halfway through the journey. The lakehouse hybrid is TSA PreCheck — let everyone into the terminal (Bronze), but validate thoroughly before they board the flight (Silver).

### 💡 Interview Insight

> When asked *"How does the lakehouse handle schema enforcement?"*, say: **"The lakehouse uses a hybrid approach. Bronze is schema-on-read — we ingest raw data without validation to maximize flexibility and preserve full source fidelity. Silver applies schema-on-write — we enforce data types, NOT NULL constraints, and business rules. Records that fail validation are quarantined to a dead-letter table, not silently dropped. This gives us the flexibility of a data lake for ingestion with the reliability of a data warehouse for consumption. In practice, Silver tables use Delta Lake or Iceberg with schema enforcement enabled, so any write that violates the schema is rejected at the storage layer."**

---

## Screen 3: When to Model at Silver vs Gold

### The Critical Decision That Defines Your Architecture

This is the question that separates junior data engineers from senior ones. Everyone understands Bronze (dump raw data) and Gold (build dashboards). The real craft is in deciding *how much modeling* happens at Silver versus Gold. Get this wrong and you either over-engineer Silver into an unusable maze of premature star schemas, or you under-engineer it and push all the complexity into dozens of redundant Gold transformations.

### Silver: Entity-Level, Cleaned, Reusable

Silver tables should represent **cleaned, conformed business entities at their natural grain**. They answer the question: *"What is the canonical, trusted version of this entity?"*

```
┌───────────────────────────────────────────────────────────┐
│                    SILVER LAYER GRAIN                      │
│                                                           │
│   clean_customers  →  One row per customer (latest)       │
│   clean_orders     →  One row per order                   │
│   clean_products   →  One row per product (latest)        │
│   clean_order_items → One row per order line item         │
│   clean_events     →  One row per event occurrence        │
│                                                           │
│   KEY PRINCIPLE: Silver tables are ENTITY-CENTRIC,        │
│                  not BUSINESS-PROCESS-CENTRIC             │
└───────────────────────────────────────────────────────────┘
```

Silver is where you:
- Deduplicate records (pick latest by `updated_at` or apply CDC merge logic)
- Cast types and enforce constraints
- Resolve business keys across source systems
- Apply SCD2 logic if historical tracking is needed
- Conform codes and enumerations ("M"/"Male"/"1" → "M")

Silver is **not** where you:
- Build star schemas (that's premature coupling to a business process)
- Pre-aggregate anything (that constrains downstream flexibility)
- Create wide, denormalized tables (that's Gold's job)
- Apply business-specific filters (Gold decides what subset matters)

### Gold: Business-Specific, Optimized, Purpose-Built

Gold tables are designed for **specific business use cases**. They answer: *"What does [specific team/dashboard/model] need?"*

```
┌────────────────────────────────────────────────────────────┐
│                    GOLD LAYER EXAMPLES                      │
│                                                            │
│   daily_sales_summary                                      │
│   ├── Grain: one row per store per day                     │
│   ├── Source: silver.clean_orders + silver.clean_stores     │
│   └── Consumer: Revenue dashboard                          │
│                                                            │
│   customer_lifetime_value                                  │
│   ├── Grain: one row per customer                          │
│   ├── Source: silver.clean_orders + silver.clean_customers  │
│   └── Consumer: Marketing team                             │
│                                                            │
│   inventory_reorder_alerts                                 │
│   ├── Grain: one row per product per warehouse             │
│   ├── Source: silver.clean_inventory + silver.clean_products│
│   └── Consumer: Supply chain ops                           │
│                                                            │
│   product_performance_obt                                  │
│   ├── Grain: one row per product per day                   │
│   ├── Source: all Silver tables joined                      │
│   └── Consumer: Exec dashboard + ad-hoc analysis           │
└────────────────────────────────────────────────────────────┘
```

### The Decision Framework

**Model at Silver when:**
- Multiple Gold tables need the same cleaned entity
- You need a single source of truth for a business key
- Deduplication/conforming logic should be applied once, not repeated
- SCD history tracking is required at the entity level

**Model at Gold when:**
- The aggregation is specific to one business process or team
- You need to denormalize for query performance (OBTs)
- Pre-computation saves significant runtime for dashboards
- The model represents a specific KPI definition (CLV, NPS, churn risk)

### Anti-Patterns to Avoid

**Anti-Pattern 1: Over-Modeling Silver (Premature Star Schema)**

```
❌ WRONG: Building star schemas in Silver
   silver.fact_sales         -- Too specific for Silver
   silver.dim_customer       -- "dim_" prefix implies Gold-level design
   silver.dim_product        -- Premature coupling to a business process
   silver.bridge_promotions  -- Way too specific

✅ RIGHT: Entity-centric Silver
   silver.clean_sales_transactions  -- One row per transaction
   silver.clean_customers           -- One row per customer
   silver.clean_products            -- One row per product
   silver.clean_promotions          -- One row per promotion
```

When you build star schemas at Silver, you lock in dimensional modeling assumptions too early. If a new Gold use case needs a different grain or different dimensions, you either hack around your Silver star schema or build a parallel Silver pipeline — both are maintenance nightmares.

**Anti-Pattern 2: Under-Modeling Silver (Raw Dump Passed Through)**

```
❌ WRONG: Silver is just Bronze with a different folder name
   silver.orders  -- Still has duplicates, mixed types, raw CDC ops

✅ RIGHT: Silver applies meaningful transformations
   silver.clean_orders  -- Deduplicated, typed, grain enforced
```

If your Silver layer is just a copy of Bronze with some column renames, you haven't added value. Every Gold pipeline will duplicate the same cleaning logic independently — a guaranteed source of inconsistency.

### Practical Example: Multi-Team Silver Reuse

```sql
-- SILVER: cleaned once, reused everywhere
CREATE TABLE silver.clean_orders AS
SELECT
    order_id,
    customer_id,
    order_date,
    CAST(total_amount AS DECIMAL(12,2)) AS total_amount,
    order_status,
    updated_at
FROM bronze.raw_orders
QUALIFY ROW_NUMBER() OVER (
    PARTITION BY order_id ORDER BY updated_at DESC
) = 1;

-- GOLD TABLE 1: Finance team needs daily revenue
CREATE TABLE gold.daily_revenue AS
SELECT
    order_date,
    SUM(total_amount) AS total_revenue,
    COUNT(DISTINCT order_id) AS order_count
FROM silver.clean_orders
WHERE order_status = 'completed'
GROUP BY order_date;

-- GOLD TABLE 2: Marketing team needs customer LTV
CREATE TABLE gold.customer_ltv AS
SELECT
    c.customer_id,
    c.customer_name,
    c.signup_date,
    SUM(o.total_amount) AS lifetime_value,
    COUNT(o.order_id) AS total_orders,
    DATEDIFF(day, c.signup_date, MAX(o.order_date)) AS tenure_days
FROM silver.clean_customers c
LEFT JOIN silver.clean_orders o ON c.customer_id = o.customer_id
WHERE o.order_status = 'completed'
GROUP BY c.customer_id, c.customer_name, c.signup_date;
```

Both Gold tables share the same `silver.clean_orders` — the deduplication and type casting happen exactly once.

### 💡 Interview Insight

> When asked *"Where does dimensional modeling happen in a lakehouse?"*, say: **"Dimensional modeling lives in the Gold layer. Silver tables are entity-centric — one row per customer, one row per order — cleaned and conformed but not shaped for any specific business process. Gold is where we build star schemas, OBTs, or aggregated summaries for specific consumers. The key principle is: model at Silver what's universally true about an entity, and model at Gold what's specifically useful for a business question. This avoids premature coupling and ensures Silver tables can serve multiple Gold use cases without redundant transformations."**

---

## Screen 4: Partitioning Strategy

### Partitioning Is a Modeling Decision

Most engineers think of partitioning as infrastructure — a physical storage concern. But in a lakehouse, your partition strategy is inseparable from your data model. The wrong partition key turns a sub-second query into a full-scan nightmare. The right one can eliminate 99% of data scanned. Choosing partition keys requires understanding how your data is queried, how it arrives, and how much of it exists.

### How Partitioning Works in a Lakehouse

Unlike traditional databases where partitioning is index-adjacent, lakehouse partitioning is **file-system-level**. Each partition value creates a directory, and each directory contains one or more data files (Parquet, ORC). Query engines read only the directories that match the query predicate — this is called **partition pruning**.

```
delta_table/
├── order_date=2024-01-01/
│   ├── part-00000.parquet    (50 MB)
│   └── part-00001.parquet    (48 MB)
├── order_date=2024-01-02/
│   ├── part-00000.parquet    (52 MB)
│   └── part-00001.parquet    (47 MB)
└── order_date=2024-01-03/
    └── part-00000.parquet    (51 MB)

-- Query: WHERE order_date = '2024-01-02'
-- Engine reads ONLY the 2024-01-02 directory (~99 MB)
-- Skips all other directories entirely
```

### The Three Common Partition Strategies

**1. Partition by Time (Most Common)**

Time-based partitioning aligns with how most analytical queries filter data — by date range. It also aligns naturally with how data arrives (daily batches, hourly streams).

```sql
-- Delta Lake: partition by date
CREATE TABLE silver.clean_orders
USING DELTA
PARTITIONED BY (order_date)
AS SELECT * FROM transformed_orders;

-- Iceberg: hidden partitioning by month (no physical directory needed)
CREATE TABLE silver.clean_orders (
    order_id      BIGINT,
    customer_id   BIGINT,
    order_date    DATE,
    total_amount  DECIMAL(12,2)
) USING ICEBERG
PARTITIONED BY (months(order_date));
```

**2. Partition by Business Entity (Multi-Tenant)**

When data belongs to distinct tenants, regions, or business units with strict isolation requirements, partition by the entity identifier.

```sql
-- Multi-tenant SaaS: partition by tenant
CREATE TABLE silver.clean_events
USING DELTA
PARTITIONED BY (tenant_id)
AS SELECT * FROM transformed_events;

-- Regional data: partition by region
PARTITIONED BY (region, event_date)
```

**3. Partition by Category (High-Cardinality Dimension)**

Sometimes queries consistently filter on a categorical field like `country`, `product_category`, or `event_type`.

### The Over-Partitioning Problem (Small Files)

```
❌ OVER-PARTITIONED: partition by (year, month, day, hour, user_id)
   Result: millions of directories, each with a tiny 1KB file
   Problem: query planning overhead exceeds query execution time
   
   delta_table/
   ├── year=2024/month=01/day=01/hour=00/user_id=1/
   │   └── part-00000.parquet    (800 bytes)  ← TINY!
   ├── year=2024/month=01/day=01/hour=00/user_id=2/
   │   └── part-00000.parquet    (1.2 KB)     ← TINY!
   └── ... (2 million more directories)

✅ RIGHT-SIZED: partition by (order_date)
   Result: ~365 directories per year, each 50-200 MB per file
   
   delta_table/
   ├── order_date=2024-01-01/
   │   ├── part-00000.parquet    (128 MB)  ← GOOD
   │   └── part-00001.parquet    (115 MB)  ← GOOD
```

**Rule of thumb:** Each partition should contain at least **128 MB** of data (ideally 256 MB–1 GB). If a partition key produces millions of tiny files, it's the wrong key.

### The Under-Partitioning Problem (Full Scans)

```
❌ UN-PARTITIONED: no partition key at all
   Result: every query scans the entire table (500 GB)
   Even "WHERE order_date = '2024-01-15'" reads all 500 GB

✅ PARTITIONED BY DATE: only reads ~1.4 GB for a single day
   500 GB / 365 days ≈ 1.4 GB per partition
```

### Iceberg's Hidden Partitioning

Apache Iceberg introduced **hidden partitioning** — a game-changer. Instead of requiring users to know the partition scheme and include the partition column in queries, Iceberg applies partition transforms transparently.

```sql
-- Iceberg hidden partitioning: users query on order_ts directly
CREATE TABLE silver.clean_orders (
    order_id     BIGINT,
    order_ts     TIMESTAMP,
    amount       DECIMAL(12,2)
) USING ICEBERG
PARTITIONED BY (days(order_ts));

-- User writes a natural query — NO partition column gymnastics
SELECT * FROM silver.clean_orders
WHERE order_ts BETWEEN '2024-01-15' AND '2024-01-20';
-- Iceberg automatically prunes to only the relevant day-partitions

-- Partition evolution: change from daily to monthly WITHOUT rewriting data
ALTER TABLE silver.clean_orders
REPLACE PARTITION FIELD days(order_ts) WITH months(order_ts);
```

With Delta Lake, the partition column must appear literally in the WHERE clause for pruning to work. With Iceberg, the engine translates timestamp predicates into partition predicates automatically.

### Partition Strategy Decision Table

| Factor | Recommendation | Example |
|--------|---------------|---------|
| Queries filter by date range | Partition by date/month | `PARTITIONED BY (order_date)` |
| Multi-tenant isolation | Partition by tenant_id | `PARTITIONED BY (tenant_id)` |
| Table < 1 GB total | Don't partition at all | Small dimension tables |
| High-cardinality key (user_id) | Don't partition by it — use Z-ORDER | Z-ORDER BY (user_id) |
| Queries filter by 2 fields | Partition by low-cardinality, Z-ORDER the other | `PARTITIONED BY (region)` + Z-ORDER BY (date) |
| Streaming ingestion (many small files) | Partition by date + enable auto-compaction | Schedule OPTIMIZE regularly |
| Partition has < 128 MB per value | Coarsen the key (day → month) | `PARTITIONED BY (order_month)` |

> **Analogy:** Partitioning is like organizing a library. Partition by genre (fiction, non-fiction, science) and you can skip entire aisles. But if you partition by every author's last name AND first name AND publication year AND edition, you'll have millions of shelves each holding a single pamphlet — the librarian spends more time walking between shelves than reading. Z-ordering is like sorting books within a shelf by multiple criteria (year, then title) so you can find a specific range quickly without reorganizing the whole library.

### 💡 Interview Insight

> When asked *"How do you choose partition keys?"*, say: **"I start with query patterns — what columns appear most frequently in WHERE clauses? For time-series data, partition by date or month. For multi-tenant data, partition by tenant_id. The key constraint is partition size: each partition should hold 128 MB to 1 GB of data. Over-partitioning creates millions of small files that cripple query planning, while under-partitioning forces full table scans. For high-cardinality columns like user_id, I use Z-ORDER or Iceberg's sort orders instead of partitioning. I also prefer Iceberg's hidden partitioning when possible because it decouples the physical layout from the query interface — users don't need to know the partition scheme to get pruning."**

---

## Screen 5: Integration with Storage Formats

### Your Model Lives Inside a Format

A lakehouse data model doesn't exist in a vacuum — it lives inside Delta Lake, Apache Iceberg, or Apache Hudi. These table formats dictate what modeling operations are efficient, which schema changes are safe, and how your physical layout affects query performance. Understanding the interplay between modeling decisions and format capabilities is what makes a lakehouse engineer effective.

### Modeling for MERGE Operations: SCD2 in the Lakehouse

Traditional warehouses implement Slowly Changing Dimensions with UPDATE and INSERT statements. In the lakehouse, the **MERGE** (upsert) operation is the foundation. The choice of table format and primary key directly affects MERGE performance.

```sql
-- Delta Lake: SCD Type 2 using MERGE
-- Step 1: Expire current records that have changed
MERGE INTO silver.dim_customer AS target
USING (
    SELECT
        customer_id,
        customer_name,
        customer_email,
        city,
        current_timestamp() AS effective_from
    FROM bronze.raw_customers_batch
) AS source
ON target.customer_id = source.customer_id
   AND target.is_current = true

-- When a matching current record has changed attributes
WHEN MATCHED AND (
    target.customer_name != source.customer_name OR
    target.city != source.city
) THEN UPDATE SET
    target.is_current = false,
    target.effective_to = source.effective_from

-- When no current record exists for this customer
WHEN NOT MATCHED THEN INSERT (
    customer_id, customer_name, customer_email, city,
    effective_from, effective_to, is_current
) VALUES (
    source.customer_id, source.customer_name, source.customer_email,
    source.city, source.effective_from, NULL, true
);

-- Step 2: Insert new version for changed records
INSERT INTO silver.dim_customer
SELECT
    s.customer_id, s.customer_name, s.customer_email, s.city,
    s.effective_from, NULL AS effective_to, true AS is_current
FROM bronze.raw_customers_batch s
JOIN silver.dim_customer t
    ON s.customer_id = t.customer_id
    AND t.is_current = false
    AND t.effective_to = s.effective_from;
```

### Z-Ordering Alignment with Model Grain

**Z-ordering** (or data clustering) co-locates related data within files so that range queries on specific columns read fewer files. The key insight: **Z-ORDER columns should align with your model's most common join and filter keys**.

```sql
-- Delta Lake: Z-ORDER after writing
OPTIMIZE silver.clean_orders
ZORDER BY (customer_id, order_date);

-- Iceberg: sort order defined at table creation
CREATE TABLE silver.clean_orders (
    order_id      BIGINT,
    customer_id   BIGINT,
    order_date    DATE,
    total_amount  DECIMAL(12,2)
) USING ICEBERG
PARTITIONED BY (months(order_date))
TBLPROPERTIES (
    'write.distribution-mode' = 'hash',
    'write.sort-order' = 'customer_id ASC NULLS LAST'
);
```

**Z-ORDER strategy by model type:**

| Model Pattern | Z-ORDER Columns | Why |
|--------------|-----------------|-----|
| Fact table (orders) | customer_id, product_id | Common join keys for star schema queries |
| Event table (clickstream) | user_id, session_id | Filter/group by user then session |
| SCD2 dimension | business_key | MERGE needs to find all versions of a key quickly |
| Aggregated Gold table | No Z-ORDER needed | Already small and pre-filtered |
| OBT (One Big Table) | Top 2-3 filter columns | Most impactful for wide table scans |

### Statistics Collection for Query Optimization

Modern table formats maintain **column-level statistics** (min, max, null count, distinct count) in metadata. These statistics enable **file pruning** — the engine skips files whose min/max ranges don't overlap with the query predicate.

```python
# Delta Lake: collect statistics on key columns
spark.sql("""
    ALTER TABLE silver.clean_orders
    SET TBLPROPERTIES ('delta.dataSkippingNumIndexedCols' = '8')
""")

# Iceberg: configure column statistics
spark.sql("""
    ALTER TABLE silver.clean_orders
    WRITE ORDERED BY customer_id
""")

# Check what statistics are being collected
spark.sql("DESCRIBE DETAIL silver.clean_orders").show()
```

**Modeling implication:** Put your most-filtered columns *first* in the table definition. Many formats only collect statistics for the first N columns by default (Delta defaults to 32). If your primary filter column is column #50, the engine can't prune files for it.

### Compaction Strategy Based on Write Patterns

Streaming and frequent batch writes produce many small files. **Compaction** (OPTIMIZE in Delta, rewrite_data_files in Iceberg) merges these into optimally-sized files — but the strategy depends on your modeling pattern:

```sql
-- Delta Lake: compact and Z-ORDER in one operation
OPTIMIZE silver.clean_orders
WHERE order_date >= current_date() - INTERVAL 7 DAYS
ZORDER BY (customer_id);

-- Iceberg: rewrite small files
CALL catalog.system.rewrite_data_files(
    table => 'silver.clean_orders',
    strategy => 'sort',
    sort_order => 'customer_id ASC',
    options => map('target-file-size-bytes', '134217728')  -- 128 MB
);
```

**Compaction schedule by layer:**

| Layer | Write Pattern | Compaction Strategy |
|-------|--------------|-------------------|
| Bronze | High-frequency append | Compact daily, no Z-ORDER (preserve raw order) |
| Silver | MERGE/upsert batches | Compact after each MERGE, Z-ORDER by join keys |
| Gold | Rebuild or incremental | Usually self-optimized (CTAS or INSERT OVERWRITE) |

### Schema Evolution in Practice

One of the lakehouse's killer features is **schema evolution** — adding, renaming, or retyping columns without rewriting existing data. But this interacts with your model design:

```sql
-- Delta Lake: add a column to Silver (safe, non-breaking)
ALTER TABLE silver.clean_customers
ADD COLUMN loyalty_tier STRING AFTER customer_email;

-- Delta Lake: enable schema evolution during writes
SET spark.databricks.delta.schema.autoMerge.enabled = true;

-- Iceberg: rename a column (Iceberg tracks by column ID, not name)
ALTER TABLE silver.clean_customers
RENAME COLUMN customer_email TO email_address;
-- Existing Parquet files are NOT rewritten — Iceberg maps old name to new

-- Iceberg: evolve partition scheme without rewriting data
ALTER TABLE silver.clean_orders
ADD PARTITION FIELD bucket(16, customer_id);
```

**Key difference:** Delta Lake tracks columns by **name** (renaming is risky), while Iceberg tracks columns by **ID** (renaming is safe). This matters when your Silver model evolves over time — Iceberg handles it more gracefully.

> **Analogy:** Z-ordering is like organizing a bookstore where the most popular sections (mystery, romance) are shelved so that each shelf contains books from only a narrow range of authors. When a customer asks for "mystery novels by authors M-P," you point them to exactly two shelves instead of searching the whole store. Partition pruning skips entire sections of the store; Z-ordering skips individual shelves within a section.

### 💡 Interview Insight

> When asked *"How do storage formats affect your modeling decisions?"*, say: **"The table format dictates what operations are cheap. Delta and Iceberg make MERGE efficient, which is why SCD2 patterns work well in the lakehouse — something that was painful with plain Parquet files. I align Z-ORDER columns with my model's primary join and filter keys to minimize file reads. For schema evolution, Iceberg is safer because it tracks columns by ID rather than name, so renames don't break existing data. I also consider compaction strategy — streaming into Silver creates small files, so I schedule OPTIMIZE after MERGE jobs and target 128-256 MB per file. The model grain, the partition key, and the Z-ORDER columns should all tell a consistent story about how the data will be queried."**

---

## Screen 6: Real-World Problems & Solutions

### When Theory Meets Production

Every lakehouse eventually hits the same failure modes. You'll either experience these yourself or be asked to debug them in an interview. This section covers the four most common lakehouse modeling failures and their battle-tested solutions. These are the problems that distinguish engineers who've operated production systems from those who've only read the documentation.

### Problem 1: Silver Layer Becomes a Data Swamp

**Symptoms:**
- Silver tables have duplicate rows nobody noticed for months
- No clear grain — is `silver.orders` one row per order or one row per order update?
- Multiple Silver tables for the same entity built by different teams
- Downstream Gold pipelines each apply their own deduplication logic

**Root Cause:** Silver was treated as "cleaned Bronze" with no grain enforcement, no data quality checks, and no ownership.

**Solution: Enforce Grain, Add Constraints, Assign Ownership**

```sql
-- 1. Define and enforce grain with UNIQUE constraints (Databricks)
ALTER TABLE silver.clean_orders
ADD CONSTRAINT orders_pk PRIMARY KEY (order_id);

-- 2. Add CHECK constraints for data quality
ALTER TABLE silver.clean_orders
ADD CONSTRAINT valid_amount CHECK (total_amount >= 0);
ALTER TABLE silver.clean_orders
ADD CONSTRAINT valid_status CHECK (order_status IN
    ('pending', 'confirmed', 'shipped', 'delivered', 'cancelled'));

-- 3. Use Delta expectations (Databricks) or Iceberg row-level validation
-- Delta Live Tables example:
-- @dlt.expect_or_drop("valid_order_id", "order_id IS NOT NULL")
-- @dlt.expect_or_drop("positive_amount", "total_amount >= 0")
-- @dlt.expect_or_fail("unique_grain", "grain_check = 1")

-- 4. Post-load grain verification query (run after every pipeline)
SELECT
    order_id,
    COUNT(*) AS row_count
FROM silver.clean_orders
GROUP BY order_id
HAVING COUNT(*) > 1;
-- This MUST return zero rows. Alert if it doesn't.
```

```python
# Python: Great Expectations for Silver table validation
import great_expectations as gx

context = gx.get_context()
validator = context.sources.pandas_default.read_dataframe(silver_orders_df)

validator.expect_column_values_to_be_unique("order_id")
validator.expect_column_values_to_not_be_null("customer_id")
validator.expect_column_values_to_be_between("total_amount", min_value=0)
validator.expect_column_values_to_be_in_set(
    "order_status",
    ["pending", "confirmed", "shipped", "delivered", "cancelled"]
)

results = validator.validate()
if not results.success:
    raise DataQualityError(f"Silver validation failed: {results}")
```

### Problem 2: Gold Tables Are Too Slow

**Symptoms:**
- Dashboard queries take 30+ seconds on Gold tables
- Gold tables are just views on Silver with complex joins — no materialization
- Analysts write their own aggregations on Silver, bypassing Gold entirely
- BI tools time out during peak hours

**Root Cause:** Gold layer lacks pre-aggregation, partition alignment is wrong, or the Gold model doesn't match query patterns.

**Solution: Pre-Aggregate, Align Partitions, Materialize**

```sql
-- BEFORE: Gold is a VIEW (recomputed every query) — SLOW
CREATE VIEW gold.daily_sales AS
SELECT order_date, store_id, SUM(amount), COUNT(*)
FROM silver.clean_orders o
JOIN silver.clean_stores s ON o.store_id = s.store_id
GROUP BY order_date, store_id;
-- Every dashboard hit re-scans Silver (500 GB!)

-- AFTER: Gold is a MATERIALIZED TABLE, updated incrementally
CREATE TABLE gold.daily_sales_summary
USING DELTA
PARTITIONED BY (order_date)
AS
SELECT
    o.order_date,
    s.store_id,
    s.store_name,
    s.region,
    SUM(o.total_amount) AS total_revenue,
    COUNT(DISTINCT o.order_id) AS order_count,
    COUNT(DISTINCT o.customer_id) AS unique_customers,
    AVG(o.total_amount) AS avg_order_value
FROM silver.clean_orders o
JOIN silver.clean_stores s ON o.store_id = s.store_id
WHERE o.order_status = 'completed'
GROUP BY o.order_date, s.store_id, s.store_name, s.region;

-- Incremental update: only recompute recent days
MERGE INTO gold.daily_sales_summary AS target
USING (
    SELECT order_date, store_id, store_name, region,
           SUM(total_amount) AS total_revenue,
           COUNT(DISTINCT order_id) AS order_count,
           COUNT(DISTINCT customer_id) AS unique_customers,
           AVG(total_amount) AS avg_order_value
    FROM silver.clean_orders o
    JOIN silver.clean_stores s ON o.store_id = s.store_id
    WHERE o.order_status = 'completed'
      AND o.order_date >= current_date() - INTERVAL 3 DAYS
    GROUP BY order_date, store_id, store_name, region
) AS source
ON target.order_date = source.order_date
   AND target.store_id = source.store_id
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *;
```

### Problem 3: Schema Drift Breaks Pipelines

**Symptoms:**
- Bronze-to-Silver pipeline fails because source added a new column
- Silver table has columns that no longer exist in the source
- Column type changed upstream (INT → STRING) and MERGE fails with cast error
- Different source systems send the same field with different names

**Root Cause:** No schema enforcement at Bronze-to-Silver boundary, no evolution strategy, no alerting on schema changes.

**Solution: Schema Enforcement + Evolution Strategy**

```python
# Strategy: Detect schema changes BEFORE they break the pipeline

from pyspark.sql import SparkSession
from deepdiff import DeepDiff

spark = SparkSession.builder.getOrCreate()

def detect_schema_drift(new_df, table_name: str) -> dict:
    """Compare incoming schema against existing Silver table schema."""
    existing_schema = spark.table(table_name).schema
    new_schema = new_df.schema

    added_cols = set(new_schema.fieldNames()) - set(existing_schema.fieldNames())
    removed_cols = set(existing_schema.fieldNames()) - set(new_schema.fieldNames())

    type_changes = {}
    for field in existing_schema:
        if field.name in new_schema.fieldNames():
            new_field = new_schema[field.name]
            if field.dataType != new_field.dataType:
                type_changes[field.name] = {
                    "old": str(field.dataType),
                    "new": str(new_field.dataType)
                }

    drift = {
        "added_columns": added_cols,
        "removed_columns": removed_cols,
        "type_changes": type_changes,
        "has_breaking_changes": bool(removed_cols or type_changes)
    }

    if drift["has_breaking_changes"]:
        alert_team(f"BREAKING schema drift detected in {table_name}: {drift}")
    elif added_cols:
        log.info(f"Non-breaking drift in {table_name}: new columns {added_cols}")
        # Auto-evolve: add new columns to Silver
        for col in added_cols:
            col_type = new_schema[col].dataType.simpleString()
            spark.sql(f"ALTER TABLE {table_name} ADD COLUMN {col} {col_type}")

    return drift


# Usage in pipeline
incoming_df = spark.read.json("s3://bronze/raw_customers/2024-01-15/")
drift = detect_schema_drift(incoming_df, "silver.clean_customers")

if not drift["has_breaking_changes"]:
    # Safe to proceed with MERGE
    process_silver_merge(incoming_df, "silver.clean_customers")
else:
    # Route to quarantine for manual review
    incoming_df.write.mode("append").saveAsTable("quarantine.customers_drift")
```

### Problem 4: Late-Arriving Data in Medallion

**Symptoms:**
- An order from January 5th arrives on January 20th
- Gold `daily_sales_summary` for January 5th is already computed and served to dashboards
- Rebuilding the entire Gold table for a handful of late records is expensive
- Fact tables reference dimension values that didn't exist when the fact was originally loaded

**Root Cause:** The pipeline assumes data arrives in order. No upsert pattern at Gold. No watermark strategy.

**Solution: Upsert Patterns + Watermarks + Reprocessing Windows**

```sql
-- Strategy 1: MERGE at Gold with reprocessing window
-- Recompute last 7 days of Gold every run (catches late-arriving data)
MERGE INTO gold.daily_sales_summary AS target
USING (
    SELECT
        order_date,
        store_id,
        SUM(total_amount) AS total_revenue,
        COUNT(DISTINCT order_id) AS order_count
    FROM silver.clean_orders
    WHERE order_date >= current_date() - INTERVAL 7 DAYS
    GROUP BY order_date, store_id
) AS source
ON target.order_date = source.order_date
   AND target.store_id = source.store_id
WHEN MATCHED THEN UPDATE SET
    target.total_revenue = source.total_revenue,
    target.order_count = source.order_count
WHEN NOT MATCHED THEN INSERT *;

-- Strategy 2: Watermark column at Silver tracks when data was last updated
ALTER TABLE silver.clean_orders
ADD COLUMN _last_merged_at TIMESTAMP;

-- Pipeline reads Bronze records where _ingested_at > last watermark
-- This ensures late-arriving Bronze data flows through to Silver and Gold
```

```python
# Strategy 3: Partition-aware incremental reprocessing
def reprocess_affected_partitions(
    silver_table: str,
    gold_table: str,
    lookback_days: int = 7
):
    """Identify which Gold partitions need recomputation
    based on recently updated Silver records."""

    # Find partitions affected by recent Silver changes
    affected_dates = spark.sql(f"""
        SELECT DISTINCT order_date
        FROM {silver_table}
        WHERE _last_merged_at >= current_timestamp() - INTERVAL {lookback_days} DAYS
    """).collect()

    affected_date_list = [row.order_date for row in affected_dates]

    if affected_date_list:
        # Recompute ONLY affected Gold partitions
        date_filter = ",".join(f"'{d}'" for d in affected_date_list)
        spark.sql(f"""
            INSERT OVERWRITE TABLE {gold_table}
            PARTITION (order_date)
            SELECT order_date, store_id, SUM(total_amount), COUNT(*)
            FROM {silver_table}
            WHERE order_date IN ({date_filter})
            GROUP BY order_date, store_id
        """)
        log.info(f"Reprocessed {len(affected_date_list)} partitions in {gold_table}")
```

> **Analogy:** Late-arriving data is like a postcard from vacation that arrives three weeks after you're home. Your photo album (Gold layer) already has a "January 5th" page. You don't throw away the entire album and rebuild it — you open to that specific page and add the postcard. The reprocessing window is like checking your mailbox for late postcards every day for a week after you get home. Eventually you stop checking (the watermark moves forward), and any postcard arriving after that goes into a "late mail" folder for manual handling.

### 💡 Interview Insight

> When asked *"What's the hardest lakehouse modeling problem you've solved?"*, pick one of these four and walk through it: **"In production, our Silver layer had become a swamp — multiple teams had created overlapping Silver tables for the same entity with different dedup logic, so Gold tables downstream produced inconsistent metrics. I resolved it by establishing grain contracts: every Silver table has a documented grain, enforced by a primary key constraint and a post-load verification query. We consolidated to a single `clean_orders` table owned by one team, with schema validation in CI/CD. We added Great Expectations checks that run after every pipeline execution and alert on grain violations. Within two months, our Gold-layer metric discrepancies dropped to zero."**

---

## Screen 7: Quiz — Lakehouse Modeling Mastery

Test your understanding of the concepts covered in this module. These questions mirror the depth and style of real interview questions.

**Question 1:** In Medallion Architecture, which layer should enforce schema and data types?

- A) Bronze — catch errors as early as possible
- B) Silver — enforce after raw ingestion, before business consumption
- C) Gold — let business teams define schemas
- D) All layers should enforce the same schema

**Answer:** B) Silver. Bronze is schema-on-read to preserve raw fidelity and flexibility. Silver applies schema-on-write to enforce types, constraints, and grain. Gold inherits Silver's quality guarantees and adds business-specific shaping.

---

**Question 2:** A Silver table `clean_orders` has duplicates — the same `order_id` appears 3 times with slightly different `updated_at` timestamps. What is the most likely root cause and fix?

- A) The Bronze layer is corrupted — fix the ingestion pipeline
- B) The Silver MERGE/dedup logic is missing or using the wrong dedup key
- C) The partition key is wrong — repartition the table
- D) Z-ordering needs to be applied to fix the duplicates

**Answer:** B) The Silver transformation is either not applying deduplication or is using an incorrect deduplication strategy. The fix: apply `QUALIFY ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY updated_at DESC) = 1` during the Bronze-to-Silver transformation, and add a primary key constraint or post-load verification query to catch any future violations.

---

**Question 3:** You have a Silver table partitioned by `order_date` with 5 years of data (~200 GB). Most dashboard queries filter by the last 30 days, but occasionally analysts query full-year aggregations. The table also uses `customer_id` as a frequent join key. What is the optimal physical layout strategy?

- A) Partition by `order_date`, Z-ORDER by `customer_id`
- B) Partition by `customer_id`, Z-ORDER by `order_date`
- C) Partition by both `order_date` and `customer_id`
- D) No partitioning — rely entirely on Z-ORDER

**Answer:** A) Partition by `order_date` aligns with the primary filter pattern (last 30 days) and produces well-sized partitions (~110 MB/day). Z-ORDER by `customer_id` co-locates data for the common join key within each date partition. Option B would over-partition (millions of customers = millions of tiny directories). Option C compounds the over-partitioning problem. Option D loses the efficient date pruning.

---

**Question 4:** A source system changes a column from `INTEGER` to `STRING` for `customer_id`. Your Bronze-to-Silver pipeline uses Delta Lake with schema enforcement enabled. What happens and what should you do?

- A) Delta silently casts the STRING to INT — no action needed
- B) The write fails with a schema mismatch error — implement schema drift detection that catches type changes, quarantines the affected batch, and alerts the team for manual resolution
- C) Delta automatically evolves the schema — no action needed
- D) The Silver table becomes corrupted — rebuild from scratch

**Answer:** B) With schema enforcement enabled, Delta rejects writes that don't match the existing schema's types. The correct approach is a schema drift detection layer (as shown in Screen 6) that compares incoming vs existing schema before attempting the write. Type changes are breaking changes that require manual review — you may need to ALTER the Silver column type or add a CAST in the transformation logic.

---

**Question 5:** Your Gold table `daily_sales_summary` is a VIEW over Silver (not materialized). Dashboard queries take 45 seconds. What is the primary fix?

- A) Add more Z-ordering to Silver tables
- B) Materialize the Gold table as a Delta/Iceberg table with pre-computed aggregations, partitioned to match dashboard filter patterns
- C) Increase the cluster size
- D) Move the query logic into the BI tool

**Answer:** B) Materialize the Gold table. Views recompute on every query, re-scanning and re-joining Silver (potentially hundreds of GB). A materialized Gold table stores pre-computed results — dashboard queries scan megabytes instead of gigabytes. Partition the Gold table by the dashboard's primary time filter. Adding Z-ordering (A) helps but doesn't eliminate the fundamental problem of re-aggregating on every query.

---

**Question 6:** You need to implement SCD Type 2 in a lakehouse for `dim_customer`. The table has 50 million customers, and ~100,000 records change daily. Which approach is most efficient?

- A) Full table rebuild daily — DELETE all, INSERT from source
- B) MERGE on `customer_id` + `is_current`, expire changed rows and insert new versions
- C) Append-only with `effective_from`/`effective_to` — never update existing records
- D) Maintain a separate `customer_history` table and keep `dim_customer` as Type 1

**Answer:** B) MERGE is the standard lakehouse pattern for SCD2. It efficiently identifies the ~100K changed records without touching the other 49.9M. The MERGE matches on `customer_id WHERE is_current = true`, expires changed records (sets `is_current = false`, fills `effective_to`), and inserts new versions. Z-ORDER by `customer_id` ensures the MERGE reads minimal files. Option A is wasteful for 0.2% daily change rate. Option C works but makes "get current customer" queries slower (requires filtering on `is_current`).

---

**Question 7:** An Iceberg table is partitioned by `days(order_ts)`. A user queries `WHERE order_ts BETWEEN '2024-01-15 08:00' AND '2024-01-15 17:00'`. Does partition pruning occur?

- A) No — the query uses a timestamp range, not a date equality
- B) Yes — Iceberg's hidden partitioning automatically translates the timestamp predicate to the day partition
- C) Only if the user adds `AND order_date = '2024-01-15'` explicitly
- D) Only if the table uses Hive-style partitioning

**Answer:** B) This is the power of Iceberg's hidden partitioning. The engine knows the table is partitioned by `days(order_ts)`, so it translates the `BETWEEN` predicate on `order_ts` into a partition predicate: only scan the `2024-01-15` partition. No explicit partition column needed in the query. Delta Lake would require the partition column to appear literally in the WHERE clause for pruning.

---

## Key Takeaways

### Medallion Architecture
- **Bronze** = raw, append-only, schema-on-read — your audit trail and time machine
- **Silver** = cleaned, conformed, deduplicated, schema-enforced — your single source of truth per entity
- **Gold** = business-optimized star schemas, OBTs, aggregations — purpose-built for consumers
- Each layer has a **clear contract**: completeness → correctness → performance

### Schema Strategy
- **Schema-on-read at Bronze**, **schema-on-write at Silver** — this is the lakehouse hybrid, not a compromise
- Detect schema drift *before* it breaks pipelines — compare incoming vs existing schema programmatically
- Non-breaking changes (added columns) can auto-evolve; breaking changes (type changes, removed columns) need manual review

### Silver vs Gold Modeling
- Silver = **entity-centric** (one row per customer, one row per order) — cleaned, not shaped
- Gold = **business-process-centric** (daily_sales, customer_ltv) — shaped for specific consumers
- Anti-pattern: star schemas at Silver (premature coupling) or raw dumps passed through as Silver (no value added)
- Model at Silver what's universally true; model at Gold what's specifically useful

### Partitioning
- Partition by **low-cardinality columns** that appear in WHERE clauses (date, region, tenant)
- Target **128 MB – 1 GB per partition** — too small = small file problem, too large = full scans
- Use **Z-ORDER** for high-cardinality filter/join columns (customer_id, product_id)
- Iceberg's **hidden partitioning** decouples physical layout from query interface — prefer it when possible
- Don't partition tables under 1 GB

### Storage Format Integration
- **MERGE** is the lakehouse SCD2 pattern — align Z-ORDER with merge keys
- **Column statistics** drive file pruning — put filtered columns first in table definition
- **Compaction** after streaming/frequent writes — target 128-256 MB files
- **Iceberg** handles schema evolution more safely (column ID tracking vs Delta's name tracking)
- Your partition key, Z-ORDER columns, and model grain must tell a **consistent story**

### Production Survival Guide
- **Grain contracts** prevent Silver from becoming a swamp — enforce with constraints and post-load checks
- **Materialize Gold** tables — views that re-scan Silver on every query are a performance trap
- **Reprocessing windows** handle late-arriving data — MERGE last N days at Gold on every run
- **Schema drift detection** saves you from 3 AM pages — detect and alert before writing
- **Own your Silver tables** — unclear ownership leads to duplicated, inconsistent entity definitions

> *"The lakehouse doesn't eliminate modeling — it changes where and when you model. Bronze defers decisions, Silver makes them, and Gold serves them. Get the boundaries right and everything downstream flows."*
