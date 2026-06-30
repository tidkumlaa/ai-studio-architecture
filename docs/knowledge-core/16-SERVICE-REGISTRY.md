# KNW-KC-ARCH-016 — Service Registry

**Phase:** 3.0C  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

The Service Registry is the authoritative catalog of all platform services — cross-cutting concerns exposed through Abstract Base Class interfaces. Every service has exactly one entry. All consumers discover services through this registry.

---

## Registered Services (6 Core Platform Services)

| Service ID | Name | Interface Path | Stability |
|-----------|------|----------------|-----------|
| `service.auth` | Authentication & Authorization | `platform_kernel.contracts.auth.AuthService` | STABLE |
| `service.audit` | Audit Trail | `platform_kernel.contracts.audit.AuditService` | STABLE |
| `service.metrics` | Platform Metrics | `platform_kernel.contracts.metrics.MetricsService` | STABLE |
| `service.tracing` | Distributed Tracing | `platform_kernel.contracts.tracing.TracingService` | STABLE |
| `service.cache` | Caching Layer | `platform_kernel.contracts.cache.CacheService` | STABLE |
| `service.notifications` | Notifications | `platform_kernel.contracts.notifications.NotificationService` | BETA |

---

## Service Entry Format

```yaml
# knowledge/registry/services/service.auth.yaml
identity:
  knowledge_id: "KNW-PLAT-SVC-001"
  canonical_name: "platform.service.auth"
  knowledge_uri: "knw://platform/service/auth"
  namespace: "platform"
  version: "2.0.0"
  owner: "team:platform-core"

service_spec:
  service_id: "service.auth"
  interface_path: "platform_kernel.contracts.auth.AuthService"
  implementations:
    - "platform_kernel.services.auth.JWTAuthService"
    - "platform_kernel.services.auth.APIKeyAuthService"
  default_implementation: "platform_kernel.services.auth.JWTAuthService"
  depends_on_services: []
  stability: STABLE
  capability_id: "capability:platform:auth:v2"
  boot_priority: 1

interface_contract:
  methods:
    - name: "authenticate"
      signature: "(token: str) -> AuthResult"
      idempotent: true
      latency_budget_ms: 5
    - name: "authorize"
      signature: "(user_id: str, resource: str, action: str) -> bool"
      idempotent: true
      latency_budget_ms: 2
    - name: "create_token"
      signature: "(user_id: str, scopes: list[str]) -> str"
      idempotent: false
      latency_budget_ms: 10

lifecycle:
  status: CANONICAL
```

---

## Service Discovery Protocol

```
DISCOVER_SERVICE(service_id):
  1. Look up service_id in service registry index
  2. Load service entry
  3. Read default_implementation path
  4. Verify implementation is available in current runtime
  5. Return ServiceRef{service_id, implementation_class, interface_class}

INJECT_SERVICE(service_id, consumer_module):
  1. DISCOVER_SERVICE(service_id)
  2. Instantiate implementation (singleton per runtime)
  3. Register with DI container
  4. Record usage in service usage index
```

---

## Service Registry Rules

### SR-001 — Interface Stability Contract
Once a service reaches STABLE status, its ABC interface may not have breaking changes without a MAJOR version bump and 90-day deprecation notice.

### SR-002 — Single Interface Per Service
Each `service_id` maps to exactly one ABC interface. Multiple implementations are allowed but must all fulfill the same interface.

### SR-003 — Circular Dependency Forbidden
`depends_on_services` must not form cycles. Validated at registration time (cycle detection algorithm from `25-GRAPH-ALGORITHMS`).

### SR-004 — Boot Priority Ordering
Services must be registered in boot_priority order (1=first). Consumers must declare their required services so boot ordering can be validated.

### SR-005 — Service Versioning
Service interface changes follow semantic versioning. The service_id includes no version — consumers receive the interface version registered in `version` field.

---

## Service Registry Extensions (Future Services)

Pre-registered slots for future platform services:

| Service ID | Purpose | Target Phase |
|-----------|---------|-------------|
| `service.knowledge` | Knowledge Object queries | 3.0D |
| `service.graph` | Knowledge Graph traversal | 3.0D |
| `service.compiler` | Knowledge compilation | 3.0E |
| `service.events` | Event bus | 3.1 |
| `service.scheduler` | Task scheduling | 3.1 |

---

## Service Registry Protocol

```python
class ServiceRegistry(KnowledgeRegistryProtocol):
    def discover(self, service_id: str) -> ServiceRef: ...
    def get_implementations(self, service_id: str) -> list[str]: ...
    def get_default_implementation(self, service_id: str) -> str: ...
    def set_default_implementation(self, service_id: str, impl_path: str) -> None: ...
    def get_boot_order(self) -> list[str]: ...
    def validate_no_cycles(self) -> list[list[str]]: ...
    def record_usage(self, service_id: str, consumer_id: str) -> None: ...
    def get_consumers(self, service_id: str) -> list[str]: ...
    def get_interface(self, service_id: str) -> ServiceInterface: ...
```

---

## Cross-References

- Registry base contract → `14-REGISTRY-ARCHITECTURE`
- Service objects (ServiceObject) → `15-OBJECT-REGISTRY`
- Capability registry → Phase 2.1D.0 `15-CAPABILITY-REGISTRY`
- Boot ordering → Phase 2.1D.0 `32-RUNTIME-INTEGRATION`
