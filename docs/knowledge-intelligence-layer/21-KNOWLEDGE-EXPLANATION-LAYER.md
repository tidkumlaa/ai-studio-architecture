# KNW-KIL-DOC-021 — Knowledge Explanation Layer

**Phase:** 3.0D.0.6
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The same Knowledge Object must be explainable at multiple levels — to an
executive, an architect, a developer, an operator, and an AI agent — each with
different vocabulary, focus, and depth requirements.

The Explanation Layer pre-computes audience-appropriate explanations for every
object, ensuring that AI agents can immediately produce the right level of
explanation without deriving it from raw YAML.

This document defines `intelligence.explanation`.

---

## Audience Types

| Audience | Focus | Vocabulary | Depth |
|----------|-------|-----------|-------|
| EXECUTIVE | Business value, risk | Business terms | Minimal |
| ARCHITECT | Design rationale, patterns, trade-offs | Architecture terms | Deep |
| DEVELOPER | How to use it, API, integration | Implementation terms | Practical |
| OPERATOR | How to monitor, debug, operate | Operational terms | Operational |
| AI_AGENT | Machine-processable, factual | Structured terms | Complete |

---

## Schema

```yaml
intelligence:
  explanation:
    schema_version: "1.0"

    audiences:
      executive:
        summary: string                # ≤ 3 sentences, business language
        business_value: string         # why does this matter to the business?
        risk_if_absent: string         # what risk does this object mitigate?
        metrics: [string]              # key business metrics it affects

      architect:
        summary: string                # design rationale + pattern + placement
        design_pattern: string         # pattern it follows
        placement: string              # where it sits in the architecture
        key_decisions: [string]        # most important design decisions
        integration_points: [string]   # how it connects to the rest
        future_evolution: string       # anticipated changes

      developer:
        summary: string                # practical usage summary
        how_to_use: string             # one-paragraph usage guide
        api_surface: string            # key interfaces
        common_pitfalls: [string]      # what to avoid
        example_code_comment: string   # not code — comment describing expected usage
        debugging_hints: [string]      # how to diagnose problems

      operator:
        summary: string                # operational summary
        monitoring: string             # what to monitor
        alerts: [string]               # alert names/conditions
        failure_behavior: string       # how it behaves when failing
        recovery_steps: string         # how to restore service
        health_indicators: [string]    # what healthy looks like

      ai_agent:
        summary: string                # structured factual summary
        object_class: string           # type + namespace + pattern
        primary_capability: string     # one-sentence capability
        contract: string               # input → output contract
        critical_constraints: [string] # hard limits
        dependencies: [string]         # HARD dependencies
        scope_boundaries: [string]     # what it does NOT do
        confidence_in_claims: float    # 0.0–1.0
```

---

## Canonical Example

