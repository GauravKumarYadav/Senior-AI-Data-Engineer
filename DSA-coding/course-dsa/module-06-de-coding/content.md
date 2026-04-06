# Module 6: Data Engineering Coding

> **Goal:** Master the bread-and-butter coding patterns that appear in data engineering interviews. Every problem is solved **twice** — once in Pandas and once in PySpark — so you build muscle memory in both ecosystems.

---

## Why This Module Matters

Data engineering interviews rarely ask you to invert a binary tree. Instead, they hand you messy CSVs, event streams, and log files and say *"clean this up."* The patterns below cover **90 %+** of the coding rounds you'll face at companies like Walmart, Meta, Airbnb, and Spotify. Each section follows the same rhythm:

1. **Problem statement** — what an interviewer would say.
2. **Pandas solution** — idiomatic, vectorized where possible.
3. **PySpark solution** — using the DataFrame API (not RDDs).
4. **Interview tip** — the one-liner that makes the interviewer nod.

### Pandas vs. PySpark: When to Use Which

Before we dive in, let's establish when each tool shines:

- **Pandas** — Single-machine, in-memory. Ideal when your data fits in RAM (up to ~10 GB with careful memory management). Faster iteration, richer API, easier debugging. Use for prototyping, small-to-medium datasets, and local data exploration.
- **PySpark** — Distributed computing across a cluster. Necessary when data exceeds single-machine memory or when you need to process terabytes. Lazy evaluation means transformations are optimized before execution. Use for production pipelines, large-scale ETL, and anything running on Databricks/EMR/Dataproc.

In interviews, you'll often be asked to solve a problem in *one* framework and then the interviewer will say *"Now how would you do this in PySpark?"* — or vice versa. Mastering both translations is your competitive edge.

---

## 1. Parse & Aggregate Log Files

### Problem

You receive a 10 GB web-server log file. Each line looks like:

```
2025-03-15 08:12:01 ERROR [PaymentService] Timeout connecting to gateway
2025-03-15 08:12:03 WARN  [UserService] Slow query detected (1200ms)
2025-03-15 08:12:05 ERROR [PaymentService] Timeout connecting to gateway
```

**Tasks:**

1. Extract the log level and service name from every line.
2. Count occurrences of each `(level, service)` pair.
3. Return the top 5 most frequent errors.

### Pandas

```python
import pandas as pd
import re

# 1 — Read & parse
pattern = r"(?P<ts>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2})\s+(?P<level>\w+)\s+\[(?P<service>\w+)\]\s+(?P<msg>.*)"

with open("server.log") as f:
    records = [m.groupdict() for line in f if (m := re.match(pattern, line))]

df = pd.DataFrame(records)
df["ts"] = pd.to_datetime(df["ts"])

# 2 — Aggregate
counts = (
    df.groupby(["level", "service"])
      .size()
      .reset_index(name="count")
      .sort_values("count", ascending=False)
)

# 3 — Top 5 errors
top_errors = counts[counts["level"] == "ERROR"].head(5)
print(top_errors)
```

### PySpark

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import regexp_extract, col, desc

spark = SparkSession.builder.appName("logs").getOrCreate()

raw = spark.read.text("server.log")

pattern = r"(\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2})\s+(\w+)\s+\[(\w+)\]\s+(.*)"

parsed = (
    raw.select(
        regexp_extract("value", pattern, 1).alias("ts"),
        regexp_extract("value", pattern, 2).alias("level"),
        regexp_extract("value", pattern, 3).alias("service"),
        regexp_extract("value", pattern, 4).alias("msg"),
    )
    .filter(col("level") != "")          # drop non-matching lines
    .withColumn("ts", col("ts").cast("timestamp"))
)

