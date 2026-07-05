---
knowledge_id: KR-ARCH-005
version: "1.0.0"
status: approved
owner: Chief Knowledge Architect
phase: 2.0D.3A
created: 2026-06-29
review_date: 2026-12-29
canonical: true
domain: DOM-KNOWLEDGE
capability: knowledge-runtime
type: architecture
depends_on:
  - id: KA-SPEC-020
    reason: "Extends and refines the Knowledge Runtime Architecture blueprint"
  - id: KA-KIP-002
    reason: "Boot sequence initializes KnowledgeRegistry, KnowledgeRepository, KnowledgeResolver"
  - id: KA-KIP-003
    reason: "Phase 4 boot initializes KnowledgeGraphRuntime"
  - id: KA-KIP-004
    reason: "Phase 4 boot initializes KnowledgeQueryEngine"
implements:
  - KA-SPEC-001
---

# Knowledge Runtime — Execution Model

## Lifecycle, Boot Sequence, Caching, Recovery, and Shutdown

---

## 1. Overview

The Knowledge Runtime (phase 2.0D.3A) runs as a managed subsystem of the Platform Kernel. It is not a standalone daemon; it is a first-class runtime registered with `PlatformRegistry` and driven by `PlatformLifecycle` state transitions.

This document specifies the **execution model**: how the runtime boots, manages state, caches data, handles incremental changes, recovers from failures, and shuts down cleanly. It does not respecify the internal logic of individual components — those are covered by their own canonical specifications.

### 1.1 Runtime Identity

```
Runtime Name : KnowledgeRuntime
Registry Key : knowledge-runtime
Lifecycle    : Managed by PlatformLifecycle
Diagnostics  : Reports to PlatformDiagnostics
Scheduling   : Uses PlatformScheduler for background tasks
Phase        : 2.0D.3A
```

### 1.2 Runtime States

```
                     ┌─────────────────────────────────────────┐
                     │                                         │
  UNINITIALIZED ──► BOOTING ──► RUNNING ──► STOPPING ──► STOPPED
                       │                       ▲
                       │ (failure)             │ (manual trigger)
                       ▼                       │
                   DEGRADED ──────────────────►┘
                       │
                       ▼ (unrecoverable)
                    FAILED
```

| State | Description |
|-------|-------------|
| `UNINITIALIZED` | Runtime registered but boot not yet triggered |
| `BOOTING` | Boot sequence in progress; read-only from last snapshot |
| `RUNNING` | Fully operational; all services available |
| `STOPPING` | Draining in-flight queries; preparing shutdown |
| `STOPPED` | All services halted; WAL and indexes flushed |
| `DEGRADED` | Partial failure; read-only from last good checkpoint |
| `FAILED` | Unrecoverable error; runtime must be restarted |

---

## 2. Boot Sequence

The boot sequence is divided into five sequential phases. Each phase has a hard time budget. Failure to complete within budget triggers a timeout event and transitions the runtime to `DEGRADED`.

**Total boot target: < 5 seconds for 10,000 objects**

```
t=0ms        t=100ms      t=500ms      t=2500ms     t=5500ms     t=6500ms
  │            │            │            │            │            │
  ▼            ▼            ▼            ▼            ▼            ▼
PHASE 1      PHASE 2      PHASE 3      PHASE 4      PHASE 5      RUNNING
Foundation   Storage      Index        Intelligence Services
< 100ms      < 500ms      < 2000ms     < 3000ms     < 1000ms
```

### 2.1 Phase 1 — Foundation (< 100ms)

Establishes the observability and platform scaffolding before any domain logic runs. All subsequent phases depend on these services being ready.

| Step | Component | Action |
|------|-----------|--------|
| 1.1 | `KnowledgeAudit` | Initialize audit log; open audit store |
| 1.2 | `KnowledgeMetrics` | Register metric counters with `PlatformDiagnostics` |
| 1.3 | `KnowledgeCache` | Allocate three-tier cache structure (see Section 7) |
| 1.4 | `PlatformRegistry` | Register `KnowledgeRuntime` entry with health probe callback |
| 1.5 | `PlatformDiagnostics` | Begin emitting `knowledge.runtime.heartbeat` every 5s |

