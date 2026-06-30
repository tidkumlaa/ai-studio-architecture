# CAE-DOC-023 — Context Metrics

**Phase:** 3.0D.1
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Context Metrics define the 30 observable metrics for the Context Assembly Engine.
These metrics are the operational ground truth for CAE health, quality, and
continuous improvement.

---

## Metric Registry

### M-GROUP-1: Assembly Latency (CAE-LT)

| ID | Name | Formula | Target | Alert |
|----|------|---------|--------|-------|
| CAE-M-001 | assembly_latency_p50 | 50th percentile E2E latency | < 50ms | > 60ms |
| CAE-M-002 | assembly_latency_p95 | 95th percentile E2E latency | < 80ms | > 100ms |
| CAE-M-003 | assembly_latency_p99 | 99th percentile E2E latency | < 120ms | > 120ms |
| CAE-M-004 | stage_latency_max | Max single-stage P99 latency | < 50ms | > 60ms |

---

### M-GROUP-2: Assembly Volume (CAE-VOL)

| ID | Name | Formula | Target | Alert |
|----|------|---------|--------|-------|
| CAE-M-005 | assemblies_per_second | count(packs emitted) / second | ≥ 100 | < 50 |
| CAE-M-006 | assemblies_failed | count(AssemblyError) / total | < 1% | > 3% |
| CAE-M-007 | assemblies_degraded | count(degraded mode) / total | < 5% | > 10% |

---

### M-GROUP-3: Cache Performance (CAE-CACHE)

| ID | Name | Formula | Target | Alert |
|----|------|---------|--------|-------|
| CAE-M-008 | cache_hit_rate_l1 | L1 hits / total lookups | ≥ 60% | < 50% |
| CAE-M-009 | cache_hit_rate_combined | (L1+L2) hits / total lookups | ≥ 85% | < 70% |
| CAE-M-010 | cache_invalidation_latency | Time from object update to invalidation | < 100ms | > 200ms |
| CAE-M-011 | prefetch_hit_rate | Prefetched entries used / total prefetched | ≥ 40% | < 25% |

---

### M-GROUP-4: Context Quality (CAE-QUAL)

| ID | Name | Formula | Target | Alert |
|----|------|---------|--------|-------|
| CAE-M-012 | context_quality_mean | mean(context_quality_score per pack) | ≥ 0.80 | < 0.75 |
| CAE-M-013 | context_quality_below_threshold | count(score < 0.70) / total | < 2% | > 5% |
| CAE-M-014 | budget_utilization_mean | mean(total_tokens / budget_declared) | 0.70–0.95 | < 0.50 or > 1.00 |
| CAE-M-015 | objects_dropped_rate | count(dropped objects) / total objects selected | < 10% | > 20% |

---

### M-GROUP-5: Confidence and Safety (CAE-SAFE)

| ID | Name | Formula | Target | Alert |
|----|------|---------|--------|-------|
| CAE-M-016 | context_confidence_mean | mean(context_confidence per pack) | ≥ 0.80 | < 0.70 |
| CAE-M-017 | airs_exclusion_rate | count(objects excluded AIRS<0.60) / total candidates | < 5% | > 15% |
| CAE-M-018 | hallucination_flag_rate | count(packs flagged by consumer) / packs emitted | < 1% | > 3% |
| CAE-M-019 | deprecated_primary_rate | count(packs with deprecated primary) / total | < 0.1% | > 1% |

---

### M-GROUP-6: Intent Resolution (CAE-INTENT)

| ID | Name | Formula | Target | Alert |
|----|------|---------|--------|-------|
| CAE-M-020 | intent_resolution_rate | count(resolved) / total queries | ≥ 95% | < 90% |
| CAE-M-021 | entity_ambiguity_rate | count(AMBIGUOUS resolution) / total | < 10% | > 20% |
| CAE-M-022 | keyword_path_rate | count(fast-path resolved) / total | report only | — |
| CAE-M-023 | intent_type_distribution | count by intent type | report only | — |

---

### M-GROUP-7: Feedback Signals (CAE-FB)

| ID | Name | Formula | Target | Alert |
|----|------|---------|--------|-------|
| CAE-M-024 | helpfulness_score_mean | mean(pack_helpfulness_score) | ≥ 0.75 | < 0.65 |
| CAE-M-025 | context_gap_rate | count(re-query within 60s) / total | < 15% | > 25% |
| CAE-M-026 | abandonment_rate | count(ABANDONED signal) / total | < 20% | > 40% |
| CAE-M-027 | explicit_feedback_rate | count(explicit feedback received) / total | ≥ 5% | < 1% |

---

### M-GROUP-8: Pack Output Distribution (CAE-DIST)

| ID | Name | Formula | Target | Alert |
|----|------|---------|--------|-------|
| CAE-M-028 | pack_type_distribution | count by pack type | report only | — |
| CAE-M-029 | compression_level_distribution | count by primary compression level | report only | — |
| CAE-M-030 | token_utilization_histogram | histogram of total_tokens / budget | report only | — |

---

## Metric Collection Schedule

| Group | Collection Frequency | Retention |
|-------|---------------------|-----------|
| Latency (M-001–004) | Real-time (per assembly) | 90 days raw; 1 year aggregated |
| Volume (M-005–007) | 1-minute windows | 90 days |
| Cache (M-008–011) | 5-minute windows | 30 days |
| Quality (M-012–015) | 5-minute windows | 90 days |
| Safety (M-016–019) | 5-minute windows | 365 days |
| Intent (M-020–023) | 1-hour windows | 90 days |
| Feedback (M-024–027) | 6-hour windows | 365 days |
| Distribution (M-028–030) | Daily | 90 days |

---

## Alert Routing

```
CRITICAL alerts (immediate, page on-call):
  CAE-M-003 > 120ms (SLO breach)
  CAE-M-006 > 3%   (high failure rate)
  CAE-M-018 > 3%   (high hallucination rate)

WARNING alerts (Slack notification):
  CAE-M-002 > 100ms
  CAE-M-009 < 70%
  CAE-M-016 < 0.70

INFORMATIONAL (dashboard only):
  All distribution metrics (M-028–030)
```

---

## Rules

| Rule | Statement |
|------|-----------|
| CAE-106 | All 30 metrics must be collected — missing metrics constitute a monitoring gap |
| CAE-107 | CAE-M-003 is the primary SLO metric — breach triggers degradation protocol |
| CAE-108 | CAE-M-018 (hallucination rate) must be reviewed monthly — above 1% triggers KIL audit |
| CAE-109 | Distribution metrics (M-028–030) are report-only — no alert thresholds |
| CAE-110 | Metric collection must add < 2ms to E2E assembly latency |

---

## Cross-References

- Performance model → `21-PERFORMANCE-MODEL`
- Feedback model → `22-FEEDBACK-MODEL`
- CAE certification → `26-CAE-CERTIFICATION`
- KIL metrics → Phase 3.0D.0.6 `26-KNOWLEDGE-METRICS`
