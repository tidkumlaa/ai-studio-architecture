# KNW-CERT-ARCH-001 — Certification Overview

**Phase:** 3.0D.0  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

The Knowledge Platform Certification Suite (KPCS) is the mandatory quality gate for every KOS implementation. No implementation ships to production without a passing certification run.

---

## Certification Model

```
Implementation Under Test (IUT)
        │
        ▼
kos-cert verify [--level gold]
        │
        ├── Functional Suites (02–13)
        ├── Non-Functional Suites (14–20)
        ├── Data/AI Suites (21–23)
        └── Evolution Suites (24–26)
        │
        ▼
  Score Aggregation (27-OVERALL-SCORING)
        │
        ▼
  Certification Report (28-DASHBOARD, 29-CI)
        │
        ▼
  PASS / FAIL + Level Awarded
```

---

## Certification Levels

### Bronze (≥ 60)
Minimum viable implementation. Functional correctness is verified but performance targets are relaxed.

Required:
- All CRITICAL checks pass (no exceptions)
- Search Top1 accuracy ≥ 0.70
- Registry CRUD correctness = 100%
- No security violations (NS-001–NS-007)

### Silver (≥ 70)
Usable in non-production environments. All critical and major issues resolved.

Adds:
- All MAJOR checks pass
- Search P99 ≤ 500ms at 1K objects
- Traceability chain completeness ≥ 0.80
- Quality overall score ≥ 0.70

### Gold (≥ 80)
Production-ready. All checks pass; latency targets met at scale.

Adds:
- All checks pass (CRITICAL + MAJOR + MINOR)
- Search P99 ≤ 100ms at 100K objects
- Registry P99 ≤ 5ms at 100K objects
- Traceability completeness = 1.0
- Hallucination rate ≤ 5%
- Scalability growth factor ≤ O(N log N)

### Enterprise (≥ 90)
Enterprise-grade: Gold + concurrency + scalability + security hardening.

Adds:
- Concurrency suite full pass
- Stress suite full pass
- Security suite full pass (zero violations)
- Performance at 1M objects within budget
- Recovery time ≤ 30 seconds

### Research (≥ 95)
Research-grade: full reproducibility, statistical significance, complete documentation.

Adds:
- All results statistically significant (p < 0.05)
- Benchmarks reproducible with variance ≤ 5%
- Full traceability from every check to specification
- Hallucination rate ≤ 1%

---

## Check Severity Levels

| Severity | Blocking at | Description |
|----------|-------------|-------------|
| CRITICAL | Bronze | Implementation is broken; certification cannot proceed |
| MAJOR | Silver | Significant deficiency that must be resolved |
| MINOR | Gold | Small issue that may be acceptable at lower levels |
| INFO | — | Informational; does not affect scoring |

---

## Scoring Formula

```
overall_score = Σ (domain_score[i] × weight[i])    for i in all domains

domain_score[i] = Σ (check_score[j] × check_weight[j])   for j in domain[i]

check_score[j] ∈ [0.0, 1.0]

All weights sum to 1.0 within each domain.
All domain weights sum to 1.0.
```

See `27-OVERALL-SCORING` for full formula and weights.

---

## Report Output

Every `kos-cert verify` run produces:

```
reports/
  certification-{timestamp}/
    summary.json          ← machine-readable overall result
    summary.md            ← human-readable summary
    domains/
      search.json
      registry.json
      graph.json
      ...
    dashboard.html        ← interactive HTML dashboard
    report.csv            ← tabular data
```

---

## Reproducibility Requirements

All certification runs must be reproducible:

| Requirement | Rule |
|-------------|------|
| Fixed random seed | `--seed {N}` (default: 42) |
| Dataset version pinned | `--dataset-version {hash}` |
| Implementation version pinned | `--impl-version {git-sha}` |
| Environment captured | OS, Python version, memory in report |
| Variance declared | P99 benchmarks include standard deviation |

---

## Certification Entry Point

```bash
# Full certification at Gold level
kos-cert verify --level gold --seed 42 --output reports/cert-$(date +%Y%m%d)/

# Domain-specific
kos-cert search --queries 1000 --seed 42

# Compare to previous
kos-cert evolution --baseline reports/cert-prev/ --current reports/cert-now/
```

---

## Cross-References

- Scoring formula → `27-OVERALL-SCORING`
- Domain suites → `02-SEARCH-CERTIFICATION` through `26-COVERAGE-CERTIFICATION`
- Dashboard spec → `28-DASHBOARD`
- CI integration → `29-CI-INTEGRATION`