# Aggregate & top errors
top_errors = (
    parsed.filter(col("level") == "ERROR")
    .groupBy("level", "service")
    .count()
    .orderBy(desc("count"))
    .limit(5)
)
top_errors.show()
```

> **Interview Tip:** Mention that for a 10 GB file you would **never** `pd.read_csv()` the whole thing — you'd stream it line-by-line or use PySpark / Dask. Saying this unprompted signals production awareness.

### Key Concepts Demonstrated

- **Regex named groups** (`?P<name>`) in Python make parsed data self-documenting.
- **`regexp_extract`** in PySpark uses positional groups (1, 2, 3...) — no named groups.
- **Filtering before aggregation** (`filter(col("level") == "ERROR")` before `groupBy`) reduces shuffle volume in Spark — always push filters as early as possible in the DAG.
- For truly massive files, consider using PySpark's `spark.read.text()` which distributes the file across executors, whereas Python's `open()` is single-threaded.

---

## 2. Deduplicate Events

### Problem

An event stream has duplicates because of at-least-once delivery. Each event has a `key` and an `event_time`. Keep only the **latest** event per key.

| key | event_time          | value |
|-----|---------------------|-------|
| A   | 2025-03-15 08:00:00 | 10    |
| A   | 2025-03-15 09:00:00 | 20    |
| B   | 2025-03-15 08:30:00 | 30    |
| B   | 2025-03-15 07:00:00 | 40    |

**Expected:** key A → value 20, key B → value 30.

### Pandas

```python
# Approach 1: sort + drop_duplicates (clean & fast)
deduped = (
    df.sort_values("event_time", ascending=False)
      .drop_duplicates(subset=["key"], keep="first")
      .sort_values("key")
)

# Approach 2: explicit row_number equivalent via groupby + idxmax
idx = df.groupby("key")["event_time"].idxmax()
deduped = df.loc[idx]
```

### PySpark

```python
from pyspark.sql.window import Window
from pyspark.sql.functions import row_number, col, desc

window = Window.partitionBy("key").orderBy(desc("event_time"))

deduped = (
    df.withColumn("rn", row_number().over(window))
      .filter(col("rn") == 1)
      .drop("rn")
)
```

> **Interview Tip:** Always clarify tie-breaking: *"If two events share the same timestamp, which wins?"* This shows you think about edge cases before coding.

### Why Two Approaches in Pandas?

Approach 1 (`sort_values` + `drop_duplicates`) is concise and fast — it leverages Pandas' internal sort and dedup optimizations. Approach 2 (`idxmax`) is explicit about *which* row wins and is easier to extend (e.g., break ties by a secondary column). In an interview, show Approach 1 for speed, then mention Approach 2 if tie-breaking logic gets complex.

In PySpark, `ROW_NUMBER()` is the universal dedup pattern. It mirrors SQL exactly, which makes it easy to explain and reason about. Note that `dropDuplicates()` in PySpark does **not** let you control which row is kept — it's non-deterministic. Always prefer the window function approach when ordering matters.

---

## 3. Sessionization

### Problem

Given user click events, group them into **sessions**. A new session starts when the gap between consecutive events for the same user exceeds **30 minutes**.

| user_id | event_time          |
|---------|---------------------|
| U1      | 2025-03-15 08:00:00 |
| U1      | 2025-03-15 08:10:00 |
| U1      | 2025-03-15 09:00:00 |
| U1      | 2025-03-15 09:05:00 |

**Expected:** Events at 08:00 and 08:10 → session 1; events at 09:00 and 09:05 → session 2.

### Pandas

```python
df = df.sort_values(["user_id", "event_time"])

# Time gap from previous event (per user)
df["prev_time"] = df.groupby("user_id")["event_time"].shift(1)
df["gap_min"] = (df["event_time"] - df["prev_time"]).dt.total_seconds() / 60

# New session flag: first event or gap > 30 min
df["new_session"] = (df["gap_min"].isna()) | (df["gap_min"] > 30)

# Cumulative sum of flags → session_id per user
df["session_id"] = df.groupby("user_id")["new_session"].cumsum().astype(int)
```

### PySpark

```python
from pyspark.sql.functions import lag, col, sum as spark_sum, when, unix_timestamp
from pyspark.sql.window import Window

user_window = Window.partitionBy("user_id").orderBy("event_time")

sessionized = (
    df.withColumn("prev_time", lag("event_time").over(user_window))
      .withColumn(
          "gap_min",
          (unix_timestamp("event_time") - unix_timestamp("prev_time")) / 60,
      )
      .withColumn(
          "new_session",
          when(col("gap_min").isNull() | (col("gap_min") > 30), 1).otherwise(0),
      )
      .withColumn(
          "session_id",
          spark_sum("new_session").over(user_window),
      )
)
```

> **Interview Tip:** This is the **#1 most-asked** data engineering coding question. Practice it until you can write it from memory. The pattern is: *lag → gap → flag → cumsum*.

### Common Variations

- **Variable session timeout:** Instead of a fixed 30 minutes, the threshold might depend on the user's historical behavior (e.g., use the user's median inter-event gap × 2). You'd compute the threshold in a separate CTE/DataFrame and join it in.
- **Session metrics:** After sessionizing, compute session duration, event count per session, and bounce rate (sessions with only 1 event). These are simple groupby aggregations on the session_id.
- **Cross-device sessions:** If a user can be on multiple devices, you may need to sessionize per (user_id, device_id) pair.

---

## 4. Mini ETL Pipeline

### Problem

Build a small ETL pipeline:

1. **Read** a CSV of sales transactions.
2. **Clean:** drop nulls in critical columns, cast types, normalize strings.
3. **Transform:** add a `revenue` column (`qty × price`).
4. **Write** to Parquet with snappy compression.

### Pandas

```python
import pandas as pd

