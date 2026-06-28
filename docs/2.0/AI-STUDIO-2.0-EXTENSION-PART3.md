# AI Studio 2.0 — AI Operating System Extension
# Architecture Specification — Part 3 (Chapters 11–15 + Integration)

---

## Chapter 11 — Governance & Audit

### 11.1 Vision

Every action taken by every agent, tool, or human in the platform generates an immutable audit event. The Governance & Audit system is the compliance backbone — it answers "who did what, when, why, at what cost, using which model and prompt version" for every event in the system's history.

This is the foundation for SOC 2, ISO 27001, HIPAA, GDPR, and enterprise procurement requirements.

### 11.2 Audit Event Schema

```sql
-- Immutable audit log (partitioned, append-only)
-- Identical to the 'events' table in base schema but extended with governance fields
CREATE TABLE audit_events (
    id              BIGSERIAL,
    event_id        UUID DEFAULT gen_random_uuid(),
    project_id      UUID REFERENCES projects(id),
    tenant_id       UUID,                           -- for multi-tenant

    -- WHO
    actor_type      TEXT NOT NULL,                  -- 'agent'|'human'|'system'|'plugin'
    actor_id        TEXT NOT NULL,                  -- employee_id, user_id, or 'system'
    actor_role      TEXT,                           -- org role at time of action
    ip_address      INET,                           -- for human actors
    session_id      UUID,

    -- WHAT
    action          TEXT NOT NULL,                  -- 'tool_called'|'task_created'|'commit_pushed'|...
    resource_type   TEXT,                           -- 'file'|'task'|'deployment'|'prompt'|...
    resource_id     TEXT,
    input_summary   TEXT,                           -- truncated input (first 500 chars)
    output_summary  TEXT,                           -- truncated output (first 500 chars)

    -- WHEN
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- WHY
    task_id         UUID REFERENCES tasks(id),
    workflow_id     UUID REFERENCES workflows(id),
    decision_id     UUID REFERENCES decisions(id),
    justification   TEXT,                           -- human-provided or AI-derived reason

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
    previous_hash   TEXT,                           -- hash of previous row (chain integrity)
    row_hash        TEXT NOT NULL,                  -- SHA-256 of all fields above
    signed_by       TEXT,                           -- KMS key ID (enterprise)

    PRIMARY KEY (id, occurred_at)
) PARTITION BY RANGE (occurred_at);
-- CREATE TABLE audit_events_2026_06 PARTITION OF audit_events FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');
-- Partition created automatically by monthly cron

-- No UPDATE or DELETE allowed — enforced by:
CREATE RULE no_update_audit AS ON UPDATE TO audit_events DO INSTEAD NOTHING;
CREATE RULE no_delete_audit AS ON DELETE TO audit_events DO INSTEAD NOTHING;
```

### 11.3 Cryptographic Chain Integrity

Each audit event includes `previous_hash` (SHA-256 of the previous row) forming a hash chain. Tampering with any past row breaks all subsequent hashes — detectable in seconds:

```python
class AuditChainVerifier:
    async def verify(self, from_id: int, to_id: int) -> VerificationResult:
        events = await self.db.fetch("SELECT * FROM audit_events WHERE id BETWEEN $1 AND $2 ORDER BY id", from_id, to_id)
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
  Role: 'engineer' → can: write_file, run_tests, commit_feature_branch
  Role: 'lead' → can: approve_pr, deploy_staging, manage_team
  Role: 'admin' → can: all of above + manage_constitution, manage_users

ABAC (Attribute-Based): Context-specific access
  Allow deploy to production IF:
    user.role IN ['cto', 'admin']
    AND approval_gate.count >= 2
    AND simulation.recommendation == 'SAFE'
    AND time_of_day BETWEEN '08:00' AND '20:00' (maintenance window)
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

Agent access pattern:
  # Agents never see raw secrets in prompt context
  # Tool Runtime injects secrets as environment variables into sandbox
  # Sandbox is destroyed after tool call → no secret persistence
```

### 12.5 Tenant Isolation (Multi-Tenant SaaS)

