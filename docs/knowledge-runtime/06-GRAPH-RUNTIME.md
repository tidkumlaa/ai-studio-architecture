---
knowledge_id: KR-ARCH-006
version: "1.0.0"
status: approved
owner: Chief Knowledge Architect
phase: 2.0D.3A
created: 2026-06-29
review_date: 2026-12-29
canonical: true
domain: DOM-KNOWLEDGE
capability: knowledge-runtime
type: architecture
depends_on:
  - id: KA-SPEC-008
    reason: "Formal labeled property graph model; relationship type definitions"
  - id: KA-KIP-003
    reason: "KnowledgeGraphRuntime interface and algorithm specifications"
  - id: KR-ARCH-005
    reason: "Execution model defines when graph is built, rebuilt, and hot-reloaded"
implements:
  - KA-SPEC-001
---

# Knowledge Runtime — Graph Runtime

## Data Model, Traversal Algorithms, Inference, and Rebuild Protocol

---

## 1. Purpose

This document specifies the **graph runtime layer** of the Knowledge Runtime: the in-memory property graph that represents all knowledge objects and their relationships, the algorithms that operate on it, and the protocols for building, maintaining, and rebuilding it.

The graph runtime is the execution backend for:
- `TRAVERSE` clauses in KQL queries
- Impact analysis (change propagation)
- Dependency and ancestor enumeration
- Circular dependency detection
- Health score propagation

It does not define the query language (see [KR-ARCH-007](07-QUERY-MODEL.md)) or the boot sequence that initializes it (see [KR-ARCH-005](05-EXECUTION-MODEL.md)).

---

## 2. Graph Data Model

The Knowledge Graph is a **labeled directed property graph** conforming to the formal model in [KA-SPEC-008](../knowledge-core/08-KNOWLEDGE-GRAPH-MODEL.md):

```
G = (V, E, λ_V, λ_E, P_V, P_E)

V    — finite set of vertices (one per KnowledgeObject)
E    — finite set of directed edges (one per declared relationship)
λ_V  — vertex label function: V → KnowledgeType
λ_E  — edge label function:   E → RelationshipType
P_V  — vertex property map:   V → Map[String, Any]
P_E  — edge property map:     E → Map[String, Any]
```

### 2.1 Node Structure

Each vertex represents one `KnowledgeObject` at a specific point in time. Properties are populated from the object's persisted metadata at load time and updated on incremental index events.

```
KnowledgeGraphNode
  ├── node_id          : str        # = KnowledgeIdentity.knowledge_id; stable across versions
  ├── object_type      : str        # "spec" | "architecture" | "requirement" | "adr" | ...
  ├── domain           : str        # "DOM-KNOWLEDGE" | "DOM-WORKFLOW" | "DOM-AI" | ...
  ├── lifecycle_state  : str        # CREATED | DRAFT | REVIEW | APPROVED | DEPRECATED | ARCHIVED
  ├── health_score     : float      # 0.0–1.0; propagated from dependencies
  ├── coverage         : dict       # { arch: float, impl: float, test: float, desktop: float }
  └── properties       : dict       # all other frontmatter fields (owner, version, phase, ...)
```

**Node identity:** `node_id` is the sole stable key. `object_type`, `domain`, and all properties may change across versions; `node_id` never changes after initial registration (per [KA-SPEC-003](../knowledge-core/03-KNOWLEDGE-IDENTITY-SPECIFICATION.md)).

### 2.2 Edge Structure

Each directed edge represents one declared or inferred relationship between two nodes.

```
KnowledgeGraphEdge
  ├── edge_id           : str        # UUID; stable for the lifetime of the relationship
  ├── source_id         : str        # node_id of the originating node
  ├── target_id         : str        # node_id of the destination node
  ├── relationship_type : str        # see Section 2.3
  ├── weight            : float      # default 1.0; higher = stronger coupling
  ├── inferred          : bool       # True for edges added by ReasoningEngine
  └── metadata          : dict       # reason, criticality, confidence, last_verified, ...
```

**Edge uniqueness:** The triple `(source_id, target_id, relationship_type)` must be unique in G. Duplicate declarations are merged into the existing edge (metadata is union-merged; weight takes the max).

### 2.3 Relationship Type Semantics

