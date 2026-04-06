# Module 1: Amazon S3 — The Foundation of Data Lakes

> **Scenario**: RetailMart, a national retail chain with 4,000+ stores, is building a cloud-native data lake on AWS. Every day, 2.3 TB of point-of-sale transactions, inventory snapshots, clickstream logs, and supplier feeds land in Amazon S3. Your job is to architect the storage layer that underpins every downstream analytics pipeline.

---

## Screen 1: S3 Storage Classes — Picking the Right Tier

Amazon S3 offers a spectrum of storage classes that trade **retrieval speed** for **cost per GB**. Choosing correctly is the difference between a $4,000/month bill and a $40,000/month bill for the same data.

### Storage Class Comparison

| Storage Class | Use Case | Min Duration | Retrieval | $/GB/mo (approx) |
|---|---|---|---|---|
| S3 Standard | Hot data, frequent access | None | Instant | $0.023 |
| S3 Intelligent-Tiering | Unknown access patterns | 30 days | Instant | $0.023 → $0.0125 |
| S3 Standard-IA | Infrequent but rapid access | 30 days | Instant | $0.0125 |
| S3 One Zone-IA | Reproducible infrequent data | 30 days | Instant | $0.010 |
| S3 Glacier Instant | Archive, millisecond access | 90 days | Instant | $0.004 |
| S3 Glacier Flexible | Archive, minutes-to-hours | 90 days | 1-12 hours | $0.0036 |
| S3 Glacier Deep Archive | Long-term compliance | 180 days | 12-48 hours | $0.00099 |

### RetailMart Data Tiering Strategy

```
┌─────────────────────────────────────────────────────────────┐
│                   RetailMart S3 Data Lake                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  HOT (0-30 days)          ┌──────────────────────┐          │
│  S3 Standard              │  Daily POS txns       │         │
│  ~120 TB                  │  Clickstream logs     │         │
│                           │  Real-time inventory  │         │
│                           └──────────────────────┘          │
│                                                             │
│  WARM (30-90 days)        ┌──────────────────────┐          │
│  S3 Standard-IA           │  Monthly aggregates   │         │
│  ~400 TB                  │  Historical queries   │         │
│                           │  ML training sets     │         │
│                           └──────────────────────┘          │
│                                                             │
│  COLD (90-365 days)       ┌──────────────────────┐          │
│  Glacier Instant          │  Quarterly reports    │         │
│  ~1.2 PB                  │  Audit data           │         │
│                           └──────────────────────┘          │
│                                                             │
│  FROZEN (1-7 years)       ┌──────────────────────┐          │
│  Glacier Deep Archive     │  Regulatory/tax data  │         │
│  ~3.5 PB                  │  Legal hold records   │         │
│                           └──────────────────────┘          │
└─────────────────────────────────────────────────────────────┘
```

### 💡 Interview Insight

> **Q: "When would you pick Intelligent-Tiering over manually setting Standard-IA?"**
>
> Use Intelligent-Tiering when access patterns are **unpredictable**. It automatically moves objects between tiers with zero retrieval fees and no operational overhead. The monitoring fee is $0.0025/1,000 objects/month — negligible for large objects but expensive for millions of tiny files. For RetailMart's clickstream (billions of small JSON files), manual lifecycle rules to Standard-IA are cheaper. For ad-hoc analyst uploads with unknown access frequency, Intelligent-Tiering wins.

---

## Screen 2: Lifecycle Policies — Automating Data Movement

Lifecycle policies let you define **rules** that automatically transition objects between storage classes or expire (delete) them after a set period. This is how you avoid the #1 S3 cost mistake: forgetting data exists.

### Configuring a Lifecycle Policy (JSON)

```json
{
  "Rules": [
    {
      "ID": "retailmart-pos-lifecycle",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "raw/pos-transactions/"
      },
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 90,
          "StorageClass": "GLACIER_IR"
        },
        {
          "Days": 365,
          "StorageClass": "DEEP_ARCHIVE"
        }
      ],
      "Expiration": {
        "Days": 2555
      },
      "NoncurrentVersionTransitions": [
        {
          "NoncurrentDays": 7,
          "StorageClass": "GLACIER"
        }
      ],
      "NoncurrentVersionExpiration": {
        "NoncurrentDays": 90
      }
    }
  ]
}
```