```
Isolation levels:
  L1 — Row-Level Security (RLS): all tables have tenant_id column + RLS policy
  L2 — Schema isolation: one PostgreSQL schema per tenant (enterprise tier)
  L3 — Database isolation: one PostgreSQL instance per tenant (critical data)
  L4 — Cluster isolation: dedicated K8s namespace per tenant (regulated industries)

Default: L1 (RLS)
Enterprise: L2 or L3 (negotiated per contract)
```

### 12.6 Database Schema

```sql
-- Permission grants (ABAC rules stored as data)
CREATE TABLE permission_grants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    grantee_type    TEXT NOT NULL,          -- 'user'|'role'|'employee'
    grantee_id      TEXT NOT NULL,
    resource_type   TEXT NOT NULL,          -- 'project'|'workflow'|'deployment'|'*'
    resource_id     TEXT,                   -- specific ID or '*' for all
    action          TEXT NOT NULL,          -- 'read'|'write'|'execute'|'approve'|'*'
    conditions      JSONB DEFAULT '{}',     -- ABAC conditions
    granted_by      UUID REFERENCES users(id),
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE api_tokens (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID REFERENCES users(id),
    name            TEXT NOT NULL,
    token_hash      TEXT UNIQUE NOT NULL,   -- bcrypt hash
    scopes          TEXT[] NOT NULL,        -- ['read', 'write', 'deploy']
    last_used_at    TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    revoked_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE security_scan_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id         UUID REFERENCES tasks(id),
    scan_type       TEXT NOT NULL,          -- 'sast'|'sca'|'secret'|'container'|'iac'
    tool            TEXT NOT NULL,          -- 'semgrep'|'pip-audit'|'owasp-dc'|'trufflehog'|'trivy'
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

All findings are stored in `security_scan_results` and linked to KG via `(:SecurityFinding)-[:FOUND_IN]->(:File)`.

---

## Chapter 13 — AI Economy Engine

### 13.1 Vision

Every compute action has a cost. The AI Economy Engine makes those costs visible, attributable, and optimizable. Projects have budgets. Departments have allocations. Individual tasks have cost limits. The platform continuously identifies cheaper alternatives that achieve equivalent quality.

### 13.2 Cost Attribution Model

```
Task cost attribution hierarchy:

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
    period          TEXT NOT NULL,          -- '2026-06' (monthly) or '2026-Q2' (quarterly)
    budget_usd      NUMERIC(12,4) NOT NULL,
    alert_threshold NUMERIC(5,2) DEFAULT 0.80,   -- alert at 80% consumption
    hard_limit      BOOLEAN DEFAULT FALSE,        -- if true: block new tasks when exceeded
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
    units           NUMERIC(12,4),          -- tokens, seconds, etc.
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
    estimated_human_hours NUMERIC(10,2),    -- tasks done * avg human hours per task type
    human_cost_usd  NUMERIC(12,4),          -- estimated_human_hours * hourly_rate
    roi_ratio       NUMERIC(8,4),           -- human_cost / ai_cost
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
    """Runs weekly; analyzes spending patterns and proposes model swaps."""

    async def analyze(self, project_id: UUID, period: str) -> list[CostProposal]:
        proposals = []

        # Pattern 1: Model overkill — expensive model on simple tasks
        overpriced = await self.db.fetch("""
            SELECT ae.model, t.type, AVG(ae.cost_usd) as avg_cost,
                   AVG(CASE WHEN to2.status='completed' THEN 1.0 ELSE 0.0 END) as success_rate
            FROM agent_executions ae
            JOIN tasks t ON ae.task_id = t.id
            JOIN task_outcomes to2 ON t.id = to2.task_id
            WHERE t.project_id = $1
            GROUP BY ae.model, t.type
            HAVING AVG(ae.cost_usd) > 0.10   -- expensive tasks
        """, project_id)

        for row in overpriced:
            cheaper = await self.benchmark_center.find_cheaper_equivalent(
                task_type=row.type,
                current_model=row.model,
                min_success_rate=row.success_rate * 0.97   # allow 3% quality drop
            )
            if cheaper and cheaper.avg_cost < row.avg_cost * 0.6:  # 40%+ savings
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
     → If not: BLOCK with BudgetExceeded error; notify PM
  3. Check: task.cost_budget >= estimated cost
     → If not: WARN + create approval gate

During task:
  Streaming token counter → if tokens_used > task.token_budget × 0.9:
     emit budget.warning event → Desktop notification
  If tokens_used > task.token_budget:
     CONST-004 triggers → execution halted → partial result saved

