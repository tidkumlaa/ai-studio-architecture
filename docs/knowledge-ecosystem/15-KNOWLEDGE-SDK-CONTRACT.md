# KNW-KE-ARCH-015 — Knowledge SDK Contract

**Phase:** 3.0C.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Language-neutral contracts for the Knowledge SDK. Any language that implements these contracts can consume the Knowledge Ecosystem. No implementation here — contracts only.

---

## SDK Modules

| Module | Responsibility |
|--------|---------------|
| `kos.identity` | ID generation, checksum, fingerprint |
| `kos.registry` | Object storage and lookup |
| `kos.relationships` | Relationship creation and traversal |
| `kos.evidence` | Evidence management |
| `kos.quality` | Quality scoring |
| `kos.confidence` | Confidence computation |
| `kos.lifecycle` | Lifecycle transitions |
| `kos.query` | KQL execution |
| `kos.search` | Text and semantic search |

---

## Python SDK Contract

```python
# contracts/python/kos_sdk.pyi  (type stub — no implementation)

from typing import Protocol, runtime_checkable

# ── Identity ──────────────────────────────────────────────────────────────────

@runtime_checkable
class IdentityEngineProtocol(Protocol):
    def generate_id(self, domain_code: str, type_code: str) -> str: ...
    def compute_checksum(self, obj: "KnowledgeObjectProtocol") -> str: ...
    def compute_fingerprint(self, namespace: str, type_code: str, checksum: str) -> str: ...
    def compute_identity(self, obj: "KnowledgeObjectProtocol") -> "IdentityRecord": ...
    def find_duplicate(self, fingerprint: str) -> str | None: ...
    def is_duplicate(self, fingerprint: str, own_id: str) -> bool: ...

# ── Registry ──────────────────────────────────────────────────────────────────

@runtime_checkable
class KnowledgeRegistryProtocol(Protocol):
    def register(self, obj: "KnowledgeObjectProtocol", *, allow_update: bool = False) -> "IdentityRecord": ...
    def update(self, obj: "KnowledgeObjectProtocol") -> "IdentityRecord": ...
    def unregister(self, knowledge_id: str) -> None: ...
    def bulk_register(self, objects: list["KnowledgeObjectProtocol"]) -> "BulkRegistrationResult": ...

    def get(self, knowledge_id: str) -> "RegistryEntry | None": ...
    def get_or_raise(self, knowledge_id: str) -> "RegistryEntry": ...
    def get_by_uri(self, uri: str) -> "RegistryEntry | None": ...
    def get_by_canonical_name(self, name: str) -> "RegistryEntry | None": ...
    def contains(self, knowledge_id: str) -> bool: ...
    def count(self) -> int: ...

    def list_all(self, filters: "RegistryFilters | None" = None) -> list["RegistryEntry"]: ...
    def list_by_type(self, object_type: str) -> list["RegistryEntry"]: ...
    def list_by_namespace(self, namespace: str) -> list["RegistryEntry"]: ...
    def list_by_status(self, status: str) -> list["RegistryEntry"]: ...
    def list_by_tag(self, tag: str) -> list["RegistryEntry"]: ...
    def stats(self) -> "RegistryStats": ...

# ── Relationships ─────────────────────────────────────────────────────────────

@runtime_checkable
class RelationshipEngineProtocol(Protocol):
    def add(self, relation: "KnowledgeRelation") -> str: ...
    def remove(self, relation_id: str) -> None: ...
    def get(self, relation_id: str) -> "KnowledgeRelation | None": ...
    def get_outgoing(self, source_id: str) -> list["KnowledgeRelation"]: ...
    def get_incoming(self, target_id: str) -> list["KnowledgeRelation"]: ...
    def traverse(self, start_id: str, max_depth: int) -> list["TraversalResult"]: ...
    def detect_cycle(self, source_id: str, target_id: str) -> bool: ...

# ── Quality ───────────────────────────────────────────────────────────────────

@runtime_checkable
class QualityEngineProtocol(Protocol):
    def compute(self, obj: "KnowledgeObjectProtocol") -> "QualityScoreRecord": ...
    def compute_dimension(self, obj: "KnowledgeObjectProtocol", dimension: int) -> float: ...
    def get_alerts(self, record: "QualityScoreRecord") -> list[str]: ...

# ── Evidence ──────────────────────────────────────────────────────────────────

@runtime_checkable
class EvidenceEngineProtocol(Protocol):
    def compute_freshness(self, record: "EvidenceRecord") -> float: ...
    def compute_evidence_score(self, items: list["EvidenceRecord"]) -> float: ...
    def is_sufficient(self, obj: "KnowledgeObjectProtocol") -> bool: ...

# ── Lifecycle ─────────────────────────────────────────────────────────────────

@runtime_checkable
class LifecycleEngineProtocol(Protocol):
    def can_transition(self, obj: "KnowledgeObjectProtocol", target_status: str) -> bool: ...
    def transition(self, obj: "KnowledgeObjectProtocol", target_status: str) -> "KnowledgeObjectProtocol": ...
    def get_allowed_transitions(self, current_status: str) -> list[str]: ...

# ── Query ─────────────────────────────────────────────────────────────────────

@runtime_checkable
class QueryEngineProtocol(Protocol):
    def execute(self, kql: str) -> "KQLResult": ...
    def parse(self, kql: str) -> "KQLPlan": ...
    def validate(self, kql: str) -> list[str]: ...  # returns error messages

# ── Search ────────────────────────────────────────────────────────────────────

@runtime_checkable
class SearchEngineProtocol(Protocol):
    def search(self, request: "SearchRequest") -> "SearchResponse": ...
    def index(self, obj: "KnowledgeObjectProtocol") -> None: ...
    def remove(self, knowledge_id: str) -> None: ...
    def rebuild(self) -> None: ...
```

