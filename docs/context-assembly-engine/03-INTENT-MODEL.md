# CAE-DOC-003 — Intent Model

**Phase:** 3.0D.1
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Intent Parser classifies every incoming query into one of 12 intent types
and extracts structured entities. Intent classification drives all downstream
decisions: which objects to select, what compression to use, which pack type
to return.

---

## 12 Intent Types

| ID | Intent | Trigger Patterns | Primary Output |
|----|--------|-----------------|---------------|
| INT-01 | FACTUAL_LOOKUP | "what is X", "define X", "what does X do" | Single object, L4 |
| INT-02 | RELATIONSHIP_QUERY | "what depends on X", "what does X use", "implements X" | Object + graph |
| INT-03 | REASONING_QUERY | "why does X exist", "why was X designed this way" | Cortex + tradeoffs |
| INT-04 | IMPACT_ANALYSIS | "what breaks if X changes", "impact of removing X" | Risk + reasoning.impacts |
| INT-05 | COMPARISON | "X vs Y", "difference between X and Y", "is X the same as Y" | Semantic diff |
| INT-06 | SEARCH | "find objects that do Z", "which module handles Z" | Ranked SearchPack |
| INT-07 | EXPLANATION | "explain X to [audience]", "how does X work" | Audience-targeted HumanPack |
| INT-08 | CONTEXT_PRODUCTION | "produce context for Q", "build PromptPack for Q" | PromptPack |
| INT-09 | TRACEABILITY | "trace X through system", "what satisfies requirement X" | Traceability chain |
| INT-10 | DEPENDENCY_RESOLUTION | "what does X need", "dependency tree of X" | Dependency graph |
| INT-11 | EVOLUTION_QUERY | "how has X changed", "history of X", "what replaced X" | Evolution block |
| INT-12 | VALIDATION | "is X correct", "does X satisfy Y", "check X" | Quality + executable rules |

---

## Intent Classification Schema

```yaml
intent_record:
  query_id: string
  raw_query: string

  classification:
    primary_intent: string             # INT-01 through INT-12
    secondary_intent: string | null    # if query spans two intents
    confidence: float                  # 0.0–1.0
    classification_method: RULE | KEYWORD | SEMANTIC | HYBRID

  entities:
    primary_entity:
      raw_text: string                 # as mentioned in query
      knowledge_id: string | null      # if resolved
      entity_type: string | null       # MODULE | SERVICE | ALGORITHM | etc.
      resolution_method: EXACT | SEMANTIC | ALIAS | INFERRED

    secondary_entities:
      - raw_text: string
        knowledge_id: string | null
        role: COMPARISON_TARGET | CONTEXT | FILTER | REFERENCE

    audience:
      stated: string | null            # if query mentions audience ("explain to developer")
      inferred: string                 # inferred from consumer_type

  filters:
    namespace: string | null
    object_type: string | null
    state: string | null
    version: string | null

  depth_hint: SURFACE | STANDARD | DEEP | EXPERT
```

---

## Classification Rules

### Keyword Rules (fast path, O(1))

```
IF query contains knowledge_id pattern (KNW-*):
  → entity resolved directly; skip semantic search
  → intent = FACTUAL_LOOKUP unless other signals present

IF query contains "vs" or "difference between" or "same as":
  → intent = COMPARISON
  → extract left and right entities

IF query contains "breaks if" or "impact" or "remove":
  → intent = IMPACT_ANALYSIS

IF query contains "why" + entity reference:
  → intent = REASONING_QUERY

IF query contains "depends on" or "dependency" or "tree":
  → intent = DEPENDENCY_RESOLUTION

IF query contains "trace" or "satisfies" or "implements" or "tested by":
  → intent = TRACEABILITY

IF query contains "history" or "changed" or "replaced" or "evolution":
  → intent = EVOLUTION_QUERY

IF query contains "explain to" or specific audience word:
  → intent = EXPLANATION
  → extract audience from "explain to {audience}"

IF query contains "context for" or "PromptPack" or "build context":
  → intent = CONTEXT_PRODUCTION
```

