# Module 2: Models & Materializations

## Screen 1: What Are Materializations?

A **materialization** tells dbt *how* to persist the results of a model's `SELECT` statement in the warehouse. You write a `SELECT` — dbt decides whether to wrap it in a `CREATE VIEW`, `CREATE TABLE AS SELECT`, a `MERGE` statement, or inline it as a CTE. The materialization you choose has massive implications for query performance, storage costs, and build times.

dbt provides four built-in materializations:

```
┌─────────────────────────────────────────────────────────────┐
│              The Four Materializations                      │
├──────────────┬──────────────────────────────────────────────┤
│ Type         │ What dbt does                               │
├──────────────┼──────────────────────────────────────────────┤
│ view         │ CREATE VIEW AS (SELECT ...)                  │
│ table        │ CREATE TABLE AS SELECT (full rebuild)        │
│ incremental  │ INSERT/MERGE into existing table             │
│ ephemeral    │ Inlined as CTE — no database object created  │
└──────────────┴──────────────────────────────────────────────┘
```

**Why this matters**: Choosing the wrong materialization is one of the most common (and expensive) mistakes in dbt projects. Materialize a complex 10-join model as a `view`, and every downstream query re-executes those 10 joins. Materialize a simple rename-and-cast staging model as a `table`, and you're wasting storage and build time on something a view handles perfectly.

The right choice depends on three factors:
1. **Complexity of the transformation** — How expensive is the SQL to execute?
2. **Frequency of access** — How often do downstream models or dashboards query this?
3. **Data volume** — How much data does this model process?

```sql
-- Setting materialization in a model
{{ config(materialized='table') }}

SELECT
    order_id,
    customer_id,
    order_date,
    order_amount_cents / 100.0 AS order_amount_dollars
FROM {{ ref('stg_orders') }}
```

You can also set materializations at the directory level in `dbt_project.yml` (as we saw in Module 1), and individual models can override with their `{{ config() }}` block. The model-level config always wins.

### 💡 Interview Insight

> **Common Question**: "What are the four materializations in dbt and when would you use each?"
> **Strong Answer**: This is a foundational question — nail it with specifics. "Views for simple transformations queried infrequently, tables for complex transforms queried often, incremental for large datasets where you only want to process new rows, and ephemeral for intermediate logic you don't need to persist. The key tradeoff is build time vs. query time vs. storage cost."

---

## Screen 2: Views — The Lightweight Default

A **view** materialization creates a database view. No data is stored — the view is just a saved SQL query. Every time someone queries the view, the underlying SQL executes against the source tables.

```sql
-- models/staging/stg_orders.sql
-- This compiles to: CREATE VIEW stg_orders AS (SELECT ...)
{{ config(materialized='view') }}

SELECT
    id AS order_id,
    user_id AS customer_id,
    order_date,
    status,
    CAST(amount AS DECIMAL(10,2)) AS order_amount_cents,
    _fivetran_synced AS loaded_at
FROM {{ source('ecommerce', 'raw_orders') }}
WHERE status != 'test'   -- Filter out test orders
```

**What dbt actually executes:**
```sql
CREATE OR REPLACE VIEW ECOMMERCE_DEV.dbt_jsmith.stg_orders AS (
    SELECT
        id AS order_id,
        user_id AS customer_id,
        order_date,
        status,
        CAST(amount AS DECIMAL(10,2)) AS order_amount_cents,
        _fivetran_synced AS loaded_at
    FROM RAW_DATABASE.shopify.raw_orders
    WHERE status != 'test'
);
```

**Advantages of views:**
- **Fast to build** — Creating a view is nearly instant (no data scanned or copied)
- **Always fresh** — Queries always read the latest source data
- **No storage cost** — No duplicate data stored
- **Instant deployment** — No waiting for large table rebuilds

**Disadvantages:**
- **Slow to query** — Every query re-executes the transformation SQL
- **Cascading compute** — If downstream models also query this view, the source tables get hit repeatedly
- **Warehouse load** — Complex views queried by dashboards can be very expensive

