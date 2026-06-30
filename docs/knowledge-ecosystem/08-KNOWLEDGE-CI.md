# KNW-KE-ARCH-008 — Knowledge CI

**Phase:** 3.0C.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

The Knowledge CI pipeline enforces all repository rules automatically. No object reaches CANONICAL status without passing CI. No PR merges to main without a green pipeline.

---

## Pipeline Stages

```
[pre-commit] → [pr-check] → [merge-gate] → [nightly] → [release]
```

---

## Stage 1: Pre-commit

Runs locally on `git commit` via hook:

```bash
# .git/hooks/pre-commit (or pre-commit framework)
kos format --check {staged_yaml_files}
kos lint --severity error {staged_yaml_files}
```

| Check | Fail Action |
|-------|------------|
| Format check | Block commit; run `kos format` to fix |
| Lint errors | Block commit; must fix manually |

Pre-commit is fast (< 2s per file). Only checks staged files.

---

## Stage 2: PR Check

Runs on every pull request (GitHub Actions / similar):

```yaml
# .github/workflows/knowledge-ci.yml (schema)
stages:
  - name: schema-validation
    command: kos validate --schemas knowledge/schemas/
    timeout_s: 30

  - name: lint
    command: kos lint knowledge/packages/
    timeout_s: 60
    fail_on: error

  - name: format-check
    command: kos format --check knowledge/packages/
    timeout_s: 30

  - name: reference-check
    command: kos lint --rule KL-015,KL-016,KL-017,KL-018 --check-external-refs
    timeout_s: 60

  - name: package-verify
    command: kos verify package {changed_packages}
    timeout_s: 120

  - name: quality-gate
    command: kos quality-gate --min-score 0.70 {changed_objects}
    timeout_s: 30
```

**PR check gates:**

| Gate | Pass Condition |
|------|---------------|
| Schema validation | All YAML files validate against their JSON Schema |
| Lint | Zero ERROR-level violations |
| Format | All files pass `kos format --check` |
| Reference | All relationships point to existing objects |
| Package verify | `kos verify package {name}` passes |
| Quality gate | All changed objects ≥ 0.70 overall score |

---

## Stage 3: Merge Gate

Runs before any PR merges to `main`:

```
[PR Check passes]
     │
     ▼
[reviewer-approval: ≥ 1 reviewer]
     │
     ▼
[if CANONICAL object changed: Architecture Board review required]
     │
     ▼
[merge-gate pipeline]
```

Merge gate pipeline:

```yaml
stages:
  - name: full-lint
    command: kos lint knowledge/           # full repo, not just changed
    timeout_s: 300

  - name: dependency-check
    command: kos verify dependencies
    timeout_s: 120

  - name: cross-reference-verify
    command: kos verify cross-references
    timeout_s: 120

  - name: golden-dataset-regression
    command: kos verify golden-dataset
    timeout_s: 300
```

---

## Stage 4: Nightly

Runs every night at 00:00 UTC:

| Job | Command | Purpose |
|-----|---------|---------|
| freshness-check | `kos quality freshness` | Flag objects with stale evidence |
| confidence-decay | `kos quality confidence-decay` | Recompute decayed confidence |
| quality-report | `kos quality report --output reports/` | Generate quality report |
| broken-link-check | `kos lint --rule KL-015,KL-016` | Catch external ref rot |
| duplicate-detect | `kos verify duplicates` | Semantic duplicate scan |
| coverage-report | `kos coverage report` | Traceability coverage report |

---

## Stage 5: Release

Before any package release:

```yaml
stages:
  - name: full-verify
    command: kos verify --all
    timeout_s: 600

  - name: benchmark-check
    command: kos verify benchmarks
    timeout_s: 120

  - name: schema-freeze-check
    command: kos verify schema-freeze
    timeout_s: 30

  - name: lock-file-check
    command: kos verify lock-files
    timeout_s: 30

  - name: changelog-check
    command: kos verify changelog {package}
    timeout_s: 10
```

**Release gate: all stages must pass with exit code 0.**

---

## CI Configuration File

```yaml
# knowledge/ci-config.yaml
version: "1.0.0"
pipelines:
  pre_commit:
    enabled: true
    checks: [format-check, lint-errors]

  pr_check:
    enabled: true
    checks: [schema, lint, format, references, package-verify, quality-gate]
    quality_gate_min: 0.70

  merge_gate:
    enabled: true
    checks: [full-lint, dependencies, cross-references, golden-dataset]
    require_reviewer: true
    canonical_require_board: true

  nightly:
    enabled: true
    schedule: "0 0 * * *"   # cron

  release:
    enabled: true
    checks: [full-verify, benchmarks, schema-freeze, lock-files, changelog]
```

---

## Failure Response

| Failure Type | Action |
|-------------|--------|
| Pre-commit lint error | `kos format` then re-commit; fix lint errors |
| PR quality gate fail | Add evidence; improve completeness |
| Reference broken | Fix or create the referenced object |
| Nightly freshness warning | Refresh evidence or set evidence as `ACCEPTED_STALE` |
| Release verify fail | Block release; fix all errors |

---

## Cross-References

- Linter checks run in CI → `06-KNOWLEDGE-LINTER`
- Formatter run in CI → `07-KNOWLEDGE-FORMATTER`
- Verification suite → `20-KNOWLEDGE-VERIFICATION`
- Quality gate thresholds → `16-KNOWLEDGE-QUALITY`
- Package verify → `02-KNOWLEDGE-PACKAGES`
