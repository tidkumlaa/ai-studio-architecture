# CAE-DOC-020 — Context Cache

**Phase:** 3.0D.1
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Context Cache stores rendered object representations at each compression
level to avoid re-rendering on repeated access. Cache hits reduce assembly
latency; the cache is correctness-safe by design — a stale cache entry is
never used after the source object is updated.

---

## Cache Architecture

```
Two-tier cache:

Tier 1 — In-Process (Hot Cache)
  Type: LRU in-memory map
  Capacity: top 1,000 objects × 5 levels = 5,000 entries max
  TTL: 5 minutes for CANONICAL objects; 1 minute for EXPERIMENTAL/DRAFT
  Hit latency target: < 1ms

Tier 2 — Shared Cache (Warm Cache)
  Type: key-value store (e.g., Redis)
  Capacity: top 10,000 objects × 5 levels
  TTL: computed per object (see TTL Calculation)
  Hit latency target: < 5ms

L1 miss → check L2 → L2 miss → render from KIL index → populate both tiers
```

---

## Cache Key Format

```
cache_key(knowledge_id, level, query_type_modifier):

  base_key = f"cae:render:{knowledge_id}:{level}"

  # Query-type-specific renders are cached separately because they include
  # additional fields (see doc 08 query-type augmentations)
  IF query_type_modifier is not null:
    return f"{base_key}:{query_type_modifier}"
  ELSE:
    return base_key

Examples:
  "cae:render:KNW-PLT-MOD-001:L3"
  "cae:render:KNW-PLT-MOD-001:L4:QT-06"
  "cae:render:KNW-PLT-MOD-001:L5:QT-04"
```

---

## TTL Calculation

```
compute_ttl(index_entry):

  state = index_entry.metadata.state

  CASE state:
    CANONICAL:   base_ttl = 3600   # 1 hour
    ACTIVE:      base_ttl = 1800   # 30 minutes
    EXPERIMENTAL: base_ttl = 300   # 5 minutes
    DRAFT:       base_ttl = 60     # 1 minute
    DEPRECATED:  base_ttl = 86400  # 24 hours (rarely changes)

  # Freshness adjustment: objects updated recently get shorter TTL
  age_hours = (now - index_entry.metadata.updated_at).hours
  IF age_hours < 1:
    return min(base_ttl, 300)      # recently updated: 5 min max
  ELIF age_hours < 24:
    return min(base_ttl, 1800)     # today: 30 min max
  ELSE:
    return base_ttl
```

---

## Cache Invalidation Protocol

```
ON KnowledgeObject.updated(knowledge_id):
  # Invalidate all levels for this object
  FOR level in [L1, L2, L3, L4, L5]:
    FOR qt in QUERY_TYPE_MODIFIERS + [null]:
      key = cache_key(knowledge_id, level, qt)
      CACHE_L1.delete(key)
      CACHE_L2.delete(key)

  LOG: f"Cache invalidated for {knowledge_id}: all levels"

ON KnowledgeObject.relationship_changed(knowledge_id):
  # Relationship changes affect graph_proximity scores
  # Invalidate graph-dependent renders (L4+)
  FOR level in [L4, L5]:
    CACHE_L1.delete(cache_key(knowledge_id, level, null))
    CACHE_L2.delete(cache_key(knowledge_id, level, null))
```

---

## Cache Miss Handling

```
GET_RENDERED(knowledge_id, level, query_type):

  key = cache_key(knowledge_id, level, query_type)

  # L1 check
  entry = CACHE_L1.get(key)
  IF entry and not entry.is_expired():
    METRICS.increment("cache.hit.l1")
    RETURN entry.value

  # L2 check
  entry = CACHE_L2.get(key)
  IF entry and not entry.is_expired():
    METRICS.increment("cache.hit.l2")
    CACHE_L1.put(key, entry.value, ttl=min(entry.remaining_ttl, 300))
    RETURN entry.value

  # Cache miss — render
  METRICS.increment("cache.miss")
  index_entry = KIL_INDEX.get(knowledge_id)
  IF index_entry is null:
    RAISE CacheMissError(f"Object {knowledge_id} not in index")

  rendered = RENDER(index_entry, level, query_type)
  ttl = compute_ttl(index_entry)

  CACHE_L1.put(key, rendered, ttl=min(ttl, 300))
  CACHE_L2.put(key, rendered, ttl=ttl)

  RETURN rendered
```

---

## Cache Entry Schema

```yaml
cache_entry:
  key: string
  knowledge_id: string
  level: string
  query_type_modifier: string | null
  value: string                       # rendered content
  rendered_at: string                 # ISO 8601
  source_version: string              # object version at render time
  ttl_seconds: integer
  expiry_at: string                   # rendered_at + ttl
```

---

## Cache Metrics

| Metric | Target |
|--------|--------|
| L1 hit rate | ≥ 60% |
| L2 hit rate | ≥ 80% (including L1 hits) |
| Combined hit rate | ≥ 85% |
| Cache invalidation latency | < 100ms |
| TTL computation overhead | < 1ms |
| Stale entry served | 0 |

---

## Rules

| Rule | Statement |
|------|-----------|
| CAE-091 | A stale cache entry (TTL expired or post-invalidation) must never be served |
| CAE-092 | Cache invalidation must complete within 100ms of object update signal |
| CAE-093 | Cache miss must render and populate both tiers — cold reads must warm the cache |
| CAE-094 | Cache keys must include compression level and query_type_modifier for correctness |
| CAE-095 | Cache entries must record `source_version` — used to detect stale-on-write |

---

## Cross-References

- Prefetch engine → `19-PREFETCH-ENGINE`
- Context assembler → `10-CONTEXT-ASSEMBLER`
- Performance model → `21-PERFORMANCE-MODEL`
- Context metrics → `23-CONTEXT-METRICS`
