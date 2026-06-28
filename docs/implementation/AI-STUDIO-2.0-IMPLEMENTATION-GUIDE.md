# AI Studio 2.0 — Implementation Guide
# Engineering Specification for Software Engineers and AI Agents

**Document Class:** Implementation Engineering Specification  
**Platform Target:** AI Studio 2.0.0  
**Predecessor:** AI Studio 1.5.5 (see AI-STUDIO-1.5.5-REVERSE-ENGINEERING-SPEC.md)  
**Source Documents:** AI-STUDIO-2.0-ARCHITECTURE.md · AI-STUDIO-2.0-EXTENSION.md · AI-STUDIO-3.0-VISION.md · AI-STUDIO-3.5-UNIVERSAL-PRODUCT-FACTORY.md · AI-STUDIO-3.5.1-ENTERPRISE-EXECUTION-ENGINE.md  
**Specification Date:** 2026-06-27  
**Status:** Engineering Specification — Ready for Implementation

---

## Engineering Contract

This document is the single implementation source of truth for AI Studio 2.0. Every feature, module, table, API endpoint, event, and test described here must be built exactly as specified. Engineers and AI agents building this platform must not seek additional architectural clarification — all decisions are made here.

**This document answers for every feature:**
- What module owns it and which repository contains it
- Which database tables are required (exact DDL)
- Which REST endpoints expose it (request/response shapes)
- Which NATS subjects carry its events (payload schemas)
- Which agents participate and in what order
- Which tests verify it
- How it starts, shuts down, and recovers from failure
- How it is monitored and alerted on

---

## Table of Contents

- Part 1 — Repository Layout
- Part 2 — Module Dependency Map
- Part 3 — Runtime Lifecycle
- Part 4 — Database Specification
- Part 5 — Coordinator Implementation
- Part 6 — Worker Runtime
- Part 7 — Event Bus
- Part 8 — Memory System
- Part 9 — Brain Implementation
- Part 10 — API Specification
- Part 11 — Desktop Application
- Part 12 — Web Platform
- Part 13 — Mobile Application
- Part 14 — Marketplace
- Part 15 — Deployment
- Part 16 — Monitoring
- Part 17 — Testing Strategy
- Part 18 — Migration Guide
- Part 19 — Performance Targets
- Part 20 — Engineering Roadmap

---

# PART 1: REPOSITORY LAYOUT

## 1.1 Strategy

AI Studio 2.0 uses a **polyrepo** strategy: one Git repository per deployable service, independently versioned, built, and deployed. All Python services use Python 3.12+. Every service produces a Docker image pushed to `registry.aistudio.internal`.

## 1.2 Repositories

### `coordinator` — Central Orchestration Service

**Purpose:** Primary HTTP gateway. Handles authentication, authorization, project management, human approval gates, WebSocket real-time updates. Delegates AI execution to orchestrator via NATS.

**Technology Stack:** Python 3.12, FastAPI 0.115+, asyncpg, NATS.py JetStream, aioredis, python-jose, Alembic

**Public API:** REST on port 8090, WebSocket on `/ws/*`, OpenAPI at `/docs`

**Internal Modules:**
```
coordinator/
├── api/v2/
│   ├── projects.py         # Project CRUD
│   ├── tasks.py            # Task submission + management
│   ├── approvals.py        # Human gate CRUD
│   ├── memories.py         # Memory CRUD + hybrid search
│   ├── agents.py           # Agent status reporting
│   ├── plugins.py          # Plugin install/remove/list
│   ├── models.py           # AI provider management
│   ├── users.py            # User + membership management
│   └── ws.py               # WebSocket real-time endpoints
├── api/internal/
│   ├── health.py           # GET /healthz/live, /healthz/ready
│   └── metrics.py          # GET /metrics (Prometheus)
├── auth/
│   ├── jwt.py              # JWT decode + claims extraction
│   ├── oidc.py             # OIDC/OAuth2 provider integration
│   ├── api_key.py          # bcrypt hash + validation
│   └── rbac.py             # Role permission enforcement
├── services/
│   ├── project_service.py  # Project business logic
│   ├── task_service.py     # Task creation, plan import
│   ├── approval_service.py # Gate lifecycle management
│   ├── memory_service.py   # Memory write + retrieval
│   └── notification_service.py  # WebSocket fan-out
├── nats/
│   ├── client.py           # JetStream wrapper
│   ├── publishers.py       # Outbound event publishing
│   └── subscribers.py      # Consume agent results, approval events
├── db/
│   ├── engine.py           # Async SQLAlchemy + asyncpg pool
│   ├── models.py           # ORM models for all coordinator tables
│   └── migrations/         # Alembic versions/
├── config.py               # Pydantic Settings (env vars)
├── app.py                  # FastAPI application factory + lifespan
└── main.py                 # uvicorn entrypoint
```

**Build Pipeline:**
```
pytest tests/ --cov=coordinator --cov-fail-under=80
mypy coordinator/ --strict
ruff check coordinator/
docker build -t coordinator:${GIT_SHA} .
docker push registry.aistudio.internal/coordinator:${GIT_SHA}
```

---

### `orchestrator` — Task Scheduling and Agent Dispatch

**Purpose:** Dependency resolution, priority scheduling, retry/backoff, checkpointing, escalation. Stateless — all state in PostgreSQL. Runs as single instance with Redis-based leader election.

**Technology Stack:** Python 3.12, asyncio, asyncpg (direct, no ORM), NATS.py, Redis, APScheduler

**Internal Modules:**
```
orchestrator/
├── scheduler/
│   ├── dependency_resolver.py  # DAG traversal, readiness checks
│   ├── priority_queue.py       # Priority-weighted FIFO
│   ├── agent_dispatcher.py     # Publish to agent.{type}.inbox
│   ├── checkpoint_manager.py   # Save/restore task checkpoints
│   └── cancellation.py         # Cascade cancellation through task tree
├── recovery/
│   ├── stall_detector.py       # Heartbeat timeout per task
│   ├── retry_engine.py         # Exponential backoff + attempt tracking
│   └── escalation.py           # Create approval gate on max retry
├── health/
│   ├── tick.py                 # 30s periodic orchestration tick
│   └── workflow_health.py      # 5-check structural health monitor
├── nats/
│   ├── subscriber.py           # Consume tasks.created, agent.*.outbox
│   └── publisher.py            # Publish agent dispatch messages
├── db/queries.py               # Raw asyncpg queries (hot path — no ORM)
├── config.py
└── main.py
```

---

### `agent-runtime` — AI Agent Execution Engine

**Purpose:** Hosts the agent pool. Each agent claims tasks from NATS, assembles context from knowledge-engine, calls AI models, executes tools via tool-runtime, and emits results. Scales horizontally via NATS queue groups.

**Technology Stack:** Python 3.12 asyncio, NATS.py, httpx, anthropic SDK, openai SDK, google-generativeai, qdrant-client, tree-sitter, GitPython

**Internal Modules:**
```
agent-runtime/
├── agents/
│   ├── base.py             # BaseAgent ABC (async execute, checkpoint, resume)
│   ├── architect.py        # ArchitectAgent — design decisions, ADRs
│   ├── planner.py          # PlannerAgent — SDLC decomposition
│   ├── backend.py          # BackendAgent — APIs, services, business logic
│   ├── frontend.py         # FrontendAgent — UI components, routing
│   ├── database.py         # DatabaseAgent — schema design, migrations
│   ├── security.py         # SecurityAgent — SAST, dependency audit
│   ├── qa.py               # QAAgent — test generation, coverage
│   ├── reviewer.py         # ReviewerAgent — code review
│   ├── documentation.py    # DocumentationAgent — docstrings, README
│   ├── release.py          # ReleaseAgent — versioning, changelog, SBOM
│   ├── devops.py           # DevOpsAgent — CI config, Docker, K8s
│   ├── refactoring.py      # RefactoringAgent — complexity reduction
│   ├── monitoring.py       # MonitoringAgent — alert rules, SLOs
│   ├── learning.py         # LearningAgent — outcome collection (singleton)
│   └── registry.py         # AGENT_REGISTRY: str -> Type[BaseAgent]
├── context/
│   ├── assembler.py        # Build AgentContext from memory + graph + code
│   ├── memory_fetcher.py   # Hybrid memory retrieval (FTS + vector + graph)
│   └── token_budgeter.py   # Context window budget management
├── tools/
│   ├── base.py             # BaseTool ABC
│   ├── file_reader.py      # Read project files
│   ├── file_writer.py      # Write files (commits to workspace)
│   ├── shell_executor.py   # Delegates to tool-runtime sandboxed shell
│   ├── git_ops.py          # Git operations (commit, diff, log)
│   ├── build_runner.py     # Maven/npm/pip/cargo build invocation
│   ├── test_runner.py      # pytest/jest/junit/go test invocation
│   ├── http_client.py      # Outbound HTTP calls (research, APIs)
│   ├── db_client.py        # Database query execution
│   ├── ast_parser.py       # tree-sitter code parsing
│   ├── search.py           # Semantic + FTS search via knowledge-engine
│   └── registry.py         # TOOL_REGISTRY: str -> Type[BaseTool]
├── sandbox/
│   ├── allocator.py        # Sandbox lifecycle manager
│   ├── linux.py            # Linux namespaces + seccomp
│   └── windows.py          # Windows Job Object isolation
├── providers/
│   ├── base.py             # ModelProvider ABC
│   ├── claude.py           # Anthropic SDK adapter
│   ├── openai_provider.py  # OpenAI SDK adapter
│   ├── gemini.py           # Google Generative AI adapter
│   ├── ollama.py           # Ollama HTTP REST adapter (local)
│   └── router.py           # ModelRouter — affinity, fallback, cost
├── checkpoint/
│   ├── saver.py            # Write checkpoint to tasks.checkpoint_data
│   └── restorer.py         # Load checkpoint and resume agent execution
├── nats/
│   ├── consumer.py         # Claim from agent.{type}.inbox queue groups
│   └── publisher.py        # Publish to agent.{type}.outbox
├── metrics.py              # Prometheus counters, histograms, gauges
├── config.py
└── main.py
```

**Scaling:** Multiple replicas. HPA triggers on NATS consumer metric `nats_consumer_pending_messages > 50`.

---

### `tool-runtime` — Sandboxed Tool Execution Service

**Purpose:** Executes shell commands, builds, tests, and browser automation in OS-level isolation. DaemonSet in Kubernetes (one pod per node). Not externally accessible.

**Technology Stack:** Python 3.12, FastAPI (internal), Linux namespaces / gVisor (K8s prod), Windows Job Objects (desktop)

**Internal Modules:**
```
tool-runtime/
├── sandbox/
│   ├── linux_namespace.py  # unshare + seccomp + network restriction
│   ├── windows_job.py      # Job Object + restricted token + process limits
│   └── gvisor.py           # gVisor runsc wrapper for K8s production
├── tools/
│   ├── shell.py            # Arbitrary command execution
│   ├── build.py            # Maven/npm/cargo/pip/gradle/make
│   ├── test.py             # pytest/jest/junit/go test/cargo test
│   ├── git.py              # Git clone/commit/push/diff/log
│   └── browser.py          # Playwright headless chromium
├── api/
│   ├── execute.py          # POST /execute — run command in sandbox
│   ├── sandbox.py          # POST /sandbox, DELETE /sandbox/{id}
│   └── logs.py             # GET /sandbox/{id}/logs (Server-Sent Events)
├── config.py
└── main.py
```

**Resource Limits per Sandbox:**
- RAM: 2 GB
- CPU: 2 cores  
- Disk: 10 GB (CoW overlay, isolated)
- Network: blocked by default; allow-list: package registries, Git hosts
- Timeout: configurable per tool type (build: 600s, test: 300s, shell: 60s)

---

### `knowledge-engine` — Memory and Knowledge Graph Service

**Purpose:** Unified API over Qdrant (vector), Neo4j/Kuzu (graph), and PostgreSQL (metadata). Provides memory write/retrieval, code ingestion, and hybrid search for agent context assembly.

**Technology Stack:** Python 3.12, FastAPI, qdrant-client, neo4j-driver / kuzu Python API, asyncpg, sentence-transformers or Anthropic Embeddings

**Internal Modules:**
```
knowledge-engine/
├── memory/
│   ├── writer.py           # Write to PostgreSQL + embed to Qdrant
│   ├── retriever.py        # Hybrid search (PostgreSQL FTS + Qdrant + graph)
│   ├── compactor.py        # Background job: compress old memories
│   └── embedder.py         # Text -> dense vector embedding
├── graph/
│   ├── ingestion.py        # AST parsing -> Neo4j/Kuzu upsert pipeline
│   ├── queries.py          # Named graph queries (hotspots, dead code, callers)
│   └── analyzer.py         # Weekly analysis reports for ArchitectAgent
├── api/
│   ├── memory.py           # GET /memory, POST /memory, DELETE /memory/{id}
│   ├── search.py           # POST /search (hybrid: q, type, limit, project_id)
│   ├── graph.py            # POST /graph/query (raw Cypher/Kuzu query)
│   └── ingest.py           # POST /ingest/repository (trigger repo scan)
├── config.py
└── main.py
```

**Mode Selection:**
- Laptop: Kuzu embedded (file-based, zero config)
- Team/Enterprise: Neo4j cluster

---

### `brain` — Experience Graph and Decision Intelligence

**Purpose:** Records execution outcomes, detects patterns, provides pattern-based recommendations to agents. Background LearningAgent pipeline.

**Technology Stack:** Python 3.12, FastAPI, Neo4j (experience graph), Qdrant (pattern embeddings), asyncpg

**Internal Modules:**
```
brain/
├── experience/
│   ├── recorder.py         # POST /record-outcome handler
│   ├── graph.py            # Neo4j experience graph schema + writes
│   └── retriever.py        # Find similar past executions
├── patterns/
│   ├── detector.py         # Extract patterns from outcome corpus
│   └── scorer.py           # Confidence scoring per pattern
├── decisions/
│   ├── engine.py           # Given task context -> recommendations
│   └── explainer.py        # Pattern -> human-readable rationale
├── learning/
│   └── pipeline.py         # Background: consolidation + pattern refresh
├── api/
│   ├── recommend.py        # POST /recommend
│   ├── record.py           # POST /record-outcome
│   └── explain.py          # GET /explain/{decision_id}
├── config.py
└── main.py
```

---

### `auth` — Authentication Service

**Purpose:** OIDC/OAuth2, LDAP integration, API key management, JWT issuance. All auth decisions delegated here from coordinator.

**Endpoints:**
```
POST   /auth/token              API key -> short-lived JWT
POST   /auth/refresh            Refresh token -> new JWT
POST   /auth/oidc/callback      OIDC redirect handler
POST   /auth/ldap/login         LDAP credential validation
POST   /auth/api-key/create     Generate + hash new API key
DELETE /auth/api-key/{id}       Revoke API key
GET    /auth/me                 Current user from JWT claims
POST   /auth/logout             Revoke session in Redis + JWT JTI
GET    /auth/.well-known/jwks   Public key for JWT verification
```

---

### `sdk` — Plugin Development Kit

**Purpose:** Python package `aistudio-sdk`. Zero mandatory runtime dependencies. Entry point for all third-party plugin development.

**Install:** `pip install aistudio-sdk`

**Contents:**
```
sdk/aistudio_sdk/
├── agent/
│   ├── base.py             # BaseAgent ABC with execute/checkpoint/resume
│   └── context.py          # AgentContext, AgentResult, CheckpointState
├── tool/
│   ├── base.py             # BaseTool ABC
│   └── result.py           # ToolResult dataclass
├── plugin/
│   ├── manifest.py         # PluginManifest, plugin.yaml loader/validator
│   └── registry.py         # Plugin registration utilities
├── memory/
│   └── types.py            # MEMORY_TYPE constants enum
├── events/
│   └── schema.py           # All NATS event payload dataclasses
└── testing/
    ├── mock_context.py     # MockAgentContext for unit tests
    └── fixtures.py         # pytest fixtures (mock NATS, mock DB, mock tools)
```

---

### `deployment` — Infrastructure as Code

```
deployment/
├── docker/
│   ├── compose.laptop.yaml     # Full stack: all services on one machine
│   ├── compose.team.yaml       # Team server mode
│   └── compose.dev.yaml        # Dev with hot reload + debug ports
├── k8s/helm/
│   ├── coordinator/            # Helm chart: deployment, service, ingress, HPA
│   ├── orchestrator/           # Helm chart: deployment, leader election
│   ├── agent-runtime/          # Helm chart: deployment, HPA on NATS lag
│   ├── tool-runtime/           # Helm chart: DaemonSet
│   ├── knowledge-engine/       # Helm chart: deployment
│   └── brain/                  # Helm chart: deployment
├── terraform/
│   ├── aws/                    # EKS + RDS + ElastiCache + MSK + S3
│   ├── gcp/                    # GKE + Cloud SQL + Memorystore + GCS
│   └── azure/                  # AKS + Azure Database + Redis Cache + Blob
└── scripts/
    ├── install.sh              # One-command laptop install (Docker + compose)
    ├── upgrade.sh              # Rolling upgrade with health-check gating
    └── backup.sh               # Dump all PG schemas + Qdrant collections
```

## 1.3 Repository Dependency Matrix

```
desktop          -> coordinator (REST + WebSocket)
webapp           -> coordinator (REST + WebSocket)
coordinator      -> auth (REST), knowledge-engine (REST), postgres, redis, nats
orchestrator     -> coordinator.postgres (shared schema), nats
agent-runtime    -> tool-runtime (REST), knowledge-engine (REST), brain (REST),
                    nats, postgres (execution log writes), ai-provider APIs
knowledge-engine -> qdrant, neo4j/kuzu, postgres (direct)
brain            -> neo4j, qdrant, postgres (direct, brain.* schema)
auth             -> postgres (direct, auth.* schema)
tool-runtime     -> OS sandbox APIs, filesystem (NO network to other services)
sdk              -> stdlib only
```

No circular dependencies. Architecture test in each CI pipeline validates the dependency matrix.

---

# PART 2: MODULE DEPENDENCY MAP

## 2.1 Layered Architecture

```
Layer 0 — Human Interface
  desktop  --(REST/WS)--> coordinator
  webapp   --(REST/WS)--> coordinator

Layer 1 — Coordination
  coordinator --(REST)--------> auth
  coordinator --(REST)--------> knowledge-engine
  coordinator --(SQL)---------> postgres (coordinator schema)
  coordinator --(Redis)-------> session cache
  coordinator --(NATS)--------> publish: tasks.created, tasks.updated

Layer 2 — Orchestration
  orchestrator --(SQL)---------> postgres (coordinator schema, task state)
  orchestrator --(NATS)--------> subscribe: tasks.created
  orchestrator --(NATS)--------> publish: agent.{type}.inbox dispatch

Layer 3 — Agent Execution
  agent-runtime --(REST)-------> tool-runtime (sandboxed tools)
  agent-runtime --(REST)-------> knowledge-engine (context assembly)
  agent-runtime --(REST)-------> brain (pattern recommendations)
  agent-runtime --(HTTPS)------> AI provider APIs (Anthropic, OpenAI, Ollama)
  agent-runtime --(SQL)--------> postgres (agent_executions, tool_calls writes)
  agent-runtime --(NATS)-------> subscribe: agent.{type}.inbox
  agent-runtime --(NATS)-------> publish: agent.{type}.outbox

Layer 4 — Knowledge
  knowledge-engine --> qdrant (vector store)
  knowledge-engine --> neo4j / kuzu (knowledge graph)
  knowledge-engine --> postgres (memory metadata: memory.* schema)

Layer 5 — Intelligence
  brain --> neo4j (experience graph)
  brain --> qdrant (pattern similarity)
  brain --> postgres (outcome metadata: brain.* schema)

Layer 6 — Infrastructure (external, not owned by any service)
  postgres 15   Primary operational store (coordinator, auth, memory, brain schemas)
  redis 7       Session cache, distributed locks, pub/sub
  qdrant        Vector search (project_memories, code_chunks, agent_prompts)
  neo4j/kuzu    Knowledge graph + experience graph
  nats          Async event bus (JetStream)
  minio/s3      Artifact storage (generated code, test results, logs)
  ollama        Local model inference (laptop mode fallback)
```

## 2.2 Architectural Rules (Enforced via CI Architecture Tests)

```
RULE 1: No Layer N service may directly call a Layer N-2 or higher service.
        coordinator -> agent-runtime: FORBIDDEN
        coordinator -> tool-runtime: FORBIDDEN

RULE 2: No Python import between services.
        All cross-service: REST API or NATS messages.
        Shared types: only via aistudio-sdk package.

RULE 3: agent-runtime never calls coordinator.
        Results published to NATS; coordinator subscribes.

RULE 4: tool-runtime has no outbound network except allow-listed:
        package registries (pypi.org, npmjs.com, repo1.maven.org)
        git hosts (github.com, gitlab.com, internal git)
        No calls to any AI Studio service.

RULE 5: brain writes only to brain.* PostgreSQL schema.
        brain does NOT read or write coordinator.* or memory.* schemas.

RULE 6: sdk package zero runtime dependencies.
        No requests, no httpx, no database drivers in aistudio-sdk.
```

## 2.3 Extension Points

```
New Agent Type:
  1. class MyAgent(BaseAgent) in new file under agent-runtime/agents/
  2. Add to AGENT_REGISTRY in agent-runtime/agents/registry.py
  3. Create corresponding NATS consumer in nats/consumer.py
  4. Plugin agents: define in plugin.yaml, auto-registered at plugin install

New Tool Type:
  1. class MyTool(BaseTool) in new file under agent-runtime/tools/
  2. Add to TOOL_REGISTRY in agent-runtime/tools/registry.py
  3. Plugin tools: define in plugin.yaml

New AI Provider:
  1. class MyProvider(ModelProvider) in agent-runtime/providers/
  2. Add to PROVIDER_REGISTRY in agent-runtime/providers/router.py
  3. Add validation in startup credential check

New Workflow Template:
  1. Define YAML workflow DAG in marketplace repository
  2. Install via POST /plugins/install (downloads to orchestrator/workflows/)
  3. Reference by workflow_id in task creation

New Memory Type:
  1. Add constant to aistudio_sdk/memory/types.py
  2. Update MEMORY_TYPES list in knowledge-engine/memory/writer.py
  3. No schema migration required (memory_type is TEXT field)
```

## 2.4 Data Flow — Task Submission to Completion

```
1. User types: "Add Stripe payment with webhook support"
   -> POST /api/v2/projects/{id}/tasks  [coordinator]
   -> INSERT tasks (status=pending)
   -> NATS publish: tasks.created

2. orchestrator receives tasks.created
   -> dispatch PlannerAgent: NATS publish agent.planner.inbox

3. PlannerAgent (agent-runtime) receives task
   -> knowledge-engine: assemble context (project memories + code graph)
   -> brain: get decomposition recommendations from experience
   -> AI model: generate 8-subtask plan
   -> NATS publish: agent.planner.outbox (subtask list)

4. orchestrator receives planner result
   -> INSERT 8 subtasks into tasks table
   -> run dependency_resolver: identify independent tasks
   -> dispatch ArchitectAgent and BackendAgent in parallel

5. ArchitectAgent + BackendAgent execute concurrently
   -> Each: knowledge-engine context assembly
   -> Each: tool-runtime tool calls (file I/O, AST parse)
   -> Each: AI model calls
   -> Each: tool-runtime write output files
   -> Each: NATS publish outbox result

6. orchestrator receives agent results
   -> UPDATE task status = completed
   -> unblock downstream tasks (QA, Security)
   -> create approval gate if diff exceeds risk threshold

7. coordinator receives tasks.updated via NATS
   -> WebSocket broadcast to connected Desktop clients
   -> approval gate notification if pending human decision

8. Human approves commit gate in Desktop
   -> POST /api/v2/approvals/{id}/approve  [coordinator]
   -> orchestrator dispatches commit + CI trigger

9. CI passes -> task status = completed, merge
   -> NATS event: tasks.completed
   -> LearningAgent records outcome to brain
   -> knowledge-engine: update code graph with changed files
```

---

# PART 3: RUNTIME LIFECYCLE

## 3.1 Startup Order (Laptop Mode — docker compose)

```
[postgres + redis + qdrant + nats + kuzu] -- infrastructure (no deps)
         |
         v (all healthy)
    [auth service]   [knowledge-engine]   [brain]
         |                   |               |
         +-------------------+---------------+
                             |
                             v (all healthy)
                       [coordinator]         <- runs Alembic migrations
                             |
                  +----------+----------+
                  |                     |
           [orchestrator]         [agent-runtime]
                                        |
                                  [tool-runtime]
```

Expected startup (laptop, SSD, cold): 45–90 seconds.
Expected startup (laptop, warm restart): 15–30 seconds.

## 3.2 Coordinator Startup