# ---------- EXTRACT ----------
dtype_spec = {"order_id": str, "product": str, "qty": "Int64", "price": float}
df = pd.read_csv("sales.csv", dtype=dtype_spec, parse_dates=["order_date"])

# ---------- CLEAN ----------
critical = ["order_id", "product", "qty", "price"]
df = df.dropna(subset=critical)

df["product"] = df["product"].str.strip().str.lower()
df["qty"] = df["qty"].astype(int)
df["price"] = df["price"].round(2)

# ---------- TRANSFORM ----------
df["revenue"] = df["qty"] * df["price"]

# ---------- LOAD ----------
df.to_parquet("sales_clean.parquet", engine="pyarrow", compression="snappy", index=False)
```

### PySpark

```python
from pyspark.sql.types import StructType, StructField, StringType, IntegerType, DoubleType, DateType
from pyspark.sql.functions import col, trim, lower, round as spark_round

# ---------- EXTRACT (with explicit schema) ----------
schema = StructType([
    StructField("order_id", StringType(), False),
    StructField("product", StringType(), False),
    StructField("qty", IntegerType(), False),
    StructField("price", DoubleType(), False),
    StructField("order_date", DateType(), True),
])

df = (
    spark.read
    .option("header", True)
    .schema(schema)                          # enforce, don't infer
    .csv("sales.csv")
)

# ---------- CLEAN ----------
df = (
    df.dropna(subset=["order_id", "product", "qty", "price"])
      .withColumn("product", lower(trim(col("product"))))
      .withColumn("price", spark_round(col("price"), 2))
)

# ---------- TRANSFORM ----------
df = df.withColumn("revenue", col("qty") * col("price"))

# ---------- LOAD ----------
df.write.mode("overwrite").parquet("sales_clean.parquet", compression="snappy")
```

> **Interview Tip:** In PySpark, **always** provide an explicit schema instead of using `inferSchema=True`. Inference triggers an extra pass over the data and can guess wrong types. Mentioning this wins you bonus points.

### Production Considerations

In a real ETL pipeline, this code would be wrapped in additional safeguards:

1. **Data quality checks** — After reading, assert row counts, null percentages, and value ranges. Libraries like Great Expectations or Deequ (for Spark) automate this.
2. **Idempotency** — Use `mode("overwrite")` with partition-based writes so re-runs don't create duplicate data. For append-mode pipelines, add a dedup step.
3. **Partitioning** — Write Parquet partitioned by date (`partitionBy("order_date")`) for efficient downstream queries.
4. **Logging & monitoring** — Log input/output row counts, write durations, and schema snapshots. Alert on anomalies (e.g., row count drops > 20 %).
5. **Schema evolution** — Use `mergeSchema` option in PySpark when the source schema might change over time.

---

## 5. Data Reconciliation

### Problem

You migrated a table from Oracle to BigQuery. Given `source_df` and `target_df` with identical schemas, find:

1. Rows in source but **missing** from target.
2. Rows in target but **missing** from source.
3. Rows present in both but with **different values**.

### Pandas

```python
key = "order_id"

# Missing from target (anti-join)
missing_in_target = source_df[~source_df[key].isin(target_df[key])]

# Missing from source
missing_in_source = target_df[~target_df[key].isin(source_df[key])]

# Value mismatches — merge on key, compare columns
merged = source_df.merge(target_df, on=key, suffixes=("_src", "_tgt"))

value_cols = [c for c in source_df.columns if c != key]
for c in value_cols:
    merged[f"{c}_match"] = merged[f"{c}_src"] == merged[f"{c}_tgt"]

mismatches = merged[~merged[[f"{c}_match" for c in value_cols]].all(axis=1)]
```

### PySpark

```python
from pyspark.sql.functions import col

