# Platform Kernel — Resource Model
# Phase 2.0D.2.6A

---

## 1. Overview

The Platform Kernel Resource Model tracks, allocates, and enforces limits
on every significant resource consumed by the platform. Resources are not
managed at the OS level by the kernel — they are accounted for through
explicit allocate/release calls at the application level.

Resource accounting enables:
- Per-workspace and per-session budget enforcement
- Fair sharing across concurrent sessions
- Observability (who consumed what)
- Graceful degradation when resources are constrained

---

## 2. Resource Taxonomy

| Resource Type | Unit | Tracking Scope | Description |
|---------------|------|----------------|-------------|
| CPU | cpu_units (abstract) | per-task | Relative compute units |
| Memory | MB | per-session | Heap memory in use |
| Network | call_count | per-session | Outbound API calls |
| AI Tokens (input) | token count | per-session | LLM input tokens consumed |
| AI Tokens (output) | token count | per-session | LLM output tokens generated |
| Storage (read) | MB | per-session | Bytes read from storage |
| Storage (write) | MB | per-session | Bytes written to storage |
| Workspace slots | count | per-principal | Max simultaneous workspaces |
| Project slots | count | per-workspace | Max projects per workspace |
| Repository slots | count | per-project | Max repos per project |
| Session slots | count | per-project | Max concurrent sessions |
| Execution slots | count | per-session | Max concurrent executions |

---

## 3. ResourceBudget

ResourceBudget is carried in PlatformContext and consumed as work proceeds.

```python
@dataclass
class ResourceBudget:
    # Allocations
    cpu_units: float = 100.0
    memory_mb: float = 512.0
    network_calls: int = 100
    ai_tokens_input: int = 100_000
    ai_tokens_output: int = 50_000
    storage_read_mb: float = 500.0
    storage_write_mb: float = 100.0
    
    # Consumed (tracked internally)
    _consumed: dict[str, float] = field(default_factory=dict)
    
    def consume(self, resource: str, amount: float) -> bool:
        """Returns False if budget would be exceeded. Does NOT consume if False."""
    
    def consume_or_raise(self, resource: str, amount: float) -> None:
        """Consumes or raises PlatformException(BUDGET_EXCEEDED)."""
    
    def remaining(self, resource: str) -> float: ...
    def consumed(self, resource: str) -> float: ...
    def utilization(self, resource: str) -> float: ...   # 0.0 – 1.0
    def is_exhausted(self, resource: str) -> bool: ...
    def snapshot(self) -> dict[str, float]: ...

def create_budget(**overrides) -> ResourceBudget:
    """Factory applying defaults and per-workspace quota limits."""
```

### 3.1 Budget Propagation

When a child context is created:
```python
child_ctx = parent_ctx.child(operation="subtask")
```

The child shares the parent's budget. All consumption in the child
counts against the parent's budget. There is no per-child sub-budget —
all children of a context pool from the same ResourceBudget.

### 3.2 Budget Forking

For independent contexts that should not share budget:
```python
forked_ctx = parent_ctx.fork()
# forked_ctx gets its own fresh budget based on workspace quotas
```

---

## 4. ResourceQuota

ResourceQuota defines the maximum resources a workspace or principal
is permitted to use. Quotas are set by the platform administrator.

```python
@dataclass
class ResourceQuota:
    quota_id: str
    principal_id: str       # principal or workspace this applies to
    scope: str              # "workspace" | "project" | "session" | "principal"
    
    # Per-session limits (applied to each ResourceBudget created)
    max_cpu_units: float = 100.0
    max_memory_mb: float = 512.0
    max_network_calls: int = 100
    max_ai_tokens_input: int = 100_000
    max_ai_tokens_output: int = 50_000
    max_storage_read_mb: float = 500.0
    max_storage_write_mb: float = 100.0
    
    # Slot limits (applied globally, not per-session)
    max_workspaces: int = 10
    max_projects_per_workspace: int = 50
    max_repos_per_project: int = 20
    max_sessions_per_project: int = 5
    max_concurrent_executions: int = 3
    
    # Time window for rate limits
    rate_window_ms: int = 60_000     # 1 minute rolling window
    max_requests_per_window: int = 100

class QuotaRegistry:
    def set(self, quota: ResourceQuota) -> None: ...
    def get(self, principal_id: str) -> ResourceQuota: ...
    def effective(self, principal_id: str) -> ResourceQuota: ...
        # Merges principal quota with workspace quota (most restrictive wins)
```

---

## 5. ResourceManager

The central resource accounting service.