```python
# coordinator/app.py

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Phase 1: Infrastructure connections
    await db_engine.connect()         # asyncpg pool, min=5 max=20 connections
    await redis_client.connect()      # aioredis pool, max=10
    await nats_client.connect()       # NATS JetStream, url from config

    # Phase 2: Schema migration
    await run_alembic_upgrade("head") # Idempotent; noop if no pending migrations

    # Phase 3: NATS stream setup
    await ensure_streams_exist()      # Create streams defined in streams.yaml

    # Phase 4: Event subscriptions
    await subscribe_agent_outboxes()  # agent.*.outbox -> update task status
    await subscribe_approval_events() # approval.* -> WebSocket broadcast

    # Phase 5: Background tasks
    asyncio.create_task(approval_expiry_loop())    # Expire timed-out gates
    asyncio.create_task(notification_broadcaster()) # WS fan-out worker
    asyncio.create_task(memory_expiry_cleaner())   # Purge expired memories

    # Phase 6: Mark ready
    app.state.ready = True
    yield

    # Shutdown (SIGTERM received)
    app.state.ready = False
    await nats_client.drain()     # Flush outbound, ACK pending messages
    await redis_client.close()
    await db_engine.dispose()
```

## 3.3 Agent Runtime Startup

```python
# agent-runtime/main.py

async def startup():
    # 1. Connect all infrastructure
    await nats.connect(settings.nats_url, max_reconnect_attempts=-1)
    await db_pool.init(settings.pg_dsn, min_size=3, max_size=10)
    await qdrant_client.init(settings.qdrant_url)

    # 2. Validate AI provider credentials
    for key in settings.enabled_providers:
        provider = PROVIDER_REGISTRY[key]
        ok = await provider.validate_credentials()
        if not ok:
            log.warning("Provider %s unavailable — will not route tasks to it", key)

    # 3. Recover stale tasks from pre-crash state
    await recover_stale_tasks(instance_id=settings.instance_id)

    # 4. Subscribe all agents to their NATS inboxes (queue groups)
    for agent_key, agent_cls in AGENT_REGISTRY.items():
        agent = agent_cls()
        await nats.subscribe(
            subject=f"agent.{agent.agent_type}.inbox",
            queue=f"pool.{agent.agent_type}",   # Load-balanced across replicas
            cb=agent.dispatch
        )

    # 5. Heartbeat
    asyncio.create_task(heartbeat_loop(interval_s=30))

    log.info("Agent runtime ready: %d agents, providers: %s",
             len(AGENT_REGISTRY), settings.enabled_providers)
```

## 3.4 Graceful Shutdown

```python
async def graceful_shutdown():
    # Unsubscribe from NATS (stop accepting new tasks)
    for sub in nats_subscriptions:
        await sub.unsubscribe()

    # Signal running agents to checkpoint at next opportunity
    shutdown_event.set()

    # Wait up to 60s for in-flight agents to checkpoint
    try:
        await asyncio.wait_for(
            asyncio.gather(*[a.wait_checkpoint() for a in active_agents]),
            timeout=60.0
        )
    except asyncio.TimeoutError:
        log.warning("Checkpoint timeout — some tasks will resume from last saved state")

    # Drain NATS: flush pending publishes, ACK pending messages
    await nats.drain()

    # Close DB pool cleanly
    await db_pool.close()
    log.info("Agent runtime shutdown complete")
```

## 3.5 Crash Recovery

```python
# agent-runtime/recovery.py

async def recover_stale_tasks(instance_id: str):
    stale = await db_pool.fetch("""
        SELECT id, assigned_agent, checkpoint_data, attempt_count, max_attempts
        FROM coordinator.tasks
        WHERE assigned_instance = $1
          AND status = 'running'
          AND updated_at < NOW() - INTERVAL '5 minutes'
    """, instance_id)

    for task in stale:
        if task['checkpoint_data']:
            # Resume from checkpoint
            await nats.publish(
                f"agent.{task['assigned_agent']}.inbox",
                TaskResumeMsg(task_id=task['id'],
                              checkpoint=task['checkpoint_data']).encode()
            )
        elif task['attempt_count'] < task['max_attempts']:
            # Reset to ready for re-dispatch
            await db_pool.execute("""
                UPDATE coordinator.tasks
                SET status = 'ready',
                    assigned_instance = NULL,
                    updated_at = NOW()
                WHERE id = $1
            """, task['id'])
        else:
            # Exhausted — escalate to human
            await db_pool.execute(
                "UPDATE coordinator.tasks SET status='failed' WHERE id=$1",
                task['id']
            )
            await create_exhaustion_approval_gate(task['id'])
```

## 3.6 Health Check Contract

All services implement:

```
GET /healthz/live
  -> 200 {"status": "alive"}  (always, while process running)
  Used by: Kubernetes liveness probe

GET /healthz/ready
  -> 200 {"status": "ready", "checks": {...}, "version": "2.0.0", "uptime_seconds": N}
  -> 503 {"status": "starting"|"degraded", "checks": {...}}
  Used by: Kubernetes readiness probe, load balancer health check

Example response:
{
  "status": "ready",
  "checks": {
    "postgres": "ok",
    "nats": "ok",
    "redis": "ok",
    "qdrant": "ok"
  },
  "version": "2.0.0",
  "uptime_seconds": 3600,
  "instance_id": "agent-runtime-7d9f4b-xyz"
}
```

Kubernetes probe configuration (all services):
```yaml
livenessProbe:
  httpGet: {path: /healthz/live, port: 8090}
  initialDelaySeconds: 10
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet: {path: /healthz/ready, port: 8090}
  initialDelaySeconds: 15
  periodSeconds: 5
  failureThreshold: 3

startupProbe:
  httpGet: {path: /healthz/live, port: 8090}
  failureThreshold: 30
  periodSeconds: 5
```

---

# PART 4: DATABASE SPECIFICATION

## 4.1 Database Assignment

| Service | Database | Schema |
|---------|----------|--------|
| coordinator | PostgreSQL 15 | coordinator |
| orchestrator | PostgreSQL 15 | coordinator (shared read/write) |
| agent-runtime | PostgreSQL 15 | coordinator (execution log writes) |
| auth | PostgreSQL 15 | auth |
| knowledge-engine | PostgreSQL 15 + Qdrant + Neo4j/Kuzu | memory |
| brain | PostgreSQL 15 + Neo4j + Qdrant | brain |

All schemas on the same PostgreSQL cluster. Each service connects with its own user having schema-scoped privileges.

## 4.2 coordinator Schema — Full DDL

```sql
CREATE EXTENSION IF NOT EXISTS "pgcrypto";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";
CREATE SCHEMA IF NOT EXISTS coordinator;

-- ==== USERS ====
CREATE TABLE coordinator.users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           TEXT UNIQUE NOT NULL,
    display_name    TEXT NOT NULL DEFAULT '',
    provider        TEXT NOT NULL DEFAULT 'local',
    provider_id     TEXT,
    role            TEXT NOT NULL DEFAULT 'developer',
    api_key_hash    TEXT,
    api_key_prefix  TEXT,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT users_role_check CHECK (role IN ('admin','lead','developer','viewer'))
);
CREATE UNIQUE INDEX uq_users_provider ON coordinator.users(provider, provider_id)
    WHERE provider_id IS NOT NULL;
CREATE INDEX idx_users_api_key ON coordinator.users(api_key_prefix)
    WHERE api_key_prefix IS NOT NULL;

-- ==== PROJECTS ====
CREATE TABLE coordinator.projects (
    id                   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                 TEXT NOT NULL,
    slug                 TEXT UNIQUE NOT NULL,
    description          TEXT NOT NULL DEFAULT '',
    owner_id             UUID NOT NULL REFERENCES coordinator.users(id),
    repo_url             TEXT,
    repo_path            TEXT,
    default_branch       TEXT NOT NULL DEFAULT 'main',
    default_ai_provider  TEXT NOT NULL DEFAULT 'claude',
    default_ai_model     TEXT,
    unattended_mode      BOOLEAN NOT NULL DEFAULT FALSE,
    max_concurrent_agents INTEGER NOT NULL DEFAULT 4,
    settings             JSONB NOT NULL DEFAULT '{}',
    status               TEXT NOT NULL DEFAULT 'active',
    created_at           TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at           TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT projects_status_check CHECK (status IN ('active','archived','deleted'))
);
CREATE INDEX idx_projects_owner ON coordinator.projects(owner_id);
CREATE INDEX idx_projects_slug ON coordinator.projects(slug);

-- ==== PROJECT MEMBERS ====
CREATE TABLE coordinator.project_members (
    project_id  UUID NOT NULL REFERENCES coordinator.projects(id) ON DELETE CASCADE,
    user_id     UUID NOT NULL REFERENCES coordinator.users(id) ON DELETE CASCADE,
    role        TEXT NOT NULL DEFAULT 'developer',
    granted_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (project_id, user_id),
    CONSTRAINT members_role_check CHECK (role IN ('lead','developer','viewer'))
);

-- ==== TASKS ====
CREATE TABLE coordinator.tasks (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id          UUID NOT NULL REFERENCES coordinator.projects(id) ON DELETE CASCADE,
    parent_id           UUID REFERENCES coordinator.tasks(id),
    type                TEXT NOT NULL,
    title               TEXT NOT NULL,
    description         TEXT NOT NULL DEFAULT '',
    acceptance_criteria TEXT NOT NULL DEFAULT '',
    status              TEXT NOT NULL DEFAULT 'pending',
    priority            INTEGER NOT NULL DEFAULT 50 CHECK (priority BETWEEN 0 AND 100),
    assigned_agent      TEXT,
    assigned_instance   TEXT,
    estimated_tokens    INTEGER,
    estimated_seconds   INTEGER,
    actual_tokens       INTEGER,
    actual_seconds      INTEGER,
    cost_usd            NUMERIC(12, 6),
    attempt_count       INTEGER NOT NULL DEFAULT 0,
    max_attempts        INTEGER NOT NULL DEFAULT 3,
    checkpoint_data     JSONB,
    result_data         JSONB,
    error_data          JSONB,
    depends_on          UUID[] NOT NULL DEFAULT '{}',
    is_critical_path    BOOLEAN NOT NULL DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    started_at          TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,
    deadline_at         TIMESTAMPTZ,
    metadata            JSONB NOT NULL DEFAULT '{}',
    CONSTRAINT tasks_status_check CHECK (status IN (
        'pending','planning','ready','running','blocked',
        'awaiting_approval','completed','failed','cancelled'
    )),
    CONSTRAINT tasks_type_check CHECK (type IN (
        'feature','bug','refactor','test','deploy','security',
        'documentation','architecture','release','research'
    ))
);

CREATE INDEX idx_tasks_project_status ON coordinator.tasks(project_id, status);
CREATE INDEX idx_tasks_parent ON coordinator.tasks(parent_id)
    WHERE parent_id IS NOT NULL;
CREATE INDEX idx_tasks_depends ON coordinator.tasks USING GIN(depends_on);
CREATE INDEX idx_tasks_agent_ready ON coordinator.tasks(assigned_agent, priority DESC, created_at ASC)
    WHERE status = 'ready';
CREATE INDEX idx_tasks_critical_path ON coordinator.tasks(project_id)
    WHERE is_critical_path = TRUE;

-- ==== AGENT EXECUTIONS ====
CREATE TABLE coordinator.agent_executions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id         UUID NOT NULL REFERENCES coordinator.tasks(id),
    agent_type      TEXT NOT NULL,
    agent_instance  TEXT NOT NULL,
    model           TEXT NOT NULL,
    provider        TEXT NOT NULL,
    attempt         INTEGER NOT NULL DEFAULT 1,
    status          TEXT NOT NULL,
    input_tokens    INTEGER,
    output_tokens   INTEGER,
    cost_usd        NUMERIC(12, 6),
    wall_seconds    NUMERIC(10, 3),
    prompt_hash     TEXT,
    prompt_summary  TEXT,
    result_summary  TEXT,
    tool_call_count INTEGER NOT NULL DEFAULT 0,
    error           TEXT,
    started_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at    TIMESTAMPTZ
);
CREATE INDEX idx_executions_task ON coordinator.agent_executions(task_id);
CREATE INDEX idx_executions_agent ON coordinator.agent_executions(agent_type, started_at DESC);

-- ==== TOOL CALLS ====
CREATE TABLE coordinator.tool_calls (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    execution_id    UUID NOT NULL REFERENCES coordinator.agent_executions(id),
    tool_name       TEXT NOT NULL,
    tool_version    TEXT,
    input_summary   TEXT,
    output_summary  TEXT,
    exit_code       INTEGER,
    duration_ms     INTEGER NOT NULL DEFAULT 0,
    success         BOOLEAN NOT NULL,
    error           TEXT,
    sandbox_id      TEXT,
    called_at       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_tool_calls_execution ON coordinator.tool_calls(execution_id);
CREATE INDEX idx_tool_calls_name ON coordinator.tool_calls(tool_name, called_at DESC);

-- ==== APPROVAL GATES ====
CREATE TABLE coordinator.approval_gates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id         UUID NOT NULL REFERENCES coordinator.tasks(id),
    gate_type       TEXT NOT NULL,
    title           TEXT NOT NULL,
    description     TEXT NOT NULL DEFAULT '',
    diff_content    TEXT,
    diff_summary    TEXT,
    test_summary    TEXT,
    risk_score      INTEGER NOT NULL DEFAULT 50 CHECK (risk_score BETWEEN 0 AND 100),
    status          TEXT NOT NULL DEFAULT 'pending',
    approver_id     UUID REFERENCES coordinator.users(id),
    approver_notes  TEXT,
    auto_approve_after_seconds INTEGER,
    expires_at      TIMESTAMPTZ NOT NULL DEFAULT NOW() + INTERVAL '1 hour',
    decided_at      TIMESTAMPTZ,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT gates_status_check CHECK (status IN (
        'pending','approved','rejected','expired','auto_approved'
    )),
    CONSTRAINT gates_type_check CHECK (gate_type IN (
        'commit','deploy','delete','pr_merge','cost_limit','destructive','manual'
    ))
);
CREATE INDEX idx_gates_task ON coordinator.approval_gates(task_id);
CREATE INDEX idx_gates_pending ON coordinator.approval_gates(status, expires_at)
    WHERE status = 'pending';

-- ==== MEMORIES ====
CREATE TABLE coordinator.memories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES coordinator.projects(id) ON DELETE CASCADE,
    memory_type     TEXT NOT NULL,
    title           TEXT NOT NULL,
    content         TEXT NOT NULL,
    source          TEXT NOT NULL,
    source_task_id  UUID REFERENCES coordinator.tasks(id),
    tags            TEXT[] NOT NULL DEFAULT '{}',
    qdrant_id       UUID,
    importance      INTEGER NOT NULL DEFAULT 50 CHECK (importance BETWEEN 0 AND 100),
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_memories_type ON coordinator.memories(project_id, memory_type);
CREATE INDEX idx_memories_tags ON coordinator.memories USING GIN(tags);
CREATE INDEX idx_memories_importance ON coordinator.memories(project_id, importance DESC);
CREATE INDEX idx_memories_fts ON coordinator.memories USING GIN(
    to_tsvector('english', coalesce(title,'') || ' ' || coalesce(content,''))
);

-- ==== PLUGINS ====
CREATE TABLE coordinator.plugins (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    version         TEXT NOT NULL,
    type            TEXT NOT NULL,
    manifest        JSONB NOT NULL,
    enabled         BOOLEAN NOT NULL DEFAULT TRUE,
    scope           TEXT NOT NULL DEFAULT 'global',
    project_id      UUID REFERENCES coordinator.projects(id),
    installed_by    UUID REFERENCES coordinator.users(id),
    installed_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(name, version, COALESCE(project_id::text, 'global'))
);

-- ==== EVENTS (partitioned by month) ====
CREATE TABLE coordinator.events (
    id              BIGSERIAL,
    project_id      UUID REFERENCES coordinator.projects(id),
    task_id         UUID REFERENCES coordinator.tasks(id),
    event_type      TEXT NOT NULL,
    payload         JSONB NOT NULL DEFAULT '{}',
    source          TEXT NOT NULL,
    trace_id        TEXT,
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (id, occurred_at)
) PARTITION BY RANGE (occurred_at);

CREATE TABLE coordinator.events_2026_06 PARTITION OF coordinator.events
    FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');
CREATE TABLE coordinator.events_2026_07 PARTITION OF coordinator.events
    FOR VALUES FROM ('2026-07-01') TO ('2026-08-01');
CREATE TABLE coordinator.events_2026_08 PARTITION OF coordinator.events
    FOR VALUES FROM ('2026-08-01') TO ('2026-09-01');
CREATE TABLE coordinator.events_2026_09 PARTITION OF coordinator.events
    FOR VALUES FROM ('2026-09-01') TO ('2026-10-01');
CREATE TABLE coordinator.events_2026_10 PARTITION OF coordinator.events
    FOR VALUES FROM ('2026-10-01') TO ('2026-11-01');
CREATE TABLE coordinator.events_2026_11 PARTITION OF coordinator.events
    FOR VALUES FROM ('2026-11-01') TO ('2026-12-01');
CREATE TABLE coordinator.events_2026_12 PARTITION OF coordinator.events
    FOR VALUES FROM ('2026-12-01') TO ('2027-01-01');

CREATE INDEX idx_events_project ON coordinator.events(project_id, occurred_at DESC);
CREATE INDEX idx_events_task ON coordinator.events(task_id, occurred_at DESC)
    WHERE task_id IS NOT NULL;
CREATE INDEX idx_events_type ON coordinator.events(event_type, occurred_at DESC);

-- ==== KG INGESTION JOBS ====
CREATE TABLE coordinator.kg_ingestion_jobs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES coordinator.projects(id),
    status          TEXT NOT NULL DEFAULT 'pending',
    commit_hash     TEXT,
    files_processed INTEGER,
    nodes_upserted  INTEGER,
    edges_upserted  INTEGER,
    error           TEXT,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## 4.3 auth Schema DDL

```sql
CREATE SCHEMA IF NOT EXISTS auth;

CREATE TABLE auth.sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL,
    jwt_jti         TEXT UNIQUE NOT NULL,
    ip_address      INET,
    user_agent      TEXT,
    expires_at      TIMESTAMPTZ NOT NULL,
    revoked         BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_sessions_user ON auth.sessions(user_id, created_at DESC);
CREATE INDEX idx_sessions_jti ON auth.sessions(jwt_jti);
CREATE INDEX idx_sessions_active ON auth.sessions(expires_at)
    WHERE NOT revoked;

CREATE TABLE auth.api_keys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL,
    name            TEXT NOT NULL,
    key_hash        TEXT UNIQUE NOT NULL,
    key_prefix      TEXT NOT NULL,
    last_used_at    TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    revoked         BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_api_keys_user ON auth.api_keys(user_id);
CREATE INDEX idx_api_keys_hash ON auth.api_keys(key_hash);
```

## 4.4 brain Schema DDL

```sql
CREATE SCHEMA IF NOT EXISTS brain;

CREATE TABLE brain.execution_outcomes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id         UUID NOT NULL,
    project_id      UUID NOT NULL,
    agent_type      TEXT NOT NULL,
    task_type       TEXT NOT NULL,
    model           TEXT NOT NULL,
    outcome         TEXT NOT NULL CHECK (outcome IN ('success','failed','timeout','blocked','cancelled')),
    duration_s      NUMERIC(10,3),
    tokens_used     INTEGER,
    cost_usd        NUMERIC(12,6),
    error_category  TEXT,
    patterns_matched TEXT[] NOT NULL DEFAULT '{}',
    qdrant_id       UUID,
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_outcomes_agent ON brain.execution_outcomes(agent_type, outcome, recorded_at DESC);

