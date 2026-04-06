# Module 10: Delta Lake, Iceberg & the Lakehouse

---

## Screen 1: What Is a Lakehouse?

### The Two-World Problem

For over a decade, data teams lived in a painful split reality:

**Data Lakes** (Hadoop/S3): Cheap object storage, any format, any schema — but zero governance. No ACID transactions, no schema enforcement, no consistency guarantees. You dump CSV, JSON, Parquet into folders and *hope* downstream consumers can make sense of it. Concurrent writes corrupt data silently. There's no way to UPDATE or DELETE a single record without rewriting entire partitions. The result? "Data swamps."

**Data Warehouses** (Snowflake, Redshift, BigQuery): Structured, governed, ACID-compliant — but expensive per-terabyte, proprietary formats, and vendor lock-in. You pay for both compute *and* storage, and getting raw/semi-structured data in requires painful ETL. ML workloads don't run natively here.

The industry's dirty secret was that most enterprises ran **both** — a lake for raw/ML data and a warehouse for BI — with fragile ETL pipelines copying data between them. Double storage cost, double governance headaches, stale data.

### The Lakehouse: Best of Both Worlds

A **Lakehouse** is an architecture that puts **warehouse-like capabilities directly on top of data lake storage**. You keep the cheap, open, scalable storage (S3, ADLS, GCS) but add:

- **ACID transactions** — reads and writes are atomic, consistent, isolated, durable
- **Schema enforcement & evolution** — reject bad data at write time, evolve schemas safely
- **Time travel & audit history** — query any historical version of your data
- **Indexing & optimization** — file compaction, data skipping, Z-ordering for fast queries
- **Unified batch + streaming** — same table serves both paradigms
- **Open file formats** — no vendor lock-in; Parquet files readable by any engine

### The Layered Architecture

Think of it as a three-layer cake:

```
┌─────────────────────────────────────────────┐
│          QUERY ENGINE LAYER                  │
│    Spark · Trino · Presto · Flink · Dremio  │
├─────────────────────────────────────────────┤
│          TABLE FORMAT LAYER                  │
│    Delta Lake · Apache Iceberg · Apache Hudi │
│    (metadata, transactions, schema, stats)   │
├─────────────────────────────────────────────┤
│          STORAGE LAYER                       │
│    S3 · ADLS · GCS · HDFS · MinIO           │
│    (cheap, durable, scalable object storage) │
└─────────────────────────────────────────────┘
```

The **table format** is the magic layer. It's not a query engine and it's not a storage system — it's a **metadata specification** that sits between the two. It tracks which files belong to which table, manages transactions, and provides statistics for query optimization.

> **Analogy:** Think of the table format like a librarian's catalog system. The library (storage) holds the actual books (Parquet files). The catalog (Delta/Iceberg) knows which books exist, what's in them, and enforces rules about checking books in and out. Multiple people (query engines) can use the same catalog to find what they need.

### 💡 Interview Insight

> When asked *"What is a Lakehouse?"*, say: **"A lakehouse gives you warehouse-like reliability — ACID transactions, schema enforcement, time travel — on top of data lake economics. The key enabler is the table format layer (Delta, Iceberg, or Hudi) which adds transactional metadata on top of open file formats like Parquet stored on cheap object storage like S3."**
>
> This answer demonstrates you understand the *architecture* — not just the buzzword.

---

## Screen 2: Delta Lake Deep Dive

### What Is Delta Lake?

Delta Lake is an **open-source table format** created by Databricks. At its core, a Delta table is just **Parquet files + a transaction log**. That transaction log is what transforms a messy pile of Parquet files into a proper ACID-compliant table.

### The Transaction Log: `_delta_log/`

Every Delta table has a `_delta_log/` directory. This is the single source of truth for the table's state.

```
my_table/
├── _delta_log/
│   ├── 00000000000000000000.json    ← commit 0 (table creation)
│   ├── 00000000000000000001.json    ← commit 1 (first insert)
│   ├── 00000000000000000002.json    ← commit 2 (update)
│   ├── ...
│   └── 00000000000000000010.checkpoint.parquet  ← checkpoint every 10 commits
├── part-00000-abc123.parquet
├── part-00001-def456.parquet
└── part-00002-ghi789.parquet
```

**How it works:**

