# Module 1: Arrays, Strings & Sliding Window

> **Goal:** Master the foundational array and string techniques that appear in 40–50% of all coding interviews. By the end of this module you should be able to recognize *which* pattern fits a problem in under 60 seconds and implement it cleanly in Python.

---

## 1. Two-Pointer Technique

### When to Use It

The two-pointer technique shines when:

- The input is **sorted** (or you can afford to sort it).
- You need to find a **pair** of elements satisfying some condition.
- You want to reduce an O(n²) brute-force down to O(n).

The core idea is dead simple: place one pointer at the start and one at the end. Based on the current result, decide which pointer to move inward. Because each pointer only moves in one direction, the total work is O(n).

### How the Pointers Move

| Condition | Action |
|---|---|
| Current sum **too small** | Move left pointer right (increase) |
| Current sum **too large** | Move right pointer left (decrease) |
| **Match found** | Record result; move either/both |

### Why It Works (The Proof Intuition)

Consider a sorted array and a target sum. Suppose the answer is at indices `(i, j)`. The two-pointer approach will always find this pair because:

1. If `left < i`, then `nums[left] + nums[right] ≤ nums[i] + nums[j] = target` (since `nums[left] ≤ nums[i]`). We would move `left` right, getting closer to `i`.
2. Similarly, if `right > j`, we would move `right` left.
3. We never "skip over" the answer because we only move a pointer when the current sum tells us to.

This is a powerful argument to make in an interview — it shows you can reason about correctness, not just code a solution.

### Variations

- **Same-direction pointers** — fast/slow pointer for linked lists, removing duplicates in-place, partitioning arrays.
- **Opposite-direction pointers** — sorted two-sum, palindrome checking, container with most water, trapping rain water.
- **Three pointers** — Dutch National Flag problem (sort colors), 3Sum (fix one + two-pointer on the rest).

### Palindrome Check Example

A classic same-direction application:

```python
def is_palindrome(s: str) -> bool:
    """Check if a string reads the same forwards and backwards."""
    left, right = 0, len(s) - 1
    while left < right:
        if s[left] != s[right]:
            return False
        left += 1
        right -= 1
    return True
```

This runs in O(n) time and O(1) space — no need to reverse the string and compare.

---

## 2. Sliding Window

The sliding window pattern maintains a **window** of elements and slides it across the array, updating state incrementally instead of recomputing from scratch. Think of it as a caterpillar crawling along the array — the front extends forward, and the back catches up.

### Fixed-Size Window

When the problem says "subarray of size k," the template is:

```
1. Compute the initial window (first k elements).
2. Slide: add the new right element, remove the old left element.
3. Update the answer at each step.
```

**Time:** O(n). **Space:** O(1) for sum-based problems.

**Why is this better than brute force?** A brute-force approach would compute the sum of every size-k subarray independently, taking O(k) per window × O(n) windows = O(nk). By *reusing* the previous window's sum and adjusting incrementally (+new, -old), each slide costs O(1).

### Variable-Size Window

When you need the *shortest* or *longest* subarray satisfying a condition:

```python
left = 0
for right in range(len(arr)):
    # Expand: add arr[right] to window state
    while window_is_invalid():
        # Shrink: remove arr[left] from window state
        left += 1
    # Update answer (longest → here; shortest → inside the while)
```

**Key insight:** The `left` pointer never moves backward, so total work is O(n), not O(n²). Even though there's a `while` loop inside a `for` loop, the `left` pointer moves at most n times *total* across all iterations.

### Shortest vs Longest — Where to Update

This trips up many people:

- **Longest** subarray/substring: update the answer **after** the shrink loop (window is valid, maximize).
- **Shortest** subarray/substring: update the answer **inside** the shrink loop (window is valid, minimize before shrinking further).

### When Sliding Window Fails

If the array has **negative numbers** and you need a target sum, sliding window breaks because shrinking the window doesn't guarantee the sum increases. The monotonic property is lost. Use **prefix sums + hashmap** instead (Section 3).

---

## 3. Prefix Sums

A prefix sum array `P` where `P[i] = nums[0] + nums[1] + ... + nums[i-1]` lets you compute any range sum in O(1):

