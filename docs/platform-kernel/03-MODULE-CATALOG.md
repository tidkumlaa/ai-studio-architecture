# Platform Kernel — Module Catalog (All 40 Modules)
# Phase 2.0D.2.6A

Spec format per module:
  Purpose | Responsibilities | Public Interface | Lifecycle | Dependencies |
  Extension Points | Events | Configuration | Error Model | Thread Safety |
  Async Model | Performance Target

---

## GROUP A — OBJECT FOUNDATION

---

### A1. PlatformObject
**Purpose:** Root base class for all kernel-managed objects.
**Spec:** See `02-OBJECT-MODEL.md §2`
**Performance Target:** Construction < 1 ms; metadata read < 0.01 ms

---

### A2. PlatformIdentity
**Purpose:** Immutable value type capturing full object identity.
**Spec:** See `02-OBJECT-MODEL.md §3`
**Performance Target:** URI generation < 0.1 ms; comparison O(1)

---

### A3. PlatformLifecycle

**Purpose:** Manages the 10-state lifecycle of any PlatformObject.

**Responsibilities:**
- Own and enforce the state machine
- Log every transition to PlatformAudit
- Emit lifecycle events on every transition
- Trigger registered transition hooks
- Prevent illegal transitions (raise PlatformException)

**Public Interface:**
```python
class PlatformLifecycle:
    @property
    def state(self) -> LifecycleState: ...
    @property
    def owner_id(self) -> str: ...

    def initialize(self) -> None: ...
    def start(self) -> None: ...
    def pause(self) -> None: ...
    def resume(self) -> None: ...
    def stop(self) -> None: ...
    def fail(self, reason: str) -> None: ...
    def recover(self) -> None: ...
    def dispose(self) -> None: ...

    def on_transition(
        self, 
        from_state: LifecycleState,
        to_state: LifecycleState,
        callback: Callable,
    ) -> "SubscriptionHandle": ...

    @property
    def history(self) -> list[tuple[LifecycleState, datetime]]: ...
    
    def assert_state(self, *allowed: LifecycleState) -> None: ...
        # raises PlatformException if not in allowed states
```

**Lifecycle:** PlatformLifecycle itself is born in INITIALIZED state and
disposed when its owner object is disposed.

**Dependencies:** PlatformAudit (for transition logging), PlatformEventBus

**Events Emitted:**
- `lifecycle.initialized`
- `lifecycle.started`
- `lifecycle.paused`
- `lifecycle.resumed`
- `lifecycle.stopping`
- `lifecycle.stopped`
- `lifecycle.failed`
- `lifecycle.recovered`
- `lifecycle.disposing`
- `lifecycle.disposed`

**Error Model:**
- `INVALID_TRANSITION` — attempted illegal state change
- `ALREADY_DISPOSED` — operation on disposed lifecycle

**Thread Safety:** State reads and transitions protected by RLock.
**Performance Target:** Transition < 0.5 ms including audit write

---

### A4. PlatformStateMachine

**Purpose:** Generic, reusable finite state machine engine used by
PlatformLifecycle and by any kernel component needing governed state.

**Responsibilities:**
- Define typed states and typed transitions
- Enforce guard conditions before transitions
- Execute entry/exit actions on transition
- Support hierarchical states (sub-states)
- Record full transition history

**Public Interface:**
```python
class PlatformStateMachine(Generic[S, E]):
    """S = state enum type, E = event/trigger type."""
    
    def add_state(self, state: S, entry: Callable | None = None,
                  exit: Callable | None = None) -> None: ...
    
    def add_transition(
        self,
        from_state: S,
        event: E,
        to_state: S,
        guard: Callable[[], bool] | None = None,
        action: Callable | None = None,
    ) -> None: ...
    
    def trigger(self, event: E, **kwargs) -> S: ...
        # returns new state; raises PlatformException if no valid transition
    
    @property
    def current(self) -> S: ...
    
    def can_trigger(self, event: E) -> bool: ...
    
    @property
    def history(self) -> list[tuple[S, E, S, datetime]]: ...

    def reset(self, initial_state: S) -> None: ...
```

**Configuration:**
- `max_history_entries`: int (default: 1000)
- `emit_events`: bool (default: True)

**Thread Safety:** Full RLock protection on all transitions.
**Performance Target:** Transition < 0.1 ms

---

### A5. PlatformVersion
**Purpose:** Immutable semantic version value object.
**Spec:** See `02-OBJECT-MODEL.md §6`
**Performance Target:** Parse < 0.01 ms; comparison O(1)

---

### A6. PlatformCapability
**Purpose:** Immutable capability token. Granted to objects as proof of access.
**Spec:** See `02-OBJECT-MODEL.md §7`
**Performance Target:** Capability check < 0.1 ms

---

## GROUP B — EVENT SYSTEM

---

### B1. PlatformEvent

**Purpose:** Immutable message representing a significant occurrence in the
kernel. The unit of communication across all kernel components.

**Responsibilities:**
- Carry event type, payload, correlation, and routing metadata
- Enforce immutability after construction
- Support event inheritance (typed event hierarchy)
- Carry priority classification
- Support persistence flag for durable events

**Public Interface:**
```python
@dataclass(frozen=True)
class PlatformEvent:
    # Identity
    event_id: str               # UUID v7
    event_type: str             # e.g. "kernel.lifecycle.started"
    event_version: str          # "1.0"
    
    # Payload
    payload: dict[str, Any]     # typed, JSON-serializable
    schema_version: str         # payload schema version
    
    # Correlation
    correlation_id: str         # UUID — ties events in a flow
    causation_id: str | None    # event_id that caused this event
    trace_id: str               # distributed trace ID
    span_id: str                # span within trace
    
    # Routing
    source_id: str              # object_id of emitting object
    source_type: str            # object_type of emitter
    target_id: str | None       # targeted object_id (None = broadcast)
    
    # Metadata
    timestamp: datetime         # UTC, microsecond precision
    sequence: int               # monotonic within source
    priority: EventPriority     # CRITICAL > HIGH > NORMAL > LOW > BACKGROUND
    persistent: bool            # should this event survive a restart?
    tags: frozenset[str]        # searchable tags
    
    # Methods
    def with_correlation(self, correlation_id: str) -> "PlatformEvent": ...
    def with_causation(self, causation_id: str) -> "PlatformEvent": ...
    def matches(self, pattern: str) -> bool: ...
        # glob: "kernel.lifecycle.*", "kernel.*.started"

class EventPriority(enum.IntEnum):
    BACKGROUND = 0
    LOW        = 1
    NORMAL     = 2
    HIGH       = 3
    CRITICAL   = 4
```

**Event Type Naming Convention:**
```
{domain}.{subdomain}.{verb}
kernel.lifecycle.started
kernel.plugin.loaded
kernel.resource.exhausted
kernel.security.access.denied
```

**Error Model:**
- Events are never "rejected" at the PlatformEvent level.
  Routing errors are emitted as separate `kernel.event.routing.failed` events.

