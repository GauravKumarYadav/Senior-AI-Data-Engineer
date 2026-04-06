# Module 3: Trees & Graphs

> **Goal:** Master tree traversals, graph search algorithms, and the structural patterns that appear in system design and data engineering interviews. Trees and graphs are where interviews separate "can code" from "can think recursively and model real systems." By the end of this module, you'll handle any tree/graph problem with a clear framework.

---

## 1. Tree Fundamentals

### Binary Tree Terminology

```
        8          ← root (depth 0)
       / \
      3   10       ← depth 1
     / \    \
    1   6    14    ← depth 2 (leaves: 1, 6, 14)
       / \   /
      4   7 13     ← depth 3
```

| Term | Definition |
|---|---|
| **Root** | The topmost node (no parent) |
| **Leaf** | A node with no children |
| **Depth** of a node | Number of edges from root to that node |
| **Height** of a tree | Max depth of any leaf |
| **Balanced** | Height of left and right subtrees differ by ≤ 1 |
| **Complete** | Every level is full except possibly the last (filled left-to-right) |
| **Full** | Every node has 0 or 2 children |

### Binary Search Tree (BST) Property

For every node: `left subtree values < node.value < right subtree values`.

This gives us O(log n) search, insert, and delete — *if the tree is balanced*. In the worst case (inserting sorted data), a BST degenerates into a linked list with O(n) operations.

### When BSTs Degenerate

Inserting `[1, 2, 3, 4, 5]` into a BST produces:

```
1
 \
  2
   \
    3
     \
      4
       \
        5
```

