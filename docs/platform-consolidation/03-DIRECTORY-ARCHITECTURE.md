---
knowledge_id: KNW-PLAT-ARCH-003
title: "Canonical Directory Architecture"
status: AUTHORITATIVE
phase: "2.1D.0"
version: "1.0.0"
created: "2026-06-29"
purpose: "Define the exact canonical directory layout, naming rules, ownership, and visibility"
canonical_source: "architecture/docs/platform-consolidation/03-DIRECTORY-ARCHITECTURE.md"
dependencies:
  - "01-VISION.md"
  - "02-OBJECT-MODEL.md"
related_documents:
  - "04-PACKAGE-MODEL.md"
  - "10-DEPENDENCY-RULES.md"
  - "12-IMPORT-RULES.md"
acceptance_criteria:
  - "Every directory has an owner, visibility level, and naming rule"
  - "No ambiguous paths exist"
  - "Forbidden layouts are explicitly listed"
  - "Counterexamples provided for every rule"
verification_checklist:
  - "[ ] Target layout matches 01-VISION.md end state"
  - "[ ] All naming rules are regex-expressible"
  - "[ ] No current file violates a naming rule that would block migration"
  - "[ ] Forbidden layouts cover all known anti-patterns"
future_extensions:
  - "Monorepo tooling (Nx/Turborepo) workspace.json generation"
  - "Automatic directory template scaffolding CLI"
---

# Canonical Directory Architecture

## Full Target Layout

```
AIStudio/
└── platform/                          [ROOT — canonical platform repository]
    │
    ├── pyproject.toml                 [SINGLE package definition]
    ├── README.md
    ├── MANIFEST.md                    [generated from 14-PLATFORM-MANIFEST.md]
    │
    ├── kernel/                        [PlatformKernel — boot priority 0]
    │   ├── __init__.py
    │   ├── bootstrap.py               [startup sequence]
    │   ├── config/                    [configuration management]
    │   ├── di/                        [dependency injection container]
    │   ├── events/                    [event bus and dispatcher]
    │   ├── lifecycle/                 [lifecycle manager]
    │   ├── registry/                  [capability, service, module registries]
    │   └── exceptions/                [kernel-level exceptions]
    │
    ├── sdk/                           [PlatformSDK — public API surface]
    │   ├── __init__.py
    │   ├── ai.py                      [AI runtime facade]
    │   ├── knowledge.py               [Knowledge runtime facade]
    │   ├── providers.py               [Provider management facade]
    │   ├── workflows.py               [Workflow runtime facade]
    │   ├── resources.py               [Resource management facade]
    │   ├── events.py                  [Event publishing facade]
    │   ├── types.py                   [Shared SDK types]
    │   └── exceptions.py              [SDK-level exceptions]
    │
    ├── runtimes/
    │   ├── __init__.py
    │   ├── ai_runtime/                [AI execution, routing, resource intelligence]
    │   │   ├── __init__.py
    │   │   ├── resource_intelligence/ [Phase 2.1A/2.1B modules — already implemented]
    │   │   ├── routing/
    │   │   ├── execution/
    │   │   └── models.py
    │   │
    │   ├── knowledge_runtime/         [Knowledge graph, compiler, search]
    │   │   ├── __init__.py
    │   │   ├── graph/
    │   │   ├── compiler/
    │   │   ├── search/
    │   │   └── objects/
    │   │
    │   ├── provider_runtime/          [Provider plugin management]
    │   │   ├── __init__.py
    │   │   ├── registry/
    │   │   ├── adapters/
    │   │   └── health/
    │   │
    │   ├── workflow_runtime/          [Task graphs, orchestration primitives]
    │   │   ├── __init__.py
    │   │   ├── graph/
    │   │   ├── executor/
    │   │   └── scheduler/
    │   │
    │   ├── orchestration_runtime/     [Multi-agent coordination]
    │   │   ├── __init__.py
    │   │   ├── coordinator/
    │   │   ├── agents/
    │   │   └── sessions/
    │   │
    │   ├── event_runtime/             [Event bus, pub/sub, streaming]
    │   │   ├── __init__.py
    │   │   ├── bus/
    │   │   ├── pubsub/
    │   │   └── streaming/
    │   │
    │   └── resource_runtime/          [Resource scheduling, quota, budgets]
    │       ├── __init__.py
    │       ├── quota/
    │       ├── budget/
    │       └── scheduler/
    │
    ├── services/                      [Cross-cutting platform services]
    │   ├── __init__.py
    │   ├── auth/                      [Authentication and authorization]
    │   ├── audit/                     [Audit logging]
    │   ├── metrics/                   [Prometheus metrics]
    │   ├── tracing/                   [Distributed tracing]
    │   ├── cache/                     [Caching service]
    │   └── notifications/             [Alert and notification service]
    │
    ├── api/                           [FastAPI routers and OpenAPI schemas]
    │   ├── __init__.py
    │   ├── app.py                     [FastAPI application factory]
    │   ├── middleware/
    │   ├── routes/
    │   │   ├── v1/
    │   │   └── v2/
    │   └── schemas/                   [Pydantic request/response models]
    │
    ├── desktop/                       [PySide6 platform UI panels]
    │   ├── __init__.py
    │   ├── panels/
    │   └── widgets/
    │
    ├── security/                      [Encryption, secrets, RBAC]
    │   ├── __init__.py
    │   ├── encryption/
    │   ├── secrets/
    │   └── rbac/
    │
    ├── common/                        [Shared types, exceptions, utilities]
    │   ├── __init__.py
    │   ├── types.py
    │   ├── exceptions.py
    │   ├── utils.py
    │   └── constants.py
    │
    ├── tools/                         [Migration, verification, dev CLI tools]
    │   ├── __init__.py
    │   ├── migrate/
    │   ├── verify/
    │   ├── scaffold/
    │   └── lint/
    │
    ├── docs/                          [Architecture documents]
    │   └── platform-consolidation/    [This package]
    │
    └── tests/                         [All platform tests]
        ├── unit/
        ├── integration/
        ├── simulation/
        └── stress/
```

