# KNW-KIL-DOC-004 — AI Context Layer

**Phase:** 3.0D.0.6
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Every Knowledge Object must contain pre-computed AI context that can be injected
directly into an AI agent's prompt without requiring the agent to derive context
from raw YAML.

This document defines `intelligence.ai_context` — the AI-facing surface of every
Knowledge Object.

---

## Schema

```yaml
intelligence:
  ai_context:
    token_budget:
      short: 50                        # tokens for short_summary
      medium: 200                      # tokens for medium_summary
      full: 2000                       # tokens for full_summary
      hints: 100                       # tokens for ai_hints

    short_summary: string              # ≤ 50 tokens: one-sentence essence
    medium_summary: string             # ≤ 200 tokens: what, why, key behavior
    full_summary: string               # ≤ 2000 tokens: complete AI-ready description

    ai_hints:                          # structured hints for AI reasoning
      - hint_type: BEHAVIOR | CONSTRAINT | RELATIONSHIP | PATTERN | GOTCHA | CONTEXT
        content: string
        priority: HIGH | MEDIUM | LOW

    common_questions:                  # questions an AI is likely to be asked about this object
      - question: string
        answer: string
        confidence: CERTAIN | LIKELY | UNCERTAIN
        source: string                 # evidence ID or document reference

    prompt_examples:                   # ready-to-use prompt fragments
      - use_case: string               # e.g. "explain to a developer"
        prompt_fragment: string        # the prompt text to inject
        expected_response_type: string # e.g. "technical explanation"

    related_objects:                   # objects an AI should fetch alongside this one
      - knowledge_id: string
        relevance: string              # why this object is related
        fetch_priority: ALWAYS | LIKELY | OPTIONAL

    typical_mistakes:                  # mistakes AI agents make about this object
      - mistake_id: string             # e.g. TM-001
        description: string
        correction: string
        category: HALLUCINATION | CONFUSION | OMISSION | OVERREACH
```

---

## Token Budget by Context Type

| Context Type | Token Budget | Usage |
|-------------|-------------|-------|
| Inline hint | 50 | Summary line in a list of objects |
| Quick context | 200 | Single object in a small prompt |
| Standard context | 500 | Object with immediate neighbors |
| Full context | 2000 | Deep reasoning about one object |
| Registry context | 200/object | Many objects in a search result |
| Graph context | 100/node | Graph traversal result |

---

## AI Hint Types

| Type | Purpose |
|------|---------|
| BEHAVIOR | Describes what the object does at runtime |
| CONSTRAINT | Describes what the object cannot do |
| RELATIONSHIP | Highlights critical dependencies |
| PATTERN | Describes a design pattern this object follows |
| GOTCHA | Warns about non-obvious behavior |
| CONTEXT | Provides historical or architectural context |

---

## Typical Mistake Categories

| Category | Description |
|----------|-------------|
| HALLUCINATION | AI invents a behavior this object does not have |
| CONFUSION | AI confuses this object with a similar one |
| OMISSION | AI misses a critical constraint or dependency |
| OVERREACH | AI assumes this object does more than it does |

---

## Canonical Example