key = "order_id"

# Anti-join: rows in source missing from target
missing_in_target = source_df.join(target_df, on=key, how="left_anti")

# Anti-join: rows in target missing from source
missing_in_source = target_df.join(source_df, on=key, how="left_anti")

# Value mismatches
value_cols = [c for c in source_df.columns if c != key]

joined = source_df.alias("s").join(target_df.alias("t"), on=key, how="inner")

mismatch_cond = None
for c in value_cols:
    cond = col(f"s.{c}") != col(f"t.{c}")
    mismatch_cond = cond if mismatch_cond is None else (mismatch_cond | cond)

mismatches = joined.filter(mismatch_cond)
```

> **Interview Tip:** Use the term **"anti-join"** — it tells the interviewer you know relational algebra. In PySpark, `left_anti` is purpose-built for this. In Pandas you simulate it with `isin()` negation or `merge(how='left', indicator=True)`.

### The `indicator=True` Trick in Pandas

An alternative Pandas anti-join approach uses the merge indicator:

```python
merged = source_df.merge(target_df, on=key, how="left", indicator=True)
missing_in_target = merged[merged["_merge"] == "left_only"].drop(columns=["_merge"])
```

The `_merge` column contains `"both"`, `"left_only"`, or `"right_only"`, giving you fine-grained control. This is especially useful when you need all three reconciliation buckets (missing left, missing right, matched) from a single merge operation.

### Handling NULL Comparisons

A subtle bug in the value-mismatch logic: `NULL != NULL` evaluates to `NULL` (not `TRUE`) in both Python and SQL. To correctly detect mismatches when values can be null, use `coalesce()` in PySpark or `fillna()` in Pandas before comparing, or use null-safe equality: `col("s.x").eqNullSafe(col("t.x"))`.

---

## 6. Stream Processing Mock

### Problem

Simulate a stream of events. For each incoming event, maintain:

- A **running count** per category.
- A **running average** of the `amount` field per category.

Print the updated state after every event.

### Python (Generators / itertools)

```python
from collections import defaultdict
from typing import Generator, NamedTuple

class Event(NamedTuple):
    category: str
    amount: float

def event_stream() -> Generator[Event, None, None]:
    """Simulates an event source."""
    events = [
        Event("electronics", 299.99),
        Event("books", 12.50),
        Event("electronics", 149.00),
        Event("books", 25.00),
        Event("electronics", 499.99),
    ]
    yield from events

def process_stream(stream: Generator[Event, None, None]) -> None:
    counts: dict[str, int] = defaultdict(int)
    totals: dict[str, float] = defaultdict(float)

    for event in stream:
        counts[event.category] += 1
        totals[event.category] += event.amount
        avg = totals[event.category] / counts[event.category]
        print(
            f"[{event.category}] count={counts[event.category]}  "
            f"running_avg={avg:.2f}"
        )

process_stream(event_stream())
```

### PySpark (Structured Streaming Simulation)

```python
from pyspark.sql.functions import col, avg as spark_avg, count as spark_count
from pyspark.sql.window import Window

# Assume df has columns: category, amount, event_time (already ordered)
running_window = (
    Window.partitionBy("category")
          .orderBy("event_time")
          .rowsBetween(Window.unboundedPreceding, Window.currentRow)
)

result = (
    df.withColumn("running_count", spark_count("*").over(running_window))
      .withColumn("running_avg", spark_avg("amount").over(running_window))
)
result.show()
```

> **Interview Tip:** If they say "streaming," ask: *"Do you want me to simulate with batch window functions, or write an actual Structured Streaming job?"* This clarification shows you understand the difference between true streaming and batch simulation.

### Why Generators Matter

The Python generator approach above demonstrates a fundamental concept: **lazy evaluation with constant memory**. The `event_stream()` generator yields one event at a time — it could read from a Kafka topic, a socket, or a file of arbitrary size without loading everything into memory. The `defaultdict`-based state machine processes each event in O(1) time and O(k) space where k is the number of unique categories.

This pattern is the building block of frameworks like Faust (Python stream processing), Apache Beam's DoFn, and even PySpark Structured Streaming's `foreachBatch`. Understanding it at the raw Python level gives you flexibility when you can't use a full framework.

---

## 7. GroupBy Aggregations

### Problem

Given a sales table, compute per-product:

- Total revenue (`sum`)
- Average order value (`mean`)
- Number of orders (`count`)
- Largest single order (`max`)
- A custom metric: coefficient of variation (`std / mean`)

### Pandas — Named Aggregations

```python
agg_df = (
    df.groupby("product")
      .agg(
          total_revenue=("revenue", "sum"),
          avg_order=("revenue", "mean"),
          num_orders=("revenue", "count"),
          max_order=("revenue", "max"),
          std_revenue=("revenue", "std"),
      )
)
agg_df["coeff_variation"] = agg_df["std_revenue"] / agg_df["avg_order"]
agg_df = agg_df.drop(columns=["std_revenue"])
```

### PySpark

```python
from pyspark.sql.functions import sum as _sum, avg as _avg, count as _count, max as _max, stddev

