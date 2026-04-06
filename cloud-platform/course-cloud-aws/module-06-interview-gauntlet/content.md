# Module 6: The Interview Gauntlet 🔥

> **This module is a full simulation.** No gentle explanations. No hand-holding. You're sitting across from a senior data engineer or a principal architect. They're asking rapid-fire questions to find out if you actually understand AWS data engineering or if you memorized flashcards. Each answer includes the **reasoning** — because interviewers probe deeper when you get the first part right.

---

## Screen 1: Architecture Design — "Draw Me a Pipeline"

These questions test your ability to compose AWS services into coherent, production-ready architectures. Interviewers want to hear you reason through trade-offs, not just name services.

---

### Q1: Design a Data Lake for a 10,000-Store Retail Chain

**Interviewer**: "You're building a data lake from scratch for a 10,000-store retailer. They have POS transactions, clickstream, inventory snapshots, and supplier feeds. Data volume is 5 TB/day. Analysts want SQL access. ML engineers need raw data. Compliance requires 7-year retention and PII controls. Walk me through the architecture."

**Strong Answer**:

```
Ingestion Layer:
  POS transactions → Kinesis Data Streams (real-time, 10 shards)
                    → Firehose (buffer 300s, convert to Parquet, partition by date/store)
                    → S3 raw zone: s3://datalake/raw/pos/year=YYYY/month=MM/day=DD/

  Clickstream      → Kinesis Data Streams (20 shards, On-Demand mode)
                    → Firehose → S3 raw zone (JSON → Parquet)

  Inventory        → Batch upload via S3 Transfer Acceleration
                    → Lambda trigger validates schema on landing

  Supplier feeds   → S3 directly via vendor S3 Access Point (write-only)
                    → EventBridge rule triggers validation Lambda

Storage Layer (S3):
  raw/       → Landing zone, immutable, append-only
  curated/   → Cleaned Parquet, partitioned by date, Iceberg for mutable tables
  analytics/ → Aggregated tables for dashboards
  archive/   → Glacier Deep Archive after 1 year

  Lifecycle: Standard (30d) → Standard-IA (90d) → Glacier Instant (365d) → Deep Archive (7yr) → Delete
  Encryption: SSE-KMS with Bucket Key, key rotation enabled
  Versioning: Enabled with 3-version noncurrent retention

Processing Layer:
  Glue ETL (raw → curated): PySpark jobs with bookmarks for incremental processing
  Glue Crawlers: Update Data Catalog partitions post-ETL
  EMR (optional): For heavy ML feature engineering with Spark/PySpark
  Step Functions: Orchestrate nightly pipeline (check files → parallel ETL → DQ → load)

Query Layer:
  Athena: Ad-hoc SQL for analysts (workgroups with scan limits per team)
  Redshift Serverless: Production dashboards, complex joins, materialized views
  Redshift Spectrum: Join warehouse (Redshift) with historical data (S3)

Governance:
  Lake Formation: Centralized permissions, LF-Tags for domain/sensitivity
  Column-level security: Hide PII (customer_id, payment details) from analysts
  Cross-account sharing: Analytics subsidiary gets read-only via LF data shares
  KMS: Per-pipeline encryption keys with least-privilege key policies

Monitoring:
  CloudWatch: Glue job metrics, Kinesis IncomingBytes, S3 request counts
  EventBridge: Alert on ETL failure
  AWS Config: Track S3 bucket policy changes
```

**Why this answer works**: It covers all five layers (ingestion, storage, processing, query, governance), names specific services with justified choices, handles PII/compliance, and shows cost awareness (lifecycle rules, workgroup scan limits). The interviewer can drill into any layer.

---

### Q2: Design a Real-Time Fraud Detection Pipeline

**Interviewer**: "POS transactions need to be analyzed for fraud within 10 seconds. If a customer's card is used at 3+ different stores within 5 minutes, alert the fraud team. Design it."

**Strong Answer**:

```
┌─────────┐    ┌──────────────┐    ┌──────────────────┐    ┌──────────────┐
│ POS     │    │ Kinesis Data  │    │ Kinesis Data     │    │ Downstream   │
│ Systems │───►│ Streams       │───►│ Analytics (Flink)│───►│              │
│ (4K     │    │ (On-Demand)   │    │                  │    │ Alert stream │
│  stores)│    │               │    │ Sliding window:  │    │ → Lambda     │
└─────────┘    │ Partition key │    │ 5-min, 1-min     │    │ → SNS/PagerDuty
               │ = card_hash   │    │ slide            │    │              │
               │ (ordering per │    │                  │    │ S3 (audit)   │
               │  card)        │    │ GROUP BY card_id │    │ → Firehose   │
               └──────────────┘    │ HAVING COUNT     │    └──────────────┘
                                    │ (DISTINCT store) │
                                    │ >= 3             │
                                    └──────────────────┘

Key design decisions:
1. Partition key = card hash → all events for same card hit same shard → correct windowing
2. Flink sliding window (5-min window, 1-min slide) → re-evaluates every minute
3. Watermark = 30 seconds (POS events rarely arrive later)
4. Alert goes to a separate Kinesis stream → Lambda invokes fraud API
5. Firehose parallel consumer sends ALL events to S3 for audit trail
6. Enhanced Fan-Out: Flink consumer + Firehose consumer both get dedicated 2 MB/s

Latency budget:
  POS → Kinesis: ~200ms
  Kinesis → Flink processing: ~1-2 seconds
  Flink → Alert stream: ~100ms
  Lambda → PagerDuty: ~500ms
  Total: < 5 seconds ✅
```

