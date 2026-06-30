# KNW-KE-ARCH-013 — Knowledge Benchmarks

**Phase:** 3.0C.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Canonical benchmark objects for all 19 platform algorithms. Each benchmark specifies the performance contract that CI must enforce.

---

## Benchmark Object Index

All 19 algorithm benchmarks live in `knowledge/benchmarks/`:

| Benchmark ID | Algorithm | Operation | P50 | P99 |
|-------------|-----------|-----------|-----|-----|
| KNW-TEST-BENCH-001 | A-01 ID Generation | `generate_id("PLT", "MOD")` | < 0.1ms | < 1ms |
| KNW-TEST-BENCH-002 | A-02 Checksum | `compute_checksum(1KB object)` | < 1ms | < 5ms |
| KNW-TEST-BENCH-003 | A-03 Fingerprint | `compute_fingerprint(ns, type, cs)` | < 0.1ms | < 1ms |
| KNW-TEST-BENCH-004 | A-04 Quality Score | `compute_quality(object, all dims)` | < 5ms | < 20ms |
| KNW-TEST-BENCH-005 | A-05 Evidence Freshness | `freshness(record, now)` | < 0.1ms | < 0.5ms |
| KNW-TEST-BENCH-006 | A-06 Confidence Decay | `decayed_confidence(obj, now)` | < 0.5ms | < 2ms |
| KNW-TEST-BENCH-007 | A-07 BM25 | `bm25_score(5 terms, 1M docs)` | < 50ms | < 100ms |
| KNW-TEST-BENCH-008 | A-08 Hybrid Score | `hybrid_score(t, s, q)` | < 0.01ms | < 0.1ms |
| KNW-TEST-BENCH-009 | A-09 Traceability Coverage | `coverage(100 reqs, 1M objects)` | < 50ms | < 200ms |
| KNW-TEST-BENCH-010 | A-10 Path Confidence | `path_confidence(10-hop path)` | < 1ms | < 5ms |
| KNW-TEST-BENCH-011 | A-11 Cosine Similarity | `cosine_similarity(768-dim vecs)` | < 0.1ms | < 0.5ms |
| KNW-TEST-BENCH-012 | A-12 Bulk Validation | `bulk_validate(1000 objects)` | < 500ms | < 2s |
| KNW-TEST-BENCH-013 | G-01 DFS | `dfs(1M node graph, depth 10)` | < 50ms | < 200ms |
| KNW-TEST-BENCH-014 | G-02 BFS | `bfs(1M node graph, depth 5)` | < 20ms | < 100ms |
| KNW-TEST-BENCH-015 | G-03 Tarjan SCC | `scc(1M node graph)` | < 1s | < 5s |
| KNW-TEST-BENCH-016 | G-04 Topological Sort | `topo_sort(1M node DAG)` | < 2s | < 10s |
| KNW-TEST-BENCH-017 | G-05 Dijkstra | `dijkstra(1M nodes, single source)` | < 500ms | < 2s |
| KNW-TEST-BENCH-018 | G-06 Dependency Closure | `dep_closure(avg 50 nodes)` | < 20ms | < 100ms |
| KNW-TEST-BENCH-019 | G-07 Graph Diff | `graph_diff(1M node graphs)` | < 30s | < 120s |

---

## Canonical Benchmark Object Format

```yaml
# knowledge/benchmarks/bench-001-id-generation.yaml
knowledge_id: KNW-TEST-BENCH-001
object_type: benchmark
name: A-01 ID Generation Benchmark
version: 1.0.0
status: CANONICAL
owner: team:quality
description: Measures latency of generate_id() at 10,000 sequential calls.
tags: [test, benchmark, identity, performance]

classification:
  domain: TEST
  layer: 0
  category: PERFORMANCE

identity:
  canonical_name: test.benchmark.a-01-id-generation-benchmark
  knowledge_uri: knw://test/benchmark/a-01-id-generation-benchmark@1.0.0

evidence:
  items:
    - evidence_type: EV-DOC
      source_url: "architecture/docs/knowledge-core/37-PERFORMANCE-BUDGET"
      description: "P50 < 0.1ms, P99 < 1ms specified in performance budget"
      weight: 0.60
  evidence_score: 0.60

confidence:
  declared: 0.90

traceability:
  satisfies: []
  implements: []
  tests: []

metadata:
  created_by: team:quality
  created_at: "2026-06-30T00:00:00Z"
  source_doc: "architecture/docs/knowledge-core/37-PERFORMANCE-BUDGET"

benchmark_id: BENCH-001
operation: "identity.generate_id('PLT', 'MOD') × 10000"
p50_ms: 0.1
p99_ms: 1.0
```

---

## Benchmark Execution

```bash
kos benchmark run                     # run all 19 benchmarks
kos benchmark run --id BENCH-001      # run single benchmark
kos benchmark run --category graph    # run all graph algorithm benchmarks
kos benchmark report                  # generate report
kos benchmark compare {old} {new}     # compare two runs
```

**Benchmark runner requirements:**
- Must run against 1M object dataset (`performance/1m/`)
- Must report P50, P90, P95, P99
- Must fail if P99 > budget
- Must report memory usage (resident set size)
- Results written to `knowledge/reports/benchmarks/{date}.yaml`

---

## Benchmark Report Format

```yaml
# knowledge/reports/benchmarks/2026-06-30.yaml
run_date: "2026-06-30T00:00:00Z"
dataset_size: 1000000
results:
  - benchmark_id: BENCH-001
    operation: "identity.generate_id('PLT', 'MOD') × 10000"
    iterations: 10000
    p50_ms: 0.08
    p90_ms: 0.12
    p99_ms: 0.40
    budget_p50_ms: 0.1
    budget_p99_ms: 1.0
    p50_status: PASS
    p99_status: PASS
    memory_mb: 0.1
```

---

## Benchmark Failure Protocol

If P99 > budget:
1. CI fails with exit code 2
2. Engineer investigates regression
3. If budget must change: ADR required → update `37-PERFORMANCE-BUDGET` → update benchmark object
4. If implementation is slow: fix implementation in Phase 3.0D

No budget increases without ADR.

---

## Cross-References

- Performance budgets → Phase 3.0C `37-PERFORMANCE-BUDGET`
- Algorithm definitions → Phase 3.0C `36-ALGORITHMS`
- Graph algorithm definitions → Phase 3.0C `25-GRAPH-ALGORITHMS`
- CI runs benchmarks on release → `08-KNOWLEDGE-CI`
- Golden dataset powers performance tests → `12-KNOWLEDGE-GOLDEN-DATASET`
