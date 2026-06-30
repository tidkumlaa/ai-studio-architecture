# KNW-KIL-DOC-009 — Knowledge Memory

**Phase:** 3.0D.0.6
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Every Knowledge Object accumulates memory — a record of how it has been used,
by whom, in which contexts, and with what outcomes. This memory enables the
knowledge base to surface frequently-used objects, warn about under-used ones,
and guide learning agents toward high-value knowledge.

Memory is the only mutable part of the intelligence block. It is updated by
the Knowledge Runtime, not by authors.

This document defines `intelligence.memory`.

---

## Schema

```yaml
intelligence:
  memory:
    schema_version: "1.0"
    last_updated: string               # ISO 8601 — when memory was last written

    usage_frequency:
      total_accesses: integer          # all-time access count
      accesses_last_30d: integer
      accesses_last_7d: integer
      accesses_last_24h: integer
      peak_accesses_per_hour: integer
      first_accessed: string           # ISO 8601
      last_accessed: string            # ISO 8601

    access_breakdown:                  # by consumer type
      ai_agent: integer
      human_developer: integer
      automated_test: integer
      certification_run: integer
      search_query: integer
      other: integer

    projects:                          # projects that have used this object
      - project_id: string
        project_name: string
        first_used: string
        last_used: string
        access_count: integer
        outcome: SUCCESS | FAILURE | MIXED | UNKNOWN

    agents:                            # AI agents that have used this object
      - agent_id: string
        agent_type: string             # e.g. "reasoning-agent", "search-agent"
        access_count: integer
        last_accessed: string
        common_questions: [string]     # what questions this agent asked about it

    success_rate:
      total_uses: integer
      successful_uses: integer
      rate: float                      # 0.0–1.0
      success_definition: string       # what counts as success for this object

    failures:
      total_failures: integer
      failure_categories:
        - category: string             # e.g. "stale data", "wrong context", "hallucination"
          count: integer
          last_seen: string
          example: string

    popularity:
      rank_in_namespace: integer       # rank by access count within namespace
      rank_in_package: integer
      rank_global: integer
      trend: RISING | STABLE | DECLINING | NEW
      trend_period_days: integer       # period over which trend is measured
```

---

## Canonical Example

```yaml
# For: KNW-PLT-MOD-001 (Quota Manager)
intelligence:
  memory:
    schema_version: "1.0"
    last_updated: "2026-06-29T23:59:00Z"

    usage_frequency:
      total_accesses: 14827
      accesses_last_30d: 1203
      accesses_last_7d: 287
      accesses_last_24h: 43
      peak_accesses_per_hour: 89
      first_accessed: "2025-01-20T08:00:00Z"
      last_accessed: "2026-06-29T23:51:22Z"

    access_breakdown:
      ai_agent: 8341
      human_developer: 3201
      automated_test: 2100
      certification_run: 812
      search_query: 312
      other: 61

    projects:
      - project_id: "proj-platform-v2"
        project_name: "Platform v2 Migration"
        first_used: "2025-03-01"
        last_used: "2025-08-31"
        access_count: 4201
        outcome: SUCCESS
      - project_id: "proj-multitenant-audit"
        project_name: "Multi-tenant Compliance Audit"
        first_used: "2026-01-15"
        last_used: "2026-06-20"
        access_count: 2100
        outcome: SUCCESS

    agents:
      - agent_id: "reasoning-agent-prod-001"
        agent_type: "reasoning-agent"
        access_count: 5831
        last_accessed: "2026-06-29T23:51:22Z"
        common_questions:
          - "What does Quota Manager enforce?"
          - "What is the latency constraint?"
          - "What happens when quota is exceeded?"
      - agent_id: "search-agent-kos-001"
        agent_type: "search-agent"
        access_count: 2510
        last_accessed: "2026-06-29T22:30:00Z"
        common_questions:
          - "Which object enforces quotas?"
          - "Guard pattern in plt namespace"

    success_rate:
      total_uses: 14827
      successful_uses: 14392
      rate: 0.971
      success_definition: "AI correctly answered quota-related question with no hallucination"

    failures:
      total_failures: 435
      failure_categories:
        - category: "confusion with rate limiter"
          count: 201
          last_seen: "2026-06-10"
          example: "AI incorrectly stated that Quota Manager handles per-second rate limits"
        - category: "omitted latency constraint"
          count: 134
          last_seen: "2026-06-18"
          example: "AI described quota check without mentioning P99 < 5ms"
        - category: "stale evidence"
          count: 100
          last_seen: "2026-05-01"
          example: "AI referenced v1.0.0 behavior after v2.0.0 change"

    popularity:
      rank_in_namespace: 3
      rank_in_package: 3
      rank_global: 12
      trend: STABLE
      trend_period_days: 90
```

---

## Memory Update Protocol

Memory is written by the Knowledge Runtime, not by object authors:

```
Write conditions:
  - Every object access updates usage_frequency
  - Every search result appearance updates search_query count
  - Every AI agent fetch updates agents[] entry
  - Success/failure is tagged by the calling context

Write frequency:
  - Batch writes every 5 minutes
  - Immediate write for first access of new object

Memory is NOT part of the object checksum:
  - Memory changes never trigger quality re-scoring
  - Memory changes never invalidate DNA or Genome
```

---

## Rules

| Rule | Statement |
|------|-----------|
| KIL-055 | `intelligence.memory` is written ONLY by the Knowledge Runtime — never by authors |
| KIL-056 | Memory must be excluded from `metadata.checksum` computation |
| KIL-057 | `success_rate.success_definition` must be defined per object type, not per object |
| KIL-058 | Objects with `total_accesses == 0` after 90 days are flagged as UNUSED |
| KIL-059 | `failures.failure_categories` must feed back into `ai_context.typical_mistakes` |
| KIL-060 | Popularity ranks are recomputed nightly — not on every access |

---

## Cross-References

- Genome (full profile) → `07-KNOWLEDGE-GENOME`
- Usage model → `10-KNOWLEDGE-USAGE-MODEL`
- Learning layer → `23-KNOWLEDGE-LEARNING-LAYER`
- AI context mistakes → `04-AI-CONTEXT-LAYER`
- Metrics → `26-KNOWLEDGE-METRICS`
