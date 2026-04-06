# Module 1: What Is dbt?

## Screen 1: The T in ELT — What dbt Actually Does

If you've worked in data engineering, you've heard of ETL — Extract, Transform, Load. For decades, transformations happened in a middle layer (an ETL tool like Informatica or Talend) before data landed in the warehouse. **dbt flips this on its head.** In the modern ELT paradigm, raw data is first extracted and loaded into a cloud data warehouse (Snowflake, BigQuery, Redshift, Databricks), and *then* dbt handles the **T** — the transformation — using pure SQL executed inside the warehouse itself.

dbt (data build tool) is not an extraction tool, not a loading tool, and not a scheduler. It does one thing exceptionally well: it lets analytics engineers write modular SQL `SELECT` statements that dbt compiles into DDL/DML and executes against your warehouse. You write a `SELECT`, dbt wraps it in a `CREATE TABLE AS SELECT` or `CREATE VIEW AS`, and runs it.

```
┌─────────────────────────────────────────────────────────┐
│                   Modern ELT Pipeline                   │
├─────────────┬─────────────────────┬─────────────────────┤
│  EXTRACT    │       LOAD          │     TRANSFORM       │
│             │                     │                     │
│  Fivetran   │   Fivetran/Airbyte  │       dbt           │
│  Airbyte    │   Stitch            │                     │
│  Stitch     │   Custom Scripts    │  SELECT statements  │
│  APIs       │                     │  executed in the    │
│             │                     │  warehouse          │
│             │                     │                     │
│  Sources ──►│──► Raw Tables ─────►│──► Clean Models     │
└─────────────┴─────────────────────┴─────────────────────┘
```

**What dbt gives you on top of SQL:**
- **Modularity**: Break complex transformations into small, reusable models
- **Dependency Management**: Automatically builds a DAG (Directed Acyclic Graph) of model execution order
- **Testing**: Write assertions about your data (uniqueness, not-null, referential integrity)
- **Documentation**: Auto-generate a searchable data catalog from your models
- **Version Control**: All transformations live in Git — pull requests, code review, CI/CD
- **Environment Management**: Dev vs. staging vs. production with the same codebase

Think of dbt as "software engineering best practices applied to SQL transformations." Before dbt, SQL transformations were often a tangled mess of stored procedures, views referencing views, and tribal knowledge. dbt brings structure, testability, and reproducibility to the analytics layer.

### 💡 Interview Insight

> **Common Question**: "What is dbt and what problem does it solve?"
> **Strong Answer**: "dbt is a transformation framework that lets analytics engineers write modular SQL models inside the warehouse, following the ELT pattern. It compiles SELECT statements into DDL/DML, builds a dependency DAG, runs tests, and generates documentation — essentially bringing software engineering practices like version control, testing, and modularity to the analytics layer."

---

## Screen 2: dbt Core vs dbt Cloud

dbt comes in two flavors, and interviewers love asking about the differences. Understanding when and why you'd choose one over the other signals real-world experience.

**dbt Core** is the open-source CLI tool. You install it locally (or in a container), write your models in a code editor, and run commands like `dbt run`, `dbt test`, and `dbt build` from the terminal. It's free, community-driven, and gives you full control. You're responsible for orchestration (Airflow, Dagster, Prefect), environment management, and CI/CD pipelines.

**dbt Cloud** is the managed SaaS platform built on top of dbt Core. It provides a web-based IDE, job scheduling, environment management, CI/CD with pull-request builds, a hosted documentation site, and features like the Semantic Layer and dbt Explorer.