CREATE TABLE brain.patterns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL UNIQUE,
    description     TEXT NOT NULL,
    pattern_type    TEXT NOT NULL CHECK (pattern_type IN ('success','failure','performance','cost')),
    agent_type      TEXT,
    task_type       TEXT,
    conditions      JSONB NOT NULL DEFAULT '{}',
    recommendation  TEXT NOT NULL,
    confidence      NUMERIC(4,3) NOT NULL DEFAULT 0.5 CHECK (confidence BETWEEN 0 AND 1),
    observation_count INTEGER NOT NULL DEFAULT 0,
    last_observed   TIMESTAMPTZ,
    qdrant_id       UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE brain.decisions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id         UUID NOT NULL,
    agent_type      TEXT NOT NULL,
    decision_type   TEXT NOT NULL,
    decision        TEXT NOT NULL,
    rationale       TEXT NOT NULL,
    patterns_used   UUID[] NOT NULL DEFAULT '{}',
    confidence      NUMERIC(4,3),
    outcome_id      UUID REFERENCES brain.execution_outcomes(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_decisions_task ON brain.decisions(task_id);
```

## 4.5 Qdrant Collections Specification

```python
# knowledge-engine/vector/collections.py

COLLECTION_SPECS = {
    "project_memories": {
        "vectors_config": {"size": 1536, "distance": "Cosine"},
        "hnsw_config": {"m": 16, "ef_construct": 128},
        "on_disk_payload": True,
        "payload_schema": {
            "project_id": "keyword",
            "memory_type": "keyword",
            "memory_id": "keyword",
            "importance": "integer",
            "created_at": "datetime",
            "expires_at": "datetime"
        }
    },
    "code_chunks": {
        "vectors_config": {"size": 1536, "distance": "Cosine"},
        "hnsw_config": {"m": 16, "ef_construct": 100},
        "payload_schema": {
            "project_id": "keyword",
            "file_path": "keyword",
            "language": "keyword",
            "chunk_type": "keyword",  # function|class|module|comment|docstring
            "start_line": "integer",
            "end_line": "integer",
            "file_hash": "keyword"
        }
    },
    "agent_prompts": {
        "vectors_config": {"size": 768, "distance": "Cosine"},
        "payload_schema": {
            "agent_type": "keyword",
            "task_type": "keyword",
            "outcome": "keyword",
            "model": "keyword",
            "tokens": "integer"
        }
    },
    "experience_outcomes": {
        "vectors_config": {"size": 1536, "distance": "Cosine"},
        "payload_schema": {
            "project_id": "keyword",
            "agent_type": "keyword",
            "task_type": "keyword",
            "outcome": "keyword",
            "error_category": "keyword"
        }
    }
}
```

## 4.6 Retention Policy

| Table | Retention | Mechanism |
|-------|-----------|-----------|
| coordinator.events | 90 days | Drop monthly partition; Parquet to S3 |
| coordinator.agent_executions | 180 days | pg_cron nightly DELETE |
| coordinator.tool_calls | 30 days | pg_cron nightly DELETE |
| coordinator.memories (conversation) | 7 days | expires_at field + daily cleanup job |
| coordinator.memories (bug) | 90 days post-fix | expires_at set by LearningAgent |
| coordinator.approval_gates | 1 year | Archive to S3 JSON; soft-delete in PG |
| coordinator.tasks | Permanent | Archive to S3 Parquet after 2 years (status=completed) |
| brain.execution_outcomes | 2 years | Delete after pattern extraction |

Enforcement: `SELECT coordinator.run_retention_cleanup()` via pg_cron at 02:00 UTC daily.

## 4.7 Migration Strategy

```
Tool: Alembic (asyncpg async mode)
Location: coordinator/db/migrations/versions/
Filename convention: YYYYMMDD_HHMMSS_{slug}.py

Rules:
1. Every migration MUST implement downgrade()
2. No migration may hold a table lock longer than 30 seconds
   - Use CREATE INDEX CONCURRENTLY (no lock)
   - Add nullable column first; backfill; add NOT NULL constraint separately
   - Use SKIP LOCKED for data migrations
3. Migration runs automatically on coordinator startup (alembic upgrade head)
4. All migrations tested against PostgreSQL 15 container in CI before merge
5. SCHEMA_VERSION integer increments with each migration
6. Breaking schema changes: deploy new schema, run dual-write period, then retire old
```

---

# PART 5: COORDINATOR IMPLEMENTATION

## 5.1 Purpose

The coordinator is the externally-facing control plane. It enforces authentication and authorization, persists project/task state, manages approval gates, broadcasts real-time updates to clients, and acts as NATS publisher for task lifecycle events. It does NOT execute AI work — it delegates everything to the orchestrator via NATS.

## 5.2 API Layer Structure

```
/api/v2/                         -> Auth required (JWT Bearer or API key)
/api/v2/projects/                -> Project CRUD
/api/v2/projects/{id}/tasks/     -> Task management
/api/v2/projects/{id}/members/   -> Project membership
/api/v2/approvals/               -> Approval gate management
/api/v2/memories/                -> Memory CRUD + hybrid search
/api/v2/agents/                  -> Agent status (read-only)
/api/v2/plugins/                 -> Plugin install/list/remove
/api/v2/models/                  -> AI model management (list, activate)
/api/v2/users/me                 -> Current user profile
/ws/projects/{id}/events         -> WebSocket real-time stream
/api/internal/                   -> No auth (internal network only)
/api/internal/health             -> Health endpoint (same as /healthz)
```

## 5.3 FastAPI Application Factory

```python
# coordinator/app.py

def create_app(settings: Settings) -> FastAPI:
    app = FastAPI(
        title="AI Studio Coordinator",
        version="2.0.0",
        docs_url="/docs" if settings.is_dev else None,
        redoc_url=None,
        lifespan=lifespan
    )

    # Middleware (order matters: outermost applied last)
    app.add_middleware(RequestIdMiddleware)
    app.add_middleware(StructuredLoggingMiddleware)
    app.add_middleware(PrometheusMiddleware)
    app.add_middleware(
        CORSMiddleware,
        allow_origins=settings.cors_origins,
        allow_methods=["GET","POST","PUT","PATCH","DELETE"],
        allow_headers=["Authorization","Content-Type","X-Request-Id"]
    )

    # Router registration
    app.include_router(projects_router, prefix="/api/v2/projects", tags=["projects"])
    app.include_router(tasks_router, prefix="/api/v2/projects", tags=["tasks"])
    app.include_router(approvals_router, prefix="/api/v2/approvals", tags=["approvals"])
    app.include_router(memories_router, prefix="/api/v2/memories", tags=["memories"])
    app.include_router(agents_router, prefix="/api/v2/agents", tags=["agents"])
    app.include_router(plugins_router, prefix="/api/v2/plugins", tags=["plugins"])
    app.include_router(models_router, prefix="/api/v2/models", tags=["models"])
    app.include_router(users_router, prefix="/api/v2/users", tags=["users"])
    app.include_router(ws_router, prefix="/ws", tags=["websocket"])
    app.include_router(internal_router, prefix="/api/internal", tags=["internal"])
    app.include_router(health_router, prefix="", tags=["health"])

    return app
```

## 5.4 Authentication — Request Processing

```python
# coordinator/auth/deps.py

async def get_current_user(
    request: Request,
    db: asyncpg.Connection = Depends(get_db),
    redis: Redis = Depends(get_redis)
) -> User:
    token = extract_bearer_token(request.headers.get("Authorization"))
    if token:
        return await _verify_jwt(token, db, redis)

    api_key = request.headers.get("X-API-Key")
    if api_key:
        return await _verify_api_key(api_key, db)

    raise HTTPException(status_code=401, detail="Authentication required")

async def _verify_jwt(token: str, db, redis) -> User:
    try:
        claims = jose_jwt.decode(
            token,
            settings.jwt_public_key,
            algorithms=["RS256"],
            audience="aistudio-api"
        )
    except JWTError as e:
        raise HTTPException(status_code=401, detail=f"Invalid token: {e}")

    jti = claims["jti"]
    if await redis.get(f"revoked_jti:{jti}"):
        raise HTTPException(status_code=401, detail="Token revoked")

    user_id = UUID(claims["sub"])
    user = await db.fetchrow(
        "SELECT * FROM coordinator.users WHERE id = $1", user_id
    )
    if not user:
        raise HTTPException(status_code=401, detail="User not found")
    return User(**dict(user))

async def _verify_api_key(raw_key: str, db) -> User:
    prefix = raw_key[:8]
    row = await db.fetchrow("""
        SELECT u.* FROM coordinator.users u
        JOIN auth.api_keys k ON k.user_id = u.id
        WHERE k.key_prefix = $1
          AND k.revoked = FALSE
          AND (k.expires_at IS NULL OR k.expires_at > NOW())
    """, prefix)
    if not row:
        raise HTTPException(status_code=401, detail="Invalid API key")
    user = User(**dict(row))
    if not bcrypt.checkpw(raw_key.encode(), row['key_hash'].encode()):
        raise HTTPException(status_code=401, detail="Invalid API key")
    await db.execute(
        "UPDATE auth.api_keys SET last_used_at = NOW() WHERE key_prefix = $1", prefix
    )
    return user
```

## 5.5 Task Submission Handler

```python
# coordinator/api/v2/tasks.py

class TaskCreateRequest(BaseModel):
    type: Literal['feature','bug','refactor','test','deploy','security',
                  'documentation','architecture','release','research']
    title: str = Field(min_length=1, max_length=500)
    description: str = Field(min_length=1, max_length=50000)
    acceptance_criteria: str = Field(default="", max_length=10000)
    priority: int = Field(default=50, ge=0, le=100)
    depends_on: list[UUID] = Field(default_factory=list)
    deadline_at: datetime | None = None
    metadata: dict = Field(default_factory=dict)

@router.post("/{project_id}/tasks", response_model=TaskResponse, status_code=201)
async def create_task(
    project_id: UUID,
    req: TaskCreateRequest,
    current_user: User = Depends(require_role("developer")),
    db: asyncpg.Connection = Depends(get_db),
    nats: JetStreamContext = Depends(get_nats)
):
    project = await get_project_or_404(project_id, current_user, db)

    task_id = await db.fetchval("""
        INSERT INTO coordinator.tasks (
            project_id, type, title, description, acceptance_criteria,
            priority, depends_on, deadline_at, metadata
        ) VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)
        RETURNING id
    """,
        project_id, req.type, req.title, req.description,
        req.acceptance_criteria, req.priority,
        req.depends_on, req.deadline_at, json.dumps(req.metadata)
    )

    await nats.publish(
        "tasks.created",
        TaskCreatedEvent(
            task_id=str(task_id),
            project_id=str(project_id),
            type=req.type,
            priority=req.priority,
            depends_on=[str(d) for d in req.depends_on]
        ).model_dump_json().encode()
    )

    task = await db.fetchrow(
        "SELECT * FROM coordinator.tasks WHERE id = $1", task_id
    )
    return TaskResponse(**dict(task))
```

## 5.6 Approval Gate Handler

```python
# coordinator/api/v2/approvals.py

@router.get("/pending", response_model=list[ApprovalGateResponse])
async def list_pending_approvals(
    project_id: UUID | None = None,
    current_user: User = Depends(get_current_user),
    db = Depends(get_db)
):
    rows = await db.fetch("""
        SELECT g.*, t.title as task_title, t.project_id
        FROM coordinator.approval_gates g
        JOIN coordinator.tasks t ON g.task_id = t.id
        WHERE g.status = 'pending'
          AND ($1::uuid IS NULL OR t.project_id = $1)
          AND ($2 OR t.project_id IN (
              SELECT project_id FROM coordinator.project_members WHERE user_id = $3
          ))
        ORDER BY g.risk_score DESC, g.created_at ASC
    """, project_id, current_user.role == 'admin', current_user.id)
    return [ApprovalGateResponse(**dict(r)) for r in rows]

@router.post("/{gate_id}/approve", response_model=ApprovalGateResponse)
async def approve_gate(
    gate_id: UUID,
    body: ApproveRequest,
    current_user: User = Depends(require_role("developer")),
    db = Depends(get_db),
    nats = Depends(get_nats)
):
    gate = await get_gate_or_404(gate_id, db)
    if gate['status'] != 'pending':
        raise HTTPException(400, f"Gate already {gate['status']}")

    await db.execute("""
        UPDATE coordinator.approval_gates
        SET status = 'approved',
            approver_id = $2,
            approver_notes = $3,
            decided_at = NOW()
        WHERE id = $1
    """, gate_id, current_user.id, body.notes)

    await db.execute("""
        UPDATE coordinator.tasks
        SET status = 'ready', updated_at = NOW()
        WHERE id = $1
    """, gate['task_id'])

    await nats.publish(
        "approval.granted",
        ApprovalGrantedEvent(
            gate_id=str(gate_id),
            task_id=str(gate['task_id']),
            approver_id=str(current_user.id)
        ).model_dump_json().encode()
    )

    return await get_gate_or_404(gate_id, db)
```

## 5.7 WebSocket Real-Time Updates

```python
# coordinator/api/v2/ws.py

class ConnectionManager:
    def __init__(self):
        self._connections: dict[UUID, set[WebSocket]] = defaultdict(set)
        self._lock = asyncio.Lock()

    async def connect(self, project_id: UUID, ws: WebSocket):
        await ws.accept()
        async with self._lock:
            self._connections[project_id].add(ws)

    async def disconnect(self, project_id: UUID, ws: WebSocket):
        async with self._lock:
            self._connections[project_id].discard(ws)

    async def broadcast(self, project_id: UUID, event: dict):
        payload = json.dumps(event)
        async with self._lock:
            connections = set(self._connections.get(project_id, set()))
        dead = set()
        for ws in connections:
            try:
                await ws.send_text(payload)
            except WebSocketDisconnect:
                dead.add(ws)
        async with self._lock:
            self._connections[project_id] -= dead

manager = ConnectionManager()

@ws_router.websocket("/projects/{project_id}/events")
async def project_events(
    project_id: UUID,
    ws: WebSocket,
    token: str = Query(...),
    db = Depends(get_db)
):
    user = await verify_ws_token(token, db)
    if not await has_project_access(user, project_id, db):
        await ws.close(code=4003)
        return

    await manager.connect(project_id, ws)
    try:
        while True:
            await ws.receive_text()  # Keep alive; client pings every 30s
    except WebSocketDisconnect:
        await manager.disconnect(project_id, ws)
```

## 5.8 NATS Subscriber — Agent Outbox Consumer

```python
# coordinator/nats/subscribers.py

async def subscribe_agent_outboxes(js: JetStreamContext, db, ws_manager):
    async def on_agent_result(msg: Msg):
        result = AgentResultEvent.model_validate_json(msg.data)
        async with db.transaction():
            await db.execute("""
                UPDATE coordinator.tasks
                SET status = $2,
                    result_data = $3,
                    actual_tokens = $4,
                    cost_usd = $5,
                    completed_at = CASE WHEN $2 IN ('completed','failed')
                                        THEN NOW() ELSE NULL END,
                    updated_at = NOW()
                WHERE id = $1
            """,
                UUID(result.task_id), result.status,
                json.dumps(result.result), result.tokens_used, result.cost_usd
            )

            task = await db.fetchrow(
                "SELECT project_id FROM coordinator.tasks WHERE id = $1",
                UUID(result.task_id)
            )

            await db.execute("""
                INSERT INTO coordinator.events
                    (project_id, task_id, event_type, payload, source)
                VALUES ($1, $2, $3, $4, 'agent-runtime')
            """,
                task['project_id'], UUID(result.task_id),
                f"task.{result.status}", json.dumps(result.dict()), 
            )

        await ws_manager.broadcast(
            task['project_id'],
            {"type": "task.updated", "task_id": result.task_id, "status": result.status}
        )
        await msg.ack()

    await js.subscribe(
        subject="agent.*.outbox",
        stream="AGENT",
        durable="coordinator-result-consumer",
        cb=on_agent_result,
        manual_ack=True
    )
```

## 5.9 Configuration

```python
# coordinator/config.py

class Settings(BaseSettings):
    # Database
    pg_dsn: SecretStr = Field(..., env="PG_DSN")  # postgresql+asyncpg://...
    pg_pool_min: int = 5
    pg_pool_max: int = 20

    # Cache
    redis_url: str = Field(..., env="REDIS_URL")
    redis_pool_max: int = 10
    session_ttl_seconds: int = 86400 * 7  # 7 days

    # NATS
    nats_url: str = Field(..., env="NATS_URL")
    nats_credentials: SecretStr | None = None

    # Auth
    jwt_public_key: str = Field(..., env="JWT_PUBLIC_KEY")  # RSA PEM
    cors_origins: list[str] = Field(default=["http://localhost:3000"])

    # Services
    knowledge_engine_url: str = Field(default="http://knowledge-engine:8091")
    auth_service_url: str = Field(default="http://auth:8093")

    # Runtime
    is_dev: bool = Field(default=False, env="DEV_MODE")
    log_level: str = Field(default="INFO", env="LOG_LEVEL")
    port: int = 8090
    instance_id: str = Field(default_factory=lambda: f"coordinator-{uuid4().hex[:8]}")

    class Config:
        env_file = ".env"
        case_sensitive = False
```

## 5.10 Testing

```
Unit tests (coordinator/tests/unit/):
  test_auth_jwt.py         -> JWT decode, expiry, revocation, claims extraction
  test_auth_api_key.py     -> API key prefix matching, bcrypt validation, expiry
  test_task_validation.py  -> Pydantic model validation for all fields
  test_approval_logic.py   -> Gate state transitions, race condition prevention
  test_ws_manager.py       -> Connection add/remove/broadcast, dead connection cleanup

Integration tests (coordinator/tests/integration/):
  test_task_lifecycle.py   -> Create task -> publish NATS -> verify DB state
  test_approval_flow.py    -> Submit -> gate created -> approve -> task unblocked
  test_ws_broadcast.py     -> Multiple WS clients receive same event
  test_auth_flow.py        -> Login -> JWT -> API calls -> logout -> token rejected

Performance tests (coordinator/tests/perf/):
  test_task_throughput.py  -> 500 task creates/second sustained 30s
  test_ws_scale.py         -> 1000 concurrent WS connections, broadcast latency < 50ms
  test_query_latency.py    -> P95 API response < 100ms under 200 concurrent users

Coverage requirement: 80% minimum, enforced in CI.
```

---

# PART 6: WORKER RUNTIME (AGENT-RUNTIME)

## 6.1 Purpose

The agent-runtime hosts the AI agent pool. Each agent is stateless — per-task state is fetched from databases and written back atomically. Replicas compete on NATS queue groups for tasks of each type, providing automatic horizontal scaling. Maximum concurrency is bounded by NATS consumer settings and per-agent token budgets.

## 6.2 BaseAgent Abstract Class

```python
# agent-runtime/agents/base.py

from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from typing import Any
from uuid import UUID

@dataclass(frozen=True)
class AgentContext:
    task_id: UUID
    project_id: UUID
    task_type: str
    title: str
    description: str
    acceptance_criteria: str
    project_memories: list[dict]       # Retrieved from memory tier
    similar_past_tasks: list[dict]     # Retrieved from experience graph
    code_graph_summary: str            # KG query: hotspots, dependencies
    pattern_recommendations: list[dict] # From brain service
    model: str
    provider: str
    token_budget: int
    cost_budget_usd: float
    workspace_path: str
    project_settings: dict
    parent_context: dict[str, Any] = field(default_factory=dict)

@dataclass
class AgentResult:
    task_id: UUID
    status: str   # completed | failed | blocked | requires_approval
    summary: str
    artifacts: list[dict]     # {path, type, size_bytes, description}
    tokens_used: int
    cost_usd: float
    next_steps: list[dict]    # Suggested follow-up tasks
    checkpoint: dict | None = None
    error: str | None = None
    approval_required: bool = False
    approval_gate_spec: dict | None = None

@dataclass
class CheckpointState:
    step: str
    partial_results: dict
    tool_call_history: list[dict]
    accumulated_tokens: int

class BaseAgent(ABC):
    agent_type: str                       # Must override
    capabilities: list[str]               # Task types this agent can handle
    max_tokens_per_call: int = 8192
    preferred_models: list[str] = field(default_factory=list)
    max_concurrent: int = 4

    @abstractmethod
    async def execute(self, task: dict, context: AgentContext) -> AgentResult:
        pass

    async def checkpoint(self, state: CheckpointState) -> None:
        await db_pool.execute("""
            UPDATE coordinator.tasks
            SET checkpoint_data = $2, updated_at = NOW()
            WHERE id = $1
        """, state.task_id, json.dumps(dataclasses.asdict(state)))

    async def resume(self, state: CheckpointState) -> AgentResult:
        raise NotImplementedError("Override for stateful agents")

    def estimate_cost(self, context: AgentContext) -> float:
        rate = MODEL_COST_USD_PER_1K_TOKENS.get(context.model, 0.015)
        return (context.token_budget / 1000) * rate

    def validate_task(self, task: dict) -> tuple[bool, str]:
        if task.get("type") not in self.capabilities:
            return False, f"{self.agent_type} cannot handle task type {task.get('type')}"
        return True, ""

    async def dispatch(self, msg: Msg):
        task_msg = AgentTaskMessage.model_validate_json(msg.data)
        context = await context_assembler.assemble(task_msg.task_id)
        try:
            result = await self.execute(task_msg.task, context)
            await nats_publisher.publish_result(result)
            await msg.ack()
        except Exception as e:
            log.exception("Agent %s failed on task %s", self.agent_type, task_msg.task_id)
            await nats_publisher.publish_failure(task_msg.task_id, str(e))
            await msg.nack(delay=10)
```

## 6.3 Context Assembler

```python
# agent-runtime/context/assembler.py

class ContextAssembler:
    def __init__(self, ke_client: KnowledgeEngineClient, brain_client: BrainClient):
        self.ke = ke_client
        self.brain = brain_client

    async def assemble(self, task_id: UUID) -> AgentContext:
        task = await db_pool.fetchrow(
            "SELECT * FROM coordinator.tasks WHERE id = $1", task_id
        )
        project = await db_pool.fetchrow(
            "SELECT * FROM coordinator.projects WHERE id = $1", task['project_id']
        )

        # Parallel fetch: memories, experience, graph, patterns
        memories, past_tasks, kg_summary, patterns = await asyncio.gather(
            self.ke.hybrid_search(
                query=task['description'],
                project_id=task['project_id'],
                limit=20,
                types=['decision','architecture','constraint','bug']
            ),
            self.brain.find_similar_tasks(
                task_type=task['type'],
                description=task['description'],
                limit=10
            ),
            self.ke.graph_query(
                project_id=task['project_id'],
                query_name="file_hotspots_and_dependencies"
            ),
            self.brain.get_recommendations(
                task_type=task['type'],
                agent_type=_infer_agent_type(task['type'])
            )
        )

        model, provider = model_router.select(
            task_type=task['type'],
            preferred=project['default_ai_provider'],
            token_budget=_estimate_token_budget(task, len(memories))
        )

        return AgentContext(
            task_id=task['id'],
            project_id=task['project_id'],
            task_type=task['type'],
            title=task['title'],
            description=task['description'],
            acceptance_criteria=task['acceptance_criteria'],
            project_memories=memories,
            similar_past_tasks=past_tasks,
            code_graph_summary=kg_summary,
            pattern_recommendations=patterns,
            model=model,
            provider=provider,
            token_budget=_estimate_token_budget(task, len(memories)),
            cost_budget_usd=5.0,
            workspace_path=f"/workspace/{task_id}",
            project_settings=project['settings'] or {}
        )
```

## 6.4 BackendAgent Implementation

```python
# agent-runtime/agents/backend.py

class BackendAgent(BaseAgent):
    agent_type = "backend"
    capabilities = ["feature", "bug", "refactor"]
    max_tokens_per_call = 8192
    preferred_models = ["claude-opus-4-8", "claude-sonnet-4-6"]

    async def execute(self, task: dict, context: AgentContext) -> AgentResult:
        accumulated_tokens = 0
        artifacts = []

        # Step 1: Analyze existing code via AST and search
        codebase_summary = await self._analyze_relevant_code(context)

        # Step 2: Generate implementation plan
        plan_result = await self._call_ai(
            system=BACKEND_SYSTEM_PROMPT,
            messages=[{
                "role": "user",
                "content": f"Task: {context.title}\n\n"
                           f"Description: {context.description}\n\n"
                           f"Acceptance Criteria:\n{context.acceptance_criteria}\n\n"
                           f"Relevant Codebase:\n{codebase_summary}\n\n"
                           f"Project Memories:\n{self._format_memories(context.project_memories)}\n\n"
                           f"Pattern Recommendations:\n{json.dumps(context.pattern_recommendations, indent=2)}\n\n"
                           "First, produce an implementation plan with specific files to create/modify."
            }],
            context=context
        )
        accumulated_tokens += plan_result.usage.input_tokens + plan_result.usage.output_tokens

        # Step 3: Checkpoint after plan
        await self.checkpoint(CheckpointState(
            step="plan_complete",
            partial_results={"plan": plan_result.content[0].text},
            tool_call_history=[],
            accumulated_tokens=accumulated_tokens
        ))

        # Step 4: Execute implementation with tool calls
        implementation_result = await self._implement_with_tools(
            plan=plan_result.content[0].text,
            context=context,
            accumulated_tokens=accumulated_tokens
        )
        accumulated_tokens += implementation_result.tokens

        # Step 5: Verify with tests
        test_result = await self._run_verification(context)
        if not test_result.passed:
            return AgentResult(
                task_id=context.task_id,
                status="blocked",
                summary=f"Tests failed: {test_result.summary}",
                artifacts=artifacts,
                tokens_used=accumulated_tokens,
                cost_usd=accumulated_tokens * 0.000015,
                next_steps=[{
                    "type": "bug",
                    "title": f"Fix failing tests for: {context.title}",
                    "description": test_result.failure_details
                }]
            )

        # Step 6: Risk assessment for approval gate
        diff_size = await self._measure_diff(context)
        requires_approval = diff_size.lines_changed > 500 or diff_size.files_changed > 10

        return AgentResult(
            task_id=context.task_id,
            status="completed" if not requires_approval else "requires_approval",
            summary=implementation_result.summary,
            artifacts=implementation_result.artifacts,
            tokens_used=accumulated_tokens,
            cost_usd=accumulated_tokens * 0.000015,
            next_steps=self._suggest_next_steps(context, implementation_result),
            approval_required=requires_approval,
            approval_gate_spec={
                "gate_type": "commit",
                "title": f"Review: {context.title}",
                "diff_summary": diff_size.summary,
                "risk_score": min(100, (diff_size.lines_changed // 10) + (diff_size.files_changed * 5))
            } if requires_approval else None
        )
```

## 6.5 PlannerAgent Implementation

```python
# agent-runtime/agents/planner.py

SDLC_STAGES = [
    "architecture",    # ArchitectAgent reviews + ADR
    "database",        # DatabaseAgent handles schema + migrations
    "backend",         # BackendAgent builds services
    "security",        # SecurityAgent reviews API surface
    "frontend",        # FrontendAgent builds UI (if applicable)
    "qa",              # QAAgent generates + runs tests
    "reviewer",        # ReviewerAgent code review
    "documentation",   # DocumentationAgent docstrings + README
    "devops",          # DevOpsAgent updates CI/CD
    "release"          # ReleaseAgent handles versioning + SBOM
]

class PlannerAgent(BaseAgent):
    agent_type = "planner"
    capabilities = ["feature", "architecture", "research"]
    preferred_models = ["claude-opus-4-8"]

    async def execute(self, task: dict, context: AgentContext) -> AgentResult:
        decomposition = await self._call_ai(
            system=PLANNER_SYSTEM_PROMPT,
            messages=[{
                "role": "user",
                "content": self._build_decomposition_prompt(task, context)
            }],
            context=context,
            response_format="json"
        )

        subtasks = self._parse_subtask_plan(decomposition.content[0].text)
        validated = self._validate_dependency_dag(subtasks)

        # Publish subtasks directly via NATS
        for subtask in validated:
            await nats.publish(
                "tasks.plan.subtask",
                json.dumps({
                    "parent_task_id": str(context.task_id),
                    "project_id": str(context.project_id),
                    **subtask
                }).encode()
            )

        return AgentResult(
            task_id=context.task_id,
            status="completed",
            summary=f"Decomposed into {len(validated)} subtasks across {len(set(t['assigned_agent'] for t in validated))} agent types",
            artifacts=[{"type": "plan", "content": subtasks}],
            tokens_used=decomposition.usage.input_tokens + decomposition.usage.output_tokens,
            cost_usd=0.0,
            next_steps=[]
        )

    def _build_decomposition_prompt(self, task: dict, context: AgentContext) -> str:
        return f"""Decompose this software feature into implementation subtasks.

Feature Request: {task['title']}
Description: {task['description']}
Acceptance Criteria: {task.get('acceptance_criteria', 'None specified')}

Past Similar Tasks (for pattern matching):
{json.dumps(context.similar_past_tasks[:5], indent=2)}

Pattern Recommendations:
{json.dumps(context.pattern_recommendations, indent=2)}

Produce a JSON array of subtasks. Each subtask must have:
- type: one of {list(SDLC_STAGES)}
- title: clear, specific title
- description: detailed enough for an AI agent to execute without clarification
- acceptance_criteria: verifiable success criteria
- assigned_agent: one of architect|database|backend|security|frontend|qa|reviewer|documentation|devops|release
- depends_on: list of indices into this array (0-based) representing prerequisite subtasks
- estimated_tokens: integer
- priority: 0-100 (higher = more urgent)

Return ONLY valid JSON. No explanation."""
```

## 6.6 Model Router

```python
# agent-runtime/providers/router.py

MODEL_AFFINITY = {
    "architecture":    ["claude-opus-4-8", "claude-sonnet-4-6"],
    "planner":         ["claude-opus-4-8"],
    "security":        ["claude-opus-4-8", "claude-sonnet-4-6"],
    "backend":         ["claude-opus-4-8", "claude-sonnet-4-6"],
    "database":        ["claude-sonnet-4-6"],
    "frontend":        ["claude-sonnet-4-6"],
    "qa":              ["claude-sonnet-4-6", "claude-haiku-4-5-20251001"],
    "reviewer":        ["claude-sonnet-4-6"],
    "documentation":   ["claude-haiku-4-5-20251001", "claude-sonnet-4-6"],
    "devops":          ["claude-sonnet-4-6"],
    "refactoring":     ["claude-opus-4-8"],
    "monitoring":      ["claude-haiku-4-5-20251001"],
    "learning":        ["claude-haiku-4-5-20251001"],
}

FALLBACK_CHAIN = [
    "claude-opus-4-8",
    "claude-sonnet-4-6",
    "claude-haiku-4-5-20251001",
    "ollama/llama3:70b"  # Local fallback (laptop mode)
]

class ModelRouter:
    async def select(self, task_type: str, preferred: str, token_budget: int) -> tuple[str, str]:
        affinity_list = MODEL_AFFINITY.get(task_type, FALLBACK_CHAIN)
        for model_id in affinity_list:
            provider = _provider_from_model(model_id)
            if await self._is_available(provider, model_id):
                return model_id, provider
        return "ollama/llama3:70b", "ollama"  # Last resort

    async def _is_available(self, provider: str, model: str) -> bool:
        # Check circuit breaker in Redis
        key = f"circuit_breaker:{provider}:{model}"
        if await redis.get(key):
            return False
        return provider in settings.enabled_providers
```

## 6.7 Tool Call Audit Trail

Every tool call is recorded to `coordinator.tool_calls` with input/output summaries (NOT raw content, to avoid large payload storage):

```python
# agent-runtime/tools/base.py

class BaseTool(ABC):
    tool_name: str
    tool_version: str = "1.0.0"
    requires_sandbox: bool = False

    @abstractmethod
    async def run(self, input: dict, context: AgentContext) -> dict:
        pass

    async def call(self, input: dict, context: AgentContext, execution_id: UUID) -> dict:
        started = time.monotonic()
        result = None
        error = None
        try:
            result = await self.run(input, context)
            success = True
        except Exception as e:
            error = str(e)
            success = False
            raise
        finally:
            duration_ms = int((time.monotonic() - started) * 1000)
            await db_pool.execute("""
                INSERT INTO coordinator.tool_calls
                    (execution_id, tool_name, tool_version, input_summary,
                     output_summary, success, error, duration_ms)
                VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
            """,
                execution_id, self.tool_name, self.tool_version,
                _summarize(input), _summarize(result) if result else None,
                success, error, duration_ms
            )
        return result

def _summarize(data: dict | None) -> str | None:
    if data is None:
        return None
    text = json.dumps(data)
    if len(text) <= 500:
        return text
    return text[:497] + "..."
```

---

# PART 7: EVENT BUS SPECIFICATION

## 7.1 NATS JetStream Configuration

```yaml
# deployment/nats/nats.conf
jetstream: {
  store_dir: /data/jetstream
  max_memory_store: 2GB
  max_file_store: 50GB
}
accounts: {
  aistudio: {
    jetstream: enabled
    users: [
      {user: coordinator, password: $NATS_COORDINATOR_PASS}
      {user: orchestrator, password: $NATS_ORCHESTRATOR_PASS}
      {user: agent-runtime, password: $NATS_AGENT_PASS}
      {user: knowledge-engine, password: $NATS_KE_PASS}
    ]
  }
}
```

## 7.2 Stream Definitions

```python
# deployment/nats/streams.py

STREAM_CONFIGS = [
    StreamConfig(
        name="TASKS",
        subjects=["tasks.*", "tasks.plan.*"],
        retention=RetentionPolicy.LIMITS,
        max_age=timedelta(days=7),
        storage=StorageType.FILE,
        num_replicas=1,   # 3 in production cluster
        duplicate_window=timedelta(minutes=5),
        deny_delete=True
    ),
    StreamConfig(
        name="AGENT",
        subjects=["agent.>"],
        retention=RetentionPolicy.LIMITS,
        max_age=timedelta(days=1),
        max_msg_size=4 * 1024 * 1024,  # 4 MB per message
        storage=StorageType.FILE,
        num_replicas=1,
        deny_delete=False
    ),
    StreamConfig(
        name="MEMORY",
        subjects=["memory.>"],
        retention=RetentionPolicy.LIMITS,
        max_age=timedelta(days=3),
        storage=StorageType.FILE
    ),
    StreamConfig(
        name="APPROVAL",
        subjects=["approval.>"],
        retention=RetentionPolicy.LIMITS,
        max_age=timedelta(days=30),
        storage=StorageType.FILE,
        deny_delete=True
    ),
    StreamConfig(
        name="EVENTS",
        subjects=["events.>"],
        retention=RetentionPolicy.LIMITS,
        max_age=timedelta(days=90),
        storage=StorageType.FILE
    ),
    StreamConfig(
        name="CI",
        subjects=["ci.>"],
        retention=RetentionPolicy.LIMITS,
        max_age=timedelta(days=7),
        storage=StorageType.FILE
    )
]
```

## 7.3 Event Payload Schemas

```python
# aistudio_sdk/events/schema.py

from dataclasses import dataclass, field
from datetime import datetime
from uuid import UUID, uuid4

@dataclass
class BaseEvent:
    event_id: str = field(default_factory=lambda: str(uuid4()))
    version: str = "1"
    timestamp: str = field(default_factory=lambda: datetime.utcnow().isoformat())
    trace_id: str = field(default_factory=lambda: str(uuid4()))
    source: str = ""

@dataclass
class TaskCreatedEvent(BaseEvent):
    source: str = "coordinator"
    task_id: str = ""
    project_id: str = ""
    type: str = ""
    priority: int = 50
    depends_on: list[str] = field(default_factory=list)

@dataclass
class AgentDispatchMessage(BaseEvent):
    source: str = "orchestrator"
    task_id: str = ""
    project_id: str = ""
    agent_type: str = ""
    model: str = ""
    provider: str = ""
    resume_from_checkpoint: bool = False
    checkpoint_data: dict = field(default_factory=dict)

@dataclass
class AgentResultEvent(BaseEvent):
    source: str = "agent-runtime"
    task_id: str = ""
    agent_type: str = ""
    status: str = ""   # completed | failed | blocked | requires_approval
    summary: str = ""
    tokens_used: int = 0
    cost_usd: float = 0.0
    artifacts: list[dict] = field(default_factory=list)
    next_steps: list[dict] = field(default_factory=list)
    approval_gate_spec: dict | None = None
    error: str | None = None

@dataclass
class ApprovalRequestedEvent(BaseEvent):
    source: str = "coordinator"
    gate_id: str = ""
    task_id: str = ""
    project_id: str = ""
    gate_type: str = ""
    risk_score: int = 50
    expires_at: str = ""
    description: str = ""

@dataclass
class ApprovalGrantedEvent(BaseEvent):
    source: str = "coordinator"
    gate_id: str = ""
    task_id: str = ""
    approver_id: str = ""
    notes: str = ""

@dataclass
class ApprovalRejectedEvent(BaseEvent):
    source: str = "coordinator"
    gate_id: str = ""
    task_id: str = ""
    approver_id: str = ""
    reason: str = ""

@dataclass
class MemoryWriteEvent(BaseEvent):
    source: str = "agent-runtime"
    project_id: str = ""
    memory_type: str = ""
    content: str = ""
    title: str = ""
    importance: int = 50
    tags: list[str] = field(default_factory=list)

@dataclass
class CIRunCompleteEvent(BaseEvent):
    source: str = "devops-agent"
    task_id: str = ""
    project_id: str = ""
    run_id: str = ""
    passed: bool = False
    coverage_pct: float = 0.0
    failed_tests: list[str] = field(default_factory=list)
    artifact_url: str = ""
```

## 7.4 Dead Letter Queue

```python
# Every consumer configures a DLQ threshold.
# After max_deliver attempts, messages move to DLQ stream for human review.

DLQ_CONSUMER_CONFIG = ConsumerConfig(
    filter_subject="*.*.dead",
    deliver_policy=DeliverPolicy.ALL,
    max_deliver=1,
    ack_wait=timedelta(seconds=30)
)

# DLQ Monitor — runs in orchestrator, checks every 5 minutes
async def dlq_monitor():
    while True:
        msgs = await fetch_dlq_messages(limit=100)
        for msg in msgs:
            await db_pool.execute("""
                INSERT INTO coordinator.events
                    (event_type, payload, source)
                VALUES ('dlq.message', $1, 'nats-dlq-monitor')
            """, json.dumps({"subject": msg.subject, "data": msg.data.decode()}))
            # Alert to admin via WebSocket broadcast project_id=None (global channel)
            await alert_admin(f"DLQ message on {msg.subject}")
            await msg.ack()
        await asyncio.sleep(300)
```

## 7.5 Consumer Configurations by Service

```
coordinator:
  - Stream: AGENT, Subject: agent.*.outbox, Durable: coordinator-result-consumer
    MaxDeliver: 3, AckWait: 60s
  - Stream: APPROVAL, Subject: approval.granted|rejected, Durable: coordinator-approval-consumer
    MaxDeliver: 3, AckWait: 30s

orchestrator:
  - Stream: TASKS, Subject: tasks.created, Durable: orchestrator-dispatch
    MaxDeliver: 5, AckWait: 120s
  - Stream: AGENT, Subject: agent.*.outbox, Durable: orchestrator-result-handler
    MaxDeliver: 3, AckWait: 60s

agent-runtime (per replica, queue group per agent type):
  - Stream: AGENT, Subject: agent.{type}.inbox, Queue: pool.{type}
    MaxDeliver: 3, AckWait: 600s (agents can run for up to 10 minutes)

knowledge-engine:
  - Stream: MEMORY, Subject: memory.write, Durable: ke-memory-writer
    MaxDeliver: 5, AckWait: 30s

brain:
  - Stream: AGENT, Subject: agent.*.outbox, Durable: brain-outcome-recorder
    MaxDeliver: 3, AckWait: 30s
```

---

# PART 8: MEMORY SYSTEM

## 8.1 Memory Types and Tiers

```python
# aistudio_sdk/memory/types.py

class MemoryType:
    # Project-scoped long-term memories
    ARCHITECTURE    = "architecture"      # ADRs, design decisions
    DECISION        = "decision"          # One-off choices made during tasks
    CONSTRAINT      = "constraint"        # Never-violate rules (e.g., "don't use MongoDB")
    BUG             = "bug"               # Known issues, workarounds
    PATTERN         = "pattern"           # Successful repeatable patterns
    FAILURE         = "failure"           # Anti-patterns to avoid
    DEPENDENCY      = "dependency"        # Dependency decisions (why, when, alternatives)
    DOMAIN          = "domain"            # Domain knowledge specific to the project
    PERFORMANCE     = "performance"       # Performance bottlenecks, benchmarks
    SECURITY        = "security"          # Security decisions, audit findings
    CONVERSATION    = "conversation"      # Short-lived session context (7d TTL)
    PREFERENCE      = "preference"        # Developer preferences for this project
    EXPERIMENT      = "experiment"        # A/B test results, outcomes
    COMPLIANCE      = "compliance"        # Regulatory constraints

MEMORY_TTL_DAYS = {
    MemoryType.CONVERSATION: 7,
    MemoryType.EXPERIMENT: 30,
    MemoryType.BUG: 90,     # Reset to None when bug is fixed
    MemoryType.PERFORMANCE: 180,
    # All others: None (no expiry)
}
```

## 8.2 Memory Write Flow

```python
# knowledge-engine/memory/writer.py

async def write_memory(
    project_id: UUID,
    memory_type: str,
    title: str,
    content: str,
    source: str,
    tags: list[str] = None,
    importance: int = 50,
    source_task_id: UUID = None
) -> UUID:
    # 1. Embed the content
    vector = await embedder.embed(f"{title}\n\n{content}")

    # 2. Insert to PostgreSQL (memory metadata)
    ttl_days = MEMORY_TTL_DAYS.get(memory_type)
    expires_at = datetime.utcnow() + timedelta(days=ttl_days) if ttl_days else None

    memory_id = await db_pool.fetchval("""
        INSERT INTO coordinator.memories
            (project_id, memory_type, title, content, source, source_task_id,
             tags, importance, expires_at)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)
        RETURNING id
    """,
        project_id, memory_type, title, content, source,
        source_task_id, tags or [], importance, expires_at
    )

    # 3. Upsert to Qdrant
    qdrant_id = uuid4()
    await qdrant_client.upsert(
        collection_name="project_memories",
        points=[
            PointStruct(
                id=str(qdrant_id),
                vector=vector,
                payload={
                    "project_id": str(project_id),
                    "memory_id": str(memory_id),
                    "memory_type": memory_type,
                    "importance": importance,
                    "created_at": datetime.utcnow().isoformat()
                }
            )
        ]
    )

    # 4. Update qdrant_id back-reference
    await db_pool.execute(
        "UPDATE coordinator.memories SET qdrant_id = $2 WHERE id = $1",
        memory_id, qdrant_id
    )

    # 5. Publish NATS event for other subscribers
    await nats.publish(
        "memory.write",
        MemoryWriteEvent(
            project_id=str(project_id),
            memory_type=memory_type,
            title=title,
            content=content[:500],
            importance=importance,
            tags=tags or []
        ).model_dump_json().encode()
    )

    return memory_id
```

## 8.3 Hybrid Memory Retrieval

```python
# knowledge-engine/memory/retriever.py

async def hybrid_search(
    query: str,
    project_id: UUID,
    limit: int = 20,
    types: list[str] = None,
    min_importance: int = 0
) -> list[dict]:
    # Branch 1: Qdrant vector search (semantic similarity)
    query_vector = await embedder.embed(query)
    qdrant_filter = Filter(
        must=[
            FieldCondition(key="project_id", match=MatchValue(value=str(project_id))),
            FieldCondition(key="importance", range=Range(gte=min_importance))
        ]
    )
    if types:
        qdrant_filter.must.append(
            FieldCondition(key="memory_type", match=MatchAny(any=types))
        )

    vector_results = await qdrant_client.search(
        collection_name="project_memories",
        query_vector=query_vector,
        query_filter=qdrant_filter,
        limit=limit * 2,
        with_payload=True
    )

    # Branch 2: PostgreSQL full-text search (lexical recall)
    fts_results = await db_pool.fetch("""
        SELECT id, memory_type, title, content, importance, tags,
               ts_rank(to_tsvector('english', title || ' ' || content),
                       plainto_tsquery($2)) as rank
        FROM coordinator.memories
        WHERE project_id = $1
          AND (expires_at IS NULL OR expires_at > NOW())
          AND ($3::text[] IS NULL OR memory_type = ANY($3))
          AND importance >= $4
          AND to_tsvector('english', title || ' ' || content)
              @@ plainto_tsquery($2)
        ORDER BY rank DESC
        LIMIT $5
    """, project_id, query, types, min_importance, limit)

    # Merge + RRF (Reciprocal Rank Fusion)
    qdrant_ids = {r.payload['memory_id']: (1.0 / (i + 60), r.score)
                  for i, r in enumerate(vector_results)}
    fts_ids = {str(r['id']): (1.0 / (i + 60), r['rank'])
               for i, r in enumerate(fts_results)}

    all_ids = set(qdrant_ids.keys()) | set(fts_ids.keys())
    scored = []
    for mid in all_ids:
        rrf_score = qdrant_ids.get(mid, (0, 0))[0] + fts_ids.get(mid, (0, 0))[0]
        scored.append((mid, rrf_score))
    scored.sort(key=lambda x: x[1], reverse=True)

    # Fetch full records for top results
    top_ids = [UUID(mid) for mid, _ in scored[:limit]]
    rows = await db_pool.fetch("""
        SELECT * FROM coordinator.memories
        WHERE id = ANY($1) AND project_id = $2
    """, top_ids, project_id)

    id_to_row = {str(r['id']): dict(r) for r in rows}
    return [id_to_row[mid] for mid, _ in scored[:limit] if mid in id_to_row]
```

## 8.4 Knowledge Graph — Kuzu Schema

```
NODE TABLES:
  Project(id STRING, name STRING, slug STRING, created_at TIMESTAMP)
  Module(id STRING, project_id STRING, name STRING, path STRING, language STRING)
  File(id STRING, project_id STRING, path STRING, language STRING, line_count INT64, hash STRING)
  Class(id STRING, file_id STRING, name STRING, line_start INT64, line_end INT64)
  Method(id STRING, class_id STRING, name STRING, signature STRING, line_start INT64, complexity INT64)
  Function(id STRING, file_id STRING, name STRING, signature STRING, line_start INT64)
  Table(id STRING, project_id STRING, schema_name STRING, table_name STRING)
  Column(id STRING, table_id STRING, name STRING, type STRING, nullable BOOLEAN)
  ApiEndpoint(id STRING, project_id STRING, method STRING, path STRING, handler STRING)
  Requirement(id STRING, project_id STRING, text STRING, source STRING, priority INT64)

RELATIONSHIP TABLES:
  BELONGS_TO(From: Module, To: Project)
  DEFINED_IN_FILE(From: Class, To: File)
  DEFINED_IN_FILE_FUNC(From: Function, To: File)
  DEFINED_IN_CLASS(From: Method, To: Class)
  CALLS(From: Method, To: Method, call_count INT64)
  CALLS_FUNC(From: Method, To: Function)
  READS(From: Method, To: Table)
  WRITES(From: Method, To: Table)
  HANDLED_BY(From: ApiEndpoint, To: Method)
  DEPENDS_ON_CLASS(From: Class, To: Class)
  IMPORTS(From: File, To: File)
  IMPLEMENTS(From: Class, To: Class)
  HAS_COLUMN(From: Table, To: Column)
  MODIFIES(From: Requirement, To: File)
  APPLIES_TO(From: Requirement, To: ApiEndpoint)
```

## 8.5 Code Ingestion Pipeline

```python
# knowledge-engine/graph/ingestion.py

class RepositoryIngestionPipeline:
    def __init__(self, kg_client: KuzuClient, qdrant_client: QdrantClient, embedder: Embedder):
        self.kg = kg_client
        self.qdrant = qdrant_client
        self.embedder = embedder

    async def ingest(self, project_id: UUID, repo_path: str) -> IngestionResult:
        changed_files = await self._get_changed_files(repo_path)
        results = []

        for batch in _batch(changed_files, size=20):
            batch_results = await asyncio.gather(*[
                self._process_file(project_id, repo_path, f) for f in batch
            ])
            results.extend(batch_results)

        stats = IngestionResult(
            files_processed=len(changed_files),
            nodes_upserted=sum(r.nodes for r in results),
            edges_upserted=sum(r.edges for r in results)
        )

        await db_pool.execute("""
            UPDATE coordinator.kg_ingestion_jobs
            SET status='completed', files_processed=$2,
                nodes_upserted=$3, edges_upserted=$4, completed_at=NOW()
            WHERE project_id=$1 AND status='running'
        """, project_id, stats.files_processed, stats.nodes_upserted, stats.edges_upserted)

        return stats

    async def _process_file(self, project_id: UUID, base_path: str, file_path: str):
        # 1. Parse AST with tree-sitter
        ast_nodes = await ast_parser.parse_file(os.path.join(base_path, file_path))

        # 2. Upsert File node
        file_id = _stable_id(project_id, file_path)
        await self.kg.execute("""
            MERGE (f:File {id: $file_id})
            SET f.project_id=$pid, f.path=$path,
                f.language=$lang, f.line_count=$lines
        """, file_id=file_id, pid=str(project_id), path=file_path,
             lang=ast_nodes.language, lines=ast_nodes.line_count)

        # 3. Upsert Classes and Methods
        for cls in ast_nodes.classes:
            class_id = _stable_id(project_id, file_path, cls.name)
            await self.kg.execute("""
                MERGE (c:Class {id: $id})
                SET c.name=$name, c.file_id=$fid, c.line_start=$ls, c.line_end=$le
                MERGE (c)-[:DEFINED_IN_FILE]->(f:File {id: $fid})
            """, id=class_id, name=cls.name, fid=file_id,
                 ls=cls.line_start, le=cls.line_end)

        # 4. Embed code chunks for vector search
        for chunk in ast_nodes.code_chunks:
            vector = await self.embedder.embed(chunk.text)
            await self.qdrant.upsert(
                collection_name="code_chunks",
                points=[PointStruct(
                    id=str(uuid4()),
                    vector=vector,
                    payload={
                        "project_id": str(project_id),
                        "file_path": file_path,
                        "chunk_type": chunk.type,
                        "start_line": chunk.start_line,
                        "end_line": chunk.end_line
                    }
                )]
            )
```

---

# PART 9: BRAIN — EXPERIENCE INTELLIGENCE

## 9.1 Purpose

The brain records every execution outcome, extracts patterns, and provides pattern-based recommendations to agents before they start new tasks. It answers: "What worked before in similar situations?"

## 9.2 Outcome Recording

```python
# brain/experience/recorder.py

class OutcomeRecorder:
    async def record(self, event: AgentResultEvent) -> UUID:
        # 1. Persist to PostgreSQL brain schema
        outcome_id = await db_pool.fetchval("""
            INSERT INTO brain.execution_outcomes
                (task_id, project_id, agent_type, task_type, model,
                 outcome, duration_s, tokens_used, cost_usd, error_category)
            VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10)
            RETURNING id
        """,
            UUID(event.task_id),
            UUID(event.project_id) if hasattr(event, 'project_id') else None,
            event.agent_type,
            event.task_type if hasattr(event, 'task_type') else 'unknown',
            event.model if hasattr(event, 'model') else 'unknown',
            event.status,
            event.duration_s if hasattr(event, 'duration_s') else None,
            event.tokens_used,
            event.cost_usd,
            _classify_error(event.error) if event.error else None
        )

        # 2. Embed outcome description for vector similarity
        outcome_text = f"{event.agent_type} {event.status}: {event.summary[:500]}"
        vector = await embedder.embed(outcome_text)
        qdrant_id = uuid4()
        await qdrant_client.upsert(
            collection_name="experience_outcomes",
            points=[PointStruct(
                id=str(qdrant_id),
                vector=vector,
                payload={
                    "agent_type": event.agent_type,
                    "outcome": event.status,
                    "outcome_id": str(outcome_id)
                }
            )]
        )

        # 3. Update Neo4j experience graph
        await neo4j_client.execute("""
            MERGE (o:Outcome {id: $oid})
            SET o.agent_type=$at, o.task_type=$tt, o.outcome=$outcome,
                o.tokens=$tokens, o.cost=$cost, o.recorded_at=datetime()
            WITH o
            MATCH (prev:Outcome {agent_type: $at})
            WHERE prev.id <> $oid
            WITH o, prev ORDER BY prev.recorded_at DESC LIMIT 5
            MERGE (prev)-[:FOLLOWED_BY]->(o)
        """,
            oid=str(outcome_id), at=event.agent_type,
            tt=getattr(event, 'task_type', 'unknown'),
            outcome=event.status, tokens=event.tokens_used, cost=event.cost_usd
        )

        return outcome_id
```

## 9.3 Pattern Detector

```python
# brain/patterns/detector.py

class PatternDetector:
    async def run_detection(self):
        # Called nightly by LearningAgent
        await self._detect_failure_patterns()
        await self._detect_success_patterns()
        await self._detect_cost_anomalies()
        await self._detect_performance_patterns()

    async def _detect_failure_patterns(self):
        # Find agent+task_type combinations with >30% failure rate in last 7 days
        rows = await db_pool.fetch("""
            SELECT agent_type, task_type, error_category,
                   COUNT(*) as total,
                   COUNT(*) FILTER (WHERE outcome = 'failed') as failures,
                   ROUND(100.0 * COUNT(*) FILTER (WHERE outcome = 'failed') / COUNT(*), 1) as failure_rate
            FROM brain.execution_outcomes
            WHERE recorded_at > NOW() - INTERVAL '7 days'
              AND outcome IS NOT NULL
            GROUP BY agent_type, task_type, error_category
            HAVING COUNT(*) >= 5
               AND COUNT(*) FILTER (WHERE outcome = 'failed') * 100 / COUNT(*) > 30
            ORDER BY failure_rate DESC
        """)

        for row in rows:
            pattern_name = f"high_failure_{row['agent_type']}_{row['task_type']}"
            recommendation = self._generate_failure_recommendation(row)
            confidence = min(0.95, 0.5 + (row['total'] - 5) * 0.01)

            await db_pool.execute("""
                INSERT INTO brain.patterns
                    (name, description, pattern_type, agent_type, task_type,
                     conditions, recommendation, confidence, observation_count, last_observed)
                VALUES ($1, $2, 'failure', $3, $4, $5, $6, $7, $8, NOW())
                ON CONFLICT (name) DO UPDATE
                SET confidence = EXCLUDED.confidence,
                    observation_count = EXCLUDED.observation_count,
                    recommendation = EXCLUDED.recommendation,
                    last_observed = NOW(),
                    updated_at = NOW()
            """,
                pattern_name,
                f"{row['agent_type']} fails {row['failure_rate']}% on {row['task_type']} tasks",
                row['agent_type'], row['task_type'],
                json.dumps({"error_category": row['error_category']}),
                recommendation, confidence, row['total']
            )

    def _generate_failure_recommendation(self, row: dict) -> str:
        recs = {
            "token_limit": "Break task into smaller subtasks; reduce context window size",
            "tool_error": "Add explicit error handling; increase sandbox timeout",
            "model_unavailable": "Add fallback model configuration",
            "context_overflow": "Use streaming response mode; summarize intermediate results",
            "test_failure": "Request QA agent review before committing",
        }
        return recs.get(row['error_category'],
                        f"Review {row['agent_type']} handling of {row['task_type']} tasks")
```

## 9.4 Recommendation API

```python
# brain/api/recommend.py

class RecommendRequest(BaseModel):
    task_type: str
    agent_type: str
    description: str = ""
    model: str = ""
    project_id: str = ""

class Recommendation(BaseModel):
    pattern_name: str
    recommendation: str
    confidence: float
    pattern_type: str
    rationale: str

@router.post("/recommend", response_model=list[Recommendation])
async def get_recommendations(req: RecommendRequest):
    # 1. Get pattern-based recommendations
    patterns = await db_pool.fetch("""
        SELECT name, description, pattern_type, recommendation, confidence
        FROM brain.patterns
        WHERE (agent_type IS NULL OR agent_type = $1)
          AND (task_type IS NULL OR task_type = $2)
          AND confidence > 0.6
          AND observation_count >= 3
        ORDER BY confidence DESC
        LIMIT 10
    """, req.agent_type, req.task_type)

    # 2. Find similar past outcomes via vector search
    if req.description:
        query_vec = await embedder.embed(req.description)
        similar = await qdrant_client.search(
            collection_name="experience_outcomes",
            query_vector=query_vec,
            query_filter=Filter(must=[
                FieldCondition(key="agent_type", match=MatchValue(value=req.agent_type)),
                FieldCondition(key="outcome", match=MatchValue(value="success"))
            ]),
            limit=5
        )
    else:
        similar = []

    recs = [
        Recommendation(
            pattern_name=p['name'],
            recommendation=p['recommendation'],
            confidence=float(p['confidence']),
            pattern_type=p['pattern_type'],
            rationale=p['description']
        )
        for p in patterns
    ]

    return recs
```

---

# PART 10: REST API SPECIFICATION

## 10.1 Projects API

```
POST   /api/v2/projects
  Request: {name, slug, description, repo_url?, repo_path?, settings?}
  Response: ProjectResponse (201)
  Auth: any authenticated user
  Validation: slug must be [a-z0-9-]+, unique

GET    /api/v2/projects
  Query: ?status=active&page=1&page_size=20
  Response: {items: ProjectResponse[], total: int, page: int}
  Auth: returns only projects user is member of; admin sees all

GET    /api/v2/projects/{id}
  Response: ProjectResponse with member_count
  Auth: must be project member or admin

PATCH  /api/v2/projects/{id}
  Request: {name?, description?, settings?, default_ai_provider?, unattended_mode?}
  Response: ProjectResponse
  Auth: project lead or admin

DELETE /api/v2/projects/{id}
  Response: 204
  Auth: project lead or admin
  Effect: sets status='deleted'; background job purges data after 30 days
```

## 10.2 Tasks API

```
POST   /api/v2/projects/{project_id}/tasks
  Request: TaskCreateRequest (type, title, description, acceptance_criteria,
                               priority, depends_on, deadline_at, metadata)
  Response: TaskResponse (201)
  Auth: developer or above
  Effect: INSERT task, publish tasks.created NATS event

GET    /api/v2/projects/{project_id}/tasks
  Query: ?status=running&assigned_agent=backend&page=1&page_size=50
  Response: {items: TaskResponse[], total: int}
  Auth: project member

GET    /api/v2/tasks/{id}
  Response: TaskDetailResponse with subtasks, approval_gates, agent_executions
  Auth: project member

PATCH  /api/v2/tasks/{id}
  Request: {priority?, metadata?, deadline_at?}   (title/description immutable after creation)
  Response: TaskResponse
  Auth: developer+; admin can modify any task

POST   /api/v2/tasks/{id}/cancel
  Request: {reason: string}
  Response: TaskResponse (status=cancelled)
  Auth: developer+
  Effect: cascade cancel to subtasks; publish tasks.cancelled

GET    /api/v2/tasks/{id}/lineage
  Response: {ancestors: TaskResponse[], descendants: TaskResponse[]}
  Auth: project member

GET    /api/v2/tasks/{id}/events
  Response: {events: EventResponse[]}
  Auth: project member
```

## 10.3 Approvals API

```
GET    /api/v2/approvals/pending
  Query: ?project_id=...
  Response: ApprovalGateResponse[] (sorted by risk_score DESC)
  Auth: developer+

GET    /api/v2/approvals/{id}
  Response: ApprovalGateDetailResponse with diff_content, test_summary
  Auth: project member

POST   /api/v2/approvals/{id}/approve
  Request: {notes?: string}
  Response: ApprovalGateResponse
  Auth: developer+
  Effect: status=approved, task unblocked, approval.granted published

POST   /api/v2/approvals/{id}/reject
  Request: {reason: string}
  Response: ApprovalGateResponse
  Auth: developer+
  Effect: status=rejected, task status=blocked, approval.rejected published

POST   /api/v2/approvals/{id}/extend
  Request: {additional_seconds: int}  (max: 3600)
  Response: ApprovalGateResponse
  Auth: developer+
  Effect: extends expires_at
```

## 10.4 Memory API

```
POST   /api/v2/memories
  Request: {project_id, memory_type, title, content, tags?, importance?, source_task_id?}
  Response: MemoryResponse (201)
  Auth: developer+

GET    /api/v2/memories
  Query: ?project_id=...&type=architecture&q=search+query&limit=20
  Response: MemorySearchResponse[]
  Auth: project member
  Behavior: q triggers hybrid search; without q returns recent memories

GET    /api/v2/memories/{id}
  Response: MemoryResponse
  Auth: project member

PATCH  /api/v2/memories/{id}
  Request: {importance?, tags?, expires_at?}
  Response: MemoryResponse
  Auth: developer+

DELETE /api/v2/memories/{id}
  Response: 204
  Auth: developer+
  Effect: DELETE from postgres, delete from Qdrant by id
```

## 10.5 Plugins API

```
GET    /api/v2/plugins
  Query: ?type=agent&enabled=true&scope=project&project_id=...
  Response: PluginResponse[]
  Auth: developer+

POST   /api/v2/plugins/install
  Request: {name, version, source_url, project_id?}
  Response: PluginResponse (201)
  Auth: admin (global) or project lead (project-scoped)
  Effect: download plugin.yaml, validate manifest, register agents/tools

PATCH  /api/v2/plugins/{id}
  Request: {enabled: bool}
  Response: PluginResponse
  Auth: admin or project lead

DELETE /api/v2/plugins/{id}
  Response: 204
  Auth: admin or project lead
  Effect: deregister agents/tools from registries
```

## 10.6 Models API

```
GET    /api/v2/models
  Response: {models: ModelInfo[], providers: ProviderStatus[]}
  Auth: developer+

POST   /api/v2/models/activate
  Request: {model_id: string, provider: string, api_key_masked: string}
  Response: {status: "ok"|"error", message: string}
  Auth: admin only
  Effect: validate credentials, store encrypted in settings

GET    /api/v2/models/cost-estimate
  Query: ?agent_type=backend&task_type=feature&complexity=high
  Response: {min_usd: float, max_usd: float, tokens: int}
  Auth: developer+
```

## 10.7 Response Models

```python
# coordinator/api/v2/schemas.py

class ProjectResponse(BaseModel):
    id: UUID
    name: str
    slug: str
    description: str
    owner_id: UUID
    status: str
    default_ai_provider: str
    unattended_mode: bool
    created_at: datetime
    updated_at: datetime

class TaskResponse(BaseModel):
    id: UUID
    project_id: UUID
    parent_id: UUID | None
    type: str
    title: str
    description: str
    status: str
    priority: int
    assigned_agent: str | None
    attempt_count: int
    actual_tokens: int | None
    cost_usd: float | None
    created_at: datetime
    updated_at: datetime
    started_at: datetime | None
    completed_at: datetime | None

class TaskDetailResponse(TaskResponse):
    acceptance_criteria: str
    depends_on: list[UUID]
    subtasks: list[TaskResponse]
    approval_gates: list[ApprovalGateResponse]
    recent_executions: list[AgentExecutionResponse]
    result_summary: str | None

class ApprovalGateResponse(BaseModel):
    id: UUID
    task_id: UUID
    task_title: str
    gate_type: str
    title: str
    description: str
    risk_score: int
    status: str
    expires_at: datetime
    decided_at: datetime | None
    approver_notes: str | None

class MemorySearchResponse(BaseModel):
    id: UUID
    project_id: UUID
    memory_type: str
    title: str
    content: str
    tags: list[str]
    importance: int
    relevance_score: float | None = None
    created_at: datetime
```

## 10.8 Error Response Format

```python
# All API errors return:
{
    "error": {
        "code": "TASK_NOT_FOUND",   # Machine-readable constant
        "message": "Task abc123 not found",   # Human-readable
        "field": null | "title",    # For validation errors
        "request_id": "req-xyz"     # X-Request-Id header value
    }
}

# HTTP Status Code conventions:
400 Bad Request      — Validation failure (Pydantic or business rule)
401 Unauthorized     — Missing or invalid auth
403 Forbidden        — Auth valid but insufficient role
404 Not Found        — Resource does not exist
409 Conflict         — Duplicate (unique constraint violation)
422 Unprocessable    — Request body parses but fields invalid
429 Too Many Requests — Rate limit
500 Internal Error   — Unhandled exception (always logged with trace_id)
503 Service Unavailable — Unhealthy dependency (DB down, NATS down)
```

---

# PART 11: DESKTOP APPLICATION

## 11.1 Purpose

The desktop application (PySide6) is the primary interface for developers working locally. It renders the project workspace, displays agent activity in real-time, presents approval gates for human decision, and provides the editor integration layer. It runs on Windows, macOS, and Linux. It communicates exclusively with the coordinator via REST and WebSocket.

## 11.2 Application Structure

```
desktop/
├── app/
│   ├── __init__.py
│   ├── application.py          # QApplication subclass, startup, theming
│   ├── main_window.py          # QMainWindow: project list + task panels
│   └── tray.py                 # System tray icon with quick-actions
├── windows/
│   ├── project_window.py       # Per-project workspace window
│   ├── task_window.py          # Task detail: description, subtasks, logs
│   ├── approval_window.py      # Diff viewer + approve/reject controls
│   ├── memory_window.py        # Memory browser + search
│   └── settings_window.py      # API key, model config, theme
├── panels/
│   ├── project_panel.py        # Left sidebar: project tree
│   ├── task_panel.py           # Center: task list with status indicators
│   ├── agent_panel.py          # Right: live agent activity stream
│   ├── cost_panel.py           # Cost budget widget (daily/project totals)
│   └── approval_panel.py       # Notification badge + pending gates list
├── widgets/
│   ├── diff_viewer.py          # Syntax-highlighted side-by-side diff (QTextEdit)
│   ├── log_viewer.py           # Scrolling log stream (QPlainTextEdit + QTimer)
│   ├── status_badge.py         # Colored status indicator chip
│   ├── cost_badge.py           # USD cost with trend indicator
│   ├── markdown_view.py        # Rendered markdown via QTextBrowser
│   └── agent_activity_card.py  # Agent type, model, current step, token count
├── services/
│   ├── api_client.py           # httpx async REST calls to coordinator
│   ├── ws_client.py            # WebSocket client, reconnect loop, event routing
│   └── credential_store.py     # OS keychain (keyring library) for JWT/API key
├── models/
│   ├── project_model.py        # Qt item model for project list
│   ├── task_model.py           # Qt item model for task table
│   └── approval_model.py       # Qt item model for approval gate list
├── signals/
│   └── bus.py                  # App-wide Qt signal bus (ws events -> UI updates)
├── config.py                   # Local config: coordinator URL, theme, window state
└── main.py                     # Entry point: QApplication.exec()
```

## 11.3 WebSocket Connection Manager

```python
# desktop/services/ws_client.py

class WebSocketClient(QObject):
    event_received = pyqtSignal(dict)
    connected = pyqtSignal()
    disconnected = pyqtSignal()

    def __init__(self, coordinator_url: str, project_id: str, token: str):
        super().__init__()
        self._url = f"ws://{coordinator_url}/ws/projects/{project_id}/events?token={token}"
        self._reconnect_delay = 1.0
        self._running = False

    def start(self):
        self._running = True
        self._thread = threading.Thread(target=self._run_loop, daemon=True)
        self._thread.start()

    def _run_loop(self):
        loop = asyncio.new_event_loop()
        loop.run_until_complete(self._connect_loop())

    async def _connect_loop(self):
        while self._running:
            try:
                async with websockets.connect(self._url, ping_interval=30) as ws:
                    self.connected.emit()
                    self._reconnect_delay = 1.0
                    async for message in ws:
                        event = json.loads(message)
                        self.event_received.emit(event)
            except (websockets.ConnectionClosed, OSError):
                self.disconnected.emit()
                await asyncio.sleep(self._reconnect_delay)
                self._reconnect_delay = min(self._reconnect_delay * 2, 60.0)

    def stop(self):
        self._running = False
```

## 11.4 Approval Gate UI

```python
# desktop/windows/approval_window.py

class ApprovalWindow(QDialog):
    def __init__(self, gate: dict, api_client: ApiClient, parent=None):
        super().__init__(parent)
        self._gate = gate
        self._api = api_client
        self._setup_ui()
        self._start_expiry_countdown()

    def _setup_ui(self):
        self.setWindowTitle(f"Review Required: {self._gate['title']}")
        self.setMinimumSize(1200, 800)
        layout = QVBoxLayout()

        # Header: risk score, type, expiry countdown
        header = self._build_header()
        layout.addWidget(header)

        # Diff viewer (left: before, right: after)
        diff_splitter = QSplitter(Qt.Horizontal)
        diff_splitter.addWidget(DiffViewer(self._gate.get('diff_content', '')))
        diff_splitter.addWidget(self._build_test_summary_panel())
        diff_splitter.setSizes([700, 500])
        layout.addWidget(diff_splitter, stretch=1)

        # Notes input
        self._notes_input = QPlainTextEdit()
        self._notes_input.setPlaceholderText("Optional notes for this decision...")
        self._notes_input.setMaximumHeight(80)
        layout.addWidget(QLabel("Decision Notes:"))
        layout.addWidget(self._notes_input)

        # Action buttons
        button_row = QHBoxLayout()
        reject_btn = QPushButton("Reject")
        reject_btn.setObjectName("danger")
        reject_btn.clicked.connect(self._reject)
        approve_btn = QPushButton("Approve")
        approve_btn.setObjectName("success")
        approve_btn.clicked.connect(self._approve)
        button_row.addWidget(reject_btn)
        button_row.addStretch()
        button_row.addWidget(approve_btn)
        layout.addLayout(button_row)

        self.setLayout(layout)

    def _start_expiry_countdown(self):
        self._countdown_timer = QTimer()
        self._countdown_timer.timeout.connect(self._update_countdown)
        self._countdown_timer.start(1000)

    def _update_countdown(self):
        expires = datetime.fromisoformat(self._gate['expires_at'])
        remaining = expires - datetime.utcnow()
        if remaining.total_seconds() <= 0:
            self.reject()
            return
        mins, secs = divmod(int(remaining.total_seconds()), 60)
        self._expiry_label.setText(f"Expires in: {mins:02d}:{secs:02d}")
        if remaining.total_seconds() < 300:
            self._expiry_label.setStyleSheet("color: red; font-weight: bold;")

    async def _approve(self):
        notes = self._notes_input.toPlainText().strip()
        result = await self._api.approve_gate(self._gate['id'], notes)
        if result.ok:
            self.accept()
        else:
            QMessageBox.warning(self, "Error", f"Failed to approve: {result.error}")

    async def _reject(self):
        notes = self._notes_input.toPlainText().strip()
        if not notes:
            QMessageBox.warning(self, "Reason Required", "Please provide a rejection reason.")
            return
        result = await self._api.reject_gate(self._gate['id'], notes)
        if result.ok:
            self.reject()
```

## 11.5 Real-Time Agent Activity Feed

```python
# desktop/panels/agent_panel.py

class AgentActivityPanel(QWidget):
    def __init__(self, parent=None):
        super().__init__(parent)
        self._cards: dict[str, AgentActivityCard] = {}
        self._layout = QVBoxLayout()
        self._layout.setAlignment(Qt.AlignTop)
        scroll = QScrollArea()
        scroll.setWidgetResizable(True)
        container = QWidget()
        container.setLayout(self._layout)
        scroll.setWidget(container)
        outer = QVBoxLayout()
        outer.addWidget(QLabel("Agent Activity"))
        outer.addWidget(scroll)
        self.setLayout(outer)

    @pyqtSlot(dict)
    def on_ws_event(self, event: dict):
        event_type = event.get("type")
        task_id = event.get("task_id")

        if event_type == "agent.started":
            card = AgentActivityCard(
                task_id=task_id,
                agent_type=event["agent_type"],
                model=event["model"],
                task_title=event["task_title"]
            )
            self._cards[task_id] = card
            self._layout.insertWidget(0, card)

        elif event_type == "agent.step":
            if card := self._cards.get(task_id):
                card.update_step(event["step"], event["tokens_so_far"])

        elif event_type in ("task.completed", "task.failed"):
            if card := self._cards.get(task_id):
                card.set_final_status(event_type == "task.completed")
                # Fade out after 30 seconds
                QTimer.singleShot(30000, lambda: self._remove_card(task_id))

    def _remove_card(self, task_id: str):
        if card := self._cards.pop(task_id, None):
            self._layout.removeWidget(card)
            card.deleteLater()
```

## 11.6 Desktop Configuration

```python
# desktop/config.py

DEFAULTS = {
    "coordinator_url": "http://localhost:8090",
    "theme": "dark",
    "font_size": 13,
    "log_level": "INFO",
    "auto_open_approval_gates": True,
    "approval_gate_notification_sound": True,
    "cost_budget_daily_usd": 10.0,
    "cost_budget_warn_pct": 80,
}

# Stored in:
# Windows: %APPDATA%\AIStudio\config.json
# macOS:   ~/Library/Application Support/AIStudio/config.json
# Linux:   ~/.config/aistudio/config.json
```

## 11.7 Desktop Testing

```
Unit tests (desktop/tests/unit/):
  test_ws_client.py       -> reconnect backoff logic, event routing
  test_diff_viewer.py     -> diff parsing, syntax highlighting correctness
  test_approval_window.py -> expiry countdown, form validation, button states
  test_task_model.py      -> Qt item model CRUD, sorting, filtering

Integration tests (desktop/tests/integration/):
  test_full_approval_flow.py  -> WS event -> approval window opens -> approve/reject -> gate updated
  test_ws_reconnect.py        -> Drop WS connection -> app reconnects -> resumes event stream

Manual test checklist (per release):
  [ ] Can submit a task and see it appear in task list
  [ ] Agent activity cards appear and update in real-time
  [ ] Approval window opens for high-risk gates automatically
  [ ] Diff viewer renders Python and TypeScript syntax correctly
  [ ] Cost badge increments as agent executions complete
  [ ] Dark/light theme toggle persists across restarts
  [ ] Window state (size, position) persists across restarts
```

---

# PART 12: WEB PLATFORM

## 12.1 Technology Stack

```
Framework:      React 19 + TypeScript 5.5
Build:          Vite 6
State:          Zustand 5
Server State:   TanStack Query v5
Routing:        TanStack Router v1
Real-time:      Browser WebSocket API (auto-reconnect hook)
UI Components:  shadcn/ui (Radix primitives + Tailwind)
Code Diff:      Monaco Editor (read-only diff view)
Charts:         Recharts (cost trend, agent performance)
Auth:           PKCE OAuth2 (browser-native, no backend session)
Deploy:         Nginx container, static SPA
```

## 12.2 Application Structure

```
webapp/src/
├── app/
│   ├── router.tsx              # TanStack Router: all route definitions
│   ├── providers.tsx           # QueryClient, auth, WS context
│   └── layout.tsx              # Shell: sidebar + main content area
├── pages/
│   ├── projects/
│   │   ├── list.tsx            # Project grid with search
│   │   └── detail.tsx          # Project workspace (tasks + agents)
│   ├── tasks/
│   │   ├── list.tsx            # Filterable task table
│   │   └── detail.tsx          # Task detail: subtasks, timeline, logs
│   ├── approvals/
│   │   ├── queue.tsx           # Pending gates priority queue
│   │   └── review.tsx          # Diff viewer + decision form
│   ├── memories/
│   │   ├── browser.tsx         # Memory search + list
│   │   └── detail.tsx          # Memory view/edit
│   ├── settings/
│   │   ├── models.tsx          # AI model configuration
│   │   ├── plugins.tsx         # Plugin management
│   │   └── team.tsx            # User/member management
│   └── auth/
│       └── callback.tsx        # PKCE OAuth2 callback handler
├── features/
│   ├── task-submit/
│   │   ├── TaskSubmitDrawer.tsx     # Slide-in task creation form
│   │   └── use-task-submit.ts       # Mutation hook
│   ├── approval-flow/
│   │   ├── ApprovalBanner.tsx       # Top banner: N pending gates
│   │   ├── ApprovalReviewModal.tsx  # Full-screen diff review
│   │   └── use-approval-polling.ts  # Poll when WS unavailable
│   ├── agent-activity/
│   │   ├── AgentActivityFeed.tsx    # Live scrolling activity log
│   │   └── AgentCard.tsx            # Per-task agent status card
│   └── cost-tracking/
│       ├── CostMeter.tsx            # Budget usage bar
│       └── CostHistory.tsx          # Daily cost recharts chart
├── hooks/
│   ├── use-ws.ts               # WebSocket connection with auto-reconnect
│   ├── use-projects.ts         # Projects query + mutation hooks
│   ├── use-tasks.ts            # Tasks query + mutation hooks
│   ├── use-approvals.ts        # Approvals query + mutation hooks
│   └── use-memories.ts         # Memories search hook
├── api/
│   ├── client.ts               # axios instance with JWT interceptor
│   ├── projects.ts             # Project API calls
│   ├── tasks.ts                # Task API calls
│   ├── approvals.ts            # Approval API calls
│   └── memories.ts             # Memory API calls
├── stores/
│   ├── auth.store.ts           # JWT, user profile, login/logout
│   └── ws.store.ts             # WS event queue, connection status
├── types/
│   └── api.ts                  # All API response TypeScript types
└── main.tsx                    # ReactDOM.createRoot
```

## 12.3 WebSocket Hook

```typescript
// webapp/src/hooks/use-ws.ts

type WsEvent = {
  type: string;
  task_id?: string;
  [key: string]: unknown;
};

export function useProjectWebSocket(projectId: string) {
  const { token } = useAuthStore();
  const [connected, setConnected] = useState(false);
  const eventListeners = useRef<Map<string, Set<(e: WsEvent) => void>>>(new Map());
  const ws = useRef<WebSocket | null>(null);
  const reconnectTimer = useRef<number>(0);
  const reconnectDelay = useRef(1000);

  const connect = useCallback(() => {
    const url = `${WS_BASE_URL}/ws/projects/${projectId}/events?token=${token}`;
    ws.current = new WebSocket(url);

    ws.current.onopen = () => {
      setConnected(true);
      reconnectDelay.current = 1000;
    };

    ws.current.onmessage = (e) => {
      const event: WsEvent = JSON.parse(e.data);
      eventListeners.current.get(event.type)?.forEach(fn => fn(event));
      eventListeners.current.get('*')?.forEach(fn => fn(event));
    };

    ws.current.onclose = () => {
      setConnected(false);
      reconnectTimer.current = window.setTimeout(() => {
        reconnectDelay.current = Math.min(reconnectDelay.current * 2, 60000);
        connect();
      }, reconnectDelay.current);
    };
  }, [projectId, token]);

  useEffect(() => {
    connect();
    return () => {
      clearTimeout(reconnectTimer.current);
      ws.current?.close();
    };
  }, [connect]);

  const subscribe = useCallback((type: string, fn: (e: WsEvent) => void) => {
    if (!eventListeners.current.has(type)) {
      eventListeners.current.set(type, new Set());
    }
    eventListeners.current.get(type)!.add(fn);
    return () => eventListeners.current.get(type)?.delete(fn);
  }, []);

  return { connected, subscribe };
}
```

## 12.4 Task Submit Form

```typescript
// webapp/src/features/task-submit/TaskSubmitDrawer.tsx

const TASK_TYPES = [
  'feature', 'bug', 'refactor', 'test', 'deploy',
  'security', 'documentation', 'architecture', 'release', 'research'
] as const;

interface TaskSubmitForm {
  type: typeof TASK_TYPES[number];
  title: string;
  description: string;
  acceptance_criteria: string;
  priority: number;
}

export function TaskSubmitDrawer({ projectId, open, onClose }: Props) {
  const { mutateAsync: createTask, isPending } = useTaskSubmit(projectId);
  const form = useForm<TaskSubmitForm>({
    defaultValues: { type: 'feature', priority: 50 },
    resolver: zodResolver(taskSubmitSchema)
  });

  const onSubmit = async (data: TaskSubmitForm) => {
    await createTask(data);
    onClose();
    toast.success('Task submitted — AI agents will begin shortly');
  };

  return (
    <Sheet open={open} onOpenChange={onClose}>
      <SheetContent side="right" className="w-[600px]">
        <SheetHeader>
          <SheetTitle>Submit New Task</SheetTitle>
          <SheetDescription>
            Describe what you want to build. Be specific about acceptance criteria.
          </SheetDescription>
        </SheetHeader>
        <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4 mt-6">
          <div className="grid grid-cols-2 gap-4">
            <FormField name="type" label="Task Type" control={form.control}>
              {(field) => (
                <Select value={field.value} onValueChange={field.onChange}>
                  <SelectTrigger><SelectValue /></SelectTrigger>
                  <SelectContent>
                    {TASK_TYPES.map(t => (
                      <SelectItem key={t} value={t}>{t}</SelectItem>
                    ))}
                  </SelectContent>
                </Select>
              )}
            </FormField>
            <FormField name="priority" label="Priority (0-100)" control={form.control}>
              {(field) => (
                <Slider min={0} max={100} step={5}
                        value={[field.value]} onValueChange={([v]) => field.onChange(v)} />
              )}
            </FormField>
          </div>
          <FormField name="title" label="Title" control={form.control}>
            {(field) => <Input {...field} placeholder="Brief task title" />}
          </FormField>
          <FormField name="description" label="Description" control={form.control}>
            {(field) => (
              <Textarea {...field} rows={8}
                        placeholder="Describe what to build in detail..." />
            )}
          </FormField>
          <FormField name="acceptance_criteria" label="Acceptance Criteria" control={form.control}>
            {(field) => (
              <Textarea {...field} rows={4}
                        placeholder="How will you know this is done?" />
            )}
          </FormField>
          <Button type="submit" disabled={isPending} className="w-full">
            {isPending ? 'Submitting...' : 'Submit to AI Agents'}
          </Button>
        </form>
      </SheetContent>
    </Sheet>
  );
}
```

## 12.5 Approval Review Modal

```typescript
// webapp/src/features/approval-flow/ApprovalReviewModal.tsx

export function ApprovalReviewModal({ gate, onClose }: Props) {
  const [notes, setNotes] = useState('');
  const { mutateAsync: approve } = useApprove(gate.id);
  const { mutateAsync: reject } = useReject(gate.id);
  const [timeLeft, setTimeLeft] = useState(0);

  useEffect(() => {
    const expires = new Date(gate.expires_at).getTime();
    const tick = setInterval(() => {
      setTimeLeft(Math.max(0, Math.floor((expires - Date.now()) / 1000)));
    }, 1000);
    return () => clearInterval(tick);
  }, [gate.expires_at]);

  const mins = Math.floor(timeLeft / 60);
  const secs = timeLeft % 60;
  const urgency = timeLeft < 300 ? 'destructive' : timeLeft < 600 ? 'warning' : 'default';

  return (
    <Dialog open onOpenChange={onClose}>
      <DialogContent className="max-w-[1400px] h-[90vh] flex flex-col">
        <DialogHeader>
          <div className="flex items-center gap-3">
            <RiskBadge score={gate.risk_score} />
            <DialogTitle>{gate.title}</DialogTitle>
            <Badge variant={urgency}>
              {timeLeft === 0 ? 'Expired' : `${mins}:${secs.toString().padStart(2,'0')} remaining`}
            </Badge>
          </div>
        </DialogHeader>

        <div className="flex-1 overflow-hidden grid grid-cols-2 gap-4">
          <div className="flex flex-col">
            <h3 className="font-semibold mb-2">Code Changes</h3>
            <div className="flex-1 overflow-hidden rounded border">
              <MonacoDiffEditor
                original={gate.diff_before ?? ''}
                modified={gate.diff_after ?? gate.diff_content ?? ''}
                language="python"
                options={{ readOnly: true, renderSideBySide: true }}
                height="100%"
              />
            </div>
          </div>
          <div className="flex flex-col gap-4 overflow-y-auto">
            <TestSummaryCard summary={gate.test_summary} />
            <MarkdownCard title="Summary" content={gate.description} />
          </div>
        </div>

        <div className="space-y-3 pt-3 border-t">
          <Textarea
            value={notes}
            onChange={e => setNotes(e.target.value)}
            placeholder="Decision notes (required to reject)"
            rows={2}
          />
          <div className="flex gap-3">
            <Button variant="destructive" className="flex-1"
                    onClick={() => reject({ reason: notes })}
                    disabled={!notes.trim()}>
              Reject
            </Button>
            <Button variant="default" className="flex-1"
                    onClick={() => approve({ notes })}>
              Approve Changes
            </Button>
          </div>
        </div>
      </DialogContent>
    </Dialog>
  );
}
```

## 12.6 Build and Deploy

```
Build:
  npm run build       -> dist/ (static files, Vite output)
  npm run typecheck   -> tsc --noEmit (0 errors required)
  npm run lint        -> eslint src/ --max-warnings 0
  npm test            -> vitest run (unit + component tests)

Docker:
  FROM node:22-slim AS builder
  WORKDIR /app
  COPY package*.json .
  RUN npm ci
  COPY . .
  RUN npm run build

  FROM nginx:1.27-alpine
  COPY --from=builder /app/dist /usr/share/nginx/html
  COPY nginx.conf /etc/nginx/conf.d/default.conf
  EXPOSE 3000

nginx.conf:
  server {
    listen 3000;
    root /usr/share/nginx/html;
    index index.html;
    location /api/ { proxy_pass http://coordinator:8090; }
    location /ws/  { proxy_pass http://coordinator:8090; proxy_http_version 1.1;
                     proxy_set_header Upgrade $http_upgrade;
                     proxy_set_header Connection "upgrade"; }
    location /     { try_files $uri $uri/ /index.html; }
  }

Environment variables (injected at container startup via envsubst):
  VITE_API_BASE_URL    = http://coordinator:8090
  VITE_WS_BASE_URL     = ws://coordinator:8090
  VITE_OIDC_CLIENT_ID  = aistudio-webapp
  VITE_OIDC_AUTHORITY  = https://auth.company.com
```

## 12.7 Web Platform Testing

```
Unit tests (vitest):
  tests/hooks/use-ws.test.ts         -> connect, disconnect, reconnect, event routing
  tests/hooks/use-tasks.test.ts      -> CRUD operations, optimistic updates
  tests/components/approval.test.tsx -> form validation, countdown, button states

Component tests (Testing Library):
  tests/features/TaskSubmitDrawer.test.tsx  -> form submit, validation errors
  tests/features/ApprovalModal.test.tsx     -> diff renders, approve/reject flows

E2E tests (Playwright):
  e2e/task-flow.spec.ts   -> submit task -> see in list -> watch agent progress
  e2e/approval.spec.ts    -> pending gate appears -> open -> approve -> task continues
  e2e/auth.spec.ts        -> login -> authenticated routes -> logout -> redirect

Coverage: 70% statement coverage minimum (lower than backend; UI visual testing fills gap).
```

---

# PART 13: MOBILE APPLICATION

## 13.1 Scope and Deferred Features

The mobile application (iOS and Android) is **Milestone 3** of the product roadmap. It provides read access and approval decisions only. It does NOT support task creation, diff review, or model configuration from mobile.

**In scope for Milestone 3:**
- View project list and task status
- Approve or reject pending gates (with mandatory notes for rejection)
- Receive push notifications for pending approvals
- View agent activity feed (read-only)
- View memory browser (read-only)
- Cost summary dashboard

**Out of scope for Milestone 3 (deferred to Milestone 4+):**
- Task creation from mobile
- Code diff review on mobile
- Plugin management
- Model configuration

## 13.2 Technology

```
Framework:     React Native 0.75 + Expo SDK 52
State:         Zustand + TanStack Query
Navigation:    Expo Router v4 (file-based)
Push Notify:   Expo Notifications + APNs + FCM
Auth:          expo-auth-session (PKCE OAuth2)
Storage:       expo-secure-store (JWT token)
```

## 13.3 Push Notification Architecture

```
1. Device registers for push token (APNs / FCM) via Expo
2. Mobile app sends push token to coordinator:
   POST /api/v2/users/me/push-token
   Body: {token: "ExponentPushToken[...]", platform: "ios"|"android"}

3. Stored in: coordinator.users.metadata.push_tokens[]

4. When approval gate created:
   coordinator -> push_notification_service.send(
     tokens=project_members_push_tokens,
     title="Review Required",
     body=f"{gate.gate_type.title()}: {gate.title[:50]}",
     data={gate_id, task_id, risk_score}
   )

5. User taps notification -> app deep links to approval screen

Push service: Expo Push API (server-side endpoint)
Fallback: in-app polling every 60s when app is foregrounded
```

## 13.4 Mobile API Client

```typescript
// mobile/src/lib/api.ts

const api = axios.create({
  baseURL: COORDINATOR_BASE_URL,
  timeout: 30000
});

api.interceptors.request.use(async (config) => {
  const token = await SecureStore.getItemAsync('jwt_token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

api.interceptors.response.use(
  response => response,
  async (error) => {
    if (error.response?.status === 401) {
      await SecureStore.deleteItemAsync('jwt_token');
      router.replace('/auth/login');
    }
    return Promise.reject(error);
  }
);
```

## 13.5 Mobile Testing

```
Unit:     Jest + React Native Testing Library
E2E:      Detox (iOS Simulator + Android Emulator)

E2E test scenarios:
  - Login with OAuth2 PKCE -> JWT stored -> dashboard visible
  - Receive push notification -> tap -> approval screen opens
  - Approve gate with notes -> confirm gate status updates
  - Reject gate without notes -> validation error shown
  - Background app -> push notification received -> badge count increments
```

---

# PART 14: MARKETPLACE

## 14.1 Scope

The marketplace is **Milestone 4**. It is a plugin distribution system where the AI Studio community can publish, discover, and install:
- Custom agent types (implementing BaseAgent)
- Custom tool types (implementing BaseTool)
- Workflow templates (YAML DAG definitions)
- Prompt templates (system prompts for existing agents)
- Memory type extensions

## 14.2 Plugin Manifest Format

```yaml
# plugin.yaml — Required in root of every plugin package

name: stripe-payments-agent
version: 1.2.0
description: "Specialized agent for Stripe payment integration tasks"
author: "Example Corp"
license: MIT
min_platform_version: "2.0.0"
homepage: https://github.com/example/stripe-payments-agent

agents:
  - type: stripe-backend
    class: stripe_agent.StripeBackendAgent
    capabilities:
      - feature
      - bug
    preferred_models:
      - claude-sonnet-4-6
    description: "Stripe API integration, webhook handling, payment flows"

tools:
  - name: stripe_api_call
    class: stripe_agent.tools.StripeApiTool
    description: "Execute Stripe API calls with proper error handling"

workflow_templates:
  - name: payment-integration
    file: workflows/payment_integration.yaml
    description: "Full Stripe payment flow: checkout, webhooks, refunds"

permissions:
  - tool_access:
      - shell_executor   # Allow shell commands
      - http_client      # Allow outbound HTTP (to Stripe API)
      - file_writer      # Allow file writes
  - network_allow:
      - api.stripe.com
      - hooks.stripe.com

settings_schema:
  stripe_api_key:
    type: string
    secret: true
    description: "Stripe secret API key (sk_live_...)"
  stripe_webhook_secret:
    type: string
    secret: true
    description: "Stripe webhook signing secret"
```

## 14.3 Plugin Installation Flow

```python
# coordinator/services/plugin_service.py

class PluginInstaller:
    async def install(
        self,
        name: str,
        version: str,
        source_url: str,
        project_id: UUID | None,
        installed_by: UUID
    ) -> PluginRecord:
        # 1. Download and validate manifest
        manifest_raw = await self._download_manifest(source_url)
        manifest = PluginManifest.model_validate(yaml.safe_load(manifest_raw))
        self._validate_manifest(manifest)

        # 2. Security check: verify all requested permissions
        for perm in manifest.permissions:
            if not self._is_permission_allowed(perm):
                raise PluginInstallError(f"Permission not allowed: {perm}")

        # 3. Register in database
        plugin_id = await db_pool.fetchval("""
            INSERT INTO coordinator.plugins
                (name, version, type, manifest, scope, project_id, installed_by)
            VALUES ($1, $2, $3, $4, $5, $6, $7)
            RETURNING id
        """,
            manifest.name, manifest.version, 'agent_bundle',
            json.dumps(manifest.dict()),
            'project' if project_id else 'global',
            project_id, installed_by
        )

        # 4. Download plugin package and extract to plugin path
        plugin_dir = await self._download_and_extract(source_url, manifest.name, manifest.version)

        # 5. Notify agent-runtime of new plugin via NATS
        await nats.publish("plugins.installed", PluginInstalledEvent(
            plugin_id=str(plugin_id),
            plugin_dir=str(plugin_dir),
            manifest=manifest.dict()
        ).model_dump_json().encode())

        return await self._get_plugin(plugin_id)
```

## 14.4 Marketplace Repository Structure

```
marketplace/
├── registry/
│   ├── index.json              # Public registry index (all published plugins)
│   └── plugins/{name}/{version}/
│       ├── plugin.yaml         # Canonical manifest
│       ├── checksums.sha256    # Package integrity
│       └── metadata.json       # Downloads, ratings, publisher
├── api/
│   ├── search.py               # GET /marketplace/search?q=stripe&type=agent
│   ├── publish.py              # POST /marketplace/publish (authenticated)
│   ├── versions.py             # GET /marketplace/plugins/{name}/versions
│   └── download.py             # GET /marketplace/plugins/{name}/{version}/download
├── validation/
│   ├── manifest_validator.py   # Schema + security permission validation
│   └── package_scanner.py      # Static analysis: no network calls in agent code
├── frontend/
│   └── (React app for browsing, searching, plugin detail pages)
└── main.py
```

## 14.5 Marketplace Security Model

```
SECURITY RULES FOR PLUGIN VALIDATION:
1. All plugin code runs in agent-runtime sandbox (same Linux namespace/gVisor restrictions)
2. Network allow-list in plugin.yaml is merged with tool-runtime allow-list (tool-runtime wins)
3. Plugin agents CANNOT bypass human approval gates (approval logic in orchestrator, not agents)
4. Plugin package checksums verified against registry before extraction
5. Plugin code scanned for: subprocess.run, os.system, __import__, eval, exec
6. Secret settings (type: string, secret: true) stored encrypted in coordinator settings
7. Plugins installed per-project are isolated to that project's agent pool
8. Platform version constraint enforced: min_platform_version must be <= running version
9. Plugin author identity verified via GPG signature (Milestone 4.1)
10. Plugin rating system: auto-suspend plugins with > 5% failure rate across installations
```

---

# PART 15: DEPLOYMENT

## 15.1 Deployment Modes

```
Laptop    — Single developer; all services on one machine via Docker Compose
Team      — Shared server; Docker Compose; SQLite replaced with PostgreSQL
Enterprise — Kubernetes cluster; full HA; Terraform infrastructure
Cloud-K8s — Managed K8s (EKS/GKE/AKS); Terraform provisioned
```

## 15.2 Laptop Mode — docker-compose.laptop.yaml

```yaml
version: "3.9"

x-common: &common
  restart: unless-stopped
  logging:
    driver: json-file
    options: {max-size: "50m", max-file: "3"}

services:
  postgres:
    image: postgres:15-alpine
    <<: *common
    environment:
      POSTGRES_USER: aistudio
      POSTGRES_PASSWORD: ${PG_PASSWORD:-devpassword}
      POSTGRES_DB: aistudio
    volumes:
      - pg_data:/var/lib/postgresql/data
      - ./sql/init/:/docker-entrypoint-initdb.d/
    ports: ["127.0.0.1:5432:5432"]
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "aistudio"]
      interval: 5s
      timeout: 5s
      retries: 10

  redis:
    image: redis:7-alpine
    <<: *common
    command: redis-server --save 60 1 --loglevel warning
    volumes: [redis_data:/data]
    ports: ["127.0.0.1:6379:6379"]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s

  qdrant:
    image: qdrant/qdrant:v1.12
    <<: *common
    volumes: [qdrant_data:/qdrant/storage]
    ports: ["127.0.0.1:6333:6333"]
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:6333/healthz"]
      interval: 10s

  nats:
    image: nats:2.10-alpine
    <<: *common
    command: -js -sd /data -c /etc/nats/nats.conf
    volumes:
      - nats_data:/data
      - ./deployment/nats/nats.conf:/etc/nats/nats.conf
    ports: ["127.0.0.1:4222:4222", "127.0.0.1:8222:8222"]
    healthcheck:
      test: ["CMD", "nats-server", "--signal", "status"]
      interval: 10s

  ollama:
    image: ollama/ollama:latest
    <<: *common
    volumes: [ollama_data:/root/.ollama]
    ports: ["127.0.0.1:11434:11434"]
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]

  auth:
    image: registry.aistudio.internal/auth:${VERSION:-latest}
    <<: *common
    environment:
      PG_DSN: postgresql://aistudio:${PG_PASSWORD:-devpassword}@postgres:5432/aistudio
      JWT_PRIVATE_KEY_FILE: /run/secrets/jwt_private_key
      REDIS_URL: redis://redis:6379/1
    secrets: [jwt_private_key]
    depends_on:
      postgres: {condition: service_healthy}
    ports: ["127.0.0.1:8093:8093"]

  knowledge-engine:
    image: registry.aistudio.internal/knowledge-engine:${VERSION:-latest}
    <<: *common
    environment:
      PG_DSN: postgresql://aistudio:${PG_PASSWORD:-devpassword}@postgres:5432/aistudio
      QDRANT_URL: http://qdrant:6333
      KUZU_DB_PATH: /data/kuzu
      EMBEDDER: ollama
      OLLAMA_URL: http://ollama:11434
    volumes: [kuzu_data:/data/kuzu]
    depends_on:
      postgres: {condition: service_healthy}
      qdrant: {condition: service_healthy}
    ports: ["127.0.0.1:8091:8091"]

  brain:
    image: registry.aistudio.internal/brain:${VERSION:-latest}
    <<: *common
    environment:
      PG_DSN: postgresql://aistudio:${PG_PASSWORD:-devpassword}@postgres:5432/aistudio
      QDRANT_URL: http://qdrant:6333
      NEO4J_URL: ""   # Uses PostgreSQL for experience graph in laptop mode
    depends_on:
      postgres: {condition: service_healthy}
      qdrant: {condition: service_healthy}
    ports: ["127.0.0.1:8094:8094"]

  coordinator:
    image: registry.aistudio.internal/coordinator:${VERSION:-latest}
    <<: *common
    environment:
      PG_DSN: postgresql://aistudio:${PG_PASSWORD:-devpassword}@postgres:5432/aistudio
      REDIS_URL: redis://redis:6379/0
      NATS_URL: nats://nats:4222
      JWT_PUBLIC_KEY_FILE: /run/secrets/jwt_public_key
      KNOWLEDGE_ENGINE_URL: http://knowledge-engine:8091
      AUTH_SERVICE_URL: http://auth:8093
      DEV_MODE: "true"
    secrets: [jwt_public_key]
    depends_on:
      postgres: {condition: service_healthy}
      redis: {condition: service_healthy}
      nats: {condition: service_healthy}
      auth: {condition: service_started}
      knowledge-engine: {condition: service_started}
    ports: ["127.0.0.1:8090:8090"]

  orchestrator:
    image: registry.aistudio.internal/orchestrator:${VERSION:-latest}
    <<: *common
    environment:
      PG_DSN: postgresql://aistudio:${PG_PASSWORD:-devpassword}@postgres:5432/aistudio
      NATS_URL: nats://nats:4222
      REDIS_URL: redis://redis:6379/2
    depends_on:
      coordinator: {condition: service_started}
      nats: {condition: service_healthy}

  agent-runtime:
    image: registry.aistudio.internal/agent-runtime:${VERSION:-latest}
    <<: *common
    environment:
      PG_DSN: postgresql://aistudio:${PG_PASSWORD:-devpassword}@postgres:5432/aistudio
      NATS_URL: nats://nats:4222
      KNOWLEDGE_ENGINE_URL: http://knowledge-engine:8091
      BRAIN_URL: http://brain:8094
      TOOL_RUNTIME_URL: http://tool-runtime:8092
      ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY:-}
      OPENAI_API_KEY: ${OPENAI_API_KEY:-}
      OLLAMA_URL: http://ollama:11434
      ENABLED_PROVIDERS: "${ENABLED_PROVIDERS:-ollama}"
    depends_on:
      coordinator: {condition: service_started}
      knowledge-engine: {condition: service_started}

  tool-runtime:
    image: registry.aistudio.internal/tool-runtime:${VERSION:-latest}
    <<: *common
    privileged: true   # Required for user namespace creation (sandbox)
    environment:
      SANDBOX_MODE: linux_namespace
      MAX_CONCURRENT_SANDBOXES: "4"
      SANDBOX_TIMEOUT_SECONDS: "300"
    volumes:
      - workspace_data:/workspace
    ports: ["127.0.0.1:8092:8092"]

  webapp:
    image: registry.aistudio.internal/webapp:${VERSION:-latest}
    <<: *common
    ports: ["127.0.0.1:3000:3000"]
    depends_on:
      coordinator: {condition: service_started}

volumes:
  pg_data:
  redis_data:
  qdrant_data:
  nats_data:
  kuzu_data:
  ollama_data:
  workspace_data:

secrets:
  jwt_private_key:
    file: ./secrets/jwt_private_key.pem
  jwt_public_key:
    file: ./secrets/jwt_public_key.pem
```

## 15.3 Kubernetes Helm — Coordinator Chart

```yaml
# deployment/k8s/helm/coordinator/values.yaml

replicaCount: 2

image:
  repository: registry.aistudio.internal/coordinator
  pullPolicy: IfNotPresent
  tag: ""   # Overridden at deploy time with GIT_SHA

service:
  type: ClusterIP
  port: 8090

ingress:
  enabled: true
  className: nginx
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
  hosts:
    - host: api.aistudio.company.com
      paths: [{path: /, pathType: Prefix}]
  tls:
    - secretName: aistudio-tls
      hosts: [api.aistudio.company.com]

resources:
  requests: {cpu: 500m, memory: 512Mi}
  limits: {cpu: 2, memory: 2Gi}

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

podDisruptionBudget:
  minAvailable: 1

env:
  PG_POOL_MIN: "5"
  PG_POOL_MAX: "20"
  REDIS_POOL_MAX: "10"
  LOG_LEVEL: "INFO"

envFromSecrets:
  - secretName: coordinator-secrets
    keys: [PG_DSN, REDIS_URL, NATS_URL, JWT_PUBLIC_KEY]
```

## 15.4 Kubernetes Helm — Agent Runtime HPA

```yaml
# deployment/k8s/helm/agent-runtime/values.yaml

replicaCount: 2

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 20
  # Scale on NATS consumer lag (custom metric via KEDA)
  keda:
    enabled: true
    triggers:
      - type: nats-jetstream
        metadata:
          natsServerMonitoringEndpoint: "nats-monitor.aistudio:8222"
          account: aistudio
          stream: AGENT
          consumer: pool.backend
          lagThreshold: "50"

resources:
  requests: {cpu: 1, memory: 2Gi}
  limits: {cpu: 4, memory: 8Gi}
```

## 15.5 One-Command Laptop Install

```bash
#!/usr/bin/env bash
# deployment/scripts/install.sh

set -euo pipefail

echo "=== AI Studio 2.0 Laptop Install ==="

# Prerequisites
command -v docker >/dev/null 2>&1 || { echo "Docker required"; exit 1; }
command -v docker-compose >/dev/null 2>&1 || { echo "Docker Compose required"; exit 1; }

# Create secrets dir and generate JWT keys
mkdir -p secrets
if [ ! -f secrets/jwt_private_key.pem ]; then
    openssl genrsa -out secrets/jwt_private_key.pem 4096
    openssl rsa -in secrets/jwt_private_key.pem -pubout -out secrets/jwt_public_key.pem
    echo "JWT keys generated"
fi

# Create .env if not exists
if [ ! -f .env ]; then
    cat > .env <<'ENVEOF'
PG_PASSWORD=changeme_devonly
ANTHROPIC_API_KEY=
OPENAI_API_KEY=
ENABLED_PROVIDERS=ollama
VERSION=latest
ENVEOF
    echo ".env created — add API keys if you want cloud AI providers"
fi

# Pull and start
docker-compose -f docker/compose.laptop.yaml pull
docker-compose -f docker/compose.laptop.yaml up -d

echo ""
echo "Waiting for coordinator to be healthy..."
for i in $(seq 1 30); do
    if curl -sf http://localhost:8090/healthz/ready > /dev/null 2>&1; then
        echo "AI Studio 2.0 is ready!"
        echo "  Web UI:       http://localhost:3000"
        echo "  API:          http://localhost:8090"
        echo "  API Docs:     http://localhost:8090/docs"
        echo "  NATS Monitor: http://localhost:8222"
        exit 0
    fi
    sleep 3
done
echo "Startup timed out — check: docker-compose logs"
exit 1
```

## 15.6 Upgrade Procedure

```bash
#!/usr/bin/env bash
# deployment/scripts/upgrade.sh

VERSION=${1:?Usage: upgrade.sh <version>}

echo "Upgrading AI Studio to $VERSION"

# 1. Pull new images
docker-compose -f docker/compose.laptop.yaml pull

# 2. Run migrations first (coordinator runs Alembic on startup)
docker-compose -f docker/compose.laptop.yaml up -d coordinator
sleep 10
docker-compose -f docker/compose.laptop.yaml exec coordinator alembic upgrade head

# 3. Rolling restart: one service at a time
for svc in orchestrator agent-runtime brain knowledge-engine auth webapp tool-runtime; do
    docker-compose -f docker/compose.laptop.yaml up -d --no-deps "$svc"
    sleep 5
    STATUS=$(docker-compose -f docker/compose.laptop.yaml ps --format json "$svc" | python3 -c "import sys,json; print(json.load(sys.stdin)[0]['Health'])")
    if [ "$STATUS" != "healthy" ] && [ "$STATUS" != "running" ]; then
        echo "Service $svc failed to start — rollback?"
        exit 1
    fi
    echo "  $svc: ok"
done

echo "Upgrade to $VERSION complete"
```

## 15.7 Terraform AWS Outline

```hcl
# terraform/aws/main.tf

module "vpc" {
  source = "./modules/vpc"
  cidr   = "10.0.0.0/16"
}

module "eks" {
  source                = "./modules/eks"
  cluster_name          = "aistudio-prod"
  kubernetes_version    = "1.31"
  node_instance_type    = "m6i.2xlarge"
  min_nodes             = 3
  max_nodes             = 20
}

module "rds" {
  source                = "./modules/rds"
  engine_version        = "15.6"
  instance_class        = "db.r6g.2xlarge"
  multi_az              = true
  storage_encrypted     = true
  deletion_protection   = true
  backup_retention_days = 14
}

module "elasticache" {
  source         = "./modules/elasticache"
  engine_version = "7.1"
  node_type      = "cache.r7g.large"
  num_clusters   = 2
}

module "s3" {
  source               = "./modules/s3"
  versioning           = true
  lifecycle_rules = {
    artifacts = {transition_to_glacier_days = 90}
    pg_backups = {expiry_days = 30}
  }
}
```

---

# PART 16: MONITORING

## 16.1 Observability Stack

```
Metrics:    Prometheus 2.54 + Grafana 11
Tracing:    OpenTelemetry -> Jaeger (or Tempo)
Logging:    Structured JSON -> Loki -> Grafana
Alerting:   Alertmanager -> PagerDuty (on-call) + Slack (non-critical)
```

Every AI Studio service exports Prometheus metrics at `/metrics`. All log output is JSON on stdout. Traces are emitted via OpenTelemetry SDK with automatic instrumentation for FastAPI and asyncpg.

## 16.2 Prometheus Metrics — Per Service

```python
# Defined in: each service/metrics.py

from prometheus_client import Counter, Histogram, Gauge, Summary

# coordinator
REQUESTS_TOTAL = Counter(
    'coordinator_requests_total', 'HTTP requests',
    labelnames=['method', 'path', 'status']
)
REQUEST_DURATION = Histogram(
    'coordinator_request_duration_seconds', 'HTTP request latency',
    labelnames=['method', 'path'],
    buckets=[0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0]
)
ACTIVE_WS_CONNECTIONS = Gauge(
    'coordinator_ws_connections_active', 'Active WebSocket connections'
)
APPROVAL_GATES_PENDING = Gauge(
    'coordinator_approval_gates_pending', 'Pending approval gates'
)
TASKS_CREATED_TOTAL = Counter('coordinator_tasks_created_total', 'Tasks submitted')

# orchestrator
TASKS_DISPATCHED = Counter(
    'orchestrator_tasks_dispatched_total', 'Tasks dispatched to agents',
    labelnames=['agent_type']
)
TASKS_RETRIED = Counter(
    'orchestrator_tasks_retried_total', 'Task retry count',
    labelnames=['agent_type', 'reason']
)
NATS_CONSUMER_LAG = Gauge(
    'orchestrator_nats_consumer_lag', 'NATS consumer pending message count',
    labelnames=['stream', 'consumer']
)

# agent-runtime
AGENT_EXECUTIONS_TOTAL = Counter(
    'agent_executions_total', 'Total agent task executions',
    labelnames=['agent_type', 'status', 'model', 'provider']
)
AGENT_EXECUTION_DURATION = Histogram(
    'agent_execution_duration_seconds', 'Agent task execution time',
    labelnames=['agent_type', 'model'],
    buckets=[10, 30, 60, 120, 300, 600, 1200]
)
AGENT_TOKENS_USED = Counter(
    'agent_tokens_total', 'Total tokens consumed',
    labelnames=['agent_type', 'model', 'provider', 'token_type']
)
AGENT_COST_USD = Counter(
    'agent_cost_usd_total', 'Total cost in USD',
    labelnames=['agent_type', 'model', 'provider', 'project_id']
)
ACTIVE_AGENTS = Gauge(
    'agent_runtime_active_count', 'Currently executing agents',
    labelnames=['agent_type']
)
TOOL_CALLS_TOTAL = Counter(
    'tool_calls_total', 'Total tool invocations',
    labelnames=['tool_name', 'success']
)

# knowledge-engine
MEMORY_WRITES_TOTAL = Counter(
    'ke_memory_writes_total', 'Memory writes',
    labelnames=['memory_type']
)
SEARCH_LATENCY = Histogram(
    'ke_search_duration_seconds', 'Hybrid search latency',
    labelnames=['mode'],   # qdrant | fts | rrf
    buckets=[0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.0]
)
EMBEDDING_LATENCY = Histogram(
    'ke_embedding_duration_seconds', 'Vector embedding latency',
    buckets=[0.05, 0.1, 0.25, 0.5, 1.0, 2.0]
)
QDRANT_POINTS_TOTAL = Gauge(
    'ke_qdrant_points_total', 'Qdrant vector count per collection',
    labelnames=['collection']
)
```

## 16.3 Grafana Dashboards

### Dashboard 1: Platform Overview

```
Row 1: SLIs
  - API Success Rate: 1 - (rate(coordinator_requests_total{status=~"5.."}[5m]) / rate(coordinator_requests_total[5m]))
  - P95 API Latency: histogram_quantile(0.95, rate(coordinator_request_duration_seconds_bucket[5m]))
  - Task Throughput: rate(coordinator_tasks_created_total[5m])
  - Pending Approvals: coordinator_approval_gates_pending

Row 2: Agent Activity
  - Active Agents by Type: sum by (agent_type) (agent_runtime_active_count)
  - Completion Rate: rate(agent_executions_total{status="completed"}[5m])
  - Failed Rate: rate(agent_executions_total{status="failed"}[5m])
  - Avg Execution Time: rate(agent_execution_duration_seconds_sum[5m]) / rate(agent_execution_duration_seconds_count[5m])

Row 3: Cost and Tokens
  - Hourly Token Consumption: increase(agent_tokens_total[1h])
  - Hourly Cost USD: increase(agent_cost_usd_total[1h])
  - Cost per Agent Type: sum by (agent_type) (increase(agent_cost_usd_total[24h]))
  - Model Distribution: sum by (model) (increase(agent_tokens_total[1h]))

Row 4: Infrastructure
  - PostgreSQL Connections: pg_stat_database_numbackends
  - Redis Memory Usage: redis_memory_used_bytes
  - Qdrant Collection Sizes: ke_qdrant_points_total by collection
  - NATS Consumer Lag: orchestrator_nats_consumer_lag by stream
```

### Dashboard 2: Agent Deep Dive

```
Time range selector: Agent type filter dropdown
  - Execution Duration Heatmap (by model)
  - Token Usage Trend (input vs output)
  - Cost Trend vs Token Count scatter
  - Success/Fail ratio over time
  - Tool call breakdown by tool_name
  - Retry rate by failure reason
  - P50/P95/P99 duration table by task_type
```

### Dashboard 3: Knowledge Engine

```
  - Search latency (FTS vs vector vs RRF combined)
  - Embedding latency percentiles by model
  - Qdrant points per collection (capacity planning)
  - Memory type distribution donut chart
  - KG ingestion jobs: duration, nodes/edges per run
  - Cache hit rate for repeated embeddings
```

## 16.4 Alerting Rules

```yaml
# Prometheus alert rules

groups:
  - name: aistudio-platform
    rules:
      - alert: CoordinatorHighErrorRate
        expr: |
          (rate(coordinator_requests_total{status=~"5.."}[5m])
          / rate(coordinator_requests_total[5m])) > 0.02
        for: 2m
        labels: {severity: critical, team: platform}
        annotations:
          summary: "Coordinator error rate > 2% for 2 minutes"
          runbook: "https://wiki.internal/aistudio/runbooks/coordinator-errors"

      - alert: CoordinatorHighLatency
        expr: |
          histogram_quantile(0.95,
            rate(coordinator_request_duration_seconds_bucket[5m])) > 2.0
        for: 5m
        labels: {severity: warning}
        annotations:
          summary: "Coordinator P95 latency > 2s"

      - alert: ApprovalGateBacklog
        expr: coordinator_approval_gates_pending > 20
        for: 30m
        labels: {severity: warning}
        annotations:
          summary: "More than 20 approval gates pending for 30 minutes — humans not reviewing"

      - alert: AgentHighFailureRate
        expr: |
          sum by (agent_type) (
            rate(agent_executions_total{status="failed"}[10m])
          ) / sum by (agent_type) (
            rate(agent_executions_total[10m])
          ) > 0.2
        for: 5m
        labels: {severity: critical}
        annotations:
          summary: "Agent {{ $labels.agent_type }} failure rate > 20%"

      - alert: NATSConsumerLagHigh
        expr: orchestrator_nats_consumer_lag > 200
        for: 5m
        labels: {severity: warning}
        annotations:
          summary: "NATS consumer lag > 200 messages — scale up agent-runtime"

      - alert: PostgresConnectionsSaturated
        expr: pg_stat_database_numbackends > 150
        for: 2m
        labels: {severity: critical}
        annotations:
          summary: "PostgreSQL connections > 150 — connection pool exhaustion imminent"

      - alert: QdrantCapacityWarning
        expr: ke_qdrant_points_total{collection="project_memories"} > 5000000
        labels: {severity: warning}
        annotations:
          summary: "Qdrant project_memories collection > 5M points — plan capacity"

      - alert: CostBudgetExceeded
        expr: increase(agent_cost_usd_total[24h]) > 100
        labels: {severity: warning}
        annotations:
          summary: "Daily AI cost > $100 — review task volume and model selection"

      - alert: PostgresDown
        expr: up{job="postgres-exporter"} == 0
        for: 1m
        labels: {severity: critical}
        annotations:
          summary: "PostgreSQL is down — all services will fail within seconds"

      - alert: NATSDown
        expr: up{job="nats"} == 0
        for: 1m
        labels: {severity: critical}
        annotations:
          summary: "NATS is down — task dispatch and agent results will stall"
```

## 16.5 Structured Logging Contract

Every log line is JSON on stdout. All services conform to this schema:

```python
# Canonical log fields

{
  "timestamp": "2026-06-27T12:34:56.789Z",   # ISO 8601 UTC always
  "level": "INFO",                             # DEBUG|INFO|WARNING|ERROR|CRITICAL
  "service": "coordinator",                    # Service name constant
  "instance_id": "coordinator-abc123",         # Unique per pod/process
  "trace_id": "abc...",                        # OpenTelemetry trace_id
  "span_id": "def...",                         # OpenTelemetry span_id
  "request_id": "req-xyz",                     # HTTP X-Request-Id if applicable
  "task_id": "uuid",                           # If log relates to a task
  "project_id": "uuid",                        # If log relates to a project
  "user_id": "uuid",                           # If log relates to a user
  "agent_type": "backend",                     # If from agent-runtime
  "message": "Task dispatched to backend agent",
  "data": {}                                   # Optional structured context
}
```

Log levels by category:
```
DEBUG  — Internal state, variable values, decision branches (disabled in prod)
INFO   — Task state transitions, agent start/complete, API requests (normal traffic)
WARNING — Unexpected but recoverable: retry, fallback model used, rate limit hit
ERROR  — Failed operations requiring investigation: task failed, tool error, DB timeout
CRITICAL — Service-affecting failures: DB down, NATS down, OOM
```

## 16.6 Distributed Tracing

```python
# Each service initializes OpenTelemetry on startup
from opentelemetry import trace
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.asyncpg import AsyncPGInstrumentor

def init_tracing(service_name: str, otlp_endpoint: str):
    provider = TracerProvider(
        resource=Resource.create({"service.name": service_name})
    )
    provider.add_span_processor(
        BatchSpanProcessor(OTLPSpanExporter(endpoint=otlp_endpoint))
    )
    trace.set_tracer_provider(provider)
    FastAPIInstrumentor.instrument()
    AsyncPGInstrumentor.instrument()
```

Critical spans to instrument manually:
```
coordinator: task_create, approval_decide, ws_broadcast
orchestrator: dependency_resolve, agent_dispatch, retry_decision
agent-runtime: context_assemble, ai_model_call, tool_call_execute
knowledge-engine: vector_embed, qdrant_search, fts_search, rrf_merge
brain: outcome_record, pattern_detect, recommendation_generate
```

---

# PART 17: TESTING STRATEGY

## 17.1 Testing Pyramid

```
Layer 4: E2E Tests (Playwright, Detox) — 10% of tests
  Scope: Full user journeys through UI (web and mobile)
  Run: Nightly CI + before release

Layer 3: Integration Tests (pytest + real services) — 20% of tests
  Scope: Cross-service flows (REST + DB + NATS)
  Infrastructure: Docker Compose test environment
  Run: Per PR in CI

Layer 2: Component/Contract Tests — 20% of tests
  Scope: Service boundaries (REST API shapes, NATS message schemas)
  Tool: Pact for consumer-driven contracts
  Run: Per PR

Layer 1: Unit Tests (pytest, vitest, jest) — 50% of tests
  Scope: Single function/class, mocked dependencies
  Run: Per commit in CI (< 2 minutes)
```

## 17.2 Unit Test Requirements by Service

```
coordinator:
  - All Pydantic models: valid + invalid inputs
  - Auth: JWT decode/expiry/revocation, API key prefix+hash match
  - RBAC: each role against each endpoint action
  - Business logic: approval gate state machine transitions
  - NATS publisher: correct payload shape per event type
  Coverage: 80% minimum

orchestrator:
  - Dependency resolver: DAG with cycles, isolated nodes, fan-in/fan-out
  - Priority queue ordering: equal priority + FIFO tiebreak
  - Retry backoff: exponential intervals, max attempt check
  - Stall detector: timeout calculation per task type
  - Workflow health checks: each check independently
  Coverage: 75% minimum

agent-runtime:
  - BaseAgent: validate_task (type in capabilities), estimate_cost
  - ContextAssembler: token budget calculation, memory ranking
  - ModelRouter: affinity selection, fallback chain, circuit breaker skip
  - Each agent: happy path + tool failure + token limit handling
  - Tool summarizer: truncation at exactly 500 chars
  Coverage: 70% minimum

knowledge-engine:
  - Memory writer: embed -> PG insert -> Qdrant upsert order
  - Hybrid search: RRF score calculation, rank merging
  - Code ingestion: file path -> stable ID, batch size
  Coverage: 75% minimum

brain:
  - Outcome recorder: field mapping, error category classification
  - Pattern detector: failure rate threshold, minimum observation count
  - Recommendation: confidence filter, observation count filter
  Coverage: 75% minimum
```

## 17.3 Integration Test Suite

```python
# coordinator/tests/integration/test_task_lifecycle.py

@pytest.mark.asyncio
async def test_task_creates_nats_event(
    test_app: AsyncClient,
    test_nats: NATSTestServer,
    test_db: asyncpg.Pool
):
    # Arrange: subscribe to NATS before creating task
    received = []
    async def capture(msg):
        received.append(json.loads(msg.data))
        await msg.ack()
    await test_nats.subscribe("tasks.created", cb=capture)

    # Act: create task via API
    resp = await test_app.post("/api/v2/projects/{pid}/tasks".format(pid=TEST_PROJECT_ID),
        json={"type": "feature", "title": "Test feature", "description": "Test"},
        headers={"Authorization": f"Bearer {TEST_JWT}"}
    )
    await asyncio.sleep(0.1)  # Allow NATS delivery

    # Assert
    assert resp.status_code == 201
    task_id = resp.json()["id"]
    assert len(received) == 1
    assert received[0]["task_id"] == task_id
    assert received[0]["type"] == "feature"

    # Verify DB
    row = await test_db.fetchrow(
        "SELECT status FROM coordinator.tasks WHERE id = $1", UUID(task_id)
    )
    assert row["status"] == "pending"
```

```python
# coordinator/tests/integration/test_approval_flow.py

@pytest.mark.asyncio
async def test_approve_unblocks_task(test_app, test_db, test_nats):
    # Setup: create task with awaiting_approval status + gate
    task_id = await _create_test_task(test_db)
    gate_id = await _create_test_gate(test_db, task_id)

    # Subscribe to NATS approval event
    granted_events = []
    await test_nats.subscribe("approval.granted",
        cb=lambda m: granted_events.append(json.loads(m.data)) or asyncio.ensure_future(m.ack()))

    # Approve via API
    resp = await test_app.post(
        f"/api/v2/approvals/{gate_id}/approve",
        json={"notes": "LGTM"},
        headers={"Authorization": f"Bearer {DEVELOPER_JWT}"}
    )
    await asyncio.sleep(0.1)

    assert resp.status_code == 200
    assert resp.json()["status"] == "approved"

    task_row = await test_db.fetchrow(
        "SELECT status FROM coordinator.tasks WHERE id = $1", task_id
    )
    assert task_row["status"] == "ready"
    assert len(granted_events) == 1
    assert granted_events[0]["gate_id"] == str(gate_id)
```

## 17.4 Contract Tests (Pact)

```python
# coordinator/tests/contract/test_coordinator_consumer.py
# Consumer: coordinator produces tasks.created events
# Provider: orchestrator consumes them

from pact import MessagePact

pact = MessagePact(consumer="coordinator", provider="orchestrator")

def test_task_created_event_contract():
    expected_event = {
        "event_id": pact.like("uuid-string"),
        "version": "1",
        "task_id": pact.like("uuid-string"),
        "project_id": pact.like("uuid-string"),
        "type": pact.term(
            matcher="feature|bug|refactor|test|deploy",
            generate="feature"
        ),
        "priority": pact.like(50),
        "depends_on": pact.each_like("uuid-string", min=0)
    }

    with pact.given("a task is created") \
             .expects_to_receive("tasks.created") \
             .with_content(expected_event) \
             .with_metadata({"contentType": "application/json"}):
        task_event = TaskCreatedEvent(
            task_id="test-task-id",
            project_id="test-project-id",
            type="feature",
            priority=75,
            depends_on=[]
        )
        assert task_event.dict() == expected_event
```

## 17.5 Performance Benchmarks

```
coordinator:
  Benchmark: 500 task POSTs/second sustained 30 seconds
  Target:    P95 < 200ms, 0 errors
  Tool:      k6 (load-test/coordinator_throughput.js)

coordinator (WebSocket):
  Benchmark: 1000 concurrent WS connections, 10 events/second broadcast
  Target:    P95 broadcast latency < 50ms, 0 dropped events
  Tool:      k6 WebSocket extension

agent-runtime:
  Benchmark: 50 concurrent agent executions (mocked AI responses)
  Target:    NATS dispatch < 10ms, context assembly < 500ms, 0 deadlocks
  Tool:      pytest-benchmark + asyncio concurrency stress

knowledge-engine:
  Benchmark: 100 hybrid search queries/second against 1M vectors
  Target:    P95 < 200ms, P99 < 500ms
  Tool:      locust (load-test/ke_search.py)

PostgreSQL:
  Target:    < 10ms P95 for all indexed queries
  Monitoring: pg_stat_statements, auto_explain for queries > 100ms
```

## 17.6 Agent Behavior Tests

```python
# agent-runtime/tests/behavior/test_backend_agent.py
# Tests agent decision-making with controlled AI responses

@pytest.mark.asyncio
async def test_backend_agent_requests_approval_for_large_diff(
    mock_context: MockAgentContext,
    mock_tools: MockToolRegistry
):
    # Simulate a diff of 600 changed lines
    mock_tools.set_response("diff_measurement", {
        "lines_changed": 600, "files_changed": 8, "summary": "Large refactor"
    })
    mock_tools.set_response("test_runner", {"passed": True, "coverage": 82})

    agent = BackendAgent()
    result = await agent.execute(
        task={"type": "feature", "title": "Refactor auth module"},
        context=mock_context
    )

    assert result.approval_required is True
    assert result.approval_gate_spec["gate_type"] == "commit"
    assert result.approval_gate_spec["risk_score"] >= 60
    assert result.status == "requires_approval"

@pytest.mark.asyncio
async def test_backend_agent_creates_followup_for_test_failure(
    mock_context: MockAgentContext,
    mock_tools: MockToolRegistry
):
    mock_tools.set_response("test_runner", {
        "passed": False,
        "failure_details": "test_payment_validation: assertion error line 45"
    })

    agent = BackendAgent()
    result = await agent.execute(
        task={"type": "feature", "title": "Add payment validation"},
        context=mock_context
    )

    assert result.status == "blocked"
    assert len(result.next_steps) == 1
    assert result.next_steps[0]["type"] == "bug"
    assert "Fix failing tests" in result.next_steps[0]["title"]
```

## 17.7 CI/CD Pipeline

```yaml
# .github/workflows/ci.yaml (per service repository)

name: CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15-alpine
        env: {POSTGRES_USER: test, POSTGRES_PASSWORD: test, POSTGRES_DB: test}
        options: >-
          --health-cmd pg_isready
          --health-interval 5s
          --health-retries 10
      nats:
        image: nats:2.10-alpine
        options: --entrypoint "nats-server -js"
      redis:
        image: redis:7-alpine
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: {python-version: "3.12"}
      - run: pip install -r requirements.txt -r requirements-test.txt
      - run: alembic upgrade head
        env: {PG_DSN: "postgresql://test:test@localhost/test"}
      - run: pytest tests/ --cov --cov-fail-under=75 -v
      - run: mypy . --strict
      - run: ruff check .
      - name: Architecture dependency check
        run: python scripts/check_dependencies.py  # Fails if layer rules violated

  build:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - name: Build image
        run: |
          IMAGE=registry.aistudio.internal/${SERVICE_NAME}:${GITHUB_SHA}
          docker build -t $IMAGE .
          docker push $IMAGE
      - name: Update manifests
        run: |
          # Update Helm values with new image tag
          sed -i "s|tag:.*|tag: ${GITHUB_SHA}|" ../deployment/k8s/helm/${SERVICE_NAME}/values.yaml
          git commit -am "ci: bump ${SERVICE_NAME} to ${GITHUB_SHA}"
          git push
```

---

# PART 18: MIGRATION GUIDE — 1.5.5 TO 2.0

## 18.1 Scope

AI Studio 1.5.5 uses a single-process Python runtime with SQLite storage, no NATS, and a polling-based orchestration loop. AI Studio 2.0 is a distributed polyrepo system with PostgreSQL, NATS JetStream, and a pool of AI agents. There is no automated migration path — this is a complete platform replacement with a controlled cutover procedure.

## 18.2 Differences Summary

| Dimension | 1.5.5 | 2.0 |
|-----------|-------|-----|
| Runtime | Single Python process | 9 independent services |
| Storage | SQLite + YAML files | PostgreSQL + Qdrant + Kuzu |
| Messaging | Direct function calls | NATS JetStream async events |
| Agent types | 10 worker types (ARCHITECT…RELEASE) | 18 agent types |
| Task states | 11 states (BACKLOG…RELEASED) | 9 states (pending…cancelled) |
| Orchestration | 8-step tick every 30s | Event-driven, immediate dispatch |
| Approval gates | 3 gates (Gate 2/3/4) | Per-task, configurable |
| Memory | No persistent memory | 4-tier: context/Redis/PG+Qdrant/KG |
| Deployment | Single Docker container | Docker Compose / Kubernetes |
| Horizontal scale | Not supported | NATS queue groups, HPA |

## 18.3 Data Migration

### SQLite Tasks → PostgreSQL Tasks

```python
# migration/migrate_tasks.py

import sqlite3
import asyncpg
import json
from uuid import uuid4

STATUS_MAP_1_5_5_TO_2 = {
    "BACKLOG":    "pending",
    "READY":      "ready",
    "PREPARING":  "running",
    "IN_PROGRESS": "running",
    "REVIEW":     "awaiting_approval",
    "APPROVED":   "ready",
    "MERGED":     "completed",
    "BLOCKED":    "blocked",
    "SUSPENDED":  "blocked",
    "STALLED":    "failed",
    "RELEASED":   "completed"
}

TYPE_MAP_1_5_5_TO_2 = {
    "ARCHITECT": "architecture",
    "AUTH":      "security",
    "PLAYER":    "feature",
    "ECONOMY":   "feature",
    "COLLECTION": "feature",
    "BATTLE":    "feature",
    "LIVEOPS":   "feature",
    "SECURITY":  "security",
    "QA":        "test",
    "RELEASE":   "release"
}

async def migrate_tasks(sqlite_path: str, pg_dsn: str, project_id: str):
    conn = sqlite3.connect(sqlite_path)
    conn.row_factory = sqlite3.Row
    pg = await asyncpg.connect(pg_dsn)

    rows = conn.execute("SELECT * FROM tasks ORDER BY created_at").fetchall()
    print(f"Migrating {len(rows)} tasks...")

    for row in rows:
        new_status = STATUS_MAP_1_5_5_TO_2.get(row["status"], "pending")
        new_type = TYPE_MAP_1_5_5_TO_2.get(row.get("worker_type", "ARCHITECT"), "feature")

        await pg.execute("""
            INSERT INTO coordinator.tasks
                (id, project_id, type, title, description, status, priority,
                 metadata, created_at)
            VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9::timestamptz)
            ON CONFLICT (id) DO NOTHING
        """,
            uuid4(), project_id, new_type,
            row.get("title", row.get("name", "Untitled")),
            row.get("description", ""),
            new_status, 50,
            json.dumps({"migrated_from": "1.5.5", "original_status": row["status"]}),
            row["created_at"]
        )

    print("Task migration complete")
    await pg.close()
    conn.close()
```

## 18.4 Cutover Procedure

```
Phase 0: Preparation (1 week before cutover)
  1. Deploy AI Studio 2.0 alongside 1.5.5 (different ports)
  2. Migrate historical SQLite tasks to PostgreSQL (read-only migration)
  3. Run 2.0 in shadow mode: receive same inputs, compare outputs (do not act)
  4. Resolve all discrepancies found in shadow mode

Phase 1: Soft Cutover (day 0)
  1. Set 1.5.5 to MAINTENANCE MODE (no new tasks accepted)
  2. Wait for all in-flight 1.5.5 tasks to reach MERGED or STALLED
  3. Final SQLite -> PostgreSQL sync (incremental, ~5 minute window)
  4. DNS cutover: ai-studio.internal -> 2.0 coordinator
  5. Verify: POST /api/v2/projects/.../tasks succeeds
  6. Monitor error rate for 30 minutes
  7. If error rate > 2%: DNS rollback to 1.5.5

Phase 2: Validation (day 1-7)
  1. Monitor all Grafana dashboards continuously
  2. Compare task completion rate vs 1.5.5 historical baseline
  3. Verify approval gate notifications reach developers (Desktop + Web)
  4. Check memory system: run knowledge-engine /ingest on migrated projects

Phase 3: Decommission (day 30)
  1. Archive 1.5.5 container image to registry with tag "legacy-1.5.5"
  2. Stop 1.5.5 process
  3. Archive SQLite database to S3 (retain for 1 year)
  4. Remove 1.5.5 deployment configuration
```

## 18.5 Feature Parity Matrix

| 1.5.5 Feature | 2.0 Equivalent | Notes |
|---------------|----------------|-------|
| 8-step orchestration tick | Event-driven orchestration | No polling; immediate on event |
| Gate 2 (ARCHITECT review) | ApprovalGate (gate_type=commit) | Created by agent, risk_score-based |
| Gate 3 (SECURITY, conditional) | SecurityAgent output + ApprovalGate | SecurityAgent decides if gate needed |
| Gate 4 (QA) | QAAgent output + CI check | CI result determines pass/fail |
| root_cause analysis | LearningAgent + brain patterns | Continuous, not per-incident |
| TASK_BLOCKED: token | status=blocked in DB | Structured; reason in error_data |
| TASK_COMPLETE token | status=completed in DB | Structured; result_data populated |
| workflow_health check | /healthz/ready on all services | Per-service, not per-orchestration-tick |
| sla monitoring | Prometheus deadline_at alert | Alert fires if running > deadline |
| Worker type ARCHITECT | ArchitectAgent | Same purpose, richer context |
| Worker type AUTH | SecurityAgent | AUTH renamed + expanded |

---

# PART 19: PERFORMANCE TARGETS

## 19.1 SLOs

```
SLO 1: API Availability
  Target:  99.5% uptime (monthly)
  Error budget: 3.6 hours/month
  Measurement: rate(coordinator_requests_total{status!~"5.."}[1h]) / rate(coordinator_requests_total[1h])

SLO 2: Task Submission Latency
  Target:  P95 < 200ms, P99 < 500ms
  Measurement: histogram_quantile(0.95, rate(coordinator_request_duration_seconds_bucket{path="/api/v2/projects/*/tasks",method="POST"}[5m]))

SLO 3: Task Completion Time
  Targets (by type, from submission to completed status):
    feature    P50 < 12 min,  P95 < 30 min
    bug        P50 < 8 min,   P95 < 20 min
    refactor   P50 < 15 min,  P95 < 45 min
    test       P50 < 5 min,   P95 < 15 min
    deploy     P50 < 10 min,  P95 < 20 min
    document   P50 < 3 min,   P95 < 8 min
  Measurement: coordinator.tasks.completed_at - created_at per task_type

SLO 4: Approval Gate Notification Latency
  Target:  < 5 seconds from gate creation to WebSocket delivery
  Measurement: trace span: approval_gate_created -> ws_broadcast_complete

SLO 5: Memory Search Latency
  Target:  P95 < 200ms for hybrid search
  Measurement: ke_search_duration_seconds histogram

SLO 6: Cost per Task
  Targets:
    feature task:    < $0.50 average
    bug task:        < $0.30 average
    test task:       < $0.20 average
    document task:   < $0.10 average
  Measurement: agent_cost_usd_total / agent_executions_total by task_type
```

## 19.2 Capacity Planning

```
Laptop Mode (single developer):
  Expected load: 1-5 concurrent tasks
  Required: 8 GB RAM, 4 CPU cores, 50 GB SSD
  AI providers: Ollama (llama3:70b) OR cloud API keys
  Expected cost: $0-$5/day with cloud; $0 with Ollama

Team Mode (10 developers):
  Expected load: 20-50 concurrent tasks
  Required: 32 GB RAM server, 16 CPU cores, 500 GB SSD
  Docker Compose with team profiles
  Expected cost: $20-$80/day with cloud AI

Enterprise Mode (100+ developers):
  Expected load: 200-500 concurrent tasks
  Architecture: 3-node Kubernetes, 3 agent-runtime replicas (autoscaling to 20)
  Database: RDS PostgreSQL r6g.2xlarge (Multi-AZ)
  Redis: ElastiCache r7g.large (2 nodes)
  NATS: 3-node JetStream cluster
  Expected cost: $200-$800/day AI + $500-$1500/day infrastructure
```

## 19.3 Database Sizing

```
PostgreSQL storage estimates (per year, 10-developer team):

coordinator.tasks:          ~50K rows/year    = 500 MB
coordinator.agent_executions: ~200K rows/year = 2 GB
coordinator.tool_calls:     ~2M rows/year     = 5 GB (with 30d retention)
coordinator.events:         ~5M rows/year     = 10 GB (with 90d partition drop)
coordinator.memories:       ~100K rows/year   = 1 GB

Total coordinator schema: ~20 GB/year (with retention applied: ~8 GB steady state)

Qdrant:
  project_memories collection: 100K vectors * 1536 dims * 4 bytes = 600 MB
  code_chunks collection:       1M vectors * 1536 dims * 4 bytes  = 6 GB

Kuzu (knowledge graph):
  Medium codebase (100K LOC): ~500K nodes, ~2M edges = 2 GB
  Large codebase (500K LOC):  ~2.5M nodes, ~10M edges = 10 GB
```

## 19.4 Optimization Runbook

```
SYMPTOM: High coordinator latency (P95 > 500ms)
DIAGNOSIS:
  1. Check pg_stat_statements for slow queries (> 100ms)
  2. Check redis_memory_used_bytes (near limit?)
  3. Check coordinator_ws_connections_active (> 500?)
REMEDIES:
  - Add indexes on frequently filtered columns
  - Increase pg_pool_max from 20 -> 40
  - Scale coordinator replicas from 2 -> 4
  - Reduce WebSocket keep-alive overhead with connection pooling at nginx

SYMPTOM: Agent tasks taking longer than SLO
DIAGNOSIS:
  1. Check agent_execution_duration_seconds histogram per agent_type
  2. Check AI provider latency (separate span in traces)
  3. Check tool_calls duration_ms for slowest tools
  4. Check ke_search_duration_seconds (context assembly bottleneck?)
REMEDIES:
  - Switch to faster model for specific agent_type (e.g., haiku for documentation)
  - Reduce token budget for lightweight tasks
  - Cache common code_chunks embeddings in Redis
  - Increase tool-runtime sandbox resource limits

SYMPTOM: NATS consumer lag growing (NATSConsumerLagHigh alert)
DIAGNOSIS:
  1. Check ACTIVE_AGENTS gauge (all replicas at capacity?)
  2. Check agent_executions task_type distribution (all in one type?)
REMEDIES:
  - Scale agent-runtime replicas (KEDA trigger)
  - Adjust NATS consumer MaxAckPending to allow more parallel task processing
  - Check if one stuck task is holding a NATS message without ACKing
```

## 19.5 Token Optimization Rules

```
Rule 1: Context budget limits
  - documentation agent:  max 4K context tokens (simple tasks)
  - qa agent:             max 6K context tokens
  - backend/architect:    max 16K context tokens
  Enforcement: token_budgeter.py clips project_memories list to fit budget

Rule 2: Memory relevance threshold
  - minimum_similarity_score: 0.75 for Qdrant results
  - minimum_importance: 30 for memories included in context
  - max memories in context: 20 (hard limit)

Rule 3: Code chunk limits
  - Max 10 code chunks per agent context
  - Chunks ranked by: (relevance_score * 0.7) + (recency_score * 0.3)

Rule 4: Model selection for cost
  - Tasks with description length < 200 chars: force haiku (20x cheaper)
  - Tasks estimated_tokens < 1000: prefer haiku or sonnet
  - Architecture/Security tasks: always opus (quality-critical)

Rule 5: Streaming responses
  - All agent AI calls use streaming (stream=True) to reduce TTFB
  - Token counting done from stream deltas, not final usage object
  - Streaming allows early cancellation if token budget exceeded mid-response
```

---

# PART 20: ENGINEERING ROADMAP

## 20.1 Milestone Schedule

```
Milestone 1: Core Platform (Foundation)          Q3 2026
  - All 9 services deployed and operational
  - 18 agent types implemented and tested
  - PostgreSQL + Qdrant + Kuzu data layers live
  - NATS JetStream event bus operational
  - Desktop application (PySide6) feature-complete
  - Human approval gates working end-to-end
  - Laptop mode (Docker Compose) one-command install
  - Coverage: 70%+ across all services

Milestone 2: Team Collaboration               Q4 2026
  - Web application (React 19) feature-complete
  - OIDC/OAuth2 SSO integration (Okta, Azure AD, Google)
  - LDAP integration for enterprise directories
  - Role-based access control fully enforced
  - PostgreSQL-based experience graph (brain)
  - Pattern-based agent recommendations operational
  - Prometheus + Grafana dashboards live
  - Team mode Docker Compose profile

Milestone 3: Mobile + Scale                   Q1 2027
  - Mobile app (React Native, iOS + Android) — read + approve only
  - Push notifications for approval gates
  - Kubernetes Helm charts production-ready
  - Terraform for AWS + GCP (Azure in 3.1)
  - KEDA-based autoscaling for agent-runtime
  - Multi-region deployment support (active-passive)
  - 99.5% SLO verified in production load test

Milestone 4: Marketplace + Ecosystem          Q2 2027
  - Plugin marketplace (discovery + install + publish)
  - GPG-signed plugin packages
  - Plugin sandbox security review process
  - Community workflow template library (50+ templates)
  - OpenTelemetry tracing to Jaeger/Tempo
  - Neo4j enterprise knowledge graph option
  - API v3 (breaking changes managed via versioning)
```

## 20.2 Technical Debt Register

```
DEBT-001: Kuzu vs Neo4j abstraction
  Issue: knowledge-engine/graph/ tightly coupled to Kuzu API
  Impact: Switching to Neo4j (enterprise) requires full rewrite of graph/
  Resolution: Define GraphClient ABC in sdk; implement KuzuAdapter and Neo4jAdapter
  Priority: P2 — resolve before Milestone 3 multi-region

DEBT-002: Embedding model dependency
  Issue: knowledge-engine embeds via Ollama (laptop) or hardcoded Anthropic
  Impact: Embedding model change requires re-embedding all stored vectors
  Resolution: Store embedding_model_id with each Qdrant point;
              migration job to re-embed when model changes
  Priority: P1 — resolve before Milestone 2 (embedding model lock-in is breaking)

DEBT-003: Approval gate auto-expiry not distributed
  Issue: approval_expiry_loop runs only in coordinator; coordinator restart loses timers
  Impact: Gates may expire late if coordinator restarts
  Resolution: Use Redis EXPIRE + keyspace notifications instead of in-process loop
  Priority: P3 — acceptable until team mode scale

DEBT-004: agent-runtime writes to coordinator schema directly
  Issue: agent-runtime writes agent_executions and tool_calls rows directly
  Impact: Tight coupling; DB password for agent-runtime has write access to coordinator tables
  Resolution: coordinator provides internal endpoint for execution result submission;
              agent-runtime posts results to coordinator REST, not DB
  Priority: P2 — security risk; resolve before enterprise deployment

DEBT-005: No rate limiting on task submission
  Issue: POST /api/v2/projects/{id}/tasks has no per-user rate limit
  Impact: One developer can flood task queue
  Resolution: Redis-backed sliding window rate limiter (100 tasks/minute/user)
  Priority: P2 — resolve before team mode
```

## 20.3 Version Compatibility Policy

```
API Versioning:
  /api/v2/ is stable — breaking changes require /api/v3/
  Non-breaking additions (new fields, new endpoints) are allowed in /api/v2/
  /api/v2/ minimum support: 12 months from /api/v3/ GA

NATS Event Schema Versioning:
  All events have "version" field (currently "1")
  Schema changes: bump version string, emit both version="1" and version="2" for 3 months
  Consumers must handle both versions during transition window

Database Migration Compatibility:
  Migrations must be backward-compatible for 1 release cycle
  New columns: nullable at first, NOT NULL constraint added in next release
  Column removal: mark DEPRECATED in a comment, remove after 2 releases

Plugin API Versioning:
  Plugin SDK (aistudio-sdk) follows semver
  BaseAgent and BaseTool ABCs: no breaking changes within major version
  min_platform_version in plugin.yaml enforced at install time

Python Runtime:
  Minimum Python 3.12 (EOL: October 2028)
  Python 3.13 support: add in next minor release after 3.13 GA
  Dependencies pinned in requirements.txt; updated monthly via Dependabot
```

## 20.4 Open Architecture Questions

```
DECISION-001: Graph Database for Production Scale
  Status: PENDING — needs resolution before Milestone 3
  Options:
    A) Kuzu embedded everywhere (current): simple, no network hop, limited HA
    B) Neo4j Community (self-hosted): HA support, Cypher, licensing cost
    C) Memgraph (open source): Cypher-compatible, better write performance
  Decision criteria: query latency at 10M nodes, HA requirement, licensing cost
  Owner: Platform team
  Deadline: 2026-10-01

DECISION-002: Real-Time Event Delivery Guarantee
  Status: PENDING
  Current: NATS at-least-once delivery; coordinator deduplicates on task_id
  Question: Does the brain need exactly-once recording of outcomes?
  Options:
    A) Idempotent inserts (ON CONFLICT DO NOTHING) — current approach
    B) NATS exactly-once with deduplication window
  Owner: Brain service team
  Deadline: 2026-09-01

