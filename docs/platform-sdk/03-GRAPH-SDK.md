---
knowledge_id: KA-SDK-004
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
    reason: "Vertex/edge collections"
  - id: KA-SDK-003
    reason: "Graph algorithms are Algorithm instances"
---

# Graph SDK

## Generic Labeled Property Graph — Directed · Undirected · Weighted · Multi

---

## 1. Purpose

The Graph SDK provides a reusable, abstract graph substrate that generalizes
the Knowledge Graph Runtime. It is domain-agnostic: vertices and edges are
typed by generic type parameters. Any component requiring graph traversal,
cycle detection, path finding, or connectivity analysis depends on this SDK,
not on knowledge-specific graph implementations.

---

## 2. Graph Taxonomy

```
Graph[V, E]
├── DirectedGraph[V, E]
│   ├── DAG[V, E]                    — Directed Acyclic Graph
│   └── DirectedMultiGraph[V, E]     — Multiple edges between same pair
├── UndirectedGraph[V, E]
│   └── UndirectedMultiGraph[V, E]
└── PropertyGraph[V, E, VP, EP]      — Labeled vertices + edges with properties
    ├── ImmutablePropertyGraph[...]
    └── MutablePropertyGraph[...]
```

---

## 3. Interfaces

### 3.1 Vertex and Edge Models

```python
@dataclass(frozen=True)
class Vertex(Generic[V]):
    id:         str
    label:      str
    data:       V           # Domain-typed payload
    properties: ImmutableMap[str, Any]

@dataclass(frozen=True)
class Edge(Generic[E]):
    id:         str
    source_id:  str
    target_id:  str
    label:      str
    data:       E
    weight:     float       # Default 1.0
    properties: ImmutableMap[str, Any]
    directed:   bool
```

### 3.2 Graph[V, E] — base interface

```python
class Graph(Protocol[V, E]):
    def vertex(self, id: str) -> Vertex[V] | None       # O(1)
    def edge(self, id: str) -> Edge[E] | None           # O(1)
    def vertices(self) -> Sequence[Vertex[V]]           # O(V)
    def edges(self) -> Sequence[Edge[E]]                # O(E)
    def vertex_count(self) -> int                       # O(1)
    def edge_count(self) -> int                         # O(1)
    def contains_vertex(self, id: str) -> bool          # O(1)
    def contains_edge(self, id: str) -> bool            # O(1)
    def out_edges(self, vertex_id: str) -> Sequence[Edge[E]]  # O(deg)
    def in_edges(self, vertex_id: str) -> Sequence[Edge[E]]   # O(deg)
    def neighbors(self, vertex_id: str) -> Sequence[str]      # O(deg)
    def predecessors(self, vertex_id: str) -> Sequence[str]   # O(deg)
    def degree(self, vertex_id: str) -> int                   # O(1)
    def in_degree(self, vertex_id: str) -> int                # O(1)
    def out_degree(self, vertex_id: str) -> int               # O(1)
```

### 3.3 MutableGraph (extends Graph)

```python
class MutableGraph(Graph[V, E]):
    def add_vertex(self, id: str, label: str, data: V,
                   properties: dict[str, Any] | None = None) -> None  # O(1)
    def add_edge(self, id: str, source_id: str, target_id: str,
                 label: str, data: E, weight: float = 1.0,
                 properties: dict[str, Any] | None = None) -> None     # O(1)
    def remove_vertex(self, id: str) -> None  # O(deg) — removes incident edges
    def remove_edge(self, id: str) -> None    # O(1)
    def update_vertex(self, id: str, properties: dict[str, Any]) -> None  # O(1)
    def update_edge(self, id: str, weight: float | None = None,
                    properties: dict[str, Any] | None = None) -> None
```

### 3.4 GraphView — derived read-only projections

```python
class GraphView(Graph[V, E]):
    """A filtered or projected view, not a copy."""

class VertexFilterView(GraphView[V, E]):
    def __init__(self, graph: Graph[V, E], predicate: Callable[[Vertex[V]], bool])

class EdgeFilterView(GraphView[V, E]):
    def __init__(self, graph: Graph[V, E], predicate: Callable[[Edge[E]], bool])

class SubgraphView(GraphView[V, E]):
    def __init__(self, graph: Graph[V, E], vertex_ids: Set[str])

class ReverseView(GraphView[V, E]):
    def __init__(self, graph: Graph[V, E])  # Swaps in/out edges — O(1) creation
```

### 3.5 GraphBuilder

