# KVF-DOC-006 — Token Efficiency

**Phase:** 3.0D.1.5
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Token Efficiency measures the most critical economic claim of KOS:
that using KOS to assemble context requires dramatically fewer tokens
than providing the same information without KOS, while maintaining
equivalent or higher context quality.

**Target: > 95% token reduction vs. naive baseline.**

---

## The Efficiency Claim

```
Without KOS:
  An AI agent needing to answer a question about system X would receive:
  - Full source code files (10K–100K tokens)
  - Full documentation files (5K–50K tokens)
  - Meeting notes, issues, PRs (varies)
  Total: 20K–500K tokens per query

With KOS:
  The same AI agent receives a context pack:
  - Primary object at L4 (≤ 500 tokens)
  - 3–5 context objects at L2 (≤ 250 tokens)
  - Guard block (≤ 100 tokens)
  Total: 500–2,000 tokens per query

Token Reduction = 1 - (tokens_with_kos / tokens_without_kos)
Target: > 95% reduction (i.e. KOS uses < 5% of naive tokens)
```

---

## Baseline Definition

```
BASELINE (without KOS):
  For a given query Q about knowledge object O:
  naive_context = all files that a developer would manually gather:
    - Source file(s) containing O (full file)
    - Documentation files mentioning O (full files)
    - Interface/config files O depends on (full files)
    - README files for the packages containing O

  naive_tokens = count_tokens(naive_context)

  Baseline must be constructed by human annotator who identifies
  what files they would actually open to answer query Q.
  Not algorithmic — not the full corpus.
```

---

## KOS Context Tokens

```
KOS (with KOS):
  kos_context = assemble(Q, consumer=AI_AGENT, budget=STANDARD=1000)
  kos_tokens = pack.pack_metadata.total_tokens
```

---

## Token Efficiency Metrics

```
token_reduction(Q) = 1 - (kos_tokens(Q) / naive_tokens(Q))

Expected distribution:
  Simple FACTUAL_LOOKUP: 97–99% reduction
    naive: ~20K tokens (source + docs)
    kos: ~500–800 tokens

  Complex REASONING_QUERY: 93–97% reduction
    naive: ~50K tokens
    kos: ~1,000–2,000 tokens

  IMPACT_ANALYSIS: 90–95% reduction
    naive: ~100K tokens (full dependency tree)
    kos: ~3,000–8,000 tokens
```

---

## Information Preservation Check

Token reduction alone is insufficient — the reduced context must preserve
the information needed to answer the query correctly.

```
For each query in the efficiency benchmark:

  1. Compute token_reduction
  2. Ask a reference AI agent to answer Q using naive_context
     → naive_answer
  3. Ask the same AI agent to answer Q using kos_context
     → kos_answer
  4. Human judge: does kos_answer preserve all critical facts from naive_answer?
     → information_preserved (boolean)
  5. Human judge: does kos_answer introduce any errors not in naive_answer?
     → no_errors_introduced (boolean)

  Efficiency with preservation =
    token_reduction × information_preserved × no_errors_introduced
```

---

## Duplicate Removal Measurement

KOS eliminates information that appears in multiple source files:

```
duplicate_removal_rate(O):
  full_corpus_mentions = count(files in naive_context mentioning O)
  kos_mentions = 1  (KOS has one canonical source)

  duplicate_removal_rate = 1 - (1 / full_corpus_mentions)
  Expected: > 80% of information is duplicated across naive files
```

---

## Token Efficiency Result Schema

```yaml
token_efficiency_result:
  queries_measured: integer
  baselines_human_annotated: integer

  token_reduction:
    mean: float
    median: float
    p10: float                          # worst 10% cases
    p90: float                          # best 90% cases
    above_95pct: float                  # proportion meeting target
    above_90pct: float

  by_intent_type:
    - intent: string
      mean_reduction: float
      median_reduction: float
      query_count: integer

  information_preservation:
    queries_preserved: integer
    preservation_rate: float
    errors_introduced: integer
    error_rate: float

  duplicate_removal:
    mean_duplicate_removal_rate: float
    objects_with_>3_source_files: integer

  overall_efficiency_score: float       # token_reduction × preservation_rate
```

---

## Efficiency Targets by Certification Level

| Level | Mean Token Reduction | Preservation Rate |
|-------|---------------------|------------------|
| BRONZE | ≥ 80% | ≥ 0.85 |
| SILVER | ≥ 90% | ≥ 0.90 |
| GOLD | ≥ 95% | ≥ 0.95 |
| ENTERPRISE | ≥ 97% | ≥ 0.97 |
| RESEARCH | ≥ 99% | ≥ 0.99 |

---

## Benchmark Scale

| Benchmark | Queries | Baseline Annotation |
|-----------|---------|-------------------|
| SMOKE | 10 | Human |
| STANDARD | 100 | Human (50) + automated (50) |
| FULL | 500 | Human (100) + automated (400) |
| RESEARCH | 1,000 | Human (200) + automated (800) |

---

## Rules

| Rule | Statement |
|------|-----------|
| KVF-026 | Baseline must be human-annotated for at least 50 queries per benchmark run |
| KVF-027 | Token reduction claimed without information preservation measurement is invalid |
| KVF-028 | Token count must use the same tokenizer for both naive and KOS measurements |
| KVF-029 | IMPACT_ANALYSIS queries have lower reduction targets — they genuinely need more context |
| KVF-030 | Overall efficiency score = token_reduction × preservation_rate (both components required) |

---

## Cross-References

- Context quality validation → `04-CONTEXT-QUALITY-VALIDATION`
- Context compression → `11-CONTEXT-COMPRESSION`
- Knowledge coverage → `07-KNOWLEDGE-COVERAGE`
- KIL compression spec → Phase 3.0D.0.6 `05-KNOWLEDGE-COMPRESSION`