DECISION-003: LLM Fine-Tuning
  Status: FUTURE — post Milestone 4
  Concept: Fine-tune a base model on successful agent completions from brain.execution_outcomes
  Blocker: Requires 10K+ high-quality outcome examples
  Evaluation metric: task completion rate improvement vs. base model
  Owner: AI/ML team
  Timeline: 2027 Q3

DECISION-004: Multi-Tenant Isolation
  Status: FUTURE — enterprise tier requirement
  Current: All customers share one PostgreSQL cluster (schema-based isolation)
  Future: Dedicated PostgreSQL cluster per enterprise customer
  Trigger: First enterprise customer with data residency requirement
  Owner: Infrastructure team
```

## 20.5 Agent Implementation Priority Queue

```
Milestone 1 (required at launch):
  1. PlannerAgent      — without this, no task decomposition
  2. BackendAgent      — core feature implementation
  3. QAAgent           — without this, no test coverage
  4. SecurityAgent     — human gates depend on security review
  5. ReviewerAgent     — PR-level code review
  6. DatabaseAgent     — schema design + migrations
  7. DevOpsAgent       — CI/CD configuration
  8. DocumentationAgent — README + docstrings
  9. ArchitectAgent    — ADRs + design decisions
  10. ReleaseAgent     — versioning + SBOM

