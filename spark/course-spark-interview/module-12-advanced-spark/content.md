# Module 12: Partitioning, File Formats & Modern Spark

---

## Screen 1: Partitioning Strategy Deep Dive

### Tuning `spark.sql.shuffle.partitions`

Every shuffle operation in Spark — joins, aggregations, `groupBy`, `distinct` — produces output partitions. The number of those partitions is controlled by a single config:

```python
spark.conf.set("spark.sql.shuffle.partitions", 200)  # This is the default
```

**200 is almost never the right number for production.** It's a compromise: small enough to not overwhelm tiny datasets, large enough to not OOM on moderate ones. But for TB-scale jobs, 200 partitions means each partition holds ~5GB of a 1TB shuffle — guaranteed OOM or at minimum massive GC pressure and spill-to-disk.

#### The 128MB Rule

The industry-standard rule of thumb:

```
ideal_partitions = total_shuffle_data_size / 128MB
```

Why 128MB? It's large enough to amortize task scheduling overhead (~50-100ms per task), small enough to fit comfortably in executor memory alongside other work, and aligns well with HDFS/cloud storage block sizes.

**Practical examples:**

| Shuffle Data Size | Default (200) Partition Size | Ideal Partitions | Ideal Partition Size |
|-------------------|------------------------------|------------------|----------------------|
| 10 GB             | 50 MB                        | ~80              | 128 MB               |
| 100 GB            | 500 MB ⚠️                    | ~800             | 128 MB               |
| 1 TB              | 5 GB 💥                      | ~8,000           | 128 MB               |
| 10 TB             | 50 GB 💀                     | ~80,000          | 128 MB               |

Notice that for 10GB of data, the default 200 actually *over-partitions* (50MB each — below ideal). For 100GB+, it dangerously under-partitions.

```python
# In practice, calculate and set dynamically:
shuffle_data_mb = 100_000  # 100 GB estimated
target_partitions = max(shuffle_data_mb // 128, 200)  # never go below 200
spark.conf.set("spark.sql.shuffle.partitions", str(target_partitions))
```

> **How do you estimate shuffle data size?** Check the Spark UI's "Shuffle Write" metric from a previous run. Or estimate: if you're grouping 500GB of raw data down to 100GB of aggregated output, the shuffle is ~100GB.

### Hash Partitioning vs Range Partitioning

When you call `repartition()`, Spark uses **hash partitioning** by default. But there's a second strategy — **range partitioning** — and knowing when to use each is a senior-level skill.

#### Hash Partitioning

```python
# Hash partitioning: each row's partition = hash(key) % num_partitions
df.repartition(100, "user_id")
```

- **How it works:** Spark hashes the partition column value and assigns each row to `hash(value) % N`. Rows with the same key always land in the same partition.
- **Strengths:** Even distribution (assuming non-skewed keys), deterministic placement, perfect for subsequent joins on the same key.
- **Weakness:** No ordering within or across partitions. If you want sorted output, you'll need an explicit sort afterward.
- **Use case:** Pre-partitioning before a join to ensure co-located data. Two DataFrames hash-partitioned by `user_id` with the same partition count can join without a shuffle.

#### Range Partitioning

```python
# Range partitioning: sample data, create boundaries, assign by range
df.repartitionByRange(100, "timestamp")
```

- **How it works:** Spark samples the data to determine range boundaries (e.g., partition 1 gets timestamps Jan 1–Jan 4, partition 2 gets Jan 4–Jan 7, etc.). Each partition contains a contiguous range of values.
- **Strengths:** Data is globally ordered across partitions — partition 1's values are all less than partition 2's. Enables efficient range queries and sorted writes.
- **Weakness:** Susceptible to skew — if 80% of your data falls in one range, one partition gets 80% of the work.
- **Use case:** Writing sorted output files (e.g., time-series data where each file covers a time range), or preparing data for range-based lookups.

**Think of it this way:**
- Hash = randomly but evenly distributing books across shelves by title hash. Fast to find any specific book, but shelves have no logical order.
- Range = organizing books alphabetically across shelves (A-D on shelf 1, E-H on shelf 2...). Easy to scan a range, but some letters have more books.

### Custom Partitioning Patterns

```python
# Partition by column (hash, Spark chooses count based on default shuffle partitions)
df.repartition("user_id")

# Partition by column with explicit count
df.repartition(100, "user_id")

# Partition by multiple columns (compound hash key)
df.repartition(100, "user_id", "date")

# Range partition for sorted output
df.repartitionByRange(100, col("event_time").asc())
```

**`repartition("column")` vs `repartition(N, "column")`:** Without specifying N, Spark uses `spark.sql.shuffle.partitions` (default 200) as the partition count. Always specify N explicitly in production — it makes your intent clear and avoids surprises when someone changes the config.

### Over-Partitioning vs Under-Partitioning

| Problem | Symptom | Root Cause | Fix |
|---------|---------|------------|-----|
| **Over-partitioning** | Thousands of tasks finishing in <100ms, long total job time despite low per-task time | Scheduler overhead dominates. Each task has ~50-100ms of scheduling/serialization cost. 10,000 tiny tasks = 10+ minutes of pure overhead. | Reduce partitions. Target 128MB per partition. |
| **Under-partitioning** | OOM errors, massive GC time, spill-to-disk, low parallelism (few active tasks) | Partitions too large to fit in memory. Fewer partitions than available cores means wasted parallelism. | Increase partitions. No partition should exceed 1GB. |

