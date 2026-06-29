---
knowledge_id: KNW-PLAT-ARCH-022
title: "Platform Migration Engine"
status: AUTHORITATIVE
phase: "2.1D.0"
version: "1.0.0"
created: "2026-06-29"
purpose: "Specify the migration engine design: what moves, how it moves, verification, and rollback triggers"
canonical_source: "architecture/docs/platform-consolidation/22-MIGRATION-ENGINE.md"
dependencies:
  - "19-STATE-MACHINES.md"
  - "03-DIRECTORY-ARCHITECTURE.md"
  - "12-IMPORT-RULES.md"
related_documents:
  - "23-IMPORT-REWRITE.md"
  - "24-ROLLBACK-ENGINE.md"
  - "25-VERIFICATION.md"
acceptance_criteria:
  - "Migration steps are atomic at the file level"
  - "Every migration step is reversible"
  - "Migration engine does NOT run during Phase 2.1D.0"
verification_checklist:
  - "[ ] All migration steps listed"
  - "[ ] Each step has a verification check"
  - "[ ] Rollback trigger conditions defined"
future_extensions:
  - "Parallel file migration for large runtimes"
  - "Incremental migration with partial activation"
---

# Platform Migration Engine

## IMPORTANT — Phase Gate

```
╔═══════════════════════════════════════════════════════════╗
║  THE MIGRATION ENGINE DOES NOT RUN DURING PHASE 2.1D.0   ║
║  Phase 2.1D.0 = Architecture Specification Only           ║
║  Migration begins in Phase 2.1D.1 after Architecture Freeze ║
╚═══════════════════════════════════════════════════════════╝
```

This document specifies the migration engine **design**. No code runs yet.

---

## Migration Scope

### What Moves

| Source | Destination | Operation |
|--------|-------------|-----------|
| `platform_kernel/` | `platform/kernel/` | COPY then validate, then delete source |
| `platform_sdk/` | `platform/sdk/` | COPY then validate, then delete source |
| `platform/ai_runtime/` | `platform/runtimes/ai_runtime/` | MOVE within repo |

### What Does NOT Move

| Item | Reason |
|------|--------|
| `platform/` existing modules | Already canonical — no movement needed |
| `source/*/` product directories | Products are consumers, not migrated |
| `tests/` directories | Stays alongside modules |
| `architecture/` | Documentation only |

---

## Migration Phases

### Phase 0 — Pre-Migration Checks
```
1. Assert git working tree is clean
2. Assert all 37 architecture documents exist
3. Assert Architecture Freeze document is FROZEN status
4. Assert CI is green on current state
5. Create snapshot: git tag migration/phase-2.1D.1/start
```

### Phase 1 — Kernel Integration
```
Source: platform_kernel/
Target: platform/kernel/

Steps:
1.1  Copy platform_kernel/di/         → platform/kernel/di/
1.2  Copy platform_kernel/events/     → platform/kernel/events/
1.3  Copy platform_kernel/config/     → platform/kernel/config/
1.4  Copy platform_kernel/lifecycle/  → platform/kernel/lifecycle/
1.5  Copy platform_kernel/registry/   → platform/kernel/registry/
1.6  Run import rewrite on all copied files (rule DR-007)
1.7  Run verification check: import platform.kernel.di
1.8  Run unit tests: platform/tests/unit/test_kernel_*
1.9  If any check fails → trigger rollback to snapshot
```

### Phase 2 — SDK Integration
```
Source: platform_sdk/
Target: platform/sdk/

Steps:
2.1  Copy platform_sdk/ai.py         → platform/sdk/ai.py
2.2  Copy platform_sdk/providers.py  → platform/sdk/providers.py
2.3  Copy platform_sdk/knowledge.py  → platform/sdk/knowledge.py
2.4  Copy platform_sdk/workflows.py  → platform/sdk/workflows.py
2.5  Copy platform_sdk/resources.py  → platform/sdk/resources.py
2.6  Copy platform_sdk/events.py     → platform/sdk/events.py
2.7  Copy platform_sdk/types.py      → platform/sdk/types.py
2.8  Copy platform_sdk/exceptions.py → platform/sdk/exceptions.py
2.9  Run import rewrite (rule DR-007)
2.10 Verify: from platform.sdk.ai import optimize_routing
2.11 Run SDK tests
2.12 If any fail → rollback
```

### Phase 3 — AI Runtime Path Normalization
```
Source: platform/ai_runtime/
Target: platform/runtimes/ai_runtime/

Steps:
3.1  Create platform/runtimes/ if not exists
3.2  Move platform/ai_runtime/ → platform/runtimes/ai_runtime/
3.3  Run import rewrite: "from platform.ai_runtime" → "from platform.runtimes.ai_runtime"
3.4  Update pyproject.toml package includes
3.5  Run full test suite: 160+ tests
3.6  If any fail → rollback
```

### Phase 4 — Source Cleanup
```
Steps:
4.1  Delete platform_kernel/ (source confirmed dead via verification)
4.2  Delete platform_sdk/ (source confirmed dead via verification)
4.3  Update all product imports in source/*/
4.4  Run full integration test suite
4.5  git tag migration/phase-2.1D.1/complete
```

---

## Migration Engine Interface

```python
class MigrationEngine:
    def __init__(self, config: MigrationConfig) -> None: ...

    def check_preconditions(self) -> list[str]: ...
    # Returns list of failed preconditions; empty = ready to migrate

    def plan(self) -> MigrationPlan: ...
    # Returns complete plan without executing

    def execute_phase(self, phase: int) -> MigrationResult: ...
    # Executes one phase; triggers rollback on failure

    def rollback(self, to_snapshot: str) -> None: ...
    # Hard rollback to git snapshot

    def status(self) -> MigrationStatus: ...
    # Current state machine position
```

---

## Migration Config

```python
class MigrationConfig(BaseModel):
    dry_run: bool = True              # ALWAYS start with dry_run=True
    snapshot_tag_prefix: str = "migration/phase-2.1D.1"
    verify_after_each_phase: bool = True
    rollback_on_first_failure: bool = True
    parallel_file_ops: bool = False   # sequential for safety
    log_path: str = "reports/migration/"
```

---

## Rollback Triggers

Automatic rollback is triggered if ANY of the following occur during a phase:

| Trigger | Check |
|---------|-------|
| Import fails | `python -c "import platform.kernel.di"` exits non-zero |
| Test failure | pytest exits non-zero |
| File count mismatch | Source files != destination files after copy |
| Checksum mismatch | Copied file SHA256 != original |
| Circular import detected | `platform-verify imports --circular` exits non-zero |
| Layer violation detected | `platform-verify layers --strict` exits non-zero |

See 24-ROLLBACK-ENGINE.md for rollback procedures.

---

## Migration Log Format

```
[2026-06-29T12:00:00Z] MIGRATION phase=1 step=1.1 action=copy status=OK
  src=platform_kernel/di/ dst=platform/kernel/di/ files=4 duration_ms=23

[2026-06-29T12:00:01Z] MIGRATION phase=1 step=1.7 action=verify status=FAIL
  check="import platform.kernel.di" error="ModuleNotFoundError: No module named 'platform.kernel'"
  → triggering rollback

[2026-06-29T12:00:02Z] ROLLBACK snapshot=migration/phase-2.1D.1/start status=OK
  restored=3 files deleted=0 duration_ms=512
```
