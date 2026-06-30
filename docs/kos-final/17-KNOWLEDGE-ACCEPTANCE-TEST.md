# KNW-FINAL-017 — Knowledge Acceptance Test

**Phase:** 3.0D.0.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Defines the 10 official Knowledge Acceptance Tests (KAT-001 through KAT-010). An implementation must pass all 10 to achieve KOS v1.0 certification.

---

## Acceptance Test Framework

Each KAT test:
- Has a fixed set of questions (pinned, versioned)
- Has machine-verifiable ground-truth answers
- Has a minimum pass rate to achieve each certification level
- Is reproducible with seed 42 and golden dataset v1.0

---

## KAT-001 — Architecture Question Answering

**What it tests:** AI must answer architecture questions about KOS.

```yaml
kat_id: KAT-001
name: Architecture Question Answering
question_count: 100
categories: [identity, architecture, capability]
sample_questions:
  - "What are the 9 KOS packages?"
  - "How many object types does KOS v1.0 define?"
  - "What is the minimum quality score for CANONICAL state?"
  - "What does kos format guarantee?"
```

| Level | Min Pass Rate |
|-------|--------------|
| Bronze | ≥ 0.70 |
| Silver | ≥ 0.78 |
| Gold | ≥ 0.86 |
| Enterprise | ≥ 0.93 |
| Research | ≥ 0.97 |

---

## KAT-002 — Relationship Resolution

**What it tests:** AI must correctly identify and traverse relationships between objects.

```yaml
kat_id: KAT-002
name: Relationship Resolution
question_count: 100
categories: [relationship, dependency]
sample_questions:
  - "What does KNW-PLT-MOD-001 depend on?"
  - "What implements Routing Engine?"
  - "What tests KNW-PLT-MOD-001?"
  - "What does Provider Registry provide?"
```

Pass rate targets: same as KAT-001.

---

## KAT-003 — Traceability Traversal

**What it tests:** AI must traverse complete 9-level traceability chains.

```yaml
kat_id: KAT-003
name: Traceability Traversal
question_count: 50
categories: [traceability]
sample_questions:
  - "What architecture document satisfies KNW-PLT-REQ-001?"
  - "What test verifies the Quota Manager?"
  - "Is the quota enforcement chain complete?"
  - "What is missing from the AI routing traceability chain?"
pass_criteria:
  chain_traversal_accuracy: ≥ 0.85 (Gold)
  gap_detection_accuracy: ≥ 0.80 (Gold)
```

---

## KAT-004 — Dependency Identification

**What it tests:** AI must correctly identify package and object dependencies.

```yaml
kat_id: KAT-004
name: Dependency Identification
question_count: 100
categories: [dependency]
sample_questions:
  - "What does kos.platform.package depend on?"
  - "What are the leaf packages?"
  - "Can kos.test.package use kos.runtime.package objects?"
  - "What is the dependency closure of kos.runtime.package?"
```

---

## KAT-005 — Conflict Detection

**What it tests:** AI must detect dependency conflicts, circular dependencies, and broken references.

```yaml
kat_id: KAT-005
name: Conflict Detection
question_count: 50
categories: [dependency, security]
sample_questions:
  - "Is there a circular dependency between X and Y?"
  - "What packages conflict in this scenario?"
  - "Is this cross-namespace reference valid?"
  - "What is the resolution order for this dependency graph?"
pass_criteria:
  conflict_detection_accuracy: ≥ 0.90 (Gold)
```

---

## KAT-006 — Architecture Explanation

**What it tests:** AI must explain KOS architecture components clearly and correctly.

```yaml
kat_id: KAT-006
name: Architecture Explanation
question_count: 50
categories: [architecture, reasoning]
sample_questions:
  - "Explain how the lifecycle promotion works"
  - "Why does the identity engine exclude knowledge_id from checksums?"
  - "What is the difference between DEPENDS_ON and USES?"
  - "Explain the quality scoring formula"
evaluation:
  type: rubric   # structured rubric, not exact match
  criteria:
    - technically_accurate: weight 0.50
    - cites_source: weight 0.30
    - not_hallucinated: weight 0.20
```

---

## KAT-007 — Execution Context Production

**What it tests:** AI must produce correct PromptPack execution context for given queries.

```yaml
kat_id: KAT-007
name: Execution Context Production
question_count: 50
categories: [ai_context]
task: "Given question Q, produce the PromptPack context that an AI should use to answer it"
pass_criteria:
  context_completeness: ≥ 0.85 (Gold)
  relevance_ratio: ≥ 0.80 (Gold)
  within_token_budget: = 1.0
```

---

## KAT-008 — Canonical Source Identification

**What it tests:** AI must identify the canonical (highest-authority) source for any claim.

```yaml
kat_id: KAT-008
name: Canonical Source Identification
question_count: 50
categories: [evidence, sources]
sample_questions:
  - "What is the canonical source for the P99 < 5ms registry latency target?"
  - "What evidence supports BM25 as the ranking algorithm?"
  - "Which is more authoritative: ADR-001 or the production code?"
pass_criteria:
  source_accuracy: ≥ 0.85 (Gold)
  level_accuracy: ≥ 0.90 (Gold)
```

---

## KAT-009 — Evidence Identification

**What it tests:** AI must correctly identify and evaluate evidence for objects.

```yaml
kat_id: KAT-009
name: Evidence Identification
question_count: 50
categories: [evidence]
sample_questions:
  - "What evidence supports KNW-PLT-MOD-001?"
  - "Is the evidence for BM25 ranker fresh?"
  - "What is the quality score of KNW-PLT-MOD-001 and why?"
  - "Which object has the weakest evidence?"
pass_criteria:
  evidence_accuracy: ≥ 0.85 (Gold)
```

---

## KAT-010 — Impact Analysis

**What it tests:** AI must calculate the full impact of removing, deprecating, or changing an object.

```yaml
kat_id: KAT-010
name: Impact Analysis
question_count: 50
categories: [impact]
sample_questions:
  - "What breaks if KNW-RT-RT-001 is removed?"
  - "What is the minimum risk change to kos.platform.package?"
  - "If we upgrade kos.meta.package to 2.0.0, what breaks?"
  - "What is the ripple effect of deprecating Provider Registry?"
pass_criteria:
  direct_impact_accuracy: ≥ 0.90 (Gold)
  transitive_impact_accuracy: ≥ 0.80 (Gold)
  severity_classification: ≥ 0.85 (Gold)
```

---

## Acceptance Test Summary

| Test | Description | Questions | Primary Metric |
|------|-------------|-----------|----------------|
| KAT-001 | Architecture Q&A | 100 | Correct rate |
| KAT-002 | Relationship resolution | 100 | Correct rate |
| KAT-003 | Traceability traversal | 50 | Chain accuracy |
| KAT-004 | Dependency identification | 100 | Correct rate |
| KAT-005 | Conflict detection | 50 | Detection accuracy |
| KAT-006 | Architecture explanation | 50 | Rubric score |
| KAT-007 | Context production | 50 | Completeness |
| KAT-008 | Source identification | 50 | Source accuracy |
| KAT-009 | Evidence identification | 50 | Evidence accuracy |
| KAT-010 | Impact analysis | 50 | Impact accuracy |
| **Total** | | **650 questions** | |

---

## Cross-References

- Quality gate → `18-KNOWLEDGE-QUALITY-GATE`
- Reasoning certification → Phase 3.0D.0 `13-REASONING-CERTIFICATION`
- Hallucination targets → Phase 3.0D.0 `23-HALLUCINATION-CERTIFICATION`