**Performance Target:** Construction < 0.01 ms; serialization < 0.1 ms

---

### B2. PlatformEventBus

**Purpose:** The central in-process event distribution fabric. Implements
publish-subscribe with priority ordering, filtering, dead-letter handling,
and optional persistence.

**Responsibilities:**
- Route events from publishers to all matching subscribers
- Enforce priority ordering (CRITICAL events skip the queue)
- Filter events by type pattern, source, tags
- Maintain a dead-letter queue for events with no subscribers
- Support event replay from a persisted log
- Support synchronous and asynchronous delivery modes
- Enforce subscriber isolation (one slow subscriber does not block others)

**Public Interface:**
```python
class PlatformEventBus:
    # Publishing
    def publish(self, event: PlatformEvent) -> None: ...
        # fire-and-forget; thread-safe

    async def publish_async(self, event: PlatformEvent) -> None: ...
    
    def publish_many(self, events: list[PlatformEvent]) -> None: ...
        # ordered batch; maintains sequence

    # Subscribing
    def subscribe(
        self,
        pattern: str,                           # glob event type pattern
        handler: Callable[[PlatformEvent], None],
        *,
        priority: int = 0,                      # handler priority (higher = earlier)
        filter: Callable[[PlatformEvent], bool] | None = None,
        delivery: DeliveryMode = DeliveryMode.ASYNC,
        max_queue: int = 1000,
    ) -> "SubscriptionHandle": ...
    
    async def subscribe_async(
        self,
        pattern: str,
        handler: Callable[[PlatformEvent], Awaitable[None]],
        **kwargs,
    ) -> "SubscriptionHandle": ...

    def unsubscribe(self, handle: "SubscriptionHandle") -> None: ...

    # Replay
    def replay(
        self,
        pattern: str,
        since: datetime | None = None,
        until: datetime | None = None,
        correlation_id: str | None = None,
    ) -> list[PlatformEvent]: ...

    # Dead letter
    @property
    def dead_letter(self) -> "DeadLetterQueue": ...

    # Introspection
    def subscription_count(self, pattern: str | None = None) -> int: ...
    def pending_count(self) -> int: ...

class SubscriptionHandle:
    subscription_id: str
    pattern: str
    created_at: datetime
    event_count: int            # total events delivered to this subscription
    
    def pause(self) -> None: ...
    def resume(self) -> None: ...
    def cancel(self) -> None: ...

class DeadLetterQueue:
    def peek(self, limit: int = 100) -> list[PlatformEvent]: ...
    def requeue(self, event_id: str) -> None: ...
    def discard(self, event_id: str) -> None: ...
    def drain(self) -> list[PlatformEvent]: ...

class DeliveryMode(enum.Enum):
    SYNC  = "sync"     # handler called on publisher's thread
    ASYNC = "async"    # handler dispatched to event loop / thread pool
```

**Event Ordering Guarantees:**
- Within a single source: events are delivered in sequence order
- Across sources at same priority: best-effort FIFO
- CRITICAL priority: synchronous delivery, blocks publisher until complete
- BACKGROUND priority: delivered in idle cycles only

**Persistence Model:**
- Events with `persistent=True` are written to PlatformAudit before delivery
- On restart, persistent events are replayed to subscribers that request replay

**Dead Letter Rules:**
- Events with no matching subscriber → dead letter after 1 routing attempt
- Events that cause subscriber exceptions → dead letter after `max_retries`
- Dead letter events emit `kernel.event.dead_letter` notification

**Configuration:**
- `max_queue_depth`: int (default: 10_000)
- `max_retries`: int (default: 3)
- `retry_delay_ms`: int (default: 100)
- `persistence_enabled`: bool (default: True)
- `replay_window_hours`: int (default: 24)

**Thread Safety:** Lock-free internal queue (collections.deque + threading.Event).
Publishers never block on subscriber slow consumption.
**Performance Target:** Dispatch per event < 0.1 ms; 50,000 events/sec throughput

---

## GROUP C — SERVICE INFRASTRUCTURE

---

### C1. PlatformRegistry

**Purpose:** Central, typed, thread-safe catalog of all active kernel
services, components, and extensions. The single source of truth for
"what is running."

**Responsibilities:**
- Register/unregister any PlatformObject by type and ID
- Support lookup by type, capability, tag, metadata
- Emit events on registration and unregistration
- Detect duplicate registrations (configurable: error or replace)
- Support registration with lease (auto-expire after TTL)
- Validate that registered objects are in a healthy lifecycle state

**Public Interface:**
```python
class PlatformRegistry:
    def register(
        self,
        obj: PlatformObject,
        *,
        tags: list[str] | None = None,
        ttl_seconds: int | None = None,
        replace_on_duplicate: bool = False,
    ) -> "RegistrationHandle": ...
    
    def unregister(self, object_id: str) -> None: ...
    
    def get(self, object_id: str) -> PlatformObject | None: ...
    
    def get_typed(self, object_id: str, expected_type: type[T]) -> T | None: ...
    
    def find_by_type(self, object_type: type[T]) -> list[T]: ...
    
    def find_by_capability(self, capability: str) -> list[PlatformObject]: ...
    
    def find_by_tag(self, *tags: str) -> list[PlatformObject]: ...
    
    def exists(self, object_id: str) -> bool: ...
    
    def count(self, object_type: type | None = None) -> int: ...
    
    def snapshot(self) -> list["RegistrationRecord"]: ...

@dataclass(frozen=True)
class RegistrationRecord:
    object_id: str
    object_type: str
    registered_at: datetime
    ttl_expires_at: datetime | None
    tags: frozenset[str]
```

**Events:**
- `kernel.registry.registered` — new object registered
- `kernel.registry.unregistered` — object unregistered
- `kernel.registry.expired` — TTL-based expiry triggered

**Error Model:**
- `DUPLICATE_REGISTRATION` — object already registered (when not replacing)
- `NOT_REGISTERED` — lookup by ID of unknown object
- `TYPE_MISMATCH` — get_typed called with wrong expected type

**Thread Safety:** RLock on all mutations; read-only queries lock-free via
copy-on-write snapshot.
**Performance Target:** Register/unregister < 0.5 ms; lookup < 0.05 ms

---

### C2. PlatformPlugin

**Purpose:** Represents a dynamically loaded unit of functionality.
Plugins extend the kernel without modifying kernel code.

**Responsibilities:**
- Load from a PlatformManifest
- Validate manifest signature before loading
- Isolate plugin failures from kernel stability
- Expose a typed API surface declared in the manifest
- Honor capability grants declared in the manifest
- Support lazy initialization and hot-reload

