# KNW-CERT-ARCH-013 — Reasoning Certification

**Phase:** 3.0D.0  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Certifies the quality of AI-assisted reasoning over the Knowledge Graph. The reasoning engine must answer structured questions about the knowledge base with measurable accuracy, minimal hallucination, and appropriate confidence.

---

## Question Batches

| Batch | Questions | Seed | Level |
|-------|-----------|------|-------|
| R-500 | 500 | 42 | Bronze/Silver |
| R-1K | 1,000 | 42 | Gold |
| R-5K | 5,000 | 42 | Enterprise/Research |

---

## Question Categories

| Category | % of batch | Examples |
|----------|------------|---------|
| Identity | 10% | "What is KNW-PLT-MOD-001?" |
| Relationship | 15% | "What depends on Provider Registry?" |
| Dependency | 15% | "What breaks if Registry is removed?" |
| Architecture | 10% | "Which Runtime owns Execution Planner?" |
| Implementation | 10% | "What implements Routing Engine?" |
| API | 10% | "What API does the Quota Manager expose?" |
| Graph | 10% | "Which algorithm computes SCC?" |
| Capability | 5% | "Can the platform handle 1M objects?" |
| Lifecycle | 5% | "What is the status of BM25 Text Ranker?" |
| Evidence | 5% | "What is the evidence for Quota Manager performance?" |
| Impact | 5% | "What changes if I remove KNW-ALG-ALG-007?" |

---

## Answer Categories

Every question is classified into one of 4 answer categories:

| Category | Definition | Symbol |
|----------|------------|--------|
| Correct | Answer matches ground truth | ✓ |
| Incorrect | Wrong answer (factual error) | ✗ |
| Unknown | "I don't know" response | ? |
| Hallucination | Confident but wrong | ⚠ |

---

## Metrics

| Metric | Formula | Bronze | Silver | Gold | Enterprise | Research |
|--------|---------|--------|--------|------|------------|---------|
| Correct rate | correct / total | ≥ 0.65 | ≥ 0.73 | ≥ 0.82 | ≥ 0.89 | ≥ 0.95 |
| Incorrect rate | incorrect / total | ≤ 0.20 | ≤ 0.15 | ≤ 0.10 | ≤ 0.06 | ≤ 0.03 |
| Hallucination rate | hallucination / total | ≤ 0.15 | ≤ 0.10 | ≤ 0.05 | ≤ 0.02 | ≤ 0.01 |
| Confidence calibration | abs(confidence − correct_rate) | ≤ 0.15 | ≤ 0.10 | ≤ 0.07 | ≤ 0.05 | ≤ 0.03 |
| Reasoning depth (mean hops) | avg hops traced | ≥ 1 | ≥ 2 | ≥ 3 | ≥ 4 | ≥ 5 |
| Context completeness | pct questions with full context | ≥ 0.60 | ≥ 0.72 | ≥ 0.82 | ≥ 0.90 | ≥ 0.97 |

---

## Hallucination Definition

A response is classified as a **hallucination** when:
- The system provides a specific, confident answer (confidence ≥ 0.70)
- The answer does not correspond to any object in the knowledge base
- The referenced entity does not exist, or the stated relationship does not exist

Unknown responses (hedged, "I don't know") are not hallucinations.

---

## Checks

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Correct rate ≥ level target | RC-001 | CRITICAL | |
| Hallucination rate ≤ level target | RC-002 | CRITICAL | |
| All answers include source citation | RC-003 | MAJOR | Source knowledge_id present |
| Confidence score ∈ [0.0, 1.0] | RC-004 | MAJOR | |
| Confidence calibrated (abs diff ≤ level target) | RC-005 | MAJOR | |
| Each answer traces ≥ 1 graph hop | RC-006 | MAJOR | Reasoning chain logged |
| "What breaks if X removed?" = impact analysis | RC-007 | MAJOR | Uses dependency closure |
| "What implements X?" = registry lookup | RC-008 | MAJOR | Uses implements relationship |
| Category-specific accuracy ≥ 0.80 for each | RC-009 | MINOR | Per-category breakdown |

---

## Example Questions and Ground Truth

```yaml
- question: "What implements Routing Engine?"
  ground_truth:
    type: relationship
    relation: IMPLEMENTS
    source: KNW-RT-SVC-007
    target: KNW-PLT-MOD-004
  correct_answer: "KNW-RT-SVC-007 (AI Router Service) implements the Routing Engine (KNW-PLT-MOD-004)"

- question: "Which algorithm computes SCC?"
  ground_truth:
    type: identity
    answer: "Tarjan's SCC algorithm (KNW-ALG-ALG-009)"
  correct_answer: "Tarjan SCC Algorithm, KNW-ALG-ALG-009"

- question: "What breaks if Registry is removed?"
  ground_truth:
    type: impact
    impact_set: [KNW-RT-RT-001, KNW-PLT-MOD-003, KNW-API-API-002]
  correct_answer: "AI Runtime, Quota Manager, and Provider API all depend on the Registry"
```

---

## Report Format

```json
{
  "domain": "reasoning",
  "batch": "R-1K",
  "seed": 42,
  "results": {
    "correct": 847,
    "incorrect": 93,
    "unknown": 42,
    "hallucination": 18
  },
  "rates": {
    "correct": 0.847,
    "incorrect": 0.093,
    "hallucination": 0.018,
    "unknown": 0.042
  },
  "confidence_calibration": 0.063,
  "reasoning_depth_mean": 3.4,
  "context_completeness": 0.891,
  "by_category": {
    "identity": {"correct": 0.93},
    "relationship": {"correct": 0.86},
    "hallucination": {"correct": 0.81},
    "impact": {"correct": 0.79}
  },
  "domain_score": 0.841,
  "level_achieved": "Gold"
}
```

---

## CLI

```bash
kos-cert reasoning                       # R-500 batch
kos-cert reasoning --questions 1000
kos-cert reasoning --questions 5000 --seed 99
kos-cert reasoning --category impact     # impact questions only
kos-cert reasoning --output reports/reasoning.json
```

---

## Cross-References

- Hallucination detail → `23-HALLUCINATION-CERTIFICATION`
- AI context quality → `22-AI-CONTEXT-CERTIFICATION`
- Overall score weight (15%) → `27-OVERALL-SCORING`