```
┌────────────────────────────────────────────────────────────┐
│                  dbt Core vs dbt Cloud                     │
├─────────────────────┬──────────────────────────────────────┤
│ Feature             │  Core (OSS)  │  Cloud (Managed)     │
├─────────────────────┼──────────────┼──────────────────────┤
│ Cost                │  Free        │  Paid (free tier)    │
│ IDE                 │  Your editor │  Web IDE + VS Code   │
│ Scheduling          │  BYO (Air-   │  Built-in scheduler  │
│                     │  flow, etc.) │                      │
│ CI/CD               │  BYO (GH     │  Slim CI on PRs      │
│                     │  Actions)    │                      │
│ Documentation       │  Self-host   │  Hosted auto-docs    │
│ Semantic Layer      │  No          │  Yes (MetricFlow)    │
│ Environment Mgmt    │  Manual      │  Built-in            │
│ Auth & Permissions  │  N/A         │  RBAC, SSO           │
│ Execution           │  Local/CI    │  Cloud-hosted runs   │
│ Version             │  You manage  │  Managed upgrades    │
└─────────────────────┴──────────────┴──────────────────────┘
```

**When teams choose dbt Core**: They already have Airflow/Dagster for orchestration, want full control, have strong engineering culture, or have budget constraints. Common in companies with mature data platforms.

**When teams choose dbt Cloud**: They want rapid onboarding, don't want to manage orchestration infrastructure, need built-in CI/CD, or have analytics engineers who prefer a web IDE over terminal commands.

**Important**: The SQL you write and the project structure are **identical** between Core and Cloud. You can migrate between them without rewriting any models. The difference is the execution and management layer, not the transformation logic.

### 💡 Interview Insight

> **Common Question**: "Have you used dbt Core or Cloud? What would you recommend?"
> **Strong Answer**: "I've used both. The core SQL and project structure are identical — the difference is the operational layer. For teams with existing orchestration like Airflow, dbt Core gives full control. For teams wanting faster onboarding and built-in CI/CD, dbt Cloud reduces operational overhead. The transformation code itself is portable between them."

---

## Screen 3: Project Structure — Anatomy of a dbt Project

Every dbt project follows a standard directory layout. Understanding this structure is essential — interviewers expect you to describe where things live and why they're separated.

```
my_ecommerce_project/
├── dbt_project.yml          # Project configuration (name, version, paths)
├── profiles.yml             # Warehouse connection (often lives in ~/.dbt/)
├── packages.yml             # External dbt packages (dbt-utils, etc.)
│
├── models/                  # SQL models — the core of your project
│   ├── staging/             #   1:1 with source tables (renaming, casting)
│   │   ├── stg_orders.sql
│   │   ├── stg_customers.sql
│   │   └── _stg_sources.yml  # Source definitions + tests
│   ├── intermediate/        #   Business logic joins, aggregations
│   │   └── int_order_items_pivoted.sql
│   └── marts/               #   Business-ready fact & dimension tables
│       ├── fct_orders.sql
│       └── dim_customers.sql
│
├── tests/                   # Custom singular tests (SQL queries)
│   └── assert_positive_order_total.sql
│
├── macros/                  # Reusable Jinja functions
│   └── cents_to_dollars.sql
│
├── seeds/                   # Static CSV files loaded into warehouse
│   └── country_codes.csv
│
├── snapshots/               # SCD Type 2 tracking of source changes
│   └── snap_customers.sql
│
├── analyses/                # Ad-hoc SQL queries (compiled, not executed)
│   └── monthly_revenue.sql
│
└── target/                  # Compiled SQL output (gitignored)
    └── compiled/
```

**Key directories explained:**

- **`models/`**: Where your SQL `SELECT` statements live. Each `.sql` file is a model that becomes a table or view. This is where you spend 90% of your time.
- **`tests/`**: Custom data quality assertions. Each file is a SQL query that should return **zero rows** if the test passes.
- **`macros/`**: Reusable Jinja2 template functions. Think of them as SQL helper functions — DRY up repeated logic across models.
- **`seeds/`**: Small CSV files (country codes, status mappings) that dbt loads into your warehouse with `dbt seed`. Not for large datasets.
- **`snapshots/`**: Track how source data changes over time using SCD Type 2 patterns. Adds `dbt_valid_from` and `dbt_valid_to` columns.
- **`analyses/`**: SQL files that dbt compiles (resolving `ref()` and Jinja) but doesn't execute. Useful for ad-hoc queries that benefit from dbt's templating.
- **`target/`**: Auto-generated compiled SQL. Always gitignored. Useful for debugging — see the actual SQL dbt sends to the warehouse.