**Public Interface:**
```python
class PlatformPlugin(PlatformComponent):
    @property
    def manifest(self) -> "PlatformManifest": ...
    
    @property
    def plugin_id(self) -> str: ...
    
    @property
    def plugin_version(self) -> PlatformVersion: ...
    
    @property
    def capabilities_granted(self) -> frozenset[PlatformCapability]: ...
    
    def get_service(self, service_type: type[T]) -> T: ...
        # access a service exported by this plugin
    
    def call(self, operation: str, **kwargs) -> Any: ...
        # invoke a named operation exposed by the plugin

    def reload(self) -> None: ...
        # hot-reload (must be in RUNNING or STOPPED state)

class PluginManager:
    def load(self, manifest: "PlatformManifest") -> PlatformPlugin: ...
    def unload(self, plugin_id: str) -> None: ...
    def reload(self, plugin_id: str) -> None: ...
    def loaded_plugins(self) -> list[PlatformPlugin]: ...
    def discover(self, search_path: str) -> list["PlatformManifest"]: ...
```

**Lifecycle:** NEW → LOADED → INITIALIZED → RUNNING → STOPPED → UNLOADED

**Error Model:**
- `PLUGIN_MANIFEST_INVALID` — manifest failed validation
- `PLUGIN_CAPABILITY_DENIED` — plugin requests capability it was not granted
- `PLUGIN_LOAD_FAILED` — import/initialization error
- `PLUGIN_NOT_FOUND` — lookup of unloaded plugin

**Extension Points:**
- `on_before_load` hook — validate or transform manifest
- `on_after_load` hook — post-load configuration
- `on_unload` hook — cleanup before unload

**Thread Safety:** Plugin loading is serialized; plugin calls are concurrent.
**Performance Target:** Load < 50 ms; call overhead < 1 ms

---

### C3. PlatformServiceLocator

**Purpose:** Governed facade over the kernel's injected service graph.
Allows component code to resolve services without direct constructor
injection when the dependency is optional or context-dependent.

**Note:** This is NOT a general-purpose service locator anti-pattern.
Access is governed by PlatformCapability. Only services explicitly
registered in PlatformRegistry are resolvable.

**Public Interface:**
```python
class PlatformServiceLocator:
    def resolve(self, service_type: type[T]) -> T: ...
        # raises PlatformException if not registered or no capability
    
    def try_resolve(self, service_type: type[T]) -> T | None: ...
    
    def resolve_all(self, service_type: type[T]) -> list[T]: ...
    
    def is_registered(self, service_type: type[T]) -> bool: ...
    
    def with_capability(
        self, capability: PlatformCapability
    ) -> "PlatformServiceLocator": ...
        # scoped locator requiring capability check
```

**Dependencies:** PlatformRegistry, PlatformSecurity (capability check)
**Performance Target:** resolve() < 0.05 ms (cached after first resolution)

---

### C4. PlatformModule

**Purpose:** A named, versioned, self-contained bundle of related kernel
components. Modules are the unit of deployment and dependency declaration.

**Responsibilities:**
- Declare provided and required capabilities
- Own a collection of PlatformPlugin instances
- Declare dependencies on other modules (by name + version constraint)
- Boot in dependency order
- Support module-level configuration

**Public Interface:**
```python
class PlatformModule(PlatformComponent):
    @property
    def module_id(self) -> str: ...
    
    @property
    def module_name(self) -> str: ...
    
    @property
    def module_version(self) -> PlatformVersion: ...
    
    @property
    def required_capabilities(self) -> frozenset[PlatformCapability]: ...
    
    @property
    def provided_capabilities(self) -> frozenset[PlatformCapability]: ...
    
    @property
    def dependencies(self) -> list["ModuleDependency"]: ...
    
    def plugins(self) -> list[PlatformPlugin]: ...
    
    def activate(self) -> None: ...
    def deactivate(self) -> None: ...

@dataclass(frozen=True)
class ModuleDependency:
    module_name: str
    version_constraint: str   # semver constraint e.g. ">=1.0.0,<2.0.0"
    optional: bool = False
```

**Performance Target:** Module boot (no plugins) < 10 ms

---

### C5. PlatformManifest

**Purpose:** Immutable declaration of a plugin or module's identity,
capabilities, dependencies, exports, and configuration schema.
The manifest is validated before any code from the plugin is loaded.

**Public Interface:**
```python
@dataclass(frozen=True)
class PlatformManifest:
    plugin_id: str
    name: str
    version: PlatformVersion
    description: str
    author: str
    license: str
    
    # Capability declarations
    required_capabilities: frozenset[str]
    provided_capabilities: frozenset[str]
    
    # Module dependencies
    dependencies: tuple["ManifestDependency", ...]
    
    # Python entry point
    entry_module: str          # importable module path
    entry_class: str           # class name within entry_module
    
    # Exports (services this plugin registers)
    exports: tuple[str, ...]   # fully-qualified type names
    
    # Configuration schema (JSON Schema)
    config_schema: dict | None
    
    # Signature (for integrity verification)
    signature: str | None
    
    @classmethod
    def from_yaml(cls, path: str) -> "PlatformManifest": ...
    
    def validate(self) -> list[str]: ...
        # returns list of validation errors; empty = valid
    
    def to_yaml(self) -> str: ...
```

**Manifest File Format:**
```yaml
plugin_id: "ai-studio.runtime.provider.anthropic"
name: "Anthropic Provider Plugin"
version: "1.0.0"
description: "Anthropic Claude API adapter"
author: "AI Studio Platform Team"
license: "Proprietary"

required_capabilities:
  - kernel.network.outbound
  - kernel.config.read

provided_capabilities:
  - ai.provider.anthropic

dependencies:
  - name: "ai-studio.platform.core"
    version: ">=1.0.0,<2.0.0"

entry_module: "ai_runtime.adapters.anthropic"
entry_class: "AnthropicProviderPlugin"

exports:
  - "ai_runtime.adapters.anthropic.AnthropicProvider"

config_schema:
  type: object
  required: [api_key_secret]
  properties:
    api_key_secret:
      type: string
      description: "Name of secret in PlatformSecurity secrets store"
```

---

## GROUP D — CONFIGURATION

---

### D1. PlatformConfiguration

**Purpose:** Hierarchical, typed, thread-safe configuration system
supporting multiple sources, hot-reload, and change notification.

**Responsibilities:**
- Aggregate configuration from multiple sources (files, env vars,
  remote config, defaults)
- Apply source priority (env > remote > file > default)
- Validate values against typed schemas
- Notify subscribers when configuration changes
- Support namespaced configuration per module/plugin

**Public Interface:**
```python
class PlatformConfiguration:
    def get(self, key: str, default: T | None = None) -> T | None: ...
    def get_required(self, key: str) -> Any: ...
        # raises PlatformException if missing
    
    def get_typed(self, key: str, type_: type[T], default: T | None = None) -> T: ...
    
    def get_section(self, prefix: str) -> "ConfigSection": ...
        # returns a scoped view of keys matching prefix
    
    def set(self, key: str, value: Any, source: str = "runtime") -> None: ...
        # only for runtime-override; cannot override env vars
    
    def on_change(
        self, 
        key_pattern: str,
        callback: Callable[[str, Any, Any], None],
    ) -> "SubscriptionHandle": ...
        # callback(key, old_value, new_value)
    
    def reload(self) -> None: ...
        # reload from all sources
    
    def snapshot(self) -> dict[str, Any]: ...
        # complete flattened key-value snapshot

class ConfigSection:
    def get(self, key: str, default: T | None = None) -> T | None: ...
    def all(self) -> dict[str, Any]: ...
    def has(self, key: str) -> bool: ...
```

