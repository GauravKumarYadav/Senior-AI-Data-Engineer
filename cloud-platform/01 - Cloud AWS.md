---
tags: [cloud, aws, phase-3, data-engineering]
phase: 3
status: not-started
priority: high
---

# ☁️ Cloud Platform — AWS

> **Phase:** 3 | **Duration:** ~2 weeks | **Priority:** High
> **Related:** [[02 - Data Storage Formats]], [[03 - Data Warehousing]], [[05 - Spark in Production]], [[05 - Orchestration]]

---

## Checklist

### Amazon S3
- [ ] Storage classes: Standard, IA, Glacier, Glacier Deep Archive, Intelligent-Tiering
- [ ] Lifecycle policies: transition objects between storage classes, expiration
- [ ] Event notifications: S3 → Lambda / SQS / SNS (trigger pipelines on file arrival)
- [ ] S3 Select / Glacier Select: query in-place with SQL (predicate pushdown)
- [ ] Versioning: object version history, protect against accidental deletes
- [ ] Cross-Region Replication (CRR) — disaster recovery
- [ ] S3 performance: multipart upload, Transfer Acceleration, prefix design
- [ ] Encryption: SSE-S3, SSE-KMS, SSE-C, client-side encryption

### AWS Glue
- [ ] Glue Data Catalog: centralized metadata store (Hive metastore compatible)
- [ ] Crawlers: auto-discover schemas from S3/RDS/DynamoDB
- [ ] Glue ETL jobs: Spark-based, PySpark/Scala, auto-generated scripts
- [ ] Glue Studio: visual ETL, drag-and-drop
- [ ] Glue Bookmarks: incremental processing, track processed data
- [ ] Glue Schema Registry: schema validation for streaming (Avro, JSON Schema)
- [ ] Dynamic Frames: Glue's superset of Spark DataFrames, `resolveChoice`

### Amazon Athena
- [ ] Presto/Trino engine: serverless SQL queries on S3
- [ ] Partitioning: Hive-style partition projection → faster queries
- [ ] CTAS (Create Table As Select): materialize query results as Parquet/ORC
- [ ] Query federation: query across data sources (DynamoDB, RDS, Redshift)
- [ ] Cost: $5/TB scanned → columnar + partition = huge savings
- [ ] Athena Spark: serverless Spark notebooks in Athena
- [ ] Iceberg support: ACID queries on Iceberg tables

### Amazon EMR (Managed Spark/Hadoop)
- [ ] EMR on EC2: traditional, flexible, customizable
- [ ] EMR on EKS: Spark on Kubernetes
- [ ] EMR Serverless: fully managed, auto-scaling, pay per vCPU-hour
- [ ] Cluster configuration: instance types, instance fleets, spot instances
- [ ] EMR Studio: JupyterLab notebooks
- [ ] Bootstrap actions: install custom software on cluster nodes

### Amazon Kinesis
- [ ] Kinesis Data Streams: real-time ingestion, 1MB/sec per shard
- [ ] Kinesis Data Firehose: managed delivery to S3/Redshift/Elasticsearch
- [ ] Kinesis vs Kafka: shards vs partitions, managed vs self-managed
- [ ] Enhanced fan-out: dedicated throughput per consumer
- [ ] Kinesis Data Analytics: SQL/Flink on streaming data

### AWS Redshift
- [ ] See [[03 - Data Warehousing]] for Redshift internals
- [ ] Redshift Spectrum: query S3 data (extend warehouse to lake)
- [ ] Redshift Serverless: auto-scaling, pay per RPU
- [ ] Data sharing: cross-account, cross-region without data movement
- [ ] Federated queries: query RDS/Aurora from Redshift

### AWS Lake Formation
- [ ] Centralized governance: permissions on Data Catalog tables
- [ ] Fine-grained access: column-level, row-level, cell-level security
- [ ] Governed tables: ACID transactions on S3
- [ ] LF-Tags: tag-based access control (scalable permissions)
- [ ] Cross-account sharing with Lake Formation

### AWS Step Functions
- [ ] State machines: coordinate multiple AWS services
- [ ] Use with Glue, EMR, Lambda for data pipeline orchestration
- [ ] Standard vs Express workflows (duration, pricing)
- [ ] Error handling: retry, catch, fallback states

### IAM for Data Pipelines
- [ ] IAM roles: service roles for Glue, EMR, Lambda
- [ ] Cross-account access: assume role pattern
- [ ] S3 bucket policies vs IAM policies
- [ ] KMS: key management for encryption
- [ ] Least privilege principle: scoped policies per pipeline

---

## 📝 Notes

_Start writing notes here as you study..._

---

## 🔗 Resources
- [ ] AWS Well-Architected Framework — Data Analytics Lens
- [ ] AWS docs: Analytics services
- [ ] A Cloud Guru / Stephane Maarek — AWS Data Engineering courses
- [ ] AWS re:Invent talks (YouTube)

---

## 💡 Key Takeaways

1. 

---

## ❓ Interview Questions to Practice
- Design a data lake architecture on AWS (S3, Glue, Athena, Lake Formation)
- When would you use EMR vs Glue for Spark jobs?
- Compare Kinesis vs MSK (Managed Kafka) — tradeoffs
- How do you manage permissions for a multi-team data platform on AWS?
