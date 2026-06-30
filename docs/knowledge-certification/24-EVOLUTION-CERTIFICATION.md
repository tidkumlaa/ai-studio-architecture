# KNW-CERT-ARCH-024 — Evolution Certification

**Phase:** 3.0D.0  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Certifies that the KOS platform evolves correctly — new versions improve or maintain all key metrics without introducing regressions. Evolution certification compares the current implementation against the most recent certified baseline.

---

## Comparison Model

```
Baseline (previous certified version)
    │
    ├── Certification Report (archived)
    ├── Dataset hash (pinned)
    └── Environment spec

Current (version under test)
    │
    ├── Run same certification suite
    ├── Same dataset hash
    └── Same seed

Evolution Score = Σ (metric_delta[i] × weight[i])
```

---

## Metrics Compared

| Domain | Metric | Direction | Weight |
|--------|--------|-----------|--------|
| Search | Top1 Accuracy | higher is better | 0.15 |
| Reasoning | Correct rate | higher is better | 0.15 |
| Hallucination | Hallucination rate | lower is better | 0.15 |
| Coverage | Overall coverage | higher is better | 0.10 |
| Graph | Correctness | higher is better | 0.10 |
| Registry | Correctness | higher is better | 0.10 |
| Performance | P99 latency (search) | lower is better | 0.10 |
| Quality | Mean overall score | higher is better | 0.10 |
| Knowledge health | Health score | higher is better | 0.05 |

---

## Evolution Checks

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Search Top1 accuracy not regressed | EVC-001 | MAJOR | Δ ≥ −0.02 (max 2% regression allowed) |
| Hallucination rate not increased | EVC-002 | CRITICAL | Δ ≤ 0.01 (max 1% increase allowed) |
| Correct rate not regressed | EVC-003 | MAJOR | Δ ≥ −0.02 |
| Registry correctness not regressed | EVC-004 | CRITICAL | Δ ≥ 0.0 (no regression allowed) |
| Graph correctness not regressed | EVC-005 | MAJOR | Δ ≥ −0.01 |
| Performance P99 not regressed | EVC-006 | MAJOR | Δ ≤ +20% (20% increase allowed) |
| Coverage not regressed | EVC-007 | MAJOR | Δ ≥ −0.02 |
| Mean quality score not regressed | EVC-008 | MINOR | Δ ≥ −0.01 |

---

## Evolution Score Formula

```
For each metric m:
  if direction == "higher is better":
    delta_score[m] = clamp((current[m] - baseline[m]) / baseline[m], -1, +1)
  if direction == "lower is better":
    delta_score[m] = clamp((baseline[m] - current[m]) / baseline[m], -1, +1)

evolution_score = Σ (delta_score[m] × weight[m]) + 0.5
# 0.5 offset: 0.5 = no change; >0.5 = improved; <0.5 = regressed

# Range: [0.0 (total regression), 1.0 (massive improvement)]
# Pass: evolution_score ≥ 0.45 (allows minor regression across the board)
# Gold: evolution_score ≥ 0.50 (no net regression)
```

---

## Knowledge Health Tracking

Track the health of the knowledge base itself (not just the engine):

| Metric | Formula |
|--------|---------|
| Health score | mean quality score of all CANONICAL objects |
| Coverage score | % object types with ≥ 5 CANONICAL objects |
| Traceability score | % CANONICAL objects with complete trace chain |
| Evidence freshness | % evidence items within freshness window |

These are tracked across versions to detect knowledge rot.

---

## Version Comparison Report

```yaml
# reports/evolution-{version}-vs-{baseline}.yaml
current_version: "1.1.0"
baseline_version: "1.0.0"
dataset_hash: "sha256:abc123"
seed: 42

metrics:
  search_top1:
    baseline: 0.831
    current: 0.847
    delta: +0.016
    direction: higher_better
    pass: true

  hallucination_rate:
    baseline: 0.050
    current: 0.036
    delta: -0.014
    direction: lower_better
    pass: true

  performance_p99_ms:
    baseline: 100.0
    current: 87.3
    delta: -12.7%
    direction: lower_better
    pass: true

evolution_score: 0.623
level_achieved: Gold
```

---

## CLI

```bash
kos-cert evolution                       # compare vs latest baseline
kos-cert evolution --baseline reports/cert-2026-06-01/ --current reports/cert-2026-06-30/
kos-cert evolution --metrics search,reasoning,hallucination
kos-cert evolution --output reports/evolution.json
```

---

## Cross-References

- Regression suite → `25-REGRESSION-CERTIFICATION`
- Dashboard trend view → `28-DASHBOARD`
- Overall score weight (10%) → `27-OVERALL-SCORING`
