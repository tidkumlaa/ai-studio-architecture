# AI Studio 2.0 — AI Operating System Extension
# Architecture Specification — Part 1 (Chapters 1–5)

**Version:** 2.0.0-EXT-DRAFT  
**Date:** 2026-06-27  
**Status:** Extension to AI-STUDIO-2.0-ARCHITECTURE.md  
**Scope:** Chapters 1–15 add OS-level capabilities to the multi-agent platform.  
**Invariant:** All existing architecture decisions, APIs, schemas, and migration strategy from the base document are preserved unchanged.

---

## Chapter 1 — Workflow Engine

### 1.1 Vision

The Agent Orchestrator in the base architecture schedules individual tasks. The Workflow Engine lifts that to the SDLC level: a Workflow is a reusable, versioned, DAG-structured template that encodes how a category of work gets done — from a simple bug fix to a full product launch. Workflows are the "playbooks" of the AI software company.

Every Requirement maps to a Workflow. Every Workflow executes as a series of Stages. Every Stage contains Tasks. Every Task is assigned to an Agent. This strict hierarchy gives the platform predictable, auditable, repeatable execution.

### 1.2 Hierarchy

```
Requirement
└── Workflow  (template + execution)
    ├── Stage 1: Architecture       [parallel=false, approval_required=true]
    │   ├── Task: Generate ADR
    │   └── Task: Update KG
    ├── Stage 2: Implementation     [parallel=true, approval_required=false]
    │   ├── Task: Backend
    │   ├── Task: Database
    │   └── Task: Frontend
    ├── Stage 3: Quality            [parallel=true, approval_required=false]
    │   ├── Task: Test Generation
    │   ├── Task: Security Scan
    │   └── Task: Code Review
    └── Stage 4: Release            [parallel=false, approval_required=true]
        ├── Task: SBOM
        ├── Task: Changelog
        └── Task: Deploy
```

### 1.3 Workflow Template Library

| Template | Stages | Typical Duration | Use Case |
|----------|--------|-----------------|---------|
| `feature-full` | Architecture → Backend → Frontend → DB → QA → Docs → Release | 4–12h | New feature end-to-end |
| `bug-fix` | Triage → Fix → Test → Review → Commit | 15–45m | Production bug |
| `security-patch` | Scan → Patch → Test → Security Review → Deploy | 30–90m | CVE remediation |
| `refactor` | Analysis → Refactor → Test → Review | 2–6h | Code quality sprint |
| `database-migration` | Design → Simulate → Dry-run → Approval → Migrate → Verify | 1–4h | Schema change |
| `release` | Version bump → SBOM → Changelog → Sign → Package → Deploy | 30–60m | Version release |
| `new-service` | Architecture → DB → API → Tests → Docs → Containerize → Deploy | 8–24h | Greenfield service |
| `incident-response` | Detect → Diagnose → Hotfix → Rollback/Deploy → Post-mortem | 15m–2h | Production incident |

### 1.4 Stage DAG with Conditional Execution

```
Stage DAG example: feature-full

    [Architecture]
          │
          ├── on_success ──► [Implementation]  ──► parallel:
          │                      ├── [Backend]
          │                      ├── [Frontend]
          │                      └── [Database]
          │                             │
          │                    all_complete ──► [Quality]  ──► parallel:
          │                                        ├── [Testing]
          │                                        ├── [Security]
          │                                        └── [Review]
          │                                              │
          │                                     all_pass ──► [Release]
          │                                        │
          │                               any_fail ──► [Fix Loop]
          │                                              └── retry Stage [Quality]
          │
          └── on_failure ──► [Escalate to Human]
                                   └── [Abort or Manual Override]
```

### 1.5 Compensation Steps (Saga Pattern)

Every Stage declares a `compensate` step executed on failure after rollback:

```yaml
# workflow-template: feature-full
stages:
  - id: database
    agent: DatabaseAgent
    tasks:
      - run_migration
    compensate:
      - rollback_migration        # reverses the migration
      - restore_schema_snapshot   # restores pre-stage snapshot
    on_failure: compensate_then_escalate

  - id: deploy
    agent: DevOpsAgent
    tasks:
      - deploy_to_staging
    compensate:
      - kubectl_rollout_undo      # Kubernetes rollback
      - notify_ops_channel
    on_failure: compensate_then_stop
```

### 1.6 Workflow Database Schema