```
sum(nums[i:j]) = P[j] - P[i]
```

### Building a Prefix Sum Array

```python
def build_prefix_sum(nums: list[int]) -> list[int]:
    """Build prefix sum array. P[0] = 0, P[i] = sum(nums[:i])."""
    prefix = [0] * (len(nums) + 1)
    for i in range(len(nums)):
        prefix[i + 1] = prefix[i] + nums[i]
    return prefix

# Usage: sum of nums[i:j] = prefix[j] - prefix[i]
```

This is the foundation for range-sum queries, 2D prefix sums (submatrix sums), and the powerful prefix-sum-plus-hashmap pattern.

### Prefix Sum + HashMap Pattern

For "subarray sum equals K" problems:

1. Maintain a running `prefix_sum`.
2. At each index, check if `prefix_sum - K` exists in a hashmap of previously seen prefix sums.
3. If it does, that means there's a subarray ending here with sum K.

This pattern handles **negative numbers** gracefully — something sliding window cannot do.

### Why It Works

If `prefix_sum[j] - prefix_sum[i] == K`, then the subarray `nums[i+1..j]` sums to K. By storing prefix sums in a hashmap as we go, we turn an O(n²) enumeration of all pairs `(i, j)` into O(n) lookups.

### Product of Array Except Self

A related prefix technique: build a **prefix product** (left-to-right) and a **suffix product** (right-to-left). For each index `i`, the answer is `left_product[i] * right_product[i]`. This avoids division (which would fail with zeros) and runs in O(n) time.

```python
def product_except_self(nums: list[int]) -> list[int]:
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

---

## 4. Kadane's Algorithm

Kadane's algorithm finds the **maximum subarray sum** in O(n) time and O(1) space. The intuition is elegantly simple:

> At each position, decide: is it better to **extend** the current subarray, or **start fresh** from here?

If the running sum is positive, it can potentially help future elements. If it's negative, it can only hurt — so we reset. This greedy choice happens to produce the globally optimal answer.

```python
current_sum = max_sum = nums[0]
for num in nums[1:]:
    current_sum = max(num, current_sum + num)  # extend or reset
    max_sum = max(max_sum, current_sum)
```

### The DP Connection

Kadane's is actually dynamic programming in disguise. Define `dp[i]` as the maximum subarray sum ending at index `i`. The recurrence is:

```
dp[i] = max(nums[i], dp[i-1] + nums[i])
```

Since `dp[i]` only depends on `dp[i-1]`, we optimize the array away to a single variable (`current_sum`). This DP-to-optimized progression is a common interview pattern — show the DP array first, then optimize.

### Handling All-Negative Arrays

The version above already handles all-negative arrays correctly because we initialize with `nums[0]` and always consider the single-element case (`num` alone). A common bug is initializing `max_sum = 0`, which would return 0 for `[-3, -2, -1]` instead of the correct `-1`.

### Variations

- **Maximum subarray product** — track both max and min (a negative × negative = positive). At each step: `new_max = max(num, max_prev * num, min_prev * num)`.
- **Circular subarray** — the max circular subarray is either a normal subarray (Kadane's) or wraps around. Wrapping = total_sum - min_subarray. Answer = max(kadane_max, total_sum - kadane_min). Edge case: if all elements are negative, kadane_min = total_sum, so the wrap case gives 0 — handle separately.
- **Maximum sum with at most k deletions** — extend Kadane's with a 2D DP table.

---

## 5. String Manipulation

### Core String Patterns

| Pattern | Technique |
|---|---|
| Anagram detection | Sort both → compare, OR frequency count |
| Palindrome check | Two pointers from edges inward |
| Longest common prefix | Vertical scan or sort + compare first/last |
| String parsing (atoi) | State machine, handle signs/overflow |
| Substring search | Sliding window, or KMP for pattern matching |
| Character replacement | Sliding window + frequency count |

### Python String Tips

- Strings are **immutable** in Python. Building a string with `+=` in a loop is O(n²) because each concatenation creates a new string object. Use `"".join(parts)` instead — it pre-allocates and copies once, giving O(n) total.
- `collections.Counter` is your best friend for frequency counting.
- `ord(c) - ord('a')` gives you a 0-25 index for lowercase letters — useful for fixed-size frequency arrays.
- `str.isalnum()`, `str.isdigit()`, `str.isalpha()` — know these for input validation problems.

### Valid Anagram

Two approaches, both O(n):

```python
from collections import Counter

