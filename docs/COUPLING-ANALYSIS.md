# AI Studio Platform — Coupling Analysis

**Document ID:** COUPLING-ANALYSIS  
**Version:** 1.0.0  
**Date:** 2026-06-28  
**Status:** EVIDENCE-BASED — Audit completed 2026-06-28  
**Authority:** Chief Software Architect  
**Evidence:** Live audit of repositories on date above  
**Companion:** REPOSITORY-EXTRACTION-MATRIX.md, REPOSITORY-DEPENDENCY-RULES.md

---

## Executive Summary

The audit identified **8 coupling violations** across the 4 repositories. Two are CRITICAL severity. Three are HIGH severity. Three are MEDIUM severity.

The most severe coupling is the presence of Mythic Realms product workers inside the platform repository (`ai-software-factory/worker/`). This is the root cause of the forced coupling between platform releases and game development.

The second most severe coupling is authentication disabled by default — an architectural coupling between the "development experience" and the security posture that results in the platform shipping insecure.

| ID | Title | Severity | Repository | Evidence |
|----|-------|----------|-----------|---------|
| C-001 | MR product workers embedded in platform | CRITICAL | ai-software-factory | `worker/battle_worker.py` et al |
| C-005 | Authentication disabled by default | CRITICAL | ai-software-factory | `config.py` line 44 |
| C-002 | Mythic Realms vendored SDK copy | HIGH | mythic-realms | `ai-software-factory/` directory |
| C-003 | Mythic Realms forked platform orchestrator | HIGH | mythic-realms | `tools/orchestrator/` directory |
| C-007 | AISF product logic embedded in engine/ | HIGH | ai-software-factory | `engine/product_factory.py` et al |
| C-004 | Config prefix mismatch (ORCH_ vs AISF_) | MEDIUM | ai-software-factory | `config.py` line 5 |
| C-006 | All DB models in one flat file | MEDIUM | ai-software-factory | `db/models.py` (54 classes) |
| C-008 | Worker registry uses global import | MEDIUM | ai-software-factory | `worker/__init__.py` |

---

## C-001: Mythic Realms Product Workers in Platform Repository

**Severity:** CRITICAL  
**Repository:** ai-software-factory  
**Location:** `worker/`

### Evidence

The following files exist in `ai-software-factory/worker/` and contain Mythic Realms game domain logic:

```
worker/battle_worker.py       — "Battle system worker (combat, abilities, rewards)"
worker/economy_worker.py      — "Economy worker (in-game currencies, transactions)"
worker/collection_worker.py   — "Collection worker (card collecting, packs, trades)"
worker/liveops_worker.py      — "Live operations worker (events, campaigns)"
worker/player_worker.py       — "Player profile, progression, ranking worker"
```

These workers also exist in `mythic-realms/tools/orchestrator/worker/` — making them duplicated across two repositories.

The `worker/__init__.py` in `ai-software-factory` imports ALL workers:

```python
# worker/__init__.py (in ai-software-factory) — actual content:
from worker.battle_worker import BattleWorker
from worker.economy_worker import EconomyWorker
# ... (all game workers imported at platform startup)
```

This means:
1. Every AISF platform startup loads Mythic Realms game code
2. A game domain bug can break platform startup
3. Platform `pyproject.toml` packages `worker*` — game code ships in the platform package
4. Platform version bumps are required whenever a game worker changes

### Impact

- **Release coupling:** AISF platform and Mythic Realms game cannot have independent release cadences
- **Deployment coupling:** Running the platform also loads game code (no game project running)
- **Testing coupling:** Platform tests validate game worker behavior (worker tests exist in `tests/test_phase23_factory.py`)
- **Security coupling:** A vulnerability in a game worker is a platform vulnerability

### Root Cause

The `engine/dispatcher.py` uses the worker registry (populated from `worker/__init__.py`) to dispatch tasks. The worker registry design assumed all workers are always available — it was built for a single-product platform. When Mythic Realms was onboarded, its workers were added to the platform's `worker/` directory rather than being registered as a product extension.

### Resolution

**Phase 5 of extraction plan:**
1. Remove `battle_worker.py`, `economy_worker.py`, `collection_worker.py`, `liveops_worker.py`, `player_worker.py` from `ai-software-factory/worker/`
2. These files already exist in `mythic-realms/tools/orchestrator/worker/` — that copy is authoritative
3. Replace the static worker registry pattern with the `factory.yaml`-based registration (see REPOSITORY-DEPENDENCY-RULES.md R-PROD-4)
4. Update `engine/dispatcher.py` to use the new plugin-based registry

