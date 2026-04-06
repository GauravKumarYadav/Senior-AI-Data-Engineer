# Module 2: Hash Maps & Sets

> **Goal:** Understand hash maps at the implementation level, master the patterns that depend on O(1) lookup, and build a production-quality LRU Cache from scratch. Hash maps are the single most frequently used data structure in coding interviews — if you master nothing else, master this.

---

## 1. Hash Map Internals

### How Hashing Works

A hash map stores key-value pairs in an underlying array. To decide *where* a key goes:

1. Compute `hash(key)` → an integer.
2. Reduce it to a valid index: `index = hash(key) % array_size`.
3. Store the key-value pair at that index.

Lookup follows the same path: hash the key, go to that index, return the value. This is O(1) on average because computing a hash and indexing into an array are both constant-time operations.

### Collision Handling

Two keys can map to the same index (a **collision**). Two strategies:

**Chaining (Separate Chaining):**
Each array slot holds a linked list (or another collection). Colliding keys are appended to the list. Lookup scans the list at the target index.

```
Index 0: → (key_a, val_a) → (key_d, val_d)
Index 1: → (key_b, val_b)
Index 2: → (key_c, val_c)
```

- **Average:** O(1) per operation (if load factor is low).
- **Worst case:** O(n) if all keys hash to the same slot.

**Open Addressing (Linear Probing):**
If the target slot is occupied, probe the next slot (or use quadratic probing / double hashing). All entries live directly in the array.

- Better cache locality (no pointer chasing).
- Degrades badly at high load factors (clustering).

### Python's `dict` Implementation

Python uses **open addressing** with a compact hash table. Key details:

- Hash table resizes when it's about 2/3 full (load factor ≈ 0.66).
- Since Python 3.7, `dict` **preserves insertion order** (guaranteed by spec since 3.7, implemented since 3.6).
- Python uses **perturbation** to spread out clusters — not pure linear probing.
- Average-case O(1) for `get`, `set`, `del`, and `in`.
- `dict` in CPython uses a **compact layout** since Python 3.6: a dense array of `(hash, key, value)` tuples for iteration order, and a sparse hash table of indices into that array. This makes iteration faster and reduces memory by ~25%.

### Memory Usage

A Python `dict` with n entries uses roughly `72n + overhead` bytes (CPython 3.11+). Compare:
- `dict` with string keys: ~100 bytes per entry (key object + value object + hash table slot).
- Fixed-size `list` of 26 ints: ~280 bytes total (far less for small alphabets).

This is why using `[0] * 26` for character frequency is not premature optimization — it's a 10× memory reduction for single-character keys.

### When O(1) Becomes O(n)

If a malicious actor crafts keys that all hash to the same bucket, every operation degrades to O(n). Python mitigates this with **hash randomization** (`PYTHONHASHSEED`) since Python 3.3. In interviews, always state "O(1) average, O(n) worst case" to show you understand the nuance.

### Sets vs Dicts

A `set` is essentially a `dict` without values — same hash table, same O(1) lookup. Use a `set` when you only need membership testing ("have I seen this before?"). It's clearer in intent and slightly more memory-efficient.

---

## 2. Frequency Counting

Frequency counting is the bread and butter of hashmap usage. Python's `collections.Counter` makes it trivial:

```python
from collections import Counter

freq = Counter("hello")  # Counter({'l': 2, 'h': 1, 'e': 1, 'o': 1})
freq.most_common(2)       # [('l', 2), ('h', 1)]
```

### When to Use a Fixed-Size Array Instead

If your key space is small and known (e.g., lowercase English letters), a list of size 26 is faster and uses less memory:

```python
freq = [0] * 26
for c in s:
    freq[ord(c) - ord('a')] += 1
```

This avoids the overhead of hashing entirely. Use this in performance-critical paths or when explicitly asked about optimization.

### Top-K Pattern: Heap + HashMap

For "top K frequent elements," the classic approach is:

1. Count frequencies with a hashmap → O(n).
2. Use a min-heap of size K to find the top K → O(n log k).

Total: **O(n log k)**, better than sorting O(n log n) when k ≪ n.

Alternatively, **bucket sort** achieves O(n): create buckets indexed by frequency (max frequency = n), then collect elements from the highest bucket down.

---

## 3. Two Sum Pattern

