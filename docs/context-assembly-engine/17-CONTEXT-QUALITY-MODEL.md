# CAE-DOC-017 — Context Quality Model

**Phase:** 3.0D.1
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Context Quality Model defines how to measure whether an assembled context pack
is fit for purpose. Quality is measured at the pack level, not the object level
(object quality is measured by the KIL quality model in Phase 3.0D.0.6).

A pack with high-quality objects assembled in the wrong structure, at the wrong
compression, for the wrong intent is still a low-quality pack.

---

## Context Quality Score Formula

```
context_quality_score =
  relevance    × 0.35 +
  completeness × 0.30 +
  safety       × 0.25 +
  efficiency   × 0.10

Minimum to emit: 0.70
All components: 0.0–1.0
```

---

## Component Definitions

### relevance (weight 0.35)

```
relevance =
  (weighted_avg_relevance_score × 0.60) +
  (intent_alignment × 0.40)

where:
  weighted_avg_relevance_score = avg(obj.relevance_score for all objects in pack)

  intent_alignment:
    = 1.0 if primary_object matches primary entity of query
    = 0.8 if primary entity could not be resolved (AMBIGUOUS)
    = 0.5 if primary entity was resolved by fallback
    = 0.0 if wrong entity in primary slot
```

### completeness (weight 0.30)

```
completeness =
  (primary_depth × 0.50) +
  (required_context_coverage × 0.30) +
  (guard_completeness × 0.20)

where:
  primary_depth:
    L5 → 1.00
    L4 → 0.85
    L3 → 0.65
    L2 → 0.30   # below minimum — should not happen for primary
    L1 → 0.10

  required_context_coverage:
    required = count(ai_context.related_objects WHERE priority==ALWAYS)
    included = count(required objects in pack)
    = included / required   (1.0 if required == 0)

  guard_completeness:
    all required guard fields present → 1.0
    missing confidence_hedge → 0.5
    missing scope_boundaries → 0.3
    guard absent → 0.0
    (for non-PromptPack: guard_completeness = 1.0)
```

### safety (weight 0.25)

```
safety =
  (airs_floor × 0.50) +
  (confidence_floor × 0.30) +
  (status_risk × 0.20)

where:
  airs_floor = min(obj.ai_readiness_score for all objects in pack)
    # High min AIRS → low hallucination risk

  confidence_floor = min(obj.confidence.overall_score for all objects)
    # Minimum confidence across pack

  status_risk:
    no DEPRECATED or EXPERIMENTAL objects → 1.0
    has EXPERIMENTAL: → 0.8
    has DEPRECATED: → 0.5
    primary is DEPRECATED or DRAFT: → 0.0
```

### efficiency (weight 0.10)

```
efficiency =
  (utilization_band × 0.60) +
  (object_density × 0.40)

where:
  utilization_band:
    # Good utilization: not too low (wasted), not over budget
    utilization_rate = pack_metadata.total_tokens / budget_declared
    0.70 ≤ rate ≤ 1.00 → 1.00
    0.50 ≤ rate <  0.70 → 0.70
    rate < 0.50         → 0.40
    rate > 1.00         → 0.00  # over-budget pack fails V-08 before quality check

  object_density:
    # Useful objects per 1000 tokens
    density = objects_included / (total_tokens / 1000)
    density >= 3.0 → 1.0
    density >= 2.0 → 0.8
    density >= 1.0 → 0.6
    density <  1.0 → 0.4
```

---

## Quality Score Thresholds

| Score | Quality Level | Disposition |
|-------|--------------|-------------|
| ≥ 0.90 | EXCELLENT | Emit with no warning |
| ≥ 0.80 | GOOD | Emit normally |
| ≥ 0.70 | ACCEPTABLE | Emit — minimum threshold |
| ≥ 0.60 | MARGINAL | Reject — log and suggest retry with larger budget |
| < 0.60 | POOR | Reject with AssemblyError(QUALITY_TOO_LOW) |

---

## Quality Report

Every pack that passes includes a quality report in pack_metadata:

```yaml
quality_report:
  context_quality_score: float
  quality_level: string              # EXCELLENT | GOOD | ACCEPTABLE | ...
  components:
    relevance: float
    completeness: float
    safety: float
    efficiency: float
  improvement_hints: [string]        # actionable hints if quality < 0.90
```

---

## Improvement Hints

```
IF relevance < 0.70:
  hints.append("Primary entity match confidence is low — consider rephrasing the query.")

IF completeness.required_context_coverage < 0.80:
  missing = required - included
  hints.append(f"Required context missing: {missing} object(s). Increase budget.")

IF completeness.primary_depth < 0.65:
  hints.append("Primary object at L3 (minimum). Increase budget for richer detail.")

IF safety.airs_floor < 0.75:
  hints.append(f"Low AIRS object in pack: {min_airs_obj}. Improve KIL block completeness.")

IF efficiency.utilization_band == 0.40:
  hints.append("Budget under-utilized (<50%). Consider reducing budget for this query type.")
```

---

## Pack Type Quality Profiles

Different pack types have expected quality ranges:

| Pack Type | Typical Quality | Safety Weight Applies? |
|-----------|----------------|----------------------|
| PromptPack | 0.80–0.95 | Yes (AH Guard required) |
| HumanPack | 0.70–0.90 | No guard — safety via audience |
| OperatorPack | 0.75–0.90 | No guard — safety via status warning |
| SearchPack | 0.65–0.85 | No guard — safety not applicable |

---

## Rules

| Rule | Statement |
|------|-----------|
| CAE-076 | Context quality score must use the exact formula above — no approximations |
| CAE-077 | Minimum to emit is 0.70 — this is a hard check enforced by Validator V-15 |
| CAE-078 | Quality report must be included in every emitted pack |
| CAE-079 | For non-PromptPack, guard_completeness must be 1.0 (guard not required = no penalty) |
| CAE-080 | Improvement hints must be actionable — "quality is low" alone is forbidden |

---

## Cross-References

- Context validator → `16-CONTEXT-VALIDATOR` (V-15 check)
- AH Guard → `15-ANTI-HALLUCINATION-GUARD`
- Context metrics → `23-CONTEXT-METRICS` (KIM tracking)
- KIL quality model → Phase 3.0D.0.6 `14-KNOWLEDGE-QUALITY-MODEL`
