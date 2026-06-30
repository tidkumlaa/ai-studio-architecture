# KNW-KC-ARCH-002 — Universal Knowledge Schema

**Phase:** 3.0C  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Every Knowledge Object — regardless of type, domain, or lifecycle stage — inherits the Universal Knowledge Schema. No exceptions. No partial implementations. This schema is the atomic contract of existence in KOS.

---

## Schema Sections (16 Required)

### Section 1 — Identity
Fields: `global_id`, `knowledge_id`, `knowledge_uri`, `namespace`, `canonical_name`, `aliases`, `version`, `owner`, `domain`, `created_at`, `updated_at`, `checksum`, `fingerprint`, `lineage`

Full definition: `01-IDENTITY-ENGINE`

### Section 2 — Classification
```yaml
classification:
  object_type: KnowledgeObjectType   # 33 values defined in 3.0B
  sub_type: string                   # domain-specific subtype
  category: string                   # grouping label
  tags: list[string]                 # free-form searchable labels
  maturity: EXPERIMENTAL|BETA|STABLE|CANONICAL
  tier: 0..7                         # platform layer (0=stdlib, 7=products)
```

### Section 3 — Metadata
```yaml
metadata:
  title: string
  description: string                # ≥ 10 chars
  purpose: string
  scope: string
  constraints: list[string]
  assumptions: list[string]
  custom: dict                       # domain-specific extension fields
```

### Section 4 — Lifecycle
```yaml
lifecycle:
  status: DRAFT|REVIEW|VERIFIED|APPROVED|CANONICAL|DEPRECATED|ARCHIVED
  transition_history: list[Transition]
  review_status: PENDING|IN_REVIEW|PASSED|FAILED
  approved_by: string
  approved_at: datetime | null
  deprecated_at: datetime | null
  archived_at: datetime | null
  deprecation_reason: string
  successor_id: string | null
```

Full state machine: `34-STATE-MACHINES`

### Section 5 — Relationships
```yaml
relationships:
  depends_on: list[RelationRef]
  implements: list[RelationRef]
  extends: list[RelationRef]
  related_to: list[RelationRef]
  # ... all 24 relationship types
```

Full catalog: `07-RELATIONSHIP-TYPES`

### Section 6 — Evidence
```yaml
evidence:
  items: list[EvidenceRecord]
  overall_confidence: float          # 0.0..1.0
  evidence_score: float              # 0.0..1.0
  last_verified_at: datetime | null
```

Full schema: `10-EVIDENCE-ENGINE`

### Section 7 — Quality
```yaml
quality:
  completeness: float
  consistency: float
  coverage: float
  freshness: float
  evidence: float
  confidence: float
  usage: float
  reasoning: float
  health: float
  overall_score: float               # weighted composite
  computed_at: datetime
```

Full model: `11-QUALITY-ENGINE`

### Section 8 — Ownership
```yaml
ownership:
  primary_owner: OwnerRef
  secondary_owners: list[OwnerRef]
  reviewers: list[OwnerRef]
  domain_owner: OwnerRef
  approval_required_from: list[OwnerRef]
```

### Section 9 — Security
```yaml
security:
  classification: PUBLIC|INTERNAL|CONFIDENTIAL|SECRET
  access_roles: list[string]
  write_roles: list[string]
  approve_roles: list[string]
  encryption_required: bool
  audit_required: bool
```

### Section 10 — Dependencies
```yaml
dependencies:
  hard: list[DependencyRef]          # must be VERIFIED before this object
  soft: list[DependencyRef]          # should be VERIFIED; warns if not
  runtime: list[DependencyRef]       # required at execution time
  build: list[DependencyRef]         # required at compile time
  test: list[DependencyRef]          # required for testing
```

### Section 11 — Capabilities
```yaml
capabilities:
  provides: list[CapabilityRef]      # capabilities this object exposes
  requires: list[CapabilityRef]      # capabilities this object consumes
  optional: list[CapabilityRef]
```

### Section 12 — Interfaces
```yaml
interfaces:
  exposes: list[InterfaceRef]        # APIs, protocols, contracts
  implements: list[InterfaceRef]     # contracts this object fulfills
```

### Section 13 — Inputs / Outputs
```yaml
io:
  inputs: list[IOSpec]
  outputs: list[IOSpec]
  side_effects: list[string]
  idempotent: bool
```

### Section 14 — History
```yaml
history:
  versions: list[VersionRef]         # all past versions
  snapshots: list[SnapshotRef]       # point-in-time captures
  mutations: list[MutationRecord]    # field-level change log
  last_read_at: datetime | null
  read_count: int
  write_count: int
```

### Section 15 — Extensions
```yaml
extensions:
  domain_fields: dict                # domain-specific typed extensions
  plugin_data: dict                  # plugin-registered extra fields
  experimental: dict                 # fields not yet promoted to schema
```

### Section 16 — Traceability
```yaml
traceability:
  requirement_ids: list[string]
  architecture_ids: list[string]
  decision_ids: list[string]
  test_ids: list[string]
  deployment_ids: list[string]
  coverage_score: float              # 0.0..1.0
```

Full model: `13-TRACEABILITY`

---

## Schema Validation Rules

| Rule | Description |
|------|-------------|
| US-001 | All 16 sections must be present (may be empty, not null) |
| US-002 | `identity` section must be complete before any other section is validated |
| US-003 | `lifecycle.status` must be consistent with populated fields (e.g., CANONICAL requires `approved_by`) |
| US-004 | `quality.overall_score` ≥ `lifecycle.status.min_quality_threshold` |
| US-005 | `evidence.items` must be non-empty for status ≥ VERIFIED |
| US-006 | `security.classification` must match the classification of any referenced secrets |
| US-007 | Circular `dependencies.hard` chains are rejected at write time |

---

## Minimal Valid Object (DRAFT)

```yaml
identity:
  global_id: "a3f2c1d4-0000-0000-0000-000000000001"
  knowledge_id: "KNW-KOS-ARCH-001"
  namespace: "kos"
  canonical_name: "kos.arch.vision"
  version: "1.0.0"
  owner: "team:architecture-board"
  domain: "ARCHITECTURE"
  created_at: "2026-01-01T00:00:00Z"
  updated_at: "2026-01-01T00:00:00Z"
  checksum: "sha256:abc123"
  fingerprint: "fp:kos:arch:0001"
  lineage: null
  aliases: []
  knowledge_uri: "knw://kos/arch/001"

classification:
  object_type: ARCHITECTURE
  maturity: EXPERIMENTAL
  tier: 2
  tags: ["kos", "vision"]

lifecycle:
  status: DRAFT
  review_status: PENDING

# All other sections: present but empty/default
```

---

## Cross-References

- Identity fields → `01-IDENTITY-ENGINE`
- Lifecycle states → `34-STATE-MACHINES`
- Relationship fields → `06-RELATIONSHIP-ENGINE`
- Evidence fields → `10-EVIDENCE-ENGINE`
- Quality computation → `11-QUALITY-ENGINE`
- Traceability fields → `13-TRACEABILITY`
