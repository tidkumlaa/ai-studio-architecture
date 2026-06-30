# KNW-KIL-DOC-015 — Knowledge Risk Model

**Phase:** 3.0D.0.6
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Every Knowledge Object must quantify the risk it carries — the likelihood and
impact of failure, the criticality of its role, the danger of depending on it,
and the security and migration risks associated with changing it.

Risk metadata enables safe change management, impact analysis, and AI agents
to reason about consequences before recommending actions.

This document defines `intelligence.risk`.

---

## Risk Categories

| Category | Abbreviation | What It Measures |
|----------|-------------|-----------------|
| Criticality | RC | How central is this object to system operation |
| Failure Impact | RF | What happens when this object fails |
| Dependency Risk | RD | Risk from objects this depends on |
| Security Risk | RS | Security exposure introduced by this object |
| Migration Risk | RM | Risk of changing or replacing this object |

---

## Schema

```yaml
intelligence:
  risk:
    schema_version: "1.0"
    overall_risk_score: float          # 0.0–1.0 (0=no risk, 1=maximum risk)
    risk_tier: CRITICAL | HIGH | MEDIUM | LOW | MINIMAL

    criticality:                       # RC: centrality to system
      score: float                     # 0.0–1.0
      weight: 0.30
      reasoning: string
      indicators:
        - is_on_critical_path: boolean
        - dependent_object_count: integer
        - namespace_coverage: float    # fraction of namespace that depends on this
        - single_point_of_failure: boolean
        - alternatives_available: boolean

    failure_impact:                    # RF: consequences of failure
      score: float
      weight: 0.25
      reasoning: string
      failure_modes:
        - mode_id: string              # e.g. FM-001
          name: string
          trigger: string
          impact_scope: OBJECT | PACKAGE | NAMESPACE | PLATFORM | ECOSYSTEM
          severity: CRITICAL | HIGH | MEDIUM | LOW
          probability: HIGH | MEDIUM | LOW | VERY_LOW
          mitigation: string
          detection: string

    dependency_risk:                   # RD: risk inherited from dependencies
      score: float
      weight: 0.20
      reasoning: string
      high_risk_dependencies: [string] # knowledge_ids with risk_tier >= HIGH
      transitive_risk_score: float     # propagated risk from dependency graph
      single_vendor_risk: boolean

    security_risk:                     # RS: security exposure
      score: float
      weight: 0.15
      reasoning: string
      exposure_vectors:
        - vector_id: string
          name: string
          type: INJECTION | ESCALATION | EXFILTRATION | DENIAL | TAMPERING | REPUDIATION
          likelihood: HIGH | MEDIUM | LOW | VERY_LOW
          impact: CRITICAL | HIGH | MEDIUM | LOW
          mitigated: boolean
          mitigation: string

    migration_risk:                    # RM: cost of changing this object
      score: float
      weight: 0.10
      reasoning: string
      change_cost: TRIVIAL | SMALL | MEDIUM | LARGE | VERY_LARGE
      callers_affected: integer
      breaking_interface_count: integer
      blast_radius: string             # description of change impact scope

    formula: >
      overall_risk_score =
        (criticality.score × 0.30) +
        (failure_impact.score × 0.25) +
        (dependency_risk.score × 0.20) +
        (security_risk.score × 0.15) +
        (migration_risk.score × 0.10)
```

---

## Risk Tier Thresholds

| Tier | Range | Action Required |
|------|-------|----------------|
| CRITICAL | 0.80–1.00 | Architecture board review before any change |
| HIGH | 0.65–0.79 | Team lead review + impact analysis required |
| MEDIUM | 0.45–0.64 | Standard review process |
| LOW | 0.25–0.44 | Normal development process |
| MINIMAL | 0.00–0.24 | No special process required |

---

## Canonical Example

```yaml
# For: KNW-PLT-MOD-001 (Quota Manager)
intelligence:
  risk:
    schema_version: "1.0"
    overall_risk_score: 0.72
    risk_tier: HIGH

    criticality:
      score: 0.85
      weight: 0.30
      reasoning: "On every request path; without it, no quota enforcement exists"
      indicators:
        is_on_critical_path: true
        dependent_object_count: 18
        namespace_coverage: 0.72
        single_point_of_failure: true
        alternatives_available: false

    failure_impact:
      score: 0.80
      weight: 0.25
      reasoning: "Failure means tenants can consume unlimited resources"
      failure_modes:
        - mode_id: FM-001
          name: "Registry unreachable"
          trigger: "KNW-RT-RT-001 becomes unavailable"
          impact_scope: PLATFORM
          severity: CRITICAL
          probability: LOW
          mitigation: "Circuit breaker returns DENY on Registry timeout"
          detection: "KNW-PLT-MON-001 alerts on Registry latency > 10ms"
        - mode_id: FM-002
          name: "Quota state corruption"
          trigger: "Registry write fails after ALLOW decision"
          impact_scope: NAMESPACE
          severity: HIGH
          probability: VERY_LOW
          mitigation: "Two-phase commit with Registry before returning ALLOW"
          detection: "Audit log inconsistency detection (nightly)"

    dependency_risk:
      score: 0.65
      weight: 0.20
      reasoning: "HARD dependency on Registry (HIGH risk) propagates risk"
      high_risk_dependencies: [KNW-RT-RT-001]
      transitive_risk_score: 0.71
      single_vendor_risk: false

    security_risk:
      score: 0.45
      weight: 0.15
      reasoning: "Quota bypass would allow DoS; all vectors are mitigated"
      exposure_vectors:
        - vector_id: SV-001
          name: "Quota bypass via negative amounts"
          type: ESCALATION
          likelihood: LOW
          impact: HIGH
          mitigated: true
          mitigation: "Input validation rejects negative or zero requested_amount"
        - vector_id: SV-002
          name: "Quota state manipulation via direct Registry write"
          type: TAMPERING
          likelihood: VERY_LOW
          impact: CRITICAL
          mitigated: true
          mitigation: "Registry access control; Quota Manager is the only writer"

    migration_risk:
      score: 0.80
      weight: 0.10
      reasoning: "18 callers; ecosystem-level interface; any change is BREAKING"
      change_cost: VERY_LARGE
      callers_affected: 18
      breaking_interface_count: 2
      blast_radius: "Any interface change to QuotaCheck or QuotaDecision requires updating all 18 platform services"
```

---

## Rules

| Rule | Statement |
|------|-----------|
| KIL-087 | Every `failure_mode` must have a `detection` — silent failure is not acceptable |
| KIL-088 | Every `exposure_vector` with `impact: CRITICAL` must be `mitigated: true` at CANONICAL state |
| KIL-089 | `risk_tier: CRITICAL` objects require Architecture Board sign-off before schema changes |
| KIL-090 | `transitive_risk_score` must be recomputed when any HARD dependency's risk changes |
| KIL-091 | `single_point_of_failure: true` requires at least one FM with `impact_scope: PLATFORM` |
| KIL-092 | Risk score is read-only — never set by authors; computed by Risk Engine |

---

## Cross-References

- Reasoning model (blocks/impacts) → `06-REASONING-MODEL`
- Tradeoff model → `16-KNOWLEDGE-TRADEOFF-MODEL`
- Security certification → Phase 3.0D.0 `18-SECURITY-CERTIFICATION`
- Impact analysis → Phase 3.0D.0.5 `17-KNOWLEDGE-ACCEPTANCE-TEST` (KAT-010)
