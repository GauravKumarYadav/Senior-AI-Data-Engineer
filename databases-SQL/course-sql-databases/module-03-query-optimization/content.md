# Module 3: Query Performance & Optimization

> **Goal**: Learn to read query plans like a mechanic reads engine diagnostics, understand index types and when to use each, and master the join algorithms and optimization techniques that turn a 30-second query into a 30-millisecond one. This is where senior-level SQL interviews live.

---

## Screen 1: EXPLAIN ANALYZE — Reading Query Plans

### Your X-Ray Vision

`EXPLAIN` shows you **what the database plans to do**. `EXPLAIN ANALYZE` actually **runs the query** and shows what it really did, with timing and row counts.

```sql
-- PostgreSQL syntax
EXPLAIN ANALYZE
SELECT c.name, COUNT(*) AS order_count, SUM(o.amount) AS total_spent
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_date >= '2024-01-01'
GROUP BY c.name
ORDER BY total_spent DESC
LIMIT 10;
```

### Reading the Output

```
Limit  (cost=1234.56..1234.60 rows=10 width=48) (actual time=12.3..12.4 rows=10 loops=1)
  ->  Sort  (cost=1234.56..1240.12 rows=2500 width=48) (actual time=12.3..12.3 rows=10 loops=1)
        Sort Key: sum(o.amount) DESC
        Sort Method: top-N heapsort  Memory: 26kB
        ->  HashAggregate  (cost=1100.00..1125.00 rows=2500 width=48) (actual time=11.1..11.8 rows=2500 loops=1)
              Group Key: c.name
              ->  Hash Join  (cost=45.00..950.00 rows=15000 width=20) (actual time=0.5..8.2 rows=15000 loops=1)
                    Hash Cond: (o.customer_id = c.customer_id)
                    ->  Seq Scan on orders o  (cost=0.00..800.00 rows=15000 width=16) (actual time=0.02..4.1 rows=15000 loops=1)
                          Filter: (order_date >= '2024-01-01')
                          Rows Removed by Filter: 35000
                    ->  Hash  (cost=30.00..30.00 rows=2500 width=20) (actual time=0.4..0.4 rows=2500 loops=1)
                          Buckets: 4096  Batches: 1  Memory Usage: 145kB
                          ->  Seq Scan on customers c  (cost=0.00..30.00 rows=2500 width=20) (actual time=0.01..0.2 rows=2500 loops=1)
```

### Decoding the Key Elements

```
Node Type  (cost=startup..total rows=estimated width=bytes) (actual time=start..end rows=actual loops=N)
```

| Element              | Meaning                                          |
|:---------------------|:-------------------------------------------------|
| `cost=startup..total`| Estimated cost in arbitrary units (lower = better)|
| `rows=`              | Estimated rows (EXPLAIN) or actual (ANALYZE)     |
| `actual time=`       | Real wall-clock time in milliseconds              |
| `loops=`             | How many times this node executed                 |
| `Rows Removed`       | Rows that didn't pass the filter                  |
| `width=`             | Estimated row size in bytes                       |

### Common Scan Types

```
┌─────────────────────────────────────────────────────────┐
│  Seq Scan          → Full table scan (reads every row)  │
│  Index Scan        → B-tree lookup + table fetch        │
│  Index Only Scan   → Reads only from index (no table!)  │
│  Bitmap Index Scan → Index → bitmap → table fetch       │
│  Bitmap Heap Scan  → Reads table pages from bitmap      │
└─────────────────────────────────────────────────────────┘
```

**When Seq Scan is FINE**: Small tables (<1000 rows), queries returning >10-15% of rows (index overhead not worth it), or no suitable index exists.

**Red flags**: Seq Scan on a large table with a selective WHERE clause, actual rows >> estimated rows (bad statistics), nested loops with high `loops=` count.

### The "Actual vs Estimated" Discrepancy

When estimated rows and actual rows diverge significantly, the optimizer made a **bad plan choice** based on stale or missing statistics. This is the #1 cause of slow queries.

```
-- Estimated: 100 rows.  Actual: 500,000 rows.  ← BAD STATS!
-- Fix: ANALYZE tablename; (updates statistics)
```

### 💡 Interview Insight

> **"How do you diagnose a slow SQL query?"**
>
> *"I start with EXPLAIN ANALYZE to get the actual execution plan. I look for three things: (1) Seq Scans on large tables with selective filters — these need indexes. (2) Big discrepancies between estimated and actual row counts — this means stale statistics, fixed with ANALYZE. (3) The most expensive node — I follow the cost numbers and actual time to find the bottleneck. Common culprits are missing indexes, bad join order, or correlated subqueries that should be rewritten as joins."*