**The sweet spot:** 128MB–256MB per partition, with total partition count at least 2-3× the total number of available cores for good parallelism.

> 💡 **Interview Insight:** *"The default 200 shuffle partitions works for small data. For production TB-scale jobs, I calculate: total shuffle data size ÷ 128MB = target partition count. This avoids both tiny-task overhead and OOM from oversized partitions. I also verify in the Spark UI that task durations are between 200ms and 2 minutes — that's the healthy range."*

---

## Screen 2: Bucketing — The Hidden Join Optimization

### What Is Bucketing?

Bucketing is a **write-time optimization** that pre-organizes data on disk by a key column. When you bucket a table, Spark:

1. **Hashes** each row's bucket column value
2. **Assigns** it to one of N buckets (files)
3. **Sorts** the data within each bucket (if you specify `sortBy`)

The result: data with the same key value always lives in the same bucket file, and it's already sorted.

**Analogy:** Imagine you run a mail room. Every day, you join employee records with payroll records by employee_id. Without bucketing, every morning you dump both filing cabinets on the floor and re-sort everything to match them up (that's the shuffle). With bucketing, both cabinets are permanently organized by employee_id — drawer 1 has IDs 1-100, drawer 2 has 101-200, etc. Now joining is just opening matching drawers.

### How Bucketing Eliminates Shuffle

```python
# STEP 1: Write both tables with identical bucketing
orders_df.write \
    .bucketBy(256, "user_id") \
    .sortBy("user_id") \
    .saveAsTable("orders_bucketed")

users_df.write \
    .bucketBy(256, "user_id") \
    .sortBy("user_id") \
    .saveAsTable("users_bucketed")

# STEP 2: Join — Spark detects matching bucketing → SKIPS SHUFFLE
result = spark.table("orders_bucketed") \
    .join(spark.table("users_bucketed"), "user_id")
```

**What happens under the hood:**

- Without bucketing: Spark shuffles both 100GB tables → 200GB of network I/O, temporary disk writes, and GC pressure.
- With bucketing: Spark reads bucket 0 from orders + bucket 0 from users → joins locally. Repeats for all 256 buckets. **Zero shuffle.** The data is already co-partitioned and co-sorted.

You can verify in the Spark UI: the Exchange (shuffle) node disappears from the physical plan.

```python
# Check the plan to confirm no exchange:
result.explain()
# Look for "BucketedScan" and absence of "Exchange hashpartitioning"
```

### Bucketing Requirements & Limitations

| Requirement | Detail |
|-------------|--------|
| **Storage** | Must use `saveAsTable()` (Hive metastore). Does NOT work with `.save(path)` or `.parquet(path)`. |
| **Bucket count** | Both tables must have the **exact same** number of buckets. 256 and 512 won't co-optimize. |
| **Bucket column** | Both tables must be bucketed on the **same column(s)**. |
| **Catalog awareness** | Spark must read from the Hive metastore (not direct file path) to know about bucketing metadata. |
| **File format** | Works with Parquet, ORC. The table metadata stores bucketing info. |
| **No dynamic changes** | If you append unbucketed data or change the bucket count, you lose the optimization. |

### Choosing the Right Bucket Count

```python
# Rule of thumb: bucket count = expected_table_size / target_file_size
# For a 100GB table with 256MB target files:
bucket_count = 100_000 // 256  # ≈ 390 → round to 400 or nearest power of 2 (512)
```

- **Too few buckets:** Large files, limited parallelism during reads.
- **Too many buckets:** Small files problem. 10,000 buckets on a 1GB table = 100KB files.
- **Powers of 2** are conventional (128, 256, 512) but not required.

### Bucketing vs Broadcast vs Shuffle Join — Decision Matrix

| Scenario | Best Strategy | Why |
|----------|--------------|-----|
| Small table (< 10MB) + Large table | **Broadcast join** | Small table fits in memory, no shuffle needed |
| Large + Large, joined once | **Shuffle join** (let AQE optimize) | Bucketing setup cost not justified for one-time use |
| Large + Large, joined repeatedly on same key | **Bucketing** | Amortize the one-time bucketing cost across many queries |
| Large + Large, different join keys each time | **Shuffle join** | Can't bucket for every possible key combination |

> 💡 **Interview Insight:** *"Bucketing is the answer when two LARGE tables are joined frequently on the same key. Pre-bucketing eliminates shuffle at query time — it's like pre-sorting your filing cabinet so you never have to reorganize. The key requirements: same bucket column, same bucket count, and data must be read through the Hive metastore. It's a write-time investment that pays off at read time."*

---

## Screen 3: File Formats — Parquet vs ORC vs Avro (Deep Dive)

### Columnar vs Row-Based: When Architecture Matters

This isn't just trivia — the storage layout determines I/O patterns, compression ratios, and which workloads are fast vs slow.

**Columnar (Parquet, ORC):**
```
Row 1:  Alice  | 28 | NYC
Row 2:  Bob    | 35 | LA
Row 3:  Carol  | 42 | NYC

Stored as:  [Alice, Bob, Carol] [28, 35, 42] [NYC, LA, NYC]
            ← names column →   ← ages →     ← cities →
```
- **Read benefit:** `SELECT avg(age) FROM users` reads ONLY the age column — skips names and cities entirely. On a 100-column table, reading 3 columns means ~97% I/O savings.
- **Compression benefit:** Same-type values grouped together compress dramatically. A column of ages (28, 35, 42, 29, 31...) compresses much better than mixed-type rows (Alice, 28, NYC, Bob, 35, LA...). Dictionary encoding turns repeated city names into integer lookups.
- **Write cost:** Writes are slower — each row must be split across column buffers, then flushed together. But for analytics, you write once and read thousands of times.

**Row-Based (Avro, CSV, JSON):**
```
Stored as:  [Alice, 28, NYC] [Bob, 35, LA] [Carol, 42, NYC]
            ← complete row → ← complete row → ← complete row →
```
- **Write benefit:** Appending a row is simple — just serialize and append. No columnar buffering needed. Low-latency writes.
- **Read cost:** Reading one column means reading every column. On wide tables, this is catastrophic for analytics.
- **Use case:** Streaming, CDC (change data capture), message queues — where you write frequently and often need the full row.

### Parquet Internals: Row Groups, Column Chunks, and Pages

Understanding Parquet's internal structure explains *why* it's fast and how to tune it:

```
┌─────────────────── Parquet File ───────────────────┐
│                                                     │
│  ┌─── Row Group 1 (default: 128MB) ──────────────┐ │
│  │  Column Chunk: name   [min=Alice, max=Eve]     │ │
│  │    Page 1 (1MB): [Alice, Bob, Carol...]        │ │
│  │    Page 2 (1MB): [Dave, Eve...]                │ │
│  │  Column Chunk: age    [min=22, max=65]         │ │
│  │    Page 1 (1MB): [28, 35, 42...]               │ │
│  │  Column Chunk: city   [min=LA, max=NYC]        │ │
│  │    Page 1 (1MB): [NYC, LA, NYC...]             │ │
│  └────────────────────────────────────────────────┘ │
│                                                     │
│  ┌─── Row Group 2 ───────────────────────────────┐  │
│  │  ...                                           │  │
│  └────────────────────────────────────────────────┘  │
│                                                     │
│  Footer: schema, row group offsets, column stats     │
└─────────────────────────────────────────────────────┘
```

**Key concepts:**

- **Row Group** (~128MB default): A horizontal slice of the table. Each row group contains all columns for a subset of rows. Spark maps each row group to one task.
- **Column Chunk**: One column's data within a row group. Contains min/max statistics.
- **Page** (~1MB default): The unit of compression and encoding. Pages within a column chunk are individually compressed.
- **Footer**: Metadata at the end of the file — schema, row group locations, and per-column-chunk statistics.

**Why this matters for performance:**

1. **Data skipping via min/max stats:** If you filter `WHERE age > 60` and a column chunk's max age is 45, Spark skips that entire row group without reading it. This is *not* the same as predicate pushdown (which pushes filters to the reader). This is **data skipping** — the reader uses statistics to avoid decompressing data entirely.

2. **Row group sizing:** `spark.sql.parquet.rowGroupSize` (default 128MB). Larger row groups = fewer groups per file = less metadata overhead but coarser data skipping. Smaller = finer skipping but more overhead.

```python
# Tuning row group size (rarely needed, but good to know)
spark.conf.set("spark.sql.parquet.rowGroupSize", str(128 * 1024 * 1024))  # 128MB
```

### ORC vs Parquet: The Nuanced Comparison

Both are columnar, both are fast. The differences are ecosystem and features:

| Feature | Parquet | ORC |
|---------|---------|-----|
| **Origin** | Twitter + Cloudera (2013) | Hortonworks for Hive (2013) |
| **Best with** | Spark, Flink, Trino, BigQuery, Snowflake, Athena | Hive, Presto (Meta fork) |
| **Nested data** | Excellent (Dremel encoding) | Good but less mature |
| **Compression** | Snappy (default), GZIP, ZSTD, LZ4 | ZLIB (default), Snappy, ZSTD |
| **Built-in indexes** | Min/max stats only | Min/max + **bloom filters** + row-level index |
| **ACID support** | Via Delta Lake / Iceberg (external) | Built-in with Hive ACID |
| **Predicate pushdown** | Via column stats | Via column stats + bloom filters |
| **Schema evolution** | Add/rename columns | Add columns |
| **Default in Spark** | ✅ Yes (`spark.sql.sources.default=parquet`) | No (must specify) |

**Bloom filters in ORC** are worth calling out: a bloom filter is a probabilistic data structure that can tell you "this value is DEFINITELY NOT in this stripe" or "this value MIGHT be in this stripe." For high-cardinality columns (like user_id), bloom filters dramatically reduce I/O by skipping stripes that don't contain the target value.

```python
# Writing ORC with bloom filters
df.write.format("orc") \
    .option("orc.bloom.filter.columns", "user_id") \
    .option("orc.bloom.filter.fpp", "0.05") \  # 5% false positive probability
    .save("/data/users_orc")
```

> Parquet is adding bloom filter support (PARQUET-1823), but it's not yet as mature or widely adopted as ORC's implementation.

### Avro: The Streaming & Schema Evolution Champion

Avro is fundamentally different from Parquet/ORC — it's row-based, designed for serialization rather than analytics.

**Why Avro for Kafka/streaming:**

1. **Schema registry integration:** Each Avro message carries a schema ID (not the full schema — just a 4-byte integer). The consumer looks up the full schema from a central registry. This means: tiny message overhead + guaranteed schema compatibility.

2. **Schema evolution:** Avro natively handles adding fields (with defaults), removing fields, renaming fields (via aliases). Producers and consumers can evolve independently as long as they follow compatibility rules.

3. **Compact binary encoding:** Avro's binary format doesn't include field names (unlike JSON). A row with 20 fields is just 20 values packed sequentially — the schema tells you what each position means.

```python
# Reading Avro in Spark
df = spark.read.format("avro").load("/data/events.avro")

# Writing Avro
df.write.format("avro").save("/data/events_out.avro")

# With schema registry (for Kafka)
df = spark.readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "broker:9092") \
    .option("subscribe", "events") \
    .load()
# Deserialize Avro payload using from_avro() with schema registry
```

### Format Decision Matrix

| Use Case | Best Format | Why |
|----------|------------|-----|
| Analytics / OLAP queries | **Parquet** | Columnar, great compression, Spark-native, ecosystem standard |
| Hive-centric environment | **ORC** | Hive-optimized, bloom filters, built-in ACID |
| Kafka message serialization | **Avro** | Row-based, schema registry, compact, schema evolution |
| Schema-heavy CDC pipelines | **Avro → Parquet** | Avro for transport (change events), Parquet for storage (analytics) |
| Data lake storage | **Parquet + Delta/Iceberg** | Columnar efficiency + ACID + time travel + schema enforcement |
| ML feature stores | **Parquet** | Fast columnar reads for feature vectors, good pandas/Arrow interop |
| Archival / compliance | **Parquet + ZSTD** | Best compression ratio for long-term cold storage |

> 💡 **Interview Insight:** *"Never say 'Parquet is always best.' The senior answer is: 'Parquet for analytics reads, Avro for streaming transport, ORC if you're in a Hive-heavy environment.' The key distinction is columnar vs row-based — analytics reads benefit from columnar (read only needed columns), while streaming writes benefit from row-based (append a full record quickly). In modern data lakes, you often see Avro for the ingestion layer and Parquet for the serving layer."*

---

## Screen 4: Small Files Problem & Compaction

### The Problem

Imagine a directory listing like this:

```
/data/events/date=2024-01-15/
    part-00000.parquet    (47 KB)
    part-00001.parquet    (12 KB)
    part-00002.parquet    (89 KB)
    ... 
    part-04999.parquet    (23 KB)
    → 5,000 files, average size: 34 KB
```

This is the **small files problem**, and it's the most common production Spark issue. It silently degrades performance until something breaks.

### Why Small Files Are Devastating

1. **File open/close overhead:** Each file requires a separate metadata request to the storage system (HDFS NameNode call or S3 HEAD + GET). Reading 5,000 × 34KB files is far slower than reading 1 × 170MB file, even though the total data is identical.

2. **Driver-side metadata explosion:** When Spark plans a read, the driver lists all files, parses their footers, and builds a scan plan — all in the driver's JVM. Millions of small files can OOM the driver before a single executor starts working.

3. **NameNode / S3 rate limits:** HDFS NameNode stores all file metadata in memory — millions of tiny files consume NameNode heap. S3 has request rate limits (~5,500 GETs/second per prefix) — 500,000 files means 90+ seconds just for listing.

4. **Poor compression:** Parquet/ORC column chunks need enough rows to build effective dictionaries and run-length encoding. A 34KB file might have 100 rows — not enough for meaningful compression.

5. **Task overhead:** Each file may become a separate Spark task. 50,000 tiny tasks with 100ms scheduling overhead = 83 minutes of pure overhead.

### How Small Files Are Created

| Source | Mechanism |
|--------|-----------|
| **Structured Streaming** | Each micro-batch (e.g., every 10 seconds) writes a new set of files. 8,640 batches/day × files per batch = hundreds of thousands of files. |
| **Over-partitioned writes** | `df.write.partitionBy("date", "hour", "category")` with 365 days × 24 hours × 50 categories = 438,000 partition directories, many with tiny files. |
| **High shuffle partitions** | Default 200 shuffle partitions → 200 output files, even if total output is 500MB (2.5MB each). |
| **Frequent appends** | Hourly ETL jobs appending to the same table without compaction. |

### Detection

```python
# Quick diagnostic: check file sizes in a directory
import subprocess
result = subprocess.run(
    ["hadoop", "fs", "-ls", "/data/events/date=2024-01-15/"],
    capture_output=True, text=True
)
# Or in Spark:
files = spark.read.parquet("/data/events/date=2024-01-15/").inputFiles()
print(f"File count: {len(files)}")

# Rule of thumb:
# < 10 MB average file size → small files problem
# 128 MB - 1 GB → healthy range
# > 2 GB per file → consider splitting (though less critical than small files)
```

### Solutions

#### 1. Coalesce Before Write

```python
# Calculate target file count
output_size_mb = 5000  # 5 GB estimated output
target_file_size_mb = 256
target_files = max(output_size_mb // target_file_size_mb, 1)  # 20 files

df.coalesce(target_files).write.mode("overwrite").parquet("/data/events/")
```

Best for: Batch jobs where you know the approximate output size. Cheap operation (no shuffle) when reducing partitions.

#### 2. `maxRecordsPerFile` — Size-Based File Limiting

```python
# Spark automatically splits output into files of ~N records each
df.write \
    .option("maxRecordsPerFile", 1_000_000) \
    .mode("append") \
    .parquet("/data/events/")
```

Best for: When you don't know the output size in advance, or when partition skew means some partitions have far more data than others. Spark will split large partitions into multiple files but won't combine small ones — so this prevents large files but doesn't fix existing small files.

#### 3. Compaction Job (Post-Hoc Fix)

```python
# Read all the small files → write as fewer large files
small_files_path = "/data/events/date=2024-01-15/"
df = spark.read.parquet(small_files_path)

# Count rows to estimate good partition count
row_count = df.count()
# Assume ~1KB per row → estimate total size → divide by 256MB
estimated_mb = row_count * 1 / 1024  # rough estimate
target_files = max(int(estimated_mb / 256), 1)

df.coalesce(target_files).write.mode("overwrite").parquet(small_files_path + "_compacted/")
# Then swap directories (rename old, rename compacted to original)
```

Best for: Fixing an existing small files mess. Schedule as a periodic maintenance job.

#### 4. Write-Partition Strategy: Think Before You `partitionBy`

```python
# DANGEROUS: high-cardinality partitioning
df.write.partitionBy("user_id").parquet(path)  # 10M users = 10M directories! 💀

# SAFE: low-cardinality partitioning
df.write.partitionBy("date").parquet(path)  # 365 partitions/year ✓

# CAREFUL: multi-level partitioning
df.write.partitionBy("date", "hour").parquet(path)  # 8,760 partitions/year — monitor file sizes

# SMART: combine coalesce with partitionBy
df.repartition("date").write.partitionBy("date").parquet(path)
# repartition by the same key ensures each date partition gets reasonable file count
```

**The key insight:** `partitionBy` controls *directory structure* on disk (for partition pruning at read time). `repartition`/`coalesce` controls *how many files* go into each directory. Use them together:

```python
# Best practice: repartition by the partition column, then write
df.repartition(100, "date") \
    .write \
    .partitionBy("date") \
    .parquet("/data/events/")
# Result: ~100 reasonably-sized files distributed across date directories
```

#### 5. Streaming-Specific Solutions

```python
# For Structured Streaming: use trigger once/availableNow for batch-like compaction
spark.readStream.format("kafka").load() \
    .writeStream \
    .trigger(availableNow=True) \  # Process all available data, then stop
    .option("checkpointLocation", "/checkpoints/compact") \
    .start("/data/events_compacted/")

# Or: write streaming to Delta and run OPTIMIZE periodically (cross-ref Module 10)
```

> 💡 **Interview Insight:** *"Small files are the most common production Spark problem. My playbook: for batch, coalesce before write targeting 256MB files. For streaming, I use Delta with periodic OPTIMIZE. For existing messes, I run compaction jobs. The critical metric is average file size — healthy is 128MB to 1GB. I also avoid high-cardinality partitionBy columns and always pair partitionBy with a repartition on the same key."*

---

## Screen 5: Dynamic Partition Pruning (DPP)

### The Problem DPP Solves

Consider this classic star-schema query:

```sql
SELECT sum(s.amount)
FROM sales s                       -- fact table: 10 TB, partitioned by date_key
JOIN dim_date d ON s.date_key = d.key
WHERE d.year = 2024                -- filter on the dimension table
```

**Without DPP (Spark 2.x behavior):**
1. Spark reads ALL partitions of `sales` (10 TB) — because there's no direct filter on `sales`.
2. Spark broadcasts the small `dim_date` table (filtered to year=2024).
3. Spark joins them — and throws away 9+ TB of sales data that doesn't match.

The filter `d.year = 2024` tells us *implicitly* which `date_key` values we need, but Spark 2.x couldn't propagate that insight to the scan of `sales`.

**With DPP (Spark 3.0+):**
1. Spark first resolves the filter: `dim_date WHERE year = 2024` → gets, say, 365 date_key values.
2. Spark **injects these 365 values as a runtime filter** on the `sales` scan.
3. Spark reads ONLY the 365 matching partitions of `sales` (~3% of the data).
4. The join proceeds on dramatically less data.

**Result:** 10 TB scan → ~300 GB scan. Potentially 30× faster.

### How DPP Works Under the Hood

```
┌─────────────────────────────────────────────────────┐
│                    Query Plan                        │
│                                                     │
│  ┌─────────────┐         ┌───────────────────────┐  │
│  │ dim_date     │         │ sales (10 TB)         │  │
│  │ WHERE yr=2024│         │ partitioned by date_key│ │
│  └──────┬──────┘         └───────────┬───────────┘  │
│         │                            │               │
│         │    DPP: extract date_keys  │               │
│         ├───────────────────────────►│               │
│         │    from dim_date result    │               │
│         │    inject as runtime filter│               │
│         │                            │               │
│         └──────────┬─────────────────┘               │
│                    │ Join (only matching data)        │
│                    ▼                                 │
│              Result                                  │
└─────────────────────────────────────────────────────┘
```

Spark physically rewrites the scan on `sales` to include a subquery filter:

```sql
-- Conceptually, DPP transforms the plan to:
SELECT sum(s.amount)
FROM sales s
WHERE s.date_key IN (SELECT d.key FROM dim_date d WHERE d.year = 2024)
JOIN dim_date d ON s.date_key = d.key
WHERE d.year = 2024
```

### Configuration

```python
# DPP is enabled by default in Spark 3.0+
spark.conf.set("spark.sql.optimizer.dynamicPartitionPruning.enabled", "true")  # default

# Controls when DPP kicks in:
# The dimension side must be broadcastable (or small enough to collect)
spark.conf.set("spark.sql.optimizer.dynamicPartitionPruning.useStats", "true")
spark.conf.set("spark.sql.optimizer.dynamicPartitionPruning.fallbackFilterRatio", "0.5")
# If DPP would prune less than 50% of partitions, Spark skips it (overhead not worth it)
```

### Requirements for DPP to Activate

| Requirement | Why |
|-------------|-----|
| **Pruned table must be partitioned** by the join key | DPP prunes at the partition level — without partitions, there's nothing to prune |
| **Dimension side must be broadcastable** (or small enough to build a filter from) | Spark needs to materialize the filter values first |
| **Join type: inner or left semi** | DPP doesn't work with outer joins (you can't skip partitions when NULLs matter) |
| **Filter must be on the dimension side** | The filter values flow from dimension → fact, not the reverse |
| **Spark 3.0+** | Not available in Spark 2.x |

