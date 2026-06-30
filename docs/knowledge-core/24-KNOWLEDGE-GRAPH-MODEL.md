# KNW-KC-ARCH-024 — Knowledge Graph Model

**Phase:** 3.0C  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

The Knowledge Graph is the structural backbone of KOS. Every Knowledge Object is a node; every relationship is a typed, weighted, directed edge. The graph is the queryable, traversable representation of all knowledge in the ecosystem.

---

## Graph Type

**Property Graph** — typed nodes, typed edges, properties on both.

```
G = (V, E, λ_v, λ_e, P_v, P_e)

V   = set of Knowledge Object nodes
E   = set of directed, labeled edges (KnowledgeRelation)
λ_v = node label function: v → KnowledgeObjectType
λ_e = edge label function: e → RelationType
P_v = node property function: v → {field → value} (full KnowledgeObject)
P_e = edge property function: e → {field → value} (full KnowledgeRelation)
```

---

## Node Model

```python
@dataclass
class KnowledgeNode:
    node_id: str                        # = knowledge_id
    label: KnowledgeObjectType
    namespace: str
    status: KnowledgeLifecycleStatus
    version: str
    confidence: float
    quality_score: float | None
    properties: dict                    # full KnowledgeObject fields
    in_edges: list[str]                 # relation_ids pointing TO this node
    out_edges: list[str]                # relation_ids pointing FROM this node
```

---

## Edge Model

```python
@dataclass
class KnowledgeEdge:
    edge_id: str                        # = relation_id
    label: RelationType
    source: str                         # node_id (knowledge_id)
    target: str                         # node_id (knowledge_id)
    weight: float                       # confidence * strength_multiplier
    direction: RelationDirection
    is_transitive: bool
    is_symmetric: bool
    properties: dict                    # full KnowledgeRelation fields
```

Edge weight computation:
```
strength_multipliers:
  WEAK: 0.25, NORMAL: 0.5, STRONG: 0.75, CRITICAL: 1.0

weight = edge.confidence * strength_multiplier[edge.strength]
```

---

## Graph Invariants

| Code | Invariant | Enforcement |
|------|-----------|-------------|
| GI-001 | No isolated nodes — every node has ≥ 1 edge OR is explicitly tagged as `standalone` | Write-time check |
| GI-002 | No dangling edges — both source and target must exist | Write-time check |
| GI-003 | Acyclic for all non-cyclic relation types | Write-time cycle detection |
| GI-004 | Edge count reflects node count — adding a node triggers relationship requirement notification | Event |
| GI-005 | Mirror edges for symmetric relations — auto-created and auto-deleted | Relationship Engine |

---

## Graph Storage Format

```yaml
# graph/graph.yaml (snapshot representation)
nodes:
  KNW-KOS-ARCH-001:
    label: ARCHITECTURE
    namespace: kos
    status: CANONICAL
    version: "1.0.0"
    confidence: 0.92
    quality_score: 0.88
    out_edges: ["REL-IMPL-KOS-001", "REL-DOC-KOS-001"]
    in_edges: []

edges:
  REL-IMPL-KOS-001:
    label: implements
    source: KNW-KOS-ARCH-001
    target: KNW-KOS-REQ-001
    weight: 0.90
    direction: FORWARD
    is_transitive: false
    is_symmetric: false
```

---

## Graph Projections

A **graph projection** is a filtered view of the full graph:

| Projection | Nodes Included | Edges Included |
|------------|----------------|----------------|
| `dependency_graph` | all | `depends_on`, `imports` |
| `traceability_graph` | REQ, ARCH, MOD, SVC, API, TEST, DEPLOY | `implements`, `tests`, `verifies` |
| `inheritance_graph` | all | `extends`, `inherits` |
| `quality_graph` | all | `tests`, `benchmarks`, `verifies` |
| `product_graph` | PRODUCT, AGENT, RUNTIME, SERVICE | `depends_on`, `uses`, `contains` |
| `knowledge_graph` | ARCHITECTURE, DECISION, REQUIREMENT | `documents`, `implements`, `references` |

---

## Subgraph Extraction

```
SUBGRAPH(root_id, relation_types, direction, depth):
  1. BFS from root_id
  2. Include only edges matching relation_types + direction
  3. Stop at depth
  4. Return: SubGraph{nodes: set, edges: set}

COMPLEXITY: O(min(N, b^d)) where b=branching factor, d=depth
```

---

## Incremental Graph Updates

```
ADD_NODE(obj):
  1. Create KnowledgeNode from obj
  2. Insert into node index
  3. Emit knowledge.graph.node_added event
  4. Check GI-001 (isolated node alert if no edges)

ADD_EDGE(relation):
  1. Validate GI-002 (both endpoints exist)
  2. Check GI-003 (cycle detection for non-cyclic types)
  3. Create KnowledgeEdge
  4. Append to source.out_edges and target.in_edges
  5. If symmetric: auto-create mirror edge
  6. Update transitive closure (incremental, not full rebuild)
  7. Emit knowledge.graph.edge_added event

REMOVE_NODE(node_id):
  1. Mark node as TOMBSTONED (not deleted)
  2. Mark all edges as DEPRECATED
  3. Emit knowledge.graph.node_tombstoned
  4. Trigger GI-002 validation for affected consumers

INCREMENTAL REBUILD: O(1) for single node/edge — no full rebuild needed
```

---

## Graph Events

| Event | Trigger |
|-------|---------|
| `knowledge.graph.node_added` | New node inserted |
| `knowledge.graph.node_tombstoned` | Node removed |
| `knowledge.graph.edge_added` | New edge created |
| `knowledge.graph.edge_deprecated` | Edge deprecated |
| `knowledge.graph.cycle_detected` | Cycle found in acyclic relation type |
| `knowledge.graph.isolated_node` | Node has no edges after GI-001 check |

---

## Knowledge Graph Protocol

```python
class KnowledgeGraph(Protocol):
    def add_node(self, obj: KnowledgeObject) -> None: ...
    def add_edge(self, relation: KnowledgeRelation) -> None: ...
    def get_node(self, node_id: str) -> KnowledgeNode: ...
    def get_edge(self, edge_id: str) -> KnowledgeEdge: ...
    def get_out_edges(self, node_id: str, rel_type: RelationType | None) -> list[KnowledgeEdge]: ...
    def get_in_edges(self, node_id: str, rel_type: RelationType | None) -> list[KnowledgeEdge]: ...
    def get_projection(self, projection: str) -> KnowledgeGraph: ...
    def get_subgraph(self, root: str, types: list[RelationType], depth: int) -> KnowledgeGraph: ...
    def check_invariants(self) -> list[InvariantViolation]: ...
    def node_count(self) -> int: ...
    def edge_count(self) -> int: ...
    def export_yaml(self) -> str: ...
```

---

## Performance Targets

| Operation | P50 | P99 | Scale |
|-----------|-----|-----|-------|
| Add node | < 1ms | < 5ms | 1M nodes |
| Add edge | < 1ms | < 5ms | 10M edges |
| Single-hop traversal | < 1ms | < 5ms | any |
| 3-hop traversal | < 10ms | < 50ms | any |
| Full graph load (1M nodes) | < 30s | < 120s | cold start |
| Incremental update | < 5ms | < 20ms | any |

---

## Cross-References

- Relationship types on edges → `07-RELATIONSHIP-TYPES`
- Graph algorithms → `25-GRAPH-ALGORITHMS`
- Graph indexes → `26-GRAPH-INDEXES`
- Traversal in traceability → `13-TRACEABILITY`
- Graph included in snapshots → `09-SNAPSHOT-ENGINE`
