---
tags: [python, data-engineering, phase-1]
phase: 1
status: not-started
priority: high
---

# 🐍 Python — Data Engineering

> **Phase:** 1 | **Duration:** ~3 days | **Priority:** High
> **Related:** [[01 - Python Core]], [[04 - ETL ELT Pipelines]], [[01 - Spark Core]], [[02 - Data Storage Formats]]

---

## Checklist

### File I/O at Scale
- [ ] Reading large files: chunked reads with `iter()` / generator patterns
- [ ] `mmap` — memory-mapped files for huge files without loading to RAM
- [ ] Binary file I/O — `struct.pack`/`unpack` for binary protocols
- [ ] Working with compressed files: `gzip`, `bz2`, `lzma` in streaming mode
- [ ] `pathlib` — modern path handling, glob patterns
- [ ] Temp files/dirs: `tempfile` module patterns for ETL

### Serialization & Data Formats
- [ ] `pickle` — fast but Python-only, security concerns (never unpickle untrusted)
- [ ] `json` — `json.dumps`/`loads`, custom encoders for dates/decimals
- [ ] `msgpack` — binary JSON, faster serialization
- [ ] `protobuf` — schema-defined serialization (used in [[01 - Streaming Kafka]])
- [ ] `avro` — schema evolution, used in [[02 - Data Storage Formats]]
- [ ] `pyarrow` — Arrow format, zero-copy reads, Parquet I/O
- [ ] Comparing formats: speed vs size vs schema evolution vs language support

### Performance & Profiling
- [ ] `cProfile` — function-level profiling, `snakeviz` visualization
- [ ] `line_profiler` — line-by-line execution time
- [ ] `memory_profiler` — `@profile` decorator, line-by-line memory
- [ ] `timeit` — microbenchmarking functions
- [ ] Optimization patterns: avoid loops → vectorize, use built-ins, list vs generator
- [ ] `functools.lru_cache` for memoization
- [ ] `__slots__` in data-heavy classes for memory reduction

### Packaging & Dependency Management
- [ ] `pyproject.toml` — modern Python packaging standard
- [ ] `uv` — fast dependency resolution & virtual environments
- [ ] `poetry` — dependency management, lock files, publishing
- [ ] Building wheels — `python -m build`, source vs binary distributions
- [ ] `pip install -e .` — editable installs for development
- [ ] Managing multiple Python versions: `pyenv`

### Testing for Data Pipelines
- [ ] `pytest` — fixtures, parametrize, markers, conftest.py
- [ ] Mocking I/O: `unittest.mock`, `pytest-mock`, patching external services
- [ ] Testing with `testcontainers` — spin up Postgres/Kafka in tests
- [ ] Data pipeline testing: schema validation, row count checks, data quality
- [ ] `pytest-cov` — coverage reports
- [ ] `hypothesis` — property-based testing for data transforms
- [ ] Integration vs unit tests — testing strategy for ETL

### Python CLI & Scripting
- [ ] `argparse` / `click` / `typer` — building CLI tools
- [ ] `logging` module — formatters, handlers, structured logging
- [ ] Environment variables: `os.environ`, `python-dotenv`, `pydantic-settings`
- [ ] `subprocess` — calling external commands safely

---

## 📝 Notes

_Start writing notes here as you study..._

---

## 🔗 Resources
- [ ] "High Performance Python" by Gorelick & Ozsvald
- [ ] Real Python: File Handling series
- [ ] PyArrow documentation
- [ ] uv docs: https://docs.astral.sh/uv/

---

## 💡 Key Takeaways

1. 

---

## ❓ Interview Questions to Practice
- How would you process a 50GB CSV file in Python without loading it all into memory?
- Compare serialization formats for a data pipeline (when to use which?)
- How do you test a data pipeline that reads from S3 and writes to Postgres?