---

## Screen 2: Index Types — B-tree, Hash, GIN, GiST

### B-tree — The Default Workhorse

B-tree (Balanced tree) indexes are the default in virtually every database. They support equality, range queries, sorting, and prefix matching.

```sql
CREATE INDEX idx_orders_date ON orders (order_date);

-- These ALL use the B-tree index:
WHERE order_date = '2024-01-15'          -- Equality
WHERE order_date > '2024-01-01'          -- Range
WHERE order_date BETWEEN '2024-01' AND '2024-06'  -- Range
ORDER BY order_date                      -- Sorting
WHERE order_date IS NULL                 -- NULL lookup
```

```
B-tree Structure (simplified):
                    ┌──────────┐
                    │ Jan, Jul  │           ← Root
                    └─┬──────┬─┘
              ┌──────┘        └──────┐
         ┌────┴────┐           ┌─────┴────┐
         │Jan, Mar  │           │Jul, Oct  │  ← Internal
         └─┬────┬──┘           └─┬────┬───┘
        ┌──┘    └──┐          ┌──┘    └──┐
     ┌──┴──┐  ┌──┴──┐    ┌──┴──┐   ┌──┴──┐
     │Jan  │  │Feb  │    │Jul  │   │Sep  │    ← Leaf (linked list)
     │Feb  │  │Mar  │    │Aug  │   │Oct  │
     └─────┘  └─────┘    └─────┘   └─────┘
```

B-trees have O(log n) lookup. For a 10M-row table, that's ~23 comparisons to find any value. The leaf pages form a linked list, making range scans efficient.

### Hash Index

Hash indexes only support **equality** lookups — no ranges, no sorting. O(1) lookup but limited use.

```sql
CREATE INDEX idx_customers_email_hash ON customers USING HASH (email);

-- ✅ Works:   WHERE email = 'user@example.com'
-- ❌ No help:  WHERE email LIKE 'user%'
-- ❌ No help:  ORDER BY email
-- ❌ No help:  WHERE email > 'a'
```

In PostgreSQL, hash indexes weren't crash-safe until v10 and aren't generally recommended — B-trees handle equality just as well and support more operations.

### GIN — Generalized Inverted Index

GIN indexes are designed for **composite values** — arrays, JSONB, and full-text search. They map each element/key to the rows containing it.

```sql
-- Full-text search
CREATE INDEX idx_products_search ON products USING GIN (to_tsvector('english', description));

SELECT * FROM products
WHERE to_tsvector('english', description) @@ to_tsquery('wireless & headphones');

-- JSONB containment
CREATE INDEX idx_orders_metadata ON orders USING GIN (metadata);

SELECT * FROM orders
WHERE metadata @> '{"source": "mobile", "campaign": "summer2024"}';

-- Array operations
CREATE INDEX idx_products_tags ON products USING GIN (tags);

SELECT * FROM products WHERE tags @> ARRAY['electronics', 'bluetooth'];
```

### GiST — Generalized Search Tree

GiST indexes support **spatial data**, range types, and nearest-neighbor searches.

```sql
-- PostGIS spatial queries
CREATE INDEX idx_stores_location ON stores USING GIST (location);

SELECT name, ST_Distance(location, ST_MakePoint(-73.98, 40.75)) AS distance
FROM stores
ORDER BY location <-> ST_MakePoint(-73.98, 40.75)  -- KNN search
LIMIT 5;

-- Range types
CREATE INDEX idx_events_timerange ON events USING GIST (time_range);

SELECT * FROM events
WHERE time_range && tsrange('2024-01-01', '2024-02-01');  -- Overlaps
```

### Quick Reference

| Index Type | Best For                    | Equality | Range | Sort | Pattern | Special          |
|:-----------|:----------------------------|:---------|:------|:-----|:--------|:-----------------|
| B-tree     | Most queries               | ✅        | ✅     | ✅    | Prefix  | Default          |
| Hash       | Exact match only           | ✅        | ❌     | ❌    | ❌       | Rarely useful    |
| GIN        | JSONB, arrays, full-text   | ✅        | ❌     | ❌    | ✅       | Multi-key values |
| GiST       | Spatial, ranges, KNN       | ✅        | ✅     | ❌    | ❌       | Geometric ops    |

### 💡 Interview Insight

> **"What index type would you use for a JSONB column that stores product attributes?"**
>
> *"GIN (Generalized Inverted Index). GIN indexes map each key-value pair in the JSONB document to the rows containing it, enabling efficient containment queries with the @> operator. For example, `WHERE attributes @> '{"color": "red", "size": "L"}'` would use the GIN index to find matching rows without scanning the entire table. If I only need to index specific paths, I could use a B-tree on an expression: `CREATE INDEX ON products ((attributes->>'color'))` — which is smaller and faster for single-key lookups."*