| Type | Direction | Cardinality | Transitive | Description |
|------|-----------|-------------|-----------|-------------|
| `IMPLEMENTS` | A → B | Many-to-one | Yes | A provides a concrete realization of abstract spec B |
| `DEPENDS_ON` | A → B | Many-to-many | Yes | A cannot function without B |
| `VALIDATES` | A → B | Many-to-many | No | Test/evidence A verifies the claims of spec B |
| `SUPERSEDES` | A → B | One-to-one | No | A replaces B; B transitions to DEPRECATED |
| `EVIDENCED_BY` | A → B | Many-to-many | No | Claim in A is supported by artifact B |
| `COVERS` | A → B | Many-to-many | No | A provides coverage for requirement/capability B |
| `TRACES_TO` | A → B | Many-to-one | Yes | Child A traces to parent/source requirement B |
| `IMPACTS` | A → B | Many-to-many | Yes | Change to A causes downstream effects in B (inferred) |

**Transitivity interpretation:** For transitive types (`IMPLEMENTS`, `DEPENDS_ON`, `TRACES_TO`, `IMPACTS`), the runtime maintains both direct edges and materializes reachability sets in the traversal cache. For non-transitive types, only direct edges are stored; traversal stops at depth 1 unless explicitly requested.

### 2.4 In-Memory Storage Layout

```
KnowledgeGraphStore
  ├── vertices   : HashMap<node_id, KnowledgeGraphNode>    # O(1) lookup
  ├── edges      : HashMap<edge_id, KnowledgeGraphEdge>    # O(1) lookup by edge_id
  ├── adj_out    : HashMap<node_id, List<edge_id>>          # outbound adjacency
  ├── adj_in     : HashMap<node_id, List<edge_id>>          # inbound adjacency
  ├── by_type    : HashMap<object_type, Set<node_id>>       # type index
  ├── by_domain  : HashMap<domain, Set<node_id>>            # domain index
  └── by_rel     : HashMap<relationship_type, Set<edge_id>> # relationship type index
```

**Memory estimate (10,000 nodes, 40,000 edges):**

| Structure | Est. Size |
|-----------|----------|
| `vertices` map | ~50 MB |
| `edges` map | ~20 MB |
| Adjacency lists | ~8 MB |
| Type/domain indexes | ~2 MB |
| **Total** | **~80 MB** |

---

## 3. Graph Variants (Derived Views)

Six specialized views are available over the same underlying property graph. All views are derived by edge-type filtering — no duplicate node or edge storage.

| View | Vertex Filter | Edge Filter | Primary Use |
|------|--------------|-------------|-------------|
| Full Graph | All | All | Default traversal; KQL TRAVERSE |
| Dependency Graph | All | `DEPENDS_ON` only | Dependency resolution; build order |
| Traceability Graph | All | `IMPLEMENTS`, `TRACES_TO` | Requirement traceability |
| Evidence Graph | All | `EVIDENCED_BY`, `VALIDATES` | Coverage and confidence |
| Impact Graph | Changed set (seed) | Reverse `DEPENDS_ON`, `IMPACTS` | Change blast radius |
| Domain Graph | Domain nodes | Cross-domain edges | Domain coupling analysis |
| Capability Graph | Capability nodes | `COVERS`, `DEPENDS_ON` | Capability coupling |

Views are not cached as separate data structures. They are produced on demand by `KnowledgeGraphRuntime.subgraph(edge_filter)`, which iterates the `by_rel` index in O(|filtered_edges|) time.

---

## 4. Graph Loading

See [KR-ARCH-005](05-EXECUTION-MODEL.md) §4 for the full boot loading sequence. This section supplements with runtime-specific detail.

### 4.1 Node Load Order

Nodes are loaded by iterating `KnowledgeRegistry.all()`. The registry returns objects in stable insertion order (order of first registration during Phase 3). Load order does not affect correctness; all nodes must exist before any edges are created (invariant enforced by the runtime).

### 4.2 Edge Deduplication

During loading, if the same `(source_id, target_id, relationship_type)` triple appears in multiple index sources (e.g., TraceabilityIndex and CapabilityIndex both declare the same relationship), the runtime deduplicates:

```
FUNCTION add_edge_dedup(source_id, target_id, rel_type, weight, metadata):
  key = (source_id, target_id, rel_type)
  IF key IN edge_index:
    existing = edges[edge_index[key]]
    existing.weight = max(existing.weight, weight)
    existing.metadata = merge(existing.metadata, metadata)
    RETURN  # No new edge created
  CREATE new edge; add to edges, adj_out, adj_in, by_rel
```