```python
class GraphBuilder(Protocol[V, E]):
    def vertex(self, id: str, label: str, data: V, **props) -> "GraphBuilder[V, E]"
    def edge(self, source: str, target: str, label: str, data: E,
             weight: float = 1.0, **props) -> "GraphBuilder[V, E]"
    def build(self) -> MutableGraph[V, E]
    def build_immutable(self) -> Graph[V, E]

class GraphSerializer(Protocol[V, E]):
    def to_dict(self, graph: Graph[V, E]) -> dict
    def from_dict(self, data: dict) -> Graph[V, E]
    def to_adjacency_list(self, graph: Graph[V, E]) -> dict[str, list[str]]
    def to_adjacency_matrix(self, graph: Graph[V, E]) -> list[list[float]]
```

---

## 4. Contracts

| ID | Contract |
|----|----------|
| C-GRF-001 | `vertex_count()` and `edge_count()` are always O(1). |
| C-GRF-002 | `remove_vertex` removes all incident edges. |
| C-GRF-003 | `GraphView` is a view: changes to the underlying graph are reflected. |
| C-GRF-004 | `ReverseView` swaps in/out edges without copying; O(1) creation. |
| C-GRF-005 | `add_edge` with a non-existent `source_id` or `target_id` raises `VertexNotFoundError`. |
| C-GRF-006 | `DAG` raises `CycleError` if `add_edge` would create a cycle. |
| C-GRF-007 | Edge IDs are globally unique within a graph instance. |
| C-GRF-008 | `weight` defaults to 1.0 for unweighted usage. |
| C-GRF-009 | An undirected graph's `in_edges` and `out_edges` return the same set. |
| C-GRF-010 | `GraphBuilder.build()` produces a complete, valid graph or raises `GraphBuildError`. |

---

## 5. Algorithms

The following algorithms are Graph SDK first-class operations (delegated to Traversal SDK):

| Algorithm | Complexity | Notes |
|-----------|-----------|-------|
| BFS | O(V+E) | Level-order traversal |
| DFS | O(V+E) | Pre/post-order variants |
| Shortest Path (BFS) | O(V+E) | Unweighted |
| Shortest Path (Dijkstra) | O((V+E) log V) | Non-negative weights |
| Shortest Path (Bellman-Ford) | O(V·E) | Negative weights |
| Topological Sort (Kahn's) | O(V+E) | DAG only |
| SCC (Tarjan's) | O(V+E) | Directed graphs |
| Cycle Detection | O(V+E) | DFS coloring |
| Connected Components | O(V+E) | Undirected |
| Minimum Spanning Tree (Kruskal's) | O(E log E) | Weighted undirected |
| PageRank | O(V+E·iterations) | Convergence-based |

---

## 6. Dependencies

- Collections SDK (KA-SDK-002) — vertex/edge storage
- Algorithms SDK (KA-SDK-003) — algorithm composition
- Traversal SDK (KA-SDK-016) — traversal protocol

---

## 7. Extension Points

```python
class GraphStorage(Protocol[V, E]):
    """Pluggable storage backend (in-memory, persistent, remote)."""
    def get_vertex(self, id: str) -> Vertex[V] | None: ...
    def put_vertex(self, v: Vertex[V]) -> None: ...
    def get_out_edges(self, vertex_id: str) -> list[Edge[E]]: ...
    def put_edge(self, e: Edge[E]) -> None: ...

class GraphIndex(Protocol):
    """Secondary index on graph properties."""
    def index(self, vertex: Vertex) -> None: ...
    def query(self, predicate: dict) -> list[str]: ...  # Returns vertex IDs

class GraphEventListener(Protocol[V, E]):
    """Observe graph mutations."""
    def on_vertex_added(self, v: Vertex[V]) -> None: ...
    def on_edge_added(self, e: Edge[E]) -> None: ...
    def on_vertex_removed(self, id: str) -> None: ...
    def on_edge_removed(self, id: str) -> None: ...
```

---

## 8. Verification

| Check | Criterion |
|-------|-----------|
| V-GRF-001 | `add_vertex` then `contains_vertex` returns True |
| V-GRF-002 | `remove_vertex` also removes incident edges |
| V-GRF-003 | `ReverseView`: original `out_edges(A)` = reverse `in_edges(A)` |
| V-GRF-004 | `DAG.add_edge` creating cycle raises `CycleError` |
| V-GRF-005 | `VertexFilterView` hides vertices not matching predicate |
| V-GRF-006 | `SubgraphView` contains only specified vertices and their inter-edges |
| V-GRF-007 | `vertex_count()` returns 0 for empty graph |
| V-GRF-008 | `edge_count()` matches actual number of edges added |
| V-GRF-009 | Undirected edge appears in both `out_edges(A)` and `out_edges(B)` |
| V-GRF-010 | `GraphSerializer.from_dict(to_dict(g))` produces structurally equal graph |
