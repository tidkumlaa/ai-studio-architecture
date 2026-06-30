# UICE-DOC-020 — Context Metrics

**Phase:** 3.0D.2
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Context Metrics defines the 50 UICE metrics (UICE-M-001 through UICE-M-050)
across 8 categories: Efficiency, Quality, Caching, Conversation, Learning,
Performance, Validation, and Costs. These metrics are the ground truth for
measuring UICE success and triggering alerts.

---

## 50 UICE Metrics

### Category 1: Token Efficiency (UICE-M-001 through UICE-M-008)

| ID | Metric | Target | Alert |
|----|--------|--------|-------|
| UICE-M-001 | token_reduction_pct — mean reduction vs naive | ≥ 99% | < 95% |
| UICE-M-002 | fla_savings_pct — FLA vs L4 Standard | ≥ 70% | < 50% |
| UICE-M-003 | sco_savings_pct — shared context savings | > 50% | < 30% |
| UICE-M-004 | delta_savings_pct — conversation delta savings | > 90% | < 70% |
| UICE-M-005 | dedup_eliminated_tokens — tokens removed by deduplicator | > 0 | never increases |
| UICE-M-006 | omit_rate_pct — fraction of slots that are OMIT mode | informational | — |
| UICE-M-007 | tokens_per_pack_mean — mean final token count per pack | < 200 | > 500 |
| UICE-M-008 | field_precision_pct — fraction of included fields that were referenced by LLM | > 99% | < 90% |

### Category 2: Quality (UICE-M-009 through UICE-M-016)

| ID | Metric | Target | Alert |
|----|--------|--------|-------|
| UICE-M-009 | quality_score_mean | ≥ 0.85 | < 0.80 |
| UICE-M-010 | quality_score_p10 — 10th percentile | ≥ 0.75 | < 0.70 |
| UICE-M-011 | packs_below_floor_pct — quality < 0.80 | < 5% | > 10% |
| UICE-M-012 | hallucination_rate_type_a — wrong fact | < 1% | > 2% |
| UICE-M-013 | hallucination_rate_type_b — wrong dependency | < 1% | > 2% |
| UICE-M-014 | guard_effectiveness_pct — hallucination reduction vs baseline | > 30% | < 20% |
| UICE-M-015 | coverage_score_mean — relevant objects included / total relevant | ≥ 0.90 | < 0.80 |
| UICE-M-016 | completeness_score_mean — required fields present / total required | ≥ 0.95 | < 0.90 |

### Category 3: Caching (UICE-M-017 through UICE-M-022)

| ID | Metric | Target | Alert |
|----|--------|--------|-------|
| UICE-M-017 | l1_cache_hit_rate | ≥ 60% | < 40% |
| UICE-M-018 | l2_cache_hit_rate | ≥ 80% | < 60% |
| UICE-M-019 | semantic_cache_hit_rate | ≥ 40% | < 20% |
| UICE-M-020 | reuse_engine_hit_rate | ≥ 60% | < 40% |
| UICE-M-021 | cache_invalidation_rate — invalidations per hour | informational | > 1000/hr |
| UICE-M-022 | stale_entry_rate — stale entries served before eviction | < 0.1% | > 1% |

### Category 4: Conversation (UICE-M-023 through UICE-M-027)

| ID | Metric | Target | Alert |
|----|--------|--------|-------|
| UICE-M-023 | conversation_reuse_rate — sessions with memory active | ≥ 70% | < 50% |
| UICE-M-024 | delta_pack_rate — packs using delta mode | ≥ 30% | < 10% |
| UICE-M-025 | memory_capacity_utilization — mean entries / 10000 | < 80% | > 90% |
| UICE-M-026 | re_query_rate — consumer re-queries within 30s | < 5% | > 15% |
| UICE-M-027 | session_satisfaction_score — mean explicit rating | > 4.0 | < 3.5 |

### Category 5: Learning (UICE-M-028 through UICE-M-032)

