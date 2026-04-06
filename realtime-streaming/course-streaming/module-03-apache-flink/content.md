# Module 3: Apache Flink — True Stream Processing

> **Scenario Throughout**: ShopStream needs real-time fraud detection on payment events, live revenue dashboards with session-based user analytics, and inventory alerts when stock dips below thresholds — all with exactly-once guarantees and handling late-arriving clickstream data.

---

## Screen 1: Flink Architecture — JobManager, TaskManagers & Parallelism

Apache Flink is a **true stream processing engine** — it processes events one at a time (not micro-batches). This gives it lower latency and more natural semantics for event-driven applications.

```
┌─────────────────────────────────────────────────────────────┐
│                     Flink Cluster                           │
│                                                             │
│  ┌───────────────────────┐                                  │
│  │    JobManager (JM)    │    ← The "brain"                 │
│  │  • Schedules tasks    │                                  │
│  │  • Manages checkpoints│                                  │
│  │  • Coordinates recovery│                                 │
│  │  • REST API (:8081)   │                                  │
│  └──────────┬────────────┘                                  │
│             │  distributes work                             │
│    ┌────────┼─────────┬────────────┐                        │
│    ▼        ▼         ▼            ▼                        │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐                       │
│  │ TM-1 │ │ TM-2 │ │ TM-3 │ │ TM-4 │  ← TaskManagers      │
│  │      │ │      │ │      │ │      │    (the "muscles")    │
│  │[slot]│ │[slot]│ │[slot]│ │[slot]│                       │
│  │[slot]│ │[slot]│ │[slot]│ │[slot]│  ← Task slots         │
│  └──────┘ └──────┘ └──────┘ └──────┘    (unit of resource) │
└─────────────────────────────────────────────────────────────┘
```

**Key components:**

| Component | Role | Analogy |
|---|---|---|
| JobManager | Schedules tasks, checkpoints, recovery | Orchestra conductor |
| TaskManager | Executes tasks, manages memory/network | Musicians |
| Task Slot | Fixed resource partition within a TM | A seat in the orchestra |
| Parallelism | Number of parallel instances of an operator | How many musicians play same part |

**Parallelism levels:**

```python
# Operator-level parallelism
stream.key_by(lambda x: x.user_id) \n      .window(TumblingEventTimeWindows.of(Time.minutes(5))) \n      .aggregate(CountAgg()) \n      .set_parallelism(8)    # This operator runs 8 parallel instances

# Job-level parallelism
env.set_parallelism(4)       # Default for all operators

# System-level
# flink-conf.yaml: parallelism.default: 4
```
\ow graph** — how Flink sees your job:

```
Source (Kafka)     →    Map/Filter    →    KeyBy + Window    →    Sink
  parallelism=4         parallelism=4       parallelism=8         parallelism=4

  ┌──┐  ┌──┐  ┌──┐  ┌──┐    ┌──┐  ┌──┐  ┌──┐  ┌──┐    ┌──┐  ┌──┐  ...
  │S1│→ │S2│→ │S3│→ │S4│    │M1│→ │M2│→ │M3│→ │M4│    │W1│→ │W2│→ ...
  └──┘  └──┘  └──┘  └──┘    └──┘  └──┘  └──┘  └──┘    └──┘  └──┘

  Source reads from Kafka partitions (1:1 mapping ideal)
  Map/Filter: stateless, embarrassingly parallel
  KeyBy: SHUFFLE — records redistributed by key hash
  Window: stateful aggregation per key per window
```

**Task Slots vs Parallelism**: A TaskManager has a fixed number of slots (configured at startup). Each slot gets a fraction of the TM's memory. Operators from the SAME job can share a slot (operator chaining), reducing serialization overhead. Total cluster parallelism = `num_task_managers × slots_per_task_manager`.

For ShopStream fraud detection: 4 TaskManagers × 4 slots = 16 max parallel tasks. Set source parallelism to match Kafka partition count for optimal throughput.

### 💡 Interview Insight

> **Q: "Explain Flink's architecture and how it differs from Spark Streaming."**
> "Flink has a JobManager (coordinator) and TaskManagers (workers). The JM schedules the dataflow graph, manages checkpoints, and handles recovery. TMs execute tasks in slots — fixed resource partitions. The key difference from Spark Streaming: Flink pvents **one at a time** with continuous operators, not micro-batches. This means lower latency (milliseconds vs seconds), true event-time processing, and more natural windowing semantics. Spark triggers processing at batch intervals; Flink processes as events arrive."

---

## Screen 2: DataStream API — Sources, Transformations & Sinks

The DataStream API is Flink's core programming model. It represents a pipeline of transformations on an unbounded stream of events.

