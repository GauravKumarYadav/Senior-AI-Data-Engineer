# Module 5: NoSQL Essentials for Data Engineers

> **Goal**: Understand the major NoSQL paradigms — key-value, document, graph, and in-memory stores — well enough to design schemas, write queries, and make informed SQL-vs-NoSQL decisions in system design interviews. You don't need to be a NoSQL expert, but you need to know *when* and *why* to reach for each tool.

---

## Screen 1: DynamoDB — Key-Value at Planet Scale

### The Mental Model

DynamoDB is a fully managed key-value + document database from AWS. Think of it as a **distributed hash map** where every item is identified by a primary key and stored across partitions based on that key.

```
┌──────────────────────────────────────────────────────────┐
│  DynamoDB Table: orders                                   │
│                                                           │
│  Partition Key (PK) ──→ determines which physical        │
│                          partition stores the item        │
│  Sort Key (SK) ────────→ orders items WITHIN a partition  │
│                                                           │
│  Partition A              Partition B            Partition C
│  ┌───────────────┐        ┌───────────────┐      ┌────────┐
│  │ PK: CUST#001  │        │ PK: CUST#002  │      │ PK: .. │
│  │ SK: ORD#2024..│        │ SK: ORD#2024..│      │        │
│  │ SK: ORD#2024..│        │ SK: ORD#2024..│      │        │
│  └───────────────┘        └───────────────┘      └────────┘
└──────────────────────────────────────────────────────────┘
```

### Key Design: Partition Key + Sort Key

The **partition key** (PK) determines data distribution. The **sort key** (SK) enables range queries within a partition.

```
Table: orders
PK: customer_id
SK: order_date#order_id    (composite string for uniqueness + sorting)

┌─────────────┬────────────────────┬─────────┬──────────┐
│ customer_id │ order_date#oid     │ amount  │ status   │
│ (PK)        │ (SK)               │         │          │
├─────────────┼────────────────────┼─────────┼──────────┤
│ CUST#001    │ 2024-01-15#ORD-101 │ 59.99   │ shipped  │
│ CUST#001    │ 2024-02-20#ORD-205 │ 124.50  │ pending  │
│ CUST#001    │ 2024-03-08#ORD-312 │ 89.00   │ shipped  │
│ CUST#002    │ 2024-01-03#ORD-055 │ 250.00  │ delivered│
│ CUST#002    │ 2024-04-11#ORD-401 │ 34.99   │ pending  │
└─────────────┴────────────────────┴─────────┴──────────┘
```

### Query Capabilities

```python
# Get all orders for a customer (efficient — queries ONE partition)
table.query(
    KeyConditionExpression=Key('customer_id').eq('CUST#001')
)

# Get orders for a customer in a date range (sort key range)
table.query(
    KeyConditionExpression=(
        Key('customer_id').eq('CUST#001') &
        Key('order_date_oid').between('2024-01-01', '2024-03-31')
    )
)

# Get a single item (fastest — direct key lookup)
table.get_item(Key={'customer_id': 'CUST#001', 'order_date_oid': '2024-01-15#ORD-101'})

# ❌ This is a SCAN (reads entire table — expensive!):
# "Find all orders with status = 'pending'" without an index
```

### Global Secondary Indexes (GSIs)

A GSI is a **full copy of the table** with a different key schema, maintained automatically by DynamoDB.

```
Base Table:           PK=customer_id, SK=order_date#order_id
GSI-1 (by status):   PK=status,      SK=order_date
GSI-2 (by product):  PK=product_id,  SK=order_date

Original access pattern:
  "All orders for customer X" → Query base table ✅

New access patterns via GSIs:
  "All pending orders" → Query GSI-1 where PK='pending' ✅
  "Orders for product Y in last month" → Query GSI-2 ✅
```

```
⚠️ GSI Gotchas:
  - Each GSI doubles your write cost (DynamoDB replicates writes)
  - GSIs are eventually consistent (slight lag from base table)
  - Max 20 GSIs per table
  - Hot partition on GSI can throttle writes to the base table
```

### Single-Table Design Pattern

Instead of multiple tables (orders, customers, products), DynamoDB experts put **everything in one table** using prefixed keys.