---

## Screen 3: Composite Indexes — Column Order Matters

### The Leftmost Prefix Rule

A composite (multi-column) index on `(a, b, c)` can satisfy queries on:
- `(a)` ✅
- `(a, b)` ✅
- `(a, b, c)` ✅
- `(b)` ❌ — Can't skip the leftmost column!
- `(a, c)` ⚠️ — Uses `a` only, scans `c` within those matches
- `(b, c)` ❌

Think of it like a phone book sorted by (last_name, first_name, city). You can look up by last name, or last+first, but you can't jump straight to first name.

```sql
CREATE INDEX idx_orders_composite ON orders (customer_id, order_date, status);

-- ✅ Uses full index:
WHERE customer_id = 100 AND order_date = '2024-01-15' AND status = 'shipped'

-- ✅ Uses first two columns:
WHERE customer_id = 100 AND order_date > '2024-01-01'

-- ✅ Uses first column only:
WHERE customer_id = 100

-- ❌ Cannot use this index:
WHERE order_date = '2024-01-15'
WHERE status = 'shipped'

-- ⚠️ Uses customer_id, then scans for status (skips order_date):
WHERE customer_id = 100 AND status = 'shipped'
```

### Column Order Strategy

**Rule**: Put columns in this order:
1. **Equality columns first** (exact match narrows results the most)
2. **Range/inequality columns next** (the index can only handle ONE range condition efficiently)
3. **Sort/order columns last**

```sql
-- Query pattern: WHERE status = 'active' AND created_at > '2024-01-01' ORDER BY amount DESC
-- Optimal index:
CREATE INDEX idx_optimal ON orders (status, created_at, amount);
--                                   ^equality  ^range      ^sort
```

### Why Column Order Matters — A Real Scenario

```sql
-- 10M orders, 5K customers, query: recent orders for a specific customer
-- Index A: (order_date, customer_id) → scans ALL orders for that date range, then filters customer
-- Index B: (customer_id, order_date) → jumps straight to customer, then scans their date range

-- With Index A: reads ~30K rows (all orders on those dates), filters to ~6
-- With Index B: reads ~6 rows directly
-- That's a 5000x difference!
```

### The Range Column Trap

An index can efficiently handle **at most one range condition**. After the first range predicate, subsequent columns in the index aren't used for seeking.

```sql
CREATE INDEX idx_orders ON orders (amount, order_date);

-- This can use both columns:
WHERE amount = 100 AND order_date > '2024-01-01'  -- amount is equality

-- This can only use amount's range:
WHERE amount > 50 AND order_date > '2024-01-01'   -- amount is range, order_date becomes filter
```

### 💡 Interview Insight

> **"How do you decide the column order for a composite index?"**
>
> *"I follow the equality-range-sort (ERS) principle. Equality predicates go first because they maximally narrow the search. Then the range or inequality column, because the B-tree can only efficiently handle one range scan. Finally, sort columns to enable index-ordered output and avoid a separate sort step. I also consider query frequency — if 80% of queries filter by customer_id first, it should be the leftmost column even if other orderings are slightly better for rare queries. And I always remember: a composite index on (A, B) does NOT help queries that only filter on B."*

---

## Screen 4: Covering Indexes and Index-Only Scans

### The Extra Trip Problem

A regular index lookup is a **two-step process**:

```
Step 1: Search the index → find row pointers (TIDs)
Step 2: Fetch actual rows from the table (heap) → get the columns you need
```

Step 2 is called a "heap fetch" or "table lookup" and it's often the expensive part — especially if the rows are scattered across many disk pages (random I/O).

### Covering Indexes to the Rescue

A **covering index** includes all columns the query needs, eliminating the heap fetch entirely. The query can be satisfied entirely from the index — an **Index Only Scan**.

```sql
-- Query: get order dates and amounts for a customer
SELECT order_date, amount
FROM orders
WHERE customer_id = 123;

-- Regular index: still needs heap fetch for order_date and amount
CREATE INDEX idx_orders_customer ON orders (customer_id);

-- Covering index: includes everything the query needs!
CREATE INDEX idx_orders_customer_covering ON orders (customer_id) 
    INCLUDE (order_date, amount);
```

### The INCLUDE Clause (PostgreSQL 11+, SQL Server 2005+)

`INCLUDE` adds non-key columns to the **leaf pages** of the index. These columns:
- ✅ Are available for Index Only Scans
- ❌ Are NOT part of the B-tree search key
- ❌ Can NOT be used for filtering or sorting

