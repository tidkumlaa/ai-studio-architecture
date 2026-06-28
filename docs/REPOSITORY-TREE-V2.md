# AI Studio Platform — Repository Tree v2

**Document ID:** REPOSITORY-TREE-V2  
**Version:** 1.0.0  
**Date:** 2026-06-28  
**Status:** PROPOSED — Target State After Phase 0B Extraction  
**Authority:** Chief Software Architect  
**Companion:** REPOSITORY-SEPARATION-PLAN.md, REPOSITORY-EXTRACTION-MATRIX.md

---

## Overview

This document shows the complete target directory tree for every repository and workspace directory after Phase 0B extraction is complete. Use this as the authoritative reference for where files should end up.

**Legend:**
```
/           — directory
*.py        — Python module
*.toml      — package configuration
*.yaml      — configuration/manifest
*.md        — documentation
[NEW]       — does not exist yet; must be created
[MOVED]     — moved from current location
[EXTRACTED] — module content moved into this file from a larger source
[SPLIT]     — split from a single file into multiple
[SHIM]      — backward compatibility re-export; deleted in Phase 6
[DELETE]    — deleted during extraction
```

---

## 1. Workspace Root

```
E:/UserData/MyData/Content/AIStudio/
├── workspace.yaml                          [UPDATED v3.0]
├── architecture/                           [UNCHANGED]
│   └── docs/
│       ├── AI-STUDIO-PLATFORM-REFACTORING-BLUEPRINT.md
│       ├── PLATFORM-STANDARDS.md
│       ├── PLATFORM-CONTRACTS.md
│       ├── EVENT-CATALOG.md
│       ├── API-CATALOG.md
│       ├── DATABASE-CATALOG.md
│       ├── SECURITY-MODEL.md
│       ├── REPOSITORY-SEPARATION-PLAN.md   [NEW — this phase]
│       ├── REPOSITORY-EXTRACTION-MATRIX.md [NEW — this phase]
│       ├── REPOSITORY-DEPENDENCY-RULES.md  [NEW — this phase]
│       ├── COUPLING-ANALYSIS.md            [NEW — this phase]
│       ├── PLATFORM-EXTRACTION-ROADMAP.md  [NEW — this phase]
│       ├── REPOSITORY-MIGRATION-CHECKLIST.md [NEW — this phase]
│       └── REPOSITORY-TREE-V2.md           [NEW — this document]
│
├── platform/                               [NEW — extracted from ai-software-factory]
│   └── (see Section 2)
│
├── products/                               [NEW]
│   ├── ai-software-factory/               [EXTRACTED]
│   ├── content-factory/                   [MOVED from source/]
│   └── mythic-realms/                     [MOVED from source/]
│
├── desktop/                               [NEW]
│   └── ai-studio-desktop/                 [MOVED from source/]
│
├── sdk/                                   [NEW]
│   ├── python/                            [EXTRACTED from factory/sdk/]
│   ├── java/                              [NEW — stub]
│   └── typescript/                        [NEW — stub]
│
├── tools/                                 [NEW]
│   ├── generator/                         [EXTRACTED from factory/cli/]
│   ├── architecture-linter/               [NEW]
│   └── migration-tools/                   [EXTRACTED from db/migrator.py]
│
├── examples/                              [NEW]
│   ├── python-project/                    [from factory/templates/python/]
│   ├── java-project/                      [from factory/templates/springboot/]
│   ├── flutter-project/                   [from factory/templates/flutter/]
│   └── ai-agent/                          [from factory/templates/ai-agent/]
│
├── marketplace/                           [NEW — stub]
│   └── catalog/
│
└── playground/                            [NEW — empty]
```

---

## 2. Platform Directory Tree

