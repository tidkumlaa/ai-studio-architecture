# CAE-DOC-015 — Anti-Hallucination Guard

**Phase:** 3.0D.1
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Anti-Hallucination Guard (AH Guard) is Stage S8 of the CAE pipeline. It runs
after the Context Assembler and before the Validator. It inspects the raw context
pack and injects structured protection signals to prevent downstream AI consumers
from generating false or misleading statements.

The guard does NOT modify knowledge content — it ADDS guard metadata.

---

## Guard Invariant (CI-03)

```
AH Guard ALWAYS runs — it cannot be skipped, suppressed, or made optional.
No PromptPack may leave the CAE without an attached guard block.
(SearchPack and OperatorPack/HumanPack are exempt — see CAE-066.)
```

---

## What the Guard Protects Against

```
H-TYPE-01: Scope Hallucination
  AI claims an object does things outside its declared scope.
  Protection: scope_boundaries block

H-TYPE-02: Confidence Inflation
  AI presents uncertain knowledge as certain.
  Protection: confidence_hedge + hedges block

H-TYPE-03: Missing Context Error
  AI answers a question that requires context not included in the pack.
  Protection: required_context_missing signal

H-TYPE-04: Common Mistake Repetition
  AI reproduces known incorrect patterns about the object.
  Protection: typical_mistakes block

H-TYPE-05: State Anachronism
  AI describes a deprecated or draft object as if it is current.
  Protection: status warning appended to preamble

H-TYPE-06: Relationship Fabrication
  AI invents dependencies or relationships not in the KIL graph.
  Protection: relationship_scope block (list of declared relationships only)
```

---

## Guard Algorithm

```
RUN_AH_GUARD(raw_pack, index) → GuardedPack:

  guard = GuardBlock()
  primary_kil = index.get(raw_pack.primary.knowledge_id)

  # H-TYPE-01: Scope boundaries
  guard.scope_boundaries = []
  guard.scope_boundaries += primary_kil.cortex.why_not      # "what this is NOT"
  guard.scope_boundaries += primary_kil.semantic.scope_exclusions  # if present
  IF not guard.scope_boundaries:
    guard.scope_boundaries.append(
      f"{primary_kil.canonical_name} is a {primary_kil.object_type} "
      f"in namespace {primary_kil.namespace}. Capabilities are limited "
      f"to those declared in its intelligence.semantic.capabilities block."
    )

  # H-TYPE-02: Confidence hedge
  confidence = primary_kil.confidence
  guard.confidence_hedge = confidence.hedging_rule_for_level()
  guard.context_confidence = min(
    obj.confidence.overall_score for obj in raw_pack.all_objects
  )

  # H-TYPE-03: Missing required context
  required_ids = {
    r.knowledge_id
    for r in primary_kil.ai_context.related_objects
    WHERE r.fetch_priority == ALWAYS
  }
  included_ids = {obj.knowledge_id for obj in raw_pack.all_objects}
  guard.missing_required_context = [
    {
      knowledge_id: kid,
      relationship: next(r.relationship for r in primary_kil.ai_context.related_objects
                         WHERE r.knowledge_id == kid)
    }
    for kid in required_ids - included_ids
  ]

  # H-TYPE-04: Common mistakes
  guard.typical_mistakes = primary_kil.cortex.common_errors[:5]

  # H-TYPE-05: Status warning
  IF primary_kil.metadata.state in {DEPRECATED, DRAFT, EXPERIMENTAL}:
    guard.status_warning = (
      f"WARNING: {primary_kil.canonical_name} is in state "
      f"{primary_kil.metadata.state}. Do not treat it as production-ready."
    )
  ELSE:
    guard.status_warning = null

  # H-TYPE-06: Relationship scope
  declared_relationships = [
    f"{r.type}: {r.target_id}"
    for r in primary_kil.reasoning.relationships
  ]
  guard.relationship_scope = declared_relationships
  guard.relationship_scope_note = (
    "Only the above relationships are declared. "
    "Do not infer additional relationships from context."
  )

  # Attach guard to pack
  guarded_pack = raw_pack.copy()
  guarded_pack.guard_block = guard

  RETURN guarded_pack
```

---

## Guard Block Schema

```yaml
guard_block:
  scope_boundaries: [string]            # what this object is NOT
  confidence_hedge: string              # text hedge for the confidence level
  context_confidence: float             # min confidence across all objects
  missing_required_context:             # required objects not in pack
    - knowledge_id: string
      relationship: string
  typical_mistakes: [string]            # common errors to avoid
  status_warning: string | null         # non-null if state is DRAFT/DEPRECATED
  relationship_scope: [string]          # declared relationships only
  relationship_scope_note: string
```

---

## Hedging Rules by Confidence Level

| Level | Score Range | Hedging Rule Text |
|-------|-------------|-------------------|
| AUTHORITATIVE | ≥ 0.95 | "This information is authoritative." |
| CONFIDENT | ≥ 0.80 | "High confidence. Minor details may require verification." |
| PROBABLE | ≥ 0.65 | "Likely correct but not fully verified. Cross-check before acting." |
| UNCERTAIN | ≥ 0.50 | "Uncertain. Treat as hypothesis; verify independently." |
| SPECULATIVE | < 0.50 | "Speculative. Do not use for decisions without independent verification." |

---

## Guard Output in PromptPack

The guard block is rendered verbatim in the `=== GUARD ===` section of
every PromptPack (see `11-PROMPT-PACK`). It appears LAST in the context block.

---

## Rules

| Rule | Statement |
|------|-----------|
| CAE-066 | AH Guard is mandatory for PromptPack — exempt for SearchPack, HumanPack, OperatorPack |
| CAE-067 | Guard must always populate `scope_boundaries` — fallback text is required if KIL block is absent |
| CAE-068 | `context_confidence` is the MINIMUM confidence across all objects, not average |
| CAE-069 | Missing required context must be declared in `missing_required_context` — never silently omitted |
| CAE-070 | Status warning is mandatory when object state is DEPRECATED, DRAFT, or EXPERIMENTAL |

---

## Cross-References

- Context assembler → `10-CONTEXT-ASSEMBLER`
- Context validator → `16-CONTEXT-VALIDATOR`
- Prompt pack → `11-PROMPT-PACK`
- KIL cortex → Phase 3.0D.0.6 `19-KNOWLEDGE-CORTEX`
- KIL confidence → Phase 3.0D.0.6 `13-KNOWLEDGE-CONFIDENCE`
