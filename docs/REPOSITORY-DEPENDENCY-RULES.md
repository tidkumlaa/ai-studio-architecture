# AI Studio Platform — Repository Dependency Rules

**Document ID:** REPOSITORY-DEPENDENCY-RULES  
**Version:** 1.0.0  
**Date:** 2026-06-28  
**Status:** PROPOSED — Pending Architect Approval  
**Authority:** Chief Software Architect  
**Enforcement:** Architecture Linter (`tools/architecture-linter/`)  
**Companion:** REPOSITORY-SEPARATION-PLAN.md, COUPLING-ANALYSIS.md

---

## Purpose

This document defines the allowed dependency graph for the AI Studio workspace after Phase 0B extraction. These rules are:
- **Binding** — violations block CI
- **Enforceable** — implementable with `import-linter` or equivalent
- **Complete** — every allowed edge is listed; everything else is forbidden

---

## 1. Repository Dependency Graph

```
┌─────────────────────────────────────────────────────────────────────┐
│  ALLOWED DEPENDENCY EDGES (→ means "may import from")              │
│                                                                     │
│  products/ai-software-factory  →  platform/  →  sdk/python         │
│  products/content-factory      →  (none — standalone Java)         │
│  products/mythic-realms        →  sdk/python                        │
│  desktop/ai-studio-desktop     →  (HTTP only — no imports)          │
│  tools/*                       →  sdk/python                        │
│  examples/*                    →  sdk/python                        │
│  marketplace/*                 →  sdk/python                        │
│                                                                     │
│  FORBIDDEN EDGES:                                                   │
│  platform/  →  products/*      (platform cannot depend on products) │
│  platform/  →  desktop/*       (platform cannot depend on desktop)  │
│  sdk/*      →  platform/       (SDK cannot depend on platform)      │
│  sdk/*      →  products/*      (SDK cannot depend on products)      │
│  desktop/*  →  platform/* (import) (HTTP only — no Python imports)  │
│  products/X →  products/Y      (no cross-product imports)           │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. Platform Layer Rules

### R-PLAT-1: Platform is dependency-free of products

The `platform/` directory — including all sub-directories — must not import from any module in:
- `products/`
- `desktop/`
- `tools/` (except at startup — config loading is permitted)
- `examples/`

**Violation example:**
```python
# FORBIDDEN in platform/workflow-runtime/dispatcher.py
from products.ai_software_factory.workers import ArchitectWorker  # NO
from worker.architect_worker import ArchitectWorker               # NO (legacy)
```

**Correct pattern — plugin registry:**
```python
# CORRECT in platform/workflow-runtime/dispatcher.py
from platform.workflow_runtime.worker_registry import WorkerRegistry
worker = WorkerRegistry.get(agent_name)  # workers registered at startup via config
```

### R-PLAT-2: Platform internal dependency order

Within the `platform/` directory, dependencies flow from lower layers to higher layers. Circular imports are forbidden. Allowed dependency order (lower = fewer deps):

```
Layer 0: configuration/, metrics/, event-bus/in_process
   ↑
Layer 1: security/, storage/, event-bus/nats_client
   ↑
Layer 2: workflow-runtime/, prompt-os/, provider-runtime/
   ↑
Layer 3: central-brain/, decision-engine/, knowledge/
   ↑
Layer 4: ai-runtime/, plugin-runtime/
   ↑
Layer 5: governance/, review-engine/, workspace/
   ↑
