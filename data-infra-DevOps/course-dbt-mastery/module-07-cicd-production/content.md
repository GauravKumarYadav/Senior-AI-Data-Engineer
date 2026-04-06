# Module 7: CI/CD & dbt in Production

## Screen 1: From `dbt run` to Production — The Full Picture

Running `dbt run` on your laptop is development. Running dbt reliably, on schedule, with tests, alerts, documentation, and rollback capability — that's production. This module covers everything between "it works on my machine" and "it runs at 4am and wakes someone up if it breaks."

**The production dbt stack:**

```
┌──────────────────────────────────────────────────────────────────┐
│           Production dbt Architecture                            │
│                                                                  │
│  ┌───────────┐   ┌──────────┐   ┌─────────────┐   ┌──────────┐ │
│  │ Git Repo  │──▶│ CI/CD    │──▶│ Orchestrator│──▶│ Warehouse│ │
│  │ (GitHub)  │   │ (Actions)│   │ (Airflow)   │   │(Snowflake│ │
│  └───────────┘   └──────────┘   └─────────────┘   └──────────┘ │
│       │               │               │                │        │
│  PR opened       Slim CI:         Scheduled:       Executes    │
│  ↓               dbt build        dbt build        SQL         │
│  Pre-commit      --select         (daily 4am)                  │
│  hooks:          state:modified+  ↓                            │
│  SQLFluff lint   ↓                Alert on                     │
│  YAML validate   Artifacts:       failure                      │
│                  manifest.json    ↓                             │
│                  run_results.json dbt docs                     │
│                                   serve                        │
└──────────────────────────────────────────────────────────────────┘
```

**The evolution of dbt execution commands:**

| Command | What It Does | Use When |
|---------|-------------|----------|
| `dbt run` | Build models only | Development (legacy) |
| `dbt test` | Run tests only | Ad-hoc validation |
| `dbt snapshot` | Run snapshots only | Standalone snapshot job |
| `dbt seed` | Load CSV seed files | Reference data loading |
| `dbt build` | Models + tests + snapshots + seeds in DAG order | **Production standard** |

