# Module 2: CTEs & Advanced Query Patterns

> **Goal**: Master CTEs (including recursive), lateral joins, advanced aggregations, and subquery patterns. These are the building blocks of complex analytical queries — the kind that separate mid-level from senior SQL engineers in interviews.

---

## Screen 1: Common Table Expressions — Your Query's Outline

### What Is a CTE?

A Common Table Expression (CTE) is a **named temporary result set** that exists only for the duration of a single query. Think of it as giving a name to a subquery and placing it at the top of your SQL statement, like defining variables before using them.

```sql
WITH monthly_revenue AS (
    SELECT 
        DATE_TRUNC('month', order_date) AS month,
        SUM(amount) AS revenue
    FROM orders
    GROUP BY DATE_TRUNC('month', order_date)
)
SELECT month, revenue,
       revenue - LAG(revenue) OVER (ORDER BY month) AS mom_change
FROM monthly_revenue;
```

### Why CTEs Beat Nested Subqueries

Without CTEs, complex queries become "Russian nesting dolls" — subqueries inside subqueries inside subqueries. Consider this nightmare:

```sql
-- ❌ The subquery nesting nightmare
SELECT customer_id, name, total_spend, avg_category_spend
FROM (
    SELECT customer_id, name, total_spend,
           AVG(total_spend) OVER (PARTITION BY category) AS avg_category_spend
    FROM (
        SELECT c.customer_id, c.name, p.category,
               SUM(o.amount) AS total_spend
        FROM customers c
        JOIN orders o ON c.customer_id = o.customer_id
        JOIN products p ON o.product_id = p.product_id
        GROUP BY c.customer_id, c.name, p.category
    ) sub1
) sub2
WHERE total_spend > avg_category_spend;
```

Now with CTEs:

```sql
-- ✅ The CTE way — reads top-to-bottom like a story
WITH customer_category_spend AS (
    SELECT c.customer_id, c.name, p.category,
           SUM(o.amount) AS total_spend
    FROM customers c
    JOIN orders o ON c.customer_id = o.customer_id
    JOIN products p ON o.product_id = p.product_id
    GROUP BY c.customer_id, c.name, p.category
),
category_averages AS (
    SELECT *,
           AVG(total_spend) OVER (PARTITION BY category) AS avg_category_spend
    FROM customer_category_spend
)
SELECT customer_id, name, total_spend, avg_category_spend
FROM category_averages
WHERE total_spend > avg_category_spend;
```

### Chaining Multiple CTEs

You can define multiple CTEs separated by commas, and each CTE can reference the ones defined before it:

```sql
WITH 
step1 AS (...),
step2 AS (SELECT ... FROM step1 ...),   -- Can reference step1
step3 AS (SELECT ... FROM step1, step2)  -- Can reference step1 AND step2
SELECT * FROM step3;
```

This creates a **data pipeline in SQL** — each step transforms data from the previous step.

### CTE vs Subquery vs Temp Table

| Feature          | CTE                    | Subquery                | Temp Table            |
|:-----------------|:-----------------------|:------------------------|:----------------------|
| Scope            | Single statement       | Single use (inline)     | Entire session        |
| Readability      | ⭐⭐⭐ Excellent       | ⭐ Poor when nested     | ⭐⭐ Good             |
| Reusable in query| Yes (ref multiple×)    | No (must repeat)        | Yes                   |
| Materialized?    | Optimizer decides*     | Optimizer decides       | Yes (on disk)         |
| Recursive?       | Yes!                   | No                      | N/A                   |

*In PostgreSQL, CTEs were **materialization fences** before v12 — the optimizer couldn't push predicates into them. Since v12, non-recursive CTEs are inlined by default. In MySQL, CTEs may still be materialized. Know your engine.

### 💡 Interview Insight

> **"Are CTEs materialized or inlined?"**
>
> *"It depends on the database engine. In PostgreSQL < 12, CTEs were always materialized — creating an optimization barrier. Since PostgreSQL 12, non-recursive CTEs are inlined by default (the optimizer can push predicates into them), but you can force materialization with `AS MATERIALIZED`. In MySQL 8.0+, CTEs referenced multiple times are materialized. In BigQuery and Snowflake, the optimizer makes its own decision. The key takeaway: don't assume CTE = materialization, and test with EXPLAIN if performance matters."*

