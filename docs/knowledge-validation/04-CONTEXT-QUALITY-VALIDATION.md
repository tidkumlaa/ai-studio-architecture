# KVF-DOC-004 — Context Quality Validation

**Phase:** 3.0D.1.5
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Context Quality Validation measures whether context packs produced by a KOS
Runtime are actually good — relevant to the query, complete for the task,
safe from hallucination risk, and efficient with tokens.

This document defines the measurement methodology for all 8 context quality
dimensions.

---

## 8 Quality Dimensions

```
Dimension 1: RELEVANCE
  Is the primary object the correct answer to the query?
  Is each context object genuinely related?

Dimension 2: COMPLETENESS
  Are all required knowledge objects present?
  Is the primary at sufficient compression depth?

Dimension 3: CONSISTENCY
  Do all objects in the pack agree on shared facts?
  Is there internal contradiction?

Dimension 4: SAFETY
  Is every object's AIRS >= 0.60?
  Is guard block present and complete?
  Are deprecated objects flagged?

Dimension 5: CONFIDENCE
  Does context_confidence accurately reflect object confidence?
  Is confidence propagation correctly applied (minimum, not mean)?

Dimension 6: COMPRESSION
  Is the compression level appropriate for the budget and role?
  Is the information loss at each level acceptable?

Dimension 7: RANKING
  Are context objects ordered correctly (prerequisites first)?
  Are anti-confusion objects labeled and last?

Dimension 8: COVERAGE
  What fraction of the task's required knowledge is present in the pack?
```

---

## Quality Measurement Protocol

```
For each test query Q in the test set:

  1. Run the Runtime: pack = assemble(Q, consumer, budget)
  2. Run human expert review (for golden dataset queries)
     OR automated evaluation (for bulk queries)
  3. Score each dimension 0.0–1.0
  4. Compute CAE quality score (from doc 17):
       quality = relevance×0.35 + completeness×0.30 + safety×0.25 + efficiency×0.10
  5. Compare Runtime's declared quality_score vs. externally measured quality
  6. Record: match within ±0.05 tolerance = pass
```

---

## Dimension Measurement Methods

### Relevance

```
Automated measurement:
  relevance = cosine_similarity(query.embedding, primary_object.embedding)

  For top-k: proportion of top-K objects that are rated relevant by annotators
  OR (automated): relevance_score >= 0.60

Human annotation (golden dataset):
  Annotators rate: Is this the right object? (0 = wrong, 1 = correct)
  Annotators rate: Are context objects relevant? (0–1 per object)
  Agreement threshold: kappa >= 0.70
```

### Completeness

```
Automated measurement:
  primary_depth = compression_level_score(pack.primary.compression_level)
    L5 → 1.00, L4 → 0.85, L3 → 0.65

  required_context_coverage = (
    objects_in_pack that are in ai_context.related_objects[ALWAYS]
    / total ai_context.related_objects[ALWAYS]
  )

  completeness = primary_depth×0.50 + required_context_coverage×0.30 + guard_completeness×0.20
```

### Consistency

```
Automated measurement:
  For each pair of objects (A, B) in the pack that share facts:
    Check: A.fact == B.fact for declared shared facts
    Consistency score = consistent_pairs / total_pairs

  Consistency check is applied to:
    - Dependency claims (if A says it depends on B, B should appear)
    - Version claims (all objects agree on package version if stated)
    - Status claims (no ACTIVE object depends on DEPRECATED without warning)
```

### Safety

```
Automated measurement (exact):
  safety_checks = [
    all_objects_airs >= 0.60,           # KC-5.03 equivalent
    guard_block_present,                 # for PromptPack
    scope_boundaries_non_empty,
    confidence_hedge_present,
    deprecated_status_warned
  ]
  safety = sum(passed_checks) / len(safety_checks)
```

---

## Context Quality Validation Result Schema

```yaml
context_quality_result:
  queries_evaluated: integer
  human_annotated: integer              # subset with human review
  automated: integer                    # automated evaluation

  dimension_scores:
    relevance:    {mean: float, p25: float, p75: float}
    completeness: {mean: float, p25: float, p75: float}
    consistency:  {mean: float, p25: float, p75: float}
    safety:       {mean: float, p25: float, p75: float}
    confidence:   {mean: float, p25: float, p75: float}
    compression:  {mean: float, p25: float, p75: float}
    ranking:      {mean: float, p25: float, p75: float}
    coverage:     {mean: float, p25: float, p75: float}

  cae_quality_score:
    mean: float
    below_threshold_0_70: integer       # packs below minimum
    below_threshold_pct: float

  accuracy_of_declared_score:           # how often Runtime's score ≈ external
    within_0_05: float                  # proportion within tolerance
    mean_absolute_error: float
```

---

## Target Quality Metrics

| Dimension | Minimum (Bronze) | Target (Gold) |
|-----------|-----------------|--------------|
| Relevance | ≥ 0.70 | ≥ 0.85 |
| Completeness | ≥ 0.70 | ≥ 0.85 |
| Consistency | ≥ 0.90 | ≥ 0.97 |
| Safety | ≥ 0.95 | 1.00 |
| Confidence | ≥ 0.85 | ≥ 0.95 |
| Compression | ≥ 0.80 | ≥ 0.90 |
| Ranking | ≥ 0.80 | ≥ 0.92 |
| Coverage | ≥ 0.65 | ≥ 0.80 |

---

## Rules

| Rule | Statement |
|------|-----------|
| KVF-016 | Safety score of < 0.95 is a critical failure — it blocks all certification above BRONZE |
| KVF-017 | Consistency checks must cover at least 50 shared-fact pairs per test run |
| KVF-018 | Declared quality_score accuracy (within ±0.05) must be ≥ 80% of packs |
| KVF-019 | Relevance measurement must use pre-computed embeddings — not inference |
| KVF-020 | Human annotation is mandatory for the golden dataset (doc 23) — automation alone insufficient |

---

## Cross-References

- Context quality model → Phase 3.0D.1 `17-CONTEXT-QUALITY-MODEL`
- Confidence propagation → Phase 3.0D.1 `18-CONFIDENCE-PROPAGATION`
- Context correctness → `05-CONTEXT-CORRECTNESS`
- Golden dataset → `23-GOLDEN-DATASET-VALIDATION`
