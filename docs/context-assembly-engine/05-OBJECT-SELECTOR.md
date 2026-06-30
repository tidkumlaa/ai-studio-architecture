# CAE-DOC-005 — Object Selector

**Phase:** 3.0D.1
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Object Selector takes an IntentRecord and returns a candidate set of
KnowledgeObjects that are potentially relevant to the query. It is responsible
for recall — getting all objects that COULD be relevant into the candidate set
before the Relevance Ranker narrows them down.

---

## Selection Modes

| Mode | When Used | Sources |
|------|-----------|---------|
| PRIMARY_ONLY | INT-01 with high-confidence entity resolution | IDX-001 direct lookup |
| PRIMARY + GRAPH | INT-02, INT-10 | IDX-001 + graph traversal |
| PRIMARY + CORTEX | INT-03 | IDX-001 + cortex-referenced objects |
| PRIMARY + IMPACT_GRAPH | INT-04 | IDX-001 + reasoning.impacts chain |
| DUAL_PRIMARY + DIFF | INT-05 | IDX-001 × 2 + diff objects |
| MULTI_PRIMARY | INT-06 SEARCH | IDX-009 capability search |
| PRIMARY + AUDIENCE | INT-07 | IDX-001 + audience explanation objects |
| PRIMARY + FULL_CONTEXT | INT-08 | IDX-001 + all ALWAYS prefetch targets |
| CHAIN | INT-09 TRACEABILITY | traceability chain traversal |
| DEPENDENCY_TREE | INT-10 | dependency graph traversal |
| PRIMARY + EVOLUTION | INT-11 | IDX-001 + evolution.predecessors |
| PRIMARY + RULES | INT-12 | IDX-001 + executable rules objects |

---

## Selection Algorithm

```
SELECT(intent_record) → CandidateSet:

Phase 1: Primary Resolution
──────────────────────────
  primary = RESOLVE(intent_record.entities.primary_entity)
  IF primary is null:
    RAISE AssemblyError(OBJECT_NOT_FOUND)

Phase 2: Mode-Specific Expansion
─────────────────────────────────
  SELECT mode based on intent_record.primary_intent:

  case PRIMARY_ONLY:
    candidates = {primary}

  case PRIMARY + GRAPH:
    candidates = {primary}
    candidates += graph.neighbors(primary, max_hops=2, min_strength=HARD)
    candidates += primary.genome.relationships.provides_to[:5]

  case PRIMARY + CORTEX:
    candidates = {primary}
    candidates += RESOLVE_ALL(primary.cortex.decision_history[*].decision_id)
    candidates += RESOLVE_ALL(primary.tradeoffs[*] .options[*].knowledge_id)

  case PRIMARY + IMPACT_GRAPH:
    candidates = {primary}
    for impact in primary.reasoning.impacts:
      candidates += RESOLVE(impact.knowledge_id)
      if impact.severity in {CRITICAL, HIGH}:
        # one more hop for critical impacts
        candidates += graph.neighbors(impact.knowledge_id, max_hops=1)

  case DUAL_PRIMARY + DIFF:
    p1 = RESOLVE(intent_record.entities.primary_entity)
    p2 = RESOLVE(intent_record.entities.secondary_entities[0])
    candidates = {p1, p2}
    candidates += RESOLVE_ALL(p1.alternatives[*].knowledge_id)

  case MULTI_PRIMARY (SEARCH):
    candidates = SEARCH(intent_record.entities.primary_entity.raw_text)
    # returns ranked list; selector returns top 20 for ranker to score

  case PRIMARY + FULL_CONTEXT:
    candidates = {primary}
    for obj in primary.ai_context.related_objects:
      if obj.fetch_priority == ALWAYS:
        candidates += RESOLVE(obj.knowledge_id)
      elif obj.fetch_priority == LIKELY and len(candidates) < 8:
        candidates += RESOLVE(obj.knowledge_id)
    candidates += PREFETCH_CACHE.get(primary.knowledge_id, [])

  case CHAIN (TRACEABILITY):
    chain = TRACE_CHAIN(primary)
    candidates = chain.all_nodes

  case DEPENDENCY_TREE:
    tree = BUILD_DEP_TREE(primary, max_depth=3)
    candidates = tree.all_nodes

Phase 3: Always-Include Objects
────────────────────────────────
  For every primary object:
    candidates += primary.ai_context.related_objects
                    WHERE fetch_priority == ALWAYS

  For comparison (INT-05):
    candidates += primary.alternatives
                    WHERE similarity_to_this > 0.70

Phase 4: Deduplication
──────────────────────
  candidates = deduplicate(candidates, key=knowledge_id)

Phase 5: AIRS Filter
────────────────────
  IF pack_type == PROMPT:
    candidates = filter(candidates, ai_readiness_score >= 0.60)

  RETURN candidates
```

---

## Graph Traversal Limits

| Parameter | Value | Reason |
|-----------|-------|--------|
| max_hops for GRAPH mode | 2 | Beyond 2 hops, relevance decays sharply |
| max_hops for IMPACT_ANALYSIS | 3 | Critical impacts cascade |
| max_dependency_depth | 3 | Full dep tree; deeper rarely needed |
| max_candidates before ranking | 50 | Ranking cap |
| min_strength for HARD graph | HARD | Soft dependencies excluded unless budget allows |

---

## Candidate Object Record

```yaml
candidate_object:
  knowledge_id: string
  selection_source: DIRECT | GRAPH | CORTEX | IMPACT | COMPARISON |
                    PREFETCH | CHAIN | ALWAYS_INCLUDE | SEARCH
  selection_hop: integer               # 0 = primary, 1 = direct neighbor, etc.
  preliminary_score: float             # pre-ranking relevance estimate
  resolution_method: EXACT | SEMANTIC | ALIAS | INFERRED
```

---

## Rules

| Rule | Statement |
|------|-----------|
| CAE-016 | Primary object must always be in candidate set — it is never filtered out |
| CAE-017 | Candidates with `state: DEPRECATED` or `state: ARCHIVED` are excluded unless INT-11 |
| CAE-018 | Maximum candidate set size is 50 — beyond this, ranker performance degrades |
| CAE-019 | Objects with `ai_readiness_score < 0.60` excluded from PromptPack candidates |
| CAE-020 | ALWAYS-INCLUDE objects bypass the AIRS filter |

---

## Cross-References

- Intent model → `03-INTENT-MODEL`
- Relevance ranking → `06-RELEVANCE-MODEL`
- KIL Canonical Index → Phase 3.0D.0.6 `29-KNOWLEDGE-CANONICAL-INDEX`
- Prefetch engine → `19-PREFETCH-ENGINE`