---

## Screen 2: Recursive CTEs — Trees, Graphs, and Sequences

### The Concept

A recursive CTE references **itself**. It has two parts:

1. **Base case (anchor)**: The starting point — doesn't reference the CTE
2. **Recursive step**: References the CTE to build upon previous results
3. Connected by `UNION ALL`

```sql
WITH RECURSIVE cte_name AS (
    -- Base case: seed rows
    SELECT ... 
    
    UNION ALL
    
    -- Recursive step: join CTE to itself
    SELECT ... 
    FROM cte_name
    JOIN some_table ON ...
    WHERE termination_condition
)
SELECT * FROM cte_name;
```

### Example 1: Organizational Hierarchy

*"Given an `employees` table with a `manager_id` column, show the full reporting chain for each employee."*

```sql
-- employees: (employee_id, name, manager_id, title)

WITH RECURSIVE org_chart AS (
    -- Base case: top-level executives (no manager)
    SELECT employee_id, name, title, manager_id,
           1 AS level,
           name::TEXT AS reporting_chain
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Recursive step: find direct reports of known employees
    SELECT e.employee_id, e.name, e.title, e.manager_id,
           oc.level + 1,
           oc.reporting_chain || ' → ' || e.name
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.employee_id
)
SELECT employee_id, name, title, level, reporting_chain
FROM org_chart
ORDER BY reporting_chain;
```

Output:
```
employee_id | name    | title     | level | reporting_chain
1           | Alice   | CEO       | 1     | Alice
2           | Bob     | VP Eng    | 2     | Alice → Bob
4           | Diana   | Sr Eng    | 3     | Alice → Bob → Diana
3           | Charlie | VP Sales  | 2     | Alice → Charlie
```

### Example 2: Bill of Materials / Product Components

```sql
-- product_components: (product_id, component_id, quantity)

WITH RECURSIVE bom AS (
    -- Base case: top-level product
    SELECT product_id, component_id, quantity, 1 AS depth
    FROM product_components
    WHERE product_id = 'LAPTOP-X1'
    
    UNION ALL
    
    -- Recursive: components of components
    SELECT bom.product_id, pc.component_id, 
           bom.quantity * pc.quantity AS quantity,  -- Multiply quantities down
           bom.depth + 1
    FROM bom
    JOIN product_components pc ON bom.component_id = pc.product_id
)
SELECT * FROM bom;
```

### Example 3: Sequence Generation

```sql
-- Generate dates for the last 30 days (useful for gap-filling)
WITH RECURSIVE date_series AS (
    SELECT CURRENT_DATE - INTERVAL '30 days' AS dt
    
    UNION ALL
    
    SELECT dt + INTERVAL '1 day'
    FROM date_series
    WHERE dt < CURRENT_DATE
)
SELECT dt::DATE FROM date_series;
```

### Safeguards Against Infinite Loops

Recursive CTEs can loop forever if the termination condition is wrong. Protect yourself:

```sql
-- PostgreSQL: LIMIT or max recursion depth check
WITH RECURSIVE cte AS (
    SELECT 1 AS n
    UNION ALL
    SELECT n + 1 FROM cte WHERE n < 1000  -- Explicit termination
)
SELECT * FROM cte;

-- SQL Server: OPTION (MAXRECURSION 100)
-- PostgreSQL: No built-in limit, but you can add WHERE depth < 100
```

### 💡 Interview Insight

> **"When would you use a recursive CTE?"**
>
> *"Recursive CTEs are my go-to for hierarchical or graph-like data: org charts, category trees, bill-of-materials, file system paths, and social network friend-of-friend queries. They're also great for generating sequences — like a date series for gap-filling in reports. The pattern is always: anchor query seeds the starting rows, recursive step extends them by one level, and a WHERE clause prevents infinite recursion. I'm careful about performance though — for deep hierarchies (1000+ levels), I might consider a materialized path or nested set model instead."*

---

## Screen 3: Lateral Joins — Correlated Subqueries Done Right

### The Problem

You want to find the **3 most recent orders for each customer**. The naive approach uses a correlated subquery:

```sql
-- Correlated subquery — ugly and often slow
SELECT c.customer_id, c.name, o.order_id, o.amount
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
WHERE o.order_id IN (
    SELECT o2.order_id 
    FROM orders o2 
    WHERE o2.customer_id = c.customer_id
    ORDER BY o2.order_date DESC
    LIMIT 3
);
```

Some databases won't even allow `LIMIT` inside a correlated subquery. Enter **lateral joins**.

### LATERAL / CROSS APPLY

A `LATERAL` join (PostgreSQL, MySQL 8+) or `CROSS APPLY` (SQL Server) lets a subquery **reference columns from preceding tables** in the FROM clause. It's like a correlated subquery, but it can return multiple rows and columns.

```sql
-- PostgreSQL: LATERAL
SELECT c.customer_id, c.name, recent.order_id, recent.amount, recent.order_date
FROM customers c
CROSS JOIN LATERAL (
    SELECT o.order_id, o.amount, o.order_date
    FROM orders o
    WHERE o.customer_id = c.customer_id
    ORDER BY o.order_date DESC
    LIMIT 3
) AS recent;
```

```sql
-- SQL Server: CROSS APPLY (same semantics)
SELECT c.customer_id, c.name, recent.order_id, recent.amount, recent.order_date
FROM customers c
CROSS APPLY (
    SELECT TOP 3 o.order_id, o.amount, o.order_date
    FROM orders o
    WHERE o.customer_id = c.customer_id
    ORDER BY o.order_date DESC
) AS recent;
```

### CROSS JOIN LATERAL vs LEFT JOIN LATERAL

```
CROSS JOIN LATERAL = CROSS APPLY  → Excludes customers with 0 matching rows
LEFT  JOIN LATERAL = OUTER APPLY  → Includes all customers, NULLs if no matches
```

```sql
-- Include customers even if they have no orders
SELECT c.customer_id, c.name, recent.order_id, recent.amount
FROM customers c
LEFT JOIN LATERAL (
    SELECT o.order_id, o.amount
    FROM orders o
    WHERE o.customer_id = c.customer_id
    ORDER BY o.order_date DESC
    LIMIT 3
) AS recent ON TRUE;  -- ON TRUE required for LEFT JOIN LATERAL in PostgreSQL
```

### When LATERAL Beats Alternatives

| Approach                    | Lateral Equivalent? | Limitation                        |
|:----------------------------|:--------------------|:----------------------------------|
| Window function + filter    | Often yes           | Can't LIMIT per group easily      |
| Correlated subquery         | Yes                 | Returns only 1 value              |
| Self-join + ROW_NUMBER CTE  | Yes                 | More verbose, full sort required   |
| LATERAL subquery            | —                   | Clean, efficient, supports LIMIT  |

The optimizer can often convert a LATERAL join into an efficient **nested loop with index seek** — especially if the subquery's WHERE clause hits an index.

### 💡 Interview Insight

> **"What is a lateral join, and when would you use one?"**
>
> *"A LATERAL join allows a subquery in the FROM clause to reference columns from preceding tables — essentially a correlated subquery that can return multiple rows. I'd use it for 'top-N per group' queries where I need LIMIT per group (which window functions can't do directly), for calling table-valued functions per row, or for unnesting arrays per row. In PostgreSQL it's `JOIN LATERAL`, in SQL Server it's `CROSS APPLY`. It's often more efficient than the ROW_NUMBER CTE pattern because the optimizer can use an index seek + limit per group instead of computing ranks for all rows."*

---

## Screen 4: GROUPING SETS, CUBE, and ROLLUP

### Beyond Simple GROUP BY

Sometimes you need **multiple levels of aggregation** in a single query — totals by product, by category, by region, and a grand total. Without `GROUPING SETS`, you'd write multiple queries and `UNION ALL` them:

```sql
-- ❌ The tedious way
SELECT category, NULL AS region, SUM(revenue) FROM sales GROUP BY category
UNION ALL
SELECT NULL, region, SUM(revenue) FROM sales GROUP BY region
UNION ALL
SELECT NULL, NULL, SUM(revenue) FROM sales;
```

### GROUPING SETS — Pick Your Aggregation Levels

```sql
-- ✅ GROUPING SETS — all levels in one pass
SELECT category, region, SUM(revenue) AS total_revenue
FROM sales
GROUP BY GROUPING SETS (
    (category),           -- Subtotal per category
    (region),             -- Subtotal per region
    (category, region),   -- Detail per category+region
    ()                    -- Grand total
);
```

