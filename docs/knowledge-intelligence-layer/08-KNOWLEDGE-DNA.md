# KNW-KIL-DOC-008 — Knowledge DNA

**Phase:** 3.0D.0.6
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Every Knowledge Object has an immutable DNA — a 5-field record that captures its
fundamental, unchanging essence. Unlike the Genome (which evolves as the object
matures), the DNA is set once at object creation and never modified.

The DNA answers: *What is this object at its most fundamental level?*

If the DNA changes, a new object must be created. The old object is deprecated.

This document defines `intelligence.dna`.

---

## DNA Fields

| Field | # | Immutable? | Description |
|-------|---|-----------|-------------|
| identity | 1 | YES | Canonical name + object type at birth |
| purpose | 2 | YES | The single irreducible reason this object exists |
| core_behavior | 3 | YES | The behavior that makes this object what it is |
| primary_interfaces | 4 | YES | The interfaces that define this object's contract |
| fundamental_constraints | 5 | YES | Hard limits that can never be removed |

---

## Schema

```yaml
intelligence:
  dna:
    version: "1.0"
    sealed_at: string                  # ISO 8601 — when DNA was locked
    sealed_by: string                  # who/what sealed it
    dna_hash: string                   # sha256 of all 5 fields

    identity:                          # Field 1: Fundamental identity
      original_name: string            # name at first CANONICAL promotion
      original_type: string            # type at creation — cannot change
      original_namespace: string       # namespace at creation — cannot change
      birth_version: string            # version when first CANONICAL

    purpose:                           # Field 2: Irreducible purpose
      statement: string                # one sentence — the reason this object exists
      derived_from: string             # knowledge_id of requirement or decision
      not_in_scope: [string]           # what this object explicitly does NOT do

    core_behavior:                     # Field 3: Essential behavior
      primary_action: string           # the one thing this object does above all else
      input_contract: string           # minimal input it needs (one sentence)
      output_contract: string          # minimal output it always produces (one sentence)
      invariants: [string]             # conditions that must always hold

    primary_interfaces:                # Field 4: Defining contracts
      - interface_name: string
        direction: IN | OUT | BIDIRECTIONAL
        description: string
        breaking_change_impact: ECOSYSTEM | PACKAGE | LOCAL

    fundamental_constraints:          # Field 5: Non-negotiable limits
      - id: string                     # e.g. FDC-001
        statement: string
        reason: string
        cannot_be_removed_because: string
```

---

## DNA Immutability Rules

Once DNA is sealed (at first CANONICAL promotion), it becomes immutable:

```
DNA Change → New Object Required

Allowed after sealing:
  - Genome updates (version, evidence, relationships)
  - Intelligence block additions
  - Non-core behavior additions
  - Performance improvements that preserve output contract

NOT allowed after sealing (requires new object):
  - Changing identity.original_type
  - Changing identity.original_namespace
  - Changing purpose.statement
  - Changing core_behavior.primary_action
  - Changing core_behavior.output_contract
  - Removing a fundamental_constraint
  - Removing a primary_interface
```

---

## Canonical Example

```yaml
# For: KNW-PLT-MOD-001 (Quota Manager)
intelligence:
  dna:
    version: "1.0"
    sealed_at: "2025-01-15T00:00:00Z"
    sealed_by: "architecture-board"
    dna_hash: "sha256:d1e2f3a4b5c6..."

    identity:
      original_name: "Quota Manager"
      original_type: MODULE
      original_namespace: plt
      birth_version: "1.0.0"

    purpose:
      statement: >
        Enforce per-tenant resource quota limits to prevent any single tenant
        from consuming more than their allocated share of platform resources.
      derived_from: KNW-PLT-REQ-001
      not_in_scope:
        - "Configuring or updating quota limits (Admin API responsibility)"
        - "Per-endpoint rate limiting (KNW-ALG-ALG-009 responsibility)"
        - "Billing and cost attribution (KNW-FIN-SVC-001 responsibility)"
        - "Cross-tenant quota sharing or pooling"

    core_behavior:
      primary_action: >
        Given (tenant_id, resource_type, requested_amount), return
        ALLOW or DENY with remaining quota.
      input_contract: >
        Must receive a valid tenant_id, a recognized resource_type, and a
        non-negative requested_amount.
      output_contract: >
        Must always return a QuotaDecision (ALLOW|DENY) and a non-negative
        remaining_quota float, or raise a structured error — never silently fail.
      invariants:
        - "No ALLOW is returned unless quota state is persisted first"
        - "P99 check latency < 5ms under nominal load"
        - "Quota state is consistent with Registry"
        - "Every DENY emits a QuotaExceededEvent"

    primary_interfaces:
      - interface_name: "QuotaCheck"
        direction: IN
        description: "Receives resource request and returns quota decision"
        breaking_change_impact: ECOSYSTEM
      - interface_name: "QuotaDecision"
        direction: OUT
        description: "Returns ALLOW/DENY and remaining quota to callers"
        breaking_change_impact: ECOSYSTEM
      - interface_name: "QuotaExceededEvent"
        direction: OUT
        description: "Emitted to event bus on every DENY decision"
        breaking_change_impact: PACKAGE

    fundamental_constraints:
      - id: FDC-001
        statement: "P99 check latency must be < 5ms"
        reason: "Quota check is on every request's hot path"
        cannot_be_removed_because: >
          Removing this constraint would make quota enforcement too slow to be
          practical on a high-throughput platform. It is a system-level
          requirement (KNW-META-STD-007).
      - id: FDC-002
        statement: "Quota limits are read-only to this module"
        reason: "Separation of enforcement from configuration prevents privilege escalation"
        cannot_be_removed_because: >
          Allowing Quota Manager to modify limits would conflate enforcement
          with governance, creating a security boundary violation.
      - id: FDC-003
        statement: "Every DENY must emit QuotaExceededEvent — silent denial is forbidden"
        reason: "Observability requirement for audit and debugging"
        cannot_be_removed_because: >
          Silent denial makes quota enforcement unauditable. Platform compliance
          (KNW-META-STD-012) requires every enforcement action to be logged.
```

---

## DNA Comparison

Use DNA to determine if two objects are fundamentally the same:

```
Two objects share DNA if:
  dna.identity.original_type == same
  dna.identity.original_namespace == same
  dna.purpose.statement == semantically equivalent
  dna.core_behavior.primary_action == same
  dna.core_behavior.output_contract == same

If DNA differs → different objects, not versions
If DNA matches but genome differs → same object, different version
```

---

## Rules

| Rule | Statement |
|------|-----------|
| KIL-048 | DNA is sealed at first CANONICAL promotion and never modified thereafter |
| KIL-049 | `dna_hash` must be verified on every read; tampering is a critical security event |
| KIL-050 | `fundamental_constraints` may only grow, never shrink after sealing |
| KIL-051 | `not_in_scope` must list at least one exclusion |
| KIL-052 | `primary_interfaces` with `breaking_change_impact: ECOSYSTEM` require RFC before change |
| KIL-053 | `purpose.derived_from` must reference an existing REQUIREMENT or DECISION object |
| KIL-054 | Changing any DNA field requires deprecating the current object and creating a successor |

---

## Cross-References

- Genome (mutable profile) → `07-KNOWLEDGE-GENOME`
- Evolution (change history) → `11-KNOWLEDGE-EVOLUTION`
- Decision model → `17-KNOWLEDGE-DECISION-MODEL`
- Self-describing → `01-SELF-DESCRIBING-KNOWLEDGE`
