# KVF-DOC-016 — Dependency Validation

**Phase:** 3.0D.1.5
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Dependency Validation verifies the correctness and completeness of declared
dependency relationships. Correct dependencies are essential for impact analysis,
graph traversal, and understanding blast radius of changes.

---

## Dependency Types to Validate

```
HARD DEPENDS_ON:
  Object A cannot function without object B.
  Most critical — must be complete and correct.

SOFT DEPENDS_ON:
  Object A has optional dependency on B.
  Required if present in code/config.

COMPONENT_OF:
  Object A is a part of object B.
  Parent-child structural dependency.

IMPLEMENTS:
  Object A implements interface B.
  Contract dependency — must be bidirectional (B has IMPLEMENTED_BY A).

DEPENDENCY_OF:
  Inverse of DEPENDS_ON — B declares that A depends on it.
  Must be consistent with A's DEPENDS_ON declaration.
```

---

## Dependency Validation Checks

### DV-1: Completeness

```
DV-1.01: Every object used in production has at least 1 declared relationship
DV-1.02: Objects with COMPONENT_OF must have the parent in the corpus
DV-1.03: Objects with IMPLEMENTS must have the interface object in the corpus
DV-1.04: reasoning.requires lists match declared HARD DEPENDS_ON relationships
```

### DV-2: Consistency

```
DV-2.01: If A DEPENDS_ON B, then B must not DEPENDS_ON A (no cycles in HARD deps)
DV-2.02: If A IMPLEMENTS B, then B has IMPLEMENTED_BY A
DV-2.03: If A is COMPONENT_OF B, then B has CONSISTS_OF A
DV-2.04: reasoning.requires must be a subset of declared HARD DEPENDS_ON targets
```

### DV-3: Impact Analysis Accuracy

```
For each object O with declared reasoning.impacts:
  DV-3.01: Every impact target exists in the corpus
  DV-3.02: CRITICAL impacts are for objects with direct DEPENDENCY_OF relationship
  DV-3.03: No impact target is listed more than once
  DV-3.04: Impact severity levels are from declared vocabulary (CRITICAL/HIGH/MEDIUM/LOW)
```

### DV-4: Dependency Depth Sanity

```
DV-4.01: No HARD dependency chain exceeds depth 15
           (deeper chains indicate architecture problems, not KOS errors)
DV-4.02: Objects with HARD dependency depth > 10 have a documented reason
DV-4.03: The mean HARD dependency depth across corpus is <= 5
```

---

## Dependency Completeness Score

```
For each object O:
  declared_hard_deps = count(O.reasoning.requires)
  actual_hard_deps = count(edges WHERE type=HARD_DEPENDS_ON and source=O)

  dep_completeness(O) = 1 if declared_hard_deps == actual_hard_deps else 0

  overall_dep_completeness = mean(dep_completeness(O) for all O)
  Target: >= 0.99
```

---

## Dependency Validation Result Schema

```yaml
dependency_validation_result:
  objects_evaluated: integer

  completeness:
    objects_with_no_relationships: integer
    component_of_target_missing: integer
    implements_target_missing: integer
    dep_completeness_rate: float

  consistency:
    bidirectional_implements_rate: float
    bidirectional_component_rate: float
    requires_dep_consistency_rate: float

  impact_accuracy:
    targets_exist_rate: float
    severity_valid_rate: float

  depth_analysis:
    max_hard_dep_depth: integer
    mean_hard_dep_depth: float
    objects_depth_gt_10: integer

  cycles_in_hard_deps: integer          # must be 0 (see graph validation)
```

---

## Rules

| Rule | Statement |
|------|-----------|
| KVF-076 | DV-2.01 (no HARD dep cycles) is re-verified here independently from graph validation |
| KVF-077 | DV-1.04 (reasoning.requires matches HARD DEPENDS_ON) must be 100% — mismatch = documentation bug |
| KVF-078 | DV-2.02 (IMPLEMENTS bidirectionality) failure triggers interface relationship review |
| KVF-079 | HARD dependency depth > 15 is a structural anomaly requiring architecture review |
| KVF-080 | Impact severity levels must use CRITICAL/HIGH/MEDIUM/LOW exactly — not abbreviated |

---

## Cross-References

- Graph validation → `12-GRAPH-VALIDATION`
- Traceability validation → `15-TRACEABILITY-VALIDATION`
- Reasoning model → Phase 3.0D.0.6 `06-REASONING-MODEL`
- Object selector (graph traversal) → Phase 3.0D.1 `05-OBJECT-SELECTOR`