| category    | region | total_revenue |
|:------------|:-------|:-------------|
| Electronics | NULL   | 50000         |
| Clothing    | NULL   | 30000         |
| NULL        | East   | 45000         |
| NULL        | West   | 35000         |
| Electronics | East   | 28000         |
| Electronics | West   | 22000         |
| Clothing    | East   | 17000         |
| Clothing    | West   | 13000         |
| NULL        | NULL   | 80000         |

The NULLs indicate "all values" at that level. To distinguish between a NULL meaning "aggregated" vs a genuine NULL in the data, use the `GROUPING()` function:

```sql
SELECT 
    CASE WHEN GROUPING(category) = 1 THEN 'ALL' ELSE category END AS category,
    CASE WHEN GROUPING(region) = 1 THEN 'ALL' ELSE region END AS region,
    SUM(revenue) AS total_revenue
FROM sales
GROUP BY GROUPING SETS ((category), (region), (category, region), ());
```

### ROLLUP — Hierarchical Subtotals

`ROLLUP` generates subtotals that "roll up" from right to left — perfect for hierarchical dimensions like (year, quarter, month):

```sql
-- ROLLUP(a, b, c) = GROUPING SETS ((a,b,c), (a,b), (a), ())
SELECT 
    EXTRACT(YEAR FROM order_date) AS yr,
    EXTRACT(QUARTER FROM order_date) AS qtr,
    EXTRACT(MONTH FROM order_date) AS mo,
    SUM(amount) AS revenue
FROM orders
GROUP BY ROLLUP (
    EXTRACT(YEAR FROM order_date),
    EXTRACT(QUARTER FROM order_date),
    EXTRACT(MONTH FROM order_date)
);
```

This produces:
```
Year + Quarter + Month detail   (most granular)
Year + Quarter subtotals
Year subtotals
Grand total                     (least granular)
```

### CUBE — All Possible Combinations

`CUBE` generates aggregations for **every possible combination** of the listed columns:

```sql
-- CUBE(a, b) = GROUPING SETS ((a,b), (a), (b), ())
SELECT category, region, SUM(revenue) AS total_revenue
FROM sales
GROUP BY CUBE (category, region);
```

```
CUBE(a, b, c) produces 2^3 = 8 grouping sets
CUBE(a, b)    produces 2^2 = 4 grouping sets
```

### Quick Reference

| Syntax                 | Generates                        | # Sets for n cols |
|:-----------------------|:---------------------------------|:------------------|
| `GROUPING SETS (...)`  | Exactly what you specify         | Explicit          |
| `ROLLUP (a, b, c)`    | `(a,b,c), (a,b), (a), ()`       | n + 1             |
| `CUBE (a, b, c)`      | All subsets of `{a, b, c}`       | 2^n               |

### 💡 Interview Insight

> **"When would you use ROLLUP vs CUBE?"**
>
> *"ROLLUP is for hierarchical dimensions — like geographic rollups (city → state → country) or time rollups (day → month → year) where there's a natural parent-child relationship. CUBE is for cross-tabulation where every combination matters — like analyzing revenue by both product category AND region independently. ROLLUP is more common in practice because most reporting has a natural hierarchy. Both are more efficient than multiple GROUP BY queries with UNION ALL because the database scans the data once."*

---

## Screen 5: UNION ALL vs UNION — The Performance Trap

### The Critical Difference

```sql
-- UNION: removes duplicates (sorts/hashes to deduplicate)
SELECT customer_id FROM online_orders
UNION
SELECT customer_id FROM in_store_orders;

-- UNION ALL: keeps everything, no dedup overhead
SELECT customer_id FROM online_orders
UNION ALL
SELECT customer_id FROM in_store_orders;
```

### Under the Hood

```
UNION ALL:
  Table A rows ──┐
                  ├──→ Result (fast, streaming)
  Table B rows ──┘

UNION:
  Table A rows ──┐
                  ├──→ [Hash / Sort to deduplicate] ──→ Result (slower)
  Table B rows ──┘
```

`UNION` must build a hash table or sort the entire combined result set to find and remove duplicates. On large datasets, this can be **enormously expensive** — it's essentially doing a `DISTINCT` on the entire output.

