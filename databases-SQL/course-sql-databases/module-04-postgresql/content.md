# Module 4: PostgreSQL Internals & Power Features

> **Goal**: Go beyond basic SQL and master the PostgreSQL-specific features that separate production engineers from tutorial followers. You'll learn JSONB for flexible schemas, table partitioning for scale, VACUUM for maintenance, connection pooling for concurrency, and the diagnostic tools that keep billion-row databases humming.

---

## Screen 1: JSONB — The Best of Both Worlds

### Why JSONB Exists

Relational schemas are great until they're not. Product attributes vary wildly — a laptop has RAM and screen size, a shirt has fabric and fit. Creating 200 nullable columns is ugly. A separate `attributes` table with EAV (Entity-Attribute-Value) is a query nightmare. JSONB gives you **schema flexibility inside a relational database**.

```sql
CREATE TABLE products (
    product_id   SERIAL PRIMARY KEY,
    name         TEXT NOT NULL,
    category     TEXT NOT NULL,
    price        DECIMAL(10,2) NOT NULL,
    attributes   JSONB DEFAULT '{}'
);

INSERT INTO products (name, category, price, attributes) VALUES
('MacBook Pro 16"', 'Electronics', 2499.99,
 '{"ram_gb": 32, "storage_tb": 1, "color": "Space Gray", "ports": ["USB-C", "HDMI", "MagSafe"]}'),
('Wool Sweater', 'Clothing', 89.99,
 '{"size": "L", "color": "Navy", "material": "Merino Wool", "care": ["hand wash", "lay flat to dry"]}'),
('Standing Desk', 'Furniture', 599.00,
 '{"height_min_in": 25, "height_max_in": 50, "motor": "dual", "weight_capacity_lb": 300}');
```

### JSON vs JSONB — Always Use JSONB

| Feature           | `JSON`                   | `JSONB`                        |
|:------------------|:-------------------------|:-------------------------------|
| Storage           | Raw text (preserves ws)  | Binary decomposed format       |
| Duplicate keys    | Preserved                | Last value wins                |
| Key order         | Preserved                | Not guaranteed                 |
| Indexing (GIN)    | ❌ Not supported          | ✅ Full GIN support            |
| Comparison ops    | ❌ No equality            | ✅ `=`, `@>`, `<@`, `?`, `?&`  |
| Insert speed      | Slightly faster          | Slightly slower (parse cost)   |
| Read/query speed  | Slower (reparse each time)| Faster (pre-parsed)           |

**Rule**: Always use `JSONB` unless you specifically need to preserve key order or whitespace (almost never).

### JSONB Operators — Your Navigation Toolkit

```sql
-- -> returns JSONB (keeps the JSON type)
SELECT attributes->'color' FROM products;
-- Result: "Space Gray"  (with quotes — it's still JSON)

-- ->> returns TEXT (extracts as plain string)
SELECT attributes->>'color' FROM products;
-- Result: Space Gray  (no quotes — it's text now)

-- Nested navigation: chain operators
-- {"specs": {"display": {"size": 16, "resolution": "3456x2234"}}}
SELECT attributes->'specs'->'display'->>'resolution' FROM products;

-- #> and #>> navigate by path array
SELECT attributes #>> '{specs,display,resolution}' FROM products;
```

### Containment and Existence Operators

```sql
-- @> "contains" — does the JSONB contain this sub-document?
SELECT * FROM products
WHERE attributes @> '{"color": "Navy"}';
-- Finds the sweater — GIN indexable!

-- <@ "is contained by" — reverse of @>
SELECT * FROM products
WHERE '{"color": "Navy", "size": "L", "extra": true}' @> attributes;

-- ? "key exists" — does the top-level key exist?
SELECT * FROM products WHERE attributes ? 'motor';
-- Finds the standing desk

-- ?& "all keys exist"
SELECT * FROM products WHERE attributes ?& array['color', 'size'];

-- ?| "any key exists"
SELECT * FROM products WHERE attributes ?| array['motor', 'ram_gb'];
```

### Indexing JSONB with GIN