```sql
-- The key column (customer_id) is in the B-tree structure
-- The included columns (order_date, amount) are just stored alongside
CREATE INDEX idx_covering ON orders (customer_id) INCLUDE (order_date, amount);

-- Equivalent without INCLUDE (but larger index, and order_date is searchable):
CREATE INDEX idx_composite ON orders (customer_id, order_date, amount);
```

### When to Use INCLUDE vs Composite

| Feature                    | `(a, b, c)` composite  | `(a) INCLUDE (b, c)`  |
|:---------------------------|:------------------------|:-----------------------|
| Filter on b or c           | ✅ Yes                  | ❌ No                  |
| Sort on b or c             | ✅ Yes                  | ❌ No                  |
| Index-only scan for b, c   | ✅ Yes                  | ✅ Yes                 |
| Index size                 | Larger (b,c in tree)    | Smaller (b,c in leaf)  |
| Index maintenance cost     | Higher                  | Lower                  |

**Rule of thumb**: Use `INCLUDE` for columns you SELECT but never filter/sort on. Use composite keys for columns you filter or sort on.

### Visibility Map and Index-Only Scans

In PostgreSQL, an Index Only Scan still needs to check the **visibility map** to confirm rows haven't been deleted/updated (MVCC). If many rows are recently modified, PostgreSQL falls back to a regular index scan. Running `VACUUM` updates the visibility map.

```sql
-- Check if your query gets an Index Only Scan
EXPLAIN ANALYZE
SELECT order_date, amount FROM orders WHERE customer_id = 123;

-- Look for: "Index Only Scan using idx_orders_customer_covering"
-- If you see: "Heap Fetches: 0" → pure index-only scan!
-- If you see: "Heap Fetches: 500" → visibility map needs VACUUM
```

### 💡 Interview Insight

> **"What is a covering index and when would you use one?"**
>
> *"A covering index includes all columns a query needs — both the filter columns and the output columns — so the database can answer the query entirely from the index without touching the table. This eliminates random I/O heap fetches and can dramatically speed up queries. I use the INCLUDE clause to add non-searchable columns to the index leaf pages, keeping the index tree small. The tradeoff is increased index size and write overhead, so I only create covering indexes for high-frequency, performance-critical queries."*

---

## Screen 5: Join Algorithms — Nested Loop, Hash Join, Merge Join

### The Three Join Strategies

When you write `JOIN`, the database must decide **how** to physically combine the rows. There are three fundamental algorithms, and the optimizer picks based on table sizes, available indexes, and sort order.

### Nested Loop Join

```
FOR each row in outer_table:          ← "Driving" table
    FOR each row in inner_table:      ← "Probe" table
        IF join_condition matches:
            output combined row
```

**Best when**: Outer table is small, inner table has an index on the join column.

```
Outer (10 rows) × Inner (1M rows with index)
= 10 index lookups × O(log n) each = ~10 × 20 = 200 operations
```

**Worst when**: Both tables are large and no index exists → becomes O(n × m).

```sql
-- The optimizer often picks Nested Loop for:
EXPLAIN ANALYZE
SELECT c.name, o.amount
FROM customers c            -- Small table (driving)
JOIN orders o ON c.customer_id = o.customer_id  -- Large table (indexed)
WHERE c.customer_id = 42;   -- Very selective → tiny outer set
```

### Hash Join

```
Step 1: Build a hash table from the SMALLER table (build side)
Step 2: Probe the hash table for each row of the LARGER table (probe side)
```

```
Build (5K customers → hash table in memory)
Probe (1M orders → hash lookup per row)
= 1M hash lookups ≈ O(n + m) total
```

**Best when**: No useful indexes, both tables are moderately sized, and the smaller table fits in memory (`work_mem` in PostgreSQL).

**Worst when**: Build table doesn't fit in memory → spills to disk (multi-batch hash join).

```sql
-- Hash Join is common for large equi-joins without indexes
EXPLAIN ANALYZE
SELECT c.name, SUM(o.amount)
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id  -- No index? Hash Join.
GROUP BY c.name;
```

### Merge Join (Sort-Merge Join)

```
Step 1: Sort both tables on the join key (or use existing sort/index order)
Step 2: Walk through both sorted lists simultaneously, matching rows
```

```
Sort A (n log n) + Sort B (m log m) + Merge (n + m)
If already sorted: just O(n + m) — the fastest possible!
```

**Best when**: Both inputs are already sorted (from an index or a preceding ORDER BY), or the result itself needs to be sorted.

