# Module 4: Testing & Data Quality in dbt

## Screen 1: Why Testing Is Non-Negotiable in dbt

In traditional SQL pipelines, data quality problems are discovered when a VP opens a dashboard and says "these numbers look wrong." By then, the bad data has cascaded through 15 downstream models, corrupted 8 dashboards, and been exported to 3 external systems. **dbt testing exists to catch problems at the source, before they propagate.**

dbt tests are SQL queries that return **failing rows**. If a test returns zero rows, it passes. If it returns any rows, those rows represent data quality violations. This simple model — "show me the bad data" — makes tests easy to write, easy to debug, and easy to explain to non-technical stakeholders.

```
┌──────────────────────────────────────────────────────────────┐
│              Data Quality Without vs With dbt Tests          │
│                                                              │
│  WITHOUT TESTS:                                              │
│  raw_orders → stg_orders → fct_orders → dashboard            │
│       ↑                                      ↓               │
│  Bad data enters                    VP: "Numbers are wrong"  │
│  silently                           (discovered days later)  │
│                                                              │
│  WITH TESTS:                                                 │
│  raw_orders → stg_orders ──[TESTS]──► fct_orders → dashboard │
│                               ↓                              │
│                          ❌ FAIL: 3 null customer_ids found  │
│                          Pipeline halts. Alert fires.         │
│                          Fix applied BEFORE bad data spreads. │
└──────────────────────────────────────────────────────────────┘
```

**Types of tests in dbt:**

| Test Type | Where Defined | Purpose |
|-----------|---------------|---------|
| Generic tests | YAML schema files | Reusable: unique, not_null, etc. |
| Singular tests | `tests/` directory | Custom SQL for specific assertions |
| Package tests | `dbt-expectations` etc. | Community-built test library |
| Source freshness | Source YAML | Is data arriving on schedule? |
| Contract tests | Model YAML | Enforce column types/constraints |

**Running tests:**

```bash
# Run all tests
$ dbt test

# Run tests for a specific model
$ dbt test --select fct_orders

# Run tests for all models in a directory
$ dbt test --select path:models/marts

# Run models AND their tests together (recommended)
$ dbt build

# Run tests with a specific tag
$ dbt test --select tag:critical
```

The `dbt build` command is the production standard — it runs models **and** their tests in DAG order. If `stg_orders` tests fail, dbt won't build `fct_orders` (which depends on it). This prevents bad data from cascading downstream.

### 💡 Interview Insight

> **Common Question**: "How do you ensure data quality in your dbt project?"
> **Strong Answer**: "I implement tests at every layer. Staging models get generic tests — unique, not_null on primary keys, accepted_values on status columns, and relationships tests for foreign keys. Mart models get custom singular tests for business logic assertions (e.g., order totals must be positive, no future-dated orders). I use `dbt build` in production so tests run inline with model execution — if tests fail, downstream models don't execute. I also monitor source freshness to catch ingestion issues early."

---

## Screen 2: Generic Tests — The Four Built-In Tests

dbt ships with four generic tests that cover the vast majority of data quality needs. They're defined in YAML schema files alongside your model definitions.

```yaml
# models/staging/_stg_schema.yml
version: 2

models:
  - name: stg_orders
    description: "Staged orders — one row per order"
    columns:
      - name: order_id
        description: "Primary key"
        tests:
          - unique
          - not_null

      - name: customer_id
        description: "FK to stg_customers"
        tests:
          - not_null
          - relationships:
              to: ref('stg_customers')
              field: customer_id

      - name: status
        description: "Current order status"
        tests:
          - accepted_values:
              values: ['placed', 'shipped', 'completed', 'returned', 'cancelled']

      - name: order_amount_cents
        description: "Order total in cents"
        tests:
          - not_null

  - name: stg_customers
    description: "Staged customers — one row per customer"
    columns:
      - name: customer_id
        tests:
          - unique
          - not_null
      - name: email
        tests:
          - unique
          - not_null
```

