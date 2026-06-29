---
knowledge_id: KNW-KOS-ARCH-005
title: "KOS Knowledge Graph"
status: AUTHORITATIVE
phase: "3.0A"
version: "1.0.0"
created: "2026-06-30"
purpose: "Specify the Knowledge Graph model: nodes, edges, traversal rules, and relationship types"
canonical_source: "architecture/docs/kos/05-KNOWLEDGE-GRAPH.md"
dependencies:
  - "04-OBJECT-MODEL.md"
related_documents:
  - "06-KNOWLEDGE-COMPILER.md"
  - "08-QUERY-ENGINE.md"
  - "09-REASONING-ENGINE.md"
acceptance_criteria:
  - "All relationship types are named and directional"
  - "Bidirectional traversal is specified"
  - "Canonical dependency chain is documented"
verification_checklist:
  - "[ ] Node types correspond to KnowledgeObjectType"
  - "[ ] All edge types are defined"
  - "[ ] No cycles in the core dependency chain"
future_extensions:
  - "Temporal graph: knowledge graph snapshots over time"
  - "Probabilistic edges for uncertain relationships"
---

# KOS Knowledge Graph

## Graph Model

The Knowledge Graph is a **directed labeled multigraph**:

- **Nodes:** Knowledge Objects (every entity in the ecosystem)
- **Edges:** Named, directed relationships between objects
- **Labels:** Edge type names (see below)
- **Multigraph:** Two nodes may be connected by multiple edge types simultaneously

---

## Core Dependency Chain

The primary traversal path through the graph:

```
Requirement
    │  FULFILLED_BY
    ▼
Architecture
    │  SPECIFIES
    ▼
Module
    │  IMPLEMENTS
    ▼
Service
    │  EXPOSES
    ▼
API
    │  TESTED_BY
    ▼
Test
    │  VALIDATES
    ▼
Deployment
    │  OBSERVED_AS
    ▼
Benchmark
    │  REFINES
    ▼
Requirement  (loop: observation improves requirements)
```

This chain is the basis of GR-008 (Everything Must Be Traceable).

---

## Edge Type Catalog

### Dependency Edges

| Edge Type | From | To | Meaning |
|-----------|------|----|---------|
| `DEPENDS_ON` | Any | Any | Hard dependency; if target changes, source may break |
| `OPTIONALLY_USES` | Any | Any | Soft dependency; fallback exists |
| `SUPERSEDES` | Any | Any | The source replaces the target |
| `EXTENDS` | Module | Module | Source adds capabilities to target |

### Architecture Edges

| Edge Type | From | To | Meaning |
|-----------|------|----|---------|
| `SPECIFIES` | Architecture | Module/Service/API | Architecture defines what must be built |
| `FULFILLED_BY` | Requirement | Module/Service | This implementation satisfies the requirement |
| `DECIDED_IN` | Architecture | Decision | This architecture was shaped by this decision |
| `MOTIVATED_BY` | Decision | Requirement | This decision responds to this requirement |

### Implementation Edges

| Edge Type | From | To | Meaning |
|-----------|------|----|---------|
| `IMPLEMENTS` | Module | Algorithm/Pattern | Module is the implementation of this algorithm |
| `EXPOSES` | Service/Module | API/Capability | This API/capability is accessible through this service |
| `PROVIDES` | Runtime | Capability | Runtime makes this capability available |
| `CONSUMES` | Module/Service | Capability | This module/service uses this capability |
| `REGISTERED_IN` | Any | Registry | This object appears in this registry |

### Quality Edges

| Edge Type | From | To | Meaning |
|-----------|------|----|---------|
| `TESTED_BY` | Module/Service/API | Test | This test verifies the source |
| `BENCHMARKED_BY` | Module/Service | Benchmark | This benchmark measures the source |
| `VALIDATES` | Test | Module/Service/API | What the test checks |
| `COVERS` | Test | Requirement | This test covers this requirement |

### Product Edges

