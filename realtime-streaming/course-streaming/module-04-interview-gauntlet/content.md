# Module 4: The Interview Gauntlet 🔥

> **Format**: Rapid-fire questions with detailed model answers. Practice these until you can answer confidently in 60-90 seconds each. Every answer follows the **Situation → Approach → Trade-offs** structure interviewers love.

---

## Screen 1: Architecture Design Questions

### Q1: "Design a Kafka-based system for an e-commerce platform processing 100K orders/hour with real-time inventory updates."

**Model Answer:**

```
┌──────────────┐    ┌──────────────────────────────────────────┐
│ Order Service│    │           Kafka Cluster (KRaft)           │
│ (Producer)   │───►│  order-events: 12 parts, RF=3, acks=all  │
│ key=order_id │    │  inventory-updates: 20 parts, compacted  │
│              │    │  payment-results: 6 parts, RF=3          │
└──────────────┘    └────┬─────────────┬──────────────┬────────┘
                         │             │              │
                    ┌────▼────┐  ┌─────▼─────┐  ┌────▼──────┐
                    │Order    │  │Inventory  │  │Analytics  │
                    │Processor│  │Service    │  │Pipeline   │
                    │(CG, EOS)│  │(CG,compact│  │(CG, Flink)│
                    └─────────┘  │ reader)   │  └───────────┘
                                 └───────────┘
```

"I'd create three topics: `order-events` (12 partitions — 100K orders/hour ≈ 28/sec, well within capacity, room for 12 parallel consumers), `inventory-updates` (compacted, key=`warehouse-sku`, 20 partitions for 200 warehouses), and `payment-results` (6 partitions). All with RF=3, `min.insync.replicas=2`, `acks=all` for durability.

Partition key for orders is `order_id` to keep all lifecycle events ordered. Inventory uses compaction so consumers always see the latest stock level. I'd use Schema Registry with Avro and BACKWARD compatibility.

Three consumer groups: order processing (EOS with transactions), inventory service (reads compacted topic for current state), and analytics (Flink for real-time dashboards). The trade-off is the operational complexity of managing EOS transactions for order processing, but the data integrity justifies it."

---

### Q2: "You have a clickstream of 50K events/second. Design the Kafka topic configuration."

**Model Answer:**

"50K events/sec with average 500-byte events = 25 MB/s sustained. Each partition handles roughly 10 MB/s, so I need at least 3 partitions for throughput — but I'd provision **30-50 partitions** for consumer parallelism and growth headroom.

Configuration:
- `partitions: 40` (allows 40 parallel consumers)
- `replication.factor: 3`
- `acks: 1` (clickstream tolerates some loss for lower latency)
- `retention.ms: 604800000` (7 days — short-lived analytical value)
- `compression.type: lz4` (fast compression, ~60% reduction)
- `batch.size: 65536` and `linger.ms: 10` for producer batching

Key = `user_id` for per-user ordering. If ordering isn't needed and you want max throughput, use `null` keys with sticky partitioning.

Trade-off: 40 partitions increases rebalancing time and file handle count, but the throughput and parallelism benefits outweigh this for high-volume clickstream."

---

### Q3: "Design a partition key strategy for a multi-tenant SaaS platform."

**Model Answer:**

"The naive approach — `tenant_id` as partition key — creates **hot partitions**. If Walmart has 10M events/day and a small business has 100, Walmart's partition is 100,000x hotter.

Better approaches:

**Option A: Composite key** — `tenant_id + entity_id`. Distributes large tenants across partitions while maintaining per-entity ordering. `walmart-order-123` and `walmart-order-456` go to different partitions.

**Option B: Dedicated topics per tier** — Premium tenants get their own topic (SLA isolation). Standard tenants share a multi-tenant topic with composite keys.

**Option C: Custom partitioner** — Hash on `entity_id` for large tenants, hash on `tenant_id` for small tenants. Requires maintaining a tenant tier config.

I'd choose Option A for most cases — it's simple, distributes load, and maintains per-entity ordering. Option B if you have strict tenant isolation SLAs."

---

### Q4: "How would you migrate from a monolithic Kafka topic to a microservices event architecture?"

**Model Answer:**

"Strangler fig pattern applied to Kafka:

1. **Phase 1: Shadow topic** — Create new domain-specific topics (`order-events`, `payment-events`, `shipping-events`). Deploy a 'splitter' consumer that reads the monolith topic and routes events to domain topics based on event type.

2. **Phase 2: Dual write** — New services produce to both the monolith (backward compatibility) and their domain topic. Existing consumers unchanged.

3. **Phase 3: Consumer migration** — Migrate consumers one-by-one from monolith to domain topics. Each migration is independent and reversible.

4. **Phase 4: Decommission** — Once all consumers are migrated, stop the dual write and decommission the monolith topic.

Key considerations: Schema Registry subjects change per topic, so register schemas in advance. Use `RecordNameStrategy` for schemas shared across topics. Each phase should be independently deployable and rollbackable."

---

### Q5: "Design a system where Kafka consumers need to process messages in strict global order."

**Model Answer:**

"Global ordering in Kafka is hard by design — ordering is only guaranteed within a partition. Options:

**Option A: Single partition** — Guarantees global order but limits throughput to one consumer. Viable only for low-throughput topics (<1K events/sec).

**Option B: Sequential processing layer** — Use Kafka for transport (multiple partitions for throughput), then a downstream service that buffers and reorders based on a sequence number or timestamp before processing. More complex but scalable.

**Option C: Challenge the requirement** — Most 'global ordering' requirements are actually 'per-entity ordering.' Does order-123's events really need to be ordered relative to order-456? Usually no. If per-entity ordering suffices, use entity ID as partition key.

I'd always challenge the requirement first. In 15 years I've seen exactly two legitimate global ordering needs — and both used a single-partition topic with a standby consumer for failover."

---

### Q6: "Design Kafka-based event sourcing for an order management system."

**Model Answer:**

"Event sourcing stores state as a sequence of events rather than current state. The Kafka implementation:

**Event store**: `order-events` topic, compacted + time-retained (90 days), key = `order_id`. Each event is a state transition: `OrderCreated`, `ItemAdded`, `PaymentReceived`, `Shipped`.

**Materialized views**: Consumer groups rebuild current state by replaying events. One CG materializes into PostgreSQL for queries, another into Elasticsearch for search, another into Redis for real-time lookups.

**Snapshots**: For orders with 50+ events, replaying is slow. Periodically publish snapshot events to a `order-snapshots` compacted topic. On startup, read snapshot first, then replay events after the snapshot.

**Key challenge**: Event schema evolution. Use Schema Registry with FULL_TRANSITIVE compatibility — every version must be readable by every other version since events live forever. Use Avro's union types for backward-compatible event type evolution."

---

### Q7: "How would you handle Kafka across multiple data centers?"

**Model Answer:**

"Three patterns:

**Active-Passive (simplest)**: One primary cluster, one DR cluster. Use MirrorMaker 2 (or Confluent Replicator) for async replication. RPO = replication lag (seconds). On failover, consumers start from replicated offsets. Challenge: offset translation between clusters.

**Active-Active (complex)**: Both clusters accept writes. Applications produce to local cluster. MirrorMaker 2 replicates bidirectionally with topic name prefixing (`dc1.order-events`, `dc2.order-events`). Consumers read from all prefixed topics. Challenge: deduplication, conflict resolution, circular replication prevention.

**Stretch cluster (lowest RPO)**: Single Kafka cluster across data centers with rack-aware replication. Each partition has replicas in both DCs. Zero RPO but requires low-latency inter-DC links (<10ms). Only viable with nearby DCs.

For ShopStream: Active-Passive for simplicity. RPO of ~5 seconds is acceptable. We'd use MirrorMaker 2 with `sync.topic.configs.enabled=true` and test failover monthly."

---

## Screen 2: Debugging Scenarios

### S1: "Consumer lag is growing steadily on 3 out of 12 partitions. Other partitions are fine."

**Diagnosis**: Uneven partition distribution — hot partitions. The 3 lagging partitions likely have disproportionately more data (hot keys).

**Investigation steps:**
1. Check partition sizes: `kafka-log-dirs.sh` — are the 3 partitions much larger?
2. Check message key distribution: are a few keys producing most traffic?
3. Check consumer processing time per partition — one consumer might be slower (GC, resource-constrained host)

**Fix**: If hot keys → implement a custom partitioner or add a random suffix to hot keys (`hot-key-{random}`) to spread load. If consumer issue → check the specific host's CPU, memory, GC logs.

