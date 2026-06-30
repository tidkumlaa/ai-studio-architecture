# UICE-DOC-024 — Latency Optimizer

**Phase:** 3.0D.2
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Latency Optimizer ensures that UICE meets its P99 compilation target of
< 60ms. It identifies latency bottlenecks in the assembly pipeline and applies
optimization strategies: parallel execution, cache-first routing, prefetch
scheduling, and depth reduction for graph traversal.

The latency optimizer operates at plan time (before assembly) and at assembly
time (concurrent stage execution where dependencies allow).

---

## UICE Latency Budget (60ms P99)

```
Stage budgets (sum to 60ms):
  Intent analysis + Query classification:   3ms
  Cache lookups (Semantic + L1 + L2):       8ms
  Graph traversal:                         20ms  ← largest budget
  Object ranking:                           3ms
  Context planning + Budget allocation:     2ms
  Compression (FLA + DELTA + SUMMARY):      5ms
  SCO + Deduplication:                      3ms
  Guard generation:                         3ms
  Quality + Verification:                   4ms
  Model adaptation:                         2ms
  Profiling overhead:                       2ms
  Slack:                                    5ms
  ─────────────────────────────────────────────
  TOTAL P99:                               60ms
```

---

## Parallel Execution Map

```
UICE stages that can run in parallel:

Group A (all independent, run together):
  Intent Analyzer
  Query Classifier
  Semantic Cache lookup   ← early start, doesn't need intent result
                            (uses raw query text)

Group B (after Group A):
  L1/L2 Cache lookup     ← needs intent_type, query_type
  Conversation Memory filter  ← needs session_id (from request, not intent)
  Graph Traversal        ← needs intent_profile, query_profile

Group C (after Graph Traversal):
  Object Ranking         ← needs traversal results
  Reuse Engine lookup    ← parallel with ranking

Group D (after Planning):
  FLA per object         ← all objects in parallel
  SCO                    ← after FLA
  Deduplication          ← after SCO

Group E (final, sequential):
  Guard Generation       ← after compression (needs primary object)
  Quality Optimization
  Context Verification
  Model Adaptation
```

---

## LatencyOptimizationPlan Schema

```yaml
LatencyOptimizationPlan:
  estimated_total_ms: float
  meets_target: bool               # estimated_total_ms < 60ms

  optimizations_applied: list[LatencyOptimization]
    LatencyOptimization:
      optimization_type: PARALLEL | PREFETCH | CACHE_FIRST | DEPTH_REDUCTION | SKIP_TRAVERSAL
      rationale: string
      estimated_savings_ms: float

  parallel_groups: list[ParallelGroup]
    ParallelGroup:
      group_id: string
      stages: list[string]
      can_start_after: string | null   # dependency (null = immediate)
      estimated_wall_ms: float         # max of individual stages

  prefetch_hints: list[PrefetchHint]   # objects to prefetch before request
    PrefetchHint:
      knowledge_id: string
      prediction_confidence: float
      trigger: FREQUENT_CO_OCCURRENCE | HIGH_REUSE | EXPLICIT_PREFETCH
```

---

## Latency Optimization Strategies

```
CACHE_FIRST (always active):
  Check Semantic Cache → L1 → L2 → Reuse BEFORE graph traversal.
  Expected latency: 8ms on hit (vs 28ms for traversal + FLA).
  Hit rate contribution: 85% of latency budget eliminated on cache hit.

PARALLEL (always active):
  Execute Group A stages in parallel (see Parallel Execution Map).
  Expected saving: 5–8ms vs sequential execution.

SKIP_TRAVERSAL (intent-triggered):
  For FACTUAL_LOOKUP + IDENTITY: skip graph traversal entirely.
  depth = 0 → no traversal → save 20ms.

DEPTH_REDUCTION (budget-triggered):
  When estimated latency > 50ms: reduce traversal depth by 1.
  TRACEABILITY depth 4 → 3: save 8ms; IMPACT depth 2 → 1: save 10ms.

PREFETCH (async, for frequent objects):
  When Reuse Engine identifies objects that appear in > 30% of recent packs:
  Pre-assemble FLA slices before the next request arrives.
  Expected latency: 0ms (already assembled).
```

