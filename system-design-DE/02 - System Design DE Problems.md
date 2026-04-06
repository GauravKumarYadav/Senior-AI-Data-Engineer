---
tags: [system-design, data-engineering, phase-7]
phase: 7
status: not-started
priority: high
---

# 🏛️ System Design — DE Problems

> **Phase:** 7 | **Duration:** ~4 days | **Priority:** High
> **Related:** [[01 - System Design Data Platform]], [[03 - System Design DE Concepts]]

---

## Checklist

### Practice Problems (1 per day)

#### Problem 1: Uber Surge Pricing Pipeline
- [ ] Real-time demand signals: ride requests per area per minute
- [ ] Supply signals: available drivers per area
- [ ] Geo-spatial partitioning: H3 hexagons for area aggregation
- [ ] Streaming pipeline: Kafka → Flink (real-time aggregation) → pricing service
- [ ] Batch pipeline: historical patterns for baseline pricing
- [ ] ML model serving: demand prediction model for proactive pricing

#### Problem 2: E-Commerce Data Warehouse
- [ ] Source systems: orders, products, customers, inventory, clickstream
- [ ] Dimensional model: `fct_orders`, `dim_customer`, `dim_product`, `dim_date`
- [ ] SCD Type 2 for customers/products
- [ ] Aggregation marts: daily revenue, category performance, cohort analysis
- [ ] Real-time inventory tracking alongside batch analytics

#### Problem 3: Recommendation System Data Pipeline
- [ ] Feature engineering: user behavior, item attributes, interaction history
- [ ] Batch features: user embeddings, item popularity (daily)
- [ ] Real-time features: recent clicks, session context (streaming)
- [ ] Feature store: offline (training) + online (serving)
- [ ] Training pipeline: scheduled retraining with fresh data
- [ ] Serving pipeline: feature lookup → model inference → personalized results

#### Problem 4: Log Analytics System
- [ ] Collection: Fluentd/Fluent Bit from application servers
- [ ] Transport: Kafka for buffering and fan-out
- [ ] Processing: Flink/Spark for enrichment, aggregation, anomaly detection
- [ ] Storage: Elasticsearch/OpenSearch for search, S3 for long-term archive
- [ ] Querying: Kibana dashboards, ad-hoc search
- [ ] Scale: handling 1M+ events/sec, retention policies

#### Problem 5: Event-Driven Fintech Architecture
- [ ] Event sourcing: transactions as immutable events
- [ ] CQRS: separate read and write models
- [ ] Kafka as event backbone: ordered transactions per account
- [ ] Real-time fraud detection: streaming ML model
- [ ] Regulatory compliance: audit trail, data retention, encryption
- [ ] Exactly-once processing for financial transactions

#### Problem 6: Data Mesh Architecture
- [ ] Domain-oriented ownership: each team owns their data products
- [ ] Data products: discoverable, addressable, self-describing, interoperable
- [ ] Self-serve platform: standardized tools (Spark, dbt, Airflow) provided centrally
- [ ] Federated governance: global policies, local autonomy
- [ ] Challenges: organizational change, data duplication, cross-domain queries

#### Problem 7: SCD Handler at Scale
- [ ] Requirements: track 500M dimension records with daily changes
- [ ] Approach: Spark + Delta Lake MERGE for SCD Type 2
- [ ] Optimization: partition by business key hash, incremental processing
- [ ] Storage format: Parquet with Z-ordering on surrogate key
- [ ] Serving: time-travel queries for point-in-time analysis

---

## 📝 System Design Framework
1. **Clarify requirements:** functional, non-functional, scale
2. **High-level architecture:** draw components, data flow
3. **Data model:** schema, partitioning, format
4. **Deep dive:** critical components, tradeoffs
5. **Failure handling:** fault tolerance, monitoring, alerting
6. **Cost & scale:** growth plan, cost optimization

---

## 🔗 Resources
- [ ] "Designing Data-Intensive Applications" by Martin Kleppmann
- [ ] Company tech blogs (Uber, Netflix, Airbnb, LinkedIn, Spotify)
- [ ] System design interview prep: YouTube channels (Jordan Has No Life, ByteByteGo)