```
platform/                                  [NEW root for extracted platform]
│
├── configuration/                         [EXTRACTED]
│   ├── __init__.py
│   ├── settings.py                        [from config.py — AISF_ prefix]
│   └── version.py                         [from platform_version.py]
│
├── metrics/                               [EXTRACTED from factory/]
│   ├── __init__.py
│   ├── logging/
│   │   ├── __init__.py                    [from factory/logging/__init__.py]
│   │   ├── json_formatter.py
│   │   ├── context.py
│   │   └── setup.py
│   ├── prometheus/
│   │   ├── __init__.py                    [from factory/metrics/__init__.py]
│   │   ├── registry.py
│   │   ├── http_middleware.py
│   │   ├── setup.py
│   │   └── system_collector.py
│   └── tracing/
│       ├── __init__.py                    [from factory/tracing/__init__.py]
│       ├── setup.py
│       └── propagation.py
│
├── event-bus/                             [EXTRACTED from engine/]
│   ├── __init__.py
│   ├── in_process.py                      [from engine/event_bus.py]
│   ├── nats_client.py                     [from engine/nats_client.py]
│   └── schemas.py                         [from engine/event_schemas.py]
│
├── security/                              [EXTRACTED from engine/security.py + api/auth.py]
│   ├── __init__.py
│   ├── auth_middleware.py                 [from api/auth.py — auth required by default]
│   ├── rbac.py                            [from engine/security.py — RBACManager]
│   ├── rate_limiter.py                    [from engine/security.py — RateLimiter]
│   └── prompt_validator.py               [from engine/security.py — PromptValidator]
│
├── storage/                               [EXTRACTED from factory/storage/]
│   ├── __init__.py
│   ├── spi.py
│   ├── service.py
│   ├── model.py
│   ├── inventory.py
│   ├── streaming.py
│   ├── exceptions.py
│   ├── security.py
│   ├── assets/                            [from factory/assets/]
│   │   ├── __init__.py
│   │   ├── inventory.py
│   │   ├── model.py
│   │   ├── registry.py
│   │   ├── service.py
│   │   └── uri.py
│   └── providers/
│       ├── __init__.py
│       ├── local.py
│       ├── s3.py
│       ├── azure.py
│       ├── gcs.py
│       └── minio.py
│
├── provider-runtime/                      [EXTRACTED from factory/providers/ + runtime/]
│   ├── __init__.py
│   ├── base.py                            [from factory/providers/base.py]
│   ├── runtime.py                         [from runtime/provider_runtime.py]
│   ├── claude_runtime.py                  [from runtime/claude_runtime.py]
│   ├── claude_executor.py                 [from runtime/claude_executor.py]
│   ├── claude_code.py                     [from factory/providers/claude_code_provider.py]
│   ├── anthropic.py                       [from factory/providers/anthropic_provider.py]
│   ├── openai.py                          [from factory/providers/openai_provider.py]
│   ├── gemini.py                          [from factory/providers/gemini_provider.py]
│   ├── ollama.py                          [from factory/providers/ollama_provider.py]
│   └── router.py                          [from factory/providers/router.py]
│
├── workflow-runtime/                      [EXTRACTED from engine/ + root]
│   ├── __init__.py
│   ├── engine.py                          [from engine/workflow_engine.py]
│   ├── runtime.py                         [from engine/workflow_runtime.py]
│   ├── supervisor.py                      [from engine/workflow_supervisor.py]
│   ├── worker_supervisor.py               [from supervisor.py]
│   ├── worker_registry.py                 [NEW — plugin-based registry]
│   ├── dispatcher.py                      [from engine/dispatcher.py — refactored]
│   ├── dependency_resolver.py             [from engine/dependency_resolver.py]
│   ├── blocker_monitor.py                 [from engine/blocker_monitor.py]
│   └── execution_flow.py                  [from runtime/execution_flow.py]
│
├── prompt-os/                             [EXTRACTED from engine/prompt_os/ + engine/]
│   ├── __init__.py
│   ├── orchestrator.py                    [from engine/prompt_os/orchestrator.py]
│   ├── template.py                        [from engine/prompt_os/template.py]
│   ├── governance.py                      [from engine/prompt_os/governance.py]
│   ├── execution.py                       [from engine/prompt_os/execution.py]
│   ├── composition.py                     [from engine/prompt_os/composition.py]
│   ├── context_manager.py                 [from engine/prompt_os/context.py]
│   ├── brain_bridge.py                    [from engine/prompt_os/brain_bridge.py]
│   ├── security.py                        [from engine/prompt_os/security.py]
│   ├── metrics.py                         [from engine/prompt_os/metrics.py]
│   ├── marketplace.py                     [from engine/prompt_os/marketplace.py]
│   ├── _db.py                             [from engine/prompt_os/_db.py]
│   ├── runtime.py                         [from engine/prompt_runtime.py]
│   ├── intelligence.py                    [from engine/prompt_intelligence.py]
│   ├── context_engine.py                  [from engine/context_engine.py]
│   └── conversation_engine.py            [from engine/conversation_engine.py]
│
├── central-brain/                         [EXTRACTED from engine/]
│   ├── __init__.py
│   ├── central_brain.py                   [from engine/central_brain.py]
│   ├── brain.py                           [from engine/brain.py]
│   ├── memory_os.py                       [from engine/memory_os.py]
│   ├── experience_recorder.py             [from engine/experience_recorder.py]
│   ├── outcome_collector.py               [from engine/outcome_collector.py]
│   ├── learning_agent.py                  [from engine/learning_agent.py]
│   ├── self_improvement.py                [from engine/self_improvement.py]
│   ├── failure_analyzer.py                [from engine/failure_analyzer.py]
│   └── memory/                            [from factory/memory/]
│       ├── __init__.py
│       └── service.py
│
├── decision-engine/                       [EXTRACTED from engine/]
│   ├── __init__.py
│   └── engine.py                          [from engine/decision_engine.py]
│
├── knowledge/                             [EXTRACTED from engine/central_brain.py]
│   ├── __init__.py
│   ├── graph.py                           [GraphEngine — Kuzu migration target]
│   └── node_types.py                      [Node type definitions]
│
├── ai-runtime/                            [EXTRACTED from engine/]
│   ├── __init__.py
│   ├── router.py                          [from engine/ai_router.py]
│   ├── model_registry.py                  [from engine/model_registry.py]
│   ├── execution.py                       [from engine/ai_execution.py]
│   └── cost_engine.py                     [from engine/cost_engine.py]
│
├── plugin-runtime/                        [EXTRACTED from engine/]
│   ├── __init__.py
│   └── tool_runtime.py                    [from engine/tool_runtime.py]
│
├── review-engine/                         [EXTRACTED from engine/]
│   ├── __init__.py
│   ├── review_router.py                   [from engine/review_router.py]
│   └── merge_manager.py                   [from engine/merge_manager.py]
│
├── governance/                            [EXTRACTED from engine/]
│   ├── __init__.py
│   └── approval_service.py                [from engine/approval_service.py]
│
├── observability/                         [EXTRACTED from engine/]
│   ├── __init__.py
│   └── sla_monitor.py                     [from engine/sla_monitor.py]
│
├── workspace/                             [NEW — workspace manifest management]
│   ├── __init__.py
│   └── capabilities.py                    [from api/capabilities_routes.py logic]
│
└── api/                                   [EXTRACTED from api/ + db/ + app.py]
    ├── __init__.py
    ├── app.py                             [from app.py — factory function]
    ├── schemas.py                         [from api/schemas.py]
    ├── correlation_middleware.py          [from api/correlation_middleware.py]
    ├── routes.py                          [from api/routes.py]
    ├── websocket_routes.py                [from api/ws_routes.py]
    ├── proxy_routes.py                    [from api/proxy_routes.py]
    ├── health_routes.py                   [from api/health_routes.py]
    ├── capabilities_routes.py             [from api/capabilities_routes.py]
    ├── runtime_routes.py                  [from api/runtime_routes.py]
    ├── db_routes.py                       [from api/db_routes.py]
    ├── db/
    │   ├── __init__.py
    │   ├── engine.py                      [from db/engine.py]
    │   ├── init_db.py                     [from db/init_db.py]
    │   ├── pg_config.py                   [from db/pg_config.py]
    │   └── models/
    │       ├── __init__.py                [re-exports — deleted Phase 6]
    │       ├── base.py
    │       ├── workflow.py                [SPLIT from db/models.py]
    │       ├── prompt.py                  [SPLIT]
    │       ├── provider.py                [SPLIT]
    │       ├── ai_runtime.py              [SPLIT]
    │       └── brain.py                   [SPLIT]
    ├── management/                        [from runtime/lib/]
    │   ├── __init__.py
    │   ├── start.py
    │   ├── stop.py
    │   ├── restart.py
    │   ├── status.py
    │   ├── logs.py
    │   ├── resume.py
    │   ├── update.py
    │   └── doctor.py
    └── alembic/                           [from alembic/]
        ├── env.py
        └── versions/
            └── *.py
```