**Completion signal:** `knowledge.runtime.boot.phase1.complete` (persistent=False)

Phase 1 has no dependencies on the file system or storage. If Phase 1 fails, the runtime transitions to `FAILED` immediately — no recovery is possible without observability.

### 2.2 Phase 2 — Storage (< 500ms)

Opens and validates the persistent storage layer. Must succeed before any objects can be loaded.

| Step | Component | Action |
|------|-----------|--------|
| 2.1 | `KnowledgeStore` | Open Write-Ahead Log (WAL); verify WAL integrity |
| 2.2 | `KnowledgeStore` | Run WAL integrity check; detect uncommitted entries |
| 2.3 | `KnowledgeRepository` | Initialize root-path binding; do not scan yet |
| 2.4 | `KnowledgeVersionManager` | Load version manifest from disk |

**Completion signal:** `knowledge.runtime.boot.phase2.complete` (persistent=False)

If the WAL has uncommitted entries, Phase 2 sets a `RECOVERY_NEEDED` flag before completing. Recovery is deferred to after Phase 3 (indexes are required for WAL replay).

### 2.3 Phase 3 — Index (< 2s for 10,000 objects)

Builds or warms the in-memory indexes that back all subsequent operations.

**Fast path (indexes are fresh):**
```
WARM from persisted indexes
  ├── Load index.yaml → KnowledgeRegistry  (O(n))
  ├── Load capability-index.yaml → KnowledgeCatalog
  ├── Load traceability-index.yaml → TraceabilityIndex
  └── Validate freshness timestamps against WAL

Target: < 500ms for 10,000 objects (IO-bound)
```

**Slow path (indexes are stale or missing):**
```
FULL REBUILD
  ├── Scan all *.md files under root
  ├── Parse frontmatter → KnowledgeObjects
  ├── Two-pass dependency resolution (see Section 3)
  ├── Build all seven index structures
  └── Serialize to disk

Target: < 2000ms for 10,000 objects (CPU + IO bound)
```

| Step | Component | Depends on |
|------|-----------|-----------|
| 3.1 | `KnowledgeIndexer` | `KnowledgeStore` ready |
| 3.2 | `KnowledgeResolver` | `KnowledgeIndexer` pass-1 complete |
| 3.3 | `KnowledgeRegistry` | `KnowledgeIndexer` pass-2 complete |

**Completion signal:** `knowledge.runtime.boot.phase3.complete` (persistent=False)

Metric emitted: `knowledge.index.object_count`, `knowledge.index.build_ms`

### 2.4 Phase 4 — Intelligence (< 3s)

Builds the graph and activates all intelligence engines. Steps 4.1 and 4.2 are sequential (graph before traversal); steps 4.3 through 4.5 can begin in parallel once the graph is ready.

```
4.1 KnowledgeGraphRuntime  ── build from index ──────────────── (sequential)
4.2 KnowledgeTraversalEngine ── bind to graph ─────────────────  (sequential)
    │
    ├── 4.3 KnowledgeQueryEngine    ── warm query plan cache ──┐ (parallel)
    ├── 4.4 KnowledgeReasoningEngine ── load inference rules ──┤ (parallel)
    ├── 4.5 KnowledgeEvidenceEngine ── load evidence registry ─┤ (parallel)
    └── 4.6 KnowledgeCoverageEngine ── load coverage data ─────┘ (parallel)
```

**Completion signal:** `knowledge.runtime.boot.phase4.complete` (persistent=False)

### 2.5 Phase 5 — Services (< 1s)

Activates all remaining domain services and registers health checks. All services in Phase 5 can initialize in parallel (no inter-service dependencies within this phase).

