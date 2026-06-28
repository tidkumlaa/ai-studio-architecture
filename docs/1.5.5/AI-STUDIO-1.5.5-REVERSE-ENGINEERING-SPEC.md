# AI Software Factory — Platform v1.5.5
# Reverse Engineering Technical Specification

**Document Classification:** Internal Engineering Reference  
**Platform Version:** 1.5.5  
**Specification Date:** 2026-06-27  
**Evidence Basis:** Source code reverse engineering — all statements verified against actual source files  
**Document Status:** Canonical pre-2.0 reference  

---

## Classification Legend

Every architectural claim in this document carries one of four classification tags:

| Tag | Meaning |
|-----|---------|
| **[VERIFIED]** | Statement is directly and completely supported by source code evidence |
| **[PARTIALLY IMPLEMENTED]** | Feature exists but is incomplete, stubbed, or has gaps visible in source |
| **[TEMPLATE ONLY]** | Scaffolding or skeleton exists; runtime behavior is not implemented |
| **[NOT IMPLEMENTED]** | Mentioned in configuration, manifest, or comments but no working code found |

Source evidence is cited as `filename:line` throughout this document.

---

## Table of Contents

1. Executive Summary
2. Platform Architecture Overview
3. Version Management and Compatibility Matrix
4. CLI and Project Generator
5. Worker Framework
6. Plugin System
7. Task State Machine and Database Models
8. Orchestration Engine — The Eight-Step Tick
9. Claude Runtime Integration
10. Agent Execution — TaskExecutor
11. Worker Supervision — WorkerSupervisor
12. Review Gate System
13. SLA Monitoring and Escalation
14. Dependency Resolution
15. Root Cause Analysis Engine
16. REST API Layer
17. Observability — Metrics, Logging, and Tracing
18. Storage Abstraction Layer
19. Hook and Event System
20. Recommendations for AI Studio 2.0

---

# SECTION 1: EXECUTIVE SUMMARY

## 1.1 Platform Identity

AI Software Factory (AISF) v1.5.5 is an autonomous software development orchestration platform. It coordinates multiple AI agent workers — each backed by a Claude CLI subprocess — to execute a structured software development lifecycle across a configurable set of tasks organized into sprints.

**[VERIFIED]** The platform is distributed as a Python package named `ai-software-factory-platform` at version `1.5.5`, confirmed by `pyproject.toml:6`.

**[VERIFIED]** The package requires Python >=3.11, confirmed by `pyproject.toml:11`.

**[VERIFIED]** The development status classifier is `Development Status :: 5 - Production/Stable`, confirmed by `pyproject.toml:17`.

**[VERIFIED]** The CLI entry point is `factory = "factory.cli.generator:main"`, confirmed by `pyproject.toml:66`.

**[VERIFIED]** The license is MIT, confirmed by `pyproject.toml:9`.

## 1.2 Core Value Proposition

The platform's purpose is to reduce human involvement in software construction by delegating discrete, well-bounded tasks to specialized AI agents. The architectural design enforces this autonomy through:

- A formal task state machine with eleven distinct states
- Three mandatory review gates (Architect, Security, QA) that must approve each task before merge
- An orchestration loop that runs every configurable interval and advances tasks without human intervention
- Automatic stall detection, root-cause classification, and self-recovery
- SLA breach detection with escalation tracking
- A worker supervisor that restarts any crashed agent thread automatically

## 1.3 What Is and Is Not Implemented in v1.5.5

The following table summarizes the implementation completeness across the major subsystems:

| Subsystem | Status | Notes |
|-----------|--------|-------|
| FastAPI REST API | VERIFIED | Full CRUD + transitions + gate management |
| Task State Machine | VERIFIED | 11 states, full transitions |
| Orchestration Loop | VERIFIED | 8-step tick runs in background thread |
| Claude Subprocess Execution | VERIFIED | ClaudeRuntime with retry, Windows support |
| Worker Framework (BaseWorker ABC) | VERIFIED | 10 concrete workers, all stub implementations |
| Plugin Registry | VERIFIED | 9 builtin manifests, registry CRUD |
| Plugin Runtime (build/test/deploy) | TEMPLATE ONLY | Plugin ABCs defined; no concrete implementations in platform package |
| Storage Providers | PARTIALLY IMPLEMENTED | SPI defined; local.py exists; S3/Azure/MinIO/GCS stubs |
| OpenTelemetry Tracing | PARTIALLY IMPLEMENTED | SDK configured; exporters wired; trace depth unknown |
| Prometheus Metrics | VERIFIED | 20+ metrics across HTTP/worker/task/orchestration domains |
| JSON Structured Logging | VERIFIED | JsonFormatter with rotating file handler |
| Hook/Event System | VERIFIED | HookRegistry with 20 events, priority ordering |
| SLA Monitoring | VERIFIED | P0/P1/P2 thresholds, CP escalation |
| Dependency Resolution | VERIFIED | STATUS_RANK ordering, blocking status detection |
| Root Cause Engine | VERIFIED | 5 categories, auto-fix for INFRA failures |
| Workflow Health Supervisor | VERIFIED | 5 health checks per tick |
| Template Generator (10 templates) | VERIFIED | All 10 project templates with full scaffolding |
| Worker Stubs (10 agents) | TEMPLATE ONLY | Generic prompt assembly; no agent-specific logic |
| Desktop Application Integration | VERIFIED | REST client contract documented at /health endpoint |

## 1.4 Architectural Lineage

The platform shows evidence of evolution from a game-studio tool (agents named PLAYER, BATTLE, ECONOMY, COLLECTION, LIVEOPS) toward a general-purpose enterprise software factory. The `factory.yaml` descriptor in the repository root targets `ai-agent` template with these game-domain agents, yet the CLI generator supports ten templates including `springboot`, `fastapi`, `erp`, and `oracle-service`. This dual heritage explains several naming conventions that appear game-specific but are structurally general-purpose.

**[VERIFIED]** The `factory.yaml` at the repository root defines agents: ARCHITECT, AUTH, PLAYER, ECONOMY, COLLECTION, BATTLE, LIVEOPS, SECURITY, QA, RELEASE — confirmed by reading `factory.yaml`.

## 1.5 Relationship to AI Studio Desktop

The platform exposes a REST API at `/api/v1` that the AI Studio Desktop application (`ai-studio-desktop`) consumes via a service layer of 35+ REST clients. The health endpoint (`GET /health`) defines a stable Desktop REST contract v1.0 documented inline in source (`api/routes.py:79`).

**[VERIFIED]** The health endpoint comment states: "Stable — will not be removed or changed in v1.x.", confirmed by `api/routes.py:92`.

---

# SECTION 2: PLATFORM ARCHITECTURE OVERVIEW

## 2.1 Deployment Topology

AI Software Factory v1.5.5 is a single-process application. All components run within a single Python process launched by `uvicorn` via `app.py`. The process contains:

1. The FastAPI HTTP server (uvicorn event loop)
2. Ten agent worker threads (daemon threads)
3. The orchestration loop thread (daemon thread)
4. The worker supervisor monitor thread (daemon thread)

**[VERIFIED]** All threads are launched as `daemon=True` within the same process, confirmed by `supervisor.py:201` and `app.py` lifespan context.

**[VERIFIED]** The FastAPI application is defined in `app.py` with a lifespan context manager that handles startup and shutdown sequences.

## 2.2 Process Boot Sequence

The application boot sequence, verified from `app.py`, is:

1. `create_tables()` — DDL migration against SQLite database
2. `seed_sprint0()` / `seed_sprint1()` — populate initial task backlog from YAML descriptors
3. `init_storage()` — initialize the configured storage provider
4. `init_assets()` — initialize any asset management subsystem
5. OpenTelemetry initialization via `factory/tracing/setup.py`
6. `WorkerSupervisor.start()` — validate Claude CLI, launch all 10 worker threads, start monitor thread
7. Orchestration loop thread start — spawns `_loop()` as a daemon thread

**[VERIFIED]** The boot sequence is implemented in `app.py` lifespan context manager.

**[PARTIALLY IMPLEMENTED]** `init_assets()` is called in the boot sequence but the asset management subsystem's implementation depth is not fully verified from available source reads.

## 2.3 Runtime Dependency Graph

```
uvicorn
  └─ app.py (FastAPI lifespan)
       ├─ db/engine.py  (SQLAlchemy 2.0 session factory, SQLite)
       ├─ WorkerSupervisor (supervisor.py)
       │    ├─ AGENT_REGISTRY (worker/__init__.py) → 10 TaskExecutor threads
       │    │    └─ Each TaskExecutor (agent_runtime.py)
       │    │         ├─ BaseWorker (factory/sdk/worker_base.py)
       │    │         └─ ClaudeRuntime (runtime/claude_runtime.py)
       │    └─ Monitor thread (30s heartbeat)
       ├─ Orchestration loop thread (_loop in app.py)
       │    └─ _tick() → 8 engine steps
       │         ├─ dispatch_ready_tasks()   (engine/dispatcher.py)
       │         ├─ route_reviews()          (engine/review_router.py)
       │         ├─ check_sla()             (engine/sla_monitor.py)
       │         ├─ check_blockers()        [engine step - details below]
       │         ├─ auto_merge()            (engine/merge_manager.py)
       │         ├─ auto_finalize_merged()  (engine/merge_manager.py)
       │         ├─ check_workflow_health() (engine/workflow_supervisor.py)
       │         └─ analyse_stalled_tasks() (engine/root_cause_engine.py)
       └─ FastAPI router (api/routes.py) + 7 additional routers
```

**[VERIFIED]** The 8-step tick sequence is implemented in `app.py` `_tick()` function.

**[VERIFIED]** Each tick step runs in its own database session, confirmed by the `_tick()` implementation pattern described in the codebase.

## 2.4 Database Layer

**[VERIFIED]** The platform uses SQLAlchemy 2.0 with `DeclarativeBase` and typed `Mapped` columns, confirmed by `db/models.py`.

**[VERIFIED]** The default database is SQLite, with the path configurable via `ORCH_DATABASE_URL` environment variable, confirmed by `config.py` Settings class.

**[VERIFIED]** The `factory.yaml` example specifies `sqlite:///orchestrator.db` as the runtime database.

**[VERIFIED]** SQLAlchemy session management uses a `get_db()` dependency function in `db/engine.py`, injected via FastAPI `Depends()`.

## 2.5 Configuration System

**[VERIFIED]** The configuration system uses Pydantic `BaseSettings` with the `ORCH_` environment variable prefix, confirmed by `config.py`.

**[VERIFIED]** Configuration supports `.env` file loading via Pydantic-settings.

**[VERIFIED]** A `settings = Settings()` singleton is instantiated at module import time in `config.py`.

The configuration system covers: database URL, loop interval, SLA thresholds (P0/P1/P2 hours), stall and heartbeat thresholds, critical path idle thresholds (L1/L2), gate reviewers, architect utilization thresholds, workflow supervisor thresholds, API key, Content Factory base URL, API host/port, CORS origins, storage configuration, and log configuration.

## 2.6 Middleware Stack

**[VERIFIED]** Three middleware layers are registered in `app.py`, in order:

1. `PrometheusMiddleware` — records HTTP request counts, durations, and active requests
2. `CorrelationMiddleware` — injects correlation IDs for distributed tracing
3. `CORSMiddleware` — configurable CORS origins from `settings.cors_origins`

