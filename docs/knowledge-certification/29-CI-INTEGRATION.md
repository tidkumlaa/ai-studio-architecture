# KNW-CERT-ARCH-029 — CI Integration

**Phase:** 3.0D.0  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Defines how the KOS Certification Suite integrates into the CI/CD pipeline — which checks run at each stage, what constitutes a pass, and how reports are published.

---

## CI Pipeline Stages

| Stage | Trigger | Suite | Max Duration | Fail Behaviour |
|-------|---------|-------|-------------|----------------|
| pre-commit | local hook | Smoke (15 checks) | 60s | Block commit |
| PR check | PR opened | Core (75 checks) | 10 min | Block merge |
| Merge gate | Main branch push | Full (all checks) | 2 hours | Block release |
| Nightly | 02:00 UTC | Full + Evolution | 3 hours | Alert on-call |
| Release | Tag push | Full + Evolution + Benchmarks | 4 hours | Block release |

---

## CI Configuration

```yaml
# .kos/ci.yaml — certification CI configuration

certification:
  dataset_version: "sha256:abc123"  # pinned
  seed: 42
  environment:
    python_min: "3.13"
    memory_min_gb: 8

stages:
  pre_commit:
    suite: smoke
    timeout_s: 60
    fail_on: any_failure

  pr_check:
    suite: core
    timeout_s: 600
    fail_on: critical_or_major_failure
    report_format: [markdown, json]
    post_to_pr: true

  merge_gate:
    suite: full
    timeout_s: 7200
    fail_on: critical_failure
    min_level: Silver
    report_format: [html, json, csv]
    archive_report: true

  nightly:
    suite: [full, evolution]
    timeout_s: 10800
    fail_on: regression
    alert_on_fail: true
    compare_to: "latest-merge-gate"

  release:
    suite: [full, evolution, benchmarks]
    timeout_s: 14400
    fail_on: any_failure
    min_level: Gold
    report_format: [html, markdown, json, csv]
    publish_report: true
```

---

## CI Pass/Fail Criteria

### pre-commit (Smoke)
- PASS: All 15 smoke checks pass
- FAIL: Any smoke check fails

### PR check (Core)
- PASS: All CRITICAL checks pass; ≥ 85% MAJOR checks pass
- FAIL: Any CRITICAL failure OR < 85% MAJOR pass rate

### Merge gate (Full)
- PASS: Overall score ≥ Silver (0.70); zero CRITICAL failures
- FAIL: Overall score < Silver OR any CRITICAL failure

### Nightly (Full + Evolution)
- PASS: No regressions from last merge gate; evolution score ≥ 0.45
- WARN: Minor regression (MINOR checks only)
- FAIL: Any CRITICAL regression

### Release (Full + Evolution + Benchmarks)
- PASS: Overall score ≥ Gold (0.80); all performance targets met; evolution score ≥ 0.50
- FAIL: Anything below Gold

---

## PR Comment Format

When `post_to_pr: true`, post a markdown summary:

```markdown
## KOS Certification — PR Check Results

**Result:** ✅ PASS (Core Layer)  
**Score:** 0.831 (Silver level)  
**Duration:** 4m 32s  
**Seed:** 42

| Domain | Score | Status |
|--------|-------|--------|
| Search | 0.831 | ✅ |
| Registry | 0.921 | ✅ |
| Graph | 0.974 | ✅ |
| Lifecycle | 0.893 | ✅ |
| Dependency | 0.891 | ✅ |
| Security | 0.933 | ✅ |

**Failed Checks (1):**
- `RC-022` (MINOR): Namespace collision detection — see [report](...)

[Full Report](reports/cert-2026-06-30/)
```

---

## Report Archival

Every certification run produces a report archived to `reports/`:

```
reports/
  cert-{timestamp}/
    summary.json          ← machine-readable
    summary.md            ← human-readable
    dashboard.html        ← interactive
    report.csv            ← tabular
    domains/
      search.json
      registry.json
      ...
    environment.json      ← OS, Python, memory, CPU
    dataset-manifest.yaml ← dataset hash
```

Reports are retained for 365 days. Release reports are retained indefinitely.

---

## Exit Codes

| Code | Meaning |
|------|---------|
| 0 | All checks passed at required level |
| 1 | CRITICAL check failed |
| 2 | MAJOR check failed (below level target) |
| 3 | MINOR check failed (warning only) |
| 4 | Suite timeout exceeded |
| 5 | Dataset integrity failure |
| 6 | Environment mismatch (Python version, memory) |
| 7 | Internal certification error |

---

## CLI

```bash
kos-cert verify --stage pre-commit       # smoke suite
kos-cert verify --stage pr               # core suite
kos-cert verify --stage release          # full + evolution + benchmarks
kos-cert verify --stage nightly          # full + evolution
kos-cert verify --min-level gold         # fail if < Gold
kos-cert verify --output reports/cert-$(date +%Y%m%d)/
```

---

## Cross-References

- Smoke suite checks → `25-REGRESSION-CERTIFICATION`
- Score formula → `27-OVERALL-SCORING`
- Report formats → `28-DASHBOARD`
- Knowledge CI pipeline → Phase 3.0C.5 `08-KNOWLEDGE-CI`