---

### Q3: Design a CDC Pipeline from RDS to the Data Lake

**Interviewer**: "The operational database is RDS PostgreSQL. You need change data capture to keep the data lake in sync — inserts, updates, and deletes. How?"

**Strong Answer**:

```
Option A (Recommended for most cases): DMS + Kinesis + Glue

  RDS PostgreSQL ──► AWS DMS (CDC mode) ──► Kinesis Data Stream
       │                                          │
       │ WAL-based logical replication             │
       │ (no application changes)                  ▼
       │                                    Firehose → S3 (raw CDC events)
       │                                          │
       │                                    Glue ETL (apply CDC to Iceberg table)
       │                                          │
       │                                    curated.orders (Iceberg)
       │                                    MERGE INTO on primary key
       └──────────────────────────────────────────┘

Option B (Simpler, higher latency): DMS direct to S3

  RDS PostgreSQL ──► AWS DMS (CDC mode) ──► S3 (CSV/Parquet)
                                                │
                                          Glue ETL (merge)
                                                │
                                          curated.orders

DMS CDC event format:
  {"Op": "U", "before": {"id": 42, "status": "pending"},
   "after":  {"id": 42, "status": "shipped"}, "ts": "2026-04-05T14:30:00Z"}

Glue ETL applies changes:
  MERGE INTO curated.orders_iceberg target
  USING staging.cdc_events source
  ON target.order_id = source.after.id
  WHEN MATCHED AND source.Op = 'D' THEN DELETE
  WHEN MATCHED AND source.Op = 'U' THEN UPDATE SET *
  WHEN NOT MATCHED AND source.Op IN ('I', 'L') THEN INSERT *;

Key considerations:
  • DMS uses PostgreSQL logical replication slots (monitor replication lag)
  • Iceberg is essential for DELETE operations (Hive tables are append-only)
  • Full load first, then CDC — DMS handles this transition automatically
  • Schema evolution: DMS task can detect new columns if configured
```

---

### Q4: Design a Multi-Region Disaster Recovery Strategy

**Interviewer**: "RetailMart's data lake is in us-east-1. If the region goes down, how do we keep analytics running?"

**Strong Answer**:

```
Tier 1 — Critical (RPO: 15 min, RTO: 1 hour):
  S3 CRR: curated/ prefix replicates to eu-west-1 (15-min SLA)
  Redshift: Cross-region snapshots every 1 hour (automated)
  Glue Catalog: Replicate via Glue API (export/import) or use LF cross-region sharing
  Kinesis: Producers failover to eu-west-1 stream (DNS-based routing)

Tier 2 — Important (RPO: 4 hours, RTO: 24 hours):
  S3 CRR: raw/ prefix replicates with lower priority
  Redshift: Manual snapshot restore

Tier 3 — Historical (RPO: 24 hours, RTO: 72 hours):
  S3 CRR for analytics/ prefix
  Archive data in both regions (Glacier Deep Archive)

Failover process:
  1. Route 53 health check detects us-east-1 outage
  2. DNS failover switches producers to eu-west-1 Kinesis endpoints
  3. Stand up Redshift Serverless in eu-west-1 from latest snapshot
  4. Point Athena to eu-west-1 Glue Catalog (replicated)
  5. Step Functions pipeline in eu-west-1 activates (dormant until failover)

Cost optimization:
  • DR region uses Redshift Serverless (scales to zero when not active)
  • S3 replication to Standard-IA in DR region (cheaper storage)
  • Only replicate curated/ (re-derivable from raw/) — reduces CRR data by 60%
```

---

### Q5: Design a Data Mesh on AWS

**Interviewer**: "Your organization has 5 business domains: Sales, Inventory, HR, Finance, Marketing. Each domain wants to own their data pipeline. How do you implement a data mesh?"

**Strong Answer**:

```
Per-Domain (each domain gets):
  • Separate AWS account (org unit)
  • Own S3 bucket, Glue Catalog, Glue ETL jobs
  • Own Redshift Serverless namespace
  • Domain team manages their pipelines end-to-end

Cross-Domain Sharing (the mesh):
  • Lake Formation cross-account sharing via LF-Tags
  • Tag taxonomy: domain, sensitivity, data-product-status
  • Central governance account:
    - Defines LF-Tags (the taxonomy)
    - Sets minimum security standards (encryption, PII rules)
    - Does NOT own the data or pipelines
  • Redshift data sharing: each domain publishes "data products"
    as shared datasets accessible cross-account

Data Product Contract:
  • Each domain publishes a "data product" with:
    - Schema (Glue Catalog table)
    - SLA (freshness guarantee)
    - Quality metrics (Great Expectations results)
    - Owner (team + PagerDuty)
    - Tags (domain, sensitivity, PII flag)

Discovery:
  • Central Glue Catalog (aggregated via LF cross-account)
  • Data catalog UI (AWS DataZone or custom)
  • Each data product has a README in its S3 prefix

Governance guardrails (central):
  • SCPs enforce encryption, block public S3, require tagging
  • Lake Formation enforces PII rules across all accounts
  • AWS Config rules audit compliance
```

---

### Q6: Design a Slowly Changing Dimension (SCD Type 2) Pipeline

**Interviewer**: "The store dimension table changes when stores are remodeled, relocated, or change format. You need to track history (SCD Type 2). Design the pipeline."