```python
from pyflink.datastream import StreamExecutionEnvironment
from pyflink.datastream.connectors.kafka import KafkaSource, KafkaOffsetsInitializer
from pyflink.common.serialization import SimpleStringSchema

env = StreamExecutionEnvironment.get_execution_environment()
env.set_parallelism(4)

# SOURCE: Read from Kafka
kafka_source = KafkaSource.builder() \n    .set_bootstrap_servers("kafka:9092") \n    .set_topics("order-events") \n    .set_group_id("flink-order-processor") \n    .set_starting_offsets(KafkaOffsetsInitializer.earliest()) \n    .set_value_only_deserializer(SimpleStringSchema()) \n    .build()

order_stream = env.from_source(
    kafka_source,
    WatermarkStrategy.for_bounded_out_of_orderness(Duration.of_seconds(5)),
    "Kafka Order Source"
)

# TRANSFORMATIONS
parsed = order_stream \n    .map(parse_order_json) \           # Parse JSON → OrderEvent
    .filter(lambda e: e.amount > 0)    # Remove invalid orders

# KEY BY + WINDOW
revenue_per_minute = parsed \n    .key_by(lambda e: e.category) \n    .window(TumblingEventTimeWindows.of(Time.minutes(1))) \n    .reduce(lambda a, b: a.amount + b.amount)

# SINK: Write results
revenue_per_minute.print()  # or .add_sink(kafka_sink)

env.execute("ShopStream Revenue Pipeline")
```

**Core transformations:**

| Transformation | Type | Description |
|---|---|---|
| `map()` | 1:1 | Transform each element |
| `flat_map()` | 1:N | Transform, possibly emit multiple |
| `filter()` | 1:0/1 | Keep or discard elements |
| `key_by()` | Partition | Partition stream by key |
| `window()` | Group | Group elements by time/count |
| `reduce()` | Aggregate | Combine elements in window |
| `process()` | Flexible | Full access to context, state, timers |
| `union()` | Merge | Merge multiple streams (same type) |
| `connect()` | Join | Connect two streams (different types) |
| `side_output()` | Split | Route elements to side outputs |

**Operator chaining** — Flink automatically chains compatible operators into a single task to avoid serialization overhead:

```
Before chaining:
  Source → [serialize] → Map → [serialize] → Filter → [serialize] → KeyBy

After chaining:
  [Source → Map → Filter] → [serialize] → KeyBy
   ─── single task ───       network shuffle

  Chaining is broken at: keyBy, rebalance, shuffle, rescale
```

**ShopStream example — fraud detection pipeline:**

```
order-events → Parse → Enrich (user profile) → Key by user_id
                                                     │
                                    ┌────────────────┼────────────────┐
                                    ▼                ▼                ▼
                              Rule Engine      ML Scorer       Velocity Check
                              (> $5000?)       (fraud prob)    (5+ orders/min?)
                                    │                │                │
                                    └────────────────┼────────────────┘
                                                     ▼
                                               Merge + Score
                                                     │
                                           ┌─────────┴─────────┐
                                           ▼                   ▼
                                     fraud-alerts         order-approved
                                     (Kafka topic)        (Kafka topic)
```

### 💡 Interview Insight

> **Q: "How would you build a real-time fraud detection system with Flink?"**
> "I'd consume from the `order-events` Kafka topic, parse and enrich events with user profile data (via async I/O or a broadcast state pattern). Then key by `user_id` to partition the stream. Apply multiple fraud checks in parallel: rule-based (amount thresholds), ML scoring (pre-trained model), and velocity checks (count orders in sliding window). Merge scores and route high-risk orders to a `fraud-alerts` topic via side outputs, while approved orders continue to `order-approved`. Flink's keyed state tracks per-user history, and checkpointing ensures exactly-once processing."

---

## Screen 3: Event Time vs Processing Time vs Ingestion Time

Time semantics are **the most critical concept** in stream processing. Getting this wrong means your results are wrong.

```
                     Event happens        Enters Kafka       Flink processes
                     at source            (broker time)      (wall clock)
                         │                     │                  │
Timeline: ───────────────┼─────────────────────┼──────────────────┼──────
                         │                     │                  │
                    EVENT TIME           INGESTION TIME     PROCESSING TIME
                    (embedded in         (Kafka broker      (Flink operator
                     the event)           timestamp)         wall clock)

Example — Mobile order placed at 2:00 PM, phone was offline:
  Event Time:      2:00 PM  ← when the user clicked "Buy"
  Ingestion Time:  2:05 PM  ← when phone reconnected, event hit Kafka
  Processing Time: 2:05.3 PM ← when Flink processed the event
```

**Why event time matters for ShopStream:**

```
Scenario: "Revenue per minute" dashboard

Processing time approach:
  2:00-2:01 window: 100 orders
  2:01-2:02 window: 200 orders  ← includes delayed mobile orders from 2:00!
  Result: WRONG — attributes revenue to the wrong minute

Event time approach:
  2:00-2:01 window: 150 orders  ← includes late mobile orders, correctly placed
  2:01-2:02 window: 150 orders
  Result: CORRECT — revenue attributed to when orders actually happened
```

