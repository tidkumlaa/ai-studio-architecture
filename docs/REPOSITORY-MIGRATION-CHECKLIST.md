# AI Studio Platform — Repository Migration Checklist

**Document ID:** REPOSITORY-MIGRATION-CHECKLIST  
**Version:** 1.0.0  
**Date:** 2026-06-28  
**Status:** PROPOSED  
**Authority:** Chief Software Architect  
**Intended User:** Platform Engineer performing extraction  
**Companion:** PLATFORM-EXTRACTION-ROADMAP.md

---

## How to Use This Checklist

Each phase has a checklist. Work top-to-bottom within a phase. Each item is a concrete, verifiable action. An item is not done until it is verified — not just attempted.

**Before starting any phase:**
- [ ] Create a git branch: `git checkout -b phase-0b-extract-phase-N`
- [ ] Confirm all tests pass on the current state: `python -m pytest tests/ -v -q`
- [ ] Commit count baseline: `git log --oneline | head -5`

**After completing any phase:**
- [ ] Run full test suite: `python -m pytest tests/ -v -q`
- [ ] Start the platform: `python app.py`
- [ ] Hit health endpoint: `curl http://localhost:8088/api/v1/health`
- [ ] Commit the phase: `git commit -m "Phase 0B Phase N: <description>"`
- [ ] Create PR for review before proceeding to next phase

---

## Preparation Checklist (Before Phase 1)

### Environment Setup

- [ ] Confirm Python 3.13 is installed: `python --version`
- [ ] Confirm all 4 repos are on `main` branch with no uncommitted changes
- [ ] Confirm 233 tests pass: `python -m pytest tests/ -v -q`
- [ ] Install `import-linter`: `pip install import-linter`
- [ ] Install `pyflakes`: `pip install pyflakes`
- [ ] Back up `ai-software-factory/db/models.py` to `db/models_ORIGINAL.py` (reference copy)

### Target Directory Structure Creation

Create the target directories (empty — content added in each phase):