---

## 3. Products Directory Tree

### 3.1 products/ai-software-factory/

```
products/ai-software-factory/              [EXTRACTED + REORGANIZED]
│
├── factory.yaml                           [NEW — product manifest + worker registry]
├── pyproject.toml                         [UPDATED — depends on aisf-sdk + platform]
│
├── engine/                                [EXTRACTED from ai-software-factory/engine/]
│   ├── __init__.py
│   ├── product_factory.py                 [from engine/product_factory.py]
│   ├── product_intake.py                  [from engine/product_intake.py]
│   ├── business_analyst.py                [from engine/business_analyst.py]
│   ├── architect_agent.py                 [from engine/architect_agent.py]
│   ├── planner_engine.py                  [from engine/planner_engine.py]
│   ├── artifact_generator.py             [from engine/artifact_generator.py]
│   ├── infra_patterns.py                  [from engine/infra_patterns.py]
│   ├── org_engine.py                      [from engine/org_engine.py]
│   ├── employee_engine.py                 [from engine/employee_engine.py]
│   ├── docker_service.py                  [from engine/docker_service.py]
│   └── root_cause_engine.py              [from engine/root_cause_engine.py]
│
├── workers/                               [EXTRACTED from worker/]
│   ├── __init__.py
│   ├── architect_worker.py                [from worker/architect_worker.py]
│   ├── auth_worker.py                     [from worker/auth_worker.py]
│   ├── qa_worker.py                       [from worker/qa_worker.py]
│   ├── security_worker.py                 [from worker/security_worker.py]
│   └── release_worker.py                  [from worker/release_worker.py]
│
├── api/                                   [EXTRACTED from api/]
│   ├── __init__.py
│   ├── product_routes.py                  [from api/product_routes.py]
│   ├── improvement_routes.py             [from api/improvement_routes.py]
│   ├── org_routes.py                      [from api/org_routes.py]
│   ├── employee_routes.py                 [from api/employee_routes.py]
│   ├── agent_metrics_routes.py           [from api/agent_metrics_routes.py]
│   ├── git_routes.py                      [from api/git_routes.py]
│   └── docker_routes.py                   [from api/docker_routes.py]
│
├── runtime/                               [from runtime/]
│   ├── __init__.py
│   ├── git_manager.py                     [from runtime/git_manager.py]
│   └── build_runner.py                    [from runtime/build_runner.py]
│
├── dashboard/                             [from dashboard/]
│   ├── __init__.py
│   └── builder.py
│
├── db/
│   └── models/
│       └── aisf_product.py               [SPLIT from db/models.py — AISF-specific models]
│
├── startup.py                             [NEW — registers routes + workers with platform]
│
└── tests/                                 [from tests/ — product-specific tests]
    ├── test_product_factory.py
    ├── test_ai_execution_engine.py
    ├── test_employee_engine.py
    ├── test_org_engine.py
    └── ...
```

