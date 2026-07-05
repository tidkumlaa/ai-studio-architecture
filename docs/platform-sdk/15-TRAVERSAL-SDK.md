---
knowledge_id: KA-SDK-016
version: "1.0.0"
status: approved
owner: Chief Platform Architect
phase: 2.0D.2.5A
created: 2026-06-29
review_date: 2026-12-29
canonical: true
domain: DOM-PLATFORM
capability: platform-sdk
type: specification
depends_on:
  - id: KA-SDK-002
    reason: "Node/edge visit sequences"
  - id: KA-SDK-004
    reason: "Graph SDK provides TraversalTarget implementations"
---

# Traversal SDK

## BFS · DFS · Topological · Post-Order · Level-Order · Visitor · Iterator

---

## 1. Purpose

The Traversal SDK abstracts all graph traversal patterns behind a common
protocol so that any graph-like structure can be traversed without coupling
to a specific graph implementation. The Graph SDK, Knowledge Graph Runtime,
AST Walker, Dependency Resolver, and Impact Analyzer all consume the Traversal
SDK. Traversal strategies are pluggable and composable.

---

## 2. Traversal Architecture

```
TraversalTarget          — any structure with neighbors
    │
    ▼
TraversalStrategy        — determines visit order
├── BFSStrategy           — breadth-first
├── DFSStrategy           — depth-first (pre/post-order variants)
├── TopologicalStrategy   — Kahn's algorithm
├── LevelOrderStrategy    — level by level
├── PostOrderStrategy     — children before parent
└── PreOrderStrategy      — parent before children
    │
    ▼
Traversal[N]             — stateful, lazy iterator over nodes
    │
    ▼
TraversalVisitor[N, T]   — applied to each visited node → produces T
```

---

## 3. Interfaces

### 3.1 TraversalTarget

```python
class TraversalTarget(Protocol[N]):
    """
    Minimum protocol for any structure that can be traversed.
    N = node type (vertex ID, AST node, task ID, etc.)
    """
    def neighbors(self, node: N) -> Sequence[N]   # O(deg)
    def predecessors(self, node: N) -> Sequence[N] # O(deg) — for reverse traversal
    def all_nodes(self) -> Sequence[N]             # O(V)
    def contains(self, node: N) -> bool            # O(1)
```

### 3.2 Traversal[N]

```python
class Traversal(Protocol[N]):
    """
    Stateful traversal iterator. Lazy — computes next node on demand.
    """
    def has_next(self) -> bool
    def next(self) -> N
    def peek(self) -> N | None
    def skip(self) -> None       # Discard next without visiting
    def reset(self, start: N | None = None) -> None
    def depth(self) -> int       # Current traversal depth
    def visited(self) -> frozenset[N]
    def path_to_current(self) -> Sequence[N]

    def __iter__(self) -> Iterator[N]      # Convenience
    def to_list(self) -> Sequence[N]       # Materialize all remaining
```

### 3.3 TraversalStrategy

```python
class TraversalStrategy(Protocol[N]):
    """Creates a Traversal instance from a target and start node."""
    def create(self, target: TraversalTarget[N],
               start: N,
               max_depth: int | None = None,
               filter_fn: Callable[[N], bool] | None = None) -> Traversal[N]
    def name(self) -> str
    def complexity(self) -> str   # e.g. "O(V+E)"

class TraversalFactory(Protocol[N]):
    """Selects strategy by name."""
    def bfs(self, target: TraversalTarget[N], start: N, ...) -> Traversal[N]
    def dfs_pre(self, target: TraversalTarget[N], start: N, ...) -> Traversal[N]
    def dfs_post(self, target: TraversalTarget[N], start: N, ...) -> Traversal[N]
    def topological(self, target: TraversalTarget[N]) -> Traversal[N]
    def level_order(self, target: TraversalTarget[N], start: N, ...) -> Traversal[N]
    def reverse_bfs(self, target: TraversalTarget[N], start: N, ...) -> Traversal[N]
```

### 3.4 BFS Strategy

```python
class BFSStrategy(TraversalStrategy[N]):
    """
    Breadth-first search.
    Uses a FIFO queue.
    Complexity: O(V + E) time, O(V) space.
    Visits nodes level by level from start.
    """
    def create(self, target, start, max_depth=None, filter_fn=None) -> Traversal[N]
    # depth() returns BFS level of current node
```

### 3.5 DFS Strategies

```python
class DFSPreOrderStrategy(TraversalStrategy[N]):
    """
    Depth-first, pre-order: parent before children.
    Uses an explicit stack.
    Complexity: O(V + E) time, O(V) space.
    """

class DFSPostOrderStrategy(TraversalStrategy[N]):
    """
    Depth-first, post-order: children before parent.
    Used by: compiler IR emission, AST evaluation.
    Complexity: O(V + E) time, O(V) space.
    """
    # Requires all children fully visited before yielding parent
```

### 3.6 Topological Strategy

```python
class TopologicalStrategy(TraversalStrategy[N]):
    """
    Topological ordering using Kahn's algorithm.
    Only valid on DAGs — raises CycleError if cycle detected.
    Complexity: O(V + E) time, O(V) space.
    Produces: nodes in order such that all predecessors appear before successors.
    """
    def create(self, target: TraversalTarget[N],
               start: N | None = None,  # None = all roots
               ...) -> Traversal[N]
    def is_dag(self, target: TraversalTarget[N]) -> bool
```

### 3.7 TraversalVisitor