**In the AIStudio workspace root:**
- [ ] Create `E:\UserData\MyData\Content\AIStudio\platform\` directory
- [ ] Create `E:\UserData\MyData\Content\AIStudio\platform\ai-runtime\`
- [ ] Create `E:\UserData\MyData\Content\AIStudio\platform\workflow-runtime\`
- [ ] Create `E:\UserData\MyData\Content\AIStudio\platform\prompt-os\`
- [ ] Create `E:\UserData\MyData\Content\AIStudio\platform\decision-engine\`
- [ ] Create `E:\UserData\MyData\Content\AIStudio\platform\central-brain\`
- [ ] Create `E:\UserData\MyData\Content\AIStudio\platform\security\`
- [ ] Create `E:\UserData\MyData\Content\AIStudio\platform\knowledge\`
- [ ] Create `E:\UserData\MyData\Content\AIStudio\platform\event-bus\`
- [ ] Create `E:\UserData\MyData\Content\AIStudio\platform\provider-runtime\`
- [ ] Create `E:\UserData\MyData\Content\AIStudio\platform\plugin-runtime\`
- [ ] Create `E:\UserData\MyData\Content\AIStudio\platform\metrics\`
- [ ] Create `E:\UserData\MyData\Content\AIStudio\platform\observability\`
- [ ] Create `E:\UserData\MyData\Content\AIStudio\platform\configuration\`
- [ ] Create `E:\UserData\MyData\Content\AIStudio\platform\workspace\`
- [ ] Create `E:\UserData\MyData\Content\AIStudio\platform\review-engine\`
- [ ] Create `E:\UserData\MyData\Content\AIStudio\platform\governance\`
- [ ] Create `E:\UserData\MyData\Content\AIStudio\platform\storage\`
- [ ] Create `E:\UserData\MyData\Content\AIStudio\platform\api\`
- [ ] Create `E:\UserData\MyData\Content\AIStudio\products\`
- [ ] Create `E:\UserData\MyData\Content\AIStudio\products\ai-software-factory\`
- [ ] Create `E:\UserData\MyData\Content\AIStudio\products\ai-software-factory\engine\`
- [ ] Create `E:\UserData\MyData\Content\AIStudio\products\ai-software-factory\workers\`
- [ ] Create `E:\UserData\MyData\Content\AIStudio\products\ai-software-factory\api\`
- [ ] Create `E:\UserData\MyData\Content\AIStudio\desktop\`
- [ ] Create `E:\UserData\MyData\Content\AIStudio\sdk\`
- [ ] Create `E:\UserData\MyData\Content\AIStudio\sdk\python\`
- [ ] Create `E:\UserData\MyData\Content\AIStudio\tools\`
- [ ] Create `E:\UserData\MyData\Content\AIStudio\tools\generator\`
- [ ] Create `E:\UserData\MyData\Content\AIStudio\tools\architecture-linter\`
- [ ] Create `E:\UserData\MyData\Content\AIStudio\tools\migration-tools\`
- [ ] Create `E:\UserData\MyData\Content\AIStudio\examples\`
- [ ] Create `E:\UserData\MyData\Content\AIStudio\marketplace\`
- [ ] Create `E:\UserData\MyData\Content\AIStudio\playground\`

### Workspace YAML Preparation

- [ ] Review `workspace.yaml` — note all path references to update during migration
- [ ] Confirm `workspace.yaml` `migration_phase: 2` is current
- [ ] Draft `workspace.yaml` v3.0 with new paths (hold — apply in Phase 6)

---

## Phase 1 Checklist: Configuration and DB Models

### Step 1.1 — Configuration Module

**Action:**
- [ ] Create `platform/configuration/` as a Python package (`__init__.py`)
- [ ] Copy `ai-software-factory/config.py` to `platform/configuration/settings.py`
- [ ] Modify `settings.py`:
  - [ ] Change `env_prefix` from `"ORCH_"` to `"AISF_"`
  - [ ] Add dual-read: if `AISF_API_KEY` not set, try `ORCH_API_KEY` with DeprecationWarning
  - [ ] Change `api_key: str = ""` to validation that requires key (or allows local skip)
  - [ ] Add `environment: str = "local"` field
  - [ ] Add `auth_skip_localhost: bool = False` field
- [ ] Create `ai-software-factory/config.py` as a re-export shim:
  ```python
  # config.py — backward compat shim during transition
  from platform.configuration.settings import settings, Settings  # noqa: F401
  ```

**Verification:**
- [ ] `from config import settings` still works in Python shell
- [ ] `from platform.configuration.settings import settings` works
- [ ] `python -c "from config import settings; print(settings.model_fields_set)"` runs
- [ ] Setting `ORCH_API_KEY=test` logs a DeprecationWarning
- [ ] Setting `AISF_API_KEY=test` has no warning

### Step 1.2 — Platform Version

**Action:**
- [ ] Copy `ai-software-factory/platform_version.py` to `platform/configuration/version.py`
- [ ] Create `platform_version.py` as re-export shim

**Verification:**
- [ ] `from platform_version import PLATFORM_VERSION` still works

### Step 1.3 — DB Models Split

**Action:**
- [ ] Create `ai-software-factory/db/models/` directory
- [ ] Create `db/models/base.py` with `Base, DeclarativeBase` import
- [ ] Create `db/models/workflow.py` — move 16 workflow/task model classes
- [ ] Create `db/models/prompt.py` — move 5 prompt/conversation model classes
- [ ] Create `db/models/provider.py` — move 3 provider/model classes
- [ ] Create `db/models/ai_runtime.py` — move 3 AI execution/cost model classes
- [ ] Create `db/models/brain.py` — move 8 brain model classes
- [ ] Create `db/models/aisf_product.py` — move remaining 19 AISF-specific classes
- [ ] Create `db/models/__init__.py` with `from db.models.* import *` for all schemas
- [ ] Delete old `db/models.py`

**Verification:**
- [ ] `python -c "from db.models import Task, BrainExperience, Product"` succeeds
- [ ] `python -c "from db.models.workflow import Task"` succeeds
- [ ] `python -c "from db.models.brain import BrainExperience"` succeeds
- [ ] `python -m pytest tests/ -v -q` — all 233 tests pass
- [ ] `python app.py` starts (with `AISF_API_KEY=test AISF_ENVIRONMENT=local AISF_AUTH_SKIP_LOCALHOST=true`)
- [ ] `alembic check` — no pending migrations detected

**Phase 1 gate:**
- [ ] All 233 tests pass
- [ ] Platform starts without error
- [ ] Health endpoint returns 200
- [ ] PR created and reviewed

---

## Phase 2 Checklist: Event Bus, Security, Auth Fix

### Step 2.1 — Event Bus Extraction

**Action:**
- [ ] Copy `engine/event_bus.py` to `platform/event-bus/in_process.py`
- [ ] Copy `engine/nats_client.py` to `platform/event-bus/nats_client.py`
- [ ] Copy `engine/event_schemas.py` to `platform/event-bus/schemas.py`
- [ ] Create `platform/event-bus/__init__.py`
- [ ] Update imports in `api/ws_routes.py`: `from engine.event_bus` → `from platform.event_bus.in_process`
- [ ] Update imports in `api/nats_routes.py`: `from engine.nats_client` → `from platform.event_bus.nats_client`
- [ ] Update imports in `app.py`: same
- [ ] Replace `engine/event_bus.py` with re-export shim
- [ ] Replace `engine/nats_client.py` with re-export shim

**Verification:**
- [ ] `from engine.event_bus import EventBus` still works (via shim)
- [ ] `from platform.event_bus.in_process import EventBus` works
- [ ] WebSocket route `/api/v1/desktop/ws` connects successfully
- [ ] NATS route `/api/v1/nats/health` responds

### Step 2.2 — Security Module Extraction

**Action:**
- [ ] Copy `engine/security.py` to `platform/security/rbac.py`
- [ ] Extract `RateLimiter` to `platform/security/rate_limiter.py`
- [ ] Extract `PromptValidator` to `platform/security/prompt_validator.py`
- [ ] Create `platform/security/__init__.py`
- [ ] Replace `engine/security.py` with re-export shim

**Verification:**
- [ ] `from engine.security import RBACManager` still works
- [ ] `from platform.security.rbac import RBACManager` works

### Step 2.3 — Authentication Hardening (C-005 fix)

**Action:**
- [ ] Copy `api/auth.py` to `platform/security/auth_middleware.py`
- [ ] **Remove** the `if not settings.api_key: return` auth-disabled path
- [ ] Add local-only bypass:
  ```python
  if (settings.environment == "local"
      and settings.auth_skip_localhost
      and request.client.host == "127.0.0.1"):
      request.state.auth = _anonymous_local_context()
      return
  ```
- [ ] Update `app.py` to import from `platform.security.auth_middleware`
- [ ] Update CI test configuration to set `AISF_API_KEY=test-key-local AISF_ENVIRONMENT=local AISF_AUTH_SKIP_LOCALHOST=true`
- [ ] Update any test fixtures that call platform without auth

**Verification:**
- [ ] `curl http://localhost:8088/api/v1/workflow/` with no key returns 401
- [ ] `curl http://localhost:8088/api/v1/health` with no key returns 200 (public endpoint)
- [ ] `curl -H "Authorization: Bearer test-key-local" http://localhost:8088/api/v1/workflow/` returns 200
- [ ] All 233 tests pass (with API key set in test environment)

