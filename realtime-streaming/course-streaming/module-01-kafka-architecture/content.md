# Module 1: Kafka Architecture & Core Concepts

> **Scenario Throughout**: You're building the streaming backbone for **ShopStream**, a large e-commerce platform processing 500K orders/day, real-time clickstream analytics, and live inventory updates across 200 warehouses.

---

## Screen 1: Topics — The Logical Backbone

A **topic** in Kafka is a named, logical channel to which producers write records and from which consumers read. Think of it as an infinitely growing, append-only, immutable log. Once a record is written, it cannot be modified or deleted (until retention kicks in). This immutability is fundamental — it's what makes Kafka so powerful for replay, auditing, and event sourcing.

```
┌──────────────────────────────────────────────────────┐
│                   Kafka Cluster                      │
│                                                      │
│   Topic: "order-events"                              │
│   ┌────┬────┬────┬────┬────┬────┬────┬────┐         │
│   │ r0 │ r1 │ r2 │ r3 │ r4 │ r5 │ r6 │ r7 │→ ...   │
│   └────┴────┴────┴────┴────┴────┴────┴────┘         │
│     ↑                                  ↑             │
│   oldest                            newest           │
│                                                      │
│   Topic: "clickstream"                               │
│   ┌────┬────┬────┬────┬────┬────┐                    │
│   │ r0 │ r1 │ r2 │ r3 │ r4 │ r5 │→ ...              │
│   └────┴────┴────┴────┴────┴────┘                    │
│                                                      │
│   Topic: "inventory-updates"                         │
│   ┌────┬────┬────┬────┬────┬────┬────┐               │
│   │ r0 │ r1 │ r2 │ r3 │ r4 │ r5 │ r6 │→ ...         │
│   └────┴────┴────┴────┴────┴────┴────┘               │
└──────────────────────────────────────────────────────┘
```

**ShopStream example**: You'd create separate topics for `order-events`, `clickstream`, `inventory-updates`, and `payment-results`. Each topic isolates a domain concern. A single Kafka cluster can host thousands of topics.

**Key properties of topics:**

| Property | Description |
|---|---|
| Append-only | Records are always added to the end |
| Immutable | Once written, records cannot change |
| Durable | Persisted to disk, replicated |
| Retention-based | Old records removed by time or size |
| Ordered per partition | Not globally ordered across topic |

Topics are **logical** — they don't map 1:1 to physical files. Under the hood, each topic is split into **partitions**, and each partition maps to a set of log segment files on disk. The topic is the abstraction; partitions are the physical reality.

**Naming conventions matter.** In production, use namespaced names: `shopstream.orders.v1`, `shopstream.clicks.raw`, `shopstream.inventory.updates`. This makes ACL management, monitoring, and schema registry organization much cleaner.

### 💡 Interview Insight

> **Q: "What is a Kafka topic?"**
> "A topic is a named, append-only, immutable log that acts as a logical channel for a category of events. Producers publish records to topics, and consumers subscribe to them. Topics are split into partitions for parallelism and are persisted to disk with configurable retention. The immutability guarantee is what enables Kafka's replay capability — any consumer can re-read the entire history of a topic from any offset."

---

## Screen 2: Partitions — The Unit of Parallelism

Partitions are where Kafka's scalability magic happens. Each topic is divided into one or more partitions, and each partition is an independent, ordered, append-only log. **Ordering is guaranteed within a partition, but NOT across partitions.**

```
Topic: "order-events" (3 partitions)

  Partition 0: [r0] [r1] [r2] [r3] [r4] → ...
  Partition 1: [r0] [r1] [r2] [r3] → ...
  Partition 2: [r0] [r1] [r2] [r3] [r4] [r5] → ...
                 ↑                          ↑
               offset 0                  latest offset

  Key "order-123" → hash("order-123") % 3 = Partition 1
  Key "order-456" → hash("order-456") % 3 = Partition 0
  Key "order-789" → hash("order-789") % 3 = Partition 2
```

**Why partitions matter for ShopStream:**

1. **Parallelism**: If `order-events` has 12 partitions, you can have up to 12 consumers reading in parallel.
2. **Ordering**: By using `order_id` as the message key, all events for order-123 (created, paid, shipped, delivered) land in the same partition and are read in order.
3. **Throughput**: Each partition can handle ~10 MB/s writes. 12 partitions = ~120 MB/s total throughput.

