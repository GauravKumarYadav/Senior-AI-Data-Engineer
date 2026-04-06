# Module 8: The Interview Gauntlet 🔥

> **Goal:** Simulate a real interview day. This module throws 6 back-to-back rounds at you — algorithms, data engineering, SQL, system design, rapid fire, and a closing strategy session. Treat each screen as a timed round. If you can get through this module without peeking, you're ready.

---

## Screen 1: Algorithm Challenges

Six medium-difficulty problems that cover the patterns from Modules 1–5. For each: solve it in under 15 minutes, then check.

---

### Problem 1: Product of Array Except Self

**Problem:** Given an integer array `nums`, return an array `result` where `result[i]` equals the product of all elements in `nums` except `nums[i]`. You must solve it in O(n) time **without using division**.

**Approach:** Build a left-product prefix and a right-product suffix. For each index `i`, the answer is `left[i] * right[i]`. We can do this in two passes using a single output array to achieve O(1) extra space.

```python
def product_except_self(nums: list[int]) -> list[int]:
    """
    Return products of all elements except self.

    Time:  O(n) — two linear passes.
    Space: O(1) — output array doesn't count as extra space.
    """
    n = len(nums)
    result = [1] * n

    # Left pass: result[i] = product of nums[0..i-1]
    left = 1
    for i in range(n):
        result[i] = left
        left *= nums[i]

    # Right pass: multiply by product of nums[i+1..n-1]
    right = 1
    for i in range(n - 1, -1, -1):
        result[i] *= right
        right *= nums[i]

    return result
```

| | Value |
|---|---|
| Time | O(n) |
| Space | O(1) extra |

**Common follow-ups:**
- *"What if the array contains zeros?"* — The algorithm handles it naturally. One zero means only that index has a non-zero product. Two+ zeros means all products are zero.
- *"Can you do it with division?"* — Yes: total product ÷ `nums[i]`, but it fails with zeros and the problem forbids it. Mention both approaches to show range.

> **Interview Tip:** This problem tests prefix/suffix thinking. If you've drilled the prefix-sumrn from Module 1, this is the multiplicative analog. Interviewers love seeing you connect patterns across problem types.

---

### Problem 2: Merge Intervals

**Problem:** Given a list of intervals `[start, end]`, merge all overlapping intervals and return the non-overlapping result.

**Approach:** Sort by start time. Iterate through intervals: if the current interval overlaps with the last merged one (i.e., `current.start <= last_merged.end`), extend the end. Otherwise, start a new merged interval.

```python
def merge(intervals: list[list[int]]) -> list[list[int]]:
    """
    Merge overlapping intervals.

    Time:  O(n log n) — dominated by the sort.
    Space: O(n) — output list in worst case (no overlaps).
    """
    intervals.sort(key=lambda x: x[0])
    merged: list[list[int]] = []

    for start, end in intervals:
        if merged and start <= merged[-1][1]:
            merged[-1][1] = max(merged[-1][1], end)  # extend
        else:
            merged.append([start, end])

    return merged
```

| | Value |
|---|---|
| Time | O(n log n) |
| Space | O(n) |

**Common follow-ups:**
- *"Insert a new interval into a sorted non-overlapping list."* — Binary search for position, then merge neighbors.
- *"What if intervals are streaming?"* — Use a min-heap or balanced BST to maintain sorted order.

> **Interview Tip:** The sort-then-greedy pattern appears in scheduling, meeting rooms, and calendar problems. If you see intervals, sort first and ask questions later.

---

### Problem 3: Binary Tree Right Side View

**Problem:** Given the root of a binary tree, return the values of the nodes you can see when looking at the tree from the right side (i.e., the last node at each level).

**Approach:** BFS level-order traversal. At each level, record the last node's value. Alternatively, DFS with right-child-first traversal, recording the first node seen at each new depth.

```python
from collections import deque


def right_side_view(root) -> list[int]:
    """
    Return rightmost node values at each tree level.

    Time:  O(n) — visit every node once.
    Space: O(w) — where w is maximum width of the tree.
    """
    if not root:
        return []

    result: list[int] = []
    queue: deque = deque([root])

    while queue:
        level_size = len(queue)
        for i in range(level_size):
            node = queue.popleft()
            if i == level_size - 1:  # last node in this level
                result.append(node.val)
            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)

    return result
```

| | Value |
|---|---|
| Time | O(n) |
| Space | O(w) — tree width |

**Common follow-ups:**
- *"Left side view?"* — Same BFS, but take `i == 0` instead of `i == level_size - 1`.
- *"What about a zigzag level-order traversal?"* — Alternate appending direction at each level.

> **Interview Tip:** BFS level-order is the Swiss Army knife of tree problems. Any question that mentions "level," "depth," or "layer" is almost certainly BFS. Use `deque` for O(1) popleft — never `list.pop(0)` which is O(n).

---

### Problem 4: Word Break

