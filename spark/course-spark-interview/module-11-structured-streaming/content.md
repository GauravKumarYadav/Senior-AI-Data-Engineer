# Module 11: Structured Streaming Mastery

---

## Screen 1: The Structured Streaming Model

### The Unbounded Table Mental Model

Forget everything you know about traditional stream processing frameworks that force you to think in terms of "events arriving one at a time." Structured Streaming flips the model: **every streaming source is just a table that keeps getting new rows appended to it.** Your query runs against this ever-growing table, and Spark figures out how to execute it incrementally.

Here's the mental model you need to internalize:

```
Input Table          →  Query (DataFrame ops)  →  Result Table  →  Output Sink
(unbounded, growing)    (same API as batch!)       (updated each    (Kafka, Delta,
                                                    trigger)         console, etc.)
```

Think of it like a Google Sheet that other people keep adding rows to. You've written a formula (your query) that summarizes the data. Every time new rows appear, the spreadsheet recalculates. Structured Streaming does exactly this — but at scale, with fault tolerance, and with the full power of Catalyst and Tungsten under the hood.

### Why This Matters

The brilliance of this design is **API unification**. The same DataFrame/Dataset/SQL code you write for a batch job works for streaming — no new abstractions to learn, no special "streaming operators." You write a `groupBy().count()` and Spark handles the incremental execution automatically.

This is a **fundamentally different execution model** from DStreams. DStreams were RDD-based micro-batches — you were essentially running a loop of tiny batch jobs with no native event-time support. Structured Streaming is built on the Catalyst optimizer and Tungsten execution engine, which means your streaming query gets:

- **Predicate pushdown** into sources (e.g., only read relevant Kafka partitions)
- **Whole-stage code generation** (compiled bytecode, not interpreted row-by-row)
- **Logical plan optimization** (filter reordering, projection pruning)
- **Native event-time windowing** (not bolted on as an afterthought)

### End-to-End PySpark Example: Kafka → Windowed Aggregation → Console

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import window, col

spark = SparkSession.builder.appName("StructuredStreamingDemo").getOrCreate()

# Step 1: Read from Kafka — returns a DataFrame with columns:
# key, value, topic, partition, offset, timestamp, timestampType
df = (spark.readStream
    .format("kafka")
    .option("kafka.bootstrap.servers", "broker1:9092,broker2:9092")
    .option("subscribe", "clickstream")        # subscribe to topic
    .option("startingOffsets", "latest")        # or "earliest" for backfill
    .load()
)

# Step 2: Parse and transform — same DataFrame API as batch!
parsed = (df
    .selectExpr(
        "CAST(value AS STRING) AS raw_event",  # Kafka value is binary
        "timestamp AS kafka_timestamp"          # Kafka ingestion timestamp
    )
    .where("raw_event IS NOT NULL")            # filter out bad records
)

# Step 3: Windowed aggregation — count events per 5-minute tumbling window
windowed_counts = (parsed
    .groupBy(
        window("kafka_timestamp", "5 minutes"),  # tumbling window
        "raw_event"
    )
    .count()
)

# Step 4: Write results to console (for debugging)
query = (windowed_counts.writeStream
    .outputMode("update")        # only emit changed rows
    .format("console")           # sink = console (dev only)
    .option("truncate", False)   # show full values
    .start()
)