After task:
  Record actual cost to cost_ledger
  Update budget consumed amount
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
├──────────────────────────────────────────────────────────────────--─┤
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

### 14.2 Benchmark Suite

```
benchmark-suite/
├── code_generation/
│   ├── python_api_endpoint.yaml       # write a FastAPI endpoint
│   ├── sql_migration.yaml             # write a SQLAlchemy migration
│   ├── react_component.yaml           # write a TypeScript React component
│   └── fix_failing_test.yaml          # fix a broken test case
├── code_review/
│   ├── security_vulnerability.yaml    # detect injected vulnerability
│   ├── logic_error.yaml               # detect subtle logic bug
│   └── performance_issue.yaml         # identify N+1 query
├── planning/
│   ├── decompose_feature.yaml         # break down a feature into tasks
│   └── estimate_complexity.yaml       # estimate story points
├── documentation/
│   ├── generate_readme.yaml           # generate README from code
│   └── generate_changelog.yaml        # generate changelog from commits
└── reasoning/
    ├── architecture_tradeoff.yaml     # recommend architecture pattern
    └── debug_stack_trace.yaml         # identify root cause from stack trace
```

Each benchmark task has:
- Fixed input (deterministic)
- Reference answer (for accuracy scoring)
- Evaluation rubric (for LLM-as-judge scoring)
- Expected token budget

### 14.3 Database Schema

```sql
CREATE TABLE benchmark_tasks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    category        TEXT NOT NULL,
    name            TEXT NOT NULL,
    description     TEXT,
    input           JSONB NOT NULL,         -- fixed input
    reference_answer TEXT,                  -- for exact-match metrics
    evaluation_rubric JSONB,               -- for LLM-as-judge
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
    triggered_by    TEXT DEFAULT 'scheduled',   -- 'scheduled'|'on_model_add'|'manual'
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
    accuracy_score  NUMERIC(5,4),           -- 0.0-1.0 (exact match or rubric score)
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
    category        TEXT NOT NULL,          -- task category
    avg_accuracy    NUMERIC(5,4),
    avg_latency_ms  INTEGER,
    avg_cost_usd    NUMERIC(10,6),
    hallucination_rate NUMERIC(5,4),
    pass_rate       NUMERIC(5,4),           -- % tasks passing acceptance threshold
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
    judge_prompt = f"Rate 1-10: does this answer satisfy: {rubric}\nAnswer: {output}"
    score = judge_model.complete(judge_prompt) / 10.0
    accuracy = score

Hallucination Detection:
  - Code tasks: compile + run tests → hallucination if syntax error
  - Factual tasks: cross-reference against KG → flag unsupported claims

Composite Score (for leaderboard):
  composite = (0.50 × accuracy) + (0.20 × cost_efficiency) + (0.20 × latency_score) + (0.10 × (1 - hallucination_rate))
  where cost_efficiency = min(expected_tokens, actual_tokens) / max(expected_tokens, actual_tokens)
  where latency_score = min(2000, latency_ms) / 2000  (inverted: lower latency = higher score)
```

### 14.5 Automated Scheduling

