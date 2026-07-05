# Platform Kernel — Event System
# Phase 2.0D.2.6A

---

## 1. Overview

The Platform Event System is the primary communication fabric between all
kernel components. It replaces direct method calls between unrelated
components with loosely coupled, ordered, correlated event flows.

Key properties:
- **In-process**: All events are in-process by default. No network IO.
- **Ordered**: Events from a single source are delivered in sequence order.
- **Correlated**: Every event carries a correlation ID tracing it to a flow.
- **Priority-aware**: CRITICAL events skip the queue and deliver synchronously.
- **Persistent**: Durable events survive restart and support replay.
- **Isolated**: A slow or failing subscriber never blocks other subscribers.

---

## 2. Core Types

### 2.1 PlatformEvent

Full specification in `03-MODULE-CATALOG.md § B1`.

Summary of key fields:
```
event_id        UUID v7 — unique, monotonic
event_type      dot-separated string — "kernel.lifecycle.started"
correlation_id  UUID — groups all events in one logical flow
causation_id    event_id of the event that triggered this one
trace_id        distributed trace ID (for cross-process correlation)
sequence        monotonic counter within each source
priority        CRITICAL > HIGH > NORMAL > LOW > BACKGROUND
persistent      bool — whether to write to durable log before dispatch
```

### 2.2 EventPriority Delivery Semantics

| Priority | Delivery Mode | Subscriber Isolation |
|----------|--------------|----------------------|
| CRITICAL | Synchronous, blocks publisher | None — all on publisher thread |
| HIGH | Async, dedicated dispatch thread | Full queue isolation |
| NORMAL | Async, shared pool | Full queue isolation |
| LOW | Async, shared pool, lower-priority queue | Full queue isolation |
| BACKGROUND | Async, idle cycles only | Full queue isolation |

---

## 3. Event Type Registry

All event types used in the kernel are registered at startup. Unknown event
types are accepted but logged as warnings.

### 3.1 Kernel System Events

```
kernel.object.created           object_id, object_type
kernel.object.disposed          object_id, object_type
kernel.object.meta.changed      object_id, key, old_value, new_value

kernel.lifecycle.initialized    object_id, object_type
kernel.lifecycle.started        object_id
kernel.lifecycle.paused         object_id
kernel.lifecycle.resumed        object_id
kernel.lifecycle.stopping       object_id
kernel.lifecycle.stopped        object_id
kernel.lifecycle.failed         object_id, reason
kernel.lifecycle.recovered      object_id
kernel.lifecycle.disposing      object_id
kernel.lifecycle.disposed       object_id

kernel.registry.registered      object_id, object_type, tags
kernel.registry.unregistered    object_id, object_type
kernel.registry.expired         object_id, ttl_seconds

kernel.plugin.loading           plugin_id, plugin_version
kernel.plugin.loaded            plugin_id, plugin_version
kernel.plugin.unloading         plugin_id
kernel.plugin.unloaded          plugin_id
kernel.plugin.failed            plugin_id, error_code, message
kernel.plugin.reloading         plugin_id

kernel.event.dead_letter        event_id, event_type, reason
kernel.event.routing.failed     event_id, error

kernel.task.submitted           task_id, priority
kernel.task.started             task_id
kernel.task.completed           task_id, duration_ms
kernel.task.failed              task_id, error_code
kernel.task.cancelled           task_id
kernel.task.timeout             task_id, timeout_ms

kernel.config.changed           key, old_value, new_value, source
kernel.config.reloaded          source_count

kernel.security.access.denied   principal_id, action, resource_id
kernel.security.capability.granted   principal_id, capability_id
kernel.security.capability.revoked   principal_id, capability_id

kernel.resource.allocated       resource_type, amount, requestor_id
kernel.resource.released        resource_type, amount, requestor_id
kernel.resource.exhausted       resource_type, utilization
kernel.resource.quota.exceeded  resource_type, quota, requested

kernel.health.status.changed    component, old_status, new_status
kernel.health.check.failed      component, message

kernel.performance.degraded     operation, target_ms, actual_ms
```

---

## 4. Event Correlation Model

### 4.1 Flow Anatomy

```
User action: "create workspace"
│
├── PlatformCommand{correlation_id: "flow-A", causation_id: null}
│
├── kernel.object.created{correlation_id: "flow-A", causation_id: "cmd-1"}
│     └── workspace_id: "ws-42"
│
├── kernel.lifecycle.initialized{correlation_id: "flow-A", causation_id: "evt-2"}
│
├── kernel.lifecycle.started{correlation_id: "flow-A", causation_id: "evt-3"}
│
└── kernel.registry.registered{correlation_id: "flow-A", causation_id: "evt-4"}
```

All events in a single user-initiated action share the same `correlation_id`.
The `causation_id` forms the causation chain: each event points to the
event (or command) that directly caused it.

### 4.2 Correlation Propagation Rules

1. A new `correlation_id` is generated at the boundary of each user action.
2. `PlatformContext.correlation_id` carries it through all code in that action.
3. When one event causes another, the new event copies the `correlation_id`
   and sets its `causation_id` to the causing event's `event_id`.
4. Async tasks inherit the `correlation_id` of the context that submitted them.
5. Plugin code is given a context with the active `correlation_id` pre-set.

