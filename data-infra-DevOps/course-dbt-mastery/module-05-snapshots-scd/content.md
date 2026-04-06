# Module 5: Snapshots & SCD Type 2

## Screen 1: Why Track Historical Changes?

In a typical e-commerce database, when a customer upgrades from "bronze" to "gold" tier, the source system overwrites the old value. Yesterday they were bronze; today they're gold. The old value is gone — **mutated in place**. But your analytics team needs to answer: "How much revenue did gold-tier customers generate last quarter?" If a customer upgraded mid-quarter, should their earlier purchases count as bronze or gold?

This is the **Slowly Changing Dimension (SCD)** problem. Dimensions — customers, products, stores — change over time, and your analytics warehouse needs to preserve that history even when the source system doesn't.

```
┌──────────────────────────────────────────────────────────────────┐
│           The Problem: Source Systems Mutate In Place             │
│                                                                  │
│  SOURCE DATABASE (raw_customers):                                │
│  ┌─────────────┬────────┬──────────────┬─────────────────────┐   │
│  │ customer_id │ tier   │ email        │ updated_at          │   │
│  ├─────────────┼────────┼──────────────┼─────────────────────┤   │
│  │ 101         │ gold   │ ali@shop.com │ 2024-06-15 10:30:00 │   │
│  └─────────────┴────────┴──────────────┴─────────────────────┘   │
│  ↑ Was "bronze" before June 15 — but that info is GONE.          │
│                                                                  │
│  WHAT WE WANT (snapshot table):                                  │
│  ┌─────────────┬────────┬──────────────┬────────────┬──────────┐ │
│  │ customer_id │ tier   │ email        │ valid_from │ valid_to │ │
│  ├─────────────┼────────┼──────────────┼────────────┼──────────┤ │
│  │ 101         │ bronze │ ali@shop.com │ 2024-01-01 │ 2024-06-15│ │
│  │ 101         │ gold   │ ali@shop.com │ 2024-06-15 │ NULL     │ │
│  └─────────────┴────────┴──────────────┴────────────┴──────────┘ │
│  ↑ Both versions preserved! NULL valid_to = current record.      │
└──────────────────────────────────────────────────────────────────┘
```

**SCD Types — a quick taxonomy:**

| Type | Behavior | Use Case |
|------|----------|----------|
| Type 0 | Never update — keep original | Rarely used (original signup date) |
| Type 1 | Overwrite — no history | Small/unimportant dimensions |
| Type 2 | Add new row — full history | **dbt snapshots** — the standard |
| Type 3 | Add column (current + previous) | When only one prior value matters |

**dbt snapshots implement SCD Type 2** — they create a new row every time a tracked column changes, with `dbt_valid_from` and `dbt_valid_to` timestamps marking each version's lifespan.

### 💡 Interview Insight

> **Common Question**: "What is SCD Type 2 and how does dbt implement it?"
> **Strong Answer**: "SCD Type 2 preserves full history by adding a new row every time a dimension changes, with validity timestamps marking each version. dbt implements this through `snapshot` blocks that compare the current source state against the snapshot table. When a row changes, dbt sets `dbt_valid_to` on the old row and inserts a new row with `dbt_valid_from` = now and `dbt_valid_to` = NULL. The current version always has `dbt_valid_to IS NULL`."

---

## Screen 2: Snapshot Blocks — Anatomy and Syntax

Snapshots live in the `snapshots/` directory (not `models/`) and use a special `{% snapshot %}` block syntax. Let's build a customer snapshot for our e-commerce platform.

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
    shipping_country,
    lifetime_spend_cents,
    updated_at
FROM {{ source('ecommerce', 'raw_customers') }}

{% endsnapshot %}
```

**Key elements explained:**

- **`{% snapshot snap_customers %}`** — Defines the snapshot block with a name. This becomes the table name in the warehouse.
- **`target_database` / `target_schema`** — Where the snapshot table is created. Best practice: isolate snapshots in their own schema.
- **`unique_key`** — The natural key that identifies each entity. Must be truly unique in the source.
- **`strategy`** — How dbt detects changes: `timestamp` or `check` (see next screen).
- **`updated_at`** — The timestamp column used by the `timestamp` strategy.
- **`invalidate_hard_deletes`** — Whether to track rows deleted from the source (see Screen 5).

**Running snapshots:**

```bash
# Run all snapshots
$ dbt snapshot

# Run a specific snapshot
$ dbt snapshot --select snap_customers