**When to use views:**
- Staging models with simple transformations (renaming, casting, filtering)
- Models queried infrequently
- Models where data freshness is critical (real-time or near-real-time)
- Development environments (fast iteration — don't wait for table builds)

**When NOT to use views:**
- Complex multi-join transformations
- Models queried by dashboards with many concurrent users
- Models referenced by many downstream models (compute multiplies)

### 💡 Interview Insight

> **Common Question**: "Why are staging models typically views?"
> **Strong Answer**: "Staging models are 1:1 with source tables and perform lightweight transformations — renaming columns, casting types, filtering test data. These are cheap SQL operations that don't benefit from table materialization. Views keep staging builds instant, and since staging models are only queried by downstream dbt models (not dashboards), the compute cost is manageable."

---

## Screen 3: Tables — Precomputed for Performance

A **table** materialization creates a physical table using `CREATE TABLE AS SELECT`. The query runs once during `dbt run`, data is stored in the warehouse, and subsequent queries read the precomputed results — no re-execution of the transformation SQL.

```sql
-- models/marts/dim_customers.sql
-- This compiles to: CREATE TABLE dim_customers AS (SELECT ...)
{{ config(materialized='table') }}

WITH customer_orders AS (
    SELECT
        customer_id,
        MIN(order_date) AS first_order_date,
        MAX(order_date) AS most_recent_order_date,
        COUNT(*) AS number_of_orders,
        SUM(order_amount_cents) / 100.0 AS lifetime_value
    FROM {{ ref('stg_orders') }}
    GROUP BY customer_id
)

SELECT
    c.customer_id,
    c.first_name,
    c.last_name,
    c.email,
    c.created_at AS customer_since,
    COALESCE(co.first_order_date, NULL) AS first_order_date,
    COALESCE(co.most_recent_order_date, NULL) AS most_recent_order_date,
    COALESCE(co.number_of_orders, 0) AS total_orders,
    COALESCE(co.lifetime_value, 0) AS lifetime_value_dollars,
    CASE
        WHEN co.lifetime_value >= 1000 THEN 'platinum'
        WHEN co.lifetime_value >= 500 THEN 'gold'
        WHEN co.lifetime_value >= 100 THEN 'silver'
        ELSE 'bronze'
    END AS customer_tier
FROM {{ ref('stg_customers') }} AS c
LEFT JOIN customer_orders AS co
    ON c.customer_id = co.customer_id
```

**What dbt actually executes** (simplified):
```sql
-- Step 1: Create a temporary table with the new data
CREATE OR REPLACE TABLE ECOMMERCE_DEV.dbt_jsmith.dim_customers AS (
    -- the full SELECT query above
);
```

In practice, dbt uses an atomic swap pattern on most warehouses — it creates a temporary table, then renames it to replace the old one. This prevents downstream queries from seeing a partially-built table.

**Advantages of tables:**
- **Fast to query** — Data is precomputed and stored; no re-execution
- **Predictable performance** — Dashboard queries hit a static table
- **Can be indexed/clustered** — Optimize for specific query patterns

**Disadvantages:**
- **Slow to build** — Full table rebuild scans all source data every run
- **Storage cost** — Duplicate data stored in the warehouse
- **Staleness** — Data is only as fresh as the last `dbt run`

**Table vs. View decision matrix:**

```
┌───────────────────┬────────────┬────────────┐
│ Factor            │ Use View   │ Use Table  │
├───────────────────┼────────────┼────────────┤
│ Transform cost    │ Low        │ High       │
│ Query frequency   │ Low        │ High       │
│ Data freshness    │ Critical   │ Can wait   │
│ Downstream users  │ Few models │ Dashboards │
│ Build time budget │ Tight      │ Flexible   │
│ Data volume       │ Small      │ Any        │
└───────────────────┴────────────┴────────────┘
```

### 💡 Interview Insight

> **Common Question**: "When would you switch a model from a view to a table?"
> **Strong Answer**: "When the transformation is expensive and gets queried frequently. For example, a `dim_customers` model that joins three tables, aggregates order history, and calculates customer tiers — this should be a table because it would be wasteful to re-compute on every dashboard query. I also switch to table when I notice slow dashboard performance traced to view cascading."

---

## Screen 4: Incremental Models — Process Only New Data

**Incremental** is dbt's most powerful materialization and the one you'll use most in production. Instead of rebuilding the entire table from scratch, an incremental model only processes **new or changed rows** since the last run. For a table with 500 million rows where 100,000 rows arrive daily, this means processing 100K rows instead of 500M.

```sql
-- models/marts/fct_orders.sql
{{ config(
    materialized='incremental',
    unique_key='order_id',
    incremental_strategy='merge'
) }}

SELECT
    o.order_id,
    o.customer_id,
    o.order_date,
    o.status,
    o.order_amount_cents / 100.0 AS order_amount_dollars,
    c.first_name,
    c.last_name,
    c.email,
    CURRENT_TIMESTAMP() AS dbt_updated_at
FROM {{ ref('stg_orders') }} AS o
LEFT JOIN {{ ref('stg_customers') }} AS c
    ON o.customer_id = c.customer_id

{% if is_incremental() %}
    -- Only process rows newer than the latest row in the existing table
    WHERE o.order_date > (SELECT MAX(order_date) FROM {{ this }})
{% endif %}
```

**How it works:**

1. **First run**: The `is_incremental()` block is `False`. dbt creates the full table (like a `table` materialization).
2. **Subsequent runs**: `is_incremental()` is `True`. dbt only processes rows matching the `WHERE` clause, then merges them into the existing table using the `unique_key`.

```
┌──────────────────────────────────────────────────────┐
│           Incremental Model Execution                │
│                                                      │
│  First Run:    SELECT * FROM source                  │
│                ──► CREATE TABLE (full build)          │
│                                                      │
│  Next Runs:    SELECT * FROM source                  │
│                WHERE date > max(existing_table.date)  │
│                ──► MERGE INTO existing table          │
│                                                      │
│  Full Refresh: dbt run --full-refresh                │
│                ──► DROP + CREATE TABLE (rebuild)      │
└──────────────────────────────────────────────────────┘
```

The `{{ this }}` variable refers to the current model's existing table in the warehouse. It's only valid inside an `{% if is_incremental() %}` block.

We'll cover incremental models in depth in Module 3 — this is a preview of the concept and its place in the materialization spectrum.

### 💡 Interview Insight

> **Common Question**: "Why use incremental instead of table?"
> **Strong Answer**: "Performance and cost. A table materialization rebuilds from scratch every run — fine for small datasets, but a 500M-row fact table would take hours and scan massive amounts of data. Incremental processes only new/changed rows, reducing build time from hours to minutes and cutting warehouse compute costs significantly. The tradeoff is added complexity in handling edge cases like late-arriving data."

---

## Screen 5: Ephemeral — The Invisible Model

An **ephemeral** model doesn't create any object in the database — no table, no view, nothing. Instead, dbt inlines the model's SQL as a **Common Table Expression (CTE)** in every model that references it. Think of it as a reusable SQL snippet that gets copy-pasted at compile time.

```sql
-- models/intermediate/int_order_totals.sql
-- This model is NOT persisted — it becomes a CTE in downstream models
{{ config(materialized='ephemeral') }}

SELECT
    customer_id,
    COUNT(*) AS total_orders,
    SUM(order_amount_cents) / 100.0 AS total_spent_dollars,
    AVG(order_amount_cents) / 100.0 AS avg_order_dollars
FROM {{ ref('stg_orders') }}
GROUP BY customer_id
```

```sql
-- models/marts/dim_customers.sql
-- When compiled, int_order_totals is inlined as a CTE above this query
{{ config(materialized='table') }}

SELECT
    c.customer_id,
    c.first_name,
    c.last_name,
    c.email,
    COALESCE(ot.total_orders, 0) AS total_orders,
    COALESCE(ot.total_spent_dollars, 0) AS total_spent_dollars,
    COALESCE(ot.avg_order_dollars, 0) AS avg_order_dollars
FROM {{ ref('stg_customers') }} AS c
LEFT JOIN {{ ref('int_order_totals') }} AS ot
    ON c.customer_id = ot.customer_id
```

**What dbt compiles `dim_customers` to:**
```sql
-- The ephemeral model is inlined as a CTE
WITH __dbt__cte__int_order_totals AS (
    SELECT
        customer_id,
        COUNT(*) AS total_orders,
        SUM(order_amount_cents) / 100.0 AS total_spent_dollars,
        AVG(order_amount_cents) / 100.0 AS avg_order_dollars
    FROM ECOMMERCE_DEV.dbt_jsmith.stg_orders
    GROUP BY customer_id
)

SELECT
    c.customer_id,
    c.first_name,
    c.last_name,
    c.email,
    COALESCE(ot.total_orders, 0) AS total_orders,
    COALESCE(ot.total_spent_dollars, 0) AS total_spent_dollars,
    COALESCE(ot.avg_order_dollars, 0) AS avg_order_dollars
FROM ECOMMERCE_DEV.dbt_jsmith.stg_customers AS c
LEFT JOIN __dbt__cte__int_order_totals AS ot
    ON c.customer_id = ot.customer_id
```

**When to use ephemeral:**
- Intermediate logic shared by a few downstream models
- When you want to keep your warehouse clean (fewer objects)
- Simple aggregations or transformations that don't need to be queried directly

**When NOT to use ephemeral:**
- The model is referenced by many downstream models (the CTE gets duplicated in each)
- You need to debug the intermediate data directly (no table to query)
- The SQL is expensive and gets inlined in multiple places (compute multiplies)

### 💡 Interview Insight

> **Common Question**: "What is an ephemeral model and when would you avoid using it?"
> **Strong Answer**: "Ephemeral models are inlined as CTEs in downstream models — no database object is created. I'd avoid them when the model is referenced by many downstream models, because the CTE gets duplicated in each compiled query, multiplying compute. I'd also avoid them when I need to debug intermediate results, since there's no table to query. They're best for simple, focused logic referenced by one or two downstream models."

---

## Screen 6: Model Naming Conventions & Layers

A well-organized dbt project uses a **layered architecture** with clear naming conventions. This isn't just about aesthetics — it communicates the purpose, reliability, and expected usage of every model at a glance.

```
┌──────────────────────────────────────────────────────────────┐
│                  dbt Model Layers                            │
│                                                              │
│  ┌───────────┐    ┌───────────────┐    ┌─────────────────┐  │
│  │ STAGING   │    │ INTERMEDIATE  │    │     MARTS       │  │
│  │  stg_*    │───►│    int_*      │───►│  fct_* / dim_*  │  │
│  │           │    │               │    │                 │  │
│  │ 1:1 with  │    │ Business      │    │ Business-ready  │  │
│  │ sources   │    │ logic joins   │    │ fact & dim      │  │
│  │           │    │               │    │ tables          │  │
│  │ view      │    │ ephemeral     │    │ table /         │  │
│  │           │    │ or view       │    │ incremental     │  │
│  └───────────┘    └───────────────┘    └─────────────────┘  │
│                                                              │
│  Consumed by:      Consumed by:        Consumed by:          │
│  Other dbt models  Other dbt models    Dashboards, APIs,    │
│  only              only                analysts, ML          │
└──────────────────────────────────────────────────────────────┘
```

**Naming conventions:**

| Prefix | Layer | Purpose | Example |
|--------|-------|---------|---------|
| `stg_` | Staging | 1:1 source mapping | `stg_orders` |
| `int_` | Intermediate | Business logic | `int_order_items_pivoted` |
| `fct_` | Marts (facts) | Event/transaction | `fct_orders` |
| `dim_` | Marts (dimensions) | Entity attributes | `dim_customers` |
| `rpt_` | Reports (optional) | Pre-aggregated | `rpt_monthly_revenue` |

**Staging layer rules:**
- One staging model per source table (1:1 mapping)
- Rename columns to consistent conventions (snake_case)
- Cast types explicitly (don't trust source types)
- Filter out obviously invalid data (test records, nulls)
- No joins, no aggregations — keep it simple
- Always materialized as `view`

```sql
-- models/staging/stg_customers.sql
{{ config(materialized='view') }}

SELECT
    -- Rename to consistent conventions
    id AS customer_id,
    LOWER(TRIM(first_name)) AS first_name,
    LOWER(TRIM(last_name)) AS last_name,
    LOWER(TRIM(email)) AS email,
    -- Explicit type casting
    CAST(created_at AS TIMESTAMP) AS created_at,
    CAST(_fivetran_synced AS TIMESTAMP) AS loaded_at
FROM {{ source('ecommerce', 'raw_customers') }}
-- Filter junk
WHERE id IS NOT NULL
```

**Intermediate layer rules:**
- Named `int_<entity>_<verb>` (e.g., `int_orders_pivoted`, `int_payments_aggregated`)
- Contains business logic: joins, pivots, window functions, complex CASE statements
- Not queried by end users — only by mart models
- Typically ephemeral or view

**Marts layer rules:**
- Facts (`fct_`) represent events or transactions — one row per event (order, click, payment)
- Dimensions (`dim_`) represent entities — one row per entity (customer, product, store)
- Materialized as `table` or `incremental`
- This is what dashboards and analysts query

### 💡 Interview Insight

> **Common Question**: "How do you organize models in a dbt project?"
> **Strong Answer**: "I follow the staging → intermediate → marts pattern. Staging is 1:1 with sources — renaming, casting, filtering — always views. Intermediate holds reusable business logic — joins, aggregations — typically ephemeral. Marts are business-ready fact and dimension tables — materialized as tables or incremental. Naming conventions (stg_, int_, fct_, dim_) make it immediately clear where a model sits in the pipeline and who should be querying it."

---

## Screen 7: The config() Block — Model-Level Configuration

Every dbt model can include a `{{ config() }}` block at the top of the file to set model-specific configurations. This overrides any directory-level settings from `dbt_project.yml`.

```sql
-- Full config block example
{{ config(
    materialized='incremental',
    schema='marts',
    alias='fact_orders',
    tags=['daily', 'critical'],
    unique_key='order_id',
    incremental_strategy='merge',
    cluster_by=['order_date'],
    partition_by={
        "field": "order_date",
        "data_type": "date",
        "granularity": "month"
    },
    on_schema_change='append_new_columns',
    enabled=true,
    persist_docs={"relation": true, "columns": true},
    pre_hook="ALTER SESSION SET TIMEZONE = 'UTC'",
    post_hook="GRANT SELECT ON {{ this }} TO ROLE reporter"
) }}

SELECT ...
```

**Key configuration options:**

| Config | Purpose | Example |
|--------|---------|---------|
| `materialized` | How to persist | `'table'`, `'view'` |
| `schema` | Target schema | `'marts'` |
| `alias` | Override table name | `'fact_orders'` |
| `tags` | Categorize models | `['daily', 'pii']` |
| `enabled` | Toggle model on/off | `true` / `false` |
| `unique_key` | Dedup key for incr. | `'order_id'` |
| `cluster_by` | Warehouse clustering | `['order_date']` |
| `partition_by` | BQ partitioning | `{"field": "date"}` |
| `pre_hook` | SQL before model | `"SET TIMEZONE..."` |
| `post_hook` | SQL after model | `"GRANT SELECT..."` |

**Tags and selectors** — Tags let you run subsets of your project:

```bash
# Run only models tagged 'daily'
$ dbt run --select tag:daily

# Run only models tagged 'critical' AND in the marts directory
$ dbt run --select tag:critical,path:models/marts

# Run a specific model and all its downstream dependencies
$ dbt run --select stg_orders+

# Run a specific model and all its upstream dependencies
$ dbt run --select +fct_orders
```

**Schema configuration and the `generate_schema_name` macro**: By default, when you set `schema='marts'`, dbt creates the model in `<target_schema>_marts` (e.g., `dbt_jsmith_marts`). To deploy to exactly `marts`, you need to customize the `generate_schema_name` macro:

```sql
-- macros/generate_schema_name.sql
{% macro generate_schema_name(custom_schema_name, node) %}
    {%- if custom_schema_name is not none -%}
        {{ custom_schema_name | trim }}
    {%- else -%}
        {{ target.schema }}
    {%- endif -%}
{% endmacro %}
```

### 💡 Interview Insight

> **Common Question**: "How do you handle deploying models to different schemas?"
> **Strong Answer**: "I use the `schema` config to assign models to logical schemas — `staging`, `intermediate`, `marts`. By default, dbt prepends the target schema, so I customize the `generate_schema_name` macro. In dev, I keep the prefix (each dev gets their own namespace). In prod, I override to use exact schema names. This gives us clean, predictable schema organization in production while maintaining developer isolation."

---

## Screen 8: Custom Schemas and Multi-Schema Deployments

In production, you rarely want all models in a single schema. You want staging models in a `staging` schema, marts in a `marts` or `analytics` schema, and perhaps PII models in a restricted `sensitive` schema. dbt's custom schema handling is one of the most commonly misunderstood features.

**The default behavior**: When you set `+schema: marts` in `dbt_project.yml`, dbt creates models in `<target_schema>_marts`. If your target schema is `dbt_jsmith`, the model lands in `dbt_jsmith_marts`. If your target schema is `analytics`, it lands in `analytics_marts`.

This default behavior exists to prevent developers from accidentally overwriting each other's tables during development. But in production, you typically want clean schema names.

**Common production pattern**: Different behavior in dev vs. prod:

```sql
-- macros/generate_schema_name.sql
{% macro generate_schema_name(custom_schema_name, node) -%}
    {%- set default_schema = target.schema -%}

    {%- if target.name == 'prod' -%}
        {# In production, use the custom schema directly #}
        {%- if custom_schema_name is not none -%}
            {{ custom_schema_name | trim }}
        {%- else -%}
            {{ default_schema }}
        {%- endif -%}
    {%- else -%}
        {# In dev, prefix with target schema for isolation #}
        {%- if custom_schema_name is not none -%}
            {{ default_schema }}_{{ custom_schema_name | trim }}
        {%- else -%}
            {{ default_schema }}
        {%- endif -%}
    {%- endif -%}
{%- endmacro %}
```

**Resulting schema layout:**

```
┌────────────────────────────────────────────────────────┐
│              Schema Resolution by Environment          │
├────────────────────┬──────────────┬────────────────────┤
│ Model Config       │ Dev Schema   │ Prod Schema        │
├────────────────────┼──────────────┼────────────────────┤
│ schema: staging    │ dbt_jsmith_  │ staging            │
│                    │   staging    │                    │
│ schema: marts      │ dbt_jsmith_  │ marts              │
│                    │   marts      │                    │
│ (no custom schema) │ dbt_jsmith   │ analytics          │
│                    │              │ (target default)   │
└────────────────────┴──────────────┴────────────────────┘
```

**Database-level separation**: Some teams go further and deploy to different databases:

```sql
-- models/marts/fct_orders.sql
{{ config(
    materialized='table',
    database='ANALYTICS_DB',
    schema='marts'
) }}
```

**Access control with post-hooks**: Combine custom schemas with post-hooks to automate permissions:

```yaml
# dbt_project.yml
models:
  ecommerce_analytics:
    marts:
      +schema: marts
      +post-hook:
        - "GRANT SELECT ON {{ this }} TO ROLE analyst"
        - "GRANT SELECT ON {{ this }} TO ROLE dashboard_reader"
    staging:
      +schema: staging
      +post-hook:
        - "GRANT SELECT ON {{ this }} TO ROLE engineer"
```

### 💡 Interview Insight

> **Common Question**: "Have you encountered issues with dbt custom schemas?"
> **Strong Answer**: "Yes — the most common confusion is that dbt prepends the target schema by default. In dev, this is actually helpful for isolation. In prod, I customize `generate_schema_name` to use the custom schema name directly. This gives us clean production schemas like `staging` and `marts` while keeping developer schemas isolated as `dbt_<username>_staging`, etc."

---

## Screen 9: Quiz — Module 2 Review

### Question 1
**A complex model joins 5 tables, includes window functions, and is queried by 10 Looker dashboards. Which materialization should you use?**

A) `view` — so dashboards always see fresh data
B) `table` — precompute the expensive query so dashboards are fast
C) `ephemeral` — to keep the warehouse clean
D) `incremental` — always use incremental for everything

