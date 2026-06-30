# KNW-KIL-DOC-020 — Knowledge Thinking Layer

**Phase:** 3.0D.0.6
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Thinking Layer defines the *structured reasoning protocol* for any agent
reasoning about a Knowledge Object. It answers: in what order should an agent
process the object's information to arrive at a correct, non-hallucinated conclusion?

While the Cortex captures static meta-knowledge about the object, the Thinking
Layer is a dynamic protocol — a sequence of reasoning steps that any agent
(human or AI) should follow when answering questions about this object.

This document defines `intelligence.thinking`.

---

## Schema

```yaml
intelligence:
  thinking:
    schema_version: "1.0"

    reasoning_protocol:
      name: string                     # protocol name e.g. "QuotaManager-Protocol"
      version: string
      applicable_question_types: [string] # types this protocol handles

      steps:
        - step_id: string              # e.g. THINK-001
          name: string
          question: string             # what does the agent ask at this step?
          information_sources:         # where to find the answer
            - source_type: FIELD | BLOCK | RELATED_OBJECT | EXTERNAL_DOC
              location: string         # e.g. "dna.core_behavior.primary_action"
              priority: PRIMARY | FALLBACK
          output: string               # what the step produces (feeds into next step)
          failure_action: string       # what to do if information is unavailable

      guard_conditions:                # conditions that must be checked first
        - condition_id: string
          check: string
          failure_means: string
          recovery: string

      completion_criteria:
        sufficient_confidence: float   # minimum confidence to emit an answer
        required_sources: [string]     # step_ids that must complete before answering
        fallback: string               # what to say if criteria not met

    chain_of_thought_template:         # CoT template for this object type
      preamble: string                 # context to establish before reasoning
      reasoning_chain: [string]        # ordered reasoning statements
      conclusion_format: string        # how to structure the conclusion

    anti_reasoning:                    # reasoning patterns that lead to wrong answers
      - pattern_id: string
        name: string
        description: string
        why_wrong: string
        correction: string
```

---

## Canonical Example

