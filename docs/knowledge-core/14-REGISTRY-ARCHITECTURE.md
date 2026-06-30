# KNW-KC-ARCH-014 — Registry Architecture

**Phase:** 3.0C  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

The Registry Architecture defines the structural contract shared by all domain registries in KOS. Every registry is versioned, searchable, indexed, and machine-readable. Nine domain registries are defined in documents 15–23.

---

## Registry Hierarchy

```
KnowledgeRegistry (master)
│
├── ObjectRegistry          (15) — all KnowledgeObject instances
├── ServiceRegistry         (16) — platform services (ABC interfaces)
├── APIRegistry             (17) — API surface catalog
├── EventRegistry           (18) — domain event catalog
├── RuntimeRegistry         (19) — runtime catalog
├── ProductRegistry         (20) — product catalog
├── AlgorithmRegistry       (21) — algorithm catalog
├── PatternRegistry         (22) — pattern catalog
└── DatasetRegistry         (23) — dataset catalog
```

The `KnowledgeRegistry` (master) is the single index that knows about all domain registries. Domain registries are authoritative for their own objects. All writes go through the domain registry and are reflected in the master index.

---

## Universal Registry Contract

Every registry must implement:

```python
class KnowledgeRegistryProtocol(Protocol):
    # Write operations
    def register(self, obj: KnowledgeObject) -> str: ...          # knowledge_id
    def update(self, obj: KnowledgeObject) -> str: ...
    def deprecate(self, knowledge_id: str, reason: str) -> None: ...
    def tombstone(self, knowledge_id: str, successor: str | None) -> None: ...

    # Read operations
    def get(self, knowledge_id: str) -> KnowledgeObject: ...
    def get_by_uri(self, uri: str) -> KnowledgeObject: ...
    def get_version(self, knowledge_id: str, version: str) -> KnowledgeObject: ...
    def list_all(self, filters: RegistryFilters) -> list[KnowledgeObject]: ...

    # Search
    def search(self, query: str, filters: RegistryFilters) -> list[SearchResult]: ...
    def find_by_tag(self, tag: str) -> list[KnowledgeObject]: ...
    def find_by_owner(self, owner: str) -> list[KnowledgeObject]: ...
    def find_by_status(self, status: LifecycleStatus) -> list[KnowledgeObject]: ...

    # Versioning
    def get_all_versions(self, knowledge_id: str) -> list[VersionRecord]: ...
    def get_latest(self, knowledge_id: str) -> KnowledgeObject: ...

    # Health
    def validate(self, knowledge_id: str) -> RegistryValidationResult: ...
    def count(self, filters: RegistryFilters | None) -> int: ...
    def stats(self) -> RegistryStats: ...
```

---

## Registry Storage Format

```
knowledge/registry/
  master-index.yaml           # top-level index of all registries
  objects/                    # ObjectRegistry
    {knowledge_id}.yaml       # one file per object
  services/                   # ServiceRegistry
    {knowledge_id}.yaml
  apis/
    {knowledge_id}.yaml
  events/
    {knowledge_id}.yaml
  runtimes/
    {knowledge_id}.yaml
  products/
    {knowledge_id}.yaml
  algorithms/
    {knowledge_id}.yaml
  patterns/
    {knowledge_id}.yaml
  datasets/
    {knowledge_id}.yaml
  relations/
    {source_id}/
      {relation_id}.yaml      # grouped by source for fast lookup
  versions/
    {knowledge_id}/
      {version}.yaml          # all version records
  snapshots/
    {snapshot_id}/
      ...
  indexes/
    by-tag.yaml               # tag → list[knowledge_id]
    by-owner.yaml             # owner → list[knowledge_id]
    by-namespace.yaml         # namespace → list[knowledge_id]
    by-status.yaml            # status → list[knowledge_id]
    by-type.yaml              # object_type → list[knowledge_id]
    uri-index.yaml            # uri → knowledge_id
    fingerprint-index.yaml    # fingerprint → knowledge_id (dedup)
```

---

## Master Index Format

```yaml
# master-index.yaml
kos_version: "1.0.0"
generated_at: datetime
total_objects: 0
total_relations: 0
registries:
  objects:
    count: 0
    last_updated: datetime
    path: "objects/"
    health: HEALTHY
  services:
    count: 0
    last_updated: datetime
    path: "services/"
    health: HEALTHY
  # ... all 9 registries
indexes:
  by-tag:
    count: 0
    last_rebuilt: datetime
  by-owner:
    count: 0
    last_rebuilt: datetime
```

---

## Registry Write Protocol

```
REGISTER(object):
  1. Validate identity (01-IDENTITY-ENGINE)
  2. Validate Universal Schema (02-UNIVERSAL-SCHEMA)
  3. Check fingerprint for duplicates (dedup index)
  4. Check canonical_name uniqueness in namespace (namespace index)
  5. Write {knowledge_id}.yaml to domain registry path
  6. Update master-index.yaml count and timestamp
  7. Update all secondary indexes (by-tag, by-owner, etc.)
  8. Create VERSION_SNAPSHOT (09-SNAPSHOT-ENGINE)
  9. Emit registry.object.registered event (18-EVENT-REGISTRY)
  10. Return knowledge_id

COMPLEXITY: O(log N) for index updates
```

---

## Registry Consistency Rules

| Rule | Description |
|------|-------------|
| RC-001 | Write-ahead index update — indexes must update before write is acknowledged |
| RC-002 | Atomic writes — a partial write (crashed mid-write) is rolled back on next startup |
| RC-003 | Index rebuilds — if index is corrupt, rebuild from object files (O(N)) |
| RC-004 | Master index — always reflects current state of all domain registries |
| RC-005 | No direct file writes — all mutations go through registry protocol |

---

## Registry Filters

```python
@dataclass
class RegistryFilters:
    namespace: str | None = None
    object_type: KnowledgeObjectType | None = None
    status: LifecycleStatus | None = None
    owner: str | None = None
    tags: list[str] = field(default_factory=list)
    created_after: datetime | None = None
    updated_after: datetime | None = None
    min_quality_score: float | None = None
    max_quality_score: float | None = None
    limit: int = 100
    offset: int = 0
    sort_by: str = "updated_at"
    sort_order: str = "desc"
```

---

## Registry Events

All write operations emit events to the Event Registry (18):

| Event | Trigger |
|-------|---------|
| `registry.object.registered` | New object written |
| `registry.object.updated` | Existing object updated |
| `registry.object.deprecated` | Object deprecated |
| `registry.object.tombstoned` | Object permanently removed |
| `registry.index.rebuilt` | Secondary index rebuilt |
| `registry.health.alert` | Registry consistency violation |

---

## Cross-References

- Domain registries → `15` through `23`
- Object files format → `02-UNIVERSAL-SCHEMA`
- Event format → `18-EVENT-REGISTRY`
- Snapshot on write → `09-SNAPSHOT-ENGINE`
- Search implementation → `28-SEARCH-ENGINE`
