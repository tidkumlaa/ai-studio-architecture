---
knowledge_id: KNW-PLAT-ARCH-024
title: "Platform Rollback Engine"
status: AUTHORITATIVE
phase: "2.1D.0"
version: "1.0.0"
created: "2026-06-29"
purpose: "Specify rollback procedures, triggers, mechanisms, and verification for failed migrations"
canonical_source: "architecture/docs/platform-consolidation/24-ROLLBACK-ENGINE.md"
dependencies:
  - "22-MIGRATION-ENGINE.md"
  - "19-STATE-MACHINES.md"
related_documents:
  - "25-VERIFICATION.md"
  - "23-IMPORT-REWRITE.md"
acceptance_criteria:
  - "Rollback is automated for all trigger conditions"
  - "Rollback leaves working tree identical to pre-migration state"
  - "Rollback is idempotent"
verification_checklist:
  - "[ ] Rollback procedure verified in dry-run mode"
  - "[ ] Post-rollback verification defined"
  - "[ ] Manual rollback path documented"
future_extensions:
  - "Partial rollback: revert only failed phase"
  - "Rollback diff report"
---

# Platform Rollback Engine

## Rollback Philosophy

Migration steps either **fully succeed** or **fully roll back**. There is no partial migration state.
The rollback engine restores the repository to a verified-clean git snapshot.

---

## Pre-Migration Snapshot

Before any migration step, the engine creates a snapshot:

```bash
# Pre-migration snapshot
git add -A
git stash push -m "pre-migration-snapshot"
git tag migration/phase-2.1D.1/start

# Or if already committed:
git tag migration/phase-2.1D.1/start HEAD
```

The snapshot tag is stored in `MigrationConfig.snapshot_tag_prefix`.

---

## Rollback Trigger Conditions

See 22-MIGRATION-ENGINE.md for the full trigger list. Summary:

| Trigger | Source |
|---------|--------|
| Import verification fails | `python -c "import X"` exits non-zero |
| Test suite failure | pytest exits non-zero |
| File count mismatch | copy verification |
| Checksum mismatch | SHA256 comparison |
| Circular import | `platform-verify imports --circular` |
| Layer violation | `platform-verify layers --strict` |
| Timeout (> 10 min per phase) | watchdog timer |

---

## Rollback Procedure

### Step 1 — Stop all migration activity
```python
migration_engine.halt()
# Cancels any in-progress file operations
# Sets migration state machine → ROLLBACK
```

### Step 2 — Restore from snapshot
```bash
# Hard reset to pre-migration snapshot
git reset --hard migration/phase-2.1D.1/start

# If files were added (not just modified):
git clean -fd --exclude="*.log"
```

### Step 3 — Post-rollback verification
```bash
# Verify working tree matches snapshot
git status  # must show: nothing to commit, working tree clean

# Verify imports still work
python -c "from platform.ai_runtime.resource_intelligence import ResourceIntelligenceService"

# Run test suite on restored state
pytest platform/tests/ -x --tb=short
```

### Step 4 — Emit rollback event
```python
event_bus.publish(PlatformEvent(
    topic="platform.migration.rollback.completed",
    payload={
        "snapshot": "migration/phase-2.1D.1/start",
        "trigger": trigger_description,
        "phase": failed_phase,
        "duration_ms": rollback_duration,
    }
))
```

### Step 5 — Generate rollback report
```
reports/migration/rollback-{timestamp}.md
```

---

## Rollback Engine Interface

```python
class RollbackEngine:
    def __init__(self, config: MigrationConfig) -> None: ...

    def create_snapshot(self) -> str: ...
    # Creates git tag; returns tag name

    def rollback(self, to_snapshot: str) -> RollbackResult: ...
    # Hard reset + clean; returns result

    def verify_rollback(self) -> list[str]: ...
    # Returns list of problems; empty = clean

    def is_idempotent(self, snapshot: str) -> bool: ...
    # True if running rollback again would be a no-op
```

---

## RollbackResult

```python
@dataclass(frozen=True)
class RollbackResult:
    success: bool
    snapshot_restored: str
    files_restored: int
    files_deleted: int
    duration_ms: int
    errors: list[str]
    post_verification_passed: bool
```

---

## Manual Rollback Path

If the automated rollback fails (e.g., git is in a conflicted state):

```bash
# 1. Find the snapshot tag
git tag | grep migration/phase-2.1D.1

# 2. Hard reset
git reset --hard migration/phase-2.1D.1/start

# 3. Remove untracked migration artifacts
git clean -fd

# 4. Verify
git status
pytest platform/tests/ -x

# 5. Delete the failed migration tag (optional cleanup)
git tag -d migration/phase-2.1D.1/start
```

---

## Rollback Log Format

```
[2026-06-29T12:01:00Z] ROLLBACK triggered
  reason="test_failure"
  phase=1
  step=1.7
  snapshot=migration/phase-2.1D.1/start

[2026-06-29T12:01:01Z] ROLLBACK git_reset snapshot=migration/phase-2.1D.1/start status=OK

[2026-06-29T12:01:02Z] ROLLBACK git_clean status=OK files_removed=4

[2026-06-29T12:01:05Z] ROLLBACK verification status=PASS tests=160/160

[2026-06-29T12:01:05Z] ROLLBACK complete duration_ms=5000 state=IDLE
```

---

## Non-Rollbackable Operations

The following operations, if performed, cannot be automatically rolled back:

| Operation | Why |
|-----------|-----|
| `git push` after migration | Remote history changed |
| Deleting source repos outside git | No git history for external repos |
| Database schema changes | Out of scope for this engine |

**Guard:** The migration engine checks `git remote` before any operation. If a remote
push would be triggered, the engine aborts with an error requiring explicit `--allow-push` flag.
