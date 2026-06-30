# KNW-KC-ARCH-008 — Version Engine

**Phase:** 3.0C  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

The Version Engine governs how Knowledge Objects evolve over time. Every mutation produces a new version. Every version is immutable after publication. History is never deleted.

---

## Semantic Versioning

All Knowledge Objects use semantic versioning: `MAJOR.MINOR.PATCH`

| Component | Increment When |
|-----------|---------------|
| MAJOR | Breaking change — incompatible with previous consumers |
| MINOR | Backward-compatible addition — new fields, new relationships |
| PATCH | Backward-compatible fix — typo, clarification, metadata only |

### Version Rules

| Rule | Description |
|------|-------------|
| VR-001 | Initial version is always `1.0.0` |
| VR-002 | DRAFT objects may freely increment PATCH version |
| VR-003 | Transition from DRAFT→REVIEW bumps MINOR at minimum |
| VR-004 | Transition to CANONICAL locks version — only PATCH bumps allowed thereafter |
| VR-005 | MAJOR bump requires Architecture Board sign-off for objects in APPROVED or CANONICAL state |
| VR-006 | Version `0.x.x` is forbidden — all objects start at `1.0.0` |

---

## Version Record Schema

```yaml
version_record:
  version: "1.3.0"
  knowledge_id: "KNW-KOS-ARCH-001"
  global_id: "a3f2c1d4-..."
  parent_version: "1.2.0"
  change_type: MAJOR|MINOR|PATCH
  change_summary: string
  changed_by: string
  changed_at: datetime
  checksum: "sha256:..."
  content_snapshot_ref: string     # pointer to frozen content in Snapshot Engine
  lifecycle_status_at_change: DRAFT|REVIEW|...
  breaking_changes: list[string]
  migration_notes: string
  is_published: bool               # published versions are immutable
```

---

## Version Graph

Each object's version history forms a directed acyclic graph:

```
1.0.0 → 1.1.0 → 1.2.0 → 1.3.0 (current)
                    ↘
                   1.2.1 (hotfix branch)
```

### Branch Model

| Branch Type | Description |
|-------------|-------------|
| `main` | Default branch — all canonical versions |
| `hotfix` | Emergency patch applied to published version |
| `experimental` | Exploratory change not yet on main path |

### Branch Rules

- Branches merge back to `main` before reaching CANONICAL
- `hotfix` branches may be published without reaching CANONICAL
- At most one `experimental` branch per object at any time

---

## Fork and Merge

### Fork
```
FORK(source_id, source_version):
  1. Create new KnowledgeObject with new global_id
  2. Copy content from source at source_version
  3. Set lineage = "{source_id}@{source_version}"
  4. Set version = "1.0.0" (new lineage starts fresh)
  5. Create relationship: new EXTENDS source (or SUPERSEDES if replacing)
```

### Merge
```
MERGE(branch_version → main_version):
  1. Compute diff(branch, main)
  2. Resolve conflicts (human review required if semantic conflict)
  3. Create new main version with MINOR bump
  4. Record merge_from: branch_version in version record
  5. Deprecate branch version
```

---

## Diff Engine

```
DIFF(version_a, version_b):
  Returns: VersionDiff
    - added_fields: list[FieldChange]
    - removed_fields: list[FieldChange]
    - modified_fields: list[FieldChange]
    - added_relations: list[RelationRef]
    - removed_relations: list[RelationRef]
    - is_breaking: bool
    - estimated_change_type: MAJOR|MINOR|PATCH
```

---

## Compatibility Matrix

| From Version | To Version | Compatible? | Notes |
|-------------|-----------|-------------|-------|
| 1.0.0 | 1.1.0 | Yes | Additive only |
| 1.0.0 | 1.2.0 | Yes | Additive only |
| 1.0.0 | 2.0.0 | No | Breaking change — migration required |
| 1.2.0 | 1.1.0 | No | Downgrade — never permitted for CANONICAL |

---

## Migration Specification

When MAJOR is bumped, a Migration Specification is required:

```yaml
migration:
  from_version: "1.x.x"
  to_version: "2.0.0"
  type: AUTOMATIC|ASSISTED|MANUAL
  steps:
    - field: "old_field_name"
      action: RENAME
      new_name: "new_field_name"
    - field: "removed_field"
      action: REMOVE
      replacement: "new_field"
  backwards_compatible: false
  migration_script: "scripts/migrate_v1_to_v2.py"
  rollback_supported: true
```

---

## Version Query Interface

```python
class VersionEngine(Protocol):
    def get_version(self, knowledge_id: str, version: str) -> VersionRecord: ...
    def get_all_versions(self, knowledge_id: str) -> list[VersionRecord]: ...
    def get_latest(self, knowledge_id: str) -> VersionRecord: ...
    def get_stable(self, knowledge_id: str) -> VersionRecord: ...
    def diff(self, knowledge_id: str, v1: str, v2: str) -> VersionDiff: ...
    def fork(self, knowledge_id: str, version: str) -> str: ...
    def merge(self, source_id: str, target_id: str) -> VersionRecord: ...
    def get_successors(self, knowledge_id: str, version: str) -> list[str]: ...
    def get_predecessors(self, knowledge_id: str, version: str) -> list[str]: ...
    def is_compatible(self, v1: str, v2: str) -> bool: ...
```

---

## Cross-References

- Snapshot capture → `09-SNAPSHOT-ENGINE`
- Lifecycle triggers → `34-STATE-MACHINES`
- URI version channels → `04-URI-SPECIFICATION`
- Registry versioning → `14-REGISTRY-ARCHITECTURE`
