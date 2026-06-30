# KVF-DOC-027 — Certification Scoring

**Phase:** 3.0D.1.5
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Certification Scoring defines how KOS Runtime validation results translate
into a certification level. There are 5 levels: Bronze, Silver, Gold,
Enterprise, and Research. Each level has specific requirements across all
7 validation domains.

---

## 5 Certification Levels

```
BRONZE:   Minimum viable KOS implementation
          Suitable for: prototypes, internal demos, development environments

SILVER:   Standard production-ready implementation
          Suitable for: team-level production use, internal products

GOLD:     High-quality production implementation
          Suitable for: company-wide production, external products

ENTERPRISE: Enterprise-grade implementation
            Suitable for: regulated industries, critical production systems

RESEARCH: Highest possible level
          Suitable for: AI training data, academic benchmarks, reference implementations
```

---

## Certification Requirements Matrix

### Domain 1: Conformance

| Metric | BRONZE | SILVER | GOLD | ENTERPRISE | RESEARCH |
|--------|--------|--------|------|------------|----------|
| Knowledge conformance score | ≥ 0.90 | ≥ 0.95 | ≥ 0.99 | ≥ 0.999 | 1.000 |
| Runtime conformance score | ≥ 34/40 | ≥ 37/40 | ≥ 39/40 | 40/40 | 40/40 |
| No critical conformance failures | YES | YES | YES | YES | YES |

### Domain 2: Context Quality

| Metric | BRONZE | SILVER | GOLD | ENTERPRISE | RESEARCH |
|--------|--------|--------|------|------------|----------|
| Mean context quality score | ≥ 0.70 | ≥ 0.75 | ≥ 0.80 | ≥ 0.85 | ≥ 0.90 |
| Safety score minimum | ≥ 0.95 | ≥ 0.97 | ≥ 0.99 | ≥ 0.999 | 1.000 |
| Token reduction (mean) | ≥ 80% | ≥ 90% | ≥ 95% | ≥ 97% | ≥ 99% |

### Domain 3: Knowledge Coverage

| Metric | BRONZE | SILVER | GOLD | ENTERPRISE | RESEARCH |
|--------|--------|--------|------|------------|----------|
| Mean coverage (recall) | ≥ 0.65 | ≥ 0.75 | ≥ 0.85 | ≥ 0.90 | ≥ 0.95 |
| Primary correct rate | ≥ 0.95 | ≥ 0.98 | ≥ 0.99 | ≥ 0.999 | 1.000 |
| Traceability coverage | ≥ 0.60 | ≥ 0.70 | ≥ 0.80 | ≥ 0.90 | ≥ 0.95 |

### Domain 4: Search and Reasoning

| Metric | BRONZE | SILVER | GOLD | ENTERPRISE | RESEARCH |
|--------|--------|--------|------|------------|----------|
| Search Top-1 accuracy | ≥ 0.70 | ≥ 0.80 | ≥ 0.85 | ≥ 0.90 | ≥ 0.95 |
| MRR | ≥ 0.70 | ≥ 0.80 | ≥ 0.85 | ≥ 0.90 | ≥ 0.95 |
| Reasoning correct rate | ≥ 0.65 | ≥ 0.75 | ≥ 0.85 | ≥ 0.90 | ≥ 0.95 |
| Critical hallucination (A+B) rate | < 0.10 | < 0.05 | < 0.02 | < 0.01 | < 0.005 |

### Domain 5: AI Capability

| Metric | BRONZE | SILVER | GOLD | ENTERPRISE | RESEARCH |
|--------|--------|--------|------|------------|----------|
| AI understanding success rate | ≥ 0.65 | ≥ 0.75 | ≥ 0.85 | ≥ 0.90 | ≥ 0.95 |
| Context accuracy | ≥ 0.80 | ≥ 0.85 | ≥ 0.90 | ≥ 0.95 | ≥ 0.98 |
| Cross-agent fact consistency | ≥ 0.85 | ≥ 0.90 | ≥ 0.95 | ≥ 0.97 | ≥ 0.99 |