1. Each **JSON commit file** records the *actions* performed: which Parquet files were **added** (`add`) and which were **removed** (`remove`).
2. To read the current table state, Delta replays all commit files in order to determine which Parquet files are currently "active."
3. Every **10 commits** (configurable), Delta writes a **checkpoint file** (in Parquet format) that consolidates the table state. This avoids replaying hundreds of JSON files — the reader starts from the latest checkpoint and only replays commits after it.

> **Analogy:** The transaction log is like a bank ledger. Each entry records deposits and withdrawals. To know your current balance, you replay the ledger from the start — or from the last bank statement (checkpoint).

### ACID Guarantees

- **Atomicity:** Each commit is all-or-nothing. If a write fails halfway, the commit JSON is never written, so readers never see partial data.
- **Consistency:** Schema enforcement ensures only valid data is written.
- **Isolation:** Delta uses **optimistic concurrency control (OCC)**. Multiple writers can write simultaneously. At commit time, Delta checks whether the files you read were modified by another writer. If there's a conflict, the losing writer retries. Reads are always isolated — they see a consistent snapshot.
- **Durability:** Data files are on durable object storage (S3/ADLS). The commit log is the record of truth.

### Key DML Operations with PySpark

**Create a Delta Table:**

```python
# Write a DataFrame as a Delta table
df.write.format("delta").mode("overwrite").save("/data/events")

# Or using saveAsTable for catalog-managed tables
df.write.format("delta").saveAsTable("events")
```

**INSERT (append new data):**

```python
new_data.write.format("delta").mode("append").save("/data/events")
```

**UPDATE (modify existing rows):**

```python
from delta.tables import DeltaTable

delta_table = DeltaTable.forPath(spark, "/data/events")

# Update all rows where the condition matches
delta_table.update(
    condition="event_type = 'click'",        # WHERE clause
    set={"processed": "true"}                 # SET clause
)
```

**DELETE (remove rows):**

```python
delta_table.delete(condition="event_date < '2023-01-01'")
```

**MERGE (upsert) — The Most Interview-Asked Operation:**

`MERGE` is the Swiss Army knife of Delta operations. It combines INSERT, UPDATE, and DELETE in a single atomic operation — essential for CDC (Change Data Capture) and SCD (Slowly Changing Dimensions).

```python
from delta.tables import DeltaTable

# Load the existing Delta table
target = DeltaTable.forPath(spark, "/data/customers")

# New/updated records from your CDC feed
source = spark.read.parquet("/data/customer_updates")

# The MERGE operation
target.alias("t").merge(
    source.alias("s"),
    "t.customer_id = s.customer_id"          # Join condition
).whenMatchedUpdate(
    # If the customer exists AND something changed, update them
    condition="s.updated_at > t.updated_at",
    set={
        "name": "s.name",
        "email": "s.email",
        "updated_at": "s.updated_at"
    }
).whenNotMatchedInsert(
    # If the customer doesn't exist, insert them
    values={
        "customer_id": "s.customer_id",
        "name": "s.name",
        "email": "s.email",
        "updated_at": "s.updated_at"
    }
).execute()
```

