# KVF-DOC-015 — Traceability Validation

**Phase:** 3.0D.1.5
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Traceability Validation verifies that the knowledge graph supports complete
traceability chains — from business requirements down through architecture,
components, implementation, and tests. KOS claims that every architectural
decision is traceable; this document defines how to prove that claim.

---

## Traceability Chain Structure

```
Level 1: BUSINESS_REQUIREMENT
  WHY this exists at the business level

Level 2: ARCHITECTURE_DECISION
  HOW the requirement shaped the architecture (from decisions[])

Level 3: CAPABILITY
  WHAT the object does (from semantic.capabilities)

Level 4: IMPLEMENTATION
  HOW it is implemented (from genome.relationships)

Level 5: TEST
  HOW it is verified (from TESTED_BY relationships)

A complete chain runs from Level 1 to Level 5.
A partial chain (e.g., Levels 2–5 only) is valid if Level 1 is N/A.
```

---

## Traceability Validation Checks

### TV-1: Chain Completeness

```
For each CANONICAL object O:

  TV-1.01: O has at least one architecture_decision in decisions[]
  TV-1.02: O has at least one capability in semantic.capabilities
  TV-1.03: O has at least one TESTED_BY relationship (or exemption note)
  TV-1.04: If O has implementation relationship: O has IMPLEMENTS or COMPONENT_OF

Chain completeness score per object:
  chain_completeness = levels_present / 5
  (Level 1 is optional: use 4 as denominator if no business requirement context)
```

### TV-2: Horizontal Traceability

```
Can we trace FROM a test TO the requirement it covers?

For each test object T (object_type == TEST or has TESTS relationship):
  TV-2.01: T has at least one TESTS relationship to an implementation object
  TV-2.02: The implementation object has a chain to a capability
  TV-2.03: The capability is declared in the parent object's semantic.capabilities
```

### TV-3: Decision Traceability

```
For each object O:
  TV-3.01: Every constraint in dna.fundamental_constraints has an associated
           decision in decisions[] explaining why that constraint exists
  TV-3.02: Every alternative in intelligence.alternatives has a REJECTED
           status if not chosen, with rejection_reason non-empty
```

### TV-4: Impact Traceability

```
For each CRITICAL or HIGH impact declared in reasoning.impacts:
  TV-4.01: The impact target exists in the corpus
  TV-4.02: The impact target declares a DEPENDENCY_OF relationship back
            (bidirectional impact traceability)
```

---

## Traceability Coverage Metric

```
traceability_coverage =
  count(objects with complete chain, >= 4 of 5 levels)
  / count(CANONICAL + ACTIVE objects)

Target: >= 0.80

Orphan rate:
  orphan = object with chain_completeness < 0.40
  orphan_rate = count(orphans) / count(objects)
  Target: <= 0.10
```

---

## Traceability Validation Result Schema

```yaml
traceability_validation_result:
  objects_evaluated: integer

  chain_completeness:
    mean: float
    above_0_80: integer                  # count with ≥ 4/5 levels
    coverage_rate: float

  by_level:
    level_1_present_rate: float          # business requirement
    level_2_present_rate: float          # architecture decision
    level_3_present_rate: float          # capability
    level_4_present_rate: float          # implementation
    level_5_present_rate: float          # test

  horizontal_traceability_rate: float    # tests → requirements linkage
  decision_traceability_rate: float      # constraints → decisions linkage
  impact_bidirectionality_rate: float    # TV-4 check

  orphan_count: integer
  orphan_rate: float
```

---

## Rules

| Rule | Statement |
|------|-----------|
| KVF-071 | Level 5 (test) absence must be documented with exemption reason — silent absence fails TV-1.03 |
| KVF-072 | Orphan rate > 0.20 blocks SILVER certification — traceability is foundational to KOS |
| KVF-073 | TV-4 (impact bidirectionality) is a warning, not a failure — bidirectionality is advisory |
| KVF-074 | Decision traceability check must use the object's own decisions[] — inherited decisions are out of scope |
| KVF-075 | Traceability coverage must be measured on CANONICAL objects only — DRAFT/EXPERIMENTAL are excluded |

---

## Cross-References

- Dependency validation → `16-DEPENDENCY-VALIDATION`
- Reasoning model → Phase 3.0D.0.6 `06-REASONING-MODEL`
- KIL decision model → Phase 3.0D.0.6 `17-KNOWLEDGE-DECISION-MODEL`
- CAE context planner (traceability intent) → Phase 3.0D.1 `09-CONTEXT-PLANNER`
