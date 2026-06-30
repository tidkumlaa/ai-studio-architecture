# KNW-KC-ARCH-033 — Lifecycle Extensions

**Phase:** 3.0C  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Lifecycle Extensions define domain-specific additional lifecycle events, guards, and policies that extend the base 7-state lifecycle (DRAFT→...→ARCHIVED) without replacing it. Every domain may register extensions; the base lifecycle is immutable.

---

## Base Lifecycle (Immutable)

From Phase 3.0A `13-LIFECYCLE`:
```
DRAFT → REVIEW → VERIFIED → APPROVED → CANONICAL → DEPRECATED → ARCHIVED
```

The base lifecycle is defined in `34-STATE-MACHINES`. This document defines extensions only.

---

## Extension Types

| Extension Type | Description |
|---------------|-------------|
| `SUBSTATUS` | Sub-states within a base state (does not change the base state) |
| `GATE` | Additional guard conditions for a specific transition |
| `HOOK` | Actions triggered on entering/exiting a state |
| `BYPASS` | Emergency bypass of a gate (requires special authority) |

---

## Registered Extensions by Domain

### Domain: Architecture Documents

```yaml
extension: kos.arch.lifecycle
applies_to: [ARCHITECTURE, DECISION, REQUIREMENT, SPECIFICATION]

substatus:
  REVIEW:
    - UNDER_REVIEW          # reviewer assigned, active review in progress
    - REVISION_REQUESTED    # reviewer has requested changes
    - REVIEW_COMPLETE       # reviewer approved, awaiting VERIFIED transition

gates:
  VERIFIED:
    - "All MUST requirements are covered by at least one architecture object"
    - "ADR exists for every MAJOR architectural choice in this document"

hooks:
  on_enter.CANONICAL:
    - "Emit knowledge.architecture.frozen event"
    - "Notify all dependent modules via event"
  on_enter.DEPRECATED:
    - "Check for unresolved implements edges — warn if any module still references this"
```

### Domain: Platform Modules

```yaml
extension: platform.module.lifecycle
applies_to: [MODULE, SERVICE]

substatus:
  CANONICAL:
    - STABLE                # in production, no known issues
    - MONITORING            # in production, under active monitoring
    - INCIDENT              # in production, active incident affecting this module

gates:
  VERIFIED:
    - "Test coverage >= target_coverage for this module's stability tier"
    - "No cyclic dependency with any other module"
  CANONICAL:
    - "Boot time within budget (see 37-PERFORMANCE-BUDGET)"
    - "At least one integration test passing in CI"

hooks:
  on_enter.DEPRECATED:
    - "Notify all consumer modules"
    - "Set EOL date = now + 90 days"
```

### Domain: APIs

```yaml
extension: platform.api.lifecycle
applies_to: [API]

substatus:
  DEPRECATED:
    - SUNSET_NOTICE_SENT    # 90-day notice sent
    - SUNSET_ACTIVE         # past sunset date, requests still served
    - SUNSET_COMPLETE       # endpoint removed

gates:
  APPROVED:
    - "OpenAPI spec exists and validates"
    - "At least one client example documented"
  DEPRECATED:
    - "Successor API exists at APPROVED or above"
    - "Migration guide written"

hooks:
  on_enter.DEPRECATED:
    - "Set Deprecation header on all responses"
    - "Emit api.deprecated event with sunset_date"
```

### Domain: Evidence Records

```yaml
extension: kos.evidence.lifecycle
applies_to: [evidence records — not KnowledgeObjects themselves]

substatus:
  ACTIVE:
    - FRESH                 # freshness_score >= 0.70
    - DEGRADING             # freshness_score 0.30..0.70
    - STALE                 # freshness_score < 0.30

hooks:
  on_enter_substatus.STALE:
    - "Emit evidence.record.stale event"
    - "Decrement parent object quality score"
```

### Domain: Datasets

```yaml
extension: financial.dataset.lifecycle
applies_to: [DATASET]

substatus:
  VERIFIED:
    - SCHEMA_VALIDATED
    - QUALITY_ASSESSED
    - READY_FOR_USE

gates:
  APPROVED:
    - "Dataset quality score >= 0.80"
    - "License field non-empty"
    - "Provenance documented"
```

---

## Bypass Policy

Emergency bypass of a lifecycle gate:

```yaml
bypass:
  authority: "Architecture Board (unanimous vote)"
  conditions:
    - "Production incident requiring immediate fix"
    - "Legal or compliance deadline"
  requirements:
    - "Bypass reason documented in knowledge object"
    - "Post-incident evidence filed within 7 days"
    - "Bypass audit log entry created"
  max_duration: "90 days — object must reach target state or be reverted"
```

---

## Extension Registration Format

```yaml
# lifecycle-extensions/platform.module.lifecycle.yaml
extension_id: "platform.module.lifecycle"
applies_to: [MODULE, SERVICE]
owner: "team:platform-core"
version: "1.0.0"
status: ACTIVE

substatus:
  - state: CANONICAL
    substates: [STABLE, MONITORING, INCIDENT]

gates:
  - transition: "REVIEW → VERIFIED"
    conditions:
      - "tests.coverage >= module.coverage_target"

hooks:
  - event: "on_enter.DEPRECATED"
    action: "notify_consumers"
    async: true
```

---

## Lifecycle Extension Protocol

```python
class LifecycleExtensionRegistry(Protocol):
    def register_extension(self, ext: LifecycleExtension) -> str: ...
    def get_extensions(self, object_type: KnowledgeObjectType) -> list[LifecycleExtension]: ...
    def evaluate_gates(
        self, object_id: str, transition: tuple[str, str]
    ) -> GateEvaluationResult: ...
    def trigger_hooks(self, object_id: str, event: str) -> list[HookResult]: ...
    def set_substatus(self, object_id: str, substatus: str) -> None: ...
    def get_substatus(self, object_id: str) -> str | None: ...
    def request_bypass(self, object_id: str, gate: str, reason: str) -> BypassRequest: ...
```

---

## Cross-References

- Base lifecycle → `34-STATE-MACHINES`
- Evidence substatus → `10-EVIDENCE-ENGINE`
- Gate conditions use quality → `11-QUALITY-ENGINE`
- Hooks emit events → `18-EVENT-REGISTRY`
- Governance authority → Phase 2.1D.0 `33-PLATFORM-GOVERNANCE`
