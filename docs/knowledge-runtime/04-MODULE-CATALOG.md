---
knowledge_id: KR-ARCH-004
version: "1.0.0"
status: approved
owner: Chief Knowledge Architect
phase: 2.0D.3A
created: 2026-06-29
canonical: true
domain: DOM-KNOWLEDGE
capability: knowledge-runtime
type: catalog
depends_on:
  - id: KA-SPEC-020
    reason: "Runtime Architecture blueprint this catalog implements"
  - id: KA-KIP-002
    reason: "Knowledge Intelligence Platform specifications"
  - id: KA-SPEC-007
    reason: "KQL definition used by KnowledgeQueryEngine"
  - id: KA-SPEC-009
    reason: "Compiler specification used by KnowledgeCompiler"
  - id: KA-SPEC-010
    reason: "Index engine specification used by KnowledgeIndexer"
  - id: KA-SPEC-012
    reason: "Evidence model used by KnowledgeEvidenceEngine"
  - id: KA-SPEC-013
    reason: "Coverage model used by KnowledgeCoverageEngine"
---

# Knowledge Runtime — Complete Module Catalog

## All 31 Runtime Modules: Identity, Interfaces, Dependencies, Contracts

---

## Overview

Knowledge Runtime (Phase 2.0D.3A) is the deployable service layer that brings the knowledge
object model to life. It is built on top of Platform Kernel — every module either extends
`PlatformService`, `PlatformComponent`, or a specialized Platform base class. All knowledge
objects extend `PlatformEntity`. All events flow through `PlatformEventBus`. All scheduled
work uses `PlatformScheduler`.

This catalog is the authoritative reference for all 31 runtime modules. Each entry specifies
identity, public interface, dependencies, event contracts, performance targets, and failure
recovery. No implementation code is included — this is a specification.

### Module Count by Group

| Group | Name                      | Count |
|-------|---------------------------|-------|
| A     | Runtime Core              | 3     |
| B     | Resolution & Registry     | 2     |
| C     | Indexing                  | 1     |
| D     | Graph                     | 2     |
| E     | Intelligence              | 2     |
| F     | Compilation & Validation  | 2     |
| G     | Version Management        | 4     |
| H     | Evidence & Coverage       | 2     |
| I     | Recommendations           | 1     |
| J     | Context & Workspace       | 4     |
| K     | Observability             | 3     |
| L     | Security                  | 1     |
| M     | Events & Subscriptions    | 2     |
| N     | Distribution              | 2     |
| **Total** |                       | **31** |

### Boot Order

```
A1.KnowledgeRuntime
  └─ Phase 1 — Foundation
       A2.KnowledgeStore
       A3.KnowledgeRepository
       B1.KnowledgeResolver
       B2.KnowledgeRegistry
       K3.KnowledgeCache
       L1.KnowledgeSecurity
  └─ Phase 2 — Index
       C1.KnowledgeIndexer
  └─ Phase 3 — Graph
       D1.KnowledgeGraphRuntime
       D2.KnowledgeTraversalEngine
  └─ Phase 4 — Intelligence
       E1.KnowledgeReasoningEngine
       E2.KnowledgeQueryEngine
       F1.KnowledgeCompiler
       F2.KnowledgeValidator
       G1.KnowledgeVersionManager
       G2.KnowledgeSnapshotManager
       G3.KnowledgeDiffEngine
       G4.KnowledgeMergeEngine
       H1.KnowledgeEvidenceEngine
       H2.KnowledgeCoverageEngine
       I1.KnowledgeRecommendationEngine
       J1.KnowledgeExecutionContext  (factory, not a running service)
       J2.KnowledgeSession           (factory)
       J3.KnowledgeWorkspace         (factory)
       J4.KnowledgeProject           (factory)
       K1.KnowledgeAudit
       K2.KnowledgeMetrics
       M1.KnowledgeEvents            (constants only, no boot)
       M2.KnowledgeSubscriptions
       N1.KnowledgeReplication
       N2.KnowledgeImportExport
  └─ Phase 5 — Ready
       knowledge.runtime.ready emitted
```

---

## Group A — Runtime Core

### A1. KnowledgeRuntime

**Module identity**

| Field      | Value                        |
|------------|------------------------------|
| Name       | KnowledgeRuntime             |
| Group      | A — Runtime Core             |
| Class      | `KnowledgeRuntime`           |
| Extends    | `PlatformService`            |
| Module ID  | KR-MOD-A1                   |

**Responsibility**

- Acts as the root service that owns and coordinates all 30 sub-services; nothing in the
  runtime starts without it starting first.
- Executes a five-phase boot sequence (Foundation → Index → Graph → Intelligence → Ready),
  starting each phase only after all prior-phase services report healthy.
- Manages the full lifecycle (start, stop, reload, health-check) for every sub-service.
- Provides a `get_service(name: str)` service-locator so modules can obtain typed references
  to peers without hard constructor coupling.
- Coordinates graceful shutdown: drains in-flight events, checkpoints state via
  KnowledgeSnapshotManager, and stops phases in reverse order.

**Key public interface**

```
class KnowledgeRuntime(PlatformService):

    async def start() -> None
    async def stop() -> None
    async def reload() -> None

    def get_service(name: str) -> PlatformService
    def get_service_typed(name: str, type_: Type[T]) -> T

    def health() -> RuntimeHealth
    def boot_phase() -> BootPhase   # FOUNDATION | INDEX | GRAPH | INTELLIGENCE | READY

    def register_extension(ext: KnowledgeExtension) -> None
    def list_services() -> List[ServiceDescriptor]
```

**Dependencies**

- All 30 sub-services (owned, not injected)
- `PlatformScheduler` — schedules health-check heartbeat
- `PlatformEventBus` — wires all inter-module event flows

**Events emitted**

| Event                         | Trigger                                     | Payload                                  |
|-------------------------------|---------------------------------------------|------------------------------------------|
| `knowledge.runtime.started`   | All Foundation services healthy             | `{phase: "FOUNDATION", ts: ISO8601}`     |
| `knowledge.runtime.ready`     | All phases complete                         | `{boot_duration_ms: int, phase: "READY"}`|
| `knowledge.runtime.stopping`  | `stop()` called                             | `{reason: str, ts: ISO8601}`             |
| `knowledge.runtime.error`     | Any phase fails to boot                     | `{phase: str, error: str, module: str}`  |

**Performance contract**

- Boot (10,000 objects): < 5 seconds end-to-end
- `get_service()`: O(1), < 0.01ms
- Health-check heartbeat interval: 30 seconds

**Failure modes and recovery**

| Failure                       | Recovery                                                      |
|-------------------------------|---------------------------------------------------------------|
| Sub-service fails during boot | Emit `knowledge.runtime.error`; abort boot; log full trace    |
| Sub-service crashes at runtime| Attempt restart up to 3 times; quarantine after 3rd failure   |
| Shutdown timeout (> 30s)      | Force-kill remaining services; write crash snapshot           |

---

### A2. KnowledgeStore

**Module identity**

| Field      | Value                        |
|------------|------------------------------|
| Name       | KnowledgeStore               |
| Group      | A — Runtime Core             |
| Class      | `KnowledgeStore`             |
| Extends    | `PlatformService`            |
| Module ID  | KR-MOD-A2                   |

**Responsibility**

- Provides durable persistence for all KnowledgeObjects; abstracts the file system so that
  the rest of the runtime never touches files directly.
- Supports two backend modes: `FileSystemBackend` (production, YAML + binary) and
  `InMemoryBackend` (test isolation, zero disk I/O).
- Maintains a write-ahead log (WAL) so that any crash mid-write can be detected and rolled
  back on next start.
- Offers two serialization paths: YAML (canonical, human-readable, authoritative) and a
  binary format (fast path for bulk reads — MessagePack-encoded KnowledgeObject).
- Exposes a change-stream: each write fires `knowledge.store.written` so subscribers
  (KnowledgeIndexer, KnowledgeGraphRuntime, KnowledgeCompiler) can react.

**Key public interface**

```
class KnowledgeStore(PlatformService):

    async def read(knowledge_id: str) -> Optional[KnowledgeObject]
    async def read_many(ids: List[str]) -> List[KnowledgeObject]
    async def write(obj: KnowledgeObject) -> WriteResult
    async def write_batch(objs: List[KnowledgeObject]) -> List[WriteResult]
    async def delete(knowledge_id: str) -> bool
    async def exists(knowledge_id: str) -> bool

    async def scan(predicate: StorePredicate) -> AsyncIterator[KnowledgeObject]
    async def count() -> int

    def backend() -> StoreBackend          # FileSystemBackend | InMemoryBackend
    def wal_path() -> Optional[Path]
    async def replay_wal() -> int          # returns number of entries replayed
    async def compact() -> CompactionResult
```

**Dependencies**

- `PlatformEventBus` — emits write/delete events
- `PlatformScheduler` — schedules WAL compaction

**Events emitted**

| Event                       | Trigger              | Payload                                                   |
|-----------------------------|----------------------|-----------------------------------------------------------|
| `knowledge.store.written`   | Successful write     | `{id: str, version: str, ts: ISO8601, backend: str}`      |
| `knowledge.store.deleted`   | Successful delete    | `{id: str, ts: ISO8601}`                                  |
| `knowledge.store.wal.error` | WAL write fails      | `{id: str, error: str, ts: ISO8601}`                      |

**Performance contract**

- In-memory read: < 1ms (p99)
- File-system read (binary path): < 5ms (p99)
- File-system write: < 10ms (p99)
- WAL flush: < 2ms (p99)
- Bulk scan (10,000 objects): < 10s

**Failure modes and recovery**

| Failure               | Recovery                                                    |
|-----------------------|-------------------------------------------------------------|
| WAL corruption        | Detect checksum mismatch on startup; truncate to last-good  |
| Disk full             | Emit error event; block writes; alert via PlatformMetrics   |
| Binary file corrupt   | Fall back to YAML canonical; log warning; rebuild binary    |

---

### A3. KnowledgeRepository

**Module identity**

| Field      | Value                         |
|------------|-------------------------------|
| Name       | KnowledgeRepository           |
| Group      | A — Runtime Core              |
| Class      | `KnowledgeRepository`         |
| Extends    | `PlatformService`             |
| Module ID  | KR-MOD-A3                    |

**Responsibility**

- Provides a typed, domain-aware CRUD layer on top of KnowledgeStore; callers use typed
  query methods rather than raw IDs.
- Implements the Unit of Work pattern: multiple object writes can be grouped into a single
  atomic transaction that either all succeeds or all rolls back.
- Supports typed projection queries: `find_by_type`, `find_by_domain`, `find_by_capability`,
  `find_by_tag`, each returning typed, paginated result sets.
- Enforces pre-write validation via KnowledgeValidator before any object reaches the store.
- Emits lifecycle events (created, updated, deleted) carrying the full object so downstream
  subscribers receive a consistent view without a second fetch.

**Key public interface**

```
class KnowledgeRepository(PlatformService):

    async def find_by_id(knowledge_id: str,
                         ctx: KnowledgeExecutionContext = None) -> Optional[KnowledgeObject]
    async def find_by_type(type_: KnowledgeType,
                           page: PageRequest = None) -> Page[KnowledgeObject]
    async def find_by_domain(domain: str,
                             page: PageRequest = None) -> Page[KnowledgeObject]
    async def find_by_capability(capability: str,
                                 page: PageRequest = None) -> Page[KnowledgeObject]
    async def find_by_tag(tag: str,
                          page: PageRequest = None) -> Page[KnowledgeObject]

    async def save(obj: KnowledgeObject,
                   ctx: KnowledgeExecutionContext = None) -> KnowledgeObject
    async def delete(knowledge_id: str,
                     ctx: KnowledgeExecutionContext = None) -> bool

    def unit_of_work() -> KnowledgeUnitOfWork    # context manager
    async def count_by_domain() -> Dict[str, int]
    async def count_by_type() -> Dict[str, int]
```

**Dependencies**

- `KnowledgeStore` (A2) — persistence backend
- `KnowledgeValidator` (F2) — pre-write validation
- `PlatformEventBus` — event emission

**Events emitted**

| Event                                    | Trigger          | Payload                                          |
|------------------------------------------|------------------|--------------------------------------------------|
| `knowledge.repository.object.created`    | First save       | `{id: str, type: str, domain: str, ts: ISO8601}` |
| `knowledge.repository.object.updated`    | Subsequent save  | `{id: str, version: str, ts: ISO8601}`           |
| `knowledge.repository.object.deleted`    | Delete           | `{id: str, ts: ISO8601}`                         |

**Performance contract**

- `find_by_id`: < 2ms (in-memory cache hit) / < 15ms (store fetch)
- `find_by_type` (1,000 results, paginated): < 50ms
- Unit of Work commit (10 objects): < 50ms
- Pre-write validation overhead: < 5ms per object

**Failure modes and recovery**

| Failure                      | Recovery                                          |
|------------------------------|---------------------------------------------------|
| Validation error on save     | Raise `KnowledgeValidationError`; do not persist  |
| Unit of Work partial failure | Roll back all writes in the UoW; raise error      |
| Concurrent write conflict    | Detect via version mismatch; raise `ConflictError`|

---

## Group B — Resolution & Registry

### B1. KnowledgeResolver

**Module identity**

| Field      | Value                    |
|------------|--------------------------|
| Name       | KnowledgeResolver        |
| Group      | B — Resolution & Registry|
| Class      | `KnowledgeResolver`      |
| Extends    | `PlatformService`        |
| Module ID  | KR-MOD-B1               |

