# KNW-CERT-ARCH-023 — Hallucination Certification

**Phase:** 3.0D.0  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Certifies that the KOS reasoning system minimises hallucination — the production of confident, specific claims about the knowledge base that are factually incorrect or refer to non-existent entities.

---

## Hallucination Definition

A response is classified as a **hallucination** when ALL of the following hold:

1. The system produces a specific, non-hedged answer
2. The answer confidence ≥ 0.70 (system is confident)
3. The answer is factually wrong OR references an entity that does not exist in the registry

A hedged response ("I'm not certain, but...") or an "unknown" response is NOT a hallucination.

---

## Hallucination Categories

| Category | Code | Definition |
|----------|------|------------|
| Non-existent entity | HAL-NE | References a knowledge_id or canonical_name that does not exist |
| Wrong relationship | HAL-WR | Claims A DEPENDS_ON B when the relationship does not exist |
| Wrong attribute | HAL-WA | States an incorrect field value (wrong status, owner, type) |
| Fabricated evidence | HAL-FE | Cites evidence that does not exist in the knowledge base |
| Wrong algorithm | HAL-WLG | Names the wrong algorithm for a computation |
| Invented chain | HAL-IC | Fabricates a traceability chain with non-existent links |

---

## Test Procedure

1. Generate 1,000 questions from the question bank (R-1K batch, seed 42)
2. For each answer, classify as: Correct / Incorrect / Unknown / Hallucination
3. For each hallucination, classify by category (HAL-NE, HAL-WR, etc.)
4. Compute rates and per-category breakdown

---

## Hallucination Targets

| Metric | Bronze | Silver | Gold | Enterprise | Research |
|--------|--------|--------|------|------------|---------|
| Overall hallucination rate | ≤ 0.15 | ≤ 0.10 | ≤ 0.05 | ≤ 0.02 | ≤ 0.01 |
| HAL-NE (non-existent entity) | ≤ 0.10 | ≤ 0.06 | ≤ 0.02 | ≤ 0.01 | ≤ 0.005 |
| HAL-FE (fabricated evidence) | ≤ 0.08 | ≤ 0.05 | ≤ 0.02 | ≤ 0.01 | ≤ 0.005 |
| HAL-IC (invented chain) | ≤ 0.05 | ≤ 0.03 | ≤ 0.01 | ≤ 0.005 | ≤ 0.002 |

---

## Hallucination Checks

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Overall rate ≤ level target | HC-001 | CRITICAL | |
| HAL-NE rate ≤ level target | HC-002 | CRITICAL | |
| HAL-FE rate ≤ level target | HC-003 | MAJOR | |
| HAL-IC rate ≤ level target | HC-004 | MAJOR | |
| Every non-hedged answer cites source | HC-005 | MAJOR | Source knowledge_id present |
| Citation exists in registry | HC-006 | MAJOR | All cited IDs resolve |
| High-confidence wrong answers ≤ 3% | HC-007 | MAJOR | confidence ≥ 0.80 but wrong |
| Unknown rate ≤ 20% | HC-008 | MINOR | System shouldn't over-hedge |

---

## Mitigation Protocol

The KOS reasoning engine MUST implement the following hallucination mitigations:

| Mitigation | Requirement |
|------------|-------------|
| Grounding | Every specific claim must cite a knowledge_id |
| Existence check | Before citing KNW-X, verify KNW-X exists in registry |
| Confidence calibration | Confidence must reflect actual correctness rate |
| Hedge threshold | If confidence < 0.70, response must be hedged |
| Unknown response | If no relevant object found, return "unknown" |

These are architectural requirements, not implementation — the certification verifies their effect, not their mechanism.

---

## Example Classifications

```
Q: "What implements Routing Engine?"
A: "The AI Router (KNW-RT-SVC-007) implements the Routing Engine."
   KNW-RT-SVC-007 exists, relationship exists → CORRECT

Q: "What implements Routing Engine?"
A: "The Dynamic Router (KNW-RT-SVC-999) implements the Routing Engine."
   KNW-RT-SVC-999 does NOT exist → HAL-NE

Q: "What is the evidence for Quota Manager's performance?"
A: "Benchmark result EV-BENCH-999 shows P99 < 5ms."
   EV-BENCH-999 does NOT exist → HAL-FE

Q: "What implements Routing Engine?"
A: "I'm not certain, but it may be the Router Service."
   Hedged → UNKNOWN (not hallucination)
```

---

## Report Format

```json
{
  "domain": "hallucination",
  "questions_tested": 1000,
  "results": {
    "correct": 847,
    "incorrect_non_hallucination": 75,
    "unknown": 42,
    "hallucination": 36
  },
  "hallucination_rate": 0.036,
  "by_category": {
    "HAL-NE": 14,
    "HAL-WR": 9,
    "HAL-WA": 7,
    "HAL-FE": 4,
    "HAL-WLG": 2,
    "HAL-IC": 0
  },
  "checks": {
    "HC-001": "PASS", "HC-002": "PASS", "HC-005": "PASS",
    "HC-006": "PASS", "HC-007": "PASS"
  },
  "domain_score": 0.891,
  "level_achieved": "Gold"
}
```

---

## CLI

```bash
kos-cert hallucination                   # R-1K batch
kos-cert hallucination --questions 5000
kos-cert hallucination --category HAL-NE # specific category only
kos-cert hallucination --output reports/hallucination.json
```

---

## Cross-References

- Reasoning overall → `13-REASONING-CERTIFICATION`
- Context completeness (root cause of HAL-NE) → `22-AI-CONTEXT-CERTIFICATION`
- Evidence correctness (root cause of HAL-FE) → `11-EVIDENCE-CERTIFICATION`
