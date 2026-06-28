# AI Studio Platform — Repository Extraction Matrix

**Document ID:** REPOSITORY-EXTRACTION-MATRIX  
**Version:** 1.0.0  
**Date:** 2026-06-28  
**Status:** PROPOSED  
**Authority:** Chief Software Architect  
**Evidence Source:** Audit of ai-software-factory, ai-studio-desktop, content-factory, mythic-realms  
**Companion:** REPOSITORY-SEPARATION-PLAN.md, COUPLING-ANALYSIS.md

---

## How to Read This Document

Each section covers one source repository. Within each section, every directory or significant module is classified and assigned:
- **Category:** PLATFORM / PRODUCT / SDK / DESKTOP / INFRA / UTILITY / LEGACY / DELETE
- **Target:** Where it goes in the new layout
- **Migration complexity:** LOW / MEDIUM / HIGH / CRITICAL
- **Breaking risk:** Whether moving it will immediately break other code
- **Estimated effort:** Engineer-days for the move + import rewrite + test fix

---

## Section 1: ai-software-factory

### 1.1 api/ — REST API Layer

**Current path:** `ai-software-factory/api/`  
**Files:** 40 route modules, ~3,000 lines

| Module | Category | Target | Complexity | Breaking Risk | Effort |
|--------|----------|--------|-----------|--------------|--------|
| `api/routes.py` | PLATFORM | `platform/api/routes.py` | LOW | MEDIUM (40 imports in app.py) | 0.5d |
| `api/auth.py` | PLATFORM | `platform/security/auth_middleware.py` | MEDIUM | HIGH (every request) | 1d |
| `api/correlation_middleware.py` | PLATFORM | `platform/api/correlation_middleware.py` | LOW | LOW | 0.25d |
| `api/health_routes.py` | PLATFORM | `platform/api/health_routes.py` | LOW | LOW | 0.25d |
| `api/capabilities_routes.py` | PLATFORM | `platform/workspace/capabilities_routes.py` | LOW | LOW | 0.25d |
| `api/metrics_routes.py` | PLATFORM | `platform/metrics/metrics_routes.py` | LOW | LOW | 0.25d |
| `api/workflow_routes.py` | PLATFORM | `platform/workflow-runtime/api.py` | MEDIUM | MEDIUM | 0.5d |
| `api/workflow_runtime_routes.py` | PLATFORM | `platform/workflow-runtime/runtime_api.py` | MEDIUM | MEDIUM | 0.5d |
| `api/prompt_os_routes.py` | PLATFORM | `platform/prompt-os/api.py` | MEDIUM | MEDIUM | 0.5d |
| `api/prompt_intelligence_routes.py` | PLATFORM | `platform/prompt-os/intelligence_api.py` | MEDIUM | MEDIUM | 0.5d |
| `api/brain_routes.py` | PLATFORM | `platform/central-brain/api.py` | MEDIUM | MEDIUM | 0.5d |
| `api/central_brain_routes.py` | PLATFORM | `platform/central-brain/central_api.py` | MEDIUM | MEDIUM | 0.5d |
| `api/decision_routes.py` | PLATFORM | `platform/decision-engine/api.py` | MEDIUM | LOW | 0.5d |
| `api/memory_routes.py` | PLATFORM | `platform/central-brain/memory_api.py` | MEDIUM | LOW | 0.5d |
| `api/experience_routes.py` | PLATFORM | `platform/central-brain/experience_api.py` | LOW | LOW | 0.25d |
| `api/ai_execution_routes.py` | PLATFORM | `platform/ai-runtime/api.py` | MEDIUM | MEDIUM | 0.5d |
| `api/cost_routes.py` | PLATFORM | `platform/ai-runtime/cost_api.py` | LOW | LOW | 0.25d |
| `api/security_routes.py` | PLATFORM | `platform/security/api.py` | MEDIUM | HIGH | 1d |
| `api/approval_routes.py` | PLATFORM | `platform/governance/api.py` | MEDIUM | MEDIUM | 0.5d |
| `api/storage_routes.py` | PLATFORM | `platform/storage/api.py` | LOW | LOW | 0.25d |
| `api/asset_routes.py` | PLATFORM | `platform/storage/asset_api.py` | LOW | LOW | 0.25d |
| `api/nats_routes.py` | PLATFORM | `platform/event-bus/api.py` | LOW | LOW | 0.25d |
| `api/ws_routes.py` | PLATFORM | `platform/api/websocket_routes.py` | MEDIUM | MEDIUM | 0.5d |
| `api/proxy_routes.py` | PLATFORM | `platform/api/proxy_routes.py` | LOW | LOW | 0.25d |
| `api/db_routes.py` | INFRA | `platform/api/db_routes.py` | LOW | LOW | 0.25d |
| `api/runtime_routes.py` | INFRA | `platform/api/runtime_routes.py` | LOW | LOW | 0.25d |
| `api/product_routes.py` | PRODUCT (AISF) | `products/ai-software-factory/api.py` | MEDIUM | MEDIUM | 0.5d |
| `api/improvement_routes.py` | PRODUCT (AISF) | `products/ai-software-factory/improvement_api.py` | LOW | LOW | 0.25d |
| `api/org_routes.py` | PRODUCT (AISF) | `products/ai-software-factory/org_api.py` | LOW | LOW | 0.25d |
| `api/employee_routes.py` | PRODUCT (AISF) | `products/ai-software-factory/employee_api.py` | LOW | LOW | 0.25d |
| `api/agent_metrics_routes.py` | PRODUCT (AISF) | `products/ai-software-factory/agent_metrics_api.py` | LOW | LOW | 0.25d |
| `api/git_routes.py` | PRODUCT (AISF) | `products/ai-software-factory/git_api.py` | LOW | LOW | 0.25d |
| `api/docker_routes.py` | PRODUCT (AISF) | `products/ai-software-factory/docker_api.py` | LOW | LOW | 0.25d |
| `api/plugin_sdk_routes.py` | SDK | `sdk/python/api_routes.py` | LOW | LOW | 0.25d |
| `api/schemas.py` | PLATFORM | `platform/api/schemas.py` | LOW | MEDIUM | 0.5d |