agg_df = (
    df.groupBy("product")
      .agg(
          _sum("revenue").alias("total_revenue"),
          _avg("revenue").alias("avg_order"),
          _count("revenue").alias("num_orders"),
          _max("revenue").alias("max_order"),
          stddev("revenue").alias("std_revenue"),
      )
      .withColumn("coeff_variation", col("std_revenue") / col("avg_order"))
      .drop("std_revenue")
)
```

> **Interview Tip:** In Pandas, prefer **named aggregations** (`col=("src", "func")`) over the old `agg({"col": "func"})` dict style — it avoids MultiIndex headaches and reads cleaner.

### Custom Aggregations

Sometimes you need aggregations that aren't built-in. In Pandas, pass a lambda or a named function:

```python
# Interquartile range as a custom agg
def iqr(series: pd.Series) -> float:
    return series.quantile(0.75) - series.quantile(0.25)

df.groupby("product").agg(revenue_iqr=("revenue", iqr))
```

In PySpark, custom aggregations require a UDF (user-defined function) or `Aggregator` via `pandas_udf`:

```python
from pyspark.sql.functions import pandas_udf
from pyspark.sql.types import DoubleType

@pandas_udf(DoubleType())
def iqr_udf(series: pd.Series) -> float:
    return series.quantile(0.75) - series.quantile(0.25)

df.groupBy("product").agg(iqr_udf("revenue").alias("revenue_iqr"))
```

**Performance note:** UDFs in PySpark involve serialization between JVM and Python — they're 2–10× slower than built-in functions. Use them only when no built-in alternative exists.

---

## 8. Pivot / Unpivot

### Problem

**Pivot:** Given long-format sales data (`product, quarter, revenue`), create a wide table with one column per quarter.

**Unpivot:** Convert the wide table back to long format.

### Pandas

```python
# ----- Pivot (long → wide) -----
wide = df.pivot_table(
    index="product",
    columns="quarter",
    values="revenue",
    aggfunc="sum",
    fill_value=0,
)
wide.columns = [f"Q{q}" for q in wide.columns]   # flatten MultiIndex
wide = wide.reset_index()

# ----- Unpivot (wide → long) -----
long = wide.melt(
    id_vars=["product"],
    value_vars=["Q1", "Q2", "Q3", "Q4"],
    var_name="quarter",
    value_name="revenue",
)
```

### PySpark

```python
from pyspark.sql.functions import expr

# ----- Pivot (long → wide) -----
wide = (
    df.groupBy("product")
      .pivot("quarter", values=["Q1", "Q2", "Q3", "Q4"])   # specify values!
      .agg(_sum("revenue"))
      .fillna(0)
)

# ----- Unpivot (wide → long) using stack() -----
unpivot_expr = """
    stack(4,
        'Q1', Q1,
        'Q2', Q2,
        'Q3', Q3,
        'Q4', Q4
    ) as (quarter, revenue)
"""
long = wide.select("product", expr(unpivot_expr)).filter(col("revenue") != 0)
```

> **Interview Tip:** In PySpark, **always pass the list of pivot values** explicitly. Without it, Spark runs an extra aggregation job to discover distinct values — a performance killer on large datasets.

### When to Pivot vs. When to Stay Long

Long format is better for storage, joins, and most analytical queries. Wide format is better for human readability, ML feature matrices, and specific reporting layouts. In interviews, if you're asked to "reshape" data, clarify which direction and why — it shows you think about data modeling, not just syntax.

---

## 9. Time-Series Resampling

### Problem

Given minute-level sensor data, produce:

1. **Hourly averages.**
2. **Forward-filled** missing hours.
3. A **7-period rolling average** on the hourly data.

### Pandas

```python
df = df.set_index("timestamp")

