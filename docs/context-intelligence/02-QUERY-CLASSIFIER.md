# UICE-DOC-002 — Query Classifier

**Phase:** 3.0D.2
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Query Classifier takes the IntentProfile produced by the Intent Analyzer
and the raw query text, and produces a **QueryProfile** — a structured
representation of the SHAPE of the query: what type of answer is expected,
how many hops the graph traversal needs, and which entities are comparison targets.

Together, IntentProfile + QueryProfile determine exactly which KIL fields to
select (via the 120-combination FIELD_MATRIX) and how deep graph traversal goes.

---

## QueryProfile Schema

```yaml
QueryProfile:
  # Classification
  query_type: QueryType              # 10 types from CAE
  query_confidence: float            # 0.0–1.0

  # Tokenization
  query_tokens: list[str]            # stop-word-filtered terms
  query_length: int                  # token count

  # Entity information
  entity_mentions: list[str]         # knowledge_ids explicitly mentioned
  named_objects: list[str]           # human names resolved to knowledge_ids

  # Query structure
  multi_hop: bool                    # requires graph traversal depth > 1
  hop_depth: int                     # 1–4; capped at 4
  comparison_targets: list[str]      # knowledge_ids being compared (COMPARISON queries)
  temporal_scope: CURRENT | HISTORY | EVOLUTION | ALL
    # CURRENT:   present state only
    # HISTORY:   past states / evolution
    # EVOLUTION: trajectory and decisions
    # ALL:       no temporal restriction

  # Derived flags
  expects_list: bool                 # answer is a list (e.g., "what are all X")
  expects_narrative: bool            # answer is explanatory prose
  expects_code: bool                 # answer includes code
  expects_table: bool                # answer includes structured comparison

  # Metadata
  classifier_version: string
  classification_ms: float
```

---

## Query Types (10)

Inherited from CAE without modification:

```
IDENTITY     — "what is X" / "describe X" / "what type is X"
BEHAVIORAL   — "how does X work" / "what does X do" / "what is X's behavior"
RELATIONAL   — "what depends on X" / "what does X need" / "X's dependencies"
CAUSAL       — "why does X exist" / "why was X built this way" / "X's rationale"
COMPARATIVE  — "X vs Y" / "compare X and Y" / "alternatives to X"
IMPACT       — "what if X fails" / "what breaks if X changes" / "X's failure modes"
STRUCTURAL   — "how is X organized" / "X's structure" / "X's components"
TEMPORAL     — "how did X evolve" / "X's history" / "when was X introduced"
OPERATIONAL  — "how to run X" / "how to deploy X" / "X's configuration"
MULTI_HOP    — "X's dependency Y's impact on Z" / chains of relationships
```

---

## Classification Algorithm

