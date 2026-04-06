---
tags:
  - python
  - phase-1
  - fundamentals
phase: 1
status: started
priority: high
---

# 🐍 Python — Core

> **Phase:** 1 | **Duration:** ~4 days | **Priority:** High
> **Related:** [[02 - Python Data Engineering]], [[03 - Python AI-ML]], [[01 - Spark Core]]

---

## Checklist

### Data Structures Internals
- [ ] `list` internals — dynamic array, amortized O(1) append, resizing strategy
- [ ] `list` vs `collections.deque` — when to use which (O(1) popleft)
- [ ] `dict` ordering (insertion order since 3.7), hash table internals, collision resolution
- [ ] `set` — hashing, frozenset, when to prefer over list for lookups
- [ ] `defaultdict`, `Counter`, `OrderedDict` — practical use cases
- [ ] `heapq` — min-heap operations, nlargest/nsmallest, priority queue pattern
- [ ] `namedtuple` vs `dataclass` — when to use which
- [ ] Immutability — why tuples are hashable, implications for dict keys

### Generators & Iterators
- [ ] Iterator protocol: `__iter__` and `__next__`
- [ ] Generator functions with `yield` — lazy evaluation, memory efficiency
- [ ] `yield from` — delegating to sub-generators
- [ ] Generator expressions vs list comprehensions — `()` vs `[]`
- [ ] `itertools` essentials: `chain`, `islice`, `groupby`, `product`, `combinations`
- [ ] Building data pipelines with generators (read → transform → write)
- [ ] Coroutines with `send()` — basics for understanding async

### Decorators
- [ ] Function decorators — `@decorator` syntax, closure pattern
- [ ] `functools.wraps` — preserving function metadata
- [ ] Decorators with arguments — triple nesting pattern
- [ ] Class decorators — decorating classes, `__init_subclass__`
- [ ] Common decorators: `@property`, `@staticmethod`, `@classmethod`, `@lru_cache`
- [ ] Stacking decorators — execution order
- [ ] Practical: write a `@timer`, `@retry`, `@validate_schema` decorator

### Context Managers
- [ ] `__enter__` / `__exit__` protocol
- [ ] `contextlib.contextmanager` — generator-based context managers
- [ ] `contextlib.suppress`, `contextlib.redirect_stdout`
- [ ] Use cases: DB connections, file locks, temp directories, timing blocks
- [ ] Async context managers: `async with`, `__aenter__`/`__aexit__`

### Type Hints & Typing
- [ ] Basic types: `int`, `str`, `list[int]`, `dict[str, Any]`, `Optional`, `Union`
- [ ] `TypeVar`, `Generic` — writing generic functions/classes
- [ ] `Protocol` — structural subtyping (duck typing with types)
- [ ] `dataclasses` — `@dataclass`, `field()`, `__post_init__`, frozen dataclasses
- [ ] `Literal`, `TypedDict`, `Annotated` — advanced type hints
- [ ] Running `mypy` — catching type errors before runtime
- [ ] Pydantic v2 — runtime validation (bridge to [[01 - Backend FastAPI]])

### Concurrency & Parallelism
- [ ] `threading` — Thread, Lock, RLock, Semaphore, Event
- [ ] `multiprocessing` — Process, Pool, shared memory, `Manager`
- [ ] `concurrent.futures` — `ThreadPoolExecutor`, `ProcessPoolExecutor`
- [ ] `asyncio` — event loop, `async def`, `await`, `gather`, `create_task`
- [ ] `asyncio` internals — event loop, coroutines vs tasks vs futures
- [ ] When to use which: I/O bound (threading/async) vs CPU bound (multiprocessing)
- [ ] Practical: async HTTP client with `aiohttp`, async DB queries

### GIL (Global Interpreter Lock)
- [ ] What the GIL actually does — protects CPython reference counting
- [ ] What it blocks: CPU-bound Python code in threads
- [ ] What it doesn't block: I/O operations, C extensions (NumPy), multiprocessing
- [ ] GIL-free Python (3.13+ `--disable-gil`) — current state
- [ ] Implications for [[01 - Spark Core]] — why PySpark uses separate processes

### Memory Management
- [ ] Reference counting — how objects are freed
- [ ] `gc` module — generational garbage collection, cycle detection
- [ ] `__slots__` — memory savings for many instances
- [ ] Weak references — `weakref` module, avoiding circular references
- [ ] `sys.getsizeof()` — understanding object memory footprint
- [ ] Memory profiling with `memory_profiler`, `tracemalloc`
- [ ] Object interning — small integer and string caching

---

## 📝 Notes

_Start writing notes here as you study..._

---

## 🔗 Resources
- [ ] Python docs: Data Model
- [ ] "Fluent Python" by Luciano Ramalho (Chapters 1-9, 17-21)
- [ ] Real Python: Concurrency series
- [ ] David Beazley: "A Curious Course on Coroutines and Concurrency"

---

## 💡 Key Takeaways

1. 

---

## ❓ Interview Questions to Practice
- Explain the GIL and how it affects multi-threaded Python programs
- When would you use `asyncio` vs `multiprocessing`?
- Implement a generator-based data pipeline
- Write a decorator with arguments that retries a function N times
