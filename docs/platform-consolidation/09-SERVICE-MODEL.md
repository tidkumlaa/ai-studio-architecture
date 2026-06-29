---
knowledge_id: KNW-PLAT-ARCH-009
title: "Platform Service Model"
status: AUTHORITATIVE
phase: "2.1D.0"
version: "1.0.0"
created: "2026-06-29"
purpose: "Define all cross-cutting platform services, their interfaces, and dependency rules"
canonical_source: "architecture/docs/platform-consolidation/09-SERVICE-MODEL.md"
dependencies:
  - "07-KERNEL-MODEL.md"
related_documents:
  - "08-SDK-MODEL.md"
  - "16-SERVICE-REGISTRY.md"
  - "10-DEPENDENCY-RULES.md"
acceptance_criteria:
  - "Every service has a Python ABC or Protocol interface"
  - "Service dependency graph is acyclic"
  - "Every service has a default (no-op or in-memory) implementation"
verification_checklist:
  - "[ ] Service interfaces are Python Protocols"
  - "[ ] No circular dependencies between services"
  - "[ ] Default implementations exist for all services"
future_extensions:
  - "Service mesh integration (Istio/Envoy)"
  - "Service-level SLOs defined in service.yaml"
---

# Platform Service Model

## Service Catalogue

| Service ID | Name | Stability | Depends On |
|-----------|------|-----------|------------|
| `service.auth` | Authentication & Authorization | STABLE | kernel |
| `service.audit` | Audit Logging | STABLE | kernel, service.auth |
| `service.metrics` | Prometheus Metrics | STABLE | kernel |
| `service.tracing` | Distributed Tracing | BETA | kernel, service.metrics |
| `service.cache` | Caching | STABLE | kernel |
| `service.notifications` | Alerts & Notifications | BETA | kernel, service.audit |

---

## Service Contract Template

Every service follows this pattern:

```python
class PlatformServiceBase(ABC):
    """Base class for all platform services."""
    service_id: ClassVar[str]
    service_name: ClassVar[str]

    @abstractmethod
    async def start(self, kernel: PlatformKernel) -> None: ...

    @abstractmethod
    async def stop(self) -> None: ...

    @abstractmethod
    async def health(self) -> ServiceHealth: ...
```

---

## Service 1: Authentication & Authorization

```python
class AuthService(PlatformServiceBase):
    service_id = "service.auth"

    async def authenticate(self, token: str) -> AuthContext | None:
        """Validate token and return auth context, or None if invalid."""

    async def authorize(self, context: AuthContext, resource: str, action: str) -> bool:
        """Check if context has permission to perform action on resource."""

    async def create_token(self, org_id: UUID, scopes: list[str]) -> str:
        """Issue a new JWT token."""

    async def revoke_token(self, token: str) -> None:
        """Invalidate a token immediately."""
```

**Implementations:**
- `JWTAuthService` — default; validates JWT via `python-jose`
- `NoOpAuthService` — development/testing; accepts all tokens

---

## Service 2: Audit Logging

```python
class AuditService(PlatformServiceBase):
    service_id = "service.audit"

    async def log(self, entry: AuditEntry) -> None:
        """Record an audit event."""

    async def query(
        self,
        org_id: UUID,
        since: datetime,
        action: str | None = None,
        limit: int = 100,
    ) -> list[AuditEntry]: ...

@dataclass
class AuditEntry:
    entry_id: UUID
    org_id: UUID
    actor_id: str
    action: str         # "ai.execution.started", "quota.exceeded", etc.
    resource_id: str
    outcome: Literal["SUCCESS", "FAILURE", "DENIED"]
    timestamp: datetime
    metadata: dict
```

**Implementations:**
- `SQLiteAuditService` — default; persists to SQLite
- `NoOpAuditService` — testing

---

## Service 3: Metrics

```python
class MetricsService(PlatformServiceBase):
    service_id = "service.metrics"

    def counter(self, name: str, labels: dict) -> Counter: ...
    def gauge(self, name: str, labels: dict) -> Gauge: ...
    def histogram(self, name: str, labels: dict, buckets: list[float] | None = None) -> Histogram: ...
    def expose(self) -> str:
        """Return Prometheus text format for /metrics endpoint."""
```

**Standard platform metrics:**
```
platform_ai_requests_total{provider, model, status}
platform_ai_tokens_total{provider, model, type}
platform_ai_cost_usd_total{provider, model}
platform_quota_usage_pct{org_id, provider}
platform_runtime_health{runtime}
platform_event_published_total{event_type}
```

---

## Service 4: Distributed Tracing (BETA)

```python
class TracingService(PlatformServiceBase):
    service_id = "service.tracing"

    def start_span(self, name: str, parent: Span | None = None) -> Span: ...
    def finish_span(self, span: Span, status: str = "OK") -> None: ...

    @contextmanager
    def trace(self, name: str) -> Iterator[Span]: ...
```

**Backends:** OpenTelemetry (default); Jaeger; Zipkin.

---

## Service 5: Caching

```python
class CacheService(PlatformServiceBase):
    service_id = "service.cache"

    async def get(self, key: str) -> bytes | None: ...
    async def set(self, key: str, value: bytes, ttl: int | None = None) -> None: ...
    async def delete(self, key: str) -> None: ...
    async def exists(self, key: str) -> bool: ...
```

**Backends:**
- `InMemoryCache` — default; TTL-aware LRU cache
- `RedisCache` — production; requires `PLATFORM_CACHE_URL`

---

## Service 6: Notifications

```python
class NotificationService(PlatformServiceBase):
    service_id = "service.notifications"

    async def send(self, notification: Notification) -> None: ...
    async def send_batch(self, notifications: list[Notification]) -> None: ...

@dataclass
class Notification:
    recipient: str       # org_id, user_id, or channel name
    channel: Literal["webhook", "email", "slack", "log"]
    subject: str
    body: str
    priority: Literal["critical", "warning", "info"]
    metadata: dict
```

---

## Service Dependency Graph

```
kernel
 ├── service.auth
 │    └── service.audit
 ├── service.metrics
 │    └── service.tracing
 ├── service.cache
 └── service.notifications
      └── service.audit
```

All arrows point from dependent → dependency.
No cycles permitted (enforced by layer linter).

---

## Service Registration

Services are registered in DI container at kernel boot, before any runtime starts:

```python
# kernel/bootstrap.py
container.register(AuthService, JWTAuthService, scope="singleton")
container.register(AuditService, SQLiteAuditService, scope="singleton")
container.register(MetricsService, PrometheusMetricsService, scope="singleton")
container.register(CacheService, InMemoryCache, scope="singleton")
container.register(TracingService, OpenTelemetryTracingService, scope="singleton")
container.register(NotificationService, WebhookNotificationService, scope="singleton")
```

Services are available to all runtimes via `kernel.di.resolve(ServiceType)`.
