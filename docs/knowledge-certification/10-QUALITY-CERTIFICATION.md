# KNW-CERT-ARCH-010 — Quality Certification

**Phase:** 3.0D.0  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Certifies that the quality scoring engine correctly computes all 9 quality dimensions for every Knowledge Object, and that quality gates are enforced at each lifecycle transition.

---

## Quality Dimensions Under Test

From Phase 3.0C.5 `16-KNOWLEDGE-QUALITY` and `17-KNOWLEDGE-SCORING`:

| Dim | Name | Weight (general) | Check ID |
|-----|------|------------------|----------|
| QD-1 | Completeness | 0.25 | QC-001 |
| QD-2 | Consistency | 0.15 | QC-002 |
| QD-3 | Evidence | 0.20 | QC-003 |
| QD-4 | Freshness | 0.10 | QC-004 |
| QD-5 | Confidence | 0.10 | QC-005 |
| QD-6 | Traceability | 0.10 | QC-006 |
| QD-7 | Usage | 0.05 | QC-007 |
| QD-8 | Coverage | 0.05 | QC-008 |
| QD-9 | Health penalty | multiplier | QC-009 |

---

## Scoring Engine Checks

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| QD-1 Completeness score ∈ [0.0, 1.0] | QC-001 | CRITICAL | All objects |
| QD-2 Consistency score ∈ [0.0, 1.0] | QC-002 | CRITICAL | |
| QD-3 Evidence score ∈ [0.0, 1.0] | QC-003 | CRITICAL | |
| QD-4 Freshness score ∈ [0.0, 1.0] | QC-004 | CRITICAL | |
| QD-5 Confidence score ∈ [0.0, 1.0] | QC-005 | CRITICAL | |
| QD-6 Traceability score ∈ [0.0, 1.0] | QC-006 | CRITICAL | |
| QD-7 Usage score ∈ [0.0, 1.0] | QC-007 | MAJOR | |
| QD-8 Coverage score ∈ [0.0, 1.0] | QC-008 | MAJOR | |
| QD-9 Health penalty ∈ [0.0, 1.0] | QC-009 | CRITICAL | |
| Overall score = Σ(dim × weight) × health | QC-010 | CRITICAL | Formula verified on 200 objects |
| Overall score ∈ [0.0, 1.0] | QC-011 | CRITICAL | |
| Type-specific weights sum to 1.0 | QC-012 | MAJOR | For each object type |
| Score is deterministic (same input → same output) | QC-013 | CRITICAL | |
| Score recalculated after evidence change | QC-014 | MAJOR | Updated immediately |

---

## Gate Enforcement Checks

| Lifecycle Gate | Check ID | Severity | Pass Criteria |
|----------------|----------|----------|---------------|
| PROPOSED gate: schema valid | QC-015 | CRITICAL | Score not checked |
| VERIFIED gate: overall ≥ 0.60 | QC-016 | CRITICAL | Objects < 0.60 blocked |
| CANONICAL gate: overall ≥ 0.80 | QC-017 | CRITICAL | Objects < 0.80 blocked |
| CANONICAL gate: evidence ≥ 0.55 | QC-018 | MAJOR | Objects with evidence < 0.55 blocked |

---

## Alert Checks (QA-001–QA-006)

| Alert | ID | Severity | Trigger |
|-------|----|----------|---------|
| QA-001 CriticalDrop | QC-019 | MAJOR | overall drops > 0.10 in one update |
| QA-002 EvidenceGap | QC-020 | MAJOR | evidence = 0 for CANONICAL object |
| QA-003 StaleEvidence | QC-021 | MINOR | All evidence older than threshold |
| QA-004 FreshnessDrop | QC-022 | MINOR | freshness < 0.40 |
| QA-005 LowConfidence | QC-023 | MINOR | confidence < 0.50 |
| QA-006 ZeroUsage | QC-024 | INFO | usage = 0 for > 90 days |

---

## Benchmark Scoring

For the 19 benchmark objects (KNW-TEST-BENCH-001 through KNW-TEST-BENCH-019):
- Compute expected score using canonical formula
- Compare implementation output to expected score
- Tolerance: ≤ 0.001 absolute difference

| Check | ID | Severity |
|-------|----|----------|
| Benchmark score matches expected (all 19) | QC-025 | MAJOR |
| Tolerance ≤ 0.001 | QC-026 | MAJOR |

---

## Aggregate Quality Metrics

Run on 200 golden dataset objects:

| Metric | Target (Gold) |
|--------|---------------|
| Mean overall score | ≥ 0.75 |
| Objects with score ≥ 0.80 | ≥ 70% |
| Zero CANONICAL objects with score < 0.80 | required |
| Zero CANONICAL objects with evidence = 0 | required |

---

## Report Format

```json
{
  "domain": "quality",
  "objects_tested": 200,
  "scoring_checks": {
    "QC-010": "PASS", "QC-013": "PASS", "QC-025": "PASS"
  },
  "gate_checks": {
    "QC-016": "PASS", "QC-017": "FAIL"
  },
  "aggregate": {
    "mean_score": 0.781,
    "pct_above_0_80": 0.72,
    "canonical_below_0_80": 2
  },
  "domain_score": 0.853,
  "level_achieved": "Silver"
}
```

---

## CLI

```bash
kos-cert quality                         # all objects in workspace
kos-cert quality --package kos.platform.package
kos-cert quality --benchmark-only        # 19 benchmark objects only
kos-cert quality --output reports/quality.json
```

---

## Cross-References

- Quality dimensions → Phase 3.0C.5 `16-KNOWLEDGE-QUALITY`
- Scoring algorithm A-04 → Phase 3.0C.5 `17-KNOWLEDGE-SCORING`
- Evidence weights → `11-EVIDENCE-CERTIFICATION`
- Coverage → `26-COVERAGE-CERTIFICATION`
- Overall score weight (10%) → `27-OVERALL-SCORING`
