# CAE-DOC-021 — Performance Model

**Phase:** 3.0D.1
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Performance Model defines latency targets, capacity requirements, and
degradation behavior for the CAE pipeline. All numeric targets in this document
are architecture constraints — not aspirational goals.

---

## End-to-End Latency Target

```
P50: < 50ms
P95: < 80ms
P99: < 120ms

Measured from: query_received
Measured to:   first_byte_of_emitted_pack
```

---

## Per-Stage Latency Budget (P99)

| Stage | Name | P99 Budget |
|-------|------|-----------|
| S1 | Intent Parser | 5ms |
| S2 | Object Selector | 20ms |
| S3 | Relevance Ranker | 15ms |
| S4 | Budget Manager | 2ms |
| S5 | Compression Selector | 2ms |
| S6 | Context Planner | 3ms |
| S7 | Context Assembler | 50ms |
| S8 | AH Guard | 5ms |
| S9 | Context Validator | 5ms |
| **Total** | | **107ms** (13ms margin to P99=120ms) |

---

## Stage Latency Allocation Rationale

```
S7 Context Assembler (50ms) dominates because:
  - Renders objects from KIL index (cache hits: ~1ms, misses: ~30ms)
  - Cache hit rate target ≥ 85%
  - On cache miss: KIL index lookup + RENDER() + cache write

S2 Object Selector (20ms) is second:
  - Graph traversal (max 3 hops, max 50 candidates)
  - KIL index lookup for always_include references

S3 Relevance Ranker (15ms):
  - Cosine similarity (precomputed embeddings, vector dot product only)
  - BM25 scoring
  - Co-occurrence lookup
```

---

## Cache Hit Impact

```
Expected S7 breakdown given 85% cache hit rate:

  85% objects from L1 cache:
    render time ≈ 1ms per object

  10% objects from L2 cache:
    render time ≈ 5ms per object

  5% objects from index (cache miss):
    render time ≈ 30ms per object

  For a standard pack with 6 objects total:
    Expected S7 = 6 × (0.85×1 + 0.10×5 + 0.05×30)
               = 6 × (0.85 + 0.50 + 1.50)
               = 6 × 2.85
               ≈ 17ms (well within 50ms budget)
```

---

## Capacity Model

```
Assumptions for sizing:
  Peak QPS:     1,000 concurrent assembly requests
  Average pack: 6 objects × avg L3 = 6 × 200 tokens = 1,200 tokens
  Cache size:   top 10,000 objects × 5 levels = 50,000 entries

Index access pattern:
  Sequential reads (no transactions needed)
  P99 index lookup: < 5ms for top 10,000 objects
  Index size: 10,000 entries × ~20KB each = 200MB in-memory

Prefetch queue:
  Steady state: 200–500 tasks
  Peak burst: 2,000 tasks (handled by bounded queue — drop LOW priority)
```

---

## Performance Degradation Protocol

When the system cannot meet P99 targets, it must degrade gracefully:

```
Level 1 — Soft Degradation (P99 approaching 120ms):
  Action: Skip LIKELY prefetch tasks
  Action: Reduce max_candidates in object selector from 50 to 30
  Alert: KIM-CAE-LATENCY-P99 WARNING

Level 2 — Hard Degradation (P99 > 120ms for 3+ consecutive seconds):
  Action: Skip ALL prefetch tasks
  Action: Reduce max_candidates to 15
  Action: Force all context objects to L2 max
  Action: Drop EVIDENCE objects entirely
  Alert: KIM-CAE-LATENCY-P99 CRITICAL

Level 3 — Emergency Mode (P99 > 200ms):
  Action: Return cached pack for identical recent query (if available)
  Action: If no cached pack: AssemblyError(LATENCY_BUDGET_EXCEEDED)
  Action: Page on-call SRE
```

---

## Latency SLO Breach Response

```
SLO: P99 < 120ms measured over any 5-minute window

Breach definition: P99 >= 120ms for 3 consecutive 5-minute windows

On breach:
  1. Log SLO_BREACH event with metrics snapshot
  2. Activate Level 2 degradation
  3. Alert KIM-037 (assembly_latency_p99)
  4. Auto-recover when P99 returns to < 80ms for 2 consecutive windows
```

---

## Performance Anti-Patterns

The following patterns are explicitly forbidden:

```
FORBIDDEN: Synchronous database transactions in S1–S9
  Use: read-only index lookups only

FORBIDDEN: Blocking network calls in S1–S9
  Use: pre-loaded in-memory index or local L2 cache

FORBIDDEN: Recursive graph traversal without hop limit
  Max hops: 3 (enforced by Object Selector)
  Max candidates: 50 (enforced by Object Selector)

FORBIDDEN: LLM inference within the CAE pipeline
  Use: pre-computed embeddings only (cosine lookup, not generation)
  LLM inference is in the consumer layer, never in CAE

FORBIDDEN: Retry loops within a single assembly request
  If an object cannot be fetched, skip it — never retry synchronously
```

---

## Performance Metrics

| Metric ID | Name | Target | Alert |
|-----------|------|--------|-------|
| KIM-CAE-P50 | Assembly P50 latency | < 50ms | > 60ms |
| KIM-CAE-P95 | Assembly P95 latency | < 80ms | > 100ms |
| KIM-CAE-P99 | Assembly P99 latency | < 120ms | > 120ms (CRITICAL: > 200ms) |
| KIM-CAE-CACHE | Cache hit rate | ≥ 85% | < 70% |
| KIM-CAE-STAGE-MAX | Single stage max P99 | < 50ms (S7) | > 60ms |

---

## Rules

| Rule | Statement |
|------|-----------|
| CAE-096 | P99 < 120ms is an architecture constraint — any design violating it is invalid |
| CAE-097 | LLM inference is forbidden inside the CAE pipeline at all stages |
| CAE-098 | Degradation must be automatic — manual intervention is not fast enough |
| CAE-099 | Stage latency budgets in this document are binding — not informational |
| CAE-100 | Performance anti-patterns constitute architecture violations requiring remediation |

---

## Cross-References

- Context cache → `20-CONTEXT-CACHE`
- Prefetch engine → `19-PREFETCH-ENGINE`
- Context metrics → `23-CONTEXT-METRICS`
- CAE certification → `26-CAE-CERTIFICATION`