query.awaitTermination()  # block until query is stopped or fails
```

**Key observation:** If you removed `.readStream` and `.writeStream` and replaced them with `.read` and `.write`, this would be a perfectly valid batch job. That's the power of the unified API.

### How Incremental Execution Actually Works

Under the hood, Spark doesn't re-run your entire query on every trigger. The planner converts your logical plan into an **incremental execution plan**. For a `groupBy().count()`:

1. On each trigger, Spark reads only **new data** from the source
2. It updates an internal **state store** (key → running count)
3. It emits results based on the output mode (new counts, changed counts, or everything)

The state store is the secret weapon — it lets Spark maintain running aggregations across micro-batches without re-reading all historical data.

> **Interview Insight:** *"Structured Streaming is NOT micro-batching DStreams. It's a completely different execution model built on Catalyst and Tungsten, using DataFrames — which means you get all the optimizations (predicate pushdown, whole-stage code generation) for free. The key abstraction is the unbounded table: your query is logically applied to the entire input history, but Spark executes it incrementally using a state store."*

---

## Screen 2: Output Modes — Append, Complete, Update

### The Three Output Modes Explained

When Spark finishes processing a micro-batch, it has an updated **Result Table**. The output mode controls **which part of that result table gets written to the sink.**

This is one of the most commonly confused topics in interviews, so let's get it crystal clear.

### Append Mode

```python
.writeStream.outputMode("append")
```

**What it does:** Only rows that are **newly added** to the result table since the last trigger are written to the sink. Once emitted, rows are never changed or retracted.

**When to use it:** When your result table only grows — no rows ever change. This means:
- Simple projections, filters, maps (no aggregation)
- Aggregations **with a watermark** — Spark waits until the watermark passes the window boundary, emits the final result, and guarantees it won't change

**The catch:** If you use aggregations without a watermark, Spark **cannot** use append mode because it can never be sure a window's result is "final" (late data might arrive and change it). You'll get an `AnalysisException`.

**Analogy:** Append mode is like a printer — once a row is printed, it's done. You can't go back and change it. So Spark only prints rows it's sure about.

### Complete Mode

```python
.writeStream.outputMode("complete")
```

**What it does:** The **entire result table** is written to the sink on every trigger. Every. Single. Time. Previous output is overwritten.

**When to use it:** Aggregations where you need the full picture. Think real-time dashboards showing total counts, running averages, etc.

**The catch:** Only works with aggregations. If your query is a simple `select/filter` with no `groupBy`, complete mode throws an error — there's no "result table" to replace, just a stream of rows.

**Performance warning:** As your result table grows, complete mode gets increasingly expensive because it re-writes everything. Use it for queries with bounded cardinality (e.g., count by country — at most ~200 rows) not unbounded cardinality (e.g., count by user_id — millions of rows).

**Analogy:** Complete mode is like a whiteboard — you erase it and redraw the entire picture every time something changes.

### Update Mode

```python
.writeStream.outputMode("update")
```

**What it does:** Only rows that **changed** in the result table since the last trigger are written. If a row was updated, the new version is emitted. If nothing changed, nothing is written.

**When to use it:** Most scenarios. It works with aggregations (emitting only changed counts) and without aggregations (behaves like append). It's the most flexible mode.

**The catch:** Not all sinks support update mode well. File sinks (Parquet, JSON) don't support it because files are immutable — you can't "update" a row in a Parquet file. Use it with databases, Delta Lake, or console sinks.

**Analogy:** Update mode is like a notification system — you only get notified about things that changed.

### Comparison Table

| Aspect | Append | Complete | Update |
|---|---|---|---|
| **What's Written** | New rows only | Entire result table | Only changed rows |
| **Works With Aggregations?** | Only with watermark | Yes (required, actually) | Yes |
| **Works Without Aggregations?** | Yes | No | Yes (behaves like append) |
| **Rows Can Change After Emit?** | No — final once emitted | Yes — full rewrite each time | Yes — updates overwrite |
| **State Cleanup** | After watermark passes | Never (keeps all state) | After watermark passes |
| **Best Sink Types** | Files, append-only logs | Dashboards, in-memory tables | Databases, Delta Lake |
| **Typical Use Case** | Log ingestion, event archival | Real-time dashboards, metrics | Incremental DB updates |

### Common Mistakes

1. **Using append mode with aggregations but no watermark** → `AnalysisException`. Fix: add `.withWatermark()` before the aggregation.
2. **Using complete mode with huge cardinality** → sink gets overwhelmed writing millions of rows every trigger. Fix: switch to update mode.
3. **Using update mode with a file sink** → `AnalysisException`. Files are append-only. Fix: use append mode or switch to Delta Lake (which supports updates via merge).

> **Interview Insight:** *"Start with `update` mode as your default mental model — it's the most flexible and efficient. Switch to `append` for write-once sinks like raw Parquet files or Kafka topics. Use `complete` only when the downstream system needs the full picture every time, and only when the result table has bounded cardinality."*

---

## Screen 3: Triggers — When Does Processing Happen?

### Understanding Triggers

A **trigger** defines **when** Spark kicks off the next micro-batch. It doesn't control *what* data is processed — that's determined by source offsets. It controls the *cadence* of processing.

### Default Trigger (No Trigger Specified)

```python
query = result.writeStream.outputMode("update").format("console").start()
# No .trigger() call — default behavior
```

Spark starts the next micro-batch **immediately** after the previous one finishes. There's no pause. This gives you the lowest latency possible under the micro-batch model (typically 100ms–few seconds depending on data volume and query complexity).

**When to use:** When you need near-real-time and have a dedicated, always-on cluster. This is the standard "streaming" mode.

### Fixed Interval Trigger

```python
.trigger(processingTime="30 seconds")
```

Spark waits **at least 30 seconds** between the start of consecutive micro-batches. If a batch takes 10 seconds, Spark idles for 20 seconds. If a batch takes 45 seconds (longer than the interval), Spark starts the next batch immediately — it never skips data.

**When to use:** When you want to control resource usage or match downstream SLAs. A 30-second trigger on a shared cluster is a polite neighbor.

### Trigger Once (The Batch-Streaming Bridge)

```python
.trigger(once=True)
```

Spark processes **all available data** in a single micro-batch, then **stops the query.** The checkpoint is saved, so next time you start the same query, it picks up where it left off.

**This is an underrated superpower.** Think about it:
- You get **exactly-once semantics** (checkpoint tracks offsets)
- You get **incremental processing** (only new data since last run)
- You can run it as a **scheduled batch job** (hourly cron, Airflow DAG)
- You pay for compute **only while processing** (no idle cluster costs)

**Example architecture:** An Airflow DAG triggers a Spark job hourly with `trigger(once=True)`. Each run reads new Kafka messages since the last run, processes them, writes to Delta Lake, saves checkpoint, and shuts down. You get streaming-grade reliability with batch-grade cost efficiency.

### Trigger Available Now (Spark 3.3+)

```python
.trigger(availableNow=True)
```

Like `once`, but instead of cramming everything into one massive micro-batch, Spark splits the available data into **multiple smaller micro-batches** based on source rate limits (e.g., `maxOffsetsPerTrigger` for Kafka). After all available data is processed, the query stops.

**Why it's better than `once` for large backlogs:** If you have 10 million unprocessed Kafka messages and use `trigger(once=True)`, Spark tries to read all 10M in one batch — potentially causing OOM. With `availableNow=True`, Spark respects your `maxOffsetsPerTrigger` setting and processes in manageable chunks.

```python
# Process backlog in chunks of 100K messages, then stop
(result.writeStream
    .trigger(availableNow=True)
    .option("maxOffsetsPerTrigger", 100000)
    .format("delta")
    .start()
)
```

### Continuous Trigger (Experimental)

```python
.trigger(continuous="1 second")  # checkpoint interval, NOT processing interval
```

**True continuous processing** — not micro-batching at all. Each task runs a continuous loop, processing records one at a time with ~1ms latency. The "1 second" parameter controls how often offsets are checkpointed.

**Limitations (as of Spark 3.5):**
- Only supports `map`-like operations (select, filter, map) — no aggregations, no joins
- Only supports Kafka and rate source
- Only supports append output mode
- Exactly-once guarantee requires idempotent sink

**When to use:** When you need sub-second latency and your query is simple (e.g., ETL with transformations but no aggregations). For most production workloads, the micro-batch model with default trigger is sufficient.

### Trigger Decision Tree

```
Need sub-second latency?
  └─ Yes → continuous trigger (if query is simple enough)
  └─ No → Need always-on near-real-time?
              └─ Yes → default trigger (no trigger specified)
              └─ No → Need cost savings / scheduled runs?
                          └─ Yes → Large backlog possible?
                                      └─ Yes → availableNow=True
                                      └─ No  → once=True
                          └─ No → Fixed interval trigger