def is_anagram(s: str, t: str) -> bool:
    """Check if t is an anagram of s."""
    return Counter(s) == Counter(t)

def is_anagram_array(s: str, t: str) -> bool:
    """Same thing, but with a fixed-size array (faster for lowercase)."""
    if len(s) != len(t):
        return False
    freq = [0] * 26
    for a, b in zip(s, t):
        freq[ord(a) - ord('a')] += 1
        freq[ord(b) - ord('a')] -= 1
    return all(f == 0 for f in freq)
```

### Longest Common Prefix

Vertical scanning — compare character by character across all strings:

```python
def longest_common_prefix(strs: list[str]) -> str:
    """Find the longest common prefix among a list of strings."""
    if not strs:
        return ""
    for i, char in enumerate(strs[0]):
        for s in strs[1:]:
            if i >= len(s) or s[i] != char:
                return strs[0][:i]
    return strs[0]
```

### String to Integer (atoi) — Edge Case Minefield

This problem tests your ability to handle edge cases methodically:

1. Strip leading whitespace.
2. Handle optional `+` or `-` sign.
3. Convert digits until a non-digit is encountered.
4. Clamp to 32-bit integer range `[-2^31, 2^31 - 1]`.

```python
def my_atoi(s: str) -> int:
    """Convert string to 32-bit signed integer."""
    s = s.lstrip()  # step 1: strip whitespace
    if not s:
        return 0

    sign = 1
    idx = 0

    if s[0] in ('+', '-'):  # step 2: handle sign
        sign = -1 if s[0] == '-' else 1
        idx = 1

    result = 0
    while idx < len(s) and s[idx].isdigit():  # step 3: convert digits
        result = result * 10 + int(s[idx])
        idx += 1

    result *= sign

    # Step 4: clamp to 32-bit range
    INT_MIN, INT_MAX = -(2**31), 2**31 - 1
    return max(INT_MIN, min(INT_MAX, result))
```

---

## Problem Solutions

---

### Problem 1: Two Sum II — Sorted Array (Two Pointers)

**Problem:** Given a 1-indexed sorted array `numbers` and a `target`, find two numbers that add up to `target`. Return their indices `[i, j]` (1-indexed).

**Approach:**
Place `left` at index 0 and `right` at the last index. If the sum is too small, move `left` right. If too large, move `right` left. The sorted order guarantees convergence.

```python
def two_sum_sorted(numbers: list[int], target: int) -> list[int]:
    """
    Find two numbers in a sorted array that sum to target.
    Returns 1-indexed positions.

    Time:  O(n) — each pointer moves at most n steps total.
    Space: O(1) — only two variables.
    """
    left, right = 0, len(numbers) - 1

    while left < right:
        current_sum = numbers[left] + numbers[right]

        if current_sum == target:
            return [left + 1, right + 1]  # 1-indexed
        elif current_sum < target:
            left += 1   # need a bigger sum → move left pointer right
        else:
            right -= 1  # need a smaller sum → move right pointer left

    return []  # no solution found (problem guarantees one exists)
```

**Edge Cases:**
- Duplicate values: works fine — pointers skip naturally.
- Negative numbers: the sorted property still holds; algorithm is unchanged.

> **Interview Tip:** If the array is *unsorted*, don't sort it and use two pointers (you'd lose original indices). Use a hashmap instead — that's the classic Two Sum (Module 2). Always clarify: "Is the input sorted?" before choosing your approach.

---

### Problem 2: Container With Most Water

**Problem:** Given an array `height` where each element represents a vertical line, find two lines that together with the x-axis form a container holding the most water.

**Approach:**
Start with the widest container (pointers at both ends). The area is `min(height[left], height[right]) * (right - left)`. Move the pointer pointing to the **shorter** line inward — keeping the shorter line can never improve the area since width is decreasing.

```python
def max_area(height: list[int]) -> int:
    """
    Find the maximum water a container can hold.

    Time:  O(n) — single pass with two pointers.
    Space: O(1).
    """
    left, right = 0, len(height) - 1
    best = 0

    while left < right:
        # Area is limited by the shorter wall
        width = right - left
        h = min(height[left], height[right])
        best = max(best, h * width)

        # Move the shorter side inward — keeping it can't help
        if height[left] < height[right]:
            left += 1
        else:
            right -= 1

    return best