```python
# Run full benchmark suite: weekly (Sunday 02:00 UTC)
# Run category benchmarks on model add or model config change
# Run targeted re-benchmark after model version update

BenchmarkScheduler:
  - full_suite_cron: "0 2 * * 0"
  - trigger: model_added, model_version_changed, provider_config_changed
  - on_complete: update model_leaderboard, emit benchmark.completed event
  - notification: "Model claude-opus-4-8 ranked #1 for planning tasks (score: 0.94)"
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

This is how the platform evolves without breaking what works.

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
         │  Never: Lab writes to prod DB directly
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
| `prompt_experiment` | Prompt version A/B | improvement_proposals accuracy +5% | With approval |
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
   Parallel: real task runs on prod, experiment runs in lab → compare

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
    change_spec         JSONB NOT NULL,         -- what to change (prompt diff, model name, etc.)
    success_criteria    JSONB NOT NULL,
    max_budget_usd      NUMERIC(10,4),
    max_tasks           INTEGER DEFAULT 100,
    status              TEXT DEFAULT 'defined',
    -- defined|simulating|benchmarking|shadow_running|regression_check|awaiting_approval|promoting|completed|failed|cancelled
    simulation_id       UUID REFERENCES simulations(id),
    benchmark_run_id    UUID REFERENCES benchmark_runs(id),
    shadow_task_count   INTEGER DEFAULT 0,
    control_success_rate NUMERIC(5,4),
    treatment_success_rate NUMERIC(5,4),
    regression_passed   BOOLEAN,
    approved_by         UUID REFERENCES users(id),
    promoted_at         TIMESTAMPTZ,
    rollback_snapshot   JSONB,                  -- state before promotion, for rollback
    created_by          UUID REFERENCES users(id),
    created_at          TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE lab_shadow_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    experiment_id   UUID REFERENCES lab_experiments(id),
    task_id         UUID REFERENCES tasks(id),      -- prod task being shadowed
    lab_status      TEXT,
    lab_cost_usd    NUMERIC(10,6),
    lab_tokens      INTEGER,
    lab_duration_s  NUMERIC(10,3),
    lab_success     BOOLEAN,
    prod_success    BOOLEAN,
    delta_cost      NUMERIC(10,6),                  -- lab - prod
    delta_quality   NUMERIC(5,4),                   -- lab_score - prod_score
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
├──────────────────────────────────────────────────────────────────--┤
│  NAME                   TYPE           STATUS          RESULT      │
│  Concise BackendAgent   prompt         shadow_running  +4.2% ✦    │
│  Haiku for QA docs      model          awaiting_appr.  -0.8% cost │
│  Parallel stage refact  workflow       benchmarking    running...  │
│  GPT-4o for planning    model_exper.   failed          -12% ✗     │
├─────────────────────────────────────────────────────────────────--─┤
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

All 15 chapter systems integrate with the existing Coordinator service (base §1.2) via:

| Integration Point | Method |
|-----------------|--------|
| Chapter 1 (Workflow) | Coordinator routes task submissions to Workflow Engine first |
| Chapter 2 (Prompts) | Coordinator resolves prompts from registry for all agent calls |
| Chapter 3 (Self-Improvement) | Coordinator emits `task.completed` events; SIE subscribes |
| Chapter 4 (Org) | Coordinator enforces org authority in all approval gate creation |
| Chapter 5 (Employees) | Coordinator replaces basic agent scheduling with Employee Scheduler |
| Chapter 6 (Memory OS) | Coordinator's Knowledge Engine extended with Experience Graph queries |
| Chapter 7 (Decisions) | Coordinator adds Decision Engine step before every task dispatch |
| Chapter 8 (KG 2.0) | KG ingestion extends existing KG worker (base §11.1) |
| Chapter 9 (Simulation) | Coordinator checks risk score; triggers simulation before dispatch |
| Chapter 10 (Constitution) | PDP integrated into Tool Runtime (every tool call) |
| Chapter 11 (Audit) | Coordinator writes to audit_events on every API call and tool call |
| Chapter 12 (Security) | Coordinator enforces RBAC/ABAC on every API endpoint |
| Chapter 13 (Economy) | Coordinator checks budget before task start; ledger updated after |
| Chapter 14 (Benchmarks) | Coordinator's model selection reads from model_leaderboard |
| Chapter 15 (Lab) | Lab has its own Coordinator instance; shares only PostgreSQL schema |

---

## Revised Technology Stack (Extended)

| Addition | Technology | Chapter | Rationale |
|----------|-----------|---------|-----------|
| Workflow Engine | Custom Python (DAG executor) | 1 | Full control over saga, compensation, parallel stages |
| Prompt Registry | PostgreSQL + Redis cache | 2 | Already present; no new infra |
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
YEAR 1–2: AI-Assisted Development (current v1.5.5 + v2.0 base)
  Human writes requirements → AI assists with code, tests, review

YEAR 2–3: AI-Led Development (v2.0 + Extension Chapters 1–10)
  Human defines goals → AI plans, executes, and delivers
  Human approves checkpoints → not every line of code

YEAR 3–5: Autonomous Software Teams (v2.0 full OS)
  Human sets product vision → AI company delivers quarterly roadmap
  Human audits and governs → not daily execution
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
*Part 3 of 3 — Chapters 11–15 + Integration*  
*Last updated: 2026-06-27*
