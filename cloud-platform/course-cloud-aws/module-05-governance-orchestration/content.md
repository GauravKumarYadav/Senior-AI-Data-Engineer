# Module 5: Lake Formation, Step Functions & IAM — Governance & Orchestration

> **Scenario**: RetailMart's data lake now spans S3 (Module 1), Glue/Athena (Module 2), EMR (Module 3), and Kinesis/Redshift (Module 4). But who can access what? The compliance team needs column-level PII restrictions, the analytics team needs cross-account data sharing, and the nightly ETL pipeline has 12 interdependent jobs that need orchestration with error handling. This module is about **governance** (who can see what) and **orchestration** (making it all run reliably).

---

## Screen 1: Lake Formation — Centralized Data Governance

AWS Lake Formation is a **centralized governance layer** on top of the Glue Data Catalog. Instead of managing S3 bucket policies, IAM policies, and Glue resource policies separately, Lake Formation provides a **single pane of glass** for data access control.

### Before vs. After Lake Formation

```
BEFORE Lake Formation:
┌─────────────────────────────────────────────────────────────────┐
│  Access Control Nightmare                                       │
│                                                                  │
│  S3 Bucket Policy: 200 lines of JSON                            │
│  + IAM Policy for Role A: 50 lines                              │
│  + IAM Policy for Role B: 50 lines                              │
│  + Glue Resource Policy: 80 lines                               │
│  + KMS Key Policy: 40 lines                                     │
│  + Athena Workgroup permissions                                  │
│  + Redshift Spectrum IAM role trust                              │
│  = 15 different places to manage, easy to misconfigure           │
└─────────────────────────────────────────────────────────────────┘

AFTER Lake Formation:
┌─────────────────────────────────────────────────────────────────┐
│  Centralized Governance                                          │
│                                                                  │
│  Lake Formation Console/API:                                     │
│    GRANT SELECT(store_id, total_amount) ON curated.daily_sales   │
│    TO ROLE data-analyst-role;                                    │
│                                                                  │
│  That's it. One statement. Lake Formation handles:               │
│    ✓ S3 access (via credential vending)                          │
│    ✓ Glue Catalog permissions                                    │
│    ✓ Athena query filtering                                      │
│    ✓ Redshift Spectrum access                                    │
│    ✓ EMR access                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Lake Formation Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                    AWS Lake Formation                             │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                 Permission Layer                             │ │
│  │                                                              │ │
│  │  Database Permissions ──► Table Permissions ──► Column/Row   │ │
│  │     CREATE_DATABASE         SELECT                 SELECT    │ │
│  │     ALTER_DATABASE          INSERT                 on cols   │ │
│  │     DROP_DATABASE           DELETE                 WHERE     │ │
│  │     DESCRIBE                ALTER                  filter    │ │
│  │                             DROP                             │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                          │                                        │
│                          ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │              Glue Data Catalog (Metadata)                    │ │
│  │  Databases → Tables → Columns → Partitions                  │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                          │                                        │
│                          ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │              S3 Data Lake (Data)                              │ │
│  │  Credential vending: LF provides temporary S3 credentials    │ │
│  │  to authorized services — no direct S3 IAM needed            │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                   │
│  Consumers: Athena │ EMR │ Redshift Spectrum │ Glue ETL          │
│  All check Lake Formation permissions before data access         │
└──────────────────────────────────────────────────────────────────┘
```

### Setting Up Lake Formation

```bash
# Step 1: Register S3 locations with Lake Formation
aws lakeformation register-resource \
  --resource-arn "arn:aws:s3:::retailmart-data-lake-prod" \
  --use-service-linked-role

# Step 2: Make Lake Formation the authority (revoke IAMAllowedPrincipals)
# This is THE critical step — without it, old IAM policies still bypass LF
aws lakeformation put-data-lake-settings \
  --data-lake-settings '{
    "DataLakeAdmins": [
      {"DataLakePrincipalIdentifier": "arn:aws:iam::123456789:role/DataLakeAdminRole"}
    ],
    "CreateDatabaseDefaultPermissions": [],
    "CreateTableDefaultPermissions": []
  }'

# Step 3: Revoke the default "IAMAllowedPrincipals" super-permission
aws lakeformation batch-revoke-permissions \
  --entries '[{
    "Id": "1",
    "Principal": {"DataLakePrincipalIdentifier": "IAM_ALLOWED_PRINCIPALS"},
    "Resource": {"Database": {"Name": "curated"}},
    "Permissions": ["ALL"],
    "PermissionsWithGrantOption": ["ALL"]
  }]'
```

### Granting Permissions

```bash
# Grant the analytics team SELECT on specific columns only
aws lakeformation grant-permissions \
  --principal '{"DataLakePrincipalIdentifier": "arn:aws:iam::123456789:role/AnalystRole"}' \
  --resource '{
    "TableWithColumns": {
      "DatabaseName": "curated",
      "Name": "daily_sales",
      "ColumnNames": ["store_id", "product_sku", "quantity", "total_amount", "transaction_ts"]
    }
  }' \
  --permissions '["SELECT"]'

# Note: customer_id and payment_method columns are NOT granted
# Analysts simply cannot see PII columns
```

### 💡 Interview Insight

> **Q: "Why not just use IAM policies for data lake access control?"**
>
> IAM policies operate at the **S3 object level** — you grant access to paths like `s3://bucket/curated/*`. But data access should be at the **table and column level** — "analyst X can see columns A, B, C of table Y but not column D (PII)." IAM can't do column-level filtering. Lake Formation adds a semantic layer: permissions on databases, tables, columns, and rows — regardless of the underlying S3 layout. It also centralizes what would otherwise be scattered across S3 policies, IAM roles, Glue policies, and KMS grants. The trade-off: initial setup complexity (revoking `IAMAllowedPrincipals`) trips up many teams.