### Step 2.4 — Metrics and Logging Extraction

**Action:**
- [ ] Copy `factory/logging/` to `platform/metrics/logging/`
- [ ] Copy `factory/metrics/` to `platform/metrics/prometheus/`
- [ ] Copy `factory/tracing/` to `platform/metrics/tracing/`
- [ ] Create re-export shims in `factory/logging/`, `factory/metrics/`, `factory/tracing/`
- [ ] Update `app.py` imports

**Verification:**
- [ ] `python app.py` starts with structured JSON logging
- [ ] `curl http://localhost:8088/metrics` returns Prometheus metrics
- [ ] All 233 tests pass

**Phase 2 gate:**
- [ ] All 233 tests pass
- [ ] Auth is required by default
- [ ] Platform starts with `AISF_API_KEY` set
- [ ] PR created and reviewed

---

## Phase 3 Checklist: Core Services and AISF Product Extraction

### Step 3.1 — Provider Runtime

**Action:**
- [ ] Copy `factory/providers/` to `platform/provider-runtime/`
- [ ] Copy `runtime/claude_runtime.py` to `platform/provider-runtime/claude_runtime.py`
- [ ] Copy `runtime/claude_executor.py` to `platform/provider-runtime/claude_executor.py`
- [ ] Copy `runtime/provider_runtime.py` to `platform/provider-runtime/runtime.py`
- [ ] Update imports in `engine/ai_execution.py`
- [ ] Create re-export shims in `factory/providers/`

**Verification:**
- [ ] `curl -H "Authorization: Bearer ..." http://localhost:8088/api/v1/ai/providers` returns provider list
- [ ] All 233 tests pass

