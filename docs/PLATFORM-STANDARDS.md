# AI Studio Platform โ€” Engineering Standards

**Document ID:** PLATFORM-STANDARDS  
**Version:** 1.0.0  
**Date:** 2026-06-28  
**Status:** RATIFIED  
**Classification:** INTERNAL โ€” ENGINEERING GOVERNANCE  
**Authority:** Chief Software Architect  
**Predecessor:** AI-STUDIO-PLATFORM-REFACTORING-BLUEPRINT.md  
**Review Cycle:** Quarterly (every 90 days or upon major platform release)

---

## Constitutional Notice

> These standards are the constitutional law of the AI Studio platform. They apply to every repository, every module, every AI agent, and every human contributor. They are not guidelines. They are not suggestions. They are not aspirational.
>
> A contribution that violates these standards is not merged, regardless of its functional correctness.
>
> Exceptions require written justification and CTO approval. Exceptions are tracked in `docs/standard-exceptions.md`.
>
> Where a standard contradicts an existing pattern in the codebase, the standard wins. The existing pattern is technical debt to be scheduled for remediation.

---

## How to Use This Document

- **Before writing code:** Read the relevant section. Every section has explicit rules and good/bad examples.
- **During code review:** Reference the checklist in Section 23. A PR fails review if checklist items are not met.
- **When onboarding:** Read Sections 1โ€“6 first. Then read the section for your domain.
- **When disagreeing with a standard:** Raise it via the quarterly review process, not by quietly violating it.

---

## Table of Contents

1. Purpose
2. Guiding Principles
3. Repository Standards
4. Folder Structure Standards
5. Module Naming Convention
6. Package Naming Convention
7. API Naming Standards
8. DTO Standards
9. Domain Model Rules
10. Dependency Injection Standards
11. Logging Standards
12. Error Handling Standards
13. Exception Hierarchy
14. Configuration Standards
15. Secrets Management
16. Event Naming Standards
17. Versioning Policy
18. Feature Flag Policy
19. Database Migration Standards
20. Documentation Standards
21. Testing Standards
22. Security Standards
23. Code Review Checklist
24. Definition of Done
25. Release Standards

---

---

# 1. Purpose

## 1.1 Why This Document Exists

AI Studio is a multi-product platform built on a shared infrastructure. As the codebase grows across products (`ai-software-factory`, `content-factory`, `mythic-realms`, and future products), across layers (`platform`, `sdk`, `products`, `desktop`), and across contributors (human engineers and AI agents), the absence of enforceable standards produces the following failure modes โ€” all already observed in the codebase as of the architecture review:

| Failure Mode | Observed Instance |
|-------------|------------------|
| Duplicate implementations | 4 concurrent prompt renderers; 3 ExperienceRecorder implementations |
| Security gaps from inconsistency | 10 route files missing auth dependency while 20 have it |
| Silent configuration drift | memory.db hardcoded in app.py; NATS disabled by default |
| Architectural violations | ProductFactory bypasses WorkflowRuntime; DecisionEngine has hardcoded model catalog |
| Test coverage gaps | No security regression tests; no E2E execution test |
| Naming chaos | `brain.py` and `central_brain.py` both exist; unclear which is canonical |

This document eliminates ambiguity. When a standard exists, the answer is known. Engineers spend time building, not debating.

## 1.2 Scope

These standards govern:
- All code in `E:\UserData\MyData\Content\AIStudio\` and its monorepo subdirectories
- All AI agents operating on this codebase (Claude Code and future agents)
- All automated CI/CD pipelines
- All human contributors

These standards do **not** govern:
- Third-party vendor code (providers, libraries)
- External services (NATS server configuration, PostgreSQL cluster setup)
- Generated code produced by platform tooling (scaffold outputs, migration boilerplate)

## 1.3 Authority and Maintenance

**Owner:** Chief Software Architect  
**Ratification:** CTO approval required to ratify or amend any section  
**Review:** Quarterly review cycle. Any team member may propose amendments via pull request  
**Exceptions:** Tracked in `docs/standard-exceptions.md` with mandatory fields: exception ID, standard violated, justification, expiry date, approver  

---

---

# 2. Guiding Principles

These principles are the foundation from which all specific standards are derived. When two standards appear to conflict, the principle takes precedence.

## 2.1 The Principles

### P-1: Platform Serves All Products Equally

The platform layer exists to serve every product uniformly. No platform module may include logic specific to one product. No product may receive preferential treatment in platform interfaces. Any feature request that benefits only one product belongs in the product layer, not the platform.

**Rationale:** The moment the platform includes product-specific logic, it can no longer be updated without coordinating with that product. The platform becomes a bottleneck instead of an accelerator.

---

### P-2: Explicit Is Better Than Implicit

Configuration, dependencies, and contracts must be explicit. Hidden defaults, implicit conventions, and environment-specific behavior that is not declared are prohibited.

**Rationale:** The architecture review found `memory.db` hardcoded in `app.py` โ€” a hardcoded implicit default that cannot be overridden via configuration. This caused the memory database to silently use a wrong path in non-local environments.

---

### P-3: One Canonical Implementation

For any capability (prompt rendering, AI execution, experience recording), there is exactly one canonical implementation. Multiple implementations of the same capability are forbidden unless they serve explicitly different purposes at different layers.

**Rationale:** Four prompt renderers means four places to fix a security vulnerability. One prompt renderer means one.

---

### P-4: Security Is Non-Negotiable and Non-Bypassable

Authentication, authorization, and rate limiting are applied at the platform middleware layer. No product code can bypass them. No route can be inadvertently unprotected. Security defaults are always the restrictive option.

**Rationale:** The architecture review found that the default configuration (`ORCH_API_KEY=""`) disables authentication entirely. A default that disables security is a catastrophic default.

---

### P-5: Every Action Is Observable

Every platform operation that affects state emits a structured event and a structured log line. Every AI execution is traced. Every error is recorded with full context. You can never have too much observability; you can always have too little.

**Rationale:** Without structured events, the Brain cannot learn. Without traces, production failures cannot be diagnosed.

---

### P-6: Tests Are Not Optional

Code without tests is not done. A feature without tests is a liability, not an asset. Tests are written before or alongside the feature โ€” never after merge.

**Rationale:** The architecture review found 39 pre-existing test failures and no E2E execution test for the core product creation workflow. The core value proposition was not tested.

---

### P-7: Products Are Consumers, Not Owners

Products consume platform capabilities via SDK clients. Products own only domain entities, business rules, and product-specific routes. Platform capabilities (AI routing, workflow dispatch, prompt governance, brain intelligence) are never owned by a product.

**Rationale:** When ProductFactory owned workflow dispatch, it replaced real dispatch with simulation. Platform ownership prevents this class of architectural compromise.

---

### P-8: Backward Compatibility Is a Contract

A published interface (SDK method signature, API endpoint, event schema, database column) may not be changed in a way that breaks existing consumers without a migration path and a version increment.

**Rationale:** AI Studio serves multiple products. A breaking change to the Workflow SDK without notice breaks every product simultaneously.

---

### P-9: Fail Fast, Fail Loud

Platform services that cannot start correctly must fail at startup, not at first use. Configuration errors must be caught at startup. Missing required dependencies must be caught at startup. Silent degradation is worse than a loud failure.

**Rationale:** A service that starts silently with a misconfigured database URL and then fails on the first request provides a misleading health signal.

---

### P-10: No Clever Code

Code must be readable by a new contributor without explanation. Cleverness that requires a comment to explain is not permitted. Optimize for clarity, not conciseness or performance, except at proven bottlenecks.

**Rationale:** AI agents, junior engineers, and on-call responders all read this code under time pressure. Clever code under time pressure causes incidents.

---

---

# 3. Repository Standards

**Rationale:** Consistent repository structure enables AI agents and engineers to navigate any repository without a guide. Every repository is predictable. Every tool knows where to look.

## 3.1 Rules

### R3.1 โ€” Monorepo layout is mandatory for platform and product code

All platform modules, SDK packages, products, and the desktop application reside in the single monorepo at `AIStudio/source/`. The canonical directory tree is defined in the approved Blueprint (Section 5). No new top-level directories may be added without CTO approval.

### R3.2 โ€” Every repository subdirectory is a self-contained Python package

Every `platform/*/`, `sdk/*/`, `products/*/`, and `desktop/*/` directory must contain:
- `pyproject.toml` with `[project]` section, dependencies, and `[tool.pytest.ini_options]`
- `src/` containing the importable source code
- `tests/` containing all tests for that package
- No code at the top level of the package directory (no `__init__.py` next to `pyproject.toml`)

### R3.3 โ€” Root-level files are restricted

The monorepo root (`AIStudio/source/`) contains only:
- `workspace.yaml` โ€” workspace definition
- `pyproject.toml` โ€” root workspace tooling configuration
- `.github/` โ€” CI/CD workflows
- `tools/` โ€” monorepo tooling (linters, scaffolding)
- `docs/` โ€” cross-cutting documentation

No application code. No `app.py`. No `main.py`. No `requirements.txt`.

### R3.4 โ€” Branch naming is enforced

| Branch type | Pattern | Example |
|-------------|---------|---------|
| Feature | `feature/<ticket-id>-<short-description>` | `feature/AISF-142-ai-ros-circuit-breaker` |
| Bug fix | `fix/<ticket-id>-<short-description>` | `fix/AISF-187-nats-subject-plural` |
| Platform release | `release/v<major>.<minor>.<patch>` | `release/v2.1.0` |
| Hotfix | `hotfix/<ticket-id>-<short-description>` | `hotfix/AISF-201-auth-bypass` |

`main` is the integration branch. Direct commits to `main` are prohibited. All changes enter via pull request.

### R3.5 โ€” Commit messages follow Conventional Commits

Format: `<type>(<scope>): <description>`

| Type | When to use |
|------|-------------|
| `feat` | New feature or capability |
| `fix` | Bug fix |
| `refactor` | Code change that neither adds a feature nor fixes a bug |
| `test` | Adding or fixing tests |
| `docs` | Documentation only |
| `chore` | Build system, CI, tooling |
| `perf` | Performance improvement |
| `security` | Security fix (use this type, triggers security review in CI) |

Scope is the package directory name: `ai-ros`, `workflow-runtime`, `prompt-os`, `aisf`, `desktop`.

### R3.6 โ€” `.gitignore` is standardized

The root `.gitignore` covers all packages. Individual packages must not have their own `.gitignore` files unless they require package-specific ignores (e.g., compiled Java artifacts in `content-factory`).

## 3.2 Good Examples

```
feat(workflow-runtime): add circuit breaker to task dispatch with configurable failure threshold
fix(aisf): remove _simulate_task_execution and wire WorkflowClient.submit
security(platform-security): replace != with hmac.compare_digest in api key validation
test(ai-ros): add BudgetManager daily limit enforcement tests
```

## 3.3 Bad Examples

```
# No type
updated workflow stuff

# No scope
feat: added thing

# Too vague
fix: fixed bug

# Too detailed (belongs in PR description)
feat(ai-ros): implement SchedulingEngine with URGENT/STANDARD/BATCH priority queues,
BudgetManager with per-product daily limits, QuotaManager with per-minute rate limiting,
CapabilityMatrix for provider capability matching, CircuitBreaker with 3-state FSM,
and TokenForecaster using historical averages and tiktoken estimation

# Wrong type
chore(aisf): fix critical security vulnerability
```

## 3.4 Common Mistakes

- Using `main` as scope (scope is a package, not a branch)
- Using present tense instead of imperative ("adds" instead of "add")
- Putting implementation details in the commit subject (use the body)
- Not using the `security` type for security fixes (bypasses CI security review trigger)

---

---

# 4. Folder Structure Standards

**Rationale:** When every package has the same layout, any engineer (or AI agent) can navigate any package without a tour. Tools like pytest, mypy, and the architecture linter depend on predictable structure.

## 4.1 Rules

### R4.1 โ€” Platform package structure

```
platform/<module-name>/
โ”โ”€โ”€ src/
โ”   โ””โ”€โ”€ aistudio/
โ”       โ””โ”€โ”€ platform/
โ”           โ””โ”€โ”€ <module_name>/
โ”               โ”โ”€โ”€ __init__.py         # Public API exports only
โ”               โ”โ”€โ”€ _internal/          # Private implementation (prefix _)
โ”               โ”   โ””โ”€โ”€ ...
โ”               โ”โ”€โ”€ models.py           # Pydantic models (public DTOs)
โ”               โ”โ”€โ”€ exceptions.py       # Module-specific exceptions
โ”               โ””โ”€โ”€ service.py          # Primary service class (or services/)
โ”โ”€โ”€ tests/
โ”   โ”โ”€โ”€ unit/
โ”   โ”   โ””โ”€โ”€ test_<component>.py
โ”   โ”โ”€โ”€ integration/
โ”   โ”   โ””โ”€โ”€ test_<component>_integration.py
โ”   โ””โ”€โ”€ conftest.py
โ””โ”€โ”€ pyproject.toml
```

### R4.2 โ€” SDK package structure

```
sdk/<sdk-name>/
โ”โ”€โ”€ src/
โ”   โ””โ”€โ”€ aistudio/
โ”       โ””โ”€โ”€ sdk/
โ”           โ””โ”€โ”€ <sdk_name>/
โ”               โ”โ”€โ”€ __init__.py         # Re-exports the public client class
โ”               โ”โ”€โ”€ client.py           # The SDK client class
โ”               โ”โ”€โ”€ models.py           # Request/response types
โ”               โ””โ”€โ”€ exceptions.py       # SDK-level exceptions
โ”โ”€โ”€ tests/
โ”   โ””โ”€โ”€ test_client.py
โ””โ”€โ”€ pyproject.toml
```

### R4.3 โ€” Product package structure

```
products/<product-name>/
โ”โ”€โ”€ src/
โ”   โ””โ”€โ”€ <product_namespace>/           # e.g., aisf/, mythic/, content/
โ”       โ”โ”€โ”€ domain/                    # Domain entities (pure Python, no DB)
โ”       โ”   โ”โ”€โ”€ <entity>.py
โ”       โ”   โ””โ”€โ”€ ...
โ”       โ”โ”€โ”€ <subdomain>/               # e.g., intake/, planning/, agents/
โ”       โ”   โ”โ”€โ”€ service.py             # Business logic service
โ”       โ”   โ””โ”€โ”€ models.py              # Subdomain-specific models
โ”       โ”โ”€โ”€ api/
โ”       โ”   โ””โ”€โ”€ <entity>_routes.py     # FastAPI routers
โ”       โ””โ”€โ”€ app.py                     # Product FastAPI application
โ”โ”€โ”€ tests/
โ”   โ”โ”€โ”€ unit/
โ”   โ”โ”€โ”€ integration/
โ”   โ””โ”€โ”€ conftest.py
โ””โ”€โ”€ pyproject.toml
```

### R4.4 โ€” Tests mirror source structure

For every `src/โ€ฆ/module.py` there must be a corresponding `tests/unit/test_module.py`. If a source file has no tests, it must have a documented reason in `tests/unit/test_<module>.py` that explains why (e.g., it is a pure data class tested via integration tests).

### R4.5 โ€” No circular package layouts

No package may import from its own `tests/` directory. No `__init__.py` may import implementation modules that depend on each other in a cycle. The architecture linter checks for this at CI time.

### R4.6 โ€” Private vs public distinction is enforced

Files and directories with a leading underscore (`_internal/`, `_helpers.py`) are private to the package. They must not be imported from outside the package. The architecture linter treats any cross-package import of a `_`-prefixed module as a violation.

## 4.2 Good Examples

```
platform/ai-ros/src/aistudio/platform/ai_ros/__init__.py  โ’ exports AIROSClient
platform/ai-ros/src/aistudio/platform/ai_ros/service.py   โ’ AIROSService implementation
platform/ai-ros/src/aistudio/platform/ai_ros/_internal/scheduling.py  โ’ private scheduler
platform/ai-ros/tests/unit/test_service.py
platform/ai-ros/tests/integration/test_ai_ros_integration.py
```

## 4.3 Bad Examples

```
# Flat structure โ€” no src/ layout
platform/ai-ros/ai_ros.py             โ missing src/ layout

