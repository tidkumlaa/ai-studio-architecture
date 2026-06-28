# AI Studio Platform — Platform Extraction Roadmap

**Document ID:** PLATFORM-EXTRACTION-ROADMAP  
**Version:** 1.0.0  
**Date:** 2026-06-28  
**Status:** PROPOSED  
**Authority:** Chief Software Architect  
**Companion:** REPOSITORY-SEPARATION-PLAN.md, REPOSITORY-EXTRACTION-MATRIX.md

---

## Overview

This roadmap defines the extraction sequence. Six phases, each building on the previous. Each phase has a clear definition of done, a rollback strategy, and a verification gate before proceeding.

**Constraint:** Do not move everything at once. Each phase must leave the platform in a working state. No phase may break existing tests. Each phase ends with a full test run.

---

## Phase 1: Foundation (Configuration, DB Schema, Models Split)

**Target duration:** 5–7 engineer-days  
**Risk level:** MEDIUM  
**Dependencies:** None — Phase 1 is the starting point

### Objective

Extract the foundational configuration module (with prefix migration) and split the monolithic `db/models.py` into schema-namespaced files. This unblocks every other phase because all other modules import from `db.models`.

### Why First

Every module in `engine/`, `api/`, and `factory/` imports from `db.models`. Before any module can move to its target location, the models it imports must be resolvable in their new paths. The model split is the highest-dependency blocker.

### Steps

**Step 1.1 — Configuration module migration**
```
ai-software-factory/config.py
    → platform/configuration/settings.py

Changes required:
  - Rename env_prefix from "ORCH_" to "AISF_"
  - Add backward compatibility: read ORCH_* as fallback, emit DeprecationWarning
  - Remove api_key default="" — make it required with startup-time validation
  - Add AISF_AUTH_SKIP_LOCALHOST: bool = False (local-only dev bypass)
  - Add AISF_ENVIRONMENT: str = "local" setting
  
New import path: from platform.configuration.settings import settings
Backward compat: from config import settings still works (re-export shim)
```

**Step 1.2 — Platform version module**
```
ai-software-factory/platform_version.py
    → platform/configuration/version.py

No import changes required (simple constants file)
```

**Step 1.3 — Database models split**

Split `db/models.py` (54 classes) into schema-namespaced files:

```
db/models.py (current — 54 classes)
    → db/models/__init__.py        (re-exports ALL for backward compat)
    → db/models/base.py            (Base, DeclarativeBase)
    → db/models/workflow.py        (Task, TaskDependency, TaskStatus, GateStatus,
                                    GateReview, TaskTransition, SLAEvent,
                                    WorkflowDefinitionRecord, WorkflowInstance,
                                    WorkflowInstanceTask, WorkflowCheckpoint,
                                    WorkflowInstanceHistory, WorkflowEvent,
                                    AgentHeartbeat, DashboardSnapshot)
    → db/models/prompt.py          (AiPrompt, AiPromptVersion, AiConversation,
                                    AiMessage, AiToolCall)
    → db/models/provider.py        (AiProvider, AiModel, ClaudeExecution)
    → db/models/ai_runtime.py      (AiExecution, AiCost, AiBenchmark)
    → db/models/brain.py           (BrainExperience, BrainPattern, BrainLesson,
                                    BrainBlueprint, BrainRecommendation,
                                    BrainProjectSimilarity, BrainGraphEdge,
                                    BrainStatistics)
    → db/models/aisf_product.py    (Product, ProductArtifact, ProductExperience,
                                    ProductStatus, TaskOutcome, ImprovementProposal,
                                    OrgRole, OrgPolicy, Employee, EmployeeSkill,
                                    EmployeeProject, EmployeeKpi, PromotionHistory,
                                    Decision, Experience, OutcomeStatus,
                                    ProposalType, ProposalStatus)
```

**Step 1.4 — Verify backward compatibility**