### Applying the Lifecycle Policy via AWS CLI

```bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket retailmart-data-lake-prod \
  --lifecycle-configuration file://lifecycle-policy.json
```

### Key Transition Rules

```
Day 0          Day 30           Day 90            Day 365        Day 2555
  │              │                │                  │               │
  ▼              ▼                ▼                  ▼               ▼
Standard ──► Standard-IA ──► Glacier Instant ──► Deep Archive ──► DELETE
                                                                     │
                                                              7-year retention
                                                              (tax compliance)
```

**Important constraints:**
- You cannot transition **from** a colder class **to** a warmer one via lifecycle.
- Minimum object size for IA classes is **128 KB** (smaller objects are billed as 128 KB).
- Minimum duration charges apply — moving out before the period incurs a pro-rata fee.
- Transitions only go in one direction: Standard → IA → Glacier → Deep Archive.

### 💡 Interview Insight

> **Q: "How do you handle the 128 KB minimum for Standard-IA when you have millions of small log files?"**
>
> Two strategies: (1) **Aggregate before transitioning** — use a Glue ETL job or Lambda to compact small files into larger Parquet files, then lifecycle the compacted objects. (2) **Use Intelligent-Tiering** instead of IA, as it has no minimum object size charge. At RetailMart, we compact raw JSON clickstream into 128 MB Parquet files daily, then apply lifecycle rules to the compacted prefix.

---

## Screen 3: Event Notifications — Triggering Pipelines on File Arrival

S3 Event Notifications are the **nervous system** of an event-driven data lake. When a file lands, S3 can fire a notification to Lambda, SQS, SNS, or EventBridge — kicking off ETL, validation, or alerting.

### Event-Driven Architecture for RetailMart

```
                          ┌─────────────────────────────────────────────┐
                          │            S3 Event Notifications           │
                          └─────────────┬───────────────────────────────┘
                                        │
   ┌────────────────────────────────────┼────────────────────────────────┐
   │                                    │                                │
   ▼                                    ▼                                ▼
┌──────────┐                     ┌──────────┐                     ┌──────────┐
│  Lambda   │                     │   SQS    │                     │   SNS    │
│ Validator │                     │  Queue   │                     │  Topic   │
└────┬─────┘                     └────┬─────┘                     └────┬─────┘
     │                                │                                │
     ▼                                ▼                                ▼
 Validates                      Glue ETL job                    Email alert
 schema +                       picks up files                  to data ops
 file size                      in batches                      on failure
     │                                │
     ▼                                ▼
 Moves to                       Writes to
 processed/                     curated/
 or quarantine/                 (Parquet)
```

### Configuring S3 → SQS Notification (AWS CLI)

```bash
# Create the SQS queue
aws sqs create-queue \
  --queue-name retailmart-raw-file-arrivals \
  --attributes '{
    "VisibilityTimeout": "300",
    "MessageRetentionPeriod": "86400"
  }'

# Configure S3 event notification
aws s3api put-bucket-notification-configuration \
  --bucket retailmart-data-lake-prod \
  --notification-configuration '{
    "QueueConfigurations": [
      {
        "QueueArn": "arn:aws:sqs:us-east-1:123456789:retailmart-raw-file-arrivals",
        "Events": ["s3:ObjectCreated:*"],
        "Filter": {
          "Key": {
            "FilterRules": [
              {"Name": "prefix", "Value": "raw/pos-transactions/"},
              {"Name": "suffix", "Value": ".parquet"}
            ]
          }
        }
      }
    ]
  }'
```

### EventBridge vs. Native S3 Notifications

| Feature | S3 Notifications | EventBridge |
|---|---|---|
| Targets | Lambda, SQS, SNS | 20+ services |
| Filtering | Prefix + suffix only | JSON content filtering |
| Fan-out | 1 target per event type | Multiple rules per event |
| Replay | No | Yes (event archive) |
| Cost | Free | $1/million events |

