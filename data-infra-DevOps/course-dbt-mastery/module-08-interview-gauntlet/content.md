# Module 8: The Interview Gauntlet 🔥

This module is different. No lectures — just scenario-based questions that mirror real dbt interviews at companies like Walmart, Meta, Airbnb, and top data consultancies. Work through each question, formulate your answer, then check the provided response.

---

## Screen 1: Architecture & Design — "Design Me a dbt Project for X"

These questions test your ability to make architectural decisions and justify them. Interviewers want to hear your reasoning, not just the answer.

---

**Q1: You're joining an e-commerce company with 50 raw source tables, 3 data analysts, and a Snowflake warehouse. Design the dbt project structure.**

**Strong Answer:**

```
ecommerce-dbt/
├── dbt_project.yml
├── packages.yml
├── profiles.yml (gitignored — or use env vars)
├── models/
│   ├── staging/                    # 1:1 with source tables
│   │   ├── shopify/
│   │   │   ├── _shopify__sources.yml
│   │   │   ├── _shopify__models.yml
│   │   │   ├── stg_shopify__orders.sql
│   │   │   ├── stg_shopify__customers.sql
│   │   │   └── stg_shopify__products.sql
│   │   ├── stripe/
│   │   │   ├── _stripe__sources.yml
│   │   │   ├── stg_stripe__payments.sql
│   │   │   └── stg_stripe__refunds.sql
│   │   └── google_analytics/
│   │       └── stg_ga__sessions.sql
│   ├── intermediate/               # Business logic joins, pre-aggregation
│   │   ├── int_orders_with_payments.sql
│   │   └── int_customer_order_history.sql
│   └── marts/                      # Business-facing, consumption-ready
│       ├── core/
│       │   ├── fct_orders.sql
│       │   ├── dim_customers.sql
│       │   └── dim_products.sql
│       ├── finance/
│       │   ├── fct_revenue.sql
│       │   └── rpt_monthly_revenue.sql
│       └── marketing/
│           ├── fct_customer_acquisition.sql
│           └── rpt_campaign_performance.sql
├── snapshots/
│   ├── snap_customers.sql
│   └── snap_products.sql
├── tests/
│   ├── assert_positive_order_amounts.sql
│   └── assert_revenue_reconciliation.sql
├── macros/
│   ├── cents_to_dollars.sql
│   ├── generate_schema_name.sql
│   └── grant_select.sql
├── seeds/
│   ├── country_codes.csv
│   └── exchange_rates.csv
└── analyses/
    └── ad_hoc_revenue_check.sql
```

Key decisions: staging is 1:1 with sources (materialized as views), organized by source system. Intermediate models handle complex joins. Marts are organized by business domain. Snapshots on slowly changing dimensions (customers, products). Custom `generate_schema_name` macro so dev uses prefixed schemas while prod uses clean names.

---

**Q2: Your `fct_orders` table has 2 billion rows and takes 90 minutes to build. How do you optimize it?**

**Strong Answer:**

1. **Materialize as incremental** with merge strategy, `unique_key='order_id'`, and 3-day lookback for late-arriving data
2. **Partition by `order_date`** (BigQuery) or cluster by `order_date, customer_id` (Snowflake)
3. **Reduce upstream scanning** — ensure staging views filter appropriately
4. **Use `insert_overwrite`** on BigQuery instead of merge (faster for partition-level replacement)
5. **Schedule full refresh weekly** (Sunday 2am) for data consistency, incremental daily
6. **Optimize Snowflake warehouse size** — use MEDIUM for this model's task, SMALL for lighter models via `{{ config(snowflake_warehouse='TRANSFORM_MEDIUM') }}`
7. **Consider splitting** — if the 90 minutes is due to complex joins, pre-compute intermediate models

---

**Q3: Two teams (Finance and Marketing) both need customer data but with different definitions. How do you handle this?**

**Strong Answer:**

