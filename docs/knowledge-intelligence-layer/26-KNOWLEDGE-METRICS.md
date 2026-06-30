# KNW-KIL-DOC-026 — Knowledge Metrics

**Phase:** 3.0D.0.6
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Defines all 40 Knowledge Intelligence Layer metrics — their IDs, formulas,
targets, collection frequency, and alert thresholds.

---

## Metric Catalog

### Block Completeness Metrics (KIM-001–KIM-010)

| ID | Name | Formula | Target | Alert |
|----|------|---------|--------|-------|
| KIM-001 | Self-Describing Completeness | objects with complete self_describing / total CANONICAL | 1.00 | < 0.95 |
| KIM-002 | Executable Completeness | objects with all rules machine-readable / total CANONICAL | 1.00 | < 0.95 |
| KIM-003 | AI Context Completeness | objects with short+medium+full summaries / total CANONICAL | 1.00 | < 0.98 |
| KIM-004 | Genome Completeness | objects with all 8 genome fields / total CANONICAL | 1.00 | < 0.98 |
| KIM-005 | DNA Seal Rate | objects with sealed + hash-valid DNA / total CANONICAL | 1.00 | < 1.00 |
| KIM-006 | Reasoning Completeness | objects with requires+provides+enables / total CANONICAL | 0.95 | < 0.80 |
| KIM-007 | Cortex Completeness | objects with why+why_not+≥3 common_questions / total CANONICAL | 0.95 | < 0.85 |
| KIM-008 | Tradeoff Coverage | objects with ≥1 tradeoff / MODULE+SERVICE count | 0.90 | < 0.70 |
| KIM-009 | Decision Coverage | objects with ≥1 decision / MODULE+SERVICE count | 0.90 | < 0.70 |
| KIM-010 | Alternative Coverage | objects with ≥1 alternative / total CANONICAL | 0.80 | < 0.60 |

---

### AI Readiness Metrics (KIM-011–KIM-015)

| ID | Name | Formula | Target | Alert |
|----|------|---------|--------|-------|
| KIM-011 | Average AI Readiness Score | mean(ai_readiness_score) across CANONICAL objects | ≥ 0.80 | < 0.75 |
| KIM-012 | AI-Ready Object Rate | count(ai_readiness_score ≥ 0.80) / total CANONICAL | ≥ 0.90 | < 0.80 |
| KIM-013 | AI Readiness P10 | 10th percentile of ai_readiness_score | ≥ 0.60 | < 0.50 |
| KIM-014 | AI Readiness Trend | delta(avg_ai_readiness, 30d) | ≥ 0 | < -0.05 |
| KIM-015 | AI Context Token Budget Compliance | summaries within budget / total | 1.00 | < 0.99 |

---

### Confidence Metrics (KIM-016–KIM-020)

| ID | Name | Formula | Target | Alert |
|----|------|---------|--------|-------|
| KIM-016 | Average Confidence Score | mean(confidence.overall_score) across CANONICAL | ≥ 0.75 | < 0.65 |
| KIM-017 | HIGH+ Confidence Rate | count(confidence.level ∈ {HIGH, VERY_HIGH}) / total | ≥ 0.80 | < 0.70 |
| KIM-018 | Very Low Confidence Objects | count(confidence.level = VERY_LOW) | 0 | > 5 |
| KIM-019 | Uncalibrated Confidence | count(confidence.calibration.method = ASSUMED) for CANONICAL objects | 0 | > 0 |
| KIM-020 | Confidence vs Quality Alignment | correlation(confidence_score, quality_score) | ≥ 0.70 | < 0.50 |

---

### Quality Intelligence Metrics (KIM-021–KIM-025)

