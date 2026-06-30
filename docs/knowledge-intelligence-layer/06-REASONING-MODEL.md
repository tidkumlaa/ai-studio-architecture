# KNW-KIL-DOC-006 — Reasoning Model

**Phase:** 3.0D.0.6
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Every Knowledge Object must express its role in the system's reasoning graph —
what it requires, what it provides, what it blocks, what it enables, and how it
impacts other objects.

This is distinct from the relationship graph (Phase 3.0C). The reasoning model
captures *logical dependencies for inference* rather than *structural dependencies
for execution*.

This document defines `intelligence.reasoning`.

---

## Reasoning Relation Types

| Relation | Direction | Meaning |
|----------|-----------|---------|
| REQUIRES | → | This object's reasoning depends on this fact/object being true/present |
| PROVIDES | ← | This object establishes this fact, enabling downstream reasoning |
| DEPENDS | → | A reasoning step about this object must first reason about this |
| CONFLICTS | ↔ | Two objects cannot both be true in the same reasoning context |
| SUPPORTS | → | This object provides evidence that strengthens another object's claim |
| BLOCKS | → | If this object is in a certain state, reasoning about another is blocked |
| ENABLES | → | This object's presence makes a new category of reasoning possible |
| SUGGESTS | → | Weak implication — this object hints at, but doesn't prove, another |
| IMPACTS | → | Changing this object has downstream reasoning consequences |

---

## Schema

```yaml
intelligence:
  reasoning:
    requires:                          # what must be known/true to reason about this object
      - fact: string                   # e.g. "KNW-RT-RT-001 is operational"
        knowledge_id: string | null    # if resolvable to a knowledge object
        condition: string              # when this requirement applies
        blocking: boolean             # if true, reasoning stops without this

    provides:                          # what facts this object establishes
      - fact: string
        knowledge_id: string | null
        confidence: float              # 0.0–1.0 confidence that this fact holds
        conditions: [string]          # conditions under which this fact holds

    depends:                           # objects that must be reasoned about first
      - knowledge_id: string
        reason: string
        order: integer                 # reasoning order (lower = earlier)

    conflicts:                         # reasoning contexts in which this conflicts
      - knowledge_id: string
        conflict_type: MUTUAL_EXCLUSION | CONTRADICTION | SCOPE_OVERLAP
        description: string
        resolution: string

    supports:                          # objects whose claims this object strengthens
      - knowledge_id: string
        support_type: EVIDENCE | DEMONSTRATION | CORROBORATION | EXAMPLE
        strength: STRONG | MODERATE | WEAK
        description: string

    blocks:                            # objects that cannot be reasoned about if this is in a state
      - knowledge_id: string
        when_state: string             # e.g. "DEPRECATED"
        reason: string
        workaround: string

    enables:                           # new reasoning categories this object unlocks
      - capability: string             # e.g. "quota-aware scheduling reasoning"
        description: string
        requires_objects: [string]     # other objects needed alongside

    suggests:                          # weak implications this object hints at
      - knowledge_id: string
        suggestion: string
        strength: float                # 0.0–1.0
        conditions: [string]

    impacts:                           # downstream reasoning consequences of changing this
      - knowledge_id: string
        impact_type: BREAKS | WEAKENS | REQUIRES_UPDATE | INVALIDATES_CACHE
        severity: CRITICAL | HIGH | MEDIUM | LOW
        description: string
```

---

## Canonical Example