**Problem:** Given a string `s` and a dictionary of words `wordDict`, return `True` if `s` can be segmented into a space-separated sequence of dictionary words.

**Approach:** 1-D DP. `dp[i]` is `True` if `s[0:i]` can be segmented. For each position `i`, check all possible last-word boundaries: if `dp[j]` is `True` and `s[j:i]` is in the dictionary, then `dp[i] = True`.

```python
def word_break(s: str, word_dict: list[str]) -> bool:
    """
    Return True if s can be segmented into dictionary words.

    Time:  O(n² × k) — n positions × n lookups × avg word length for slicing.
    Space: O(n + m) — dp array + word set.
    """
    word_set = set(word_dict)  # O(1) lookup instead of O(m) list search
    n = len(s)
    dp = [False] * (n + 1)
    dp[0] = True  # empty string is always valid

    for i in range(1, n + 1):
        for j in range(i):
            if dp[j] and s[j:i] in word_set:
                dp[i] = True
                break  # no need to check other j values

    return dp[n]
```

| | Value |
|---|---|
| Time | O(n²) average with early break |
| Space | O(n) |

**Common follow-ups:**
- *"Return all possible segmentations."* — Backtracking with memoization (Word Break II).
- *"What if the dictionary is huge?"* — Use a Trie for prefix lookups instead of a set.

> **Interview Tip:** The `break` after finding a valid segmentation is a key optimization that many candidates forget. It cuts average runtime significantly on strings with many valid split points.

---

### Problem 5: Clone Graph

**Problem:** Given a reference to a node in a connected undirected graph, return a deep copy of the entire graph. Each node has a `val` and a list of `neighbors`.

**Approach:** BFS or DFS with a hashmap that maps original nodes to their clones. When visiting a neighbor: if it's already cloned (in the map), reuse the clone; otherwise, create it and continue traversal.

```python
from collections import deque


class Node:
    def __init__(self, val: int = 0, neighbors: list["Node"] | None = None):
        self.val = val
        self.neighbors = neighbors if neighbors is not None else []


def clone_graph(node: Node | None) -> Node | None:
    """
    Deep-copy an undirected graph using BFS.

    Time:  O(V + E) — visit every node and edge once.
    Space: O(V) — hashmap stores one clone per node.
    """
    if not node:
        return None

    clones: dict[Node, Node] = {node: Node(node.val)}
    queue: deque[Node] = deque([node])

    while queue:
        current = queue.popleft()
        for neighbor in current.neighbors:
            if neighbor not in clones:
                clones[neighbor] = Node(neighbor.val)
                queue.append(neighbor)
            clones[current].neighbors.append(clones[neighbor])

    return clones[node]
```

| | Value |
|---|---|
| Time | O(V + E) |
| Space | O(V) |

**Common follow-ups:**
- *"What about a directed graph with cycles?"* — Same approach; the hashmap prevents infinite loops.
- *"Can you do it recursively?"* — Yes, DFS with the same hashmap. But watch the stack depth for large graphs.

> **Interview Tip:** Graph cloning tests two things: (1) can you traverse a graph, and (2) do you understand reference vs. value semantics. The hashmap is the key insight — it serves as both a "visited" set and a clone registry.

---

### Problem 6: Decode Ways

**Problem:** A message containing letters `A-Z` is encoded as numbers `1-26`. Given a string `s` of digits, return the number of ways to decode it. For example, `"226"` can be decoded as `"BZ"` (2, 26), `"VF"` (22, 6), or `"BBF"` (2, 2, 6) → answer is 3.

**Approach:** Linear DP, similar to Climbing Stairs. `dp[i]` = number of ways to decode `s[0:i]`. At each position, check: (1) can the single digit `s[i-1]` form a valid letter (1–9)? (2) can the two digits `s[i-2:i]` form a valid letter (10–26)?

```python
def num_decodings(s: str) -> int:
    """
    Count the number of ways to decode a digit string.

    Time:  O(n) — single pass.
    Space: O(1) — only two variables.
    """
    if not s or s[0] == "0":
        return 0

    # prev2 = dp[i-2], prev1 = dp[i-1]
    prev2, prev1 = 1, 1

    for i in range(1, len(s)):
        current = 0

        # Single digit: s[i] must be '1'-'9'
        if s[i] != "0":
            current += prev1

        # Two digits: s[i-1:i+1] must be '10'-'26'
        two_digit = int(s[i - 1 : i + 1])
        if 10 <= two_digit <= 26:
            current += prev2

        prev2, prev1 = prev1, current

    return prev1
```

| | Value |
|---|---|
| Time | O(n) |
| Space | O(1) |

**Common follow-ups:**
- *"What if `*` can represent 1–9?"* — Decode Ways II. The DP recurrence expands to handle wildcard cases with multiplication.
- *"What about leading zeros?"* — `"06"` is invalid because `06` isn't a valid encoding. The check `s[i] != '0'` handles this.