**Choosing partition count** is a critical design decision:

| Factor | Guidance |
|---|---|
| Consumer parallelism | Partitions ≥ max consumers |
| Throughput target | More partitions = more throughput |
| Ordering requirements | Events needing order share a key |
| End-to-end latency | More partitions = slightly more |
| Rebalancing time | More partitions = slower rebalance |
| Open file handles | Each partition = ~2 file handles |

**Rule of thumb**: Start with `max(throughput_needs / partition_throughput, expected_max_consumers)`. For ShopStream's order-events at 500K orders/day (~6 orders/sec), 6–12 partitions is reasonable. For clickstream at 50K events/sec, you might want 30–50 partitions.

**Critical rule**: You can **increase** partitions later, but you can **never decrease** them. And increasing partitions breaks key-based ordering guarantees for existing keys. Plan carefully.

### 💡 Interview Insight

> **Q: "How do you decide the number of partitions for a topic?"**
> "I consider three factors: (1) target throughput divided by per-partition throughput gives minimum partitions, (2) maximum consumer parallelism needed — you can't have more active consumers than partitions, and (3) ordering requirements — all events that must be ordered together share a partition key. I also factor in that partitions can only be increased (never decreased), and increasing them breaks existing key-based routing. For a high-throughput clickstream topic, I'd start with 30+ partitions; for a lower-volume order-events topic, 6–12 is usually sufficient."

---

## Screen 3: Brokers & Replication

A **broker** is a single Kafka server. A Kafka **cluster** is a group of brokers working together. Each broker stores a subset of partitions. Replication copies each partition across multiple brokers for fault tolerance.

```
Kafka Cluster (3 Brokers)

Topic: "order-events" — 3 partitions, replication factor 3

Broker 1                 Broker 2                 Broker 3
┌──────────────┐        ┌──────────────┐        ┌──────────────┐
│ P0 (LEADER)  │        │ P0 (follower)│        │ P0 (follower)│
│ P1 (follower)│        │ P1 (LEADER)  │        │ P1 (follower)│
│ P2 (follower)│        │ P2 (follower)│        │ P2 (LEADER)  │
└──────────────┘        └──────────────┘        └──────────────┘

  Write to P0 → Broker 1 (leader) → replicates to Broker 2, 3
  Read from P0 → Broker 1 (leader) by default
```

**Leader vs Follower:**
- Each partition has exactly **one leader** and zero or more **followers**.
- All reads and writes go through the leader (by default — KIP-392 added follower reads).
- Followers passively replicate from the leader to stay in sync.
- If the leader dies, one of the in-sync followers is elected as the new leader.

**In-Sync Replicas (ISR):**

The ISR set contains the leader plus all followers that are "caught up" (within `replica.lag.time.max.ms`, default 30 seconds). A follower that falls behind is removed from ISR. This is critical because `acks=all` means "all ISR members acknowledged," not "all replicas."

```
Normal state:     ISR(P0) = {Broker1, Broker2, Broker3}
Broker 3 slow:    ISR(P0) = {Broker1, Broker2}           ← Broker 3 removed
Broker 3 catches up: ISR(P0) = {Broker1, Broker2, Broker3}  ← re-added
```

**The `acks` setting** (producer-side) controls durability:

| Setting | Behavior | Durability | Latency |
|---|---|---|---|
| `acks=0` | Fire and forget, don't wait | Lowest — may lose data | ~0.5ms |
| `acks=1` | Wait for leader only | Medium — loses if leader dies | ~2ms |
| `acks=all` | Wait for all ISR replicas | Highest — survives broker loss | ~5-10ms |

**ShopStream guidance**: Use `acks=all` for `order-events` and `payment-results` (money matters!). Use `acks=1` for `clickstream` (some loss acceptable for lower latency).

Set `min.insync.replicas=2` with `acks=all` and `replication.factor=3`. This means at least 2 replicas must acknowledge, and the cluster can tolerate 1 broker failure without data loss OR unavailability.

### 💡 Interview Insight

