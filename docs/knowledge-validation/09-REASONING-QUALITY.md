# KVF-DOC-009 — Reasoning Quality

**Phase:** 3.0D.1.5
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Reasoning Quality measures whether an AI agent using KOS context can answer
architecture questions correctly, with appropriate reasoning depth, without
inventing facts. The agent is given KOS-assembled context and asked questions.
Its answers are classified as CORRECT, UNKNOWN, or HALLUCINATION.

---

## 4 Question Types

### QT-ARCH: Architecture Questions

```
Definition: Questions about the structural design of the system.
Examples:
  - "What does the Quota Manager do?"
  - "Why does the Quota Manager exist?"
  - "What is the purpose of the rate limiter?"

Answer source: dna.purpose, cortex.why, genome.capabilities
Expected answer characteristics:
  - Names the object's core function
  - Names the domain it belongs to
  - Does not invent capabilities not declared
```

### QT-DEP: Dependency Questions

```
Definition: Questions about what an object depends on or is depended on by.
Examples:
  - "What does the Quota Manager depend on?"
  - "Which components use the Quota Manager?"
  - "What breaks if the Quota Manager is removed?"

Answer source: reasoning.requires, reasoning.impacts, semantic.consumes
Expected answer characteristics:
  - Lists only declared dependencies
  - Distinguishes HARD from SOFT dependencies
  - Does not invent new dependency relationships
```

### QT-IMPACT: Impact Questions

```
Definition: Questions about what happens if an object changes or fails.
Examples:
  - "What services are affected if the Quota Manager fails?"
  - "What is the blast radius of changing the Quota Manager's interface?"
  - "Which tests would fail if Quota Manager is deprecated?"

Answer source: reasoning.impacts, risk.failure_modes, semantic.provides
Expected answer characteristics:
  - Lists downstream objects in order of severity
  - Distinguishes CRITICAL from HIGH from MEDIUM impacts
  - Does not claim impact on unrelated objects
```

### QT-LIFECYCLE: Lifecycle Questions

```
Definition: Questions about an object's state, history, and future.
Examples:
  - "Is the Quota Manager still active?"
  - "When was the Quota Manager's current behavior introduced?"
  - "Will the Quota Manager be deprecated?"

Answer source: metadata.state, evolution.history, decisions, deprecation
Expected answer characteristics:
  - Correctly states current state
  - References actual evolution history
  - Does not predict future state without evidence
```

---

## Answer Classification

```
CORRECT:
  The answer is factually accurate, agrees with the source KIL object,
  and does not contain unsupported claims.

UNKNOWN (acceptable):
  The agent says "I don't know" or "This information is not in the
  provided context." This is the correct response when context is
  insufficient — it means the guard block worked.

HALLUCINATION:
  The answer contains at least one factual claim that:
    (a) directly contradicts the KIL object, OR
    (b) is not derivable from any field of any object in the context pack

Hallucination types from doc 10:
  Type A: Wrong fact
  Type B: Wrong dependency
  Type C: Wrong architecture
  Type D: Wrong relationship
  Type E: Wrong confidence
  Type F: Unsupported conclusion
```

---

## Reasoning Depth Measurement

```
For each CORRECT answer:
  reasoning_depth = count(distinct reasoning steps used)

  Step types:
    FACT_RETRIEVAL:    agent cited a direct fact from context
    INFERENCE:         agent derived conclusion from 2+ facts
    CAUSAL_CHAIN:      agent followed A → B → C relationship chain
    COMPARISON:        agent compared two objects
    QUALIFICATION:     agent applied a guard/hedge appropriately

  reasoning_depth score:
    1 step (fact retrieval only): depth = 1
    2–3 steps: depth = 2
    4+ steps with causal chain: depth = 3
    4+ steps with qualification AND causal chain: depth = 4

Target mean reasoning depth: >= 2.5 for REASONING_QUERY intent
```

---

## Reasoning Quality Benchmark

```
Question set composition (per certification run):
  50 QT-ARCH questions
  50 QT-DEP questions
  50 QT-IMPACT questions
  50 QT-LIFECYCLE questions
  Total: 200 questions minimum

Objects covered:
  At least 20 distinct KnowledgeObjects
  Including at least 5 objects from different packages
  Including at least 2 DEPRECATED objects (for lifecycle questions)

Evaluator:
  AI judge (GPT-4 class or equivalent)
  Human spot-check: 20% of answers reviewed by human annotator
  Agreement threshold: AI judge agrees with human on 90%+ of cases
```

---

## Reasoning Quality Result Schema

```yaml
reasoning_quality_result:
  questions_evaluated: integer

  by_question_type:
    - type: string                      # QT-ARCH | QT-DEP | QT-IMPACT | QT-LIFECYCLE
      total: integer
      correct: integer
      unknown: integer
      hallucination: integer
      correct_rate: float
      hallucination_rate: float

  overall:
    correct_rate: float
    unknown_rate: float
    hallucination_rate: float

  reasoning_depth:
    mean: float
    distribution: {1: integer, 2: integer, 3: integer, 4: integer}

  hallucination_breakdown:
    type_a: integer                     # wrong fact
    type_b: integer                     # wrong dependency
    type_c: integer                     # wrong architecture
    type_d: integer                     # wrong relationship
    type_e: integer                     # wrong confidence
    type_f: integer                     # unsupported conclusion
```

---

## Reasoning Quality Targets

| Metric | BRONZE | SILVER | GOLD | ENTERPRISE |
|--------|--------|--------|------|------------|
| Correct rate | ≥ 0.65 | ≥ 0.75 | ≥ 0.85 | ≥ 0.90 |
| Hallucination rate | < 0.15 | < 0.10 | < 0.05 | < 0.02 |
| Unknown rate | ≤ 0.30 | ≤ 0.20 | ≤ 0.15 | ≤ 0.10 |
| Mean reasoning depth | ≥ 1.5 | ≥ 2.0 | ≥ 2.5 | ≥ 3.0 |

---

## Rules

| Rule | Statement |
|------|-----------|
| KVF-041 | Reasoning quality must be measured on all 4 question types — not just QT-ARCH |
| KVF-042 | UNKNOWN answers must not be penalized — they are the correct response to insufficient context |
| KVF-043 | Hallucination classification must specify the type (A–F) — unclassified hallucinations count as Type F |
| KVF-044 | AI judge must be validated against human annotation on 20% of cases before use |
| KVF-045 | Reasoning depth must be measured only on CORRECT answers — depth of wrong answers is meaningless |

---

## Cross-References

- Hallucination measurement → `10-HALLUCINATION-MEASUREMENT`
- AI understanding → `19-AI-UNDERSTANDING`
- Reasoning model spec → Phase 3.0D.0.6 `06-REASONING-MODEL`
- Context thinking layer → Phase 3.0D.0.6 `20-KNOWLEDGE-THINKING-LAYER`