# Run snapshots as part of a full build
$ dbt build    # Runs models, tests, snapshots, seeds in DAG order
```

**What dbt generates under the hood (first run):**

```sql
CREATE TABLE ANALYTICS_DB.snapshots.snap_customers AS (
    SELECT
        customer_id,
        first_name,
        last_name,
        email,
        subscription_tier,
        shipping_country,
        lifetime_spend_cents,
        updated_at,
        -- dbt adds these meta columns:
        MD5(customer_id || '|' || updated_at)  AS dbt_scd_id,
        updated_at                              AS dbt_valid_from,
        NULL::TIMESTAMP                         AS dbt_valid_to,
        CURRENT_TIMESTAMP()                     AS dbt_updated_at
    FROM RAW_DB.ecommerce.raw_customers
);
```

**On subsequent runs**, dbt compares each source row against the snapshot table. If `updated_at` has changed for a given `customer_id`, dbt:
1. Sets `dbt_valid_to = CURRENT_TIMESTAMP()` on the old row
2. Inserts a new row with `dbt_valid_from = CURRENT_TIMESTAMP()` and `dbt_valid_to = NULL`

```
┌────────────────────────────────────────────────────────────────┐
│            Snapshot Execution Flow                              │
│                                                                │
│  Source Row (customer_id=101, tier='gold', updated_at=Jun 15)  │
│                    │                                           │
│                    ▼                                           │
│  ┌─── Compare with snapshot ───┐                               │
│  │ snapshot.updated_at = Jan 1 │                               │
│  │ source.updated_at  = Jun 15 │                               │
│  │ CHANGED? YES                │                               │
│  └─────────────┬───────────────┘                               │
│                │                                               │
│      ┌─────────┴──────────┐                                    │
│      ▼                    ▼                                    │
│  Old row:              New row:                                │
│  SET dbt_valid_to      INSERT with                             │
│  = NOW()               dbt_valid_from = NOW()                  │
│                        dbt_valid_to   = NULL                   │
└────────────────────────────────────────────────────────────────┘
```

### 💡 Interview Insight

> **Common Question**: "Where do snapshots live in a dbt project and how are they different from models?"
> **Strong Answer**: "Snapshots live in the `snapshots/` directory, not `models/`. They use `{% snapshot %}` blocks instead of `{{ config() }}`. Unlike models, snapshots always query source tables directly — not `ref()` to other models — because they need to compare raw source state against the historical snapshot. They're run with `dbt snapshot` or as part of `dbt build`. The output table includes `dbt_valid_from`, `dbt_valid_to`, and `dbt_scd_id` columns automatically."

---

## Screen 3: Strategies — `timestamp` vs `check`

dbt offers two strategies for detecting changes. Choose wisely — the wrong strategy can miss changes or create unnecessary versions.

**Strategy 1: `timestamp` (preferred)**

Detects changes by comparing the `updated_at` column in the source against the snapshot. If the timestamp is newer, dbt considers the row changed.

```sql
{% snapshot snap_products %}

{{ config(
    target_schema='snapshots',
    unique_key='product_id',
    strategy='timestamp',
    updated_at='updated_at'
) }}

SELECT
    product_id,
    product_name,
    category,
    price_cents,
    is_active,
    updated_at
FROM {{ source('ecommerce', 'raw_products') }}

{% endsnapshot %}
```

**Strategy 2: `check` (column comparison)**

Detects changes by comparing specific column values. If any of the checked columns differ, dbt considers the row changed.

```sql
{% snapshot snap_products_check %}

{{ config(
    target_schema='snapshots',
    unique_key='product_id',
    strategy='check',
    check_cols=['product_name', 'category', 'price_cents', 'is_active']
) }}

SELECT
    product_id,
    product_name,
    category,
    price_cents,
    is_active
FROM {{ source('ecommerce', 'raw_products') }}

