---
knowledge_id: KR-ARCH-009
version: "1.0.0"
status: approved
owner: Chief Knowledge Architect
phase: 2.0D.3A
created: 2026-06-29
review_date: 2027-06-29
canonical: true
domain: DOM-KNOWLEDGE
capability: knowledge-runtime
type: architecture
depends_on:
  - id: KR-ARCH-008
    reason: "Reasoning engine emits events defined here"
  - id: KA-SPEC-014
    reason: "Lifecycle states govern which lifecycle events fire"
---

# Knowledge Runtime — Event Model

## PlatformEventBus Integration · Complete Event Catalog · Subscription Patterns · Priority and Persistence Rules

---

## 1. Event Model Overview

All Knowledge Runtime events are published to the `PlatformEventBus` as `PlatformEvent` objects. The event model is the primary integration contract between the Knowledge Runtime and all external consumers (application layer, dashboards, CI pipelines, replication nodes).

### 1.1 Naming Convention

```
knowledge.<subsystem>.<action>

subsystem  : runtime | object | store | index | graph | query |
             reasoning | coverage | evidence | snapshot | version |
             cache | recommendation | validation | compiler |
             replication | session

action     : created | updated | deleted | started | completed |
             failed | detected | changed | rebuilt | generated |
             verified | expired | flushed | hit | warmed | evicted |
             synced | bumped | pinned | ready | boot.complete | ...
```

All multi-word actions use dot notation: `boot.phase1.complete`, `lifecycle.changed`.

### 1.2 Universal Envelope Fields

Every event, regardless of type, carries:

| Field | Type | Description |
|-------|------|-------------|
| `event_type` | str | Full dotted event type string |
| `event_id` | str | UUID v4, globally unique |
| `source_id` | str | `object_id` of the publishing module |
| `correlation_id` | str | Request or operation ID; ties events in one operation together |
| `timestamp` | datetime | UTC ISO-8601 timestamp at publish time |
| `priority` | Priority | BACKGROUND / LOW / NORMAL / HIGH / CRITICAL |
| `persistent` | bool | Whether the event is written to the persistent event log |
| `payload` | dict | Event-specific fields (see catalog below) |

### 1.3 Priority Semantics

| Priority | Use | Processing |
|----------|-----|-----------|
| `CRITICAL` | Errors that compromise runtime integrity | Synchronous; consumers block until handled |
| `HIGH` | Lifecycle-changing operations | Synchronous; published before return |
| `NORMAL` | Standard operations (writes, updates) | Asynchronous; queued |
| `LOW` | Query telemetry, index updates | Asynchronous; low-priority queue |
| `BACKGROUND` | Metrics, cache signals | Best-effort; dropped under load |

### 1.4 Persistence Rules

