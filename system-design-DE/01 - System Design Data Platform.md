---
tags: [system-design, data-engineering, phase-7]
phase: 7
status: not-started
priority: high
---

# 🏛️ System Design — Data Platform

> **Phase:** 7 | **Duration:** ~3 days | **Priority:** High
> **Related:** [[02 - System Design DE Problems]], [[03 - System Design DE Concepts]], [[02 - Data Storage Formats]], [[01 - Cloud AWS]]

---

## Checklist

### Design a Data Lake / Lakehouse from Scratch
- [ ] Requirements gathering: data sources, volume, velocity, access patterns
- [ ] Storage layer: S3/GCS + table format ([[02 - Data Storage Formats]] — Delta/Iceberg)
- [ ] Medallion architecture: Bronze → Silver → Gold layers
- [ ] Ingestion: batch ([[04 - ETL ELT Pipelines]]) + streaming ([[01 - Streaming Kafka]])
- [ ] Processing: [[01 - Spark Core]] for batch, [[04 - Spark Streaming]]/[[02 - Apache Flink]] for stream
- [ ] Transformation: [[01 - dbt]] for SQL-based transforms in Gold layer
- [ ] Orchestration: [[05 - Orchestration]] (Airflow) for pipeline scheduling
- [ ] Serving: data warehouse ([[03 - Data Warehousing]]) for BI, feature store for ML
- [ ] Governance: catalog, lineage, access control ([[01 - Data Governance]])
- [ ] Monitoring: data quality checks, pipeline SLAs, alerting

### Design a Real-Time Analytics Pipeline
- [ ] Ingestion: Kafka for event collection
- [ ] Stream processing: Flink/Spark Streaming for aggregations
- [ ] Serving layer: Druid/Pinot/ClickHouse for OLAP queries
- [ ] Materialized views for pre-computed aggregations
- [ ] Dual-write: stream to both lake (historical) + serving (real-time)
- [ ] Tradeoffs: latency vs completeness vs cost

### Design a CDC Pipeline
- [ ] Source: OLTP database (Postgres/MySQL)
- [ ] Capture: Debezium → Kafka Connect → Kafka topics
- [ ] Process: Spark Streaming or Flink — apply transforms
- [ ] Sink: Delta Lake (MERGE for upserts)
- [ ] Transform: dbt for analytics models
- [ ] Serve: data warehouse / BI tools
- [ ] Handle: schema evolution, deletes, late events

### Design a Data Quality & Observability System
- [ ] Quality checks at ingestion, transformation, serving layers
- [ ] Metrics: completeness, freshness, accuracy, volume anomalies
- [ ] Tools: Great Expectations, Soda, dbt tests
- [ ] Data lineage: column-level, impact analysis (DataHub, OpenLineage)
- [ ] Alerting: threshold-based + anomaly detection
- [ ] SLAs: define and monitor for each pipeline/dataset

### Design a Multi-Tenant Data Platform
- [ ] Isolation: separate namespaces, schemas, or accounts per tenant
- [ ] Access control: RBAC, row-level security, column masking
- [ ] Cost allocation: tagging, chargeback per team/project
- [ ] Self-service: data catalog, governed access requests
- [ ] Shared infrastructure vs dedicated: tradeoffs

---

## 📝 Notes

_Start writing notes here as you study..._

---

## 🔗 Resources
- [ ] "Fundamentals of Data Engineering" by Joe Reis
- [ ] "Designing Data-Intensive Applications" by Martin Kleppmann
- [ ] Databricks architecture reference guides
- [ ] Company tech blogs: Netflix, Uber, Airbnb, LinkedIn

---

## 💡 Key Takeaways

1. 