{% endsnapshot %}
```

You can also use `check_cols='all'` to monitor every column (except the `unique_key`):

```sql
{{ config(
    strategy='check',
    check_cols='all'    -- Track ALL column changes
) }}
```

**Comparison table:**

```
┌──────────────────────────────────────────────────────────────────┐
│           timestamp vs check Strategy                            │
├──────────────────┬──────────────────┬────────────────────────────┤
│ Factor           │ timestamp        │ check                      │
├──────────────────┼──────────────────┼────────────────────────────┤
│ Requires         │ updated_at col   │ No timestamp needed        │
│ Detection        │ Timestamp compare│ Value-by-value compare     │
│ Performance      │ Fast (single col)│ Slower (multi-col compare) │
│ False positives  │ Possible*        │ No — checks actual values  │
│ False negatives  │ No**             │ Possible if cols excluded  │
│ Best for         │ Sources with     │ Sources without reliable   │
│                  │ reliable timestamps│ timestamps                │
│ dbt_valid_from   │ = source updated_at│ = snapshot run timestamp  │
│ Recommendation   │ ✅ DEFAULT CHOICE │ Use only when no timestamp │
└──────────────────┴──────────────────┴────────────────────────────┘

* False positive: source updates updated_at without changing data
** Assumes updated_at is always set correctly on changes
```

**When `timestamp` can fail:**
- Source system updates `updated_at` without actually changing data (ETL re-syncs)
- Source system changes data WITHOUT updating `updated_at` (buggy application)
- Multiple changes between snapshot runs (only the latest state is captured)

**When `check` is necessary:**
- The source has no `updated_at` column
- The source's `updated_at` is unreliable (not updated on all changes)
- You only care about changes to specific columns

**Critical caveat for both strategies**: Snapshots only capture the state **at the moment `dbt snapshot` runs**. If a customer changes from bronze → silver → gold between two snapshot runs, you'll only see bronze → gold. The silver state is lost.

```
┌────────────────────────────────────────────────────────────┐
│           The In-Between Problem                           │
│                                                            │
│  Actual changes:  bronze → silver → gold                   │
│                   (Jan 1)  (Jan 5)  (Jan 10)               │
│                                                            │
│  Snapshot runs:   Jan 1          Jan 15                    │
│                   ↓              ↓                         │
│  Captured:        bronze ──────→ gold                      │
│                   (silver state is LOST forever)            │
│                                                            │
│  Solution: Run snapshots frequently (daily or more)        │
└────────────────────────────────────────────────────────────┘
```

### 💡 Interview Insight

> **Common Question**: "When would you use `check` strategy vs `timestamp`?"
> **Strong Answer**: "I default to `timestamp` — it's faster and simpler. But I use `check` when the source system doesn't have an `updated_at` column, or when the timestamp is unreliable (e.g., an ETL tool updates `updated_at` on every sync regardless of actual changes). With `check`, I specify exactly which columns to monitor, so non-meaningful changes (like internal audit columns) don't trigger new snapshot versions. The tradeoff is performance — `check` compares every tracked column value on every run."

---

## Screen 4: Output Columns — Understanding the Snapshot Table

Every snapshot table includes four dbt-managed columns alongside your source columns. Understanding these is critical for building downstream dimension models.

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  snap_customers — After Multiple Runs                                        │
├─────────────┬────────┬──────────────┬──────────────────┬──────────────────┬──┤
│ customer_id │ tier   │ email        │ dbt_valid_from   │ dbt_valid_to     │…│
├─────────────┼────────┼──────────────┼──────────────────┼──────────────────┼──┤
│ 101         │ bronze │ ali@shop.com │ 2024-01-01 06:00 │ 2024-03-15 06:00 │  │
│ 101         │ silver │ ali@shop.com │ 2024-03-15 06:00 │ 2024-06-15 06:00 │  │
│ 101         │ gold   │ ali@b.com    │ 2024-06-15 06:00 │ NULL             │  │
│ 102         │ gold   │ bob@shop.com │ 2024-02-01 06:00 │ NULL             │  │
│ 103         │ bronze │ cat@shop.com │ 2024-01-15 06:00 │ 2024-09-01 06:00 │  │
│ 103         │ NULL   │ cat@shop.com │ 2024-09-01 06:00 │ NULL             │  │
└─────────────┴────────┴──────────────┴──────────────────┴──────────────────┴──┘
  ↑ customer_id=103 was hard-deleted from source;
    invalidate_hard_deletes=True sets dbt_valid_to on the old row
    and optionally creates a "deleted" version.
```

**Column definitions:**

| Column | Type | Description |
|--------|------|-------------|
| `dbt_scd_id` | VARCHAR | Unique surrogate key for each snapshot row (hash of unique_key + valid_from) |
| `dbt_valid_from` | TIMESTAMP | When this version became active (from `updated_at` or snapshot runtime) |
| `dbt_valid_to` | TIMESTAMP | When this version was superseded (`NULL` = current active version) |
| `dbt_updated_at` | TIMESTAMP | When dbt last processed this row (snapshot runtime) |

