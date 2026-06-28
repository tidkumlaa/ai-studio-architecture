# AI Studio 2.0 — AI Operating System Extension
# Architecture Specification

**Version:** 2.0.0-EXT-DRAFT  
**Date:** 2026-06-27  
**Status:** Extension to AI-STUDIO-2.0-ARCHITECTURE.md  
**Scope:** Chapters 1–15 add OS-level capabilities to the multi-agent platform.  
**Invariant:** All existing architecture decisions, APIs, schemas, and migration strategy from the base document are preserved unchanged.

---

## Table of Contents

1. [Workflow Engine](#chapter-1--workflow-engine)
2. [Prompt Intelligence Platform](#chapter-2--prompt-intelligence-platform)
3. [Self-Improvement Engine](#chapter-3--self-improvement-engine)
4. [Organization Engine](#chapter-4--organization-engine)
5. [Digital Employee System](#chapter-5--digital-employee-system)
6. [Memory Operating System](#chapter-6--memory-operating-system)
7. [Decision Engine](#chapter-7--decision-engine)
8. [Knowledge Graph 2.0](#chapter-8--knowledge-graph-20)
9. [Simulation Engine](#chapter-9--simulation-engine)
10. [AI Constitution](#chapter-10--ai-constitution)
11. [Governance & Audit](#chapter-11--governance--audit)
12. [Security Platform](#chapter-12--security-platform)
13. [AI Economy Engine](#chapter-13--ai-economy-engine)
14. [Benchmark Center](#chapter-14--benchmark-center)
15. [Evolution Lab](#chapter-15--evolution-lab)
16. [Integration Map](#integration-map)
17. [Revised Technology Stack](#revised-technology-stack-extended)
18. [Revised Development Roadmap](#revised-development-roadmap-extensions)
19. [5–10 Year Product Vision](#510-year-product-vision)

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
    availability_pct INTEGER DEFAULT 100,

    -- Identity
    avatar          TEXT,
    bio             TEXT,
    specializations TEXT[],                     -- ['django', 'postgresql', 'react']
    certifications  JSONB DEFAULT '[]',

    -- Performance
    tasks_completed INTEGER DEFAULT 0,
    tasks_failed    INTEGER DEFAULT 0,
    total_cost_usd  NUMERIC(12,4) DEFAULT 0,
    total_tokens    BIGINT DEFAULT 0,
    avg_success_rate NUMERIC(5,4),
    avg_cost_per_task NUMERIC(10,6),
    reputation_score INTEGER DEFAULT 50,        -- 0-100, ELO-like

    -- Learning
    prompt_version_map  JSONB DEFAULT '{}',
    preferred_models    TEXT[],
    learned_patterns    JSONB DEFAULT '{}',
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
    skill           TEXT NOT NULL,
    level           INTEGER DEFAULT 1,          -- 1=novice, 2=competent, 3=expert
    demonstrated_at TIMESTAMPTZ,
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
    human_approval_rate NUMERIC(5,4),
    promotable_score NUMERIC(5,4),
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
        candidates = await self._available_employees(task.required_role)
        scores = [(emp, await self._score(emp, task)) for emp in candidates]
        scores.sort(key=lambda x: x[1], reverse=True)
        winner = scores[0][0]
        await self._assign(winner, task)
        return winner

    async def _score(self, emp: Employee, task: Task) -> float:
        weights = {
            'skill_match': 0.40,        # skill overlap with task.required_skills
            'success_rate': 0.25,       # historical success on similar task types
            'cost_efficiency': 0.15,    # avg_cost_per_task vs task.budget
            'availability': 0.10,       # emp.availability_pct
            'project_familiarity': 0.10 # has emp worked on this project before?
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

Each employee has a private Qdrant namespace (`employee_{id}`) storing personal learnings: patterns that worked, mistakes corrected, team conventions absorbed from human feedback, and project-specific context. This is separate from the global project memory (§10 base doc) and the Experience Graph (Chapter 6).

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
├─────────────────────────────────────────────────────────────────────┤
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

---

## Chapter 6 — Memory Operating System

### 6.1 Vision

Traditional RAG retrieves documents similar to a query. The Memory Operating System stores *experiences* — structured records of problems encountered, decisions made, solutions applied, outcomes observed, and lessons learned. It is organizational long-term memory that accumulates across projects, employees, and time.

The Experience Graph makes the platform wiser with every task it executes.

### 6.2 Experience Graph Structure

```
┌─────────────────────────────────────────────────────────────────────┐
│  EXPERIENCE GRAPH (Neo4j / Kuzu extension)                          │
│                                                                     │
│  (:Problem)──[:TRIGGERED]──►(:Decision)──[:PRODUCED]──►(:Solution)  │
│       │                          │                          │       │
│       │                     [:USED_MODEL]            [:RESULTED_IN] │
│       │                          │                          │       │
│  [:OCCURRED_IN]            (:AIModel)               (:Outcome)      │
│       │                                                  │         │
│  (:Project)                                        [:GENERATED]     │
│       │                                                  │         │
│  [:INVOLVED]                                       (:Lesson)        │
│       │                                                  │         │
│  (:Employee)                                      [:APPLIES_TO]     │
│                                                          │         │
│                                               (:Module|:AgentType) │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.3 Node Types

| Node | Key Properties | Created By |
|------|---------------|-----------|
| `Problem` | description, category, severity, first_seen, occurrence_count | MonitoringAgent, LearningAgent |
| `Decision` | rationale, alternatives_considered, decided_by, decided_at | ArchitectAgent, humans |
| `Solution` | approach, implementation_summary, files_changed | BackendAgent, DevOpsAgent |
| `Outcome` | success, cost_usd, test_pass_rate, time_to_resolve, human_override | Self-Improvement Engine |
| `Lesson` | title, content, applicability, confidence | LearningAgent |
| `Failure` | error_type, error_message, root_cause, reproduction_steps | Self-Improvement Engine |
| `Success` | approach, what_worked, why_it_worked | Self-Improvement Engine |

### 6.4 Experience Creation Pipeline

```
Task completes
     │
     ▼
ExperienceRecorder.record(task_id)
     │
     ├── Problem: extract from task.description + any error messages
     ├── Decision: extract key choices from agent_executions (which approach chosen)
     ├── Solution: summarize files_changed + approach taken
     ├── Outcome: from task_outcomes (Chapter 3)
     ├── Lesson: generated by LearningAgent if outcome was notably good or bad
     │
     ├── MERGE nodes (idempotent — same problem seen before → increment occurrence_count)
     ├── Create relationships
     └── Embed nodes → Qdrant (experience collection)
```

### 6.5 Experience Retrieval

Before any agent execution, the Knowledge Engine queries the Experience Graph:

```cypher
// "What experience do we have with database migration failures?"
MATCH (p:Problem {category: 'database_migration'})
     -[:TRIGGERED]->(d:Decision)
     -[:PRODUCED]->(s:Solution)
     -[:RESULTED_IN]->(o:Outcome)
     -[:GENERATED]->(l:Lesson)
WHERE o.success = true
RETURN p.description, d.rationale, s.approach, l.content
ORDER BY o.test_pass_rate DESC, o.cost_usd ASC
LIMIT 5

// "Has this exact error occurred before? What fixed it?"
MATCH (f:Failure {error_type: $error_type})
     <-[:CAUSED]-(p:Problem)
     -[:TRIGGERED]->(d:Decision)
     -[:PRODUCED]->(s:Solution)
     -[:RESULTED_IN]->(o:Outcome {success: true})
RETURN f, d.rationale, s.implementation_summary, o.time_to_resolve
ORDER BY o.time_to_resolve ASC
LIMIT 3
```

### 6.6 Cross-Project Learning

The Experience Graph is **org-global**, not project-scoped. When a BackendAgent at project A encounters a Django migration issue, it can retrieve experiences from project B where the same pattern appeared and was resolved.

Privacy controls:
- Projects can be marked `private` → experiences tagged with `visibility=private`
- Private experiences are not retrieved by agents in other projects
- Enterprise: tenant-isolated Experience Graphs (separate Neo4j databases per tenant)

### 6.7 Database Schema

```sql
CREATE TABLE experiences (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID REFERENCES projects(id),
    task_id         UUID REFERENCES tasks(id),
    type            TEXT NOT NULL,  -- 'problem_solution'|'decision'|'lesson'|'failure'|'success'
    title           TEXT NOT NULL,
    content         JSONB NOT NULL,
    outcome_success BOOLEAN,
    outcome_cost    NUMERIC(10,6),
    visibility      TEXT DEFAULT 'org',   -- 'org'|'project'|'private'
    tags            TEXT[],
    qdrant_id       UUID,
    kg_node_id      TEXT,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_experiences_type ON experiences(type, outcome_success);
CREATE INDEX idx_experiences_tags ON experiences USING GIN(tags);
```

### 6.8 Memory Compaction

Monthly, the LearningAgent runs `memory_compaction`:
1. Group similar Problems (cosine similarity > 0.92 in vector space)
2. Merge duplicates → single canonical Problem node with `occurrence_count` incremented
3. Summarize multiple Solutions for same Problem → "best known solution" Lesson
4. Archive Failures older than 1 year with `outcome_success=true` (fixed and stable)
5. Promote high-confidence Lessons → shared `coding_standards.md` (Prompt Platform §2.3)

---

## Chapter 7 — Decision Engine

### 7.1 Vision

Today, task-to-agent assignment is based on agent type matching. The Decision Engine replaces this with a multi-factor decision framework: before every execution, the platform evaluates the best combination of workflow, agent/employee, model, tool configuration, and budget strategy — and records why each decision was made.

This is the intelligence layer that connects all other chapters.

### 7.2 Decision Factors

```
Input: Task + Context
         │
         ▼
Decision Engine evaluates:

  1. WORKFLOW SELECTION
     - Which template best matches the task type?
     - Has this workflow succeeded on similar tasks? (Experience Graph)
     - Does the project's compliance policy restrict which workflows can run?

  2. EMPLOYEE SELECTION
     - Competency score (Chapter 5)
     - Availability
     - Project familiarity
     - Historical success on this task category

  3. MODEL SELECTION
     - Benchmark Center data: which model scores best for this task type?
     - Budget constraint: does preferred model fit within task.cost_budget?
     - Latency requirement: is there a time_budget?
     - Provider availability: circuit breaker status

  4. TOOL CONFIGURATION
     - Which tools does the selected employee have permission to use?
     - Are there Constitution restrictions (Chapter 10) on tools for this task?
     - Sandbox level required?

  5. EXECUTION STRATEGY
     - Parallel vs sequential stages (based on dependency graph)
     - Simulation required? (Chapter 9) — triggered if risk_score > threshold
     - Approval gates required? (based on task impact score)
```

### 7.3 Decision Record (Immutable)

```sql
CREATE TABLE decisions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id             UUID REFERENCES tasks(id),
    decided_at          TIMESTAMPTZ DEFAULT NOW(),

    -- Inputs
    task_type           TEXT,
    task_description    TEXT,
    project_id          UUID REFERENCES projects(id),
    cost_budget_usd     NUMERIC(10,4),
    time_budget_seconds INTEGER,

    -- Decisions made
    selected_workflow   TEXT,
    selected_employee   UUID REFERENCES employees(id),
    selected_model      TEXT,
    selected_provider   TEXT,
    execution_strategy  JSONB,

    -- Reasoning
    workflow_score      NUMERIC(5,4),
    employee_score      NUMERIC(5,4),
    model_score         NUMERIC(5,4),
    reasoning           JSONB NOT NULL,
    -- {
    --   "workflow": {"winner": "feature-full", "alternatives": [...], "reason": "..."},
    --   "employee": {"winner": "emp-uuid", "score_breakdown": {...}},
    --   "model":    {"winner": "claude-sonnet-4-6", "benchmark_score": 0.87, "cost_fit": true}
    -- }

    -- Policy checks
    constitution_checks JSONB,
    constitution_passed BOOLEAN,

    -- Outcome (filled after task completes)
    decision_was_correct BOOLEAN,
    outcome_task_id     UUID REFERENCES tasks(id)
);
```

### 7.4 Decision Scoring — Model Selection

```python
class ModelSelector:
    async def select(self, task: Task, budget: Decimal) -> ModelSelection:
        task_category = self._categorize(task)

        candidates = await self.benchmark_center.get_ranked_models(
            task_category=task_category,
            min_score=0.70
        )

        for model in candidates:
            cost_estimate = self._estimate_cost(task, model)
            if cost_estimate <= budget:
                latency_estimate = self._estimate_latency(task, model)
                if latency_estimate <= task.time_budget_seconds:
                    availability = await self.provider_gateway.check(model.provider)
                    if availability.healthy:
                        return ModelSelection(
                            model=model,
                            cost_estimate=cost_estimate,
                            reason=f"Best scoring model ({model.score:.2f}) within budget and latency"
                        )

        return self._fallback_selection(candidates, budget)
```

### 7.5 Risk Scoring

```python
def compute_task_risk(task: Task, project: Project) -> RiskScore:
    risk = 0
    if task.affects_production_db:   risk += 30
    if task.deploys_to_production:   risk += 25
    if task.modifies_auth_code:      risk += 20
    if task.estimated_files > 20:    risk += 10
    failure_rate = get_failure_rate(task.type, task.project_id)
    risk += int(failure_rate * 20)
    if task.estimated_cost_usd > 10: risk += 5
    return RiskScore(value=min(risk, 100))

# risk_score >= 70 → SimulationEngine.simulate() before execution
# risk_score >= 50 → approval gate required regardless of workflow config
# risk_score >= 90 → CTO approval required (escalation)
```

### 7.6 API

```
POST   /api/v2/decisions/evaluate    Evaluate options for a task (dry-run, no execution)
GET    /api/v2/decisions/{task_id}   Get decision record for a task
GET    /api/v2/decisions/history     Decision history with outcome correctness
GET    /api/v2/decisions/accuracy    Decision engine accuracy metrics over time
```

---

## Chapter 8 — Knowledge Graph 2.0

### 8.1 Vision

The base document's Knowledge Graph (§11) covers code entities. KG 2.0 extends it to every artifact in the software lifecycle: requirements, deployments, incidents, employees, documents, prompts, models, and tools. The result is a complete, traversable map of the AI software company.

### 8.2 Extended Node Catalog

```cypher
// Code entities (from base KG)
(:Project), (:Module), (:File), (:Class), (:Method)
(:Table), (:Column), (:ApiEndpoint)

// SDLC artifacts (new)
(:Epic {id, title, status, priority})
(:Story {id, title, acceptance_criteria, points})
(:Requirement {id, title, type})
(:TestSuite {id, name, framework})
(:TestCase {id, name, type, file_path})
(:Commit {sha, message, author, timestamp})
(:PullRequest {id, title, status, merged_at})
(:Deployment {id, environment, version, deployed_at, status})
(:Release {id, version, released_at, artifacts})

// Operations (new)
(:Incident {id, severity, status, started_at, resolved_at})
(:AlertRule {id, metric, threshold, action})
(:Dashboard {id, name, metrics})

// Organization (new — integrates Chapter 4 & 5)
(:Employee {id, name, role})
(:OrgRole {name, level})

// AI artifacts (new)
(:AIModel {id, provider, name, version})
(:Prompt {id, agent_type, prompt_key})
(:PromptVersion {id, version, sha256})
(:Tool {id, name, version})
(:Workflow {id, slug, version})
(:Plugin {id, name, version, type})

// Documents (new)
(:Document {id, title, type, path})
-- type: 'adr'|'readme'|'runbook'|'api_spec'|'post_mortem'
```

### 8.3 Extended Relationship Catalog

```cypher
// Requirements → Code
(:Story)-[:IMPLEMENTED_BY]->(:Task)
(:Task)-[:MODIFIES]->(:File)
(:Task)-[:CREATES]->(:ApiEndpoint)
(:Requirement)-[:REQUIRES]->(:Table)

// Code → Tests
(:TestCase)-[:TESTS]->(:Method)
(:TestCase)-[:TESTS]->(:ApiEndpoint)
(:TestSuite)-[:COVERS]->(:Module)

// Code → Deployments
(:Commit)-[:DEPLOYED_IN]->(:Deployment)
(:Release)-[:CONTAINS]->(:Commit)
(:Deployment)-[:TO_ENV]->(:Environment)

// Incidents
(:Incident)-[:CAUSED_BY]->(:Commit)
(:Incident)-[:AFFECTED]->(:ApiEndpoint)
(:Incident)-[:RESOLVED_BY]->(:Commit)
(:AlertRule)-[:TRIGGERED]->(:Incident)
(:Incident)-[:DOCUMENTED_IN]->(:Document)

// AI artifacts
(:Employee)-[:USES]->(:AIModel)
(:Employee)-[:EXECUTES]->(:Task)
(:Prompt)-[:USED_IN]->(:Agent_Execution)
(:PromptVersion)-[:PRODUCED]->(:TaskOutcome)
(:Workflow)-[:USES]->(:Tool)
(:Workflow)-[:SPAWNS]->(:Task)

// Knowledge
(:Decision)-[:DOCUMENTED_IN]->(:Document)
(:Document)-[:REFERENCES]->(:Module)
(:Lesson)-[:APPLIES_TO]->(:Module)
(:Lesson)-[:DERIVES_FROM]->(:Incident)
```

### 8.4 Cross-Artifact Query Examples

```cypher
// "Which incidents were caused by commits in the payment module?"
MATCH (i:Incident)-[:CAUSED_BY]->(c:Commit)-[:MODIFIES]->(f:File)
      -[:BELONGS_TO]->(m:Module {name: 'payment'})
RETURN i.id, i.severity, c.sha, c.message, i.started_at
ORDER BY i.severity DESC

// "What tests cover the methods changed by this PR?"
MATCH (pr:PullRequest {id: $pr_id})-[:CONTAINS]->(c:Commit)
      -[:MODIFIES]->(f:File)<-[:DEFINED_IN]-(method:Method)
      <-[:TESTS]-(tc:TestCase)
RETURN DISTINCT tc.name, tc.file_path, method.name

// "Which employees have contributed most to modules with high incident rate?"
MATCH (e:Employee)-[:EXECUTES]->(t:Task)-[:MODIFIES]->(f:File)
      -[:BELONGS_TO]->(m:Module)
      <-[:AFFECTED]-(i:Incident)
WHERE i.started_at > datetime() - duration('P90D')
RETURN e.name, m.name, COUNT(i) AS incident_count
ORDER BY incident_count DESC

// "Which prompts are used in tasks that touch security-sensitive code?"
MATCH (ae:AgentExecution)-[:USED_PROMPT]->(pv:PromptVersion)
      <-[:HAS_VERSION]-(p:Prompt)
WHERE (ae)-[:MODIFIES_FILE]->(:File)-[:TAGGED]->(t:Tag {name:'security'})
RETURN p.agent_type, p.prompt_key, pv.version, COUNT(*) AS usage_count
```

### 8.5 Ingestion Sources

| Source | Ingested By | Frequency |
|--------|-----------|-----------|
| Repository (AST) | KG Ingestion Worker | On every commit |
| GitHub/GitLab API | KG Ingestion Worker | Hourly |
| Deployment events | DevOpsAgent | On deploy |
| Incident alerts | MonitoringAgent | On alert fire |
| Employee actions | Employee System | On task completion |
| Prompt deployments | Prompt Platform | On deploy |
| Documents (README, ADR) | DocumentationAgent | On file write |

### 8.6 KG Health Monitoring

```python
class KGHealthMonitor:
    async def check(self):
        stale_files = await self.kg.query("""
            MATCH (f:File) WHERE NOT f.path IN $disk_paths RETURN f
        """, disk_paths=self.repo.list_files())

        untested = await self.kg.query("""
            MATCH (m:Method) WHERE NOT (m)<-[:TESTS]-(:TestCase)
            AND m.is_public = true RETURN m
        """)

        self.metrics.set('kg_stale_nodes', len(stale_files))
        self.metrics.set('kg_untested_public_methods', len(untested))
```

---

## Chapter 9 — Simulation Engine

### 9.1 Vision

Before executing any high-risk action, the platform runs it in a digital simulation: a shadow environment that mimics production state without touching production. The simulation produces a risk score, predicted impact analysis, and a rollback plan — all before a human sees the approval gate.

This eliminates the "I didn't know this would break production" class of failures.

### 9.2 Risk Trigger Thresholds

| Risk Score | Action |
|-----------|--------|
| 0–49 | Execute directly (no simulation) |
| 50–69 | Simulation optional; approval gate required |
| 70–89 | Simulation **required** before approval gate |
| 90–100 | Simulation required + CTO-level approval required |

### 9.3 Simulation Types

| Simulation Type | Target Actions | Environment |
|----------------|---------------|------------|
| `database_migration` | ALTER TABLE, DROP COLUMN, data backfill | Cloned DB (pg_dump restore to temp DB) |
| `mass_delete` | DELETE WHERE, bulk UPDATE | Cloned DB |
| `deployment` | Container deploy, K8s rollout | Staging namespace |
| `prompt_replacement` | New prompt version deployed | Shadow agent pool (5% of tasks) |
| `model_replacement` | Switch AI provider/model | Benchmark tasks on new model |
| `infrastructure_change` | Terraform apply, K8s config | Plan-only or staging cluster |
| `security_patch` | Dependency upgrade, auth change | Staging + security scan |

### 9.4 Database Migration Simulation

```
1. CLONE
   pg_dump production_db | pg_restore → simulation_db_<task_id>
   Duration: proportional to DB size (~30s for 10GB)

2. APPLY
   Run migration script against simulation_db
   Capture: execution time, rows affected, any errors

3. ANALYZE
   - Row count delta per table
   - Index rebuild required?
   - Lock contention detected (pg_locks analysis)?
   - Estimated downtime if run on production

4. ROLLBACK PLAN
   Generate reverse migration (DROP column / restore from backup)
   Verify reverse migration applies cleanly

5. REPORT
   {
     "risk_score": 35,
     "duration_seconds": 12.4,
     "rows_affected": {"orders": 0, "payments": 142857},
     "downtime_estimate_seconds": 14,
     "locks_detected": false,
     "rollback_plan": "DROP COLUMN payment_method_v2 FROM payments",
     "recommendation": "SAFE — proceed with approval"
   }
```

### 9.5 Deployment Simulation (K8s)

```
1. DRY-RUN
   kubectl apply --dry-run=server -f deployment.yaml
   Capture: validation errors, resource conflicts

2. STAGING DEPLOY
   Deploy to staging/<project>-sim namespace
   Run smoke tests (existing test suite, subset)

3. HEALTH CHECK
   Wait for deployment rollout: kubectl rollout status
   Check pod readiness, container logs for exceptions

4. LOAD TEST (optional, risk_score >= 80)
   Run k6 script (5% production load profile) against staging
   Capture: latency p99, error rate

5. REPORT + ROLLBACK PLAN
   "kubectl rollout undo deployment/<name> -n staging"
   Estimated production rollout time, confidence score
```

### 9.6 Database Schema

```sql
CREATE TABLE simulations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id         UUID REFERENCES tasks(id),
    proposal_id     UUID REFERENCES improvement_proposals(id),
    sim_type        TEXT NOT NULL,
    status          TEXT DEFAULT 'pending',
    risk_score_before INTEGER,
    risk_score_after  INTEGER,
    environment     TEXT,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    duration_seconds NUMERIC(10,3),
    result          JSONB,
    recommendation  TEXT,           -- 'SAFE'|'CAUTION'|'HIGH_RISK'|'ABORT'
    rollback_plan   TEXT,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE simulation_steps (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    simulation_id   UUID REFERENCES simulations(id) ON DELETE CASCADE,
    step_name       TEXT NOT NULL,
    status          TEXT,
    output          JSONB,
    error           TEXT,
    duration_ms     INTEGER,
    executed_at     TIMESTAMPTZ DEFAULT NOW()
);
```

### 9.7 Simulation API

```
POST   /api/v2/simulations                    Start a simulation for a task
GET    /api/v2/simulations/{id}               Get simulation status + result
GET    /api/v2/simulations/{id}/steps         Step-by-step execution log
DELETE /api/v2/simulations/{id}/environment   Clean up simulation environment
POST   /api/v2/simulations/estimate           Estimate simulation duration before running
```

### 9.8 Desktop Integration

The Approval Gate dialog is extended: when a simulation has run, it shows risk score, simulation results, and rollback plan inline before the approve/reject buttons.

```
┌──────────────────────────────────────────────────────────────┐
│  ⚠ APPROVAL REQUIRED                              [×]        │
├──────────────────────────────────────────────────────────────┤
│  Action: Run database migration — add payments.stripe_id     │
│  Risk Score: 42/100  ████░░░░░░  MODERATE                   │
├──────────────────────────────────────────────────────────────┤
│  SIMULATION RESULTS                       ✓ Simulation ran   │
│  Duration on prod estimate:  14 seconds                      │
│  Rows affected:              142,857                         │
│  Lock contention:            None detected                   │
│  Downtime required:          No                              │
│  Rollback plan:              ALTER TABLE DROP COLUMN [ready] │
│  Recommendation:             SAFE — no issues found          │
├──────────────────────────────────────────────────────────────┤
│           [✗ Reject]    [→ Request Changes]   [✓ Approve]   │
└──────────────────────────────────────────────────────────────┘
```

---

## Chapter 10 — AI Constitution

### 10.1 Vision

The AI Constitution is a policy engine that defines immutable rules governing all agent behavior. No agent, no matter how capable or how confidently it acts, can violate a Constitutional rule. The Constitution is not a suggestion — it is an enforcement layer that intercepts tool calls, task claims, and workflow transitions before they execute.

### 10.2 Constitutional Rules (Built-In, Non-Deletable)

```yaml
# constitution/core.yaml — system rules, cannot be overridden by any user

rules:
  - id: CONST-001
    name: never_commit_failing_tests
    description: "No commit may proceed if any test in the project test suite fails."
    enforcement: BLOCK
    applies_to: [git_ops.commit, git_ops.push]
    condition: "test_result.passed < test_result.total"
    exception_path: emergency_override

  - id: CONST-002
    name: never_deploy_without_approval
    description: "No deployment to production may proceed without at least one human approval."
    enforcement: BLOCK
    applies_to: [deployment_engine.deploy]
    condition: "environment == 'production' AND approval_count == 0"

  - id: CONST-003
    name: never_delete_production_data
    description: "No DELETE or TRUNCATE on production database without simulation + dual approval."
    enforcement: BLOCK
    applies_to: [db_client.execute]
    condition: "sql MATCHES '^(DELETE|TRUNCATE)' AND environment == 'production' AND NOT simulation_verified"

  - id: CONST-004
    name: never_exceed_token_budget
    description: "No agent execution may continue after exceeding the task's token budget."
    enforcement: BLOCK
    applies_to: [ai_provider_gateway.complete]
    condition: "tokens_used > task.token_budget"

  - id: CONST-005
    name: never_expose_secrets
    description: "No file write, commit, or log may contain patterns matching secret patterns."
    enforcement: BLOCK
    applies_to: [file_writer.write, git_ops.commit, tool_calls.log]
    condition: "content MATCHES secret_patterns"
    secret_patterns: ['(sk-[a-zA-Z0-9]{48})', '(AKIA[0-9A-Z]{16})', '(?i)(password|secret|api_key)\\s*=\\s*["\'][^"\']{8,}']

  - id: CONST-006
    name: never_bypass_security_scan
    description: "No release may proceed if security scan has not run or has critical findings."
    enforcement: BLOCK
    applies_to: [workflow_engine.advance_stage]
    condition: "stage_key == 'release' AND security_scan_status NOT IN ('passed', 'waived_by_security_lead')"

  - id: CONST-007
    name: never_write_to_main_without_review
    description: "Direct commits to main/master require at least one reviewer approval."
    enforcement: BLOCK
    applies_to: [git_ops.commit, git_ops.push]
    condition: "branch IN ('main', 'master') AND reviewer_approvals == 0"

  - id: CONST-008
    name: sandbox_required_for_shell
    description: "Shell execution must always use sandboxed environment."
    enforcement: BLOCK
    applies_to: [shell_executor.execute]
    condition: "NOT context.sandboxed"
```

### 10.3 Project-Level Constitutional Rules

Projects can add additional rules (but cannot remove or weaken core rules):

```yaml
# constitution/project-my-saas-app.yaml

rules:
  - id: PROJ-001
    name: no_frontend_agent_touching_payments
    description: "Frontend agents may not modify any file in services/payment/"
    enforcement: BLOCK
    applies_to: [file_writer.write]
    condition: "agent_type == 'FrontendAgent' AND file_path STARTS WITH 'services/payment/'"

  - id: PROJ-002
    name: pii_data_stays_in_eu
    description: "Any deployment targeting non-EU regions requires compliance officer approval."
    enforcement: BLOCK
    applies_to: [deployment_engine.deploy]
    condition: "region NOT IN ['eu-west-1', 'eu-central-1']"
```

### 10.4 Policy Decision Point (PDP) Architecture

```
Agent calls tool X
       │
       ▼
Tool Runtime intercepts call
       │
       ▼
PolicyDecisionPoint.evaluate(agent_type, tool_name, tool_input, context)
       │
       ├── Load applicable rules: core + project-level
       ├── Evaluate each condition (Python AST evaluator, sandboxed)
       ├── First BLOCK match → deny immediately
       ├── All WARN matches → allow + log warnings
       └── No matches → allow
       │
       ▼
Decision: ALLOW | BLOCK | ALLOW_WITH_WARNINGS
       │
       ├── BLOCK → raise ConstitutionViolation
       │           → log to constitution_violations table
       │           → emit constitution.violated event
       │           → optionally: create escalation gate for emergency override
       │
       └── ALLOW → execute tool, log to tool_calls
```

### 10.5 Database Schema

```sql
CREATE TABLE constitution_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rule_id         TEXT UNIQUE NOT NULL,
    scope           TEXT NOT NULL,              -- 'core'|'project'|'org'
    project_id      UUID REFERENCES projects(id),
    name            TEXT NOT NULL,
    description     TEXT NOT NULL,
    enforcement     TEXT NOT NULL,              -- 'BLOCK'|'WARN'|'LOG'
    applies_to      TEXT[],
    condition       TEXT NOT NULL,
    exception_path  TEXT,
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    created_by      UUID REFERENCES users(id)
);

CREATE TABLE constitution_violations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rule_id         TEXT NOT NULL,
    task_id         UUID REFERENCES tasks(id),
    employee_id     UUID REFERENCES employees(id),
    agent_type      TEXT,
    tool_name       TEXT,
    tool_input      JSONB,
    violation_detail TEXT,
    enforcement_action TEXT,
    override_gate_id UUID REFERENCES approval_gates(id),
    occurred_at     TIMESTAMPTZ DEFAULT NOW()
);
-- APPEND-ONLY — no UPDATE or DELETE (enforced via trigger + RLS)

CREATE TABLE constitution_overrides (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    violation_id    UUID REFERENCES constitution_violations(id),
    rule_id         TEXT NOT NULL,
    approved_by     UUID[] NOT NULL,            -- dual approval: min 2 approvers for core rules
    justification   TEXT NOT NULL,
    expires_at      TIMESTAMPTZ,
    executed_at     TIMESTAMPTZ DEFAULT NOW()
);
```

### 10.6 Emergency Override Protocol

For `CONST-001` through `CONST-008`, an override requires:
1. Written justification (minimum 100 characters)
2. Dual approval (CTO + CEO, or any two users with `role=admin`)
3. Override is time-limited (max 4 hours)
4. Override is logged immutably in `constitution_overrides`
5. Override triggers an automatic Post-Mortem task assigned to ArchitectAgent

### 10.7 API

```
GET    /api/v2/constitution/rules              List all rules (core + project)
POST   /api/v2/constitution/rules              Create project-level rule
PATCH  /api/v2/constitution/rules/{id}         Update rule (project-level only; core: read-only)
GET    /api/v2/constitution/violations         List violations (audit view)
POST   /api/v2/constitution/violations/{id}/override   Request emergency override
GET    /api/v2/constitution/evaluate           Dry-run: evaluate a tool call against rules
```

### 10.8 Scalability

The PDP is the hottest code path in the system — it intercepts every tool call:
- Rules compiled to bytecode at startup (not re-parsed per call)
- In-memory rule cache (Redis-backed, TTL 60s)
- PDP evaluation: P99 < 5ms
- Async non-blocking (never holds up the tool call for more than 10ms)
- Horizontally scalable (stateless, reads from Redis cache)

---

## Chapter 11 — Governance & Audit

### 11.1 Vision

Every action taken by every agent, tool, or human in the platform generates an immutable audit event. The Governance & Audit system is the compliance backbone — it answers "who did what, when, why, at what cost, using which model and prompt version" for every event in the system's history.

This is the foundation for SOC 2, ISO 27001, HIPAA, GDPR, and enterprise procurement requirements.

### 11.2 Audit Event Schema

```sql
CREATE TABLE audit_events (
    id              BIGSERIAL,
    event_id        UUID DEFAULT gen_random_uuid(),
    project_id      UUID REFERENCES projects(id),
    tenant_id       UUID,

    -- WHO
    actor_type      TEXT NOT NULL,              -- 'agent'|'human'|'system'|'plugin'
    actor_id        TEXT NOT NULL,
    actor_role      TEXT,
    ip_address      INET,
    session_id      UUID,

    -- WHAT
    action          TEXT NOT NULL,
    resource_type   TEXT,
    resource_id     TEXT,
    input_summary   TEXT,                       -- truncated (first 500 chars)
    output_summary  TEXT,

    -- WHEN
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- WHY
    task_id         UUID REFERENCES tasks(id),
    workflow_id     UUID REFERENCES workflows(id),
    decision_id     UUID REFERENCES decisions(id),
    justification   TEXT,

    -- AI PROVENANCE
    model           TEXT,
    provider        TEXT,
    prompt_id       UUID REFERENCES prompts(id),
    prompt_version  TEXT,
    input_tokens    INTEGER,
    output_tokens   INTEGER,

    -- COST
    cost_usd        NUMERIC(10,6),

    -- OUTCOME
    success         BOOLEAN,
    error           TEXT,
    duration_ms     INTEGER,

    -- INTEGRITY
    previous_hash   TEXT,                       -- hash of previous row (chain integrity)
    row_hash        TEXT NOT NULL,
    signed_by       TEXT,                       -- KMS key ID (enterprise)

    PRIMARY KEY (id, occurred_at)
) PARTITION BY RANGE (occurred_at);

CREATE RULE no_update_audit AS ON UPDATE TO audit_events DO INSTEAD NOTHING;
CREATE RULE no_delete_audit AS ON DELETE TO audit_events DO INSTEAD NOTHING;
```

### 11.3 Cryptographic Chain Integrity

Each audit event includes `previous_hash` forming a hash chain. Tampering with any past row breaks all subsequent hashes — detectable in seconds:

```python
class AuditChainVerifier:
    async def verify(self, from_id: int, to_id: int) -> VerificationResult:
        events = await self.db.fetch(
            "SELECT * FROM audit_events WHERE id BETWEEN $1 AND $2 ORDER BY id",
            from_id, to_id
        )
        broken_at = None
        for i, event in enumerate(events[1:], 1):
            expected_hash = sha256(serialize(events[i-1]))
            if event.previous_hash != expected_hash:
                broken_at = event.id
                break
        return VerificationResult(intact=(broken_at is None), broken_at=broken_at)
```

### 11.4 Compliance Profiles

| Profile | Requirements Met | Configuration |
|---------|----------------|--------------|
| **SOC 2 Type II** | Audit log retention (1 year), access control logs, change management | Default enterprise mode |
| **ISO 27001** | Asset inventory (KG), risk assessment, incident log, change log | + incident workflow + risk scoring |
| **HIPAA** | Data residency, PHI access logs, audit trails with minimum 6-year retention | + tenant isolation + geo-lock |
| **GDPR** | Data subject audit trail, right-to-erasure log, consent tracking | + PII detection + erasure workflow |
| **CCPA** | Consumer data access log, opt-out tracking | Subset of GDPR config |

### 11.5 Audit API

```
GET    /api/v2/audit/events                  Query audit events (filter: actor, action, date range)
GET    /api/v2/audit/events/{id}             Single event detail
GET    /api/v2/audit/chain/verify            Verify chain integrity for a date range
GET    /api/v2/audit/export                  Export to JSON/CSV/CEF for SIEM
GET    /api/v2/audit/reports/compliance      Compliance summary report (SOC2, ISO27001)
GET    /api/v2/audit/reports/cost            Cost audit by actor, project, model
GET    /api/v2/audit/reports/access          Access audit by user, resource
POST   /api/v2/audit/retention/configure     Set retention policy (enterprise)
```

### 11.6 SIEM Integration

```
Audit events → Fluent Bit (sidecar) → SIEM
                                        ├── Splunk (via HEC)
                                        ├── Elastic SIEM (via Logstash)
                                        ├── Microsoft Sentinel (via Log Analytics)
                                        └── Sumo Logic (via HTTP Source)

CEF Format example:
CEF:0|AIStudio|AuditLog|2.0|tool_called|Agent executed shell command|5|
  src=10.0.0.5 suser=Alex-Backend-7 act=shell_executor.execute
  msg=pytest tests/ outcome=success dvc=LAPTOP-TM8BC7MD
  cs1=claude-sonnet-4-6 cs1Label=model cs2=1.4.2 cs2Label=promptVersion
```

---

## Chapter 12 — Security Platform

### 12.1 Vision

Security is not a feature added at the end — it is a cross-cutting enforcement layer present in every service boundary. The Security Platform implements Zero Trust: every service-to-service call is authenticated, every agent action is authorized, every tool input is validated, and every secret is vaulted.

### 12.2 Zero Trust Architecture

```
Principle: Never trust. Always verify. Assume breach.

┌──────────────────────────────────────────────────────────────────────┐
│  All traffic encrypted: mTLS between all internal services           │
│  All identities verified: OIDC JWT or service account certificates   │
│  All access decisions logged: audit_events table                     │
│  All secrets vaulted: never in env vars, config files, or DB plain   │
│  All inputs validated: schema validation before tool execution        │
│  All outputs scanned: secret detection before any write/log          │
└──────────────────────────────────────────────────────────────────────┘
```

### 12.3 RBAC + ABAC Model

```
RBAC (Role-Based): What role can do
  Role: 'engineer'  → can: write_file, run_tests, commit_feature_branch
  Role: 'lead'      → can: approve_pr, deploy_staging, manage_team
  Role: 'admin'     → can: all above + manage_constitution, manage_users

ABAC (Attribute-Based): Context-specific access
  Allow deploy to production IF:
    user.role IN ['cto', 'admin']
    AND approval_gate.count >= 2
    AND simulation.recommendation == 'SAFE'
    AND time_of_day BETWEEN '08:00' AND '20:00'
    AND NOT incident.active

Policy evaluation order: ABAC > RBAC > Constitution
Constitution always has final veto.
```

### 12.4 Secret Management

```
Secret Lifecycle:
  1. Developer adds secret → Vault UI or CLI
  2. Vault stores encrypted (AES-256-GCM, envelope encryption with KMS)
  3. Agent requests secret → Vault issues short-lived lease (max 1h)
  4. Agent uses secret in-memory only (never written to disk, logs, or DB)
  5. Constitution rule CONST-005 scans all outputs for secret patterns
  6. Lease expires → secret auto-revoked

Vault Integration:
  - HashiCorp Vault (enterprise) or Infisical (open source)
  - Local dev mode: encrypted .aistudio/secrets.enc (master key in OS keychain)
  - Kubernetes: External Secrets Operator → Vault → K8s Secret (ephemeral)
```

### 12.5 Tenant Isolation (Multi-Tenant SaaS)

```
Isolation levels:
  L1 — Row-Level Security (RLS):       all tables have tenant_id + RLS policy  [default]
  L2 — Schema isolation:               one PostgreSQL schema per tenant         [enterprise]
  L3 — Database isolation:             one PostgreSQL instance per tenant       [regulated]
  L4 — Cluster isolation:              dedicated K8s namespace per tenant       [critical]
```

### 12.6 Database Schema

```sql
CREATE TABLE permission_grants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    grantee_type    TEXT NOT NULL,          -- 'user'|'role'|'employee'
    grantee_id      TEXT NOT NULL,
    resource_type   TEXT NOT NULL,
    resource_id     TEXT,
    action          TEXT NOT NULL,
    conditions      JSONB DEFAULT '{}',
    granted_by      UUID REFERENCES users(id),
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE api_tokens (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID REFERENCES users(id),
    name            TEXT NOT NULL,
    token_hash      TEXT UNIQUE NOT NULL,
    scopes          TEXT[] NOT NULL,
    last_used_at    TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    revoked_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE security_scan_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id         UUID REFERENCES tasks(id),
    scan_type       TEXT NOT NULL,          -- 'sast'|'sca'|'secret'|'container'|'iac'
    tool            TEXT NOT NULL,
    tool_version    TEXT,
    status          TEXT NOT NULL,          -- 'pass'|'fail'|'warn'
    critical_count  INTEGER DEFAULT 0,
    high_count      INTEGER DEFAULT 0,
    medium_count    INTEGER DEFAULT 0,
    low_count       INTEGER DEFAULT 0,
    findings        JSONB DEFAULT '[]',
    report_path     TEXT,
    scanned_at      TIMESTAMPTZ DEFAULT NOW()
);
```

### 12.7 Security Scanning Integration

| Scanner | Triggers | Blocks Release If |
|---------|---------|------------------|
| `semgrep` (SAST) | Every commit via SecurityAgent | critical_count > 0 |
| `pip-audit` / `OWASP DC` | Every dependency change | CVSS ≥ 9 |
| `truffleHog` | Every commit (pre-commit hook + CI) | Any secret found |
| `trivy` | Every container build | critical_count > 0 |
| `checkov` (IaC) | Every Terraform/K8s manifest change | HIGH severity findings |

---

## Chapter 13 — AI Economy Engine

### 13.1 Vision

Every compute action has a cost. The AI Economy Engine makes those costs visible, attributable, and optimizable. Projects have budgets. Departments have allocations. Individual tasks have cost limits. The platform continuously identifies cheaper alternatives that achieve equivalent quality.

### 13.2 Cost Attribution Model

```
Organization
└── Department (Engineering, Security, DevOps...)
    └── Project
        └── Workflow Execution
            └── Task
                └── Agent Execution
                    ├── Model inference cost (tokens × price)
                    ├── Tool compute cost (build time, CI minutes)
                    └── Infrastructure cost (agent worker time)
```

### 13.3 Database Schema

```sql
CREATE TABLE budgets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scope_type      TEXT NOT NULL,          -- 'org'|'department'|'project'|'employee'
    scope_id        TEXT NOT NULL,
    period          TEXT NOT NULL,          -- '2026-06' or '2026-Q2'
    budget_usd      NUMERIC(12,4) NOT NULL,
    alert_threshold NUMERIC(5,2) DEFAULT 0.80,
    hard_limit      BOOLEAN DEFAULT FALSE,
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(scope_type, scope_id, period)
);

CREATE TABLE cost_ledger (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id         UUID REFERENCES tasks(id),
    execution_id    UUID REFERENCES agent_executions(id),
    project_id      UUID REFERENCES projects(id),
    employee_id     UUID REFERENCES employees(id),
    department      TEXT,
    cost_type       TEXT NOT NULL,          -- 'model_inference'|'tool_compute'|'infra'
    provider        TEXT,
    model           TEXT,
    units           NUMERIC(12,4),
    unit_price_usd  NUMERIC(12,8),
    total_usd       NUMERIC(10,6) NOT NULL,
    recorded_at     TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_ledger_project ON cost_ledger(project_id, recorded_at);
CREATE INDEX idx_ledger_department ON cost_ledger(department, recorded_at);

CREATE TABLE roi_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID REFERENCES projects(id),
    period          TEXT NOT NULL,
    ai_cost_usd     NUMERIC(12,4),
    estimated_human_hours NUMERIC(10,2),
    human_cost_usd  NUMERIC(12,4),
    roi_ratio       NUMERIC(8,4),
    tasks_completed INTEGER,
    lines_written   INTEGER,
    tests_generated INTEGER,
    bugs_fixed      INTEGER,
    recorded_at     TIMESTAMPTZ DEFAULT NOW()
);
```

### 13.4 Cost Optimization Engine

```python
class CostOptimizer:
    async def analyze(self, project_id: UUID, period: str) -> list[CostProposal]:
        proposals = []

        overpriced = await self.db.fetch("""
            SELECT ae.model, t.type, AVG(ae.cost_usd) as avg_cost,
                   AVG(CASE WHEN to2.status='completed' THEN 1.0 ELSE 0.0 END) as success_rate
            FROM agent_executions ae
            JOIN tasks t ON ae.task_id = t.id
            JOIN task_outcomes to2 ON t.id = to2.task_id
            WHERE t.project_id = $1
            GROUP BY ae.model, t.type
            HAVING AVG(ae.cost_usd) > 0.10
        """, project_id)

        for row in overpriced:
            cheaper = await self.benchmark_center.find_cheaper_equivalent(
                task_type=row.type,
                current_model=row.model,
                min_success_rate=row.success_rate * 0.97
            )
            if cheaper and cheaper.avg_cost < row.avg_cost * 0.6:
                proposals.append(CostProposal(
                    type='model_swap',
                    task_type=row.type,
                    current_model=row.model,
                    proposed_model=cheaper.model,
                    estimated_monthly_savings_usd=self._project_savings(row, cheaper),
                    confidence=cheaper.benchmark_sample_count / 100
                ))

        return proposals
```

### 13.5 Budget Enforcement

```
Before task starts:
  1. Estimate cost (Decision Engine §7.4)
  2. Check: project budget remaining >= estimated cost
     → If not: BLOCK with BudgetExceeded; notify PM
  3. Check: task.cost_budget >= estimated cost
     → If not: WARN + create approval gate

During task:
  Streaming token counter → if tokens_used > task.token_budget × 0.9:
     emit budget.warning event → Desktop notification
  If tokens_used > task.token_budget:
     CONST-004 triggers → execution halted → partial result saved

After task:
  Record actual cost to cost_ledger
  If month total > alert_threshold: notify budget owner
  If hard_limit AND month total > budget: block all new tasks in scope
```

### 13.6 Economy Dashboard (Desktop)

```
┌────────────────────────────────────────────────────────────────────┐
│  AI ECONOMY — June 2026  [Project: All ▼]  [Export]               │
├─────────────────────────────────────────────────────────────────────┤
│  SPENDING                          ROI                              │
│  Month total:     $142.83          Human hours saved: 847h          │
│  vs budget:       71.4% ████████▒░ Human cost equiv: $42,350       │
│  Daily avg:       $4.76            ROI ratio:         296×          │
│  Forecast EOM:    $152 (+7.2%)                                      │
├─────────────────────────────────────────────────────────────────────┤
│  BY MODEL                          BY AGENT                         │
│  claude-sonnet: $89.21 (62%)       BackendAgent: $58.33 (41%)      │
│  claude-opus:   $31.44 (22%)       QAAgent:      $29.17 (20%)      │
│  ollama/qwen:   $0.00 (local)      ArchAgent:    $22.19 (16%)      │
├─────────────────────────────────────────────────────────────────────┤
│  OPTIMIZATION PROPOSALS                                             │
│  💡 Switch QA doc generation to claude-haiku → save $8.40/month    │
│  💡 Use local Ollama for research tasks → save $12.20/month        │
│                                               [Apply All]          │
└────────────────────────────────────────────────────────────────────┘
```

---

## Chapter 14 — Benchmark Center

### 14.1 Vision

The platform continuously runs controlled benchmarks on every AI model it has access to, using its own task suite as the evaluation corpus. This produces an internal leaderboard that reflects how each model actually performs on *this organization's work* — not on academic benchmarks.

The Decision Engine (Chapter 7) consumes this data in real time for model selection.

### 14.2 Benchmark Suite Structure

```
benchmark-suite/
├── code_generation/
│   ├── python_api_endpoint.yaml
│   ├── sql_migration.yaml
│   ├── react_component.yaml
│   └── fix_failing_test.yaml
├── code_review/
│   ├── security_vulnerability.yaml
│   ├── logic_error.yaml
│   └── performance_issue.yaml
├── planning/
│   ├── decompose_feature.yaml
│   └── estimate_complexity.yaml
├── documentation/
│   ├── generate_readme.yaml
│   └── generate_changelog.yaml
└── reasoning/
    ├── architecture_tradeoff.yaml
    └── debug_stack_trace.yaml
```

### 14.3 Database Schema

```sql
CREATE TABLE benchmark_tasks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    category        TEXT NOT NULL,
    name            TEXT NOT NULL,
    description     TEXT,
    input           JSONB NOT NULL,
    reference_answer TEXT,
    evaluation_rubric JSONB,
    expected_tokens INTEGER,
    difficulty      INTEGER DEFAULT 3,      -- 1=easy, 3=medium, 5=hard
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(category, name)
);

CREATE TABLE benchmark_runs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model           TEXT NOT NULL,
    provider        TEXT NOT NULL,
    model_version   TEXT,
    run_date        DATE NOT NULL,
    triggered_by    TEXT DEFAULT 'scheduled',
    status          TEXT DEFAULT 'running',
    total_tasks     INTEGER,
    completed_tasks INTEGER DEFAULT 0,
    started_at      TIMESTAMPTZ DEFAULT NOW(),
    completed_at    TIMESTAMPTZ
);

CREATE TABLE benchmark_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    run_id          UUID REFERENCES benchmark_runs(id) ON DELETE CASCADE,
    task_id         UUID REFERENCES benchmark_tasks(id),
    model           TEXT NOT NULL,
    output          TEXT,
    accuracy_score  NUMERIC(5,4),
    latency_ms      INTEGER,
    input_tokens    INTEGER,
    output_tokens   INTEGER,
    cost_usd        NUMERIC(10,6),
    hallucination_detected BOOLEAN DEFAULT FALSE,
    error           TEXT,
    scored_at       TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE model_leaderboard (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model           TEXT NOT NULL,
    provider        TEXT NOT NULL,
    category        TEXT NOT NULL,
    avg_accuracy    NUMERIC(5,4),
    avg_latency_ms  INTEGER,
    avg_cost_usd    NUMERIC(10,6),
    hallucination_rate NUMERIC(5,4),
    pass_rate       NUMERIC(5,4),
    sample_count    INTEGER,
    last_updated    TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(model, provider, category)
);
```

### 14.4 Scoring Methodology

```
Accuracy Score (per task):
  IF task has reference_answer:
    exact_match_score = (1.0 if output == reference else 0.0)
    semantic_similarity = cosine_sim(embed(output), embed(reference))
    accuracy = 0.4 × exact_match + 0.6 × semantic_similarity

  IF task uses LLM-as-judge:
    score = judge_model.rate(output, rubric) / 10.0
    accuracy = score

Hallucination Detection:
  - Code tasks: compile + run tests → hallucination if syntax error
  - Factual tasks: cross-reference against KG → flag unsupported claims

Composite Score (for leaderboard):
  composite = (0.50 × accuracy)
            + (0.20 × cost_efficiency)    -- min/max ratio
            + (0.20 × latency_score)      -- inverted, lower=better
            + (0.10 × (1 - hallucination_rate))
```

### 14.5 Automated Scheduling

```
Full benchmark suite:  Sunday 02:00 UTC (weekly)
Category benchmarks:   On model_added, model_version_changed
Targeted re-benchmark: After model config change in a project

On complete:
  → update model_leaderboard
  → emit benchmark.completed event
  → Decision Engine cache invalidated
  → notify: "claude-opus-4-8 ranked #1 for planning tasks (score: 0.94)"
```

### 14.6 API

```
GET    /api/v2/benchmarks/leaderboard           Full leaderboard (all models, all categories)
GET    /api/v2/benchmarks/leaderboard/{category} Category-specific leaderboard
POST   /api/v2/benchmarks/run                   Trigger benchmark run for a model
GET    /api/v2/benchmarks/runs/{id}             Run status + progress
GET    /api/v2/benchmarks/compare               Compare two models head-to-head
GET    /api/v2/benchmarks/recommend/{task_type} Best model recommendation for task type
```

---

## Chapter 15 — Evolution Lab

### 15.1 Vision

The Evolution Lab is the safe experimentation environment for every change to the platform's intelligence: new prompts, new workflows, new models, new agent configurations, and new tools. Nothing in the Evolution Lab touches production. Every experiment must pass simulation, benchmark regression, and approval before production rollout.

### 15.2 Lab Architecture

```
PRODUCTION ENVIRONMENT
┌────────────────────────────────────┐
│  Coordinator (prod)                │
│  Agent Pool (prod)                 │    ← Constitution enforcement active
│  Tool Runtime (prod, sandboxed)    │    ← Full audit logging
│  All data: production PostgreSQL   │
└────────────────────────────────────┘
         ↑
         │  One-way promotion gate (requires approval + passing gates)
         │  Lab NEVER writes to prod DB directly
         │
EVOLUTION LAB ENVIRONMENT
┌────────────────────────────────────┐
│  Lab Coordinator (isolated)        │
│  Lab Agent Pool (experiment builds)│    ← Shadow mode only
│  Lab Tool Runtime (isolated net)   │
│  Lab PostgreSQL (prod snapshot)    │    ← Refreshed nightly
│  Lab Qdrant (prod clone)           │
└────────────────────────────────────┘
```

### 15.3 Experiment Types

| Type | What Changes | Success Criteria | Auto-Promote? |
|------|-------------|-----------------|--------------|
| `prompt_experiment` | Prompt version A/B | improvement accuracy +5% | With approval |
| `workflow_experiment` | New workflow template | stage_success_rate ≥ current | With approval |
| `model_experiment` | New AI model provider | benchmark_score ≥ current × 0.95 | With CTO approval |
| `agent_experiment` | New agent class | task_success_rate ≥ 0.85 | With approval |
| `tool_experiment` | New tool capability | zero errors in N=50 tasks | With approval |
| `plugin_experiment` | New marketplace plugin | passes plugin_contract_tests | Auto (community plugins) |

### 15.4 Experiment Lifecycle

```
1. DEFINE
   POST /api/v2/lab/experiments
   {type, hypothesis, change_spec, success_criteria, max_budget_usd, max_tasks}

2. SIMULATE (Chapter 9)
   SimulationEngine.run(experiment.change_spec)
   → risk_score, predicted_impact

3. BENCHMARK (Chapter 14)
   BenchmarkCenter.run_targeted(experiment.affected_task_types)
   → lab_scores vs production_scores

4. SHADOW EXECUTION
   Route N% of non-critical tasks to Lab environment
   Collect outcomes (without affecting production artifacts)
   Parallel: real task on prod + experiment in lab → compare

5. REGRESSION CHECK
   Automated: all benchmark scores ≥ current × 0.97 (3% regression allowed)
   Automated: constitution_violations in lab == 0
   Automated: error_rate in lab ≤ prod_error_rate

6. APPROVAL GATE
   Present results to lead engineer / CTO (depending on experiment type)
   Show: control vs treatment performance, cost delta, risk assessment

7. PROMOTE
   If approved: deploy change to production
   Rollback plan: one-click revert to pre-experiment config
   Lab environment: reset for next experiment
```

### 15.5 Database Schema

```sql
CREATE TABLE lab_experiments (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                TEXT NOT NULL,
    type                TEXT NOT NULL,
    hypothesis          TEXT NOT NULL,
    change_spec         JSONB NOT NULL,
    success_criteria    JSONB NOT NULL,
    max_budget_usd      NUMERIC(10,4),
    max_tasks           INTEGER DEFAULT 100,
    status              TEXT DEFAULT 'defined',
    -- defined|simulating|benchmarking|shadow_running|regression_check|
    -- awaiting_approval|promoting|completed|failed|cancelled
    simulation_id       UUID REFERENCES simulations(id),
    benchmark_run_id    UUID REFERENCES benchmark_runs(id),
    shadow_task_count   INTEGER DEFAULT 0,
    control_success_rate NUMERIC(5,4),
    treatment_success_rate NUMERIC(5,4),
    regression_passed   BOOLEAN,
    approved_by         UUID REFERENCES users(id),
    promoted_at         TIMESTAMPTZ,
    rollback_snapshot   JSONB,
    created_by          UUID REFERENCES users(id),
    created_at          TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE lab_shadow_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    experiment_id   UUID REFERENCES lab_experiments(id),
    task_id         UUID REFERENCES tasks(id),
    lab_status      TEXT,
    lab_cost_usd    NUMERIC(10,6),
    lab_tokens      INTEGER,
    lab_duration_s  NUMERIC(10,3),
    lab_success     BOOLEAN,
    prod_success    BOOLEAN,
    delta_cost      NUMERIC(10,6),
    delta_quality   NUMERIC(5,4),
    recorded_at     TIMESTAMPTZ DEFAULT NOW()
);
```

### 15.6 API

```
POST   /api/v2/lab/experiments              Define new experiment
GET    /api/v2/lab/experiments              List experiments
GET    /api/v2/lab/experiments/{id}         Experiment status + results
POST   /api/v2/lab/experiments/{id}/start   Begin simulation + shadow execution
POST   /api/v2/lab/experiments/{id}/promote Promote to production (post-approval)
POST   /api/v2/lab/experiments/{id}/cancel  Cancel and reset lab
GET    /api/v2/lab/experiments/{id}/compare Control vs treatment comparison report
POST   /api/v2/lab/reset                    Reset lab environment (nightly automated)
```

### 15.7 Desktop Integration

New panel: **Evolution Lab**

```
┌────────────────────────────────────────────────────────────────────┐
│  EVOLUTION LAB                                [+ New Experiment]   │
├────────────────────────────────────────────────────────────────────┤
│  NAME                   TYPE           STATUS          RESULT      │
│  Concise BackendAgent   prompt         shadow_running  +4.2% ✦    │
│  Haiku for QA docs      model          awaiting_appr.  -0.8% cost │
│  Parallel stage refact  workflow       benchmarking    running...  │
│  GPT-4o for planning    model_exper.   failed          -12% ✗     │
├────────────────────────────────────────────────────────────────────┤
│  LAB HEALTH: ● Online   Snapshot age: 6h   Budget used: $4.20     │
└────────────────────────────────────────────────────────────────────┘
```

---

## Integration Map

### Cross-Chapter Data Flows

```
Requirement submitted
        │
        ▼
Decision Engine (Ch.7) ──► selects Workflow (Ch.1) + Employee (Ch.5) + Model (Ch.14)
        │
        ▼
Constitution (Ch.10) ──► validates all decisions
        │
        ▼
Risk Score ≥ 70? ──► Simulation Engine (Ch.9) ──► risk report
        │
        ▼
Workflow Engine (Ch.1) ──► executes stages via Agent Orchestrator (base §5)
        │
        ├── Prompt Registry (Ch.2) ──► resolves prompt version per agent
        ├── Employee Scheduler (Ch.5) ──► assigns best employee per stage
        ├── Tool Runtime (base §12) ──► executes tools within sandbox
        ├── Constitution PDP (Ch.10) ──► enforces rules on every tool call
        └── Audit Logger (Ch.11) ──► records every event immutably
        │
        ▼
Task completes
        │
        ├── Cost Ledger (Ch.13) ──► records cost attribution
        ├── Prompt Scorer (Ch.2) ──► scores prompt versions used
        ├── Experience Recorder (Ch.6) ──► writes to Experience Graph
        ├── KG 2.0 Updater (Ch.8) ──► updates code + artifact graph
        ├── Outcome Collector (Ch.3) ──► records to task_outcomes
        └── Employee KPI Updater (Ch.5) ──► updates employee metrics
                │
                ▼
        Self-Improvement Engine (Ch.3)
                ├── FailureAnalyzer ──► improvement_proposals
                └── → Evolution Lab (Ch.15) ──► experiment if warranted
                        └── Benchmark Center (Ch.14) ──► validates improvement
```

### Coordinator Integration Points

| Chapter | Integration |
|---------|------------|
| Ch.1 — Workflow Engine | Coordinator routes task submissions to Workflow Engine first |
| Ch.2 — Prompts | Coordinator resolves prompts from registry for all agent calls |
| Ch.3 — Self-Improvement | Coordinator emits `task.completed` events; SIE subscribes |
| Ch.4 — Org Engine | Coordinator enforces org authority in all approval gate creation |
| Ch.5 — Employees | Coordinator replaces basic agent scheduling with Employee Scheduler |
| Ch.6 — Memory OS | Coordinator's Knowledge Engine extended with Experience Graph queries |
| Ch.7 — Decision Engine | Coordinator adds Decision Engine step before every task dispatch |
| Ch.8 — KG 2.0 | KG ingestion extends existing KG worker (base §11.1) |
| Ch.9 — Simulation | Coordinator checks risk score; triggers simulation before dispatch |
| Ch.10 — Constitution | PDP integrated into Tool Runtime (every tool call) |
| Ch.11 — Audit | Coordinator writes to audit_events on every API call and tool call |
| Ch.12 — Security | Coordinator enforces RBAC/ABAC on every API endpoint |
| Ch.13 — Economy | Coordinator checks budget before task start; ledger updated after |
| Ch.14 — Benchmarks | Coordinator's model selection reads from model_leaderboard |
| Ch.15 — Evolution Lab | Lab has its own Coordinator instance; shares only PostgreSQL schema |

---

## Revised Technology Stack (Extended)

| Addition | Technology | Chapter | Rationale |
|----------|-----------|---------|-----------|
| Workflow Engine | Custom Python (DAG executor) | 1 | Full control over saga, compensation, parallel stages |
| Prompt Registry | PostgreSQL + Redis cache | 2 | No new infra — extends existing DB |
| Outcome Store | PostgreSQL (task_outcomes) | 3 | Same DB, new table |
| Experience Graph | Neo4j extension (new node types) | 6 | Extends existing KG |
| Decision Engine | Python service | 7 | Lightweight, many DB reads |
| Secret Management | HashiCorp Vault / Infisical | 12 | Industry standard; local mode via OS keychain |
| Audit Chain | PostgreSQL (append-only + triggers) | 11 | No new infra; hash chain in DB |
| Cost Ledger | PostgreSQL + TimescaleDB extension | 13 | TimescaleDB for time-series aggregation |
| Benchmark Runner | Python worker | 14 | Extends existing agent pool |
| Lab Environment | Docker Compose (isolated) | 15 | Same images as prod, network-isolated |

---

## Revised Development Roadmap (Extensions)

| Phase | Chapters | Timeline | After Base Phase |
|-------|---------|----------|-----------------|
| Ext-1 | Ch.1 (Workflow) + Ch.10 (Constitution) | Months 4–5 | After base Phase 1 |
| Ext-2 | Ch.2 (Prompts) + Ch.3 (Self-Improvement) + Ch.11 (Audit) | Months 6–8 | After base Phase 2 |
| Ext-3 | Ch.5 (Employees) + Ch.4 (Org) + Ch.13 (Economy) | Months 8–10 | During base Phase 3 |
| Ext-4 | Ch.6 (Memory OS) + Ch.7 (Decisions) + Ch.8 (KG 2.0) | Months 10–12 | During base Phase 3–4 |
| Ext-5 | Ch.9 (Simulation) + Ch.12 (Security) + Ch.14 (Benchmarks) | Months 12–14 | During base Phase 4 |
| Ext-6 | Ch.15 (Evolution Lab) + full integration | Months 14–15 | Before base GA |

---

## 5–10 Year Product Vision

```
YEAR 1–2: AI-Assisted Development (v1.5.5 + v2.0 base)
  Human writes requirements → AI assists with code, tests, review

YEAR 2–3: AI-Led Development (v2.0 + Chapters 1–10)
  Human defines goals → AI plans, executes, and delivers
  Human approves checkpoints — not every line of code

YEAR 3–5: Autonomous Software Teams (v2.0 full OS)
  Human sets product vision → AI company delivers quarterly roadmap
  Human audits and governs — not daily execution
  Digital employees have career paths, specializations, reputation

YEAR 5–7: AI Software Company
  Multiple AI organizations running in parallel (enterprise multi-tenant)
  AI employees collaborate across org boundaries
  Self-improving platform outperforms static tools by 10×

YEAR 7–10: AI Operating System for Enterprises
  Every software team augmented by persistent AI organization
  Code, infrastructure, security, compliance — fully autonomous
  Human role: executive oversight, ethics, product vision
  AI Studio 2.0 becomes the OS layer for software civilization
```

---

*Document version: 2.0.0-EXT-DRAFT*  
*Extension to: AI-STUDIO-2.0-ARCHITECTURE.md*  
*All 15 chapters — complete single document*  
*Last updated: 2026-06-27*