```
CLASSIFY_QUERY(query_text, intent_profile) → QueryProfile:

  tokens = TOKENIZE_STOP_WORDS(query_text)

  // Stage 1: Query type signal detection
  signals = {
    IDENTITY:    keywords_match(tokens, ["what is", "describe", "define", "tell me about"]),
    BEHAVIORAL:  keywords_match(tokens, ["how does", "how works", "how do", "behavior"]),
    RELATIONAL:  keywords_match(tokens, ["depends", "dependency", "requires", "uses", "needs"]),
    CAUSAL:      keywords_match(tokens, ["why", "reason", "rationale", "purpose", "exist"]),
    COMPARATIVE: keywords_match(tokens, ["vs", "versus", "compare", "difference", "better"]),
    IMPACT:      keywords_match(tokens, ["fail", "impact", "breaks", "affects", "if X"]),
    STRUCTURAL:  keywords_match(tokens, ["structure", "organized", "components", "parts"]),
    TEMPORAL:    keywords_match(tokens, ["history", "evolved", "changed", "when", "version"]),
    OPERATIONAL: keywords_match(tokens, ["run", "deploy", "configure", "monitor", "install"]),
    MULTI_HOP:   count_entity_mentions(tokens) >= 3 or intent_profile.requires_traceability,
  }

  // Stage 2: Intent-to-query-type prior
  // Intent strongly suggests query type in many cases
  prior_map = {
    DIAGNOSTIC:     BEHAVIORAL | OPERATIONAL,
    MIGRATION_GUIDE: TEMPORAL,
    ARCHITECTURE_REVIEW: CAUSAL | STRUCTURAL,
    CODE_GENERATION: BEHAVIORAL | OPERATIONAL,
    TRACEABILITY:   RELATIONAL | MULTI_HOP,
    IMPACT_ANALYSIS: IMPACT,
    COMPARISON:     COMPARATIVE,
  }
  if intent_profile.intent_type in prior_map:
    boost(signals, prior_map[intent_profile.intent_type], 0.30)

  // Stage 3: Select query type
  query_type = ARGMAX(signals)
  query_confidence = signals[query_type]

  // Stage 4: Hop depth calculation
  hop_depth = 1  // default: single object
  if MULTI_HOP selected or intent_profile.requires_traceability:
    hop_depth = min(4, count_relationship_keywords(tokens) + 1)
  if intent_type == IMPACT_ANALYSIS:
    hop_depth = min(3, hop_depth + 1)  // impact propagates

  // Stage 5: Comparison targets
  comparison_targets = []
  if query_type == COMPARATIVE:
    comparison_targets = EXTRACT_ENTITY_PAIRS(tokens)

  // Stage 6: Temporal scope
  temporal_scope = CURRENT
  if TEMPORAL selected or "history" in tokens or "evolved" in tokens:
    temporal_scope = HISTORY if "history" in tokens else EVOLUTION
  if "all" in tokens and TEMPORAL selected:
    temporal_scope = ALL

  // Stage 7: Answer format flags
  expects_list     = query starts with "list" or "all" or "what are" (plural)
  expects_code     = intent_type == CODE_GENERATION or "code" in tokens
  expects_table    = query_type == COMPARATIVE
  expects_narrative = query_type in {CAUSAL, REASONING_QUERY, TEMPORAL}

  return QueryProfile(
    query_type         = query_type,
    query_confidence   = query_confidence,
    query_tokens       = tokens,
    query_length       = len(tokens),
    entity_mentions    = intent_profile.detected_entities,
    named_objects      = RESOLVE_NAMES(intent_profile.detected_entities),
    multi_hop          = hop_depth > 1,
    hop_depth          = hop_depth,
    comparison_targets = comparison_targets,
    temporal_scope     = temporal_scope,
    expects_list       = expects_list,
    expects_narrative  = expects_narrative,
    expects_code       = expects_code,
    expects_table      = expects_table,
  )
```

---

## (IntentType × QueryType) Matrix Key

The combination of IntentProfile.intent_type and QueryProfile.query_type
is the lookup key for the 120-combination FIELD_MATRIX in module 03.

Common high-value combinations:

| Intent | Query | Primary Fields | Typical Tokens |
|--------|-------|---------------|----------------|
| FACTUAL_LOOKUP | IDENTITY | dna.identity, purpose, object_type, version | 40–60 |
| CODE_GENERATION | BEHAVIORAL | core_behavior, inputs, outputs, errors[:3], ai_agent | 80–120 |
| IMPACT_ANALYSIS | IMPACT | impacts[:8], errors[:5], risks[:5], constraints | 120–180 |
| REASONING_QUERY | CAUSAL | cortex.why, decisions[:3], requires[:3], risks[:2] | 100–140 |
| TRACEABILITY | RELATIONAL | requires[:10], implements, provides[:5], relationships[:8] | 140–200 |
| CONTEXT_PRODUCTION | ANY | full L5 | ≤ 2000 |

---

## Rules

| Rule | Statement |
|------|-----------|
| UICE-006 | QueryProfile must always be produced even when intent_confidence < 0.60 |
| UICE-007 | hop_depth must be capped at 4; deeper traversal is never permitted |
| UICE-008 | comparison_targets are only populated for COMPARATIVE query_type |
| UICE-009 | temporal_scope = CURRENT by default; HISTORY and EVOLUTION require explicit signal |
| UICE-010 | (intent_type, query_type) must form a valid FIELD_MATRIX key or fall back to L3 floor |

---

## Cross-References

- Intent Analyzer → `01-INTENT-ANALYZER`
- Field-Level Assembly → `03-FIELD-LEVEL-ASSEMBLY`
- Context Graph Traversal → `08-CONTEXT-GRAPH-TRAVERSAL`
- Dynamic Context Planner → `04-DYNAMIC-CONTEXT-PLANNER`