### DPP vs Static Predicate Pushdown

| Aspect | Static Predicate Pushdown | Dynamic Partition Pruning |
|--------|--------------------------|--------------------------|
| **When filter is known** | At planning time (from your WHERE clause) | At runtime (derived from join results) |
| **Example** | `WHERE date = '2024-01-15'` | `WHERE dim.year = 2024` (date values derived at runtime) |
| **Mechanism** | Push filter to Parquet/ORC reader → skip row groups | Build filter from broadcast result → skip partitions |
| **Works without joins** | Yes | No — requires a join to derive the filter |
| **Spark version** | All versions | 3.0+ |

**They're complementary, not alternatives.** You can have both on the same query:

```sql
SELECT * FROM sales s
JOIN dim_date d ON s.date_key = d.key
WHERE d.year = 2024          -- DPP: prunes sales partitions via join
  AND s.amount > 1000        -- Predicate pushdown: pushed to Parquet reader
```

### Verifying DPP in the Query Plan

```python
# Check if DPP is being applied:
spark.sql("""
    SELECT sum(s.amount) FROM sales s
    JOIN dim_date d ON s.date_key = d.key
    WHERE d.year = 2024
""").explain(True)

# Look for: "DynamicPruningExpression" in the physical plan
# It appears as a subquery filter on the fact table scan
```

