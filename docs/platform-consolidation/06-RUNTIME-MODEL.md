---
knowledge_id: KNW-PLAT-ARCH-006
title: "Platform Runtime Model"
status: AUTHORITATIVE
phase: "2.1D.0"
version: "1.0.0"
created: "2026-06-29"
purpose: "Define the 7 runtimes, their contracts, boot order, isolation boundaries, and capabilities"
canonical_source: "architecture/docs/platform-consolidation/06-RUNTIME-MODEL.md"
dependencies:
  - "02-OBJECT-MODEL.md"
  - "07-KERNEL-MODEL.md"
related_documents:
  - "05-MODULE-CATALOG.md"
  - "10-DEPENDENCY-RULES.md"
  - "18-EVENT-MODEL.md"
  - "32-RUNTIME-INTEGRATION.md"
acceptance_criteria:
  - "Every runtime has a defined boot order"
  - "Every runtime has a complete isolation specification"
  - "Cross-runtime communication uses only kernel events"
  - "Every runtime has health check specification"
verification_checklist:
  - "[ ] Boot order is total (no ties)"
  - "[ ] No direct runtime-to-runtime imports anywhere"
  - "[ ] Health check interface defined"
  - "[ ] Runtime metadata schema matches 13-METADATA-STANDARD.md"
future_extensions:
  - "Runtime hot-reload support"
  - "Runtime sandboxing (separate process per runtime)"
---

# Platform Runtime Model

## Runtime Taxonomy

```
PlatformKernel (boot: 0)
    └── event_runtime      (boot: 1) — must be up before any runtime publishes events
    └── resource_runtime   (boot: 2) — quota/budget available early
    └── provider_runtime   (boot: 3) — providers available before AI runtime
    └── knowledge_runtime  (boot: 4) — knowledge index ready
    └── ai_runtime         (boot: 5) — depends on provider + resource + knowledge
    └── workflow_runtime   (boot: 6) — depends on ai_runtime
    └── orchestration_runtime (boot: 7) — depends on workflow + ai
```

**Shutdown order is reverse of boot order.**

---

## Runtime Contract Interface

Every runtime implements this Python Protocol:

```python
class PlatformRuntime(Protocol):
    runtime_id: str              # "runtime:ai:v2"
    name: str                    # "ai_runtime"
    version: str                 # semver
    boot_order: int              # 1–7
    capabilities: list[str]      # capability IDs provided

    async def start(self, kernel: PlatformKernel) -> None:
        """Boot the runtime. Kernel is available. Register capabilities."""

    async def stop(self) -> None:
        """Graceful shutdown. Deregister capabilities."""

    async def health(self) -> RuntimeHealth:
        """Return current health status."""

    async def pause(self) -> None:
        """Pause processing. Do not shut down."""

    async def resume(self) -> None:
        """Resume after pause."""
```

---

## Runtime 1: event_runtime

**Purpose:** Event bus infrastructure. All other runtimes depend on it.

| Property | Value |
|----------|-------|
| ID | `runtime:event:v1` |
| Boot order | 1 |
| Dependencies | Kernel only |
| Capabilities | `event.publish`, `event.subscribe`, `event.stream` |
| Isolation | Runs in-process; optionally backed by Redis/AMQP |

**Key interfaces:**
```python
async def publish(event: PlatformEvent) -> None
async def subscribe(topic: str, handler: AsyncCallable) -> Subscription
async def stream(topic: str) -> AsyncIterator[PlatformEvent]
```

**Cross-runtime rule:** All other runtimes call `event_runtime` via kernel injection only.

---

## Runtime 2: resource_runtime

**Purpose:** Quota enforcement, budget management, resource scheduling.

| Property | Value |
|----------|-------|
| ID | `runtime:resource:v1` |
| Boot order | 2 |
| Dependencies | Kernel, event_runtime |
| Capabilities | `resource.check_quota`, `resource.spend`, `resource.schedule` |
| Isolation | In-process state; SQLite persistence |

**Modules (consolidated from ai_runtime):**
- `quota/` — per-org/project/model quota tracking
- `budget/` — API spend budgets  
- `scheduler/` — job scheduling and priority queues

**Events published:** `QuotaExceeded`, `BudgetAlert`, `ResourceAllocated`

---

## Runtime 3: provider_runtime

**Purpose:** Provider plugin registry, health monitoring, adapter management.

| Property | Value |
|----------|-------|
| ID | `runtime:provider:v1` |
| Boot order | 3 |
| Dependencies | Kernel, event_runtime |
| Capabilities | `provider.register`, `provider.get`, `provider.health` |
| Isolation | Plugin discovery via entry_points; no provider imports in core |

