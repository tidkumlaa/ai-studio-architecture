---
knowledge_id: KNW-PLAT-ARCH-007
title: "Platform Kernel Model"
status: AUTHORITATIVE
phase: "2.1D.0"
version: "1.0.0"
created: "2026-06-29"
purpose: "Complete specification of the PlatformKernel — DI, events, lifecycle, config, registry"
canonical_source: "architecture/docs/platform-consolidation/07-KERNEL-MODEL.md"
dependencies:
  - "02-OBJECT-MODEL.md"
  - "06-RUNTIME-MODEL.md"
related_documents:
  - "08-SDK-MODEL.md"
  - "10-DEPENDENCY-RULES.md"
  - "18-EVENT-MODEL.md"
  - "19-STATE-MACHINES.md"
acceptance_criteria:
  - "Kernel has zero external runtime dependencies"
  - "All 5 kernel subsystems are fully specified"
  - "Boot sequence is deterministic and ordered"
  - "Every kernel interface is a Python Protocol or ABC"
verification_checklist:
  - "[ ] Kernel imports only stdlib + platform.common"
  - "[ ] DI container supports async factories"
  - "[ ] Event bus is typed (no raw dict payloads)"
  - "[ ] Configuration supports environment overrides"
future_extensions:
  - "Kernel plugin hooks (pre/post boot)"
  - "Kernel telemetry export"
---

# Platform Kernel Model

## Kernel Axioms

1. **Zero external dependencies.** `platform.kernel` imports only: Python stdlib + `platform.common`.
2. **Boot first.** Kernel starts at order 0, before all runtimes.
3. **Universal provider.** Every runtime receives the kernel as its only constructor argument.
4. **Stateless core.** Kernel itself holds no AI state. It manages registries and wiring only.
5. **Typed interfaces.** Every kernel subsystem exposes a typed Python Protocol.

---

## Kernel Subsystems

```
platform/kernel/
├── bootstrap.py      ← ordered startup coordinator
├── config/           ← configuration management
├── di/               ← dependency injection container
├── events/           ← event bus and dispatcher
├── lifecycle/        ← runtime lifecycle manager
├── registry/         ← capability, service, module registries
└── exceptions/       ← kernel-level exceptions
```

---

## Subsystem 1: Bootstrap

**File:** `platform/kernel/bootstrap.py`

```python
class PlatformBootstrap:
    """Coordinates ordered startup of all registered runtimes."""

    async def boot(self, config: PlatformConfig) -> PlatformKernel:
        """
        1. Load config
        2. Initialise DI container
        3. Initialise event bus
        4. Start runtimes in boot_order ascending
        5. Register all capabilities
        6. Return ready kernel
        """

    async def shutdown(self) -> None:
        """Shutdown runtimes in reverse boot_order. Wait for graceful stop."""
```

**Boot sequence:**
```
Config.load()
    → DI.initialise()
    → EventBus.start()
    → [event_runtime.start(kernel)]      # boot order 1
    → [resource_runtime.start(kernel)]   # boot order 2
    → [provider_runtime.start(kernel)]   # boot order 3
    → [knowledge_runtime.start(kernel)]  # boot order 4
    → [ai_runtime.start(kernel)]         # boot order 5
    → [workflow_runtime.start(kernel)]   # boot order 6
    → [orchestration_runtime.start(kernel)] # boot order 7
    → CapabilityRegistry.lock()
    → Kernel.state = READY
```

---

## Subsystem 2: Configuration

**File:** `platform/kernel/config/`

```python
class PlatformConfig(BaseSettings):
    """Pydantic Settings — loads from environment, .env, and config files."""

    # Platform identity
    platform_version: str = "2.0.0"
    environment: Literal["development", "staging", "production"] = "development"

    # Kernel
    boot_timeout_seconds: float = 30.0
    health_poll_interval_seconds: float = 30.0
    max_runtime_restart_attempts: int = 3

    # Event bus
    event_bus_backend: Literal["memory", "redis", "amqp"] = "memory"
    event_bus_url: str | None = None
    event_bus_max_queue_size: int = 10_000

    # Runtimes (each runtime has its own sub-settings)
    ai_runtime: AIRuntimeConfig = AIRuntimeConfig()
    knowledge_runtime: KnowledgeRuntimeConfig = KnowledgeRuntimeConfig()
    # ... per-runtime config

    model_config = SettingsConfigDict(
        env_prefix="PLATFORM_",
        env_nested_delimiter="__",
    )
```

