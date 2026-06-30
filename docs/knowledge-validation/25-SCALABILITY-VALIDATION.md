# KVF-DOC-025 — Scalability Validation

**Phase:** 3.0D.1.5
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Scalability Validation benchmarks the KOS Runtime at four corpus sizes:
1K / 10K / 100K / 1M KnowledgeObjects. It verifies that latency, memory,
and search quality degrade gracefully rather than catastrophically as the
knowledge corpus grows.

---

## Scale Tiers

| Tier | Objects | Target Use Case |
|------|---------|----------------|
| SMALL | 1,000 | Single team / small product |
| MEDIUM | 10,000 | Mid-size product / department |
| LARGE | 100,000 | Enterprise / platform-level |
| XLARGE | 1,000,000 | Research / AI training corpus |

---

## Scalability Invariants

```
The following properties must hold at ALL scale tiers:

SI-01: Assembly latency invariant
  P99 assembly latency must not exceed 200ms even at 1M objects
  (Relaxed from standard 120ms — 1M objects requires larger index)

SI-02: Search quality invariant
  Top-1 accuracy must remain >= 0.80 at all scales
  (Larger corpus must not dilute search quality)

SI-03: Budget invariant
  Token budget enforcement is O(1) — scale-independent
  total_tokens <= budget in 100% of packs at all scales

SI-04: Guard block invariant
  Guard block generation is O(1) — scale-independent
  AH Guard always runs regardless of corpus size

SI-05: Memory sub-linear invariant
  Memory must grow sub-linearly with corpus size
  Expected: O(N × log N) or O(N) — never O(N²)
```

---

## Latency by Scale

```
Assembly P99 Latency Targets:

  1K objects:    < 50ms   (hot index, high cache hit rate)
  10K objects:   < 80ms   (standard target)
  100K objects:  < 120ms  (index becomes larger, cache miss rate increases)
  1M objects:    < 200ms  (relaxed; index lookup and traversal slower)

Key drivers of latency at scale:
  - Index lookup: O(log N) for sorted index
  - Graph traversal: bounded by max_hops=3, max_candidates=50 → O(1)
  - Cache miss: cold read from large index is slower
  - Relevance ranking: bounded by max_candidates=50 → O(1)
```

---

## Memory by Scale

```
Memory Targets:

  1K objects:    < 200MB
  10K objects:   < 2GB
  100K objects:  < 20GB  (requires distributed setup)
  1M objects:    < 200GB (requires distributed setup)

Memory growth model:
  Per object in index: ~20KB (index entry + compression levels)
  1K:   ~20MB (in-memory)
  10K:  ~200MB (in-memory)
  100K: ~2GB (may require partial loading)
  1M:   ~20GB (requires sharded index)

Cache overhead (in addition to index):
  L1 in-process cache: 1,000 entries × 500 tokens × 4 bytes ≈ 2MB
  L2 shared cache: 10,000 entries × 500 tokens × 4 bytes ≈ 20MB
  Cache size does NOT scale with corpus (LRU eviction)
```

---

## Search Quality by Scale

```
Search quality must not degrade with corpus size:

Top-1 Accuracy:
  1K:   >= 0.90
  10K:  >= 0.87
  100K: >= 0.85
  1M:   >= 0.80

MRR:
  1K:   >= 0.90
  10K:  >= 0.87
  100K: >= 0.85
  1M:   >= 0.80

Rationale: larger corpus means more irrelevant objects competing in rankings.
BM25 and vector search naturally handle this if index is properly configured.
```

---

## Scalability Test Protocol

```
For each scale tier:
  1. Load corpus with N synthetic objects (using KOS schema)
  2. Warm cache with 1,000 queries
  3. Run 500 queries measuring P50/P95/P99 latency
  4. Measure memory (RSS, heap)
  5. Run search quality test (100 queries from golden dataset)
  6. Run 1-hour stability test (50 rps at scale)
```

---

## Scalability Result Schema

```yaml
scalability_result:
  tiers_tested: [string]

  by_tier:
    - tier: string
      corpus_size: integer

      latency:
        p50_ms: float
        p95_ms: float
        p99_ms: float
        si_01_passed: boolean

      memory:
        peak_rss_mb: float
        index_memory_mb: float

      search_quality:
        top1_accuracy: float
        mrr: float
        si_02_passed: boolean

      budget_enforcement_rate: float    # SI-03: must be 1.00

  sub_linear_memory_growth:
    1k_to_10k_ratio: float             # should be ~10x, not 100x
    10k_to_100k_ratio: float
    growth_model: string               # O(N), O(N log N), O(N^2), etc.
```

---

## Rules

| Rule | Statement |
|------|-----------|
| KVF-121 | SI-01 (P99 < 200ms at 1M objects) is the maximum relaxation — 200ms is not negotiable |
| KVF-122 | SI-02 (Top-1 >= 0.80 at 1M) is the search quality floor — larger corpus ≠ worse search |
| KVF-123 | SI-05 (sub-linear memory) must be demonstrated with data, not assumed |
| KVF-124 | 100K and 1M tiers may require distributed deployment — the spec does not prohibit this |
| KVF-125 | Scalability tests must use realistic object distributions, not uniform synthetic data |

---

## Cross-References

- Performance validation → `24-PERFORMANCE-VALIDATION`
- CAE performance model → Phase 3.0D.1 `21-PERFORMANCE-MODEL`
- Context cache → Phase 3.0D.1 `20-CONTEXT-CACHE`
- KIL canonical index → Phase 3.0D.0.6 `29-KNOWLEDGE-CANONICAL-INDEX`