```yaml
# For: KNW-PLT-MOD-001 (Quota Manager)
intelligence:
  thinking:
    schema_version: "1.0"

    reasoning_protocol:
      name: "QuotaManager-Reasoning-Protocol"
      version: "1.0"
      applicable_question_types:
        - "what does X do?"
        - "how does X work?"
        - "why does X exist?"
        - "what happens when X fails?"
        - "what does X depend on?"
        - "is X the same as Y?"

      steps:
        - step_id: THINK-001
          name: "Establish object identity"
          question: "What type of object is this and what is its fundamental purpose?"
          information_sources:
            - source_type: FIELD
              location: "dna.identity"
              priority: PRIMARY
            - source_type: FIELD
              location: "dna.purpose.statement"
              priority: PRIMARY
          output: "Object is a [type] in [namespace] that [purpose]"
          failure_action: "Halt — cannot reason without identity"

        - step_id: THINK-002
          name: "Identify scope boundaries"
          question: "What does this object explicitly NOT do?"
          information_sources:
            - source_type: FIELD
              location: "dna.purpose.not_in_scope"
              priority: PRIMARY
            - source_type: BLOCK
              location: "cortex.why_not"
              priority: PRIMARY
          output: "This object does NOT: [list]"
          failure_action: "Warn — scope boundaries unclear"

        - step_id: THINK-003
          name: "Identify core behavior"
          question: "What is the essential input → output contract?"
          information_sources:
            - source_type: FIELD
              location: "dna.core_behavior"
              priority: PRIMARY
            - source_type: BLOCK
              location: "self_describing.inputs"
              priority: FALLBACK
          output: "Given [inputs], it produces [outputs], constrained by [invariants]"
          failure_action: "Use self_describing as fallback"

        - step_id: THINK-004
          name: "Map dependencies"
          question: "What does it depend on to function?"
          information_sources:
            - source_type: BLOCK
              location: "reasoning.requires"
              priority: PRIMARY
            - source_type: BLOCK
              location: "genome.relationships.strong_dependencies"
              priority: FALLBACK
          output: "Requires: [list of dependencies and their role]"
          failure_action: "Note that dependency map is incomplete"

        - step_id: THINK-005
          name: "Apply known common errors"
          question: "What are the most common wrong answers about this object?"
          information_sources:
            - source_type: BLOCK
              location: "cortex.common_errors"
              priority: PRIMARY
            - source_type: BLOCK
              location: "ai_context.typical_mistakes"
              priority: PRIMARY
          output: "Must NOT say: [list of wrong statements]"
          failure_action: "Proceed without error check — increased hallucination risk"

        - step_id: THINK-006
          name: "Check confidence"
          question: "How confident should I be in this answer?"
          information_sources:
            - source_type: BLOCK
              location: "confidence"
              priority: PRIMARY
          output: "Confidence [level] — apply hedging: [hedge phrase]"
          failure_action: "Assume MEDIUM confidence"

      guard_conditions:
        - condition_id: GC-001
          check: "Is the question about quota configuration (setting limits)?"
          failure_means: "Question is outside this object's scope"
          recovery: "Redirect to Admin API; note that Quota Manager only enforces limits"
        - condition_id: GC-002
          check: "Is the question about rate limiting (req/s)?"
          failure_means: "Confusion with KNW-ALG-ALG-009"
          recovery: "Clarify distinction: quota = cumulative resources; rate = req/s velocity"

      completion_criteria:
        sufficient_confidence: 0.70
        required_sources: ["THINK-001", "THINK-003", "THINK-005"]
        fallback: >
          "I cannot answer this with sufficient confidence. Available information
          indicates [partial answer], but [specific gap] is unclear."

    chain_of_thought_template:
      preamble: >
        KNW-PLT-MOD-001 is the Quota Manager MODULE in the plt namespace.
        It enforces per-tenant resource quotas. It does NOT configure quotas.
        It does NOT rate-limit by req/s. It depends on KNW-RT-RT-001.
      reasoning_chain:
        - "The question is about [question topic]"
        - "Quota Manager's relevant behavior is [core_behavior or capability]"
        - "The key constraint is [relevant constraint]"
        - "This is supported by [evidence or source]"
        - "Common confusion to avoid: [relevant why_not or common_error]"
      conclusion_format: >
        "[Answer statement]. [Evidence citation]. [Constraint note if relevant]."

    anti_reasoning:
      - pattern_id: AR-001
        name: "Assuming enforcement implies configuration"
        description: "Concluding that since Quota Manager enforces limits, it also sets them"
        why_wrong: "Enforcement and configuration are intentionally separated (FDC-002)"
        correction: "Enforcement reads limits from Config; Admin API writes them"
      - pattern_id: AR-002
        name: "Ignoring the P99 constraint in reasoning"
        description: "Describing quota check behavior without mentioning latency requirement"
        why_wrong: "P99 < 5ms is a fundamental constraint that drives all implementation choices"
        correction: "Always include P99 < 5ms when describing how the quota check works"
      - pattern_id: AR-003
        name: "Treating quota check as idempotent"
        description: "Assuming calling quota check twice for the same request is safe"
        why_wrong: "Each check consumes quota — double-checking double-counts"
        correction: "Quota check must be called exactly once per resource request"
```

---

## Rules

| Rule | Statement |
|------|-----------|
| KIL-113 | Every CANONICAL MODULE and SERVICE must have a `reasoning_protocol` in `thinking` |
| KIL-114 | `completion_criteria.sufficient_confidence` must match the object's `confidence.level` threshold |
| KIL-115 | `anti_reasoning` must be derived from `cortex.common_errors` — they must not contradict |
| KIL-116 | `guard_conditions` must cover all `cortex.why_not` exclusions |
| KIL-117 | `chain_of_thought_template` must be testable against KAT-006 rubric |

---

## Cross-References

- Cortex → `19-KNOWLEDGE-CORTEX`
- Explanation layer → `21-KNOWLEDGE-EXPLANATION-LAYER`
- Reasoning model → `06-REASONING-MODEL`
- Confidence → `13-KNOWLEDGE-CONFIDENCE`
- Acceptance tests KAT-006 → Phase 3.0D.0.5 `17-KNOWLEDGE-ACCEPTANCE-TEST`