> **Q: "How does Kafka ensure data durability?"**
> "Through replication. Each partition is replicated across multiple brokers — typically RF=3. One replica is the leader (handles reads/writes), and followers replicate passively. The ISR (In-Sync Replicas) tracks which followers are caught up. With `acks=all`, a produce request only succeeds when all ISR members acknowledge. Combined with `min.insync.replicas=2`, this guarantees data survives any single broker failure. The trade-off is slightly higher write latency (~5-10ms vs ~2ms for `acks=1`)."

---

## Screen 4: Producers — Publishing Messages

Producers are client applications that publish records to Kafka topics. A record consists of a **key** (optional), a **value** (the payload), a **timestamp**, and optional **headers**.

```
Producer Record Structure:
┌───────────────────────────────────────────┐
│  Key: "order-123"    (byte[])             │
│  Value: {"event":"created","amount":99.50}│
│  Timestamp: 1712345678000                 │
│  Headers: [("source","checkout-svc")]     │
│  Topic: "order-events"                    │
│  Partition: null (let Kafka decide)       │
└───────────────────────────────────────────┘
```

**Partition selection logic** inside the producer:

```
if record.partition is explicitly set:
    → use that partition
elif record.key is not null:
    → partition = hash(key) % num_partitions     # deterministic!
else:
    → sticky partitioning (batch to one partition, then rotate)
```

**Key-based partitioning** is essential for ShopStream orders. By using `order_id` as the key, all lifecycle events (created → paid → shipped → delivered) land in the same partition, preserving order.

**Producer internals — the batching pipeline:**

```
┌──────────┐     ┌──────────────────────┐     ┌──────────┐
│  send()  │ →   │  RecordAccumulator   │ →   │  Sender  │ → Broker
│ (async)  │     │  (batches per        │     │ (network │
│          │     │   partition)         │     │  thread)  │
└──────────┘     └──────────────────────┘     └──────────┘

  batch.size = 16384 (16KB default)
  linger.ms = 0 (send immediately by default)
```

**Critical producer configs for ShopStream:**

| Config | Value | Why |
|---|---|---|
| `acks` | `all` | Durability for orders |
| `retries` | `Integer.MAX_VALUE` | Retry on transient failure |
| `enable.idempotence` | `true` | Prevent duplicates on retry |
| `max.in.flight.requests` | `5` | Allows batching with idempotence |
| `linger.ms` | `5` | Wait 5ms to batch more records |
| `batch.size` | `32768` | 32KB batches for throughput |
| `compression.type` | `lz4` | Good compression/speed ratio |

**Idempotent producers** solve a subtle problem: if a producer sends a record, the broker writes it, but the ACK is lost (network blip), the producer retries and creates a **duplicate**. With `enable.idempotence=true`, the producer attaches a sequence number — the broker deduplicates retries. This is the foundation of Exactly-Once Semantics.

```python
from confluent_kafka import Producer

config = {
    'bootstrap.servers': 'kafka-1:9092,kafka-2:9092,kafka-3:9092',
    'acks': 'all',
    'enable.idempotence': True,
    'compression.type': 'lz4',
    'linger.ms': 5,
}
producer = Producer(config)

def on_delivery(err, msg):
    if err:
        print(f"Delivery failed: {err}")
    else:
        print(f"Delivered to {msg.topic()}[{msg.partition()}] @ offset {msg.offset()}")

producer.produce(
    topic='order-events',
    key='order-123',
    value='{"event":"created","amount":99.50}',
    callback=on_delivery,
)
producer.flush()  # Wait for all messages to be delivered
```

### 💡 Interview Insight

> **Q: "How does Kafka's producer decide which partition to send a message to?"**
> "If a partition is explicitly specified, that's used. If a key is provided, Kafka hashes the key (murmur2 hash) modulo the number of partitions — this guarantees all records with the same key go to the same partition, preserving per-key ordering. If no key is provided, Kafka uses 'sticky partitioning' — it fills a batch to one partition, then rotates to the next for better batching efficiency. In our e-commerce system, we use `order_id` as the key so all events for a single order are ordered."

---

## Screen 5: Consumers & Consumer Groups

Consumers **pull** records from Kafka (not push!). This pull model lets each consumer control its own read rate, making backpressure natural. Consumers are organized into **consumer groups** for parallel processing.