`db/models/__init__.py` re-exports everything:
```python
# db/models/__init__.py
from db.models.workflow import *
from db.models.prompt import *
from db.models.provider import *
from db.models.ai_runtime import *
from db.models.brain import *
from db.models.aisf_product import *
```

All existing `from db.models import X` imports continue working. No other module needs updating in Phase 1.

### Phase 1 Definition of Done

- [ ] `config.py` uses `AISF_` prefix; `ORCH_*` triggers DeprecationWarning
- [ ] `api_key` has no default (startup fails if not set, unless `AISF_ENVIRONMENT=local`)
- [ ] `db/models/` directory created with 7 sub-files
- [ ] `db/models/__init__.py` re-exports all 54 classes
- [ ] All 233 existing tests pass
- [ ] `python app.py` starts and health endpoint responds

### Phase 1 Rollback

Restore `config.py` from git. Replace `db/models/` with original `db/models.py`. No other files changed.

---

## Phase 2: Infrastructure Services (Event Bus, Security Middleware)

**Target duration:** 4–5 engineer-days  
**Risk level:** MEDIUM  
**Dependencies:** Phase 1 complete

### Objective

Extract the event bus and security modules to their target platform locations. Fix the CRITICAL authentication vulnerability (C-005).

### Steps

**Step 2.1 — Event bus extraction**
```
engine/event_bus.py   → platform/event-bus/in_process.py
engine/nats_client.py → platform/event-bus/nats_client.py
engine/event_schemas.py → platform/event-bus/schemas.py

Import updates required:
  app.py:        from engine.nats_client → from platform.event_bus.nats_client
  api/ws_routes.py: from engine.event_bus → from platform.event_bus.in_process
  
Backward compat shim: engine/event_bus.py → re-export from platform.event_bus.in_process
```

**Step 2.2 — Security module extraction**
```
engine/security.py → platform/security/rbac.py

Import updates required:
  api/auth.py:         from engine.security → from platform.security.rbac
  api/security_routes.py: same
  
Backward compat shim: engine/security.py → re-export
```

**Step 2.3 — Authentication middleware hardening (C-005 fix)**

This step is paired with Phase 2 because extracting the auth module is the right moment to fix it.

Changes to `api/auth.py` → `platform/security/auth_middleware.py`:
```python
# REMOVE: the auth-disabled path
# OLD (forbidden after this step):
if not settings.api_key:
    return  # auth disabled — REMOVED

# NEW: authentication is always required
# Local development: set AISF_ENVIRONMENT=local AND AISF_AUTH_SKIP_LOCALHOST=true
# This only works for 127.0.0.1 requests in local environment
if settings.environment == "local" and settings.auth_skip_localhost:
    if request.client.host == "127.0.0.1":
        return _anonymous_context()
```

**Step 2.4 — Rate limiter extraction**

The rate limiter in `engine/security.py` (RateLimiter class) moves to `platform/security/rate_limiter.py`.

**Step 2.5 — Metrics and logging extraction**
```
factory/logging/   → platform/metrics/logging/
factory/metrics/   → platform/metrics/prometheus/
factory/tracing/   → platform/metrics/tracing/

app.py calls configure_logging(), init_metrics(), configure_tracing()
  → from platform.metrics.logging import configure_logging
  → from platform.metrics.prometheus import init_metrics
  → from platform.metrics.tracing import configure_tracing
  
Backward compat shims in factory/logging/__init__.py, etc.
```

### Phase 2 Definition of Done

- [ ] `platform/event-bus/` directory exists with 3 modules
- [ ] `platform/security/` directory exists with auth_middleware, rbac, rate_limiter
- [ ] Authentication is required by default (C-005 fixed)
- [ ] `AISF_ENVIRONMENT=local AISF_AUTH_SKIP_LOCALHOST=true` allows anonymous local access
- [ ] Platform starts with a required `AISF_API_KEY` set in environment
- [ ] All 233 existing tests pass (test environment sets `AISF_API_KEY=test-key-local`)
- [ ] `engine/security.py` still importable (re-export shim)