**What each test does under the hood:**

**`unique`** — Returns rows with duplicate values:
```sql
-- Compiled test SQL (simplified)
SELECT order_id
FROM stg_orders
GROUP BY order_id
HAVING COUNT(*) > 1
```

**`not_null`** — Returns rows where the column is NULL:
```sql
SELECT order_id
FROM stg_orders
WHERE order_id IS NULL
```

**`accepted_values`** — Returns rows with unexpected values:
```sql
SELECT status
FROM stg_orders
WHERE status NOT IN ('placed', 'shipped', 'completed', 'returned', 'cancelled')
```

**`relationships`** — Returns rows with orphaned foreign keys:
```sql
SELECT o.customer_id
FROM stg_orders o
LEFT JOIN stg_customers c
    ON o.customer_id = c.customer_id
WHERE c.customer_id IS NULL
    AND o.customer_id IS NOT NULL
```

**Best practices for generic tests:**

```
┌────────────────────────────────────────────────────────┐
│          Generic Test Placement Guide                  │
│                                                        │
│  PRIMARY KEYS:     unique + not_null (always!)         │
│  FOREIGN KEYS:     relationships + not_null            │
│  STATUS/ENUM COLS: accepted_values                     │
│  AMOUNT COLS:      not_null                            │
│  DATE COLS:        not_null (if required)              │
│  EMAIL/ID COLS:    unique (if business rule)           │
│                                                        │
│  Rule: Every model should have at least:               │
│  • unique + not_null on its primary key                │
│  • relationships tests for all foreign keys            │
└────────────────────────────────────────────────────────┘
```

### 💡 Interview Insight

> **Common Question**: "What are the four built-in dbt tests?"
> **Strong Answer**: "unique, not_null, accepted_values, and relationships. I apply unique + not_null to every primary key — that's non-negotiable. I use accepted_values for status/enum columns to catch unexpected values from source systems. And relationships tests verify referential integrity between models — for example, every customer_id in fct_orders should exist in dim_customers. These four tests catch 80% of data quality issues."

---

## Screen 3: Singular (Custom) Tests — SQL-Based Assertions

When the four generic tests aren't enough, you write **singular tests** — custom SQL queries saved in the `tests/` directory. Each file contains a SELECT that returns **failing rows**. If the query returns zero rows, the test passes.

```sql
-- tests/assert_positive_order_amounts.sql
-- PASS: zero rows returned (all amounts are positive)
-- FAIL: returns rows where order_amount_dollars <= 0

SELECT
    order_id,
    customer_id,
    order_amount_dollars,
    order_date
FROM {{ ref('fct_orders') }}
WHERE order_amount_dollars <= 0
```

```sql
-- tests/assert_no_future_orders.sql
-- Orders should never have a future date

SELECT
    order_id,
    order_date,
    CURRENT_DATE() AS today
FROM {{ ref('fct_orders') }}
WHERE order_date > CURRENT_DATE()
```

```sql
-- tests/assert_customer_order_count_matches.sql
-- Cross-model consistency: dim_customers.total_orders should match
-- the actual count in fct_orders

WITH actual_counts AS (
    SELECT
        customer_id,
        COUNT(*) AS actual_order_count
    FROM {{ ref('fct_orders') }}
    GROUP BY customer_id
),

dimension_counts AS (
    SELECT
        customer_id,
        total_orders AS reported_order_count
    FROM {{ ref('dim_customers') }}
    WHERE total_orders > 0
)

SELECT
    d.customer_id,
    d.reported_order_count,
    COALESCE(a.actual_order_count, 0) AS actual_order_count,
    d.reported_order_count - COALESCE(a.actual_order_count, 0) AS difference
FROM dimension_counts d
LEFT JOIN actual_counts a
    ON d.customer_id = a.customer_id
WHERE d.reported_order_count != COALESCE(a.actual_order_count, 0)
```