```
Topic: "order-events" (4 partitions)

Consumer Group: "order-processing-service"

  ┌────────────┐    ┌────────────┐    ┌────────────┐    ┌────────────┐
  │ Partition 0│    │ Partition 1│    │ Partition 2│    │ Partition 3│
  └─────┬──────┘    └─────┬──────┘    └─────┬──────┘    └─────┬──────┘
        │                 │                 │                 │
        ▼                 ▼                 ▼                 ▼
  ┌───────────┐    ┌───────────┐    ┌───────────┐    ┌───────────┐
  │Consumer A │    │Consumer B │    │Consumer C │    │Consumer D │
  └───────────┘    └───────────┘    └───────────┘    └───────────┘

  Rule: Each partition → exactly 1 consumer in a group
        Each consumer  → can read from 1+ partitions
```

**The golden rule**: Within a consumer group, each partition is assigned to **exactly one** consumer. This means:
- 4 partitions + 4 consumers = 1 partition each (ideal)
- 4 partitions + 2 consumers = 2 partitions each
- 4 partitions + 6 consumers = 4 active + 2 **idle** (wasted!)

**Multiple consumer groups** can read the same topic independently. Each group maintains its own offsets:

```
Topic: "order-events"
  │
  ├── Consumer Group "order-processing" → reads for fulfillment
  ├── Consumer Group "analytics-pipeline" → reads for dashboards
  └── Consumer Group "fraud-detection" → reads for fraud scoring

  Each group reads ALL records independently (pub/sub pattern)
```

**Offset management** — consumers track their position per partition:

```
Partition 0: [r0][r1][r2][r3][r4][r5][r6][r7][r8][r9]
                                  ↑              ↑
                           committed offset   latest offset
                           (last processed)   (LOG-END)

Consumer lag = latest offset - committed offset = 9 - 5 = 4
```

Offsets are committed to an internal topic `__consumer_offsets`. Two commit strategies:

| Strategy | How | Risk |
|---|---|---|
| Auto-commit (`enable.auto.commit=true`) | Commits every 5s automatically | May lose records on crash |
| Manual commit | Call `commitSync()` or `commitAsync()` | More control, must handle |

**For ShopStream order processing**, use manual commit: process the record, update the database, **then** commit the offset. This gives at-least-once semantics — you might process a record twice on crash, but you'll never skip one.

```python
from confluent_kafka import Consumer

config = {
    'bootstrap.servers': 'kafka-1:9092',
    'group.id': 'order-processing-service',
    'auto.offset.reset': 'earliest',
    'enable.auto.commit': False,  # Manual commit!
}
consumer = Consumer(config)
consumer.subscribe(['order-events'])

while True:
    msg = consumer.poll(timeout=1.0)
    if msg is None:
        continue
    if msg.error():
        handle_error(msg.error())
        continue

    process_order(msg.value())     # Step 1: Process
    consumer.commit(message=msg)   # Step 2: Commit offset
```

### 💡 Interview Insight

> **Q: "What happens if you have more consumers than partitions in a group?"**
> "The extra consumers sit idle. Kafka's rule is: each partition is assigned to exactly one consumer within a group. So with 4 partitions and 6 consumers, only 4 are active. The idle ones serve as hot standbys — if an active consumer dies, an idle one picks up its partition during rebalancing. This is why partition count sets the ceiling for consumer parallelism."

---

## Screen 6: Consumer Rebalancing

**Rebalancing** is the process of redistributing partitions among consumers in a group when membership changes. It happens when a consumer joins, leaves, crashes, or when topic partitions change.

```
BEFORE: 4 partitions, 2 consumers
  Consumer A → [P0, P1]
  Consumer B → [P2, P3]

Consumer C joins → REBALANCE TRIGGERED

AFTER: 4 partitions, 3 consumers
  Consumer A → [P0]
  Consumer B → [P2]
  Consumer C → [P1, P3]
```

**Rebalancing strategies:**

| Strategy | Behavior | Impact |
|---|---|---|
| Eager (legacy) | Revoke ALL, then reassign | Full stop-the-world! |
| Cooperative Sticky | Revoke only changed partitions | Minimal disruption |
| Static Membership | Uses `group.instance.id`, no rebal on restart | Best for rolling deploys |