```

> **Interview Insight:** *"Trigger.Once() is an underrated superpower. It lets you run a streaming pipeline as a scheduled batch job — say, hourly in Airflow — while keeping exactly-once semantics and checkpoint state across runs. You get streaming reliability with batch cost efficiency. For large backlogs, Spark 3.3's Trigger.AvailableNow is even better because it respects rate limits and processes in manageable chunks instead of one giant batch."*

---

## Screen 4: Watermarks & Late Data

### The Problem: Time Is Messy

In a perfect world, events arrive in order and on time. In reality:
- A mobile app buffers events while the user is on a subway (30 minutes late)
- A sensor's network connection drops for an hour, then reconnects and sends a burst
- A Kafka consumer lags behind and replays old data
- Timezone bugs cause events to appear with past timestamps

If you're doing **event-time** windowed aggregations (e.g., "count events per 5-minute window based on when they actually happened"), late data means a window you thought was complete can get new events at any time.

Without any mechanism to handle this, Spark would have to keep **every window's state forever**, because a late event for *any* past window could theoretically arrive. This means unbounded state growth → eventual OutOfMemoryError.

### Watermarks: The Solution

A **watermark** tells Spark: *"I accept that data can arrive late, but I guarantee I don't care about data more than X time units late. Discard it."*

```python
.withWatermark("event_time", "10 minutes")
```

This means: the **watermark** is defined as `max(event_time seen so far) - 10 minutes`. Any event with an `event_time` older than the current watermark is considered "too late" and is **silently dropped.**

**Analogy:** Imagine you're a professor collecting homework. You have a "10-minute grace period" policy. Once the latest student has submitted (let's say at 3:15 PM), you draw a line: anything earlier than 3:05 PM (3:15 minus 10 minutes) is no longer accepted. As more students submit, the line moves forward. It never moves backward.

### How Watermarks Enable State Cleanup

Without watermark:
```
Window [1:00-1:05] → state kept forever (might get late data)
Window [1:05-1:10] → state kept forever
Window [1:10-1:15] → state kept forever
... state grows and grows until OOM
```

With watermark of 10 minutes:
```
Current max event_time = 1:25
Watermark = 1:25 - 10min = 1:15
Window [1:00-1:05] → state CLEARED (window end 1:05 < watermark 1:15)
Window [1:05-1:10] → state CLEARED (window end 1:10 < watermark 1:15)
Window [1:10-1:15] → state CLEARED (window end 1:15 ≤ watermark 1:15)
Window [1:15-1:20] → state KEPT (still within watermark)
Window [1:20-1:25] → state KEPT (active window)
```

### Full PySpark Example

```python
from pyspark.sql.functions import window, col, from_json
from pyspark.sql.types import StructType, StringType, TimestampType