```sql
-- Index ALL keys and values (most flexible)
CREATE INDEX idx_products_attrs ON products USING GIN (attributes);

-- Now these queries are indexed:
WHERE attributes @> '{"color": "Navy"}'     -- ✅ containment
WHERE attributes ? 'motor'                   -- ✅ key existence
WHERE attributes ?& array['color', 'size']   -- ✅ all keys exist

-- For specific path queries, use a B-tree expression index (smaller, faster):
CREATE INDEX idx_products_color ON products ((attributes->>'color'));

-- Now this is indexed with B-tree:
WHERE attributes->>'color' = 'Navy'          -- ✅ equality
WHERE attributes->>'color' LIKE 'N%'         -- ✅ prefix search

-- For numeric range queries on a JSONB field:
CREATE INDEX idx_products_price ON products (((attributes->>'ram_gb')::int));
WHERE (attributes->>'ram_gb')::int >= 16     -- ✅ range scan
```

### jsonb_array_elements — Unnesting Arrays

```sql
-- Explode the "ports" array into rows
SELECT p.name, port.value AS port
FROM products p,
     jsonb_array_elements_text(p.attributes->'ports') AS port(value)
WHERE p.category = 'Electronics';
```

| name             | port    |
|:-----------------|:--------|
| MacBook Pro 16"  | USB-C   |
| MacBook Pro 16"  | HDMI    |
| MacBook Pro 16"  | MagSafe |

```sql
-- Aggregate back: find products that support USB-C
SELECT p.name
FROM products p
WHERE EXISTS (
    SELECT 1
    FROM jsonb_array_elements_text(p.attributes->'ports') AS port
    WHERE port = 'USB-C'
);

-- Or with the containment operator (simpler + GIN-indexed):
SELECT name FROM products
WHERE attributes @> '{"ports": ["USB-C"]}';
```

### Updating JSONB

```sql
-- Set a single key (creates or overwrites)
UPDATE products
SET attributes = jsonb_set(attributes, '{color}', '"Midnight"')
WHERE product_id = 1;

-- Add a nested key
UPDATE products
SET attributes = jsonb_set(attributes, '{specs,weight_kg}', '2.14')
WHERE product_id = 1;

-- Remove a key
UPDATE products
SET attributes = attributes - 'care'
WHERE product_id = 2;

-- Append to an array
UPDATE products
SET attributes = jsonb_set(
    attributes,
    '{ports}',
    (attributes->'ports') || '"SD Card"'::jsonb
)
WHERE product_id = 1;

-- Merge entire objects (PostgreSQL 16+: || operator)
UPDATE products
SET attributes = attributes || '{"warranty_years": 2, "refurbished": false}'::jsonb
WHERE product_id = 1;
```

### 💡 Interview Insight

> **"When would you use JSONB columns vs a normalized schema?"**
>
> *"I'd use JSONB for truly variable attributes where different rows have fundamentally different fields — like product specifications across categories or user preferences. The key test: if I'd need dozens of sparse nullable columns or an EAV table, JSONB is cleaner. I'd keep high-cardinality, frequently-queried, or JOIN-key columns as regular columns (customer_id, order_date, price) and put the 'long tail' of attributes in JSONB. I'd always index the JSONB column with GIN for containment queries, and add expression indexes on specific paths I query heavily. The anti-pattern is dumping everything into JSONB — you lose type safety, foreign key constraints, and query optimizer insight."*

---

## Screen 2: Table Partitioning — Declarative Partitioning & Partition Pruning

### When to Partition

Partitioning shines when a table has **hundreds of millions of rows** and queries predictably filter on a specific column — usually a date.

```
Before partitioning:
┌─────────────────────────────────────────┐
│            orders (500M rows)            │
│  Query: WHERE order_date = '2024-03-15' │
│  → Full table scan: 500M rows examined  │
└─────────────────────────────────────────┘

After partitioning (monthly):
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│ orders   │ │ orders   │ │ orders   │ │ orders   │ ...
│ 2024_01  │ │ 2024_02  │ │ 2024_03  │ │ 2024_04  │
│ ~10M rows│ │ ~10M rows│ │ ~10M rows│ │ ~10M rows│
│ PRUNED ✗ │ │ PRUNED ✗ │ │ SCANNED ✓│ │ PRUNED ✗ │
└──────────┘ └──────────┘ └──────────┘ └──────────┘
             Only 10M rows examined! 50x faster.
```

### Declarative Partitioning (PostgreSQL 10+)

