# KNW-KIL-DOC-016 — Knowledge Tradeoff Model

**Phase:** 3.0D.0.6
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Every significant architecture decision embedded in a Knowledge Object must
record the alternatives that were considered, the tradeoffs evaluated, and the
reasoning that led to the chosen approach.

Without recorded tradeoffs, future architects and AI agents cannot understand
WHY a design is the way it is — they can only see WHAT it is. This leads to
repeated debates, accidental reversals of deliberate decisions, and inability
to evaluate whether a new constraint changes the original tradeoff.

This document defines `intelligence.tradeoffs`.

---

## Schema

```yaml
intelligence:
  tradeoffs:
    - tradeoff_id: string              # e.g. TO-001
      title: string
      context: string                  # what problem or decision created this tradeoff
      decision_date: string            # ISO 8601

      options:
        - option_id: string            # e.g. OPT-A
          name: string
          description: string
          pros:
            - dimension: string        # e.g. "latency", "simplicity", "correctness"
              description: string
              evidence: string | null  # knowledge_id or document reference
          cons:
            - dimension: string
              description: string
              evidence: string | null
          score: float                 # 0.0–1.0 scoring of this option

      chosen_option: string            # option_id of the selected approach
      decision_rationale: string       # why this option was chosen over others
      decision_authority: string       # who made the decision
      decision_id: string | null       # ADR/RFC reference

      consequences:
        positive:
          - string
        negative:
          - string
        neutral:
          - string

      revisit_conditions:              # when should this tradeoff be re-evaluated?
        - condition: string
          trigger: string              # what would trigger this condition
```

---

## Canonical Example

```yaml
# For: KNW-PLT-MOD-001 (Quota Manager)
intelligence:
  tradeoffs:
    - tradeoff_id: TO-001
      title: "Synchronous vs Asynchronous Quota Check"
      context: >
        The initial design required a decision: should quota checks be
        synchronous (blocking the request until quota is confirmed) or
        asynchronous (allowing the request immediately, enforcing later)?
        This affects latency, correctness, and implementation complexity.
      decision_date: "2025-01-10"

      options:
        - option_id: OPT-A
          name: "Synchronous with Registry"
          description: "Every request blocks on a synchronous registry read before proceeding"
          pros:
            - dimension: "correctness"
              description: "Quota is always accurate — no over-allowance possible"
              evidence: "KNW-TEST-TST-001"
            - dimension: "auditability"
              description: "Every decision is immediately recorded"
              evidence: null
          cons:
            - dimension: "latency"
              description: "Adds Registry round-trip to every request"
              evidence: "KNW-TEST-BENCH-001"
            - dimension: "coupling"
              description: "Request path depends on Registry availability"
              evidence: null
          score: 0.82

        - option_id: OPT-B
          name: "In-process cache with async Registry sync"
          description: "Quota check uses in-process cache; Registry syncs asynchronously"
          pros:
            - dimension: "latency"
              description: "Cache hit < 1ms — no Registry round-trip on hot path"
              evidence: "KNW-TEST-BENCH-001"
            - dimension: "availability"
              description: "Works even if Registry is briefly unavailable"
              evidence: null
          cons:
            - dimension: "correctness"
              description: "Cache staleness window allows brief over-allowance"
              evidence: null
            - dimension: "complexity"
              description: "Cache invalidation adds implementation complexity"
              evidence: null
          score: 0.74

        - option_id: OPT-C
          name: "Asynchronous (allow-then-enforce)"
          description: "Allow all requests, enforce quotas retroactively"
          pros:
            - dimension: "latency"
              description: "Zero overhead on request path"
              evidence: null
          cons:
            - dimension: "correctness"
              description: "Tenants can burst above quota before enforcement"
              evidence: null
            - dimension: "compliance"
              description: "Violates KNW-PLT-REQ-001 which requires pre-enforcement"
              evidence: "KNW-PLT-REQ-001"
          score: 0.25

      chosen_option: OPT-A
      decision_rationale: >
        OPT-A chosen because correctness is non-negotiable (KNW-PLT-REQ-001
        requires pre-enforcement). OPT-B considered acceptable but cache
        invalidation complexity rejected given team capacity. OPT-C rejected
        as it violates the requirement. The P99 < 5ms constraint was validated
        against OPT-A using in-process quota state caching (see KNW-TEST-BENCH-001).
      decision_authority: "architecture-board"
      decision_id: "ADR-001"

      consequences:
        positive:
          - "Quota enforcement is always accurate"
          - "Audit trail is complete — every decision logged"
          - "Simpler implementation than cache approach"
        negative:
          - "Every request incurs Registry latency (mitigated by P99 < 5ms requirement)"
          - "Registry becomes a HARD dependency on the critical path"
        neutral:
          - "Cache still used for configuration (quota limits); only state read is synchronous"

      revisit_conditions:
        - condition: "Registry P99 consistently > 3ms"
          trigger: "KNW-PLT-MON-001 alert REGISTRY-P99-HIGH sustained for 7 days"
        - condition: "Request throughput exceeds 100K req/s"
          trigger: "Platform scale milestone reached"

    - tradeoff_id: TO-002
      title: "Positional vs Named Parameters in QuotaCheck Interface"
      context: >
        v1.x used positional parameters. Adding STORAGE as fourth resource type
        in v2.1.0 required a decision on parameter passing style.
      decision_date: "2025-08-15"

      options:
        - option_id: OPT-A
          name: "Named parameters"
          description: "QuotaCheck(tenant_id=str, resource_type=str, requested_amount=float)"
          pros:
            - dimension: "extensibility"
              description: "New parameters can be added without breaking callers"
              evidence: null
            - dimension: "clarity"
              description: "Interface is self-documenting"
              evidence: null
          cons:
            - dimension: "migration"
              description: "Breaking change for all existing callers"
              evidence: null
          score: 0.85

        - option_id: OPT-B
          name: "Keep positional"
          description: "QuotaCheck(tenant_id: str, resource_type: str, amount: float)"
          pros:
            - dimension: "migration"
              description: "No change for existing callers"
              evidence: null
          cons:
            - dimension: "extensibility"
              description: "Adding 4th argument breaks callers in future"
              evidence: null
          score: 0.50

      chosen_option: OPT-A
      decision_rationale: "Named parameters chosen to prevent future breaking changes. Cost: one migration now vs many migrations later."
      decision_authority: "platform-team"
      decision_id: "ADR-019"
      consequences:
        positive: ["No future breaking changes for new resource dimensions"]
        negative: ["All v1.x callers require migration to v2.0.0"]
        neutral: []
      revisit_conditions:
        - condition: "Interface needs to support batch quota checks"
          trigger: "Performance requirement for batch operations"
```

---

## Rules

| Rule | Statement |
|------|-----------|
| KIL-093 | Every architecture decision recorded in the object's DNA must have a corresponding `tradeoff` entry |
| KIL-094 | `chosen_option` must be one of the listed `options[*].option_id` |
| KIL-095 | Every `tradeoff` with `decision_id` set must reference an existing DECISION object |
| KIL-096 | `revisit_conditions` must be defined for any tradeoff involving performance or scale |
| KIL-097 | The rejected options must be honest — cons must be acknowledged, not minimized |

---

## Cross-References

- Decision model → `17-KNOWLEDGE-DECISION-MODEL`
- Alternative model → `18-KNOWLEDGE-ALTERNATIVE-MODEL`
- Cortex (why/why-not) → `19-KNOWLEDGE-CORTEX`
- Evolution (change history) → `11-KNOWLEDGE-EVOLUTION`