**When to use each:**

| Time Semantic | Use When | ShopStream Example |
|---|---|---|
| Event Time | Results must reflect when events happened | Revenue reports, fraud time windows |
| Processing Time | Low latency matters more than accuracy | Real-time monitoring dashboards |
| Ingestion Time | No event timestamp available | Legacy systems without timestamps |

**Assigning timestamps and watermarks:**

```python
from pyflink.common import WatermarkStrategy, Duration

# Strategy 1: Bounded out-of-orderness (most common)
# "Events may arrive up to 5 seconds late"
strategy = WatermarkStrategy \n    .for_bounded_out_of_orderness(Duration.of_seconds(5)) \n    .with_timestamp_assigner(
        lambda event, _: event.timestamp_ms  # Extract event time
    )

# Strategy 2: Monotonous timestamps (events perfectly ordered)
strategy = WatermarkStrategy.for_monotonous_timestamps()

# Apply to source
order_stream = env.from_source(kafka_source, strategy, "Orders")
```

### 💡 Interview Insight

> **Q: "When would you use event time vs processing time?"**
> "Event time whenever correctness matters — revenue calculations, fraud detection windows, SLA measurements. Event time answers 'when did this actually happen?' Processing time when you need lowest latency and approximate results are acceptable — like monitoring dashboards or alerting. The cost of event time is complexity (watermarks, late data handling) and slightly higher latency (waiting for watermarks to advance). In practice, 90% of production streaming jobs should use event time."

---

## Screen 4: Watermarks & Late Data Handling

**Watermarks** are Flink's mechanism to track progress in event time. A watermark `W(t)` declares: "No events with timestamp ≤ t will arrive after this point."

```
Event stream (time on x-axis, arrival order top-to-bottom):

  Arriving events:  e(2:01)  e(2:03)  e(2:02)  e(2:04)  e(2:03)  e(2:06)
                      │        │        │        │        │        │
  Watermarks:       W(1:56)  W(1:58)  W(1:58)  W(1:59)  W(1:59)  W(2:01)
                    ────────────────────────────────────────────────────→

  With boundedOutOfOrderness = 5 seconds:
  Watermark = max_event_time_seen - 5 seconds

  After seeing e(2:06): watermark = 2:06 - 5s = 2:01
  → Window [2:00-2:01) can now FIRE (watermark passed 2:01)
  → Any event with timestamp < 2:01 arriving after this is "late"
```

**Watermark generation strategies:**

```
┌─────────────────────────────────────────────────────────┐
│ forBoundedOutOfOrderness(Duration.ofSeconds(5))         │
│                                                         │
│ Best for: Most real-world streams with some disorder    │
│ How: watermark = maxTimestamp - maxOutOfOrderness        │
│ Trade-off: Higher bound → more late events accepted     │
│            but higher latency for window results        │
├─────────────────────────────────────────────────────────┤
│ forMonotonousTimestamps()                               │
│                                                         │
│ Best for: Perfectly ordered sources (rare)              │
│ How: watermark = lastEventTimestamp                     │
│ Trade-off: Zero tolerance for out-of-order = data loss  │
├─────────────────────────────────────────────────────────┤
│ Custom WatermarkStrategy                                │
│                                                         │
│ Best for: Sources with known delay patterns             │
│ How: You implement the logic                            │
│ Example: Different delays per data center               │
└─────────────────────────────────────────────────────────┘
```

**Handling late data:**

```python
from pyflink.datastream.window import TumblingEventTimeWindows
from pyflink.common import Time, Duration

result = order_stream \n    .key_by(lambda e: e.category) \n    .window(TumblingEventTimeWindows.of(Time.minutes(1))) \n    .allowed_lateness(Time.minutes(5)) \           # Accept late data up to 5 min
    .side_output_late_data(late_output_tag) \       # Route very late data to side output
    .aggregate(RevenueAggregator())

# Late data flow:
#   Event arrives within window → normal processing
#   Event arrives after watermark but within allowed lateness → window RE-FIRES
#   Event arrives after allowed lateness → routed to side output (late_output_tag)
```

```
Timeline for window [2:00 - 2:01):

  ──────────┬──────────┬──────────────────┬─────────────────→
            │          │                  │
        Window fires   │           Allowed lateness
        (watermark     │           expires (2:06)
         passes 2:01)  │
                       │
                  Late event arrives (2:03)
                  → window RE-FIRES with updated result

  After 2:06: events for [2:00-2:01) → side output (too late)
```

**ShopStream watermark design:**

- **Order events**: `boundedOutOfOrderness = 10 seconds`. Orders from mobile apps may have slight delays, but 10s covers most cases.
- **Clickstream**: `boundedOutOfOrderness = 30 seconds`. Browser events from slow connections can be significantly delayed.
- **Inventory updates**: `forMonotonousTimestamps()`. Database CDC events arrive in WAL order — perfectly monotonic.