```sql
-- Step 1: Create the partitioned parent table
CREATE TABLE orders (
    order_id     BIGINT NOT NULL,
    customer_id  INT NOT NULL,
    order_date   DATE NOT NULL,
    amount       DECIMAL(10,2),
    status       TEXT
) PARTITION BY RANGE (order_date);

-- Step 2: Create partition children
CREATE TABLE orders_2024_q1 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');
CREATE TABLE orders_2024_q2 PARTITION OF orders
    FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');
CREATE TABLE orders_2024_q3 PARTITION OF orders
    FOR VALUES FROM ('2024-07-01') TO ('2024-10-01');
CREATE TABLE orders_2024_q4 PARTITION OF orders
    FOR VALUES FROM ('2024-10-01') TO ('2025-01-01');

-- Step 3: Create a DEFAULT partition (catches rows that don't fit anywhere)
CREATE TABLE orders_default PARTITION OF orders DEFAULT;

-- Step 4: Create indexes on the parent (automatically created on all children)
CREATE INDEX idx_orders_date ON orders (order_date);
CREATE INDEX idx_orders_customer ON orders (customer_id);
```

### Partition Pruning — Proof It Works

```sql
EXPLAIN (ANALYZE, COSTS OFF)
SELECT SUM(amount) FROM orders
WHERE order_date BETWEEN '2024-04-01' AND '2024-06-30';

-- Output:
-- Aggregate
--   ->  Append
--         Subplans Removed: 3                ← 3 partitions pruned!
--         ->  Seq Scan on orders_2024_q2     ← only Q2 scanned
--               Filter: (order_date >= '2024-04-01' AND order_date <= '2024-06-30')
```

### Dynamic vs Static Pruning

**Static pruning**: The planner eliminates partitions at plan time (literal values in WHERE).

**Dynamic pruning** (PostgreSQL 11+): Pruning at execution time — works with parameterized queries and joins.

```sql
-- Dynamic pruning with a subquery
SELECT * FROM orders
WHERE order_date = (SELECT MAX(order_date) FROM recent_activity WHERE user_id = 42);
-- The partition is determined at runtime, not plan time
```

### Automated Partition Management

```sql
-- Create a function to auto-create monthly partitions
CREATE OR REPLACE FUNCTION create_monthly_partition(target_date DATE)
RETURNS void AS $$
DECLARE
    partition_name TEXT;
    start_date DATE;
    end_date DATE;
BEGIN
    start_date := DATE_TRUNC('month', target_date);
    end_date   := start_date + INTERVAL '1 month';
    partition_name := 'orders_' || TO_CHAR(start_date, 'YYYY_MM');

    EXECUTE format(
        'CREATE TABLE IF NOT EXISTS %I PARTITION OF orders
         FOR VALUES FROM (%L) TO (%L)',
        partition_name, start_date, end_date
    );
END;
$$ LANGUAGE plpgsql;

-- Call ahead of time: create next month's partition
SELECT create_monthly_partition(CURRENT_DATE + INTERVAL '1 month');
```

### 💡 Interview Insight

> **"How do you handle data retention with partitioned tables?"**
>
> *"This is one of partitioning's killer features. To drop old data, I simply detach and drop the partition — `ALTER TABLE orders DETACH PARTITION orders_2022_q1; DROP TABLE orders_2022_q1;` — which is instantaneous regardless of row count. Compare that to `DELETE FROM orders WHERE order_date < '2022-04-01'` which would generate millions of dead tuples, bloat the table, thrash the WAL, and take hours. I'd schedule a cron job to create new partitions ahead of time and detach expired ones on a retention schedule."*

---

## Screen 3: VACUUM and AUTOVACUUM — Taming Dead Tuples

### Why VACUUM Exists: MVCC and Dead Tuples

PostgreSQL uses **MVCC** (Multi-Version Concurrency Control). When you UPDATE or DELETE a row, PostgreSQL doesn't modify the row in-place — it marks the old version as "dead" and creates a new version. This means readers never block writers and vice versa.

The downside: dead tuples pile up.

```
UPDATE orders SET status = 'shipped' WHERE order_id = 1;

What actually happens in the heap:
┌──────────────────────────────────────────────────┐
│  Page 42                                          │
│  ┌──────────────────────────────────────────────┐ │
│  │ order_id=1, status='pending'  [xmax=100] DEAD│ │  ← old version, invisible
│  │ order_id=1, status='shipped'  [xmin=100] LIVE│ │  ← new version, visible
│  │ order_id=2, status='pending'  [            ] │ │
│  └──────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────┘
```

### What VACUUM Does

```
┌─────────────────────────────────────────────────────┐
│  VACUUM performs three critical tasks:                │
│                                                       │
│  1. RECLAIM SPACE: Marks dead tuples as reusable     │
│     (new inserts can use that space)                  │
│                                                       │
│  2. UPDATE VISIBILITY MAP: Tells the planner which   │
│     pages are "all-visible" (enables Index Only Scans)│
│                                                       │
│  3. FREEZE OLD TUPLES: Prevents transaction ID        │
│     wraparound (PostgreSQL's XID is 32-bit = ~4B)    │
└─────────────────────────────────────────────────────┘
```

