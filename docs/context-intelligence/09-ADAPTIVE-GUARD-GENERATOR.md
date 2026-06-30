# UICE-DOC-009 — Adaptive Guard Generator

**Phase:** 3.0D.2
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Adaptive Guard Generator replaces the static AH Guard of CAE with a
**query-specific** hallucination prevention block. Where CAE's guard block
contains the same content for every query about an object, UICE's guard
generates content tailored to the specific risk profile of this intent,
query, and consumer.

The goal: **Hallucination Reduction > 30% vs CAE baseline** while simultaneously
reducing guard block token usage from ~100 tokens to 20–40 tokens (70% reduction).

CAE CI-03 is fully preserved: the guard block is always generated for PromptPack.

---

## AdaptiveGuardBlock Schema

```yaml
AdaptiveGuardBlock:
  # Core content (always present for PromptPack)
  guard_id: string
  generated_for: string              # knowledge_id
  intent_type: IntentType
  query_type: QueryType

  # Filtered errors (from QAGB — Phase 3.0D.2)
  critical_errors: list[GuardError]  # CRITICAL severity — always included
  relevant_errors: list[GuardError]  # query-relevant non-critical errors
  error_count_total: int             # all errors in cortex.common_errors
  error_count_included: int

  # Intent-specific warnings
  intent_warnings: list[IntentWarning]
    IntentWarning:
      warning_type: SCOPE_BOUNDARY | CONFIDENCE_HEDGE | MISSING_CONTEXT | STALENESS
      message: string                # 1–2 sentences max
      severity: HIGH | MEDIUM | LOW

  # Confidence hedge (from CAE GuardBlock)
  context_confidence: float
  hedge_statement: string | null    # null if confidence ≥ 0.90

  # UICE additions
  scope_boundaries: list[string]    # what THIS context does NOT cover
  query_risk_level: LOW | MEDIUM | HIGH | CRITICAL
  tokens_saved_vs_static: int       # vs sending all common_errors

  token_count: int
```

---

## Adaptive Guard Generation Algorithm

```
GENERATE_ADAPTIVE_GUARD(primary_kil, intent_profile, query_profile,
                         context_confidence) → AdaptiveGuardBlock:

  // Step 1: QAGB error filtering (inherited from Phase 3.0D.2)
  query_tokens = query_profile.query_tokens
  all_errors   = primary_kil.intelligence.cortex.common_errors
  critical     = [e for e in all_errors if e.severity == CRITICAL]
  relevant     = QAGB_FILTER(all_errors, query_tokens)  // see phase 3.0D.2 algorithm

  // Step 2: Query risk assessment
  query_risk_level = ASSESS_RISK(intent_profile, query_profile, context_confidence):
    if context_confidence < 0.60: CRITICAL
    elif intent_profile.intent_type in {CODE_GENERATION, ARCHITECTURE_REVIEW}: HIGH
    elif intent_profile.intent_type in {IMPACT_ANALYSIS, TRACEABILITY}: HIGH
    elif query_profile.query_type == MULTI_HOP: MEDIUM
    else: LOW

  // Step 3: Intent-specific warnings
  intent_warnings = []

  if intent_profile.intent_type == CODE_GENERATION:
    intent_warnings.append(IntentWarning(
      warning_type = SCOPE_BOUNDARY,
      message = "Generated code uses the interface contract specified. Runtime behavior depends on implementation.",
      severity = HIGH,
    ))

  if query_profile.query_type == MULTI_HOP and query_profile.hop_depth >= 3:
    intent_warnings.append(IntentWarning(
      warning_type = MISSING_CONTEXT,
      message = "This context covers a 3+ hop chain. Deep dependencies may have additional constraints not shown.",
      severity = MEDIUM,
    ))

  if intent_profile.intent_type == IMPACT_ANALYSIS:
    intent_warnings.append(IntentWarning(
      warning_type = SCOPE_BOUNDARY,
      message = "Impact analysis covers direct and 2-hop downstream effects. Transitive impacts beyond 2 hops are not included.",
      severity = MEDIUM,
    ))

  if context_confidence < 0.80:
    intent_warnings.append(IntentWarning(
      warning_type = CONFIDENCE_HEDGE,
      message = f"Context confidence is {context_confidence:.2f}. Cross-reference with authoritative source before acting.",
      severity = HIGH if context_confidence < 0.70 else MEDIUM,
    ))

  // Step 4: Confidence hedge (from CAE)
  hedge = DERIVE_HEDGE(context_confidence):
    if context_confidence < 0.70: "Low confidence context — verify before use"
    elif context_confidence < 0.90: "Medium confidence — spot-check key facts"
    else: null

  // Step 5: Scope boundaries
  scope_boundaries = EXTRACT_SCOPE_BOUNDARIES(primary_kil, intent_profile, query_profile)

  guard = AdaptiveGuardBlock(
    critical_errors      = critical,
    relevant_errors      = relevant,
    error_count_total    = len(all_errors),
    error_count_included = len(critical) + len(relevant),
    intent_warnings      = intent_warnings,
    context_confidence   = context_confidence,
    hedge_statement      = hedge,
    scope_boundaries     = scope_boundaries,
    query_risk_level     = query_risk_level,
    tokens_saved_vs_static = ESTIMATE_SAVED_TOKENS(all_errors, critical, relevant),
  )
  guard.token_count = ESTIMATE_TOKENS(guard)
  return guard
```