### 💡 Interview Insight

> **Q: "You need to trigger both a Lambda validator AND a Glue ETL job when a file lands. How?"**
>
> Option A: S3 → **SNS** (fan-out) → Lambda + SQS (Glue polls SQS). Option B: S3 → **EventBridge** → multiple targets directly. EventBridge is the modern answer — it supports content-based filtering, can target Step Functions, and has event replay. At RetailMart, we use EventBridge because we also route events to a dead-letter queue and CloudWatch for observability.

---

## Screen 4: S3 Select & Glacier Select — Query in Place

S3 Select lets you run **SQL expressions** directly against objects in S3 without downloading the entire file. This is transformative when you need a needle from a haystack.

### S3 Select Example: Extracting High-Value Transactions

```bash
aws s3api select-object-content \
  --bucket retailmart-data-lake-prod \
  --key "raw/pos-transactions/2026/04/05/store-0042.csv" \
  --expression "SELECT s.transaction_id, s.total_amount
                FROM s3object s
                WHERE CAST(s.total_amount AS FLOAT) > 500.00" \
  --expression-type SQL \
  --input-serialization '{"CSV": {"FileHeaderInfo": "USE"}}' \
  --output-serialization '{"JSON": {}}' \
  output.json
```

### Python SDK (boto3) — S3 Select with Parquet

```python
import boto3
import json

s3 = boto3.client("s3")

response = s3.select_object_content(
    Bucket="retailmart-data-lake-prod",
    Key="curated/daily-sales/2026/04/05/summary.parquet",
    Expression="""
        SELECT store_id, SUM(total_amount) AS revenue
        FROM s3object s
        WHERE category = 'Electronics'
        GROUP BY store_id
    """,
    ExpressionType="SQL",
    InputSerialization={"Parquet": {}},
    OutputSerialization={"JSON": {}},
)

for event in response["Payload"]:
    if "Records" in event:
        print(event["Records"]["Payload"].decode("utf-8"))
```

### When to Use S3 Select vs. Athena

| Criteria | S3 Select | Athena |
|---|---|---|
| Scope | Single object | Entire dataset |
| Query complexity | Simple filter/project | Complex joins, CTEs |
| Cost model | Per byte scanned | $5/TB scanned |
| Latency | Milliseconds | Seconds |
| Best for | Lambda preprocessing | Ad-hoc analytics |

**Glacier Select** works similarly but on Glacier Flexible Retrieval. Queries are queued and results delivered to an S3 staging bucket. Useful for compliance queries against archived data without restoring entire archives.

### How S3 Select Reduces Costs

Consider RetailMart's daily reconciliation process: a Lambda function fires when a 2 GB CSV lands in S3. Without S3 Select, Lambda downloads the entire 2 GB file (takes ~20 seconds, requires 2 GB memory = expensive Lambda). With S3 Select, Lambda sends a SQL filter and receives only matching rows (~5 MB). That's a 400x reduction in data transfer and a 10x reduction in Lambda memory cost.

```
Without S3 Select:                With S3 Select:
┌──────────┐   2 GB    ┌──────┐   ┌──────────┐  SQL query  ┌──────┐
│  S3      │──────────►│Lambda│   │  S3      │────────────►│Lambda│
│  (2 GB)  │           │(2 GB │   │  (2 GB)  │  5 MB resp  │(256MB│
│          │           │ mem) │   │          │◄────────────│ mem) │
└──────────┘           └──────┘   └──────────┘             └──────┘

Cost per invocation:              Cost per invocation:
  Data: $0.0004                     Data: $0.000001
  Lambda: $0.033                    Lambda: $0.004
  Total: ~$0.033                    Total: ~$0.004 (88% savings)
```

### 💡 Interview Insight

