# UICE-DOC-017 — Context Reuse Engine

**Phase:** 3.0D.2
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Context Reuse Engine identifies context blocks that appear frequently across
many different queries and pre-computes them for indefinite cache storage.
Where the Semantic Cache caches full packs per query, the Reuse Engine caches
individual object slices that are reused across many distinct packs.

A high-frequency pattern: the same 10 platform services appear in 80% of all
context packs. Reuse Engine pre-computes their FLA slices for all 12 intent
types and stores them permanently — bypassing FLA computation on cache hit.

---

## ReusableBlock Schema

```yaml
ReusableBlock:
  block_id: string                    # sha256(knowledge_id + intent_type + query_type)
  knowledge_id: string
  intent_type: IntentType
  query_type: QueryType

  # Precomputed content
  fla_slice: FieldLevelSlice
  summary_micro: SummarySlice         # MICRO mode pre-computed
  summary_mini: SummarySlice          # MINI mode pre-computed

  # Usage stats
  access_count: int
  last_accessed_at: datetime
  promotion_score: float              # access_count / session_count (≥0.20 for promotion)
  is_active: bool

  # Lifecycle
  computed_at: datetime
  object_version_at_compute: string   # invalidate when version changes
  ttl: PERMANENT                      # reuse blocks have no TTL (invalidated by version change only)
```

---

## Promotion Algorithm

```
Objects are promoted to Reuse Engine when they appear frequently:

PROMOTION_CRITERIA:
  promotion_score = access_count / total_session_count >= 0.20
    (appears in ≥ 20% of all sessions)
  AND
  object_status == CANONICAL

PROMOTION_PROCESS (runs daily, async):
  candidates = FIND_HIGH_FREQUENCY_OBJECTS(threshold=0.20)
  for candidate in candidates:
    for intent_type in IntentType:
      for query_type in QueryType:
        if not is_l5_full(intent_type, query_type):
          fla_slice  = FLA_MODULE.select(candidate, intent_type, query_type)
          micro = SUMMARIZER.summarize(candidate, intent_type, query_type, budget=30)
          mini  = SUMMARIZER.summarize(candidate, intent_type, query_type, budget=80)
          REUSE_ENGINE.store(ReusableBlock(
            knowledge_id = candidate.knowledge_id,
            intent_type  = intent_type,
            query_type   = query_type,
            fla_slice    = fla_slice,
            summary_micro = micro,
            summary_mini  = mini,
            promotion_score = CURRENT_SCORE(candidate),
          ))

Total blocks per high-frequency object: 12 × 10 = 120 blocks (one per combination)
```

---

## Reuse Lookup

```
REUSE_LOOKUP(knowledge_id, intent_type, query_type, mode) → Slice | MISS:

  block_id = sha256(knowledge_id + intent_type + query_type)
  block = REUSE_STORE.get(block_id)

  if block is None:
    return MISS
  if not block.is_active:
    return MISS
  if block.object_version_at_compute != GET_CURRENT_VERSION(knowledge_id):
    INVALIDATE(block)
    return MISS

  block.access_count += 1
  block.last_accessed_at = NOW()

  match mode:
    FLA:           return block.fla_slice
    SUMMARY_MICRO: return block.summary_micro
    SUMMARY_MINI:  return block.summary_mini
    _:             return MISS   // DELTA, FULL, OMIT handled elsewhere
```

---

## Reuse Engine Lookup Position in Pipeline

```
Pipeline position: after Cache hit check, before FLA computation.

Full lookup order:
  1. Semantic Cache      → full pack hit
  2. Context Cache (L1)  → slice hit
  3. Context Cache (L2)  → slice hit
  4. Reuse Engine        → precomputed FLA/SUMMARY slice
  5. FLA computation     → fresh computation
  6. Graph traversal     → fresh traversal + FLA
```

---

## Invalidation

```
Invalidation triggers:
  - Object version change: all blocks for that knowledge_id are invalidated
  - Object state change to DEPRECATED: FLA blocks kept for MIGRATION_GUIDE;
    other intent blocks invalidated
  - Promotion score drops below 0.10: block is demoted (is_active = False)
    and not re-computed until score recovers

Invalidation is immediate for version changes (consistency requirement).
Demotion from low score runs in background (async, daily).
```

---

## Expected Coverage

```
After 30 days of production traffic:
  High-frequency objects (top 100): fully pre-computed (12,000 blocks)
  Medium-frequency objects (top 500): FLA pre-computed (60,000 blocks)
  Low-frequency objects: no pre-computation (computed fresh)

Expected reuse hit rate: 60–70% for high-frequency object queries
Expected latency saving: 5–15ms per reused slice (FLA + graph traversal avoided)
```

---

## Rules

| Rule | Statement |
|------|-----------|
| UICE-081 | Promotion threshold is 20% of sessions; objects below this threshold are not pre-computed |
| UICE-082 | Reuse blocks have no TTL — they are valid until the object version changes |
| UICE-083 | Reuse Engine lookup is attempted after cache misses, before FLA computation |
| UICE-084 | FULL (CONTEXT_PRODUCTION) slices are never pre-computed in Reuse Engine |
| UICE-085 | Promotion computation runs asynchronously; it must not block the request pipeline |

---

## Cross-References

- Field-Level Assembly → `03-FIELD-LEVEL-ASSEMBLY`
- Context Summarizer → `13-CONTEXT-SUMMARIZER`
- Context Cache → `14-CONTEXT-CACHE`
- Semantic Cache → `15-SEMANTIC-CACHE`
- Context Evolution → `18-CONTEXT-EVOLUTION`
