# Module 2: AWS Glue & Athena — Serverless ETL & Queries

> **Scenario**: RetailMart's S3 data lake (Module 1) is filling up with raw POS transactions, clickstream logs, and inventory feeds. Now you need to **catalog** the data so analysts can find it, **transform** it into query-optimized formats, and let business users **query** it with SQL — all without provisioning a single server.

---

## Screen 1: Glue Data Catalog — The Central Nervous System

The Glue Data Catalog is a **fully managed, Hive-compatible metastore**. It stores table definitions, column schemas, partition metadata, and data locations. Every service — Athena, EMR, Redshift Spectrum, Glue ETL — reads from this single source of truth.

### Architecture: Data Catalog in RetailMart's Ecosystem

```
┌──────────────────────────────────────────────────────────────────────┐
│                      AWS Glue Data Catalog                          │
│                                                                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                  │
│  │  Database:   │  │  Database:   │  │  Database:   │                 │
│  │  raw_zone    │  │  curated     │  │  analytics   │                 │
│  ├─────────────┤  ├─────────────┤  ├─────────────┤                  │
│  │ pos_txns     │  │ daily_sales  │  │ store_perf   │                 │
│  │ clickstream  │  │ inventory    │  │ customer_360 │                 │
│  │ supplier_feed│  │ product_dim  │  │ forecast     │                 │
│  └─────────────┘  └─────────────┘  └─────────────┘                  │
│         │                │                │                          │
└─────────┼────────────────┼────────────────┼──────────────────────────┘
          │                │                │
    ┌─────┼────────────────┼────────────────┼─────┐
    │     ▼                ▼                ▼     │
    │  ┌──────┐  ┌──────┐  ┌──────┐  ┌────────┐  │
    │  │Athena│  │ EMR  │  │Redsh.│  │  Glue  │  │
    │  │      │  │Spark │  │Spectr│  │  ETL   │  │
    │  └──────┘  └──────┘  └──────┘  └────────┘  │
    │         All query the same catalog          │
    └─────────────────────────────────────────────┘
```

### Creating a Database and Table via AWS CLI

```bash
# Create a database
aws glue create-database \
  --database-input '{
    "Name": "curated",
    "Description": "Cleaned, partitioned, Parquet-formatted retail data",
    "LocationUri": "s3://retailmart-data-lake-prod/curated/"
  }'

# Create a table (manual definition)
aws glue create-table \
  --database-name curated \
  --table-input '{
    "Name": "daily_sales",
    "StorageDescriptor": {
      "Columns": [
        {"Name": "transaction_id", "Type": "string"},
        {"Name": "store_id", "Type": "int"},
        {"Name": "product_sku", "Type": "string"},
        {"Name": "quantity", "Type": "int"},
        {"Name": "total_amount", "Type": "decimal(10,2)"},
        {"Name": "payment_method", "Type": "string"},
        {"Name": "transaction_ts", "Type": "timestamp"}
      ],
      "Location": "s3://retailmart-data-lake-prod/curated/daily_sales/",
      "InputFormat": "org.apache.hadoop.hive.ql.io.parquet.MapredParquetInputFormat",
      "OutputFormat": "org.apache.hadoop.hive.ql.io.parquet.MapredParquetOutputFormat",
      "SerdeInfo": {
        "SerializationLibrary": "org.apache.hadoop.hive.ql.io.parquet.serde.ParquetHiveSerDe"
      }
    },
    "PartitionKeys": [
      {"Name": "year", "Type": "string"},
      {"Name": "month", "Type": "string"},
      {"Name": "day", "Type": "string"}
    ],
    "TableType": "EXTERNAL_TABLE",
    "Parameters": {
      "classification": "parquet",
      "compressionType": "snappy"
    }
  }'
```

### 💡 Interview Insight

> **Q: "How does the Glue Data Catalog compare to a standalone Hive metastore?"**
>
> The Glue Data Catalog is API-compatible with Apache Hive Metastore (HMS), so EMR clusters can use it as a drop-in replacement by setting `hive.metastore.client.factory.class` to the AWS Glue factory. Key advantages: (1) **Serverless** — no EC2 for the metastore, no RDS backend to manage. (2) **Cross-service** — Athena, Redshift Spectrum, and EMR all share one catalog. (3) **IAM-based access control** vs. Hive's limited authorization. (4) **Schema versioning** built in. Limitation: The catalog has a soft limit of 1 million tables per account — for massive multi-tenant setups, consider Lake Formation or separate accounts.

