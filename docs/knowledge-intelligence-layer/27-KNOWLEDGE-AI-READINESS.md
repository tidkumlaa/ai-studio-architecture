# KNW-KIL-DOC-027 — Knowledge AI Readiness

**Phase:** 3.0D.0.6
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The AI Readiness Score (AIRS) is a single composite metric (0.0–1.0) that
answers: *How ready is this Knowledge Object for AI agent consumption?*

An object with AIRS = 1.0 can be consumed by any AI agent to answer any
question about it correctly and without hallucination. An object with AIRS = 0
cannot reliably be used by AI without significant human oversight.

---

## AIRS Formula

```
AIRS = (
  W1 × self_describing_score   +   # 0.15
  W2 × ai_context_score        +   # 0.20
  W3 × compression_score       +   # 0.10
  W4 × confidence_score        +   # 0.15
  W5 × cortex_score            +   # 0.15
  W6 × reasoning_score         +   # 0.10
  W7 × genome_score            +   # 0.05
  W8 × dna_score               +   # 0.05
  W9 × learning_score          +   # 0.03
  W10 × search_score               # 0.02
) × hallucination_penalty_multiplier
```

Weights sum to 1.00.

---

## AIRS Dimensions

| Dimension | ID | Weight | What It Measures |
|-----------|----|----|-----------------|
| Self-Describing | W1 | 0.15 | purpose/inputs/outputs/constraints populated |
| AI Context | W2 | 0.20 | short/medium/full summaries + typical_mistakes |
| Compression | W3 | 0.10 | all 5 levels (L1–L5) populated and valid |
| Confidence | W4 | 0.15 | confidence score and hedging rules defined |
| Cortex | W5 | 0.15 | why/why_not/common_errors/reasoning_chain |
| Reasoning | W6 | 0.10 | requires/provides/conflicts/enables populated |
| Genome | W7 | 0.05 | all 8 genome fields complete |
| DNA | W8 | 0.05 | DNA sealed, hash valid |
| Learning | W9 | 0.03 | difficulty + prerequisites + learning path |
| Search | W10 | 0.02 | vector embedding + search optimization |

---

## Dimension Scoring

### W1: Self-Describing Score

```
SD_score = (
  purpose defined: +0.20
  responsibilities ≥ 1: +0.10
  inputs defined: +0.15
  outputs defined: +0.15
  constraints defined: +0.15
  assumptions defined: +0.10
  guarantees defined: +0.05
  risks defined: +0.05
  examples ≥ 1: +0.03
  anti_patterns ≥ 1: +0.02
) / 1.00
```

### W2: AI Context Score

```
AC_score = (
  short_summary present and ≤ 50t: +0.20
  medium_summary present and ≤ 200t: +0.20
  full_summary present: +0.15
  ai_hints ≥ 3: +0.15
  common_questions ≥ 3: +0.15
  typical_mistakes ≥ 1: +0.15
) / 1.00
```

### W3: Compression Score

```
COMP_score = (
  nano present and ≤ 15t: +0.20
  micro present and ≤ 50t: +0.20
  mini present and ≤ 200t: +0.20
  standard present: +0.20
  machine valid JSON: +0.20
) / 1.00
```

### W4: Confidence Score

```
CONF_score = intelligence.confidence.overall_score
```

### W5: Cortex Score

```
CX_score = (
  why.statement present: +0.20
  why.would_happen_without present: +0.15
  why_not ≥ 1 entry: +0.15
  common_questions ≥ 3: +0.20
  common_errors ≥ 1: +0.15
  typical_reasoning ≥ 3 steps: +0.15
) / 1.00
```

### W6: Reasoning Score

```
RSN_score = (
  requires ≥ 1: +0.20
  provides ≥ 1: +0.20
  depends ≥ 1 (if dependencies exist): +0.15
  enables ≥ 1: +0.15
  impacts ≥ 1: +0.15
  conflicts defined (or explicitly empty): +0.15
) / 1.00
```

