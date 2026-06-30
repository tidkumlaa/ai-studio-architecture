# KVF-DOC-019 — AI Understanding Benchmark

**Phase:** 3.0D.1.5
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The AI Understanding Benchmark is the highest-level validation of KOS: it
takes a new LLM with no prior knowledge of the project, gives it only the
KOS-assembled context (no source code, no documentation), and measures
whether the LLM can successfully complete real engineering tasks.

If KOS is working, the LLM should succeed at nearly all tasks using only
context assembled from the knowledge base.

---

## Benchmark Setup

```
Preconditions:
  1. A fresh LLM instance with no project-specific training
  2. KOS Runtime available to assemble context on demand
  3. Task set prepared (100 / 500 / 1,000 tasks)

Protocol per task:
  1. Task T is presented to the LLM
  2. LLM queries KOS: context = KOS.assemble(T.query, AI_AGENT, budget=STANDARD)
  3. LLM attempts to complete T using only the context pack
  4. Result is evaluated against ground truth
  5. LLM is NOT allowed to ask follow-up questions to KOS in this benchmark
     (single-shot: one context pack per task)
```

---

## Task Set

### Task Scale

| Scale | Task Count | Purpose |
|-------|-----------|---------|
| SMOKE | 100 | Quick validation in CI |
| STANDARD | 500 | Certification benchmark |
| FULL | 1,000 | Enterprise/Research certification |

### Task Types

```
TT-01 IDENTIFICATION (20%)
  "Identify what KNW-PLT-MOD-001 does in one sentence."
  Answer source: dna.purpose / ai_context.short_summary
  Success: answer matches ground truth semantically

TT-02 DEPENDENCY (15%)
  "List the 3 most critical dependencies of the Quota Manager."
  Answer source: reasoning.requires (sorted by severity)
  Success: all 3 dependencies named, in correct severity order

TT-03 IMPACT (15%)
  "What breaks if the Quota Manager is removed?"
  Answer source: reasoning.impacts (CRITICAL + HIGH)
  Success: all CRITICAL impacts named

TT-04 CONSTRAINT (10%)
  "What are the hard constraints on the Quota Manager?"
  Answer source: dna.fundamental_constraints
  Success: all constraints listed, none invented

TT-05 REASONING (20%)
  "Why does the Quota Manager exist? Explain its architectural purpose."
  Answer source: cortex.why, evolution.origin
  Success: reasoning uses declared WHY without inventing additional rationale

TT-06 COMPARISON (10%)
  "Compare the Quota Manager with the Rate Limiter."
  Answer source: diff block, alternatives
  Success: similarity_score within ±0.10 of declared, key differences named

TT-07 LIFECYCLE (10%)
  "Is the Quota Manager safe to use in production? Explain."
  Answer source: metadata.state, risk.failure_modes, confidence.level
  Success: correct state declared, risk summary accurate
```

---

## Evaluation Dimensions

```
Success Rate:
  proportion of tasks where the AI's answer is correct
  (task type-specific rubric; see below)

Completion Rate:
  proportion of tasks where the AI provides a substantive answer
  (vs. "I don't have enough information")
  Note: "I don't know" is CORRECT for tasks requiring missing context

Reasoning Quality:
  for TT-05 and TT-06 only
  1–4 depth scale from doc 09

Context Accuracy:
  proportion of AI claims that are traceable to the context pack
  (detects hallucination at task level)
```

---

## Success Rubrics by Task Type

```
TT-01 IDENTIFICATION:
  Full credit (1.0): semantically equivalent to ground truth
  Partial credit (0.5): major concept correct, detail wrong
  No credit (0.0): wrong object described

TT-02 DEPENDENCY:
  Full credit: all required deps named, no fabricated deps
  Partial credit (per dep): correct dep named
  No credit: fabricated dependency

TT-03 IMPACT:
  Full credit: all CRITICAL impacts named
  Partial credit: some CRITICAL, all HIGH
  No credit: critical impact missed or fabricated

TT-04 CONSTRAINT:
  Full credit: all constraints listed, none invented
  Partial credit: some constraints listed, none invented
  No credit: invented constraint

TT-05 REASONING:
  Full credit: WHY uses declared reasoning, reasoning depth >= 3
  Partial credit: some reasoning correct, some invented
  No credit: fabricated WHY

TT-06 COMPARISON:
  Full credit: sim_score within ±0.10, ≥ 3 key differences named
  Partial credit: sim_score within ±0.20 OR ≥ 2 key differences
  No credit: wrong objects compared or sim_score off by > 0.20

TT-07 LIFECYCLE:
  Full credit: correct state, risk level acknowledged
  Partial credit: correct state, risk not addressed
  No credit: wrong state declared
```

---

## AI Understanding Result Schema

```yaml
ai_understanding_result:
  task_scale: string                   # SMOKE | STANDARD | FULL
  tasks_run: integer
  llm_model: string

  success_rate:
    overall: float
    by_task_type:
      - type: string
        success_rate: float
        task_count: integer

  completion_rate: float               # substantive answer rate
  context_accuracy: float              # claims traceable to context

  reasoning_quality:
    mean_depth: float
    depth_distribution: {1: int, 2: int, 3: int, 4: int}

  hallucination_summary:
    tasks_with_any_hallucination: integer
    hallucination_rate_per_task: float

  context_sufficiency:
    single_pack_sufficient_rate: float  # task answerable with 1 pack
    would_need_more_context_rate: float # task needed more than 1 pack
```

---

## AI Understanding Targets

| Scale | Success Rate | Context Accuracy |
|-------|-------------|-----------------|
| BRONZE (100 tasks) | ≥ 0.65 | ≥ 0.80 |
| SILVER (500 tasks) | ≥ 0.75 | ≥ 0.85 |
| GOLD (500 tasks) | ≥ 0.85 | ≥ 0.90 |
| ENTERPRISE (1000 tasks) | ≥ 0.90 | ≥ 0.95 |
| RESEARCH (1000 tasks) | ≥ 0.95 | ≥ 0.98 |

---

## Rules

| Rule | Statement |
|------|-----------|
| KVF-091 | LLM must receive no project-specific training or examples — knowledge from KOS only |
| KVF-092 | Single-shot protocol: one context pack per task — no iterative querying |
| KVF-093 | "I don't know" for tasks with insufficient context counts as correct completion, not failure |
| KVF-094 | Task evaluation rubric must be applied consistently — the same rubric for all runs |
| KVF-095 | Context accuracy check must use hallucination detection from doc 10 |

---

## Cross-References

- Reasoning quality → `09-REASONING-QUALITY`
- Hallucination measurement → `10-HALLUCINATION-MEASUREMENT`
- Cross-agent consistency → `20-CROSS-AGENT-CONSISTENCY`
- Cold-start benchmark → `22-COLD-START-BENCHMARK`