### Rules of Thumb

1. **Default to `UNION ALL`** unless you specifically need deduplication
2. If the sources are guaranteed disjoint (e.g., partitioned by date range), `UNION` is wasted work
3. If you need dedup, consider whether it should happen per-source or on the combined result
4. Column count and types must match across all branches (names come from the first branch)

### Common Interview Patterns

```sql
-- Combining event tables (definitely use UNION ALL — events are unique)
SELECT event_id, user_id, 'click' AS event_type, created_at FROM click_events
UNION ALL
SELECT event_id, user_id, 'purchase', created_at FROM purchase_events
UNION ALL
SELECT event_id, user_id, 'refund', created_at FROM refund_events
ORDER BY created_at;
```

```sql
-- Finding customers who appear in BOTH channels (use INTERSECT or JOIN instead)
-- DON'T do this:
SELECT customer_id FROM online_orders
UNION
SELECT customer_id FROM in_store_orders;
-- This finds customers in EITHER channel, not BOTH!
```

### Performance Impact — Real Numbers

On a 10M row + 10M row combination:
- `UNION ALL`: ~2 seconds (just concatenation)
- `UNION`: ~15+ seconds (needs to hash/sort 20M rows for deduplication)

### 💡 Interview Insight

> **"When should you use UNION vs UNION ALL?"**
>
> *"UNION ALL should be the default — it's a simple concatenation with no overhead. UNION adds a deduplication step (hash or sort) that can be very expensive on large datasets. I only use UNION when I specifically need to eliminate duplicates across the combined result set, and even then I first ask: could the data sources already be disjoint? If I'm combining partitioned tables, event streams with unique IDs, or tables with non-overlapping primary keys, UNION's dedup is pure waste."*

---

## Screen 6: Correlated vs Non-Correlated Subqueries

### Non-Correlated Subquery

A non-correlated subquery runs **once**, independently of the outer query. It produces a fixed result that the outer query uses.

```sql
-- Non-correlated: the subquery doesn't reference the outer query
SELECT customer_id, name, total_spend
FROM customers
WHERE total_spend > (
    SELECT AVG(total_spend) FROM customers  -- Runs once, returns a single value
);
```

### Correlated Subquery

A correlated subquery runs **once per row** of the outer query. It references columns from the outer query, making it dependent.

```sql
-- Correlated: the subquery references c.customer_id from the outer query
SELECT c.customer_id, c.name,
    (SELECT COUNT(*) 
     FROM orders o 
     WHERE o.customer_id = c.customer_id) AS order_count  -- Runs per customer
FROM customers c;
```

```
Non-Correlated:                    Correlated:
┌──────────────────┐               ┌──────────────────┐
│   Outer Query    │               │   Outer Query    │
│   ┌──────────┐   │               │   Row 1 → ┌────┐ │
│   │ Subquery │───┼─ Runs ONCE    │           │ SQ │─┼─ Run
│   └──────────┘   │               │   Row 2 → │    │─┼─ Run
│                  │               │   Row 3 → │    │─┼─ Run
│                  │               │           └────┘ │
└──────────────────┘               └──────────────────┘
```

### Performance Implications

Non-correlated subqueries in `WHERE` are generally fine — the optimizer executes them once and uses the result as a constant. Correlated subqueries can be devastating: on a 1M-row outer table, a correlated subquery runs 1M times.

However, modern optimizers are smart. They often **decorrelate** correlated subqueries into joins:

```sql
-- The optimizer might transform this correlated subquery...
SELECT c.customer_id, c.name
FROM customers c
WHERE (SELECT MAX(o.amount) FROM orders o WHERE o.customer_id = c.customer_id) > 1000;

-- ...into this join (internally):
SELECT c.customer_id, c.name
FROM customers c
JOIN (SELECT customer_id, MAX(amount) AS max_amt FROM orders GROUP BY customer_id) o
    ON c.customer_id = o.customer_id
WHERE o.max_amt > 1000;
```

### Correlated Subquery with EXISTS

The most efficient use of correlated subqueries is with `EXISTS`:

```sql
-- "Find customers who have at least one order over $500"
SELECT c.customer_id, c.name
FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.customer_id = c.customer_id
    AND o.amount > 500
);
```