> **Interview Tip:** This is Climbing Stairs in disguise — but the "zero" handling is the trap. Half of candidates fail on inputs like `"10"`, `"20"`, or `"30"`. Walk through these edge cases explicitly before submitting.

---

## Screen 2: Data Engineering Coding Challenges

Six Pandas/PySpark problems. Show both implementations for each.

---

### Problem 1: Daily Active Users

**Problem:** Given an `events` table with `user_id`, `event_type`, and `event_time`, compute the Daily Active Users (DAU) — the count of **distinct** users per day.

```python
# ── Pandas ──
df["date"] = df["event_time"].dt.date
dau = df.groupby("date")["user_id"].nunique().reset_index(name="dau")
```

```python
# ── PySpark ──
from pyspark.sql.functions import to_date, countDistinct

dau = (
    df.withColumn("date", to_date("event_time"))
      .groupBy("date")
      .agg(countDistinct("user_id").alias("dau"))
      .orderBy("date")
)
```

> **Interview Tip:** Always use `nunique()` / `countDistinct()` — not `count()`. A single user generating 50 events should count as 1 active user, not 50.

---

### Problem 2: Revenue Attribution (First Touch)

**Problem:** Given `events` (user_id, event_type, channel, event_time) and `purchases` (user_id, amount, purchase_time), attribute each purchase's revenue to the user's **first** marketing channel.

```python
# ── Pandas ──
first_touch = (
    events.sort_values("event_time")
          .drop_duplicates(subset=["user_id"], keep="first")[["user_id", "channel"]]
)
attributed = purchases.merge(first_touch, on="user_id", how="left")
revenue_by_channel = attributed.groupby("channel")["amount"].sum().reset_index()
```

```python
# ── PySpark ──
from pyspark.sql.window import Window
from pyspark.sql.functions import row_number, col, desc, sum as _sum

w = Window.partitionBy("user_id").orderBy("event_time")
first_touch = (
    events.withColumn("rn", row_number().over(w))
          .filter(col("rn") == 1)
          .select("user_id", "channel")
)
attributed = purchases.join(first_touch, on="user_id", how="left")
revenue_by_channel = attributed.groupBy("channel").agg(_sum("amount").alias("revenue"))
```

> **Interview Tip:** Clarify the attribution model — first-touch, last-touch, or multi-touch. Each has a different window function strategy. First-touch = `ORDER BY event_time ASC`, last-touch = `DESC`.

---

### Problem 3: Data Quality Report

**Problem:** Given a DataFrame, produce a data quality summary: for each column, report the null count, null percentage, distinct count, and duplicate row count.

```python
# ── Pandas ──
def data_quality_report(df: pd.DataFrame) -> pd.DataFrame:
    report = pd.DataFrame({
        "column": df.columns,
        "null_count": df.isnull().sum().values,
        "null_pct": (df.isnull().mean() * 100).round(2).values,
        "distinct_count": df.nunique().values,
        "dtype": df.dtypes.astype(str).values,
    })
    report.loc[0, "total_rows"] = len(df)
    report.loc[0, "duplicate_rows"] = df.duplicated().sum()
    return report
```

```python
# ── PySpark ──
from pyspark.sql.functions import col, count, countDistinct, sum as _sum, lit, round as _round

total = df.count()
exprs = []
for c in df.columns:
    exprs.extend([
        _sum(col(c).isNull().cast("int")).alias(f"{c}_nulls"),
        countDistinct(col(c)).alias(f"{c}_distinct"),
    ])

stats = df.agg(*exprs).collect()[0]

rows = []
for c in df.columns:
    null_ct = stats[f"{c}_nulls"]
    rows.append({
        "column": c,
        "null_count": null_ct,
        "null_pct": round(100.0 * null_ct / total, 2),
        "distinct_count": stats[f"{c}_distinct"],
    })

quality_df = spark.createDataFrame(rows)
```

> **Interview Tip:** In production, use a framework like Great Expectations or Deequ rather than hand-rolling quality checks. But knowing how to build it from scratch shows you understand what those tools do under the hood.

---

### Problem 4: SCD Type 2 — Point-in-Time Query

**Problem:** Given an SCD Type 2 table with `effective_date` and `end_date` columns, retrieve the state of every record as of a specific point in time.

```python
# ── Pandas ──
as_of = pd.Timestamp("2025-06-15")
snapshot = scd_df[
    (scd_df["effective_date"] <= as_of) & (scd_df["end_date"] > as_of)
]
```

```python
# ── PySpark ──
from pyspark.sql.functions import lit

as_of = lit("2025-06-15").cast("date")
snapshot = scd_df.filter(
    (col("effective_date") <= as_of) & (col("end_date") > as_of)
)
```

> **Interview Tip:** The convention is `effective_date` inclusive, `end_date` exclusive (half-open interval). Active records typically have `end_date = '9999-12-31'`. Always clarify the boundary convention before coding.

---