**`dbt build` is the production command** because it:
1. Runs everything in DAG order (dependencies respected)
2. Tests gate downstream execution (bad data doesn't propagate)
3. Snapshots run at the right time relative to models
4. Seeds are loaded before models that depend on them

```bash
# Production run — the one command
$ dbt build

# Production run with selector
$ dbt build --select tag:daily

# CI run — only changed models and their downstream
$ dbt build --select state:modified+

# Full refresh (weekly maintenance)
$ dbt build --full-refresh
```

### 💡 Interview Insight

> **Common Question**: "What command do you use in production and why?"
> **Strong Answer**: "`dbt build`. It executes models, tests, snapshots, and seeds in DAG order with test gating. If `stg_orders` tests fail, `fct_orders` is skipped — bad data never reaches marts. This is superior to running `dbt run` + `dbt test` separately, which builds all models first (allowing bad data to propagate) and only then discovers test failures. I use `dbt build --select state:modified+` in CI for slim builds."

---

## Screen 2: Slim CI — `state:modified+` and Artifacts

In a project with 500+ models, you don't want every PR to rebuild everything. **Slim CI** uses dbt artifacts to identify which models changed and only builds those (plus their downstream dependents).

**How Slim CI works:**

```
┌──────────────────────────────────────────────────────────────────┐
│           Slim CI Flow                                           │
│                                                                  │
│  1. Production run generates manifest.json (saved as artifact)   │
│  2. Developer opens PR with changes to stg_orders.sql            │
│  3. CI pipeline:                                                 │
│     a. dbt compile (generates new manifest.json)                 │
│     b. dbt build --select state:modified+                        │
│        --state ./prod-artifacts/                                 │
│     c. Compares new manifest vs prod manifest                    │
│     d. Finds: stg_orders was modified                            │
│     e. Builds: stg_orders + all downstream (fct_orders, etc.)    │
│     f. Runs tests on those models                                │
│                                                                  │
│  Result: Only 5 models built instead of 500+                     │
│  CI time: 2 minutes instead of 45 minutes                        │
└──────────────────────────────────────────────────────────────────┘
```

**The `state:modified+` selector:**

```bash
# Build modified models AND their downstream dependencies
$ dbt build --select state:modified+ --state ./prod-artifacts/

# state:modified includes models where:
# - The SQL file changed
# - The YAML config changed (tests, descriptions, configs)
# - The macro used by the model changed
# - The schema.yml changed

# The + suffix means "and all downstream dependencies"
# Without +, only the modified models themselves are built
```

**State comparison selectors:**

```
┌────────────────────────────────────────────────────────────┐
│  State Selectors                                           │
├────────────────────────┬───────────────────────────────────┤
│ state:modified         │ Models that changed (SQL or YAML) │
│ state:modified+        │ Changed models + downstream       │
│ state:new              │ Models that are new (not in state) │
│ state:modified.body    │ Only SQL body changes              │
│ state:modified.configs │ Only config/YAML changes           │
│ result:error           │ Models that errored last run       │
│ result:fail            │ Tests that failed last run         │
└────────────────────────┴───────────────────────────────────┘
```

**GitHub Actions CI workflow:**

```yaml
# .github/workflows/dbt-ci.yml
name: dbt Slim CI

on:
  pull_request:
    branches: [main]

jobs:
  dbt-ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dbt
        run: pip install dbt-snowflake

      - name: Install packages
        run: dbt deps

      - name: Download production manifest
        # Fetch the latest prod manifest from artifact storage
        run: |
          mkdir -p prod-artifacts
          aws s3 cp s3://dbt-artifacts/prod/manifest.json prod-artifacts/

      - name: dbt build (Slim CI)
        run: |
          dbt build \
            --select state:modified+ \
            --state ./prod-artifacts/ \
            --target ci
        env:
          DBT_PROFILES_DIR: .
          SNOWFLAKE_ACCOUNT: ${{ secrets.SNOWFLAKE_ACCOUNT }}
          SNOWFLAKE_PASSWORD: ${{ secrets.SNOWFLAKE_PASSWORD }}

      - name: Upload CI artifacts
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: dbt-ci-artifacts
          path: target/
```

### 💡 Interview Insight

> **Common Question**: "How do you implement CI/CD for dbt?"
> **Strong Answer**: "Slim CI using `dbt build --select state:modified+`. On every PR, the CI pipeline downloads the production `manifest.json`, compares it against the current branch, and only builds modified models plus their downstream dependencies. This turns a 45-minute full build into a 2-minute targeted build. I store the production manifest as a CI artifact after each successful production run. The CI target uses an isolated schema (e.g., `ci_pr_123`) so PR builds don't affect production data."

---

## Screen 3: Artifacts — `manifest.json` and `run_results.json`

dbt generates metadata artifacts in the `target/` directory after every run. These files are the foundation for Slim CI, documentation, and monitoring.

**Key artifacts:**

```
target/
├── manifest.json        # Complete project graph (models, tests, sources, macros)
├── run_results.json     # Results of the latest run (timing, status, rows)
├── catalog.json         # Column-level metadata (after dbt docs generate)
├── sources.json         # Source freshness results
└── compiled/            # Compiled SQL for every model
    └── your_project/
        └── models/
            └── marts/
                └── fct_orders.sql   # The actual SQL sent to warehouse
```

**`manifest.json`** — The project graph:

```json
{
  "nodes": {
    "model.ecommerce.fct_orders": {
      "unique_id": "model.ecommerce.fct_orders",
      "resource_type": "model",
      "depends_on": {
        "nodes": [
          "model.ecommerce.stg_orders",
          "model.ecommerce.stg_customers"
        ]
      },
      "config": {
        "materialized": "incremental",
        "unique_key": "order_id"
      },
      "columns": { ... },
      "compiled_code": "SELECT ... FROM analytics.staging.stg_orders ...",
      "checksum": { "name": "sha256", "checksum": "abc123..." }
    }
  },
  "sources": { ... },
  "macros": { ... },
  "exposures": { ... }
}
```

**`run_results.json`** — Execution metrics:

```json
{
  "results": [
    {
      "unique_id": "model.ecommerce.fct_orders",
      "status": "success",
      "execution_time": 12.45,
      "rows_affected": 245000,
      "adapter_response": {
        "rows_affected": 245000,
        "bytes_processed": 1048576
      }
    },
    {
      "unique_id": "test.ecommerce.unique_fct_orders_order_id",
      "status": "pass",
      "execution_time": 0.8,
      "failures": 0
    }
  ],
  "elapsed_time": 142.3,
  "metadata": {
    "dbt_version": "1.7.0",
    "generated_at": "2024-06-15T06:15:00Z"
  }
}
```

**Using artifacts for monitoring:**

```python
# scripts/check_run_results.py
# Parse run_results.json to detect anomalies

import json
from datetime import datetime

with open("target/run_results.json") as f:
    results = json.load(f)

failed = [r for r in results["results"] if r["status"] in ("error", "fail")]
slow = [r for r in results["results"]
        if r.get("execution_time", 0) > 300]  # > 5 minutes

if failed:
    print(f"🔴 {len(failed)} failures detected!")
    for f_item in failed:
        print(f"  - {f_item['unique_id']}: {f_item['status']}")

if slow:
    print(f"🐌 {len(slow)} slow models (>5 min):")
    for s in slow:
        print(f"  - {s['unique_id']}: {s['execution_time']:.1f}s")
```

```
┌────────────────────────────────────────────────────────────────┐
│           Artifact Usage Matrix                                │
├────────────────────┬───────────────────────────────────────────┤
│ Artifact           │ Used For                                  │
├────────────────────┼───────────────────────────────────────────┤
│ manifest.json      │ Slim CI (state comparison)                │
│                    │ dbt docs (lineage graph)                  │
│                    │ External tools (dbt Cloud, Elementary)    │
│                    │                                           │
│ run_results.json   │ Post-run monitoring & alerting            │
│                    │ Performance trending                      │
│                    │ Retry failed models (result:error)        │
│                    │                                           │
│ catalog.json       │ Column-level documentation                │
│                    │ Schema change detection                   │
│                    │                                           │
│ sources.json       │ Source freshness monitoring               │
│                    │ Ingestion SLA tracking                    │
└────────────────────┴───────────────────────────────────────────┘
```

### 💡 Interview Insight

> **Common Question**: "What are dbt artifacts and how do you use them?"
> **Strong Answer**: "`manifest.json` contains the full project graph — every model, test, source, and their dependencies. I use it for Slim CI (comparing what changed between PRs and production) and documentation. `run_results.json` captures execution results — timing, status, row counts. I parse it post-run to detect failures, alert on slow models, and track performance trends. In production, I upload both artifacts to S3 after every successful run so CI pipelines can reference them."

---

## Screen 4: dbt with Airflow — Orchestration Patterns

dbt handles transformation, but it doesn't handle scheduling, dependencies on external systems (ingestion completion), or alerting. That's where an orchestrator like **Airflow** comes in.

**Pattern 1: BashOperator (simple but brittle):**

```python
# dags/dbt_daily.py — Basic approach
from airflow import DAG
from airflow.operators.bash import BashOperator
from datetime import datetime

with DAG(
    dag_id="dbt_daily_build",
    schedule_interval="0 6 * * *",  # 6am daily
    start_date=datetime(2024, 1, 1),
    catchup=False,
    tags=["dbt", "analytics"],
) as dag:

    dbt_deps = BashOperator(
        task_id="dbt_deps",
        bash_command="cd /opt/dbt/ecommerce && dbt deps",
    )

    dbt_source_freshness = BashOperator(
        task_id="dbt_source_freshness",
        bash_command="cd /opt/dbt/ecommerce && dbt source freshness",
    )

    dbt_build = BashOperator(
        task_id="dbt_build",
        bash_command="cd /opt/dbt/ecommerce && dbt build --target prod",
    )

    dbt_docs = BashOperator(
        task_id="dbt_docs_generate",
        bash_command="cd /opt/dbt/ecommerce && dbt docs generate",
    )

    dbt_deps >> dbt_source_freshness >> dbt_build >> dbt_docs
```

**Pattern 2: Cosmos (recommended) — dbt tasks as Airflow nodes:**

```python
# dags/dbt_cosmos_daily.py — Production-grade approach
from airflow import DAG
from cosmos import DbtTaskGroup, ProjectConfig, ProfileConfig, RenderConfig
from cosmos.constants import LoadMode
from datetime import datetime

profile_config = ProfileConfig(
    profile_name="ecommerce",
    target_name="prod",
    profiles_yml_filepath="/opt/dbt/.dbt/profiles.yml",
)

with DAG(
    dag_id="dbt_cosmos_daily",
    schedule_interval="0 6 * * *",
    start_date=datetime(2024, 1, 1),
    catchup=False,
    tags=["dbt", "analytics"],
) as dag:

    # Each dbt model becomes an Airflow task!
    dbt_tasks = DbtTaskGroup(
        group_id="dbt_build",
        project_config=ProjectConfig(
            dbt_project_path="/opt/dbt/ecommerce",
        ),
        profile_config=profile_config,
        render_config=RenderConfig(
            load_method=LoadMode.DBT_MANIFEST,
            select=["path:models/staging", "path:models/marts"],
        ),
        # Each model is a separate Airflow task
        # Tests run as separate tasks after their models
        # DAG parallelism is preserved!
    )
```

**Cosmos advantages over BashOperator:**

```
┌──────────────────────────────────────────────────────────────────┐
│           BashOperator vs Cosmos                                 │
├────────────────────┬─────────────────────┬───────────────────────┤
│ Feature            │ BashOperator        │ Cosmos                │
├────────────────────┼─────────────────────┼───────────────────────┤
│ Granularity        │ One task = entire   │ One task = one model  │
│                    │ dbt build           │                       │
│ Retry on failure   │ Retries everything  │ Retries single model  │
│ Parallelism        │ dbt handles it      │ Airflow handles it    │
│ Visibility         │ One green/red box   │ Full DAG visibility   │
│ Partial success    │ All-or-nothing      │ Some models succeed   │
│ Setup complexity   │ Low                 │ Medium                │
│ Monitoring         │ Parse logs manually │ Airflow UI per model  │
└────────────────────┴─────────────────────┴───────────────────────┘
```

**Production DAG pattern — ingestion → transformation → reporting:**

```
┌────────────────────────────────────────────────────────────────┐
│           Production Airflow DAG                               │
│                                                                │
│  [wait_for_fivetran_sync]                                      │
│         │                                                      │
│         ▼                                                      │
│  [dbt_source_freshness]                                        │
│         │                                                      │
│         ▼                                                      │
│  [dbt_build_staging]  ──→  [dbt_build_marts]                   │
│         │                       │                              │
│         ▼                       ▼                              │
│  [dbt_snapshot]           [dbt_build_reports]                   │
│                                 │                              │
│                                 ▼                              │
│                          [upload_artifacts]                     │
│                                 │                              │
│                                 ▼                              │
│                          [notify_slack]                         │
└────────────────────────────────────────────────────────────────┘
```

### 💡 Interview Insight

> **Common Question**: "How do you orchestrate dbt in production?"
> **Strong Answer**: "I use Airflow with Cosmos (astronomer-cosmos). Cosmos maps each dbt model to an individual Airflow task, giving per-model visibility, retry, and parallelism in the Airflow UI. The DAG flow is: wait for ingestion completion → check source freshness → `dbt build` staging → marts → reports → upload artifacts → notify. For simpler setups, BashOperator works but loses per-model granularity. I store dbt artifacts (manifest.json, run_results.json) after each successful run for Slim CI."

---

## Screen 5: SQLFluff — SQL Linting for dbt

SQLFluff is the standard linter for dbt SQL files. It enforces consistent formatting, catches style violations, and integrates with CI pipelines and pre-commit hooks.

**Installation and configuration:**

```bash
# Install SQLFluff with dbt templater
$ pip install sqlfluff sqlfluff-templater-dbt
```

```yaml
# .sqlfluff (project root)
[sqlfluff]
templater = dbt
dialect = snowflake
max_line_length = 120
indent_unit = space

[sqlfluff:templater:dbt]
project_dir = .
profiles_dir = .
profile = ecommerce
target = dev

[sqlfluff:indentation]
indent_unit = space
tab_space_size = 4
indented_joins = false
indented_ctes = true
indented_using_on = true

[sqlfluff:rules:capitalisation.keywords]
capitalisation_policy = lower

[sqlfluff:rules:capitalisation.identifiers]
capitalisation_policy = lower

[sqlfluff:rules:capitalisation.functions]
capitalisation_policy = lower

[sqlfluff:rules:capitalisation.literals]
capitalisation_policy = lower

[sqlfluff:rules:aliasing.table]
aliasing = explicit

[sqlfluff:rules:aliasing.column]
aliasing = explicit

[sqlfluff:rules:aliasing.length]
min_alias_length = 2
```

**Running SQLFluff:**

```bash
# Lint a specific file
$ sqlfluff lint models/marts/fct_orders.sql

# Lint the entire project
$ sqlfluff lint models/

# Auto-fix violations
$ sqlfluff fix models/marts/fct_orders.sql

# Check a specific rule
$ sqlfluff lint models/ --rules LT01,LT02
```

**Common SQLFluff rules for dbt projects:**

```
┌────────────────────────────────────────────────────────────────┐
│  Key SQLFluff Rules                                            │
├──────────┬─────────────────────────────────────────────────────┤
│ LT01     │ Trailing whitespace                                 │
│ LT02     │ Indentation not a multiple of configured spaces     │
│ LT04     │ Leading commas vs trailing commas                   │
│ AM01     │ Non-ANSI-92 joins (implicit joins)                  │
│ ST06     │ SELECT * in production models                       │
│ RF01     │ References to undeclared source/ref                 │
│ CP01-04  │ Capitalization (keywords, identifiers, functions)   │
│ AL01-03  │ Aliasing rules (explicit AS, min length)            │
│ JJ01     │ Jinja template spacing                              │
└──────────┴─────────────────────────────────────────────────────┘
```

**Example — before and after SQLFluff fix:**

```sql
-- BEFORE (violations marked)
SELECT o.Order_ID,                    -- CP02: inconsistent casing
       c.first_name,
    SUM(o.amount) total               -- AL02: missing explicit AS
FROM {{ ref('stg_orders') }} o        -- AL01: short alias
join {{ ref('stg_customers') }} c     -- CP01: lowercase JOIN keyword
ON o.customer_id = c.customer_id
GROUP BY 1,2                          -- LT01: missing spaces
```

```sql
-- AFTER (sqlfluff fix)
select
    o.order_id,
    c.first_name,
    sum(o.amount) as total
from {{ ref('stg_orders') }} as o
join {{ ref('stg_customers') }} as c
    on o.customer_id = c.customer_id
group by 1, 2
```

### 💡 Interview Insight

> **Common Question**: "How do you enforce SQL style consistency in a dbt project?"
> **Strong Answer**: "SQLFluff with the dbt templater. I configure `.sqlfluff` at the project root with dialect-specific rules — consistent keyword casing, explicit aliases with `AS`, 4-space indentation, and 120-char line limits. It runs in CI as a PR check (fail on violations) and locally via pre-commit hooks. The `sqlfluff fix` command auto-corrects most violations. Consistent SQL style makes code reviews faster and reduces cognitive load."

---

## Screen 6: Pre-commit Hooks for dbt Projects

Pre-commit hooks catch issues **before** code reaches the repo — linting, YAML validation, and dbt compilation happen automatically on every `git commit`.

**Setup:**

```bash
$ pip install pre-commit
```

```yaml
# .pre-commit-config.yaml
repos:
  # General file checks
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
        args: ['--allow-multiple-documents']
      - id: check-added-large-files
        args: ['--maxkb=500']
      - id: no-commit-to-branch
        args: ['--branch', 'main']

  # SQLFluff linting
  - repo: https://github.com/sqlfluff/sqlfluff
    rev: 3.0.0
    hooks:
      - id: sqlfluff-lint
        args: ['--dialect', 'snowflake']
        additional_dependencies:
          - sqlfluff-templater-dbt
        files: ^models/
      - id: sqlfluff-fix
        args: ['--dialect', 'snowflake', '--force']
        additional_dependencies:
          - sqlfluff-templater-dbt
        files: ^models/

  # YAML schema validation for dbt
  - repo: https://github.com/python-jsonschema/check-jsonschema
    rev: 0.28.0
    hooks:
      - id: check-jsonschema
        name: Validate dbt_project.yml
        files: dbt_project.yml
        args: ['--schemafile', 'https://raw.githubusercontent.com/dbt-labs/dbt-jsonschema/main/schemas/dbt_project.json']

  # dbt-specific checks
  - repo: https://github.com/dbt-checkpoint/dbt-checkpoint
    rev: v2.0.1
    hooks:
      - id: check-model-has-tests
        args: ['--test-cnt', '2', '--']  # At least 2 tests per model
      - id: check-model-has-description
        args: ['--']
      - id: check-source-has-freshness
        args: ['--']
      - id: check-model-name-contract
        args: ['--pattern', '(stg|int|fct|dim|rpt)_.*', '--']
```

**Install and run:**

```bash
# Install hooks (one-time per clone)
$ pre-commit install

# Now every `git commit` runs these checks automatically

# Run manually against all files
$ pre-commit run --all-files

# Skip hooks for emergency commits (use sparingly!)
$ git commit --no-verify -m "hotfix: emergency production fix"
```

**What `dbt-checkpoint` validates:**

```
┌────────────────────────────────────────────────────────────────┐
│  dbt-checkpoint Hooks                                          │
├───────────────────────────────┬────────────────────────────────┤
│ check-model-has-tests         │ Every model has ≥N tests       │
│ check-model-has-description   │ Every model has a description  │
│ check-model-has-properties    │ Model is defined in YAML       │
│ check-source-has-freshness    │ Sources have freshness config  │
│ check-model-name-contract     │ Names match pattern (stg_...)  │
│ check-model-parents-schema    │ Marts don't ref sources direct │
│ check-script-has-no-table-name│ No hardcoded table references  │
└───────────────────────────────┴────────────────────────────────┘
```

### 💡 Interview Insight

> **Common Question**: "What pre-commit hooks do you use for dbt?"
> **Strong Answer**: "I use `pre-commit` with several hooks: SQLFluff for SQL linting, `dbt-checkpoint` for structural validation (every model has tests, descriptions, proper naming), YAML schema validation, and standard checks (trailing whitespace, large file detection, no direct commits to main). This catches 90% of PR review issues before code even reaches GitHub, making reviews faster and more focused on logic."

---

## Screen 7: Documentation — `dbt docs generate` and `dbt docs serve`

dbt auto-generates a documentation website from your project — model descriptions, column definitions, lineage graphs, and test coverage. It's one of dbt's most underrated features.

**Generating and serving docs:**

```bash
# Generate documentation (creates target/catalog.json)
$ dbt docs generate

# Serve locally on port 8080
$ dbt docs serve --port 8081   # Avoid port 8080 (Teams uses it)

# Generate and serve in one shot
$ dbt docs generate && dbt docs serve --port 8081
```

**What documentation includes:**

```
┌──────────────────────────────────────────────────────────────────┐
│           dbt Docs Features                                      │
│                                                                  │
│  📊 Lineage Graph                                                │
│  └── Visual DAG showing all models and their dependencies        │
│      Click any model to see upstream/downstream                  │
│                                                                  │
│  📋 Model Details                                                │
│  └── Description, SQL code, compiled code                        │
│      Column names, types, descriptions                           │
│      Tests attached to each column                               │
│      Config (materialization, tags, schema)                       │
│                                                                  │
│  🔗 Source Details                                               │
│  └── Database, schema, table names                               │
│      Freshness configuration                                     │
│      Columns and descriptions                                    │
│                                                                  │
│  📈 Test Coverage                                                │
│  └── Which models have tests                                     │
│      Which columns are tested                                    │
│      Test types and severity                                     │
└──────────────────────────────────────────────────────────────────┘
```

**Writing good documentation:**

```yaml
# models/marts/_marts_schema.yml
version: 2

models:
  - name: fct_orders
    description: >
      **Fact table for all customer orders.** One row per order.
      Includes order amounts in dollars (converted from cents in staging).
      Updated incrementally — merge on order_id with 3-day lookback.

      **Grain**: One row per `order_id`.
      **Update frequency**: Daily at 6am UTC.
      **Owner**: Analytics Engineering team.
    columns:
      - name: order_id
        description: >
          Primary key. Unique order identifier from the Shopify source system.
          Maps 1:1 to `raw_orders.id`.
        tests:
          - unique
          - not_null
      - name: order_amount_dollars
        description: >
          Total order value in USD. Converted from cents in `stg_orders`.
          Includes product cost + shipping + tax. Does NOT include discounts
          (see `discount_amount_dollars` column).
        tests:
          - not_null
```

**`docs` blocks for rich descriptions:**

```sql
-- models/marts/fct_orders.sql

{% docs fct_orders_description %}
# Orders Fact Table

This table contains **one row per order** across all sales channels.

## Key business rules
- Order amounts are in **USD dollars** (converted from cents)
- Returns are tracked via the `status` column ('returned')
- `order_date` reflects the date the order was **placed**, not shipped

## Update schedule
- Incremental daily at 6am UTC
- Full refresh every Sunday at 2am UTC

## Known caveats
- International orders have amounts converted at daily exchange rates
- Gift card orders may show $0 for `order_amount_dollars`
{% enddocs %}
```

```yaml
# Reference the docs block in YAML
models:
  - name: fct_orders
    description: '{{ doc("fct_orders_description") }}'
```

**Exposures — connecting dbt to downstream consumers:**

```yaml
# models/exposures.yml
version: 2

exposures:
  - name: weekly_revenue_dashboard
    type: dashboard
    maturity: high
    url: https://lookerstudio.google.com/your-dashboard
    description: >
      Executive revenue dashboard showing weekly KPIs.
      Refreshed daily at 7am UTC after dbt build completes.
    depends_on:
      - ref('fct_orders')
      - ref('dim_customers')
      - ref('rpt_weekly_revenue')
    owner:
      name: Analytics Team
      email: analytics@company.com

  - name: customer_churn_ml_model
    type: ml
    maturity: medium
    description: "ML model predicting customer churn risk"
    depends_on:
      - ref('fct_customer_activity')
      - ref('dim_customers')
    owner:
      name: Data Science Team
      email: datascience@company.com
```

Exposures show up in the lineage graph, creating end-to-end visibility from source → staging → marts → dashboards/ML models.

### 💡 Interview Insight

> **Common Question**: "How do you document your dbt project?"
> **Strong Answer**: "Every model and column gets descriptions in YAML — including grain, update frequency, business rules, and caveats. I use `{% docs %}` blocks for rich markdown descriptions with headers and bullet points. I define exposures for dashboards and ML models that consume dbt models — this creates end-to-end lineage from source to consumer. `dbt docs generate` produces a browsable website with lineage graphs. I host docs on an internal server and link them in our data catalog."

---

## Screen 8: Putting It All Together — The Production Workflow

Here's the complete production workflow for a dbt project, from code change to deployed and monitored.

```
┌──────────────────────────────────────────────────────────────────┐
│           Complete Production Workflow                            │
│                                                                  │
│  DEVELOPMENT:                                                    │
│  1. Developer creates feature branch                             │
│  2. Writes/modifies models, tests, YAML                         │
│  3. Runs locally: dbt build --select state:modified+             │
│  4. Commits → pre-commit hooks run (SQLFluff, dbt-checkpoint)   │
│  5. Pushes branch, opens PR                                     │
│                                                                  │
│  CI (Pull Request):                                              │
│  6. GitHub Actions triggers Slim CI                              │
│  7. dbt build --select state:modified+ --state prod-artifacts/  │
│  8. SQLFluff lint --diff (only changed files)                    │
│  9. dbt docs generate (check for missing docs)                  │
│  10. PR comment: models built, tests passed, timing             │
│                                                                  │
│  MERGE → DEPLOY:                                                 │
│  11. PR approved, merged to main                                 │
│  12. Deploy branch triggers production build                     │
│  13. dbt build --target prod                                     │
│  14. Upload artifacts: manifest.json, run_results.json → S3     │
│  15. dbt docs generate → deploy to internal docs site            │
│                                                                  │
│  SCHEDULED PRODUCTION:                                           │
│  16. Airflow DAG: 6am daily                                      │
│      a. Wait for Fivetran sync completion                        │
│      b. dbt source freshness                                     │
│      c. dbt build (incremental)                                  │
│      d. Upload artifacts                                         │
│      e. Parse run_results → Slack alert on failures              │
│  17. Sunday 2am: dbt build --full-refresh (weekly maintenance)   │
│                                                                  │
│  MONITORING:                                                     │
│  18. Track model execution times (trending via run_results.json) │
│  19. Track test failure rates                                    │
│  20. Source freshness dashboards                                 │
│  21. Data observability (Elementary, Monte Carlo, etc.)           │
└──────────────────────────────────────────────────────────────────┘
```

**Essential `dbt_project.yml` configs for production:**

```yaml
# dbt_project.yml
name: ecommerce
version: '1.0.0'
profile: ecommerce
config-version: 2

model-paths: ["models"]
test-paths: ["tests"]
snapshot-paths: ["snapshots"]
macro-paths: ["macros"]
seed-paths: ["seeds"]
analysis-paths: ["analyses"]
target-path: "target"
clean-targets: ["target", "dbt_packages"]

models:
  ecommerce:
    staging:
      +materialized: view
      +schema: staging
      +tags: ['staging']
    marts:
      +materialized: table
      +schema: marts
      +tags: ['marts']
    reports:
      +materialized: table
      +schema: reports
      +tags: ['reports']

on-run-end:
  - "{{ grant_select_on_schemas(['staging', 'marts', 'reports'], 'analyst_role') }}"

vars:
  lookback_days: 3
  start_date: '2020-01-01'
```

### 💡 Interview Insight

> **Common Question**: "Walk me through your dbt production workflow end-to-end."
> **Strong Answer**: "Development: feature branch → local dbt build → pre-commit hooks (SQLFluff, dbt-checkpoint). CI: PR triggers Slim CI with `state:modified+` against the production manifest — only changed models build. On merge: production deploy runs full `dbt build`. Scheduled: Airflow DAG at 6am — waits for ingestion, checks source freshness, runs `dbt build`, uploads artifacts, alerts Slack on failures. Weekly: full refresh for data quality assurance. I also monitor execution times and test failure trends from run_results.json."

---

## Screen 9: Quiz — Module 7 Review

### Question 1
**What is the main advantage of `dbt build` over `dbt run` + `dbt test`?**

A) `dbt build` is faster
B) `dbt build` runs tests in DAG order, gating downstream models on test results
C) `dbt build` supports more model types
D) `dbt build` generates documentation