# 1 — Resample to hourly
hourly = df["value"].resample("1h").mean()

# 2 — Forward-fill gaps
hourly = hourly.ffill()

# 3 — Rolling average
hourly_df = hourly.to_frame(name="avg_value")
hourly_df["rolling_7"] = hourly_df["avg_value"].rolling(window=7).mean()
```

### PySpark

```python
from pyspark.sql.functions import window, avg as _avg, col, last
from pyspark.sql.window import Window

# 1 — Resample (tumbling window)
hourly = (
    df.groupBy(window("timestamp", "1 hour"))
      .agg(_avg("value").alias("avg_value"))
      .withColumn("hour", col("window.start"))
      .drop("window")
      .orderBy("hour")
)

# 2 — Forward-fill: use last() with ignorenulls over ordered window
fill_window = Window.orderBy("hour").rowsBetween(
    Window.unboundedPreceding, Window.currentRow
)
hourly = hourly.withColumn(
    "avg_value", last("avg_value", ignorenulls=True).over(fill_window)
)

# 3 — Rolling 7-period average
roll_window = Window.orderBy("hour").rowsBetween(-6, 0)
hourly = hourly.withColumn("rolling_7", _avg("avg_value").over(roll_window))
```

> **Interview Tip:** Pandas `resample()` is a one-liner, but Spark has no direct equivalent — you need `groupBy(window(...))`. Knowing this translation is a differentiator.

### Handling Missing Time Periods

When resampling, some time periods may have no data. Pandas `resample()` automatically creates rows for empty periods (filled with NaN). PySpark does **not** — you need to generate a complete time spine (a DataFrame of all expected timestamps) and LEFT JOIN your aggregated data to it. This is a common interview follow-up:

```python
# PySpark: generate hourly time spine
from pyspark.sql.functions import explode, sequence, to_timestamp, lit

time_spine = spark.sql("""
    SELECT explode(
        sequence(
            CAST('2025-03-15 00:00:00' AS TIMESTAMP),
            CAST('2025-03-16 00:00:00' AS TIMESTAMP),
            INTERVAL 1 HOUR
        )
    ) AS hour
""")

result = time_spine.join(hourly, on="hour", how="left").fillna(0)
```

---

## 10. Join Patterns

### Problem

Given `orders` and `customers` tables:

1. **Left join** — all orders with customer info (handle nulls).
2. **Self-join** — find pairs of orders from the same customer placed within 24 hours.
3. **Broadcast join** — join a large `orders` table with a small `regions` lookup.

### Pandas

```python
# 1 — Left join
result = orders.merge(customers, on="customer_id", how="left")
result["customer_name"] = result["customer_name"].fillna("Unknown")

# 2 — Self-join: orders within 24h for same customer
self_joined = orders.merge(orders, on="customer_id", suffixes=("_a", "_b"))
self_joined = self_joined[
    (self_joined["order_id_a"] < self_joined["order_id_b"])        # avoid self-pairs
    & (
        abs(
            (self_joined["order_time_a"] - self_joined["order_time_b"])
            .dt.total_seconds()
        )
        <= 86400
    )
]

# 3 — Broadcast (N/A in Pandas — it's always in-memory)
result = orders.merge(regions, on="region_id", how="left")
```

### PySpark

```python
from pyspark.sql.functions import broadcast, abs as spark_abs, unix_timestamp

# 1 — Left join with null handling
result = (
    orders.join(customers, on="customer_id", how="left")
          .fillna({"customer_name": "Unknown"})
)

# 2 — Self-join
o1 = orders.alias("a")
o2 = orders.alias("b")

self_joined = (
    o1.join(o2, (col("a.customer_id") == col("b.customer_id"))
              & (col("a.order_id") < col("b.order_id")))
      .filter(
          spark_abs(
              unix_timestamp("a.order_time") - unix_timestamp("b.order_time")
          ) <= 86400
      )
)

