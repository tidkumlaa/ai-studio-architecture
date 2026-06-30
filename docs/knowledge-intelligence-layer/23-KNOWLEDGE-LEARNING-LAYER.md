# KNW-KIL-DOC-023 — Knowledge Learning Layer

**Phase:** 3.0D.0.6
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Learning Layer describes how Knowledge Objects are learned — by humans,
by AI agents, and by the knowledge base itself. It captures the difficulty of
mastering an object, the prerequisites needed, the recommended learning order,
and the knowledge that AI agents have accumulated from their interactions.

This enables the Knowledge Runtime to serve objects in optimal learning order
and to track which concepts are hard to teach.

This document defines `intelligence.learning`.

---

## Schema

```yaml
intelligence:
  learning:
    schema_version: "1.0"

    difficulty:
      level: TRIVIAL | EASY | MEDIUM | HARD | EXPERT
      score: float                     # 0.0–1.0 (0 = trivial, 1 = expert)
      reasoning: string                # why this difficulty rating
      hard_parts: [string]             # specific aspects that are hard to learn

    prerequisites:                     # what must be understood first
      - knowledge_id: string
        reason: string                 # why this prerequisite is needed
        depth: AWARENESS | WORKING | PROFICIENT | EXPERT
        mandatory: boolean

    learning_path:                     # recommended sequence for mastering this object
      beginner:
        steps:
          - step: integer
            action: string             # e.g. "read nano summary"
            time_estimate_minutes: integer
            resource: string           # what to read/do
      intermediate:
        steps:
          - step: integer
            action: string
            time_estimate_minutes: integer
            resource: string
      advanced:
        steps:
          - step: integer
            action: string
            time_estimate_minutes: integer
            resource: string

    popularity:                        # how widely known/used this knowledge is
      in_domain: float                 # 0.0–1.0 among domain experts
      cross_domain: float              # 0.0–1.0 across all domains
      agent_familiarity: float         # 0.0–1.0 — how well AI agents know it

    knowledge_gaps:                    # gaps commonly seen in learners
      - gap_id: string
        description: string
        frequency: HIGH | MEDIUM | LOW
        symptom: string                # what behavior reveals this gap
        cure: string                   # how to close the gap

    ai_learning:                       # what AI agents have learned from interactions
      total_training_exposures: integer
      correct_response_rate: float
      improvement_over_time: float     # delta in correctness since first exposure
      hardest_concept: string          # the single hardest thing for AI to learn
      best_prompt_pattern: string      # the prompt style that produces best results
```

---

## Difficulty Levels

| Level | Score | Meaning | Typical Prerequisite Count |
|-------|-------|---------|---------------------------|
| TRIVIAL | 0.0–0.2 | Any developer can use without preparation | 0–1 |
| EASY | 0.2–0.4 | Requires 30 min reading | 1–2 |
| MEDIUM | 0.4–0.6 | Requires understanding 3–5 related concepts | 3–5 |
| HARD | 0.6–0.8 | Requires deep domain knowledge | 5–8 |
| EXPERT | 0.8–1.0 | Requires multiple years of domain experience | 8+ |

---

## Prerequisite Depth Levels

| Depth | Meaning |
|-------|---------|
| AWARENESS | Know it exists and what it does |
| WORKING | Can use it without deep understanding |
| PROFICIENT | Can modify/extend it |
| EXPERT | Can design alternatives to it |

---

## Canonical Example

