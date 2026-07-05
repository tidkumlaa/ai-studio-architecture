---
knowledge_id: KR-ARCH-002
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
  - id: KR-ARCH-001
    reason: "Kernel integration is the operational backbone of the Knowledge Runtime"
  - id: KA-SPEC-001
    reason: "KnowledgeObject is the entity registered with the kernel"
  - id: KA-KIP-002
    reason: "KnowledgeRegistry is registered as a PlatformService"
implements:
  - KA-VIS-001
references:
  - path: platform-kernel/02-OBJECT-MODEL.md
    reason: "PlatformObject, PlatformEntity, PlatformService hierarchy"
  - path: platform-kernel/04-EVENT-SYSTEM.md
    reason: "PlatformEventBus event type and priority model"
  - path: platform-kernel/05-LIFECYCLE.md
    reason: "12-state PlatformLifecycle FSM"
  - path: platform-kernel/06-SERVICES.md
    reason: "PlatformRegistry and dependency injection model"
  - path: platform-kernel/07-SCHEDULING.md
    reason: "PlatformScheduler task priorities and periodic scheduling"
  - path: platform-kernel/09-DIAGNOSTICS.md
    reason: "PlatformDiagnostics metrics, health checks, and trace spans"
---

# Knowledge Runtime — Kernel Integration

## Platform Kernel Integration Specification for the Knowledge Runtime

---

## 1. Integration Overview

The Knowledge Runtime integrates with six Platform Kernel subsystems. Each integration is deliberate, bounded, and fully specified in this document.

```
╔══════════════════════════════════════════════════════════════════╗
║              Knowledge Runtime Kernel Integration Map            ║
╠══════════════════════╦═══════════════════════════════════════════╣
║  Kernel Subsystem    ║  Knowledge Runtime Integration            ║
╠══════════════════════╬═══════════════════════════════════════════╣
║  PlatformObject      ║  Base class for all KnowledgeObjects,     ║
║                      ║  KnowledgeServices, KnowledgeComponents   ║
╠══════════════════════╬═══════════════════════════════════════════╣
║  PlatformEventBus    ║  All state changes emit knowledge.*       ║
║                      ║  events; consumers subscribe, never poll  ║
╠══════════════════════╬═══════════════════════════════════════════╣
║  PlatformLifecycle   ║  KnowledgeRuntime tracks all 12 states;   ║
║                      ║  each state has defined knowledge actions  ║
╠══════════════════════╬═══════════════════════════════════════════╣
║  PlatformRegistry    ║  All 31 modules self-register at boot;    ║
║                      ║  consumers discover via registry lookup   ║
╠══════════════════════╬═══════════════════════════════════════════╣
║  PlatformScheduler   ║  5 task classes; 5 priority tiers;        ║
║                      ║  periodic indexing, checkpointing         ║
╠══════════════════════╬═══════════════════════════════════════════╣
║  PlatformDiagnostics ║  47 metrics, 6 health checks, all         ║
║                      ║  cross-service calls traced               ║
╚══════════════════════╩═══════════════════════════════════════════╝
```

All integration is injected at construction time. No Knowledge Runtime module accesses any kernel service via a global accessor or service locator.

---

## 2. PlatformObject Integration

### 2.1 Inheritance Map

Every entity in the Knowledge Runtime derives from a Platform Kernel base class. The full inheritance map:

```
PlatformObject (ABC)
│
├── PlatformEntity
│   └── KnowledgeObject (ABC)               [KR-ARCH-003 § 2]
│       ├── KnowledgeDocument               Specs, ADRs, architecture docs
│       ├── KnowledgeArtifact               Source files, binaries, outputs
│       ├── KnowledgeRelationship           Typed edge in the knowledge graph
│       ├── KnowledgeCapability             Declared capability
│       ├── KnowledgeDomain                 Domain grouping entity
│       └── KnowledgeSnapshot               Point-in-time capture of a subtree
│
├── PlatformService (ABC)
│   └── KnowledgeService (ABC)              Base for all runtime services
│       ├── KnowledgeRuntime                [KR-MOD-031] Orchestrator
│       ├── KnowledgeStore                  [KR-MOD-021] Persistence
│       ├── KnowledgeRegistry               [KR-MOD-019] Primary catalog
│       ├── KnowledgeIndexer                [KR-MOD-023] Background indexing
│       ├── KnowledgeQueryEngine            [KR-MOD-024] Query execution
│       ├── KnowledgeReasoningEngine        [KR-MOD-025] Inference
│       ├── KnowledgeGraphRuntime           [KR-MOD-020] Live graph
│       ├── KnowledgeHealthRuntime          [KR-MOD-027] Health scoring
│       ├── ImpactAnalysisEngine            [KR-MOD-026] Impact computation
│       ├── KnowledgeEvidenceRuntime        [KR-MOD-028] Evidence graph
│       ├── KnowledgeCompilerRuntime        [KR-MOD-029] Compilation pipeline
│       └── AIContextCompiler               [KR-MOD-030] AI context assembly
│
├── PlatformComponent (ABC)
│   └── KnowledgeComponent (ABC)            Stateful, lifecycle-managed
│       ├── KQLEngine                       [KR-MOD-007] KQL parser/executor
│       ├── KnowledgeGraphEngine            [KR-MOD-008] Property graph
│       ├── KnowledgeIndexEngine            [KR-MOD-010] Index structures
│       ├── KnowledgeLifecycleManager       [KR-MOD-014] Lifecycle FSM
│       ├── KnowledgeCompiler               [KR-MOD-009] Multi-target compiler
│       ├── KnowledgeCoverageAnalyzer       [KR-MOD-013] Coverage engine
│       ├── KnowledgeTraceabilityEngine     [KR-MOD-011] Trace chains
│       └── KnowledgeEvidenceEngine         [KR-MOD-012] Evidence records
│
└── PlatformValueObject (ABC)
    └── KnowledgeValueObject (ABC)          Immutable, identity-less
        ├── KnowledgeIdentity               ka:// URI + KA-XXX-NNN ID
        ├── KnowledgeMetadata               Status, owner, dates, phase
        ├── KnowledgeEvidence               Proof record with confidence
        ├── KnowledgeCoverage               Coverage scores per dimension
        └── KnowledgeVersion                Semantic version + history chain
```

### 2.2 KnowledgeObject Extends PlatformEntity

`KnowledgeObject` adds the following fields to `PlatformEntity`:

| Field | Type | Source | Description |
|-------|------|--------|-------------|
| `knowledge_id` | `KnowledgeIdentity` | KA-SPEC-003 | Human-readable canonical ID (KA-XXX-NNN) |
| `knowledge_version` | `KnowledgeVersion` | KA-SPEC-001 | Semantic version of the knowledge content |
| `knowledge_status` | `KnowledgeLifecycleState` | KA-SPEC-014 | Current state in the 7-state knowledge FSM |
| `knowledge_type` | `KnowledgeType` | KA-SPEC-002 | From the Knowledge Type System |
| `knowledge_domain` | `DomainID` | KR-ARCH-001 § 3 | One of the ten canonical domain IDs |
| `knowledge_capability` | `CapabilityID` | KA-SPEC-001 | Capability this object belongs to |
| `embedding_vector` | `list[float] \| None` | KR-ARCH-003 § 8 | Forward-compatibility field; runtime stores, never computes |
| `health_score` | `float` | KA-KIP-007 | Last computed health score [0.0, 100.0] |
| `evidence_count` | `int` | KA-SPEC-012 | Number of attached evidence records |
| `coverage_score` | `float` | KA-SPEC-013 | Aggregate coverage score [0.0, 1.0] |

The `object_id` field inherited from `PlatformObject` is a UUID v7. The `knowledge_id` is the human-readable identifier (`KA-ARCH-001`). Both are permanent and immutable after object creation.

### 2.3 KnowledgeService Extends PlatformService

`KnowledgeService` adds:

| Field / Method | Description |
|---------------|-------------|
| `service_domain` | Declares which knowledge domain this service primarily serves |
| `service_capabilities` | List of `PlatformCapability` tokens this service provides |
| `health_check_interval_ms` | Override default 30,000 ms health check interval |
| `on_knowledge_event(event)` | Hook for receiving `knowledge.*` events from the EventBus |

All `KnowledgeService` implementations are stateless between requests. State lives in `KnowledgeStore`, `KnowledgeCache`, or `KnowledgeRegistry`. This enables safe concurrent execution across the thread pool.

### 2.4 MetadataStore Integration

Every `KnowledgeObject` leverages `PlatformObject.set_meta()` / `get_meta()` to store knowledge-specific metadata not in the primary field set:

| Metadata Key | Type | Description |
|-------------|------|-------------|
| `knowledge.evidence_ids` | `list[str]` | IDs of attached evidence records |
| `knowledge.relationship_ids` | `list[str]` | IDs of declared outbound relationships |
| `knowledge.reasoning_state` | `dict` | Last computed inference state snapshot |
| `knowledge.coverage_detail` | `dict` | Per-dimension coverage breakdown |
| `knowledge.reviewer_ids` | `list[str]` | Reviewers who approved this object |
| `knowledge.superseded_by` | `str \| None` | ID of the superseding object if deprecated |
| `knowledge.deprecated_date` | `str \| None` | ISO date of deprecation |
| `knowledge.archived_date` | `str \| None` | ISO date of archival |
| `knowledge.restricted` | `bool` | Whether this object requires capability check to read |

