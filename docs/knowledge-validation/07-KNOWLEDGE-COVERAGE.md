# KVF-DOC-007 — Knowledge Coverage

**Phase:** 3.0D.1.5
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Knowledge Coverage measures whether, for a given task, the KOS Runtime retrieves
all the knowledge objects actually needed to complete that task — and does not
retrieve irrelevant objects that waste the token budget.

---

## Coverage Definitions

```
Required Objects (R):
  The set of KnowledgeObjects that a domain expert would consider
  necessary to correctly answer query Q.
  Source: golden dataset (doc 23) human annotation.

Retrieved Objects (P):
  The set of KnowledgeObjects included in the assembled context pack.

Missing Objects (M):
  M = R - P  (required but not retrieved)
  Causes: object selector missed it; budget too small; AIRS too low

Extra Objects (E):
  E = P - R  (retrieved but not needed)
  Causes: false positives in relevance ranking; poor graph traversal
```

---

## Coverage Metrics

```
Coverage (Recall):
  coverage = |R ∩ P| / |R|
  = fraction of required objects that were retrieved
  Target: >= 0.85

Precision:
  precision = |R ∩ P| / |P|
  = fraction of retrieved objects that are required
  Target: >= 0.75

F1 Score:
  f1 = 2 × (precision × coverage) / (precision + coverage)
  Target: >= 0.80

Missing Object Rate:
  missing_rate = |M| / |R|
  Target: <= 0.15

Extra Object Rate:
  extra_rate = |E| / |P|
  Target: <= 0.25
```

---

## Coverage by Object Role

Different object roles have different coverage expectations:

```
PRIMARY:
  Must be exactly correct — 100% coverage
  Metric: primary_correct_rate (= 1.0 means primary is always in R)
  Target: >= 0.99

ALWAYS_INCLUDE (from ai_context.related_objects):
  Objects with fetch_priority=ALWAYS must be retrieved if in R
  Target: >= 0.95

PREREQUISITE:
  Objects that must be understood first
  Target: >= 0.90

CONTEXT:
  Supporting objects
  Target: >= 0.80

EVIDENCE:
  Corroborating objects
  Target: >= 0.60 (these are least critical)
```

---

## Coverage Measurement Protocol

```
For each task Q in the coverage test set:

  1. Load golden required set R(Q) from golden dataset
  2. Run: pack = Runtime.assemble(Q, consumer, budget=STANDARD)
  3. Extract P = {obj.knowledge_id for all obj in pack}
  4. Compute coverage, precision, F1, missing, extra
  5. For each missing object in M:
     - Was it in the KIL index? (not in index = index gap)
     - Was it excluded by AIRS filter? (AIRS too low = KIL quality gap)
     - Was it budget-dropped? (dropped_objects log)
     - Was it below relevance threshold? (relevance gap)
  6. Record root cause for each miss
```

---

## Missing Object Root Cause Analysis

```yaml
missing_object_record:
  knowledge_id: string
  query_id: string
  was_in_index: boolean
  airs: float | null
  was_airs_excluded: boolean
  was_budget_dropped: boolean
  relevance_score_if_ranked: float | null
  root_cause: NOT_IN_INDEX | AIRS_TOO_LOW | BUDGET_DROPPED | RELEVANCE_MISS | UNKNOWN
```

Root cause distribution helps distinguish implementation bugs (RELEVANCE_MISS)
from KIL quality issues (AIRS_TOO_LOW) from corpus completeness issues (NOT_IN_INDEX).

---

## Coverage by Budget Tier

Coverage naturally varies with budget. Validation must test all tiers:

| Budget Tier | Tokens | Expected Coverage |
|-------------|--------|-----------------|
| NANO | 100 | ≥ 0.50 (primary only) |
| MICRO | 300 | ≥ 0.65 |
| STANDARD | 1,000 | ≥ 0.80 |
| EXTENDED | 3,000 | ≥ 0.90 |
| FULL | 8,000 | ≥ 0.95 |

---

## Coverage Result Schema

```yaml
coverage_result:
  queries_evaluated: integer
  budget_tier: string

  aggregate:
    mean_coverage: float
    mean_precision: float
    mean_f1: float
    missing_rate: float
    extra_rate: float

  by_object_role:
    primary_correct_rate: float
    always_include_coverage: float
    prerequisite_coverage: float
    context_coverage: float
    evidence_coverage: float

  missing_object_analysis:
    total_missing: integer
    by_root_cause:
      not_in_index: integer
      airs_too_low: integer
      budget_dropped: integer
      relevance_miss: integer
      unknown: integer

  sample_misses:
    - query_id: string
      missing_knowledge_id: string
      root_cause: string
```

---

## Rules

| Rule | Statement |
|------|-----------|
| KVF-031 | PRIMARY correct rate must be ≥ 0.99 for Gold+ certification |
| KVF-032 | Coverage tests must run at all budget tiers — STANDARD tier alone is insufficient |
| KVF-033 | Missing object root cause must be identified — UNKNOWN > 10% of misses is a monitoring failure |
| KVF-034 | ALWAYS_INCLUDE coverage below 0.90 triggers KIL ai_context review |
| KVF-035 | Precision below 0.60 is a token efficiency failure — irrelevant objects waste budget |

---

## Cross-References

- Token efficiency → `06-TOKEN-EFFICIENCY`
- Object selector spec → Phase 3.0D.1 `05-OBJECT-SELECTOR`
- Relevance model spec → Phase 3.0D.1 `06-RELEVANCE-MODEL`
- Golden dataset → `23-GOLDEN-DATASET-VALIDATION`
