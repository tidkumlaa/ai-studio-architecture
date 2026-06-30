# KNW-KIL-DOC-014 — Knowledge Quality Model

**Phase:** 3.0D.0.6
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The KOS v1.0 quality scoring model (Phase 3.0C QD-1 through QD-9) produces an
aggregate quality score. The Intelligence Layer extends this with a per-object
quality profile — a detailed breakdown showing exactly where an object excels,
where it is weak, and what must change to improve.

This document defines `intelligence.quality`.

---

## Schema

```yaml
intelligence:
  quality:
    schema_version: "1.0"
    computed_at: string

    # Inherited from Phase 3.0C quality model (Phase 3.0C QD-1..QD-9)
    overall_score: float               # the existing quality_score
    health_multiplier: float           # QD-9 health (0.0–1.0 × everything)

    # Per-dimension scores
    dimensions:
      completeness:                    # QD-1 weight: 0.25
        score: float
        weight: 0.25
        missing_fields: [string]       # which required fields are absent
        completion_percentage: float

      consistency:                     # QD-2 weight: 0.15
        score: float
        weight: 0.15
        inconsistencies: [string]      # detected inconsistencies

      evidence:                        # QD-3 weight: 0.20
        score: float
        weight: 0.20
        item_count: integer
        level_distribution: {}
        oldest_item_days: integer

      freshness:                       # QD-4 weight: 0.10
        score: float
        weight: 0.10
        last_updated: string
        staleness_days: integer
        freshness_rating: FRESH | AGING | STALE | CRITICAL

      confidence:                      # QD-5 weight: 0.10
        score: float
        weight: 0.10
        confidence_level: string

      traceability:                    # QD-6 weight: 0.10
        score: float
        weight: 0.10
        traced_links: integer
        untraced_links: integer
        traceability_percentage: float

      usage:                           # QD-7 weight: 0.05
        score: float
        weight: 0.05
        access_count: integer
        success_rate: float

      coverage:                        # QD-8 weight: 0.05
        score: float
        weight: 0.05
        covered_dimensions: integer
        total_dimensions: integer

      health:                          # QD-9: multiplier
        score: float                   # 0.0–1.0 (applied as multiplier to all)
        open_issues: integer
        critical_issues: integer
        health_factors: [string]

    intelligence_quality:              # KIL-specific quality dimensions
      self_describing_score: float     # completeness of intelligence.self_describing
      executable_score: float          # completeness of intelligence.executable
      ai_context_score: float          # completeness of intelligence.ai_context
      genome_score: float              # completeness of intelligence.genome
      dna_score: float                 # DNA sealed and valid?
      reasoning_score: float           # completeness of intelligence.reasoning
      kil_overall_score: float         # composite of above

    quality_profile:                   # strengths and weaknesses
      strengths: [string]              # dimensions where score ≥ 0.85
      weaknesses: [string]             # dimensions where score < 0.70
      critical_gaps: [string]          # dimensions at 0 that are required

    improvement_plan:                  # ordered actions to improve quality
      - rank: integer
        action: string
        expected_improvement: float    # expected score delta
        effort: TRIVIAL | SMALL | MEDIUM | LARGE
        dimension: string
```

---

## Quality Score Formula (unchanged from Phase 3.0C)

```
quality_score =
  (QD1.completeness  × 0.25) +
  (QD2.consistency   × 0.15) +
  (QD3.evidence      × 0.20) +
  (QD4.freshness     × 0.10) +
  (QD5.confidence    × 0.10) +
  (QD6.traceability  × 0.10) +
  (QD7.usage         × 0.05) +
  (QD8.coverage      × 0.05)
  × QD9.health_multiplier
```

The intelligence block adds `kil_overall_score` as an additional metric but
does NOT modify the existing `metadata.quality_score` computation.

---

## KIL Quality Dimensions

| Dimension | What It Measures | Target |
|-----------|-----------------|--------|
| self_describing_score | All 11 fields complete | 1.0 |
| executable_score | All rules machine-readable | 1.0 |
| ai_context_score | short/medium/full summaries present | 1.0 |
| genome_score | All 8 genome fields complete | 1.0 |
| dna_score | DNA sealed + hash valid | 1.0 |
| reasoning_score | requires/provides/enables populated | 0.8 |
| kil_overall_score | Weighted average of above | ≥ 0.8 |

---

## Canonical Example

```yaml
# For: KNW-PLT-MOD-001
intelligence:
  quality:
    schema_version: "1.0"
    computed_at: "2026-06-30T00:00:00Z"

    overall_score: 0.87
    health_multiplier: 1.0

    dimensions:
      completeness:
        score: 0.95
        weight: 0.25
        missing_fields: []
        completion_percentage: 0.97
      consistency:
        score: 0.90
        weight: 0.15
        inconsistencies: []
      evidence:
        score: 0.88
        weight: 0.20
        item_count: 5
        level_distribution: {1: 1, 2: 1, 3: 2, 4: 1}
        oldest_item_days: 45
      freshness:
        score: 0.85
        weight: 0.10
        last_updated: "2026-06-15"
        staleness_days: 15
        freshness_rating: FRESH
      confidence:
        score: 0.91
        weight: 0.10
        confidence_level: VERY_HIGH
      traceability:
        score: 0.92
        weight: 0.10
        traced_links: 12
        untraced_links: 1
        traceability_percentage: 0.923
      usage:
        score: 0.97
        weight: 0.05
        access_count: 14827
        success_rate: 0.971
      coverage:
        score: 0.90
        weight: 0.05
        covered_dimensions: 9
        total_dimensions: 10
      health:
        score: 1.0
        open_issues: 0
        critical_issues: 0
        health_factors: []

    intelligence_quality:
      self_describing_score: 1.0
      executable_score: 0.95
      ai_context_score: 1.0
      genome_score: 1.0
      dna_score: 1.0
      reasoning_score: 0.90
      kil_overall_score: 0.975

    quality_profile:
      strengths:
        - "completeness (0.95)"
        - "confidence (0.91)"
        - "traceability (0.92)"
        - "usage (0.97)"
      weaknesses: []
      critical_gaps: []

    improvement_plan:
      - rank: 1
        action: "Add missing traceability link to KNW-PLT-SVC-003"
        expected_improvement: 0.005
        effort: TRIVIAL
        dimension: "traceability"
      - rank: 2
        action: "Refresh oldest evidence item (45 days old)"
        expected_improvement: 0.015
        effort: SMALL
        dimension: "evidence"
```

---

## Rules

| Rule | Statement |
|------|-----------|
| KIL-082 | `intelligence.quality.overall_score` must equal `metadata.quality_score` exactly |
| KIL-083 | `kil_overall_score` is a separate metric and must NOT modify `metadata.quality_score` |
| KIL-084 | Every CANONICAL object must have `kil_overall_score ≥ 0.80` |
| KIL-085 | `improvement_plan` must be regenerated whenever any dimension score changes |
| KIL-086 | `critical_gaps` block CANONICAL promotion until resolved |

---

## Cross-References

- Quality dimensions → Phase 3.0C `08-QUALITY-SCORING`
- Confidence score → `13-KNOWLEDGE-CONFIDENCE`
- Risk model → `15-KNOWLEDGE-RISK-MODEL`
- Quality gate → Phase 3.0D.0.5 `18-KNOWLEDGE-QUALITY-GATE`