**Strong Answer**:

```sql
-- Iceberg table with SCD Type 2 structure
CREATE TABLE curated.dim_store_scd2 (
    store_sk        BIGINT,          -- Surrogate key
    store_id        INTEGER,          -- Business key
    store_name      VARCHAR(100),
    region          VARCHAR(30),
    format          VARCHAR(20),      -- e.g., "Supercenter" → "Neighborhood Market"
    sq_footage      INTEGER,
    effective_date  DATE,
    end_date        DATE,             -- NULL = current record
    is_current      BOOLEAN,
    record_hash     VARCHAR(64)       -- SHA-256 of changeable columns
) USING ICEBERG
PARTITIONED BY (is_current);

-- Glue ETL logic (PySpark):
-- 1. Read incoming store feed (daily snapshot)
-- 2. Compute hash of changeable columns
-- 3. Compare with current records in dim_store_scd2
-- 4. For changed records:
--    a. UPDATE existing: set end_date = today - 1, is_current = false
--    b. INSERT new version: effective_date = today, is_current = true
-- 5. For new stores: INSERT with is_current = true
-- 6. For deleted stores: set end_date = today, is_current = false

-- Using Iceberg MERGE for the update step:
MERGE INTO curated.dim_store_scd2 target
USING (
    SELECT s.*, SHA2(CONCAT(store_name, region, format, sq_footage), 256) AS new_hash
    FROM staging.store_daily_feed s
) source
ON target.store_id = source.store_id AND target.is_current = true
WHEN MATCHED AND target.record_hash != source.new_hash THEN
    UPDATE SET end_date = CURRENT_DATE - 1, is_current = false;

-- Then INSERT the new versions in a second pass
INSERT INTO curated.dim_store_scd2
SELECT
    NEXT_SURROGATE_KEY(),
    s.store_id, s.store_name, s.region, s.format, s.sq_footage,
    CURRENT_DATE, NULL, true,
    SHA2(CONCAT(s.store_name, s.region, s.format, s.sq_footage), 256)
FROM staging.store_daily_feed s
JOIN curated.dim_store_scd2 t
    ON s.store_id = t.store_id
    AND t.is_current = false
    AND t.end_date = CURRENT_DATE - 1;
```

---

### Q7: Design a Cost-Optimized Analytics Architecture

**Interviewer**: "Budget is tight. The team spends $15,000/month on AWS analytics. Reduce it by 50% without sacrificing core capabilities."

**Strong Answer**:

```
Current state analysis:
  Redshift dc2.large × 4 nodes (always on): $5,400/mo
  Glue ETL (10 DPU jobs, full reprocessing nightly): $4,200/mo
  Athena (analysts scanning raw CSV data): $3,000/mo
  S3 (2 PB all in Standard): $2,400/mo

Optimized ($7,200/mo target):

  Redshift dc2 → Redshift Serverless (32 RPU base):
    Active ~8 hours/day → ~$1,800/mo (67% savings)

  Glue ETL:
    Enable bookmarks (process only new data): -60% runtime
    Switch to Glue Flex for non-urgent jobs: -35% on remaining
    Python Shell for small tasks (metadata refresh): -90% per task
    New cost: ~$1,500/mo (64% savings)

  Athena:
    Convert CSV → Parquet + Snappy: -90% scan volume
    Partition by date: -95% for time-bounded queries
    Workgroup scan limits: prevent accidental full-table scans
    New cost: ~$300/mo (90% savings)

  S3:
    Lifecycle policies: 
      30d → Standard-IA: 400 TB @ $0.0125 = $5,000 → saves $3,600/yr
      90d → Glacier Instant: 800 TB
      365d → Deep Archive: 600 TB
    Delete old noncurrent versions
    New cost: ~$1,200/mo (50% savings)

  Total optimized: ~$4,800/mo (68% reduction)
```

---

### Q8: Design an Event-Driven Microservice Integration with the Data Lake

**Interviewer**: "The order management microservice emits events (order-created, order-shipped, order-returned). How do you ingest these into the data lake for analytics?"

**Strong Answer**:

```
Microservice → EventBridge (custom event bus) → Rules:
  Rule 1: order-* events → Kinesis Data Stream (for real-time)
  Rule 2: order-* events → SQS (dead-letter for failed processing)
  Rule 3: order-returned events → SNS → Lambda (immediate alert to ops)

Kinesis → Firehose → S3 (raw/orders/year=YYYY/month=MM/day=DD/)
  • Dynamic partitioning by event_type
  • Parquet conversion via Glue schema
  • Buffer: 128 MB or 300 seconds

Glue ETL (hourly):
  • Read raw CDC-like events
  • Apply to Iceberg orders table (MERGE)
  • Maintain current_state view + full event history

EventBridge advantages over direct Kinesis:
  • Content-based filtering (only route order-returned to Lambda)
  • Schema validation via EventBridge Schema Registry
  • Event replay (archive + replay for reprocessing)
  • Decoupled from producers (microservice doesn't know about Kinesis)
```

---

## Screen 2: Service Selection — "When Would You Use X vs. Y?"

These questions test whether you understand the subtle differences between overlapping services.

---

### Q1: EMR vs. Glue — When Do You Choose Each?