**Answer: B** — With complex SQL and frequent dashboard queries, a `table` materialization precomputes the results so dashboards read from a static, fast table. A view would re-execute the expensive query on every dashboard load. Incremental could also work if the data volume is large and append-friendly, but `table` is the straightforward choice here.

---

### Question 2
**What does an ephemeral model become in the compiled SQL?**

A) A temporary table in the warehouse
B) A Common Table Expression (CTE) inlined in downstream models
C) A view that's dropped after the run completes
D) A stored procedure

**Answer: B** — Ephemeral models are compiled as CTEs (WITH clauses) and inlined into every downstream model that references them. No database object is created.

---

### Question 3
**What is the correct naming convention for a dbt staging model that maps to the `raw_orders` source table?**

A) `orders.sql`
B) `raw_orders_staging.sql`
C) `stg_orders.sql`
D) `fct_orders.sql`

**Answer: C** — Staging models use the `stg_` prefix. The convention is `stg_<entity>` where the entity name drops the `raw_` prefix. `fct_` is for fact tables in the marts layer, not staging.

---

### Question 4
**By default, if your target schema is `dbt_jsmith` and you set `+schema: marts` in `dbt_project.yml`, what schema will dbt use?**

A) `marts`
B) `dbt_jsmith`
C) `dbt_jsmith_marts`
D) `public_marts`

