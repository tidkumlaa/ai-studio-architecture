# UICE-DOC-011 — Context Ranking Engine

**Phase:** 3.0D.2
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Context Ranking Engine orders the candidate objects returned by Context Graph
Traversal by their relevance to the current intent and query. The ranking
determines which objects enter the context pack and in what order, directly
affecting the quality of the assembled context.

UICE extends the CAE S3 relevance formula with two new dimensions:
**intent_alignment** (how well the object serves this specific intent type)
and **history_weight** (whether this object has been useful in similar past queries).

---

## UICE Relevance Formula

```
relevance_score =
  semantic_similarity   × 0.30  +   // embedding-based match to query
  keyword_match         × 0.15  +   // query term overlap with KIL fields
  graph_proximity       × 0.15  +   // hops from primary (1-hop = 1.0, 2-hop = 0.5, 3-hop = 0.25)
  ai_readiness_score    × 0.10  +   // higher AIRS = better AI context quality
  co_occurrence_rate    × 0.10  +   // how often appears with primary in other packs
  freshness_factor      × 0.05  +   // 1.0 = current, decays to 0.5 for deprecated
  intent_alignment      × 0.10  +   // NEW: how relevant for this specific intent_type
  history_weight        × 0.05      // NEW: performance in past similar queries

Total: 1.00
```

### Comparison to CAE S3 Formula

```
CAE:  semantic×0.35 + keyword×0.20 + graph×0.20 + airs×0.10 + co_occurrence×0.10 + freshness×0.05
UICE: semantic×0.30 + keyword×0.15 + graph×0.15 + airs×0.10 + co_occurrence×0.10 + freshness×0.05
      + intent_alignment×0.10 + history_weight×0.05
```

UICE redistributes 0.15 weight from semantic+keyword+graph to intent_alignment+history.

---

## New Dimension: Intent Alignment

```
COMPUTE_INTENT_ALIGNMENT(candidate_object, intent_type) → float:

  // Objects directly relevant to the intent type score higher
  intent_relevance_map = {
    (CODE_GENERATION, INTERFACE):      1.00,
    (CODE_GENERATION, SPECIFICATION):  0.90,
    (CODE_GENERATION, PATTERN):        0.80,
    (TRACEABILITY, any):               based on relationship depth (1/depth)
    (IMPACT_ANALYSIS, direct_dep):     1.00,
    (IMPACT_ANALYSIS, indirect_dep):   0.60,
    (MIGRATION_GUIDE, DEPRECATED):     1.00,
    (ARCHITECTURE_REVIEW, DECISION):   0.90,
    (DIAGNOSTIC, SERVICE):             0.80,
    (DIAGNOSTIC, CONFIGURATION):       0.90,
  }

  // Object type alignment
  obj_type = candidate_object.object_type
  type_alignment = intent_relevance_map.get((intent_type, obj_type), 0.50)

  // AIRS threshold bonus: AI-NATIVE objects align better with AI-targeted intents
  airs_bonus = 0.10 if candidate_object.airs >= 0.90 and intent_type in AI_INTENTS else 0.0

  return min(1.0, type_alignment + airs_bonus)
```

---

## New Dimension: History Weight

```
COMPUTE_HISTORY_WEIGHT(candidate_id, intent_type, query_type) → float:

  // Query Context Learning module for historical performance
  history = CONTEXT_LEARNING.get_history(candidate_id, intent_type, query_type)

  if history is None:
    return 0.50  // no history — neutral weight

  // Score based on past feedback
  helpfulness_mean = history.mean_helpfulness_score  // 0.0–1.0
  inclusion_count  = history.times_included_in_packs
  positive_rate    = history.positive_feedback_rate  // explicit positive / total

  // Weight formula: recent history matters more
  return (
    helpfulness_mean  × 0.50 +
    positive_rate     × 0.30 +
    min(1.0, inclusion_count / 100) × 0.20
  )
```

---

## RankedObject Schema

```yaml
RankedObject:
  knowledge_id: string
  name: string
  object_type: string
  relevance_score: float           # 0.0–1.0 final score
  relevance_breakdown:
    semantic_similarity: float
    keyword_match: float
    graph_proximity: float
    ai_readiness_score: float
    co_occurrence_rate: float
    freshness_factor: float
    intent_alignment: float
    history_weight: float
  rank: int                        # 1 = highest relevance
  in_memory: bool
```

---

## Ranking Algorithm

```
RANK_OBJECTS(traversal_results, intent_profile, query_profile,
             memory_filter, learning) → list[RankedObject]:

  ranked = []
  for obj in traversal_results:
    score = (
      SEMANTIC_SIMILARITY(obj, query_profile.query_tokens)  × 0.30 +
      KEYWORD_MATCH(obj, query_profile.query_tokens)         × 0.15 +
      GRAPH_PROXIMITY(obj.depth)                             × 0.15 +
      obj.ai_readiness_score                                 × 0.10 +
      CO_OCCURRENCE_RATE(obj.knowledge_id, primary_id)       × 0.10 +
      FRESHNESS(obj.status, obj.updated_at)                  × 0.05 +
      INTENT_ALIGNMENT(obj, intent_profile.intent_type)      × 0.10 +
      HISTORY_WEIGHT(obj.knowledge_id, intent_profile, query_profile) × 0.05
    )
    ranked.append(RankedObject(
      knowledge_id         = obj.knowledge_id,
      relevance_score      = score,
      relevance_breakdown  = {...},
      in_memory            = memory_filter.contains(obj.knowledge_id),
    ))

  return SORT_DESC(ranked, key=lambda r: r.relevance_score)
```

---

## Freshness Factor

```
FRESHNESS(status, updated_at) → float:
  if status == DEPRECATED:  return 0.50
  if status == DRAFT:       return 0.70
  if status == CANONICAL:
    age_days = (NOW - updated_at).days
    if age_days < 30:  return 1.00
    if age_days < 90:  return 0.95
    if age_days < 365: return 0.85
    else:              return 0.75
```

---

## Rules

| Rule | Statement |
|------|-----------|
| UICE-051 | Relevance formula weights must sum to exactly 1.00 |
| UICE-052 | DEPRECATED objects score freshness_factor = 0.50 — they may still be relevant for MIGRATION_GUIDE |
| UICE-053 | History weight defaults to 0.50 (neutral) when no history exists; it never blocks inclusion |
| UICE-054 | In-memory objects receive no automatic relevance bonus; their relevance reflects content quality |
| UICE-055 | Relevance breakdown must be stored in RankedObject for Context Profiler explainability |

---

## Cross-References

- Context Graph Traversal → `08-CONTEXT-GRAPH-TRAVERSAL`
- Context Learning (history source) → `19-CONTEXT-LEARNING`
- Dynamic Context Planner → `04-DYNAMIC-CONTEXT-PLANNER`
- CAE S3 Relevance Ranker → Phase 3.0D.1 `06-RELEVANCE-RANKER`
