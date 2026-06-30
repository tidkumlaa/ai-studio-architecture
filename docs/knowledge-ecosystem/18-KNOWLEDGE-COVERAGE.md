# KNW-KE-ARCH-018 — Knowledge Coverage

**Phase:** 3.0C.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Coverage measures how completely the Knowledge Ecosystem addresses the system's requirements, implementations, and tests. Low coverage means unknown territory — things that may exist in the runtime but are not captured as knowledge.

---

## Coverage Dimensions

| Dimension | What it measures | Target |
|-----------|-----------------|--------|
| COV-1: Requirement Coverage | % of requirements with ≥ 1 implementing object | ≥ 90% |
| COV-2: Architecture Coverage | % of modules with ≥ 1 architecture object | ≥ 80% |
| COV-3: Implementation Coverage | % of architecture objects with ≥ 1 implementing module | ≥ 80% |
| COV-4: Test Coverage | % of modules with ≥ 1 test object | ≥ 85% |
| COV-5: Evidence Coverage | % of CANONICAL objects with ≥ 1 evidence record | ≥ 95% |
| COV-6: Benchmark Coverage | % of algorithms with ≥ 1 benchmark object | 100% |
| COV-7: Relationship Coverage | % of objects with ≥ 1 relationship | ≥ 70% |
| COV-8: Documentation Coverage | % of packages with complete README | 100% |

---

## Coverage Formulas

### COV-1: Requirement Coverage

```
requirements = registry.list_by_type("requirement")
covered = [r for r in requirements if registry.list_by_tag("satisfies:" + r.knowledge_id)]
                                    # or: r.fulfilled_by is non-empty
cov_1 = len(covered) / max(1, len(requirements))
```

### COV-2: Architecture Coverage

```
modules = registry.list_by_type("module")
covered = [m for m in modules if m.traceability.implements]
          # implements points to an ARCHITECTURE object
cov_2 = len(covered) / max(1, len(modules))
```

### COV-3: Implementation Coverage

```
arch_objects = registry.list_by_type("architecture")
covered = [a for a in arch_objects if any_module_implements(a.knowledge_id, registry)]
cov_3 = len(covered) / max(1, len(arch_objects))
```

### COV-4: Test Coverage

```
modules = registry.list_by_type("module") + registry.list_by_type("service")
covered = [m for m in modules if m.traceability.tests]
cov_4 = len(covered) / max(1, len(modules))
```

### COV-5: Evidence Coverage

```
canonical = registry.list_by_status("CANONICAL")
with_evidence = [o for o in canonical if o.evidence.evidence_score > 0.0]
cov_5 = len(with_evidence) / max(1, len(canonical))
```

### COV-6: Benchmark Coverage

```
algorithms = registry.list_by_type("algorithm")
benchmarks = registry.list_by_type("benchmark")
bench_targets = {b.traceability.satisfies[0] for b in benchmarks if b.traceability.satisfies}
covered = [a for a in algorithms if a.knowledge_id in bench_targets]
cov_6 = len(covered) / max(1, len(algorithms))
```

### COV-7: Relationship Coverage

```
all_objects = registry.list_all()
with_rels = [o for o in all_objects if o.relationships]  # at least 1 relationship
cov_7 = len(with_rels) / max(1, len(all_objects))
```

### COV-8: Documentation Coverage

```
packages = list_all_packages()
with_readme = [p for p in packages if exists(p.path / "docs/README.md")]
cov_8 = len(with_readme) / max(1, len(packages))
```

---

## Coverage Report Format

```yaml
# knowledge/reports/coverage-{date}.yaml
generated_at: "2026-06-30T00:00:00Z"
overall_coverage: 0.0    # weighted average of all dimensions

dimensions:
  COV-1:
    name: "Requirement Coverage"
    score: 0.0
    target: 0.90
    status: BELOW_TARGET   # PASS | BELOW_TARGET | CRITICAL
    covered: 0
    total: 0
    gaps: []   # knowledge_ids of uncovered requirements

  COV-2:
    name: "Architecture Coverage"
    score: 0.0
    target: 0.80
    status: BELOW_TARGET
    covered: 0
    total: 0
    gaps: []

  # ... COV-3 through COV-8

gaps_by_type:
  requirement: []
  module: []
  architecture: []
```

---

## Gap Classification

| Gap Severity | Condition |
|-------------|-----------|
| CRITICAL | COV-1 < 0.70 OR COV-4 < 0.70 OR COV-6 < 1.00 |
| HIGH | Any dimension < target by > 15% |
| MEDIUM | Any dimension < target by 5–15% |
| LOW | Any dimension < target by < 5% |

---

## Coverage Commands

```bash
kos coverage report                    # generate full coverage report
kos coverage report --dimension COV-4  # single dimension
kos coverage gaps --dimension COV-1    # list uncovered requirements
kos coverage gate                      # check all dimension targets
kos coverage history                   # compare to previous reports
```

---

## Coverage Gate in CI

From `08-KNOWLEDGE-CI`, nightly pipeline:

```yaml
- name: coverage-report
  command: kos coverage report --output reports/
  fail_on: CRITICAL    # fail if any CRITICAL gap
```

Release gate checks all dimension targets.

---

## Cross-References

- Traceability chain → `29-KNOWLEDGE-TRACEABILITY`
- Coverage report output → `knowledge/reports/`
- Quality scoring uses coverage → `17-KNOWLEDGE-SCORING` QD-3
- CI runs coverage check → `08-KNOWLEDGE-CI`