All metadata values are JSON-serializable, consistent with the `PlatformObject` invariant.

---

## 3. PlatformEventBus Integration

### 3.1 Namespace and Registration

All Knowledge Runtime events are published under the `knowledge.*` namespace. The full event type registry is declared at `KnowledgeRuntime` startup via `PlatformEventBus.register_event_types()`.

Unknown event types are accepted by the kernel bus but logged as warnings. All knowledge event types are registered to suppress those warnings and enable schema validation.

### 3.2 Priority Assignment

| Event Class | Priority | Delivery Mode | Reason |
|-------------|----------|--------------|--------|
| `knowledge.corruption.detected` | CRITICAL | Synchronous | Must halt immediately |
| `knowledge.object.lifecycle.*` | HIGH | Async, dedicated thread | Lifecycle changes drive consistency |
| `knowledge.snapshot.created` | HIGH | Async, dedicated thread | Checkpoint integrity notification |
| `knowledge.object.created` | NORMAL | Async, shared pool | Standard write notification |
| `knowledge.object.updated` | NORMAL | Async, shared pool | Standard write notification |
| `knowledge.object.deprecated` | NORMAL | Async, shared pool | Standard write notification |
| `knowledge.graph.*` | NORMAL | Async, shared pool | Graph topology updates |
| `knowledge.health.*` | NORMAL | Async, shared pool | Health score changes |
| `knowledge.query.executed` | LOW | Async, background queue | Diagnostic only |
| `knowledge.index.updated` | LOW | Async, background queue | Background operation |
| `knowledge.cache.*` | BACKGROUND | Async, idle cycles | Telemetry only |

### 3.3 Persistence Model

| Persistent (written to durable log before dispatch) | Non-Persistent (in-memory only) |
|----------------------------------------------------|--------------------------------|
| `knowledge.object.created` | `knowledge.query.executed` |
| `knowledge.object.approved` | `knowledge.index.updated` |
| `knowledge.object.deprecated` | `knowledge.cache.hit` |
| `knowledge.object.archived` | `knowledge.cache.miss` |
| `knowledge.snapshot.created` | `knowledge.graph.traversal.completed` |
| `knowledge.corruption.detected` | `knowledge.reasoning.rule.applied` |
| `knowledge.recovery.completed` | `knowledge.health.score.computed` |

### 3.4 Complete Event Catalog

#### Object Events

```
knowledge.object.created
    Payload: knowledge_id, object_type, domain, capability, version
    Persistent: yes
    Priority: NORMAL

knowledge.object.updated
    Payload: knowledge_id, old_version, new_version, changed_fields[]
    Persistent: yes
    Priority: NORMAL

knowledge.object.lifecycle.draft
    Payload: knowledge_id, previous_state, triggered_by
    Persistent: yes
    Priority: HIGH

knowledge.object.lifecycle.review
    Payload: knowledge_id, previous_state, triggered_by, reviewer_ids[]
    Persistent: yes
    Priority: HIGH

knowledge.object.approved
    Payload: knowledge_id, version, approved_by, review_date
    Persistent: yes
    Priority: HIGH

knowledge.object.deprecated
    Payload: knowledge_id, superseded_by?, deprecated_date
    Persistent: yes
    Priority: NORMAL

knowledge.object.archived
    Payload: knowledge_id, archived_date
    Persistent: yes
    Priority: NORMAL

knowledge.object.superseded
    Payload: old_knowledge_id, new_knowledge_id, version
    Persistent: yes
    Priority: HIGH
```

#### Graph Events

```
knowledge.graph.edge.added
    Payload: source_id, target_id, relationship_type, confidence
    Persistent: no
    Priority: NORMAL

knowledge.graph.edge.removed
    Payload: source_id, target_id, relationship_type
    Persistent: no
    Priority: NORMAL

knowledge.graph.cycle.detected
    Payload: cycle_ids[], relationship_type
    Persistent: yes
    Priority: HIGH

knowledge.graph.scc.computed
    Payload: scc_count, largest_scc_size, duration_ms
    Persistent: no
    Priority: LOW

knowledge.graph.traversal.completed
    Payload: root_id, algorithm, nodes_visited, duration_ms
    Persistent: no
    Priority: LOW
```

#### Index Events

```
knowledge.index.updated
    Payload: domain?, capability?, objects_indexed, duration_ms
    Persistent: no
    Priority: LOW

knowledge.index.rebuild.started
    Payload: scope, estimated_objects
    Persistent: no
    Priority: NORMAL

knowledge.index.rebuild.completed
    Payload: scope, objects_indexed, duration_ms
    Persistent: yes
    Priority: NORMAL

knowledge.index.rebuild.failed
    Payload: scope, error_code, message
    Persistent: yes
    Priority: HIGH
```

