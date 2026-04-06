# Module 6: The Interview Gauntlet 🔥

> **Goal**: Battle-test your knowledge with the exact types of questions you'll face in data engineering and backend engineering interviews. Each question includes a detailed answer with the reasoning an interviewer expects. Don't just memorize — understand the *why*.

---

## Screen 1: Architecture & Design Questions

### Q1: "Design a schema for an e-commerce order system that handles millions of orders per day."

**Answer:**

```sql
-- Core tables
CREATE TABLE customers (
    customer_id   BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email         TEXT UNIQUE NOT NULL,
    name          TEXT NOT NULL,
    created_at    TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE products (
    product_id    BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name          TEXT NOT NULL,
    category_id   INT REFERENCES categories(category_id),
    price         DECIMAL(10,2) NOT NULL,
    attributes    JSONB DEFAULT '{}'
);

CREATE TABLE orders (
    order_id      BIGINT GENERATED ALWAYS AS IDENTITY,
    customer_id   BIGINT NOT NULL REFERENCES customers(customer_id),
    order_date    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    status        TEXT NOT NULL DEFAULT 'pending',
    total_amount  DECIMAL(12,2) NOT NULL,
    PRIMARY KEY (order_id, order_date)
) PARTITION BY RANGE (order_date);

CREATE TABLE order_items (
    item_id       BIGINT GENERATED ALWAYS AS IDENTITY,
    order_id      BIGINT NOT NULL,
    order_date    TIMESTAMPTZ NOT NULL,
    product_id    BIGINT NOT NULL REFERENCES products(product_id),
    quantity      INT NOT NULL CHECK (quantity > 0),
    unit_price    DECIMAL(10,2) NOT NULL,
    FOREIGN KEY (order_id, order_date) REFERENCES orders(order_id, order_date)
);
```

**Key design decisions:**
- **Partitioned orders table** by `order_date` for time-range query performance and data retention (drop old partitions).
- **`unit_price` on order_items** — captures the price at time of purchase (products change price!).
- **JSONB `attributes`** on products for variable attributes across categories.
- **`GENERATED ALWAYS AS IDENTITY`** over `SERIAL` — modern PostgreSQL best practice.
- **Indexes**: `orders(customer_id, order_date)`, `order_items(order_id)`, `order_items(product_id)`, `products(category_id)`.

---

### Q2: "Your orders table has 500M rows and a dashboard query aggregating monthly revenue takes 45 seconds. How do you optimize it?"

**Answer — layered approach:**

1. **Materialized view** for pre-computed daily/monthly rollups:
   ```sql
   CREATE MATERIALIZED VIEW mv_monthly_revenue AS
   SELECT DATE_TRUNC('month', order_date) AS month,
          COUNT(*) AS order_count,
          SUM(total_amount) AS revenue
   FROM orders
   GROUP BY DATE_TRUNC('month', order_date);
   ```
   Refresh nightly. Dashboard queries hit the mat view (milliseconds, not seconds).

2. **Partition the orders table** by month — partition pruning reduces scan to ~10M rows per month.

3. **Covering index** if the query doesn't need a mat view:
   ```sql
   CREATE INDEX idx_orders_date_amount ON orders (order_date) INCLUDE (total_amount);
   ```
   Index-only scan avoids heap fetches.

4. **Check `pg_stat_statements`** — is the query running frequently? What's the cache hit rate? Low cache hit → increase `shared_buffers`.

5. **Check for bloat** — `VACUUM ANALYZE orders` if stats are stale or dead tuples are excessive.

---

### Q3: "Design a schema for a product review system with ratings, text reviews, and helpful/not-helpful votes."

**Answer:**

```sql
CREATE TABLE reviews (
    review_id    BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    product_id   BIGINT NOT NULL REFERENCES products(product_id),
    customer_id  BIGINT NOT NULL REFERENCES customers(customer_id),
    rating       SMALLINT NOT NULL CHECK (rating BETWEEN 1 AND 5),
    title        TEXT,
    body         TEXT,
    created_at   TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE (product_id, customer_id)  -- one review per customer per product
);

CREATE TABLE review_votes (
    vote_id      BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    review_id    BIGINT NOT NULL REFERENCES reviews(review_id),
    customer_id  BIGINT NOT NULL REFERENCES customers(customer_id),
    is_helpful   BOOLEAN NOT NULL,
    UNIQUE (review_id, customer_id)  -- one vote per customer per review
);

-- Denormalized counters on reviews for fast display
ALTER TABLE reviews ADD COLUMN helpful_count INT DEFAULT 0;
ALTER TABLE reviews ADD COLUMN unhelpful_count INT DEFAULT 0;

-- Key indexes
CREATE INDEX idx_reviews_product ON reviews (product_id, rating);
CREATE INDEX idx_reviews_customer ON reviews (customer_id);
CREATE INDEX idx_reviews_search ON reviews USING GIN (to_tsvector('english', body));
```

