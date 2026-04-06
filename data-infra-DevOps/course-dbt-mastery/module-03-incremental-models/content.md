# Module 3: Incremental Models — The Heart of Production dbt

## Screen 1: Why Incremental Models Matter

In toy dbt projects, you can rebuild every table from scratch on every run. In production — with billions of rows, tight SLAs, and real money riding on warehouse compute costs — that's simply not possible. **Incremental models are the single most important materialization for production dbt projects.**

Consider this scenario: Your `fct_orders` table has 800 million rows spanning 3 years of order history. Every day, 200,000 new orders arrive. A `table` materialization would scan all 800M rows on every run, taking 45 minutes and costing $50+ in compute per run. An incremental model processes only the 200K new rows in under 2 minutes.

```
┌──────────────────────────────────────────────────────────┐
│           Table vs Incremental: The Math                 │
│                                                          │
│  TABLE materialization:                                  │
│  ┌──────────────────────────────────────────┐            │
│  │ Scan 800,000,000 rows every run         │            │
│  │ Build time: ~45 minutes                 │            │
│  │ Daily compute cost: ~$50                │            │
│  │ Monthly cost: ~$1,500                   │            │
│  └──────────────────────────────────────────┘            │
│                                                          │
│  INCREMENTAL materialization:                            │
│  ┌──────────────────────────────────────────┐            │
│  │ Scan 200,000 new rows per run           │            │
│  │ Build time: ~2 minutes                  │            │
│  │ Daily compute cost: ~$0.50              │            │
│  │ Monthly cost: ~$15                      │            │
│  └──────────────────────────────────────────┘            │
│                                                          │
│  Savings: 96% less time, 99% less cost                   │
└──────────────────────────────────────────────────────────┘
```

**The fundamental idea**: An incremental model keeps its existing data and only processes rows that are new or changed since the last run. dbt determines what's "new" based on a filter you provide inside an `{% if is_incremental() %}` block.

**The tradeoff**: Incremental models are more complex to write, harder to debug, and introduce edge cases around late-arriving data, schema changes, and data corrections. They require careful thought about:
- **What defines "new" data?** (a timestamp? an ID?)
- **What happens if a row is updated?** (merge vs. append)
- **What about late-arriving data?** (data that arrives after the cutoff)
- **What if the schema changes?** (new columns added to source)

This module digs deep into every one of these concerns — because this is where interviews separate juniors from seniors.

### 💡 Interview Insight

> **Common Question**: "When would you use an incremental model vs. a table?"
> **Strong Answer**: "When the data volume makes full rebuilds impractical or expensive. I use incremental models for fact tables with millions+ rows where new data arrives in a predictable, append-like pattern — event logs, orders, clickstream data. I'd use a table for smaller datasets or dimension tables that need full refreshes. The key decision factor is: does the cost/time of a full rebuild justify the added complexity of incremental logic?"

---

## Screen 2: The Core Pattern — config + is_incremental()

Every incremental model has two parts: the config block and the incremental filter. Let's break down the anatomy:

```sql
-- models/marts/fct_orders.sql

-- Part 1: The config block
{{ config(
    materialized='incremental',
    unique_key='order_id',
    incremental_strategy='merge',
    on_schema_change='append_new_columns'
) }}

-- Part 2: The base SELECT (always runs)
SELECT
    o.order_id,
    o.customer_id,
    o.order_date,
    o.status,
    o.order_amount_cents / 100.0 AS order_amount_dollars,
    c.first_name || ' ' || c.last_name AS customer_name,
    c.email AS customer_email,
    CURRENT_TIMESTAMP() AS dbt_loaded_at
FROM {{ ref('stg_orders') }} AS o
LEFT JOIN {{ ref('stg_customers') }} AS c
    ON o.customer_id = c.customer_id

-- Part 3: The incremental filter (only on subsequent runs)
{% if is_incremental() %}
    WHERE o.loaded_at > (SELECT MAX(dbt_loaded_at) FROM {{ this }})
{% endif %}
```