This is why self-balancing trees exist: AVL trees, Red-Black trees (used in Java's `TreeMap`), and B-trees (used in databases). In Python, `sortedcontainers.SortedList` provides a balanced BST-like interface.

### The Node Class

```python
class TreeNode:
    """Standard binary tree node used throughout this module."""
    def __init__(self, val: int = 0, left: 'TreeNode | None' = None,
                 right: 'TreeNode | None' = None) -> None:
        self.val = val
        self.left = left
        self.right = right
```

---

## 2. BFS (Breadth-First Search)

### Core Idea

BFS explores nodes **level by level** using a queue (FIFO). It visits all nodes at depth `d` before visiting any node at depth `d + 1`.

### BFS Template

```python
from collections import deque

def bfs(root: TreeNode | None) -> list[list[int]]:
    if not root:
        return []

    result: list[list[int]] = []
    queue: deque[TreeNode] = deque([root])

    while queue:
        level_size = len(queue)  # nodes at current level
        level: list[int] = []

        for _ in range(level_size):
            node = queue.popleft()
            level.append(node.val)

            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)

        result.append(level)

    return result
```

### When to Use BFS

- **Level-order traversal** — process nodes level by level.
- **Shortest path in unweighted graph** — BFS finds it naturally since it explores in order of distance.
- **Minimum depth of tree** — BFS hits the first leaf at the shallowest depth.
- **Serialization** — level-order is often the most natural tree serialization format.

### BFS for Graphs

For graphs (not trees), add a `visited` set to avoid infinite loops in cycles:

```python
def bfs_graph(start: str, graph: dict[str, list[str]]) -> list[str]:
    visited: set[str] = {start}
    queue: deque[str] = deque([start])
    order: list[str] = []

    while queue:
        node = queue.popleft()
        order.append(node)
        for neighbor in graph[node]:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(neighbor)

    return order
```

---

## 3. DFS (Depth-First Search)

### Core Idea

DFS explores as **deep** as possible along each branch before backtracking. It uses a stack (explicit or implicit via recursion).

### Three Traversal Orders

For the tree:
```
    1
   / \
  2   3
 / \
4   5
```

| Order | Sequence | Mnemonic | Use Case |
|---|---|---|---|
| **Pre-order** | 1, 2, 4, 5, 3 | Node → Left → Right | Copying/serializing a tree |
| **In-order** | 4, 2, 5, 1, 3 | Left → Node → Right | BST gives sorted order |
| **Post-order** | 4, 5, 2, 3, 1 | Left → Right → Node | Deleting a tree, evaluating expressions |

### Recursive Implementations

```python
def preorder(root: TreeNode | None) -> list[int]:
    if not root:
        return []
    return [root.val] + preorder(root.left) + preorder(root.right)

def inorder(root: TreeNode | None) -> list[int]:
    if not root:
        return []
    return inorder(root.left) + [root.val] + inorder(root.right)

def postorder(root: TreeNode | None) -> list[int]:
    if not root:
        return []
    return postorder(root.left) + postorder(root.right) + [root.val]
```

### Iterative DFS (Pre-order with Explicit Stack)

```python
def preorder_iterative(root: TreeNode | None) -> list[int]:
    if not root:
        return []

    result: list[int] = []
    stack: list[TreeNode] = [root]

    while stack:
        node = stack.pop()
        result.append(node.val)
        # Push right first so left is processed first (LIFO)
        if node.right:
            stack.append(node.right)
        if node.left:
            stack.append(node.left)

    return result
```

### When to Use DFS

- **Path-related problems** — "find all paths from root to leaf," "path sum equals K."
- **Tree validation** — validate BST (in-order must be sorted).
- **Tree construction** — build tree from traversal sequences.
- **Backtracking** — DFS is the backbone of all backtracking algorithms.
- **Connected components** — DFS from each unvisited node in a graph.

---

## 4. Topological Sort

### What It Is

A **topological ordering** of a Directed Acyclic Graph (DAG) is a linear ordering of vertices such that for every edge `u → v`, vertex `u` comes before `v`. It answers: "In what order should I process tasks if some tasks depend on others?"

### Real-World Connection: Airflow DAGs 🔥

If you're interviewing for data engineering roles, this connection is **gold**:

```
extract_data → transform_data → load_to_warehouse → generate_report
                                                   → send_notification
```

Each Airflow DAG is literally a directed acyclic graph. The scheduler performs a topological sort to determine task execution order. Tasks with no unfinished dependencies run first. This is *exactly* Kahn's algorithm.

**In an interview, say this:** "Topological sort is how Airflow determines task execution order. Each task's dependencies must complete before it runs, and Kahn's algorithm naturally handles this by processing tasks whose in-degree (number of unfinished dependencies) reaches zero."

### Kahn's Algorithm (BFS-Based)

1. Compute **in-degree** (number of incoming edges) for each node.
2. Add all nodes with in-degree 0 to a queue (they have no dependencies).
3. Process the queue: for each node, reduce the in-degree of its neighbors. If any neighbor's in-degree becomes 0, add it to the queue.
4. If you process all nodes, the order is a valid topological sort. If some nodes remain (in-degree > 0), the graph has a **cycle**.

```python
from collections import deque


def topological_sort_kahn(
    num_nodes: int,
    edges: list[tuple[int, int]],
) -> list[int]:
    """
    Kahn's algorithm — BFS-based topological sort.
    edges: list of (prerequisite, dependent) pairs.

    Returns topological order, or empty list if cycle detected.

    Time:  O(V + E) — visit every vertex and edge once.
    Space: O(V + E) — adjacency list + in-degree array + queue.
    """
    # Build adjacency list and in-degree count
    graph: dict[int, list[int]] = {i: [] for i in range(num_nodes)}
    in_degree = [0] * num_nodes

    for prereq, dependent in edges:
        graph[prereq].append(dependent)
        in_degree[dependent] += 1

    # Start with all nodes that have no prerequisites
    queue: deque[int] = deque()
    for node in range(num_nodes):
        if in_degree[node] == 0:
            queue.append(node)

    order: list[int] = []

    while queue:
        node = queue.popleft()
        order.append(node)

        for neighbor in graph[node]:
            in_degree[neighbor] -= 1
            if in_degree[neighbor] == 0:
                queue.append(neighbor)

    # If we didn't process all nodes, there's a cycle
    if len(order) != num_nodes:
        return []  # cycle detected!

    return order
```

### DFS-Based Topological Sort

An alternative: run DFS and add nodes to the result in **post-order** (after visiting all descendants). Reverse the result for topological order. Cycle detection: if you visit a node that's currently on the recursion stack (state = "in progress"), there's a cycle.

---

## 5. Graph Representations

### Adjacency List

```python
# Dictionary of lists
graph: dict[str, list[str]] = {
    "A": ["B", "C"],
    "B": ["D"],
    "C": ["D"],
    "D": [],
}
```

- **Space:** O(V + E)
- **Check if edge exists:** O(degree of node) — scan the neighbor list
- **Best for:** Sparse graphs (most real-world graphs), traversal algorithms

### Adjacency Matrix

```python
#     A  B  C  D
# A [[0, 1, 1, 0],
# B  [0, 0, 0, 1],
# C  [0, 0, 0, 1],
# D  [0, 0, 0, 0]]
matrix = [[0]*4 for _ in range(4)]
matrix[0][1] = 1  # edge A → B
```

- **Space:** O(V²)
- **Check if edge exists:** O(1) — direct index
- **Best for:** Dense graphs, Floyd-Warshall algorithm, small graphs

### Grid as Implicit Graph

Many interview problems use a 2D grid where each cell connects to its 4 (or 8) neighbors. You don't build an explicit graph — you traverse the grid directly:

```python
DIRECTIONS = [(0, 1), (0, -1), (1, 0), (-1, 0)]  # right, left, down, up

def get_neighbors(row: int, col: int, rows: int, cols: int) -> list[tuple[int, int]]:
    neighbors = []
    for dr, dc in DIRECTIONS:
        nr, nc = row + dr, col + dc
        if 0 <= nr < rows and 0 <= nc < cols:
            neighbors.append((nr, nc))
    return neighbors
```

---

## 6. Union-Find (Disjoint Set)

### What It Solves

Union-Find manages a collection of disjoint sets and supports two operations:
- **Find(x):** Which set does x belong to? (Returns the "root" representative.)
- **Union(x, y):** Merge the sets containing x and y.

### Optimizations

1. **Path Compression:** During `find(x)`, make every node on the path point directly to the root. This flattens the tree, making future lookups nearly O(1).
2. **Union by Rank:** When merging, attach the smaller tree under the root of the larger tree. This keeps trees balanced.

With both optimizations, each operation is **O(α(n))** — effectively O(1) (α is the inverse Ackermann function, which is ≤ 4 for any practical input size).

```python
class UnionFind:
    """
    Disjoint Set Union with path compression and union by rank.

    Time:  O(α(n)) ≈ O(1) per operation.
    Space: O(n).
    """

    def __init__(self, n: int) -> None:
        self.parent = list(range(n))  # each node is its own root
        self.rank = [0] * n
        self.components = n  # number of connected components

    def find(self, x: int) -> int:
        """Find root of x with path compression."""
        if self.parent[x] != x:
            self.parent[x] = self.find(self.parent[x])  # path compression
        return self.parent[x]

    def union(self, x: int, y: int) -> bool:
        """
        Merge sets containing x and y.
        Returns True if they were in different sets (merge happened).
        """
        root_x, root_y = self.find(x), self.find(y)
        if root_x == root_y:
            return False  # already in the same set

        # Union by rank: attach smaller tree under larger
        if self.rank[root_x] < self.rank[root_y]:
            root_x, root_y = root_y, root_x
        self.parent[root_y] = root_x
        if self.rank[root_x] == self.rank[root_y]:
            self.rank[root_x] += 1

        self.components -= 1
        return True

    def connected(self, x: int, y: int) -> bool:
        """Check if x and y are in the same set."""
        return self.find(x) == self.find(y)
```

### Use Cases

- **Connected components** in undirected graphs.
- **Cycle detection** in undirected graphs (if `union(u, v)` returns False, the edge creates a cycle).
- **Kruskal's MST** algorithm (process edges by weight, union vertices).
- **Dynamic connectivity** — efficiently answer "are A and B connected?" as edges are added.

---

## 7. Dijkstra's Algorithm (Awareness Level)

Dijkstra's finds the **shortest path** in a weighted graph with **non-negative** edge weights.

### Key Idea

Use a priority queue (min-heap). Always process the unvisited node with the smallest known distance. Relax its neighbors: if the path through the current node is shorter than their known distance, update.

```python
import heapq


def dijkstra(
    graph: dict[str, list[tuple[str, int]]],
    start: str,
) -> dict[str, int]:
    """
    Shortest path from start to all reachable nodes.
    graph: adjacency list where graph[u] = [(v, weight), ...]

    Time:  O((V + E) log V) with a binary heap.
    Space: O(V) for distances + O(V) for the heap.
    """
    distances: dict[str, int] = {start: 0}
    heap: list[tuple[int, str]] = [(0, start)]

    while heap:
        dist, node = heapq.heappop(heap)

        # Skip if we've already found a shorter path
        if dist > distances.get(node, float("inf")):
            continue

        for neighbor, weight in graph[node]:
            new_dist = dist + weight
            if new_dist < distances.get(neighbor, float("inf")):
                distances[neighbor] = new_dist
                heapq.heappush(heap, (new_dist, neighbor))

    return distances
```

> **Interview Tip:** You rarely need to implement Dijkstra's from scratch in a 45-minute interview. But knowing *when* to apply it and the general approach (priority queue + relaxation) is essential for system design and graph-heavy interviews. The key limitation: it doesn't work with negative edge weights (use Bellman-Ford instead).

---

## Problem Solutions

---

### Problem 1: Binary Tree Level Order Traversal (BFS)

**Problem:** Given the root of a binary tree, return its level-order traversal as a list of lists (each inner list contains node values at that level).

**Approach:**
Classic BFS with level tracking. Process all nodes at the current level before moving to the next. Use `len(queue)` at the start of each iteration to know how many nodes belong to the current level.

```python
from collections import deque


def level_order(root: TreeNode | None) -> list[list[int]]:
    """
    BFS level-order traversal of a binary tree.

    Time:  O(n) — visit every node exactly once.
    Space: O(n) — queue holds at most one level; widest level can be n/2.
    """
    if not root:
        return []

    result: list[list[int]] = []
    queue: deque[TreeNode] = deque([root])

    while queue:
        level_size = len(queue)
        level: list[int] = []

        for _ in range(level_size):
            node = queue.popleft()
            level.append(node.val)

            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)

        result.append(level)

    return result
```

**Variations** (common follow-ups):
- **Zigzag level order:** Alternate appending left-to-right and right-to-left per level. Use a flag to reverse alternate levels.
- **Right side view:** Only take the last element of each level.
- **Average of levels:** Compute `sum(level) / len(level)` for each level.

> **Interview Tip:** The `level_size = len(queue)` line is the crux of level-aware BFS. Without it, you can't separate nodes by level. This one line is what distinguishes "plain BFS" from "level-order BFS." Make sure you explain why it works: at the start of each iteration, the queue contains exactly the nodes of the current level.

---

### Problem 2: Maximum Depth of Binary Tree (DFS)

**Problem:** Given the root of a binary tree, return its maximum depth (number of nodes along the longest root-to-leaf path).

**Approach:**
Recursive DFS. The depth of a tree is `1 + max(depth of left subtree, depth of right subtree)`. Base case: an empty tree has depth 0.

```python
def max_depth(root: TreeNode | None) -> int:
    """
    Find the maximum depth of a binary tree.

    Time:  O(n) — visit every node once.
    Space: O(h) — recursion stack, where h is the height.
           O(log n) for balanced, O(n) for skewed.
    """
    if not root:
        return 0

    left_depth = max_depth(root.left)
    right_depth = max_depth(root.right)

    return 1 + max(left_depth, right_depth)
```

**Iterative version (BFS):**

```python
def max_depth_iterative(root: TreeNode | None) -> int:
    """Iterative BFS approach — count the number of levels."""
    if not root:
        return 0

    depth = 0
    queue: deque[TreeNode] = deque([root])

    while queue:
        depth += 1
        for _ in range(len(queue)):
            node = queue.popleft()
            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)

    return depth
```

> **Interview Tip:** This is often the first tree problem in an interview. Solve it in 2 minutes and use it to build your confidence. The recursive solution is elegant but know the iterative version for follow-up questions about stack overflow on very deep trees (e.g., 100,000 nodes in a skewed tree). Python's default recursion limit is 1,000 — mention this if asked about production constraints.

---

### Problem 3: Validate Binary Search Tree

**Problem:** Given the root of a binary tree, determine if it is a valid BST. A valid BST has: left subtree values strictly less than the node, right subtree values strictly greater, and both subtrees are also valid BSTs.

**Approach:**
Pass a valid range `(low, high)` down the tree. Each node must satisfy `low < node.val < high`. The root starts with `(-∞, +∞)`. When going left, update `high = node.val`. When going right, update `low = node.val`.

```python
def is_valid_bst(root: TreeNode | None) -> bool:
    """
    Validate that a binary tree satisfies BST properties.

    Time:  O(n) — visit every node once.
    Space: O(h) — recursion stack.
    """
    def validate(
        node: TreeNode | None,
        low: float,
        high: float,
    ) -> bool:
        if not node:
            return True  # empty tree is a valid BST

        # Current node must be within the valid range
        if not (low < node.val < high):
            return False

        # Left subtree: all values must be < node.val
        # Right subtree: all values must be > node.val
        return (
            validate(node.left, low, node.val)
            and validate(node.right, node.val, high)
        )

    return validate(root, float("-inf"), float("inf"))
```

**Common mistake:** Only checking `node.left.val < node.val` and `node.right.val > node.val`. This misses cases where a deeper node violates the BST property:

```
    5
   / \
  1   6
     / \
    3   7    ← 3 is less than 5 but in the RIGHT subtree!
```

The range-based approach catches this because `3` would be checked against `low=5`, failing `5 < 3`.

**In-order traversal approach:**

```python
def is_valid_bst_inorder(root: TreeNode | None) -> bool:
    """
    In-order traversal of a valid BST produces sorted output.
    Track the previous value and ensure strict increase.
    """
    prev = [float("-inf")]  # use list for mutability in nested function

    def inorder(node: TreeNode | None) -> bool:
        if not node:
            return True
        if not inorder(node.left):
            return False
        if node.val <= prev[0]:
            return False
        prev[0] = node.val
        return inorder(node.right)

    return inorder(root)
```

> **Interview Tip:** The range-based approach is cleaner and easier to explain under pressure. The in-order approach is clever but uses a mutable container (`prev = [float("-inf")]`) as a workaround for Python's scoping rules — this can look awkward. Use `nonlocal` if you prefer: `nonlocal prev` inside the nested function, with `prev = float("-inf")` in the outer scope.

---

### Problem 4: Number of Islands (DFS/BFS Grid)

**Problem:** Given a 2D grid of `'1'`s (land) and `'0'`s (water), count the number of islands. An island is a group of connected `'1'`s (horizontally or vertically).

**Approach:**
Iterate through every cell. When you find an unvisited `'1'`, that's a new island — run DFS/BFS to mark all connected land cells as visited. Count how many times you start a new search.

```python
def num_islands(grid: list[list[str]]) -> int:
    """
    Count connected components of '1's in a 2D grid.

    Time:  O(m * n) — visit every cell at most once.
    Space: O(m * n) — recursion stack in worst case (all land).
    """
    if not grid:
        return 0

    rows, cols = len(grid), len(grid[0])
    count = 0

    def dfs(r: int, c: int) -> None:
        """Sink the island — mark all connected land as visited."""
        # Boundary checks and water/visited check
        if r < 0 or r >= rows or c < 0 or c >= cols or grid[r][c] != '1':
            return

        grid[r][c] = '0'  # mark as visited (sink the land)

        # Explore all 4 directions
        dfs(r + 1, c)  # down
        dfs(r - 1, c)  # up
        dfs(r, c + 1)  # right
        dfs(r, c - 1)  # left

    for r in range(rows):
        for c in range(cols):
            if grid[r][c] == '1':
                count += 1
                dfs(r, c)  # sink the entire island

    return count
```

**BFS version** (avoids recursion depth issues):

```python
def num_islands_bfs(grid: list[list[str]]) -> int:
    if not grid:
        return 0

    rows, cols = len(grid), len(grid[0])
    count = 0

    for r in range(rows):
        for c in range(cols):
            if grid[r][c] == '1':
                count += 1
                # BFS to sink the island
                queue: deque[tuple[int, int]] = deque([(r, c)])
                grid[r][c] = '0'

                while queue:
                    cr, cc = queue.popleft()
                    for dr, dc in [(0, 1), (0, -1), (1, 0), (-1, 0)]:
                        nr, nc = cr + dr, cc + dc
                        if 0 <= nr < rows and 0 <= nc < cols and grid[nr][nc] == '1':
                            grid[nr][nc] = '0'
                            queue.append((nr, nc))

    return count
```

> **Interview Tip:** The trick of modifying the grid in-place (`grid[r][c] = '0'`) avoids needing a separate `visited` set, saving O(m·n) space. However, mention to your interviewer: "I'm modifying the input. If that's not allowed, I'd use a visited set instead." This shows you're thinking about side effects and API contracts — senior-level thinking.

---

### Problem 5: Course Schedule (Topological Sort — Cycle Detection)

**Problem:** There are `numCourses` courses labeled `0` to `numCourses - 1`. You're given prerequisite pairs `[a, b]` meaning you must take course `b` before course `a`. Return `True` if you can finish all courses (i.e., no circular dependencies).

**Approach:**
This is cycle detection in a directed graph. Build a dependency graph and run Kahn's algorithm (BFS topological sort). If you can process all nodes, there's no cycle. If some nodes remain (in-degree > 0), there's a cycle.

```python
from collections import deque


def can_finish(num_courses: int, prerequisites: list[list[int]]) -> bool:
    """
    Detect if course schedule has circular dependencies.
    Uses Kahn's algorithm (BFS topological sort).

    Time:  O(V + E) — V = num_courses, E = len(prerequisites).
    Space: O(V + E) — adjacency list + in-degree array.
    """
    # Build adjacency list and in-degree array
    graph: list[list[int]] = [[] for _ in range(num_courses)]
    in_degree = [0] * num_courses

    for course, prereq in prerequisites:
        graph[prereq].append(course)
        in_degree[course] += 1

    # Start with courses that have no prerequisites
    queue: deque[int] = deque()
    for i in range(num_courses):
        if in_degree[i] == 0:
            queue.append(i)

    completed = 0

    while queue:
        course = queue.popleft()
        completed += 1

        for next_course in graph[course]:
            in_degree[next_course] -= 1
            if in_degree[next_course] == 0:
                queue.append(next_course)

    # If we completed all courses, no cycle exists
    return completed == num_courses
```

**The Airflow connection:**

Think of courses as Airflow tasks and prerequisites as task dependencies. This algorithm answers: "Can this DAG actually execute, or does it have circular dependencies that would cause an infinite loop?"

In Airflow, a circular dependency like `task_A → task_B → task_C → task_A` would be caught at DAG parse time — using essentially this same algorithm. If your interviewer asks about data engineering, connecting topological sort to DAG scheduling shows you understand the theory behind the tools.

**Follow-up: Course Schedule II** (return the order):

```python
def find_order(num_courses: int, prerequisites: list[list[int]]) -> list[int]:
    """Return a valid course order, or empty list if impossible."""
    graph: list[list[int]] = [[] for _ in range(num_courses)]
    in_degree = [0] * num_courses

    for course, prereq in prerequisites:
        graph[prereq].append(course)
        in_degree[course] += 1

    queue: deque[int] = deque(
        i for i in range(num_courses) if in_degree[i] == 0
    )
    order: list[int] = []

    while queue:
        course = queue.popleft()
        order.append(course)
        for next_course in graph[course]:
            in_degree[next_course] -= 1
            if in_degree[next_course] == 0:
                queue.append(next_course)

    return order if len(order) == num_courses else []
```

> **Interview Tip:** Always state upfront: "I'll use Kahn's algorithm — BFS-based topological sort — because it naturally detects cycles." This signals you know multiple approaches (DFS-based exists too) and chose deliberately. Kahn's is generally easier to implement correctly under pressure because the DFS approach requires careful state tracking (unvisited / in-progress / completed) to detect back edges.

---

### Problem 6: Lowest Common Ancestor of a BST

**Problem:** Given a BST and two nodes `p` and `q`, find their lowest common ancestor (LCA). The LCA is the deepest node that has both `p` and `q` as descendants (a node can be its own descendant).

**Approach:**
Exploit the BST property. Starting from the root:
- If both `p` and `q` are smaller, LCA is in the left subtree.
- If both are larger, LCA is in the right subtree.
- Otherwise, the current node is the split point — it *is* the LCA.

```python
def lowest_common_ancestor(
    root: TreeNode | None,
    p: TreeNode,
    q: TreeNode,
) -> TreeNode | None:
    """
    Find the LCA of two nodes in a BST.
    Exploits BST ordering for O(h) time.

    Time:  O(h) — h is the height of the tree.
           O(log n) if balanced, O(n) if skewed.
    Space: O(1) — iterative, no recursion stack.
    """
    node = root

    while node:
        if p.val < node.val and q.val < node.val:
            node = node.left   # both are in left subtree
        elif p.val > node.val and q.val > node.val:
            node = node.right  # both are in right subtree
        else:
            # Split point: one is left, one is right (or one equals node)
            return node

    return None
```

**For a general binary tree (not BST):**

The BST property doesn't apply. Use recursive DFS:

```python
def lca_general(
    root: TreeNode | None,
    p: TreeNode,
    q: TreeNode,
) -> TreeNode | None:
    """
    LCA in a general binary tree (no BST property).

    Time:  O(n) — may need to visit all nodes.
    Space: O(h) — recursion stack.
    """
    if not root or root == p or root == q:
        return root

    left = lca_general(root.left, p, q)
    right = lca_general(root.right, p, q)

    if left and right:
        return root   # p and q are in different subtrees → root is LCA
    return left or right  # both are in the same subtree
```

> **Interview Tip:** Always ask: "Is this a BST or a general binary tree?" The BST version is O(h) and beautifully simple. The general version is O(n) and requires a different approach entirely. Clarifying this question upfront shows maturity and avoids solving the wrong problem.

---

## Tree vs Graph Problem Recognition Guide

### How to Tell What You're Dealing With

| Signal | Data Structure | First Instinct |
|---|---|---|
| "Binary tree," "root," "left/right child" | Tree | DFS (recursive) or BFS |
| "BST," "sorted," "validate" | BST | In-order traversal or range checking |
| "Level order," "minimum depth" | Tree | BFS with level tracking |
| "Grid," "islands," "connected" | Implicit graph (grid) | DFS or BFS flood fill |
| "Prerequisites," "dependencies" | DAG | Topological sort (Kahn's) |
| "Shortest path," "unweighted" | Graph | BFS |
| "Shortest path," "weighted" | Graph | Dijkstra's (non-negative) |
| "Connected components" | Undirected graph | Union-Find or DFS |
| "Cycle detection" (directed) | Directed graph | Topological sort or DFS with states |
| "Cycle detection" (undirected) | Undirected graph | Union-Find |

### DFS vs BFS Decision Framework

```
Need shortest path / minimum depth?
├── Yes → BFS (explores in order of distance)
└── No
    ├── Need to explore all paths / backtrack?
    │   └── Yes → DFS (natural for recursion + backtracking)
    ├── Need level-by-level processing?
    │   └── Yes → BFS (level-order traversal)
    ├── Tree problem with recursive structure?
    │   └── Yes → DFS (matches tree's recursive definition)
    └── Graph with very deep paths?
        └── Use BFS or iterative DFS (avoid stack overflow)
```

### Complexity Reference

| Algorithm | Time | Space | Notes |
|---|---|---|---|
| DFS (tree) | O(n) | O(h) | h = height; O(log n) balanced, O(n) skewed |
| BFS (tree) | O(n) | O(w) | w = max width; up to O(n/2) for complete tree |
| DFS (graph) | O(V + E) | O(V) | Need visited set for graphs |
| BFS (graph) | O(V + E) | O(V) | Need visited set + queue |
| Topological Sort | O(V + E) | O(V + E) | Only for DAGs |
| Union-Find | O(α(n)) per op | O(n) | Nearly O(1) with optimizations |
| Dijkstra's | O((V+E) log V) | O(V) | Non-negative weights only |

### Common Mistakes

1. **Forgetting the visited set in graph BFS/DFS** — infinite loops in cyclic graphs.
2. **Using DFS when BFS is required** — DFS does NOT guarantee shortest path.
3. **Confusing tree height vs depth** — height is from node down to leaf; depth is from root down to node.
4. **Not handling disconnected graphs** — run BFS/DFS from every unvisited node, not just one start node.
5. **Modifying input without permission** — ask before sinking islands or marking nodes visited in-place.
6. **Forgetting BST vs general tree** — always clarify. BST problems have O(h) solutions; general tree problems often require O(n).

---

*You've now covered the three foundational pillars: Arrays/Strings (Module 1), Hash Maps (Module 2), and Trees/Graphs (Module 3). These three modules cover roughly 70% of all coding interview problems. Next steps: recursion & backtracking, dynamic programming, and system design.*