# Mixed test and source
platform/ai-ros/src/test_helpers.py   โ tests do not belong in src/

# Private module imported externally
from aistudio.platform.ai_ros._internal.scheduling import SchedulingQueue   โ forbidden

# Product importing from another product
from aisf.domain.product import Product  # used in products/mythic-realms/   โ forbidden
```

## 4.4 Common Mistakes

- Placing `conftest.py` fixtures in `src/` instead of `tests/`
- Using `tests/test_*.py` flat structure instead of `tests/unit/` and `tests/integration/`
- Not creating a `tests/unit/` file for every source module (silent coverage gaps)
- Using relative imports (`from . import X`) across package boundaries

---

---

# 5. Module Naming Convention

**Rationale:** Module names are permanent. Renaming a module after it has been imported by consumers requires coordinated updates across all consumers. Get the name right the first time.

## 5.1 Rules

### R5.1 โ€” Module names are lowercase snake_case

All Python module files use `lowercase_snake_case.py`. No camelCase, no PascalCase, no hyphens in filenames.

### R5.2 โ€” Module names are nouns describing their primary responsibility

A module name answers: "What does this module contain?" It does not answer: "What does this module do?" A module named `execute_workflow.py` is a verb โ€” bad. A module named `workflow_executor.py` is a noun describing what lives there โ€” good.

### R5.3 โ€” Service classes end in `Service` or `Manager` or `Engine`, not the module name

If the module is `workflow_runtime.py`, the primary class is `WorkflowRuntime`, not `WorkflowRuntimeService`. The class name describes the concept, not the layer it occupies.

Suffix conventions:

| Suffix | When to use | Example |
|--------|-------------|---------|
| `Service` | Stateless operation orchestrator | `CostTrackingService` |
| `Manager` | Stateful resource manager | `SessionManager`, `BudgetManager` |
| `Engine` | Complex computational process | `WorkflowRuntime`, `TemplateEngine` |
| `Client` | SDK client (consumer-facing) | `AIClient`, `WorkflowClient` |
| `Provider` | AI provider adapter | `AnthropicProvider`, `OllamaProvider` |
| `Repository` | Data access layer | `WorkflowRepository`, `ProductRepository` |
| `Handler` | Event or error handler | `BudgetExceededHandler` |
| `Middleware` | ASGI middleware | `SecurityMiddleware`, `CORSMiddleware` |
| `Router` | FastAPI router (routes, not logic) | Only in `api/` directories |

### R5.4 โ€” Platform module names must not include product names

A platform module named `aisf_brain.py` or `software_factory_brain.py` violates P-1 (platform serves all products equally). Platform modules are named for their concept, not their consumer.

### R5.5 โ€” Abbreviations are forbidden unless universally understood

`ai_ros.py` โ€” acceptable (AI ROS is the established name from the blueprint)  
`prom_met.py` โ€” not acceptable (abbreviation of `prometheus_metrics`)  
`wf_rt.py` โ€” not acceptable (abbreviation of `workflow_runtime`)

Universally understood abbreviations permitted: `api`, `http`, `sdk`, `db`, `dto`, `id`, `url`.

## 5.2 Good Examples

```
workflow_runtime.py       โ’ WorkflowRuntime class
budget_manager.py         โ’ BudgetManager class
anthropic_provider.py     โ’ AnthropicProvider class
prompt_template.py        โ’ PromptTemplate class
security_middleware.py    โ’ SecurityMiddleware class
experience_recorder.py    โ’ ExperienceRecorder class (single canonical one)
```

## 5.3 Bad Examples

```
exec_wf.py                โ’ abbreviated, cryptic
executeWorkflow.py        โ’ camelCase filename
WorkflowRuntime.py        โ’ PascalCase filename
aisf_product_helper.py    โ’ product name in platform module
do_ai_stuff.py            โ’ verb phrase, non-specific noun
utils.py                  โ’ too generic (what kind of utils?)
helpers.py                โ’ too generic
common.py                 โ’ too generic (use specific names: models.py, exceptions.py)
```

## 5.4 Common Mistakes

- Creating `utils.py` files that accumulate unrelated functions (this is a design smell โ€” split into specific modules)
- Using the same module name in both `platform/` and `products/` (causes import ambiguity)
- Naming a test file `test_stuff.py` instead of mirroring the module name

---

---

# 6. Package Naming Convention

**Rationale:** Python import paths are the public interface of a package. Once used in production code, changing them requires a migration. Design them to be stable.

## 6.1 Rules

### R6.1 โ€” All platform packages share the `aistudio.platform` namespace

```
aistudio.platform.ai_ros
aistudio.platform.workflow_runtime
aistudio.platform.prompt_os
aistudio.platform.knowledge
aistudio.platform.security
aistudio.platform.messaging
aistudio.platform.persistence
aistudio.platform.storage
aistudio.platform.observability
```

### R6.2 โ€” All SDK packages share the `aistudio.sdk` namespace

```
aistudio.sdk.ai
aistudio.sdk.workflow
aistudio.sdk.prompt
aistudio.sdk.brain
aistudio.sdk.security
aistudio.sdk.workspace
aistudio.sdk.desktop
```

### R6.3 โ€” Product packages use a short product-specific namespace

```
aisf.*          โ’ ai-software-factory product
content.*       โ’ content-factory product
mythic.*        โ’ mythic-realms product
```

Products do not use the `aistudio` namespace. The `aistudio` namespace is reserved for platform and SDK.

### R6.4 โ€” Package names in `pyproject.toml` use kebab-case

The distribution package name (what you `pip install`) uses kebab-case:
- `aistudio-platform-ai-ros`
- `aistudio-sdk-ai`
- `aistudio-product-aisf`

The importable namespace uses snake_case (as above).

### R6.5 โ€” `__init__.py` exports the public API only

The package `__init__.py` contains only `__all__` and re-exports of the public classes and functions. It contains no logic. It imports nothing from `_internal`. A reader of `__init__.py` understands the complete public API of the package.

## 6.2 Good Examples

```python
# aistudio/platform/ai_ros/__init__.py
from aistudio.platform.ai_ros.service import AIROSService
from aistudio.platform.ai_ros.models import ExecutionRequest, ExecutionResult, ProviderHealth

__all__ = ["AIROSService", "ExecutionRequest", "ExecutionResult", "ProviderHealth"]
```

```python
# aistudio/sdk/ai/__init__.py
from aistudio.sdk.ai.client import AIClient
from aistudio.sdk.ai.models import ExecutionRequest, ExecutionResult

__all__ = ["AIClient", "ExecutionRequest", "ExecutionResult"]
```

## 6.3 Bad Examples

```python
# __init__.py that imports implementation details
from aistudio.platform.ai_ros._internal.scheduling import SchedulingQueue  # private!
from aistudio.platform.ai_ros.service import _build_completion_request  # private function!

# __init__.py with logic
def get_client():   # logic in __init__.py โ€” forbidden
    return AIROSService()
```

```
# Wrong namespace for a product
aistudio.product.aisf.*   # products use own namespace (aisf.*), not aistudio.*
```

## 6.4 Common Mistakes

- Adding new public classes without updating `__all__`
- Importing internal modules in `__init__.py` (leaks private API)
- Using `aistudio` namespace for product code
- Inconsistent casing between `pyproject.toml` name and import namespace

---

---

# 7. API Naming Standards

**Rationale:** The REST API is a public contract consumed by the desktop, external clients, and AI agents. Inconsistent naming creates confusion and makes client code brittle.

## 7.1 Rules

### R7.1 โ€” All API paths use kebab-case

URL path segments use `kebab-case`. No snake_case, no camelCase, no PascalCase in URLs.

### R7.2 โ€” Resource names are plural nouns

REST resources are collections. The URL identifies a member of a collection. Always plural.

```
/api/v1/products           โ’ collection
/api/v1/products/{id}      โ’ member
/api/v1/workflows          โ’ collection
/api/v1/workflows/{id}     โ’ member
```

Exception: singleton resources (e.g., a product has one `configuration`): `/api/v1/products/{id}/configuration`

### R7.3 โ€” Actions on resources use sub-resources, not verbs in the path

When an action changes resource state, model it as a sub-resource with a POST:

```
POST /api/v1/workflows/{id}/cancel     โ’ cancel a workflow
POST /api/v1/workflows/{id}/pause      โ’ pause a workflow
POST /api/v1/prompts/{id}/activate     โ’ activate a prompt (governance state change)
POST /api/v1/products/{id}/phases/{phase_id}/approve  โ’ approve a phase gate
```

Never use verbs in resource paths: `/api/v1/cancelWorkflow` is forbidden.

### R7.4 โ€” Query parameter names use snake_case

```
GET /api/v1/products?product_type=web_app&status=active&page_size=20&page_cursor=abc123
```

### R7.5 โ€” Response envelopes are consistent

All list responses:
```json
{
  "items": [...],
  "total": 142,
  "page_size": 20,
  "next_cursor": "eyJpZCI6IjEyMyJ9",
  "has_more": true
}
```

All single-resource responses: the resource object directly (no wrapper).

All error responses:
```json
{
  "error": {
    "code": "WORKFLOW_NOT_FOUND",
    "message": "Workflow abc123 does not exist",
    "request_id": "req-xyz-789",
    "timestamp": "2026-06-28T14:30:00Z",
    "details": {}
  }
}
```

### R7.6 โ€” HTTP status codes follow RFC 7231

| Status | When to use |
|--------|-------------|
| 200 OK | GET, PUT success; action completed |
| 201 Created | POST that creates a resource; include `Location` header |
| 202 Accepted | Async operation accepted (e.g., submit workflow) |
| 204 No Content | DELETE success; action with no response body |
| 400 Bad Request | Invalid request body or query params (validation error) |
| 401 Unauthorized | Missing or invalid credentials |
| 403 Forbidden | Valid credentials but insufficient permission |
| 404 Not Found | Resource does not exist |
| 409 Conflict | State conflict (e.g., workflow already cancelled) |
| 422 Unprocessable Entity | Pydantic validation failure (FastAPI default) |
| 429 Too Many Requests | Rate limit exceeded; include `Retry-After` header |
| 500 Internal Server Error | Unhandled platform error |
| 503 Service Unavailable | Platform module unavailable; include `Retry-After` |

### R7.7 โ€” All API versions use the `/api/v1/` prefix

The current version is v1. When a breaking change is introduced, v2 routes are added alongside v1. v1 routes remain functional for a minimum of 6 months after v2 launch.

### R7.8 โ€” Route files contain no business logic

FastAPI route files contain only:
- Request/response models (or imports of them from `models.py`)
- Route handler functions (thin: validate โ’ call service โ’ return response)
- Dependency injection declarations

Business logic lives in service classes. A route handler must not contain an `if` statement based on domain state. If it does, that logic belongs in a service.

## 7.2 Good Examples

```
GET    /api/v1/products                          โ’ list products
POST   /api/v1/products                          โ’ create product
GET    /api/v1/products/{product_id}             โ’ get product
DELETE /api/v1/products/{product_id}             โ’ delete product
POST   /api/v1/products/{product_id}/phases/{phase_id}/approve   โ’ approve phase
GET    /api/v1/workflows?status=active&page_size=20&page_cursor=xyz
POST   /api/v1/workflows/{workflow_id}/cancel
```

## 7.3 Bad Examples

```
GET  /api/v1/getProduct/{id}       โ’ verb in path
GET  /api/v1/product/{id}          โ’ singular resource name
POST /api/v1/cancelWorkflow        โ’ action as top-level path
GET  /api/v1/Products              โ’ PascalCase in URL
POST /api/v1/products/{id}?action=cancel   โ’ action as query param
GET  /api/v1/workflows?pageSize=20  โ’ camelCase query param
```

## 7.4 Common Mistakes

- Using `page` (integer offset) instead of `page_cursor` (opaque cursor) โ€” offset pagination breaks with concurrent inserts
- Returning 200 with `{"success": false, "error": "..."}` โ€” use proper HTTP error status codes
- Mixing sync and async handlers in the same router file
- Embedding product version in the URL path instead of using API versioning (`/api/v1/aisf/...` vs `/api/v1/...` with product-scoped resources)

---

---

# 8. DTO Standards

**Rationale:** Data Transfer Objects (DTOs) are the contracts between layers. Inconsistent DTOs cause serialization bugs, validation gaps, and ambiguous API documentation.

## 8.1 Rules

### R8.1 โ€” All DTOs are Pydantic v2 models

Every data structure crossing a layer boundary (API request/response, SDK call input/output, event payload, platform service input/output) is a Pydantic `BaseModel`. No raw `dict`, no TypedDict, no dataclass at layer boundaries.

Exception: internal computational results that do not cross a layer boundary may use dataclasses.

### R8.2 โ€” DTO naming convention

| DTO category | Suffix | Example |
|-------------|--------|---------|
| API request body | `Request` | `CreateProductRequest` |
| API response body | `Response` | `ProductResponse`, `WorkflowListResponse` |
| SDK input | `Request` | `ExecutionRequest`, `WorkflowSubmitRequest` |
| SDK output | `Result` | `ExecutionResult`, `WorkflowHandle` |
| Event payload | `Event` | `WorkflowTaskStartedEvent` |
| Platform service input | `Command` or `Query` | `DispatchTaskCommand`, `GetWorkflowQuery` |
| Config section | `Config` | `AIROSConfig`, `SecurityConfig` |

### R8.3 โ€” DTOs have explicit field types โ€” no `Any` without justification

Every field in a DTO has an explicit type annotation. The `Any` type is forbidden without an inline comment explaining why the type cannot be made specific and a `# standards: any-justified` marker that the linter tracks.

