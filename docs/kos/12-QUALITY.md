---
knowledge_id: KNW-KOS-ARCH-012
title: "KOS Knowledge Quality"
status: AUTHORITATIVE
phase: "3.0A"
version: "1.0.0"
created: "2026-06-30"
purpose: "Specify the Knowledge Quality scoring system: 8 dimensions, computation, thresholds, and alerting"
canonical_source: "architecture/docs/kos/12-QUALITY.md"
dependencies:
  - "04-OBJECT-MODEL.md"
  - "11-GOVERNANCE.md"
related_documents:
  - "13-LIFECYCLE.md"
  - "07-KNOWLEDGE-REGISTRY.md"
acceptance_criteria:
  - "All 9 quality dimensions are defined"
  - "Overall health score formula is specified"
  - "Threshold for CANONICAL eligibility is stated"
verification_checklist:
  - "[ ] All 9 scores defined with formula or criteria"
  - "[ ] Minimum thresholds per lifecycle status"
  - "[ ] Quality engine is automated (not manual)"
future_extensions:
  - "Machine learning-based quality prediction"
  - "Cross-object quality correlation"
---

# KOS Knowledge Quality

## Quality Engine

The Knowledge Quality Engine automatically scores every Knowledge Object on 9 dimensions.
Scores are computed at registration time and recomputed whenever the object or its
related objects change. No manual quality assessment.

---

## Quality Dimensions

### QD-1 — Knowledge Score

**Definition:** How completely is this object defined within KOS?  
**Formula:** `(filled_optional_fields / total_optional_fields) × 0.5 + (relationships_count / min_expected_relationships) × 0.5`  
**Range:** 0.0–1.0  
**Minimum for CANONICAL:** 0.70

### QD-2 — Coverage Score

**Definition:** What percentage of this object's acceptance criteria have linked tests?  
**Formula:** `linked_test_count / total_acceptance_criteria`  
**Range:** 0.0–1.0  
**Minimum for CANONICAL:** 0.80

### QD-3 — Confidence Score

**Definition:** How certain is the author/approver in this object's correctness?  
**Source:** `confidence` field on the KnowledgeObject (author-set, 0.0–1.0)  
**Range:** 0.0–1.0  
**Minimum for CANONICAL:** 0.70

### QD-4 — Completeness Score

**Definition:** Are all required governance fields present and non-empty?  
**Formula:** `required_fields_filled / total_required_fields`  
**Range:** 0.0–1.0  
**Minimum for CANONICAL:** 1.0 (all required fields must be present)

### QD-5 — Consistency Score

**Definition:** Does this object's content agree with its related objects?  
**Method:**
- Check that dependencies exist in the Registry
- Check that capability IDs claimed exist in Capability Registry
- Check that python_module paths follow import rules
- Check that version is > all older versions

**Formula:** `(consistent_checks / total_checks)`  
**Range:** 0.0–1.0  
**Minimum for CANONICAL:** 0.90

### QD-6 — Freshness Score

**Definition:** How recently was this object updated?  
**Formula:**

```python
age_days = (now - updated_at).days
if age_days <= 30:    freshness = 1.0
elif age_days <= 90:  freshness = 0.8
elif age_days <= 180: freshness = 0.6
elif age_days <= 365: freshness = 0.4
else:                 freshness = 0.2
```

**Range:** 0.0–1.0  
**Minimum for CANONICAL:** 0.60 (must be updated within 6 months)

### QD-7 — Evidence Score

**Definition:** Is the evidence field complete?  
**Method:**
- Presence: non-empty → 0.3
- Length: > 100 chars → +0.2
- References external source: → +0.2
- Contains alternatives section (for decisions): → +0.3

**Formula:** sum of above checks (capped at 1.0)  
**Range:** 0.0–1.0  
**Minimum for CANONICAL:** 0.50; for Decision objects: 0.80

### QD-8 — Reasoning Score

**Definition:** Can the Reasoning Engine answer all query types for this object?  
**Method:** Run R1–R9 queries for this object; count answered vs. failed  
**Formula:** `answered_queries / 9`  
**Range:** 0.0–1.0  
**Minimum for CANONICAL:** 0.60

### QD-9 — Overall Health

**Definition:** Composite weighted score across all 8 dimensions.  
**Formula:**

```python
overall_health = (
    QD_1 * 0.15 +  # knowledge score
    QD_2 * 0.20 +  # coverage
    QD_3 * 0.10 +  # confidence
    QD_4 * 0.15 +  # completeness
    QD_5 * 0.15 +  # consistency
    QD_6 * 0.10 +  # freshness
    QD_7 * 0.10 +  # evidence
    QD_8 * 0.05    # reasoning
)
```

**Range:** 0.0–1.0  
**Minimum for CANONICAL:** 0.80

---

## Quality Thresholds by Lifecycle Status

| Status | Min Overall Health | Notes |
|--------|-------------------|-------|
| DRAFT | No minimum | Work in progress |
| REVIEW | 0.60 | Must be reviewable |
| VERIFIED | 0.70 | Passed review |
| APPROVED | 0.75 | Approved but not yet canonical |
| CANONICAL | 0.80 | Production quality |
| DEPRECATED | N/A | No longer enforced |
| ARCHIVED | N/A | Historical record |

---

## Quality Score Storage

```python
@dataclass(frozen=True)
class KnowledgeQualityScore:
    knowledge_id: str
    knowledge_score: float       # QD-1
    coverage_score: float        # QD-2
    confidence_score: float      # QD-3
    completeness_score: float    # QD-4
    consistency_score: float     # QD-5
    freshness_score: float       # QD-6
    evidence_score: float        # QD-7
    reasoning_score: float       # QD-8
    overall_health: float        # QD-9
    computed_at: datetime
    failing_dimensions: list[str]
```

---

## Quality CLI

```bash
# Score a single object
platform-kos quality score KNW-AI-MOD-quota_manager

# Score all objects in a domain
platform-kos quality score --domain ai

# List all objects below threshold
platform-kos quality report --below 0.80

# Show quality dashboard summary
platform-kos quality dashboard

# Recompute all scores
platform-kos quality recompute --all
```

---

## Quality Dashboard Summary

```
platform-kos quality dashboard

╔══════════════════════════════════════════════════════╗
║            Knowledge Quality Dashboard               ║
╠══════════════════════════════════════════════════════╣
║ Domain  | Count | Avg Health | Below 0.80 | Critical ║
╠══════════════════════════════════════════════════════╣
║ ai      |   29  |   0.87     |    3       |    0     ║
║ platform|   15  |   0.91     |    1       |    0     ║
║ kos     |   20  |   0.72     |    8       |    2     ║
║ product |    4  |   0.65     |    2       |    1     ║
╠══════════════════════════════════════════════════════╣
║ Total   |   68  |   0.82     |   14       |    3     ║
╚══════════════════════════════════════════════════════╝

Critical objects (health < 0.50):
  KNW-PROD-001  health=0.43  missing: coverage, evidence
  KNW-KOS-ARCH-000  health=0.48  missing: tests linked
  ...
```

---

## Quality Alerts

| Condition | Alert |
|-----------|-------|
| CANONICAL object health drops below 0.80 | `platform.kos.quality.degraded` |
| CANONICAL object freshness drops below 0.60 | `platform.kos.quality.stale` |
| Any object completeness < 1.0 | `platform.kos.quality.incomplete` |
| Domain average health < 0.70 | `platform.kos.quality.domain_alert` |