**How dbt executes this:**

**First run** — `is_incremental()` returns `False`:
```sql
-- dbt builds the full table (no WHERE clause)
CREATE TABLE fct_orders AS (
    SELECT
        o.order_id, o.customer_id, o.order_date, ...
    FROM stg_orders o
    LEFT JOIN stg_customers c ON o.customer_id = c.customer_id
    -- No WHERE clause — all data included
);
```

**Subsequent runs** — `is_incremental()` returns `True`:
```sql
-- dbt selects only new rows, then merges them
MERGE INTO fct_orders AS target
USING (
    SELECT
        o.order_id, o.customer_id, o.order_date, ...
    FROM stg_orders o
    LEFT JOIN stg_customers c ON o.customer_id = c.customer_id
    WHERE o.loaded_at > (SELECT MAX(dbt_loaded_at) FROM fct_orders)
) AS source
ON target.order_id = source.order_id
WHEN MATCHED THEN UPDATE SET ...
WHEN NOT MATCHED THEN INSERT ...;
```

**Key elements explained:**

- **`unique_key`**: The column(s) that uniquely identify each row. Used by the `merge` strategy to determine whether to UPDATE (row exists) or INSERT (new row).
- **`{{ this }}`**: A reference to the current model's existing table. Only valid inside `{% if is_incremental() %}`.
- **`is_incremental()`**: Returns `True` when: (1) the model is configured as incremental, (2) the target table already exists, AND (3) you're not running with `--full-refresh`.

```
┌──────────────────────────────────────────────────┐
│        is_incremental() Decision Tree            │
│                                                  │
│  materialized='incremental'?                     │
│  ├── No  → False (treated as table)              │
│  └── Yes                                         │
│       └── Target table exists?                   │
│            ├── No  → False (first run, full build)│
│            └── Yes                               │
│                 └── --full-refresh flag?          │
│                      ├── Yes → False (rebuild)   │
│                      └── No  → TRUE ✓            │
└──────────────────────────────────────────────────┘
```

### 💡 Interview Insight

> **Common Question**: "Explain how `is_incremental()` works."
> **Strong Answer**: "It returns True only when three conditions are met: the model is configured as incremental, the target table already exists in the warehouse, and the run doesn't include `--full-refresh`. On the first run, it's False (no existing table), so dbt builds the full table. On subsequent runs, it's True, so the WHERE clause filters to only new rows. Running `--full-refresh` forces a complete rebuild."

---

## Screen 3: Incremental Strategies — append, merge, delete+insert, insert_overwrite

dbt supports four incremental strategies, and each warehouse supports a different subset. The strategy determines **how** new rows are combined with existing data.

```
┌──────────────────────────────────────────────────────────────────┐
│              Incremental Strategies Compared                     │
├─────────────────┬────────────────────────────────────────────────┤
│ Strategy        │ How it works                                  │
├─────────────────┼────────────────────────────────────────────────┤
│ append          │ INSERT new rows. Never updates existing rows. │
│                 │ Fastest. Can create duplicates if rerun.      │
│                 │                                                │
│ merge           │ MERGE (upsert) using unique_key.              │
│                 │ Updates existing rows, inserts new ones.       │
│                 │ Most common. Requires unique_key.             │
│                 │                                                │
│ delete+insert   │ DELETE matching rows, then INSERT new ones.    │
│                 │ Useful when MERGE isn't supported or is slow. │
│                 │ Requires unique_key.                          │
│                 │                                                │
│ insert_overwrite│ Overwrite entire partitions containing new     │
│                 │ data. Best for partitioned tables (BigQuery).  │
│                 │ Requires partition_by.                        │
└─────────────────┴────────────────────────────────────────────────┘
```

**Warehouse support matrix:**