Shared foundation, domain-specific marts:
- `dim_customers` in `marts/core/` — the canonical customer dimension with attributes both teams need
- `fct_customer_revenue` in `marts/finance/` — Finance's view with LTV, payment terms, credit status
- `fct_customer_engagement` in `marts/marketing/` — Marketing's view with acquisition channel, campaign attribution, engagement scores

Both domain marts `ref('dim_customers')` for shared attributes. Business-specific logic stays in domain models. This avoids duplication while respecting different business contexts. Document the "official" definition in `dim_customers` description.

---

**Q4: When would you use an ephemeral materialization vs. a view vs. a table?**

**Strong Answer:**

| Materialization | When to Use |
|----------------|-------------|
| **Ephemeral** | CTE-like helper logic referenced by ONE model. Never queried directly. Example: a currency conversion CTE. Avoid if referenced by multiple models (SQL gets duplicated into each). |
| **View** | Staging models (lightweight, always current). Small datasets. Models where you want zero storage cost and real-time source reflection. |
| **Table** | Mart/report models queried by BI tools. Complex transformations that are expensive to recompute. Models queried frequently by analysts. |
| **Incremental** | Large fact tables (millions+ rows) where only new/changed data needs processing. Event/log tables. |

---

**Q5: You inherit a dbt project where all 200 models are materialized as tables, there are zero tests, and everything runs sequentially. What's your remediation plan?**

**Strong Answer:**

Phase 1 (Week 1-2): **Triage and stabilize**
- Add `unique` + `not_null` tests to every primary key (use `codegen` package to auto-generate YAML)
- Switch staging models from `table` to `view` (immediate cost savings)
- Run `dbt_project_evaluator` to identify structural issues

Phase 2 (Week 3-4): **Incremental wins**
- Convert the 5 largest fact tables to incremental (biggest cost savings)
- Add `relationships` tests for foreign keys between staging and marts
- Implement `dbt build` instead of `dbt run` + `dbt test`

Phase 3 (Month 2): **CI/CD and quality**
- Set up Slim CI with `state:modified+`
- Add pre-commit hooks (SQLFluff, dbt-checkpoint)
- Add source freshness checks
- Write singular tests for critical business logic

Phase 4 (Month 3): **Documentation and scale**
- Add descriptions to all models and key columns
- Set up snapshots for slowly changing dimensions
- Implement monitoring (parse `run_results.json` for alerts)

---

**Q6: How do you handle different environments (dev, staging, prod) in dbt?**

**Strong Answer:**

Three-pronged approach:
1. **`profiles.yml`** — different targets per environment (dev uses personal schema, prod uses production database)
2. **`generate_schema_name` macro override** — dev: `dev_alice_staging`, prod: `staging`
3. **`target.name` conditionals** — mask PII in dev with `{% if target.name != 'prod' %}`, use different warehouse sizes, etc.

CI gets its own target with isolated schemas: `ci_pr_{{ var('pr_number') }}`. Each PR builds into its own schema that's cleaned up after merge.

---

**Q7: Explain how you'd implement a star schema in dbt for an e-commerce analytics warehouse.**

**Strong Answer:**

```
                    ┌──────────────┐
                    │ dim_products │
                    └──────┬───────┘
                           │
┌──────────────┐    ┌──────┴───────┐    ┌──────────────┐
│ dim_customers├────┤  fct_orders  ├────┤  dim_dates   │
└──────────────┘    └──────┬───────┘    └──────────────┘
                           │
                    ┌──────┴───────┐
                    │ dim_channels │
                    └──────────────┘
```

- **Fact table** (`fct_orders`): grain = one row per order. Contains foreign keys (customer_id, product_id, order_date) and measures (amount, quantity, discount). Materialized as incremental.
- **Dimensions**: `dim_customers` (SCD2 from snapshot), `dim_products` (SCD2 from snapshot), `dim_dates` (generated from `dbt_utils.date_spine`), `dim_channels` (seed or view on source). Materialized as tables.
- **Staging layer**: 1:1 views on raw sources, handling type casting, renaming, and basic cleaning.
- **Intermediate layer**: `int_orders_with_payments` joins orders + payments before the fact table.