```yaml
# For: KNW-PLT-MOD-001 (Quota Manager)
intelligence:
  explanation:
    schema_version: "1.0"

    audiences:
      executive:
        summary: >
          The Quota Manager ensures that no single customer can consume an
          unfair share of the platform's computing resources, protecting all
          customers from the "noisy neighbor" problem. It is a critical
          component of the platform's service level guarantee.
        business_value: >
          Enables the platform to offer fair, predictable performance to all
          customers at all times. Without it, one misconfigured tenant can
          degrade the experience of all others — as occurred in the Platform v1
          incident (2024-Q3).
        risk_if_absent: >
          Single-tenant resource starvation events. SLA breaches for other
          tenants. Potential loss of enterprise customers. Regulatory compliance
          risk (resource fairness obligations).
        metrics:
          - "Tenant quota violations per day (target: 0)"
          - "Cross-tenant performance degradation events (target: 0)"
          - "Quota check latency P99 (target: < 5ms)"

      architect:
        summary: >
          KNW-PLT-MOD-001 implements the Guard pattern (KNW-META-PAT-003) for
          per-tenant resource enforcement. It sits on the platform's request
          critical path, synchronously checking quota state via the Registry
          before any resource is allocated.
        design_pattern: "Guard (KNW-META-PAT-003)"
        placement: >
          On the platform request path, between the API Gateway and the
          resource allocation layer. Called by all service endpoints that
          allocate platform resources.
        key_decisions:
          - "Synchronous enforcement chosen over async to satisfy pre-enforcement requirement (ADR-001)"
          - "Named parameters in v2.0.0 to support extensible resource dimensions (ADR-019)"
          - "Read-only to quota limits — configuration separation of concerns"
        integration_points:
          - "KNW-RT-RT-001 Registry: reads/writes quota state"
          - "KNW-PLT-CFG-001: reads quota limit configuration"
          - "KNW-PLT-MON-001: emits QuotaExceededEvent"
        future_evolution: >
          Async pre-authorization interface planned for batch operations (v3.0.0).
          AI-driven dynamic quota suggestion in Phase 3.0E.

      developer:
        summary: >
          QuotaManager takes a tenant_id, resource_type, and requested_amount
          and returns ALLOW or DENY with remaining quota. Call it BEFORE allocating
          any resource. It is not idempotent — each call consumes quota.
        how_to_use: >
          Inject QuotaManager into your service. Before allocating any platform
          resource (CPU, MEMORY, REQUESTS, STORAGE), call check(tenant_id,
          resource_type, requested_amount). If ALLOW, proceed with allocation.
          If DENY, return 429 to the caller. Never call after allocation.
        api_surface: >
          check(tenant_id: str, resource_type: ResourceType, requested_amount: float)
          → QuotaDecision(status: ALLOW|DENY, remaining: float)
        common_pitfalls:
          - "Calling check() after allocating the resource (must be before)"
          - "Calling check() twice for one request (double-counts quota)"
          - "Calling setLimit() on QuotaManager (it has no such method)"
          - "Treating DENY as an error (it is a normal business decision)"
        example_code_comment: >
          # Before any resource allocation: check quota.
          # result.status == ALLOW means proceed; DENY means return 429.
          # Do not retry on DENY without releasing prior allocations first.
        debugging_hints:
          - "Check Registry health if quota check P99 > 5ms"
          - "Check QuotaExceededEvent stream if tenants complain of unexpected denials"
          - "Check KNW-PLT-CFG-001 if quota limits appear wrong"

      operator:
        summary: >
          Quota Manager is a critical-path component. Its health directly affects
          all tenant requests. Monitor check latency P99 and QuotaExceeded event rate.
        monitoring: >
          Monitor: quota_check_latency_p99, quota_check_count, quota_denied_count,
          quota_state_sync_lag (Registry sync delay).
        alerts:
          - "QUOTA_CHECK_P99_HIGH: P99 > 4ms for 5 minutes"
          - "QUOTA_DENIED_SPIKE: denied rate > 1% of total checks"
          - "REGISTRY_SYNC_LAG_HIGH: quota state > 1s behind Registry"
          - "QUOTA_MANAGER_DOWN: no health check response for 10s"
        failure_behavior: >
          If Registry is unreachable: circuit breaker activates, all checks return
          DENY until Registry recovers. This is intentional conservative behavior.
        recovery_steps: >
          1. Check Registry health (KNW-RT-RT-001 health endpoint).
          2. If Registry is down, restore Registry first.
          3. Quota Manager recovers automatically when Registry is available.
          4. Verify quota_state_sync_lag returns to < 100ms after recovery.
        health_indicators:
          - "quota_check_latency_p99 < 5ms"
          - "quota_check_error_rate < 0.01%"
          - "quota_state_sync_lag < 500ms"
          - "circuit_breaker_state == CLOSED"

      ai_agent:
        summary: >
          KNW-PLT-MOD-001 is a Guard-pattern MODULE (plt namespace) that enforces
          per-tenant resource quotas synchronously. Input: (tenant_id, resource_type,
          requested_amount). Output: QuotaDecision(ALLOW|DENY, remaining_quota).
          P99 < 5ms. HARD dependency on Registry. Read-only to quota limits.
        object_class: "MODULE / plt / Guard"
        primary_capability: "Enforce per-tenant resource quotas (ALLOW/DENY)"
        contract: "check(tenant_id, resource_type, requested_amount) → ALLOW/DENY + remaining"
        critical_constraints:
          - "P99 < 5ms per quota check (FDC-001)"
          - "Quota state persisted before returning ALLOW (FDC-002)"
          - "Read-only to quota limits (FDC-002)"
          - "Every DENY must emit QuotaExceededEvent (FDC-003)"
        dependencies:
          - "KNW-RT-RT-001 Registry (HARD — quota state)"
          - "KNW-PLT-CFG-001 Config (HARD — quota limits)"
        scope_boundaries:
          - "Does NOT configure quota limits (Admin API)"
          - "Does NOT rate-limit by req/s (KNW-ALG-ALG-009)"
          - "Does NOT handle billing (KNW-FIN-SVC-001)"
        confidence_in_claims: 0.91
```

---

## Rules

| Rule | Statement |
|------|-----------|
| KIL-118 | Every CANONICAL object must have all 5 audience explanations populated |
| KIL-119 | `ai_agent.scope_boundaries` must match `dna.purpose.not_in_scope` exactly |
| KIL-120 | `developer.common_pitfalls` must include all pitfalls from `cortex.common_errors` |
| KIL-121 | `executive.summary` must be intelligible without any technical knowledge |
| KIL-122 | `operator.recovery_steps` must be an ordered, actionable list — not narrative prose |

---

## Cross-References

- AI context → `04-AI-CONTEXT-LAYER`
- Cortex → `19-KNOWLEDGE-CORTEX`
- Thinking layer → `20-KNOWLEDGE-THINKING-LAYER`
- Summarization → `22-KNOWLEDGE-SUMMARIZATION`