---

## Screen 2: Fine-Grained Access — Column, Row & Cell-Level Security

### Column-Level Security

Restrict which columns a principal can see — the most common use case for PII protection.

```sql
-- What the analyst sees after column-level grant:
-- They query: SELECT * FROM curated.daily_sales
-- They get:

-- ✅ Visible:
-- store_id | product_sku | quantity | total_amount | transaction_ts

-- ❌ Hidden (Access Denied if queried directly):
-- customer_id | payment_method | card_last_four
```

### Row-Level Security (Data Filters)

Restrict which rows a principal can see — e.g., regional managers see only their region's data.

```bash
# Create a data cell filter for the Northeast region manager
aws lakeformation create-data-cells-filter \
  --table-data '{
    "TableCatalogId": "123456789012",
    "DatabaseName": "curated",
    "TableName": "daily_sales",
    "Name": "northeast-only",
    "RowFilter": {
      "FilterExpression": "region = '\''northeast'\''"
    },
    "ColumnNames": ["store_id", "product_sku", "quantity", "total_amount", "transaction_ts", "region"]
  }'

# Grant the filter to the Northeast manager role
aws lakeformation grant-permissions \
  --principal '{"DataLakePrincipalIdentifier": "arn:aws:iam::123456789:role/NortheastManagerRole"}' \
  --resource '{
    "DataCellsFilter": {
      "DatabaseName": "curated",
      "TableName": "daily_sales",
      "Name": "northeast-only",
      "TableCatalogId": "123456789012"
    }
  }' \
  --permissions '["SELECT"]'
```

### Cell-Level Security (Column + Row Filters Combined)

```
Full Table (what the data engineer sees):
┌──────────┬───────────┬────────────┬──────────┬──────────────┬────────┐
│ store_id │ cust_id   │ product    │ amount   │ card_last4   │ region │
├──────────┼───────────┼────────────┼──────────┼──────────────┼────────┤
│ 42       │ C-99381   │ TV-55      │ 499.99   │ 4242         │ NE     │
│ 17       │ C-44210   │ Laptop     │ 899.00   │ 1234         │ SE     │
│ 103      │ C-77512   │ Headphones │ 149.99   │ 5678         │ NE     │
│ 88       │ C-11293   │ Tablet     │ 329.00   │ 9999         │ MW     │
└──────────┴───────────┴────────────┴──────────┴──────────────┴────────┘

Northeast Manager (row filter: region='NE', column filter: no PII):
┌──────────┬────────────┬──────────┬────────┐
│ store_id │ product    │ amount   │ region │
├──────────┼────────────┼──────────┼────────┤
│ 42       │ TV-55      │ 499.99   │ NE     │
│ 103      │ Headphones │ 149.99   │ NE     │
└──────────┴────────────┴──────────┴────────┘

2 columns hidden (cust_id, card_last4) + 2 rows filtered (SE, MW)
= Cell-level security
```

### 💡 Interview Insight

> **Q: "How do you implement GDPR right-to-be-forgotten in a data lake?"**
>
> Multiple layers: (1) **Column-level security** via Lake Formation ensures analysts never see PII columns in the first place. (2) **Row-level filtering** can exclude flagged customers (`WHERE NOT deleted_flag`). (3) For actual deletion from S3, use **Apache Iceberg** with `DELETE FROM` or Delta Lake `VACUUM` to physically remove records. (4) For pseudonymization, use **S3 Object Lambda** or a Glue ETL job that hashes PII fields (e.g., `customer_id → SHA-256`). Lake Formation alone doesn't delete data — it controls visibility. Physical deletion requires a separate process.

---

## Screen 3: LF-Tags — Tag-Based Access Control at Scale

Managing individual table/column grants doesn't scale when you have 500+ tables across 20 teams. **LF-Tags** enable **attribute-based access control (ABAC)**: tag resources, tag principals, and let Lake Formation match them automatically.

### LF-Tag Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                  LF-Tag Based Access Control                     │
│                                                                  │
│  Tag: "sensitivity" = [public, internal, confidential, pii]     │
│  Tag: "domain"      = [sales, inventory, hr, finance]           │
│  Tag: "environment" = [dev, staging, prod]                      │
│                                                                  │
│  Resource Tagging:                                               │
│    curated.daily_sales          → sensitivity=internal,          │
│                                   domain=sales                   │
│    curated.daily_sales.cust_id  → sensitivity=pii               │
│    curated.employee_data        → sensitivity=confidential,      │
│                                   domain=hr                      │
│                                                                  │
│  Principal Tagging:                                              │
│    AnalystRole                  → sensitivity≤internal,          │
│                                   domain=sales                   │
│    HRManagerRole                → sensitivity≤confidential,      │
│                                   domain=hr                      │
│    DataEngineerRole             → sensitivity≤pii,               │
│                                   domain=*                       │
│                                                                  │
│  Auto-match:                                                     │
│    AnalystRole CAN access curated.daily_sales (internal+sales)  │
│    AnalystRole CANNOT access cust_id column (pii > internal)    │
│    AnalystRole CANNOT access employee_data (domain=hr ≠ sales)  │
│    HRManagerRole CAN access employee_data (confidential+hr)     │
│                                                                  │
│  New table tagged domain=sales, sensitivity=internal?            │
│    → AnalystRole AUTOMATICALLY gets access. Zero manual grants. │
└─────────────────────────────────────────────────────────────────┘
```

### Configuring LF-Tags

```bash
# Step 1: Create tag keys with allowed values
aws lakeformation create-lf-tag \
  --tag-key "sensitivity" \
  --tag-values '["public", "internal", "confidential", "pii"]'

aws lakeformation create-lf-tag \
  --tag-key "domain" \
  --tag-values '["sales", "inventory", "hr", "finance"]'