```
┌────────────┬───────────────────┬───────────────────────────┐
│ PK         │ SK                │ Attributes                │
├────────────┼───────────────────┼───────────────────────────┤
│ CUST#001   │ PROFILE           │ name, email, created_at   │
│ CUST#001   │ ORD#2024-01-15    │ amount, status, items     │
│ CUST#001   │ ORD#2024-02-20    │ amount, status, items     │
│ PROD#A100  │ METADATA          │ name, price, category     │
│ PROD#A100  │ REVIEW#2024-01-20 │ rating, comment, user_id  │
│ PROD#A100  │ REVIEW#2024-02-15 │ rating, comment, user_id  │
└────────────┴───────────────────┴───────────────────────────┘

Access patterns:
  Customer profile:   PK='CUST#001', SK='PROFILE'
  Customer orders:    PK='CUST#001', SK begins_with('ORD#')
  Product reviews:    PK='PROD#A100', SK begins_with('REVIEW#')
  Product metadata:   PK='PROD#A100', SK='METADATA'
```

### 💡 Interview Insight

> **"How do you design a DynamoDB schema?"**
>
> *"DynamoDB design is access-pattern-first — the opposite of SQL where you normalize first and query later. I start by listing every access pattern my application needs, then work backward to design keys that serve those patterns efficiently. The partition key should distribute data evenly (avoid hot partitions), and the sort key should enable the range queries I need. For additional access patterns, I use GSIs. For related entities (customers + their orders), I use single-table design with prefixed keys to fetch everything in one query. The cardinal sin is designing DynamoDB like a relational database — you'll end up with scans everywhere."*

---

## Screen 2: MongoDB — Documents and Aggregation

### The Document Model

MongoDB stores data as **BSON documents** (binary JSON). Each document is schema-flexible — different documents in the same collection can have different fields.

```javascript
// orders collection
{
  _id: ObjectId("65a1b2c3d4e5f6a7b8c9d0e1"),
  customer: {
    id: "CUST-001",
    name: "Alice Chen",
    email: "alice@example.com"
  },
  items: [
    { product_id: "PROD-A100", name: "Wireless Mouse", qty: 2, price: 29.99 },
    { product_id: "PROD-B200", name: "USB-C Hub", qty: 1, price: 49.99 }
  ],
  total: 109.97,
  status: "shipped",
  order_date: ISODate("2024-03-15T14:30:00Z"),
  shipping: {
    address: { street: "123 Main St", city: "Austin", state: "TX", zip: "78701" },
    carrier: "UPS",
    tracking: "1Z999AA10123456784"
  }
}
```

**Key difference from SQL**: Related data (customer info, line items, shipping) is **embedded** in a single document instead of normalized across tables. This means one read to fetch everything — no JOINs needed.

### When to Embed vs Reference

| Pattern          | Embed (denormalize)              | Reference (normalize)            |
|:-----------------|:---------------------------------|:---------------------------------|
| Relationship     | 1:1, 1:few                       | 1:many, many:many                |
| Read pattern     | Always read together             | Read independently               |
| Update pattern   | Rarely changes                   | Changes frequently               |
| Document size    | < 16 MB limit                    | Risk of exceeding limit          |
| Example          | Order ← embed → line items       | Product ← reference → reviews    |

### Aggregation Pipeline

MongoDB's aggregation pipeline is a sequence of **stages** that transform documents — conceptually similar to Unix pipes.

```javascript
// "Find the top 5 customers by total spend in Q1 2024"
db.orders.aggregate([
  // Stage 1: Filter to Q1 2024
  { $match: {
      order_date: {
        $gte: ISODate("2024-01-01"),
        $lt:  ISODate("2024-04-01")
      }
  }},

  // Stage 2: Group by customer, sum their totals
  { $group: {
      _id: "$customer.id",
      customer_name: { $first: "$customer.name" },
      total_spend: { $sum: "$total" },
      order_count: { $sum: 1 },
      avg_order: { $avg: "$total" }
  }},

  // Stage 3: Sort by total spend descending
  { $sort: { total_spend: -1 } },

  // Stage 4: Take top 5
  { $limit: 5 },

  // Stage 5: Reshape output
  { $project: {
      customer_id: "$_id",
      customer_name: 1,
      total_spend: { $round: ["$total_spend", 2] },
      order_count: 1,
      avg_order: { $round: ["$avg_order", 2] },
      _id: 0
  }}
]);
```