### 3.2 products/content-factory/

```
products/content-factory/                  [MOVED from source/content-factory/]
│                                          (directory structure unchanged — Java project)
├── cf-agent-core/
├── cf-agents/
├── cf-ai-router/
├── cf-analytics/
├── cf-api/
├── cf-assets/
├── cf-audio/
├── cf-character-media/
├── cf-claude/
├── cf-comfyui/
├── cf-common/
├── cf-contract/
├── cf-creative/
├── cf-director/
├── cf-elevenlabs/
├── cf-executive/
├── cf-filesystem/
├── cf-frontend/
├── cf-gemini/
├── cf-image/
├── cf-integration/
├── cf-knowledge/
├── cf-localization/
├── cf-media-asset/
├── cf-memory/
├── cf-ollama/
├── cf-openai/
├── cf-piper/
├── cf-postgres/
├── cf-prompt/
├── cf-publish/
├── cf-quality/
├── cf-rabbitmq/
├── cf-romanization/
├── cf-story/
├── cf-subtitle/
├── cf-tts/
├── cf-video/
├── cf-worker/
├── cf-workflow/
├── pom.xml
└── scripts/
```

### 3.3 products/mythic-realms/

```
products/mythic-realms/                    [MOVED from source/mythic-realms/ + REORGANIZED]
│
├── factory.yaml                           [NEW — product manifest + worker registry]
│
├── backend/                               [Java Spring Boot backend — UNCHANGED]
│   └── src/
│
├── client/                                [Game client — UNCHANGED]
│
├── database/                              [DB scripts — UNCHANGED]
│
├── ddl/                                   [Schema DDL — UNCHANGED]
│
├── agents/                                [Agent prompts — UNCHANGED]
│   ├── BALANCE-AGENT.md
│   ├── GAME-DIRECTOR.md
│   ├── LIVEOPS-AGENT.md
│   └── MARKETING-AGENT.md
│
├── docs/                                  [Game docs — UNCHANGED]
│
├── orchestrator/                          [REPLACED tools/orchestrator/ — NEW thin launcher]
│   ├── __init__.py
│   ├── app.py                             [NEW — thin wrapper: create_platform_app() + workers]
│   └── requirements.txt                   [UPDATED: depends on platform package, aisf-sdk]
│
├── workers/                               [MOVED from tools/orchestrator/worker/]
│   ├── __init__.py
│   ├── architect_worker.py
│   ├── auth_worker.py
│   ├── battle_worker.py
│   ├── collection_worker.py
│   ├── economy_worker.py
│   ├── liveops_worker.py
│   ├── player_worker.py
│   ├── qa_worker.py
│   ├── release_worker.py
│   └── security_worker.py
│
├── localization/                          [Game localization — UNCHANGED]
│
├── specs/                                 [Game specs — UNCHANGED]
│
├── k8s/                                   [Kubernetes manifests — UNCHANGED]
│
└── tests/                                 [UNCHANGED]
```

