# KNW-KIL-DOC-005 — Knowledge Compression

**Phase:** 3.0D.0.6
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Every Knowledge Object must be expressible at multiple levels of detail.
A 2,000-field object cannot be embedded whole into every AI context window.

The compression layer defines 5 standard views, each with a fixed token budget
and required content. Compression is lossless at the structural level — every
compressed view is derivable from the full object.

This document defines `intelligence.compression`.

---

## Compression Levels

| Level | Name | Max Tokens | Format | Consumer |
|-------|------|-----------|--------|---------|
| L1 | Nano | 15 | String | List items, graph node labels |
| L2 | Micro | 50 | String | Inline references, search results |
| L3 | Mini | 200 | Structured string | Quick context, brief answers |
| L4 | Standard | 500 | YAML/JSON | Standard reasoning context |
| L5 | Full | 2000 | YAML/JSON | Deep analysis, certification |

---

## Schema

```yaml
intelligence:
  compression:
    nano: string                       # L1: ≤ 15 tokens — ID + name only
    micro: string                      # L2: ≤ 50 tokens — one sentence
    mini: string                       # L3: ≤ 200 tokens — what/why/key constraint
    standard:                          # L4: ≤ 500 tokens structured
      id: string
      name: string
      type: string
      namespace: string
      purpose: string
      key_capabilities: [string]
      key_dependencies: [string]
      key_constraints: [string]
      quality_score: float
      state: string
    machine:                           # L5: machine-readable minimal representation
      id: string
      type: string
      version: string
      capabilities: [string]
      provides: [string]
      consumes: [string]
      depends_on: [string]
      implements: [string]
      quality_score: float
      confidence_score: float
      ai_readiness_score: float

    compression_rules:
      - level: string                  # L1 | L2 | L3 | L4 | L5
        include_fields: [string]       # which fields must appear
        exclude_fields: [string]       # fields never shown at this level
        truncation_strategy: ELLIPSIS | SUMMARIZE | OMIT
        token_enforcement: HARD | SOFT  # HARD = error if exceeded; SOFT = warning
```

---

## Canonical Example

```yaml
# For: KNW-PLT-MOD-001 (Quota Manager)
intelligence:
  compression:
    nano: "KNW-PLT-MOD-001 Quota Manager"

    micro: >
      KNW-PLT-MOD-001 Quota Manager: enforces per-tenant resource quotas
      (CPU/Memory/Requests/Storage), P99 < 5ms, returns ALLOW/DENY.

    mini: >
      KNW-PLT-MOD-001 Quota Manager (plt.module.quota-manager v2.1.0)
      Purpose: Enforce per-tenant resource quotas.
      Capabilities: quota-enforcement (GUARD), quota-reporting (MONITOR).
      Critical constraint: P99 < 5ms check latency.
      Depends on: KNW-RT-RT-001 (HARD), KNW-PLT-REQ-001 (implements).
      Provides: QuotaDecision (ALLOW/DENY), QuotaExceededEvent.
      State: CANONICAL. Quality: 0.87.

    standard:
      id: KNW-PLT-MOD-001
      name: "Quota Manager"
      type: MODULE
      namespace: plt
      purpose: "Enforce per-tenant CPU/Memory/Request/Storage quotas"
      key_capabilities:
        - "quota-enforcement (GUARD, STABLE)"
        - "quota-reporting (MONITOR, STABLE)"
      key_dependencies:
        - "KNW-RT-RT-001 Registry (HARD)"
        - "KNW-PLT-CFG-001 Config (HARD)"
      key_constraints:
        - "P99 < 5ms per quota check"
        - "Quota state persisted before ALLOW"
        - "Read-only to quota limits"
      quality_score: 0.87
      state: CANONICAL

    machine:
      id: KNW-PLT-MOD-001
      type: MODULE
      version: "2.1.0"
      capabilities: ["quota-enforcement", "quota-reporting"]
      provides: ["QuotaDecision", "QuotaExceededEvent"]
      consumes: ["TenantQuotaConfig", "RegistryState"]
      depends_on: ["KNW-RT-RT-001", "KNW-PLT-CFG-001"]
      implements: ["KNW-PLT-REQ-001"]
      quality_score: 0.87
      confidence_score: 0.91
      ai_readiness_score: 0.84

    compression_rules:
      - level: L1
        include_fields: ["knowledge_id", "name"]
        exclude_fields: ["*"]
        truncation_strategy: OMIT
        token_enforcement: HARD
      - level: L2
        include_fields: ["knowledge_id", "name", "object_type", "purpose"]
        exclude_fields: ["evidence", "history", "evolution"]
        truncation_strategy: ELLIPSIS
        token_enforcement: HARD
      - level: L3
        include_fields: ["knowledge_id", "name", "object_type", "purpose",
                         "capabilities", "dependencies", "constraints", "quality_score"]
        exclude_fields: ["evidence.items", "history", "alternatives"]
        truncation_strategy: SUMMARIZE
        token_enforcement: SOFT
      - level: L4
        include_fields: ["*"]
        exclude_fields: ["evidence.items[].raw_data", "history.full_log"]
        truncation_strategy: SUMMARIZE
        token_enforcement: SOFT
      - level: L5
        include_fields: ["*"]
        exclude_fields: []
        truncation_strategy: OMIT
        token_enforcement: SOFT
```

---

## Compression Selection Logic

```
PromptPack selects compression level based on:
  IF   remaining_token_budget >= 2000  → L5
  ELIF remaining_token_budget >= 500   → L4
  ELIF remaining_token_budget >= 200   → L3
  ELIF remaining_token_budget >= 50    → L2
  ELSE                                 → L1

Exception: objects in related_objects with fetch_priority=ALWAYS
  use L3 minimum regardless of budget.
```

---

## Rules

| Rule | Statement |
|------|-----------|
| KIL-029 | Every CANONICAL object MUST have all 5 compression levels populated |
| KIL-030 | `nano` must be derivable from `knowledge_id` + `name` alone |
| KIL-031 | `micro` must be derivable from `ai_context.short_summary` |
| KIL-032 | `machine` must be valid JSON/YAML that parses without error |
| KIL-033 | `standard` must include `quality_score` and `state` |
| KIL-034 | No `nano` string may exceed 15 tokens — this is HARD enforced |
| KIL-035 | Compression is READ-ONLY; it must never be used to update the object |

---

## Cross-References

- AI context summaries → `04-AI-CONTEXT-LAYER`
- AI readiness score → `27-KNOWLEDGE-AI-READINESS`
- PromptPack selection → Phase 3.0D.0.5 `17-KNOWLEDGE-ACCEPTANCE-TEST` (KAT-007)
- Search index → `29-KNOWLEDGE-CANONICAL-INDEX`
