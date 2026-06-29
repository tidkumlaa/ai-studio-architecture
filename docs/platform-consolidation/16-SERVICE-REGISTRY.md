---
knowledge_id: KNW-PLAT-ARCH-016
title: "Platform Service Registry"
status: AUTHORITATIVE
phase: "2.1D.0"
version: "1.0.0"
created: "2026-06-29"
purpose: "Complete registry of all platform services with interfaces, implementations, and DI registration"
canonical_source: "architecture/docs/platform-consolidation/16-SERVICE-REGISTRY.md"
dependencies:
  - "09-SERVICE-MODEL.md"
  - "02-OBJECT-MODEL.md"
related_documents:
  - "15-CAPABILITY-REGISTRY.md"
  - "17-MODULE-REGISTRY.md"
  - "07-KERNEL-MODEL.md"
acceptance_criteria:
  - "Every service has an ABC interface and at least one implementation"
  - "Every service has a capability ID"
  - "DI registration pattern documented for all services"
verification_checklist:
  - "[ ] All 6 services listed"
  - "[ ] Interfaces are Python ABCs"
  - "[ ] No service creates a dependency cycle"
future_extensions:
  - "Service health dashboard"
  - "Service canary deployment pattern"
---

# Platform Service Registry

## Service ID Convention

```
service.{name}
```
Examples: `service.auth`, `service.metrics`

---

## Service 1 — Auth

| Field | Value |
|-------|-------|
| ID | `service.auth` |
| Capability | `capability:service:auth:v1` |
| Stability | STABLE |
| Layer | SERVICES (3) |
| Depends on | `service.cache` |

**Interface (ABC):**
```python
class AuthService(ABC):
    @abstractmethod
    def verify_token(self, token: str) -> AuthContext: ...

    @abstractmethod
    def create_token(self, identity: str, scopes: list[str]) -> str: ...

    @abstractmethod
    def revoke_token(self, token: str) -> None: ...
```

**Default Implementation:** `platform.services.auth.JWTAuthService`

**DI Registration:**
```python
container.register(AuthService, JWTAuthService, lifetime=Singleton)
```

---

## Service 2 — Audit

| Field | Value |
|-------|-------|
| ID | `service.audit` |
| Capability | `capability:service:audit:v1` |
| Stability | STABLE |
| Layer | SERVICES (3) |
| Depends on | `service.auth` |

**Interface (ABC):**
```python
class AuditService(ABC):
    @abstractmethod
    def log(self, event: AuditEvent) -> None: ...

    @abstractmethod
    def query(self, filter: AuditFilter) -> list[AuditEvent]: ...
```

**Default Implementation:** `platform.services.audit.FileAuditService`

**DI Registration:**
```python
container.register(AuditService, FileAuditService, lifetime=Singleton)
```

---

## Service 3 — Metrics

| Field | Value |
|-------|-------|
| ID | `service.metrics` |
| Capability | `capability:service:metrics:v1` |
| Stability | STABLE |
| Layer | SERVICES (3) |
| Depends on | None |

**Interface (ABC):**
```python
class MetricsService(ABC):
    @abstractmethod
    def counter(self, name: str, value: int = 1, labels: dict | None = None) -> None: ...

    @abstractmethod
    def gauge(self, name: str, value: float, labels: dict | None = None) -> None: ...

    @abstractmethod
    def histogram(self, name: str, value: float, labels: dict | None = None) -> None: ...

    @abstractmethod
    def snapshot(self) -> MetricsSnapshot: ...
```

**Default Implementation:** `platform.services.metrics.InMemoryMetricsService`

**DI Registration:**
```python
container.register(MetricsService, InMemoryMetricsService, lifetime=Singleton)
```

---

## Service 4 — Tracing

| Field | Value |
|-------|-------|
| ID | `service.tracing` |
| Capability | `capability:service:tracing:v1` |
| Stability | BETA |
| Layer | SERVICES (3) |
| Depends on | None |

**Interface (ABC):**
```python
class TracingService(ABC):
    @abstractmethod
    def start_span(self, name: str, parent: Span | None = None) -> Span: ...

    @abstractmethod
    def finish_span(self, span: Span) -> None: ...

    @abstractmethod
    def inject_headers(self, span: Span) -> dict[str, str]: ...
```

**Default Implementation:** `platform.services.tracing.NoOpTracingService`

**DI Registration:**
```python
container.register(TracingService, NoOpTracingService, lifetime=Singleton)
```

---

## Service 5 — Cache

| Field | Value |
|-------|-------|
| ID | `service.cache` |
| Capability | `capability:service:cache:v1` |
| Stability | STABLE |
| Layer | SERVICES (3) |
| Depends on | None |

**Interface (ABC):**
```python
class CacheService(ABC):
    @abstractmethod
    def get(self, key: str) -> bytes | None: ...

    @abstractmethod
    def set(self, key: str, value: bytes, ttl: int | None = None) -> None: ...

    @abstractmethod
    def delete(self, key: str) -> None: ...

    @abstractmethod
    def clear(self, pattern: str = "*") -> int: ...
```

**Default Implementation:** `platform.services.cache.InMemoryCacheService`

**DI Registration:**
```python
container.register(CacheService, InMemoryCacheService, lifetime=Singleton)
```

---

## Service 6 — Notifications

| Field | Value |
|-------|-------|
| ID | `service.notifications` |
| Capability | `capability:service:notifications:v1` |
| Stability | BETA |
| Layer | SERVICES (3) |
| Depends on | `service.audit` |

**Interface (ABC):**
```python
class NotificationService(ABC):
    @abstractmethod
    def send(self, notification: Notification) -> str: ...  # returns notification_id

    @abstractmethod
    def list_pending(self) -> list[Notification]: ...
```

**Default Implementation:** `platform.services.notifications.InMemoryNotificationService`

**DI Registration:**
```python
container.register(NotificationService, InMemoryNotificationService, lifetime=Singleton)
```

---

## Service Dependency Graph

```
service.auth ──────────────────────────────────────────────┐
    │                                                       │
    └──▶ service.cache                                      │
                                                            │
service.audit ──────────────────────────────────────────▶ service.auth
    │
    └──▶ service.notifications
```

**Acyclic constraint:** No cycles exist in the above graph. CI enforces this.

---

## Service Registry Summary

| Service | Stability | Depends On | Implementations |
|---------|-----------|------------|-----------------|
| `service.auth` | STABLE | `service.cache` | JWTAuthService |
| `service.audit` | STABLE | `service.auth` | FileAuditService |
| `service.metrics` | STABLE | — | InMemoryMetricsService |
| `service.tracing` | BETA | — | NoOpTracingService |
| `service.cache` | STABLE | — | InMemoryCacheService |
| `service.notifications` | BETA | `service.audit` | InMemoryNotificationService |

---

## Standard Service Metrics

Every service implementation must instrument these metrics:

| Metric | Type | Labels |
|--------|------|--------|
| `platform_service_{name}_calls_total` | Counter | `method`, `status` |
| `platform_service_{name}_latency_seconds` | Histogram | `method` |
| `platform_service_{name}_errors_total` | Counter | `method`, `error_type` |

Emitted via `MetricsService.counter()` and `MetricsService.histogram()`.