```

**Why move the shorter side?**

Imagine `height[left] = 3` and `height[right] = 7`. The area is bounded by `3 * width`. If we move `right` inward, width shrinks and the bounding height stays ≤ 3 (it can't improve). Moving `left` inward might find a taller left wall, actually increasing the area despite the smaller width.

> **Interview Tip:** This problem tests whether you can *prove* greedy correctness. Practice articulating *why* moving the shorter side is optimal — interviewers love hearing that reasoning.

---

### Problem 3: Maximum Sum Subarray of Size K (Fixed Sliding Window)

**Problem:** Given an array of integers and a number `k`, find the maximum sum of any contiguous subarray of size `k`.

**Approach:**
Compute the sum of the first `k` elements. Then slide the window: add the new right element, subtract the old left element. Track the maximum.

```python
def max_sum_subarray_k(nums: list[int], k: int) -> int:
    """
    Find the maximum sum of a subarray of exactly size k.

    Time:  O(n) — single pass after initial window.
    Space: O(1).
    """
    if len(nums) < k:
        return 0  # or raise ValueError

    # Compute initial window sum
    window_sum = sum(nums[:k])
    max_sum = window_sum

    # Slide the window: add right, remove left
    for i in range(k, len(nums)):
        window_sum += nums[i] - nums[i - k]
        max_sum = max(max_sum, window_sum)

    return max_sum
```

**Dry Run** with `nums = [2, 1, 5, 1, 3, 2]`, `k = 3`:

| Window | Elements | Sum |
|---|---|---|
| [0:3] | 2, 1, 5 | 8 |
| [1:4] | 1, 5, 1 | 7 |
| [2:5] | 5, 1, 3 | 9 ← max |
| [3:6] | 1, 3, 2 | 6 |

> **Interview Tip:** Fixed-size sliding window is the simplest pattern. If you see "contiguous subarray of size k," your muscle memory should immediately reach for this. The key optimization insight: instead of recomputing the sum each time (O(k) per window = O(nk) total), you update incrementally in O(1) per step.

---

### Problem 4: Longest Substring Without Repeating Characters (Variable Sliding Window)

**Problem:** Given a string `s`, find the length of the longest substring without repeating characters.

**Approach:**
Use a variable-size sliding window with a hashmap tracking the last seen index of each character. When a repeat is found, jump the left pointer past the previous occurrence.

```python
def length_of_longest_substring(s: str) -> int:
    """
    Find the longest substring with all unique characters.

    Time:  O(n) — each character is visited at most twice.
    Space: O(min(n, 26)) — hashmap stores at most alphabet-size entries.
    """
    char_index: dict[str, int] = {}  # char → last seen index
    left = 0
    max_length = 0

    for right, char in enumerate(s):
        # If char was seen and is inside our current window
        if char in char_index and char_index[char] >= left:
            left = char_index[char] + 1  # shrink past the duplicate

        char_index[char] = right  # update last seen position
        max_length = max(max_length, right - left + 1)

    return max_length