**Eager rebalancing** is painful — during the rebalance window (can be 10-30 seconds), **no consumer processes any messages**. For ShopStream processing 500K orders/day, that's ~350 orders missed per 30-second rebalance!

**Cooperative Sticky rebalancing** (default since Kafka 3.x) is much better:

```
Eager Rebalance:
  Time 0: STOP ALL consumers → revoke all partitions
  Time 1: Reassign all partitions
  Time 2: Resume all consumers
  ──── Full pause: ~10-30 seconds ────

Cooperative Sticky Rebalance:
  Time 0: Only revoke partitions that need to MOVE
  Time 1: Reassign those partitions to new owners
  Time 2: Other consumers NEVER STOPPED
  ──── Partial pause: ~2-5 seconds, most consumers unaffected ────
```

**Static Group Membership** eliminates rebalancing during rolling deployments. Each consumer instance gets a persistent `group.instance.id`. When a consumer restarts (same ID), Kafka recognizes it and skips rebalancing:

```python
config = {
    'group.id': 'order-processing-service',
    'group.instance.id': 'order-processor-pod-3',  # Static!
    'partition.assignment.strategy': 'cooperative-sticky',
    'session.timeout.ms': 45000,
}
```

**Causes of unexpected rebalancing** (common production headache):
1. Long GC pauses → consumer misses heartbeat → kicked from group
2. `max.poll.interval.ms` exceeded → processing too slow → kicked
3. Network blip → missed heartbeat
4. Rolling deployment without static membership

**Tuning tips for ShopStream:**
- Set `session.timeout.ms=45000` (default 10s is aggressive)
- Set `heartbeat.interval.ms=15000` (1/3 of session timeout)
- Set `max.poll.interval.ms=300000` (5 min for slow processing)
- Use `max.poll.records=100` to limit batch size per poll

### 💡 Interview Insight

> **Q: "How would you minimize rebalancing impact in production?"**
> "Three strategies: (1) Use **cooperative sticky** assignment so only affected partitions are revoked, not all. (2) Use **static group membership** with `group.instance.id` for rolling deployments — the broker recognizes a restarted consumer and skips rebalancing. (3) Tune `session.timeout.ms` and `max.poll.interval.ms` to avoid false evictions from GC pauses or slow processing. In critical systems, I also monitor the `rebalance-latency-avg` metric to catch problematic rebalancing patterns."

---

## Screen 7: ZooKeeper (Legacy) vs KRaft

Kafka historically depended on **ZooKeeper** for cluster metadata, broker registration, leader election, and topic configuration. Starting with Kafka 3.3 (production-ready), the **KRaft** (Kafka Raft) mode replaces ZooKeeper entirely.

```
LEGACY (with ZooKeeper):

  ┌─────────────────────┐
  │  ZooKeeper Ensemble  │
  │  (3-5 nodes)         │
  │  • Broker registry   │
  │  • Controller elect  │
  │  • Topic metadata    │
  │  • ACLs              │
  └──────────┬──────────┘
             │ watches + updates
  ┌──────────┴──────────────────────┐
  │         Kafka Brokers           │
  │  Broker 1   Broker 2   Broker 3 │
  │  (controller)                   │
  └─────────────────────────────────┘


KRaft MODE (no ZooKeeper):

  ┌─────────────────────────────────┐
  │         Kafka Brokers           │
  │  Broker 1   Broker 2   Broker 3 │
  │  (voter)    (voter)    (voter)  │
  │                                 │
  │  Controller quorum built-in     │
  │  Metadata stored in internal    │
  │  __cluster_metadata topic       │
  └─────────────────────────────────┘
```

**Why KRaft?**

| Aspect | ZooKeeper | KRaft |
|---|---|---|
| Architecture | Separate system to manage | Single system |
| Partition limit | ~200K practical limit | Millions of partitions |
| Controller failover | ~30 seconds | ~5 seconds |
| Operational overhead | High (2 systems to manage) | Low (1 system) |
| Metadata propagation | Async via ZK watches | Raft log, event-driven |
| Security | Separate ACL system | Unified Kafka security |
| Status (2025+) | Deprecated, removed in 4.0 | Production default |