# Step 2: Assign tags to resources
aws lakeformation add-lf-tags-to-resource \
  --resource '{"Table": {"DatabaseName": "curated", "Name": "daily_sales"}}' \
  --lf-tags '[
    {"TagKey": "sensitivity", "TagValues": ["internal"]},
    {"TagKey": "domain", "TagValues": ["sales"]}
  ]'

# Tag PII columns separately
aws lakeformation add-lf-tags-to-resource \
  --resource '{
    "TableWithColumns": {
      "DatabaseName": "curated",
      "Name": "daily_sales",
      "ColumnNames": ["customer_id", "payment_method", "card_last_four"]
    }
  }' \
  --lf-tags '[{"TagKey": "sensitivity", "TagValues": ["pii"]}]'

# Step 3: Grant permissions using tags (tag-based policy)
aws lakeformation grant-permissions \
  --principal '{"DataLakePrincipalIdentifier": "arn:aws:iam::123456789:role/AnalystRole"}' \
  --resource '{
    "LFTagPolicy": {
      "ResourceType": "TABLE",
      "Expression": [
        {"TagKey": "sensitivity", "TagValues": ["public", "internal"]},
        {"TagKey": "domain", "TagValues": ["sales"]}
      ]
    }
  }' \
  --permissions '["SELECT", "DESCRIBE"]'
```

### Named Resources vs. LF-Tags

| Approach | Named Resources | LF-Tags |
|---|---|---|
| Granularity | Per-table/column grants | Tag-based policies |
| Scale | O(tables × roles) grants | O(tags × roles) policies |
| New table onboarding | Manual grant required | Auto if tagged correctly |
| Auditability | List all grants per role | Policy = tag expression |
| Best for | < 50 tables, few roles | 50+ tables, many teams |
| Flexibility | Maximum precision | Requires tag discipline |

### 💡 Interview Insight

> **Q: "How do you manage data access for 200 tables across 15 teams without drowning in IAM policies?"**
>
> **LF-Tags.** Define a tag taxonomy: `sensitivity` (public/internal/confidential/pii), `domain` (sales/inventory/hr/finance), `environment` (dev/staging/prod). Tag every table and column at creation time (enforce via Glue ETL pipelines or CI/CD). Create tag-based policies: "Role X gets SELECT on tables where sensitivity ≤ internal AND domain = sales." When a new sales table is created and tagged `domain=sales, sensitivity=internal`, Role X automatically gets access — zero tickets, zero manual grants. This scales from 200 to 2,000 tables without additional policies.

---

## Screen 4: Cross-Account Sharing with Lake Formation

RetailMart's analytics subsidiary operates in a separate AWS account. They need access to curated sales data without duplicating it.

### Cross-Account Architecture

```
┌────────────────────────────────────┐    ┌────────────────────────────────────┐
│  Producer Account (123456789012)   │    │  Consumer Account (987654321098)   │
│                                    │    │                                    │
│  Lake Formation Admin              │    │  Lake Formation Admin              │
│         │                          │    │         │                          │
│         ▼                          │    │         ▼                          │
│  ┌──────────────────┐              │    │  ┌──────────────────┐              │
│  │ Grant cross-acct │              │    │  │ Create resource  │              │
│  │ SELECT on        │──────────────┼───►│  │ link to shared   │              │
│  │ curated.sales    │   RAM Share  │    │  │ database         │              │
│  │ to account       │   or Direct  │    │  │                  │              │
│  │ 987654321098     │              │    │  │ Grant to local   │              │
│  └──────────────────┘              │    │  │ roles/users      │              │
│                                    │    │  └──────────────────┘              │
│  Data stays here (S3)              │    │                                    │
│  No data movement                  │    │  Query with Athena/EMR/Redshift   │
│                                    │    │  Compute is consumer-side          │
└────────────────────────────────────┘    └────────────────────────────────────┘
```

```bash
# On the PRODUCER account:
# Grant cross-account access using Lake Formation
aws lakeformation grant-permissions \
  --principal '{"DataLakePrincipalIdentifier": "987654321098"}' \
  --resource '{
    "Table": {
      "DatabaseName": "curated",
      "Name": "daily_sales"
    }
  }' \
  --permissions '["SELECT", "DESCRIBE"]' \
  --permissions-with-grant-option '["SELECT", "DESCRIBE"]'

# On the CONSUMER account:
# Accept the share (via RAM or direct Lake Formation)
# Create a resource link (a pointer to the shared database)
aws glue create-database \
  --database-input '{
    "Name": "shared_curated",
    "TargetDatabase": {
      "CatalogId": "123456789012",
      "DatabaseName": "curated"
    }
  }'

# Grant local roles access to the resource link
aws lakeformation grant-permissions \
  --principal '{"DataLakePrincipalIdentifier": "arn:aws:iam::987654321098:role/LocalAnalystRole"}' \
  --resource '{
    "Table": {
      "DatabaseName": "shared_curated",
      "Name": "daily_sales"
    }
  }' \
  --permissions '["SELECT"]'
```

### 💡 Interview Insight

> **Q: "What's the difference between cross-account access via S3 bucket policies vs. Lake Formation?"**
>
> **S3 bucket policies** give path-level access (e.g., `s3://bucket/curated/*`) — the consumer can read raw files but gets no schema, no column filtering, and you must separately grant Glue Catalog access. **Lake Formation** shares at the **table/column** level with full governance — the consumer queries through Athena/EMR/Spectrum and sees only permitted columns. LF also handles credential vending (temporary S3 credentials scoped to the granted data) so you don't need to expose your S3 bucket at all. The consumer doesn't even know your bucket name.

---

## Screen 5: Step Functions — State Machine Orchestration

AWS Step Functions coordinate multiple AWS services into **visual workflows** using state machines defined in Amazon States Language (ASL). Think of them as Airflow, but serverless and AWS-native.