---

## LATENCY_OPTIMIZE Algorithm

```
OPTIMIZE_FOR_LATENCY(context_plan, constraints) → LatencyOptimizationPlan:

  if constraints.max_latency_ms is None:
    return DEFAULT_PLAN()  // no latency constraint — standard parallel execution

  // Estimate current latency
  estimated = ESTIMATE_PIPELINE_LATENCY(context_plan)

  optimizations = []

  // Always: cache-first (no cost)
  optimizations.append(CACHE_FIRST_OPTIMIZATION())

  // Always: parallel group execution
  optimizations.append(PARALLEL_EXECUTION_OPTIMIZATION())

  if estimated.total_ms > constraints.max_latency_ms:
    savings_needed = estimated.total_ms - constraints.max_latency_ms

    // Check SKIP_TRAVERSAL
    if context_plan.primary.assembly_mode != FULL and context_plan.context_slots == []:
      optimizations.append(SKIP_TRAVERSAL(saving=20))
      savings_needed -= 20

    // Check DEPTH_REDUCTION
    if savings_needed > 0 and context_plan.traversal_depth > 1:
      new_depth = max(1, context_plan.traversal_depth - 1)
      saving = ESTIMATE_DEPTH_SAVING(context_plan.traversal_depth, new_depth)
      optimizations.append(DEPTH_REDUCTION(new_depth=new_depth, saving=saving))
      savings_needed -= saving

  return LatencyOptimizationPlan(
    estimated_total_ms   = estimated.total_ms - SUM_SAVINGS(optimizations),
    meets_target         = estimated.total_ms - SUM_SAVINGS(optimizations) <= constraints.max_latency_ms,
    optimizations_applied = optimizations,
    parallel_groups      = BUILD_PARALLEL_GROUPS(context_plan),
    prefetch_hints       = REUSE_ENGINE.get_prefetch_hints(context_plan.primary_id),
  )
```

---

## Prefetch Protocol

```
PREFETCH SCHEDULE (async, runs between requests):

  After each request completes:
    analysis = ANALYZE_REQUEST_PATTERN(last_100_requests)
    for pattern in analysis.frequent_co_occurrences:
      if pattern.frequency > 0.30:
        for (intent, query) in pattern.likely_next_intents:
          SCHEDULE_PREFETCH(knowledge_id=pattern.target_id, intent=intent, query=query)

  Prefetch execution:
    FLA_MODULE.select(obj, intent, query) → stored in L1 cache
    Priority: LOW (does not compete with live requests)
```

---

## Rules

| Rule | Statement |
|------|-----------|
| UICE-116 | P99 compilation target is 60ms; breach triggers immediate alert (UICE-M-033) |
| UICE-117 | Parallel Group A (Intent + Classifier + Semantic Cache) always executes in parallel |
| UICE-118 | SKIP_TRAVERSAL is only applied when context_plan has no context slots and intent is point-lookup |
| UICE-119 | DEPTH_REDUCTION is a last resort; it reduces quality and must be logged in ProfileReport |
| UICE-120 | Prefetch execution runs at LOW priority and must not affect live request latency |

---

## Cross-References

- Context Metrics → `20-CONTEXT-METRICS` (UICE-M-033 through UICE-M-037)
- Semantic Cache → `15-SEMANTIC-CACHE` (cache-first)
- Context Graph Traversal → `08-CONTEXT-GRAPH-TRAVERSAL` (depth control)
- Context Reuse Engine → `17-CONTEXT-REUSE-ENGINE` (prefetch source)
- Cost Optimizer → `23-COST-OPTIMIZER`