The YAML files (ending in `.yml`) scattered through `models/` define **sources**, **tests**, **documentation**, and **column descriptions**. By convention, they're prefixed with an underscore (`_sources.yml`, `_schema.yml`).

### 💡 Interview Insight

> **Common Question**: "Walk me through a dbt project structure."
> **Strong Answer**: Walk through each directory with purpose: "Models are organized into staging, intermediate, and marts layers. Staging models are 1:1 with sources and handle renaming and type casting. Intermediate models contain business logic. Marts are the final business-ready tables. Tests validate data quality, macros keep SQL DRY, seeds handle static reference data, and snapshots track historical changes."

---

## Screen 4: dbt_project.yml — The Project Configuration File

The `dbt_project.yml` is the heartbeat of every dbt project. It tells dbt your project's name, where to find models, how to materialize them by default, and how to apply configurations across entire directories.

```yaml
# dbt_project.yml
name: 'ecommerce_analytics'
version: '1.0.0'
config-version: 2

# Default profile to use for warehouse connection
profile: 'ecommerce'

# Where dbt looks for each type of file
model-paths: ["models"]
analysis-paths: ["analyses"]
test-paths: ["tests"]
seed-paths: ["seeds"]
macro-paths: ["macros"]
snapshot-paths: ["snapshots"]
target-path: "target"        # Compiled SQL output
clean-targets: ["target", "dbt_packages"]

# Model-level configurations (applied by directory)
models:
  ecommerce_analytics:       # Must match the 'name' field above
    staging:
      +materialized: view    # All staging models are views
      +schema: staging       # Deploy to the 'staging' schema
      +tags: ['staging', 'daily']
    intermediate:
      +materialized: ephemeral  # Intermediate = CTEs, not persisted
    marts:
      +materialized: table      # Marts are full tables
      +schema: marts
      +tags: ['marts']

# Seed configurations
seeds:
  ecommerce_analytics:
    +schema: seeds

# Snapshot configurations
snapshots:
  ecommerce_analytics:
    +strategy: timestamp
    +updated_at: updated_at

# Variable definitions (accessible via var())
vars:
  payment_methods: ['credit_card', 'coupon', 'bank_transfer']
  start_date: '2023-01-01'
```

**Key concepts in this file:**

1. **Cascading Configuration**: Settings applied at a directory level cascade to all models within. A model in `models/staging/` inherits `+materialized: view` without needing an explicit config block. Individual models can override with their own `{{ config() }}` block.

2. **The `+` prefix**: In `dbt_project.yml`, the `+` before a config key means "apply this to all resources in this directory." Without the `+`, dbt would interpret it as a subdirectory name.

3. **Profile reference**: The `profile` key links to `profiles.yml`, which contains the warehouse credentials. This separation means your project config (committed to Git) never contains secrets.

4. **Variables**: The `vars` section defines project-wide variables accessible in models via `{{ var('payment_methods') }}`. You can override them at runtime: `dbt run --vars '{"start_date": "2024-01-01"}'`.

5. **Version**: `config-version: 2` is the current configuration version format. This determines how dbt interprets the YAML structure.

### 💡 Interview Insight

> **Common Question**: "How do you configure materialization for different model layers?"
> **Strong Answer**: "In `dbt_project.yml`, I use directory-level configuration with the `+` prefix. Staging models default to views for simplicity, intermediate models are ephemeral (compiled as CTEs), and mart models are tables for query performance. Individual models can override these defaults with an inline `{{ config() }}` block — for example, a high-traffic staging model might override to `table` for performance."

---

## Screen 5: Profiles — Connecting to Your Warehouse

The `profiles.yml` file defines how dbt connects to your data warehouse. It lives in `~/.dbt/profiles.yml` by default (outside your project directory) because it contains credentials that should **never** be committed to Git.