### 4.3 Post-Load Structural Analysis

After all nodes and edges are loaded:

```
1. Run Tarjan's SCC
   FOR EACH SCC with |SCC| > 1:
     CLASSIFY as circular dependency cluster
     EMIT knowledge.graph.cycle_detected {nodes: [...], relationship_types: [...]}

2. Run health score propagation (Section 5.2)

3. Compute orphan set
   orphans = {v ∈ V : in_degree(v) = 0 AND out_degree(v) = 0}
   EMIT knowledge.graph.orphans_detected {count: |orphans|} IF |orphans| > 0

4. Emit knowledge.graph.loaded {
     node_count, edge_count, scc_clusters, orphan_count,
     load_duration_ms
   }
```

---

## 5. Traversal Algorithms

All traversal algorithms are exposed through `KnowledgeTraversalEngine`, which wraps `KnowledgeGraphRuntime`. The traversal engine adds:
- Result caching (keyed on traversal parameters)
- Max-depth enforcement (prevents infinite loops on cyclic subgraphs)
- Edge-type filtering
- Direction reversal (forward = follow `adj_out`; reverse = follow `adj_in`)

### 5.1 Breadth-First Search (BFS)

**Use:** Discover all objects reachable from a root within N hops; level-order traversal for impact radius by distance.

```
INPUT:  root_id, max_depth (default unlimited), edge_filter (default all)
OUTPUT: List of (node_id, depth, path_from_root)
COMPLEXITY: O(V + E) time, O(V) space

ALGORITHM BFS(root_id, max_depth, edge_filter):
  queue   = deque([(root_id, 0, [root_id])])
  visited = {root_id}
  results = []

  WHILE queue NOT empty:
    (node_id, depth, path) = queue.popleft()
    results.append((node_id, depth, path))

    IF max_depth >= 0 AND depth >= max_depth: CONTINUE

    FOR edge_id IN adj_out[node_id]:
      edge = edges[edge_id]
      IF edge_filter AND edge.relationship_type NOT IN edge_filter: CONTINUE
      IF edge.target_id NOT IN visited:
        visited.add(edge.target_id)
        queue.append((edge.target_id, depth + 1, path + [edge.target_id]))

  RETURN results
```

**Cache key:** `(root_id, max_depth, frozenset(edge_filter), direction)`
**Cache TTL:** invalidated on `knowledge.index.updated` if any node in the result set is affected

### 5.2 Depth-First Search (DFS)

**Use:** Full reachability, cycle detection path reporting, pre/post-order processing.

```
INPUT:  root_id, max_depth (default unlimited), edge_filter
OUTPUT: List of node_ids in DFS discovery order + (discovery_time, finish_time) per node
COMPLEXITY: O(V + E) time, O(V) space (recursion stack)

ALGORITHM DFS(root_id, max_depth, edge_filter):
  visited     = {}
  timer       = [0]
  order       = []
  timestamps  = {}

  FUNCTION visit(node_id, depth):
    IF node_id IN visited: RETURN
    IF max_depth >= 0 AND depth > max_depth: RETURN
    visited.add(node_id)
    timestamps[node_id] = (timer[0], None)
    timer[0] += 1
    order.append(node_id)

    FOR edge_id IN adj_out[node_id]:
      edge = edges[edge_id]
      IF edge_filter AND edge.relationship_type NOT IN edge_filter: CONTINUE
      visit(edge.target_id, depth + 1)

    timestamps[node_id] = (timestamps[node_id][0], timer[0])
    timer[0] += 1

  visit(root_id, 0)
  RETURN order, timestamps
```

The DFS implementation uses an explicit stack (not Python recursion) to avoid stack overflow on deep graphs.

### 5.3 Topological Sort

**Use:** Determine the correct initialization or build order for a set of objects with declared `DEPENDS_ON` edges. Guarantees that all dependencies of X appear before X in the output list.