`EXISTS` short-circuits: it stops scanning as soon as it finds one matching row. This makes correlated `EXISTS` highly efficient, especially with an index on `orders(customer_id, amount)`.

### 💡 Interview Insight

> **"What's the difference between a correlated and non-correlated subquery, and which is faster?"**
>
> *"A non-correlated subquery is independent of the outer query — it runs once and produces a fixed result. A correlated subquery references the outer query, so conceptually it runs once per outer row. Performance-wise, non-correlated is generally faster, but modern query optimizers can decorrelate many correlated subqueries into joins. The one exception where correlated subqueries shine is with EXISTS — the short-circuit evaluation makes it very efficient for semi-join patterns, often outperforming IN with large subquery results."*

---

## Screen 7: EXISTS vs IN vs JOIN — The Performance Showdown

### The Three Ways to Filter

*"Find customers who have placed orders."*

```sql
-- Method 1: IN
SELECT customer_id, name
FROM customers
WHERE customer_id IN (SELECT customer_id FROM orders);

-- Method 2: EXISTS
SELECT c.customer_id, c.name
FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id);

-- Method 3: JOIN
SELECT DISTINCT c.customer_id, c.name
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id;
```

### Behavioral Differences

| Scenario                    | IN                    | EXISTS               | JOIN                  |
|:----------------------------|:----------------------|:---------------------|:----------------------|
| Subquery returns NULLs      | ⚠️ Three-valued logic | ✅ Ignores NULLs     | ✅ NULLs don't match  |
| Duplicates in subquery      | Handled internally    | Short-circuits       | Creates duplicates!   |
| Need data from both tables  | ❌ Only outer columns | ❌ Only outer columns | ✅ Both tables        |
| Empty subquery              | Returns nothing       | Returns nothing       | Returns nothing       |

### The NULL Trap with IN

```sql
-- This returns NO ROWS if the subquery contains any NULL:
SELECT * FROM customers
WHERE customer_id NOT IN (SELECT manager_id FROM employees);
-- If ANY manager_id is NULL, NOT IN returns empty because:
-- x NOT IN (1, 2, NULL) → x!=1 AND x!=2 AND x!=NULL → always UNKNOWN
```

**Fix**: Always use `NOT EXISTS` instead of `NOT IN` when NULLs are possible:

```sql
-- Safe version
SELECT * FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM employees e WHERE e.manager_id = c.customer_id
);
```

### Performance Comparison

```
Small subquery result, large outer table:
  IN ≈ EXISTS ≈ JOIN  (optimizer converts to similar plans)

Large subquery result, small outer table:
  EXISTS > IN > JOIN  (EXISTS short-circuits)

Both tables large:
  JOIN with hash = EXISTS with hash  (optimizer chooses hash semi-join)
  IN may materialize the subquery list

General advice:
  EXISTS for semi-joins (especially with NOT)
  JOIN when you need columns from both tables
  IN for small, known-no-NULLs lists
```

### The Modern Optimizer Reality

In PostgreSQL and modern SQL Server, all three often produce **identical query plans** — the optimizer transforms them into the same semi-join internally. But that's not universal:

- MySQL historically optimized `EXISTS` better than `IN` with subqueries
- BigQuery and Snowflake handle all three similarly
- The optimizer can't help if the correlated column lacks an index

### 💡 Interview Insight

> **"When do you choose EXISTS over IN or JOIN?"**
>
> *"I default to EXISTS for semi-join patterns ('find rows that have a match') and especially for NOT EXISTS instead of NOT IN — because NOT IN with NULLs in the subquery returns no rows due to three-valued logic, which is a subtle but devastating bug. I use JOIN when I need columns from both tables. I use IN for static lists or small subqueries where I'm confident there are no NULLs. In modern PostgreSQL, the optimizer often converts all three to the same plan, but EXISTS with NOT is the only one that's always NULL-safe."*

---

## Screen 8: Quiz

**Q1: Which of the following is TRUE about CTEs in PostgreSQL 12+?**
- A) CTEs are always materialized as temporary tables
- B) Non-recursive CTEs are inlined by default and can be optimized ✅
- C) CTEs cannot reference other CTEs defined in the same WITH block
- D) Recursive CTEs don't require UNION ALL

