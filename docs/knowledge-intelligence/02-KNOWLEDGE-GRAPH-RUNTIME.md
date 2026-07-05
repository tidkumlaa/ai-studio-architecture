---
knowledge_id: KA-KIP-003
version: "1.0.0"
status: approved
owner: Chief Knowledge Architect
phase: 2.0D.2
created: 2026-06-29
review_date: 2026-12-29
canonical: true
domain: DOM-KNOWLEDGE
capability: knowledge-intelligence
type: specification
depends_on:
  - id: KA-SPEC-008
    reason: "Property graph formal definition"
  - id: KA-SPEC-017
    reason: "Graph algorithm specifications (ALG-003 through ALG-010)"
---

# Knowledge Graph Runtime

## Property Graph + BFS/DFS/SCC/Topological Sort/Cycle Detection

---

## 1. Graph Model

The Knowledge Graph is a labeled property graph:

```
G = (V, E, λ_V, λ_E, P_V, P_E)

V  — Vertex set (one per KnowledgeObject)
E  — Directed edge set (relationships)
λ_V — Vertex label function: V → DocumentType
λ_E — Edge label function: E → RelationshipType
P_V — Vertex property function: V → {id, capability, domain, status, health}
P_E — Edge property function: E → {confidence, inferred, weight}
```

---

## 2. Graph Variants

Six specialized views over the same underlying property graph:

| Graph | Vertices | Edges | Purpose |
|-------|----------|-------|---------|
| Relationship Graph | All objects | All relationships | Full graph; default traversal |
| Dependency Graph | All objects | depends_on edges only | What must exist for X to work |
| Capability Graph | Capabilities only | cross-capability edges | Capability coupling analysis |
| Evidence Graph | Objects + evidence | has_evidence edges | Traceability from fact to source |
| Impact Graph | Changed set | reverse depends_on | What breaks when X changes |
| Domain Graph | Domains only | cross-domain edges | Domain-level coupling |

All six are derived views — no duplicate storage. Built by filtering the property graph.

---

## 3. Interface

```python
class KnowledgeGraph:
    # Mutation (build phase only)
    def add_vertex(self, id: str, label: str, **props) -> None
    def add_edge(self, from_id: str, to_id: str, label: str, **props) -> None
    def remove_vertex(self, id: str) -> None  # Also removes incident edges
    def remove_edge(self, from_id: str, to_id: str, label: str) -> None

    # Vertex access — O(1)
    def vertex(self, id: str) -> Vertex | None
    def edges_from(self, id: str) -> list[Edge]
    def edges_to(self, id: str) -> list[Edge]
    def neighbors(self, id: str, labels: set[str] | None = None) -> list[str]
    def degree(self, id: str) -> int
    def in_degree(self, id: str) -> int

    # Traversal — O(V + E)
    def bfs(self, start: str, max_depth: int = -1,
            edge_labels: set[str] | None = None) -> Iterator[tuple[str, int]]
    def dfs(self, start: str, max_depth: int = -1,
            edge_labels: set[str] | None = None) -> Iterator[str]
    def reachable(self, start: str, edge_labels: set[str] | None = None) -> set[str]

    # Path queries
    def shortest_path(self, start: str, end: str) -> list[str] | None  # BFS unweighted
    def all_paths(self, start: str, end: str, max_depth: int = 8) -> list[list[str]]

    # Structural analysis
    def topological_sort(self) -> list[str]   # Kahn's algorithm O(V+E)
    def detect_cycles(self) -> list[list[str]]  # DFS coloring O(V+E)
    def scc(self) -> list[list[str]]            # Tarjan's O(V+E)
    def is_dag(self) -> bool
    def orphans(self) -> list[str]              # Vertices with no incident edges

    # Views
    def dependency_subgraph(self) -> KnowledgeGraph
    def capability_subgraph(self) -> KnowledgeGraph
    def impact_subgraph(self, seeds: list[str]) -> KnowledgeGraph

    # I/O
    def to_json(self) -> dict
    def from_json(self, data: dict) -> None
    def vertex_count(self) -> int
    def edge_count(self) -> int
```

---

## 4. BFS Implementation

```
BFS(start, max_depth, edge_labels):
  queue = deque([(start, 0)])
  visited = {start}
  WHILE queue:
    id, depth = queue.popleft()
    YIELD (id, depth)
    IF max_depth >= 0 AND depth >= max_depth: CONTINUE
    FOR edge IN edges_from(id):
      IF edge_labels AND edge.label NOT IN edge_labels: CONTINUE
      IF edge.to_id NOT IN visited:
        visited.add(edge.to_id)
        queue.append((edge.to_id, depth + 1))

Complexity: O(V + E)  Space: O(V)
```

---

## 5. Topological Sort (Kahn's Algorithm)

```
TOPOLOGICAL_SORT():
  in_degree = {v: 0 for v in vertices}
  FOR edge IN all_edges:
    in_degree[edge.to_id] += 1
  queue = deque(v for v in vertices if in_degree[v] == 0)
  order = []
  WHILE queue:
    v = queue.popleft()
    order.append(v)
    FOR neighbor IN neighbors(v):
      in_degree[neighbor] -= 1
      IF in_degree[neighbor] == 0: queue.append(neighbor)
  IF len(order) != len(vertices): RAISE CycleError
  RETURN order

Complexity: O(V + E)  Space: O(V)
```

---

## 6. Tarjan's SCC

```
TARJAN_SCC():
  index = 0; stack = []; on_stack = {}; indices = {}; lowlinks = {}; sccs = []

  STRONGCONNECT(v):
    indices[v] = lowlinks[v] = index++
    stack.push(v); on_stack[v] = True
    FOR neighbor IN neighbors(v):
      IF neighbor NOT IN indices:
        STRONGCONNECT(neighbor)
        lowlinks[v] = min(lowlinks[v], lowlinks[neighbor])
      ELIF on_stack[neighbor]:
        lowlinks[v] = min(lowlinks[v], indices[neighbor])
    IF lowlinks[v] == indices[v]:  # v is root of SCC
      scc = []
      WHILE True:
        w = stack.pop(); on_stack[w] = False; scc.append(w)
        IF w == v: BREAK
      sccs.append(scc)

  FOR v IN vertices:
    IF v NOT IN indices: STRONGCONNECT(v)
  RETURN sccs

Complexity: O(V + E)  Space: O(V)
```

---

## 7. Complexity Summary

| Operation | Time | Space |
|-----------|------|-------|
| Vertex lookup | O(1) | — |
| Edge lookup | O(degree) | — |
| BFS / DFS | O(V+E) | O(V) |
| Reachability | O(V+E) | O(V) |
| Shortest path | O(V+E) | O(V) |
| Topological sort | O(V+E) | O(V) |
| Cycle detection | O(V+E) | O(V) |
| Tarjan's SCC | O(V+E) | O(V) |
| Orphan detection | O(V+E) | O(V) |

---

## References

- [KA-SPEC-008](../knowledge-core/08-KNOWLEDGE-GRAPH-MODEL.md) — Formal graph model
- [KA-SPEC-017](../knowledge-core/17-ALGORITHMS-CATALOG.md) — ALG-003 through ALG-006, ALG-010
- [06-IMPACT-ANALYSIS-ENGINE.md](06-IMPACT-ANALYSIS-ENGINE.md) — Uses impact_subgraph