Layer 6: api/ (FastAPI routes — depends on all above)
```

A module in Layer N must not import from a module in Layer N+1 or higher.

**Violation example:**
```python
# FORBIDDEN in platform/event-bus/ (Layer 1)
from platform.ai_runtime.router import AIRouter  # NO — Layer 4 is above Layer 1
```

### R-PLAT-3: Platform uses event bus for cross-module communication

When a platform module in Layer N needs to trigger behavior in Layer M (where M ≤ N), it publishes an event. It does not import from Layer M.

**Correct pattern:**
```python
# In platform/workflow-runtime/ (Layer 2) triggering brain recording (Layer 3):
# CORRECT — publish event; brain subscribes
event_bus.publish("workflows.task.completed", payload)
# Brain consumer handles the event — no import needed
```

### R-PLAT-4: Platform module namespace

All platform modules use the `platform.` import prefix after extraction. During the transition period (before full extraction), platform modules continue using their current import paths.

---

## 3. SDK Rules

### R-SDK-1: SDK has no runtime dependencies

The Python SDK package (`sdk/python/`) installs with zero external dependencies beyond the Python standard library.

**Forbidden in sdk/python/:**
```
fastapi, uvicorn, sqlalchemy, pydantic, httpx, nats-py, 
prometheus-client, opentelemetry-sdk
```

**Permitted in sdk/python/:**
```
typing, dataclasses, abc, pathlib, subprocess, json, re, time, threading
```

If the SDK needs to communicate with the platform, it does so over HTTP (`httpx` as an optional dependency for the client SDK extension). The core SDK must be dependency-free.

### R-SDK-2: SDK has no product dependencies

`sdk/python/` must not import from `products/`, `platform/`, or `desktop/`.

### R-SDK-3: SDK has no platform internal module imports

Products that install the SDK get `aisf.*` module names, not `factory.*` or `engine.*`. The SDK is a public interface; platform internals are not.

**Before (current — FORBIDDEN after extraction):**
```python
from factory.sdk.worker_base import BaseWorker       # internal path
from engine.prompt_os.template import TemplateSpec   # internal path
```

**After (correct — SDK public interface):**
```python
from aisf.worker_base import BaseWorker              # SDK public path
from aisf.types import TemplateSpec                  # SDK public path
```

### R-SDK-4: SDK version is independent of platform version

The SDK is versioned independently. Platform v1.0 may ship with SDK v1.0. A bug fix to the platform that does not change the SDK interface does not require an SDK version bump.

**Compatibility contract:**
- Platform v1.x is compatible with SDK v1.x (minor-version compatible)
- SDK breaking changes require a new major version (SDK v2.0)
- Products pin to a SDK major version: `aisf>=1.0,<2.0`

---

## 4. Product Rules

### R-PROD-1: Products may depend on Platform API (HTTP) and SDK (import)

Products communicate with the platform in two ways:
1. **HTTP REST** — calling platform API endpoints (workflow, prompt, brain, provider)
2. **SDK import** — using `from aisf.worker_base import BaseWorker` etc.

Products must not import from platform internal modules (`platform.*`, `engine.*`, `factory.*`, `db.*`).

### R-PROD-2: Products may not import from other products

Product A may not import from Product B. There are no cross-product Python imports.

**Forbidden:**
```python
# In products/mythic-realms/ 
from products.ai_software_factory.engine import ArchitectAgent  # FORBIDDEN
```

**If cross-product data sharing is needed:** Both products use the Platform API. Product A writes to the platform's knowledge graph; Product B reads from it.

### R-PROD-3: Product configuration is product-namespaced

Product configuration variables use a product-specific prefix:
- AISF product: `AISF_PROD_*`
- Mythic Realms: `MR_*` or `MYTHIC_*`
- Content Factory: `CF_*` (already uses this — compatible)

Products do not read platform `AISF_*` settings directly.

### R-PROD-4: Product workers are registered via configuration

Workers are not discovered by importing from a shared `worker/__init__.py`. Each product's `factory.yaml` declares its workers:

```yaml
# products/mythic-realms/factory.yaml
workers:
  - agent: BATTLE-AGENT
    class: mythic_realms.workers.battle_worker.BattleWorker
  - agent: ECONOMY-AGENT
    class: mythic_realms.workers.economy_worker.EconomyWorker
```

The platform's WorkerRegistry loads worker classes from these declarations at startup.

### R-PROD-5: Products own their database schemas

Each product may have its own database tables in a product-namespaced schema:
- AISF product: `aisf.*` (tables specific to AISF product logic)
- Mythic Realms: no platform DB tables (uses its own Java backend DB)
- Content Factory: no platform DB tables (standalone)

Platform schemas (`security.*`, `workflow.*`, `prompt.*`, etc.) are owned by the platform. Products do not write to platform schemas directly — they use the platform API.

---

## 5. Desktop Rules

### R-DESK-1: Desktop has zero Python import coupling to platform

The desktop (`desktop/ai-studio-desktop/`) has no `import` or `from ... import` statements referencing any platform or product module. All communication is HTTP.

This rule must be enforced by CI: `grep -r "from engine\|from factory\|from platform\|from products\|from worker" desktop/` must return zero results.

### R-DESK-2: Desktop discovers services from workspace.yaml

The desktop reads `workspace.yaml` to discover platform URLs. It does not hardcode `http://localhost:8088`. Service discovery allows the platform to run on any host/port.

### R-DESK-3: Desktop authenticates with an API key

Every HTTP request from the desktop includes the `Authorization: Bearer <key>` header. There is no special desktop-bypass authentication.

### R-DESK-4: Desktop may not start the platform inline

The desktop is a UI client. It does not embed or start the platform server. Platform lifecycle is managed separately. The desktop shows platform status but does not control it.

---

## 6. Tools Rules

### R-TOOLS-1: Tools may depend on SDK but not on platform internals

`tools/generator/`, `tools/architecture-linter/`, `tools/migration-tools/` may import from `sdk/python/` but must not import from `platform/` internal modules.

**Exception:** `tools/migration-tools/` may import from `platform/api/db/` (Alembic migration scripts) because its sole purpose is database management. This exception is documented here and does not extend to other tools.

### R-TOOLS-2: Architecture linter enforces these rules

`tools/architecture-linter/` runs as a CI check on every pull request. It checks:
- No platform → product imports
- No cross-product imports
- No desktop → platform imports
- No SDK → platform imports
- Platform layer order violations

---

## 7. Configuration and Secrets Rules

### R-CFG-1: Platform configuration uses AISF_ prefix

All platform environment variables use `AISF_` prefix. The legacy `ORCH_` prefix is forbidden after migration (deprecated with warning during transition).