# 3 — Broadcast join (small dimension table)
result = orders.join(broadcast(regions), on="region_id", how="left")
```

> **Interview Tip:** Always mention `broadcast()` for small dimension tables. In production, Spark's AQE (Adaptive Query Execution) may auto-broadcast, but explicitly marking it shows you understand the physical plan.

### Join Performance in PySpark

Understanding join strategies is crucial for production pipelines:

| Join Type | When to Use | How It Works |
|---|---|---|
| **Sort-Merge Join** | Both sides large | Both DataFrames sorted by key, then merged. Default for large-large joins. |
| **Broadcast Hash Join** | One side < 10 MB (configurable) | Small table broadcast to all executors. O(n) instead of O(n log n). |
| **Shuffle Hash Join** | Medium-sized tables | Hash-partitions both sides by key, then joins per partition. |

You can check which strategy Spark chose with `df.explain(True)` — look for `BroadcastHashJoin` vs `SortMergeJoin` in the physical plan.

---

## 11. Window Functions

### Problem

Given an `employees` table (`emp_id, department, salary, hire_date`), compute:

1. `rank` and `dense_rank` of salary within each department.
2. `row_number` for deterministic ordering.
3. `lag` (previous salary) and `lead` (next salary) within department.
4. **Running total** of salary within department (ordered by hire_date).

### Pandas

```python
df = df.sort_values(["department", "salary"], ascending=[True, False])

# rank / dense_rank
df["salary_rank"]       = df.groupby("department")["salary"].rank(method="min", ascending=False).astype(int)
df["salary_dense_rank"] = df.groupby("department")["salary"].rank(method="dense", ascending=False).astype(int)

# row_number (needs a secondary sort for determinism)
df = df.sort_values(["department", "salary", "emp_id"], ascending=[True, False, True])
df["row_num"] = df.groupby("department").cumcount() + 1

# lag / lead
df = df.sort_values(["department", "hire_date"])
df["prev_salary"] = df.groupby("department")["salary"].shift(1)
df["next_salary"] = df.groupby("department")["salary"].shift(-1)

# Running total (ordered by hire_date)
df["running_total"] = df.groupby("department")["salary"].cumsum()
```

### PySpark

```python
from pyspark.sql.window import Window
from pyspark.sql.functions import (
    rank, dense_rank, row_number, lag, lead,
    sum as _sum, col, desc,
)

dept_salary_w = Window.partitionBy("department").orderBy(desc("salary"))

df = (
    df.withColumn("salary_rank",       rank().over(dept_salary_w))
      .withColumn("salary_dense_rank", dense_rank().over(dept_salary_w))
      .withColumn("row_num",           row_number().over(dept_salary_w))
)

dept_date_w = Window.partitionBy("department").orderBy("hire_date")