Milestone 2 (team collaboration):
  11. LearningAgent    — brain feedback loop
  12. MonitoringAgent  — alert rule generation
  13. FrontendAgent    — UI component generation

Milestone 3+:
  14. RefactoringAgent — legacy code improvement
  15. ResearchAgent    — technology evaluation
  16. UXAgent          — user research synthesis (requires external data access)
  17. ProductManagerAgent — requirements synthesis
  18. SupportAgent     — issue triage, runbook generation
```

---

# APPENDIX A: ENVIRONMENT VARIABLE REFERENCE

```
COORDINATOR
  PG_DSN                   postgresql+asyncpg://user:pass@host/db
  REDIS_URL                redis://host:6379/0
  NATS_URL                 nats://host:4222
  JWT_PUBLIC_KEY           RSA PEM string (or JWT_PUBLIC_KEY_FILE for file path)
  KNOWLEDGE_ENGINE_URL     http://knowledge-engine:8091
  AUTH_SERVICE_URL         http://auth:8093
  CORS_ORIGINS             http://localhost:3000,https://app.company.com
  DEV_MODE                 true|false (enables /docs, debug logging)
  LOG_LEVEL                DEBUG|INFO|WARNING|ERROR
  PORT                     8090
  PG_POOL_MIN              5
  PG_POOL_MAX              20

