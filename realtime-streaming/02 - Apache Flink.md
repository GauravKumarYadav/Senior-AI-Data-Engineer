---
tags: [streaming, flink, phase-3, data-engineering]
phase: 3
status: not-started
priority: medium
---

# 🌊 Apache Flink

> **Phase:** 3 | **Duration:** ~2 days | **Priority:** Medium
> **Related:** [[01 - Streaming Kafka]], [[04 - Spark Streaming]], [[04 - ETL ELT Pipelines]]

---

## Checklist

### Flink Architecture
- [ ] JobManager (master) + TaskManagers (workers)
- [ ] Dataflow graph: source → transformations → sink
- [ ] Task slots: unit of resource allocation within TaskManager
- [ ] Parallelism: per-operator, per-job, system-level
- [ ] Flink vs Spark Streaming: true stream processing vs micro-batch
- [ ] DataStream API vs Table API vs SQL API

### Event-Time Processing
- [ ] Event-time vs processing-time vs ingestion-time
- [ ] Watermarks: mechanism to track event-time progress
- [ ] Watermark strategies: `forBoundedOutOfOrderness`, `forMonotonousTimestamps`
- [ ] Late data handling: allowed lateness, side outputs
- [ ] Windows: tumbling, sliding, session, global windows
- [ ] Window assigners and triggers

### State Management
- [ ] Keyed state: per-key, partitioned across operators
- [ ] State types: `ValueState`, `ListState`, `MapState`, `ReducingState`
- [ ] State backends:
  - [ ] HashMapStateBackend: in-memory, fast, limited by heap
  - [ ] EmbeddedRocksDBStateBackend: disk-based, unlimited size, slightly slower
- [ ] State TTL: automatic state cleanup, prevent unbounded growth
- [ ] Checkpointing: periodic snapshots for fault recovery
- [ ] Savepoints: manual checkpoints for upgrades, redeployment

### Exactly-Once Guarantees
- [ ] Checkpointing: Chandy-Lamport algorithm, barrier alignment
- [ ] Exactly-once with Kafka: Flink's Kafka consumer + producer transactions
- [ ] Two-phase commit: for exactly-once sinks
- [ ] At-least-once mode: faster, barrier not aligned

### Flink SQL & Table API
- [ ] Dynamic tables: streaming equivalent of database tables
- [ ] Temporal joins: join with time-versioned tables
- [ ] CDC integration: Debezium → Flink SQL → downstream
- [ ] Connectors: Kafka, JDBC, Elasticsearch, filesystem

### When Flink vs Spark Streaming
- [ ] Flink: true event-by-event, lower latency, complex event processing (CEP)
- [ ] Spark Streaming: micro-batch, better batch-streaming unification, larger ecosystem
- [ ] Flink: better for stateful streaming, session windows, complex event patterns
- [ ] Spark: better if team already uses Spark for batch

---

## 📝 Notes

_Start writing notes here as you study..._

---

## 🔗 Resources
- [ ] Flink docs: https://flink.apache.org/
- [ ] "Stream Processing with Apache Flink" (O'Reilly)
- [ ] Ververica blog (Flink creators)

---

## 💡 Key Takeaways

1. 

---

## ❓ Interview Questions to Practice
- Compare Flink vs Spark Streaming for a real-time fraud detection use case
- How does Flink achieve exactly-once semantics?
- Explain watermarks in Flink with an example