```
SQL equivalent:
  SELECT customer_id, customer_name,
         ROUND(SUM(total), 2) AS total_spend,
         COUNT(*) AS order_count,
         ROUND(AVG(total), 2) AS avg_order
  FROM orders
  WHERE order_date >= '2024-01-01' AND order_date < '2024-04-01'
  GROUP BY customer_id, customer_name
  ORDER BY total_spend DESC
  LIMIT 5;
```

### $unwind — Exploding Arrays (Like UNNEST/LATERAL)

```javascript
// "Which products generate the most revenue?"
db.orders.aggregate([
  { $unwind: "$items" },     // One doc per item (like unnesting)
  { $group: {
      _id: "$items.product_id",
      product_name: { $first: "$items.name" },
      total_revenue: { $sum: { $multiply: ["$items.qty", "$items.price"] } },
      times_ordered: { $sum: "$items.qty" }
  }},
  { $sort: { total_revenue: -1 } },
  { $limit: 10 }
]);
```

### MongoDB Indexes

```javascript
// Single-field index
db.orders.createIndex({ "customer.id": 1 });                // ascending
db.orders.createIndex({ order_date: -1 });                   // descending

// Compound index (same leftmost-prefix rules as SQL!)
db.orders.createIndex({ status: 1, order_date: -1 });

// Multikey index (automatically indexes array elements)
db.orders.createIndex({ "items.product_id": 1 });
// Now: db.orders.find({ "items.product_id": "PROD-A100" }) uses the index

// Text index for search
db.products.createIndex({ name: "text", description: "text" });
db.products.find({ $text: { $search: "wireless bluetooth headphones" } });

// TTL index (auto-delete documents after expiration)
db.sessions.createIndex({ created_at: 1 }, { expireAfterSeconds: 3600 });
// Documents auto-delete 1 hour after created_at — perfect for sessions!
```

### 💡 Interview Insight

> **"When would you choose MongoDB over PostgreSQL?"**
>
> *"I'd choose MongoDB when the data is naturally document-shaped and varies across records — like product catalogs with different attributes per category, user-generated content with flexible schemas, or IoT event data. MongoDB's embedded documents eliminate JOINs for read-heavy patterns, and the schema flexibility speeds up development when requirements evolve quickly. But I'd stick with PostgreSQL when I need ACID transactions across multiple entities, complex joins, or strong schema enforcement. With PostgreSQL's JSONB support, many document use cases can be handled without leaving the relational world."*

---

## Screen 3: Redis — The In-Memory Swiss Army Knife

### Data Structures — Not Just Key-Value

Redis is an **in-memory data structure store**. The "value" in each key-value pair can be one of several rich data structures.

```
┌──────────────────────────────────────────────────────────────┐
│  Redis Data Structures                                        │
│                                                               │
│  STRING    "session:abc123" → "user_data_json_blob"           │
│  HASH      "user:1001"     → {name: "Alice", email: "..."}   │
│  LIST      "queue:emails"  → [msg3, msg2, msg1]              │
│  SET       "product:42:tags" → {electronics, bluetooth, audio}│
│  SORTED SET "leaderboard"  → {alice:9500, bob:8200, ...}     │
│  STREAM    "events:orders" → timestamped event log            │
└──────────────────────────────────────────────────────────────┘
```

### Strings — Caching and Counters

```redis
-- Cache a serialized object with TTL
SET product:42:details '{"name":"Wireless Mouse","price":29.99}' EX 3600
GET product:42:details

-- Atomic counter (page views, inventory)
INCR product:42:view_count          -- 1, 2, 3, ...
INCRBY product:42:stock -1          -- decrement stock atomically

-- Distributed lock (simplified)
SET lock:checkout:order-789 "worker-3" NX EX 30
-- NX = only set if not exists, EX 30 = expire in 30 seconds
```

### Hashes — Structured Objects

