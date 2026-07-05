# Platform Kernel — Scheduling
# Phase 2.0D.2.6A

---

## 1. Overview

The scheduling subsystem manages all asynchronous and background work.
It provides a unified abstraction over Python's threading and asyncio
primitives, enforcing resource budgets, priority ordering, cancellation,
timeout, and retry.

Key design decisions:
- **Priority queue**: All work items carry an integer priority (0–10).
  Higher numbers run first.
- **Unified API**: A single PlatformScheduler handles both threading
  and asyncio work. Callers do not manage pools directly.
- **Cancellation**: All work is cancellable via CancellationToken.
- **Timeout**: All work has a configurable deadline.
- **Retry**: Retry with configurable backoff is built-in, not caller concern.

---

## 2. Scheduling Tiers

```
Tier        Priority    Thread Model         Typical Work
────────────────────────────────────────────────────────────────────────
CRITICAL    10          Inline (no queue)    Kernel invariant enforcement
HIGH        8–9         Dedicated threads    Event dispatch, health checks
NORMAL      5–7         Thread pool          Plugin calls, task execution
LOW         2–4         Thread pool          Background sync, indexing
IDLE        0–1         Single thread        Housekeeping, GC hints, cleanup
```

Work is never promoted to a higher tier automatically.
Work may be demoted by the resource governor if budgets are exceeded.

---

## 3. Task Scheduling

### 3.1 Immediate Submission

```python
scheduler = kernel.scheduler

handle = scheduler.submit(
    TaskBuilder()
    .name("analyze-repository")
    .with_fn(lambda ctx: analyze(ctx.workspace_id))
    .timeout(30_000)
    .retries(max=2, backoff_ms=1000)
    .build(),
    priority=7,
)

# Synchronous wait (blocks caller)
result = handle.wait(timeout_ms=35_000)

# Async wait (non-blocking)
result = await handle.wait_async()
```

### 3.2 Delayed Submission

```python
handle = scheduler.submit(
    my_task,
    delay_ms=5_000,   # execute 5 seconds from now
    priority=5,
)
```

### 3.3 Periodic Scheduling

```python
handle = scheduler.schedule_periodic(
    cleanup_task,
    interval_ms=60_000,    # every 60 seconds
    initial_delay_ms=0,
)

# Stop the periodic task
handle.cancel()
```

Periodic tasks:
- Next execution is scheduled only after the previous one completes
- If an execution exceeds `interval_ms`, the next fires immediately
- If an execution fails, the periodic schedule continues (failure is logged)
- `max_retries` applies per-execution, not globally

### 3.4 Scheduled-At Submission

```python
from datetime import datetime, timezone, timedelta

handle = scheduler.schedule_at(
    my_task,
    when=datetime.now(timezone.utc) + timedelta(hours=1),
)
```

---

## 4. Cancellation Model

### 4.1 CancellationToken

```python
class CancellationToken:
    def cancel(self, reason: str = "") -> None: ...
    def is_cancelled(self) -> bool: ...
    def throw_if_cancelled(self) -> None: ...
        # raises PlatformException(TASK_CANCELLED) if cancelled
    def on_cancel(self, callback: Callable) -> None: ...
```

CancellationTokens are linked: cancelling a parent cancels all children.

```python
parent_ctx = kernel.create_context()
child_ctx = parent_ctx.child(operation="subtask")

# Cancel the parent
parent_ctx.cancellation.cancel("user requested stop")

# child_ctx.is_cancelled() is now True
```

### 4.2 Cooperative Cancellation

Tasks must cooperate with cancellation by polling at safe points:

```python
class MyTask(PlatformTask):
    def execute(self, context: PlatformContext) -> PlatformResult:
        for item in self._items:
            context.cancellation.throw_if_cancelled()  # check at each item
            process(item)
        return PlatformResult.ok(None)
```

The scheduler does not forcibly terminate non-cooperative tasks.
After `cancel_timeout_ms` (default: 5000ms), a stuck task is logged
as a health warning but continues running.

### 4.3 TaskHandle.cancel()

```python
handle = scheduler.submit(my_task)
cancelled = handle.cancel()
# True  → task was pending or cooperative-cancelled
# False → task already completed before cancel reached it
```

---

## 5. Timeout Model

### 5.1 Task-Level Timeout

```python
TaskBuilder()
.timeout(30_000)       # ms — task must complete within 30 seconds
.build()
```

When a task exceeds its timeout:
1. CancellationToken is cancelled (reason: "timeout")
2. Task's `on_timeout()` callback is called
3. TaskHandle status → TIMED_OUT
4. `kernel.task.timeout` event emitted
5. PlatformResult.fail_with(TASK_TIMEOUT, ...) returned

