---
knowledge_id: KNW-PLAT-ARCH-034
title: "Architecture Decision Records"
status: AUTHORITATIVE
phase: "2.1D.0"
version: "1.0.0"
created: "2026-06-29"
purpose: "Record all significant architecture decisions with context, options considered, and rationale"
canonical_source: "architecture/docs/platform-consolidation/34-ARCHITECTURE-DECISIONS.md"
dependencies:
  - "01-VISION.md"
  - "10-DEPENDENCY-RULES.md"
related_documents:
  - "33-PLATFORM-GOVERNANCE.md"
  - "35-ARCHITECTURE-FREEZE.md"
acceptance_criteria:
  - "Every ADR has a status: ACCEPTED, SUPERSEDED, or DEPRECATED"
  - "Every ADR states context, decision, and consequences"
  - "Superseded ADRs reference their replacement"
verification_checklist:
  - "[ ] All major decisions recorded"
  - "[ ] No contradiction between ADRs"
  - "[ ] Consequences (positive and negative) documented"
future_extensions:
  - "ADR graph visualization (which decisions depend on others)"
  - "ADR review schedule"
---

# Architecture Decision Records

## ADR Format

```
ADR-NNN: Title
Status: ACCEPTED | SUPERSEDED by ADR-XXX | DEPRECATED
Date: YYYY-MM-DD
Context: Why did we need to make a decision?
Decision: What did we decide?
Consequences: What are the positive and negative outcomes?
```

---

## ADR-001: Single Canonical Repository

**Status:** ACCEPTED  
**Date:** 2026-06-29

**Context:**  
Platform code was spread across three directories: `platform/`, `platform_kernel/`, `platform_sdk/`.
This caused import path duplication, unclear ownership, and diverging versions.

**Decision:**  
Consolidate all platform code into `platform/` as the single canonical source.
`platform_kernel/` and `platform_sdk/` will be merged into `platform/kernel/` and `platform/sdk/`
respectively. No new code will be written in the old directories.

**Consequences:**  
(+) Single `pyproject.toml`, single test runner, single version  
(+) Import paths become stable and predictable  
(-) Migration required to rewrite all `platform_kernel.*` and `platform_sdk.*` imports  
(-) Products referencing old paths must be updated

---

## ADR-002: Event Bus for Cross-Runtime Communication

**Status:** ACCEPTED  
**Date:** 2026-06-29

**Context:**  
Runtimes needed to communicate (e.g., AI runtime learning about provider health changes).
Direct function calls would create tight coupling and violate layer isolation.

**Decision:**  
All cross-runtime communication must use the `EventBus`. No runtime may call another
runtime's Python functions directly. Capabilities are resolved at boot time, not at call time.

**Consequences:**  
(+) Runtimes are independently deployable in theory  
(+) Communication is auditable (all events go through one bus)  
(+) Decoupled boot order is easier to test  
(-) Debugging requires tracing events, not call stacks  
(-) Eventual consistency — subscriber may lag behind publisher

---

## ADR-003: Pydantic for All Data Models

**Status:** ACCEPTED  
**Date:** 2026-06-29

**Context:**  
Platform modules needed to exchange data in a validated, serializable way.
Plain dicts caused runtime errors when fields were missing or wrong type.

**Decision:**  
All external-facing data structures (API requests/responses, cross-module data) use
Pydantic `BaseModel`. Internal-only immutable data uses `@dataclass(frozen=True)`.
No plain dicts as function return types in the public interface.

**Consequences:**  
(+) Runtime validation at boundaries  
(+) JSON serialization built-in  
(+) IDE autocomplete and type checking  
(-) Pydantic is a runtime dependency (adds ~10 MB to package)  
(-) `protected_namespaces` must be set for models with `model_id` fields

---

## ADR-004: Protocol-Based Capability Interfaces

**Status:** ACCEPTED  
**Date:** 2026-06-29

**Context:**  
Capabilities needed to be resolved across runtime boundaries without coupling
the consumer to the provider's concrete class.

**Decision:**  
All capability interfaces are Python `Protocol` classes. Consumers type-hint against
the Protocol, not the implementation. Implementations are registered via `CapabilityRegistry`.

**Consequences:**  
(+) Implementations can be swapped without changing consumers  
(+) Type checking still works (structural subtyping)  
(+) No import coupling between runtimes  
(-) Protocol matching can hide implementation errors that ABCs would catch  
(-) Runtime type checking is the programmer's responsibility

---

## ADR-005: ABC for Service Contracts

**Status:** ACCEPTED  
**Date:** 2026-06-29