---

**Q8: When would you use dbt Mesh (multi-project) vs. a monolith single project?**

**Strong Answer:**

| Factor | Single Project | dbt Mesh |
|--------|---------------|----------|
| Team size | 1-10 engineers | 10+ across domains |
| Model count | < 500 | 500+ |
| Ownership | Shared ownership | Domain ownership needed |
| Deploy cadence | Same schedule | Independent deploys |
| Cross-team deps | Implicit (same DAG) | Explicit (contracts + APIs) |

Use Mesh when teams need independent deployment cycles and clear ownership boundaries. Finance team deploys on their schedule, Marketing on theirs. Cross-project `ref()` with contracts ensures interface stability.

---

## Screen 2: Incremental Model Debugging — "My Model Has a Bug"

These scenarios test your ability to diagnose and fix common incremental model issues. Each is a situation you'll face in production.

---

**Q1: Your incremental `fct_orders` model has duplicate rows. Orders placed yesterday appear twice. What happened?**

**Diagnosis checklist:**
1. **Missing `unique_key`** — Without it, merge behaves like append. Fix: add `unique_key='order_id'`
2. **Wrong `unique_key`** — If you used `customer_id` instead of `order_id`, only one order per customer survives
3. **Upstream duplicates** — `stg_orders` itself has duplicates (source system issue). Fix: deduplicate in staging
4. **Overlapping lookback** + **append strategy** — If using `append` with a lookback window, reprocessed rows get inserted again. Fix: switch to `merge` strategy

**Fix:** Verify `unique_key` matches the grain, ensure staging is deduplicated, use `merge` strategy, then `--full-refresh` to rebuild clean.

---

**Q2: After deploying a logic change to `fct_orders`, new rows use the new formula but historical rows still show the old calculation. How do you fix it?**

**Answer:** This is expected behavior for incremental models — changes only apply to newly processed rows. Fix: `dbt run --select fct_orders --full-refresh`. This drops and rebuilds the table with the new logic applied to ALL rows. Then validate that the full-refresh output matches expectations before the next incremental run resumes.

---

**Q3: Your incremental model's `is_incremental()` block filters on `WHERE updated_at > (SELECT MAX(updated_at) FROM {{ this }})`. Late-arriving orders from 3 days ago are being missed. Why?**

**Answer:** The MAX(updated_at) in the target table is from today's data. Late-arriving orders have `updated_at` from 3 days ago — which is older than MAX. They're filtered out.

**Fix:** Add a lookback window:
```sql
WHERE updated_at >= (
    SELECT DATEADD(day, -3, MAX(updated_at)) FROM {{ this }}
)
```
The `merge` strategy with `unique_key` ensures reprocessed rows update in place (no duplicates).

---

**Q4: You add a new column `discount_amount` to your incremental model. After running, the new column exists but all values are NULL — even for new rows. What's wrong?**

**Answer:** Check `on_schema_change` config:
- If set to `ignore` (default): the new column is silently dropped — it won't be added to the target table
- If set to `append_new_columns`: the column is added, new rows should have values, old rows will be NULL

**Fix:** Set `on_schema_change='append_new_columns'` and re-run. If you need ALL rows to have the new column populated, run `--full-refresh`.

---

**Q5: Your incremental model ran successfully but processed 0 rows. No errors, no data. What do you check?**

**Answer:** Diagnostic steps:
1. **Source freshness** — Is new data arriving? Check `dbt source freshness`
2. **Incremental filter** — Is the WHERE clause too restrictive? Check `SELECT MAX(updated_at) FROM {{ this }}` — if it's in the future (timezone bug), nothing will match
3. **`is_incremental()` behavior** — Is this actually running incrementally? Check that the target table exists
4. **Compiled SQL** — Run `dbt compile --select fct_orders` and inspect the actual WHERE clause in `target/compiled/`
5. **Timezone mismatch** — Source `updated_at` is in UTC but `{{ this }}` has values in local time