**Configuration Key Convention:**
```
{domain}.{module}.{setting}
kernel.eventbus.max_queue_depth
kernel.scheduler.max_workers
ai.provider.anthropic.timeout_seconds
```

**Source Priority (highest to lowest):**
1. Environment variables (PLATFORM_{KEY_UPPER})
2. Runtime overrides (set via API)
3. Remote configuration service (if configured)
4. Configuration files (YAML, TOML, JSON)
5. Module defaults (declared in manifest config_schema)

**Thread Safety:** All reads lock-free; writes use RLock + notify.
**Performance Target:** get() < 0.1 ms (cached); reload < 50 ms

---

### D2. PlatformSettings

**Purpose:** Immutable, typed settings snapshot for a specific module
or component. Loaded from PlatformConfiguration at component initialization
and frozen thereafter.

```python
@dataclass(frozen=True)
class PlatformSettings:
    namespace: str
    values: dict[str, Any]
    loaded_at: datetime
    source_hash: str            # hash of source config at load time
    
    def get(self, key: str, default: Any = None) -> Any: ...
    def require(self, key: str) -> Any: ...
    def as_dict(self) -> dict[str, Any]: ...
    
    def typed(self, key: str, type_: type[T]) -> T: ...

class SettingsBuilder:
    def from_config(
        self,
        config: PlatformConfiguration,
        namespace: str,
    ) -> PlatformSettings: ...
    
    def with_defaults(self, defaults: dict[str, Any]) -> "SettingsBuilder": ...
    def validated_by(self, schema: dict) -> "SettingsBuilder": ...
    def build(self) -> PlatformSettings: ...
```

---

## GROUP E — SCHEDULING

---

### E1. PlatformScheduler

**Purpose:** Central task and job scheduling coordinator. Manages
execution of PlatformTask instances across a managed thread pool.

**Responsibilities:**
- Schedule tasks for immediate, delayed, periodic, or event-triggered execution
- Enforce priority ordering via a priority queue
- Apply resource budgets (CPU, memory) per task
- Support cancellation, timeout, and retry with backoff
- Emit task lifecycle events
- Monitor for stuck or overdue tasks

**Public Interface:**
```python
class PlatformScheduler:
    def submit(
        self,
        task: "PlatformTask",
        *,
        delay_ms: int = 0,
        priority: int = 5,          # 0 = lowest, 10 = highest
    ) -> "TaskHandle": ...
    
    def schedule_periodic(
        self,
        task: "PlatformTask",
        interval_ms: int,
        *,
        initial_delay_ms: int = 0,
    ) -> "TaskHandle": ...
    
    def schedule_at(
        self,
        task: "PlatformTask",
        when: datetime,
    ) -> "TaskHandle": ...
    
    def cancel(self, handle: "TaskHandle") -> bool: ...
    
    def pause_all(self) -> None: ...
    def resume_all(self) -> None: ...
    
    @property
    def queue_depth(self) -> int: ...
    
    @property
    def active_count(self) -> int: ...
    
    def stats(self) -> "SchedulerStats": ...

class TaskHandle:
    task_id: str
    status: TaskStatus          # PENDING | RUNNING | DONE | FAILED | CANCELLED
    submitted_at: datetime
    started_at: datetime | None
    completed_at: datetime | None
    
    def cancel(self) -> bool: ...
    def wait(self, timeout_ms: int | None = None) -> "PlatformResult": ...
    async def wait_async(self) -> "PlatformResult": ...
    def result(self) -> "PlatformResult": ...
        # raises if not yet complete
```

**Configuration:**
- `kernel.scheduler.max_workers`: int (default: CPU count)
- `kernel.scheduler.max_queue_depth`: int (default: 10_000)
- `kernel.scheduler.default_timeout_ms`: int (default: 30_000)
- `kernel.scheduler.stuck_threshold_ms`: int (default: 60_000)

**Performance Target:** submit() < 0.5 ms; dispatch overhead < 1 ms

---

### E2. PlatformTask

**Purpose:** The unit of work submitted to PlatformScheduler.

```python
class PlatformTask(PlatformObject):
    @property
    def task_name(self) -> str: ...
    
    @property
    def timeout_ms(self) -> int | None: ...
    
    @property
    def max_retries(self) -> int: ...
    
    @property
    def retry_backoff_ms(self) -> int: ...
    
    @abstractmethod
    def execute(self, context: "PlatformContext") -> "PlatformResult": ...
    
    @abstractmethod
    async def execute_async(
        self, context: "PlatformContext"
    ) -> "PlatformResult": ...
    
    def on_success(self, result: "PlatformResult") -> None: ...
    def on_failure(self, error: "PlatformException") -> None: ...
    def on_cancel(self) -> None: ...
    def on_timeout(self) -> None: ...
    
    # Retry policy
    def should_retry(
        self, attempt: int, error: "PlatformException"
    ) -> bool: ...

class TaskBuilder:
    def name(self, name: str) -> "TaskBuilder": ...
    def timeout(self, ms: int) -> "TaskBuilder": ...
    def retries(self, max: int, backoff_ms: int = 1000) -> "TaskBuilder": ...
    def with_fn(
        self, fn: Callable[["PlatformContext"], Any]
    ) -> "TaskBuilder": ...
    def build(self) -> PlatformTask: ...
```

---

### E3. PlatformWorker

**Purpose:** Long-lived background worker that continuously processes
work from a queue or event stream. Unlike PlatformTask (one-shot),
workers run until explicitly stopped.

```python
class PlatformWorker(PlatformComponent):
    @property
    def worker_name(self) -> str: ...
    
    @property
    def is_busy(self) -> bool: ...
    
    @abstractmethod
    async def process(self, item: Any) -> None: ...
    
    def feed(self, item: Any) -> None: ...
        # put an item in the worker's queue
    
    @property
    def queue_depth(self) -> int: ...
    
    def stats(self) -> "WorkerStats": ...

class WorkerPool:
    def __init__(self, worker_factory: Callable[[], PlatformWorker],
                 size: int) -> None: ...
    
    def submit(self, item: Any) -> None: ...
    def resize(self, new_size: int) -> None: ...
    def drain(self, timeout_ms: int | None = None) -> None: ...
    def stats(self) -> list["WorkerStats"]: ...
```

---

## GROUP F — DIAGNOSTICS

---

### F1. PlatformMetrics

**Purpose:** Structured, high-performance metrics collection and aggregation.