### RetailMart Nightly ETL Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│              RetailMart Nightly ETL Pipeline                     │
│              (Step Functions State Machine)                      │
│                                                                  │
│  START                                                           │
│    │                                                             │
│    ▼                                                             │
│  ┌──────────────────┐                                            │
│  │ Check S3 for     │                                            │
│  │ today's raw files│ ←── Lambda: list S3 prefix, count files    │
│  └────────┬─────────┘                                            │
│           │                                                      │
│     ┌─────┴─────┐                                                │
│     │ Files > 0?│                                                │
│     └──┬─────┬──┘                                                │
│        │YES  │NO                                                 │
│        ▼     ▼                                                   │
│  ┌──────────┐ ┌──────────┐                                       │
│  │ Parallel │ │ Alert &  │                                       │
│  │ Branch   │ │ Wait 30m │──► Retry check (max 3 retries)       │
│  └──┬───┬───┘ └──────────┘                                       │
│     │   │                                                        │
│     │   ├──► Glue Job: POS ETL (raw → curated)                  │
│     │   ├──► Glue Job: Clickstream ETL                           │
│     │   └──► Glue Job: Inventory ETL                             │
│     │                                                            │
│     ▼ (wait for all 3)                                           │
│  ┌──────────────────┐                                            │
│  │ Data Quality      │ ←── Lambda: run Great Expectations checks │
│  │ Validation        │                                            │
│  └────────┬─────────┘                                            │
│     ┌─────┴─────┐                                                │
│     │ DQ Pass?  │                                                │
│     └──┬─────┬──┘                                                │
│        │YES  │NO                                                 │
│        ▼     ▼                                                   │
│  ┌──────────┐ ┌────────────────┐                                 │
│  │ Redshift │ │ Quarantine +   │                                 │
│  │ COPY     │ │ Alert PagerDuty│                                 │
│  └────┬─────┘ └────────────────┘                                 │
│       ▼                                                          │
│  ┌──────────────────┐                                            │
│  │ Update Glue      │ ←── Crawler: update partition metadata     │
│  │ Catalog           │                                            │
│  └────────┬─────────┘                                            │
│           ▼                                                      │
│  ┌──────────────────┐                                            │
│  │ Notify Success   │ ←── SNS: email data team                  │
│  └──────────────────┘                                            │
│       ▼                                                          │
│     END                                                          │
└─────────────────────────────────────────────────────────────────┘
```

### State Machine Definition (ASL)

```json
{
  "Comment": "RetailMart Nightly ETL Pipeline",
  "StartAt": "CheckForRawFiles",
  "States": {
    "CheckForRawFiles": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789:function:check-raw-files",
      "ResultPath": "$.fileCheck",
      "Next": "FilesExist?"
    },
    "FilesExist?": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.fileCheck.fileCount",
          "NumericGreaterThan": 0,
          "Next": "RunParallelETL"
        }
      ],
      "Default": "AlertAndWait"
    },
    "AlertAndWait": {
      "Type": "Wait",
      "Seconds": 1800,
      "Next": "CheckForRawFiles"
    },
    "RunParallelETL": {
      "Type": "Parallel",
      "Branches": [
        {
          "StartAt": "POS_ETL",
          "States": {
            "POS_ETL": {
              "Type": "Task",
              "Resource": "arn:aws:states:::glue:startJobRun.sync",
              "Parameters": {
                "JobName": "retailmart-pos-etl",
                "Arguments": {
                  "--job-bookmark-option": "job-bookmark-enable",
                  "--source_database": "raw_zone",
                  "--source_table": "pos_transactions"
                }
              },
              "Retry": [
                {
                  "ErrorEquals": ["Glue.ConcurrentRunsExceededException"],
                  "IntervalSeconds": 60,
                  "MaxAttempts": 3,
                  "BackoffRate": 2.0
                }
              ],
              "Catch": [
                {
                  "ErrorEquals": ["States.ALL"],
                  "Next": "POS_ETL_Failed",
                  "ResultPath": "$.error"
                }
              ],
              "End": true
            },
            "POS_ETL_Failed": {
              "Type": "Fail",
              "Cause": "POS ETL Glue job failed",
              "Error": "GlueJobError"
            }
          }
        },
        {
          "StartAt": "Clickstream_ETL",
          "States": {
            "Clickstream_ETL": {
              "Type": "Task",
              "Resource": "arn:aws:states:::glue:startJobRun.sync",
              "Parameters": {
                "JobName": "retailmart-clickstream-etl",
                "Arguments": {
                  "--job-bookmark-option": "job-bookmark-enable"
                }
              },
              "Retry": [
                {
                  "ErrorEquals": ["States.TaskFailed"],
                  "IntervalSeconds": 120,
                  "MaxAttempts": 2,
                  "BackoffRate": 2.0
                }
              ],
              "End": true
            }
          }
        },
        {
          "StartAt": "Inventory_ETL",
          "States": {
            "Inventory_ETL": {
              "Type": "Task",
              "Resource": "arn:aws:states:::glue:startJobRun.sync",
              "Parameters": {
                "JobName": "retailmart-inventory-etl"
              },
              "End": true
            }
          }
        }
      ],
      "ResultPath": "$.etlResults",
      "Next": "DataQualityCheck",
      "Catch": [
        {
          "ErrorEquals": ["States.ALL"],
          "Next": "QuarantineAndAlert",
          "ResultPath": "$.parallelError"
        }
      ]
    },
    "DataQualityCheck": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789:function:run-data-quality",
      "ResultPath": "$.dqResult",
      "Next": "DQPassed?"
    },
    "DQPassed?": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.dqResult.passed",
          "BooleanEquals": true,
          "Next": "LoadRedshift"
        }
      ],
      "Default": "QuarantineAndAlert"
    },
    "LoadRedshift": {
      "Type": "Task",
      "Resource": "arn:aws:states:::redshift-data:executeStatement.sync",
      "Parameters": {
        "ClusterIdentifier": "retailmart-analytics",
        "Database": "warehouse",
        "Sql": "CALL sp_load_daily_sales(:run_date)",
        "Parameters": [
          {
            "name": "run_date",
            "value.$": "$.fileCheck.runDate"
          }
        ]
      },
      "Next": "UpdateCatalog"
    },
    "UpdateCatalog": {
      "Type": "Task",
      "Resource": "arn:aws:states:::glue:startCrawler.sync",
      "Parameters": {
        "Name": "retailmart-curated-crawler"
      },
      "Next": "NotifySuccess"
    },
    "NotifySuccess": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:us-east-1:123456789:etl-notifications",
        "Subject": "✅ RetailMart Nightly ETL Complete",
        "Message.$": "States.Format('ETL completed successfully. Files processed: {}', $.fileCheck.fileCount)"
      },
      "End": true
    },
    "QuarantineAndAlert": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:us-east-1:123456789:etl-alerts-critical",
        "Subject": "🚨 RetailMart ETL FAILED — Data Quarantined",
        "Message.$": "States.Format('Pipeline failed. Error: {}', $.parallelError)"
      },
      "Next": "PipelineFailed"
    },
    "PipelineFailed": {
      "Type": "Fail",
      "Cause": "ETL pipeline failed quality checks or job execution",
      "Error": "PipelineError"
    }
  }
}
```

### 💡 Interview Insight

> **Q: "Why Step Functions over Airflow (MWAA) for orchestration?"**
>
> **Step Functions** when: (1) Workflow is AWS-service-centric (Glue, Lambda, ECS, Redshift). (2) You want zero infrastructure (no Airflow scheduler/workers to manage). (3) Workflows are event-driven (triggered by S3 events, EventBridge). (4) Visual debugging in the console is valuable. (5) Cost-per-state-transition pricing works for your volume.
>
> **MWAA (Airflow)** when: (1) Team has existing Airflow DAGs and expertise. (2) You need complex scheduling (cron, sensors, backfill). (3) Pipelines span non-AWS services. (4) You need dynamic task generation (Airflow's dynamic DAGs). (5) You want a Python-first API vs. JSON/YAML state definitions. RetailMart uses Step Functions for the nightly ETL (all AWS services) and MWAA for the monthly ML retraining pipeline (spans SageMaker, on-prem data sources, and Databricks).

---

## Screen 6: Standard vs. Express Workflows

### Comparison

| Feature | Standard Workflows | Express Workflows |
|---|---|---|
| Duration | Up to 1 year | Up to 5 minutes |
| Execution model | Exactly-once | At-least-once (async) or at-most-once (sync) |
| Pricing | Per state transition ($0.025/1K) | Per request + duration |
| Execution history | Full (90 days in console) | CloudWatch Logs only |
| Max executions | 25,000 concurrent | 100,000/sec start rate |
| Use case | Long-running ETL, approvals | High-volume, short transforms |

### When to Use Express

```
Standard Workflow (RetailMart nightly ETL):
  • 12 states, runs for 45 minutes
  • Must be exactly-once (no duplicate Redshift loads)
  • 1 execution per night
  • Cost: 12 transitions × $0.025/1K = negligible

