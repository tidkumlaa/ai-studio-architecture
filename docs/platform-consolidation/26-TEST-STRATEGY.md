---
knowledge_id: KNW-PLAT-ARCH-026
title: "Platform Test Strategy"
status: AUTHORITATIVE
phase: "2.1D.0"
version: "1.0.0"
created: "2026-06-29"
purpose: "Define the test architecture, naming conventions, coverage targets, and test organization"
canonical_source: "architecture/docs/platform-consolidation/26-TEST-STRATEGY.md"
dependencies:
  - "06-RUNTIME-MODEL.md"
  - "09-SERVICE-MODEL.md"
  - "25-VERIFICATION.md"
related_documents:
  - "21-ALGORITHMS.md"
  - "27-PERFORMANCE-BUDGET.md"
acceptance_criteria:
  - "Test types and scopes are unambiguous"
  - "Coverage targets are per-stability-level"
  - "Test file naming convention is enforced by CI"
verification_checklist:
  - "[ ] All test types defined"
  - "[ ] Coverage targets stated per module stability"
  - "[ ] No test depends on external APIs (unit tests)"
future_extensions:
  - "Property-based testing with Hypothesis"
  - "Mutation testing for critical algorithms"
---

# Platform Test Strategy

## Test Pyramid

```
         ╱╲
        ╱E2E╲         (few — test full flows end-to-end)
       ╱──────╲
      ╱ Integ  ╲      (some — test runtime interactions)
     ╱──────────╲
    ╱    Unit    ╲    (many — test each module in isolation)
   ╱──────────────╲
  ╱  Architecture  ╲  (always — doc/spec compliance)
 ╱────────────────────╲
```

---

## Test Types

### Unit Tests
- **Location:** `platform/tests/unit/`
- **Scope:** Single module; no file I/O, no network, no external APIs
- **Mocking:** Mock all dependencies injected via constructor
- **Speed:** Must complete in < 100ms each
- **Framework:** pytest, no extra plugins required

### Integration Tests
- **Location:** `platform/tests/integration/`
- **Scope:** Multiple modules or runtimes interacting
- **Mocking:** Real EventBus, real DI container; mock providers only
- **Speed:** Must complete in < 5s each
- **Framework:** pytest + pytest-asyncio

### Architecture Tests
- **Location:** `platform/tests/architecture/`
- **Scope:** Verify layer rules, import rules, metadata standards
- **Mocking:** None — reads actual source files
- **Speed:** < 30s total
- **Framework:** pytest + custom file scanners

### E2E Tests (Planned)
- **Location:** `platform/tests/e2e/`
- **Scope:** Full platform boot to task execution
- **Mocking:** Real providers in sandbox environment
- **Speed:** < 60s each
- **Framework:** pytest + docker-compose

---

## Test File Naming Convention

```
test_{module_name}.py            — unit tests for a module
test_{runtime}_integration.py   — integration tests for a runtime
test_arch_{domain}.py           — architecture compliance tests
test_e2e_{scenario}.py          — end-to-end scenarios
```

Examples:
- `test_ri2_workload_profiler.py` — unit tests for workload_profiler (Phase 2.1B)
- `test_ai_runtime_integration.py` — AI runtime integration
- `test_arch_imports.py` — import rule compliance
- `test_e2e_task_execution.py` — full task execution flow

---

## Coverage Targets

| Stability Level | Line Coverage | Branch Coverage |
|-----------------|---------------|-----------------|
| STABLE | ≥ 90% | ≥ 80% |
| BETA | ≥ 80% | ≥ 70% |
| EXPERIMENTAL | ≥ 60% | ≥ 50% |
| PLANNED | N/A | N/A |

Coverage is measured per module. Modules below threshold block CI.

---

## Current Test Inventory

| Phase | Test File | Count |
|-------|-----------|-------|
| 2.1A | `test_usage_collector.py` | ~8 |
| 2.1A | `test_quota_manager.py` | ~10 |
| 2.1A | `test_routing_optimizer.py` | ~8 |
| 2.1A | `test_cost_optimizer.py` | ~6 |
| 2.1A | `test_forecast_engine.py` | ~8 |
| 2.1A | `test_account_pool.py` | ~8 |
| 2.1A | `test_subscription_manager.py` | ~10 |
| 2.1A | `test_session_analyzer.py` | ~8 |
| 2.1A | `test_token_estimator.py` | ~8 |
| 2.1A | `test_budget_manager.py` | ~8 |
| 2.1A | `test_recommendation_engine.py` | ~8 |
| 2.1A | `test_workload_classifier.py` | ~8 |
| **2.1A Total** | | **~88** |
| 2.1B | `test_ri2_workload_profiler.py` | 14 |
| 2.1B | `test_ri2_task_complexity.py` | 10 |
| 2.1B | `test_ri2_token_predictor.py` | 8 |
| 2.1B | `test_ri2_cost_predictor.py` | 6 |
| 2.1B | `test_ri2_quality_predictor.py` | 7 |
| 2.1B | `test_ri2_provider_selector.py` | 6 |
| 2.1B | `test_ri2_execution_planner.py` | 7 |
| 2.1B | `test_ri2_learning_engine.py` | 8 |
| 2.1B | `test_ri2_recommendation_center.py` | 6 |
| **2.1B Total** | | **72** |
| **Grand Total** | | **≥ 160** |

---

## Test Requirements per Module

### Workload Profiler
- Each WorkloadType detected for at least one prompt
- UNKNOWN returned for empty / ambiguous input
- Vision/tool/reasoning flags correctly set

### Task Complexity Analyzer
- Each ComplexityLevel reachable
- Thresholds verified at boundaries
- Retries higher for complex tasks

### Token Predictor
- EMA updates converge over time
- Prediction correct within ±20% for known workload types
- Edge case: zero-length prompt

### Cost Predictor
- best_case < expected < worst_case always
- Decimal precision — no float errors
- All 11 models in catalog covered

### Provider Selector
- Budget filter: `is not None` check (not falsy)
- All providers considered when no budget constraint
- LOCAL provider always available as fallback

### Execution Planner
- Single-agent for simple tasks
- Multi-agent-parallel for complex+parallelizable
- Retry strategy populated

### Learning Engine
- Accuracy trend: improving / stable / degrading all reachable
- Max 100,000 records respected
- Thread-safe record insertion

---

## Test Isolation Rules

| Rule | Rationale |
|------|-----------|
| No test shares state with another test | `pytest` fixture scope = `function` by default |
| No test reads from production config files | Use `tmp_path` or in-memory config |
| No test writes to `platform/` source files | Tests are read-only regarding source |
| No test hits external APIs | Stub or mock all provider calls |
| No test depends on system clock | Inject `datetime.utcnow` as parameter |

---

## Running Tests

```bash
# All unit tests
pytest platform/tests/unit/ -v

# Phase 2.1B tests only
pytest platform/tests/unit/ -k "ri2" -v

# All tests with coverage
pytest platform/tests/ --cov=platform --cov-report=term-missing

# Architecture tests
pytest platform/tests/architecture/ -v
```