**Total API extraction effort:** ~9 engineer-days

---

### 1.2 engine/ — Core Runtime Modules

**Current path:** `ai-software-factory/engine/`  
**Files:** 43 modules + 11 in prompt_os/ subdirectory

#### 1.2.1 Platform Core Modules

| Module | Category | Target | Complexity | Breaking Risk | Effort | Depends On |
|--------|----------|--------|-----------|--------------|--------|-----------|
| `engine/workflow_engine.py` | PLATFORM | `platform/workflow-runtime/engine.py` | MEDIUM | HIGH | 1d | db.models, engine.dispatcher |
| `engine/workflow_runtime.py` | PLATFORM | `platform/workflow-runtime/runtime.py` | HIGH | HIGH | 2d | db.models, engine.nats_client |
| `engine/workflow_supervisor.py` | PLATFORM | `platform/workflow-runtime/supervisor.py` | MEDIUM | MEDIUM | 1d | db.models, engine.event_bus |
| `engine/dispatcher.py` | PLATFORM | `platform/workflow-runtime/dispatcher.py` | HIGH | HIGH | 1.5d | db.models, worker.* (coupling!) |
| `engine/dependency_resolver.py` | PLATFORM | `platform/workflow-runtime/dependency_resolver.py` | LOW | LOW | 0.5d | db.models |
| `engine/blocker_monitor.py` | PLATFORM | `platform/workflow-runtime/blocker_monitor.py` | LOW | LOW | 0.5d | db.models |
| `engine/sla_monitor.py` | PLATFORM | `platform/observability/sla_monitor.py` | LOW | LOW | 0.5d | db.models |
| `engine/review_router.py` | PLATFORM | `platform/review-engine/review_router.py` | MEDIUM | MEDIUM | 0.5d | db.models, worker.* (coupling!) |
| `engine/merge_manager.py` | PLATFORM | `platform/review-engine/merge_manager.py` | MEDIUM | MEDIUM | 0.5d | db.models |
| `engine/approval_service.py` | PLATFORM | `platform/governance/approval_service.py` | MEDIUM | MEDIUM | 1d | db.models |
| `engine/event_bus.py` | PLATFORM | `platform/event-bus/in_process.py` | LOW | MEDIUM | 0.5d | (no deps) |
| `engine/nats_client.py` | PLATFORM | `platform/event-bus/nats_client.py` | MEDIUM | LOW | 1d | (no deps) |
| `engine/event_schemas.py` | PLATFORM | `platform/event-bus/schemas.py` | LOW | LOW | 0.25d | (no deps) |
| `engine/security.py` | PLATFORM | `platform/security/rbac.py` | MEDIUM | HIGH | 1d | (no deps) |
| `engine/cost_engine.py` | PLATFORM | `platform/ai-runtime/cost_engine.py` | MEDIUM | MEDIUM | 1d | db.models |
| `engine/ai_router.py` | PLATFORM | `platform/ai-runtime/router.py` | MEDIUM | MEDIUM | 1d | db.models, engine.model_registry |
| `engine/model_registry.py` | PLATFORM | `platform/ai-runtime/model_registry.py` | LOW | LOW | 0.5d | db.models |
| `engine/ai_execution.py` | PLATFORM | `platform/ai-runtime/execution.py` | HIGH | HIGH | 2d | db.models, factory.providers |
| `engine/tool_runtime.py` | PLATFORM | `platform/plugin-runtime/tool_runtime.py` | MEDIUM | LOW | 1d | (no deps) |
| `engine/context_engine.py` | PLATFORM | `platform/prompt-os/context.py` | LOW | LOW | 0.5d | db.models |
| `engine/conversation_engine.py` | PLATFORM | `platform/prompt-os/conversation.py` | LOW | LOW | 0.5d | db.models |
| `engine/prompt_runtime.py` | PLATFORM | `platform/prompt-os/runtime.py` | MEDIUM | MEDIUM | 1d | db.models |
| `engine/prompt_intelligence.py` | PLATFORM | `platform/prompt-os/intelligence.py` | MEDIUM | LOW | 1d | db.models |
| `engine/memory_os.py` | PLATFORM | `platform/central-brain/memory_os.py` | MEDIUM | MEDIUM | 1d | db.models |
| `engine/brain.py` | PLATFORM | `platform/central-brain/brain.py` | MEDIUM | MEDIUM | 1d | db.models |
| `engine/central_brain.py` | PLATFORM | `platform/central-brain/central_brain.py` | HIGH | MEDIUM | 2d | db.models |
| `engine/decision_engine.py` | PLATFORM | `platform/decision-engine/engine.py` | MEDIUM | LOW | 1d | db.models |
| `engine/experience_recorder.py` | PLATFORM | `platform/central-brain/experience_recorder.py` | LOW | LOW | 0.5d | db.models |
| `engine/outcome_collector.py` | PLATFORM | `platform/central-brain/outcome_collector.py` | LOW | LOW | 0.5d | db.models |
| `engine/learning_agent.py` | PLATFORM | `platform/central-brain/learning_agent.py` | MEDIUM | LOW | 1d | db.models |
| `engine/self_improvement.py` | PLATFORM | `platform/central-brain/self_improvement.py` | MEDIUM | LOW | 1d | db.models |
| `engine/failure_analyzer.py` | PLATFORM | `platform/central-brain/failure_analyzer.py` | LOW | LOW | 0.5d | db.models |