### 💡 Interview Insight

> **Q: "What happens when data arrives after the watermark has passed?"**
> "It depends on configuration. By default, late data is dropped — the window has already fired and its state is purged. With `allowedLateness`, the window state is kept longer and the window re-fires with updated results when late data arrives. For data arriving after even the allowed lateness, I use side outputs to capture it for separate processing — maybe writing to a 'late-events' topic for batch reconciliation. The key trade-off: longer watermark delays and allowed lateness increase accuracy but also increase state size and result latency."

---

## Screen 5: Window Types — Tumbling, Sliding, Session & Global

Windows group elements for bounded computations over an unbounded stream. Choosing the right window type is a critical design decision.

```
TUMBLING WINDOWS (fixed size, no overlap)
  ┌────────┐┌────────┐┌────────┐┌────────┐
  │ 0-5min ││ 5-10m  ││10-15m  ││15-20m  │
  └────────┘└────────┘└────────┘└────────┘
  Use: Revenue per minute, hourly order counts
  Every event belongs to exactly ONE window

SLIDING WINDOWS (fixed size, with overlap)
  ┌──────────────┐
  │  0-10 min    │
  └──┬───────────┘
     ┌──────────────┐
     │  5-15 min    │  ← overlapping!
     └──┬───────────┘
        ┌──────────────┐
        │ 10-20 min    │
        └──────────────┘
  Size=10min, Slide=5min → each event in 2 windows
  Use: "Orders in last 10 minutes, updated every 5 min"

SESSION WINDOWS (dynamic, gap-based)
  ┌───────────────┐    ┌────────┐    ┌──────────────────────┐
  │ User A session│    │ User A │    │ User A session       │
  │ click,click,  │    │ (gap   │    │ click,add-cart,buy   │
  │ click         │    │ >30min)│    │                      │
  └───────────────┘    └────────┘    └──────────────────────┘
  Gap=30min → if no event for 30min, session closes
  Use: User session analytics, shopping session duration

GLOBAL WINDOWS (you control trigger)
  ┌──────────────────────────────────────────────────────┐
  │ Everything in one window, custom trigger decides     │
  │ when to compute                                      │
  └──────────────────────────────────────────────────────┘
  Use: Custom aggregation logic (e.g., "every 1000 events")
```

**ShopStream window use cases:**

```python
# Tumbling: Revenue per minute for dashboard
revenue = orders \n    .key_by(lambda o: o.category) \n    .window(TumblingEventTimeWindows.of(Time.minutes(1))) \n    .reduce(lambda a, b: a + b)

# Sliding: Average order value over last hour, updated every 5 min
avg_value = orders \n    .key_by(lambda o: o.region) \n    .window(SlidingEventTimeWindows.of(Time.hours(1), Time.minutes(5))) \n    .aggregate(AverageAggregator())

# Session: Shopping session duration per user
sessions = clickstream \n    .key_by(lambda c: c.user_id) \n    .window(EventTimeSessionWindows.with_gap(Time.minutes(30))) \n    .process(SessionAnalyzer())

# Count-based (global window + count trigger)
batches = orders \n    .key_by(lambda o: o.warehouse_id) \n    .window(GlobalWindows.create()) \n    .trigger(CountTrigger.of(100)) \n    .process(BatchProcessor())  # Process every 100 orders per warehouse
```

**Window internals — triggers and evictors:**

| Component | Role | Default |
|---|---|---|
| WindowAssigner | Assigns events to windows | By window type |
| Trigger | Decides when window fires | Watermark (event time) |
| Evictor | Optionally removes elements before/after | None |
| AllowedLateness | How long to keep window state after fire | 0 (drop late) |

**Memory impact:**

```
Tumbling (1 min): State = 1 window per key
Sliding (1hr / 5min): State = 12 overlapping windows per key ← 12x more!
Session (30min gap): State = variable, depends on user activity
Global: State grows unbounded without proper trigger/evictor ← dangerous!
```

### 💡 Interview Insight

> **Q: "You need to track user shopping sessions that end after 30 minutes of inactivity. Which window type and why?"**
> "Session windows with a 30-minute gap. Unlike tumbling or sliding windows which have fixed boundaries, session windows are dynamic — they open when a user's first event arrives and close when no event arrives for 30 minutes. The window boundary is per-key (per user), so different users have different session lengths. I'd key by `user_id` and use event-time session windows. One consideration: session windows can merge — if a late event arrives that bridges the gap between two previously separate sessions, Flink automatically merges them into one session."

---

## Screen 6: State Management — The Heart of Stateful Streaming

State is what makes Flink powerful for complex event processing. Unlike stateless map/filter operations, stateful operators remember information across events.

**State types:**

