# Module 1: Window Functions Deep Dive

> **Goal**: Master every window function you'll encounter in SQL interviews — from ranking to running aggregates to sessionization. By the end of this module, you'll be able to solve 90% of "analytics SQL" interview questions on sight.

---

## Screen 1: What Are Window Functions & Why They Exist

### The Problem Window Functions Solve

Imagine you work at an e-commerce company. Your manager asks: *"Show me each order alongside the customer's total lifetime spend."* Without window functions, you'd write something ugly — a subquery or a self-join:

```sql
-- The painful way (correlated subquery)
SELECT o.order_id, o.customer_id, o.amount,
       (SELECT SUM(o2.amount) FROM orders o2 WHERE o2.customer_id = o.customer_id) AS lifetime_spend
FROM orders o;
```

This fires a subquery **for every single row**. On a million-row table, that's a million subqueries. Window functions solve this elegantly:

```sql
-- The window function way
SELECT order_id, customer_id, amount,
       SUM(amount) OVER (PARTITION BY customer_id) AS lifetime_spend
FROM orders;
```

### The Mental Model: A Window Into Your Data

Think of a window function as a **sliding glass panel** that you hold up against your result set. Through this window, each row can "see" other rows — its neighbors, its group, or the entire table — and compute something based on what it sees. Crucially, **the row count never changes**. Unlike `GROUP BY`, which collapses rows, window functions **add columns** to existing rows.

```
┌─────────────────────────────────────────────────────┐
│  Regular Query Result Set (all rows preserved)       │
│  ┌─────────┐                                         │
│  │ Window  │  ← Each row peeks through its window    │
│  │ Frame   │    to see neighboring rows and compute  │
│  └─────────┘    aggregates WITHOUT collapsing them   │
└─────────────────────────────────────────────────────┘
```

### Anatomy of a Window Function Call

```sql
function_name(args) OVER (
    [PARTITION BY column1, column2, ...]   -- Defines groups (like GROUP BY but no collapse)
    [ORDER BY column3 ASC/DESC, ...]       -- Defines ordering within each partition
    [frame_clause]                         -- Defines which rows in the partition to consider
)
```

| Component        | Purpose                          | Required? |
|:-----------------|:---------------------------------|:----------|
| `PARTITION BY`   | Splits rows into groups          | Optional  |
| `ORDER BY`       | Orders rows within partition     | Depends   |
| Frame clause     | Narrows visible rows further     | Optional  |

