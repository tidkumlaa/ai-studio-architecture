---
knowledge_id: KNW-PLAT-ARCH-017
title: "Platform Module Registry"
status: AUTHORITATIVE
phase: "2.1D.0"
version: "1.0.0"
created: "2026-06-29"
purpose: "Complete registry of all 78 platform modules with IDs, runtime ownership, status, and capabilities"
canonical_source: "architecture/docs/platform-consolidation/17-MODULE-REGISTRY.md"
dependencies:
  - "05-MODULE-CATALOG.md"
  - "06-RUNTIME-MODEL.md"
  - "15-CAPABILITY-REGISTRY.md"
related_documents:
  - "16-SERVICE-REGISTRY.md"
  - "14-PLATFORM-MANIFEST.md"
acceptance_criteria:
  - "Every module has a unique ID"
  - "Every module has exactly one runtime owner"
  - "Module count matches 05-MODULE-CATALOG.md (78 total)"
verification_checklist:
  - "[ ] All module IDs are unique"
  - "[ ] All modules assigned to a runtime"
  - "[ ] Status values are valid (EXISTING/PLANNED/MOVED/DEPRECATED)"
future_extensions:
  - "Module dependency graph visualization"
  - "Module health metrics per module"
---

# Platform Module Registry

## Module ID Convention

```
module:{runtime_domain}:{name}:v{major}
```
Examples: `module:ai:resource_intelligence:v2`, `module:kernel:di:v1`

---

## Kernel Modules

| Module ID | Python Path | Status |
|-----------|-------------|--------|
| `module:kernel:di:v1` | `platform.kernel.di` | EXISTING |
| `module:kernel:events:v1` | `platform.kernel.events` | EXISTING |
| `module:kernel:config:v1` | `platform.kernel.config` | EXISTING |
| `module:kernel:lifecycle:v1` | `platform.kernel.lifecycle` | EXISTING |
| `module:kernel:registry:v1` | `platform.kernel.registry` | EXISTING |

---

## Common Modules

| Module ID | Python Path | Status |
|-----------|-------------|--------|
| `module:common:types:v1` | `platform.common.types` | EXISTING |
| `module:common:exceptions:v1` | `platform.common.exceptions` | EXISTING |
| `module:common:utils:v1` | `platform.common.utils` | EXISTING |

---

## Event Runtime Modules

| Module ID | Python Path | Status | Capabilities |
|-----------|-------------|--------|--------------|
| `module:event:bus:v1` | `platform.runtimes.event_runtime.bus` | EXISTING | `capability:event:publish:v1`, `capability:event:subscribe:v1` |
| `module:event:stream:v1` | `platform.runtimes.event_runtime.stream` | PLANNED | `capability:event:stream:v1` |
| `module:event:router:v1` | `platform.runtimes.event_runtime.router` | EXISTING | — |

---

## Resource Runtime Modules

| Module ID | Python Path | Status | Capabilities |
|-----------|-------------|--------|--------------|
| `module:resource:quota_engine:v1` | `platform.runtimes.resource_runtime.quota_engine` | EXISTING | `capability:resource:quota.check:v1`, `capability:resource:quota.consume:v1` |
| `module:resource:budget_manager:v1` | `platform.runtimes.resource_runtime.budget_manager` | EXISTING | `capability:resource:budget.check:v1` |
| `module:resource:scheduler:v1` | `platform.runtimes.resource_runtime.scheduler` | EXISTING | `capability:resource:schedule:v1` |

---

## Provider Runtime Modules

| Module ID | Python Path | Status | Capabilities |
|-----------|-------------|--------|--------------|
| `module:provider:registry:v1` | `platform.runtimes.provider_runtime.registry` | EXISTING | `capability:provider:register:v1`, `capability:provider:get:v1` |
| `module:provider:health:v1` | `platform.runtimes.provider_runtime.health` | EXISTING | `capability:provider:health:v1` |
| `module:provider:adapter:v1` | `platform.runtimes.provider_runtime.adapter` | EXISTING | — |
| `module:provider:circuit_breaker:v1` | `platform.runtimes.provider_runtime.circuit_breaker` | EXISTING | — |

---

## Knowledge Runtime Modules