**Responsibility**

- Resolves `ka://` URIs (e.g., `ka://DOM-KNOWLEDGE/specification/KA-SPEC-007`) and plain
  `KA-ARCH-001`-style IDs to full KnowledgeObjects using a two-level cache → repository
  lookup chain.
- Handles relative URI resolution within a domain context so that specs can reference each
  other with short paths (e.g., `../graph/KA-SPEC-008`).
- Maintains an LRU resolution cache with a configurable TTL (default: 5 minutes) per entry;
  cache is invalidated by `knowledge.store.written` events.
- Supports version-pinned URIs (`ka://domain/type/id@1.2.3`) by delegating version lookup to
  KnowledgeVersionManager.
- Returns typed resolution results including the resolved object and resolution metadata
  (cache hit vs. miss, latency, source).

**Key public interface**

```
class KnowledgeResolver(PlatformService):

    async def resolve(uri: str,
                      ctx: KnowledgeExecutionContext = None) -> ResolveResult
    async def resolve_many(uris: List[str],
                           ctx: KnowledgeExecutionContext = None) -> List[ResolveResult]
    async def resolve_relative(path: str,
                               base_domain: str) -> ResolveResult
    async def resolve_pinned(uri: str, version: str) -> ResolveResult

    def cache_stats() -> CacheStats
    def invalidate(knowledge_id: str) -> None
    def clear_cache() -> None

    def configure_ttl(ttl_seconds: int) -> None
    def configure_max_size(max_entries: int) -> None
```

**Dependencies**

- `KnowledgeRepository` (A3) — cache-miss fallback
- `KnowledgeVersionManager` (G1) — version-pinned resolution
- `KnowledgeCache` (K3) — hot-tier cache layer

**Events emitted**

| Event                         | Trigger             | Payload                                          |
|-------------------------------|---------------------|--------------------------------------------------|
| `knowledge.resolver.cache.hit`| Resolved from cache | `{uri: str, id: str, latency_us: int}`           |
| `knowledge.resolver.miss`     | Not in cache        | `{uri: str, latency_ms: int, found: bool}`       |
| `knowledge.resolver.error`    | Malformed URI       | `{uri: str, error: str}`                         |

**Performance contract**

- Cache hit: < 0.1ms (p99)
- Cache miss with repository lookup: < 5ms (p99)
- `resolve_many` (100 URIs, all cached): < 5ms total
- Cache max size (default): 5,000 entries

**Failure modes and recovery**

| Failure                   | Recovery                                                  |
|---------------------------|-----------------------------------------------------------|
| Malformed URI             | Return `ResolveResult(found=False, error=...)` immediately |
| Repository unreachable    | Return miss; emit error event; do not throw               |
| Circular alias chain      | Detect cycle at depth 10; raise `CircularReferenceError`  |

---

### B2. KnowledgeRegistry

**Module identity**

| Field      | Value                    |
|------------|--------------------------|
| Name       | KnowledgeRegistry        |
| Group      | B — Resolution & Registry|
| Class      | `KnowledgeRegistry`      |
| Extends    | `PlatformService` wrapping `PlatformRegistry` |
| Module ID  | KR-MOD-B2               |

**Responsibility**

- Maintains a thread-safe, in-memory catalog of every KnowledgeObject currently active in
  the running runtime, serving as the definitive source of truth for "what is registered".
- Maintains five indexes simultaneously: by-id (primary, O(1)), by-type, by-domain,
  by-capability, and by-tag (all secondary, O(1) lookup via inverted index).
- Triggers a pipeline on every registration: (1) re-index all five indexes, (2) signal
  KnowledgeGraphRuntime to rebuild graph edges, (3) update the HealthIndex.
- Supports weak references for objects with no active consumers so that the GC can reclaim
  memory on large registries without explicit eviction logic.
- Provides a Bloom filter for fast absent-ID rejection, preventing unnecessary index scans.

**Key public interface**

```
class KnowledgeRegistry(PlatformService):

    def register(obj: KnowledgeObject) -> RegistrationResult
    def unregister(knowledge_id: str) -> bool
    def get(knowledge_id: str) -> Optional[KnowledgeObject]
    def get_all() -> List[KnowledgeObject]

    def find_by_type(type_: KnowledgeType) -> List[KnowledgeObject]
    def find_by_domain(domain: str) -> List[KnowledgeObject]
    def find_by_capability(capability: str) -> List[KnowledgeObject]
    def find_by_tag(tag: str) -> List[KnowledgeObject]

    def contains(knowledge_id: str) -> bool       # Bloom filter fast path
    def size() -> int
    def index_stats() -> IndexStats

    def snapshot_ids() -> FrozenSet[str]          # point-in-time ID snapshot
    def iterate(predicate: Callable[[KnowledgeObject], bool]) -> Iterator[KnowledgeObject]
```

**Dependencies**

- `KnowledgeIndexer` (C1) — notified on registration to update persistent indexes
- `KnowledgeGraphRuntime` (D1) — notified to add/remove graph nodes
- `PlatformEventBus` — event emission

**Events emitted**

| Event                              | Trigger            | Payload                                             |
|------------------------------------|--------------------|-----------------------------------------------------|
| `knowledge.registry.registered`    | `register()` call  | `{id: str, type: str, domain: str, ts: ISO8601}`    |
| `knowledge.registry.unregistered`  | `unregister()` call| `{id: str, ts: ISO8601}`                            |
| `knowledge.registry.collision`     | Duplicate ID       | `{id: str, existing_version: str, new_version: str}`|

**Performance contract**

- `get()` by ID: O(1), < 0.05ms
- `register()`: O(log n) (Bloom filter + index update), < 1ms
- `find_by_domain()` (1,000 results): < 5ms
- Memory overhead per object: < 2KB (index entries only; object held by reference)

**Failure modes and recovery**

| Failure                    | Recovery                                               |
|----------------------------|--------------------------------------------------------|
| Duplicate registration     | Emit collision event; overwrite if version is higher   |
| Index out of sync          | Detect via periodic checksum; trigger full re-index    |
| Weak reference GC'd        | Re-fetch from KnowledgeStore on next `get()` call      |

---

## Group C — Indexing

### C1. KnowledgeIndexer

**Module identity**

| Field      | Value                |
|------------|----------------------|
| Name       | KnowledgeIndexer     |
| Group      | C — Indexing         |
| Class      | `KnowledgeIndexer`   |
| Extends    | `PlatformService`    |
| Module ID  | KR-MOD-C1           |

**Responsibility**

- Builds and maintains the seven runtime indexes defined in KA-SPEC-010: RegistryIndex,
  DomainIndex, CapabilityIndex, TypeIndex, HealthIndex, TagIndex, and TraceabilityIndex.
- Performs a full rebuild on boot by scanning all objects in KnowledgeStore, populating all
  seven indexes from scratch before emitting `knowledge.index.rebuilt`.
- Performs incremental updates by subscribing to `knowledge.store.written` and
  `knowledge.store.deleted` events, updating only the affected index entries (< 200ms).
- Persists all seven indexes to `dist/indexes/` in a binary format for warm-up on restart,
  avoiding a full rebuild if the persisted indexes are fresh (< 5 minutes old).
- Emits `knowledge.index.stale` if an incremental update is queued but not applied within
  the configured staleness threshold (default: 1 second).

**Key public interface**

```
class KnowledgeIndexer(PlatformService):

    async def rebuild_full() -> RebuildResult
    async def rebuild_index(index_type: IndexType) -> RebuildResult
    async def update_incremental(obj: KnowledgeObject,
                                 op: IndexOperation) -> UpdateResult

    def get_index(index_type: IndexType) -> KnowledgeIndex
    def is_stale() -> bool
    def staleness_ms() -> int
    def index_sizes() -> Dict[IndexType, int]

    async def persist_indexes() -> None
    async def load_persisted_indexes() -> LoadResult
    def index_checksum() -> str
```

**Dependencies**

- `KnowledgeStore` (A2) — scan source for full rebuild
- `KnowledgeRegistry` (B2) — post-rebuild registration
- `PlatformEventBus` — event subscription and emission
- `PlatformScheduler` — periodic staleness check

**Events emitted**

| Event                       | Trigger                      | Payload                                             |
|-----------------------------|------------------------------|-----------------------------------------------------|
| `knowledge.index.rebuilt`   | Full rebuild complete        | `{duration_ms: int, object_count: int, ts: ISO8601}`|
| `knowledge.index.updated`   | Incremental update applied   | `{id: str, index_type: str, op: str}`               |
| `knowledge.index.stale`     | Staleness threshold exceeded | `{staleness_ms: int, pending_updates: int}`         |
| `knowledge.index.persisted` | Indexes written to disk      | `{path: str, size_bytes: int}`                      |

**Performance contract**

- Full rebuild (10,000 objects): < 30 seconds
- Incremental update (single object): < 200ms
- Index warm-up from disk: < 5 seconds
- Index lookup (any type): O(1), < 0.1ms

**Failure modes and recovery**

| Failure                       | Recovery                                              |
|-------------------------------|-------------------------------------------------------|
| Incremental update fails      | Queue for retry; mark index stale; emit stale event   |
| Persisted index corrupt       | Detect via checksum on load; trigger full rebuild      |
| Rebuild aborted mid-way       | Discard partial result; retain last-good index         |

---

## Group D — Graph

### D1. KnowledgeGraphRuntime

**Module identity**

| Field      | Value                       |
|------------|-----------------------------|
| Name       | KnowledgeGraphRuntime       |
| Group      | D — Graph                   |
| Class      | `KnowledgeGraphRuntime`     |
| Extends    | `PlatformService`           |
| Module ID  | KR-MOD-D1                  |

**Responsibility**

- Maintains the authoritative in-memory property graph of all knowledge objects and their
  typed relationships, supporting up to 10,000 nodes and 100,000 edges.
- Represents nodes as `KnowledgeObject` instances augmented with runtime properties (type,
  domain, health score, lifecycle state, last-modified timestamp).
- Represents edges as `KnowledgeRelationship` instances carrying a typed relation
  (IMPLEMENTS, DEPENDS_ON, VALIDATES, EXTENDS, REFERENCES, SUPERSEDES, DERIVES_FROM),
  a weight (0.0–1.0), and arbitrary metadata.
- Rebuilds the full graph on boot from KnowledgeStore and thereafter applies incremental
  node/edge mutations triggered by `knowledge.store.written` events.
- Exposes a read-only query API for node, edge, and neighbour lookups; all write operations
  originate from KnowledgeRegistry or KnowledgeStore events.

**Key public interface**

```
class KnowledgeGraphRuntime(PlatformService):

    # Node operations
    def get_node(knowledge_id: str) -> Optional[GraphNode]
    def get_nodes_by_type(type_: KnowledgeType) -> List[GraphNode]
    def get_nodes_by_domain(domain: str) -> List[GraphNode]
    def node_count() -> int

    # Edge operations
    def get_edges(knowledge_id: str,
                  direction: EdgeDirection = BOTH) -> List[GraphEdge]
    def get_edge(from_id: str, to_id: str,
                 rel_type: RelationshipType = None) -> Optional[GraphEdge]
    def edge_count() -> int

    # Neighbourhood
    def neighbours(knowledge_id: str,
                   rel_type: RelationshipType = None) -> List[GraphNode]
    def in_neighbours(knowledge_id: str) -> List[GraphNode]
    def out_neighbours(knowledge_id: str) -> List[GraphNode]

    # Graph-level
    async def rebuild() -> GraphRebuildResult
    def graph_stats() -> GraphStats
    def adjacency_matrix(domain: str = None) -> AdjacencyMatrix
```

**Dependencies**

- `KnowledgeStore` (A2) — source for full rebuild
- `KnowledgeRegistry` (B2) — node registration events
- `KnowledgeTraversalEngine` (D2) — consumer (not a dependency of D1)
- `PlatformEventBus` — event subscription and emission

**Events emitted**

| Event                           | Trigger               | Payload                                               |
|---------------------------------|-----------------------|-------------------------------------------------------|
| `knowledge.graph.node.added`    | Node registered       | `{id: str, type: str, domain: str}`                   |
| `knowledge.graph.node.removed`  | Node unregistered     | `{id: str}`                                           |
| `knowledge.graph.edge.added`    | Relationship detected | `{from_id: str, to_id: str, rel_type: str, weight: float}` |
| `knowledge.graph.edge.removed`  | Relationship removed  | `{from_id: str, to_id: str, rel_type: str}`           |
| `knowledge.graph.rebuilt`       | Full rebuild done     | `{nodes: int, edges: int, duration_ms: int}`          |

**Performance contract**

- Node/edge lookup: O(1), < 0.1ms
- Neighbour enumeration (degree 100): < 1ms
- BFS traversal (10,000 nodes): < 50ms
- Full graph rebuild (10,000 nodes, 100,000 edges): < 20s
- Memory budget: < 500MB for maximum graph size

**Failure modes and recovery**

| Failure                    | Recovery                                                    |
|----------------------------|-------------------------------------------------------------|
| Memory limit approached    | Emit warning; evict cold nodes to disk-backed overflow      |
| Rebuild fails mid-way      | Retain last-good graph; schedule retry                      |
| Edge references missing node| Log warning; add ghost node; flag for validation            |

---

### D2. KnowledgeTraversalEngine

**Module identity**