> **Q: "Why not always use S3 Select instead of Athena?"**
>
> S3 Select operates on a **single object** — no joins, no cross-file aggregation, no catalog integration. It's ideal inside a Lambda function for filtering one file before processing. Athena queries an entire **table** (thousands of objects) with full SQL. Think of S3 Select as `grep` and Athena as a full database engine.

---

## Screen 5: Versioning — Protecting Your Data Lake

S3 Versioning keeps **every version** of every object. Combined with MFA Delete, it's your last line of defense against accidental deletions and overwrites.

### Enabling Versioning

```bash
# Enable versioning on the bucket
aws s3api put-bucket-versioning \
  --bucket retailmart-data-lake-prod \
  --versioning-configuration Status=Enabled

# Enable MFA Delete (requires root account credentials)
aws s3api put-bucket-versioning \
  --bucket retailmart-data-lake-prod \
  --versioning-configuration Status=Enabled,MFADelete=Enabled \
  --mfa "arn:aws:iam::123456789:mfa/root-device 123456"
```

### How Versioning Works — Delete and Overwrite

```
Object: raw/pos-transactions/2026/04/05/store-0042.parquet

Timeline:
─────────────────────────────────────────────────────────
 t=1: PUT (v1)     ──► Version ID: abc111  ◄── Original upload
 t=2: PUT (v2)     ──► Version ID: abc222  ◄── Schema change re-upload
 t=3: DELETE        ──► Delete Marker added ◄── "Deleted" (but v1, v2 remain)
 t=4: DELETE v-abc222 ► Permanently removes v2

Listing versions:
  aws s3api list-object-versions \
    --bucket retailmart-data-lake-prod \
    --prefix "raw/pos-transactions/2026/04/05/store-0042.parquet"

Restoring a "deleted" object:
  aws s3api delete-object \
    --bucket retailmart-data-lake-prod \
    --key "raw/pos-transactions/2026/04/05/store-0042.parquet" \
    --version-id "DELETE_MARKER_VERSION_ID"
```

### Versioning Cost Implications

Every version is a **full copy** billed at the object's storage class rate. Without lifecycle rules for noncurrent versions, storage costs grow unbounded. Always pair versioning with noncurrent version expiration:

```json
{
  "NoncurrentVersionExpiration": {
    "NoncurrentDays": 30,
    "NewerNoncurrentVersions": 3
  }
}
```

This keeps the **3 most recent noncurrent versions** and deletes anything older than 30 days — a balance between safety and cost.

### 💡 Interview Insight

> **Q: "A junior engineer accidentally deleted the entire `curated/` prefix. How do you recover?"**
>
> If versioning is enabled, no data is lost — S3 added **delete markers** on top of every object. Recovery: (1) List all delete markers under `curated/`, (2) Delete each delete marker using its version ID, (3) The previous version becomes current again. Script it with `aws s3api list-object-versions --prefix curated/ | jq` and a batch delete loop. Prevention: enable MFA Delete and restrict `s3:DeleteObject` via IAM policy.

---

## Screen 6: Cross-Region Replication & Transfer Acceleration

### Cross-Region Replication (CRR)

CRR asynchronously copies objects from a source bucket to a destination bucket in a **different AWS region**. Use cases: disaster recovery, data sovereignty, latency reduction.

```
┌─────────────────┐     Async Replication      ┌─────────────────┐
│   us-east-1     │ ──────────────────────────► │   eu-west-1     │
│   (Primary)     │                             │   (DR Replica)  │
│                 │     ~15 min SLA (99.99%     │                 │
│ retailmart-     │      of objects in 15min)   │ retailmart-     │
│ data-lake-prod  │                             │ data-lake-dr    │
└─────────────────┘                             └─────────────────┘
      │                                               │
      │  Same-Region Replication (SRR)                │
      ▼                                               │
┌─────────────────┐                                   │
│   us-east-1     │◄──────────── Read replicas ───────┘
│   (Analytics)   │         for EU analysts
│ retailmart-     │
│ data-lake-anlyt │
└─────────────────┘
```

### Configuring CRR