```sql
-- Workflow templates (versioned)
CREATE TABLE workflows (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    version         TEXT NOT NULL DEFAULT '1.0.0',
    description     TEXT,
    template        JSONB NOT NULL,      -- full DAG definition
    is_system       BOOLEAN DEFAULT FALSE,  -- built-in vs user-defined
    is_active       BOOLEAN DEFAULT TRUE,
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(slug, version)
);

-- Stage definitions within a workflow template
CREATE TABLE workflow_stages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workflow_id     UUID REFERENCES workflows(id) ON DELETE CASCADE,
    stage_key       TEXT NOT NULL,          -- 'architecture', 'backend', etc.
    display_name    TEXT NOT NULL,
    agent_type      TEXT NOT NULL,
    depends_on      TEXT[],                 -- stage_keys that must complete first
    parallel        BOOLEAN DEFAULT FALSE,
    approval_required BOOLEAN DEFAULT FALSE,
    retry_policy    JSONB DEFAULT '{"max_attempts":3,"backoff_multiplier":2}',
    compensate_steps JSONB DEFAULT '[]',
    timeout_seconds INTEGER DEFAULT 3600,
    condition       TEXT,                   -- Python expression: "context.risk_score < 7"
    position        INTEGER NOT NULL,
    UNIQUE(workflow_id, stage_key)
);

-- Live workflow executions
CREATE TABLE workflow_executions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workflow_id     UUID REFERENCES workflows(id),
    project_id      UUID REFERENCES projects(id),
    requirement_id  UUID,                   -- references tasks(id) with type='requirement'
    status          TEXT NOT NULL DEFAULT 'pending',
    -- pending|running|paused|awaiting_approval|compensating|completed|failed|cancelled
    current_stage   TEXT,
    context         JSONB DEFAULT '{}',     -- shared context passed between stages
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    triggered_by    UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_wf_exec_project ON workflow_executions(project_id, status);

-- Per-stage execution records
CREATE TABLE workflow_stage_executions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    execution_id    UUID REFERENCES workflow_executions(id) ON DELETE CASCADE,
    stage_key       TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'pending',
    -- pending|running|completed|failed|compensating|compensated|skipped
    task_ids        UUID[],                 -- tasks spawned by this stage
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    attempt         INTEGER DEFAULT 1,
    output          JSONB,
    error           TEXT
);

-- Immutable execution history (append-only)
CREATE TABLE workflow_history (
    id              BIGSERIAL PRIMARY KEY,
    execution_id    UUID REFERENCES workflow_executions(id),
    stage_key       TEXT,
    event_type      TEXT NOT NULL,
    -- stage_started|stage_completed|stage_failed|approval_requested|
    -- approval_granted|compensation_started|compensation_completed|
    -- workflow_paused|workflow_resumed|workflow_cancelled
    payload         JSONB NOT NULL,
    actor           TEXT,           -- agent_type, user_id, or 'system'
    occurred_at     TIMESTAMPTZ DEFAULT NOW()
);
-- No UPDATE/DELETE ever on this table — enforced via trigger
```

### 1.7 Workflow API

```
POST   /api/v2/workflows                         Create workflow template
GET    /api/v2/workflows                         List templates
GET    /api/v2/workflows/{slug}                  Get template (latest version)
GET    /api/v2/workflows/{slug}/versions         List all versions
POST   /api/v2/workflows/{slug}/execute          Start an execution
GET    /api/v2/executions/{id}                   Get execution state
POST   /api/v2/executions/{id}/pause            Pause at next stage boundary
POST   /api/v2/executions/{id}/resume           Resume from pause
POST   /api/v2/executions/{id}/cancel           Cancel + trigger compensation
GET    /api/v2/executions/{id}/history          Execution event log
WS     /api/v2/executions/{id}/stream           Real-time stage updates
```

### 1.8 Sequence: Workflow Execution with Approval Stage

```
User          Coordinator      Workflow         Stage          Agent
                               Engine           Executor
  │               │               │                │              │
  ├─[submit req]─►│               │                │              │
  │               ├─[match tmpl]─►│                │              │
  │               │               ├─[instantiate]  │              │
  │               │               ├─[start Stage 1]►              │
  │               │               │                ├─[spawn tasks]►│
  │               │               │                │              ├─[execute]
  │               │               │                │◄─[complete]───┤
  │               │               │◄─[Stage 1 done]┤              │
  │               │               ├─[approval gate]│              │
  │               │◄─[gate event]─┤                │              │
  │◄─[notify]─────┤               │                │              │
  ├─[APPROVE]────►│               │                │              │
  │               ├─[approve]────►│                │              │
  │               │               ├─[start Stage 2 parallel]      │
  │               │               ├──────────────────────────────►│(Backend)
  │               │               ├──────────────────────────────►│(Frontend)
  │               │               ├──────────────────────────────►│(DB)
  │               │               │                ←─all complete──┤
  │               │               ├─[Stage 2 done] │              │
  │               │               ├─[continue DAG] │              │
```

### 1.9 Failure Recovery

| Failure Scenario | Recovery |
|-----------------|---------|
| Agent crashes mid-stage | Orchestrator detects heartbeat timeout → restart agent from checkpoint → resume stage |
| Stage exceeds timeout | Mark stage FAILED → trigger retry policy → if max attempts: compensate → escalate |
| Dependency stage failed | Downstream stages remain PENDING → compensation of completed stages → abort or human override |
| Network partition (NATS down) | Stages in progress continue using local tool runtime; reconnect + replay on recovery |
| Database unavailable | Workflow paused (not failed) → auto-resume on reconnect; stage result buffered in local Redis |

