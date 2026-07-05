---
knowledge_id: KA-SDK-011
version: "1.0.0"
status: approved
owner: Chief Platform Architect
phase: 2.0D.2.5A
created: 2026-06-29
review_date: 2026-12-29
canonical: true
domain: DOM-PLATFORM
capability: platform-sdk
type: specification
depends_on:
  - id: KA-SDK-002
    reason: "Task queue collections"
  - id: KA-SDK-010
    reason: "Optimization SDK for schedule optimization"
  - id: KA-SDK-016
    reason: "Topological ordering for dependency scheduling"
---

# Scheduling SDK

## Priority · Deadline · Topology · Rate Limiting · Work Stealing · Backpressure

---

## 1. Purpose

The Scheduling SDK manages the lifecycle of executable tasks across all AI
Studio Runtimes. It provides six scheduling strategies as pluggable primitives,
separating task definition from execution policy. The Platform Kernel Scheduler
is built on this SDK. All scheduling is single-threaded by default; concurrency
is a deployment concern above this layer.

---

## 2. Scheduling Taxonomy

```
Scheduler
├── PriorityScheduler      — tasks ordered by priority score
├── DeadlineScheduler      — earliest-deadline-first (EDF)
├── TopologicalScheduler   — respects dependency edges (DAG-based)
├── RateLimitedScheduler   — token bucket / leaky bucket
├── WorkStealingScheduler  — multi-queue load balancing
└── CompositeScheduler     — chains multiple strategies
```

---

## 3. Interfaces

### 3.1 Task

```python
@dataclass
class Task(Generic[T]):
    task_id:      str
    name:         str
    fn:           Callable[[], T]      # The unit of work
    priority:     float                # Lower = higher urgency
    deadline:     float | None         # Unix timestamp
    dependencies: Sequence[str]        # task_ids that must complete first
    timeout_ms:   int | None
    retry_limit:  int = 0
    tags:         frozenset[str] = frozenset()
    metadata:     ImmutableMap[str, Any] = field(default_factory=dict)

@dataclass
class TaskResult(Generic[T]):
    task_id:    str
    status:     str    # "success" | "failure" | "timeout" | "cancelled"
    value:      T | None
    error:      Exception | None
    started_at: float
    ended_at:   float
    elapsed_ms: float
    attempt:    int
```

### 3.2 Scheduler (base)

```python
class Scheduler(Protocol[T]):
    def submit(self, task: Task[T]) -> str     # Returns task_id; O(log N)
    def cancel(self, task_id: str) -> bool
    def next(self) -> Task[T] | None           # O(1) or O(log N)
    def peek(self) -> Task[T] | None           # O(1) — does not dequeue
    def size(self) -> int
    def is_empty(self) -> bool
    def clear(self) -> None
    def stats(self) -> SchedulerStats
```

### 3.3 PriorityScheduler[T]

```python
class PriorityScheduler(Scheduler[T]):
    """Min-heap by priority score."""
    # submit: O(log N)
    # next:   O(log N)
    # peek:   O(1)
    def update_priority(self, task_id: str, new_priority: float) -> None  # O(log N)
    def boost_priority(self, tag: str, delta: float) -> int   # Returns count boosted
```

### 3.4 DeadlineScheduler[T]

```python
class DeadlineScheduler(Scheduler[T]):
    """Earliest Deadline First (EDF). Tasks without deadline use priority."""
    # submit: O(log N)
    # next:   O(log N) — always returns task with nearest deadline
    def overdue(self) -> Sequence[Task[T]]   # Tasks past their deadline
    def time_to_deadline(self, task_id: str) -> float | None  # Seconds remaining
```

### 3.5 TopologicalScheduler[T]

```python
class TopologicalScheduler(Scheduler[T]):
    """Respects task dependency edges. Tasks ready when all deps complete."""
    # Uses Kahn's algorithm incrementally
    # submit: O(D) D = dependencies
    # next:   O(1) from ready queue
    def add_dependency(self, task_id: str, dep_id: str) -> None
    def mark_complete(self, task_id: str) -> Sequence[Task[T]]  # Returns newly-ready tasks
    def ready_count(self) -> int
    def blocked_count(self) -> int
    def has_cycle(self) -> bool
```

### 3.6 RateLimitedScheduler[T]

```python
class RateLimitedScheduler(Scheduler[T]):
    """Token bucket rate limiter. Controls submission and execution rate."""
    def configure(self, rate: float, burst: int) -> None
        # rate: tokens per second, burst: max token bucket size
    def available_tokens(self) -> float
    def wait_time_ms(self) -> float    # Estimated wait before next token available

class LeakyBucketScheduler(RateLimitedScheduler[T]):
    """Fixed output rate regardless of input burst."""
    def queue_depth(self) -> int
    def drop_policy(self) -> str  # "tail_drop" | "head_drop" | "random"
```

### 3.7 WorkStealingScheduler[T]

