# KNW-KIL-DOC-017 — Knowledge Decision Model

**Phase:** 3.0D.0.6
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Every significant decision that shaped a Knowledge Object — including decisions
made by AI agents at runtime — must be recorded in a structured decision log.

The Decision Model captures the context, reasoning process, alternatives
considered, confidence in the decision, and its observable outcome.

This enables AI agents to learn from past decisions and avoid repeating known
mistakes.

This document defines `intelligence.decisions`.

---

## Schema

```yaml
intelligence:
  decisions:
    - decision_id: string              # e.g. DEC-001
      title: string
      type: ARCHITECTURE | DESIGN | POLICY | PROCESS | IMPLEMENTATION | AI_GENERATED
      status: PROPOSED | APPROVED | SUPERSEDED | REVERTED

      context:
        problem: string                # what problem was being solved
        drivers: [string]              # constraints and requirements that drove the decision
        constraints: [string]          # hard limits that narrowed the option space
        assumptions: [string]          # what was assumed to be true

      reasoning_process:
        method: ANALYSIS | EXPERIMENT | CONSENSUS | EXPERT | AI_REASONING
        evidence_used: [string]        # knowledge_ids of supporting evidence
        reasoning_steps: [string]      # ordered list of reasoning steps

      options_considered: [string]     # tradeoff_id or brief descriptions

      decision:
        choice: string                 # what was decided
        rationale: string              # why this choice
        confidence: float              # 0.0–1.0 decision confidence
        authority: string              # who approved this
        decided_at: string             # ISO 8601

      outcome:
        status: CONFIRMED | UNKNOWN | CHALLENGED | REVERTED
        observed_effects: [string]     # what actually happened
        evaluation_date: string | null
        notes: string | null

      superseded_by: string | null     # decision_id that replaces this
```

---

## Decision Types

| Type | Meaning |
|------|---------|
| ARCHITECTURE | Structural choices (patterns, interfaces, boundaries) |
| DESIGN | Implementation-level choices |
| POLICY | Governance and process decisions |
| PROCESS | How work is done (CI, review, naming) |
| IMPLEMENTATION | Runtime behavior choices |
| AI_GENERATED | Decision made autonomously by an AI agent |

---

## Canonical Example

```yaml
# For: KNW-PLT-MOD-001 (Quota Manager)
intelligence:
  decisions:
    - decision_id: DEC-001
      title: "Enforce quotas synchronously on the request path"
      type: ARCHITECTURE
      status: APPROVED

      context:
        problem: >
          The platform needed to enforce per-tenant resource quotas without
          allowing any tenant to exceed their allocation under concurrent load.
        drivers:
          - "KNW-PLT-REQ-001 requires pre-enforcement (not retroactive)"
          - "P99 < 5ms (KNW-META-STD-007) must be maintained"
          - "Multi-tenant fairness is a platform guarantee"
        constraints:
          - "Cannot use asynchronous enforcement — violates KNW-PLT-REQ-001"
          - "Registry must be available on the hot path"
        assumptions:
          - "Registry P99 will remain < 3ms under nominal load"
          - "In-process cache for quota config reduces Registry reads by 90%"

      reasoning_process:
        method: ANALYSIS
        evidence_used:
          - KNW-TEST-BENCH-001
          - KNW-PLT-REQ-001
        reasoning_steps:
          - "Identify requirement: pre-enforcement required"
          - "Evaluate async: rejected — violates requirement"
          - "Evaluate sync with cache: accepted — P99 meets target"
          - "Validate with benchmark: KNW-TEST-BENCH-001 confirms P99 = 2.3ms"
          - "Confirm with architecture board: ADR-001 approved"

      options_considered: ["TO-001 OPT-A", "TO-001 OPT-B", "TO-001 OPT-C"]

      decision:
        choice: "Synchronous quota check with in-process quota config cache"
        rationale: "Only approach satisfying both the pre-enforcement requirement and P99 constraint"
        confidence: 0.95
        authority: "architecture-board"
        decided_at: "2025-01-10T14:00:00Z"

      outcome:
        status: CONFIRMED
        observed_effects:
          - "P99 = 2.3ms in production (target < 5ms)"
          - "Zero quota overruns in 500+ days of operation"
          - "Registry dependency caused 2 brief quota unavailability incidents (mitigated)"
        evaluation_date: "2026-06-01"
        notes: "Decision holds. Registry incidents confirmed the need for circuit breaker (added v2.0.0)"

      superseded_by: null

    - decision_id: DEC-002
      title: "Switch to named parameters in QuotaCheck interface"
      type: DESIGN
      status: APPROVED

      context:
        problem: "Positional parameters would require breaking change for every new resource dimension"
        drivers:
          - "STORAGE resource dimension requires 4th parameter"
          - "Future dimensions (NETWORK, GPU) anticipated"
        constraints:
          - "Must maintain semantic equivalence with v1.x behavior"
        assumptions:
          - "All callers can be migrated before v2.0.0 release"

      reasoning_process:
        method: CONSENSUS
        evidence_used: []
        reasoning_steps:
          - "Identify: positional params require caller migration for every new dimension"
          - "Identify: named params are a one-time migration cost"
          - "Decision: accept one migration now to avoid many in future"

      options_considered: ["TO-002 OPT-A", "TO-002 OPT-B"]

      decision:
        choice: "Named parameters in v2.0.0"
        rationale: "One breaking change now prevents N breaking changes in future"
        confidence: 0.90
        authority: "platform-team"
        decided_at: "2025-08-15T10:00:00Z"

      outcome:
        status: CONFIRMED
        observed_effects:
          - "18 callers migrated successfully"
          - "v2.1.0 added STORAGE with zero additional breaking changes"
        evaluation_date: "2026-03-01"
        notes: "Decision validated by STORAGE addition in v2.1.0"

      superseded_by: null
```

---

## Rules

| Rule | Statement |
|------|-----------|
| KIL-098 | Every `type: AI_GENERATED` decision must have `outcome.status` evaluated within 30 days |
| KIL-099 | Every `status: REVERTED` decision must record the reversion reason in `outcome.notes` |
| KIL-100 | `confidence < 0.60` decisions require a revisit date in `outcome.evaluation_date` |
| KIL-101 | `outcome.observed_effects` must be populated once the decision is in production |
| KIL-102 | `superseded_by` must reference an existing decision_id when set |

---

## Cross-References

- Tradeoff model → `16-KNOWLEDGE-TRADEOFF-MODEL`
- Cortex (decision history) → `19-KNOWLEDGE-CORTEX`
- Alternative model → `18-KNOWLEDGE-ALTERNATIVE-MODEL`
- Evolution (change history) → `11-KNOWLEDGE-EVOLUTION`
