---
tags: [sql, databases, phase-1]
phase: 1
status: not-started
priority: high
---

# 🗄️ SQL Advanced

> **Phase:** 1 | **Duration:** ~4 days | **Priority:** High
> **Related:** [[01 - Data Modeling]], [[03 - Data Warehousing]], [[02 - Spark SQL DataFrames]], [[01 - dbt]]

---

## Checklist

### Window Functions Mastery
- [ ] `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()` — differences and when to use each
- [ ] `LEAD()` / `LAG()` — accessing adjacent rows, gap analysis
- [ ] `FIRST_VALUE()` / `LAST_VALUE()` / `NTH_VALUE()` — frame-aware
- [ ] `PARTITION BY` + `ORDER BY` — defining windows
- [ ] Frame specification: `ROWS` vs `RANGE` vs `GROUPS`
- [ ] `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` — running totals
- [ ] `SUM() OVER`, `AVG() OVER`, `COUNT() OVER` — running aggregates
- [ ] `PERCENT_RANK()`, `CUME_DIST()`, `NTILE()` — statistical windows
- [ ] Practical: sessionization with window functions
- [ ] Practical: running totals, moving averages, YoY comparisons

### CTEs & Advanced Queries
- [ ] Common Table Expressions — readability, avoiding subquery nesting
- [ ] Recursive CTEs — hierarchical data, graph traversal, sequence generation
- [ ] Lateral joins — `LATERAL` / `CROSS APPLY`, correlated subqueries made cleaner
- [ ] `GROUPING SETS`, `CUBE`, `ROLLUP` — multi-level aggregations
- [ ] `UNION ALL` vs `UNION` — performance implications
- [ ] Correlated vs non-correlated subqueries
- [ ] `EXISTS` vs `IN` vs `JOIN` — performance comparison

### Query Performance & Optimization
- [ ] `EXPLAIN ANALYZE` — reading query plans (Seq Scan, Index Scan, Hash Join, etc.)
- [ ] Index types: B-tree (default), Hash, GIN (full-text/JSONB), GiST (spatial)
- [ ] Composite indexes — column order matters, leftmost prefix rule
- [ ] Covering indexes — `INCLUDE` clause, index-only scans
- [ ] Join algorithms: Nested Loop, Hash Join, Merge Join — when optimizer picks each
- [ ] Statistics: `ANALYZE`, histogram buckets, selectivity estimation
- [ ] Query rewriting: predicate pushdown, join reordering
- [ ] Materialized views — when to use, refresh strategies
- [ ] Partitioning: range, list, hash — pruning benefits

### PostgreSQL Specifics
- [ ] JSONB: operators (`->`, `->>`, `@>`), indexing with GIN, `jsonb_array_elements`
- [ ] Table partitioning: declarative partitioning, partition pruning
- [ ] `VACUUM` and `AUTOVACUUM` — dead tuples, bloat, tuning
- [ ] Connection pooling: PgBouncer — transaction vs session mode
- [ ] `pg_stat_statements` — finding slow queries
- [ ] Advisory locks, row-level locking, deadlock detection
- [ ] `COPY` command — bulk loading performance

### NoSQL (Awareness Level)
- [ ] DynamoDB: partition key design, sort keys, GSIs, single-table design pattern
- [ ] MongoDB: document model, aggregation pipeline, indexes
- [ ] Redis: data structures (strings, hashes, lists, sets, sorted sets)
- [ ] Redis patterns: caching (TTL, LRU), pub/sub, Streams
- [ ] Neo4j basics — nodes, relationships, Cypher query language (for [[03 - RAG Architecture]] knowledge graphs)
- [ ] When SQL vs NoSQL — decision framework

---

## 📝 Notes

_Start writing notes here as you study..._

---

## 🔗 Resources
- [ ] "SQL Performance Explained" by Markus Winand (use-the-index-luke.com)
- [ ] PostgreSQL docs: Window Functions
- [ ] Mode Analytics SQL Tutorial (advanced sections)
- [ ] LeetCode SQL problems (Medium/Hard)

---

## 💡 Key Takeaways

1. 

---

## ❓ Interview Questions to Practice
- Write a query to find the top 3 products by revenue per category
- Explain the difference between `ROW_NUMBER`, `RANK`, and `DENSE_RANK`
- How would you optimize a slow query? Walk through the process
- Design a DynamoDB table for a ride-sharing app
