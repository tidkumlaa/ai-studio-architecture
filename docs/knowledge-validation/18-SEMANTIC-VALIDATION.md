# KVF-DOC-018 — Semantic Validation

**Phase:** 3.0D.1.5
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Semantic Validation verifies that the declared semantic relationships between
KnowledgeObjects are correct — that objects claiming to implement, extend,
replace, or conflict with each other actually have the correct semantic
relationship as evidenced by their content.

---

## Semantic Relationship Types to Validate

```
intelligence.semantic:
  capabilities:   what the object provides
  provides:       interfaces/APIs the object exposes
  consumes:       interfaces/APIs the object uses
  extends:        objects this object extends (inheritance)
  implements:     interfaces this object satisfies
  replaces:       objects this object supersedes
  conflicts:      objects that cannot coexist with this object
  equivalents:    objects semantically equivalent to this object
```

---

## Semantic Validation Checks

### SV-1: Capability Consistency

```
For each capability declared in semantic.capabilities:
  SV-1.01: Capability has a name, category, and description
  SV-1.02: Capability category is from declared vocabulary (8 categories from KIL doc 03)
  SV-1.03: dna.core_behavior mentions the primary capability (semantic alignment)
  SV-1.04: If capability claims "provides X", then semantic.provides includes X
```

### SV-2: Provides / Consumes Consistency

```
SV-2.01: Every item in semantic.provides has a corresponding object or interface in corpus
           OR is a well-defined external interface
SV-2.02: Every item in semantic.consumes has a corresponding DEPENDS_ON relationship
SV-2.03: semantic.provides does not overlap with semantic.consumes for the same interface
```

### SV-3: Extends / Implements Validation

```
For each item E in semantic.extends:
  SV-3.01: E exists in the corpus
  SV-3.02: E is an ancestor type of this object (not a sibling)
  SV-3.03: This object's capabilities are a superset of E's declared capabilities

For each item I in semantic.implements:
  SV-3.04: I exists in the corpus and has object_type INTERFACE
  SV-3.05: This object provides all capabilities declared in I.semantic.capabilities
```

### SV-4: Replaces / Conflicts Validation

```
For each item R in semantic.replaces:
  SV-4.01: R exists in corpus and is in state DEPRECATED
  SV-4.02: This object covers the primary capabilities of R
  SV-4.03: R has DEPRECATED_BY relationship pointing to this object

For each item C in semantic.conflicts:
  SV-4.04: C exists in the corpus
  SV-4.05: C also declares conflict with this object (symmetric)
```

### SV-5: Equivalents Validation

```
For each item Q in semantic.equivalents:
  SV-5.01: Q exists in the corpus
  SV-5.02: Q also declares this object in its equivalents (symmetric)
  SV-5.03: intelligence.diff.object_comparison similarity_score >= 0.70
```

---

## Semantic Coherence Score

```
semantic_coherence_score(O) =
  (capability_consistency × 0.30) +
  (provides_consumes_consistency × 0.25) +
  (extends_implements_validity × 0.25) +
  (replaces_conflicts_validity × 0.20)

Target mean: >= 0.85
```

---

## Semantic Validation Result Schema

```yaml
semantic_validation_result:
  objects_evaluated: integer

  capability_consistency_rate: float
  provides_consumes_consistency_rate: float
  extends_implements_validity_rate: float
  replaces_conflicts_validity_rate: float
  equivalents_symmetry_rate: float

  mean_semantic_coherence_score: float

  critical_failures:
    - type: string
      count: integer
      examples: [string]

  warnings:
    - type: string
      count: integer
```

---

## Rules

| Rule | Statement |
|------|-----------|
| KVF-086 | SV-3.04 (implements target is INTERFACE type) must be checked — non-INTERFACE "implements" is wrong |
| KVF-087 | SV-4.01 (replaces target is DEPRECATED) must be exact — ACTIVE object in replaces is incorrect |
| KVF-088 | SV-4.05 (conflicts symmetry) failure is a warning at BRONZE, failure at GOLD |
| KVF-089 | SV-5.03 (equivalents similarity >= 0.70) requires similarity from KIL diff block |
| KVF-090 | Semantic coherence score < 0.70 is a corpus quality problem, not a Runtime problem |

---

## Cross-References

- KIL semantic knowledge → Phase 3.0D.0.6 `03-SEMANTIC-KNOWLEDGE`
- KIL diff model → Phase 3.0D.0.6 `12-KNOWLEDGE-DIFF`
- Dependency validation → `16-DEPENDENCY-VALIDATION`
- Knowledge conformance → `02-KNOWLEDGE-CONFORMANCE`