```python
class ResourceManager:
    def allocate(
        self,
        resource_type: ResourceType,
        amount: float,
        requestor_id: str,
        context: PlatformContext,
    ) -> "AllocationHandle": ...
    
    def release(self, handle: "AllocationHandle") -> None: ...
    
    def current_usage(
        self,
        resource_type: ResourceType,
        scope_id: str | None = None,
    ) -> float: ...
    
    def usage_by_requestor(
        self, resource_type: ResourceType
    ) -> dict[str, float]: ...
    
    def top_consumers(
        self,
        resource_type: ResourceType,
        limit: int = 10,
    ) -> list[tuple[str, float]]: ...
    
    def check_quota(
        self,
        principal_id: str,
        resource_type: ResourceType,
        requested: float,
    ) -> bool: ...

@dataclass
class AllocationHandle:
    allocation_id: str
    resource_type: ResourceType
    amount: float
    requestor_id: str
    allocated_at: datetime
    
    def __enter__(self) -> "AllocationHandle": ...
    def __exit__(self, *args) -> None: ...
        # auto-releases on context exit
```

---

## 6. AI Token Budget

AI Token consumption is a first-class resource concern. The kernel tracks
input and output tokens separately.

### 6.1 Token Recording

```python
class TokenBudget:
    def record_input(self, tokens: int, model: str) -> None: ...
    def record_output(self, tokens: int, model: str) -> None: ...
    
    def remaining_input(self) -> int: ...
    def remaining_output(self) -> int: ...
    
    @property
    def total_input_consumed(self) -> int: ...
    @property
    def total_output_consumed(self) -> int: ...
    
    def by_model(self) -> dict[str, tuple[int, int]]: ...
        # model → (input_tokens, output_tokens)
```

### 6.2 Token Events

Every AI invocation emits:
```
kernel.ai.tokens.consumed
  payload:
    model: str
    input_tokens: int
    output_tokens: int
    session_id: str
    execution_id: str
    remaining_input: int
    remaining_output: int
```

When tokens are exhausted:
```
kernel.resource.exhausted
  payload:
    resource_type: "ai_tokens_input" | "ai_tokens_output"
    session_id: str
    consumed: int
    budget: int
```

The AI Runtime must check `context.budget.is_exhausted("ai_tokens_input")`
before each LLM call and raise BUDGET_EXCEEDED if exhausted.

---

## 7. Storage Resource

Storage resources cover file system and knowledge-base access.

```python
class StorageResource(PlatformResource):
    def read(
        self,
        path: str,
        requestor_id: str,
        budget: ResourceBudget,
    ) -> bytes: ...
        # Records bytes read against budget; raises BUDGET_EXCEEDED if exceeded
    
    def write(
        self,
        path: str,
        data: bytes,
        requestor_id: str,
        budget: ResourceBudget,
    ) -> None: ...
    
    def size(self, path: str) -> float: ...   # MB
```

Note: PlatformResource subclasses do not perform raw file I/O.
They delegate to platform_sdk file utilities. The resource layer
only applies budget accounting and quota enforcement.

---

## 8. Workspace and Project Slots

Slot resources are counted rather than metered:

```python
class SlotManager:
    def acquire_workspace_slot(self, principal_id: str) -> "SlotHandle": ...
    def acquire_project_slot(self, workspace_id: str) -> "SlotHandle": ...
    def acquire_session_slot(self, project_id: str) -> "SlotHandle": ...
    def acquire_execution_slot(self, session_id: str) -> "SlotHandle": ...
    
    def slot_usage(
        self,
        scope_type: str,
        scope_id: str,
    ) -> tuple[int, int]: ...
        # returns (current, max)
    
    def wait_for_slot(
        self,
        scope_type: str,
        scope_id: str,
        timeout_ms: int = 30_000,
    ) -> "SlotHandle": ...
        # blocks until a slot is available or timeout

@dataclass
class SlotHandle:
    slot_id: str
    scope_type: str
    scope_id: str
    acquired_at: datetime
    
    def release(self) -> None: ...
    def __enter__(self) -> "SlotHandle": ...
    def __exit__(self, *args) -> None: ...
```

---

## 9. Resource Events and Metrics

### 9.1 Events

```
kernel.resource.allocated       resource_type, amount, requestor_id, context_id
kernel.resource.released        resource_type, amount, requestor_id, duration_ms
kernel.resource.exhausted       resource_type, utilization, scope_id
kernel.resource.quota.exceeded  resource_type, quota, requested, principal_id
kernel.resource.slot.acquired   scope_type, scope_id, current, max
kernel.resource.slot.released   scope_type, scope_id, current, max
kernel.resource.slot.wait       scope_type, scope_id, wait_ms
```

### 9.2 Metrics

```
kernel.resources.cpu.utilization        [gauge]
kernel.resources.memory.mb              [gauge]
kernel.resources.network.calls.total    [counter]
kernel.resources.ai.tokens.input.total  [counter]
kernel.resources.ai.tokens.output.total [counter]
kernel.resources.storage.read.mb.total  [counter]
kernel.resources.storage.write.mb.total [counter]
kernel.resources.slots.workspace        [gauge, label: principal_id]
kernel.resources.slots.session          [gauge, label: project_id]
```

---

## 10. Resource Model Invariants

1. Every allocation has a corresponding release (enforced via AllocationHandle context).
2. No allocation is accepted that would push total usage beyond the quota.
3. Token consumption is always recorded before the AI call completes.
4. Slot counts never go negative (double-release is a no-op with a warning).
5. ResourceBudget consumption is atomic — a failed consume() has no side effects.