| Edge Type | From | To | Meaning |
|-----------|------|----|---------|
| `USES_RUNTIME` | Product | Runtime | Product depends on this runtime |
| `USES_API` | Product | API | Product calls this API |
| `GENERATED_BY` | Prompt | Agent | Prompt is used by this agent |
| `EXECUTED_BY` | Task | Agent | Agent executes this task |
| `PLANNED_BY` | ExecutionPlan | Strategy | Plan was derived from this strategy |

### Knowledge Edges

| Edge Type | From | To | Meaning |
|-----------|------|----|---------|
| `RELATED_TO` | Any | Any | Semantic relationship (non-dependency) |
| `AUTHORED_BY` | Any | Any (owner) | Who created this object |
| `REVIEWED_BY` | Any | Any (reviewer) | Who reviewed this object |
| `GENERATED_FROM` | Any | Any | This object was compiled from the source |

---

## Canonical Knowledge Path

For any implementation artifact, the canonical path must be traceable:

```
REQ → ARCH → MODULE → SERVICE → API → TEST → DEPLOY
```

**Example — Quota Check:**
```
KNW-REQ-001 (resource quota control)
  FULFILLED_BY → KNW-PLAT-ARCH-006 (runtime model)
  SPECIFIES    → KNW-AI-MOD-quota_manager (module)
  EXPOSES      → capability:ai:quota.check:v1
  TESTED_BY    → KNW-TEST-quota-001 (test_quota_manager.py)
  VALIDATES    → KNW-REQ-001 (closes the loop)
```

---

## Graph Interface

```python
class KnowledgeGraph(Protocol):
    def add_node(self, obj: KnowledgeObject) -> None: ...
    def add_edge(self, from_id: str, to_id: str, edge_type: str) -> None: ...
    def get_node(self, knowledge_id: str) -> KnowledgeObject | None: ...
    def get_edges(self, knowledge_id: str, direction: str = "both") -> list[Edge]: ...
    def traverse(self, start_id: str, edge_types: list[str], max_depth: int = 10) -> list[KnowledgeObject]: ...
    def shortest_path(self, from_id: str, to_id: str) -> list[KnowledgeObject]: ...
    def find_cycles(self) -> list[list[str]]: ...
    def subgraph(self, root_id: str, depth: int) -> KnowledgeGraph: ...
```

---

## Graph Storage Format

```yaml
# graph_node.yaml — stored alongside each Knowledge Object
knowledge_id: KNW-AI-MOD-quota_manager
edges_outbound:
  - edge_type: IMPLEMENTS
    to: KNW-ALG-quota-check-001
  - edge_type: PROVIDES
    to: capability:ai:quota.check:v1
  - edge_type: DEPENDS_ON
    to: KNW-AI-MOD-usage_collector
edges_inbound:
  - edge_type: SPECIFIES
    from: KNW-PLAT-ARCH-006
  - edge_type: TESTED_BY
    from: KNW-TEST-quota-001
```

---

## Graph Invariants

| Invariant | Rule |
|-----------|------|
| GI-001 | No orphan nodes (every node has at least one edge) |
| GI-002 | No cycles in DEPENDS_ON edges (DAG required) |
| GI-003 | Every FULFILLED_BY edge must trace back to a Requirement |
| GI-004 | Every deployment artifact must have a VALIDATED_BY path to a Test |
| GI-005 | Every Decision must be reachable from at least one Requirement |

```bash
platform-kos graph --validate
# Checks all 5 invariants; exits non-zero if any violated
```

---

## Bidirectional Traversal

Every edge type supports reverse traversal:

| Forward | Reverse |
|---------|---------|
| `DEPENDS_ON` | `DEPENDED_ON_BY` |
| `FULFILLED_BY` | `FULFILLS` |
| `SPECIFIES` | `SPECIFIED_BY` |
| `TESTED_BY` | `TESTS` |
| `EXPOSES` | `EXPOSED_BY` |

The Knowledge Graph maintains both directions automatically.
Adding `A →DEPENDS_ON→ B` automatically creates `B ←DEPENDED_ON_BY← A`.
