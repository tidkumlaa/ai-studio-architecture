---
knowledge_id: KA-SPEC-014
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
  - id: KA-SPEC-005
    reason: "Lifecycle states are defined in the schema"
---

# Knowledge Lifecycle

## Formal State Machine for Knowledge Object Lifecycle

---

## 1. Purpose

Every KnowledgeObject exists in a lifecycle state. The lifecycle determines how the object participates in indexes, how it affects health scores, whether it is navigable, and when it can be modified.

This specification defines the complete lifecycle as a formal finite state machine (FSM), with all states, transitions, guards, and actions declared explicitly.

---

## 2. Lifecycle States

| State | Symbol | Description | Indexed | Navigable | Modifiable |
|-------|--------|-------------|---------|-----------|-----------|
| `created` | ○ | Stub created; no content | No | No | Yes |
| `draft` | ◑ | Being authored; not ready for review | No | No | Yes |
| `review` | ◕ | Submitted for review; awaiting approval | No | No | Owner only |
| `approved` | ● | Active, authoritative, canonical | Yes | Yes | Yes (version bump) |
| `deprecated` | ◉ | Known to be replaced or retired | Yes (warning) | Yes (warning) | No |
| `archived` | ✕ | Permanently removed from active use | Archive only | No | No |

---

## 3. Formal State Machine

### 3.1 States (Q)

```
Q = { created, draft, review, approved, deprecated, archived }
```

### 3.2 Initial State

```
q₀ = created
```

### 3.3 Accepting States (terminal — no further transitions possible)

```
F = { archived }
```

### 3.4 Transitions (δ)

```
δ: Q × Event → Q

δ(created, begin_authoring)        = draft
δ(draft, submit_for_review)        = review
δ(review, approve)                 = approved
δ(review, reject)                  = draft
δ(approved, begin_revision)        = draft       -- Revision creates new version
δ(approved, deprecate)             = deprecated
δ(deprecated, archive)             = archived
δ(deprecated, restore)             = approved    -- Exceptional; requires ADR
```

### 3.5 Transition Guards

Guards are preconditions that must be true for a transition to fire.

| Transition | Guard |
|-----------|-------|
| `draft → review` | Required metadata fields are complete |
| `review → approved` | Owner has reviewed; no Critical schema violations |
| `review → approved` | `canonical` is explicitly set |
| `approved → deprecated` | `deprecated_date` is set in frontmatter |
| `deprecated → archived` | 90 days have elapsed since `deprecated_date` |
| `deprecated → archived` | All inbound references have been updated |
| `deprecated → restored` | An ADR authorizes the restoration |

### 3.6 Transition Actions

Actions are side effects that execute when a transition fires.

| Transition | Actions |
|-----------|--------|
| `draft → review` | Create review entry in history.md |
| `review → approved` | Add to KNOWLEDGE-REGISTRY.yaml; add to all indexes; set `review_date` |
| `review → draft` | Record rejection reason in history.md |
| `approved → deprecated` | Set deprecation banner; update indexes with warning marker; start 90-day clock |
| `deprecated → archived` | Move to archive/; remove from active indexes; add to archive/index.yaml |
| `deprecated → restored` | Remove deprecation banner; update superseded_by on prior successor; log in history.md |

---

## 4. Lifecycle Invariants

| Invariant | Rule |
|-----------|------|
| **L1 — Monotonicity (mostly)** | Status advances forward except for draft↔review and the exceptional deprecate→restore |
| **L2 — No Archive Modification** | Archived documents are read-only. No field changes. |
| **L3 — Deprecation Requires Date** | Cannot transition to `deprecated` without `deprecated_date` |
| **L4 — Deprecated Docs Not Canonical** | Cannot have `canonical: true` in `deprecated` state |
| **L5 — Archive Delay** | `deprecated → archived` only after 90 days and owner confirmation |
| **L6 — ADR Documents Are Immutable** | ADRs skip the `deprecated` state; they are `superseded` by a new ADR (status: `superseded` is added for ADRs only) |
| **L7 — History Is Append-Only** | History documents are always `approved`; they never change status |

---

## 5. Type-Specific Lifecycle Rules

### 5.1 ADR Lifecycle (Modified FSM)

ADRs have an additional status:

```
Q_adr = { proposed, accepted, superseded }

δ(proposed, accept)         = accepted
δ(proposed, reject)         = [ADR is deleted, not transitioned]
δ(accepted, supersede)      = superseded
```

ADRs are **never deprecated** — they are superseded. The distinction matters: deprecated = no longer relevant; superseded = replaced by a newer decision.

### 5.2 History Document Lifecycle

History documents are always `approved` and never transition. They are append-only:

```
Q_history = { approved }    -- Fixed state
```

The only modification allowed is appending new entries to the body. Version bumps use MINOR increments only.

### 5.3 Index File Lifecycle

`index.yaml` files are always `approved` (they are structural, not knowledge objects). They are regenerated, not authored.

---

## 6. Review Lifecycle

The review cycle is distinct from the document lifecycle. A document in `approved` state is still subject to regular review:

```
FUNCTION review_cycle(obj):
  IF today() > obj.review_date:
    -- Document is overdue for review
    health_score.freshness -= calculate_staleness_penalty(obj)

  WHEN owner reviews:
    IF content is accurate:
      SET obj.review_date = today() + review_interval(obj.type)
      SET obj.version = increment_patch(obj.version)
    IF content is stale:
      INITIATE deprecation workflow
```

Review intervals are defined in [KA-STD-002](../knowledge-architecture/METADATA-STANDARD.md).

---

## 7. Lifecycle Events

The KIE emits lifecycle events that the Documentation Intelligence Platform can subscribe to:

| Event | Trigger | Payload |
|-------|---------|---------|
| `knowledge.created` | New document detected | `{ id, type, capability }` |
| `knowledge.approved` | Status transitions to approved | `{ id, version, owner }` |
| `knowledge.updated` | Version incremented | `{ id, old_version, new_version }` |
| `knowledge.deprecated` | Status transitions to deprecated | `{ id, superseded_by, deprecated_date }` |
| `knowledge.stale` | Review date passed | `{ id, owner, days_overdue }` |
| `knowledge.archived` | Status transitions to archived | `{ id, archive_path }` |
| `knowledge.health_degraded` | Health score drops below threshold | `{ id, old_score, new_score }` |

---

## 8. Lifecycle Visualization

```
           ┌──────────────────────────────────────────────────┐
           │                  Lifecycle FSM                   │
           └──────────────────────────────────────────────────┘

  [created] ──begin_authoring──▶ [draft] ──submit──▶ [review]
                                   ▲                    │
                                   └────── reject ──────┘
                                                        │
                                                     approve
                                                        │
                                                        ▼
             [archived] ◀─archive─── [deprecated] ◀─deprecate─ [approved]
                                                                  │  ▲
                                                                  │  │
                                                                  └──┘
                                                              begin_revision
                                                              (new version)
```

---

## References

- [01-KNOWLEDGE-OBJECT-SPECIFICATION.md](01-KNOWLEDGE-OBJECT-SPECIFICATION.md) — KnowledgeObject that participates in lifecycle
- [05-KNOWLEDGE-SCHEMA.md](05-KNOWLEDGE-SCHEMA.md) — Status enum values
- [KA-STD-007](../knowledge-architecture/KNOWLEDGE-EVOLUTION-STRATEGY.md) — Deprecation and archival process
- [KA-STD-006](../knowledge-architecture/KNOWLEDGE-HEALTH-METRICS.md) — How lifecycle state affects health scoring