**Metric Types:**
```python
class MetricType(enum.Enum):
    COUNTER   = "counter"    # monotonic increment
    GAUGE     = "gauge"      # current value (up/down)
    HISTOGRAM = "histogram"  # distribution with buckets
    SUMMARY   = "summary"    # pre-computed quantiles
    TIMER     = "timer"      # duration measurement

class PlatformMetrics:
    def counter(self, name: str, labels: dict | None = None) -> "Counter": ...
    def gauge(self, name: str, labels: dict | None = None) -> "Gauge": ...
    def histogram(
        self, name: str,
        buckets: list[float] | None = None,
        labels: dict | None = None,
    ) -> "Histogram": ...
    def timer(self, name: str, labels: dict | None = None) -> "Timer": ...
    
    def snapshot(self) -> list["MetricSample"]: ...
    def export_prometheus(self) -> str: ...
    def export_json(self) -> dict: ...
```

**Well-Known Kernel Metrics:**
```
kernel.objects.created.total          [counter]
kernel.objects.disposed.total         [counter]
kernel.objects.active                 [gauge]
kernel.events.published.total         [counter, label: event_type]
kernel.events.delivered.total         [counter]
kernel.events.dead_letter.total       [counter]
kernel.tasks.submitted.total          [counter]
kernel.tasks.completed.total          [counter]
kernel.tasks.failed.total             [counter]
kernel.tasks.duration_ms              [histogram]
kernel.plugins.loaded                 [gauge]
kernel.resources.allocated            [gauge, label: resource_type]
kernel.security.access.denied.total   [counter]
kernel.config.reload.total            [counter]
```

**Performance Target:** Metric emit < 0.01 ms; snapshot < 10 ms

---

### F2. PlatformHealth

**Purpose:** Structured health check system for all kernel components.

```python
class HealthStatus(enum.Enum):
    HEALTHY   = "healthy"
    DEGRADED  = "degraded"
    UNHEALTHY = "unhealthy"
    UNKNOWN   = "unknown"

@dataclass
class HealthCheckResult:
    component: str
    status: HealthStatus
    message: str
    checked_at: datetime
    duration_ms: float
    details: dict[str, Any]

class PlatformHealth:
    def register_check(
        self,
        name: str,
        check: Callable[[], HealthCheckResult],
        interval_ms: int = 30_000,
    ) -> None: ...
    
    def check(self, name: str) -> HealthCheckResult: ...
    def check_all(self) -> dict[str, HealthCheckResult]: ...
    
    @property
    def overall_status(self) -> HealthStatus: ...
    
    def on_status_change(
        self,
        callback: Callable[[str, HealthStatus, HealthStatus], None],
    ) -> "SubscriptionHandle": ...
    
    def export_json(self) -> dict: ...
```

**Built-in Health Checks:**
- `kernel.eventbus` — queue depth, dead letter count
- `kernel.scheduler` — queue depth, stuck task count
- `kernel.registry` — registered count, expired count
- `kernel.resources` — allocation levels per resource type
- `kernel.plugins` — loaded count, failed count

---

### F3. PlatformDiagnostics

**Purpose:** Unified diagnostic interface combining metrics, health,
tracing, and debug information. Single entry point for observability.

```python
class PlatformDiagnostics:
    @property
    def metrics(self) -> PlatformMetrics: ...
    
    @property
    def health(self) -> PlatformHealth: ...
    
    @property
    def logger(self) -> "PlatformLogger": ...
    
    @property
    def audit(self) -> "PlatformAudit": ...
    
    def trace(
        self,
        operation: str,
        correlation_id: str | None = None,
    ) -> "TraceSpan": ...
    
    def diagnostic_report(self) -> "DiagnosticReport": ...
        # full snapshot of all diagnostic state
    
    def register_debug_hook(
        self,
        name: str,
        hook: Callable[[], dict],
    ) -> None: ...
        # custom debug info attached to diagnostic report

class TraceSpan:
    span_id: str
    trace_id: str
    operation: str
    started_at: datetime
    
    def set_attribute(self, key: str, value: Any) -> None: ...
    def add_event(self, name: str, attributes: dict | None = None) -> None: ...
    def set_status(self, status: str) -> None: ...
    
    def __enter__(self) -> "TraceSpan": ...
    def __exit__(self, *args) -> None: ...
    async def __aenter__(self) -> "TraceSpan": ...
    async def __aexit__(self, *args) -> None: ...
```

---

### F4. PlatformAudit

**Purpose:** Immutable, append-only audit trail for all significant
kernel operations. Required for compliance and forensic analysis.

```python
@dataclass(frozen=True)
class AuditRecord:
    audit_id: str             # UUID v7
    timestamp: datetime       # UTC, microsecond
    actor_id: str             # who performed the action
    action: str               # e.g. "kernel.plugin.loaded"
    target_id: str | None     # affected object_id
    target_type: str | None
    outcome: str              # "success" | "failure" | "denied"
    details: dict[str, Any]
    correlation_id: str
    session_id: str | None

class PlatformAudit:
    def record(self, record: AuditRecord) -> None: ...
    
    def query(
        self,
        *,
        actor_id: str | None = None,
        action_pattern: str | None = None,
        target_id: str | None = None,
        since: datetime | None = None,
        until: datetime | None = None,
        outcome: str | None = None,
        limit: int = 1000,
    ) -> list[AuditRecord]: ...
    
    def export_jsonl(self, since: datetime | None = None) -> str: ...
    
    @property
    def record_count(self) -> int: ...
```

**Audit Retention:** Configurable (default: 90 days).
**Performance Target:** record() < 0.05 ms (async write to buffer)

---

## GROUP G — LOGGING

---

### G1. PlatformLogger

**Purpose:** Structured, levelled logging with context propagation,
correlation ID attachment, and configurable sink routing.

```python
class LogLevel(enum.IntEnum):
    TRACE   = 0
    DEBUG   = 1
    INFO    = 2
    WARNING = 3
    ERROR   = 4
    CRITICAL = 5

class PlatformLogger:
    def __init__(self, name: str) -> None: ...
    
    def trace(self, message: str, **fields) -> None: ...
    def debug(self, message: str, **fields) -> None: ...
    def info(self, message: str, **fields) -> None: ...
    def warning(self, message: str, **fields) -> None: ...
    def error(self, message: str, exc: Exception | None = None, **fields) -> None: ...
    def critical(self, message: str, exc: Exception | None = None, **fields) -> None: ...
    
    def with_context(self, **context) -> "PlatformLogger": ...
        # returns child logger with added context fields
    
    def with_correlation(self, correlation_id: str) -> "PlatformLogger": ...
    
    def with_span(self, span: "TraceSpan") -> "PlatformLogger": ...

class LoggerFactory:
    @staticmethod
    def get(name: str) -> PlatformLogger: ...
    
    @staticmethod
    def configure(
        level: LogLevel = LogLevel.INFO,
        sinks: list["LogSink"] | None = None,
        format: str = "json",
    ) -> None: ...

class LogSink(ABC):
    @abstractmethod
    def emit(self, record: "LogRecord") -> None: ...
```