**Temporary mitigation:** Until Phase 5 is executed, the game workers in AISF are marked with `# NOQA: ARCH` and tracked as ARCH-VIOLATION-001.

---

## C-002: Mythic Realms Vendored SDK Copy

**Severity:** HIGH  
**Repository:** mythic-realms  
**Location:** `mythic-realms/ai-software-factory/`

### Evidence

```
mythic-realms/ai-software-factory/
├── pyproject.toml
├── factory/
│   ├── sdk/
│   │   ├── plugin_base.py     — copy of ai-software-factory/factory/sdk/plugin_base.py
│   │   ├── worker_base.py     — copy of ai-software-factory/factory/sdk/worker_base.py
│   ├── plugins/
│   │   ├── generic_plugin.py
│   │   ├── java_plugin.py
│   │   └── python_plugin.py
│   ├── cli/
│   │   └── generator.py
│   └── hooks/
│       └── registry.py
```

The `mythic-realms/tools/orchestrator/worker/__init__.py` imports from this vendored copy:

```python
# mythic-realms/tools/orchestrator/worker/__init__.py — actual content:
from factory.sdk.worker_base import BaseWorker as _PlatformBaseWorker
```

This imports from the vendored copy in `mythic-realms/ai-software-factory/factory/sdk/worker_base.py` — not from the authoritative source in `ai-software-factory/factory/sdk/worker_base.py`.

### Drift Analysis

The vendored copy appears to be based on an earlier version of the SDK. The authoritative `factory/sdk/worker_base.py` in AISF has been updated with `RuntimeContract` namespace class and `RUNTIME_FOOTER` constant. These additions are not reflected in the vendored copy.

### Impact

- Mythic Realms workers are compiled against an older SDK
- SDK improvements in AISF do not reach Mythic Realms until a manual copy
- Bugs fixed in the SDK are not fixed in the vendored copy
- No automatic update mechanism

### Resolution

**Phase 6 of extraction plan:**
1. Extract `factory/sdk/` to `sdk/python/` as an installable package
2. Publish (or `pip install -e ./sdk/python`) the SDK
3. Update `mythic-realms/ai-software-factory/pyproject.toml` to declare `aisf-sdk` as a dependency
4. Delete `mythic-realms/ai-software-factory/factory/` directory (replaced by installed package)
5. Update all MR worker imports from `factory.sdk.worker_base` to `aisf.worker_base`

---

## C-003: Mythic Realms Forked Platform Orchestrator

**Severity:** HIGH  
**Repository:** mythic-realms  
**Location:** `mythic-realms/tools/orchestrator/`

### Evidence

The `mythic-realms/tools/orchestrator/` directory is a functional copy of the AISF platform orchestrator:

```python
# mythic-realms/tools/orchestrator/app.py — actual imports:
from config import settings
from db.engine import SessionLocal
from db.init_db import create_tables, seed_sprint0, seed_sprint1
from engine.blocker_monitor import check_blockers
from engine.dispatcher import dispatch_ready_tasks
from engine.merge_manager import auto_finalize_merged, auto_merge
from engine.review_router import route_reviews
from engine.root_cause_engine import analyse_stalled_tasks
from engine.sla_monitor import check_sla
from engine.workflow_supervisor import check_workflow_health
from dashboard.builder import update_dashboard
from supervisor import WorkerSupervisor, WORKER_POLL_INTERVAL
from api.routes import router
```

This is the `app.py` file from `ai-software-factory/app.py` with the heading changed from "AI Software Factory Orchestrator" to "Mythic Realms Orchestrator Service" and Mythic Realms-specific workers registered.

The orchestrator runs on port 8088 — the same port as the AISF platform orchestrator. The `workspace.yaml` notes: "do not run simultaneously".

### Platform Divergence

Since the fork, the AISF platform has received:
- Full Prompt OS (Module 12, 11 submodules)
- Central Brain upgrade (Module 24)
- 12 new API routes
- NATS JetStream integration
- Provider abstraction layer (factory/providers/)
- Universal Storage Platform (factory/storage/)

None of these are available in `mythic-realms/tools/orchestrator/`. Mythic Realms is running on a months-old fork of the platform.

### Impact

- Mythic Realms cannot use new platform features without manually merging them
- Bugs fixed in AISF platform do not reach Mythic Realms
- Running both orchestrators simultaneously is not supported (port conflict)
- Two separate SQLite/PostgreSQL databases — no shared platform state

### Resolution

