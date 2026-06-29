---
knowledge_id: KNW-PLAT-ARCH-005
title: "Platform Module Catalog"
status: AUTHORITATIVE
phase: "2.1D.0"
version: "1.0.0"
created: "2026-06-29"
purpose: "Complete catalog of every module in every runtime with identity, capabilities, and status"
canonical_source: "architecture/docs/platform-consolidation/05-MODULE-CATALOG.md"
dependencies:
  - "02-OBJECT-MODEL.md"
  - "06-RUNTIME-MODEL.md"
related_documents:
  - "17-MODULE-REGISTRY.md"
  - "15-CAPABILITY-REGISTRY.md"
acceptance_criteria:
  - "Every module has a unique ID"
  - "Every module has status (EXISTING | PLANNED | MOVED)"
  - "No two modules share the same import path"
  - "Total module count matches 17-MODULE-REGISTRY.md"
verification_checklist:
  - "[ ] EXISTING modules importable from current codebase"
  - "[ ] PLANNED modules have spec references"
  - "[ ] No orphaned modules (no owner)"
future_extensions:
  - "Auto-generate from directory scan"
  - "Module dependency graph visualisation"
---

# Platform Module Catalog

Status codes: `E` = EXISTING (already implemented), `P` = PLANNED, `M` = MOVED (exists elsewhere)

---

## Kernel Modules

| Module ID | Import Path | Status | Capabilities |
|-----------|-------------|--------|-------------|
| `kernel.bootstrap` | `platform.kernel.bootstrap` | P | boot_sequence, ordered_startup |
| `kernel.config` | `platform.kernel.config` | M | configuration, env_loading |
| `kernel.di` | `platform.kernel.di` | M | dependency_injection, service_locator |
| `kernel.events.bus` | `platform.kernel.events.bus` | M | event_dispatch, pub_sub |
| `kernel.events.dispatcher` | `platform.kernel.events.dispatcher` | M | async_dispatch, sync_dispatch |
| `kernel.lifecycle` | `platform.kernel.lifecycle` | M | start_stop, health_check |
| `kernel.registry.capability` | `platform.kernel.registry.capability` | P | capability_registration |
| `kernel.registry.service` | `platform.kernel.registry.service` | P | service_registration |
| `kernel.registry.module` | `platform.kernel.registry.module` | P | module_registration |
| `kernel.exceptions` | `platform.kernel.exceptions` | M | kernel_errors |

---

## SDK Modules

| Module ID | Import Path | Status | Capabilities |
|-----------|-------------|--------|-------------|
| `sdk.ai` | `platform.sdk.ai` | P | ai_facade, routing_facade |
| `sdk.knowledge` | `platform.sdk.knowledge` | P | knowledge_facade |
| `sdk.providers` | `platform.sdk.providers` | P | provider_facade |
| `sdk.workflows` | `platform.sdk.workflows` | P | workflow_facade |
| `sdk.resources` | `platform.sdk.resources` | P | resource_facade |
| `sdk.events` | `platform.sdk.events` | P | event_publishing_facade |
| `sdk.types` | `platform.sdk.types` | P | shared_types |
| `sdk.exceptions` | `platform.sdk.exceptions` | P | sdk_errors |

---

## AI Runtime Modules (`runtimes/ai_runtime/`)

| Module ID | Import Path | Status | Capabilities |
|-----------|-------------|--------|-------------|
| `ai.resource_intelligence.usage_collector` | `platform.runtimes.ai_runtime.resource_intelligence.usage_collector` | E | usage_recording, buffer_management |
| `ai.resource_intelligence.subscription_manager` | `...subscription_manager` | E | subscription_tracking, reset_periods |
| `ai.resource_intelligence.quota_manager` | `...quota_manager` | E | quota_enforcement, hard_soft_limits |
| `ai.resource_intelligence.api_budget_manager` | `...api_budget_manager` | E | budget_tracking, spend_enforcement |
| `ai.resource_intelligence.forecast_engine` | `...forecast_engine` | E | ema_forecast, linear_regression |
| `ai.resource_intelligence.workload_classifier` | `...workload_classifier` | E | workload_classification |
| `ai.resource_intelligence.provider_policy` | `...provider_policy` | E | policy_chains, provider_selection |
| `ai.resource_intelligence.routing_optimizer` | `...routing_optimizer` | E | pre_execution_routing |
| `ai.resource_intelligence.cost_optimizer` | `...cost_optimizer` | E | cost_reduction |
| `ai.resource_intelligence.account_pool` | `...account_pool` | E | account_selection, round_robin |
| `ai.resource_intelligence.session_analyzer` | `...session_analyzer` | E | session_profiling |
| `ai.resource_intelligence.token_estimator` | `...token_estimator` | E | token_estimation |
| `ai.resource_intelligence.resource_scheduler` | `...resource_scheduler` | E | job_scheduling |
| `ai.resource_intelligence.recommendation_engine` | `...recommendation_engine` | E | quota_recommendations |
| `ai.resource_intelligence.dashboard` | `...dashboard` | E | dashboard_snapshots |
| `ai.resource_intelligence.automation` | `...automation` | E | automation_rules |
| `ai.resource_intelligence.reports` | `...reports` | E | report_generation |
| `ai.resource_intelligence.workload_profiler` | `...workload_profiler` | E | 19_type_profiling |
| `ai.resource_intelligence.task_complexity` | `...task_complexity` | E | complexity_analysis |
| `ai.resource_intelligence.token_predictor` | `...token_predictor` | E | token_prediction, ema_learning |
| `ai.resource_intelligence.cost_predictor` | `...cost_predictor` | E | cost_prediction |
| `ai.resource_intelligence.quality_predictor` | `...quality_predictor` | E | quality_requirements |
| `ai.resource_intelligence.provider_selector` | `...provider_selector` | E | full_stack_selection |
| `ai.resource_intelligence.model_selector` | `...model_selector` | E | model_catalog_selection |
| `ai.resource_intelligence.account_selector` | `...account_selector` | E | account_selection |
| `ai.resource_intelligence.execution_planner` | `...execution_planner` | E | execution_plan_generation |
| `ai.resource_intelligence.execution_optimizer` | `...execution_optimizer` | E | plan_optimization |
| `ai.resource_intelligence.recommendation_center` | `...recommendation_center` | E | intelligence_recommendations |
| `ai.resource_intelligence.learning_engine` | `...learning_engine` | E | prediction_learning |

