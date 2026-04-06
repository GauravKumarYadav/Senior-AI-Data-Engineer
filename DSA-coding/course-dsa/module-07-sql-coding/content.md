# Module 7: SQL Coding Problems

> **Goal:** Work through 12 progressively harder SQL problems that cover every major pattern you'll face in data engineering and analytics interviews. Each problem includes sample data, a complete solution, a line-by-line walkthrough, and an interview tip.

---

## How to Use This Module

1. **Read the problem statement.** Try to solve it yourself for 10–15 minutes before looking at the solution.
2. **Understand the pattern.** Each problem maps to a reusable SQL technique — learn the *pattern*, not just the query.
3. **Practice on a real engine.** Spin up a SQLite, PostgreSQL, or BigQuery sandbox and run these queries against actual data.
4. **All SQL is written in ANSI-standard syntax** with notes where BigQuery, PostgreSQL, or MySQL differ.

---

## Problem 1: Top 3 Salaries per Department

### Problem Statement

Given an `employees` table, find the **top 3 distinct salaries** in each department. If a department has fewer than 3 distinct salaries, show all of them.

| emp_id | name    | department | salary |
|--------|---------|------------|--------|
| 1      | Alice   | Eng        | 120000 |
| 2      | Bob     | Eng        | 120000 |
| 3      | Carol   | Eng        | 100000 |
| 4      | Dave    | Eng        | 90000  |
| 5      | Eve     | Sales      | 80000  |
| 6      | Frank   | Sales      | 75000  |

### Approach

Use `DENSE_RANK()` (not `RANK()` or `ROW_NUMBER()`) because the problem asks for top 3 **distinct** salaries. `DENSE_RANK` assigns the same rank to ties and doesn't skip numbers, so filtering `<= 3` gives exactly what we need.

### Solution

```sql
WITH ranked AS (
    SELECT
        emp_id,
        name,
        department,
        salary,
        DENSE_RANK() OVER (
            PARTITION BY department
            ORDER BY salary DESC
        ) AS dr
    FROM employees
)
SELECT emp_id, name, department, salary
FROM ranked
WHERE dr <= 3
ORDER BY department, salary DESC;
```

### Explanation

1. **CTE `ranked`** — For each row, compute `DENSE_RANK` partitioned by department, ordered by salary descending. Alice and Bob both get `dr = 1` (tied salary), Carol gets `dr = 2`, Dave gets `dr = 3`.
2. **Filter `dr <= 3`** — Keeps only the top 3 distinct salary tiers.
3. **ORDER BY** — Clean output sorted by department then salary.

> **Interview Tip:** If the interviewer says "top 3 employees" (not distinct salaries), switch to `ROW_NUMBER()`. If they say "top 3 salaries allowing ties," use `RANK()`. Clarify before you code.

---

## Problem 2: Employees Earning More Than Their Manager

### Problem Statement

Find all employees who earn more than their direct manager.

| emp_id | name    | salary | manager_id |
|--------|---------|--------|------------|
| 1      | Alice   | 150000 | NULL       |
| 2      | Bob     | 120000 | 1          |
| 3      | Carol   | 160000 | 1          |
| 4      | Dave    | 130000 | 2          |

**Expected:** Carol (earns 160 k, manager Alice earns 150 k), Dave (earns 130 k, manager Bob earns 120 k).

### Approach

Classic **self-join** — join the table to itself on `emp.manager_id = mgr.emp_id`, then filter where the employee's salary exceeds the manager's.

### Solution

```sql
SELECT
    e.name  AS employee,
    e.salary AS employee_salary,
    m.name  AS manager,
    m.salary AS manager_salary
FROM employees e
JOIN employees m ON e.manager_id = m.emp_id
WHERE e.salary > m.salary;
```

### Explanation

1. **Self-join:** We alias the same table as `e` (employee) and `m` (manager).
2. **Join condition:** Links each employee to their manager row via `manager_id`.
3. **WHERE clause:** Filters to only those whose salary exceeds their manager's.

> **Interview Tip:** This is often the first SQL question in a screen. Solve it in under 2 minutes to set a strong tone. Note that `INNER JOIN` naturally excludes employees with `NULL` manager_id (i.e., the CEO).