---

## Naming Rules

### Directory Names
| Rule | Pattern | Example | Counter-example |
|------|---------|---------|-----------------|
| All lowercase | `^[a-z][a-z0-9_]*$` | `ai_runtime` | `AIRuntime`, `ai-runtime` |
| No hyphens | no `-` | `event_runtime` | `event-runtime` |
| No version suffixes | no `_v[0-9]` suffix | `routing` | `routing_v2` |
| Runtime suffix required | `*_runtime` | `ai_runtime` | `ai_engine`, `ai` |
| Module dirs: noun | descriptive noun | `routing`, `quota` | `do_routing` |

### File Names
| Rule | Pattern | Example | Counter-example |
|------|---------|---------|-----------------|
| Python modules: snake_case | `^[a-z][a-z0-9_]*\.py$` | `resource_manager.py` | `ResourceManager.py` |
| Models file | `models.py` | `models.py` | `model.py`, `schemas.py` |
| Entry point | `__init__.py` | `__init__.py` | `init.py` |
| Test prefix | `test_*.py` | `test_quota.py` | `quota_test.py` |

### Special Files
| File | Location | Purpose | Required |
|------|----------|---------|---------|
| `pyproject.toml` | `platform/` root only | Single package definition | YES |
| `module.yaml` | each module dir | Module metadata | YES |
| `runtime.yaml` | each runtime dir | Runtime metadata | YES |
| `service.yaml` | each service dir | Service metadata | YES |
| `__init__.py` | every Python package dir | Package marker | YES |

---

## Ownership

| Directory | Owner | Write Access |
|-----------|-------|-------------|
| `kernel/` | Kernel Team | Kernel Team only |
| `sdk/` | SDK Team | SDK Team + Architecture review |
| `runtimes/*/` | Runtime Team (per runtime) | Runtime Team only |
| `services/` | Services Team | Services Team |
| `api/` | API Team | API Team |
| `common/` | Platform Team | Any Team + Architecture review |
| `tools/` | Platform Team | Any Team |
| `tests/` | Any Team | Any Team for their scope |

---

## Visibility Levels

| Level | Meaning | Example |
|-------|---------|---------|
| PUBLIC | Accessible via `platform.sdk.*` | `sdk/ai.py` |
| INTERNAL | Accessible within same runtime | `ai_runtime/routing/` |
| PRIVATE | Within same module only | `ai_runtime/routing/_internal.py` |
| EXPERIMENTAL | May change without notice | `runtimes/*/experimental/` |
| GENERATED | Do not edit manually | `runtimes/*/generated/` |

### Visibility Markers
- Public modules: exported in `sdk/__init__.py`
- Internal modules: NOT in `sdk/`; accessible only within runtime
- Private: prefix with `_` (e.g., `_internal.py`)
- Experimental: in `experimental/` subdirectory
- Generated: in `generated/` subdirectory with `.generated` suffix in first line

---

## Forbidden Layouts

| Anti-pattern | Why Forbidden | Correct Alternative |
|-------------|---------------|---------------------|
| `platform/platform_kernel/` | Nested namespace pollution | `platform/kernel/` |
| `platform/runtimes/ai/` | Missing `_runtime` suffix | `platform/runtimes/ai_runtime/` |
| `platform/v2/` | Version in path | Version in `pyproject.toml` |
| `platform/core/` | Ambiguous name | `platform/kernel/` |
| `runtimes/ai_runtime/sdk/` | SDK inside runtime | `platform/sdk/ai.py` |
| `runtimes/*/kernel/` | Kernel inside runtime | `platform/kernel/` |
| Multiple `pyproject.toml` | Fragmented packaging | One at `platform/` root |
| `runtimes/ai_runtime/tests/` | Tests inside runtime | `platform/tests/unit/` |

---

## Examples vs. Counter-examples

### CORRECT
```python
# Product imports only from SDK
from platform.sdk.ai import optimize_routing, classify_workload
from platform.sdk.resources import check_quota
```

### INCORRECT
```python
# Product imports directly from runtime — FORBIDDEN
from platform.runtimes.ai_runtime.routing_optimizer.optimizer import RoutingOptimizer
from platform_kernel.di import Container  # Old path — FORBIDDEN after consolidation
```

### CORRECT runtime-to-runtime communication
```python
# ai_runtime signals knowledge_runtime via event bus
await kernel.events.publish(AIExecutionCompleted(result=result))
# knowledge_runtime subscribes — no direct import
```

### INCORRECT runtime-to-runtime
```python
# Direct import between runtimes — FORBIDDEN
from platform.runtimes.knowledge_runtime.graph import KnowledgeGraph
```