```sql
-- Manual vacuum (reclaims space within existing pages)
VACUUM orders;

-- Verbose mode — see what it did
VACUUM VERBOSE orders;
-- INFO: "orders": removed 15432 dead row versions in 892 pages
-- INFO: "orders": found 15432 removable, 1234567 nonremovable row versions

-- VACUUM FULL — rewrites the entire table, reclaiming disk space
-- WARNING: Takes an exclusive lock! Use only as last resort.
VACUUM FULL orders;

-- VACUUM ANALYZE — vacuum + update planner statistics in one shot
VACUUM ANALYZE orders;
```

### VACUUM vs VACUUM FULL

| Feature          | `VACUUM`                    | `VACUUM FULL`               |
|:-----------------|:----------------------------|:----------------------------|
| Lock level       | Does NOT block reads/writes | **Exclusive lock** (blocks) |
| Space reclaimed  | Within existing pages       | Returns space to OS         |
| Table rewritten  | No                          | Yes (full rewrite)          |
| Speed            | Fast (incremental)          | Slow (entire table)         |
| When to use      | Regularly / autovacuum      | Extreme bloat only          |

### Monitoring Bloat

```sql
-- Check dead tuple count per table
SELECT
    schemaname, relname,
    n_live_tup,
    n_dead_tup,
    ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct,
    last_vacuum,
    last_autovacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 10;
```

### AUTOVACUUM Tuning

Autovacuum runs automatically but may need tuning for high-write tables.

```sql
-- Default trigger: vacuum when dead tuples > threshold + scale_factor × table_size
-- Default: threshold = 50, scale_factor = 0.2
-- For a 10M row table: triggers at 50 + 0.2 × 10M = 2,000,050 dead tuples
-- That's 20% bloat before vacuum runs!

-- Per-table tuning for a hot table:
ALTER TABLE orders SET (
    autovacuum_vacuum_threshold = 1000,
    autovacuum_vacuum_scale_factor = 0.01,     -- 1% instead of 20%
    autovacuum_analyze_threshold = 500,
    autovacuum_analyze_scale_factor = 0.005
);

-- Increase autovacuum workers for busy databases
-- In postgresql.conf:
-- autovacuum_max_workers = 5        (default: 3)
-- autovacuum_vacuum_cost_limit = 400 (default: 200, higher = faster vacuum)
```

### 💡 Interview Insight

> **"What is table bloat and how do you fix it?"**
>
> *"Bloat happens when dead tuples accumulate faster than VACUUM can reclaim them, or when autovacuum is too conservative. The table file grows but most pages are half-empty. I'd diagnose it by comparing `pg_relation_size` to the expected size based on live row count, or by checking `n_dead_tup` in `pg_stat_user_tables`. For mild bloat, I tune autovacuum to run more aggressively (lower scale_factor). For severe bloat, `pg_repack` can rebuild the table online without an exclusive lock — unlike VACUUM FULL which blocks all access. Prevention is key: tune autovacuum per-table for high-write workloads."*

---

## Screen 4: Connection Pooling with PgBouncer

### The Problem: PostgreSQL's Process Model

PostgreSQL spawns a **dedicated OS process** per connection. Each process uses ~5–10 MB of RAM. With 500 connections, that's 2.5–5 GB just for connections — before any query work.

```
Without pooling:
┌─────────────────┐         ┌──────────────────────┐
│  Application    │         │  PostgreSQL           │
│  (500 threads)  │────────→│  500 backend processes│
│                 │         │  ~5 GB RAM overhead   │
└─────────────────┘         └──────────────────────┘

With PgBouncer:
┌─────────────────┐    ┌──────────┐    ┌──────────────────────┐
│  Application    │    │PgBouncer │    │  PostgreSQL           │
│  (500 threads)  │───→│ (pool)   │───→│  20 backend processes │
│                 │    │          │    │  ~200 MB RAM overhead │
└─────────────────┘    └──────────┘    └──────────────────────┘
```

### PgBouncer Pool Modes

