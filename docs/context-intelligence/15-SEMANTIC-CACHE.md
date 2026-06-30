# UICE-DOC-015 — Semantic Cache

**Phase:** 3.0D.2
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Semantic Cache caches compiled context packs indexed by **query embedding**
rather than exact key match. When a new query is semantically equivalent to a
recent query (same intent, same entities, same information need) — even if the
wording differs — the Semantic Cache returns the previous pack without re-assembling.

This enables the "cache-first" policy (UI-04): Semantic Cache lookup precedes
graph traversal, preventing the most expensive pipeline stages from running
unnecessarily for semantically repeated queries.

---

## Semantic Similarity Model

```
UICE Semantic Cache uses embedding-based similarity:

  Embedding:  a dense vector representation of the query
  Model:      lightweight text embedding (not an LLM)
              dimension = 256 (compact; fits in memory for 100K entries)
  Similarity: cosine similarity

  HIT threshold:   cosine_similarity ≥ 0.95
  SOFT HIT:        cosine_similarity ≥ 0.85 (return hit but flag as soft)
  MISS:            cosine_similarity < 0.85
```

The embedding model is part of the UICE runtime; it is NOT a KOS architecture
component (no runtime implementation in this phase).

---

## SemanticCacheEntry Schema

```yaml
SemanticCacheEntry:
  entry_id: string
  query_text: string          # canonical query text that produced this entry
  query_embedding: list[float]  # 256-dimensional embedding vector
  intent_type: IntentType
  query_type: QueryType
  primary_id: string
  context_pack: UICEContextPack  # the compiled pack

  created_at: datetime
  ttl_seconds: int
  hit_count: int
  last_hit_at: datetime | null

  # Eviction metadata
  soft_hit_warnings: int      # incremented on soft hits
```

---

## Semantic Cache Lookup Algorithm

```
SEMANTIC_CACHE_LOOKUP(query_text, intent_type, query_type) → CacheResult:

  query_embedding = EMBED(query_text)  // fast, < 5ms

  // Step 1: Filter by intent+query type (reduces search space)
  candidates = SEMANTIC_INDEX.filter(
    intent_type = intent_type,
    query_type  = query_type,
  )

  // Step 2: Approximate nearest neighbor search
  best_match = ANN_SEARCH(
    query_embedding = query_embedding,
    candidates      = candidates,
    top_k           = 1,
  )

  if best_match is None:
    return CacheMiss()

  similarity = COSINE_SIMILARITY(query_embedding, best_match.query_embedding)

  if similarity >= 0.95:
    RECORD_HIT(semantic=True, soft=False)
    return CacheHit(pack=best_match.context_pack, similarity=similarity, soft=False)

  elif similarity >= 0.85:
    RECORD_HIT(semantic=True, soft=True)
    return CacheHit(pack=best_match.context_pack, similarity=similarity, soft=True)

  return CacheMiss()
```

---

## Soft Hit Handling

```
A soft hit (0.85 ≤ similarity < 0.95) may be returned with a staleness warning:

  soft_hit_response = {
    pack:        cached_pack,
    soft_hit:    True,
    similarity:  0.87,
    warning:     "Context retrieved from semantic cache (87% similarity). "
                 "Verify if this query has different intent nuance.",
  }

Soft hits increment soft_hit_warnings counter on the entry.
If soft_hit_warnings >= 3, the entry is demoted (ANN search excludes it for 60s).
```

---

## Semantic Cache Invalidation

```
Invalidation events:

1. Primary object version change:
   All semantic cache entries where primary_id == changed_id are invalidated.

2. TTL expiry:
   Entries expire based on object state (same TTL rules as Context Cache).

3. Similarity drift:
   After 10 soft hits, the entry is proactively re-built with current data
   and the old entry is replaced.

4. Explicit flush:
   Admin operation — clears all entries for a primary_id or all entries.
```

---

## Cache Capacity and ANN Index

```
Maximum entries: 50,000 per UICE instance
Index type:      HNSW (Hierarchical Navigable Small World)
                 — approximate nearest neighbor; O(log N) search
Index rebuild:   incremental (no full rebuild required)
Memory:          50,000 × 256 dims × 4 bytes = ~51MB per index

Lookup latency:
  Embedding computation: < 5ms
  ANN search: < 3ms (50K entries)
  Total: < 8ms (well within 20ms traversal budget per UICE-040)
```

---

## Semantic Cache vs Context Cache

```
Context Cache (module 14):
  Key: exact {knowledge_id + intent_type + query_type + model_id}
  Returns: compiled slice or full pack for EXACT same query parameters
  Hit rate target: ≥ 85% in steady state

Semantic Cache (module 15):
  Key: query embedding (semantic meaning)
  Returns: full pack for SEMANTICALLY EQUIVALENT queries
  Hit rate target: ≥ 40% for conversational queries

Combined pipeline:
  Semantic Cache → (miss) → Context Cache → (miss) → Graph Traversal → Assembly
```

---

## Rules

| Rule | Statement |
|------|-----------|
| UICE-071 | Semantic Cache lookup must run before graph traversal (UI-04 — cache-first policy) |
| UICE-072 | Hard hit threshold is cosine_similarity ≥ 0.95; anything below this is not a guaranteed hit |
| UICE-073 | Soft hits must be flagged to the consumer; they may not be returned as if they were exact hits |
| UICE-074 | Semantic Cache capacity is bounded at 50,000 entries; LRU eviction applies beyond this limit |
| UICE-075 | Embedding computation is a runtime concern; this spec defines the interface only — not the model |

---

## Cross-References

- Context Cache → `14-CONTEXT-CACHE`
- Context Graph Traversal → `08-CONTEXT-GRAPH-TRAVERSAL` (bypassed on hit)
- Context Reuse Engine → `17-CONTEXT-REUSE-ENGINE`
- UICE invariant UI-04 → `README.md`