| Service | Role |
|---------|------|
| `KnowledgeCompiler` | Multi-target document compilation |
| `KnowledgeValidator` | Schema and integrity validation |
| `KnowledgeSnapshotManager` | Snapshot creation and restoration |
| `KnowledgeDiffEngine` | Inter-version diff computation |
| `KnowledgeMergeEngine` | Branch merge operations |
| `KnowledgeRecommendationEngine` | Authoring recommendations |
| `KnowledgeSecurityService` | Access control enforcement |
| `KnowledgeSubscriptionService` | Event subscription management |
| `KnowledgeReplicationClient` | Upstream replication sync |
| `KnowledgeImportExportService` | Bulk import/export operations |

After all services register their health checks with `PlatformRegistry`:

**Completion signal:** `knowledge.runtime.boot.complete` (persistent=**True**)

**Lifecycle transition:** `PlatformLifecycle` → `RUNNING`

---

## 3. Dependency Resolution on Boot

Forward references are a structural property of the knowledge graph: object A may declare a relationship to object B before B has been indexed. Naive single-pass indexing would silently drop or flag these as errors. The runtime uses a **two-pass resolution protocol** to handle them correctly.

### 3.1 Pass 1 — Registration

```
FOR EACH file in scan order:
  PARSE frontmatter → extract knowledge_id, type, domain, capability
  REGISTER knowledge_id in KnowledgeRegistry (object shell — no relationships yet)
  ADD to by_type, by_domain, by_capability catalog indexes
  SET resolution_state = REGISTERED

After pass 1: all IDs are known; no relationships are resolved
```

### 3.2 Pass 2 — Resolution

```
FOR EACH registered object:
  FOR EACH declared relationship (depends_on, implements, ...):
    IF target_id IN KnowledgeRegistry:
      CREATE edge in TraceabilityIndex
      SET resolution_state = RESOLVED
    ELSE:
      EMIT KnowledgeValidator WARNING: "Unresolvable reference: {target_id}"
      SET resolution_state = UNRESOLVED_REF
      (Do not fail boot — emit warning only)
```

### 3.3 Post-Resolution Integrity Check

After both passes complete:

```
FOR EACH edge in TraceabilityIndex:
  ASSERT edge.source IN KnowledgeRegistry
  ASSERT edge.target IN KnowledgeRegistry
  IF assertion fails: EMIT KnowledgeValidator ERROR (runtime integrity violation)

RESULT: all traversable relationships are referentially sound
        unresolvable references are warned but do not block RUNNING state
```

The distinction between WARNING (first boot with missing targets) and ERROR (integrity violation post-resolution) prevents false boot failures when documents reference objects yet to be added to the repository.

---

## 4. Graph Loading

The Knowledge Graph is populated from the index state established in Phase 3. Graph loading is the first step of Phase 4.

### 4.1 Loading Sequence

```
Step 1 — Node Creation
  FOR EACH KnowledgeObject in KnowledgeRegistry.all():
    CREATE KnowledgeGraphNode:
      node_id         = object.id
      object_type     = object.type
      domain          = object.domain
      lifecycle_state = object.status
      health_score    = object.health_score
      coverage        = object.metadata.coverage (or defaults)
      properties      = object.metadata (full)

Step 2 — Edge Creation (nodes must exist before edges)
  FOR EACH relationship in TraceabilityIndex:
    CREATE KnowledgeGraphEdge:
      source_id         = relationship.from_id
      target_id         = relationship.to_id
      relationship_type = relationship.type
      weight            = relationship.weight (default 1.0)
      metadata          = relationship.properties

  FOR EACH capability_link in CapabilityIndex:
    CREATE KnowledgeGraphEdge (COVERS, TRACES_TO as applicable)

Step 3 — Structural Analysis
  RUN Tarjan's SCC on full graph
  FOR EACH SCC with |SCC| > 1:
    EMIT knowledge.graph.cycle_detected {scc: [node_ids], size: n}
    LOG WARNING (do not fail — cycles are reported, not rejected)

Step 4 — Health Score Propagation
  RUN health propagation (see 06-GRAPH-RUNTIME.md §4)
  UPDATE node.health_score for all nodes affected by dependency degradation

Step 5 — Emit Completion
  EMIT knowledge.graph.loaded {node_count: n, edge_count: m, scc_count: k}
```