**Design reasoning**: The `UNIQUE(product_id, customer_id)` constraint prevents duplicate reviews. Denormalized counters (`helpful_count`) avoid counting votes on every page load — updated via trigger or application code. GIN index enables full-text search on review body.

---

### Q4: "How would you implement a flash sale where only 100 units are available and thousands of users try to buy simultaneously?"

**Answer:**
```sql
-- Approach 1: SELECT FOR UPDATE with NOWAIT
BEGIN;
SELECT quantity FROM inventory
WHERE product_id = 42 AND quantity > 0
FOR UPDATE NOWAIT;
-- If locked → immediately fails, user sees "try again"

UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 42;
INSERT INTO orders (...) VALUES (...);
COMMIT;

-- Approach 2: Atomic UPDATE with RETURNING (simpler, no explicit lock)
UPDATE inventory
SET quantity = quantity - 1
WHERE product_id = 42 AND quantity > 0
RETURNING quantity;
-- If no rows returned → out of stock. Atomic, no race condition.
```

**The atomic UPDATE approach is preferred** — it's a single statement, inherently serialized by PostgreSQL's row-level lock, and returns the result in one round trip. No explicit BEGIN/COMMIT needed for a single statement. For distributed systems, add an idempotency key to prevent double-ordering.

---

### Q5: "Your application has a slow query that joins 6 tables. How do you systematically diagnose and fix it?"

**Answer — systematic approach:**

1. **EXPLAIN ANALYZE** — get the actual execution plan with timing.
2. **Find the bottleneck node** — which step takes the longest?
3. **Check estimated vs actual rows** — big mismatch → run `ANALYZE` on the tables.
4. **Look for Seq Scans on large tables** — add indexes for selective filters.
5. **Check join order** — is the optimizer joining in a bad order? Consider restructuring the query.
6. **Consider breaking it up** — use CTEs or temporary tables for complex intermediate results.
7. **Check for implicit type casts** — `WHERE varchar_col = 123` prevents index use.
8. **Look at `work_mem`** — sorts or hash joins spilling to disk? Increase `work_mem`.

---

### Q6: "Design an audit trail that tracks every change to the orders table."

**Answer:**
```sql
CREATE TABLE orders_audit (
    audit_id     BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    order_id     BIGINT NOT NULL,
    operation    TEXT NOT NULL,  -- INSERT, UPDATE, DELETE
    changed_at   TIMESTAMPTZ DEFAULT NOW(),
    changed_by   TEXT DEFAULT current_user,
    old_values   JSONB,
    new_values   JSONB
);

CREATE OR REPLACE FUNCTION audit_orders() RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'DELETE' THEN
        INSERT INTO orders_audit (order_id, operation, old_values)
        VALUES (OLD.order_id, 'DELETE', to_jsonb(OLD));
        RETURN OLD;
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO orders_audit (order_id, operation, old_values, new_values)
        VALUES (OLD.order_id, 'UPDATE', to_jsonb(OLD), to_jsonb(NEW));
        RETURN NEW;
    ELSIF TG_OP = 'INSERT' THEN
        INSERT INTO orders_audit (order_id, operation, new_values)
        VALUES (NEW.order_id, 'INSERT', to_jsonb(NEW));
        RETURN NEW;
    END IF;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_orders_audit
    AFTER INSERT OR UPDATE OR DELETE ON orders
    FOR EACH ROW EXECUTE FUNCTION audit_orders();
```

Using JSONB for `old_values`/`new_values` means the audit table doesn't need to change when the orders schema evolves.

---

### Q7: "How would you design a database for multi-tenant SaaS?"

**Answer — three approaches:**

| Approach             | Isolation | Complexity | Use Case                    |
|:---------------------|:----------|:-----------|:----------------------------|
| Shared table + `tenant_id` | Low    | Low        | Many small tenants          |
| Schema per tenant    | Medium    | Medium     | Moderate tenants, compliance|
| Database per tenant  | High      | High       | Enterprise, strict isolation|

**Most common: shared tables with `tenant_id`:**
```sql
-- Every table gets a tenant_id column
CREATE TABLE orders (
    order_id   BIGINT GENERATED ALWAYS AS IDENTITY,
    tenant_id  INT NOT NULL,
    ...
    PRIMARY KEY (tenant_id, order_id)
);

-- Row-Level Security prevents cross-tenant access
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON orders
    USING (tenant_id = current_setting('app.tenant_id')::int);

-- Every query automatically filtered — no accidental data leaks
SET app.tenant_id = '42';
SELECT * FROM orders;  -- Only sees tenant 42's orders
```