```sql
-- tests/assert_revenue_not_zero.sql
-- Daily revenue should never be zero (business is always active)

SELECT
    DATE(order_date) AS revenue_date,
    SUM(order_amount_dollars) AS daily_revenue
FROM {{ ref('fct_orders') }}
WHERE order_date >= DATEADD(day, -30, CURRENT_DATE())
GROUP BY DATE(order_date)
HAVING SUM(order_amount_dollars) = 0
```

**When to use singular vs. generic tests:**

| Use Generic Tests When... | Use Singular Tests When... |
|---------------------------|---------------------------|
| Testing a single column | Testing relationships between models |
| Standard assertion (null, unique) | Complex business logic |
| Reusable across models | Specific to one model/scenario |
| Simple threshold | Multi-column conditions |

**Singular test configuration** — you can add config blocks to singular tests:

```sql
-- tests/assert_positive_order_amounts.sql
{{ config(
    severity='error',
    tags=['critical', 'finance'],
    enabled=true,
    store_failures=true,    -- Save failing rows to a table
    schema='test_results'
) }}

SELECT
    order_id,
    order_amount_dollars
FROM {{ ref('fct_orders') }}
WHERE order_amount_dollars <= 0
```

The `store_failures=true` config saves failing rows to a table (e.g., `test_results.assert_positive_order_amounts`), making it easy to investigate failures without re-running the test.

### 💡 Interview Insight

> **Common Question**: "Give me an example of a custom dbt test you've written."
> **Strong Answer**: "I wrote a cross-model consistency test that compares `dim_customers.total_orders` against the actual count in `fct_orders`. This catches bugs where the dimension aggregation drifts from reality — for example, if an incremental run missed late-arriving orders. I also use `store_failures=true` to save failing rows to a table, so I can investigate without re-running the test."

---

## Screen 4: dbt-expectations — Great Expectations-Style Tests

The `dbt-expectations` package brings Great Expectations-style data quality checks into dbt. It provides 50+ additional generic tests that go far beyond the four built-in ones.

**Installation:**

```yaml
# packages.yml
packages:
  - package: calogica/dbt_expectations
    version: [">=0.10.0", "<0.11.0"]
```

```bash
$ dbt deps  # Install the package
```

**Commonly used tests:**

```yaml
# models/marts/_marts_schema.yml
version: 2

models:
  - name: fct_orders
    description: "Fact table — one row per order"
    tests:
      # Table-level: row count should be within expected range
      - dbt_expectations.expect_table_row_count_to_be_between:
          min_value: 1000
          max_value: 10000000

      # Table-level: row count should not drop more than 10% from yesterday
      - dbt_expectations.expect_table_row_count_to_equal_other_table:
          compare_model: ref('stg_orders')

    columns:
      - name: order_amount_dollars
        tests:
          # Column values should be within a range
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: 0
              max_value: 100000
              severity: error

          # Mean should be within expected bounds
          - dbt_expectations.expect_column_mean_to_be_between:
              min_value: 50
              max_value: 500
              severity: warn

      - name: email
        tests:
          # Email format validation
          - dbt_expectations.expect_column_values_to_match_regex:
              regex: "^[a-zA-Z0-9_.+-]+@[a-zA-Z0-9-]+\\.[a-zA-Z0-9-.]+$"
              severity: warn

      - name: order_date
        tests:
          # No dates before the company was founded
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: "'2020-01-01'"
              max_value: "CURRENT_DATE()"

          # Should have recent data (no gaps)
          - dbt_expectations.expect_row_values_to_have_recent_data:
              datepart: day
              interval: 2

      - name: status
        tests:
          # Distribution check — no single status should be > 50%
          - dbt_expectations.expect_column_proportion_of_unique_values_to_be_between:
              min_value: 0.0001
```

**Most useful dbt-expectations tests:**

