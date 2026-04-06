# Module 6: Macros, Jinja & Packages

## Screen 1: Jinja in SQL — The Template Engine That Powers dbt

Every `.sql` file in dbt is a **Jinja template**, not plain SQL. Before dbt sends anything to the warehouse, the Jinja engine processes your file — resolving variables, executing control flow, and expanding macros into raw SQL. Understanding Jinja is what separates someone who *uses* dbt from someone who *masters* it.

**Three Jinja delimiters:**

```
┌──────────────────────────────────────────────────────────────────┐
│           Jinja Syntax — The Three Delimiters                    │
│                                                                  │
│  {{ expression }}     → Output: evaluates and prints the result  │
│                         Example: {{ ref('stg_orders') }}         │
│                         Renders: ANALYTICS_DB.staging.stg_orders │
│                                                                  │
│  {% statement %}      → Logic: control flow, no output           │
│                         Example: {% if target.name == 'prod' %}  │
│                         Used for: if/else, for loops, set vars   │
│                                                                  │
│  {# comment #}        → Comment: completely stripped from output  │
│                         Example: {# TODO: add discount logic #}  │
│                         Not visible in compiled SQL               │
└──────────────────────────────────────────────────────────────────┘
```

**Real example — Jinja template vs compiled SQL:**

```sql
-- models/marts/fct_orders.sql (Jinja template)

{{ config(materialized='table') }}

{# This model builds the core orders fact table #}

SELECT
    order_id,
    customer_id,
    order_date,
    status,
    order_amount_cents / 100.0 AS order_amount_dollars,
    {% if target.name == 'prod' %}
        -- In production, include PII
        customer_email,
    {% else %}
        -- In dev, mask PII
        MD5(customer_email) AS customer_email,
    {% endif %}
    CURRENT_TIMESTAMP() AS dbt_loaded_at
FROM {{ ref('stg_orders') }}
WHERE status != 'test'
```

**Compiled SQL (what the warehouse actually sees):**

```sql
-- Compiled for dev environment
SELECT
    order_id,
    customer_id,
    order_date,
    status,
    order_amount_cents / 100.0 AS order_amount_dollars,
    MD5(customer_email) AS customer_email,
    CURRENT_TIMESTAMP() AS dbt_loaded_at
FROM ANALYTICS_DEV.staging.stg_orders
WHERE status != 'test'
```

Notice: the `{# comment #}` is gone, the `{% if %}` block resolved to the `else` branch (dev environment), and `{{ ref() }}` resolved to the full table path.

**Viewing compiled SQL:**

```bash
# Compile without running — see the generated SQL
$ dbt compile --select fct_orders

# Compiled output is in: target/compiled/your_project/models/marts/fct_orders.sql
```

**Key Jinja variables available in dbt:**

| Variable | What It Contains | Example Use |
|----------|-----------------|-------------|
| `{{ ref('model') }}` | Resolved table reference | `FROM {{ ref('stg_orders') }}` |
| `{{ source('src', 'tbl') }}` | Resolved source reference | `FROM {{ source('ecom', 'raw_orders') }}` |
| `{{ this }}` | Current model's table | `SELECT MAX(id) FROM {{ this }}` |
| `{{ target.name }}` | Environment name | `{% if target.name == 'prod' %}` |
| `{{ target.schema }}` | Target schema | Schema-based logic |
| `{{ var('my_var') }}` | Project variable | `{{ var('lookback_days', 3) }}` |
| `{{ env_var('API_KEY') }}` | Environment variable | Secrets, config |

### 💡 Interview Insight

> **Common Question**: "Explain how Jinja works in dbt."
> **Strong Answer**: "Every dbt SQL file is a Jinja template processed before execution. There are three delimiters: `{{ }}` for expressions that output values (like `ref()` and `source()`), `{% %}` for control flow (if/else, for loops) that doesn't produce output, and `{# #}` for comments stripped from compiled SQL. I use `dbt compile` to inspect the generated SQL during development. Key use cases include environment-specific logic with `target.name`, dynamic SQL generation with for loops, and reusable logic via macros."

---

## Screen 2: Built-in Macros — `ref()`, `source()`, `config()`

dbt ships with several macros that form the backbone of every project. These aren't just convenience functions — they're what makes dbt's DAG, documentation, and environment awareness possible.