### Phase 2 Rollback

Restore `api/auth.py` from git. This immediately restores auth-optional behavior. Restore `engine/security.py`, `engine/event_bus.py`, `engine/nats_client.py` from git.

---

## Phase 3: Core Platform Services (Workflow, Prompt OS, Providers)

**Target duration:** 8–10 engineer-days  
**Risk level:** HIGH  
**Dependencies:** Phase 1 + Phase 2 complete

### Objective

Extract the three core platform service groups: workflow runtime, Prompt OS, and provider runtime. These are the highest-traffic platform services.

### Steps

**Step 3.1 — Provider runtime extraction**
```
factory/providers/  → platform/provider-runtime/
runtime/claude_runtime.py → platform/provider-runtime/claude_runtime.py
runtime/claude_executor.py → platform/provider-runtime/claude_executor.py
runtime/provider_runtime.py → platform/provider-runtime/runtime.py

Import updates:
  engine/ai_execution.py: from factory.providers → from platform.provider_runtime
  api/ai_execution_routes.py: same
```

**Step 3.2 — Workflow runtime extraction**
```
engine/workflow_engine.py      → platform/workflow-runtime/engine.py
engine/workflow_runtime.py     → platform/workflow-runtime/runtime.py
engine/workflow_supervisor.py  → platform/workflow-runtime/supervisor.py
engine/dependency_resolver.py  → platform/workflow-runtime/dependency_resolver.py
engine/blocker_monitor.py      → platform/workflow-runtime/blocker_monitor.py
engine/dispatcher.py           → platform/workflow-runtime/dispatcher.py (NOTE: see below)
engine/review_router.py        → platform/review-engine/review_router.py
engine/merge_manager.py        → platform/review-engine/merge_manager.py
engine/approval_service.py     → platform/governance/approval_service.py
engine/sla_monitor.py          → platform/observability/sla_monitor.py
runtime/execution_flow.py      → platform/workflow-runtime/execution_flow.py
supervisor.py                  → platform/workflow-runtime/worker_supervisor.py
```

**Step 3.2a — dispatcher.py special handling**

The `engine/dispatcher.py` has a CRITICAL coupling: it imports workers from `worker/__init__.py`. This import must be replaced with the WorkerRegistry plugin pattern BEFORE moving dispatcher.

```python
# BEFORE (coupling):
from worker import ArchitectWorker, BattleWorker, ...  # FORBIDDEN

# AFTER (plugin registry):
from platform.workflow_runtime.worker_registry import WorkerRegistry

def dispatch_ready_tasks(db):
    agent_name = task.agent
    worker_cls = WorkerRegistry.get(agent_name)
    if worker_cls is None:
        log.error(f"No worker registered for agent {agent_name}")
        return
    worker = worker_cls()
    worker.execute(task)
```

This breaks the C-001 and C-008 coupling at the same time. Workers are no longer imported at platform startup.

**Step 3.3 — Prompt OS extraction**
```
engine/prompt_os/           → platform/prompt-os/
engine/context_engine.py    → platform/prompt-os/context_engine.py
engine/conversation_engine.py → platform/prompt-os/conversation_engine.py
engine/prompt_runtime.py    → platform/prompt-os/runtime.py
engine/prompt_intelligence.py → platform/prompt-os/intelligence.py

Import updates:
  api/prompt_os_routes.py: from engine.prompt_os → from platform.prompt_os
  api/prompt_intelligence_routes.py: same
```

**Step 3.4 — Storage extraction**
```
factory/storage/ → platform/storage/
factory/assets/  → platform/storage/assets/
factory/memory/  → platform/central-brain/memory/

Import updates:
  api/storage_routes.py: from factory.storage → from platform.storage
  api/asset_routes.py: same
```

**Step 3.5 — AISF product logic extraction from engine/ (C-007 fix)**