Exception: `metadata: dict[str, Any]` fields for extensible key-value bags are permitted โ€” they must be named `metadata`, `context`, or `extra` to signal their open-ended nature.

### R8.4 โ€” DTOs are immutable

Pydantic v2 DTOs are configured with `model_config = ConfigDict(frozen=True)`. DTOs are not mutated after construction. If a DTO must be updated, construct a new one (use `model.model_copy(update={...})`).

Exception: DTOs used as internal mutable state within a service (not crossing boundaries) may be mutable โ€” document this explicitly.

### R8.5 โ€” IDs are UUID, not integer

All resource identifiers are `UUID`. Exposed in API as `str` (UUID string). Integer auto-increment IDs are never exposed in the API.

```python
product_id: UUID = Field(default_factory=uuid4)
```

### R8.6 โ€” Timestamps are UTC datetime, always

All datetime fields are `datetime` with timezone. Stored as UTC. Serialized as ISO 8601 with Z suffix. No naive datetime objects.

```python
created_at: datetime = Field(default_factory=lambda: datetime.now(UTC))
```

### R8.7 โ€” Optional fields have explicit defaults

No field is implicitly optional. If a field is optional, its default is explicit:

```python
description: str | None = None           # explicitly nullable
tags: list[str] = Field(default_factory=list)   # explicitly empty list
```

## 8.2 Good Examples

```python
# API request DTO
class CreateWorkflowRequest(BaseModel):
    model_config = ConfigDict(frozen=True)

    product_id: UUID
    plan: WorkflowPlan
    priority: Literal["urgent", "standard", "batch"] = "standard"
    metadata: dict[str, Any] = Field(default_factory=dict)

# API response DTO
class WorkflowResponse(BaseModel):
    model_config = ConfigDict(frozen=True)

    workflow_id: UUID
    product_id: UUID
    status: WorkflowStatus
    created_at: datetime
    updated_at: datetime
    task_count: int
    completed_task_count: int
```

## 8.3 Bad Examples

```python
# Raw dict at layer boundary
def submit_workflow(plan: dict) -> dict:   # must be Pydantic DTOs

# Mutable DTO
class WorkflowRequest(BaseModel):
    product_id: str   # should be UUID
    status: str       # should be enum or Literal โ€” unvalidated string

# Implicit optional
class ProductRequest(BaseModel):
    name: str
    description: str  # is this optional? Unclear โ€” must be str | None = None if optional

# Integer ID exposed in API
class ProductResponse(BaseModel):
    id: int    # must be UUID
```

## 8.4 Common Mistakes

- Reusing the same DTO for both request and response (request DTOs lack auto-generated IDs and timestamps; response DTOs lack writable fields)
- Using `str` for IDs instead of `UUID` (allows invalid UUIDs through)
- Placing DTO definitions in route files instead of `models.py`
- Using `datetime.utcnow()` (deprecated, returns naive datetime) instead of `datetime.now(UTC)`

---

---

# 9. Domain Model Rules

**Rationale:** Domain models are the core of product business logic. They must be pure, testable, and independent of infrastructure (database, HTTP, AI providers).

## 9.1 Rules

### R9.1 โ€” Domain entities are pure Python objects

Domain entities contain only business rules and invariants. They have no database imports, no HTTP imports, no AI provider imports, no Pydantic dependency. They are instantiable without any infrastructure.

### R9.2 โ€” Domain entities live in `domain/` directories

Each product has a `src/<namespace>/domain/` directory. No domain entity lives outside this directory. Infrastructure representations (ORM models, API DTOs) are separate classes that map to and from domain entities.

### R9.3 โ€” Domain entities enforce their own invariants

A domain entity is always in a valid state. Invalid state is prevented at construction. If an invariant is violated, a domain exception is raised โ€” not an HTTP exception, not a database error.

### R9.4 โ€” Domain entities have no setters

Domain state transitions happen through explicit methods that name the transition:

```
workflow.cancel(reason="User requested cancellation")   โ’ valid
workflow.status = "cancelled"                            โ’ forbidden
```

### R9.5 โ€” ORM models are not domain models

SQLAlchemy ORM models are infrastructure. They live in `platform/persistence/src/models/`. Products never import ORM models directly. The persistence layer maps between ORM models and domain entities.

### R9.6 โ€” Domain events are raised by domain entities, not service classes

When a significant state change happens in a domain entity, the entity records a domain event. The service layer dispatches recorded events to the messaging platform after persisting the entity.

## 9.2 Good Examples

```python
# Good: pure domain entity with invariant enforcement
class WorkflowPlan:
    def __init__(self, product_id: UUID, tasks: list[TaskDefinition]) -> None:
        if not tasks:
            raise EmptyWorkflowPlanError("A workflow plan must have at least one task")
        if product_id is None:
            raise ValueError("product_id is required")
        self._product_id = product_id
        self._tasks = list(tasks)
        self._events: list[DomainEvent] = []

    def add_task(self, task: TaskDefinition) -> None:
        if any(t.task_id == task.task_id for t in self._tasks):
            raise DuplicateTaskError(f"Task {task.task_id} already exists in plan")
        self._tasks.append(task)
        self._events.append(TaskAddedToPlanEvent(plan=self, task=task))

    def collect_events(self) -> list[DomainEvent]:
        events = list(self._events)
        self._events.clear()
        return events
```

## 9.3 Bad Examples

```python
# Bad: domain entity with infrastructure dependency
from sqlalchemy.orm import Session
from fastapi import HTTPException

class WorkflowPlan:
    def save(self, db: Session) -> None:   # infrastructure in domain entity!
        db.add(self)
        db.commit()

    def validate(self):
        if not self.tasks:
            raise HTTPException(status_code=400, ...)   # HTTP exception in domain entity!
```

```python
# Bad: service class raises domain event (should be entity)
class WorkflowService:
    def cancel(self, workflow_id: UUID) -> None:
        workflow = self._repo.get(workflow_id)
        workflow.status = "cancelled"   # setter โ€” forbidden
        self._event_bus.publish(WorkflowCancelledEvent(...))  # event from service โ€” should be from entity
```

## 9.4 Common Mistakes

- Putting business rules in service classes (belongs in domain entities)
- Using ORM model attributes directly in business logic (ORM models are infrastructure)
- Creating "anemic domain models" โ€” data classes with all logic in services
- Mixing validation (is this data valid?) with business rules (is this transition allowed?)

---

---

# 10. Dependency Injection Standards

**Rationale:** Dependency injection makes components testable, decoupled, and configurable. Without it, singletons accumulate and tests require global state manipulation.

## 10.1 Rules

### R10.1 โ€” Platform services are injected, never imported as singletons in business logic

Service classes receive their dependencies through their constructor. No service class calls `get_workflow_runtime()` or imports a global singleton inside a method. Global singletons are permitted only in the platform bootstrapper (`app.py` or equivalent) as the composition root.

### R10.2 โ€” FastAPI dependency injection is the composition mechanism for routes

Route handlers declare service dependencies via `Depends()`. Services are not imported at module level in route files.

### R10.3 โ€” SDK clients are injected into products via constructor or FastAPI Depends

Product code that needs `AIClient`, `WorkflowClient`, or `BrainClient` receives it through DI, not by instantiating it directly.

### R10.4 โ€” Database sessions are always injected โ€” never created ad-hoc

Database sessions are provided via `Depends(get_db_session)` in routes and via constructor injection in service classes. No service class calls `SessionLocal()` inside a method.

### R10.5 โ€” The composition root (app.py) is the only place that wires dependencies

`app.py` (or the platform bootstrapper) creates all service instances and wires them together. All other code receives dependencies through constructor arguments. `app.py` is allowed to import singletons; no other module is.

### R10.6 โ€” Dependencies are declared in `__init__` with type annotations

```python
class ProductCreationService:
    def __init__(
        self,
        workflow_client: WorkflowClient,
        brain_client: BrainClient,
        prompt_client: PromptClient,
    ) -> None:
        self._workflow_client = workflow_client
        self._brain_client = brain_client
        self._prompt_client = prompt_client
```

All dependencies are stored as private attributes (leading underscore). No dependency is mutated after construction.

## 10.2 Good Examples

```python
# Route handler with DI
@router.post("/products", response_model=ProductResponse, status_code=201)
async def create_product(
    request: CreateProductRequest,
    service: ProductCreationService = Depends(get_product_creation_service),
    auth: AuthContext = Depends(require_permission(Permission.PRODUCT_CREATE)),
) -> ProductResponse:
    product = await service.create(request)
    return ProductResponse.from_domain(product)
```

```python
# FastAPI dependency factory
def get_product_creation_service(
    workflow_client: WorkflowClient = Depends(get_workflow_client),
    brain_client: BrainClient = Depends(get_brain_client),
    prompt_client: PromptClient = Depends(get_prompt_client),
) -> ProductCreationService:
    return ProductCreationService(workflow_client, brain_client, prompt_client)
```

## 10.3 Bad Examples

```python
# Global singleton used in business logic
class ProductCreationService:
    def create(self, request):
        runtime = get_workflow_runtime()   # importing singleton inside method โ€” forbidden
        runtime.submit(plan)
```

```python
# Direct DB session creation
class ExperienceRecorder:
    def record(self, experience):
        db = SessionLocal()   # forbidden โ€” must be injected
        db.add(...)
        db.commit()
        db.close()
```

```python
# Instantiating SDK client directly
class PlannerService:
    def __init__(self):
        self._ai = AIClient()   # forbidden โ€” must be injected, not instantiated here
```

## 10.4 Common Mistakes

- Using `Depends()` on a service that has its own database connection (creates N connections per request)
- Storing the database session as an instance variable on a long-lived service (session must be request-scoped)
- Using class-level attributes as mutable state (thread-safety violation)
- Injecting concrete classes instead of abstract interfaces (makes mocking impossible)

---

---

# 11. Logging Standards

**Rationale:** Structured logs are machine-readable and can be queried, alerted on, and ingested by SIEM systems. Plaintext log messages are not queryable and waste observability infrastructure.

## 11.1 Rules

### R11.1 โ€” All logs are structured JSON

Every log line is a JSON object. No plaintext log messages. The logging platform (`platform/observability/src/logging/`) provides the structured logger. All code uses the structured logger โ€” never `print()`, never raw `logging.getLogger()`.

### R11.2 โ€” Every log line has mandatory fields

| Field | Type | Example |
|-------|------|---------|
| `timestamp` | ISO 8601 UTC | `"2026-06-28T14:30:00.123Z"` |
| `level` | string | `"INFO"`, `"WARNING"`, `"ERROR"`, `"DEBUG"` |
| `logger` | string | `"aistudio.platform.workflow_runtime"` |
| `message` | string | Human-readable summary |
| `correlation_id` | UUID string | From request context |
| `module` | string | `"workflow_runtime"` |
| `function` | string | `"dispatch_task"` |

### R11.3 โ€” Log levels are used correctly

| Level | When to use |
|-------|-------------|
| DEBUG | Detailed diagnostic information โ€” never enabled in production by default |
| INFO | Normal operational events โ€” task dispatched, workflow completed, provider selected |
| WARNING | Unexpected but recoverable situation โ€” provider latency spike, budget at 80% |
| ERROR | Error that prevented an operation but platform is still running |
| CRITICAL | Error that requires immediate human intervention โ€” platform module failed to start |

### R11.4 โ€” Log messages describe events, not states

Log messages are past-tense event descriptions: "Task dispatched", "Budget limit exceeded", "Provider circuit breaker opened". Not state descriptions: "Dispatching task", "Checking budget", "Circuit breaker is open".

### R11.5 โ€” Sensitive data is never logged

The following data is never included in log output, directly or indirectly:
- API keys, JWT tokens, secrets
- Full request/response bodies from AI providers (log token counts, not content)
- User-provided prompt content (log prompt ID, not content)
- Database connection strings
- Personal data (email addresses, names)

### R11.6 โ€” Errors are logged with full context at the point of handling

An error is logged once โ€” at the point where it is handled (converted to a user response or recorded as a failure). It is not re-logged at every level of the call stack. Include: error type, message, relevant IDs (workflow_id, task_id, product_id), and the correlation_id.

### R11.7 โ€” Performance-sensitive paths use lazy log message construction

Log messages that require string formatting are only constructed if the log level is enabled. Use the structured logger's keyword argument form โ€” never f-strings in log calls.

## 11.2 Good Examples

