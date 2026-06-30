# KNW-CERT-ARCH-007 — Lifecycle Certification

**Phase:** 3.0D.0  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Certifies that lifecycle state transitions are enforced correctly — valid transitions proceed, invalid transitions are rejected, and gates are honoured.

---

## Lifecycle States

```
DRAFT → PROPOSED → VERIFIED → CANONICAL → DEPRECATED → ARCHIVED
```

---

## Valid Transitions

| From | To | Gate | Check ID | Severity |
|------|----|------|----------|----------|
| DRAFT | PROPOSED | Schema valid; owner set | LC-001 | CRITICAL |
| PROPOSED | VERIFIED | lint pass; quality ≥ 0.60 | LC-002 | CRITICAL |
| VERIFIED | CANONICAL | quality ≥ 0.80; evidence complete | LC-003 | CRITICAL |
| CANONICAL | DEPRECATED | successor_id set; deprecated_at set | LC-004 | CRITICAL |
| DEPRECATED | ARCHIVED | sunset_at past | LC-005 | MAJOR |
| DRAFT | DRAFT | version bump only | LC-006 | MAJOR |

---

## Invalid Transition Checks (must be rejected)

| Transition | Check ID | Severity | Pass Criteria |
|------------|----------|----------|---------------|
| DRAFT → CANONICAL (skip) | LC-007 | CRITICAL | Returns InvalidTransitionError |
| PROPOSED → DEPRECATED | LC-008 | CRITICAL | Returns InvalidTransitionError |
| CANONICAL → DRAFT (backwards) | LC-009 | CRITICAL | Returns InvalidTransitionError |
| ARCHIVED → any | LC-010 | CRITICAL | Returns InvalidTransitionError |
| DEPRECATED → VERIFIED (backwards) | LC-011 | CRITICAL | Returns InvalidTransitionError |

---

## Gate Enforcement Checks

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| PROPOSED gate fails if schema invalid | LC-012 | CRITICAL | Transition blocked |
| VERIFIED gate fails if quality < 0.60 | LC-013 | MAJOR | Transition blocked |
| CANONICAL gate fails if quality < 0.80 | LC-014 | MAJOR | Transition blocked |
| CANONICAL gate fails if evidence incomplete | LC-015 | MAJOR | Transition blocked |
| DEPRECATED gate fails if no successor | LC-016 | MAJOR | Transition blocked |
| ARCHIVED gate fails if before sunset_at | LC-017 | MINOR | Transition blocked |

---

## Event Log Checks

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Every transition creates lifecycle event | LC-018 | MAJOR | Event log grows by 1 per transition |
| Event records `at`, `by`, `reason` | LC-019 | MAJOR | All three fields non-null |
| Events are append-only | LC-020 | CRITICAL | No event is ever removed or modified |
| Events are ordered chronologically | LC-021 | MAJOR | `at` timestamps non-decreasing |

---

## Package Lifecycle Checks

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Package cannot be CANONICAL if any contained object is DRAFT | LC-022 | MAJOR | |
| Package deprecation notifies dependents | LC-023 | MINOR | Notification event emitted |

---

## Lifecycle Command Checks

| Command | Check ID | Severity | Pass Criteria |
|---------|----------|----------|---------------|
| `kos lifecycle propose KNW-X` | LC-024 | MAJOR | State becomes PROPOSED if gate passes |
| `kos lifecycle verify KNW-X` | LC-025 | MAJOR | State becomes VERIFIED if gate passes |
| `kos lifecycle approve KNW-X` | LC-026 | MAJOR | State becomes CANONICAL if gate passes |
| `kos lifecycle deprecate KNW-X --successor Y` | LC-027 | MAJOR | State becomes DEPRECATED |
| `kos lifecycle archive KNW-X` | LC-028 | MINOR | State becomes ARCHIVED |

---

## Metrics

| Metric | Bronze | Silver | Gold |
|--------|--------|--------|------|
| Valid transitions pass rate | = 1.0 | = 1.0 | = 1.0 |
| Invalid transitions rejection rate | = 1.0 | = 1.0 | = 1.0 |
| Gate enforcement pass rate | ≥ 0.90 | ≥ 0.97 | = 1.0 |
| Event log integrity | = 1.0 | = 1.0 | = 1.0 |

---

## Report Format

```json
{
  "domain": "lifecycle",
  "valid_transitions_tested": 6,
  "invalid_transitions_tested": 5,
  "gate_checks_tested": 6,
  "checks": {
    "LC-001": "PASS", "LC-007": "PASS", "LC-009": "PASS",
    "LC-013": "FAIL", "LC-020": "PASS"
  },
  "critical_pass": true,
  "major_pass": false,
  "domain_score": 0.893,
  "level_achieved": "Silver"
}
```

---

## CLI

```bash
kos-cert lifecycle                       # full lifecycle certification
kos-cert lifecycle --check LC-009        # single check
kos-cert lifecycle --output reports/lifecycle.json
```

---

## Cross-References

- Lifecycle model → Phase 3.0C.5 `19-KNOWLEDGE-LIFECYCLE`
- Lifecycle rules EL-001–EL-008 → Phase 3.0C.5 `19-KNOWLEDGE-LIFECYCLE`
- Quality gates → Phase 3.0C.5 `16-KNOWLEDGE-QUALITY`