Remove 11 AISF product modules from `engine/` to `products/ai-software-factory/engine/`:
```
engine/product_factory.py    → products/ai-software-factory/engine/product_factory.py
engine/product_intake.py     → products/ai-software-factory/engine/product_intake.py
engine/business_analyst.py   → products/ai-software-factory/engine/business_analyst.py
engine/architect_agent.py    → products/ai-software-factory/engine/architect_agent.py
engine/planner_engine.py     → products/ai-software-factory/engine/planner_engine.py
engine/artifact_generator.py → products/ai-software-factory/engine/artifact_generator.py
engine/infra_patterns.py     → products/ai-software-factory/engine/infra_patterns.py
engine/org_engine.py         → products/ai-software-factory/engine/org_engine.py
engine/employee_engine.py    → products/ai-software-factory/engine/employee_engine.py
engine/docker_service.py     → products/ai-software-factory/engine/docker_service.py
engine/root_cause_engine.py  → products/ai-software-factory/engine/root_cause_engine.py
```

Corresponding API routes also move:
```
api/product_routes.py        → products/ai-software-factory/api/product_routes.py
api/improvement_routes.py    → products/ai-software-factory/api/improvement_routes.py
api/org_routes.py            → products/ai-software-factory/api/org_routes.py
api/employee_routes.py       → products/ai-software-factory/api/employee_routes.py
api/agent_metrics_routes.py  → products/ai-software-factory/api/agent_metrics_routes.py
api/git_routes.py            → products/ai-software-factory/api/git_routes.py
api/docker_routes.py         → products/ai-software-factory/api/docker_routes.py
```

The AISF product registers its routes at startup via extension point:
```python
# products/ai-software-factory/startup.py
from platform.api.app import get_platform_app
from products.ai_software_factory.api import register_routes

app = get_platform_app()
register_routes(app)
```

### Phase 3 Definition of Done

- [ ] `platform/workflow-runtime/` exists with all workflow modules
- [ ] `platform/prompt-os/` exists with all prompt OS modules
- [ ] `platform/provider-runtime/` exists with all provider adapters
- [ ] `platform/storage/` exists
- [ ] `products/ai-software-factory/engine/` exists with 11 AISF modules
- [ ] WorkerRegistry plugin pattern implemented (no worker imports in dispatcher)
- [ ] All 233 existing tests pass
- [ ] Platform starts and all API endpoints respond

### Phase 3 Rollback

Restore all moved files from git. The re-export shims in original locations handle import compatibility. Restore `engine/dispatcher.py` to the version with direct worker imports.

---

## Phase 4: Intelligence Services (Brain, Decision, AI Runtime)

**Target duration:** 6–7 engineer-days  
**Risk level:** MEDIUM  
**Dependencies:** Phase 3 complete

### Objective

Extract the intelligence platform services: Central Brain, Decision Engine, and AI Runtime (Router).

### Steps

**Step 4.1 — Central Brain extraction**
```
engine/central_brain.py         → platform/central-brain/central_brain.py
engine/brain.py                 → platform/central-brain/brain.py
engine/memory_os.py             → platform/central-brain/memory_os.py
engine/experience_recorder.py   → platform/central-brain/experience_recorder.py
engine/outcome_collector.py     → platform/central-brain/outcome_collector.py
engine/learning_agent.py        → platform/central-brain/learning_agent.py
engine/self_improvement.py      → platform/central-brain/self_improvement.py
engine/failure_analyzer.py      → platform/central-brain/failure_analyzer.py
factory/memory/                 → platform/central-brain/memory/
```

**Step 4.2 — Decision Engine extraction**
```
engine/decision_engine.py → platform/decision-engine/engine.py
```

**Step 4.3 — AI Runtime (AI ROS) extraction**
```
engine/ai_router.py      → platform/ai-runtime/router.py
engine/model_registry.py → platform/ai-runtime/model_registry.py
engine/ai_execution.py   → platform/ai-runtime/execution.py
engine/cost_engine.py    → platform/ai-runtime/cost_engine.py
engine/tool_runtime.py   → platform/plugin-runtime/tool_runtime.py
```

