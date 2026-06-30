# KNW-KIL-DOC-019 — Knowledge Cortex

**Phase:** 3.0D.0.6
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Cortex is an object's "thinking layer" — the collection of meta-knowledge
about how to reason correctly about this object.

Every object in the knowledge base has factual content (what it is) and
executable rules (what it enforces). The Cortex adds a third layer: guidance
on how to *think* about this object, including what questions it commonly
answers, what mistakes are commonly made, what the typical reasoning path is,
and what the future holds.

This document defines `intelligence.cortex`.

---

## Schema

```yaml
intelligence:
  cortex:
    schema_version: "1.0"

    why:                               # why does this object exist?
      statement: string                # one paragraph: the root cause
      would_happen_without: string     # what breaks if this object didn't exist
      created_to_solve: string         # specific problem it was created to solve

    why_not:                           # why is this object NOT something it could be?
      - not_this: string               # e.g. "not a rate limiter"
        because: string                # why it is explicitly scoped to exclude this
        confused_with: string | null   # knowledge_id of the thing it is confused with

    common_questions:                  # questions humans and AI agents typically ask
      - question: string
        answer: string
        difficulty: TRIVIAL | EASY | MEDIUM | HARD | EXPERT
        requires_context: [string]     # knowledge_ids needed to answer fully

    common_errors:                     # errors that arise from misunderstanding this object
      - error_id: string
        category: DESIGN | IMPLEMENTATION | USAGE | REASONING | INTEGRATION
        description: string
        root_cause: string
        correction: string
        example: string | null

    typical_reasoning:                 # the standard reasoning path for this object
      - step: integer
        question: string               # what the reasoner asks at this step
        answer_source: string          # where to find the answer
        common_wrong_turn: string | null

    decision_history:                  # summary of key decisions (pointers to full records)
      - decision_id: string            # DEC-NNN reference
        summary: string                # one line
        still_valid: boolean
        challenge: string | null       # has anything challenged this decision?

    future_improvements:               # known improvements that haven't been made yet
      - improvement_id: string         # e.g. FI-001
        description: string
        motivation: string
        blocked_by: string | null      # what is blocking this improvement
        target_phase: string | null    # e.g. "Phase 3.0D.3"
        priority: CRITICAL | HIGH | MEDIUM | LOW
```

---

## Canonical Example

