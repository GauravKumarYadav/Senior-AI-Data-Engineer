---
tags: [python, ai-ml, phase-1]
phase: 1
status: not-started
priority: high
---

# 🐍 Python — AI/ML Libraries

> **Phase:** 1 | **Duration:** ~2 days | **Priority:** High
> **Related:** [[01 - Python Core]], [[01 - ML Fundamentals]], [[02 - Deep Learning Foundations]]

---

## Checklist

### NumPy Internals
- [ ] ndarray memory layout: contiguous (C-order vs Fortran-order), strides
- [ ] Broadcasting rules — shape compatibility, automatic expansion
- [ ] Vectorization — why loops are slow, ufuncs, `np.vectorize` (not actually fast)
- [ ] `np.einsum` — Einstein summation for matrix operations
- [ ] Views vs copies — when slicing creates a view, `.copy()` explicitly
- [ ] Random: `np.random.Generator` (new API), reproducibility with seeds
- [ ] Structured arrays — for tabular-like data without Pandas overhead

### Pandas Advanced
- [ ] Method chaining — `.pipe()`, `.assign()`, `.query()` for readable transforms
- [ ] `eval()` and `query()` — NumExpr backend for fast filtering
- [ ] Categorical dtype — memory savings, faster groupby
- [ ] MultiIndex — `set_index`, `unstack`, `stack`, `xs` (cross-section)
- [ ] Avoiding copies: `inplace` debate, copy-on-write (Pandas 2.0+)
- [ ] `pd.api.types` — type checking utilities
- [ ] Memory optimization: downcasting dtypes, sparse arrays
- [ ] GroupBy advanced: `transform`, `filter`, `apply` with custom functions
- [ ] Window operations: `rolling`, `expanding`, `ewm`
- [ ] Merging: `merge_asof` (time-based), indicator column, validate parameter

### Polars
- [ ] Why Polars: Rust-based, no GIL, true parallelism
- [ ] Lazy vs eager evaluation — `LazyFrame` vs `DataFrame`
- [ ] Expression API: `col()`, `when/then/otherwise`, `over()` (window functions)
- [ ] When to choose Polars over Pandas — decision framework
- [ ] Polars with Parquet — predicate pushdown, projection pushdown
- [ ] Converting between Pandas/Polars/Arrow — zero-copy when possible

---

## 📝 Notes

_Start writing notes here as you study..._

---

## 🔗 Resources
- [ ] NumPy docs: Broadcasting
- [ ] "Effective Pandas" by Matt Harrison
- [ ] Polars User Guide: https://pola.rs/
- [ ] Jake VanderPlas: "Python Data Science Handbook" (NumPy chapter)

---

## 💡 Key Takeaways

1. 

---

## ❓ Interview Questions to Practice
- Explain NumPy broadcasting with an example
- When would you choose Polars over Pandas?
- Optimize a slow Pandas pipeline — what techniques would you use?