> 💡 **Interview Insight:** *"DPP is the runtime cousin of predicate pushdown. Static pushdown handles explicit filters in your query — like `WHERE date = '2024-01-15'`. DPP handles implicit filters that flow through joins — like joining with a date dimension that filters to year 2024, which automatically prunes the fact table's date partitions. It's a Spark 3.0+ feature, requires the fact table to be partitioned by the join key, and needs the dimension side to be broadcastable. In star-schema queries, DPP can reduce I/O by 90%+."*

---

## Screen 6: Pandas UDFs & PySpark Performance

### The PySpark Serialization Problem

PySpark has an architectural bottleneck: Spark's engine runs on the **JVM** (Java/Scala), but your Python code runs in a **separate Python process**. Data must cross this boundary.

```
┌────────────┐    pickle/serialize    ┌────────────┐
│  JVM       │  ──── row by row ────► │  Python    │
│  Executor  │                        │  Worker    │
│            │  ◄── row by row ────   │            │
│  (fast)    │    deserialize/unpickle│  (slow)    │
└────────────┘                        └────────────┘
```

**Regular Python UDF cost per row:**
1. JVM serializes the row to bytes (pickle format)
2. Bytes sent to Python worker via socket
3. Python deserializes bytes to Python objects
4. Your UDF function runs on Python objects
5. Result serialized back to bytes
6. Bytes sent back to JVM
7. JVM deserializes result

