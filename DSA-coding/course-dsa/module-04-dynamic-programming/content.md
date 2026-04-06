# Module 4: Dynamic Programming

> *"Those who cannot remember the past are condemned to repeat it."*
> — George Santayana (and every recursive function without memoization)

Dynamic Programming (DP) is the single most tested topic in coding interviews at
top-tier companies. It feels intimidating at first, but every DP problem follows
a small set of repeatable patterns. This module breaks them down with a
pattern-first approach so you can **recognize** and **solve** DP problems
systematically.

---

## 1. What Is Dynamic Programming?

Dynamic Programming is an **optimization technique** that solves complex problems
by breaking them into simpler, overlapping subproblems. Two properties must hold
for DP to apply:

### 1.1 Overlapping Subproblems

The same smaller problem is solved **many times** during recursion. Consider
computing `fib(5)` naïvely — `fib(3)` is computed twice, `fib(2)` three times.
DP stores (caches) these results so each subproblem is solved **exactly once**.

### 1.2 Optimal Substructure

An optimal solution to the whole problem can be **constructed from optimal
solutions to its subproblems**. If the shortest path from A→C goes through B,
then the A→B segment must itself be the shortest path from A→B.

### 1.3 The Recognition Test

Ask yourself these questions when you see a new problem:

| Question | If Yes → |
|---|---|
| Can I break this into smaller **identical** subproblems? | DP candidate |
| Does the problem ask for **min/max/count** of something? | Strong DP signal |
| Are there **choices** at each step (take/skip, left/right)? | DP with decision |
| Is brute-force **exponential** but subproblems overlap? | DP will optimize |
| Does order of operations not matter (only end state)? | DP over greedy |

> **Interview Tip:** When an interviewer says *"Can you optimize your brute-force
> solution?"* and your recursion tree has repeated nodes — that's your cue to
> reach for DP.

---

## 2. Memoization vs. Tabulation

Every DP problem can be solved two ways. Master **both** — interviewers may ask
you to convert between them. The difference is purely mechanical — the
underlying recurrence relation is identical in both approaches.

### 2.1 Top-Down (Memoization)

Start by writing the natural **recursive** solution — the one that directly
expresses the answer in terms of smaller versions of itself. Then add a
**cache** (dictionary or array) to store results of subproblems you have already
solved. The next time that subproblem is encountered, return the cached result
instead of recomputing it.

```python
from functools import lru_cache

@lru_cache(maxsize=None)
def fib(n: int) -> int:
    if n <= 1:
        return n
    return fib(n - 1) + fib(n - 2)
```

**Pros:** Intuitive, mirrors the recurrence relation directly, only solves
subproblems that are actually needed (lazy evaluation).