ORCHESTRATOR
  PG_DSN                   Same PostgreSQL as coordinator (coordinator schema)
  NATS_URL                 nats://host:4222
  REDIS_URL                redis://host:6379/2  (different DB index)
  TICK_INTERVAL_SECONDS    30
  STALL_TIMEOUT_SECONDS    300
  MAX_RETRY_ATTEMPTS       3
  LEADER_ELECTION_KEY      orchestrator:leader
  LEADER_TTL_SECONDS       60

AGENT-RUNTIME
  PG_DSN                   postgresql+asyncpg://user:pass@host/db
  NATS_URL                 nats://host:4222
  KNOWLEDGE_ENGINE_URL     http://knowledge-engine:8091
  BRAIN_URL                http://brain:8094
  TOOL_RUNTIME_URL         http://tool-runtime:8092
  ENABLED_PROVIDERS        anthropic,openai,ollama  (comma-separated)
  ANTHROPIC_API_KEY        sk-ant-...
  OPENAI_API_KEY           sk-...
  GOOGLE_API_KEY           ...
  OLLAMA_URL               http://ollama:11434
  MAX_CONCURRENT_AGENTS    8
  DEFAULT_TOKEN_BUDGET     16000
  CHECKPOINT_INTERVAL      60  (seconds between checkpoints)