```python
# Using the structured logger
logger = get_structured_logger(__name__)

logger.info(
    "Task dispatched to agent",
    task_id=str(task.task_id),
    workflow_id=str(task.workflow_id),
    agent_type=task.agent_type,
    priority=task.priority,
)

logger.warning(
    "Budget threshold reached",
    product_id=str(product_id),
    daily_spend_usd=daily_spend,
    daily_limit_usd=daily_limit,
    threshold_pct=0.80,
)

logger.error(
    "Provider circuit breaker opened",
    provider_id="anthropic",
    failure_count=5,
    failure_window_seconds=60,
    exc_info=exc,
)
```

## 11.3 Bad Examples

```python
# Plaintext log
print(f"Dispatching task {task_id}")   # forbidden

# Raw logger
import logging
logging.info("task done")   # not structured; missing fields

# Sensitive data in log
logger.info(f"API key validated: {api_key}")   # API key in log โ€” forbidden

# f-string in log call (eagerly evaluated regardless of level)
logger.debug(f"Request payload: {json.dumps(large_payload)}")   # expensive even if DEBUG disabled

# Re-logging at multiple levels
except Exception as e:
    logger.error(f"Error: {e}")
    raise   # let the caller log it too โ’ duplicate log entries
```

## 11.4 Common Mistakes

- Using exception `str()` as log message instead of the structured `exc_info` parameter
- Logging AI prompt content (must log prompt ID only)
- Using WARNING for expected application flow (budget exceeded is WARNING; health check is INFO)
- Not including correlation_id (makes request tracing impossible)

---

---

# 12. Error Handling Standards

**Rationale:** Inconsistent error handling produces inconsistent error responses, makes debugging difficult, and can leak internal details to external callers.

## 12.1 Rules

### R12.1 โ€” Errors are caught at the correct layer

| Layer | Responsibility |
|-------|---------------|
| Platform middleware | Catch unhandled exceptions; return structured 500 response |
| Route handlers | Catch service-layer domain exceptions; map to HTTP status codes |
| Service classes | Catch infrastructure exceptions; re-raise as domain exceptions |
| Infrastructure (DB, provider) | Raise typed infrastructure exceptions |

Errors are never swallowed (caught and not re-raised or logged). Errors are never caught and re-raised as a less specific type.

### R12.2 โ€” Every exception has a machine-readable error code

All exceptions raised by platform code carry an `error_code: str` field. Error codes are SCREAMING_SNAKE_CASE, prefixed by domain:

```
WORKFLOW_NOT_FOUND
WORKFLOW_ALREADY_CANCELLED
BUDGET_DAILY_LIMIT_EXCEEDED
PROVIDER_CIRCUIT_OPEN
PROMPT_TEMPLATE_NOT_FOUND
AUTH_INVALID_API_KEY
RATE_LIMIT_EXCEEDED
```

### R12.3 โ€” Exceptions never include sensitive data in their message

Exception messages are safe to serialize into API responses and log files. They must not include API keys, tokens, database connection strings, file paths with credentials, or user content.

### R12.4 โ€” `except Exception` is forbidden without specificity and re-raise

Bare `except:` is forbidden. `except Exception as e` is permitted only in:
- The global error handler middleware (to catch everything at the edge)
- Context managers that need to clean up resources before re-raising

All `except Exception` blocks must either re-raise the caught exception or raise a new, more specific exception wrapping the original.

### R12.5 โ€” External service failures are wrapped before crossing module boundaries

When a provider, database, or external service raises an exception, it is caught at the infrastructure layer and re-raised as a typed platform exception. No library-specific exceptions (e.g., `anthropic.APIConnectionError`, `sqlalchemy.exc.OperationalError`) cross module boundaries.

### R12.6 โ€” Async code handles cancellation

Async service methods handle `asyncio.CancelledError` correctly โ€” they must not catch and suppress it. Resources held at the time of cancellation (DB sessions, HTTP connections) must be released in a `finally` block.

## 12.2 Good Examples

```python
# Service catching infrastructure exception and re-raising as domain exception
class WorkflowRepository:
    def get(self, workflow_id: UUID) -> WorkflowPlan:
        try:
            row = self._session.get(WorkflowORM, workflow_id)
        except OperationalError as exc:
            raise PersistenceUnavailableError(
                "Database unavailable when retrieving workflow",
                workflow_id=workflow_id,
            ) from exc

        if row is None:
            raise WorkflowNotFoundError(workflow_id=workflow_id)

        return self._mapper.to_domain(row)

# Route catching domain exception and mapping to HTTP
@router.get("/workflows/{workflow_id}")
async def get_workflow(
    workflow_id: UUID,
    service: WorkflowService = Depends(get_workflow_service),
) -> WorkflowResponse:
    try:
        workflow = await service.get(workflow_id)
    except WorkflowNotFoundError:
        raise HTTPException(status_code=404, detail={"code": "WORKFLOW_NOT_FOUND", ...})
    return WorkflowResponse.from_domain(workflow)
```

## 12.3 Bad Examples

```python
# Swallowing exception
try:
    workflow = self._repo.get(workflow_id)
except Exception:
    workflow = None   # silently swallowed โ€” forbidden

# Leaking library exception across boundary
try:
    row = db.get(WorkflowORM, workflow_id)
except sqlalchemy.exc.OperationalError as exc:
    raise exc   # raw SQLAlchemy exception crossing module boundary โ€” forbidden

# Sensitive data in exception
raise AuthenticationError(f"API key {api_key} is invalid")   # API key in message โ€” forbidden

# Catching cancellation
try:
    result = await long_running_operation()
except Exception:   # catches CancelledError โ€” forbidden
    return None
```

## 12.4 Common Mistakes

- Catching `Exception` and logging without re-raising (silences the error for the caller)
- Raising `ValueError` or `RuntimeError` from platform code (must use typed domain exceptions)
- Not using `from exc` when re-raising (loses the original traceback)
- Mapping all errors to 500 in route handlers (should map to specific HTTP codes based on exception type)

---

---

# 13. Exception Hierarchy

**Rationale:** A consistent exception hierarchy allows callers to catch at the right level โ€” broadly if needed, precisely if possible. Without it, callers must catch `Exception` because they cannot predict what type to expect.

## 13.1 Hierarchy Definition

```
AiStudioError (base)
โ”โ”€โ”€ PlatformError
โ”   โ”โ”€โ”€ AIROSError
โ”   โ”   โ”โ”€โ”€ BudgetExceededError
โ”   โ”   โ”โ”€โ”€ QuotaExceededError
โ”   โ”   โ”โ”€โ”€ ProviderUnavailableError
โ”   โ”   โ”   โ””โ”€โ”€ ProviderCircuitOpenError
โ”   โ”   โ”โ”€โ”€ NoSuitableProviderError
โ”   โ”   โ””โ”€โ”€ TokenLimitExceededError
โ”   โ”โ”€โ”€ WorkflowError
โ”   โ”   โ”โ”€โ”€ WorkflowNotFoundError
โ”   โ”   โ”โ”€โ”€ WorkflowAlreadyCompletedError
โ”   โ”   โ”โ”€โ”€ WorkflowCancelledError
โ”   โ”   โ””โ”€โ”€ TaskDispatchError
โ”   โ”โ”€โ”€ PromptOSError
โ”   โ”   โ”โ”€โ”€ TemplateNotFoundError
โ”   โ”   โ”โ”€โ”€ TemplateRenderError
โ”   โ”   โ”โ”€โ”€ TemplateSignatureInvalidError
โ”   โ”   โ””โ”€โ”€ GovernanceStateError
โ”   โ”โ”€โ”€ KnowledgeError
โ”   โ”   โ””โ”€โ”€ BrainUnavailableError
โ”   โ”โ”€โ”€ SecurityError
โ”   โ”   โ”โ”€โ”€ AuthenticationError
โ”   โ”   โ”โ”€โ”€ AuthorizationError
โ”   โ”   โ””โ”€โ”€ RateLimitError
โ”   โ”โ”€โ”€ MessagingError
โ”   โ”   โ”โ”€โ”€ EventPublishError
โ”   โ”   โ””โ”€โ”€ NATSUnavailableError
โ”   โ””โ”€โ”€ PersistenceError
โ”       โ”โ”€โ”€ PersistenceUnavailableError
โ”       โ””โ”€โ”€ OptimisticLockError
โ”
โ”โ”€โ”€ ProductError (base for all product-specific exceptions)
โ”   โ”โ”€โ”€ AISFError (ai-software-factory)
โ”   โ”   โ”โ”€โ”€ ProductNotFoundError
โ”   โ”   โ”โ”€โ”€ InvalidProductSpecError
โ”   โ”   โ””โ”€โ”€ PhasGateRejectedError
โ”   โ”โ”€โ”€ ContentFactoryError
โ”   โ””โ”€โ”€ MythicRealmsError
โ”
โ””โ”€โ”€ ConfigurationError
    โ”โ”€โ”€ MissingRequiredConfigError
    โ””โ”€โ”€ InvalidConfigValueError
```

## 13.2 Rules

### R13.1 โ€” All platform exceptions inherit from `PlatformError`

All product exceptions inherit from `ProductError`. Both inherit from `AiStudioError`. No platform code raises `ProductError` subclasses. No product code raises `PlatformError` subclasses (products catch platform errors and re-raise as product errors if context enrichment is needed).

### R13.2 โ€” Every exception carries structured context

All exceptions carry structured context as keyword arguments, not interpolated into the message string:

```python
class WorkflowNotFoundError(WorkflowError):
    error_code = "WORKFLOW_NOT_FOUND"

    def __init__(self, workflow_id: UUID) -> None:
        super().__init__(
            message=f"Workflow {workflow_id} does not exist",
            workflow_id=str(workflow_id),
        )
```

The base class stores all keyword arguments as a `context` dict for structured logging.

### R13.3 โ€” HTTP status mapping is defined on the exception class

The exception class declares its intended HTTP status code. Route handlers use this mapping rather than hardcoding status codes in route files:

```python
class WorkflowNotFoundError(WorkflowError):
    http_status = 404
    error_code = "WORKFLOW_NOT_FOUND"
```

### R13.4 โ€” Exception modules live in `exceptions.py` within each package

Every platform module and product has an `exceptions.py` file containing all exceptions for that module. Exceptions are never defined in `__init__.py`, service files, or route files.

## 13.2 Good Examples

```python
# Raising with context
raise WorkflowNotFoundError(workflow_id=workflow_id)

raise BudgetExceededError(
    product_id=product_id,
    daily_spend_usd=147.83,
    daily_limit_usd=150.00,
)

raise ProviderCircuitOpenError(
    provider_id="anthropic",
    state_duration_seconds=45,
    retry_after_seconds=15,
)
```

## 13.3 Bad Examples

```python
# Raising generic exceptions
raise ValueError(f"Workflow {workflow_id} not found")   # not typed
raise RuntimeError("Budget exceeded")                    # no context

# Exception with no context
raise WorkflowNotFoundError()   # missing workflow_id โ€” makes debugging impossible

# Wrong exception type
raise PlatformError("Product not found")   # too generic; use ProductNotFoundError
```

## 13.4 Common Mistakes

- Defining exceptions in the file where they are raised (should be in `exceptions.py`)
- Not inheriting from the correct base (product code raising `PlatformError`)
- Using `str` exception arguments instead of structured keyword arguments
- Not providing `error_code` โ€” makes programmatic error handling impossible for clients

---

---

# 14. Configuration Standards

**Rationale:** Configuration that is hardcoded, scattered, or inconsistently typed produces deployment failures, security vulnerabilities, and environments that cannot be reproduced.

## 14.1 Rules

### R14.1 โ€” All configuration uses Pydantic Settings

Every configuration value is declared in a Pydantic `BaseSettings` subclass. No configuration is read via `os.environ.get()` directly in application code. No configuration is hardcoded as a module-level constant in any file other than the settings module.

### R14.2 โ€” Configuration is hierarchical and namespaced

Platform configuration follows the schema defined in the Blueprint (Section 6.3). Environment variable names are prefixed by `AISF_` for platform-wide settings. Sub-section variables are further prefixed:

```
AISF_DATABASE_URL=postgresql+asyncpg://...
AISF_AI_ROS__DEFAULT_PROVIDER=claude_code
AISF_SECURITY__JWT_EXPIRY_SECONDS=3600
AISF_WORKFLOW__STALL_THRESHOLD_SECONDS=1800
```

Double underscore (`__`) separates nested config levels (Pydantic Settings convention).

### R14.3 โ€” Security-sensitive settings have no default values

Settings that represent secrets or security parameters must have no default value. Pydantic raises a `ValidationError` at startup if they are not provided. This enforces the fail-fast principle (P-9).

```python
class SecurityConfig(BaseModel):
    api_key: str   # No default โ€” startup fails if not set
    jwt_secret: str   # No default
    cors_origins: list[str]   # No default โ€” must be explicitly configured
```

### R14.4 โ€” All settings have descriptions

Every settings field has a `Field(description="...")` that explains its purpose. This description appears in the auto-generated configuration documentation.

### R14.5 โ€” Settings are validated at startup, not at first use

The root settings object is constructed at application startup (during `lifespan`). If validation fails, startup fails loudly. No setting is read lazily inside a request handler.

### R14.6 โ€” No hardcoded values in application code

The following are **always** read from configuration:
- Database URLs and file paths
- API keys and secrets
- Hostnames, ports, base URLs
- Timeout values
- Rate limits and budgets
- Feature flags
- Provider names and model names

Exception: immutable protocol constants (e.g., HTTP status codes, JWT algorithm names like `"HS256"`) may be constants if they are not deployment-variable.

### R14.7 โ€” Environment-specific configuration is via `.env` files, not committed code

`.env` files for `local`, `staging`, and `production` environments are maintained by the operations team. They are never committed to the repository. `.env.example` with documented placeholder values (never real values) is committed.

## 14.2 Good Examples