**Deleted from mythic-realms:**
```
[DELETE] ai-software-factory/factory/      — vendored SDK (replaced by aisf-sdk package)
[DELETE] tools/orchestrator/               — forked platform (replaced by orchestrator/ above)
```

---

## 4. Desktop Directory Tree

```
desktop/
└── ai-studio-desktop/                     [MOVED from source/ai-studio-desktop/]
    │                                      (internal structure UNCHANGED except path fix)
    ├── app/
    │   ├── application.py
    │   ├── settings.py
    │   └── ... (11 files total)
    │
    ├── bootstrap/
    │   ├── aisf_checker.py
    │   ├── launcher.py
    │   └── ... (13 files total)
    │
    ├── controllers/                        (55 files — unchanged)
    │
    ├── dialogs/                            (7 files — unchanged)
    │
    ├── models/                             (19 files — unchanged)
    │
    ├── services/
    │   ├── api_client.py                   [UNCHANGED — HTTP-only coupling]
    │   ├── workspace_service.py            [UPDATED: parents[3] → parents[4] path fix]
    │   └── ... (60 client files — unchanged)
    │
    ├── ui/                                 (all panels — unchanged)
    │
    ├── widgets/                            (14 files — unchanged)
    │
    ├── plugins/
    │
    ├── tests/                              (28 test files — unchanged)
    │
    ├── main.py
    └── pyproject.toml
```

---

## 5. SDK Directory Tree

```
sdk/
├── python/                                [NEW — extracted from factory/sdk/]
│   ├── pyproject.toml                     [name="aisf-sdk" version="1.0.0" deps=[]]
│   ├── README.md
│   └── aisf/
│       ├── __init__.py
│       ├── worker_base.py                 [from factory/sdk/worker_base.py]
│       ├── plugin_base.py                 [from factory/sdk/plugin_base.py]
│       ├── api.py                         [from factory/sdk/api.py]
│       └── plugins/
│           ├── __init__.py
│           ├── generic_plugin.py          [from factory/plugins/]
│           ├── python_plugin.py
│           ├── java_plugin.py
│           ├── cicd_plugin.py
│           └── content_factory_plugin.py
│
├── java/                                  [NEW — stub]
│   └── README.md
│
└── typescript/                            [NEW — stub]
    └── README.md
```

---

## 6. Tools Directory Tree