**Plugin contract:**
```python
class AIProviderPlugin(Protocol):
    provider_id: str          # "anthropic", "openai", "google"
    models: list[str]
    async def complete(self, request: CompletionRequest) -> CompletionResponse: ...
    async def health(self) -> ProviderHealth: ...
```

**Events published:** `ProviderRegistered`, `ProviderUnhealthy`, `ProviderRecovered`

---

## Runtime 4: knowledge_runtime

**Purpose:** Knowledge graph, compilation, indexing, semantic search.

| Property | Value |
|----------|-------|
| ID | `runtime:knowledge:v1` |
| Boot order | 4 |
| Dependencies | Kernel, event_runtime |
| Capabilities | `knowledge.compile`, `knowledge.search`, `knowledge.link` |
| Isolation | In-process graph; optional vector store |

**Key interfaces:**
```python
async def compile(source: Path) -> KnowledgeObject
async def search(query: str, limit: int) -> list[KnowledgeResult]
async def link(source_id: str, target_id: str, relation: str) -> None
```

**Platform objects → Knowledge objects:**
Every `PlatformDocument`, `PlatformModule`, `PlatformCapability` compiles to a `KnowledgeObject`.

**Events published:** `KnowledgeCompiled`, `KnowledgeIndexUpdated`, `KnowledgeSearched`

---

## Runtime 5: ai_runtime

**Purpose:** AI execution, intelligent routing, resource intelligence. Contains all Phase 2.1A/2.1B modules.

| Property | Value |
|----------|-------|
| ID | `runtime:ai:v2` |
| Boot order | 5 |
| Dependencies | Kernel, event_runtime, resource_runtime, provider_runtime, knowledge_runtime |
| Capabilities | 160+ (see 15-CAPABILITY-REGISTRY.md) |
| Isolation | In-process; stateful (usage history in memory with eviction) |

**Sub-systems:**
- `resource_intelligence/` — 29 modules (Phase 2.1A + 2.1B, all implemented)
- `routing/` — provider selection and routing
- `execution/` — task execution management

**Events published:** `AIExecutionCompleted`, `RoutingDecisionMade`, `QuotaWarning`
**Events consumed:** `QuotaExceeded`, `ProviderUnhealthy`, `ResourceAllocated`

---

## Runtime 6: workflow_runtime

**Purpose:** Task dependency graphs, sequential/parallel execution, workflow scheduling.

| Property | Value |
|----------|-------|
| ID | `runtime:workflow:v1` |
| Boot order | 6 |
| Dependencies | Kernel, event_runtime, ai_runtime |
| Capabilities | `workflow.define`, `workflow.execute`, `workflow.status` |
| Isolation | DAG state in memory; persisted to SQLite |

**Key interfaces:**
```python
async def define(workflow: WorkflowDefinition) -> WorkflowID
async def execute(workflow_id: WorkflowID, inputs: dict) -> WorkflowRun
async def status(run_id: WorkflowRunID) -> WorkflowStatus
```

**Events published:** `WorkflowStarted`, `WorkflowCompleted`, `WorkflowFailed`, `TaskStarted`

---

## Runtime 7: orchestration_runtime

**Purpose:** Multi-agent coordination, session management, agent lifecycle.

| Property | Value |
|----------|-------|
| ID | `runtime:orchestration:v1` |
| Boot order | 7 |
| Dependencies | Kernel, event_runtime, ai_runtime, workflow_runtime |
| Capabilities | `orchestration.spawn`, `orchestration.coordinate`, `orchestration.terminate` |
| Isolation | Session state in memory |

**Key interfaces:**
```python
async def spawn(agent: AgentDefinition, session_id: str) -> AgentHandle
async def coordinate(agents: list[AgentHandle], task: str) -> CoordinationResult
async def terminate(agent_handle: AgentHandle) -> None
```

**Events published:** `AgentSpawned`, `AgentTerminated`, `CoordinationCompleted`

---

## Runtime Isolation Rules

| Rule | Description |
|------|-------------|
| No direct imports | Runtime A never imports `from platform.runtimes.runtime_b` |
| Event-only communication | Cross-runtime calls via `kernel.events.publish()` |
| No shared mutable state | Runtimes do not share in-memory objects |
| Failure isolation | One runtime failure must not cascade (kernel health monitors) |
| Independent start/stop | Each runtime can stop and restart without full platform shutdown |

---

## Health Check Protocol

```python
@dataclass
class RuntimeHealth:
    runtime_id: str
    state: Literal["READY", "DEGRADED", "FAILED", "PAUSED"]
    latency_ms: float
    error_count_last_60s: int
    active_connections: int
    message: str | None = None
```

Platform kernel polls all runtime health endpoints every 30 seconds.
A runtime in FAILED state triggers `PlatformDegraded` event.
Three consecutive FAILED reports trigger automatic runtime restart.