df = (
    df.withColumn("prev_salary",  lag("salary", 1).over(dept_date_w))
      .withColumn("next_salary",  lead("salary", 1).over(dept_date_w))
      .withColumn("running_total",
                  _sum("salary").over(
                      dept_date_w.rowsBetween(
                          Window.unboundedPreceding, Window.currentRow
                      )
                  ))
)
```

> **Interview Tip:** In PySpark, `sum().over(window)` without explicit frame boundaries defaults to `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`, which can behave unexpectedly with ties. Always specify `rowsBetween` for running totals.

### ROWS vs. RANGE — The Subtle Difference

This is one of the most common window-function bugs:

- **`ROWS BETWEEN`** — Physical rows. "The 3 rows before me" means exactly 3 rows.
- **`RANGE BETWEEN`** — Logical values. "All rows with values within 3 of mine" — could be 0 rows or 100.

With `ORDER BY salary` and two employees sharing `salary = 100000`:
- `ROWS` treats them as separate rows → running total includes one, then the other.
- `RANGE` treats them as the same logical position → running total includes *both* at once.

For running totals and moving averages, **always use `ROWS`** for deterministic results.

---

## Pandas ↔ PySpark Translation Cheat Sheet

| Operation | Pandas | PySpark |
|---|---|---|
| Read CSV | `pd.read_csv("f.csv")` | `spark.read.csv("f.csv", header=True)` |
| Select columns | `df[["a", "b"]]` | `df.select("a", "b")` |
| Filter rows | `df[df["x"] > 5]` | `df.filter(col("x") > 5)` |
| Add column | `df["new"] = df["a"] + 1` | `df.withColumn("new", col("a") + 1)` |
| Rename column | `df.rename(columns={"a": "b"})` | `df.withColumnRenamed("a", "b")` |
| Drop column | `df.drop(columns=["a"])` | `df.drop("a")` |
| Sort | `df.sort_values("a")` | `df.orderBy("a")` |
| GroupBy + agg | `df.groupby("a").agg(...)` | `df.groupBy("a").agg(...)` |
| Drop duplicates | `df.drop_duplicates(["a"])` | `df.dropDuplicates(["a"])` |
| Fill nulls | `df["a"].fillna(0)` | `df.fillna({"a": 0})` |
| Cast type | `df["a"].astype(int)` | `df.withColumn("a", col("a").cast("int"))` |
| Join | `df1.merge(df2, on="k", how="left")` | `df1.join(df2, on="k", how="left")` |
| Anti-join | `df1[~df1["k"].isin(df2["k"])]` | `df1.join(df2, on="k", how="left_anti")` |
| Pivot | `df.pivot_table(...)` | `df.groupBy(...).pivot(...)` |
| Unpivot / Melt | `df.melt(...)` | `selectExpr("stack(...)")` |
| Window — lag | `df.groupby("g")["c"].shift(1)` | `lag("c", 1).over(window)` |
| Window — cumsum | `df.groupby("g")["c"].cumsum()` | `sum("c").over(window_with_rows)` |
| Window — rank | `df.groupby("g")["c"].rank()` | `rank().over(window)` |
| Resample time | `df.resample("1h").mean()` | `groupBy(window("ts","1 hour"))` |
| Forward-fill | `df["c"].ffill()` | `last("c", True).over(ordered_w)` |
| Rolling window | `df["c"].rolling(7).mean()` | `avg("c").over(w.rowsBetween(-6,0))` |
| Write Parquet | `df.to_parquet("out.parquet")` | `df.write.parquet("out.parquet")` |
| Row count | `len(df)` | `df.count()` |
| Show data | `df.head()` | `df.show()` |
| Schema / dtypes | `df.dtypes` | `df.printSchema()` |
| Broadcast join | *(N/A — always local)* | `df1.join(broadcast(df2), ...)` |
| Coalesce files | *(N/A)* | `df.coalesce(1).write...` |
| Repartition | *(N/A)* | `df.repartition(n, "col")` |
| Explain plan | *(N/A)* | `df.explain(True)` |
| UDF | `df["c"].apply(func)` | `udf(func, returnType)` |
| Collect to list | `df["c"].tolist()` | `[r.c for r in df.collect()]` |

---

---

## Common Interview Mistakes to Avoid

1. **Using `.apply()` in Pandas when vectorized operations exist.** `.apply()` is a Python-level loop — it's 10–100× slower than vectorized alternatives. Always look for a built-in method first.

2. **Forgetting `.cache()` in PySpark.** If you reuse a DataFrame multiple times (e.g., in reconciliation — join it twice), call `.cache()` to avoid recomputing.

3. **Not handling NULLs.** In Pandas, `NaN != NaN` returns `True`. In PySpark, `NULL != NULL` returns `NULL`. Always use `.isna()` / `.isNull()` for null checks, never `==`.

4. **Collecting large DataFrames in PySpark.** `df.collect()` pulls all data to the driver — instant OOM on large datasets. Use `.show()`, `.take(n)`, or write to storage.

5. **Ignoring sort order.** `drop_duplicates` in Pandas respects the current sort order. If you don't sort first, the "kept" row is arbitrary. Always sort before dedup.

6. **Mixing up `groupby().transform()` vs `groupby().agg()` in Pandas.** `transform` returns a Series the same length as the input (broadcasts the result). `agg` returns a reduced DataFrame. Wrong choice = wrong shape.

---

## Final Study Strategy

1. **Pick 3 problems** from this module and solve them **from scratch** in both Pandas and PySpark — no peeking.
2. **Time yourself:** aim for 15–20 minutes per problem (that's the interview pace).
3. **Talk out loud** as you code. Interviewers care about your thought process as much as the code.
4. **Know the translations:** if an interviewer asks for one framework and you only know the other, map it using the cheat sheet above.
5. **Edge cases to always mention:** nulls, duplicates, empty DataFrames, schema mismatches, time zones.

> *"The best data engineers don't just write queries — they think about data quality, performance, and failure modes while they code."*

Good luck — you've got this! 🚀

---

## Appendix: Environment Setup

To practice these problems locally:

```bash
# Create a virtual environment
uv venv .venv
source .venv/bin/activate

# Install dependencies
uv pip install pandas pyarrow pyspark jupyter \
    --index-url https://pypi.ci.artifacts.walmart.com/artifactory/api/pypi/external-pypi/simple \
    --allow-insecure-host pypi.ci.artifacts.walmart.com

# Launch Jupyter for interactive practice
jupyter notebook
```

For PySpark, you'll also need Java 11+ installed. On Mac: `brew install openjdk@11`.
