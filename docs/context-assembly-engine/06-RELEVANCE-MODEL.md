# CAE-DOC-006 — Relevance Model

**Phase:** 3.0D.1
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Relevance Ranker scores every candidate object against the query and intent,
then selects the top-K for assembly. High recall (Object Selector) + precise
ranking (Relevance Model) together determine context quality.

---

## Relevance Score Formula

```
relevance_score(object, query) =
  (semantic_similarity  × 0.35) +
  (keyword_match        × 0.20) +
  (graph_proximity      × 0.20) +
  (ai_readiness_score   × 0.10) +
  (co_occurrence_rate   × 0.10) +
  (freshness_factor     × 0.05)

All components: 0.0–1.0
Weights sum to 1.00
```

---

## Component Definitions

### semantic_similarity (weight 0.35)

```
semantic_similarity =
  cosine_similarity(query.embedding, object.search.vector.embedding)

Where:
  query.embedding    = embed(query.raw_query)
  object.search.vector.embedding = pre-computed from KIL intelligence.search.vector

If object has no embedding:
  semantic_similarity = 0.0
  (triggers KIM-036 alert: vector not populated)
```

### keyword_match (weight 0.20)

```
keyword_match = BM25_score(query.tokens, object.search.keyword.primary_terms
                           + object.search.keyword.aliases
                           + object.name)

Boosted fields:
  canonical_name  × 3.0
  short_summary   × 2.0
  capabilities    × 1.5
  name            × 1.0

Negative keywords:
  IF any query token in object.search.keyword.negative_keywords:
    keyword_match × 0.50  (penalty, not zero — object may still be relevant)
```

### graph_proximity (weight 0.20)

```
graph_proximity =
  1.0 / (hop_distance(primary_entity, object) + 1)

hop_distance definitions:
  0 hops (is the primary)         → 1.00
  1 hop (direct neighbor)         → 0.50
  2 hops                          → 0.33
  3 hops                          → 0.25
  not connected                   → 0.05

Boost for relationship type:
  HARD DEPENDS_ON or DEPENDENCY_OF  × 1.2
  IMPLEMENTS or IMPLEMENTED_BY       × 1.1
  TESTS or TESTED_BY                 × 1.0
  RELATED_TO                         × 0.8
```

### ai_readiness_score (weight 0.10)

```
ai_readiness_score = object.intelligence.ai_readiness_score

Rationale: a highly AI-ready object produces better context
even if slightly less semantically similar.
```

### co_occurrence_rate (weight 0.10)

```
co_occurrence_rate =
  primary_object.usage.co_occurrence_matrix
    .find(knowledge_id == object.knowledge_id)
    .co_access_rate

Default if not in co-occurrence matrix: 0.0
```

### freshness_factor (weight 0.05)

```
staleness_days = (now - object.metadata.updated_at).days

freshness_factor =
  IF staleness_days <= 30:  1.00
  IF staleness_days <= 90:  0.80
  IF staleness_days <= 180: 0.60
  IF staleness_days <= 365: 0.40
  IF staleness_days >  365: 0.20
```

---

## Intent-Weighted Variants

For specific intents, component weights shift:

| Intent | semantic | keyword | graph | airs | co-occur | fresh |
|--------|---------|---------|-------|------|----------|-------|
| INT-06 SEARCH | 0.50 | 0.30 | 0.05 | 0.10 | 0.05 | 0.00 |
| INT-04 IMPACT | 0.20 | 0.10 | 0.45 | 0.10 | 0.10 | 0.05 |
| INT-03 REASONING | 0.40 | 0.15 | 0.20 | 0.10 | 0.10 | 0.05 |
| INT-08 CONTEXT | 0.35 | 0.15 | 0.20 | 0.15 | 0.10 | 0.05 |
| INT-09 TRACEABILITY | 0.15 | 0.10 | 0.55 | 0.10 | 0.05 | 0.05 |

All intent-variant weights sum to 1.00.

---

## Object Role Assignment

After scoring, each candidate is assigned a role based on score and selection_source:

```
ASSIGN_ROLE(object, score, selection_source, intent):

  IF object.knowledge_id == primary_entity.knowledge_id:
    role = PRIMARY

  ELIF score >= 0.70 AND selection_source in {ALWAYS_INCLUDE, GRAPH}:
    role = CONTEXT

  ELIF score >= 0.50 AND selection_source == GRAPH AND hop_distance <= 1:
    role = CONTEXT

  ELIF object in primary.alternatives AND object.similarity_to_this > 0.70:
    role = ANTI_CONFUSION

  ELIF score >= 0.40 AND object in primary.reasoning.requires:
    role = PREREQUISITE

  ELIF score >= 0.30:
    role = EVIDENCE

  ELSE:
    role = DROPPED                     # not included in assembly
```

---

## Top-K Selection

```
RANK_AND_SELECT(candidates, budget):

  1. Score all candidates
  2. Sort by relevance_score DESC
  3. Force PRIMARY to position 0
  4. Force ANTI_CONFUSION objects if similarity_to_this > 0.70 (max 2)
  5. Fill remaining slots greedily until:
     estimated_tokens(selected) + guard_block_tokens >= budget × 0.90
  6. Always reserve 10% of budget for guard block
  7. Return selected with roles
```

---

## Rules

| Rule | Statement |
|------|-----------|
| CAE-021 | Relevance formula weights must sum to exactly 1.00 for all intent variants |
| CAE-022 | semantic_similarity = 0.0 when no embedding exists — never interpolated |
| CAE-023 | ANTI_CONFUSION objects must be included when `similarity_to_this > 0.70` |
| CAE-024 | DROPPED objects must be logged for feedback — they improve future selection |
| CAE-025 | The top-K selection must reserve guard_block budget before filling content |

---

## Cross-References

- Object selector → `05-OBJECT-SELECTOR`
- Budget manager → `07-BUDGET-MANAGER`
- KIL search block → Phase 3.0D.0.6 `28-KNOWLEDGE-SEARCH-OPTIMIZATION`
- Co-occurrence matrix → Phase 3.0D.0.6 `10-KNOWLEDGE-USAGE-MODEL`
