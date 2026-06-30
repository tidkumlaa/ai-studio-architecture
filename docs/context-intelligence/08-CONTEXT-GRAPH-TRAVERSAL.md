# UICE-DOC-008 — Context Graph Traversal

**Phase:** 3.0D.2
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Context Graph Traversal selects which KnowledgeObjects to include in a context
pack by traversing the knowledge graph starting from the primary object.
Unlike static "add top-5 related objects," UICE graph traversal is intent-aware:
a TRACEABILITY query traverses dependency chains, while a FACTUAL_LOOKUP+IDENTITY
query needs no traversal at all.

The traversal must be bounded, relevance-scored, and memory-aware (objects
already in conversation memory need fewer tokens and may be traversed more
aggressively).

---

## Traversal Strategy by Intent

```
FACTUAL_LOOKUP + IDENTITY:
  depth = 0 (no traversal — primary only)
  max_objects = 1

FACTUAL_LOOKUP + BEHAVIORAL:
  depth = 1 (direct dependencies only)
  max_objects = 3

IMPACT_ANALYSIS:
  depth = 2 (what does primary affect, and what does that affect)
  max_objects = 8
  direction = DOWNSTREAM (follow impacts, not requirements)

TRACEABILITY + RELATIONAL:
  depth = min(query_profile.hop_depth, 4)
  max_objects = 12
  direction = UPSTREAM (follow requirements chain)

CODE_GENERATION:
  depth = 1 (implements, extends, requires)
  max_objects = 5
  direction = MIXED

CONTEXT_PRODUCTION:
  depth = 2 (wide context for AI agent)
  max_objects = 15
  direction = MIXED

COMPARISON:
  depth = 1 per comparison target
  max_objects = 3 per target × number_of_targets
  direction = MIXED

SEARCH:
  depth = 0 (search result list — no deep traversal)
  max_objects = 10 (top search hits)
```

---

## Traversal Algorithm

```
TRAVERSE_GRAPH(primary_id, intent_profile, query_profile, budget) → list[TraversalResult]:

  strategy = GET_STRATEGY(intent_profile.intent_type, query_profile.query_type)
  max_depth = strategy.depth
  max_objects = strategy.max_objects
  direction = strategy.direction

  // BFS with relevance-scored priority queue
  visited = {primary_id}
  queue = PriorityQueue()  // min-heap by negative relevance
  results = []

  // Seed with primary's relationships
  relationships = LOAD_RELATIONSHIPS(primary_id)
  for rel in FILTER_BY_DIRECTION(relationships, direction):
    if rel.target_id not in visited:
      relevance = SCORE_RELEVANCE(rel, intent_profile, query_profile)
      queue.push(TraversalNode(
        knowledge_id = rel.target_id,
        depth        = 1,
        rel_type     = rel.type,
        relevance    = relevance,
        path         = [primary_id, rel.target_id],
      ))

  while queue is not empty and len(results) < max_objects:
    node = queue.pop_max_relevance()
    if node.knowledge_id in visited: continue
    if node.depth > max_depth: continue

    visited.add(node.knowledge_id)
    results.append(TraversalResult(
      knowledge_id  = node.knowledge_id,
      depth         = node.depth,
      path          = node.path,
      relevance     = node.relevance,
      rel_type      = node.rel_type,
    ))

    // Continue traversal if depth < max_depth and budget allows
    if node.depth < max_depth and BUDGET_PERMITS(len(results), budget):
      child_rels = LOAD_RELATIONSHIPS(node.knowledge_id)
      for rel in FILTER_BY_DIRECTION(child_rels, direction):
        if rel.target_id not in visited:
          child_relevance = PROPAGATE_RELEVANCE(node.relevance, rel)
          queue.push(TraversalNode(
            knowledge_id = rel.target_id,
            depth        = node.depth + 1,
            rel_type     = rel.type,
            relevance    = child_relevance,
            path         = node.path + [rel.target_id],
          ))

  return SORT_BY_RELEVANCE_DESC(results)
```

---

## Relevance Scoring for Graph Nodes

```
SCORE_RELEVANCE(relationship, intent_profile, query_profile) → float:

  // Relationship type weight by intent
  type_weight = {
    (TRACEABILITY, REQUIRES):     0.95,
    (IMPACT_ANALYSIS, IMPACTS):   0.95,
    (CODE_GENERATION, IMPLEMENTS): 0.90,
    (CODE_GENERATION, EXTENDS):   0.85,
    (FACTUAL_LOOKUP, PROVIDES):   0.70,
    (COMPARISON, ANY):            0.80,
    (ANY, RELATED):               0.50,
    (ANY, BACKGROUND):            0.30,
  }.get((intent_profile.intent_type, relationship.type), 0.40)

  // Depth penalty: each hop reduces relevance
  depth_penalty = 0.20 * (relationship.depth - 1)

  // Entity match bonus: if the target is explicitly mentioned in query
  entity_bonus = 0.25 if relationship.target_id in query_profile.entity_mentions else 0.0

  // Memory bonus: in-memory objects are "free" — slight preference
  memory_bonus = 0.10 if MEMORY.contains(relationship.target_id) else 0.0

  return min(1.0, type_weight - depth_penalty + entity_bonus + memory_bonus)
```

---

## TraversalResult Schema

```yaml
TraversalResult:
  knowledge_id: string
  depth: int                      # 1–4
  path: list[str]                 # [primary_id, ..., knowledge_id]
  relevance: float                # 0.0–1.0
  rel_type: string                # REQUIRES | IMPACTS | IMPLEMENTS | EXTENDS | etc.
  estimated_tokens: int           # estimated for budget planning
  in_memory: bool                 # already sent in this session
```

---

## Graph Traversal Limits

```
Hard limits (never exceeded):
  max_depth:   4 hops (UI-02 via Query Classifier cap)
  max_objects: 15 per pack (prevents budget exhaustion)
  max_latency: 20ms for traversal phase alone (P99)

Soft limits (override with explicit consumer request):
  typical depth: 1–2 for most intents
  typical max_objects: 3–8 for most packs
```

---

## Rules

| Rule | Statement |
|------|-----------|
| UICE-036 | Traversal depth must not exceed query_profile.hop_depth (capped at 4) |
| UICE-037 | FACTUAL_LOOKUP + IDENTITY performs zero traversal — primary object only |
| UICE-038 | Traversal relevance drops monotonically with depth; deeper objects are never scored higher |
| UICE-039 | In-memory objects (Conversation Memory) receive a 0.10 relevance bonus |
| UICE-040 | Graph traversal must complete within 20ms P99; exceeding this triggers Semantic Cache fallback |

---

## Cross-References

- Query Classifier → `02-QUERY-CLASSIFIER` (hop_depth source)
- Context Ranking Engine → `11-CONTEXT-RANKING-ENGINE`
- Dynamic Context Planner → `04-DYNAMIC-CONTEXT-PLANNER`
- Conversation Memory → `16-CONVERSATION-MEMORY`
- Semantic Cache → `15-SEMANTIC-CACHE`