---

## Screen 2: Crawlers — Automatic Schema Discovery

Crawlers scan data sources (S3, JDBC, DynamoDB), infer schemas, and register or update tables in the Data Catalog. They're the "auto-discovery" engine that saves you from writing DDL by hand.

### Crawler Configuration for RetailMart Raw Zone

```bash
aws glue create-crawler \
  --name retailmart-raw-pos-crawler \
  --role "arn:aws:iam::123456789:role/GlueCrawlerRole" \
  --database-name raw_zone \
  --targets '{
    "S3Targets": [
      {
        "Path": "s3://retailmart-data-lake-prod/raw/pos-transactions/",
        "Exclusions": ["**/_temporary/**", "**/_SUCCESS"]
      }
    ]
  }' \
  --schema-change-policy '{
    "UpdateBehavior": "UPDATE_IN_DATABASE",
    "DeleteBehavior": "LOG"
  }' \
  --recrawl-policy '{"RecrawlBehavior": "CRAWL_NEW_FOLDERS_ONLY"}' \
  --schedule '{"ScheduleExpression": "cron(0 6 * * ? *)"}'
```

### How a Crawler Works

```
Step 1: CLASSIFY                Step 2: INFER SCHEMA          Step 3: REGISTER
┌──────────────────┐            ┌──────────────────┐           ┌──────────────┐
│ Read file samples │           │ Detect columns,  │           │ Create/update │
│ from S3 path      │──────────►│ types, partitions│──────────►│ table in Data │
│                   │           │ from data content│           │ Catalog       │
│ Detect format:    │           │                  │           │               │
│ Parquet/CSV/JSON  │           │ Handle schema    │           │ Add new       │
│ Avro/ORC/...      │           │ evolution:       │           │ partitions    │
└──────────────────┘            │ - New column     │           └──────────────┘
                                │ - Type change    │
                                │ - Missing field  │
                                └──────────────────┘
```

### Crawler Best Practices

| Practice | Why |
|---|---|
| Use `CRAWL_NEW_FOLDERS_ONLY` | Avoids re-scanning the entire bucket |
| Set exclusions for temp files | `_SUCCESS`, `_temporary`, `_committed` |
| Separate crawlers per table | One crawler per distinct schema |
| Use `LOG` delete behavior | Don't auto-delete tables on data removal |
| Run on schedule after ETL | Crawl after data lands, not before |
| Set table prefixes | Avoid naming collisions: `raw_pos_transactions` |

### 💡 Interview Insight

> **Q: "Your crawler keeps creating new tables instead of updating the existing one. Why?"**
>
> Most common cause: **schema changes** that the crawler interprets as a new table. If the partition structure changes or column types differ significantly, the crawler may create a separate table like `pos_transactions_1`. Fix: (1) Set a **table prefix** to group related tables. (2) Configure `grouping behavior` to merge compatible schemas. (3) Use `SchemaChangePolicy.UpdateBehavior = UPDATE_IN_DATABASE` to update in place. (4) If the raw data genuinely has inconsistent schemas, consider adding a Glue ETL job to normalize before crawling.

---

## Screen 3: Glue ETL Jobs — Spark-Based Transformations

Glue ETL jobs are **managed Apache Spark** environments. You write PySpark (or Scala) scripts, and Glue handles the cluster provisioning, scaling, and teardown.

### RetailMart ETL Pipeline: Raw CSV → Curated Parquet

