# KNW-KIL-DOC-003 — Semantic Knowledge

**Phase:** 3.0D.0.6
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Every Knowledge Object must expose its semantic role in the ecosystem — what it
provides, what it consumes, what it extends, what it conflicts with, and what
objects are semantically equivalent.

This enables AI agents and search systems to discover objects not just by name
but by capability and semantic role.

This document defines `intelligence.semantic`.

---

## Schema

```yaml
intelligence:
  semantic:
    capabilities:                      # what this object is capable of doing
      - id: string                     # e.g. CAP-001
        name: string
        description: string
        category: COMPUTE | STORE | ROUTE | VALIDATE | MONITOR | TRANSFORM | COORDINATE | GUARD
        maturity: STABLE | BETA | EXPERIMENTAL

    provides:                          # what this object makes available to others
      - name: string
        type: INTERFACE | DATA | BEHAVIOR | CONTRACT | EVENT | CONFIGURATION
        consumers: [string]            # knowledge_id list of known consumers
        description: string

    consumes:                          # what this object requires from others
      - name: string
        type: INTERFACE | DATA | BEHAVIOR | CONTRACT | EVENT | CONFIGURATION
        provider_id: string            # knowledge_id of the provider
        required: boolean
        description: string

    extends:                           # objects this object specializes or extends
      - knowledge_id: string
        extension_type: SPECIALIZES | INHERITS | OVERRIDES | COMPOSES
        description: string

    implements:                        # specifications/contracts this object satisfies
      - knowledge_id: string           # REQUIREMENT | SPECIFICATION | STANDARD | CONTRACT
        completeness: FULL | PARTIAL
        note: string

    replaces:                          # objects this object supersedes
      - knowledge_id: string
        migration_guide: string        # one sentence on how to migrate
        compatibility: BACKWARD | BREAKING | PARTIAL

    conflicts:                         # objects that cannot coexist with this one
      - knowledge_id: string
        reason: string
        resolution: string             # how to resolve if both are needed

    equivalents:                       # semantically equivalent objects (same meaning, different form)
      - knowledge_id: string
        equivalence_type: FULL | FUNCTIONAL | BEHAVIORAL | CONCEPTUAL
        differences: string            # what is different despite semantic equivalence
```

---

## Capability Categories

| Category | Meaning |
|----------|---------|
| COMPUTE | Performs computation or transformation |
| STORE | Persists or caches data |
| ROUTE | Directs or dispatches requests |
| VALIDATE | Enforces rules or constraints |
| MONITOR | Observes and measures |
| TRANSFORM | Converts data from one form to another |
| COORDINATE | Orchestrates multiple components |
| GUARD | Controls access or enforces policies |

---

## Extension Types

| Type | Meaning |
|------|---------|
| SPECIALIZES | Narrows the behavior of the parent for a specific case |
| INHERITS | Reuses parent's schema and behavior without modification |
| OVERRIDES | Replaces specific parts of the parent's behavior |
| COMPOSES | Combines multiple parents into a new object |

---

## Canonical Example

```yaml
# For: KNW-PLT-MOD-001 (Quota Manager)
intelligence:
  semantic:
    capabilities:
      - id: CAP-001
        name: "quota-enforcement"
        description: "Enforce per-tenant CPU/Memory/Request/Storage quotas"
        category: GUARD
        maturity: STABLE
      - id: CAP-002
        name: "quota-reporting"
        description: "Report current quota consumption per tenant"
        category: MONITOR
        maturity: STABLE

    provides:
      - name: "QuotaDecision"
        type: BEHAVIOR
        consumers: [KNW-PLT-SVC-002, KNW-API-API-001]
        description: "ALLOW/DENY decision for resource requests"
      - name: "QuotaExceededEvent"
        type: EVENT
        consumers: [KNW-PLT-MON-001]
        description: "Event emitted when a tenant exceeds quota"

    consumes:
      - name: "TenantQuotaConfig"
        type: CONFIGURATION
        provider_id: KNW-PLT-CFG-001
        required: true
        description: "Per-tenant quota limits configuration"
      - name: "RegistryState"
        type: DATA
        provider_id: KNW-RT-RT-001
        required: true
        description: "Current quota usage state from registry"

    extends:
      - knowledge_id: KNW-META-PAT-003
        extension_type: SPECIALIZES
        description: "Specializes the Rate Limiter pattern for resource quotas"

    implements:
      - knowledge_id: KNW-PLT-REQ-001
        completeness: FULL
        note: "Implements all quota enforcement requirements"
      - knowledge_id: KNW-META-STD-007
        completeness: PARTIAL
        note: "Implements latency standard; storage durability deferred to RT"

    replaces:
      - knowledge_id: KNW-PLT-MOD-LEGACY-001
        migration_guide: "Replace all QuotaCheck calls with QuotaManager.check()"
        compatibility: BACKWARD

    conflicts:
      - knowledge_id: KNW-PLT-MOD-007
        reason: "Both claim ownership of tenant quota state"
        resolution: "Use QuotaManager exclusively; disable bypass mode in MOD-007"

    equivalents:
      - knowledge_id: KNW-PROV-MOD-012
        equivalence_type: FUNCTIONAL
        differences: "Provider-scoped; does not enforce platform-level limits"
```

---

## Semantic Search Index

The `semantic` block drives semantic object discovery:

```
Query: "what enforces quotas?"
  → search capabilities WHERE category=GUARD AND name matches "quota"
  → find KNW-PLT-MOD-001

Query: "what does the Quota Manager provide?"
  → read semantic.provides for KNW-PLT-MOD-001
  → QuotaDecision, QuotaExceededEvent

Query: "what conflicts with KNW-PLT-MOD-001?"
  → read semantic.conflicts
  → KNW-PLT-MOD-007
```

---

## Rules

| Rule | Statement |
|------|-----------|
| KIL-015 | Every CANONICAL object MUST have ≥ 1 `capability` in `semantic` |
| KIL-016 | Every `provides` entry MUST have ≥ 1 known `consumer` or justify zero consumers |
| KIL-017 | Every `consumes` entry with `required: true` MUST have a `provider_id` that exists in registry |
| KIL-018 | Every `implements` entry MUST reference a REQUIREMENT, SPECIFICATION, STANDARD, or CONTRACT |
| KIL-019 | Every `conflicts` entry MUST have a `resolution` — "cannot coexist" is a valid value |
| KIL-020 | `replaces` entries MUST only reference DEPRECATED or ARCHIVED objects |
| KIL-021 | `equivalents` of type FULL must have identical capability sets |

---

## Cross-References

- Self-describing capabilities → `01-SELF-DESCRIBING-KNOWLEDGE`
- Reasoning model (enables/blocks) → `06-REASONING-MODEL`
- Search optimization → `28-KNOWLEDGE-SEARCH-OPTIMIZATION`
- Relationship types → Phase 3.0C.5 `06-CANONICAL-RELATIONSHIPS`
