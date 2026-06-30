# KNW-CERT-ARCH-027 — Overall Scoring

**Phase:** 3.0D.0  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Defines the authoritative formula for computing the overall KOS Certification Score from domain scores. The formula is deterministic, fully specified, and machine-executable.

---

## Domain Weights

| Domain | Weight | Document |
|--------|--------|----------|
| Search | 0.10 | 02-SEARCH-CERTIFICATION |
| Registry | 0.10 | 03-REGISTRY-CERTIFICATION |
| Graph | 0.10 | 04-GRAPH-CERTIFICATION |
| Traceability | 0.10 | 05-TRACEABILITY-CERTIFICATION |
| Reasoning | 0.15 | 13-REASONING-CERTIFICATION |
| Performance | 0.10 | 14-PERFORMANCE-CERTIFICATION |
| Scalability | 0.10 | 15-SCALABILITY-CERTIFICATION |
| Reliability | 0.10 | 16 + 17 + 19 + 20 combined |
| Quality | 0.10 | 10-QUALITY-CERTIFICATION |
| Security | 0.05 | 18-SECURITY-CERTIFICATION |
| Evolution | 0.10 | 24-EVOLUTION-CERTIFICATION |
| **Total** | **1.00** | |

---

## Reliability Score Composition

The Reliability score (weight 0.10) is a weighted composite of 4 sub-domains:

| Sub-domain | Weight within Reliability |
|------------|--------------------------|
| Stress (16) | 0.30 |
| Concurrency (17) | 0.30 |
| Recovery (19) | 0.25 |
| Backup (20) | 0.15 |

---

## Overall Score Formula

```
reliability_score = (stress × 0.30) + (concurrency × 0.30) 
                  + (recovery × 0.25) + (backup × 0.15)

overall_score = (search × 0.10)
              + (registry × 0.10)
              + (graph × 0.10)
              + (traceability × 0.10)
              + (reasoning × 0.15)
              + (performance × 0.10)
              + (scalability × 0.10)
              + (reliability_score × 0.10)
              + (quality × 0.10)
              + (security × 0.05)
              + (evolution × 0.10)
```

All domain scores ∈ [0.0, 1.0]. Overall score ∈ [0.0, 1.0].

---

## Domain Score Formula

Each domain score is computed from its checks:

```
domain_score = Σ (check_passed[j] × check_weight[j])   for j in domain

check_passed[j] ∈ {0.0, 1.0}  (binary pass/fail)
Σ check_weight[j] = 1.0 within each domain
```

Severity-to-weight mapping within a domain:

| Severity | Weight |
|----------|--------|
| CRITICAL | 0.40 / CRITICAL count |
| MAJOR | 0.40 / MAJOR count |
| MINOR | 0.15 / MINOR count |
| INFO | 0.05 / INFO count |

Weights are normalised within each domain so they sum to 1.0.

---

## Certification Level Gates

| Level | Min Overall Score | Additional Gates |
|-------|-------------------|-----------------|
| Bronze | ≥ 0.60 | Zero CRITICAL failures |
| Silver | ≥ 0.70 | Zero CRITICAL failures; MAJOR pass rate ≥ 0.85 |
| Gold | ≥ 0.80 | All checks pass; P99 targets met at 100K objects |
| Enterprise | ≥ 0.90 | Gold + concurrency + stress + security all pass |
| Research | ≥ 0.95 | Enterprise + variance ≤ 5% + p < 0.05 |

Level is awarded at the **highest level where all gates are met**.

---

## Bonus / Penalty Rules

| Rule | Effect |
|------|--------|
| Zero hallucinations in R-5K | +0.02 bonus (capped at 1.0) |
| Any CRITICAL security failure | −0.10 penalty |
| Dataset certification failed | Score capped at 0.50 |
| Evolution score < 0.40 | Score capped at current level −1 |
| Checksum mismatch in golden dataset | Certification void |

---

## Score Precision

- All domain scores reported to 3 decimal places
- Overall score reported to 3 decimal places
- Level awarded based on floored score (0.799 = Silver, not Gold)

---

## Example Scoring

```
search_score       = 0.831  (× 0.10 = 0.0831)
registry_score     = 0.921  (× 0.10 = 0.0921)
graph_score        = 0.974  (× 0.10 = 0.0974)
traceability_score = 0.847  (× 0.10 = 0.0847)
reasoning_score    = 0.841  (× 0.15 = 0.1262)
performance_score  = 0.851  (× 0.10 = 0.0851)
scalability_score  = 0.912  (× 0.10 = 0.0912)
reliability_score  = 0.934  (× 0.10 = 0.0934)
quality_score      = 0.853  (× 0.10 = 0.0853)
security_score     = 0.933  (× 0.05 = 0.0467)
evolution_score    = 0.623  (× 0.10 = 0.0623)

overall = 0.0831 + 0.0921 + 0.0974 + 0.0847 + 0.1262
        + 0.0851 + 0.0912 + 0.0934 + 0.0853 + 0.0467 + 0.0623
overall = 0.847

Level = Gold (0.847 ≥ 0.80, all gates met)
```

---

## Report Format

```json
{
  "overall_score": 0.847,
  "level_achieved": "Gold",
  "domain_scores": {
    "search": 0.831,
    "registry": 0.921,
    "graph": 0.974,
    "traceability": 0.847,
    "reasoning": 0.841,
    "performance": 0.851,
    "scalability": 0.912,
    "reliability": 0.934,
    "quality": 0.853,
    "security": 0.933,
    "evolution": 0.623
  },
  "gates": {
    "zero_critical_failures": true,
    "major_pass_rate": 0.962,
    "p99_targets_100k": true
  },
  "bonuses": [],
  "penalties": [],
  "certified_at": "2026-06-30T00:00:00Z",
  "seed": 42,
  "dataset_version": "sha256:abc123"
}
```

---

## Cross-References

- Domain check definitions → `02` through `26`
- Dashboard display → `28-DASHBOARD`
- CI pass/fail gate → `29-CI-INTEGRATION`