```python
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from awsglue.context import GlueContext
from awsglue.job import Job
from awsglue.dynamicframe import DynamicFrame
from pyspark.context import SparkContext
from pyspark.sql.functions import col, to_timestamp, year, month, dayofmonth

# ── Initialize Glue context ────────────────────────────────────────
args = getResolvedOptions(sys.argv, ["JOB_NAME", "source_database", "source_table"])
sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args["JOB_NAME"], args)

# ── Step 1: Read from Data Catalog (DynamicFrame) ──────────────────
raw_dyf = glueContext.create_dynamic_frame.from_catalog(
    database=args["source_database"],
    table_name=args["source_table"],
    transformation_ctx="raw_dyf",
    additional_options={"mergeSchema": "true"},
)

print(f"Record count: {raw_dyf.count()}")
raw_dyf.printSchema()

# ── Step 2: Resolve ambiguous types with resolveChoice ─────────────
resolved_dyf = raw_dyf.resolveChoice(
    choice="match_catalog",
    database=args["source_database"],
    table_name=args["source_table"],
    transformation_ctx="resolved_dyf",
)

# ── Step 3: Convert to Spark DataFrame for complex transforms ─────
df = resolved_dyf.toDF()

# Clean and enrich
df_clean = (
    df.filter(col("total_amount") > 0)
    .filter(col("transaction_id").isNotNull())
    .withColumn("transaction_ts", to_timestamp(col("transaction_ts"), "yyyy-MM-dd HH:mm:ss"))
    .withColumn("year", year(col("transaction_ts")).cast("string"))
    .withColumn("month", month(col("transaction_ts")).cast("string").lpad(2, "0"))
    .withColumn("day", dayofmonth(col("transaction_ts")).cast("string").lpad(2, "0"))
    .drop("_raw_line", "_source_file")
)

# ── Step 4: Write back as partitioned Parquet ──────────────────────
output_dyf = DynamicFrame.fromDF(df_clean, glueContext, "output_dyf")

sink = glueContext.getSink(
    connection_type="s3",
    path="s3://retailmart-data-lake-prod/curated/daily_sales/",
    enableUpdateCatalog=True,
    updateBehavior="UPDATE_IN_DATABASE",
    partitionKeys=["year", "month", "day"],
    transformation_ctx="sink",
)
sink.setFormat("glueparquet", compression="snappy")
sink.setCatalogInfo(catalogDatabase="curated", catalogTableName="daily_sales")
sink.writeFrame(output_dyf)

job.commit()
```

### Glue Job Types

| Job Type | Engine | Best For | Pricing |
|---|---|---|---|
| Glue ETL (Spark) | Apache Spark | Large-scale transforms | DPU-hours |
| Glue Streaming | Spark Structured Streaming | Near-real-time from Kinesis/Kafka | DPU-hours |
| Glue Python Shell | Pure Python (no Spark) | Small tasks, API calls | DPU-hours (0.0625 DPU) |
| Glue Ray | Ray (distributed Python) | ML preprocessing, pandas-like | DPU-hours |
| Glue Flex | Spark (preemptible) | Non-urgent, cost-sensitive | 35% cheaper |

### 💡 Interview Insight

> **Q: "When would you use a Glue Python Shell job instead of a Spark ETL job?"**
>
> Python Shell is ideal for tasks that **don't need distributed compute**: (1) API calls to external systems (pulling reference data from a supplier API), (2) Small file operations (moving/renaming < 1 GB), (3) Orchestration logic (triggering other jobs), (4) Data quality checks on metadata. It uses 1/16th of a DPU — dramatically cheaper for lightweight work. At RetailMart, our store-metadata refresh job calls the internal Store API, writes a 50 MB JSON file, and costs $0.02/run vs. $0.44 for a minimum Spark job.

---

## Screen 4: Glue Bookmarks & Incremental Processing

Glue Bookmarks track **which data has already been processed**, enabling incremental ETL without reprocessing the entire dataset.

### How Bookmarks Work

```
Run 1 (Day 1):
  Reads:  s3://raw/pos/2026/04/01/  →  Processes 2.3 TB
  Saves bookmark: {"path": "s3://raw/pos/2026/04/01/", "timestamp": "2026-04-01T23:59:59"}

Run 2 (Day 2):
  Reads bookmark → Knows Day 1 is done
  Reads:  s3://raw/pos/2026/04/02/  →  Processes 2.1 TB (NEW files only)
  Updates bookmark: {"path": "s3://raw/pos/2026/04/02/", "timestamp": "2026-04-02T23:59:59"}

Run 3 (Day 3):
  Reads:  s3://raw/pos/2026/04/03/  →  Only new data
  ...
```

### Enabling Bookmarks

```bash
aws glue start-job-run \
  --job-name retailmart-pos-etl \
  --arguments '{
    "--job-bookmark-option": "job-bookmark-enable",
    "--source_database": "raw_zone",
    "--source_table": "pos_transactions"
  }'
```

### Bookmark Caveats

| Scenario | Bookmark Behavior |
|---|---|
| New files added to existing partition | ✅ Detected and processed |
| Modified existing files | ⚠️ Detected only if timestamp changes |
| Deleted files | ❌ Not tracked |
| Schema changes | ✅ Works (re-reads with new schema) |
| Job fails mid-run | ⟲ Bookmark NOT updated; full retry |
| Re-partition existing data | ❌ Must reset bookmark |