```sql
-- Merge Join is chosen when:
-- 1. Both sides have indexes that provide sorted output
-- 2. The query has ORDER BY on the join key
EXPLAIN ANALYZE
SELECT c.name, o.amount
FROM customers c                   -- Index on customer_id (sorted)
JOIN orders o ON c.customer_id = o.customer_id  -- Index on customer_id (sorted)
ORDER BY c.customer_id;
```

### Comparison Matrix

| Algorithm    | Time Complexity      | Memory          | Best Scenario                  | Index Required? |
|:-------------|:---------------------|:----------------|:-------------------------------|:----------------|
| Nested Loop  | O(n × m) worst       | O(1)            | Small outer + indexed inner    | Helps greatly   |
| Hash Join    | O(n + m)             | O(smaller)      | Large tables, no index, equi   | No              |
| Merge Join   | O(n log n + m log m) | O(sort buffers) | Pre-sorted inputs, large data  | Helps (sorted)  |

```
When the optimizer chooses each:

Small × Large (indexed)  →  Nested Loop
Large × Large (unsorted) →  Hash Join
Large × Large (sorted)   →  Merge Join
Any × Any (non-equi join like >, <) → Nested Loop (hash/merge need equality)
```

### 💡 Interview Insight

> **"Explain the three join algorithms and when each is optimal."**
>
> *"Nested Loop is best for small-large joins with an index — it loops through the small table and does an index lookup per row. Hash Join is optimal for large equi-joins without indexes — it builds a hash table from the smaller side and probes it with the larger side, giving O(n+m) performance. Merge Join excels when both inputs are already sorted (from indexes or preceding operations) — it walks both sorted streams in tandem. The optimizer considers table sizes, available indexes, join predicate type (= vs > vs LIKE), and memory (work_mem) to choose. Non-equi joins can only use Nested Loop."*

---

## Screen 6: Statistics, ANALYZE, and Selectivity

### Why the Optimizer Needs Statistics

The query optimizer is a **cost-based planner**. It estimates the cost of different strategies and picks the cheapest one. But to estimate costs, it needs to know:

- How many rows does this table have?
- How many distinct values does this column have?
- What's the data distribution? (uniform? skewed?)
- What fraction of rows will pass this WHERE filter? (selectivity)

These come from **table statistics**, updated by the `ANALYZE` command.

### What ANALYZE Collects

```sql
ANALYZE orders;  -- Update stats for the orders table
ANALYZE;         -- Update stats for ALL tables in the database
```

PostgreSQL samples rows (default: 300 × `default_statistics_target`, typically 30,000 rows) and builds:

| Statistic             | What It Stores                          | Used For                    |
|:----------------------|:----------------------------------------|:----------------------------|
| `n_distinct`          | Number of distinct values               | Estimating GROUP BY sizes   |
| `null_frac`           | Fraction of NULLs                       | NULL handling               |
| `avg_width`           | Average column width in bytes           | Memory/IO estimation        |
| `most_common_vals`    | Most frequent values + their frequencies| Skewed data estimation      |
| `histogram_bounds`    | Evenly-spaced value boundaries          | Range query estimation      |
| `correlation`         | Physical vs logical ordering            | Index scan vs bitmap choice |

### Selectivity Estimation

**Selectivity** = the fraction of rows a predicate matches (0.0 to 1.0).

```sql
-- If orders has 1M rows and the optimizer estimates selectivity of 0.001:
-- Estimated rows = 1,000,000 × 0.001 = 1,000 rows

-- How it estimates:
WHERE status = 'shipped'        -- Check most_common_vals: 'shipped' appears 35% → sel = 0.35
WHERE amount > 500              -- Check histogram: 500 falls in bucket 7/10 → sel ≈ 0.30
WHERE status = 'rare_value'     -- Not in most_common_vals → estimate using n_distinct
```

### When Statistics Go Wrong

```sql
-- Symptom: EXPLAIN shows estimated rows = 100, actual rows = 500,000
-- Cause: stale statistics after bulk load/delete
-- Fix:
ANALYZE orders;

-- For columns with unusual distributions, increase statistics target:
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 1000;  -- default is 100
ANALYZE orders;
```

### Extended Statistics (PostgreSQL 10+)

Standard statistics are **per-column**. But what if columns are correlated?

```sql
-- city and zip_code are highly correlated — knowing city tells you the zip
-- Without extended stats, optimizer assumes independence:
-- P(city='NYC' AND zip='10001') = P(city='NYC') × P(zip='10001') = 0.05 × 0.0001 = 0.000005
-- Reality: P(city='NYC' AND zip='10001') = 0.003 (600× higher!)

CREATE STATISTICS stat_city_zip (dependencies, ndistinct) 
    ON city, zip_code FROM addresses;
ANALYZE addresses;
```

### 💡 Interview Insight