---

### S2: "After a deployment, consumers are rebalancing every 60 seconds in a loop."

**Diagnosis**: Rebalancing storm — likely caused by consumers being evicted faster than they can rejoin.

**Root causes:**
1. `max.poll.interval.ms` too low + processing takes longer than allowed
2. `session.timeout.ms` too low + network/GC delays
3. Consumer failing to start properly, crashing, and restarting

**Investigation:**
```bash
# Check consumer group status
kafka-consumer-groups.sh --describe --group my-group
# Look for: consumers with no assigned partitions, frequent member ID changes

# Check application logs for:
# "Member X was removed from the group because it failed to send heartbeat"
# "Consumer poll timeout expired"
```

**Fix**: Increase `max.poll.interval.ms` to 300000 (5 min). Reduce `max.poll.records` to process less per poll. Use `group.instance.id` for static membership during rolling deploys.

---

### S3: "We lost messages in production. acks=1, replication.factor=3."

**Diagnosis**: With `acks=1`, only the leader acknowledges. If the leader dies immediately after acknowledging but BEFORE followers replicate, those messages are lost.

**Timeline:**
```
1. Producer sends message → Leader writes → ACK sent (acks=1)
2. Leader crashes BEFORE followers replicate
3. Follower elected as new leader — doesn't have the message
4. Message is LOST
```

**Fix**: Change to `acks=all` + `min.insync.replicas=2`. This guarantees at least 2 replicas (including leader) have the message before acknowledging. Trade-off: ~3-5ms additional latency per produce.

---

### S4: "Consumer group shows 'EMPTY' state but our application pods are running."

**Diagnosis**: Application pods are running but consumers aren't actually connecting to the group — or they're connecting to the wrong cluster/group.

**Investigation:**
1. Verify `bootstrap.servers` points to the correct cluster
2. Verify `group.id` matches what you're querying
3. Check for authentication errors in application logs (SASL/SSL misconfiguration)
4. Check network connectivity from pods to Kafka brokers

**Common cause**: Environment variable overrides — production pods reading staging Kafka config. Another common one: `auto.offset.reset=latest` with no historical data = consumer joins, poll returns nothing, app logs nothing, appears broken but technically working.

---

### S5: "Debezium CDC connector keeps restarting with 'WAL segment has already been removed' error."

**Diagnosis**: PostgreSQL removed WAL segments that Debezium hasn't consumed yet. The connector fell behind and the replication slot's WAL was recycled.

**Root causes:**
1. Connector was down too long (maintenance, crash)
2. `max_wal_size` too low in PostgreSQL
3. Heavy write load exceeded WAL retention

**Fix**: 
- Increase `max_wal_size` and `wal_keep_size` in PostgreSQL
- Set up monitoring on replication slot lag
- For recovery: drop and recreate the connector with `snapshot.mode=initial` to rebuild from a full table snapshot
- Long-term: set `slot.drop.on.stop=false` (default) and monitor replication slot size

---

### S6: "Producer is getting 'NOT_ENOUGH_REPLICAS' errors intermittently."

**Diagnosis**: `acks=all` with `min.insync.replicas=2` but only 1 replica is in-sync (the leader). The ISR has shrunk below the minimum.

**Investigation:**
```bash
kafka-topics.sh --describe --topic order-events
# Check ISR column — should show RF replicas, shows fewer

# Check under-replicated partitions
kafka-topics.sh --under-replicated-partitions
```

**Root causes:** A broker is down, a broker's disk is full, network partition between brokers, or followers can't keep up with write rate.

**Fix**: Immediate — identify the unhealthy broker and restore it. If disk full, add disk or lower retention. If follower can't keep up, check `replica.fetch.max.bytes` and network bandwidth. Temporary workaround (dangerous): lower `min.insync.replicas=1` but this sacrifices durability.

---

### S7: "Consumer processes records but offset commits keep failing with 'CommitFailedException'."

**Diagnosis**: The consumer was evicted from the group (rebalance happened) while processing, and it's trying to commit offsets for partitions it no longer owns.

**Timeline:**
```
1. Consumer polls batch of records
2. Processing takes 6 minutes (max.poll.interval.ms = 5 min)
3. Consumer evicted from group → rebalance → partitions reassigned
4. Original consumer finishes processing, tries to commit → FAIL
5. New consumer owner reprocesses same records → duplicates!
```

