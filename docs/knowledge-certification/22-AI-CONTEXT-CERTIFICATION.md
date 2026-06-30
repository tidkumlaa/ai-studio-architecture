# KNW-CERT-ARCH-022 — AI Context Certification

**Phase:** 3.0D.0  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Certifies that the Knowledge System provides AI models (LLMs) with context that is complete, relevant, non-redundant, and within token budget. Context quality directly impacts reasoning accuracy and hallucination rate.

---

## Context Definition

When the KOS reasoning engine answers a question, it assembles a **context window** from Knowledge Objects. Context certification verifies that:
1. The context is complete enough to answer the question
2. The context is not padded with irrelevant objects
3. The context fits within the declared token budget
4. The context is structured consistently (PromptPack format)

---

## Context Quality Dimensions

| Dimension | Weight | Formula |
|-----------|--------|---------|
| Completeness | 0.35 | relevant_objects_included / total_relevant |
| Relevance | 0.30 | relevant_in_context / total_in_context |
| Conciseness | 0.15 | 1 − (redundant_tokens / total_tokens) |
| Token efficiency | 0.10 | information_bits / total_tokens |
| Structure consistency | 0.10 | correctly_formatted_objects / total |

---

## Token Budget

| Context Type | Budget (tokens) |
|--------------|-----------------|
| Single object lookup | ≤ 500 |
| Relationship query | ≤ 2,000 |
| Impact analysis | ≤ 5,000 |
| Full reasoning chain | ≤ 10,000 |
| Deep architecture question | ≤ 20,000 |

---

## Context Checks

### Completeness Checks

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Identity question context includes target object | AC-001 | CRITICAL | Target object always in context |
| Relationship question includes source + target | AC-002 | CRITICAL | Both endpoints in context |
| Impact question includes all direct dependents | AC-003 | MAJOR | All 1-hop dependents in context |
| Traceability question includes full chain | AC-004 | MAJOR | All chain objects in context |
| Evidence question includes evidence records | AC-005 | MAJOR | Evidence items in context |

### Relevance Checks

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Relevance ratio ≥ 0.80 | AC-006 | MAJOR | ≤ 20% irrelevant objects in context |
| No archived objects in context | AC-007 | MAJOR | Archived objects excluded by default |
| No DRAFT objects in context | AC-008 | MINOR | Unless explicitly requested |
| Context ordered by relevance | AC-009 | MINOR | Most relevant first |

### Token Budget Checks

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Context within declared budget | AC-010 | MAJOR | Token count ≤ budget for question type |
| Token count reported accurately | AC-011 | MINOR | Reported count matches actual |
| Budget exceeded: truncation not random | AC-012 | MAJOR | Least-relevant objects truncated |
| Empty context not returned | AC-013 | CRITICAL | Always ≥ 1 object in context |

### Format Checks

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Context is valid PromptPack format | AC-014 | MAJOR | Validates against PromptPack schema |
| Each object has status and confidence | AC-015 | MINOR | All objects include status + confidence fields |
| Context includes source citations | AC-016 | MAJOR | Each claim has knowledge_id citation |

---

## Metrics (run on 1,000 random questions)

| Metric | Bronze | Silver | Gold | Enterprise |
|--------|--------|--------|------|------------|
| Context completeness | ≥ 0.70 | ≥ 0.80 | ≥ 0.90 | ≥ 0.97 |
| Context relevance ratio | ≥ 0.65 | ≥ 0.75 | ≥ 0.85 | ≥ 0.93 |
| Token budget adherence | ≥ 0.85 | ≥ 0.92 | ≥ 0.97 | = 1.0 |
| Format validity | ≥ 0.95 | ≥ 0.98 | = 1.0 | = 1.0 |

---

## Report Format

```json
{
  "domain": "ai_context",
  "questions_tested": 1000,
  "context_completeness": 0.921,
  "context_relevance": 0.883,
  "token_budget_adherence": 0.973,
  "format_validity": 1.0,
  "mean_tokens_per_question": 3241,
  "checks": {
    "AC-001": "PASS", "AC-006": "PASS",
    "AC-010": "PASS", "AC-013": "PASS",
    "AC-014": "PASS"
  },
  "domain_score": 0.912,
  "level_achieved": "Gold"
}
```

---

## CLI

```bash
kos-cert ai-context                      # 1000 questions
kos-cert ai-context --questions 5000
kos-cert ai-context --budget-only        # token budget checks only
kos-cert ai-context --output reports/ai-context.json
```

---

## Cross-References

- PromptPack format → Phase 3.0C.5 `24-KNOWLEDGE-IMPORT-EXPORT`
- Reasoning certification → `13-REASONING-CERTIFICATION`
- Hallucination (caused by incomplete context) → `23-HALLUCINATION-CERTIFICATION`