#### Query Events

```
knowledge.query.executed
    Payload: correlation_id, kql_hash, result_count, duration_ms, cache_hit
    Persistent: no
    Priority: LOW

knowledge.query.slow
    Payload: correlation_id, kql_hash, duration_ms, threshold_ms
    Persistent: yes
    Priority: HIGH

knowledge.query.failed
    Payload: correlation_id, kql_hash, error_code, message
    Persistent: yes
    Priority: HIGH
```

#### Health and Integrity Events

```
knowledge.health.score.computed
    Payload: knowledge_id, score, previous_score, dimension_scores{}
    Persistent: no
    Priority: NORMAL

knowledge.health.degraded
    Payload: knowledge_id, score, threshold
    Persistent: yes
    Priority: HIGH

knowledge.coverage.computed
    Payload: knowledge_id, coverage_score, gap_count
    Persistent: no
    Priority: LOW

knowledge.corruption.detected
    Payload: object_id?, index_name?, description, severity
    Persistent: yes
    Priority: CRITICAL
```

#### Checkpoint and Recovery Events

```
knowledge.snapshot.started
    Payload: snapshot_id, scope, object_count
    Persistent: no
    Priority: NORMAL

knowledge.snapshot.created
    Payload: snapshot_id, scope, object_count, size_bytes, duration_ms
    Persistent: yes
    Priority: HIGH

knowledge.snapshot.failed
    Payload: snapshot_id, error_code, message
    Persistent: yes
    Priority: HIGH

knowledge.recovery.started
    Payload: snapshot_id, reason
    Persistent: yes
    Priority: CRITICAL

knowledge.recovery.completed
    Payload: snapshot_id, objects_recovered, duration_ms
    Persistent: yes
    Priority: HIGH

knowledge.recovery.failed
    Payload: snapshot_id, error_code, message
    Persistent: yes
    Priority: CRITICAL
```

### 3.5 Correlation Model

Every KQL query execution generates a `correlation_id` (UUID v7). All events emitted during that query share this `correlation_id`. This enables exact reconstruction of the event chain for any query.

```
Query submitted by caller
  └─ correlation_id: corr-7f3a-...
      ├─ knowledge.query.executed           { correlation_id: corr-7f3a-... }
      ├─ knowledge.graph.traversal.completed { correlation_id: corr-7f3a-... }
      └─ knowledge.reasoning.rule.applied   { correlation_id: corr-7f3a-... }
```

For write operations, the correlation chain extends across the event cascade:

```
Write operation by caller
  └─ correlation_id: corr-9b11-...
      ├─ knowledge.object.created           { correlation_id: corr-9b11-... }
      ├─ knowledge.index.updated            { correlation_id: corr-9b11-... }
      ├─ knowledge.graph.edge.added         { correlation_id: corr-9b11-... }
      └─ knowledge.health.score.computed    { correlation_id: corr-9b11-... }
```

---

## 4. PlatformLifecycle Integration

### 4.1 KnowledgeRuntime Lifecycle — All 12 States

`KnowledgeRuntime` (KR-MOD-031) transitions through all 12 states defined by `PlatformLifecycle`. Each state has specific knowledge-level actions.

| State | Knowledge Runtime Action |
|-------|-------------------------|
| UNINITIALIZED | No action. `KernelContainer` has not yet called `initialize()`. |
| INITIALIZED | Configuration loaded. Log level, cache size, checkpoint interval, domain list set. No I/O yet. |
| STARTING | (1) Validate file store integrity. (2) Load last checkpoint. (3) Rebuild dirty index partitions. (4) Warm cache with pinned objects. (5) Register all 31 modules with PlatformRegistry. (6) Start background indexer task. |
| RUNNING | Accepting KQL queries. Publishing knowledge events. Background indexing active. Periodic checkpointing active. |
| PAUSING | Stop accepting new KQL queries. Drain in-flight queries (up to `drain_timeout_ms`). Suspend background indexer. |
| PAUSED | No queries accepted. Graph and index are frozen. State is consistent. Can be resumed or stopped. |
| STOPPING | (1) Cancel background tasks. (2) Flush dirty cache entries to store. (3) Write shutdown checkpoint. (4) Unregister all 31 modules from PlatformRegistry. |
| STOPPED | All resources released. Transaction log closed. Ready for restart without process restart. |
| FAILED | Published `knowledge.corruption.detected` or raised unrecoverable exception. No queries served. Recovery is the only path forward. |
| RECOVERING | (1) Validate last checkpoint. (2) Replay transaction log from checkpoint. (3) Rebuild affected index partitions. (4) Verify graph integrity (SCC check). (5) Transition to RUNNING if all checks pass; to FAILED if any fail. |
| DISPOSING | Release all object references. Clear registry, graph, indexes, cache. Signal GC. |
| DISPOSED | Terminal. Object is inert. |