**`{{ ref('model_name') }}`** — The heart of dbt:

```sql
-- What ref() does:
-- 1. Resolves the model name to its full table/view path
-- 2. Registers a dependency in the DAG
-- 3. Handles environment-specific schemas

SELECT * FROM {{ ref('stg_orders') }}
-- Dev:  → SELECT * FROM DEV_SCHEMA.stg_orders
-- Prod: → SELECT * FROM ANALYTICS.staging.stg_orders

-- Cross-project ref (dbt Mesh):
SELECT * FROM {{ ref('other_project', 'shared_model') }}
```

**`{{ source('source_name', 'table_name') }}`** — External data entry point:

```sql
-- Declares the ingestion boundary — raw data enters here
SELECT * FROM {{ source('ecommerce', 'raw_orders') }}
-- Resolves to: RAW_DB.shopify.raw_orders (based on YAML config)

-- source() also:
-- 1. Enables source freshness checks
-- 2. Registers the source in the DAG
-- 3. Creates documentation lineage from source → staging
```

**`{{ config() }}`** — Model configuration:

```sql
-- config() can be in the SQL file or in dbt_project.yml
{{ config(
    materialized='incremental',
    unique_key='order_id',
    tags=['finance', 'critical'],
    schema='marts',
    pre_hook="ALTER SESSION SET TIMEZONE = 'UTC'",
    post_hook="GRANT SELECT ON {{ this }} TO ROLE analyst"
) }}
```

**`{{ var('variable_name', default) }}`** — Project variables:

```sql
-- Use variables for parameterized runs
{% set lookback = var('lookback_days', 3) %}

SELECT *
FROM {{ ref('stg_orders') }}
WHERE order_date >= DATEADD(day, -{{ lookback }}, CURRENT_DATE())
```

```bash
# Override at runtime
$ dbt run --vars '{"lookback_days": 7}'
```

**`{{ target }}`** — Environment context object:

```sql
-- target contains environment information from profiles.yml
{% if target.name == 'prod' %}
    -- Production: use production database
    {{ config(database='ANALYTICS_PROD') }}
{% elif target.name == 'ci' %}
    -- CI: use isolated schema
    {{ config(schema='pr_' ~ var('pr_number', 'local')) }}
{% else %}
    -- Dev: default personal schema
{% endif %}
```

```
┌────────────────────────────────────────────────────────────┐
│  target Object Properties                                  │
├────────────────────┬───────────────────────────────────────┤
│ target.name        │ Profile target name (dev/prod/ci)     │
│ target.schema      │ Default schema                        │
│ target.database    │ Default database                      │
│ target.type        │ Adapter type (snowflake/bigquery/etc) │
│ target.threads     │ Number of parallel threads            │
│ target.profile_name│ Profile name from profiles.yml        │
└────────────────────┴───────────────────────────────────────┘
```

### 💡 Interview Insight

> **Common Question**: "What does `ref()` do beyond resolving table names?"
> **Strong Answer**: "Three things: (1) It resolves the model to its environment-specific full path — different schemas in dev vs prod. (2) It registers a dependency in the DAG — dbt knows `fct_orders` depends on `stg_orders` because of the `ref()` call. This controls execution order and enables `dbt build --select stg_orders+` to select downstream models. (3) It creates lineage documentation — `dbt docs generate` builds the lineage graph from `ref()` calls."

---

## Screen 3: Custom Macros — Reusable SQL Functions

Macros are Jinja functions defined in `.sql` files in the `macros/` directory. They accept arguments and return SQL strings — the dbt equivalent of Python functions.

**Basic macro structure:**

```sql
-- macros/cents_to_dollars.sql

{% macro cents_to_dollars(column_name, decimal_places=2) %}
    ROUND({{ column_name }} / 100.0, {{ decimal_places }})
{% endmacro %}
```

**Using the macro in a model:**

```sql
-- models/staging/stg_orders.sql
SELECT
    order_id,
    customer_id,
    {{ cents_to_dollars('order_amount_cents') }}       AS order_amount_dollars,
    {{ cents_to_dollars('shipping_cost_cents', 2) }}   AS shipping_cost_dollars,
    {{ cents_to_dollars('tax_cents') }}                 AS tax_dollars
FROM {{ source('ecommerce', 'raw_orders') }}
```

**Compiled SQL:**