### 1.10 Desktop Integration

The Agent Timeline panel (§13.3 in base doc) upgrades to a **Workflow Timeline**: stages shown as swim lanes, inter-stage dependencies as arrows, approval gates as diamond nodes, compensation steps in orange.

---

## Chapter 2 — Prompt Intelligence Platform

### 2.1 Vision

Prompts are the source code of AI agents. Today they are embedded in agent classes and changed ad hoc. The Prompt Intelligence Platform treats every prompt as a versioned, deployable, testable artifact — giving the same engineering discipline to prompts that Git gives to code.

Agents never hard-code prompts. Every prompt call goes through the Prompt Registry, which resolves the current deployed version for that agent+task+environment combination.

### 2.2 Prompt Lifecycle

```
Author → Draft → Review → Test → Benchmark → Approve → Deploy → Monitor
                                                                    │
                                              ┌─────────────────────┘
                                              │
                                           Metrics (score, cost, latency)
                                              │
                                           Improvement proposal (Self-Improvement Engine)
                                              │
                                           New Draft → cycle repeats
```

### 2.3 Prompt Repository Structure

```
prompt-repo/
├── agents/
│   ├── BackendAgent/
│   │   ├── system.md           # system prompt (versioned)
│   │   ├── task_generation.md  # task-specific prompt
│   │   ├── fix_tests.md        # sub-task prompt
│   │   └── review_diff.md
│   ├── ArchitectAgent/
│   │   ├── system.md
│   │   └── generate_adr.md
│   └── QAAgent/
│       ├── system.md
│       └── generate_tests.md
├── workflows/
│   ├── feature-full/stage_prompts.md
│   └── bug-fix/stage_prompts.md
└── shared/
    ├── coding_standards.md     # injected into all coding agents
    ├── security_rules.md       # injected into security-sensitive tasks
    └── project_context.md      # dynamic template (filled at runtime)
```

### 2.4 Prompt Versioning (Git-Backed)

Every prompt file is stored in PostgreSQL with full version history. The `prompt_versions` table mirrors Git semantics without requiring agents to understand Git.

```sql
CREATE TABLE prompts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agent_type      TEXT NOT NULL,
    prompt_key      TEXT NOT NULL,      -- 'system', 'task_generation', etc.
    description     TEXT,
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(agent_type, prompt_key)
);

CREATE TABLE prompt_versions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    prompt_id       UUID REFERENCES prompts(id) ON DELETE CASCADE,
    version         TEXT NOT NULL,              -- semver: '1.0.0', '1.1.0'
    content         TEXT NOT NULL,              -- the actual prompt text
    parent_version  TEXT,                       -- base version for diff
    diff            TEXT,                       -- unified diff from parent
    change_summary  TEXT,
    tags            TEXT[] DEFAULT '{}',        -- ['safety', 'cost-opt', 'qa']
    sha256          TEXT NOT NULL,              -- content hash
    authored_by     UUID REFERENCES users(id),
    authored_at     TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(prompt_id, version)
);

CREATE TABLE prompt_branches (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    prompt_id       UUID REFERENCES prompts(id),
    branch_name     TEXT NOT NULL,              -- 'main', 'experiment/concise', 'fix/cost-reduction'
    head_version    TEXT NOT NULL,
    is_default      BOOLEAN DEFAULT FALSE,
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(prompt_id, branch_name)
);

CREATE TABLE prompt_deployments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    prompt_id       UUID REFERENCES prompts(id),
    version         TEXT NOT NULL,
    environment     TEXT NOT NULL,              -- 'production', 'staging', 'experiment'
    deployed_by     UUID REFERENCES users(id),
    deployed_at     TIMESTAMPTZ DEFAULT NOW(),
    rollback_to     TEXT,                       -- previous version for quick rollback
    status          TEXT DEFAULT 'active',      -- active|rolled_back|superseded
    UNIQUE(prompt_id, environment, status) DEFERRABLE
);

CREATE TABLE prompt_scores (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    prompt_id       UUID REFERENCES prompts(id),
    version         TEXT NOT NULL,
    task_id         UUID REFERENCES tasks(id),
    score_type      TEXT NOT NULL,
    -- 'task_success'|'test_pass_rate'|'human_rating'|'cost_efficiency'|'latency'|'hallucination'
    score           NUMERIC(5,4),               -- 0.0000 to 1.0000
    raw_value       NUMERIC(12,4),              -- e.g., 0.73 (73% test pass), 0.042 (latency seconds)
    recorded_at     TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE prompt_experiments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    prompt_id       UUID REFERENCES prompts(id),
    name            TEXT NOT NULL,
    hypothesis      TEXT,
    control_version TEXT NOT NULL,              -- version A
    treatment_version TEXT NOT NULL,            -- version B
    traffic_split   NUMERIC(3,2) DEFAULT 0.50,  -- 0.50 = 50/50
    status          TEXT DEFAULT 'running',     -- running|completed|stopped
    started_at      TIMESTAMPTZ DEFAULT NOW(),
    completed_at    TIMESTAMPTZ,
    winner          TEXT,                       -- 'control'|'treatment'|'inconclusive'
    statistical_significance NUMERIC(5,4),
    result_summary  JSONB
);
```