```redis
-- Store a user profile as a hash (like a row in a table)
HSET user:1001 name "Alice Chen" email "alice@example.com" plan "premium"
HGET user:1001 name              -- "Alice Chen"
HGETALL user:1001                -- all fields and values
HINCRBY user:1001 login_count 1  -- atomic field increment

-- More memory-efficient than storing serialized JSON in a string
-- when you frequently read/write individual fields
```

### Lists — Queues and Activity Feeds

```redis
-- Message queue (producer/consumer)
LPUSH queue:emails '{"to":"alice@...","subject":"Order Shipped"}'
RPOP queue:emails          -- consumer grabs from the other end (FIFO)
BRPOP queue:emails 30      -- blocking pop — waits up to 30s for a message

-- Recent activity feed (keep last 100 items)
LPUSH user:1001:feed "Purchased Wireless Mouse"
LTRIM user:1001:feed 0 99  -- trim to 100 entries
LRANGE user:1001:feed 0 9  -- get 10 most recent
```

### Sets and Sorted Sets

```redis
-- Sets: unique collections, set operations
SADD product:42:buyers "user:1001" "user:1002" "user:1003"
SADD product:99:buyers "user:1002" "user:1004"
SINTER product:42:buyers product:99:buyers   -- {"user:1002"} (bought both)
SUNION product:42:buyers product:99:buyers   -- all buyers of either
SCARD product:42:buyers                      -- 3 (count)

-- Sorted Sets: ordered by score — perfect for leaderboards and rankings
ZADD leaderboard 9500 "alice" 8200 "bob" 7800 "charlie"
ZREVRANGE leaderboard 0 2 WITHSCORES   -- top 3: alice(9500), bob(8200), charlie(7800)
ZRANK leaderboard "bob"                 -- rank 1 (0-indexed)
ZINCRBY leaderboard 500 "charlie"       -- charlie now 8300, overtakes bob!
```

### Sorted Sets for Time-Series and Rate Limiting

```redis
-- Rate limiting: track API requests per user per minute
-- Score = timestamp, Member = unique request ID
ZADD ratelimit:user:1001 1711987200 "req-abc"
ZADD ratelimit:user:1001 1711987201 "req-def"

-- Count requests in the last 60 seconds
ZCOUNT ratelimit:user:1001 (NOW-60) +inf

-- Clean up old entries
ZREMRANGEBYSCORE ratelimit:user:1001 0 (NOW-60)
```

### 💡 Interview Insight

> **"How would you implement a real-time leaderboard for an e-commerce platform showing top sellers?"**
>
> *"Redis Sorted Sets are perfect for this. Each time a sale occurs, I'd run `ZINCRBY leaderboard:daily <revenue> <seller_id>` to atomically increment the seller's score. To get the top 10 sellers: `ZREVRANGE leaderboard:daily 0 9 WITHSCORES`. For a specific seller's rank: `ZREVRANK leaderboard:daily <seller_id>`. I'd create separate keys for daily/weekly/monthly leaderboards and expire old ones with TTL. The entire leaderboard is O(log N) for updates and O(log N + M) for range queries — fast enough for millions of sellers with sub-millisecond responses."*

---

## Screen 4: Redis Patterns — Caching, Pub/Sub, and Streams

### Cache-Aside (Lazy Loading) Pattern

```
┌──────────┐   1. GET product:42    ┌──────────┐
│          │ ◄──────────────────────│          │
│   App    │     Cache MISS (nil)   │  Redis   │
│  Server  │                        │  Cache   │
│          │                        │          │
│          │   2. SELECT * FROM     ┌──────────┐
│          │ ─────────────────────→ │PostgreSQL│
│          │   ← result ───────────│          │
│          │                        └──────────┘
│          │   3. SET product:42    ┌──────────┐
│          │ ─────────────────────→ │  Redis   │
│          │      result EX 3600    │  Cache   │
└──────────┘                        └──────────┘

Next request for product:42 → Cache HIT → skip database entirely!
```