**Fix**: Increase `max.poll.interval.ms` or reduce `max.poll.records`. Implement a `ConsumerRebalanceListener` to detect partition revocation and stop processing. Consider async processing with a separate thread pool (but manage offset commits carefully).

---

### S8: "Kafka cluster disk usage is growing faster than expected despite retention.ms being set."

**Diagnosis**: Multiple possible causes:

1. **Log compaction backlog** — compacted topics retain ALL data until the cleaner catches up. Check `log.cleaner.threads` and cleaner lag.
2. **Consumer offsets topic** — `__consumer_offsets` grows if many consumer groups are created and abandoned.
3. **Transaction markers** — heavy transaction usage creates marker records that occupy space.
4. **Misconfigured retention** — `retention.bytes` not set, and `retention.ms` is longer than expected for the data volume.

**Fix**: Check per-topic disk usage with `kafka-log-dirs.sh`. Delete abandoned consumer groups. Increase `log.cleaner.threads` for compacted topics. Set both `retention.ms` AND `retention.bytes` as safety nets.

---

## Screen 3: Exactly-Once & Ordering Questions

### Q1: "Can you have exactly-once processing without Kafka transactions?"

**Answer**: "Yes — if your sink is **idempotent**. For example, writing to a database with an upsert (INSERT ON CONFLICT UPDATE) using the Kafka record's offset or a business key as the deduplication key. The consumer may process a record twice (at-least-once), but the idempotent sink ensures the effect is applied only once. This is simpler than Kafka transactions and works with any sink that supports idempotent writes. Many teams achieve 'effectively exactly-once' this way without the overhead of transactions."

---

### Q2: "You need messages ordered by customer_id but your topic has 12 partitions. Some customers generate 100x more traffic than others. How do you handle the hot partition?"

**Answer**: "The challenge: `customer_id` as partition key guarantees ordering but creates hot partitions for high-volume customers.

**Approach 1: Sub-key partitioning** — For hot customers, use `customer_id + order_id` as the key. This spreads one customer's orders across partitions. But you lose per-customer ordering. Acceptable if the downstream consumer only needs per-order ordering.

**Approach 2: Separate topic for hot customers** — Route VIP customers (top 1% by volume) to a dedicated topic with more partitions. Normal customers use the standard topic. Application-level routing based on a config map.

**Approach 3: Local ordering buffer** — Keep `customer_id` as key (accept hot partition), but scale the consumer vertically for the hot partition. Use a buffered processing pattern with in-memory reordering if needed.

The right answer depends on whether per-customer ordering is a hard business requirement or just a preference."

---

### Q3: "Explain the difference between 'exactly-once' within Kafka vs end-to-end exactly-once."

**Answer**: "**Within Kafka**: Idempotent producers + transactions guarantee that a consume-transform-produce pipeline within Kafka processes each record exactly once. Kafka manages this internally with producer IDs, sequence numbers, and transaction coordinators.

**End-to-end**: When your sink is external (database, API, file system), Kafka transactions alone aren't enough. The sink must participate in the exactly-once guarantee. Two approaches:

1. **Two-phase commit (2PC)**: Flink does this — pre-commit to the sink during checkpoint, full commit when checkpoint completes. Only works with sinks that support transactions (PostgreSQL, Kafka).

2. **Idempotent sink**: Write to the sink with a deduplication key (offset-based or business key). Even if you write twice, the result is the same. Works with any upsert-capable store.

The hard truth: true end-to-end exactly-once requires BOTH the source AND sink to cooperate. If your sink is a non-transactional REST API, the best you can achieve is at-least-once with client-side deduplication."

---

### Q4: "Your Kafka Streams application processes events and writes to a database. How do you prevent duplicates if the app crashes mid-transaction?"

**Answer**: "The **changelog + offset pattern**: Store the Kafka consumer offset alongside the business data in the same database transaction.

```
BEGIN TRANSACTION;
  INSERT INTO orders (order_id, ...) VALUES (...);
  UPDATE consumer_offsets SET offset = 42
    WHERE topic = 'order-events' AND partition = 3;
COMMIT;
```

On restart, read the stored offset from the database and seek the consumer to that position. This makes the database the source of truth for consumption progress — not Kafka's `__consumer_offsets`. If the app crashes after database commit but before Kafka offset commit, we simply reprocess — but the database transaction is already committed, so we detect the duplicate via the business key and skip it."

