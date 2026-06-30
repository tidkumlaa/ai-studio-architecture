---
knowledge_id: KA-SPEC-003
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
---

# Knowledge Identity Specification

## Permanent, Unique, Human-Readable Identity for Every Knowledge Object

---

## 1. Purpose

Identity is the foundation of every other property of the knowledge system. Without stable identity:
- Relationships cannot be declared — they reference names, not objects
- Indexes cannot point — they point to locations, not concepts
- Health cannot be tracked — baselines change with document moves
- History is lost — version control operates on paths, not objects

This specification defines the identity system: the format, generation rules, stability guarantees, namespace allocation, and registry protocol for Knowledge IDs.

---

## 2. Knowledge ID Format

### 2.1 Canonical Format

```
KA-[TYPE]-[NNN]

Where:
  KA    = Knowledge Architecture namespace prefix (constant)
  TYPE  = 2–5 uppercase letters identifying the object type
  NNN   = 3-digit zero-padded decimal sequence number

Examples:
  KA-ARCH-001    First architecture document
  KA-ADR-042     42nd Architecture Decision Record
  KA-SPEC-001    First specification (this document)
  KA-API-007     7th API document
  KA-STD-002     Second standard
```

### 2.2 Type Codes

| Type Code | Object Type | Notes |
|-----------|------------|-------|
| `VIS` | Vision | Max 1 per repository |
| `OVW` | Overview | 1 per capability |
| `ARCH` | Architecture | 1 canonical per capability |
| `API` | API document | 1 per capability |
| `DB` | Database document | 1 per capability |
| `EVT` | Event document | 1 per capability |
| `SEC` | Security document | 1 per capability |
| `DSK` | Desktop document | 1 per capability |
| `IMPL` | Implementation | 1 per capability |
| `TEST` | Testing document | 1 per capability |
| `ROAD` | Roadmap | 1 per capability |
| `HIST` | History | 1 per capability |
| `REF` | References | 1 per capability |
| `STD` | Standard | Repository-scoped |
| `SPEC` | Specification | Repository-scoped |
| `ADR` | Architecture Decision Record | Repository-scoped |
| `IDX` | Index | Folder-scoped |
| `CAP` | Capability (structural) | Registry entry only |
| `DOM` | Domain (structural) | Registry entry only |
| `PROD` | Product (structural) | Registry entry only |

### 2.3 Sequence Number Rules

- Sequence numbers start at `001` and increment by `1`
- Numbers are zero-padded to 3 digits: `001`, `002`, ..., `099`, `100`
- Sequence numbers do not restart per capability — they are per TYPE CODE across the repository
- Sequence numbers above `999` are permitted: `KA-ARCH-1000` (no zero-padding beyond 3 digits)
- Retired IDs are never reused

---

## 3. Identity Properties

### 3.1 Permanence

A Knowledge ID, once assigned, is permanent for the lifetime of the repository.

**Moving** a document does not change its ID. The ID is not derived from path.
**Renaming** a document does not change its ID. The ID is not derived from filename.
**Splitting** a document produces new IDs for the new documents. The original ID either continues (if the document still exists in modified form) or is superseded.

### 3.2 Uniqueness

No two KnowledgeObjects in the repository share the same ID.

Uniqueness is enforced by:
1. The `KNOWLEDGE-REGISTRY.yaml` as the authoritative allocation ledger
2. The audit rule `RULE-M002` which detects collisions
3. The index generator which fails on collision

### 3.3 Non-Reuse

When a KnowledgeObject is archived, its ID is marked `retired` in the registry but the ID is never reallocated. This ensures that historical references to a retired ID are always unambiguous: they refer to the archived document, not an unrelated successor.

### 3.4 Stability Guarantee

The stability guarantee is: **any system that stores a Knowledge ID can use that ID to retrieve the KnowledgeObject (or its successor) indefinitely.**

If a document is deprecated, the ID resolves to the deprecated document, which declares `superseded_by` pointing to the current canonical.

If a document is archived, the ID resolves to the archive entry.

If a document is deleted (should never happen — violation of the no-delete rule), the ID resolves to `null` with an error annotation in the registry.

---

## 4. ID Generation Protocol

### 4.1 New Document ID Assignment

```
1. Author determines type code from the document type
2. Author looks up next available ID in KNOWLEDGE-REGISTRY.yaml:
     next_available.[TYPE_CODE]
3. Author assigns the next available ID to the document
4. Author adds the ID to KNOWLEDGE-REGISTRY.yaml:
     registry.[ID].status = "active"
     registry.[ID].file = "[relative path]"
     registry.[ID].type = "[type]"
     registry.[ID].created = "[date]"
5. Author updates next_available.[TYPE_CODE] to ID+1
6. Audit tool validates: no collision exists
```