```python
def get_product(product_id: int) -> dict:
    cache_key = f"product:{product_id}"

    # 1. Try cache first
    cached = redis.get(cache_key)
    if cached:
        return json.loads(cached)

    # 2. Cache miss — query database
    product = db.execute("SELECT * FROM products WHERE id = %s", product_id)

    # 3. Populate cache with TTL
    redis.set(cache_key, json.dumps(product), ex=3600)  # 1 hour TTL
    return product

def update_product(product_id: int, data: dict):
    db.execute("UPDATE products SET ... WHERE id = %s", product_id)
    redis.delete(f"product:{product_id}")  # Invalidate cache
```

### Eviction Policies

When Redis runs out of memory, it evicts keys based on the configured policy.

| Policy              | Behavior                                        | Best For            |
|:--------------------|:------------------------------------------------|:--------------------|
| `noeviction`        | Reject writes (return error)                    | Critical data       |
| `allkeys-lru`       | Evict least recently used across all keys       | **General caching** |
| `volatile-lru`      | LRU only among keys with TTL set                | Mixed data + cache  |
| `allkeys-lfu`       | Evict least frequently used                     | Frequency matters   |
| `allkeys-random`    | Random eviction                                 | Uniform access      |
| `volatile-ttl`      | Evict keys closest to expiration                | TTL-heavy workloads |

### Pub/Sub — Real-Time Event Broadcasting

```redis
-- Subscriber (listens for events)
SUBSCRIBE orders:new
-- Waiting for messages...

-- Publisher (broadcasts events)
PUBLISH orders:new '{"order_id":"ORD-501","customer":"alice","total":89.99}'

-- Pattern-based subscription
PSUBSCRIBE orders:*
-- Receives: orders:new, orders:shipped, orders:cancelled, etc.
```

```
⚠️ Pub/Sub limitations:
  - Fire-and-forget: if no subscriber is listening, the message is LOST
  - No message persistence or replay
  - No consumer groups or load balancing
  → Use Redis Streams for reliable messaging
```

### Redis Streams — Reliable Event Log

Streams are an **append-only log** with consumer groups — like a lightweight Kafka.

```redis
-- Producer: add events to a stream
XADD orders:stream * customer_id CUST-001 amount 89.99 status new
-- Returns: "1711987200000-0" (auto-generated ID = timestamp-sequence)

-- Consumer: read new events
XREAD COUNT 10 BLOCK 5000 STREAMS orders:stream $
-- Reads up to 10 events, blocks for 5 seconds if none available

-- Consumer groups: distribute work across multiple consumers
XGROUP CREATE orders:stream processors $ MKSTREAM
XREADGROUP GROUP processors worker-1 COUNT 1 BLOCK 5000 STREAMS orders:stream >
XREADGROUP GROUP processors worker-2 COUNT 1 BLOCK 5000 STREAMS orders:stream >
-- Events are load-balanced across worker-1 and worker-2

-- Acknowledge processing (removes from pending list)
XACK orders:stream processors "1711987200000-0"
```

### 💡 Interview Insight

> **"How do you handle cache invalidation?"**
>
> *"Cache invalidation is famously one of the two hard problems in CS. I use three strategies depending on the use case: (1) **TTL-based expiration** — simple, works for data that can be slightly stale (product listings). (2) **Write-through invalidation** — on every database write, delete or update the cache key. Simple but risks race conditions if two writers overlap. (3) **Event-driven invalidation** — database changes emit events (via CDC or application events) to a Redis Stream, and a consumer invalidates or refreshes the affected cache keys. This decouples writes from cache management. I avoid the 'cache stampede' problem — where a popular key expires and 1000 requests simultaneously hit the database — by using a mutex lock or probabilistic early expiration."*

---

## Screen 5: Neo4j — Graphs and Cypher

### When Graphs Win

Relational databases handle graphs poorly — recursive self-joins, CTEs with unknown depth, exponential query complexity. Graph databases store relationships as **first-class citizens**.

```
Use cases where graphs excel:
  - Social networks: "Friends of friends who bought X"
  - Recommendation engines: "Customers who bought A also bought B"
  - Fraud detection: "Find circular money flows"
  - Knowledge graphs: "What entities are related to X?"
  - Network topology: "Shortest path between nodes"
```

### The Property Graph Model