---

## Problem 3: Customers Who Never Ordered

### Problem Statement

Given `customers` and `orders` tables, find customers who have never placed an order.

| customer_id | name   |       | order_id | customer_id |
|-------------|--------|-------|----------|-------------|
| 1           | Alice  |       | 101      | 1           |
| 2           | Bob    |       | 102      | 1           |
| 3           | Carol  |       | 103      | 3           |
| 4           | Dave   |       |          |             |

**Expected:** Bob (id 2), Dave (id 4).

### Approach

Two valid approaches: **LEFT JOIN + IS NULL** or **NOT EXISTS**. Both are anti-join patterns.

### Solution

```sql
-- Approach 1: LEFT JOIN + IS NULL
SELECT c.customer_id, c.name
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_id IS NULL;

-- Approach 2: NOT EXISTS (often preferred by optimizers)
SELECT c.customer_id, c.name
FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o
    WHERE o.customer_id = c.customer_id
);
```

### Explanation

**Approach 1:** The LEFT JOIN keeps all customers. Where there's no matching order, all order columns are NULL. Filtering `WHERE o.order_id IS NULL` isolates the "no-match" rows.

**Approach 2:** `NOT EXISTS` is a correlated subquery that returns TRUE when the subquery finds zero rows for that customer. Most modern engines (BigQuery, PostgreSQL) optimize both identically.

> **Interview Tip:** Avoid `NOT IN` — it breaks silently when the subquery contains NULLs (the entire `NOT IN` returns no rows). `NOT EXISTS` is always safe.

---

## Problem 4: Running Total of Sales

### Problem Statement

Compute a **running total** of daily sales, ordered by date.

| sale_date  | amount |
|------------|--------|
| 2025-01-01 | 100    |
| 2025-01-02 | 250    |
| 2025-01-03 | 175    |
| 2025-01-04 | 300    |

**Expected:** 100, 350, 525, 825.

### Approach

Window function `SUM() OVER (ORDER BY ... ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)`.

### Solution

```sql
SELECT
    sale_date,
    amount,
    SUM(amount) OVER (
        ORDER BY sale_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total
FROM daily_sales
ORDER BY sale_date;
```

### Explanation

1. **`SUM(amount) OVER (...)`** — Applies a window sum (no `PARTITION BY` since we want a global running total).
2. **`ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`** — Explicitly defines the frame: from the first row up to the current row.
3. Why not just `SUM(amount) OVER (ORDER BY sale_date)`? That *works* in most engines, but the default frame with `ORDER BY` is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`, which groups ties together. `ROWS` is deterministic.

> **Interview Tip:** Always specify the frame explicitly. It shows you understand the subtle difference between `ROWS` and `RANGE` — a common gotcha that trips up even experienced engineers.

---

## Problem 5: Month-over-Month Revenue Growth

### Problem Statement

Calculate the **month-over-month percentage change** in revenue.

| month   | revenue |
|---------|---------|
| 2025-01 | 10000   |
| 2025-02 | 12000   |
| 2025-03 | 11500   |
| 2025-04 | 13000   |

**Expected:** Jan = NULL, Feb = +20.0 %, Mar = −4.2 %, Apr = +13.0 %.

### Approach

`LAG()` to access the previous month's revenue, then simple percentage math.

### Solution

```sql
SELECT
    month,
    revenue,
    LAG(revenue) OVER (ORDER BY month) AS prev_revenue,
    ROUND(
        100.0 * (revenue - LAG(revenue) OVER (ORDER BY month))
        / NULLIF(LAG(revenue) OVER (ORDER BY month), 0),
        1
    ) AS mom_pct_change
