---
knowledge_id: KA-SPEC-016
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
  - KA-SPEC-015
depends_on:
  - id: KA-SPEC-001
    reason: "Data structures model KnowledgeObjects"
  - id: KA-SPEC-008
    reason: "Graph data structures model the Knowledge Graph"
---

# Data Structure Catalog

## All Data Structures Used in the Knowledge System

---

## 1. Core Object Structures

### DS-001 — KnowledgeObject

**Purpose:** Primary entity. Every document is represented as a KnowledgeObject.

**Implementation:** Frozen dataclass (immutable identity fields, mutable content fields via copy-on-write)

```python
@dataclass
class KnowledgeObject:
    # Identity (immutable)
    knowledge_id: str           # Pattern: KA-[TYPE]-[NNN]
    type: KnowledgeType         # Immutable after creation

    # Content (versioned)
    version: str                # Semantic version
    title: str
    body: str                   # Markdown content

    # Classification
    domain: str
    capability: str

    # Ownership
    owner: str
    created: date
    review_date: Optional[date]
    phase: str

    # Lifecycle
    status: LifecycleState
    canonical: bool
    deprecated_date: Optional[date] = None
    superseded_by: Optional[str] = None

    # Quality
    confidence: ConfidenceLevel
    coverage: CoverageMap

    # Relationships (declared)
    relationships: Relationships

    # Evidence
    evidence: List[EvidenceRecord]

    # Tags
    tags: List[str]

    # Computed (not stored in source — derived by KIE)
    health_score: Optional[int] = None
    health_computed_at: Optional[datetime] = None

Space: O(C + R + E) where C=content size, R=relationship count, E=evidence count
```

### DS-002 — Relationships

**Purpose:** Container for all relationship declarations of a KnowledgeObject.

```python
@dataclass
class Relationships:
    implements: List[str]          # Knowledge IDs
    depends_on: List[Dependency]
    supersedes: Optional[str]
    implemented_by: List[ArtifactRef]
    tested_by: List[TestRef]
    related_apis: List[RelatedRef]
    related_database: List[RelatedRef]
    related_events: List[RelatedRef]
    related_adr: List[RelatedRef]
    related_products: List[ProductRef]
    consumed_by: List[ConsumerRef]
    decided_by: List[DecisionRef]
```

### DS-003 — EvidenceRecord

```python
@dataclass
class EvidenceRecord:
    type: EvidenceType          # source_file | test | schema | config | log | adr
    ref: str                    # File path or URI
    line: Optional[int]
    verified: bool
    verified_by: Optional[str]
    last_verified: Optional[date]
    notes: Optional[str]
    confidence: Optional[ConfidenceLevel]
    expiry_days: int = 90       # Default expiry
    state: EvidenceState = EvidenceState.PROPOSED

    @property
    def is_expired(self) -> bool:
        if self.last_verified is None: return True
        return (date.today() - self.last_verified).days > self.expiry_days
```

### DS-004 — HealthScore

```python
@dataclass
class HealthScore:
    total: int                  # 0–100
    structural: int             # 0–25
    relationship: int           # 0–20
    freshness: int              # 0–20
    evidence: int               # 0–15
    coverage: int               # 0–10
    consistency: int            # 0–10
    computed_at: datetime
    violations: List[str]       # Rules violated
```

### DS-005 — CoverageMap

```python
@dataclass
class CoverageMap:
    architecture: int = 0       # 0–100
    implementation: int = 0     # 0–100
    testing: int = 0            # 0–100
    desktop: int = 0            # 0–100 (0 = not applicable)
```

---

## 2. Graph Data Structures

### DS-006 — KnowledgeGraph

**Purpose:** The property graph of all KnowledgeObjects and their relationships.

```python
@dataclass
class KnowledgeGraph:
    vertices: Dict[str, KnowledgeObject]        # id → object
    edges: Dict[str, Edge]                      # edge_id → edge
    adj_out: Dict[str, List[str]]               # id → [edge_id]
    adj_in:  Dict[str, List[str]]               # id → [edge_id]

    # Secondary indexes for performance
    by_type: Dict[KnowledgeType, Set[str]]      # type → {id}
    by_capability: Dict[str, Set[str]]          # cap → {id}
    by_domain: Dict[str, Set[str]]              # domain → {id}
    by_status: Dict[LifecycleState, Set[str]]   # status → {id}

    @property
    def vertex_count(self) -> int: return len(self.vertices)

    @property
    def edge_count(self) -> int: return len(self.edges)

Space: O(V + E) where V=vertices, E=edges
Query time: O(1) for vertex lookup, O(k) for neighbor listing
```

### DS-007 — Edge

```python
@dataclass(frozen=True)
class Edge:
    id: str                         # UUID
    from_id: str                    # Source KnowledgeID
    to_id: str                      # Target KnowledgeID
    type: RelationshipType          # Edge type label
    properties: EdgeProperties      # Edge-specific properties
```