### 4.2 Boot Sequence — Detailed

The boot sequence during STARTING spans 8 phases. Each phase is atomic; failure in any phase transitions to FAILED.

```
Phase 1: Store Validation
  ├─ Check KNOWLEDGE-REGISTRY.yaml exists and is parseable
  ├─ Validate transaction log checksum
  └─ Detect any incomplete write records → mark dirty

Phase 2: Checkpoint Load
  ├─ Locate most recent valid snapshot file
  ├─ Deserialize snapshot into KnowledgeRegistry
  └─ Populate KnowledgeGraphEngine adjacency lists from snapshot

Phase 3: Log Replay
  ├─ Read transaction log from checkpoint LSN
  ├─ Apply each log entry to registry and graph
  └─ Emit knowledge.recovery.completed if replay was needed

Phase 4: Index Rebuild
  ├─ Identify dirty index partitions (log replay may expand this)
  ├─ Schedule incremental rebuild as NORMAL priority task
  └─ KnowledgeIndexer begins rebuild asynchronously

Phase 5: Cache Warm
  ├─ Load domain-specific pin sets from configuration
  ├─ Pre-load pinned objects into KnowledgeCache
  └─ Emit knowledge.cache.warmed

Phase 6: Module Registration
  ├─ Register KnowledgeRuntime (self) in PlatformRegistry
  ├─ Register all 30 sub-modules in PlatformRegistry
  └─ Register health checks for each module

Phase 7: Scheduler Setup
  ├─ Schedule periodic checkpoint task (default: every 60 s, HIGH)
  ├─ Schedule periodic health recompute (default: every 30 s, NORMAL)
  └─ Schedule incremental index update (default: every 10 s, LOW)

Phase 8: Ready Signal
  ├─ Emit kernel.lifecycle.started for KnowledgeRuntime
  └─ Transition to RUNNING
```

### 4.3 Shutdown Sequence — Detailed

The shutdown sequence during STOPPING is the mirror of boot, executed in reverse:

```
Phase 1: Drain
  ├─ Stop accepting new KQL queries
  ├─ Wait for in-flight queries to complete (max drain_timeout_ms = 5,000)
  └─ Force-cancel remaining queries; log each cancelled query_id

Phase 2: Task Cancellation
  ├─ Cancel periodic checkpoint task
  ├─ Cancel periodic health recompute task
  └─ Cancel incremental indexer task; wait for completion or timeout

Phase 3: Cache Flush
  ├─ Write all dirty cache entries to KnowledgeStore
  └─ Close cache; release memory

Phase 4: Shutdown Checkpoint
  ├─ Create final snapshot covering all domains
  ├─ Emit knowledge.snapshot.created
  └─ Close transaction log with final LSN record

Phase 5: Deregistration
  ├─ Unregister all 30 sub-modules from PlatformRegistry
  └─ Unregister KnowledgeRuntime (self) from PlatformRegistry

Phase 6: Complete
  └─ Transition to STOPPED
```

---

## 5. PlatformRegistry Integration

### 5.1 Registration Contract

At the completion of Phase 6 of the boot sequence, all 31 Knowledge Runtime modules are present in `PlatformRegistry`. Each registration entry carries:

| Registration Field | Value |
|-------------------|-------|
| `object_id` | UUID v7 of the service instance |
| `object_type` | Fully qualified type name (e.g., `knowledge.runtime.KnowledgeRegistry`) |
| `tags` | See § 5.2 |
| `capabilities` | See § 5.3 |
| `health_check` | Registered health check function (see § 7.2) |
| `ttl` | None — knowledge services are permanent until STOPPING |

### 5.2 Registration Tags

Each module registers with a base set of tags plus module-specific tags:

| Tag Category | Values |
|-------------|--------|
| Namespace tag | `"knowledge"` — all modules |
| Layer tag | `"runtime"` — KR-MOD-031; `"store"` — KR-MOD-021, 022, 023; `"graph"` — KR-MOD-008, 020; `"query"` — KR-MOD-007, 024; `"intelligence"` — KR-MOD-025, 026, 027, 028, 029, 030 |
| Domain tag | `"dom-arch"`, `"dom-source"`, etc. — modules that serve a single domain include its tag |
| Phase tag | `"phase-2.0d.3a"` — all modules |

### 5.3 Capabilities Provided

The following `PlatformCapability` tokens are published in the registry at boot. Other platform services check for these before calling Knowledge Runtime APIs.

