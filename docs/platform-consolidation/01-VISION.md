---
knowledge_id: KNW-PLAT-ARCH-001
title: "Platform Consolidation Vision"
status: AUTHORITATIVE
phase: "2.1D.0"
version: "1.0.0"
created: "2026-06-29"
purpose: "Define WHY we consolidate and WHAT the end state looks like"
canonical_source: "architecture/docs/platform-consolidation/01-VISION.md"
dependencies: []
related_documents:
  - "00-README.md"
  - "03-DIRECTORY-ARCHITECTURE.md"
  - "34-ARCHITECTURE-DECISIONS.md"
  - "35-ARCHITECTURE-FREEZE.md"
acceptance_criteria:
  - "Vision is unambiguous — no two engineers can interpret it differently"
  - "End state is fully described with zero implementation details"
  - "Problem statement references actual current repositories"
  - "Success criteria are measurable"
verification_checklist:
  - "[ ] Problem statement lists real current repos"
  - "[ ] End state directory matches 03-DIRECTORY-ARCHITECTURE.md"
  - "[ ] Every principle has a rationale"
  - "[ ] Success metrics are quantitative"
future_extensions:
  - "Expand to multi-language platform vision (Rust kernel, Python SDK)"
---

# Platform Consolidation Vision

## Problem Statement

The AI Studio platform is currently fragmented across multiple repositories with overlapping
responsibilities, no clear ownership boundaries, and no canonical dependency direction.

### Current Fragmentation (Observed State)

| Repository | Location | Problem |
|------------|----------|---------|
| `platform/` | `AIStudio/platform/` | Contains ai_runtime but lacks kernel/SDK separation |
| `platform_kernel/` | `AIStudio/platform_kernel/` | Kernel exists separately — products import directly |
| `platform_sdk/` | `AIStudio/platform_sdk/` | SDK exists separately — no unified surface |

**Symptoms of fragmentation:**
- Products import from `platform_kernel/` and `platform/` interchangeably
- No single pyproject.toml governs the platform
- Kernel and runtime have no enforced boundary
- SDK has no stable public API contract
- Adding a new runtime requires touching three repositories

---

## End State Vision

One repository. One import surface. One governance boundary.

```
AIStudio/
└── platform/                    ← THE canonical Platform
    ├── kernel/                  ← PlatformKernel: DI, events, lifecycle, config
    ├── sdk/                     ← PlatformSDK: public API for product developers
    ├── runtimes/
    │   ├── ai_runtime/          ← AI execution, routing, resource intelligence
    │   ├── knowledge_runtime/   ← Knowledge graph, compiler, search
    │   ├── provider_runtime/    ← Provider plugin management
    │   ├── workflow_runtime/    ← Task graphs, orchestration primitives
    │   ├── orchestration_runtime/ ← Multi-agent coordination
    │   ├── event_runtime/       ← Event bus, pub/sub, streaming
    │   └── resource_runtime/   ← Resource scheduling, quota, budgets
    ├── services/                ← Cross-cutting services (auth, audit, metrics)
    ├── api/                     ← FastAPI routers, OpenAPI schemas
    ├── desktop/                 ← PySide6 platform panels (not products)
    ├── security/                ← Encryption, secrets, RBAC
    ├── common/                  ← Shared types, exceptions, utilities
    ├── tools/                   ← Migration, verification, CLI tools
    ├── docs/                    ← Architecture documents (this package)
    └── tests/                   ← All platform tests
```

**Products** live outside `platform/` and depend on it only through `platform.sdk.*`.

---

## Core Principles

### P1 — Single Source of Truth
There is exactly one `platform/` directory. All platform code lives there. No exceptions.

**Rationale:** Eliminates import ambiguity and circular dependency risk.

### P2 — Strict Dependency Direction
```
Products → SDK → Services → Runtimes → Kernel → Common
```
No upward dependencies. No lateral dependencies between runtimes except via kernel events.

**Rationale:** Enforces modularity; any runtime can be replaced without touching others.

### P3 — Public API via SDK Only
Products never import from `platform.runtimes.*` or `platform.kernel.*` directly.
They import from `platform.sdk.*` only.

**Rationale:** SDK provides stability guarantees; internals can be refactored freely.

### P4 — Kernel is Zero-Dependency
`platform.kernel` has NO runtime dependencies. It imports only Python stdlib and `platform.common`.

**Rationale:** Kernel must be importable in any context, including tools and tests.

### P5 — Runtime Isolation
Runtimes do NOT import from each other directly.
Cross-runtime communication happens exclusively via `platform.kernel.events`.

**Rationale:** Enables independent deployment, testing, and replacement of runtimes.

### P6 — Plugin Compatibility
All provider references are string IDs. No runtime contains hard-coded provider logic.
Every external integration is a plugin.

**Rationale:** Consumers (products) choose and configure plugins; platform never dictates.

### P7 — Architecture Before Implementation
No file is moved, no import is rewritten, and no runtime code is changed until
35-ARCHITECTURE-FREEZE.md is signed.

**Rationale:** Avoids mid-migration architectural surprises that require full rollback.

---

## What Changes (vs. Current State)

| Aspect | Current | Target |
|--------|---------|--------|
| Repositories | 3+ (`platform/`, `platform_kernel/`, `platform_sdk/`) | 1 (`platform/`) |
| Import surface | Inconsistent | `platform.sdk.*` only |
| Kernel location | `platform_kernel/` | `platform/kernel/` |
| SDK location | `platform_sdk/` | `platform/sdk/` |
| Runtimes | `platform/ai_runtime/` only structured | 7 runtimes, all canonical |
| Pyproject | Multiple, inconsistent | One `platform/pyproject.toml` |
| Tests | Split across repos | All in `platform/tests/` |
| Governance | None | 33-PLATFORM-GOVERNANCE.md |

---

## What Does NOT Change

- Product codebases are not touched during Platform Consolidation
- Existing `platform/ai_runtime/` code is not refactored (only re-homed)
- External API contracts are not changed
- Database schemas are not changed
- Provider plugin interface is not changed

---

## Success Criteria

| # | Criterion | Measurement |
|---|-----------|-------------|
| S1 | One platform directory | `find AIStudio -name "platform_kernel" -o -name "platform_sdk"` returns empty |
| S2 | Zero cross-repo platform imports in products | Import scanner finds 0 violations |
| S3 | All platform tests pass | `pytest platform/tests/` exit 0 |
| S4 | Single pyproject.toml | `find platform -name pyproject.toml | wc -l` == 1 |
| S5 | Dependency direction enforced | Layer linter exit 0 |
| S6 | Knowledge objects registered | All platform modules in Knowledge Registry |
| S7 | Architecture score | Verification engine reports ≥ 95/100 |
