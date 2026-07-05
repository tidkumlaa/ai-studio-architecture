# Platform Kernel — Lifecycle System
# Phase 2.0D.2.6A

---

## 1. Overview

Every PlatformObject that has lifecycle semantics (PlatformComponent,
PlatformPlugin, PlatformModule, PlatformSession, PlatformExecution, etc.)
progresses through a deterministic 10-state lifecycle managed by
PlatformLifecycle.

The lifecycle system provides:
- A finite state machine with explicit allowed transitions
- Audit logging on every state change
- Event emission on every state change
- Hook callbacks for transition observers
- Protection against illegal transitions (raises PlatformException)

---

## 2. The 10 States

```
┌───────────────┬──────────────────────────────────────────────────────────┐
│ State         │ Meaning                                                   │
├───────────────┼──────────────────────────────────────────────────────────┤
│ UNINITIALIZED │ Object constructed but initialize() not yet called        │
│ INITIALIZED   │ Configuration loaded; resources not yet acquired          │
│ STARTING      │ start() called; resources being acquired                  │
│ RUNNING       │ Fully operational; processing work                        │
│ PAUSING       │ pause() called; draining current work                     │
│ PAUSED        │ Work suspended; resources held                            │
│ STOPPING      │ stop() called; releasing resources                        │
│ STOPPED       │ Resources released; can be restarted or disposed          │
│ FAILED        │ Unrecoverable error; fail() called                        │
│ RECOVERING    │ recover() called; attempting to return to RUNNING         │
│ DISPOSING     │ dispose() called; releasing all references                │
│ DISPOSED      │ Terminal state; object is inert                           │
└───────────────┴──────────────────────────────────────────────────────────┘
```

Note: DISPOSED is terminal — no further transitions are possible.
FAILED is near-terminal — the only exit is RECOVERING or DISPOSING.

---

## 3. State Machine Diagram

```
                  ┌──────────────┐
                  │ UNINITIALIZED│
                  └──────┬───────┘
                         │ initialize()
                         ▼
                  ┌──────────────┐
                  │ INITIALIZED  │◄──────────────────────────┐
                  └──────┬───────┘                           │
                         │ start()                           │
                         ▼                                   │
                  ┌──────────────┐        stop()    ┌────────┴─────┐
                  │   STARTING   │──────────────────►│   STOPPING   │
                  └──────┬───────┘                  └──────┬───────┘
                         │ ready                           │ done
                         ▼                                 ▼
         pause()  ┌──────────────┐        stop()    ┌──────────────┐
         ┌────────│   RUNNING    │──────────────────►│   STOPPED    │
         │        └──────┬───────┘                  └──────┬───────┘
         ▼               │ fail()                          │ start()
  ┌──────────────┐       │                                 │
  │   PAUSING    │       ▼                                 │
  └──────┬───────┘ ┌──────────────┐                       │
         │ drained │    FAILED    │                        │
         ▼         └──────┬───────┘                       │
  ┌──────────────┐        │ recover()                      │
  │    PAUSED    │        ▼                                │
  └──────┬───────┘ ┌──────────────┐                       │
         │ resume()│  RECOVERING  │──► RUNNING (if ok)     │
         └────────►└──────────────┘──► FAILED (if fail)   │
                                                           │
         (any non-DISPOSED state) ──► DISPOSING ──► DISPOSED
```

---

## 4. Allowed Transitions Table

| From \ To        | INIT | START | RUNNING | PAUSING | PAUSED | STOPPING | STOPPED | FAILED | RECOVERING | DISPOSING | DISPOSED |
|------------------|------|-------|---------|---------|--------|----------|---------|--------|------------|-----------|----------|
| UNINITIALIZED    |  ✓   |       |         |         |        |          |         |        |            |    ✓      |          |
| INITIALIZED      |      |  ✓    |         |         |        |          |         |        |            |    ✓      |          |
| STARTING         |      |       |   ✓     |         |        |    ✓     |         |   ✓    |            |    ✓      |          |
| RUNNING          |      |       |         |   ✓     |        |    ✓     |         |   ✓    |            |    ✓      |          |
| PAUSING          |      |       |         |         |   ✓    |    ✓     |         |   ✓    |            |    ✓      |          |
| PAUSED           |      |       |   ✓     |         |        |    ✓     |         |   ✓    |            |    ✓      |          |
| STOPPING         |      |       |         |         |        |          |   ✓     |   ✓    |            |    ✓      |          |
| STOPPED          |      |  ✓    |         |         |        |          |         |        |            |    ✓      |          |
| FAILED           |      |       |         |         |        |          |         |        |    ✓       |    ✓      |          |
| RECOVERING       |      |       |   ✓     |         |        |          |         |   ✓    |            |    ✓      |          |
| DISPOSING        |      |       |         |         |        |          |         |        |            |           |    ✓     |
| DISPOSED         |      |       |         |         |        |          |         |        |            |           |          |

A blank cell means the transition is illegal and raises PlatformException(INVALID_TRANSITION).

---

## 5. Trigger Methods

```python
class PlatformLifecycle:
    def initialize(self) -> None:
        """UNINITIALIZED → INITIALIZED"""
    
    def start(self) -> None:
        """INITIALIZED | STOPPED → STARTING → (auto) RUNNING"""
    
    def pause(self) -> None:
        """RUNNING → PAUSING → (auto when drained) PAUSED"""
    
    def resume(self) -> None:
        """PAUSED → RUNNING"""
    
    def stop(self) -> None:
        """RUNNING | STARTING | PAUSING | PAUSED → STOPPING → (auto) STOPPED"""
    
    def fail(self, reason: str) -> None:
        """Any non-terminal, non-UNINITIALIZED state → FAILED"""
    
    def recover(self) -> None:
        """FAILED → RECOVERING → (if ok) RUNNING | (if fail) FAILED"""
    
    def dispose(self) -> None:
        """Any non-DISPOSED state → DISPOSING → DISPOSED
           Idempotent: calling on DISPOSED is a no-op."""
```