### Semantic Rules (fallback, uses embeddings)

```
IF keyword rules produce confidence < 0.70:
  → compute semantic similarity to 12 intent prototype embeddings
  → select intent with highest similarity
  → if similarity < 0.60: intent = FACTUAL_LOOKUP (safe default)
```

---

## Intent → Pipeline Configuration

| Intent | Object Selector Mode | Min Compression | Guard Level | Pack Type |
|--------|---------------------|-----------------|-------------|-----------|
| INT-01 FACTUAL | PRIMARY_ONLY | L4 | STANDARD | PROMPT/HUMAN |
| INT-02 RELATIONSHIP | PRIMARY + GRAPH | L3 | STANDARD | PROMPT/HUMAN |
| INT-03 REASONING | PRIMARY + CORTEX | L4 + CORTEX | HIGH | PROMPT/HUMAN |
| INT-04 IMPACT | PRIMARY + IMPACT_GRAPH | L3 + RISK | HIGH | PROMPT |
| INT-05 COMPARISON | DUAL_PRIMARY + DIFF | L3 each | HIGH | PROMPT/HUMAN |
| INT-06 SEARCH | MULTI_PRIMARY | L2 each | MINIMAL | SEARCH |
| INT-07 EXPLANATION | PRIMARY + AUDIENCE | L5 (audience block) | STANDARD | HUMAN |
| INT-08 CONTEXT | PRIMARY + FULL_CONTEXT | L4 primary + L3 rest | HIGH | PROMPT |
| INT-09 TRACEABILITY | CHAIN | L3 each | STANDARD | PROMPT/HUMAN |
| INT-10 DEPENDENCY | DEPENDENCY_TREE | L2 each | MINIMAL | PROMPT/HUMAN |
| INT-11 EVOLUTION | PRIMARY + EVOLUTION | L3 + EVOLUTION | MINIMAL | HUMAN |
| INT-12 VALIDATION | PRIMARY + RULES | L4 + EXECUTABLE | HIGH | PROMPT |

---

## Entity Resolution

```
RESOLVE(raw_text):

  Step 1: Exact ID match
    IF raw_text matches /^KNW-[A-Z]+-[A-Z]+-\d+$/:
      → lookup IDX-001 (knowledge_id index)
      → if found: return entity with method=EXACT

  Step 2: Canonical name match
    IF raw_text matches /^[a-z]+\.[a-z]+\.[a-z-]+$/:
      → lookup IDX-002 (canonical_name index)
      → if found: return entity with method=EXACT

  Step 3: Alias/keyword match
    → lookup IDX-009 (capability index) for raw_text tokens
    → lookup search.keyword.aliases in matched objects
    → if match: return entity with method=ALIAS

  Step 4: Semantic match
    → embed raw_text
    → search vector store (IDX-010)
    → return top-1 if similarity ≥ 0.75 with method=SEMANTIC

  Step 5: Inferred
    → use intent context to narrow type
    → return best candidate with method=INFERRED, confidence reduced

  Failure:
    → return null entity
    → escalate to AssemblyError OBJECT_NOT_FOUND if primary entity
```

---

## Rules

| Rule | Statement |
|------|-----------|
| CAE-007 | Classification confidence < 0.60 must fall back to FACTUAL_LOOKUP |
| CAE-008 | Primary entity resolution must complete in < 20ms |
| CAE-009 | INT-05 COMPARISON always requires exactly two entities — error if only one |
| CAE-010 | INT-04 IMPACT_ANALYSIS requires risk model present on primary object |
| CAE-011 | Audience extraction for INT-07 must check both stated and inferred audience |

---

## Cross-References

- Query types → `04-QUERY-TYPES`
- Object selector → `05-OBJECT-SELECTOR`
- Entity resolution uses → KIL Canonical Index docs 28–29