FROM monthly_revenue
ORDER BY month;
```

### Explanation

1. **`LAG(revenue) OVER (ORDER BY month)`** — Fetches the previous row's revenue.
2. **`NULLIF(..., 0)`** — Prevents division-by-zero if the previous month had zero revenue.
3. **`ROUND(..., 1)`** — Clean output to one decimal place.
4. The first row naturally returns NULL since there's no previous month.

> **Interview Tip:** Mention `NULLIF` unprompted — it shows defensive coding habits. Also note that `LEAD()` works the same way but looks forward instead of backward.

---

## Problem 6: Pivot Sales by Quarter

### Problem Statement

Show total revenue per product, with one column per quarter.

| product | quarter | revenue |
|---------|---------|---------|
| Widget  | Q1      | 1000    |
| Widget  | Q2      | 1500    |
| Gadget  | Q1      | 800     |
| Gadget  | Q3      | 1200    |

**Expected output:**

| product | Q1   | Q2   | Q3   | Q4   |
|---------|------|------|------|------|
| Widget  | 1000 | 1500 | 0    | 0    |
| Gadget  | 800  | 0    | 1200 | 0    |

### Approach

Use **conditional aggregation**: `SUM(CASE WHEN ... THEN ... ELSE 0 END)` for each quarter. This is the most portable approach (works in every SQL engine).

### Solution

```sql
SELECT
    product,
    SUM(CASE WHEN quarter = 'Q1' THEN revenue ELSE 0 END) AS Q1,
    SUM(CASE WHEN quarter = 'Q2' THEN revenue ELSE 0 END) AS Q2,
    SUM(CASE WHEN quarter = 'Q3' THEN revenue ELSE 0 END) AS Q3,
    SUM(CASE WHEN quarter = 'Q4' THEN revenue ELSE 0 END) AS Q4
FROM sales
GROUP BY product
ORDER BY product;
```

### Explanation

1. **`CASE WHEN quarter = 'Q1' THEN revenue ELSE 0 END`** — Zeroes out non-Q1 rows so `SUM` only totals Q1 revenue.
2. **One expression per pivot column** — Repeat for each quarter.
3. **`GROUP BY product`** — Collapses all rows for a product into one.

> **Interview Tip:** If the interviewer asks *"what if we don't know the quarters in advance?"* — that's dynamic pivot. In PostgreSQL you'd use `crosstab()`, in BigQuery you'd use a scripting block, in application code you'd build the SQL string dynamically. There's no clean ANSI-standard way.

---

## Problem 7: Dedup Records — Keep Latest

### Problem Statement

A `user_profiles` table has duplicate entries per user due to CDC (change data capture). Keep only the **most recent** record per user.

| user_id | email             | updated_at          |
|---------|-------------------|---------------------|
| 1       | alice@v1.com      | 2025-01-01 08:00:00 |
| 1       | alice@v2.com      | 2025-03-15 12:00:00 |
| 2       | bob@company.com   | 2025-02-01 09:00:00 |
| 2       | bob@newjob.com    | 2025-02-20 14:00:00 |

### Approach

`ROW_NUMBER()` partitioned by `user_id`, ordered by `updated_at DESC`. Keep `rn = 1`.

### Solution

```sql
WITH numbered AS (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY user_id
            ORDER BY updated_at DESC
        ) AS rn
    FROM user_profiles
)
SELECT user_id, email, updated_at
FROM numbered
WHERE rn = 1;
```

### Explanation

1. **`ROW_NUMBER()`** assigns 1 to the most recent row per user (due to `DESC` ordering).
2. **Filter `rn = 1`** keeps exactly one row per user — the latest.
3. This is strictly better than `GROUP BY` + `MAX(updated_at)` because it returns ALL columns, not just the grouped ones.

> **Interview Tip:** If building a production pipeline, you'd use this pattern inside a `MERGE` / `INSERT OVERWRITE` statement to maintain an SCD Type 1 table. Mentioning this shows you connect coding patterns to real architectures.

---

## Problem 8: Consecutive Login Days (Gap & Island)

### Problem Statement

Find users who logged in for **3 or more consecutive days**.

| user_id | login_date |
|---------|------------|
| 1       | 2025-03-01 |
| 1       | 2025-03-02 |
| 1       | 2025-03-03 |
| 1       | 2025-03-05 |
| 2       | 2025-03-01 |
| 2       | 2025-03-03 |

**Expected:** User 1 had a 3-day streak (Mar 1–3).

### Approach

Classic **gap-and-island** technique:

1. Assign a `ROW_NUMBER()` per user ordered by date.
2. Subtract the row number (in days) from the login date → consecutive dates produce the **same** "island" identifier.
3. Group by (user, island), count the size, filter ≥ 3.

### Solution

```sql
WITH islands AS (
    SELECT
        user_id,
        login_date,
        login_date - INTERVAL '1 day' * ROW_NUMBER() OVER (
            PARTITION BY user_id ORDER BY login_date
        ) AS island_id
    FROM logins
)
SELECT
    user_id,
    MIN(login_date) AS streak_start,
    MAX(login_date) AS streak_end,
    COUNT(*)        AS streak_length