```bash
# Both buckets must have versioning enabled
aws s3api put-bucket-replication \
  --bucket retailmart-data-lake-prod \
  --replication-configuration '{
    "Role": "arn:aws:iam::123456789:role/S3ReplicationRole",
    "Rules": [
      {
        "ID": "replicate-curated-to-eu",
        "Status": "Enabled",
        "Filter": {
          "Prefix": "curated/"
        },
        "Destination": {
          "Bucket": "arn:aws:s3:::retailmart-data-lake-dr",
          "StorageClass": "STANDARD_IA",
          "ReplicationTime": {
            "Status": "Enabled",
            "Time": {"Minutes": 15}
          },
          "Metrics": {
            "Status": "Enabled",
            "EventThreshold": {"Minutes": 15}
          }
        },
        "DeleteMarkerReplication": {
          "Status": "Enabled"
        }
      }
    ]
  }'
```

### S3 Transfer Acceleration

Uses **CloudFront edge locations** to accelerate uploads over long distances. RetailMart's stores in remote areas upload daily reconciliation files 40-60% faster.

```bash
# Enable Transfer Acceleration
aws s3api put-bucket-accelerate-configuration \
  --bucket retailmart-data-lake-prod \
  --accelerate-configuration Status=Enabled

# Upload using the accelerated endpoint
aws s3 cp store-recon-0042.parquet \
  s3://retailmart-data-lake-prod/raw/store-recon/ \
  --endpoint-url https://retailmart-data-lake-prod.s3-accelerate.amazonaws.com
```

### 💡 Interview Insight

> **Q: "Does CRR replicate existing objects?"**
>
> **No.** CRR only replicates objects uploaded **after** the rule is enabled. To replicate existing objects, use **S3 Batch Replication** — submit a batch operations job with a manifest of existing objects. This catches people off-guard in interviews. Also note: delete markers can optionally be replicated, but **version deletions** (permanent deletes) are NOT replicated — this is a safety feature to prevent cross-region cascade deletion.

---

## Screen 7: S3 Performance — Maximizing Throughput

S3 supports **5,500 GET/HEAD** and **3,500 PUT/POST/DELETE** requests per second **per prefix**. Understanding prefix design is critical for data lake performance.

### Prefix Design for RetailMart

```
❌ BAD — All files under one prefix (bottleneck):
  s3://retailmart/data/2026-04-05-store-0042-txn-0001.parquet
  s3://retailmart/data/2026-04-05-store-0042-txn-0002.parquet

✅ GOOD — Distributed across prefixes:
  s3://retailmart/raw/pos-transactions/2026/04/05/store=0042/part-0001.parquet
  s3://retailmart/raw/clickstream/2026/04/05/session=abc123/part-0001.json
  s3://retailmart/raw/inventory/2026/04/05/region=northeast/snapshot.parquet

Each unique prefix path gets its own 5,500/3,500 RPS budget.
```

### Multipart Upload

Objects > 100 MB **should** use multipart upload. Objects > 5 GB **must** use multipart upload.

```bash
# AWS CLI handles multipart automatically for large files
aws s3 cp large-dataset.parquet s3://retailmart-data-lake-prod/raw/ \
  --expected-size 5368709120

# Configure multipart thresholds
aws configure set default.s3.multipart_threshold 64MB
aws configure set default.s3.multipart_chunksize 16MB
aws configure set default.s3.max_concurrent_requests 20
```

### Python Multipart Upload with TransferConfig

```python
import boto3
from boto3.s3.transfer import TransferConfig

s3 = boto3.client("s3")

config = TransferConfig(
    multipart_threshold=64 * 1024 * 1024,   # 64 MB
    max_concurrency=20,
    multipart_chunksize=16 * 1024 * 1024,   # 16 MB
    use_threads=True,
)

s3.upload_file(
    "daily-pos-aggregate.parquet",
    "retailmart-data-lake-prod",
    "curated/daily-pos/2026/04/05/aggregate.parquet",
    Config=config,
)
```

### Byte-Range Fetches

Download only the bytes you need — ideal for Parquet files where you can read just the footer or specific row groups:

```python
# Read only the last 8 bytes (Parquet magic number check)
response = s3.get_object(
    Bucket="retailmart-data-lake-prod",
    Key="curated/daily-pos/2026/04/05/aggregate.parquet",
    Range="bytes=-8"
)
```

### 💡 Interview Insight

> **Q: "Your Spark job reading from S3 is slow. How do you diagnose and fix it?"**
>
> Diagnosis: (1) Check if all keys share the same prefix — fix with hash-prefixed or date-partitioned paths. (2) Check file sizes — thousands of tiny files cause "small file problem" (each file = one HTTP request). Fix by compacting into 128 MB–1 GB files. (3) Enable S3 request metrics in CloudWatch to measure 503 SlowDown errors. (4) Use `s3a://` connector with `fs.s3a.connection.maximum=100` in Spark config. (5) Enable S3 Transfer Acceleration if cross-region.

---

## Screen 8: S3 Access Points & Object Lambda

As RetailMart's data lake grows, managing bucket policies with dozens of conditional statements becomes unmanageable. **S3 Access Points** create named network endpoints with dedicated access policies — one per application or team.

### Access Point Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│            retailmart-data-lake-prod (single bucket)            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Access Point: ap-data-eng        Access Point: ap-analysts     │
│  ┌──────────────────────┐         ┌──────────────────────┐      │
│  │ Policy: Full R/W to  │         │ Policy: Read-only    │      │
│  │ raw/ and curated/    │         │ curated/ and         │      │
│  │ VPC: vpc-prod-123    │         │ analytics/ only      │      │
│  └──────────────────────┘         │ VPC: vpc-analytics   │      │
│                                   └──────────────────────┘      │
│                                                                 │
│  Access Point: ap-ml-team         Access Point: ap-vendor-feed  │
│  ┌──────────────────────┐         ┌──────────────────────┐      │
│  │ Policy: Read curated/│         │ Policy: Write-only   │      │
│  │ Write ml-features/   │         │ raw/supplier-feeds/  │      │
│  │ VPC: vpc-ml-123      │         │ Internet-facing      │      │
│  └──────────────────────┘         │ (with IAM auth)      │      │
│                                   └──────────────────────┘      │
└─────────────────────────────────────────────────────────────────┘
```

```bash
# Create a VPC-restricted access point for the analytics team
aws s3control create-access-point \
  --account-id 123456789012 \
  --name ap-analysts \
  --bucket retailmart-data-lake-prod \
  --vpc-configuration VpcId=vpc-analytics-456

# Use the access point ARN instead of the bucket name
aws s3 ls s3://arn:aws:s3:us-east-1:123456789012:accesspoint/ap-analysts/curated/
```

### S3 Object Lambda

Object Lambda lets you run a Lambda function to **transform data on the fly** as it's retrieved — no copy needed. RetailMart uses this to redact PII from customer records when the analytics team reads them:

```
Analyst GET request ──► S3 Object Lambda Access Point ──► Lambda (redact PII) ──► Return sanitized data
                                    │
                                    ▼
                           S3 (original data with PII intact)