> **"A query was fast yesterday but is slow today after a bulk data load. What happened?"**
>
> *"The table statistics are likely stale. After a large data load, the row counts and value distributions have changed, but the optimizer is still using the old statistics. This causes it to make bad estimates — for example, choosing a Seq Scan when an Index Scan would be far better, or picking the wrong join order. The fix is to run ANALYZE on the affected tables. In production, I'd also configure autovacuum/autoanalyze to run more aggressively on tables that receive bulk loads, using per-table settings for `autovacuum_analyze_threshold` and `autovacuum_analyze_scale_factor`."*

---

## Screen 7: Query Rewriting and Materialized Views

### Predicate Pushdown

The optimizer tries to push WHERE filters **as early as possible** — ideally into the scan layer so fewer rows flow through joins and aggregations.

```sql
-- The optimizer rewrites this:
SELECT * FROM (
    SELECT c.name, o.amount, o.order_date
    FROM customers c
    JOIN orders o ON c.customer_id = o.customer_id
) sub
WHERE order_date > '2024-01-01';

-- Into this (pushes filter down to the orders scan):
SELECT c.name, o.amount, o.order_date
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_date > '2024-01-01';  -- Filter applied during scan, not after join
```

**When predicate pushdown fails**: Across views with `UNION`, `DISTINCT`, `GROUP BY`, or (in older PostgreSQL) materialized CTEs.

### Join Reordering

The optimizer can rearrange join order to minimize intermediate result sizes:

```sql
-- You write:
FROM orders o                        -- 10M rows
JOIN order_items oi ON ...           -- 50M rows
JOIN products p ON ...               -- 100K rows
WHERE p.category = 'Electronics'

-- Optimizer might rewrite as:
FROM products p                       -- 100K rows
WHERE p.category = 'Electronics'      -- → 5K rows after filter
JOIN order_items oi ON ...            -- → 500K matching items
JOIN orders o ON ...                  -- → 500K matching orders
```

Starting with the most selective filter produces smaller intermediate results.

### Materialized Views

A materialized view is a **cached query result** stored as a physical table. Unlike regular views (which are just saved SQL), materialized views store actual data.

```sql
-- Create a materialized view for a common dashboard query
CREATE MATERIALIZED VIEW mv_daily_revenue AS
SELECT 
    DATE_TRUNC('day', order_date) AS day,
    category,
    COUNT(*) AS order_count,
    SUM(amount) AS revenue,
    AVG(amount) AS avg_order_value
FROM orders o
JOIN products p ON o.product_id = p.product_id
WHERE order_date >= CURRENT_DATE - INTERVAL '2 years'
GROUP BY DATE_TRUNC('day', order_date), category;

-- Create an index on the materialized view for fast lookups
CREATE INDEX idx_mv_daily_rev ON mv_daily_revenue (day, category);

-- Query it like a table (instant!)
SELECT * FROM mv_daily_revenue
WHERE day >= '2024-01-01' AND category = 'Electronics';
```

### Refresh Strategies

```sql
-- Full refresh (locks the view, replaces all data)
REFRESH MATERIALIZED VIEW mv_daily_revenue;

-- Concurrent refresh (no lock, but requires a unique index)
CREATE UNIQUE INDEX idx_mv_unique ON mv_daily_revenue (day, category);
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_revenue;
```

| Strategy             | Freshness           | Downtime  | Complexity |
|:---------------------|:--------------------|:----------|:-----------|
| Manual refresh       | On-demand           | Brief     | Low        |
| Cron job (hourly)    | Up to 1 hour stale  | Brief     | Medium     |
| Trigger-based        | Near real-time      | None      | High       |
| Logical replication  | Near real-time      | None      | Very high  |

### When to Use Materialized Views

✅ Dashboard queries that aggregate millions of rows but don't need real-time data
✅ Complex joins that rarely change (product catalogs, historical reports)
✅ Pre-computed rollups for BI tools
❌ Data that needs to be real-time (use indexed views or caching instead)
❌ Tables with high write rates (refresh cost eats your gains)

### 💡 Interview Insight

> **"How do you optimize a dashboard that queries billions of rows?"**
>
> *"I'd use a layered approach. First, I'd create materialized views for the common aggregation patterns — daily/weekly/monthly rollups — and refresh them on a schedule (hourly or nightly depending on freshness requirements). I'd add indexes on the materialized view for common filter patterns. For the refresh, I'd use CONCURRENTLY to avoid blocking reads. This turns a 30-second billion-row aggregation into a 10ms lookup on a pre-computed table. For real-time needs on top, I'd layer in an incremental cache that combines the materialized view (historical) with a live query over recent data (since last refresh)."*

---

## Screen 8: Table Partitioning — Range, List, Hash