**Key queries you'll write against snapshots:**

```sql
-- Get the CURRENT state of all customers (latest version)
SELECT *
FROM snap_customers
WHERE dbt_valid_to IS NULL;

-- Get the state of a customer at a specific point in time
SELECT *
FROM snap_customers
WHERE customer_id = 101
  AND dbt_valid_from <= '2024-04-01'
  AND (dbt_valid_to > '2024-04-01' OR dbt_valid_to IS NULL);

-- Count how many times each customer's tier changed
SELECT
    customer_id,
    COUNT(*) - 1 AS tier_changes
FROM snap_customers
GROUP BY customer_id
HAVING COUNT(*) > 1;

-- Get all customers who were 'gold' tier during Q2 2024
SELECT DISTINCT customer_id
FROM snap_customers
WHERE subscription_tier = 'gold'
  AND dbt_valid_from < '2024-07-01'
  AND (dbt_valid_to >= '2024-04-01' OR dbt_valid_to IS NULL);
```

**The `dbt_scd_id` column** is a hash that uniquely identifies each version row. It's generated as `MD5(unique_key || '|' || dbt_valid_from)`. Use it as the primary key for the snapshot table in downstream joins and tests.

```yaml
# models/staging/_stg_schema.yml
models:
  - name: snap_customers
    columns:
      - name: dbt_scd_id
        tests:
          - unique
          - not_null
```

### 💡 Interview Insight

> **Common Question**: "How do you get the current version of a record from a snapshot table?"
> **Strong Answer**: "Filter for `WHERE dbt_valid_to IS NULL`. The current (active) version always has a NULL `dbt_valid_to`. For point-in-time queries, I filter where `dbt_valid_from <= target_date AND (dbt_valid_to > target_date OR dbt_valid_to IS NULL)`. The `dbt_scd_id` column serves as the unique surrogate key for each version row — I use it for joins and primary key tests."

---

## Screen 5: Hard Deletes — `invalidate_hard_deletes`

When a row is **deleted** from the source system, what should happen in the snapshot? By default, dbt does nothing — the snapshot retains the last known state with `dbt_valid_to = NULL` forever. The row looks "current" even though it no longer exists in the source.

The `invalidate_hard_deletes=True` config fixes this by detecting deleted rows and closing them out.

```sql
{% snapshot snap_customers %}

{{ config(
    target_schema='snapshots',
    unique_key='customer_id',
    strategy='timestamp',
    updated_at='updated_at',
    invalidate_hard_deletes=True   -- ← Track deletions
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

**How it works:**

```
┌────────────────────────────────────────────────────────────────┐
│           Hard Delete Detection                                │
│                                                                │
│  Run 1 (Jan 1): Source has customers 101, 102, 103             │
│  Snapshot: 101 (valid), 102 (valid), 103 (valid)               │
│                                                                │
│  ── Customer 103 is deleted from source ──                     │
│                                                                │
│  Run 2 (Jan 15): Source has customers 101, 102 (no 103!)       │
│                                                                │
│  invalidate_hard_deletes=False (default):                      │
│  → 103 stays with dbt_valid_to=NULL (looks current!)           │
│  → No way to tell it was deleted ❌                            │
│                                                                │
│  invalidate_hard_deletes=True:                                 │
│  → 103 gets dbt_valid_to=CURRENT_TIMESTAMP()                   │
│  → Row is properly closed out ✅                               │
└────────────────────────────────────────────────────────────────┘
```

**Performance consideration**: With `invalidate_hard_deletes=True`, dbt must perform a full anti-join between the source and snapshot on every run to detect missing rows. For large tables (100M+ rows), this can be expensive. If hard deletes are rare or irrelevant to your business, consider leaving this disabled.

**Detecting hard deletes in downstream models:**

```sql
-- models/marts/dim_customers.sql
-- Build dimension from snapshot, flagging deleted customers

SELECT
    dbt_scd_id          AS customer_key,
    customer_id,
    first_name,
    last_name,
    email,
    subscription_tier,
    dbt_valid_from      AS effective_from,
    dbt_valid_to        AS effective_to,
    CASE
        WHEN dbt_valid_to IS NULL THEN TRUE
        ELSE FALSE
    END                 AS is_current,
    CASE
        WHEN dbt_valid_to IS NOT NULL
         AND customer_id NOT IN (
             SELECT customer_id FROM {{ ref('snap_customers') }}
             WHERE dbt_valid_to IS NULL
         )
        THEN TRUE
        ELSE FALSE
    END                 AS is_deleted