```
INPUT:  (optional) subgraph filter; default = full Dependency Graph
OUTPUT: Ordered list of node_ids; OR CycleError if graph is not a DAG
COMPLEXITY: O(V + E) time, O(V) space
ALGORITHM: Kahn's (iterative BFS variant — avoids recursion)

ALGORITHM TOPOLOGICAL_SORT(edge_filter = {DEPENDS_ON}):
  in_degree = {v: 0 for v in vertices}
  FOR edge_id IN by_rel[edge_filter]:
    edge = edges[edge_id]
    in_degree[edge.target_id] += 1

  queue = deque(v for v in vertices if in_degree[v] == 0)
  order = []

  WHILE queue NOT empty:
    v = queue.popleft()
    order.append(v)
    FOR edge_id IN adj_out[v]:
      edge = edges[edge_id]
      IF edge.relationship_type NOT IN edge_filter: CONTINUE
      in_degree[edge.target_id] -= 1
      IF in_degree[edge.target_id] == 0:
        queue.append(edge.target_id)

  IF len(order) != len(vertices):
    RAISE CycleError(remaining = {v for v in vertices if in_degree[v] > 0})

  RETURN order
```

### 5.4 Strongly Connected Components — Tarjan's Algorithm

**Use:** Detect circular dependency clusters. A cluster of size > 1 represents objects that mutually depend on each other, which is an architectural violation for `DEPENDS_ON` and `SUPERSEDES` edges.

```
INPUT:  (optional) edge_filter; default = {DEPENDS_ON, IMPLEMENTS}
OUTPUT: List[List[node_id]] — each inner list is one SCC; single-node SCCs are normal
COMPLEXITY: O(V + E) time, O(V) space

ALGORITHM TARJAN_SCC(edge_filter):
  index_counter = [0]
  stack = []
  on_stack = {}
  indices = {}
  lowlinks = {}
  sccs = []

  FUNCTION strongconnect(v):
    indices[v]  = lowlinks[v] = index_counter[0]
    index_counter[0] += 1
    stack.append(v)
    on_stack[v] = True

    FOR edge_id IN adj_out[v]:
      edge = edges[edge_id]
      IF edge.relationship_type NOT IN edge_filter: CONTINUE
      w = edge.target_id
      IF w NOT IN indices:
        strongconnect(w)
        lowlinks[v] = min(lowlinks[v], lowlinks[w])
      ELIF on_stack[w]:
        lowlinks[v] = min(lowlinks[v], indices[w])

    IF lowlinks[v] == indices[v]:   # v is root of an SCC
      scc = []
      WHILE True:
        w = stack.pop()
        on_stack[w] = False
        scc.append(w)
        IF w == v: BREAK
      sccs.append(scc)

  FOR v IN vertices:
    IF v NOT IN indices:
      strongconnect(v)

  RETURN sccs
```

**Reporting:** SCCs of size > 1 are reported as `knowledge.graph.cycle_detected` events. The runtime does not reject cyclic graphs; cycles are tracked in the `CycleRegistry` and surfaced as audit warnings.

### 5.5 Shortest Path (Dijkstra)

**Use:** Find the minimum-cost path between two knowledge objects. "Cost" is the inverse of edge weight — a strongly coupled pair (weight=2.0) contributes less distance than a weakly coupled pair (weight=0.5).

```
INPUT:  source_id, target_id, edge_filter (default all)
OUTPUT: List of (node_id, edge_type) tuples from source to target + total_distance
        OR None if no path exists
COMPLEXITY: O((V + E) log V) with binary heap

ALGORITHM DIJKSTRA(source_id, target_id, edge_filter):
  dist     = {v: ∞ for v in vertices};  dist[source_id] = 0
  prev     = {v: None for v in vertices}
  prev_rel = {v: None for v in vertices}
  heap     = [(0, source_id)]

  WHILE heap NOT empty:
    (d, u) = heappop(heap)
    IF d > dist[u]: CONTINUE
    IF u == target_id: BREAK

    FOR edge_id IN adj_out[u]:
      edge = edges[edge_id]
      IF edge_filter AND edge.relationship_type NOT IN edge_filter: CONTINUE
      edge_cost = 1.0 / max(edge.weight, 0.001)   # invert weight to cost
      alt = dist[u] + edge_cost
      IF alt < dist[edge.target_id]:
        dist[edge.target_id] = alt
        prev[edge.target_id] = u
        prev_rel[edge.target_id] = edge.relationship_type
        heappush(heap, (alt, edge.target_id))

  IF dist[target_id] == ∞: RETURN None

  path = []
  curr = target_id
  WHILE curr IS NOT None:
    path.append((curr, prev_rel[curr]))
    curr = prev[curr]
  RETURN reversed(path), dist[target_id]
```

