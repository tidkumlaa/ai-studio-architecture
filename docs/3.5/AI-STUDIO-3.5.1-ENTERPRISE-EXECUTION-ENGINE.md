# AI Studio 3.5.1 — Enterprise Execution Engine

**Version:** 3.5.1-DRAFT  
**Date:** 2026-06-27  
**Status:** Engineering Specification  
**Extends:** AI-STUDIO-3.5-UNIVERSAL-PRODUCT-FACTORY.md  
**Invariant:** All capabilities from AI Studio v2.0 through v3.5 are preserved unchanged. This document specifies the runtime execution layer only.

---

## Chapter 1 — Runtime Architecture

### 1.1 Purpose

The Runtime Architecture is the execution substrate that makes every capability described in AI Studio 3.5 physically possible. AI Studio 3.5 specifies *what* the platform does. AI Studio 3.5.1 specifies *how* the platform executes: the processes, data flows, isolation boundaries, and coordination protocols that turn a workflow definition into a running product.

### 1.2 Execution Lifecycle Overview

```mermaid
flowchart TD
    A([User submits request]) --> B[Intake API]
    B --> C[Execution Context Created]
    C --> D[Workspace Provisioned]
    D --> E[Workflow Compiler]
    E --> F{Compilation valid?}
    F -- No --> G[ValidationError → User]
    F -- Yes --> H[Task Graph Built]
    H --> I[Task Scheduler]
    I --> J[Agent Runtime Pool]
    J --> K[Tool Runtime / Sandbox]
    K --> L[Artifact Store]
    L --> M[QA Gate]
    M --> N{Gate passes?}
    N -- No --> O[Retry / Compensation]
    O --> I
    N -- Yes --> P[Approval Gate if required]
    P --> Q[Deployment Engine]
    Q --> R[Execution Complete]
    R --> S[Context Archived]
    S --> T[Experience Graph Updated]
```

### 1.3 Execution Context

Every unit of work — from a single task to a full product factory — runs inside an **Execution Context**. The context is immutable after creation and carries all information needed to reproduce the execution exactly.

```python
@dataclass(frozen=True)
class ExecutionContext:
    # Identity
    execution_id:   UUID
    parent_id:      UUID | None         # parent context if spawned by workflow
    factory_id:     UUID | None
    project_id:     UUID
    company_id:     UUID | None

    # Versioning
    workflow_id:    UUID
    workflow_version: str               # semver: '2.4.1'
    engine_version: str                 # runtime version: '3.5.1'

    # Resource budget
    token_budget:   int                 # max tokens for this context
    cost_budget_usd: Decimal
    time_budget_s:  int                 # wall-clock timeout in seconds

    # Security
    principal_id:   str                 # user or agent identity
    tenant_id:      UUID
    permission_set: frozenset[str]

    # Routing
    model:          str
    provider:       str
    priority:       int                 # 0=lowest, 100=highest

    # Observability
    trace_id:       str                 # OpenTelemetry trace ID
    span_id:        str
    correlation_id: UUID

    # Timestamps
    created_at:     datetime
    deadline_at:    datetime            # created_at + time_budget_s
```

### 1.4 Workspace

Each execution context receives an isolated **Workspace** — an ephemeral, sandboxed virtual filesystem. The workspace contains cloned repository state, intermediate artifacts, and scratch space. It is destroyed after the execution completes unless the artifacts are committed.

```
Workspace layout:
  /workspace/{execution_id}/
  ├── repo/                   ← sparse checkout of project repository
  │   ├── src/
  │   ├── tests/
  │   └── ...
  ├── artifacts/              ← outputs produced by this execution
  │   ├── generated_code/
  │   ├── test_results/
  │   └── reports/
  ├── scratch/                ← agent working space (not persisted)
  ├── .context.json           ← serialised ExecutionContext
  └── .checkpoint/            ← checkpoint files for resume
```

**Workspace isolation mechanisms:**

| Mechanism | Technology | Purpose |
|-----------|-----------|---------|
| Filesystem | OverlayFS (Linux) / Junction (Windows) | Copy-on-write layer over repo snapshot |
| Network | Network namespace + egress proxy | Prevent unauthorized external calls |
| Process | gVisor (Linux) / Job Objects (Windows) | Kernel-level sandbox for tool execution |
| Memory | cgroup memory limit | Prevent runaway allocations |
| CPU | cgroup CPU quota | Fair scheduling per execution |

### 1.5 Artifact Store

Every file produced by an agent — code, test output, report, diagram — is an **Artifact**. Artifacts are immutable once written and content-addressable (SHA-256).

```
Artifact lifecycle:
  1. Agent writes file → Artifact Store intercepts write syscall
  2. Content hashed (SHA-256) → deduplicated if identical content exists
  3. Metadata recorded: producer agent, execution_id, timestamp, type
  4. File available to all agents in same execution context via read
  5. On execution success → artifacts promoted to Artifact Repository (MinIO)
  6. On execution failure → artifacts retained for 7 days then purged
```

**Artifact metadata schema:**

```sql
CREATE TABLE artifacts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    execution_id    UUID NOT NULL,
    task_id         UUID,
    artifact_type   TEXT NOT NULL,
    -- 'source_code'|'test_file'|'test_result'|'report'|'config'|
    -- 'migration'|'diagram'|'log'|'sbom'|'build_output'
    file_path       TEXT NOT NULL,          -- relative to workspace
    content_hash    TEXT NOT NULL,          -- SHA-256
    size_bytes      BIGINT,
    mime_type       TEXT,
    producer_agent  TEXT,
    storage_key     TEXT,                   -- MinIO object key after promotion
    is_promoted     BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    promoted_at     TIMESTAMPTZ
);
CREATE INDEX idx_artifacts_exec ON artifacts(execution_id, artifact_type);
```

### 1.6 Shared Context

Agents within the same workflow execution share a **Shared Context** — a structured key-value store that persists across stage boundaries. This replaces the need for agents to pass data via files or function return values.

```python
class SharedContext:
    """
    Thread-safe, append-only within an execution.
    Backed by Redis with write-through to PostgreSQL for durability.
    TTL: execution deadline + 1 hour (for post-mortem inspection).
    """
    def __init__(self, execution_id: UUID):
        self.execution_id = execution_id
        self._redis_key = f"ctx:{execution_id}"

    async def set(self, key: str, value: Any, produced_by: str) -> None:
        payload = {
            "value": value,
            "produced_by": produced_by,
            "set_at": datetime.utcnow().isoformat()
        }
        await self.redis.hset(self._redis_key, key, json.dumps(payload))
        await self.db.insert_context_entry(self.execution_id, key, payload)

    async def get(self, key: str) -> Any | None:
        raw = await self.redis.hget(self._redis_key, key)
        if raw:
            return json.loads(raw)["value"]
        return None

    async def require(self, key: str) -> Any:
        val = await self.get(key)
        if val is None:
            raise ContextKeyMissing(f"Required context key '{key}' not set")
        return val

    async def snapshot(self) -> dict:
        """Return full context snapshot — used for checkpoint and replay."""
        raw = await self.redis.hgetall(self._redis_key)
        return {k: json.loads(v) for k, v in raw.items()}
```

### 1.7 Execution Pipeline

The complete pipeline from request intake to completion:

```mermaid
sequenceDiagram
    participant U as User / Factory
    participant API as Intake API
    participant EC as ExecutionContext
    participant WC as WorkflowCompiler
    participant TS as TaskScheduler
    participant AR as AgentRuntime
    participant TR as ToolRuntime
    participant AS as ArtifactStore
    participant QG as QAGate
    participant EB as EventBus

    U->>API: POST /executions { workflow_id, input }
    API->>EC: create_context(request)
    EC->>API: execution_id
    API->>U: 202 Accepted { execution_id, stream_url }

    API->>WC: compile(workflow_id, context)
    WC->>TS: submit_task_graph(dag)

    loop For each ready task
        TS->>AR: dispatch(task, context)
        AR->>AR: initialize_agent(task.agent_type)
        AR->>TR: execute_tools(tool_calls)
        TR->>AS: write_artifact(file, content)
        AR->>EB: emit(task.completed)
        EB->>TS: advance_dag(completed_task_id)
    end

    TS->>QG: check_quality_gates()
    QG->>EB: emit(execution.completed or execution.failed)
    EB->>EC: archive_context()
```

### 1.8 Runtime Component Registry

| Component | Process | Port | Protocol | Replicas |
|-----------|---------|------|----------|---------|
| Intake API | `runtime-api` | 8090 | HTTP/2 + WebSocket | 2–N |
| Workflow Compiler | `wf-compiler` | internal | gRPC | 2 |
| Task Scheduler | `task-scheduler` | internal | gRPC | 2 (active/standby) |
| Agent Runtime Worker | `agent-worker` | internal | NATS pull | 4–N |
| Tool Runtime | `tool-runtime` | internal | gRPC | per worker |
| Event Bus | `nats` | 4222 | NATS JetStream | 3 (cluster) |
| State Store | `postgres` | 5432 | SQL | 1 primary + 2 replica |
| Context Cache | `redis` | 6379 | Redis | 3 (cluster) |
| Artifact Store | `minio` | 9000 | S3-compatible | 4 (erasure) |

---

## Chapter 2 — Workflow Engine

### 2.1 Purpose

The Workflow Engine compiles human-readable workflow definitions into executable task graphs, validates them against the AI Constitution and resource policies, and drives execution through the DAG — handling parallel stages, conditional branches, loops, nested sub-workflows, and compensation on failure.

### 2.2 Workflow DSL

Workflows are defined in YAML. The compiler transforms YAML into an internal Task Graph representation.

```yaml
# workflow: feature-full/2.4.1
id: feature-full
version: "2.4.1"
description: "End-to-end feature development workflow"
timeout_seconds: 86400          # 24h hard limit
retry_policy:
  max_attempts: 2
  backoff: exponential
  initial_delay_s: 30

input_schema:
  type: object
  required: [requirement_id, project_id]
  properties:
    requirement_id: {type: string, format: uuid}
    project_id:     {type: string, format: uuid}
    priority:       {type: integer, minimum: 0, maximum: 100, default: 50}

stages:
  - id: architecture
    agent: ArchitectAgent
    timeout_seconds: 3600
    approval_required: true
    approval_roles: [architect, cto]
    outputs:
      - adr_document
      - module_boundaries
      - database_changes_required
    on_success: [backend, frontend, database]   # fan-out to parallel
    on_failure: escalate

  - id: backend
    agent: BackendAgent
    depends_on: [architecture]
    parallel_group: implementation
    timeout_seconds: 7200
    inputs:
      adr_document: "{{stages.architecture.outputs.adr_document}}"
    outputs: [backend_code, api_spec]
    on_success: [quality]
    on_failure: retry_then_escalate

  - id: frontend
    agent: FrontendAgent
    depends_on: [architecture]
    parallel_group: implementation
    timeout_seconds: 7200
    inputs:
      api_spec: "{{stages.backend.outputs.api_spec}}"   # soft dep — await if not ready
    outputs: [frontend_code]
    on_success: [quality]

  - id: database
    agent: DatabaseAgent
    depends_on: [architecture]
    parallel_group: implementation
    condition: "{{stages.architecture.outputs.database_changes_required}} == true"
    timeout_seconds: 3600
    outputs: [migration_files]
    compensate:
      - rollback_migration
    on_success: [quality]

  - id: quality
    agent: QAAgent
    depends_on: [backend, frontend]     # waits for both
    depends_on_optional: [database]     # waits only if database stage ran
    parallel_group: qa
    timeout_seconds: 3600
    outputs: [test_results, coverage_report]
    loop:
      condition: "{{outputs.test_results.pass_rate}} < 0.80"
      max_iterations: 3
      on_max_reached: escalate

  - id: security_scan
    agent: SecurityAgent
    depends_on: [backend, frontend]
    parallel_group: qa
    timeout_seconds: 1800
    outputs: [scan_results]
    on_failure: block_release

  - id: release
    agent: ReleaseAgent
    depends_on: [quality, security_scan]
    approval_required: true
    approval_roles: [lead, cto]
    timeout_seconds: 1800
    outputs: [release_package, changelog]

output_schema:
  release_package:    {type: string, description: "Path to release artifact"}
  test_results:       {type: object}
  changelog:          {type: string}

on_complete: archive_and_learn
```

### 2.3 Workflow Compiler

The compiler validates the DSL and produces an immutable `CompiledWorkflow`.

```mermaid
flowchart LR
    A[YAML DSL] --> B[YAML Parser]
    B --> C{Schema valid?}
    C -- No --> Z[ValidationError]
    C -- Yes --> D[Stage Graph Builder]
    D --> E{DAG acyclic?}
    E -- No --> Z
    E -- Yes --> F[Input/Output Resolver]
    F --> G{All refs resolved?}
    G -- No --> Z
    G -- Yes --> H[Condition Compiler]
    H --> I[Constitution Checker]
    I --> J{Policies pass?}
    J -- No --> Z
    J -- Yes --> K[Resource Estimator]
    K --> L[CompiledWorkflow]
    L --> M[Stored in workflow_compiled table]
```

```python
class WorkflowCompiler:
    def compile(self, definition: WorkflowDefinition, context: ExecutionContext) -> CompiledWorkflow:
        # 1. Parse and schema-validate
        parsed = self._parse(definition)

        # 2. Build stage dependency graph
        graph = StageGraph()
        for stage in parsed.stages:
            graph.add_node(stage)
            for dep in stage.depends_on:
                graph.add_edge(dep, stage.id)

        # 3. Validate DAG (no cycles)
        if not graph.is_dag():
            raise CyclicDependencyError(graph.find_cycles())

        # 4. Resolve input/output references
        resolver = IOResolver(graph)
        resolved = resolver.resolve_all()

        # 5. Compile conditions to AST (Python AST, sandboxed eval)
        for stage in resolved.stages:
            if stage.condition:
                stage.compiled_condition = ConditionCompiler.compile(stage.condition)
            if stage.loop:
                stage.loop.compiled_condition = ConditionCompiler.compile(stage.loop.condition)

        # 6. Check against AI Constitution (Ch.10 of 2.0 Extension)
        violations = self.constitution.check_workflow(resolved)
        if violations:
            raise ConstitutionViolationError(violations)

        # 7. Estimate resources
        estimate = self.resource_estimator.estimate(resolved, context)

        return CompiledWorkflow(
            workflow_id=definition.id,
            version=definition.version,
            compiled_at=datetime.utcnow(),
            stage_graph=graph,
            resource_estimate=estimate,
            checksum=sha256(resolved.canonical_json())
        )
```

### 2.4 Task Graph

After compilation, the workflow is instantiated as a **Task Graph** — a live DAG of `TaskNode` objects, each tracking execution state.

```python
@dataclass
class TaskNode:
    node_id:        UUID
    stage_id:       str
    execution_id:   UUID
    agent_type:     str
    status:         TaskStatus          # see §6.2
    depends_on:     list[UUID]          # node_ids that must be COMPLETED first
    dependents:     list[UUID]          # node_ids unblocked when this completes
    parallel_group: str | None
    condition:      CompiledCondition | None
    loop_config:    LoopConfig | None
    loop_iteration: int = 0
    attempt:        int = 1
    assigned_to:    UUID | None         # employee_id
    scheduled_at:   datetime | None
    started_at:     datetime | None
    completed_at:   datetime | None
    outputs:        dict = field(default_factory=dict)
    error:          str | None = None

class TaskGraph:
    nodes: dict[UUID, TaskNode]
    edges: dict[UUID, list[UUID]]       # node_id → list of dependent node_ids

    def ready_nodes(self) -> list[TaskNode]:
        """Return all nodes whose dependencies are all COMPLETED."""
        return [
            n for n in self.nodes.values()
            if n.status == TaskStatus.PENDING
            and all(self.nodes[dep].status == TaskStatus.COMPLETED
                    for dep in n.depends_on)
            and self._condition_passes(n)
        ]

    def is_complete(self) -> bool:
        return all(n.status in (TaskStatus.COMPLETED, TaskStatus.SKIPPED)
                   for n in self.nodes.values())

    def failed_nodes(self) -> list[TaskNode]:
        return [n for n in self.nodes.values() if n.status == TaskStatus.FAILED]
```

### 2.5 DAG Execution Engine

```mermaid
flowchart TD
    A[CompiledWorkflow submitted] --> B[TaskGraph.instantiate]
    B --> C[Persist graph to DB]
    C --> D[Enter execution loop]
    D --> E[TaskGraph.ready_nodes]
    E --> F{Any ready?}
    F -- None, graph not done --> G[Await event: task_completed]
    G --> D
    F -- Yes --> H[For each ready node]
    H --> I{Has parallel_group?}
    I -- Yes --> J[Dispatch all in group concurrently]
    I -- No  --> K[Dispatch single node]
    J --> L[Await group completion]
    K --> M[Await node completion]
    L --> D
    M --> D
    F -- None, graph done --> N{Any failures?}
    N -- Yes --> O[Compensation Engine]
    N -- No  --> P[Execution Complete]
```

### 2.6 Parallel Execution

Nodes in the same `parallel_group` are dispatched concurrently. The group completes when all nodes in the group have reached a terminal state.

```python
class ParallelGroupExecutor:
    async def execute(self, group: str, nodes: list[TaskNode], ctx: ExecutionContext):
        tasks = [self.scheduler.dispatch(node, ctx) for node in nodes]
        results = await asyncio.gather(*tasks, return_exceptions=True)
        failed = [r for r in results if isinstance(r, Exception)]
        if failed and self._all_required(nodes):
            raise ParallelGroupFailure(group, failed)
        return results
```

### 2.7 Conditional Execution

Stage conditions are compiled to sandboxed Python AST expressions. The evaluator receives the SharedContext snapshot as the evaluation namespace.

```python
class ConditionEvaluator:
    ALLOWED_NODES = {ast.Expression, ast.Compare, ast.BoolOp, ast.UnaryOp,
                     ast.Name, ast.Constant, ast.Attribute, ast.Subscript,
                     ast.And, ast.Or, ast.Not, ast.Eq, ast.NotEq,
                     ast.Lt, ast.Gt, ast.LtE, ast.GtE, ast.In, ast.NotIn}

    def evaluate(self, condition: CompiledCondition, ctx_snapshot: dict) -> bool:
        tree = condition.ast_tree
        self._validate_nodes(tree)                  # whitelist only
        namespace = {"ctx": ctx_snapshot, "true": True, "false": False}
        return bool(eval(compile(tree, "<condition>", "eval"), namespace))
```

### 2.8 Loop Execution

Loops repeat a stage until a condition becomes false or max_iterations is reached.