| Strategy | Snowflake | BigQuery | Redshift | Databricks |
|----------|-----------|----------|----------|------------|
| append | ✅ | ✅ | ✅ | ✅ |
| merge | ✅ (default) | ✅ | ✅ | ✅ (default) |
| delete+insert | ✅ | ❌ | ✅ (default) | ✅ |
| insert_overwrite | ❌ | ✅ | ❌ | ✅ |

**Append strategy** — simplest, fastest, but no deduplication:

```sql
{{ config(
    materialized='incremental',
    incremental_strategy='append'
) }}

SELECT
    event_id,
    user_id,
    event_type,
    event_timestamp,
    event_properties
FROM {{ ref('stg_clickstream_events') }}

{% if is_incremental() %}
    WHERE event_timestamp > (SELECT MAX(event_timestamp) FROM {{ this }})
{% endif %}
```

Use `append` for immutable event data (clickstream, logs) where rows are never updated and events have monotonically increasing timestamps.

**Merge strategy** — the workhorse of production dbt:

```sql
{{ config(
    materialized='incremental',
    unique_key='order_id',
    incremental_strategy='merge'
) }}

SELECT
    order_id,
    customer_id,
    status,        -- This field can change (shipped → delivered)
    order_amount_dollars,
    updated_at
FROM {{ ref('stg_orders') }}

{% if is_incremental() %}
    WHERE updated_at > (SELECT MAX(updated_at) FROM {{ this }})
{% endif %}
```

Use `merge` when rows can be **updated** (order status changes, customer info updates). The `unique_key` determines whether to INSERT or UPDATE.

**Delete+Insert strategy** — an alternative when MERGE is slow:

```sql
{{ config(
    materialized='incremental',
    unique_key='order_id',
    incremental_strategy='delete+insert'
) }}
-- Same SELECT as above
```

On some warehouses, DELETE + INSERT is faster than MERGE for large batches. The behavior is the same — old rows are removed, new versions are inserted.

**Insert_overwrite strategy** — partition-level replacement:

```sql
{{ config(
    materialized='incremental',
    incremental_strategy='insert_overwrite',
    partition_by={
        "field": "order_date",
        "data_type": "date",
        "granularity": "day"
    }
) }}

SELECT * FROM {{ ref('stg_orders') }}

{% if is_incremental() %}
    WHERE order_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 3 DAY)
{% endif %}
```

This overwrites entire date partitions. Perfect for BigQuery where you want to reprocess the last 3 days of data (handling late-arriving events).

### 💡 Interview Insight

> **Common Question**: "What incremental strategy would you use for an orders fact table where order status can change?"
> **Strong Answer**: "Merge, with `unique_key='order_id'`. When an order status changes from 'shipped' to 'delivered', the merge strategy updates the existing row instead of creating a duplicate. I'd filter on `updated_at` to capture both new orders and status changes. For immutable event data like clickstream, I'd use append since rows never change."

---

## Screen 4: unique_key — Deduplication and Composite Keys

The `unique_key` is what makes the `merge` and `delete+insert` strategies work. It tells dbt how to identify whether an incoming row already exists in the target table. Get this wrong, and you'll either create duplicates or lose data.

**Single column unique key** (most common):
```sql
{{ config(
    materialized='incremental',
    unique_key='order_id',
    incremental_strategy='merge'
) }}
```

**Composite unique key** (when no single column is unique):
```sql
-- Each customer can have one row per date
{{ config(
    materialized='incremental',
    unique_key=['customer_id', 'activity_date'],
    incremental_strategy='merge'
) }}

SELECT
    customer_id,
    activity_date,
    SUM(page_views) AS total_page_views,
    SUM(orders_placed) AS total_orders,
    MAX(last_active_at) AS last_active_at
FROM {{ ref('stg_customer_daily_activity') }}

{% if is_incremental() %}
    WHERE activity_date >= (SELECT MAX(activity_date) FROM {{ this }})
{% endif %}
GROUP BY customer_id, activity_date
```