**This happens FOR EVERY ROW.** On a 100M row dataset, that's 100M serialize/deserialize round trips. It's easily 10-100× slower than an equivalent Scala UDF.

```python
# ❌ SLOW: Regular Python UDF (row-by-row serialization)
from pyspark.sql.functions import udf
from pyspark.sql.types import DoubleType

@udf(returnType=DoubleType())
def slow_normalize(value, mean, std):
    return (value - mean) / std

# Each row: JVM → pickle → socket → Python → compute → socket → unpickle → JVM
df.withColumn("norm", slow_normalize(df.value, df.mean_val, df.std_val))
```

### Pandas UDFs: The Arrow-Based Solution

Pandas UDFs (introduced in Spark 2.3, redesigned in Spark 3.0) solve this by using **Apache Arrow** — a columnar, zero-copy in-memory format that both JVM and Python understand.

```
┌────────────┐    Arrow columnar batch   ┌────────────┐
│  JVM       │  ──── 10,000 rows ──────► │  Python    │
│  Executor  │    (zero-copy possible)   │  Worker    │
│            │  ◄── 10,000 rows ───────  │  (pandas)  │
│            │    Arrow columnar batch    │            │
└────────────┘                           └────────────┘
```

**Key differences:**
- Data transferred in **columnar batches** (thousands of rows at once) instead of one row at a time
- Arrow format is understood by both JVM and Python — **no pickle serialization**
- Python side operates on **pandas Series/DataFrames** — vectorized NumPy operations, not Python loops
- Result: **10-100× faster** than regular Python UDFs

