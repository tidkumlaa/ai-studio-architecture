# KVF-DOC-022 — Cold-Start Benchmark

**Phase:** 3.0D.1.5
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Cold-Start Benchmark measures KOS Runtime performance when starting from
zero — no warm cache, no prefetch queue, no hot index. This is the worst-case
scenario that happens at service startup, after a deployment, or in a new
environment.

Cold-start performance reveals the true floor of the system before any
caching optimization takes effect.

---

## Cold-Start Definition

```
Cold start conditions:
  - Cache tier 1 (in-process): empty
  - Cache tier 2 (shared): empty
  - Prefetch queue: empty
  - Index: loaded into memory (index does not reset)

Cold start is NOT:
  - Starting with an empty knowledge index
  - Starting from a fresh database (index remains)
```

---

## Cold-Start Scenarios

```
CS-01: Single Object First Fetch
  Query: first-ever query for KNW-PLT-MOD-001
  Budget: STANDARD (1,000 tokens)
  Expected: all objects fetched from index (0% cache hit)
  Latency target: < 500ms P99

CS-02: Repeated Object (Cache Warm-Up)
  Same query 10 times sequentially after CS-01
  Expected: latency drops from ~500ms to ~50ms by run 3
  Warm-up curve: documented per run

CS-03: Parallel Cold Start
  10 concurrent queries, all for different objects (all cache misses)
  Budget: STANDARD
  Latency target: < 800ms P99 under concurrent cold load

CS-04: Post-Deployment Cold Start
  Simulate service restart: flush all caches, run 100 queries
  Measure time until cache reaches 85% hit rate
  Target: < 5 minutes from cold start to warm state

CS-05: Cold Start with Prefetch
  Start cold, but with prefetch enabled
  After 50 queries, measure cache hit rate
  Expected: prefetch reduces cold-start recovery time
```

---

## Cold-Start Metrics

```
cs_first_query_latency_ms:
  The latency of the very first query from cold state
  Target: < 500ms P99

cs_warmup_curve:
  Latency by query number (1 through 50):
  [q1_p99, q2_p99, ..., q50_p99]
  Expected: latency falls to < 200ms by query 5
            latency falls to < 80ms by query 20

cs_cache_warmup_time_sec:
  Seconds from cold start until cache hit rate >= 85%
  Target: < 300 seconds (5 minutes) under normal query load

cs_cold_error_rate:
  Proportion of cold-start queries that fail (timeout, OOM, etc.)
  Target: 0%

cs_index_load_time_ms:
  Time to load the KIL index into memory on startup
  Target: < 10,000ms (10 seconds) for corpus up to 10,000 objects
```

---

## Cache Warm-Up Rate

```
After cold start, tracking cache hit rate every 10 seconds:

  t=0s   (cold):  0% hit rate
  t=10s:          5–20% hit rate
  t=60s:          30–50% hit rate
  t=180s:         60–75% hit rate
  t=300s:         ≥ 85% hit rate  (warm state achieved)

Expected curve shape: logarithmic (fast improvement early, slows as hits fill cache)
```

---

## Cold-Start Result Schema

```yaml
cold_start_result:
  scenarios_run: [string]

  first_query_latency_p99_ms: float
  first_query_latency_p50_ms: float

  warmup_curve:
    - query_number: integer
      p99_latency_ms: float
      cache_hit_rate: float

  cache_warmup_time_sec: float
  warm_state_hit_rate: float            # hit rate when warm_state achieved

  cold_error_rate: float
  index_load_time_ms: float

  with_prefetch_vs_without:
    warmup_time_with_prefetch_sec: float
    warmup_time_without_prefetch_sec: float
    prefetch_speedup_ratio: float
```

---

## Rules

| Rule | Statement |
|------|-----------|
| KVF-106 | Cold-start test must flush BOTH cache tiers — L1 and L2 — before each run |
| KVF-107 | Index must remain loaded during cold-start — cold cache ≠ cold index |
| KVF-108 | cs_cold_error_rate must be 0% — no queries may fail due to cold-start conditions |
| KVF-109 | cs_cache_warmup_time_sec must be measured under realistic query load (not idle) |
| KVF-110 | CS-04 (post-deployment) must simulate real deployment: flush + 5-minute soak test |

---

## Cross-References

- Context cache spec → Phase 3.0D.1 `20-CONTEXT-CACHE`
- Prefetch engine spec → Phase 3.0D.1 `19-PREFETCH-ENGINE`
- Performance model → Phase 3.0D.1 `21-PERFORMANCE-MODEL`
- Performance validation → `24-PERFORMANCE-VALIDATION`