With a list of columns, dbt generates a MERGE condition like:
```sql
MERGE INTO fct_customer_daily AS target
USING (...) AS source
ON target.customer_id = source.customer_id
   AND target.activity_date = source.activity_date
WHEN MATCHED THEN UPDATE SET ...
WHEN NOT MATCHED THEN INSERT ...;
```

**Surrogate key pattern** — when you need to create a unique key:

```sql
{{ config(
    materialized='incremental',
    unique_key='surrogate_key',
    incremental_strategy='merge'
) }}

SELECT
    {{ dbt_utils.generate_surrogate_key(['order_id', 'line_item_id']) }}
        AS surrogate_key,
    order_id,
    line_item_id,
    product_id,
    quantity,
    unit_price,
    updated_at
FROM {{ ref('stg_order_items') }}

{% if is_incremental() %}
    WHERE updated_at > (SELECT MAX(updated_at) FROM {{ this }})
{% endif %}
```

**What happens without a unique_key?** If you use `merge` strategy without a `unique_key`, dbt treats every row as new — effectively behaving like `append`. This is a common source of duplicates.

**Common gotcha — choosing the wrong key:**

```
┌────────────────────────────────────────────────────────┐
│              unique_key Gotchas                        │
│                                                        │
│  ❌ Using a non-unique column:                         │
│     unique_key='customer_id' on an orders table        │
│     → Only keeps one order per customer!               │
│                                                        │
│  ❌ Forgetting composite keys:                         │
│     unique_key='date' on a per-customer-per-day table  │
│     → Only one customer's data survives per day!       │
│                                                        │
│  ✅ Correct: unique_key matches the grain              │
│     One row per order → unique_key='order_id'          │
│     One row per customer per day →                     │
│       unique_key=['customer_id', 'date']               │
└────────────────────────────────────────────────────────┘
```

**Rule of thumb**: The `unique_key` should match the **grain** of the table. Ask yourself: "What combination of columns makes each row unique?" That's your key.

### 💡 Interview Insight

> **Common Question**: "How do you handle deduplication in incremental models?"
> **Strong Answer**: "Through the `unique_key` config, which must match the table's grain. For an orders table, it's `order_id`. For a daily customer activity table, it's `['customer_id', 'activity_date']`. When I don't have a natural key, I generate a surrogate key using `dbt_utils.generate_surrogate_key()`. The unique_key drives the MERGE statement's ON clause — get it wrong and you get duplicates or data loss."

---

## Screen 5: on_schema_change — Handling Evolving Schemas

In production, source schemas change. A product team adds a `discount_amount` column to the orders table. An API starts returning a new field. Your incremental model's SELECT now returns more columns than the existing target table has. The `on_schema_change` config determines what dbt does.

```sql
{{ config(
    materialized='incremental',
    unique_key='order_id',
    on_schema_change='sync_all_columns'
) }}
```

**Four options:**

| Option | Behavior | Use When |
|--------|----------|----------|
| `ignore` (default) | New columns silently dropped | You control schema tightly |
| `append_new_columns` | New columns added (nullable) | Schema evolves, no removals |
| `sync_all_columns` | Add new, remove old columns | Full schema sync needed |
| `fail` | Raise an error on mismatch | You want explicit control |

**Example scenario:**

Your current `fct_orders` table has columns: `order_id, customer_id, order_date, status, amount`.

A new source column `discount_amount` appears, and your staging model now includes it. Your incremental model's SELECT returns 6 columns, but the target table has 5.

```
on_schema_change='ignore':
  → discount_amount is silently dropped
  → Target table still has 5 columns
  → No error, but you lose data!

on_schema_change='append_new_columns':
  → ALTER TABLE ADD COLUMN discount_amount
  → New rows have discount_amount populated
  → Old rows have NULL for discount_amount
  → Target table now has 6 columns

on_schema_change='sync_all_columns':
  → Adds new columns, removes columns no longer in SELECT
  → Most aggressive — can remove columns you still need!

on_schema_change='fail':
  → dbt raises an error: "Schema mismatch detected"
  → You must explicitly handle the change
  → Safest for critical production tables
```

