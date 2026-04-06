---
tags: [data-engineering, dbt, phase-2]
phase: 2
status: not-started
priority: high
---

# 🔨 dbt (data build tool)

> **Phase:** 2 | **Duration:** ~3 days | **Priority:** High
> **Related:** [[01 - Data Modeling]], [[03 - Data Warehousing]], [[01 - SQL Advanced]], [[04 - ETL ELT Pipelines]]

---

## Checklist

### dbt Core Concepts
- [ ] What dbt does: transforms data in the warehouse (the T in ELT)
- [ ] dbt Core (CLI, open source) vs dbt Cloud (managed)
- [ ] Project structure: `models/`, `tests/`, `macros/`, `seeds/`, `snapshots/`, `analyses/`
- [ ] `dbt_project.yml` — project configuration
- [ ] Profiles: warehouse connections (`profiles.yml`)
- [ ] `ref()` — model references, DAG construction
- [ ] `source()` — reference raw tables, freshness checks

### Models
- [ ] Materializations: `view`, `table`, `incremental`, `ephemeral`
- [ ] When to use each: view (simple), table (heavy transforms), incremental (large/append)
- [ ] Model naming conventions: `stg_`, `int_`, `fct_`, `dim_` (staging → intermediate → marts)
- [ ] Model layers: staging (1:1 source), intermediate (joins/logic), marts (business-ready)
- [ ] `config()` block: materialization, schema, tags, cluster_by
- [ ] Custom schemas: deploy models to different schemas

### Incremental Models
- [ ] `{{ config(materialized='incremental') }}` + `{% if is_incremental() %}`
- [ ] Incremental strategies: `append`, `merge`, `delete+insert`, `insert_overwrite`
- [ ] `unique_key` — deduplication key for merge strategy
- [ ] `on_schema_change`: `ignore`, `append_new_columns`, `sync_all_columns`, `fail`
- [ ] Full refresh: `dbt run --full-refresh` — rebuild from scratch
- [ ] Gotchas: late-arriving data, backfill handling

### Tests & Data Quality
- [ ] Generic tests: `unique`, `not_null`, `accepted_values`, `relationships`
- [ ] Custom tests: SQL-based, return failing rows
- [ ] `dbt-expectations` package — Great Expectations-like tests in dbt
- [ ] Source freshness: `loaded_at_field`, warn/error thresholds
- [ ] `dbt test` vs `dbt build` — running tests with models
- [ ] Test severity: `warn` vs `error`
- [ ] Contract enforcement: `data_tests`, `columns` with constraints

### Snapshots (SCD Type 2)
- [ ] `snapshot` blocks — track changes over time
- [ ] Strategy: `timestamp` (prefer) vs `check` (column comparison)
- [ ] Output columns: `dbt_valid_from`, `dbt_valid_to`, `dbt_scd_id`
- [ ] Hard deletes: `invalidate_hard_deletes` config
- [ ] When to use snapshots vs manual SCD logic

### Macros & Jinja
- [ ] Jinja in SQL: `{{ }}` expressions, `{% %}` control flow, `{# #}` comments
- [ ] `{{ ref() }}`, `{{ source() }}`, `{{ config() }}` — built-in macros
- [ ] Custom macros: reusable SQL functions, DRY principle
- [ ] `{% if %}`, `{% for %}` — conditional logic, looping
- [ ] `{{ adapter.dispatch() }}` — cross-database macros
- [ ] `dbt_utils` package: `surrogate_key`, `pivot`, `union_relations`, `generate_series`

### dbt Packages
- [ ] `packages.yml` — dependency management
- [ ] Key packages: `dbt_utils`, `dbt_expectations`, `dbt_date`, `codegen`
- [ ] `dbt deps` — install packages
- [ ] Writing custom packages

### CI/CD & dbt in Production
- [ ] `dbt build` — models + tests + snapshots in DAG order
- [ ] Slim CI: `dbt build --select state:modified+` — only changed models
- [ ] `manifest.json` and `run_results.json` — metadata artifacts
- [ ] dbt with [[05 - Orchestration]]: Airflow `DbtTaskGroup`, Cosmos
- [ ] SQLFluff — SQL linting for dbt models
- [ ] Pre-commit hooks for dbt projects
- [ ] Documentation: `dbt docs generate`, `dbt docs serve`

---

## 📝 Notes

_Start writing notes here as you study..._

---

## 🔗 Resources
- [ ] dbt docs: https://docs.getdbt.com/
- [ ] dbt Learn (free course): https://courses.getdbt.com/
- [ ] "Analytics Engineering with dbt" (O'Reilly)
- [ ] dbt Discourse: community Q&A

---

## 💡 Key Takeaways

1. 

---

## ❓ Interview Questions to Practice
- Explain dbt's role in the modern data stack
- When would you use incremental models? What are the gotchas?
- How do you implement CI/CD for a dbt project?
- Design a dbt project structure for an e-commerce company