```python
class LoopExecutor:
    async def execute(self, node: TaskNode, ctx: ExecutionContext) -> TaskResult:
        config = node.loop_config
        iteration = 0
        while True:
            result = await self.scheduler.dispatch_single(node, ctx)
            node.outputs.update(result.outputs)
            iteration += 1
            node.loop_iteration = iteration

            ctx_snap = await ctx.shared_context.snapshot()
            if not self.condition_eval.evaluate(config.compiled_condition, ctx_snap):
                return result                       # exit loop: condition now false

            if iteration >= config.max_iterations:
                if config.on_max_reached == "escalate":
                    await self.approval_gate.create(node, "max_loop_iterations_reached")
                    decision = await self.approval_gate.wait(node, timeout_s=3600)
                    if decision == "continue":
                        config.max_iterations += iteration   # extend
                        continue
                raise MaxLoopIterationsError(node.stage_id, iteration)

            await asyncio.sleep(config.backoff_s)
```

### 2.9 Nested Workflows

A stage can delegate to a complete sub-workflow. The sub-workflow runs in a child execution context that shares the parent's workspace and artifact store.

```yaml
  - id: database_migration_subprocess
    type: sub_workflow
    workflow: database-migration
    version: "1.3.0"
    inherit_context: true
    inputs:
      migration_files: "{{stages.database.outputs.migration_files}}"
    outputs: [migration_result]
```

```python
class SubWorkflowExecutor:
    async def execute(self, stage: SubWorkflowStage, parent_ctx: ExecutionContext) -> TaskResult:
        child_ctx = parent_ctx.derive_child(
            workflow_id=stage.workflow_id,
            token_budget=stage.token_budget or parent_ctx.remaining_token_budget(),
            workspace=parent_ctx.workspace     # shared workspace
        )
        result = await self.execution_engine.run(child_ctx, stage.inputs)
        return TaskResult(outputs=result.outputs, status=result.status)
```

### 2.10 Workflow Versioning

Every workflow definition is stored with a semantic version. Running executions are pinned to the version that was compiled at dispatch time — they are not affected by subsequent version updates.

```sql
CREATE TABLE workflow_versions (
    workflow_id     TEXT NOT NULL,
    version         TEXT NOT NULL,
    compiled_at     TIMESTAMPTZ NOT NULL,
    definition_yaml TEXT NOT NULL,
    compiled_json   JSONB NOT NULL,
    checksum        TEXT NOT NULL,
    is_active       BOOLEAN DEFAULT TRUE,
    created_by      UUID,
    PRIMARY KEY (workflow_id, version)
);
-- Running executions reference this table at dispatch time.
-- No updates to this table after creation (append-only).
```

---

## Chapter 3 — Agent Runtime

### 3.1 Purpose

The Agent Runtime is the execution environment for every agent in the system. It manages the complete agent lifecycle from initialization through termination, injects all dependencies, registers tool capabilities, maintains checkpoints, and enforces resource limits. Every agent runs inside an `AgentProcess` — a lightweight, isolated unit of execution.

### 3.2 Agent Lifecycle

```mermaid
stateDiagram-v2
    [*] --> INITIALIZING : Scheduler dispatches task
    INITIALIZING --> READY : Context + tools loaded
    INITIALIZING --> FAILED : Dependency injection error
    READY --> RUNNING : execute() called
    RUNNING --> CHECKPOINTED : Checkpoint interval fires
    CHECKPOINTED --> RUNNING : Resume from checkpoint
    RUNNING --> SUSPENDED : Human approval gate raised
    SUSPENDED --> RUNNING : Gate approved
    SUSPENDED --> CANCELLED : Gate rejected / timeout
    RUNNING --> COMPLETED : execute() returns success
    RUNNING --> FAILED : Unhandled exception or budget exceeded
    FAILED --> RUNNING : Retry policy permits (new attempt)
    COMPLETED --> [*]
    CANCELLED --> [*]
    FAILED --> [*] : Max attempts reached
```

### 3.3 AgentProcess

```python
class AgentProcess:
    """
    Lightweight execution container for a single agent task.
    One AgentProcess per TaskNode. Runs in a worker process (not thread).
    """
    def __init__(
        self,
        task: TaskNode,
        context: ExecutionContext,
        agent_impl: BaseAgent,
        tool_runtime: ToolRuntime,
        shared_context: SharedContext,
        artifact_store: ArtifactStore,
        event_bus: EventBus,
    ):
        self.task = task
        self.context = context
        self.agent = agent_impl
        self.tools = tool_runtime
        self.shared_ctx = shared_context
        self.artifacts = artifact_store
        self.events = event_bus
        self._checkpoint_interval_s = 60
        self._last_checkpoint = time.monotonic()

    async def run(self) -> TaskResult:
        await self._emit(AgentEvent.STARTED)
        try:
            # Resume from checkpoint if available
            checkpoint = await self._load_checkpoint()
            if checkpoint:
                self.agent.restore(checkpoint)
                await self._emit(AgentEvent.RESUMED, checkpoint=checkpoint)

            result = await self.agent.execute(
                task=self.task,
                context=self.context,
                tools=self.tools,
                shared_ctx=self.shared_ctx,
                artifacts=self.artifacts,
                on_checkpoint=self._checkpoint_callback,
            )
            await self._emit(AgentEvent.COMPLETED, result=result)
            return result

        except TokenBudgetExceeded as e:
            await self._emit(AgentEvent.BUDGET_EXCEEDED, detail=str(e))
            raise
        except HumanApprovalRequired as e:
            await self._handle_approval_gate(e)
        except Exception as e:
            await self._emit(AgentEvent.FAILED, error=str(e))
            raise
        finally:
            await self._cleanup()

    async def _checkpoint_callback(self, state: AgentState) -> None:
        now = time.monotonic()
        if now - self._last_checkpoint >= self._checkpoint_interval_s:
            serialized = state.serialize()
            await self._save_checkpoint(serialized)
            self._last_checkpoint = now
            await self._emit(AgentEvent.CHECKPOINTED)
```

### 3.4 BaseAgent Interface

```python
class BaseAgent(ABC):
    """All 18 agent types implement this interface."""

    @property
    @abstractmethod
    def agent_type(self) -> str: ...

    @property
    @abstractmethod
    def capabilities(self) -> list[Capability]: ...

    @abstractmethod
    async def execute(
        self,
        task: TaskNode,
        context: ExecutionContext,
        tools: ToolRuntime,
        shared_ctx: SharedContext,
        artifacts: ArtifactStore,
        on_checkpoint: Callable[[AgentState], Awaitable[None]],
    ) -> TaskResult: ...

    @abstractmethod
    def restore(self, checkpoint: AgentCheckpoint) -> None: ...

    async def _call_model(
        self,
        messages: list[Message],
        tools: list[ToolSpec] | None = None,
        stream: bool = False,
    ) -> ModelResponse:
        """Route through ProviderGateway — never call provider SDK directly."""
        return await self._gateway.complete(
            model=self._context.model,
            messages=messages,
            tools=tools,
            stream=stream,
            execution_id=self._context.execution_id,
        )
```

### 3.5 Initialization Sequence

```mermaid
sequenceDiagram
    participant S as Scheduler
    participant W as WorkerPool
    participant AP as AgentProcess
    participant AR as AgentRegistry
    participant DI as DIContainer
    participant TR as ToolRuntime
    participant PM as PromptRegistry
    participant MO as MemoryOS

    S->>W: dispatch(task, context)
    W->>AR: resolve_agent(task.agent_type)
    AR->>DI: build_agent(agent_class, context)
    DI->>PM: resolve_prompts(agent_type, task_type, env)
    PM->>DI: ResolvedPrompts
    DI->>MO: load_employee_memory(employee_id)
    MO->>DI: MemoryContext
    DI->>TR: register_tools(agent.capabilities)
    TR->>DI: ToolHandles
    DI->>AP: AgentProcess(agent, tools, prompts, memory)
    AP->>AP: run()
```

### 3.6 Dependency Injection

The DI container provides all agent dependencies. Agents never directly instantiate infrastructure clients.

```python
class AgentDIContainer:
    def build(self, agent_class: type[BaseAgent], ctx: ExecutionContext) -> BaseAgent:
        return agent_class(
            context=ctx,
            gateway=self.provider_gateway,                      # AI provider calls
            tool_runtime=self._build_tool_runtime(ctx),
            prompt_registry=self.prompt_registry,
            memory=self._load_memory(ctx),
            shared_ctx=self._build_shared_ctx(ctx),
            artifact_store=self._build_artifact_store(ctx),
            event_bus=self.event_bus,
            constitution=self.constitution_pdp,
            logger=self._build_logger(ctx),
            tracer=self._build_tracer(ctx),
            cost_tracker=self._build_cost_tracker(ctx),
        )
```

### 3.7 Tool Registration

Tools are declared by capability and registered dynamically. The Constitution PDP intercepts every tool invocation before execution.

```python
class ToolRuntime:
    def register(self, capability: Capability, handler: ToolHandler) -> None:
        self._registry[capability.tool_name] = ToolEntry(
            capability=capability,
            handler=handler,
            schema=capability.input_schema,
            sandbox_required=capability.requires_sandbox,
        )

    async def execute(
        self,
        tool_name: str,
        tool_input: dict,
        caller: AgentProcess,
    ) -> ToolResult:
        entry = self._registry.get(tool_name)
        if not entry:
            raise ToolNotFound(tool_name)

        # Constitution PDP intercept — always before execution
        decision = await self.constitution_pdp.evaluate(
            agent_type=caller.agent.agent_type,
            tool_name=tool_name,
            tool_input=tool_input,
            context=caller.context,
        )
        if decision.action == "BLOCK":
            raise ConstitutionViolation(decision.rule_id, decision.reason)

        # Run in sandbox if required
        if entry.sandbox_required:
            return await self._sandbox.execute(entry.handler, tool_input)
        return await entry.handler(tool_input)
```

### 3.8 Context Injection into Model Calls

Before each LLM call, the agent assembles a context packet:

```python
class ContextInjector:
    async def build_messages(
        self,
        agent: BaseAgent,
        task: TaskNode,
        shared_ctx: SharedContext,
    ) -> list[Message]:
        # 1. System prompt (from Prompt Registry)
        system_prompt = await self.prompts.resolve(agent.agent_type, "system")

        # 2. Shared coding standards (always injected)
        coding_std = await self.prompts.resolve("shared", "coding_standards")

        # 3. Employee memory context (relevant past experience)
        memory_ctx = await self.memory.retrieve_relevant(
            namespace=agent.employee.memory_namespace,
            query=task.description,
            top_k=5,
        )

        # 4. Shared context snapshot (current workflow state)
        ctx_snap = await shared_ctx.snapshot()

        # 5. Task instruction
        task_prompt = await self.prompts.resolve(agent.agent_type, "task_execution")

        return [
            Message(role="system", content=f"{system_prompt}\n\n{coding_std}"),
            Message(role="user", content=format_memory(memory_ctx)),
            Message(role="user", content=format_context(ctx_snap)),
            Message(role="user", content=format_task(task_prompt, task)),
        ]
```

### 3.9 Checkpoint Protocol

Checkpoints allow execution to resume after a crash, timeout, or deliberate suspension.

```python
@dataclass
class AgentCheckpoint:
    checkpoint_id:  UUID
    execution_id:   UUID
    task_id:        UUID
    agent_type:     str
    iteration:      int                 # for loop stages
    token_count:    int                 # tokens consumed so far
    cost_usd:       Decimal             # cost so far
    conversation:   list[Message]       # full message history
    tool_results:   list[ToolResult]    # completed tool calls
    partial_output: dict                # outputs produced so far
    state_payload:  bytes               # agent-specific serialized state
    created_at:     datetime
    expires_at:     datetime            # checkpoint TTL = deadline_at + 1h

class CheckpointStore:
    async def save(self, checkpoint: AgentCheckpoint) -> None:
        # Write to PostgreSQL (durable) + Redis (fast read)
        await self.db.upsert_checkpoint(checkpoint)
        await self.redis.setex(
            f"chk:{checkpoint.execution_id}:{checkpoint.task_id}",
            int((checkpoint.expires_at - datetime.utcnow()).total_seconds()),
            checkpoint.serialize()
        )

    async def load(self, execution_id: UUID, task_id: UUID) -> AgentCheckpoint | None:
        raw = await self.redis.get(f"chk:{execution_id}:{task_id}")
        if raw:
            return AgentCheckpoint.deserialize(raw)
        return await self.db.load_checkpoint(execution_id, task_id)
```

### 3.10 Suspension and Resume

An agent raises `HumanApprovalRequired` when it encounters a task requiring human authorization. The `AgentProcess` catches this, serializes state, and suspends.

```python
async def _handle_approval_gate(self, exc: HumanApprovalRequired) -> None:
    # 1. Save current state as checkpoint
    checkpoint = await self.agent.get_checkpoint()
    await self.checkpoint_store.save(checkpoint)

    # 2. Create approval gate record
    gate = await self.approval_service.create_gate(
        task_id=self.task.node_id,
        execution_id=self.context.execution_id,
        reason=exc.reason,
        risk_score=exc.risk_score,
        simulation_id=exc.simulation_id,
        context_snapshot=await self.shared_ctx.snapshot(),
        timeout_s=exc.timeout_s or 86400,
    )

    # 3. Emit suspension event
    await self._emit(AgentEvent.SUSPENDED, gate_id=gate.id)

    # 4. Block this coroutine — resume when gate resolved
    decision = await self.approval_service.wait(gate.id)
    if decision.action == "approve":
        await self._emit(AgentEvent.RESUMED, gate_id=gate.id)
        # Re-enter execute() from checkpoint — handled by AgentProcess.run()
    elif decision.action == "reject":
        raise ExecutionCancelled(f"Approval gate rejected: {decision.reason}")
    elif decision.action == "modify":
        # Human modified the inputs — update shared context and resume
        await self.shared_ctx.set("human_override", decision.modified_input, "human")
        await self._emit(AgentEvent.RESUMED, gate_id=gate.id, modified=True)
```

### 3.11 Termination and Cleanup

```python
async def _cleanup(self) -> None:
    # Release tool runtime resources (close file handles, terminate subprocesses)
    await self.tools.release()
    # Flush cost records to ledger
    await self.context.cost_tracker.flush()
    # Update employee statistics
    await self.employee_service.update_stats(
        employee_id=self.task.assigned_to,
        task_result=self._last_result,
    )
    # Expire checkpoint if execution completed (no need to retain for resume)
    if self._last_result and self._last_result.status == TaskStatus.COMPLETED:
        await self.checkpoint_store.expire(self.context.execution_id, self.task.node_id)
```

---

## Chapter 4 — Task Scheduler

### 4.1 Purpose

The Task Scheduler receives `TaskNode` objects from the Workflow Engine and manages their dispatch to the Agent Worker Pool. It implements priority queuing, delayed and recurring tasks, fair load distribution across workers, retry management, and dead-letter handling. The scheduler is the central dispatcher — nothing runs without passing through it.

### 4.2 Queue Architecture

```mermaid
flowchart LR
    subgraph Input
        A[Workflow Engine] -->|submit| PQ
        B[Retry Queue] -->|requeue| PQ
        C[Delayed Queue] -->|trigger| PQ
    end

    subgraph Queues
        PQ[Priority Queue<br/>NATS JetStream]
        DQ[Delayed Queue<br/>Redis ZSET]
        RQ[Retry Queue<br/>NATS JetStream]
        DLQ[Dead Letter Queue<br/>PostgreSQL]
    end

    subgraph Workers
        PQ -->|pull| W1[Worker 1]
        PQ -->|pull| W2[Worker 2]
        PQ -->|pull| W3[Worker N]
    end

    W1 -->|task_failed, retry_allowed| RQ
    W1 -->|task_failed, max_retries| DLQ
    DQ -->|scheduled_at reached| PQ
```

### 4.3 Priority Queue

Tasks are scored and enqueued with a priority integer (0–100). The NATS JetStream priority is mapped via the `Nats-Priority` header.

```python
class PriorityScheduler:
    def compute_priority(self, task: TaskNode, ctx: ExecutionContext) -> int:
        base = ctx.priority                         # from ExecutionContext (0–100)
        # Boost for tasks blocking many dependents
        critical_path_boost = min(20, len(task.dependents) * 5)
        # Boost for tasks near deadline
        time_remaining_s = (ctx.deadline_at - datetime.utcnow()).total_seconds()
        urgency_boost = 15 if time_remaining_s < 3600 else 0
        # Penalty for expensive tasks (save budget)
        cost_penalty = 5 if task.estimated_cost_usd and task.estimated_cost_usd > 5.0 else 0
        return min(100, base + critical_path_boost + urgency_boost - cost_penalty)

    async def enqueue(self, task: TaskNode, ctx: ExecutionContext) -> None:
        priority = self.compute_priority(task, ctx)
        msg = TaskMessage(task=task, context_ref=ctx.execution_id, priority=priority)
        await self.nats.publish(
            subject=f"tasks.{task.agent_type.lower()}",
            payload=msg.serialize(),
            headers={"Nats-Priority": str(priority)},
        )
        await self.db.update_task_status(task.node_id, TaskStatus.QUEUED, priority=priority)
```

### 4.4 Delayed Tasks

Delayed tasks are stored in a Redis sorted set (score = epoch of trigger time) and promoted to the priority queue by a polling loop running every 100ms.

```python
class DelayedTaskQueue:
    REDIS_KEY = "scheduler:delayed"

    async def schedule(self, task: TaskNode, ctx: ExecutionContext, delay_s: int) -> None:
        fire_at = time.time() + delay_s
        payload = DelayedTaskEntry(task=task, context_id=ctx.execution_id).serialize()
        await self.redis.zadd(self.REDIS_KEY, {payload: fire_at})

    async def poll_loop(self) -> None:
        while True:
            now = time.time()
            due = await self.redis.zrangebyscore(self.REDIS_KEY, 0, now, start=0, num=100)
            for payload in due:
                entry = DelayedTaskEntry.deserialize(payload)
                await self.priority_scheduler.enqueue(entry.task, await self._load_ctx(entry.context_id))
                await self.redis.zrem(self.REDIS_KEY, payload)
            await asyncio.sleep(0.1)
```

### 4.5 Recurring Tasks

Recurring tasks (used by Continuous Evolution, Benchmark Center, APL maintenance checks) are registered as cron expressions and managed separately.

```sql
CREATE TABLE recurring_tasks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT UNIQUE NOT NULL,
    cron_expression TEXT NOT NULL,          -- '0 2 * * 0' (Sunday 02:00 UTC)
    workflow_id     TEXT NOT NULL,
    workflow_version TEXT NOT NULL,
    input_template  JSONB NOT NULL,
    enabled         BOOLEAN DEFAULT TRUE,
    last_fired_at   TIMESTAMPTZ,
    next_fire_at    TIMESTAMPTZ NOT NULL,
    created_by      UUID
);
```

The `RecurringTaskTrigger` (singleton) checks this table every 30 seconds and submits due tasks to the priority queue.

### 4.6 Worker Pool