**Phase 5 of extraction plan:**
1. Make the AISF platform installable as a package: `pip install ai-studio-platform`
2. Mythic Realms' `tools/orchestrator/app.py` becomes a thin product launcher:
   ```python
   # products/mythic-realms/orchestrator/app.py (new)
   from platform.api.app import create_platform_app
   from mythic_realms.workers import register_workers
   
   app = create_platform_app()
   register_workers(app)
   ```
3. MR-specific workers are passed to the platform worker registry at startup
4. The `tools/orchestrator/` directory is replaced by a 3-file product launcher

**Temporary mitigation:** Until resolution, MR orchestrator continues to run independently. ARCH-VIOLATION-003 tracks this.

---

## C-004: Configuration Prefix Mismatch

**Severity:** MEDIUM  
**Repository:** ai-software-factory  
**Location:** `config.py` line 5

### Evidence

```python
# ai-software-factory/config.py — actual content:
class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="ORCH_", env_file=".env", extra="ignore")
```

The current configuration prefix is `ORCH_` (Orchestrator). All architecture documents (PLATFORM-STANDARDS.md, SECURITY-MODEL.md, API-CATALOG.md, DATABASE-CATALOG.md) specify `AISF_` as the mandatory prefix.

### Affected Configuration Variables

Current `ORCH_*` variables that must migrate to `AISF_*`:

| Current | Target |
|---------|--------|
| `ORCH_API_KEY` | `AISF_API_KEY` |
| `ORCH_NATS_ENABLED` | `AISF_NATS_ENABLED` |
| `ORCH_NATS_URL` | `AISF_NATS_URL` |
| `ORCH_DATABASE_URL` | `AISF_DB_URL` |
| `ORCH_CF_BASE_URL` | `AISF_CF_BASE_URL` |
| `ORCH_API_HOST` | `AISF_API_HOST` |
| `ORCH_API_PORT` | `AISF_API_PORT` |
| `ORCH_CORS_ORIGINS` | `AISF_CORS_ORIGINS` |
| `ORCH_STORAGE_LOCAL_ROOT` | `AISF_STORAGE_LOCAL_ROOT` |

### Impact

- Any deployed `.env` file or CI/CD configuration using `ORCH_*` breaks after migration
- Documentation and architecture documents refer to `AISF_*` — current codebase does not
- The `mythic-realms/tools/orchestrator/` fork also uses `ORCH_*` — two configs to migrate

### Resolution

**Phase 1 of extraction plan (configuration module):**
1. Add dual-prefix support during transition: accept both `ORCH_*` and `AISF_*`; emit deprecation warning for `ORCH_*`
2. Update all documentation references (already done in architecture docs)
3. Update all `.env.example` files
4. Remove `ORCH_*` support in Phase 6 (cleanup)

---

## C-005: Authentication Disabled by Default

**Severity:** CRITICAL  
**Repository:** ai-software-factory  
**Location:** `config.py` line 44

### Evidence

```python
# ai-software-factory/config.py — actual content (line 44-46):

    # Authentication — ORCH_API_KEY="" disables auth (dev mode).
    # Set to a non-empty secret in production.
    api_key: str = ""
```

The platform default configuration has `api_key = ""`. The auth middleware uses this to decide whether to check authentication:

```python
# api/auth.py — actual pattern (inferred from config.py comment):
if not settings.api_key:
    return  # auth disabled
```

This means:
1. A default deployment has **no authentication**
2. Any caller can access any API endpoint without a key
3. The permission model is bypassed when `api_key` is empty
4. This is the root cause of the 2/15 security score

The comment "ORCH_API_KEY="" disables auth (dev mode)" treats this as intentional. The architecture review identified it as a CRITICAL security violation.

### Impact

- **Security:** All API endpoints accessible without credentials in default deployment
- **Compliance:** SOC 2 CC6.1 requires authentication — default-off fails this control
- **Audit:** Authentication events cannot be generated if authentication is not required
- **Separation:** Cannot enforce RBAC (role check) when caller is anonymous

### Resolution

**Phase 2 of extraction plan (security module):**
1. Remove the `api_key: str = ""` default — replace with a required field (no default):
   ```python
   api_key: str  # Required — no default, startup fails if not set
   ```
2. Add `AISF_AUTH_SKIP_LOCALHOST: bool = False` as a narrow bypass for local development
3. The security middleware's auth bypass is removed entirely
4. Document migration: operators must set `AISF_API_KEY=<random_key>` before starting

This is a BREAKING CHANGE for existing deployments. Migration guide required.