```yaml
# ~/.dbt/profiles.yml
ecommerce:                    # Profile name (matches dbt_project.yml)
  target: dev                 # Default target environment
  outputs:

    dev:                      # Development environment
      type: snowflake
      account: xy12345.us-east-1
      user: "{{ env_var('DBT_USER') }}"
      password: "{{ env_var('DBT_PASSWORD') }}"
      role: transformer_dev
      database: ECOMMERCE_DEV
      warehouse: DEV_WH
      schema: dbt_jsmith      # Developer-specific schema
      threads: 4              # Parallel model execution

    prod:                     # Production environment
      type: snowflake
      account: xy12345.us-east-1
      user: "{{ env_var('DBT_PROD_USER') }}"
      password: "{{ env_var('DBT_PROD_PASSWORD') }}"
      role: transformer_prod
      database: ECOMMERCE_PROD
      warehouse: PROD_WH
      schema: analytics
      threads: 8

    # BigQuery example
    bigquery_dev:
      type: bigquery
      method: oauth            # or service-account
      project: my-gcp-project
      dataset: dbt_dev
      threads: 4
      location: US
```

**Key concepts:**

- **Target**: The active environment. `dbt run` uses the default target (`dev`). Switch with `dbt run --target prod`. This is how the **same code** runs against different databases/schemas.
- **Threads**: How many models dbt runs in parallel. Independent models (no dependency between them) execute concurrently. More threads = faster runs, but more warehouse load.
- **Schema**: In dev, each developer typically gets their own schema (`dbt_jsmith`, `dbt_mgarcia`) to avoid conflicts. In prod, models land in shared schemas like `analytics` or `marts`.
- **Environment Variables**: Use `{{ env_var('DBT_PASSWORD') }}` to inject secrets from the environment. Never hardcode credentials.

```
┌───────────────────────────────────────────────────────┐
│              Profile → Target Resolution              │
│                                                       │
│   dbt_project.yml          profiles.yml               │
│   ┌──────────────┐        ┌─────────────────────┐    │
│   │ profile:     │───────►│ ecommerce:          │    │
│   │  ecommerce   │        │   target: dev       │    │
│   └──────────────┘        │   outputs:          │    │
│                           │     dev: ──► DEV DB │    │
│   $ dbt run               │     prod: ─► PROD DB│    │
│     (uses 'dev')          └─────────────────────┘    │
│                                                       │
│   $ dbt run --target prod                             │
│     (overrides to 'prod')                             │
└───────────────────────────────────────────────────────┘
```

**Connection verification**: After setting up profiles, run `dbt debug` to validate the connection. This checks that dbt can reach the warehouse, authenticate, and access the target schema.

### 💡 Interview Insight

> **Common Question**: "How do you manage dev vs. prod environments in dbt?"
> **Strong Answer**: "Through `profiles.yml` targets. Each developer has a personal schema in a dev database. The same dbt code runs against prod by switching the target. In CI/CD, the target is set via environment variables. Credentials are injected with `env_var()` — never hardcoded. This gives us identical transformation logic across environments with isolated execution contexts."

---

## Screen 6: ref() — The Function That Builds the DAG

The `ref()` function is arguably the most important concept in dbt. Every time you reference another model, you use `ref()` instead of hardcoding a table name. This does two critical things: (1) resolves the correct table/view name in the current environment, and (2) tells dbt about the dependency, building the DAG.

```sql
-- models/staging/stg_orders.sql
SELECT
    id AS order_id,
    user_id AS customer_id,
    order_date,
    status,
    amount AS order_amount_cents
FROM {{ source('ecommerce', 'raw_orders') }}

-- models/staging/stg_customers.sql
SELECT
    id AS customer_id,
    first_name,
    last_name,
    email,
    created_at
FROM {{ source('ecommerce', 'raw_customers') }}

-- models/marts/fct_orders.sql
SELECT
    o.order_id,
    o.customer_id,
    c.first_name,
    c.last_name,
    c.email,
    o.order_date,
    o.status,
    o.order_amount_cents / 100.0 AS order_amount_dollars
FROM {{ ref('stg_orders') }} AS o
LEFT JOIN {{ ref('stg_customers') }} AS c
    ON o.customer_id = c.customer_id
```