**What happens under the hood during MERGE:**
1. Spark reads the `source` and joins it against the `target` files
2. For each match/non-match, it applies the specified action
3. Affected Parquet files are **rewritten entirely** (Delta doesn't do in-place edits)
4. The commit log records which old files were `removed` and which new files were `added`
5. The whole operation is **atomic** — readers see either the old state or the new state, never a mix

### 💡 Interview Insight

> **"How does Delta Lake handle concurrent writes?"**
>
> Say: **"Delta Lake uses optimistic concurrency control. Multiple writers proceed independently without locking. At commit time, Delta checks the transaction log to see if any conflicting changes occurred since the writer began. If there's a conflict — like two writers modifying the same partition — one writer succeeds and the other gets a `ConcurrentModificationException` and can retry. This is more performant than pessimistic locking because conflicts are rare in most workloads."**

---

## Screen 3: Time Travel & Schema Evolution

### Time Travel: Your Data's Undo Button

Because Delta's transaction log records every change as a versioned commit, you can **read any historical version** of your table. This is called **time travel**.

**By version number:**

```python
# Read the table as it was at version 5
df = (spark.read
    .format("delta")
    .option("versionAsOf", 5)
    .load("/data/events")
)
```

**By timestamp:**

```python
# Read the table as it was at a specific point in time
df = (spark.read
    .format("delta")
    .option("timestampAsOf", "2024-01-15 09:30:00")
    .load("/data/events")
)
```

**Using SQL:**

```sql
-- By version
SELECT * FROM events VERSION AS OF 5;

-- By timestamp
SELECT * FROM events TIMESTAMP AS OF '2024-01-15';
```

**View table history:**

```python
delta_table = DeltaTable.forPath(spark, "/data/events")
delta_table.history().show()

# Shows: version, timestamp, operation, operationParameters, ...
```

### Real-World Use Cases for Time Travel

| Use Case | How Time Travel Helps |
|----------|----------------------|
| **Auditing** | "Show me what the customer table looked like on Dec 31 for year-end compliance" |
| **Debugging** | "Our dashboard broke yesterday — let me compare today's data vs yesterday's to find what changed" |
| **Rollback** | "A bad ETL job corrupted our table — restore it to the last known good version" |
| **Reproducibility** | "Re-run the ML model training on exactly the same data we used last month" |
| **Change comparison** | `df_before.exceptAll(df_after)` to see exactly what rows changed |

**Rollback with RESTORE:**

```sql
-- Restore to a specific version
RESTORE TABLE events TO VERSION AS OF 42;

-- Restore to a timestamp
RESTORE TABLE events TO TIMESTAMP AS OF '2024-01-15';
```

```python
# PySpark
delta_table.restoreToVersion(42)
delta_table.restoreToTimestamp("2024-01-15")
```

> ⚠️ **Important:** Time travel only works as long as the underlying Parquet files still exist. If you run `VACUUM` with a short retention period, old files are deleted and time travel to those versions becomes impossible.

### Schema Enforcement (Write-Time Validation)

Delta Lake **rejects** writes whose schema doesn't match the table's schema. This prevents data corruption from upstream changes.

```python
# This will FAIL if 'events' expects columns [id, name, value]
# but bad_df has columns [id, name, amount]
bad_df.write.format("delta").mode("append").save("/data/events")
# AnalysisException: A schema mismatch detected when writing to the Delta table
```

This is a *huge* advantage over raw Parquet, where any schema can be written to any directory, causing silent downstream failures.

### Schema Evolution (Controlled Change)

When you *intentionally* want to change the schema:

**Additive changes (new columns) — `mergeSchema`:**

```python
# DataFrame has a new column "region" not in the existing table
df_with_new_col.write \
    .format("delta") \
    .mode("append") \
    .option("mergeSchema", "true") \
    .save("/data/events")
# Existing rows get NULL for the new "region" column
```

`mergeSchema` handles safe, additive changes:
- ✅ Adding a new column
- ✅ Widening a type (e.g., IntegerType → LongType)
- ❌ Won't handle renaming, dropping, or type narrowing

**Breaking changes (full schema replace) — `overwriteSchema`:**

```python
# Completely replace the table with a new schema
df_new_schema.write \
    .format("delta") \
    .mode("overwrite") \
    .option("overwriteSchema", "true") \
    .save("/data/events")
```

Use `overwriteSchema` when you need to fundamentally restructure a table (e.g., dropping columns, renaming columns, changing types incompatibly).

### 💡 Interview Insight

> **"How does Delta Lake's time travel work?"**
>
> Say: **"Time travel is powered by the transaction log. Each commit creates a new version of the table by recording which Parquet files were added or removed. When you query version N, Delta reads the log up to commit N and determines which Parquet files composed the table at that point. No data is duplicated — unchanged files are shared across versions. The only constraint is that VACUUM can delete old files, so you need to keep retention periods aligned with your time travel needs."**

---

## Screen 4: Optimization Operations

### The Small Files Problem

Streaming jobs and frequent appends create thousands of tiny Parquet files (often <1MB each). This destroys read performance because:
- Each file requires a separate I/O operation (high overhead per file)
- Object store LIST operations are slow with many files
- Spark creates one task per file — thousands of tiny tasks have massive scheduling overhead
- Parquet metadata (column stats, schema) is duplicated in every file

> **Analogy:** Imagine searching 10,000 one-page documents vs. searching 10 books with 1,000 pages each. The content is the same, but the overhead of opening/closing each document kills you.

### OPTIMIZE: File Compaction

`OPTIMIZE` reads small files and rewrites them into larger, optimally-sized files (target ~1GB by default).

```sql
-- Compact all small files in the table
OPTIMIZE events;

-- Compact only a specific partition (much faster for large tables)
OPTIMIZE events WHERE event_date = '2024-01-15';
```

```python
from delta.tables import DeltaTable

delta_table = DeltaTable.forPath(spark, "/data/events")
delta_table.optimize().executeCompaction()

# With a partition filter
delta_table.optimize().where("event_date = '2024-01-15'").executeCompaction()
```

**What happens:** Small Parquet files are read, merged, and rewritten into fewer, larger files. The old small files remain on disk (referenced by older versions for time travel) until `VACUUM` removes them.

### Z-ORDER BY: Data Clustering

Z-ordering physically co-locates related data within the same files based on one or more columns. This dramatically improves query performance by enabling **data skipping** — Spark reads file-level min/max statistics and skips files that can't contain matching rows.

```sql
-- Compact files AND co-locate data by user_id and event_date
OPTIMIZE events ZORDER BY (user_id, event_date);
```

```python
delta_table.optimize().executeZOrderBy("user_id", "event_date")
```

**When to Z-ORDER:**
- Choose columns you **frequently filter on** in WHERE clauses
- High-cardinality columns benefit most (e.g., `user_id`, not `country`)
- Limit to 2-4 columns — effectiveness decreases with more columns (multidimensional trade-off)
- Z-ordering rewrites ALL data, so it's more expensive than plain `OPTIMIZE`

> **Analogy:** Think of a library. Plain `OPTIMIZE` is like consolidating small scraps of paper into proper books. `ZORDER BY` goes further — it *re-shelves* the books so all books about the same topic are on the same shelf. When someone asks for "all books about Python," you go to one shelf instead of checking every shelf in the building.

### VACUUM: Storage Cleanup

`VACUUM` deletes old Parquet files that are no longer referenced by any version within the retention period.

```sql
-- Delete files older than 168 hours (7 days, the default)
VACUUM events;

-- Custom retention period
VACUUM events RETAIN 720 HOURS;  -- 30 days
```

```python
delta_table.vacuum(retentionHours=168)
```

**Critical rules:**
- Default retention: **7 days (168 hours)**. Files referenced only by versions older than this are eligible for deletion.
- After VACUUM, **time travel to versions older than the retention period is impossible** — the Parquet files are gone.
- Never set retention below 7 days in production without understanding the consequences (you can override with `spark.databricks.delta.retentionDurationCheck.enabled = false`, but don't).
- Always run `OPTIMIZE` **before** `VACUUM` — otherwise you compact files and then vacuum can't clean the old small files until the retention period passes.

### Auto-Optimization

For tables that receive frequent small writes (streaming), configure automatic optimization:

```python
# Enable at table creation or ALTER TABLE
spark.sql("""
    ALTER TABLE events SET TBLPROPERTIES (
        'delta.autoOptimize.optimizeWrite' = 'true',   -- bin-packing on write
        'delta.autoOptimize.autoCompact' = 'true'       -- automatic compaction
    )
""")
```

| Feature | What It Does | When It Runs |
|---------|-------------|--------------|
| **Optimized Writes** | Coalesces small partitions before writing — reduces file count at write time | During write |
| **Auto Compaction** | Triggers a lightweight `OPTIMIZE` after writes if many small files are detected | After write |

These are **not** a replacement for periodic `OPTIMIZE + ZORDER` but they keep the small files problem from getting out of hand between maintenance runs.

### Maintenance Playbook

A production Delta table maintenance routine typically looks like:

```python
# Run nightly or weekly depending on write volume
from delta.tables import DeltaTable

table = DeltaTable.forPath(spark, "/data/events")

# Step 1: Compact small files and cluster by key query columns
table.optimize().executeZOrderBy("user_id", "event_date")

# Step 2: Clean up old files (after retention period)
table.vacuum(retentionHours=168)

# Step 3: Analyze table statistics (for query optimizer)
spark.sql("ANALYZE TABLE events COMPUTE STATISTICS FOR ALL COLUMNS")
```

### 💡 Interview Insight

> **"How do you handle the small files problem in Delta Lake?"**
>
> Say: **"I use a combination of strategies. First, `OPTIMIZE` compacts small files into optimally-sized ones, and I apply `ZORDER BY` on frequently-queried columns for data skipping. Second, `VACUUM` cleans up the old files after a retention period to save storage. For streaming tables, I enable `autoOptimize.optimizeWrite` and `autoCompact` to prevent the problem at write time. I typically run OPTIMIZE + VACUUM as a nightly or weekly maintenance job."**

---

## Screen 5: Apache Iceberg

### What Is Apache Iceberg?

Apache Iceberg is an **open table format** originally created at Netflix (2017) and donated to the Apache Software Foundation. Like Delta Lake, it brings ACID transactions and schema management to data lakes — but with a different architecture and some distinct advantages.

**Key distinction:** Iceberg is a **table format specification**, not a query engine. It defines how table metadata is organized; the actual query execution is handled by engines like Spark, Trino, Presto, Flink, Dremio, or Snowflake (which all natively support Iceberg).

### Iceberg's Metadata Architecture

Iceberg uses a layered metadata tree — this is fundamentally different from Delta's flat transaction log:

```
┌──────────────────────────┐
│     Catalog              │  ← Points to current metadata file
│  (Hive/Glue/REST/Nessie) │     (the "entry point")
└──────────┬───────────────┘
           ▼
┌──────────────────────────┐
│   Metadata File (.json)  │  ← Table schema, partition spec,
│                          │     current snapshot ID, properties
└──────────┬───────────────┘
           ▼
┌──────────────────────────┐
│   Manifest List          │  ← One per snapshot; lists all
│   (snap-xxx.avro)        │     manifest files in this snapshot
└──────────┬───────────────┘
           ▼
┌──────────────────────────┐
│   Manifest Files         │  ← Track sets of data files +
│   (manifest-xxx.avro)    │     per-file column statistics
└──────────┬───────────────┘     (min/max/null count/distinct)
           ▼
┌──────────────────────────┐
│   Data Files             │  ← Actual Parquet/ORC/Avro files
│   (data-xxx.parquet)     │     containing your records
└──────────────────────────┘
```

**Why this matters for performance:**
- Manifest files contain **column-level statistics** (min, max, null count) for each data file
- Query planning uses these statistics to **skip entire files** without opening them
- For a table with 100,000 files, Iceberg might determine from metadata alone that only 50 files are relevant — without listing any files on S3

### Hidden Partitioning — Iceberg's Killer Feature

In traditional partitioning (Hive-style, used by Delta and raw Parquet), the **user** must know the partition structure and write queries accordingly:

```sql
-- Hive-style: user MUST know data is partitioned by year/month/day
-- and use the partition columns explicitly
SELECT * FROM events
WHERE year = 2024 AND month = 1 AND day = 15;

-- If you write this instead, NO partition pruning happens:
SELECT * FROM events
WHERE event_timestamp = '2024-01-15 10:30:00';
-- Full table scan! 😱
```

With Iceberg's hidden partitioning, partition transforms are defined in **metadata**, not in the data layout visible to users:

```python
# Creating an Iceberg table with hidden partitioning
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .config("spark.sql.catalog.my_catalog", "org.apache.iceberg.spark.SparkCatalog") \
    .config("spark.sql.catalog.my_catalog.type", "hadoop") \
    .config("spark.sql.catalog.my_catalog.warehouse", "s3://my-bucket/warehouse") \
    .getOrCreate()

# Define the table with partition transforms
spark.sql("""
    CREATE TABLE my_catalog.db.events (
        event_id     BIGINT,
        user_id      BIGINT,
        event_type   STRING,
        event_ts     TIMESTAMP,
        payload      STRING
    )
    USING iceberg
    PARTITIONED BY (
        days(event_ts),          -- partition by day extracted from timestamp
        bucket(16, user_id)      -- hash-bucket user_id into 16 buckets
    )
""")
```

Now users query naturally — **no knowledge of partitioning required:**

```sql
-- Iceberg automatically applies partition pruning!
SELECT * FROM my_catalog.db.events
WHERE event_ts BETWEEN '2024-01-15' AND '2024-01-16'
  AND user_id = 42;
-- Engine uses metadata to skip irrelevant day-partitions AND user-buckets
```

**Available partition transforms:**

| Transform | Example | What It Does |
|-----------|---------|-------------|
| `year(ts)` | `2024` | Extract year from timestamp |
| `month(ts)` | `2024-01` | Extract year-month |
| `day(ts)` | `2024-01-15` | Extract date |
| `hour(ts)` | `2024-01-15-10` | Extract date-hour |
| `bucket(N, col)` | `bucket(16, user_id)` | Hash into N buckets |
| `truncate(L, col)` | `truncate(3, city)` → `"Chi"` | Truncate string/number |

### Partition Evolution

With Delta or Hive tables, changing the partitioning strategy requires **rewriting the entire table**. With Iceberg, you can evolve partitions in-place:

```sql
-- Start partitioned by day
ALTER TABLE my_catalog.db.events
    ADD PARTITION FIELD hour(event_ts);

-- Old data stays partitioned by day (no rewrite)
-- New data is partitioned by hour
-- Queries on both old and new data work seamlessly
```

This is extremely powerful for growing tables. You might start partitioning by month, then switch to day as volume increases, then to hour — all without rewriting terabytes of historical data.

### Schema Evolution

Iceberg supports full schema evolution without rewriting data:

```sql
-- Add a column
ALTER TABLE my_catalog.db.events ADD COLUMN region STRING;

-- Drop a column
ALTER TABLE my_catalog.db.events DROP COLUMN payload;

-- Rename a column (Delta can't do this without overwriteSchema)
ALTER TABLE my_catalog.db.events RENAME COLUMN event_type TO action;

-- Reorder columns
ALTER TABLE my_catalog.db.events ALTER COLUMN region AFTER user_id;
```

Iceberg tracks columns by **unique IDs** (not by name or position), so renames and reorders are metadata-only operations — zero data rewriting.

### Snapshot Isolation

Every read in Iceberg sees a **consistent snapshot**. Even if a writer is mid-write (adding new files), readers continue to see the last committed snapshot. This is similar to Delta but implemented through the manifest list layer — each snapshot points to an immutable manifest list that defines exactly which files are in scope.

### 💡 Interview Insight

> **"What makes Iceberg different from Delta Lake?"**
>
> Say: **"The biggest differentiator is hidden partitioning — Iceberg applies partition transforms in metadata so users write natural queries without knowing how data is partitioned, and the engine prunes automatically. Iceberg also supports partition evolution (change partitioning without rewriting data) and full schema evolution by tracking columns with unique IDs. Architecturally, Iceberg is engine-agnostic — it's used natively by Spark, Trino, Flink, Snowflake, and others — whereas Delta has historically been Databricks-centric, though that's changing with Delta UniForm."**

---

## Screen 6: Delta vs Iceberg vs Hudi — The Comparison

### The Big Three Table Formats

All three solve the same core problem (ACID on data lakes), but each originated from different pain points:

| Feature | Delta Lake | Apache Iceberg | Apache Hudi |
|---------|-----------|----------------|-------------|
| **Origin** | Databricks (2019) | Netflix → Apache (2017) | Uber → Apache (2016) |
| **Core design goal** | Unified batch+streaming on Spark | Large-scale analytics with engine flexibility | Incremental data processing (CDC/upserts) |
| **ACID transactions** | ✅ | ✅ | ✅ |
| **Time travel** | ✅ (robust) | ✅ (robust) | ✅ (limited, depends on timeline retention) |
| **Schema evolution** | ✅ (add columns, widen types) | ✅ (best: add, drop, rename, reorder — all metadata-only) | ✅ (add columns, limited renames) |
| **Hidden partitioning** | ❌ | ✅ (killer feature) | ❌ |
| **Partition evolution** | ❌ (requires full rewrite) | ✅ (no rewrite needed) | ❌ (requires full rewrite) |
| **File formats** | Parquet only | Parquet, ORC, Avro | Parquet only |
| **Ecosystem lock-in** | Databricks-centric (improving with UniForm) | Engine-agnostic (Spark, Trino, Flink, Snowflake) | Spark, Flink |
| **MERGE performance** | Excellent (copy-on-write) | Good (improving rapidly) | Excellent (Merge-on-Read option) |
| **Streaming support** | Excellent (native Structured Streaming) | Good (Flink integration strong) | Excellent (core design goal) |
| **Community adoption** | Very high | Growing fastest | Moderate |
| **Metadata approach** | Flat transaction log (JSON + checkpoints) | Layered tree (metadata → manifest list → manifests) | Timeline-based (instants on timeline) |

### Apache Hudi — The Upsert Specialist

Hudi (Hadoop Upserts Deletes and Incrementals) was built at Uber specifically for **high-frequency upserts** from streaming CDC pipelines. Its unique contribution is two storage types:

**Copy-on-Write (CoW):**
- On every write, affected Parquet files are **rewritten entirely**
- Reads are fast (pure Parquet columnar scans)
- Writes are slower (full file rewrite on every update)
- Best for: **read-heavy workloads** with infrequent updates

**Merge-on-Read (MoR):**
- Updates are written as small **delta log files** (Avro) alongside the base Parquet files
- Reads must merge base files + delta logs on the fly
- Writes are very fast (append-only log files)
- Periodically, a **compaction** job merges delta logs into base files
- Best for: **write-heavy workloads** with frequent updates (CDC, streaming)

```
CoW Table:                     MoR Table:
┌─────────────────┐           ┌─────────────────┐
│ base_file_1.pqt │←rewrite   │ base_file_1.pqt │
│ base_file_2.pqt │           │   └─ .log.1      │←append log
│ base_file_3.pqt │           │   └─ .log.2      │←append log
└─────────────────┘           │ base_file_2.pqt │
                              └─────────────────┘
                              Compaction merges logs into base
```

### When to Choose What

| Scenario | Best Choice | Why |
|----------|------------|-----|
| Databricks-native shop | **Delta Lake** | Tightest integration, best tooling, managed by Databricks |
| Multi-engine environment (Spark + Trino + Flink) | **Iceberg** | Engine-agnostic design, no vendor lock-in |
| Streaming CDC with millions of upserts/sec | **Hudi** | MoR storage type optimized for high write throughput |
| Need to change partitioning on large tables | **Iceberg** | Partition evolution without data rewrite |
| Snowflake + data lake integration | **Iceberg** | Snowflake has native Iceberg table support |
| Simple, well-supported, large community | **Delta Lake** | Largest community, most tutorials, Databricks support |

### The Convergence Trend

The three formats are **converging** rapidly:
- **Delta UniForm** (2023): Delta tables that emit Iceberg-compatible metadata, allowing Iceberg readers to query Delta tables directly
- **Apache XTable** (formerly OneTable): An abstraction layer that translates between Delta, Iceberg, and Hudi metadata — write once, read from any format
- Snowflake, BigQuery, and Redshift are all adopting **Iceberg** as a native format for external tables

In 2-3 years, the format you choose may matter less as interoperability improves. But **understanding the differences** is what separates a senior engineer from a junior one in interviews.

### 💡 Interview Insight

> **"Which table format would you choose and why?"**
>
> **Never** say one is universally "better." Instead:
>
> **"It depends on the stack and workload. For a Databricks-centric shop, Delta Lake is the natural choice — tightest integration, best tooling, and the largest community. For multi-engine environments where we use Spark, Trino, and Flink together, I'd lean toward Iceberg because of its engine-agnostic design and features like hidden partitioning and partition evolution. For streaming-heavy CDC workloads with millions of upserts per second, Hudi's Merge-on-Read storage type is hard to beat. That said, the formats are converging — Delta UniForm and Apache XTable are reducing the lock-in concern."**
>
> This answer shows you understand the trade-offs, not just the features.

---

## Screen 7: Quiz — Test Your Lakehouse Knowledge

### Question 1

**What problem does the Lakehouse architecture solve?**

- A) It replaces both data lakes and data warehouses with a new storage system
- B) It combines data lake economics (cheap open storage) with warehouse capabilities (ACID, schema enforcement, governance)
- C) It eliminates the need for ETL pipelines
- D) It makes all queries run in real-time