---

### Q8: "Explain the CAP theorem and how it applies to database selection."

**Answer:**

```
CAP Theorem: A distributed system can guarantee at most 2 of 3:

  C — Consistency: Every read gets the most recent write
  A — Availability: Every request gets a response (no errors/timeouts)
  P — Partition tolerance: System works despite network failures

┌───────────────────────────────────────┐
│              CAP Triangle              │
│                                       │
│              Consistency              │
│                 /\                     │
│                /  \                    │
│               / CP \                  │
│              /______\                 │
│             /   CA    \               │
│            /            \             │
│    Availability ──── Partition        │
│                  AP    Tolerance      │
└───────────────────────────────────────┘

CP (Consistency + Partition Tolerance): PostgreSQL, MongoDB
   → Rejects writes during network partitions to stay consistent

AP (Availability + Partition Tolerance): DynamoDB, Cassandra
   → Accepts writes during partitions, resolves conflicts later

CA (Consistency + Availability): Single-node databases
   → Not really distributed — network partitions aren't handled
```

In practice: network partitions **will** happen, so you're choosing between CP and AP. PostgreSQL (single primary) is CP. DynamoDB defaults to AP but offers strongly consistent reads. Most systems use CP for source-of-truth (payments) and AP for high-availability reads (product catalog).

---

## Screen 2: Query Writing Challenges

### Challenge 1: "Find the top 3 products by revenue in each category"

```sql
WITH ranked AS (
    SELECT
        p.category,
        p.name,
        SUM(oi.quantity * oi.unit_price) AS revenue,
        RANK() OVER (
            PARTITION BY p.category
            ORDER BY SUM(oi.quantity * oi.unit_price) DESC
        ) AS rn
    FROM order_items oi
    JOIN products p ON oi.product_id = p.product_id
    GROUP BY p.category, p.name
)
SELECT category, name, revenue
FROM ranked
WHERE rn <= 3
ORDER BY category, revenue DESC;
```

**Why RANK over ROW_NUMBER?** If two products tie in revenue, both appear. Use ROW_NUMBER if you strictly want 3 rows per category.

---

### Challenge 2: "Find customers whose order frequency increased month-over-month for 3 consecutive months"

```sql
WITH monthly_counts AS (
    SELECT customer_id,
           DATE_TRUNC('month', order_date) AS month,
           COUNT(*) AS order_count
    FROM orders
    GROUP BY customer_id, DATE_TRUNC('month', order_date)
),
with_change AS (
    SELECT *,
           order_count - LAG(order_count) OVER (
               PARTITION BY customer_id ORDER BY month
           ) AS change,
           ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY month)
           - ROW_NUMBER() OVER (
               PARTITION BY customer_id,
               CASE WHEN order_count > LAG(order_count) OVER (
                   PARTITION BY customer_id ORDER BY month
               ) THEN 1 ELSE 0 END
               ORDER BY month
           ) AS streak_group
    FROM monthly_counts
),
streaks AS (
    SELECT customer_id,
           COUNT(*) AS consecutive_increases
    FROM with_change
    WHERE change > 0
    GROUP BY customer_id, streak_group
)
SELECT DISTINCT customer_id
FROM streaks
WHERE consecutive_increases >= 3;
```

**Simpler approach using LAG:**
```sql
WITH monthly AS (
    SELECT customer_id,
           DATE_TRUNC('month', order_date) AS month,
           COUNT(*) AS cnt
    FROM orders
    GROUP BY 1, 2
),
flagged AS (
    SELECT *,
           CASE WHEN cnt > LAG(cnt, 1) OVER w
                 AND LAG(cnt, 1) OVER w > LAG(cnt, 2) OVER w
                 AND LAG(cnt, 2) OVER w > LAG(cnt, 3) OVER w
                THEN 1 ELSE 0 END AS three_month_increase
    FROM monthly
    WINDOW w AS (PARTITION BY customer_id ORDER BY month)
)
SELECT DISTINCT customer_id FROM flagged WHERE three_month_increase = 1;
```

---

### Challenge 3: "Sessionize clickstream data with a 30-minute timeout"

```sql
WITH gaps AS (
    SELECT user_id, event_time, page_url,
           EXTRACT(EPOCH FROM (
               event_time - LAG(event_time) OVER (
                   PARTITION BY user_id ORDER BY event_time
               )
           )) / 60.0 AS gap_minutes
    FROM clickstream
),
boundaries AS (
    SELECT *,
           CASE WHEN gap_minutes > 30 OR gap_minutes IS NULL THEN 1 ELSE 0 END
               AS new_session
    FROM gaps
),
sessions AS (
    SELECT *,
           SUM(new_session) OVER (
               PARTITION BY user_id ORDER BY event_time
           ) AS session_id
    FROM boundaries
)
SELECT user_id, session_id, event_time, page_url
FROM sessions
ORDER BY user_id, event_time;
```