```
KEYED STATE (most common — partitioned by key)
┌──────────────────────────────────────────────────────┐
│ key_by(user_id) → each key has independent state     │
│                                                      │
│ User "alice": ValueState(last_order_time=2:05)       │
│ User "bob":   ValueState(last_order_time=1:58)       │
│ User "carol": ValueState(last_order_time=2:12)       │
│                                                      │
│ State types:                                         │
│  • ValueState<T>   → single value per key            │
│  • ListState<T>    → list of values per key          │
│  • MapState<K,V>   → map of key-values per key       │
│  • ReducingState   → aggregated value per key        │
│  • AggregatingState → pre-aggregated per key         │
└──────────────────────────────────────────────────────┘

OPERATOR STATE (rare — partitioned by operator instance)
┌──────────────────────────────────────────────────────┐
│ Not partitioned by key — each parallel instance has  │
│ its own state. Used in sources/sinks.                │
│                                                      │
│ Example: Kafka source stores partition offsets        │
│  Operator instance 0: {P0: offset 45023}             │
│  Operator instance 1: {P1: offset 38291}             │
└──────────────────────────────────────────────────────┘
```

**State backends:**

```
┌──────────────────────────┬──────────────────────────────┐
│   HashMapStateBackend    │  EmbeddedRocksDBStateBackend │
├──────────────────────────┼──────────────────────────────┤
│ In JVM heap memory       │ On local disk (SSD)          │
│ Very fast (ns access)    │ Slower (μs access, serde)    │
│ Limited by heap size     │ Limited by disk size         │
│ GC pressure with large   │ No GC pressure               │
│   state                  │                              │
│ Checkpoints: full snap   │ Checkpoints: incremental!    │
│ Best: state < 10 GB      │ Best: state > 10 GB          │
│ Use: small state, low    │ Use: large state, production │
│   latency critical       │   workloads                  │
└──────────────────────────┴──────────────────────────────┘
```

**ShopStream fraud detection with keyed state:**

```python
class FraudDetector(KeyedProcessFunction):
    def open(self, runtime_context):
        # State: count of orders in last 5 minutes per user
        self.order_count = runtime_context.get_state(
            ValueStateDescriptor("order-count", Types.INT())
        )
        self.last_reset_time = runtime_context.get_state(
            ValueStateDescriptor("last-reset", Types.LONG())
        )

    def process_element(self, event, ctx):
        current_count = self.order_count.value() or 0
        current_count += 1
        self.order_count.update(current_count)

        # Register timer to reset count after 5 minutes
        ctx.timer_service().register_event_time_timer(
            event.timestamp + 5 * 60 * 1000
        )

        # Fraud rule: more than 10 orders in 5 minutes
        if current_count > 10:
            yield FraudAlert(user_id=ctx.get_current_key(),
                           count=current_count,
                           severity="HIGH")

    def on_timer(self, timestamp, ctx):
        self.order_count.clear()  # Reset counter
```

**State TTL — preventing unbounded state growth:**

```python
from pyflink.datastream.state import StateTtlConfig, Time

ttl_config = StateTtlConfig \n    .new_builder(Time.hours(24)) \n    .set_update_type(StateTtlConfig.UpdateType.OnCreateAndWrite) \n    .set_state_visibility(StateTtlConfig.StateVisibility.NeverReturnExpired) \n    .build()

state_descriptor = ValueStateDescriptor("user-profile", Types.STRING())
state_descriptor.enable_time_to_live(ttl_config)
# State auto-cleaned after 24 hours of no updates
```

### 💡 Interview Insight

> **Q: "How does Flink handle state that grows unbounded?"**
> "Three strategies: (1) **State TTL** — configure automatic expiration for state entries. For user session data, I'd set TTL to 24 hours so inactive user state is cleaned up. (2) **RocksDB state backend** — stores state on disk instead of heap, supporting terabytes of state without GC pressure. (3) **Application-level cleanup** — use timers to explicitly clear state after a business-defined period. For ShopStream fraud detection, I register a timer to clear the order count after the 5-minute detection window. Without these strategies, state grows indefinitely and eventually crashes the TaskManager."

---

## Screen 7: Checkpointing & Fault Tolerance

Checkpointing is how Flink achieves **exactly-once processing** even when failures occur. It's based on the **Chandy-Lamport distributed snapshot algorithm**.

```
HOW CHECKPOINTING WORKS:

1. JobManager injects checkpoint BARRIERS into all source streams
2. Barriers flow through the dataflow graph with the data
3. When an operator receives barriers from ALL inputs, it snapshots its state
4. State snapshots are stored to durable storage (S3, HDFS, etc.)

  Source 1: ──[e1]──[e2]──║BARRIER║──[e3]──[e4]──→
                            │
  Source 2: ──[e5]──[e6]──║BARRIER║──[e7]──→
                            │
                    ┌───────┴───────┐
                    │   Operator    │
                    │ (waits for    │
                    │  ALL barriers)│
                    │ → snapshot    │
                    │   state       │
                    └───────┬───────┘
                            │
                   ──║BARRIER║──→ downstream
```