| Field      | Value                        |
|------------|------------------------------|
| Name       | KnowledgeTraversalEngine     |
| Group      | D — Graph                    |
| Class      | `KnowledgeTraversalEngine`   |
| Extends    | `PlatformComponent`          |
| Module ID  | KR-MOD-D2                   |

**Responsibility**

- Executes graph traversal algorithms against KnowledgeGraphRuntime, providing both
  synchronous (small graphs) and async streaming (large traversals) interfaces.
- Implements: DFS, BFS, topological sort, Tarjan's SCC, Dijkstra's shortest path, and
  ancestor / descendant enumeration with configurable depth limits.
- Provides a dependency query API: forward (what does X depend on, transitively?) and
  reverse (what objects depend on X?), both with depth limits and domain filters.
- Computes blast-radius impact graphs: given a change to node X, returns the set of
  objects that would be affected along DEPENDS_ON edges, ranked by dependency distance.
- Detects cycles and returns a human-readable explanation naming the nodes that form each
  cycle, rather than just asserting a cycle exists.

**Key public interface**

```
class KnowledgeTraversalEngine(PlatformComponent):

    # Core traversals
    def bfs(start_id: str,
            max_depth: int = None,
            edge_filter: EdgeFilter = None) -> TraversalResult
    def dfs(start_id: str,
            max_depth: int = None,
            edge_filter: EdgeFilter = None) -> TraversalResult
    def topological_sort(domain: str = None) -> List[str]
    def strongly_connected_components() -> List[List[str]]

    # Path queries
    def shortest_path(from_id: str, to_id: str) -> Optional[GraphPath]
    def all_paths(from_id: str, to_id: str,
                  max_paths: int = 10) -> List[GraphPath]

    # Dependency queries
    def dependencies(knowledge_id: str,
                     transitive: bool = True,
                     depth: int = None) -> DependencyGraph
    def dependents(knowledge_id: str,
                   transitive: bool = True,
                   depth: int = None) -> DependencyGraph

    # Impact analysis
    def blast_radius(knowledge_id: str,
                     change_type: ChangeType = None) -> ImpactGraph

    # Cycle analysis
    def find_cycles() -> List[Cycle]
    def explain_cycle(cycle: Cycle) -> CycleExplanation

    # Streaming
    async def stream_bfs(start_id: str,
                         chunk_size: int = 100) -> AsyncIterator[List[GraphNode]]
```

**Dependencies**

- `KnowledgeGraphRuntime` (D1) — graph data source (required)
- `PlatformEventBus` — result streaming for large traversals

**Events emitted**

| Event                              | Trigger                    | Payload                                  |
|------------------------------------|----------------------------|------------------------------------------|
| `knowledge.traversal.started`      | Traversal begins           | `{algorithm: str, start_id: str}`        |
| `knowledge.traversal.chunk`        | Streaming chunk ready      | `{chunk_index: int, node_count: int}`    |
| `knowledge.traversal.cycle.found`  | Cycle detected             | `{cycle_nodes: List[str], length: int}`  |

**Performance contract**

- BFS/DFS (10,000 nodes): < 50ms
- Topological sort (10,000 nodes): < 100ms
- Shortest path: < 20ms
- Blast radius (1,000 affected nodes): < 100ms
- Streaming chunk delivery interval: < 10ms per chunk

**Failure modes and recovery**

| Failure                    | Recovery                                          |
|----------------------------|---------------------------------------------------|
| Depth limit exceeded       | Return partial result with `truncated=True` flag  |
| Graph modified during BFS  | Detect via version check; restart traversal once  |
| No path found              | Return `None` (not an error)                      |

---

## Group E — Intelligence

### E1. KnowledgeReasoningEngine

**Module identity**

| Field      | Value                        |
|------------|------------------------------|
| Name       | KnowledgeReasoningEngine     |
| Group      | E — Intelligence             |
| Class      | `KnowledgeReasoningEngine`   |
| Extends    | `PlatformService`            |
| Module ID  | KR-MOD-E1                   |

**Responsibility**

- Applies a forward-chaining rule engine to the registered knowledge graph, deriving new
  facts (relationships, coverage inferences, consistency violations) until no new facts can
  be derived (fixpoint).
- Supports four built-in rule categories: dependency propagation (infer transitive
  DEPENDS_ON edges), coverage inference (propagate coverage from implementation to
  architecture), consistency checking (detect contradictory status fields), and impact
  propagation (mark downstream objects IMPACTED when an upstream changes).
- Exposes an extension point for custom rules: plug-ins may register `KnowledgeRule`
  instances that participate in the forward-chaining iteration.
- Differentiates derived facts from asserted facts so that derived facts can be re-derived
  without loss; no derived fact is ever persisted to KnowledgeStore.
- Scheduled via `PlatformScheduler` for a full reasoning pass every configurable interval
  (default: 15 minutes) and triggered on-demand by `knowledge.graph.rebuilt`.

**Key public interface**

```
class KnowledgeReasoningEngine(PlatformService):

    async def run_full_pass() -> ReasoningResult
    async def run_incremental(changed_ids: List[str]) -> ReasoningResult

    def register_rule(rule: KnowledgeRule) -> None
    def unregister_rule(rule_id: str) -> bool
    def list_rules() -> List[RuleDescriptor]

    def derived_facts() -> List[DerivedFact]
    def derived_facts_for(knowledge_id: str) -> List[DerivedFact]
    def inconsistencies() -> List[Inconsistency]

    def stats() -> ReasoningStats    # iterations, facts derived, rules fired
    async def explain(fact: DerivedFact) -> ReasoningExplanation
```

**Dependencies**

- `KnowledgeGraphRuntime` (D1) — graph read access for rule evaluation
- `KnowledgeCoverageEngine` (H2) — coverage data for coverage-inference rules
- `PlatformEventBus` — event subscription (graph.rebuilt) and emission
- `PlatformScheduler` — scheduled full-pass trigger

**Events emitted**

| Event                                      | Trigger                     | Payload                                          |
|--------------------------------------------|-----------------------------|--------------------------------------------------|
| `knowledge.reasoning.fact.derived`         | New fact inferred           | `{fact_type: str, subject: str, object: str}`    |
| `knowledge.reasoning.inconsistency.detected`| Contradiction found        | `{subjects: List[str], rule: str, severity: str}`|
| `knowledge.reasoning.pass.completed`       | Full or incremental pass done| `{duration_ms: int, facts_derived: int, iterations: int}` |

**Performance contract**

- Full reasoning pass (10,000 objects, 50 rules): < 10 seconds
- Incremental pass (10 changed objects): < 500ms
- `derived_facts_for()` lookup: < 1ms

**Failure modes and recovery**

| Failure                  | Recovery                                              |
|--------------------------|-------------------------------------------------------|
| Rule throws exception    | Log rule error; skip rule for this iteration; continue|
| Fixpoint not reached     | Abort after 100 iterations; emit warning              |
| Inconsistency detected   | Log inconsistency; continue; surface via API          |

---

### E2. KnowledgeQueryEngine

**Module identity**

| Field      | Value                     |
|------------|---------------------------|
| Name       | KnowledgeQueryEngine      |
| Group      | E — Intelligence          |
| Class      | `KnowledgeQueryEngine`    |
| Extends    | `PlatformService`         |
| Module ID  | KR-MOD-E2                |

**Responsibility**

- Executes KQL (Knowledge Query Language, KA-SPEC-007) queries against the live runtime,
  operating a six-stage pipeline: Parse → Validate → Plan → Optimize → Execute → Stream.
- KQL is strictly read-only; the engine rejects any statement that would mutate state.
- The planner generates a cost-based query plan selecting between index scans, graph
  traversals, and full-table scans based on estimated cardinality.
- The optimizer applies predicate pushdown (filter before join), index selection (prefer
  O(1) index over scan), and traversal pruning (early termination if result cap reached).
- Streams large result sets as pages (default page size: 100 rows) via `AsyncIterator`;
  callers may also request a synchronous collect for small result sets.
- Maintains an LRU query cache keyed on the normalized query text; invalidated by
  `knowledge.index.updated` events.

**Key public interface**

```
class KnowledgeQueryEngine(PlatformService):

    async def execute(kql: str,
                      ctx: KnowledgeExecutionContext = None) -> QueryResult
    async def execute_paged(kql: str,
                            page: PageRequest,
                            ctx: KnowledgeExecutionContext = None) -> Page[QueryRow]
    async def stream(kql: str,
                     ctx: KnowledgeExecutionContext = None) -> AsyncIterator[QueryRow]

    def parse(kql: str) -> KQLAst
    def plan(ast: KQLAst) -> QueryPlan
    def explain(kql: str) -> QueryPlan         # plan without execution

    def cache_stats() -> QueryCacheStats
    def invalidate_cache() -> None
    async def validate(kql: str) -> List[KQLDiagnostic]
```

**Dependencies**

- `KnowledgeIndexer` (C1) — index access for fast-path queries
- `KnowledgeGraphRuntime` (D1) — graph traversal for TRAVERSE queries
- `KnowledgeTraversalEngine` (D2) — complex path queries
- `KnowledgeCache` (K3) — result caching
- `PlatformEventBus` — cache invalidation subscription

**Events emitted**

| Event                          | Trigger             | Payload                                                  |
|--------------------------------|---------------------|----------------------------------------------------------|
| `knowledge.query.executed`     | Query completes     | `{kql_hash: str, duration_ms: int, rows: int, plan: str}`|
| `knowledge.query.cache.hit`    | Cache serves result | `{kql_hash: str, latency_us: int}`                       |
| `knowledge.query.failed`       | Parse/exec error    | `{kql_hash: str, error: str, stage: str}`                |

**Performance contract**

- Simple SELECT query (index scan): < 50ms (p99)
- Complex TRAVERSE query (10 hops): < 500ms (p99)
- Cache hit: < 1ms (p99)
- Page delivery: < 10ms per page
- Query cache size (default): 500 entries, LRU eviction

**Failure modes and recovery**

| Failure                   | Recovery                                              |
|---------------------------|-------------------------------------------------------|
| Parse error               | Return typed diagnostic list; do not execute          |
| Timeout (default: 5s)     | Cancel execution; return `QueryResult(timeout=True)`  |
| Cache corrupt             | Discard cache; execute fresh; rebuild silently         |

---

## Group F — Compilation & Validation

### F1. KnowledgeCompiler

**Module identity**

| Field      | Value                  |
|------------|------------------------|
| Name       | KnowledgeCompiler      |
| Group      | F — Compilation & Validation |
| Class      | `KnowledgeCompiler`    |
| Extends    | `PlatformService`      |
| Module ID  | KR-MOD-F1             |

**Responsibility**

- Transforms KnowledgeObjects into output artifacts for all targets defined in KA-SPEC-009:
  Markdown, HTML, Wiki, PromptPack, LLMMemory, Dataset, and AIContext.
- Compilation is a pure transformation: the compiler never mutates KnowledgeObjects;
  output artifacts are written to `dist/compiled/{target}/` directories.
- Operates as an incremental pipeline: subscribes to `knowledge.store.written` and
  recompiles only affected objects and their transitive dependents.
- Supports full compilation runs scheduled via PlatformScheduler (default: on shutdown
  and on phase completion) and on-demand triggered compilation.
- Each compilation target is implemented as a `CompilerBackend` registered via an extension
  point, allowing new output formats without modifying the core compiler.

**Key public interface**

```
class KnowledgeCompiler(PlatformService):

    async def compile_all(targets: List[CompilerTarget] = None) -> CompilationResult
    async def compile_object(obj: KnowledgeObject,
                             targets: List[CompilerTarget] = None) -> CompilationResult
    async def compile_domain(domain: str,
                             targets: List[CompilerTarget] = None) -> CompilationResult

    def register_backend(backend: CompilerBackend) -> None
    def list_backends() -> List[BackendDescriptor]
    def get_backend(target: CompilerTarget) -> CompilerBackend

    async def clean_output(target: CompilerTarget = None) -> None
    def output_path(target: CompilerTarget) -> Path
    def last_compilation_result() -> Optional[CompilationResult]
```

**Dependencies**

- `KnowledgeRepository` (A3) — object retrieval
- `KnowledgeGraphRuntime` (D1) — dependency graph for incremental recompilation scope
- `PlatformEventBus` — store.written subscription and event emission
- `PlatformScheduler` — scheduled full compilation runs

**Events emitted**

| Event                             | Trigger                   | Payload                                             |
|-----------------------------------|---------------------------|-----------------------------------------------------|
| `knowledge.compiler.output.ready` | Target compilation done   | `{target: str, object_count: int, path: str}`       |
| `knowledge.compiler.started`      | Compilation begins        | `{targets: List[str], scope: str}`                  |
| `knowledge.compiler.failed`       | Backend error             | `{target: str, object_id: str, error: str}`         |

**Performance contract**

- Full compile (10,000 objects, all 7 targets): < 60 seconds
- Incremental compile (1 object, all targets): < 2 seconds
- Domain compile (1,000 objects, 1 target): < 10 seconds

**Failure modes and recovery**

| Failure                     | Recovery                                              |
|-----------------------------|-------------------------------------------------------|
| Backend throws for one obj  | Log error; emit failed event; continue other objects  |
| Output directory missing    | Create directories on start; warn if creation fails   |
| Full compile OOM            | Compile in domain batches; emit warning               |

---

### F2. KnowledgeValidator

**Module identity**