```yaml
# For: KNW-PLT-MOD-001 (Quota Manager)
intelligence:
  reasoning:
    requires:
      - fact: "KNW-RT-RT-001 Registry is operational"
        knowledge_id: KNW-RT-RT-001
        condition: "always — quota state is registry-backed"
        blocking: true
      - fact: "KNW-PLT-REQ-001 requirement is understood"
        knowledge_id: KNW-PLT-REQ-001
        condition: "when reasoning about quota enforcement scope"
        blocking: false

    provides:
      - fact: "Tenant resource quotas are enforced at the platform level"
        knowledge_id: null
        confidence: 0.97
        conditions: ["Registry operational", "Config loaded", "Module CANONICAL"]
      - fact: "No single tenant can exceed allocated resource limits"
        knowledge_id: null
        confidence: 0.92
        conditions: ["Quota Manager is on the request path"]

    depends:
      - knowledge_id: KNW-RT-RT-001
        reason: "Must understand registry before reasoning about quota state"
        order: 1
      - knowledge_id: KNW-PLT-REQ-001
        reason: "Must understand requirement before reasoning about enforcement scope"
        order: 2
      - knowledge_id: KNW-PLT-CFG-001
        reason: "Must understand config structure before reasoning about quota limits"
        order: 3

    conflicts:
      - knowledge_id: KNW-PLT-MOD-007
        conflict_type: SCOPE_OVERLAP
        description: "Both claim to manage tenant quota state"
        resolution: "KNW-PLT-MOD-001 is authoritative; KNW-PLT-MOD-007 scope is deprecated"

    supports:
      - knowledge_id: KNW-PLT-REQ-001
        support_type: DEMONSTRATION
        strength: STRONG
        description: "Quota Manager is the live demonstration that quota enforcement is possible"

    blocks:
      - knowledge_id: KNW-PLT-SVC-002
        when_state: "DEPRECATED"
        reason: "Services that consume QuotaDecision cannot reason about quota enforcement"
        workaround: "Use KNW-PLT-MOD-001's replacement if one exists"

    enables:
      - capability: "Multi-tenant resource isolation reasoning"
        description: "Allows architects to reason about fair resource distribution"
        requires_objects: [KNW-RT-RT-001, KNW-PLT-CFG-001]
      - capability: "Cost attribution reasoning"
        description: "Quota data enables per-tenant cost calculation"
        requires_objects: [KNW-FIN-SVC-001]

    suggests:
      - knowledge_id: KNW-FIN-SVC-001
        suggestion: "Quota usage data may be used for billing calculations"
        strength: 0.65
        conditions: ["financial module is present", "usage tracking is enabled"]

    impacts:
      - knowledge_id: KNW-PLT-SVC-002
        impact_type: BREAKS
        severity: CRITICAL
        description: "Changing QuotaDecision interface breaks all callers"
      - knowledge_id: KNW-PLT-MON-001
        impact_type: REQUIRES_UPDATE
        severity: HIGH
        description: "Changing QuotaExceededEvent schema requires monitoring update"
```

---

## Reasoning Graph

The `reasoning` block contributes to the Knowledge Reasoning Graph (KRG):

```
Reasoning graph nodes: all CANONICAL Knowledge Objects
Reasoning graph edges: requires, provides, depends, enables, impacts, blocks

Query: "If I remove Quota Manager, what reasoning breaks?"
  → find all knowledge_ids that require Quota Manager
  → trace impacts transitively
  → report: critical reasoning paths that are broken
```

---

## Rules

| Rule | Statement |
|------|-----------|
| KIL-036 | Every `requires` entry with `blocking: true` MUST be declared in `self_describing.dependencies` |
| KIL-037 | `provides.confidence` must be empirically derived, not assumed to be 1.0 |
| KIL-038 | `depends` ordering must be consistent — no circular `depends` relationships |
| KIL-039 | Every `impacts` entry with `severity: CRITICAL` must be reviewed before any schema change |
| KIL-040 | `conflicts` entries must not reference objects the current object SUPERSEDES |
| KIL-041 | `enables` must list at least one `requires_object` — standalone capabilities are not valid |

---

## Cross-References

- Executable rules → `02-EXECUTABLE-KNOWLEDGE`
- Semantic capabilities → `03-SEMANTIC-KNOWLEDGE`
- Cortex (why) → `19-KNOWLEDGE-CORTEX`
- Tradeoffs → `16-KNOWLEDGE-TRADEOFF-MODEL`
- Risk model → `15-KNOWLEDGE-RISK-MODEL`
