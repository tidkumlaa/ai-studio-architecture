---
knowledge_id: KNW-PLAT-ARCH-008
title: "Platform SDK Model"
status: AUTHORITATIVE
phase: "2.1D.0"
version: "1.0.0"
created: "2026-06-29"
purpose: "Define the PlatformSDK — the only public API surface products may use"
canonical_source: "architecture/docs/platform-consolidation/08-SDK-MODEL.md"
dependencies:
  - "07-KERNEL-MODEL.md"
  - "09-SERVICE-MODEL.md"
related_documents:
  - "10-DEPENDENCY-RULES.md"
  - "12-IMPORT-RULES.md"
  - "30-API.md"
acceptance_criteria:
  - "SDK imports only from platform.kernel and platform.services"
  - "Every SDK function is typed with complete signatures"
  - "Stability level defined for every SDK module"
  - "Breaking change policy documented"
verification_checklist:
  - "[ ] SDK has no imports from platform.runtimes.*"
  - "[ ] All public functions have docstrings"
  - "[ ] Backwards compatibility tests exist"
  - "[ ] Version bumping policy documented"
future_extensions:
  - "SDK type stubs (.pyi) for IDE support"
  - "SDK versioning independent of platform version"
---

# Platform SDK Model

## SDK Axioms

1. **Products import only from `platform.sdk.*`**. Never from runtimes, services, or kernel directly.
2. **SDK is a stable facade**. Internal refactoring never breaks SDK signatures.
3. **SDK imports only from kernel and services**. Never from runtimes directly.
4. **Async-first**. All I/O operations return coroutines.
5. **Typed**. All inputs and outputs are Pydantic models or Python builtins.

---

## SDK Surface Map

```
platform/sdk/
├── __init__.py        ← re-exports all stable public symbols
├── ai.py              ← AI execution and routing (STABLE)
├── knowledge.py       ← Knowledge graph queries (STABLE)
├── providers.py       ← Provider management (STABLE)
├── workflows.py       ← Workflow definition and execution (BETA)
├── resources.py       ← Resource and quota management (STABLE)
├── events.py          ← Event publishing (STABLE)
├── types.py           ← Shared types (STABLE)
└── exceptions.py      ← SDK exceptions (STABLE)
```

---

## sdk.ai — AI Execution and Routing

```python
"""platform.sdk.ai — AI routing and execution facade. STABLE."""

async def optimize_routing(request: RoutingRequest) -> RoutingResult:
    """Select the optimal provider/model/account for a task."""

async def analyze_task(task_text: str, org_id: UUID) -> TaskAnalysis:
    """Profile workload, estimate complexity, tokens, cost, quality requirements."""

async def plan_execution(task_text: str, org_id: UUID, **opts) -> ExecutionPlan:
    """Build a single or multi-agent execution plan."""

async def predict_cost(
    provider_id: str,
    model_id: str,
    task_text: str,
) -> CostPrediction:
    """Predict token count and cost for a provider/model."""

async def get_recommendations(org_id: UUID, task_text: str = "") -> list[Recommendation]:
    """Get actionable recommendations for current resource state."""

async def record_execution_outcome(outcome: ExecutionOutcome) -> None:
    """Record actual token usage for learning improvement."""
```

**Stability:** STABLE — no breaking changes without major version bump.

---

## sdk.knowledge — Knowledge Graph Queries

```python
"""platform.sdk.knowledge — Knowledge graph facade. STABLE."""

async def search(query: str, limit: int = 10) -> list[KnowledgeResult]:
    """Semantic search across all knowledge objects."""

async def get_object(knowledge_id: str) -> KnowledgeObject | None:
    """Retrieve a specific knowledge object by ID."""

async def compile_source(source_path: str) -> KnowledgeObject:
    """Compile a source file or document into a knowledge object."""

async def link(source_id: str, target_id: str, relation: str) -> None:
    """Create a directed relationship between knowledge objects."""

async def related(knowledge_id: str, relation: str | None = None) -> list[KnowledgeObject]:
    """Find objects related to a given knowledge object."""
```