| Old (FORBIDDEN after migration) | New (REQUIRED) |
|---------------------------------|----------------|
| `ORCH_API_KEY` | `AISF_API_KEY` |
| `ORCH_NATS_ENABLED` | `AISF_NATS_ENABLED` |
| `ORCH_DATABASE_URL` | `AISF_DB_URL` |
| `ORCH_CF_BASE_URL` | `AISF_CF_BASE_URL` |

### R-CFG-2: Authentication is enabled by default

There is no configuration variable that disables authentication. The legacy `ORCH_API_KEY=""` disabling pattern is removed. To run without authentication in local development, use `AISF_AUTH_SKIP_LOCALHOST=true` (loopback only, local env only).

### R-CFG-3: Secrets use AISF_SECRET_ prefix

All platform secrets follow SECURITY-MODEL.md Section 6.3:
```
AISF_SECRET_ANTHROPIC_API_KEY
AISF_SECRET_PROMPT_HMAC_KEY
AISF_SECRET_DB_PASSWORD
```

---

## 8. Event Bus Rules

### R-EVT-1: Events use the canonical subject format

All NATS subjects follow EVENT-CATALOG.md naming convention: `<plural-domain>.<entity>.<past-tense-verb>`

**Forbidden:**
```
workflow.created     # singular domain
workflows.create     # present tense
workflows.created.v1 # version in subject
```

**Required:**
```
workflows.workflow.submitted
prompts.template.state_changed
```

### R-EVT-2: Platform modules publish; products subscribe

Platform modules publish events. Products (and other platform modules at higher layers) subscribe. Platform modules at Layer N must not subscribe to events that are emitted by Layer N+1.

### R-EVT-3: Cross-product communication via events only

If Product A needs to trigger behavior in Product B, it publishes an event to the platform event bus. Product B subscribes. There is no direct API call from Product A to Product B.

---

## 9. Import Rules Enforcement

### 9.1 import-linter Configuration (target state)

```ini
# .importlinter (at workspace root after extraction)
[importlinter]
root_packages =
    platform
    sdk
    products.ai_software_factory
    products.mythic_realms
    desktop

[importlinter:contract:platform-no-product]
name = Platform must not import from products
type = forbidden
source_modules = platform
forbidden_modules = products

[importlinter:contract:platform-no-desktop]
name = Platform must not import from desktop
type = forbidden
source_modules = platform
forbidden_modules = desktop

[importlinter:contract:sdk-no-platform]
name = SDK must not import from platform internals
type = forbidden
source_modules = sdk
forbidden_modules =
    platform
    products

[importlinter:contract:no-cross-product]
name = No cross-product imports
type = forbidden
source_modules = products.ai_software_factory
forbidden_modules = products.mythic_realms

[importlinter:contract:no-cross-product-reverse]
name = No cross-product imports (reverse)
type = forbidden
source_modules = products.mythic_realms
forbidden_modules = products.ai_software_factory

[importlinter:contract:platform-layer-order]
name = Platform layer dependency order
type = layers
containers = platform
layers =
    api
    governance | review_engine | workspace
    ai_runtime | plugin_runtime
    central_brain | decision_engine | knowledge
    workflow_runtime | prompt_os | provider_runtime
    security | storage | event_bus
    configuration | metrics
```

### 9.2 CI Enforcement

The architecture linter runs as a mandatory CI check:
```yaml
# CI step
- name: Architecture Linter
  run: |
    python -m importlinter --config .importlinter
    python tools/architecture-linter/check_events.py
    python tools/architecture-linter/check_config_prefix.py
```

A failing architecture linter check blocks the pull request merge.

---

## 10. Rule Violation Handling

### 10.1 Finding a violation

When an engineer discovers a dependency rule violation in existing code:
1. Do not fix it silently — open a tracking issue
2. Label it `arch-violation`
3. It is prioritized above feature work

### 10.2 Requesting an exception

If a dependency rule cannot be satisfied without significant rework:
1. Open an RFC (Request for Comment) in `architecture/decisions/`
2. State the case for the exception with evidence
3. Chief Software Architect must approve
4. Exception is documented in this file (Appendix: Exceptions)

### 10.3 Temporary bypass

During the extraction phases, violations that cannot yet be fixed are tracked in `COUPLING-ANALYSIS.md` with a target resolution date. Temporary bypass markers:

```python
# ARCH-VIOLATION: C-001 — Mythic Realms worker in platform
# Resolution: Phase 5 extraction — target 2026-07-15
# Tracking: arch-violation-001
from worker.battle_worker import BattleWorker  # NOQA: ARCH
```

The CI tool ignores lines with `# NOQA: ARCH` but reports a count of bypasses. More than 10 active bypasses fails the build.

---

## Appendix: Exceptions Register

| Exception ID | Rule | Exception | Rationale | Expiry |
|-------------|------|-----------|-----------|--------|
| (none at v1.0) | | | | |

---

*End of REPOSITORY-DEPENDENCY-RULES.md*  
*Version 1.0.0 | Status: PROPOSED | 10 rule categories | 20+ binding rules*