The Two Sum pattern is arguably the most important single pattern in interviews. It embodies the core idea of hashmaps: **trade space for time** by pre-storing complements.

### The Core Idea

Instead of checking every pair (O(n²)), iterate once and ask: "Have I already seen the number that *complements* the current one to reach the target?"

```python
for each number:
    complement = target - number
    if complement in seen:
        return answer
    seen[number] = index
```

This reduces O(n²) to O(n) by using O(n) extra space. The hashmap acts as a "memory" of what we've seen so far.

### Why One Pass Works

A common question: "Don't you need two passes — one to build the map, one to query?" No! When you reach the *second* number of a valid pair, the *first* is already in the hashmap. You can't double-count because you check *before* inserting the current number. This subtlety is worth mentioning in an interview.

### Variations

| Problem | Key Change |
|---|---|
| Two Sum (unsorted) | HashMap of complement → index |
| Two Sum II (sorted) | Two pointers instead of HashMap |
| 3Sum | Sort + fix one element + two pointers on rest |
| 4Sum | Sort + fix two elements + two pointers on rest |
| Two Sum — count pairs | HashMap of complement → frequency count |
| Two Sum — all pairs | HashMap of complement → list of indices |
| Two Sum — closest | Sort + two pointers, track min diff |

The escalation from 2Sum → 3Sum → 4Sum adds one loop per level. 3Sum is O(n²), 4Sum is O(n³). Beyond that, use a hashmap of pair sums for O(n²) generalized k-sum.

---

## 4. Grouping Problems

### Group Anagrams

To group words that are anagrams of each other, you need a **canonical key** that's the same for all anagrams. Two approaches:

1. **Sorted string key:** `sorted("eat")` → `"aet"`. All anagrams sort to the same string.
2. **Frequency tuple key:** Count character frequencies and use the tuple as a key.

The sorted approach is simpler and fast enough for most cases — O(n · k log k) where k is the max word length.

### General Grouping Pattern

```python
from collections import defaultdict

groups: dict[KeyType, list[ValueType]] = defaultdict(list)
for item in items:
    key = compute_group_key(item)
    groups[key].append(item)
return list(groups.values())
```

This pattern applies any time you need to bucket items by a shared property: group files by extension, group employees by department, group logs by error code.

---

## 5. Sliding Window + HashMap

Many sliding window problems use a hashmap to track window state (character frequencies, element counts). The combination is powerful:

- **HashMap** tells you what's in the current window.
- **Sliding window** efficiently adds/removes elements at the edges.

### Template for "Minimum Window" Problems

```python
from collections import Counter

def min_window_template(s: str, target_freq: Counter) -> str:
    window_freq: dict[str, int] = {}
    have, need = 0, len(target_freq)
    left = 0
    result = ""

    for right, char in enumerate(s):
        # Expand: add char to window
        window_freq[char] = window_freq.get(char, 0) + 1
        if char in target_freq and window_freq[char] == target_freq[char]:
            have += 1

        # Shrink: try to minimize while window is valid
        while have == need:
            # Update result
            window_size = right - left + 1
            if not result or window_size < len(result):
                result = s[left:right + 1]

            # Remove leftmost character
            left_char = s[left]
            window_freq[left_char] -= 1
            if left_char in target_freq and window_freq[left_char] < target_freq[left_char]:
                have -= 1
            left += 1

    return result
```

---

## 6. LRU Cache Design

The LRU (Least Recently Used) Cache is a *design* problem, not just an algorithm problem. It tests:

- Data structure composition (hashmap + doubly-linked list).
- O(1) constraints for both `get` and `put`.
- Edge case handling (capacity limits, key updates).
- Your ability to design clean APIs and think about system-level concerns.

This problem appears at Google, Meta, Amazon, and Microsoft with high frequency. It's also practically relevant — LRU caches are used in operating systems (page replacement), databases (buffer pools), and web applications (session caches).

### Why a HashMap Alone Isn't Enough

A hashmap gives O(1) lookup by key, but doesn't track *access order*. You need to know which key was used least recently to evict it. A hashmap has no concept of "oldest" or "most recent."

### Why a Linked List Alone Isn't Enough

A linked list tracks order, but finding a node by key is O(n) — you'd have to traverse the entire list. You need both structures working together: the hashmap for O(1) key lookup, the linked list for O(1) order maintenance.

### The Architecture