```python
class AgentWorkerPool:
    """
    Manages a pool of worker processes.
    Each worker process hosts an AgentProcessRunner.
    Workers pull tasks from NATS JetStream using consumer pull model.
    """
    def __init__(self, config: WorkerPoolConfig):
        self._min_workers = config.min_workers      # default 2
        self._max_workers = config.max_workers      # default 32
        self._workers: list[WorkerProcess] = []
        self._autoscaler = WorkerAutoscaler(config)

    async def start(self) -> None:
        for _ in range(self._min_workers):
            await self._spawn_worker()
        asyncio.create_task(self._autoscaler.loop(self._workers))

    async def _spawn_worker(self) -> WorkerProcess:
        worker = WorkerProcess(
            nats_url=self._config.nats_url,
            subjects=self._config.subjects,
            max_concurrent=self._config.tasks_per_worker,   # default 4
        )
        await worker.start()
        self._workers.append(worker)
        return worker

    def current_load(self) -> float:
        active = sum(w.active_tasks for w in self._workers)
        capacity = sum(w.max_tasks for w in self._workers)
        return active / capacity if capacity else 0.0
```

### 4.7 Load Balancing

Workers use NATS JetStream pull consumers with `max_waiting` to self-regulate load. When a worker is at capacity, it stops pulling until a slot opens.

```
NATS JetStream consumer config per worker:
  durable:          "worker-{worker_id}"
  ack_policy:       explicit
  ack_wait:         30s
  max_deliver:      retry_policy.max_attempts
  filter_subjects:  ["tasks.backendagent", "tasks.qaagent", ...]  # agent-type filtering
  max_waiting:      1         # worker pulls one ahead — stops when at capacity
```

### 4.8 Retry Queue

When a task fails with a retryable error, it is re-enqueued with exponential backoff.

```python
class RetryPolicy:
    @classmethod
    def from_workflow(cls, workflow_def: WorkflowDefinition, stage_id: str) -> "RetryPolicy":
        stage = workflow_def.get_stage(stage_id)
        return cls(
            max_attempts=stage.retry_policy.max_attempts if stage.retry_policy else 3,
            backoff=stage.retry_policy.backoff if stage.retry_policy else "exponential",
            initial_delay_s=stage.retry_policy.initial_delay_s if stage.retry_policy else 30,
            retryable_errors={"ModelError", "ToolTimeout", "NetworkError"},
            non_retryable_errors={"ConstitutionViolation", "BudgetExceeded", "Cancelled"},
        )

    def next_delay(self, attempt: int) -> int:
        if self.backoff == "exponential":
            return min(self.initial_delay_s * (2 ** (attempt - 1)), 3600)
        return self.initial_delay_s

    def is_retryable(self, error: Exception) -> bool:
        return type(error).__name__ in self.retryable_errors
```

### 4.9 Dead Letter Queue

Tasks that exhaust all retry attempts are moved to the Dead Letter Queue, triggering an escalation.

```sql
CREATE TABLE dead_letter_queue (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id         UUID NOT NULL,
    execution_id    UUID NOT NULL,
    workflow_id     TEXT NOT NULL,
    stage_id        TEXT NOT NULL,
    agent_type      TEXT NOT NULL,
    attempts        INTEGER NOT NULL,
    last_error      TEXT NOT NULL,
    last_error_type TEXT NOT NULL,
    task_payload    JSONB NOT NULL,
    context_payload JSONB NOT NULL,
    enqueued_at     TIMESTAMPTZ DEFAULT NOW(),
    resolution      TEXT,           -- 'manual_retry'|'cancelled'|'escalated'
    resolved_at     TIMESTAMPTZ,
    resolved_by     UUID
);
```

DLQ entries trigger:
1. An `EscalationEngine` call (Ch.4 of 2.0 Extension) to notify the responsible engineer
2. An approval gate allowing manual retry or cancellation
3. A `task.dead_lettered` event for observability

---

## Chapter 5 — Event Bus

### 5.1 Purpose

The Event Bus is the nervous system of the AI Studio runtime. Every state change — task started, agent checkpointed, approval granted, artifact produced, budget threshold crossed — is an event. Components communicate exclusively via events; there are no synchronous point-to-point calls between runtime services except for the Intake API and gRPC tool calls.

### 5.2 Event-Driven Architecture

```mermaid
flowchart LR
    subgraph Producers
        WE[Workflow Engine]
        AR[Agent Runtime]
        TR[Tool Runtime]
        SIM[Simulation Engine]
        BI[Business Intelligence]
    end

    subgraph NATS_JetStream[NATS JetStream]
        TASKS[Stream: TASKS]
        AGENT[Stream: AGENT]
        TOOLS[Stream: TOOLS]
        EXEC[Stream: EXECUTIONS]
        COST[Stream: COSTS]
        AUDIT[Stream: AUDIT]
    end

    subgraph Consumers
        TS[Task Scheduler]
        OB[Observability]
        AL[Audit Logger]
        CT[Cost Tracker]
        EVO[Evolution Engine]
        NTF[Notifier]
    end

    WE --> TASKS
    AR --> AGENT
    TR --> TOOLS
    AR --> EXEC
    AR --> COST
    AR --> AUDIT

    TASKS --> TS
    AGENT --> OB
    AGENT --> AL
    AGENT --> EVO
    EXEC --> NTF
    COST --> CT
    AUDIT --> AL
```

### 5.3 Stream Configuration

```python
STREAM_DEFINITIONS = {
    "TASKS": StreamConfig(
        subjects=["tasks.>"],
        retention=RetentionPolicy.WORK_QUEUE,
        storage=StorageType.FILE,
        max_age=timedelta(hours=24),
        replicas=3,
    ),
    "AGENT": StreamConfig(
        subjects=["agent.>"],
        retention=RetentionPolicy.LIMITS,
        storage=StorageType.FILE,
        max_age=timedelta(days=7),
        replicas=3,
    ),
    "EXECUTIONS": StreamConfig(
        subjects=["execution.>"],
        retention=RetentionPolicy.LIMITS,
        storage=StorageType.FILE,
        max_age=timedelta(days=30),
        replicas=3,
    ),
    "AUDIT": StreamConfig(
        subjects=["audit.>"],
        retention=RetentionPolicy.LIMITS,
        storage=StorageType.FILE,
        max_age=timedelta(days=365),    # 1-year retention for compliance
        replicas=3,
    ),
    "COSTS": StreamConfig(
        subjects=["cost.>"],
        retention=RetentionPolicy.LIMITS,
        storage=StorageType.FILE,
        max_age=timedelta(days=90),
        replicas=3,
    ),
}
```

### 5.4 Event Schema

All events share a common envelope:

```python
@dataclass
class RuntimeEvent:
    event_id:       UUID = field(default_factory=uuid4)
    event_type:     str  = ""           # e.g., 'agent.task.started'
    version:        str  = "1.0"        # event schema version
    source:         str  = ""           # component name
    execution_id:   UUID | None = None
    task_id:        UUID | None = None
    trace_id:       str  = ""
    correlation_id: UUID | None = None
    timestamp:      datetime = field(default_factory=datetime.utcnow)
    payload:        dict = field(default_factory=dict)

    def to_nats_msg(self) -> bytes:
        return orjson.dumps(asdict(self))

    @classmethod
    def from_nats_msg(cls, data: bytes) -> "RuntimeEvent":
        return cls(**orjson.loads(data))
```

### 5.5 Event Catalog

| Subject | Event Type | Payload Keys | Published By |
|---------|-----------|-------------|-------------|
| `execution.created` | `execution.created` | execution_id, workflow_id, factory_id | Intake API |
| `execution.completed` | `execution.completed` | execution_id, duration_s, cost_usd | Workflow Engine |
| `execution.failed` | `execution.failed` | execution_id, error, failed_task_id | Workflow Engine |
| `agent.task.started` | `agent.task.started` | task_id, agent_type, employee_id | Agent Runtime |
| `agent.task.completed` | `agent.task.completed` | task_id, outputs, cost_usd, tokens | Agent Runtime |
| `agent.task.failed` | `agent.task.failed` | task_id, error, attempt | Agent Runtime |
| `agent.task.suspended` | `agent.task.suspended` | task_id, gate_id, risk_score | Agent Runtime |
| `agent.task.checkpointed` | `agent.task.checkpointed` | task_id, checkpoint_id | Agent Runtime |
| `tool.called` | `tool.called` | task_id, tool_name, duration_ms | Tool Runtime |
| `tool.blocked` | `tool.blocked` | task_id, tool_name, rule_id | Constitution PDP |
| `artifact.produced` | `artifact.produced` | artifact_id, type, size_bytes | Artifact Store |
| `budget.warning` | `budget.threshold` | execution_id, pct_consumed, remaining_usd | Cost Tracker |
| `budget.exceeded` | `budget.exceeded` | execution_id, budget_usd, actual_usd | Cost Tracker |
| `approval.requested` | `approval.gate.requested` | gate_id, task_id, risk_score | Agent Runtime |
| `approval.granted` | `approval.gate.resolved` | gate_id, decision, modified | Approval Service |
| `approval.rejected` | `approval.gate.resolved` | gate_id, decision, reason | Approval Service |

### 5.6 Publisher

```python
class EventPublisher:
    def __init__(self, nc: NATSClient, tracer: Tracer):
        self._nc = nc
        self._tracer = tracer

    async def publish(self, event: RuntimeEvent) -> None:
        subject = self._subject(event.event_type)
        with self._tracer.start_span(f"event.publish.{event.event_type}") as span:
            span.set_attribute("event.id", str(event.event_id))
            span.set_attribute("event.type", event.event_type)
            await self._nc.publish(
                subject=subject,
                payload=event.to_nats_msg(),
                headers={
                    "Event-Id":        str(event.event_id),
                    "Event-Type":      event.event_type,
                    "Trace-Id":        event.trace_id,
                    "Correlation-Id":  str(event.correlation_id or ""),
                },
            )

    def _subject(self, event_type: str) -> str:
        # 'agent.task.completed' → 'agent.task.completed'
        # Map to stream subject via prefix
        return event_type
```

### 5.7 Subscriber

```python
class EventSubscriber:
    async def subscribe(
        self,
        subject: str,
        consumer_name: str,
        handler: Callable[[RuntimeEvent], Awaitable[None]],
        batch_size: int = 10,
    ) -> None:
        consumer = await self._js.find_or_create_consumer(subject, consumer_name)
        while True:
            msgs = await consumer.fetch(batch=batch_size, timeout=5.0)
            for msg in msgs:
                event = RuntimeEvent.from_nats_msg(msg.data)
                try:
                    await handler(event)
                    await msg.ack()
                except Exception as e:
                    if self._is_retryable(e):
                        await msg.nak(delay=30)     # NATS NAK with redelivery delay
                    else:
                        await msg.term()            # non-retryable: terminate delivery
                        await self._dlq_publisher.publish(event, error=e)
```

### 5.8 Event Store and Replay

Every event is persisted to the `AUDIT` stream (retention: 1 year) and mirrored to PostgreSQL for structured querying. Replay allows re-driving any execution from a specific event offset — used for debugging, post-mortem analysis, and testing.

```python
class EventReplay:
    async def replay(
        self,
        execution_id: UUID,
        from_event_id: UUID | None = None,
        to_event_id: UUID | None = None,
        target: EventHandler | None = None,
    ) -> AsyncIterator[RuntimeEvent]:
        events = await self.event_store.load(
            execution_id=execution_id,
            from_id=from_event_id,
            to_id=to_event_id,
        )
        for event in events:
            if target:
                await target.handle(event)
            yield event
```

### 5.9 Idempotency

Every event handler that produces side effects must be idempotent. The platform enforces idempotency via a Redis idempotency key (TTL = 24h):

```python
class IdempotentEventHandler:
    async def handle(self, event: RuntimeEvent) -> None:
        idem_key = f"idem:{self.handler_name}:{event.event_id}"
        already_processed = await self.redis.set(
            idem_key, "1", nx=True, ex=86400
        )
        if not already_processed:
            return  # duplicate delivery — skip
        await self._do_handle(event)
```

### 5.10 Correlation IDs and Distributed Tracing

Every event carries a `trace_id` (OpenTelemetry W3C format) and a `correlation_id` (UUID that groups all events from one factory execution). The tracing backend (Jaeger or Tempo) receives spans via OTLP. Correlation IDs allow filtering the full event history of a single product factory run in Kibana or Grafana Explore.

```python
class TracingMiddleware:
    async def __call__(self, event: RuntimeEvent, handler: EventHandler) -> None:
        ctx = extract_trace_context(event.trace_id)
        with self.tracer.start_as_current_span(
            name=f"event.handle.{event.event_type}",
            context=ctx,
            kind=SpanKind.CONSUMER,
        ) as span:
            span.set_attribute("event.id", str(event.event_id))
            span.set_attribute("correlation.id", str(event.correlation_id))
            span.set_attribute("execution.id", str(event.execution_id))
            await handler.handle(event)
```

---


---

## Chapter 6 — State Machine

### 6.1 Purpose

Every entity in the runtime — executions, workflows, tasks, agents — has a formally defined set of states and a set of legal transitions between them. The State Machine Engine enforces these transitions, records every change immutably, and drives recovery and compensation when the expected path is interrupted. No component may change an entity's state except through the State Machine Engine.

### 6.2 Entity States

#### Execution States

```mermaid
stateDiagram-v2
    [*] --> CREATED : Intake API receives request
    CREATED --> COMPILING : Workflow Compiler invoked
    COMPILING --> SCHEDULED : Compilation success
    COMPILING --> FAILED : Compilation error
    SCHEDULED --> RUNNING : First task dispatched
    RUNNING --> PAUSED : Human pause request
    PAUSED --> RUNNING : Resume request
    RUNNING --> AWAITING_APPROVAL : Approval gate raised
    AWAITING_APPROVAL --> RUNNING : Gate approved
    AWAITING_APPROVAL --> CANCELLED : Gate rejected
    RUNNING --> COMPENSATING : Unrecoverable failure detected
    COMPENSATING --> FAILED : Compensation complete
    COMPENSATING --> RUNNING : Compensation succeeded, retry allowed
    RUNNING --> COMPLETED : All tasks terminal, no failures
    RUNNING --> FAILED : Unrecoverable failure, no compensation
    COMPLETED --> ARCHIVED : Context archival complete
    FAILED --> ARCHIVED : Failure context archived
    CANCELLED --> ARCHIVED : Cancellation finalized
```

#### Task States

```mermaid
stateDiagram-v2
    [*] --> PENDING : Task node created in graph
    PENDING --> BLOCKED : Dependency not satisfied
    BLOCKED --> PENDING : Dependency completes
    PENDING --> QUEUED : Scheduler enqueues
    QUEUED --> RUNNING : Worker pulls from queue
    RUNNING --> CHECKPOINTED : Checkpoint saved
    CHECKPOINTED --> RUNNING : Continues (checkpoint is non-terminal)
    RUNNING --> SUSPENDED : Human approval required
    SUSPENDED --> RUNNING : Approval granted
    SUSPENDED --> CANCELLED : Approval rejected
    RUNNING --> COMPLETED : Task execution success
    RUNNING --> FAILED : Execution error
    FAILED --> QUEUED : Retry policy permits
    FAILED --> DEAD_LETTERED : Max retries exhausted
    PENDING --> SKIPPED : Stage condition evaluated false
    COMPLETED --> [*]
    SKIPPED --> [*]
    CANCELLED --> [*]
    DEAD_LETTERED --> [*]
```

#### Agent States

```
INITIALIZING → READY → RUNNING → [CHECKPOINTED] → COMPLETED
                                       ↓
                              SUSPENDED → RUNNING
                                       ↓
                              CANCELLED
                       ↓
               FAILED → RUNNING (retry)
                       ↓
               TERMINATED (max retries or budget)
```

### 6.3 State Transition Engine

```python
class StateMachineEngine:
    # Allowed transitions per entity type
    EXECUTION_TRANSITIONS: dict[str, set[str]] = {
        "CREATED":            {"COMPILING"},
        "COMPILING":          {"SCHEDULED", "FAILED"},
        "SCHEDULED":          {"RUNNING"},
        "RUNNING":            {"PAUSED", "AWAITING_APPROVAL", "COMPENSATING", "COMPLETED", "FAILED"},
        "PAUSED":             {"RUNNING", "CANCELLED"},
        "AWAITING_APPROVAL":  {"RUNNING", "CANCELLED"},
        "COMPENSATING":       {"FAILED", "RUNNING"},
        "COMPLETED":          {"ARCHIVED"},
        "FAILED":             {"ARCHIVED"},
        "CANCELLED":          {"ARCHIVED"},
    }

    TASK_TRANSITIONS: dict[str, set[str]] = {
        "PENDING":       {"BLOCKED", "QUEUED", "SKIPPED"},
        "BLOCKED":       {"PENDING"},
        "QUEUED":        {"RUNNING", "CANCELLED"},
        "RUNNING":       {"COMPLETED", "FAILED", "SUSPENDED", "CHECKPOINTED"},
        "CHECKPOINTED":  {"RUNNING"},
        "SUSPENDED":     {"RUNNING", "CANCELLED"},
        "FAILED":        {"QUEUED", "DEAD_LETTERED"},
    }

    async def transition(
        self,
        entity_type: str,
        entity_id: UUID,
        to_state: str,
        triggered_by: str,
        payload: dict | None = None,
    ) -> StateTransition:
        current = await self.db.get_state(entity_type, entity_id)
        transitions = self._transitions_for(entity_type)
        if to_state not in transitions.get(current, set()):
            raise IllegalStateTransition(entity_type, entity_id, current, to_state)

        transition = StateTransition(
            entity_type=entity_type,
            entity_id=entity_id,
            from_state=current,
            to_state=to_state,
            triggered_by=triggered_by,
            payload=payload or {},
            transitioned_at=datetime.utcnow(),
        )
        await self.db.apply_transition(transition)
        await self.event_bus.publish(self._transition_event(transition))
        return transition
```

### 6.4 State Persistence Schema

```sql
-- Current state (fast lookup)
CREATE TABLE entity_states (
    entity_type     TEXT NOT NULL,
    entity_id       UUID NOT NULL,
    current_state   TEXT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (entity_type, entity_id)
);

-- Append-only state history
CREATE TABLE state_transitions (
    id              BIGSERIAL PRIMARY KEY,
    entity_type     TEXT NOT NULL,
    entity_id       UUID NOT NULL,
    from_state      TEXT NOT NULL,
    to_state        TEXT NOT NULL,
    triggered_by    TEXT NOT NULL,
    payload         JSONB DEFAULT '{}',
    transitioned_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_st_entity ON state_transitions(entity_type, entity_id, transitioned_at);
-- No UPDATE or DELETE on state_transitions — append-only enforced via trigger.
```

### 6.5 Recovery

On service restart or crash, the State Machine Engine scans for executions in transient states and triggers recovery:

```python
class RecoveryEngine:
    TRANSIENT_EXECUTION_STATES = {"RUNNING", "COMPILING", "COMPENSATING"}
    TRANSIENT_TASK_STATES = {"QUEUED", "RUNNING", "CHECKPOINTED"}

    async def recover_on_startup(self) -> None:
        stuck_executions = await self.db.find_entities_in_states(
            "execution", self.TRANSIENT_EXECUTION_STATES,
            last_updated_before=datetime.utcnow() - timedelta(minutes=5)
        )
        for exec_id in stuck_executions:
            await self._recover_execution(exec_id)

        stuck_tasks = await self.db.find_entities_in_states(
            "task", self.TRANSIENT_TASK_STATES,
            last_updated_before=datetime.utcnow() - timedelta(minutes=5)
        )
        for task_id in stuck_tasks:
            await self._recover_task(task_id)

    async def _recover_task(self, task_id: UUID) -> None:
        task = await self.db.load_task(task_id)
        checkpoint = await self.checkpoint_store.load(task.execution_id, task_id)
        if checkpoint:
            # Resume from checkpoint
            await self.scheduler.enqueue_resume(task, checkpoint)
        else:
            # Re-queue from scratch if within retry limits
            if task.attempt < task.retry_policy.max_attempts:
                await self.scheduler.enqueue(task, await self.db.load_context(task.execution_id))
            else:
                await self.state_machine.transition("task", task_id, "DEAD_LETTERED", "recovery")
```

### 6.6 Compensation

When a workflow stage fails and the stage declares `compensate` steps, the Compensation Engine executes them in reverse order of the completed stages.

```python
class CompensationEngine:
    async def compensate(self, execution: Execution) -> CompensationResult:
        # Identify completed stages in reverse chronological order
        completed = await self.db.get_completed_stages(execution.id)
        completed.sort(key=lambda s: s.completed_at, reverse=True)

        results = []
        for stage in completed:
            if stage.compensate_steps:
                for step_name in stage.compensate_steps:
                    try:
                        result = await self._run_compensate_step(step_name, stage, execution)
                        results.append(CompensateResult(stage=stage.stage_id, step=step_name, success=True))
                    except Exception as e:
                        results.append(CompensateResult(stage=stage.stage_id, step=step_name, success=False, error=str(e)))
                        # Compensation failure is logged but does not abort other compensations
                        await self.event_bus.publish(RuntimeEvent(
                            event_type="compensation.step.failed",
                            execution_id=execution.id,
                            payload={"stage": stage.stage_id, "step": step_name, "error": str(e)}
                        ))
        return CompensationResult(steps=results, all_succeeded=all(r.success for r in results))
```

### 6.7 Rollback

Rollback is a special case of compensation targeting a specific point in time:

```python
class RollbackService:
    async def rollback_to_checkpoint(
        self,
        execution_id: UUID,
        target_checkpoint_id: UUID,
    ) -> None:
        checkpoint = await self.checkpoint_store.load_by_id(target_checkpoint_id)
        # 1. Cancel all running tasks
        await self.scheduler.cancel_all(execution_id)
        # 2. Restore workspace to checkpoint state
        await self.workspace_service.restore(execution_id, checkpoint.workspace_snapshot)
        # 3. Restore shared context to checkpoint state
        await self.shared_context_service.restore(execution_id, checkpoint.context_snapshot)
        # 4. Re-queue tasks from the checkpoint point
        await self.workflow_engine.resume_from_checkpoint(execution_id, checkpoint)
        # 5. Emit rollback event
        await self.event_bus.publish(RuntimeEvent(
            event_type="execution.rolled_back",
            execution_id=execution_id,
            payload={"checkpoint_id": str(target_checkpoint_id)}
        ))
```

---

## Chapter 7 — Distributed Execution

### 7.1 Purpose

A single AI Studio instance can process dozens of simultaneous product factory executions, each running dozens of agent tasks in parallel. The Distributed Execution system manages a cluster of Coordinator and Worker nodes, handles leader election, detects and recovers from node failures, and scales horizontally based on queue depth and CPU load.

### 7.2 Cluster Architecture

```mermaid
flowchart TB
    subgraph External
        LB[Load Balancer<br/>nginx / Traefik]
        U[User / Factory API]
    end

    subgraph Coordinator_Cluster
        C1[Coordinator-1<br/>★ LEADER]
        C2[Coordinator-2<br/>STANDBY]
    end

    subgraph Worker_Nodes
        WN1[Worker Node 1<br/>4 × AgentWorker]
        WN2[Worker Node 2<br/>4 × AgentWorker]
        WN3[Worker Node N<br/>4 × AgentWorker]
    end

    subgraph Data_Layer
        PG[(PostgreSQL<br/>Primary + 2 Replicas)]
        RD[(Redis<br/>3-node cluster)]
        NJ[(NATS JetStream<br/>3-node cluster)]
        MN[(MinIO<br/>4-node erasure)]
        NEO[(Neo4j<br/>Core + 2 Read)]
        QD[(Qdrant<br/>3-shard cluster)]
    end

    U --> LB
    LB --> C1
    LB --> C2
    C1 <-->|Raft| C2
    C1 --> NJ
    NJ --> WN1
    NJ --> WN2
    NJ --> WN3
    C1 --> PG
    WN1 --> PG
    WN1 --> RD
    WN1 --> MN
    WN1 --> NEO
    WN1 --> QD
```

### 7.3 Leader Election

The Coordinator cluster uses NATS JetStream's built-in leader election (KV-based mutex) to elect a single active Coordinator. The leader owns the task scheduling loop; standbys are hot replicas.

```python
class CoordinatorLeaderElection:
    LOCK_KEY = "coordinator:leader"
    TTL_S = 15                  # leader must renew every 15s

    async def campaign(self) -> None:
        while True:
            acquired = await self.nats_kv.create(
                key=self.LOCK_KEY,
                value=self.node_id.encode(),
                ttl=self.TTL_S,
            )
            if acquired:
                self._is_leader = True
                await self.event_bus.publish(RuntimeEvent(event_type="coordinator.elected_leader",
                                                          payload={"node_id": str(self.node_id)}))
                asyncio.create_task(self._renew_loop())
                return
            # Not leader — watch for key expiry and retry
            await self.nats_kv.watch_until_deleted(self.LOCK_KEY)

    async def _renew_loop(self) -> None:
        while self._is_leader:
            await asyncio.sleep(self.TTL_S * 0.6)
            renewed = await self.nats_kv.update(
                key=self.LOCK_KEY,
                value=self.node_id.encode(),
                revision=self._last_revision,
                ttl=self.TTL_S,
            )
            if not renewed:
                self._is_leader = False
                await self.campaign()       # lost lock, re-campaign
```

### 7.4 Worker Node Registration

Worker nodes self-register on startup and maintain a heartbeat:

```python
class WorkerNode:
    HEARTBEAT_INTERVAL_S = 5
    HEARTBEAT_TTL_S = 20

    async def start(self) -> None:
        await self._register()
        asyncio.create_task(self._heartbeat_loop())
        asyncio.create_task(self._pull_loop())

    async def _register(self) -> None:
        await self.nats_kv.put(
            key=f"workers:{self.node_id}",
            value=orjson.dumps({
                "node_id": str(self.node_id),
                "hostname": self.hostname,
                "capacity": self.max_tasks,
                "active_tasks": 0,
                "registered_at": datetime.utcnow().isoformat(),
            }),
        )

    async def _heartbeat_loop(self) -> None:
        while True:
            await self.nats_kv.put(
                key=f"workers:{self.node_id}:heartbeat",
                value=orjson.dumps({
                    "active_tasks": self.active_task_count(),
                    "cpu_pct": psutil.cpu_percent(),
                    "mem_pct": psutil.virtual_memory().percent,
                    "ts": datetime.utcnow().isoformat(),
                }),
                ttl=self.HEARTBEAT_TTL_S,
            )
            await asyncio.sleep(self.HEARTBEAT_INTERVAL_S)
```

The Coordinator watches heartbeat keys. If a heartbeat key expires (TTL elapsed), the worker is considered failed.

### 7.5 Health Checks

```python
class ClusterHealthMonitor:
    async def check(self) -> ClusterHealth:
        workers = await self._load_all_workers()
        return ClusterHealth(
            coordinator_leader=await self._get_leader(),
            total_workers=len(workers),
            healthy_workers=sum(1 for w in workers if w.is_healthy()),
            total_capacity=sum(w.max_tasks for w in workers),
            active_tasks=sum(w.active_tasks for w in workers),
            queue_depth=await self._queue_depth(),
            nats_healthy=await self._check_nats(),
            postgres_healthy=await self._check_postgres(),
            redis_healthy=await self._check_redis(),
        )

    async def _check_nats(self) -> bool:
        try:
            await self.nats.publish("health.ping", b"ping")
            return True
        except Exception:
            return False
```

**Health check endpoints (HTTP GET, no auth):**

| Endpoint | Response | Purpose |
|----------|----------|---------|
| `GET /health/live` | `200 OK` if process is running | K8s liveness probe |
| `GET /health/ready` | `200 OK` if all deps connected | K8s readiness probe |
| `GET /health/cluster` | JSON cluster summary | Ops monitoring |

### 7.6 Autoscaling

The Coordinator's autoscaler monitors queue depth and worker load to trigger horizontal scaling.

```python
class WorkerAutoscaler:
    CHECK_INTERVAL_S = 30
    SCALE_UP_THRESHOLD = 0.80       # queue_depth / capacity > 80%
    SCALE_DOWN_THRESHOLD = 0.20     # queue_depth / capacity < 20%
    MIN_WORKERS = 2
    MAX_WORKERS = 32
    COOLDOWN_S = 120                # min seconds between scale events

    async def loop(self) -> None:
        last_scale_at = 0.0
        while True:
            await asyncio.sleep(self.CHECK_INTERVAL_S)
            if time.monotonic() - last_scale_at < self.COOLDOWN_S:
                continue
            health = await self.health_monitor.check()
            utilization = health.active_tasks / health.total_capacity if health.total_capacity else 1.0
            queue_ratio = health.queue_depth / health.total_capacity if health.total_capacity else 1.0

            if queue_ratio > self.SCALE_UP_THRESHOLD and health.total_workers < self.MAX_WORKERS:
                await self._scale_up(1)
                last_scale_at = time.monotonic()
            elif queue_ratio < self.SCALE_DOWN_THRESHOLD and health.total_workers > self.MIN_WORKERS:
                await self._scale_down(1)
                last_scale_at = time.monotonic()

    async def _scale_up(self, count: int) -> None:
        # Trigger K8s HPA or custom cloud API
        await self.k8s.scale_deployment("agent-worker", delta=+count)
        await self.event_bus.publish(RuntimeEvent(event_type="cluster.scaled_up", payload={"delta": count}))

    async def _scale_down(self, count: int) -> None:
        # Only scale down idle workers (zero active tasks)
        idle = await self._find_idle_workers(count)
        for worker_id in idle:
            await self.k8s.cordon_and_drain(worker_id)
        await self.event_bus.publish(RuntimeEvent(event_type="cluster.scaled_down", payload={"count": len(idle)}))
```

**K8s HPA manifest for agent-worker:**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: agent-worker-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: agent-worker
  minReplicas: 2
  maxReplicas: 32
  metrics:
    - type: External
      external:
        metric:
          name: nats_consumer_pending_messages
          selector:
            matchLabels:
              stream: TASKS
        target:
          type: Value
          value: "10"       # scale up when > 10 pending per replica
```

### 7.7 Task Redistribution on Worker Failure

When a worker fails (heartbeat TTL expires), its in-progress tasks are detected and re-queued:

```python
class FailoverService:
    async def handle_worker_failure(self, worker_id: UUID) -> None:
        # Find tasks that were running on this worker
        orphaned = await self.db.find_tasks_by_worker(worker_id, status="RUNNING")
        for task in orphaned:
            checkpoint = await self.checkpoint_store.load(task.execution_id, task.node_id)
            if checkpoint:
                # Re-queue with checkpoint — minimal work lost
                await self.scheduler.enqueue_resume(task, checkpoint)
            else:
                # Re-queue from scratch if within retry limits
                task.attempt += 1
                if task.attempt <= task.retry_policy.max_attempts:
                    await self.scheduler.enqueue(task, await self.db.load_context(task.execution_id))
                else:
                    await self.state_machine.transition("task", task.node_id, "DEAD_LETTERED", "failover")
        await self.event_bus.publish(RuntimeEvent(
            event_type="worker.failed",
            payload={"worker_id": str(worker_id), "orphaned_tasks": len(orphaned)}
        ))
```

---

## Chapter 8 — Reliability

### 8.1 Purpose

The Reliability layer ensures that temporary failures — network partitions, model provider outages, transient database errors, LLM hallucinations causing bad tool calls — do not result in lost work or permanent execution failures. Every failure mode has a defined response: retry, circuit break, compensate, checkpoint-resume, or escalate.

### 8.2 Retry Policies

```python
class RetryPolicy:
    """
    Defined per-stage in workflow DSL.
    Defaults applied if not specified.
    """
    def __init__(
        self,
        max_attempts: int = 3,
        backoff: Literal["constant", "linear", "exponential"] = "exponential",
        initial_delay_s: int = 30,
        max_delay_s: int = 3600,
        jitter: bool = True,                    # prevent thundering herd
        retryable_errors: set[str] | None = None,
    ):
        self.max_attempts = max_attempts
        self.backoff = backoff
        self.initial_delay_s = initial_delay_s
        self.max_delay_s = max_delay_s
        self.jitter = jitter
        self.retryable_errors = retryable_errors or {
            "ModelError", "ToolTimeout", "NetworkError",
            "RateLimitError", "ProviderUnavailable",
        }

    def delay_for(self, attempt: int) -> float:
        if self.backoff == "constant":
            base = self.initial_delay_s
        elif self.backoff == "linear":
            base = self.initial_delay_s * attempt
        else:  # exponential
            base = self.initial_delay_s * (2 ** (attempt - 1))
        base = min(base, self.max_delay_s)
        if self.jitter:
            base *= (0.8 + random.random() * 0.4)   # ±20% jitter
        return base
```

### 8.3 Circuit Breaker

Applied at the AI provider gateway. Prevents cascading failures when a provider is degraded.

```python
class CircuitBreaker:
    def __init__(self, name: str, config: CircuitBreakerConfig):
        self.name = name
        self._state = CircuitState.CLOSED
        self._failure_count = 0
        self._last_failure_at: float = 0.0
        self._config = config           # failure_threshold, recovery_timeout_s, half_open_max

    async def call(self, fn: Callable) -> Any:
        if self._state == CircuitState.OPEN:
            if time.monotonic() - self._last_failure_at > self._config.recovery_timeout_s:
                self._state = CircuitState.HALF_OPEN
            else:
                raise CircuitOpenError(self.name)

        try:
            result = await fn()
            if self._state == CircuitState.HALF_OPEN:
                self._reset()
            return result
        except Exception as e:
            self._record_failure()
            raise

    def _record_failure(self) -> None:
        self._failure_count += 1
        self._last_failure_at = time.monotonic()
        if self._failure_count >= self._config.failure_threshold:
            self._state = CircuitState.OPEN
            asyncio.create_task(self.event_bus.publish(RuntimeEvent(
                event_type="circuit_breaker.opened",
                payload={"name": self.name}
            )))

    def _reset(self) -> None:
        self._failure_count = 0
        self._state = CircuitState.CLOSED
```

**Circuit breaker configuration per provider:**

```yaml
circuit_breakers:
  anthropic:
    failure_threshold: 5
    recovery_timeout_s: 60
    half_open_max: 2
  openai:
    failure_threshold: 5
    recovery_timeout_s: 60
  ollama:
    failure_threshold: 3
    recovery_timeout_s: 30
```

### 8.4 Timeout Handling

Every async operation in the runtime carries an explicit timeout. Timeouts are propagated from ExecutionContext.deadline_at.

```python
class TimeoutGuard:
    async def __aenter__(self) -> "TimeoutGuard":
        remaining = (self.context.deadline_at - datetime.utcnow()).total_seconds()
        if remaining <= 0:
            raise ExecutionDeadlineExceeded(self.context.execution_id)
        self._timeout = min(remaining, self.operation_timeout_s)
        return self

    async def __aexit__(self, exc_type, exc, tb) -> None:
        pass

# Usage:
async with TimeoutGuard(context, operation_timeout_s=300):
    result = await agent.execute(...)
```

**Timeout hierarchy (innermost wins if stricter):**

```
ExecutionContext.time_budget_s          → global execution deadline
  Stage.timeout_seconds                 → per-stage limit
    Task retry total                    → sum of all attempt budgets
      LLM completion timeout            → 120s default
        Tool execution timeout          → tool-specific (10s-300s)
          Shell command timeout         → 60s default
```

### 8.5 Cancellation

Cancellation is cooperative via a `CancellationToken` threaded through the execution:

```python
class CancellationToken:
    def __init__(self):
        self._cancelled = asyncio.Event()
        self._reason: str = ""

    def cancel(self, reason: str = "") -> None:
        self._reason = reason
        self._cancelled.set()

    def is_cancelled(self) -> bool:
        return self._cancelled.is_set()

    async def wait_or_raise(self, coro: Awaitable[T]) -> T:
        done, pending = await asyncio.wait(
            {asyncio.create_task(coro), asyncio.create_task(self._cancelled.wait())},
            return_when=asyncio.FIRST_COMPLETED,
        )
        if self._cancelled.is_set():
            for t in pending:
                t.cancel()
            raise ExecutionCancelled(self._reason)
        return done.pop().result()
```

### 8.6 Fallback Chain

When the preferred model or tool fails beyond circuit breaker threshold, the fallback chain activates:

```python
FALLBACK_CHAINS: dict[str, list[str]] = {
    "claude-sonnet-4-6": ["claude-haiku-4-5", "gpt-4o", "ollama/qwen2.5-coder:32b"],
    "claude-opus-4-8":   ["claude-sonnet-4-6", "gpt-4o"],
    "gpt-4o":            ["claude-sonnet-4-6", "ollama/llama3.3:70b"],
}

class FallbackChainExecutor:
    async def execute_with_fallback(
        self,
        preferred_model: str,
        request: CompletionRequest,
        context: ExecutionContext,
    ) -> CompletionResponse:
        chain = [preferred_model] + FALLBACK_CHAINS.get(preferred_model, [])
        last_error = None
        for model in chain:
            try:
                if not self.circuit_breakers[self._provider(model)].is_open():
                    return await self.gateway.complete(request.with_model(model))
            except (CircuitOpenError, ProviderUnavailable) as e:
                last_error = e
                await self.event_bus.publish(RuntimeEvent(
                    event_type="model.fallback",
                    execution_id=context.execution_id,
                    payload={"from": preferred_model, "to": model, "reason": str(e)}
                ))
        raise AllProvidersUnavailable(preferred_model, last_error)