Express Workflow (RetailMart per-order processing):
  • Triggered per order event from Kinesis
  • 4 states: validate → enrich → write → notify
  • 50,000 orders/hour at peak
  • Duration: 2-3 seconds per execution
  • Cost: much cheaper than 50K × 12 state transitions
```

```json
{
  "Comment": "Express: per-order enrichment",
  "StartAt": "ValidateOrder",
  "States": {
    "ValidateOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789:function:validate-order",
      "Next": "EnrichWithCustomerData"
    },
    "EnrichWithCustomerData": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:getItem",
      "Parameters": {
        "TableName": "customer-profiles",
        "Key": {"customer_id": {"S.$": "$.customerId"}}
      },
      "ResultPath": "$.customerProfile",
      "Next": "WriteToS3"
    },
    "WriteToS3": {
      "Type": "Task",
      "Resource": "arn:aws:states:::s3:putObject",
      "Parameters": {
        "Bucket": "retailmart-enriched-orders",
        "Key.$": "States.Format('orders/{}/{}.json', $.orderDate, $.orderId)",
        "Body.$": "$"
      },
      "End": true
    }
  }
}
```

### 💡 Interview Insight

> **Q: "You have a Step Functions workflow triggered 100,000 times per day with 5 states each. Is Standard or Express cheaper?"**
>
> **Standard**: 100K executions × 5 transitions = 500K transitions/day × 30 = 15M/month. Cost: 15M × $0.025/1K = **$375/month**.
> **Express**: 100K × $0.000001 per request = $0.10/day for requests + duration charges. If each takes 3 seconds with 64 MB: 100K × 3s × $0.00001667/sec = $5/day ≈ **$153/month**.
> Express is ~60% cheaper AND handles the concurrency better. Rule of thumb: high-volume, short-lived → Express. Low-volume, long-running → Standard.

---

## Screen 7: Error Handling — Retry, Catch & Fallback

Step Functions provide built-in error handling that's more robust than most orchestrators.

### Error Handling Patterns

```
┌─────────────────────────────────────────────────────┐
│                 Error Handling Flow                   │
│                                                      │
│  Task Executes                                       │
│       │                                              │
│       ├── Success ──► Next State                     │
│       │                                              │
│       ├── Retryable Error ──► Retry Policy           │
│       │   (e.g., throttling)    │                    │
│       │                         ├── Attempt 1 (wait 5s)
│       │                         ├── Attempt 2 (wait 10s)
│       │                         └── Attempt 3 (wait 20s)
│       │                               │              │
│       │                         ┌─────┴─────┐        │
│       │                         │ Recovered? │       │
│       │                         └──┬─────┬──┘        │
│       │                            YES   NO          │
│       │                            │     │           │
│       │                            ▼     ▼           │
│       │                         Next   Catch         │
│       │                         State  Handler       │
│       │                                  │           │
│       └── Non-retryable Error ───────────┤           │
│           (e.g., invalid input)          │           │
│                                          ▼           │
│                                    Fallback State    │
│                                    (alert, cleanup,  │
│                                     alternative path)│
└─────────────────────────────────────────────────────┘
```

### Retry Configuration

```json
{
  "Retry": [
    {
      "ErrorEquals": ["Glue.ConcurrentRunsExceededException"],
      "IntervalSeconds": 60,
      "MaxAttempts": 5,
      "BackoffRate": 2.0,
      "MaxDelaySeconds": 300
    },
    {
      "ErrorEquals": ["Lambda.ServiceException", "Lambda.TooManyRequestsException"],
      "IntervalSeconds": 5,
      "MaxAttempts": 3,
      "BackoffRate": 2.0
    },
    {
      "ErrorEquals": ["States.Timeout"],
      "IntervalSeconds": 30,
      "MaxAttempts": 2,
      "BackoffRate": 1.0
    }
  ],
  "Catch": [
    {
      "ErrorEquals": ["Glue.EntityNotFoundException"],
      "Next": "HandleMissingTable",
      "ResultPath": "$.errorInfo"
    },
    {
      "ErrorEquals": ["States.ALL"],
      "Next": "GenericErrorHandler",
      "ResultPath": "$.errorInfo"
    }
  ]
}
```

### Built-In Error Types

| Error | Meaning |
|---|---|
| `States.ALL` | Catches everything (wildcard) |
| `States.Timeout` | Task exceeded `TimeoutSeconds` |
| `States.TaskFailed` | Task returned a failure |
| `States.Permissions` | Insufficient IAM permissions |
| `States.ResultPathMatchFailure` | ResultPath can't apply to input |
| `States.HeartbeatTimeout` | Long task missed heartbeat |

### 💡 Interview Insight

> **Q: "A Glue job in your Step Functions pipeline intermittently fails due to Spark executor OOM. How do you handle it?"**
>
> Multi-layered approach: (1) **Retry** in Step Functions with exponential backoff — OOM can be transient if other jobs were competing for cluster resources. (2) **Catch** after max retries → trigger a fallback state that re-runs the Glue job with a larger worker type (`G.2X` instead of `G.1X`). (3) Use `ResultPath` to preserve the error details and pass them to the fallback. (4) If the fallback also fails, **Catch** → alert PagerDuty with full error context. The state machine encodes the escalation policy: retry → scale up → alert human. No silent failures.

---

## Screen 8: IAM for Data Pipelines

### Service Roles — The Identity Chain

Every AWS service in your pipeline needs an IAM role. Understanding the trust chain is critical for debugging "Access Denied" errors.

```
┌──────────────────────────────────────────────────────────────────┐
│                   IAM Role Chain for ETL Pipeline                 │
│                                                                   │
│  EventBridge Rule                                                 │
│    → Assumes: StepFunctions Execution Role                       │
│       Trust: events.amazonaws.com                                 │
│       Permissions: states:StartExecution                         │
│                                                                   │
│  Step Functions Execution Role                                    │
│    → Assumes: passes role to Glue                                │
│       Trust: states.amazonaws.com                                 │
│       Permissions: glue:StartJobRun, lambda:InvokeFunction,      │
│                    iam:PassRole, sns:Publish                      │
│                                                                   │
│  Glue ETL Job Role                                                │
│    → Reads S3, writes S3, accesses Catalog                       │
│       Trust: glue.amazonaws.com                                   │
│       Permissions: s3:GetObject, s3:PutObject,                   │
│                    glue:GetTable, glue:UpdateTable,               │
│                    logs:CreateLogStream, kms:Decrypt              │
│                                                                   │
│  Redshift Copy Role                                               │
│    → Reads S3 curated data into Redshift                         │
│       Trust: redshift.amazonaws.com                               │
│       Permissions: s3:GetObject, s3:ListBucket,                  │
│                    kms:Decrypt (for encrypted S3 data)            │
│                                                                   │
│  Lambda Data Quality Role                                         │
│    → Reads S3, queries Athena, writes results                    │
│       Trust: lambda.amazonaws.com                                 │
│       Permissions: s3:GetObject, athena:StartQueryExecution,     │
│                    athena:GetQueryResults, s3:PutObject           │
└──────────────────────────────────────────────────────────────────┘
```

### Least-Privilege IAM Policy for Glue ETL

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadRawData",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::retailmart-data-lake-prod",
        "arn:aws:s3:::retailmart-data-lake-prod/raw/*"
      ]
    },
    {
      "Sid": "WriteCuratedData",
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::retailmart-data-lake-prod/curated/*"
    },
    {
      "Sid": "WriteTempDir",
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:GetObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::retailmart-temp/glue/*"
    },
    {
      "Sid": "GlueCatalogAccess",
      "Effect": "Allow",
      "Action": [
        "glue:GetTable", "glue:GetTables",
        "glue:GetDatabase", "glue:GetDatabases",
        "glue:UpdateTable", "glue:CreatePartition",
        "glue:BatchCreatePartition", "glue:GetPartitions"
      ],
      "Resource": [
        "arn:aws:glue:us-east-1:123456789:catalog",
        "arn:aws:glue:us-east-1:123456789:database/raw_zone",
        "arn:aws:glue:us-east-1:123456789:database/curated",
        "arn:aws:glue:us-east-1:123456789:table/raw_zone/*",
        "arn:aws:glue:us-east-1:123456789:table/curated/*"
      ]
    },
    {
      "Sid": "KMSDecryptEncrypt",
      "Effect": "Allow",
      "Action": ["kms:Decrypt", "kms:GenerateDataKey"],
      "Resource": "arn:aws:kms:us-east-1:123456789:key/mrk-abcdef",
      "Condition": {
        "StringEquals": {
          "kms:ViaService": "s3.us-east-1.amazonaws.com"
        }
      }
    },
    {
      "Sid": "CloudWatchLogs",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup", "logs:CreateLogStream", "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:us-east-1:123456789:log-group:/aws-glue/*"
    }
  ]
}
```