```
Nodes (entities) have labels and properties:
  (:Customer {id: "C001", name: "Alice", tier: "gold"})
  (:Product  {id: "P100", name: "Wireless Mouse", price: 29.99})
  (:Category {name: "Electronics"})

Relationships (edges) have types, direction, and properties:
  (:Customer)-[:PURCHASED {date: "2024-03-15", qty: 2}]->(:Product)
  (:Product)-[:BELONGS_TO]->(:Category)
  (:Customer)-[:REVIEWED {rating: 5, text: "Great!"}]->(:Product)
```

```
Visual:

  (Alice)──PURCHASED──→(Wireless Mouse)──BELONGS_TO──→(Electronics)
     │                        ↑
     │                        │
     └──REVIEWED {★★★★★}──────┘
```

### Cypher Query Language

```cypher
// Find all products Alice purchased
MATCH (c:Customer {name: "Alice"})-[:PURCHASED]->(p:Product)
RETURN p.name, p.price;

// Recommendation: "Customers who bought X also bought..."
MATCH (target:Product {name: "Wireless Mouse"})<-[:PURCHASED]-
      (customer)-[:PURCHASED]->(recommended:Product)
WHERE recommended <> target
RETURN recommended.name, COUNT(customer) AS co_purchases
ORDER BY co_purchases DESC
LIMIT 5;

// Find the shortest path between two customers (social network)
MATCH path = shortestPath(
  (a:Customer {name: "Alice"})-[:FRIENDS_WITH*..6]-(b:Customer {name: "Zara"})
)
RETURN path, length(path);

// Fraud detection: find circular payment flows
MATCH path = (a:Account)-[:TRANSFERRED_TO*3..5]->(a)
WHERE ALL(t IN relationships(path) WHERE t.amount > 10000)
RETURN path;
```

### Cypher vs SQL Comparison

```
SQL (find friends-of-friends):
  SELECT DISTINCT f2.friend_id
  FROM friendships f1
  JOIN friendships f2 ON f1.friend_id = f2.user_id
  WHERE f1.user_id = 'alice'
    AND f2.friend_id != 'alice';
  -- Gets ugly fast at 3+ hops. At 6 hops? Forget about it.

Cypher:
  MATCH (:Customer {name:"Alice"})-[:FRIENDS_WITH*2]->(fof:Customer)
  RETURN DISTINCT fof.name;
  -- Change *2 to *6 for 6 hops — same simplicity!
```

### 💡 Interview Insight

> **"When would you use a graph database instead of a relational database?"**
>
> *"I'd reach for a graph database when the core value of the data is in the relationships, not the entities — and when queries need to traverse those relationships to variable depth. Classic examples: social networks (friends-of-friends), recommendation engines (collaborative filtering), fraud detection (circular flows), and knowledge graphs. The key indicator is when your SQL queries need recursive CTEs or multiple self-joins that grow with traversal depth. Graph databases store relationships as first-class pointers, making multi-hop traversals O(1) per hop instead of requiring full-table joins. But for transactional e-commerce CRUD, aggregations, and reporting — SQL is still king."*

---

## Screen 6: SQL vs NoSQL — The Decision Framework

### It's Not SQL *or* NoSQL — It's SQL *and* NoSQL

Most production systems use **polyglot persistence** — the right database for each use case.

```
Typical e-commerce architecture:

┌─────────────────────────────────────────────────────────┐
│                    Application Layer                      │
└──┬──────────┬──────────┬──────────┬──────────┬──────────┘
   │          │          │          │          │
   ▼          ▼          ▼          ▼          ▼
┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐
│Postgre│  │Redis │  │Dynamo│  │Elastic│  │Neo4j │
│SQL    │  │      │  │DB    │  │Search │  │      │
├───────┤  ├──────┤  ├──────┤  ├──────┤  ├──────┤
│Orders │  │Cache │  │User  │  │Product│  │Recom-│
│Payments│ │Sessions│ │Prefs │  │Search │  │menda-│
│Inventory││Rate  │  │Cart  │  │Logs   │  │tions │
│Users  │  │Limit │  │Activity│ │Analytics││Fraud │
└───────┘  └──────┘  └──────┘  └──────┘  └──────┘
 Source     Speed      Scale    Full-text  Graphs
 of Truth   Layer      Layer    Layer      Layer
```