**Answer: B** — `dbt build` executes models and tests in DAG order. If a model's tests fail, downstream models are skipped. This prevents bad data from propagating. Running `dbt run` + `dbt test` separately builds all models first (including downstream ones that may consume bad data) before any tests run.

---

### Question 2
**What does `dbt build --select state:modified+ --state ./prod-artifacts/` do?**

A) Builds all models in the modified state
B) Compares current project against prod manifest, builds only changed models and their downstream dependencies
C) Modifies the production state
D) Restores models to their production state

**Answer: B** — This is Slim CI. dbt compares the current project's manifest against the production manifest (from `./prod-artifacts/`), identifies which models changed, and builds only those models plus everything downstream (the `+` suffix). This dramatically reduces CI build times.

---

### Question 3
**Which artifact is used for Slim CI state comparison?**

A) `run_results.json`
B) `catalog.json`
C) `manifest.json`
D) `sources.json`

**Answer: C** — `manifest.json` contains the complete project graph including model checksums (hashes of SQL + config). Slim CI compares the PR's manifest against the production manifest to determine what changed. `run_results.json` contains execution results, not project structure.

---

### Question 4
**What is the main advantage of using Cosmos over BashOperator for dbt in Airflow?**

A) Cosmos is faster
B) Cosmos maps each dbt model to an individual Airflow task, enabling per-model retry and visibility
C) Cosmos doesn't require Airflow
D) Cosmos supports more databases