```
HashMap: key → Node (pointer to doubly-linked list node)
Doubly-Linked List: head ↔ node ↔ node ↔ ... ↔ tail
                    (most recent)              (least recent)
```

- **`get(key)`:** Look up node via hashmap (O(1)), move it to the head of the list (O(1)).
- **`put(key, value)`:** If key exists, update and move to head. If new and at capacity, remove the tail node (LRU) from both the list and the hashmap, then insert the new node at the head.

### Sentinel Nodes

The dummy head and tail nodes are a critical design pattern. Without them, you'd need special cases for:
- Inserting into an empty list.
- Removing the only node.
- Removing the head or tail.

With sentinels, every real node always has a valid `prev` and `next`, eliminating all these edge cases. This pattern appears frequently in linked list problems — learn it once, use it everywhere.

---

## Problem Solutions

---

### Problem 1: Two Sum (Unsorted — HashMap)

**Problem:** Given an unsorted array `nums` and a `target`, return the indices of the two numbers that add up to `target`. Each input has exactly one solution, and you may not use the same element twice.

**Approach:**
Iterate through the array. For each number, compute `complement = target - num`. If the complement is already in our hashmap, we found our pair. Otherwise, store `num → index` for future lookups.

```python
def two_sum(nums: list[int], target: int) -> list[int]:
    """
    Find two indices whose values sum to target.
    Uses a hashmap for O(1) complement lookup.

    Time:  O(n) — single pass, each lookup is O(1) average.
    Space: O(n) — hashmap stores up to n entries.
    """
    seen: dict[int, int] = {}  # value → index

    for i, num in enumerate(nums):
        complement = target - num
        if complement in seen:
            return [seen[complement], i]
        seen[num] = i

    return []  # problem guarantees a solution exists
```

**Why one pass works:**
When we reach the *second* number of the pair, the *first* number is already in the hashmap. We don't need two passes.

> **Interview Tip:** This is often the first interview question. Nail it fast and clean to build momentum. Common follow-ups: "What if there are multiple valid pairs?" (return all), "What if the array is sorted?" (use two pointers — Module 1), "Can you do it without extra space?" (sort + two pointers, but you lose original indices).

---

### Problem 2: Group Anagrams

**Problem:** Given a list of strings, group the anagrams together. Return the groups in any order.

**Approach:**
Sort each string to create a canonical key. All anagrams produce the same sorted key. Use a hashmap of `sorted_key → list of original strings`.

```python
from collections import defaultdict


def group_anagrams(strs: list[str]) -> list[list[str]]:
    """
    Group strings that are anagrams of each other.

    Time:  O(n * k log k) — n strings, each of max length k (sorting).
    Space: O(n * k) — storing all strings in the hashmap.
    """
    groups: dict[str, list[str]] = defaultdict(list)

    for s in strs:
        # Canonical form: sorted characters
        key = "".join(sorted(s))
        groups[key].append(s)

    return list(groups.values())
```

**Alternative — O(n · k) with frequency tuple key:**

```python
def group_anagrams_optimal(strs: list[str]) -> list[list[str]]:
    """
    Uses character frequency as the key to avoid sorting.

    Time:  O(n * k) — counting is linear in word length.
    Space: O(n * k).
    """
    groups: dict[tuple[int, ...], list[str]] = defaultdict(list)

    for s in strs:
        freq = [0] * 26
        for c in s:
            freq[ord(c) - ord('a')] += 1
        groups[tuple(freq)].append(s)

    return list(groups.values())
```

> **Interview Tip:** Start with the sorted-key approach — it's cleaner and easier to explain. Mention the frequency-tuple optimization as a follow-up if the interviewer asks about improving time complexity. The sorted approach is O(n · k log k) vs O(n · k), but for typical word lengths (k ≤ 20), the practical difference is negligible.

---

### Problem 3: Top K Frequent Elements

**Problem:** Given an integer array `nums` and an integer `k`, return the `k` most frequent elements. You may return the answer in any order.

**Approach:**
1. Count frequencies with `Counter` — O(n).
2. Use a min-heap of size k to find the top k — O(n log k).

```python
import heapq
from collections import Counter


def top_k_frequent(nums: list[int], k: int) -> list[int]:
    """
    Find the k most frequent elements.

    Time:  O(n log k) — heap operations are O(log k), done n times.
    Space: O(n) — for the frequency map.
    """
    freq = Counter(nums)

    # nlargest uses a min-heap of size k internally
    return [item for item, count in heapq.nlargest(k, freq.items(), key=lambda x: x[1])]
```