```bash
# Reset bookmark (reprocess everything)
aws glue reset-job-bookmark --job-name retailmart-pos-etl
```

### 💡 Interview Insight

> **Q: "Bookmarks missed processing some files that arrived late. How do you handle late-arriving data?"**
>
> Bookmarks track by file timestamp and path. If a file is written to a partition **after** the bookmark passes that partition, it's missed. Solutions: (1) Use **S3 event notifications + SQS** to track every file arrival explicitly, then process the SQS queue. (2) Set a **lookback window** — always re-read the last 2 days of partitions in addition to new ones. (3) Use **Apache Iceberg or Delta Lake** on Glue, which handle late-arriving data natively via merge operations. At RetailMart, supplier feeds arrive 6-12 hours late, so we use approach (2) with a 24-hour lookback.

---

## Screen 5: Dynamic Frames, resolveChoice & Schema Registry

### DynamicFrame vs. Spark DataFrame

| Feature | DynamicFrame | Spark DataFrame |
|---|---|---|
| Schema | Per-record (flexible) | Enforced (rigid) |
| Type mismatches | `resolveChoice()` | Fails or casts silently |
| Null handling | Built-in transforms | Manual `.na.fill()` |
| Data Catalog integration | Direct read/write | Via Glue catalog API |
| Performance | Slight overhead | Faster for large joins |
| Best for | Ingestion, raw data | Complex transforms, ML |

### resolveChoice — Handling Messy Real-World Data

When RetailMart's POS systems send `total_amount` as sometimes a string, sometimes a double:

```python
# Option 1: Cast to a specific type
resolved = raw_dyf.resolveChoice(specs=[
    ("total_amount", "cast:double"),
    ("quantity", "cast:int"),
    ("store_id", "cast:int"),
])

# Option 2: Match the catalog definition
resolved = raw_dyf.resolveChoice(
    choice="match_catalog",
    database="raw_zone",
    table_name="pos_transactions",
)

# Option 3: Create separate columns for each type
resolved = raw_dyf.resolveChoice(specs=[
    ("total_amount", "make_cols"),
])
# Results in: total_amount_double, total_amount_string
```

### Glue Schema Registry

For **streaming** workloads (Kinesis, Kafka), the Schema Registry validates that incoming records match an expected schema before they enter the lake.

```
Producer ──► Schema Registry ──► Kinesis ──► Glue Streaming ETL ──► S3
               │                                    │
               │  Validates against                  │
               │  registered Avro/JSON schema        │
               │                                    │
               └── Rejects invalid records ─────────┘
                   (schema evolution rules)
```

```bash
# Create a registry
aws glue create-registry \
  --registry-name retailmart-streaming

# Register a schema
aws glue create-schema \
  --registry-id '{"RegistryName": "retailmart-streaming"}' \
  --schema-name pos-transaction-event \
  --data-format AVRO \
  --compatibility BACKWARD \
  --schema-definition '{
    "type": "record",
    "name": "PosTransaction",
    "fields": [
      {"name": "transaction_id", "type": "string"},
      {"name": "store_id", "type": "int"},
      {"name": "total_amount", "type": "double"},
      {"name": "transaction_ts", "type": "long", "logicalType": "timestamp-millis"}
    ]
  }'
```

### Glue Schema Registry Compatibility Modes

| Mode | Rules | Use Case |
|---|---|---|
| BACKWARD | New schema can read old data | Consumers updated before producers |
| FORWARD | Old schema can read new data | Producers updated before consumers |
| FULL | Both directions compatible | Safest — both sides evolve independently |
| NONE | No validation | Development only |

For RetailMart's streaming POS events, we use **BACKWARD** compatibility — the consumer (Glue Streaming ETL) always runs the latest schema and must handle records written with any previous schema. Adding a new optional field is allowed; removing or renaming a required field is rejected.

### 💡 Interview Insight

> **Q: "When should you convert a DynamicFrame to a Spark DataFrame?""
>
> Convert when you need: (1) **Complex joins** — DynamicFrame's join support is limited. (2) **Window functions** — `rank()`, `lag()`, `lead()` require DataFrames. (3) **ML feature engineering** — Spark MLlib only works with DataFrames. (4) **UDFs** — custom transformations with Python UDFs. The common pattern: read with DynamicFrame (schema flexibility), `resolveChoice` to clean types, `.toDF()` for transforms, `DynamicFrame.fromDF()` to write back via Glue sink. This gives you the best of both worlds.