**Production recommendation:**

```sql
-- For critical fact tables, fail on schema changes
-- Forces explicit handling via --full-refresh or schema migration
{{ config(
    materialized='incremental',
    unique_key='order_id',
    on_schema_change='fail'
) }}
```

For tables where you expect schema evolution (e.g., event properties), use `append_new_columns`. For tables where schema changes should never happen silently, use `fail`.

**Handling the failure:**

When `on_schema_change='fail'` triggers an error, you have two options:
1. **Full refresh**: `dbt run --select fct_orders --full-refresh` — rebuilds the table with the new schema
2. **Manual ALTER**: Add the column manually, then run the incremental model normally

### 💡 Interview Insight

> **Common Question**: "How do you handle schema changes in incremental models?"
> **Strong Answer**: "With the `on_schema_change` config. For critical tables, I use `fail` to force explicit handling — I don't want schema changes sneaking in silently. For event tables with evolving properties, I use `append_new_columns` since new fields are expected. I avoid `ignore` in production because it silently drops data. When a schema change is detected, I do a `--full-refresh` to rebuild with the correct schema."

---

## Screen 6: Full Refresh — The Escape Hatch

`dbt run --full-refresh` is your emergency reset button for incremental models. It forces dbt to drop the existing table and rebuild it from scratch, as if it were a `table` materialization. This is essential for fixing data issues, applying schema changes, and performing periodic backfills.

```bash
# Full refresh a single model
$ dbt run --select fct_orders --full-refresh

# Full refresh all incremental models
$ dbt run --full-refresh

# Full refresh a model and all its downstream dependencies
$ dbt run --select fct_orders+ --full-refresh
```

**When to use full refresh:**

| Scenario | Why Full Refresh |
|----------|-----------------|
| Schema change | New columns need to be added |
| Bug fix | Historical data was calculated wrong |
| Backfill | Source data was retroactively corrected |
| Logic change | Business rule changed; need to recompute |
| Data corruption | Bad data got into the incremental table |
| Periodic maintenance | Weekly/monthly full rebuild for data quality |

**Production pattern — scheduled full refreshes:**

Many production dbt projects run incremental daily but full-refresh weekly or monthly. This catches drift between the incremental and source data — late-arriving records, corrections, and edge cases that the incremental filter might miss.

```
┌──────────────────────────────────────────────────────┐
│          Typical Production Schedule                 │
│                                                      │
│  Mon-Sat:  dbt run                                   │
│            (incremental — process new rows only)     │
│                                                      │
│  Sunday:   dbt run --full-refresh                    │
│            (complete rebuild — catch any drift)      │
│                                                      │
│  Cost: 6 fast runs + 1 slow run per week             │
│  Benefit: Guaranteed data consistency                │
└──────────────────────────────────────────────────────┘
```

**Designing for full-refresh safety**: Your incremental model should produce the **exact same results** whether run incrementally or as a full refresh. This is critical — if your incremental logic diverges from a full rebuild, you have a bug.

```sql
-- GOOD: This model produces identical results either way
{{ config(materialized='incremental', unique_key='order_id') }}

SELECT
    order_id,
    customer_id,
    order_date,
    status,
    amount
FROM {{ ref('stg_orders') }}

{% if is_incremental() %}
    WHERE updated_at > (SELECT MAX(updated_at) FROM {{ this }})
{% endif %}

-- BAD: This model produces DIFFERENT results on full refresh
-- because it uses CURRENT_DATE() in a way that changes behavior
{% if is_incremental() %}
    WHERE order_date >= DATEADD(day, -3, CURRENT_DATE())
{% else %}
    WHERE order_date >= '2020-01-01'  -- Different filter = different results!
{% endif %}
```

### 💡 Interview Insight