### Problem 5: Event Funnel with Drop-Off Rates

**Problem:** Given user events with steps (view → cart → checkout → purchase), compute the user count and step-over-step drop-off rate at each stage.

```python
# ── Pandas ──
funnel_order = ["view", "cart", "checkout", "purchase"]
funnel = (
    df[df["step"].isin(funnel_order)]
    .groupby("step")["user_id"]
    .nunique()
    .reindex(funnel_order)
    .reset_index(name="users")
)
funnel["drop_off_pct"] = (
    (1 - funnel["users"] / funnel["users"].shift(1)) * 100
).round(1)
```

```python
# ── PySpark ──
from pyspark.sql.functions import countDistinct, lag
from pyspark.sql.window import Window

funnel_order = ["view", "cart", "checkout", "purchase"]
funnel = (
    df.filter(col("step").isin(funnel_order))
      .groupBy("step")
      .agg(countDistinct("user_id").alias("users"))
)

# Add ordering for lag calculation
from pyspark.sql.functions import when, coalesce

ordering = {s: i for i, s in enumerate(funnel_order)}
order_expr = coalesce(*[when(col("step") == s, lit(i)) for s, i in ordering.items()])

w = Window.orderBy("step_order")
funnel = (
    funnel.withColumn("step_order", order_expr)
          .withColumn("prev_users", lag("users").over(w))
          .withColumn("drop_off_pct",
                      _round((1 - col("users") / col("prev_users")) * 100, 1))
          .orderBy("step_order")
)
```

> **Interview Tip:** A strict funnel requires sequential event ordering per user (cart only counts if view happened first). Mention this distinction even if the problem doesn't require it — it shows analytical depth.

---

### Problem 6: Rolling 7-Day Average

**Problem:** Given daily metrics with `date` and `value`, compute a trailing 7-day rolling average.

```python
# ── Pandas ──
df = df.sort_values("date").set_index("date")
df["rolling_7d_avg"] = df["value"].rolling(window=7, min_periods=1).mean()
```

```python
# ── PySpark ──
from pyspark.sql.functions import avg as _avg
from pyspark.sql.window import Window

w = Window.orderBy("date").rowsBetween(-6, 0)
result = df.withColumn("rolling_7d_avg", _avg("value").over(w))
```

> **Interview Tip:** Note that `rowsBetween(-6, 0)` gives the current row plus the 6 preceding rows = 7 rows total. A common off-by-one error is using `-7, 0` which gives 8 rows. Always count on your fingers.

---

## Screen 3: SQL Challenges

Six medium-hard SQL problems. Write each in under 12 minutes.

---

### Problem 1: Consecutive Numbers

**Problem:** Find all numbers that appear at least three times consecutively in a `logs` table with columns `id` (auto-increment) and `num`.

```sql
SELECT DISTINCT l1.num AS consecutive_num
FROM logs l1
JOIN logs l2 ON l1.id = l2.id - 1
JOIN logs l3 ON l2.id = l3.id - 1
WHERE l1.num = l2.num
  AND l2.num = l3.num;
```

**Alternative (window function — more robust if IDs have gaps):**

```sql
WITH grouped AS (
    SELECT num,
           id - ROW_NUMBER() OVER (PARTITION BY num ORDER BY id) AS grp
    FROM logs
)
SELECT DISTINCT num AS consecutive_num
FROM grouped
GROUP BY num, grp
HAVING COUNT(*) >= 3;
```

> **Interview Tip:** The self-join approach assumes contiguous IDs. Always ask: *"Are the IDs guaranteed sequential with no gaps?"* If not, use the gap-and-island window approach.

---

### Problem 2: Department Top 3 Salaries

```sql
WITH ranked AS (
    SELECT
        d.name AS department,
        e.name AS employee,
        e.salary,
        DENSE_RANK() OVER (
            PARTITION BY e.department_id ORDER BY e.salary DESC
        ) AS dr
    FROM employees e
    JOIN departments d ON e.department_id = d.id
)
SELECT department, employee, salary
FROM ranked
WHERE dr <= 3;
```

> **Interview Tip:** `DENSE_RANK` keeps ties and doesn't skip ranks. If salary 100k appears twice, both get rank 1, and the next distinct salary gets rank 2. Clarify whether the problem wants top 3 **people** (use `ROW_NUMBER`) or top 3 **salary levels** (use `DENSE_RANK`).

---

### Problem 3: Trips and Users Cancellation Rate

**Problem:** Find the cancellation rate of requests with unbanned users for each day between `'2025-10-01'` and `'2025-10-03'`.

```sql
SELECT
    t.request_at AS day,
    ROUND(
        SUM(CASE WHEN t.status LIKE 'cancelled%' THEN 1.0 ELSE 0 END)
        / COUNT(*),
        2
    ) AS cancellation_rate
FROM trips t
JOIN users c ON t.client_id = c.users_id AND c.banned = 'No'
JOIN users d ON t.driver_id = d.users_id AND d.banned = 'No'
WHERE t.request_at BETWEEN '2025-10-01' AND '2025-10-03'
GROUP BY t.request_at
ORDER BY t.request_at;
```