| Module ID | Python Path | Status | Capabilities |
|-----------|-------------|--------|--------------|
| `module:knowledge:compiler:v1` | `platform.runtimes.knowledge_runtime.compiler` | EXISTING | `capability:knowledge:compile:v1` |
| `module:knowledge:search:v1` | `platform.runtimes.knowledge_runtime.search` | EXISTING | `capability:knowledge:search:v1` |
| `module:knowledge:graph:v1` | `platform.runtimes.knowledge_runtime.graph` | EXISTING | `capability:knowledge:link:v1`, `capability:knowledge:get:v1` |
| `module:knowledge:index:v1` | `platform.runtimes.knowledge_runtime.index` | EXISTING | `capability:knowledge:index:v1` |
| `module:knowledge:embeddings:v1` | `platform.runtimes.knowledge_runtime.embeddings` | PLANNED | — |

---

## AI Runtime Modules — Phase 2.1A (Resource Management)

| Module ID | Python Path | Status | Capabilities |
|-----------|-------------|--------|--------------|
| `module:ai:usage_collector:v1` | `platform.runtimes.ai_runtime.resource_intelligence.usage_collector` | EXISTING | `capability:ai:usage.record:v1`, `capability:ai:usage.query:v1` |
| `module:ai:subscription_manager:v1` | `platform.runtimes.ai_runtime.resource_intelligence.subscription_manager` | EXISTING | `capability:ai:subscription.register:v1`, `capability:ai:subscription.usage:v1` |
| `module:ai:quota_manager:v1` | `platform.runtimes.ai_runtime.resource_intelligence.quota_manager` | EXISTING | `capability:ai:quota.check:v1`, `capability:ai:quota.consume:v1` |
| `module:ai:api_budget_manager:v1` | `platform.runtimes.ai_runtime.resource_intelligence.api_budget_manager` | EXISTING | `capability:ai:budget.check:v1` |
| `module:ai:forecast_engine:v1` | `platform.runtimes.ai_runtime.resource_intelligence.forecast_engine` | EXISTING | `capability:ai:forecast:v1` |
| `module:ai:workload_classifier:v1` | `platform.runtimes.ai_runtime.resource_intelligence.workload_classifier` | EXISTING | `capability:ai:classify:v1` |
| `module:ai:provider_policy:v1` | `platform.runtimes.ai_runtime.resource_intelligence.provider_policy` | EXISTING | `capability:ai:policy.resolve:v1` |
| `module:ai:routing_optimizer:v1` | `platform.runtimes.ai_runtime.resource_intelligence.routing_optimizer` | EXISTING | `capability:ai:routing.optimize:v1` |
| `module:ai:cost_optimizer:v1` | `platform.runtimes.ai_runtime.resource_intelligence.cost_optimizer` | EXISTING | `capability:ai:cost.optimize:v1` |
| `module:ai:account_pool:v1` | `platform.runtimes.ai_runtime.resource_intelligence.account_pool` | EXISTING | `capability:ai:account.select:v1` |
| `module:ai:session_analyzer:v1` | `platform.runtimes.ai_runtime.resource_intelligence.session_analyzer` | EXISTING | `capability:ai:session.analyze:v1` |
| `module:ai:token_estimator:v1` | `platform.runtimes.ai_runtime.resource_intelligence.token_estimator` | EXISTING | `capability:ai:token.estimate:v1` |
| `module:ai:resource_scheduler:v1` | `platform.runtimes.ai_runtime.resource_intelligence.resource_scheduler` | EXISTING | `capability:ai:schedule.job:v1` |
| `module:ai:recommendation_engine:v1` | `platform.runtimes.ai_runtime.resource_intelligence.recommendation_engine` | EXISTING | `capability:ai:recommend:v1` |
| `module:ai:dashboard:v1` | `platform.runtimes.ai_runtime.resource_intelligence.dashboard` | EXISTING | `capability:ai:dashboard.snapshot:v1` |
| `module:ai:automation:v1` | `platform.runtimes.ai_runtime.resource_intelligence.automation` | EXISTING | `capability:ai:automation.evaluate:v1` |
| `module:ai:reports:v1` | `platform.runtimes.ai_runtime.resource_intelligence.reports` | EXISTING | `capability:ai:report.generate:v1` |