| ID | Name | Formula | Target | Alert |
|----|------|---------|--------|-------|
| KIM-021 | KIL Overall Score Average | mean(intelligence.quality.kil_overall_score) | ≥ 0.80 | < 0.75 |
| KIM-022 | Critical Gap Count | count(objects with critical_gaps non-empty) | 0 | > 0 |
| KIM-023 | Improvement Plan Coverage | count(objects with improvement_plan) / total CANONICAL | ≥ 0.50 | < 0.30 |
| KIM-024 | Self-Describing Quality Score | mean(self_describing_score across all CANONICAL) | ≥ 0.90 | < 0.80 |
| KIM-025 | Executable Rules Coverage | mean(executable_score across all CANONICAL) | ≥ 0.90 | < 0.80 |

---

### Memory & Usage Metrics (KIM-026–KIM-030)

| ID | Name | Formula | Target | Alert |
|----|------|---------|--------|-------|
| KIM-026 | Unused Object Rate | count(memory.total_accesses = 0, age > 90d) / total | 0 | > 0.05 |
| KIM-027 | Global Success Rate | mean(memory.success_rate.rate) | ≥ 0.95 | < 0.90 |
| KIM-028 | Failure Category Coverage | count(failures without typical_mistake entry) | 0 | > 0 |
| KIM-029 | Usage Pattern Coverage | count(objects with ≥2 access_patterns) / total CANONICAL | ≥ 0.70 | < 0.50 |
| KIM-030 | Co-occurrence Data Age | days since co_occurrence_matrix last computed | ≤ 7 | > 14 |

---

### Risk Metrics (KIM-031–KIM-035)

| ID | Name | Formula | Target | Alert |
|----|------|---------|--------|-------|
| KIM-031 | Unmitigated CRITICAL Security Vectors | count(exposure_vectors WHERE impact=CRITICAL AND mitigated=false) | 0 | > 0 |
| KIM-032 | CRITICAL Risk Objects | count(risk_tier = CRITICAL) | ≤ 10 | > 15 |
| KIM-033 | Failure Mode Coverage | count(failure_modes with detection defined) / total failure_modes | 1.00 | < 0.95 |
| KIM-034 | Single Point of Failure Count | count(criticality.single_point_of_failure = true) | ≤ 3 | > 5 |
| KIM-035 | Transitive Risk Propagation Depth | max depth of risk propagation graph | ≤ 5 | > 8 |

---

### Learning Metrics (KIM-036–KIM-040)

| ID | Name | Formula | Target | Alert |
|----|------|---------|--------|-------|
| KIM-036 | Prerequisite Coverage | count(objects with ≥1 prerequisite) / total CANONICAL | ≥ 0.80 | < 0.60 |
| KIM-037 | Knowledge Gap Documentation | count(objects with knowledge_gaps defined) / total CANONICAL | ≥ 0.70 | < 0.50 |
| KIM-038 | AI Agent Correct Response Rate | mean(ai_learning.correct_response_rate) | ≥ 0.95 | < 0.90 |
| KIM-039 | Agent Improvement Rate | count(ai_learning.improvement_over_time > 0) / total | ≥ 0.80 | < 0.60 |
| KIM-040 | Hardest Concept Coverage | count(ai_learning.hardest_concept defined) / total CANONICAL | ≥ 0.90 | < 0.70 |

---

## Metric Collection Schedule

| Frequency | Metrics |
|-----------|---------|
| Real-time | KIM-005 (DNA integrity), KIM-031 (security) |
| Per-access | KIM-026–028 (usage/memory) |
| Hourly | KIM-011–015 (AI readiness) |
| Daily | KIM-001–010 (completeness), KIM-016–025 (quality/confidence) |
| Weekly | KIM-029–030 (co-occurrence), KIM-036–040 (learning) |
| Nightly | KIM-031–035 (risk) |

---

## Cross-References

- Intelligence dashboard → `25-KNOWLEDGE-INTELLIGENCE-DASHBOARD`
- AI readiness score → `27-KNOWLEDGE-AI-READINESS`
- Quality model → `14-KNOWLEDGE-QUALITY-MODEL`
- Risk model → `15-KNOWLEDGE-RISK-MODEL`
