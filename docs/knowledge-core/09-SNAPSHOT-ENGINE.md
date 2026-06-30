# KNW-KC-ARCH-009 — Snapshot Engine

**Phase:** 3.0C  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

The Snapshot Engine captures point-in-time immutable copies of Knowledge Objects and the Knowledge Graph. Snapshots support rollback, auditing, forensics, and offline compilation.

---

## Snapshot Types

| Type | Trigger | Scope | Retention |
|------|---------|-------|-----------|
| `VERSION_SNAPSHOT` | On every version publication | Single object | Forever |
| `LIFECYCLE_SNAPSHOT` | On every lifecycle transition | Single object | Forever |
| `DAILY_SNAPSHOT` | Nightly automated | Full registry | 90 days |
| `RELEASE_SNAPSHOT` | On CANONICAL promotion | Full registry | Forever |
| `PRE_MIGRATION_SNAPSHOT` | Before any migration | Affected objects | Until migration verified |
| `MANUAL_SNAPSHOT` | Human-triggered | Custom scope | Until manually deleted |

---

## Snapshot Record Schema

```yaml
snapshot:
  snapshot_id: "SNAP-{type}-{timestamp}-{scope_hash}"
  snapshot_type: VERSION_SNAPSHOT|LIFECYCLE_SNAPSHOT|DAILY_SNAPSHOT|RELEASE_SNAPSHOT|PRE_MIGRATION_SNAPSHOT|MANUAL_SNAPSHOT
  scope:
    type: SINGLE_OBJECT|NAMESPACE|FULL_REGISTRY|CUSTOM
    object_ids: list[string]
    namespace: string | null
  captured_at: datetime
  captured_by: string
  trigger_event: string
  checksum: "sha256:..."
  size_bytes: int
  object_count: int
  relation_count: int
  storage_path: string
  compressed: bool
  encryption_key_id: string | null
  tags: list[string]
  metadata:
    git_commit: string | null
    platform_version: string
    kos_version: string
```

---

## Snapshot Content Format

Each snapshot stores a complete, self-contained bundle:

```
snapshot-SNAP-RELEASE-20260101-abc123/
  manifest.yaml           # snapshot metadata
  objects/
    {knowledge_id}.yaml   # one file per object (full content at snapshot time)
  relations/
    relations.yaml        # all active relations
  registry/
    index.yaml            # registry index at snapshot time
  graph/
    graph.yaml            # full graph adjacency list
  checksums.sha256        # per-file checksums
```

---

## Snapshot Lifecycle

```
CREATED → VERIFIED → AVAILABLE → EXPIRED | RETAINED_FOREVER

CREATED:          Snapshot job started, files being written
VERIFIED:         Checksums validated, manifest complete
AVAILABLE:        Ready for use (restore, diff, compile)
EXPIRED:          Past retention date, eligible for deletion
RETAINED_FOREVER: Tagged as permanent (release or lifecycle snapshots)
```

---

## Restore Protocol

```
RESTORE(snapshot_id, target_scope):
  PRE-CONDITIONS:
    1. snapshot.status == AVAILABLE
    2. snapshot.checksum verified
    3. User has RESTORE permission
    4. Current state snapshotted (PRE_MIGRATION_SNAPSHOT auto-created)

  RESTORE STEPS:
    1. Load snapshot bundle
    2. Validate all object checksums
    3. For each object in scope:
       a. Write snapshot content to registry
       b. Set version = snapshot version
       c. Set lifecycle status = snapshot lifecycle status
    4. Rebuild graph from snapshot relations
    5. Rebuild registry index
    6. Validate restored state
    7. Emit RESTORE_COMPLETED event

  ROLLBACK (if restore fails):
    1. Restore from PRE_MIGRATION_SNAPSHOT taken in step 4 above
    2. Emit RESTORE_FAILED event
```

---

## Snapshot Comparison (Diff)

```
SNAPSHOT_DIFF(snap_a, snap_b):
  Returns: SnapshotDiff
    - new_objects: list[knowledge_id]
    - deleted_objects: list[knowledge_id]
    - modified_objects: list[ObjectDiff{id, field_changes}]
    - new_relations: list[relation_id]
    - deleted_relations: list[relation_id]
    - lifecycle_changes: list[LifecycleChange]
    - summary: {added: int, removed: int, modified: int}

COMPLEXITY: O(N + R) where N=objects, R=relations
```

---

## Tombstone Model

When an object is permanently removed, a tombstone record is preserved:

```yaml
tombstone:
  knowledge_id: "KNW-KOS-ARCH-OLD-001"
  global_id: "..."
  knowledge_uri: "knw://kos/arch/old-001"
  tombstoned_at: datetime
  tombstoned_by: string
  reason: string
  successor_id: "KNW-KOS-ARCH-NEW-001" | null
  last_snapshot_ref: "SNAP-LIFECYCLE-..."
  preserved_forever: true
```

Tombstones are stored in a dedicated Tombstone Registry. The URI `knw://kos/arch/old-001` resolves to the tombstone — never returns 404.

---

## Snapshot Engine Protocol

```python
class SnapshotEngine(Protocol):
    def capture(
        self,
        scope: SnapshotScope,
        snapshot_type: SnapshotType,
        trigger: str,
    ) -> str: ...                            # returns snapshot_id

    def restore(
        self,
        snapshot_id: str,
        target_scope: SnapshotScope | None,
    ) -> RestoreResult: ...

    def diff(self, snap_a: str, snap_b: str) -> SnapshotDiff: ...

    def get(self, snapshot_id: str) -> SnapshotRecord: ...

    def list_snapshots(
        self,
        scope: SnapshotScope | None,
        snapshot_type: SnapshotType | None,
    ) -> list[SnapshotRecord]: ...

    def verify(self, snapshot_id: str) -> bool: ...

    def expire(self, snapshot_id: str) -> None: ...

    def get_tombstone(self, knowledge_id: str) -> TombstoneRecord | None: ...
```

---

## Performance Requirements

| Operation | P50 | P99 |
|-----------|-----|-----|
| Single-object snapshot | < 5ms | < 20ms |
| Namespace snapshot (≤1000 objects) | < 500ms | < 2s |
| Full registry snapshot (≤1M objects) | < 60s | < 300s |
| Restore single object | < 10ms | < 50ms |
| Snapshot diff | < 100ms | < 500ms |

---

## Cross-References

- Version snapshots triggered by → `08-VERSION-ENGINE`
- Lifecycle snapshots triggered by → `34-STATE-MACHINES`
- Graph included in snapshots → `24-KNOWLEDGE-GRAPH-MODEL`
- Registry included in snapshots → `14-REGISTRY-ARCHITECTURE`