**Log Record Structure:**
```json
{
  "timestamp": "2026-06-29T13:00:00.000000Z",
  "level": "INFO",
  "logger": "kernel.scheduler",
  "message": "Task submitted",
  "correlation_id": "...",
  "span_id": "...",
  "task_id": "...",
  "priority": 5
}
```

**Performance Target:** Emit < 0.05 ms (async buffered sink)

---

## GROUP H — SECURITY

---

### H1. PlatformSecurity

**Purpose:** Unified security enforcement layer. All access to resources,
capabilities, and secrets flows through PlatformSecurity.

**Responsibilities:**
- Authenticate principals (PlatformIdentity)
- Authorize operations against PlatformPermission sets
- Manage capability grants and revocations
- Enforce sandbox boundaries between plugins
- Provide encrypted secrets access
- Audit all security events

**Public Interface:**
```python
class PlatformSecurity:
    # Authentication
    def authenticate(
        self, credentials: "Credentials"
    ) -> "AuthenticationResult": ...
    
    async def authenticate_async(
        self, credentials: "Credentials"
    ) -> "AuthenticationResult": ...
    
    # Authorization
    def authorize(
        self,
        principal_id: str,
        action: str,
        resource_id: str | None = None,
    ) -> bool: ...
    
    def assert_authorized(
        self, principal_id: str, action: str,
        resource_id: str | None = None
    ) -> None: ...
        # raises PlatformException if denied
    
    # Capabilities
    def grant_capability(
        self,
        principal_id: str,
        capability: PlatformCapability,
        granted_by: str,
        expires_at: datetime | None = None,
    ) -> "CapabilityGrant": ...
    
    def revoke_capability(
        self, principal_id: str, capability_id: str
    ) -> None: ...
    
    def has_capability(
        self, principal_id: str, capability_id: str
    ) -> bool: ...
    
    def capabilities(
        self, principal_id: str
    ) -> list["CapabilityGrant"]: ...
    
    # Secrets
    def get_secret(self, name: str, accessor_id: str) -> str: ...
    def set_secret(self, name: str, value: str, owner_id: str) -> None: ...
    def delete_secret(self, name: str, owner_id: str) -> None: ...
    
    # Sandbox
    def create_sandbox(
        self, plugin_id: str, capabilities: list[PlatformCapability]
    ) -> "PluginSandbox": ...

class PluginSandbox:
    plugin_id: str
    allowed_capabilities: frozenset[str]
    
    def check(self, action: str) -> bool: ...
    def enter(self) -> None: ...
    def exit(self) -> None: ...
    def __enter__(self) -> "PluginSandbox": ...
    def __exit__(self, *args) -> None: ...
```

**Security Policy — Deny by Default:**
- Every action is DENIED unless an explicit ALLOW rule exists
- Capability grants are checked at enforcement points, not trusted from callers
- Secrets are only accessible to principals with `kernel.secrets.{name}.read`
- Plugin code always runs inside a PluginSandbox

---

### H2. PlatformPermission

```python
@dataclass(frozen=True)
class PlatformPermission:
    permission_id: str         # e.g. "workspace.read"
    action: str                # "read" | "write" | "execute" | "admin"
    resource_type: str         # "workspace" | "session" | "plugin" | "*"
    resource_id: str | None    # specific resource or None for all
    conditions: dict | None    # optional ABAC conditions
    
    def matches(self, action: str, resource_id: str | None) -> bool: ...

class PermissionSet:
    def add(self, permission: PlatformPermission) -> None: ...
    def remove(self, permission_id: str) -> None: ...
    def check(self, action: str, resource_id: str | None = None) -> bool: ...
    def all(self) -> list[PlatformPermission]: ...
```

---

### H3. PlatformFeatureFlag

**Purpose:** Runtime feature toggle system. Controls access to new or
experimental features without code deployments.

```python
@dataclass
class PlatformFeatureFlag:
    flag_id: str
    name: str
    description: str
    enabled: bool
    rollout_pct: float          # 0.0 – 1.0
    allowed_principals: list[str] | None   # None = all
    expires_at: datetime | None

class FeatureFlagManager:
    def is_enabled(
        self, flag_id: str, principal_id: str | None = None
    ) -> bool: ...
    
    def enable(self, flag_id: str) -> None: ...
    def disable(self, flag_id: str) -> None: ...
    def set_rollout(self, flag_id: str, pct: float) -> None: ...
    def register(self, flag: PlatformFeatureFlag) -> None: ...
    def all(self) -> list[PlatformFeatureFlag]: ...
    
    def on_change(
        self,
        flag_id: str,
        callback: Callable[[bool], None],
    ) -> "SubscriptionHandle": ...
```

---

## GROUP I — CONTEXT & WORKSPACE

---

### I1. PlatformContext

**Purpose:** Ambient execution context. Carries correlation IDs,
principal identity, active session, resource budgets, and cancellation
token for a single unit of work.

```python
class PlatformContext(PlatformEntity):
    @property
    def correlation_id(self) -> str: ...
    
    @property
    def principal_id(self) -> str: ...
    
    @property
    def session_id(self) -> str | None: ...
    
    @property
    def workspace_id(self) -> str | None: ...
    
    @property
    def cancellation(self) -> "CancellationToken": ...
    
    @property
    def deadline(self) -> datetime | None: ...
    
    @property
    def budget(self) -> "ResourceBudget": ...
    
    @property
    def logger(self) -> PlatformLogger: ...
        # pre-bound with correlation_id
    
    def is_cancelled(self) -> bool: ...
    def is_expired(self) -> bool: ...
    
    def child(
        self, 
        *,
        operation: str | None = None,
        timeout_ms: int | None = None,
    ) -> "PlatformContext": ...
        # creates a child context (inherits correlation_id, inherits budget)
    
    def fork(self) -> "PlatformContext": ...
        # creates an independent context (new correlation_id)

class CancellationToken:
    def cancel(self, reason: str = "") -> None: ...
    def is_cancelled(self) -> bool: ...
    def throw_if_cancelled(self) -> None: ...
    def on_cancel(self, callback: Callable) -> None: ...

class ResourceBudget:
    cpu_units: float            # abstract CPU units
    memory_mb: float
    ai_tokens: int
    network_calls: int
    storage_mb: float
    
    def consume(self, resource: str, amount: float) -> bool: ...
        # returns False if budget exhausted
    
    def remaining(self, resource: str) -> float: ...
```

---

### I2. PlatformWorkspace

**Purpose:** Top-level organizational unit. Every session, project, and
repository belongs to a workspace.

```python
class PlatformWorkspace(PlatformResource):
    @property
    def workspace_id(self) -> str: ...
    
    @property
    def name(self) -> str: ...
    
    @property
    def owner_principal(self) -> str: ...
    
    def projects(self) -> list["PlatformProject"]: ...
    def create_project(self, name: str) -> "PlatformProject": ...
    def open_project(self, project_id: str) -> "PlatformProject": ...
    def delete_project(self, project_id: str) -> None: ...
    
    @property
    def resource_quota(self) -> "ResourceQuota": ...
```

---

### I3. PlatformProject