**Pattern**: LAG → detect gaps → flag boundaries → running SUM of flags = session IDs.

---

### Challenge 4: "Calculate a 7-day rolling average of daily revenue, including days with no orders"

```sql
-- Generate all dates to fill gaps
WITH date_series AS (
    SELECT generate_series(
        (SELECT MIN(order_date) FROM orders),
        (SELECT MAX(order_date) FROM orders),
        '1 day'::interval
    )::date AS day
),
daily_revenue AS (
    SELECT d.day,
           COALESCE(SUM(o.total_amount), 0) AS revenue
    FROM date_series d
    LEFT JOIN orders o ON o.order_date::date = d.day
    GROUP BY d.day
)
SELECT day, revenue,
       ROUND(AVG(revenue) OVER (
           ORDER BY day ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
       ), 2) AS rolling_7day_avg
FROM daily_revenue
ORDER BY day;
```

**Key insight**: You must generate a complete date series and LEFT JOIN to it, otherwise days with zero orders are missing and the rolling average window shifts incorrectly.

---

### Challenge 5: "Find pairs of products frequently bought together (market basket analysis)"

```sql
SELECT
    p1.name AS product_a,
    p2.name AS product_b,
    COUNT(DISTINCT oi1.order_id) AS co_purchase_count
FROM order_items oi1
JOIN order_items oi2
    ON oi1.order_id = oi2.order_id
    AND oi1.product_id < oi2.product_id   -- avoid duplicate pairs & self-pairs
JOIN products p1 ON oi1.product_id = p1.product_id
JOIN products p2 ON oi2.product_id = p2.product_id
GROUP BY p1.name, p2.name
HAVING COUNT(DISTINCT oi1.order_id) >= 10  -- minimum support threshold
ORDER BY co_purchase_count DESC
LIMIT 20;
```

**The `<` trick**: `oi1.product_id < oi2.product_id` ensures each pair appears only once (A,B but not B,A) and excludes self-pairs.

---

### Challenge 6: "Find the median order value per category"

```sql
-- PostgreSQL: use PERCENTILE_CONT
SELECT
    p.category,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY o.total_amount) AS median_order_value,
    AVG(o.total_amount) AS mean_order_value
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
GROUP BY p.category;

-- Generic SQL (works everywhere):
WITH ranked AS (
    SELECT p.category, o.total_amount,
           ROW_NUMBER() OVER (PARTITION BY p.category ORDER BY o.total_amount) AS rn,
           COUNT(*) OVER (PARTITION BY p.category) AS cnt
    FROM orders o
    JOIN order_items oi ON o.order_id = oi.order_id
    JOIN products p ON oi.product_id = p.product_id
)
SELECT category,
       AVG(total_amount) AS median
FROM ranked
WHERE rn IN (FLOOR((cnt + 1) / 2.0), CEIL((cnt + 1) / 2.0))
GROUP BY category;
```

---

### Challenge 7: "Find customers who made a purchase within 7 days of their first-ever purchase"

```sql
WITH first_orders AS (
    SELECT customer_id,
           MIN(order_date) AS first_order_date
    FROM orders
    GROUP BY customer_id
)
SELECT DISTINCT o.customer_id
FROM orders o
JOIN first_orders f ON o.customer_id = f.customer_id
WHERE o.order_date > f.first_order_date
  AND o.order_date <= f.first_order_date + INTERVAL '7 days';
```

This identifies customers with high early engagement — a key retention metric.

---

## Screen 3: Performance Debugging

### Scenario 1: "This query is doing a Seq Scan on a 50M row table despite having an index on `email`"

```sql
-- The query:
SELECT * FROM customers WHERE LOWER(email) = 'alice@example.com';
```

**Problem**: The index is on `email`, but the query applies `LOWER()` — this prevents index usage because the stored index values don't match the expression being searched.

**Fix**: Create an expression index:
```sql
CREATE INDEX idx_customers_email_lower ON customers (LOWER(email));
```

---

### Scenario 2: "This query uses an index but is still slow"

```sql
EXPLAIN ANALYZE
SELECT * FROM orders WHERE customer_id = 42;
-- Index Scan using idx_orders_customer on orders
-- actual time=0.02..890.45 rows=500000 loops=1
-- Heap Fetches: 500000
```

**Problem**: The index efficiently finds 500K matching rows, but each row requires a **random I/O heap fetch**. With 500K scattered heap fetches, the I/O is enormous.