```
┌─────────────────────────────────────────────────────────────────┐
│  SESSION MODE (default)                                          │
│  Connection assigned to client for entire session lifetime       │
│  Client connects → gets a backend → keeps it until disconnect    │
│  Pooling benefit: minimal (only helps with connect/disconnect)   │
│                                                                   │
│  TRANSACTION MODE (recommended for most apps)                    │
│  Connection assigned per transaction only                        │
│  BEGIN → assign backend → COMMIT/ROLLBACK → return to pool       │
│  Pooling benefit: HIGH (backends shared between transactions)    │
│  ⚠️ Restrictions: no session-level state (SET, prepared stmts)   │
│                                                                   │
│  STATEMENT MODE (aggressive)                                     │
│  Connection returned after every single statement                │
│  Pooling benefit: MAXIMUM                                        │
│  ⚠️ Restrictions: no multi-statement transactions allowed        │
└─────────────────────────────────────────────────────────────────┘
```

| Mode          | Multiplexing | Session State | Transactions | Use Case              |
|:--------------|:-------------|:--------------|:-------------|:----------------------|
| Session       | Low          | ✅ Full        | ✅ Multi-stmt | Legacy apps           |
| Transaction   | High         | ❌ Per-txn only | ✅ Multi-stmt | **Most web apps**     |
| Statement     | Maximum      | ❌ None        | ❌ Single only | Simple read queries   |

### PgBouncer Configuration

```ini
; pgbouncer.ini
[databases]
mydb = host=127.0.0.1 port=5432 dbname=ecommerce

[pgbouncer]
listen_port = 6432
listen_addr = 0.0.0.0
pool_mode = transaction
max_client_conn = 1000       ; max clients connecting to PgBouncer
default_pool_size = 20       ; backends per database/user pair
min_pool_size = 5            ; keep 5 backends warm
reserve_pool_size = 5        ; extra backends for burst traffic
reserve_pool_timeout = 3     ; wait 3s before using reserve pool
server_idle_timeout = 300    ; close idle backends after 5 min
query_wait_timeout = 120     ; max time a client waits for a backend
```

### Monitoring PgBouncer

```sql
-- Connect to PgBouncer's admin console
psql -p 6432 -U admin pgbouncer

-- Show pool status
SHOW POOLS;
-- database | user  | cl_active | cl_waiting | sv_active | sv_idle | pool_mode
-- ecommerce| app   | 150       | 0          | 12        | 8       | transaction

-- Key metrics:
-- cl_waiting > 0 → clients waiting for backends (increase pool_size!)
-- sv_active always maxed → pool is saturated
-- sv_idle high → pool is oversized (reduce to save resources)

SHOW STATS;   -- query count, bytes, timing
SHOW SERVERS; -- backend connection details
SHOW CLIENTS; -- client connection details
```

### 💡 Interview Insight

> **"How would you handle 10,000 concurrent connections to PostgreSQL?"**
>
> *"Direct connections would be catastrophic — PostgreSQL would spawn 10K processes and OOM. I'd put PgBouncer in front in transaction mode with a pool size of maybe 50–100 backends. Since most web requests hold a connection for milliseconds (one transaction), 50 backends can serve 10K clients because they time-share. I'd set max_client_conn to 10,000 and monitor cl_waiting. If clients start queuing, I'd increase pool_size — but cautiously, because too many backends cause CPU context-switching overhead. For very high scale, I'd also consider read replicas behind a separate pool to offload read traffic."*

---

## Screen 5: pg_stat_statements — Finding Slow Queries

### Enable It

```sql
-- In postgresql.conf:
-- shared_preload_libraries = 'pg_stat_statements'
-- pg_stat_statements.track = all

-- Then in your database:
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

### The Query Performance Dashboard

```sql
-- Top 10 queries by total execution time
SELECT
    LEFT(query, 80) AS query_preview,
    calls,
    ROUND(total_exec_time::numeric, 2) AS total_time_ms,
    ROUND(mean_exec_time::numeric, 2) AS avg_time_ms,
    ROUND(stddev_exec_time::numeric, 2) AS stddev_ms,
    rows,
    ROUND((100.0 * shared_blks_hit / NULLIF(shared_blks_hit + shared_blks_read, 0))::numeric, 2)
        AS cache_hit_pct
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

```
┌──────────────────────────────────────────────────────────────────────────┐
│  query_preview                    │ calls  │ total_ms │ avg_ms │ cache% │
├───────────────────────────────────┼────────┼──────────┼────────┼────────┤
│ SELECT * FROM orders WHERE cust.. │ 892431 │ 345621   │ 0.39   │ 99.8   │
│ SELECT p.*, c.name FROM product.. │ 12455  │ 298334   │ 23.95  │ 67.2   │  ← slow!
│ UPDATE orders SET status = $1 W.. │ 45221  │ 156000   │ 3.45   │ 95.1   │
│ INSERT INTO order_items (order_.. │ 234111 │ 89000    │ 0.38   │ 99.9   │
└──────────────────────────────────────────────────────────────────────────┘
```

