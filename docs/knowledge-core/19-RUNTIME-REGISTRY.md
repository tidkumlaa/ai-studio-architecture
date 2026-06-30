# KNW-KC-ARCH-019 — Runtime Registry

**Phase:** 3.0C  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

The Runtime Registry catalogs all platform runtimes — their boot order, module inventory, capability contracts, and health status. Every runtime that participates in the platform must have a registered entry before boot.

---

## Registered Runtimes

| Runtime ID | Name | Boot Order | Status | Phase |
|-----------|------|-----------|--------|-------|
| `runtime:kernel:v1` | Platform Kernel | 1 | CANONICAL | 2.0D |
| `runtime:sdk:v2` | Platform SDK | 2 | CANONICAL | 2.0D |
| `runtime:ai:v2` | AI Runtime | 3 | CANONICAL | 2.0D |
| `runtime:knowledge:v1` | Knowledge Runtime | 4 | BETA | 3.0B |
| `runtime:financial:v1` | Financial Runtime | 5 | PLANNED | Future |
| `runtime:forex:v1` | Forex Runtime | 6 | PLANNED | Future |
| `runtime:research:v1` | Research Runtime | 7 | PLANNED | Future |
| `runtime:agent:v1` | Agent Runtime | 8 | PLANNED | Future |
| `runtime:content:v1` | Content Runtime | 9 | PLANNED | Future |

---

## Runtime Entry Format

```yaml
# knowledge/registry/runtimes/runtime.ai.v2.yaml
identity:
  knowledge_id: "KNW-AI-RUN-001"
  canonical_name: "ai.runtime.v2"
  knowledge_uri: "knw://ai/runtime/v2"
  namespace: "ai"
  version: "2.0.0"
  owner: "team:ai-runtime"

runtime_spec:
  runtime_id: "runtime:ai:v2"
  boot_order: 3
  platform_id: "KNW-PLAT-ARCH-001"
  python_path: "ai_runtime"

  # Dependencies
  depends_on_runtimes:
    - runtime_id: "runtime:kernel:v1"
      required: true
    - runtime_id: "runtime:sdk:v2"
      required: true

  # Modules (29 modules as per Phase 2.1D.0 32-RUNTIME-INTEGRATION)
  module_count: 29
  module_ids:
    - "KNW-AI-MOD-quota_manager"
    - "KNW-AI-MOD-provider_manager"
    # ... all 29 modules

  # Capabilities
  capability_ids:
    - "capability:ai:model_selection:v2"
    - "capability:ai:workload_routing:v2"
    - "capability:ai:cost_prediction:v1"

  # Boot configuration
  boot_timeout_ms: 1450
  shutdown_timeout_ms: 30000       # 30s drain timeout

  # Health
  health_check_path: "ai_runtime.health.check"
  health_check_interval_s: 30

lifecycle:
  status: CANONICAL
```

---

## Runtime Boot Dependency Graph

```
runtime:kernel:v1 (boot_order=1)
    └── runtime:sdk:v2 (boot_order=2)
            └── runtime:ai:v2 (boot_order=3)
                    └── runtime:knowledge:v1 (boot_order=4)
                            └── runtime:financial:v1 (boot_order=5)
                                    └── runtime:forex:v1 (boot_order=6)
                                            └── ...
```

Boot order is strictly sequential within a priority tier. Parallel boot is allowed only between runtimes with no dependency relationship.

---

## Runtime Registry Rules

### RR-001 — Sequential Boot Ordering
`boot_order` must be globally unique. No two runtimes may have the same `boot_order`.

### RR-002 — Dependency Validation
Every `depends_on_runtimes` entry must reference a runtime with a lower `boot_order`.

### RR-003 — Module Inventory
All module_ids must be registered in the Object Registry before the runtime may boot.

### RR-004 — Timeout Contracts
`boot_timeout_ms` is a hard contract — a runtime that does not boot within this window is marked FAILED and the platform boot is aborted.

### RR-005 — Version Lock
Once a runtime version is CANONICAL, its `boot_order` and `module_ids` are immutable. Changes require a MAJOR version bump.

---

## Runtime Health Status

```yaml
runtime_health:
  runtime_id: "runtime:ai:v2"
  status: HEALTHY|DEGRADED|FAILED|STARTING|STOPPING
  last_health_check: datetime
  boot_duration_ms: 1320
  module_health:
    healthy: 28
    degraded: 1
    failed: 0
  uptime_seconds: 86400
  last_restart_at: datetime | null
  restart_count: 0
```

---

## Runtime Registry Protocol

```python
class RuntimeRegistry(KnowledgeRegistryProtocol):
    def get_boot_order(self) -> list[RuntimeEntry]: ...      # sorted by boot_order
    def get_dependencies(self, runtime_id: str) -> list[str]: ...
    def get_modules(self, runtime_id: str) -> list[str]: ...
    def get_capabilities(self, runtime_id: str) -> list[str]: ...
    def record_boot(self, runtime_id: str, duration_ms: int) -> None: ...
    def record_health(self, runtime_id: str, status: RuntimeHealth) -> None: ...
    def get_health(self, runtime_id: str) -> RuntimeHealth: ...
    def get_all_health(self) -> dict[str, RuntimeHealth]: ...
    def validate_boot_graph(self) -> list[ValidationError]: ...
```

---

## Cross-References

- Registry base contract → `14-REGISTRY-ARCHITECTURE`
- Runtime boot sequence → Phase 2.1D.0 `32-RUNTIME-INTEGRATION`
- Module catalog → Phase 2.1D.0 `17-MODULE-REGISTRY`
- Runtime state machine → Phase 2.1D.0 `19-STATE-MACHINES`