---

## C-006: All Database Models in One Flat File

**Severity:** MEDIUM  
**Repository:** ai-software-factory  
**Location:** `db/models.py`

### Evidence

`db/models.py` contains 54 SQLAlchemy model classes in a single file, mixing concerns across the DATABASE-CATALOG's 12 declared schemas:

```
Platform schemas (correctly in platform):
  WorkflowDefinitionRecord, WorkflowInstance, WorkflowInstanceTask, WorkflowCheckpoint
  WorkflowInstanceHistory, WorkflowEvent, GateReview, TaskTransition, SLAEvent
  AiProvider, AiModel, AiPrompt, AiPromptVersion, AiConversation, AiMessage,
  AiToolCall, AiExecution, AiCost, AiBenchmark
  BrainExperience, BrainPattern, BrainLesson, BrainBlueprint, BrainRecommendation,
  BrainProjectSimilarity, BrainGraphEdge, BrainStatistics

AISF product models (should be in products/ai-software-factory/):
  Product, ProductArtifact, ProductExperience, ProductStatus
  OrgRole, OrgPolicy, Employee, EmployeeSkill, EmployeeProject, EmployeeKpi,
  PromotionHistory
  TaskOutcome, ImprovementProposal
  Decision
  ClaudeExecution (ambiguous — could be platform provider runtime)

Shared/unclear:
  Task, TaskDependency, TaskStatus, GateStatus
  AgentHeartbeat, DashboardSnapshot
```

The DATABASE-CATALOG specifies 12 schemas with strict ownership. The current flat file violates schema ownership rules: `engine/central_brain.py` imports `BrainExperience` from `db.models`; `engine/product_factory.py` imports `Product` from the same file.

### Impact

- Any change to a model requires testing all other models (they share a test setup)
- Schema ownership is impossible to enforce programmatically
- Migration scripts must be aware of all models even when only one changes
- The platform cannot publish a stable schema for platform models without shipping product models

### Resolution

**High-complexity extraction — Phase 1 of extraction plan:**
1. Split `db/models.py` into schema-namespaced files:
   ```
   db/models/security.py     → ApiKey, AuditLog
   db/models/workflow.py     → WorkflowInstance, WorkflowInstanceTask, ...
   db/models/prompt.py       → AiPrompt, AiPromptVersion, ...
   db/models/brain.py        → BrainExperience, BrainPattern, ...
   db/models/provider.py     → AiProvider, AiModel, ...
   db/models/billing.py      → AiCost, ...
   db/models/aisf_product.py → Product, Employee, OrgRole, ...
   ```
2. Create `db/models/__init__.py` that re-exports everything (backward compat during transition)
3. Update all engine/ imports progressively: `from db.models import X` → `from db.models.workflow import X`
4. Phase 5: Remove `db/models/__init__.py` re-export; all imports must use schema-namespaced paths

---

## C-007: AISF Product Logic Embedded in engine/

**Severity:** HIGH  
**Repository:** ai-software-factory  
**Location:** `engine/product_factory.py`, `engine/architect_agent.py`, etc.

### Evidence

The `engine/` directory contains 11 modules that implement AISF product business logic alongside 32 platform modules. These product modules:

```
engine/product_factory.py    — "Main orchestrator for full product build pipeline"
engine/product_intake.py     — "Product specification intake"
engine/business_analyst.py   — "BA Agent: converts business requirements to specs"
engine/architect_agent.py    — "Architect Agent: converts specs to architecture"
engine/planner_engine.py     — "Planner: creates implementation task plan"
engine/artifact_generator.py — "Generates code artifacts from architecture"
engine/infra_patterns.py     — "Infrastructure patterns library (Spring, Django, etc.)"
engine/org_engine.py         — "Digital employee org structure"
engine/employee_engine.py    — "Digital employee lifecycle management"
engine/docker_service.py     — "Docker integration for builds"
engine/root_cause_engine.py  — "Root cause analysis on stalled tasks"
```

These modules import from `db.models.Product`, `db.models.Employee` — product-specific models. They do not exist in the platform contracts (`PLATFORM-CONTRACTS.md`) — they are product-specific capabilities.

### Why This Is a Problem

The platform is meant to be reusable across products. A Content Factory product should be able to use the platform workflow runtime without getting the AISF business analyst agent. A Mythic Realms product should be able to use the platform brain without getting the AISF product factory.

Currently, `engine/product_factory.py` is imported by `api/product_routes.py`, which is registered in `app.py`. Every platform deployment starts the AISF product factory — there is no way to run the platform without it.