### 4.2 Automated ID Generation

In Phase 2.0D.2+, the index engine generates IDs automatically:

```
FUNCTION generate_id(type_code):
  registry = load KNOWLEDGE-REGISTRY.yaml
  next = registry.next_available[type_code]
  IF next is None:
    next = type_code + "-001"
  WHILE next in registry.registry:
    next = increment(next)
  registry.next_available[type_code] = increment(next)
  RETURN next
```

### 4.3 ID Reservation

An author may reserve an ID before creating a document (e.g., when planning a split):

```yaml
# In KNOWLEDGE-REGISTRY.yaml
registry:
  KA-ARCH-015:
    status: reserved
    reserved_by: "Capability Architect"
    reserved_date: 2026-06-29
    intended_for: "capabilities/workflow-runtime/architecture.md (post-split)"
```

Reserved IDs are not active — they are excluded from indexes. Reservations expire after 30 days if not activated.

---

## 5. KNOWLEDGE-REGISTRY.yaml Specification

### 5.1 Full Schema

```yaml
meta:
  total_active: 247
  total_reserved: 3
  total_retired: 560
  last_updated: 2026-06-29
  schema_version: "1.0.0"

next_available:
  ARCH: "KA-ARCH-013"
  API:  "KA-API-009"
  ADR:  "KA-ADR-048"
  SPEC: "KA-SPEC-021"
  STD:  "KA-STD-009"
  OVW:  "KA-OVW-013"
  DB:   "KA-DB-005"
  EVT:  "KA-EVT-004"
  SEC:  "KA-SEC-004"
  DSK:  "KA-DSK-003"
  IMPL: "KA-IMPL-004"
  TEST: "KA-TEST-004"
  ROAD: "KA-ROAD-003"
  HIST: "KA-HIST-004"
  REF:  "KA-REF-004"
  VIS:  "KA-VIS-002"
  IDX:  "KA-IDX-002"
  CAP:  "KA-CAP-011"
  DOM:  "KA-DOM-010"
  PROD: "KA-PROD-003"

registry:
  KA-SPEC-001:
    status: active            # active | reserved | retired
    type: specification
    title: "Knowledge Object Specification"
    file: docs/knowledge-core/01-KNOWLEDGE-OBJECT-SPECIFICATION.md
    domain: DOM-KNOWLEDGE
    capability: knowledge-core
    created: 2026-06-29
    version: "1.0.0"

  KA-ARCH-000:
    status: retired           # Archived document
    type: architecture
    title: "Workflow Runtime Architecture (v1)"
    file: archive/v2.0/workflow-runtime-arch-v1.md
    superseded_by: KA-ARCH-001
    retired_date: 2026-01-15
```

### 5.2 Registry Integrity Rules

| Rule | Check |
|------|-------|
| No collision | `registry` keys are unique |
| No active gaps | Sequence numbers are not skipped without a reason |
| No retired reuse | `status: retired` IDs never appear in `next_available` |
| File exists | Every `active` entry's `file` resolves on disk |
| Supersession chain valid | `superseded_by` references an existing entry |

---

## 6. External References

When a KnowledgeObject references an entity outside the repository (e.g., a platform source file, an external API), it uses a reference object rather than a Knowledge ID:

```yaml
implemented_by:
  - type: module
    ref: platform/workflow-runtime/src/executor.py   # Path, not ID
    confidence: high
```

External references are validated for existence but not tracked in the registry. Their identity is the filesystem path or URL.

---

## 7. Cross-Repository Identity

When multiple repositories participate in the knowledge system (e.g., `architecture` and `platform`), each repository maintains its own registry. Cross-repository references use qualified IDs:

```
[repository-name]::KA-ARCH-001

Examples:
  architecture::KA-ARCH-001      # Within the architecture repo
  platform::KA-ARCH-001          # Hypothetical platform repo ADR
```

Within a single-repository context, the qualifier is omitted.

---

## References

- [01-KNOWLEDGE-OBJECT-SPECIFICATION.md](01-KNOWLEDGE-OBJECT-SPECIFICATION.md) — Identity is one property of a KnowledgeObject
- [04-KNOWLEDGE-URI-SPECIFICATION.md](04-KNOWLEDGE-URI-SPECIFICATION.md) — URI representation of identity
- [08-KNOWLEDGE-GRAPH-MODEL.md](08-KNOWLEDGE-GRAPH-MODEL.md) — IDs as graph node keys
- [KA-STD-002](../knowledge-architecture/METADATA-STANDARD.md) — knowledge_id frontmatter field