### Cross-Account Assume Role

```python
import boto3

def get_cross_account_session(
    role_arn: str,
    session_name: str = "cross-account-etl",
    duration_seconds: int = 3600,
) -> boto3.Session:
    """Assume a role in another AWS account and return a session."""
    sts = boto3.client("sts")
    
    credentials = sts.assume_role(
        RoleArn=role_arn,
        RoleSessionName=session_name,
        DurationSeconds=duration_seconds,
    )["Credentials"]
    
    return boto3.Session(
        aws_access_key_id=credentials["AccessKeyId"],
        aws_secret_access_key=credentials["SecretAccessKey"],
        aws_session_token=credentials["SessionToken"],
    )

# Read data from Account B's S3 bucket
session = get_cross_account_session(
    "arn:aws:iam::987654321098:role/CrossAccountDataReader"
)
s3 = session.client("s3")
response = s3.list_objects_v2(
    Bucket="partner-data-feed",
    Prefix="daily-exports/2026/04/",
)
```

### 💡 Interview Insight

> **Q: "Your Glue job gets 'Access Denied' reading from S3. Walk me through debugging."**
>
> Systematic approach: (1) **Which role?** Check the Glue job's configured IAM role. (2) **S3 bucket policy?** Does it explicitly deny the role or require specific conditions? (3) **KMS?** If the S3 object is encrypted with KMS, the role needs `kms:Decrypt` on that specific key. (4) **Lake Formation?** If LF is enabled, IAM alone isn't sufficient — the role needs LF grants too. (5) **VPC endpoint policy?** If Glue runs in a VPC with an S3 endpoint, the endpoint policy may restrict access. (6) **SCP?** Organization Service Control Policies can override everything. Debug order: IAM → Bucket Policy → KMS → Lake Formation → VPC → SCP. The `aws iam simulate-principal-policy` CLI command helps test without trial-and-error.

