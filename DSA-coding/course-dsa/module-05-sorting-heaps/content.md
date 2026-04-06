# Module 5: Sorting, Searching & Heaps

> *"Give me six hours to chop down a tree and I will spend the first four
> sharpening the axe."* — Abraham Lincoln (on the importance of choosing the
> right algorithm)

Sorting and searching are the **bread and butter** of computer science. Nearly
every real-world system — from databases to search engines — relies on sorted
data and efficient lookups. Heaps add a powerful tool for problems where you need
**repeated access to the extreme value** (min or max) in a changing dataset.

This module covers the algorithms you must know cold for interviews, plus six
classic problems that interviewers rotate through constantly.

---

## 1. Binary Search

Binary search is deceptively simple to describe and notoriously tricky to
implement without off-by-one errors. Master one **template** and stick with it.

### 1.1 The Core Idea

Given a **sorted** array, eliminate half the search space each step. This gives
O(log n) time — searching 1 billion elements takes only ~30 comparisons.

### 1.2 The Template

```python
def binary_search(nums: list[int], target: int) -> int:
    """Return index of target, or -1 if not found."""
    left, right = 0, len(nums) - 1

    while left <= right:
        mid = left + (right - left) // 2  # Avoids overflow in other languages.

        if nums[mid] == target:
            return mid
        elif nums[mid] < target:
            left = mid + 1
        else:
            right = mid - 1

    return -1
```

Key decision: `left <= right` (inclusive bounds) or `left < right` (exclusive
right)? Pick one and be consistent. The inclusive template above is the most
common in interviews.

### 1.3 `bisect_left` and `bisect_right` in Python

Python's `bisect` module provides battle-tested binary search:

```python
from bisect import bisect_left, bisect_right

nums = [1, 3, 3, 3, 5, 7]

# bisect_left: leftmost insertion point (first index where value >= target)
bisect_left(nums, 3)   # → 1

# bisect_right: rightmost insertion point (first index where value > target)
bisect_right(nums, 3)  # → 4
```

**Use `bisect_left`** to find the **first occurrence** of a value.
**Use `bisect_right`** to find the position **after the last occurrence**.

> **Interview Tip:** Mention `bisect` to show Python fluency, but be ready to
> implement binary search from scratch — interviewers will ask.

### 1.4 The Search Space Reduction Pattern

Binary search isn't limited to sorted arrays. It works on **any monotonic
predicate**: a function that returns `False, False, ..., True, True, True` (or
vice versa). You're searching for the **boundary**.

Examples beyond arrays:

- **Minimum capacity to ship packages in D days** — binary search on the answer.
- **Sqrt(x)** — binary search integers where `mid * mid <= x`.
- **Koko eating bananas** — binary search on eating speed.

The pattern:

```python
def binary_search_on_answer(lo: int, hi: int) -> int:
    """Find the smallest value where condition(mid) is True."""
    while lo < hi:
        mid = lo + (hi - lo) // 2
        if condition(mid):
            hi = mid       # mid might be the answer, keep searching left
        else:
            lo = mid + 1   # mid is too small
    return lo
```

---

## 2. Merge Sort

### 2.1 The Algorithm

Merge Sort is a **divide-and-conquer** algorithm:

1. **Divide** the array into two halves.
2. **Recursively sort** each half.
3. **Merge** the two sorted halves into one sorted array.

### 2.2 Why It Matters