### Domain 6: Performance

| Metric | BRONZE | SILVER | GOLD | ENTERPRISE | RESEARCH |
|--------|--------|--------|------|------------|----------|
| Assembly P99 | < 200ms | < 150ms | < 120ms | < 100ms | < 80ms |
| Assembly P50 | < 100ms | < 70ms | < 50ms | < 40ms | < 30ms |
| Memory leak | None | None | None | None | None |
| 1K object P99 | < 80ms | < 60ms | < 50ms | < 40ms | < 30ms |
| 10K object P99 | < 120ms | < 100ms | < 80ms | < 70ms | < 60ms |

### Domain 7: Certification Checks

| Metric | BRONZE | SILVER | GOLD | ENTERPRISE | RESEARCH |
|--------|--------|--------|------|------------|----------|
| CAE certification checks (of 50) | ≥ 30 | ≥ 40 | ≥ 47 | ≥ 49 | 50 |
| No CRITICAL check failures | YES | YES | YES | YES | YES |

---

## Certification Score Formula

```
Each domain contributes a weighted score to the overall certification score:

certification_score =
  conformance         × 0.20 +
  context_quality     × 0.20 +
  knowledge_coverage  × 0.15 +
  search_reasoning    × 0.20 +
  ai_capability       × 0.15 +
  performance         × 0.10

Minimum certification score by level:
  BRONZE:     ≥ 0.60
  SILVER:     ≥ 0.72
  GOLD:       ≥ 0.83
  ENTERPRISE: ≥ 0.91
  RESEARCH:   ≥ 0.97

NOTE: Score alone is not sufficient. All hard requirements
(NO critical failures, specific per-metric thresholds) must also be met.
A score of 0.95 with one critical failure = BRONZE maximum.
```

---

## Hard Gates

The following are binary gates — fail any one and the level is blocked:

```
For ALL levels:
  - Knowledge conformance checks: no CG-1 or CG-2 critical failures
  - Memory leak: none detected

For SILVER+:
  - AI capability: task scale STANDARD (500 tasks) must be run
  - Regression: must compare against a valid baseline

For GOLD+:
  - CAE certification checks: all 20 CRITICAL checks must pass
  - Hallucination Type A rate: < 2%
  - Hallucination Type B rate: < 2%

For ENTERPRISE+:
  - AI understanding scale: FULL (1,000 tasks)
  - Scalability: must test 100K tier
  - Performance: 1-hour stability test passed

For RESEARCH:
  - Scalability: 1M tier tested
  - All 50 CAE certification checks passed
  - All 50 KVF metrics within RESEARCH targets
```

---

## Certification Report Schema

```yaml
certification_report:
  report_id: string
  issued_at: string
  valid_until: string                   # 1 year from issuance
  implementation_id: string
  implementation_version: string
  kos_schema_version: string
  kvf_version: string

  level: BRONZE | SILVER | GOLD | ENTERPRISE | RESEARCH | NONE
  certification_score: float

  domain_scores:
    conformance: float
    context_quality: float
    knowledge_coverage: float
    search_reasoning: float
    ai_capability: float
    performance: float

  hard_gates:
    - gate: string
      passed: boolean

  level_justification: string          # why this level was awarded/denied

  certifier: string
  auditor: string | null               # required for ENTERPRISE+
```

---

## Rules

| Rule | Statement |
|------|-----------|
| KVF-131 | Certification score formula weights must sum to exactly 1.00 |
| KVF-132 | Hard gates are binary — no partial credit for failing a hard gate |
| KVF-133 | Certification is valid for 1 year; after 1 year, re-validation required |
| KVF-134 | ENTERPRISE and RESEARCH certifications require independent auditor review |
| KVF-135 | Certification level must match the LOWEST level met across all hard gates |

---

## Cross-References

- All domain validation docs → docs 02–26
- CAE certification → Phase 3.0D.1 `26-CAE-CERTIFICATION`
- Dashboard → `28-DASHBOARD`
- Architecture freeze → `29-ARCHITECTURE-FREEZE`
