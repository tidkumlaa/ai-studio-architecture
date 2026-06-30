# KNW-CERT-ARCH-025 — Regression Certification

**Phase:** 3.0D.0  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Certifies that no previously passing check regresses across implementation updates. Regression certification runs a fixed, pinned subset of the full certification suite against version-locked expected results.

---

## Regression Suite Composition

The regression suite consists of 3 layers:

| Layer | Description | Size |
|-------|-------------|------|
| Smoke | Fastest checks — gate for any PR | 15 checks |
| Core | Critical path checks — gate for merge | 75 checks |
| Full | Complete regression — nightly | All certification checks |

---

## Smoke Layer (15 checks — runs in < 30 seconds)

| Check | Domain | What it verifies |
|-------|--------|-----------------|
| RS-001 | Registry | Lookup by ID returns correct object |
| RS-002 | Registry | Duplicate ID rejected |
| RS-003 | Search | Top1 on 10 fixed queries matches expected |
| RS-004 | Graph | BFS from node-1 matches expected visited set |
| RS-005 | Graph | Cycle detected in cycle fixture |
| RS-006 | Lifecycle | DRAFT → PROPOSED transition succeeds |
| RS-007 | Lifecycle | CANONICAL → DRAFT rejected |
| RS-008 | Metadata | knowledge_id format valid on 20 objects |
| RS-009 | Quality | Quality score on fixture object = 0.875 ± 0.001 |
| RS-010 | Security | Duplicate ID injection rejected |
| RS-011 | Security | SQL injection in ID rejected |
| RS-012 | Dependency | 9-package graph resolves without error |
| RS-013 | Dependency | Cycle in synthetic graph detected |
| RS-014 | Relationship | Bidirectionality on 10 fixture relationships |
| RS-015 | Traceability | Chain traversal on 5 fixture requirements |

---

## Core Layer (75 checks — runs in < 5 minutes)

Includes all CRITICAL-severity checks from domains:
- 02-SEARCH (SC-001 through SC-012)
- 03-REGISTRY (RC-001 through RC-022)
- 04-GRAPH (GC-001 through GC-019)
- 07-LIFECYCLE (LC-001 through LC-021)
- 18-SECURITY (NS-001 through NS-022)
- 09-DEPENDENCY (DC-001 through DC-018)

---

## Full Layer (all checks — nightly, ≤ 2 hours)

All checks from all 28 certification documents.

---

## Regression Data Format

Expected results are stored in the regression dataset:

```yaml
# benchmarks/kos/datasets/regression/expected/RS-009.yaml
check_id: RS-009
description: "Quality score on fixture object KNW-TEST-BENCH-001"
expected:
  quality_score: 0.875
  tolerance: 0.001
fixture_id: KNW-TEST-BENCH-001
```

---

## Regression Check Protocol

For each check in the regression suite:

```
1. Load fixture data (pinned, version-locked)
2. Run check against implementation
3. Compare result to expected result
4. PASS if within declared tolerance
5. FAIL if outside tolerance or error raised
6. Record delta (current - expected) in regression report
```

---

## Regression Rules

| Rule | Description |
|------|-------------|
| RR-001 | Regression suite uses pinned dataset (never auto-updated) |
| RR-002 | Expected results files are commit-signed |
| RR-003 | Any regression in Smoke or Core layer is a blocking failure |
| RR-004 | Full layer regressions in MINOR checks are warnings, not blockers |
| RR-005 | Expected results updated only via explicit PR with architecture board approval |
| RR-006 | Regression reports are archived for 1 year |

---

## Regression Metrics

| Metric | Pass Criteria |
|--------|---------------|
| Smoke layer pass rate | = 1.0 (no exceptions) |
| Core layer pass rate | = 1.0 (no exceptions) |
| Full layer pass rate | ≥ 0.97 (MINOR regressions tolerated) |
| No CRITICAL regression | Absolute requirement |
| Regression score (delta from baseline) | ≥ 0.95 |

---

## Report Format

```json
{
  "domain": "regression",
  "layers": {
    "smoke": {
      "checks": 15,
      "pass": 15,
      "fail": 0,
      "duration_s": 18.4
    },
    "core": {
      "checks": 75,
      "pass": 74,
      "fail": 1,
      "failed_checks": ["RC-022"],
      "duration_s": 284.1
    },
    "full": {
      "checks": 312,
      "pass": 307,
      "fail": 5,
      "warnings": 3,
      "duration_s": 4201.0
    }
  },
  "blocking_regressions": 1,
  "regression_score": 0.984,
  "domain_score": 0.961,
  "level_achieved": "Gold"
}
```

---

## CLI

```bash
kos-cert regression                      # Core layer (default)
kos-cert regression --layer smoke        # Smoke only (30s)
kos-cert regression --layer full         # Full nightly
kos-cert regression --check RS-009       # single check
kos-cert regression --update-expected    # update expected results (requires board approval)
kos-cert regression --output reports/regression.json
```

---

## Cross-References

- Evolution (trend comparison) → `24-EVOLUTION-CERTIFICATION`
- CI integration (runs smoke on every PR) → `29-CI-INTEGRATION`
- Dataset pinning → `21-DATASET-CERTIFICATION`