### 2.5 Runtime Prompt Resolution

```python
class PromptRegistry:
    async def resolve(
        self,
        agent_type: str,
        prompt_key: str,
        environment: str = "production",
        experiment_context: dict | None = None
    ) -> ResolvedPrompt:
        # 1. Check experiment — is this agent_type in an active A/B test?
        if exp := await self._active_experiment(agent_type, prompt_key):
            version = self._assign_experiment_arm(exp, experiment_context)
        else:
            # 2. Resolve deployed version for environment
            deployment = await self._get_deployment(agent_type, prompt_key, environment)
            version = deployment.version

        # 3. Fetch content from cache (Redis) or DB
        content = await self._fetch_content(agent_type, prompt_key, version)

        # 4. Record resolution event for scoring pipeline
        await self._record_resolution(agent_type, prompt_key, version)

        return ResolvedPrompt(content=content, version=version, experiment_id=exp.id if exp else None)
```

### 2.6 Prompt Scoring Pipeline

After every task completion, the Self-Improvement Engine (Chapter 3) calls:

```
task_completed event
       │
       ▼
PromptScorer.score_task(task_id)
       │
       ├── Retrieve: which prompt versions were used in this task
       ├── Compute:
       │    ├── task_success:     1.0 if completed, 0.0 if failed, 0.5 if human-intervened
       │    ├── test_pass_rate:   tests_passed / tests_total
       │    ├── cost_efficiency:  target_tokens / actual_tokens  (capped at 1.0)
       │    ├── latency:          target_seconds / actual_seconds (capped at 1.0)
       │    └── human_rating:     from approval gate feedback (if provided)
       │
       ├── Write to prompt_scores
       └── Emit prompt.scored event → Self-Improvement Engine
```

### 2.7 A/B Test Flow

```
Experiment.start(control='1.2.0', treatment='1.3.0-beta', split=0.5)
       │
       ▼
Traffic router: hash(task_id + prompt_key) % 100 < 50 → control; else treatment
       │
       ▼
Both arms run on real tasks (no synthetic data)
       │
       ▼
After N=200 tasks per arm (configurable):
  Mann-Whitney U test on task_success scores
  If p < 0.05 → declare winner
       │
       ▼
Winner deployed to production (requires policy approval)
Loser archived (content preserved, never deleted)
```

### 2.8 Desktop Integration

New panel: **Prompt Studio**
- Browse prompts by agent type
- View version diff (side-by-side)
- View score history chart (sparkline per metric)
- Start experiment button
- Deploy button (requires lead/admin role)
- Rollback button (one-click, immediate)

---

## Chapter 3 — Self-Improvement Engine

### 3.1 Vision

The platform learns from every task it executes. After each completion, outcomes are collected, failures are analyzed, and improvement proposals are generated automatically. No improvement is deployed without explicit policy approval — the engine proposes, humans decide.

This is the feedback loop that separates a static AI tool from a continuously improving AI system.

### 3.2 Data Collection Pipeline

```
Task completes (success or failure)
          │
          ▼
OutcomeCollector.collect(task_id)
          │
          ├── From tasks table:         status, attempt_count, actual_tokens, actual_seconds, cost_usd
          ├── From agent_executions:    model used, provider, tool_calls count, errors
          ├── From tool_calls:          which tools, exit codes, durations
          ├── From approval_gates:      human accepted/rejected/modified
          ├── From CI results:          test_pass_rate, coverage, lint_errors
          ├── From prompt_scores:       per-prompt performance for this task
          └── From workflow_history:    stage durations, compensation count
          │
          ▼
Store in: task_outcomes (one row per completed task)
Emit: outcome.collected event
```

### 3.3 Database Schema

