# KNW-KIL-DOC-024 — Knowledge Optimization

**Phase:** 3.0D.0.6
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Every Knowledge Object carries optimization hints — guidance for the Knowledge
Runtime on how to cache, index, pre-fetch, and serve this object most efficiently.

Optimization is separated from the object itself because optimal access patterns
depend on runtime context (query patterns, access frequency, token budgets) that
are known at deployment time, not authoring time.

This document defines `intelligence.optimization`.

---

## Schema

```yaml
intelligence:
  optimization:
    schema_version: "1.0"

    caching:
      cacheable: boolean               # should this object be cached?
      cache_strategy: LRU | LFU | TTL | PERSISTENT | NEVER
      ttl_seconds: integer | null      # for TTL strategy
      cache_key_fields: [string]       # which fields form the cache key
      invalidation_triggers: [string]  # events that invalidate the cache
      cache_priority: HIGH | MEDIUM | LOW  # priority when cache is full

    indexing:
      indexed_fields:                  # fields to index for search
        - field_path: string
          index_type: EXACT | PREFIX | FULLTEXT | VECTOR | GRAPH
          priority: PRIMARY | SECONDARY
      vector_embedding:                # fields to embed for semantic search
        source_fields: [string]        # which fields to include in embedding
        embedding_model: string        # e.g. "text-embedding-3-small"
        dimensions: integer
        recompute_on: [string]         # which field changes trigger re-embedding

    prefetch:
      auto_prefetch: boolean           # should related objects be pre-fetched?
      prefetch_depth: integer          # how many hops to pre-fetch
      prefetch_targets:                # specific objects to always pre-fetch
        - knowledge_id: string
          priority: ALWAYS | LIKELY | OPTIONAL
          compression_level: string    # which level to pre-fetch at

    access_patterns:
      hot_fields: [string]             # most frequently accessed fields
      cold_fields: [string]            # rarely accessed (lazy-load candidates)
      batch_access: boolean            # is this object often accessed in batches?
      concurrent_access: boolean       # accessed by multiple agents simultaneously?
      read_write_ratio: string         # e.g. "100:1" (mostly read)

    compression_preference:
      default_level: string            # L1-L5 — default compression for this object
      agent_level: string              # level to use for AI agent access
      search_level: string             # level to use in search results
```

---

## Canonical Example

```yaml
# For: KNW-PLT-MOD-001 (Quota Manager)
intelligence:
  optimization:
    schema_version: "1.0"

    caching:
      cacheable: true
      cache_strategy: LRU
      ttl_seconds: 3600                # 1 hour — genome/DNA stable
      cache_key_fields:
        - "knowledge_id"
        - "metadata.version"
      invalidation_triggers:
        - "lifecycle.state.changed"
        - "evidence.items.updated"
        - "metadata.quality_score.changed"
      cache_priority: HIGH

    indexing:
      indexed_fields:
        - field_path: "knowledge_id"
          index_type: EXACT
          priority: PRIMARY
        - field_path: "metadata.canonical_name"
          index_type: EXACT
          priority: PRIMARY
        - field_path: "intelligence.ai_context.short_summary"
          index_type: FULLTEXT
          priority: SECONDARY
        - field_path: "intelligence.semantic.capabilities[*].name"
          index_type: FULLTEXT
          priority: SECONDARY
      vector_embedding:
        source_fields:
          - "intelligence.ai_context.medium_summary"
          - "intelligence.semantic.capabilities[*].description"
          - "intelligence.cortex.why.statement"
        embedding_model: "text-embedding-3-small"
        dimensions: 1536
        recompute_on:
          - "intelligence.ai_context.medium_summary"
          - "intelligence.semantic.capabilities"

    prefetch:
      auto_prefetch: true
      prefetch_depth: 1                # only immediate neighbors
      prefetch_targets:
        - knowledge_id: KNW-PLT-REQ-001
          priority: ALWAYS
          compression_level: L3
        - knowledge_id: KNW-RT-RT-001
          priority: ALWAYS
          compression_level: L3
        - knowledge_id: KNW-META-PAT-003
          priority: OPTIONAL
          compression_level: L2

    access_patterns:
      hot_fields:
        - "intelligence.ai_context.short_summary"
        - "intelligence.compression.nano"
        - "intelligence.compression.micro"
        - "intelligence.dna.purpose.statement"
        - "intelligence.dna.core_behavior"
      cold_fields:
        - "intelligence.evolution.changes"
        - "intelligence.history.full_log"
        - "evidence.items[*].raw_data"
      batch_access: false
      concurrent_access: true
      read_write_ratio: "1000:1"

    compression_preference:
      default_level: L3
      agent_level: L3
      search_level: L2
```

---

## Optimization Strategy by Object Type

| Object Type | Default Cache | Default Index | Prefetch Depth |
|-------------|--------------|--------------|---------------|
| MODULE | LRU 1h | EXACT + VECTOR | 1 |
| SERVICE | LRU 1h | EXACT + VECTOR | 1 |
| REQUIREMENT | LRU 24h | EXACT + FULLTEXT | 0 |
| ALGORITHM | LRU 24h | EXACT + VECTOR | 0 |
| TEST | TTL 30m | EXACT | 0 |
| BENCHMARK | TTL 30m | EXACT | 0 |
| DECISION | PERSISTENT | EXACT + FULLTEXT | 0 |
| META | PERSISTENT | EXACT | 0 |

---

## Rules

| Rule | Statement |
|------|-----------|
| KIL-131 | Objects with `cache_priority: HIGH` must have `cacheable: true` |
| KIL-132 | Every `vector_embedding` must specify `recompute_on` — no stale embeddings |
| KIL-133 | `prefetch_targets` must be a subset of `ai_context.related_objects` |
| KIL-134 | `hot_fields` must include at least `compression.nano` and `ai_context.short_summary` |

---

## Cross-References

- AI context related objects → `04-AI-CONTEXT-LAYER`
- Search optimization → `28-KNOWLEDGE-SEARCH-OPTIMIZATION`
- Canonical index → `29-KNOWLEDGE-CANONICAL-INDEX`
- Memory (access patterns) → `09-KNOWLEDGE-MEMORY`