---

## 5. Subscription Model

### 5.1 Pattern Syntax

Patterns use glob-style matching on `event_type`:

```
*                       all events
kernel.*                all kernel events
kernel.lifecycle.*      all lifecycle events
kernel.*.failed         any failure event in kernel namespace
kernel.task.completed   exact match
```

### 5.2 Subscription Lifecycle

```
subscribe(pattern, handler) → SubscriptionHandle
    │
    ├── SubscriptionHandle.pause()   — temporarily stop delivery
    ├── SubscriptionHandle.resume()  — resume delivery
    └── SubscriptionHandle.cancel()  — permanent unsubscribe
```

### 5.3 Delivery Guarantees

- **At-least-once** for persistent events (replay enabled)
- **At-most-once** for non-persistent events (best effort)
- **In-order** within a single source+pattern combination
- **No ordering** across different sources

### 5.4 Subscriber Error Handling

When a subscriber throws an exception during delivery:
1. The exception is caught and logged.
2. The event is retried (up to `max_retries` times with exponential backoff).
3. After all retries exhausted, the event is moved to the dead-letter queue.
4. A `kernel.event.dead_letter` event is emitted (with NORMAL priority).
5. The subscription remains active — the error does not cancel it.

---

## 6. Dead Letter Queue

### 6.1 When Events Go to Dead Letter

- Event published with no matching subscribers (when `require_subscriber=True`)
- Event delivery failed after all retries
- Event replay rejected by subscriber

### 6.2 Dead Letter Operations

```python
class DeadLetterQueue:
    def peek(self, limit: int = 100) -> list[PlatformEvent]: ...
    def requeue(self, event_id: str) -> None: ...   # retry delivery
    def discard(self, event_id: str) -> None: ...   # permanently drop
    def drain(self) -> list[PlatformEvent]: ...     # remove all; return them
    def count(self) -> int: ...
```

### 6.3 Dead Letter Monitoring

The health check `kernel.eventbus` is `DEGRADED` when dead letter count > 100.
The health check is `UNHEALTHY` when dead letter count > 1000.

---

## 7. Event Replay

### 7.1 Purpose

Replay allows a subscriber to receive past events when it first subscribes
or when it recovers from a failure. Only persistent events (`persistent=True`)
are stored in the replay log.

### 7.2 Replay API

```python
class PlatformEventBus:
    def replay(
        self,
        pattern: str,
        since: datetime | None = None,
        until: datetime | None = None,
        correlation_id: str | None = None,
        target_id: str | None = None,
    ) -> list[PlatformEvent]: ...
    
    def replay_to_handler(
        self,
        pattern: str,
        handler: Callable[[PlatformEvent], None],
        since: datetime | None = None,
        batch_size: int = 100,
    ) -> int: ...
        # returns count of events replayed
```

### 7.3 Replay Ordering

Replayed events are delivered in `(source, sequence)` order.
Within a correlation, events are replayed in causation-chain order.

### 7.4 Replay Window

Default replay window: 24 hours (configurable via
`kernel.eventbus.replay_window_hours`).

Events older than the window are not retained in the replay log.
The audit log (PlatformAudit) retains all persistent events for 90 days.

---

## 8. Event Bus Initialization

### 8.1 Boot Sequence

```
1. PlatformEventBus instantiated (empty — no subscribers, no log)
2. Kernel core components subscribe to required events
3. Replay log hydrated from PlatformAudit (last N hours)
4. "kernel.eventbus.ready" event emitted (CRITICAL priority — synchronous)
5. Plugins notified of bus availability
6. Plugin subscriptions accepted
```

### 8.2 Shutdown Sequence

```
1. New publishes accepted but queued — no new delivery
2. All in-flight deliveries complete or time out (30s max)
3. Remaining queued events flushed to dead letter log
4. "kernel.eventbus.stopping" event emitted (CRITICAL)
5. All subscriptions cancelled
6. Event bus disposed
```

---

## 9. Performance Design

### 9.1 Internal Queue Architecture

```
Publisher thread
    │
    ▼
[Priority Queue (heapq)]
    │
    ├── CRITICAL → delivered inline (bypasses queue)
    ├── HIGH     → dedicated dispatch thread
    ├── NORMAL   → thread pool worker
    ├── LOW      → thread pool worker (lower priority)
    └── BACKGROUND → single background thread (idle)
```

### 9.2 Lock-Free Hot Path

The publish path from producer to queue is lock-free using `collections.deque`
(GIL-safe for append/popleft). Priority arbitration uses a `threading.Event`
signal, not polling.

### 9.3 Per-Subscriber Queues

Each subscription has its own bounded queue (configurable `max_queue` depth,
default 1000). If a subscription's queue is full:
- LOW/BACKGROUND: oldest events are dropped (drop-head policy)
- NORMAL/HIGH: publisher is throttled for up to 10ms before drop
- CRITICAL: never queued — always inline

### 9.4 Performance Target

| Operation | Target |
|-----------|--------|
| publish() | < 0.1 ms (enqueue only) |
| End-to-end NORMAL delivery | < 1 ms (with idle bus) |
| Peak throughput | 50,000 events/sec |
| Subscription match | < 0.1 ms (regex cache) |
| Replay 1000 events | < 100 ms |