---

## Screen 9: S3 Bucket Policies vs. IAM Policies & KMS

### Policy Evaluation Logic

```
Request to access S3 object
         │
         ▼
┌────────────────────┐
│ Organization SCP   │──── Deny? ──► DENIED
│ (account-level)    │
└────────┬───────────┘
         │ Allow
         ▼
┌────────────────────┐
│ IAM Policy         │──── Explicit Deny? ──► DENIED
│ (principal-level)  │
└────────┬───────────┘
         │ Allow or no statement
         ▼
┌────────────────────┐
│ S3 Bucket Policy   │──── Explicit Deny? ──► DENIED
│ (resource-level)   │
└────────┬───────────┘
         │
         ▼
┌────────────────────┐
│ Union of Allows    │──── Any Allow? ──► ALLOWED
│ (IAM + Bucket)     │──── No Allow?  ──► DENIED
└────────────────────┘
         │
         ▼ (if encrypted)
┌────────────────────┐
│ KMS Key Policy     │──── Key policy allows? ──► ALLOWED
│                    │──── Key policy denies?  ──► DENIED
└────────────────────┘
```

### Same-Account vs. Cross-Account Access

```
SAME ACCOUNT:
  IAM Policy allows s3:GetObject on bucket → ALLOWED
  (Bucket policy doesn't need to explicitly allow — union applies)

CROSS ACCOUNT:
  IAM Policy in Account A allows s3:GetObject on Account B's bucket → NOT ENOUGH
  Bucket Policy in Account B MUST also allow Account A's role → THEN ALLOWED
  (Both sides must explicitly allow for cross-account)
```

### KMS Key Management for Data Pipelines

```bash
# Create a customer-managed KMS key for the data lake
aws kms create-key \
  --description "RetailMart Data Lake Encryption Key" \
  --key-usage ENCRYPT_DECRYPT \
  --key-spec SYMMETRIC_DEFAULT \
  --tags '[
    {"TagKey": "Project", "TagValue": "DataLake"},
    {"TagKey": "Environment", "TagValue": "Production"}
  ]'

# Key policy: allow data pipeline roles to use the key
aws kms put-key-policy \
  --key-id "mrk-abcdef123456" \
  --policy-name default \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "RootAccountAccess",
        "Effect": "Allow",
        "Principal": {"AWS": "arn:aws:iam::123456789:root"},
        "Action": "kms:*",
        "Resource": "*"
      },
      {
        "Sid": "GlueETLAccess",
        "Effect": "Allow",
        "Principal": {"AWS": "arn:aws:iam::123456789:role/GlueETLRole"},
        "Action": ["kms:Decrypt", "kms:GenerateDataKey"],
        "Resource": "*",
        "Condition": {
          "StringEquals": {
            "kms:ViaService": "s3.us-east-1.amazonaws.com"
          }
        }
      },
      {
        "Sid": "RedshiftCopyAccess",
        "Effect": "Allow",
        "Principal": {"AWS": "arn:aws:iam::123456789:role/RedshiftCopyRole"},
        "Action": "kms:Decrypt",
        "Resource": "*"
      },
      {
        "Sid": "CrossAccountAnalytics",
        "Effect": "Allow",
        "Principal": {"AWS": "arn:aws:iam::987654321098:role/AnalyticsRole"},
        "Action": "kms:Decrypt",
        "Resource": "*",
        "Condition": {
          "StringEquals": {
            "kms:ViaService": "s3.us-east-1.amazonaws.com"
          }
        }
      }
    ]
  }'

# Enable automatic key rotation (every year)
aws kms enable-key-rotation --key-id "mrk-abcdef123456"
```

