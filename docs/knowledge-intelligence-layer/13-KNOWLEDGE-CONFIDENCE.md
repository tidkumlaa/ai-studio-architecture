# KNW-KIL-DOC-013 — Knowledge Confidence

**Phase:** 3.0D.0.6
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Every Knowledge Object must express the confidence with which its claims can be
trusted. Confidence is distinct from quality: quality measures completeness and
consistency, confidence measures trustworthiness given the available evidence.

An object can be complete (high quality) but based on weak evidence (low
confidence). An object can have strong evidence (high confidence) but be
outdated (low freshness).

This document defines `intelligence.confidence`.

---

## Confidence Levels

| Level | Range | Meaning | AI Behavior |
|-------|-------|---------|-------------|
| VERY_HIGH | 0.90–1.00 | Proven by benchmark + approval + code | Assert directly |
| HIGH | 0.75–0.89 | Supported by test + documentation | Assert with citation |
| MEDIUM | 0.60–0.74 | Supported by documentation only | Assert with hedge |
| LOW | 0.40–0.59 | Based on proposal or design doc only | State as design intent |
| VERY_LOW | 0.00–0.39 | Inferred or assumed | State as uncertainty |

---

## Schema

```yaml
intelligence:
  confidence:
    schema_version: "1.0"
    computed_at: string                # ISO 8601

    overall_score: float               # 0.0–1.0 weighted composite
    level: string                      # VERY_LOW | LOW | MEDIUM | HIGH | VERY_HIGH

    dimensions:
      evidence_confidence:
        score: float                   # from evidence quality and level distribution
        weight: 0.35
        notes: string

      source_authority:
        score: float                   # how authoritative are the sources?
        weight: 0.25
        notes: string

      temporal_freshness:
        score: float                   # how recent is the evidence?
        weight: 0.20
        notes: string

      cross_validation:
        score: float                   # confirmed by multiple independent sources?
        weight: 0.15
        notes: string

      usage_confirmation:
        score: float                   # confirmed by real-world usage data?
        weight: 0.05
        notes: string

    formula: >
      confidence_score =
        (evidence_confidence × 0.35) +
        (source_authority × 0.25) +
        (temporal_freshness × 0.20) +
        (cross_validation × 0.15) +
        (usage_confirmation × 0.05)

    calibration:
      method: EMPIRICAL | EXPERT | CONSENSUS | ASSUMED
      calibrated_by: string
      last_calibration: string         # ISO 8601
      calibration_notes: string

    uncertainty:                       # structured uncertainty declaration
      - claim: string                  # what is uncertain
        uncertainty_type: UNKNOWN | CONFLICTING | OUTDATED | UNVERIFIED
        impact: string                 # how uncertainty affects this object's claims
        resolution: string            # how to resolve (or "accepted")

    hedging_rules:                     # how AI must hedge when confidence is low
      - confidence_threshold: float    # if overall_score < this
        required_hedge: string         # prefix AI must use
        forbidden_words: [string]      # words AI must not use at this confidence
```

---

## Hedging Rules by Level

| Level | Required Hedge | Forbidden Words |
|-------|---------------|-----------------|
| VERY_HIGH | (none required) | — |
| HIGH | "According to [source]," | — |
| MEDIUM | "Based on [doc], it appears that" | "definitely", "always", "never" |
| LOW | "The design intent is" | "is", "does", "will", "always", "guarantees" |
| VERY_LOW | "This is uncertain, but" | all strong assertions |

---

## Confidence vs Quality

```
Quality Score (QD-1 through QD-9):
  Measures: Is the object complete, consistent, and traceable?
  Source: Phase 3.0C quality scoring model
  Updates: When object content changes

Confidence Score (KIL):
  Measures: How trustworthy are the claims this object makes?
  Source: Evidence quality + authority + freshness + cross-validation + usage
  Updates: When evidence changes OR when usage data confirms/contradicts claims
```

---

## Canonical Example

```yaml
# For: KNW-PLT-MOD-001 (Quota Manager)
intelligence:
  confidence:
    schema_version: "1.0"
    computed_at: "2026-06-30T00:00:00Z"

    overall_score: 0.91
    level: VERY_HIGH

    dimensions:
      evidence_confidence:
        score: 0.94
        weight: 0.35
        notes: "EV-HAPPROVAL (L1) + EV-BENCHMARK (L3) + EV-TEST (L3) present"
      source_authority:
        score: 0.95
        weight: 0.25
        notes: "Architecture Board approval is highest authority (L1)"
      temporal_freshness:
        score: 0.88
        weight: 0.20
        notes: "Most recent evidence is 15 days old; within 30-day freshness window"
      cross_validation:
        score: 0.85
        weight: 0.15
        notes: "Claims confirmed by benchmark results and 3 independent test suites"
      usage_confirmation:
        score: 0.97
        weight: 0.05
        notes: "97.1% success rate across 14,827 usages"

    formula: >
      (0.94 × 0.35) + (0.95 × 0.25) + (0.88 × 0.20) + (0.85 × 0.15) + (0.97 × 0.05)
      = 0.329 + 0.2375 + 0.176 + 0.1275 + 0.0485
      = 0.918  → rounded to 0.91

    calibration:
      method: EMPIRICAL
      calibrated_by: "certification-suite v1.0"
      last_calibration: "2026-06-28"
      calibration_notes: "Calibrated against KAT-001 through KAT-010 results"

    uncertainty:
      - claim: "P99 < 5ms is maintained under all load patterns"
        uncertainty_type: UNKNOWN
        impact: "Benchmark covers nominal load; extreme spike scenarios untested"
        resolution: "Accepted — spike scenario test scheduled for Phase 3.0D.5"

    hedging_rules:
      - confidence_threshold: 0.60
        required_hedge: "Based on available documentation, it appears that"
        forbidden_words: ["definitely", "always", "guarantees", "ensures"]
      - confidence_threshold: 0.40
        required_hedge: "The design intent is (not yet empirically verified):"
        forbidden_words: ["is", "does", "will", "always", "never", "guarantees"]
```

---

## Rules

| Rule | Statement |
|------|-----------|
| KIL-076 | `confidence.overall_score` formula must use fixed weights; weights must sum to 1.0 |
| KIL-077 | Every `uncertainty` entry must have an `impact` — "none" is not acceptable |
| KIL-078 | AI agents MUST apply `hedging_rules` when generating responses |
| KIL-079 | `confidence.level` must be derived from `overall_score` using the level table — not manually set |
| KIL-080 | Objects with `level: VERY_LOW` must not be cited in arguments without explicit uncertainty declaration |
| KIL-081 | `calibration.method: ASSUMED` is valid only for objects in DRAFT or PROPOSED state |

---

## Cross-References

- Quality model → `14-KNOWLEDGE-QUALITY-MODEL`
- Evidence certification → Phase 3.0D.0 `11-EVIDENCE-CERTIFICATION`
- AI context (typical mistakes) → `04-AI-CONTEXT-LAYER`
- Hallucination prevention → Phase 3.0D.0 `23-HALLUCINATION-CERTIFICATION`
