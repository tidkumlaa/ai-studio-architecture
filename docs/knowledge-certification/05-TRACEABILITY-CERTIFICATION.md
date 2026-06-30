# KNW-CERT-ARCH-005 — Traceability Certification

**Phase:** 3.0D.0  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Certifies that the full traceability chain — from Requirement through Architecture Decision, Specification, Module, Service, API, Test, Deployment, and Monitoring — is complete, bidirectional, and traversable.

---

## Traceability Chain

Every requirement must trace through all 9 levels:

```
Requirement (KNW-{D}-REQ-NNN)
    ↓ satisfies
Architecture (KNW-{D}-ARCH-NNN)
    ↓ implements
Decision (KNW-{D}-DEC-NNN)
    ↓ specifies
Specification (KNW-{D}-SPEC-NNN)
    ↓ implemented_by
Module (KNW-{D}-MOD-NNN)
    ↓ exposed_by
Service (KNW-{D}-SVC-NNN)
    ↓ contracted_by
API (KNW-{D}-API-NNN)
    ↓ verified_by
Test (KNW-{D}-TST-NNN)
    ↓ deployed_by
Deployment (KNW-{D}-DEP-NNN)
    ↓ monitored_by
Monitoring (KNW-{D}-MON-NNN)
```

---

## Test Procedure

1. Load the Traceability Dataset (500 requirements from the golden dataset)
2. For each requirement R:
   - Traverse the full 9-level chain from R downward
   - Traverse the full chain upward from Monitoring back to R
   - Verify bidirectionality at each link
   - Record chain completeness score

---

## Chain Checks

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Full downward chain reachable | TC-001 | CRITICAL | All 9 levels reachable from requirement |
| Full upward chain reachable | TC-002 | CRITICAL | All 9 levels reachable from monitoring |
| Bidirectionality at each link | TC-003 | CRITICAL | Both directions traversable |
| No dangling references | TC-004 | MAJOR | Every link target exists in registry |
| Chain completeness ≥ 0.90 | TC-005 | MAJOR | ≥ 90% of requirements have full chains |
| Chain completeness = 1.0 for CANONICAL | TC-006 | MAJOR (Gold+) | All CANONICAL requirements fully traced |
| No broken link in 500 chains | TC-007 | MAJOR | All links resolve |
| Chain traversal time ≤ 50ms P99 | TC-008 | MINOR | Per chain |
| Cross-package traceability | TC-009 | MAJOR | Chains crossing package boundaries work |
| Alias links resolved | TC-010 | MAJOR | Aliases in chain links resolve correctly |

---

## Completeness Metrics

| Metric | Formula | Bronze | Silver | Gold | Enterprise |
|--------|---------|--------|--------|------|------------|
| Chain completeness | complete_chains / 500 | ≥ 0.70 | ≥ 0.80 | ≥ 0.95 | = 1.0 |
| Link coverage | non-null links / total links | ≥ 0.75 | ≥ 0.85 | ≥ 0.97 | = 1.0 |
| Bidirectional coverage | bidi_links / total links | ≥ 0.70 | ≥ 0.82 | ≥ 0.96 | = 1.0 |
| Cross-package coverage | cross_pkg_links / total | ≥ 0.60 | ≥ 0.75 | ≥ 0.90 | ≥ 0.98 |

---

## Matrix Output

The traceability certification produces a matrix:

```
REQ                  ARCH    DEC     SPEC    MOD     SVC     API     TST     DEP     MON     COMPLETE
KNW-PLT-REQ-001      ✓       ✓       ✓       ✓       ✓       ✓       ✓       ✓       ✓       YES
KNW-PLT-REQ-002      ✓       ✓       ✓       ✓       ✗       ✗       ✓       ✗       ✗       NO
KNW-PLT-REQ-003      ✓       ✓       ✓       ✓       ✓       ✓       ✓       ✓       ✓       YES
...
TOTAL (500)                                                                          445/500  89.0%
```

---

## Report Format

```json
{
  "domain": "traceability",
  "requirements_tested": 500,
  "chains_complete": 445,
  "chain_completeness": 0.890,
  "link_coverage": 0.934,
  "bidirectional_coverage": 0.921,
  "checks": {
    "TC-001": "PASS",
    "TC-002": "PASS",
    "TC-003": "PASS",
    "TC-004": "PASS",
    "TC-005": "PASS",
    "TC-006": "FAIL",
    "TC-007": "PASS",
    "TC-008": "PASS",
    "TC-009": "PASS",
    "TC-010": "PASS"
  },
  "broken_requirements": ["KNW-PLT-REQ-002", "..."],
  "traversal_p99_ms": 38.4,
  "domain_score": 0.847,
  "level_achieved": "Silver"
}
```

---

## CLI

```bash
kos-cert traceability                    # 500-requirement batch
kos-cert traceability --requirements 100 # quick check
kos-cert traceability --report-matrix    # output full matrix
kos-cert traceability --output reports/traceability.json
```

---

## Cross-References

- Traceability model → Phase 3.0C.5 `29-KNOWLEDGE-TRACEABILITY`
- Traceability rules TR-001–TR-010 → Phase 3.0C.5 `29-KNOWLEDGE-TRACEABILITY`
- Bidirectionality linter rule KL-031 → Phase 3.0C.5 `06-KNOWLEDGE-LINTER`
- Overall score weight (10%) → `27-OVERALL-SCORING`