---

### Q5: "If `max.in.flight.requests.per.connection=5` with idempotence enabled, how does Kafka maintain ordering?"

**Answer**: "With idempotence, the broker tracks sequence numbers per producer per partition. Even with 5 in-flight requests, if request 3 arrives before request 2 (out-of-order delivery), the broker buffers request 3 and waits for request 2. Once request 2 arrives and is written, request 3 is then written in order.

Without idempotence, `max.in.flight > 1` can cause ordering issues if a retry of batch 1 arrives after batch 2 has been written. That's why the old guidance was `max.in.flight=1` — but with idempotence enabled, `max.in.flight=5` is safe for ordering AND gives better throughput."

---

### Q6: "A consumer reads from a compacted topic. Can it see duplicate records for the same key?"

**Answer**: "Yes! Compaction is an **asynchronous background process**, not instantaneous. Between compaction runs, the topic can contain multiple records for the same key. A consumer reading the topic may see: `[k1:v1, k1:v2, k1:v3]` before compaction runs. After compaction: `[k1:v3]`.

Additionally, the **active segment** (currently being written to) is never compacted — only closed (inactive) segments. So recent duplicates always exist until the segment rolls and compaction runs.

Consumers of compacted topics should always treat records as upserts — apply the latest value per key and handle duplicates gracefully."

---

## Screen 4: Flink Challenges

### F1: "Your Flink job's watermark is stuck. Events are being consumed but no windows are firing. What's wrong?"

**Answer**: "A stuck watermark means Flink's perception of event time isn't advancing. Common causes:

1. **Idle source partition**: One Kafka partition has no data. Flink's watermark is the minimum across all partitions — one idle partition holds back the entire job. Fix: configure `withIdleness(Duration.ofMinutes(5))` on the watermark strategy.

2. **Timestamp extraction returning wrong values**: Extracting seconds instead of milliseconds, or a null timestamp defaulting to 0. Fix: verify the timestamp assigner is returning millisecond epoch.

3. **Single key dominating with monotonous watermarks**: If all events have the same key and arrive in order, but there's one stream with an old event, that stream holds back the watermark.

Debugging: Check Flink's web UI → watermarks tab. It shows per-subtask watermarks. The lowest one is your bottleneck."

---

### F2: "Your Flink job has 200GB of state and checkpoints are taking 10 minutes, causing backpressure. How do you fix it?"

**Answer**: "Three optimizations:

1. **Enable incremental checkpoints** (RocksDB only): Instead of snapshotting all 200GB, only the delta since the last checkpoint is saved. This can reduce checkpoint size from 200GB to a few GB.

2. **Increase checkpoint interval**: If checkpoints take 10 minutes, don't set the interval to 5 minutes. Set it to 15-20 minutes with `minPauseBetweenCheckpoints = 5 minutes`.

3. **Use local recovery**: Enable `state.backend.local-recovery=true` — TaskManagers keep a local copy of state, so recovery doesn't need to download 200GB from S3.

4. **Tune RocksDB**: Increase `state.backend.rocksdb.block.cache-size`, enable `state.backend.rocksdb.metrics.num-running-compactions` monitoring.

5. **State TTL**: Review if all 200GB is necessary. Often, old state entries should expire. Add TTL to state descriptors."

---

### F3: "You have two Kafka topics that need to be joined by user_id. One has 50 events/sec, the other has 5000/sec. How do you handle the skew?"

**Answer**: "This is a classic **data skew** problem in stream joins. The join is keyed by `user_id`, so if a few users generate most of the traffic on the high-volume stream, those keys create hot subtasks.

**Solutions:**

1. **Pre-aggregate the high-volume stream**: Before the join, window-aggregate the 5000/sec stream into 1-minute summaries per user. This reduces the event rate before the join.

2. **Broadcast the small stream**: If the 50/sec stream is small enough, use a `BroadcastStream` — broadcast it to all parallel operators and join locally. Avoids the shuffle entirely.

3. **Salting + secondary join**: For specific hot keys, add a random salt (0-9) to the key, join 10 copies of the small stream's record against each salted key, then de-duplicate.

I'd start with the broadcast approach since 50/sec × small event size easily fits in memory."

