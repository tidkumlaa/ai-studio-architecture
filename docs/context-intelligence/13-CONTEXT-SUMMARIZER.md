# UICE-DOC-013 — Context Summarizer

**Phase:** 3.0D.2
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Context Summarizer produces a **SummarySlice** — a compressed representation
of a KnowledgeObject that fits within a strict token budget. It is invoked when
a context slot has been assigned SUMMARY mode by the Compression Engine:
either because the slot's FLA output exceeds its budget, or because the object
is a low-relevance background slot needing minimal representation.

The Summarizer is NOT a generative model. It uses deterministic rule-based
field selection — choosing the single most informative field per KIL block
and truncating arrays to a configurable depth.

---

## SummarySlice Schema

```yaml
SummarySlice:
  knowledge_id: string
  name: string
  summary_mode: MICRO | MINI | STANDARD  # determines depth

  # Core identity (always included)
  identity: string          # dna.identity
  purpose: string           # dna.purpose (truncated to 80 chars max)
  object_type: string
  version: string

  # Top-1 field per intent axis (intent-aware selection)
  top_fields: dict[str, Any]   # key = field_path, value = truncated value

  # Relationship summary
  dependency_count: int        # len(reasoning.requires)
  top_dependency: string | null # first item in reasoning.requires

  # Quality signal
  ai_readiness_score: float | null
  context_confidence: float | null

  # Summarization metadata
  original_token_estimate: int
  summary_token_count: int
  compression_ratio: float   # original / summary
  fields_included: int
  fields_omitted: int
  summarizer_version: string
```

---

## Summary Modes

```
MICRO (≤ 30 tokens):
  Included: identity, purpose (≤40 chars), object_type, version
  Use case: very tight budget; background reference only

MINI (≤ 80 tokens):
  Included: MICRO fields + top_dependency + ai_readiness_score
  + top-1 field for the primary intent axis
  Use case: low-relevance related objects

STANDARD (≤ 150 tokens):
  Included: MINI fields + top-3 fields across intent axes
  + relationship summary
  Use case: context slots where FLA would exceed budget
```

---

## Summarization Algorithm

```
SUMMARIZE(kil_object, intent_profile, query_profile, token_budget) → SummarySlice:

  // Determine summary mode
  mode = MICRO    if token_budget <= 30
         else MINI     if token_budget <= 80
         else STANDARD

  intel = kil_object.intelligence

  // Always include: identity core
  result = {
    "identity":    TRUNCATE(intel.dna.identity, 60),
    "purpose":     TRUNCATE(intel.dna.purpose, 80),
    "object_type": kil_object.object_type,
    "version":     kil_object.version,
  }

  if mode == MICRO:
    return BUILD_SUMMARY_SLICE(kil_object, result, mode)

  // MINI: add dependency count + AIRS + top intent field
  result["dependency_count"] = len(intel.reasoning.get("requires", []))
  result["top_dependency"]   = FIRST(intel.reasoning.get("requires", []))
  result["ai_readiness"]     = intel.ai_readiness_score

  top_field = SELECT_TOP_INTENT_FIELD(intel, intent_profile.intent_type):
    match intent_profile.intent_type:
      FACTUAL_LOOKUP:     intel.dna.core_behavior[:100]
      IMPACT_ANALYSIS:    FIRST(intel.reasoning.impacts)
      CODE_GENERATION:    intel.explanation.ai_agent[:100]
      TRACEABILITY:       intel.semantic.implements
      DIAGNOSTIC:         FIRST(intel.cortex.common_errors)
      ARCHITECTURE_REVIEW: intel.cortex.why[:100]
      _:                  intel.dna.core_behavior[:80]

  if top_field:
    result[INTENT_FIELD_NAME(intent_profile.intent_type)] = top_field

  if mode == MINI:
    return BUILD_SUMMARY_SLICE(kil_object, result, mode)

  // STANDARD: add top-3 fields across axes
  intent_fields = SELECT_TOP_N_INTENT_FIELDS(intel, intent_profile, query_profile, n=3)
  result.update(intent_fields)

  return BUILD_SUMMARY_SLICE(kil_object, result, mode)


BUILD_SUMMARY_SLICE(kil_object, fields, mode) → SummarySlice:
  token_count = ESTIMATE_TOKENS(fields)
  return SummarySlice(
    knowledge_id          = kil_object.knowledge_id,
    name                  = kil_object.name,
    summary_mode          = mode,
    top_fields            = fields,
    original_token_estimate = ESTIMATE_TOKENS(kil_object),
    summary_token_count   = token_count,
    compression_ratio     = ESTIMATE_TOKENS(kil_object) / max(1, token_count),
    fields_included       = len(fields),
    fields_omitted        = COUNT_TOTAL_FIELDS(kil_object) - len(fields),
  )
```

---

## TRUNCATE Specification

```
TRUNCATE(text, max_chars) → string:
  if len(text) <= max_chars: return text
  // Truncate at word boundary before max_chars
  truncated = text[:max_chars].rsplit(" ", 1)[0]
  return truncated + "…"

Array truncation:
  list[:N]  — take first N items
  N = 1 for MICRO, N = 1 for MINI, N = 3 for STANDARD
```

---

## Token Targets by Mode

| Mode | Token Target | Typical Compression Ratio |
|------|-------------|--------------------------|
| MICRO | ≤ 30t | 15×–30× |
| MINI | ≤ 80t | 6×–15× |
| STANDARD | ≤ 150t | 3×–6× |

---

## Rules

| Rule | Statement |
|------|-----------|
| UICE-061 | Summarizer is deterministic: same inputs always produce identical SummarySlice |
| UICE-062 | MICRO mode includes only 4 identity fields; no other fields may be added at MICRO level |
| UICE-063 | Summary mode is STANDARD by default; MICRO and MINI require explicit budget below 30/80 tokens |
| UICE-064 | SummarySlice must record fields_omitted accurately — required for explainability (UI-01) |
| UICE-065 | Summarizer never calls external ML models — all field selection is rule-based and deterministic |

---

## Cross-References

- Context Compression Engine → `12-CONTEXT-COMPRESSION-ENGINE`
- Field-Level Assembly → `03-FIELD-LEVEL-ASSEMBLY`
- Token Budget Manager → `10-TOKEN-BUDGET-MANAGER`
- Context Verifier → `26-CONTEXT-VERIFIER`
