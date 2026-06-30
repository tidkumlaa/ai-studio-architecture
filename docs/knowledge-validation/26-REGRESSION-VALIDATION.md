# KVF-DOC-026 — Regression Validation

**Phase:** 3.0D.1.5
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Regression Validation compares the current KOS Runtime implementation against
a previous baseline to detect quality, performance, or search regressions.
It ensures that improvements in one area do not silently degrade another.

---

## Regression Test Scope

```
Regression covers:
  RR-1: Search Regression (search quality metrics)
  RR-2: Reasoning Regression (correct/hallucination rate)
  RR-3: Coverage Regression (knowledge coverage %)
  RR-4: Performance Regression (latency, memory)
  RR-5: Knowledge Health Regression (conformance, quality scores)
```

---

## Baseline Definition

```
Baseline:
  The result of the most recent FULL validation run that achieved
  SILVER or higher certification.
  Stored as: validation_baseline_{timestamp}.json

  If no baseline exists: run FULL validation first to create baseline.

Current:
  The current validation run being compared.
```

---

## Regression Metrics

### RR-1: Search Regression

```
For each search metric M:
  delta(M) = current(M) - baseline(M)

  Regression threshold:
    Top-1 accuracy:  delta < -0.05 → WARNING; delta < -0.10 → CRITICAL
    MRR:            delta < -0.05 → WARNING; delta < -0.10 → CRITICAL
    NDCG@10:        delta < -0.05 → WARNING; delta < -0.08 → CRITICAL
    P99 latency:    delta > +20ms  → WARNING; delta > +40ms → CRITICAL
```

### RR-2: Reasoning Regression

```
  Correct rate:        delta < -0.05 → WARNING; delta < -0.10 → CRITICAL
  Hallucination rate:  delta > +0.02 → WARNING; delta > +0.05 → CRITICAL
  Unknown rate:        delta > +0.05 → WARNING (increased unknown = reduced coverage)
```

### RR-3: Coverage Regression

```
  Mean coverage:       delta < -0.05 → WARNING; delta < -0.10 → CRITICAL
  Primary correct:     delta < -0.01 → WARNING; delta < -0.05 → CRITICAL
```

### RR-4: Performance Regression

```
  P99 assembly:        delta > +10ms → WARNING; delta > +30ms → CRITICAL
  P50 assembly:        delta > +5ms  → WARNING; delta > +15ms → CRITICAL
  Peak RSS:            delta > +200MB → WARNING; delta > +500MB → CRITICAL
```

### RR-5: Knowledge Health Regression

```
  Conformance score:   delta < -0.02 → WARNING; delta < -0.05 → CRITICAL
  Mean quality score:  delta < -0.02 → WARNING; delta < -0.05 → CRITICAL
  AIRS mean:           delta < -0.02 → WARNING; delta < -0.03 → CRITICAL
```

---

## Regression Detection Algorithm

```
COMPARE(current_result, baseline_result):

  regressions = []
  improvements = []

  FOR each metric M in [RR-1 through RR-5]:
    delta = current[M] - baseline[M]
    severity = classify(delta, M)

    IF severity in {WARNING, CRITICAL}:
      regressions.append(RegressionEvent(metric=M, delta=delta, severity=severity))
    ELIF delta > improvement_threshold(M):
      improvements.append(ImprovementEvent(metric=M, delta=delta))

  RETURN RegressionReport(
    regressions=regressions,
    improvements=improvements,
    overall_status = CRITICAL if any CRITICAL else
                     WARNING if any WARNING else
                     IMPROVED if any improvements else
                     STABLE
  )
```

---

## Regression Report Schema

```yaml
regression_report:
  current_run_id: string
  baseline_run_id: string
  baseline_timestamp: string
  comparison_timestamp: string

  overall_status: CRITICAL | WARNING | STABLE | IMPROVED

  regressions:
    - metric: string
      category: string                  # RR-1 through RR-5
      baseline_value: float
      current_value: float
      delta: float
      severity: WARNING | CRITICAL

  improvements:
    - metric: string
      baseline_value: float
      current_value: float
      delta: float

  summary:
    critical_count: integer
    warning_count: integer
    improvement_count: integer
    stable_count: integer
    metrics_total: integer
```

---

## Regression Handling Policy

```
If regression_status == CRITICAL:
  Block deployment until root cause identified and resolved.
  Must run FULL validation after fix.

If regression_status == WARNING:
  Document root cause in deployment notes.
  Deploy is allowed but must be monitored.
  Schedule re-validation within 7 days.

If regression_status == STABLE or IMPROVED:
  Update baseline to current run.
  Deploy proceeds normally.
```

---

## Rules

| Rule | Statement |
|------|-----------|
| KVF-126 | Baseline must be a SILVER+ validation run — BRONZE baselines are not accepted |
| KVF-127 | Hallucination rate regression > +0.05 is CRITICAL regardless of other metrics |
| KVF-128 | Primary correct rate delta < -0.05 is CRITICAL — this is the most important coverage metric |
| KVF-129 | Regression comparison must use the same test corpus as the baseline — different corpus = invalid |
| KVF-130 | Improvements must also be recorded — regression detection is not one-directional |

---

## Cross-References

- Search quality → `08-SEARCH-QUALITY`
- Hallucination measurement → `10-HALLUCINATION-MEASUREMENT`
- Knowledge coverage → `07-KNOWLEDGE-COVERAGE`
- Performance validation → `24-PERFORMANCE-VALIDATION`
- Certification scoring → `27-CERTIFICATION-SCORING`