```sql
SELECT
    order_id,
    customer_id,
    ROUND(order_amount_cents / 100.0, 2)   AS order_amount_dollars,
    ROUND(shipping_cost_cents / 100.0, 2)  AS shipping_cost_dollars,
    ROUND(tax_cents / 100.0, 2)            AS tax_dollars
FROM RAW_DB.ecommerce.raw_orders
```

**More practical macros for an e-commerce project:**

```sql
-- macros/generate_schema_name.sql
-- Override dbt's default schema naming behavior

{% macro generate_schema_name(custom_schema_name, node) -%}
    {%- set default_schema = target.schema -%}
    {%- if custom_schema_name is none -%}
        {{ default_schema }}
    {%- elif target.name == 'prod' -%}
        {# In prod, use the custom schema directly #}
        {{ custom_schema_name | trim }}
    {%- else -%}
        {# In dev, prefix with personal schema #}
        {{ default_schema }}_{{ custom_schema_name | trim }}
    {%- endif -%}
{%- endmacro %}
```

```sql
-- macros/safe_divide.sql
-- Prevent division by zero errors

{% macro safe_divide(numerator, denominator, default=0) %}
    CASE
        WHEN {{ denominator }} = 0 OR {{ denominator }} IS NULL
        THEN {{ default }}
        ELSE {{ numerator }}::FLOAT / {{ denominator }}
    END
{% endmacro %}
```

```sql
-- macros/date_spine.sql
-- Generate a column-level date filter with configurable lookback

{% macro limit_date_range(date_column, lookback_days=90) %}
    {{ date_column }} >= DATEADD(day, -{{ lookback_days }}, CURRENT_DATE())
    AND {{ date_column }} < CURRENT_DATE()
{% endmacro %}

-- Usage: WHERE {{ limit_date_range('order_date', 30) }}
```

```sql
-- macros/star_except.sql
-- Select all columns except specified ones

{% macro star_except(relation, except_columns) %}
    {%- set columns = adapter.get_columns_in_relation(relation) -%}
    {%- for col in columns if col.name | lower not in except_columns | map('lower') | list -%}
        {{ col.name }}{% if not loop.last %}, {% endif %}
    {%- endfor -%}
{% endmacro %}

-- Usage: SELECT {{ star_except(ref('stg_orders'), ['_fivetran_synced', '_row_id']) }}
```

**Macro organization:**

```
macros/
├── _macros.yml              # Documentation for macros
├── generate_schema_name.sql # Schema override (dbt convention)
├── grants.sql               # Post-hook grant macros
├── transformations/
│   ├── cents_to_dollars.sql
│   ├── safe_divide.sql
│   └── date_helpers.sql
└── testing/
    ├── test_row_count.sql
    └── test_freshness.sql
```

### 💡 Interview Insight

> **Common Question**: "Give me an example of a custom macro you've written."
> **Strong Answer**: "I wrote a `cents_to_dollars` macro that standardizes currency conversion across all models — `ROUND(column / 100.0, 2)`. Every model that deals with money uses it, so if we ever change the rounding logic (say, to banker's rounding), we update one file. I also wrote a `safe_divide` macro to prevent division-by-zero errors in metrics calculations, and a `generate_schema_name` override so dev uses prefixed schemas while prod uses clean schema names."

---

## Screen 4: Control Flow — `{% if %}`, `{% for %}`, `{% set %}`

Jinja's control flow statements let you generate dynamic SQL — conditionally including columns, looping over lists to build CASE statements, and setting variables for reuse.

**`{% if %}` — Conditional SQL generation:**

```sql
-- models/marts/fct_orders.sql
{{ config(materialized='table') }}

SELECT
    order_id,
    customer_id,
    order_date,
    status,
    order_amount_dollars,

    {% if target.name == 'prod' %}
        customer_email,
        shipping_address,
    {% else %}
        MD5(customer_email)       AS customer_email,
        'REDACTED'                AS shipping_address,
    {% endif %}

    {% if var('include_returns', false) %}
        return_date,
        return_reason,
    {% endif %}

    CURRENT_TIMESTAMP() AS dbt_loaded_at
FROM {{ ref('stg_orders') }}
```

**`{% for %}` — Looping to generate repetitive SQL:**

