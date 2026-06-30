# KVF-DOC-011 — Context Compression Validation

**Phase:** 3.0D.1.5
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Context Compression Validation measures the correctness, consistency, and
information fidelity of the 5 compression levels (L1–L5) produced by the KOS
Runtime. It verifies that higher compression genuinely preserves more information
and that each level meets its token budget constraint.

---

## Compression Validation Dimensions

```
CV-DIM-1: Budget Conformance
  Does each level stay within its declared token budget?

CV-DIM-2: Superset Property
  Does each level contain all information from the level below?
  (L2 ⊇ L1, L3 ⊇ L2, L4 ⊇ L3, L5 ⊇ L4)

CV-DIM-3: Required Field Presence
  Are the required fields present at each level?

CV-DIM-4: Information Loss Quantification
  How much information is lost at each compression level?

CV-DIM-5: Compression Ratio Accuracy
  Does the compression ratio match the KIL compression spec?
```

---

## Budget Conformance Test

```
For each object in test corpus × each compression level:

  rendered = Runtime.render(object, level)
  tokens = count_tokens(rendered)

  CV-1.01 L1: tokens <= 15
  CV-1.02 L2: tokens <= 50
  CV-1.03 L3: tokens <= 200
  CV-1.04 L4: tokens <= 500
  CV-1.05 L5: tokens <= 2000 (machine) | uncapped (audience)

  Pass rate target: 100% — any budget violation is a failure
```

---

## Superset Property Test

```
For each pair of compression levels (L_n, L_{n+1}):
  rendered_n = Runtime.render(object, L_n)
  rendered_n1 = Runtime.render(object, L_{n+1})

  Extract key facts from rendered_n.
  Verify that each key fact is also present in rendered_n1.

  superset_rate = facts_preserved_in_higher_level / total_facts_in_lower_level
  Target: >= 0.95 (minor reformatting allowed, facts must be preserved)

Pairs tested: (L1,L2), (L2,L3), (L3,L4), (L4,L5)
```

---

## Required Field Presence Test

For each level, the following fields must be present in the rendered output:

```
L1 (nano, ≤ 15 tokens):
  - knowledge_id OR name

L2 (micro, ≤ 50 tokens):
  - knowledge_id
  - name
  - object_type
  - purpose (one sentence)

L3 (mini, ≤ 200 tokens):
  All L2 fields, plus:
  - at least 2 capabilities
  - at least 2 constraints
  - primary dependency names (if any)
  - state
  - quality_score

L4 (standard, ≤ 500 tokens):
  All L3 fields, plus:
  - all capabilities
  - all constraints
  - confidence.overall_score
  - at least 3 decisions

L5 machine (≤ 2000 tokens):
  All L4 fields, plus:
  - cortex.why
  - cortex.why_not
  - cortex.common_errors
  - thinking.reasoning_protocol
  - valid JSON structure
```

---

## Information Loss Quantification

```
information_loss(object, level):
  full_content = object.full_serialized_content
  rendered = Runtime.render(object, level)

  # Use embedding distance as information proxy
  full_embedding = embed(full_content)
  rendered_embedding = embed(rendered)
  similarity = cosine_similarity(full_embedding, rendered_embedding)

  information_preserved = similarity
  information_loss = 1 - similarity

Expected information loss by level:
  L1: 85–95% loss (expected: extreme compression)
  L2: 70–85% loss
  L3: 40–60% loss
  L4: 15–30% loss
  L5: < 10% loss
```

---

## Compression Ratio Accuracy

```
For each object:
  For each level:
    kos_ratio = count_tokens(L1) / count_tokens(level)
    e.g., L1(15 tokens) / L4(400 tokens) = 0.0375 → 26.7:1 compression vs L4

Expected compression ratios (L5 to Lx):
  L5 → L4: ~4:1
  L5 → L3: ~10:1
  L5 → L2: ~40:1
  L5 → L1: ~133:1

Validation: Runtime's compression ratios must be within ±20% of expected
```

---

## Compression Validation Result Schema

```yaml
compression_validation_result:
  objects_tested: integer

  budget_conformance:
    L1: {pass_rate: float, max_tokens_seen: integer}
    L2: {pass_rate: float, max_tokens_seen: integer}
    L3: {pass_rate: float, max_tokens_seen: integer}
    L4: {pass_rate: float, max_tokens_seen: integer}
    L5: {pass_rate: float, max_tokens_seen: integer}

  superset_property:
    L1_to_L2: float
    L2_to_L3: float
    L3_to_L4: float
    L4_to_L5: float

  required_fields:
    L1: {pass_rate: float}
    L2: {pass_rate: float}
    L3: {pass_rate: float}
    L4: {pass_rate: float}
    L5: {pass_rate: float}

  information_loss_mean:
    L1: float
    L2: float
    L3: float
    L4: float
    L5: float

  l5_json_valid_rate: float
```

---

## Rules

| Rule | Statement |
|------|-----------|
| KVF-051 | Budget conformance must be 100% — no overages allowed at any level |
| KVF-052 | Superset property at ≥ 0.95 is required for Gold certification |
| KVF-053 | L5 machine format must be tested for JSON validity — parse failure = critical |
| KVF-054 | Information loss measurement must use embeddings, not character count |
| KVF-055 | Required fields are the minimum — their absence at any level is a conformance failure |

---

## Cross-References

- Token efficiency → `06-TOKEN-EFFICIENCY`
- KIL compression spec → Phase 3.0D.0.6 `05-KNOWLEDGE-COMPRESSION`
- Compression selector spec → Phase 3.0D.1 `08-COMPRESSION-SELECTOR`