| Capability Token | Provided By | Meaning |
|-----------------|-------------|---------|
| `knowledge.query` | KR-MOD-024 | Caller may execute KQL queries |
| `knowledge.reason` | KR-MOD-025 | Caller may invoke inference engine |
| `knowledge.write` | KR-MOD-021 | Caller may create or update knowledge objects |
| `knowledge.compile` | KR-MOD-029 | Caller may compile knowledge objects |
| `knowledge.context` | KR-MOD-030 | Caller may assemble AI context packages |
| `knowledge.impact` | KR-MOD-026 | Caller may request impact analysis |
| `knowledge.health` | KR-MOD-027 | Caller may read health scores and trends |
| `knowledge.admin` | KR-MOD-031 | Caller may perform administrative operations (hot reload, checkpoint) |

### 5.4 Consumer Discovery

Application-layer consumers discover Knowledge Runtime services via `PlatformRegistry`:

```
# Discover query service
query_engine = registry.find_by_tag("knowledge", "query")[0]

# Discover all intelligence services
intel_services = registry.find_by_tag("knowledge", "intelligence")

# Discover by capability
capable_services = registry.find_by_capability("knowledge.context")
```

No application-layer code holds a direct reference to a `KnowledgeService` instance. All access is via registry lookup. This enables transparent hot reload without client rebinding.

---

## 6. PlatformScheduler Integration

### 6.1 Task Class Definitions

Five task classes cover all background work in the Knowledge Runtime:

| Task Class | Priority Tier | Priority Value | Description |
|-----------|--------------|---------------|-------------|
| Recovery | CRITICAL | 10 | Invoked only during RECOVERING state; preempts all other work |
| Checkpoint | HIGH | 8 | Periodic snapshot creation; guaranteed slot every 60 s |
| Hot Reload | NORMAL | 6 | Module refresh on file system change; must complete within 1,000 ms |
| Health Recompute | NORMAL | 5 | Periodic health score propagation; every 30 s |
| Incremental Index | LOW | 3 | Background index update after write events; every 10 s |

### 6.2 Periodic Task Schedule

```
Task Name                   Interval     Priority  Timeout     Max Retries
─────────────────────────────────────────────────────────────────────────────
knowledge.checkpoint         60 s         HIGH      30,000 ms   0 (no retry)
knowledge.health.recompute   30 s         NORMAL    10,000 ms   1
knowledge.index.incremental  10 s         LOW        5,000 ms   2
knowledge.graph.integrity    300 s        LOW       30,000 ms   0
knowledge.evidence.propagate 120 s        LOW       15,000 ms   1
```

The checkpoint task has `max_retries = 0` because a failed checkpoint must emit `knowledge.snapshot.failed` and surface to operations immediately. Silent retry of a checkpoint failure is dangerous.

### 6.3 Hot Reload Task

Hot reload is triggered by `PlatformEventBus` event `kernel.config.reloaded` or `kernel.plugin.reloading`. The hot reload task:

1. Pauses the affected sub-module (transitions to PAUSED state)
2. Flushes in-flight work for that sub-module
3. Reloads the sub-module's configuration or code
4. Resumes the sub-module (transitions back to RUNNING)
5. Emits `knowledge.module.reloaded` with module ID and duration

Hot reload is scoped to a single module. `KnowledgeRuntime` (KR-MOD-031) itself cannot be hot-reloaded; it requires a graceful restart.

### 6.4 Recovery Task

Recovery is submitted as a CRITICAL task when `KnowledgeRuntime` enters the RECOVERING state. The scheduler guarantees inline execution (no queue delay) for CRITICAL tasks. The recovery task has a hard timeout of 300 seconds.

---

## 7. PlatformDiagnostics Integration

### 7.1 Metrics — Full Catalog

All metrics follow the naming convention `knowledge.{component}.{measurement}[.{suffix}]`.

#### Counters (monotonically increasing)