```

This avoids maintaining two copies of the data (one with PII, one without) and ensures every read through the analytics access point is automatically sanitized.

### 💡 Interview Insight

> **Q: "How do you give 15 different teams access to the same S3 bucket without a 500-line bucket policy?"**
>
> **S3 Access Points.** Each team gets a dedicated access point with its own IAM policy, VPC restriction, and prefix scope. The bucket policy simply delegates to access points: `"Condition": {"StringEquals": {"s3:DataAccessPointAccount": "123456789012"}}`. This scales cleanly — adding a new team means creating a new access point, not editing a monolithic bucket policy. For cross-account access, you can even create access points in the consuming account.

---

## Screen 9: Encryption — Securing Data at Rest and in Transit

All data in RetailMart's lake contains PII (customer IDs, payment tokens). Encryption is **mandatory**.

### Encryption Options Comparison

| Method | Key Management | Performance | Use Case |
|---|---|---|---|
| SSE-S3 (AES-256) | AWS manages keys | Fastest | Default, no compliance req |
| SSE-KMS | AWS KMS (your CMK) | Slight overhead | Audit trail, key rotation |
| SSE-KMS + Bucket Key | KMS with caching | Near SSE-S3 speed | High-volume + compliance |
| SSE-C | Customer-provided | Same as SSE-S3 | Keys never stored in AWS |
| Client-side | You encrypt before upload | Varies | Zero-trust, end-to-end |

### Configuring Default Bucket Encryption

```bash
# Set SSE-KMS as default encryption with a Bucket Key
aws s3api put-bucket-encryption \
  --bucket retailmart-data-lake-prod \
  --server-side-encryption-configuration '{
    "Rules": [
      {
        "ApplyServerSideEncryptionByDefault": {
          "SSEAlgorithm": "aws:kms",
          "KMSMasterKeyID": "arn:aws:kms:us-east-1:123456789:key/mrk-abcdef123456"
        },
        "BucketKeyEnabled": true
      }
    ]
  }'

# Enforce encryption — deny unencrypted uploads
aws s3api put-bucket-policy \
  --bucket retailmart-data-lake-prod \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "DenyUnencryptedUploads",
        "Effect": "Deny",
        "Principal": "*",
        "Action": "s3:PutObject",
        "Resource": "arn:aws:s3:::retailmart-data-lake-prod/*",
        "Condition": {
          "StringNotEquals": {
            "s3:x-amz-server-side-encryption": "aws:kms"
          }
        }
      }
    ]
  }'
```

### SSE-KMS Bucket Key — Why It Matters

Without Bucket Key, **every object** operation calls KMS (= throttling at 10,000 requests/sec per region). With Bucket Key enabled, S3 generates a **bucket-level key** from your CMK and reuses it, reducing KMS calls by up to **99%**.

```
Without Bucket Key:                With Bucket Key:
PUT obj-1 → KMS call               PUT obj-1 ──┐
PUT obj-2 → KMS call                            ├──► KMS call (1x)
PUT obj-3 → KMS call               PUT obj-2 ──┤    → Bucket Key generated
PUT obj-4 → KMS call               PUT obj-3 ──┤    → Reused for all objects
...10,000 objects = 10,000 calls    PUT obj-4 ──┘
                                    ...10,000 objects ≈ 1 KMS call
```

### 💡 Interview Insight

> **Q: "You're hitting KMS throttling errors during a bulk data load. What's your fix?"**
>
> (1) **Enable Bucket Key** — reduces KMS API calls by 99%. (2) **Request a KMS quota increase** from AWS Support for the affected region. (3) If using SSE-KMS for cross-account access, make sure the KMS key policy grants the loading role `kms:GenerateDataKey` and `kms:Decrypt`. (4) Consider **SSE-S3** if you don't need audit-trail per-key — it has no API limits.

---

## Screen 10: Module 1 Quiz & Key Takeaways

### Bonus: S3 Bucket Policy vs. IAM Policy vs. ACL

A common interview side-question when discussing S3 security:

| Mechanism | Scope | Attached To | Use Case |
|---|---|---|---|
| IAM Policy | User/role-level | IAM principal | "User X can access bucket Y" |
| Bucket Policy | Bucket-level | S3 bucket | "This bucket allows account Z" |
| ACL | Object/bucket-level | Object or bucket | Legacy — avoid in new designs |
| Access Point | Application-level | Virtual endpoint | Per-app permissions at scale |
| VPC Endpoint Policy | Network-level | VPC endpoint | Restrict S3 to specific VPC |

Best practice: Use **IAM policies** for user permissions + **bucket policies** for cross-account access + **VPC endpoints** for network isolation. **Block public access** at the account level. ACLs are legacy — AWS recommends disabling them with `BucketOwnerEnforced` object ownership.

```bash
# Enforce bucket owner for all objects (disable ACLs)
aws s3api put-bucket-ownership-controls \
  --bucket retailmart-data-lake-prod \
  --ownership-controls '{"Rules": [{"ObjectOwnership": "BucketOwnerEnforced"}]}'