**What `ref()` compiles to:** In dev, `{{ ref('stg_orders') }}` might compile to `ECOMMERCE_DEV.dbt_jsmith.stg_orders`. In prod, the same code compiles to `ECOMMERCE_PROD.analytics.stg_orders`. You never change your SQL — the profile and target handle environment resolution.

**The DAG (Directed Acyclic Graph)**: dbt parses every `ref()` call to build a dependency graph. This graph determines execution order — dbt guarantees that `stg_orders` runs before `fct_orders` because `fct_orders` references it.

```
┌─────────────────────────────────────────────────┐
│                  dbt DAG Example                │
│                                                 │
│  raw_orders ──► stg_orders ──┐                  │
│                              ├──► fct_orders    │
│  raw_customers──►stg_customers┘                 │
│                      │                          │
│                      └──────────► dim_customers │
└─────────────────────────────────────────────────┘
```

**Cross-project references**: In dbt Mesh (multi-project setups), you can reference models from other projects: `{{ ref('other_project', 'shared_model') }}`. This enables large organizations to split their dbt codebase into smaller, independently deployable projects.

**Why not just hardcode table names?** Three reasons: (1) environment portability breaks, (2) dbt can't determine execution order, and (3) you lose lineage tracking in the documentation site.

### 💡 Interview Insight

> **Common Question**: "What does ref() do and why is it important?"
> **Strong Answer**: "ref() serves two purposes: it resolves model names to the correct database.schema.table for the current environment, and it registers a dependency in the DAG so dbt knows the execution order. Without ref(), dbt can't build the lineage graph, can't parallelize independent models, and your code isn't portable across environments. It's the foundational mechanism that makes dbt's modularity work."

---

## Screen 7: source() — Referencing Raw Data Tables

While `ref()` references other dbt models, `source()` references raw tables that exist in your warehouse — tables loaded by your ingestion tools (Fivetran, Airbyte, Stitch, custom pipelines). Sources are defined in YAML files and give you a clean abstraction over raw table names.

```yaml
# models/staging/_stg_sources.yml
version: 2

sources:
  - name: ecommerce                    # Logical source name
    database: RAW_DATABASE             # Physical database
    schema: shopify                    # Physical schema
    description: "Raw data from Shopify via Fivetran"

    freshness:                         # Source freshness checks
      warn_after: {count: 12, period: hour}
      error_after: {count: 24, period: hour}

    loaded_at_field: _fivetran_synced   # Timestamp column for freshness

    tables:
      - name: raw_orders
        description: "One row per order placed on the platform"
        columns:
          - name: id
            description: "Primary key"
            tests:
              - unique
              - not_null
          - name: user_id
            description: "FK to raw_customers"
          - name: status
            description: "Order status"
            tests:
              - accepted_values:
                  values: ['placed', 'shipped', 'completed', 'returned']

      - name: raw_customers
        description: "One row per registered customer"
        columns:
          - name: id
            tests:
              - unique
              - not_null

      - name: raw_order_items
        description: "One row per item in an order"
```

**Using source() in models:**

```sql
-- models/staging/stg_orders.sql
SELECT
    id AS order_id,
    user_id AS customer_id,
    order_date,
    status,
    amount AS order_amount_cents,
    _fivetran_synced AS loaded_at
FROM {{ source('ecommerce', 'raw_orders') }}
```

`{{ source('ecommerce', 'raw_orders') }}` compiles to `RAW_DATABASE.shopify.raw_orders`. If the raw table moves to a different schema, you update the YAML once — not every model that references it.

**Source freshness**: Run `dbt source freshness` to check whether your raw tables are being updated on schedule. This is critical for production monitoring — if Fivetran stops syncing, you want to know before stale data reaches dashboards.

```
$ dbt source freshness

Running source freshness checks...
  ecommerce.raw_orders       OK  (last loaded: 2 hours ago)
  ecommerce.raw_customers    WARN (last loaded: 14 hours ago)
  ecommerce.raw_order_items  ERROR (last loaded: 26 hours ago)
```