| Factor | Choose Glue | Choose EMR |
|---|---|---|
| Team size | Small (< 5 data engineers) | Large (dedicated platform team) |
| Job duration | Minutes to low hours | Hours to days (long-running) |
| Customization | Standard PySpark + transforms | Custom libraries, GPU, Presto, Hudi |
| Scaling | Auto-scaling built-in | Fine-grained cluster tuning |
| Cost model | Per-DPU-hour, no idle cost | Per-instance-hour (Spot = 60-90% savings) |
| Streaming | Glue Streaming (limited) | Spark Structured Streaming (full) |
| Interactive | Glue Studio notebooks | EMR Studio, Zeppelin, JupyterHub |
| Ecosystem | Glue-specific (DynamicFrame) | Full Hadoop ecosystem |

**One-liner**: Glue for standard serverless ETL; EMR when you need full cluster control, Spot instances, or non-Spark engines.

---

### Q2: Kinesis vs. MSK (Kafka) — When Do You Choose Each?

**Choose Kinesis when**: AWS-native, small team, < 200 MB/s, tight Lambda/Firehose integration, zero ops desired.

**Choose MSK when**: Existing Kafka producers/consumers, need Kafka Connect, KSQL, or Schema Registry, throughput > 200 MB/s, multi-cloud strategy, log compaction.

**One-liner**: If you're already on Kafka or need the ecosystem → MSK. Green-field AWS project → Kinesis (unless throughput is massive).

---

### Q3: Athena vs. Redshift — When Do You Choose Each?

**Choose Athena when**: Ad-hoc queries on S3, no infrastructure commitment, data exploration, infrequent queries, cost = per-query.

**Choose Redshift when**: Predictable BI workloads, complex multi-table joins, sub-second dashboard latency, materialized views, streaming ingestion.

**One-liner**: Athena = serverless SQL scratchpad for S3. Redshift = production data warehouse for BI.

---

### Q4: Step Functions vs. MWAA (Airflow)

**Choose Step Functions when**: All-AWS pipeline, event-driven triggers, visual debugging, serverless, low-medium complexity.

**Choose MWAA when**: Existing Airflow DAGs, Python-first orchestration, complex scheduling/sensors/backfill, non-AWS integrations, dynamic DAG generation.

**One-liner**: Step Functions for AWS-native; Airflow for Python-native and cross-platform.

---

### Q5: Lake Formation vs. IAM for Data Access

**Choose Lake Formation when**: Column/row-level security needed, 50+ tables, multiple consuming teams, cross-account data sharing, PII compliance.

**Choose IAM-only when**: Simple setup (< 10 tables), path-level access is sufficient, no PII concerns, single-account environment.

**One-liner**: IAM = path-level (S3 prefixes). Lake Formation = table/column/row-level (semantic).

---

### Q6: S3 vs. EBS vs. EFS for Data Pipelines

| Factor | S3 | EBS | EFS |
|---|---|---|---|
| Access pattern | Object (HTTP API) | Block (mounted to EC2) | File (NFS, mounted) |
| Sharing | Multi-consumer | Single EC2 (or multi-attach) | Multi-EC2, multi-AZ |
| Cost | $0.023/GB/mo | $0.08-0.10/GB/mo | $0.30/GB/mo |
| Throughput | Very high (prefix-parallel) | Provisioned IOPS | Bursting/provisioned |
| Data lake use | Primary storage | EMR local shuffle | Shared config/code |

**One-liner**: S3 for everything in a data lake. EBS for EMR instance store. EFS only for shared file-system needs (rare in data pipelines).

---

### Q7: Glue Crawlers vs. Manual Schema Definition

**Choose Crawlers when**: New/unknown data sources, exploring raw data, initial schema discovery, CSV/JSON with predictable structure.

**Choose Manual Definition when**: Production tables with controlled schemas, Iceberg/Delta Lake tables, when crawlers produce incorrect types, CI/CD-managed schemas.

**One-liner**: Crawlers for discovery, manual for production control.

---

### Q8: Firehose vs. Lambda Consumer for Kinesis

**Choose Firehose when**: Destination is S3, Redshift, OpenSearch, or HTTP endpoint. You want managed batching, compression, and format conversion.

**Choose Lambda consumer when**: Per-record processing logic (enrichment, filtering, API calls). Low-latency response needed (< 1s). Complex branching logic per record.

**One-liner**: Firehose = managed bulk delivery. Lambda = per-record processing.

---

## Screen 3: Cost & Security — "How Do You Keep It Cheap and Safe?"

---

### Q1: "S3 costs are spiraling. How do you reduce them?"

1. **Lifecycle policies**: Automate tier transitions (Standard → IA → Glacier → Deep Archive)
2. **Delete old versions**: Noncurrent version expiration (keep last 3, delete after 30 days)
3. **Compact small files**: Reduce request costs (PUT/GET charges dominate for millions of tiny files)
4. **S3 Storage Lens**: Identify unused prefixes, stale data, and tier opportunities
5. **Requester Pays**: If external teams query your data, they pay data transfer
6. **S3 Intelligent-Tiering**: For unpredictable access patterns (large objects only)
7. **Clean up incomplete multipart uploads**: `AbortIncompleteMultipartUpload` in lifecycle rules

---

### Q2: "Athena costs are $5,000/month. Reduce by 80%."

1. Convert CSV/JSON → **Parquet with Snappy** (90% scan reduction)
2. Add **date partitioning** (WHERE year=X, month=Y prunes entire directories)
3. Use **partition projection** (avoid MSCK REPAIR TABLE, instant query start)
4. Set **workgroup scan limits** (BytesScannedCutoffPerQuery)
5. Use **CTAS** to materialize frequently-run queries as pre-aggregated tables
6. **Column pruning**: SELECT only needed columns, never `SELECT *`
7. **Athena result caching**: Re-use cached results for identical queries (within same workgroup)