FROM {{ ref('snap_customers') }}
```

### 💡 Interview Insight

> **Common Question**: "What happens when a source row is deleted? How does dbt handle that in snapshots?"
> **Strong Answer**: "By default, dbt does nothing — the deleted row stays in the snapshot with `dbt_valid_to = NULL`, looking like a current record. With `invalidate_hard_deletes=True`, dbt detects missing rows via an anti-join and sets their `dbt_valid_to` to close them out. I enable this for entity dimensions (customers, products) where deletions are meaningful. I disable it for high-volume tables where deletions are rare and the anti-join cost is high."

---

## Screen 6: Building Dimension Tables from Snapshots

Snapshots are raw SCD Type 2 tables. In practice, you build polished **dimension models** on top of them — adding business logic, derived columns, and proper naming conventions.

```sql
-- models/marts/dim_customers.sql
-- SCD Type 2 dimension built from snapshot

{{ config(
    materialized='table'
) }}

WITH snapshot_data AS (
    SELECT
        dbt_scd_id,
        customer_id,
        first_name,
        last_name,
        email,
        subscription_tier,
        shipping_country,
        lifetime_spend_cents,
        dbt_valid_from,
        dbt_valid_to,
        dbt_updated_at
    FROM {{ ref('snap_customers') }}
),

enriched AS (
    SELECT
        -- Surrogate key for this version
        dbt_scd_id                              AS customer_version_key,

        -- Natural key
        customer_id,

        -- Attributes
        first_name,
        last_name,
        first_name || ' ' || last_name          AS full_name,
        LOWER(email)                            AS email,
        COALESCE(subscription_tier, 'free')     AS subscription_tier,
        shipping_country,
        lifetime_spend_cents / 100.0            AS lifetime_spend_dollars,

        -- Tier classification
        CASE
            WHEN subscription_tier = 'gold'     THEN 3
            WHEN subscription_tier = 'silver'   THEN 2
            WHEN subscription_tier = 'bronze'   THEN 1
            ELSE 0
        END                                     AS tier_rank,

        -- SCD metadata
        dbt_valid_from                          AS effective_from,
        dbt_valid_to                            AS effective_to,
        dbt_valid_to IS NULL                    AS is_current_version,

        -- Row ordering within each customer
        ROW_NUMBER() OVER (
            PARTITION BY customer_id
            ORDER BY dbt_valid_from
        )                                       AS version_number,

        -- Was this a tier upgrade, downgrade, or lateral?
        LAG(subscription_tier) OVER (
            PARTITION BY customer_id
            ORDER BY dbt_valid_from
        )                                       AS previous_tier

    FROM snapshot_data
)

SELECT
    *,
    CASE
        WHEN previous_tier IS NULL THEN 'initial'
        WHEN tier_rank > LAG(tier_rank) OVER (
            PARTITION BY customer_id ORDER BY effective_from
        ) THEN 'upgrade'
        WHEN tier_rank < LAG(tier_rank) OVER (
            PARTITION BY customer_id ORDER BY effective_from
        ) THEN 'downgrade'
        ELSE 'lateral_change'
    END AS change_type
FROM enriched
```

**Joining fact tables to SCD Type 2 dimensions:**

This is where it gets tricky. You must join on the natural key AND the validity period to get the correct dimension attributes at the time of each fact event.

```sql
-- models/marts/fct_orders_enriched.sql
-- Join orders to the customer dimension valid at order time

SELECT
    o.order_id,
    o.order_date,
    o.order_amount_dollars,
    o.status,
    c.customer_id,
    c.full_name,
    c.subscription_tier    AS tier_at_order_time,
    c.tier_rank
FROM {{ ref('fct_orders') }} o
LEFT JOIN {{ ref('dim_customers') }} c
    ON o.customer_id = c.customer_id
    AND o.order_date >= c.effective_from
    AND (o.order_date < c.effective_to OR c.effective_to IS NULL)