```
tools/
├── generator/                             [EXTRACTED from factory/cli/]
│   ├── __init__.py
│   ├── generator.py                       [from factory/cli/generator.py]
│   └── hooks.py                           [from factory/hooks/registry.py]
│
├── architecture-linter/                   [NEW]
│   ├── check_imports.py                   [enforces dependency rules]
│   ├── check_events.py                    [validates event naming conventions]
│   ├── check_config_prefix.py             [detects ORCH_ usage]
│   └── .importlinter                      [import-linter configuration]
│
└── migration-tools/                       [EXTRACTED from db/]
    ├── __init__.py
    └── migrator.py                        [from db/migrator.py]
```

---

## 7. ai-software-factory After Extraction

The original `ai-software-factory/` repository after extraction is complete — this remains the canonical platform git repository, now focused on platform code only.

```
source/ai-software-factory/               [PLATFORM — cleaned up]
│
├── platform/                             [symlink or Python path reference to ../../platform/]
│
├── api/                                  [SHIM layer during transition → deleted Phase 6]
│   ├── *.py                              [All non-product routes become shims to platform.api.*]
│   └── [product routes deleted in Phase 3]
│
├── db/                                   [CLEANED UP]
│   ├── engine.py                         [SHIM → platform.api.db.engine]
│   ├── init_db.py                        [SHIM → platform.api.db.init_db]
│   ├── pg_config.py                      [SHIM → platform.api.db.pg_config]
│   ├── migrator.py                       [MOVED to tools/migration-tools/]
│   └── models/
│       ├── __init__.py                   [SHIM — deleted Phase 6]
│       ├── base.py
│       ├── workflow.py
│       ├── prompt.py
│       ├── provider.py
│       ├── ai_runtime.py
│       ├── brain.py
│       └── [aisf_product.py → products/ai-software-factory/db/models/]
│
├── engine/                               [SHIMS ONLY — all content moved]
│   ├── workflow_engine.py                [SHIM → platform.workflow_runtime.engine]
│   ├── central_brain.py                  [SHIM → platform.central_brain.central_brain]
│   ├── security.py                       [SHIM → platform.security.rbac]
│   ├── event_bus.py                      [SHIM → platform.event_bus.in_process]
│   ├── nats_client.py                    [SHIM → platform.event_bus.nats_client]
│   ├── [product modules deleted Phase 3]
│   └── prompt_os/                        [SHIMS → platform.prompt_os.*]
│
├── factory/                              [SHIMS ONLY — all content moved]
│   ├── providers/                        [SHIMS → platform.provider_runtime.*]
│   ├── sdk/                              [SHIMS → aisf.*]
│   ├── plugins/                          [SHIMS → aisf.plugins.*]
│   ├── logging/                          [SHIMS → platform.metrics.logging.*]
│   ├── metrics/                          [SHIMS → platform.metrics.prometheus.*]
│   ├── tracing/                          [SHIMS → platform.metrics.tracing.*]
│   ├── storage/                          [SHIMS → platform.storage.*]
│   ├── memory/                           [SHIMS → platform.central_brain.memory.*]
│   ├── assets/                           [SHIMS → platform.storage.assets.*]
│   ├── cli/                              [SHIMS → tools.generator.*]
│   └── hooks/                            [SHIMS → tools.generator.hooks]
│
├── worker/                               [SHIMS ONLY]
│   ├── __init__.py                       [EMPTY — WorkerRegistry in platform now]
│   └── [all worker files moved Phase 5]
│
├── runtime/                              [SHIMS ONLY — all content moved]
│   ├── claude_runtime.py                 [SHIM → platform.provider_runtime.claude_runtime]
│   ├── claude_executor.py                [SHIM → platform.provider_runtime.claude_executor]
│   ├── provider_runtime.py               [SHIM → platform.provider_runtime.runtime]
│   ├── execution_flow.py                 [SHIM → platform.workflow_runtime.execution_flow]
│   ├── git_manager.py                    [MOVED → products/ai-software-factory/runtime/]
│   ├── build_runner.py                   [MOVED → products/ai-software-factory/runtime/]
│   └── lib/                              [SHIMS → platform.api.management.*]
│
├── installer/                            [MOVED → tools/installer/ — unchanged]
│
├── dashboard/                            [MOVED → products/ai-software-factory/dashboard/]
│
├── alembic/                              [MOVED → platform/api/alembic/ — unchanged]
│
├── tests/                                [SPLIT between platform + product]
│   ├── [platform tests remain here]
│   └── [AISF product tests moved to products/ai-software-factory/tests/]
│
├── app.py                                [UPDATED — calls platform factory + loads AISF product]
├── config.py                             [SHIM → platform.configuration.settings]
├── supervisor.py                         [SHIM → platform.workflow_runtime.worker_supervisor]
├── platform_version.py                   [SHIM → platform.configuration.version]
├── agent_runtime.py                      [MOVED → products/ai-software-factory/]
├── pyproject.toml                        [UPDATED — declares platform as dependency]
└── .importlinter                         [NEW — enforces dependency rules in this repo]
```