---

## sdk.providers — Provider Management

```python
"""platform.sdk.providers — Provider plugin management facade. STABLE."""

def register_provider(plugin: AIProviderPlugin) -> None:
    """Register an AI provider plugin."""

async def get_provider(provider_id: str) -> AIProviderPlugin | None:
    """Retrieve a registered provider by ID."""

async def provider_health(provider_id: str) -> ProviderHealth:
    """Check current health of a provider."""

def list_providers() -> list[str]:
    """List all registered provider IDs."""
```

---

## sdk.resources — Resource and Quota Management

```python
"""platform.sdk.resources — Resource and quota facade. STABLE."""

async def check_quota(
    org_id: UUID,
    provider_id: str,
    estimated_tokens: int,
) -> QuotaCheckResult:
    """Check if quota is available before execution."""

async def record_usage(record: UsageRecord) -> None:
    """Record completed execution for quota tracking."""

async def get_subscription_status(
    org_id: UUID,
) -> list[SubscriptionStatus]:
    """Get current status of all subscriptions for an org."""

async def get_budget_status(org_id: UUID) -> list[BudgetStatus]:
    """Get current API budget status for an org."""

async def forecast_usage(
    org_id: UUID,
    provider_id: str,
    days: int = 7,
) -> ForecastResult:
    """Forecast token usage and quota exhaustion."""
```

---

## sdk.workflows — Workflow Execution (BETA)

```python
"""platform.sdk.workflows — Workflow definition and execution. BETA."""

async def define_workflow(definition: WorkflowDefinition) -> WorkflowID:
    """Register a workflow definition."""

async def run_workflow(
    workflow_id: WorkflowID,
    inputs: dict,
    org_id: UUID,
) -> WorkflowRun:
    """Execute a workflow and return the run handle."""

async def get_workflow_status(run_id: WorkflowRunID) -> WorkflowStatus:
    """Poll workflow run status."""
```

**Stability:** BETA — signatures may change with minor version bumps (semver minor).

---

## sdk.events — Event Publishing

```python
"""platform.sdk.events — Event publishing facade. STABLE."""

async def publish(event: PlatformEvent) -> None:
    """Publish an event to the platform event bus."""

async def subscribe(
    event_type: str,
    handler: AsyncCallable[[PlatformEvent], None],
) -> Subscription:
    """Subscribe to platform events."""
```

---

## SDK Types (`sdk.types`)

```python
# All SDK-level Pydantic models live here
class RoutingRequest(BaseModel): ...
class RoutingResult(BaseModel): ...
class TaskAnalysis(BaseModel): ...
class ExecutionPlan(BaseModel): ...
class CostPrediction(BaseModel): ...
class Recommendation(BaseModel): ...
class UsageRecord(BaseModel): ...
class QuotaCheckResult(BaseModel): ...
class ForecastResult(BaseModel): ...
class WorkflowDefinition(BaseModel): ...
class WorkflowRun(BaseModel): ...
class KnowledgeResult(BaseModel): ...
class ProviderHealth(BaseModel): ...
```

All SDK types are re-exported from `platform.sdk.types`. Products use only these types.

---

## Stability Policy

| Level | Guarantees | Change notification |
|-------|-----------|---------------------|
| STABLE | No breaking changes in minor or patch | Breaking change = major version |
| BETA | May break in minor versions | 1 minor version deprecation warning |
| EXPERIMENTAL | May break in any release | No notice required |
| DEPRECATED | Will be removed | Removed in next major version |

### Breaking vs. Non-breaking

**Breaking:**
- Removing a function
- Changing a function signature (adding required params, changing return type)
- Renaming a module

**Non-breaking:**
- Adding optional parameters with defaults
- Adding new functions
- Expanding union types
- Adding new fields to response models

---

## SDK Versioning

SDK version is a semver value in `pyproject.toml`. Products pin to `>=2.0.0,<3.0.0`.

```toml
[project]
version = "2.0.0"  # major.minor.patch
```

SDK changelog lives in `platform/docs/SDK-CHANGELOG.md`.