**Cons:** Recursion stack overhead, possible stack overflow for very large inputs
(`sys.setrecursionlimit` won't save you in production).

### 2.2 Bottom-Up (Tabulation)

Instead of letting recursion discover which subproblems are needed, **you**
decide the order. Build a table (array) iteratively, starting from the **base
cases** and filling in progressively larger subproblems until you reach the
final answer. There is no recursion and no call stack overhead.

```python
def fib(n: int) -> int:
    if n <= 1:
        return n
    dp = [0] * (n + 1)
    dp[1] = 1
    for i in range(2, n + 1):
        dp[i] = dp[i - 1] + dp[i - 2]
    return dp[n]
```

**Pros:** No recursion overhead, cache-friendly memory access, easier to
optimize space.

**Cons:** Must figure out the correct iteration order; may compute subproblems
you don't actually need.

### 2.3 Space Optimization

When `dp[i]` depends only on a **fixed window** of previous values, you can
reduce space from O(n) to O(1):

```python
def fib(n: int) -> int:
    if n <= 1:
        return n
    prev2, prev1 = 0, 1
    for _ in range(2, n + 1):
        prev2, prev1 = prev1, prev2 + prev1
    return prev1
```

> **Interview Tip:** Always mention the space-optimized version after presenting
> the full table. It shows depth of understanding and interviewers love it.

### 2.4 When to Use Which

| Situation | Prefer |
|---|---|
| Problem has a natural recursive structure | Top-down |
| You need to solve **all** subproblems anyway | Bottom-up |
| Deep recursion risk (n > 1000) | Bottom-up |
| Only a few subproblems are needed | Top-down |
| You want to optimize space | Bottom-up + rolling |

---

## 3. Fibonacci-Type Problems (Linear DP)

These are 1-D DP problems where `dp[i]` depends on a small number of previous
entries. They are the **easiest** DP pattern — nail these first.

---

### Problem 1: Climbing Stairs

**LeetCode 70**

#### Problem Statement

You are climbing a staircase with `n` steps. Each time you can climb **1 or 2**
steps. In how many **distinct ways** can you reach the top?

#### How to Recognize This Pattern

- Counting the **number of ways** to reach a goal.
- At each step, you have a **small fixed set of choices**.
- The result for step `i` depends only on previous steps.

#### Approach

This is literally Fibonacci in disguise. From step `i`, you could have arrived
from step `i-1` (took 1 step) or step `i-2` (took 2 steps).

#### State Definition

`dp[i]` = number of distinct ways to reach step `i`.

#### Recurrence Relation

```
dp[i] = dp[i - 1] + dp[i - 2]
```

#### Base Cases

```
dp[0] = 1  (one way to stand at the ground — do nothing)
dp[1] = 1  (one way to reach step 1 — take one step)
```

#### Code

```python
def climb_stairs(n: int) -> int:
    """Return the number of distinct ways to climb n stairs."""
    if n <= 2:
        return n

    prev2, prev1 = 1, 2
    for _ in range(3, n + 1):
        prev2, prev1 = prev1, prev2 + prev1
    return prev1
```

#### Complexity

| | Value |
|---|---|
| Time | O(n) |
| Space | O(1) — space-optimized |

> **Interview Tip:** If the interviewer changes it to "1, 2, or 3 steps," just
> extend: `dp[i] = dp[i-1] + dp[i-2] + dp[i-3]`. The pattern is the same.

---

### Problem 2: House Robber

**LeetCode 198**

#### Problem Statement

You are a robber planning to rob houses along a street. Each house has a certain
amount of money. **Adjacent houses have connected alarms** — if two adjacent
houses are robbed, the alarm triggers. Find the **maximum** amount you can rob
without triggering alarms.

#### How to Recognize This Pattern

- **Maximize** a value with a **constraint** (can't pick adjacent elements).
- At each element, you make a **binary decision**: take or skip.

#### Approach

At each house `i`, you have two choices:

1. **Rob house `i`:** You gain `nums[i]`, but you must skip house `i-1`. Best
   you can do = `nums[i] + dp[i - 2]`.
2. **Skip house `i`:** Your best remains `dp[i - 1]`.

Take the maximum of both choices.

#### State Definition

`dp[i]` = maximum money you can rob from the first `i` houses.

#### Recurrence Relation

```
dp[i] = max(dp[i - 1], dp[i - 2] + nums[i])
```

#### Base Cases

```
dp[0] = nums[0]
dp[1] = max(nums[0], nums[1])
```

#### Code

```python
def rob(nums: list[int]) -> int:
    """Return max money robable without robbing adjacent houses."""
    if not nums:
        return 0
    if len(nums) == 1:
        return nums[0]

    prev2, prev1 = nums[0], max(nums[0], nums[1])
    for i in range(2, len(nums)):
        prev2, prev1 = prev1, max(prev1, prev2 + nums[i])
    return prev1
```

#### Complexity

| | Value |
|---|---|
| Time | O(n) |
| Space | O(1) |

> **Interview Tip:** House Robber II (circular street) is the same idea — just
> run the algorithm twice: once excluding the first house, once excluding the
> last, and take the max.

---

### Problem 3: Coin Change

**LeetCode 322**

#### Problem Statement

Given an array `coins` of coin denominations and an integer `amount`, return the
**fewest number of coins** needed to make up that amount. If it's impossible,
return `-1`.

#### How to Recognize This Pattern

- **Minimize** the count of items to reach a **target sum**.
- **Unbounded choices** — you can use each coin denomination unlimited times.
- Classic "unbounded knapsack" variant.

#### Approach

For every amount from `1` to `amount`, try every coin. If the coin fits
(`coin <= current_amount`), see if using it leads to fewer total coins.

#### State Definition

`dp[a]` = minimum number of coins needed to make amount `a`.

#### Recurrence Relation

```
dp[a] = min(dp[a - coin] + 1) for each coin in coins, if a - coin >= 0
```

#### Base Cases

```
dp[0] = 0  (zero coins needed for amount 0)
dp[1..amount] = infinity  (not yet achievable)
```

#### Code

```python
def coin_change(coins: list[int], amount: int) -> int:
    """Return the fewest coins to make `amount`, or -1 if impossible."""
    dp = [float("inf")] * (amount + 1)
    dp[0] = 0

    for a in range(1, amount + 1):
        for coin in coins:
            if coin <= a and dp[a - coin] + 1 < dp[a]:
                dp[a] = dp[a - coin] + 1

    return dp[amount] if dp[amount] != float("inf") else -1
```

#### Complexity

| | Value |
|---|---|
| Time | O(amount × len(coins)) |
| Space | O(amount) |

> **Interview Tip:** If the problem asks for the **number of combinations** to
> make the amount (not minimum coins), it's a different DP — Coin Change II
> (LeetCode 518). Know both.

---

## 4. Grid Problems (2-D DP)

In grid DP, you fill a 2-D table where `dp[r][c]` represents the answer for the
sub-grid ending at row `r`, column `c`.

---

### Problem 4: Unique Paths

**LeetCode 62**

#### Problem Statement

A robot is on an `m × n` grid, starting at the **top-left corner**. It can only
move **right** or **down**. How many unique paths exist to reach the
**bottom-right corner**?

#### How to Recognize This Pattern

- Grid traversal with **restricted movement** (right/down only).
- Counting **number of ways** to reach a destination.

#### Approach

Each cell `(r, c)` can only be reached from `(r-1, c)` (came from above) or
`(r, c-1)` (came from the left). The total paths to `(r, c)` is the sum of
paths to those two predecessors.

#### State Definition

`dp[r][c]` = number of unique paths from `(0, 0)` to `(r, c)`.

#### Recurrence Relation

```
dp[r][c] = dp[r - 1][c] + dp[r][c - 1]
```

#### Base Cases

```
dp[0][c] = 1  for all c  (only one way along the top edge — keep going right)
dp[r][0] = 1  for all r  (only one way along the left edge — keep going down)
```

#### Code

```python
def unique_paths(m: int, n: int) -> int:
    """Return the number of unique paths in an m x n grid."""
    # Space-optimized: keep only one row at a time.
    row = [1] * n

    for r in range(1, m):
        for c in range(1, n):
            row[c] += row[c - 1]

    return row[-1]
```

#### Complexity

| | Value |
|---|---|
| Time | O(m × n) |
| Space | O(n) — space-optimized |

> **Interview Tip:** "Unique Paths II" adds obstacles. Just set `dp[r][c] = 0`
> for any obstacle cell. Same pattern, minor tweak.

---

### Minimum Path Sum (Bonus Pattern)

**LeetCode 64** — Given an `m × n` grid of non-negative numbers, find a path
from top-left to bottom-right that **minimizes the sum** of numbers along the
path. You can only move right or down.

This is nearly identical to Unique Paths, but instead of counting paths, you
pick the path with the smallest total:

```
dp[r][c] = grid[r][c] + min(dp[r-1][c], dp[r][c-1])
```

```python
def min_path_sum(grid: list[list[int]]) -> int:
    """Return the minimum path sum from top-left to bottom-right."""
    m, n = len(grid), len(grid[0])
    dp = [0] * n
    dp[0] = grid[0][0]

    # First row: can only come from the left.
    for c in range(1, n):
        dp[c] = dp[c - 1] + grid[0][c]

    for r in range(1, m):
        dp[0] += grid[r][0]  # First column: can only come from above.
        for c in range(1, n):
            dp[c] = grid[r][c] + min(dp[c], dp[c - 1])

    return dp[-1]
```

**Complexity:** Time O(m × n), Space O(n). Notice how grid DP problems share
the exact same skeleton — only the combination function changes (sum vs. count
vs. min vs. max).

---

## 5. String DP

String DP problems compare, transform, or analyze **one or two strings**. The
table is typically `(len(s1) + 1) × (len(s2) + 1)`.

---

### Problem 5: Edit Distance (Levenshtein Distance)

**LeetCode 72**

#### Problem Statement

Given two strings `word1` and `word2`, return the **minimum number of
operations** to convert `word1` into `word2`. Allowed operations: **Insert**,
**Delete**, **Replace** (each costs 1).

#### How to Recognize This Pattern

- Transforming one string into another with **minimum cost**.
- Three choices at each position (insert, delete, replace).
- Classic 2-string DP.

#### Approach — Full Walkthrough

Let's trace through `word1 = "horse"` and `word2 = "ros"`.

**Step 1: Define the table.**

`dp[i][j]` = minimum edits to convert `word1[0..i-1]` into `word2[0..j-1]`.

The table has dimensions `(len(word1) + 1) × (len(word2) + 1)`.

**Step 2: Fill base cases.**

- `dp[i][0] = i` → deleting all `i` characters from `word1` to get empty string.
- `dp[0][j] = j` → inserting all `j` characters of `word2`.

**Step 3: Fill the grid.**

For each cell `(i, j)`:

- If `word1[i-1] == word2[j-1]`: characters match, no operation needed →
  `dp[i][j] = dp[i-1][j-1]`
- Else, take the minimum of three operations:
  - **Replace:** `dp[i-1][j-1] + 1`
  - **Delete:** `dp[i-1][j] + 1` (remove char from word1)
  - **Insert:** `dp[i][j-1] + 1` (insert char into word1)

**Grid Visualization:**

```
        ""    r    o    s
  ""  [  0    1    2    3 ]
   h  [  1    1    2    3 ]
   o  [  2    2    1    2 ]
   r  [  3    2    2    2 ]
   s  [  4    3    3    2 ]
   e  [  5    4    4    3 ]
```

Reading the path: The answer is `dp[5][3] = 3`. The three operations are:
replace `h→r`, delete `r`, delete `e`.

#### State Definition

`dp[i][j]` = minimum operations to convert `word1[0..i-1]` to `word2[0..j-1]`.

#### Recurrence Relation

```
If word1[i-1] == word2[j-1]:
    dp[i][j] = dp[i-1][j-1]
Else:
    dp[i][j] = 1 + min(dp[i-1][j-1],   # Replace
                        dp[i-1][j],      # Delete
                        dp[i][j-1])      # Insert
```

#### Base Cases

```
dp[i][0] = i    for all i
dp[0][j] = j    for all j
```

#### Code

```python
def min_distance(word1: str, word2: str) -> int:
    """Return the minimum edit distance between word1 and word2."""
    m, n = len(word1), len(word2)
    dp = [[0] * (n + 1) for _ in range(m + 1)]

    # Base cases
    for i in range(m + 1):
        dp[i][0] = i
    for j in range(n + 1):
        dp[0][j] = j

    # Fill the table
    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if word1[i - 1] == word2[j - 1]:
                dp[i][j] = dp[i - 1][j - 1]
            else:
                dp[i][j] = 1 + min(
                    dp[i - 1][j - 1],  # Replace
                    dp[i - 1][j],      # Delete
                    dp[i][j - 1],      # Insert
                )

    return dp[m][n]
```

#### Space-Optimized Version

Since each row only depends on the current and previous row, we can reduce space
to O(min(m, n)):

```python
def min_distance_optimized(word1: str, word2: str) -> int:
    """Space-optimized edit distance using two rows."""
    m, n = len(word1), len(word2)

    # Ensure n <= m for minimal space usage.
    if m < n:
        return min_distance_optimized(word2, word1)

    prev = list(range(n + 1))
    curr = [0] * (n + 1)

    for i in range(1, m + 1):
        curr[0] = i
        for j in range(1, n + 1):
            if word1[i - 1] == word2[j - 1]:
                curr[j] = prev[j - 1]
            else:
                curr[j] = 1 + min(prev[j - 1], prev[j], curr[j - 1])
        prev, curr = curr, prev

    return prev[n]
```

#### Complexity

| | Full Table | Space-Optimized |
|---|---|---|
| Time | O(m × n) | O(m × n) |
| Space | O(m × n) | O(min(m, n)) |

> **Interview Tip:** Walk through the grid on the whiteboard with a small
> example. Interviewers want to see you **understand** the recurrence, not just
> memorize the code. Explain what each of the three operations means
> physically: "Insert means we still need to match `word1[0..i-1]` against a
> shorter prefix of `word2`."

---

### Problem 6: Longest Common Subsequence (LCS)

**LeetCode 1143**

#### Problem Statement

Given two strings `text1` and `text2`, return the length of their **longest
common subsequence**. A subsequence is a sequence derived by deleting zero or
more characters **without changing the relative order** of the remaining
characters.

#### How to Recognize This Pattern

- Comparing **two sequences** for common structure.
- Subsequence (not substring) — elements don't have to be contiguous.
- Classic 2-string DP.

#### Approach

Compare characters from both strings. If they match, extend the LCS by 1. If
they don't, try skipping a character from either string and take the better
result.

#### State Definition

`dp[i][j]` = length of LCS of `text1[0..i-1]` and `text2[0..j-1]`.

#### Recurrence Relation

```
If text1[i-1] == text2[j-1]:
    dp[i][j] = dp[i-1][j-1] + 1
Else:
    dp[i][j] = max(dp[i-1][j], dp[i][j-1])
```

#### Base Cases

```
dp[i][0] = 0    for all i  (LCS with empty string is 0)
dp[0][j] = 0    for all j
```

#### Grid Visualization

For `text1 = "abcde"` and `text2 = "ace"`:

```
        ""    a    c    e
  ""  [  0    0    0    0 ]
   a  [  0    1    1    1 ]
   b  [  0    1    1    1 ]
   c  [  0    1    2    2 ]
   d  [  0    1    2    2 ]
   e  [  0    1    2    3 ]
```

The LCS has length **3** (`"ace"`).

#### Code

```python
def longest_common_subsequence(text1: str, text2: str) -> int:
    """Return the length of the longest common subsequence."""
    m, n = len(text1), len(text2)
    dp = [[0] * (n + 1) for _ in range(m + 1)]

    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if text1[i - 1] == text2[j - 1]:
                dp[i][j] = dp[i - 1][j - 1] + 1
            else:
                dp[i][j] = max(dp[i - 1][j], dp[i][j - 1])

    return dp[m][n]
```

#### Recovering the Actual Subsequence

Interviewers sometimes ask you to **print** the LCS, not just its length.
Backtrack through the table:

```python
def get_lcs(text1: str, text2: str) -> str:
    """Return one longest common subsequence string."""
    m, n = len(text1), len(text2)
    dp = [[0] * (n + 1) for _ in range(m + 1)]

    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if text1[i - 1] == text2[j - 1]:
                dp[i][j] = dp[i - 1][j - 1] + 1
            else:
                dp[i][j] = max(dp[i - 1][j], dp[i][j - 1])

    # Backtrack to find the actual subsequence.
    result: list[str] = []
    i, j = m, n
    while i > 0 and j > 0:
        if text1[i - 1] == text2[j - 1]:
            result.append(text1[i - 1])
            i -= 1
            j -= 1
        elif dp[i - 1][j] >= dp[i][j - 1]:
            i -= 1
        else:
            j -= 1

    return "".join(reversed(result))
```

#### Complexity

| | Value |
|---|---|
| Time | O(m × n) |
| Space | O(m × n) — can be optimized to O(min(m, n)) |

> **Interview Tip:** LCS is a **building block** for many other problems.
> Longest Palindromic Subsequence = LCS of the string and its reverse. Shortest
> Common Supersequence = `m + n - LCS`. Know these reductions.

---

## 6. The Knapsack Pattern

The 0/1 Knapsack is not a single problem — it's a **meta-pattern** that appears
in dozens of interview questions.

### 6.1 Classic 0/1 Knapsack

Given `n` items, each with a `weight` and a `value`, and a knapsack with
capacity `W`, find the **maximum total value** you can carry. Each item is used
**at most once**.

#### State Definition

`dp[i][w]` = max value achievable using items `0..i-1` with capacity `w`.

#### Recurrence

```
If weight[i-1] <= w:
    dp[i][w] = max(dp[i-1][w],                         # Skip item i
                   dp[i-1][w - weight[i-1]] + value[i-1])  # Take item i
Else:
    dp[i][w] = dp[i-1][w]  # Can't take item i
```

#### Code

```python
def knapsack(weights: list[int], values: list[int], capacity: int) -> int:
    """Solve the 0/1 knapsack problem."""
    n = len(weights)
    dp = [[0] * (capacity + 1) for _ in range(n + 1)]

    for i in range(1, n + 1):
        for w in range(capacity + 1):
            dp[i][w] = dp[i - 1][w]  # Skip
            if weights[i - 1] <= w:
                dp[i][w] = max(
                    dp[i][w],
                    dp[i - 1][w - weights[i - 1]] + values[i - 1],
                )

    return dp[n][capacity]
```

**Complexity:** Time O(n × W), Space O(n × W) — can be optimized to O(W) with
a 1-D array iterated **backwards**.

### 6.2 Problems That Reduce to Knapsack

| Problem | Knapsack Mapping |
|---|---|
| Subset Sum | Values = weights, target = sum |
| Partition Equal Subset Sum | Target = total_sum / 2 |
| Target Sum (+/- signs) | Subset sum variant |
| Coin Change (bounded) | Bounded knapsack |
| Coin Change (unlimited) | Unbounded knapsack |
| Rod Cutting | Unbounded knapsack |

### 6.3 Longest Palindromic Subsequence

A quick note on this classic: the longest palindromic subsequence of string `s`
equals the **LCS of `s` and `reverse(s)`**. No new DP needed — just reuse the
LCS solution:

```python
def longest_palindrome_subseq(s: str) -> int:
    """Return the length of the longest palindromic subsequence."""
    return longest_common_subsequence(s, s[::-1])
```

This reduction is a perfect example of why mastering core patterns pays off.

### 6.4 Space-Optimized 0/1 Knapsack

Because row `i` only depends on row `i-1`, we can collapse the 2-D table to a
single 1-D array — but we must iterate **backwards** through capacities to
avoid overwriting values we still need:

```python
def knapsack_1d(weights: list[int], values: list[int], capacity: int) -> int:
    """Space-optimized 0/1 knapsack using a single row."""
    dp = [0] * (capacity + 1)

    for i in range(len(weights)):
        for w in range(capacity, weights[i] - 1, -1):  # Backwards!
            dp[w] = max(dp[w], dp[w - weights[i]] + values[i])

    return dp[capacity]
```

Why backwards? If we iterate forward, `dp[w - weights[i]]` may already reflect
the current item `i` (not the previous row), effectively allowing us to pick
item `i` multiple times — that would be the **unbounded** knapsack, which is a
different problem.

> **Interview Tip:** If you can explain *why* the loop goes backwards, you
> demonstrate genuine understanding rather than memorization. Interviewers love
> this.

---

## DP Pattern Recognition Cheat Sheet

Use this table during practice to **quickly classify** a new DP problem:

| Pattern | Clues in Problem Statement | State Shape | Examples |
|---|---|---|---|
| **Linear (1-D)** | "steps," "houses," single sequence | `dp[i]` | Climbing Stairs, House Robber, Decode Ways |
| **Target Sum** | "make amount," "reach target" | `dp[amount]` | Coin Change, Perfect Squares |
| **Grid (2-D)** | "grid," "matrix," "move right/down" | `dp[r][c]` | Unique Paths, Min Path Sum, Dungeon Game |
| **Two-String** | Two input strings, compare/transform | `dp[i][j]` | Edit Distance, LCS, Interleaving |
| **Single-String** | Palindromes, partitioning one string | `dp[i][j]` on substrings | Palindrome Partition, Longest Palindromic Substr |
| **0/1 Knapsack** | "pick or skip," capacity constraint | `dp[i][w]` | Subset Sum, Partition, Target Sum |
| **Unbounded Knapsack** | "unlimited supply," reuse allowed | `dp[w]` | Coin Change, Rod Cutting |
| **Interval DP** | "merge," "burst," ranges `[i..j]` | `dp[i][j]` on intervals | Burst Balloons, Matrix Chain Mult |
| **Bitmask DP** | Small `n` (≤ 20), subsets matter | `dp[mask]` | Travelling Salesman, Partition to K Subsets |
| **Tree DP** | Decisions on tree nodes (root/children) | `dp[node]` | House Robber III, Binary Tree Cameras |

### The 5-Step DP Framework (Use This in Every Interview)

1. **Define the state** — What does `dp[i]` (or `dp[i][j]`) represent?
2. **Find the recurrence** — How does `dp[i]` relate to smaller subproblems?
3. **Identify base cases** — What are the trivial answers?
4. **Determine iteration order** — Which direction fills the table correctly?
5. **Optimize space** — Can you reduce from 2-D to 1-D, or 1-D to O(1)?

Write these five steps on the whiteboard **before** coding. It shows structured
thinking and buys you time to think without awkward silence.

---

## Summary

| Problem | Pattern | Time | Space |
|---|---|---|---|
| Climbing Stairs | Linear | O(n) | O(1) |
| House Robber | Linear (take/skip) | O(n) | O(1) |
| Coin Change | Target Sum | O(amount × coins) | O(amount) |
| Unique Paths | Grid | O(m × n) | O(n) |
| Edit Distance | Two-String | O(m × n) | O(min(m, n)) |
| Longest Common Subsequence | Two-String | O(m × n) | O(m × n) |

Dynamic Programming is not about memorizing solutions — it's about recognizing
**patterns** and applying a **systematic framework**. Practice these six
problems until the state definitions and recurrences feel automatic. Then move to
harder variants (House Robber II, Coin Change II, Edit Distance with different
costs) — you'll find they're all variations of the patterns you already know.

Next up: **Module 5 — Sorting, Searching & Heaps**, where we trade overlapping
subproblems for divide-and-conquer and priority queues. Onward! 🚀
