---
knowledge_id: KNW-PLAT-ARCH-010
title: "Platform Dependency Rules"
status: AUTHORITATIVE
phase: "2.1D.0"
version: "1.0.0"
created: "2026-06-29"
purpose: "Define all allowed and forbidden dependencies — the law of the platform"
canonical_source: "architecture/docs/platform-consolidation/10-DEPENDENCY-RULES.md"
dependencies:
  - "06-RUNTIME-MODEL.md"
  - "07-KERNEL-MODEL.md"
  - "08-SDK-MODEL.md"
related_documents:
  - "11-LAYER-RULES.md"
  - "12-IMPORT-RULES.md"
  - "25-VERIFICATION.md"
acceptance_criteria:
  - "Every allowed/forbidden combination has a rule number"
  - "The dependency matrix covers all layer combinations"
  - "Forbidden rules are machine-verifiable"
  - "No exceptions exist without an ADR reference"
verification_checklist:
  - "[ ] Dependency matrix is complete (N×N for all layers)"
  - "[ ] Every rule in this doc is implemented in 25-VERIFICATION.md"
  - "[ ] Linter can reject every forbidden combination"
future_extensions:
  - "Per-runtime dependency allow-list (finer than layer-level)"
  - "Time-bounded exceptions with expiry"
---

# Platform Dependency Rules

## Rule Numbering

All rules are numbered `DR-NNN`. Every rule is machine-verifiable (import scanner).

---

## Canonical Dependency Direction

```
Products
    ↓ (allowed: SDK only)
platform.sdk
    ↓ (allowed: services, kernel)
platform.services
    ↓ (allowed: kernel, common)
platform.runtimes.*
    ↓ (allowed: kernel, common)
platform.kernel
    ↓ (allowed: common, stdlib)
platform.common
    ↓ (allowed: stdlib only)
Python stdlib
```

**Every arrow is one-way. No upward arrows. No lateral arrows except via events.**

---

## Dependency Matrix

`A` = Allowed | `F` = Forbidden | `E` = Event-only (via kernel.events)

| From \ To | common | kernel | services | runtimes | sdk | products | stdlib |
|-----------|--------|--------|----------|---------|-----|----------|--------|
| common | — | F | F | F | F | F | A |
| kernel | A | — | F | F | F | F | A |
| services | A | A | A* | F | F | F | A |
| runtimes | A | A | A | E | F | F | A |
| sdk | A | A | A | F | — | F | A |
| products | F | F | F | F | A | — | A |

*services→services: acyclic only (no cycles)

---

## Specific Rules

### DR-001 — Product imports SDK only
```
ALLOWED:   from platform.sdk.ai import optimize_routing
FORBIDDEN: from platform.runtimes.ai_runtime.routing_optimizer import RoutingOptimizer
FORBIDDEN: from platform.kernel.di import Container
FORBIDDEN: from platform_kernel.di import Container
```

### DR-002 — No runtime-to-runtime imports
```
ALLOWED:   await kernel.events.publish(AIExecutionCompleted(...))
FORBIDDEN: from platform.runtimes.knowledge_runtime.graph import KnowledgeGraph
FORBIDDEN: from platform.runtimes.workflow_runtime import WorkflowExecutor
```
Exception: runtime_a may import TYPE_CHECKING-only from runtime_b IF an ADR approves it.

### DR-003 — Kernel has no runtime dependencies
```
ALLOWED:   from platform.common.types import BaseID
ALLOWED:   import asyncio
FORBIDDEN: from platform.runtimes.ai_runtime import ResourceIntelligenceService
FORBIDDEN: import anthropic
FORBIDDEN: import fastapi
```

### DR-004 — SDK imports no runtimes
```
ALLOWED:   from platform.services.auth import AuthService
ALLOWED:   from platform.kernel.registry import CapabilityRegistry
FORBIDDEN: from platform.runtimes.ai_runtime.routing_optimizer import RoutingOptimizer
```

### DR-005 — Services import no runtimes
```
ALLOWED:   from platform.kernel.events import EventBus
FORBIDDEN: from platform.runtimes.ai_runtime import ResourceIntelligenceService
```

### DR-006 — Common imports stdlib only
```
ALLOWED:   import dataclasses, typing, uuid, datetime, enum, abc
FORBIDDEN: from platform.kernel import anything
FORBIDDEN: import pydantic
FORBIDDEN: import fastapi
```
Exception: `platform.common.types` may import pydantic BaseModel (pydantic is in common deps).
This exception is documented in ADR-002.

### DR-007 — No old platform paths after consolidation
```
FORBIDDEN: from platform_kernel import anything
FORBIDDEN: from platform_sdk import anything
FORBIDDEN: import platform_kernel
FORBIDDEN: import platform_sdk
```

### DR-008 — Plugin references are string IDs only
```
ALLOWED:   provider_id: str = "anthropic"
FORBIDDEN: from anthropic_provider import AnthropicPlugin
```
Plugin imports only inside `platform.runtimes.provider_runtime.adapters.*`.

### DR-009 — Tests may import anything within platform
```
ALLOWED:   from platform.runtimes.ai_runtime.routing_optimizer import RoutingOptimizer
ALLOWED:   from platform.kernel.di import DIContainer
```
Tests are exempt from visibility rules for white-box testing.

### DR-010 — Tools import platform.tools, platform.kernel, platform.common only
```
ALLOWED:   from platform.tools.verify import VerificationEngine
ALLOWED:   from platform.kernel.registry import ModuleRegistry
FORBIDDEN: from platform.runtimes.ai_runtime import ResourceIntelligenceService
```

---

## Cross-Runtime Communication Protocol

When runtime A needs data from runtime B:

```
Step 1: runtime_a publishes PlatformEvent via kernel.events
Step 2: kernel dispatches to all subscribers
Step 3: runtime_b handler processes and may publish a response event
Step 4: runtime_a receives response via correlated subscription
```

Correlation is via `PlatformEvent.correlation_id: UUID`.

**No RPC.** No function calls across runtimes. Events only.

---

## Layer Dependency Rules

### DR-L001 — Upward dependencies forbidden
If `layer(A) < layer(B)`, then `B` may not import `A`.

Layer ordering (lower = more fundamental):
```
0 = stdlib
1 = common
2 = kernel
3 = services
4 = runtimes
5 = sdk
6 = products
```

### DR-L002 — Lateral dependencies forbidden within runtimes
`runtime_a` cannot import from `runtime_b` at the same layer.

### DR-L003 — Extension point exception
Runtimes may declare `extension_points` that other runtimes register implementations for.
The registration must go through `kernel.registry`, not direct imports.

---

## Dependency Matrix — Runtime Level

| From \ To | event_rt | resource_rt | provider_rt | knowledge_rt | ai_rt | workflow_rt | orchestration_rt |
|-----------|----------|------------|-------------|-------------|-------|------------|-----------------|
| event_rt | — | F | F | F | F | F | F |
| resource_rt | E | — | F | F | F | F | F |
| provider_rt | E | F | — | F | F | F | F |
| knowledge_rt | E | F | F | — | F | F | F |
| ai_rt | E | E | E | E | — | F | F |
| workflow_rt | E | E | E | E | E | — | F |
| orchestration_rt | E | E | E | E | E | E | — |

E = events only (via kernel.events); F = Forbidden

---

## Exception Process

A dependency rule exception requires:
1. An Architecture Decision Record (ADR) in 34-ARCHITECTURE-DECISIONS.md
2. Approval from Architecture Team
3. A time limit (not permanent exceptions)
4. A ticket tracking removal of the exception

Current exceptions: **none**