**Barrier alignment** (exactly-once):

```
Operator with 2 inputs:

  Input 1: ──[e1]──║B║──[e2]──[e3]──→
  Input 2: ──[e4]──[e5]──[e6]──║B║──→

  Step 1: Barrier arrives on Input 1 → BLOCK Input 1, buffer e2, e3
  Step 2: Continue processing Input 2 (e4, e5, e6)
  Step 3: Barrier arrives on Input 2 → snapshot state → release barrier
  Step 4: Unblock Input 1, process buffered e2, e3

  Result: Snapshot reflects exactly events BEFORE barrier on ALL inputs
          → exactly-once semantics!

WITHOUT alignment (at-least-once, faster):
  Process all events as they arrive, don't buffer
  Snapshot may include some events from after barrier on one input
  → at-least-once (may reprocess some events on recovery)
```

**Checkpoint configuration:**

```python
env.enable_checkpointing(60000)  # Checkpoint every 60 seconds

config = env.get_checkpoint_config()
config.set_checkpointing_mode(CheckpointingMode.EXACTLY_ONCE)
config.set_min_pause_between_checkpoints(30000)  # 30s min gap
config.set_checkpoint_timeout(600000)             # 10 min timeout
config.set_max_concurrent_checkpoints(1)          # One at a time
config.set_tolerable_checkpoint_failure_number(3) # Allow 3 failures

# Incremental checkpoints (RocksDB only — huge for large state)
config.enable_incremental_checkpoints(True)

# Checkpoint storage
config.set_checkpoint_storage("s3://shopstream-flink/checkpoints/")
```

**Checkpoints vs Savepoints:**

| Feature | Checkpoint | Savepoint |
|---|---|---|
| Triggered by | Flink (automatic, periodic) | User (manual) |
| Purpose | Fault recovery | Planned operations |
| Use case | Crash recovery | Job upgrade, rescale, migration |
| Format | Implementation-specific | Portable, stable format |
| Cleanup | Auto-cleaned on job cancel | Persisted until manual delete |
| Incremental | Yes (RocksDB) | No (always full) |

**Recovery from failure:**

```
Normal operation: CP1 ─────── CP2 ─────── CP3 ──── CRASH!
                                           ↑
Recovery:                          Restore from CP3
                                   • Kafka offsets → reset to CP3 position
                                   • State → restored from CP3 snapshot
                                   • Reprocess events from CP3 offset
                                   → exactly-once: events between CP3 and crash
                                     are reprocessed but NOT duplicated in output
                                     (via transaction commit)
```

### 💡 Interview Insight

> **Q: "How does Flink achieve exactly-once semantics end-to-end with Kafka?"**
> "Three mechanisms work together: (1) **Checkpointing** snapshots operator state and Kafka consumer offsets atomically using the Chandy-Lamport algorithm with barrier alignment. (2) **Kafka consumer** offsets are committed as part of the checkpoint — on recovery, consumers rewind to the checkpointed offset and reprocess. (3) **Kafka producer** uses transactions — output records are committed only when the checkpoint completes via two-phase commit. If the job crashes, the incomplete transaction is aborted, so downstream `read_committed` consumers never see duplicate output. The end result: each input event affects the output exactly once."

---

## Screen 8: Flink SQL, Table API & CDC

Flink SQL brings the power of SQL to stream processing. **Dynamic tables** are the key abstraction — a SQL table that continuously updates as new events arrive.

```
Traditional SQL:
  SELECT category, SUM(amount)
  FROM orders
  GROUP BY category;
  → Runs once, returns static result

Flink SQL:
  SELECT category, SUM(amount)
  FROM orders
  GROUP BY category;
  → Continuously updates as new orders arrive!
  → Result is a "dynamic table" (changelog stream)
```

**Flink SQL for ShopStream:**

```sql
-- Create source table (Kafka topic)
CREATE TABLE order_events (
    order_id     STRING,
    event_type   STRING,
    amount       DECIMAL(10,2),
    currency     STRING,
    customer_id  STRING,
    category     STRING,
    event_time   TIMESTAMP(3),
    WATERMARK FOR event_time AS event_time - INTERVAL '10' SECOND
) WITH (
    'connector' = 'kafka',
    'topic' = 'order-events',
    'properties.bootstrap.servers' = 'kafka:9092',
    'properties.group.id' = 'flink-sql-revenue',
    'format' = 'json',
    'scan.startup.mode' = 'earliest-offset'
);

-- Real-time revenue per category per minute
SELECT
    category,
    TUMBLE_START(event_time, INTERVAL '1' MINUTE) AS window_start,
    COUNT(*) AS order_count,
    SUM(amount) AS total_revenue,
    AVG(amount) AS avg_order_value
FROM order_events
WHERE event_type = 'CREATED'
GROUP BY
    category,
    TUMBLE(event_time, INTERVAL '1' MINUTE);

-- Session-based user analytics
SELECT
    customer_id,
    SESSION_START(event_time, INTERVAL '30' MINUTE) AS session_start,
    SESSION_END(event_time, INTERVAL '30' MINUTE) AS session_end,
    COUNT(*) AS events_in_session
FROM clickstream
GROUP BY
    customer_id,
    SESSION(event_time, INTERVAL '30' MINUTE);
```