**Context:**  
Services (Auth, Audit, Metrics, etc.) needed strong contracts to ensure all
implementations provide required functionality.

**Decision:**  
Services use Python `ABC` (Abstract Base Class) with `@abstractmethod`. This forces
all implementations to implement the full interface at class definition time,
not at runtime as with Protocol.

**Consequences:**  
(+) Incomplete implementations fail at import time, not call time  
(+) Clear distinction between "service interface" (ABC) and "capability interface" (Protocol)  
(-) ABC requires inheriting from the abstract class (nominal subtyping)  
(-) Cannot use ABC for cross-runtime capabilities (would create import coupling)

---

## ADR-006: Provider Plugin Pattern

**Status:** ACCEPTED  
**Date:** 2026-06-29

**Context:**  
AI providers (Anthropic, OpenAI, Google, etc.) needed to be referenced without
hardcoding provider-specific imports throughout the codebase.

**Decision:**  
All provider references use string IDs (`"anthropic"`, `"openai"`, `"google"`).
Provider implementations are registered with `ProviderRegistry` at boot.
No module outside `provider_runtime` imports provider-specific libraries.

**Consequences:**  
(+) Providers can be added without touching any consumer code  
(+) Provider libraries are optional runtime dependencies  
(+) Test mocking is straightforward (replace string ID → mock implementation)  
(-) String IDs are not type-checked at compile time  
(-) Provider capability discovery requires registry lookup at boot

---

## ADR-007: Sequential Runtime Boot Order

**Status:** ACCEPTED  
**Date:** 2026-06-29

**Context:**  
Runtimes have dependencies (AI runtime needs resource quota, provider registry).
Parallel boot could cause race conditions where a runtime tries to use a capability
before it's registered.

**Decision:**  
Runtimes boot sequentially in `boot_order` (1→7). Each runtime must reach READY
state before the next begins. Boot order is: event(1) → resource(2) → provider(3) →
knowledge(4) → ai(5) → workflow(6) → orchestration(7).

**Consequences:**  
(+) Deterministic, debuggable boot sequence  
(+) No race conditions in capability registration  
(-) Total boot time is sum of individual boot times (~1.5–2s)  
(-) A slow runtime blocks all higher-order runtimes

---

## ADR-008: Architecture-First, Implementation-Later

**Status:** ACCEPTED  
**Date:** 2026-06-29

**Context:**  
Previous consolidation attempts failed because implementation started before
the target architecture was agreed upon, leading to mid-migration confusion.

**Decision:**  
Phase 2.1D.0 produces ONLY architecture documents. No code is moved, modified,
or written. Implementation begins only after all 37 documents are AUTHORITATIVE
and the Architecture Freeze document is FROZEN.

**Consequences:**  
(+) All stakeholders agree on the end state before any code changes  
(+) Migration can be planned precisely (known file counts, import rewrites)  
(+) Rollback plan is designed before it's needed  
(-) Longer time to first implementation  
(-) Architecture documents may become stale if implementation takes long

---

## ADR-009: Decimal for All Monetary Values

**Status:** ACCEPTED  
**Date:** 2026-06-29

**Context:**  
Cost predictions and budget checks used floating-point arithmetic, causing
precision errors (e.g., `0.1 + 0.2 != 0.3` in float).

**Decision:**  
All monetary values (USD costs, budgets, prices) use Python `Decimal`.
No `float` for any cost or price field. All pricing tables use `Decimal` literals.

**Consequences:**  
(+) Exact decimal arithmetic — no rounding surprises  
(+) Budget comparisons are correct (`Decimal("0") is not None`, avoiding falsy-zero bug)  
(-) Slightly more verbose (`Decimal("0.003")` vs `0.003`)  
(-) Requires explicit conversion when interacting with external APIs that use float

---

## ADR-010: EMA Learning Rate α=0.3

**Status:** ACCEPTED  
**Date:** 2026-06-29

**Context:**  
The token prediction learning engine needed to adapt to new data while not
overreacting to individual outliers. Multiple α values were considered.

**Decision:**  
Use EMA with α=0.3 (new data weight) and β=0.7 (existing knowledge weight).

**Rationale:**  
- α=0.1 → too slow to adapt to new workload patterns
- α=0.5 → too volatile; single outlier shifts prediction significantly  
- α=0.3 → good balance: adapts within ~10 observations, resistant to single outliers

**Consequences:**  
(+) Stable predictions that improve over time  
(+) Resistant to noisy measurements  
(-) Takes ~10 observations to fully reflect a new pattern  
(-) α is not self-tuning; if workload distribution shifts dramatically, α may need adjustment