```

```
┌────────────────────────────────────────────────────────────────┐
│           SCD Type 2 Join Pattern                              │
│                                                                │
│  fct_orders:                                                   │
│  ┌──────────┬────────────┬─────────────┐                       │
│  │ order_id │ order_date │ customer_id │                       │
│  │ 501      │ 2024-02-10 │ 101         │  ← tier was "bronze" │
│  │ 502      │ 2024-05-20 │ 101         │  ← tier was "silver" │
│  │ 503      │ 2024-08-01 │ 101         │  ← tier was "gold"   │
│  └──────────┴────────────┴─────────────┘                       │
│                                                                │
│  dim_customers (SCD2):                                         │
│  ┌─────────────┬────────┬────────────┬────────────┐            │
│  │ customer_id │ tier   │ eff_from   │ eff_to     │            │
│  │ 101         │ bronze │ 2024-01-01 │ 2024-03-15 │            │
│  │ 101         │ silver │ 2024-03-15 │ 2024-06-15 │            │
│  │ 101         │ gold   │ 2024-06-15 │ NULL       │            │
│  └─────────────┴────────┴────────────┴────────────┘            │
│                                                                │
│  Join: ON customer_id AND order_date BETWEEN eff_from/eff_to   │
│  Result: Each order gets the tier that was active at the time.  │
└────────────────────────────────────────────────────────────────┘
```

### 💡 Interview Insight

> **Common Question**: "How do you join a fact table to an SCD Type 2 dimension?"
> **Strong Answer**: "You join on the natural key AND the validity period: `ON fact.customer_id = dim.customer_id AND fact.event_date >= dim.effective_from AND (fact.event_date < dim.effective_to OR dim.effective_to IS NULL)`. The NULL check on `effective_to` handles the current version. This ensures each fact row gets the dimension attributes that were active at the time of the event — not the current values."

---

## Screen 7: Snapshots vs Manual SCD — When to Use Which

Not every SCD scenario calls for dbt snapshots. Understanding when to use snapshots vs. building SCD logic manually is a mark of senior dbt proficiency.

**Use dbt snapshots when:**

```
┌────────────────────────────────────────────────────────────────┐
│  ✅ USE dbt SNAPSHOTS                                          │
│                                                                │
│  • Source has no change history (mutations overwrite in place)  │
│  • You need SCD Type 2 tracking for a dimension                │
│  • The source table is reasonably sized (< 50M rows)           │
│  • Changes are infrequent (dimensions change slowly)           │
│  • You're okay with daily-granularity change tracking           │
│  • Standard timestamp or check strategy suffices               │
└────────────────────────────────────────────────────────────────┘
```

**Build manual SCD logic when:**

```
┌────────────────────────────────────────────────────────────────┐
│  🔧 BUILD MANUAL SCD LOGIC                                     │
│                                                                │
│  • Source already provides change history (CDC/event stream)    │
│  • You need sub-daily granularity (snapshots run on schedule)   │
│  • Source table is huge (100M+ rows — snapshot perf degrades)   │
│  • You need SCD Type 3 or Type 6 (hybrid)                      │
│  • Complex merge logic (conditional updates, partial tracking)  │
│  • You want to track changes at the column level                │
└────────────────────────────────────────────────────────────────┘
```

**Manual SCD from a CDC source (e.g., Debezium events):**

```sql
-- models/marts/dim_customers_from_cdc.sql
-- Build SCD Type 2 from a CDC event stream — no snapshot needed

{{ config(materialized='incremental', unique_key='customer_version_key') }}

WITH cdc_events AS (
    SELECT
        customer_id,
        first_name,
        last_name,
        email,
        subscription_tier,
        operation,           -- 'INSERT', 'UPDATE', 'DELETE'
        event_timestamp,
        LEAD(event_timestamp) OVER (
            PARTITION BY customer_id
            ORDER BY event_timestamp
        ) AS next_event_timestamp
    FROM {{ ref('stg_customer_cdc_events') }}
    {% if is_incremental() %}
        WHERE event_timestamp > (SELECT MAX(effective_from) FROM {{ this }})
    {% endif %}
)

SELECT
    {{ dbt_utils.generate_surrogate_key(
        ['customer_id', 'event_timestamp']
    ) }}                                    AS customer_version_key,
    customer_id,
    first_name,
    last_name,
    email,
    subscription_tier,
    event_timestamp                         AS effective_from,
    next_event_timestamp                    AS effective_to,
    next_event_timestamp IS NULL            AS is_current_version,
    operation = 'DELETE'                    AS is_deleted