### Step 3.2 — WorkerRegistry Pattern (C-001 + C-008 fix, prerequisite for step 3.3)

**Action:**
- [ ] Create `platform/workflow-runtime/worker_registry.py` with `WorkerRegistry` class
- [ ] Modify `engine/dispatcher.py`:
  - [ ] Remove all `from worker import ...` imports
  - [ ] Replace worker class lookup with `WorkerRegistry.get(agent_name)`
  - [ ] Add graceful failure when worker not found
- [ ] Create `app.py` worker registration section:
  ```python
  # Register AISF product workers (temporary — moves to AISF product startup in Phase 5)
  from worker.architect_worker import ArchitectWorker
  WorkerRegistry.register("ARCHITECT", ArchitectWorker)
  # ... etc.
  ```

**Verification:**
- [ ] `python -c "from engine.dispatcher import dispatch_ready_tasks"` succeeds
- [ ] Worker dispatch test (`tests/test_workflow_engine.py`) passes
- [ ] All 233 tests pass

### Step 3.3 — Workflow Runtime

**Action:**
- [ ] Copy `engine/workflow_engine.py` to `platform/workflow-runtime/engine.py`
- [ ] Copy `engine/workflow_runtime.py` to `platform/workflow-runtime/runtime.py`
- [ ] Copy `engine/workflow_supervisor.py` to `platform/workflow-runtime/supervisor.py`
- [ ] Copy `engine/dependency_resolver.py` to `platform/workflow-runtime/dependency_resolver.py`
- [ ] Copy `engine/dispatcher.py` (updated) to `platform/workflow-runtime/dispatcher.py`
- [ ] Copy `engine/blocker_monitor.py` to `platform/workflow-runtime/blocker_monitor.py`
- [ ] Copy `engine/review_router.py` to `platform/review-engine/review_router.py`
- [ ] Copy `engine/merge_manager.py` to `platform/review-engine/merge_manager.py`
- [ ] Copy `engine/approval_service.py` to `platform/governance/approval_service.py`
- [ ] Copy `engine/sla_monitor.py` to `platform/observability/sla_monitor.py`
- [ ] Copy `supervisor.py` to `platform/workflow-runtime/worker_supervisor.py`
- [ ] Update imports in all `api/workflow_routes.py`, `api/workflow_runtime_routes.py`
- [ ] Create re-export shims for all original paths

**Verification:**
- [ ] `curl ... /api/v1/workflow/` returns workflow list
- [ ] `tests/test_workflow_runtime.py` passes
- [ ] All 233 tests pass

### Step 3.4 — Prompt OS

**Action:**
- [ ] Copy `engine/prompt_os/` to `platform/prompt-os/`
- [ ] Copy `engine/context_engine.py` to `platform/prompt-os/context_engine.py`
- [ ] Copy `engine/conversation_engine.py` to `platform/prompt-os/conversation_engine.py`
- [ ] Copy `engine/prompt_runtime.py` to `platform/prompt-os/runtime.py`
- [ ] Copy `engine/prompt_intelligence.py` to `platform/prompt-os/intelligence.py`
- [ ] Update `api/prompt_os_routes.py`, `api/prompt_intelligence_routes.py`
- [ ] Create re-export shims

**Verification:**
- [ ] `tests/test_prompt_os.py` passes
- [ ] All 233 tests pass

### Step 3.5 — AISF Product Extraction (C-007 fix)

**Action:**
- [ ] Move (do NOT copy) 11 product modules from `engine/` to `products/ai-software-factory/engine/`:
  - [ ] `engine/product_factory.py`
  - [ ] `engine/product_intake.py`
  - [ ] `engine/business_analyst.py`
  - [ ] `engine/architect_agent.py`
  - [ ] `engine/planner_engine.py`
  - [ ] `engine/artifact_generator.py`
  - [ ] `engine/infra_patterns.py`
  - [ ] `engine/org_engine.py`
  - [ ] `engine/employee_engine.py`
  - [ ] `engine/docker_service.py`
  - [ ] `engine/root_cause_engine.py`
- [ ] Create `engine/product_factory.py` re-export shim
- [ ] Move API routes to `products/ai-software-factory/api/`
- [ ] Create product startup extension: `products/ai-software-factory/startup.py`
- [ ] Update `app.py` to call `aisf_product.startup.register_routes(app)` instead of direct route import

