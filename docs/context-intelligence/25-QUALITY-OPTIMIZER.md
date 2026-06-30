# UICE-DOC-025 — Quality Optimizer

**Phase:** 3.0D.2
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Quality Optimizer enforces UICE's quality floor (UI-06: quality_score ≥ 0.80)
and optimizes across 7 quality dimensions. It runs after compression and before
Context Verifier. If the assembled pack scores below the floor, the optimizer
applies targeted improvements — expanding compressed slots, promoting omitted
objects, or adjusting guard content.

---

## 7 Quality Dimensions

```
1. COVERAGE      — relevant objects included / total relevant objects
2. COMPLETENESS  — required fields present / total required fields for intent
3. PRECISION     — included fields that are relevant / total included fields
4. RECALL        — relevant fields included / total relevant fields available
5. SAFETY        — guard block present and complete for PromptPack
6. CONFIDENCE    — context_confidence score of the assembled pack
7. COMPRESSION   — token_count / baseline_token_count (lower is better — higher score)
```

---

## Quality Score Formula

```
quality_score =
  coverage    × 0.25 +
  completeness × 0.25 +
  precision   × 0.15 +
  recall      × 0.10 +
  safety      × 0.15 +
  confidence  × 0.05 +
  compression × 0.05

Weights sum: 1.00

Minimum to emit: quality_score ≥ 0.80 (UI-06)
```

---

## Dimension Measurement

```
COVERAGE:
  relevant_object_ids = ORACLE_RELEVANT(query, intent)  // from ranking engine
  included_ids        = {s.knowledge_id for s in pack.all_slots}
  coverage = len(included_ids ∩ relevant_object_ids) / max(len(relevant_object_ids), 1)

COMPLETENESS:
  required_fields = FIELD_MATRIX[(intent_type, query_type)]  // from FLA matrix
  present_fields  = FLATTEN_FIELD_PATHS(pack.primary.fields)
  completeness = len(present_fields ∩ required_fields) / max(len(required_fields), 1)

PRECISION:
  total_included = COUNT_ALL_FIELDS(pack)
  useful_included = total_included - dedup_log.tokens_removed
  precision = useful_included / max(total_included, 1)

RECALL:
  available_fields = GET_ALL_KIL_FIELDS(primary_kil_object)
  relevant_available = FILTER_RELEVANT(available_fields, intent, query)
  present            = FLATTEN_FIELD_PATHS(pack.primary.fields)
  recall = len(present ∩ relevant_available) / max(len(relevant_available), 1)

SAFETY:
  if pack_type != PROMPT_PACK: safety = 1.0
  elif guard is present and len(guard.critical_errors) + len(guard.relevant_errors) > 0:
    safety = 1.0
  elif guard is present but empty (fallback): safety = 0.85
  else: safety = 0.0   // guard missing — CI-03 violation

CONFIDENCE:
  confidence = pack.context_confidence  // propagated min(chain) per CAE CI-06

COMPRESSION:
  // Inverted — lower token count = higher compression score
  compression = 1.0 - (pack.total_token_count / pack.baseline_token_estimate)
```

---

## Quality Improvement Actions

```
If quality_score < 0.80, apply targeted improvements in priority order:

1. IF coverage < 0.80:
   Expand context slots: add highest-relevance OMIT objects up to budget.

2. IF completeness < 0.85:
   Expand primary: switch from SUMMARY to FLA or from FLA to STANDARD FLA.

3. IF safety < 0.90:
   Regenerate guard block: expand QAGB to include more errors.
   If still < 0.90: include all errors (static guard fallback).

4. IF precision < 0.85:
   Re-run deduplication: remove any fields added by improvement #1 or #2
   that are already covered by preamble.

5. IF quality_score still < 0.80 after all improvements:
   Log quality failure. Emit pack with quality warning in metadata.
   (Never block emission for quality — quality warning is always better
   than no response.)
```

---

## QualityReport Schema

```yaml
QualityReport:
  pack_id: string
  quality_score: float
  passed: bool                         # quality_score ≥ 0.80

  dimension_scores:
    coverage: float
    completeness: float
    precision: float
    recall: float
    safety: float
    confidence: float
    compression: float

  improvements_applied: list[string]   # descriptions of what was improved
  improvement_iterations: int          # how many improvement passes ran

  # If quality still fails
  quality_warning: string | null       # emitted if passed == False
```

---

## Rules

| Rule | Statement |
|------|-----------|
| UICE-121 | quality_score ≥ 0.80 is the emission floor (UI-06); packs below this floor are emitted with warning |
| UICE-122 | Safety dimension < 0.90 triggers guard regeneration before other improvements |
| UICE-123 | Quality improvement actions never violate budget constraints or CAE invariants |
| UICE-124 | Quality improvement loop runs maximum 3 iterations before emitting with warning |
| UICE-125 | QualityReport must be included in every ProfileReport — quality without evidence is INVALID |

---

## Cross-References

- CAE quality model → Phase 3.0D.1 `17-CONTEXT-QUALITY-MODEL`
- Context Verifier → `26-CONTEXT-VERIFIER`
- Adaptive Guard Generator → `09-ADAPTIVE-GUARD-GENERATOR`
- Context Compression Engine → `12-CONTEXT-COMPRESSION-ENGINE`
- Context Metrics → `20-CONTEXT-METRICS` (UICE-M-009 through UICE-M-016)