---

## Screen 6: Athena — Serverless SQL on S3

Amazon Athena is a **serverless, interactive query service** built on Trino (formerly Presto). You point it at S3 data registered in the Glue Data Catalog and run standard SQL.

### Athena Architecture

```
┌──────────────┐      ┌──────────────────┐      ┌─────────────────┐
│   Analyst /   │      │                  │      │                 │
│   BI Tool     │─────►│   Amazon Athena   │─────►│   S3 Data Lake  │
│  (SQL query)  │      │   (Trino engine)  │      │   (Parquet/ORC) │
│               │◄─────│                  │◄─────│                 │
│   Results     │      │  Glue Data       │      │  Scans only     │
│               │      │  Catalog lookup  │      │  needed columns │
└──────────────┘      └──────────────────┘      │  & partitions   │
                                                 └─────────────────┘

Cost: $5.00 per TB of data scanned
```

### Querying RetailMart Sales Data

```sql
-- Top 10 stores by revenue, last 7 days
SELECT
    store_id,
    COUNT(*)                  AS transaction_count,
    SUM(total_amount)         AS total_revenue,
    AVG(total_amount)         AS avg_transaction_value,
    APPROX_PERCENTILE(total_amount, 0.95) AS p95_transaction
FROM curated.daily_sales
WHERE year = '2026'
  AND month = '04'
  AND day BETWEEN '01' AND '07'
  AND total_amount > 0
GROUP BY store_id
ORDER BY total_revenue DESC
LIMIT 10;
```

### Partition Projection — Avoid `MSCK REPAIR TABLE`

Instead of storing partition metadata in the Glue Data Catalog (requiring `MSCK REPAIR TABLE` or crawler runs), **partition projection** tells Athena to compute partitions on the fly.

```sql
CREATE EXTERNAL TABLE curated.daily_sales_projected (
    transaction_id STRING,
    store_id       INT,
    product_sku    STRING,
    quantity       INT,
    total_amount   DECIMAL(10,2),
    payment_method STRING,
    transaction_ts TIMESTAMP
)
PARTITIONED BY (year STRING, month STRING, day STRING)
STORED AS PARQUET
LOCATION 's3://retailmart-data-lake-prod/curated/daily_sales/'
TBLPROPERTIES (
    'projection.enabled'        = 'true',
    'projection.year.type'      = 'integer',
    'projection.year.range'     = '2020,2030',
    'projection.month.type'     = 'integer',
    'projection.month.range'    = '1,12',
    'projection.month.digits'   = '2',
    'projection.day.type'       = 'integer',
    'projection.day.range'      = '1,31',
    'projection.day.digits'     = '2',
    'storage.location.template' =
        's3://retailmart-data-lake-prod/curated/daily_sales/year=${year}/month=${month}/day=${day}/'
);
```

**Benefits:** No crawler needed for partitions, no `MSCK REPAIR TABLE`, queries start instantly, and the Glue Data Catalog doesn't need to store millions of partition entries.

### CTAS — Create Table As Select

```sql
-- Materialize a denormalized view as optimized Parquet
CREATE TABLE analytics.store_daily_summary
WITH (
    format = 'PARQUET',
    parquet_compression = 'SNAPPY',
    partitioned_by = ARRAY['year', 'month'],
    external_location = 's3://retailmart-data-lake-prod/analytics/store_daily_summary/',
    bucketed_by = ARRAY['store_id'],
    bucket_count = 32
) AS
SELECT
    store_id,
    day,
    COUNT(*)          AS txn_count,
    SUM(total_amount) AS revenue,
    COUNT(DISTINCT product_sku) AS unique_products,
    year,
    month
FROM curated.daily_sales
WHERE year = '2026'
GROUP BY store_id, day, year, month;
```

### 💡 Interview Insight