> **Common Question**: "How often do you full-refresh your incremental models?"
> **Strong Answer**: "I run incremental daily for cost efficiency and schedule weekly full refreshes as a safety net. This catches late-arriving data, retroactive corrections, and any incremental drift. For critical models, I also validate that incremental and full-refresh produce identical row counts as a data quality check. Full refresh is also necessary after schema changes, bug fixes, or business logic updates."

---

## Screen 7: Gotchas — Late-Arriving Data and Backfill Patterns

Late-arriving data is the #1 production issue with incremental models. Your filter says "give me everything after the max timestamp in the table," but what if data arrives *after* the cutoff because of processing delays, timezone issues, or batch retries?

**The problem:**

```
Timeline:
  Run at 6am: MAX(loaded_at) = 5:59am
  → Processes all rows up to 5:59am ✓

  At 7am: A batch of orders from 4am-5am finally arrives
  → These rows have loaded_at between 4am-5am
  → Next run at 6am tomorrow: MAX(loaded_at) is now ~5:59am today
  → The late rows (4am-5am yesterday) are OLDER than the MAX
  → They're filtered out forever! ✗
```

**Solution 1: Lookback window** — Always reprocess the last N hours/days:

```sql
{{ config(
    materialized='incremental',
    unique_key='order_id',
    incremental_strategy='merge'
) }}

SELECT
    order_id,
    customer_id,
    order_date,
    status,
    order_amount_dollars,
    loaded_at
FROM {{ ref('stg_orders') }}

{% if is_incremental() %}
    -- Look back 3 days to catch late-arriving data
    WHERE loaded_at >= (
        SELECT DATEADD(day, -3, MAX(loaded_at))
        FROM {{ this }}
    )
{% endif %}
```

The `merge` strategy with `unique_key` ensures that reprocessing the same rows doesn't create duplicates — existing rows get updated, new rows get inserted.

**Solution 2: Use a monotonically increasing column** — Prefer a column that's set on arrival, not on event time:

```sql
{% if is_incremental() %}
    -- _fivetran_synced is when the row was loaded, not when the event happened
    -- This is monotonically increasing and doesn't have late-arrival issues
    WHERE _fivetran_synced > (SELECT MAX(_fivetran_synced) FROM {{ this }})
{% endif %}
```

**Solution 3: Partition-based reprocessing** (BigQuery):

```sql
{{ config(
    materialized='incremental',
    incremental_strategy='insert_overwrite',
    partition_by={"field": "order_date", "data_type": "date"},
) }}

SELECT * FROM {{ ref('stg_orders') }}

{% if is_incremental() %}
    -- Reprocess the last 3 days of partitions entirely
    WHERE order_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 3 DAY)
{% endif %}
```

**Backfill patterns** — When you need to reprocess a specific date range:

```sql
-- Using dbt vars for controlled backfills
{% if is_incremental() %}
    {% if var('backfill_start_date', none) is not none %}
        -- Manual backfill mode
        WHERE order_date BETWEEN '{{ var("backfill_start_date") }}'
                          AND '{{ var("backfill_end_date") }}'
    {% else %}
        -- Normal incremental mode
        WHERE loaded_at > (SELECT MAX(loaded_at) FROM {{ this }})
    {% endif %}
{% endif %}
```

```bash
# Normal run
$ dbt run --select fct_orders

# Backfill a specific date range
$ dbt run --select fct_orders \
  --vars '{"backfill_start_date": "2024-01-15", "backfill_end_date": "2024-01-20"}'
```

### 💡 Interview Insight

> **Common Question**: "How do you handle late-arriving data in incremental models?"
> **Strong Answer**: "Three strategies: (1) A lookback window — reprocess the last 3 days with a merge strategy so existing rows get updated and late arrivals are captured. (2) Use a monotonically increasing column like `_fivetran_synced` instead of event time. (3) On BigQuery, use `insert_overwrite` to rewrite recent partitions entirely. I also schedule periodic full refreshes as a safety net. For targeted backfills, I use dbt vars to specify date ranges."

---