#### 1.2.2 Prompt OS Subpackage

| Module | Category | Target | Complexity | Breaking Risk | Effort |
|--------|----------|--------|-----------|--------------|--------|
| `engine/prompt_os/__init__.py` | PLATFORM | `platform/prompt-os/__init__.py` | LOW | MEDIUM | 0.25d |
| `engine/prompt_os/orchestrator.py` | PLATFORM | `platform/prompt-os/orchestrator.py` | HIGH | HIGH | 2d |
| `engine/prompt_os/template.py` | PLATFORM | `platform/prompt-os/template.py` | MEDIUM | HIGH | 1d |
| `engine/prompt_os/governance.py` | PLATFORM | `platform/prompt-os/governance.py` | MEDIUM | MEDIUM | 1d |
| `engine/prompt_os/execution.py` | PLATFORM | `platform/prompt-os/execution.py` | MEDIUM | MEDIUM | 1d |
| `engine/prompt_os/composition.py` | PLATFORM | `platform/prompt-os/composition.py` | LOW | LOW | 0.5d |
| `engine/prompt_os/context.py` | PLATFORM | `platform/prompt-os/context_manager.py` | LOW | LOW | 0.5d |
| `engine/prompt_os/brain_bridge.py` | PLATFORM | `platform/prompt-os/brain_bridge.py` | LOW | LOW | 0.5d |
| `engine/prompt_os/security.py` | PLATFORM | `platform/prompt-os/security.py` | LOW | LOW | 0.5d |
| `engine/prompt_os/metrics.py` | PLATFORM | `platform/prompt-os/metrics.py` | LOW | LOW | 0.25d |
| `engine/prompt_os/marketplace.py` | PLATFORM | `platform/prompt-os/marketplace.py` | LOW | LOW | 0.5d |
| `engine/prompt_os/_db.py` | PLATFORM | `platform/prompt-os/_db.py` | LOW | LOW | 0.25d |