**Step 4.4 — Knowledge module (new)**

The knowledge module does not yet exist as a distinct module (knowledge graph lives in `engine/central_brain.py`). Phase 4 extracts it:

```
# Extract from engine/central_brain.py (GraphEngine class):
platform/knowledge/graph.py       — GraphEngine (Kuzu migration target)
platform/knowledge/node_types.py  — Node type definitions
```

### Phase 4 Definition of Done

- [ ] `platform/central-brain/` exists with all 8 modules
- [ ] `platform/decision-engine/` exists
- [ ] `platform/ai-runtime/` exists with 4 modules
- [ ] `platform/knowledge/` exists (extracted from central_brain.py)
- [ ] All 233 existing tests pass

---

## Phase 5: Products and Desktop Reorganization

**Target duration:** 8–10 engineer-days  
**Risk level:** HIGH  
**Dependencies:** Phase 3 + Phase 4 complete

### Objective

Move Mythic Realms workers to the correct repository, remove MR workers from the platform, move the desktop to its target location, and begin migrating the MR forked orchestrator to use the platform as a package.

### Steps

**Step 5.1 — Remove MR workers from platform (C-001 resolution)**

Workers to remove from `ai-software-factory/worker/`:
```
DELETE: worker/battle_worker.py       (already in mythic-realms/tools/orchestrator/worker/)
DELETE: worker/economy_worker.py      (already in mythic-realms/tools/orchestrator/worker/)
DELETE: worker/collection_worker.py   (already in mythic-realms/tools/orchestrator/worker/)
DELETE: worker/liveops_worker.py      (already in mythic-realms/tools/orchestrator/worker/)
DELETE: worker/player_worker.py       (already in mythic-realms/tools/orchestrator/worker/)
```

Prerequisite: WorkerRegistry plugin pattern must be in place (Phase 3 Step 3.2a). Once the dispatcher no longer imports workers statically, removing these files does not break anything.

**Step 5.2 — Move AISF workers to product**
```
worker/architect_worker.py  → products/ai-software-factory/workers/architect_worker.py
worker/auth_worker.py       → products/ai-software-factory/workers/auth_worker.py
worker/qa_worker.py         → products/ai-software-factory/workers/qa_worker.py
worker/security_worker.py   → products/ai-software-factory/workers/security_worker.py
worker/release_worker.py    → products/ai-software-factory/workers/release_worker.py
```

**Step 5.3 — AISF product factory.yaml**

Create worker registry manifest for AISF product:
```yaml
# products/ai-software-factory/factory.yaml
product:
  id: ai-software-factory
  name: "AI Software Factory"
  
workers:
  - agent: ARCHITECT
    class: products.ai_software_factory.workers.architect_worker.ArchitectWorker
  - agent: AUTH
    class: products.ai_software_factory.workers.auth_worker.AuthWorker
  - agent: QA
    class: products.ai_software_factory.workers.qa_worker.QAWorker
  - agent: SECURITY
    class: products.ai_software_factory.workers.security_worker.SecurityWorker
  - agent: RELEASE
    class: products.ai_software_factory.workers.release_worker.ReleaseWorker
```

**Step 5.4 — Mythic Realms factory.yaml**