### 4.2 Load Order Invariant

Nodes **must** be created before edges. The runtime enforces this by separating the two loops and asserting node existence before each edge creation. A missing node reference at edge creation time is an internal consistency error (not a WARNING) because it means the index is out of sync with the registry.

---

## 5. Incremental Indexing

After boot, the runtime maintains index freshness through incremental updates triggered by storage events.

### 5.1 Trigger

```
Event: knowledge.store.written
  Payload: { object_id, change_type: [CREATED | UPDATED | DELETED] }
```

### 5.2 Debounce Window

Multiple rapid writes within a 100ms window are batched into a single incremental update cycle. This prevents thrashing when an editor saves a document multiple times in quick succession.

```
DEBOUNCE: 100ms
BATCH: collect all knowledge.store.written events within window
PROCESS: single incremental update cycle for the entire batch
```

### 5.3 Incremental Update Cycle

```
FOR EACH changed_object_id in batch:
  1. RELOAD object from KnowledgeStore
  2. RE-VALIDATE frontmatter (schema check)
  3. UPDATE KnowledgeRegistry entry
  4. UPDATE catalog indexes (by_type, by_domain, by_capability)
  5. RECOMPUTE direct relationships:
       a. REMOVE all edges where source_id = changed_object_id
       b. RE-RUN Pass 2 resolution for changed_object_id only
       c. ADD new edges
  6. UPDATE KnowledgeGraphRuntime:
       a. UPDATE node properties
       b. SWAP edges for this node
  7. INVALIDATE query cache entries:
       query_cache.invalidate_by_object(changed_object_id)

EMIT knowledge.index.updated { updated_ids: [...], duration_ms: n }
```

### 5.4 Scope

Incremental indexing operates **only** on the changed objects and their direct relationships. It does not re-run SCC, full health propagation, or rule inference. Those are triggered separately by `knowledge.index.major_change` if the incremental delta exceeds a configurable threshold (default: 10% of total objects changed in a 60-second window).

---

## 6. Lazy Loading

Objects that were not accessed during boot are available for on-demand loading. Lazy loading coexists with the three-tier cache model.

### 6.1 Existence Check

Before any disk access, `KnowledgeResolver` uses a **Bloom filter** to determine whether an ID might exist:

```
FUNCTION resolve_lazy(id: str) -> KnowledgeObject | None:
  IF id IN hot_tier OR id IN warm_tier:
    RETURN cache.get(id)
  IF NOT bloom_filter.contains(id):
    RETURN None  # Definitely absent — no disk access
  # Probable hit — access disk
  obj = KnowledgeStore.load(id)
  IF obj is None:
    bloom_filter.update()  # False positive — refresh filter
    RETURN None
  VALIDATE obj against schema
  RESOLVE all relationships for obj (pass-2 resolution, single object)
  cache.put(obj, tier=WARM)
  RETURN obj
```

### 6.2 Relationship Resolution on Lazy Load

When an object is lazy-loaded, all its declared relationships are resolved immediately against the current registry state. This ensures that any traversal starting from a lazy-loaded object is complete. Unresolvable references at lazy-load time emit DEBUG-level events (not WARNINGs, because the system is already RUNNING and forward references at boot were already reported).

---

## 7. Caching

The runtime uses a **three-tier cache** to balance memory consumption against access latency.

### 7.1 Tier Definitions

| Tier | Name | Capacity | Storage | Eviction | Latency |
|------|------|----------|---------|----------|---------|
| L1 | Hot | 100 objects | In-memory dict | LRU (fixed size) | < 1μs |
| L2 | Warm | 10,000 objects | In-memory LRU | LRU on pressure | < 10μs |
| L3 | Cold | Full object set | `KnowledgeStore` | Never evicted | < 5ms |

