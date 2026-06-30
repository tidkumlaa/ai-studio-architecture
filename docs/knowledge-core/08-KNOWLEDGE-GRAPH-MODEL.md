---
knowledge_id: KA-SPEC-008
version: "1.0.0"
status: approved
owner: Chief Knowledge Architect
phase: 2.0D.0.5
created: 2026-06-29
review_date: 2026-12-29
canonical: true
domain: DOM-KNOWLEDGE
capability: knowledge-core
type: specification
implements:
  - KA-SPEC-001
depends_on:
  - id: KA-SPEC-002
    reason: "Nodes in the graph are typed KnowledgeObjects"
  - id: KA-SPEC-004
    reason: "Graph nodes are addressable via Knowledge URIs"
---

# Knowledge Graph Model

## A Property Graph Model for Architecture Knowledge

---

## 1. Purpose

The Knowledge Graph is the structural backbone of the knowledge system. It represents all KnowledgeObjects as nodes and all relationships as typed, directed edges. Operations that require traversal — impact analysis, dependency resolution, traceability, duplicate detection — all operate on this graph.

The Knowledge Graph is a **labeled property graph** (LPG): nodes and edges both carry typed properties.

---

## 2. Graph Formal Model

### 2.1 Definition

```
G = (V, E, λ_V, λ_E, P_V, P_E)

Where:
  V         = set of vertices (KnowledgeObjects)
  E         = set of directed edges (relationships)
  λ_V : V → KnowledgeType      (vertex type labeling function)
  λ_E : E → RelationshipType   (edge type labeling function)
  P_V : V → Map[String, Any]   (vertex property map)
  P_E : E → Map[String, Any]   (edge property map)
```

### 2.2 Vertex (Node)

Every vertex represents one KnowledgeObject.

```
Vertex v = {
  id:         KnowledgeID         # Primary key
  type:       KnowledgeType       # Node type label
  properties: {                   # From KnowledgeObject metadata
    status, version, owner,
    canonical, health_score,
    confidence, domain, capability,
    created, review_date, ...
  }
}
```

### 2.3 Edge (Relationship)

Every edge represents one declared relationship.

```
Edge e = {
  id:         EdgeID              # UUID — stable but not user-facing
  from:       KnowledgeID         # Source vertex
  to:         KnowledgeID         # Target vertex
  type:       RelationshipType    # Edge type label
  direction:  forward | reverse   # Canonical direction
  properties: {
    reason?,                      # Human-readable reason
    criticality?,                 # required | optional | enhances
    verified?,                    # Boolean — for implementation edges
    last_verified?,               # Date
    confidence?                   # high | medium | low
  }
}
```

---

## 3. Graph Vertex Types

| Vertex Type | Count (estimated) | Description |
|-------------|------------------|-------------|
| DocumentObject | ~250 | Markdown knowledge documents |
| CapabilityObject | 10 | Capability structural nodes |
| DomainObject | 10 | Domain structural nodes |
| ProductObject | 3 | Product nodes |
| ModuleObject | ~200 | Platform code modules (referenced) |
| ProviderObject | ~10 | AI provider nodes |
| AgentObject | ~5 | Agent nodes |
| WorkflowObject | ~20 | Workflow definition nodes |

---

## 4. Graph Edge Types

### 4.1 Knowledge Relationships (Document → Document)

| Edge Type | Cardinality | Description |
|-----------|------------|-------------|
| `implements` | Many → Many | A implements standard B |
| `depends_on` | Many → Many | A requires B to exist |
| `supersedes` | One → One | A replaces B (chain) |
| `references` | Many → Many | A cites B for context |
| `extends` | Many → One | A adds to B |

### 4.2 Artifact Relationships (Document → Code)

| Edge Type | Direction | Description |
|-----------|-----------|-------------|
| `implemented_by` | Doc → Module | Architecture is implemented by code |
| `tested_by` | Doc → Test | Architecture is tested by test suite |
| `specified_by` | API → Doc | API is specified in document |

### 4.3 Structural Relationships (Capability → *)

| Edge Type | Direction | Description |
|-----------|-----------|-------------|
| `contains` | Capability → Document | A capability owns documents |
| `depends_on` | Capability → Capability | Capability structural dependency |
| `consumed_by` | Capability → Product | Product uses capability |
| `produces_event` | Capability → Event | Capability emits this event |
| `consumes_event` | Capability → Event | Capability reacts to this event |
| `stores_in` | Capability → DB | Capability owns this database |

### 4.4 Decision Relationships (ADR → *)

| Edge Type | Direction | Description |
|-----------|-----------|-------------|
| `decided_by` | Doc → ADR | Document's design was decided by ADR |
| `supersedes` | ADR → ADR | ADR supersedes an older ADR |
| `affects` | ADR → Doc | ADR impacts this document |

### 4.5 Intelligence Relationships

| Edge Type | Direction | Description |
|-----------|-----------|-------------|
| `wraps` | Capability → Provider | Capability wraps this provider |
| `executes_with` | Workflow → Agent | Workflow uses this agent |
| `uses_prompt` | Agent → Prompt | Agent uses this prompt |

---

## 5. Graph Invariants

