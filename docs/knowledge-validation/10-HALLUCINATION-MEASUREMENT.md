# KVF-DOC-010 — Hallucination Measurement

**Phase:** 3.0D.1.5
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Hallucination Measurement defines a taxonomy of 6 hallucination types that
can occur when an AI agent uses KOS context, protocols to detect each type,
and rate targets that must be met for certification.

---

## 6 Hallucination Types

### Type A — Wrong Fact

```
Definition:
  The AI states a fact about a KnowledgeObject that directly contradicts
  the object's KIL fields.

Examples:
  - "The Quota Manager has a rate of 100 req/s" when the actual rate is 1,000/s
  - "The Quota Manager is DEPRECATED" when it is CANONICAL
  - "The Quota Manager was created in 2020" when it was created in 2023

Detection method:
  Extract factual claims from AI answer.
  Verify each claim against source KIL object fields.
  Any contradiction = Type A hallucination.

Severity: CRITICAL
```

### Type B — Wrong Dependency

```
Definition:
  The AI claims a dependency relationship that is not declared in the KIL
  intelligence.reasoning.requires or intelligence.semantic blocks.

Examples:
  - "The Quota Manager depends on the Cache Service" when it does not
  - "The Quota Manager is used by the Payment Service" when there is no
    such relationship declared

Detection method:
  Extract all dependency claims from AI answer.
  Check each against reasoning.requires, semantic.consumes, semantic.provides.
  Any undeclared dependency = Type B hallucination.

Severity: CRITICAL
```

### Type C — Wrong Architecture

```
Definition:
  The AI misclassifies the architectural role, type, or design of an object.

Examples:
  - "The Quota Manager is a SERVICE" when it is a MODULE
  - "The Quota Manager is a stateless function" when it maintains state
  - "The Quota Manager is part of the auth package" when it is in platform.core

Detection method:
  Extract architectural claims (type, namespace, package, design pattern).
  Verify each against genome.category, object_type, namespace, dna.core_behavior.
  Any mismatch = Type C hallucination.

Severity: HIGH
```

### Type D — Wrong Relationship

```
Definition:
  The AI claims a non-dependency semantic relationship that is not declared.

Examples:
  - "The Quota Manager implements the RateLimiter interface" when it does not
  - "The Quota Manager extends BasePolicy" when it does not
  - "The Quota Manager replaces the old ThrottleService" when that is not declared

Detection method:
  Extract all semantic relationships from AI answer.
  Verify against semantic.implements, semantic.extends, semantic.replaces, semantic.conflicts.
  Any undeclared relationship = Type D hallucination.

Severity: HIGH
```

### Type E — Wrong Confidence

```
Definition:
  The AI presents information with a higher confidence level than the
  KIL object's declared confidence allows.

Examples:
  - AI states a fact as certain when the object's confidence is UNCERTAIN (< 0.65)
  - AI does not hedge a claim when the context_confidence is PROBABLE
  - AI ignores the guard block's confidence_hedge

Detection method:
  Compare the certainty of language in the AI answer to context_confidence.
  If context_confidence < 0.80 and AI uses language like "definitely/always/certainly"
  without hedging: Type E hallucination.

Severity: MEDIUM
```

### Type F — Unsupported Conclusion

```
Definition:
  The AI draws a conclusion that cannot be derived from any field in the
  context pack — even through valid reasoning steps.

Examples:
  - "Based on the Quota Manager's architecture, the system likely uses Redis"
    (when no Redis is mentioned anywhere in the context)
  - "The Quota Manager was probably designed by the platform team"
    (when authorship is not in the context)

Detection method:
  For each conclusion in the AI answer, verify that it can be derived
  from at least one field in the context pack through ≤ 3 reasoning steps.
  If not derivable: Type F hallucination.

Severity: MEDIUM-LOW
```

---

## Hallucination Detection Pipeline