### What to Optimize

The second query stands out: only 12K calls but 298 seconds total — 24ms average with a 67% cache hit rate. That's a **missing index** or a **seq scan on a cold table**.

```sql
-- Get the full query text
SELECT query, calls, mean_exec_time, shared_blks_read
FROM pg_stat_statements
WHERE query LIKE '%product%'
ORDER BY mean_exec_time DESC;

-- Reset stats after optimization to measure improvement
SELECT pg_stat_statements_reset();
```

### Combining with pg_stat_user_tables

```sql
-- Find tables with sequential scan problems
SELECT
    schemaname, relname,
    seq_scan,
    idx_scan,
    ROUND(100.0 * idx_scan / NULLIF(seq_scan + idx_scan, 0), 1) AS idx_scan_pct,
    n_live_tup
FROM pg_stat_user_tables
WHERE n_live_tup > 10000
ORDER BY seq_scan DESC
LIMIT 10;

-- Tables with high seq_scan and low idx_scan_pct need indexes!
```

### 💡 Interview Insight

> **"How do you find and fix slow queries in a production PostgreSQL database?"**
>
> *"Step 1: Enable `pg_stat_statements` and query it sorted by total_exec_time — this shows which queries consume the most cumulative time (high calls × avg time). Step 2: For each top offender, run EXPLAIN ANALYZE to get the execution plan. Step 3: Check `pg_stat_user_tables` for tables with high seq_scan counts — these likely need indexes. Step 4: Look at cache_hit_pct — if it's low, the working set exceeds shared_buffers and I'd either add memory or optimize the query to read less data. I focus on total time, not just average, because a 1ms query called 10M times is worse than a 10s query called once."*

---

## Screen 6: Locking — Advisory Locks, Row-Level Locks, and Deadlocks

### Row-Level Locking with SELECT FOR UPDATE

```sql
-- Scenario: two checkout processes grab the same inventory item

-- Process A:
BEGIN;
SELECT quantity FROM inventory WHERE product_id = 42 FOR UPDATE;
-- Locks row → Process B will WAIT here if it tries the same row
UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 42;
COMMIT;

-- FOR UPDATE variants:
SELECT ... FOR UPDATE;           -- Exclusive lock, wait if locked
SELECT ... FOR UPDATE NOWAIT;    -- Error immediately if locked
SELECT ... FOR UPDATE SKIP LOCKED; -- Skip locked rows (great for job queues!)
SELECT ... FOR SHARE;            -- Shared lock (allow other readers)
```

### SKIP LOCKED — Building a Job Queue

```sql
-- Worker process grabs the next available job
BEGIN;
SELECT job_id, payload
FROM job_queue
WHERE status = 'pending'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;
-- If another worker already locked a job, this skips it and grabs the next one!

UPDATE job_queue SET status = 'processing', worker_id = 'worker-3'
WHERE job_id = <grabbed_job_id>;
COMMIT;
```

### Advisory Locks — Application-Level Coordination

Advisory locks are **voluntary** — PostgreSQL doesn't enforce them automatically. Your application decides when to acquire and check them.

```sql
-- Session-level advisory lock (held until session ends or explicitly released)
SELECT pg_advisory_lock(12345);        -- Acquire (blocks if held by another session)
-- ... do critical work ...
SELECT pg_advisory_unlock(12345);      -- Release

-- Transaction-level advisory lock (auto-released on COMMIT/ROLLBACK)
SELECT pg_advisory_xact_lock(12345);

-- Try without waiting
SELECT pg_try_advisory_lock(12345);    -- Returns true/false, never blocks

-- Common pattern: prevent duplicate cron job execution
SELECT pg_try_advisory_lock(hashtext('nightly_report'));
-- Returns false if another instance is already running the report
```

### Deadlock Detection