**Answer: C** — dbt's default `generate_schema_name` macro concatenates the target schema with the custom schema: `<target_schema>_<custom_schema>`. To use just `marts`, you must customize the macro.

---

### Question 5
**Which layer in the dbt project should dashboards and analysts query?**

A) Staging (`stg_*`)
B) Intermediate (`int_*`)
C) Marts (`fct_*` and `dim_*`)
D) Sources (raw tables)

**Answer: C** — Marts are the business-ready layer designed for consumption by dashboards, analysts, and downstream applications. Staging and intermediate models are internal to dbt and should not be directly queried by end users.

---

### 🔑 Key Takeaways

1. **Four materializations**: `view` (no storage, re-executes), `table` (precomputed, stored), `incremental` (process new rows only), `ephemeral` (inlined as CTE)
2. **View for staging**, **table for marts**, **incremental for large facts**, **ephemeral for intermediate logic** — this is the standard production pattern
3. **Model layers**: staging (1:1 source, `stg_`), intermediate (business logic, `int_`), marts (business-ready, `fct_`/`dim_`)
4. **`config()` block** sets model-level configurations that override directory-level settings
5. **Custom schemas** require understanding the `generate_schema_name` macro — default behavior prepends the target schema
6. **Tags and selectors** let you run subsets of models: `dbt run --select tag:daily`
7. **Post-hooks** automate operational tasks like granting permissions after model builds