### DS-008 — EdgeProperties

```python
@dataclass
class EdgeProperties:
    reason: Optional[str] = None
    criticality: Optional[str] = None    # required | optional | enhances
    verified: Optional[bool] = None
    last_verified: Optional[date] = None
    confidence: Optional[ConfidenceLevel] = None
    coverage_percent: Optional[int] = None
```

---

## 3. Index Data Structures

### DS-009 — KnowledgeRegistry

**Purpose:** O(1) lookup by Knowledge ID. The single authoritative store.

```python
class KnowledgeRegistry:
    _store: Dict[str, KnowledgeObject]    # id → object
    _by_file: Dict[Path, str]             # file path → id
    _by_status: Dict[str, Set[str]]       # status → {id}
    _next_available: Dict[str, str]       # type_code → next_id

    def get(self, id: str) → Optional[KnowledgeObject]  # O(1)
    def put(self, obj: KnowledgeObject) → None           # O(1)
    def by_file(self, path: Path) → Optional[str]        # O(1)
    def all_active(self) → List[KnowledgeObject]         # O(n)
    def next_id(self, type_code: str) → str              # O(1)
```

### DS-010 — CapabilityIndex

**Purpose:** Navigate all documents within a capability.

```python
class CapabilityIndex:
    # capability → type → canonical_id
    _canonical: Dict[str, Dict[KnowledgeType, str]]
    # capability → type → [all ids]
    _all: Dict[str, Dict[KnowledgeType, List[str]]]
    # capability → metadata (from index.yaml)
    _meta: Dict[str, CapabilityMeta]

    def canonical(self, cap: str, type: KnowledgeType) → Optional[str]  # O(1)
    def all_in_capability(self, cap: str) → Dict[KnowledgeType, List[str]]  # O(1)
    def missing_required(self, cap: str) → List[str]  # O(1)
```

### DS-011 — HealthIndex

**Purpose:** Fast health score retrieval and stale/critical document listing.

```python
class HealthIndex:
    _scores: Dict[str, HealthScore]
    _stale: SortedList[(int, str)]          # (days_overdue, id) — sorted desc
    _critical: SortedList[(int, str)]       # (score, id) — sorted asc
    _by_capability: Dict[str, int]          # cap → aggregate score
    _by_domain: Dict[str, int]              # domain → aggregate score

    def score(self, id: str) → Optional[HealthScore]  # O(1)
    def stale_top(self, n: int) → List[str]            # O(n)
    def critical_top(self, n: int) → List[str]         # O(n)
    def repository_score(self) → int                   # O(1)
```

---

## 4. Supporting Structures

### DS-012 — Diagnostic

```python
@dataclass
class Diagnostic:
    rule_id: str                    # RULE-M001, etc.
    severity: Severity              # ERROR | WARNING | INFO
    object_id: Optional[str]        # Affected knowledge ID
    file: Optional[Path]            # Affected file
    line: Optional[int]             # Affected line
    message: str
    remediation: str
    auto_fixable: bool = False
```

### DS-013 — QueryResult

```python
@dataclass
class QueryResult:
    query: str
    executed_at: datetime
    duration_ms: int
    total_results: int
    results: List[Dict]             # Projected fields or full objects
    paths: Optional[List[List[Edge]]] = None  # For TRAVERSE queries
```

### DS-014 — TraversalPath

```python
@dataclass
class TraversalPath:
    nodes: List[str]                # KnowledgeID sequence
    edges: List[Edge]               # Edge sequence
    depth: int

    @property
    def start(self) → str: return self.nodes[0]

    @property
    def end(self) → str: return self.nodes[-1]
```

---

## 5. Data Structure Selection Guide

| Need | Use |
|------|-----|
| Object identity lookup by ID | `KnowledgeRegistry` (DS-009) — O(1) |
| All documents in a capability | `CapabilityIndex` (DS-010) — O(1) |
| Canonical document for a concept | `CapabilityIndex.canonical()` — O(1) |
| Relationship traversal | `KnowledgeGraph` (DS-006) — O(V+E) DFS |
| Health scores | `HealthIndex` (DS-011) — O(1) per lookup |
| Stale documents | `HealthIndex.stale_top()` — O(n) |
| Full graph analysis | `KnowledgeGraph` with `networkx` — varies |

---

## References

- [17-ALGORITHMS-CATALOG.md](17-ALGORITHMS-CATALOG.md) — Algorithms that operate on these structures
- [18-COMPLEXITY-GUIDE.md](18-COMPLEXITY-GUIDE.md) — Complexity of operations on these structures
- [10-KNOWLEDGE-INDEX-ENGINE.md](10-KNOWLEDGE-INDEX-ENGINE.md) — Engine that populates these structures
- [15-COMPUTER-SCIENCE-STANDARDS.md](15-COMPUTER-SCIENCE-STANDARDS.md) — Implementation standards