FROM cdc_events
WHERE operation != 'DELETE' OR next_event_timestamp IS NULL
```

**Decision matrix:**

| Factor | dbt Snapshot | Manual SCD |
|--------|-------------|------------|
| Setup complexity | Low (config only) | High (custom SQL) |
| Source requirements | Just needs current state | Needs CDC/event stream |
| Change granularity | Per snapshot run (daily) | Per source event (real-time) |
| Performance at scale | Degrades > 50M rows | Incremental, scales well |
| Column-level tracking | No (row-level only) | Yes (custom logic) |
| Maintenance | dbt manages it | You maintain it |

### 💡 Interview Insight

> **Common Question**: "When would you NOT use dbt snapshots?"
> **Strong Answer**: "Three scenarios: (1) When the source already provides change history via CDC — I'd build an incremental model from the CDC events instead of redundantly snapshotting. (2) When the source table is very large (100M+ rows) — the full-table comparison on each snapshot run becomes expensive. (3) When I need sub-daily change tracking — snapshots only capture state at run time, so if changes happen between runs, intermediate states are lost."

---

## Screen 8: Real-World Gotchas and Best Practices

After running snapshots in production for months, these are the issues that bite teams — and how to prevent them.

**Gotcha 1: Snapshot must query source, not a ref()**

```sql
-- ❌ WRONG — Don't snapshot a staging model
{% snapshot snap_customers %}
{{ config(strategy='timestamp', unique_key='customer_id', updated_at='updated_at') }}
SELECT * FROM {{ ref('stg_customers') }}   -- ❌ staging model!
{% endsnapshot %}

-- ✅ CORRECT — Snapshot the raw source directly
{% snapshot snap_customers %}
{{ config(strategy='timestamp', unique_key='customer_id', updated_at='updated_at') }}
SELECT * FROM {{ source('ecommerce', 'raw_customers') }}   -- ✅ raw source!
{% endsnapshot %}
```

**Why?** If `stg_customers` has transformations that change column values, your snapshot tracks changes in your transformation logic — not changes in the source data. The snapshot should capture the raw state.

**Gotcha 2: Never modify a snapshot's `unique_key`**

Once a snapshot is running in production, changing the `unique_key` will corrupt the table. dbt won't know how to match old rows to new ones. If you must change the key, drop the snapshot table and rebuild from scratch.

**Gotcha 3: Column additions require care**

Adding a column to the snapshot SELECT is fine — dbt will add it to the table. But **removing a column** can cause issues. The old rows will still have the column (with historical values), but new rows won't populate it. Use `on_schema_change` behavior varies for snapshots — be cautious.

**Gotcha 4: Timezone mismatches**

```sql
-- ❌ Source uses UTC, but snapshot runs in America/Chicago
-- dbt_valid_from timestamps won't align with source updated_at

-- ✅ Always ensure consistent timezone handling
SELECT
    customer_id,
    CONVERT_TIMEZONE('UTC', updated_at) AS updated_at  -- Normalize!
FROM {{ source('ecommerce', 'raw_customers') }}
```

**Gotcha 5: Running snapshots too infrequently**

If you run snapshots weekly, you'll miss interim state changes. A customer going bronze → silver → gold in one week shows up as bronze → gold.

**Best practices checklist:**

```
┌────────────────────────────────────────────────────────────────┐
│           Snapshot Best Practices                               │
│                                                                │
│  1. ✅ Always snapshot source(), never ref()                   │
│  2. ✅ Run snapshots daily (minimum) — more for volatile data  │
│  3. ✅ Use timestamp strategy when possible (faster)           │
│  4. ✅ Enable invalidate_hard_deletes for entity dimensions    │
│  5. ✅ Test dbt_scd_id for unique + not_null                   │
│  6. ✅ Build clean dim_ models on top of snapshots             │
│  7. ✅ Never change the unique_key after initial deployment    │
│  8. ✅ Normalize timezones in the snapshot SELECT              │
│  9. ✅ Store snapshots in a dedicated schema (e.g., snapshots) │
│ 10. ✅ Monitor snapshot table growth — SCD2 tables grow fast!  │
└────────────────────────────────────────────────────────────────┘
```

**Monitoring snapshot growth:**

```sql
-- Useful query: How many versions per entity?
SELECT
    customer_id,
    COUNT(*) AS version_count,
    MIN(dbt_valid_from) AS first_seen,
    MAX(dbt_valid_from) AS last_changed
FROM snap_customers
GROUP BY customer_id
ORDER BY version_count DESC
LIMIT 20;

