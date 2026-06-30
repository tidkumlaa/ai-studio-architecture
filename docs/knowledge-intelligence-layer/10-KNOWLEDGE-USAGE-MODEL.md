# KNW-KIL-DOC-010 — Knowledge Usage Model

**Phase:** 3.0D.0.6
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Memory block (doc 09) captures raw access counts. The Usage Model goes
deeper: it describes *how* an object is used — in which contexts, by which
consumer types, for which purposes, and with which co-occurring objects.

This enables the Knowledge Runtime to proactively pre-fetch related objects
and recommend access patterns to AI agents.

This document defines `intelligence.usage`.

---

## Schema

```yaml
intelligence:
  usage:
    schema_version: "1.0"

    access_patterns:                   # how this object is typically accessed
      - pattern_id: string             # e.g. PAT-001
        name: string
        description: string
        frequency: ALWAYS | OFTEN | SOMETIMES | RARELY
        consumer_type: AI_AGENT | DEVELOPER | TEST | CERTIFICATION | SEARCH
        context: string                # e.g. "answering architecture questions"
        co_accessed_objects: [string]  # objects typically fetched in same session

    primary_use_cases:                 # top use cases this object serves
      - use_case_id: string            # e.g. UC-001
        title: string
        description: string
        actor: string                  # who performs this use case
        trigger: string                # what initiates it
        outcome: string                # what is achieved
        required_objects: [string]     # objects needed alongside this one

    query_patterns:                    # how this object surfaces in search
      - query_type: KEYWORD | SEMANTIC | GRAPH | REASONING
        sample_queries: [string]
        typical_rank: integer          # typical position in search results

    anti_patterns:                     # usage patterns that indicate misunderstanding
      - id: string
        name: string
        description: string
        why_wrong: string
        correct_usage: string

    context_requirements:             # what context an agent needs before using this
      - context_item: string
        why_needed: string
        provided_by: string            # knowledge_id or description

    co_occurrence_matrix:              # objects frequently accessed together (top 10)
      - knowledge_id: string
        co_access_rate: float          # 0.0–1.0
        typical_sequence: BEFORE | AFTER | CONCURRENT
        reason: string
```

---

## Canonical Example