### The Decision Matrix

| Factor                    | SQL (PostgreSQL)          | DynamoDB             | MongoDB              | Redis                |
|:--------------------------|:--------------------------|:---------------------|:---------------------|:---------------------|
| **Data model**            | Fixed schema, relations   | Key-value + document | Flexible documents   | Data structures      |
| **Query flexibility**     | ★★★★★ (ad-hoc JOINs)     | ★★ (key-based only)  | ★★★★ (aggregation)   | ★★ (key-based)       |
| **Write scale**           | ★★★ (vertical)            | ★★★★★ (horizontal)   | ★★★★ (horizontal)    | ★★★★★ (in-memory)    |
| **Read latency**          | ★★★ (ms range)            | ★★★★ (single-digit ms)| ★★★ (ms range)       | ★★★★★ (sub-ms)       |
| **ACID transactions**     | ✅ Full                    | ⚠️ Limited (2 items)  | ✅ Multi-doc (4.0+)   | ⚠️ Lua scripts       |
| **Schema evolution**      | Migrations required       | Schema-free          | Schema-free          | Schema-free          |
| **Operational cost**      | Self-managed or RDS       | Fully managed        | Atlas (managed)      | ElastiCache/self     |
| **Best for**              | Source of truth, reports  | Massive scale, known access patterns | Flexible docs, prototyping | Caching, real-time |

### Decision Flowchart

```
START: "What kind of data access do I need?"
  │
  ├─→ Complex queries, JOINs, aggregations, ad-hoc analysis?
  │     → PostgreSQL (or MySQL). No question.
  │
  ├─→ Simple key-value lookups at massive scale (100K+ req/s)?
  │     → DynamoDB or Cassandra. Design your keys carefully.
  │
  ├─→ Flexible documents with varying schemas?
  │     │
  │     ├─→ Need ACID transactions?  → MongoDB 4.0+ or PostgreSQL JSONB
  │     └─→ Don't need transactions? → MongoDB
  │
  ├─→ Sub-millisecond latency for hot data?
  │     → Redis (cache layer in front of your primary DB)
  │
  ├─→ Relationship traversal (friends-of-friends, recommendations)?
  │     → Neo4j or Amazon Neptune
  │
  └─→ Full-text search with relevance ranking?
        → Elasticsearch or PostgreSQL full-text (if simpler needs)
```

### Common Anti-Patterns

```
❌ "We chose MongoDB because it's web-scale"
   → Then they needed JOINs, transactions, and complex reports.
   → Should have used PostgreSQL.

❌ "We put everything in Redis for speed"
   → Redis is RAM-bound. Storing 500GB of data in Redis costs a fortune.
   → Use Redis as a CACHE in front of a cheaper persistent store.

❌ "We normalized everything in DynamoDB"
   → Created 12 tables with "joins" in application code = slow & expensive.
   → Should have used single-table design or switched to SQL.

❌ "We stored time-series in a regular SQL table"
   → 10 billion rows, queries timing out.
   → Should have used TimescaleDB, InfluxDB, or at minimum partitioned tables.
```

### 💡 Interview Insight

> **"How do you choose between SQL and NoSQL for a new project?"**
>
> *"I start with three questions: (1) What are my access patterns — do I need flexible ad-hoc queries or are they predefined? SQL for ad-hoc, NoSQL for predefined. (2) What are my consistency requirements — do I need strict ACID or is eventual consistency acceptable? Strict ACID points to PostgreSQL. (3) What's my scale profile — read-heavy, write-heavy, or both? For extreme write scale with simple lookups, DynamoDB; for read-heavy analytics, PostgreSQL with read replicas. In practice, I default to PostgreSQL (it handles 90% of use cases including JSONB for documents) and add specialized stores only when PostgreSQL becomes a bottleneck for specific access patterns."*

---

## Screen 7: Quiz

**Q1: In DynamoDB, what determines which physical partition stores an item?**
- A) The sort key
- B) The partition key  ✅
- C) The Global Secondary Index
- D) The table's region setting