```

### 8.7 Saga Pattern

Multi-stage workflows that touch multiple external systems use the Saga pattern to ensure all-or-nothing semantics across distributed operations.

```python
class SagaOrchestrator:
    """
    Manages a sequence of compensatable steps.
    Executes forward steps in order. On any failure, executes
    compensation steps in reverse order.
    """
    def __init__(self, saga_id: UUID, steps: list[SagaStep]):
        self.saga_id = saga_id
        self.steps = steps
        self.completed: list[SagaStep] = []

    async def execute(self, context: ExecutionContext) -> SagaResult:
        for step in self.steps:
            try:
                result = await step.execute(context)
                self.completed.append(step)
                await self._persist_progress(step, result)
            except Exception as e:
                await self._compensate(context, e)
                return SagaResult(success=False, failed_step=step.name, error=str(e))
        return SagaResult(success=True)

    async def _compensate(self, context: ExecutionContext, original_error: Exception) -> None:
        for step in reversed(self.completed):
            try:
                await step.compensate(context)
            except Exception as comp_error:
                await self.event_bus.publish(RuntimeEvent(
                    event_type="saga.compensation_failed",
                    payload={"saga_id": str(self.saga_id), "step": step.name, "error": str(comp_error)}
                ))
```

### 8.8 Checkpoint-Based Recovery

The complete recovery flow after a crash:

```mermaid
sequenceDiagram
    participant RE as RecoveryEngine
    participant DB as PostgreSQL
    participant CS as CheckpointStore
    participant SCH as Scheduler
    participant COMP as CompensationEngine

    RE->>DB: find stuck executions (state=RUNNING, no heartbeat)
    DB->>RE: [execution_1, execution_2]
    RE->>DB: find tasks for execution_1 (state=RUNNING|QUEUED)
    DB->>RE: [task_a, task_b]
    RE->>CS: load_checkpoint(execution_1, task_a)
    CS->>RE: AgentCheckpoint(iteration=3, tokens=1200)
    RE->>SCH: enqueue_resume(task_a, checkpoint)
    RE->>DB: find tasks (state=QUEUED)
    RE->>SCH: re_enqueue(task_b)
    RE->>DB: update execution_1 state RUNNING
    note over RE: No work lost; task_a resumes from checkpoint
```

### 8.9 Failure Isolation

Failures in one execution must not propagate to others. The isolation mechanisms:

| Level | Mechanism | Scope |
|-------|-----------|-------|
| Process | Worker process per agent task | Task crash does not kill other tasks |
| Memory | cgroup memory limit per worker pod | OOM kill isolated to one pod |
| Network | Namespace-per-workspace | Rogue tool calls can't reach other workspaces |
| Database | Row-level locking | Task state mutations scoped to task_id |
| Event Bus | Per-execution NATS subject prefix | Event storms from one execution don't starve others |
| Cost | Per-execution budget enforcement | Budget exhaustion blocks one execution, not cluster |

---

## Chapter 9 — Execution Security

### 9.1 Purpose

Security is enforced at every layer: the API boundary (authentication), the service mesh (mTLS), the agent decision layer (Constitution PDP), the tool execution boundary (sandbox), and the audit layer (immutable log). No component trusts another without verification.

### 9.2 Authentication

All external API calls require a valid credential. Internal service-to-service calls use mTLS client certificates.

```
External authentication options:
  1. Bearer token (JWT)  — issued by /auth/token, signed HS256 or RS256
  2. API key             — hashed (Argon2id) in api_tokens table; Vault stores plaintext
  3. OIDC               — enterprise SSO (Azure AD, Okta, Google Workspace)

JWT payload:
{
  "sub":    "user:uuid",
  "tenant": "tenant:uuid",
  "roles":  ["engineer", "reviewer"],
  "scopes": ["executions:read", "executions:write", "workflows:read"],
  "iat":    1751000000,
  "exp":    1751003600,      // 1 hour
  "jti":    "uuid"           // prevents replay
}
```

### 9.3 Authorization

All runtime API endpoints enforce RBAC + ABAC at the middleware layer. No handler code checks permissions — the middleware does.

```python
class AuthorizationMiddleware:
    async def __call__(self, request: Request, call_next: RequestHandler) -> Response:
        principal = await self._extract_principal(request)
        required_permission = self._route_permission(request.method, request.url.path)

        # RBAC check
        if not self._has_role_permission(principal, required_permission):
            return JSONResponse({"error": "forbidden"}, status_code=403)

        # ABAC check (context-sensitive)
        resource_id = self._extract_resource_id(request)
        if resource_id and not await self._abac_check(principal, required_permission, resource_id):
            return JSONResponse({"error": "forbidden"}, status_code=403)

        # Inject principal into request state
        request.state.principal = principal
        return await call_next(request)

    def _route_permission(self, method: str, path: str) -> str:
        # Maps HTTP verb + path pattern to permission string
        return PERMISSION_MAP.get((method, self._path_pattern(path)), "unknown")
```

### 9.4 Sandbox Architecture

Tool execution (shell commands, file writes, test runners, build tools) runs inside a hardware-isolated sandbox.

```
Linux (production):
  Runtime: gVisor (runsc)
  Syscall filter: seccomp whitelist
  Network: isolated netns + egress proxy (allowed: package registries, git servers)
  Filesystem: OverlayFS copy-on-write layer, read-only below workspace
  Memory: cgroup v2 memory.max = 2GiB per sandbox
  CPU: cgroup v2 cpu.max = 200% (2 vCPU)

Windows (Desktop mode):
  Runtime: Windows Job Objects
  Process tree containment: Job Object with JOBOBJECT_BASIC_LIMIT_INFORMATION
  Network: Windows Filtering Platform (WFP) rules
  Filesystem: Windows Sandbox filesystem isolation
```

```python
class SandboxExecutor:
    async def execute(
        self,
        command: list[str],
        cwd: str,
        env: dict[str, str],
        stdin: bytes | None,
        timeout_s: int = 60,
    ) -> SandboxResult:
        proc = await asyncio.create_subprocess_exec(
            *self._wrap_command(command),   # adds gVisor runsc wrapper
            cwd=cwd,
            env=self._sanitize_env(env),
            stdin=asyncio.subprocess.PIPE if stdin else None,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE,
            limit=10 * 1024 * 1024,        # 10MB output limit
        )
        try:
            stdout, stderr = await asyncio.wait_for(
                proc.communicate(input=stdin), timeout=timeout_s
            )
            return SandboxResult(
                exit_code=proc.returncode,
                stdout=stdout.decode("utf-8", errors="replace"),
                stderr=stderr.decode("utf-8", errors="replace"),
                timed_out=False,
            )
        except asyncio.TimeoutError:
            proc.kill()
            return SandboxResult(exit_code=-1, stdout="", stderr="", timed_out=True)
```

### 9.5 Secret Management

Secrets (API keys, database credentials, signing keys) are never stored in plaintext in environment variables or config files.

```
Secret access flow:
  1. Service startup → authenticate to Vault using K8s ServiceAccount JWT
  2. Vault validates JWT against K8s API Server
  3. Vault issues short-lived lease (1h) for secret path
  4. Service reads secret → stores in memory only
  5. Background renewal: 75% of TTL → renew lease
  6. Service shutdown → revoke lease immediately

Agent secret access:
  Agents never call Vault directly.
  SecretResolver service mediates:
    agent requests: needs ANTHROPIC_API_KEY for task
    SecretResolver checks: agent has permission for this secret
    SecretResolver returns: ephemeral value (max TTL = task deadline)
    Agent uses in memory only — never writes to file/log
    Constitution rule CONST-005 scans all outputs for secret patterns
```

### 9.6 Permission Model

```python
class PermissionModel:
    """
    Hierarchical permission model.
    Agent permissions derive from employee role + task context.
    """
    TOOL_PERMISSIONS: dict[str, set[str]] = {
        # role → allowed tool names
        "engineer":  {"file_read", "file_write", "shell_exec", "git_diff", "git_commit_feature"},
        "lead":      {"file_read", "file_write", "shell_exec", "git_commit_main", "git_push", "pr_create"},
        "architect": {"file_read", "file_write", "adr_write", "kg_write"},
        "devops":    {"file_read", "file_write", "shell_exec", "docker_build", "k8s_apply", "deploy"},
        "security":  {"file_read", "shell_exec", "sast_scan", "sca_scan"},
        "cto":       {"*"},   # all tools
    }

    def agent_can_use_tool(self, agent_role: str, tool_name: str, context: ExecutionContext) -> bool:
        allowed = self.TOOL_PERMISSIONS.get(agent_role, set())
        if "*" in allowed:
            return True
        return tool_name in allowed

    def task_scope_check(self, tool_name: str, target_path: str, context: ExecutionContext) -> bool:
        # Enforce workspace scope — agents may only write within their workspace
        workspace = str(context.workspace_root)
        if "file_write" in tool_name:
            return target_path.startswith(workspace)
        return True