**Fix options**:
1. Add a covering index: `CREATE INDEX ... ON orders (customer_id) INCLUDE (order_date, amount, status);` → Index Only Scan, no heap fetches.
2. If you don't need all columns, narrow the SELECT list.
3. `CLUSTER orders USING idx_orders_customer;` → physically reorders the table by customer_id (one-time, requires exclusive lock).

---

### Scenario 3: "The query was fast with 1M rows but crawls at 100M rows"

```sql
SELECT DISTINCT customer_id FROM orders WHERE status = 'active';
```

**Problem**: `DISTINCT` on a non-indexed column requires sorting or hashing all matching rows. At 100M rows with maybe 50M "active" rows, that's a massive sort.

**Fix**:
```sql
-- Option 1: Use EXISTS if you're checking membership
SELECT customer_id FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id AND o.status = 'active');

-- Option 2: Index for loose index scan
CREATE INDEX idx_orders_status_customer ON orders (status, customer_id);
```

---

### Scenario 4: "This UPDATE takes 30 minutes on a 10M row table"

```sql
UPDATE orders SET status = 'archived' WHERE order_date < '2023-01-01';
```

**Problem**: This updates millions of rows in a single transaction. Each row generates WAL entries, updates indexes, and creates dead tuples. The transaction holds locks for the entire duration.

**Fix — batch it:**
```sql
-- Process in chunks of 10,000
DO $$
DECLARE
    rows_updated INT;
BEGIN
    LOOP
        UPDATE orders SET status = 'archived'
        WHERE order_id IN (
            SELECT order_id FROM orders
            WHERE order_date < '2023-01-01' AND status != 'archived'
            LIMIT 10000
        );
        GET DIAGNOSTICS rows_updated = ROW_COUNT;
        EXIT WHEN rows_updated = 0;
        COMMIT;  -- Release locks, let VACUUM work
    END LOOP;
END $$;
```

Or if the table is partitioned, simply detach the old partition.

---

### Scenario 5: "Joining two large tables is slow — EXPLAIN shows a Hash Join spilling to disk"

```
Hash Join (actual time=2.1..45600.0 rows=5000000)
  Buckets: 131072  Batches: 32  Memory Usage: 4096kB
```

**Problem**: `Batches: 32` means the hash table didn't fit in `work_mem` and spilled to disk 32 times. Each batch requires a full re-read of the probe side.

**Fix**:
```sql
SET work_mem = '256MB';  -- Increase for this session (default is often 4MB)
-- Rerun the query — should now be Batches: 1 (all in memory)
```

Caution: `work_mem` is per-operation-per-connection. 256MB × 100 connections × 3 operations = 75 GB. Set it per-session, not globally.

---

### Scenario 6: "The query plan shows `Rows Removed by Filter: 4,999,000` out of 5,000,000 scanned"

```sql
SELECT * FROM orders WHERE customer_id = 42 AND order_date > '2024-01-01';
-- Seq Scan on orders
--   Filter: (customer_id = 42 AND order_date > '2024-01-01')
--   Rows Removed by Filter: 4999000
--   Actual rows: 1000
```

**Problem**: No index — scanning 5M rows to find 1,000. 99.98% of rows are thrown away.

**Fix**:
```sql
CREATE INDEX idx_orders_cust_date ON orders (customer_id, order_date);
-- Now: Index Scan, reads ~1000 rows directly
```

Column order matters: `customer_id` first (equality), `order_date` second (range).

---

### Scenario 7: "A simple COUNT(*) on a 100M row table takes 15 seconds"

```sql
SELECT COUNT(*) FROM orders;
-- Seq Scan: actual time=0.03..15234.56 rows=100000000
```

**Problem**: PostgreSQL doesn't maintain an exact row count (due to MVCC, different transactions see different rows). `COUNT(*)` must scan the entire table.

**Fix options**:
1. **Approximate count** (instant): `SELECT reltuples::bigint FROM pg_class WHERE relname = 'orders';`
2. **Materialized counter**: Maintain a counts table updated by triggers.
3. **Index Only Scan**: `CREATE INDEX idx_orders_id ON orders (order_id);` — sometimes the planner can count from a smaller index.
4. **HyperLogLog extension** (`postgresql-hll`) for approximate distinct counts.

---

### Scenario 8: "Adding an index to a production table is blocking writes"

**Problem**: `CREATE INDEX` takes a `SHARE` lock, blocking all INSERT/UPDATE/DELETE for the duration.

**Fix**:
```sql
CREATE INDEX CONCURRENTLY idx_orders_status ON orders (status);
-- Takes longer, but doesn't block writes
-- ⚠️ Cannot be run inside a transaction
-- ⚠️ If it fails, leaves an INVALID index — drop and retry
```

---

## Screen 4: NoSQL Design Questions