## Screen 8: Real-World Patterns — Event Tables, Daily Snapshots, SCD

Let's look at three production patterns that come up in every real dbt project and every senior-level interview.

**Pattern 1: High-Volume Event Table (Clickstream/Logs)**

```sql
-- models/marts/fct_page_views.sql
{{ config(
    materialized='incremental',
    incremental_strategy='append',     -- Events are immutable
    partition_by={
        "field": "event_date",
        "data_type": "date",
        "granularity": "day"
    },
    cluster_by=['user_id']
) }}

SELECT
    event_id,
    user_id,
    page_url,
    referrer_url,
    session_id,
    CAST(event_timestamp AS DATE) AS event_date,
    event_timestamp,
    user_agent,
    {{ dbt_utils.generate_surrogate_key(['event_id']) }} AS page_view_key
FROM {{ ref('stg_page_views') }}

{% if is_incremental() %}
    WHERE event_timestamp > (SELECT MAX(event_timestamp) FROM {{ this }})
{% endif %}
```

Why `append`? Events are immutable — a page view never gets "updated." Append is faster than merge because it skips the deduplication step.

**Pattern 2: Daily Snapshot Table (Aggregated Metrics)**

```sql
-- models/marts/fct_daily_order_metrics.sql
{{ config(
    materialized='incremental',
    unique_key=['metric_date'],
    incremental_strategy='merge'
) }}

SELECT
    DATE(order_date) AS metric_date,
    COUNT(DISTINCT order_id) AS total_orders,
    COUNT(DISTINCT customer_id) AS unique_customers,
    SUM(order_amount_dollars) AS total_revenue,
    AVG(order_amount_dollars) AS avg_order_value,
    SUM(CASE WHEN status = 'returned' THEN 1 ELSE 0 END) AS returned_orders,
    CURRENT_TIMESTAMP() AS calculated_at
FROM {{ ref('fct_orders') }}

{% if is_incremental() %}
    -- Recompute the last 3 days (handles late orders and status changes)
    WHERE DATE(order_date) >= (
        SELECT DATEADD(day, -3, MAX(metric_date))
        FROM {{ this }}
    )
{% endif %}

GROUP BY DATE(order_date)
```

Why `merge` with lookback? Daily metrics need to be recalculated when late orders arrive or order statuses change. The 3-day lookback ensures accuracy.

**Pattern 3: SCD Type 2 with dbt Snapshots**

While not technically an incremental model, dbt's `snapshot` functionality is closely related. It tracks how dimension data changes over time.

```sql
-- snapshots/snap_customers.sql
{% snapshot snap_customers %}

{{ config(
    target_database='ANALYTICS_DB',
    target_schema='snapshots',
    unique_key='customer_id',
    strategy='timestamp',
    updated_at='updated_at',
    invalidate_hard_deletes=True
) }}

SELECT
    customer_id,
    first_name,
    last_name,
    email,
    subscription_tier,
    updated_at
FROM {{ source('ecommerce', 'raw_customers') }}

{% endsnapshot %}
```

This produces a table with `dbt_valid_from`, `dbt_valid_to`, and `dbt_scd_id` columns, tracking every historical state of each customer:

```
┌─────────────┬────────┬──────────┬─────────────────┬─────────────────┐
│ customer_id │ tier   │ email    │ dbt_valid_from  │ dbt_valid_to    │
├─────────────┼────────┼──────────┼─────────────────┼─────────────────┤
│ 101         │ bronze │ a@b.com  │ 2024-01-01      │ 2024-06-15      │
│ 101         │ gold   │ a@b.com  │ 2024-06-15      │ NULL            │
│ 102         │ silver │ c@d.com  │ 2024-03-01      │ NULL            │
└─────────────┴────────┴──────────┴─────────────────┴─────────────────┘
```

### 💡 Interview Insight

