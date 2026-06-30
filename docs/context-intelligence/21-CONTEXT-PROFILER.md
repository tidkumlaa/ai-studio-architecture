# UICE-DOC-021 — Context Profiler

**Phase:** 3.0D.2
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Context Profiler records the complete timing, token, and field-usage
breakdown for every context assembly. Profiling enables UICE's explainability
requirement: for any assembled pack, it must be possible to answer:
- Why was each field included?
- Why was each object selected?
- Where did the 60ms P99 budget go?
- Which stage was the bottleneck?

---

## ProfileReport Schema

```yaml
ProfileReport:
  profile_id: string
  pack_id: string
  session_id: string | null
  timestamp: datetime

  # Stage timing (milliseconds)
  stage_timing:
    intent_analysis_ms: float
    query_classification_ms: float
    cache_lookup_ms: float          # Semantic + L1 + L2 combined
    graph_traversal_ms: float
    object_ranking_ms: float
    context_planning_ms: float
    budget_allocation_ms: float
    compression_ms: float           # FLA + DELTA + SUMMARY combined
    sco_ms: float
    deduplication_ms: float
    guard_generation_ms: float
    quality_optimization_ms: float
    model_adaptation_ms: float
    verification_ms: float
    total_ms: float

  # Token breakdown
  token_breakdown:
    primary_tokens: int
    guard_tokens: int
    context_slot_tokens: dict[str, int]  # knowledge_id → tokens
    sco_preamble_tokens: int
    total_tokens: int
    baseline_tokens: int             # L4 equivalent without UICE
    token_reduction_pct: float

  # Field inclusion audit (explainability)
  field_inclusion_log: list[FieldInclusionEntry]
    FieldInclusionEntry:
      knowledge_id: string
      field_path: string
      included: bool
      reason: string                 # "FLA-matrix", "L3-floor", "SCO-delta", "dedup-removed"
      token_contribution: int

  # Object selection audit
  object_selection_log: list[ObjectSelectionEntry]
    ObjectSelectionEntry:
      knowledge_id: string
      included: bool
      assembly_mode: FLA | DELTA | SUMMARY | FULL | OMIT
      relevance_score: float
      relevance_breakdown: dict[str, float]
      reason_excluded: string | null   # if included=False

  # Cache performance
  cache_profile:
    semantic_hit: bool
    l1_hit: bool
    l2_hit: bool
    reuse_hit: bool
    hit_tier: SEMANTIC | L1 | L2 | REUSE | MISS

  # Quality profile
  quality_profile:
    quality_score: float
    quality_breakdown: dict[str, float]   # dimension → score
    passed: bool

  # Anomalies
  anomalies: list[string]          # any invariant warnings or unusual conditions
```

---

## Profiling Data Collection

```
Profiling is active on EVERY assembly (not sampling):
  - Stage timing: monotonic clock wrap around each stage
  - Token breakdown: estimate_tokens() called after each assembly step
  - Field inclusion: logged during FIELD_EXTRACT and DEDUPLICATE
  - Object selection: logged during RANK_OBJECTS and PLAN_CONTEXT

Storage:
  - In-memory ring buffer: last 10,000 profiles
  - Export: on demand (API) or on P99 breach (alert trigger)
  - Retention: profiles older than 1 hour purged from buffer
```

---

## Profiler API

```
GET /profiles/{pack_id}
  Returns: ProfileReport for a specific pack

GET /profiles?intent_type={intent}&query_type={qtype}&since={timestamp}
  Returns: list[ProfileReport] filtered by intent+query+time window

GET /profiles/stage-stats?window=1h
  Returns: per-stage P50/P95/P99 latency for last 1 hour

GET /profiles/field-stats?intent_type={intent}&query_type={qtype}
  Returns: field inclusion rate per field path for given (intent, query_type)
           (used by Context Learning to detect unused fields)
```

---

## Bottleneck Detection

```
DETECT_BOTTLENECK(profile) → list[BottleneckWarning]:
  warnings = []

  if profile.stage_timing.graph_traversal_ms > 20:
    warnings.append("Graph traversal exceeded 20ms — check traversal depth or index")

  if profile.stage_timing.cache_lookup_ms > 10:
    warnings.append("Cache lookup exceeded 10ms — check L1/L2 connectivity")

  if profile.stage_timing.total_ms > 60:
    warnings.append(f"Total exceeded P99 target (60ms) — worst stage: {WORST_STAGE(profile)}")

  if profile.token_breakdown.primary_tokens > 200 and intent_type != CONTEXT_PRODUCTION:
    warnings.append("Primary exceeded 200 tokens for non-full-context request — check FLA matrix")

  return warnings
```

---

## Field Usage Analysis (for Learning)

```
The profiler's field_inclusion_log feeds the Learning module with data on
which fields are actually used vs included:

  INCLUDED but NOT LLM_REFERENCED → candidate for demotion
  LLM_REFERENCED but NOT INCLUDED → candidate for promotion

This analysis is run weekly by the Context Learning cycle.
```

---

## Rules

| Rule | Statement |
|------|-----------|
| UICE-101 | Profiling is active for 100% of assemblies — no sampling; every pack has a ProfileReport |
| UICE-102 | field_inclusion_log must contain an entry for every field path that was considered during assembly |
| UICE-103 | Stage timing must use monotonic clock; wall-clock timestamps are not acceptable for latency measurement |
| UICE-104 | ProfileReport is retained for minimum 1 hour; expired profiles are purged (not archived) |
| UICE-105 | Profiling overhead must not exceed 2ms per pack; profiler budget is included in the 60ms P99 target |

---

## Cross-References

- Context Metrics → `20-CONTEXT-METRICS`
- Context Learning → `19-CONTEXT-LEARNING`
- Context Benchmark → `27-CONTEXT-BENCHMARK`
- Dashboard → `29-DASHBOARD`
