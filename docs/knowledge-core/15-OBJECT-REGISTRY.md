# KNW-KC-ARCH-015 — Object Registry

**Phase:** 3.0C  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

The Object Registry is the primary, authoritative store for all 33 KnowledgeObject types. Every object in the KOS ecosystem has exactly one entry in this registry. All other domain registries are projections or subsets of the Object Registry.

---

## Scope

The Object Registry stores **all** `KnowledgeObject` instances across all 33 types:

| Group | Types |
|-------|-------|
| Platform | Platform, Runtime, Kernel, SDK, Package |
| Architecture | Architecture, Decision, Requirement, Pattern, Specification |
| Implementation | Module, Service, Algorithm, Configuration |
| Interface | API, Capability, Provider |
| Quality | Test, Benchmark |
| Product | Product, Agent, Prompt, Conversation, Task, Strategy, ExecutionPlan |
| Financial | Market, Forex, News, Dataset |
| Meta | Document, KnowledgeBase |

---

## Storage Layout

```
knowledge/registry/objects/
  KNW-KOS-ARCH-001.yaml
  KNW-KOS-ARCH-002.yaml
  KNW-AI-MOD-quota_manager.yaml
  KNW-PLAT-SVC-001.yaml
  ...
```

Each file is a complete serialization of a `KnowledgeObject` conforming to `02-UNIVERSAL-SCHEMA`.

---

## Object File Format

```yaml
# KNW-KOS-ARCH-001.yaml
identity:
  global_id: "a3f2c1d4-0000-0000-0000-000000000001"
  knowledge_id: "KNW-KOS-ARCH-001"
  knowledge_uri: "knw://kos/arch/vision"
  namespace: "kos"
  canonical_name: "kos.arch.vision"
  aliases: ["vision", "kos-vision"]
  version: "1.0.0"
  owner: "team:architecture-board"
  domain: "ARCHITECTURE"
  created_at: "2026-01-01T00:00:00Z"
  updated_at: "2026-01-01T00:00:00Z"
  checksum: "sha256:abc123..."
  fingerprint: "fp:kos:arch:vision:1"
  lineage: null

classification:
  object_type: ARCHITECTURE
  sub_type: "vision_document"
  category: "core"
  tags: ["kos", "vision", "architecture"]
  maturity: CANONICAL
  tier: 0

metadata:
  title: "KOS Vision & Master Architecture"
  description: "Defines the purpose, scope, and philosophy of the Knowledge Operating System"
  purpose: "Establish KOS as the single source of truth for all ecosystem knowledge"

lifecycle:
  status: CANONICAL
  review_status: PASSED
  approved_by: "arch-board"
  approved_at: "2026-06-01T00:00:00Z"

# ... all 16 sections per 02-UNIVERSAL-SCHEMA
```

---

## Registration Rules (Object-Specific)

### OR-001 — One Entry Per Object
Every `KnowledgeObject` has exactly one file in the Object Registry, regardless of how many domain registries also reference it.

### OR-002 — Domain Registry Sync
When an object is updated in the Object Registry, all domain registries that reference it receive an automatic update notification within 100ms.

### OR-003 — Deletion Policy
Objects are never deleted from the Object Registry. They are DEPRECATED then ARCHIVED then TOMBSTONED. Tombstone records are permanent.

### OR-004 — Type Enforcement
The `object_type` field is immutable after first write. Changing the type requires creating a new object with a SUPERSEDES relationship.

### OR-005 — Bulk Registration
The registry supports bulk registration of up to 1000 objects in a single atomic transaction. Either all succeed or all are rolled back.

---

## Object Registry Statistics

```yaml
# Tracked in master-index.yaml
object_registry_stats:
  total: 0
  by_status:
    DRAFT: 0
    REVIEW: 0
    VERIFIED: 0
    APPROVED: 0
    CANONICAL: 0
    DEPRECATED: 0
    ARCHIVED: 0
  by_type:
    ARCHITECTURE: 0
    MODULE: 0
    # ... all 33 types
  by_namespace:
    kos: 0
    platform: 0
    # ... all registered namespaces
  health:
    orphaned_objects: 0      # objects with no relationships
    stale_objects: 0         # quality score decayed below threshold
    duplicate_alerts: 0
    total_healthy: 0
```

---

## Object Registry Protocol

```python
class ObjectRegistry(KnowledgeRegistryProtocol):
    # Inherits all methods from KnowledgeRegistryProtocol (14-REGISTRY-ARCHITECTURE)

    # Object-specific extensions
    def get_by_type(
        self, object_type: KnowledgeObjectType, filters: RegistryFilters | None
    ) -> list[KnowledgeObject]: ...

    def get_by_fingerprint(self, fingerprint: str) -> KnowledgeObject | None: ...

    def get_orphaned(self) -> list[KnowledgeObject]: ...

    def get_stale(self, max_quality_score: float) -> list[KnowledgeObject]: ...

    def bulk_register(
        self, objects: list[KnowledgeObject]
    ) -> BulkRegistrationResult: ...

    def get_stats(self) -> ObjectRegistryStats: ...
```

---

## Object Registry Indexes

| Index | Key | Value | Use Case |
|-------|-----|-------|---------|
| Primary | `knowledge_id` | file path | Direct lookup |
| URI | `knowledge_uri` | `knowledge_id` | URI resolution |
| Fingerprint | `fingerprint` | `knowledge_id` | Deduplication |
| Canonical Name | `canonical_name` | `knowledge_id` | Name lookup |
| Type | `object_type` | `list[knowledge_id]` | Type queries |
| Status | `lifecycle.status` | `list[knowledge_id]` | Lifecycle queries |
| Owner | `owner` | `list[knowledge_id]` | Owner queries |
| Tag | `tag` | `list[knowledge_id]` | Tag queries |
| Namespace | `namespace` | `list[knowledge_id]` | Namespace scoping |
| Quality | `overall_score` (sorted) | `knowledge_id` | Quality ranking |

---

## Cross-References

- Registry base contract → `14-REGISTRY-ARCHITECTURE`
- Object schema → `02-UNIVERSAL-SCHEMA`
- Type hierarchy → `05-OBJECT-INHERITANCE`
- Search across objects → `28-SEARCH-ENGINE`