schema = StructType().add("user_id", StringType()).add("event_time", TimestampType())

events = (spark.readStream
    .format("kafka")
    .option("kafka.bootstrap.servers", "broker:9092")
    .option("subscribe", "user_events")
    .load()
    .select(from_json(col("value").cast("string"), schema).alias("data"))
    .select("data.user_id", "data.event_time")
)

# Watermark: tolerate up to 10 minutes of lateness
windowed = (events
    .withWatermark("event_time", "10 minutes")  # MUST come before groupBy
    .groupBy(
        window("event_time", "5 minutes"),  # 5-min tumbling windows
        "user_id"
    )
    .count()
)

# Append mode: results emitted ONLY after watermark passes window boundary
# This guarantees completeness — the count won't change after emission
query = (windowed.writeStream
    .outputMode("append")          # safe with watermark
    .format("parquet")
    .option("path", "/output/user_counts")
    .option("checkpointLocation", "/checkpoints/user_counts")
    .start()
)
```

### Watermark + Output Mode Interaction

This is the subtle part that trips up many candidates:

| Output Mode | With Watermark | Behavior |
|---|---|---|
| **Append** | Required for aggregations | Results emitted **only after watermark passes the window end**. Guarantees completeness — once emitted, the count won't change. Late data arriving after emission is dropped. |
| **Update** | Optional but recommended | Results emitted **on every trigger** as they update. Watermark used only for **state cleanup** — old windows are purged. Late data arriving before state cleanup still updates the count. |
| **Complete** | Ignored for output | Entire result table emitted every trigger regardless. Watermark is still used for state cleanup internally, but all windows (including old ones) are in the output. |

**Key subtlety for append mode:** There's a latency tradeoff. With a 10-minute watermark and 5-minute windows, a window's result won't be emitted until 10 minutes after the window closes (15 minutes after the window starts). If you need faster results, use update mode and accept that counts might change.

### Choosing the Right Watermark Duration

- **Too short** (e.g., 1 minute): You'll lose a lot of legitimate late data. Bad for accuracy.
- **Too long** (e.g., 24 hours): State grows huge because Spark keeps 24 hours of windows in memory. Bad for resources.
- **Just right:** Analyze your data's lateness distribution. If 99.9% of events arrive within 15 minutes, set a 15–20 minute watermark.

**Pro tip:** Many production systems use a watermark equal to **2–3x the expected maximum lateness** as a safety margin.

> **Interview Insight:** *"Watermarks solve two problems simultaneously: they prevent unbounded state growth (which would cause OOM), and they define when results are 'final' enough to emit in append mode. The formula is simple: watermark = max(event_time) - threshold. Any event older than the watermark is dropped. In production, always set a watermark for windowed aggregations — the question is not if, but how long."*

---

## Screen 5: Streaming Joins

### The Complexity Spectrum

Streaming joins come in three flavors, each with increasing complexity. Understanding the differences is a high-signal interview topic — it separates candidates who've actually built streaming pipelines from those who've only read the docs.

### Stream-Static Joins

The simplest case: join a streaming DataFrame with a regular (batch/static) DataFrame.

```python
# Static DataFrame — e.g., a dimension table
user_profiles = spark.read.parquet("/data/user_profiles")