### Why Partition?

Partitioning splits a single logical table into multiple physical sub-tables. The database can then **prune** partitions that don't match query filters, dramatically reducing I/O.

```
Without partitioning:
  Query scans ALL 500M rows to find January 2024 data

With partitioning (by month):
  Query scans ONLY the January 2024 partition (~10M rows)
  = 50x less data scanned
```

### Range Partitioning

Best for **time-series data** — the most common partitioning strategy.

```sql
-- PostgreSQL declarative partitioning
CREATE TABLE orders (
    order_id    BIGINT,
    customer_id INT,
    order_date  DATE,
    amount      DECIMAL(10,2),
    status      TEXT
) PARTITION BY RANGE (order_date);

-- Create monthly partitions
CREATE TABLE orders_2024_01 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
CREATE TABLE orders_2024_02 PARTITION OF orders
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');
CREATE TABLE orders_2024_03 PARTITION OF orders
    FOR VALUES FROM ('2024-03-01') TO ('2024-04-01');
-- ... and so on
```

```sql
-- This query only scans the January partition (partition pruning):
EXPLAIN ANALYZE
SELECT * FROM orders WHERE order_date = '2024-01-15';
-- Output: "Seq Scan on orders_2024_01" (not the other partitions!)
```

### List Partitioning

Best for **categorical data** with discrete values.

```sql
CREATE TABLE orders (
    order_id    BIGINT,
    customer_id INT,
    region      TEXT,
    amount      DECIMAL(10,2)
) PARTITION BY LIST (region);

CREATE TABLE orders_east    PARTITION OF orders FOR VALUES IN ('east', 'northeast', 'southeast');
CREATE TABLE orders_west    PARTITION OF orders FOR VALUES IN ('west', 'northwest', 'southwest');
CREATE TABLE orders_central PARTITION OF orders FOR VALUES IN ('central', 'midwest');
```

### Hash Partitioning

Best for **even data distribution** when there's no natural range or list.

```sql
CREATE TABLE orders (
    order_id    BIGINT,
    customer_id INT,
    amount      DECIMAL(10,2)
) PARTITION BY HASH (customer_id);

CREATE TABLE orders_p0 PARTITION OF orders FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE orders_p1 PARTITION OF orders FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE orders_p2 PARTITION OF orders FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE orders_p3 PARTITION OF orders FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

### Partition Pruning in Action

```sql
-- EXPLAIN shows which partitions are actually scanned
EXPLAIN ANALYZE
SELECT SUM(amount)
FROM orders
WHERE order_date BETWEEN '2024-01-01' AND '2024-03-31';

