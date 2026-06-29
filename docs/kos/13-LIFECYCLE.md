---
knowledge_id: KNW-KOS-ARCH-013
title: "KOS Knowledge Lifecycle"
status: AUTHORITATIVE
phase: "3.0A"
version: "1.0.0"
created: "2026-06-30"
purpose: "Define the Knowledge Object lifecycle state machine with states, transitions, guards, and events"
canonical_source: "architecture/docs/kos/13-LIFECYCLE.md"
dependencies:
  - "04-OBJECT-MODEL.md"
  - "11-GOVERNANCE.md"
  - "12-QUALITY.md"
related_documents:
  - "07-KNOWLEDGE-REGISTRY.md"
  - "03-GOLDEN-RULES.md"
acceptance_criteria:
  - "All 7 states are defined"
  - "All transitions have guard conditions"
  - "Terminal states are identified"
verification_checklist:
  - "[ ] State machine is deterministic"
  - "[ ] No unreachable states"
  - "[ ] Events emitted on all transitions"
future_extensions:
  - "Automatic lifecycle advancement based on quality scores"
  - "Lifecycle expiry for time-sensitive objects"
---

# KOS Knowledge Lifecycle

## States

| State | Meaning | Writeable | Deleteable |
|-------|---------|-----------|------------|
| `DRAFT` | Being authored; incomplete | YES | YES |
| `REVIEW` | Under peer review; frozen for review | NO | NO |
| `VERIFIED` | Review complete; quality checks passed | NO | NO |
| `APPROVED` | Approved by owner team | NO | NO |
| `CANONICAL` | Authoritative, production-grade | NO | NO |
| `DEPRECATED` | Still valid but being phased out | NO | NO |
| `ARCHIVED` | Superseded; historical record only | NO | NO |

---

## State Machine

```
DRAFT ──────────────────────────────────────────────────────▶ REVIEW
   │                                                              │
   │  (author submits                                     (reviewer rejects)
   │   for review)                                               │
   │                                                             ▼
   ◀──────────────────────────────────────────────── REJECTED (returns to DRAFT)
                                                                  │
                                                         (reviewer approves)
                                                                  │
                                                                  ▼
                                                            VERIFIED
                                                                  │
                                                         (owner approves)
                                                                  │
                                                                  ▼
                                                            APPROVED
                                                                  │
                                                    (platform-kos --canonicalize)
                                                         (quality ≥ 0.80)
                                                         (all deps CANONICAL)
                                                                  │
                                                                  ▼
                                                            CANONICAL ◀─────┐
                                                                  │         │
                                                    (newer version         │
                                                     created for          │
                                                     same concept)        │
                                                                  │        │
                                                                  ▼        │
                                                           DEPRECATED      │
                                                                  │        │
                                                    (superseded by         │
                                                     canonical new         │
                                                     object)               │
                                                                  │        │
                                                                  ▼        │
                                                            ARCHIVED ───────┘
                                                                          (new canonical
                                                                           replaces it)
```

---

## Transitions

### DRAFT → REVIEW

**Trigger:** Author calls `platform-kos governance --submit-review KNW-{id}`

**Guard conditions:**
- All required governance fields present and non-empty
- Quality completeness score = 1.0 (QD-4)
- At least 1 relationship edge exists in the graph
- knowledge_id follows naming convention

**Effect:**
- Object becomes read-only for modification
- Event: `platform.kos.lifecycle.review_requested`

---

### REVIEW → VERIFIED (approval)

**Trigger:** Reviewer calls `platform-kos governance --verify KNW-{id}`

**Guard conditions:**
- Reviewer is not the same person as the author
- Quality health score ≥ 0.60 (QD-9)
- Evidence field passes evidence score ≥ 0.50 (QD-7)

**Effect:**
- Object marked VERIFIED
- Event: `platform.kos.lifecycle.verified`

---

### REVIEW → DRAFT (rejection)

**Trigger:** Reviewer calls `platform-kos governance --reject KNW-{id} --reason "..."`

