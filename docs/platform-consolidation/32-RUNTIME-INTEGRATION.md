---
knowledge_id: KNW-PLAT-ARCH-032
title: "Platform Runtime Integration"
status: AUTHORITATIVE
phase: "2.1D.0"
version: "1.0.0"
created: "2026-06-29"
purpose: "Specify how all 7 runtimes integrate at boot, communicate, and share the kernel"
canonical_source: "architecture/docs/platform-consolidation/32-RUNTIME-INTEGRATION.md"
dependencies:
  - "06-RUNTIME-MODEL.md"
  - "07-KERNEL-MODEL.md"
  - "18-EVENT-MODEL.md"
  - "10-DEPENDENCY-RULES.md"
related_documents:
  - "09-SERVICE-MODEL.md"
  - "19-STATE-MACHINES.md"
acceptance_criteria:
  - "Full boot sequence is deterministic"
  - "No runtime holds a reference to another runtime's objects"
  - "Runtime shutdown is orderly (reverse boot order)"
verification_checklist:
  - "[ ] Boot sequence documented with timing"
  - "[ ] Shutdown sequence is reverse of boot"
  - "[ ] Cross-runtime communication uses events only"
future_extensions:
  - "Hot-reload of individual runtimes without full platform restart"
  - "Runtime isolation via OS processes"
---

# Platform Runtime Integration

## Boot Sequence

Runtimes boot in strict order. Each runtime must reach READY state before the next starts.

```
T=0    PlatformKernel boots
       ├── DIContainer initializes
       ├── PlatformConfig loads
       ├── EventBus starts
       ├── LifecycleManager starts
       └── CapabilityRegistry initializes

T+0ms  event_runtime boots (boot_order=1)
       ├── Subscribes to kernel events
       ├── Registers event.publish, event.subscribe capabilities
       └── → READY

T+50ms resource_runtime boots (boot_order=2)
       ├── Requires: capability:kernel:events:v1 (from kernel)
       ├── Initializes QuotaEngine, BudgetManager, ResourceScheduler
       ├── Registers resource.* capabilities
       └── → READY

T+150ms provider_runtime boots (boot_order=3)
       ├── Requires: capability:kernel:events:v1
       ├── Loads provider configs from PlatformConfig
       ├── Registers provider.* capabilities
       └── → READY

T+350ms knowledge_runtime boots (boot_order=4)
       ├── Requires: capability:kernel:events:v1
       ├── Initializes KnowledgeGraph, KnowledgeSearch, KnowledgeIndex
       ├── Compiles knowledge documents (async, non-blocking)
       └── → READY

T+850ms ai_runtime boots (boot_order=5)
       ├── Requires: capability:resource:quota.check:v1
       ├── Requires: capability:provider:register:v1
       ├── Requires: capability:knowledge:search:v1 (optional)
       ├── Initializes all 29 resource intelligence modules
       ├── Registers ai.* capabilities
       └── → READY

T+1350ms workflow_runtime boots (boot_order=6) [PLANNED]
       ├── Requires: capability:ai:execution.plan:v1
       └── → READY

T+1450ms orchestration_runtime boots (boot_order=7) [PLANNED]
       ├── Requires: capability:workflow:execute:v1
       └── → READY

T+1450ms Platform READY
```

---

## Boot Protocol

Each runtime implements `PlatformRuntime` Protocol:

```python
class PlatformRuntime(Protocol):
    runtime_id: str
    boot_order: int

    async def initialize(self, kernel: PlatformKernel) -> None:
        """Called by LifecycleManager in boot_order sequence."""
        ...

    async def start(self) -> None:
        """Start accepting work. Called after initialize()."""
        ...

    async def stop(self) -> None:
        """Graceful shutdown. Called in reverse boot_order."""
        ...

    def health(self) -> RuntimeHealth:
        """Current health status."""
        ...
```

---

## Capability Registration

At boot, each runtime registers its capabilities with the kernel:

```python
# Inside ai_runtime.initialize()
kernel.capability_registry.register(
    capability_id="capability:ai:quota.check:v1",
    provider=self,
    interface=QuotaEngine,
)
kernel.capability_registry.register(
    capability_id="capability:ai:routing.optimize:v1",
    provider=self,
    interface=RoutingOptimizer,
)
# ... all 32 ai runtime capabilities
```

Runtimes that depend on capabilities check them at init:

```python
# Inside ai_runtime.initialize()
quota_cap = kernel.capability_registry.get("capability:resource:quota.check:v1")
if quota_cap is None:
    raise RuntimeBootError("ai_runtime requires resource:quota.check — boot resource_runtime first")
```

---

## Cross-Runtime Communication

**Rule:** No runtime calls another runtime's functions directly (DR-002).

All cross-runtime coordination is event-based:

```python
# ai_runtime → publishes → EventBus
await event_bus.publish(PlatformEvent(
    topic="platform.ai.quota.consumed",
    source_runtime="runtime:ai:v2",
    payload={"account_id": "acc-001", "tokens": 1000},
))

# resource_runtime → subscribes → EventBus
event_bus.subscribe(
    topic_pattern="platform.ai.quota.*",
    handler=resource_runtime.on_ai_quota_event,
)
```

---

## Shutdown Sequence

Runtimes shut down in **reverse boot order** (7→6→5→4→3→2→1→kernel):

```
orchestration_runtime.stop()
workflow_runtime.stop()
ai_runtime.stop()        ← drains in-flight execution plans
knowledge_runtime.stop() ← finalizes index writes
provider_runtime.stop()  ← closes provider connections
resource_runtime.stop()  ← flushes usage records
event_runtime.stop()     ← drains event queue
PlatformKernel.stop()    ← DI container, config, EventBus teardown
```

**Drain timeout:** Each runtime has 30 seconds to stop cleanly.
If it doesn't stop within 30 seconds, the shutdown continues (forced stop).

---

## Runtime Dependency Matrix

Cross-reference with 10-DEPENDENCY-RULES.md:

| Consumer | Dependency | Mechanism |
|----------|-----------|-----------|
| resource_runtime | kernel events | EventBus subscription |
| provider_runtime | kernel events | EventBus subscription |
| knowledge_runtime | kernel events | EventBus subscription |
| ai_runtime | resource quota | Capability lookup at boot |
| ai_runtime | provider registry | Capability lookup at boot |
| ai_runtime | kernel events | EventBus subscription |
| workflow_runtime | ai execution plan | Capability lookup at boot |
| orchestration_runtime | workflow execute | Capability lookup at boot |

---

## Runtime Isolation Rules

| Rule | Description |
|------|-------------|
| RI-001 | No runtime holds a direct Python reference to another runtime object |
| RI-002 | No runtime imports from another runtime's Python package |
| RI-003 | All shared data passes through Pydantic models (not mutable objects) |
| RI-004 | EventBus payloads are dicts (serializable), not runtime objects |
| RI-005 | Capability interfaces are Protocols — no concrete class references cross runtime boundary |

---

## AI Runtime Internal Boot Sequence

Within `ai_runtime`, modules boot in this order:

```
1. usage_collector
2. subscription_manager
3. quota_manager          (requires usage_collector)
4. api_budget_manager
5. workload_classifier
6. token_estimator
7. account_pool           (requires subscription_manager)
8. provider_policy
9. routing_optimizer      (requires quota_manager, account_pool, provider_policy)
10. cost_optimizer        (requires routing_optimizer)
11. forecast_engine       (requires usage_collector)
12. resource_scheduler    (requires quota_manager)
13. session_analyzer
14. recommendation_engine (requires all above)
15. dashboard             (requires all above)
16. automation            (requires recommendation_engine)
17. reports               (requires usage_collector, forecast_engine)
--- Phase 2.1B modules ---
18. workload_profiler
19. task_complexity
20. token_predictor       (requires workload_profiler)
21. cost_predictor
22. quality_predictor
23. model_selector
24. account_selector      (requires account_pool)
25. provider_selector     (requires quota_manager, account_pool)
26. execution_planner     (requires all predictors + selectors)
27. execution_optimizer   (requires execution_planner)
28. recommendation_center (requires all above)
29. learning_engine
```