### 5.6 Ancestor Enumeration

**Use:** Given object X, enumerate all objects that X transitively depends on — the full set X needs in order to function.

```
INPUT:  node_id, rel_filter (default {DEPENDS_ON, IMPLEMENTS})
OUTPUT: Set[node_id] — all transitive predecessors; does not include node_id itself
COMPLEXITY: O(V + E)

ALGORITHM ANCESTORS(node_id, rel_filter):
  # Reverse BFS: follow adj_in instead of adj_out
  visited = {node_id}
  queue   = deque([node_id])

  WHILE queue NOT empty:
    current = queue.popleft()
    FOR edge_id IN adj_in[current]:
      edge = edges[edge_id]
      IF rel_filter AND edge.relationship_type NOT IN rel_filter: CONTINUE
      IF edge.source_id NOT IN visited:
        visited.add(edge.source_id)
        queue.append(edge.source_id)

  visited.remove(node_id)
  RETURN visited
```

### 5.7 Descendant Enumeration

**Use:** Given object X, enumerate all objects that transitively depend on X — the "blast radius" of a change to X.

```
INPUT:  node_id, rel_filter (default {DEPENDS_ON, IMPLEMENTS})
OUTPUT: Set[node_id] — all transitive successors; does not include node_id itself
COMPLEXITY: O(V + E)

ALGORITHM DESCENDANTS(node_id, rel_filter):
  # Forward BFS on adj_out
  visited = {node_id}
  queue   = deque([node_id])

  WHILE queue NOT empty:
    current = queue.popleft()
    FOR edge_id IN adj_out[current]:
      edge = edges[edge_id]
      IF rel_filter AND edge.relationship_type NOT IN rel_filter: CONTINUE
      IF edge.target_id NOT IN visited:
        visited.add(edge.target_id)
        queue.append(edge.target_id)

  visited.remove(node_id)
  RETURN visited
```

### 5.8 Impact Graph Construction

**Use:** Given a set of changed objects (seeds), compute the full impact set — all objects that may be affected by the changes.

```
INPUT:  seed_ids: Set[node_id], rel_filter (default {DEPENDS_ON, IMPACTS, IMPLEMENTS})
OUTPUT: ImpactReport

ALGORITHM IMPACT_GRAPH(seed_ids, rel_filter):
  all_impacted = set()

  FOR each seed_id IN seed_ids:
    impacted = DESCENDANTS(seed_id, rel_filter)
    all_impacted = all_impacted ∪ impacted

  # Group by impact severity (estimated by distance from seed)
  FOR each impacted_id IN all_impacted:
    min_dist = min(SHORTEST_PATH(seed_id, impacted_id).distance for seed_id in seed_ids)
    CLASSIFY:
      min_dist == 1 → DIRECT
      min_dist == 2 → INDIRECT
      min_dist  > 2 → TRANSITIVE

  RETURN ImpactReport {
    seeds:      seed_ids,
    direct:     Set[node_id]  (distance = 1),
    indirect:   Set[node_id]  (distance = 2),
    transitive: Set[node_id]  (distance > 2),
    total:      |all_impacted|
  }
```

### 5.9 Traversal Algorithm Comparison

| Algorithm | Time | Space | Max Depth | Cache | Primary Use |
|-----------|------|-------|-----------|-------|------------|
| BFS | O(V+E) | O(V) | Configurable | Yes | Reachability, level-order |
| DFS | O(V+E) | O(V) | Configurable | No | Full exploration, timestamps |
| Topological Sort | O(V+E) | O(V) | N/A | Yes | Build order |
| Tarjan's SCC | O(V+E) | O(V) | N/A | No | Cycle detection |
| Dijkstra | O((V+E)logV) | O(V) | N/A | Yes | Shortest path |
| Ancestor Enum | O(V+E) | O(V) | Unlimited | Yes | Dependency set |
| Descendant Enum | O(V+E) | O(V) | Unlimited | Yes | Blast radius |
| Impact Graph | O(k(V+E)) | O(V) | Unlimited | No | Change analysis |

`k` = number of seed nodes in Impact Graph.

---

## 6. Inference — Derived Edges