**Source vs. ref()**: Sources point to tables **outside** dbt's control (loaded by other tools). `ref()` points to tables that dbt **creates and manages**. A staging model uses `source()` to read raw data and `ref()` is used by downstream models to read the staging output.

### 💡 Interview Insight

> **Common Question**: "What is the difference between source() and ref()?"
> **Strong Answer**: "source() references raw tables loaded by external tools — it provides an abstraction over physical table names and enables freshness monitoring. ref() references models within dbt itself. The pattern is: staging models use source() to read raw data, and all downstream models use ref() to read from staging. This creates a clean boundary between ingested data and transformed data."

---

## Screen 8: Putting It All Together — End-to-End Flow

Let's trace a complete data flow through our e-commerce dbt project, from raw data to a business-ready fact table. This is the kind of walkthrough interviewers love.

```
┌──────────────────────────────────────────────────────────────┐
│                  End-to-End dbt Pipeline                     │
│                                                              │
│  ┌──────────┐    ┌─────────────┐    ┌───────────────────┐   │
│  │ Fivetran  │    │  STAGING    │    │     MARTS         │   │
│  │ loads raw │───►│  stg_*      │───►│  fct_* / dim_*    │   │
│  │ tables    │    │  (views)    │    │  (tables)         │   │
│  └──────────┘    └─────────────┘    └───────────────────┘   │
│                                              │               │
│                                              ▼               │
│                                     ┌────────────────┐      │
│                                     │  Dashboards    │      │
│                                     │  (Looker, etc) │      │
│                                     └────────────────┘      │
└──────────────────────────────────────────────────────────────┘
```

**Step 1 — Define Sources** (YAML):
```yaml
# models/staging/_stg_sources.yml
sources:
  - name: ecommerce
    schema: shopify
    tables:
      - name: raw_orders
      - name: raw_customers
```

**Step 2 — Create Staging Models** (light transformations):
```sql
-- models/staging/stg_orders.sql
SELECT
    id AS order_id,
    user_id AS customer_id,
    order_date,
    status,
    amount AS order_amount_cents
FROM {{ source('ecommerce', 'raw_orders') }}
```

**Step 3 — Build Mart Models** (business logic):
```sql
-- models/marts/dim_customers.sql
{{ config(materialized='table') }}

WITH customer_orders AS (
    SELECT
        customer_id,
        MIN(order_date) AS first_order_date,
        MAX(order_date) AS most_recent_order_date,
        COUNT(*) AS number_of_orders,
        SUM(order_amount_cents) / 100.0 AS lifetime_value_dollars
    FROM {{ ref('stg_orders') }}
    GROUP BY customer_id
)

SELECT
    c.customer_id,
    c.first_name,
    c.last_name,
    c.email,
    c.created_at,
    COALESCE(co.first_order_date, NULL) AS first_order_date,
    COALESCE(co.most_recent_order_date, NULL) AS most_recent_order_date,
    COALESCE(co.number_of_orders, 0) AS number_of_orders,
    COALESCE(co.lifetime_value_dollars, 0) AS lifetime_value_dollars
FROM {{ ref('stg_customers') }} AS c
LEFT JOIN customer_orders AS co
    ON c.customer_id = co.customer_id
```

**Step 4 — Run dbt**:
```bash
# Compile and execute all models in dependency order
$ dbt run

Running with dbt=1.8.0
Found 4 models, 6 tests, 2 sources

Concurrency: 4 threads

1 of 4 START view model dbt_jsmith.stg_orders ........... [RUN]
2 of 4 START view model dbt_jsmith.stg_customers ........ [RUN]
1 of 4 OK view model dbt_jsmith.stg_orders .............. [OK in 1.2s]
2 of 4 OK view model dbt_jsmith.stg_customers ........... [OK in 1.1s]
3 of 4 START table model dbt_jsmith.fct_orders .......... [RUN]
4 of 4 START table model dbt_jsmith.dim_customers ....... [RUN]
3 of 4 OK table model dbt_jsmith.fct_orders ............. [OK in 3.4s]
4 of 4 OK table model dbt_jsmith.dim_customers .......... [OK in 2.8s]

Finished running 2 view models, 2 table models in 8.5s.
```