### 7.2 Promotion and Demotion

```
Access flow:
  L1 HIT  → return immediately
  L1 MISS → check L2
    L2 HIT  → promote to L1 (evict oldest L1 entry to L2)
    L2 MISS → load from L3
      L3 HIT  → promote to L2 (and L1 if hot)
      L3 MISS → lazy load from disk (Section 6)
```

### 7.3 Invalidation Rules

| Trigger Event | Invalidation Action |
|---------------|---------------------|
| `knowledge.store.written` (id=X) | Evict X from L1 and L2; L3 (store) is authoritative |
| `knowledge.snapshot.restored` | Flush all tiers (L1 and L2 cleared; L3 reloaded from snapshot) |
| `knowledge.runtime.pausing` | Write all dirty L2 entries to L3 before pausing |
| `knowledge.index.major_change` | Flush L1; L2 entries older than 60s are evicted |

### 7.4 Cache Metrics

| Metric | Description |
|--------|-------------|
| `knowledge.cache.l1.hit_rate` | L1 hit / total access ratio |
| `knowledge.cache.l2.hit_rate` | L2 hit / (L1 miss) ratio |
| `knowledge.cache.l2.eviction_rate` | L2 evictions per second |
| `knowledge.cache.l3.load_ms` | Average L3 load latency |

---

## 8. Hot Reload

Hot reload allows a specific domain's knowledge objects to be refreshed without restarting the runtime. It is designed to keep unaffected domains fully operational throughout the reload cycle.

### 8.1 Hot Reload Sequence

```
TRIGGER: knowledge.runtime.hot_reload_requested {domain: D}

1. PAUSE
   - Set domain D status to RELOADING in KnowledgeRegistry
   - Stop accepting new queries scoped to domain D (return DOMAIN_RELOADING)
   - Allow existing queries on domain D up to 5s to drain

2. FLUSH
   - Flush all dirty L2 cache entries for domain D to L3
   - Remove domain D entries from L1 and L2

3. RE-SCAN
   - Scan all files in domain D's file tree
   - Parse frontmatter for all files
   - Build updated object list for domain D

4. REBUILD INDEXES
   - Remove all domain D entries from KnowledgeRegistry, KnowledgeCatalog
   - Two-pass resolution for domain D objects (pass 1 then pass 2)
   - Rebuild TraceabilityIndex edges touching domain D nodes

5. REBUILD GRAPH
   - Remove all nodes where domain = D from KnowledgeGraphRuntime
   - Remove all edges incident to removed nodes
   - Add new nodes for domain D
   - Re-resolve edges (both outbound from D and inbound to D from other domains)
   - Re-run health score propagation for affected subgraph

6. RE-RUN REASONING
   - Re-apply inference rules for domain D
   - Update IMPACTS edges for domain D objects

7. RESUME
   - Set domain D status to ACTIVE
   - Resume accepting queries for domain D
   - Emit: knowledge.runtime.hot_reload.complete {domain: D, duration_ms: n}
```

### 8.2 Domain Isolation Guarantee

Hot reload of domain A **does not** affect the following for domain B:
- In-flight query execution
- Cache state for domain B objects
- Graph nodes and edges that do not touch domain A

The only cross-domain effect is re-resolution of edges that cross the A–B boundary (step 5), which may briefly lock those specific edges. This lock is held for < 50ms in practice.

---

## 9. Checkpointing

The runtime creates periodic checkpoints to bound recovery time after a crash. Checkpoints capture the minimum state needed to resume without a full reindex.

### 9.1 Schedule

Checkpoints are scheduled via `PlatformScheduler` at `BACKGROUND` priority, every **5 minutes**. A checkpoint can also be triggered manually or by `PlatformLifecycle` events (e.g., pre-shutdown).

### 9.2 Checkpoint Contents

