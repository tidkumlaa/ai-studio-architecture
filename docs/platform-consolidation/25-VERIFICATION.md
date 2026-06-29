---
knowledge_id: KNW-PLAT-ARCH-025
title: "Platform Verification Specification"
status: AUTHORITATIVE
phase: "2.1D.0"
version: "1.0.0"
created: "2026-06-29"
purpose: "Define all verification checks for architecture compliance, migration success, and runtime health"
canonical_source: "architecture/docs/platform-consolidation/25-VERIFICATION.md"
dependencies:
  - "10-DEPENDENCY-RULES.md"
  - "11-LAYER-RULES.md"
  - "12-IMPORT-RULES.md"
  - "13-METADATA-STANDARD.md"
  - "14-PLATFORM-MANIFEST.md"
related_documents:
  - "26-TEST-STRATEGY.md"
  - "22-MIGRATION-ENGINE.md"
acceptance_criteria:
  - "Every verification check is machine-executable"
  - "Exit codes are documented"
  - "Checks are grouped by domain and phase"
verification_checklist:
  - "[ ] All check IDs are unique"
  - "[ ] Every check has a remediation hint"
  - "[ ] CI integration documented"
future_extensions:
  - "Verification dashboard in 28-DASHBOARD.md"
  - "Scheduled nightly verification run"
---

# Platform Verification Specification

## Verification Tool

All checks run via:
```bash
platform-verify [domain] [flags]
```

Exit codes:
- `0` — all checks pass
- `1` — one or more checks fail
- `2` — verification tool itself errored

---

## Domain: imports

```bash
platform-verify imports [--strict] [--fix-dry-run]
```

| Check ID | Description | Severity |
|----------|-------------|---------|
| IMPORT-001 | No `from platform_kernel` imports remain | ERROR |
| IMPORT-002 | No `from platform_sdk` imports remain | ERROR |
| IMPORT-003 | No `from platform.ai_runtime` (old path) remains | ERROR |
| IMPORT-004 | No star imports (`from X import *`) in non-SDK files | ERROR |
| IMPORT-005 | No relative depth > 3 | WARNING |
| IMPORT-006 | `from __future__ import annotations` in all platform files | WARNING |
| IMPORT-007 | `__all__` defined in all public SDK modules | WARNING |
| IMPORT-008 | No `importlib.import_module` outside DI container | ERROR |

---

## Domain: layers

```bash
platform-verify layers [--strict]
```

| Check ID | Description | Severity |
|----------|-------------|---------|
| LAYER-001 | COMMON imports only STDLIB | ERROR |
| LAYER-002 | KERNEL imports only COMMON and STDLIB | ERROR |
| LAYER-003 | SERVICES import only KERNEL, COMMON, STDLIB | ERROR |
| LAYER-004 | RUNTIMES import only KERNEL, SERVICES, COMMON, STDLIB | ERROR |
| LAYER-005 | SDK imports only SERVICES, KERNEL, COMMON, STDLIB | ERROR |
| LAYER-006 | PRODUCTS import only SDK (via `platform.sdk.*`) | ERROR |
| LAYER-007 | No runtime imports another runtime | ERROR |
| LAYER-008 | No service imports a runtime | ERROR |
| LAYER-009 | No kernel imports a service | ERROR |
| LAYER-010 | No circular dependencies within a runtime | ERROR |

---

## Domain: metadata

```bash
platform-verify metadata [--strict]
```

| Check ID | Description | Severity |
|----------|-------------|---------|
| META-001 | Every module directory has `module.yaml` | ERROR |
| META-002 | Every runtime directory has `runtime.yaml` | ERROR |
| META-003 | Every service directory has `service.yaml` | ERROR |
| META-004 | All IDs are globally unique | ERROR |
| META-005 | All `version` fields are valid semver | ERROR |
| META-006 | All `python_module` values are importable (or PLANNED) | WARNING |
| META-007 | All `dependencies` reference existing IDs | ERROR |
| META-008 | Capability IDs claimed in `module.yaml` exist in `capability.yaml` | ERROR |

---

## Domain: manifest

```bash
platform-verify manifest [--validate] [--generate] [--strict]
```

| Check ID | Description | Severity |
|----------|-------------|---------|
| MANIFEST-001 | Manifest file exists at `platform/MANIFEST.yaml` | ERROR |
| MANIFEST-002 | Manifest checksum matches computed checksum | ERROR |
| MANIFEST-003 | Module count in manifest matches actual module count | ERROR |
| MANIFEST-004 | All runtime IDs in manifest match `runtime.yaml` files | ERROR |
| MANIFEST-005 | Manifest `generated_at` is within 7 days (staleness check) | WARNING |

---

## Domain: tests

```bash
platform-verify tests [--unit] [--integration] [--all]
```

| Check ID | Description | Severity |
|----------|-------------|---------|
| TEST-001 | All unit tests pass (pytest exits 0) | ERROR |
| TEST-002 | Test count ≥ 160 (no test deletion) | ERROR |
| TEST-003 | No tests import from old paths (`platform_kernel`, `platform_sdk`) | ERROR |
| TEST-004 | Test coverage ≥ 80% for all STABLE modules | WARNING |

---

## Domain: architecture

```bash
platform-verify architecture [--docs] [--freeze]
```

| Check ID | Description | Severity |
|----------|-------------|---------|
| ARCH-001 | All 37 architecture documents exist | ERROR |
| ARCH-002 | All documents have required YAML front-matter | ERROR |
| ARCH-003 | All Knowledge IDs follow `KNW-PLAT-ARCH-NNN` format | ERROR |
| ARCH-004 | No document exceeds 400 lines | WARNING |
| ARCH-005 | `35-ARCHITECTURE-FREEZE.md` status is FROZEN | ERROR (migration gate) |
| ARCH-006 | `index.yaml` references all 37 documents | ERROR |

---

## Domain: runtime

```bash
platform-verify runtime [--health] [--boot]
```

| Check ID | Description | Severity |
|----------|-------------|---------|
| RUNTIME-001 | All 5 existing runtimes start successfully | ERROR |
| RUNTIME-002 | Boot order is respected (event→resource→...→orchestration) | ERROR |
| RUNTIME-003 | No runtime emits events during INITIALIZING state | WARNING |
| RUNTIME-004 | All kernel subsystems register before any runtime boots | ERROR |

---

## CI Integration

```yaml
# Verification pipeline — runs on every PR and main branch push
jobs:
  verify:
    steps:
      - name: Import checks
        run: platform-verify imports --strict

      - name: Layer checks
        run: platform-verify layers --strict

      - name: Metadata checks
        run: platform-verify metadata --strict

      - name: Manifest check
        run: platform-verify manifest --validate

      - name: Architecture docs
        run: platform-verify architecture --docs

      - name: Tests
        run: platform-verify tests --all
```

---

## Verification Summary Output

```
platform-verify --all --summary

╔══════════════════════════════════════════════════════╗
║           Platform Verification Report               ║
╠══════════════════════════════════════════════════════╣
║ imports    ██████████  8/8  PASS                     ║
║ layers     ██████████ 10/10 PASS                     ║
║ metadata   █████████░  8/8  PASS                     ║
║ manifest   ██████████  5/5  PASS                     ║
║ tests      ████████░░  3/4  WARN (coverage 79%)      ║
║ arch docs  ██████████  6/6  PASS                     ║
╠══════════════════════════════════════════════════════╣
║ Total: 40 PASS  1 WARN  0 FAIL                       ║
╚══════════════════════════════════════════════════════╝
```