```
┌──────────────────────────────────────────────────────────────────┐
│              Top dbt-expectations Tests                          │
├────────────────────────────────────────┬─────────────────────────┤
│ Test                                   │ What It Checks         │
├────────────────────────────────────────┼─────────────────────────┤
│ expect_column_values_to_be_between     │ Value range bounds     │
│ expect_column_values_to_not_be_null    │ Nullability (alt)      │
│ expect_column_values_to_match_regex    │ Format validation      │
│ expect_table_row_count_to_be_between   │ Table size bounds      │
│ expect_column_mean_to_be_between       │ Statistical bounds     │
│ expect_column_distinct_count_to_equal  │ Cardinality check      │
│ expect_row_values_to_have_recent_data  │ Data freshness         │
│ expect_column_values_to_be_increasing  │ Monotonic sequence     │
│ expect_compound_columns_to_be_unique   │ Composite uniqueness   │
│ expect_table_columns_to_match_ordered  │ Schema contract        │
│  _list                                 │                        │
└────────────────────────────────────────┴─────────────────────────┘
```

**Why use dbt-expectations over custom tests?** Consistency and reusability. Instead of writing custom SQL for "values should be between 0 and 100,000," you use a parameterized generic test that's been battle-tested across thousands of projects.

### 💡 Interview Insight

> **Common Question**: "What dbt packages do you use for testing?"
> **Strong Answer**: "dbt-expectations is my go-to. It provides statistical and distribution tests that go beyond the built-in four — things like value range checks, regex validation, row count comparisons between tables, and freshness assertions. I use `expect_column_values_to_be_between` for numeric bounds, `expect_table_row_count_to_be_between` to catch data volume anomalies, and `expect_row_values_to_have_recent_data` as a freshness check at the model level."

---

## Screen 5: Source Freshness — Is My Data Arriving?

Source freshness monitoring answers a critical question: **"Is the ingestion pipeline still working?"** If Fivetran stops syncing and your raw tables go stale, dbt will happily run transformations on old data — producing dashboards that look fine but show yesterday's (or last week's) numbers.

```yaml
# models/staging/_stg_sources.yml
version: 2

sources:
  - name: ecommerce
    database: RAW_DATABASE
    schema: shopify
    description: "Raw e-commerce data from Shopify via Fivetran"

    # Default freshness for all tables in this source
    freshness:
      warn_after: {count: 12, period: hour}
      error_after: {count: 24, period: hour}
    loaded_at_field: _fivetran_synced

    tables:
      - name: raw_orders
        description: "Order data — synced every hour"
        # Override freshness for this specific table
        freshness:
          warn_after: {count: 6, period: hour}
          error_after: {count: 12, period: hour}
        loaded_at_field: _fivetran_synced

      - name: raw_customers
        description: "Customer data — synced daily"
        freshness:
          warn_after: {count: 24, period: hour}
          error_after: {count: 48, period: hour}

      - name: raw_page_views
        description: "Clickstream — synced every 15 minutes"
        freshness:
          warn_after: {count: 1, period: hour}
          error_after: {count: 3, period: hour}

      - name: raw_country_codes
        description: "Static reference table — no freshness check"
        freshness: null   # Disable freshness for static tables
```

**Running freshness checks:**

```bash
$ dbt source freshness

Running source freshness checks...

20:15:01  Done.
20:15:01  Freshness results:
┌─────────────────────────────┬────────┬──────────────────┐
│ Source                      │ Status │ Last Loaded      │
├─────────────────────────────┼────────┼──────────────────┤
│ ecommerce.raw_orders        │ PASS   │ 2 hours ago      │
│ ecommerce.raw_customers     │ PASS   │ 8 hours ago      │
│ ecommerce.raw_page_views    │ WARN   │ 1.5 hours ago    │
│ ecommerce.raw_country_codes │ SKIP   │ (no freshness)   │
└─────────────────────────────┴────────┴──────────────────┘
```

**How dbt checks freshness:** It runs a simple query against the `loaded_at_field`:

```sql
-- What dbt executes under the hood
SELECT MAX(_fivetran_synced) AS max_loaded_at
FROM RAW_DATABASE.shopify.raw_orders
```

It then compares `max_loaded_at` to `CURRENT_TIMESTAMP()` and checks against your warn/error thresholds.

**Production pattern — freshness checks before model runs:**

```bash
# In your orchestrator (Airflow, etc.), run freshness first
$ dbt source freshness

# If freshness passes, run models
$ dbt build
```

If freshness checks fail with `error`, you can configure your orchestrator to skip the dbt run and send an alert instead. This prevents building models on stale data.

**Freshness in dbt build:** When you use `dbt build`, source freshness checks run automatically before the models that depend on those sources. A freshness `error` will prevent downstream models from executing.

### 💡 Interview Insight

> **Common Question**: "How do you monitor whether source data is arriving on time?"
> **Strong Answer**: "I configure source freshness in the YAML with `loaded_at_field`, `warn_after`, and `error_after` thresholds. I run `dbt source freshness` as a pre-step in our Airflow DAG — if any critical source is stale, the pipeline halts and alerts fire before we build models on bad data. Different sources get different thresholds: orders sync hourly (warn at 6h, error at 12h), reference data syncs daily (warn at 24h, error at 48h)."

---

## Screen 6: dbt test vs dbt build — Execution Strategies

Understanding the difference between `dbt test` and `dbt build` — and when to use each — is essential for production dbt.

**`dbt test`**: Runs tests only. Does not build models. Tests the data **as it currently exists** in the warehouse.

**`dbt build`**: Runs models, tests, snapshots, and seeds in DAG order. Tests run **immediately after** each model they're attached to. If a model's tests fail, downstream models are skipped.

```
┌──────────────────────────────────────────────────────────────┐
│              dbt test vs dbt build                           │
│                                                              │
│  $ dbt test                                                  │
│  ┌──────────────────────────────────────────┐                │
│  │ Run all tests against existing tables    │                │
│  │ (models are NOT rebuilt)                 │                │
│  │                                          │                │
│  │ test stg_orders_unique ............. OK  │                │
│  │ test stg_orders_not_null ........... OK  │                │
│  │ test fct_orders_unique ............. OK  │                │
│  │ test assert_positive_amounts ....... FAIL│                │
│  └──────────────────────────────────────────┘                │
│                                                              │
│  $ dbt build                                                 │
│  ┌──────────────────────────────────────────┐                │
│  │ Build models AND run tests in DAG order  │                │
│  │                                          │                │
│  │ model stg_orders .................. OK   │                │
│  │ test  stg_orders_unique ........... OK   │                │
│  │ test  stg_orders_not_null ......... OK   │                │
│  │ model stg_customers ............... OK   │                │
│  │ test  stg_customers_unique ........ OK   │                │
│  │ model fct_orders .................. OK   │                │
│  │ test  fct_orders_unique ........... OK   │                │
│  │ test  assert_positive_amounts ..... FAIL │                │
│  │ model rpt_revenue ............. SKIPPED  │ ← blocked!    │
│  └──────────────────────────────────────────┘                │
└──────────────────────────────────────────────────────────────┘
```

**Why `dbt build` is the production standard:**

The key insight is **test gating**. With `dbt build`, if `stg_orders` tests fail, `fct_orders` never gets built. Bad data is stopped at the staging layer, never reaching marts or dashboards. With separate `dbt run` + `dbt test`, all models build first — bad data reaches downstream tables — and only then do tests tell you something went wrong.

```
┌───────────────────────────────────────────────────────┐
│           Anti-Pattern vs Best Practice               │
│                                                       │
│  ❌ Anti-Pattern:                                     │
│     $ dbt run   (all models build, bad data spreads)  │
│     $ dbt test  (tests fail AFTER the damage is done) │
│                                                       │
│  ✅ Best Practice:                                    │
│     $ dbt build (tests gate each model in DAG order)  │
└───────────────────────────────────────────────────────┘
```

