# UICE-DOC-003 — Field-Level Assembly

**Phase:** 3.0D.2
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Field-Level Assembly (FLA) replaces fixed L1–L5 compression tiers with
**per-field selection** driven by (IntentType × QueryType). Instead of
choosing "give me L4 Standard = 500 tokens of everything," FLA asks:
"for THIS intent and THIS query type, which individual KIL fields are needed?"

The result is surgical precision: a FACTUAL_LOOKUP + IDENTITY query receives
40–60 tokens containing exactly identity, purpose, object_type, namespace,
and version — nothing more.

Phase 3.0D.2 inherits the implementation from Phase 3.0D.1 (platform/context_engine/)
and defines the architecture contract it must satisfy.

---

## Architecture Contract

```
FLA satisfies:
  120-combination FIELD_MATRIX (12 intent types × 10 query types)
  L3 floor enforcement (CAE CI-01 preserved)
  Stateless pure function (CAE CI-05 preserved)
  L5 Full bypass for CONTEXT_PRODUCTION (CAE CI-01 + UICE UI-01)
```

---

## Field Selection Modes

```
L-QUERY (FLA mode):
  Selected fields from FIELD_MATRIX[(intent_type, query_type)]
  Applied when: all primary objects in non-CONTEXT_PRODUCTION flows
  Token range: 40–200 tokens per object

L5-FULL (bypass mode):
  All KIL fields included
  Applied when: intent_type == CONTEXT_PRODUCTION
  Token range: ≤ 2000 tokens per object

L3-FLOOR (enforcement):
  Applied when: L-QUERY selection < 50 tokens
  Merges L3_FLOOR_FIELDS into result
  Guarantees: primary object always carries semantic minimum
```

---

## FIELD_MATRIX Summary (120 Combinations)

Field paths use KIL `intelligence:` block notation. Arrays use slice notation.

**Selected high-value combinations:**

```
(FACTUAL_LOOKUP, IDENTITY):
  dna.identity, dna.purpose, object_type, namespace, version, status

(FACTUAL_LOOKUP, BEHAVIORAL):
  dna.core_behavior, self_describing.inputs, self_describing.outputs,
  semantic.capabilities[:3]

(REASONING_QUERY, CAUSAL):
  cortex.why, evolution.decisions[:3], reasoning.requires[:3],
  reasoning.impacts[:3], genome.risks[:2]

(IMPACT_ANALYSIS, IMPACT):
  reasoning.impacts[:8], self_describing.errors[:5], genome.risks[:5],
  dna.fundamental_constraints

(TRACEABILITY, RELATIONAL):
  reasoning.requires[:10], semantic.implements, semantic.provides[:5],
  genome.relationships[:8]

(CODE_GENERATION, BEHAVIORAL):
  dna.core_behavior, self_describing.inputs, self_describing.outputs,
  self_describing.errors[:3], explanation.ai_agent

(ARCHITECTURE_REVIEW, CAUSAL):
  cortex.why, evolution.decisions[:4], reasoning.tradeoffs[:3],
  dna.fundamental_constraints

(MIGRATION_GUIDE, TEMPORAL):
  evolution.history[:5], evolution.decisions[:4], evolution.migration,
  evolution.deprecation

(DIAGNOSTIC, OPERATIONAL):
  explanation.operator, self_describing.errors[:5],
  cortex.common_errors[:5], self_describing.performance

(SEARCH, IDENTITY):
  dna.identity, dna.purpose, object_type, namespace, version
```

Full 120-combination matrix: see platform/context_engine/field_matrix.py

---

## FIELD_EXTRACT Algorithm

```
FIELD_EXTRACT(kil_object, intent_type, query_type) → FieldLevelSlice:

  // Check bypass conditions
  if is_l5_full(intent_type, query_type):
    return FieldLevelSlice(fields=kil_object, mode=L5_FULL)

  // Lookup field paths
  field_paths = FIELD_MATRIX.get((intent_type, query_type), FALLBACK_L3_PATHS)

  // Extract fields
  extracted = {}
  for path in field_paths:
    value = NAVIGATE(kil_object.intelligence, path)
    if value is not None:
      SET_NESTED(extracted, path, value)

  // Apply L3 floor (CAE CI-01)
  if ESTIMATE_TOKENS(extracted) < 50:
    floor_fields = EXTRACT_L3_FLOOR_FIELDS(kil_object)
    extracted = MERGE(floor_fields, extracted)
    used_l3_floor = True

  return FieldLevelSlice(
    knowledge_id = kil_object.knowledge_id,
    fields       = extracted,
    token_count  = ESTIMATE_TOKENS(extracted),
    intent_type  = intent_type,
    query_type   = query_type,
    used_l3_floor = used_l3_floor,
    mode         = L_QUERY,
  )
```

---

## Token Reduction Targets

| Combination Category | Current (L4) | FLA | Reduction |
|---------------------|-------------|-----|-----------|
| Point lookup (IDENTITY) | 500t | 40–60t | 88–92% |
| Behavioral query | 500t | 60–100t | 80–88% |
| Causal / reasoning | 500t | 80–140t | 72–84% |
| Impact analysis | 500t | 100–180t | 64–80% |
| Traceability (deep) | 500t | 120–200t | 60–76% |
| CONTEXT_PRODUCTION | 2000t | 2000t | 0% (correct — full needed) |

---

## L3 Floor Fields (CI-01 minimum)

```
dna.identity
dna.purpose
dna.core_behavior
dna.fundamental_constraints
namespace
version
status
object_type
```

These 8 fields constitute the semantic minimum. FLA never produces a primary
object slice below this floor.

---

## Rules

| Rule | Statement |
|------|-----------|
| UICE-011 | FLA must consult FIELD_MATRIX for every non-CONTEXT_PRODUCTION primary assembly |
| UICE-012 | L3 floor is applied when estimated token count of extracted fields < 50 tokens |
| UICE-013 | Array slice limits in FIELD_MATRIX are hard maximums — never exceeded |
| UICE-014 | Unknown (intent_type, query_type) combinations fall back to FALLBACK_L3_PATHS |
| UICE-015 | FLA is stateless: same inputs must produce identical outputs every invocation |

---

## Cross-References

- Platform implementation → platform/context_engine/fla.py
- Field matrix → platform/context_engine/field_matrix.py
- Query Classifier → `02-QUERY-CLASSIFIER`
- Shared Context Optimizer → `05-SHARED-CONTEXT-OPTIMIZER`
- Context Compression Engine → `12-CONTEXT-COMPRESSION-ENGINE`
- CAE CI-01 → Phase 3.0D.1 `03-CAE-INVARIANTS`