I `PARTITION BY`, the entire result set is one partition. If you omit the frame clause, the default depends on whether `ORDER BY` is present (we'll cover this gotcha later — it trips up even senior engineers).

### 💡 Interview Insight

> **"What's the difference between GROUP BY and window functions?"**
>
> *"GROUP BY collapses rows into groups and returns one row per group. Window functions compute across a set of rows related to the current row but preserve every row in the output. This means I can show per-row detail alongside aggregate calculations — like showing each order's amount next to the customer's average order value — without self-joining or subquerying."*

---

## Screen 2: ROW_NUMBER, RANK, and DENSE_RANK

These three ranking functions are the **bread and butter** of SQL interviews. They all assign numbers to rows within a partition, but they handle ties differently.

### Setup: Our E-Commerce Data

```sql
-- Sample data: product sales
SELECT product_id, category, revenue
FROM product_sales;
```

| product_id | category    | revenue |
|:-----------|:------------|--------:|
| 101        | Electronics | 5000    |
| 102        | Electronics | 5000    |
| 103        | Electronics | 3000    |
| 201        | Clothing    | 4000    |
| 202        | Clothing    | 4000    |
| 203        | Clothing    | 2000    |

### The Three Functions Compared

```sql
SELECT product_id, category, revenue,
       ROW_NUMBER() OVER (PARTITION BY category ORDER BY revenue DESC) AS row_num,
       RANK()       OVER (PARTITION BY category ORDER BY revenue DESC) AS rank_val,
       DENSE_RANK() OVER (PARTITION BY category ORDER BY revenue DESC) AS dense_rank_val
FROM product_sales;
```

| product_id | category    | revenue | row_num | rank_val | dense_rank_val |
|:-----------|:------------|--------:|--------:|---------:|---------------:|
| 101        | Electronics | 5000    | 1       | 1        | 1              |
| 102        | Electronics | 5000    | 2       | 1        | 1              |
| 103        | Electronics | 3000    | 3       | 3        | 2              |
| 201        | Clothing    | 4000    | 1       | 1        | 1              |
| 202        | Clothing    | 4000    | 2       | 1        | 1              |
| 203        | Clothing    | 2000    | 3       | 3        | 2              |

### The Analogy: A Race Finish

- **ROW_NUMBER()** → The **bib number** at the finish line. Even if two runners cross at the exact same time, one gets bib #1 and the other gets bib #2. **No ties, ever.** Non-deterministic for ties (database picks arbitrarily).
- **RANK()** → The **Olympic medal** system. Two runners tie for gold? They both get rank 1, but the next runner gets rank 3 (no silver). **Gaps appear after ties.**
- **DENSE_RANK()** → The **podium step** system. Two runners tie for 1st? They share step 1, and the next runner gets step 2. **No gaps, ever.**

```
Values:   100  100  90  80  80  70
           │    │   │   │    │   │
ROW_NUM:   1    2   3   4    5   6   ← Always sequential, no gaps, no ties
RANK:      1    1   3   4    4   6   ← Ties share rank, gaps after ties
DENSE_RANK:1    1   2   3    3   4   ← Ties share rank, NO gaps
```

### When to Use Each

| Use Case                                      | Function         |
|:----------------------------------------------|:-----------------|
| **"Top 1 per group"** (deduplication)          | `ROW_NUMBER()`   |
| **"Top N with competition ranking"**           | `RANK()`         |
| **"How many distinct ranks exist?"**           | `DENSE_RANK()`   |
| **Pagination** (OFFSET alternative)            | `ROW_NUMBER()`   |

### Classic Interview Pattern: Top-N Per Group

*"Find the top 2 highest-revenue products per category."*

```sql
WITH ranked AS (
    SELECT product_id, category, revenue,
           ROW_NUMBER() OVER (PARTITION BY category ORDER BY revenue DESC) AS rn
    FROM product_sales
)
SELECT product_id, category, revenue
FROM ranked
WHERE rn <= 2;
```

> **Why ROW_NUMBER here and not RANK?** If two products tie for #2, RANK would return both (giving you 3 results). ROW_NUMBER guarantees exactly 2 rows per category. The choice depends on the business requirement — ask your interviewer!

### 💡 Interview Insight

> **"When would you choose DENSE_RANK over RANK?"**
>
> *"I'd use DENSE_RANK when I care about the number of distinct ranking levels rather than positions. For example, if I'm asked 'find products with the 3rd highest revenue,' DENSE_RANK ensures I actually find that 3rd tier even when ties push RANK's values past 3. With RANK, if two products tie for 1st, the 'next' rank is 3 — there is no rank 2 — so a WHERE rank = 3 might accidentally grab the second-highest tier."*

---

## Screen 3: LEAD, LAG, and Gap Analysis

### Looking Backward and Forward

`LAG()` and `LEAD()` let each row reach out and grab values from **adjacent rows** in the ordered partition. Think of it like standing in a line and glancing at the person ahead of you or behind you.

```sql
LAG(column, offset, default)  OVER (PARTITION BY ... ORDER BY ...)
LEAD(column, offset, default) OVER (PARTITION BY ... ORDER BY ...)
```

- `offset` — how many rows back (LAG) or forward (LEAD). Default is 1.
- `default` — value to return when there's no row to look at (first/last row). Default is NULL.

### Example: Month-over-Month Revenue Change

```sql
SELECT 
    month,
    revenue,
    LAG(revenue, 1) OVER (ORDER BY month)  AS prev_month_revenue,
    revenue - LAG(revenue, 1) OVER (ORDER BY month) AS mom_change,
    ROUND(
        100.0 * (revenue - LAG(revenue, 1) OVER (ORDER BY month)) 
        / LAG(revenue, 1) OVER (ORDER BY month), 2
    ) AS mom_pct_change
FROM monthly_revenue;
```

| month   | revenue | prev_month_revenue | mom_change | mom_pct_change |
|:--------|--------:|-------------------:|-----------:|---------------:|
| 2024-01 | 10000   | NULL               | NULL       | NULL           |
| 2024-02 | 12000   | 10000              | 2000       | 20.00          |
| 2024-03 | 11500   | 12000              | -500       | -4.17          |
| 2024-04 | 15000   | 11500              | 3500       | 30.43          |

### Year-over-Year (YoY) Comparison

For YoY, use an offset of 12 (if monthly data) or self-join on year. With LAG:

```sql
SELECT 
    month,
    revenue,
    LAG(revenue, 12) OVER (ORDER BY month) AS same_month_last_year,
    ROUND(
        100.0 * (revenue - LAG(revenue, 12) OVER (ORDER BY month)) 
        / NULLIF(LAG(revenue, 12) OVER (ORDER BY month), 0), 2
    ) AS yoy_pct_change
FROM monthly_revenue;
```

> **Pro tip**: Always wrap the denominator in `NULLIF(..., 0)` to avoid division-by-zero errors. Interviewers love to see that defensive coding habit.

### Gap Analysis: Finding Missing Sequences

*"Find gaps in order IDs (missing order numbers)."*

```sql
SELECT 
    order_id,
    LEAD(order_id) OVER (ORDER BY order_id) AS next_order_id,
    LEAD(order_id) OVER (ORDER BY order_id) - order_id AS gap_size
FROM orders
WHERE LEAD(order_id) OVER (ORDER BY order_id) - order_id > 1;
```

Wait — that won't work! You **can't use window functions in WHERE**. This is a common mistake. Fix:

```sql
WITH order_gaps AS (
    SELECT 
        order_id,
        LEAD(order_id) OVER (ORDER BY order_id) AS next_order_id
    FROM orders
)
SELECT order_id, next_order_id, (next_order_id - order_id) AS gap_size
FROM order_gaps
WHERE next_order_id - order_id > 1;
```

### 💡 Interview Insight

> **"Can you use a window function in a WHERE clause?"**
>
> *"No. Window functions are evaluated after WHERE, HAVING, and GROUP BY — they're in the same phase as SELECT. To filter on a window function result, I'd wrap the query in a CTE or subquery and filter in the outer query. The SQL logical processing order is: FROM → WHERE → GROUP BY → HAVING → SELECT (window functions here) → ORDER BY → LIMIT."*

---

## Screen 4: FIRST_VALUE, LAST_VALUE, NTH_VALUE — Frame-Aware Functions

### The Trap: Default Frame Specification

These functions return a specific row's value from the window frame. But here's where most people get burned — **the default frame**.

When `ORDER BY` is specified, the default frame is:

```
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
```

This means `LAST_VALUE()` doesn't give you the last value in the partition — it gives you the **current row** (because the frame ends at the current row). This is the single most common window function bug.

```sql
-- BUGGY: LAST_VALUE sees only up to current row
SELECT 
    order_id, customer_id, order_date, amount,
    FIRST_VALUE(amount) OVER (PARTITION BY customer_id ORDER BY order_date) AS first_order_amt,
    LAST_VALUE(amount)  OVER (PARTITION BY customer_id ORDER BY order_date) AS last_order_amt  -- BUG!
FROM orders;
```

```sql
-- FIXED: Extend the frame to the entire partition
SELECT 
    order_id, customer_id, order_date, amount,
    FIRST_VALUE(amount) OVER (
        PARTITION BY customer_id ORDER BY order_date
    ) AS first_order_amt,
    LAST_VALUE(amount) OVER (
        PARTITION BY customer_id ORDER BY order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS last_order_amt
FROM orders;
```

### NTH_VALUE — Get the Nth Row

```sql
-- Get the 2nd order amount for each customer
SELECT 
    order_id, customer_id, order_date, amount,
    NTH_VALUE(amount, 2) OVER (
        PARTITION BY customer_id 
        ORDER BY order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS second_order_amt
FROM orders;
```

`NTH_VALUE` returns NULL if the partition has fewer than N rows. It also requires the same frame fix as `LAST_VALUE` if you want to look "ahead" of the current row.

### Comparison Table

| Function        | Returns                        | Frame Matters? | Common Gotcha          |
|:----------------|:-------------------------------|:---------------|:-----------------------|
| `FIRST_VALUE()` | Value from first row in frame  | Less so*       | Usually works with default |
| `LAST_VALUE()`  | Value from last row in frame   | **YES**        | Must extend frame!     |
| `NTH_VALUE(n)`  | Value from nth row in frame    | **YES**        | Returns NULL if < n rows |

*`FIRST_VALUE` usually works because the default frame starts at `UNBOUNDED PRECEDING`, which includes the first row.

### Practical Example: First and Last Purchase per Customer

```sql
SELECT DISTINCT
    customer_id,
    FIRST_VALUE(product_name) OVER w AS first_product_bought,
    LAST_VALUE(product_name)  OVER w AS latest_product_bought,
    FIRST_VALUE(order_date)   OVER w AS first_purchase_date,
    LAST_VALUE(order_date)    OVER w AS latest_purchase_date
FROM orders
JOIN products USING (product_id)
WINDOW w AS (
    PARTITION BY customer_id 
    ORDER BY order_date
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
);
```

> **Pro tip**: Use the `WINDOW` clause to define a named window and reuse it. This avoids repeating the same `OVER(...)` specification five times — cleaner and less error-prone.

### 💡 Interview Insight

> **"What's the default frame specification when ORDER BY is present in a window function?"**
>
> *"When ORDER BY is present, the default frame is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. This means aggregate window functions compute a running total by default, and `LAST_VALUE()` will just return the current row's value — not the actual last row in the partition. To fix this, I explicitly specify `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`."*

---

## Screen 5: Frame Specification — ROWS vs RANGE vs GROUPS

### Why Frames Matter

The frame clause defines **exactly which rows** within a partition the window function considers. This is the most nuanced part of window functions and the part that separates good SQL engineers from great ones.

```
ROWS | RANGE | GROUPS BETWEEN
    { UNBOUNDED PRECEDING | n PRECEDING | CURRENT ROW }
    AND
    { UNBOUNDED FOLLOWING | n FOLLOWING | CURRENT ROW }
```

### ROWS — Physical Row Count

`ROWS` counts **physical rows** regardless of values. Simple, predictable.

```sql
-- 3-day moving average (exactly 3 rows)
SELECT order_date, daily_revenue,
    AVG(daily_revenue) OVER (
        ORDER BY order_date
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) AS moving_avg_3day
FROM daily_sales;
```

```
Row 1: AVG of [Row 1]                         = Row 1 value
Row 2: AVG of [Row 1, Row 2]                  = average of 2
Row 3: AVG of [Row 1, Row 2, Row 3]           = average of 3  ← full window now
Row 4: AVG of [Row 2, Row 3, Row 4]           = average of 3  ← slides forward
```

### RANGE — Logical Value Range

`RANGE` includes all rows whose `ORDER BY` value falls within a **logical range** of the current row's value. This matters when you have **duplicate values** or need **time-based windows**.

```sql
-- Include all rows with revenue within ±100 of current row's value
SELECT order_id, revenue,
    COUNT(*) OVER (
        ORDER BY revenue
        RANGE BETWEEN 100 PRECEDING AND 100 FOLLOWING
    ) AS nearby_count
FROM orders;
```

With `RANGE`, if multiple rows have the same `ORDER BY` value, they're all treated as peers and included together.

### GROUPS — Count of Distinct Peer Groups

`GROUPS` is the newest (SQL:2011) and least commonly used. It counts **distinct groups of tied values**.

```sql
-- Include current group plus 1 group before and after
SELECT order_date, revenue,
    SUM(revenue) OVER (
        ORDER BY order_date
        GROUPS BETWEEN 1 PRECEDING AND 1 FOLLOWING
    ) AS adjacent_groups_sum
FROM daily_sales;
```

### Side-by-Side Comparison

```
Data (ORDER BY value):  10, 20, 20, 20, 30, 40

Current row: 20 (second occurrence), frame = "1 PRECEDING AND 1 FOLLOWING"

ROWS:   [20, 20, 20]       ← 1 physical row before + current + 1 physical row after
RANGE:  [10, 20, 20, 20, 30]  ← all values within 20±1 = 19..21? No! 
        Actually RANGE with integers: 1 PRECEDING = value >= 19, 1 FOLLOWING = value <= 21
        So: [20, 20, 20]   ← only values 19-21
GROUPS: [10, 20, 20, 20, 30]  ← 1 peer group before ({10}) + current group ({20,20,20}) + 1 group after ({30})
```

### When to Use Each

| Frame Type | Best For                               | Handles Ties  |
|:-----------|:---------------------------------------|:--------------|
| `ROWS`     | Fixed-size sliding windows (top-N)     | Ignores ties  |
| `RANGE`    | Value-based ranges, date intervals     | Groups peers  |
| `GROUPS`   | Peer-group-based windows               | Groups peers  |

### 💡 Interview Insight

> **"What's the difference between ROWS and RANGE in a window frame?"**
>
> *"ROWS counts physical rows — `2 PRECEDING` means exactly 2 rows before the current one. RANGE operates on logical values — `2 PRECEDING` means all rows whose ORDER BY value is within 2 of the current row's value. The key difference shows up with duplicate ORDER BY values: RANGE treats all peers as a single unit, while ROWS treats each as individual. For moving averages with potential date gaps, I'd use RANGE with date intervals; for fixed-size sliding windows, I'd use ROWS."*

---

## Screen 6: Running Aggregates — SUM, AVG, COUNT OVER

### Running Totals

The most common analytics interview question: *"Calculate a running total of revenue by date."*

```sql
SELECT 
    order_date,
    daily_revenue,
    SUM(daily_revenue) OVER (ORDER BY order_date) AS running_total,
    SUM(daily_revenue) OVER (
        ORDER BY order_date 
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total_explicit  -- Same thing, just explicit
FROM daily_sales;
```

| order_date | daily_revenue | running_total | running_total_explicit |
|:-----------|:-------------|:-------------|:----------------------|
| 2024-01-01 | 1000          | 1000          | 1000                  |
| 2024-01-02 | 1500          | 2500          | 2500                  |
| 2024-01-03 | 800           | 3300          | 3300                  |
| 2024-01-04 | 2000          | 5300          | 5300                  |

### Running Total Per Partition (Per Customer)

```sql
SELECT 
    customer_id, order_date, amount,
    SUM(amount) OVER (
        PARTITION BY customer_id 
        ORDER BY order_date
    ) AS customer_running_total
FROM orders
ORDER BY customer_id, order_date;
```

### Moving Averages

```sql
-- 7-day moving average
SELECT 
    order_date,
    daily_revenue,
    ROUND(AVG(daily_revenue) OVER (
        ORDER BY order_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ), 2) AS moving_avg_7day,
    -- Min and max in the same window
    MIN(daily_revenue) OVER (
        ORDER BY order_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS rolling_min_7day,
    MAX(daily_revenue) OVER (
        ORDER BY order_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS rolling_max_7day
FROM daily_sales;
```

### Running Count and Running Distinct (Workaround)

```sql
-- Running count of orders per customer
SELECT 
    customer_id, order_date, order_id,
    COUNT(*) OVER (
        PARTITION BY customer_id ORDER BY order_date
    ) AS order_number  -- "This is their Nth order"
FROM orders;
```

> **Note**: There's no `COUNT(DISTINCT ...) OVER(...)` in standard SQL. To get a running distinct count, you'd use `DENSE_RANK()` as a clever workaround or resort to a self-join.

### Cumulative Percentage

*"What percentage of total revenue has been earned by each date?"*

```sql
SELECT 
    order_date,
    daily_revenue,
    SUM(daily_revenue) OVER (ORDER BY order_date) AS running_total,
    ROUND(
        100.0 * SUM(daily_revenue) OVER (ORDER BY order_date)
        / SUM(daily_revenue) OVER (), -- No ORDER BY = entire partition total
        2
    ) AS cumulative_pct
FROM daily_sales;
```

The trick here is `SUM(...) OVER ()` with an **empty** OVER clause — this gives the grand total across the entire result set, perfect for computing percentages.

### 💡 Interview Insight

> **"How do you compute a running total that resets each month?"**
>
> *"I'd partition by the month (or year-month) and order by date within each partition. The running total automatically resets at each partition boundary:*
> ```sql
> SUM(amount) OVER (
>     PARTITION BY DATE_TRUNC('month', order_date)
>     ORDER BY order_date
> )
> ```
> *This works because each month becomes its own partition, and running totals are computed independently within each one."*

---

## Screen 7: Statistical Window Functions — PERCENT_RANK, CUME_DIST, NTILE

### PERCENT_RANK — Relative Standing

`PERCENT_RANK()` returns the relative rank as a percentage: `(rank - 1) / (total_rows - 1)`. The first row is always 0, the last is always 1.

```sql
SELECT 
    customer_id, 
    total_spend,
    PERCENT_RANK() OVER (ORDER BY total_spend) AS pct_rank
FROM customer_lifetime_value;
```

| customer_id | total_spend | pct_rank |
|:------------|:-----------|:---------|
| C001        | 500         | 0.00     |
| C002        | 1200        | 0.25     |
| C003        | 3000        | 0.50     |
| C004        | 5500        | 0.75     |
| C005        | 12000       | 1.00     |

### CUME_DIST — Cumulative Distribution

`CUME_DIST()` returns the fraction of rows with values **less than or equal to** the current row: `count(values <= current) / total_rows`. Always > 0, the last row is always 1.

```sql
SELECT 
    customer_id,
    total_spend,
    ROUND(CUME_DIST() OVER (ORDER BY total_spend), 4) AS cume_dist
FROM customer_lifetime_value;
```

| customer_id | total_spend | cume_dist |
|:------------|:-----------|:----------|
| C001        | 500         | 0.2000    |
| C002        | 1200        | 0.4000    |
| C003        | 3000        | 0.6000    |
| C004        | 5500        | 0.8000    |
| C005        | 12000       | 1.0000    |

### Key Difference: PERCENT_RANK vs CUME_DIST

| Aspect         | PERCENT_RANK              | CUME_DIST                     |
|:---------------|:--------------------------|:------------------------------|
| Formula        | `(rank-1)/(n-1)`          | `count(≤ current)/n`          |
| Min value      | 0                         | `1/n` (never 0)              |
| Max value      | 1                         | 1                             |
| Interpretation | "% of rows ranked lower"  | "% of rows ≤ this value"      |

### NTILE — Divide Into Buckets

`NTILE(n)` divides the ordered partition into `n` roughly equal buckets and assigns each row a bucket number (1 to n). Perfect for percentiles and segments.

```sql
-- Divide customers into 4 spending quartiles
SELECT 
    customer_id,
    total_spend,
    NTILE(4) OVER (ORDER BY total_spend) AS spend_quartile
FROM customer_lifetime_value;
```

```sql
-- Practical: segment customers for marketing
SELECT 
    customer_id,
    total_spend,
    CASE NTILE(4) OVER (ORDER BY total_spend DESC)
        WHEN 1 THEN 'VIP (Top 25%)'
        WHEN 2 THEN 'High Value'
        WHEN 3 THEN 'Medium Value'
        WHEN 4 THEN 'Low Value'
    END AS customer_segment
FROM customer_lifetime_value;
```

> **Caveat**: `NTILE` distributes rows as evenly as possible, but if 10 rows are split into 4 buckets, you get sizes 3, 3, 2, 2 — not perfectly equal. For precise percentile calculations, prefer `PERCENT_RANK` or `CUME_DIST`.

### 💡 Interview Insight

> **"How would you find the median using window functions?"**
>
> *"I'd use `PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY value)` if available (PostgreSQL, SQL Server). Alternatively, I can use NTILE or ROW_NUMBER:*
> ```sql
> WITH ranked AS (
>     SELECT value,
>            ROW_NUMBER() OVER (ORDER BY value) AS rn,
>            COUNT(*) OVER () AS total
>     FROM my_table
> )
> SELECT AVG(value) AS median
> FROM ranked
> WHERE rn IN (FLOOR((total+1)/2.0), CEIL((total+1)/2.0));
> ```
> *This handles both odd and even row counts by averaging the two middle values."*

---

## Screen 8: Practical — Sessionization with Window Functions

### The Problem

You have a `user_events` table with timestamped page views. A "session" is defined as a sequence of events where no gap exceeds 30 minutes. Assign a session ID to each event.

This is a **classic interview question** at companies like Google, Meta, and Amazon.

### Step-by-Step Approach

```sql
-- Step 1: Calculate the gap from the previous event
WITH event_gaps AS (
    SELECT 
        user_id,
        event_time,
        page_url,
        LAG(event_time) OVER (
            PARTITION BY user_id ORDER BY event_time
        ) AS prev_event_time,
        EXTRACT(EPOCH FROM (
            event_time - LAG(event_time) OVER (
                PARTITION BY user_id ORDER BY event_time
            )
        )) / 60 AS gap_minutes
    FROM user_events
),

-- Step 2: Flag session boundaries (gap > 30 min or first event)
session_boundaries AS (
    SELECT *,
        CASE 
            WHEN gap_minutes > 30 OR gap_minutes IS NULL THEN 1
            ELSE 0
        END AS is_new_session
    FROM event_gaps
),

-- Step 3: Assign session IDs using a running sum of boundary flags
sessions AS (
    SELECT *,
        SUM(is_new_session) OVER (
            PARTITION BY user_id 
            ORDER BY event_time
        ) AS session_id
    FROM session_boundaries
)

SELECT user_id, event_time, page_url, session_id
FROM sessions
ORDER BY user_id, event_time;
```

### How It Works — Visually

```
User A's events:

Time:        10:00   10:05   10:20   11:15   11:18   11:25
Gap (min):    NULL     5       15      55       3       7
New session?  1        0       0       1        0       0
SUM flags:    1        1       1       2        2       2
              └── Session 1 ──┘       └── Session 2 ──┘
```

The trick is brilliant: by converting boolean flags (0/1) into a running sum, each new session increments the counter, and all events within that session share the same sum value.

### Session-Level Aggregates

Once sessionized, you can compute session-level metrics:

```sql
-- Session duration and page count
SELECT 
    user_id,
    session_id,
    MIN(event_time) AS session_start,
    MAX(event_time) AS session_end,
    EXTRACT(EPOCH FROM MAX(event_time) - MIN(event_time)) / 60 AS session_duration_min,
    COUNT(*) AS pages_viewed,
    FIRST_VALUE(page_url) OVER (
        PARTITION BY user_id, session_id ORDER BY event_time
    ) AS landing_page,
    LAST_VALUE(page_url) OVER (
        PARTITION BY user_id, session_id ORDER BY event_time
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS exit_page
FROM sessions
GROUP BY user_id, session_id;
```

### 💡 Interview Insight

> **"Walk me through how you'd sessionize clickstream data."**
>
> *"I'd use a three-step window function approach: First, use LAG to compute the time gap between consecutive events per user. Second, flag rows where the gap exceeds my session threshold (e.g., 30 minutes) or where there's no previous event. Third, take a running SUM of those flags — this creates a monotonically increasing session counter that increments at each boundary. The result is a deterministic session ID per user. This is O(n log n) for the sort and O(n) for the scan — much more efficient than self-joins."*

---

## Screen 9: Quiz

**Q1: What does ROW_NUMBER() do when two rows have the same ORDER BY value?**
- A) Returns the same number for both rows
- B) Returns NULL for the duplicate
- C) Assigns arbitrary but distinct numbers to each row ✅
- D) Throws an error