### Types of Pandas UDFs

#### 1. Series → Series (Most Common)

Takes a pandas Series (column), returns a pandas Series. Operates element-wise across a batch.

```python
import pandas as pd
from pyspark.sql.functions import pandas_udf

@pandas_udf("double")
def normalize(s: pd.Series) -> pd.Series:
    """Vectorized normalization — runs on batches of rows."""
    return (s - s.mean()) / s.std()

# Usage: applies to each batch of the column, returns transformed column
df.withColumn("normalized_value", normalize(df["value"]))
```

#### 2. Iterator of Series → Iterator of Series (Expensive Init)

When your UDF needs expensive one-time setup (loading an ML model, connecting to a service):

```python
from typing import Iterator

@pandas_udf("double")
def predict(batch_iter: Iterator[pd.Series]) -> Iterator[pd.Series]:
    # LOAD MODEL ONCE — not per batch!
    model = load_heavy_ml_model("/models/latest")  # Runs once per partition
    
    for batch in batch_iter:
        yield pd.Series(model.predict(batch.values.reshape(-1, 1)))

df.withColumn("prediction", predict(df["features"]))
```

The iterator pattern ensures the model loads once per Python worker, not once per batch. Critical for ML inference at scale.

#### 3. Grouped Map: `applyInPandas()`