```python
class WorkStealingScheduler(Scheduler[T]):
    """Multiple queues; idle worker steals from tail of busiest queue."""
    def add_queue(self, queue_id: str, affinity_tags: frozenset[str]) -> None
    def remove_queue(self, queue_id: str) -> None
    def queue_sizes(self) -> ImmutableMap[str, int]
    def steal_count(self) -> int    # Total steal operations performed
```

### 3.8 SchedulerStats

```python
@dataclass(frozen=True)
class SchedulerStats:
    submitted:    int
    completed:    int
    failed:       int
    cancelled:    int
    timed_out:    int
    avg_wait_ms:  float
    avg_exec_ms:  float
    queue_depth:  int
    throughput:   float   # tasks per second
```

### 3.9 Backpressure

```python
class BackpressurePolicy(Protocol):
    """Applied when queue exceeds high-water mark."""
    def on_full(self, scheduler: Scheduler,
                incoming: Task) -> "BackpressureAction"

class BackpressureAction(Enum):
    REJECT    = "reject"
    BLOCK     = "block"     # Caller waits until space available
    DROP_TAIL = "drop_tail" # Drop newest task
    DROP_HEAD = "drop_head" # Drop oldest task

class BackpressureConfig:
    high_watermark: int
    low_watermark:  int
    policy:         BackpressurePolicy
```

---

## 4. Contracts

| ID | Contract |
|----|----------|
| C-SCH-001 | `next()` returns `None` on empty scheduler (does not block). |
| C-SCH-002 | `cancel` on already-running task returns `False`. |
| C-SCH-003 | `TopologicalScheduler.next()` never returns a task with incomplete dependencies. |
| C-SCH-004 | `TopologicalScheduler.has_cycle() == True` prevents any task from being scheduled. |
| C-SCH-005 | `RateLimitedScheduler` never returns more tasks per second than `rate` (steady state). |
| C-SCH-006 | `DeadlineScheduler`: among tasks with equal priority, nearest-deadline is returned first. |
| C-SCH-007 | `WorkStealingScheduler` steal: at most ⌊queue_size/2⌋ tasks stolen at once. |
| C-SCH-008 | Backpressure `BLOCK` must be implemented without busy-waiting. |
| C-SCH-009 | `SchedulerStats.throughput` is computed over a rolling window, not total lifetime. |
| C-SCH-010 | Task with `retry_limit=N` may be re-submitted up to N times on failure. |

---

## 5. Complexity Table

| Operation | Scheduler | Complexity |
|-----------|---------|------------|
| submit | Priority | O(log N) |
| next | Priority | O(log N) |
| submit | Topology | O(D) |
| next | Topology (ready) | O(1) |
| mark_complete | Topology | O(D_out) |
| submit | RateLimited | O(1) |
| next | RateLimited | O(1) with token check |
| steal | WorkStealing | O(K/2) K=victim queue size |

---

## 6. Dependencies

- Collections SDK (KA-SDK-002) — PriorityQueue for heap-based schedulers
- Optimization SDK (KA-SDK-010) — schedule cost modeling
- Traversal SDK (KA-SDK-016) — topological ordering of dependency DAG

---

## 7. Extension Points

```python
class SchedulingPolicy(Protocol[T]):
    """Custom task ordering beyond built-in strategies."""
    def compare(self, a: Task[T], b: Task[T]) -> int: ...  # negative, 0, positive

class TaskLifecycleListener(Protocol[T]):
    def on_submit(self, task: Task[T]) -> None: ...
    def on_start(self, task: Task[T]) -> None: ...
    def on_complete(self, task: Task[T], result: TaskResult[T]) -> None: ...
    def on_fail(self, task: Task[T], error: Exception) -> None: ...
    def on_cancel(self, task: Task[T]) -> None: ...

class SchedulerPlugin(Protocol[T]):
    """Contributes a new scheduling strategy."""
    def strategy_name(self) -> str: ...
    def create_scheduler(self, config: dict) -> Scheduler[T]: ...
```

---

## 8. Verification

| Check | Criterion |
|-------|-----------|
| V-SCH-001 | `next()` on empty scheduler returns `None` |
| V-SCH-002 | Priority: lower-priority-number task returned before higher |
| V-SCH-003 | Deadline: nearest-deadline task returned first |
| V-SCH-004 | Topology: `next()` never returns task with pending dependencies |
| V-SCH-005 | Topology: `has_cycle()` detected after creating circular dep |
| V-SCH-006 | RateLimited: N tasks in 1s does not exceed configured rate |
| V-SCH-007 | `cancel` before `next`: task not returned by subsequent `next` |
| V-SCH-008 | `BackpressurePolicy.REJECT`: submit on full queue fails immediately |
| V-SCH-009 | `SchedulerStats.completed + failed + cancelled = submitted - queue_depth` |
| V-SCH-010 | Retry: failed task re-queued up to `retry_limit` times |
