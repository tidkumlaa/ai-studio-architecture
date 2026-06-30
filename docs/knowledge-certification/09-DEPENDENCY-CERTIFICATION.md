# KNW-CERT-ARCH-009 — Dependency Certification

**Phase:** 3.0D.0  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Certifies that the dependency resolution engine correctly resolves package dependency graphs, detects cycles, enforces version constraints, and handles diamond dependencies.

---

## Functional Checks

### Resolution

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Resolve simple dependency chain A→B→C | DC-001 | CRITICAL | Returns correct install order [C, B, A] |
| Resolve diamond A→B, A→C, B→D, C→D | DC-002 | MAJOR | Returns exactly one version of D |
| Version range satisfied | DC-003 | CRITICAL | Resolves to highest compatible version |
| Version exact pin honoured | DC-004 | CRITICAL | Returns exactly pinned version |
| Optional dependency absent: resolution succeeds | DC-005 | MAJOR | |
| Hard dependency absent: resolution fails | DC-006 | CRITICAL | Returns UnsatisfiableError |
| No package installed: empty resolution | DC-007 | MAJOR | Returns empty graph |

### Conflict Detection

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Diamond conflict (A needs D>=1.0; B needs D>=2.0) | DC-008 | MAJOR | Picks D=2.0 (highest satisfying both) |
| Unsatisfiable range conflict | DC-009 | CRITICAL | Returns DependencyConflictError |
| Conflict message names both conflicting packages | DC-010 | MINOR | Error includes package names and ranges |

### Cycle Detection

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Direct cycle (A depends on B, B depends on A) | DC-011 | CRITICAL | Returns CircularDependencyError |
| Indirect cycle (A→B→C→A) | DC-012 | CRITICAL | Returns CircularDependencyError |
| Cycle error includes cycle path | DC-013 | MINOR | |
| No cycle in standard 9-package graph | DC-014 | CRITICAL | Resolution succeeds |

### Lock File

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Lock file pins all resolved versions | DC-015 | MAJOR | Every package has exact version in lock |
| `--frozen` install uses lock file exactly | DC-016 | MAJOR | No resolution run; direct install |
| `--frozen` fails if lock outdated | DC-017 | MAJOR | Returns LockFileMismatchError |
| Lock file checksum matches packages | DC-018 | MAJOR | |

### Cross-Object Reference Checks

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Object references only packages in its dependencies | DC-019 | MAJOR | Linter catches cross-package violations |
| Unused declared dependency detected | DC-020 | MINOR | `kos deps unused` finds it |
| Missing declared dependency detected | DC-021 | MAJOR | `kos deps missing` finds it |

---

## Performance

| Operation | P50 | P99 | Graph size |
|-----------|-----|-----|------------|
| Resolve 9-package graph | ≤ 50ms | ≤ 200ms | 9 packages |
| Resolve 50-package graph | ≤ 200ms | ≤ 1s | 50 packages |
| Lock file write | ≤ 10ms | ≤ 50ms | |

---

## Dependency Graph Under Test

Standard 9-package KOS graph plus 4 synthetic conflict graphs:

```
CONFLICT-1: diamond dependency, resolvable
CONFLICT-2: diamond dependency, unresolvable
CONFLICT-3: direct cycle
CONFLICT-4: indirect cycle
```

All 4 conflict graphs have defined expected behaviour.

---

## Metrics

| Metric | Bronze | Silver | Gold |
|--------|--------|--------|------|
| Resolution correctness | = 1.0 | = 1.0 | = 1.0 |
| Cycle detection | = 1.0 | = 1.0 | = 1.0 |
| Conflict detection | ≥ 0.90 | ≥ 0.97 | = 1.0 |
| Lock file correctness | ≥ 0.90 | ≥ 0.97 | = 1.0 |

---

## Report Format

```json
{
  "domain": "dependency",
  "checks": {
    "DC-001": "PASS", "DC-002": "PASS",
    "DC-011": "PASS", "DC-012": "PASS",
    "DC-015": "FAIL"
  },
  "resolution_correctness": 1.0,
  "cycle_detection": 1.0,
  "conflict_detection": 0.90,
  "lock_file_correctness": 0.75,
  "domain_score": 0.891,
  "level_achieved": "Silver"
}
```

---

## CLI

```bash
kos-cert dependency                      # full dependency certification
kos-cert dependency --conflict-graphs    # run conflict graph tests only
kos-cert dependency --output reports/dependency.json
```

---

## Cross-References

- Dependency model → Phase 3.0C.5 `27-KNOWLEDGE-DEPENDENCY-MODEL`
- Rules DM-001–DM-006 → Phase 3.0C.5 `27-KNOWLEDGE-DEPENDENCY-MODEL`
- Package manager → Phase 3.0C.5 `26-KNOWLEDGE-PACKAGE-MANAGER`