Create worker registry manifest for Mythic Realms:
```yaml
# products/mythic-realms/factory.yaml
product:
  id: mythic-realms
  name: "Mythic Realms"
  
workers:
  - agent: BATTLE-AGENT
    class: mythic_realms.workers.battle_worker.BattleWorker
  - agent: ECONOMY-AGENT
    class: mythic_realms.workers.economy_worker.EconomyWorker
  - agent: PLAYER-AGENT
    class: mythic_realms.workers.player_worker.PlayerWorker
  - agent: COLLECTION-AGENT
    class: mythic_realms.workers.collection_worker.CollectionWorker
  - agent: LIVEOPS-AGENT
    class: mythic_realms.workers.liveops_worker.LiveOpsWorker
  - agent: ARCHITECT
    class: mythic_realms.workers.architect_worker.ArchitectWorker
  - agent: AUTH
    class: mythic_realms.workers.auth_worker.AuthWorker
  - agent: QA
    class: mythic_realms.workers.qa_worker.QAWorker
  - agent: SECURITY
    class: mythic_realms.workers.security_worker.SecurityWorker
  - agent: RELEASE
    class: mythic_realms.workers.release_worker.ReleaseWorker
```

**Step 5.5 — MR orchestrator migration (C-003 partial)**

Begin migrating `mythic-realms/tools/orchestrator/` from forked platform to platform-as-package:

```python
# products/mythic-realms/orchestrator/app.py (NEW — replaces tools/orchestrator/app.py)
"""Mythic Realms product launcher — thin wrapper over the AI Studio Platform."""
from platform.api.app import create_platform_app
from platform.workflow_runtime.worker_registry import WorkerRegistry

# Register MR product workers
from mythic_realms.workers import (
    BattleWorker, EconomyWorker, PlayerWorker,
    CollectionWorker, LiveOpsWorker, ArchitectWorker,
    AuthWorker, QAWorker, SecurityWorker, ReleaseWorker
)

WorkerRegistry.register("BATTLE-AGENT", BattleWorker)
WorkerRegistry.register("ECONOMY-AGENT", EconomyWorker)
# ... etc.

app = create_platform_app()
```

**Step 5.6 — Desktop move**
```
source/ai-studio-desktop/ → desktop/ai-studio-desktop/

Update:
  services/workspace_service.py: Path(__file__).parents[3] → parents[4]
  workspace.yaml: update path references
```

**Step 5.7 — Content Factory move (organizational)**
```
source/content-factory/ → products/content-factory/
workspace.yaml: update path reference
```

### Phase 5 Definition of Done

- [ ] `worker/battle_worker.py` etc. deleted from platform
- [ ] `products/ai-software-factory/workers/` has 5 AISF workers
- [ ] `products/mythic-realms/factory.yaml` created
- [ ] `desktop/ai-studio-desktop/` is the canonical desktop location
- [ ] `products/content-factory/` is the canonical CF location
- [ ] `products/mythic-realms/orchestrator/app.py` created (MR uses platform as package)
- [ ] All 233 tests pass
- [ ] MR orchestrator starts successfully

---

## Phase 6: SDK, Tools, and Cleanup

**Target duration:** 4–5 engineer-days  
**Risk level:** LOW  
**Dependencies:** Phase 5 complete

### Objective

Extract the SDK to its own installable package. Remove the MR vendored SDK copy. Extract CLI tools. Remove backward-compat shims.

### Steps

**Step 6.1 — SDK package extraction**
```
factory/sdk/plugin_base.py  → sdk/python/aisf/plugin_base.py
factory/sdk/worker_base.py  → sdk/python/aisf/worker_base.py
factory/sdk/api.py          → sdk/python/aisf/api.py
factory/plugins/            → sdk/python/aisf/plugins/

sdk/python/pyproject.toml:
  [project]
  name = "aisf-sdk"
  version = "1.0.0"
  description = "AI Studio Platform Worker SDK"
  dependencies = []  # Zero runtime dependencies
```

**Step 6.2 — Remove MR vendored SDK (C-002 resolution)**
```
DELETE: mythic-realms/ai-software-factory/ (entire directory)

Update mythic-realms pyproject.toml:
  dependencies = [
      "aisf-sdk>=1.0,<2.0",  # replaces vendored copy
  ]
```

**Step 6.3 — CLI tools extraction**
```
factory/cli/generator.py → tools/generator/generator.py
factory/hooks/registry.py → tools/generator/hooks.py
factory/versioning.py → platform/configuration/versioning.py
```