# Streaming DataFrame — e.g., clickstream events
click_events = (spark.readStream
    .format("kafka")
    .option("kafka.bootstrap.servers", "broker:9092")
    .option("subscribe", "clicks")
    .load()
    .selectExpr("CAST(value AS STRING) AS user_id", "timestamp")
)

# Join: enrich streaming events with static user data
enriched = click_events.join(user_profiles, "user_id")  # inner join

enriched.writeStream.format("delta").start("/output/enriched_clicks")
```

**How it works:** On every micro-batch, Spark joins the new streaming rows with the **entire** static DataFrame. The static side is cached and re-used across micro-batches (but not re-read unless you restart the query).

**Important details:**
- **No watermark needed** — there's no state to manage (the static side is just a lookup table)
- **Supported join types:** Inner, left outer (stream on left), left semi, left anti
- **Right outer with stream on left is supported**, but right outer with stream on right is NOT (Spark can't guarantee completeness)
- **Stale data risk:** If the static table is updated (e.g., new user profiles added), the streaming query won't see the updates until it's restarted. For frequently changing dimensions, consider using a stream-stream join with a compacted Kafka topic instead.

### Stream-Stream Joins

This is where it gets interesting. You're joining two live streams together — e.g., matching ad impressions with ad clicks.

```python
# Stream 1: Ad impressions
impressions = (spark.readStream
    .format("kafka")
    .option("subscribe", "impressions")
    .load()
    .selectExpr(
        "CAST(value AS STRING) AS ad_id",
        "timestamp AS impression_time"
    )
    .withWatermark("impression_time", "2 hours")  # watermark on BOTH sides
)

# Stream 2: Ad clicks
clicks = (spark.readStream
    .format("kafka")
    .option("subscribe", "clicks")
    .load()
    .selectExpr(
        "CAST(value AS STRING) AS ad_id",
        "timestamp AS click_time"
    )
    .withWatermark("click_time", "3 hours")  # can be different thresholds
)

# Stream-Stream inner join with time-range condition
matched = impressions.join(
    clicks,
    expr("""
        impressions.ad_id = clicks.ad_id AND
        click_time >= impression_time AND
        click_time <= impression_time + INTERVAL 1 HOUR
    """)
)