### W7: Genome Score

```
GEN_score = filled_genome_fields / 8
```

### W8: DNA Score

```
DNA_score = (
  dna sealed: +0.50
  dna_hash valid: +0.30
  purpose.not_in_scope ≥ 1: +0.10
  fundamental_constraints ≥ 1: +0.10
) / 1.00
```

### W9: Learning Score

```
LRN_score = (
  difficulty.level defined: +0.25
  prerequisites ≥ 1: +0.25
  learning_path.beginner defined: +0.25
  knowledge_gaps ≥ 1: +0.25
) / 1.00
```

### W10: Search Score

```
SRH_score = (
  vector_embedding defined: +0.50
  indexed_fields ≥ 3: +0.30
  compression_preference defined: +0.20
) / 1.00
```

---

## Hallucination Penalty

```
IF memory.success_rate.rate exists:
  IF rate < 0.90: multiplier = 0.90  (10% penalty)
  IF rate < 0.80: multiplier = 0.80  (20% penalty)
  IF rate < 0.70: multiplier = 0.70  (30% penalty)
  ELSE:           multiplier = 1.00  (no penalty)
ELSE:
  multiplier = 1.00  (no data = no penalty)
```

---

## AIRS Levels

| Level | Score | Meaning |
|-------|-------|---------|
| AI-NATIVE | 0.90–1.00 | Fully optimized for AI; zero-hallucination target achievable |
| AI-READY | 0.80–0.89 | Ready for AI use; minor gaps exist |
| AI-CAPABLE | 0.70–0.79 | AI can use it; some hallucination risk |
| AI-PARTIAL | 0.60–0.69 | AI needs supplementary context to use safely |
| AI-MANUAL | 0.00–0.59 | Requires human review before AI use |

---

## Canonical Example

```yaml
# For: KNW-PLT-MOD-001 (Quota Manager)
intelligence:
  ai_readiness_score: 0.91
  # Level: AI-NATIVE

  # Component scores:
  # W1 self_describing:  0.97 × 0.15 = 0.1455
  # W2 ai_context:       0.98 × 0.20 = 0.1960
  # W3 compression:      1.00 × 0.10 = 0.1000
  # W4 confidence:       0.91 × 0.15 = 0.1365
  # W5 cortex:           0.95 × 0.15 = 0.1425
  # W6 reasoning:        0.90 × 0.10 = 0.0900
  # W7 genome:           1.00 × 0.05 = 0.0500
  # W8 dna:              1.00 × 0.05 = 0.0500
  # W9 learning:         0.90 × 0.03 = 0.0270
  # W10 search:          0.95 × 0.02 = 0.0190
  # ─────────────────────────────────
  # Subtotal:            0.9565
  # Hallucination mult:  × 1.00 (success_rate = 0.971 ≥ 0.90)
  # Final AIRS:          0.9565 → rounded to 0.91 (two decimal places)
```

---

## Rules

| Rule | Statement |
|------|-----------|
| KIL-135 | AIRS must be recomputed whenever any contributing dimension changes |
| KIL-136 | AIRS weights must sum to exactly 1.00 |
| KIL-137 | Objects with AIRS < 0.60 must be flagged and reported in the intelligence dashboard |
| KIL-138 | AIRS is a derived metric — it is never manually set |
| KIL-139 | CANONICAL objects must achieve AIRS ≥ 0.70 (AI-CAPABLE minimum) |

---

## Cross-References

- Intelligence dashboard → `25-KNOWLEDGE-INTELLIGENCE-DASHBOARD`
- KIL metrics → `26-KNOWLEDGE-METRICS`
- Confidence score → `13-KNOWLEDGE-CONFIDENCE`
- Cortex → `19-KNOWLEDGE-CORTEX`
- AI context → `04-AI-CONTEXT-LAYER`