```python
# Platform configuration
class AIROSConfig(BaseModel):
    model_config = ConfigDict(env_prefix="AISF_AI_ROS__")

    default_provider: str = Field(
        default="claude_code",
        description="Default AI provider when no preference specified"
    )
    max_concurrent_requests: int = Field(
        default=10,
        ge=1,
        le=100,
        description="Maximum concurrent AI execution requests across all queues"
    )
    budget_daily_limit_usd: float = Field(
        default=50.0,
        gt=0,
        description="Global daily AI spend limit in USD. Platform rejects requests beyond this."
    )
    stall_threshold_seconds: int = Field(
        default=1800,
        ge=60,
        description="Seconds before an in-progress task is considered stalled"
    )
```

## 14.3 Bad Examples

```python
# Hardcoded value โ€” found in current app.py
init_memory_service(db_url="sqlite:///memory.db")   # forbidden โ€” must be config

# Direct os.environ access
import os
api_key = os.environ.get("API_KEY", "")   # forbidden โ€” must use Pydantic Settings

# Default that disables security
class SecurityConfig(BaseModel):
    api_key: str = ""   # empty default disables auth โ€” catastrophic default
```

## 14.4 Common Mistakes

- Providing empty string defaults for required security settings (disables security silently)
- Using `os.getenv()` instead of Pydantic Settings in utility modules
- Different environment variable names for the same setting in different files
- Not validating types (accepting `str` instead of `int` for timeout values)

---

---

# 15. Secrets Management

**Rationale:** Secrets that appear in code, logs, or configuration files become compromised the moment those files are shared. Secret management is not optional security hardening โ€” it is baseline security.

## 15.1 Rules

### R15.1 โ€” Secrets are never committed to the repository

Secrets include: API keys, JWT signing secrets, database passwords, private keys, OAuth client secrets, any credential. A secret committed to git is permanently compromised (git history is permanent). This rule has no exceptions.

**Detection:** The repository has a pre-commit hook that runs `detect-secrets scan`. If potential secrets are detected, the commit is rejected. The hook is configured in `.pre-commit-config.yaml` and cannot be bypassed without explicit `--no-verify` (which triggers a CI failure and requires justification).

### R15.2 โ€” Secrets are provided exclusively via environment variables

Application code reads secrets only from environment variables via Pydantic Settings. No secrets in:
- Configuration files checked into git
- Docker image layers
- Command-line arguments (visible in process list)
- Log files

### R15.3 โ€” Secrets in memory are not logged or serialized

Pydantic Settings fields that contain secrets are annotated with `Field(repr=False)` so they are not printed in `__repr__` output. Secret values never appear in log lines, error messages, or API responses.

### R15.4 โ€” API keys are stored as hashed values in the database

The `api_keys` table stores the SHA-256 hash of the API key, not the plaintext key. The plaintext key is returned only once at creation time. Subsequent operations use `hmac.compare_digest(hash(provided_key), stored_hash)`.

### R15.5 โ€” API key comparison is always timing-safe

All credential comparisons use `hmac.compare_digest()`. Direct equality comparison (`==`, `!=`) of credential strings is forbidden. This prevents timing attacks.

### R15.6 โ€” Secret rotation is documented and tested

Every secret has a documented rotation procedure. The rotation procedure is tested in the staging environment quarterly. Key fields:
- How to generate a new secret
- How to deploy the new secret without service interruption
- How to verify the rotation was successful
- How to roll back if the new secret fails

## 15.2 Good Examples

```python
# Pydantic Settings with secret field
class SecurityConfig(BaseModel):
    model_config = ConfigDict(env_prefix="AISF_SECURITY__")

    api_key: str = Field(
        repr=False,   # never printed
        description="Primary API key for authentication. No default โ€” must be set."
    )
    jwt_secret: str = Field(
        repr=False,
        description="HS256 signing secret for JWT tokens. Minimum 32 characters."
    )
```

```python
# Timing-safe comparison
import hmac

def verify_api_key(provided_key: str, stored_hash: str) -> bool:
    provided_hash = hashlib.sha256(provided_key.encode()).hexdigest()
    return hmac.compare_digest(provided_hash, stored_hash)
```

## 15.3 Bad Examples

```python
# Secret in code
ANTHROPIC_API_KEY = "sk-ant-api03-abc123..."   # forbidden

# Direct equality comparison (timing attack)
if provided_key != stored_key:   # forbidden
    raise AuthenticationError(...)

# Secret in log
logger.info(f"Validating API key: {api_key}")   # forbidden

# Secret in error message
raise AuthenticationError(f"Key {api_key} not found in database")   # forbidden
```

## 15.4 Common Mistakes

- Checking in `.env` files (even for local development โ€” use `.env.example` instead)
- Using `os.environ.get("KEY", "fallback_key")` with a hardcoded fallback
- Logging request headers without redacting `X-API-Key` and `Authorization`
- Not rotating secrets after a team member leaves

---

---

# 16. Event Naming Standards

**Rationale:** Events are the contracts between platform modules and between products. An event name is never changed after consumers exist โ€” like a database column name, it is permanent.

## 16.1 Rules

### R16.1 โ€” Event subjects follow a three-part hierarchy

```
<domain>.<entity>.<verb>
```

| Segment | Rules | Examples |
|---------|-------|---------|
| `<domain>` | Platform domain or product namespace | `workflows`, `ai`, `prompts`, `brain`, `security`, `aisf` |
| `<entity>` | The entity that changed state | `task`, `workflow`, `execution`, `template`, `pattern` |
| `<verb>` | Past-tense action | `started`, `completed`, `failed`, `created`, `approved` |

### R16.2 โ€” Domain names are plural

The domain segment of an event subject is always plural: `workflows.*`, `ai.*`, `prompts.*`, `users.*`.

This is the fix to the current bug where `workflow_runtime.py` publishes to `workflow.*` (singular) but the NATS streams are defined for `workflows.*` (plural). The plural form is canonical.

### R16.3 โ€” Verbs are past tense

Events describe things that happened, not things that are happening:
- `workflows.task.started` โ€” not `workflows.task.starting`
- `ai.execution.completed` โ€” not `ai.execution.complete`
- `security.auth.failed` โ€” not `security.auth.failure`

### R16.4 โ€” The complete event catalog is defined in the SDK

`sdk/messaging/src/aistudio/sdk/messaging/events.py` contains the complete catalog of all platform event subjects as typed constants. Products do not define their own event subject strings โ€” they import from the SDK catalog.

### R16.5 โ€” Event payloads are versioned Pydantic models

Every event has a versioned payload model: `WorkflowTaskStartedEvent` with `schema_version: str = "1.0"`. When a payload changes, the schema version increments. Consumers check the schema version before processing.

### R16.6 โ€” All events carry mandatory envelope fields

```python
class PlatformEvent(BaseModel):
    event_id: UUID = Field(default_factory=uuid4)
    event_type: str       # the full subject, e.g. "workflows.task.started"
    source: str           # module path, e.g. "platform/workflow-runtime"
    correlation_id: UUID | None = None
    causation_id: UUID | None = None
    timestamp: datetime = Field(default_factory=lambda: datetime.now(UTC))
    schema_version: str = "1.0"
    payload: dict[str, Any]
```

## 16.2 Complete Platform Event Catalog

| Subject | Description | Publisher |
|---------|-------------|---------|
| `workflows.task.started` | Task began execution | workflow-runtime |
| `workflows.task.completed` | Task completed successfully | workflow-runtime |
| `workflows.task.failed` | Task failed with error | workflow-runtime |
| `workflows.task.stalled` | Task stalled beyond threshold | workflow-runtime |
| `workflows.task.approved` | Human approval gate passed | approval-service |
| `workflows.task.rejected` | Human approval gate rejected | approval-service |
| `workflows.workflow.created` | New workflow submitted | workflow-runtime |
| `workflows.workflow.completed` | All tasks completed | workflow-runtime |
| `workflows.workflow.cancelled` | Workflow cancelled | workflow-runtime |
| `ai.execution.started` | AI provider call initiated | ai-ros |
| `ai.execution.completed` | AI provider call returned | ai-ros |
| `ai.execution.failed` | AI provider call failed | ai-ros |
| `ai.provider.circuit_opened` | Circuit breaker tripped | ai-ros |
| `ai.provider.circuit_closed` | Circuit breaker recovered | ai-ros |
| `ai.budget.threshold_reached` | Spend at warning threshold | ai-ros |
| `ai.budget.limit_exceeded` | Spend at hard limit | ai-ros |
| `prompts.template.activated` | Template moved to ACTIVE | prompt-os |
| `prompts.template.deprecated` | Template deprecated | prompt-os |
| `brain.pattern.discovered` | New pattern found by learning agent | knowledge |
| `brain.experience.recorded` | New experience recorded | knowledge |
| `security.auth.failed` | Authentication failure | security |
| `security.rate_limit.exceeded` | Rate limit triggered | security |

## 16.3 Bad Examples

```
# Singular domain (wrong)
workflow.task.started      โ’ workflows.task.started

# Present tense verb (wrong)
workflows.task.starting    โ’ workflows.task.started

# Verb first (wrong)
started.workflows.task     โ’ workflows.task.started

# Product name in platform event (wrong)
aisf.workflow.task.started โ’ workflows.task.started  (platform events have no product prefix)

# Too vague
system.event               โ’ must be specific
```

## 16.4 Common Mistakes

- Publishing to a subject that matches no NATS stream (silently dropped โ€” not an error)
- Changing an event payload shape without incrementing `schema_version`
- Using `"workflow.*"` wildcard subscription when intending `"workflows.*"` (one-character bug that silently misses all events)
- Not including `correlation_id` (breaks distributed trace correlation)

---



---

---

# 17. Versioning Policy

**Rationale:** Version numbers communicate intent to consumers. Breaking changes without version increments destroy trust and cause production incidents. A clear versioning policy eliminates surprise.

## 17.1 Rules

### R17.1 โ€” All packages follow Semantic Versioning 2.0.0

`MAJOR.MINOR.PATCH`

| Increment | When |
|-----------|------|
| MAJOR | Incompatible API change โ€” callers must update |
| MINOR | New backward-compatible capability |
| PATCH | Backward-compatible bug fix |

### R17.2 โ€” What constitutes a breaking change

The following are breaking changes that require a MAJOR increment:

**API breaking changes:**
- Removing an endpoint
- Changing an endpoint path
- Removing a request or response field
- Changing a field type to an incompatible type
- Adding a required request field (clients that don't send it will fail)
- Changing HTTP status codes (callers may branch on them)

**SDK breaking changes:**
- Removing a method from a public SDK class
- Changing a method signature (parameter names, types, order)
- Removing a public class
- Changing the type of a return value

**Event breaking changes:**
- Removing an event field from a payload
- Changing the type of an event payload field
- Renaming an event subject

**Database breaking changes:**
- Dropping a column
- Renaming a column
- Changing a column's type to an incompatible type
- Removing a table

### R17.3 โ€” Breaking changes require a migration path

A MAJOR version increment must be accompanied by:
1. Documentation of what changed and why
2. A migration guide for all consumers
3. A compatibility period (minimum 2 minor releases) where both old and new versions coexist
4. Automated tests verifying the old interface still works during the compatibility period

### R17.4 โ€” Platform, SDK, and products version independently

The platform layer at v1.3.0 and the aisf product at v2.1.0 are independent version numbers. An SDK at v1.0.0 may consume platform at v1.3.0. There is no requirement that version numbers align across layers.

### R17.5 โ€” Pre-release versions use the `-alpha`, `-beta`, `-rc` suffix

```
2.0.0-alpha.1    โ’ internal development, no stability guarantee
2.0.0-beta.1     โ’ feature complete, API subject to change
2.0.0-rc.1       โ’ release candidate, API frozen
2.0.0            โ’ stable release
```

### R17.6 โ€” Version is the single source of truth in `pyproject.toml`

The version is declared once in `pyproject.toml`. It is not duplicated in `__init__.py`, `version.py`, or any other file. If the version is needed at runtime, it is read from the package metadata:

```python
from importlib.metadata import version
__version__ = version("aistudio-platform-ai-ros")
```

## 17.2 Good Examples

```
# Adding a new optional field to a response โ’ MINOR
v1.2.0 โ’ v1.3.0

# Fixing a bug in budget calculation โ’ PATCH
v1.3.0 โ’ v1.3.1

# Removing the /api/v1/workflows/legacy endpoint โ’ MAJOR
v1.3.1 โ’ v2.0.0
(with 2-minor-release compatibility period where the endpoint returns a deprecation warning)

# Adding a new required field to CreateWorkflowRequest โ’ MAJOR
v1.3.1 โ’ v2.0.0
(with migration guide showing how to add the new field)
```

## 17.3 Bad Examples

```
# Breaking change shipped as PATCH
v1.3.0 โ’ v1.3.1  (removed a response field โ€” should be v2.0.0)

# No migration guide for MAJOR
v1.3.1 โ’ v2.0.0  (no changelog, no migration guide โ€” consumers discover breakage in production)

# Version in multiple places
# pyproject.toml: version = "2.0.0"
# __init__.py: __version__ = "2.0.0"
# version.py: VERSION = "2.0.1"    โ drift โ€” which is correct?
```

## 17.4 Common Mistakes

- Treating "internal" changes as non-breaking when they affect behavior observable by consumers
- Shipping MAJOR changes without notifying product teams in advance
- Not tagging git releases (version number in pyproject.toml with no git tag is unverifiable)

---

---

# 18. Feature Flag Policy

**Rationale:** Feature flags enable gradual rollouts, A/B testing, and safe rollback of in-progress features. Without a policy, flags accumulate, are never cleaned up, and become permanent source of complexity.

## 18.1 Rules

### R18.1 โ€” Feature flags are used for three purposes only

1. **Gradual rollout:** A new feature that is deployed but not yet enabled for all users
2. **Safe migration:** A code path switch during migration (old implementation vs. new)
3. **Emergency kill switch:** Ability to disable a specific feature without a deployment

Feature flags are not used for:
- Long-lived product configuration (use configuration settings instead)
- Separating test environments from production (use environment-specific config)
- A/B testing product features (use Prompt OS A/B testing for prompt experiments)

### R18.2 โ€” Every feature flag has an expiry date

When a feature flag is created, its expiry date is set. The expiry date is when the flag should be removed โ€” it is not the date the flag is disabled. A flag with no expiry date is not merged.

Typical expiry periods:
- Gradual rollout flags: 2-4 weeks after 100% rollout
- Migration flags: When migration is complete and verified
- Kill switch flags: When the underlying issue is resolved

### R18.3 โ€” Feature flags are declared in a central registry

All active feature flags are declared in `platform/config/src/aistudio/platform/config/feature_flags.py`. No feature flag is declared inline in application code with a bare `os.environ.get("SOME_FLAG")`.

### R18.4 โ€” Feature flags are boolean only

Feature flags are `True` or `False`. No feature flag is a string with multiple values (use configuration for multi-value options). No feature flag is a float (use configuration for percentages).

Exception: Prompt OS A/B test split percentages are managed by Prompt OS, not by feature flags.

### R18.5 โ€” Dead flags are removed by their expiry date

A feature flag past its expiry date is technical debt. Removing it is a code improvement, not optional cleanup. The CI pipeline warns when a flag is past its expiry date. After 2 weeks past expiry, CI fails.

### R18.6 โ€” The default value of a feature flag is the safe production state

If a service starts with no feature flag configuration (e.g., the environment variable is not set), the flag defaults to the current production behavior. A flag that defaults to enabling an untested feature is dangerous.

## 18.2 Feature Flag Registry Format

```python
# platform/config/src/aistudio/platform/config/feature_flags.py

@dataclass(frozen=True)
class FeatureFlag:
    name: str
    description: str
    default: bool           # Safe production default
    owner: str              # Team or person responsible
    created: date
    expires: date           # When to remove this flag
    ticket: str             # Link to tracking ticket

FEATURE_FLAGS: dict[str, FeatureFlag] = {
    "WORKFLOW_REAL_DISPATCH": FeatureFlag(
        name="WORKFLOW_REAL_DISPATCH",
        description="Enable real WorkflowRuntime dispatch. False = legacy simulation path.",
        default=False,
        owner="platform-team",
        created=date(2026, 7, 1),
        expires=date(2026, 9, 1),
        ticket="AISF-203",
    ),
    "AI_ROS_BUDGET_ENFORCEMENT": FeatureFlag(
        name="AI_ROS_BUDGET_ENFORCEMENT",
        description="Enable BudgetManager hard stops on AI spend.",
        default=False,
        owner="platform-team",
        created=date(2026, 8, 1),
        expires=date(2026, 10, 1),
        ticket="AISF-251",
    ),
}
```

## 18.3 Good Examples

```python
# Reading a feature flag
from aistudio.platform.config.feature_flags import FEATURE_FLAGS, is_enabled

if is_enabled("WORKFLOW_REAL_DISPATCH"):
    handle = workflow_client.submit(plan)
else:
    handle = legacy_simulation_service.simulate(plan)
```

## 18.4 Bad Examples

```python
# Inline flag without registry
if os.environ.get("USE_NEW_WORKFLOW", "false").lower() == "true":   # no registry entry

# Flag as string (multi-value)
mode = os.environ.get("WORKFLOW_MODE", "simulation")   # should be configuration, not a flag

# No expiry โ€” permanent flag
# (no expires field โ’ CI warning, then CI failure)

# Default that enables untested code
FeatureFlag(
    name="EXPERIMENTAL_BRAIN_V2",
    default=True,   # dangerous โ€” enables experimental code by default
    ...
)
```

## 18.5 Common Mistakes

- Creating a flag to hide a feature that was already deployed to production (too late for a flag)
- Using the same flag for rollout AND kill switch (different lifecycles โ€” use separate flags)
- Not removing the flag after the expiry date (accumulates and becomes permanent config)

---

---

# 19. Database Migration Standards

**Rationale:** Database migrations are irreversible operations on production data. A bad migration can destroy data, lock tables for minutes, or corrupt the schema in ways that require manual recovery. Migrations must be treated with more care than application code.

## 19.1 Rules

### R19.1 โ€” All schema changes use Alembic migrations

No schema change is made by any means other than an Alembic migration: not by `CREATE TABLE` in a script, not by ORM `create_all()` in production, not by hand in a database console. Every change has a migration file with a downgrade path.

### R19.2 โ€” Migrations are atomic and single-purpose

Each migration file changes one thing: one new table, one new column, one new index. Migrations that change multiple unrelated things are split. A migration that creates a table AND adds an index to an existing table is two migrations.

Exception: When a migration creates a new table and immediately adds indexes to it, those are one migration โ€” they are logically inseparable.

### R19.3 โ€” Every migration has a downgrade function

`downgrade()` reverses exactly what `upgrade()` did. A migration with a `pass` in `downgrade()` is not merged unless the operation is inherently irreversible (e.g., data migrations โ€” mark these explicitly with a comment: `# Data migration โ€” downgrade would cause data loss. Intentionally incomplete.`).

### R19.4 โ€” Migrations never modify tables from a previous migration's scope

A migration must not `ALTER` tables created in a different migration to add things that were missed. The first migration to create the table is the canonical definition. If something was missed, a new migration adds it. The current codebase has a violation in migration 0012 (modifies 0011's tables) โ€” this is technical debt to be resolved.

### R19.5 โ€” Migrations are tested before merging

Every migration must be run in both directions (upgrade and downgrade) in a CI environment with the production schema before merging. The CI pipeline runs:
```
alembic upgrade head
alembic downgrade -1
alembic upgrade head
```
A migration that fails any of these three steps is not merged.

### R19.6 โ€” Large table migrations are safe for concurrent use

For tables with more than 100,000 rows:
- Adding a column: use `ALTER TABLE ... ADD COLUMN ... DEFAULT NULL` (null default; backfill separately)
- Adding an index: use `CREATE INDEX CONCURRENTLY` (does not block reads or writes)
- Adding a NOT NULL column: must have a default value; backfill in a separate step

### R19.7 โ€” Migration file naming is sequential and descriptive

```
0001_initial_schema.py
0002_add_workflow_instances.py
0013_add_api_keys_table.py
0014_add_critical_query_indexes.py
```

Numbers are zero-padded to 4 digits. Names are descriptive. The number is the only dependency indicator โ€” never rely on alphabetic ordering.

### R19.8 โ€” No cross-migration dependencies in SQL

A migration must not reference the content of another migration's `upgrade()` function. If migration 0014 needs to know about the `tasks` table structure, it reads the current schema state, not migration 0002's code.

### R19.9 โ€” Data migrations are separate from schema migrations

A migration that changes schema (DDL) must not also migrate data (DML) in the same file. Schema migrations run fast and can be rolled back. Data migrations may take minutes on large tables and require different rollback strategies.

## 19.2 Good Examples

```python
# 0014_add_task_status_index.py
"""Add index on tasks.status for orchestration tick performance

Revision ID: 0014
Revises: 0013
Create Date: 2026-07-01
"""

from alembic import op

def upgrade() -> None:
    op.create_index(
        "idx_tasks_status",
        "tasks",
        ["status"],
        postgresql_using="btree",
    )

def downgrade() -> None:
    op.drop_index("idx_tasks_status", table_name="tasks")
```

## 19.3 Bad Examples

```python
# Cross-migration ALTER โ€” forbidden
def upgrade() -> None:
    # Modifying a column that was created in migration 0011
    op.alter_column("tasks", "assigned_agent", nullable=True)   # should be in 0012 or a new migration

# No downgrade
def downgrade() -> None:
    pass   # forbidden unless explicitly marked as intentional data migration

# Schema + data in one migration
def upgrade() -> None:
    op.add_column("tasks", sa.Column("priority_score", sa.Float(), nullable=True))
    # Backfill data in same migration โ€” forbidden
    op.execute("UPDATE tasks SET priority_score = 5.0 WHERE priority_score IS NULL")

# Lock-unsafe index on large table
def upgrade() -> None:
    op.create_index("idx_tasks_status", "tasks", ["status"])
    # Missing postgresql_concurrently=True for large table
```

## 19.4 Common Mistakes

- Running `alembic stamp head` to "fix" a migration mismatch without understanding what was skipped
- Forgetting to update `alembic.ini` when changing database URLs for testing
- Creating the migration file without running `alembic upgrade head` locally to verify it works
- Not considering the downgrade path for a NOT NULL constraint (requires a default or backfill before the constraint)

---

---

# 20. Documentation Standards

**Rationale:** Documentation that is inaccurate is worse than no documentation โ€” it misleads readers. Documentation that is accurate but unreadable is unused. The standard prioritizes accuracy and liveness over comprehensiveness.

## 20.1 Rules

### R20.1 โ€” Code is the primary documentation

Well-named identifiers, clear module structures, and typed interfaces are the primary documentation. If the code requires a comment to explain what it does, the code should be renamed or restructured. Comments explain **why**, not **what**.

### R20.2 โ€” Comments explain non-obvious decisions, not obvious actions

Permitted comment content:
- A constraint that is not visible from the code: `# SHA-256 required here by FIPS 140-2 compliance`
- A non-obvious invariant: `# Must check circuit breaker BEFORE quota โ€” quota updates are not rolled back on circuit trip`
- A known limitation: `# Jaccard similarity is O(nยฒ) โ€” replace with Qdrant when project count exceeds 1000`
- A workaround for a specific bug: `# anthropic SDK 0.28+ changed stream chunk format โ€” this normalizes it`

Forbidden comment content:
- What the code does: `# Get the workflow from the database` (the code already says this)
- Who wrote it or when: `# Added by John, 2026-06-15` (git blame has this)
- What the task was: `# Fixed as part of AISF-142` (belongs in commit message, not code)
- Commented-out code (delete it โ€” git history preserves it)

### R20.3 โ€” Public API methods have a single-line docstring when non-obvious

A single line that answers: "What does calling this do?" Docstrings are not required for methods whose name and type signature are fully self-explanatory. Docstrings are required when the behavior is non-obvious from the name.

```python
def estimate_cost(self, request: CompletionRequest) -> CostEstimate:
    """Return a cost estimate without making any API calls."""
```

No multi-paragraph docstrings. No parameter lists (the type annotations provide this). No return value descriptions (the return type annotation provides this).

### R20.4 โ€” Architecture Decision Records are mandatory for significant decisions

Any decision that affects platform structure, inter-module contracts, data models, or security must have an ADR in `docs/architecture/`. ADRs use the format established in the Blueprint (Context โ’ Decision โ’ Consequences โ’ Rejected Alternatives).

ADRs are immutable once ratified. If a decision changes, a new ADR supersedes the old one โ€” the old ADR is marked "Superseded by ADR-XXX", not deleted.

### R20.5 โ€” README files follow a strict template

Every package directory has a `README.md` with exactly these sections:
1. **Purpose** โ€” one paragraph: what problem does this package solve?
2. **Installation** โ€” how to install the package
3. **Quick Start** โ€” one working example (the simplest useful thing a caller can do)
4. **Configuration** โ€” list of all configuration settings with types and defaults
5. **API Reference** โ€” link to generated API docs or inline for small packages
6. **Testing** โ€” how to run the tests for this package

No "About", no "Contributing" (covered by root-level CONTRIBUTING.md), no "License" (covered by root-level LICENSE).

### R20.6 โ€” OpenAPI specs are auto-generated and not hand-edited

The OpenAPI specification for all API routes is generated by FastAPI from the route definitions and Pydantic models. No hand-edited `openapi.yaml` files. If the generated spec is wrong, fix the route definition or model โ€” not the spec file.

### R20.7 โ€” CHANGELOG.md tracks user-facing changes

Every package has a `CHANGELOG.md` that records user-facing changes for each release. Internal refactoring that does not change behavior is not logged. Breaking changes are marked with `**BREAKING**`. The format is reverse-chronological (newest first).

## 20.2 Good Examples

```python
# Good: explains WHY, not WHAT
# Using RLock instead of Lock here because this method is called
# from within a callback that the lock-holder also triggers โ€” see deadlock analysis in ADR-012
self._lock = threading.RLock()
```

```python
# Good: single-line docstring for non-obvious method
def compose(self, part_ids: list[str], variables: dict[str, Any]) -> str:
    """Render multiple prompt templates and concatenate them in order."""
```

## 20.3 Bad Examples

```python
# Bad: explains WHAT (the code already says this)
# Get the database session and query for the workflow
session = self._session
workflow = session.get(WorkflowORM, workflow_id)

# Bad: commented-out code
# result = old_ai_router.route(request)   # old implementation
result = ai_ros_client.execute(request)

# Bad: author and ticket reference in code
# Author: Jane Smith | AISF-142 | 2026-06-14
def dispatch_task(self, task_id: UUID) -> None:

# Bad: multi-line docstring with parameter list
def estimate_cost(self, request: CompletionRequest) -> CostEstimate:
    """
    Estimate the cost of executing a completion request.

    Args:
        request: The CompletionRequest containing model and token information.

    Returns:
        A CostEstimate with prompt_tokens, completion_tokens, and cost_usd fields.

    Note:
        This method does not make any API calls.
    """
```

## 20.4 Common Mistakes

- Writing docstrings that duplicate the type annotations (`Args: request (CompletionRequest)` โ€” already in the signature)
- Letting ADRs fall out of date (an ADR that describes a decision that was later reversed is misleading)
- Using README as the only documentation (it covers the package; the architecture docs cover the system)

---

---

# 21. Testing Standards

**Rationale:** Tests that pass but do not verify behavior provide false confidence. Tests that are slow, flaky, or require complex setup are not run consistently. The standard optimizes for tests that are fast, reliable, and genuinely verify behavior.

## 21.1 Rules

### R21.1 โ€” Tests are classified by scope

| Type | What it tests | Speed | External dependencies |
|------|--------------|-------|----------------------|
| Unit | Single class or function, all dependencies mocked | < 50ms each | None |
| Integration | Multiple real classes collaborating, test database | < 5s each | Test DB only |
| E2E | Full stack from API call to observable outcome | < 60s each | Real services |
| Security | Specific security control behaviors | < 5s each | None or test DB |

### R21.2 โ€” Unit tests isolate from infrastructure

Unit tests have zero external dependencies. They do not connect to databases, call AI providers, publish to NATS, or read from the filesystem. All dependencies that touch infrastructure are mocked.

The existing `StaticPool` pattern for SQLite in-memory testing is the correct model for tests that need a database schema but not a real database engine. The `patch_db` pattern from the Prompt OS test suite is the canonical example of correct DB mocking.

### R21.3 โ€” The test doubles hierarchy

| Technique | When to use |
|-----------|------------|
| Fake (in-memory implementation) | When a real implementation is too slow but you need realistic behavior |
| Stub (predetermined return value) | When you need to control what a dependency returns |
| Mock (verify call was made) | When you need to verify a side effect occurred |
| Spy (real object + call verification) | Rarely โ€” only when you cannot inject a test double |

Prefer fakes over stubs over mocks. A test that only asserts `mock.assert_called_once_with(...)` without asserting on behavior may pass even when the system is broken.

### R21.4 โ€” Tests follow Arrange-Act-Assert structure

Every test is structured with three clearly separated phases:
1. **Arrange:** Set up the system under test and its dependencies
2. **Act:** Call the code under test (exactly one action per test)
3. **Assert:** Verify the outcome (state or event)

No test has two Act phases. If you find yourself calling the code twice in one test, split it into two tests.

### R21.5 โ€” Every public API route has a security test

Every route that requires authentication must have a test that verifies:
- A request without credentials returns 401
- A request with credentials but wrong permission returns 403
- A request with valid credentials and correct permission returns 200

Security tests live in `tests/security/test_route_auth.py`.

### R21.6 โ€” Every list endpoint has a pagination test

Every API route that returns a list must have a test that verifies:
- A request with `page_size=1` returns exactly 1 item and a `next_cursor`
- A subsequent request with the `next_cursor` returns the next item
- A request with no items returns `{"items": [], "total": 0, "has_more": false}`

### R21.7 โ€” No sleep in tests

`time.sleep()` in test code is forbidden. Tests that need to wait for async operations use `asyncio.wait_for()` with a timeout, event-based synchronization, or mock the time source. A test with `sleep(3)` for "waiting for background tasks" is a flaky test in disguise.

### R21.8 โ€” Test names describe the scenario and expected outcome

```
test_<method_name>_<scenario>_<expected_outcome>
```

Examples:
- `test_create_product_with_empty_name_raises_invalid_spec_error`
- `test_execute_ai_request_when_budget_exceeded_raises_budget_exceeded_error`
- `test_submit_workflow_without_auth_returns_401`
- `test_list_products_with_page_size_1_returns_next_cursor`

### R21.9 โ€” Flaky tests are treated as failing tests

A test that fails intermittently (passes on some runs, fails on others) is a bug in the test. Flaky tests are not masked with retry decorators โ€” they are fixed. A test that cannot be made deterministic is deleted and replaced with a different approach.

### R21.10 โ€” Coverage minimums are enforced per layer

| Layer | Statement coverage minimum |
|-------|--------------------------|
| `platform/security/` | 95% |
| `platform/ai-ros/` | 90% |
| `platform/workflow-runtime/` | 90% |
| `platform/prompt-os/` | 85% |
| `platform/knowledge/` | 85% |
| All other platform | 80% |
| `products/` | 80% |
| `sdk/` | 90% |

Coverage below minimum fails CI. Coverage minimum applies to statement coverage, not branch coverage (branch coverage is tracked but not yet enforced).

### R21.11 โ€” End-to-end tests verify the critical path

The following E2E scenarios must pass on every merge to `main`:
1. Create a product via API โ’ verify workflow created โ’ verify at least one task dispatched
2. Submit a workflow with a budget that exceeds the daily limit โ’ verify budget error returned
3. Call a protected route without credentials โ’ verify 401 returned
4. Render a Prompt OS template with variables โ’ verify rendered content matches expected

## 21.2 Good Examples

```python
# Good: clear AAA structure, descriptive name
def test_submit_workflow_when_daily_budget_exceeded_raises_budget_exceeded_error():
    # Arrange
    budget_manager = BudgetManagerFake(daily_limit_usd=10.00, current_spend_usd=10.01)
    service = WorkflowSubmissionService(budget_manager=budget_manager, ...)

    # Act & Assert
    with pytest.raises(BudgetExceededError) as exc_info:
        service.submit(WorkflowPlanFactory.build())

    assert exc_info.value.daily_limit_usd == 10.00
    assert exc_info.value.current_spend_usd == 10.01
```

```python
# Good: security test
async def test_create_product_without_credentials_returns_401(client: AsyncClient):
    response = await client.post("/api/v1/products", json={"name": "test"})
    assert response.status_code == 401
    assert response.json()["error"]["code"] == "AUTH_INVALID_API_KEY"
```

## 21.3 Bad Examples

```python
# Bad: no assertion on behavior โ€” only asserts the mock was called
def test_dispatch_task():
    mock_runtime = MagicMock()
    service.dispatch(task)
    mock_runtime.execute.assert_called_once()   # doesn't verify the task was correctly dispatched

# Bad: sleep in test
def test_background_task_completes():
    service.start_background_task()
    time.sleep(3)   # forbidden โ€” makes test slow and flaky
    assert service.task_completed

# Bad: non-descriptive test name
def test_product():   # what scenario? what outcome?
    ...

# Bad: two Act phases in one test
def test_create_and_cancel_product():
    product = service.create(...)   # Act 1
    service.cancel(product.id)      # Act 2 โ€” split into two tests
```

## 21.4 Common Mistakes

- Using `MagicMock` for everything (mocks hide behavior; fakes expose it)
- Not asserting on the content of raised exceptions (only asserting the type is raised)
- Writing tests that only test the happy path (most bugs are in error paths)
- Creating test fixtures that require a real database when a fake would work

---

---

# 22. Security Standards

**Rationale:** Security defects in a platform that executes arbitrary AI-generated code, manages secrets, and processes user-provided prompts can result in data exfiltration, arbitrary code execution, and credential theft. Security is not a feature โ€” it is a foundation.

## 22.1 Rules

### R22.1 โ€” Authentication is required on all routes by default

The SecurityMiddleware authenticates every request. A route is unauthenticated only if its path is in the `PUBLIC_PATHS` frozenset. `PUBLIC_PATHS` contains exactly: `/api/v1/health`, `/api/v1/capabilities`, `/api/v1/metrics`. Any addition to `PUBLIC_PATHS` requires a security review and CTO approval.

### R22.2 โ€” RBAC is required for all state-mutating operations

Every `POST`, `PUT`, `PATCH`, and `DELETE` route declares a `require_permission()` dependency. The permission is checked against the authenticated user's role. A route handler that modifies state without a `require_permission()` check is a security defect.

### R22.3 โ€” The default security configuration is the most restrictive option

| Setting | Default | Rationale |
|---------|---------|---------|
| `api_key` | (no default โ€” startup fails if not set) | Empty key disables auth |
| `cors_origins` | (no default โ€” startup fails if not set) | Wildcard CORS is forbidden |
| `nats_enabled` | `False` | External service; enable explicitly |
| `jwt_expiry_seconds` | `3600` (1 hour) | Short-lived tokens |
| `rate_limit_per_minute` | `60` | Conservative default |

### R22.4 โ€” Prompt validation is mandatory before all AI execution

Every prompt that goes to an AI provider must pass through `PromptValidator` before execution. The risk threshold for blocking is `risk_score >= 0.75`. If validation is bypassed by a feature flag (during migration), the bypass is logged at WARNING level with the skipped prompt ID.

### R22.5 โ€” Tool execution is sandboxed

`shell_exec` and `python_exec` tool calls are executed in an isolated subprocess with:
- Resource limits (CPU time, memory, file descriptors)
- No network access (unless explicitly granted per-execution)
- Restricted filesystem access (only the designated working directory)
- A timeout that kills the process after the configured limit

No tool execution occurs in the main process. No exceptions.

### R22.6 โ€” Credential comparisons are timing-safe

All credential comparisons use `hmac.compare_digest()`. This is enforced by the architecture linter:
```
Rule: no-timing-unsafe-comparison
Pattern: (api_key|secret|token|password)\s*(==|!=)\s*
Action: FAIL
```

### R22.7 โ€” SQL queries use parameterized statements

No SQL query is constructed by string concatenation or f-string formatting. All values are passed as parameters. SQLAlchemy ORM and its text() function with bound parameters are the approved approaches. Raw string-concatenated SQL is a linter violation.

### R22.8 โ€” Input validation occurs at the API boundary

All input is validated at the API boundary using Pydantic models before reaching service code. Service code trusts that its inputs are valid (they were validated before the call). No validation is duplicated in service methods.

### R22.9 โ€” Security events are audited

The following events are written to the security audit log (separate from the application log):
- Authentication success and failure
- Authorization failures (403 responses)
- Rate limit triggers
- Prompt validation rejections (risk_score logged, not content)
- API key creation and revocation
- Role assignments

The security audit log is append-only and cannot be deleted via normal application operations.

### R22.10 โ€” Dependencies are scanned for vulnerabilities in CI

`pip-audit` (or equivalent) runs in CI on every PR. PRs that introduce dependencies with known critical or high-severity CVEs are blocked. Known false positives are documented in `tools/security/pip-audit-ignore.toml` with justification and an expiry date.

## 22.2 Good Examples

```python
# Route with explicit RBAC
@router.post("/workflows", response_model=WorkflowResponse, status_code=202)
async def submit_workflow(
    request: SubmitWorkflowRequest,
    service: WorkflowService = Depends(get_workflow_service),
    _: AuthContext = Depends(require_permission(Permission.WORKFLOW_CREATE)),
) -> WorkflowResponse:
    handle = await service.submit(request)
    return WorkflowResponse(workflow_id=handle.workflow_id, status="accepted")
```

```python
# Timing-safe credential verification
import hmac
import hashlib

def verify_api_key(provided: str, stored_hash: str) -> bool:
    provided_hash = hashlib.sha256(provided.encode("utf-8")).hexdigest()
    return hmac.compare_digest(provided_hash, stored_hash)
```

## 22.3 Bad Examples

```python
# Route without auth โ€” forbidden (10 routes had this in the architecture review)
@router.post("/products")
async def create_product(request: CreateProductRequest) -> ProductResponse:
    ...   # No auth dependency โ€” this is a security defect

# Timing-unsafe comparison
if provided_key != stored_key:   # == or != on secrets โ€” forbidden
    raise HTTPException(401, ...)

# SQL injection via concatenation
query = f"SELECT * FROM tasks WHERE status = '{status}'"   # forbidden
db.execute(query)

# Tool execution in main process
import subprocess
result = subprocess.run(user_shell_command, shell=True)   # no sandbox โ€” forbidden
```

## 22.4 Common Mistakes

- Using `is_authenticated=True` boolean in AuthContext instead of checking the role (role encodes permission level)
- Forgetting `require_permission()` on a new PATCH or DELETE route
- Using `http://` (not `https://`) in CORS origins for non-local environments
- Not invalidating JWT tokens on logout (stateless JWT cannot be invalidated โ€” use short expiry + refresh tokens)

---

---

# 23. Code Review Checklist

**Rationale:** Code review is the last line of defense before code reaches `main`. A consistent checklist ensures that reviewers check for the same things every time, regardless of who is reviewing.

## 23.1 Rules

### R23.1 โ€” Every PR requires at least one approval before merge

The approver must not be the author. AI agents reviewing code must flag this checklist explicitly in their review comment.

### R23.2 โ€” The checklist is applied to every PR without exception

The reviewer works through the checklist in order. If any item fails, the PR is returned with specific, actionable feedback โ€” not vague requests to "clean it up."

---

## 23.2 The Checklist

### Group 1: Architecture Compliance

- [ ] **Layer separation:** Does the PR respect layer boundaries? (platform imports from product โ’ FAIL; product direct-imports platform bypassing SDK โ’ FAIL)
- [ ] **Single canonical implementation:** Does the PR add a new implementation of a capability that already has a canonical one? (new prompt renderer, new experience recorder โ’ FAIL)
- [ ] **No simulation or shortcuts:** Does the PR contain `time.sleep()` for "waiting for async operations", `_simulate_*` methods, or hardcoded timeouts in business flows? (โ’ FAIL)
- [ ] **SDK compliance:** Does product code access platform services only via SDK clients? (โ’ FAIL if direct import)
- [ ] **Event naming:** Do any new events follow the `<domain>.<entity>.<verb>` convention with plural domain and past-tense verb? (โ’ FAIL if not)

### Group 2: Security

- [ ] **Auth on new routes:** Does every new route have a `require_permission()` dependency or explicit justification for why it is public? (โ’ FAIL if missing)
- [ ] **No timing-unsafe comparisons:** Are there any `==` or `!=` comparisons of credential strings? (โ’ FAIL)
- [ ] **No secrets in code:** Does the PR contain API keys, passwords, tokens, or any hardcoded credential? (โ’ FAIL; report as security issue)
- [ ] **Prompt validation:** If the PR adds a new AI execution path, does it call `PromptValidator` before sending to provider? (โ’ FAIL if missing)
- [ ] **Input validation:** Does all user-supplied input pass through Pydantic validation at the API boundary? (โ’ FAIL if bypassed)
- [ ] **SQL safety:** Are all SQL queries parameterized? No string-concatenated SQL? (โ’ FAIL)
- [ ] **Sensitive data logging:** Are any secrets, API keys, or user content logged? (โ’ FAIL)

### Group 3: Data Model and Configuration

- [ ] **UUIDs for IDs:** Are all new resource identifiers UUID, not integer? (โ’ FAIL)
- [ ] **UTC datetimes:** Are all datetime fields using `datetime.now(UTC)`, not `datetime.utcnow()`? (โ’ FAIL)
- [ ] **No hardcoded config:** Are all configuration values (URLs, timeouts, limits) read from settings? No hardcoded strings for environment-specific values? (โ’ FAIL)
- [ ] **Settings have no dangerous defaults:** Do new security settings have no default value (startup fails if not set)? (โ’ FAIL)

### Group 4: Database

- [ ] **Migration included:** If the PR changes the data model, does it include an Alembic migration? (โ’ FAIL if missing)
- [ ] **Migration has downgrade:** Does the migration's `downgrade()` function reverse the upgrade? (โ’ FAIL if `pass`)
- [ ] **Indexes for new query columns:** If new columns will be queried in WHERE or ORDER BY clauses, are indexes included? (โ’ FAIL if missing)
- [ ] **No cross-migration ALTER:** Does the migration modify tables that were created in a different migration file? (โ’ FAIL)

### Group 5: Testing

- [ ] **Tests included:** Does the PR include tests for all new behavior? Is statement coverage at or above the minimum for the layer? (โ’ FAIL)
- [ ] **Security tests included:** If new routes were added, are there tests for 401 (no auth) and 403 (wrong permission)? (โ’ FAIL)
- [ ] **No sleep in tests:** Does any test call `time.sleep()`? (โ’ FAIL)
- [ ] **Test names are descriptive:** Do test names describe scenario and expected outcome? (โ’ FAIL for `test_method()` style names)
- [ ] **No skipped tests:** Are any tests marked `@pytest.skip` without a link to a tracking ticket? (โ’ FAIL)

### Group 6: Code Quality

- [ ] **No commented-out code:** Is there any code that has been commented out? (โ’ FAIL โ€” delete it, git has it)
- [ ] **No TODO/FIXME:** Are there TODO or FIXME comments? (โ’ FAIL โ€” create a ticket instead)
- [ ] **Comments explain WHY:** Do comments explain a non-obvious constraint or invariant, not what the code does? (โ’ FAIL for "# get the workflow" style comments)
- [ ] **Exception types are specific:** Does the PR use typed exceptions from the hierarchy, not `ValueError` or `RuntimeError`? (โ’ FAIL)
- [ ] **Dependency injection:** Are all dependencies injected, not instantiated inside methods? (โ’ FAIL)

### Group 7: Documentation

- [ ] **README updated:** If the PR changes the public interface of a package, is the README updated? (โ’ FAIL)
- [ ] **CHANGELOG updated:** If the PR is user-facing behavior (new feature, bug fix, breaking change), is CHANGELOG updated? (โ’ FAIL)
- [ ] **ADR created:** If the PR makes a significant architectural decision, is an ADR created? (โ’ FAIL for architectural decisions without ADRs)

### Group 8: Reviewer Declaration

The reviewer states explicitly in the review comment:
```
Code review checklist completed.
Groups passed: 1, 2, 3, 4, 5, 6, 7
Items failed: [list any that failed, or "None"]
Approval status: APPROVED / CHANGES REQUESTED
```

---

---

# 24. Definition of Done

**Rationale:** "Done" means different things to different people without a shared definition. A shared DoD eliminates the "it works on my machine" category of incomplete work.

## 24.1 The Definition

A unit of work (ticket, PR, feature, phase) is **DONE** when ALL of the following are true. Partial completion is not done.

### Code Criteria

- [ ] All code written and committed
- [ ] All code reviewed and approved (Section 23 checklist completed)
- [ ] Architecture linter passes with 0 violations
- [ ] All new tests written and passing
- [ ] No pre-existing tests newly broken by this work
- [ ] Statement coverage at or above layer minimum (Section 21, R21.10)
- [ ] Security tests written for all new routes (if applicable)
- [ ] Pagination tests written for all new list endpoints (if applicable)

### Quality Criteria

- [ ] No commented-out code
- [ ] No TODO/FIXME comments (all converted to tickets)
- [ ] No `time.sleep()` in tests
- [ ] No hardcoded configuration values
- [ ] No hardcoded secrets
- [ ] All exceptions use the typed hierarchy
- [ ] All log statements use the structured logger

### Functional Criteria

- [ ] Feature behaves as specified (manually verified or automated E2E test)
- [ ] Edge cases verified: empty inputs, null values, max values, concurrent access
- [ ] Error cases verified: what happens when dependencies fail
- [ ] Performance: no new O(nยฒ) operations on unbounded collections

### Operational Criteria

- [ ] New configuration settings documented in README and `.env.example`
- [ ] New events documented in the event catalog
- [ ] Alembic migration included if schema changed (migration tested up and down)
- [ ] New platform module implements `PlatformModule.startup()`, `shutdown()`, `health()`

### Documentation Criteria

- [ ] CHANGELOG updated if user-facing behavior changed
- [ ] README updated if public interface changed
- [ ] ADR created if architectural decision made
- [ ] Breaking changes explicitly documented with migration guide

### CI Criteria

- [ ] All CI checks pass on the PR branch (not just on main)
- [ ] `pytest` passes
- [ ] `mypy` passes (or new errors are explicitly justified)
- [ ] `ruff` passes
- [ ] Architecture linter passes
- [ ] Dependency security scan passes

## 24.2 "Done" for Phases

A Phase (Phase A, B, C, etc.) is DONE when:
1. All tickets in the phase meet the individual Definition of Done
2. The Phase Readiness Gate criteria (Section 18.1 of the Blueprint) are verified
3. The pre-deployment verification commands (Appendix KK of the Architecture Review) pass
4. CTO or delegate has explicitly signed off

**"Complete" is never self-declared by the implementer.** Completion is declared by the reviewer or sign-off authority after independently verifying the above criteria.

---

---

# 25. Release Standards

**Rationale:** Releases are the mechanism by which work reaches users. An inconsistent release process produces unreliable deployments, missed steps, and rollbacks under pressure. Consistent process produces predictable releases.

## 25.1 Release Types

| Type | Trigger | Scope | Approval |
|------|---------|-------|---------|
| Patch release | Critical bug fix or security fix | Bug only, no features | Platform Architect |
| Minor release | Feature work complete | New capabilities, backward-compatible | Chief Architect |
| Major release | Breaking changes | API or contract break | CTO |
| Hotfix | Production incident | Minimal change to resolve incident | On-call + Chief Architect |

## 25.2 Rules

### R25.1 โ€” Releases are tagged from `main` only

All releases are tagged from the `main` branch. Releases from feature branches or release branches are not permitted. The only exception is hotfixes, which may be cherry-picked to a running release branch.

### R25.2 โ€” Every release has a git tag and a GitHub release

The git tag follows `v<major>.<minor>.<patch>` (e.g., `v2.1.0`). A GitHub release is created from the tag with:
- The relevant CHANGELOG section as the release body
- A list of all PRs included (auto-generated from git log)
- A list of all breaking changes (if MAJOR)
- Migration instructions (if MAJOR)

### R25.3 โ€” Releases are preceded by a release checklist run

Before any release is tagged:

**Pre-release checks:**
- [ ] All CI checks pass on `main`
- [ ] All tests pass (including E2E suite)
- [ ] Security scan passes
- [ ] Architecture linter passes
- [ ] CHANGELOG is complete and accurate
- [ ] Version number updated in `pyproject.toml`
- [ ] `.env.example` is current (no undocumented new settings)
- [ ] Migration files are verified (upgrade and downgrade work)

**For MAJOR releases, additionally:**
- [ ] Breaking change documentation complete
- [ ] Migration guide reviewed by at least one consumer team
- [ ] Compatibility shims in place (old interfaces still work during compatibility period)
- [ ] CTO sign-off obtained

### R25.4 โ€” Production deployments require a deployment runbook

Every deployment to production uses a written runbook. The runbook includes:
1. Pre-deployment health check (verify current production is healthy before deploying)
2. Backup verification (database backup completed and verified before schema migrations)
3. Migration execution steps
4. Smoke test steps (verify core workflows after deployment)
5. Rollback procedure (exact steps to revert if deployment fails)
6. Success criteria (how to know the deployment succeeded)

Runbooks are stored in `docs/runbooks/`. A deployment without a runbook is not executed.

### R25.5 โ€” Database migrations run before application deployment

Migration order: deploy migration โ’ verify migration โ’ deploy application. Never: deploy application + migration simultaneously. Never: deploy application before migration.

### R25.6 โ€” Hotfixes follow an abbreviated but not absent process

A hotfix under time pressure still requires:
1. A branch from the affected release tag
2. A PR (even if self-approved in extreme emergencies โ€” documented and reviewed retroactively)
3. The fix tested locally before deployment
4. A post-incident review within 48 hours

### R25.7 โ€” Release schedule is predictable

- Minor releases: every 4-6 weeks
- Patch releases: as needed (targeted within 24 hours for critical bugs)
- Major releases: announced minimum 4 weeks in advance to all consumers

No surprise releases. No emergency MAJOR releases (a MAJOR release with 4 weeks notice is never a surprise to consumers).

### R25.8 โ€” Post-release monitoring period

After every release, the engineering team monitors the following for a minimum of 24 hours:
- Error rate (Prometheus alert rule: error rate > 5% for 5 minutes)
- AI cost rate (ensure no unexpected spend increase)
- Workflow throughput (ensure no degradation)
- Response time p99 (ensure no latency regression)

If any metric degrades, the rollback decision is made within 1 hour of detection.

## 25.2 Release Communication

| Audience | Minor release | Major release | Hotfix |
|---------|-------------|--------------|--------|
| Engineering team | PR merged + CHANGELOG | 4-week advance notice + migration guide | Incident channel immediately |
| Product teams | CHANGELOG at release | 4-week advance + implementation support | Incident channel + impact assessment |
| External operators | CHANGELOG at release | 4-week advance + upgrade guide | Incident notification + ETA |

## 25.3 Good Examples

```
# Good release tag
v2.1.0    โ’ minor release with backward-compatible features

# Good hotfix process
1. Incident declared
2. Hotfix branch created from v2.0.3 tag
3. Fix implemented, tested locally
4. PR created, self-approved with comment: "Hotfix for prod incident INC-2026-0628-001"
5. Deployed to staging, verified
6. Deployed to production
7. Post-incident review scheduled within 48 hours

# Good release body (GitHub release)
## v2.1.0 โ€” 2026-08-15

### New Features
- AI ROS: BudgetManager daily limit enforcement per product (#251)
- Workflow Runtime: Real task dispatch (removes simulation) (#203)
- Security: JWT dual-path authentication (#187)

### Bug Fixes
- Fix NATS subject pluralization bug (workflow.* โ’ workflows.*) (#199)

### Migration Notes
- Set AISF_SECURITY__API_KEY in your environment before upgrading
- Run `alembic upgrade head` before starting the new version
```

## 25.4 Bad Examples

```
# Bad: no migration instructions
Release v2.0.0 โ€” lots of changes!

# Bad: deploying application before migration
docker pull aistudio:v2.1.0 && docker run aistudio:v2.1.0   # migration not mentioned

# Bad: no rollback plan
Deployment plan:
1. Deploy new version
2. Hope it works
```

---

---

# Appendix A โ€” Standards Enforcement Summary

| Standard area | Primary enforcement | Secondary enforcement |
|---------------|--------------------|-----------------------|
| Layer boundaries | Architecture linter (CI) | Code review checklist Group 1 |
| Security controls | SecurityMiddleware + linter | Code review checklist Group 2 |
| Event naming | SDK event catalog (typed) | Code review checklist Group 1 |
| Test coverage | pytest-cov threshold in CI | Code review checklist Group 5 |
| Commit messages | commitlint in CI | PR description requirement |
| Secrets | detect-secrets pre-commit + CI | Code review checklist Group 2 |
| Dependencies | Architecture linter | Code review checklist Group 1 |
| Migration safety | Alembic CI test (up/down/up) | Code review checklist Group 4 |
| Timing-safe comparisons | Architecture linter | Code review checklist Group 2 |
| Feature flag expiry | CI warning โ’ CI failure | Feature flag registry review |

---

# Appendix B โ€” Standards Exceptions Register

All approved exceptions to these standards are tracked here. This register is maintained by the Chief Architect and reviewed quarterly.

| Exception ID | Standard violated | Justification | Expiry | Approver |
|-------------|------------------|---------------|--------|---------|
| EXC-001 | R22.1 (auth on all routes) | `/api/v1/metrics` is intentionally open for Prometheus scraping without credentials | 2026-12-31 | CTO |
| EXC-002 | R3.1 (monorepo) | `content-factory` Java service is in monorepo but uses Maven, not pyproject.toml | Never (Java toolchain) | Chief Architect |
| EXC-003 | R21.5 (security test for every route) | 39 pre-existing routes were created before this standard. Security tests to be added in Phase A (deadline: 2026-09-01) | 2026-09-01 | CTO |

---

# Appendix C โ€” Quick Reference: The Ten Commandments

These ten rules are the most commonly violated. Memorize them.

1. **Platform code does not import from products.** Products do not import from platform directly โ€” they use the SDK.

2. **Secrets never appear in code, logs, or error messages.** `hmac.compare_digest()` for all credential comparisons.

3. **Every route requires authentication by default.** New routes without `require_permission()` are a security defect.

4. **One canonical implementation per capability.** No second prompt renderer. No second experience recorder.

5. **All configuration comes from Pydantic Settings.** No `os.environ.get()`. No hardcoded paths or URLs.

6. **All IDs are UUID. All timestamps are UTC datetime.** No integers as IDs. No naive datetimes.

7. **Migrations are atomic, single-purpose, and tested up and down.**  No migration modifies another migration's tables.

8. **Tests are written alongside the code, not after merge.** No `time.sleep()` in tests. No `@pytest.skip` without a ticket.

9. **Events are `<plural-domain>.<entity>.<past-tense-verb>`.** `workflows.task.completed`. Not `workflow.task.complete`.

10. **"Done" is verified by the reviewer, not declared by the author.** The Definition of Done checklist is a requirement, not a suggestion.

---

*End of document.*

---

**AI Studio Platform โ€” Engineering Standards**  
**Version:** 1.0.0  
**Date:** 2026-06-28  
**Status:** RATIFIED  
**Next Review:** 2026-09-28  
**Document Owner:** Chief Software Architect  

*This document supersedes all previous informal guidelines, README notes, and verbal agreements about coding standards. Where this document conflicts with existing code, this document is correct and the code is technical debt.*