> **Interview Tip:** The double-join on `users` (once for client, once for driver) is the key insight. Both sides must be unbanned. Using `LIKE 'cancelled%'` handles both `cancelled_by_driver` and `cancelled_by_client`.

---

### Problem 4: Exchange Seats

**Problem:** Swap the seat IDs of every two consecutive students. If the last student is odd-numbered, they stay in place.

```sql
SELECT
    CASE
        WHEN id % 2 = 1 AND id = (SELECT MAX(id) FROM seat) THEN id
        WHEN id % 2 = 1 THEN id + 1
        ELSE id - 1
    END AS id,
    student
FROM seat
ORDER BY id;
```

> **Interview Tip:** The edge case is the last row when the total count is odd. Without the `id = MAX(id)` check, the last student swaps with a nonexistent row. Handling this edge case cleanly is what separates pass from fail.

---

### Problem 5: Human Traffic of Stadium

**Problem:** Find rows where 3 or more consecutive rows each have `people >= 100`. Return all rows in those streaks.

```sql
WITH filtered AS (
    SELECT *,
           id - ROW_NUMBER() OVER (ORDER BY id) AS grp
    FROM stadium
    WHERE people >= 100
),
streaks AS (
    SELECT grp
    FROM filtered
    GROUP BY grp
    HAVING COUNT(*) >= 3
)
SELECT f.id, f.visit_date, f.people
FROM filtered f
JOIN streaks s ON f.grp = s.grp
ORDER BY f.id;
```

> **Interview Tip:** This is the gap-and-island pattern again. Filtering first (`people >= 100`), then computing the island group, then filtering islands of size ≥ 3. Master this three-step flow and you can solve any "consecutive rows matching condition" problem.

---

### Problem 6: Median Employee Salary

**Problem:** Find the median salary per department. If even number of employees, return both middle values.

```sql
WITH counted AS (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary) AS rn,
           COUNT(*) OVER (PARTITION BY department) AS cnt
    FROM employees
)
SELECT department, salary AS median_salary
FROM counted
WHERE rn IN (FLOOR((cnt + 1) / 2.0), CEIL((cnt + 1) / 2.0));
```

> **Interview Tip:** `PERCENTILE_CONT(0.5)` is cleaner in engines that support it (PostgreSQL, BigQuery). But the `ROW_NUMBER` approach works everywhere and shows deeper understanding. Know both.

---

## Screen 4: System Design Coding

Design it, then build it. Working code, not just diagrams.

---

### Problem 1: LRU Cache

**Design:** Doubly linked list for O(1) move-to-front + hashmap for O(1) key lookup. Most recently used at the head; evict from the tail.

```python
class Node:
    __slots__ = ("key", "val", "prev", "next")

    def __init__(self, key: int = 0, val: int = 0):
        self.key = key
        self.val = val
        self.prev: Node | None = None
        self.next: Node | None = None


class LRUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache: dict[int, Node] = {}
        # Sentinel nodes eliminate edge-case checks
        self.head = Node()
        self.tail = Node()
        self.head.next = self.tail
        self.tail.prev = self.head

    def _remove(self, node: Node) -> None:
        node.prev.next = node.next
        node.next.prev = node.prev

    def _add_to_front(self, node: Node) -> None:
        node.next = self.head.next
        node.prev = self.head
        self.head.next.prev = node
        self.head.next = node

    def get(self, key: int) -> int:
        if key not in self.cache:
            return -1
        node = self.cache[key]
        self._remove(node)
        self._add_to_front(node)
        return node.val

    def put(self, key: int, value: int) -> None:
        if key in self.cache:
            self._remove(self.cache[key])
        node = Node(key, value)
        self._add_to_front(node)
        self.cache[key] = node

        if len(self.cache) > self.capacity:
            lru = self.tail.prev
            self._remove(lru)
            del self.cache[lru.key]
```

> **Interview Tip:** The sentinel nodes (`head`, `tail`) are the secret weapon. Without them, every `_remove` and `_add` method needs null checks for empty lists and boundary conditions. Sentinels eliminate four `if` statements.

---

### Problem 2: Rate Limiter (Sliding Window Counter)

```python
import time
from collections import defaultdict


class RateLimiter:
    """Sliding window counter: allows `max_requests` per `window_seconds`."""

    def __init__(self, max_requests: int, window_seconds: int):
        self.max_requests = max_requests
        self.window = window_seconds
        self.requests: dict[str, list[float]] = defaultdict(list)

    def allow(self, client_id: str) -> bool:
        now = time.time()
        timestamps = self.requests[client_id]

        # Evict expired timestamps
        while timestamps and timestamps[0] <= now - self.window:
            timestamps.pop(0)

        if len(timestamps) < self.max_requests:
            timestamps.append(now)
            return True
        return False
```