```
DETECT_HALLUCINATIONS(query, context_pack, ai_answer):

  claims = EXTRACT_CLAIMS(ai_answer)
  hallucinations = []

  FOR claim in claims:
    source = find_source_object(claim, context_pack)

    # Type A: factual contradiction
    IF source and contradicts(claim, source.fields):
      hallucinations.append(Hallucination(type=A, claim, evidence=source.field))

    # Type B: undeclared dependency
    IF is_dependency_claim(claim):
      IF not in declared_dependencies(context_pack.all_objects):
        hallucinations.append(Hallucination(type=B, claim))

    # Type C: wrong architecture
    IF is_architecture_claim(claim):
      IF contradicts_architecture(claim, source.genome, source.object_type):
        hallucinations.append(Hallucination(type=C, claim))

    # Type D: wrong relationship
    IF is_semantic_relationship_claim(claim):
      IF not in declared_semantic_rels(context_pack.all_objects):
        hallucinations.append(Hallucination(type=D, claim))

    # Type E: wrong confidence
    IF is_certainty_expression(claim) and context_pack.context_confidence < 0.80:
      IF not hedged_appropriately(claim, context_pack.guard_block):
        hallucinations.append(Hallucination(type=E, claim))

    # Type F: unsupported conclusion
    IF is_conclusion(claim):
      IF not derivable_from_context(claim, context_pack, max_steps=3):
        hallucinations.append(Hallucination(type=F, claim))

  RETURN hallucinations
```

---

## Hallucination Rate Measurement

```
For each query Q with AI answer A and context pack P:

  hallucinations = DETECT_HALLUCINATIONS(Q, P, A)
  hallucination_rate(Q) = len(hallucinations) / len(claims_in_A)

Aggregate:
  overall_hallucination_rate = mean(hallucination_rate(Q) for all Q)
  critical_hallucination_rate = (Type_A + Type_B) / total_claims
  total_hallucination_events = sum(len(hallucinations(Q)) for all Q)
```

---

## Hallucination Measurement Result Schema

```yaml
hallucination_result:
  queries_evaluated: integer
  total_claims_extracted: integer
  total_hallucinations: integer
  overall_hallucination_rate: float

  by_type:
    type_a: {count: integer, rate: float}    # wrong fact
    type_b: {count: integer, rate: float}    # wrong dependency
    type_c: {count: integer, rate: float}    # wrong architecture
    type_d: {count: integer, rate: float}    # wrong relationship
    type_e: {count: integer, rate: float}    # wrong confidence
    type_f: {count: integer, rate: float}    # unsupported conclusion

  critical_hallucination_rate: float        # (A + B) / total claims
  guard_block_adherence_rate: float         # % where guard block prevented Type E
```

---

## Hallucination Rate Targets

| Type | BRONZE | SILVER | GOLD | ENTERPRISE |
|------|--------|--------|------|------------|
| Type A (wrong fact) | < 5% | < 3% | < 1% | < 0.5% |
| Type B (wrong dependency) | < 5% | < 3% | < 1% | < 0.5% |
| Type C (wrong architecture) | < 8% | < 5% | < 2% | < 1% |
| Type D (wrong relationship) | < 8% | < 5% | < 2% | < 1% |
| Type E (wrong confidence) | < 15% | < 10% | < 5% | < 2% |
| Type F (unsupported) | < 20% | < 15% | < 8% | < 4% |
| Critical (A+B) rate | < 10% | < 5% | < 2% | < 1% |

---

## Rules

| Rule | Statement |
|------|-----------|
| KVF-046 | Type A and Type B are CRITICAL — any rate above GOLD threshold blocks Gold certification |
| KVF-047 | Hallucination detection must extract and classify all verifiable claims, not a sample |
| KVF-048 | Type E detection requires comparison to guard_block.confidence_hedge — not just score |
| KVF-049 | Type F derivability check must use the context pack, not the full KIL corpus |
| KVF-050 | Hallucination results must include sample hallucinations for each type found |

---

## Cross-References

- Reasoning quality → `09-REASONING-QUALITY`
- Anti-hallucination guard → Phase 3.0D.1 `15-ANTI-HALLUCINATION-GUARD`
- Confidence propagation → Phase 3.0D.1 `18-CONFIDENCE-PROPAGATION`
- KIL cortex → Phase 3.0D.0.6 `19-KNOWLEDGE-CORTEX`