**O(n) Bucket Sort approach:**

```python
def top_k_frequent_bucket(nums: list[int], k: int) -> list[int]:
    """
    Bucket sort approach — O(n) time.

    Create buckets indexed by frequency. The max frequency is len(nums),
    so we allocate len(nums)+1 buckets. Collect from highest bucket down.
    """
    freq = Counter(nums)

    # Buckets: index = frequency, value = list of elements with that frequency
    buckets: list[list[int]] = [[] for _ in range(len(nums) + 1)]
    for num, count in freq.items():
        buckets[count].append(num)

    # Collect top k from highest frequency down
    result: list[int] = []
    for i in range(len(buckets) - 1, 0, -1):
        for num in buckets[i]:
            result.append(num)
            if len(result) == k:
                return result

    return result
```

> **Interview Tip:** The `heapq.nlargest` one-liner is Pythonic but hides the algorithm. In an interview, explain that you're using a min-heap of size k: each element is pushed (O(log k)), and if the heap exceeds size k, the smallest is popped. The bucket sort approach is O(n) and impressive as a follow-up — mention it to show depth.

---

### Problem 4: Longest Substring Without Repeating Characters (HashMap Version)

**Problem:** Given a string `s`, find the length of the longest substring without repeating characters.

**Approach:**
This is a sliding window problem where the hashmap tracks the last-seen index of each character. When a character repeats within the window, jump the left pointer past its previous occurrence.

```python
def length_of_longest_substring(s: str) -> int:
    """
    Variable sliding window with hashmap tracking character positions.

    Time:  O(n) — right pointer visits each character once; left only
           moves forward, never backward.
    Space: O(min(n, m)) — m is the size of the character set (e.g., 128 ASCII).
    """
    last_seen: dict[str, int] = {}
    left = 0
    max_len = 0

    for right, char in enumerate(s):
        if char in last_seen and last_seen[char] >= left:
            # Duplicate found inside current window — shrink past it
            left = last_seen[char] + 1

        last_seen[char] = right
        max_len = max(max_len, right - left + 1)

    return max_len
```

**Critical detail — `last_seen[char] >= left`:**
Without this check, you might move `left` backward. Example: `s = "abba"`. When we hit the second `a` at index 3, `last_seen['a'] = 0`, but `left` is already at 2 (moved when we hit the second `b`). Without the check, we'd set `left = 1`, moving it backward and breaking the window.

> **Interview Tip:** This problem appeared in Module 1 as well — intentionally. It bridges arrays and hashmaps. In Module 1 we focused on the sliding window pattern; here we emphasize the hashmap's role. Being able to explain the same problem from multiple angles shows deep understanding.

---

### Problem 5: Minimum Window Substring

**Problem:** Given strings `s` and `t`, return the minimum window in `s` that contains all characters of `t` (including duplicates). If no such window exists, return `""`.

**Approach:**
Use a sliding window with two frequency maps. Expand `right` until the window contains all of `t`'s characters, then shrink `left` to minimize. Track how many characters are "fully satisfied" to avoid comparing entire frequency maps each step.

```python
from collections import Counter


def min_window(s: str, t: str) -> str:
    """
    Find the smallest substring of s containing all characters of t.

    Time:  O(|s| + |t|) — each pointer traverses s at most once.
    Space: O(|s| + |t|) — frequency maps.
    """
    if not t or not s:
        return ""

    target_freq = Counter(t)
    window_freq: dict[str, int] = {}

    # 'have' = number of unique chars with sufficient frequency
    # 'need' = number of unique chars required
    have, need = 0, len(target_freq)
    result: tuple[int, int] = (-1, -1)
    result_len = float("inf")
    left = 0

    for right, char in enumerate(s):
        # Expand window — add right character
        window_freq[char] = window_freq.get(char, 0) + 1

        # Check if this character's requirement is now fully met
        if char in target_freq and window_freq[char] == target_freq[char]:
            have += 1

        # Shrink window — try to minimize while all requirements are met
        while have == need:
            window_size = right - left + 1
            if window_size < result_len:
                result_len = window_size
                result = (left, right)

            # Remove leftmost character
            left_char = s[left]
            window_freq[left_char] -= 1
            if left_char in target_freq and window_freq[left_char] < target_freq[left_char]:
                have -= 1
            left += 1

    left_idx, right_idx = result
    return s[left_idx:right_idx + 1] if result_len != float("inf") else ""
```

