# KNW-KIL-DOC-002 — Executable Knowledge

**Phase:** 3.0D.0.6
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Every specification rule embedded in a Knowledge Object must be machine-readable.
Free-text-only specifications cannot be automatically validated, tested, or enforced.

This document defines the `intelligence.executable` block — structured, typed
representations of every rule, policy, constraint, lifecycle transition, and
relationship invariant.

---

## Schema

```yaml
intelligence:
  executable:
    schema_version: "1.0"

    validation_rules:                  # field-level validation rules
      - id: string                     # e.g. VR-001
        field: string                  # JSON path e.g. "metadata.version"
        rule_type: RANGE | REGEX | ENUM | REQUIRED | UNIQUE | FOREIGN_KEY | CUSTOM
        expression: string             # rule-type-specific expression
        severity: ERROR | WARNING | INFO
        message: string                # human-readable error message

    policies:                          # behavioral policies
      - id: string                     # e.g. POL-001
        name: string
        trigger: string                # event or condition that activates this policy
        condition: string              # boolean expression (CEL syntax)
        action: string                 # what happens when condition is true
        enforcement: BLOCK | WARN | LOG | AUDIT

    lifecycle_rules:                   # lifecycle transition guards
      - from_state: string             # DRAFT | PROPOSED | VERIFIED | CANONICAL | DEPRECATED | ARCHIVED
        to_state: string
        allowed: boolean
        gates:                         # what must be true before transition
          - gate_type: SCORE | EVIDENCE | REVIEW | TEST | APPROVAL
            target: string             # e.g. "quality_score >= 0.80"
        actions:                       # what happens on transition
          - action_type: SET_FIELD | EMIT_EVENT | NOTIFY | LOCK
            target: string
            value: string

    constraint_rules:                  # structural constraints
      - id: string                     # e.g. CR-001
        name: string
        scope: OBJECT | PACKAGE | NAMESPACE | GLOBAL
        expression: string             # evaluable expression
        severity: ERROR | WARNING
        auto_fix: boolean              # whether a fix can be auto-applied

    relationship_rules:                # relationship invariants
      - id: string                     # e.g. RR-001
        relationship_type: string      # DEPENDS_ON | IMPLEMENTS | USES | etc.
        source_type: string            # allowed source object type(s)
        target_type: string            # allowed target object type(s)
        max_count: integer | null
        min_count: integer
        bidirectional: boolean
        inverse_label: string          # label of the inverse relationship

    transition_guards:                 # pre/post conditions for state changes
      - transition_id: string
        pre_conditions:
          - id: string
            expression: string
            failure_message: string
        post_conditions:
          - id: string
            expression: string
            failure_message: string
```

---

## Canonical Example

