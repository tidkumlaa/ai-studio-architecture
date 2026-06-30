# KNW-KIL-DOC-029 — Knowledge Canonical Index

**Phase:** 3.0D.0.6
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Canonical Index is the master lookup table for the Knowledge Intelligence
Layer. It provides a pre-computed, structured index over every KIL-augmented
object, enabling O(1) lookup by any of the standard identifiers and O(log n)
lookup by any indexed field.

This document defines the index format, entry structure, maintenance rules,
and consistency requirements.

---

## Index Types

| Index ID | Key | Coverage | Use Case |
|----------|-----|---------|---------|
| IDX-001 | knowledge_id | All objects | Primary lookup |
| IDX-002 | canonical_name | All objects | Canonical name resolution |
| IDX-003 | object_uri | All objects | URI-based lookup |
| IDX-004 | dna_hash | All CANONICAL | Duplicate detection |
| IDX-005 | genome_fingerprint | All CANONICAL | Semantic equivalence |
| IDX-006 | namespace | All objects | Namespace browsing |
| IDX-007 | object_type | All objects | Type-based queries |
| IDX-008 | ai_readiness_score (range) | CANONICAL | AI readiness queries |
| IDX-009 | capability (inverted) | CANONICAL | Capability search |
| IDX-010 | vector_embedding_id | All CANONICAL | Vector store reference |

---

## Index Entry Schema

```yaml
# One entry per KIL-augmented KnowledgeObject
kil_index_entry:
  # Core identity (IDX-001, IDX-002, IDX-003)
  knowledge_id: string
  canonical_name: string
  object_uri: string
  object_type: string
  namespace: string
  version: string
  state: string

  # Intelligence summary
  ai_readiness_score: float
  confidence_score: float
  quality_score: float
  risk_tier: string
  dna_hash: string
  genome_fingerprint: string

  # AI access (hot fields for fast serving)
  nano: string
  micro: string
  short_summary: string

  # Capability index (IDX-009)
  capabilities: [string]          # capability names for inverted index

  # Search pointers
  vector_embedding_id: string
  search_phrase_hash: string      # hash of semantic search phrases

  # Graph index
  hub_score: float
  direct_dependency_count: integer
  direct_dependent_count: integer

  # Timestamps
  kil_last_updated: string
  memory_last_updated: string
  index_entry_version: integer    # incremented on every update
```

---

## Full Index Structure

```yaml
kil_canonical_index:
  version: "1.0"
  generated_at: string
  total_entries: integer
  kos_version: "KOS v1.0.0"

  statistics:
    by_namespace: {}               # namespace → count
    by_object_type: {}             # type → count
    by_state: {}                   # state → count
    by_risk_tier: {}               # risk tier → count
    by_ai_readiness_band:
      ai_native: integer           # ≥ 0.90
      ai_ready: integer            # 0.80–0.89
      ai_capable: integer          # 0.70–0.79
      ai_partial: integer          # 0.60–0.69
      ai_manual: integer           # < 0.60
    avg_ai_readiness_score: float
    avg_confidence_score: float
    avg_quality_score: float
    total_capabilities: integer
    unique_capabilities: integer

  entries:
    - kil_index_entry              # one per object
```

---

## Index Maintenance

### Update Triggers

```
Triggers that require index entry update:
  - metadata.quality_score changed
  - metadata.lifecycle.state changed
  - intelligence.ai_readiness_score changed  (any component changed)
  - intelligence.confidence.overall_score changed
  - intelligence.dna.dna_hash changed        (CRITICAL — re-verify)
  - intelligence.genome.fingerprint changed
  - intelligence.memory.last_updated changed (memory fields only)
  - intelligence.compression.nano changed
  - intelligence.ai_context.short_summary changed

Batch refresh:
  - Nightly full index rebuild to catch any missed updates
  - Weekly consistency verification (index vs. object store)
```

### Consistency Invariants

```
Index consistency rules:
  IC-001: index.knowledge_id must match object.knowledge_id
  IC-002: index.dna_hash must match object.intelligence.dna.dna_hash
  IC-003: index.ai_readiness_score must equal computed AIRS
  IC-004: index.nano must equal compression.nano
  IC-005: No two entries may share the same genome_fingerprint
  IC-006: index.entry count must equal count of KIL-augmented objects
```

---

## Query Interface

The Canonical Index supports these query operations:

```
# By knowledge_id (O(1))
get_entry(knowledge_id: str) → kil_index_entry

# By canonical_name (O(1))
get_by_name(canonical_name: str) → kil_index_entry

# By capability (O(cap) inverted index)
find_by_capability(capability: str) → List[kil_index_entry]

# By AI readiness range (O(log n) range scan)
find_by_readiness(min_airs: float, max_airs: float) → List[kil_index_entry]

# Duplicate detection (O(1) by hash)
find_duplicates(dna_hash: str) → List[kil_index_entry]

# Namespace browser (O(n) in namespace)
list_namespace(namespace: str, state: str = None) → List[kil_index_entry]
```

---

## Rules

| Rule | Statement |
|------|-----------|
| KIL-145 | The canonical index must be updated within 60 seconds of any object change |
| KIL-146 | Consistency invariants IC-001 through IC-006 must pass nightly verification |
| KIL-147 | Any `genome_fingerprint` collision must trigger an immediate deduplication alert |
| KIL-148 | The index must be readable without loading the full object store |
| KIL-149 | Index entries are derived — the object store is always authoritative |

---

## Cross-References

- Genome fingerprint → `07-KNOWLEDGE-GENOME`
- DNA hash → `08-KNOWLEDGE-DNA`
- Search optimization → `28-KNOWLEDGE-SEARCH-OPTIMIZATION`
- Optimization (caching) → `24-KNOWLEDGE-OPTIMIZATION`
- KIL metrics → `26-KNOWLEDGE-METRICS`
