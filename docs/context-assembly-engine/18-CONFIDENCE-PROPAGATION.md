# CAE-DOC-018 — Confidence Propagation

**Phase:** 3.0D.1
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Confidence Propagation defines how individual object confidence scores combine
into a single `context_confidence` value for the assembled pack, and how
confidence degrades when objects depend on each other.

A pack's overall confidence cannot exceed the minimum confidence of any object
it relies upon. This is the propagation invariant.

---

## Propagation Invariant (CI-06)

```
context_confidence = min(obj.confidence.overall_score
                         for all objects in pack)

NOT: average
NOT: weighted average
ALWAYS: minimum

Rationale: a chain of reasoning is only as confident as its weakest link.
An AI consumer must not be presented with high-context_confidence while
one included object is UNCERTAIN — because that uncertain fact contaminates
the entire reasoning chain.
```

---

## Confidence Degradation by Relationship Type

When object B is included because it relates to object A, B's effective
confidence for the pack may be further degraded based on the relationship:

```
effective_confidence(B, relationship_type):

  base = B.confidence.overall_score

  CASE relationship_type:
    DEPENDS_ON, IMPLEMENTS:
      return base                           # no degradation (direct dependency)

    EXTENDS:
      return base × 0.95                   # slight degradation (extension adds uncertainty)

    RELATED_TO:
      return base × 0.90                   # relationship itself adds uncertainty

    CO_OCCURS_WITH:
      return base × 0.85                   # weaker association

    ALTERNATIVE_TO:
      return base × 0.80                   # alternative may not apply here

    DEPRECATED_BY:
      return base × 0.50                   # deprecated relationship is uncertain

    NOT_SET:
      return base × 0.70                   # unknown relationship type → penalize
```

---

## Transitive Confidence

For multi-hop context (QT-10 MULTI_HOP or INT-04 IMPACT):

```
transitive_confidence(chain):
  # chain = [obj_A, obj_B, obj_C, ...]

  conf = chain[0].confidence.overall_score

  FOR i in range(1, len(chain)):
    rel_type = relationship_type(chain[i-1], chain[i])
    hop_factor = effective_confidence_factor(rel_type)
    conf = conf × hop_factor × chain[i].confidence.overall_score

  return min(conf, 1.0)
```

Transitive confidence is always lower than direct confidence — multi-hop
assemblies should declare this in their guard block.

---

## Confidence Summary in Pack Metadata

```yaml
confidence_summary:
  context_confidence: float            # minimum across all objects (CI-06)
  confidence_level: string             # AUTHORITATIVE | CONFIDENT | PROBABLE | UNCERTAIN | SPECULATIVE
  contributing_objects:
    - knowledge_id: string
      confidence_score: float
      confidence_level: string
      is_minimum: boolean              # true for the object that determines context_confidence
  confidence_limiting_object: string   # knowledge_id of object with lowest confidence
  transitive_chain_confidence: float | null   # only set for MULTI_HOP assemblies
```

---

## Confidence Level Boundaries (Recap)

Defined in KIL Phase 3.0D.0.6, referenced here:

| Level | Range | Hedging |
|-------|-------|---------|
| AUTHORITATIVE | ≥ 0.95 | None required |
| CONFIDENT | ≥ 0.80 | Minor hedging |
| PROBABLE | ≥ 0.65 | Verification recommended |
| UNCERTAIN | ≥ 0.50 | Treat as hypothesis |
| SPECULATIVE | < 0.50 | Do not act on without independent verification |

---

## Guard Block Integration

Confidence propagation informs the guard block:

```
IF context_confidence < 0.65:
  guard.hedges.append(
    f"Context confidence is {context_confidence:.2f} ({level}). "
    f"Limiting object: {limiting_object.canonical_name} "
    f"({limiting_object.confidence.overall_score:.2f}). "
    f"Do not present this context as authoritative."
  )

IF transitive_chain_confidence and transitive_chain_confidence < 0.50:
  guard.hedges.append(
    f"Multi-hop reasoning chain has transitive confidence "
    f"{transitive_chain_confidence:.2f}. Each hop introduces additional uncertainty."
  )
```

---

## Rules

| Rule | Statement |
|------|-----------|
| CAE-081 | `context_confidence` MUST be the minimum, not mean — this is enforced by CI-06 |
| CAE-082 | Effective confidence after relationship degradation may not exceed base confidence |
| CAE-083 | For MULTI_HOP assemblies, transitive_chain_confidence must be computed and stored |
| CAE-084 | Guard block must include confidence hedge if `context_confidence < 0.65` |
| CAE-085 | `confidence_limiting_object` must identify the actual object — not be left null |

---

## Cross-References

- KIL confidence model → Phase 3.0D.0.6 `13-KNOWLEDGE-CONFIDENCE`
- AH Guard → `15-ANTI-HALLUCINATION-GUARD`
- Context quality model → `17-CONTEXT-QUALITY-MODEL` (safety component)
- Prompt pack → `11-PROMPT-PACK` (guard block rendering)