```sql
-- Generate multiple metric columns dynamically
{% set statuses = ['placed', 'shipped', 'completed', 'returned', 'cancelled'] %}

SELECT
    customer_id,
    COUNT(*) AS total_orders,

    {% for status in statuses %}
        SUM(CASE WHEN status = '{{ status }}' THEN 1 ELSE 0 END)
            AS {{ status }}_count
        {% if not loop.last %},{% endif %}
    {% endfor %}

FROM {{ ref('stg_orders') }}
GROUP BY customer_id
```

**Compiled SQL:**

```sql
SELECT
    customer_id,
    COUNT(*) AS total_orders,
    SUM(CASE WHEN status = 'placed' THEN 1 ELSE 0 END) AS placed_count,
    SUM(CASE WHEN status = 'shipped' THEN 1 ELSE 0 END) AS shipped_count,
    SUM(CASE WHEN status = 'completed' THEN 1 ELSE 0 END) AS completed_count,
    SUM(CASE WHEN status = 'returned' THEN 1 ELSE 0 END) AS returned_count,
    SUM(CASE WHEN status = 'cancelled' THEN 1 ELSE 0 END) AS cancelled_count
FROM ANALYTICS.staging.stg_orders
GROUP BY customer_id
```

**`{% set %}` — Variables and complex expressions:**

```sql
-- Set a variable for reuse
{% set payment_methods = ['credit_card', 'paypal', 'apple_pay', 'gift_card'] %}
{% set lookback_days = var('lookback_days', 30) %}

SELECT
    order_date,
    {% for method in payment_methods %}
        SUM(CASE WHEN payment_method = '{{ method }}'
            THEN amount ELSE 0 END) AS {{ method }}_revenue
        {% if not loop.last %},{% endif %}
    {% endfor %}
FROM {{ ref('stg_payments') }}
WHERE order_date >= DATEADD(day, -{{ lookback_days }}, CURRENT_DATE())
GROUP BY order_date
```

**`loop` special variables inside `{% for %}`:**

```
┌────────────────────────────────────────────────────┐
│  Jinja Loop Variables                              │
├──────────────────┬─────────────────────────────────┤
│ loop.index       │ Current iteration (1-indexed)   │
│ loop.index0      │ Current iteration (0-indexed)   │
│ loop.first       │ True if first iteration         │
│ loop.last        │ True if last iteration          │
│ loop.length      │ Total number of iterations      │
│ loop.revindex    │ Iterations remaining (1-indexed)│
└──────────────────┴─────────────────────────────────┘
```

**Advanced pattern — dynamic UNION ALL:**

```sql
-- Union multiple monthly tables dynamically
{% set months = ['2024_01', '2024_02', '2024_03', '2024_04'] %}

{% for month in months %}
    SELECT
        event_id,
        user_id,
        event_type,
        event_timestamp,
        '{{ month }}' AS source_month
    FROM {{ source('events', 'events_' ~ month) }}
    {% if not loop.last %}UNION ALL{% endif %}
{% endfor %}
```

### 💡 Interview Insight

> **Common Question**: "How do you avoid repetitive SQL in dbt?"
> **Strong Answer**: "Three techniques: (1) `{% for %}` loops — I generate CASE WHEN statements for multiple statuses or payment methods from a list, so adding a new status means adding one string to the list, not copying 3 lines of SQL. (2) Custom macros — reusable functions like `cents_to_dollars()` or `safe_divide()` that standardize common patterns. (3) The `{% if %}` block for environment-specific logic like masking PII in dev but including it in prod."

---

## Screen 5: `adapter.dispatch()` — Cross-Database Compatibility

When your dbt project needs to run on multiple warehouses (Snowflake AND BigQuery, or Snowflake AND Postgres for testing), function syntax differs. `adapter.dispatch()` routes to the correct implementation based on the active adapter.

```sql
-- macros/date_trunc_month.sql

{% macro date_trunc_month(date_column) %}
    {{ return(adapter.dispatch('date_trunc_month')(date_column)) }}
{% endmacro %}

-- Snowflake implementation
{% macro snowflake__date_trunc_month(date_column) %}
    DATE_TRUNC('month', {{ date_column }})
{% endmacro %}

-- BigQuery implementation
{% macro bigquery__date_trunc_month(date_column) %}
    DATE_TRUNC({{ date_column }}, MONTH)
{% endmacro %}

-- PostgreSQL implementation (also used for DuckDB testing)
{% macro postgres__date_trunc_month(date_column) %}
    DATE_TRUNC('month', {{ date_column }}::DATE)
{% endmacro %}

-- Default fallback
{% macro default__date_trunc_month(date_column) %}
    DATE_TRUNC('month', {{ date_column }})
{% endmacro %}
```