**Walk-through** with `s = "ADOBECODEBANC"`, `t = "ABC"`:

1. Expand until window `"ADOBEC"` contains A, B, C → record length 6.
2. Shrink from left: remove A → no longer valid. Stop.
3. Continue expanding... find `"CODEBA"` → length 6 (same).
4. Continue... find `"BANC"` → length 4 ← **minimum**.

**Answer:** `"BANC"`

**The `have/need` optimization:**
Comparing two frequency maps every step would be O(26) or O(|charset|). By tracking `have` (count of satisfied characters), we reduce each check to O(1). This is a common optimization in sliding-window-plus-hashmap problems.

> **Interview Tip:** This is a **hard** problem — one of the most asked hard problems at top companies. The key insight interviewers look for is the `have/need` tracking to avoid O(|charset|) comparisons. If you get stuck, start by describing the brute force (check all substrings — O(n² · m)), then explain how sliding window improves to O(n + m). Build up incrementally.

---

### Problem 6: LRU Cache (Full Design + Implementation)

**Problem:** Design a data structure that follows the Least Recently Used (LRU) cache constraints:
- `LRUCache(capacity)` — initialize with positive capacity.
- `get(key)` — return the value if key exists (and mark as recently used), otherwise return -1.
- `put(key, value)` — update or insert. If inserting exceeds capacity, evict the least recently used key.

Both operations must run in **O(1)** average time.

**Approach:**
Combine a hashmap (O(1) key lookup) with a doubly-linked list (O(1) insertion/removal/reordering). The list maintains access order: most recent at head, least recent at tail.

```python
class Node:
    """Doubly-linked list node for the LRU Cache."""
    __slots__ = ("key", "value", "prev", "next")

    def __init__(self, key: int = 0, value: int = 0) -> None:
        self.key = key
        self.value = value
        self.prev: Node | None = None
        self.next: Node | None = None


class LRUCache:
    """
    Least Recently Used Cache with O(1) get and put.

    Architecture:
        HashMap:  key → Node (for O(1) lookup)
        DLL:      head ↔ ... ↔ tail (head = most recent, tail = LRU)

    Sentinel nodes (head/tail) simplify edge cases — no null checks
    when adding/removing nodes at the boundaries.

    Time:  O(1) for both get and put (average).
    Space: O(capacity) for the cache storage.
    """

    def __init__(self, capacity: int) -> None:
        self.capacity = capacity
        self.cache: dict[int, Node] = {}

        # Sentinel nodes — simplify boundary operations
        self.head = Node()  # dummy head (most recently used side)
        self.tail = Node()  # dummy tail (least recently used side)
        self.head.next = self.tail
        self.tail.prev = self.head

    def _remove(self, node: Node) -> None:
        """Remove a node from anywhere in the doubly-linked list. O(1)."""
        prev_node = node.prev
        next_node = node.next
        if prev_node:
            prev_node.next = next_node
        if next_node:
            next_node.prev = prev_node

    def _add_to_front(self, node: Node) -> None:
        """Insert a node right after the head sentinel. O(1)."""
        node.prev = self.head
        node.next = self.head.next
        if self.head.next:
            self.head.next.prev = node
        self.head.next = node

    def _move_to_front(self, node: Node) -> None:
        """Mark a node as most recently used by moving it to the front."""
        self._remove(node)
        self._add_to_front(node)

    def get(self, key: int) -> int:
        """
        Retrieve value by key. Returns -1 if not found.
        Marks the key as recently used.
        """
        if key not in self.cache:
            return -1

        node = self.cache[key]
        self._move_to_front(node)  # mark as recently used
        return node.value

    def put(self, key: int, value: int) -> None:
        """
        Insert or update a key-value pair.
        If at capacity, evict the least recently used entry.
        """
        if key in self.cache:
            # Update existing node
            node = self.cache[key]
            node.value = value
            self._move_to_front(node)
        else:
            # Insert new node
            if len(self.cache) >= self.capacity:
                # Evict LRU (node just before tail sentinel)
                lru_node = self.tail.prev
                if lru_node and lru_node != self.head:
                    self._remove(lru_node)
                    del self.cache[lru_node.key]

            new_node = Node(key, value)
            self._add_to_front(new_node)
            self.cache[key] = new_node
```