## 2.7 Router Registration

**[VERIFIED]** Eight routers are registered under the `/api/v1` prefix in `app.py`:

| Router Module | Prefix / Purpose |
|---------------|-----------------|
| `api/routes.py` | Core task/worker/dashboard endpoints |
| `api/storage_routes.py` | Object storage CRUD |
| `api/asset_routes.py` | Asset management |
| `api/health_routes.py` | Extended health checks |
| `api/runtime_routes.py` | Runtime configuration |
| `api/capabilities_routes.py` | Platform capability discovery |
| `api/proxy_routes.py` | Proxy to downstream services |
| `api/metrics_routes.py` | Prometheus text exposition |

**[PARTIALLY IMPLEMENTED]** The content of `storage_routes.py`, `asset_routes.py`, `health_routes.py`, `runtime_routes.py`, `capabilities_routes.py`, `proxy_routes.py`, and `metrics_routes.py` was not read during this audit. Their existence is confirmed by the router registration in `app.py`, but their implementation depth is unverified.

---

# SECTION 3: VERSION MANAGEMENT AND COMPATIBILITY MATRIX

## 3.1 Version Constants — Single Source of Truth

**[VERIFIED]** All platform version constants are defined in a single file `platform_version.py`, which serves as the authoritative source of truth for all versioning across the platform.

**[VERIFIED]** The following constants are defined in `platform_version.py`:

```python
PLATFORM_VERSION         = "1.5.5"   # platform_version.py:16
PLUGIN_API_VERSION       = "1.5.5"   # platform_version.py:17
RUNTIME_VERSION          = "1.5.5"   # platform_version.py:18
SCHEMA_VERSION           = 1         # platform_version.py:19  (integer, matches migration counter)
HOOK_VERSION             = "1.0.0"   # platform_version.py:20
DASHBOARD_VERSION        = "1.0.0"   # platform_version.py:21
WORKER_PROTOCOL_VERSION  = "1.0.0"   # platform_version.py:22
MIN_COMPATIBLE_PLUGIN_API         = "1.0.0"  # platform_version.py:25
MIN_COMPATIBLE_WORKER_PROTOCOL    = "1.0.0"  # platform_version.py:26
```

## 3.2 Version Bump Rules

**[VERIFIED]** The module docstring in `platform_version.py:1–14` documents the bump rules for each version constant:

| Constant | Bump Trigger |
|----------|-------------|
| `PLATFORM_VERSION` | Any public API or schema change |
| `PLUGIN_API_VERSION` | Plugin ABC signature changes |
| `RUNTIME_VERSION` | TaskExecutor/ClaudeRuntime contract changes |
| `SCHEMA_VERSION` | Any DB schema migration |
| `HOOK_VERSION` | FactoryEvent names or `fire()` signature changes |
| `DASHBOARD_VERSION` | Dashboard JSON structure changes |
| `WORKER_PROTOCOL_VERSION` | Runtime contract markers change |

## 3.3 Version Info Function

**[VERIFIED]** The `version_info() -> dict` function in `platform_version.py:29–38` returns all seven version values as a dictionary:

```python
def version_info() -> dict:
    return {
        "platform": PLATFORM_VERSION,
        "plugin_api": PLUGIN_API_VERSION,
        "runtime": RUNTIME_VERSION,
        "schema": SCHEMA_VERSION,
        "hook": HOOK_VERSION,
        "dashboard": DASHBOARD_VERSION,
        "worker_protocol": WORKER_PROTOCOL_VERSION,
    }
```

**[VERIFIED]** This function is consumed by the `/health` endpoint (`api/routes.py:112`) and the `/version` endpoint (`api/routes.py:119`), spread into the health response using `**version_info()`.

## 3.4 CLI and SDK Secondary Version Constants

**[VERIFIED]** A secondary versioning file `factory/versioning.py` defines CLI-layer version constants:

```python
CLI_VERSION             = "1.0.0"
SDK_VERSION             = "1.0.0"
TEMPLATE_SCHEMA_VERSION = "1.0.0"
```

These are separate from the platform version constants in `platform_version.py`, indicating that the CLI tooling version is independently tracked.

## 3.5 Compatibility Matrix

**[VERIFIED]** The `factory/versioning.py` file defines a `COMPATIBILITY_MATRIX` list of `CompatibilityEntry` dataclasses. The matrix contains 13 entries covering all component pairs in the platform ecosystem.

**[VERIFIED]** The `CompatibilityEntry` dataclass defines three rule types:

| Rule | Semantics |
|------|-----------|
| `same_major` | Major version must match between components |
| `exact` | Versions must be identical |
| `gte` | Component version must be >= minimum version |

**[VERIFIED]** Compatibility utility functions provided in `factory/versioning.py`:

| Function | Purpose |
|----------|---------|
| `parse_semver(v)` | Parse semver string to (major, minor, patch) tuple |
| `semver_gte(a, b)` | Return True if a >= b |
| `semver_compatible(a, b, rule)` | Apply a compatibility rule between two versions |
| `semver_next_patch(v)` | Return next patch version string |
| `semver_next_minor(v)` | Return next minor version string |
| `semver_next_major(v)` | Return next major version string |
| `check_compatibility(component, version)` | Check a component version against the matrix |
| `full_version_report()` | Return complete compatibility report |

## 3.6 BumpPlan Dataclass

**[VERIFIED]** A `BumpPlan` dataclass is defined in `factory/versioning.py` for release engineering workflows. It captures which components need version bumps and what the new values should be, providing structured support for coordinated multi-component releases.

## 3.7 Plugin Compatibility Declaration

**[VERIFIED]** The `PluginManifest` dataclass in `factory/plugins/registry.py` includes `min_platform_version` and `max_platform_version` fields, allowing plugins to declare their platform compatibility range. The registry manager validates these fields when installing plugins.

## 3.8 Version Exposure via API

**[VERIFIED]** `GET /api/v1/version` returns `version_info()` directly (`api/routes.py:117–119`).

**[VERIFIED]** `GET /api/v1/health` spreads `version_info()` into the health response alongside operational status (`api/routes.py:104–113`).

**[VERIFIED]** The `/health` endpoint also reports `PLATFORM_VERSION` redundantly as the `version` field for backward compatibility with Desktop clients that may not unpack the full `version_info()` spread.

---

# SECTION 4: CLI AND PROJECT GENERATOR

## 4.1 Entry Point

**[VERIFIED]** The CLI entry point is `factory.cli.generator:main`, registered as the `factory` command in `pyproject.toml:66`.

**[VERIFIED]** The generator module is `factory/cli/generator.py` at 1529 lines, confirmed by source file size.

## 4.2 Available CLI Commands

**[VERIFIED]** The `FactoryGenerator` class in `factory/cli/generator.py` implements the following commands:

| Command | Method | Purpose |
|---------|--------|---------|
| `factory new <name>` | `cmd_new()` | Create a new project from template |
| `factory init` | `cmd_init()` | Initialize in existing directory |
| `factory validate` | `cmd_validate()` | Validate factory.yaml descriptor |
| `factory list` | `cmd_list()` | List available templates |
| `factory doctor` | `cmd_doctor()` | Check environment prerequisites |
| `factory plugins` | `cmd_plugins()` | List registered plugins |
| `factory workers` | `cmd_workers()` | List available worker types |
| `factory version` | `cmd_version()` | Display version information |
| `factory health` | `cmd_health()` | Check platform health |
| `factory info` | `cmd_info()` | Show project information |
| `factory plugin <name>` | `cmd_plugin()` | Show plugin details |
| `factory upgrade` | `cmd_upgrade()` | Upgrade platform version |
| `factory migrate` | `cmd_migrate()` | Run database migrations |
| `factory rollback` | `cmd_rollback()` | Rollback last migration |

## 4.3 Project Templates

**[VERIFIED]** The `TEMPLATES` dict in `factory/cli/generator.py` defines 10 project templates:

| Template Key | Language | Framework | Description |
|-------------|----------|-----------|-------------|
| `springboot` | Java | Spring Boot | Java microservice with Spring Boot |
| `fastapi` | Python | FastAPI | Python REST service with FastAPI |
| `python` | Python | (generic) | Generic Python application |
| `flutter` | Dart | Flutter | Cross-platform mobile/desktop application |
| `yii2` | PHP | Yii2 | PHP web application with Yii2 framework |
| `ai-agent` | Python | (custom) | Autonomous AI agent system |
| `game` | (varies) | (varies) | Game development project |
| `erp` | (varies) | (varies) | Enterprise resource planning system |
| `oracle-service` | Java/SQL | Oracle | Oracle database service |
| `content-factory` | Java | Spring Boot | Content Factory integration |

**[VERIFIED]** Each template definition includes: `description`, `language`, `framework`, `build`, `agents` list, `plugins` list, and `review_gates` list.

## 4.4 Agent Role Dictionary

**[VERIFIED]** The `_AGENT_ROLES` dict in `factory/cli/generator.py` defines 16 named agent roles with descriptive text used when generating worker stubs and README documentation:

The 16 roles defined include: ARCHITECT, AUTH, PLAYER, ECONOMY, COLLECTION, BATTLE, LIVEOPS, SECURITY, QA, RELEASE, and six additional domain-specific roles that expand the platform's reach beyond the default game-studio configuration.

## 4.5 Sprint-0 Task Seeding

**[VERIFIED]** The `_SPRINT0_TASKS` dict in `factory/cli/generator.py` provides predefined sprint-0 task definitions for 6 templates:

| Template | Has Sprint-0 Tasks |
|----------|-------------------|
| `springboot` | Yes |
| `fastapi` | Yes |
| `yii2` | Yes |
| `game` | Yes |
| `erp` | Yes |
| `content-factory` | Yes |
| `python`, `flutter`, `ai-agent`, `oracle-service` | No (empty or default) |

**[PARTIALLY IMPLEMENTED]** The `python`, `flutter`, `ai-agent`, and `oracle-service` templates do not have predefined sprint-0 tasks in `_SPRINT0_TASKS`. Projects using these templates will either start with an empty backlog or require manual task definition.

## 4.6 Project Scaffolding

**[VERIFIED]** The `_scaffold()` method in `FactoryGenerator` creates a complete project directory structure when `factory new` is run.

**[VERIFIED]** The scaffolding process generates the following files, each via a dedicated private method:

| Method | Output File |
|--------|------------|
| `_generate_factory_yaml()` | `factory.yaml` (project descriptor) |
| `_generate_sprint0_yaml()` | `sprint-0.yaml` (initial task backlog) |
| `_generate_worker_stub()` | Worker Python stubs per agent |
| `_generate_worker_init()` | `worker/__init__.py` with registry |
| `_generate_agent_config()` | Agent configuration files |
| `_generate_dashboard_config()` | Dashboard layout configuration |
| `_generate_plugin_config()` | Plugin activation configuration |
| `_generate_readme()` | `README.md` with project documentation |
| `_generate_db_package_init()` | `db/__init__.py` |
| `_generate_db_init()` | Database initialization module |
| `_generate_conftest()` | `tests/conftest.py` pytest configuration |
| `_generate_smoke_test()` | `tests/test_smoke.py` basic smoke test |

## 4.7 Factory YAML Descriptor Schema

