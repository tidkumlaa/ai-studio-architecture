# UICE-DOC-027 — Context Benchmark

**Phase:** 3.0D.2
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Context Benchmark defines the measurement protocol for comparing UICE
against the CAE baseline. The benchmark is the evidence that UICE actually
meets its success targets — token reduction, latency, quality, and hallucination.

Four scales: 100 / 1,000 / 10,000 / 50,000 queries.
The primary certification gate requires the 10,000-query scale.

---

## Benchmark Scales

```
SMOKE (100 queries):
  Purpose: rapid sanity check during development
  Required for: any code change
  Duration: < 5 minutes
  Coverage: 1 query per (intent, query_type) combination (120 combinations sampled)

STANDARD (1,000 queries):
  Purpose: pre-release validation
  Required for: UICE-STANDARD certification
  Duration: < 30 minutes
  Coverage: proportional distribution across all intent×query combinations

LARGE (10,000 queries):
  Purpose: primary certification gate
  Required for: UICE-ADVANCED and UICE-RESEARCH certification
  Duration: < 3 hours
  Coverage: full distribution + stress conditions (cold cache, concurrent, large packs)

FULL (50,000 queries):
  Purpose: comprehensive research benchmark
  Required for: UICE-RESEARCH certification only
  Duration: < 24 hours
  Coverage: all conditions including adversarial queries
```

---

## Benchmark Metrics (8)

```
1. TOKEN_REDUCTION:
   Measurement: (baseline_tokens - uice_tokens) / baseline_tokens
   Baseline: CAE L4 Standard (500 tokens/primary, 100 tokens/guard)
   UICE target: ≥ 99%

2. COMPILATION_LATENCY:
   Measurement: wall-clock time from request receipt to pack emit (P99)
   UICE target: < 60ms
   CAE baseline: < 120ms

3. QUALITY_SCORE:
   Measurement: mean quality_score across all benchmark packs
   UICE target: ≥ 0.85
   CAE baseline: ≥ 0.80

4. HALLUCINATION_RATE:
   Measurement: fraction of packs where LLM response contained hallucination
   Types: Type A (wrong fact) and Type B (wrong dependency) as primary
   UICE target: > 30% reduction vs CAE baseline
   Method: human annotation of 100 random LLM responses per scale

5. CACHE_HIT_RATE:
   Measurement: fraction of requests served from any cache tier
   UICE target: ≥ 85% at steady state (warm cache)

6. CONVERSATION_SAVINGS:
   Measurement: token reduction in queries 2–N within same session
   UICE target: > 90% savings vs first query
   Method: 10-query sessions × 50 objects

7. FIELD_PRECISION:
   Measurement: fields included that were referenced in LLM response / total included
   UICE target: > 99%
   Method: automated LLM response analysis

8. DEDUP_EFFECTIVENESS:
   Measurement: duplicate tokens eliminated / total duplicate tokens detected
   UICE target: 100% for TYPE-2 (exact object); ≥ 95% for TYPE-1 (exact field)
```

---

## Benchmark Query Corpus

```
Query corpus for STANDARD (1,000 queries):
  Distribution:
    FACTUAL_LOOKUP:       200 queries (20%)
    CAPABILITY_DISCOVERY: 100 queries (10%)
    REASONING_QUERY:      100 queries (10%)
    IMPACT_ANALYSIS:      100 queries (10%)
    COMPARISON:            80 queries  (8%)
    SEARCH:                80 queries  (8%)
    DIAGNOSTIC:           100 queries (10%)
    CONTEXT_PRODUCTION:    50 queries  (5%)
    TRACEABILITY:          80 queries  (8%)
    MIGRATION_GUIDE:       40 queries  (4%)
    ARCHITECTURE_REVIEW:   30 queries  (3%)
    CODE_GENERATION:       40 queries  (4%)

  Object corpus: 1,000 diverse KnowledgeObjects
  Intent distribution: reflect production traffic
```

---

## BenchmarkReport Schema

```yaml
BenchmarkReport:
  report_id: string
  benchmark_scale: SMOKE | STANDARD | LARGE | FULL
  run_started_at: datetime
  run_completed_at: datetime
  uice_version: string
  cae_baseline_version: string

  # 8 primary metrics vs CAE baseline
  metrics:
    token_reduction_pct:       {uice: float, cae: float, delta: float, target: float, passed: bool}
    compilation_p99_ms:        {uice: float, cae: float, delta: float, target: float, passed: bool}
    quality_score_mean:        {uice: float, cae: float, delta: float, target: float, passed: bool}
    hallucination_reduction_pct: {uice: float, cae: float, delta: float, target: float, passed: bool}
    cache_hit_rate:            {uice: float, cae: float, delta: float, target: float, passed: bool}
    conversation_savings_pct:  {uice: float, cae: float, delta: float, target: float, passed: bool}
    field_precision_pct:       {uice: float, cae: float, delta: float, target: float, passed: bool}
    dedup_effectiveness_pct:   {uice: float, cae: float, delta: float, target: float, passed: bool}

  # Pass/fail summary
  metrics_passed: int     # of 8
  benchmark_passed: bool  # all primary metrics must pass

  # Breakdown by intent type
  by_intent: dict[IntentType, MetricSet]
```

---

## Rules

| Rule | Statement |
|------|-----------|
| UICE-131 | SMOKE scale (100 queries) must be run on every code change before merge |
| UICE-132 | STANDARD scale (1,000 queries) must be run before every release |
| UICE-133 | All 8 benchmark metrics must be measured and reported; partial benchmarks are INVALID |
| UICE-134 | Hallucination measurement requires human annotation; automated proxies are supplementary only |
| UICE-135 | benchmark_passed = True requires ALL 8 primary metrics to meet their targets |

---

## Cross-References

- Context Metrics → `20-CONTEXT-METRICS`
- Context Certification → `28-CONTEXT-CERTIFICATION`
- KVF benchmark protocol → Phase 3.0D.1.5 `27-CONTEXT-BENCHMARK`
- CAE baseline → Phase 3.0D.1 `27-CAE-TESTING`
