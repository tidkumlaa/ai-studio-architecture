---
knowledge_id: KNW-PLAT-ARCH-015
title: "Platform Capability Registry"
status: AUTHORITATIVE
phase: "2.1D.0"
version: "1.0.0"
created: "2026-06-29"
purpose: "Complete registry of all platform capabilities with IDs, providers, and consumers"
canonical_source: "architecture/docs/platform-consolidation/15-CAPABILITY-REGISTRY.md"
dependencies:
  - "02-OBJECT-MODEL.md"
  - "06-RUNTIME-MODEL.md"
related_documents:
  - "16-SERVICE-REGISTRY.md"
  - "17-MODULE-REGISTRY.md"
  - "07-KERNEL-MODEL.md"
acceptance_criteria:
  - "Every capability has a unique ID"
  - "Every capability has a provider and stability level"
  - "Total count matches 14-PLATFORM-MANIFEST.md"
verification_checklist:
  - "[ ] All kernel capabilities listed"
  - "[ ] All runtime capabilities listed"
  - "[ ] IDs follow naming convention"
future_extensions:
  - "Capability versioning and deprecation notices"
  - "Capability dependency graph"
---

# Platform Capability Registry

## ID Convention
```
capability:{domain}:{name}:v{major}
```
Examples: `capability:kernel:events:v1`, `capability:ai:routing:v1`

---

## Kernel Capabilities

| Capability ID | Interface | Stability |
|---------------|-----------|-----------|
| `capability:kernel:di:v1` | `DIContainer` | STABLE |
| `capability:kernel:events:v1` | `EventBus` | STABLE |
| `capability:kernel:config:v1` | `PlatformConfig` | STABLE |
| `capability:kernel:lifecycle:v1` | `LifecycleManager` | STABLE |
| `capability:kernel:registry.capability:v1` | `CapabilityRegistry` | STABLE |
| `capability:kernel:registry.service:v1` | `ServiceRegistry` | STABLE |
| `capability:kernel:registry.module:v1` | `ModuleRegistry` | STABLE |

---

## Service Capabilities

| Capability ID | Provider | Stability |
|---------------|----------|-----------|
| `capability:service:auth:v1` | `service.auth` | STABLE |
| `capability:service:audit:v1` | `service.audit` | STABLE |
| `capability:service:metrics:v1` | `service.metrics` | STABLE |
| `capability:service:tracing:v1` | `service.tracing` | BETA |
| `capability:service:cache:v1` | `service.cache` | STABLE |
| `capability:service:notifications:v1` | `service.notifications` | BETA |

---

## Event Runtime Capabilities

| Capability ID | Interface | Stability |
|---------------|-----------|-----------|
| `capability:event:publish:v1` | `EventBus.publish` | STABLE |
| `capability:event:subscribe:v1` | `EventBus.subscribe` | STABLE |
| `capability:event:stream:v1` | `EventBus.stream` | BETA |

---

## Resource Runtime Capabilities

| Capability ID | Interface | Stability |
|---------------|-----------|-----------|
| `capability:resource:quota.check:v1` | `QuotaEngine.check` | STABLE |
| `capability:resource:quota.consume:v1` | `QuotaEngine.consume` | STABLE |
| `capability:resource:budget.check:v1` | `BudgetManager.check` | STABLE |
| `capability:resource:schedule:v1` | `ResourceScheduler.submit` | STABLE |

---

## Provider Runtime Capabilities

| Capability ID | Interface | Stability |
|---------------|-----------|-----------|
| `capability:provider:register:v1` | `ProviderRegistry.register` | STABLE |
| `capability:provider:get:v1` | `ProviderRegistry.get` | STABLE |
| `capability:provider:health:v1` | `ProviderRegistry.health` | STABLE |

---

## Knowledge Runtime Capabilities

| Capability ID | Interface | Stability |
|---------------|-----------|-----------|
| `capability:knowledge:compile:v1` | `KnowledgeCompiler.compile` | STABLE |
| `capability:knowledge:search:v1` | `KnowledgeSearch.search` | STABLE |
| `capability:knowledge:link:v1` | `KnowledgeGraph.link` | STABLE |
| `capability:knowledge:get:v1` | `KnowledgeGraph.get` | STABLE |
| `capability:knowledge:index:v1` | `KnowledgeIndex.update` | STABLE |