```

### 9.7 Execution Policies

Execution policies define per-tenant or per-project constraints:

```yaml
execution_policies:
  tenant_default:
    max_concurrent_executions: 10
    max_cost_per_execution_usd: 100.00
    max_tokens_per_execution: 2000000
    max_wall_time_hours: 24
    allowed_models:
      - claude-sonnet-4-6
      - claude-haiku-4-5-20251001
      - ollama/*
    blocked_tools:
      - k8s_delete_namespace
      - db_drop_table
    require_simulation_above_risk_score: 70
    require_approval_above_risk_score: 50
    audit_all_tool_calls: true

  enterprise_plan:
    max_concurrent_executions: 100
    max_cost_per_execution_usd: 1000.00
    allowed_models: ["*"]
    require_simulation_above_risk_score: 85
```

### 9.8 Resource Limits

```python
@dataclass
class ResourceLimits:
    # Compute
    max_worker_cpu_pct: float = 200.0       # 2 vCPU
    max_worker_memory_mb: int = 2048
    max_sandbox_cpu_pct: float = 100.0
    max_sandbox_memory_mb: int = 1024
    max_sandbox_disk_mb: int = 4096
    max_sandbox_network_bps: int = 10 * 1024 * 1024    # 10 Mbps

    # AI
    max_tokens_per_request: int = 200000
    max_output_tokens: int = 32000
    max_parallel_requests_per_worker: int = 4
    request_timeout_s: int = 120

    # Artifacts
    max_artifact_size_mb: int = 100
    max_artifacts_per_execution: int = 500
    max_workspace_size_mb: int = 2048
```

### 9.9 Audit Trail

Every security-relevant event writes to the `audit_events` table (from 2.0 Extension Ch.11) with hash-chain integrity. The runtime adds these additional audit event types:

| Event Type | Trigger | Recorded Fields |
|-----------|---------|----------------|
| `auth.login` | User authenticates | principal, ip, method, success |
| `auth.token_issued` | JWT issued | principal, scopes, exp |
| `auth.token_revoked` | Token revoked | token_jti, reason |
| `execution.created` | Execution started | execution_id, workflow_id, principal |
| `tool.allowed` | Tool call permitted by PDP | tool_name, agent, rule_matches |
| `tool.blocked` | Tool call blocked by Constitution | tool_name, agent, rule_id |
| `secret.accessed` | Agent retrieves secret | secret_path, agent, task_id |
| `secret.denied` | Secret access denied | secret_path, agent, reason |
| `sandbox.process_exec` | Command run in sandbox | command_hash, cwd, exit_code |
| `approval.requested` | Human gate created | gate_id, task_id, risk_score |
| `approval.granted` | Human approves | gate_id, approver_id, modified |
| `approval.rejected` | Human rejects | gate_id, approver_id, reason |
| `policy.violated` | Policy check failed | policy_key, actual_value, limit |

---

## Chapter 10 — Observability

### 10.1 Purpose

The Observability stack gives operations teams complete visibility into every execution: what ran, how long it took, what it cost, where it failed, and what the system's current health is. Observability is built into the runtime — it is not an afterthought. Every component emits structured logs, metrics, and traces without explicit instrumentation at the call site.

### 10.2 Three Pillars

```mermaid
flowchart LR
    R[Runtime] --> M[Metrics\nPrometheus]
    R --> L[Logs\nStructured JSON]
    R --> T[Traces\nOpenTelemetry]

    M --> G[Grafana Dashboards]
    L --> EL[Elasticsearch / Loki]
    T --> JA[Jaeger / Tempo]

    G --> AL[Alertmanager]
    EL --> AL
    AL --> NF[PagerDuty / Slack]
```

### 10.3 Metrics

All runtime services expose a `/metrics` endpoint (Prometheus exposition format). Key metric families:

```
# Executions
aistudio_executions_total{status="completed|failed|cancelled"} counter
aistudio_executions_active gauge
aistudio_execution_duration_seconds histogram (buckets: 60, 300, 900, 3600, 14400, 86400)
aistudio_execution_cost_usd histogram

# Tasks
aistudio_tasks_total{agent_type, status} counter
aistudio_tasks_active{agent_type} gauge
aistudio_task_duration_seconds{agent_type} histogram
aistudio_task_retries_total{agent_type} counter
aistudio_tasks_dead_lettered_total{agent_type} counter

# Queue
aistudio_queue_depth{agent_type} gauge
aistudio_queue_wait_seconds{agent_type} histogram
aistudio_worker_pool_size gauge
aistudio_worker_pool_active gauge

# AI Model
aistudio_model_requests_total{model, provider, status} counter
aistudio_model_tokens_total{model, provider, direction="input|output"} counter
aistudio_model_cost_usd_total{model, provider} counter
aistudio_model_latency_seconds{model, provider} histogram

# Tool calls
aistudio_tool_calls_total{tool_name, status="success|blocked|error"} counter
aistudio_tool_duration_seconds{tool_name} histogram

# Circuit breaker
aistudio_circuit_breaker_state{provider, state="open|closed|half_open"} gauge

# Approval gates
aistudio_approval_gates_pending gauge
aistudio_approval_gate_wait_seconds histogram

# Cluster
aistudio_cluster_workers_total gauge
aistudio_cluster_workers_healthy gauge
aistudio_cluster_leader_changes_total counter
```

### 10.4 Structured Logging

All log output is structured JSON, written to stdout (captured by Fluent Bit → Elasticsearch or Loki).

```python
class RuntimeLogger:
    def __init__(self, component: str, context: ExecutionContext | None = None):
        self._component = component
        self._context = context
        self._base_fields = {
            "service": "aistudio-runtime",
            "version": "3.5.1",
            "component": component,
        }
        if context:
            self._base_fields.update({
                "execution_id": str(context.execution_id),
                "trace_id": context.trace_id,
                "tenant_id": str(context.tenant_id),
            })

    def info(self, message: str, **extra) -> None:
        self._write("INFO", message, **extra)

    def _write(self, level: str, message: str, **extra) -> None:
        print(orjson.dumps({
            "@timestamp": datetime.utcnow().isoformat() + "Z",
            "level": level,
            "message": message,
            **self._base_fields,
            **extra,
        }).decode())
```

**Log output example:**

```json
{
  "@timestamp": "2026-06-27T14:32:01.342Z",
  "level": "INFO",
  "message": "Agent task completed",
  "service": "aistudio-runtime",
  "component": "agent_runtime",
  "execution_id": "550e8400-e29b-41d4-a716-446655440000",
  "task_id": "7d793037-a076-4f40-b0a4-b975cd3614c0",
  "agent_type": "BackendAgent",
  "employee_id": "3f2504e0-4f89-11d3-9a0c-0305e82c3301",
  "model": "claude-sonnet-4-6",
  "duration_ms": 24331,
  "input_tokens": 3842,
  "output_tokens": 1204,
  "cost_usd": "0.019802",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "tenant_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479"
}
```

### 10.5 Distributed Tracing

Every execution creates a root OpenTelemetry span. Agent tasks, tool calls, and LLM requests are child spans.

```python
class TracingInstrumentation:
    def trace_execution(self, context: ExecutionContext) -> contextmanager:
        return self.tracer.start_as_current_span(
            name="execution",
            attributes={
                "execution.id":    str(context.execution_id),
                "workflow.id":     str(context.workflow_id),
                "workflow.version": context.workflow_version,
                "factory.id":      str(context.factory_id) if context.factory_id else "",
                "tenant.id":       str(context.tenant_id),
            }
        )

    def trace_task(self, task: TaskNode, span_context: SpanContext) -> contextmanager:
        return self.tracer.start_as_current_span(
            name=f"task.{task.stage_id}",
            context=span_context,
            attributes={
                "task.id":        str(task.node_id),
                "task.stage":     task.stage_id,
                "agent.type":     task.agent_type,
                "task.attempt":   task.attempt,
            }
        )

    def trace_llm_call(self, model: str, provider: str) -> contextmanager:
        return self.tracer.start_as_current_span(
            name=f"llm.{provider}.complete",
            attributes={
                "llm.model":    model,
                "llm.provider": provider,
                "llm.system":   "aistudio",
            }
        )
```

**Trace example (Jaeger):**

```
execution [span, 41.2s]
  └── task.architecture [span, 12.4s]
      ├── llm.anthropic.complete [span, 8.2s]  model=claude-opus-4-8 tokens=4821
      ├── tool.kg_write [span, 0.3s]           adr_id=...
      └── tool.file_write [span, 0.1s]         path=docs/adr-001.md
  ├── task.backend [span, 18.7s]
      ├── llm.anthropic.complete [span, 14.1s] model=claude-sonnet-4-6 tokens=8203
      ├── tool.file_write [span, 0.2s]         path=src/api/payments.py
      └── tool.shell_exec [span, 4.2s]         cmd=pytest tests/unit/
  └── task.quality [span, 9.1s]
      ├── llm.anthropic.complete [span, 6.4s]  model=claude-sonnet-4-6
      └── tool.shell_exec [span, 2.5s]         cmd=pytest tests/integration/
```

### 10.6 Execution Timeline

The Desktop UI renders a live execution timeline using WebSocket event streaming:

```
WebSocket: ws://runtime/api/v3/executions/{id}/stream

Events consumed:
  task.started     → add swim lane for stage
  task.completed   → close swim lane, mark green
  task.failed      → close swim lane, mark red
  task.suspended   → show approval gate icon
  artifact.produced → show artifact badge
  budget.warning   → show cost alert banner
  model.fallback   → show provider switch indicator

Timeline layout:
  TIME:     0s        30s        60s        90s       120s
  ──────────────────────────────────────────────────────────
  Arch:     [████████████]
  Backend:                  [████████████████████]
  Frontend:                 [█████████████████]
  QA:                                         [████████]
  Security:                 [████]
  Release:                                             [██]
```

### 10.7 Dashboards

**Runtime Performance Dashboard (Grafana):**

```
Row 1: Execution Health
  Active executions | Completed last 1h | Failed last 1h | Success rate

Row 2: Queue & Workers
  Queue depth by agent type | Workers active/idle | Avg queue wait time

Row 3: AI Model Usage
  Requests/min by model | Cost/hour by model | p50/p95/p99 latency by model
  Token usage rate | Circuit breaker states

Row 4: Task Performance
  Tasks/min by agent type | p95 task duration | Retry rate | DLQ depth

Row 5: Approval Gates
  Gates pending | Gate wait time p50/p95 | Gate outcome (approve/reject/modify)
```

**Cost Dashboard (Grafana):**

```
Row 1: Org Cost
  MTD cost | Daily cost trend | Cost by project | Cost by model

Row 2: Per Execution Cost
  Avg cost per execution | Cost distribution histogram
  Budget utilization by factory | Budget alerts fired

Row 3: Efficiency
  Cost per story point | ROI by factory | Cost reduction proposals pending
```

**Health Dashboard (Grafana):**

```
Row 1: System Health
  Coordinator leader | Worker count | NATS cluster health
  PostgreSQL lag | Redis cluster health | MinIO availability

Row 2: Error Rates
  Execution error rate | Task retry rate | DLQ growth rate
  Circuit breaker open events | Constitution violations

Row 3: SLO Tracking
  P99 task completion time | Agent checkpoint coverage
  Approval gate availability | Audit log write success rate
```

### 10.8 Alerting Rules

```yaml
# Alertmanager rules
groups:
  - name: runtime_critical
    rules:
      - alert: ExecutionFailureRateCritical
        expr: rate(aistudio_executions_total{status="failed"}[5m]) > 0.1
        for: 5m
        labels: {severity: critical}
        annotations:
          summary: "Execution failure rate > 10% for 5 minutes"

      - alert: DLQDepthHigh
        expr: sum(aistudio_tasks_dead_lettered_total) > 10
        for: 2m
        labels: {severity: warning}

      - alert: ClusterLeaderMissing
        expr: aistudio_cluster_workers_healthy == 0
        for: 1m
        labels: {severity: critical}

      - alert: ModelCircuitBreakerOpen
        expr: aistudio_circuit_breaker_state{state="open"} == 1
        for: 0m
        labels: {severity: warning}

      - alert: ApprovalGatesStarved
        expr: aistudio_approval_gates_pending > 20
        for: 10m
        labels: {severity: warning}
        annotations:
          summary: "More than 20 approval gates pending for 10 minutes"
```

---


---

## Chapter 11 — Runtime APIs

### 11.1 Purpose

All runtime capabilities are exposed through a versioned, authenticated REST API. This chapter specifies every public endpoint, its request/response contracts, authentication requirements, rate limits, and example payloads. Internal service communication uses gRPC; the REST API is the external interface for the Desktop UI, Factory API, and third-party integrations.

### 11.2 API Conventions

```
Base URL:        https://runtime.aistudio.local/api/v3
Authentication:  Authorization: Bearer {jwt} or X-API-Key: {key}
Content-Type:    application/json
Accept:          application/json
Rate limits:     60 requests/minute per API key (burst: 200)
Versioning:      /api/v3 — breaking changes increment major version
Errors:          RFC 7807 Problem Details (application/problem+json)
Pagination:      ?page=1&per_page=50 (default 50, max 200)
```

**Error response format:**

```json
{
  "type":     "https://aistudio.local/errors/validation_error",
  "title":    "Validation Error",
  "status":   422,
  "detail":   "workflow_id is required",
  "instance": "/api/v3/executions",
  "errors": [
    {"field": "workflow_id", "message": "required field missing"}
  ]
}
```

### 11.3 Execution APIs

#### POST /api/v3/executions — Create Execution

Submit a workflow for execution.

```
POST /api/v3/executions
Authorization: Bearer {jwt}

Request:
{
  "workflow_id":      "feature-full",
  "workflow_version": "2.4.1",           // optional; defaults to active version
  "input": {
    "requirement_id": "uuid",
    "project_id":     "uuid",
    "priority":       80
  },
  "budget_usd":       50.00,             // optional; uses tenant default if omitted
  "time_budget_s":    86400,
  "factory_id":       "uuid",            // optional — links to product factory
  "metadata": {
    "triggered_by":  "sprint_planning",
    "sprint":        8
  }
}

Response 202 Accepted:
{
  "execution_id":  "uuid",
  "state":         "CREATED",
  "workflow_id":   "feature-full",
  "stream_url":    "wss://runtime.aistudio.local/api/v3/executions/uuid/stream",
  "cost_estimate": {
    "low_usd":      18.40,
    "expected_usd": 28.60,
    "high_usd":     42.90
  },
  "created_at":    "2026-06-27T14:30:00Z"
}
```

#### GET /api/v3/executions/{id} — Get Execution

```
Response 200:
{
  "execution_id":   "uuid",
  "state":          "RUNNING",
  "workflow_id":    "feature-full",
  "workflow_version": "2.4.1",
  "factory_id":     "uuid",
  "started_at":     "2026-06-27T14:30:01Z",
  "deadline_at":    "2026-06-28T14:30:01Z",
  "progress": {
    "total_stages":     6,
    "completed_stages": 3,
    "running_stages":   2,
    "pending_stages":   1,
    "pct_complete":     50
  },
  "cost": {
    "spent_usd":         14.23,
    "budget_usd":        50.00,
    "pct_consumed":      28.5,
    "forecast_usd":      29.80
  },
  "current_stages": ["backend", "frontend"],
  "checkpoints":    3,
  "approval_gates": {"pending": 0, "resolved": 1}
}
```

#### GET /api/v3/executions — List Executions

```
GET /api/v3/executions?state=RUNNING&factory_id=uuid&page=1&per_page=20

Response 200:
{
  "executions": [...],
  "pagination": {
    "page": 1, "per_page": 20, "total": 47, "pages": 3
  }
}
```

#### POST /api/v3/executions/{id}/pause

```
Request: {"reason": "Friday deployment freeze"}
Response 200: {"state": "PAUSED", "paused_at": "..."}
```

#### POST /api/v3/executions/{id}/resume

```
Request: {}
Response 200: {"state": "RUNNING", "resumed_at": "..."}
```

#### POST /api/v3/executions/{id}/cancel

```
Request: {"reason": "Requirements changed", "compensate": true}
Response 202: {
  "state": "CANCELLED",
  "compensation_triggered": true,
  "cancelled_at": "..."
}
```

#### GET /api/v3/executions/{id}/tasks — List Tasks

```
Response 200:
{
  "tasks": [
    {
      "task_id":     "uuid",
      "stage_id":    "backend",
      "status":      "COMPLETED",
      "agent_type":  "BackendAgent",
      "employee_id": "uuid",
      "attempt":     1,
      "started_at":  "2026-06-27T14:32:00Z",
      "completed_at":"2026-06-27T14:57:22Z",
      "duration_s":  1522,
      "cost_usd":    8.34,
      "tokens":      {"input": 12843, "output": 4201},
      "outputs": {
        "backend_code":  "artifact:uuid",
        "api_spec":      "artifact:uuid"
      }
    }
  ]
}
```

#### GET /api/v3/executions/{id}/artifacts — List Artifacts

```
Response 200:
{
  "artifacts": [
    {
      "artifact_id":    "uuid",
      "task_id":        "uuid",
      "artifact_type":  "source_code",
      "file_path":      "src/api/payments.py",
      "size_bytes":     4821,
      "content_hash":   "sha256:abc123...",
      "producer_agent": "BackendAgent",
      "created_at":     "2026-06-27T14:56:01Z",
      "download_url":   "/api/v3/artifacts/uuid/download"
    }
  ]
}
```

#### WS /api/v3/executions/{id}/stream — Real-Time Stream

```
WebSocket — events published as JSON text frames:

Frame: task.started
{"event_type":"agent.task.started","task_id":"uuid","stage_id":"backend",
 "agent_type":"BackendAgent","employee_id":"uuid","timestamp":"2026-06-27T14:32:00Z"}

Frame: task.completed
{"event_type":"agent.task.completed","task_id":"uuid","stage_id":"backend",
 "duration_s":1522,"cost_usd":8.34,"outputs":{"api_spec":"artifact:uuid"}}

Frame: approval.requested
{"event_type":"approval.gate.requested","gate_id":"uuid","task_id":"uuid",
 "risk_score":72,"reason":"Production deployment requires authorization",
 "simulation_summary":{"recommendation":"SAFE","duration_s":14}}

Frame: execution.completed
{"event_type":"execution.completed","execution_id":"uuid","total_cost_usd":27.84,
 "duration_s":4122,"artifacts_count":23}
```

### 11.4 Workflow APIs

#### POST /api/v3/workflows — Create Workflow

```
POST /api/v3/workflows
{
  "id":       "custom-feature",
  "version":  "1.0.0",
  "definition": "<YAML string>"
}

Response 201:
{
  "workflow_id":    "custom-feature",
  "version":        "1.0.0",
  "compiled_at":    "2026-06-27T14:00:00Z",
  "validation": {
    "valid":    true,
    "warnings": ["Stage 'frontend' has no approval gate — consider adding for production deployments"],
    "errors":   []
  },
  "resource_estimate": {
    "expected_tokens":  45000,
    "expected_cost_usd": 2.80,
    "expected_duration_s": 3600
  }
}
```

#### GET /api/v3/workflows/{id}/validate — Dry-Run Compile

```
POST /api/v3/workflows/{id}/validate
{
  "input": {"requirement_id": "uuid", "project_id": "uuid"}
}

Response 200:
{
  "valid": true,
  "task_graph": {
    "nodes": 6,
    "edges": 8,
    "parallel_groups": ["implementation", "qa"],
    "critical_path_stages": ["architecture", "backend", "quality", "release"],
    "critical_path_duration_s": 14400
  },
  "constitution_checks": {
    "passed": true,
    "rules_evaluated": 8,
    "warnings": []
  }
}
```

### 11.5 Agent APIs

#### GET /api/v3/agents — List Agent Types

```
Response 200:
{
  "agent_types": [
    {
      "agent_type":   "BackendAgent",
      "capabilities": ["file_write", "shell_exec", "git_commit_feature", "api_design"],
      "default_model": "claude-sonnet-4-6",
      "default_role":  "engineer",
      "active_count":  4
    }
  ]
}
```

#### GET /api/v3/agents/{execution_id}/{task_id}/logs — Agent Execution Log

```
Response 200:
{
  "task_id":    "uuid",
  "agent_type": "BackendAgent",
  "log_entries": [
    {
      "seq":       1,
      "timestamp": "2026-06-27T14:32:00.123Z",
      "type":      "message",
      "role":      "assistant",
      "content":   "I'll start by reviewing the architecture decisions...",
      "tokens":    124
    },
    {
      "seq":       2,
      "timestamp": "2026-06-27T14:32:01.441Z",
      "type":      "tool_call",
      "tool_name": "file_read",
      "input":     {"path": "docs/adr-001.md"},
      "output":    {"content": "..."},
      "duration_ms": 18
    }
  ]
}
```

### 11.6 Approval Gate APIs

#### GET /api/v3/approvals — List Pending Gates

```
Response 200:
{
  "approval_gates": [
    {
      "gate_id":       "uuid",
      "execution_id":  "uuid",
      "task_id":       "uuid",
      "stage_id":      "release",
      "risk_score":    72,
      "reason":        "Deploying to production",
      "simulation_id": "uuid",
      "simulation_summary": {
        "recommendation": "SAFE",
        "risk_score_after": 28,
        "duration_estimate_s": 14
      },
      "required_roles": ["lead", "cto"],
      "current_approvals": 1,
      "required_approvals": 2,
      "expires_at": "2026-06-28T14:30:00Z",
      "requested_at": "2026-06-27T14:30:00Z"
    }
  ]
}
```

#### POST /api/v3/approvals/{id}/decide — Resolve Approval Gate

```
POST /api/v3/approvals/{gate_id}/decide
{
  "decision":     "approve",          // "approve" | "reject" | "modify"
  "note":         "Simulation confirms safe. Proceed.",
  "modified_input": null              // if decision="modify": provide modified task input
}

Response 200:
{
  "gate_id":    "uuid",
  "decision":   "approve",
  "decided_by": "user:uuid",
  "decided_at": "2026-06-27T14:35:00Z",
  "execution_resuming": true
}
```

### 11.7 Administration APIs

#### GET /api/v3/admin/cluster — Cluster Status

```
Response 200:
{
  "coordinator": {
    "leader":          "coordinator-1",
    "standby_count":   1,
    "leader_since":    "2026-06-27T08:00:00Z"
  },
  "workers": {
    "total":    8,
    "healthy":  8,
    "capacity": 32,
    "active_tasks": 14
  },
  "queues": {
    "total_pending":   3,
    "by_agent_type": {
      "BackendAgent":  2,
      "QAAgent":       1
    }
  },
  "circuit_breakers": {
    "anthropic": "CLOSED",
    "openai":    "CLOSED",
    "ollama":    "CLOSED"
  },
  "dlq_depth": 0
}
```

#### POST /api/v3/admin/dlq/{id}/retry — Retry Dead-Lettered Task

```
Response 202: {"task_id": "uuid", "status": "QUEUED", "attempt": 4}
```

#### POST /api/v3/admin/workers/{id}/drain — Graceful Worker Drain

```
Response 202: {"worker_id": "uuid", "draining": true, "active_tasks": 2}
```

### 11.8 Monitoring APIs

#### GET /api/v3/monitoring/health — Runtime Health

```
Response 200:
{
  "status": "healthy",             // "healthy" | "degraded" | "critical"
  "checks": {
    "postgres":  {"status": "ok", "latency_ms": 2},
    "redis":     {"status": "ok", "latency_ms": 1},
    "nats":      {"status": "ok", "cluster_size": 3},
    "minio":     {"status": "ok"},
    "neo4j":     {"status": "ok"},
    "qdrant":    {"status": "ok"}
  },
  "uptime_s":  86400,
  "version":   "3.5.1"
}
```

#### GET /api/v3/monitoring/metrics/summary — Metrics Summary

```
Response 200:
{
  "period": "last_1h",
  "executions": {
    "started":    24,
    "completed":  21,
    "failed":     1,
    "cancelled":  2,
    "success_rate": 0.913
  },
  "tasks": {
    "completed":  384,
    "failed":     8,
    "retry_rate": 0.018,
    "avg_duration_s": 312
  },
  "cost": {
    "total_usd":      142.83,
    "by_model": {
      "claude-sonnet-4-6":      89.21,
      "claude-opus-4-8":        31.44,
      "claude-haiku-4-5":        2.18
    }
  },
  "approval_gates": {
    "created":  6,
    "approved": 5,
    "rejected": 1,
    "avg_wait_s": 284
  }
}
```

---

## Chapter 12 — Runtime Data Model

### 12.1 Overview

The runtime data model is the authoritative schema for all execution state, history, and configuration. PostgreSQL is the system of record. Redis is the cache and ephemeral state layer. NATS JetStream is the event log. Neo4j and Qdrant are used for knowledge and memory respectively.

### 12.2 Entity Relationship Diagram

```mermaid
erDiagram
    EXECUTIONS {
        uuid id PK
        text workflow_id FK
        text workflow_version
        uuid factory_id FK
        uuid project_id
        uuid tenant_id
        text state
        jsonb input
        jsonb output
        decimal budget_usd
        decimal spent_usd
        text trace_id
        timestamptz created_at
        timestamptz started_at
        timestamptz completed_at
        timestamptz deadline_at
    }

    TASK_NODES {
        uuid id PK
        uuid execution_id FK
        text stage_id
        text agent_type
        text status
        uuid assigned_employee FK
        integer attempt
        integer loop_iteration
        jsonb outputs
        text error
        timestamptz started_at
        timestamptz completed_at
    }

    AGENT_CHECKPOINTS {
        uuid id PK
        uuid execution_id FK
        uuid task_id FK
        integer iteration
        int token_count
        decimal cost_usd
        jsonb conversation
        jsonb tool_results
        jsonb partial_output
        bytes state_payload
        timestamptz created_at
        timestamptz expires_at
    }

    ARTIFACTS {
        uuid id PK
        uuid execution_id FK
        uuid task_id FK
        text artifact_type
        text file_path
        text content_hash
        bigint size_bytes
        text storage_key
        boolean is_promoted
        timestamptz created_at
    }

    APPROVAL_GATES {
        uuid id PK
        uuid execution_id FK
        uuid task_id FK
        text stage_id
        integer risk_score
        text reason
        uuid simulation_id FK
        text[] required_roles
        integer required_approvals
        integer current_approvals
        text status
        timestamptz expires_at
        timestamptz created_at
    }

    APPROVAL_DECISIONS {
        uuid id PK
        uuid gate_id FK
        uuid decided_by
        text decision
        text note
        jsonb modified_input
        timestamptz decided_at
    }

    WORKFLOW_VERSIONS {
        text workflow_id PK
        text version PK
        text definition_yaml
        jsonb compiled_json
        text checksum
        boolean is_active
        timestamptz compiled_at
    }

    EXECUTIONS ||--o{ TASK_NODES : "contains"
    TASK_NODES ||--o{ AGENT_CHECKPOINTS : "has"
    TASK_NODES ||--o{ ARTIFACTS : "produces"
    EXECUTIONS ||--o{ APPROVAL_GATES : "has"
    APPROVAL_GATES ||--o{ APPROVAL_DECISIONS : "receives"
    EXECUTIONS }o--|| WORKFLOW_VERSIONS : "runs"
```

### 12.3 Core Tables (DDL)

```sql
CREATE TABLE executions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workflow_id         TEXT NOT NULL,
    workflow_version    TEXT NOT NULL,
    factory_id          UUID REFERENCES product_factories(id),
    project_id          UUID NOT NULL,
    tenant_id           UUID NOT NULL,
    state               TEXT NOT NULL DEFAULT 'CREATED',
    input               JSONB NOT NULL DEFAULT '{}',
    output              JSONB,
    budget_usd          NUMERIC(12,4),
    spent_usd           NUMERIC(12,6) DEFAULT 0,
    token_budget        BIGINT,
    tokens_used         BIGINT DEFAULT 0,
    trace_id            TEXT,
    correlation_id      UUID,
    principal_id        TEXT NOT NULL,
    priority            INTEGER DEFAULT 50,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    started_at          TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,
    deadline_at         TIMESTAMPTZ NOT NULL,
    metadata            JSONB DEFAULT '{}'
);
CREATE INDEX idx_exec_factory   ON executions(factory_id, state);
CREATE INDEX idx_exec_tenant    ON executions(tenant_id, state, created_at DESC);
CREATE INDEX idx_exec_project   ON executions(project_id, created_at DESC);

CREATE TABLE task_nodes (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    execution_id        UUID NOT NULL REFERENCES executions(id) ON DELETE CASCADE,
    stage_id            TEXT NOT NULL,
    agent_type          TEXT NOT NULL,
    status              TEXT NOT NULL DEFAULT 'PENDING',
    parallel_group      TEXT,
    depends_on          UUID[] DEFAULT '{}',
    assigned_employee   UUID REFERENCES employees(id),
    assigned_worker     UUID,
    attempt             INTEGER DEFAULT 1,
    loop_iteration      INTEGER DEFAULT 0,
    priority            INTEGER DEFAULT 50,
    estimated_cost_usd  NUMERIC(10,6),
    actual_cost_usd     NUMERIC(10,6),
    input_tokens        INTEGER DEFAULT 0,
    output_tokens       INTEGER DEFAULT 0,
    outputs             JSONB DEFAULT '{}',
    error               TEXT,
    error_type          TEXT,
    scheduled_at        TIMESTAMPTZ,
    started_at          TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,
    UNIQUE(execution_id, stage_id, attempt)
);
CREATE INDEX idx_tasks_exec    ON task_nodes(execution_id, status);
CREATE INDEX idx_tasks_worker  ON task_nodes(assigned_worker, status);
CREATE INDEX idx_tasks_agent   ON task_nodes(agent_type, status, scheduled_at);

CREATE TABLE agent_checkpoints (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    execution_id    UUID NOT NULL,
    task_id         UUID NOT NULL REFERENCES task_nodes(id) ON DELETE CASCADE,
    iteration       INTEGER DEFAULT 0,
    token_count     INTEGER DEFAULT 0,
    cost_usd        NUMERIC(10,6) DEFAULT 0,
    conversation    JSONB NOT NULL DEFAULT '[]',
    tool_results    JSONB NOT NULL DEFAULT '[]',
    partial_output  JSONB NOT NULL DEFAULT '{}',
    state_payload   BYTEA,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    expires_at      TIMESTAMPTZ NOT NULL,
    UNIQUE(execution_id, task_id)
);

CREATE TABLE approval_gates (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    execution_id        UUID NOT NULL REFERENCES executions(id),
    task_id             UUID NOT NULL REFERENCES task_nodes(id),
    stage_id            TEXT NOT NULL,
    risk_score          INTEGER NOT NULL,
    reason              TEXT NOT NULL,
    simulation_id       UUID REFERENCES simulations(id),
    context_snapshot    JSONB NOT NULL DEFAULT '{}',
    required_roles      TEXT[] NOT NULL,
    required_approvals  INTEGER DEFAULT 1,
    current_approvals   INTEGER DEFAULT 0,
    status              TEXT NOT NULL DEFAULT 'pending',
    -- pending|approved|rejected|expired|cancelled
    expires_at          TIMESTAMPTZ NOT NULL,
    created_at          TIMESTAMPTZ DEFAULT NOW(),
    resolved_at         TIMESTAMPTZ
);
CREATE INDEX idx_gates_pending ON approval_gates(status, expires_at) WHERE status = 'pending';

CREATE TABLE approval_decisions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    gate_id         UUID NOT NULL REFERENCES approval_gates(id),
    decided_by      UUID NOT NULL REFERENCES users(id),
    decision        TEXT NOT NULL,      -- 'approve'|'reject'|'modify'
    note            TEXT,
    modified_input  JSONB,
    decided_at      TIMESTAMPTZ DEFAULT NOW()
);
-- Append-only. No UPDATE or DELETE.

CREATE TABLE execution_context_entries (
    id              BIGSERIAL PRIMARY KEY,
    execution_id    UUID NOT NULL,
    key             TEXT NOT NULL,
    value           JSONB NOT NULL,
    produced_by     TEXT NOT NULL,
    set_at          TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_ctx_exec ON execution_context_entries(execution_id, key);
```

### 12.4 JSON Examples

**Execution object:**

```json
{
  "id":               "550e8400-e29b-41d4-a716-446655440000",
  "workflow_id":      "feature-full",
  "workflow_version": "2.4.1",
  "factory_id":       "7d793037-a076-4f40-b0a4-b975cd3614c0",
  "project_id":       "3f2504e0-4f89-11d3-9a0c-0305e82c3301",
  "tenant_id":        "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "state":            "RUNNING",
  "input": {
    "requirement_id": "a3f7b2c1-1234-5678-abcd-ef0123456789",
    "project_id":     "3f2504e0-4f89-11d3-9a0c-0305e82c3301",
    "priority":       80
  },
  "budget_usd":       50.00,
  "spent_usd":        14.2384,
  "token_budget":     2000000,
  "tokens_used":      284231,
  "trace_id":         "4bf92f3577b34da6a3ce929d0e0e4736",
  "priority":         80,
  "created_at":       "2026-06-27T14:30:00Z",
  "started_at":       "2026-06-27T14:30:01Z",
  "deadline_at":      "2026-06-28T14:30:00Z"
}
```

**TaskNode object:**

```json
{
  "id":               "9a8b7c6d-5e4f-3a2b-1c0d-e9f8a7b6c5d4",
  "execution_id":     "550e8400-e29b-41d4-a716-446655440000",
  "stage_id":         "backend",
  "agent_type":       "BackendAgent",
  "status":           "COMPLETED",
  "parallel_group":   "implementation",
  "depends_on":       ["arch-task-uuid"],
  "assigned_employee": "emp-uuid-alex-backend-14",
  "attempt":          1,
  "loop_iteration":   0,
  "actual_cost_usd":  8.3402,
  "input_tokens":     12843,
  "output_tokens":    4201,
  "outputs": {
    "backend_code":   "artifact:uuid-1",
    "api_spec":       "artifact:uuid-2"
  },
  "started_at":   "2026-06-27T14:32:00Z",
  "completed_at": "2026-06-27T14:57:22Z"
}
```

**AgentCheckpoint object:**

```json
{
  "id":            "c1b2a3d4-e5f6-7890-abcd-ef1234567890",
  "execution_id":  "550e8400-e29b-41d4-a716-446655440000",
  "task_id":       "9a8b7c6d-5e4f-3a2b-1c0d-e9f8a7b6c5d4",
  "iteration":     0,
  "token_count":   7842,
  "cost_usd":      4.1201,
  "conversation": [
    {"role": "system",    "content": "You are a senior backend engineer..."},
    {"role": "user",      "content": "Implement the payment service module..."},
    {"role": "assistant", "content": "I'll start by reviewing the ADR..."},
    {"role": "tool",      "name": "file_read", "result": "...adr-001.md content..."}
  ],
  "tool_results": [
    {"tool_name": "file_read", "input": {"path": "docs/adr-001.md"}, "output": "..."}
  ],
  "partial_output": {
    "backend_code": "artifact:uuid-partial"
  },
  "created_at":  "2026-06-27T14:45:00Z",
  "expires_at":  "2026-06-28T15:30:00Z"
}
```

---

## Chapter 13 — High Availability

### 13.1 Purpose

AI Studio runtime must remain available and consistent through node failures, network partitions, rolling upgrades, and disaster scenarios. This chapter specifies the HA topology for every component, the failover procedures, data replication strategy, backup schedule, and disaster recovery targets.

### 13.2 HA Architecture

```mermaid
flowchart TB
    subgraph AZ1[Availability Zone 1]
        C1[Coordinator-1 LEADER]
        W1[Worker Node 1]
        W2[Worker Node 2]
        PG1[(PostgreSQL Primary)]
        RD1[(Redis Primary)]
        NJ1[(NATS Node 1)]
    end

    subgraph AZ2[Availability Zone 2]
        C2[Coordinator-2 STANDBY]
        W3[Worker Node 3]
        W4[Worker Node 4]
        PG2[(PostgreSQL Replica)]
        RD2[(Redis Replica)]
        NJ2[(NATS Node 2)]
    end

    subgraph AZ3[Availability Zone 3]
        W5[Worker Node 5]
        PG3[(PostgreSQL Replica)]
        RD3[(Redis Replica)]
        NJ3[(NATS Node 3)]
    end

    C1 <-->|Raft| C2
    PG1 -->|streaming replication| PG2
    PG1 -->|streaming replication| PG3
    RD1 -->|replication| RD2
    RD1 -->|replication| RD3
    NJ1 <-->|JetStream cluster| NJ2
    NJ1 <-->|JetStream cluster| NJ3
    NJ2 <-->|JetStream cluster| NJ3
```

### 13.3 RTO and RPO Targets

| Component | RTO | RPO | Strategy |
|-----------|-----|-----|---------|
| Coordinator | < 15s | 0 | Automatic failover via Raft leader election |
| Worker Node | < 30s | Task-level | Heartbeat expiry → task redistribution from checkpoint |
| PostgreSQL | < 30s | < 1s | Patroni auto-promotion; streaming replication |
| Redis | < 10s | 1s (AOF) | Sentinel auto-promotion |
| NATS JetStream | < 10s | 0 | Raft-based cluster consensus |
| MinIO | < 60s | 0 | Erasure coding (4+2); no failover needed for reads |
| Full region failure | < 4h | < 5min | DR restore from cross-region backup |

### 13.4 PostgreSQL HA with Patroni

```yaml
# patroni.yaml (applied to all PG nodes)
name: postgres-primary
scope: aistudio-pg
restapi:
  listen: 0.0.0.0:8008
  connect_address: ${POD_IP}:8008
etcd3:
  hosts: etcd-0:2379,etcd-1:2379,etcd-2:2379
bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576      # 1 MB max lag before refusing promotion
  pg_hba:
    - host replication replicator 0.0.0.0/0 md5
postgresql:
  parameters:
    wal_level: replica
    hot_standby: "on"
    wal_keep_size: "1GB"
    max_wal_senders: 10
    max_replication_slots: 10
    synchronous_commit: "on"
    synchronous_standby_names: "ANY 1 (postgres-replica-1, postgres-replica-2)"
```

### 13.5 Rolling Upgrade

Zero-downtime upgrade procedure:

```mermaid
sequenceDiagram
    participant OPS as Operator
    participant K8S as Kubernetes
    participant C1 as Coordinator-1
    participant C2 as Coordinator-2
    participant WN as Worker Nodes

    OPS->>K8S: kubectl set image deployment/coordinator image:v3.5.2
    K8S->>C2: update standby first (Coordinator-2)
    C2->>C2: drain → upgrade → restart
    C2->>C1: ready (standby, v3.5.2)
    OPS->>C1: kubectl annotate pod coordinator-1 lifecycle=pre_stop
    C1->>C1: stop accepting new executions
    C1->>C2: transfer leader (Raft election)
    C2->>C2: becomes LEADER (v3.5.2)
    K8S->>C1: upgrade Coordinator-1 to v3.5.2
    C1->>C2: rejoins as STANDBY
    note over C1,C2: Both coordinators now on v3.5.2

    K8S->>WN: rolling update worker nodes (2 at a time)
    note over WN: In-flight tasks checkpoint → continue on new workers
```

### 13.6 Backup Schedule

```
PostgreSQL:
  Full backup:   Daily at 02:00 UTC via pg_basebackup → S3 cross-region
  WAL archiving: Continuous → S3 (enables PITR to any second)
  Retention:     30 days full + 7 days WAL
  Verification:  Weekly restore test to isolated environment

MinIO:
  Policy:        Erasure coding (4 data + 2 parity) — no separate backup needed
  Cross-region:  MinIO bucket replication to DR region (RPO: near-zero)

NATS JetStream:
  Policy:        Stream data replicated across 3 nodes (Raft consensus)
  Snapshot:      Daily JetStream snapshot → S3

Redis:
  Policy:        AOF (appendonly yes, appendfsync everysec)
  Backup:        Daily BGSAVE → S3
  Retention:     7 days

Configuration:
  GitOps:        All runtime configuration in Git (source of truth)
  Vault:         Secrets backed up via Vault Enterprise Replication or export
```

### 13.7 Disaster Recovery

**DR playbook (full region loss):**

```
1. DETECTION (0–5 minutes)
   Health checks fail → PagerDuty alert → incident commander declares DR

2. DATA RESTORE (5–30 minutes)
   PostgreSQL: restore from latest S3 backup + WAL replay → < 5 min data loss
   Redis:      restore from latest RDB + AOF
   MinIO:      bucket replication already live in DR region → no restore needed
   NATS:       restore from latest JetStream snapshot

3. INFRASTRUCTURE PROVISION (30–60 minutes)
   Terraform: apply DR environment (pre-configured, tested quarterly)
   DNS:       update Coordinator endpoint to DR region
   Workers:   K8s auto-scales in DR region

4. EXECUTION RECOVERY (60–240 minutes)
   RecoveryEngine detects stuck executions
   Resumes from checkpoints (RPO: last checkpoint = max 60s of work lost)
   DLQ tasks manually reviewed

5. VERIFICATION
   Health dashboards green
   Test execution completes successfully
   Incident commander closes DR
```

---

## Chapter 14 — Performance

### 14.1 Purpose

AI Studio runtime handles large execution volumes, long-running agent tasks, and high token throughput. This chapter specifies caching strategies, context optimization techniques, streaming, parallelism patterns, batch execution, and resource scheduling that allow the runtime to meet performance targets under production load.

### 14.2 Performance Targets

| Metric | Target | Measurement |
|--------|--------|------------|
| Execution scheduling latency | < 100ms p99 | Time from API POST to first task queued |
| Task dispatch latency | < 200ms p99 | Time from task ready to agent started |
| LLM streaming first token | < 3s p95 | Model-dependent; measured after prompt delivery |
| Checkpoint write latency | < 50ms p99 | Redis write (sync) + PostgreSQL write (async) |
| Approval gate notification | < 1s | Event → WebSocket push to Desktop |
| Artifact store write | < 200ms p95 | For files < 10MB |
| Concurrent executions (single node) | 50 | At 4 workers, 4 tasks/worker, mixed load |
| Cluster concurrent executions | 500 | At 32 workers |
| Event bus throughput | 10,000 events/s | Single NATS cluster |

### 14.3 Context Caching

LLM providers with prompt caching (Anthropic Claude: caches prefixes > 1024 tokens) significantly reduce cost and latency for repeated agent patterns.

```python
class ContextCachingOptimizer:
    """
    Structures message lists to maximize provider-side prompt cache hits.
    The system prompt + coding standards + project context are stable across tasks.
    The task-specific instruction is the only variable tail.
    """
    def optimize_for_cache(self, messages: list[Message]) -> list[Message]:
        # Move large stable prefixes first with cache_control: {"type": "ephemeral"}
        # Anthropic caches up to 4 cache breakpoints per request
        system_parts = self._split_system_prompt(messages[0].content)
        return [
            Message(
                role="system",
                content=[
                    {"type": "text", "text": system_parts.stable, "cache_control": {"type": "ephemeral"}},
                    {"type": "text", "text": system_parts.task_specific}
                ]
            ),
            *messages[1:]
        ]
```

**Cache effectiveness metrics (tracked per model):**

```
Anthropic cache hit rate target: > 70% of input tokens served from cache
Estimated cost reduction: 50-90% on cached tokens (provider-specific discount)
```

### 14.4 Shared Context Optimization

The SharedContext serializes only deltas on each write (not the full snapshot):

```python
class DeltaSharedContext(SharedContext):
    """
    Maintains an in-memory write-through cache.
    Redis stores individual keys (hset), not full snapshots.
    Snapshot() assembles from Redis HGETALL — O(n) where n = keys in context.
    """
    async def get(self, key: str) -> Any | None:
        # L1: in-process dict (0ms)
        if key in self._cache:
            return self._cache[key]
        # L2: Redis (< 1ms LAN)
        raw = await self.redis.hget(self._redis_key, key)
        if raw:
            val = json.loads(raw)["value"]
            self._cache[key] = val
            return val
        return None
```

### 14.5 Streaming LLM Responses

Agents stream LLM responses rather than waiting for completion. This enables:
- Progressive artifact generation (file contents appear as the model writes them)
- Early tool-call detection (tool use is dispatched before the full response arrives)
- Budget monitoring (token count updated in real time)

```python
async def _call_model_streaming(
    self,
    messages: list[Message],
    tools: list[ToolSpec],
) -> AsyncIterator[StreamChunk]:
    async for chunk in self._gateway.complete_stream(
        model=self._context.model,
        messages=messages,
        tools=tools,
    ):
        # Track tokens in real time
        if chunk.usage:
            self._context.cost_tracker.record_tokens(
                input=chunk.usage.input_tokens,
                output=chunk.usage.output_tokens,
            )
        # Detect tool calls as they arrive
        if chunk.tool_use:
            asyncio.create_task(self._dispatch_tool(chunk.tool_use))
        yield chunk
```

### 14.6 Parallelism Patterns

The runtime uses three levels of parallelism:

**Level 1 — Stage-level parallelism** (DAG parallel groups)
Multiple stages execute concurrently when their dependency constraints allow.

**Level 2 — Tool-call parallelism** (within one agent)
When an agent decides to call multiple independent tools (e.g., read three files), tool calls are dispatched as `asyncio.gather`.

```python
class ParallelToolDispatcher:
    async def dispatch_parallel(self, tool_calls: list[ToolCall]) -> list[ToolResult]:
        independent = self._find_independent(tool_calls)
        dependent = [t for t in tool_calls if t not in independent]
        parallel_results = await asyncio.gather(
            *[self.tool_runtime.execute(tc.tool_name, tc.input, self._caller)
              for tc in independent],
            return_exceptions=True,
        )
        sequential_results = []
        for tc in dependent:
            sequential_results.append(
                await self.tool_runtime.execute(tc.tool_name, tc.input, self._caller)
            )
        return parallel_results + sequential_results
```

**Level 3 — Worker-level parallelism** (across the worker pool)
Each worker runs up to 4 agent tasks concurrently using asyncio.

### 14.7 Batch Execution

For tasks that can be batched (e.g., generating tests for 50 methods), the QAAgent uses a batch execution strategy:

```python
class BatchExecutor:
    BATCH_SIZE = 10             # methods per LLM call
    MAX_CONCURRENT_BATCHES = 4

    async def execute_batch(self, items: list[BatchItem], context: ExecutionContext) -> list[BatchResult]:
        batches = [items[i:i+self.BATCH_SIZE] for i in range(0, len(items), self.BATCH_SIZE)]
        semaphore = asyncio.Semaphore(self.MAX_CONCURRENT_BATCHES)

        async def run_batch(batch: list[BatchItem]) -> list[BatchResult]:
            async with semaphore:
                return await self._process_batch(batch, context)

        results = await asyncio.gather(*[run_batch(b) for b in batches])
        return [item for batch_result in results for item in batch_result]
```

### 14.8 Resource Optimization

**Token budget allocation:**

```
Execution budget: 2,000,000 tokens
  Architecture stage:  200,000  (10%)
  Implementation:      900,000  (45%)  — split across Backend/Frontend/DB
  QA:                  400,000  (20%)
  Security:            100,000   (5%)
  Documentation:       200,000  (10%)
  Release:             100,000   (5%)
  Buffer:              100,000   (5%)

Per-task enforcement:
  TokenBudgetMiddleware intercepts every LLM call
  Tracks cumulative usage against task allocation
  Emits budget.warning at 80% consumed
  Raises TokenBudgetExceeded at 100% → task fails → checkpoint saved
```

**Model selection for cost efficiency:**

```python
class CostAwareModelSelector:
    """
    Selects the cheapest model that meets the task's quality requirement.
    Quality requirement inferred from task criticality (risk_score).
    """
    def select(self, task: TaskNode, context: ExecutionContext) -> str:
        if task.risk_score >= 70:           # high-risk → always use best model
            return "claude-opus-4-8"
        if task.task_type in SIMPLE_TASK_TYPES:  # routing, summarization, doc gen
            return "claude-haiku-4-5-20251001"
        if context.spent_usd > context.budget_usd * 0.75:  # budget running low
            return "claude-haiku-4-5-20251001"   # downgrade to preserve budget
        return "claude-sonnet-4-6"          # default for most tasks
```

### 14.9 GPU Scheduling (Local Ollama)

For organizations using local GPU inference (Ollama), the runtime schedules GPU-bound tasks to avoid contention:

```python
class GPUScheduler:
    """
    Serializes concurrent Ollama requests when GPU memory is constrained.
    Multiple models loaded simultaneously causes swapping → worse than serialized.
    """
    def __init__(self, gpu_memory_gb: float, concurrency: int = 2):
        self._semaphore = asyncio.Semaphore(concurrency)
        self._current_model: str | None = None

    async def acquire(self, model: str) -> AsyncContextManager:
        async with self._semaphore:
            if self._current_model != model:
                await self._preload(model)      # ensure model loaded before call
            yield

    async def _preload(self, model: str) -> None:
        await self.ollama_client.pull(model)    # no-op if already loaded
        self._current_model = model
```

---

## Chapter 15 — Complete Runtime Example

### 15.1 Overview

This chapter traces the complete execution of a single product factory request: "Build a Vocal Coach application for Desktop, Web, Mobile and Cloud." It shows every layer of the runtime in action — from the API call to the deployed product — with event sequences, workflow diagrams, execution logs, API calls, and generated artifacts.

### 15.2 Factory Submission

The Executive Sponsor submits the product request via the Intake API.

**API call:**

```bash
curl -X POST https://runtime.aistudio.local/api/v3/factory \
  -H "Authorization: Bearer {jwt}" \
  -H "Content-Type: application/json" \
  -d '{
    "request": "Build a Vocal Coach application for Desktop, Web, Mobile and Cloud.",
    "budget_usd": 350.00,
    "priority": 80
  }'
```

**Response:**

```json
{
  "factory_id":   "fac-550e8400-vocal-coach",
  "state":        "INTAKE",
  "created_at":   "2026-06-27T09:00:00Z",
  "stream_url":   "wss://runtime.aistudio.local/api/v3/factory/fac-550e8400/stream",
  "next_step":    "Understanding — RequirementEngine processing"
}
```

### 15.3 Factory Pipeline Execution

```mermaid
sequenceDiagram
    participant ES as Executive Sponsor
    participant API as Intake API
    participant ARE as RequirementEngine
    participant BI as BusinessIntelligence
    participant CG as CompanyGenerator
    participant WC as WorkflowCompiler
    participant TS as TaskScheduler
    participant AR as AgentRuntime
    participant EB as EventBus
    participant AS as ArtifactStore

    ES->>API: POST /factory {"request": "Build a Vocal Coach app..."}
    API->>ARE: process("Build a Vocal Coach app...")
    ARE->>ARE: classify intent (consumer_app, audio_coaching)
    ARE->>ARE: load blueprint consumer_app_v2
    ARE->>ARE: generate ProductSpec + RequirementPackage
    ARE->>API: ProductSpec + cost_estimate ($198–$470)
    API->>BI: analyze(product_spec)
    BI->>BI: market research, competitor analysis
    BI->>API: GO recommendation, score=0.82
    API->>ES: Checkpoint 1 — Feasibility [approve/cancel]
    ES->>API: APPROVE
    API->>CG: generate_company(product_spec, budget=350)
    CG->>CG: spawn 12 departments, 31 employees
    CG->>API: ai_company_id="vocalcoach-inc"
    API->>WC: compile("feature-full", context)
    WC->>WC: validate DAG (6 stages, 0 cycles)
    WC->>TS: submit_task_graph(dag)
    note over TS: Sprint 1 of 12 begins
    TS->>AR: dispatch(task=architecture, context)
    AR->>AR: initialize ArchitectAgent + Dana-Architect-09
    AR->>AR: execute (ADR, module boundaries)
    AR->>AS: write artifact: adr-001.md
    AR->>EB: emit(task.architecture.completed)
    EB->>TS: advance DAG → unblock [backend, frontend, database]
    TS->>AR: dispatch(backend, frontend, database) concurrently
    note over AR: 3 agents running in parallel
    AR->>AS: write: src/api/*, src/services/*
    AR->>AS: write: src/frontend/*, src/mobile/*
    AR->>AS: write: migrations/001_initial.sql
    AR->>EB: all three emit completed
    EB->>TS: advance → QA, Security
    TS->>AR: dispatch(qa, security) concurrently
    AR->>AR: QAAgent: generate tests, run pytest
    AR->>AR: SecurityAgent: run semgrep, pip-audit
    AR->>AS: write: tests/*, scan_results.json
    AR->>EB: qa.completed, security.completed
    EB->>TS: advance → release
    TS->>API: approval gate: RELEASE (risk=68)
    API->>ES: Checkpoint 3 — Release [approve/reject]
    ES->>API: APPROVE
    TS->>AR: dispatch(release)
    AR->>AR: ReleaseAgent: build, SBOM, changelog, package
    AR->>AS: write: dist/vocal-coach-1.0.0.*, CHANGELOG.md, sbom.json
    AR->>EB: release.completed
    EB->>TS: execution.completed
```

### 15.4 Sprint 1 Task Graph State

After architecture completes and before parallel implementation begins:

```
TaskGraph state at t=12m:
  ┌─────────────────────────────────────────────────────┐
  │  [architecture]  COMPLETED  ✓  12m  $3.20           │
  │         │                                           │
  │  ┌──────┼──────────────┐                            │
  │  ▼      ▼              ▼                            │
  │ [backend] [frontend] [database]                     │
  │  RUNNING  RUNNING    RUNNING                        │
  │  (18m)    (16m)      (11m)                          │
  │    │        │          │                            │
  │    └────────┴──────────┘                            │
  │              ▼                                      │
  │    [quality] [security]  PENDING                    │
  │         │                                           │
  │         ▼                                           │
  │    [release]             PENDING                    │
  └─────────────────────────────────────────────────────┘
```

### 15.5 Execution Log Stream (Selected Events)

```
[09:00:00] execution.created          execution_id=550e8400 workflow=feature-full
[09:00:01] execution.compiling        stages=6 parallel_groups=2
[09:00:02] execution.scheduled        task_graph_nodes=6 task_graph_edges=8
[09:00:02] agent.task.started         stage=architecture agent=ArchitectAgent employee=Dana-Architect-09
[09:00:02] agent.initialized          model=claude-opus-4-8 prompts=system+adr_generation+coding_standards
[09:02:41] tool.called                tool=file_write path=docs/adr-001.md size=4821B
[09:02:41] artifact.produced          type=adr_document path=docs/adr-001.md hash=sha256:a1b2c3...
[09:11:58] tool.called                tool=kg_write node_type=ADR id=adr-001
[09:12:04] approval.requested         gate_id=gate-arch-001 stage=architecture risk_score=42
[09:12:04] [WebSocket push to Desktop: Approval gate: Architecture ADR review]
[09:15:22] approval.granted           gate_id=gate-arch-001 decided_by=user:dana-lead
[09:15:22] agent.task.completed       stage=architecture duration_s=920 cost_usd=3.20 tokens=21841
[09:15:23] agent.task.started         stage=backend   agent=BackendAgent   employee=Alex-Backend-14
[09:15:23] agent.task.started         stage=frontend  agent=FrontendAgent  employee=Sam-Frontend-03
[09:15:23] agent.task.started         stage=database  agent=DatabaseAgent  employee=Riley-DB-02
[09:16:00] agent.task.checkpointed    stage=backend   checkpoint_id=chk-001 tokens_so_far=2841
[09:34:07] tool.called                tool=shell_exec cmd=pytest tests/unit/ exit_code=0
[09:44:51] agent.task.completed       stage=backend   duration_s=1768 cost_usd=8.34
[09:51:03] agent.task.completed       stage=frontend  duration_s=2140 cost_usd=7.92
[09:26:44] agent.task.completed       stage=database  duration_s=681  cost_usd=1.84
[09:51:04] agent.task.started         stage=quality   agent=QAAgent       employee=Sam-QA-05
[09:51:04] agent.task.started         stage=security  agent=SecurityAgent employee=Riley-Security-02
[09:55:12] tool.called                tool=shell_exec cmd=pytest tests/ exit_code=0 pass_rate=0.94
[09:57:01] tool.called                tool=sast_scan  findings=0
[10:02:44] agent.task.completed       stage=quality   cost_usd=4.21 test_pass_rate=0.94
[10:01:29] agent.task.completed       stage=security  cost_usd=1.83 critical_findings=0
[10:02:45] simulation.started         type=deployment risk_before=68
[10:03:02] simulation.completed       recommendation=SAFE risk_after=22 duration_s=14
[10:03:02] approval.requested         gate_id=gate-rel-001 stage=release risk_score=68 simulation=SAFE
[10:06:14] approval.granted           gate_id=gate-rel-001 decided_by=user:cto-001
[10:06:15] agent.task.started         stage=release   agent=ReleaseAgent  employee=Morgan-DevOps-03
[10:11:33] artifact.produced          type=build_output path=dist/vocal-coach-1.0.0-win.exe
[10:11:34] artifact.produced          type=sbom        path=sbom.json     components=187
[10:11:35] artifact.produced          type=changelog   path=CHANGELOG.md
[10:11:38] agent.task.completed       stage=release    duration_s=323 cost_usd=1.22
[10:11:38] execution.completed        total_cost_usd=28.56 duration_s=4298 artifacts=34
[10:11:38] factory.state_transition   state=DEPLOYMENT
```

### 15.6 Generated Artifacts

By execution completion, the Artifact Store contains:

| Artifact | Path | Produced By | Size |
|---------|------|------------|------|
| Architecture Decision Record | docs/adr-001.md | ArchitectAgent | 4.8 KB |
| Backend API implementation | src/api/*.py (14 files) | BackendAgent | 38.2 KB |
| Service layer | src/services/*.py (8 files) | BackendAgent | 22.1 KB |
| Database migration | migrations/001_initial.sql | DatabaseAgent | 3.4 KB |
| Frontend components | src/frontend/*.tsx (22 files) | FrontendAgent | 61.3 KB |
| React Native mobile | src/mobile/*.tsx (18 files) | FrontendAgent | 44.7 KB |
| Unit tests | tests/unit/*.py (41 files) | QAAgent | 28.4 KB |
| Integration tests | tests/integration/*.py (12 files) | QAAgent | 9.8 KB |
| SAST scan results | reports/sast_results.json | SecurityAgent | 1.2 KB |
| SCA scan results | reports/sca_results.json | SecurityAgent | 2.1 KB |
| Windows build | dist/vocal-coach-1.0.0-win.exe | ReleaseAgent | 48.3 MB |
| macOS build | dist/vocal-coach-1.0.0-macos.dmg | ReleaseAgent | 52.1 MB |
| Web bundle | dist/vocal-coach-1.0.0-web.zip | ReleaseAgent | 3.8 MB |
| CycloneDX SBOM | sbom.json | ReleaseAgent | 18.4 KB |
| Changelog | CHANGELOG.md | ReleaseAgent | 2.1 KB |
| Execution report | reports/execution_summary.json | Runtime | 4.3 KB |

### 15.7 Execution Summary Report

```json
{
  "execution_id":    "550e8400-e29b-41d4-a716-446655440000",
  "factory_id":      "fac-550e8400-vocal-coach",
  "workflow":        "feature-full v2.4.1",
  "status":          "COMPLETED",
  "started_at":      "2026-06-27T09:00:01Z",
  "completed_at":    "2026-06-27T10:11:38Z",
  "wall_time_s":     4297,
  "cost": {
    "total_usd":     28.56,
    "budget_usd":    50.00,
    "pct_used":      57.1,
    "by_stage": {
      "architecture": 3.20,
      "backend":      8.34,
      "frontend":     7.92,
      "database":     1.84,
      "quality":      4.21,
      "security":     1.83,
      "release":      1.22
    },
    "by_model": {
      "claude-opus-4-8":   7.14,
      "claude-sonnet-4-6": 20.01,
      "claude-haiku-4-5":  1.41
    }
  },
  "tokens": {
    "input":  284231,
    "output": 92847,
    "total":  377078
  },
  "quality": {
    "test_pass_rate":   0.94,
    "test_count":       284,
    "sast_findings":    0,
    "sca_findings":     0,
    "human_approvals":  2,
    "human_rejections": 0,
    "compensation_triggered": false,
    "retries":          0,
    "checkpoints_used": 3
  },
  "artifacts": {
    "count":      34,
    "total_bytes": 52943827,
    "promoted":   34
  },
  "employees": {
    "assigned": [
      {"name": "Dana-Architect-09", "stage": "architecture", "cost_usd": 3.20},
      {"name": "Alex-Backend-14",   "stage": "backend",      "cost_usd": 8.34},
      {"name": "Sam-Frontend-03",   "stage": "frontend",     "cost_usd": 7.92},
      {"name": "Riley-DB-02",       "stage": "database",     "cost_usd": 1.84},
      {"name": "Sam-QA-05",         "stage": "quality",      "cost_usd": 4.21},
      {"name": "Riley-Security-02", "stage": "security",     "cost_usd": 1.83},
      {"name": "Morgan-DevOps-03",  "stage": "release",      "cost_usd": 1.22}
    ]
  },
  "experience_graph_updated": true,
  "blueprint_updated":        true,
  "factory_state":            "DEPLOYMENT"
}
```

### 15.8 Post-Execution: Experience Graph Update

After execution completes, the Continuous Evolution engine updates the organizational knowledge base:

```
ExperienceRecorder.record(execution_id="550e8400...")

  Created nodes:
    Problem:    "WebAudio API latency target: 50ms"
    Decision:   "Use AudioWorklet with Web Worker fallback"
    Solution:   "browser_audio_pipeline_v1 (implemented in frontend/audio/)"
    Outcome:    success=true, test_pass_rate=0.94, cost_usd=28.56
    Lesson:     "AudioWorklet + ScriptProcessorNode fallback achieves <50ms on 98% of browsers"

  Relationships:
    Problem -[:TRIGGERED]-> Decision
    Decision -[:PRODUCED]-> Solution
    Solution -[:RESULTED_IN]-> Outcome
    Outcome -[:GENERATED]-> Lesson
    Lesson -[:APPLIES_TO]-> (:Module {name: 'audio_processing'})

  Blueprint consumer_app_v2 updated:
    lessons_learned: +1 (WebAudio API latency target strategy)
    quality_score: 0.92 → 0.923 (weighted average with this execution's 0.94)

  Company DNA updated:
    'audio_processing_approach': AudioWorklet (confidence 0.61, projects=1)
    'test_coverage_target': 0.80 (confirmed by 4 consecutive projects)
```

### 15.9 Configuration Reference

```yaml
# runtime-config.yaml — complete reference

runtime:
  version: "3.5.1"
  environment: production      # production | staging | development

coordinator:
  replicas: 2
  election_ttl_s: 15
  heartbeat_interval_s: 5

workers:
  min_replicas: 2
  max_replicas: 32
  tasks_per_worker: 4
  checkpoint_interval_s: 60
  startup_timeout_s: 30

scheduler:
  dlq_max_attempts: 3
  delayed_poll_interval_ms: 100
  recurring_check_interval_s: 30

workflow_engine:
  compilation_timeout_s: 30
  max_stages_per_workflow: 50
  max_parallel_group_size: 20
  max_loop_iterations_default: 10

agent_runtime:
  context_window_reserve_tokens: 1000   # reserve for system overhead
  tool_call_timeout_s: 120
  max_tool_calls_per_task: 200
  max_conversation_turns: 50

sandbox:
  provider: gvisor                      # gvisor | docker | none (dev only)
  memory_limit_mb: 1024
  cpu_limit_pct: 100
  disk_limit_mb: 4096
  network_egress: allowed               # allowed | proxied | blocked
  network_proxy: "http://sandbox-egress-proxy:3128"

event_bus:
  nats_url: "nats://nats-0:4222,nats-1:4222,nats-2:4222"
  max_reconnects: -1                    # infinite reconnect attempts
  reconnect_wait_s: 2

storage:
  postgres_dsn: "postgresql://aistudio:${PG_PASS}@pg-primary:5432/aistudio"
  redis_url: "redis://redis-cluster:6379"
  minio_endpoint: "minio:9000"
  minio_bucket: "aistudio-artifacts"

secret_management:
  provider: vault                       # vault | infisical | env (dev only)
  vault_address: "http://vault:8200"
  vault_auth_method: kubernetes
  vault_role: "aistudio-runtime"

observability:
  metrics_port: 9090
  trace_endpoint: "http://tempo:4317"
  log_level: info                       # debug | info | warn | error
  log_format: json

circuit_breakers:
  anthropic:
    failure_threshold: 5
    recovery_timeout_s: 60
  openai:
    failure_threshold: 5
    recovery_timeout_s: 60
  ollama:
    failure_threshold: 3
    recovery_timeout_s: 30

autoscaler:
  scale_up_threshold: 0.80
  scale_down_threshold: 0.20
  cooldown_s: 120
```

---

## Appendix A — Component Interaction Summary

```mermaid
flowchart TD
    subgraph External
        UI[Desktop UI / API Clients]
    end

    subgraph Core_Runtime
        IA[Intake API<br/>Ch.11]
        WE[Workflow Engine<br/>Ch.2]
        SM[State Machine<br/>Ch.6]
        TS[Task Scheduler<br/>Ch.4]
        AR[Agent Runtime<br/>Ch.3]
        EB[Event Bus<br/>Ch.5]
        DE[Distributed Exec<br/>Ch.7]
    end

    subgraph Cross_Cutting
        SEC[Security<br/>Ch.9]
        REL[Reliability<br/>Ch.8]
        OBS[Observability<br/>Ch.10]
    end

    subgraph Persistence
        PG[(PostgreSQL)]
        RD[(Redis)]
        MN[(MinIO)]
    end

    UI --> IA
    IA --> WE
    WE --> SM
    SM --> TS
    TS --> AR
    AR --> EB
    EB --> SM
    DE --> TS
    DE --> AR

    SEC -.->|intercepts| AR
    SEC -.->|intercepts| IA
    REL -.->|wraps| AR
    REL -.->|wraps| TS
    OBS -.->|instruments| IA
    OBS -.->|instruments| AR
    OBS -.->|instruments| EB

    AR --> PG
    AR --> RD
    AR --> MN
    TS --> PG
    SM --> PG
```

---

*Document version: 3.5.1-DRAFT*  
*Extends: AI-STUDIO-3.5-UNIVERSAL-PRODUCT-FACTORY.md*  
*All 15 chapters complete.*  
*Last updated: 2026-06-27*