**Usage:**

```python
cache = LRUCache(2)
cache.put(1, 1)         # cache: {1=1}
cache.put(2, 2)         # cache: {1=1, 2=2}
cache.get(1)            # returns 1, cache: {2=2, 1=1} (1 moved to front)
cache.put(3, 3)         # evicts key 2, cache: {1=1, 3=3}
cache.get(2)            # returns -1 (not found)
cache.put(4, 4)         # evicts key 1, cache: {3=3, 4=4}
cache.get(1)            # returns -1
cache.get(3)            # returns 3
cache.get(4)            # returns 4
```

**Python shortcut with `OrderedDict`:**

```python
from collections import OrderedDict


class LRUCacheSimple(OrderedDict):
    """LRU Cache using Python's OrderedDict (interview-acceptable shortcut)."""

    def __init__(self, capacity: int) -> None:
        super().__init__()
        self.capacity = capacity

    def get(self, key: int) -> int:
        if key not in self:
            return -1
        self.move_to_end(key)  # mark as recently used
        return self[key]

    def put(self, key: int, value: int) -> None:
        if key in self:
            self.move_to_end(key)
        self[key] = value
        if len(self) > self.capacity:
            self.popitem(last=False)  # remove oldest (first) item
```

> **Interview Tip:** Always implement the full doubly-linked-list version first — it shows you understand the underlying mechanics. Then mention the `OrderedDict` shortcut to show you know Python's stdlib. Many interviewers will *require* the manual implementation to test your pointer manipulation skills. The sentinel node pattern (dummy head/tail) eliminates annoying null checks — use it liberally.

---

## When to Reach for a HashMap — Decision Guide

Use this mental checklist when analyzing a problem:

### ✅ Reach for a HashMap When...

| Situation | HashMap Use |
|---|---|
| You need O(1) lookup by key | Direct key-value storage |
| You're counting frequencies | `Counter` or manual freq dict |
| You need to find complements | Store value → index (Two Sum) |
| You need to group items by property | `defaultdict(list)` |
| You need to detect duplicates | Store seen elements in a set |
| You need to cache computed results | Memoization dict |
| You're tracking window state | Sliding window + freq map |
| You need O(1) insert + delete + lookup | `dict` (Python's is ordered!) |

### ❌ Don't Use a HashMap When...

| Situation | Better Alternative |
|---|---|
| Input is sorted + need a pair | Two pointers (O(1) space) |
| You need ordering/sorting | Sorted array or BST |
| You need range queries | Segment tree or sorted container |
| Keys are small integers (0–26) | Fixed-size array (faster) |
| Memory is extremely constrained | Bit manipulation or two pointers |

### HashMap Complexity Cheat Sheet

| Operation | Average | Worst Case |
|---|---|---|
| `dict[key]` (get) | O(1) | O(n) |
| `dict[key] = val` (set) | O(1) | O(n) |
| `key in dict` (lookup) | O(1) | O(n) |
| `del dict[key]` (delete) | O(1) | O(n) |
| Iterating all items | O(n) | O(n) |
| `Counter(iterable)` | O(n) | O(n) |

### Python-Specific Tips

1. **`defaultdict` vs `dict.setdefault` vs `.get`:**
   - `defaultdict(list)` is cleanest for grouping: `groups[key].append(val)`.
   - `dict.get(key, default)` for safe access without modification.
   - `dict.setdefault(key, default)` inserts the default if missing — rarely the best choice.

2. **`Counter` arithmetic:**
   ```python
   Counter("aab") + Counter("bcc")  # Counter({'b': 2, 'a': 2, 'c': 2})
   Counter("aab") - Counter("ab")   # Counter({'a': 1})  — only positive counts
   ```

3. **Sets for membership testing:**
   If you only need to check existence (no values), use a `set` instead of a `dict`. Same O(1) lookup, less memory, clearer intent.

4. **Hashable keys:**
   Dict keys must be hashable. Lists aren't hashable — convert to tuples. For custom objects, implement `__hash__` and `__eq__`.

---

*Next up: Module 3 — Trees & Graphs, where we leave the flat world of arrays and enter hierarchical and networked data structures. The patterns shift from "iterate and track state" to "explore and backtrack."*