**Usage in models (adapter-agnostic):**

```sql
SELECT
    {{ date_trunc_month('order_date') }} AS order_month,
    COUNT(*) AS total_orders,
    SUM(order_amount_dollars) AS monthly_revenue
FROM {{ ref('stg_orders') }}
GROUP BY {{ date_trunc_month('order_date') }}
```

**How dispatch works:**

```
┌────────────────────────────────────────────────────────┐
│  adapter.dispatch('date_trunc_month') Flow             │
│                                                        │
│  1. Check: does snowflake__date_trunc_month exist?     │
│     └── If adapter is Snowflake: YES → use it          │
│  2. Check: does bigquery__date_trunc_month exist?      │
│     └── If adapter is BigQuery: YES → use it           │
│  3. Fallback: use default__date_trunc_month            │
│                                                        │
│  Naming convention: {adapter}__macro_name               │
│  snowflake__ | bigquery__ | postgres__ | default__     │
└────────────────────────────────────────────────────────┘
```

**Package dispatch** — overriding macros from packages:

```yaml
# dbt_project.yml
dispatch:
  - macro_namespace: dbt_utils
    search_order:
      - my_project        # Check my project first
      - dbt_utils          # Then the package
```

This lets you override a `dbt_utils` macro with your own implementation while still falling back to the package version for macros you haven't overridden.

### 💡 Interview Insight

> **Common Question**: "How do you handle SQL syntax differences across warehouses?"
> **Strong Answer**: "With `adapter.dispatch()`. I write a base macro that delegates to adapter-specific implementations: `snowflake__macro_name`, `bigquery__macro_name`, etc. The macro automatically routes to the correct implementation based on the active adapter. This is especially useful for date functions, string functions, and type casting that differ across platforms. I also use the dispatch search order to override package macros when needed."

---

## Screen 6: dbt_utils — The Essential Package

`dbt_utils` is the most widely used dbt package — it provides battle-tested macros that every production project needs. Think of it as the `lodash` of dbt.

**Installation:**

```yaml
# packages.yml
packages:
  - package: dbt-labs/dbt_utils
    version: [">=1.1.0", "<2.0.0"]
```

```bash
$ dbt deps   # Install packages
```

**Top macros from dbt_utils:**

**`generate_surrogate_key`** — Create deterministic composite keys:

```sql
SELECT
    {{ dbt_utils.generate_surrogate_key(['order_id', 'line_item_id']) }}
        AS order_line_key,
    order_id,
    line_item_id,
    product_id,
    quantity
FROM {{ ref('stg_order_items') }}

-- Compiles to: MD5(CAST(order_id AS VARCHAR) || '-' || CAST(line_item_id AS VARCHAR))
```

**`pivot`** — Pivot rows to columns:

```sql
SELECT
    customer_id,
    {{ dbt_utils.pivot(
        column='status',
        values=['placed', 'shipped', 'completed', 'returned'],
        agg='COUNT',
        then_value=1,
        else_value=0,
        quote_identifiers=false
    ) }}
FROM {{ ref('stg_orders') }}
GROUP BY customer_id

-- Generates:
-- COUNT(CASE WHEN status = 'placed' THEN 1 ELSE 0 END) AS placed,
-- COUNT(CASE WHEN status = 'shipped' THEN 1 ELSE 0 END) AS shipped,
-- ...
```

**`union_relations`** — Union multiple tables with schema alignment:

```sql
{{ dbt_utils.union_relations(
    relations=[
        ref('stg_orders_us'),
        ref('stg_orders_eu'),
        ref('stg_orders_apac')
    ],
    include=['order_id', 'customer_id', 'order_date', 'amount'],
    source_column_name='region_source'
) }}
-- Handles mismatched columns, adds NULLs for missing columns,
-- and adds a source identifier column
```

**`date_spine`** — Generate a continuous date series:

```sql
{{ dbt_utils.date_spine(
    datepart="day",
    start_date="CAST('2023-01-01' AS DATE)",
    end_date="CURRENT_DATE()"
) }}
-- Generates one row per day — perfect for filling date gaps
```