| Field      | Value                    |
|------------|--------------------------|
| Name       | KnowledgeValidator       |
| Group      | F — Compilation & Validation |
| Class      | `KnowledgeValidator`     |
| Extends    | `PlatformService`        |
| Module ID  | KR-MOD-F2               |

**Responsibility**

- Validates all KnowledgeObjects against the 34 schema and cross-reference rules defined
  in KA-STD-008, producing typed `Diagnostic` objects with severity, rule ID, and location.
- Classifies diagnostics by severity: ERROR (blocks approval and repository write),
  WARNING (logged and metriced but does not block), INFO (metric only, not logged by default).
- Runs automatically on every `KnowledgeRepository.save()` call to prevent invalid objects
  from reaching KnowledgeStore, and executes a full validation pass on boot.
- Supports custom rule registration so downstream packages can add domain-specific
  validation without modifying the core validator.
- Provides a `validate_batch()` API for efficient bulk validation during import or migration
  operations, running rules in parallel across objects.

**Key public interface**

```
class KnowledgeValidator(PlatformService):

    def validate(obj: KnowledgeObject) -> ValidationResult
    def validate_batch(objs: List[KnowledgeObject]) -> List[ValidationResult]
    async def validate_all() -> BulkValidationResult

    def register_rule(rule: ValidationRule) -> None
    def unregister_rule(rule_id: str) -> bool
    def list_rules() -> List[RuleDescriptor]
    def rule_by_id(rule_id: str) -> Optional[ValidationRule]

    def errors_for(knowledge_id: str) -> List[Diagnostic]
    def warnings_for(knowledge_id: str) -> List[Diagnostic]
    def validation_summary() -> ValidationSummary
```

**Dependencies**

- `KnowledgeRegistry` (B2) — cross-reference validation (referenced IDs must exist)
- `PlatformEventBus` — event emission

**Events emitted**

| Event                           | Trigger              | Payload                                                 |
|---------------------------------|----------------------|---------------------------------------------------------|
| `knowledge.validation.error`    | ERROR severity found | `{id: str, rule_id: str, message: str, field: str}`     |
| `knowledge.validation.warning`  | WARNING found        | `{id: str, rule_id: str, message: str}`                 |
| `knowledge.validation.passed`   | No errors            | `{id: str, warning_count: int}`                         |
| `knowledge.validation.full_pass.completed` | Boot pass done | `{errors: int, warnings: int, duration_ms: int}` |

**Performance contract**

- Single object validation: < 5ms (p99)
- Full pass (10,000 objects): < 10 seconds
- Batch validation (100 objects): < 200ms

**Failure modes and recovery**

| Failure                 | Recovery                                              |
|-------------------------|-------------------------------------------------------|
| Rule throws exception   | Log rule error; treat as WARNING; continue            |
| Cross-ref lookup fails  | Treat missing ID as ERROR; log                        |
| Full pass aborted       | Retain last-good summary; log abort reason            |

---

## Group G — Version Management

### G1. KnowledgeVersionManager

**Module identity**

| Field      | Value                       |
|------------|-----------------------------|
| Name       | KnowledgeVersionManager     |
| Group      | G — Version Management      |
| Class      | `KnowledgeVersionManager`   |
| Extends    | `PlatformService`           |
| Module ID  | KR-MOD-G1                  |

**Responsibility**

- Tracks the complete semantic version history of every KnowledgeObject; every write to
  KnowledgeStore creates an immutable version record archived in the version history.
- Enforces semantic versioning rules: MAJOR bumps for breaking schema changes, MINOR for
  backward-compatible additions, PATCH for corrections and metadata updates.
- Supports version pinning: callers can lock a consumer to a specific version via the URI
  syntax `ka://domain/type/id@1.2.3`, resolved by KnowledgeResolver.
- Provides version-range queries so that dependents can discover the latest version
  satisfying a constraint (e.g., `>=1.2.0 <2.0.0`).
- Applies a configurable retention policy (default: last 50 versions per object); older
  versions are archived to `dist/versions/archive/` rather than deleted outright.

**Key public interface**

```
class KnowledgeVersionManager(PlatformService):

    def record_version(obj: KnowledgeObject) -> VersionRecord
    def get_version(knowledge_id: str, version: str) -> Optional[KnowledgeObject]
    def list_versions(knowledge_id: str) -> List[VersionRecord]
    def latest_version(knowledge_id: str) -> Optional[VersionRecord]

    def pin(knowledge_id: str, version: str) -> PinRecord
    def unpin(knowledge_id: str) -> bool
    def get_pin(knowledge_id: str) -> Optional[PinRecord]

    def resolve_constraint(knowledge_id: str,
                           constraint: str) -> Optional[VersionRecord]
    def diff_versions(knowledge_id: str,
                      v1: str, v2: str) -> VersionDiff

    async def apply_retention_policy() -> RetentionResult
    def version_count(knowledge_id: str) -> int
```

**Dependencies**

- `KnowledgeStore` (A2) — version record persistence
- `PlatformEventBus` — event emission
- `PlatformScheduler` — retention policy runs

**Events emitted**

| Event                         | Trigger                  | Payload                                           |
|-------------------------------|--------------------------|---------------------------------------------------|
| `knowledge.version.bumped`    | New version recorded     | `{id: str, old: str, new: str, bump_type: str}`   |
| `knowledge.version.pinned`    | Pin created              | `{id: str, pinned_version: str}`                  |
| `knowledge.version.archived`  | Retention applied        | `{id: str, versions_archived: int}`               |

**Performance contract**

- `record_version()`: < 5ms
- `list_versions()` (50 versions): < 10ms
- `resolve_constraint()`: < 5ms
- Retention policy run (full catalog): < 60s

**Failure modes and recovery**

| Failure                  | Recovery                                            |
|--------------------------|-----------------------------------------------------|
| Version store corrupted  | Reconstruct from KnowledgeStore scan; log warning   |
| Constraint unsatisfiable | Return `None`; do not throw                         |
| Retention fails          | Log; retry on next scheduled run                    |

---

### G2. KnowledgeSnapshotManager

**Module identity**

| Field      | Value                        |
|------------|------------------------------|
| Name       | KnowledgeSnapshotManager     |
| Group      | G — Version Management       |
| Class      | `KnowledgeSnapshotManager`   |
| Extends    | `PlatformService`            |
| Module ID  | KR-MOD-G2                   |

**Responsibility**

- Creates and manages point-in-time snapshots of the entire knowledge graph: all objects,
  all indexes, and all derived state, compressed into a single archive.
- Triggers snapshot creation automatically on phase completion, on graceful shutdown, and
  on manual request; snapshots can also be triggered by external tools via the API.
- Supports full restores (replace entire state) and partial restores (single domain or
  capability) with conflict resolution against the current live state.
- Writes a manifest file at `dist/snapshots/SNAPSHOT-{timestamp}.manifest.yaml` describing
  the snapshot contents, checksum, and object counts per domain.
- Maintains a configurable snapshot retention policy (default: 30 snapshots) with the
  oldest snapshots pruned automatically.

**Key public interface**

```
class KnowledgeSnapshotManager(PlatformService):

    async def create(label: str = None,
                     reason: SnapshotReason = MANUAL) -> SnapshotResult
    async def restore(snapshot_id: str,
                      scope: SnapshotScope = FULL) -> RestoreResult
    async def restore_domain(snapshot_id: str,
                             domain: str) -> RestoreResult

    def list_snapshots() -> List[SnapshotManifest]
    def get_manifest(snapshot_id: str) -> Optional[SnapshotManifest]
    async def delete_snapshot(snapshot_id: str) -> bool
    async def verify_snapshot(snapshot_id: str) -> VerificationResult

    async def apply_retention_policy() -> int     # returns count pruned
    def snapshot_dir() -> Path
```

**Dependencies**

- `KnowledgeStore` (A2) — object source for snapshot creation
- `KnowledgeIndexer` (C1) — index state for snapshot
- `PlatformEventBus` — event emission
- `PlatformScheduler` — automated snapshot triggers

**Events emitted**

| Event                           | Trigger              | Payload                                              |
|---------------------------------|----------------------|------------------------------------------------------|
| `knowledge.snapshot.created`    | Snapshot written     | `{id: str, label: str, size_bytes: int, ts: ISO8601}`|
| `knowledge.snapshot.restored`   | Restore completes    | `{id: str, scope: str, object_count: int}`           |
| `knowledge.snapshot.failed`     | Create/restore error | `{id: str, error: str, stage: str}`                  |

**Performance contract**

- Create snapshot (10,000 objects): < 30 seconds
- Restore full snapshot: < 60 seconds
- Restore single domain (< 500 objects): < 10 seconds
- Manifest write: < 1 second

**Failure modes and recovery**

| Failure                  | Recovery                                             |
|--------------------------|------------------------------------------------------|
| Snapshot corrupt         | Detect via checksum; mark unusable; do not restore   |
| Disk full during create  | Abort; delete partial; emit error event              |
| Restore partial failure  | Roll back to pre-restore state via WAL               |

---

### G3. KnowledgeDiffEngine

**Module identity**

| Field      | Value                    |
|------------|--------------------------|
| Name       | KnowledgeDiffEngine      |
| Group      | G — Version Management   |
| Class      | `KnowledgeDiffEngine`    |
| Extends    | `PlatformComponent`      |
| Module ID  | KR-MOD-G3               |

**Responsibility**

- Computes structural diffs between two versions of a KnowledgeObject, two snapshots, or
  two subgraphs, yielding typed `DiffResult` objects rather than raw text patches.
- Supports three diff granularities: object diff (field-by-field comparison of frontmatter
  and sections), graph diff (added/removed/changed nodes and edges between snapshots), and
  coverage diff (change in coverage scores per domain and dimension).
- Produces both a machine-readable `StructuredDiff` for programmatic consumption (used by
  KnowledgeMergeEngine and impact analyzer) and a human-readable summary for display.
- Used by KnowledgeMergeEngine to detect conflicts before attempting a semantic merge.
- Does not persist diffs; results are computed on demand and returned to the caller.

**Key public interface**

```
class KnowledgeDiffEngine(PlatformComponent):

    def diff_objects(a: KnowledgeObject,
                     b: KnowledgeObject) -> ObjectDiff
    def diff_versions(knowledge_id: str,
                      v1: str, v2: str) -> ObjectDiff
    def diff_snapshots(snapshot_a: str,
                       snapshot_b: str) -> SnapshotDiff
    def diff_domains(domain: str,
                     snapshot_a: str,
                     snapshot_b: str) -> DomainDiff
    def diff_coverage(snapshot_a: str,
                      snapshot_b: str) -> CoverageDiff

    def summary(diff: ObjectDiff) -> str
    def is_breaking(diff: ObjectDiff) -> bool
    def affected_ids(diff: SnapshotDiff) -> Set[str]
```

**Dependencies**

- `KnowledgeVersionManager` (G1) — version retrieval for version-based diffs
- `KnowledgeSnapshotManager` (G2) — snapshot retrieval for snapshot diffs
- `KnowledgeCoverageEngine` (H2) — coverage data for coverage diffs

**Events emitted**

| Event                       | Trigger           | Payload                                        |
|-----------------------------|-------------------|------------------------------------------------|
| `knowledge.diff.computed`   | Diff completes    | `{type: str, added: int, removed: int, changed: int}` |

**Performance contract**

- Object diff (50 fields): < 5ms
- Snapshot diff (10,000 objects): < 30s
- Domain diff (500 objects): < 5s

**Failure modes and recovery**

| Failure                  | Recovery                                        |
|--------------------------|-------------------------------------------------|
| Version not found        | Return `DiffResult(error="version_not_found")`  |
| Snapshot inaccessible    | Return error result; do not throw               |

---

### G4. KnowledgeMergeEngine

**Module identity**

| Field      | Value                    |
|------------|--------------------------|
| Name       | KnowledgeMergeEngine     |
| Group      | G — Version Management   |
| Class      | `KnowledgeMergeEngine`   |
| Extends    | `PlatformComponent`      |
| Module ID  | KR-MOD-G4               |

**Responsibility**

- Merges divergent knowledge branches (e.g., two authors editing the same domain
  independently) into a single consistent state using one of three strategies.
- Supports strategies: `LAST_WRITE_WINS` (simple, no conflict detection), `SEMANTIC_MERGE`
  (field-level merge preserving non-conflicting changes from both sides), and
  `MANUAL_QUEUE` (conflicts pushed to a human-resolution queue).
- Delegates conflict detection to KnowledgeDiffEngine to produce a conflict report before
  attempting any merge; the merge is only attempted if conflicts are within the selected
  strategy's tolerance.
- Guarantees atomicity: either all objects from the source branch are merged successfully,
  or none are; uses a Unit of Work via KnowledgeRepository to ensure this.
- Merge results include a full audit record of which objects were merged, which were
  conflicted, and which merge strategy was applied per object.

**Key public interface**

```
class KnowledgeMergeEngine(PlatformComponent):

    async def merge(source: MergeSource,
                    target: MergeTarget,
                    strategy: MergeStrategy = SEMANTIC_MERGE) -> MergeResult
    async def merge_domain(domain: str,
                           source_snapshot: str,
                           strategy: MergeStrategy = SEMANTIC_MERGE) -> MergeResult

    def preview(source: MergeSource,
                target: MergeTarget) -> MergePreview
    def conflicts(source: MergeSource,
                  target: MergeTarget) -> List[MergeConflict]

    def list_pending_conflicts() -> List[MergeConflict]
    async def resolve_conflict(conflict_id: str,
                               resolution: ConflictResolution) -> bool
```