#### 1.2.3 AISF Product Modules (currently in engine/)

| Module | Category | Target | Complexity | Breaking Risk | Effort |
|--------|----------|--------|-----------|--------------|--------|
| `engine/product_factory.py` | PRODUCT (AISF) | `products/ai-software-factory/engine/product_factory.py` | HIGH | HIGH | 2d |
| `engine/product_intake.py` | PRODUCT (AISF) | `products/ai-software-factory/engine/product_intake.py` | MEDIUM | MEDIUM | 1d |
| `engine/business_analyst.py` | PRODUCT (AISF) | `products/ai-software-factory/engine/business_analyst.py` | MEDIUM | LOW | 1d |
| `engine/architect_agent.py` | PRODUCT (AISF) | `products/ai-software-factory/engine/architect_agent.py` | HIGH | HIGH | 2d |
| `engine/planner_engine.py` | PRODUCT (AISF) | `products/ai-software-factory/engine/planner_engine.py` | MEDIUM | MEDIUM | 1d |
| `engine/artifact_generator.py` | PRODUCT (AISF) | `products/ai-software-factory/engine/artifact_generator.py` | HIGH | HIGH | 2d |
| `engine/infra_patterns.py` | PRODUCT (AISF) | `products/ai-software-factory/engine/infra_patterns.py` | LOW | LOW | 0.5d |
| `engine/org_engine.py` | PRODUCT (AISF) | `products/ai-software-factory/engine/org_engine.py` | MEDIUM | LOW | 1d |
| `engine/employee_engine.py` | PRODUCT (AISF) | `products/ai-software-factory/engine/employee_engine.py` | MEDIUM | LOW | 1d |
| `engine/docker_service.py` | PRODUCT (AISF) | `products/ai-software-factory/engine/docker_service.py` | LOW | LOW | 0.5d |
| `engine/root_cause_engine.py` | PRODUCT (AISF) | `products/ai-software-factory/engine/root_cause_engine.py` | MEDIUM | MEDIUM | 1d |

---

### 1.3 worker/ — Worker Registry

**Current path:** `ai-software-factory/worker/`  
**CRITICAL COUPLING:** This directory contains both AISF product workers AND Mythic Realms product workers embedded in the platform repository.

| Module | Category | Current Owner | Target | Complexity | Breaking Risk | Effort |
|--------|----------|--------------|--------|-----------|--------------|--------|
| `worker/architect_worker.py` | PRODUCT (AISF) | AISF | `products/ai-software-factory/workers/architect_worker.py` | LOW | HIGH (dispatcher coupling) | 1d |
| `worker/auth_worker.py` | PRODUCT (AISF) | AISF | `products/ai-software-factory/workers/auth_worker.py` | LOW | HIGH | 0.5d |
| `worker/qa_worker.py` | PRODUCT (AISF) | AISF | `products/ai-software-factory/workers/qa_worker.py` | LOW | HIGH | 0.5d |
| `worker/security_worker.py` | PRODUCT (AISF) | AISF | `products/ai-software-factory/workers/security_worker.py` | LOW | HIGH | 0.5d |
| `worker/release_worker.py` | PRODUCT (AISF) | AISF | `products/ai-software-factory/workers/release_worker.py` | LOW | HIGH | 0.5d |
| `worker/battle_worker.py` | PRODUCT **(MYTHIC REALMS)** | MR | `products/mythic-realms/workers/battle_worker.py` | LOW | CRITICAL (wrong repo) | 1d |
| `worker/economy_worker.py` | PRODUCT **(MYTHIC REALMS)** | MR | `products/mythic-realms/workers/economy_worker.py` | LOW | CRITICAL (wrong repo) | 1d |
| `worker/collection_worker.py` | PRODUCT **(MYTHIC REALMS)** | MR | `products/mythic-realms/workers/collection_worker.py` | LOW | CRITICAL (wrong repo) | 1d |
| `worker/liveops_worker.py` | PRODUCT **(MYTHIC REALMS)** | MR | `products/mythic-realms/workers/liveops_worker.py` | LOW | CRITICAL (wrong repo) | 1d |
| `worker/player_worker.py` | PRODUCT **(MYTHIC REALMS)** | MR | `products/mythic-realms/workers/player_worker.py` | LOW | CRITICAL (wrong repo) | 1d |
| `worker/__init__.py` | INFRA | Both | `platform/workflow-runtime/worker_registry.py` | HIGH | CRITICAL | 2d |