# Block ALL public access at bucket level
aws s3api put-public-access-block \
  --bucket retailmart-data-lake-prod \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

### Quiz

**Q1.** RetailMart stores 500 TB of raw POS data that's accessed frequently for the first 7 days, then rarely touched for 6 months, and must be retained for 7 years for tax compliance. Which lifecycle progression is most cost-effective?

- A) Standard → Glacier Deep Archive (Day 7) → Delete (Year 7)
- B) Standard → Standard-IA (Day 30) → Glacier Instant (Day 90) → Deep Archive (Day 180) → Delete (Year 7)
- C) Intelligent-Tiering for everything, delete at Year 7
- D) Standard → Standard-IA (Day 7) → Glacier Flexible (Day 90) → Deep Archive (Day 180) → Delete (Year 7)

**Answer: B.** You can't transition to Standard-IA before 30 days (minimum storage duration charge). Option B respects all minimum duration constraints and uses Glacier Instant for data that may occasionally be queried (quarterly reports). Option D violates the 30-day minimum for IA.

---

**Q2.** An S3 bucket has versioning enabled. A user runs `aws s3 rm s3://bucket/important-file.parquet`. What happens?

- A) The file is permanently deleted
- B) A delete marker is placed; all versions are retained
- C) The latest version is deleted; previous versions remain
- D) The operation fails because versioning is enabled

**Answer: B.** A simple DELETE on a versioned bucket creates a **delete marker**. The object appears deleted in standard listings, but all versions remain and can be restored by removing the delete marker.

---

**Q3.** RetailMart's Spark jobs read 50,000 small CSV files (avg 200 KB each) from a single S3 prefix and experience severe slowdowns. What's the PRIMARY fix?

- A) Enable Transfer Acceleration
- B) Compact the files into larger Parquet files
- C) Add more prefixes by hashing file names
- D) Switch to Glacier to reduce costs

**Answer: B.** The "small file problem" means each 200 KB file requires a separate HTTP request, and the overhead dominates. Compacting into larger files (128 MB–1 GB Parquet) dramatically reduces the number of requests and enables columnar predicate pushdown. Prefix distribution (C) helps with request throttling, but the primary bottleneck here is per-file overhead.

---

**Q4.** Which encryption method provides an **audit trail** of every key usage in AWS CloudTrail?

- A) SSE-S3
- B) SSE-KMS
- C) SSE-C
- D) Client-side encryption

**Answer: B.** SSE-KMS logs every `GenerateDataKey` and `Decrypt` call in CloudTrail, giving you a full audit trail of who accessed which encryption key and when. SSE-S3 uses AWS-managed keys with no per-object audit trail. SSE-C and client-side never touch KMS.

---

**Q5.** You configure CRR from us-east-1 to eu-west-1. The source bucket already has 2 PB of data. What happens to the existing data?

- A) It begins replicating immediately
- B) Nothing — only new objects are replicated
- C) It replays all PUT events and replicates
- D) AWS automatically creates a Batch Replication job

**Answer: B.** CRR only applies to objects created **after** the rule is enabled. You must use **S3 Batch Replication** to replicate existing objects.

---

### 🔑 Key Takeaways

1. **Storage classes are a cost lever** — the right tiering strategy can cut storage costs by 70%+. Know the minimum duration charges.
2. **Lifecycle policies automate what humans forget** — always pair versioning with noncurrent version expiration.
3. **Event notifications are your pipeline trigger** — EventBridge is the modern choice for fan-out and complex routing.
4. **S3 Select is `grep` for S3** — single-object, simple SQL. Athena is for dataset-wide analytics.
5. **Versioning + MFA Delete = ransomware protection** — but manage costs with noncurrent version policies.
6. **CRR doesn't replicate existing objects** — use Batch Replication for backfill.
7. **Prefix design drives throughput** — 5,500 GETs/sec per prefix. Compact small files.
8. **SSE-KMS + Bucket Key** is the enterprise standard — audit trail without KMS throttling.