The `KnowledgeReasoningEngine` adds derived edges to the graph after initial loading. Derived edges are marked `inferred=True` and are never persisted to the TraceabilityIndex; they are recomputed on every graph load and after every significant incremental update.

### 6.1 IMPACTS Edge Derivation

`IMPACTS` edges are derived from the transitive closure of `DEPENDS_ON` edges combined with a change-signal model:

```
RULE: IF A DEPENDS_ON B (directly or transitively)
      THEN B IMPACTS A

DERIVATION:
  FOR each node X:
    ancestors_of_X = ANCESTORS(X, rel_filter={DEPENDS_ON})
    FOR each ancestor A:
      IF NOT edge_exists(A, X, IMPACTS):
        ADD_EDGE(A, X, IMPACTS, weight=1.0/(hop_distance), inferred=True)
```

`IMPACTS` edges are directional in the **reverse** direction of `DEPENDS_ON`: if X depends on A, then a change in A impacts X.

### 6.2 Health Score Propagation

Health scores propagate down the dependency graph: a failing dependency degrades its dependents.

```
RULE: node.health_score = min(
        node.declared_health_score,
        min(dep.health_score * PROPAGATION_FACTOR for dep in direct_dependencies)
      )

PROPAGATION_FACTOR = 0.9  (10% degradation per hop)

ALGORITHM PROPAGATE_HEALTH():
  order = TOPOLOGICAL_SORT(edge_filter={DEPENDS_ON, IMPLEMENTS})
  FOR node_id IN order:  # Process in dependency order (dependencies before dependents)
    node = vertices[node_id]
    dep_scores = []
    FOR edge_id IN adj_in[node_id]:
      edge = edges[edge_id]
      IF edge.relationship_type IN {DEPENDS_ON, IMPLEMENTS}:
        dep_node = vertices[edge.source_id]
        dep_scores.append(dep_node.health_score * PROPAGATION_FACTOR)
    IF dep_scores:
      node.health_score = min(node.declared_health_score, min(dep_scores))
```

Health propagation is re-run after every incremental index update that touches a node's `health_score` property or changes the dependency graph topology.

### 6.3 Coverage Propagation

Evidence declared on a child object lifts the coverage score of its parent:

```
RULE: IF child TRACES_TO parent AND child.coverage[dim] > parent.coverage[dim]
      THEN parent.effective_coverage[dim] = max(parent.coverage[dim], child.coverage[dim] * 0.8)

0.8 = child evidence contributes 80% of its value to the parent
```

Coverage propagation runs after health propagation, in the same topological order.

---

## 7. Semantic Graph Layer (Future-Ready)

A semantic similarity layer is defined here for forward compatibility. It is **not** activated in phase 2.0D.3A but the data model reserves space for it.

### 7.1 Semantic Node Properties (Reserved)

```
KnowledgeGraphNode.properties["embedding_vector"] : List[float] | None
  # populated by AI layer (phase 2.0E+)
  # dimension: 1536 (OpenAI text-embedding-3-small) or 768 (platform embedding model)
```

### 7.2 Semantic Edge Construction (Future)

```
FOR each pair (A, B) where A.embedding_vector IS NOT None AND B.embedding_vector IS NOT None:
  similarity = cosine_similarity(A.embedding_vector, B.embedding_vector)
  IF similarity >= SEMANTIC_THRESHOLD (default 0.85):
    ADD_EDGE(A, B, SEMANTIC_SIMILAR, weight=similarity, inferred=True)
```

Semantic edges are:
- Bidirectional (cosine similarity is symmetric)
- Never persisted
- Used only for `FIND SIMILAR TO` KQL queries
- Not used in dependency resolution, health propagation, or impact analysis

---

## 8. Graph Rebuild Protocol

A full graph rebuild replaces the entire in-memory graph atomically. It is triggered when incremental updates are insufficient or after a snapshot restore.

### 8.1 Rebuild Triggers

| Trigger | Event | Description |
|---------|-------|-------------|
| Runtime boot | (always) | Fresh graph from index state |
| Snapshot restore | `knowledge.snapshot.restored` | Index state has changed wholesale |
| Corruption detected | `knowledge.graph.corrupted` | Graph invariants violated |
| Admin command | Manual | Operator-initiated rebuild |
| Index major change | `knowledge.index.major_change` | > 10% of objects changed |

### 8.2 Atomic Rebuild Sequence