Notice how dbt runs the staging views first (in parallel, since they're independent), then runs the mart tables (which depend on staging). This is the DAG in action.

### 💡 Interview Insight

> **Common Question**: "Walk me through how you'd build a new dbt project from scratch."
> **Strong Answer**: Describe the full flow: "First, I'd define sources pointing to raw tables. Then, create staging models with 1:1 source mappings — renaming columns, casting types, and applying light transformations. Next, build intermediate models if complex joins or business logic are needed. Finally, create mart-layer fact and dimension tables that are business-ready. I'd add tests at each layer and document columns in YAML files. Then `dbt build` runs everything — models in DAG order, then tests."

---

## Screen 9: Quiz — Module 1 Review

Test your understanding of dbt fundamentals before moving on.

### Question 1
**What does dbt handle in the ELT pipeline?**

A) Extract and Load
B) Extract, Transform, and Load
C) Transform only (the T in ELT)
D) Load and Transform

**Answer: C** — dbt handles only the Transform step. It assumes data has already been extracted and loaded into the warehouse by tools like Fivetran or Airbyte. dbt compiles SQL SELECT statements into DDL/DML and executes them against the warehouse.

---

### Question 2
**What are the two main purposes of the `ref()` function?**

A) Connects to the warehouse and authenticates the user
B) Resolves table names for the current environment and registers dependencies in the DAG
C) Creates tables and drops old ones
D) Defines source freshness and data quality tests

**Answer: B** — `ref()` resolves the correct `database.schema.table` name based on the active target/profile, and it registers a dependency edge in the DAG so dbt knows the correct execution order.

---

### Question 3
**Where does `profiles.yml` typically live, and why?**

A) In the dbt project root, committed to Git
B) In `~/.dbt/profiles.yml`, outside the project, because it contains credentials
C) In the `models/` directory alongside SQL files
D) It's embedded inside `dbt_project.yml`

**Answer: B** — `profiles.yml` lives in `~/.dbt/` by default because it contains warehouse credentials. It should never be committed to version control. Credentials should use `env_var()` for additional security.

---

### Question 4
**What is the difference between `source()` and `ref()`?**

A) They are interchangeable
B) `source()` references raw tables loaded by external tools; `ref()` references dbt-managed models
C) `source()` is for production; `ref()` is for development
D) `ref()` is deprecated in favor of `source()`

**Answer: B** — `source()` provides an abstraction over raw tables managed outside of dbt (by ingestion tools). `ref()` references models that dbt creates. Staging models use `source()` to read raw data; everything downstream uses `ref()`.

---

### Question 5
**In `dbt_project.yml`, what does the `+` prefix before a configuration key mean?**

A) The setting is optional
B) The setting is applied to all resources in that directory (not treated as a subdirectory name)
C) The setting overrides all model-level configs
D) The setting is only for production environments

**Answer: B** — The `+` prefix tells dbt to treat the key as a configuration property applied to all resources in that directory path, rather than interpreting it as a subdirectory name.

---

### 🔑 Key Takeaways

1. **dbt is the T in ELT** — it transforms data already loaded in the warehouse using SQL SELECT statements compiled into DDL/DML
2. **dbt Core vs Cloud** — Same SQL, different operational layer. Core = open-source CLI; Cloud = managed platform with IDE, scheduling, and CI/CD
3. **Project structure** follows a clear convention: `models/`, `tests/`, `macros/`, `seeds/`, `snapshots/`, `analyses/`
4. **`dbt_project.yml`** configures your project and applies cascading settings to model directories
5. **`profiles.yml`** manages warehouse connections and environment targets — never committed to Git
6. **`ref()`** builds the DAG and resolves table names across environments — always use it to reference dbt models
7. **`source()`** abstracts raw table references and enables freshness monitoring