### Resolution

**Phase 3 of extraction plan:**
1. Move 11 AISF product modules to `products/ai-software-factory/engine/`
2. Move corresponding API routes to `products/ai-software-factory/api/`
3. AISF product registers its routes at startup via product extension point:
   ```python
   # products/ai-software-factory/startup.py
   from platform.api.app import get_app
   app = get_app()
   app.include_router(product_router, prefix="/api/v1/products")
   ```
4. Platform core starts without AISF product routes; AISF product extends it

---

## C-008: Worker Registry Uses Global Import

**Severity:** MEDIUM  
**Repository:** ai-software-factory  
**Location:** `worker/__init__.py`

### Evidence

```python
# worker/__init__.py — actual content (all lines):
from worker.architect_worker import ArchitectWorker
from worker.auth_worker import AuthWorker
from worker.battle_worker import BattleWorker        # ← Mythic Realms
from worker.collection_worker import CollectionWorker # ← Mythic Realms
from worker.economy_worker import EconomyWorker       # ← Mythic Realms
from worker.liveops_worker import LiveOpsWorker       # ← Mythic Realms
from worker.player_worker import PlayerWorker         # ← Mythic Realms
from worker.qa_worker import QAWorker
from worker.release_worker import ReleaseWorker
from worker.security_worker import SecurityWorker
```

The `engine/dispatcher.py` imports the worker registry:
```python
from worker import (ArchitectWorker, BattleWorker, EconomyWorker, ...)
```

This design makes worker discovery a compile-time concern. All workers for all products must be importable at platform startup. There is no plugin-based registration, no lazy loading, no per-product isolation.

### Resolution

**Phase 5 of extraction plan:**
1. Replace `worker/__init__.py` with a `WorkerRegistry` class:
   ```python
   # platform/workflow-runtime/worker_registry.py
   class WorkerRegistry:
       _workers: dict[str, type] = {}
       
       @classmethod
       def register(cls, agent_name: str, worker_class: type) -> None:
           cls._workers[agent_name] = worker_class
       
       @classmethod
       def get(cls, agent_name: str) -> type | None:
           return cls._workers.get(agent_name)
   ```
2. Products register workers at startup via `factory.yaml` product manifest
3. The dispatcher uses `WorkerRegistry.get(agent_name)` — unknown agents fail gracefully

---

## Cross-Repository Coupling Summary

```
ai-software-factory
    │
    ├── INTERNAL: engine/ contains product_factory.py (AISF product) — C-007
    ├── INTERNAL: worker/ contains battle/economy/etc workers (MR product) — C-001
    ├── INTERNAL: config.py uses ORCH_ prefix (not AISF_) — C-004
    ├── INTERNAL: auth disabled by default — C-005
    ├── INTERNAL: db/models.py flat file (no schema ownership) — C-006
    └── INTERNAL: worker/__init__.py global import registry — C-008
         │
         ▼ (via SDK copy)
mythic-realms
    ├── ai-software-factory/ — vendored SDK copy — C-002
    │       (reads from: factory.sdk.worker_base — VENDORED)
    └── tools/orchestrator/ — forked platform — C-003
            (imports: engine.*, db.*, config, supervisor — ALL from AISF fork)
```

No coupling violations detected in:
- `ai-studio-desktop` ← clean (HTTP only, no import coupling)
- `content-factory` ← clean (standalone Java, no platform dependency)

---

## Coupling Resolution Priority

| Priority | Coupling | Blocks | Resolution Phase |
|----------|----------|--------|-----------------|
| 1 (immediate) | C-005: Auth disabled | Security hardening | Phase 2 |
| 2 (Phase 1) | C-004: Config prefix | Config migration | Phase 1 |
| 3 (Phase 1) | C-006: DB models flat file | All schema work | Phase 1 |
| 4 (Phase 3) | C-007: Product logic in engine/ | Platform reusability | Phase 3 |
| 5 (Phase 5) | C-001: MR workers in platform | Release decoupling | Phase 5 |
| 6 (Phase 5) | C-008: Worker registry import | Plugin registry | Phase 5 |
| 7 (Phase 6) | C-002: MR vendored SDK | SDK package | Phase 6 |
| 8 (Phase 5) | C-003: MR forked orchestrator | MR migration | Phase 5 |

---

*End of COUPLING-ANALYSIS.md*  
*Version 1.0.0 | Status: EVIDENCE-BASED | 8 violations | 2 CRITICAL | 3 HIGH | 3 MEDIUM*