KNOWLEDGE-ENGINE
  PG_DSN                   postgresql+asyncpg://user:pass@host/db
  QDRANT_URL               http://qdrant:6333
  KUZU_DB_PATH             /data/kuzu  (or NEO4J_URL for Neo4j mode)
  NEO4J_URL                bolt://neo4j:7687  (empty = use Kuzu)
  NEO4J_USER               neo4j
  NEO4J_PASSWORD           ...
  EMBEDDER                 anthropic|openai|ollama
  EMBEDDING_MODEL          text-embedding-3-small  (OpenAI) | nomic-embed-text (Ollama)
  ANTHROPIC_API_KEY        sk-ant-...  (required if EMBEDDER=anthropic)
  OLLAMA_URL               http://ollama:11434

BRAIN
  PG_DSN                   postgresql+asyncpg://user:pass@host/db
  QDRANT_URL               http://qdrant:6333
  NEO4J_URL                bolt://neo4j:7687  (empty = use postgres for experience graph)

AUTH
  PG_DSN                   postgresql+asyncpg://user:pass@host/db
  REDIS_URL                redis://host:6379/1
  JWT_PRIVATE_KEY          RSA PEM string (or JWT_PRIVATE_KEY_FILE)
  JWT_ALGORITHM            RS256
  JWT_EXPIRY_SECONDS       3600  (1 hour)
  JWT_REFRESH_EXPIRY       604800  (7 days)
  OIDC_ENABLED             true|false
  OIDC_AUTHORITY           https://auth.company.com
  OIDC_CLIENT_ID           aistudio-api
  OIDC_CLIENT_SECRET       ...
  LDAP_ENABLED             true|false
  LDAP_URL                 ldap://ldap.company.com
  LDAP_BASE_DN             DC=company,DC=com
  LDAP_BIND_DN             CN=service,DC=company,DC=com
  LDAP_BIND_PASS           ...