**Note on `worker/__init__.py`:** This file is the worker registry — it imports ALL workers and registers them. After separation, the platform will have a plugin-based worker registry. Products register their workers via configuration rather than by being imported in a shared `__init__.py`. The new pattern is `factory.yaml` worker declarations in each product.

---

### 1.4 factory/ — SDK, Providers, Infrastructure

| Module | Category | Target | Complexity | Breaking Risk | Effort |
|--------|----------|--------|-----------|--------------|--------|
| `factory/sdk/plugin_base.py` | SDK | `sdk/python/aisf/plugin_base.py` | LOW | HIGH (foundation of MR workers) | 1d |
| `factory/sdk/worker_base.py` | SDK | `sdk/python/aisf/worker_base.py` | LOW | HIGH (foundation of all workers) | 1d |
| `factory/sdk/api.py` | SDK | `sdk/python/aisf/api.py` | LOW | MEDIUM | 0.5d |
| `factory/providers/base.py` | PLATFORM | `platform/provider-runtime/base.py` | LOW | MEDIUM | 0.5d |
| `factory/providers/anthropic_provider.py` | PLATFORM | `platform/provider-runtime/anthropic.py` | LOW | LOW | 0.25d |
| `factory/providers/claude_code_provider.py` | PLATFORM | `platform/provider-runtime/claude_code.py` | MEDIUM | MEDIUM | 0.5d |
| `factory/providers/openai_provider.py` | PLATFORM | `platform/provider-runtime/openai.py` | LOW | LOW | 0.25d |
| `factory/providers/gemini_provider.py` | PLATFORM | `platform/provider-runtime/gemini.py` | LOW | LOW | 0.25d |
| `factory/providers/ollama_provider.py` | PLATFORM | `platform/provider-runtime/ollama.py` | LOW | LOW | 0.25d |
| `factory/providers/router.py` | PLATFORM | `platform/provider-runtime/router.py` | MEDIUM | MEDIUM | 0.5d |
| `factory/logging/__init__.py` | PLATFORM | `platform/metrics/logging/__init__.py` | LOW | MEDIUM | 0.5d |
| `factory/logging/json_formatter.py` | PLATFORM | `platform/metrics/logging/json_formatter.py` | LOW | LOW | 0.25d |
| `factory/logging/context.py` | PLATFORM | `platform/metrics/logging/context.py` | LOW | LOW | 0.25d |
| `factory/logging/setup.py` | PLATFORM | `platform/metrics/logging/setup.py` | LOW | LOW | 0.25d |
| `factory/metrics/__init__.py` | PLATFORM | `platform/metrics/prometheus/__init__.py` | LOW | MEDIUM | 0.5d |
| `factory/metrics/registry.py` | PLATFORM | `platform/metrics/prometheus/registry.py` | LOW | LOW | 0.25d |
| `factory/metrics/http_middleware.py` | PLATFORM | `platform/metrics/prometheus/middleware.py` | LOW | LOW | 0.25d |
| `factory/metrics/setup.py` | PLATFORM | `platform/metrics/prometheus/setup.py` | LOW | LOW | 0.25d |
| `factory/metrics/system_collector.py` | PLATFORM | `platform/metrics/prometheus/system.py` | LOW | LOW | 0.25d |
| `factory/tracing/__init__.py` | PLATFORM | `platform/metrics/tracing/__init__.py` | LOW | LOW | 0.25d |
| `factory/tracing/setup.py` | PLATFORM | `platform/metrics/tracing/setup.py` | LOW | LOW | 0.25d |
| `factory/tracing/propagation.py` | PLATFORM | `platform/metrics/tracing/propagation.py` | LOW | LOW | 0.25d |
| `factory/storage/` (8 files) | PLATFORM | `platform/storage/` | MEDIUM | MEDIUM | 2d |
| `factory/assets/` (5 files) | PLATFORM | `platform/storage/assets/` | LOW | LOW | 1d |
| `factory/memory/` (2 files) | PLATFORM | `platform/central-brain/memory/` | LOW | LOW | 0.5d |
| `factory/plugins/` (7 files) | SDK | `sdk/python/aisf/plugins/` | LOW | LOW | 1d |
| `factory/cli/generator.py` | UTILITY | `tools/generator/generator.py` | LOW | LOW | 0.5d |
| `factory/hooks/registry.py` | UTILITY | `tools/generator/hooks.py` | LOW | LOW | 0.5d |
| `factory/versioning.py` | UTILITY | `platform/configuration/versioning.py` | LOW | LOW | 0.25d |

