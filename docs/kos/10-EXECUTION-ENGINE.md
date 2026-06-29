---
knowledge_id: KNW-KOS-ARCH-010
title: "KOS Knowledge Execution Engine"
status: AUTHORITATIVE
phase: "3.0A"
version: "1.0.0"
created: "2026-06-30"
purpose: "Specify the Knowledge Execution Engine: transforming knowledge into executable plans"
canonical_source: "architecture/docs/kos/10-EXECUTION-ENGINE.md"
dependencies:
  - "04-OBJECT-MODEL.md"
  - "05-KNOWLEDGE-GRAPH.md"
  - "09-REASONING-ENGINE.md"
related_documents:
  - "06-KNOWLEDGE-COMPILER.md"
  - "16-IMPLEMENTATION-ROADMAP.md"
acceptance_criteria:
  - "All 6 plan types are specified"
  - "Plan generation pipeline is defined"
  - "Plans are traceable to knowledge objects"
verification_checklist:
  - "[ ] ExecutionPlan type hierarchy defined"
  - "[ ] Plan generation inputs specified"
  - "[ ] Output format is machine-parseable"
future_extensions:
  - "Autonomous plan execution via AI agents"
  - "Plan simulation before execution"
---

# KOS Knowledge Execution Engine

## Purpose

The Knowledge Execution Engine transforms Knowledge Objects into **executable plans**.
These are not vague instructions — they are structured, step-by-step plans that agents
and runtimes can execute directly.

The Execution Engine consumes:
- Knowledge Objects (what exists and what it means)
- Knowledge Graph (what depends on what)
- Reasoning Engine outputs (why and how)
- Execution Context (current state of reality)

It produces:
- Executable plans of 6 types (see below)

---

## Plan Types

### Plan Type 1 — Software Plan

A plan for implementing a software artifact from its Knowledge Objects.

```python
@dataclass(frozen=True)
class SoftwarePlan:
    plan_id: str
    plan_type: Literal["software"]
    target_knowledge_id: str          # What is being built
    steps: list[SoftwareStep]
    estimated_files: int
    estimated_lines: int
    estimated_tests: int
    dependencies_to_build_first: list[str]
    knowledge_ids_referenced: list[str]
```

**Example:** Plan to implement `KNW-AI-MOD-quota_manager`:
```
Step 1: Verify prerequisites (usage_collector exists and CANONICAL)
Step 2: Generate module scaffold from compiler template
Step 3: Implement check() method per ALG-quota-check-001
Step 4: Implement consume() method per ALG-quota-consume-001
Step 5: Register capabilities in module.yaml
Step 6: Write unit tests for all 5 acceptance criteria
Step 7: Run platform-kos validate --implementation
```

---

### Plan Type 2 — Agent Plan

A plan for AI agents to execute multi-step tasks.
Generated from `Task` and `Strategy` knowledge objects.

```python
@dataclass(frozen=True)
class AgentPlan:
    plan_id: str
    plan_type: Literal["agent"]
    task_knowledge_id: str
    strategy_knowledge_id: str
    agents: list[AgentTask]
    execution_strategy: str        # single_agent | multi_agent_parallel | sequential
    estimated_tokens: int
    estimated_cost_usd: Decimal
```

---

### Plan Type 3 — Testing Plan

A plan for verifying that implementations match their knowledge specifications.

```python
@dataclass(frozen=True)
class TestingPlan:
    plan_id: str
    plan_type: Literal["testing"]
    scope_knowledge_ids: list[str]
    test_types: list[str]         # unit, integration, architecture, e2e
    generated_tests: list[str]   # file paths of generated test files
    manual_tests_needed: list[str]
    coverage_target: float
    estimated_runtime_seconds: int
```

---

### Plan Type 4 — Deployment Plan

A plan for releasing verified artifacts to an environment.

```python
@dataclass(frozen=True)
class DeploymentPlan:
    plan_id: str
    plan_type: Literal["deployment"]
    deployment_knowledge_id: str
    environment: str              # "development" | "staging" | "production"
    pre_deployment_checks: list[str]
    artifact_checksums: dict[str, str]
    rollback_strategy: str
    knowledge_gate: str           # GR-008 traceability must pass
```

---

### Plan Type 5 — Research Plan

A plan for investigating a domain, question, or problem.
Used by the Research Runtime (future).

```python
@dataclass(frozen=True)
class ResearchPlan:
    plan_id: str
    plan_type: Literal["research"]
    research_question: str
    knowledge_ids_to_investigate: list[str]
    data_sources: list[str]
    expected_outputs: list[str]   # new knowledge objects to create
    estimated_duration_hours: float
```

---

### Plan Type 6 — Trading Plan

A plan for financial strategy execution.
Used by the Financial Runtime (future).

```python
@dataclass(frozen=True)
class TradingPlan:
    plan_id: str
    plan_type: Literal["trading"]
    strategy_knowledge_id: str
    market_knowledge_ids: list[str]
    entry_conditions: list[str]
    exit_conditions: list[str]
    risk_parameters: dict
    estimated_exposure_usd: Decimal
```

---

## Plan Generation Pipeline

```
Request (what needs to be planned)
    │
    ▼
1. IDENTIFY target knowledge object(s)
    │
    ▼
2. FETCH knowledge graph subgraph (relevant context)
    │
    ▼
3. REASON about current state vs. target state
   (Reasoning Engine: what exists vs. what's needed)
    │
    ▼
4. SEQUENCE steps in dependency order
   (topological sort of knowledge dependencies)
    │
    ▼
5. ESTIMATE resources
   (tokens, time, files, cost — via AI Intelligence Engine)
    │
    ▼
6. VALIDATE plan against Golden Rules
   (GR-005: no step without knowledge reference)
    │
    ▼
7. ANNOTATE plan with knowledge_ids at every step
    │
    ▼
ExecutionPlan
```

---

## Execution Engine Interface

```python
class KnowledgeExecutionEngine(Protocol):
    def plan_software(
        self,
        knowledge_id: str,
        context: ExecutionContext,
    ) -> SoftwarePlan: ...

    def plan_agents(
        self,
        task_id: str,
        strategy_id: str,
    ) -> AgentPlan: ...

    def plan_testing(
        self,
        scope: list[str],
        test_types: list[str],
    ) -> TestingPlan: ...

    def plan_deployment(
        self,
        deployment_id: str,
        environment: str,
    ) -> DeploymentPlan: ...

    def plan_research(
        self,
        question: str,
        domain: str,
    ) -> ResearchPlan: ...

    def plan_trading(
        self,
        strategy_id: str,
        market_ids: list[str],
    ) -> TradingPlan: ...

    def from_question(
        self,
        question: str,
    ) -> ExecutionPlan: ...
    # Auto-detect plan type from natural language
```

---

## Execution Context

The Execution Engine needs to know the current state of the world to plan
the delta between now and the target:

```python
@dataclass(frozen=True)
class ExecutionContext:
    compiled_at: datetime
    platform_version: str
    canonical_objects: int
    approved_objects: int
    implementation_coverage: float   # % of modules with CANONICAL knowledge
    test_coverage: float
    active_runtimes: list[str]
    active_products: list[str]
    pending_migrations: list[str]
```

---

## Plan CLI

```bash
# Plan implementation of a module
platform-kos plan software KNW-AI-MOD-new_feature

# Plan all tests for a domain
platform-kos plan testing --domain ai --types unit,integration

# Plan a deployment
platform-kos plan deployment --env staging --version 2.2.0

# Plan from natural language
platform-kos plan "implement the financial runtime"
```