**Selective execution:**

```bash
# Build a specific model and run its tests
$ dbt build --select fct_orders

# Build everything downstream of stg_orders (including tests)
$ dbt build --select stg_orders+

# Build only models tagged 'critical' and their tests
$ dbt build --select tag:critical

# Build only models that have changed (CI/CD — slim CI)
$ dbt build --select state:modified+
```

### 💡 Interview Insight

> **Common Question**: "Do you use `dbt run` + `dbt test` or `dbt build`?"
> **Strong Answer**: "Always `dbt build` in production. It runs models and tests in DAG order, so tests gate downstream execution. If `stg_orders` tests fail, `fct_orders` is skipped — bad data never reaches dashboards. Running `dbt run` then `dbt test` separately means all models build before any tests run, allowing bad data to propagate. I only use `dbt test` alone for ad-hoc validation during development."

---

## Screen 7: Test Severity — warn vs error

Not all test failures are created equal. A missing email address is a data quality issue, but it shouldn't stop your entire pipeline. A duplicate primary key, however, means your model is fundamentally broken. dbt's **severity** config lets you distinguish between warnings (log and continue) and errors (halt the pipeline).

```yaml
# models/marts/_marts_schema.yml
version: 2

models:
  - name: fct_orders
    columns:
      - name: order_id
        tests:
          # PRIMARY KEY: Must be unique and not null — ALWAYS error
          - unique:
              severity: error
          - not_null:
              severity: error

      - name: customer_id
        tests:
          # FK integrity — error if broken
          - relationships:
              to: ref('dim_customers')
              field: customer_id
              severity: error

      - name: status
        tests:
          # Unexpected status — warn, don't halt
          - accepted_values:
              values: ['placed', 'shipped', 'completed', 'returned', 'cancelled']
              severity: warn   # New status? Log it, don't break the pipeline

      - name: order_amount_dollars
        tests:
          # Zero-dollar orders are suspicious but not always wrong
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: 0.01
              max_value: 100000
              severity: warn

      - name: customer_email
        tests:
          # Missing emails happen — log, don't halt
          - not_null:
              severity: warn
              warn_if: ">10"     # Warn if more than 10 nulls
              error_if: ">1000"  # Error if more than 1000 nulls
```

**The `warn_if` / `error_if` thresholds** are incredibly useful. They let you set numeric thresholds instead of binary pass/fail:

```yaml
- name: customer_id
  tests:
    - not_null:
        severity: warn
        warn_if: ">0"      # Any null is a warning
        error_if: ">100"   # More than 100 nulls is an error
```

This means: "A few null customer_ids is acceptable (data quality issue to investigate), but more than 100 means something is seriously wrong — halt the pipeline."

**Severity decision framework:**

```
┌────────────────────────────────────────────────────────────┐
│              Severity Decision Guide                       │
│                                                            │
│  ALWAYS ERROR:                                             │
│  • Primary key uniqueness (data integrity)                 │
│  • Primary key not-null (data integrity)                   │
│  • Critical FK relationships (join correctness)            │
│  • Business-critical amounts (revenue calculations)        │
│                                                            │
│  USUALLY WARN:                                             │
│  • Non-critical column nulls (email, phone)                │
│  • Accepted values (new statuses appear over time)         │
│  • Statistical bounds (mean, distribution)                 │
│  • Format validation (regex for emails, phones)            │
│                                                            │
│  USE THRESHOLDS (warn_if / error_if):                      │
│  • When some violations are expected but many signal a bug │
│  • Example: <10 null emails = warn, >1000 = error          │
└────────────────────────────────────────────────────────────┘
```

**In `dbt build`**: `error` severity stops downstream models. `warn` severity logs the warning but allows downstream models to proceed. This is why getting severity right is critical — too many errors and your pipeline never completes; too many warns and you miss real problems.