**Step 6.4 — Templates extraction**
```
factory/plugins/cicd_plugin.py    → sdk/python/aisf/plugins/cicd_plugin.py
factory/plugins/java_plugin.py    → sdk/python/aisf/plugins/java_plugin.py
factory/plugins/python_plugin.py  → sdk/python/aisf/plugins/python_plugin.py
factory/plugins/content_factory_plugin.py → sdk/python/aisf/plugins/content_factory_plugin.py

(to examples/)
ai-software-factory/templates/ → examples/
mythic-realms/ai-software-factory/templates/ → examples/ (de-duplicate)
```

**Step 6.5 — Remove ORCH_ backward compat**

Remove the `ORCH_*` → `AISF_*` backward compatibility shims from configuration module. Any remaining `ORCH_*` usage after this step fails at startup with an explicit error message: "ORCH_* env vars are no longer supported. Migrate to AISF_* prefix."

**Step 6.6 — Remove db/models/__init__.py re-export shim**

Update all remaining `from db.models import X` to `from db.models.<schema> import X`. Remove the flat re-export shim.

**Step 6.7 — Architecture linter setup**
```
tools/architecture-linter/
├── check_imports.py     — enforces dependency rules
├── check_events.py      — validates event naming conventions
├── check_config_prefix.py — detects ORCH_ usage
└── .importlinter        — import-linter configuration
```

Add architecture linter as mandatory CI check.

### Phase 6 Definition of Done

- [ ] `sdk/python/` exists as installable package with zero runtime deps
- [ ] `mythic-realms/ai-software-factory/` deleted
- [ ] MR imports `aisf.worker_base` not `factory.sdk.worker_base`
- [ ] No `ORCH_` env var references anywhere in codebase
- [ ] No `from db.models import` (only `from db.models.schema import`)
- [ ] Architecture linter passes (zero violations)
- [ ] All 233 tests pass

---

## Phase Gate Summary

```
Phase 1: Config + DB Models Split
    Gate: 233 tests pass, platform starts with AISF_API_KEY set
         ↓
Phase 2: Event Bus, Security, Auth Fix
    Gate: 233 tests pass, auth required by default
         ↓
Phase 3: Workflow, Prompt OS, Providers, AISF product extracted
    Gate: 233 tests pass, WorkerRegistry pattern in place
         ↓
Phase 4: Brain, Decision, AI Runtime
    Gate: 233 tests pass, all intelligence services extracted
         ↓
Phase 5: Products, Desktop, MR Workers relocated
    Gate: 233 tests pass, zero MR workers in platform, MR orchestrator uses platform
         ↓
Phase 6: SDK, Tools, Cleanup
    Gate: 233 tests pass, architecture linter passes, zero ORCH_ references
         ↓
Phase 0B: COMPLETE — Platform separated from products
```

---

## Risk Register

| Risk | Phase | Likelihood | Impact | Mitigation |
|------|-------|-----------|--------|-----------|
| db/models split breaks alembic migrations | 1 | HIGH | HIGH | Run `alembic check` after split; update migration `from db.models import` refs |
| Auth hardening breaks CI that uses no key | 2 | HIGH | MEDIUM | Update CI to set `AISF_API_KEY=test-key-local AISF_ENVIRONMENT=local` |
| WorkerRegistry breaks test that expect static imports | 3 | MEDIUM | HIGH | Update test fixtures to register workers before testing dispatch |
| MR vendored SDK removal breaks MR tests | 6 | MEDIUM | MEDIUM | Install SDK package first, verify MR tests pass, then delete vendored copy |
| ORCH_ removal breaks a deployed .env | 6 | HIGH | LOW | Migration guide; 90-day warning period in Phase 1 |

---

*End of PLATFORM-EXTRACTION-ROADMAP.md*  
*Version 1.0.0 | Status: PROPOSED | 6 phases | ~35 engineer-days total*
