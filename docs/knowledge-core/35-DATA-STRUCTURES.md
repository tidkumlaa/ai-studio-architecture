# KNW-KC-ARCH-035 — Data Structures

**Phase:** 3.0C  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

This document defines the canonical data structures used throughout the Knowledge Core. Every structure has a Python dataclass definition, field descriptions, and invariants.

---

## DS-01: KnowledgeRef

A lightweight reference to a Knowledge Object (avoids embedding full objects).

```python
@dataclass
class KnowledgeRef:
    knowledge_id: str           # KNW-{DOMAIN}-{TYPE}-{NNN}
    canonical_name: str         # kos.arch.vision
    knowledge_uri: str          # knw://kos/arch/vision
    version: str                # pinned version or "latest"
    object_type: KnowledgeObjectType
    confidence: float           # 0.0..1.0
```

---

## DS-02: RelationRef

```python
@dataclass
class RelationRef:
    relation_id: str
    relation_type: RelationType
    source_id: str
    target_id: str
    confidence: float
    strength: str               # WEAK|NORMAL|STRONG|CRITICAL
```

---

## DS-03: OwnerRef

```python
@dataclass
class OwnerRef:
    owner_id: str               # "team:arch-board" or "user:jane.doe"
    owner_type: str             # TEAM|USER|SYSTEM
    display_name: str
    email: str | None
```

---

## DS-04: DependencyRef

```python
@dataclass
class DependencyRef:
    knowledge_id: str
    dep_type: str               # HARD|SOFT|RUNTIME|BUILD|TEST
    version_constraint: str     # ">=1.0.0" | "==1.2.3" | "*"
    optional: bool
    reason: str
```

---

## DS-05: VersionRecord

```python
@dataclass
class VersionRecord:
    knowledge_id: str
    version: str                # semver
    parent_version: str | None
    change_type: str            # MAJOR|MINOR|PATCH
    change_summary: str
    changed_by: str
    changed_at: datetime
    checksum: str
    content_snapshot_ref: str
    is_published: bool
    breaking_changes: list[str]
    migration_notes: str
```

---

## DS-06: TraversalResult

```python
@dataclass
class TraversalResult:
    node_id: str
    depth: int
    via_relation: RelationRef | None
    path: list[str]             # sequence of knowledge_ids from root to this node
    confidence: float           # product of edge confidences along path
```

---

## DS-07: ImpactReport

```python
@dataclass
class ImpactReport:
    object_id: str
    blast_radius: int
    severity: str               # LOW|MEDIUM|HIGH|CRITICAL
    affected_objects: list[KnowledgeRef]
    affected_by_type: dict[str, int]
    affected_by_namespace: dict[str, int]
    critical_path: list[str]    # highest-impact dependency chain
    computed_at: datetime
    confidence: float
```

---

## DS-08: CoverageReport

```python
@dataclass
class CoverageReport:
    namespace: str
    coverage_type: str          # TRACEABILITY|QUALITY|EVIDENCE
    total_objects: int
    covered_objects: int
    coverage_score: float       # covered / total
    gaps: list[TraceabilityGap]
    computed_at: datetime

@dataclass
class TraceabilityGap:
    requirement_id: str
    gap_type: str               # UNARCHITECTED|UNIMPLEMENTED|UNTESTED|UNDEPLOYED
    missing_link_type: str
    severity: str               # LOW|MEDIUM|HIGH|CRITICAL
```

---

## DS-09: RegistryStats

```python
@dataclass
class RegistryStats:
    total_objects: int
    by_status: dict[str, int]
    by_type: dict[str, int]
    by_namespace: dict[str, int]
    total_relations: int
    orphaned_objects: int
    stale_objects: int
    avg_quality_score: float
    computed_at: datetime
```

---

## DS-10: GraphStats

```python
@dataclass
class GraphStats:
    node_count: int
    edge_count: int
    avg_out_degree: float
    max_out_degree: int
    connected_components: int
    has_cycles: bool            # for non-cyclic relation types
    largest_cycle: list[str]    # knowledge_ids in largest detected cycle
    diameter: int               # longest shortest path
    computed_at: datetime
```

---

## DS-11: SearchIndexStats

```python
@dataclass
class SearchIndexStats:
    total_indexed: int
    full_text_index_size_bytes: int
    vector_index_size_bytes: int
    last_full_rebuild: datetime
    incremental_updates_since_rebuild: int
    avg_search_ms: float
    index_health: str           # HEALTHY|DEGRADED|REBUILDING
```

---

## DS-12: BulkRegistrationResult

```python
@dataclass
class BulkRegistrationResult:
    total_attempted: int
    succeeded: int
    failed: int
    failures: list[tuple[str, str]]   # (knowledge_id, error_message)
    duration_ms: float
    rolled_back: bool
```

---

## DS-13: KQLResult

```python
@dataclass
class KQLResult:
    query: str
    rows: list[dict]
    total_count: int
    returned_count: int
    execution_ms: float
    index_hits: list[str]
    plan: str
    warnings: list[str]
```

---

## DS-14: QualityScoreRecord

```python
@dataclass
class QualityScoreRecord:
    knowledge_id: str
    computed_at: datetime
    qd_1_completeness: float
    qd_2_consistency: float
    qd_3_coverage: float
    qd_4_freshness: float
    qd_5_evidence: float
    qd_6_confidence: float
    qd_7_usage: float
    qd_8_reasoning: float
    qd_9_health: float
    weighted_score: float
    overall_score: float
    alerts: list[str]
```

---

## DS-15: SnapshotScope

```python
@dataclass
class SnapshotScope:
    scope_type: str             # SINGLE_OBJECT|NAMESPACE|FULL_REGISTRY|CUSTOM
    object_ids: list[str] | None
    namespace: str | None
    include_relations: bool = True
    include_versions: bool = False
```

---

## Immutability Rules

| Structure | Immutable After |
|-----------|----------------|
| `VersionRecord` | `is_published = True` |
| `KnowledgeRef` | Once embedded in a CANONICAL object |
| `SnapshotScope` | Once snapshot is CREATED |
| `ImpactReport` | 24 hours (then stale — recompute) |
| All others | Mutable unless embedded in published version |

---

## Cross-References

- These structures used throughout → all documents in this set
- Pydantic models in Phase 3.0B → `platform/knowledge_runtime/objects/`
- Performance data for structures → `37-PERFORMANCE-BUDGET`