**Purpose:** Named collection of repositories, sessions, and executions
within a workspace.

```python
class PlatformProject(PlatformResource):
    @property
    def project_id(self) -> str: ...
    
    @property
    def workspace_id(self) -> str: ...
    
    def repositories(self) -> list["PlatformRepository"]: ...
    def sessions(self) -> list["PlatformSession"]: ...
    def add_repository(self, path: str) -> "PlatformRepository": ...
    def create_session(self, context: PlatformContext) -> "PlatformSession": ...
```

---

### I4. PlatformRepository

**Purpose:** Represents a source code or knowledge repository registered
with the platform. Provides metadata about the repo's structure and status.

```python
class PlatformRepository(PlatformResource):
    @property
    def repository_id(self) -> str: ...
    
    @property
    def path(self) -> str: ...                 # local filesystem path
    
    @property
    def remote_url(self) -> str | None: ...
    
    @property
    def vcs(self) -> str: ...                  # "git" | "none"
    
    def metadata(self) -> dict[str, Any]: ...
        # language, size, file count, branch, commit
    
    def health(self) -> HealthCheckResult: ...
```

---

### I5. PlatformSession

**Purpose:** A bounded user or agent interaction session. Groups related
executions and carries session-scoped state.

```python
class PlatformSession(PlatformEntity):
    @property
    def session_id(self) -> str: ...
    
    @property
    def principal_id(self) -> str: ...
    
    @property
    def project_id(self) -> str: ...
    
    @property
    def started_at(self) -> datetime: ...
    
    @property
    def last_active(self) -> datetime: ...
    
    def executions(self) -> list["PlatformExecution"]: ...
    
    def create_execution(
        self, context: PlatformContext
    ) -> "PlatformExecution": ...
    
    def set_session_data(self, key: str, value: Any) -> None: ...
    def get_session_data(self, key: str) -> Any: ...
    
    def end(self) -> None: ...
```

---

## GROUP J — EXECUTION

---

### J1. PlatformExecution

**Purpose:** A single tracked unit of work (AI request, command, query)
with full observability and result capture.

```python
class PlatformExecution(PlatformEntity):
    @property
    def execution_id(self) -> str: ...
    
    @property
    def session_id(self) -> str: ...
    
    @property
    def operation(self) -> str: ...
    
    @property
    def context(self) -> PlatformContext: ...
    
    @property
    def status(self) -> ExecutionStatus: ...
    
    @property
    def result(self) -> "PlatformResult | None": ...
    
    @property
    def duration_ms(self) -> float | None: ...
    
    def run(self, command: "PlatformCommand") -> "PlatformResult": ...
    async def run_async(self, command: "PlatformCommand") -> "PlatformResult": ...
    
    def pipeline(self, pipeline: "PlatformPipeline") -> "PlatformResult": ...

class ExecutionStatus(enum.Enum):
    PENDING   = "pending"
    RUNNING   = "running"
    SUCCEEDED = "succeeded"
    FAILED    = "failed"
    CANCELLED = "cancelled"
    TIMED_OUT = "timed_out"
```

---

### J2. PlatformCommand

**Purpose:** Immutable value object representing an intent to change
system state. Commands are validated, authorized, and executed exactly once.

```python
@dataclass(frozen=True)
class PlatformCommand:
    command_id: str
    command_type: str           # e.g. "ai.generate", "workspace.create"
    payload: dict[str, Any]
    issued_by: str              # principal_id
    correlation_id: str
    issued_at: datetime
    idempotency_key: str | None # for at-most-once semantics

class CommandBus:
    def dispatch(
        self, command: PlatformCommand, context: PlatformContext
    ) -> "PlatformResult": ...
    
    async def dispatch_async(
        self, command: PlatformCommand, context: PlatformContext
    ) -> "PlatformResult": ...
    
    def register_handler(
        self,
        command_type: str,
        handler: Callable[["PlatformCommand", "PlatformContext"], "PlatformResult"],
    ) -> None: ...
```

---

### J3. PlatformQuery

**Purpose:** Immutable value object representing a read-only request
for information. Queries never change state.

```python
@dataclass(frozen=True)
class PlatformQuery:
    query_id: str
    query_type: str
    parameters: dict[str, Any]
    requested_by: str
    correlation_id: str
    requested_at: datetime
    max_results: int | None
    timeout_ms: int | None

class QueryBus:
    def execute(
        self, query: PlatformQuery, context: PlatformContext
    ) -> "PlatformResult": ...
    
    async def execute_async(
        self, query: PlatformQuery, context: PlatformContext
    ) -> "PlatformResult": ...
    
    def register_handler(
        self,
        query_type: str,
        handler: Callable[["PlatformQuery", "PlatformContext"], "PlatformResult"],
    ) -> None: ...
```

---

### J4. PlatformPipeline

**Purpose:** Ordered, composable chain of processing steps. Each step
receives the output of the previous step. Supports branching, filtering,
error recovery, and parallel execution.

```python
class PlatformPipeline(PlatformComponent):
    @property
    def pipeline_id(self) -> str: ...
    
    @property
    def steps(self) -> list["PipelineStep"]: ...
    
    def add_step(self, step: "PipelineStep") -> "PlatformPipeline": ...
    def add_filter(
        self,
        predicate: Callable[[Any], bool]
    ) -> "PlatformPipeline": ...
    def add_transform(
        self,
        fn: Callable[[Any], Any]
    ) -> "PlatformPipeline": ...
    def branch(
        self,
        condition: Callable[[Any], bool],
        if_true: "PlatformPipeline",
        if_false: "PlatformPipeline",
    ) -> "PlatformPipeline": ...
    
    def execute(
        self, input: Any, context: PlatformContext
    ) -> "PlatformResult": ...
    
    async def execute_async(
        self, input: Any, context: PlatformContext
    ) -> "PlatformResult": ...

class PipelineStep(ABC):
    @property
    def step_name(self) -> str: ...
    
    @abstractmethod
    def process(self, input: Any, context: PlatformContext) -> Any: ...
    
    @abstractmethod
    async def process_async(
        self, input: Any, context: PlatformContext
    ) -> Any: ...
```

---

## GROUP K — INFRASTRUCTURE

---

### K1. PlatformResource

**Purpose:** Represents a bounded, trackable system resource.
All resource allocation and deallocation flows through PlatformResource.

```python
class PlatformResource(PlatformEntity):
    @property
    def resource_type(self) -> "ResourceType": ...
    
    @property
    def allocated_amount(self) -> float: ...
    
    @property
    def max_amount(self) -> float: ...
    
    @property
    def utilization(self) -> float: ...  # 0.0 – 1.0
    
    def allocate(self, amount: float, requestor_id: str) -> bool: ...
    def release(self, amount: float, requestor_id: str) -> None: ...
    
    def is_exhausted(self) -> bool: ...
    def on_exhaustion(self, callback: Callable) -> None: ...

class ResourceType(enum.Enum):
    CPU         = "cpu"
    MEMORY      = "memory"
    NETWORK     = "network"
    AI_TOKENS   = "ai_tokens"
    STORAGE     = "storage"
    WORKSPACE   = "workspace"
    REPOSITORY  = "repository"
    PROJECT     = "project"
```