> **Q: "An Athena query is scanning 500 GB but only needs 1 day of data. How do you reduce cost?"**
>
> Three levers: (1) **Partition pruning** — ensure the WHERE clause filters on partition columns (`year`, `month`, `day`). If the table isn't partitioned, create a new partitioned version with CTAS. (2) **Columnar format** — convert CSV/JSON to Parquet or ORC. A query selecting 5 out of 50 columns scans ~10% of the data. (3) **Compression** — Snappy or ZSTD reduces bytes scanned further. Combined, these can reduce scans from 500 GB to under 5 GB — saving $2.50 per query. At RetailMart, we mandated Parquet + Snappy + date-partitioning and cut Athena costs by 94%.

---

## Screen 7: Athena Advanced — Federation, Iceberg & Spark Notebooks

### Query Federation

Athena can query data **outside S3** using federated connectors — DynamoDB, RDS, Redshift, CloudWatch Logs, and custom sources via Lambda.

```
┌────────────┐    ┌────────────┐    ┌────────────┐    ┌────────────┐
│  S3 Data   │    │ DynamoDB   │    │   RDS      │    │ CloudWatch │
│  Lake      │    │ (product   │    │ (store     │    │  Logs      │
│  (sales)   │    │  catalog)  │    │  metadata) │    │ (app logs) │
└─────┬──────┘    └─────┬──────┘    └─────┬──────┘    └─────┬──────┘
      │                 │                 │                 │
      └────────┬────────┴────────┬────────┴────────┬────────┘
               │     Athena Federated Query        │
               │     (Lambda connectors)           │
               ▼                                   ▼
        ┌──────────────────────────────────────────────┐
        │                Amazon Athena                  │
        │                                              │
        │  SELECT s.store_id, p.product_name,          │
        │         SUM(s.total_amount)                   │
        │  FROM curated.daily_sales s                   │
        │  JOIN dynamodb.product_catalog p              │
        │    ON s.product_sku = p.sku                   │
        │  JOIN rds.store_metadata m                    │
        │    ON s.store_id = m.store_id                 │
        │  GROUP BY s.store_id, p.product_name          │
        └──────────────────────────────────────────────┘
```

### Apache Iceberg on Athena

Iceberg tables add **ACID transactions**, **time travel**, **schema evolution**, and **row-level updates/deletes** to your S3 data lake.

```sql
-- Create an Iceberg table
CREATE TABLE curated.daily_sales_iceberg (
    transaction_id STRING,
    store_id       INT,
    product_sku    STRING,
    quantity       INT,
    total_amount   DECIMAL(10,2),
    payment_method STRING,
    transaction_ts TIMESTAMP
)
PARTITIONED BY (day(transaction_ts))
LOCATION 's3://retailmart-data-lake-prod/curated/daily_sales_iceberg/'
TBLPROPERTIES (
    'table_type' = 'ICEBERG',
    'format'     = 'PARQUET',
    'write_compression' = 'snappy'
);

-- UPSERT (MERGE) — handle late-arriving corrections
MERGE INTO curated.daily_sales_iceberg target
USING staging.corrections source
ON target.transaction_id = source.transaction_id
WHEN MATCHED THEN
    UPDATE SET total_amount = source.corrected_amount
WHEN NOT MATCHED THEN
    INSERT (transaction_id, store_id, product_sku, quantity,
            total_amount, payment_method, transaction_ts)
    VALUES (source.transaction_id, source.store_id, source.product_sku,
            source.quantity, source.corrected_amount, source.payment_method,
            source.transaction_ts);

-- Time travel — query data as it was 3 days ago
SELECT COUNT(*), SUM(total_amount)
FROM curated.daily_sales_iceberg
FOR TIMESTAMP AS OF TIMESTAMP '2026-04-03 00:00:00';
```

### Athena Spark Notebooks

Athena now supports **Apache Spark** sessions for data exploration that goes beyond SQL:

```python
# In Athena Spark notebook
from pyspark.sql import functions as F

df = spark.sql("SELECT * FROM curated.daily_sales WHERE year='2026' AND month='04'")

# Compute rolling 7-day average per store
from pyspark.sql.window import Window

window_7d = Window.partitionBy("store_id").orderBy("day").rowsBetween(-6, 0)

df_rolling = df.groupBy("store_id", "day") \
    .agg(F.sum("total_amount").alias("daily_revenue")) \
    .withColumn("rolling_7d_avg", F.avg("daily_revenue").over(window_7d))

df_rolling.show(10)
```

### 💡 Interview Insight