---

### Q3: "How do you prevent a junior engineer from exposing PII?"

```
Defense in depth:
  Layer 1: Lake Formation column-level security
           → PII columns excluded from analyst grants

  Layer 2: S3 Block Public Access (account-level)
           → No bucket can ever be public, even by accident

  Layer 3: VPC endpoints for S3
           → Data never traverses the internet

  Layer 4: KMS encryption
           → Even if data leaks, it's encrypted without the key

  Layer 5: SCPs (Service Control Policies)
           → Prevent disabling encryption, block public access changes

  Layer 6: AWS Config rules
           → Alert on non-compliant buckets within 1 hour

  Layer 7: CloudTrail + Athena
           → Audit who accessed what, query access logs

  Layer 8: Macie
           → ML-based PII detection in S3 data (automated scanning)
```

---

### Q4: "Your Redshift cluster costs $8,000/month and is 70% idle. What do you do?"

1. **Migrate to Redshift Serverless** → pay only for active query time → ~$2,400/month
2. If staying provisioned: **Reserved Instances** (1-year RI = 35% savings, 3-year = 75%)
3. **Pause cluster** during off-hours (evenings/weekends) → 60% savings on compute
4. **Concurrency Scaling**: Free 1 hour/day, use instead of over-provisioning
5. **Materialized views**: Pre-compute heavy queries to reduce runtime RPU consumption
6. **WLM queues**: Prioritize critical queries, throttle expensive ad-hoc queries

---

### Q5: "A security audit finds that 3 IAM roles have `s3:*` on `*`. How do you remediate?"

1. **IAM Access Analyzer**: Identify which S3 actions each role actually uses
2. **CloudTrail analysis**: Query 90 days of API history to find actual usage patterns
3. **Generate least-privilege policy**: Use IAM Access Analyzer's policy generation
4. **Apply scoped policies**: Replace `s3:*` with specific actions on specific ARNs
5. **Test with IAM Policy Simulator**: Verify no breakage before applying
6. **Add a Deny policy**: Immediately deny `s3:DeleteBucket`, `s3:PutBucketPolicy` on critical buckets
7. **SCP guardrail**: At the org level, deny `s3:*` on `*` for all non-admin roles

---

### Q6: "How do you implement data encryption for a multi-account data mesh?"

```
Central KMS strategy:
  • Shared Services account owns KMS multi-region keys (MRKs)
  • Each domain account has a KMS grant for Encrypt/Decrypt
  • Key policy condition: kms:ViaService = s3.*.amazonaws.com

Per-domain option:
  • Each domain account creates its own CMK
  • Cross-account consumers get kms:Decrypt via key policy
  • Central governance account audits key policies via AWS Config

Recommendation: Central keys for shared data, per-domain keys for domain-owned data.
  • Central key = easier cross-domain access, single rotation policy
  • Per-domain key = stronger isolation, domain controls their own key

All buckets: SSE-KMS with Bucket Key enabled (reduce KMS API costs by 99%)
All buckets: Deny unencrypted uploads via bucket policy
```

---

### Q7: "An S3 bucket was accidentally made public for 2 hours. Incident response?"

1. **Immediate**: Reapply S3 Block Public Access (account + bucket level)
2. **Assess scope**: CloudTrail → check `PutBucketPolicy`/`PutBucketAcl` events to find who/when
3. **Data exposure**: S3 Server Access Logs → identify public IPs that accessed the bucket
4. **Sensitive data check**: Macie scan → were any PII objects downloaded?
5. **Revoke remaining access**: Review bucket policy, ACLs, Access Points for residual exposure
6. **Preventive**: Add SCP to deny `s3:PutBucketPolicy` that includes `"Principal": "*"`
7. **Detective**: AWS Config rule `s3-bucket-public-read-prohibited` with auto-remediation Lambda
8. **Report**: Document timeline, affected data, and remediation steps for compliance

---

### Q8: "How do you monitor and control costs across a data platform?"

```
AWS Cost Allocation Tags:
  • Tag everything: Project, Team, Environment, CostCenter
  • Activate tags in Billing → Cost Explorer shows per-team breakdown

Budgets:
  • AWS Budgets per team/service with email + SNS alerts at 80%/100%
  • Forecasting: AWS Budget Actions auto-stop non-critical resources

Controls:
  • Athena workgroups: BytesScannedCutoffPerQuery
  • Redshift Serverless: Usage limits (max RPU-hours per day)
  • Glue: MaxConcurrentRuns to prevent runaway parallel jobs
  • EMR: Managed scaling with min/max constraints + Spot fleet

Visibility:
  • S3 Storage Lens: Dashboard for storage patterns and cost optimization
  • Cost Explorer: Weekly review of top-5 cost drivers
  • Trusted Advisor: S3 bucket lifecycle recommendations
```

---

## Screen 4: Debugging & Operations — "It's Broken, Fix It"

---

### Q1: "The nightly Glue ETL job failed at 2 AM. Walk me through debugging."

1. **CloudWatch Logs**: Check `/aws-glue/jobs/error` log group for stack trace
2. **Glue console**: Job run details → error message, driver/executor logs
3. **Common errors**:
   - `OutOfMemoryError`: Worker type too small → scale to G.2X or reduce `groupSize`
   - `S3 Access Denied`: IAM role, bucket policy, or KMS key issue
   - `Schema mismatch`: Source schema changed → update crawler or use `resolveChoice`
   - `ConcurrentRunsExceededException`: Previous run still active → check `MaxConcurrentRuns`
