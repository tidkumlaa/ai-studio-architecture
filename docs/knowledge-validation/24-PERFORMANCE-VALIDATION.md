# KVF-DOC-024 — Performance Validation

**Phase:** 3.0D.1.5
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Performance Validation verifies that the KOS Runtime meets all latency, memory,
CPU, and disk benchmarks defined in the CAE Performance Model and extended here
for validation purposes. Performance failure blocks production deployment.

---

## Performance Dimensions

```
PV-DIM-1: Assembly Latency
  E2E latency of context assembly (S1–S9 pipeline)
  Source: CAE Performance Model (Phase 3.0D.1 doc 21)

PV-DIM-2: Search Latency
  Latency of search queries (from doc 08)
  Extended here with detailed component breakdown

PV-DIM-3: Index Operation Latency
  Latency of KIL index lookup, write, and invalidation

PV-DIM-4: Memory Usage
  RSS and heap usage under standard and peak load

PV-DIM-5: CPU Usage
  CPU utilization under standard and peak load

PV-DIM-6: Disk I/O
  Read/write patterns for index persistence and cache

PV-DIM-7: Resource Stability
  Memory growth rate under sustained load (leak detection)
```

---

## Assembly Latency Benchmarks

```
Load: 100 rps sustained

Latency targets (from CAE doc 21):
  P50: < 50ms
  P95: < 80ms
  P99: < 120ms

Component breakdown (P99 budgets):
  S1 Intent Parser:      < 5ms
  S2 Object Selector:    < 20ms
  S3 Relevance Ranker:   < 15ms
  S4 Budget Manager:     < 2ms
  S5 Compression:        < 2ms
  S6 Context Planner:    < 3ms
  S7 Context Assembler:  < 50ms
  S8 AH Guard:           < 5ms
  S9 Validator:          < 5ms
```

---

## Memory Usage Benchmarks

```
Test: Sustained 100 rps for 1 hour

Memory targets:
  Baseline RSS at startup:     < 500MB
  RSS under load (100 rps):    < 2GB
  RSS growth rate:             < 10MB/hour (leak threshold)
  Peak RSS (burst to 500 rps): < 4GB

Cache memory:
  L1 in-process cache:         < 500MB (1,000 entries × ~500KB)
  Total memory with cache:     < 2.5GB at steady state
```

---

## CPU Usage Benchmarks

```
Test: Sustained 100 rps for 1 hour, standard core count (4 cores)

CPU targets:
  Mean CPU at 100 rps:         < 50% (single core)
  Peak CPU (burst 500 rps):    < 90% (single core)
  S7 Assembler CPU share:      < 30% of total CAE CPU
  Embedding lookup CPU:        < 10% of total (precomputed vectors only)
```

---

## Index Operation Latency

```
KIL Index Lookup (by knowledge_id):
  P50: < 1ms
  P99: < 5ms
  P999: < 10ms

KIL Index Write (object update):
  P99: < 50ms
  (writes are less frequent than reads; higher tolerance)

Cache Invalidation (on object update):
  P99: < 100ms  (from update signal to all cache entries invalidated)
```

---

## Disk I/O Benchmarks

```
Index persistence:
  Full index snapshot write: < 30 seconds (10,000 objects)
  Incremental snapshot: < 1 second per batch (100 objects)
  Index reload on startup: < 10 seconds (10,000 objects)

Cache persistence (optional):
  L2 cache read: < 5ms P99 (network KV store)
  L2 cache write: < 10ms P99
```

---

## Resource Stability Test

```
Duration: 1 hour at 100 rps

Measurements every 5 minutes:
  RSS (resident set size)
  heap used
  P99 assembly latency
  cache hit rate

Memory leak threshold:
  Linear regression of RSS over time
  slope > 10MB/hour → POTENTIAL_LEAK alert
  slope > 50MB/hour → CONFIRMED_LEAK (critical failure)

Latency degradation threshold:
  P99 at t=60min must not exceed 1.5× P99 at t=5min
  (no gradual degradation under sustained load)
```

---

## Performance Validation Result Schema

```yaml
performance_validation_result:
  test_duration_minutes: integer
  target_rps: integer
  achieved_rps: float

  assembly_latency:
    p50_ms: float
    p95_ms: float
    p99_ms: float
    stage_breakdown: {S1: float, S2: float, S3: float, S4: float,
                      S5: float, S6: float, S7: float, S8: float, S9: float}

  memory:
    baseline_rss_mb: float
    peak_rss_mb: float
    rss_growth_rate_mb_per_hour: float
    leak_detected: boolean

  cpu:
    mean_cpu_pct: float
    peak_cpu_pct: float

  index_latency:
    lookup_p99_ms: float
    write_p99_ms: float
    invalidation_p99_ms: float

  disk:
    index_load_time_sec: float
    snapshot_write_time_sec: float

  resource_stability:
    latency_degradation_ratio: float    # p99 at t=60 / p99 at t=5
    memory_stable: boolean
    latency_stable: boolean
```

---

## Rules

| Rule | Statement |
|------|-----------|
| KVF-116 | P99 assembly latency > 120ms under 100 rps is a CRITICAL failure |
| KVF-117 | Memory leak confirmed (>50MB/hour growth) blocks all certification above BRONZE |
| KVF-118 | CPU > 90% at 100 rps means the implementation cannot handle burst load |
| KVF-119 | Component latency breakdown must be measured — total P99 alone is insufficient |
| KVF-120 | Resource stability test must run for at least 1 hour — 10-minute tests miss slow leaks |

---

## Cross-References

- CAE performance model → Phase 3.0D.1 `21-PERFORMANCE-MODEL`
- Scalability validation → `25-SCALABILITY-VALIDATION`
- Cold-start benchmark → `22-COLD-START-BENCHMARK`
- Regression validation → `26-REGRESSION-VALIDATION`
