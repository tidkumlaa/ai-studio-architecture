# UICE-DOC-001 — Intent Analyzer

**Phase:** 3.0D.2
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Intent Analyzer is the first stage of the UICE pipeline. It takes a raw
query string and consumer context, and produces an **IntentProfile** — a
structured representation of WHY the query was asked and WHAT the consumer
will do with the result.

Without intent understanding, every query receives the same context shape.
With intent understanding, a "what is X" query gets identity fields only,
while a "how to migrate from X" query gets evolution and migration fields.

---

## IntentProfile Schema

```yaml
IntentProfile:
  # Primary intent classification
  intent_type: IntentType             # 12 types from CAE (FACTUAL_LOOKUP, etc.)
  intent_confidence: float            # 0.0–1.0; below 0.60 → FALLBACK_L3
  sub_intents: list[IntentType]       # secondary intents (empty if single-intent)

  # Consumer context
  consumer_type: AI_AGENT | HUMAN_DEV | OPERATOR | SEARCH_UI
  urgency: EXPLORATION | TASK | PRODUCTION
    # EXPLORATION: developer browsing / learning
    # TASK:        completing a specific work item
    # PRODUCTION:  live system, latency-critical

  # Entity extraction
  detected_entities: list[str]        # knowledge_id or name mentions in query
  entity_count: int                   # shorthand

  # Derived signals
  requires_traceability: bool         # true if query implies "why" / "trace"
  requires_examples: bool             # true if query implies "show me" / "example"
  requires_comparison: bool           # true if query implies "vs" / "compare"
  multi_intent: bool                  # true if sub_intents is non-empty

  # Metadata
  analyzer_version: string
  detection_ms: float
```

---

## Intent Types (12)

Inherited from CAE without modification (CAE CI-10 preserved):

```
FACTUAL_LOOKUP       — point-fact retrieval ("what is X", "what version is X")
CAPABILITY_DISCOVERY — capability exploration ("what can X do", "what does X provide")
REASONING_QUERY      — why/how reasoning ("why does X exist", "how does X work internally")
IMPACT_ANALYSIS      — change analysis ("what breaks if X changes", "what depends on X")
COMPARISON           — comparison ("X vs Y", "alternatives to X")
SEARCH               — broad search ("find services that do A")
DIAGNOSTIC           — error diagnosis ("why is X failing", "what's wrong with X")
CONTEXT_PRODUCTION   — full context for AI agent ("give me everything about X")
TRACEABILITY         — dependency chain ("trace X's dependencies", "what requires X")
MIGRATION_GUIDE      — migration ("how to migrate from X to Y", "X is deprecated")
ARCHITECTURE_REVIEW  — design audit ("review X's design", "ADR context for X")
CODE_GENERATION      — code synthesis ("write code using X", "implement X interface")
```

---

## Detection Algorithm