---

**Q6: Your model works perfectly with `--full-refresh` but produces different row counts on incremental runs. How do you diagnose this drift?**

**Answer:** This is a classic "incremental/full-refresh parity" bug. Causes:
1. **Late-arriving data** without lookback — some rows are never captured incrementally
2. **Different filter logic** — the `{% if is_incremental() %}` block vs. the full-load block use different WHERE conditions
3. **Source mutations** — rows change after initial load but the incremental filter only captures new rows (no merge on `unique_key`)
4. **Aggregation windows** — incremental processes a slice, but the aggregation depends on the full dataset

**Fix:** Run `dbt_audit_helper.compare_relations` to find the exact rows that differ. Then adjust your incremental logic to match full-refresh results.

---

## Screen 3: Testing & Data Quality Scenarios — "The Dashboard Shows Wrong Numbers"

---

**Q1: An analyst reports that `dim_customers.total_orders` doesn't match the count in `fct_orders` for some customers. How do you investigate?**

**Answer:**
1. Write a singular test that cross-validates:
```sql
-- tests/assert_customer_order_count_consistency.sql
SELECT c.customer_id, c.total_orders, COUNT(o.order_id) AS actual
FROM {{ ref('dim_customers') }} c
LEFT JOIN {{ ref('fct_orders') }} o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.total_orders
HAVING c.total_orders != COUNT(o.order_id)
```
2. Check if `dim_customers` is refreshed on a different schedule than `fct_orders`
3. Check if `dim_customers` filters differ (e.g., excludes cancelled orders while the count includes them)
4. If using incremental, check for drift — run `--full-refresh` on `dim_customers` and compare

---

**Q2: Your `accepted_values` test on `orders.status` starts failing because a new status 'backordered' appeared. What do you do?**

**Answer:**
1. **Immediate**: Change test severity to `warn` (not `error`) so production doesn't halt
2. **Investigate**: Confirm 'backordered' is a legitimate new status from the source system
3. **Update**: Add 'backordered' to the accepted values list
4. **Process**: Ensure any downstream CASE WHEN statements handle the new status (not silently falling through to NULL/other)
5. **Prevent**: Consider using `dbt_utils.get_column_values()` to dynamically fetch statuses instead of hardcoding

---

**Q3: Source freshness shows `raw_orders` last loaded 36 hours ago (error threshold: 24h). Your dbt build is about to run. What do you do?**

**Answer:**
1. **Don't build** — stale source data means models will produce incomplete results
2. **Alert the ingestion team** — Fivetran/Airbyte may have a sync failure
3. **Check the ingestion tool** — Is it a one-time failure or recurring?
4. **Design**: Configure Airflow DAG to check source freshness BEFORE `dbt build` — if error, skip build and alert
5. **Communicate**: Notify stakeholders that today's dashboard data may be stale

---

**Q4: You need to ensure that daily revenue never drops more than 50% compared to the previous day. How do you test this?**

**Answer:**
```sql
-- tests/assert_revenue_no_cliff_drop.sql
{{ config(severity='warn') }}

WITH daily AS (
    SELECT
        DATE(order_date) AS order_day,
        SUM(order_amount_dollars) AS revenue
    FROM {{ ref('fct_orders') }}
    WHERE order_date >= DATEADD(day, -30, CURRENT_DATE())
    GROUP BY DATE(order_date)
),
with_prev AS (
    SELECT
        order_day,
        revenue,
        LAG(revenue) OVER (ORDER BY order_day) AS prev_revenue
    FROM daily
)
SELECT *
FROM with_prev
WHERE prev_revenue IS NOT NULL
  AND revenue < prev_revenue * 0.5  -- More than 50% drop
```

---

**Q5: How do you implement data quality monitoring that goes beyond pass/fail tests?**

