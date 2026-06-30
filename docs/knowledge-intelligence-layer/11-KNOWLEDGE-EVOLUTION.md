# KNW-KIL-DOC-011 — Knowledge Evolution

**Phase:** 3.0D.0.6
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Every Knowledge Object must record its evolutionary history — where it came from,
what changed and why, how consumers should migrate, and what it will become next.

Evolution is not just a change log. It is the intelligence that allows an AI
agent to understand why an object is the way it is and predict where it is heading.

This document defines `intelligence.evolution`.

---

## Schema

```yaml
intelligence:
  evolution:
    schema_version: "1.0"

    origin:                            # where this object came from
      phase: string                    # e.g. "3.0C"
      created_at: string               # ISO 8601
      created_by: string
      genesis: string                  # one sentence: why was this object created?
      inspired_by: [string]            # knowledge_ids that inspired this object
      split_from: string | null        # if split from a larger object
      merged_from: [string]            # if merged from multiple objects

    changes:                           # complete change history
      - version: string                # semver
        date: string                   # ISO 8601
        change_type: MAJOR | MINOR | PATCH | STRUCTURAL | SEMANTIC
        description: string
        reason: string                 # why was this change made?
        decision_id: string | null     # ADR/RFC reference
        breaking: boolean
        migration: string | null       # how to migrate if breaking

    current_state:                     # current evolutionary position
      maturity: EXPERIMENTAL | BETA | STABLE | MATURE | LEGACY
      stability_since: string          # version since which this is stable
      expected_next_change: string     # e.g. "v3.0.0 - async interface refactor"
      change_frequency: HIGH | MEDIUM | LOW | FROZEN
      next_version_target: string | null

    migration:                         # guidance for migrations from predecessors
      from_predecessors:
        - predecessor_id: string
          migration_type: DROP_IN | ADAPTER | REWRITE | MANUAL
          effort: TRIVIAL | SMALL | MEDIUM | LARGE | VERY_LARGE
          guide: string
          automated: boolean

    replacement:                       # if this object is being superseded
      successor_id: string | null
      replacement_date: string | null  # target date for full replacement
      reason: string | null
      migration_path: string | null

    deprecation:                       # if this object is being deprecated
      is_deprecated: boolean
      deprecated_at: string | null
      sunset_date: string | null       # when it becomes ARCHIVED
      deprecated_because: string | null
      replaced_by: string | null

    evolution_score:                   # composite metric
      velocity: float                  # changes/year (higher = more active)
      stability: float                 # 1.0 - velocity/10 (capped at 1.0)
      maturity_score: float            # 0.0–1.0 based on maturity level
```

---

## Change Types

| Type | Meaning | Breaking? |
|------|---------|-----------|
| PATCH | Bug fix, documentation | No |
| MINOR | Backward-compatible addition | No |
| MAJOR | Breaking interface or behavior change | Yes |
| STRUCTURAL | Object type or namespace change | Yes |
| SEMANTIC | Meaning change without interface change | Sometimes |

---

## Maturity Levels

| Level | Meaning | Change Frequency |
|-------|---------|-----------------|
| EXPERIMENTAL | API unstable, may be removed | HIGH |
| BETA | Stabilizing, some breaking changes possible | MEDIUM |
| STABLE | Stable API, minor/patch only | LOW |
| MATURE | Minimal changes, long-term stable | LOW |
| LEGACY | No new features, maintenance mode | LOW |

---

## Canonical Example

```yaml
# For: KNW-PLT-MOD-001 (Quota Manager)
intelligence:
  evolution:
    schema_version: "1.0"

    origin:
      phase: "3.0C"
      created_at: "2025-01-15T00:00:00Z"
      created_by: "platform-team"
      genesis: >
        Created to solve the multi-tenant resource starvation problem discovered
        in the Platform v1 performance incident where tenant-A consumed 100% CPU.
      inspired_by:
        - KNW-META-PAT-003
      split_from: null
      merged_from: []

    changes:
      - version: "1.0.0"
        date: "2025-01-15"
        change_type: MAJOR
        description: "Initial CANONICAL promotion"
        reason: "First stable implementation of quota enforcement"
        decision_id: "ADR-001"
        breaking: false
        migration: null
      - version: "1.5.0"
        date: "2025-04-20"
        change_type: MINOR
        description: "Added quota-reporting capability (remaining quota in response)"
        reason: "Callers needed remaining quota to implement backoff strategies"
        decision_id: null
        breaking: false
        migration: null
      - version: "2.0.0"
        date: "2025-09-01"
        change_type: MAJOR
        description: "Changed QuotaCheck input from positional to named parameters"
        reason: "Positional interface caused caller bugs when resource types were added"
        decision_id: "ADR-019"
        breaking: true
        migration: "Replace QuotaCheck(tid, rt, amt) with QuotaCheck(tenant_id=tid, resource_type=rt, requested_amount=amt)"
      - version: "2.1.0"
        date: "2026-02-10"
        change_type: MINOR
        description: "Added STORAGE as fourth resource dimension"
        reason: "Platform storage provisioning required quota control"
        decision_id: null
        breaking: false
        migration: null

    current_state:
      maturity: STABLE
      stability_since: "2.0.0"
      expected_next_change: "v3.0.0 — async interface for high-throughput scenarios"
      change_frequency: LOW
      next_version_target: "3.0.0"

    migration:
      from_predecessors:
        - predecessor_id: KNW-PLT-MOD-LEGACY-001
          migration_type: ADAPTER
          effort: SMALL
          guide: "Wrap legacy QuotaCheck calls with QuotaManager adapter (see WF-006)"
          automated: true

    replacement:
      successor_id: null
      replacement_date: null
      reason: null
      migration_path: null

    deprecation:
      is_deprecated: false
      deprecated_at: null
      sunset_date: null
      deprecated_because: null
      replaced_by: null

    evolution_score:
      velocity: 1.2                   # ~1.2 changes/year
      stability: 0.88
      maturity_score: 0.80
```

---

## Rules

| Rule | Statement |
|------|-----------|
| KIL-065 | Every `change_type: MAJOR` entry MUST have a `migration` field (not null) |
| KIL-066 | Every `change_type: BREAKING` entry MUST reference a `decision_id` |
| KIL-067 | `deprecation.sunset_date` must be ≥ 90 days after `deprecation.deprecated_at` |
| KIL-068 | Objects with `is_deprecated: true` must have `replaced_by` set |
| KIL-069 | `evolution_score.stability` must be recomputed when `velocity` changes |
| KIL-070 | `origin.genesis` must explain the problem that created this object, not its implementation |

---

## Cross-References

- DNA (immutable identity) → `08-KNOWLEDGE-DNA`
- Diff (semantic comparison) → `12-KNOWLEDGE-DIFF`
- Decision model → `17-KNOWLEDGE-DECISION-MODEL`
- Alternative model → `18-KNOWLEDGE-ALTERNATIVE-MODEL`