| Metric Name | Labels | Description |
|-------------|--------|-------------|
| `knowledge.objects.created.total` | `domain`, `type` | Total objects created |
| `knowledge.objects.updated.total` | `domain`, `type` | Total object updates |
| `knowledge.objects.deprecated.total` | `domain` | Total deprecations |
| `knowledge.objects.archived.total` | `domain` | Total archivements |
| `knowledge.queries.executed.total` | `result_type` | Total KQL queries executed |
| `knowledge.queries.failed.total` | `error_code` | Total failed queries |
| `knowledge.queries.slow.total` | — | Queries exceeding p99 threshold |
| `knowledge.cache.hits.total` | `domain` | Total cache hits |
| `knowledge.cache.misses.total` | `domain` | Total cache misses |
| `knowledge.cache.evictions.total` | `reason` | Total cache evictions |
| `knowledge.index.updates.total` | `partition` | Total incremental index updates |
| `knowledge.index.rebuilds.total` | `scope` | Total index rebuilds |
| `knowledge.graph.edges.added.total` | `relationship_type` | Total graph edges added |
| `knowledge.graph.edges.removed.total` | `relationship_type` | Total graph edges removed |
| `knowledge.graph.cycles.detected.total` | — | Total graph cycles detected |
| `knowledge.snapshots.created.total` | `scope` | Total checkpoints created |
| `knowledge.snapshots.failed.total` | — | Total failed checkpoints |
| `knowledge.recoveries.completed.total` | — | Total successful recoveries |
| `knowledge.recoveries.failed.total` | — | Total failed recoveries |
| `knowledge.events.published.total` | `event_type`, `priority` | Total events published |
| `knowledge.reasoning.rules.applied.total` | `rule_id` | Total inference rules applied |
| `knowledge.evidence.records.total` | `domain` | Total evidence records |

#### Gauges (current state)

| Metric Name | Labels | Description |
|-------------|--------|-------------|
| `knowledge.objects.active` | `domain` | Currently active (approved) object count |
| `knowledge.objects.total` | `domain`, `status` | Total object count by status |
| `knowledge.cache.size` | — | Objects currently in cache |
| `knowledge.cache.bytes` | — | Bytes currently in cache |
| `knowledge.graph.vertices` | — | Vertices in the live graph |
| `knowledge.graph.edges` | — | Edges in the live graph |
| `knowledge.index.size` | `partition` | Entries per index partition |
| `knowledge.scheduler.tasks.active` | `priority` | Active background tasks per tier |
| `knowledge.scheduler.tasks.queued` | `priority` | Queued background tasks per tier |
| `knowledge.health.score.mean` | `domain` | Mean health score across a domain |
| `knowledge.coverage.score.mean` | `domain` | Mean coverage score across a domain |
| `knowledge.snapshot.age_seconds` | — | Seconds since last successful checkpoint |

#### Histograms (distributions)

| Metric Name | Buckets (ms unless noted) | Description |
|-------------|--------------------------|-------------|
| `knowledge.query.duration_ms` | 1, 5, 10, 25, 50, 100, 250, 500, 1000 | End-to-end KQL query latency |
| `knowledge.query.result_count` | 0, 1, 5, 10, 25, 50, 100, 500, 1000 | Objects returned per query |
| `knowledge.object.load_duration_ms` | 1, 2, 5, 10, 25, 50, 100 | Object load from store |
| `knowledge.index.update_duration_ms` | 1, 5, 10, 25, 50, 100, 500 | Incremental index update |
| `knowledge.index.rebuild_duration_ms` | 100, 500, 1000, 2000, 5000, 10000 | Full index rebuild |
| `knowledge.snapshot.duration_ms` | 100, 500, 1000, 2000, 5000, 30000 | Checkpoint creation |
| `knowledge.recovery.duration_ms` | 500, 1000, 5000, 15000, 60000, 300000 | Full recovery |
| `knowledge.graph.traversal_duration_ms` | 1, 5, 10, 25, 50, 100, 500 | Graph traversal |
| `knowledge.reasoning.duration_ms` | 1, 5, 10, 25, 50, 100, 500 | Inference engine execution |
| `knowledge.health.compute_duration_ms` | 1, 5, 10, 25, 50, 100 | Health score computation per object |
| `knowledge.object.size_bytes` | 512, 1k, 4k, 16k, 64k, 256k | Knowledge object content size |

### 7.2 Health Checks

Six health checks are registered at boot. Each returns `HEALTHY`, `DEGRADED`, or `UNHEALTHY` with a human-readable message.

| Health Check Name | Component | HEALTHY | DEGRADED | UNHEALTHY |
|------------------|-----------|---------|----------|-----------|
| `knowledge.graph.integrity` | KR-MOD-020 | Graph is acyclic (no unintended cycles) | ≤ 3 weak cycles detected | > 3 cycles or structural corruption |
| `knowledge.index.freshness` | KR-MOD-023 | Index updated within 2× interval | Index updated within 5× interval | Index last updated > 5× interval ago |
| `knowledge.cache.hit_rate` | KR-MOD-022 | Hit rate ≥ 80% | Hit rate 50–80% | Hit rate < 50% |
| `knowledge.snapshot.age` | KR-MOD-021 | Last checkpoint < 2× interval | Last checkpoint < 5× interval | No checkpoint in > 5× interval |
| `knowledge.objects.approved_ratio` | KR-MOD-019 | ≥ 70% of objects are `approved` | 40–70% approved | < 40% approved |
| `knowledge.health.mean_score` | KR-MOD-027 | Mean health ≥ 75.0 | Mean health 50–75 | Mean health < 50 |