4. **Spark UI** (if enabled): Check for data skew, straggler tasks, shuffle read/write
5. **Job bookmark**: If bookmark corrupted, reset and rerun with `job-bookmark-pause`
6. **Fix and re-run**: Apply fix, test in dev, re-run with `--job-bookmark-option job-bookmark-pause` to reprocess today's partition only

---

### Q2: "Athena query returns zero results but data exists in S3."

1. **Partition registered?**: `SHOW PARTITIONS table_name` — is the partition listed?
2. **Partition projection**: If enabled, check `storage.location.template` matches actual S3 path
3. **S3 path mismatch**: `aws s3 ls s3://bucket/prefix/` — does the path structure match the table definition?
4. **Hive partition format**: Paths must be `year=2026/month=04/day=05/` not `2026/04/05/`
5. **Case sensitivity**: Athena is case-insensitive for SQL but S3 keys are case-sensitive
6. **File format mismatch**: Table says PARQUET but files are actually JSON
7. **Empty files**: Files exist but have 0 bytes (failed ETL wrote empty partitions)
8. **Fix**: `MSCK REPAIR TABLE` or `ALTER TABLE ADD PARTITION` for the missing partition

---

### Q3: "Kinesis consumer is falling behind — iterator age is growing."

1. **Check CloudWatch**: `GetRecords.IteratorAgeMilliseconds` > acceptable threshold
2. **Cause: insufficient throughput**: More data coming in than consumer can process
3. **Fix options**:
   - **Scale shards**: `UpdateShardCount` (doubles capacity, takes minutes)
   - **Enhanced Fan-Out**: If multiple consumers compete for 2 MB/s shared throughput
   - **Increase consumer parallelism**: More KCL workers (1 worker per shard)
   - **Batch size**: Process more records per GetRecords call (up to 10,000)
   - **Reduce per-record processing time**: Profile the consumer Lambda/application
4. **If Lambda consumer**: Increase `BatchSize`, `ParallelizationFactor`, memory
5. **On-Demand mode**: If shard management is the bottleneck, switch to On-Demand

---

### Q4: "Redshift COPY command is slow — loading 100 GB takes 2 hours."

1. **File count and size**: COPY is parallel — one file per slice per node. Too few large files = under-utilization. Split into `number_of_slices` files.
2. **Compression**: Source files should be compressed (GZIP, LZO, ZSTD). Reduces I/O by 3-5x.
3. **File format**: Parquet/ORC > CSV (columnar reads, type-safe, compressed).
4. **COPY options**: `COMPUPDATE OFF` and `STATUPDATE OFF` for bulk loads (run ANALYZE after).
5. **Distribution**: Check for skew — if DISTKEY is skewed, one node does most of the work.
6. **Network**: Is Redshift in the same region as S3? Cross-region COPY = slow.
7. **Manifest file**: Use a manifest for explicit file listing (avoids S3 ListObjects).
8. **Concurrency**: Reduce concurrent queries during COPY (WLM queue allocation).

---

### Q5: "Step Functions execution shows 'States.Permissions' error on a Glue state."

1. **Root cause**: The Step Functions execution role lacks `iam:PassRole` for the Glue job's role
2. **Fix**: Add `iam:PassRole` to the execution role with a condition:
   ```json
   {
     "Effect": "Allow",
     "Action": "iam:PassRole",
     "Resource": "arn:aws:iam::123456789:role/GlueETLRole",
     "Condition": {
       "StringEquals": {
         "iam:PassedToService": "glue.amazonaws.com"
       }
     }
   }
   ```
3. **Also check**: `glue:StartJobRun` permission on the specific job ARN
4. **Common pattern**: Step Functions → needs `PassRole` to the Glue role + `StartJobRun` on the job

---

### Q6: "S3 PUT requests are returning 503 SlowDown errors."

1. **Cause**: Exceeding 3,500 PUT/sec on a single S3 prefix
2. **Diagnosis**: S3 Request Metrics in CloudWatch → check `5xxErrors` per prefix
3. **Fix**:
   - Distribute writes across more prefixes (hash prefixes, add randomness)
   - Add exponential backoff retry in the writer application
   - If using multipart upload, reduce concurrency per prefix
4. **S3 auto-partitions**: S3 automatically scales prefix throughput, but new prefixes start at 3,500/sec and ramp up. Pre-warm by gradually increasing load.

---

### Q7: "Lake Formation permissions were granted but the analyst still gets 'Access Denied' in Athena."

1. **IAMAllowedPrincipals still active?**: The most common issue — legacy wildcard grant bypasses LF. Check and revoke.
2. **S3 location registered?**: `DescribeResource` — is the S3 path registered with Lake Formation?
3. **Grant path**: Was the grant made to the correct IAM role ARN? (Not the user ARN)
4. **Database + Table permissions**: Need DESCRIBE on database AND SELECT on table
5. **Athena workgroup**: Does the workgroup have permission to the output S3 bucket?
6. **Column-level grant**: If only specific columns were granted, `SELECT *` fails — must select only granted columns

---

### Q8: "A Glue Crawler is creating new tables instead of updating the existing one."