---

## 8. Dependency Graph (Post-Extraction)

```
                    ┌─────────────────────────────────┐
                    │  platform/                      │
                    │  (Layer 0–6 stack)              │
                    └──────────────┬──────────────────┘
                                   │ publishes API + SDK
                         ┌─────────┴─────────┐
                         │                   │
              ┌──────────▼──────┐   ┌────────▼───────┐
              │  sdk/python/    │   │  HTTP REST API  │
              │  (aisf package) │   │  (port 8088)    │
              └──┬──────────────┘   └────────┬────────┘
                 │                           │
      ┌──────────┼──────────────────────────┐│
      │          │                          ││
      ▼          ▼                          ▼▼
 products/   products/               desktop/
 ai-sw-fac/  mythic-realms/          ai-studio-desktop/
 (imports    (imports aisf.*,        (HTTP only — no imports)
  platform   registers workers
  as lib,    via factory.yaml)
  aisf SDK)
      │
      ▼
 products/
 content-factory/
 (standalone Java —
  no Python imports)
```

**Edges NOT shown above (forbidden by REPOSITORY-DEPENDENCY-RULES.md):**
- `platform/ → products/` — FORBIDDEN
- `products/X → products/Y` — FORBIDDEN  
- `sdk/ → platform/` — FORBIDDEN
- `desktop/ → platform/ (import)` — FORBIDDEN

---

## 9. File Count Summary (Post-Extraction)

| Location | Python Files | Notes |
|----------|-------------|-------|
| `platform/` | ~120 | All platform runtime modules |
| `products/ai-software-factory/` | ~30 | AISF product logic only |
| `products/content-factory/` | 0 Python | Java-only product |
| `products/mythic-realms/workers/` | ~10 | MR game workers |
| `products/mythic-realms/orchestrator/` | ~3 | Thin platform launcher |
| `desktop/ai-studio-desktop/` | ~210 | UI + clients (unchanged) |
| `sdk/python/aisf/` | ~12 | SDK package (zero deps) |
| `tools/` | ~8 | Generator, linter, migration tools |
| `source/ai-software-factory/` shims | ~60 | Backward compat shims (deleted Phase 6) |
| **Total** | **~453** | |

---

## 10. workspace.yaml v3.0 Key Changes

```yaml
# workspace.yaml v3.0 (post-extraction)
workspace:
  name: "AI Studio"
  version: "3.0"
  migration_phase: 3

repositories:
  # PLATFORM (single source of truth for platform runtime)
  platform:
    path: "E:/UserData/MyData/Content/AIStudio/platform"
    type: "platform"
    language: "python"
    
  # SOURCE REPO (now contains only product code + shims)
  ai-software-factory:
    path: "E:/UserData/MyData/Content/AIStudio/source/ai-software-factory"
    type: "platform-source"  # still the git repo for platform code
    
  # PRODUCTS
  products/ai-software-factory:
    path: "E:/UserData/MyData/Content/AIStudio/products/ai-software-factory"
    type: "product"
    
  products/content-factory:
    path: "E:/UserData/MyData/Content/AIStudio/products/content-factory"
    type: "product"
    
  products/mythic-realms:
    path: "E:/UserData/MyData/Content/AIStudio/products/mythic-realms"
    type: "product"
    
  # DESKTOP
  desktop/ai-studio-desktop:
    path: "E:/UserData/MyData/Content/AIStudio/desktop/ai-studio-desktop"
    type: "desktop"
    
  # SDK
  sdk/python:
    path: "E:/UserData/MyData/Content/AIStudio/sdk/python"
    type: "sdk"
```

---

*End of REPOSITORY-TREE-V2.md*  
*Version 1.0.0 | Status: PROPOSED | Complete target state for all repositories post-extraction*