| Invariant | Formal Rule |
|-----------|------------|
| **G1 — No isolated vertices** | Every approved vertex has degree ≥ 1 (except domain nodes) |
| **G2 — No self-loops** | ∀ e ∈ E: e.from ≠ e.to |
| **G3 — No supersession cycles** | `supersedes` edges form a DAG (directed acyclic graph) |
| **G4 — No duplicate edges** | ∀ e1, e2: (e1.from, e1.to, e1.type) ≠ (e2.from, e2.to, e2.type) |
| **G5 — Type compatibility** | e.type must be valid for (λ_V(e.from), λ_V(e.to)) per the type system |
| **G6 — Reference integrity** | ∀ e ∈ E: e.from ∈ V ∧ e.to ∈ V |
| **G7 — Single canonical per concept** | For any (capability, type), at most one vertex has canonical=true |

---

## 6. Graph Algorithms

### 6.1 Depth-First Traversal

Used for: impact analysis, dependency resolution, supersession chain following.

```
ALGORITHM DFS(start: KnowledgeID, edge_types: RelationshipType[],
              direction: forward|reverse, max_depth: Integer)
  RETURNS: List[TraversalPath]

  visited = Set()
  stack = [(start, [], 0)]
  results = []

  WHILE stack not empty:
    (current, path, depth) = stack.pop()
    IF current in visited: CONTINUE
    IF depth > max_depth: CONTINUE
    visited.add(current)

    neighbors = GRAPH.neighbors(current, edge_types, direction)
    FOR each (neighbor, edge) in neighbors:
      new_path = path + [Edge(current, neighbor, edge.type)]
      results.append(new_path)
      stack.push((neighbor, new_path, depth + 1))

  RETURN results

Complexity: O(V + E) for unlimited depth
```

### 6.2 Strongly Connected Component Detection

Used for: cycle detection in `depends_on` and `supersedes` edges.

```
ALGORITHM Tarjan_SCC(G)
  RETURNS: List[List[KnowledgeID]]  -- each component

-- Standard Tarjan's algorithm applied to relevant edge types
-- Cycles in depends_on → architecture violation (circular dependency)
-- Cycles in supersedes → registry corruption

Complexity: O(V + E)
```

### 6.3 Shortest Path

Used for: understanding the minimal relationship chain between two KnowledgeObjects.

```
ALGORITHM BFS_shortest_path(start, end, edge_types)
  RETURNS: List[Edge] | null

-- Standard BFS on the edge-type-filtered graph

Complexity: O(V + E)
```

### 6.4 Reachability (Orphan Detection)

Used for: identifying vertices that cannot be reached from known entry points.

```
ALGORITHM reachability_from(roots: KnowledgeID[], edge_types)
  RETURNS: Set[KnowledgeID]  -- all reachable vertices

-- Multi-source BFS from all root nodes
-- Vertices not in the reachable set are orphans

Complexity: O(V + E)
```

---

## 7. Graph Storage Model

### 7.1 In-Memory Representation

The knowledge graph is held in memory during index generation and query execution:

```
Graph {
  vertices: HashMap<KnowledgeID, Vertex>
  edges:    HashMap<EdgeID, Edge>
  adj_out:  HashMap<KnowledgeID, List<EdgeID>>  # Outbound adjacency list
  adj_in:   HashMap<KnowledgeID, List<EdgeID>>  # Inbound adjacency list
  by_type:  HashMap<KnowledgeType, Set<KnowledgeID>>
  by_cap:   HashMap<CapabilityID, Set<KnowledgeID>>
}
```

### 7.2 Serialized Representation

The graph is serialized to `architecture/graph.json` after index generation:

```json
{
  "generated": "2026-06-29T14:00:00Z",
  "vertex_count": 247,
  "edge_count": 892,
  "vertices": {
    "KA-ARCH-001": {
      "type": "architecture",
      "status": "approved",
      "health_score": 82,
      ...
    }
  },
  "edges": [
    {
      "from": "KA-ARCH-001",
      "to": "KA-ARCH-005",
      "type": "depends_on",
      "properties": { "reason": "Provider delegation", "criticality": "required" }
    }
  ]
}
```

---

## 8. Graph Health Dimensions

The graph itself has health properties:

| Metric | Definition | Target |
|--------|-----------|--------|
| **Connectivity** | % of vertices with degree ≥ 2 | > 80% |
| **Orphan Ratio** | % of vertices with degree 0 | < 5% |
| **SCC Count** | Number of strongly connected components | 0 (for depends_on) |
| **Avg Path Length** | Average shortest path between capabilities | < 4 |
| **Reference Integrity** | % of edges with valid endpoints | 100% |
| **Relationship Completeness** | % of expected edges declared | > 85% |

---

## References

- [07-KNOWLEDGE-QUERY-LANGUAGE.md](07-KNOWLEDGE-QUERY-LANGUAGE.md) — KQL traverses this graph
- [10-KNOWLEDGE-INDEX-ENGINE.md](10-KNOWLEDGE-INDEX-ENGINE.md) — Engine that builds and maintains the graph
- [16-DATA-STRUCTURE-CATALOG.md](16-DATA-STRUCTURE-CATALOG.md) — Data structures for graph storage
- [17-ALGORITHMS-CATALOG.md](17-ALGORITHMS-CATALOG.md) — Full algorithm specifications
- [18-COMPLEXITY-GUIDE.md](18-COMPLEXITY-GUIDE.md) — Complexity analysis for graph operations