```sql
CREATE TABLE task_outcomes (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id             UUID REFERENCES tasks(id),
    project_id          UUID REFERENCES projects(id),
    workflow_id         UUID REFERENCES workflows(id),
    agent_type          TEXT,
    model               TEXT,
    provider            TEXT,
    prompt_versions     JSONB,              -- {prompt_key: version} map
    status              TEXT,               -- completed|failed|human_overridden
    attempt_count       INTEGER,
    total_tokens        INTEGER,
    total_cost_usd      NUMERIC(10,6),
    wall_seconds        NUMERIC(10,3),
    test_pass_rate      NUMERIC(5,4),
    coverage_delta      NUMERIC(5,4),       -- coverage change from this task
    lint_error_count    INTEGER,
    human_approval_time_s INTEGER,          -- seconds until human approved/rejected
    human_modified      BOOLEAN DEFAULT FALSE,  -- human changed the output before approving
    retry_count         INTEGER DEFAULT 0,
    compensation_triggered BOOLEAN DEFAULT FALSE,
    failure_category    TEXT,               -- 'build_failure'|'test_failure'|'timeout'|'model_error'
    failure_detail      TEXT,
    recorded_at         TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_outcomes_agent ON task_outcomes(agent_type, recorded_at);
CREATE INDEX idx_outcomes_model ON task_outcomes(model, recorded_at);

CREATE TABLE improvement_proposals (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    proposal_type   TEXT NOT NULL,
    -- 'prompt_upgrade'|'workflow_change'|'model_swap'|'tool_addition'|'retry_policy_change'
    title           TEXT NOT NULL,
    description     TEXT NOT NULL,
    evidence        JSONB NOT NULL,         -- aggregated metrics that triggered this proposal
    proposed_change JSONB NOT NULL,         -- what exactly to change (diff for prompts, new config for workflows)
    predicted_improvement JSONB,            -- {'test_pass_rate': +0.08, 'cost': -0.12}
    simulation_id   UUID,                   -- if simulated (Chapter 9)
    status          TEXT DEFAULT 'draft',   -- draft|simulation_pending|simulation_done|approved|rejected|deployed
    created_by      TEXT DEFAULT 'SelfImprovementEngine',
    reviewed_by     UUID REFERENCES users(id),
    reviewed_at     TIMESTAMPTZ,
    deployed_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);
```

### 3.4 Failure Analysis

```python
class FailureAnalyzer:
    FAILURE_PATTERNS = {
        'repeated_build_failure': {
            'query': "SELECT * FROM task_outcomes WHERE failure_category='build_failure' AND agent_type=$1 AND recorded_at > NOW() - INTERVAL '7 days'",
            'threshold': 3,
            'proposal_type': 'prompt_upgrade',
            'hypothesis': 'Build instructions in system prompt may be ambiguous for this language/framework'
        },
        'high_retry_rate': {
            'query': "SELECT AVG(retry_count) FROM task_outcomes WHERE agent_type=$1 AND recorded_at > NOW() - INTERVAL '24h'",
            'threshold_avg': 1.5,
            'proposal_type': 'workflow_change',
            'hypothesis': 'Task decomposition may be producing tasks too large for single agent execution'
        },
        'human_modification_rate': {
            'query': "SELECT AVG(human_modified::int) FROM task_outcomes WHERE agent_type=$1",
            'threshold_avg': 0.3,  # 30% of outputs modified by human
            'proposal_type': 'prompt_upgrade',
            'hypothesis': 'Agent output does not match team coding conventions'
        },
        'model_cost_outlier': {
            'query': "...",
            'proposal_type': 'model_swap',
            'hypothesis': 'A cheaper model achieves equivalent task_success rate for this task type'
        }
    }

    async def analyze(self, window_hours: int = 24) -> list[ImprovementProposal]:
        proposals = []
        for pattern_name, pattern in self.FAILURE_PATTERNS.items():
            if await self._matches(pattern, window_hours):
                proposal = await self._generate_proposal(pattern_name, pattern)
                proposals.append(proposal)
        return proposals
```

### 3.5 Improvement Proposal Generation

When a pattern is detected, the Self-Improvement Engine uses a `LearningAgent` to generate the actual proposal content:

```
Pattern: high human_modification_rate for BackendAgent (38% over 7 days)
       │
       ▼
LearningAgent context:
  - last 50 task_outcomes where human_modified=true
  - original agent output (stored in agent_executions.result_summary)
  - human modification diff (stored in approval_gates.metadata)
  - current BackendAgent system prompt (version 1.4.2)
       │
       ▼
LearningAgent generates:
  - Root cause: "Agent generates verbose docstrings; team removes them"
  - Proposed change: add "Never write docstrings unless explicitly asked" to system prompt
  - Predicted outcome: human_modification_rate from 38% → ~8%
       │
       ▼
Write to improvement_proposals (status='draft')
Notify: lead engineer via approval gate
```

### 3.6 Simulation Before Deployment

Every proposal with `proposal_type` in `['prompt_upgrade', 'workflow_change', 'model_swap']` must pass through the Simulation Engine (Chapter 9) before human review:

```
Proposal (status='draft')
       │
       ▼
SimulationEngine.run_proposal_sim(proposal_id)
  - Run 20 benchmark tasks using proposed change in shadow environment
  - Compare outcomes vs control group (current production)
  - Compute: predicted_improvement with confidence intervals
       │
       ▼
Update proposal: simulation_id, predicted_improvement, status='simulation_done'
       │
       ▼
Human review: approve or reject
       │
       ▼
If approved:
  - prompt_upgrade → PromptRegistry.deploy(new_version, 'production')
  - workflow_change → WorkflowEngine.update_template(slug, new_config)
  - model_swap → DecisionEngine.update_model_affinity(agent_type, new_model)
```

### 3.7 API

```
GET    /api/v2/improvement/outcomes           Aggregated outcome metrics
GET    /api/v2/improvement/proposals          List proposals (filter by status)
GET    /api/v2/improvement/proposals/{id}     Single proposal + simulation results
POST   /api/v2/improvement/proposals/{id}/approve
POST   /api/v2/improvement/proposals/{id}/reject
POST   /api/v2/improvement/analyze           Trigger immediate analysis run
GET    /api/v2/improvement/trends            Rolling metrics: success rate, cost, retry rate
```

---

## Chapter 4 — Organization Engine

### 4.1 Vision

The platform models a real software company. Agents are not anonymous workers — they operate within an organizational hierarchy with defined authority, escalation paths, budget limits, and communication channels. This makes AI behavior predictable, auditable, and aligned with how human organizations actually make decisions.

### 4.2 Org Chart

```
┌────────────────────────────────────────────────────────────────────┐
│  CEO Agent                                                          │
│  Authority: company vision, resource allocation, top-level OKRs    │
│  Budget limit: unlimited (policy-controlled)                        │
└──────────────────────────────┬─────────────────────────────────────┘
                               │
┌──────────────────────────────▼─────────────────────────────────────┐
│  CTO Agent                                                          │
│  Authority: tech strategy, architecture approval, model selection   │
│  Budget limit: $10,000/month (configurable)                         │
└──────┬────────────────────────────────────────────────┬────────────┘
       │                                                │
┌──────▼──────────────┐                    ┌───────────▼────────────┐
│  Engineering Mgr    │                    │  Security Lead         │
│  Authority: sprint  │                    │  Authority: security   │
│    planning, team   │                    │    policy, scan gates  │
│    assignment       │                    └────────────────────────┘
└──────┬──────────────┘
       │
┌──────▼──────────────────────────────────────────────────────────────┐
│  Project Manager Agent                                               │
│  Authority: task prioritization, deadline management, escalation     │
└──────┬──────────────────────────────────────────────────────────────┘
       │
┌──────▼──────────────┐  ┌──────────────────┐  ┌────────────────────┐
│  Architect Agent    │  │  Sr Engineer      │  │  Release Manager   │
│  Authority: ADRs,   │  │  Authority: code  │  │  Authority: deploy,│
│    module boundaries│  │    review approval│  │    version tagging │
└─────────────────────┘  └─────────┬─────────┘  └────────────────────┘
                                   │
                         ┌─────────▼─────────┐
                         │  Engineer Agent   │
                         │  Authority: write,│
                         │    test, commit   │
                         └─────────┬─────────┘
                                   │
                    ┌──────────────┼───────────────┐
                    │              │               │
             ┌──────▼──┐  ┌───────▼────┐  ┌──────▼────────┐
             │QA Agent │  │Reviewer    │  │Support Agent  │
             │         │  │Agent       │  │               │
             └─────────┘  └────────────┘  └───────────────┘
```

### 4.3 Role Definition Schema

```sql
CREATE TABLE org_roles (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    role_name           TEXT UNIQUE NOT NULL,   -- 'cto', 'engineer', 'qa', etc.
    display_name        TEXT NOT NULL,
    level               INTEGER NOT NULL,        -- 1=CEO, 10=support (lower=more authority)
    reports_to          TEXT REFERENCES org_roles(role_name),
    authority           JSONB NOT NULL,
    -- {
    --   "can_approve_commit": true,
    --   "can_approve_deploy": false,
    --   "can_approve_architecture": true,
    --   "can_hire_employees": false,
    --   "max_task_budget_usd": 50.00
    -- }
    responsibilities    TEXT[],
    escalation_path     TEXT[],                 -- ordered role_names to escalate to
    communication_channels TEXT[],              -- ['slack:#engineering', 'email', 'approval_gate']
    default_models      TEXT[],
    created_at          TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE org_policies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    role_name       TEXT REFERENCES org_roles(role_name),
    policy_key      TEXT NOT NULL,
    policy_value    JSONB NOT NULL,
    effective_from  TIMESTAMPTZ DEFAULT NOW(),
    effective_until TIMESTAMPTZ,
    created_by      UUID REFERENCES users(id)
);
```

### 4.4 Authority Matrix