**✅ Correct Answer: B**

*Explanation: The Lakehouse doesn't replace storage systems — it adds a metadata/transaction layer (Delta, Iceberg, Hudi) on top of existing cheap object storage (S3, ADLS) to provide ACID transactions, schema enforcement, and time travel that previously required an expensive data warehouse.*

---

### Question 2

**How does Delta Lake achieve ACID transactions?**

- A) By using a distributed lock manager that prevents concurrent writes
- B) Through a transaction log (`_delta_log/`) that records every change as an ordered, atomic JSON commit
- C) By writing data to a relational database first and syncing to Parquet
- D) By using write-ahead logs in the Spark driver

**✅ Correct Answer: B**

*Explanation: The `_delta_log/` directory contains sequentially numbered JSON files, each recording which Parquet files were added or removed in that commit. Atomicity comes from the fact that a commit file is written only after all data files are successfully created. Checkpoints (Parquet format) are created every 10 commits to speed up log replay.*

---

### Question 3

**You discover a bad ETL job corrupted your Delta table 3 hours ago. The table has a 7-day VACUUM retention. How do you fix it?**

- A) Delete the Parquet files manually and re-run the ETL
- B) Use `RESTORE TABLE events TO TIMESTAMP AS OF '3-hours-ago'` to roll back to the pre-corruption state
- C) Re-create the table from scratch using a backup
- D) Use `VACUUM` to clean up the corrupted files