### 💡 Interview Insight

> **Common Question**: "How do you decide between warn and error severity?"
> **Strong Answer**: "Error for anything that breaks data integrity — duplicate primary keys, null PKs, broken foreign key relationships. These mean the model is fundamentally wrong. Warn for data quality issues that are concerning but don't break correctness — missing optional fields, unexpected enum values, statistical outliers. I also use `warn_if`/`error_if` thresholds: a few null emails is a warn, but 1000+ nulls signals a source system failure and should error."

---

## Screen 8: Contract Enforcement — Model Contracts and Constraints

Starting with dbt 1.5+, **model contracts** let you enforce column names, types, and constraints as a formal agreement between model producers and consumers. Think of it as a schema contract — if the model output doesn't match the contract, dbt refuses to build it.

```yaml
# models/marts/_marts_schema.yml
version: 2

models:
  - name: fct_orders
    description: "Fact table for orders — one row per order"
    config:
      materialized: table
      contract:
        enforced: true    # ← Enable contract enforcement

    columns:
      - name: order_id
        data_type: integer
        description: "Primary key — unique order identifier"
        constraints:
          - type: not_null
          - type: primary_key    # Enforced at warehouse level
        tests:
          - unique
          - not_null

      - name: customer_id
        data_type: integer
        description: "FK to dim_customers"
        constraints:
          - type: not_null
          - type: foreign_key
            expression: "dim_customers (customer_id)"
        tests:
          - relationships:
              to: ref('dim_customers')
              field: customer_id

      - name: order_date
        data_type: date
        constraints:
          - type: not_null

      - name: status
        data_type: varchar(20)
        constraints:
          - type: not_null
          - type: check
            expression: "status IN ('placed','shipped','completed','returned','cancelled')"

      - name: order_amount_dollars
        data_type: decimal(12,2)
        constraints:
          - type: not_null
          - type: check
            expression: "order_amount_dollars >= 0"

      - name: customer_email
        data_type: varchar(255)
        # No not_null constraint — email is optional

      - name: dbt_loaded_at
        data_type: timestamp
        constraints:
          - type: not_null
```

**What contract enforcement does:**

1. **Column names must match**: If your SELECT returns a column not in the contract, or misses a column that's defined, dbt errors before sending any SQL to the warehouse.
2. **Data types must match**: If you declare `data_type: integer` but your SELECT returns a string, dbt catches it.
3. **Constraints are applied**: dbt generates DDL with the constraints (NOT NULL, PRIMARY KEY, CHECK) depending on warehouse support.

```
┌──────────────────────────────────────────────────────────────┐
│          Tests vs Contracts vs Constraints                    │
├──────────────┬──────────────┬────────────────────────────────┤
│ Feature      │ When Checked │ What Happens on Failure        │
├──────────────┼──────────────┼────────────────────────────────┤
│ dbt tests    │ After build  │ Rows identified as failing     │
│ (YAML tests) │              │ (post-hoc validation)          │
│              │              │                                │
│ Contract     │ Before build │ Build refuses to start         │
│ (schema)     │ (compile)    │ (compile-time check)           │
│              │              │                                │
│ Constraints  │ During build │ Warehouse rejects bad rows     │
│ (DDL)        │ (execution)  │ (runtime enforcement)          │
└──────────────┴──────────────┴────────────────────────────────┘
```

**Contracts are especially important in dbt Mesh** (multi-project setups). When Project A exposes `fct_orders` for Project B to consume, the contract is the formal API agreement. Project A can't accidentally rename a column and break Project B.

**Contract enforcement vs. tests — they're complementary:**

- **Contracts** catch structural issues at compile time (wrong column names, wrong types)
- **Tests** catch data quality issues at runtime (null values in supposedly non-null columns, broken referential integrity in the actual data)
- Use **both** for comprehensive coverage

### 💡 Interview Insight