**Guard conditions:** Reviewer provides non-empty reason

**Effect:**
- Object returns to DRAFT (writeable again)
- Rejection reason stored in metadata
- Event: `platform.kos.lifecycle.rejected`

---

### VERIFIED → APPROVED

**Trigger:** Owner team calls `platform-kos governance --approve KNW-{id}`

**Guard conditions:**
- Approver is from owner team
- Quality health score ≥ 0.75 (QD-9)
- No blocking governance violations

**Effect:**
- `approved_by` and `approved_at` fields set
- Event: `platform.kos.lifecycle.approved`

---

### APPROVED → CANONICAL

**Trigger:** `platform-kos governance --canonicalize KNW-{id}`

**Guard conditions:**
- Quality health score ≥ 0.80 (QD-9 overall)
- Quality completeness = 1.0 (QD-4)
- All dependencies are CANONICAL or APPROVED
- At least one test with COVERS edge to this object
- GR-008 traceability chain is complete

**Effect:**
- Object is now the authoritative source
- Used by Knowledge Compiler
- Event: `platform.kos.lifecycle.canonicalized`

---

### CANONICAL → DEPRECATED

**Trigger:** `platform-kos lifecycle --deprecate KNW-{id} --superseded-by KNW-{new-id}`

**Guard conditions:**
- A new canonical or approved object exists to replace it
- All consumers of this object have been notified (via events)
- Deprecation timeline is set

**Effect:**
- Object remains readable and valid during deprecation period
- Compiler generates deprecation warnings for consumers
- Event: `platform.kos.lifecycle.deprecated`

---

### DEPRECATED → ARCHIVED

**Trigger:** `platform-kos lifecycle --archive KNW-{id}` or automatic after deprecation_end_date

**Guard conditions:**
- All consumers have migrated to the replacement object
- No CANONICAL object has DEPENDS_ON edge to this object

**Effect:**
- Object moves to `knowledge/archive/`
- Remains queryable (historical) but not compiled
- Event: `platform.kos.lifecycle.archived`

---

## Lifecycle Events

| Event Topic | Trigger |
|------------|---------|
| `platform.kos.lifecycle.review_requested` | DRAFT → REVIEW |
| `platform.kos.lifecycle.verified` | REVIEW → VERIFIED |
| `platform.kos.lifecycle.rejected` | REVIEW → DRAFT |
| `platform.kos.lifecycle.approved` | VERIFIED → APPROVED |
| `platform.kos.lifecycle.canonicalized` | APPROVED → CANONICAL |
| `platform.kos.lifecycle.deprecated` | CANONICAL → DEPRECATED |
| `platform.kos.lifecycle.archived` | DEPRECATED → ARCHIVED |

All events carry: `knowledge_id`, `object_type`, `old_status`, `new_status`, `actor`, `timestamp`.

---

## KnowledgeLifecycleStatus Enum

```python
class KnowledgeLifecycleStatus(str, Enum):
    DRAFT = "DRAFT"
    REVIEW = "REVIEW"
    VERIFIED = "VERIFIED"
    APPROVED = "APPROVED"
    CANONICAL = "CANONICAL"
    DEPRECATED = "DEPRECATED"
    ARCHIVED = "ARCHIVED"
```

---

## Lifecycle CLI

```bash
# Submit for review
platform-kos lifecycle --submit-review KNW-AI-MOD-quota_manager

# Verify (reviewer action)
platform-kos lifecycle --verify KNW-AI-MOD-quota_manager --reviewer "Knowledge Team"

# Approve (owner action)
platform-kos lifecycle --approve KNW-AI-MOD-quota_manager --approver "AI Runtime Team"

# Canonicalize
platform-kos lifecycle --canonicalize KNW-AI-MOD-quota_manager

# Deprecate with replacement
platform-kos lifecycle --deprecate KNW-AI-MOD-old --superseded-by KNW-AI-MOD-new

# Show lifecycle history
platform-kos lifecycle --history KNW-AI-MOD-quota_manager
```