```python
class TraversalVisitor(Protocol[N, T]):
    """Applied to each node during traversal. Accumulates result."""
    def visit(self, node: N, depth: int,
              parent: N | None) -> T | None
    def result(self) -> Sequence[T]

class CollectVisitor(TraversalVisitor[N, N]):
    """Collects all visited nodes into a list."""
    def result(self) -> Sequence[N]

class FilterVisitor(TraversalVisitor[N, N]):
    """Keeps only nodes matching predicate."""
    def __init__(self, predicate: Callable[[N], bool])

class TransformVisitor(TraversalVisitor[N, T]):
    """Applies transformation fn to each node."""
    def __init__(self, fn: Callable[[N], T])

class AggregateVisitor(TraversalVisitor[N, T]):
    """Folds visited nodes into a single accumulated result."""
    def __init__(self, initial: T, fn: Callable[[T, N], T])
    def result(self) -> T   # Returns single accumulated value
```

### 3.8 Traversal Combinators

```python
class TraversalCombinator(Protocol[N]):
    """Compose multiple traversals."""
    def union(self, a: Traversal[N], b: Traversal[N]) -> Traversal[N]
        # All nodes from A, then remaining from B
    def intersect(self, a: Traversal[N], b: Traversal[N]) -> Traversal[N]
        # Only nodes that appear in both A and B
    def difference(self, a: Traversal[N], b: Traversal[N]) -> Traversal[N]
        # Nodes in A that are NOT in B
    def limit(self, traversal: Traversal[N], n: int) -> Traversal[N]
    def skip_first(self, traversal: Traversal[N], n: int) -> Traversal[N]
    def filter(self, traversal: Traversal[N],
               pred: Callable[[N], bool]) -> Traversal[N]
```

### 3.9 Traversal Result Models

```python
@dataclass(frozen=True)
class TraversalEvent(Generic[N]):
    node:    N
    depth:   int
    parent:  N | None
    kind:    str   # "enter" | "exit" | "back_edge" | "cross_edge"

@dataclass(frozen=True)
class TraversalPath(Generic[N]):
    nodes:    Sequence[N]
    length:   int
    is_cycle: bool

@dataclass
class TraversalResult(Generic[N]):
    visited_count: int
    max_depth:     int
    has_cycles:    bool
    connected_components: int
    paths:         Sequence[TraversalPath[N]]  # Populated for path-finding traversals
```

---

## 4. Contracts

| ID | Contract |
|----|----------|
| C-TRV-001 | BFS yields nodes in non-decreasing depth order. |
| C-TRV-002 | DFS post-order yields all children of N before N itself. |
| C-TRV-003 | `Topological.create` raises `CycleError` if target has a cycle. |
| C-TRV-004 | Each node is yielded at most once per traversal (no revisits). |
| C-TRV-005 | `visited()` grows monotonically — once added, a node is never removed. |
| C-TRV-006 | `filter_fn` returning False excludes a node AND all nodes reachable only through it. |
| C-TRV-007 | `max_depth=0` traversal yields only the start node. |
| C-TRV-008 | `reset(start)` restores traversal to initial state from new start. |
| C-TRV-009 | `TraversalCombinator.union` yields each node at most once. |
| C-TRV-010 | `AggregateVisitor.result()` after empty traversal returns `initial`. |

---

## 5. Complexity Table

| Strategy | Time | Space | Notes |
|----------|------|-------|-------|
| BFS | O(V+E) | O(V) | FIFO queue |
| DFS pre/post | O(V+E) | O(V) | Explicit stack |
| Topological (Kahn's) | O(V+E) | O(V) | In-degree counts |
| Level-order | O(V+E) | O(W) | W = max width |
| Reverse BFS | O(V+E) | O(V) | On reversed graph |
| Visitor application | O(V) | O(1) | Per visit |
| Combinator union | O(V+E) | O(V) | Union of visited sets |

---

## 6. Dependencies

- Collections SDK (KA-SDK-002) — visited sets, queue, stack
- Graph SDK (KA-SDK-004) — primary TraversalTarget implementor

---

## 7. Extension Points

```python
class TraversalPlugin(Protocol[N]):
    """Contributes a new traversal strategy."""
    def strategy_name(self) -> str: ...
    def create_strategy(self) -> TraversalStrategy[N]: ...

class TraversalListener(Protocol[N]):
    """Observe traversal events without modifying traversal."""
    def on_visit(self, event: TraversalEvent[N]) -> None: ...
    def on_complete(self, result: TraversalResult[N]) -> None: ...

class WeightedTraversalStrategy(TraversalStrategy[N]):
    """Traversal ordered by edge weights (e.g. Dijkstra-style)."""
    def edge_weight(self, source: N, target: N) -> float: ...
```

---

## 8. Verification

| Check | Criterion |
|-------|-----------|
| V-TRV-001 | BFS on linear chain: nodes yielded in insertion order |
| V-TRV-002 | BFS on tree: nodes at depth d yielded before depth d+1 |
| V-TRV-003 | DFS post-order: leaf nodes yielded before their parents |
| V-TRV-004 | `Topological.create` on DAG: every predecessor appears before successor |
| V-TRV-005 | Topological on cyclic graph: raises `CycleError` |
| V-TRV-006 | `filter_fn=lambda n: n != "X"` excludes X from visited set |
| V-TRV-007 | `max_depth=1` yields start and its direct neighbors only |
| V-TRV-008 | `visited()` contains same set regardless of traversal order |
| V-TRV-009 | `union(A, B)` contains all nodes from both, each at most once |
| V-TRV-010 | Empty `AggregateVisitor.result()` returns `initial` |