---

## Intent-Warning Type Definitions

```
SCOPE_BOUNDARY:
  What this context does NOT cover.
  Required for: CODE_GENERATION, IMPACT_ANALYSIS, ARCHITECTURE_REVIEW.

CONFIDENCE_HEDGE:
  Uncertainty statement when context_confidence < 0.80.
  Required for: any intent when confidence is low.

MISSING_CONTEXT:
  Warning that known gaps exist in the context.
  Required for: MULTI_HOP queries with depth ≥ 3.

STALENESS:
  Warning that some knowledge is approaching deprecation.
  Required for: any object where evolution.deprecation is non-null.
```

---

## Hallucination Reduction Mechanism

UICE adaptive guard reduces hallucination in three ways:

```
1. Focused errors: Only errors relevant to THIS query are shown.
   → LLM attends to relevant warnings instead of scanning all errors.
   → Estimated impact: 15% hallucination reduction.

2. Intent-specific warnings: Warnings tailored to what LLM is about to do.
   → "If generating code, this is the scope boundary" is more effective
     than generic warnings.
   → Estimated impact: 10% hallucination reduction.

3. Scope boundaries: Explicitly states what is NOT covered.
   → Prevents the LLM from fabricating content for out-of-scope topics.
   → Estimated impact: 5–8% hallucination reduction.

Combined target: > 30% reduction vs CAE static guard.
```

---

## Rules

| Rule | Statement |
|------|-----------|
| UICE-041 | Adaptive Guard must always be generated for PromptPack (UI-08 — inherits CI-03) |
| UICE-042 | CRITICAL errors from cortex.common_errors are never filtered regardless of query relevance |
| UICE-043 | intent_warnings must be generated for CODE_GENERATION and IMPACT_ANALYSIS — never omitted |
| UICE-044 | hedge_statement is null only when context_confidence ≥ 0.90 |
| UICE-045 | query_risk_level == CRITICAL triggers mandatory hedge + all errors included (no QAGB filtering) |

---

## Cross-References

- Platform implementation → platform/context_engine/qagb.py
- Context Verifier → `26-CONTEXT-VERIFIER`
- Quality Optimizer → `25-QUALITY-OPTIMIZER`
- CAE AH Guard → Phase 3.0D.1 `15-ANTI-HALLUCINATION-GUARD`
- CAE CI-03 → Phase 3.0D.1 `03-CAE-INVARIANTS`