---

## TypeScript SDK Contract

```typescript
// contracts/typescript/kos-sdk.d.ts

interface IdentityEngineContract {
  generateId(domainCode: string, typeCode: string): string;
  computeChecksum(obj: KnowledgeObjectContract): string;
  computeFingerprint(namespace: string, typeCode: string, checksum: string): string;
  findDuplicate(fingerprint: string): string | null;
}

interface KnowledgeRegistryContract {
  register(obj: KnowledgeObjectContract, options?: { allowUpdate: boolean }): IdentityRecord;
  update(obj: KnowledgeObjectContract): IdentityRecord;
  unregister(knowledgeId: string): void;
  get(knowledgeId: string): RegistryEntry | null;
  getOrThrow(knowledgeId: string): RegistryEntry;
  getByUri(uri: string): RegistryEntry | null;
  listAll(filters?: RegistryFilters): RegistryEntry[];
  contains(knowledgeId: string): boolean;
  count(): number;
  stats(): RegistryStats;
}

interface QualityEngineContract {
  compute(obj: KnowledgeObjectContract): QualityScoreRecord;
}
```

---

## Java SDK Contract

```java
// contracts/java/KnowledgeRegistryContract.java

public interface KnowledgeRegistryContract {
    IdentityRecord register(KnowledgeObjectContract obj) throws DuplicateObjectException;
    IdentityRecord register(KnowledgeObjectContract obj, boolean allowUpdate) throws DuplicateObjectException;
    IdentityRecord update(KnowledgeObjectContract obj) throws ObjectNotFoundException;
    void unregister(String knowledgeId) throws ObjectNotFoundException;
    Optional<RegistryEntry> get(String knowledgeId);
    RegistryEntry getOrThrow(String knowledgeId) throws ObjectNotFoundException;
    Optional<RegistryEntry> getByUri(String uri);
    List<RegistryEntry> listAll(RegistryFilters filters);
    boolean contains(String knowledgeId);
    int count();
    RegistryStats stats();
}
```

---

## SDK Rules

| Rule | Description |
|------|-------------|
| SDK-001 | All contracts are pure interfaces — no implementation in this document |
| SDK-002 | Method signatures are identical across languages (adjusted for idioms) |
| SDK-003 | Error types map to same semantic errors: DuplicateObjectError, ObjectNotFoundError |
| SDK-004 | No SDK method may mutate the repository without going through the Registry contract |
| SDK-005 | All contracts must be versioned (contract version in package header) |

---

## Cross-References

- Runtime implements these contracts → Phase 3.0D
- Python stub matches Phase 3.0B object model → `platform/knowledge_runtime/objects/`
- Registry protocol → Phase 3.0C `14-REGISTRY-ARCHITECTURE`
