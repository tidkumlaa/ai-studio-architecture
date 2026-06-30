# KNW-KC-ARCH-018 — Event Registry

**Phase:** 3.0C  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

The Event Registry is the authoritative catalog of all domain events emitted by the KOS and platform. Every event has a topic, schema, producer, and set of registered consumers. No implicit events.

---

## Topic Convention

```
{domain}.{subject}.{verb}[.{qualifier}]

Examples:
  knowledge.object.registered
  knowledge.object.lifecycle_changed
  registry.index.rebuilt
  platform.service.boot_completed
  evidence.record.added
  evidence.record.stale
  quality.score.below_threshold
```

---

## Event Domains

| Domain | Subject | Events |
|--------|---------|--------|
| `knowledge` | object, graph, registry | registered, updated, deprecated, tombstoned |
| `lifecycle` | object | draft, review, verified, approved, canonical, deprecated, archived |
| `evidence` | record | added, stale, expired, disputed, resolved |
| `quality` | score | computed, below_threshold, alert |
| `registry` | object, index | registered, updated, rebuilt, health_alert |
| `relationship` | relation | created, updated, deprecated, removed, cycle_detected |
| `version` | object | published, branched, merged, forked |
| `snapshot` | snapshot | captured, verified, restored, expired |
| `platform` | service, runtime | boot_started, boot_completed, shutdown_started, shutdown_completed |
| `traceability` | chain | gap_detected, coverage_changed |

---

## Event Record Schema

```yaml
event:
  event_id: "EVT-{domain}-{verb}-{timestamp}-{sequence}"
  topic: "knowledge.object.registered"
  schema_version: "1.0.0"

  # Header
  produced_at: datetime
  producer_id: string              # knowledge_id of emitting component
  correlation_id: string           # trace across related events
  causation_id: string | null      # event_id that caused this event

  # Payload
  subject_id: string               # knowledge_id of affected object
  subject_type: KnowledgeObjectType
  payload: dict                    # event-specific data (see schemas below)

  # Delivery
  partitioning_key: string         # used for ordered delivery
  priority: CRITICAL|HIGH|NORMAL|LOW
  ttl_seconds: int                 # 0 = no expiry
  retry_count: int
```

---

## Core Event Schemas

### `knowledge.object.registered`
```yaml
payload:
  knowledge_id: string
  namespace: string
  object_type: string
  version: string
  owner: string
```

### `lifecycle.object.status_changed`
```yaml
payload:
  knowledge_id: string
  from_status: string
  to_status: string
  changed_by: string
  reason: string
```

### `evidence.record.stale`
```yaml
payload:
  evidence_id: string
  object_id: string
  evidence_type: string
  freshness_score: float
  days_stale: int
```

### `quality.score.below_threshold`
```yaml
payload:
  knowledge_id: string
  overall_score: float
  required_score: float
  lifecycle_status: string
  dim_scores: dict[str, float]
  alerts: list[str]
```

### `traceability.chain.gap_detected`
```yaml
payload:
  requirement_id: string
  gap_type: UNARCHITECTED|UNIMPLEMENTED|UNTESTED|UNDEPLOYED
  missing_link_type: string
  coverage_score: float
```

---

## Event Registry Entry Format

```yaml
# knowledge/registry/events/knowledge.object.registered.yaml
identity:
  knowledge_id: "KNW-KC-EVT-knowledge-registered"
  canonical_name: "event.knowledge.object.registered"
  knowledge_uri: "knw://kos/event/knowledge-object-registered"
  namespace: "kos"
  version: "1.0.0"

event_spec:
  topic: "knowledge.object.registered"
  schema_version: "1.0.0"
  payload_schema: {...}       # JSON Schema of payload
  producer_ids: ["KNW-KC-SVC-registry"]
  consumer_ids: ["KNW-KC-SVC-graph", "KNW-KC-SVC-search"]
  ordering_guarantee: PARTITION_ORDERED
  delivery_guarantee: AT_LEAST_ONCE
  max_payload_bytes: 65536
  retention_days: 30
```

---

## Delivery Guarantees

| Guarantee | Description |
|-----------|-------------|
| `AT_LEAST_ONCE` | Default. Event may be delivered more than once. Consumers must be idempotent. |
| `EXACTLY_ONCE` | For CRITICAL events. Requires distributed transaction. |
| `AT_MOST_ONCE` | For telemetry. Loss is acceptable. |

---

## Event Routing Rules (from Phase 2.1D.0 Event Model)

| Rule | Description |
|------|-------------|
| ER-001 | All KOS events use the `{domain}.{subject}.{verb}` topic convention |
| ER-002 | Payload ≤ 64 KB |
| ER-003 | Every event carries `correlation_id` for tracing |
| ER-004 | Dead-letter queue after 3 failed delivery attempts |
| ER-005 | CRITICAL priority events bypass rate limiting |
| ER-006 | All events are persisted for `retention_days` before expiry |

---

## Event Registry Protocol

```python
class EventRegistry(KnowledgeRegistryProtocol):
    def register_event_type(self, spec: EventSpec) -> str: ...
    def get_event_type(self, topic: str) -> EventSpec: ...
    def get_producers(self, topic: str) -> list[str]: ...
    def get_consumers(self, topic: str) -> list[str]: ...
    def register_consumer(self, topic: str, consumer_id: str) -> None: ...
    def validate_payload(self, topic: str, payload: dict) -> bool: ...
    def get_by_domain(self, domain: str) -> list[EventSpec]: ...
```

---

## Cross-References

- Registry base contract → `14-REGISTRY-ARCHITECTURE`
- Events in Phase 2.1D.0 → `18-EVENT-MODEL`
- Event consumers include Graph Engine → `24-KNOWLEDGE-GRAPH-MODEL`
- Event consumers include Search Engine → `28-SEARCH-ENGINE`
