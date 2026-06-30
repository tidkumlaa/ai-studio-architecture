# KNW-KE-ARCH-029 — Knowledge Traceability

**Phase:** 3.0C.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Every Knowledge Object must be traceable — from architectural decision to specification to implementation to test to benchmark. This document defines the traceability model, chain format, and verification rules.

---

## Traceability Model

Traceability has two dimensions:

1. **Vertical traceability** — across levels of abstraction (requirement → module → test)
2. **Horizontal traceability** — across objects at the same level (module → algorithm it uses)

```
Architecture Decision (ADR)
    │ satisfies
    ▼
Requirement (KNW-PLT-REQ-NNN)
    │ satisfies
    ▼
Module / Service (KNW-PLT-MOD-NNN)
    │ implements
    ▼
Algorithm / Pattern (KNW-ALG-ALG-NNN)
    │ tested_by
    ▼
Test (KNW-TEST-TST-NNN)
    │ benchmarked_by
    ▼
Benchmark (KNW-TEST-BENCH-NNN)
```

---

## Traceability Fields

Every Knowledge Object carries a `traceability` block:

```yaml
traceability:
  satisfies:            # list of requirement IDs this object satisfies
    - KNW-PLT-REQ-001
    - KNW-PLT-REQ-002
  implements:           # list of abstract/interface IDs this object implements
    - KNW-ALG-ALG-007
  tests:                # list of object IDs this object tests (used on TEST objects)
    - KNW-PLT-MOD-001
  tested_by:            # list of test IDs that verify this object
    - KNW-TEST-TST-001
  benchmarked_by:       # list of benchmark IDs for this object
    - KNW-TEST-BENCH-001
  derived_from:         # source object (if this is a derived/specialized version)
    - KNW-META-PAT-003
  supersedes:           # object IDs this replaces
    - KNW-PLT-MOD-000   # retired predecessor
```

---

## Traceability Link Types

| Link | Direction | Meaning |
|------|-----------|---------|
| `satisfies` | object → requirement | Object satisfies this requirement |
| `implements` | object → interface/algorithm | Object implements this specification |
| `tests` | test → object | Test verifies this object |
| `tested_by` | object → test | Object is covered by this test |
| `benchmarked_by` | object → benchmark | Object is measured by this benchmark |
| `derived_from` | object → source | Object is derived from this object |
| `supersedes` | new → old | New object replaces old object |

---

## Traceability Rules

| Rule | Description |
|------|-------------|
| TR-001 | Every CANONICAL MODULE must `satisfies` at least one REQUIREMENT |
| TR-002 | Every CANONICAL MODULE must have at least one entry in `tested_by` |
| TR-003 | Every TEST object must populate `tests` with at least one target |
| TR-004 | Every BENCHMARK object must populate `tests` with at least one target |
| TR-005 | `satisfies` IDs must exist and be of type REQUIREMENT |
| TR-006 | `implements` IDs must exist and be of compatible type |
| TR-007 | `tested_by` IDs must exist and be of type TEST |
| TR-008 | `benchmarked_by` IDs must exist and be of type BENCHMARK |
| TR-009 | `derived_from` IDs must exist |
| TR-010 | Bidirectionality: if A.tested_by = [B], then B.tests must include A |

---

## Bidirectionality Enforcement

The linter (KL-031) checks bidirectional links at PR time:

```
For each object O with traceability.tested_by = [T1, T2, ...]:
  For each Ti:
    Assert Ti.traceability.tests contains O.knowledge_id

For each object O with traceability.satisfies = [R1, R2, ...]:
  For each Ri:
    Assert Ri object exists and has type REQUIREMENT
```

If a link is one-directional, the linter emits `KL-031: broken-bidirectional-link`.

---

## Traceability Matrix

The catalog generates a traceability matrix:

```bash
kos catalog matrix --package kos.platform.package
```

Output format:
```
REQUIREMENT       MODULE            TEST              BENCHMARK
KNW-PLT-REQ-001  KNW-PLT-MOD-001  KNW-TEST-TST-001  KNW-TEST-BENCH-001
KNW-PLT-REQ-001  KNW-PLT-MOD-002  KNW-TEST-TST-003  KNW-TEST-BENCH-003
KNW-PLT-REQ-002  KNW-PLT-MOD-001  KNW-TEST-TST-001  (none)
KNW-PLT-REQ-003  (UNTESTED)       (none)             (none)
```

Untested requirements (no TEST entry) are flagged in the CI report.

---

## Traceability Coverage

Coverage is calculated as:

```
REQUIREMENT_COVERAGE = |{R: R has ≥1 MODULE}| / |{R: all requirements}|
TEST_COVERAGE        = |{M: M has ≥1 TEST}|   / |{M: all CANONICAL modules}|
BENCH_COVERAGE       = |{M: M has ≥1 BENCH}|  / |{M: all CANONICAL modules}|
```

See `18-KNOWLEDGE-COVERAGE` for full coverage model.

---

## Traceability Commands

```bash
kos trace KNW-PLT-MOD-001          # show full trace chain for an object
kos trace --up KNW-PLT-MOD-001     # trace up to requirements
kos trace --down KNW-PLT-MOD-001   # trace down to tests and benchmarks
kos trace matrix --package plt     # generate full traceability matrix
kos trace gaps --package plt       # list objects with missing traces
kos trace verify KNW-PLT-MOD-001   # verify bidirectionality for an object
```

---

## Trace Chain Example

```
kos trace KNW-PLT-MOD-001

KNW-PLT-MOD-001 (Quota Manager, MODULE)
├── satisfies:
│   └── KNW-PLT-REQ-001 (quota limits must be enforced per session)
├── implements:
│   └── KNW-ALG-ALG-003 (Token Bucket Rate Limiter)
├── tested_by:
│   └── KNW-TEST-TST-001 (Test: Quota enforcement)
├── benchmarked_by:
│   └── KNW-TEST-BENCH-001 (Bench: Quota performance)
└── derived_from: (none)
```

---

## Cross-References

- Coverage of traceability links → `18-KNOWLEDGE-COVERAGE`
- Canonical source traceability → `23-KNOWLEDGE-CANONICAL-SOURCES`
- Linter rule KL-031 (bidirectionality) → `06-KNOWLEDGE-LINTER`
- Test objects tracing modules → Phase 3.0C `13-TEST-REGISTRY`