matched.writeStream.outputMode("append").format("delta").start("/output/matched_ads")
```

### Stream-Stream Join Internals: The State Buffer

Here's what happens under the hood:

1. On each micro-batch, new rows from **both** streams arrive
2. Spark stores new rows from each stream in a **state store** (one per side of the join)
3. Each new row from Stream A is matched against **all buffered rows** from Stream B, and vice versa
4. Matched pairs are emitted as join results
5. Old rows are purged from the state store based on watermarks and time-range conditions

**Without a watermark or time-range condition:** Spark must buffer **every row from both streams forever** because a match could theoretically come at any time. This means unbounded state growth → OOM.

**With a watermark + time-range condition:** Spark can reason about when a row can no longer possibly match anything, and purge it from state.

### Join Type Constraints

| Join Type | Watermark Required? | Why |
|---|---|---|
| **Inner** | Optional (but strongly recommended) | Without watermark, state grows forever. With watermark + time range, old unmatched rows are purged. |
| **Left Outer** | Required on the **right** stream | Spark needs to know when a left row will NEVER find a match on the right → emit (left_row, NULL). The watermark on the right stream defines this deadline. |
| **Right Outer** | Required on the **left** stream | Mirror of left outer — needs to know when right rows will never match. |
| **Full Outer** | Required on **both** streams | Both sides need deadlines for emitting unmatched rows with NULLs. |

### Why Outer Joins Need Watermarks

Think about a left outer join: for every row on the left side, you need to output either `(left, right)` if there's a match, or `(left, NULL)` if there isn't. But how does Spark know there will *never* be a match? In a streaming context, a matching right-side row could arrive at any moment in the future.

The watermark solves this: once the watermark on the right stream advances past the time range where a match is possible, Spark knows the match will never happen and emits `(left, NULL)`.

**Without a watermark:** Spark would have to wait **forever** before emitting the null, which means it can never emit the row → outer joins become impossible.

### Stream-Stream Join Best Practices

1. **Always add watermarks on both streams** — even for inner joins (for state cleanup)
2. **Always add a time-range condition** — it dramatically reduces state size by limiting how far back Spark looks for matches
3. **Monitor state store size** — use `query.lastProgress` or Spark UI to track state rows
4. **Consider asymmetric watermarks** — if clicks lag behind impressions by more than impressions lag behind clicks, use different watermark thresholds (as shown in the example above)

> **Interview Insight:** *"Stream-stream joins are the hardest Structured Streaming topic. The key insight is about state management: inner joins work without watermarks but state grows forever. Outer joins REQUIRE watermarks because Spark needs to know when to emit NULL for unmatched rows — it's the deadline for 'giving up' on finding a match. Always pair watermarks with a time-range condition in the join predicate to bound state size."*

---

## Screen 6: Exactly-Once & Fault Tolerance

### The Exactly-Once Triangle

End-to-end exactly-once processing in Structured Streaming requires **three components working together.** If any one is missing, you degrade to at-least-once (or worse).

```
        ┌──────────────────┐
        │   Replayable     │
        │   Source         │  ← Can re-read data from a specific offset
        └────────┬─────────┘
                 │
        ┌────────▼─────────┐
        │   Checkpointing  │  ← Tracks what's been processed
        └────────┬─────────┘
                 │
        ┌────────▼─────────┐
        │   Idempotent     │
        │   Sink           │  ← Handles duplicate writes gracefully
        └──────────────────┘