The rebuild is **double-buffered**: a new graph is built in the background while the existing graph continues serving queries. The swap is atomic from the caller's perspective.

```
REBUILD SEQUENCE:

1. ALLOCATE new_graph = KnowledgeGraphStore()
2. BUILD new_graph (Section 4 loading sequence) using current KnowledgeRegistry state
3. RUN Tarjan's SCC on new_graph
4. RUN health propagation on new_graph
5. VALIDATE new_graph invariants (Section 9)
6. IF validation fails:
     DISCARD new_graph
     EMIT knowledge.graph.rebuild_failed {reason}
     RETAIN existing graph (continue serving)
7. ELSE:
     ACQUIRE graph_swap_lock (< 1ms hold time)
     graph_store = new_graph
     RELEASE graph_swap_lock
     EMIT knowledge.graph.rebuilt {
       node_count, edge_count, inferred_edge_count,
       scc_clusters, orphan_count, rebuild_duration_ms
     }
```

In-flight queries that hold a reference to the old graph complete against the old graph. The lock is only held during the pointer swap, not during the build.

---

## 9. Graph Invariants

The runtime enforces the following invariants after every rebuild and incremental update. Violation of any invariant is reported as an ERROR-severity audit finding.

| ID | Invariant | Formal Rule |
|----|-----------|------------|
| GI-1 | No self-loops | ∀ e ∈ E: e.source_id ≠ e.target_id |
| GI-2 | Reference integrity | ∀ e ∈ E: e.source_id ∈ V ∧ e.target_id ∈ V |
| GI-3 | No duplicate edges | ∀ e1,e2: (e1.source, e1.target, e1.type) ≠ (e2.source, e2.target, e2.type) |
| GI-4 | Supersession is a DAG | SUPERSEDES edges contain no cycles (|SCC| = 1 for all SCCs on this type) |
| GI-5 | Single canonical per concept | ∀ (capability, object_type): |{v : v.canonical=True}| ≤ 1 |
| GI-6 | Type compatibility | e.relationship_type is valid for (λ_V(e.source), λ_V(e.target)) per type matrix |
| GI-7 | Weight is positive | ∀ e ∈ E: e.weight > 0 |

GI-1 through GI-3 are checked on every edge add operation (O(1) per edge).
GI-4 is checked by Tarjan's SCC after full load.
GI-5 is checked by the registry before node update.
GI-6 is checked at edge creation time against the type compatibility matrix.
GI-7 is enforced by defaulting to 1.0 if weight ≤ 0.

---

## 10. Graph Health Metrics

The graph runtime emits structural health metrics to `PlatformDiagnostics`:

| Metric | Target | Definition |
|--------|--------|-----------|
| `knowledge.graph.node_count` | — | Total vertices in G |
| `knowledge.graph.edge_count` | — | Total edges in G (including inferred) |
| `knowledge.graph.inferred_edge_count` | — | Edges with `inferred=True` |
| `knowledge.graph.scc_cluster_count` | 0 | SCCs with |SCC| > 1 (circular dependencies) |
| `knowledge.graph.orphan_count` | < 5% of nodes | Nodes with degree = 0 |
| `knowledge.graph.avg_out_degree` | — | Mean outbound edge count per node |
| `knowledge.graph.connectivity_pct` | > 80% | % of nodes with degree ≥ 2 |
| `knowledge.graph.reference_integrity_pct` | 100% | % of edges with valid endpoints |

---

## References

- [KA-SPEC-008](../knowledge-core/08-KNOWLEDGE-GRAPH-MODEL.md) — Formal property graph model and edge type definitions
- [KA-KIP-003](../knowledge-intelligence/02-KNOWLEDGE-GRAPH-RUNTIME.md) — KnowledgeGraphRuntime interface and low-level algorithm implementations
- [KA-SPEC-017](../knowledge-core/17-ALGORITHMS-CATALOG.md) — ALG-003 through ALG-010 (BFS, DFS, Tarjan, Kahn, Dijkstra)
- [KR-ARCH-005](05-EXECUTION-MODEL.md) — Boot sequence that populates this graph
- [KR-ARCH-007](07-QUERY-MODEL.md) — Query model that traverses this graph via KQL
- [KA-KIP-006](../knowledge-intelligence/06-IMPACT-ANALYSIS-ENGINE.md) — Consumer of Impact Graph construction (Section 5.8)
