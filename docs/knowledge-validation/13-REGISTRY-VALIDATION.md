# KVF-DOC-013 — Registry Validation

**Phase:** 3.0D.1.5
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Registry Validation validates the KOS registry — the lookup system that maps
canonical names, aliases, and package paths to knowledge_ids. A correct registry
is required for entity resolution (Stage S1/S2 of the CAE pipeline).

---

## Registry Components to Validate

```
RV-COMP-1: Primary Name Index
  Maps canonical_name → knowledge_id
  Must be injective (one name → one ID)

RV-COMP-2: Alias Index
  Maps intelligence.search.keyword.aliases → knowledge_id
  May be non-injective (one alias → multiple candidates)

RV-COMP-3: Package Index
  Maps package.name.object_name → knowledge_id
  e.g., "platform.core.QuotaManager" → KNW-PLT-MOD-001

RV-COMP-4: Namespace Index
  Lists all objects in namespace X
  namespace → [knowledge_id, ...]

RV-COMP-5: Type Index
  Lists all objects of type X
  object_type → [knowledge_id, ...]
```

---

## Registry Validation Checks

### RV-1: Primary Name Index

```
RV-1.01: Every CANONICAL and ACTIVE object's canonical_name is in the index
RV-1.02: canonical_name → knowledge_id mapping is injective (no duplicates)
RV-1.03: No canonical_name maps to a non-existent knowledge_id
RV-1.04: Lookup latency P99 < 2ms
```

### RV-2: Alias Index

```
RV-2.01: All aliases in intelligence.search.keyword.aliases are indexed
RV-2.02: Alias lookup returns a list of candidates (not just one)
RV-2.03: The correct object is always in the alias candidate list
RV-2.04: Alias candidate list size <= 10
```

### RV-3: Package Index

```
RV-3.01: Full package path resolves to correct knowledge_id
         e.g., "platform.core.QuotaManager" → KNW-PLT-MOD-001
RV-3.02: Package index is consistent with object.namespace and object.name
RV-3.03: Package path lookup P99 < 2ms
```

### RV-4: Namespace Index

```
RV-4.01: namespace_index[ns].count == count(objects WHERE namespace == ns)
RV-4.02: All namespace values in namespace_index match declared namespaces
RV-4.03: List is sorted by object creation date DESC
```

### RV-5: Type Index

```
RV-5.01: type_index[type].count == count(objects WHERE object_type == type)
RV-5.02: All type values in type_index are from the declared KOS type vocabulary
```

---

## Registry Accuracy Test

```
Test: Submit 1,000 lookup queries of all types.

Registry accuracy metrics:
  primary_lookup_accuracy:   exact hit / total primary lookups
  alias_recall:              correct object in candidates / total alias lookups
  package_lookup_accuracy:   exact hit / total package lookups

  primary_lookup_accuracy: target >= 1.00
  alias_recall:             target >= 0.95
  package_lookup_accuracy:  target >= 0.99
```

---

## Registry Consistency Test

```
Registry must be consistent with the KIL index at all times.

For each object O updated in KIL:
  RV-C.01: Registry update arrives within 500ms of KIL update
  RV-C.02: Stale registry entries (pointing to old data) last < 1 second
  RV-C.03: DELETED objects are removed from registry within 1 second
```

---

## Registry Validation Result Schema

```yaml
registry_validation_result:
  lookups_tested: integer

  accuracy:
    primary_lookup_accuracy: float
    alias_recall: float
    package_lookup_accuracy: float

  latency:
    primary_p99_ms: float
    alias_p99_ms: float
    package_p99_ms: float

  consistency:
    update_propagation_max_ms: float
    stale_entry_count: integer          # entries pointing to stale data
    dangling_registry_entries: integer  # entries pointing to non-existent IDs

  by_component:
    - component: string
      checks_passed: integer
      checks_failed: integer
```

---

## Rules

| Rule | Statement |
|------|-----------|
| KVF-061 | Primary name index accuracy must be 1.00 — any miss is a critical entity resolution failure |
| KVF-062 | Dangling registry entries (pointing to deleted objects) > 0 is a critical failure |
| KVF-063 | Registry consistency test must simulate concurrent updates, not sequential |
| KVF-064 | Alias recall target 0.95 means at least 1 correct candidate in top-10 |
| KVF-065 | Lookup latency P99 < 2ms is measured after warm-up, not cold start |

---

## Cross-References

- Graph validation → `12-GRAPH-VALIDATION`
- KIL canonical index → Phase 3.0D.0.6 `29-KNOWLEDGE-CANONICAL-INDEX`
- Intent model (entity resolution) → Phase 3.0D.1 `03-INTENT-MODEL`
