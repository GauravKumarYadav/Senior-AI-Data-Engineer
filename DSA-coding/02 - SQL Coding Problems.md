---
tags: [sql, coding, interview, phase-8]
phase: 8
status: not-started
priority: high
---

# 🗄️ SQL Coding Problems

> **Phase:** 8 | **Duration:** ~1 week | **Priority:** High
> **Related:** [[01 - SQL Advanced]], [[01 - DSA Coding]], [[01 - Data Modeling]]

---

## Checklist

### SQL Problem Categories

#### Window Functions Problems
- [ ] Rank employees by salary within each department
- [ ] Find running total of sales per customer
- [ ] Calculate month-over-month revenue growth
- [ ] Find the Nth highest salary (without LIMIT/OFFSET)
- [ ] Identify consecutive login days per user
- [ ] Moving average of 7-day sales

#### Join Problems
- [ ] Self-join: find employees earning more than their manager
- [ ] Cross join: generate date spine, product combinations
- [ ] Anti-join: find customers with no orders (LEFT JOIN + IS NULL)
- [ ] Multiple joins: 3+ table joins with conditions
- [ ] Inequality joins: find overlapping date ranges

#### Aggregation Problems
- [ ] HAVING clause: groups meeting conditions
- [ ] Conditional aggregation: `SUM(CASE WHEN ...)`
- [ ] Pivot data: rows to columns using CASE
- [ ] Percent of total: using window function or subquery
- [ ] Histogram/bucketing: GROUP BY with CASE expressions

#### CTE & Subquery Problems
- [ ] Recursive CTE: generate date series, org hierarchy
- [ ] CTE for readability: break complex query into steps
- [ ] Correlated subquery: for each row, compute value from related rows
- [ ] EXISTS vs IN: performance and NULL handling differences
- [ ] Inline views: subquery in FROM clause

#### Data Engineering SQL Problems
- [ ] Deduplication: ROW_NUMBER() to keep latest record per key
- [ ] Gap and island: find consecutive sequences, missing values
- [ ] Sessionization: group events into sessions by time gap
- [ ] SCD Type 2: query for as-of date (point-in-time)
- [ ] Funnel analysis: conversion rates through steps
- [ ] Cohort analysis: retention by signup month
- [ ] Attribution modeling: first-touch, last-touch, multi-touch

### Practice Platforms
- [ ] LeetCode SQL (Medium/Hard): 20-30 problems
- [ ] HackerRank SQL: Advanced section
- [ ] DataLemur: SQL interview questions (free, categorized)
- [ ] StrataScratch: company-specific SQL problems

---

## 📝 Notes

_Start writing notes here as you study..._

---

## 🔗 Resources
- [ ] DataLemur: https://datalemur.com/
- [ ] LeetCode SQL Study Plan
- [ ] Mode Analytics SQL tutorial (advanced)
