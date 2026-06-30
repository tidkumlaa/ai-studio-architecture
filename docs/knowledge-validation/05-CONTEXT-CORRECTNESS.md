# KVF-DOC-005 — Context Correctness

**Phase:** 3.0D.1.5
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Context Correctness validates that facts present in assembled context packs
are factually accurate — they agree with the authoritative KnowledgeObject
records and do not introduce errors through compression, rendering, or assembly.

Correctness is distinct from quality: a pack can be complete and relevant
but still contain factually wrong statements.

---

## Correctness Dimensions

```
CC-DIM-1: Field Accuracy
  Are rendered field values identical to source object field values?

CC-DIM-2: Relationship Accuracy
  Do relationship claims in context match the object's declared relationships?

CC-DIM-3: Compression Accuracy
  Does compressed content preserve all critical facts?
  Does it introduce no new claims?

CC-DIM-4: Guard Block Accuracy
  Do scope_boundaries accurately describe what the object is NOT?
  Do typical_mistakes accurately reflect cortex.common_errors?

CC-DIM-5: Confidence Accuracy
  Does context_confidence equal the minimum (not mean) confidence?
  Does confidence_hedge match the object's declared confidence level?
```

---

## Field Accuracy Test

```
For each PromptPack in the test set:
  For each object_rendering in the pack:
    source = KIL_INDEX.get(object_rendering.knowledge_id)
    rendered = object_rendering.content

    FOR each CRITICAL field in rendered:
      extracted_value = extract_field(rendered, field_name)
      source_value = getattr(source, field_name)

      IF extracted_value != source_value:
        RECORD field_accuracy_error(
          knowledge_id, field_name, expected=source_value, actual=extracted_value
        )

CRITICAL fields (must match exactly):
  knowledge_id, canonical_name, object_type, namespace,
  version, metadata.state, dna.identity, dna.purpose
```

---

## Relationship Accuracy Test

```
For each context pack:
  For each object O in the pack:
    source_rels = {(r.type, r.target_id) for r in source.reasoning.relationships}
    rendered_rels = extract_relationships(rendered_content)

    false_rels = rendered_rels - source_rels  # claimed but not declared
    missing_rels = source_rels - rendered_rels  # declared but not rendered (OK)

    IF false_rels:
      RECORD relationship_error(
        knowledge_id, type=FABRICATED, rels=false_rels
      )

Target: false_rels = 0 (fabricated relationships are hallucinations)
Missing relationships are acceptable (compression removes some).
```

---

## Compression Accuracy Test

```
For each object rendered at each compression level (L1–L5):

  At L3 (mini):
    REQUIRED in rendering:
      - knowledge_id
      - canonical_name (or name)
      - purpose (one sentence)
      - at least 2 capabilities
      - at least 2 constraints
    FORBIDDEN in rendering:
      - Any fact not derivable from the source object
      - Any relationship not declared in source

  At L4 (standard):
    REQUIRED in rendering: all L3 requirements plus
      - at least 3 decisions
      - confidence.overall_score
    FORBIDDEN: any fabricated field value

  Correctness score per level =
    required_fields_present / total_required
    AND
    fabricated_claims == 0
```

---

## Correctness Scoring

```yaml
correctness_result:
  objects_tested: integer
  packs_tested: integer

  field_accuracy:
    total_critical_fields_checked: integer
    fields_correct: integer
    fields_wrong: integer
    accuracy_rate: float

  relationship_accuracy:
    total_claims_checked: integer
    correct_claims: integer
    fabricated_claims: integer
    fabrication_rate: float

  compression_accuracy_by_level:
    L1: {required_present_rate: float, fabrication_rate: float}
    L2: {required_present_rate: float, fabrication_rate: float}
    L3: {required_present_rate: float, fabrication_rate: float}
    L4: {required_present_rate: float, fabrication_rate: float}
    L5: {required_present_rate: float, fabrication_rate: float}

  overall_correctness_score: float
  critical_failures:                    # fabrication_rate > 0 is always critical
    - type: string
      count: integer
```

---

## Correctness Thresholds

| Metric | Minimum | Target |
|--------|---------|--------|
| Field accuracy rate | ≥ 0.999 | 1.000 |
| Relationship fabrication rate | 0.000 | 0.000 |
| L3 required fields present | ≥ 0.95 | ≥ 0.99 |
| Overall correctness score | ≥ 0.99 | 1.000 |

Any relationship fabrication rate > 0 is a critical failure regardless of score.

---

## Rules

| Rule | Statement |
|------|-----------|
| KVF-021 | Relationship fabrication rate must be 0.000 — any fabricated relationship is a critical failure |
| KVF-022 | Field accuracy is measured on CRITICAL fields only — optional fields are excluded |
| KVF-023 | Correctness tests must be run separately from quality tests — they are not the same |
| KVF-024 | Compression accuracy must be tested at all 5 levels — testing L4 only is insufficient |
| KVF-025 | Guard block accuracy must be verified against cortex.common_errors, not generated text |

---

## Cross-References

- Context quality validation → `04-CONTEXT-QUALITY-VALIDATION`
- Hallucination measurement → `10-HALLUCINATION-MEASUREMENT`
- KIL cortex → Phase 3.0D.0.6 `19-KNOWLEDGE-CORTEX`
- Compression spec → Phase 3.0D.0.6 `05-KNOWLEDGE-COMPRESSION`