> **Answer: B** — The partition key is hashed to determine the physical partition. All items with the same partition key are stored together (co-located), which is why partition key design is critical for even data distribution. The sort key orders items *within* a partition but doesn't affect partition placement.

**Q2: Which MongoDB aggregation stage is equivalent to SQL's `UNNEST` or a LATERAL join on an array?**
- A) `$group`
- B) `$lookup`
- C) `$unwind`  ✅
- D) `$project`

> **Answer: C** — `$unwind` deconstructs an array field, creating one output document per array element. This is equivalent to unnesting/exploding an array in SQL. `$lookup` is MongoDB's equivalent of a LEFT OUTER JOIN. `$group` is like GROUP BY, and `$project` is like SELECT.

**Q3: Which Redis data structure would you use for a real-time leaderboard that needs to efficiently return top-N rankings?**
- A) Hash
- B) List
- C) Set
- D) Sorted Set  ✅

> **Answer: D** — Sorted Sets maintain elements ordered by score with O(log N) insertion and O(log N + M) range retrieval. `ZREVRANGE leaderboard 0 9 WITHSCORES` returns the top 10 in sub-millisecond time. Lists are ordered by insertion time (not score), Sets are unordered, and Hashes have no ordering.

**Q4: What is the main limitation of Redis Pub/Sub compared to Redis Streams?**
- A) Pub/Sub is slower than Streams
- B) Pub/Sub messages are lost if no subscriber is listening  ✅
- C) Pub/Sub doesn't support pattern matching
- D) Pub/Sub can only send string messages

> **Answer: B** — Pub/Sub is fire-and-forget. If no subscriber is connected when a message is published, it's gone forever. Redis Streams persist messages in an append-only log, support consumer groups for load balancing, and allow message replay — making them suitable for reliable event processing.

**Q5: In Neo4j Cypher, what does `MATCH (a)-[:KNOWS*2..4]->(b)` mean?**
- A) Find nodes where `a` knows exactly 2 to 4 people
- B) Traverse 2 to 4 hops along KNOWS relationships from `a` to `b`  ✅
- C) Find the 2nd through 4th KNOWS relationships in the database
- D) Return paths where KNOWS has a weight between 2 and 4

> **Answer: B** — The `*2..4` syntax specifies a variable-length path of 2 to 4 hops along KNOWS relationships. This is the graph equivalent of 2 to 4 self-joins in SQL — but expressed in one elegant line. This is one of the key advantages of graph databases: variable-depth traversals are trivial to express and efficient to execute.

---

## Screen 8: Key Takeaways

- **DynamoDB is access-pattern-first.** Design your partition key for even distribution and your sort key for the range queries you need. Use GSIs for additional access patterns. Single-table design packs related entities into one table with prefixed keys — powerful but requires careful planning.

- **MongoDB embeds related data in documents** to avoid JOINs. Use embedding for 1:few relationships read together; use references for 1:many relationships or frequently-updated data. The aggregation pipeline (`$match → $group → $sort → $project`) maps to SQL concepts but operates on documents.

- **Redis is a data structure server**, not just a cache. Strings for simple caching, Hashes for objects, Lists for queues, Sets for membership, Sorted Sets for leaderboards/rankings. Choose the right structure for your access pattern.

- **Redis caching patterns**: Cache-aside (lazy loading) is the most common — check cache, miss → query DB → populate cache. Always set TTLs. Invalidate on writes. Prevent cache stampedes with locking or probabilistic early expiration.

- **Neo4j and graph databases** excel at relationship-heavy queries — social networks, recommendations, fraud detection. Cypher makes variable-depth traversals trivial (`*2..6`) compared to recursive CTEs in SQL.

- **Default to PostgreSQL** for new projects — it handles relational data, JSONB documents, full-text search, and even time-series (with TimescaleDB). Add specialized NoSQL stores only when you hit a specific bottleneck that PostgreSQL can't address efficiently.

- **Polyglot persistence is the norm.** Production systems use PostgreSQL as the source of truth, Redis for caching and real-time features, and maybe DynamoDB or Elasticsearch for specific high-scale access patterns. The key is knowing which tool fits which problem.