Apply an arbitrary pandas function to each group independently. The most flexible — you get a full pandas DataFrame per group and return a full DataFrame.

```python
def train_per_store(pdf: pd.DataFrame) -> pd.DataFrame:
    """Train a separate model for each store — full pandas power per group."""
    from sklearn.linear_model import LinearRegression
    
    model = LinearRegression()
    model.fit(pdf[["day_of_week", "is_holiday"]], pdf["sales"])
    pdf["predicted_sales"] = model.predict(pdf[["day_of_week", "is_holiday"]])
    return pdf

# Apply to each store_id group independently
result_schema = "store_id long, day_of_week int, is_holiday int, sales double, predicted_sales double"
df.groupBy("store_id").applyInPandas(train_per_store, schema=result_schema)
```

#### 4. Map: `mapInPandas()`

Apply a pandas function to each partition (not group). Useful for partition-level transformations.

```python
def add_row_numbers(batch_iter: Iterator[pd.DataFrame]) -> Iterator[pd.DataFrame]:
    for pdf in batch_iter:
        pdf["row_num"] = range(len(pdf))
        yield pdf

df.mapInPandas(add_row_numbers, schema="id long, value double, row_num int")
```

### The Performance Hierarchy

Always prefer options higher in this list:

```
1. Native Spark SQL functions          ← FASTEST (JVM-only, no serialization)
   df.withColumn("upper", upper(col("name")))

2. Spark SQL expressions               ← FAST (JVM-only)
   df.selectExpr("name", "value * 2 as doubled")

3. Pandas UDFs (Arrow-based)           ← GOOD (vectorized, Arrow batches)
   @pandas_udf("double")
   def my_func(s: pd.Series) -> pd.Series: ...

4. Regular Python UDFs                 ← SLOW (row-by-row pickle serialization)
   @udf(returnType=DoubleType())
   def my_func(x): ...

5. RDD .map() with Python lambdas      ← SLOWEST (full Python path, no Catalyst)
   rdd.map(lambda x: x[0].upper())
```

**Rule:** If Spark has a built-in function for it (`upper()`, `split()`, `regexp_extract()`, `when()`, etc.), use it. There are 300+ built-in functions. Only reach for UDFs when you need custom logic that can't be expressed with built-ins.

```python
# Check available built-in functions:
from pyspark.sql import functions as F
print(dir(F))  # 300+ functions — check here before writing a UDF
```

### Spark Connect (Spark 3.4+)

Spark Connect is a thin-client architecture that decouples the client from the Spark cluster:

```
┌──────────────┐     gRPC      ┌──────────────────┐
│ Python Client│ ◄───────────► │ Spark Server     │
│ (any env)    │  (DataFrame   │ (cluster)        │
│ laptop/IDE   │   operations  │                  │
│ no Spark JAR │   as protobuf)│                  │
└──────────────┘               └──────────────────┘
```

**What it changes:**
- No need to bundle Spark JARs in your Python environment
- Connect from Jupyter, VS Code, any Python script — just `pip install pyspark[connect]`
- Cluster upgrades don't require client updates (protocol is stable)
- Multiple clients can share one Spark cluster

```python
# Spark Connect client — no local Spark installation needed
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .remote("sc://spark-server:15002") \
    .getOrCreate()

# Use exactly like regular PySpark — same API
df = spark.read.parquet("/data/events")
df.groupBy("date").count().show()
```

> 💡 **Interview Insight:** *"If asked about PySpark performance, I give the three-level answer: (1) Use native Spark SQL functions — they run on the JVM, no serialization overhead, always fastest. There are 300+ built-in functions. (2) If you need Python logic, use Pandas UDFs — they're vectorized via Apache Arrow, processing batches of rows instead of one at a time, 10-100× faster than regular UDFs. (3) Never use plain Python UDFs or RDD .map() with Python lambdas in production — the row-by-row JVM↔Python pickle serialization is a performance killer. For ML inference, I use Iterator Pandas UDFs to load the model once and score in batches."*

---

## Screen 7: Quiz — Test Your Mastery

### Question 1: Shuffle Partition Tuning

The default `spark.sql.shuffle.partitions` is 200. Your job shuffles **50 GB** of data. How many partitions should you target, and why?

- A) Keep the default 200 — it's fine for most jobs
- B) Set to 50 — one partition per GB
- **C) Set to ~400 — using the 128MB rule: 50,000 MB ÷ 128 MB ≈ 390, round to 400** ✅
- D) Set to 2,000 — more partitions is always better

**Explanation:** The 128MB rule gives us 50,000 MB ÷ 128 MB ≈ 390. Rounding to 400 gives clean, well-sized partitions. 200 would mean 250MB partitions (a bit large but survivable). 2,000 would mean 25MB partitions — too small, causing scheduler overhead. Option C balances parallelism and per-task efficiency.

---

### Question 2: Bucketing for Shuffle Elimination

Two large tables (`orders`: 100 GB, `users`: 80 GB) are joined daily on `user_id`. What optimization eliminates the shuffle entirely?

