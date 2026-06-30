# KNW-FINAL-013 — Canonical Evidence

**Phase:** 3.0D.0.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Defines canonical evidence patterns for each evidence type, with good and bad examples. Evidence is the primary driver of the QD-3 quality dimension.

---

## Evidence Type Quick Reference

| Type | Code | Level | Weight | Freshness window |
|------|------|-------|--------|-----------------|
| Board Decision | EV-HAPPROVAL | 1 | 0.80 | 1 year |
| Architecture Doc | EV-DOC | 2 | 0.60 | 6 months |
| External Standard | EV-STANDARD | 2 | 0.80 | 1 year |
| Benchmark Result | EV-BENCHMARK | 3 | 0.70 | 90 days |
| Test Result | EV-TEST | 3 | 0.75 | 30 days |
| Monitoring Data | EV-MONITORING | 3 | 0.70 | 7 days |
| Research Paper | EV-RESEARCH | 3 | 0.65 | 2 years |
| Production Code | EV-CODE | 4 | 0.60 | 180 days |
| Engineering Proposal | EV-PROPOSAL | 5 | 0.40 | 30 days |
| User Feedback | EV-USER | 5 | 0.50 | 30 days |

---

## EV-HAPPROVAL — Board Decision (Canonical Example)

```yaml
- evidence_type: EV-HAPPROVAL
  source_id: KNW-META-DEC-001
  source_url: "architecture/docs/kos-final/20-KOS-V1-FREEZE.md"
  description: "Architecture Board approved KOS v1.0 on 2026-06-30"
  weight: 0.80
  level: 1
  captured_at: "2026-06-30T00:00:00Z"
  freshness_score: 1.0
  disputed: false
```

**Use when:** An Architecture Board meeting or vote approves a decision that underpins this object.

---

## EV-DOC — Architecture Document

```yaml
- evidence_type: EV-DOC
  source_id: KNW-ARCH-ARCH-001
  source_url: "architecture/docs/knowledge-core/37-PERFORMANCE-BUDGET.md"
  description: "P99 < 5ms specified in Section 'Registry Budgets'"
  weight: 0.60
  level: 2
  captured_at: "2026-06-30T00:00:00Z"
  freshness_score: 1.0
  disputed: false
```

**Use when:** A frozen architecture document specifies a property this object is claimed to satisfy.

---

## EV-BENCHMARK — Benchmark Result

```yaml
- evidence_type: EV-BENCHMARK
  source_id: KNW-TEST-BENCH-001
  source_url: "reports/certification/cert-2026-06-30/performance.json"
  description: "Identity engine benchmark: generate_id P99 = 0.4ms at 100K ops"
  weight: 0.70
  level: 3
  captured_at: "2026-06-30T00:00:00Z"
  freshness_score: 1.0
  disputed: false
  result_data:
    p50_ms: 0.1
    p99_ms: 0.4
    operations: 100000
    dataset_size: 100000
```

**Use when:** A benchmark run provides measured performance evidence.

---

## EV-TEST — Test Result

```yaml
- evidence_type: EV-TEST
  source_id: KNW-TEST-TST-001
  description: "Quota enforcement test passes 100% in CI"
  weight: 0.75
  level: 3
  captured_at: "2026-06-30T00:00:00Z"
  freshness_score: 1.0
  result_data:
    pass_rate: 1.0
    tests_run: 15
    ci_run: "ci-2026-06-30-1234"
```

**Use when:** A test suite verifies the correctness of this object's implementation.

---

## EV-CODE — Production Code

```yaml
- evidence_type: EV-CODE
  source_url: "platform/knowledge_runtime/identity/engine.py"
  description: "Identity engine implemented and committed in platform/"
  weight: 0.60
  level: 4
  captured_at: "2026-06-30T00:00:00Z"
  freshness_score: 1.0
  git_sha: "ce0680c"
```

**Use when:** The object's implementation exists in committed production code.

---

## EV-RESEARCH — Research Paper

```yaml
- evidence_type: EV-RESEARCH
  source_url: "https://doi.org/10.1561/1500000019"
  description: "Robertson & Zaragoza 2009 — The Probabilistic Relevance Framework: BM25 and Beyond"
  weight: 0.65
  level: 3
  captured_at: "2009-01-01T00:00:00Z"
  freshness_score: 0.85
  doi: "10.1561/1500000019"
```

**Use when:** A published research paper establishes the theoretical basis for an algorithm.

---

## Evidence Anti-Patterns

| Anti-Pattern | Error | Fix |
|-------------|-------|-----|
| No evidence on CANONICAL object | CS-001 violation | Add ≥ 1 evidence item |
| All evidence Level 5–6 | CS-003 violation | Add ≥ 1 Level 1–4 source |
| Stale benchmark (> 90 days) | freshness_score low | Re-run benchmark; update captured_at |
| Disputed item without reason | EC-007 violation | Add `dispute_reason` field |
| URL-only (no source_id) | best practice | Archive URL; use DOI or git SHA |
| EV-MONITORING with 30-day-old data | Very stale | Re-collect monitoring data (7-day window) |

---

## Evidence Score Calculation

```
For each evidence item:
  level_bonus = {1: 1.0, 2: 0.85, 3: 0.75, 4: 0.65, 5: 0.50, 6: 0.30}
  item_score = item.weight × item.freshness_score × level_bonus[item.level]

QD3 = mean(item_scores for all non-disputed items)
      (disputed items weighted × 0.5)
```

---

## Cross-References

- Canonical sources → Phase 3.0C.5 `23-KNOWLEDGE-CANONICAL-SOURCES`
- Evidence engine → Phase 3.0C `10-EVIDENCE-ENGINE`
- Evidence certification → Phase 3.0D.0 `11-EVIDENCE-CERTIFICATION`
- QD-3 in scoring → Phase 3.0C.5 `17-KNOWLEDGE-SCORING`
