---
knowledge_id: KA-PLAT-010
version: "1.0.0"
status: approved
owner: Chief Knowledge Architect
phase: 2.0D.1
created: 2026-06-29
review_date: 2026-12-29
canonical: true
domain: DOM-KNOWLEDGE
capability: migration-platform
type: specification
depends_on:
  - id: KA-STD-006
    reason: "Health metrics specification defines 6-dimension scoring"
  - id: KA-SPEC-017
    reason: "ALG-007 defines health score computation algorithm"
---

# Health Analyzer

## 6-Dimension Health Scoring and Knowledge Debt Calculation

---

## 1. Purpose

The HealthAnalyzer implements ALG-007 from KA-SPEC-017 to compute document health
scores across six weighted dimensions. It produces a `HealthReport` with per-document
scores, capability aggregates, and a repository-level score with a knowledge debt
estimate.

---

## 2. Scoring Dimensions

Exact weights and rules from KA-STD-006:

| Dimension | Weight | Max Points |
|-----------|--------|-----------|
| Structural | 25% | 25 |
| Relationship | 20% | 20 |
| Freshness | 20% | 20 |
| Evidence | 15% | 15 |
| Coverage | 10% | 10 |
| Consistency | 10% | 10 |
| **Total** | 100% | **100** |

---

## 3. Structural Scoring (25 points)

| Field Present | Points |
|--------------|--------|
| `knowledge_id` | 5 |
| `version` | 2 |
| `status` | 2 |
| `canonical` | 2 |
| `domain` | 3 |
| `capability` | 3 |
| `type` | 3 |
| `owner` | 2 |
| `review_date` | 3 |
| **Total** | **25** |

A document with no frontmatter scores 0/25 on structural.

---

## 4. Relationship Scoring (20 points)

| Condition | Points |
|-----------|--------|
| At least 1 `implements` or `implemented_by` | 5 |
| At least 1 `depends_on` | 5 |
| At least 1 `related_to` or `references` | 5 |
| No broken links | 5 |
| **Total** | **20** |

---

## 5. Freshness Scoring (20 points)

| Days Since Review | Points |
|-------------------|--------|
| 0–30 | 20 |
| 31–60 | 14 |
| 61–90 | 8 |
| 91–180 | 4 |
| 181–365 | 1 |
| 365+ | 0 |
| No review_date | 0 |

---

## 6. Evidence Scoring (15 points)

| Condition | Points |
|-----------|--------|
| Evidence field exists | 5 |
| At least 1 evidence item is `validated` or `verified` | 5 |
| Evidence verified within 90 days | 5 |
| **Total** | **15** |

For the migration phase, evidence is typically absent. Most documents score 0/15
until evidence is added during Phase 2.0D.2.

---

## 7. Coverage Scoring (10 points)

| Condition | Points |
|-----------|--------|
| Architecture coverage declared > 0 | 3 |
| Architecture coverage > 50% | 1 |
| Implementation coverage declared > 0 | 3 |
| Testing coverage declared > 0 | 3 |
| **Total** | **10** |

---

## 8. Consistency Scoring (10 points)

Start at 10, deduct:

| Violation | Deduction |
|-----------|-----------|
| `status: approved` but `review_date` in past | -3 |
| `canonical: true` but another canonical exists for same (cap, type) | -5 |
| `version` does not match semver pattern | -2 |
| `knowledge_id` format invalid | -3 |

Minimum 0.

---

## 9. Repository Health Score

```
REPO_SCORE(all_document_scores):
  IF len == 0: RETURN 0
  avg = mean(all_document_scores)
  
  # Penalty factors (KA-STD-006)
  orphan_ratio   = orphan_count / total_count
  broken_ratio   = broken_link_count / total_link_count
  no_meta_ratio  = no_metadata_count / total_count
  
  penalties = (
    orphan_ratio  * 10 +
    broken_ratio  * 15 +
    no_meta_ratio * 20
  )
  
  RETURN max(0, avg - penalties)
```

---

## 10. Knowledge Debt

Knowledge debt estimates the remediation effort:

```yaml
KnowledgeDebt:
  total_score: float             # Repository health score (0-100)
  documents_without_metadata: int
  documents_oversized: int
  broken_links: int
  orphan_documents: int
  missing_evidence: int
  stale_reviews: int

  estimated_remediation_days: float
  # = (no_metadata * 0.1) + (oversized * 0.5) + (broken_links * 0.05)
  #   + (orphans * 0.25) + (missing_evidence * 0.1)
```

---

## 11. Health Report Structure

```yaml
HealthReport:
  generated: "2026-06-29"
  repository_score: 15.2
  knowledge_debt:
    estimated_remediation_days: 38.4
  
  thresholds:
    critical:  0-24
    poor:      25-49
    fair:      50-69
    good:      70-84
    excellent: 85-100
  
  by_document:
    - path: "docs/workflow/architecture.md"
      total: 72
      structural: 25
      relationship: 15
      freshness: 20
      evidence: 0
      coverage: 7
      consistency: 5
      issues:
        - "Missing evidence"
        - "Coverage not declared for testing"
  
  by_capability:
    - capability: "workflow-runtime"
      avg_score: 58.3
      document_count: 7
      min_score: 32
      max_score: 72
  
  phase_gate:
    required_score: 80
    current_score: 15.2
    gap: 64.8
    status: "FAIL"
```

---

## References

- [KA-STD-006](../knowledge-architecture/KNOWLEDGE-HEALTH-METRICS.md) — Health metrics definition
- [KA-SPEC-017](../knowledge-core/17-ALGORITHMS-CATALOG.md) — ALG-007 health score algorithm
- [10-MIGRATION-PLANNER.md](10-MIGRATION-PLANNER.md) — Consumes HealthReport for planning
