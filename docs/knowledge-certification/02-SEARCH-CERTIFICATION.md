# KNW-CERT-ARCH-002 — Search Certification

**Phase:** 3.0D.0  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Certifies that the Knowledge Search subsystem delivers correct, fast, and robust results across a range of query sizes and query types.

---

## Test Dataset

Search certification uses the Golden Dataset (`21-DATASET-CERTIFICATION`):
- 200 curated objects with known-correct search results
- 500 relationships
- Pre-computed ground-truth answer sets for each query batch

---

## Query Batches

| Batch | Queries | Seed | Purpose |
|-------|---------|------|---------|
| S-100 | 100 | 42 | Smoke test |
| S-500 | 500 | 42 | Bronze/Silver baseline |
| S-1K | 1,000 | 42 | Gold baseline |
| S-5K | 5,000 | 42 | Enterprise baseline |
| S-10K | 10,000 | 42 | Research baseline |

Default run uses S-500 for Bronze/Silver and S-1K for Gold+.

---

## Accuracy Metrics

| Metric | Formula | Bronze | Silver | Gold | Enterprise | Research |
|--------|---------|--------|--------|------|------------|---------|
| Top1 Accuracy | correct@1 / total | ≥ 0.70 | ≥ 0.75 | ≥ 0.82 | ≥ 0.88 | ≥ 0.93 |
| Top5 Accuracy | correct@5 / total | ≥ 0.80 | ≥ 0.85 | ≥ 0.90 | ≥ 0.93 | ≥ 0.97 |
| Top10 Accuracy | correct@10 / total | ≥ 0.85 | ≥ 0.90 | ≥ 0.94 | ≥ 0.97 | ≥ 0.99 |
| Precision@10 | relevant in top10 / 10 | ≥ 0.65 | ≥ 0.72 | ≥ 0.80 | ≥ 0.87 | ≥ 0.93 |
| Recall@10 | relevant found / total relevant | ≥ 0.60 | ≥ 0.68 | ≥ 0.78 | ≥ 0.85 | ≥ 0.92 |
| MRR | mean reciprocal rank | ≥ 0.65 | ≥ 0.72 | ≥ 0.80 | ≥ 0.87 | ≥ 0.93 |
| NDCG@10 | normalized discounted cumulative gain | ≥ 0.60 | ≥ 0.68 | ≥ 0.78 | ≥ 0.86 | ≥ 0.93 |

---

## Latency Targets (at 1K objects)

| Percentile | Bronze | Silver | Gold | Enterprise | Research |
|------------|--------|--------|------|------------|---------|
| P50 | ≤ 200ms | ≤ 100ms | ≤ 20ms | ≤ 10ms | ≤ 5ms |
| P95 | ≤ 500ms | ≤ 300ms | ≤ 50ms | ≤ 25ms | ≤ 15ms |
| P99 | ≤ 1000ms | ≤ 500ms | ≤ 100ms | ≤ 50ms | ≤ 25ms |

---

## Resource Limits (at 1K objects)

| Resource | Limit |
|----------|-------|
| Memory | ≤ 512 MB for index |
| CPU per query | ≤ 100ms user time |
| Index build time | ≤ 30s for 1K objects |

---

## Edge Case Tests

| Check | ID | Severity |
|-------|----|----------|
| Typo tolerance (1 edit distance) | SC-001 | MAJOR |
| Alias handling (canonical_name vs knowledge_id) | SC-002 | MAJOR |
| Namespace-qualified search (`plt:quota`) | SC-003 | MAJOR |
| Empty query returns zero results | SC-004 | CRITICAL |
| Query longer than 512 chars handled | SC-005 | MAJOR |
| Unicode query (Thai, CJK) handled | SC-006 | MINOR |
| Wildcard query (`quota*`) | SC-007 | MINOR |
| Boolean query (`quota AND manager`) | SC-008 | MINOR |
| Phrase query (`"quota manager"`) | SC-009 | MINOR |
| Search across archived objects excluded | SC-010 | MAJOR |
| False positive rate ≤ 5% | SC-011 | MAJOR |
| False negative rate ≤ 10% | SC-012 | MAJOR |

---

## Search Check Output Format

```json
{
  "domain": "search",
  "batch": "S-1K",
  "seed": 42,
  "timestamp": "2026-06-30T00:00:00Z",
  "metrics": {
    "top1_accuracy": 0.847,
    "top5_accuracy": 0.921,
    "top10_accuracy": 0.956,
    "precision_at_10": 0.831,
    "recall_at_10": 0.809,
    "mrr": 0.842,
    "ndcg_at_10": 0.813,
    "p50_ms": 12.4,
    "p95_ms": 38.7,
    "p99_ms": 87.2,
    "false_positive_rate": 0.031,
    "false_negative_rate": 0.072
  },
  "edge_cases": {
    "SC-001": "PASS",
    "SC-002": "PASS",
    "SC-003": "PASS",
    "SC-004": "PASS",
    "SC-005": "PASS",
    "SC-006": "PASS",
    "SC-007": "FAIL",
    "SC-008": "PASS",
    "SC-009": "PASS",
    "SC-010": "PASS",
    "SC-011": "PASS",
    "SC-012": "PASS"
  },
  "domain_score": 0.831,
  "level_achieved": "Gold"
}
```

---

## CLI

```bash
kos-cert search                          # S-500 with default seed
kos-cert search --queries 1000           # S-1K
kos-cert search --queries 10000 --seed 99
kos-cert search --edge-cases-only        # skip query batches
kos-cert search --output reports/search.json
```

---

## Cross-References

- Golden dataset → `21-DATASET-CERTIFICATION`
- Overall score weight (10%) → `27-OVERALL-SCORING`
- Reasoning search questions → `13-REASONING-CERTIFICATION`