```yaml
# checkpoints/CHECKPOINT-{timestamp}.yaml
checkpoint_id: CHK-{timestamp}
created_at: "2026-06-29T14:00:00Z"
runtime_version: "1.0.0"
wal_position: 8472          # Last committed WAL sequence number
graph_state_hash: "sha256:..." # Hash of node+edge set; used for staleness detection
index_freshness:
  registry_index:       "2026-06-29T13:58:00Z"
  traceability_index:   "2026-06-29T13:58:02Z"
  capability_index:     "2026-06-29T13:58:01Z"
object_count: 10247
edge_count: 38921
dirty_entries: []           # WAL entries not yet fully persisted
```

### 9.3 Checkpoint Files

```
checkpoints/
  CHECKPOINT-20260629T140000Z.yaml
  CHECKPOINT-20260629T140500Z.yaml
  CHECKPOINT-20260629T141000Z.yaml
  LATEST                          ← pointer to most recent valid checkpoint
```

The `LATEST` file contains only the filename of the last successfully written and validated checkpoint. It is updated atomically (write-rename) after each successful checkpoint write.

### 9.4 Checkpoint Retention

The runtime retains the last **12 checkpoints** (60 minutes of history). Older checkpoints are deleted during the write of each new checkpoint.

---

## 10. Recovery

Recovery is triggered when the runtime detects inconsistency between the WAL, the persisted indexes, and the last checkpoint.

### 10.1 Inconsistency Detection

During Phase 2 boot:

```
DETECT inconsistency IF:
  (a) WAL has entries with sequence_number > checkpoint.wal_position, OR
  (b) index freshness timestamps are older than checkpoint.created_at, OR
  (c) graph_state_hash does not match current graph hash (computed during Phase 4)
```

### 10.2 Recovery Protocol

```
RECOVERY SEQUENCE:

1. LOAD last-good checkpoint (LATEST pointer)
2. IDENTIFY WAL entries with sequence > checkpoint.wal_position
3. FOR EACH uncommitted WAL entry (in sequence order):
     APPLY entry to KnowledgeStore
     RE-INDEX affected objects (incremental update, Section 5)
4. REBUILD stale indexes (identified by freshness timestamp comparison)
5. RUN full integrity check (Section 3.3)
6. IF all steps succeed:
     EMIT knowledge.runtime.recovery.completed {replayed_entries: n, rebuilt_indexes: k}
     CONTINUE boot normally from Phase 3 completion point
7. IF any step fails:
     TRANSITION to DEGRADED
     LOAD last-good snapshot (read-only)
     EMIT knowledge.runtime.recovery.failed {reason, last_good_snapshot}
```

### 10.3 Degraded Mode

In `DEGRADED` state:
- All read queries are served from the last-good snapshot
- All write operations return `RUNTIME_DEGRADED` error
- `PlatformDiagnostics` emits `knowledge.runtime.degraded` every 30s
- Recovery may be retried by restarting the runtime

---

## 11. Shutdown Sequence

Graceful shutdown preserves all in-progress work and leaves the runtime in a state from which the next boot is fast.

```
SHUTDOWN TRIGGER: PlatformLifecycle.STOPPING event

Step 1: Lifecycle transition
  PlatformLifecycle → STOPPING
  Emit: knowledge.runtime.shutdown.starting

Step 2: Query gate — close
  KnowledgeQueryEngine.set_gate(SHUTTING_DOWN)
  All new queries after this point receive SHUTTING_DOWN response immediately

Step 3: Drain in-flight queries
  Wait for all active queries to complete
  Timeout: 30 seconds
  If timeout: log WARNING for each uncompleted query; proceed anyway

Step 4: Shutdown snapshot
  Create a final checkpoint (treated as pre-shutdown checkpoint)
  Tag: SHUTDOWN_CHECKPOINT (retained beyond normal 12-checkpoint retention)

Step 5: Persist storage
  Flush WAL: write all committed-but-not-persisted entries to KnowledgeStore
  Flush indexes: serialize all seven indexes to disk
  Flush L2 dirty entries to L3

Step 6: Deregister
  Unregister from PlatformRegistry
  Stop emitting heartbeat to PlatformDiagnostics

Step 7: Emit completion
  Emit: knowledge.runtime.shutdown.complete (persistent=True)

Step 8: Lifecycle transition
  PlatformLifecycle → STOPPED
```