FROM islands
GROUP BY user_id, island_id
HAVING COUNT(*) >= 3
ORDER BY user_id, streak_start;
```

### Explanation

Let's trace user 1:

| login_date | rn | login_date − rn days | island_id  |
|------------|----|----------------------|------------|
| 2025-03-01 | 1  | 2025-02-28           | 2025-02-28 |
| 2025-03-02 | 2  | 2025-02-28           | 2025-02-28 |
| 2025-03-03 | 3  | 2025-02-28           | 2025-02-28 |
| 2025-03-05 | 4  | 2025-03-01           | 2025-03-01 |

The first three dates produce the same `island_id` → they're consecutive. Mar 5 breaks the pattern, getting a different island.

> **Interview Tip:** Gap-and-island is a **top-5 SQL interview pattern**. The insight — "subtract row_number from date to create a group key" — is elegant and non-obvious. If you can explain *why* it works (consecutive numbers produce the same difference), you'll impress.

**Note:** In MySQL/BigQuery, use `DATE_SUB(login_date, INTERVAL ROW_NUMBER()... DAY)` instead of the PostgreSQL interval arithmetic.

---

## Problem 9: Sessionize User Clicks

### Problem Statement

Group user click events into sessions. A new session starts when there's a **gap of more than 30 minutes** between consecutive clicks by the same user.

| user_id | click_time          | page      |
|---------|---------------------|-----------|
| U1      | 2025-03-15 08:00:00 | /home     |
| U1      | 2025-03-15 08:10:00 | /products |
| U1      | 2025-03-15 08:20:00 | /cart     |
| U1      | 2025-03-15 09:05:00 | /home     |
| U1      | 2025-03-15 09:12:00 | /checkout |

**Expected:** Clicks at 08:00–08:20 → Session 1. Clicks at 09:05–09:12 → Session 2 (45-minute gap between 08:20 and 09:05).

### Approach

This is the SQL version of the sessionization pattern from Module 6:

1. `LAG()` to get the previous click time per user.
2. Compute the gap in minutes.
3. Flag rows where the gap exceeds 30 minutes (or it's the first event).
4. `SUM()` the flag to assign session IDs.

### Solution

```sql
WITH with_gap AS (
    SELECT
        *,
        LAG(click_time) OVER (
            PARTITION BY user_id ORDER BY click_time
        ) AS prev_click,
        EXTRACT(EPOCH FROM (
            click_time - LAG(click_time) OVER (
                PARTITION BY user_id ORDER BY click_time
            )
        )) / 60.0 AS gap_minutes
    FROM clicks
),
with_flag AS (
    SELECT
        *,
        CASE
            WHEN gap_minutes IS NULL OR gap_minutes > 30
            THEN 1
            ELSE 0
        END AS new_session_flag
    FROM with_gap
)
SELECT
    user_id,
    click_time,
    page,
    SUM(new_session_flag) OVER (
        PARTITION BY user_id
        ORDER BY click_time
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS session_id
FROM with_flag
ORDER BY user_id, click_time;
```

### Explanation

1. **`LAG(click_time)`** — Gets the timestamp of the user's previous click.
2. **Gap computation** — `EXTRACT(EPOCH FROM ...)` gives seconds; divide by 60 for minutes.
3. **New-session flag** — 1 if it's the first click (`NULL` gap) or gap > 30 min, else 0.
4. **Running `SUM` of the flag** — Incrementally numbers sessions: flag values of [1,0,0,1,0] become cumulative sums [1,1,1,2,2].

> **Interview Tip:** This is the *exact* same lag → gap → flag → cumsum pattern from Module 6 (PySpark/Pandas). Being able to implement it in all three (SQL, Pandas, PySpark) is a superpower in DE interviews. For BigQuery, replace `EXTRACT(EPOCH FROM ...)` with `TIMESTAMP_DIFF(click_time, prev_click, MINUTE)`.

---

## Problem 10: Org Chart Hierarchy (Recursive CTE)

### Problem Statement

Given an `employees` table with a `manager_id` column, return the full hierarchy under a given employee, including their **level** in the tree.

| emp_id | name    | manager_id |
|--------|---------|------------|
| 1      | Alice   | NULL       |
| 2      | Bob     | 1          |
| 3      | Carol   | 1          |
| 4      | Dave    | 2          |
| 5      | Eve     | 2          |
| 6      | Frank   | 3          |

**Expected (full tree under Alice):**

| emp_id | name  | level |
|--------|-------|-------|
| 1      | Alice | 0     |
| 2      | Bob   | 1     |
| 3      | Carol | 1     |
| 4      | Dave  | 2     |
| 5      | Eve   | 2     |
| 6      | Frank | 2     |

### Approach

**Recursive CTE** — the anchor member selects the root node, and the recursive member joins the CTE back to the table to find children.

### Solution

```sql
WITH RECURSIVE org_tree AS (
    -- Anchor: start at Alice (or any root)
    SELECT emp_id, name, manager_id, 0 AS level
    FROM employees
    WHERE manager_id IS NULL          -- root nodes

    UNION ALL

    -- Recursive: find children of current level
    SELECT e.emp_id, e.name, e.manager_id, t.level + 1
    FROM employees e
    JOIN org_tree t ON e.manager_id = t.emp_id
)
SELECT emp_id, name, level
FROM org_tree
ORDER BY level, name;
```

### Explanation

1. **Anchor member** — Selects Alice (level 0) because her `manager_id IS NULL`.
2. **Recursive member** — Joins `employees` to `org_tree` on `manager_id = emp_id`. First iteration finds Bob and Carol (level 1). Second iteration finds Dave, Eve, Frank (level 2). Iteration stops when the join produces no new rows.
3. **`UNION ALL`** — Must use `ALL` (not `UNION`) for recursive CTEs in most engines.

> **Interview Tip:** Recursive CTEs can infinite-loop if there's a cycle in the data (e.g., A manages B manages A). In production, add a `WHERE level < 20` safety guard or use `CYCLE` detection (PostgreSQL 14+). Mentioning this is a *chef's kiss* moment.

**Note:** BigQuery doesn't support `WITH RECURSIVE` directly — you'd use `CONNECT BY` or iterative scripting. MySQL 8.0+ and PostgreSQL support it natively.

---

## Problem 11: Funnel Conversion Rates

### Problem Statement

Given a `user_events` table, compute a **conversion funnel** for an e-commerce flow:

Step 1: `page_view` → Step 2: `add_to_cart` → Step 3: `checkout` → Step 4: `purchase`

Show the count and **conversion rate** (relative to Step 1) at each stage.

| user_id | event_type  | event_time          |
|---------|-------------|---------------------|
| 1       | page_view   | 2025-03-15 08:00:00 |
| 1       | add_to_cart | 2025-03-15 08:05:00 |
| 1       | checkout    | 2025-03-15 08:10:00 |
| 1       | purchase    | 2025-03-15 08:12:00 |
| 2       | page_view   | 2025-03-15 09:00:00 |
| 2       | add_to_cart | 2025-03-15 09:05:00 |
| 3       | page_view   | 2025-03-15 10:00:00 |

### Approach

Count distinct users at each funnel step using conditional aggregation, then compute conversion rates against the top-of-funnel.

### Solution

```sql
WITH funnel AS (
    SELECT
        COUNT(DISTINCT CASE WHEN event_type = 'page_view'   THEN user_id END) AS step1_views,
        COUNT(DISTINCT CASE WHEN event_type = 'add_to_cart' THEN user_id END) AS step2_cart,
        COUNT(DISTINCT CASE WHEN event_type = 'checkout'    THEN user_id END) AS step3_checkout,
        COUNT(DISTINCT CASE WHEN event_type = 'purchase'    THEN user_id END) AS step4_purchase
    FROM user_events
)
SELECT
    step1_views,
    step2_cart,
    step3_checkout,
    step4_purchase,
    ROUND(100.0 * step2_cart     / NULLIF(step1_views, 0), 1) AS view_to_cart_pct,
    ROUND(100.0 * step3_checkout / NULLIF(step1_views, 0), 1) AS view_to_checkout_pct,
    ROUND(100.0 * step4_purchase / NULLIF(step1_views, 0), 1) AS view_to_purchase_pct
FROM funnel;
```

### Explanation

1. **`COUNT(DISTINCT CASE WHEN ...)`** — Counts unique users who performed each action. The `CASE` returns `NULL` for non-matching events, and `COUNT(DISTINCT ...)` ignores NULLs.
2. **Funnel CTE** — Computes all four step counts in a single pass.
3. **Rate calculation** — Each rate divides by `step1_views` (top of funnel). `NULLIF` prevents division by zero.

> **Interview Tip:** A more rigorous funnel requires **sequential** steps (cart only counts if the user also viewed). To enforce ordering, join on `user_id` with timestamp conditions: `cart.event_time > view.event_time`. Mentioning this nuance shows analytical maturity.

---

## Problem 12: Second Highest Salary Without LIMIT

### Problem Statement

Find the **second highest distinct salary** from the `employees` table. Do NOT use `LIMIT`, `TOP`, or `OFFSET`.

### Approach

Two approaches: (A) subquery with `MAX`, (B) `DENSE_RANK()` window function.

### Solution

```sql
-- Approach A: Subquery (classic)
SELECT MAX(salary) AS second_highest
FROM employees
WHERE salary < (SELECT MAX(salary) FROM employees);

-- Approach B: Window function (more flexible)
WITH ranked AS (
    SELECT
        salary,
        DENSE_RANK() OVER (ORDER BY salary DESC) AS dr
    FROM employees
)
SELECT DISTINCT salary AS second_highest
FROM ranked
WHERE dr = 2;
```

### Explanation

**Approach A:**
1. Inner subquery finds the maximum salary (e.g., 150000).
2. Outer query finds the max salary among rows *strictly less than* 150000 — which is the second highest.
3. Clean and efficient, but only works for "second." For Nth, use Approach B.

**Approach B:**
1. `DENSE_RANK()` assigns rank 1 to the highest salary, rank 2 to the next distinct value, etc.
2. Filter `dr = 2` gives the second-highest. Change to `dr = N` for any Nth value.
3. `DISTINCT` handles cases where multiple employees share the same salary.

> **Interview Tip:** This is a LeetCode classic. Approach A is faster to write; Approach B is more generalizable. Start with A, then offer B as an alternative — it shows range.

---

## SQL Pattern Recognition Guide

When you read a problem statement, look for these **signal phrases** and map them to the right technique:

| Signal Phrase | SQL Pattern | Key Syntax |
|---|---|---|
| "Top N per group" | Window + filter | `DENSE_RANK() OVER (PARTITION BY ... ORDER BY ...) <= N` |
| "Compare to manager/parent" | Self-join | `JOIN table t2 ON t1.parent_id = t2.id` |
| "Never / no matching" | Anti-join | `LEFT JOIN ... WHERE t2.id IS NULL` or `NOT EXISTS` |
| "Running total / cumulative" | Window frame | `SUM() OVER (ORDER BY ... ROWS UNBOUNDED PRECEDING)` |
| "Previous / next row" | LAG / LEAD | `LAG(col, 1) OVER (ORDER BY ...)` |
| "Pivot / crosstab" | Conditional aggregation | `SUM(CASE WHEN ... THEN ... END)` |
| "Remove duplicates" | ROW_NUMBER dedup | `ROW_NUMBER() OVER (PARTITION BY key ORDER BY ts DESC) = 1` |
| "Consecutive days/events" | Gap and island | `date - ROW_NUMBER() ... → GROUP BY island` |
| "Session / group by time gap" | Sessionization | `LAG → gap → flag → cumulative SUM` |
| "Hierarchy / tree / org chart" | Recursive CTE | `WITH RECURSIVE ... UNION ALL ... JOIN cte` |
| "Funnel / conversion" | Conditional COUNT | `COUNT(DISTINCT CASE WHEN step='X' THEN user END)` |
| "Nth highest without LIMIT" | Subquery / window | `DENSE_RANK() = N` or nested `MAX` |
| "Year-over-year / MoM" | LAG with offset | `LAG(col, 12) OVER (ORDER BY month)` |
| "Moving average" | Window frame | `AVG() OVER (ORDER BY ... ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)` |
| "Exists in A but not in B" | Anti-join | `LEFT JOIN ... IS NULL` / `NOT EXISTS` / `EXCEPT` |
| "Median" | PERCENTILE or subquery | `PERCENTILE_CONT(0.5)` or count-based |
| "First/last per group" | Window + filter | `ROW_NUMBER() OVER (...) = 1` |
| "Date series / fill gaps" | Recursive CTE + LEFT JOIN | Generate dates, then left join actuals |

---

## The 5-Step Framework for Any SQL Interview Problem

Use this mental checklist when you hear a new problem:

### Step 1: Clarify

- What are the table schemas? (Draw them out.)
- Are there duplicates? NULLs? Edge cases?
- What does "top" mean — distinct values or distinct rows?

### Step 2: Identify the Pattern

Scan the Pattern Recognition Guide above. Most problems map to 1–2 patterns.

### Step 3: Sketch the CTE Chain

Break the solution into named CTEs. Each CTE should do **one thing**:

```
WITH step1 AS (...),   -- clean / filter
     step2 AS (...),   -- transform / window
     step3 AS (...)    -- aggregate / join
SELECT ... FROM step3
```

### Step 4: Write & Talk

Write the SQL while narrating your reasoning. The interviewer wants to hear *why* you chose `DENSE_RANK` over `ROW_NUMBER`, not just see the syntax.

### Step 5: Verify

- Run through a small example mentally (2–3 rows).
- Check for NULLs in joins and divisions.
- Ask: "Would this work if the table were empty? If there were ties?"

---

## Common Pitfalls to Avoid

1. **`NOT IN` with NULLs** — If the subquery contains any NULL, the entire `NOT IN` returns empty. Use `NOT EXISTS` instead.

2. **Missing `GROUP BY` columns** — Every non-aggregated column in `SELECT` must appear in `GROUP BY` (or be functionally dependent). MySQL is lenient; PostgreSQL and BigQuery are strict.

3. **Window frame defaults** — `SUM() OVER (ORDER BY x)` uses `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` by default. With ties, this includes *all* tied rows — not just up to the current row. Use `ROWS` for deterministic behavior.

4. **Integer division** — `SELECT 1 / 3` returns `0` in many engines. Always cast: `1.0 / 3` or `CAST(1 AS DECIMAL) / 3`.

5. **Forgetting `DISTINCT` in `COUNT`** — `COUNT(user_id)` counts all rows (including duplicates). `COUNT(DISTINCT user_id)` counts unique users. Big difference in funnel analysis.

6. **Self-join double counting** — When comparing pairs (e.g., "find employees in the same department"), use `e1.id < e2.id` to avoid counting (A,B) and (B,A) as separate pairs.

7. **Recursive CTE infinite loops** — Always add a depth limit (`WHERE level < 100`) or ensure your data is acyclic.

---

## Study Plan

| Day | Problems | Focus |
|-----|----------|-------|
| 1 | 1, 2, 3 | Window basics, self-join, anti-join |
| 2 | 4, 5, 6 | Running totals, LAG, conditional agg |
| 3 | 7, 8 | Dedup pattern, gap & island |
| 4 | 9, 10 | Sessionization, recursive CTE |
| 5 | 11, 12 | Funnel analysis, Nth value |
| 6 | All | Timed practice: 12 min per problem |
| 7 | Weakest 4 | Review and re-solve from scratch |

> *"Every SQL problem is just a combination of JOINs, GROUP BYs, and window functions arranged in the right order. Learn the patterns, and the syntax writes itself."*

Good luck — go crush those SQL rounds! 🚀