### Q1: "Design a DynamoDB schema for an e-commerce shopping cart"

**Answer:**

```
Table: carts
PK: USER#<user_id>
SK: ITEM#<product_id>

┌───────────────┬──────────────────┬─────┬────────┬─────────┐
│ PK            │ SK               │ qty │ price  │ added_at│
├───────────────┼──────────────────┼─────┼────────┼─────────┤
│ USER#alice    │ ITEM#PROD-A100   │ 2   │ 29.99  │ 2024-.. │
│ USER#alice    │ ITEM#PROD-B200   │ 1   │ 49.99  │ 2024-.. │
│ USER#alice    │ METADATA         │     │ 109.97 │         │
│ USER#bob      │ ITEM#PROD-A100   │ 3   │ 29.99  │ 2024-.. │
└───────────────┴──────────────────┴─────┴────────┴─────────┘

Access patterns:
  Get cart:        Query PK='USER#alice', SK begins_with('ITEM#')
  Add item:        PutItem PK='USER#alice', SK='ITEM#PROD-C300'
  Update quantity: UpdateItem PK='USER#alice', SK='ITEM#PROD-A100'
  Remove item:     DeleteItem PK='USER#alice', SK='ITEM#PROD-A100'
  Cart metadata:   GetItem PK='USER#alice', SK='METADATA'

TTL: Set a TTL attribute to auto-expire abandoned carts after 30 days.
```

---

### Q2: "Design a Redis caching strategy for a product catalog with 1M products"

**Answer:**

```
Strategy: Cache-aside with TTL + event-driven invalidation

Key structure:
  product:{id}:detail  → Full product JSON (TTL: 1 hour)
  product:{id}:price   → Just the price (TTL: 5 min for price-sensitive data)
  category:{id}:list   → Sorted set of product IDs scored by popularity

Cache warming:
  On deploy, pre-cache the top 1000 products (80/20 rule)

Invalidation:
  Product update → DELETE product:{id}:detail (lazy reload on next read)
  Price change  → DELETE product:{id}:price AND publish to 'price-changes' channel
  New product   → ZADD to category sorted set

Memory estimation:
  1M products × ~2KB avg JSON = ~2GB
  With 10% hot products cached: ~200MB (comfortable for Redis)

Eviction policy: allkeys-lfu (evict least frequently used)
```

---

### Q3: "When would you use DynamoDB Streams vs Redis Streams?"

**Answer:**

| Feature         | DynamoDB Streams            | Redis Streams                |
|:----------------|:----------------------------|:-----------------------------|
| Source           | Table change events (CDC)   | Application-published events |
| Durability       | Stored 24 hours            | Configurable (in-memory + AOF)|
| Consumer groups  | Lambda triggers / KCL      | Built-in XREADGROUP         |
| Ordering         | Per-partition-key          | Global within stream         |
| Use case         | React to DB changes        | Event bus, job queue         |
| Managed          | Fully managed              | Self-managed or ElastiCache  |

**DynamoDB Streams**: When you need CDC (change data capture) — e.g., sync orders table to Elasticsearch, send notifications on status changes.

**Redis Streams**: When you need an application-level message queue — e.g., processing background jobs, event-driven microservice communication with low latency.

---

### Q4: "Design a rate limiter using Redis"

**Answer — Sliding window using sorted sets:**

```python
import time, redis

r = redis.Redis()

def is_rate_limited(user_id: str, max_requests: int = 100, window_secs: int = 60) -> bool:
    key = f"ratelimit:{user_id}"
    now = time.time()
    window_start = now - window_secs

    pipe = r.pipeline()
    pipe.zremrangebyscore(key, 0, window_start)  # Remove old entries
    pipe.zadd(key, {f"{now}:{uuid4()}": now})     # Add current request
    pipe.zcard(key)                                # Count requests in window
    pipe.expire(key, window_secs)                  # Auto-cleanup
    _, _, request_count, _ = pipe.execute()

    return request_count > max_requests
```

**Why sorted set?** The score is the timestamp, enabling efficient removal of expired entries with `ZREMRANGEBYSCORE`. The window slides precisely — no fixed bucket boundaries that cause edge-case bursts.

---

### Q5: "How would you model a social network 'friends' feature in DynamoDB?"

**Answer — Adjacency list pattern:**