---

### K2. PlatformHook

**Purpose:** Named extension point that allows external code to intercept
and augment kernel operations without modifying kernel internals.

```python
@dataclass(frozen=True)
class PlatformHook:
    hook_id: str
    hook_name: str          # e.g. "kernel.before_plugin_load"
    description: str
    argument_types: tuple[str, ...]
    return_type: str | None

class HookRegistry:
    def define(self, hook: PlatformHook) -> None: ...
    
    def register(
        self,
        hook_name: str,
        handler: Callable,
        *,
        priority: int = 0,
        plugin_id: str | None = None,
    ) -> "HookHandle": ...
    
    def unregister(self, handle: "HookHandle") -> None: ...
    
    def trigger(self, hook_name: str, *args, **kwargs) -> list[Any]: ...
        # returns list of handler return values in priority order
    
    async def trigger_async(
        self, hook_name: str, *args, **kwargs
    ) -> list[Any]: ...
    
    def handlers(self, hook_name: str) -> int: ...
        # count of registered handlers
```

**Well-Known Kernel Hooks:**
```
kernel.before_plugin_load       args: (manifest: PlatformManifest)
kernel.after_plugin_load        args: (plugin: PlatformPlugin)
kernel.before_plugin_unload     args: (plugin: PlatformPlugin)
kernel.before_task_execute      args: (task: PlatformTask, context: PlatformContext)
kernel.after_task_execute       args: (task: PlatformTask, result: PlatformResult)
kernel.before_command_dispatch  args: (command: PlatformCommand)
kernel.after_command_dispatch   args: (command: PlatformCommand, result: PlatformResult)
kernel.before_security_check    args: (principal_id, action, resource_id)
```

---

### K3. PlatformExtension

**Purpose:** A lightweight, named capability that can be attached to any
PlatformObject to provide additional behavior without subclassing.

```python
class PlatformExtension(PlatformComponent):
    @property
    def extension_type(self) -> str: ...     # e.g. "serialization"
    
    @property
    def extends_type(self) -> str: ...       # object_type this extends

class ExtensionPoint(Generic[T]):
    """Type-safe extension point on a PlatformObject."""
    
    def __init__(self, name: str, interface: type[T]) -> None: ...
    
    def register(
        self, implementation: T, source_plugin: str | None = None
    ) -> None: ...
    
    def get(self) -> T: ...
        # raises if not registered
    
    def get_all(self) -> list[T]: ...
    
    def is_registered(self) -> bool: ...
```

---

### K4. PlatformException

**Purpose:** Structured, rich exception carrying kernel error codes,
context, correlation, and recovery hints.

```python
class PlatformException(Exception):
    def __init__(
        self,
        code: "ErrorCode",
        message: str,
        *,
        cause: Exception | None = None,
        context: dict | None = None,
        correlation_id: str | None = None,
        recoverable: bool = False,
        retry_after_ms: int | None = None,
    ) -> None: ...
    
    @property
    def code(self) -> "ErrorCode": ...
    
    @property
    def correlation_id(self) -> str | None: ...
    
    @property
    def is_recoverable(self) -> bool: ...
    
    @property
    def context(self) -> dict: ...
    
    def to_result(self) -> "PlatformResult": ...

class ErrorCode(str, enum.Enum):
    # Object errors
    OBJECT_DISPOSED        = "OBJECT_DISPOSED"
    OBJECT_NOT_FOUND       = "OBJECT_NOT_FOUND"
    INVALID_STATE          = "INVALID_STATE"
    INVALID_TRANSITION     = "INVALID_TRANSITION"
    # Plugin errors
    PLUGIN_NOT_FOUND       = "PLUGIN_NOT_FOUND"
    PLUGIN_LOAD_FAILED     = "PLUGIN_LOAD_FAILED"
    PLUGIN_CAPABILITY_DENIED = "PLUGIN_CAPABILITY_DENIED"
    PLUGIN_MANIFEST_INVALID = "PLUGIN_MANIFEST_INVALID"
    # Security errors
    ACCESS_DENIED          = "ACCESS_DENIED"
    AUTHENTICATION_FAILED  = "AUTHENTICATION_FAILED"
    CAPABILITY_MISSING     = "CAPABILITY_MISSING"
    SECRET_NOT_FOUND       = "SECRET_NOT_FOUND"
    # Resource errors
    RESOURCE_EXHAUSTED     = "RESOURCE_EXHAUSTED"
    BUDGET_EXCEEDED        = "BUDGET_EXCEEDED"
    QUOTA_EXCEEDED         = "QUOTA_EXCEEDED"
    # Execution errors
    TASK_TIMEOUT           = "TASK_TIMEOUT"
    TASK_CANCELLED         = "TASK_CANCELLED"
    PIPELINE_FAILED        = "PIPELINE_FAILED"
    # Configuration errors
    CONFIG_MISSING         = "CONFIG_MISSING"
    CONFIG_INVALID         = "CONFIG_INVALID"
    # Registry errors
    DUPLICATE_REGISTRATION = "DUPLICATE_REGISTRATION"
    NOT_REGISTERED         = "NOT_REGISTERED"
    # Generic
    INTERNAL_ERROR         = "INTERNAL_ERROR"
    NOT_IMPLEMENTED        = "NOT_IMPLEMENTED"
```

---

### K5. PlatformResult

**Purpose:** Typed result envelope wrapping success values or structured
errors. All kernel operations return PlatformResult instead of raising
exceptions for expected failure cases.

```python
@dataclass(frozen=True)
class PlatformResult(Generic[T]):
    value: T | None
    error: PlatformException | None
    metadata: dict[str, Any]
    correlation_id: str
    duration_ms: float
    timestamp: datetime
    
    @property
    def is_success(self) -> bool: ...
    
    @property
    def is_failure(self) -> bool: ...
    
    def unwrap(self) -> T: ...
        # returns value; raises PlatformException if error
    
    def unwrap_or(self, default: T) -> T: ...
    
    def map(self, fn: Callable[[T], U]) -> "PlatformResult[U]": ...
    
    def flat_map(
        self, fn: Callable[[T], "PlatformResult[U]"]
    ) -> "PlatformResult[U]": ...
    
    def on_success(self, fn: Callable[[T], None]) -> "PlatformResult[T]": ...
    def on_failure(self, fn: Callable[[PlatformException], None]) -> "PlatformResult[T]": ...
    
    @classmethod
    def ok(cls, value: T, **metadata) -> "PlatformResult[T]": ...
    
    @classmethod
    def fail(
        cls, error: PlatformException, **metadata
    ) -> "PlatformResult[None]": ...
    
    @classmethod
    def fail_with(
        cls, code: ErrorCode, message: str, **kwargs
    ) -> "PlatformResult[None]": ...
```