---

### 1.5 runtime/ — Execution Runtime

| Module | Category | Target | Complexity | Breaking Risk | Effort |
|--------|----------|--------|-----------|--------------|--------|
| `runtime/claude_runtime.py` | PLATFORM | `platform/provider-runtime/claude_runtime.py` | MEDIUM | HIGH | 1d |
| `runtime/claude_executor.py` | PLATFORM | `platform/provider-runtime/claude_executor.py` | MEDIUM | HIGH | 1d |
| `runtime/provider_runtime.py` | PLATFORM | `platform/provider-runtime/runtime.py` | MEDIUM | MEDIUM | 1d |
| `runtime/execution_flow.py` | PLATFORM | `platform/workflow-runtime/execution_flow.py` | MEDIUM | MEDIUM | 1d |
| `runtime/git_manager.py` | PRODUCT (AISF) | `products/ai-software-factory/runtime/git_manager.py` | LOW | MEDIUM | 0.5d |
| `runtime/build_runner.py` | PRODUCT (AISF) | `products/ai-software-factory/runtime/build_runner.py` | MEDIUM | MEDIUM | 1d |
| `runtime/lib/` (8 files) | INFRA | `platform/api/management/` | LOW | LOW | 1d |

---

### 1.6 db/ — Data Layer

| Module | Category | Target | Complexity | Breaking Risk | Effort |
|--------|----------|--------|-----------|--------------|--------|
| `db/engine.py` | PLATFORM | `platform/api/db/engine.py` | LOW | HIGH (imported everywhere) | 0.5d |
| `db/models.py` | PLATFORM | Split across schemas per DATABASE-CATALOG | CRITICAL | CRITICAL | 5d |
| `db/init_db.py` | INFRA | `platform/api/db/init_db.py` | LOW | MEDIUM | 0.5d |
| `db/migrator.py` | UTILITY | `tools/migration-tools/migrator.py` | LOW | LOW | 0.5d |
| `db/pg_config.py` | PLATFORM | `platform/api/db/pg_config.py` | LOW | LOW | 0.25d |
| `alembic/` | PLATFORM | `platform/api/alembic/` | LOW | MEDIUM | 0.5d |

**Special note on db/models.py:** This single file contains 54 SQLAlchemy model classes spanning platform (WorkflowInstance, BrainExperience), AISF product (Product, ProductArtifact, Employee), and Mythic Realms concerns. Per DATABASE-CATALOG, models must be split into 12 schemas. This is the highest-complexity single extraction task. It requires:
1. Splitting into schema-namespaced files
2. Updating every import reference (every engine/ module imports from db.models)
3. Verifying 12 database migrations still apply correctly

---

### 1.7 Top-Level Modules

| Module | Category | Target | Complexity | Breaking Risk | Effort |
|--------|----------|--------|-----------|--------------|--------|
| `app.py` | INFRA | `platform/api/app.py` | HIGH | CRITICAL (entrypoint) | 2d |
| `config.py` | PLATFORM | `platform/configuration/settings.py` | MEDIUM | HIGH (prefix migration) | 1d |
| `supervisor.py` | PLATFORM | `platform/workflow-runtime/worker_supervisor.py` | MEDIUM | HIGH | 1d |
| `agent_runtime.py` | PRODUCT (AISF) | `products/ai-software-factory/agent_runtime.py` | LOW | LOW | 0.5d |
| `platform_version.py` | PLATFORM | `platform/configuration/version.py` | LOW | LOW | 0.25d |

