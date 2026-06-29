---
knowledge_id: KNW-PLAT-ARCH-029
title: "Platform CLI Architecture"
status: AUTHORITATIVE
phase: "2.1D.0"
version: "1.0.0"
created: "2026-06-29"
purpose: "Specify all platform CLI commands, subcommands, flags, and output formats"
canonical_source: "architecture/docs/platform-consolidation/29-CLI.md"
dependencies:
  - "25-VERIFICATION.md"
  - "22-MIGRATION-ENGINE.md"
  - "14-PLATFORM-MANIFEST.md"
related_documents:
  - "30-API.md"
  - "28-DASHBOARD.md"
acceptance_criteria:
  - "Every CLI command has a name, description, and flags"
  - "Exit codes are documented for all commands"
  - "Output format is machine-parseable (JSON flag available)"
verification_checklist:
  - "[ ] All commands documented"
  - "[ ] JSON output mode specified"
  - "[ ] Help text defined"
future_extensions:
  - "Shell completion (bash, zsh, fish)"
  - "Config file for default flag values"
---

# Platform CLI Architecture

## CLI Entry Point

```
platform-verify [command] [subcommand] [flags]
```

All commands share these global flags:

| Flag | Default | Description |
|------|---------|-------------|
| `--json` | false | Output as JSON instead of human-readable |
| `--quiet` | false | Suppress non-error output |
| `--verbose` | false | Show debug information |
| `--config` | `.env` | Path to platform config file |

---

## Command: `platform-verify`

Top-level verification runner.

```bash
platform-verify [domain] [flags]
```

Subcommands: `imports`, `layers`, `metadata`, `manifest`, `tests`, `architecture`, `runtime`, `performance`, `all`

---

## Command: `platform-verify imports`

```bash
platform-verify imports [--strict] [--fix-dry-run] [--path PATH]
```

| Flag | Description |
|------|-------------|
| `--strict` | Exit 1 on any WARNING (not just ERROR) |
| `--fix-dry-run` | Show what would be fixed without changing files |
| `--path PATH` | Limit scan to specific path |

**Output:**
```
✓ IMPORT-001  No platform_kernel imports (0 violations)
✓ IMPORT-002  No platform_sdk imports (0 violations)
✗ IMPORT-005  Relative depth > 3 found in 2 files (WARNING)
  platform/runtimes/ai_runtime/resource_intelligence/workload_profiler/profiler.py:12
```

**Exit codes:** `0` = pass, `1` = violations found

---

## Command: `platform-verify layers`

```bash
platform-verify layers [--strict] [--visualize]
```

| Flag | Description |
|------|-------------|
| `--strict` | Treat warnings as errors |
| `--visualize` | Print layer dependency graph |

---

## Command: `platform-verify metadata`

```bash
platform-verify metadata [--strict] [--schema-path PATH]
```

Validates all `module.yaml`, `runtime.yaml`, `service.yaml`, `capability.yaml` files.

---

## Command: `platform-verify manifest`

```bash
platform-verify manifest [--generate] [--validate] [--diff OLD NEW] [--summary]
```

| Flag | Description |
|------|-------------|
| `--generate` | Generate `platform/MANIFEST.yaml` from current state |
| `--validate` | Validate existing manifest (checksum + counts) |
| `--diff OLD NEW` | Compare two manifest files |
| `--summary` | Print human-readable summary |

**Generate output:**
```
Generated: platform/MANIFEST.yaml
  modules:      78
  capabilities: 66
  services:      6
  runtimes:      7
  checksum:     sha256:abc123...
```

---

## Command: `platform-verify architecture`

```bash
platform-verify architecture [--docs] [--freeze] [--count]
```

| Flag | Description |
|------|-------------|
| `--docs` | Verify all 37 documents exist with valid front-matter |
| `--freeze` | Verify Architecture Freeze document is FROZEN |
| `--count` | Print document counts by status |

---

## Command: `platform-verify all`

```bash
platform-verify all [--strict] [--summary]
```

Runs all verification domains in sequence. Stops at first ERROR-level failure unless `--continue-on-error` is set.

---

## Command: `platform-migrate`

Migration engine CLI. See 22-MIGRATION-ENGINE.md for phase descriptions.

```bash
platform-migrate [subcommand] [flags]
```

### `platform-migrate check`
```bash
platform-migrate check
# Runs all precondition checks before migration
# Exit 0 = ready to migrate
# Exit 1 = preconditions not met (lists what's missing)
```

### `platform-migrate plan`
```bash
platform-migrate plan [--output-format text|json]
# Prints the migration plan without executing
# Shows all phases, steps, and estimated file changes
```

### `platform-migrate run`
```bash
platform-migrate run [--dry-run] [--phase N] [--stop-at-phase N]
# Executes migration
# --dry-run: prints all steps without executing
# --phase N: run only phase N
# --stop-at-phase N: stop before phase N+1
```

### `platform-migrate status`
```bash
platform-migrate status
# Current state machine position
# Last completed step
# Snapshot tag
```

### `platform-migrate rollback`
```bash
platform-migrate rollback [--snapshot TAG]
# Manual rollback to snapshot
# Uses latest migration snapshot by default
```

---

## Command: `platform-generate`

Code generation tools.

```bash
platform-generate [subcommand]
```

### `platform-generate manifest`
Same as `platform-verify manifest --generate`.

### `platform-generate module-yaml PATH`
```bash
platform-generate module-yaml platform/runtimes/ai_runtime/new_module/
# Generates a starter module.yaml in the given directory
```

### `platform-generate capability-yaml`
```bash
platform-generate capability-yaml --id "capability:ai:new:v1" --domain ai --name new
# Generates capability.yaml
```

---

## JSON Output Examples

```bash
platform-verify imports --json
```
```json
{
  "domain": "imports",
  "checks": [
    {"id": "IMPORT-001", "status": "PASS", "violations": 0},
    {"id": "IMPORT-002", "status": "PASS", "violations": 0},
    {"id": "IMPORT-005", "status": "WARN", "violations": 2, "files": [
      "platform/runtimes/ai_runtime/resource_intelligence/workload_profiler/profiler.py"
    ]}
  ],
  "summary": {"pass": 7, "warn": 1, "fail": 0},
  "exit_code": 0
}
```

---

## CLI Implementation Location

```
platform/tools/cli/
  __init__.py
  main.py          # entry point: platform-verify, platform-migrate, platform-generate
  verify/
    imports.py
    layers.py
    metadata.py
    manifest.py
    architecture.py
    runtime.py
    tests.py
    performance.py
  migrate/
    check.py
    plan.py
    run.py
    status.py
    rollback.py
  generate/
    manifest.py
    module_yaml.py
    capability_yaml.py
```

Registered in `pyproject.toml`:
```toml
[project.scripts]
platform-verify  = "platform.tools.cli.main:verify"
platform-migrate = "platform.tools.cli.main:migrate"
platform-generate = "platform.tools.cli.main:generate"
```