**Verification:**
- [ ] `tests/test_product_factory.py` passes
- [ ] `curl ... /api/v1/products` returns product list
- [ ] All 233 tests pass

**Phase 3 gate:**
- [ ] All 233 tests pass
- [ ] WorkerRegistry in place (no direct worker imports in dispatcher)
- [ ] AISF product logic in `products/ai-software-factory/`
- [ ] PR created and reviewed

---

## Phase 4 Checklist: Intelligence Services

### Step 4.1 — Central Brain

**Action:**
- [ ] Copy `engine/central_brain.py` to `platform/central-brain/central_brain.py`
- [ ] Copy `engine/brain.py` to `platform/central-brain/brain.py`
- [ ] Copy `engine/memory_os.py` to `platform/central-brain/memory_os.py`
- [ ] Copy `engine/experience_recorder.py` to `platform/central-brain/experience_recorder.py`
- [ ] Copy `engine/outcome_collector.py` to `platform/central-brain/outcome_collector.py`
- [ ] Copy `engine/learning_agent.py` to `platform/central-brain/learning_agent.py`
- [ ] Copy `engine/self_improvement.py` to `platform/central-brain/self_improvement.py`
- [ ] Copy `engine/failure_analyzer.py` to `platform/central-brain/failure_analyzer.py`
- [ ] Copy `factory/memory/` to `platform/central-brain/memory/`
- [ ] Update all API route imports
- [ ] Create re-export shims

**Verification:**
- [ ] `tests/test_central_brain.py` passes
- [ ] `tests/test_brain.py` passes
- [ ] `tests/test_memory_os.py` passes
- [ ] All 233 tests pass

### Step 4.2 — Decision Engine and AI Runtime

**Action:**
- [ ] Copy `engine/decision_engine.py` to `platform/decision-engine/engine.py`
- [ ] Copy `engine/ai_router.py` to `platform/ai-runtime/router.py`
- [ ] Copy `engine/model_registry.py` to `platform/ai-runtime/model_registry.py`
- [ ] Copy `engine/ai_execution.py` to `platform/ai-runtime/execution.py`
- [ ] Copy `engine/cost_engine.py` to `platform/ai-runtime/cost_engine.py`
- [ ] Copy `engine/tool_runtime.py` to `platform/plugin-runtime/tool_runtime.py`
- [ ] Update all API route imports
- [ ] Create re-export shims

**Verification:**
- [ ] `tests/test_decision_engine.py` passes
- [ ] `tests/test_ai_execution_engine.py` passes
- [ ] `tests/test_cost_engine.py` passes
- [ ] All 233 tests pass

**Phase 4 gate:**
- [ ] All 233 tests pass
- [ ] All intelligence service tests pass individually
- [ ] PR created and reviewed

---

## Phase 5 Checklist: Products and Desktop Reorganization

### Step 5.1 — Remove MR Workers from Platform (C-001 resolution)

**PREREQUISITE:** WorkerRegistry pattern from Phase 3 Step 3.2 is in place.

**Action:**
- [ ] Verify `engine/dispatcher.py` no longer imports from `worker/` directly
- [ ] DELETE `ai-software-factory/worker/battle_worker.py`
- [ ] DELETE `ai-software-factory/worker/economy_worker.py`
- [ ] DELETE `ai-software-factory/worker/collection_worker.py`
- [ ] DELETE `ai-software-factory/worker/liveops_worker.py`
- [ ] DELETE `ai-software-factory/worker/player_worker.py`

**Verification:**
- [ ] `ls ai-software-factory/worker/` shows only: architect, auth, qa, security, release workers + `__init__.py`
- [ ] Platform starts without error
- [ ] All 233 tests pass

### Step 5.2 — Move AISF Workers to Product

**Action:**
- [ ] Move `worker/architect_worker.py` to `products/ai-software-factory/workers/`
- [ ] Move `worker/auth_worker.py` to `products/ai-software-factory/workers/`
- [ ] Move `worker/qa_worker.py` to `products/ai-software-factory/workers/`
- [ ] Move `worker/security_worker.py` to `products/ai-software-factory/workers/`
- [ ] Move `worker/release_worker.py` to `products/ai-software-factory/workers/`
- [ ] Create `products/ai-software-factory/workers/__init__.py`
- [ ] Update `app.py` worker registration to import from new path
- [ ] Create `products/ai-software-factory/factory.yaml`