```yaml
# For: KNW-PLT-MOD-001 (Quota Manager)
intelligence:
  ai_context:
    token_budget:
      short: 50
      medium: 200
      full: 2000
      hints: 100

    short_summary: >
      Quota Manager enforces per-tenant CPU/Memory/Request/Storage limits
      with < 5ms P99 latency.

    medium_summary: >
      KNW-PLT-MOD-001 Quota Manager is the platform's tenant resource guard.
      It receives a (tenant_id, resource_type, requested_amount) tuple and
      returns ALLOW/DENY with remaining quota. Quota state is stored in the
      Registry (KNW-RT-RT-001). It does NOT manage quotas — that is Admin API.
      It ONLY enforces them. Violation emits QuotaExceededEvent.

    full_summary: >
      KNW-PLT-MOD-001 Quota Manager is a GUARD-category MODULE in the
      kos.platform (plt) namespace. It enforces per-tenant resource quotas
      across four resource dimensions: CPU, MEMORY, REQUESTS, STORAGE.

      BEHAVIOR: Given (tenant_id, resource_type, requested_amount), returns
      QuotaDecision(ALLOW|DENY) and remaining_quota float. Check must complete
      in < 5ms P99. Uses Registry for quota state. Config loaded at startup,
      cached in process.

      CONSTRAINTS: Read-only to quota limits; Admin API owns writes.
      Quota state MUST be persisted before returning ALLOW.

      DEPENDENCIES: KNW-PLT-REQ-001 (requirement it satisfies),
      KNW-RT-RT-001 (registry for state).

      EVENTS: Emits QuotaExceededEvent to KNW-PLT-MON-001 on denial.

      QUALITY SCORE: Must be ≥ 0.80 to remain CANONICAL.

      NOT IN SCOPE: Quota configuration, quota resetting, per-endpoint
      rate limiting (see KNW-ALG-ALG-009 Rate Limiter).

    ai_hints:
      - hint_type: GOTCHA
        content: >
          Quota Manager enforces quotas; it does NOT configure them.
          The Admin API owns quota limits. Do not conflate the two.
        priority: HIGH
      - hint_type: CONSTRAINT
        content: "P99 < 5ms — synchronous blocking calls are forbidden in the check path"
        priority: HIGH
      - hint_type: RELATIONSHIP
        content: "KNW-RT-RT-001 is a HARD dependency; Quota Manager cannot operate without Registry"
        priority: HIGH
      - hint_type: PATTERN
        content: "Follows the Guard pattern (KNW-META-PAT-003)"
        priority: MEDIUM
      - hint_type: BEHAVIOR
        content: "Returns remaining quota even on DENY to help callers plan retry timing"
        priority: MEDIUM

    common_questions:
      - question: "What does Quota Manager do?"
        answer: >
          It enforces per-tenant resource quotas. Given a tenant and resource
          request, it returns ALLOW or DENY and the remaining quota.
        confidence: CERTAIN
        source: "KNW-PLT-REQ-001"
      - question: "How does Quota Manager store quota state?"
        answer: >
          It reads and writes quota state via the Registry (KNW-RT-RT-001).
          It does not own its own storage.
        confidence: CERTAIN
        source: "KNW-PLT-MOD-001 self_describing.dependencies"
      - question: "Can Quota Manager reset quotas?"
        answer: >
          No. Quota configuration and resetting belong to the Admin API.
          Quota Manager is read-only with respect to quota limits.
        confidence: CERTAIN
        source: "KNW-PLT-MOD-001 self_describing.constraints.CON-003"

    prompt_examples:
      - use_case: "Explain to a platform developer"
        prompt_fragment: >
          KNW-PLT-MOD-001 is the Quota Manager module. It enforces per-tenant
          CPU/Memory/Request/Storage quotas. Input: (tenant_id, resource_type,
          requested_amount). Output: ALLOW/DENY + remaining_quota. P99 < 5ms.
          Uses Registry for state. Does NOT configure quotas.
        expected_response_type: "technical implementation explanation"
      - use_case: "Explain to an architect"
        prompt_fragment: >
          Quota Manager (KNW-PLT-MOD-001) is a Guard-pattern MODULE in the
          plt namespace implementing KNW-PLT-REQ-001. HARD dependency on Registry.
          Emits QuotaExceededEvent. Config cached at startup.
        expected_response_type: "architectural placement explanation"

    related_objects:
      - knowledge_id: KNW-PLT-REQ-001
        relevance: "Requirement this module implements"
        fetch_priority: ALWAYS
      - knowledge_id: KNW-RT-RT-001
        relevance: "HARD dependency for quota state storage"
        fetch_priority: ALWAYS
      - knowledge_id: KNW-PLT-MON-001
        relevance: "Receives QuotaExceededEvent"
        fetch_priority: LIKELY
      - knowledge_id: KNW-META-PAT-003
        relevance: "Design pattern this module follows"
        fetch_priority: OPTIONAL

    typical_mistakes:
      - mistake_id: TM-001
        description: "AI says Quota Manager can update quota limits"
        correction: "Quota limits are owned by Admin API. Quota Manager is read-only."
        category: HALLUCINATION
      - mistake_id: TM-002
        description: "AI confuses Quota Manager with Rate Limiter (KNW-ALG-ALG-009)"
        correction: "Rate Limiter controls request rate per window. Quota Manager controls cumulative resource usage per tenant over time."
        category: CONFUSION
      - mistake_id: TM-003
        description: "AI omits the P99 < 5ms constraint when describing behavior"
        correction: "Always mention the latency constraint — it drives all implementation decisions."
        category: OMISSION
```

---

## Rules

| Rule | Statement |
|------|-----------|
| KIL-022 | Every CANONICAL object MUST have `ai_context.short_summary` ≤ 50 tokens |
| KIL-023 | Every CANONICAL object MUST have `ai_context.medium_summary` ≤ 200 tokens |
| KIL-024 | `short_summary` must mention the object's primary capability and key constraint |
| KIL-025 | Every `common_question` MUST have `confidence` set; `UNCERTAIN` requires justification |
| KIL-026 | `typical_mistakes` MUST be populated for any object that has caused AI hallucinations in testing |
| KIL-027 | `related_objects` with `fetch_priority: ALWAYS` must always be included in PromptPack context |
| KIL-028 | `full_summary` must cover: BEHAVIOR, CONSTRAINTS, DEPENDENCIES, EVENTS, SCOPE, NOT-IN-SCOPE |

---

## Cross-References

- Compression summaries → `05-KNOWLEDGE-COMPRESSION`
- AI readiness score → `27-KNOWLEDGE-AI-READINESS`
- PromptPack context → Phase 3.0D.0.5 `17-KNOWLEDGE-ACCEPTANCE-TEST` (KAT-007)
- Hallucination categories → Phase 3.0D.0 `23-HALLUCINATION-CERTIFICATION`