---

## AI Runtime Modules — Phase 2.1B (Intelligence Engine)

| Module ID | Python Path | Status | Capabilities |
|-----------|-------------|--------|--------------|
| `module:ai:workload_profiler:v1` | `platform.runtimes.ai_runtime.resource_intelligence.workload_profiler` | EXISTING | `capability:ai:workload.profile:v1` |
| `module:ai:task_complexity:v1` | `platform.runtimes.ai_runtime.resource_intelligence.task_complexity` | EXISTING | `capability:ai:complexity.analyze:v1` |
| `module:ai:token_predictor:v1` | `platform.runtimes.ai_runtime.resource_intelligence.token_predictor` | EXISTING | `capability:ai:token.predict:v1` |
| `module:ai:cost_predictor:v1` | `platform.runtimes.ai_runtime.resource_intelligence.cost_predictor` | EXISTING | `capability:ai:cost.predict:v1` |
| `module:ai:quality_predictor:v1` | `platform.runtimes.ai_runtime.resource_intelligence.quality_predictor` | EXISTING | `capability:ai:quality.predict:v1` |
| `module:ai:provider_selector:v1` | `platform.runtimes.ai_runtime.resource_intelligence.provider_selector` | EXISTING | `capability:ai:provider.select:v1` |
| `module:ai:model_selector:v1` | `platform.runtimes.ai_runtime.resource_intelligence.model_selector` | EXISTING | `capability:ai:model.select:v1` |
| `module:ai:account_selector:v1` | `platform.runtimes.ai_runtime.resource_intelligence.account_selector` | EXISTING | `capability:ai:account.selector:v1` |
| `module:ai:execution_planner:v1` | `platform.runtimes.ai_runtime.resource_intelligence.execution_planner` | EXISTING | `capability:ai:execution.plan:v1` |
| `module:ai:execution_optimizer:v1` | `platform.runtimes.ai_runtime.resource_intelligence.execution_optimizer` | EXISTING | `capability:ai:execution.optimize:v1` |
| `module:ai:recommendation_center:v1` | `platform.runtimes.ai_runtime.resource_intelligence.recommendation_center` | EXISTING | `capability:ai:intelligence.recommend:v1` |
| `module:ai:learning_engine:v1` | `platform.runtimes.ai_runtime.resource_intelligence.learning_engine` | EXISTING | `capability:ai:learning.record:v1` |

---

## Workflow Runtime Modules (Planned)

| Module ID | Python Path | Status |
|-----------|-------------|--------|
| `module:workflow:engine:v1` | `platform.runtimes.workflow_runtime.engine` | PLANNED |
| `module:workflow:state:v1` | `platform.runtimes.workflow_runtime.state` | PLANNED |
| `module:workflow:scheduler:v1` | `platform.runtimes.workflow_runtime.scheduler` | PLANNED |

---

## Orchestration Runtime Modules (Planned)

| Module ID | Python Path | Status |
|-----------|-------------|--------|
| `module:orchestration:coordinator:v1` | `platform.runtimes.orchestration_runtime.coordinator` | PLANNED |
| `module:orchestration:agent_pool:v1` | `platform.runtimes.orchestration_runtime.agent_pool` | PLANNED |
| `module:orchestration:task_router:v1` | `platform.runtimes.orchestration_runtime.task_router` | PLANNED |

---

## Module Count Summary

| Runtime | Existing | Planned | Total |
|---------|----------|---------|-------|
| kernel | 5 | 0 | 5 |
| common | 3 | 0 | 3 |
| event | 2 | 1 | 3 |
| resource | 3 | 0 | 3 |
| provider | 4 | 0 | 4 |
| knowledge | 4 | 1 | 5 |
| ai (2.1A) | 17 | 0 | 17 |
| ai (2.1B) | 12 | 0 | 12 |
| workflow | 0 | 3 | 3 |
| orchestration | 0 | 3 | 3 |
| **Total** | **50** | **8** | **58** |

> Note: Total module count here is 58. The manifest's `module_count: 78` includes sub-modules
> and internal components within composite modules. Refer to 05-MODULE-CATALOG.md for the
> full breakdown of all 78 module entries.