**KRaft internals**: The controller quorum uses the **Raft consensus protocol**. One controller is the active leader; others are voters/observers. Metadata is stored in an internal topic `__cluster_metadata` — replicated just like any other Kafka topic but with Raft semantics.

For ShopStream, KRaft means faster broker failover (5s vs 30s), support for more partitions across your high-cardinality topics, and one less system to operate. If you're on a managed service like Confluent Cloud or AWS MSK, this is handled for you.

### 💡 Interview Insight

> **Q: "Why is Kafka moving away from ZooKeeper?"**
> "ZooKeeper was an external dependency that created operational complexity, limited partition scalability to ~200K, and caused slow controller failovers (~30s). KRaft embeds metadata management directly into Kafka using Raft consensus. This eliminates an external system, enables millions of partitions, reduces failover to ~5 seconds, and unifies security under Kafka's own ACL system. ZooKeeper is fully removed in Kafka 4.0."

---

## Screen 8: Log Segments, Offsets & Retention

Under the hood, each partition is stored as a sequence of **log segment files** on disk. Understanding this physical layout is key to understanding retention and compaction.

```
Partition 0 directory: /kafka-data/order-events-0/

  00000000000000000000.log    ← Segment 1 (offsets 0-9999)
  00000000000000000000.index  ← Offset index for segment 1
  00000000000000000000.timeindex ← Time index for segment 1
  00000000000000010000.log    ← Segment 2 (offsets 10000-19999)
  00000000000000010000.index
  00000000000000010000.timeindex
  00000000000000020000.log    ← Active segment (currently being written)
  00000000000000020000.index
  00000000000000020000.timeindex
```

**Offsets** are sequential, per-partition, 64-bit integers. They are the consumer's "bookmark" — telling Kafka where to resume reading.

**Log retention** — two strategies that can coexist:

| Strategy | Config | Behavior |
|---|---|---|
| Time-based | `retention.ms=604800000` | Delete segments older than 7 days |
| Size-based | `retention.bytes=10737418240` | Delete oldest segments when total >10GB |
| Compaction | `cleanup.policy=compact` | Keep latest value per key forever |

**Log compaction** is special and powerful. Instead of deleting old segments by time, Kafka keeps the **latest record for each key** and deletes older duplicates:

```
BEFORE compaction:
  [k1:v1] [k2:v1] [k1:v2] [k3:v1] [k2:v2] [k1:v3]

AFTER compaction:
  [k3:v1] [k2:v2] [k1:v3]   ← latest value per key retained

  Tombstone: [k2:null] → eventually removes k2 entirely
```

**ShopStream use cases for each:**

- **`order-events`**: Time-based retention (90 days). You want the full event history for replay, but old orders can be purged.
- **`inventory-levels`**: Log compaction. Each key is `warehouse-sku` (e.g., `wh-42-SKU-1001`). You always want the latest inventory count, not the history. Compaction keeps only the latest per key.
- **`clickstream`**: Time-based (7 days). High volume, short-lived analytical value.

You can also combine both: `cleanup.policy=compact,delete` — compacts and enforces a time limit.

### 💡 Interview Insight

> **Q: "When would you use log compaction vs time-based retention?"**
> "Time-based retention suits event streams where you care about the sequence of events (order lifecycle, clickstream). Log compaction suits state/snapshot topics where you only care about the latest value per key — like inventory levels, user profiles, or configuration. Compacted topics act like a distributed key-value store on top of Kafka. A common pattern is to pair a compacted 'state' topic with a regular 'events' topic."

---

## Screen 9: Putting It All Together — ShopStream Architecture

Let's connect everything into ShopStream's full architecture:

```
┌─────────────┐  ┌──────────────┐  ┌───────────────┐
│ Checkout Svc │  │ Clickstream  │  │ Inventory Svc │
│ (Producer)   │  │ Collector    │  │ (Producer)    │
│ key=order_id │  │ key=user_id  │  │ key=wh-sku    │
│ acks=all     │  │ acks=1       │  │ acks=all      │
└──────┬───────┘  └──────┬───────┘  └───────┬───────┘
       │                 │                  │
       ▼                 ▼                  ▼
  ┌──────────┐    ┌────────────┐    ┌──────────────────┐
  │order-    │    │clickstream │    │inventory-updates │
  │events    │    │(50 parts)  │    │(compact, 20 parts│
  │(12 parts)│    │retain: 7d  │    │ retain: forever) │
  │retain:90d│    └─────┬──────┘    └────────┬─────────┘
  └────┬─────┘          │                    │
       │                │                    │
  ┌────┴────────────────┴────────────────────┴──────┐
  │              Kafka Cluster (KRaft)               │
  │         3 brokers, RF=3, min.isr=2               │
  └────┬────────────────┬────────────────────┬──────┘
       │                │                    │
       ▼                ▼                    ▼
  ┌──────────┐   ┌────────────┐    ┌────────────────┐
  │Order     │   │Analytics   │    │Warehouse       │
  │Processing│   │Pipeline    │    │Dashboard       │
  │(CG)      │   │(CG)        │    │(CG)            │
  └──────────┘   └────────────┘    └────────────────┘
```

**Key design decisions:**
- **order-events**: 12 partitions, `acks=all`, `key=order_id`, 90-day retention. Manual offset commit for at-least-once.
- **clickstream**: 50 partitions, `acks=1`, `key=user_id`, 7-day retention. High throughput, loss-tolerant.
- **inventory-updates**: 20 partitions, `acks=all`, `key=warehouse_id-sku_id`, **compacted**. Latest state always available.

---

## Quiz: Module 1

**Q1:** A topic has 6 partitions and your consumer group has 8 consumers. How many consumers are actively reading?

A) 8
B) 6
C) 3
D) 1

**Answer:** B) 6. Each partition maps to exactly one consumer. The remaining 2 consumers are idle standbys.

---

**Q2:** You produce records with `acks=all`, `replication.factor=3`, and `min.insync.replicas=2`. One broker goes down. What happens?

A) Writes fail because not all replicas are available
B) Writes succeed — 2 ISR members remain, meeting min.insync.replicas
C) Writes succeed but with acks=1 fallback
D) The topic becomes read-only

**Answer:** B) Writes succeed. `acks=all` means all *ISR* members, and with 2 brokers alive, ISR=2 meets min.insync.replicas=2.

---

**Q3:** Your order processing is slow — `max.poll.interval.ms` is exceeded. What happens?

A) The consumer slows down automatically
B) The consumer is evicted from the group, triggering a rebalance
C) Messages are skipped
D) The broker retries delivery

**Answer:** B) The consumer is evicted. Kafka's group coordinator treats the consumer as dead and triggers rebalancing.

---

**Q4:** You need all events for a specific user to be processed in order. Which approach works?

A) Use `user_id` as the message key
B) Use a single partition
C) Use `acks=all`
D) Set `max.in.flight.requests=1`

**Answer:** A) Using `user_id` as the key ensures all that user's events go to the same partition, which guarantees ordering.

---

**Q5:** What is the key difference between log compaction and time-based retention?

A) Compaction is faster
B) Compaction keeps the latest value per key; time-based deletes all records older than a threshold
C) Time-based is only for clickstream data
D) Compaction doesn't work with consumer groups

**Answer:** B) Compaction retains the latest record per key forever (until tombstoned), while time-based retention deletes all records past the time threshold regardless of key.

---

## Key Takeaways

1. **Topics** are logical, immutable, append-only logs — the fundamental abstraction.
2. **Partitions** are the unit of parallelism AND ordering. Choose partition keys wisely.
3. **Replication** (RF=3, `min.insync.replicas=2`, `acks=all`) is your durability guarantee.
4. **Producers** use key-based hashing for deterministic partition routing.
5. **Consumer groups** enable parallel processing; partition count caps parallelism.
6. **Rebalancing** is expensive — use cooperative sticky assignment and static membership.
7. **KRaft** replaces ZooKeeper: simpler operations, faster failover, more partitions.
8. **Log compaction** turns Kafka into a state store; time-based retention suits event streams.
9. **Offset management** is your contract with Kafka — commit after processing for at-least-once.
10. Always align partition count, replication, retention, and `acks` to your **business requirements** — not one-size-fits-all.