> **Interview Tip:** In production, use a Redis sorted set (`ZADD` + `ZRANGEBYSCORE` + `ZCARD`) for distributed rate limiting. This in-memory version is fine for interview coding rounds. Mention the Redis approach as a follow-up.

---

### Problem 3: Event Processor with Dedup and Late Arrivals

```python
from collections import defaultdict
import heapq


class EventProcessor:
    """Process events in order, handling duplicates and late arrivals."""

    def __init__(self, max_lateness_seconds: int = 300):
        self.seen_ids: set[str] = set()
        self.buffer: list[tuple[float, str, dict]] = []  # min-heap by timestamp
        self.max_lateness = max_lateness_seconds
        self.watermark: float = 0.0

    def ingest(self, event_id: str, timestamp: float, payload: dict) -> None:
        if event_id in self.seen_ids:
            return  # dedup
        if timestamp < self.watermark - self.max_lateness:
            return  # too late, drop
        self.seen_ids.add(event_id)
        heapq.heappush(self.buffer, (timestamp, event_id, payload))

    def flush(self, current_time: float) -> list[dict]:
        """Emit all events with timestamp <= current_time - max_lateness."""
        self.watermark = current_time - self.max_lateness
        results: list[dict] = []
        while self.buffer and self.buffer[0][0] <= self.watermark:
            ts, eid, payload = heapq.heappop(self.buffer)
            results.append({"event_id": eid, "timestamp": ts, **payload})
        return results
```

> **Interview Tip:** This mirrors how Apache Flink's watermark system works. The `watermark` concept — "I believe all events before this time have arrived" — is a fundamental streaming concept. Explaining it shows system design maturity.

---

### Problem 4: Min Stack

```python
class MinStack:
    """Stack that supports push, pop, top, and getMin in O(1)."""

    def __init__(self):
        self.stack: list[int] = []
        self.min_stack: list[int] = []  # parallel stack tracking minimums

    def push(self, val: int) -> None:
        self.stack.append(val)
        min_val = min(val, self.min_stack[-1]) if self.min_stack else val
        self.min_stack.append(min_val)

    def pop(self) -> None:
        self.stack.pop()
        self.min_stack.pop()

    def top(self) -> int:
        return self.stack[-1]

    def get_min(self) -> int:
        return self.min_stack[-1]
```

> **Interview Tip:** The trick is maintaining a parallel `min_stack` where `min_stack[i]` stores the minimum of `stack[0:i+1]`. Both stacks push and pop together, so they're always in sync. O(1) for every operation, O(n) extra space.

---

### Problem 5: Hit Counter

```python
from collections import deque


class HitCounter:
    """Count hits in the last 300 seconds (5 minutes)."""

    def __init__(self):
        self.hits: deque[int] = deque()

    def hit(self, timestamp: int) -> None:
        self.hits.append(timestamp)

    def get_hits(self, timestamp: int) -> int:
        # Evict hits older than 5 minutes
        while self.hits and self.hits[0] <= timestamp - 300:
            self.hits.popleft()
        return len(self.hits)
```

**For high-throughput scenarios** (millions of hits per second), use a circular buffer:

```python
class HitCounterScalable:
    """O(1) hit counter using a circular buffer of 300 buckets."""

    def __init__(self):
        self.times = [0] * 300
        self.counts = [0] * 300

    def hit(self, timestamp: int) -> None:
        idx = timestamp % 300
        if self.times[idx] != timestamp:
            self.times[idx] = timestamp
            self.counts[idx] = 1
        else:
            self.counts[idx] += 1

    def get_hits(self, timestamp: int) -> int:
        total = 0
        for i in range(300):
            if timestamp - self.times[i] < 300:
                total += self.counts[i]
        return total
```

> **Interview Tip:** Start with the deque approach (simple, correct), then optimize to the circular buffer when the interviewer asks about scale. This show-then-optimize progression is exactly what interviewers want to see.

---

## Screen 5: Rapid Fire — 20 Q&A

Answer each in under 30 seconds. No code — just crisp explanations.

---

### Algorithm Questions

**Q1:** What is the time complexity of binary search?
→ **A:** O(log n). Each comparison halves the search space.

**Q2:** When would you use a heap over a sorted array?
→ **A:** When you need efficient insert + extract-min/max. Heap gives O(log n) for both; sorted array is O(n) insert, O(1) extract.

**Q3:** What data structure gives O(1) insert, O(1) delete, and O(1) random access?
→ **A:** An array-backed list with a hashmap (index lookup). Swap the element with the last element, then pop. This is the `RandomizedSet` pattern.

**Q4:** What's the difference between BFS and DFS for shortest path?
→ **A:** BFS finds shortest path in unweighted graphs (it explores level by level). DFS does not guarantee shortest path — it just finds *a* path.