- **Stable** sort (preserves relative order of equal elements).
- **Always** O(n log n) — no worst-case degradation.
- Foundation for **external sorting** (sorting data that doesn't fit in memory).
- The merge step is used directly in "Merge K Sorted Lists" and "Count
  Inversions."

### 2.3 Code Walkthrough

```python
def merge_sort(arr: list[int]) -> list[int]:
    """Return a new sorted list using merge sort."""
    if len(arr) <= 1:
        return arr

    mid = len(arr) // 2
    left = merge_sort(arr[:mid])
    right = merge_sort(arr[mid:])

    return _merge(left, right)


def _merge(left: list[int], right: list[int]) -> list[int]:
    """Merge two sorted lists into one sorted list."""
    result: list[int] = []
    i = j = 0

    while i < len(left) and j < len(right):
        if left[i] <= right[j]:     # <= ensures stability
            result.append(left[i])
            i += 1
        else:
            result.append(right[j])
            j += 1

    # One of the halves may have remaining elements.
    result.extend(left[i:])
    result.extend(right[j:])
    return result
```

### 2.4 Complexity

| | Value |
|---|---|
| Time (all cases) | O(n log n) |
| Space | O(n) — auxiliary array for merging |
| Stable | ✅ Yes |

> **Interview Tip:** If asked "Why not always use merge sort?" — the O(n) extra
> space is a real cost. Quick sort is in-place and often faster in practice due
> to cache locality.

---

## 3. Quick Sort

### 3.1 The Algorithm

1. Choose a **pivot** element.
2. **Partition** the array so elements ≤ pivot are on the left, elements > pivot
   are on the right.
3. **Recursively sort** the left and right partitions.

### 3.2 Lomuto vs. Hoare Partition

**Lomuto Partition** (simpler, commonly taught):

```python
def _lomuto_partition(arr: list[int], lo: int, hi: int) -> int:
    """Partition using the last element as pivot. Return pivot index."""
    pivot = arr[hi]
    i = lo - 1

    for j in range(lo, hi):
        if arr[j] <= pivot:
            i += 1
            arr[i], arr[j] = arr[j], arr[i]

    arr[i + 1], arr[hi] = arr[hi], arr[i + 1]
    return i + 1
```

**Hoare Partition** (fewer swaps on average, slightly trickier):

```python
def _hoare_partition(arr: list[int], lo: int, hi: int) -> int:
    """Partition using the first element as pivot."""
    pivot = arr[lo]
    i, j = lo - 1, hi + 1

    while True:
        i += 1
        while arr[i] < pivot:
            i += 1
        j -= 1
        while arr[j] > pivot:
            j -= 1
        if i >= j:
            return j
        arr[i], arr[j] = arr[j], arr[i]
```

### 3.3 Full Quick Sort with Randomized Pivot

```python
import random


def quick_sort(arr: list[int], lo: int = 0, hi: int | None = None) -> None:
    """Sort arr in-place using randomized quick sort."""
    if hi is None:
        hi = len(arr) - 1

    if lo < hi:
        # Randomize pivot to avoid worst-case on sorted input.
        rand_idx = random.randint(lo, hi)
        arr[rand_idx], arr[hi] = arr[hi], arr[rand_idx]

        pivot_idx = _lomuto_partition(arr, lo, hi)
        quick_sort(arr, lo, pivot_idx - 1)
        quick_sort(arr, pivot_idx + 1, hi)
```

### 3.4 Quick Select — Finding the Kth Element

Quick Select is Quick Sort's cousin — instead of sorting both sides, you
**only recurse into the side that contains the target index**. Average O(n),
worst O(n²).

```python
def quick_select(arr: list[int], k: int) -> int:
    """Return the k-th smallest element (0-indexed). Mutates arr."""
    lo, hi = 0, len(arr) - 1

    while lo < hi:
        rand_idx = random.randint(lo, hi)
        arr[rand_idx], arr[hi] = arr[hi], arr[rand_idx]

        pivot_idx = _lomuto_partition(arr, lo, hi)

        if pivot_idx == k:
            return arr[k]
        elif pivot_idx < k:
            lo = pivot_idx + 1
        else:
            hi = pivot_idx - 1

    return arr[lo]
```

### 3.5 Complexity

| | Average | Worst |
|---|---|---|
| Quick Sort Time | O(n log n) | O(n²) — mitigated by random pivot |
| Quick Select Time | O(n) | O(n²) |
| Space | O(log n) stack | O(n) stack worst case |
| Stable | ❌ No | |

---

## 4. Heaps

### 4.1 What Is a Heap?

A heap is a **complete binary tree** stored as an array where every parent
satisfies the heap property:

- **Min-Heap:** parent ≤ children → root is the **minimum**.
- **Max-Heap:** parent ≥ children → root is the **maximum**.

### 4.2 Python's `heapq` Module

Python only provides a **min-heap** via the `heapq` module. For a max-heap,
negate the values.

```python
import heapq

# Create a min-heap from a list.
nums = [5, 3, 8, 1, 2]
heapq.heapify(nums)          # O(n) — in-place

# Push a new element.
heapq.heappush(nums, 4)      # O(log n)

# Pop the smallest element.
smallest = heapq.heappop(nums)  # O(log n) → returns 1

# Push and pop in one operation (more efficient than separate calls).
heapq.heapreplace(nums, 6)   # Pop smallest, then push 6.

# Get the N smallest / largest without full sort.
heapq.nsmallest(3, nums)     # O(n log 3) — efficient for small k
heapq.nlargest(3, nums)
```

**Max-heap trick:**

```python
# Negate values on push, negate again on pop.
max_heap: list[int] = []
heapq.heappush(max_heap, -val)
largest = -heapq.heappop(max_heap)
```

### 4.3 Key Operations & Complexity

| Operation | Time |
|---|---|
| `heapify` | O(n) |
| `heappush` | O(log n) |
| `heappop` | O(log n) |
| Peek at min | O(1) — just `heap[0]` |
| Search | O(n) — heaps aren't for searching |

### 4.4 When to Use a Heap

Reach for a heap when you need:

- Repeated access to the **min or max** in a dynamic dataset.
- **Top-K** elements from a stream or large collection.
- **Merging K sorted** sequences efficiently.
- **Priority scheduling** (task queues, Dijkstra's algorithm).

---

## 5. The Top-K Pattern

This pattern appears in countless interview problems. The key insight: you
**don't need to sort** the entire collection.

### 5.1 Top-K Largest → Use a Min-Heap of Size K

Maintain a min-heap of size K. For each element, if it's larger than the heap's
minimum, replace the minimum. After processing all elements, the heap contains
the K largest.

```python
def top_k_largest(nums: list[int], k: int) -> list[int]:
    """Return the k largest elements using a min-heap."""
    heap = nums[:k]
    heapq.heapify(heap)

    for num in nums[k:]:
        if num > heap[0]:
            heapq.heapreplace(heap, num)

    return heap  # Unordered; sort if order matters.
```

**Why min-heap for largest?** The min-heap's root is the **smallest of the K
largest**. Any element bigger than this root deserves a spot; the root gets
evicted. Time: O(n log k). Space: O(k).

### 5.2 Top-K Smallest → Use a Max-Heap of Size K

Same idea, but negate values (or use `nsmallest`):

```python
def top_k_smallest(nums: list[int], k: int) -> list[int]:
    """Return the k smallest elements."""
    return heapq.nsmallest(k, nums)
```

---

## Problems

---

### Problem 1: Binary Search (Classic)

**LeetCode 704**

#### Problem Statement

Given a sorted array `nums` and a `target`, return the index of `target` or `-1`
if not found.

#### Code

```python
def search(nums: list[int], target: int) -> int:
    """Classic binary search on a sorted array."""
    left, right = 0, len(nums) - 1

    while left <= right:
        mid = left + (right - left) // 2

        if nums[mid] == target:
            return mid
        elif nums[mid] < target:
            left = mid + 1
        else:
            right = mid - 1

    return -1
```

#### Complexity

| | Value |
|---|---|
| Time | O(log n) |
| Space | O(1) |

> **Interview Tip:** Always use `left + (right - left) // 2` instead of
> `(left + right) // 2`. In Python it doesn't matter (arbitrary precision ints),
> but in Java/C++ it prevents integer overflow. Mentioning this shows awareness.

---

### Problem 2: Search in Rotated Sorted Array

**LeetCode 33**

#### Problem Statement

An ascending sorted array has been **rotated** at an unknown pivot (e.g.,
`[4,5,6,7,0,1,2]`). Given a `target`, return its index or `-1`. Assume no
duplicates.

#### Approach

At any `mid` point, **one half is always sorted**. Determine which half is
sorted, then check if the target lies within that sorted half. If yes, search
there; otherwise, search the other half.

#### Code

```python
def search_rotated(nums: list[int], target: int) -> int:
    """Binary search in a rotated sorted array."""
    left, right = 0, len(nums) - 1

    while left <= right:
        mid = left + (right - left) // 2

        if nums[mid] == target:
            return mid

        # Left half is sorted.
        if nums[left] <= nums[mid]:
            if nums[left] <= target < nums[mid]:
                right = mid - 1
            else:
                left = mid + 1
        # Right half is sorted.
        else:
            if nums[mid] < target <= nums[right]:
                left = mid + 1
            else:
                right = mid - 1

    return -1
```

#### Complexity

| | Value |
|---|---|
| Time | O(log n) |
| Space | O(1) |

> **Interview Tip:** The critical insight is `nums[left] <= nums[mid]` to
> determine which half is sorted. Walk through `[4,5,6,7,0,1,2]` with
> target `0` on the whiteboard to prove it works. If duplicates are allowed
> (LeetCode 81), worst case degrades to O(n) because you may need to skip
> duplicates with `left += 1`.

---

### Problem 3: Kth Largest Element in an Array

**LeetCode 215**

#### Problem Statement

Given an unsorted array `nums` and integer `k`, return the **kth largest**
element. Note: kth largest, not kth distinct.

#### Approach 1: Min-Heap of Size K

Maintain a min-heap of the K largest elements seen so far. The root is the
answer.

```python
import heapq


def find_kth_largest_heap(nums: list[int], k: int) -> int:
    """Find the kth largest using a min-heap of size k."""
    heap = nums[:k]
    heapq.heapify(heap)

    for num in nums[k:]:
        if num > heap[0]:
            heapq.heapreplace(heap, num)

    return heap[0]
```

**Complexity:** Time O(n log k), Space O(k).

#### Approach 2: Quick Select

Convert "kth largest" to "index `n - k`" in a sorted array, then use Quick
Select.

```python
import random


def find_kth_largest_quickselect(nums: list[int], k: int) -> int:
    """Find the kth largest using Quick Select."""
    target_idx = len(nums) - k

    def partition(lo: int, hi: int) -> int:
        rand = random.randint(lo, hi)
        nums[rand], nums[hi] = nums[hi], nums[rand]
        pivot = nums[hi]
        store = lo
        for i in range(lo, hi):
            if nums[i] <= pivot:
                nums[store], nums[i] = nums[i], nums[store]
                store += 1
        nums[store], nums[hi] = nums[hi], nums[store]
        return store

    lo, hi = 0, len(nums) - 1
    while lo < hi:
        p = partition(lo, hi)
        if p == target_idx:
            return nums[p]
        elif p < target_idx:
            lo = p + 1
        else:
            hi = p - 1

    return nums[lo]
```

**Complexity:** Time O(n) average / O(n²) worst, Space O(1).

> **Interview Tip:** Present both approaches. The heap solution is
> **consistent** O(n log k). Quick Select is **faster on average** but has a
> worst case. Ask the interviewer which trade-off they prefer. In Python,
> `heapq.nlargest(k, nums)[k-1]` is the pragmatic one-liner — mention it last.

---

### Problem 4: Top K Frequent Elements

**LeetCode 347**

#### Problem Statement

Given an integer array `nums` and integer `k`, return the `k` most frequent
elements. The answer is guaranteed to be unique.

#### Approach

1. Count frequencies with a dictionary.
2. Use a **min-heap of size K** keyed by frequency.
3. The heap retains the K elements with the highest frequencies.

#### Code

```python
from collections import Counter
import heapq


def top_k_frequent(nums: list[int], k: int) -> list[int]:
    """Return the k most frequent elements using a min-heap."""
    counts = Counter(nums)

    # Min-heap of (frequency, element). Keeps k largest frequencies.
    heap: list[tuple[int, int]] = []

    for num, freq in counts.items():
        if len(heap) < k:
            heapq.heappush(heap, (freq, num))
        elif freq > heap[0][0]:
            heapq.heapreplace(heap, (freq, num))

    return [num for _, num in heap]
```

#### Alternative: Bucket Sort — O(n)

```python
def top_k_frequent_bucket(nums: list[int], k: int) -> list[int]:
    """Top K frequent via bucket sort — O(n) time."""
    counts = Counter(nums)

    # Bucket index = frequency, bucket value = list of nums with that freq.
    buckets: list[list[int]] = [[] for _ in range(len(nums) + 1)]
    for num, freq in counts.items():
        buckets[freq].append(num)

    result: list[int] = []
    for freq in range(len(buckets) - 1, 0, -1):
        for num in buckets[freq]:
            result.append(num)
            if len(result) == k:
                return result

    return result
```

#### Complexity

| Approach | Time | Space |
|---|---|---|
| Heap | O(n log k) | O(n + k) |
| Bucket Sort | O(n) | O(n) |

> **Interview Tip:** The bucket sort approach is O(n) and very elegant. It's a
> great "optimization follow-up" that impresses interviewers. Explain: "The
> maximum possible frequency is `n`, so I can use frequency as an array index."

---

### Problem 5: Merge K Sorted Lists

**LeetCode 23**

#### Problem Statement

You are given an array of `k` linked lists, each sorted in ascending order.
Merge all the linked lists into **one sorted linked list**.

#### Approach

Use a **min-heap** to always pick the smallest current head among all K lists.
Push the next node from that list back into the heap.

#### Code

```python
from __future__ import annotations
import heapq
from dataclasses import dataclass


@dataclass
class ListNode:
    val: int = 0
    next: ListNode | None = None

    # Required for heap comparison when values are equal.
    def __lt__(self, other: ListNode) -> bool:
        return self.val < other.val


def merge_k_lists(lists: list[ListNode | None]) -> ListNode | None:
    """Merge k sorted linked lists using a min-heap."""
    heap: list[ListNode] = []

    # Seed the heap with the head of each non-empty list.
    for head in lists:
        if head:
            heapq.heappush(heap, head)

    dummy = ListNode()
    current = dummy

    while heap:
        smallest = heapq.heappop(heap)
        current.next = smallest
        current = current.next

        if smallest.next:
            heapq.heappush(heap, smallest.next)

    return dummy.next
```

#### Complexity

| | Value |
|---|---|
| Time | O(N log k) — N = total nodes across all lists |
| Space | O(k) — heap holds at most k nodes |

> **Interview Tip:** The alternative is a divide-and-conquer approach — merge
> lists in pairs, like merge sort. Same O(N log k) time but O(1) extra space
> (if done in-place). Mention both to show breadth. Note the `__lt__` method on
> `ListNode` — without it, Python's `heapq` can't break ties and will raise a
> `TypeError`.

---

### Problem 6: Find Median from Data Stream

**LeetCode 295**

#### Problem Statement

Design a data structure that supports:

- `add_num(num)` — add an integer from the data stream.
- `find_median()` — return the median of all elements so far.

#### Approach: Two Heaps

Maintain two heaps:

- **`max_heap`** (left half) — stores the smaller half of numbers. We want the
  **largest** of the small half, so use a max-heap (negate values in Python).
- **`min_heap`** (right half) — stores the larger half. We want the **smallest**
  of the large half, so use a min-heap.

**Invariants:**

1. `max_heap` size is either equal to or exactly one more than `min_heap` size.
2. Every element in `max_heap` ≤ every element in `min_heap`.

The median is either `max_heap[0]` (odd total) or the average of both roots
(even total).

#### Code

```python
import heapq


class MedianFinder:
    """Find median from a data stream using two heaps."""

    def __init__(self) -> None:
        self.lo: list[int] = []   # Max-heap (negated values) — small half
        self.hi: list[int] = []   # Min-heap — large half

    def add_num(self, num: int) -> None:
        # Always push to max-heap first (negate for max-heap behavior).
        heapq.heappush(self.lo, -num)

        # Ensure max of lo <= min of hi.
        if self.hi and -self.lo[0] > self.hi[0]:
            val = -heapq.heappop(self.lo)
            heapq.heappush(self.hi, val)

        # Balance sizes: lo can have at most 1 more element than hi.
        if len(self.lo) > len(self.hi) + 1:
            val = -heapq.heappop(self.lo)
            heapq.heappush(self.hi, val)
        elif len(self.hi) > len(self.lo):
            val = heapq.heappop(self.hi)
            heapq.heappush(self.lo, -val)

    def find_median(self) -> float:
        if len(self.lo) > len(self.hi):
            return -self.lo[0]
        return (-self.lo[0] + self.hi[0]) / 2.0
```

#### Walkthrough

Adding `[2, 3, 1, 5, 4]` step by step:

| Step | Add | lo (max-heap) | hi (min-heap) | Median |
|---|---|---|---|---|
| 1 | 2 | [2] | [] | 2.0 |
| 2 | 3 | [2] | [3] | 2.5 |
| 3 | 1 | [2, 1] | [3] | 2.0 |
| 4 | 5 | [2, 1] | [3, 5] | 2.5 |
| 5 | 4 | [3, 2, 1] | [4, 5] | 3.0 |

#### Complexity

| Operation | Time | Space |
|---|---|---|
| `add_num` | O(log n) | O(n) total |
| `find_median` | O(1) | |

> **Interview Tip:** This is a **design** problem — explain the invariants
> before writing code. Draw the two heaps as triangles pointing at each other:
> `▽ max_heap | min_heap △`. The roots face each other and the median lives at
> the boundary. If the interviewer asks for a follow-up with **remove**
> operations, mention lazy deletion with a hash map of pending removals.

---

## Sorting Algorithm Comparison Table

| Algorithm | Best | Average | Worst | Space | Stable | When to Use |
|---|---|---|---|---|---|---|
| **Merge Sort** | O(n log n) | O(n log n) | O(n log n) | O(n) | ✅ | Need stability or guaranteed perf |
| **Quick Sort** | O(n log n) | O(n log n) | O(n²) | O(log n) | ❌ | General-purpose, in-place needed |
| **Heap Sort** | O(n log n) | O(n log n) | O(n log n) | O(1) | ❌ | Guaranteed O(n log n) + O(1) space |
| **Tim Sort** | O(n) | O(n log n) | O(n log n) | O(n) | ✅ | Python's built-in `sorted()` — use it! |
| **Counting Sort** | O(n + k) | O(n + k) | O(n + k) | O(k) | ✅ | Small integer range k |
| **Radix Sort** | O(d·n) | O(d·n) | O(d·n) | O(n + k) | ✅ | Fixed-width integers/strings |
| **Bucket Sort** | O(n + k) | O(n + k) | O(n²) | O(n) | ✅ | Uniformly distributed floats |
| **Quick Select** | O(n) | O(n) | O(n²) | O(1) | — | Kth element (no full sort) |

### Key Takeaways

- **Python's `list.sort()` and `sorted()`** use **Tim Sort** — a hybrid of
  merge sort and insertion sort. It's O(n) on nearly-sorted data. In interviews,
  say "I'd use Python's built-in sort which is O(n log n)" before implementing
  a custom sort.
- **Stability** matters when you sort by multiple keys (e.g., sort by name, then
  by age — stable sort preserves the name order within same-age groups).
- **Quick Sort** beats Merge Sort in practice due to **cache locality** (in-place
  array operations) despite the same Big-O.

---

## When to Use a Heap — Decision Guide

Use this flowchart when you see a new problem:

```
Do you need repeated access to the min OR max?
├── YES → Is the dataset dynamic (elements added/removed)?
│   ├── YES → Use a HEAP
│   │   ├── Need min? → Min-heap (heapq default)
│   │   ├── Need max? → Max-heap (negate values)
│   │   └── Need both min AND max? → TWO HEAPS (median pattern)
│   └── NO (static dataset) → Just sort once, O(n log n)
├── Need top-K elements?
│   ├── K is small relative to N → Heap of size K, O(n log k)
│   ├── K is close to N → Just sort, O(n log n)
│   └── Need exact Kth element only → Quick Select, O(n) avg
└── Merging K sorted sequences?
    └── Min-heap of size K → O(N log k) total
```

### Common Heap Interview Problems

| Problem | Heap Type | Size |
|---|---|---|
| Kth Largest Element | Min-heap | K |
| Top K Frequent | Min-heap | K |
| Merge K Sorted Lists | Min-heap | K |
| Find Median (stream) | Two heaps | N/2 each |
| Task Scheduler | Max-heap | variable |
| Reorganize String | Max-heap | variable |
| Dijkstra's Shortest Path | Min-heap | V |
| Meeting Rooms II (min rooms) | Min-heap | variable |

---

## Summary

| Problem | Technique | Time | Space |
|---|---|---|---|
| Binary Search | Two pointers on sorted array | O(log n) | O(1) |
| Search Rotated Array | Modified binary search | O(log n) | O(1) |
| Kth Largest (heap) | Min-heap of size K | O(n log k) | O(k) |
| Kth Largest (Quick Select) | Randomized partition | O(n) avg | O(1) |
| Top K Frequent | Counter + min-heap | O(n log k) | O(n) |
| Merge K Sorted Lists | Min-heap of K heads | O(N log k) | O(k) |
| Find Median (stream) | Two heaps (max + min) | O(log n) add | O(n) |

### The Three Pillars of This Module

1. **Binary Search** reduces O(n) to O(log n) — always ask "Is the search space
   monotonic?" If yes, binary search on the answer.
2. **Divide and Conquer** (Merge Sort, Quick Sort) breaks big problems into
   independent subproblems — the foundation of O(n log n) sorting.
3. **Heaps** give O(log n) access to extremes — whenever you hear "top K,"
   "smallest K," "median," or "merge sorted," reach for `heapq`.

Master these and you'll crush a huge percentage of interview questions. Next
up — Graphs and BFS/DFS, where we bring all these tools together on more
complex structures. Let's go! 🚀