```

**1. Replayable Source:** The source must support reading from a specific offset. Kafka does this natively (consumer offsets). File sources do this (file list = offsets). A random TCP socket does NOT — if data is lost, it's gone.

**2. Checkpointing:** Spark records processing progress in a checkpoint directory. Before starting a micro-batch, Spark writes the offset range to the checkpoint. On failure and restart, Spark reads the checkpoint to know exactly where to resume.

**3. Idempotent Sink:** Since Spark replays data on failure, the sink may receive the same records twice. The sink must handle this gracefully — either by being naturally idempotent (overwriting the same partition in a Parquet file), supporting transactions (Delta Lake MERGE), or using application-level deduplication.

### What's Inside a Checkpoint Directory

```
/checkpoints/my_query/
├── offsets/          # Offset log: planned offset ranges per micro-batch
│   ├── 0             # Batch 0: Kafka offsets {topic1: {0: 100, 1: 200}}
│   ├── 1             # Batch 1: Kafka offsets {topic1: {0: 250, 1: 350}}
│   └── 2
├── commits/          # Commit log: which batches completed successfully
│   ├── 0
│   └── 1             # Batch 2 not here = was in progress when failure happened
├── state/            # State store: running aggregation data (if stateful query)
│   └── 0/
│       ├── 1.delta
│       └── 2.delta
├── metadata          # Query metadata (query ID, etc.)
└── sources/          # Source-specific metadata
```

**Recovery flow on failure:**
1. Spark reads `commits/` — sees batch 1 was the last committed
2. Spark reads `offsets/2` — sees batch 2 was planned but not committed
3. Spark **replays batch 2** from the source using the recorded offsets
4. Spark re-processes and writes to the sink
5. If the sink received partial writes from the failed attempt, the idempotent property prevents duplicates

### State Store Backends

For stateful queries (aggregations, stream-stream joins, deduplication), Spark maintains a **state store**. There are two backends:

**HDFS-Based State Store (Default)**
- Stores all state in JVM memory (on-heap `HashMap`)
- Periodically snapshots to HDFS/S3 for durability
- **Problem:** All state must fit in executor memory. At ~100M keys, you hit GC issues and OOM.
- **Good for:** Small state (< 1M keys), simple aggregations

**RocksDB State Store (Spark 3.2+)**
```python
spark.conf.set(
    "spark.sql.streaming.stateStore.providerClass",
    "org.apache.spark.sql.execution.streaming.state.RocksDBStateStoreProvider"
)
```
- Uses RocksDB (embedded key-value store) to spill state to local disk
- Only keeps hot data in memory, rest on SSD
- Can handle **billions of keys** with bounded memory usage
- **Trade-off:** Slightly higher latency per state operation (disk I/O)
- **Good for:** Large state (> 1M keys), stream-stream joins with wide time ranges, production workloads

**Interview-ready comparison:**

| Aspect | HDFS State Store | RocksDB State Store |
|---|---|---|
| Storage | All in JVM heap | Memory + local disk (SSD) |
| Max State Size | Limited by executor memory | Limited by disk space |
| Latency | Faster (in-memory) | Slightly slower (disk I/O) |
| GC Pressure | High with large state | Low (off-heap via RocksDB) |
| Checkpointing | Full snapshot each time | Incremental (only changed keys) |
| Recommended For | < 1M state keys | > 1M state keys, production |

### Schema Evolution and Checkpoint Compatibility

One of the trickiest operational issues: **what changes can you make to a streaming query without breaking the checkpoint?**

**Compatible changes (safe):**
- Adding new columns to the output (projection changes)
- Adding or relaxing filters
- Changing the trigger interval
- Changing sink-specific options (e.g., Kafka topic name)

**Incompatible changes (require new checkpoint):**
- Changing the grouping key in an aggregation
- Changing the window duration
- Changing the watermark threshold
- Adding or removing a stateful operation (adding a new `groupBy`)
- Changing the number or type of stateful operations

**When you must start fresh:** Delete the old checkpoint directory and reprocess from the beginning. For Kafka sources, set `startingOffsets` to `earliest` to replay all data.

**Pro tip:** Version your checkpoint directories (e.g., `/checkpoints/my_query/v2/`) so you can roll back if a new query version has bugs.

> **Interview Insight:** *"Exactly-once in Structured Streaming requires three things working together: a replayable source (like Kafka that supports offset-based replay), an idempotent sink (like Delta Lake with MERGE or file overwrite semantics), and checkpointing (which stores offset ranges and state). If any one is missing, you degrade to at-least-once. The formula is: replayable source + idempotent sink + checkpointing = exactly-once. This is the three-part answer interviewers want to hear."*

---

## Screen 7: Quiz — Test Your Structured Streaming Knowledge

### Question 1: Output Modes

**What's the difference between append, complete, and update output modes?**

- A) They all write the same data, just to different sink types
- B) Append = new rows only; Complete = entire result table; Update = only changed rows
- C) Append = aggregated results; Complete = raw events; Update = deleted rows
- D) They're aliases for the same behavior

✅ **Correct Answer: B**

**Explanation:** Append writes only newly added rows (once, final). Complete rewrites the entire result table every trigger (only works with aggregations). Update writes only rows that changed since the last trigger — most flexible. Key gotcha: append mode with aggregations requires a watermark because Spark needs to know when a window's result is final.

---

### Question 2: OOM with Streaming Aggregation

**You have a streaming aggregation that runs out of memory after a few hours. What's the most likely cause and fix?**

- A) The Kafka topic has too many partitions — reduce them
- B) The output mode is wrong — switch to append
- C) A watermark is missing — without it, Spark keeps all windowed state forever
- D) The executor memory is too low — just increase `spark.executor.memory`

✅ **Correct Answer: C**

**Explanation:** Without a watermark, Spark has no way to know when a window is "done," so it keeps state for every window ever seen. State grows linearly with time → OOM. Adding `.withWatermark("event_time", "10 minutes")` tells Spark to discard state for windows older than the watermark. Increasing memory (D) only delays the inevitable — state grows forever without a watermark.

---

### Question 3: Trigger.Once for Scheduled Batch

**You want to run a streaming pipeline once per hour as a scheduled Airflow job but maintain exactly-once semantics between runs. What trigger do you use?**

- A) `.trigger(processingTime="1 hour")` — it pauses for an hour between batches
- B) `.trigger(once=True)` or `.trigger(availableNow=True)` — process all available data then stop
- C) `.trigger(continuous="1 hour")` — runs continuously with 1-hour checkpoints
- D) No trigger needed — just kill the job and restart it hourly

✅ **Correct Answer: B**

**Explanation:** `Trigger.Once()` processes all available data in a single batch and stops, saving checkpoint state. `Trigger.AvailableNow()` (Spark 3.3+) does the same but splits into multiple batches for better memory management. Both maintain checkpoint state between runs, so the next Airflow-triggered run picks up exactly where the last one left off. Option A would keep the Spark application running for the full hour, wasting cluster resources. Option D would lose checkpoint state and reprocess data.

---

### Question 4: Stream-Stream Outer Join

**Can you perform an outer join between two streams without a watermark?**

- A) Yes — Spark buffers everything and eventually matches
- B) No — outer joins require watermarks so Spark knows when to emit NULL for unmatched rows
- C) Yes, but only for full outer joins, not left or right
- D) No — stream-stream joins of any type require watermarks

✅ **Correct Answer: B**

**Explanation:** For an outer join, Spark needs to decide when an unmatched row will *never* find a partner — that's when it emits `(row, NULL)`. Without a watermark, Spark can't make this determination (a matching row could theoretically arrive at any future time), so it can never emit the null row. Inner joins work without watermarks (D is wrong) because there's no NULL to emit — unmatched rows are simply ignored. But even inner joins *should* have watermarks to prevent unbounded state growth.

---

### Question 5: The Exactly-Once Formula

**What three components are needed for end-to-end exactly-once processing in Structured Streaming?**

- A) Kafka, Spark, and HDFS
- B) Watermarks, triggers, and output modes
- C) Replayable source, idempotent sink, and checkpointing
- D) State store, RocksDB, and Delta Lake

✅ **Correct Answer: C**

**Explanation:** The three pillars: (1) **Replayable source** — can re-read data from a specific offset (Kafka, file sources). (2) **Idempotent sink** — handles duplicate writes gracefully (Delta Lake, file overwrite, dedup logic). (3) **Checkpointing** — records offsets and state so Spark knows where to resume after failure. Remove any one pillar and you fall to at-least-once. Kafka/HDFS (A) are specific technologies, not the abstract requirements. RocksDB/Delta (D) are implementation choices, not requirements.

---

### Question 6: Failure Recovery

**A streaming query fails mid-batch and restarts. How does Spark avoid processing duplicate data and producing duplicate output?**

- A) Spark contacts Kafka to decommit the consumed offsets
- B) The checkpoint stores committed offsets; on restart, Spark replays from the last committed offset, and the idempotent sink handles any duplicate writes from the failed partial batch
- C) Spark uses distributed locks to prevent any re-processing
- D) Duplicate data is unavoidable — Spark provides at-most-once guarantees

✅ **Correct Answer: B**

**Explanation:** Here's the recovery sequence: (1) Spark reads the checkpoint's `commits/` directory to find the last *completed* batch. (2) It reads the `offsets/` directory to find the planned-but-not-committed batch. (3) It replays that batch from the source using the recorded offset range. (4) The sink may have received partial writes from the failed attempt — the idempotent property ensures these don't create duplicates (e.g., Delta Lake uses transaction IDs, file sinks overwrite the same output partition). Spark does NOT interact with Kafka's consumer group offsets for this — it manages offsets independently in its own checkpoint.