**Verification:**
- [ ] Platform starts with AISF workers registered
- [ ] All 233 tests pass

### Step 5.3 — Mythic Realms Workers Cleanup

**Action:**
- [ ] Verify `mythic-realms/tools/orchestrator/worker/` has all game workers
- [ ] Create `products/mythic-realms/factory.yaml` with all MR worker declarations
- [ ] Create `products/mythic-realms/orchestrator/app.py` (thin platform launcher)
- [ ] Verify MR orchestrator starts using the platform as a library

**Verification:**
- [ ] MR orchestrator starts: `python products/mythic-realms/orchestrator/app.py`
- [ ] MR health endpoint: `curl http://localhost:8088/api/v1/health` returns 200 from MR orchestrator
- [ ] MR-specific workers are registered (BATTLE-AGENT, ECONOMY-AGENT, etc.)

### Step 5.4 — Desktop Move

**Action:**
- [ ] `git mv source/ai-studio-desktop desktop/ai-studio-desktop` (preserves git history)
- [ ] Fix `services/workspace_service.py`: update `parents[3]` to `parents[4]`
- [ ] Update `workspace.yaml` path for `ai-studio-desktop`

**Verification:**
- [ ] Desktop starts: `python desktop/ai-studio-desktop/main.py`
- [ ] All desktop tests pass: `python -m pytest desktop/ai-studio-desktop/tests/ -v -q`

### Step 5.5 — Content Factory Move

**Action:**
- [ ] `git mv source/content-factory products/content-factory` (preserves git history)
- [ ] Update `workspace.yaml` path for `content-factory`

**Verification:**
- [ ] CF health: `curl http://localhost:8090/actuator/health/liveness` returns `{"status":"UP"}`
- [ ] `workspace.yaml` paths are correct

**Phase 5 gate:**
- [ ] Zero MR workers in `ai-software-factory/worker/`
- [ ] AISF workers in `products/ai-software-factory/workers/`
- [ ] Desktop at `desktop/ai-studio-desktop/`
- [ ] Content Factory at `products/content-factory/`
- [ ] MR orchestrator uses platform as library
- [ ] All 233 tests pass
- [ ] PR created and reviewed

---

## Phase 6 Checklist: SDK, Tools, and Cleanup

### Step 6.1 — SDK Package

**Action:**
- [ ] Create `sdk/python/aisf/` Python package
- [ ] Copy `factory/sdk/plugin_base.py` to `sdk/python/aisf/plugin_base.py`
- [ ] Copy `factory/sdk/worker_base.py` to `sdk/python/aisf/worker_base.py`
- [ ] Copy `factory/sdk/api.py` to `sdk/python/aisf/api.py`
- [ ] Copy `factory/plugins/` to `sdk/python/aisf/plugins/`
- [ ] Create `sdk/python/pyproject.toml`:
  ```toml
  [project]
  name = "aisf-sdk"
  version = "1.0.0"
  dependencies = []
  ```
- [ ] Install locally: `pip install -e ./sdk/python/`

**Verification:**
- [ ] `python -c "from aisf.worker_base import BaseWorker"` succeeds
- [ ] `python -c "from aisf.plugin_base import AIProviderPlugin"` succeeds
- [ ] `pip show aisf-sdk` shows zero install-time dependencies

### Step 6.2 — Remove MR Vendored SDK (C-002 resolution)

**PREREQUISITE:** `aisf-sdk` installed from step 6.1.

**Action:**
- [ ] Update `mythic-realms/tools/orchestrator/worker/__init__.py`:
  - Change `from factory.sdk.worker_base import BaseWorker` → `from aisf.worker_base import BaseWorker`
- [ ] Verify all MR worker imports use `aisf.*` not `factory.sdk.*`
- [ ] DELETE `mythic-realms/ai-software-factory/` directory (entire)
- [ ] Update `mythic-realms/ai-software-factory/pyproject.toml` (keep as dependency declaration)

**Verification:**
- [ ] `python -m pytest mythic-realms/tools/orchestrator/tests/` passes
- [ ] `ls mythic-realms/ai-software-factory/` shows only `pyproject.toml` (factory/ dir deleted)

### Step 6.3 — Tools Extraction