**Temporal Joins** — join with time-versioned data:

```sql
-- Join orders with the product price that was valid AT THE TIME of the order
SELECT o.order_id, o.product_id, p.price, o.quantity,
       o.quantity * p.price AS total
FROM orders AS o
JOIN product_prices FOR SYSTEM_TIME AS OF o.event_time AS p
ON o.product_id = p.product_id;
```

**Flink CDC Connectors** — stream database changes directly into Flink SQL:

```sql
-- Read directly from PostgreSQL via CDC (no Kafka needed!)
CREATE TABLE inventory (
    sku          STRING,
    warehouse_id STRING,
    quantity     INT,
    updated_at   TIMESTAMP(3),
    PRIMARY KEY (sku, warehouse_id) NOT ENFORCED
) WITH (
    'connector' = 'postgres-cdc',
    'hostname' = 'postgres',
    'port' = '5432',
    'database-name' = 'shopstream',
    'schema-name' = 'public',
    'table-name' = 'inventory',
    'slot.name' = 'flink_inventory_slot'
);

-- Real-time low-stock alerts
SELECT sku, warehouse_id, quantity
FROM inventory
WHERE quantity < 10;   -- Continuously emits when stock drops below 10
```

### 💡 Interview Insight

> **Q: "When would you use Flink SQL vs the DataStream API?"**
> "Flink SQL for standard analytics queries — aggregations, joins, windowed computations — where the logic maps naturally to SQL. It's faster to develop, easier to maintain, and accessible to analysts. DataStream API for complex event processing — custom state management, multiple side outputs, async I/O, complex windowing triggers, or ML model inference. In practice, I start with SQL and drop to DataStream only when SQL can't express the logic. Flink also allows mixing both — SQL for the main pipeline, DataStream for custom operators."

---

## Screen 9: Flink vs Spark Structured Streaming

This comparison comes up in virtually every streaming interview:

```
┌─────────────────────┬──────────────────────┬──────────────────────┐
│ Aspect              │ Apache Flink         │ Spark Structured     │
│                     │                      │ Streaming            │
├─────────────────────┼──────────────────────┼──────────────────────┤
│ Processing Model    │ True per-event       │ Micro-batch          │
│                     │ streaming            │ (continuous mode     │
│                     │                      │  experimental)       │
├─────────────────────┼──────────────────────┼──────────────────────┤
│ Latency             │ Milliseconds         │ Seconds (100ms+      │
│                     │                      │  micro-batch)        │
├─────────────────────┼──────────────────────┼──────────────────────┤
│ State Management    │ First-class: keyed   │ mapGroupsWithState,  │
│                     │ state, timers, TTL,  │ limited state types  │
│                     │ RocksDB, queryable   │                      │
├─────────────────────┼──────────────────────┼──────────────────────┤
│ Windowing           │ Tumbling, sliding,   │ Tumbling, sliding    │
│                     │ session, global,     │ (no native session   │
│                     │ custom triggers      │  windows)            │
├─────────────────────┼──────────────────────┼──────────────────────┤
│ Event Time          │ Native, first-class  │ Supported but less   │
│                     │ watermarks           │ flexible watermarks  │
├─────────────────────┼──────────────────────┼──────────────────────┤
│ Exactly-Once        │ Chandy-Lamport +     │ Micro-batch +        │
│                     │ 2PC sinks            │ idempotent sinks     │
├─────────────────────┼──────────────────────┼──────────────────────┤
│ Batch + Stream      │ Unified (DataStream  │ Unified (DataFrame   │
│                     │ API for both)        │ API for both) —      │
│                     │                      │ more mature batch    │
├─────────────────────┼──────────────────────┼──────────────────────┤
│ SQL Support         │ Flink SQL (dynamic   │ Spark SQL (mature,   │
│                     │ tables)              │ large ecosystem)     │
├─────────────────────┼──────────────────────┼──────────────────────┤
│ Ecosystem           │ Growing, strong in   │ Huge — MLlib,        │
│                     │ streaming niche      │ GraphX, SparkR       │
├─────────────────────┼──────────────────────┼──────────────────────┤
│ Community/Adoption  │ Alibaba, Uber, Netflix│ Most enterprises    │
│                     │                      │                      │
├─────────────────────┼──────────────────────┼──────────────────────┤
│ CEP (Complex Event  │ Built-in CEP library │ No native CEP        │
│ Processing)         │                      │                      │
├─────────────────────┼──────────────────────┼──────────────────────┤
│ Best For            │ Low-latency,         │ Batch-first with     │
│                     │ stateful streaming,  │ streaming addition,  │
│                     │ event-driven apps    │ existing Spark teams │
└─────────────────────┴──────────────────────┴──────────────────────┘
```

