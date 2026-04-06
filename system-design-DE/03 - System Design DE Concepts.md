---
tags: [system-design, data-engineering, concepts, phase-7]
phase: 7
status: not-started
priority: high
---

# 🏛️ System Design — DE Concepts

> **Phase:** 7 | **Duration:** Review daily during Phase 7 | **Priority:** High
> **Related:** [[01 - System Design Data Platform]], [[02 - System Design DE Problems]]

---

## Checklist

### Distributed Systems Fundamentals
- [ ] CAP theorem: Consistency, Availability, Partition tolerance — pick 2
- [ ] ACID vs BASE: strong consistency vs eventual consistency
- [ ] Eventual consistency: how long until consistent, conflict resolution
- [ ] Strong consistency: linearizability, quorum reads/writes
- [ ] Consensus algorithms: Raft, Paxos (awareness level)

### Processing Guarantees
- [ ] Exactly-once: hardest, requires idempotent source + sink
- [ ] At-least-once: retry on failure, may produce duplicates
- [ ] At-most-once: fire and forget, may lose data
- [ ] Idempotent design: the practical approach to exactly-once

### Streaming Concepts
- [ ] Backpressure: what happens when consumer can't keep up with producer
- [ ] Handling: buffer, drop, flow control, auto-scaling
- [ ] Event-time vs processing-time: why ordering matters
- [ ] Out-of-order events: watermarks, allowed lateness, retractions

### Data Quality & Reliability
- [ ] Hot partitions: uneven data distribution, one partition overwhelmed
- [ ] Mitigation: salting, better partition keys, dynamic splitting
- [ ] Data skew: impacts joins, aggregations — see [[03 - Spark Performance Tuning]]
- [ ] Idempotent pipelines: see [[04 - ETL ELT Pipelines]]
- [ ] Data contracts: producer-consumer agreements

### Metadata & Lineage
- [ ] Data lineage: track data from source to consumption
- [ ] OpenLineage: open standard for lineage events
- [ ] DataHub: discovery, lineage, governance (LinkedIn)
- [ ] Amundsen: data discovery (Lyft)
- [ ] Column-level lineage: impact analysis for schema changes

### SLAs & Reliability
- [ ] SLA: Service Level Agreement (contractual)
- [ ] SLO: Service Level Objective (internal target)
- [ ] SLI: Service Level Indicator (the metric — freshness, completeness, latency)
- [ ] Pipeline SLAs: data freshness (daily table ready by 8am), completeness (>99.5%)
- [ ] Incident management: runbook, escalation, postmortem

### Cost Modeling
- [ ] Compute costs: Spark clusters, serverless invocations
- [ ] Storage costs: S3 tiers, warehouse storage pricing
- [ ] Egress costs: data transfer between regions/services
- [ ] Query costs: per-TB scanned (BigQuery), compute time (Snowflake)
- [ ] Optimization: right-sizing, auto-scaling, storage lifecycle

---

## 📝 Notes

_Start writing notes here as you study..._

---

## 🔗 Resources
- [ ] "Designing Data-Intensive Applications" by Martin Kleppmann (Chapters 5-9)
- [ ] "Fundamentals of Data Engineering" by Joe Reis (Chapter 3-4)