---

## AI Runtime Capabilities (Phase 2.1A)

| Capability ID | Module | Stability |
|---------------|--------|-----------|
| `capability:ai:usage.record:v1` | `usage_collector` | STABLE |
| `capability:ai:usage.query:v1` | `usage_collector` | STABLE |
| `capability:ai:subscription.register:v1` | `subscription_manager` | STABLE |
| `capability:ai:subscription.usage:v1` | `subscription_manager` | STABLE |
| `capability:ai:quota.check:v1` | `quota_manager` | STABLE |
| `capability:ai:quota.consume:v1` | `quota_manager` | STABLE |
| `capability:ai:budget.check:v1` | `api_budget_manager` | STABLE |
| `capability:ai:forecast:v1` | `forecast_engine` | STABLE |
| `capability:ai:classify:v1` | `workload_classifier` | STABLE |
| `capability:ai:policy.resolve:v1` | `provider_policy` | STABLE |
| `capability:ai:routing.optimize:v1` | `routing_optimizer` | STABLE |
| `capability:ai:cost.optimize:v1` | `cost_optimizer` | STABLE |
| `capability:ai:account.select:v1` | `account_pool` | STABLE |
| `capability:ai:session.analyze:v1` | `session_analyzer` | STABLE |
| `capability:ai:token.estimate:v1` | `token_estimator` | STABLE |
| `capability:ai:schedule.job:v1` | `resource_scheduler` | STABLE |
| `capability:ai:recommend:v1` | `recommendation_engine` | STABLE |
| `capability:ai:dashboard.snapshot:v1` | `dashboard` | STABLE |
| `capability:ai:automation.evaluate:v1` | `automation` | STABLE |
| `capability:ai:report.generate:v1` | `reports` | STABLE |

---

## AI Runtime Capabilities (Phase 2.1B)

| Capability ID | Module | Stability |
|---------------|--------|-----------|
| `capability:ai:workload.profile:v1` | `workload_profiler` | STABLE |
| `capability:ai:complexity.analyze:v1` | `task_complexity` | STABLE |
| `capability:ai:token.predict:v1` | `token_predictor` | STABLE |
| `capability:ai:cost.predict:v1` | `cost_predictor` | STABLE |
| `capability:ai:quality.predict:v1` | `quality_predictor` | STABLE |
| `capability:ai:provider.select:v1` | `provider_selector` | STABLE |
| `capability:ai:model.select:v1` | `model_selector` | STABLE |
| `capability:ai:account.selector:v1` | `account_selector` | STABLE |
| `capability:ai:execution.plan:v1` | `execution_planner` | STABLE |
| `capability:ai:execution.optimize:v1` | `execution_optimizer` | STABLE |
| `capability:ai:intelligence.recommend:v1` | `recommendation_center` | STABLE |
| `capability:ai:learning.record:v1` | `learning_engine` | STABLE |

---

## Workflow Runtime Capabilities (Planned)

| Capability ID | Status |
|---------------|--------|
| `capability:workflow:define:v1` | PLANNED |
| `capability:workflow:execute:v1` | PLANNED |
| `capability:workflow:status:v1` | PLANNED |

---

## Orchestration Runtime Capabilities (Planned)

| Capability ID | Status |
|---------------|--------|
| `capability:orchestration:spawn:v1` | PLANNED |
| `capability:orchestration:coordinate:v1` | PLANNED |
| `capability:orchestration:terminate:v1` | PLANNED |

---

## Registry Summary

| Domain | Stable | Beta | Planned | Total |
|--------|--------|------|---------|-------|
| kernel | 7 | 0 | 0 | 7 |
| service | 4 | 2 | 0 | 6 |
| event | 2 | 1 | 0 | 3 |
| resource | 4 | 0 | 0 | 4 |
| provider | 3 | 0 | 0 | 3 |
| knowledge | 5 | 0 | 0 | 5 |
| ai (2.1A) | 20 | 0 | 0 | 20 |
| ai (2.1B) | 12 | 0 | 0 | 12 |
| workflow | 0 | 0 | 3 | 3 |
| orchestration | 0 | 0 | 3 | 3 |
| **Total** | **57** | **3** | **6** | **66** |