-- If some entities have 50+ versions, investigate:
-- Is the source updated_at changing on every sync? (false positives)
-- Is the check strategy too broad? (monitoring noisy columns)
```

### 💡 Interview Insight

> **Common Question**: "What are the most common issues you've seen with dbt snapshots in production?"
> **Strong Answer**: "Four big ones: (1) Snapshotting a staging model instead of the raw source — you end up tracking transformation changes, not source changes. (2) Timezone mismatches between `updated_at` and `dbt_valid_from`, causing confusing validity windows. (3) Not enabling `invalidate_hard_deletes` — deleted records look current forever. (4) Running snapshots too infrequently and missing interim state changes. I also monitor version counts per entity to detect false positives from overly sensitive change detection."

---

## Screen 9: Quiz — Module 5 Review

### Question 1
**What SCD type do dbt snapshots implement?**

A) SCD Type 1 — overwrite old values
B) SCD Type 2 — add new row with validity timestamps
C) SCD Type 3 — add previous value column
D) SCD Type 6 — hybrid approach

**Answer: B** — dbt snapshots implement SCD Type 2. Each time a tracked row changes, dbt closes out the old version (sets `dbt_valid_to`) and inserts a new version with `dbt_valid_from` = now and `dbt_valid_to` = NULL. This preserves the full change history.

---

### Question 2
**Your source table has no `updated_at` column. Which snapshot strategy should you use?**

A) `timestamp` — it's always the best choice
B) `check` — compare specific column values for changes
C) `append` — just add new rows
D) Snapshots won't work without a timestamp

**Answer: B** — The `check` strategy detects changes by comparing column values directly, without needing a timestamp column. You specify which columns to monitor with `check_cols`. It's slower than `timestamp` but works when no reliable timestamp exists.

---

### Question 3
**How do you identify the current (active) version of a record in a snapshot table?**

A) `WHERE dbt_scd_id IS NOT NULL`
B) `WHERE dbt_valid_to IS NULL`
C) `WHERE dbt_updated_at = (SELECT MAX(dbt_updated_at) FROM snap_table)`
D) `WHERE version_number = 1`

**Answer: B** — The current version always has `dbt_valid_to IS NULL`. When a new version is inserted, the previous version's `dbt_valid_to` is set to the current timestamp. Only the latest version remains open (NULL).

---

### Question 4
**A customer is deleted from the source system. With `invalidate_hard_deletes=False` (default), what happens in the snapshot?**

A) The row is deleted from the snapshot
B) The row remains with `dbt_valid_to = NULL` — it looks current
C) The row gets `dbt_valid_to = CURRENT_TIMESTAMP()`
D) dbt raises an error

**Answer: B** — Without `invalidate_hard_deletes`, dbt doesn't detect deletions. The row stays in the snapshot with `dbt_valid_to = NULL`, making it appear as a current, active record — even though it's been deleted from the source.

---

### Question 5
**Why should snapshots query `source()` instead of `ref()` to a staging model?**

A) `ref()` doesn't work in snapshot blocks
B) Staging models may transform data — snapshots should capture raw source state
C) `source()` is faster than `ref()`
D) Snapshots require database-level access that `ref()` doesn't provide

**Answer: B** — Snapshots should capture the raw source state so you're tracking actual source changes, not changes in your transformation logic. If `stg_customers` renames or recasts a column, the snapshot would detect that as a "change" even if the source data didn't change — creating false SCD versions.

---

### 🔑 Key Takeaways

1. **dbt snapshots implement SCD Type 2** — preserving full history with `dbt_valid_from` / `dbt_valid_to` timestamps
2. **Two strategies**: `timestamp` (preferred, faster) and `check` (when no reliable timestamp exists)
3. **Current records** have `dbt_valid_to IS NULL` — use this filter for "latest state" queries
4. **`dbt_scd_id`** is the unique surrogate key for each version row — test it for unique + not_null
5. **`invalidate_hard_deletes=True`** detects source deletions and closes out the snapshot row
6. **Always snapshot `source()`, never `ref()`** — capture raw state, not transformed state
7. **Run snapshots frequently** (daily minimum) — changes between runs are lost forever
8. **Build `dim_` models on top of snapshots** — add business logic, derived columns, and clean naming
9. **SCD Type 2 joins** require matching on natural key AND validity period (`event_date BETWEEN effective_from AND effective_to`)
10. **Never change the `unique_key`** on a production snapshot — drop and rebuild if needed
