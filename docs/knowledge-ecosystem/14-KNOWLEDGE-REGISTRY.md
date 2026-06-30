# KNW-KE-ARCH-014 — Knowledge Registry (Ecosystem)

**Phase:** 3.0C.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

The Ecosystem Registry is the language-neutral contract for all registry operations. It is the interface between the Knowledge Repository (files) and the Knowledge Runtime (Phase 3.0D). Every implementation must satisfy this contract.

---

## Registry Contract (Language-Neutral)

```
OPERATIONS:
  register(object)          → IdentityRecord
  update(object)            → IdentityRecord
  unregister(knowledge_id)  → void
  bulk_register(objects)    → BulkResult

  get(knowledge_id)         → Object | null
  get_by_uri(uri)           → Object | null
  get_by_canonical_name(n)  → Object | null
  get_or_raise(knowledge_id)→ Object
  contains(knowledge_id)    → bool
  count()                   → int

  list_all(filters?)        → Object[]
  list_by_type(type)        → Object[]
  list_by_namespace(ns)     → Object[]
  list_by_status(status)    → Object[]
  list_by_tag(tag)          → Object[]

  stats()                   → RegistryStats

INVARIANTS:
  RI-001: Every registered object has a unique knowledge_id
  RI-002: Every registered object has a unique fingerprint
  RI-003: Every registered object has a unique canonical_name within its namespace
  RI-004: Every registered object has a unique knowledge_uri
  RI-005: count() == len(list_all())
```

---

## Registry Files

The Ecosystem Registry is file-based. All files live in `knowledge/registry/`:

```
knowledge/registry/
├── index.yaml              # primary index (knowledge_id → file path)
├── aliases.yaml            # canonical_name and URI aliases
├── search-index.yaml       # search index (machine-generated)
├── fingerprints.yaml       # fingerprint → knowledge_id (duplicate detection)
├── cross-reference.yaml    # cross-package reference map
└── stats.yaml              # registry statistics (machine-generated)
```

---

## Primary Index Format

```yaml
# knowledge/registry/index.yaml
version: "1.0.0"
generated_at: "2026-06-30T00:00:00Z"
entries:
  KNW-PLT-MOD-001:
    file: packages/platform/objects/quota-manager.yaml
    canonical_name: plt.module.quota-manager
    knowledge_uri: knw://plt/module/quota-manager@1.0.0
    object_type: module
    namespace: plt
    version: 1.0.0
    status: CANONICAL
    checksum: sha256:abc123...
    fingerprint: fp:plt:mod:deadbeef01234567
  # ... all objects
```

---

## Alias Registry

```yaml
# knowledge/registry/aliases.yaml
aliases:
  canonical_name:
    "plt.module.quota":
      canonical: "plt.module.quota-manager"
      expires_at: null
  uri:
    "knw://plt/module/quota@1.0.0":
      canonical: "knw://plt/module/quota-manager@1.0.0"
      expires_at: null
  knowledge_id:
    "KNW-PLT-MOD-OLD-001":
      canonical: "KNW-PLT-MOD-001"
      expires_at: "2027-01-01"
```

---

## Cross-Reference Map

Tracks which objects in one package depend on objects in another:

```yaml
# knowledge/registry/cross-reference.yaml
cross_references:
  - source_package: kos.platform.package
    source_id: KNW-PLT-MOD-001
    target_package: kos.runtime.package
    target_id: KNW-RT-RT-001
    relation_type: DEPENDS_ON
  # ...
```

Used by the dependency verifier and the package manager.

---

## Registry Statistics

```yaml
# knowledge/registry/stats.yaml
generated_at: "2026-06-30T00:00:00Z"
total_objects: 0
total_relationships: 0
total_namespaces: 9
by_status:
  CANONICAL: 0
  VERIFIED: 0
  DRAFT: 0
by_type: {}
by_namespace:
  plt: 0
  rt: 0
  # ...
avg_quality_score: 0.0
orphan_objects: 0
broken_references: 0
```

---

## Registry Update Protocol

1. Author writes/modifies object YAML file
2. `kos format` normalises file
3. `kos lint` validates file
4. On CI pass: `kos registry update {file}` → updates index.yaml
5. `kos registry verify` → checks all invariants
6. Catalog is rebuilt: `kos catalog build`

All steps are automated in CI.

---

## Registry Plug-in Model

The file-based registry is the canonical implementation. Runtime implementations (Phase 3.0D) may use in-memory, SQLite, or graph databases, but must expose the same contract.

```
RegistryProtocol (language-neutral)
  │
  ├── FileRegistry (this document — canonical)
  │   → reads from knowledge/registry/index.yaml
  │
  ├── MemoryRegistry (Phase 3.0D — runtime use)
  │   → implements Phase 3.0C 14-REGISTRY-ARCHITECTURE
  │
  └── GraphRegistry (Phase 3.0E — production scale)
      → uses graph database backend
```

---

## Cross-References

- Registry runtime contract → Phase 3.0C `14-REGISTRY-ARCHITECTURE`
- Alias resolution protocol → Phase 3.0C `04-URI-SPECIFICATION`
- Duplicate detection → Phase 3.0C `01-IDENTITY-ENGINE`
- Package manager uses registry → `26-KNOWLEDGE-PACKAGE-MANAGER`