```
ANALYZE_INTENT(query_text, consumer_context) → IntentProfile:

  tokens = TOKENIZE(query_text)
  
  // Stage 1: Entity extraction
  entities = EXTRACT_ENTITIES(tokens)  // match against known knowledge_ids + names
  
  // Stage 2: Signal detection
  signals = DETECT_SIGNALS(tokens):
    has_version_word     = any t in tokens: t in {"version", "v", "latest", "release"}
    has_why_word         = any t in tokens: t in {"why", "reason", "rationale", "purpose"}
    has_how_word         = any t in tokens: t in {"how", "works", "mechanism", "process"}
    has_compare_word     = any t in tokens: t in {"vs", "versus", "compare", "alternative"}
    has_fail_word        = any t in tokens: t in {"fail", "error", "broken", "wrong", "debug"}
    has_migrate_word     = any t in tokens: t in {"migrate", "migration", "deprecated", "replace"}
    has_trace_word       = any t in tokens: t in {"trace", "depends", "dependency", "requires"}
    has_generate_word    = any t in tokens: t in {"generate", "write", "implement", "code"}
    has_impact_word      = any t in tokens: t in {"impact", "break", "affect", "change"}
    has_what_is          = query starts with "what is" or "what are"
    has_search_pattern   = len(entities) == 0 and query has adjectives
  
  // Stage 3: Intent scoring (each signal votes for an intent)
  scores = DEFAULT_MAP:
    FACTUAL_LOOKUP:       0.3 * has_what_is + 0.4 * (has_version_word or len(entities) > 0)
    REASONING_QUERY:      0.5 * has_why_word + 0.3 * has_how_word
    IMPACT_ANALYSIS:      0.7 * has_impact_word
    COMPARISON:           0.8 * has_compare_word
    DIAGNOSTIC:           0.7 * has_fail_word
    MIGRATION_GUIDE:      0.8 * has_migrate_word
    TRACEABILITY:         0.7 * has_trace_word
    CODE_GENERATION:      0.7 * has_generate_word
    SEARCH:               0.6 * has_search_pattern
    CAPABILITY_DISCOVERY: 0.4 * (not has_why_word and not has_fail_word)
    ARCHITECTURE_REVIEW:  0.3 * (has_why_word and has_how_word)
    CONTEXT_PRODUCTION:   consumer_type == AI_AGENT and query has "full" or "everything"

  // Stage 4: Select primary and sub-intents
  sorted_scores = SORT_DESC(scores)
  primary = sorted_scores[0]
  sub_intents = [i for i, s in sorted_scores[1:] if s > 0.30]

  // Stage 5: Consumer type derivation
  consumer_type = consumer_context.consumer_type  // from request metadata
  urgency = DERIVE_URGENCY(consumer_context)

  return IntentProfile(
    intent_type      = primary.intent_type,
    intent_confidence = primary.score,
    sub_intents      = sub_intents[:3],
    consumer_type    = consumer_type,
    urgency          = urgency,
    detected_entities = entities,
    entity_count     = len(entities),
    requires_traceability = has_trace_word or intent_type == TRACEABILITY,
    requires_examples     = "example" in tokens or "show" in tokens,
    requires_comparison   = has_compare_word,
    multi_intent          = len(sub_intents) > 0,
  )
```

---

## Confidence Thresholds

```
intent_confidence ≥ 0.80 → high confidence, use primary intent
intent_confidence ≥ 0.60 → medium confidence, include top sub_intent in planning
intent_confidence < 0.60 → low confidence → FALLBACK: use FACTUAL_LOOKUP + L3 floor
```

---

## Consumer Type Derivation

```
consumer_type:
  AI_AGENT    — request has agent_id header; or intent == CONTEXT_PRODUCTION
  HUMAN_DEV   — request from IDE / developer portal
  OPERATOR    — request from monitoring / SRE tooling
  SEARCH_UI   — request from search interface

urgency:
  PRODUCTION  — live system flag set; or consumer_type == AI_AGENT running live task
  TASK        — single-shot task completion
  EXPLORATION — no task flag; interactive session
```

---

## Rules

| Rule | Statement |
|------|-----------|
| UICE-001 | Intent Analyzer must produce an IntentProfile for every query — it may never fail silently |
| UICE-002 | intent_confidence < 0.60 triggers FALLBACK_L3 mode; downstream modules must respect this |
| UICE-003 | Detected entities must reference known knowledge_ids; unrecognized strings are dropped |
| UICE-004 | consumer_type must be set from request metadata, never inferred from query text alone |
| UICE-005 | sub_intents must be capped at 3; the primary intent always takes priority in planning |

---

## Cross-References

- Intent types → Phase 3.0D.1 `02-INTENT-PARSER`
- Query classification → `02-QUERY-CLASSIFIER`
- Dynamic planner → `04-DYNAMIC-CONTEXT-PLANNER`
- Field-level assembly → `03-FIELD-LEVEL-ASSEMBLY`
- CAE preserved invariants → Phase 3.0D.1 `03-CAE-INVARIANTS`