### 5.2 Context-Level Deadline

A PlatformContext can carry an absolute deadline:
```python
ctx = kernel.create_context(deadline=datetime.now() + timedelta(seconds=30))
```

Tasks that receive this context automatically inherit the deadline.
If the context is expired when the task checks, it cancels immediately.

---

## 6. Retry Model

### 6.1 Retry Configuration

```python
TaskBuilder()
.retries(max=3, backoff_ms=1000)  # 3 retries, 1s/2s/4s backoff (exponential)
.build()
```

Default backoff formula: `min(base_ms * (2 ** attempt), max_backoff_ms)`
where `max_backoff_ms` defaults to 30,000ms.

### 6.2 Custom Retry Logic

```python
class MyTask(PlatformTask):
    def should_retry(
        self, attempt: int, error: PlatformException
    ) -> bool:
        # Only retry transient errors, not permanent failures
        return error.code in (ErrorCode.RESOURCE_EXHAUSTED, ErrorCode.TASK_TIMEOUT)
```

### 6.3 Retry Tracking

```python
handle.status          # PENDING | RUNNING | DONE | FAILED | CANCELLED
handle.attempt_count   # current retry attempt number (starts at 1)
handle.last_error      # PlatformException from last failed attempt
```

---

## 7. Worker Pool Model

### 7.1 Background Workers

Workers are for continuous stream processing (unlike PlatformTask which
is one-shot):

```python
class IndexingWorker(PlatformWorker):
    async def process(self, item: FileChangeEvent) -> None:
        await self._index.update(item.path, item.content)

pool = WorkerPool(
    worker_factory=lambda: IndexingWorker(indexer=my_indexer),
    size=4,
)
pool.submit(file_event)    # dispatched to next available worker
```

### 7.2 Pool Autoscaling

The scheduler can autoscale worker pools based on queue depth:

```python
pool.configure_autoscale(
    min_size=2,
    max_size=16,
    scale_up_threshold=100,    # queue depth to trigger scale-up
    scale_down_delay_ms=30_000,
)
```

### 7.3 Worker Health

Each worker reports its health to PlatformHealth:
- HEALTHY: processing normally, queue depth < scale_up_threshold
- DEGRADED: queue depth > scale_up_threshold
- UNHEALTHY: worker has been stuck on a single item for > stuck_threshold_ms

---

## 8. Resource Budget Enforcement

### 8.1 Budget Integration

Every PlatformContext carries a ResourceBudget. The scheduler checks
the budget before dispatching a task:

```python
# Before dispatch
if not context.budget.consume("cpu_units", task.estimated_cpu):
    return PlatformResult.fail_with(BUDGET_EXCEEDED, "CPU budget exhausted")
```

### 8.2 CPU Throttling

If a task's thread consumes more CPU time than budgeted, the scheduler:
1. Emits `kernel.performance.degraded`
2. Reduces the task's thread priority
3. Applies a cooperative yield hint (`time.sleep(0)`) at check points

The scheduler never forcibly kills a task for CPU overuse — it degrades
priority and logs. Hard limits are enforced at the OS/container level.

---

## 9. Scheduler Diagnostics

```python
class SchedulerStats:
    queue_depth: int
    active_tasks: int
    completed_total: int
    failed_total: int
    cancelled_total: int
    timeout_total: int
    avg_wait_ms: float
    avg_duration_ms: float
    p95_duration_ms: float
    p99_duration_ms: float
    workers_by_tier: dict[str, int]
    
scheduler.stats()       # SchedulerStats snapshot
scheduler.queue_depth   # current pending task count
scheduler.active_count  # currently executing tasks
```

Scheduler metrics emitted to PlatformMetrics:
- `kernel.tasks.submitted.total` [counter]
- `kernel.tasks.completed.total` [counter]
- `kernel.tasks.failed.total` [counter]
- `kernel.tasks.duration_ms` [histogram]
- `kernel.tasks.wait_ms` [histogram]
- `kernel.scheduler.queue_depth` [gauge]
- `kernel.scheduler.active_workers` [gauge]

---

## 10. Scheduler Configuration

```yaml
kernel:
  scheduler:
    max_workers: 16               # total thread pool size
    max_queue_depth: 10000        # max pending tasks before backpressure
    default_timeout_ms: 30000     # default task timeout
    stuck_threshold_ms: 60000     # task running longer than this → warning
    cancel_timeout_ms: 5000       # grace period for cooperative cancel
    idle_thread_keepalive_ms: 60000
    
    tiers:
      critical:
        threads: 0      # inline — no dedicated threads
      high:
        threads: 2
      normal:
        threads: 8
      low:
        threads: 4
      idle:
        threads: 1
```