---

## Knowledge Runtime Modules (`runtimes/knowledge_runtime/`)

| Module ID | Import Path | Status | Capabilities |
|-----------|-------------|--------|-------------|
| `knowledge.graph` | `platform.runtimes.knowledge_runtime.graph` | M | knowledge_graph, traversal |
| `knowledge.compiler` | `platform.runtimes.knowledge_runtime.compiler` | M | metadata_compilation |
| `knowledge.search` | `platform.runtimes.knowledge_runtime.search` | M | semantic_search |
| `knowledge.objects` | `platform.runtimes.knowledge_runtime.objects` | M | knowledge_objects |
| `knowledge.index` | `platform.runtimes.knowledge_runtime.index` | M | knowledge_indexing |

---

## Provider Runtime Modules (`runtimes/provider_runtime/`)

| Module ID | Import Path | Status | Capabilities |
|-----------|-------------|--------|-------------|
| `provider.registry` | `platform.runtimes.provider_runtime.registry` | M | provider_registration |
| `provider.adapters` | `platform.runtimes.provider_runtime.adapters` | M | provider_adapters |
| `provider.health` | `platform.runtimes.provider_runtime.health` | M | provider_health_check |
| `provider.plugin` | `platform.runtimes.provider_runtime.plugin` | M | plugin_interface |

---

## Workflow Runtime Modules (`runtimes/workflow_runtime/`)

| Module ID | Import Path | Status | Capabilities |
|-----------|-------------|--------|-------------|
| `workflow.graph` | `platform.runtimes.workflow_runtime.graph` | P | task_dag |
| `workflow.executor` | `platform.runtimes.workflow_runtime.executor` | P | dag_execution |
| `workflow.scheduler` | `platform.runtimes.workflow_runtime.scheduler` | P | workflow_scheduling |

---

## Orchestration Runtime Modules (`runtimes/orchestration_runtime/`)

| Module ID | Import Path | Status | Capabilities |
|-----------|-------------|--------|-------------|
| `orchestration.coordinator` | `platform.runtimes.orchestration_runtime.coordinator` | P | multi_agent_coordination |
| `orchestration.agents` | `platform.runtimes.orchestration_runtime.agents` | P | agent_lifecycle |
| `orchestration.sessions` | `platform.runtimes.orchestration_runtime.sessions` | P | session_management |

---

## Event Runtime Modules (`runtimes/event_runtime/`)

| Module ID | Import Path | Status | Capabilities |
|-----------|-------------|--------|-------------|
| `event.bus` | `platform.runtimes.event_runtime.bus` | M | event_bus |
| `event.pubsub` | `platform.runtimes.event_runtime.pubsub` | P | pub_sub |
| `event.streaming` | `platform.runtimes.event_runtime.streaming` | P | streaming_events |

---

## Resource Runtime Modules (`runtimes/resource_runtime/`)

| Module ID | Import Path | Status | Capabilities |
|-----------|-------------|--------|-------------|
| `resource.quota` | `platform.runtimes.resource_runtime.quota` | M | quota_management |
| `resource.budget` | `platform.runtimes.resource_runtime.budget` | M | budget_management |
| `resource.scheduler` | `platform.runtimes.resource_runtime.scheduler` | M | resource_scheduling |

---

## Services Modules

| Module ID | Import Path | Status | Capabilities |
|-----------|-------------|--------|-------------|
| `service.auth` | `platform.services.auth` | M | authentication, authorization |
| `service.audit` | `platform.services.audit` | P | audit_logging |
| `service.metrics` | `platform.services.metrics` | M | prometheus_metrics |
| `service.tracing` | `platform.services.tracing` | P | distributed_tracing |
| `service.cache` | `platform.services.cache` | M | caching |
| `service.notifications` | `platform.services.notifications` | P | alerts |

---

## Common Modules

| Module ID | Import Path | Status | Capabilities |
|-----------|-------------|--------|-------------|
| `common.types` | `platform.common.types` | M | base_types |
| `common.exceptions` | `platform.common.exceptions` | M | base_exceptions |
| `common.utils` | `platform.common.utils` | M | utilities |
| `common.constants` | `platform.common.constants` | M | system_constants |

---

## Summary

| Runtime | Existing | Planned | Moved | Total |
|---------|----------|---------|-------|-------|
| kernel | 0 | 4 | 6 | 10 |
| sdk | 0 | 8 | 0 | 8 |
| ai_runtime | 29 | 0 | 0 | 29 |
| knowledge_runtime | 0 | 0 | 5 | 5 |
| provider_runtime | 0 | 0 | 4 | 4 |
| workflow_runtime | 0 | 3 | 0 | 3 |
| orchestration_runtime | 0 | 3 | 0 | 3 |
| event_runtime | 0 | 2 | 1 | 3 |
| resource_runtime | 0 | 0 | 3 | 3 |
| services | 0 | 3 | 3 | 6 |
| common | 0 | 0 | 4 | 4 |
| **Total** | **29** | **23** | **26** | **78** |