TOOL-RUNTIME
  SANDBOX_MODE             linux_namespace|gvisor|windows_job
  MAX_CONCURRENT_SANDBOXES 4
  SANDBOX_RAM_LIMIT_MB     2048
  SANDBOX_CPU_LIMIT        2.0
  SANDBOX_TIMEOUT_SECONDS  300
  WORKSPACE_BASE_PATH      /workspace
  NETWORK_ALLOW_LIST       pypi.org,npmjs.com,github.com

DESKTOP
  COORDINATOR_URL          http://localhost:8090
  LOG_LEVEL                INFO
  THEME                    dark|light|auto
```

# APPENDIX B: KUZU QUERY EXAMPLES

```python
# knowledge-engine/graph/queries.py — Named queries available to agents

NAMED_QUERIES = {
    "file_hotspots_and_dependencies": """
        MATCH (f:File)-[r:DEFINED_IN_FILE]-(c:Class)-[:DEFINED_IN_CLASS]-(m:Method)
        WHERE f.project_id = $project_id
        WITH f, count(m) as method_count
        ORDER BY method_count DESC
        LIMIT 20
        RETURN f.path, method_count
    """,

    "callers_of_method": """
        MATCH (caller:Method)-[:CALLS]->(target:Method {name: $method_name})
        WHERE target.project_id = $project_id
        RETURN caller.name, caller.class_id, count(*) as call_count
        ORDER BY call_count DESC
    """,

    "tables_accessed_by_file": """
        MATCH (f:File)-[:DEFINED_IN_FILE]-(cls:Class)-[:DEFINED_IN_CLASS]-(m:Method)
        WHERE f.path = $file_path AND f.project_id = $project_id
        MATCH (m)-[:READS|WRITES]->(t:Table)
        RETURN DISTINCT t.table_name, t.schema_name,
               collect(DISTINCT m.name) as methods
    """,

    "dead_code_candidates": """
        MATCH (m:Method)
        WHERE m.project_id = $project_id
          AND NOT EXISTS { MATCH ()-[:CALLS]->(m) }
          AND NOT m.name STARTS WITH 'test_'
          AND NOT m.name IN ['__init__', '__str__', '__repr__', 'main']
        RETURN m.name, m.class_id, m.line_start
        ORDER BY m.line_start
        LIMIT 50
    """,

    "dependency_chain": """
        MATCH path = (start:File {path: $file_path})-[:IMPORTS*1..5]->(dep:File)
        WHERE start.project_id = $project_id
        RETURN [n IN nodes(path) | n.path] as import_chain, length(path) as depth
        ORDER BY depth
    """
}
```

# APPENDIX C: NATS SUBJECT TAXONOMY

```
tasks.*                     Task lifecycle events (coordinator publishes)
  tasks.created             New task submitted
  tasks.updated             Task status changed
  tasks.completed           Task successfully finished
  tasks.failed              Task permanently failed
  tasks.cancelled           Task cancelled by user
  tasks.plan.subtask        PlannerAgent emits subtask specs

agent.*.*                   Agent communication (bidirectional)
  agent.{type}.inbox        Tasks dispatched to agent type (orchestrator -> agent)
  agent.{type}.outbox       Agent results (agent -> coordinator + orchestrator)
  agent.heartbeat           Agent liveness (30s interval from each instance)

memory.*                    Knowledge storage events
  memory.write              Agent requests memory persistence
  memory.invalidate         Stale memory eviction

approval.*                  Human decision events
  approval.requested        New gate requires human review
  approval.granted          Human approved
  approval.rejected         Human rejected
  approval.expired          Gate timeout reached without decision
  approval.auto_approved    Auto-approved (unattended mode + low risk)

ci.*                        CI/CD pipeline events
  ci.run.start              CI pipeline triggered
  ci.run.complete           CI pipeline finished
  ci.test.result            Individual test suite result

events.*                    General system events (audit log)
  events.plugin.installed   Plugin installed
  events.project.created    Project created
  events.user.login         User authenticated
  events.cost.threshold     Cost budget threshold crossed
```

# APPENDIX D: SECURITY CHECKLIST

```
Authentication:
  [x] JWT RS256 with JTI revocation via Redis
  [x] API keys: bcrypt hash (cost=12), prefix-based lookup, expiry support
  [x] OIDC/OAuth2 PKCE in browser; confidential client on server
  [x] LDAP TLS-only (StartTLS or LDAPS)
  [x] Session tokens never in URL parameters

Authorization:
  [x] RBAC: admin / lead / developer / viewer roles
  [x] Project membership check on all project-scoped endpoints
  [x] Plugin install restricted to admin (global) or lead (project)

Data Protection:
  [x] All secrets in environment variables or Docker secrets (never in code)
  [x] PostgreSQL TLS (sslmode=require)
  [x] Redis TLS + AUTH password
  [x] NATS TLS + per-service credentials
  [x] Qdrant with API key authentication
  [x] Kubernetes secrets encrypted at rest (etcd encryption)
  [x] S3 bucket SSE-S3 encryption

Network:
  [x] tool-runtime: no outbound internet except allow-list
  [x] agent-runtime: no direct inbound connections (NATS only)
  [x] Internal services: not exposed outside cluster
  [x] Ingress: TLS termination at nginx, HSTS header
  [x] CORS: explicit allow-list, no wildcard in production

Sandboxing:
  [x] Linux namespace isolation for all shell tools (laptop + team)
  [x] gVisor (runsc) for production Kubernetes
  [x] Resource limits: 2GB RAM, 2 CPU, 10GB disk per sandbox
  [x] Network restricted: only allow-listed domains
  [x] No sandbox can access host filesystem outside /workspace

Audit:
  [x] All API requests logged with user_id, method, path, status, duration
  [x] All approval decisions stored permanently (no deletion)
  [x] All agent executions logged with token counts and cost
  [x] All tool calls logged with success/failure
  [x] Sensitive fields (JWT, API keys) never appear in logs
```

---

*AI Studio 2.0 — Implementation Guide*  
*Engineering Specification — 2026-06-27*  
*Ready for Implementation*