**Answer: B** — Cosmos creates one Airflow task per dbt model, giving you per-model visibility in the Airflow UI, granular retry (retry just the failed model, not the entire build), and Airflow-managed parallelism. BashOperator runs the entire `dbt build` as one opaque task.

---

### Question 5
**A pre-commit hook using `dbt-checkpoint` with `check-model-has-tests --test-cnt 2` fails on your commit. What does this mean?**

A) Two of your tests are failing
B) At least one model in your commit has fewer than 2 tests defined
C) You have more than 2 tests per model
D) Your test files have syntax errors

**Answer: B** — `check-model-has-tests` with `--test-cnt 2` requires every model to have at least 2 tests defined in YAML. If a model has 0 or 1 tests, the pre-commit hook blocks the commit, enforcing minimum test coverage.

---

### 🔑 Key Takeaways

1. **`dbt build` is the production standard** — models + tests + snapshots + seeds in DAG order with test gating
2. **Slim CI** uses `state:modified+` to build only changed models and downstream — reducing CI from 45 min to 2 min
3. **`manifest.json`** is the project graph — used for Slim CI, documentation, and external tools
4. **`run_results.json`** captures execution metrics — parse it for monitoring, alerting, and performance trending
5. **Cosmos** maps dbt models to individual Airflow tasks — per-model retry, visibility, and parallelism
6. **SQLFluff** enforces SQL style consistency — configure `.sqlfluff`, run in CI and pre-commit
7. **Pre-commit hooks** catch issues before code reaches the repo — SQLFluff, dbt-checkpoint, YAML validation
8. **dbt docs** generate a browsable website with lineage graphs — use `{% docs %}` blocks and exposures
9. **Exposures** connect dbt models to downstream consumers (dashboards, ML models) in the lineage graph
10. **Production workflow**: feature branch → pre-commit → Slim CI → merge → scheduled Airflow DAG → monitoring