```yaml
# For: KNW-PLT-MOD-001 (Quota Manager)
intelligence:
  cortex:
    schema_version: "1.0"

    why:
      statement: >
        The Quota Manager was created after a Platform v1 production incident
        where a single misconfigured tenant consumed 100% of platform CPU,
        causing a cascading failure across all other tenants. The platform needed
        a guaranteed, pre-enforcement mechanism that was on the critical request
        path — not a retroactive enforcement that would allow damage before acting.
      would_happen_without: >
        Without Quota Manager, the only resource protection is at the OS/infrastructure
        level. This means: (a) tenants can burst freely until infrastructure limits
        kick in, (b) there is no per-tenant accounting for compliance purposes,
        (c) burst events cause cross-tenant interference as they do in Platform v1.
      created_to_solve: >
        The "noisy neighbor" problem in multi-tenant platform operation — where one
        tenant's resource consumption degrades the experience of others.

    why_not:
      - not_this: "not a rate limiter (req/s)"
        because: >
          Rate limiting controls request velocity per time window. Quota enforcement
          controls cumulative resource consumption over periods. A rate limiter allows
          a tenant to make 1000 req/s but each consuming 0.001 CPU — the total CPU
          consumption is what Quota Manager tracks.
        confused_with: "KNW-ALG-ALG-009"
      - not_this: "not the quota configurator"
        because: >
          Quota configuration is a governance function (who gets how much).
          Quota enforcement is an operational function (ensuring they don't exceed it).
          Separating them prevents the enforcer from modifying the limits it enforces.
        confused_with: null
      - not_this: "not a billing engine"
        because: >
          Billing converts usage to cost. Quota enforcement ensures usage stays
          within limits. They share the same usage data but serve different purposes.
        confused_with: "KNW-FIN-SVC-001"

    common_questions:
      - question: "What happens when a tenant is at exactly their quota limit?"
        answer: >
          The next request with requested_amount > 0 will receive DENY.
          Requests with requested_amount = 0 (probing) receive ALLOW with remaining = 0.
        difficulty: MEDIUM
        requires_context: [KNW-PLT-MOD-001]
      - question: "Is the quota check atomic?"
        answer: >
          Yes. The check and state update are performed atomically via Registry's
          compare-and-swap operation. No two concurrent checks can both ALLOW
          when only one unit of quota remains.
        difficulty: HARD
        requires_context: [KNW-PLT-MOD-001, KNW-RT-RT-001]
      - question: "What does Quota Manager do when Registry is unavailable?"
        answer: >
          The circuit breaker activates and returns DENY for all requests until
          Registry recovers. This is the conservative choice — better to deny
          valid requests briefly than to allow unlimited consumption.
        difficulty: MEDIUM
        requires_context: [KNW-PLT-MOD-001, KNW-RT-RT-001]

    common_errors:
      - error_id: CE-001
        category: REASONING
        description: "AI states that Quota Manager can update quota limits"
        root_cause: "Conflation of enforcement and configuration roles"
        correction: "Quota Manager is READ-ONLY to quota limits. Admin API owns writes."
        example: "❌ 'Call QuotaManager.setLimit(tenant, CPU, 8.0) to increase quota'"
      - error_id: CE-002
        category: DESIGN
        description: "Architect bypasses Quota Manager for 'trusted' internal services"
        root_cause: "Assumption that internal services don't need quota control"
        correction: "All platform services go through Quota Manager — including internal ones. Internal services can consume as much as external ones."
        example: null
      - error_id: CE-003
        category: INTEGRATION
        description: "Service calls Quota Manager after allocating the resource"
        root_cause: "Misunderstanding of the pre-enforcement requirement"
        correction: "Quota check MUST happen before resource allocation. ALLOW → allocate. Never allocate → then check."
        example: "❌ resource = allocate(); if quota.check() == DENY: release(resource)"

    typical_reasoning:
      - step: 1
        question: "What is Quota Manager responsible for?"
        answer_source: "self_describing.purpose + ai_context.short_summary"
        common_wrong_turn: "Assuming it handles both enforcement AND configuration"
      - step: 2
        question: "What does it depend on?"
        answer_source: "self_describing.dependencies + genome.relationships"
        common_wrong_turn: "Forgetting KNW-RT-RT-001 is a HARD dependency"
      - step: 3
        question: "What does it promise?"
        answer_source: "dna.core_behavior.output_contract + dna.fundamental_constraints"
        common_wrong_turn: "Ignoring the P99 < 5ms constraint"
      - step: 4
        question: "What can go wrong?"
        answer_source: "risk.failure_modes + cortex.common_errors"
        common_wrong_turn: null
      - step: 5
        question: "How does it relate to other objects?"
        answer_source: "semantic.provides + semantic.conflicts + alternatives"
        common_wrong_turn: "Confusing with KNW-ALG-ALG-009 Rate Limiter"

    decision_history:
      - decision_id: DEC-001
        summary: "Synchronous enforcement with in-process config cache"
        still_valid: true
        challenge: "Registry P99 rose briefly to 4ms in Jan 2026; circuit breaker handled it"
      - decision_id: DEC-002
        summary: "Named parameters in QuotaCheck v2.0.0"
        still_valid: true
        challenge: null

    future_improvements:
      - improvement_id: FI-001
        description: "Async quota pre-authorization for batch operations"
        motivation: "Batch jobs repeatedly acquire quota in a tight loop"
        blocked_by: "Requires async interface (breaking change)"
        target_phase: "Phase 3.0D.1"
        priority: MEDIUM
      - improvement_id: FI-002
        description: "AI-driven quota limit suggestions based on usage patterns"
        motivation: "Static quota limits are hard to tune optimally"
        blocked_by: "AI Runtime (Phase 3.0E) not yet available"
        target_phase: "Phase 3.0E"
        priority: LOW
```

---

## Rules

| Rule | Statement |
|------|-----------|
| KIL-108 | `cortex.why.would_happen_without` must be non-trivial — "nothing would work" is not acceptable |
| KIL-109 | Every `why_not` entry must reference the knowledge_id it is confused with (if one exists) |
| KIL-110 | `common_errors` must be derived from real observed errors, not hypothetical ones |
| KIL-111 | `typical_reasoning` must be a linear sequence with no cycles |
| KIL-112 | Every CANONICAL object must have ≥ 3 `common_questions` in cortex |

---

## Cross-References

- AI context → `04-AI-CONTEXT-LAYER`
- Thinking layer → `20-KNOWLEDGE-THINKING-LAYER`
- Explanation layer → `21-KNOWLEDGE-EXPLANATION-LAYER`
- Memory (failures) → `09-KNOWLEDGE-MEMORY`