**`get_column_values`** — Dynamically fetch distinct values:

```sql
-- Dynamically get all payment methods from the data
{% set payment_methods = dbt_utils.get_column_values(
    table=ref('stg_payments'),
    column='payment_method'
) %}

SELECT
    order_id,
    {% for method in payment_methods %}
        SUM(CASE WHEN payment_method = '{{ method }}'
            THEN amount ELSE 0 END) AS {{ method }}_amount
        {% if not loop.last %},{% endif %}
    {% endfor %}
FROM {{ ref('stg_payments') }}
GROUP BY order_id
```

**Complete dbt_utils cheat sheet:**

```
┌──────────────────────────────────────────────────────────────┐
│  dbt_utils — Most Used Macros                                │
├───────────────────────────┬──────────────────────────────────┤
│ generate_surrogate_key    │ Create composite hash keys       │
│ pivot / unpivot           │ Row ↔ column transformations     │
│ union_relations           │ Union tables, align schemas      │
│ date_spine                │ Generate continuous dates        │
│ get_column_values         │ Fetch distinct values at compile │
│ star                      │ SELECT * with exclusions         │
│ safe_add / safe_divide    │ Null-safe arithmetic             │
│ deduplicate               │ Remove duplicates by key         │
│ get_relations_by_pattern  │ Find tables by naming pattern    │
│ type_timestamp            │ Cross-DB timestamp casting       │
│ log                       │ Debug logging during compilation │
└───────────────────────────┴──────────────────────────────────┘
```

### 💡 Interview Insight

> **Common Question**: "What dbt packages do you use and why?"
> **Strong Answer**: "`dbt_utils` is non-negotiable — I use `generate_surrogate_key` for composite keys, `union_relations` for multi-region table unions with schema alignment, and `date_spine` for building date dimension tables. I also use `dbt_expectations` for advanced testing (value ranges, regex, row counts) and `dbt_audit_helper` for comparing model results during refactors. The key is that packages provide tested, community-maintained macros — DRYer and more reliable than writing your own."

---

## Screen 7: `packages.yml` and `dbt deps` — Dependency Management

Packages extend dbt with reusable macros, models, and tests. Managing them properly is essential for a healthy project.

**`packages.yml` — Package specification:**

```yaml
# packages.yml (lives at project root)
packages:
  # PyPI-style versioned packages from dbt Hub
  - package: dbt-labs/dbt_utils
    version: [">=1.1.0", "<2.0.0"]

  - package: calogica/dbt_expectations
    version: [">=0.10.0", "<0.11.0"]

  - package: dbt-labs/dbt_audit_helper
    version: [">=0.9.0", "<1.0.0"]

  # Git-based package (for internal/private packages)
  - git: "https://github.com/your-org/dbt-internal-macros.git"
    revision: v1.2.0    # Pin to a tag or commit — NEVER use main/master

  # Local package (for monorepo setups)
  - local: ../shared-dbt-macros
```

**Installing packages:**

```bash
# Install all packages from packages.yml
$ dbt deps

# Packages are installed to: dbt_packages/ (gitignored by default)
```

**Key packages for production dbt:**

| Package | Purpose | Must-Have? |
|---------|---------|-----------|
| `dbt_utils` | Core utility macros | ✅ Yes |
| `dbt_expectations` | Advanced data tests | ✅ Yes |
| `dbt_audit_helper` | Model comparison/auditing | Recommended |
| `dbt_date` | Date dimension generation | Nice to have |
| `codegen` | Auto-generate YAML/SQL | Dev productivity |
| `dbt_project_evaluator` | Best practice linting | Recommended |
| `re_data` | Data observability | For monitoring |

**`codegen` — Generate boilerplate automatically:**

```bash
# Generate a base model SQL from a source table
$ dbt run-operation generate_base_model \
    --args '{"source_name": "ecommerce", "table_name": "raw_orders"}'

# Generate YAML schema for an existing model
$ dbt run-operation generate_model_yaml \
    --args '{"model_names": ["stg_orders"]}'
```

**`dbt_project_evaluator` — Lint your project structure:**

```yaml
# packages.yml
packages:
  - package: dbt-labs/dbt_project_evaluator
    version: [">=0.8.0", "<1.0.0"]
```

