# KNW-KC-ARCH-011 — Quality Engine

**Phase:** 3.0C  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

The Quality Engine computes a composite quality score for every Knowledge Object across 9 orthogonal dimensions. Quality is the gating mechanism for lifecycle transitions and the primary signal for health dashboards.

---

## Quality Dimensions (9)

| Code | Dimension | Definition | Weight |
|------|-----------|------------|--------|
| QD-1 | **Completeness** | Fraction of required fields populated with non-default values | 0.15 |
| QD-2 | **Consistency** | Internal field coherence + relationship coherence | 0.20 |
| QD-3 | **Coverage** | Fraction of capabilities/requirements this object addresses | 0.10 |
| QD-4 | **Freshness** | Recency of last meaningful update relative to domain volatility | 0.15 |
| QD-5 | **Evidence** | evidence_score from Evidence Engine | 0.15 |
| QD-6 | **Confidence** | Owner-declared confidence × evidence weight | 0.10 |
| QD-7 | **Usage** | Frequency and breadth of references from other objects | 0.10 |
| QD-8 | **Reasoning** | Quality of stated rationale (decision objects) | 0.05 |
| QD-9 | **Health** | Relationship health — no dangling, stale, or disputed edges | —* |

*QD-9 is a derived dimension: `overall_score * health_penalty`. Not in the weighted sum.

**Weights sum: 1.00** (QD-1 through QD-8)

---

## Dimension Formulas

### QD-1 Completeness
```
completeness =
  populated_required_fields / total_required_fields
  * populated_optional_fields / total_optional_fields * 0.2  (bonus)
```

### QD-2 Consistency
```
consistency = 1.0 - (inconsistency_violations / total_checks)

Checks:
  - lifecycle.status consistent with evidence (see 10-EVIDENCE-ENGINE)
  - relationship targets exist and are not tombstoned
  - version is valid semver and >= parent version
  - approved_by non-empty iff status >= APPROVED
  - checksum matches content
```

### QD-3 Coverage
```
coverage = covered_requirements / total_applicable_requirements
         (0.0 if no requirements traceable — not penalised for unlinked objects)
```

### QD-4 Freshness
```
domain_volatility:
  ARCHITECTURE: stable — 365 day half-life
  MODULE: active — 90 day half-life
  TEST: volatile — 14 day half-life
  MARKET: real-time — 1 hour half-life
  FOREX: real-time — 1 minute half-life

freshness = exp(-0.693 * age_in_days / half_life_days)
```

### QD-5 Evidence
```
evidence = evidence_engine.compute_evidence_score(object_id)
```

### QD-6 Confidence
```
confidence = declared_confidence * evidence_score
```

### QD-7 Usage
```
usage = min(1.0, (inbound_relation_count / 10) * 0.6
              + (distinct_namespace_count / 5) * 0.4)

Where:
  inbound_relation_count  = number of other objects with relation.target_id == this
  distinct_namespace_count = number of distinct namespaces of those referencing objects
```

### QD-8 Reasoning (for DecisionObject only, else 1.0)
```
reasoning = min(1.0,
  options_considered_count * 0.2
  + rationale_length / 500.0 * 0.3
  + (1.0 if evidence_ids else 0.0) * 0.5
)
```

---

## Overall Score Formula

```
weighted_score =
  QD-1 * 0.15 +
  QD-2 * 0.20 +
  QD-3 * 0.10 +
  QD-4 * 0.15 +
  QD-5 * 0.15 +
  QD-6 * 0.10 +
  QD-7 * 0.10 +
  QD-8 * 0.05

health_penalty = 1.0 if QD-9 >= 0.8 else QD-9
overall_score  = weighted_score * health_penalty
```

---

## QD-9 Health Formula

```
health =
  1.0
  - 0.2 * (dangling_relation_count / max(1, total_relation_count))
  - 0.3 * (disputed_evidence_count / max(1, total_evidence_count))
  - 0.2 * (stale_relation_count / max(1, total_relation_count))
  - 0.3 * (broken_dependency_count / max(1, total_dependency_count))

Clamped to [0.0, 1.0]
```

---

## Quality Thresholds by Lifecycle Status

| Status | Required overall_score |
|--------|----------------------|
| DRAFT | ≥ 0.0 |
| REVIEW | ≥ 0.50 |
| VERIFIED | ≥ 0.65 |
| APPROVED | ≥ 0.75 |
| CANONICAL | ≥ 0.80 |

---

## Quality Score Record

```yaml
quality_score:
  knowledge_id: "KNW-KOS-ARCH-001"
  computed_at: datetime
  qd_1_completeness: 0.92
  qd_2_consistency: 0.88
  qd_3_coverage: 0.75
  qd_4_freshness: 0.95
  qd_5_evidence: 0.83
  qd_6_confidence: 0.78
  qd_7_usage: 0.60
  qd_8_reasoning: 1.00
  qd_9_health: 0.90
  weighted_score: 0.836
  health_penalty: 0.90
  overall_score: 0.752
  below_threshold: false
  alerts: []
```

---

## Quality Alerts

| Alert | Trigger Condition |
|-------|-----------------|
| `QA-001 LOW_COMPLETENESS` | QD-1 < 0.5 |
| `QA-002 INCONSISTENCY` | QD-2 < 0.6 |
| `QA-003 STALE` | QD-4 < 0.3 |
| `QA-004 LOW_EVIDENCE` | QD-5 < 0.4 |
| `QA-005 UNHEALTHY` | QD-9 < 0.6 |
| `QA-006 BELOW_STATUS_THRESHOLD` | overall_score < status.min_threshold |

---

## Quality Engine Protocol

```python
class KnowledgeQualityEngine(Protocol):
    def compute(self, object_id: str) -> QualityScoreRecord: ...
    def compute_dimension(self, object_id: str, dimension: QualityDimension) -> float: ...
    def check_threshold(self, object_id: str, target_status: LifecycleStatus) -> bool: ...
    def get_alerts(self, object_id: str) -> list[QualityAlert]: ...
    def get_trend(self, object_id: str, days: int) -> list[QualityScoreRecord]: ...
    def rank(self, namespace: str) -> list[tuple[str, float]]: ...
```

---

## Cross-References

- Evidence score feeds QD-5 → `10-EVIDENCE-ENGINE`
- Confidence model → `12-CONFIDENCE-MODEL`
- Lifecycle thresholds → `34-STATE-MACHINES`
- Performance budget → `37-PERFORMANCE-BUDGET`