### 💡 Interview Insight

> **Q: "Your cross-account Glue job can read S3 objects in the other account but gets 'Access Denied' on kms:Decrypt. What's wrong?"**
>
> The KMS key policy in the source account must explicitly grant the cross-account role `kms:Decrypt`. Unlike S3 bucket policies where IAM + bucket policy union applies within the same account, **KMS key policies are the primary authority** — if the key policy doesn't grant the external role, no amount of IAM policies will help. Fix: add the cross-account role ARN to the KMS key policy with `kms:Decrypt` permission. Also add the `kms:ViaService` condition to restrict the key usage to S3 operations only (least privilege).

---

## Screen 10: Module 5 Quiz & Key Takeaways

### Quiz

**Q1.** RetailMart's data lake has 300 tables. A new "Demand Planning" team needs SELECT access to all tables tagged `domain=sales` and `sensitivity≤internal`. What's the most scalable approach?

- A) Create individual GRANT statements for each of the ~100 sales tables
- B) Use LF-Tags: grant SELECT where `domain=sales AND sensitivity IN (public, internal)`
- C) Give them the DataLakeAdmin role
- D) Create an S3 bucket policy allowing their role to read `s3://bucket/curated/sales/*`

**Answer: B.** LF-Tags provide attribute-based access control — one policy statement covers all current and future tables matching the tag expression. Option A doesn't scale. Option C is massively over-privileged. Option D operates at the S3 path level, not the table/column level.

---

**Q2.** A Step Functions Standard workflow has 8 states and runs 50,000 times per month. An Express workflow for the same pipeline takes 4 seconds per execution. Which is cheaper?

- A) Standard: 50K × 8 × $0.025/1K = $10/month
- B) Express: 50K × ($0.000001 + 4s × $0.00001667) ≈ $3.38/month
- C) They cost the same
- D) Cannot determine without knowing Lambda costs

**Answer: B.** Standard = 400K transitions × $0.025/1K = $10. Express = 50K × $0.000001 (request) + 50K × 4 × $0.00001667 (duration) ≈ $3.38. Express is ~66% cheaper for high-volume, short-duration workflows.

---

**Q3.** A Glue ETL job in Account A needs to read S3 data encrypted with a KMS CMK in Account B. Which permissions are required?

- A) Only IAM policy in Account A allowing `s3:GetObject`
- B) IAM policy (Account A) + S3 bucket policy (Account B)
- C) IAM policy (Account A) + S3 bucket policy (Account B) + KMS key policy (Account B)
- D) Lake Formation grant only

**Answer: C.** Cross-account S3 requires both IAM and bucket policy. Cross-account KMS additionally requires the key policy to grant the external role `kms:Decrypt`. All three must align — missing any one produces "Access Denied."

---

**Q4.** Which Lake Formation feature allows the Northeast regional manager to see only rows where `region = 'northeast'`?

- A) Column-level security
- B) LF-Tags
- C) Data cell filters (row-level security)
- D) S3 prefix-based access

**Answer: C.** Data cell filters enable row-level (and combined row+column) security by applying filter expressions like `region = 'northeast'` at the Lake Formation level. The filter is enforced transparently when the principal queries through Athena, EMR, or Redshift Spectrum.

---

**Q5.** Your Step Functions workflow calls a Lambda function that intermittently times out due to a downstream API being slow. Which error handling configuration is most appropriate?

- A) `"Catch": [{"ErrorEquals": ["States.ALL"], "Next": "Fail"}]`
- B) `"Retry": [{"ErrorEquals": ["States.Timeout"], "IntervalSeconds": 30, "MaxAttempts": 3, "BackoffRate": 2}]` followed by a Catch
- C) Remove the Lambda timeout
- D) Use an Express workflow instead

**Answer: B.** Retry with exponential backoff for `States.Timeout` handles transient API slowness gracefully. After max retries, the Catch sends the execution to a fallback/alert state. Option A fails immediately with no retry. Option C is dangerous (Lambda has a 15-minute max). Option D doesn't solve the timeout.

---

### 🔑 Key Takeaways

1. **Lake Formation centralizes governance** — one place for database, table, column, and row-level permissions instead of juggling S3 + IAM + Glue policies.
2. **Revoke `IAMAllowedPrincipals`** — this is THE critical setup step. Without it, old IAM policies bypass Lake Formation entirely.
3. **LF-Tags scale permissions** — tag-based policies mean new tables get automatic access grants. O(tags) policies vs. O(tables × roles) grants.
4. **Cell-level security** = column filters + row filters combined. Essential for PII compliance and multi-tenant data lakes.
5. **Cross-account sharing via Lake Formation** is cleaner than S3 bucket policies — consumers see tables/columns, not file paths.
6. **Step Functions for AWS-native orchestration** — visual, serverless, built-in retry/catch. Use Standard for long ETL, Express for high-volume micro-workflows.
7. **Error handling is not optional** — every Task state needs Retry + Catch. Encode your escalation policy in the state machine.
8. **IAM for data pipelines is a chain** — EventBridge → Step Functions → Glue → S3 → KMS. Every link needs the right trust policy and permissions.
9. **Cross-account = both sides must allow** — IAM policy + bucket policy + KMS key policy. Miss one link and you get "Access Denied."
10. **KMS key rotation and `kms:ViaService` conditions** are best practices — least privilege extends to encryption keys, not just data access.