```

**Walk-through** with `s = "abcabcbb"`:

| right | char | char_index | left | window | length |
|---|---|---|---|---|---|
| 0 | a | {a:0} | 0 | "a" | 1 |
| 1 | b | {a:0,b:1} | 0 | "ab" | 2 |
| 2 | c | {a:0,b:1,c:2} | 0 | "abc" | 3 |
| 3 | a | {a:3,b:1,c:2} | 1 | "bca" | 3 |
| 4 | b | {a:3,b:4,c:2} | 2 | "cab" | 3 |
| 5 | c | {a:3,b:4,c:5} | 3 | "abc" | 3 |
| 6 | b | {a:3,b:6,c:5} | 5 | "cb" | 2 |
| 7 | b | {a:3,b:7,c:5} | 7 | "b" | 1 |

**Answer:** 3 (`"abc"`)

> **Interview Tip:** The subtle bug here is forgetting the `char_index[char] >= left` check. Without it, you might jump `left` backward to a stale position from a previous window. Always remember: the hashmap stores *global* last-seen indices, but we only care about characters *within the current window*.

---

### Problem 5: Subarray Sum Equals K (Prefix Sum + HashMap)

**Problem:** Given an integer array `nums` and an integer `k`, return the total number of subarrays whose sum equals `k`. The array may contain negative numbers.

**Approach:**
Maintain a running `prefix_sum`. At each index, the number of valid subarrays ending here equals the number of times we've previously seen `prefix_sum - k`. Store prefix sum frequencies in a hashmap.

```python
def subarray_sum(nums: list[int], k: int) -> int:
    """
    Count subarrays with sum equal to k.
    Handles negative numbers (sliding window cannot).

    Time:  O(n) — single pass with hashmap lookups.
    Space: O(n) — hashmap stores up to n prefix sums.
    """
    count = 0
    prefix_sum = 0
    # Map: prefix_sum value → number of times we've seen it
    # Initialize with {0: 1} because a prefix_sum of 0 means
    # the subarray from index 0 to current index sums to k.
    prefix_counts: dict[int, int] = {0: 1}

    for num in nums:
        prefix_sum += num

        # If (prefix_sum - k) was seen before, those occurrences
        # mark starting points of subarrays summing to k
        complement = prefix_sum - k
        if complement in prefix_counts:
            count += prefix_counts[complement]

        # Record current prefix sum
        prefix_counts[prefix_sum] = prefix_counts.get(prefix_sum, 0) + 1

    return count
```

**Why `{0: 1}` initialization?**

Consider `nums = [3]` and `k = 3`. After processing, `prefix_sum = 3`. We check if `3 - 3 = 0` was seen before. Without the initial `{0: 1}`, we'd miss this valid subarray. The initial entry represents the "empty prefix" — the sum before any element.

**Walk-through** with `nums = [1, 2, 3]`, `k = 3`:

| Index | num | prefix_sum | complement | count | prefix_counts |
|---|---|---|---|---|---|
| — | — | 0 | — | 0 | {0: 1} |
| 0 | 1 | 1 | -2 | 0 | {0:1, 1:1} |
| 1 | 2 | 3 | 0 ✓ | 1 | {0:1, 1:1, 3:1} |
| 2 | 3 | 6 | 3 ✓ | 2 | {0:1, 1:1, 3:1, 6:1} |

Subarrays: `[1,2]` and `[3]` → count = 2. ✓

> **Interview Tip:** This is one of the most important patterns in all of DSA. The prefix-sum-plus-hashmap approach generalizes to many problems: count of subarrays divisible by K (store `prefix_sum % k`), longest subarray with sum K (store first occurrence index), and binary subarray with sum (treat 0s and 1s). Master this pattern cold.

---

### Problem 6: Maximum Subarray — Kadane's Algorithm

**Problem:** Given an integer array `nums`, find the contiguous subarray with the largest sum and return that sum.

**Approach:**
At each element, decide whether to extend the existing subarray or start a new one from the current element. If the running sum is negative, it can only hurt us — start fresh.

```python
def max_subarray(nums: list[int]) -> int:
    """
    Find the maximum sum of any contiguous subarray.
    Kadane's Algorithm.

    Time:  O(n) — single pass.
    Space: O(1) — two variables.
    """
    if not nums:
        return 0

    current_sum = max_sum = nums[0]

    for num in nums[1:]:
        # Core decision: extend the current subarray or start fresh?
        # If current_sum + num < num, the running sum is hurting us.
        current_sum = max(num, current_sum + num)
        max_sum = max(max_sum, current_sum)

    return max_sum
```

**Returning the subarray itself** (a common follow-up):

```python
def max_subarray_with_indices(nums: list[int]) -> tuple[int, int, int]:
    """
    Returns (max_sum, start_index, end_index).
    """
    current_sum = max_sum = nums[0]
    start = end = temp_start = 0

    for i in range(1, len(nums)):
        if nums[i] > current_sum + nums[i]:
            current_sum = nums[i]
            temp_start = i  # starting a new subarray
        else:
            current_sum += nums[i]

        if current_sum > max_sum:
            max_sum = current_sum
            start = temp_start
            end = i

    return max_sum, start, end