---

### F4: "Explain how you'd implement a real-time leaderboard (top 10 products by sales) in Flink."

**Answer**: "
```
order-events → KeyBy(product_id)
    → Sliding Window (1 hour, slide 1 min)
    → Sum(amount) per product
    → WindowAll (global aggregation)
    → Top-N sort (keep top 10)
    → Sink (Redis/API)
```

The challenge is the `WindowAll` — it's a single-parallelism bottleneck. For a leaderboard, this is usually acceptable because the input is already aggregated (one record per product per window).

For higher throughput: pre-aggregate in parallel windows (per product), then use a `ProcessAllWindowFunction` with parallelism=1 for the top-N selection. The pre-aggregation reduces the single-operator bottleneck to just sorting N product summaries, not processing raw events.

State consideration: The sliding window (1hr/1min) means 60 overlapping windows per key — monitor state size."

---

### F5: "Your Flink job needs to enrich order events with user profile data from a REST API. How do you do this without blocking?"

**Answer**: "Use Flink's **AsyncIO** operator. It issues non-blocking REST calls and processes responses as they arrive, maintaining high throughput.

```python
class AsyncUserEnricher(AsyncFunction):
    async def async_invoke(self, event, result_future):
        user_profile = await http_client.get(f'/users/{event.user_id}')
        enriched = EnrichedOrder(event, user_profile)
        result_future.complete([enriched])

# Apply async I/O with ordering guarantee
enriched_stream = AsyncDataStream.unordered_wait(
    order_stream,
    AsyncUserEnricher(),
    timeout=5000,      # 5 second timeout per request
    capacity=100       # Max 100 concurrent requests
)
```

**Ordered vs Unordered**: `ordered_wait` preserves event order (buffers results) — use for order-sensitive pipelines. `unordered_wait` emits results as they complete — higher throughput but out-of-order.

**Better alternative for hot path**: Cache user profiles in Flink state (refreshed periodically) or use a broadcast stream of user profile changes. REST calls per event don't scale well at 50K events/sec."

---

### F6: "How would you handle schema evolution in a running Flink job without downtime?"

**Answer**: "Use savepoints for schema changes:

1. Take a savepoint of the running job
2. Update the Flink job code with new schema handling
3. Update the Schema Registry with the new (compatible) schema version
4. Redeploy the job from the savepoint

For state schema evolution: Flink supports state migration if you use Avro or POJO state serializers. If you change a state class (add a field), Flink can migrate the existing state during savepoint restore.

For breaking state changes: Start a new job with fresh state, consuming from a historical offset to rebuild. Run old and new jobs in parallel briefly to verify correctness, then stop the old job.

Critical rule: never change the `uid()` of a stateful operator between versions — Flink maps savepoint state to operators by UID."

---

## Screen 5: Rapid Fire — 20 Quick Q&A

**1. What is a Kafka consumer group?**
A set of consumers that cooperate to consume from a topic. Each partition is read by exactly one consumer in the group.

**2. What happens when a Kafka broker dies?**
Partitions where it was leader get new leaders elected from ISR. Producers/consumers reconnect to new leaders. No data loss if RF>1 and ISR maintained.

**3. What is ISR?**
In-Sync Replicas — the set of replicas (including leader) that are fully caught up. `acks=all` waits for all ISR members.

**4. What is `auto.offset.reset`?**
What consumers do when there's no committed offset: `earliest` (read from beginning), `latest` (read only new messages), `none` (throw error).

**5. What is log compaction?**
Kafka retains only the latest record per key, removing older duplicates. Used for stateful topics (inventory, config, user profiles).

**6. What is a tombstone in Kafka?**
A record with a valid key but `null` value. After compaction, this tells Kafka to eventually delete the key entirely.

**7. What is `min.insync.replicas`?**
Minimum number of replicas that must acknowledge a write when `acks=all`. If ISR < min.insync, writes fail with `NOT_ENOUGH_REPLICAS`.

**8. How does Kafka ensure ordering?**
Ordering is guaranteed within a partition. Use a consistent message key to route related messages to the same partition.

**9. What is a Flink watermark?**
A timestamp assertion: "No events with time ≤ W will arrive." Used to trigger windows and track event-time progress.

**10. What is backpressure in Flink?**
When a downstream operator can't keep up, it signals upstream to slow down. Flink handles this naturally through its credit-based flow control.