**Answer:**
- **`store_failures=true`** — Save failing rows to tables for investigation
- **`warn_if` / `error_if` thresholds** — Nuanced severity based on failure volume
- **Elementary** (`dbt-elementary` package) — Automated anomaly detection, data quality dashboards, Slack alerts
- **Custom monitoring model** — Build a model that tracks daily row counts, null rates, and metric values; alert when anomalies are detected
- **run_results.json parsing** — Track test failure trends over time; a test that intermittently warns may indicate a growing problem

---

**Q6: Your team is debating whether to use dbt tests or warehouse-level constraints (CHECK, NOT NULL). What's your position?**

**Answer:** Use both — they serve different purposes:
- **dbt tests**: Post-load validation, flexible (can test cross-model relationships), severity levels, store_failures. Run after data is loaded.
- **Warehouse constraints**: Prevent bad data from being written in the first place. Enforced at INSERT/MERGE time. Not all warehouses enforce them (BigQuery doesn't enforce CHECK constraints).
- **Model contracts** (dbt 1.5+): Compile-time schema validation — catches structural issues before SQL runs.

Stack them: contracts (compile) → constraints (write) → tests (validate). Defense in depth.

---

## Screen 4: Jinja/Macro Challenges — "Write the Macro"

---

**Q1: Write a macro that generates a `CASE WHEN` statement mapping status codes to human-readable labels.**

```sql
-- macros/status_label.sql
{% macro status_label(column_name) %}
    case {{ column_name }}
        when 'PL' then 'Placed'
        when 'SH' then 'Shipped'
        when 'DL' then 'Delivered'
        when 'RT' then 'Returned'
        when 'CA' then 'Cancelled'
        else 'Unknown'
    end
{% endmacro %}

-- Usage: SELECT {{ status_label('status_code') }} AS status_label
```

**Follow-up**: Make it data-driven using a seed:
```sql
{% macro status_label_dynamic(column_name) %}
    {%- set status_query %}
        select code, label from {{ ref('seed_status_mapping') }}
    {% endset -%}
    {%- set statuses = run_query(status_query) -%}
    case {{ column_name }}
        {% if execute %}
            {% for row in statuses.rows %}
                when '{{ row[0] }}' then '{{ row[1] }}'
            {% endfor %}
        {% endif %}
        else 'Unknown'
    end
{% endmacro %}
```

---

**Q2: Write a macro that generates a `UNION ALL` of monthly partitioned tables named `events_2024_01`, `events_2024_02`, etc.**

```sql
-- macros/union_monthly_tables.sql
{% macro union_monthly_tables(source_name, prefix, start_date, end_date) %}
    {% set start = modules.datetime.datetime.strptime(start_date, '%Y-%m') %}
    {% set end = modules.datetime.datetime.strptime(end_date, '%Y-%m') %}

    {% set tables = [] %}
    {% set current = start %}
    {% for i in range(100) %}  {# Max 100 months #}
        {% if current <= end %}
            {% set month_str = current.strftime('%Y_%m') %}
            {% do tables.append(prefix ~ month_str) %}
            {% if current.month == 12 %}
                {% set current = current.replace(year=current.year + 1, month=1) %}
            {% else %}
                {% set current = current.replace(month=current.month + 1) %}
            {% endif %}
        {% endif %}
    {% endfor %}

    {% for table in tables %}
        select *, '{{ table }}' as source_table
        from {{ source(source_name, table) }}
        {% if not loop.last %}union all{% endif %}
    {% endfor %}
{% endmacro %}

-- Usage: {{ union_monthly_tables('analytics', 'events_', '2024-01', '2024-06') }}
```

---

**Q3: Write a macro that grants SELECT on all models in a schema to a specific role. Use it as a post-hook.**

```sql
-- macros/grant_select.sql
{% macro grant_select(schema_name, role_name) %}
    grant usage on schema {{ target.database }}.{{ schema_name }} to role {{ role_name }};
    grant select on all tables in schema {{ target.database }}.{{ schema_name }} to role {{ role_name }};
    grant select on all views in schema {{ target.database }}.{{ schema_name }} to role {{ role_name }};
{% endmacro %}

-- Usage in dbt_project.yml:
-- on-run-end:
--   - "{{ grant_select('marts', 'analyst_role') }}"
```

---

**Q4: Write a generic test macro that checks if a column's values are monotonically increasing (each row greater than the previous).**

```sql
-- macros/tests/test_is_monotonically_increasing.sql
{% test is_monotonically_increasing(model, column_name, partition_by=none, order_by=none) %}

    {% set order_col = order_by or column_name %}

    with lagged as (
        select
            {{ column_name }},
            lag({{ column_name }}) over (
                {% if partition_by %}partition by {{ partition_by }}{% endif %}
                order by {{ order_col }}
            ) as prev_value
        from {{ model }}
    )

    select *
    from lagged
    where prev_value is not null
      and {{ column_name }} <= prev_value

{% endtest %}

-- Usage in YAML:
-- tests:
--   - is_monotonically_increasing:
--       order_by: created_at
```

---

**Q5: Write a macro that dynamically pivots a column's values into separate columns without hardcoding the values.**

```sql
-- macros/dynamic_pivot.sql
{% macro dynamic_pivot(relation, pivot_column, value_column, agg_function='sum') %}
    {%- set values_query %}
        select distinct {{ pivot_column }}
        from {{ relation }}
        where {{ pivot_column }} is not null
        order by {{ pivot_column }}
    {% endset -%}

    {%- set values = run_query(values_query) -%}

    {% if execute %}
        {% for row in values.rows %}
            {{ agg_function }}(
                case when {{ pivot_column }} = '{{ row[0] }}'
                then {{ value_column }} end
            ) as {{ row[0] | lower | replace(' ', '_') | replace('-', '_') }}
            {% if not loop.last %},{% endif %}
        {% endfor %}
    {% endif %}
{% endmacro %}

-- Usage:
-- SELECT customer_id, {{ dynamic_pivot(ref('stg_payments'), 'method', 'amount') }}
-- FROM {{ ref('stg_payments') }} GROUP BY customer_id
```

---

**Q6: Explain what this Jinja code does and identify the bug:**

```sql
{% set payment_methods = ['credit_card', 'paypal', 'apple_pay'] %}
SELECT
    order_id,
    {% for method in payment_methods %}
        SUM(CASE WHEN payment_method = '{{ method }}' THEN amount ELSE 0 END)
            AS {{ method }}_total,
    {% endfor %}
FROM {{ ref('stg_payments') }}
GROUP BY order_id
```

**Answer:** This generates a SUM column for each payment method. **The bug**: there's a trailing comma after the last `{% endfor %}` — the `,` after `apple_pay_total,` creates invalid SQL (comma before FROM).

**Fix:** Use `{% if not loop.last %},{% endif %}` or move the comma:
```sql
{% for method in payment_methods %}
    SUM(CASE WHEN payment_method = '{{ method }}' THEN amount ELSE 0 END)
        AS {{ method }}_total
    {% if not loop.last %},{% endif %}
{% endfor %}
```

---

## Screen 5: Production Operations — "It's 3am and the Pipeline Broke"

---

**Q1: Your daily dbt build fails at 4am. The Slack alert says `fct_orders` test `unique_order_id` failed with 47 duplicate rows. Walk me through your incident response.**

**Answer:**
1. **Triage severity**: 47 duplicates in a table of millions = low impact but needs fixing
2. **Check downstream**: Are reports/dashboards built on `fct_orders`? Did `dbt build` gate them? (Yes, if using `dbt build` with error severity — downstream was skipped)
3. **Investigate cause**: Query the 47 duplicates — `SELECT order_id, COUNT(*) FROM fct_orders GROUP BY order_id HAVING COUNT(*) > 1`. Are they from the same source batch?
4. **Check source**: Query `stg_orders` for those order_ids — are duplicates coming from the source?
5. **Fix**: If source duplicates — add deduplication logic in `stg_orders` (ROW_NUMBER window function). If merge/incremental bug — check `unique_key` config
6. **Remediate**: Run `--full-refresh` on `fct_orders` after fix, verify duplicates are gone
7. **Prevent**: Add a `unique` test to `stg_orders.order_id` to catch source duplicates earlier

---

**Q2: A model that normally takes 5 minutes suddenly takes 90 minutes. No code changes. What do you investigate?**

**Answer:**
1. **Warehouse load**: Is the warehouse saturated with concurrent queries? Check query queue
2. **Data volume spike**: Did the source table suddenly grow? Check row counts in `stg_` model
3. **Incremental regression**: Is `is_incremental()` returning False? (table was dropped, name changed) → doing full rebuild
4. **Warehouse sizing**: Did someone change the warehouse size? Check Snowflake warehouse config
5. **Upstream changes**: Did a staging model change that returned more rows to the incremental filter?
6. **Query plan**: Check the warehouse query profile — look for spills to disk, bad join strategies
7. **Statistics**: On Snowflake, check if micro-partition pruning is effective — may need clustering keys

---

**Q3: You need to deploy a breaking change — renaming `order_total` to `order_amount_dollars` in `fct_orders`. Three dashboards and an ML pipeline consume this table. How do you handle the rollout?**

**Answer:**
1. **Announce**: Communicate the change with 2-week notice to all consumers
2. **Deprecation period**: Add both columns temporarily:
   ```sql
   order_amount_dollars,
   order_amount_dollars AS order_total  -- DEPRECATED: use order_amount_dollars
   ```
3. **Document**: Add deprecation notice in YAML description
4. **Track usage**: Query warehouse `INFORMATION_SCHEMA.ACCESS_HISTORY` (Snowflake) to see who's still using `order_total`
5. **Remove**: After consumers migrate, remove the deprecated column
6. **Contract**: Add model contract enforcement to prevent future accidental renames

---

**Q4: Your company acquires another business. They have their own data in a separate database. How do you integrate their order data into your existing dbt project?**

**Answer:**
1. **New source**: Define the acquired company's database as a new source in YAML
2. **Staging**: Create `stg_acquired__orders.sql` with column mapping to match your naming conventions
3. **Schema alignment**: Map their columns to your grain and types (currency conversion, status mapping)
4. **Union**: Create an intermediate model `int_orders_combined.sql` that UNIONs both staging models with a `source_system` column
5. **Downstream**: `fct_orders` now reads from `int_orders_combined` instead of `stg_orders` directly
6. **Testing**: Add `accepted_values` tests for the new source_system values, cross-validate row counts
7. **Deploy**: Full-refresh `fct_orders` to include historical acquired data

---

**Q5: You've been asked to reduce the dbt project's warehouse compute costs by 40%. What levers do you pull?**

**Answer:**
1. **Convert tables to views** — staging models don't need to be tables (biggest savings)
2. **Convert large tables to incremental** — stop rebuilding 100M+ row tables daily
3. **Right-size warehouses** — use SMALL for staging, MEDIUM for marts, LARGE only for full refreshes
4. **Auto-suspend warehouses** — 60-second auto-suspend between runs
5. **Eliminate unused models** — query `ACCESS_HISTORY` to find models nobody queries; disable them
6. **Optimize clustering** — add cluster keys to frequently queried large tables to reduce scan
7. **Reduce full-refresh frequency** — weekly instead of daily if data quality allows
8. **Use Slim CI** — stop rebuilding the entire project on every PR

---

**Q6: How do you handle secrets (database passwords, API keys) in dbt?**

**Answer:**
- **Never in code**: No passwords in `profiles.yml`, `dbt_project.yml`, or model files
- **`env_var()`**: Use `{{ env_var('SNOWFLAKE_PASSWORD') }}` in profiles.yml
- **CI/CD**: Store secrets in GitHub Actions secrets, Airflow connections, or a vault
- **`.env` files**: For local dev, use `.env` files (gitignored) with `direnv` or similar
- **Key-pair auth**: Prefer key-pair authentication over password for Snowflake
- **profiles.yml**: Gitignore it entirely; use environment-specific templates

---

## Screen 6: Rapid Fire — 20 Quick Q&A

**1. What does `dbt build` do that `dbt run` doesn't?**
→ Runs models, tests, snapshots, AND seeds in DAG order with test gating. `dbt run` only runs models.

**2. What's the difference between `{{ ref() }}` and `{{ source() }}`?**
→ `ref()` references dbt models (within the project). `source()` references external raw tables (entry point to dbt).

**3. When does `is_incremental()` return False?**
→ First run (table doesn't exist), `--full-refresh` flag, or model not configured as incremental.

**4. What is `{{ this }}`?**
→ A reference to the current model's existing target table. Only meaningful inside `{% if is_incremental() %}`.

**5. Name the 4 built-in dbt tests.**
→ `unique`, `not_null`, `accepted_values`, `relationships`.

**6. What does `severity: warn` do vs `severity: error`?**
→ `warn` logs the failure but allows downstream models to proceed. `error` halts downstream execution in `dbt build`.

**7. What's the difference between `snapshot` timestamp and check strategies?**
→ `timestamp` compares `updated_at` column (faster). `check` compares actual column values (no timestamp needed).

**8. How do you identify the current record in a snapshot table?**
→ `WHERE dbt_valid_to IS NULL`.

**9. What file stores the DAG/project graph?**
→ `target/manifest.json`.

**10. What does `dbt build --select state:modified+` do?**
→ Builds only models that changed since the last production run, plus all downstream dependencies.

**11. What's the `+` operator in selectors?**
→ `model+` = model and all downstream. `+model` = model and all upstream. `+model+` = both.

**12. What does `store_failures=true` do?**
→ Saves failing test rows to a table in the warehouse for investigation.

**13. How do you mask PII in dev but keep it in prod?**
→ `{% if target.name == 'prod' %}customer_email{% else %}MD5(customer_email){% endif %}`.

**14. What does `dbt deps` do?**
→ Installs packages defined in `packages.yml` into the `dbt_packages/` directory.

**15. What's the purpose of `generate_schema_name` macro?**
→ Controls how dbt builds schema names. Override it so dev uses `dev_alice_marts` while prod uses `marts`.

**16. Name three must-have dbt packages.**
→ `dbt_utils`, `dbt_expectations`, `codegen`.

**17. What's a model contract?**
→ Enforced schema definition (column names, types, constraints) that must match the model's output. Compile-time check.

**18. What does `on_schema_change='fail'` do?**
→ Raises an error when an incremental model's SELECT returns different columns than the existing target table.

**19. How do you run a one-time backfill on an incremental model?**
→ `dbt run --select my_model --vars '{"backfill_start": "2024-01-01", "backfill_end": "2024-01-31"}'` with var-driven WHERE clause. Or `--full-refresh` for complete rebuild.

**20. What's the difference between `dbt compile` and `dbt run`?**
→ `compile` resolves Jinja and writes compiled SQL to `target/compiled/` without executing anything. `run` compiles AND executes against the warehouse.

---

### 🔑 Key Takeaways

1. **Architecture questions** test your ability to structure projects, choose materializations, and justify tradeoffs
2. **Debugging incrementals** always comes back to: unique_key, incremental filter, lookback window, and full-refresh parity
3. **Data quality scenarios** require knowing tests, severity, store_failures, source freshness, and cross-model validation
4. **Jinja/macro questions** test `{% for %}` loops (trailing commas!), `run_query()` with `execute` guards, and DRY patterns
5. **Production operations** require incident response skills: triage, investigate, fix, remediate, prevent
6. **Rapid-fire questions** are about vocabulary and core concepts — know the dbt glossary cold
7. **Always explain the "why"** — interviewers care more about reasoning than memorized syntax
8. **Real-world experience wins** — reference specific scenarios, table sizes, and timing from your projects
9. **Tradeoffs matter** — every decision has a cost; articulate what you're gaining and what you're giving up
10. **Stay current** — know dbt 1.5+ features (contracts, Mesh), Slim CI, Cosmos — these signal seniority
