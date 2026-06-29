# Platform Consolidation Architecture

**Phase:** 2.1D.0  
**Status:** COMPLETE — Architecture Frozen  
**Documents:** 37  
**Knowledge IDs:** KNW-PLAT-ARCH-000 through KNW-PLAT-ARCH-035

---

## What This Is

Complete architecture specification for consolidating the AI Studio platform from three
fragmented repositories (`platform/`, `platform_kernel/`, `platform_sdk/`) into a single
canonical `platform/` repository.

**This directory contains ONLY architecture documents. No runtime code.**

---

## Quick Navigation

| Need | Document |
|------|---------|
| Start here | [00-README.md](00-README.md) |
| Why we're doing this | [01-VISION.md](01-VISION.md) |
| Where files go | [03-DIRECTORY-ARCHITECTURE.md](03-DIRECTORY-ARCHITECTURE.md) |
| What moves where | [22-MIGRATION-ENGINE.md](22-MIGRATION-ENGINE.md) |
| Import rules | [12-IMPORT-RULES.md](12-IMPORT-RULES.md) |
| All capabilities | [15-CAPABILITY-REGISTRY.md](15-CAPABILITY-REGISTRY.md) |
| Architecture decisions | [34-ARCHITECTURE-DECISIONS.md](34-ARCHITECTURE-DECISIONS.md) |
| Freeze & gate conditions | [35-ARCHITECTURE-FREEZE.md](35-ARCHITECTURE-FREEZE.md) |

Machine-readable index: [index.yaml](index.yaml)

---

## Document Map

```
Tier 0 — Navigation
  00-README.md

Tier 1 — Foundation
  01-VISION.md
  02-OBJECT-MODEL.md
  03-DIRECTORY-ARCHITECTURE.md
  04-PACKAGE-MODEL.md

Tier 2 — Architecture
  05-MODULE-CATALOG.md
  06-RUNTIME-MODEL.md
  07-KERNEL-MODEL.md
  08-SDK-MODEL.md
  09-SERVICE-MODEL.md

Tier 3 — Rules
  10-DEPENDENCY-RULES.md
  11-LAYER-RULES.md
  12-IMPORT-RULES.md
  13-METADATA-STANDARD.md
  14-PLATFORM-MANIFEST.md

Tier 4 — Registries & Models
  15-CAPABILITY-REGISTRY.md
  16-SERVICE-REGISTRY.md
  17-MODULE-REGISTRY.md
  18-EVENT-MODEL.md
  19-STATE-MACHINES.md
  20-DATA-STRUCTURES.md
  21-ALGORITHMS.md

Tier 5 — Migration
  22-MIGRATION-ENGINE.md
  23-IMPORT-REWRITE.md
  24-ROLLBACK-ENGINE.md
  25-VERIFICATION.md
  26-TEST-STRATEGY.md
  27-PERFORMANCE-BUDGET.md

Tier 6 — Interfaces
  28-DASHBOARD.md
  29-CLI.md
  30-API.md
  31-KNOWLEDGE-INTEGRATION.md
  32-RUNTIME-INTEGRATION.md

Tier 7 — Governance
  33-PLATFORM-GOVERNANCE.md
  34-ARCHITECTURE-DECISIONS.md
  35-ARCHITECTURE-FREEZE.md
```

---

## Key Numbers

| Metric | Value |
|--------|-------|
| Runtimes | 7 (event, resource, provider, knowledge, ai, workflow, orchestration) |
| Services | 6 (auth, audit, metrics, tracing, cache, notifications) |
| Capabilities | 66 (57 stable, 3 beta, 6 planned) |
| Modules | 58 primary (50 existing, 8 planned) |
| Dependency rules | DR-001 through DR-010 |
| Layer rules | LR-001 through LR-011 |
| Import rules | IR-001 through IR-008 |
| ADRs | 10 (ADR-001 through ADR-010) |
| Test count | ≥ 160 |
| Architecture documents | 37 |

---

## What Happens Next

This architecture is the input to **Phase 2.1D.1 — Platform Consolidation Implementation**.

Before implementation begins, verify all gate conditions:

```bash
platform-verify architecture --docs    # all 37 docs exist
platform-verify architecture --freeze  # freeze doc is FROZEN
platform-verify imports --strict       # no old import paths
platform-verify layers --strict        # no layer violations
pytest platform/tests/ -x              # all tests pass
```

See [35-ARCHITECTURE-FREEZE.md](35-ARCHITECTURE-FREEZE.md) for the complete gate checklist.

---

## Critical Constraint

```
╔══════════════════════════════════════════╗
║  DO NOT implement during Phase 2.1D.0   ║
║  DO NOT move files                       ║
║  DO NOT modify imports                   ║
║  DO NOT modify products                  ║
║  Architecture specification ONLY         ║
╚══════════════════════════════════════════╝
```