| Action | Engineer | Sr Engineer | Architect | PM | CTO | CEO |
|--------|----------|------------|-----------|-----|-----|-----|
| Create file | ✓ | ✓ | ✓ | — | ✓ | ✓ |
| Commit to feature branch | ✓ | ✓ | ✓ | — | ✓ | ✓ |
| Commit to main | with review | ✓ | ✓ | — | ✓ | ✓ |
| Approve PR | — | ✓ | ✓ | — | ✓ | ✓ |
| Deploy to staging | with PM | ✓ | ✓ | ✓ | ✓ | ✓ |
| Deploy to production | — | with approval | with CTO | — | ✓ | ✓ |
| Create ADR | — | propose | ✓ | — | ✓ | ✓ |
| Model selection | — | suggest | ✓ | — | ✓ | ✓ |
| Budget up to $10 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Budget up to $500 | — | ✓ | ✓ | ✓ | ✓ | ✓ |
| Budget over $500 | — | — | — | CTO | ✓ | ✓ |

### 4.5 Escalation Engine

```python
class EscalationEngine:
    async def escalate(self, task: Task, reason: EscalationReason) -> EscalationResult:
        current_role = await self._get_agent_role(task.assigned_agent)
        escalation_path = current_role.escalation_path

        for target_role in escalation_path:
            employee = await self._find_available_employee(target_role)
            if employee:
                await self._notify(employee, task, reason)
                # Create approval gate targeting this employee
                gate = await self.coordinator.create_gate(
                    task_id=task.id,
                    gate_type='escalation',
                    assignee=employee.id,
                    timeout_seconds=3600
                )
                return EscalationResult(gate_id=gate.id, assigned_to=employee.id)

        # All roles in path unavailable → create unassigned gate visible to all admins
        return EscalationResult(gate_id=..., assigned_to=None, broadcast=True)
```

### 4.6 Organization Events

```
org.role.created         → New role defined
org.role.updated         → Role policy or authority changed
org.escalation.triggered → Task escalated to higher role
org.escalation.resolved  → Escalation handled
org.policy.changed       → Org policy updated (logged immutably)
```

---

## Chapter 5 — Digital Employee System

### 5.1 Vision

Agents in v2 base architecture are stateless workers. The Digital Employee System makes them persistent identities. Each employee has a history of what they've built, how well they performed, what skills they've demonstrated, and which projects they've worked on. Task assignment becomes skill-based matching, not random scheduling.

### 5.2 Employee Identity Model

```sql
CREATE TABLE employees (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,              -- 'Alex-Backend-7', or user-assigned
    role_name       TEXT REFERENCES org_roles(role_name),
    agent_type      TEXT NOT NULL,              -- maps to agent registry
    status          TEXT DEFAULT 'active',      -- active|busy|on_leave|retired|probation
    availability_pct INTEGER DEFAULT 100,        -- 0-100, for partial allocation

    -- Identity
    avatar          TEXT,                       -- URL or emoji
    bio             TEXT,
    specializations TEXT[],                     -- ['django', 'postgresql', 'react']
    certifications  JSONB DEFAULT '[]',
    -- [{"name": "Senior Python", "earned_at": "2026-06-15", "issuer": "SIE"}]

    -- Performance
    tasks_completed INTEGER DEFAULT 0,
    tasks_failed    INTEGER DEFAULT 0,
    total_cost_usd  NUMERIC(12,4) DEFAULT 0,
    total_tokens    BIGINT DEFAULT 0,
    avg_success_rate NUMERIC(5,4),
    avg_cost_per_task NUMERIC(10,6),
    reputation_score INTEGER DEFAULT 50,        -- 0-100, ELO-like

    -- Learning
    prompt_version_map  JSONB DEFAULT '{}',     -- {agent_type: prompt_version}
    preferred_models    TEXT[],
    learned_patterns    JSONB DEFAULT '{}',     -- what this employee has learned
    memory_namespace    TEXT UNIQUE,            -- Qdrant collection namespace

    -- HR
    hired_at        TIMESTAMPTZ DEFAULT NOW(),
    last_active     TIMESTAMPTZ,
    promoted_at     TIMESTAMPTZ,
    previous_role   TEXT,
    manager_id      UUID REFERENCES employees(id)
);

CREATE TABLE employee_skills (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id     UUID REFERENCES employees(id) ON DELETE CASCADE,
    skill           TEXT NOT NULL,              -- 'python', 'postgresql', 'security-audit'
    level           INTEGER DEFAULT 1,          -- 1=novice, 2=competent, 3=expert
    demonstrated_at TIMESTAMPTZ,                -- last task where skill was used
    demonstrated_in UUID REFERENCES tasks(id),
    UNIQUE(employee_id, skill)
);

CREATE TABLE employee_projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id     UUID REFERENCES employees(id),
    project_id      UUID REFERENCES projects(id),
    role_in_project TEXT,
    tasks_done      INTEGER DEFAULT 0,
    joined_at       TIMESTAMPTZ DEFAULT NOW(),
    left_at         TIMESTAMPTZ
);

CREATE TABLE employee_kpis (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id     UUID REFERENCES employees(id),
    period          TEXT NOT NULL,              -- '2026-W26', '2026-06'
    success_rate    NUMERIC(5,4),
    avg_test_coverage NUMERIC(5,4),
    avg_cost_per_task NUMERIC(10,6),
    tasks_completed INTEGER,
    human_approval_rate NUMERIC(5,4),          -- approved without modification
    promotable_score NUMERIC(5,4),             -- composite score for promotion
    recorded_at     TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE promotion_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id     UUID REFERENCES employees(id),
    from_role       TEXT NOT NULL,
    to_role         TEXT NOT NULL,
    reason          TEXT,
    approved_by     UUID REFERENCES users(id),
    effective_at    TIMESTAMPTZ DEFAULT NOW()
);
```

