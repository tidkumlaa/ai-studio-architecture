# KVF-DOC-012 — Graph Validation

**Phase:** 3.0D.1.5
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Graph Validation verifies the structural correctness and integrity of the
knowledge relationship graph. The graph connects KnowledgeObjects through
declared relationships; its correctness is foundational to graph traversal,
impact analysis, and dependency validation.

---

## Graph Properties to Validate

```
GV-PROP-1: Vertex Completeness
  Every CANONICAL and ACTIVE object is a vertex in the graph.

GV-PROP-2: Edge Correctness
  Every declared relationship corresponds to exactly one directed edge.
  Edge type matches the declared relationship type.

GV-PROP-3: Bidirectionality
  For symmetric relationship types (RELATED_TO, CO_OCCURS_WITH),
  both directions are present.

GV-PROP-4: Acyclicity (HARD dependencies)
  HARD DEPENDS_ON relationships form a DAG (no cycles).

GV-PROP-5: Referential Integrity
  Every edge target exists as a vertex.

GV-PROP-6: Relationship Count Accuracy
  For each object: index.relationship_count matches
  the actual count of edges in the graph.
```

---

## Graph Validation Checks

### GV-1: Vertex Count

```
GV-1.01: count(graph.vertices) >= count(CANONICAL + ACTIVE objects in corpus)
GV-1.02: DEPRECATED objects may optionally be included (with DEPRECATED label)
GV-1.03: count(orphan_vertices) / count(total_vertices) <= 0.05
           (at most 5% of objects have no relationships — a warning, not failure)
```

### GV-2: Edge Correctness

```
GV-2.01: For every relationship (O, type, target) in KIL:
           graph.edge(O.id, target, type) exists
GV-2.02: No edge in graph corresponds to a non-declared relationship
           (graph cannot have edges not in KIL)
GV-2.03: Edge type is from declared KOS relationship vocabulary
```

### GV-3: Bidirectionality

```
Symmetric relationship types: RELATED_TO, CO_OCCURS_WITH, ALTERNATIVE_TO

GV-3.01: For every symmetric edge (A → B, type),
           graph also has edge (B → A, same_type)
GV-3.02: Asymmetric types (DEPENDS_ON, IMPLEMENTS, EXTENDS) are NOT bidirectional
```

### GV-4: Acyclicity

```
GV-4.01: Run DFS cycle detection on HARD DEPENDS_ON edges only
GV-4.02: Expected: 0 cycles found
GV-4.03: If cycle found: report cycle path (A → B → C → A)
GV-4.04: SOFT dependencies may form cycles (not forbidden)
```

### GV-5: Referential Integrity

```
GV-5.01: For every edge target_id: target_id exists in graph.vertices
GV-5.02: Dangling references (target not in graph) = 0
GV-5.03: Cross-namespace references are allowed but must be logged
```

### GV-6: Relationship Count

```
GV-6.01: For each object O:
           O.intelligence.search.index.relationship_count == degree(O, graph)
GV-6.02: Mismatch tolerance: 0 (exact count required)
```

---

## Graph Statistics

Graph validation also produces graph health statistics:

```yaml
graph_statistics:
  vertex_count: integer
  edge_count: integer
  relationship_type_distribution:
    DEPENDS_ON: integer
    IMPLEMENTS: integer
    COMPONENT_OF: integer
    EXTENDS: integer
    RELATED_TO: integer
    ALTERNATIVE_TO: integer
    CO_OCCURS_WITH: integer
    TESTED_BY: integer
    DEPRECATED_BY: integer
    other: integer

  degree_distribution:
    mean_in_degree: float
    mean_out_degree: float
    max_degree_object: string
    isolated_vertices: integer           # degree == 0

  connected_components: integer          # ideally 1 (fully connected)
  largest_component_size: integer
  hard_dependency_dag_depth: integer     # longest dependency chain
```

---

## Graph Validation Result Schema

```yaml
graph_validation_result:
  checks_run: integer
  checks_passed: integer

  vertex_completeness: float             # GV-1
  edge_correctness: float                # GV-2
  bidirectionality_rate: float           # GV-3
  hard_dep_cycles_found: integer         # GV-4: must be 0
  dangling_references: integer           # GV-5: must be 0
  relationship_count_accuracy: float     # GV-6

  graph_statistics: GraphStatistics

  critical_failures:
    - hard_dep_cycles: [string]          # cycle paths found
    - dangling_refs: [string]            # object IDs with dangling refs
```

---

## Rules

| Rule | Statement |
|------|-----------|
| KVF-056 | Hard dependency cycles found > 0 is a CRITICAL failure — blocks all certification |
| KVF-057 | Dangling references > 0 is a CRITICAL failure — broken referential integrity |
| KVF-058 | Bidirectionality check must cover all symmetric relationship types, not just RELATED_TO |
| KVF-059 | Relationship count accuracy must be 100% — index inconsistency blocks GOLD+ |
| KVF-060 | Graph statistics must be reported in every validation run — they indicate corpus health |

---

## Cross-References

- Dependency validation → `16-DEPENDENCY-VALIDATION`
- KOS relationship types → Phase 3.0A core documentation
- Object selector (graph traversal) → Phase 3.0D.1 `05-OBJECT-SELECTOR`
- Relevance model (graph proximity) → Phase 3.0D.1 `06-RELEVANCE-MODEL`