### 7.3 Distributed Trace Spans

Every cross-module operation is traced. Span names follow `knowledge.{source}.{operation}`.

| Span Name | Start | End | Key Attributes |
|-----------|-------|-----|---------------|
| `knowledge.query.execute` | KQL received | Result returned | `kql_hash`, `result_count`, `cache_hit` |
| `knowledge.query.plan` | Planning start | Plan ready | `plan_type`, `estimated_cost` |
| `knowledge.index.lookup` | Index scan start | Candidate set ready | `partition`, `candidates` |
| `knowledge.registry.get` | Registry lookup start | Object returned | `knowledge_id`, `hit` |
| `knowledge.graph.traverse` | Traversal start | Traversal end | `root_id`, `algorithm`, `depth` |
| `knowledge.reasoning.infer` | Inference start | Inference end | `rule_count`, `relationships_added` |
| `knowledge.store.read` | Store read start | Object returned | `knowledge_id`, `bytes` |
| `knowledge.store.write` | Store write start | Committed | `knowledge_id`, `bytes` |
| `knowledge.cache.get` | Cache lookup start | Hit/miss returned | `knowledge_id`, `hit` |
| `knowledge.snapshot.create` | Snapshot start | Snapshot complete | `snapshot_id`, `object_count` |
| `knowledge.health.compute` | Health compute start | Score stored | `knowledge_id`, `score` |
| `knowledge.compiler.compile` | Compile start | Output ready | `knowledge_id`, `target_format` |

All spans carry `correlation_id` and `trace_id` from the initiating request, enabling full distributed trace reconstruction.

---

## 8. Boot Order Within the Knowledge Runtime

The Knowledge Runtime's internal modules boot in the following strict dependency order. A module at position N may only be started after all modules at positions < N are RUNNING.

```
[1]  KnowledgeTypeSystem        (no deps)
[2]  KnowledgeIdentityService   (KnowledgeTypeSystem)
[3]  KnowledgeURIResolver       (KnowledgeIdentityService)
[4]  KnowledgeSchemaValidator   (KnowledgeTypeSystem)
[5]  KnowledgeStore             (KnowledgeSchemaValidator, PlatformSecurity)
[6]  KnowledgeCache             (KnowledgeStore)
[7]  KnowledgeRegistry          (KnowledgeStore, KnowledgeCache)
[8]  KnowledgeIndexEngine       (KnowledgeRegistry)
[9]  KnowledgeGraphEngine       (KnowledgeRegistry)
[10] KnowledgeLifecycleManager  (KnowledgeRegistry, PlatformEventBus)
[11] KnowledgeIndexer           (KnowledgeIndexEngine, PlatformScheduler)
[12] KQLEngine                  (KnowledgeRegistry, KnowledgeIndexEngine)
[13] KnowledgeQueryEngine       (KQLEngine, KnowledgeGraphEngine, KnowledgeCache)
[14] KnowledgeTraceabilityEngine (KnowledgeGraphEngine)
[15] KnowledgeEvidenceEngine    (KnowledgeRegistry, KnowledgeTraceabilityEngine)
[16] KnowledgeCoverageAnalyzer  (KnowledgeEvidenceEngine)
[17] KnowledgeReasoningEngine   (KnowledgeGraphEngine, KnowledgeEvidenceEngine)
[18] ImpactAnalysisEngine       (KnowledgeGraphEngine, KnowledgeReasoningEngine)
[19] KnowledgeHealthRuntime     (KnowledgeRegistry, KnowledgeCoverageAnalyzer)
[20] KnowledgeCompiler          (KnowledgeRegistry, KnowledgeSchemaValidator)
[21] KnowledgeCompilerRuntime   (KnowledgeCompiler, PlatformScheduler)
[22] AIContextCompiler          (KnowledgeRegistry, KnowledgeQueryEngine)
[23] KnowledgeGraphRuntime      (KnowledgeGraphEngine, KnowledgeReasoningEngine)
[24] KnowledgeEvidenceRuntime   (KnowledgeEvidenceEngine, KnowledgeGraphRuntime)
[25] KnowledgeDSLParser         (KnowledgeSchemaValidator, KnowledgeTypeSystem)
[26] CSStandardsLibrary         (no deps)
[27] DataStructureCatalog       (CSStandardsLibrary)
[28] AlgorithmCatalog           (CSStandardsLibrary, DataStructureCatalog)
[29] PerformanceBudgetEnforcer  (PlatformDiagnostics, PlatformEventBus)
[30] KnowledgeObjectFactory     (all above)
[31] KnowledgeRuntime           (all above; self-registers last)
```

Modules at the same position number may start concurrently. The `KernelContainer` parallelizes within each layer to minimize startup time.