**Action:**
- [ ] Copy `factory/cli/generator.py` to `tools/generator/generator.py`
- [ ] Copy `factory/hooks/registry.py` to `tools/generator/hooks.py`
- [ ] Create `tools/generator/__init__.py`
- [ ] Create `factory/cli/generator.py` re-export shim
- [ ] Move `db/migrator.py` to `tools/migration-tools/migrator.py`

**Verification:**
- [ ] `python tools/generator/generator.py --help` works (if applicable)

### Step 6.4 — Remove ORCH_ Backward Compatibility (C-004 final resolution)

**Action:**
- [ ] Remove ORCH_ fallback from `platform/configuration/settings.py`
- [ ] Add startup error: if any `ORCH_*` env var detected, fail with migration message
- [ ] Update all `.env.example` files from `ORCH_*` to `AISF_*`

**Verification:**
- [ ] Setting `ORCH_API_KEY=test` causes startup to fail with clear error message
- [ ] Setting `AISF_API_KEY=test` starts successfully
- [ ] All 233 tests pass (test env uses `AISF_*`)

### Step 6.5 — Remove db/models/ Re-export Shim (C-006 final resolution)

**Action:**
- [ ] Search for all `from db.models import` usages: `grep -r "from db.models import" . --include="*.py"`
- [ ] Update each to use schema-namespaced path:
  - `from db.models import Task` → `from db.models.workflow import Task`
  - `from db.models import BrainExperience` → `from db.models.brain import BrainExperience`
  - `from db.models import Product` → `from db.models.aisf_product import Product`
- [ ] Delete `db/models/__init__.py` re-export shim after all usages updated

**Verification:**
- [ ] `grep -r "from db.models import" . --include="*.py"` returns zero results
- [ ] All 233 tests pass

### Step 6.6 — Architecture Linter Setup

**Action:**
- [ ] Create `tools/architecture-linter/check_imports.py`
- [ ] Create `.importlinter` at workspace root (per REPOSITORY-DEPENDENCY-RULES.md)
- [ ] Run linter: `python -m importlinter --config .importlinter`
- [ ] Fix any reported violations
- [ ] Add linter to CI config

**Verification:**
- [ ] `python -m importlinter --config .importlinter` exits with code 0
- [ ] All known C-00X coupling violations marked as resolved

### Step 6.7 — Update workspace.yaml v3.0

**Action:**
- [ ] Update `workspace.yaml` `migration_phase: 3`
- [ ] Update all repository paths to new locations
- [ ] Add `platform/` as a new directory type entry
- [ ] Add `sdk/` entries
- [ ] Update desktop path to `desktop/ai-studio-desktop`

**Verification:**
- [ ] Desktop discovers all services via updated `workspace.yaml`
- [ ] Desktop workspace panel shows all repositories with correct paths

**Phase 6 gate:**
- [ ] `sdk/python/` installed with zero dependencies
- [ ] `mythic-realms/ai-software-factory/factory/` directory deleted
- [ ] No `ORCH_*` references anywhere in codebase
- [ ] Architecture linter passes with zero violations
- [ ] All 233 tests pass
- [ ] PR created and reviewed

---

## Final Verification (Phase 0B Complete)

After all phases pass:

- [ ] Start platform: `AISF_API_KEY=prod-key python platform/api/app.py`
- [ ] Run all tests: `python -m pytest tests/ -v -q` (233 pass)
- [ ] Run desktop: `python desktop/ai-studio-desktop/main.py`
- [ ] Desktop shows all products in workspace panel
- [ ] Run architecture linter: `python -m importlinter --config .importlinter` (0 violations)
- [ ] Run secret scanner: `grep -r "sk_live_" . --include="*.py"` (0 results)
- [ ] Check `ORCH_` prefix: `grep -r "ORCH_" . --include="*.py"` (0 results)
- [ ] Check direct worker imports in dispatcher: `grep "from worker import" platform/workflow-runtime/dispatcher.py` (0 results)
- [ ] Count db.models flat imports: `grep -r "from db.models import" . --include="*.py"` (0 results)
- [ ] Verify MR workers not in platform: `ls ai-software-factory/worker/` (5 AISF workers + __init__.py only)
- [ ] Generate architecture report: `python tools/architecture-linter/check_imports.py --report`

---

*End of REPOSITORY-MIGRATION-CHECKLIST.md*  
*Version 1.0.0 | Status: PROPOSED | 6 phases | ~180 actionable checklist items*