```yaml
# For: KNW-PLT-MOD-001 (Quota Manager)
intelligence:
  usage:
    schema_version: "1.0"

    access_patterns:
      - pattern_id: PAT-001
        name: "Quota architecture explanation"
        description: "AI agent explaining how quota enforcement works in the platform"
        frequency: ALWAYS
        consumer_type: AI_AGENT
        context: "answering 'how does quota enforcement work?' questions"
        co_accessed_objects:
          - KNW-PLT-REQ-001
          - KNW-RT-RT-001
          - KNW-META-PAT-003
      - pattern_id: PAT-002
        name: "Impact analysis for Registry changes"
        description: "Checking what breaks if Registry changes interface"
        frequency: OFTEN
        consumer_type: DEVELOPER
        context: "before making changes to KNW-RT-RT-001"
        co_accessed_objects:
          - KNW-RT-RT-001
          - KNW-PLT-SVC-002
      - pattern_id: PAT-003
        name: "Certification search"
        description: "kos-cert fetching quota domain objects for certification"
        frequency: SOMETIMES
        consumer_type: CERTIFICATION
        context: "running kos-cert verify quota-domain"
        co_accessed_objects:
          - KNW-TEST-TST-001
          - KNW-TEST-BENCH-001

    primary_use_cases:
      - use_case_id: UC-001
        title: "Answer quota enforcement question"
        description: "AI agent answers: what enforces resource quotas?"
        actor: "AI reasoning agent"
        trigger: "User asks about quota enforcement"
        outcome: "Correct, evidence-backed explanation of quota enforcement"
        required_objects:
          - KNW-PLT-MOD-001
          - KNW-PLT-REQ-001
          - KNW-RT-RT-001
      - use_case_id: UC-002
        title: "Plan Registry migration"
        description: "Developer assessing impact of Registry interface change"
        actor: "platform developer"
        trigger: "Proposed change to KNW-RT-RT-001 interface"
        outcome: "List of all callers affected by the change"
        required_objects:
          - KNW-PLT-MOD-001
          - KNW-RT-RT-001
          - KNW-PLT-SVC-002

    query_patterns:
      - query_type: SEMANTIC
        sample_queries:
          - "what enforces resource quotas?"
          - "which module handles tenant limits?"
          - "guard pattern in platform namespace"
        typical_rank: 1
      - query_type: KEYWORD
        sample_queries:
          - "quota manager"
          - "KNW-PLT-MOD-001"
          - "plt.module.quota-manager"
        typical_rank: 1
      - query_type: REASONING
        sample_queries:
          - "why does tenant A get rate limited?"
          - "what prevents overconsumption of CPU?"
        typical_rank: 1

    anti_patterns:
      - id: UAP-001
        name: "Using Quota Manager as Rate Limiter"
        description: "Asking Quota Manager to enforce per-second request rates"
        why_wrong: "Quota Manager tracks cumulative usage over periods; it is not a per-second rate limiter"
        correct_usage: "Use KNW-ALG-ALG-009 Rate Limiter for per-second rate control"
      - id: UAP-002
        name: "Direct quota state mutation"
        description: "Attempting to update quota limits via Quota Manager"
        why_wrong: "Quota Manager is read-only to quota limits by design (FDC-002)"
        correct_usage: "Use the Admin API to update quota limits"

    context_requirements:
      - context_item: "KNW-PLT-REQ-001 — the requirement this module implements"
        why_needed: "Without the requirement, AI agents may not understand the enforcement scope"
        provided_by: "KNW-PLT-REQ-001"
      - context_item: "KNW-RT-RT-001 — registry contract"
        why_needed: "Quota state mechanics require understanding Registry"
        provided_by: "KNW-RT-RT-001"
      - context_item: "Platform namespace overview"
        why_needed: "Placing Quota Manager in context of broader governance"
        provided_by: "Phase 3.0D.0.5 05-CANONICAL-PACKAGES"

    co_occurrence_matrix:
      - knowledge_id: KNW-PLT-REQ-001
        co_access_rate: 0.94
        typical_sequence: CONCURRENT
        reason: "Requirement always fetched alongside the module that implements it"
      - knowledge_id: KNW-RT-RT-001
        co_access_rate: 0.89
        typical_sequence: AFTER
        reason: "Registry fetched to understand quota state storage"
      - knowledge_id: KNW-META-PAT-003
        co_access_rate: 0.45
        typical_sequence: AFTER
        reason: "Guard pattern fetched when explaining design rationale"
```

---

## Pre-Fetch Recommendation

Based on the co-occurrence matrix, the Runtime SHOULD pre-fetch:

```
When accessing KNW-PLT-MOD-001:
  Pre-fetch if budget allows:
    - KNW-PLT-REQ-001  (co_access_rate: 0.94) → ALWAYS
    - KNW-RT-RT-001    (co_access_rate: 0.89) → ALWAYS
    - KNW-META-PAT-003 (co_access_rate: 0.45) → OPTIONAL
```

---

## Rules

| Rule | Statement |
|------|-----------|
| KIL-061 | `co_occurrence_matrix` must be derived from `memory.agents[*].common_questions` data |
| KIL-062 | Every `anti_pattern` in `usage` must have a distinct `correct_usage` |
| KIL-063 | Objects with `query_patterns.typical_rank > 10` for their primary semantic query need search optimization |
| KIL-064 | `context_requirements` must list any object appearing in `related_objects` with `fetch_priority: ALWAYS` |

---

## Cross-References

- Memory (access counts) → `09-KNOWLEDGE-MEMORY`
- AI context (related objects) → `04-AI-CONTEXT-LAYER`
- Search optimization → `28-KNOWLEDGE-SEARCH-OPTIMIZATION`
- Optimization hints → `24-KNOWLEDGE-OPTIMIZATION`