> **Common Question**: "Describe a complex incremental model you've built in production."
> **Strong Answer**: Walk through a real pattern: "I built an incremental fact table for order events processing ~500K rows daily from a 300M+ row table. I used merge strategy with `order_id` as the unique key and a 3-day lookback window to handle late-arriving data and status changes. We partitioned by order_date and clustered by customer_id for query performance. Weekly full refreshes caught any drift. We also used dbt snapshots on the customer dimension for SCD Type 2 historical tracking."

---

## Screen 9: Quiz — Module 3 Review

### Question 1
**When does `is_incremental()` return True?**

A) Always, for any model configured as incremental
B) Only when the model config says `materialized='incremental'`, the target table exists, AND `--full-refresh` is NOT used
C) Only on the first run of the model
D) When the source data has new rows

**Answer: B** — All three conditions must be met: incremental config, existing target table, and no `--full-refresh` flag. On the first run, the table doesn't exist yet, so it returns False.

---

### Question 2
**Which incremental strategy should you use for an orders table where order status can change from 'pending' to 'shipped' to 'delivered'?**

A) `append` — fastest performance
B) `merge` — updates existing rows when status changes
C) `insert_overwrite` — replaces entire partitions
D) `delete+insert` — always use this on Snowflake

**Answer: B** — Merge (upsert) with `unique_key='order_id'` will update the status of existing orders while inserting new ones. Append would create duplicate rows with different statuses.

---

### Question 3
**You notice that some orders arriving at 2am are consistently missing from your incremental model that runs at midnight. What's the likely cause and fix?**

A) The warehouse is down at 2am — schedule the run later
B) Late-arriving data — add a lookback window (e.g., `WHERE loaded_at >= MAX(loaded_at) - INTERVAL 6 HOURS`)
C) The unique_key is wrong — change it to order_date
D) Use `on_schema_change='fail'` to catch the missing data

**Answer: B** — This is a classic late-arriving data problem. The midnight run captures everything up to midnight, but orders that arrive at 2am have timestamps before midnight. A lookback window reprocesses recent data to catch late arrivals, and the merge strategy prevents duplicates.

---

### Question 4
**What does `{{ this }}` refer to in an incremental model?**

A) The source table being read
B) The current model's existing target table in the warehouse
C) The dbt project name
D) The Jinja template itself

**Answer: B** — `{{ this }}` resolves to the current model's existing table (e.g., `ANALYTICS_DB.marts.fct_orders`). It's used inside `{% if is_incremental() %}` to query the existing data for determining what's new.

---

### Question 5
**Your `fct_orders` incremental model has been running for 6 months. A developer changes the business logic for calculating `order_total`. What should you do?**

A) Just run `dbt run` — the new logic will apply to new rows automatically
B) Run `dbt run --select fct_orders --full-refresh` to rebuild with the corrected logic
C) Delete the table manually and run `dbt run`
D) Change `on_schema_change` to `sync_all_columns`

**Answer: B** — A logic change requires reprocessing all historical data, not just new rows. `--full-refresh` drops and rebuilds the table with the corrected calculation. Option A would only apply the new logic to new rows, leaving 6 months of data with the old calculation.

---

### 🔑 Key Takeaways

1. **Incremental models process only new/changed rows** — essential for large production tables to save time and compute costs
2. **`is_incremental()`** returns True only when: incremental config + table exists + no `--full-refresh`
3. **Four strategies**: `append` (immutable events), `merge` (upsert — most common), `delete+insert` (alternative to merge), `insert_overwrite` (partition-level replacement)
4. **`unique_key` must match the table grain** — wrong key = duplicates or data loss
5. **`on_schema_change`** controls what happens when source columns change: `ignore`, `append_new_columns`, `sync_all_columns`, `fail`
6. **Late-arriving data** is the #1 gotcha — use lookback windows, monotonic columns, or partition overwrites
7. **Full refresh** is your escape hatch — use it for schema changes, bug fixes, backfills, and periodic data quality assurance
8. **Design for consistency** — incremental and full-refresh should produce identical results