> **Q: "When would you choose Iceberg tables on Athena vs. standard Hive-style Parquet tables?"**
>
> Use Iceberg when: (1) You need **row-level updates/deletes** (GDPR right-to-delete, late corrections). (2) You want **time travel** for audit or debugging. (3) You have **concurrent writers** (Iceberg provides ACID). (4) **Schema evolution** is frequent — Iceberg handles column adds/renames/reorders without rewriting data. Stick with Hive-style when: data is append-only, schema is stable, and you want maximum tool compatibility. At RetailMart, `daily_sales` is Iceberg (corrections come in daily), while `clickstream` stays Hive-style (append-only, never updated).

---

## Screen 8: Cost Optimization — Athena & Glue

### Athena Cost Optimization Cheat Sheet

```
Raw CSV query:     SELECT * FROM raw.logs WHERE date='2026-04-05'
                   → Scans 500 GB → Cost: $2.50

Convert to Parquet: Same query
                   → Scans 50 GB  → Cost: $0.25  (90% reduction)

Add partitioning:  WHERE year='2026' AND month='04' AND day='05'
                   → Scans 2 GB   → Cost: $0.01  (99.6% reduction)

Add column pruning: SELECT user_id, event_type (2 of 40 columns)
                   → Scans 0.1 GB → Cost: $0.0005

Total savings:     $2.50 → $0.0005 = 99.98% cost reduction
```

### Glue ETL Cost Optimization

| Strategy | Savings | How |
|---|---|---|
| Right-size DPUs | 20-40% | Start with 2 DPUs, scale up only if needed |
| Use Glue Flex | 35% | For non-time-sensitive batch jobs |
| Enable auto-scaling | 15-25% | `MaxCapacity` + `NumberOfWorkers` dynamic |
| Bookmarks | 60-80% | Process only new data, not entire dataset |
| Worker type selection | Variable | G.1X for standard, G.2X for memory-heavy |
| Python Shell for small tasks | 90%+ | 1/16 DPU vs. minimum 2 DPU Spark |

### Glue Job Auto-Scaling Configuration

```bash
aws glue create-job \
  --name retailmart-pos-etl \
  --role "arn:aws:iam::123456789:role/GlueETLRole" \
  --command '{
    "Name": "glueetl",
    "ScriptLocation": "s3://retailmart-scripts/etl/pos_transform.py",
    "PythonVersion": "3"
  }' \
  --glue-version "4.0" \
  --worker-type "G.1X" \
  --number-of-workers 10 \
  --execution-property '{"MaxConcurrentRuns": 3}' \
  --default-arguments '{
    "--enable-auto-scaling": "true",
    "--enable-continuous-cloudwatch-log": "true",
    "--enable-metrics": "true",
    "--enable-spark-ui": "true",
    "--spark-event-logs-path": "s3://retailmart-logs/glue-spark-ui/",
    "--job-bookmark-option": "job-bookmark-enable",
    "--TempDir": "s3://retailmart-temp/glue/"
  }'
```

### 💡 Interview Insight

> **Q: "Your Glue ETL job takes 45 minutes to process 10 GB of data. How do you optimize it?"**
>
> Red flag: 10 GB should NOT take 45 minutes. Diagnosis: (1) Check if it's the **small file problem** — thousands of tiny files mean excessive S3 list/get overhead. Fix with `groupFiles` and `groupSize` options. (2) Check DPU utilization in CloudWatch — if < 30%, you're over-provisioned (wasting money, not time). (3) Look for **data skew** — one partition much larger than others causes straggler tasks. (4) Check for unnecessary shuffles — repartition or broadcast smaller DataFrames. (5) Consider **Glue Python Shell** if the logic is simple — Spark has 30+ seconds of startup overhead.
>
> ```python
> # Fix small file problem in Glue
> dyf = glueContext.create_dynamic_frame.from_catalog(
>     database="raw_zone",
>     table_name="pos_transactions",
>     additional_options={
>         "groupFiles": "inPartition",
>         "groupSize": "134217728"  # 128 MB groups
>     }
> )
> ```

---

## Screen 9: Module 2 Quiz & Key Takeaways

### Quiz

**Q1.** RetailMart's data lake has 50,000 partitions in the `daily_sales` table. Running `MSCK REPAIR TABLE` takes 20 minutes. What's the best solution?

- A) Increase Athena query timeout
- B) Use partition projection
- C) Run the crawler more frequently
- D) Reduce the number of partitions