```

**Walk-through** with `nums = [-2, 1, -3, 4, -1, 2, 1, -5, 4]`:

| Index | num | current_sum | max_sum |
|---|---|---|---|
| 0 | -2 | -2 | -2 |
| 1 | 1 | max(1, -2+1)=1 | 1 |
| 2 | -3 | max(-3, 1-3)=-2 | 1 |
| 3 | 4 | max(4, -2+4)=4 | 4 |
| 4 | -1 | max(-1, 4-1)=3 | 4 |
| 5 | 2 | max(2, 3+2)=5 | 5 |
| 6 | 1 | max(1, 5+1)=6 | 6 |
| 7 | -5 | max(-5, 6-5)=1 | 6 |
| 8 | 4 | max(4, 1+4)=5 | 6 |

**Answer:** 6 (subarray `[4, -1, 2, 1]`)

> **Interview Tip:** Kadane's is a **dynamic programming** algorithm in disguise. The recurrence is `dp[i] = max(nums[i], dp[i-1] + nums[i])` — we just optimized away the array since we only need the previous value. If an interviewer asks "can you do this with DP?", show the array version first, then optimize to O(1) space. It demonstrates your ability to evolve a solution.

---

## Key Patterns Cheat Sheet

Use this decision framework when you see an array or string problem:

| Signal in Problem | Pattern to Use | Time |
|---|---|---|
| "Sorted array" + "find pair" | **Two Pointers** (opposite ends) | O(n) |
| "Contiguous subarray of size k" | **Fixed Sliding Window** | O(n) |
| "Longest/shortest subarray with property X" (positive values) | **Variable Sliding Window** | O(n) |
| "Subarray sum equals K" (may have negatives) | **Prefix Sum + HashMap** | O(n) |
| "Maximum/minimum subarray sum" | **Kadane's Algorithm** | O(n) |
| "Anagram / permutation check" | **Frequency Count** (Counter or array) | O(n) |
| "Palindrome check" | **Two Pointers** (edges inward) | O(n) |
| "Remove duplicates in-place" | **Two Pointers** (same direction) | O(n) |

### Quick Decision Tree

```
Is the input sorted?
├── Yes → Two Pointers (opposite direction)
└── No
    ├── Need a pair/complement? → HashMap lookup (O(1))
    ├── Contiguous subarray?
    │   ├── Fixed size k? → Fixed Sliding Window
    │   ├── All positive + target sum? → Variable Sliding Window
    │   ├── Has negatives + target sum? → Prefix Sum + HashMap
    │   └── Max/min subarray sum? → Kadane's Algorithm
    └── Substring with unique/frequency constraint? → Sliding Window + HashMap
```

### Complexity Reference

| Algorithm | Time | Space | Key Constraint |
|---|---|---|---|
| Two Pointers | O(n) | O(1) | Input must be sorted |
| Fixed Sliding Window | O(n) | O(1) | Window size is fixed |
| Variable Sliding Window | O(n) | O(k) | Monotonic property when shrinking |
| Prefix Sum + HashMap | O(n) | O(n) | Works with negatives |
| Kadane's Algorithm | O(n) | O(1) | Contiguous subarray only |

### Common Mistakes to Avoid

1. **Using sliding window with negative numbers** — the window loses its monotonic property. Use prefix sums instead.
2. **Forgetting `{0: 1}` in prefix sum hashmap** — you'll miss subarrays starting at index 0.
3. **Initializing Kadane's with 0** — fails for all-negative arrays. Start with `nums[0]`.
4. **Building strings with `+=`** — O(n²) in many Python implementations. Use `"".join()`.
5. **Off-by-one in sliding window** — carefully track whether your window is `[left, right]` inclusive or `[left, right)` exclusive. Pick one convention and stick with it.

---

*Next up: Module 2 — Hash Maps & Sets, where we dive deep into the data structure that makes O(1) lookup possible and powers half the patterns you just learned.*