```bash
$ dbt build --select package:dbt_project_evaluator
# Checks for: models without tests, missing documentation,
# direct source references in marts, naming convention violations
```

**Version pinning best practices:**

```yaml
# ✅ GOOD — Pin to a range (allows patches)
- package: dbt-labs/dbt_utils
  version: [">=1.1.0", "<2.0.0"]

# ✅ GOOD — Pin to exact version (maximum reproducibility)
- package: dbt-labs/dbt_utils
  version: 1.1.1

# ❌ BAD — No version constraint (breaks on updates)
- package: dbt-labs/dbt_utils

# ❌ BAD — Git ref to a branch (not reproducible)
- git: "https://github.com/org/pkg.git"
  revision: main
```

### 💡 Interview Insight

> **Common Question**: "How do you manage dbt packages and avoid breaking changes?"
> **Strong Answer**: "I specify version ranges in `packages.yml` — like `>=1.1.0, <2.0.0` — to allow patch updates while preventing major version breaks. For git-based internal packages, I pin to tags, never branches. Packages install to `dbt_packages/` which is gitignored — they're fetched fresh with `dbt deps` in CI. I run `dbt_project_evaluator` to lint project structure and catch common anti-patterns like missing tests or direct source references in marts."

---

## Screen 8: Advanced Macro Patterns — run_query, execute, and Materialization Overrides

For power users, dbt macros can query the warehouse at compile time, run administrative operations, and even override how materializations work.

**`run_query()` — Execute SQL at compile time:**

```sql
-- macros/get_max_date.sql
-- Query the warehouse during compilation to get a value

{% macro get_max_order_date() %}
    {% set query %}
        SELECT MAX(order_date)::VARCHAR AS max_date
        FROM {{ ref('fct_orders') }}
    {% endset %}

    {% set results = run_query(query) %}

    {% if execute %}
        {% set max_date = results.columns[0].values()[0] %}
        {{ return(max_date) }}
    {% else %}
        {{ return('2020-01-01') }}
    {% endif %}
{% endmacro %}
```

**The `execute` variable**: During `dbt compile`, Jinja processes templates but doesn't actually run queries. The `execute` variable is `True` only during `dbt run` / `dbt build`. Always guard `run_query()` calls with `{% if execute %}` to prevent errors during compile-only phases.

**`run_query` for dynamic model generation:**

```sql
-- models/marts/fct_orders_by_country.sql
-- Dynamically generate columns based on actual data

{% set country_query %}
    SELECT DISTINCT shipping_country
    FROM {{ ref('stg_orders') }}
    ORDER BY shipping_country
{% endset %}

{% set countries = run_query(country_query) %}

SELECT
    order_date,
    {% if execute %}
        {% for country in countries.columns[0].values() %}
            SUM(CASE WHEN shipping_country = '{{ country }}'
                THEN order_amount_dollars ELSE 0 END)
                AS {{ country | lower | replace(' ', '_') }}_revenue
            {% if not loop.last %},{% endif %}
        {% endfor %}
    {% else %}
        NULL AS placeholder
    {% endif %}
FROM {{ ref('stg_orders') }}
GROUP BY order_date
```

**`on-run-start` / `on-run-end` hooks** — Run SQL before/after dbt:

```yaml
# dbt_project.yml
on-run-start:
  - "ALTER SESSION SET TIMEZONE = 'UTC'"
  - "{{ create_audit_schema() }}"

on-run-end:
  - "{{ grant_select_on_schemas(['staging', 'marts'], 'analyst_role') }}"
  - "{{ log_run_results() }}"
```

**Custom operation macros** — Runnable scripts:

```sql
-- macros/operations/drop_old_models.sql
{% macro drop_old_models(days=30) %}
    {% set query %}
        SELECT table_name
        FROM information_schema.tables
        WHERE table_schema = '{{ target.schema }}'
          AND last_altered < DATEADD(day, -{{ days }}, CURRENT_TIMESTAMP())
    {% endset %}

    {% set results = run_query(query) %}
    {% if execute %}
        {% for table in results.columns[0].values() %}
            {% set drop_sql = "DROP TABLE IF EXISTS " ~ target.schema ~ "." ~ table %}
            {{ log("Dropping: " ~ table, info=true) }}
            {% do run_query(drop_sql) %}
        {% endfor %}
    {% endif %}
{% endmacro %}
```