> **Answer: C** — ROW_NUMBER() never produces ties. When ORDER BY values are identical, it arbitrarily (non-deterministically) assigns distinct sequential numbers. To make it deterministic, add a tiebreaker column to the ORDER BY clause.

**Q2: What is the default frame when ORDER BY is present in a window function?**
- A) `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`
- B) `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` ✅
- C) `ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING`
- D) `RANGE BETWEEN 1 PRECEDING AND 1 FOLLOWING`

> **Answer: B** — The default is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. This is why `LAST_VALUE()` with a default frame returns the current row's value instead of the partition's last value — a classic gotcha.

**Q3: You want to find the top 3 products by revenue in each category, but if products tie, you want ALL tied products included. Which function should you use?**
- A) `ROW_NUMBER()` — filter `WHERE rn <= 3`
- B) `RANK()` — filter `WHERE rank <= 3` ✅
- C) `DENSE_RANK()` — filter `WHERE dense_rank <= 3`
- D) `NTILE(3)` — filter `WHERE ntile = 1`

> **Answer: B** — RANK() allows ties and includes all tied rows at the same rank. If 2 products tie for rank 3, both are included. DENSE_RANK with `<= 3` would include up to 3 distinct revenue *tiers*, which could return far more rows. ROW_NUMBER forces exactly 3 per category regardless of ties.