> **Answer: B** — Since PostgreSQL 12, non-recursive CTEs are inlined by default (treated as subqueries), allowing the optimizer to push predicates into them. You can force materialization with `AS MATERIALIZED` if needed. Each CTE can reference previously defined CTEs, and recursive CTEs require UNION ALL between the anchor and recursive terms.

**Q2: In a recursive CTE, what prevents infinite recursion?**
- A) The database automatically stops after 100 iterations
- B) The UNION keyword deduplicates rows, preventing cycles
- C) A WHERE clause in the recursive step that eventually produces no new rows ✅
- D) The RECURSIVE keyword sets a default maximum depth

> **Answer: C** — The recursion terminates when the recursive step returns zero rows. This typically happens via a WHERE clause (e.g., `WHERE depth < 10` or a join that finds no more children). Some databases have safety limits (SQL Server's MAXRECURSION = 100 default), but the correct approach is an explicit termination condition. Note: UNION ALL does NOT deduplicate — you need UNION (without ALL) if you need cycle detection, though that's less common.

**Q3: What does `CROSS JOIN LATERAL` do differently from a regular `CROSS JOIN`?**
- A) It performs a Cartesian product of all rows
- B) It allows the right side to reference columns from the left side ✅
- C) It automatically limits results to 1 row per left-side row
- D) It's just an alias for INNER JOIN

> **Answer: B** — The LATERAL keyword allows the subquery on the right side of the join to reference columns from tables on the left side — making it a correlated subquery in the FROM clause. A regular CROSS JOIN's right side is independent of the left. LATERAL is essential for "top-N per group" patterns where you need LIMIT in the correlated subquery.

**Q4: `NOT IN (subquery)` is dangerous because:**
- A) It's always slower than NOT EXISTS
- B) If the subquery returns any NULL value, the entire NOT IN evaluates to empty ✅
- C) It can't use indexes
- D) It doesn't work with correlated subqueries

> **Answer: B** — Due to SQL's three-valued logic, `x NOT IN (1, 2, NULL)` evaluates to `x≠1 AND x≠2 AND x≠NULL`. The last condition is always UNKNOWN, making the entire AND expression UNKNOWN, so no rows pass the filter. This is one of the most common SQL bugs. Always use `NOT EXISTS` instead, which handles NULLs correctly.

**Q5: ROLLUP(year, quarter, month) generates how many grouping levels?**
- A) 3 (one per column)
- B) 4 (each column plus grand total) ✅
- C) 8 (all combinations, like CUBE)
- D) 7 (all combinations minus one)

> **Answer: B** — ROLLUP(a, b, c) generates n+1 = 4 grouping sets: (a,b,c), (a,b), (a), and (). It "rolls up" from right to left, creating hierarchical subtotals. CUBE would generate 2^3 = 8 grouping sets (all possible combinations).

---

## Screen 9: Key Takeaways

- **CTEs transform unreadable nested subqueries into sequential, named steps** — they're your primary tool for query readability. Chain them with commas, and each can reference the ones above it.

- **Recursive CTEs follow a strict pattern**: anchor query → `UNION ALL` → recursive step with termination condition. Use them for hierarchies (org charts, category trees), graph traversal, and sequence generation.

- **LATERAL joins (CROSS APPLY in SQL Server)** let a subquery in FROM reference the outer table — perfect for top-N-per-group with LIMIT, which window functions can't do directly. Use `LEFT JOIN LATERAL ... ON TRUE` to keep rows with no matches.

- **GROUPING SETS give you multi-level aggregation in one pass**. ROLLUP is for hierarchical drill-downs (n+1 levels), CUBE is for all combinations (2^n levels). Both are vastly more efficient than multiple GROUP BY + UNION ALL queries.

- **UNION ALL should be your default** over UNION. The deduplication step in UNION requires a hash or sort of the entire result — skip it unless you genuinely need to remove duplicates.

- **NOT IN with NULLs is a silent killer** — if the subquery returns even one NULL, the entire result set is empty. Always prefer `NOT EXISTS` for anti-semi-joins. This is a top-tier interview gotcha.

- **Modern optimizers often produce identical plans** for IN, EXISTS, and JOIN — but understanding their semantic differences (especially around NULLs and duplicates) is what interviewers really test.