**✅ Correct Answer: B**

*Explanation: Since the corruption happened 3 hours ago (within the 7-day retention), all historical Parquet files still exist. `RESTORE TABLE` creates a new commit that reverts the table's file list to the state at the specified timestamp. No data is copied or moved — it just changes which files the table references. VACUUM would NOT help here — it deletes old files, which is the opposite of what you want.*

---

### Question 4

**What is Iceberg's "hidden partitioning" and why does it matter?**

- A) Partitions are encrypted so users can't see the data
- B) Partition transforms (year, month, bucket, etc.) are defined in metadata — users query naturally without knowing the partition structure, and the engine prunes automatically
- C) Partitions are hidden from the query optimizer to prevent biased execution plans
- D) Data is stored without partitions but Iceberg simulates partitioning at read time

**✅ Correct Answer: B**

*Explanation: In traditional Hive-style partitioning, users must filter on partition columns explicitly (WHERE year=2024 AND month=1) or suffer full table scans. With Iceberg, you define transforms like `days(event_ts)` at table creation. Users then query `WHERE event_ts > '2024-01-15'` naturally, and Iceberg's metadata layer automatically determines which partitions to scan. This decouples the physical layout from the logical query, preventing accidental full table scans.*

---

### Question 5

**Your Delta table receives streaming micro-batches every 30 seconds and now has 50,000 small files. What's your maintenance plan?**