- A) Broadcast the smaller table
- B) Cache both tables in memory
- **C) Bucket both tables by `user_id` with the same bucket count using `saveAsTable()`** ✅
- D) Repartition both tables by `user_id` before each join

**Explanation:** At 80 GB, `users` is far too large to broadcast (default limit is 10 MB). Caching doesn't eliminate shuffle — it just speeds up repeated reads. Repartitioning before each join still performs a shuffle every time. Bucketing pre-partitions and pre-sorts data on disk — when both tables share the same bucket count and key, Spark recognizes this and joins without shuffling. The key requirements: same bucket column (`user_id`), same bucket count, and read via `spark.table()` (not file paths).

---

### Question 3: Choosing Avro Over Parquet

When would you choose Avro over Parquet?

- A) For analytics dashboards with columnar aggregations
- **B) For Kafka message serialization with schema registry integration** ✅
- C) When you need the best compression ratio for cold storage
- D) For Spark SQL tables that are read frequently

**Explanation:** Avro is row-based — perfect for serializing individual records in streaming. It integrates with Confluent Schema Registry (each message carries a schema ID, not the full schema), supports schema evolution via compatibility rules, and has compact binary encoding. Parquet is superior for analytics (columnar, better compression for reads), but Avro excels at write-heavy, record-level streaming workloads.

---

### Question 4: Small Files Diagnosis and Fix

Your streaming job writes 500 files per micro-batch. After one week, you have **500,000 files** averaging 45 KB each. What's the problem and how do you fix it?

- A) This is normal for streaming — just add more executors
- B) Switch from Parquet to CSV for smaller files
- C) Increase `spark.sql.shuffle.partitions` to create more partitions
- **D) Small files problem. Fix with: coalesce before write, `maxRecordsPerFile`, or Delta OPTIMIZE. Target 128 MB–1 GB per file.** ✅

**Explanation:** 500,000 files at 45 KB each is a severe small files problem. Each file open has metadata overhead (S3 HEAD + GET), the driver may OOM on file listing, and compression is poor with so few rows per file. Solutions: (1) For the streaming write itself — reduce output partitions via `coalesce()` in each micro-batch. (2) Use `maxRecordsPerFile` to control file sizes. (3) If using Delta Lake, run `OPTIMIZE` periodically for automated compaction. (4) Run a periodic compaction job that reads small files and rewrites as larger ones. Never increase shuffle partitions — that creates MORE files, not fewer.

---

### Question 5: Dynamic Partition Pruning

What is Dynamic Partition Pruning (DPP), and how does it differ from static predicate pushdown?

- A) DPP pushes WHERE clause filters to the storage layer — same as predicate pushdown
- B) DPP dynamically adjusts the number of partitions during a shuffle
- **C) DPP derives filter values from a join's broadcast side at runtime to prune partitions on the fact table — predicate pushdown handles explicit filters known at planning time** ✅
- D) DPP is another name for AQE's coalesce shuffle partitions

**Explanation:** Static predicate pushdown handles filters you explicitly write (`WHERE date = '2024-01-15'`) — known at planning time, pushed to the file reader. DPP handles *implicit* filters derived from joins: if you join sales with dim_date and filter `dim_date.year = 2024`, DPP resolves the matching date_key values at runtime and uses them to skip sales partitions that don't match. Requirements: fact table partitioned by join key, dimension side broadcastable, Spark 3.0+. They're complementary — both can apply to the same query.

---

### Question 6: PySpark UDF Performance

A PySpark job using regular Python UDFs is **50× slower** than the equivalent Scala version. Why, and how do you fix it?

- A) Python is inherently 50× slower than Scala — nothing you can do
- B) Use `cache()` on the input DataFrame
- C) Increase executor memory to compensate for Python's overhead
- **D) Regular Python UDFs serialize data row-by-row between JVM and Python (pickle). Replace with Pandas UDFs (Arrow-based vectorized, 10-100× faster) or native Spark SQL functions (no serialization at all).** ✅

**Explanation:** The bottleneck isn't Python's computational speed — it's the serialization overhead. Each row is pickled in the JVM, sent via socket to a Python worker, unpickled, processed, pickled again, sent back, and unpickled in the JVM. For 100M rows, that's 100M round trips. Pandas UDFs fix this by transferring data in Arrow columnar batches (thousands of rows at once, zero-copy possible) and processing with vectorized pandas/NumPy operations. Native Spark SQL functions are even better — they run entirely on the JVM with zero Python involvement. Caching doesn't help because the bottleneck is in the UDF processing, not the input read.

---

## Summary: Key Takeaways for Interviews

| Topic | One-Liner Answer |
|-------|-----------------|
| **Shuffle partitions** | Default 200 is wrong for production. Calculate: shuffle data ÷ 128MB. |
| **Hash vs Range partitioning** | Hash for even distribution & joins. Range for sorted output & range queries. |
| **Bucketing** | Pre-partition + pre-sort on disk. Eliminates shuffle for repeated large-large joins on same key. |
| **Parquet vs ORC vs Avro** | Parquet for analytics, ORC for Hive, Avro for streaming. Never say "always Parquet." |
| **Small files** | Most common production issue. Coalesce before write, maxRecordsPerFile, Delta OPTIMIZE. |
| **DPP** | Runtime cousin of predicate pushdown. Derives join filters to prune fact table partitions. |
| **Pandas UDFs** | Arrow-based vectorized, 10-100× faster than regular Python UDFs. Always prefer native Spark functions first. |
| **Spark Connect** | Thin client (3.4+). gRPC-based, no local Spark JARs needed. |