**Q5:** Why is quicksort O(n²) worst-case but O(n log n) average?
→ **A:** Worst case occurs when the pivot is always the smallest/largest element (sorted input + bad pivot). Random pivot selection makes this astronomically unlikely in practice.

---

### Python Questions

**Q6:** What is the GIL and why does it matter?
→ **A:** The Global Interpreter Lock prevents multiple threads from executing Python bytecode simultaneously. CPU-bound work doesn't parallelize with threads — use `multiprocessing` or `concurrent.futures.ProcessPoolExecutor` instead.

**Q7:** Generator vs. list comprehension — when to use which?
→ **A:** Generator for large/infinite sequences (lazy, O(1) memory). List comprehension when you need random access or multiple passes over the data.

**Q8:** What's the difference between `list` and `tuple`?
→ **A:** Lists are mutable, tuples are immutable. Tuples are hashable (can be dict keys), slightly faster, and signal "this data shouldn't change."

**Q9:** What does `copy.deepcopy()` do that `copy.copy()` doesn't?
→ **A:** `copy()` creates a new outer object but shares references to nested objects (shallow). `deepcopy()` recursively copies all nested objects — fully independent clone.

**Q10:** How does Python's `dict` achieve O(1) average lookup?
→ **A:** Hash table with open addressing. Keys are hashed to bucket indices; collisions are resolved by probing. Amortized O(1) because the table resizes when the load factor exceeds ~2/3.

---

### SQL Questions

**Q11:** `HAVING` vs. `WHERE` — what's the difference?
→ **A:** `WHERE` filters rows *before* grouping. `HAVING` filters groups *after* aggregation. You can't use aggregate functions in `WHERE`.

**Q12:** `UNION` vs. `UNION ALL`?
→ **A:** `UNION` deduplicates results (slower — requires sort/hash). `UNION ALL` keeps all rows including duplicates (faster). Default to `UNION ALL` unless you specifically need dedup.

**Q13:** What types of indexes exist and when do you use each?
→ **A:** B-tree (default, range queries), hash (exact lookups), GIN/GiST (full-text, JSON, arrays), bitmap (low-cardinality columns in OLAP). Composite indexes follow the leftmost-prefix rule.

**Q14:** What's a correlated subquery?
→ **A:** A subquery that references a column from the outer query. It re-executes for each row of the outer query, making it O(n × m) unless optimized by the engine.

**Q15:** Explain `ROWS` vs. `RANGE` in window frames.
→ **A:** `ROWS` counts physical rows. `RANGE` groups by logical value — rows with the same `ORDER BY` value are treated as one unit. Use `ROWS` for deterministic running totals.

---

### Data Engineering Questions

**Q16:** Parquet vs. CSV — why Parquet for analytics?
→ **A:** Columnar storage (read only needed columns), built-in compression (snappy, zstd), schema enforcement, predicate pushdown, and splittable for parallel reads. CSV has none of these.

**Q17:** What is exactly-once semantics and why is it hard?
→ **A:** Every message is processed exactly one time — no duplicates, no losses. It's hard because network failures can cause retries (duplicates) or dropped messages. Achieved via idempotent writes + transactional commits (e.g., Kafka transactions + atomic sink writes).

**Q18:** What does "idempotent" mean in a pipeline context?
→ **A:** Running the pipeline multiple times with the same input produces the same output. Achieved through `MERGE`/upsert, partition-level overwrites, or dedup-on-write patterns. Essential for safe re-runs after failures.

