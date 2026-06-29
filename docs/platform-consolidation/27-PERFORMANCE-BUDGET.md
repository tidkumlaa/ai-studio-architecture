---
knowledge_id: KNW-PLAT-ARCH-027
title: "Platform Performance Budget"
status: AUTHORITATIVE
phase: "2.1D.0"
version: "1.0.0"
created: "2026-06-29"
purpose: "Define performance budgets for all platform operations: latency, throughput, memory, and boot time"
canonical_source: "architecture/docs/platform-consolidation/27-PERFORMANCE-BUDGET.md"
dependencies:
  - "06-RUNTIME-MODEL.md"
  - "09-SERVICE-MODEL.md"
  - "21-ALGORITHMS.md"
related_documents:
  - "28-DASHBOARD.md"
  - "25-VERIFICATION.md"
acceptance_criteria:
  - "Every operation has a P50 and P99 budget"
  - "Budgets are measurable (no subjective targets)"
  - "Violation thresholds trigger alerts"
verification_checklist:
  - "[ ] Budgets are realistic for current hardware"
  - "[ ] Alert thresholds are 80% of budget (not 100%)"
  - "[ ] Memory budgets account for worst-case dataset sizes"
future_extensions:
  - "Adaptive budgets based on hardware profile"
  - "Budget regression test in CI"
---

# Platform Performance Budget

## Measurement Methodology

All latency budgets are measured:
- In-process (not network round-trip unless specified)
- On a standard developer machine: 4-core CPU, 16 GB RAM
- With warm caches (not cold-start, unless specified)
- P50 = median, P99 = 99th percentile

---

## Kernel Operations

| Operation | P50 | P99 | Alert at |
|-----------|-----|-----|---------|
| `EventBus.publish()` | < 0.5 ms | < 2 ms | > 1.5 ms |
| `EventBus.subscribe()` | < 0.1 ms | < 0.5 ms | > 0.4 ms |
| `DIContainer.resolve()` | < 0.1 ms | < 0.5 ms | > 0.4 ms |
| `PlatformConfig` load | < 10 ms | < 50 ms | > 40 ms |
| Kernel full boot | < 500 ms | < 2 s | > 1.5 s |

---

## Runtime Boot Times

| Runtime | Boot Budget (P50) | Boot Budget (P99) | Alert at |
|---------|-------------------|-------------------|---------|
| event_runtime | < 50 ms | < 200 ms | > 160 ms |
| resource_runtime | < 100 ms | < 500 ms | > 400 ms |
| provider_runtime | < 200 ms | < 1 s | > 800 ms |
| knowledge_runtime | < 500 ms | < 3 s | > 2.4 s |
| ai_runtime | < 500 ms | < 3 s | > 2.4 s |
| workflow_runtime | < 100 ms | < 500 ms | > 400 ms |
| orchestration_runtime | < 100 ms | < 500 ms | > 400 ms |
| **Total platform boot** | **< 2 s** | **< 8 s** | > 6 s |

Boot order is sequential (1→7), so total = sum of individual boots.

---

## AI Resource Intelligence Operations (Phase 2.1A)

| Operation | P50 | P99 | Notes |
|-----------|-----|-----|-------|
| Quota check | < 1 ms | < 5 ms | In-memory |
| Quota consume | < 2 ms | < 10 ms | Write + event |
| Budget check | < 1 ms | < 5 ms | In-memory |
| Usage record | < 5 ms | < 20 ms | Write to store |
| Routing decision | < 10 ms | < 50 ms | Policy + quota + cost |
| Cost estimate | < 2 ms | < 10 ms | Math only |
| Forecast (24h) | < 100 ms | < 500 ms | Historical data scan |
| Report generation | < 500 ms | < 2 s | Full dataset scan |

---

## AI Intelligence Engine Operations (Phase 2.1B)

| Operation | P50 | P99 | Notes |
|-----------|-----|-----|-------|
| `profile_workload()` | < 2 ms | < 10 ms | Signal matching |
| `analyze_complexity()` | < 2 ms | < 10 ms | Scoring math |
| `predict_tokens()` | < 1 ms | < 5 ms | Ratio lookup |
| `predict_cost()` | < 1 ms | < 5 ms | Pricing math |
| `predict_quality()` | < 0.5 ms | < 2 ms | Enum lookup |
| `select_provider()` | < 5 ms | < 20 ms | Multi-provider score |
| `select_model()` | < 5 ms | < 20 ms | Catalog scan |
| `plan()` (full pipeline) | < 20 ms | < 100 ms | All modules chained |
| `optimize_plan()` | < 10 ms | < 50 ms | Opportunity scan |
| `recommend()` | < 5 ms | < 20 ms | Rule evaluation |
| `record_outcome()` (async) | < 5 ms | < 20 ms | EMA update |
| `get_learning_stats()` | < 50 ms | < 200 ms | Stats computation |

---

## Service Operations

| Service | Operation | P50 | P99 |
|---------|-----------|-----|-----|
| AuthService | `verify_token()` | < 2 ms | < 10 ms |
| AuditService | `log()` | < 1 ms | < 5 ms |
| MetricsService | `counter()` | < 0.1 ms | < 1 ms |
| CacheService | `get()` | < 0.5 ms | < 2 ms |
| CacheService | `set()` | < 1 ms | < 5 ms |

---

## Memory Budgets

| Component | Steady-State | Peak | OOM Alert |
|-----------|-------------|------|-----------|
| Kernel | 50 MB | 100 MB | > 80 MB |
| AI runtime (all modules) | 200 MB | 500 MB | > 400 MB |
| Learning engine records | 100 MB (100K records) | 200 MB | > 160 MB |
| Knowledge runtime | 200 MB | 1 GB | > 800 MB |
| Provider runtime | 50 MB | 100 MB | > 80 MB |
| **Total platform** | **700 MB** | **2 GB** | > 1.5 GB |

---

## Throughput Budgets

| Operation | Minimum Throughput |
|-----------|--------------------|
| Quota checks | 10,000 / second |
| Usage records | 1,000 / second |
| Intelligence `plan()` calls | 500 / second |
| Event bus publish | 50,000 / second |
| Cache get/set | 100,000 / second |

---

## Learning Engine Limits

| Limit | Value | Rationale |
|-------|-------|-----------|
| Max stored records | 100,000 | Memory budget |
| Record eviction policy | FIFO (oldest removed) | Recency bias |
| EMA α (learning rate) | 0.3 | Stability vs. adaptability |
| Min records for trend | 10 | Statistical significance |
| Trend window | Full record set / 2 | Compare first vs. second half |

---

## CI Performance Gate

```bash
# Benchmark suite — fails if any budget exceeded
platform-verify performance --budget

# Reports:
# PASS  plan() p50=15ms p99=88ms (budget: p50<20ms p99<100ms)
# FAIL  forecast() p99=620ms (budget: p99<500ms) ← alert
```