---

### 1.8 Other Directories

| Directory | Category | Decision | Effort |
|-----------|----------|---------|--------|
| `installer/` | INFRA | Move to `tools/installer/` — separate from platform runtime | 1d |
| `dashboard/` | PRODUCT (AISF) | Move to `products/ai-software-factory/dashboard/` | 0.5d |
| `scripts/` | UTILITY | Move to `tools/scripts/` | 0.25d |
| `templates/` | SDK | Move to `sdk/python/templates/` | 0.25d |
| `build/` | GENERATED | Delete — regenerated by build process | 0d |
| `dist/` | GENERATED | Delete — regenerated by release | 0d |
| `.venv/` | INFRA | Do not migrate — each repo has its own venv | 0d |
| `logs/` | RUNTIME | Do not migrate — runtime output | 0d |
| `storage/` | RUNTIME | Do not migrate — local data | 0d |
| `reports/` | RUNTIME | Do not migrate — generated output | 0d |
| `production/sprint1_runner.py` | LEGACY | Archive — not part of production platform | 0.25d |

---

## Section 2: ai-studio-desktop

**Assessment: LOW RISK — Minimal changes required**

The desktop is correctly positioned as a standalone application. It communicates with the platform over HTTP only. No Python import coupling to platform internals.

| Directory/Module | Category | Target | Complexity | Breaking Risk | Effort |
|-----------------|----------|--------|-----------|--------------|--------|
| `app/` (11 files) | DESKTOP | `desktop/ai-studio-desktop/app/` | LOW | LOW | 0d (move only) |
| `bootstrap/` (13 files) | DESKTOP | `desktop/ai-studio-desktop/bootstrap/` | LOW | LOW | 0d |
| `controllers/` (55 files) | DESKTOP | `desktop/ai-studio-desktop/controllers/` | LOW | LOW | 0d |
| `dialogs/` (7 files) | DESKTOP | `desktop/ai-studio-desktop/dialogs/` | LOW | LOW | 0d |
| `models/` (19 files) | DESKTOP | `desktop/ai-studio-desktop/models/` | LOW | LOW | 0d |
| `services/api_client.py` | DESKTOP | `desktop/ai-studio-desktop/services/` | LOW | LOW | 0d |
| `services/workspace_service.py` | DESKTOP | needs workspace.yaml path update | LOW | LOW | 0.25d |
| `services/*.py` (all 60 client files) | DESKTOP | `desktop/ai-studio-desktop/services/` | LOW | LOW | 0d |
| `ui/` (all panels) | DESKTOP | `desktop/ai-studio-desktop/ui/` | LOW | LOW | 0d |
| `widgets/` (14 files) | DESKTOP | `desktop/ai-studio-desktop/widgets/` | LOW | LOW | 0d |
| `plugins/` | DESKTOP | `desktop/ai-studio-desktop/plugins/` | LOW | LOW | 0d |
| `tests/` | DESKTOP | `desktop/ai-studio-desktop/tests/` | LOW | LOW | 0d |
| `main.py` | DESKTOP | `desktop/ai-studio-desktop/main.py` | LOW | LOW | 0d |

**One-time change required:**  
`services/workspace_service.py` line `Path(__file__).parents[3] / "workspace.yaml"` — parent count changes when the desktop moves into `desktop/ai-studio-desktop/`. This is a 1-line fix. Estimated effort: 0.25d.

**Total desktop extraction effort:** 0.25 engineer-days

---

## Section 3: content-factory

**Assessment: ZERO RISK — Organizational move only**

Content Factory is a standalone Java/Spring Boot application. It has no dependency on the Python platform. Moving it from `source/content-factory/` to `products/content-factory/` is a directory rename with workspace.yaml update.

| Action | Effort |
|--------|--------|
| Move directory from `source/` to `products/` | 0d (git mv) |
| Update `workspace.yaml` paths | 0.25d |
| Update any CI/CD scripts referencing the old path | 0.25d |

**Total content-factory extraction effort:** 0.5 engineer-days

---

## Section 4: mythic-realms

**Assessment: HIGH RISK — Multiple coupling violations to resolve**

