# UICE-DOC-014 — Context Cache

**Phase:** 3.0D.2
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Context Cache stores compiled context packs and field-level slices so that
repeated queries about the same objects avoid expensive re-assembly. UICE extends
the CAE two-tier L1/L2 cache with intent- and query-type awareness: the cache key
includes both the object identity and the assembly context, ensuring that
different intents about the same object produce different cache entries.

---

## Two-Tier Cache Architecture

```
L1 Cache (in-memory, per-process):
  Capacity:     1,000 entries per process
  TTL:          60 seconds (wall-clock)
  Key:          uice:l1:{knowledge_id}:{intent_type}:{query_type}:{model_id}
  Eviction:     LRU (Least Recently Used)
  Latency:      < 0.1ms lookup

L2 Cache (persistent, shared):
  Capacity:     100,000 entries per shard
  TTL:          by object lifecycle state
  Key:          uice:l2:{sha256(knowledge_id+intent_type+query_type+model_id+schema_version)}
  Eviction:     LFU (Least Frequently Used) + TTL expiry
  Latency:      < 2ms lookup

Full pack cache:
  Key:          uice:pack:{sha256(primary_id+intent+query+model+session_id)}
  TTL:          30 seconds (conversation-scoped)
  Invalidated:  when primary_id version changes
```

---

## Cache Key Schema

```
GENERATE_CACHE_KEY(knowledge_id, intent_type, query_type, model_id,
                   schema_version, level) → string:

  component_hash = sha256(
    knowledge_id   + "|" +
    intent_type    + "|" +
    query_type     + "|" +
    model_id       + "|" +
    schema_version
  )[:16]

  if level == L1:
    return f"uice:l1:{knowledge_id}:{intent_type}:{query_type}:{model_id}"
  else:
    return f"uice:l2:{component_hash}"
```

---

## TTL by Object State

```
CANONICAL active (updated < 7 days):    TTL = 300s  (5 minutes)
CANONICAL active (updated < 30 days):   TTL = 1800s (30 minutes)
CANONICAL active (updated ≥ 30 days):   TTL = 7200s (2 hours)
CANONICAL deprecated:                   TTL = 86400s (24 hours)
DRAFT:                                  TTL = 30s   (volatile)
REVIEW:                                 TTL = 60s
ARCHIVED:                               TTL = 604800s (7 days)
```

---

## Cache Lookup Algorithm

```
CACHE_LOOKUP(knowledge_id, intent_type, query_type, model_id) → Slice | MISS:

  // L1 first (fastest)
  l1_key = GENERATE_CACHE_KEY(..., L1)
  entry = L1_CACHE.get(l1_key)
  if entry is not None and not IS_EXPIRED(entry):
    RECORD_HIT(L1)
    return entry.slice

  // L2 second
  l2_key = GENERATE_CACHE_KEY(..., L2)
  entry = L2_CACHE.get(l2_key)
  if entry is not None and not IS_EXPIRED(entry) and not IS_INVALIDATED(entry):
    // Promote to L1
    L1_CACHE.set(l1_key, entry, TTL=60)
    RECORD_HIT(L2)
    return entry.slice

  RECORD_MISS()
  return MISS


CACHE_STORE(knowledge_id, intent_type, query_type, model_id, slice, object_state):

  ttl = GET_TTL(object_state)
  l1_key = GENERATE_CACHE_KEY(..., L1)
  l2_key = GENERATE_CACHE_KEY(..., L2)

  entry = CacheEntry(
    key           = l2_key,
    slice         = slice,
    knowledge_id  = knowledge_id,
    intent_type   = intent_type,
    query_type    = query_type,
    model_id      = model_id,
    stored_at     = NOW(),
    ttl_seconds   = ttl,
    schema_version = CURRENT_SCHEMA_VERSION,
  )

  L1_CACHE.set(l1_key, entry, TTL=min(ttl, 60))
  L2_CACHE.set(l2_key, entry, TTL=ttl)
```

---

## Cache Invalidation

```
INVALIDATION TRIGGERS:

1. Version change:
   When a KnowledgeObject version changes, all cache entries for that
   knowledge_id are immediately invalidated (all intents, all query types).
   Key pattern: uice:l1:{knowledge_id}:* and uice:l2:{hash containing knowledge_id}

2. Schema version change:
   When schema_version increments, all L2 entries with previous schema version
   are invalidated. L1 clears automatically via TTL.

3. State change:
   DRAFT → REVIEW: no invalidation
   REVIEW → CANONICAL: invalidate DRAFT entries for this object
   CANONICAL → DEPRECATED: invalidate and re-cache with deprecated TTL
   DEPRECATED → ARCHIVED: invalidate all non-migration entries

4. Explicit invalidation:
   Admin operation — invalidates specific key or all entries for an object.
```

---

## Cache Metrics

```
Tracked per cache tier:
  hit_count, miss_count, hit_rate
  eviction_count, expiry_count
  mean_lookup_latency_ms
  total_entries, memory_usage_bytes

Target:
  L1 hit rate: ≥ 60% for repeated queries in same session
  L2 hit rate: ≥ 80% for warm cache steady state
  Combined: ≥ 85% hit rate in production
```

---

## Rules

| Rule | Statement |
|------|-----------|
| UICE-066 | Cache key must include intent_type and query_type; intent-blind keys cause wrong-context hits |
| UICE-067 | L1 TTL is always ≤ 60 seconds regardless of object state |
| UICE-068 | Version change invalidates ALL cache entries for that knowledge_id across all intent/query combinations |
| UICE-069 | DRAFT objects have TTL = 30s; do not cache DRAFT objects in L2 |
| UICE-070 | Cache lookup must be attempted before any FLA, SCO, or graph traversal work begins |

---

## Cross-References

- Semantic Cache → `15-SEMANTIC-CACHE`
- Context Reuse Engine → `17-CONTEXT-REUSE-ENGINE`
- Context Evolution → `18-CONTEXT-EVOLUTION`
- CAE Context Cache → Phase 3.0D.1 `20-CONTEXT-CACHE`
- Context Metrics → `20-CONTEXT-METRICS`