Events marked `persistent: true` are written to the **Knowledge Event Log** (a durable append-only log backed by the Platform Kernel's WAL) in addition to being published on the bus. Persistent events:
- Survive runtime restart.
- Are replayed to new subscribers on first connection (event sourcing support).
- Are used as the source of truth for audit trails and replication.

---

## 2. Complete Event Catalog

### 2.1 knowledge.runtime.*

Boot, shutdown, and recovery lifecycle events for the `KnowledgeRuntime` module itself.

| Event Type | Priority | Persistent | Publisher | Typical Consumers | Payload |
|-----------|----------|-----------|-----------|------------------|---------|
| `knowledge.runtime.boot.phase1.complete` | HIGH | no | KnowledgeRuntime | KnowledgeBootCoordinator | `{phase: 1, phase_name: "platform_bind", duration_ms}` |
| `knowledge.runtime.boot.phase2.complete` | HIGH | no | KnowledgeRuntime | KnowledgeBootCoordinator | `{phase: 2, phase_name: "store_open", duration_ms}` |
| `knowledge.runtime.boot.phase3.complete` | HIGH | no | KnowledgeRuntime | KnowledgeBootCoordinator | `{phase: 3, phase_name: "index_load", duration_ms}` |
| `knowledge.runtime.boot.phase4.complete` | HIGH | no | KnowledgeRuntime | KnowledgeBootCoordinator | `{phase: 4, phase_name: "graph_build", duration_ms}` |
| `knowledge.runtime.boot.complete` | HIGH | **yes** | KnowledgeRuntime | All consumers | `{kernel_id, boot_time_ms, object_count, index_count, graph_node_count, graph_edge_count}` |
| `knowledge.runtime.ready` | HIGH | no | KnowledgeRuntime | Application layer | `{ready_at}` |
| `knowledge.runtime.stopping` | HIGH | no | KnowledgeRuntime | All active sessions | `{reason, grace_period_ms}` |
| `knowledge.runtime.shutdown.complete` | HIGH | **yes** | KnowledgeRuntime | Platform Kernel | `{uptime_ms, events_published}` |
| `knowledge.runtime.recovery.started` | CRITICAL | **yes** | KnowledgeRuntime | Platform monitors | `{recovery_reason, from_snapshot_id}` |
| `knowledge.runtime.recovery.completed` | HIGH | **yes** | KnowledgeRuntime | All consumers | `{recovery_duration_ms, objects_restored}` |
| `knowledge.runtime.recovery.failed` | CRITICAL | **yes** | KnowledgeRuntime | Platform monitors | `{error_code, error_message, partial_state}` |

### 2.2 knowledge.object.*

Events for individual `KnowledgeObject` lifecycle transitions.

| Event Type | Priority | Persistent | Publisher | Typical Consumers | Payload |
|-----------|----------|-----------|-----------|------------------|---------|
| `knowledge.object.created` | NORMAL | **yes** | KnowledgeStore | KnowledgeIndexer, KnowledgeGraphRuntime | `{knowledge_id, object_type, domain, lifecycle_state, canonical}` |
| `knowledge.object.updated` | NORMAL | **yes** | KnowledgeStore | KnowledgeIndexer, KnowledgeGraphRuntime | `{knowledge_id, changed_fields: list[str], version}` |
| `knowledge.object.lifecycle.changed` | HIGH | **yes** | KnowledgeLifecycleManager | KnowledgeReasoningEngine, application layer | `{knowledge_id, from_state, to_state, changed_by, reason}` |
| `knowledge.object.approved` | HIGH | **yes** | KnowledgeLifecycleManager | KnowledgeCoverageEngine, KnowledgeReasoningEngine | `{knowledge_id, approved_by, approved_at}` |
| `knowledge.object.deprecated` | HIGH | **yes** | KnowledgeLifecycleManager | KnowledgeReasoningEngine (DEP-PROP-002) | `{knowledge_id, deprecated_by, deprecated_at, successor_id}` |
| `knowledge.object.archived` | NORMAL | **yes** | KnowledgeLifecycleManager | KnowledgeIndexer | `{knowledge_id, archived_at}` |
| `knowledge.object.deleted` | HIGH | **yes** | KnowledgeStore | KnowledgeGraphRuntime, KnowledgeIndexer | `{knowledge_id, deleted_by, was_canonical}` |

### 2.3 knowledge.store.*

Events from the `KnowledgeStore` module for persistence operations.

| Event Type | Priority | Persistent | Publisher | Typical Consumers | Payload |
|-----------|----------|-----------|-----------|------------------|---------|
| `knowledge.store.written` | NORMAL | no | KnowledgeStore | KnowledgeIndexer, KnowledgeGraphRuntime, KnowledgeReasoningEngine | `{knowledge_id, operation: WRITE|UPDATE, size_bytes}` |
| `knowledge.store.deleted` | NORMAL | no | KnowledgeStore | KnowledgeIndexer, KnowledgeGraphRuntime | `{knowledge_id}` |
| `knowledge.store.batch.committed` | NORMAL | no | KnowledgeStore | KnowledgeIndexer | `{batch_id, object_count, duration_ms}` |
| `knowledge.store.wal.flushed` | LOW | no | KnowledgeStore | Replication monitors | `{wal_lsn, flush_duration_ms}` |

### 2.4 knowledge.index.*

Events from `KnowledgeIndexer` reflecting index state.

| Event Type | Priority | Persistent | Publisher | Typical Consumers | Payload |
|-----------|----------|-----------|-----------|------------------|---------|
| `knowledge.index.rebuilt` | NORMAL | no | KnowledgeIndexer | KnowledgeGraphRuntime, KnowledgeQueryEngine | `{index_count, object_count, duration_ms}` |
| `knowledge.index.updated` | NORMAL | no | KnowledgeIndexer | KnowledgeQueryEngine (cache invalidation) | `{changed_index_keys: list[str], trigger_object_id}` |
| `knowledge.index.stale` | HIGH | no | KnowledgeIndexer | KnowledgeBootCoordinator | `{stale_reason, affected_index_count}` |

### 2.5 knowledge.graph.*

Events from `KnowledgeGraphRuntime` and graph sub-components.

| Event Type | Priority | Persistent | Publisher | Typical Consumers | Payload |
|-----------|----------|-----------|-----------|------------------|---------|
| `knowledge.graph.node.added` | NORMAL | no | KnowledgeGraphRuntime | KnowledgeReasoningEngine | `{node_id, node_type}` |
| `knowledge.graph.node.removed` | NORMAL | no | KnowledgeGraphRuntime | KnowledgeReasoningEngine | `{node_id}` |
| `knowledge.graph.edge.added` | NORMAL | no | KnowledgeGraphRuntime | KnowledgeReasoningEngine | `{from_id, to_id, edge_type, weight}` |
| `knowledge.graph.edge.removed` | NORMAL | no | KnowledgeGraphRuntime | KnowledgeReasoningEngine | `{from_id, to_id, edge_type}` |
| `knowledge.graph.rebuilt` | NORMAL | no | KnowledgeGraphRuntime | KnowledgeQueryEngine, KnowledgeReasoningEngine | `{node_count, edge_count, duration_ms}` |
| `knowledge.graph.cycle.detected` | HIGH | no | KnowledgeCycleDetector | KnowledgeReasoningEngine, application layer | `{cycle_nodes: list[str], cycle_length: int, cycle_type: DIRECT|TRANSITIVE}` |
| `knowledge.graph.corrupted` | CRITICAL | **yes** | KnowledgeGraphRuntime | KnowledgeRuntime (recovery trigger) | `{corruption_type, affected_nodes: list[str], detection_method}` |
| `knowledge.graph.projection.ready` | LOW | no | KnowledgeGraphProjector | KnowledgeQueryEngine | `{projection_name, node_count, edge_count}` |

### 2.6 knowledge.query.*

Events from `KnowledgeQueryEngine`.

| Event Type | Priority | Persistent | Publisher | Typical Consumers | Payload |
|-----------|----------|-----------|-----------|------------------|---------|
| `knowledge.query.executed` | LOW | no | KnowledgeQueryEngine | Telemetry, dashboards | `{query_id, kql_hash, duration_ms, row_count, from_cache, plan_type}` |
| `knowledge.query.cache.hit` | BACKGROUND | no | KnowledgeCacheManager | Telemetry | `{cache_key, tier: HOT|WARM, age_ms}` |
| `knowledge.query.failed` | HIGH | no | KnowledgeQueryEngine | Application layer | `{query_id, error_code, error_message, kql_fragment}` |
| `knowledge.query.cost.exceeded` | NORMAL | no | KnowledgeQueryEngine | Application layer, rate limiter | `{query_id, estimated_cost, cost_limit, kql_hash}` |
| `knowledge.query.slow` | NORMAL | no | KnowledgeQueryEngine | Performance monitors | `{query_id, duration_ms, slow_threshold_ms, plan_type}` |

### 2.7 knowledge.reasoning.*

Events from `KnowledgeReasoningEngine`.

| Event Type | Priority | Persistent | Publisher | Typical Consumers | Payload |
|-----------|----------|-----------|-----------|------------------|---------|
| `knowledge.reasoning.pass.started` | LOW | no | KnowledgeReasoningEngine | Telemetry | `{trigger: BOOT|INCREMENTAL|ON_DEMAND, dirty_set_size: int}` |
| `knowledge.reasoning.pass.completed` | LOW | no | KnowledgeReasoningEngine | Telemetry, dashboards | `{duration_ms, rules_fired, facts_derived, iterations, inconsistencies_detected, coverage_gaps_detected}` |
| `knowledge.reasoning.fact.derived` | BACKGROUND | no | KnowledgeReasoningEngine | Telemetry | `{rule_id, fact_type, subject_id}` |
| `knowledge.reasoning.inconsistency.detected` | HIGH | **yes** | KnowledgeReasoningEngine | KnowledgeRecommendationEngine, application layer | `{rule_id, inconsistency_type, affected_objects: list[str], severity: ERROR|WARNING}` |

### 2.8 knowledge.coverage.*

Events from `KnowledgeCoverageEngine`.

| Event Type | Priority | Persistent | Publisher | Typical Consumers | Payload |
|-----------|----------|-----------|-----------|------------------|---------|
| `knowledge.coverage.gap.detected` | NORMAL | no | KnowledgeCoverageEngine | KnowledgeRecommendationEngine | `{object_id, dimension: architecture|implementation|testing|desktop, score: float, threshold: float}` |
| `knowledge.coverage.updated` | LOW | no | KnowledgeCoverageEngine | Dashboards, health scorer | `{object_id, dimension, old_score: float, new_score: float}` |
| `knowledge.coverage.report.ready` | LOW | no | KnowledgeCoverageEngine | Application layer | `{report_id, object_count, gap_count, average_score: float}` |

### 2.9 knowledge.evidence.*

Events from `KnowledgeEvidenceEngine`.

| Event Type | Priority | Persistent | Publisher | Typical Consumers | Payload |
|-----------|----------|-----------|-----------|------------------|---------|
| `knowledge.evidence.verified` | NORMAL | **yes** | KnowledgeEvidenceEngine | KnowledgeCoverageEngine, KnowledgeReasoningEngine | `{evidence_id, object_id, evidence_type, verified_by, confidence: float}` |
| `knowledge.evidence.expired` | NORMAL | **yes** | KnowledgeEvidenceEngine | KnowledgeReasoningEngine (COV-RULE-004), application layer | `{evidence_id, object_id, expired_at, last_verified}` |
| `knowledge.evidence.failed` | HIGH | no | KnowledgeEvidenceEngine | Application layer | `{evidence_id, object_id, failure_reason}` |

### 2.10 knowledge.snapshot.*

Events from `KnowledgeSnapshotManager`.

| Event Type | Priority | Persistent | Publisher | Typical Consumers | Payload |
|-----------|----------|-----------|-----------|------------------|---------|
| `knowledge.snapshot.created` | HIGH | **yes** | KnowledgeSnapshotManager | Replication, backup | `{snapshot_id, object_count, size_bytes, duration_ms, snapshot_type: FULL|INCREMENTAL}` |
| `knowledge.snapshot.restored` | HIGH | **yes** | KnowledgeSnapshotManager | KnowledgeRuntime (post-recovery) | `{snapshot_id, objects_restored, duration_ms}` |
| `knowledge.snapshot.expired` | LOW | no | KnowledgeSnapshotManager | Backup monitors | `{snapshot_id, expired_at, retention_policy}` |

### 2.11 knowledge.version.*

Events from `KnowledgeVersionManager`.

| Event Type | Priority | Persistent | Publisher | Typical Consumers | Payload |
|-----------|----------|-----------|-----------|------------------|---------|
| `knowledge.version.bumped` | NORMAL | **yes** | KnowledgeVersionManager | KnowledgeStore, replication | `{knowledge_id, old_version, new_version, bump_type: MAJOR|MINOR|PATCH}` |
| `knowledge.version.pinned` | NORMAL | **yes** | KnowledgeVersionManager | KnowledgeQueryEngine | `{knowledge_id, pinned_version, pinned_by, reason}` |

### 2.12 knowledge.cache.*

Events from `KnowledgeCacheManager`.

| Event Type | Priority | Persistent | Publisher | Typical Consumers | Payload |
|-----------|----------|-----------|-----------|------------------|---------|
| `knowledge.cache.evicted` | BACKGROUND | no | KnowledgeCacheManager | Telemetry | `{cache_key, tier: HOT|WARM|COLD, eviction_reason: TTL|LRU|INVALIDATE, age_ms}` |
| `knowledge.cache.warmed` | LOW | no | KnowledgeCacheManager | Telemetry | `{tier, key_count, warm_duration_ms}` |
| `knowledge.cache.invalidated` | LOW | no | KnowledgeCacheManager | Telemetry | `{cause_event_type, invalidated_keys: int}` |

### 2.13 knowledge.recommendation.*

Events from `KnowledgeRecommendationEngine`.

| Event Type | Priority | Persistent | Publisher | Typical Consumers | Payload |
|-----------|----------|-----------|-----------|------------------|---------|
| `knowledge.recommendation.generated` | LOW | no | KnowledgeRecommendationEngine | Application layer, dashboards | `{recommendation_id, type, priority: int, affected_object_id, auto_fixable: bool}` |
| `knowledge.recommendation.applied` | NORMAL | **yes** | KnowledgeRecommendationEngine | Application layer | `{recommendation_id, applied_by, applied_at, result}` |

### 2.14 knowledge.validation.*

Events from `KnowledgeSchemaValidator`.

| Event Type | Priority | Persistent | Publisher | Typical Consumers | Payload |
|-----------|----------|-----------|-----------|------------------|---------|
| `knowledge.validation.error` | HIGH | no | KnowledgeSchemaValidator | Application layer, CI pipeline | `{knowledge_id, rule_id, error_code, message, field_path}` |
| `knowledge.validation.warning` | NORMAL | no | KnowledgeSchemaValidator | Application layer | `{knowledge_id, rule_id, warning_code, message}` |
| `knowledge.validation.passed` | LOW | no | KnowledgeSchemaValidator | Telemetry | `{knowledge_id, rules_checked: int, duration_ms}` |

### 2.15 knowledge.compiler.*

Events from `KnowledgeCompiler` and its backends.

| Event Type | Priority | Persistent | Publisher | Typical Consumers | Payload |
|-----------|----------|-----------|-----------|------------------|---------|
| `knowledge.compiler.output.ready` | NORMAL | no | KnowledgeCompiler | Application layer | `{target: markdown|html|promptpack|llmmemory|dataset, artifact_count, output_path}` |
| `knowledge.compiler.failed` | HIGH | no | KnowledgeCompiler | Application layer, CI pipeline | `{target, error_code, error_message, failed_object_id}` |
| `knowledge.compiler.started` | LOW | no | KnowledgeCompiler | Telemetry | `{target, object_count, triggered_by}` |

### 2.16 knowledge.replication.*

Events from `KnowledgeReplicationManager`.

| Event Type | Priority | Persistent | Publisher | Typical Consumers | Payload |
|-----------|----------|-----------|-----------|------------------|---------|
| `knowledge.replication.synced` | LOW | no | KnowledgeReplicationManager | Replication monitors | `{replica_id, objects_synced, lag_ms, lsn}` |
| `knowledge.replication.lag.high` | HIGH | no | KnowledgeReplicationManager | Platform monitors, alerting | `{replica_id, lag_ms, lag_threshold_ms}` |
| `knowledge.replication.failed` | CRITICAL | **yes** | KnowledgeReplicationManager | Platform monitors | `{replica_id, error_code, last_successful_sync}` |

### 2.17 knowledge.session.*

Events from `KnowledgeSessionManager`.

| Event Type | Priority | Persistent | Publisher | Typical Consumers | Payload |
|-----------|----------|-----------|-----------|------------------|---------|
| `knowledge.session.started` | LOW | no | KnowledgeSessionManager | Application layer | `{session_id, client_id, query_mode}` |
| `knowledge.session.expired` | NORMAL | no | KnowledgeSessionManager | Application layer | `{session_id, duration_ms, queries_executed}` |
| `knowledge.session.terminated` | LOW | no | KnowledgeSessionManager | Telemetry | `{session_id, termination_reason: TIMEOUT|CLIENT_CLOSE|SHUTDOWN}` |

---

## 3. Event Count Summary

| Subsystem | Event Types | Persistent Events |
|-----------|-------------|------------------|
| knowledge.runtime.* | 11 | 5 |
| knowledge.object.* | 7 | 7 |
| knowledge.store.* | 4 | 0 |
| knowledge.index.* | 3 | 0 |
| knowledge.graph.* | 8 | 1 |
| knowledge.query.* | 5 | 0 |
| knowledge.reasoning.* | 4 | 1 |
| knowledge.coverage.* | 3 | 0 |
| knowledge.evidence.* | 3 | 2 |
| knowledge.snapshot.* | 3 | 2 |
| knowledge.version.* | 2 | 2 |
| knowledge.cache.* | 3 | 0 |
| knowledge.recommendation.* | 2 | 1 |
| knowledge.validation.* | 3 | 0 |
| knowledge.compiler.* | 3 | 0 |
| knowledge.replication.* | 3 | 1 |
| knowledge.session.* | 3 | 0 |
| **Total** | **71** | **22** |

---

## 4. Event Subscription Patterns

`KnowledgeSubscriptions` provides convenience helpers over the raw `PlatformEventBus` API. All helpers return a `SubscriptionHandle` that can be used to unsubscribe.

### 4.1 Available Subscription Helpers

```
KnowledgeSubscriptions:

  on_object_lifecycle_changed(domain: str, callback: Callable[[LifecycleEvent], None])
    → SubscriptionHandle
    # Fires whenever knowledge.object.lifecycle.changed is published
    # for an object in the given domain. Pass domain=None to subscribe
    # to all domains.

  on_graph_changed(callback: Callable[[GraphChangeEvent], None])
    → SubscriptionHandle
    # Fires on any of: node.added, node.removed, edge.added, edge.removed,
    # graph.rebuilt. Useful for cache invalidation.

  on_inconsistency_detected(category: str | None, callback: Callable[[InconsistencyEvent], None])
    → SubscriptionHandle
    # Fires when knowledge.reasoning.inconsistency.detected is published.
    # category filters by inconsistency_type prefix (e.g. "CIRCULAR_DEPENDENCY").
    # Pass category=None to receive all inconsistencies.

  on_coverage_gap(domain: str | None, threshold: float, callback: Callable[[CoverageGapEvent], None])
    → SubscriptionHandle
    # Fires when knowledge.coverage.gap.detected is published with score < threshold.
    # Filters to the given domain; pass domain=None for all domains.

  on_runtime_ready(callback: Callable[[RuntimeReadyEvent], None])
    → SubscriptionHandle
    # Fires once when knowledge.runtime.ready is published.
    # If the runtime is already ready, the callback fires immediately.

  on_recommendation_generated(type_filter: str | None, callback: Callable[[RecommendationEvent], None])
    → SubscriptionHandle
    # Fires on knowledge.recommendation.generated.
    # Optionally filter by recommendation type string prefix.
```

### 4.2 Event Correlation

Operations that span multiple events use `correlation_id` to group related events. The following operations produce correlated event sequences:

| Operation | Events in Correlated Sequence |
|-----------|-------------------------------|
| Object write | `store.written` → `index.updated` → `graph.edge.added` → `reasoning.pass.completed` |
| Boot | `boot.phase1.complete` → ... → `boot.phase4.complete` → `boot.complete` → `runtime.ready` |
| Recovery | `recovery.started` → `snapshot.restored` → `runtime.boot.complete` → `recovery.completed` |
| Compilation | `compiler.started` → `compiler.output.ready` (or `compiler.failed`) |

Consumers can reconstruct operation timelines by filtering on `correlation_id`.

### 4.3 Dead Letter Handling

If a consumer throws an exception while handling a CRITICAL or HIGH event, the Platform Event Bus:
1. Retries delivery up to 3 times with exponential back-off (100 ms, 500 ms, 2 s).
2. If all retries fail, writes the event to the dead-letter queue (`knowledge.dlq`).
3. Publishes a `platform.event.dlq.entry` event for monitoring.

NORMAL, LOW, and BACKGROUND events that fail are logged but not retried.

---

## 5. Event Flow Diagrams (Textual)

### 5.1 Write Path Event Flow

```
Author writes object
        │
        ▼
KnowledgeStore.write()
        │
        ├──► knowledge.object.created (NORMAL, persistent)
        │           │
        │           ▼
        │    KnowledgeIndexer.handle()
        │           │
        │           └──► knowledge.index.updated (NORMAL)
        │                        │
        │                        ▼
        │               KnowledgeQueryEngine
        │               (cache invalidation)
        │
        └──► knowledge.store.written (NORMAL)
                    │
                    ├──► KnowledgeGraphRuntime.handle()
                    │           │
                    │           └──► knowledge.graph.edge.added (NORMAL)
                    │
                    └──► KnowledgeReasoningEngine.schedule_incremental()
                                │
                            [100ms debounce]
                                │
                                ▼
                    knowledge.reasoning.pass.started (LOW)
                                │
                    [forward chain runs]
                                │
                                ▼
                    knowledge.reasoning.pass.completed (LOW)
                                │
                    [if INCONSISTENCY facts]
                                │
                                ▼
                    knowledge.reasoning.inconsistency.detected (HIGH, persistent)
```

### 5.2 Boot Event Flow

```
KnowledgeRuntime.boot()
        │
        ├── Phase 1 ──► knowledge.runtime.boot.phase1.complete
        ├── Phase 2 ──► knowledge.runtime.boot.phase2.complete
        ├── Phase 3 ──► knowledge.runtime.boot.phase3.complete
        ├── Phase 4 ──► knowledge.runtime.boot.phase4.complete
        │
        ▼
knowledge.runtime.boot.complete (HIGH, persistent)
        │
        ▼
[full reasoning pass]
        │
        ▼
knowledge.runtime.ready (HIGH)
```

---

## 6. Cross-References

| Document | Relationship |
|----------|-------------|
| `KR-ARCH-008` (08-REASONING-MODEL.md) | Defines events emitted by reasoning engine |
| `KR-ARCH-010` (10-DIAGRAMS.md) | Visual event flow diagrams |
| `KR-ARCH-011` (11-VERIFICATION.md) | Complete event catalog verification table |
| `architecture/docs/EVENT-CATALOG.md` | Platform-wide event catalog (parent catalog) |