### 11.1 Shutdown Timeout Behavior

| Wait | Max Duration | On Timeout |
|------|-------------|-----------|
| Query drain | 30s | Log uncompleted queries; proceed |
| WAL flush | 10s | Force-close WAL; mark as dirty in LATEST checkpoint |
| Index flush | 10s | Mark stale indexes; next boot takes slow path |

Forced shutdown (e.g., SIGKILL) bypasses graceful drain. Recovery handles the resulting dirty state on next boot.

---

## 12. Event Reference

All events emitted by the execution model:

| Event | Persistent | Phase | Description |
|-------|-----------|-------|-------------|
| `knowledge.runtime.boot.phase1.complete` | No | Boot | Foundation layer ready |
| `knowledge.runtime.boot.phase2.complete` | No | Boot | Storage layer ready |
| `knowledge.runtime.boot.phase3.complete` | No | Boot | Index layer ready |
| `knowledge.runtime.boot.phase4.complete` | No | Boot | Intelligence layer ready |
| `knowledge.runtime.boot.complete` | **Yes** | Boot | Runtime fully operational |
| `knowledge.runtime.heartbeat` | No | Running | 5-second liveness signal |
| `knowledge.index.updated` | No | Running | Incremental index update complete |
| `knowledge.index.major_change` | No | Running | > 10% objects changed |
| `knowledge.graph.loaded` | No | Boot | Graph build complete |
| `knowledge.graph.cycle_detected` | No | Boot/Rebuild | Circular dependency found |
| `knowledge.runtime.hot_reload.complete` | No | Running | Domain hot reload done |
| `knowledge.runtime.recovery.completed` | No | Boot | WAL replay succeeded |
| `knowledge.runtime.recovery.failed` | No | Boot | Recovery failed; degraded mode |
| `knowledge.runtime.degraded` | No | Degraded | 30s heartbeat in degraded state |
| `knowledge.runtime.shutdown.starting` | No | Shutdown | Shutdown initiated |
| `knowledge.runtime.shutdown.complete` | **Yes** | Shutdown | Runtime halted cleanly |

---

## 13. Performance Budget

| Operation | Target | Measured at |
|-----------|--------|------------|
| Full boot (10,000 objects, fresh indexes) | < 5s | Phase 5 completion |
| Full boot (10,000 objects, rebuild required) | < 8s | Phase 5 completion |
| Incremental index update (1 object) | < 200ms | `knowledge.index.updated` |
| Incremental index update (100 objects, batched) | < 1s | `knowledge.index.updated` |
| Hot reload (single domain, ~500 objects) | < 3s | `knowledge.hot_reload.complete` |
| Checkpoint write | < 500ms | Checkpoint file written |
| Graceful shutdown | < 60s | `knowledge.runtime.shutdown.complete` |

---

## References

- [KA-SPEC-020](../knowledge-core/20-KNOWLEDGE-RUNTIME-ARCHITECTURE.md) — Parent architecture specification
- [KR-ARCH-006](06-GRAPH-RUNTIME.md) — Graph runtime detail (Phase 4 boot subject)
- [KR-ARCH-007](07-QUERY-MODEL.md) — Query model (Phase 4 boot subject)
- [KA-KIP-002](../knowledge-intelligence/01-KNOWLEDGE-REGISTRY.md) — KnowledgeRegistry, KnowledgeRepository
- [KA-KIP-003](../knowledge-intelligence/02-KNOWLEDGE-GRAPH-RUNTIME.md) — KnowledgeGraphRuntime
- [KA-KIP-004](../knowledge-intelligence/03-KNOWLEDGE-QUERY-ENGINE.md) — KnowledgeQueryEngine