**11. What is operator chaining in Flink?**
Combining compatible operators (map + filter) into a single task to avoid serialization overhead. Broken at shuffle boundaries (keyBy).

**12. What is a Flink savepoint?**
A manual, portable snapshot of job state. Used for planned operations: upgrades, rescaling, migration.

**13. What is the Outbox Pattern?**
Write business data + event to the database in one transaction. CDC captures the outbox table and publishes to Kafka. Avoids dual-write inconsistency.

**14. What is Debezium?**
An open-source CDC platform. Reads database WAL/binlog and produces change events to Kafka. Supports PostgreSQL, MySQL, MongoDB, etc.

**15. What is `read_committed` isolation?**
Consumer setting that filters out records from aborted Kafka transactions. Required for end-to-end exactly-once.

**16. What is a Flink checkpoint barrier?**
A special marker injected into the data stream. When an operator receives barriers from all inputs, it snapshots its state. Based on Chandy-Lamport algorithm.

**17. What is `session.timeout.ms`?**
How long a consumer can go without sending a heartbeat before being evicted from the group. Default: 45s (Kafka 3.x).

**18. What is RocksDB in Flink?**
An embedded key-value store used as a state backend. Stores state on local disk, supports incremental checkpoints, handles terabytes of state.

**19. What is Schema Registry compatibility?**
Rules governing how schemas can evolve. BACKWARD: new schema reads old data. FORWARD: old schema reads new data. FULL: both directions.

**20. What is MirrorMaker 2?**
Kafka's cross-cluster replication tool. Replicates topics, consumer offsets, and ACLs between clusters. Used for disaster recovery and geo-replication.

---

## Screen 6: Technology Choice Scenarios

### TC1: "Your team processes batch data with Spark. Now you need real-time analytics with sub-second latency. Flink or Spark Structured Streaming?"

**Answer**: "If sub-second latency is a hard requirement, **Flink**. Spark Structured Streaming's micro-batch model adds inherent latency (100ms+ minimum, typically 1-5 seconds in practice). Flink processes per-event with single-digit millisecond latency.

However, if the team has deep Spark expertise and the 'sub-second' requirement has flexibility (1-2 seconds acceptable), Spark Structured Streaming reduces operational complexity — one framework for both batch and streaming. The training cost and operational overhead of running two frameworks (Spark + Flink) is significant.

My recommendation: Start with Spark Streaming. If latency proves insufficient, introduce Flink for the specific sub-second pipelines while keeping Spark for batch and relaxed-latency streaming."

---

### TC2: "You need a message queue for task distribution across 50 worker pods. Kafka, SQS, or RabbitMQ?"

**Answer**: "**SQS** (or RabbitMQ if on-prem). This is a task queue pattern, not an event streaming pattern.

Kafka is optimized for event streaming — ordered logs, consumer groups, replay. For task distribution, you want: competing consumers (any worker picks up any task), message-level acknowledgment, automatic retry/DLQ, and no concern about ordering.

SQS natively supports: visibility timeouts (task reservation), DLQ for failed tasks, auto-scaling with queue depth metrics, and message-level deletion (not offset-based). It's serverless — zero operational overhead.

Kafka would work but adds unnecessary complexity: you'd need to manage offsets carefully, handle partition-to-consumer binding (tasks aren't evenly distributed), and build your own retry/DLQ logic.

RabbitMQ if you need on-prem, priority queues, or complex routing (exchanges, bindings)."

---

### TC3: "You're building an event-driven microservices architecture. Should events go through Kafka, or use direct REST calls between services?"

**Answer**: "**Kafka for asynchronous events, REST for synchronous queries.**

Use Kafka when: service A doesn't need an immediate response, multiple services need the same event, you want temporal decoupling (producer/consumer don't need to be up simultaneously), or you want event replay for debugging/reprocessing.

Use REST when: the caller needs an immediate response to proceed, the interaction is request-reply (get user profile, validate payment), or the call is between tightly-coupled services in the same bounded context.

Common hybrid pattern:
```
Order Service →(REST)→ Payment Service (sync: need result now)
Order Service →(Kafka)→ Notification Service (async: email later)
Order Service →(Kafka)→ Analytics Service (async: dashboard update)
Order Service →(Kafka)→ Inventory Service (async: stock deduction)
```

