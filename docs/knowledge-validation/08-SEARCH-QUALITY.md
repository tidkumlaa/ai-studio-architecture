# KVF-DOC-008 — Search Quality

**Phase:** 3.0D.1.5
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Search Quality defines benchmarks for the KOS knowledge search capability
across query volumes of 500 / 1,000 / 5,000 / 10,000 queries, with
standard IR metrics: Top-1/5 accuracy, Precision, Recall, MRR, NDCG,
and latency.

---

## Search Benchmark Tiers

| Tier | Query Count | Annotation | Purpose |
|------|------------|------------|---------|
| MINI | 500 | Human (200) + automated (300) | Initial validation |
| STANDARD | 1,000 | Human (300) + automated (700) | Certification |
| LARGE | 5,000 | Human (500) + automated (4,500) | Scale validation |
| FULL | 10,000 | Human (500) + automated (9,500) | Enterprise certification |

---

## Search Quality Metrics

### Top-K Accuracy

```
Top-1 Accuracy:
  correct if the #1 ranked result is in the golden relevant set
  Target: >= 0.85

Top-5 Accuracy:
  correct if at least one of the top-5 results is in the golden relevant set
  Target: >= 0.95
```

### Precision and Recall

```
Precision@K:
  P@K = |relevant_in_top_K| / K

  P@1:  target >= 0.85
  P@5:  target >= 0.75
  P@10: target >= 0.70

Recall@K:
  R@K = |relevant_in_top_K| / |total_relevant|

  R@5:  target >= 0.65
  R@10: target >= 0.75
```

### Mean Reciprocal Rank (MRR)

```
MRR = (1/Q) × sum(1/rank_of_first_relevant for q in queries)

Where rank_of_first_relevant is the rank position of the first
correct result in the returned list.

Target: >= 0.85

Interpretation:
  MRR=1.0: first result is always correct
  MRR=0.5: first relevant result is at position 2 on average
  MRR=0.85: ~1.2 results before the first relevant one
```

### Normalized Discounted Cumulative Gain (NDCG)

```
DCG@K = sum(relevance_grade(i) / log2(i+1) for i in 1..K)
NDCG@K = DCG@K / IDCG@K

Where:
  relevance_grade: 3=highly relevant, 2=relevant, 1=partially, 0=irrelevant
  IDCG@K: ideal DCG (perfect ranking)

Target NDCG@10: >= 0.80
```

### Search Latency

```
P50: < 30ms
P95: < 80ms
P99: < 120ms

Measured: from search request received to SearchPack emitted
Under standard load: 100 concurrent search requests
```

---

## Search Query Taxonomy

The 500/1,000/5,000/10,000 queries are distributed across types:

| Query Type | Distribution |
|------------|-------------|
| IDENTITY (by name/ID) | 25% |
| KEYWORD (by topic) | 30% |
| SEMANTIC (natural language) | 25% |
| GRAPH (relationship-based) | 10% |
| MULTI-TERM (compound) | 10% |

---

## Search Result Quality Check

Beyond IR metrics, each SearchPack is checked for correctness:

```
For each SearchResult in the pack:
  CK-S-01: knowledge_id is a valid, existing object ID
  CK-S-02: relevance_score is in [0.0, 1.0]
  CK-S-03: match_reason is human-readable (not field names)
  CK-S-04: quality_score matches object metadata.quality_score
  CK-S-05: result count <= 10
  CK-S-06: empty_reason is set when result count == 0
  CK-S-07: facets reflect all candidates, not just top-10
```

---

## Negative Test Queries

10% of the benchmark queries are designed to have no relevant results:

```
Negative queries:
  - Query for non-existent object names
  - Query for objects that exist but fail quality threshold
  - Query with ambiguous terms that match nothing well

For negative queries:
  Expected: SearchPack with results=[] and empty_reason set
  Failure: returning irrelevant results for no-answer queries
```

---

## Search Quality Result Schema

```yaml
search_quality_result:
  tier: string                          # MINI | STANDARD | LARGE | FULL
  queries_run: integer
  queries_human_annotated: integer

  ir_metrics:
    top1_accuracy: float
    top5_accuracy: float
    precision_at_1: float
    precision_at_5: float
    precision_at_10: float
    recall_at_5: float
    recall_at_10: float
    mrr: float
    ndcg_at_10: float

  latency:
    p50_ms: float
    p95_ms: float
    p99_ms: float

  by_query_type:
    - query_type: string
      top1_accuracy: float
      mrr: float
      query_count: integer

  result_quality_checks:
    - check_id: string
      pass_rate: float

  negative_query_accuracy: float        # % correctly returning empty results
```

---

## Targets by Certification Level

| Metric | BRONZE | SILVER | GOLD | ENTERPRISE |
|--------|--------|--------|------|------------|
| Top-1 | ≥ 0.70 | ≥ 0.80 | ≥ 0.85 | ≥ 0.90 |
| MRR | ≥ 0.70 | ≥ 0.80 | ≥ 0.85 | ≥ 0.90 |
| NDCG@10 | ≥ 0.65 | ≥ 0.75 | ≥ 0.80 | ≥ 0.85 |
| P99 Latency | < 200ms | < 150ms | < 120ms | < 100ms |

---

## Rules

| Rule | Statement |
|------|-----------|
| KVF-036 | MRR must be computed using the full ranked list, not just top-10 |
| KVF-037 | Negative queries must be at least 10% of the benchmark — they test precision |
| KVF-038 | NDCG must use 3-grade relevance scale — binary relevance is insufficient |
| KVF-039 | Search latency must be measured under concurrent load, not sequential |
| KVF-040 | CK-S-06 (empty_reason for empty results) failure is a UX bug, not just metric |

---

## Cross-References

- Knowledge coverage → `07-KNOWLEDGE-COVERAGE`
- Relevance model → Phase 3.0D.1 `06-RELEVANCE-MODEL`
- Search pack spec → Phase 3.0D.1 `14-SEARCH-PACK`
- KIL search optimization → Phase 3.0D.0.6 `28-KNOWLEDGE-SEARCH-OPTIMIZATION`
- Golden dataset → `23-GOLDEN-DATASET-VALIDATION`