```
Deadlock scenario:
  Transaction A: locks row 1, then tries to lock row 2
  Transaction B: locks row 2, then tries to lock row 1
  → Both wait forever! PostgreSQL detects this in ~1 second and kills one.

  Txn A                        Txn B
  ──────                       ──────
  BEGIN;                        BEGIN;
  UPDATE orders SET ...         UPDATE orders SET ...
    WHERE order_id = 1;           WHERE order_id = 2;
  -- holds lock on row 1       -- holds lock on row 2

  UPDATE orders SET ...         UPDATE orders SET ...
    WHERE order_id = 2;           WHERE order_id = 1;
  -- WAITS for Txn B...        -- WAITS for Txn A...

  ⚡ DEADLOCK DETECTED — PostgreSQL aborts one transaction
```

**Prevention strategies**:
1. **Always lock rows in a consistent order** (e.g., by ascending primary key)
2. **Keep transactions short** — acquire locks late, release early
3. **Use `NOWAIT` or `lock_timeout`** to fail fast instead of waiting indefinitely

```sql
SET lock_timeout = '5s';  -- Don't wait more than 5 seconds for any lock
```

### 💡 Interview Insight

> **"How would you prevent double-charging a customer in a concurrent checkout system?"**
>
> *"I'd use `SELECT ... FOR UPDATE` on the inventory row to acquire a row-level exclusive lock before decrementing stock. This serializes concurrent checkouts for the same product. For distributed systems with multiple app servers, I'd combine this with an advisory lock on a hash of the order ID to prevent duplicate order submission. I'd also set a `lock_timeout` to fail fast rather than queuing up requests. The key principle: pessimistic locking for inventory (critical consistency) and idempotency keys for payment processing."*

---

## Screen 7: COPY — Bulk Loading at Wire Speed

### INSERT vs COPY Performance

```
10 million rows:
  INSERT (row by row):      ~45 minutes
  INSERT (multi-value):     ~8 minutes
  COPY (binary pipe):       ~45 seconds
  COPY (with optimizations): ~15 seconds
```

### COPY Syntax

```sql
-- Load from a CSV file
COPY orders (order_id, customer_id, order_date, amount, status)
FROM '/data/exports/orders_2024.csv'
WITH (FORMAT csv, HEADER true, DELIMITER ',', NULL '');

-- Load from STDIN (piped data)
cat orders.csv | psql -c "COPY orders FROM STDIN WITH (FORMAT csv, HEADER true)"

-- Export to a file
COPY (SELECT * FROM orders WHERE order_date >= '2024-01-01')
TO '/tmp/recent_orders.csv'
WITH (FORMAT csv, HEADER true);

-- Binary format (fastest, but not human-readable)
COPY orders TO '/tmp/orders.bin' WITH (FORMAT binary);
COPY orders FROM '/tmp/orders.bin' WITH (FORMAT binary);
```

### Optimization Checklist for Bulk Loads

```sql
-- 1. Drop indexes before load, recreate after
DROP INDEX idx_orders_date;
DROP INDEX idx_orders_customer;

-- 2. Disable triggers if safe
ALTER TABLE orders DISABLE TRIGGER ALL;

-- 3. Increase maintenance_work_mem for faster index rebuild
SET maintenance_work_mem = '1GB';

-- 4. Load the data
COPY orders FROM '/data/orders_bulk.csv' WITH (FORMAT csv, HEADER true);

-- 5. Rebuild indexes (faster than incremental index maintenance)
CREATE INDEX idx_orders_date ON orders (order_date);
CREATE INDEX idx_orders_customer ON orders (customer_id);

-- 6. Re-enable triggers
ALTER TABLE orders ENABLE TRIGGER ALL;

-- 7. Update statistics
ANALYZE orders;
```

### \copy vs COPY

```
COPY  — Server-side command. File must be on the PostgreSQL server.
       Runs as the postgres OS user. Fastest.

\copy — psql client-side command. File is on the client machine.
       Streams data through the psql connection. Slightly slower.
       Syntax: \copy orders FROM 'local_file.csv' WITH (FORMAT csv)
```

### 💡 Interview Insight

> **"You need to load 100 million rows into PostgreSQL as fast as possible. What's your approach?"**
>
> *"I'd use COPY, not INSERT. Before loading: drop all secondary indexes and disable triggers to avoid per-row overhead. I'd increase `maintenance_work_mem` to 1–2 GB for faster post-load index rebuilds. I'd split the file into chunks and load in parallel using multiple COPY sessions if the table is partitioned (each session targets a different partition). After the load: rebuild indexes with CREATE INDEX, re-enable triggers, and run ANALYZE. If it's a one-time migration, I'd also set `wal_level = minimal` and `fsync = off` temporarily (with the understanding that a crash means reloading). This approach can achieve 500K+ rows/second."*

---

## Screen 8: Quiz