**Decision framework:**

```
Choose FLINK when:
  ✓ Sub-second latency required
  ✓ Complex stateful processing (fraud, CEP, session analysis)
  ✓ Session windows needed
  ✓ Fine-grained event-time control
  ✓ Large state (100s of GB) with RocksDB

Choose SPARK STRUCTURED STREAMING when:
  ✓ Team already uses Spark for batch
  ✓ Seconds-level latency is acceptable
  ✓ Need batch + streaming unification
  ✓ Heavy ML integration (MLlib)
  ✓ Broader ecosystem (Delta Lake, Iceberg)
```

### 💡 Interview Insight

> **Q: "Compare Flink and Spark Structured Streaming for a real-time fraud detection system."**
> "I'd choose Flink. Three reasons: (1) **Latency** — fraud detection needs millisecond response time; Flink processes per-event while Spark's micro-batch adds 100ms+ delay. (2) **State management** — Flink has first-class keyed state with RocksDB backend, TTL, and timers, ideal for tracking per-user fraud patterns. Spark's `mapGroupsWithState` is more limited. (3) **Session windows** — analyzing user session behavior is natural in Flink; Spark doesn't have native session windows. I'd choose Spark Streaming if the team already had Spark expertise and latency requirements were relaxed to seconds."

---

## Quiz: Module 3

**Q1:** Your Flink job consumes from Kafka and produces to Kafka with exactly-once. Checkpoints are every 60 seconds. The job crashes at second 45 of a checkpoint interval. What happens?

A) 45 seconds of data is lost permanently
B) Flink restores from the last checkpoint, reprocesses ~45 seconds, transactions prevent duplicate output
C) Kafka replays all data from the beginning
D) The job fails permanently

**Answer:** B) Flink restores state and Kafka offsets from the last successful checkpoint. It reprocesses ~45 seconds of events, but the Kafka producer's uncommitted transaction is aborted — downstream `read_committed` consumers never see duplicates.

---

**Q2:** You set `boundedOutOfOrderness = 10 seconds`. An event with timestamp 2:00:05 arrives when the watermark is at 2:00:20. What happens?

A) Processed normally
B) Dropped (late data, past watermark)
C) Processed if within `allowedLateness`
D) Either B or C depending on configuration

**Answer:** D) If no `allowedLateness` is set, the event is dropped (B). If `allowedLateness` covers it, the window re-fires with the late event (C).

---

**Q3:** You need to track unique visitors per product page over 1-hour windows, updated every 5 minutes. Which window type?

A) Tumbling window of 1 hour
B) Sliding window of 1 hour with 5-minute slide
C) Session window with 5-minute gap
D) Global window with time trigger

**Answer:** B) Sliding window gives you a 1-hour lookback updated every 5 minutes — exactly the "rolling window" pattern needed.

---

**Q4:** Your Flink job has 100GB of keyed state. Which state backend should you use?

A) HashMapStateBackend
B) EmbeddedRocksDBStateBackend
C) Either works equally well
D) Neither — Flink can't handle 100GB state

**Answer:** B) RocksDB stores state on disk, supports incremental checkpoints, and handles terabytes without GC pressure. HashMap would cause severe GC issues with 100GB on heap.

---

**Q5:** What is the key difference between a Flink checkpoint and a savepoint?

A) Checkpoints are faster
B) Checkpoints are automatic for fault tolerance; savepoints are manual for planned operations like upgrades
C) Savepoints use less storage
D) Checkpoints work with RocksDB; savepoints don't

**Answer:** B) Checkpoints are periodic and automatic — Flink creates and cleans them. Savepoints are user-triggered, portable, and persist until manually deleted — used for job upgrades, rescaling, or migration.

---

## Key Takeaways

1. **Flink is true streaming** — per-event processing with millisecond latency, not micro-batches.
2. **JobManager** coordinates; **TaskManagers** execute. Parallelism = slots × TaskManagers.
3. **Event time** is almost always correct for business logic; processing time only for monitoring.
4. **Watermarks** track event-time progress; `boundedOutOfOrderness` handles real-world disorder.
5. **Window choice** depends on use case: tumbling for fixed reports, sliding for rolling metrics, session for user behavior.
6. **Keyed state** enables complex per-entity processing; use **RocksDB** for production-scale state.
7. **State TTL** prevents unbounded growth — always configure cleanup for production jobs.
8. **Checkpointing** (Chandy-Lamport) + Kafka transactions = end-to-end exactly-once.
9. **Flink SQL** is the fast path for standard analytics; DataStream API for complex stateful logic.
10. **Flink > Spark Streaming** for low-latency, stateful, event-driven workloads. Spark wins for batch-heavy teams.