1. **Schema change**: Crawler detects incompatible schema → creates new table with suffix
2. **Partition structure changed**: New folders don't match expected partition scheme
3. **Fix**:
   - Set `UpdateBehavior = UPDATE_IN_DATABASE` in SchemaChangePolicy
   - Use a **table prefix** to keep crawled tables grouped
   - Ensure consistent partition structure in S3
   - Set `Configuration` with `Grouping.TableGroupingPolicy` to merge
4. **Alternative**: Skip crawlers for this table, use `ALTER TABLE ADD PARTITION` manually or via Glue API in your ETL job

---

## Screen 5: Rapid Fire — 20 Quick Q&A

**Answer each in ≤ 3 sentences.**

---

**1. What's the max record size in Kinesis Data Streams?**
1 MB per record. For larger payloads, store the data in S3 and send the S3 key as the Kinesis record (claim-check pattern).

**2. What's the difference between COMPOUND and INTERLEAVED sort keys in Redshift?**
COMPOUND gives priority to the first key column (fast for queries filtering on the leading column). INTERLEAVED gives equal weight to all key columns (better for ad-hoc, but VACUUM REINDEX is expensive).

**3. Can Firehose write directly to Redshift?**
Yes. Firehose stages data in S3 first, then issues a COPY command to Redshift automatically. You configure the Redshift cluster, database, table, and COPY options in the delivery stream.

**4. What's the minimum number of DPUs for a Glue Spark job?**
2 DPUs (Glue 4.0). Each DPU = 4 vCPU + 16 GB RAM. For Python Shell jobs, it's 1/16 DPU (0.0625 DPU).

**5. How does Athena charge for queries?**
$5 per TB of data scanned. Cancelled queries are charged for data scanned before cancellation. Cached results are free (within the same workgroup, same query, within the cache TTL).

**6. What's Redshift Concurrency Scaling?**
Automatically spins up additional cluster capacity when queries queue. The first 1 hour of concurrency scaling per day per cluster is free. Beyond that, per-second billing.

**7. What's the difference between Kinesis on-demand and provisioned mode?**
Provisioned: you manage shard count (1 MB/s in per shard). On-demand: auto-scales up to 200 MB/s, priced per GB in/out. On-demand is ~15-20% more expensive at steady state but eliminates shard management.

**8. What's a Redshift materialized view?**
A pre-computed query result stored on disk. Refreshed manually or auto-refreshed. Dramatically faster for dashboard queries. Supports incremental refresh for append-only base tables.

**9. How do you handle schema evolution in Glue?**
Crawlers can detect new columns and update the catalog. For production control, use the Glue Schema Registry (Avro/JSON with compatibility modes: BACKWARD, FORWARD, FULL). Iceberg tables handle schema evolution natively.

**10. What's the difference between S3 Event Notifications and EventBridge for S3?**
S3 Event Notifications: free, simple prefix/suffix filtering, targets limited to Lambda/SQS/SNS. EventBridge: $1/million events, content-based filtering on event JSON, 20+ targets, event archive/replay.

**11. Can Redshift Spectrum write to S3?**
No. Spectrum is read-only. To write query results to S3, use `UNLOAD` from Redshift or CTAS in Athena.

**12. What's the difference between Glue Job Bookmarks and Iceberg?**
Bookmarks track which files were processed (for incremental reads). Iceberg is a table format that supports ACID, time travel, and row-level mutations. They solve different problems — bookmarks for ETL state, Iceberg for table management.

**13. What's a Kinesis shard split?**
Dividing one shard into two to increase throughput. The parent shard stops accepting writes; two child shards take over. Shard merge is the reverse. Both take minutes and are disruptive.

**14. What's AQUA in Redshift?**
Advanced Query Accelerator — a hardware-accelerated cache layer on RA3 nodes. Pushes scan and aggregation to the storage layer, reducing data movement. Available only on RA3.4xl and RA3.16xl.

**15. How do you secure Kinesis data at rest?**
Server-side encryption with KMS (SSE-KMS). Enable on the stream: `aws kinesis start-stream-encryption --stream-name X --encryption-type KMS --key-id alias/aws/kinesis`. Data is encrypted before writing to disk.

**16. What's the max concurrent Step Functions Standard executions?**
1,000,000 open executions per account per region (soft limit). Express workflows: 100,000 start rate per second. Standard workflows are long-lived (up to 1 year).

**17. What's the difference between Redshift RA3 and DC2 node types?**
DC2: compute + local SSD storage (tightly coupled). RA3: compute + managed storage (S3-backed with SSD cache). RA3 scales compute and storage independently — the modern choice.

**18. Can Lake Formation work with EMR?**
Yes. EMR clusters integrate with Lake Formation via the instance role. LF vends temporary credentials scoped to the granted tables/columns. Requires EMR 5.31+ and EMRFS integration.

**19. What's Kinesis Video Streams?**
A separate service for streaming video/audio for ML processing (not data engineering). Don't confuse it with Kinesis Data Streams — they share the name but are entirely different services.

**20. What happens when you DELETE from an Iceberg table on Athena?**
Athena writes a delete file (positional or equality) that marks rows as deleted. The original data files are untouched. When the table is read, delete files are applied to filter out deleted rows. Run `OPTIMIZE` to compact and physically remove deleted data.

---

## Screen 6: Key Takeaways & Study Strategy

### The 80/20 Rule for AWS Data Engineering Interviews

These topics cover 80% of questions:

```
MUST KNOW COLD:
  ✓ S3 storage classes, lifecycle, encryption (SSE-KMS vs SSE-S3)
  ✓ Glue ETL: DynamicFrame, resolveChoice, bookmarks, job types
  ✓ Athena: partitioning, Parquet, cost optimization, partition projection
  ✓ Kinesis: shards, throughput limits, Firehose vs Streams vs Analytics
  ✓ Redshift: distribution styles (KEY/EVEN/ALL), sort keys, Spectrum, COPY
  ✓ Lake Formation: column/row security, LF-Tags, cross-account sharing
  ✓ Step Functions: state types, retry/catch, Standard vs Express
  ✓ IAM: service roles, cross-account assume role, least privilege

SHOULD KNOW WELL:
  ○ EMR: instance fleets, Spot vs On-Demand, EMRFS consistency
  ○ Iceberg/Delta Lake on Athena/Glue (MERGE, time travel)
  ○ Kinesis Data Analytics (Flink): windows, watermarks
  ○ Redshift Serverless vs provisioned trade-offs
  ○ EventBridge vs S3 Event Notifications
  ○ Glue Schema Registry compatibility modes
  ○ S3 Access Points and Object Lambda

NICE TO HAVE:
  △ Redshift Streaming Ingestion
  △ Redshift Data Sharing cross-account
  △ DMS for CDC
  △ MSK (Kafka) specifics
  △ AQUA, Concurrency Scaling internals
```

### Study Strategy

```
Week 1: Foundations (Modules 1-2)
  • S3 + Glue + Athena — these appear in EVERY interview
  • Build: Create a bucket, lifecycle, run a Glue ETL job, query with Athena
  • Practice: Explain the cost of a SELECT * on CSV vs Parquet (with numbers)

Week 2: Compute & Streaming (Modules 3-4)
  • EMR + Kinesis + Redshift
  • Build: Put records into Kinesis, consume with Lambda, deliver with Firehose
  • Practice: Design a real-time pipeline on a whiteboard (draw the diagram)

Week 3: Governance & Orchestration (Module 5)
  • Lake Formation + Step Functions + IAM
  • Build: Set up LF permissions, create a 3-state Step Functions workflow
  • Practice: Debug an "Access Denied" scenario (walk through the IAM chain)

Week 4: Mock Interviews (Module 6)
  • Do this module's questions aloud with a timer (2 min per answer)
  • Record yourself — listen for filler words and vague answers
  • Get a peer to drill you on "why" follow-ups

Daily habit:
  • Read one AWS service FAQ (15 min)
  • Draw one architecture diagram from memory (10 min)
  • Answer 3 rapid-fire questions aloud (5 min)
```

### Interview Answer Framework: STAR-T

For every architecture/design question, use this structure:

```
S — Situation: Restate the problem in your own words (shows listening)
T — Technology: Name the AWS services and justify each choice
A — Architecture: Draw/describe the data flow (ingestion → storage → processing → serving)
R — Risks: Proactively mention failure modes and mitigations
T — Trade-offs: Acknowledge what you're sacrificing (cost vs latency, simplicity vs flexibility)

Example:
  "So you need real-time fraud detection under 10 seconds [S].
   I'd use Kinesis Data Streams for ingestion and Flink for windowed analysis [T].
   The flow is POS → Kinesis (partitioned by card hash) → Flink (5-min sliding window) → Alert stream → Lambda → PagerDuty [A].
   Risk: if Flink falls behind, iterator age grows — we'd monitor that and auto-scale [R].
   Trade-off: Flink adds complexity vs a simpler Lambda consumer, but Lambda can't do stateful windowed aggregations [T]."
```

### Common Mistakes to Avoid

```
❌ "I would use Glue for everything"
   → Shows lack of understanding when EMR, Athena, or Redshift is better

❌ "We'd use Kinesis and Kafka"
   → Pick one and justify. Using both signals confusion.

❌ "Just use IAM policies"
   → For data lakes, you NEED Lake Formation for column/row security

❌ Forgetting cost in your design
   → Always mention: lifecycle policies, Spot instances, Parquet compression,
     workgroup scan limits. Cost awareness = senior thinking.

❌ Designing without failure handling
   → Every pipeline must address: what happens when the job fails?
     What happens when data arrives late? How do you retry?

❌ Not asking clarifying questions
   → "What's the latency requirement?" "What's the data volume?"
     "Do you need exactly-once or at-least-once?"
     These show you think before you architect.
```

### 🔑 Final Key Takeaways

1. **Know the "why" not just the "what"** — Interviewers follow up with "why not X instead?" Be ready.
2. **Draw diagrams** — Even in a phone screen, describe the architecture as if you're drawing boxes and arrows. Say "data flows from A to B to C."
3. **Numbers matter** — "Kinesis supports 1 MB/s per shard" is better than "Kinesis supports a lot of throughput."
4. **Cost is a first-class design criterion** — Every architecture decision has a cost implication. Mention it proactively.
5. **Security is not an afterthought** — Encryption, least privilege, PII controls — weave these into your design, don't bolt them on at the end.
6. **Failure handling differentiates seniors from juniors** — Retry, dead-letter queues, alerting, idempotency — these show production experience.
7. **The best answer acknowledges trade-offs** — "I chose Kinesis over MSK because the team is small and we don't need the Kafka ecosystem, but if we needed log compaction or 500+ MB/s throughput, I'd reconsider."
8. **Practice aloud** — Reading answers is not the same as articulating them. Practice with a peer or record yourself.