**Q1: Which JSONB operator extracts a value as plain TEXT (not JSON)?**
- A) `->`
- B) `->>`  ✅
- C) `@>`
- D) `#>`

> **Answer: B** — `->` returns a JSONB value (still JSON-typed, with quotes around strings). `->>` extracts the value as TEXT. `@>` is the containment operator. `#>` navigates a path and returns JSONB; its text equivalent is `#>>`.

**Q2: Autovacuum runs when dead tuples exceed `threshold + scale_factor × table_size`. For a 50M row table with default settings (threshold=50, scale_factor=0.2), approximately how many dead tuples trigger a vacuum?**
- A) 50
- B) 10,000
- C) 10,000,050  ✅
- D) 50,000,000

> **Answer: C** — With defaults: 50 + 0.2 × 50,000,000 = 10,000,050. That's 20% of the table becoming dead before autovacuum kicks in! This is why high-write tables need per-table tuning with a lower `scale_factor` (e.g., 0.01 for 1%).

**Q3: In PgBouncer transaction mode, which of these will NOT work correctly?**
- A) `BEGIN; SELECT ...; UPDATE ...; COMMIT;`
- B) `SET search_path TO myschema; SELECT ...;`  ✅
- C) `SELECT * FROM orders WHERE order_id = $1;`
- D) `INSERT INTO orders (...) RETURNING order_id;`

> **Answer: B** — In transaction mode, session-level state like `SET` commands, prepared statements, and `LISTEN/NOTIFY` don't persist because the backend connection may be different for the next transaction. The `SET` runs on one backend, but the next query may go to a different backend where `search_path` is still the default.

**Q4: When using `pg_stat_statements` to find problematic queries, which metric best identifies queries worth optimizing?**
- A) Maximum execution time per call
- B) Total execution time across all calls  ✅
- C) Number of rows returned
- D) Number of shared blocks read

> **Answer: B** — Total execution time (`total_exec_time`) captures the full impact: a query averaging 1ms but called 5 million times (5,000 seconds total) is a bigger optimization target than a 10-second query called once. Always sort by total time first, then investigate the average time and call count to decide the approach.

**Q5: What is the primary advantage of `COPY` over `INSERT` for bulk loading?**
- A) COPY supports transactions while INSERT does not
- B) COPY bypasses all constraints and indexes
- C) COPY uses a binary streaming protocol with minimal per-row overhead  ✅
- D) COPY automatically parallelizes across CPU cores

> **Answer: C** — COPY streams data in a compact binary protocol, avoiding the per-statement parsing, planning, and executor overhead of individual INSERTs. It does NOT bypass constraints (CHECK, NOT NULL, FK are still enforced). Indexes are still maintained during COPY (which is why dropping them beforehand helps). Parallelism requires manual setup with multiple sessions.

---

## Screen 9: Key Takeaways

- **JSONB gives you document flexibility inside PostgreSQL.** Use `->` for JSON navigation, `->>` for text extraction, `@>` for containment queries. Always index with GIN for containment or expression indexes for specific paths. Keep structured, high-query columns as regular columns — JSONB is for the "long tail."

- **Declarative partitioning enables partition pruning** — the planner skips entire partitions that can't match your WHERE clause. Use range partitioning for time-series data. Detaching + dropping partitions is the fastest way to purge old data. Always create a DEFAULT partition to catch stray rows.

- **VACUUM reclaims dead tuples from MVCC.** Regular VACUUM frees space for reuse within the table file. VACUUM FULL rewrites the table (exclusive lock — avoid in production). Autovacuum's default settings are too conservative for high-write tables — tune `scale_factor` down to 0.01–0.05 on hot tables.

- **PgBouncer in transaction mode** is essential for high-concurrency apps. It multiplexes hundreds of application connections onto a small pool of PostgreSQL backends. Remember: session-level state (SET, prepared statements) doesn't work in transaction mode.

- **pg_stat_statements is your production query profiler.** Sort by `total_exec_time` to find the highest-impact optimization targets. Combine with `pg_stat_user_tables` to find tables with too many sequential scans.

- **Row-level locking with `FOR UPDATE`** serializes concurrent access. `SKIP LOCKED` is a gem for building job queues. Advisory locks coordinate application-level logic. Prevent deadlocks by always locking in a consistent order.

- **COPY is 10–100x faster than INSERT for bulk loads.** Drop indexes before loading, rebuild after. Use `\copy` for client-side files, `COPY` for server-side files. Always run ANALYZE after a bulk load to update planner statistics.