**Answer: B.** Partition projection eliminates the need for `MSCK REPAIR TABLE` entirely — Athena computes partition locations from the template pattern at query time. No metadata updates needed, no crawler, instant query start.

---

**Q2.** A Glue ETL job reads from a crawler-discovered table where `price` is sometimes a string ("$19.99") and sometimes a double (19.99). The job crashes on type mismatch. What's the Glue-native solution?

- A) Fix the source data
- B) Use `resolveChoice(specs=[("price", "cast:double")])`
- C) Convert to DataFrame immediately and cast
- D) Use `resolveChoice(specs=[("price", "make_cols")])`

**Answer: B.** `resolveChoice` with `cast:double` attempts to parse strings to doubles. Option D would create two columns (`price_string`, `price_double`), which may be useful for debugging but doesn't solve the problem. While A is ideal long-term, B is the Glue-native fix for mixed types.

---

**Q3.** An Athena query selecting 3 columns from a 200-column CSV table scans the full 1 TB dataset. After converting to Parquet with Snappy compression, approximately how much data will Athena scan?

- A) 1 TB (no change)
- B) ~500 GB (compression only)
- C) ~15 GB (column pruning only)
- D) ~7-10 GB (column pruning + compression)

**Answer: D.** Parquet is columnar, so only 3/200 columns are read = 1.5% of data. Snappy typically achieves 3-5x compression. So: 1 TB × 0.015 × 0.30 ≈ 4.5–10 GB. The actual number depends on data cardinality and type, but D is the closest.

---

**Q4.** Which Glue feature prevents reprocessing of data that was already transformed in a previous job run?

- A) Crawlers with `CRAWL_NEW_FOLDERS_ONLY`
- B) Job bookmarks
- C) Schema Registry
- D) DynamicFrame caching

**Answer: B.** Job bookmarks track the high-water mark of processed data, ensuring only new or modified files are read on subsequent runs.

---

**Q5.** RetailMart needs to join S3 sales data with product information stored in DynamoDB and store metadata in an RDS PostgreSQL database — all in a single SQL query. Which service supports this?

- A) Glue ETL with multiple data sources
- B) Athena with federated query connectors
- C) EMR with multi-source Spark
- D) All of the above

**Answer: D.** All three can do it, but **B (Athena federated query)** is the simplest for ad-hoc SQL. Glue ETL requires writing PySpark code. EMR requires cluster management. Athena federated query uses Lambda connectors to query DynamoDB and RDS inline.

---

### Bonus: Athena Workgroups for Cost Control

Workgroups let you isolate query execution, enforce per-query data scan limits, and track costs by team.

```bash
# Create a workgroup with a 10 GB per-query scan limit
aws athena create-work-group \
  --name "retail-analysts" \
  --configuration '{
    "ResultConfiguration": {
      "OutputLocation": "s3://retailmart-athena-results/analysts/"
    },
    "EnforceWorkGroupConfiguration": true,
    "BytesScannedCutoffPerQuery": 10737418240,
    "PublishCloudWatchMetricsEnabled": true,
    "EngineVersion": {"SelectedEngineVersion": "Athena engine version 3"}
  }' \
  --tags Key=Team,Value=RetailAnalytics Key=CostCenter,Value=RA-001
```

RetailMart uses three workgroups: `data-engineering` (100 GB limit), `retail-analysts` (10 GB limit), and `executive-dashboards` (1 GB limit — forces them to query curated tables, not raw data).

### 🔑 Key Takeaways

1. **Glue Data Catalog is THE metadata store** — shared by Athena, EMR, Redshift Spectrum, and Glue ETL. One catalog to rule them all.
2. **Crawlers are convenient but opinionated** — use `CRAWL_NEW_FOLDERS_ONLY`, set exclusions, and use one crawler per table for predictable behavior.
3. **DynamicFrames for ingestion, DataFrames for transformation** — `resolveChoice` is your friend for messy real-world data.
4. **Job bookmarks enable incremental processing** — but have edge cases with late-arriving data. Consider lookback windows.
5. **Athena cost = bytes scanned** — Parquet + partitioning + column pruning can reduce costs by 99%+.
6. **Partition projection > MSCK REPAIR TABLE** — eliminates catalog update latency for high-cardinality partitions.
7. **Iceberg on Athena** unlocks ACID, time travel, and row-level updates — use for mutable datasets.
8. **Match the tool to the task** — Python Shell for small jobs, Spark for big transforms, Flex for non-urgent work.