| ID | Metric | Target | Alert |
|----|--------|--------|-------|
| UICE-M-028 | feedback_records_per_week | ≥ 50 | < 20 |
| UICE-M-029 | proposals_generated_per_week | informational | — |
| UICE-M-030 | proposals_approved_rate | ≥ 50% | < 20% |
| UICE-M-031 | quality_delta_post_learning — quality improvement after proposal applied | > 0.02 | < 0.00 |
| UICE-M-032 | learning_coverage_pct — (intent, query_type) combinations with ≥ 50 feedback records | ≥ 80% | < 50% |

### Category 6: Performance (UICE-M-033 through UICE-M-040)

| ID | Metric | Target | Alert |
|----|--------|--------|-------|
| UICE-M-033 | compilation_p99_ms | < 60ms | > 120ms |
| UICE-M-034 | compilation_p50_ms | < 20ms | > 50ms |
| UICE-M-035 | graph_traversal_p99_ms | < 20ms | > 40ms |
| UICE-M-036 | fla_computation_p99_ms | < 5ms | > 15ms |
| UICE-M-037 | semantic_cache_lookup_p99_ms | < 8ms | > 20ms |
| UICE-M-038 | memory_rss_gb | < 2GB | > 4GB |
| UICE-M-039 | memory_leak_mb_per_hour | 0 | > 5 |
| UICE-M-040 | throughput_rps — requests per second sustained | ≥ 100 | < 50 |

### Category 7: Validation (UICE-M-041 through UICE-M-046)

| ID | Metric | Target | Alert |
|----|--------|--------|-------|
| UICE-M-041 | invariant_violation_rate — UI-01 through UI-10 | 0% | > 0% |
| UICE-M-042 | verifier_reject_rate — packs rejected by Context Verifier | < 1% | > 5% |
| UICE-M-043 | ci_compliance_rate — CAE CI-01 through CI-10 | 100% | < 100% |
| UICE-M-044 | l3_floor_trigger_rate — FLA needed L3 floor | informational | — |
| UICE-M-045 | budget_overflow_rate — assembled pack exceeded budget | 0% | > 0% |
| UICE-M-046 | dedup_100_pct — TYPE-2 duplicates eliminated | 100% | < 100% |

### Category 8: Cost (UICE-M-047 through UICE-M-050)

| ID | Metric | Target | Alert |
|----|--------|--------|-------|
| UICE-M-047 | cost_per_1k_packs_usd — assembly cost | < $0.01 | > $0.05 |
| UICE-M-048 | token_cost_reduction_pct — vs naive context | > 99% | < 95% |
| UICE-M-049 | model_cost_per_pack_usd | informational | > $0.10 |
| UICE-M-050 | cost_within_constraint_rate — packs satisfying max_cost_usd | ≥ 99% | < 95% |

---

## Metric Collection

```
All 50 metrics are collected per-pack and aggregated:
  - Per request: latency, token counts, cache hits
  - Per session: conversation savings, memory utilization
  - Per hour: throughput, memory, learning stats
  - Per day: quality scores, hallucination rates, costs
  - Per week: learning cycle, proposals, applied changes
```

---

## Rules

| Rule | Statement |
|------|-----------|
| UICE-096 | All 50 metrics must be reported in every validation run (KVF requirement) |
| UICE-097 | UICE-M-041 (invariant violations) must be 0; any violation triggers immediate alert |
| UICE-098 | UICE-M-033 (P99 compilation) alert threshold is 120ms — same as CAE; UICE target is 60ms |
| UICE-099 | Metrics UICE-M-001 and UICE-M-008 are the primary efficiency KPIs for UICE certification |
| UICE-100 | Cost metrics (UICE-M-047 through UICE-M-050) are optional if no cost constraints are configured |

---

## Cross-References

- Context Profiler → `21-CONTEXT-PROFILER`
- Context Benchmark → `27-CONTEXT-BENCHMARK`
- Context Certification → `28-CONTEXT-CERTIFICATION`
- Dashboard → `29-DASHBOARD`