```
Table: social_graph
PK: USER#<user_id>
SK: FRIEND#<friend_id>

┌────────────────┬──────────────────┬────────────┐
│ PK             │ SK               │ since      │
├────────────────┼──────────────────┼────────────┤
│ USER#alice     │ FRIEND#bob       │ 2024-01-15 │
│ USER#alice     │ FRIEND#charlie   │ 2024-02-20 │
│ USER#bob       │ FRIEND#alice     │ 2024-01-15 │
└────────────────┴──────────────────┴────────────┘

Access patterns:
  List alice's friends:     Query PK='USER#alice', SK begins_with('FRIEND#')
  Check if friends:         GetItem PK='USER#alice', SK='FRIEND#bob'
  Add friend (bidirectional): TransactWriteItems to insert both directions

⚠️ Friends-of-friends is HARD in DynamoDB (requires two queries + app-level join).
   For deep traversals, use Neo4j.
```

---

## Screen 5: Rapid Fire — 20 Quick Q&A

| # | Question | Answer |
|:--|:---------|:-------|
| 1 | What does `EXPLAIN ANALYZE` do that `EXPLAIN` doesn't? | Actually runs the query, showing real timing and row counts. |
| 2 | What's the default PostgreSQL index type? | B-tree. |
| 3 | Can you use a window function in a WHERE clause? | No — wrap in a CTE or subquery. |
| 4 | What's the difference between `DELETE` and `TRUNCATE`? | DELETE is row-by-row (logged, triggers fire, MVCC). TRUNCATE drops all rows instantly (minimal logging, no triggers). |
| 5 | What does `VACUUM FULL` do that regular `VACUUM` doesn't? | Rewrites the table, returning disk space to the OS. Requires exclusive lock. |
| 6 | What is a covering index? | An index that includes all columns a query needs, enabling an Index Only Scan. |
| 7 | What's the leftmost prefix rule? | A composite index (A,B,C) only helps queries that filter on A, or A+B, or A+B+C — never B or C alone. |
| 8 | What does `COALESCE(x, 0)` do? | Returns `x` if non-NULL, otherwise returns `0`. |
| 9 | How does PostgreSQL handle concurrent writes to the same row? | Row-level locking. Second writer waits until the first commits or rolls back. |
| 10 | What's the difference between `UNION` and `UNION ALL`? | UNION deduplicates (sorts). UNION ALL keeps all rows (faster). |
| 11 | What is a deadlock? | Two transactions each hold a lock the other needs. PostgreSQL detects and kills one. |
| 12 | What's the DynamoDB partition key design rule? | Choose high-cardinality keys for even distribution. Avoid hot partitions. |
| 13 | What's `jsonb_set()` used for? | Updating a specific key within a JSONB column without replacing the entire value. |
| 14 | What's PgBouncer? | A connection pooler that multiplexes many app connections onto few PostgreSQL backends. |
| 15 | What's the N+1 query problem? | Fetching a list (1 query), then querying details for each item (N queries). Fix: JOIN or batch fetch. |
| 16 | Redis `SET key value NX EX 30` — what does NX mean? | "Set only if Not eXists" — used for distributed locks. |
| 17 | What's a materialized view? | A cached query result stored as a physical table. Must be manually refreshed. |
| 18 | What does `ANALYZE` (without VACUUM) do? | Updates table statistics so the query planner makes better decisions. |
| 19 | When is a Seq Scan actually fine? | Small tables (<1K rows), or queries returning >10-15% of rows. |
| 20 | What's the CAP theorem tradeoff for PostgreSQL? | CP — it prioritizes Consistency over Availability during network partitions. |

---

## Screen 6: Tricky Gotchas — Common Mistakes and Edge Cases

### Gotcha 1: NULL in NOT IN

```sql
-- Find customers with NO orders
SELECT * FROM customers WHERE customer_id NOT IN (SELECT customer_id FROM orders);

-- If orders.customer_id has ANY NULL value, this returns ZERO rows!
-- Because: x NOT IN (1, 2, NULL) → x!=1 AND x!=2 AND x!=NULL → UNKNOWN → no match

-- Fix: use NOT EXISTS
SELECT * FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id);
```

### Gotcha 2: Integer Division

```sql
SELECT 5 / 2;           -- Returns 2 (integer division!)
SELECT 5 / 2.0;         -- Returns 2.5000
SELECT 5::numeric / 2;  -- Returns 2.5000
SELECT CAST(5 AS DECIMAL) / 2;  -- Returns 2.5000
```

Always cast to decimal/numeric before dividing for percentage calculations.

### Gotcha 3: GROUP BY with Non-Aggregated Columns

```sql
-- MySQL (with ONLY_FULL_GROUP_BY off): silently picks random values!
SELECT customer_id, name, SUM(amount) FROM orders GROUP BY customer_id;
-- Which "name" does it return? Undefined behavior!

-- PostgreSQL: correctly throws an error
-- Fix: include in GROUP BY or use an aggregate
SELECT customer_id, MAX(name), SUM(amount) FROM orders GROUP BY customer_id;
```

### Gotcha 4: BETWEEN is Inclusive