- A) Run VACUUM immediately to delete old files
- B) Run OPTIMIZE (with optional ZORDER BY) to compact files, then VACUUM to clean up old files after the retention period
- C) Increase the number of Spark partitions to spread the load
- D) Switch to Iceberg to solve the problem

**✅ Correct Answer: B**

*Explanation: OPTIMIZE compacts the 50,000 small files into fewer, optimally-sized files (~1GB each). If you add ZORDER BY on frequently-queried columns, it also co-locates related data for better data skipping. After OPTIMIZE, old small files still exist (for time travel). VACUUM (run after the retention period) deletes those old files to reclaim storage. Going forward, enable `delta.autoOptimize.optimizeWrite` and `delta.autoOptimize.autoCompact` to prevent the problem from recurring.*

---

### Question 6

**When would you choose Hudi over Delta Lake or Iceberg?**

- A) When you need the best schema evolution support
- B) When your primary workload is streaming CDC/upserts — Hudi's Merge-on-Read storage type is optimized for high write throughput with frequent updates
- C) When you need to query from multiple engines like Trino and Flink
- D) When you want the largest community support

**✅ Correct Answer: B**

*Explanation: Hudi was built at Uber specifically for high-frequency upsert workloads. Its Merge-on-Read (MoR) storage type writes updates as small delta log files (fast appends) instead of rewriting entire Parquet files (like Delta's copy-on-write approach). This gives significantly better write throughput for CDC-heavy workloads. Compaction jobs periodically merge the delta logs into base files. For schema evolution, Iceberg is strongest (A). For multi-engine (C), Iceberg is best. For community (D), Delta leads.*

---

## Key Takeaways for Interviews

1. **Lakehouse = table format layer.** It's not a product — it's an architecture pattern. The table format (Delta/Iceberg/Hudi) is what turns a pile of Parquet files into a proper table with ACID, schema, and time travel.

2. **Delta Lake = Parquet + transaction log.** Know the `_delta_log/` structure (JSON commits + Parquet checkpoints) and how optimistic concurrency control works.

3. **MERGE is the star operation.** Be ready to write a MERGE (upsert) from memory — it comes up in almost every lakehouse interview question.

4. **OPTIMIZE + VACUUM is your maintenance pair.** OPTIMIZE compacts files for read performance, VACUUM cleans up old files for storage cost. Always in that order.

5. **Iceberg's superpower is hidden partitioning + partition evolution.** Users query naturally, and you can change partitioning strategies without rewriting data.

6. **Don't pick favorites — show trade-off thinking.** Delta for Databricks shops, Iceberg for multi-engine environments, Hudi for streaming upserts. Knowing *when* to use each is what makes you senior.
