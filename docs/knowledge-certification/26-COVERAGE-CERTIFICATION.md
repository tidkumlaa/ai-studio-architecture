# KNW-CERT-ARCH-026 — Coverage Certification

**Phase:** 3.0D.0  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Certifies that the knowledge base covers the domains, object types, relationships, and traceability chains required for the KOS platform to function correctly. Low coverage means the system lacks the knowledge it needs.

---

## Coverage Dimensions

From Phase 3.0C.5 `18-KNOWLEDGE-COVERAGE`:

| Dim | Name | Formula | Gold Target |
|-----|------|---------|-------------|
| COV-1 | Type coverage | object types with ≥1 CANONICAL / 33 total | ≥ 0.90 |
| COV-2 | Domain coverage | domains with ≥1 CANONICAL / 9 total | = 1.0 |
| COV-3 | Requirement coverage | requirements with ≥1 module / total | ≥ 0.95 |
| COV-4 | Test coverage | modules with ≥1 test / total CANONICAL modules | ≥ 0.90 |
| COV-5 | Benchmark coverage | modules with ≥1 benchmark / total | ≥ 0.80 |
| COV-6 | Evidence coverage | objects with ≥1 evidence item / total | ≥ 0.95 |
| COV-7 | Relationship coverage | object types with ≥1 relationship / 33 | ≥ 0.85 |
| COV-8 | Lifecycle coverage | CANONICAL objects / total objects | ≥ 0.70 |

---

## Coverage Checks

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| COV-1 ≥ level target | CC-001 | MAJOR | |
| COV-2 = 1.0 | CC-002 | MAJOR | All 9 domains represented |
| COV-3 ≥ level target | CC-003 | MAJOR | |
| COV-4 ≥ level target | CC-004 | MAJOR | |
| COV-5 ≥ level target | CC-005 | MINOR | |
| COV-6 ≥ level target | CC-006 | MAJOR | |
| COV-7 ≥ level target | CC-007 | MINOR | |
| COV-8 ≥ level target | CC-008 | MINOR | |
| Zero object types with 0 CANONICAL objects | CC-009 | MAJOR | Gold+ |
| All 9 packages have ≥ 5 CANONICAL objects | CC-010 | MAJOR | |
| All 19 benchmark objects present | CC-011 | MAJOR | |
| All 33 templates exercised | CC-012 | MINOR | |

---

## Coverage Targets by Level

| Dimension | Bronze | Silver | Gold | Enterprise | Research |
|-----------|--------|--------|------|------------|---------|
| COV-1 (types) | ≥ 0.60 | ≥ 0.75 | ≥ 0.90 | ≥ 0.97 | = 1.0 |
| COV-2 (domains) | ≥ 0.78 | ≥ 0.89 | = 1.0 | = 1.0 | = 1.0 |
| COV-3 (requirements) | ≥ 0.70 | ≥ 0.82 | ≥ 0.95 | ≥ 0.99 | = 1.0 |
| COV-4 (tests) | ≥ 0.60 | ≥ 0.75 | ≥ 0.90 | ≥ 0.97 | = 1.0 |
| COV-5 (benchmarks) | ≥ 0.50 | ≥ 0.65 | ≥ 0.80 | ≥ 0.93 | = 1.0 |
| COV-6 (evidence) | ≥ 0.80 | ≥ 0.90 | ≥ 0.95 | ≥ 0.99 | = 1.0 |
| COV-7 (relationships) | ≥ 0.60 | ≥ 0.73 | ≥ 0.85 | ≥ 0.95 | = 1.0 |
| COV-8 (lifecycle) | ≥ 0.50 | ≥ 0.60 | ≥ 0.70 | ≥ 0.85 | ≥ 0.95 |

---

## Coverage Gap Report

The coverage report must include a gap analysis:

```yaml
# coverage-gaps.yaml
gaps:
  COV-1:
    missing_types:
      - CONFIGURATION     # 0 CANONICAL objects
      - DEPLOYMENT        # 0 CANONICAL objects
    present_types: 28
    total_types: 33
    coverage: 0.848

  COV-4:
    untested_modules:
      - KNW-PLT-MOD-007   # Quota v2 — no test
      - KNW-PLT-MOD-009   # Tenant Manager — no test
    tested: 14
    total: 16
    coverage: 0.875

  COV-5:
    unbenchmarked_modules:
      - KNW-RT-RT-003     # no benchmark
    benchmarked: 15
    total: 16
    coverage: 0.938
```

---

## Composite Coverage Score

```
coverage_score = (COV-1 × 0.15) + (COV-2 × 0.10) + (COV-3 × 0.20)
              + (COV-4 × 0.20) + (COV-5 × 0.10) + (COV-6 × 0.10)
              + (COV-7 × 0.10) + (COV-8 × 0.05)
```

---

## Report Format

```json
{
  "domain": "coverage",
  "dimensions": {
    "COV-1": {"value": 0.848, "target": 0.90, "pass": false},
    "COV-2": {"value": 1.0, "target": 1.0, "pass": true},
    "COV-3": {"value": 0.961, "target": 0.95, "pass": true},
    "COV-4": {"value": 0.875, "target": 0.90, "pass": false},
    "COV-5": {"value": 0.938, "target": 0.80, "pass": true},
    "COV-6": {"value": 0.980, "target": 0.95, "pass": true},
    "COV-7": {"value": 0.909, "target": 0.85, "pass": true},
    "COV-8": {"value": 0.750, "target": 0.70, "pass": true}
  },
  "composite_coverage_score": 0.913,
  "gap_report": "reports/coverage-gaps.yaml",
  "domain_score": 0.891,
  "level_achieved": "Silver"
}
```

---

## CLI

```bash
kos-cert coverage                        # all 8 dimensions
kos-cert coverage --dimension COV-4      # test coverage only
kos-cert coverage --gap-report           # gaps only
kos-cert coverage --output reports/coverage.json
```

---

## Cross-References

- Coverage model → Phase 3.0C.5 `18-KNOWLEDGE-COVERAGE`
- Traceability coverage → `05-TRACEABILITY-CERTIFICATION`
- Quality dimension QD-8 → Phase 3.0C.5 `17-KNOWLEDGE-SCORING`
- Overall score weight (10%) → `27-OVERALL-SCORING`