```yaml
# For: KNW-PLT-MOD-001 (Quota Manager)
intelligence:
  learning:
    schema_version: "1.0"

    difficulty:
      level: MEDIUM
      score: 0.55
      reasoning: >
        Using Quota Manager is easy (call check(), handle ALLOW/DENY).
        Understanding it deeply — why synchronous, why read-only, why Registry
        is HARD dependency — requires understanding multi-tenant architecture,
        the Guard pattern, and consistency requirements.
      hard_parts:
        - "Understanding WHY synchronous enforcement is required (not async)"
        - "Understanding the separation: enforcement vs. configuration"
        - "Atomic quota check-and-commit with Registry"

    prerequisites:
      - knowledge_id: KNW-PLT-REQ-001
        reason: "Explains the requirement that defines Quota Manager's behavior"
        depth: WORKING
        mandatory: true
      - knowledge_id: KNW-RT-RT-001
        reason: "Explains where quota state is stored and how atomicity works"
        depth: AWARENESS
        mandatory: true
      - knowledge_id: KNW-META-PAT-003
        reason: "Guard pattern explains the enforcement structure"
        depth: AWARENESS
        mandatory: false
      - knowledge_id: KNW-ALG-ALG-009
        reason: "Understanding Rate Limiter helps avoid confusing it with Quota Manager"
        depth: AWARENESS
        mandatory: false

    learning_path:
      beginner:
        steps:
          - step: 1
            action: "Read nano + micro summary"
            time_estimate_minutes: 2
            resource: "intelligence.compression.nano + micro"
          - step: 2
            action: "Read the AI context short_summary and medium_summary"
            time_estimate_minutes: 5
            resource: "intelligence.ai_context.short_summary + medium_summary"
          - step: 3
            action: "Read common_questions in ai_context"
            time_estimate_minutes: 5
            resource: "intelligence.ai_context.common_questions"
      intermediate:
        steps:
          - step: 1
            action: "Read the DNA: purpose, core_behavior, primary_interfaces"
            time_estimate_minutes: 10
            resource: "intelligence.dna"
          - step: 2
            action: "Read the tradeoffs (TO-001, TO-002)"
            time_estimate_minutes: 15
            resource: "intelligence.tradeoffs"
          - step: 3
            action: "Read prerequisite KNW-PLT-REQ-001 at WORKING depth"
            time_estimate_minutes: 10
            resource: "KNW-PLT-REQ-001 intelligence.ai_context.medium_summary"
      advanced:
        steps:
          - step: 1
            action: "Read full reasoning model and risk model"
            time_estimate_minutes: 20
            resource: "intelligence.reasoning + intelligence.risk"
          - step: 2
            action: "Read decisions DEC-001 and DEC-002"
            time_estimate_minutes: 15
            resource: "intelligence.decisions"
          - step: 3
            action: "Study alternatives ALT-001 through ALT-004"
            time_estimate_minutes: 20
            resource: "intelligence.alternatives"

    popularity:
      in_domain: 0.95
      cross_domain: 0.60
      agent_familiarity: 0.88

    knowledge_gaps:
      - gap_id: KG-001
        description: "Confusing Quota Manager with Rate Limiter"
        frequency: HIGH
        symptom: "Developer tries to use quota check for per-endpoint req/s control"
        cure: "Read ALT-003 (Rate Limiter as PARALLEL alternative)"
      - gap_id: KG-002
        description: "Attempting to set quota limits via Quota Manager"
        frequency: MEDIUM
        symptom: "Developer looks for setLimit() method in Quota Manager interface"
        cure: "Read dna.purpose.not_in_scope and cortex.why_not entries"
      - gap_id: KG-003
        description: "Not understanding why synchronous enforcement is required"
        frequency: MEDIUM
        symptom: "Developer proposes async quota check for performance"
        cure: "Read tradeoffs TO-001 and decisions DEC-001"

    ai_learning:
      total_training_exposures: 14827
      correct_response_rate: 0.971
      improvement_over_time: 0.12    # 12% improvement from first to current
      hardest_concept: "The synchronous enforcement requirement and its rationale"
      best_prompt_pattern: >
        Inject ai_context.full_summary + cortex.why_not + ai_context.typical_mistakes
        before the user question. This combination eliminates 90% of hallucinations.
```

---

## Rules

| Rule | Statement |
|------|-----------|
| KIL-127 | Every CANONICAL object must have `difficulty.level` and ≥ 1 `prerequisite` |
| KIL-128 | `knowledge_gaps` must be derived from real observed gaps (memory.failures) |
| KIL-129 | `ai_learning.best_prompt_pattern` is the most important field for PromptPack optimization |
| KIL-130 | `prerequisites` with `mandatory: true` must appear in `reasoning.depends` |

---

## Cross-References

- Memory (usage data) → `09-KNOWLEDGE-MEMORY`
- AI context → `04-AI-CONTEXT-LAYER`
- Cortex (common errors) → `19-KNOWLEDGE-CORTEX`
- Search optimization → `28-KNOWLEDGE-SEARCH-OPTIMIZATION`
