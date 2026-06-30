# KNW-KIL-DOC-012 — Knowledge Diff

**Phase:** 3.0D.0.6
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Text diff compares bytes. Knowledge diff compares meaning.

A semantic diff between two Knowledge Objects (or two versions of the same object)
captures what changed in terms of behavior, capabilities, interfaces, relationships,
and reasoning — not which lines of YAML were edited.

This document defines the `intelligence.diff` schema and the semantic diff protocol.

---

## Diff Dimensions

| Dimension | What Changes | Severity |
|-----------|-------------|---------|
| identity | knowledge_id, type, namespace, canonical_name | STRUCTURAL |
| purpose | Why the object exists | SEMANTIC |
| behavior | What the object does | SEMANTIC |
| capabilities | What it can do | FUNCTIONAL |
| interfaces | Input/output contracts | BREAKING |
| relationships | Dependencies, implementations | STRUCTURAL |
| constraints | Hard limits | BEHAVIORAL |
| evidence | Supporting evidence | QUALITY |
| quality | Score, confidence | QUALITY |
| metadata | Tags, labels, ownership | INFORMATIONAL |

---

## Schema

```yaml
intelligence:
  diff:
    # Schema for expressing semantic differences between object versions
    # or between two distinct objects

    object_diff:                       # diff between two versions of this object
      from_version: string
      to_version: string
      computed_at: string

      changes:
        - dimension: string            # from Diff Dimensions table
          field_path: string           # e.g. "core_behavior.primary_action"
          change_type: ADDED | REMOVED | MODIFIED | REORDERED
          severity: BREAKING | SEMANTIC | FUNCTIONAL | STRUCTURAL | INFORMATIONAL
          from_value: string | null    # human-readable previous value
          to_value: string | null      # human-readable new value
          impact: string               # what this change means for consumers
          migration_required: boolean

      summary:
        breaking_changes: integer
        semantic_changes: integer
        functional_changes: integer
        structural_changes: integer
        informational_changes: integer
        migration_required: boolean
        overall_severity: BREAKING | MAJOR | MINOR | PATCH

    object_comparison:                 # diff between two distinct objects
      compare_to_id: string
      comparison_type: VERSION | ALTERNATIVE | EQUIVALENT | PREDECESSOR | SUCCESSOR

      similarity:
        dna_match: float               # 0.0–1.0 — how similar their DNA is
        purpose_similarity: float      # semantic similarity of purpose statements
        capability_overlap: float      # % of capabilities in common
        interface_compatibility: float # % of interfaces that are compatible
        overall_similarity: float      # weighted composite

      differences:
        - dimension: string
          this_value: string
          other_value: string
          significance: DEFINING | IMPORTANT | MINOR | COSMETIC
          notes: string
```

---

## Semantic Diff Algorithm

```
SemanticDiff(object_A, object_B):

  1. DNA Diff
     Compare: dna.identity, dna.purpose, dna.core_behavior,
              dna.primary_interfaces, dna.fundamental_constraints
     If dna.identity differs → STRUCTURAL change (likely different objects)
     If dna.purpose differs → SEMANTIC change
     If dna.core_behavior.primary_action differs → SEMANTIC change
     If dna.primary_interfaces differ → may be BREAKING

  2. Capability Diff
     Compare: semantic.capabilities
     Added capability → FUNCTIONAL change (backward-compatible)
     Removed capability → BREAKING change
     Modified capability maturity STABLE→EXPERIMENTAL → BREAKING

  3. Interface Diff
     Compare: dna.primary_interfaces + self_describing.inputs/outputs
     Added optional input → backward-compatible
     Changed required input → BREAKING
     Changed output contract → BREAKING

  4. Relationship Diff
     Compare: genome.relationships
     Added relationship → STRUCTURAL (usually non-breaking)
     Removed HARD dependency → BREAKING (object may fail)
     Changed IMPLEMENTS target → SEMANTIC change

  5. Constraint Diff
     Compare: self_describing.constraints + executable.constraint_rules
     Added constraint → BEHAVIORAL change
     Removed constraint → BEHAVIORAL change (may be BREAKING for downstream)

  6. Quality Diff
     Compare: metadata.quality_score, confidence, evidence
     Quality decrease > 0.10 → QUALITY regression warning
     Evidence stale or removed → QUALITY warning
```

---

## Canonical Example: Version Diff

```yaml
# KNW-PLT-MOD-001 v1.5.0 → v2.0.0
intelligence:
  diff:
    object_diff:
      from_version: "1.5.0"
      to_version: "2.0.0"
      computed_at: "2025-09-01T00:00:00Z"

      changes:
        - dimension: "interfaces"
          field_path: "dna.primary_interfaces[0].QuotaCheck"
          change_type: MODIFIED
          severity: BREAKING
          from_value: "QuotaCheck(tenant_id: str, resource_type: str, amount: float)"
          to_value: "QuotaCheck(tenant_id=str, resource_type=str, requested_amount=float)"
          impact: "All callers using positional arguments must update to named parameters"
          migration_required: true
        - dimension: "metadata"
          field_path: "metadata.version"
          change_type: MODIFIED
          severity: INFORMATIONAL
          from_value: "1.5.0"
          to_value: "2.0.0"
          impact: "None"
          migration_required: false

      summary:
        breaking_changes: 1
        semantic_changes: 0
        functional_changes: 0
        structural_changes: 0
        informational_changes: 1
        migration_required: true
        overall_severity: BREAKING
```

## Canonical Example: Object Comparison

```yaml
# KNW-PLT-MOD-001 vs KNW-ALG-ALG-009 (Rate Limiter — often confused)
intelligence:
  diff:
    object_comparison:
      compare_to_id: KNW-ALG-ALG-009
      comparison_type: ALTERNATIVE

      similarity:
        dna_match: 0.31
        purpose_similarity: 0.55
        capability_overlap: 0.20
        interface_compatibility: 0.15
        overall_similarity: 0.30

      differences:
        - dimension: "purpose"
          this_value: "Enforce cumulative per-tenant resource quotas over time"
          other_value: "Limit request rate per window (requests/second)"
          significance: DEFINING
          notes: "Quota is about total resource consumption; rate limiting is about request velocity"
        - dimension: "capabilities"
          this_value: "GUARD (enforcement), MONITOR (reporting)"
          other_value: "GUARD (rate enforcement), ALGORITHM (token bucket)"
          significance: IMPORTANT
          notes: "Both are guards but they guard different dimensions"
        - dimension: "interfaces"
          this_value: "QuotaCheck: (tenant, resource, amount) → ALLOW/DENY"
          other_value: "RateCheck: (client_id, endpoint) → ALLOW/DENY"
          significance: DEFINING
          notes: "Incompatible interfaces — cannot be swapped"
```

---

## Rules

| Rule | Statement |
|------|-----------|
| KIL-071 | `object_diff.summary.overall_severity` must be BREAKING if any change has severity BREAKING |
| KIL-072 | Every BREAKING change must have `migration_required: true` and a non-null `impact` |
| KIL-073 | DNA diff must be the first step in any semantic diff computation |
| KIL-074 | `object_comparison.similarity.overall_similarity` must use fixed weights, not heuristics |
| KIL-075 | Objects with `overall_similarity > 0.80` must be reviewed for deduplication |

---

## Cross-References

- Evolution (change history) → `11-KNOWLEDGE-EVOLUTION`
- DNA (immutable baseline) → `08-KNOWLEDGE-DNA`
- Alternative model → `18-KNOWLEDGE-ALTERNATIVE-MODEL`
- Certification diff → Phase 3.0D.0 `24-EVOLUTION-CERTIFICATION`
