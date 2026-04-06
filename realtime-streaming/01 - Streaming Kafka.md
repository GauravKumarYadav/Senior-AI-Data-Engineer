---
tags: [streaming, kafka, phase-3, data-engineering]
phase: 3
status: not-started
priority: high
---

# 📡 Streaming — Apache Kafka

> **Phase:** 3 | **Duration:** ~3 days | **Priority:** High
> **Related:** [[04 - Spark Streaming]], [[02 - Apache Flink]], [[04 - ETL ELT Pipelines]], [[01 - Cloud AWS]]

---

## Checklist

### Kafka Architecture
- [ ] Topics: logical channels for messages, immutable append-only logs
- [ ] Partitions: unit of parallelism within a topic, ordered within partition
- [ ] Brokers: Kafka servers, each holds partition replicas
- [ ] Producers: publish messages to topics, choose partition (key-based or round-robin)
- [ ] Consumers: read messages from topics, pull-based model
- [ ] Consumer groups: parallel consumption, one partition per consumer in group
- [ ] Rebalancing: partition reassignment when consumers join/leave group
- [ ] ZooKeeper (legacy) vs KRaft (new) — metadata management

### Kafka Internals
- [ ] Log segments: partitions split into segment files (time/size based)
- [ ] Offsets: unique sequential ID per message within partition
- [ ] Offset management: committed offsets, `auto.commit` vs manual commit
- [ ] Replication: leader + followers, ISR (In-Sync Replicas), `acks` setting
  - [ ] `acks=0`: no wait (fastest, data loss risk)
  - [ ] `acks=1`: leader only (moderate)
  - [ ] `acks=all`: all ISR (safest, slowest)
- [ ] Log compaction: retain latest value per key (vs time-based retention)
- [ ] Retention policies: time-based (`retention.ms`) vs size-based (`retention.bytes`)

### Exactly-Once Semantics
- [ ] Idempotent producers: `enable.idempotence=true`, dedup at broker
- [ ] Transactions: `transactional.id`, atomic writes across partitions
- [ ] `isolation.level=read_committed` — consumers see only committed messages
- [ ] End-to-end exactly-once: producer transactions + consumer offsets
- [ ] Practical limitations: external systems may need idempotent sinks

### Schema Registry
- [ ] Why: enforce schemas for messages, prevent breaking changes
- [ ] Avro + Schema Registry: schema ID in message header
- [ ] Protobuf support: alternative to Avro, better for cross-language
- [ ] Compatibility modes: BACKWARD, FORWARD, FULL, NONE
- [ ] Schema evolution: add optional fields (backward), ignore unknown (forward)
- [ ] Confluent Schema Registry vs AWS Glue Schema Registry

### Kafka Connect
- [ ] Source connectors: pull data from external systems into Kafka
- [ ] Sink connectors: push data from Kafka to external systems
- [ ] Common connectors: JDBC, S3, Elasticsearch, Debezium (CDC)
- [ ] Single Message Transforms (SMTs): lightweight in-flight transformations
- [ ] Distributed mode: scalable, fault-tolerant connector deployment
- [ ] Dead-letter queue for failed records

### Kafka Operations
- [ ] Topic configuration: partitions, replication factor, retention
- [ ] Consumer lag monitoring: how far behind consumers are
- [ ] Partition rebalancing: manual reassignment for hotspots
- [ ] Kafka CLI tools: `kafka-topics`, `kafka-console-consumer`, `kafka-consumer-groups`
- [ ] Monitoring: JMX metrics, Kafka Exporter + Prometheus + Grafana
- [ ] Managed Kafka: Confluent Cloud, AWS MSK, Azure Event Hubs

### Pub/Sub Patterns (Comparison)
- [ ] Kafka vs AWS Kinesis: capacity (partitions vs shards), retention, pricing
- [ ] Kafka vs Google Pub/Sub: push vs pull, serverless, ordering
- [ ] Kafka vs AWS SQS/SNS: queue semantics vs log semantics
- [ ] Kafka vs Apache Pulsar: tiered storage, multi-tenancy, geo-replication

---

## 📝 Notes

_Start writing notes here as you study..._

---

## 🔗 Resources
- [ ] "Kafka: The Definitive Guide" (O'Reilly, 2nd edition)
- [ ] Confluent docs: https://docs.confluent.io/
- [ ] Kafka design docs: https://kafka.apache.org/documentation/#design
- [ ] Tim Berglund's Kafka talks (YouTube)

---

## 💡 Key Takeaways

1. 

---

## ❓ Interview Questions to Practice
- How does Kafka guarantee message ordering?
- Explain exactly-once semantics in Kafka end-to-end
- Design a Kafka topic structure for an e-commerce event system
- When would you use Kafka vs a managed queue like SQS?
