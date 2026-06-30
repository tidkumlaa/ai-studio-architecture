# KVF-DOC-017 — Lifecycle Validation

**Phase:** 3.0D.1.5
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Lifecycle Validation verifies that all KnowledgeObjects conform to the
declared state machine, that lifecycle transitions are valid, that
deprecated objects are properly handled, and that the lifecycle
evolution history is accurate and complete.

---

## KOS State Machine (from Phase 3.0A)

```
DRAFT → EXPERIMENTAL → ACTIVE → CANONICAL → DEPRECATED → ARCHIVED

Valid transitions:
  DRAFT → EXPERIMENTAL
  DRAFT → ACTIVE
  EXPERIMENTAL → ACTIVE
  EXPERIMENTAL → DEPRECATED  (removed before active)
  ACTIVE → CANONICAL
  ACTIVE → DEPRECATED
  CANONICAL → DEPRECATED
  DEPRECATED → ARCHIVED

Invalid transitions (always forbidden):
  Any state → DRAFT  (no regression to draft)
  ARCHIVED → any state  (archived is final)
  CANONICAL → ACTIVE  (cannot downgrade canonical)
  DEPRECATED → CANONICAL  (cannot re-canonicalize)
```

---

## Lifecycle Validation Checks

### LV-1: Current State Validity

```
LV-1.01: metadata.state is one of the 6 valid states
LV-1.02: CANONICAL objects have quality_score >= 0.80
LV-1.03: DEPRECATED objects have deprecation_date set
LV-1.04: ARCHIVED objects have archive_date set
LV-1.05: EXPERIMENTAL objects have an owner contact (metadata.author)
```

### LV-2: Transition History Validity

```
LV-2.01: Each transition in evolution.history is from a valid source state to a valid target state
LV-2.02: Transitions are ordered chronologically (each transition date >= previous)
LV-2.03: The final state in history matches current metadata.state
LV-2.04: No ARCHIVED object has transitions after archive_date
```

### LV-3: Deprecation Handling

```
For each DEPRECATED object D:
  LV-3.01: D has a DEPRECATED_BY relationship to its replacement (if replaced)
  LV-3.02: Objects that DEPEND ON D have a warning in their risk.failure_modes
  LV-3.03: D's context packs include status_warning (from CAE AH Guard rule CAE-070)
  LV-3.04: D's intelligence.evolution.deprecation block is non-empty
```

### LV-4: Lifecycle Executable Rules

```
For each object with intelligence.executable.lifecycle_rules:
  LV-4.01: transition_guards reference only valid state names
  LV-4.02: Each guard condition is syntactically valid (CEL expression)
  LV-4.03: All declared transitions are a subset of the master state machine
```

---

## Lifecycle Coverage

```
corpus_lifecycle_coverage:
  objects_in_DRAFT: integer
  objects_in_EXPERIMENTAL: integer
  objects_in_ACTIVE: integer
  objects_in_CANONICAL: integer
  objects_in_DEPRECATED: integer
  objects_in_ARCHIVED: integer

health_indicators:
  high_draft_rate: DRAFT_count / total > 0.30 (warning: many unfinished objects)
  high_deprecated_rate: DEPRECATED_count / total > 0.20 (warning: corpus aging)
  stale_experimental_rate: objects in EXPERIMENTAL for > 90 days
```

---

## Lifecycle Validation Result Schema

```yaml
lifecycle_validation_result:
  objects_checked: integer

  current_state_validity: float          # LV-1 checks pass rate
  transition_history_validity: float     # LV-2 checks pass rate
  deprecation_handling_rate: float       # LV-3 checks pass rate
  executable_rules_validity: float       # LV-4 checks pass rate

  critical_failures:
    invalid_states: integer
    forward_only_violation: integer      # CANONICAL → ACTIVE, etc.
    archived_with_post_transitions: integer

  corpus_lifecycle_coverage:
    draft: integer
    experimental: integer
    active: integer
    canonical: integer
    deprecated: integer
    archived: integer

  health_indicators:
    high_draft_rate: boolean
    high_deprecated_rate: boolean
    stale_experimental_count: integer
```

---

## Rules

| Rule | Statement |
|------|-----------|
| KVF-081 | CANONICAL → ACTIVE transition found in history is a CRITICAL failure |
| KVF-082 | ARCHIVED → any transition is a CRITICAL failure — archived is final |
| KVF-083 | DEPRECATED objects without deprecation_date fail LV-1.03 — date is mandatory |
| KVF-084 | LV-3.02 check (downstream objects warned about DEPRECATED dep) is advisory at BRONZE level |
| KVF-085 | Transition history must be chronologically ordered — out-of-order entries = data corruption |

---

## Cross-References

- KOS state machine → Phase 3.0A core documentation
- KIL lifecycle spec → Phase 3.0D.0.6 `11-KNOWLEDGE-EVOLUTION`
- KIL executable schema → Phase 3.0D.0.6 `02-EXECUTABLE-KNOWLEDGE`
- Anti-hallucination guard (status warning) → Phase 3.0D.1 `15-ANTI-HALLUCINATION-GUARD`