**Q19:** What's a backfill strategy?
→ **A:** Re-processing historical data after a pipeline change. Key considerations: partition-based execution (process one day at a time), idempotency (overwrite not append), rate limiting (don't overwhelm upstream systems), and validation (compare counts before/after).

**Q20:** What is schema evolution and how do you handle it?
→ **A:** When source schemas change (new columns, type changes, dropped fields). Handle with: schema registries (Avro/Protobuf), `mergeSchema` option in Spark, backward/forward compatibility rules, and default values for new fields.

---

## Screen 6: Key Takeaways & Interview Day Tips

---

### Top 10 Patterns to Memorize

These patterns cover approximately 80% of all coding interview questions. Know them cold:

| # | Pattern | Signal Phrase | Module |
|---|---|---|---|
| 1 | **Two Pointers** | "Sorted array," "pair sum" | 1 |
| 2 | **Sliding Window** | "Contiguous subarray," "longest substring" | 1 |
| 3 | **Prefix Sum + HashMap** | "Subarray sum equals K" | 1 |
| 4 | **HashMap / Set Lookup** | "Find complement," "two sum" | 2 |
| 5 | **BFS Level-Order** | "Level," "shortest path," "layer-by-layer" | 3 |
| 6 | **DFS + Backtracking** | "All paths," "permutations," "subsets" | 3 |
| 7 | **Linear DP** | "Min/max ways," "climbing stairs" | 4 |
| 8 | **Gap and Island** | "Consecutive days," "streaks" | 7 |
| 9 | **Window Functions** | "Rank," "running total," "lag/lead" | 7 |
| 10 | **Sessionization** | "Group by time gap," "sessions" | 6 |

---

### Time Management During Coding Interviews

A typical 45-minute coding round breaks down like this:

| Phase | Time | What to Do |
|---|---|---|
| **Clarify** | 3–5 min | Restate the problem, confirm inputs/outputs, ask about edge cases |
| **Approach** | 3–5 min | Explain your strategy out loud, discuss time/space complexity |
| **Code** | 15–20 min | Write clean, well-named code. Narrate as you go |
| **Test** | 5–7 min | Walk through 1–2 examples by hand, check edge cases |
| **Optimize** | 5 min | Discuss alternative approaches, trade-offs |
| **Questions** | 3–5 min | Ask the interviewer thoughtful questions about the team/role |

**Key rule:** Never start coding before you've explained your approach and gotten a nod. The approach phase is where most interviews are won or lost.

---

### How to Communicate While Coding

1. **Think out loud.** The interviewer can't read your mind. Say: *"I'm thinking of using a hashmap here because I need O(1) lookups for the complement."*

2. **Name your variables well.** `left`, `right`, `window_sum` beat `i`, `j`, `s`. The interviewer reads your code in real time.

3. **Announce edge cases before they bite you.** *"I need to handle the empty array case here."* This shows defensive thinking.

4. **Say when you're stuck.** *"I'm considering two approaches — a brute-force O(n²) and a hashmap O(n). Let me think about whether the hashmap works here..."* Silence is the enemy.

5. **Ask for a hint gracefully.** *"I'm stuck on how to handle the overlapping case — could you point me in the right direction?"* Interviewers expect this and appreciate self-awareness.

---

### Common Mistakes and How to Avoid Them

| Mistake | Why It Happens | Fix |
|---|---|---|
| Jumping straight into code | Nervous energy | Force yourself: "Let me think for 30 seconds" |
| Off-by-one errors | Unclear loop boundaries | Decide convention: `[left, right]` inclusive or `[left, right)` half-open |
| Not handling empty inputs | Assumed valid input | Start every function with: `if not nums: return ...` |
| Using `list.pop(0)` | Forgot it's O(n) | Use `collections.deque` for O(1) popleft |
| Building strings with `+=` | Python strings are immutable | Use `"".join(parts)` — one allocation |
| Forgetting `{0: 1}` in prefix sum | Initialization oversight | Memorize it: "empty prefix = sum 0, seen once" |
| `NOT IN` with NULLs in SQL | Subtle SQL behavior | Always use `NOT EXISTS` instead |
| Integer division in SQL | `1/3 = 0` in many engines | Always cast: `1.0 / 3` |

---

### The Night Before — Checklist

- [ ] Review your **top 10 patterns** (skim solutions, don't re-solve)
- [ ] Re-read your **resume** — you'll be asked about every bullet point
- [ ] Prepare **3 questions** for each interviewer (look them up on LinkedIn)
- [ ] Test your **IDE/environment** — screen sharing, microphone, camera
- [ ] Set out **water and a notepad** next to your desk
- [ ] **Sleep 7+ hours.** No late-night cramming. Seriously.
- [ ] Set **two alarms** — 60 minutes and 30 minutes before the interview

---

### What NOT to Do the Night Before

- ❌ Don't try to learn a new topic. It's too late and it'll shake your confidence.
- ❌ Don't solve hard problems. Struggle breeds doubt. Solve two easy ones for a confidence boost.
- ❌ Don't read horror stories about interview rejections. Mindset matters.
- ❌ Don't drink caffeine after 2 PM. Sleep quality is your most powerful performance enhancer.

---

### Confidence-Building Close

You've made it through 8 modules. You've studied arrays, hashmaps, trees, graphs, dynamic programming, sorting, heaps, data engineering, SQL, and system design. You've seen the patterns, written the code, and practiced the translations.

Here's what most candidates don't realize: **the bar is lower than you think.**

Interviewers aren't looking for perfection. They're looking for:

- **Structured thinking** — Can you break a problem into smaller pieces?
- **Communication** — Can you explain your reasoning while you code?
- **Pattern recognition** — Do you see that this is a sliding window problem?
- **Debugging ability** — When your code doesn't work, can you find the bug?
- **Growth mindset** — When you're stuck, do you keep pushing or shut down?

You don't need to solve every problem optimally. You need to show that you *think* like an engineer — methodically, clearly, and under pressure.

Every "no" gets you closer to the right "yes." Every interview — even the bad ones — makes you sharper. The person who gets the offer isn't the one who knows the most algorithms. It's the one who communicates the best and stays calm when the problem gets hard.

You've done the work. Now go show them what you've got.

**You're ready.** 🚀

---

*This concludes the DSA & Coding Interview Preparation Course. Go build something amazing.*