```bash
# Run as an operation (not a model)
$ dbt run-operation drop_old_models --args '{"days": 60}'
```

### 💡 Interview Insight

> **Common Question**: "What is `run_query()` and when would you use it?"
> **Strong Answer**: "run_query() executes SQL against the warehouse at compile time and returns results as a Jinja object. I use it for dynamic SQL generation — like fetching distinct country values to build pivot columns automatically. Critical caveat: always guard with `{% if execute %}` because during `dbt compile` (parse phase), the query can't actually run. I also use it in operation macros for administrative tasks like granting permissions or cleaning up old tables."

---

## Screen 9: Quiz — Module 6 Review

### Question 1
**What are the three Jinja delimiters and their purposes?**

A) `{{ }}` for comments, `{% %}` for output, `{# #}` for logic
B) `{{ }}` for output/expressions, `{% %}` for control flow/logic, `{# #}` for comments
C) `{{ }}` for variables, `{% %}` for macros, `{# #}` for SQL
D) All three produce SQL output

**Answer: B** — `{{ }}` evaluates expressions and outputs the result (like `ref()`, variables). `{% %}` executes logic without output (if/else, for loops, set). `{# #}` creates comments that are completely stripped from compiled SQL.

---

### Question 2
**What does `ref()` do BEYOND resolving the table name?**

A) Nothing — it only resolves names
B) It registers a DAG dependency and creates documentation lineage
C) It runs the referenced model first
D) It validates the referenced model's schema

**Answer: B** — `ref()` does three things: resolves the model to its environment-specific table path, registers a dependency in the DAG (so dbt knows execution order), and creates lineage in documentation. It does NOT trigger the model to run — DAG ordering handles that.

---

### Question 3
**You have 5 status values and need to generate a CASE WHEN for each. What's the most Pythonic dbt approach?**

A) Write out all 5 CASE WHENs manually
B) Use a `{% for %}` loop over a `{% set statuses = [...] %}` list
C) Create a separate model for each status
D) Use a `{% if %}` chain for each status

**Answer: B** — Define the list with `{% set statuses = ['placed', 'shipped', ...] %}`, then loop with `{% for status in statuses %}` to generate the CASE WHENs. Adding a new status means adding one string to the list — DRY principle in action.

---

### Question 4
**What happens if you call `run_query()` without guarding it with `{% if execute %}`?**

A) Nothing — it works fine
B) It may fail during `dbt compile` or parse phase when no warehouse connection exists
C) It runs twice
D) It returns an empty result

**Answer: B** — During parsing (`dbt compile`, `dbt ls`), dbt processes Jinja but doesn't connect to the warehouse. Without the `{% if execute %}` guard, `run_query()` would attempt to connect and fail. The `execute` variable is `True` only during `dbt run` / `dbt build`.

---

### Question 5
**What is `adapter.dispatch()` used for?**

A) Sending data to an external API
B) Routing to adapter-specific macro implementations for cross-database compatibility
C) Dispatching dbt runs to multiple warehouses simultaneously
D) Debugging macro execution

**Answer: B** — `adapter.dispatch()` lets you write a single macro call that routes to `snowflake__`, `bigquery__`, `postgres__`, or `default__` implementations. This enables cross-database compatibility without `{% if target.type %}` chains in every model.

---

### 🔑 Key Takeaways

1. **Every dbt SQL file is a Jinja template** — processed before SQL execution with `{{ }}`, `{% %}`, `{# #}`
2. **`ref()` does three things**: resolves table paths, registers DAG dependencies, creates documentation lineage
3. **Custom macros** live in `macros/` — use them for DRY code: `cents_to_dollars()`, `safe_divide()`, etc.
4. **`{% for %}` loops** eliminate repetitive SQL — generate CASE WHENs, pivot columns, UNION ALLs dynamically
5. **`{% if target.name %}` enables environment-specific logic** — mask PII in dev, include it in prod
6. **`adapter.dispatch()`** provides cross-database compatibility via adapter-specific implementations
7. **`dbt_utils` is essential** — `generate_surrogate_key`, `union_relations`, `date_spine`, `pivot`
8. **Pin package versions** in `packages.yml` — use ranges or exact versions, never unpinned
9. **`run_query()`** executes SQL at compile time — always guard with `{% if execute %}`
10. **`codegen` + `dbt_project_evaluator`** accelerate development and enforce project best practices