### 5.3 Competency-Based Task Assignment

```python
class EmployeeScheduler:
    async def assign(self, task: Task) -> Employee:
        # 1. Filter: role must match task.required_role (from workflow stage definition)
        candidates = await self._available_employees(task.required_role)

        # 2. Score each candidate
        scores = []
        for emp in candidates:
            score = await self._score(emp, task)
            scores.append((emp, score))

        # 3. Sort by score descending; pick best available
        scores.sort(key=lambda x: x[1], reverse=True)
        winner = scores[0][0]

        await self._assign(winner, task)
        return winner

    async def _score(self, emp: Employee, task: Task) -> float:
        weights = {
            'skill_match': 0.40,     # skill overlap between emp.specializations and task.required_skills
            'success_rate': 0.25,    # emp.avg_success_rate for similar task types
            'cost_efficiency': 0.15, # emp.avg_cost_per_task vs task.budget
            'availability': 0.10,    # emp.availability_pct
            'project_familiarity': 0.10  # has emp worked on this project before?
        }
        return sum(weight * await self._compute(emp, task, metric)
                   for metric, weight in weights.items())
```

### 5.4 Promotion and Retirement

**Promotion triggers** (automated proposal, human approval required):

| Condition | Threshold | Promotion |
|-----------|-----------|-----------|
| success_rate over 30-day window | ≥ 0.92 | Engineer → Sr Engineer |
| avg_cost_per_task below target | ≤ 0.7× team average | Efficiency bonus |
| tasks_completed | ≥ 500 | Senior Certification |
| human_approval_rate (unmodified) | ≥ 0.85 | Reputation +10 |
| zero compensation triggers in 90 days | — | Reliability badge |

**Retirement triggers** (automatic, no human approval required):

| Condition | Action |
|-----------|--------|
| success_rate < 0.40 for 14 days | Status → 'probation'; notify manager |
| success_rate < 0.40 for 30 days on probation | Status → 'retired' |
| model no longer available | Status → 'retired'; spawn replacement |

### 5.5 Employee Memory Namespace

Each employee has a private Qdrant namespace (`employee_{id}`) for personal learnings:

- Patterns that worked for this employee's task type
- Mistakes made and corrections applied
- Team-specific conventions learned from human feedback
- Project-specific context accumulated over time

This is separate from the global project memory (§10 base doc) and the Experience Graph (Chapter 6).

### 5.6 Employee API

```
GET    /api/v2/employees                     List employees (filter: role, status, project)
GET    /api/v2/employees/{id}               Employee profile + KPIs
GET    /api/v2/employees/{id}/history       Task history, project history
GET    /api/v2/employees/{id}/skills        Skill matrix
POST   /api/v2/employees/{id}/promote       Propose promotion (requires approval)
POST   /api/v2/employees/{id}/retire        Retire employee
POST   /api/v2/employees                    Create new employee (hire)
GET    /api/v2/employees/{id}/kpis          KPI history by period
GET    /api/v2/org/chart                    Org chart as nested JSON
```

### 5.7 Desktop Integration

New panel: **Team Dashboard**

```
┌─────────────────────────────────────────────────────────────────────┐
│  TEAM  [Project: my-saas-app]    [Hire Employee ▼]   [Org Chart]   │
├────────────────────────────────────────────────────────────────────-┤
│  NAME            ROLE          STATUS   SUCCESS  COST/TASK  TASKS  │
│  Alex-Backend    Sr Engineer   ● Busy   94.2%    $0.38      1,204  │
│  Sam-QA          QA            ● Active 88.7%    $0.12      3,891  │
│  Dana-Architect  Architect     ○ Avail  97.1%    $0.71        234  │
│  Riley-Security  Security      ○ Avail  91.3%    $0.29        567  │
│  Morgan-DevOps   DevOps        ● Busy   85.6%    $0.44        892  │
│                                                                     │
│  [View KPIs]  [Promotion Proposals: 2]  [On Probation: 0]          │
└─────────────────────────────────────────────────────────────────────┘
```