**Dependencies**

- `KnowledgeDiffEngine` (G3) — conflict detection
- `KnowledgeRepository` (A3) — atomic multi-object write via Unit of Work
- `PlatformEventBus` — event emission

**Events emitted**

| Event                               | Trigger              | Payload                                           |
|-------------------------------------|----------------------|---------------------------------------------------|
| `knowledge.merge.completed`         | Merge succeeds       | `{merged: int, conflicted: int, strategy: str}`   |
| `knowledge.merge.conflict.detected` | Conflict found       | `{conflict_id: str, object_id: str, fields: List[str]}` |
| `knowledge.merge.rolled_back`       | Merge aborted        | `{reason: str, rolled_back_count: int}`           |

**Performance contract**

- Merge (100 objects, no conflicts): < 10s
- Conflict detection: same as KnowledgeDiffEngine
- Manual queue depth (max pending conflicts): 1,000

**Failure modes and recovery**

| Failure                 | Recovery                                              |
|-------------------------|-------------------------------------------------------|
| Merge partially fails   | Roll back entire Unit of Work; emit rolled_back event |
| Manual queue full       | Reject new merges; emit warning; require resolution   |

---

## Group H — Evidence & Coverage

### H1. KnowledgeEvidenceEngine

**Module identity**

| Field      | Value                       |
|------------|-----------------------------|
| Name       | KnowledgeEvidenceEngine     |
| Group      | H — Evidence & Coverage     |
| Class      | `KnowledgeEvidenceEngine`   |
| Extends    | `PlatformService`           |
| Module ID  | KR-MOD-H1                  |

**Responsibility**

- Manages the complete lifecycle of evidence items attached to KnowledgeObjects, as defined
  in KA-SPEC-012; evidence types include: TEST_RESULT, CODE_REVIEW, AUDIT_TRAIL,
  BENCHMARK, and SCREENSHOT.
- Tracks four evidence states: PENDING (submitted, awaiting verification), VERIFIED
  (confirmed valid), EXPIRED (TTL elapsed), REVOKED (manually invalidated).
- Aggregates per-object confidence as the minimum confidence score across all VERIFIED
  evidence items; a single low-confidence evidence item lowers the overall score.
- Monitors evidence expiry via PlatformScheduler, transitioning VERIFIED items to EXPIRED
  when their configured TTL elapses and emitting `knowledge.evidence.expired`.
- Provides an audit trail of all evidence state transitions stored in KnowledgeAudit.

**Key public interface**

```
class KnowledgeEvidenceEngine(PlatformService):

    async def attach(evidence: EvidenceItem,
                     knowledge_id: str) -> EvidenceRecord
    async def verify(evidence_id: str,
                     verifier: str) -> EvidenceRecord
    async def revoke(evidence_id: str,
                     reason: str) -> EvidenceRecord

    def get_evidence(knowledge_id: str) -> List[EvidenceRecord]
    def get_evidence_by_type(knowledge_id: str,
                             type_: EvidenceType) -> List[EvidenceRecord]
    def confidence(knowledge_id: str) -> float       # 0.0 – 1.0
    def has_verified_evidence(knowledge_id: str) -> bool

    async def check_expiry() -> ExpiryCheckResult
    def evidence_summary() -> EvidenceSummary
```

**Dependencies**

- `KnowledgeRepository` (A3) — evidence records co-located with objects
- `KnowledgeAudit` (K1) — state-transition audit logging
- `PlatformEventBus` — event emission
- `PlatformScheduler` — expiry monitoring

**Events emitted**

| Event                          | Trigger                  | Payload                                            |
|--------------------------------|--------------------------|----------------------------------------------------|
| `knowledge.evidence.verified`  | Evidence verified        | `{evidence_id: str, object_id: str, type: str}`    |
| `knowledge.evidence.expired`   | TTL elapsed              | `{evidence_id: str, object_id: str, expired_at: ISO8601}` |
| `knowledge.evidence.revoked`   | Manual revocation        | `{evidence_id: str, object_id: str, reason: str}`  |

**Performance contract**

- `attach()`: < 10ms
- `confidence()` lookup: < 1ms (cached)
- Expiry check (10,000 evidence items): < 5s

**Failure modes and recovery**

| Failure                    | Recovery                                           |
|----------------------------|----------------------------------------------------|
| Verifier not authorized    | Raise `AuthorizationError`; do not update state    |
| Expiry check fails         | Log error; retry on next scheduled interval        |

---

### H2. KnowledgeCoverageEngine

**Module identity**

| Field      | Value                       |
|------------|-----------------------------|
| Name       | KnowledgeCoverageEngine     |
| Group      | H — Evidence & Coverage     |
| Class      | `KnowledgeCoverageEngine`   |
| Extends    | `PlatformService`           |
| Module ID  | KR-MOD-H2                  |

**Responsibility**

- Computes coverage scores across four dimensions defined in KA-SPEC-013: Architecture
  (knowledge objects exist), Implementation (code implements the spec), Testing (tests
  validate the implementation), and Desktop (desktop UX coverage).
- Coverage formula per dimension: `covered_objects / total_objects` scoped to a domain or
  capability; scores range from 0.0 to 1.0.
- Applies override rules: evidence-derived coverage (from KnowledgeEvidenceEngine) takes
  precedence over declared coverage values in frontmatter.
- Detects and emits coverage gaps whenever any domain or capability falls below the
  configured threshold (default: 0.80 for Architecture, 0.70 for all others).
- Feeds coverage data into KnowledgeReasoningEngine for coverage-inference rules.

**Key public interface**

```
class KnowledgeCoverageEngine(PlatformService):

    def coverage(domain: str,
                 dimension: CoverageDimension) -> float
    def coverage_by_capability(capability: str,
                               dimension: CoverageDimension) -> float
    def coverage_matrix(domain: str = None) -> CoverageMatrix
    def gaps(threshold: float = None) -> List[CoverageGap]

    async def recompute(domain: str = None) -> CoverageResult
    async def recompute_all() -> CoverageResult

    def set_threshold(domain: str,
                      dimension: CoverageDimension,
                      threshold: float) -> None
    def coverage_history(domain: str,
                         dimension: CoverageDimension,
                         limit: int = 30) -> List[CoverageSnapshot]
```

**Dependencies**

- `KnowledgeRegistry` (B2) — total object counts per domain
- `KnowledgeEvidenceEngine` (H1) — evidence-derived coverage overrides
- `PlatformEventBus` — event emission
- `PlatformScheduler` — periodic recompute

**Events emitted**

| Event                            | Trigger              | Payload                                                    |
|----------------------------------|----------------------|------------------------------------------------------------|
| `knowledge.coverage.gap.detected`| Score below threshold| `{domain: str, dimension: str, score: float, threshold: float}` |
| `knowledge.coverage.updated`     | Recompute completes  | `{domains_updated: int, ts: ISO8601}`                      |

**Performance contract**

- `coverage()` lookup (cached): < 1ms
- `recompute_all()` (10,000 objects): < 10s
- Gap detection: O(n) over domains, < 100ms for 100 domains

**Failure modes and recovery**

| Failure                | Recovery                                             |
|------------------------|------------------------------------------------------|
| Evidence engine down   | Fall back to declared coverage; log warning          |
| Recompute partial fail | Retain last-good scores; log domains that failed     |

---

## Group I — Recommendations

### I1. KnowledgeRecommendationEngine

**Module identity**

| Field      | Value                           |
|------------|---------------------------------|
| Name       | KnowledgeRecommendationEngine   |
| Group      | I — Recommendations             |
| Class      | `KnowledgeRecommendationEngine` |
| Extends    | `PlatformService`               |
| Module ID  | KR-MOD-I1                      |

**Responsibility**

- Generates actionable, prioritized recommendations derived from coverage gaps, reasoning
  inconsistencies, stale objects, and conflict queue depth.
- Supports four recommendation types: `FILL_COVERAGE_GAP` (create missing knowledge objects
  for uncovered capabilities), `ADD_EVIDENCE` (attach evidence to raise confidence),
  `UPDATE_STALE_OBJECT` (object not updated within staleness window), and
  `RESOLVE_CONFLICT` (pending merge conflicts require human resolution).
- Scores each recommendation with a priority formula: `priority = impact × urgency / effort`
  where impact, urgency, and effort are normalized 0.0–1.0 floats derived from gap
  severity, time since last update, and estimated work.
- Recommendations are `KnowledgeValueObject` instances: computed on demand, never persisted
  to KnowledgeStore, and regenerated on the next call if the underlying conditions change.
- Subscribes to coverage gap events and reasoning inconsistency events to invalidate the
  current recommendation cache immediately.

**Key public interface**

```
class KnowledgeRecommendationEngine(PlatformService):

    def recommend(domain: str = None,
                  limit: int = 20) -> List[Recommendation]
    def recommend_for(knowledge_id: str) -> List[Recommendation]
    def recommend_by_type(rec_type: RecommendationType) -> List[Recommendation]

    def top_recommendations(limit: int = 10) -> List[Recommendation]
    def recommendation_count() -> int
    def invalidate_cache() -> None

    def explain(recommendation: Recommendation) -> RecommendationExplanation
```

**Dependencies**

- `KnowledgeCoverageEngine` (H2) — gap data
- `KnowledgeReasoningEngine` (E1) — inconsistency data
- `KnowledgeMergeEngine` (G4) — conflict queue depth
- `KnowledgeRegistry` (B2) — staleness detection
- `PlatformEventBus` — event subscription and emission

**Events emitted**

| Event                                | Trigger               | Payload                                              |
|--------------------------------------|-----------------------|------------------------------------------------------|
| `knowledge.recommendation.generated` | Recommend call made   | `{count: int, top_priority: float, domain: str}`     |

**Performance contract**

- `recommend()` (fresh compute): < 500ms
- `recommend()` (cached): < 10ms
- Cache TTL: 5 minutes or until invalidated by event

**Failure modes and recovery**

| Failure                    | Recovery                                         |
|----------------------------|--------------------------------------------------|
| Dependency engine down     | Omit that recommendation type; log warning       |
| Empty recommendation set   | Return empty list; not an error                  |

---

## Group J — Context & Workspace

### J1. KnowledgeExecutionContext

**Module identity**

| Field      | Value                         |
|------------|-------------------------------|
| Name       | KnowledgeExecutionContext     |
| Group      | J — Context & Workspace       |
| Class      | `KnowledgeExecutionContext`   |
| Extends    | `PlatformValueObject`         |
| Module ID  | KR-MOD-J1                   |

**Responsibility**

- Provides a per-request, immutable execution scope carrying all cross-cutting concerns
  (identity, correlation, limits) that every runtime operation may inspect.
- Carries: `principal_id`, `session_id`, `workspace_id`, `correlation_id`, and `timestamp`
  for request attribution and distributed tracing.
- Encodes per-request limits that engines must respect: `max_results`, `max_depth`,
  `timeout_ms`, and `allowed_domains` (set of domains the caller may access).
- Once created, a context is immutable; callers create a modified copy via `with_*` builder
  methods rather than mutating the original.
- All runtime operations (query, traversal, resolve, compile) accept an optional context
  as a last parameter; if omitted, a default unrestricted context is used.

**Key public interface**

```
class KnowledgeExecutionContext(PlatformValueObject):

    # Factory
    @staticmethod
    def create(principal_id: str,
               session_id: str = None,
               workspace_id: str = None) -> KnowledgeExecutionContext
    @staticmethod
    def system() -> KnowledgeExecutionContext   # unrestricted system context

    # Builder (returns new instance)
    def with_correlation_id(correlation_id: str) -> KnowledgeExecutionContext
    def with_limits(max_results: int = None,
                    max_depth: int = None,
                    timeout_ms: int = None) -> KnowledgeExecutionContext
    def with_allowed_domains(domains: Set[str]) -> KnowledgeExecutionContext

    # Accessors
    @property
    def principal_id(self) -> str
    @property
    def session_id(self) -> Optional[str]
    @property
    def correlation_id(self) -> str
    @property
    def max_results(self) -> Optional[int]
    @property
    def max_depth(self) -> Optional[int]
    @property
    def timeout_ms(self) -> Optional[int]
    @property
    def allowed_domains(self) -> Optional[FrozenSet[str]]

    def is_domain_allowed(domain: str) -> bool
    def to_trace_dict() -> Dict[str, str]
```

**Dependencies**

- None (value object; no service dependencies)

**Events emitted**

- None (value object; emits no events)

**Performance contract**

- Construction: < 0.01ms
- All property accesses: O(1)

**Failure modes and recovery**

- N/A: immutable value object; invalid inputs raise `ValueError` at construction time.

---

### J2. KnowledgeSession

**Module identity**

| Field      | Value               |
|------------|---------------------|
| Name       | KnowledgeSession    |
| Group      | J — Context & Workspace |
| Class      | `KnowledgeSession`  |
| Extends    | `PlatformSession`   |
| Module ID  | KR-MOD-J2          |

**Responsibility**

- Provides a session-scoped, stateful view of the knowledge runtime for a single user or
  process; adds knowledge-specific session state on top of PlatformSession.
- Adds: `domain_filter` (restrict session to a subset of domains), `active_workspace`
  (the workspace currently associated with this session), `query_history` (ordered list of
  recent KQL queries), and `result_cache` (per-session LRU cache of recent results).