```sql
-- This includes both boundary dates!
WHERE order_date BETWEEN '2024-01-01' AND '2024-01-31'
-- Equivalent to: order_date >= '2024-01-01' AND order_date <= '2024-01-31'

-- For timestamps, this misses everything after midnight on Jan 31:
WHERE order_timestamp BETWEEN '2024-01-01' AND '2024-01-31'
-- Misses: 2024-01-31 14:30:00!

-- Fix: use half-open intervals
WHERE order_timestamp >= '2024-01-01' AND order_timestamp < '2024-02-01'
```

### Gotcha 5: Implicit Type Casting Kills Indexes

```sql
-- Table: users (phone_number TEXT, indexed)
CREATE INDEX idx_phone ON users (phone_number);

-- This CANNOT use the index:
SELECT * FROM users WHERE phone_number = 5551234567;
-- PostgreSQL casts the column (TEXT) to match the integer → function on column → no index

-- Fix: use the correct type
SELECT * FROM users WHERE phone_number = '5551234567';
```

### Gotcha 6: LIMIT Without ORDER BY is Non-Deterministic

```sql
-- Which 10 rows do you get? Different every time!
SELECT * FROM orders LIMIT 10;

-- Always pair LIMIT with ORDER BY:
SELECT * FROM orders ORDER BY order_id LIMIT 10;
```

### Gotcha 7: COUNT(*) vs COUNT(column)

```sql
-- COUNT(*): counts ALL rows (including NULLs)
-- COUNT(column): counts non-NULL values only

SELECT COUNT(*), COUNT(discount_code) FROM orders;
-- If 1000 orders and 200 have discount codes:
-- COUNT(*) = 1000, COUNT(discount_code) = 200
```

### Gotcha 8: Window Function Default Frame with Running Totals

```sql
-- This LOOKS right but has a subtle bug with duplicate dates:
SELECT order_date, amount,
       SUM(amount) OVER (ORDER BY order_date) AS running_total
FROM orders;

-- Default frame: RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
-- If two orders share the same date, RANGE groups them as peers
-- Both rows show the sum INCLUDING the other — the running total "jumps"

-- Fix: use ROWS instead of the default RANGE
SELECT order_date, amount,
       SUM(amount) OVER (ORDER BY order_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
FROM orders;
```

### Gotcha 9: UPDATE with JOIN — Accidental Cartesian

```sql
-- If the JOIN produces multiple matches, the UPDATE is non-deterministic
UPDATE orders o
SET discount = p.default_discount
FROM promotions p
WHERE o.category = p.category;

-- If promotions has two rows for 'Electronics', which discount is applied?
-- PostgreSQL picks one arbitrarily. You won't get an error.

-- Fix: ensure the join is unique, or use a subquery with explicit handling
UPDATE orders o
SET discount = (
    SELECT MAX(p.default_discount) FROM promotions p WHERE p.category = o.category
);
```

### Gotcha 10: Forgetting Transaction Isolation Side Effects

```sql
-- READ COMMITTED (default): a long-running query sees committed changes mid-query
-- This means a report scanning millions of rows can see INCONSISTENT data
-- (rows inserted/updated during the scan)

-- Fix for reports: use REPEATABLE READ or a snapshot
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT ... (long report query) ...;
COMMIT;
-- Now the query sees a consistent snapshot from the transaction start time
```

---

## Key Takeaways

- **Schema design questions test your ability to think about scale, consistency, and access patterns.** Always mention partitioning for large tables, JSONB for flexible attributes, appropriate constraints, and your indexing strategy.

- **Query writing challenges test window functions, CTEs, and set operations.** The sessionization pattern (LAG → flag → running SUM) and the top-N-per-group pattern (RANK + CTE) appear in ~60% of SQL interviews.

- **Performance debugging is about reading EXPLAIN ANALYZE.** Look for Seq Scans on large tables, estimated-vs-actual row mismatches, hash joins spilling to disk, and excessive heap fetches. Every problem maps to a specific fix: add index, update stats, increase work_mem, batch updates, or use covering indexes.

- **NoSQL design questions test whether you understand access-pattern-first design.** DynamoDB keys determine everything — design from queries backward. Redis data structure selection is critical — sorted sets for rankings, streams for reliable messaging, hashes for objects.

- **The rapid-fire section covers fundamentals that trip people up.** Know the difference between DELETE and TRUNCATE, UNION and UNION ALL, COUNT(*) and COUNT(column). These are table-stakes knowledge.

- **Gotchas reveal your real-world experience.** NULL in NOT IN, integer division, BETWEEN with timestamps, implicit type casts, and window function default frames — knowing these separates senior engineers from intermediates. The fix is always the same: understand what the database actually does, not what you assume it does.
