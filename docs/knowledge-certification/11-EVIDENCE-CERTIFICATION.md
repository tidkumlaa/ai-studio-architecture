# KNW-CERT-ARCH-011 — Evidence Certification

**Phase:** 3.0D.0  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Certifies that evidence records on Knowledge Objects are complete, correctly weighted, fresh, and that the evidence engine correctly computes the evidence dimension (QD-3) of the quality score.

---

## Evidence Types Under Test

From Phase 3.0C.5 `23-KNOWLEDGE-CANONICAL-SOURCES`:

| Type | Code | Weight | Freshness threshold |
|------|------|--------|---------------------|
| Board Decision | EV-HAPPROVAL | 0.80 | 1 year |
| Architecture Doc | EV-DOC | 0.60 | 6 months |
| Benchmark Result | EV-BENCHMARK | 0.70 | 90 days |
| Test Result | EV-TEST | 0.75 | 30 days |
| Production Code | EV-CODE | 0.60 | 180 days |
| Monitoring Data | EV-MONITORING | 0.70 | 7 days |
| Engineering Proposal | EV-PROPOSAL | 0.40 | 30 days |
| User Feedback | EV-USER | 0.50 | 30 days |
| External Standard | EV-STANDARD | 0.80 | 1 year |
| Research Paper | EV-RESEARCH | 0.65 | 2 years |

---

## Structural Checks

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Evidence type is valid code | EC-001 | CRITICAL | All in canonical type set |
| `weight` ∈ [0.0, 1.0] | EC-002 | MAJOR | |
| `level` ∈ [1, 6] | EC-003 | MAJOR | |
| `captured_at` is ISO 8601 | EC-004 | MAJOR | |
| `freshness_score` ∈ [0.0, 1.0] | EC-005 | MAJOR | |
| `freshness_score` correctly computed | EC-006 | MAJOR | Within 0.01 of expected |
| `disputed: true` items have `dispute_reason` | EC-007 | MINOR | |
| `source_id` references existing object | EC-008 | MAJOR | Zero dangling source IDs |

---

## Completeness Checks

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| CANONICAL objects have ≥ 1 evidence item | EC-009 | CRITICAL | |
| CANONICAL objects have ≥ 1 Level 1–4 source | EC-010 | MAJOR | Rule CS-002 |
| Evidence items not all Level 5–6 | EC-011 | MAJOR | Rule CS-003 |
| Evidence score QD-3 computed correctly | EC-012 | CRITICAL | Matches formula output ±0.001 |

---

## Freshness Checks

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Stale evidence items flagged | EC-013 | MINOR | STALE marker present when past threshold |
| Freshness decay function correct | EC-014 | MAJOR | 1.0 at 0 days; 0.0 at threshold |
| EV-MONITORING freshness < 7 days | EC-015 | MINOR | Alert QA-003 emitted |

---

## Evidence Formula Verification

For 200 golden dataset objects, recompute QD-3 using:

```
QD3 = Σ(item.weight × item.freshness_score × level_bonus[item.level]) / |items|

level_bonus = {1: 1.0, 2: 0.85, 3: 0.75, 4: 0.65, 5: 0.50, 6: 0.30}
```

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| QD-3 matches expected (all 200 objects) | EC-016 | CRITICAL | Absolute error ≤ 0.001 |

---

## Conflict Resolution Checks

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Higher-level source wins conflict | EC-017 | MAJOR | |
| Same-level, newer source wins | EC-018 | MINOR | |
| Disputed items reduce score | EC-019 | MINOR | Disputed item weight multiplied by 0.5 |

---

## Metrics

| Metric | Bronze | Silver | Gold |
|--------|--------|--------|------|
| Evidence formula accuracy | ≥ 0.98 | ≥ 0.99 | = 1.0 (±0.001) |
| Completeness (CANONICAL objects) | ≥ 0.90 | ≥ 0.97 | = 1.0 |
| Structural validity | ≥ 0.95 | ≥ 0.99 | = 1.0 |

---

## Report Format

```json
{
  "domain": "evidence",
  "objects_tested": 200,
  "checks": {
    "EC-009": "PASS", "EC-010": "PASS",
    "EC-012": "PASS", "EC-016": "PASS"
  },
  "evidence_formula_accuracy": 1.0,
  "canonical_completeness": 0.985,
  "stale_items": 12,
  "domain_score": 0.921,
  "level_achieved": "Gold"
}
```

---

## CLI

```bash
kos-cert evidence                        # all objects in workspace
kos-cert evidence --check-freshness      # freshness checks only
kos-cert evidence --output reports/evidence.json
```

---

## Cross-References

- Source hierarchy → Phase 3.0C.5 `23-KNOWLEDGE-CANONICAL-SOURCES`
- Evidence engine → Phase 3.0C `10-EVIDENCE-ENGINE`
- QD-3 weight → Phase 3.0C.5 `17-KNOWLEDGE-SCORING`
- Freshness decay algorithm A-05 → Phase 3.0C `36-ALGORITHMS`