**Q4: How do you calculate a 7-day moving average of daily revenue?**
- A) `AVG(revenue) OVER (ORDER BY date ROWS BETWEEN 7 PRECEDING AND CURRENT ROW)`
- B) `AVG(revenue) OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)` ✅
- C) `AVG(revenue) OVER (ORDER BY date RANGE BETWEEN 7 PRECEDING AND CURRENT ROW)`
- D) `AVG(revenue) OVER (PARTITION BY date ROWS BETWEEN 7 PRECEDING AND CURRENT ROW)`

> **Answer: B** — A 7-day window includes the current row + 6 preceding rows = 7 rows total. Using `7 PRECEDING` would give 8 rows. Option C could also work for date ranges (7 days before), but `ROWS` is specified when you mean exactly 7 physical rows. Option D is wrong because PARTITION BY date would make each date its own partition with one row.

**Q5: In the sessionization pattern, what technique creates the session IDs?**
- A) `ROW_NUMBER()` partitioned by user
- B) `DENSE_RANK()` on the timestamp
- C) Running `SUM()` of a boolean flag indicating session boundaries ✅
- D) `NTILE()` to divide events into buckets

> **Answer: C** — The sessionization pattern uses LAG to compute gaps, creates a 0/1 flag for new sessions, then takes a running SUM of those flags. Each new session increments the sum, creating a natural session counter. This is elegant because it requires only window functions — no self-joins or procedural code.

---

## Screen 10: Key Takeaways

- **ROW_NUMBER** gives unique sequential numbers (no ties), **RANK** allows ties with gaps, **DENSE_RANK** allows ties without gaps. Choose based on whether you want exactly N results or inclusive-of-ties results.

- **LAG/LEAD** let you access adjacent rows — essential for period-over-period comparisons, gap analysis, and detecting changes. Always handle NULLs for the first/last rows.

- **LAST_VALUE is broken by default** — the default frame `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` means it just returns the current row's value. Always specify `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` when you want the actual last value.

- **ROWS counts physical rows, RANGE uses logical values, GROUPS counts peer groups**. Use ROWS for fixed-size sliding windows, RANGE for value-based ranges, and GROUPS when you need to operate on distinct tiers.

- **Window functions execute in the SELECT phase** — after WHERE, GROUP BY, and HAVING. To filter on a window function result, wrap it in a CTE or subquery.

- **The sessionization pattern** (LAG → flag boundaries → running SUM) is a foundational technique. It applies to any problem where you need to group consecutive rows by detecting breaks: session analysis, island problems, streak counting.

- **`SUM(...) OVER ()` with an empty OVER clause** gives the grand total — invaluable for computing percentages of total without a self-join or additional GROUP BY.