Anti-pattern: Using Kafka as a synchronous request-reply mechanism. It adds latency, complexity, and correlation ID management overhead."

---

### TC4: "You need to replicate database changes to a data lake in real-time. Kafka + Debezium, or a direct database-to-lake pipeline?"

**Answer**: "**Kafka + Debezium** for production systems. Three advantages:

1. **Decoupling**: The database doesn't know about the data lake. If the lake goes down, Kafka buffers events (retention). Direct pipelines couple the database to every downstream system.

2. **Fan-out**: Once changes are in Kafka, multiple consumers can use them — data lake, search index, cache invalidation, analytics. With direct pipelines, you'd need separate extractors for each.

3. **Minimal database impact**: Debezium reads the WAL — zero performance impact on the database. Direct connectors (JDBC polling) add query load and miss deletes.

Direct pipeline only makes sense for: simple one-off migration, very low volume (<100 changes/day), or when operational simplicity outweighs architectural benefits (small team, non-critical system)."

---

### TC5: "You have a Flink job processing 100K events/sec with 500GB of state. Should you use managed Flink (e.g., AWS Kinesis Data Analytics / Amazon Managed Flink) or self-managed?"

**Answer**: "At 100K events/sec and 500GB of state, **self-managed** or **Confluent-managed Flink** (if available). Here's why:

Managed Flink services (AWS) have limitations at this scale: state size caps, limited RocksDB tuning, checkpoint storage constraints, and scaling isn't instant. You also lose control over exactly which Flink version, state backend configuration, and JVM tuning you use.

Self-managed on Kubernetes (Flink Kubernetes Operator) gives you: full RocksDB configuration, incremental checkpoints to S3, custom metrics, and fine-grained resource allocation. You need a strong platform team.

**Middle ground**: Start managed for faster time-to-market. When you hit limitations (and you will at 500GB state), migrate to self-managed. Use savepoints for seamless migration.

Key metric: Can your team handle Kubernetes + Flink operations? If not, start managed and accept the constraints."

---

### TC6: "You're designing a system that needs to process financial transactions with exactly-once and must comply with audit requirements. What technology stack?"

**Answer**: "**Kafka (EOS) + Flink + PostgreSQL** with comprehensive audit:

- **Kafka**: `acks=all`, `min.insync.replicas=2`, `enable.idempotence=true`, transactions for cross-topic writes. Schema Registry with FULL_TRANSITIVE compatibility. Long retention (1 year) for the transaction topic — it IS your audit log.

- **Flink**: Exactly-once checkpointing with RocksDB state backend. Two-phase commit sinks to PostgreSQL. Savepoints before any change.

- **PostgreSQL**: Idempotent upserts using transaction ID as dedup key. Write-ahead logging for durability. Point-in-time recovery enabled.

- **Audit trail**: Kafka's immutable log IS the audit trail. Every transaction event is permanently recorded with producer metadata (headers: service, version, timestamp). Consumer offsets track exactly what was processed.

- **Monitoring**: Exactly-once verification — periodically reconcile Kafka offset positions with database record counts. Alert on any discrepancy.

This is NOT over-engineering for financial data — regulators will thank you."

---

## Final Key Takeaways — The Interview Gauntlet

1. **Structure your answers**: Situation → Approach → Trade-offs. Interviewers reward structured thinking.
2. **Always discuss trade-offs**: There's no perfect solution — show you understand the costs.
3. **Draw diagrams**: Even in verbal interviews, describe your architecture visually ("imagine three boxes...").
4. **Challenge requirements**: "Do you really need global ordering, or per-entity ordering?" shows senior thinking.
5. **Know the numbers**: Partition throughput (~10MB/s), checkpoint overhead (~5%), EOS latency impact (~5-10%).
6. **Lead with the simple solution**: "I'd start with X, and if we hit limitation Y, I'd evolve to Z."
7. **Name real tools**: "I'd use Debezium for CDC" beats "I'd use a change capture tool." Specificity signals experience.
8. **Admit knowledge boundaries**: "I haven't used Pulsar in production, but my understanding is..." beats making things up.
9. **Connect to business value**: "This prevents duplicate charges to customers" beats "this achieves exactly-once."
10. **Practice the rapid-fire section**: If you can answer all 20 in under 30 seconds each, you're interview-ready.