| Directory/Module | Category | Current Issue | Target | Complexity | Breaking Risk | Effort |
|-----------------|----------|--------------|--------|-----------|--------------|--------|
| `ai-software-factory/` (entire dir) | SDK (vendored copy) | **COUPLING C-002**: vendored SDK; will drift | Delete after SDK package is installable | MEDIUM | HIGH | 2d |
| `tools/orchestrator/` (entire dir) | PRODUCT (forked platform) | **COUPLING C-003**: fork of platform | Migrate to use platform as installed package | HIGH | HIGH | 5d |
| `tools/orchestrator/worker/__init__.py` | PRODUCT | Same workers as AISF `worker/__init__.py` | Keep in MR, update SDK import path | MEDIUM | HIGH | 1d |
| `tools/orchestrator/worker/battle_worker.py` | PRODUCT (MR) | Duplicate in AISF worker/ | Keep in MR only; remove from AISF | LOW | MEDIUM | 0.5d |
| `tools/orchestrator/worker/economy_worker.py` | PRODUCT (MR) | Same | Keep in MR only | LOW | MEDIUM | 0.5d |
| `tools/orchestrator/worker/collection_worker.py` | PRODUCT (MR) | Same | Keep in MR only | LOW | MEDIUM | 0.5d |
| `tools/orchestrator/worker/liveops_worker.py` | PRODUCT (MR) | Same | Keep in MR only | LOW | MEDIUM | 0.5d |
| `tools/orchestrator/worker/player_worker.py` | PRODUCT (MR) | Same | Keep in MR only | LOW | MEDIUM | 0.5d |
| `backend/` | PRODUCT (MR) | Correct placement | `products/mythic-realms/backend/` | LOW | LOW | 0d |
| `client/` | PRODUCT (MR) | Correct placement | `products/mythic-realms/client/` | LOW | LOW | 0d |
| `database/` | PRODUCT (MR) | Correct placement | `products/mythic-realms/database/` | LOW | LOW | 0d |
| `agents/` | PRODUCT (MR) | Correct placement | `products/mythic-realms/agents/` | LOW | LOW | 0d |
| `docs/` | PRODUCT (MR) | Correct placement | `products/mythic-realms/docs/` | LOW | LOW | 0d |

**Critical path for mythic-realms:**
1. SDK package must be publishable (or installable from path) before vendored copy can be removed
2. Platform must be installable (or importable from path) before orchestrator fork can be replaced
3. Until then, MR continues to use its vendored copies (no regression)

**Total mythic-realms extraction effort:** ~11 engineer-days (mostly orchestrator migration)

---

## Section 5: Extraction Matrix Summary

### 5.1 Total Effort by Phase

| Phase | Modules | Effort (engineer-days) |
|-------|---------|----------------------|
| Phase 1 — Foundation | 15 modules | 4d |
| Phase 2 — Infrastructure Services | 12 modules | 5d |
| Phase 3 — Core Platform Services | 18 modules | 8d |
| Phase 4 — Intelligence Services | 10 modules | 6d |
| Phase 5 — Products + Desktop | 20 modules | 8d |
| Phase 6 — SDK, Tools, Cleanup | 15 modules | 4d |
| **Total** | **90 modules** | **35d (~7 weeks)** |

### 5.2 Risk Distribution

| Risk Level | Module Count | Notes |
|-----------|-------------|-------|
| CRITICAL | 3 modules | db/models.py (schema split), worker/__init__.py (registry redesign), app.py (entrypoint) |
| HIGH | 12 modules | All dispatcher-dependent modules; auth middleware; product_factory.py |
| MEDIUM | 35 modules | engine/ platform modules with db.models coupling |
| LOW | 40 modules | Most factory/, monitoring, utility modules |

### 5.3 Modules That Cannot Move Until Others Move First

```
db/models.py (schema split)
    → must complete before: any engine/* module moves
        → must complete before: api/* modules move
            → must complete before: app.py moves

factory/sdk/* (SDK package)
    → must complete before: mythic-realms ai-software-factory/ dir deleted
        → must complete before: MR orchestrator fork replaced

engine/dispatcher.py (worker registry decoupled)
    → must complete before: worker/* modules move out
```

---

*End of REPOSITORY-EXTRACTION-MATRIX.md*  
*Version 1.0.0 | Status: PROPOSED | 90 modules classified | 35d estimated effort*
