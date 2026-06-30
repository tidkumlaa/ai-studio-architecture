# KNW-KE-ARCH-019 — Knowledge Lifecycle

**Phase:** 3.0C.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Lifecycle rules for all objects in the Knowledge Ecosystem. Extends Phase 3.0C `34-STATE-MACHINES` with ecosystem-specific gates and commands.

---

## Base Lifecycle (from Phase 3.0C SM-1)

```
DRAFT → REVIEW → VERIFIED → APPROVED → CANONICAL → DEPRECATED → ARCHIVED
```

All 7 states are available to ecosystem objects.

---

## Ecosystem Lifecycle Rules

| Rule | Description |
|------|-------------|
| EL-001 | New objects start in DRAFT |
| EL-002 | Only the owner or Architecture Board may advance lifecycle |
| EL-003 | Lifecycle may not skip states (DRAFT → CANONICAL is invalid) |
| EL-004 | CANONICAL objects are version-locked (PATCH only) |
| EL-005 | Regression dataset objects may remain DRAFT permanently |
| EL-006 | Template objects do not have a lifecycle (they are not KnowledgeObjects) |
| EL-007 | A DEPRECATED object must have `lifecycle.successor_id` set |
| EL-008 | ARCHIVED objects are read-only — no field may be changed |

---

## Lifecycle Transitions

### DRAFT → REVIEW

```
Gates:
  - knowledge_id format valid
  - name non-empty
  - description ≥ 20 chars
  - owner set
  - tags non-empty
  - version valid semver

Command: kos lifecycle review KNW-PLT-MOD-001
```

### REVIEW → VERIFIED

```
Gates:
  - quality.overall_score ≥ 0.65
  - evidence.evidence_score ≥ 0.55
  - ≥ 1 evidence record
  - type-specific required fields all non-empty
  - all linter errors resolved

Command: kos lifecycle verify KNW-PLT-MOD-001
```

### VERIFIED → APPROVED

```
Gates:
  - quality.overall_score ≥ 0.75
  - approved_by is non-empty
  - ≥ 1 evidence record of type EV-HAPPROVAL

Command: kos lifecycle approve KNW-PLT-MOD-001 --approved-by team:architecture
```

### APPROVED → CANONICAL

```
Gates:
  - quality.overall_score ≥ 0.80
  - confidence.declared ≥ 0.80
  - traceability coverage ≥ 0.70
  - ≥ 1 EV-CODE or EV-TEST evidence
  - Architecture Board sign-off (for shared/cross-package objects)

Command: kos lifecycle canonicalize KNW-PLT-MOD-001 --board-approval ADR-001
```

### CANONICAL → DEPRECATED

```
Gates:
  - successor_id is set (or explicit waiver from Architecture Board)

Command: kos lifecycle deprecate KNW-PLT-MOD-001 --successor KNW-PLT-MOD-002
Effect:  - Set deprecated_at
         - Emit knowledge.object.deprecated event
         - All ACTIVE relationships auto-deprecated
```

### DEPRECATED → ARCHIVED

```
Gates:
  - deprecated_at + 90 days elapsed
  OR
  - explicit force by Architecture Board

Command: kos lifecycle archive KNW-PLT-MOD-001
Effect:  - Object becomes read-only
         - URI resolves to tombstone: {knowledge_uri} → 410 Gone
         - Emit knowledge.object.archived
```

---

## Package Lifecycle

Packages also have lifecycle states:

| State | Description |
|-------|-------------|
| ACTIVE | Objects can be added/updated |
| FROZEN | Objects cannot be changed without ADR |
| DEPRECATED | No new objects; existing objects in migration |
| ARCHIVED | All objects archived; package read-only |

Package lifecycle advances when ≥ 90% of its objects are in the matching state.

---

## Lifecycle Events

| Event | Emitted When |
|-------|-------------|
| `knowledge.object.status_changed` | Any lifecycle transition |
| `knowledge.object.deprecated` | CANONICAL → DEPRECATED |
| `knowledge.object.archived` | DEPRECATED → ARCHIVED |
| `knowledge.object.canonical` | APPROVED → CANONICAL |
| `knowledge.lifecycle.gate_failed` | A gate condition not met |

---

## Lifecycle CLI

```bash
kos lifecycle status KNW-PLT-MOD-001         # show current state
kos lifecycle history KNW-PLT-MOD-001        # show transition history
kos lifecycle check-gates KNW-PLT-MOD-001 VERIFIED  # check if can transition
kos lifecycle review KNW-PLT-MOD-001         # DRAFT → REVIEW
kos lifecycle verify KNW-PLT-MOD-001         # REVIEW → VERIFIED
kos lifecycle approve KNW-PLT-MOD-001 --approved-by team:arch
kos lifecycle canonicalize KNW-PLT-MOD-001
kos lifecycle deprecate KNW-PLT-MOD-001 --successor KNW-PLT-MOD-002
kos lifecycle archive KNW-PLT-MOD-001
```

---

## Cross-References

- State machine definition → Phase 3.0C `34-STATE-MACHINES` SM-1
- Lifecycle extensions → Phase 3.0C `33-LIFECYCLE-EXTENSIONS`
- Quality gates per state → `16-KNOWLEDGE-QUALITY`
- Linter checks lifecycle rules → `06-KNOWLEDGE-LINTER` KL-021–KL-026