-- Output:
-- Append
--   -> Seq Scan on orders_2024_01   ✅ scanned
--   -> Seq Scan on orders_2024_02   ✅ scanned
--   -> Seq Scan on orders_2024_03   ✅ scanned
--   (orders_2024_04 through orders_2024_12 pruned!)
```

### Partitioning Comparison

| Type  | Best For                    | Pruning on       | Even Distribution |
|:------|:----------------------------|:-----------------|:------------------|
| Range | Time-series, dates          | Range predicates | Depends on data   |
| List  | Regions, categories, status | IN / = predicates| Depends on data   |
| Hash  | Even distribution needed    | = predicates     | ✅ Guaranteed      |

### Partitioning Pitfalls

- **Partition key must be in every unique/primary key constraint** — you can't have a PK on `order_id` alone; it must be `(order_id, order_date)`.
- **Cross-partition queries** (without filter on partition key) scan ALL partitions.
- **Too many partitions** (>1000) can slow down planning time.
- **Foreign keys** referencing partitioned tables have limitations in PostgreSQL.

### 💡 Interview Insight

> **"When would you partition a table and what strategy would you use?"**
>
> *"I'd partition when a table exceeds tens of millions of rows and queries consistently filter on a specific column. For time-series data (the most common case), I'd use range partitioning by month or day — this enables partition pruning so queries for 'last 30 days' only scan 1-2 partitions instead of the whole table. I'd also partition for data lifecycle management — dropping old partitions is instantaneous compared to DELETE on millions of rows. Key considerations: the partition key must be in all unique constraints, and I'd keep partition count reasonable (dozens to hundreds, not thousands)."*

---

## Screen 9: Quiz

**Q1: You run EXPLAIN ANALYZE and see: estimated rows=100, actual rows=750,000. What's the most likely cause and fix?**
- A) The query is missing a LIMIT clause
- B) The index is corrupted and needs to be rebuilt
- C) Table statistics are stale; run ANALYZE on the table ✅
- D) The table needs to be partitioned

> **Answer: C** — A massive discrepancy between estimated and actual rows almost always means the optimizer is working with outdated statistics. After bulk inserts, deletes, or schema changes, running `ANALYZE tablename` refreshes the statistics (row count, value distribution histograms, distinct value counts) so the optimizer can make better decisions. Partitioning and indexes might help overall, but the root cause of a bad plan is bad estimates.

**Q2: You have a composite index on `orders(status, order_date, customer_id)`. Which query can NOT use this index efficiently?**
- A) `WHERE status = 'active' AND order_date > '2024-01-01'`
- B) `WHERE status = 'active'`
- C) `WHERE order_date > '2024-01-01' AND customer_id = 42` ✅
- D) `WHERE status = 'active' AND order_date = '2024-01-15' AND customer_id = 42`

> **Answer: C** — The leftmost prefix rule requires queries to include the first column(s) of the index. Option C skips `status` (the leftmost column), so the index can't be used for seeking. The database would need a full index scan or table scan. Options A, B, and D all start with `status`, so they can use the index.

**Q3: When does the optimizer choose a Hash Join over a Nested Loop Join?**
- A) When both tables are small and have indexes
- B) When one table is small, the other is large with an index on the join column
- C) When both tables are large, there's no useful index, and the join is an equi-join ✅
- D) When the query includes ORDER BY on the join column

> **Answer: C** — Hash Join excels at large equi-joins without indexes: it builds a hash table from the smaller table and probes it with each row of the larger table, achieving O(n+m) performance. Nested Loop is better when the outer table is small and the inner table has an index (option B). Merge Join benefits from pre-sorted input or when ORDER BY aligns with the join (option D).

**Q4: What is the purpose of the `INCLUDE` clause in an index definition?**
- A) To add columns that can be used for filtering in WHERE clauses
- B) To store additional columns in the index leaf pages for index-only scans ✅
- C) To include NULL values in the index
- D) To create a partial index on specific rows

> **Answer: B** — `INCLUDE` adds non-key columns to the leaf pages of the index, enabling index-only scans without the columns being part of the B-tree search key. These columns can't be used for filtering or sorting — they're just "along for the ride" so the database doesn't need to fetch the actual table row. This keeps the index tree smaller than a full composite index while still avoiding heap fetches.

**Q5: Range partitioning a 500M-row `orders` table by `order_date` (monthly partitions) helps performance MOST for which query?**
- A) `SELECT * FROM orders WHERE customer_id = 42`
- B) `SELECT COUNT(*) FROM orders` (full table count)
- C) `SELECT * FROM orders WHERE order_date BETWEEN '2024-03-01' AND '2024-03-31'` ✅
- D) `SELECT * FROM orders ORDER BY amount DESC LIMIT 10`

> **Answer: C** — This query filters on the partition key (`order_date`), so partition pruning kicks in and only the March 2024 partition is scanned (maybe ~10M rows instead of 500M). Options A, B, and D don't filter on the partition key, so all partitions must be scanned — partitioning provides no benefit for those queries.

---

## Screen 10: Key Takeaways

- **EXPLAIN ANALYZE is your most important diagnostic tool.** Read it bottom-up, compare estimated vs actual rows, and look for Seq Scans on large tables with selective filters. The biggest performance wins come from finding and fixing bad estimates.

- **B-tree indexes handle 90% of use cases** (equality, range, sorting, prefix). Use GIN for JSONB/arrays/full-text, GiST for spatial data, and Hash almost never. Know which operations each index type supports.

- **Composite index column order follows the ERS rule**: Equality columns first, Range columns next, Sort columns last. The leftmost prefix rule means `(A, B, C)` helps queries on `A`, `A+B`, or `A+B+C` — never `B` or `C` alone.

- **Covering indexes with INCLUDE eliminate heap fetches** — the query is answered entirely from the index. Use them for high-frequency queries where you SELECT columns beyond the filter columns.

- **Three join algorithms**: Nested Loop (small × indexed-large), Hash Join (large × large, no index, equi-join), Merge Join (pre-sorted inputs). The optimizer chooses based on table sizes, indexes, join type, and available memory.

- **Stale statistics are the #1 cause of bad query plans.** Run `ANALYZE` after bulk data changes. Increase `default_statistics_target` for columns with skewed distributions. Use extended statistics for correlated columns.

- **Materialized views trade freshness for speed** — they're pre-computed query results stored as tables. Use them for dashboard aggregations, refresh on a schedule, and add indexes for the access patterns your BI tools need.

- **Partition for data lifecycle and query pruning.** Range partitioning by date is the most common strategy. Partition pruning only helps when queries filter on the partition key. Keep partition counts reasonable (<1000) and remember the partition key must be in all unique constraints.