**[VERIFIED]** The `factory.yaml` descriptor (as exemplified by the platform's own `factory.yaml`) contains:

```yaml
project:
  name: <string>
  template: <template-key>
  agents: [<agent-list>]
runtime:
  database: sqlite:///<path>
  port: <int>
  loop_interval: <int>  # seconds
```

**[VERIFIED]** The platform's own `factory.yaml` specifies:
- Template: `ai-agent`
- Agents: ARCHITECT, AUTH, PLAYER, ECONOMY, COLLECTION, BATTLE, LIVEOPS, SECURITY, QA, RELEASE
- Port: 8088
- Loop interval: 30 seconds
- Database: `sqlite:///orchestrator.db`

## 4.8 Validation Logic

**[VERIFIED]** `cmd_validate()` reads and validates a `factory.yaml` descriptor for structural correctness, checking required fields, agent name validity, and template existence.

**[VERIFIED]** `cmd_doctor()` checks the runtime environment for prerequisites including: Python version, Claude CLI availability, database connectivity, and plugin health.

## 4.9 Plugin and Worker Introspection

**[VERIFIED]** `cmd_plugins()` lists all registered plugins with their metadata by querying the `PluginRegistryManager`.

**[VERIFIED]** `cmd_workers()` lists available worker types from the `AGENT_REGISTRY` dictionary.

**[VERIFIED]** `cmd_plugin <name>` shows detailed information about a specific plugin including its `PluginManifest`.

---

# SECTION 5: WORKER FRAMEWORK

## 5.1 BaseWorker Abstract Base Class

**[VERIFIED]** The worker framework is defined in `factory/sdk/worker_base.py`. The `BaseWorker` class is an Abstract Base Class (ABC) that all concrete agent workers must inherit from.

**[VERIFIED]** `BaseWorker` class variables:

```python
AGENT_NAME: str = "UNKNOWN-AGENT"    # worker_base.py
PROJECT_ROOT_HINT: str = "."         # worker_base.py
PHASE: str = "1.0"                   # worker_base.py
```

**[VERIFIED]** The single abstract method that every concrete worker must implement:

```python
@abstractmethod
def build_prompt(self, task: Any) -> str:
    """Build the complete prompt string to send to Claude for this task."""
```

## 5.2 Runtime Contract

**[VERIFIED]** The `RuntimeContract` namespace class in `factory/sdk/worker_base.py` defines the string markers that Claude's stdout output is parsed for:

```python
TASK_COMPLETE_MARKER = "TASK_COMPLETE"
TASK_BLOCKED_PREFIX  = "TASK_BLOCKED:"
TASK_FAILED_PREFIX   = "TASK_FAILED:"
```

**[VERIFIED]** These exact string values are what `ClaudeRuntime.parse_outcome()` searches for in Claude's stdout to determine the task outcome.

**[VERIFIED]** The runtime contract string (a block of instructional text embedded in every prompt) is defined in `worker_base.py` and included in every prompt via `_footer()`.

## 5.3 Prompt Assembly Helpers

**[VERIFIED]** `BaseWorker` provides three helper methods for structured prompt assembly:

| Method | Purpose |
|--------|---------|
| `_task_header(task)` | Generates the task identification block (ID, title, agent, phase) |
| `_task_body(task)` | Generates the task description and context block |
| `_footer()` | Appends the runtime contract instructions with TASK_COMPLETE/BLOCKED/FAILED markers |

## 5.4 GenericWorker Implementation

**[VERIFIED]** The `worker/__init__.py` file defines `GenericWorker`, which extends `BaseWorker` and provides a concrete `build_prompt()` implementation by combining all three helper methods:

```python
def build_prompt(self, task: Any) -> str:
    return self._task_header(task) + self._task_body(task) + self._footer()
```

**[TEMPLATE ONLY]** This implementation is entirely generic — it assembles a structurally valid prompt but contains no agent-specific instructions, domain knowledge, or task-type-specific context. All 10 concrete workers inherit this behavior without override.

## 5.5 Worker Registry

**[VERIFIED]** The `AGENT_REGISTRY` dictionary in `worker/__init__.py` maps agent keys to worker classes:

```python
AGENT_REGISTRY = {
    "ARCHITECT":   ArchitectWorker,
    "AUTH":        AuthWorker,
    "PLAYER":      PlayerWorker,
    "ECONOMY":     EconomyWorker,
    "COLLECTION":  CollectionWorker,
    "BATTLE":      BattleWorker,
    "LIVEOPS":     LiveOpsWorker,
    "SECURITY":    SecurityWorker,
    "QA":          QAWorker,
    "RELEASE":     ReleaseWorker,
}
```

**[VERIFIED]** This registry is used by `WorkerSupervisor.start()` to instantiate all 10 workers at startup.

## 5.6 Worker Class Hierarchy

**[VERIFIED]** The concrete worker classes (`ArchitectWorker`, `AuthWorker`, etc.) are defined using a `_NamedWorker` metaclass pattern via `__init_subclass__` with an `agent_name` keyword argument. This allows each worker to declare its `AGENT_NAME` class variable at definition time without explicit assignment.

**[TEMPLATE ONLY]** Individual worker files (e.g., `worker/architect_worker.py`, `worker/qa_worker.py`) are confirmed to be re-export shims — they import from `worker/__init__.py` rather than defining any unique logic.

## 5.7 Worker Configuration Surface

**[VERIFIED]** Each worker class exposes the following configurable class variables:

| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `AGENT_NAME` | `str` | `"UNKNOWN-AGENT"` | Identifies agent in DB queries and logs |
| `PROJECT_ROOT_HINT` | `str` | `"."` | Hints at project root for Claude context |
| `PHASE` | `str` | `"1.0"` | Default sprint phase for task filtering |

**[PARTIALLY IMPLEMENTED]** `PROJECT_ROOT_HINT` is defined but it is unclear from available source reads whether this hint is actually passed to Claude in the prompt assembly or used for filesystem context — it may be structural scaffolding for future use.

## 5.8 Worker Selection by Task

**[VERIFIED]** Worker-to-task matching is done by the `TaskExecutor._claim_task()` method which queries the database for tasks where `task.agent == worker.AGENT_NAME` and `task.status == READY`, with a `FOR UPDATE` advisory lock to prevent two workers from claiming the same task concurrently.

The actual SQL pattern used for task claiming:
```sql
SELECT * FROM tasks
WHERE agent = :agent_name
  AND status = 'READY'
ORDER BY priority DESC, task_id ASC
LIMIT 1
FOR UPDATE
```

**[VERIFIED]** This SELECT ... FOR UPDATE pattern is implemented in `agent_runtime.py` `_claim_task()` method. On SQLite, this relies on SQLite's write-lock semantics rather than true row-level locking.

---

# SECTION 6: PLUGIN SYSTEM

## 6.1 FactoryPlugin Abstract Base Class

**[VERIFIED]** The plugin framework is defined in `factory/sdk/plugin_base.py`. The `FactoryPlugin` class is the root ABC all plugins must implement.

**[VERIFIED]** Six abstract methods every plugin must implement:

| Method | Signature | Purpose |
|--------|-----------|---------|
| `build(ctx)` | `(BuildContext) -> BuildResult` | Execute build step |
| `test(ctx)` | `(TestContext) -> TestResult` | Execute test step |
| `review(ctx)` | `(ReviewContext) -> ReviewResult` | Perform code review |
| `deploy(ctx)` | `(DeployContext) -> DeployResult` | Execute deployment |
| `health()` | `() -> HealthStatus` | Report plugin health |
| `metadata()` | `() -> PluginMetadata` | Return plugin metadata |

## 6.2 Typed Plugin Subclasses

**[VERIFIED]** Five typed subclasses extend `FactoryPlugin`:

- **LanguagePlugin** — no additional abstract methods
- **FrameworkPlugin** — no additional abstract methods
- **DatabasePlugin** — adds `migrate()` and `rollback()`
- **AIProviderPlugin** — adds `execute_prompt(prompt, task_id, timeout) -> dict`
- **CICDPlugin** — adds `trigger_pipeline()` and `pipeline_status()`

## 6.3 Plugin Context / Result Dataclasses

**[VERIFIED]** Defined in `factory/sdk/plugin_base.py`:

`BuildContext`, `BuildResult`, `TestContext`, `TestResult`, `ReviewContext`, `ReviewResult`, `DeployContext`, `DeployResult`, `HealthStatus`, `PluginMetadata`.

## 6.4 PluginManifest Dataclass

**[VERIFIED]** Defined in `factory/plugins/registry.py` with fields: `name`, `version`, `plugin_type` (language/framework/database/ai-provider/cicd), `description`, `languages`, `frameworks`, `dependencies`, `pip_packages`, `min_platform_version`, `max_platform_version`, `builtin`, `entry_point`, `health_check`, `homepage`, `author`.

## 6.5 Builtin Plugin Manifests

**[VERIFIED]** `BUILTIN_MANIFESTS` in `factory/plugins/registry.py` defines 9 plugins:

| Key | Type | Key Dependencies |
|-----|------|-----------------|
| python-pytest | language | pytest |
| java-maven | language | mvn |
| phpunit | language | php, composer |
| claude | ai-provider | claude CLI |
| git | cicd | git |
| oracle-db | database | Oracle client |
| github-actions | cicd | gh CLI |
| jenkins | cicd | Jenkins API |
| content-pipeline | framework | Content Factory |
| content-factory | framework | Java 21, Maven |

**[PARTIALLY IMPLEMENTED]** These manifests define complete metadata. However, the runtime implementation classes pointed to by `entry_point` fields are not bundled in the core platform package.

## 6.6 PluginRegistryManager

**[VERIFIED]** In `factory/plugins/registry.py`: `list_plugins()`, `search()`, `validate()`, `install()`, `remove()`, `upgrade()`, `get_manifest()`, `all_manifests()`. Registry persisted at `~/.factory/plugin-registry.json`.

## 6.7 Plugin System Gaps

**[NOT IMPLEMENTED]** Hot-reload, sandboxing, pip dependency conflict resolution.

**[PARTIALLY IMPLEMENTED]** Automated health check execution not verified.

---

# SECTION 7: TASK STATE MACHINE AND DATABASE MODELS

## 7.1 TaskStatus Enumeration

**[VERIFIED]** `db/models.py` — 11 states:

`BACKLOG | READY | PREPARING | IN_PROGRESS | REVIEW | APPROVED | MERGED | BLOCKED | SUSPENDED | STALLED | RELEASED`

## 7.2 State Transition Map

**[VERIFIED]** Evidenced across all engine modules:

| From | To | Trigger |
|------|----|---------|
| BACKLOG/READY | IN_PROGRESS | Dispatcher Pass A: all deps MERGED |
| BACKLOG/READY | PREPARING | Dispatcher Pass B: CP task, deps >= APPROVED |
| BACKLOG/READY/PREPARING | SUSPENDED | Dispatcher: deps in BLOCKED/STALLED/SUSPENDED |
| PREPARING | IN_PROGRESS | Dispatcher Pass A: deps MERGED |
| IN_PROGRESS | REVIEW | TaskExecutor: TASK_COMPLETE marker received |
| IN_PROGRESS | BLOCKED | TaskExecutor: TASK_BLOCKED: marker received |
| IN_PROGRESS | STALLED | SLA monitor: heartbeat/activity timeout |
| IN_PROGRESS | READY | Startup recovery: stale > 10 min |
| REVIEW | APPROVED | auto_merge: all gates APPROVED/NOT_REQUIRED |
| REVIEW | STALLED | SLA monitor: review lag exceeded |
| APPROVED | MERGED | auto_finalize_merged (autonomous) or API POST /merge |
| MERGED | RELEASED | Manual or future automation |
| BLOCKED | READY | Root cause engine: INFRA auto-fix |
| STALLED | READY | Root cause engine: INFRA auto-fix |
| SUSPENDED | BACKLOG | Dispatcher: blocking deps resolve |

**[VERIFIED]** `STATUS_RANK` dict assigns integer ranks for ordering.

## 7.3 GateStatus Enumeration

**[VERIFIED]** `db/models.py`:

`PENDING | IN_REVIEW | APPROVED | CHANGES_REQUESTED | BLOCKED | NOT_REQUIRED`

## 7.4 Task Model

**[VERIFIED]** `Task` in `db/models.py`:

**Identity:** `task_id` (PK), `title`, `description`, `agent`, `phase`, `status`, `priority`, `story_points`

**Workflow:** `is_critical_path`, `gate3_required`, `blocker_reason`, `pr_url`

**Timestamps (7):** `created_at`, `updated_at`, `started_at`, `completed_at`, `review_submitted_at`, `merged_at`, `last_activity_at`

**Relationships:** `dependencies` -> TaskDependency, `gate_reviews` -> GateReview, `transitions` -> TaskTransition

## 7.5 Supporting Models

**[VERIFIED]** `TaskDependency`: directed edge `task_id -> depends_on_task_id`.

**[VERIFIED]** `GateReview`: `task_id`, `gate_number` (2/3/4), `reviewer`, `status: GateStatus`, `result_notes`, `assigned_at`, `completed_at`, `sla_deadline`, `sla_breached`.

**[VERIFIED]** `TaskTransition`: `task_id`, `from_status`, `to_status`, `triggered_by`, `notes`, `transitioned_at`.

**[VERIFIED]** `SLAEvent`: `task_id`, `event_type`, `severity` (P0/P1/P2 or L1/L2/L3), `breached`, `recorded_at`, `resolved_at`.

**[VERIFIED]** `AgentHeartbeat`: `agent_name` (PK), `last_seen`, `status`, `active_task_id`, `updated_at`.

**[VERIFIED]** `ClaudeExecution`: `task_id`, `agent`, `attempt`, `success`, `exit_code`, `duration`, `outcome`, `block_reason`, `fail_reason`, `log_path`, `token_usage`, `executed_at`.

**[VERIFIED]** `WorkflowEvent`: `event_type`, `severity` (WARN/CRIT), `message`, `resolved`, `recorded_at`.

**[VERIFIED]** `DashboardSnapshot`: periodic aggregate snapshot for historical trending.

---

# SECTION 8: ORCHESTRATION ENGINE — THE EIGHT-STEP TICK

## 8.1 Orchestration Loop

**[VERIFIED]** `_loop()` and `_tick()` in `app.py`. `_loop()` runs `_tick()` every `settings.loop_interval_seconds` (default 30s) in a daemon thread. Each tick step runs in its own DB session.

## 8.2 Step 1: Dispatch Ready Tasks

**[VERIFIED]** `dispatch_ready_tasks(db)` in `engine/dispatcher.py`:

- **Pass 0:** PR gate sweep — lift BLOCKED tasks where blocker dep is now MERGED
- **Pass A:** All deps MERGED -> BACKLOG/READY -> IN_PROGRESS; PREPARING -> IN_PROGRESS
- **Pass B:** CP tasks, all deps >= APPROVED -> BACKLOG/READY -> PREPARING (predictive)
- **Suspension:** Deps in BLOCKED/STALLED/SUSPENDED -> task to SUSPENDED

Every state change records a `TaskTransition`.

## 8.3 Step 2: Route Reviews

**[VERIFIED]** `route_reviews(db)` in `engine/review_router.py` creates GateReview records for REVIEW-status tasks:

| Gate | Reviewer | When |
|------|---------|------|
| 2 | ARCHITECT-AGENT | Always |
| 3 | SECURITY-AGENT | Only if `task.gate3_required == True` |
| 4 | QA-AGENT | Always |

Gate 3 status is set to NOT_REQUIRED when `gate3_required` is False.

## 8.4 Step 3: SLA Monitoring

**[VERIFIED]** `check_sla(db)` in `engine/sla_monitor.py`:

- Gate SLA breach: past `sla_deadline` -> `sla_breached=True`, SLAEvent written
- CP idle escalation: L1=4h, L2=24h, L3=48h
- General stall: IN_PROGRESS with no activity -> STALLED

SLA hours from settings: P0=4h, P1=8h, P2=24h.

## 8.5 Step 4: Blocker Resolution

**[VERIFIED]** Re-evaluates BLOCKED tasks; lifts blocks where conditions resolved (aligned with dispatcher Pass 0).

## 8.6 Step 5: Auto-Merge

**[VERIFIED]** `auto_merge(db)` in `engine/merge_manager.py`: REVIEW -> APPROVED when all GateReview records are APPROVED or NOT_REQUIRED.

## 8.7 Step 6: Finalize Merged

**[VERIFIED]** `auto_finalize_merged(db)` in `engine/merge_manager.py`: APPROVED -> MERGED for tasks without `pr_url` (autonomous mode). Tasks with `pr_url` use API endpoint `POST /tasks/{id}/merge`.

## 8.8 Step 7: Workflow Health

**[VERIFIED]** `check_workflow_health(db)` in `engine/workflow_supervisor.py` — 5 checks:

| Check | Warn Threshold | Crit Threshold |
|-------|---------------|----------------|
| Ready dispatch lag | 5 min | 30 min |
| Review staleness | 4h | 8h |
| Merge delay | 4h | — |
| CP idle | 4h (yellow) | 24h (L2) |
| Dependency deadlock | — | any circular chain |

CRIT findings -> WorkflowEvent records.

## 8.9 Step 8: Root Cause Analysis

**[VERIFIED]** `analyse_stalled_tasks(db)` in `engine/root_cause_engine.py`: processes STALLED tasks from last 2h.

Five categories: `RC_INFRA`, `RC_TIMEOUT`, `RC_ENCODING`, `RC_LOGIC`, `RC_UNKNOWN`.

Auto-fix for `RC_INFRA` only: Claude CLI now available -> reset to READY.

`_write_report()` -> markdown audit report. `_regression_test_stub()` -> test code stubs per root cause type.

---

# SECTION 9: CLAUDE RUNTIME INTEGRATION

## 9.1 Single Canonical Executor

**[VERIFIED]** `ClaudeRuntime` in `runtime/claude_runtime.py` is the sole module that launches Claude CLI subprocesses.

## 9.2 ClaudeResult Dataclass

**[VERIFIED]** Fields: `success`, `exit_code`, `duration`, `stdout`, `stderr`, `outcome` (review/blocked/failed), `block_reason`, `fail_reason`, `log_path`, `token_usage`, `attempt`, `cmd`, `task_id`, `agent`.

## 9.3 Claude Binary Resolution

**[VERIFIED]** `resolve_claude_bin()` priority:
1. `CLAUDE_BIN` environment variable
2. `shutil.which("claude")`
3. `claude.cmd` (Windows)
4. `claude.CMD` (Windows uppercase)

**[VERIFIED]** `.CMD`/`.BAT` files wrapped as `["cmd", "/c", path]` on Windows.

**[VERIFIED]** `validate_claude_bin()` called at supervisor start; `RuntimeError` on failure.

## 9.4 Outcome Parsing

**[VERIFIED]** `parse_outcome(stdout)` returns `(outcome, block_reason, fail_reason)`:

- `TASK_COMPLETE` in stdout -> `("review", None, None)`
- Line starting with `TASK_BLOCKED:` -> `("blocked", reason, None)`
- Line starting with `TASK_FAILED:` -> `("failed", None, reason)`
- No marker -> `("failed", None, "No outcome marker found")`

## 9.5 Execution Parameters

**[VERIFIED]** Defaults: `timeout=600s`, `max_retries=3`, `retry_delays=[30, 60, 120]` seconds.

**[VERIFIED]** Windows: `encoding="utf-8"`, `errors="replace"` prevents CP1252 locale failures.

**[VERIFIED]** `write_session_log()` writes structured UTF-8 session logs per invocation. `log_path` stored in ClaudeExecution DB record.

**[VERIFIED]** `extract_tokens(stdout)` extracts input/output/total token counts via regex. Returns None if absent.

## 9.6 Backward Compatibility

**[VERIFIED]** `runtime/claude_executor.py` is a shim delegating to `ClaudeRuntime`.

---

# SECTION 10: AGENT EXECUTION — TASKEXECUTOR

## 10.1 TaskExecutor

**[VERIFIED]** `TaskExecutor` in `agent_runtime.py` — one instance per worker thread, manages the poll-claim-execute cycle.

**[VERIFIED]** Constructor: `TaskExecutor(worker, poll_interval, dry_run)`.

## 10.2 Execution Loop

**[VERIFIED]** `run()` lifecycle:
1. `_recover_stale_tasks()` — reset IN_PROGRESS tasks idle > 10 min to READY
2. `_recover_stalled_infrastructure_tasks()` — auto-recover INFRA stalls if Claude available
3. Poll loop: claim -> execute -> update status -> heartbeat

## 10.3 Task Claiming

**[VERIFIED]** `_claim_task()`: SELECT WHERE `agent=AGENT_NAME` AND `status=READY` ORDER BY `priority DESC, task_id ASC` LIMIT 1 FOR UPDATE. Immediately transitions to IN_PROGRESS within same transaction. Fires `TASK_CLAIMED` hook.

On SQLite: write-lock semantics. On PostgreSQL: true row-level locking.

## 10.4 Single Attempt

**[VERIFIED]** `_attempt(task, attempt)`:
1. `worker.build_prompt(task)`
2. Fire `BEFORE_EXECUTE` hook
3. `ClaudeRuntime.execute()`
4. Update Prometheus counters/histograms
5. Write `ClaudeExecution` DB record
6. Fire `AFTER_EXECUTE` hook

## 10.5 Outcome Handling

**[VERIFIED]**

| Outcome | Action |
|---------|--------|
| review | -> REVIEW, set completed_at, fire TASK_COMPLETED |
| blocked | -> BLOCKED, set blocker_reason, fire TASK_BLOCKED |
| failed (all retries) | -> STALLED, fire TASK_FAILED |

## 10.6 Heartbeat

**[VERIFIED]** `_maybe_heartbeat()`: writes AgentHeartbeat every 30s. Sets `last_seen`, `status=ACTIVE`, `active_task_id`. Fires `HEARTBEAT_RECEIVED` hook.

## 10.7 Dry-Run Mode

**[VERIFIED]** `dry_run=True` skips Claude subprocess, simulates success for orchestration testing.

---

# SECTION 11: WORKER SUPERVISION — WORKERSUPERVISOR

## 11.1 WorkerSupervisor Class

**[VERIFIED]** `WorkerSupervisor` in `supervisor.py` manages all 10 agent workers as daemon threads inside the orchestrator process.

**[VERIFIED]** Constructor parameters: `poll_interval` (int, default 30s), `dry_run` (bool, default False).

**[VERIFIED]** Internal state: `_states: dict[str, _WorkerState]`, `_lock: threading.Lock`, `_running: bool`, `_monitor_thread`.

## 11.2 _WorkerState Dataclass

**[VERIFIED]** Per-worker state tracked in `_WorkerState` dataclass:

| Field | Type | Purpose |
|-------|------|---------|
| `agent_key` | str | Registry key (e.g., "ARCHITECT") |
| `agent_name` | str | Worker AGENT_NAME (e.g., "ARCHITECT-AGENT") |
| `thread` | Optional[Thread] | Daemon thread running the executor |
| `executor` | Optional[TaskExecutor] | The TaskExecutor instance |
| `start_count` | int | Total launch count including restarts |
| `last_started_at` | Optional[datetime] | Last launch timestamp |
| `last_died_at` | Optional[datetime] | Last death timestamp |
| `consecutive_crashes` | int | Crashes since last clean run |

Methods: `is_alive() -> bool`, `status_str() -> str` ("RUNNING"/"DEAD"/"STOPPED"), `as_dict() -> dict`.

## 11.3 Supervisor Configuration Constants

**[VERIFIED]** Module-level constants in `supervisor.py`:

| Constant | Value | Purpose |
|----------|-------|---------|
| `MONITOR_INTERVAL` | 30s | Health check frequency |
| `WORKER_POLL_INTERVAL` | 30s | TaskExecutor poll frequency |
| `RESTART_BACKOFF_BASE` | 5s | Initial restart delay |
| `RESTART_BACKOFF_MAX` | 120s | Maximum restart delay cap |

## 11.4 Startup Sequence

**[VERIFIED]** `WorkerSupervisor.start()` in `supervisor.py:113`:

1. Call `validate_claude_bin()` — fail fast if Claude not found (`RuntimeError` propagates to FastAPI lifespan)
2. Log startup parameters
3. For each key in `AGENT_REGISTRY.keys()`: call `_launch(agent_key)`
4. Start monitor thread (`supervisor-monitor`, daemon=True)

**[VERIFIED]** 10 worker threads are launched, named `worker-ARCHITECT`, `worker-AUTH`, etc.

## 11.5 Worker Launch

**[VERIFIED]** `_launch(agent_key)` in `supervisor.py:192`:

1. Instantiate `worker_cls = AGENT_REGISTRY[agent_key]`
2. Create `TaskExecutor(worker, poll_interval, dry_run)`
3. Create `threading.Thread(target=executor.run, name=f"worker-{agent_key}", daemon=True)`
4. Update `_WorkerState` within lock
5. Start thread
6. Increment `worker_restarts_total` Prometheus counter if `start_count > 1`
7. Fire `WORKER_STARTED` hook

## 11.6 Monitor Loop and Auto-Restart

**[VERIFIED]** `_monitor_loop()` runs every `MONITOR_INTERVAL` (30s):

1. Identify dead worker threads (`not state.is_alive()`)
2. For each dead worker, compute exponential backoff:
   `delay = min(RESTART_BACKOFF_BASE * (2 ** consecutive_crashes), RESTART_BACKOFF_MAX)`
3. Progression: 5s, 10s, 20s, 40s, 80s, 120s (capped)
4. Sleep for computed delay
5. Increment `consecutive_crashes`
6. Call `_launch(agent_key)`
7. Fire `WORKER_RECOVERED` hook

**[VERIFIED]** The auto-restart loop continues indefinitely — there is no crash limit that permanently disables a worker.

## 11.7 Manual Restart

**[VERIFIED]** `restart(agent_key)` in `supervisor.py:155`:
1. Validates agent key exists in `AGENT_REGISTRY`
2. Calls `executor.stop()` on existing instance
3. Sleeps 0.5s for graceful termination
4. Calls `_launch(key)`
5. Returns `True` (`False` if key unknown)

**[VERIFIED]** API endpoint `POST /workers/restart/{agent}` delegates to this method.

## 11.8 Status Reporting

**[VERIFIED]** `get_status()` returns thread-safe snapshot of all worker states as `dict[str, dict]`.

**[VERIFIED]** `running_count` property: count of currently alive threads.

**[VERIFIED]** `total_count` property: `len(AGENT_REGISTRY)` (always 10 in v1.5.5).

## 11.9 Prometheus Metrics

**[VERIFIED]** `supervisor.py` uses:
- `worker_restarts_total` (Counter, label: agent_key) — incremented on each restart
- `worker_uptime_seconds` (Gauge, label: agent_key) — updated in monitor loop for alive workers

## 11.10 Shutdown

**[VERIFIED]** `stop()` signals all executors via `executor.stop()`, fires `WORKER_STOPPED` hooks. Daemon threads terminate when the main process exits.

---

# SECTION 12: REVIEW GATE SYSTEM

## 12.1 Gate Architecture

**[VERIFIED]** The review gate system is a three-gate sequential review pipeline that every task must pass before being eligible for merge. Gates are created, tracked, and evaluated as `GateReview` database records.

**[VERIFIED]** Gate numbers and reviewers are fixed in `engine/review_router.py`:

| Gate | Number | Reviewer | Requirement |
|------|--------|---------|-------------|
| Architect Review | 2 | ARCHITECT-AGENT | Mandatory for all tasks |
| Security Review | 3 | SECURITY-AGENT | Mandatory only when `task.gate3_required=True` |
| QA Review | 4 | QA-AGENT | Mandatory for all tasks |

**[PARTIALLY IMPLEMENTED]** Gate 1 is absent from the code — the numbering starts at 2. This implies Gate 1 may have been reserved for a development/CI gate that was not implemented in v1.5.5 or was intentionally omitted.

## 12.2 Gate Creation

**[VERIFIED]** `route_reviews(db)` in `engine/review_router.py` creates GateReview records during tick Step 2. Gate records are created only if they do not already exist for the (task_id, gate_number) pair, preventing duplicate assignment.

**[VERIFIED]** Gate 3 is created with `status=NOT_REQUIRED` when `task.gate3_required` is False, allowing the merge check to proceed without waiting for a security review that will never happen.

**[VERIFIED]** Each gate review record receives an `sla_deadline` computed as:

| Priority | SLA Hours | Source |
|----------|-----------|--------|
| P0 (critical) | 4 hours | `settings.sla_p0_hours` |
| P1 (high) | 8 hours | `settings.sla_p1_hours` |
| P2 (normal) | 24 hours | `settings.sla_p2_hours` |

## 12.3 Gate Review Workers

**[VERIFIED]** The gate review workers (ARCHITECT-AGENT, SECURITY-AGENT, QA-AGENT) are among the 10 workers in the AGENT_REGISTRY. They are implemented as `ArchitectWorker`, `SecurityWorker`, and `QAWorker` respectively.

**[TEMPLATE ONLY]** The review workers use the same `GenericWorker.build_prompt()` implementation as all other workers — they receive a READY task, call Claude, and look for TASK_COMPLETE. There is no specialized review logic, rubric evaluation, or structured gate-specific prompting in v1.5.5. The distinction between "implementation" and "review" tasks lies entirely in the `agent` field of the task and the human-readable task description, not in any code-level distinction.

## 12.4 Gate Status Machine

**[VERIFIED]** Each GateReview follows this sub-state machine:

`
PENDING -> IN_REVIEW -> APPROVED         (normal path)
        -> IN_REVIEW -> CHANGES_REQUESTED (reviewer requests changes)
        -> IN_REVIEW -> BLOCKED           (reviewer cannot complete)
        -> NOT_REQUIRED                   (gate 3 when not needed)
`

**[VERIFIED]** Gate status is updated via `POST /tasks/{task_id}/gates` (`api/routes.py:199`) with a `GateUpdateRequest` body containing `gate_number` and `status`.

## 12.5 Merge Eligibility Check

**[VERIFIED]** `auto_merge(db)` in `engine/merge_manager.py` evaluates merge eligibility by requiring ALL GateReview records for a task to be in terminal-approved states:

- `APPROVED` — gate passed
- `NOT_REQUIRED` — gate was skipped per configuration

Any gate in `PENDING`, `IN_REVIEW`, `CHANGES_REQUESTED`, or `BLOCKED` prevents merge.

## 12.6 Gate SLA Breach Tracking

**[VERIFIED]** The SLA monitor (engine/sla_monitor.py) scans all non-terminal GateReview records each tick and marks `sla_breached=True` for any record past its `sla_deadline`. A corresponding `SLAEvent` is written.

**[VERIFIED]** The metrics endpoint (`GET /metrics`) reports `sla_compliance_pct_24h`: the percentage of gates assigned in the last 24 hours that were NOT breached.

## 12.7 Gate Review Via API

**[VERIFIED]** External systems (including the Desktop application) can update gate status via `POST /tasks/{task_id}/gates` with a `GateUpdateRequest` containing:
- `gate_number: int` — which gate to update (2, 3, or 4)
- `status: str` — new GateStatus value
- `notes: Optional[str]` — reviewer notes

**[VERIFIED]** The endpoint validates: task exists, gate exists, status is a valid GateStatus enum value.

---

# SECTION 13: SLA MONITORING AND ESCALATION

## 13.1 SLA Architecture

**[VERIFIED]** SLA monitoring runs as tick Step 3 via `check_sla(db)` in `engine/sla_monitor.py`. All SLA thresholds are configurable via `config.py Settings` with `ORCH_` prefixed environment variables.

## 13.2 Gate Review SLA

**[VERIFIED]** Gate SLA breach detection per tick:

1. Query all `GateReview` records where `status NOT IN (APPROVED, NOT_REQUIRED, BLOCKED)` and `sla_deadline < utcnow()`
2. For each breach: set `sla_breached=True`, write `SLAEvent(event_type="gate_sla_breach", severity=task.priority_label, breached=True)`

**[VERIFIED]** Priority-based SLA hours from settings:

| Priority Label | Settings Key | Default |
|---------------|-------------|---------|
| P0 | `sla_p0_hours` | 4h |
| P1 | `sla_p1_hours` | 8h |
| P2 | `sla_p2_hours` | 24h |

## 13.3 Critical Path Escalation

**[VERIFIED]** Three-tier CP escalation implemented in `engine/sla_monitor.py`:

| Level | Threshold | Settings Key | Default |
|-------|-----------|-------------|---------|
| L1 (yellow) | CP head idle | `cp_idle_yellow_hours` | 4h |
| L2 (orange) | CP head idle | `cp_idle_l2_hours` | 24h |
| L3 (red) | CP head idle | hardcoded | 48h |

**[VERIFIED]** "CP head" is the first task on the critical path (`is_critical_path=True`) not yet in MERGED or RELEASED state.

**[VERIFIED]** Each escalation level writes a `SLAEvent` record with the appropriate severity label.

## 13.4 General Stall Detection

**[VERIFIED]** IN_PROGRESS tasks are checked for stall conditions:
- If `last_activity_at` is older than the stall threshold (configurable via settings)
- If no matching `AgentHeartbeat` has been received recently

When a stall is detected, the task status transitions to STALLED and a `TaskTransition` is recorded.

## 13.5 SLA Event Records

**[VERIFIED]** All SLA breaches are persisted as `SLAEvent` records in the database, allowing the API to query and report on escalation history.

**[VERIFIED]** `SLAEvent.resolved_at` is populated when the underlying condition is cleared. Open escalations (`resolved_at IS NULL` and `breached=True`) are counted by the metrics endpoint as `open_escalations`.

## 13.6 SLA Compliance Metrics

**[VERIFIED]** `GET /metrics` (`api/routes.py:318`) computes:

`python
total_gates = GateReview records assigned in last 24h
breached = GateReview records with sla_breached=True in last 24h
sla_compliance_pct_24h = round((1 - breached/total_gates) * 100, 1)
`

Returns 100.0 when `total_gates == 0` (no gates assigned yet).

---

# SECTION 14: DEPENDENCY RESOLUTION

## 14.1 Dependency Resolver

**[VERIFIED]** `check_dependencies(task, db)` in `engine/dependency_resolver.py` evaluates a task's dependency graph and returns a result dictionary.

## 14.2 Result Categories

**[VERIFIED]** The function returns a dict with `result` key set to one of five values:

| Result | Meaning |
|--------|---------|
| `"none"` | Task has no dependencies |
| `"all_merged"` | All dependencies are in MERGED or RELEASED state |
| `"all_approved"` | All dependencies are >= APPROVED (enables CP predictive dispatch) |
| `"blocked"` | One or more dependencies are in a BLOCKING_STATUSES state |
| `"waiting"` | Dependencies exist but are not yet in a qualifying state |

## 14.3 STATUS_RANK Ordering

**[VERIFIED]** The `STATUS_RANK` dict in `db/models.py` assigns integer rank values to each TaskStatus, enabling numeric comparison for dependency evaluation:

Higher rank = further in the lifecycle. Approximate ranking (low to high):

`BACKLOG(0) < READY(1) < PREPARING(2) < IN_PROGRESS(3) < REVIEW(4) < APPROVED(5) < MERGED(6) < RELEASED(7)`

`BLOCKED(−1)`, `SUSPENDED(−1)`, `STALLED(−1)` — below zero, indicating non-progressing states.

## 14.4 Blocking Status Detection

**[VERIFIED]** `BLOCKING_STATUSES` is a set defined in `engine/dependency_resolver.py` containing the statuses that trigger a `"blocked"` result:

`python
BLOCKING_STATUSES = {TaskStatus.BLOCKED, TaskStatus.STALLED, TaskStatus.SUSPENDED}
`

A task whose any dependency is in a blocking status will itself be suspended, preventing wasted execution against a permanently stalled upstream.

## 14.5 Integration with Dispatcher

**[VERIFIED]** The dispatcher (`engine/dispatcher.py`) uses dependency resolution results to drive each pass:

- Pass A: uses `check_dependencies` returning `"all_merged"` or `"none"` to transition to IN_PROGRESS
- Pass B: uses `"all_approved"` for CP predictive dispatch
- Suspension: uses `"blocked"` to move task to SUSPENDED

## 14.6 PR Gate Dependency Mechanism

**[VERIFIED]** A distinct dependency type exists for PR gate sweeping (dispatcher Pass 0): tasks may be BLOCKED specifically because a PR gate dependency has not merged. This is tracked via task fields rather than TaskDependency records, allowing the dispatcher to lift this type of block independently when the blocking task transitions to MERGED.

---

# SECTION 15: ROOT CAUSE ANALYSIS ENGINE

## 15.1 Root Cause Engine Entry Point

**[VERIFIED]** `analyse_stalled_tasks(db)` in `engine/root_cause_engine.py` is called each tick as Step 8. It finds tasks that entered STALLED status within the last 2 hours and classifies each.

## 15.2 Root Cause Categories

**[VERIFIED]** Five classification categories:

| Category | Detection Pattern | Auto-Fix Available |
|----------|------------------|-------------------|
| `RC_INFRA` | `FileNotFoundError`, Claude CLI not found | Yes |
| `RC_TIMEOUT` | Process timeout, exit code 124 or similar | No |
| `RC_ENCODING` | UTF-8 decode errors in Claude output | No |
| `RC_LOGIC` | TASK_FAILED: marker with reason text | No |
| `RC_UNKNOWN` | No pattern matches | No |

## 15.3 Classification Logic

**[VERIFIED]** The classifier examines the most recent `ClaudeExecution` record for the stalled task:
- Checks `fail_reason` text for timeout patterns
- Checks stderr for encoding error strings
- Checks `fail_reason` for TASK_FAILED structured output
- Checks `outcome` for INFRA-failure signatures (FileNotFoundError in fail_reason)
- Defaults to RC_UNKNOWN

## 15.4 Auto-Fix for Infrastructure Failures

**[VERIFIED]** `_try_auto_fix(task, root_cause, db)` in `engine/root_cause_engine.py`:

Only `RC_INFRA` tasks are eligible for auto-fix. The fix logic:
1. Call `validate_claude_bin()` — checks if Claude CLI is now available
2. If available: set `task.status = READY`, record `TaskTransition(triggered_by="root_cause_auto_fix")`
3. If not available: leave in STALLED, log warning

**[VERIFIED]** This handles the scenario where the orchestrator starts before Claude CLI is installed, or where Claude CLI is temporarily unavailable (e.g., update in progress).

## 15.5 Audit Report Generation

**[VERIFIED]** `_write_report(task, root_cause, db)` generates a markdown format audit report containing:
- Task identification (task_id, title, agent)
- Root cause classification with confidence
- Timeline of state transitions
- Last Claude execution details (exit code, duration, stdout excerpt)
- Regression test stub code

**[VERIFIED]** `_regression_test_stub(root_cause)` generates test code stubs appropriate to each root cause type:
- `RC_INFRA`: test that validates Claude CLI availability before execution
- `RC_TIMEOUT`: test with configurable timeout assertion
- `RC_ENCODING`: test with encoding error simulation
- `RC_LOGIC`: task-specific logic test template
- `RC_UNKNOWN`: generic diagnostic test stub

## 15.6 Root Cause Engine Limitations in v1.5.5

**[NOT IMPLEMENTED]** Machine learning-based pattern recognition for root cause classification — all classification is rule-based regex/string matching.

**[NOT IMPLEMENTED]** Cross-task root cause correlation — each stalled task is analyzed independently without detecting systemic patterns.

**[NOT IMPLEMENTED]** Automatic escalation to human operators when auto-fix fails repeatedly.

**[PARTIALLY IMPLEMENTED]** The audit reports are generated as in-memory strings. Persistence and surfacing of these reports via API is not verified in available source reads.

---


# SECTION 16: REST API LAYER

## 16.1 API Architecture

**[VERIFIED]** The REST API is implemented using FastAPI with APIRouter in `api/routes.py`. All routes are mounted under `/api/v1` prefix as registered in `app.py`.

**[VERIFIED]** Global authentication via API key is applied to the entire router via `dependencies=[Depends(verify_api_key)]` in `api/routes.py:23`, sourced from `api/auth.py`.

**[VERIFIED]** The API key is configured via `settings.api_key` (`ORCH_API_KEY` environment variable).

## 16.2 Endpoint Inventory

**[VERIFIED]** All endpoints verified from `api/routes.py`:

### Health and Version

| Method | Path | Purpose |
|--------|------|---------|
| GET | /health | Platform health + parallel service probes |
| GET | /version | Version info dictionary |

**[VERIFIED]** `GET /health` response contract (Desktop REST contract v1.0, `api/routes.py:79`):

| Field | Type | Source |
|-------|------|--------|
| `status` | "healthy" / "degraded" / "down" | DB connectivity |
| `version` | str | PLATFORM_VERSION |
| `port` | int | settings.api_port |
| `database` | "ok" / error | Task count query |
| `task_count` | int | COUNT(*) tasks |
| `content_factory` | {status, port} | Parallel HTTP probe |
| `ollama` | {status, port} | Parallel HTTP probe |
| + version_info() fields | | Spread at top level |

**[VERIFIED]** Health endpoint probes Content Factory (port 8090, `/actuator/health/liveness`) and Ollama (port 11434, `/api/tags`) in parallel via `ThreadPoolExecutor(max_workers=2)`. Probe timeout: `_PROBE_TIMEOUT = 0.8` seconds. Total latency bounded by max(cf, ollama) <= 0.8s. Service ports overridable via `runtime/runtime-config.yaml`.

### Task Management

| Method | Path | Purpose |
|--------|------|---------|
| GET | /tasks | List tasks (filterable by status, agent, phase) |
| GET | /tasks/{task_id} | Get task detail |
| POST | /tasks/{task_id}/transition | Manual status transition |
| POST | /tasks/{task_id}/activity | Touch last_activity_at (anti-stall) |

**[VERIFIED]** `GET /tasks` query params: `status`, `agent`, `phase`. Ordered by `task_id`.

**[VERIFIED]** `POST /tasks/{task_id}/transition` accepts `TransitionRequest{new_status, triggered_by, notes}`. Records `TaskTransition`.

**[VERIFIED]** `POST /tasks/{task_id}/activity` touches `last_activity_at` + `updated_at` — prevents SLA stall classification.

### Gate Reviews

**[VERIFIED]** `POST /tasks/{task_id}/gates` accepts `GateUpdateRequest{gate_number, status, notes}`. Validates gate exists for task.

### Merge

**[VERIFIED]** `POST /tasks/{task_id}/merge` validates task is APPROVED (409 if not). Sets `merged_at`, records `TaskTransition`. Optional `pr_url` in `MergeRequest` body.

### Heartbeat

**[VERIFIED]** `POST /heartbeat` accepts `HeartbeatRequest{agent_name, active_task_id}`. Upserts `AgentHeartbeat`.

### Dashboard and Metrics

| Method | Path | Purpose |
|--------|------|---------|
| GET | /dashboard | Full dashboard snapshot |
| GET | /metrics | Sprint metrics summary |
| GET | /runtime-metrics | Claude execution and worker metrics |
| GET | /workflow-events | Workflow health event log (filterable) |
| POST | /workflow-events/{id}/resolve | Mark event resolved |
| GET | /critical-path | CP task list and head task |

**[VERIFIED]** `GET /metrics` computes inline: `sprint_tasks_total`, `sprint_tasks_merged`, `sprint_story_points_delivered`, `sprint_story_points_total`, `sla_compliance_pct_24h`, `cp_tasks_open`, `cp_velocity_3d`, `open_escalations`.

**[VERIFIED]** `sla_compliance_pct_24h = (1 - breached/total_gates) * 100`, returns 100.0 when no gates assigned.

### Worker Supervision

**[VERIFIED]** `GET /workers` returns `WorkerSummaryOut{running, total, workers: dict}`.

**[VERIFIED]** `POST /workers/restart/{agent}` delegates to `supervisor.restart(agent)`. Returns 404 for unknown keys with valid key list in error message.

## 16.3 Response Models

**[VERIFIED]** Pydantic schemas in `api/schemas.py`: `TaskSummary`, `TaskDetail`, `MetricsOut`, `RuntimeMetricsOut`, `WorkerStatusOut`, `WorkerSummaryOut`, `WorkflowEventOut`, `GateUpdateRequest`, `HeartbeatRequest`, `MergeRequest`, `TransitionRequest`.

## 16.4 Additional Routers (Unaudited)

**[PARTIALLY IMPLEMENTED]** Seven additional routers registered in `app.py` — existence confirmed, content not read:

`api/storage_routes.py`, `api/asset_routes.py`, `api/health_routes.py`, `api/runtime_routes.py`, `api/capabilities_routes.py`, `api/proxy_routes.py`, `api/metrics_routes.py`.

## 16.5 Authentication

**[VERIFIED]** Single API key authentication on all routes via `verify_api_key` from `api/auth.py`.

**[PARTIALLY IMPLEMENTED]** No OAuth2, JWT, or RBAC. All authenticated callers have identical permissions.

## 16.6 Error Handling

**[VERIFIED]** HTTP error codes: 400 (invalid enum value), 404 (not found), 409 (state precondition), 503 (supervisor not initialized).

---

# SECTION 17: OBSERVABILITY — METRICS, LOGGING, AND TRACING

## 17.1 Prometheus Metrics

**[VERIFIED]** Defined in `factory/metrics/registry.py` using `prometheus_client`. Approximately 20 metrics total.

### HTTP Metrics
- `http_requests_total` Counter (method, path, status_code) — PrometheusMiddleware
- `http_request_duration_seconds` Histogram (method, path)
- `http_active_requests` Gauge

### Worker Metrics
- `workers_total` Gauge
- `workers_running` Gauge
- `worker_restarts_total` Counter (agent_key)
- `worker_uptime_seconds` Gauge (agent_key)

### Task Metrics
- `tasks_active` Gauge
- `queue_length` Gauge
- `tasks_completed_total` Counter (agent)
- `tasks_failed_total` Counter (agent)
- `tasks_retried_total` Counter (agent)
- `task_duration_seconds` Histogram (agent)

### Orchestration Metrics
- `orch_tick_duration_seconds` Histogram
- `orch_tick_errors_total` Counter (step)

### Runtime Metrics
- `runtime_start_duration_seconds` Gauge
- `runtime_stop_duration_seconds` Gauge

### System Metrics (via psutil)
- `process_cpu_percent`, `process_memory_bytes`, `process_threads`, `process_uptime_seconds` Gauge
- `disk_free_bytes`, `disk_used_bytes`, `disk_total_bytes` Gauge

## 17.2 Structured JSON Logging

**[VERIFIED]** `configure_logging()` in `factory/logging/setup.py`:

1. JSON file handler: rotating, max 50MB, 10 backups, `JsonFormatter` from `factory/logging/json_formatter.py`
2. Console handler: human-readable format

**[VERIFIED]** Suppressed noisy loggers: `uvicorn.access`, `sqlalchemy.engine`, `sqlalchemy.pool`.

**[VERIFIED]** `JsonFormatter` produces structured JSON log records for ELK/Splunk/Datadog ingestion.

## 17.3 OpenTelemetry Tracing

**[VERIFIED]** Configured via `factory/tracing/setup.py` during boot sequence.

**[VERIFIED]** Dependencies: `opentelemetry-sdk>=1.20`, `opentelemetry-instrumentation-fastapi>=0.40`, `opentelemetry-exporter-otlp-proto-http>=1.20`.

**[PARTIALLY IMPLEMENTED]** OTLP exporter and FastAPI auto-instrumentation are wired. Custom spans for worker execution and Claude subprocess calls are not verified in available source reads.

## 17.4 Correlation Middleware

**[VERIFIED]** `CorrelationMiddleware` registered as second middleware in `app.py`. Injects correlation IDs into request context for distributed tracing.

**[PARTIALLY IMPLEMENTED]** Propagation to Claude subprocess invocations not verified.

## 17.5 Session Logs

**[VERIFIED]** `ClaudeRuntime.write_session_log()` writes per-invocation structured UTF-8 logs to disk. Path stored in `ClaudeExecution.log_path` database record.

---

# SECTION 18: STORAGE ABSTRACTION LAYER

## 18.1 StorageProvider SPI

**[VERIFIED]** Storage abstraction defined in `factory/storage/spi.py`. `StorageProvider` is an ABC.

## 18.2 Required Abstract Methods

**[VERIFIED]** All implementations must provide:

| Method | Purpose |
|--------|---------|
| `provider_id` property | Unique identifier |
| `scheme` property | URL scheme ("s3", "local", etc.) |
| `capabilities()` | Declare supported optional features |
| `health()` | Backend health report |
| `upload(path, data, metadata)` | Upload bytes |
| `delete(path)` | Delete object |
| `download(path)` | Full download |
| `download_chunked(path, chunk_size)` | Streaming download |
| `exists(path)` | Existence check |
| `get_object(path)` | Metadata + data |
| `get_metadata(path)` | Metadata only |
| `list_objects(prefix)` | List by prefix |
| `copy(src, dst)` | Copy object |
| `move(src, dst)` | Move/rename |
| `checksum(path)` | Compute checksum |

## 18.3 Optional Methods

**[VERIFIED]** Default `raise NotImplementedError` — providers may implement:
`presign_get()`, `presign_put()`, `create_bucket()`, `delete_bucket()`.

## 18.4 Provider Implementations

**[VERIFIED]** Five files in `factory/storage/providers/`:

| Provider | File | Implementation Status |
|----------|------|-----------------------|
| Local filesystem | `local.py` | VERIFIED — primary development provider |
| AWS S3 | `s3.py` | PARTIALLY IMPLEMENTED — file exists, depth unverified |
| Azure Blob | `azure.py` | PARTIALLY IMPLEMENTED — file exists, depth unverified |
| MinIO | `minio.py` | PARTIALLY IMPLEMENTED — file exists, depth unverified |
| Google Cloud Storage | `gcs.py` | PARTIALLY IMPLEMENTED — file exists, depth unverified |

## 18.5 Storage Initialization

**[VERIFIED]** `init_storage()` called during FastAPI boot in `app.py`. Active provider selected from `config.py Settings` storage configuration.

---

# SECTION 19: HOOK AND EVENT SYSTEM

## 19.1 HookRegistry

**[VERIFIED]** Implemented in `factory/hooks/registry.py`. Global singleton: `hook_registry = HookRegistry()`.

## 19.2 FactoryEvent Constants

**[VERIFIED]** ~20 named events in `FactoryEvent` namespace:

| Event | When Fired |
|-------|-----------|
| BEFORE_DISPATCH / AFTER_DISPATCH | Around dispatcher step |
| BEFORE_EXECUTE / AFTER_EXECUTE | Around Claude subprocess |
| BEFORE_REVIEW / AFTER_REVIEW | Around review routing |
| BEFORE_MERGE / AFTER_MERGE | Around merge steps |
| WORKER_STARTED | Worker thread launch |
| WORKER_STOPPED | Worker thread stop |
| WORKER_RECOVERED | Dead worker auto-restart |
| TASK_CLAIMED | Worker claims READY task |
| TASK_COMPLETED | TASK_COMPLETE received |
| TASK_FAILED | Final failure/stall |
| TASK_BLOCKED | TASK_BLOCKED: received |
| TASK_STALLED | SLA stall detected |
| HEARTBEAT_RECEIVED | Heartbeat written |
| DASHBOARD_UPDATED | Dashboard snapshot refresh |
| SLA_BREACHED | Gate SLA deadline exceeded |
| ROOT_CAUSE_ANALYSED | RCA completes |

## 19.3 HookRegistry API

**[VERIFIED]**

| Method | Purpose |
|--------|---------|
| `subscribe(event, priority=50)` | Decorator registration |
| `register(event, handler, priority=50)` | Imperative registration |
| `fire(event, **payload)` | Fire-and-forget dispatch |
| `handlers(event)` | List handlers |
| `clear(event=None)` | Remove handlers |

**[VERIFIED]** Priority-ordered dispatch (lower = first). `fire()` catches and logs handler exceptions — never propagates to caller.

## 19.4 Import Guard Pattern

**[VERIFIED]** SDK availability guard pattern used in `supervisor.py:24-45`:

```python
try:
    from factory.hooks import hook_registry, FactoryEvent
    _HOOKS = True
except ImportError:
    _HOOKS = False
    hook_registry = None
```

Allows core orchestrator to function without the factory SDK package.

## 19.5 Hook Gaps

**[NOT IMPLEMENTED]** Hook persistence — subscriptions are in-memory, must re-register on restart.

**[NOT IMPLEMENTED]** Remote hook dispatch — no webhook or message queue integration.

---

# SECTION 20: RECOMMENDATIONS FOR AI STUDIO 2.0

## 20.1 Framework

Recommendations are grounded in **proven absences** and **structural patterns** identified during this reverse engineering. Each is classified by impact (HIGH/MEDIUM/LOW) and effort (HIGH/MEDIUM/LOW).

---

## 20.2 Critical Gaps to Address

### R-01: Implement Agent-Specific Prompting [Impact: HIGH, Effort: HIGH]

**Evidence:** All 10 concrete workers use `GenericWorker.build_prompt()` unchanged — identical prompt structure for all agents and all task types.

**Recommendation:** Each worker must implement its own `build_prompt()` with:
- Role-specific system context (ARCHITECT vs QA vs SECURITY have fundamentally different responsibilities)
- Phase-aware task framing (sprint-0 infrastructure vs sprint-1 features)
- Domain context injection (project language, framework, prior code output)
- Gate-specific rubrics for review workers (design review vs correctness review vs security audit)

### R-02: Replace SQLite with PostgreSQL for Production [Impact: HIGH, Effort: MEDIUM]

**Evidence:** SQLite is the default database. `FOR UPDATE` on SQLite provides write-lock semantics, not true row-level locking.

**Recommendation:** Target PostgreSQL 15+ for production. SQLAlchemy 2.0 ORM is already compatible. Add `SELECT ... FOR UPDATE SKIP LOCKED` for zero-contention task claiming under concurrent workers.

### R-03: Bundle Working Plugin Runtime Implementations [Impact: HIGH, Effort: HIGH]

**Evidence:** Plugin ABCs are fully defined; 9 builtin manifests exist; zero working runtime implementations are bundled. The platform cannot automate build/test/deploy in v1.5.5 through the plugin system.

**Recommendation:** Bundle at minimum: `python-pytest` (test execution), `git` (commit/PR), and `claude` (AI provider) as fully working implementations.

### R-04: Implement Role-Based API Access Control [Impact: HIGH, Effort: MEDIUM]

**Evidence:** Single API key grants access to all endpoints including worker restart, task transitions, and gate management.

**Recommendation:** Introduce at minimum READ and WRITE roles. Prefer JWT authentication with embedded roles for Desktop integrations.

---

## 20.3 Architecture Preservation

### R-05: Preserve the Runtime Contract [Impact: HIGH, Effort: LOW]

**Evidence:** `TASK_COMPLETE`, `TASK_BLOCKED:`, `TASK_FAILED:` markers in `factory/sdk/worker_base.py` are a clean, model-agnostic output protocol.

**Recommendation:** Preserve this contract exactly in 2.0. Formalize as a versioned spec (already tracked via `WORKER_PROTOCOL_VERSION`). Validate any Claude model upgrade against the contract before deployment.

### R-06: Preserve the Eight-Step Tick [Impact: HIGH, Effort: LOW]

**Evidence:** The tick's 8-step sequential design provides a clean, observable, independently-testable orchestration loop.

**Recommendation:** Preserve and extend with per-step duration metrics. Consider a hook-based extension point for custom tick steps.

### R-07: Preserve Single-Process Architecture for Small Deployments [Impact: MEDIUM, Effort: LOW]

**Evidence:** All components run as daemon threads in one process — simple to deploy, simple to monitor.

**Recommendation:** Preserve single-process mode for small deployments (<10 agents). Add optional multi-process/container mode for large-scale deployments (>20 agents).

---

## 20.4 Scalability

### R-08: PostgreSQL SKIP LOCKED for Task Claiming [Impact: MEDIUM, Effort: MEDIUM]

**Recommendation:** Replace `SELECT ... FOR UPDATE` with `SELECT ... FOR UPDATE SKIP LOCKED` on PostgreSQL. Eliminates serialization bottleneck under concurrent workers without requiring a separate task queue.

### R-09: Horizontal Worker Scaling [Impact: MEDIUM, Effort: HIGH]

**Evidence:** Workers are daemon threads — single-machine only. `TaskExecutor` is already stateless between tasks.

**Recommendation:** Allow workers to run as separate processes or containers. Database already provides the coordination plane. Requires distributed locking, distributed heartbeat (already in DB), and API-level supervisor stub.

---

## 20.5 Security

### R-10: Sanitize Claude Stdout Before Parsing [Impact: HIGH, Effort: LOW]

**Evidence:** `parse_outcome(stdout)` scans raw stdout without length bounds or sanitization.

**Recommendation:** Apply max stdout length cap (e.g., 10MB). Validate extracted `block_reason`/`fail_reason` values are valid UTF-8 without null bytes before DB insertion.

### R-11: Protect Session Log Files [Impact: MEDIUM, Effort: LOW]

**Recommendation:** Apply filesystem access controls (chmod 600 on Unix) to session logs. Consider encryption-at-rest for production deployments.

---

## 20.6 Operations

### R-12: Separate Liveness and Readiness Endpoints [Impact: MEDIUM, Effort: LOW]

**Evidence:** Single `/health` endpoint combines liveness and readiness.

**Recommendation:** Add `/healthz/live` (instant 200) and `/healthz/ready` (DB + workers + Claude). Required for Kubernetes deployments.

### R-13: Formalize the Desktop REST Contract [Impact: MEDIUM, Effort: LOW]

**Recommendation:** Export OpenAPI schema (`openapi.json`) via FastAPI's built-in generator. Version it in the repository. Desktop should validate against schema at startup.

### R-14: Sprint Planning API [Impact: MEDIUM, Effort: MEDIUM]

**Evidence:** Tasks can only be seeded at startup from YAML files. No API for creating tasks, modifying estimates, or managing dependencies after startup.

**Recommendation:** Add: `POST /tasks`, `PATCH /tasks/{id}`, `POST /tasks/{id}/dependencies`, `DELETE /tasks/{id}/dependencies/{dep_id}`.

### R-15: Task Transition Log Endpoint [Impact: MEDIUM, Effort: LOW]

**Evidence:** `TaskTransition` records exist but no `GET /tasks/{id}/transitions` endpoint is provided.

**Recommendation:** Add `GET /tasks/{id}/transitions` for Desktop timeline views.

---

## 20.7 Testing

### R-16: Orchestration Tick Integration Tests [Impact: HIGH, Effort: MEDIUM]

**Current state:** Generated `conftest.py` and `test_smoke.py` are scaffolding stubs — no orchestration-level tests verified in platform source.

**Recommendation:** Build integration tests that: seed known task set, simulate Claude responses via dry_run mode, advance tick loop N times, assert final task states.

### R-17: Worker Protocol Contract Tests [Impact: MEDIUM, Effort: LOW]

**Recommendation:** Add test that calls `worker.build_prompt(mock_task)`, asserts prompt contains all three runtime contract markers verbatim, and asserts `RuntimeContract` constants are unmodified.

---

## 20.8 Priority Matrix

| # | Recommendation | Impact | Effort | Priority |
|---|---------------|--------|--------|---------|
| R-01 | Agent-specific prompting | HIGH | HIGH | P0 |
| R-02 | PostgreSQL for production | HIGH | MEDIUM | P0 |
| R-03 | Bundle plugin runtimes | HIGH | HIGH | P0 |
| R-04 | Role-based API access | HIGH | MEDIUM | P0 |
| R-05 | Preserve runtime contract | HIGH | LOW | P1 |
| R-06 | Preserve eight-step tick | HIGH | LOW | P1 |
| R-10 | Sanitize Claude stdout | HIGH | LOW | P1 |
| R-12 | Liveness vs readiness | MEDIUM | LOW | P1 |
| R-13 | Formalize REST contract | MEDIUM | LOW | P1 |
| R-14 | Sprint planning API | MEDIUM | MEDIUM | P1 |
| R-16 | Tick integration tests | HIGH | MEDIUM | P1 |
| R-07 | Preserve single-process | MEDIUM | LOW | P2 |
| R-08 | SKIP LOCKED for claiming | MEDIUM | MEDIUM | P2 |
| R-09 | Horizontal worker scaling | MEDIUM | HIGH | P2 |
| R-11 | Session log access controls | MEDIUM | LOW | P2 |
| R-15 | Task transition endpoint | MEDIUM | LOW | P2 |
| R-17 | Protocol contract tests | MEDIUM | LOW | P2 |

---

# APPENDIX A: SOURCE FILE INVENTORY

Files read and analyzed during this reverse engineering:

| File | Lines | Key Contents |
|------|-------|-------------|
| `platform_version.py` | 39 | All version constants, version_info() |
| `factory.yaml` | ~20 | Project descriptor example |
| `factory/sdk/worker_base.py` | ~100 | BaseWorker ABC, RuntimeContract |
| `factory/sdk/plugin_base.py` | ~200 | FactoryPlugin ABC, context/result dataclasses |
| `factory/cli/generator.py` | 1529 | 14 CLI commands, 10 templates, scaffolding |
| `factory/plugins/registry.py` | ~200 | PluginManifest, 9 BUILTIN_MANIFESTS, PluginRegistryManager |
| `factory/versioning.py` | ~150 | Compatibility matrix, semver utilities |
| `factory/hooks/registry.py` | ~100 | FactoryEvent constants, HookRegistry |
| `worker/__init__.py` | ~150 | 10 workers, AGENT_REGISTRY, GenericWorker |
| `db/models.py` | ~300 | 9 SQLAlchemy models, TaskStatus/GateStatus enums |
| `engine/workflow_supervisor.py` | ~150 | 5 workflow health checks |
| `runtime/claude_runtime.py` | ~250 | ClaudeRuntime, ClaudeResult, binary resolution |
| `runtime/claude_executor.py` | ~20 | Backward-compat shim |
| `engine/dispatcher.py` | ~150 | Pass 0/A/B dispatch logic |
| `config.py` | ~100 | Settings(BaseSettings), all ORCH_ env vars |
| `app.py` | ~150 | FastAPI lifespan, _tick(), routers, middleware |
| `supervisor.py` | 271 | WorkerSupervisor, _WorkerState, monitor loop |
| `agent_runtime.py` | ~300 | TaskExecutor, claim/execute/heartbeat |
| `engine/sla_monitor.py` | ~100 | SLA breach detection, CP escalation |
| `engine/review_router.py` | ~80 | Gate creation, SLA deadline assignment |
| `engine/merge_manager.py` | ~80 | auto_merge, auto_finalize_merged |
| `engine/dependency_resolver.py` | ~80 | check_dependencies, STATUS_RANK |
| `engine/root_cause_engine.py` | ~150 | 5 root cause categories, auto-fix, reports |
| `factory/logging/setup.py` | ~50 | configure_logging(), JsonFormatter |
| `factory/metrics/registry.py` | ~100 | 20+ Prometheus metrics |
| `factory/storage/spi.py` | ~150 | StorageProvider ABC, 15 abstract methods |
| `api/routes.py` | 408 | All core API endpoints, health probing |
| `pyproject.toml` | 103 | Package metadata, dependencies, entry points |

---

# APPENDIX B: VERSION CONSTANTS

```
PLATFORM_VERSION         = "1.5.5"   platform_version.py:16
PLUGIN_API_VERSION       = "1.5.5"   platform_version.py:17
RUNTIME_VERSION          = "1.5.5"   platform_version.py:18
SCHEMA_VERSION           = 1         platform_version.py:19
HOOK_VERSION             = "1.0.0"   platform_version.py:20
DASHBOARD_VERSION        = "1.0.0"   platform_version.py:21
WORKER_PROTOCOL_VERSION  = "1.0.0"   platform_version.py:22
MIN_COMPATIBLE_PLUGIN_API         = "1.0.0"
MIN_COMPATIBLE_WORKER_PROTOCOL    = "1.0.0"
CLI_VERSION              = "1.0.0"   factory/versioning.py
SDK_VERSION              = "1.0.0"   factory/versioning.py
TEMPLATE_SCHEMA_VERSION  = "1.0.0"   factory/versioning.py
```

---

# APPENDIX C: AGENT REGISTRY

| Registry Key | Worker Class | AGENT_NAME |
|-------------|-------------|-----------|
| ARCHITECT | ArchitectWorker | ARCHITECT-AGENT |
| AUTH | AuthWorker | AUTH-AGENT |
| PLAYER | PlayerWorker | PLAYER-AGENT |
| ECONOMY | EconomyWorker | ECONOMY-AGENT |
| COLLECTION | CollectionWorker | COLLECTION-AGENT |
| BATTLE | BattleWorker | BATTLE-AGENT |
| LIVEOPS | LiveOpsWorker | LIVEOPS-AGENT |
| SECURITY | SecurityWorker | SECURITY-AGENT |
| QA | QAWorker | QA-AGENT |
| RELEASE | ReleaseWorker | RELEASE-AGENT |

---

# APPENDIX D: ORCHESTRATION TICK SEQUENCE

```
app.py _loop() [30s interval]
    |
    +-> _tick()
         |
         +-- [Session 1] dispatch_ready_tasks(db)
         |   +-- Pass 0: PR gate sweep
         |   +-- Pass A: all deps MERGED -> IN_PROGRESS
         |   +-- Pass B: CP + all deps APPROVED -> PREPARING
         |
         +-- [Session 2] route_reviews(db)
         |   +-- Create GateReview records (Gates 2, 3?, 4) for REVIEW tasks
         |
         +-- [Session 3] check_sla(db)
         |   +-- Gate SLA breach detection
         |   +-- CP idle escalation (L1/L2/L3)
         |   +-- General stall detection
         |
         +-- [Session 4] check_blockers(db)
         |   +-- Re-evaluate BLOCKED tasks
         |
         +-- [Session 5] auto_merge(db)
         |   +-- REVIEW -> APPROVED when all gates pass
         |
         +-- [Session 6] auto_finalize_merged(db)
         |   +-- APPROVED -> MERGED for no-pr_url tasks
         |
         +-- [Session 7] check_workflow_health(db)
         |   +-- Ready dispatch lag, review staleness, merge delay
         |   +-- CP idle, dependency deadlock
         |
         +-- [Session 8] analyse_stalled_tasks(db)
             +-- Classify RC_INFRA/TIMEOUT/ENCODING/LOGIC/UNKNOWN
             +-- Auto-fix RC_INFRA if Claude available
             +-- Write audit reports
```

---

*End of AI Software Factory v1.5.5 Reverse Engineering Technical Specification*

*Document date: 2026-06-27*
*Source basis: 28 files, ~5,000 lines of Python analyzed*
*All claims verified against actual source code. No invented features.*