**Environment variables:**
```bash
PLATFORM_ENVIRONMENT=production
PLATFORM_EVENT_BUS_BACKEND=redis
PLATFORM_EVENT_BUS_URL=redis://localhost:6379
PLATFORM_AI_RUNTIME__MAX_CONCURRENT_REQUESTS=100
```

---

## Subsystem 3: Dependency Injection

**File:** `platform/kernel/di/`

```python
class DIContainer:
    """Lightweight DI container with async factory support."""

    def register(
        self,
        interface: type,
        implementation: type | AsyncCallable,
        scope: Literal["singleton", "transient", "request"] = "singleton",
    ) -> None: ...

    def resolve(self, interface: type[T]) -> T: ...

    async def resolve_async(self, interface: type[T]) -> T: ...

    def override(self, interface: type, implementation: Any) -> ContextManager: ...
```

**Registration example:**
```python
container.register(QuotaEngine, ResourceQuotaEngine, scope="singleton")
container.register(EventBus, InMemoryEventBus, scope="singleton")
container.register(ProviderPlugin, factory=lambda: load_plugins(), scope="singleton")
```

**Scope rules:**
- `singleton`: One instance for the Platform lifetime (most services)
- `transient`: New instance per call (factories, builders)
- `request`: New instance per HTTP request (request-scoped state)

---

## Subsystem 4: Event Bus

**File:** `platform/kernel/events/`

```python
class EventBus(Protocol):
    async def publish(self, event: PlatformEvent) -> None: ...
    async def subscribe(
        self,
        event_type: type[E],
        handler: Callable[[E], Awaitable[None]],
        filter: Callable[[E], bool] | None = None,
    ) -> Subscription: ...
    async def unsubscribe(self, subscription: Subscription) -> None: ...
```

**Event base type:**
```python
@dataclass(frozen=True)
class PlatformEvent:
    event_id: UUID
    event_type: str        # "ai.execution.completed"
    producer_id: str       # runtime or service ID
    timestamp: datetime
    correlation_id: UUID | None = None
    payload: dict = field(default_factory=dict)
```

**Delivery guarantees:**
- In-memory backend: at-least-once, no ordering guarantee
- Redis backend: at-least-once, FIFO per topic
- AMQP backend: exactly-once, causal ordering

**Error handling:** Failed handlers are retried 3 times with exponential backoff; then dead-lettered.

---

## Subsystem 5: Lifecycle Manager

**File:** `platform/kernel/lifecycle/`

```python
class LifecycleManager:
    """Tracks and manages runtime lifecycle transitions."""

    async def register_runtime(self, runtime: PlatformRuntime) -> None: ...
    async def start_all(self) -> None: ...
    async def stop_all(self) -> None: ...
    async def restart_runtime(self, runtime_id: str) -> None: ...
    def get_state(self, runtime_id: str) -> RuntimeState: ...
    async def wait_until_ready(self, timeout: float = 30.0) -> None: ...
```

**State machine:** See 19-STATE-MACHINES.md → Runtime Lifecycle.

---

## Subsystem 6: Registry

**File:** `platform/kernel/registry/`

```python
class CapabilityRegistry:
    def register(self, capability: PlatformCapability) -> None: ...
    def get(self, capability_id: str) -> PlatformCapability | None: ...
    def list(self, domain: str | None = None) -> list[PlatformCapability]: ...
    def lock(self) -> None: ...  # called after all runtimes booted

class ServiceRegistry:
    def register(self, service: PlatformService) -> None: ...
    def get(self, service_id: str) -> PlatformService | None: ...
    def list(self) -> list[PlatformService]: ...

class ModuleRegistry:
    def register(self, module: PlatformModule) -> None: ...
    def get(self, module_id: str) -> PlatformModule | None: ...
    def list(self, runtime_id: str | None = None) -> list[PlatformModule]: ...
```

After `CapabilityRegistry.lock()`, no new capabilities may be registered.
This prevents runtime code from injecting capabilities after boot.

---

## Kernel as Injection Root

Runtimes receive the kernel and use it to resolve their dependencies:

```python
class AIRuntime:
    async def start(self, kernel: PlatformKernel) -> None:
        self._quota = kernel.di.resolve(QuotaEngine)
        self._events = kernel.events
        self._config = kernel.config.ai_runtime
        kernel.registry.capability.register(
            PlatformCapability(id="capability:ai:routing:v1", ...)
        )
```

---

## Kernel Exceptions

```python
class KernelError(PlatformError): ...
class BootTimeoutError(KernelError): ...
class RuntimeNotReadyError(KernelError): ...
class CapabilityNotFoundError(KernelError): ...
class CapabilityRegistryLockedError(KernelError): ...
class CircularDependencyError(KernelError): ...
class ConfigurationError(KernelError): ...
```