> **Common Question**: "What are model contracts in dbt and when would you use them?"
> **Strong Answer**: "Model contracts enforce column names, data types, and constraints as a compile-time check. If the model's SELECT output doesn't match the declared schema, dbt errors before running any SQL. I use them for mart-layer models that serve as APIs for other teams — especially in dbt Mesh where multiple projects depend on each other. Contracts prevent accidental breaking changes. They complement tests: contracts catch structural issues at compile time, tests catch data quality issues at runtime."

---

## Screen 9: Quiz — Module 4 Review

### Question 1
**A dbt test returns 5 rows. Does it pass or fail?**

A) Pass — 5 rows is fine
B) Fail — any test returning rows indicates violations
C) Depends on the severity setting
D) Depends on the number of rows in the table

**Answer: B** — dbt tests pass when they return **zero rows**. Any returned rows represent data quality violations. However, with `warn_if`/`error_if` thresholds, 5 rows might trigger a warn instead of an error, but the test still technically "fails" — it just doesn't halt the pipeline.

---

### Question 2
**What is the key difference between `dbt test` and `dbt build`?**

A) `dbt test` is faster
B) `dbt build` runs models AND tests in DAG order, allowing tests to gate downstream execution
C) `dbt test` runs more tests
D) They produce the same results

**Answer: B** — `dbt build` runs models, tests, snapshots, and seeds in DAG order. If a model's tests fail, downstream models are skipped. `dbt test` alone only runs tests against existing data without building models, and doesn't provide gating.

---

### Question 3
**Your `stg_orders` model has an `accepted_values` test on the `status` column. A new status value 'backordered' appears in the source data. What severity would you recommend?**

A) `error` — halt the pipeline immediately
B) `warn` — log it and investigate, but don't break the pipeline
C) Disable the test
D) Remove the accepted_values test entirely

**Answer: B** — New status values are expected to appear over time in source systems. Using `warn` severity logs the unexpected value so you can investigate and update the accepted values list, without halting the entire pipeline. Halting on a new enum value is usually too aggressive for production.

---

### Question 4
**What does `store_failures=true` do in a dbt test config?**

A) Stores the test SQL in a log file
B) Saves the failing rows to a table in the warehouse for investigation
C) Prevents the test from failing
D) Emails the failures to the team

**Answer: B** — `store_failures=true` creates a table containing the rows that caused the test to fail. This makes it easy to investigate failures without re-running the test — you can query the failures table directly.

---

### Question 5
**You configure source freshness with `warn_after: {count: 12, period: hour}` and `error_after: {count: 24, period: hour}`. The source was last loaded 18 hours ago. What's the result?**

A) PASS
B) WARN
C) ERROR
D) SKIP

**Answer: B** — 18 hours exceeds the warn threshold (12 hours) but is within the error threshold (24 hours). The result is WARN — the freshness check passes with a warning, alerting the team that the source is getting stale.

---

### 🔑 Key Takeaways

1. **dbt tests are SQL queries that return failing rows** — zero rows = pass, any rows = fail
2. **Four generic tests** cover most needs: `unique`, `not_null`, `accepted_values`, `relationships`
3. **Singular tests** are custom SQL files in `tests/` for complex business logic assertions
4. **dbt-expectations** provides 50+ additional tests — value ranges, regex, statistical bounds, row counts
5. **Source freshness** monitors whether raw data is arriving on schedule — `loaded_at_field` + `warn_after`/`error_after`
6. **`dbt build` > `dbt run` + `dbt test`** — build runs models and tests in DAG order, providing test gating
7. **Severity**: `error` for data integrity (PK uniqueness), `warn` for data quality (optional fields, new enum values)
8. **`warn_if`/`error_if` thresholds** provide nuanced control — a few violations = warn, many = error
9. **Model contracts** enforce column names, types, and constraints at compile time — essential for multi-team dbt projects
10. **Tests + contracts are complementary** — contracts catch structural issues, tests catch data quality issues