- Sessions are bounded by a configurable TTL (default: 8 hours); expired sessions are
  cleaned up by PlatformScheduler and emit `knowledge.session.expired`.
- Multiple sessions may coexist simultaneously, each with independent state; a session may
  be forked to create an independent snapshot of its state.
- Provides the execution context factory: `new_context()` creates a `KnowledgeExecutionContext`
  pre-populated with the session's `principal_id`, `session_id`, and domain filters.

**Key public interface**

```
class KnowledgeSession(PlatformSession):

    def new_context(correlation_id: str = None) -> KnowledgeExecutionContext
    def set_domain_filter(domains: Set[str]) -> None
    def clear_domain_filter() -> None

    def set_workspace(workspace: KnowledgeWorkspace) -> None
    @property
    def active_workspace(self) -> Optional[KnowledgeWorkspace]

    def record_query(kql: str, result_count: int) -> None
    @property
    def query_history(self) -> List[QueryHistoryEntry]

    def cache_result(key: str, result: Any, ttl_s: int = 300) -> None
    def get_cached_result(key: str) -> Optional[Any]

    def fork() -> KnowledgeSession
    def close() -> None
    @property
    def is_expired(self) -> bool
```

**Dependencies**

- `KnowledgeWorkspace` (J3) — optional workspace association
- `PlatformEventBus` — session lifecycle events
- `PlatformScheduler` — TTL expiry monitoring

**Events emitted**

| Event                        | Trigger             | Payload                                       |
|------------------------------|---------------------|-----------------------------------------------|
| `knowledge.session.started`  | Session created     | `{session_id: str, principal_id: str}`        |
| `knowledge.session.expired`  | TTL elapsed         | `{session_id: str, duration_s: int}`          |
| `knowledge.session.closed`   | Explicit close      | `{session_id: str}`                           |

**Performance contract**

- `new_context()`: < 0.1ms
- `cache_result()` / `get_cached_result()`: < 0.1ms
- Maximum concurrent sessions: 1,000

**Failure modes and recovery**

| Failure           | Recovery                                              |
|-------------------|-------------------------------------------------------|
| Session TTL races | PlatformScheduler serializes expiry; no double-expiry |

---

### J3. KnowledgeWorkspace

**Module identity**

| Field      | Value                  |
|------------|------------------------|
| Name       | KnowledgeWorkspace     |
| Group      | J — Context & Workspace|
| Class      | `KnowledgeWorkspace`   |
| Extends    | `PlatformWorkspace`    |
| Module ID  | KR-MOD-J3             |

**Responsibility**

- Provides a workspace-scoped knowledge collection, allowing a user or process to curate
  a focused subset of the global knowledge graph for a specific working context.
- Adds: `active_domains` (the domains in scope for this workspace), `pinned_objects`
  (objects always loaded regardless of access pattern), and a `workspace_index` (a local
  subset index populated on workspace load).
- Maintains a hot-knowledge cache: the N most recently accessed objects (default: 50) are
  kept warm in memory to provide sub-millisecond access without storage lookups.
- Workspaces can be saved to disk as a workspace descriptor file, allowing reuse across
  sessions.
- Emits `knowledge.workspace.loaded` when the workspace index is fully populated and ready
  for use.

**Key public interface**

```
class KnowledgeWorkspace(PlatformWorkspace):

    async def load(domains: List[str]) -> None
    async def save() -> Path
    @staticmethod
    async def open(path: Path) -> KnowledgeWorkspace

    def set_active_domains(domains: List[str]) -> None
    def add_domain(domain: str) -> None
    def remove_domain(domain: str) -> None

    def pin(knowledge_id: str) -> None
    def unpin(knowledge_id: str) -> None
    @property
    def pinned_objects(self) -> FrozenSet[str]

    def get(knowledge_id: str) -> Optional[KnowledgeObject]
    def hot_objects() -> List[KnowledgeObject]
    def workspace_size() -> int

    def search(query: str,
               limit: int = 20) -> List[KnowledgeObject]
```

**Dependencies**

- `KnowledgeRepository` (A3) — domain loading
- `KnowledgeIndexer` (C1) — workspace-local index population
- `KnowledgeCache` (K3) — hot-tier backing
- `PlatformEventBus` — event emission

**Events emitted**

| Event                         | Trigger               | Payload                                         |
|-------------------------------|-----------------------|-------------------------------------------------|
| `knowledge.workspace.loaded`  | `load()` completes    | `{domains: List[str], object_count: int}`       |
| `knowledge.workspace.saved`   | `save()` completes    | `{path: str}`                                   |

**Performance contract**

- `load()` (1 domain, 1,000 objects): < 5s
- `get()` (hot cache hit): < 0.1ms
- `search()` (workspace-local): < 100ms

**Failure modes and recovery**

| Failure               | Recovery                                          |
|-----------------------|---------------------------------------------------|
| Domain not found      | Log warning; skip domain; continue loading others |
| Workspace file corrupt| Raise `WorkspaceLoadError`; do not auto-recover   |

---

### J4. KnowledgeProject

**Module identity**

| Field      | Value                |
|------------|----------------------|
| Name       | KnowledgeProject     |
| Group      | J — Context & Workspace |
| Class      | `KnowledgeProject`   |
| Extends    | `PlatformProject`    |
| Module ID  | KR-MOD-J4          |

**Responsibility**

- Provides project-level knowledge context, associating a software project with the set of
  knowledge domains it depends on and the custom validation rules specific to that project.