### 5.1 Automatic Transitions

Some transitions happen automatically when a condition is met:

| Trigger Method | Intermediate State | Auto-completion Condition | Final State |
|---------------|--------------------|--------------------------|-------------|
| start() | STARTING | Component signals ready | RUNNING |
| stop() | STOPPING | Component releases resources | STOPPED |
| pause() | PAUSING | Component drains current work | PAUSED |
| recover() | RECOVERING | Component initializes successfully | RUNNING |
| dispose() | DISPOSING | Component releases all references | DISPOSED |

Components signal readiness by calling `lifecycle._complete_transition()`.
This is an internal API — public callers only call the trigger methods.

---

## 6. Transition Hooks

Observers can register callbacks that fire on specific transitions.
Hooks fire before the state change becomes visible to other threads.

```python
# Register a hook
handle = lifecycle.on_transition(
    from_state=LifecycleState.STARTING,
    to_state=LifecycleState.RUNNING,
    callback=lambda: logger.info("Component became RUNNING"),
)

# Register a hook that fires on any transition FROM a state
handle = lifecycle.on_leave(LifecycleState.RUNNING, callback=my_fn)

# Register a hook that fires on any transition TO a state
handle = lifecycle.on_enter(LifecycleState.FAILED, callback=my_alert_fn)

# Cancel the hook
handle.cancel()
```

### 6.1 Hook Execution Order

1. Pre-transition hooks (`on_leave` from the current state)
2. State change applied (atomic, RLock held)
3. Post-transition hooks (`on_enter` to the new state)
4. PlatformAudit record written
5. PlatformEvent emitted (via PlatformEventBus)

Hooks may not trigger further state transitions on the same lifecycle object.
Doing so raises PlatformException(INVALID_TRANSITION).

---

## 7. Lifecycle Events

Every state transition emits a PlatformEvent with:
- `event_type`: `kernel.lifecycle.{state_name_lower}` 
- `payload.object_id`: the lifecycle owner's object_id
- `payload.object_type`: the lifecycle owner's type
- `payload.from_state`: previous state name
- `payload.to_state`: new state name
- `payload.duration_ms`: time spent in the previous state
- `priority`: HIGH for FAILED/DISPOSING/DISPOSED; NORMAL for others

---

## 8. Lifecycle Registry

The kernel maintains a global map of all active lifecycles.
This supports:
- Global health queries ("how many objects are in FAILED state?")
- Forced shutdown (dispose all objects in dependency order)
- Diagnostic inspection

```python
class LifecycleRegistry:
    def register(self, lifecycle: PlatformLifecycle) -> None: ...
    def unregister(self, object_id: str) -> None: ...
    
    def by_state(self, state: LifecycleState) -> list[PlatformLifecycle]: ...
    def count_by_state(self) -> dict[LifecycleState, int]: ...
    
    def dispose_all(
        self,
        in_dependency_order: bool = True,
        timeout_ms: int = 30_000,
    ) -> list[str]: ...
        # returns list of object_ids that failed to dispose
    
    def snapshot(self) -> list[dict]: ...
        # full state snapshot for diagnostics
```

---

## 9. Lifecycle Error Handling

### 9.1 Exception During Transition

If a component's setup code raises an exception during start():
- The lifecycle automatically transitions to FAILED
- The exception is wrapped in PlatformException(INTERNAL_ERROR)
- The exception is logged and emitted as `kernel.lifecycle.failed`
- The component is NOT disposed automatically — recovery is attempted first

### 9.2 FAILED State Semantics

In FAILED state:
- The object is not accessible to callers (PlatformRegistry removes it)
- PlatformHealth reports this component as UNHEALTHY
- A `kernel.health.status.changed` event is emitted
- Recovery can be attempted (kernel-level policy configures retry count)
- After `max_recovery_attempts` failures, the object is disposed

### 9.3 Recovery Policy

```python
@dataclass
class RecoveryPolicy:
    max_attempts: int = 3
    backoff_ms: list[int] = field(default_factory=lambda: [1000, 5000, 30000])
    dispose_on_exhaustion: bool = True
```

The kernel applies recovery policies to all components automatically
based on their configuration. Components that need custom recovery
override `on_recover()` in their implementation.

---

## 10. Lifecycle and Dependency Order

When a module or workspace is started, its components are started in
topological dependency order. When stopped, they are stopped in reverse order.

```
Dependency graph (start order):
  PlatformConfiguration → PlatformEventBus → PlatformRegistry → plugins...

Stop order (reverse):
  plugins... → PlatformRegistry → PlatformEventBus → PlatformConfiguration
```

The LifecycleOrchestrator handles this ordering using Kahn's topological sort
(identical algorithm to the SDK dependency resolver).

```python
class LifecycleOrchestrator:
    def start_in_order(
        self,
        components: list[PlatformComponent],
        dependency_fn: Callable[[str], list[str]],
        timeout_ms: int = 30_000,
    ) -> list[str]: ...
        # returns list of failed component_ids
    
    def stop_in_reverse_order(
        self,
        components: list[PlatformComponent],
        timeout_ms: int = 30_000,
    ) -> list[str]: ...
```