```yaml
# For: KNW-PLT-MOD-001 (Quota Manager)
intelligence:
  executable:
    schema_version: "1.0"

    validation_rules:
      - id: VR-001
        field: "metadata.version"
        rule_type: REGEX
        expression: "^[0-9]+\\.[0-9]+\\.[0-9]+$"
        severity: ERROR
        message: "Version must follow semver format MAJOR.MINOR.PATCH"
      - id: VR-002
        field: "evidence.items"
        rule_type: RANGE
        expression: "length >= 1"
        severity: ERROR
        message: "Every object must have at least one evidence item"
      - id: VR-003
        field: "metadata.quality_score"
        rule_type: RANGE
        expression: "0.0 <= value <= 1.0"
        severity: ERROR
        message: "Quality score must be between 0.0 and 1.0"

    policies:
      - id: POL-001
        name: "Block promotion without evidence"
        trigger: "lifecycle.transition"
        condition: "to_state == 'CANONICAL' && evidence.items.length == 0"
        action: "Reject transition with error EV-MISSING"
        enforcement: BLOCK
      - id: POL-002
        name: "Warn on stale evidence"
        trigger: "object.read"
        condition: "any(evidence.items, item.age_days > 365)"
        action: "Emit warning STALE-EVIDENCE on object header"
        enforcement: WARN

    lifecycle_rules:
      - from_state: PROPOSED
        to_state: VERIFIED
        allowed: true
        gates:
          - gate_type: SCORE
            target: "quality_score >= 0.70"
          - gate_type: EVIDENCE
            target: "evidence.items.length >= 1"
        actions:
          - action_type: SET_FIELD
            target: "lifecycle.verified_at"
            value: "now()"
          - action_type: EMIT_EVENT
            target: "knowledge.lifecycle.verified"
            value: "knowledge_id"
      - from_state: VERIFIED
        to_state: CANONICAL
        allowed: true
        gates:
          - gate_type: SCORE
            target: "quality_score >= 0.80"
          - gate_type: REVIEW
            target: "reviews.approved >= 2"
        actions:
          - action_type: LOCK
            target: "metadata.knowledge_id"
            value: "immutable"

    constraint_rules:
      - id: CR-001
        name: "No self-dependency"
        scope: OBJECT
        expression: "not any(relationships.DEPENDS_ON, rel.target == self.knowledge_id)"
        severity: ERROR
        auto_fix: false
      - id: CR-002
        name: "Namespace must match package"
        scope: OBJECT
        expression: "metadata.namespace == package.namespace"
        severity: ERROR
        auto_fix: false

    relationship_rules:
      - id: RR-001
        relationship_type: DEPENDS_ON
        source_type: "*"
        target_type: "*"
        min_count: 0
        max_count: null
        bidirectional: true
        inverse_label: DEPENDENCY_OF
      - id: RR-002
        relationship_type: IMPLEMENTS
        source_type: "MODULE | SERVICE | ALGORITHM"
        target_type: "REQUIREMENT | SPECIFICATION | STANDARD"
        min_count: 0
        max_count: null
        bidirectional: true
        inverse_label: IMPLEMENTED_BY

    transition_guards:
      - transition_id: PROPOSED_TO_VERIFIED
        pre_conditions:
          - id: PRE-001
            expression: "quality_score >= 0.70"
            failure_message: "Quality score {quality_score} below 0.70 threshold"
          - id: PRE-002
            expression: "evidence.items.length >= 1"
            failure_message: "No evidence items found"
        post_conditions:
          - id: POST-001
            expression: "lifecycle.verified_at != null"
            failure_message: "verified_at was not set during transition"
```

---

## Expression Language

All `expression` fields use a subset of CEL (Common Expression Language):

| Operation | Syntax |
|-----------|--------|
| Field access | `object.field.subfield` |
| Comparison | `== != < > <= >=` |
| Boolean | `&& \|\| not` |
| String match | `matches(field, "regex")` |
| List length | `list.length` |
| Any/all | `any(list, condition)`, `all(list, condition)` |
| Self-reference | `self.field` |
| Current time | `now()` |
| Elapsed days | `age_days(timestamp_field)` |

---

## Rules

| Rule | Statement |
|------|-----------|
| KIL-009 | Every constraint in `self_describing.constraints` MUST have a corresponding `constraint_rule` in `executable` |
| KIL-010 | Every `validation_rule` severity=ERROR must be checked before any lifecycle promotion |
| KIL-011 | Every `relationship_rule` must match an entry in Phase 3.0C relationship type definitions |
| KIL-012 | `lifecycle_rules` must cover all valid transitions defined in Phase 3.0C lifecycle spec |
| KIL-013 | Every `policy` with `enforcement: BLOCK` must emit a structured error code, never free text only |
| KIL-014 | `transition_guards.post_conditions` failures are an implementation bug, not a user error |

---

## Cross-References

- Self-describing constraints → `01-SELF-DESCRIBING-KNOWLEDGE`
- Lifecycle states → Phase 3.0C `05-LIFECYCLE-MANAGER`
- Relationship types → Phase 3.0C.5 `06-CANONICAL-RELATIONSHIPS`
- Reasoning model → `06-REASONING-MODEL`