- Adds: `knowledge_domains` (list of DOM-* domain identifiers this project engages),
  `custom_rules` (project-specific validation and reasoning rules), and a `project_workspace`
  (a pre-configured KnowledgeWorkspace for the project's scope).
- Feeds project-level knowledge into the AI context compilation pipeline, ensuring that
  compiled `AIContext` and `LLMMemory` outputs are scoped to the project's domains.
- Emits `knowledge.project.knowledge.updated` when any domain within the project's scope
  receives new or updated objects.
- Project configuration is persisted in the repository root as `.knowledge/project.yaml`.

**Key public interface**

```
class KnowledgeProject(PlatformProject):

    def add_domain(domain: str) -> None
    def remove_domain(domain: str) -> None
    @property
    def knowledge_domains(self) -> List[str]

    def register_custom_rule(rule: ValidationRule) -> None
    @property
    def custom_rules(self) -> List[ValidationRule]

    async def open_workspace() -> KnowledgeWorkspace
    async def compile_ai_context() -> Path

    @staticmethod
    async def load(root: Path) -> KnowledgeProject
    async def save() -> None
```

**Dependencies**

- `KnowledgeWorkspace` (J3) — workspace creation
- `KnowledgeCompiler` (F1) — AI context compilation
- `KnowledgeValidator` (F2) — custom rule registration
- `PlatformEventBus` — event subscription and emission

**Events emitted**

| Event                                    | Trigger                      | Payload                                  |
|------------------------------------------|------------------------------|------------------------------------------|
| `knowledge.project.knowledge.updated`    | Domain object changed        | `{domain: str, object_id: str}`          |
| `knowledge.project.loaded`               | `load()` completes           | `{domains: int, custom_rules: int}`      |

**Performance contract**

- `load()`: < 1s
- `open_workspace()`: same as KnowledgeWorkspace.load()
- `compile_ai_context()`: same as KnowledgeCompiler for selected domains

**Failure modes and recovery**

| Failure                    | Recovery                                           |
|----------------------------|----------------------------------------------------|
| Project config not found   | Create default; warn; do not throw                 |
| Domain not registered      | Log warning; skip in workspace load                |

---

## Group K — Observability

### K1. KnowledgeAudit

**Module identity**

| Field      | Value              |
|------------|--------------------|
| Name       | KnowledgeAudit     |
| Group      | K — Observability  |
| Class      | `KnowledgeAudit`   |
| Extends    | `PlatformService` wrapping `PlatformAudit` |
| Module ID  | KR-MOD-K1         |

**Responsibility**

- Records a durable, append-only audit trail for all knowledge-system actions that require
  accountability: object lifecycle, query execution, snapshot operations, and merge completions.
- Defines knowledge-specific audit action codes layered on top of PlatformAudit:
  `OBJECT_CREATED`, `OBJECT_APPROVED`, `OBJECT_DELETED`, `QUERY_EXECUTED`,
  `SNAPSHOT_CREATED`, `SNAPSHOT_RESTORED`, `MERGE_COMPLETED`, `EVIDENCE_VERIFIED`,
  `EVIDENCE_REVOKED`, `PERMISSION_DENIED`.
- Applies a configurable retention policy (default: 90 days); expired entries are archived
  to `dist/audit/archive/` rather than deleted.
- Exports audit records in JSONL format (for log pipelines) and CSV format (for compliance
  reporting); exports are streamed to avoid memory pressure on large audit logs.
- Provides tamper-detection: each audit record includes a hash chained to the previous
  record; any gap or hash mismatch is flagged on audit log verification.

**Key public interface**

```
class KnowledgeAudit(PlatformService):

    def record(action: AuditAction,
               principal_id: str,
               target_id: str,
               ctx: KnowledgeExecutionContext = None,
               metadata: Dict = None) -> AuditRecord

    def query(principal_id: str = None,
              action: AuditAction = None,
              target_id: str = None,
              since: datetime = None,
              limit: int = 100) -> List[AuditRecord]

    async def export_jsonl(path: Path,
                           since: datetime = None) -> int
    async def export_csv(path: Path,
                         since: datetime = None) -> int
    async def verify_integrity() -> IntegrityResult

    def retention_days() -> int
    async def apply_retention() -> int     # returns records archived
```

**Dependencies**

- `PlatformAudit` — base audit record infrastructure
- `PlatformEventBus` — listens for all emitted knowledge events to auto-record
- `PlatformScheduler` — retention policy runs

**Events emitted**

- None (KnowledgeAudit is a sink, not a source)

**Performance contract**

- `record()`: < 2ms (async, non-blocking to caller)
- `query()` (100 records): < 50ms
- Export (10,000 records, JSONL): < 30s
- Integrity verification (full log): < 60s

**Failure modes and recovery**

| Failure              | Recovery                                                 |
|----------------------|----------------------------------------------------------|
| Audit write fails    | Emit platform-level error; continue operation (non-blocking)|
| Hash chain break     | Flag in IntegrityResult; do not auto-repair              |
| Retention job fails  | Retry next scheduled run; log error                      |

---

### K2. KnowledgeMetrics

**Module identity**

| Field      | Value                |
|------------|----------------------|
| Name       | KnowledgeMetrics     |
| Group      | K — Observability    |
| Class      | `KnowledgeMetrics`   |
| Extends    | `PlatformMetrics`    |
| Module ID  | KR-MOD-K2           |

**Responsibility**

- Defines and maintains all knowledge-domain-specific metric names on top of the
  PlatformMetrics infrastructure, providing a typed API so modules emit metrics by name
  constant rather than raw string.
- Maintains counters (monotonically increasing integers), gauges (current values),
  and histograms (distribution of latency or size observations) as defined below.
- Exports in two formats: Prometheus text format (for scraping by a Prometheus server) and
  a JSON snapshot (for direct consumption by health dashboards).
- All metric names follow the prefix convention `knowledge_*` to prevent collision with
  Platform Kernel metrics.

**Counters**

| Name                           | Description                              |
|--------------------------------|------------------------------------------|
| `knowledge_objects_created`    | Total KnowledgeObjects created           |
| `knowledge_objects_deleted`    | Total KnowledgeObjects deleted           |
| `knowledge_queries_executed`   | Total KQL queries executed               |
| `knowledge_query_cache_hits`   | KQL cache hits                           |
| `knowledge_resolver_hits`      | Resolver cache hits                      |
| `knowledge_reasoning_rules_fired` | Total rule firings across all passes  |
| `knowledge_evidence_verified`  | Total evidence items verified            |
| `knowledge_snapshots_created`  | Total snapshots created                  |
| `knowledge_merges_completed`   | Total merge operations completed         |

**Gauges**

| Name                             | Description                            |
|----------------------------------|----------------------------------------|
| `knowledge_active_objects`       | Current registered object count        |
| `knowledge_graph_node_count`     | Current graph node count               |
| `knowledge_graph_edge_count`     | Current graph edge count               |
| `knowledge_index_freshness_ms`   | Milliseconds since last index update   |
| `knowledge_pending_conflicts`    | Merge conflicts awaiting resolution    |
| `knowledge_coverage_score`       | Aggregate coverage (labelled by domain)|

**Histograms**

| Name                             | Description                            |
|----------------------------------|----------------------------------------|
| `knowledge_query_duration_ms`    | KQL query execution latency (ms)       |
| `knowledge_traversal_depth`      | Graph traversal depth distribution     |
| `knowledge_reasoning_pass_ms`    | Full reasoning pass duration (ms)      |
| `knowledge_compile_duration_ms`  | Compilation time per target            |

**Key public interface**

```
class KnowledgeMetrics(PlatformMetrics):

    def increment(counter: KnowledgeCounter, by: int = 1) -> None
    def set_gauge(gauge: KnowledgeGauge, value: float) -> None
    def observe(histogram: KnowledgeHistogram, value: float) -> None

    def export_prometheus() -> str
    def export_json() -> Dict
    def snapshot() -> MetricsSnapshot
    def reset_counters() -> None    # for test isolation
```

**Dependencies**

- `PlatformMetrics` — base metric infrastructure

**Events emitted**

- None (KnowledgeMetrics is a sink)

**Performance contract**

- `increment()` / `set_gauge()` / `observe()`: < 0.01ms (lock-free counters)
- `export_prometheus()`: < 10ms (full catalog)

**Failure modes and recovery**

| Failure            | Recovery                                          |
|--------------------|---------------------------------------------------|
| Counter overflow   | Reset to 0 with overflow flag; log warning        |

---

### K3. KnowledgeCache

**Module identity**

| Field      | Value               |
|------------|---------------------|
| Name       | KnowledgeCache      |
| Group      | K — Observability   |
| Class      | `KnowledgeCache`    |
| Extends    | `PlatformComponent` |
| Module ID  | KR-MOD-K3          |

**Responsibility**

- Implements a three-tier LRU cache for KnowledgeObjects, providing sub-millisecond access
  for frequently used objects without hitting KnowledgeStore.
- Tier 1 — Hot (100 objects, in-memory): serves objects accessed in the last 60 seconds;
  eviction is strict LRU; target hit latency < 0.05ms.
- Tier 2 — Warm (10,000 objects, in-memory): serves moderately recent objects; eviction is
  LRU with domain-aware weighting (hot domains retain more entries); target < 0.1ms.
- Tier 3 — Cold (full catalog, disk-backed): compressed binary files written by
  KnowledgeStore; target < 5ms (same as store read).
- Invalidation triggers: `knowledge.store.written`, `knowledge.snapshot.restored`, and
  explicit `invalidate(id)` calls from KnowledgeResolver or KnowledgeRepository.

**Key public interface**

```
class KnowledgeCache(PlatformComponent):

    def get(knowledge_id: str) -> Optional[KnowledgeObject]
    def put(obj: KnowledgeObject) -> None
    def put_many(objs: List[KnowledgeObject]) -> None
    def invalidate(knowledge_id: str) -> None
    def invalidate_domain(domain: str) -> None
    def clear() -> None

    def hit_rate() -> float          # rolling 5-minute window
    def stats() -> CacheStats        # per-tier hit/miss/eviction counts
    def warm_up(ids: List[str]) -> int   # pre-populate from store
    def tier_sizes() -> Dict[str, int]   # hot/warm/cold current sizes
```

**Dependencies**

- `KnowledgeStore` (A2) — cold-tier backing and cache-miss fallback
- `PlatformEventBus` — invalidation event subscription

**Events emitted**

| Event                       | Trigger              | Payload                                         |
|-----------------------------|----------------------|-------------------------------------------------|
| `knowledge.cache.evicted`   | LRU eviction         | `{id: str, tier: str}`                          |
| `knowledge.cache.warmed`    | `warm_up()` completes| `{loaded: int, tier: str}`                      |

**Performance contract**

- Hot cache hit: < 0.05ms
- Warm cache hit: < 0.1ms
- Cold tier (disk): < 5ms
- Eviction overhead: < 0.01ms per entry

**Failure modes and recovery**

| Failure              | Recovery                                         |
|----------------------|--------------------------------------------------|
| Cache inconsistency  | `clear()` then `warm_up()`; log warning          |
| Hot/warm full        | LRU eviction is automatic; no explicit failure   |

---

## Group L — Security

### L1. KnowledgeSecurity

**Module identity**

| Field      | Value                  |
|------------|------------------------|
| Name       | KnowledgeSecurity      |
| Group      | L — Security           |
| Class      | `KnowledgeSecurity`    |
| Extends    | `PlatformService`      |
| Module ID  | KR-MOD-L1             |

**Responsibility**

- Provides knowledge-domain access control layered on top of PlatformSecurity, enforcing
  per-object and per-domain permissions for all runtime principals.
- Recognizes three principal types: `USER` (human), `SERVICE` (internal runtime service or
  tool), and `PLUGIN` (third-party extensions registered via extension point).
- Defines five permission levels: `READ_OBJECT`, `WRITE_OBJECT`, `APPROVE_OBJECT`,
  `EXPORT`, and `ADMIN`; permissions are granted per domain and optionally per object.
- `RESTRICTED` objects require an explicit `READ_RESTRICTED` grant in addition to the base
  `READ_OBJECT` permission; the flag is set in the object's frontmatter.
- All permission checks are logged to KnowledgeAudit with `PERMISSION_DENIED` action code
  for failed checks and the granted permission for successful sensitive operations.

**Key public interface**

```
class KnowledgeSecurity(PlatformService):

    def check(principal_id: str,
              permission: KnowledgePermission,
              target_id: str,
              ctx: KnowledgeExecutionContext = None) -> PermissionResult

    def grant(principal_id: str,
              permission: KnowledgePermission,
              scope: PermissionScope) -> GrantRecord
    def revoke(principal_id: str,
               permission: KnowledgePermission,
               scope: PermissionScope) -> bool

    def permissions_for(principal_id: str) -> List[GrantRecord]
    def can_read(principal_id: str, knowledge_id: str) -> bool
    def can_write(principal_id: str, knowledge_id: str) -> bool
    def can_approve(principal_id: str, knowledge_id: str) -> bool

    def assert_permission(principal_id: str,
                          permission: KnowledgePermission,
                          target_id: str) -> None   # raises AuthorizationError
```

**Dependencies**

- `PlatformSecurity` — base identity and grant infrastructure
- `KnowledgeAudit` (K1) — permission check logging
- `KnowledgeRegistry` (B2) — restricted-flag lookup

**Events emitted**

| Event                               | Trigger            | Payload                                          |
|-------------------------------------|--------------------|--------------------------------------------------|
| `knowledge.security.permission.denied` | Check fails     | `{principal: str, permission: str, target: str}` |
| `knowledge.security.grant.created`  | Grant added        | `{principal: str, permission: str, scope: str}`  |

**Performance contract**

- `check()`: < 0.5ms (in-memory policy evaluation)
- `grant()` / `revoke()`: < 5ms (policy store update)

**Failure modes and recovery**

| Failure                   | Recovery                                                 |
|---------------------------|----------------------------------------------------------|
| Policy store unavailable  | Deny all non-ADMIN operations (fail-closed)              |
| Principal not found       | Deny; emit permission.denied event                       |

---

## Group M — Events & Subscriptions

### M1. KnowledgeEvents

**Module identity**

| Field      | Value               |
|------------|---------------------|
| Name       | KnowledgeEvents     |
| Group      | M — Events & Subscriptions |
| Class      | `KnowledgeEvents`   |
| Extends    | N/A (constants module, no base class) |
| Module ID  | KR-MOD-M1          |

**Responsibility**

- Defines all 40+ event type string constants under the `knowledge.*` namespace; contains
  no logic, instantiation, or state — it is a pure constants module.
- Provides the canonical event type strings that all modules must import and use rather than
  inline strings, preventing typo-introduced mismatches between emitters and subscribers.
- Documents the `EventPriority` for each event type (CRITICAL, HIGH, NORMAL, LOW) which
  determines queue priority in PlatformEventBus.
- Defines the typed payload schema (as dataclasses or TypedDict) for each event so both
  emitters and subscribers share a single schema definition.

**Event type constants and priorities**

| Constant                                       | Priority | Payload Type                        |
|------------------------------------------------|----------|-------------------------------------|
| `knowledge.runtime.started`                    | HIGH     | `RuntimeStartedPayload`             |
| `knowledge.runtime.ready`                      | HIGH     | `RuntimeReadyPayload`               |
| `knowledge.runtime.stopping`                   | CRITICAL | `RuntimeStoppingPayload`            |
| `knowledge.runtime.error`                      | CRITICAL | `RuntimeErrorPayload`               |
| `knowledge.store.written`                      | NORMAL   | `StoreWrittenPayload`               |
| `knowledge.store.deleted`                      | NORMAL   | `StoreDeletedPayload`               |
| `knowledge.repository.object.created`          | NORMAL   | `ObjectCreatedPayload`              |
| `knowledge.repository.object.updated`          | NORMAL   | `ObjectUpdatedPayload`              |
| `knowledge.repository.object.deleted`          | NORMAL   | `ObjectDeletedPayload`              |
| `knowledge.resolver.cache.hit`                 | LOW      | `ResolverCacheHitPayload`           |
| `knowledge.resolver.miss`                      | LOW      | `ResolverMissPayload`               |
| `knowledge.registry.registered`                | NORMAL   | `RegistryRegisteredPayload`         |
| `knowledge.registry.unregistered`              | NORMAL   | `RegistryUnregisteredPayload`       |
| `knowledge.index.rebuilt`                      | HIGH     | `IndexRebuiltPayload`               |
| `knowledge.index.updated`                      | NORMAL   | `IndexUpdatedPayload`               |
| `knowledge.index.stale`                        | HIGH     | `IndexStalePayload`                 |
| `knowledge.graph.node.added`                   | NORMAL   | `GraphNodePayload`                  |
| `knowledge.graph.edge.added`                   | NORMAL   | `GraphEdgePayload`                  |
| `knowledge.graph.rebuilt`                      | HIGH     | `GraphRebuiltPayload`               |
| `knowledge.reasoning.fact.derived`             | LOW      | `FactDerivedPayload`                |
| `knowledge.reasoning.inconsistency.detected`   | HIGH     | `InconsistencyPayload`              |
| `knowledge.query.executed`                     | LOW      | `QueryExecutedPayload`              |
| `knowledge.compiler.output.ready`              | NORMAL   | `CompilerOutputPayload`             |
| `knowledge.compiler.failed`                    | HIGH     | `CompilerFailedPayload`             |
| `knowledge.validation.error`                   | HIGH     | `ValidationErrorPayload`            |
| `knowledge.validation.passed`                  | LOW      | `ValidationPassedPayload`           |
| `knowledge.version.bumped`                     | NORMAL   | `VersionBumpedPayload`              |
| `knowledge.snapshot.created`                   | NORMAL   | `SnapshotCreatedPayload`            |
| `knowledge.snapshot.restored`                  | HIGH     | `SnapshotRestoredPayload`           |
| `knowledge.diff.computed`                      | LOW      | `DiffComputedPayload`               |
| `knowledge.merge.completed`                    | NORMAL   | `MergeCompletedPayload`             |
| `knowledge.merge.conflict.detected`            | HIGH     | `MergeConflictPayload`              |
| `knowledge.evidence.verified`                  | NORMAL   | `EvidenceVerifiedPayload`           |
| `knowledge.evidence.expired`                   | NORMAL   | `EvidenceExpiredPayload`            |
| `knowledge.coverage.gap.detected`              | HIGH     | `CoverageGapPayload`                |
| `knowledge.coverage.updated`                   | NORMAL   | `CoverageUpdatedPayload`            |
| `knowledge.recommendation.generated`           | LOW      | `RecommendationPayload`             |
| `knowledge.session.started`                    | LOW      | `SessionStartedPayload`             |
| `knowledge.session.expired`                    | NORMAL   | `SessionExpiredPayload`             |
| `knowledge.cache.evicted`                      | LOW      | `CacheEvictedPayload`               |
| `knowledge.security.permission.denied`         | HIGH     | `PermissionDeniedPayload`           |

**Key public interface**

```
# This is a constants/schema module — no methods, only class-level constants

class KnowledgeEvents:
    RUNTIME_STARTED = "knowledge.runtime.started"
    RUNTIME_READY   = "knowledge.runtime.ready"
    STORE_WRITTEN   = "knowledge.store.written"
    # ... (all constants as above)

class EventPriority(Enum):
    CRITICAL = 0
    HIGH     = 1
    NORMAL   = 2
    LOW      = 3

# Payload dataclasses — one per event type
@dataclass
class StoreWrittenPayload:
    id: str
    version: str
    ts: str     # ISO 8601
    backend: str

# ... (one per event type listed in the table above)
```

**Dependencies**

- None

**Events emitted**

- None (constants module)

---

### M2. KnowledgeSubscriptions

**Module identity**

| Field      | Value                     |
|------------|---------------------------|
| Name       | KnowledgeSubscriptions    |
| Group      | M — Events & Subscriptions|
| Class      | `KnowledgeSubscriptions`  |
| Extends    | `PlatformService`         |
| Module ID  | KR-MOD-M2                |

**Responsibility**

- Provides typed, convenience subscription wrappers over the raw `PlatformEventBus` API,
  reducing subscription boilerplate from three parameters to one typed callback.
- Supports filtering subscriptions by domain, by knowledge type, and by lifecycle state
  so subscribers receive only events relevant to their context.
- Manages subscription lifecycle (register, cancel, list active) at the service level so
  that all subscriptions are cleanly cancelled on service shutdown without leaks.
- Provides subscription groups: a named group collects related subscriptions that can be
  cancelled together (e.g., cancel all subscriptions for a closed session).

**Key public interface**

```
class KnowledgeSubscriptions(PlatformService):

    # Typed subscription methods
    def on_object_created(cb: Callable[[ObjectCreatedPayload], None],
                          domain: str = None) -> SubscriptionHandle
    def on_object_updated(cb: Callable[[ObjectUpdatedPayload], None],
                          domain: str = None) -> SubscriptionHandle
    def on_graph_changed(cb: Callable[[GraphNodePayload], None]) -> SubscriptionHandle
    def on_index_updated(cb: Callable[[IndexUpdatedPayload], None]) -> SubscriptionHandle
    def on_reasoning_fact(cb: Callable[[FactDerivedPayload], None]) -> SubscriptionHandle
    def on_coverage_gap(cb: Callable[[CoverageGapPayload], None],
                        domain: str = None) -> SubscriptionHandle
    def on_conflict(cb: Callable[[MergeConflictPayload], None]) -> SubscriptionHandle

    # Raw filtered subscription
    def subscribe(event_type: str,
                  cb: Callable,
                  filter_: EventFilter = None) -> SubscriptionHandle

    # Lifecycle management
    def cancel(handle: SubscriptionHandle) -> bool
    def create_group(name: str) -> SubscriptionGroup
    def cancel_group(name: str) -> int       # returns count cancelled
    def active_subscriptions() -> List[SubscriptionDescriptor]
```

**Dependencies**

- `PlatformEventBus` — backing subscription infrastructure
- `KnowledgeEvents` (M1) — event type constants

**Events emitted**

- None (KnowledgeSubscriptions is a subscription manager, not an emitter)

**Performance contract**

- `subscribe()`: < 0.1ms
- Callback dispatch latency added vs. raw PlatformEventBus: < 0.01ms
- Max concurrent subscriptions per service: 10,000

**Failure modes and recovery**

| Failure                  | Recovery                                            |
|--------------------------|-----------------------------------------------------|
| Callback throws          | Log error; cancel subscription; do not propagate    |
| Subscription leak        | Audit on shutdown; cancel all orphaned handles      |

---

## Group N — Distribution

### N1. KnowledgeReplication

**Module identity**

| Field      | Value                   |
|------------|-------------------------|
| Name       | KnowledgeReplication    |
| Group      | N — Distribution        |
| Class      | `KnowledgeReplication`  |
| Extends    | `PlatformService`       |
| Module ID  | KR-MOD-N1              |

**Responsibility**

- Provides the architectural foundation for multi-node replication of knowledge objects,
  designed for Phase 2.0E; in Phase 2.0D.3A it operates in local-only mode with the full
  replication API surface exposed but no cross-node transport active.
- Implements a leader-follower model: one primary node accepts all write operations, N
  follower nodes receive changes via an append-only change log replicated through
  PlatformEventBus + durable persistence.
- Resolves write conflicts using vector clocks: each write carries a vector clock that
  allows the replication layer to determine causal order without a central coordinator.
- Exposes replication lag metrics (per-follower) and a replication health status that feeds
  into the overall runtime health report.
- Local-only mode: all write operations are accepted and immediately confirmed; the change
  log is maintained but not transmitted; switching to multi-node mode requires only a
  transport configuration change.

**Key public interface**

```
class KnowledgeReplication(PlatformService):

    def mode() -> ReplicationMode      # LOCAL | LEADER | FOLLOWER
    def is_writable() -> bool

    async def replicate(obj: KnowledgeObject) -> ReplicationResult
    async def replay_change_log(since: int = 0) -> int

    def lag_ms(follower_id: str = None) -> Optional[int]
    def replication_health() -> ReplicationHealth
    def change_log_size() -> int
    def change_log_head() -> int      # latest sequence number

    def configure_leader(host: str, port: int) -> None
    def configure_follower(follower_id: str, host: str, port: int) -> None
    def list_followers() -> List[FollowerDescriptor]
```

**Dependencies**

- `KnowledgeStore` (A2) — change log persistence
- `PlatformEventBus` — change propagation channel (local mode: no-op on publish)

**Events emitted**

| Event                              | Trigger              | Payload                                          |
|------------------------------------|----------------------|--------------------------------------------------|
| `knowledge.replication.lag.high`   | Lag > threshold      | `{follower_id: str, lag_ms: int}`                |
| `knowledge.replication.synced`     | Follower caught up   | `{follower_id: str}`                             |

**Performance contract**

- Local mode write (replication overhead): < 1ms additional latency
- Change log append: < 2ms
- Replay (10,000 entries): < 60s

**Failure modes and recovery**

| Failure                  | Recovery                                              |
|--------------------------|-------------------------------------------------------|
| Follower disconnects     | Buffer change log; replay on reconnect                |
| Leader unreachable       | Follower demotes to read-only; emit error             |
| Vector clock conflict    | Apply conflict resolution strategy; log               |

---

### N2. KnowledgeImportExport

**Module identity**

| Field      | Value                    |
|------------|--------------------------|
| Name       | KnowledgeImportExport    |
| Group      | N — Distribution         |
| Class      | `KnowledgeImportExport`  |
| Extends    | `PlatformService`        |
| Module ID  | KR-MOD-N2               |

**Responsibility**

- Imports KnowledgeObjects from external sources in supported formats and exports the
  knowledge graph to multiple output formats for sharing, backup, and cross-tool consumption.
- Import formats: YAML bundle (canonical, round-trips cleanly), JSON-LD (semantic web
  interoperability), CSV (tabular, for spreadsheet-driven workflows).
- Export formats: YAML bundle, JSON-LD, CSV, Markdown report (human-readable summary),
  and PromptPack (optimized for LLM consumption with token-budget awareness).
- Validates all imported objects via KnowledgeValidator before committing them to the
  repository; import is atomic per bundle (all-or-nothing).
- Streams large exports via `AsyncIterator` to avoid holding the full result set in memory;
  exports > 10,000 objects are always streamed.

**Key public interface**

```
class KnowledgeImportExport(PlatformService):

    # Import
    async def import_bundle(path: Path,
                            format_: ImportFormat,
                            dry_run: bool = False) -> ImportResult
    async def import_stream(stream: AsyncIterator[bytes],
                            format_: ImportFormat) -> ImportResult

    # Export
    async def export_all(path: Path,
                         format_: ExportFormat) -> ExportResult
    async def export_domain(domain: str,
                            path: Path,
                            format_: ExportFormat) -> ExportResult
    async def export_query(kql: str,
                           path: Path,
                           format_: ExportFormat) -> ExportResult
    async def stream_export(format_: ExportFormat,
                            domain: str = None) -> AsyncIterator[bytes]

    # Format support
    def supported_import_formats() -> List[ImportFormat]
    def supported_export_formats() -> List[ExportFormat]
    def validate_import(path: Path,
                        format_: ImportFormat) -> List[ValidationResult]
```

**Dependencies**

- `KnowledgeRepository` (A3) — commit point for imports
- `KnowledgeValidator` (F2) — pre-import validation
- `KnowledgeQueryEngine` (E2) — query-scoped exports
- `KnowledgeCompiler` (F1) — PromptPack and Markdown export backends
- `PlatformEventBus` — event emission

**Events emitted**

| Event                            | Trigger            | Payload                                              |
|----------------------------------|--------------------|------------------------------------------------------|
| `knowledge.import.completed`     | Import done        | `{format: str, objects: int, errors: int, path: str}`|
| `knowledge.export.completed`     | Export done        | `{format: str, objects: int, path: str, size_bytes: int}` |
| `knowledge.import.validation.failed` | Pre-import check fails | `{path: str, errors: int}`                |

**Performance contract**

- Import (1,000 objects, YAML bundle): < 30s
- Export (10,000 objects, YAML bundle): < 60s
- Streaming export: first chunk < 2s; subsequent chunks < 500ms
- Validation on import: same as KnowledgeValidator bulk

**Failure modes and recovery**

| Failure                      | Recovery                                              |
|------------------------------|-------------------------------------------------------|
| Import validation errors     | Abort entire import; report all errors; no partial commit |
| Export interrupted           | Delete partial file; emit error event                 |
| Unsupported format           | Raise `UnsupportedFormatError` immediately            |

---

## Dependency Matrix

The following table lists every module and the other modules it directly depends on at
runtime. Modules that depend on PlatformService infrastructure (EventBus, Scheduler) only
are marked as "Platform only" for conciseness.

| Module ID | Module Name                     | Direct Module Dependencies                              |
|-----------|---------------------------------|---------------------------------------------------------|
| A1        | KnowledgeRuntime                | All modules (owns lifecycle)                            |
| A2        | KnowledgeStore                  | Platform only                                           |
| A3        | KnowledgeRepository             | A2, F2                                                  |
| B1        | KnowledgeResolver               | A3, G1, K3                                              |
| B2        | KnowledgeRegistry               | C1, D1                                                  |
| C1        | KnowledgeIndexer                | A2, B2                                                  |
| D1        | KnowledgeGraphRuntime           | A2, B2                                                  |
| D2        | KnowledgeTraversalEngine        | D1                                                      |
| E1        | KnowledgeReasoningEngine        | D1, H2                                                  |
| E2        | KnowledgeQueryEngine            | C1, D1, D2, K3                                          |
| F1        | KnowledgeCompiler               | A3, D1                                                  |
| F2        | KnowledgeValidator              | B2                                                      |
| G1        | KnowledgeVersionManager         | A2                                                      |
| G2        | KnowledgeSnapshotManager        | A2, C1                                                  |
| G3        | KnowledgeDiffEngine             | G1, G2, H2                                              |
| G4        | KnowledgeMergeEngine            | G3, A3                                                  |
| H1        | KnowledgeEvidenceEngine         | A3, K1                                                  |
| H2        | KnowledgeCoverageEngine         | B2, H1                                                  |
| I1        | KnowledgeRecommendationEngine   | H2, E1, G4, B2                                          |
| J1        | KnowledgeExecutionContext       | None                                                    |
| J2        | KnowledgeSession                | J3                                                      |
| J3        | KnowledgeWorkspace              | A3, C1, K3                                              |
| J4        | KnowledgeProject                | J3, F1, F2                                              |
| K1        | KnowledgeAudit                  | Platform only                                           |
| K2        | KnowledgeMetrics                | Platform only                                           |
| K3        | KnowledgeCache                  | A2                                                      |
| L1        | KnowledgeSecurity               | K1, B2                                                  |
| M1        | KnowledgeEvents                 | None                                                    |
| M2        | KnowledgeSubscriptions          | M1                                                      |
| N1        | KnowledgeReplication            | A2                                                      |
| N2        | KnowledgeImportExport           | A3, F2, E2, F1                                          |

### Dependency graph summary (critical path)

```
KnowledgeStore (A2)
  └── KnowledgeRepository (A3) ──── KnowledgeValidator (F2) ─── KnowledgeRegistry (B2)
        │                                                               │
        ├── KnowledgeResolver (B1) ─── KnowledgeVersionManager (G1)   │
        │                                                               │
        └── KnowledgeCache (K3)        KnowledgeIndexer (C1) ──────────┘
                                              │
                                        KnowledgeGraphRuntime (D1)
                                              │
                              ┌───────────────┼───────────────────┐
                    D2.Traversal     E2.QueryEngine       E1.Reasoning
                                              │
                                    F1.Compiler   H2.Coverage   I1.Recommendations
```

---

## Appendix A — Event Payload Schemas

All payload fields follow these conventions:
- `ts` fields are ISO 8601 UTC strings (`2026-06-29T14:00:00Z`)
- `id` fields are knowledge IDs in `KA-SPEC-NNN` or `KR-MOD-XN` format
- `duration_ms` and `latency_ms` are integers (milliseconds)
- `latency_us` is an integer (microseconds)
- `size_bytes` is an integer

Key schemas not previously defined inline:

```
RuntimeReadyPayload:
  boot_duration_ms: int
  phase: "READY"
  module_count: int
  object_count: int

GraphRebuiltPayload:
  nodes: int
  edges: int
  duration_ms: int
  ts: str

CoverageGapPayload:
  domain: str
  capability: str | None
  dimension: "ARCHITECTURE" | "IMPLEMENTATION" | "TESTING" | "DESKTOP"
  score: float
  threshold: float
  gap_count: int

MergeConflictPayload:
  conflict_id: str
  object_id: str
  fields: List[str]
  strategy: str
  ts: str
```

---

## Appendix B — Performance Budget Summary

| Operation                                     | Target     | Measured At    |
|-----------------------------------------------|------------|----------------|
| Runtime boot (10,000 objects)                 | < 5s       | Wall clock      |
| Store read (in-memory)                        | < 1ms      | p99             |
| Store write (file system)                     | < 10ms     | p99             |
| Registry lookup by ID                         | < 0.05ms   | p99             |
| Resolver cache hit                            | < 0.1ms    | p99             |
| Resolver cache miss                           | < 5ms      | p99             |
| Index full rebuild (10,000 objects)           | < 30s      | Wall clock      |
| Index incremental update (1 object)           | < 200ms    | p99             |
| Graph BFS/DFS (10,000 nodes)                  | < 50ms     | p99             |
| Graph rebuild (10,000 nodes)                  | < 20s      | Wall clock      |
| KQL simple query                              | < 50ms     | p99             |
| KQL complex traversal query                   | < 500ms    | p99             |
| KQL cache hit                                 | < 1ms      | p99             |
| Full reasoning pass (10,000 objects, 50 rules)| < 10s      | Wall clock      |
| Full compile (10,000 objects, all 7 targets)  | < 60s      | Wall clock      |
| Single object validation                      | < 5ms      | p99             |
| Snapshot create (10,000 objects)              | < 30s      | Wall clock      |
| Snapshot restore (full)                       | < 60s      | Wall clock      |
| Hot cache hit                                 | < 0.05ms   | p99             |
| Warm cache hit                                | < 0.1ms    | p99             |
| Import bundle (1,000 objects)                 | < 30s      | Wall clock      |
| Export all (10,000 objects, YAML)             | < 60s      | Wall clock      |

---

*End of KR-ARCH-004 — Knowledge Runtime Complete Module Catalog*
*31 modules specified across 14 groups*
*Document owner: Chief Knowledge Architect | Phase: 2.0D.3A | Status: approved*
