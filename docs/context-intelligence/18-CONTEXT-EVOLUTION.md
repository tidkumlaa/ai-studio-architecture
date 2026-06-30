# UICE-DOC-018 — Context Evolution

**Phase:** 3.0D.2
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Context Evolution monitors KnowledgeObject changes and propagates invalidation
signals to all UICE caches and reuse blocks. When an object's content, version,
or lifecycle state changes, previously compiled context for that object may be
stale — Context Evolution detects this and triggers lazy re-assembly.

This module ensures that UICE never serves incorrect context due to stale cache
entries, even as the knowledge base evolves.

---

## Evolution Event Types

```
OBJECT_VERSION_CHANGE:
  Trigger: KnowledgeObject.version changed (semantic version bump)
  Impact:  All cache entries (L1, L2, Semantic, Reuse) for knowledge_id invalidated
  Action:  Hard invalidation; recompute on next request

OBJECT_FIELD_UPDATE:
  Trigger: KnowledgeObject fields changed, version unchanged
           (e.g., AIRS score refreshed, quality score updated)
  Impact:  Cache entries older than field_update_at are stale
  Action:  Soft invalidation; mark stale, recompute on next request

OBJECT_STATE_CHANGE:
  Trigger: KnowledgeObject lifecycle state changes
           (DRAFT→REVIEW, REVIEW→CANONICAL, CANONICAL→DEPRECATED)
  Impact:  Varies by transition (see State Transition Matrix below)
  Action:  State-specific invalidation

RELATIONSHIP_CHANGE:
  Trigger: Edges in knowledge graph added or removed for this object
  Impact:  Graph traversal cache for objects referencing this knowledge_id
  Action:  Invalidate traversal cache; recompute on next multi-hop request

BATCH_UPDATE:
  Trigger: Multiple objects updated simultaneously (e.g., package-level migration)
  Impact:  All objects in the batch
  Action:  Bulk invalidation; emit single EvolutionEvent with object_ids list
```

---

## State Transition Invalidation Matrix

```
DRAFT → REVIEW:
  Invalidate DRAFT TTL entries only; extend TTL to REVIEW TTL (60s)

REVIEW → CANONICAL:
  Invalidate all DRAFT and REVIEW entries for this object
  Trigger REUSE_ENGINE.promote_candidate() evaluation

CANONICAL → DEPRECATED:
  Invalidate all entries EXCEPT MIGRATION_GUIDE intent combinations
  (migration queries need deprecated context to guide migration)
  Set freshness_factor = 0.50 for remaining entries

DEPRECATED → ARCHIVED:
  Invalidate ALL cache entries
  Reuse Engine removes all blocks except MIGRATION_GUIDE

CANONICAL version change:
  Full invalidation of all entries for knowledge_id
```

---

## EvolutionEvent Schema

```yaml
EvolutionEvent:
  event_id: string
  event_type: OBJECT_VERSION_CHANGE | OBJECT_FIELD_UPDATE | OBJECT_STATE_CHANGE
              | RELATIONSHIP_CHANGE | BATCH_UPDATE
  knowledge_ids: list[str]       # one or many objects affected
  previous_version: string | null
  current_version: string | null
  previous_state: string | null
  current_state: string | null
  changed_fields: list[str]      # for FIELD_UPDATE events
  occurred_at: datetime
  source: KOS_REGISTRY | MANUAL | MIGRATION | IMPORT
```

---

## Evolution Handler Algorithm

```
HANDLE_EVOLUTION_EVENT(event) → EvolutionResult:

  invalidated_cache_entries = 0
  invalidated_reuse_blocks  = 0
  triggered_recompute       = 0

  for knowledge_id in event.knowledge_ids:

    match event.event_type:

      OBJECT_VERSION_CHANGE:
        // Hard invalidation: all caches, all intent/query combinations
        n = CONTEXT_CACHE.invalidate_all(knowledge_id)
        invalidated_cache_entries += n
        n = REUSE_ENGINE.invalidate_all(knowledge_id)
        invalidated_reuse_blocks  += n
        SEMANTIC_CACHE.invalidate_by_primary_id(knowledge_id)
        CONVERSATION_MEMORY.mark_version_changed(knowledge_id)

      OBJECT_FIELD_UPDATE:
        // Soft invalidation: mark stale, defer recompute
        CONTEXT_CACHE.mark_stale(knowledge_id, since=event.occurred_at)
        REUSE_ENGINE.schedule_recompute(knowledge_id)

      OBJECT_STATE_CHANGE:
        APPLY_STATE_TRANSITION_INVALIDATION(knowledge_id,
                                            event.previous_state,
                                            event.current_state)

      RELATIONSHIP_CHANGE:
        // Invalidate graph traversal results involving this object
        GRAPH_TRAVERSAL_CACHE.invalidate_involving(knowledge_id)

      BATCH_UPDATE:
        // Recursive handling for each ID
        for kid in event.knowledge_ids:
          sub_event = clone(event, knowledge_ids=[kid])
          HANDLE_EVOLUTION_EVENT(sub_event)

  return EvolutionResult(
    event_id                  = event.event_id,
    invalidated_cache_entries = invalidated_cache_entries,
    invalidated_reuse_blocks  = invalidated_reuse_blocks,
    triggered_recompute       = triggered_recompute,
    processing_ms             = ELAPSED_MS(),
  )
```

---

## Lazy Re-Assembly

```
When a cached entry is marked stale (FIELD_UPDATE event), the next request
for that object triggers fresh assembly. Stale entries are served until
fresh assembly completes (grace period = TTL remaining / 2, max 30s).

Stale entries are NEVER served for:
  - OBJECT_VERSION_CHANGE (hard invalidation — no stale service)
  - CRITICAL hallucination risk queries (context_verifier blocks)
  - PRODUCTION urgency requests (freshness required)
```

---

## Rules

| Rule | Statement |
|------|-----------|
| UICE-086 | OBJECT_VERSION_CHANGE triggers immediate hard invalidation — stale entries are never served after version change |
| UICE-087 | DEPRECATED objects retain MIGRATION_GUIDE cache entries; all other intents are invalidated |
| UICE-088 | Evolution events must be processed within 5 seconds of occurrence; delayed propagation is a bug |
| UICE-089 | Batch updates emit a single EvolutionEvent with all affected knowledge_ids — not N separate events |
| UICE-090 | Context Evolution is a subscriber to the KOS Registry event stream; it does not poll for changes |

---

## Cross-References

- Context Cache → `14-CONTEXT-CACHE`
- Semantic Cache → `15-SEMANTIC-CACHE`
- Context Reuse Engine → `17-CONTEXT-REUSE-ENGINE`
- Conversation Memory → `16-CONVERSATION-MEMORY`
