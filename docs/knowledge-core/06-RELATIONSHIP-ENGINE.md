# KNW-KC-ARCH-006 — Relationship Engine

**Phase:** 3.0C  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Relationships are first-class Knowledge Objects. Every connection between two Knowledge Objects is an explicit, typed, versioned, evidence-bearing record. Implicit relationships are forbidden.

---

## KnowledgeRelation Schema

Every relationship is an instance of `KnowledgeRelation`:

```yaml
# KnowledgeRelation fields
relation_id: string              # "REL-{source_type}-{rel_type}-{target_type}-{NNN}"
relation_type: RelationType      # one of 24 types (07-RELATIONSHIP-TYPES)
source_id: string                # knowledge_id of source object
target_id: string                # knowledge_id of target object
direction: FORWARD|INVERSE|BIDIRECTIONAL

# RelationIdentity
created_at: datetime
created_by: string
updated_at: datetime
version: string                  # semver of this relation record

# RelationMetadata
label: string                    # human label, e.g. "platform.service.auth depends_on platform.kernel"
description: string
context: string                  # why does this relationship exist?
strength: WEAK|NORMAL|STRONG|CRITICAL
is_transitive: bool              # if true, A→B and B→C implies A→C
is_symmetric: bool               # if true, A→B implies B→A

# RelationEvidence
evidence_ids: list[string]       # EvidenceRecord knowledge_ids
confidence: float                # 0.0..1.0
verification_status: UNVERIFIED|VERIFIED|DISPUTED

# RelationQuality
completeness: float              # 0.0..1.0
freshness: float
last_verified_at: datetime | null

# RelationVersion
parent_relation_id: string | null
history: list[RelationVersion]
is_current: bool

# RelationLifecycle
status: PROPOSED|ACTIVE|DEPRECATED|REMOVED
deprecated_at: datetime | null
removal_reason: string
```

---

## Relation Identity Rules

### RE-001 — First-Class Objects
Every relationship is stored as an independent record in the Relationship Registry. It is not an embedded field of the source or target object.

### RE-002 — Unique Relation ID
`relation_id` is unique across all relationships. Two objects may have multiple relationships of different types but not two relationships of the same type.

### RE-003 — No Dangling Relationships
Both `source_id` and `target_id` must resolve to existing, non-tombstoned objects at write time.

### RE-004 — No Self-Relationships
`source_id != target_id` is enforced. An object may not relate to itself.

### RE-005 — Versioned Relationships
When a relationship changes (confidence update, evidence added, label change), a new version record is appended. The previous version is preserved for history.

### RE-006 — Lifecycle Coupling
When a source or target object is DEPRECATED, the relationship is automatically transitioned to DEPRECATED (not REMOVED). REMOVED requires explicit human action.

---

## Traversal Model

```
TRAVERSE(start_id, relation_type, direction, depth_limit):
  1. Load start node from Object Registry
  2. Load all relations where source_id == start_id AND relation_type matches
  3. For INVERSE: load all relations where target_id == start_id
  4. For BIDIRECTIONAL: both
  5. Filter by status == ACTIVE
  6. Recurse if depth_limit > 0 (DFS)
  7. Return: list[TraversalResult{node, relation, depth}]
  8. Cycle detection: track visited node IDs

TRAVERSAL COMPLEXITY: O(N + E) where N=nodes, E=edges in subgraph
```

---

## Bidirectional Navigation Table

| Relationship Type | Forward (A→B) | Inverse (B→A) |
|-------------------|--------------|---------------|
| `depends_on` | A depends on B | B is depended on by A |
| `implements` | A implements B | B is implemented by A |
| `extends` | A extends B | B is extended by A |
| `tests` | A tests B | B is tested by A |
| `generates` | A generates B | B is generated from A |
| `contains` | A contains B | B is contained in A |
| `references` | A references B | B is referenced by A |
| `deprecates` | A deprecates B | B is deprecated by A |

Full table: `07-RELATIONSHIP-TYPES`

---

## Relation Propagation Rules

### Transitive Relations
For types where `is_transitive: true`:
```
A depends_on B AND B depends_on C
  → A transitively depends_on C
  → Stored as explicit relation with confidence = min(conf_AB, conf_BC)
  → Label: "transitive via {B}"
```

### Symmetric Relations
For types where `is_symmetric: true`:
```
A related_to B
  → B related_to A (auto-created mirror)
  → Mirror shares same evidence, confidence, version
```

---

## Relationship Index Format

```yaml
# relationship-index.yaml
relations:
  - relation_id: "REL-PLAT-SVC-DEPENDS-KERN-001"
    relation_type: depends_on
    source_id: "KNW-PLAT-SVC-001"
    target_id: "KNW-PLAT-KERN-001"
    direction: FORWARD
    strength: CRITICAL
    confidence: 0.95
    status: ACTIVE
    version: "1.0.0"
    created_at: "2026-01-01T00:00:00Z"
    context: "Auth service calls kernel contract interfaces"
```

---

## Relationship Protocol (Python)

```python
class KnowledgeRelation(Protocol):
    relation_id: str
    relation_type: RelationType
    source_id: str
    target_id: str
    direction: RelationDirection
    strength: RelationStrength
    confidence: float
    status: RelationStatus
    is_transitive: bool
    is_symmetric: bool
    evidence_ids: list[str]

    def traverse_forward(self) -> list[KnowledgeObject]: ...
    def traverse_inverse(self) -> list[KnowledgeObject]: ...
    def get_evidence(self) -> list[EvidenceRecord]: ...
    def deprecate(self, reason: str) -> None: ...

class RelationshipEngine(Protocol):
    def create(self, rel: KnowledgeRelation) -> str: ...
    def get(self, relation_id: str) -> KnowledgeRelation: ...
    def find(self, source_id: str, rel_type: RelationType) -> list[KnowledgeRelation]: ...
    def traverse(self, start_id: str, depth: int) -> list[TraversalResult]: ...
    def check_cycles(self, rel_type: RelationType) -> list[list[str]]: ...
    def compute_closure(self, start_id: str, rel_type: RelationType) -> set[str]: ...
```

---

## Error Conditions

| Code | Condition |
|------|-----------|
| `REL-001` | Dangling source or target reference |
| `REL-002` | Duplicate relationship (same type, same pair) |
| `REL-003` | Self-relationship |
| `REL-004` | Cycle detected in non-cyclic relation type |
| `REL-005` | Conflicting direction (FORWARD + INVERSE for same relation) |

---

## Cross-References

- Relationship types → `07-RELATIONSHIP-TYPES`
- Graph traversal → `24-KNOWLEDGE-GRAPH-MODEL`
- Algorithms → `25-GRAPH-ALGORITHMS`
- Evidence → `10-EVIDENCE-ENGINE`
