# KNW-KC-ARCH-022 — Pattern Registry

**Phase:** 3.0C  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

The Pattern Registry catalogs all architectural, design, and operational patterns used across the KOS and platform. Patterns encode reusable solutions that have been proven in context.

---

## Registered Patterns

### Architectural Patterns

| ID | Name | Category | Used By |
|----|------|----------|---------|
| `PAT-KC-001` | Registry + Protocol | architectural | All registries |
| `PAT-KC-002` | Knowledge-First Development | architectural | Entire KOS |
| `PAT-KC-003` | Layered Platform Architecture | architectural | Platform kernel, SDK |
| `PAT-KC-004` | Event-Driven Integration | architectural | Cross-runtime comms |
| `PAT-KC-005` | Dependency Inversion via Protocol | architectural | All platform modules |
| `PAT-KC-006` | Versioned Immutable Records | architectural | Version Engine |

### Design Patterns

| ID | Name | Category | Used By |
|----|------|----------|---------|
| `PAT-KC-007` | Provider Plugin | structural | AI provider integrations |
| `PAT-KC-008` | Discriminated Union (Pydantic Literal) | structural | KnowledgeObject subtypes |
| `PAT-KC-009` | Chain of Responsibility (Fallback Chain) | behavioral | Provider failover |
| `PAT-KC-010` | Observer (Event Bus) | behavioral | Platform event system |
| `PAT-KC-011` | Factory (Object Registry) | creational | KnowledgeObject instantiation |
| `PAT-KC-012` | Strategy (Execution Plan) | behavioral | Workload routing |

### Operational Patterns

| ID | Name | Category | Used By |
|----|------|----------|---------|
| `PAT-KC-013` | Architecture Freeze Gate | operational | Every phase transition |
| `PAT-KC-014` | Snapshot + Rollback | operational | Migration, deployment |
| `PAT-KC-015` | Evidence-Gated Lifecycle | operational | All knowledge objects |
| `PAT-KC-016` | Traceability Matrix | operational | Requirements ↔ Tests |

---

## Pattern Entry Format

```yaml
# knowledge/registry/patterns/PAT-KC-001.yaml
identity:
  knowledge_id: "KNW-KC-PAT-001"
  canonical_name: "kc.pattern.registry_protocol"
  knowledge_uri: "knw://kos/pattern/registry-protocol"
  namespace: "kos"
  version: "1.0.0"
  owner: "team:architecture-board"

pattern_spec:
  pattern_id: "PAT-KC-001"
  category: architectural
  problem: |
    Components need a shared, discoverable catalog of typed objects
    that supports versioning, lifecycle management, and search
    without coupling consumers to storage implementation details.
  solution: |
    Define a Protocol interface (KnowledgeRegistryProtocol) that all
    registries implement. Back with YAML file storage. Expose search,
    version, and lifecycle operations through the protocol. Consumers
    depend only on the Protocol, never on the storage class.
  consequences: |
    Storage backend is replaceable. Consumers are testable with mock
    implementations. Registry can evolve (YAML→SQLite→graph DB)
    without touching consumers. Tradeoff: protocol must be designed
    for extensibility up-front.
  examples:
    - "ObjectRegistry implements KnowledgeRegistryProtocol"
    - "ServiceRegistry implements KnowledgeRegistryProtocol"
    - "All 9 domain registries use identical protocol"

lifecycle:
  status: CANONICAL
```

---

## Pattern Registry Protocol

```python
class PatternRegistry(KnowledgeRegistryProtocol):
    def get_by_category(self, category: str) -> list[PatternObject]: ...
    def get_usage(self, pattern_id: str) -> list[str]: ...       # knowledge_ids using it
    def get_similar(self, pattern_id: str) -> list[PatternObject]: ...
```

---

## Cross-References

- Registry base → `14-REGISTRY-ARCHITECTURE`
- Pattern objects → Phase 3.0B `knowledge_runtime/objects/architecture.py`
- Provider Plugin pattern → Phase 2.1D.0 `34-ARCHITECTURE-DECISIONS` (ADR-006)
