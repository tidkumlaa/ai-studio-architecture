# AI Budget Engine — Cost Accounting and Budget Enforcement

**Version:** 1.0.0  
**Status:** Approved for Implementation  
**Phase:** 1A  
**Date:** 2026-06-29  
**Classification:** Internal Architecture

---

## 1. Purpose

The Budget Engine owns all cost accounting for AI operations across AI Studio. It computes the real cost of every execution, enforces project and department spending budgets, generates chargeback reports, forecasts burn rates, and provides ROI analytics. It is a parallel enforcement gate alongside the Quota Engine — a request can be blocked by either quota or budget, independently.

---

## 2. Responsibilities

- Real-time cost computation from token usage and the Model Catalog pricing data
- Budget definition: organization, project, department, and per-user limits
- Pre-dispatch budget availability check (hard gate)
- Post-execution cost recording in the cost ledger
- Budget burn rate calculation and depletion forecasting
- Spend alerts at configurable thresholds
- Chargeback report generation (by project, department, user, model, provider)
- ROI tracking: cost vs. outcome quality
- Cost optimization recommendations

---

## 3. Architecture

```
Routing Engine (pre-dispatch, parallel with Quota Engine)
        │
        │ check_budget(org_id, project_id, estimated_cost)
        ▼
┌────────────────────────────────────────────────────────────────────┐
│                       Budget Engine                                  │
│                                                                      │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  │
│  │  Budget Registry │  │  Cost Calculator │  │  Forecast Engine │  │
│  │  (config store)  │  │  (pricing lookup)│  │                  │  │
│  └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘  │
│           │                     │                      │             │
│  ┌────────▼─────────────────────▼──────────────────────▼──────────┐  │
│  │                        Budget Core                               │  │
│  │                                                                  │  │
│  │  check_budget(scope, estimated_cost) → BudgetCheckResult        │  │
│  │  record_cost(scope, request_id, actual_cost)                    │  │
│  │  refund_cost(scope, request_id, amount)                         │  │
│  │  position(scope) → BudgetPosition                               │  │
│  │  forecast(scope) → BurnForecast                                 │  │
│  │  chargeback_report(scope, period) → ChargebackReport            │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌──────────────────┐  ┌──────────────────────────────────────────┐  │
│  │  Alert Manager   │  │  Cost Optimizer                           │  │
│  └──────────────────┘  └──────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘
         │                          │
         ▼                          ▼
  BudgetLedger table          Event Bus
  (PostgreSQL)
```

---

## 4. Data Models

### 4.1 BudgetRule

| Field | Type | Description |
|-------|------|-------------|
| `rule_id` | `UUID` | Primary key |
| `org_id` | `UUID` | Organization scope |
| `project_id` | `UUID \| None` | Project-specific budget |
| `department_id` | `str \| None` | Department identifier |
| `user_id` | `UUID \| None` | Per-user budget |
| `provider_id` | `str \| None` | Provider-specific budget (or all providers) |
| `period` | `BudgetPeriod` | `DAILY` / `WEEKLY` / `MONTHLY` / `QUARTERLY` / `ANNUAL` / `LIFETIME` |
| `currency` | `str` | Always `USD` |
| `limit_usd` | `Decimal` | Hard spending cap |
| `warning_threshold_pct` | `int` | Default: 70 |
| `critical_threshold_pct` | `int` | Default: 90 |
| `on_exhaustion` | `BudgetExhaustionAction` | What to do when budget is spent |
| `approval_required_above_usd` | `Decimal \| None` | Trigger Approval Engine for single requests above this |
| `rollover_enabled` | `bool` | Carry unused budget to next period |
| `notes` | `str` | Admin notes |

### 4.2 BudgetExhaustionAction

| Action | Behavior |
|--------|---------|
| `REJECT` | Block request with structured error |
| `QUEUE` | Queue request for next budget period |
| `APPROVE_OVERRIDE` | Trigger human approval workflow |
| `ALERT_ONLY` | Allow but send alert (for monitoring budgets) |

### 4.3 CostRecord

| Field | Type | Description |
|-------|------|-------------|
| `record_id` | `UUID` | Primary key |
| `request_id` | `UUID` | AI ROS request ID |
| `org_id` | `UUID` | |
| `project_id` | `UUID \| None` | |
| `department_id` | `str \| None` | |
| `user_id` | `UUID \| None` | |
| `provider_id` | `str` | |
| `canonical_model_id` | `str` | |
| `input_tokens` | `int` | |
| `output_tokens` | `int` | |
| `cache_read_tokens` | `int` | |
| `cache_write_tokens` | `int` | |
| `reasoning_tokens` | `int` | |
| `image_count` | `int` | |
| `audio_seconds` | `int` | |
| `input_cost_usd` | `Decimal` | |
| `output_cost_usd` | `Decimal` | |
| `cache_cost_usd` | `Decimal` | |
| `image_cost_usd` | `Decimal` | |
| `audio_cost_usd` | `Decimal` | |
| `total_cost_usd` | `Decimal` | Sum of all cost components |
| `subscription_covered` | `bool` | True if covered by flat-rate subscription |
| `recorded_at` | `datetime` | |
| `task_type` | `str \| None` | Application-level task category (for ROI analysis) |
| `outcome_quality_score` | `float \| None` | From Benchmark Engine, 0.0–1.0 |

---

## 5. Cost Calculation

### 5.1 Real-Time Cost Formula

```
input_cost   = (input_tokens / 1,000,000) * model.pricing.input_per_million_tokens
output_cost  = (output_tokens / 1,000,000) * model.pricing.output_per_million_tokens
cache_read   = (cache_read_tokens / 1,000,000) * model.pricing.cache_read_per_million_tokens
cache_write  = (cache_write_tokens / 1,000,000) * model.pricing.cache_write_per_million_tokens
reason_cost  = (reasoning_tokens / 1,000,000) * model.pricing.reasoning_per_million_tokens
image_cost   = image_count * model.pricing.image_output_per_image
audio_cost   = (audio_seconds / 60) * model.pricing.audio_input_per_minute

total_cost = input_cost + output_cost + cache_read + cache_write + reason_cost + image_cost + audio_cost
```

### 5.2 Subscription Coverage

If the model is included in an active flat-rate subscription for this org/user (per Account Manager), then:
- `subscription_covered = True`
- `total_cost_usd = 0.00`
- Budget Engine still records the cost for analytics purposes, but does not deduct from budget

This allows tracking "what would this cost without subscription" — valuable for ROI analysis and subscription ROI reporting.

### 5.3 Pre-Dispatch Estimation

The Routing Engine calls `check_budget` before dispatching. At this point, the actual token counts are not yet known. The Budget Engine uses an estimation:

```
estimated_input_tokens  = count_tokens(messages)     # tokenizer estimate
estimated_output_tokens = max_tokens_requested OR model.max_output_tokens * 0.3
estimated_cost          = (above formula with estimated counts)
```

The pre-dispatch check reserves `estimated_cost * 1.2` (20% buffer) from the budget to prevent over-allocation when multiple requests run in parallel.

The reservation is released and replaced with the actual cost once the response is received.

---

## 6. Core Operations

### 6.1 check_budget()

```
check_budget(
  org_id: UUID,
  project_id: UUID | None,
  user_id: UUID | None,
  provider_id: str,
  estimated_cost_usd: Decimal
) → BudgetCheckResult
```

**BudgetCheckResult:**

| Field | Type | Description |
|-------|------|-------------|
| `allowed` | `bool` | |
| `requires_approval` | `bool` | Whether Approval Engine must be invoked |
| `blocking_rules` | `List[BudgetViolation]` | Rules that would be violated |
| `estimated_cost_usd` | `Decimal` | |
| `positions` | `List[BudgetPosition]` | Current spend position for checked rules |

### 6.2 record_cost()

```
record_cost(
  org_id: UUID,
  request_id: UUID,
  cost_record: CostRecord
)
```

Writes to `ai_ros_cost_ledger`. Updates materialized position views.

### 6.3 BudgetPosition

| Field | Type | Description |
|-------|------|-------------|
| `rule_id` | `UUID` | |
| `period` | `BudgetPeriod` | |
| `limit_usd` | `Decimal` | |
| `spent_usd` | `Decimal` | Confirmed costs in this period |
| `reserved_usd` | `Decimal` | Pre-dispatch reservations (in-flight) |
| `available_usd` | `Decimal` | `limit - spent - reserved` |
| `spent_pct` | `float` | |
| `reset_at` | `datetime \| None` | |
| `alert_level` | `AlertLevel` | `OK` / `WARNING` / `CRITICAL` / `EXHAUSTED` |

---

## 7. Burn Rate and Forecasting

### 7.1 BurnForecast

| Field | Type | Description |
|-------|------|-------------|
| `rule_id` | `UUID` | |
| `daily_burn_rate_usd` | `Decimal` | Rolling 7-day average daily spend |
| `projected_exhaustion_at` | `datetime \| None` | |
| `projected_spend_at_period_end_usd` | `Decimal` | |
| `over_budget_by_usd` | `Decimal \| None` | If projection exceeds limit |
| `confidence` | `float` | |
| `recommendation` | `str` | e.g., "Reduce batch job frequency" or "Upgrade to higher budget tier" |

### 7.2 Forecast Triggers

| Trigger | Event Published |
|---------|----------------|
| Projected spend > 100% of limit before period end | `budget.forecast.over_budget` |
| Projected spend > 90% of limit before period end | `budget.forecast.near_limit` |
| Single request > `approval_required_above_usd` | `budget.approval_required` |
| Actual spend > `warning_threshold_pct` | `budget.warning` |
| Actual spend > `critical_threshold_pct` | `budget.critical` |
| Budget exhausted | `budget.exhausted` |

---

## 8. Chargeback Reports

### 8.1 ChargebackReport

Generated on-demand or scheduled (weekly/monthly).

| Field | Type | Description |
|-------|------|-------------|
| `report_id` | `UUID` | |
| `org_id` | `UUID` | |
| `period_start` | `date` | |
| `period_end` | `date` | |
| `generated_at` | `datetime` | |
| `line_items` | `List[ChargebackLineItem]` | |
| `total_cost_usd` | `Decimal` | |
| `subscription_savings_usd` | `Decimal` | Value covered by flat-rate subscriptions |
| `actual_spend_usd` | `Decimal` | `total_cost - subscription_savings` |

### 8.2 ChargebackLineItem

| Field | Type | Description |
|-------|------|-------------|
| `dimension` | `ChargebackDimension` | `PROJECT` / `DEPARTMENT` / `USER` / `MODEL` / `PROVIDER` / `TASK_TYPE` |
| `dimension_id` | `str` | Project ID, department name, user ID, etc. |
| `dimension_label` | `str` | Human-readable name |
| `request_count` | `int` | |
| `total_tokens` | `int` | |
| `cost_usd` | `Decimal` | |
| `pct_of_total` | `float` | |

---

## 9. Cost Optimization Recommendations

The Cost Optimizer analyzes spend patterns and generates recommendations:

| Pattern Detected | Recommendation |
|----------------|----------------|
| High-cost model used for simple tasks | "Route classification tasks to haiku-4-5 (85% cheaper)" |
| Repeated identical prompts | "Enable semantic cache — 60% of last week's requests were duplicates" |
| Low cache hit rate | "Restructure prompts to move static content to system prompt for Claude prompt caching" |
| Peak-hour spending | "Schedule batch jobs during off-peak hours for better rate limit headroom" |
| Unused subscription models | "Your Claude Max subscription includes Opus 4.8 — you're routing identical tasks to paid API" |

Recommendations are published as events and surfaced in the Desktop dashboard.

---

## 10. ROI Analytics

When `outcome_quality_score` is available from the Benchmark Engine, the Budget Engine computes:

```
roi_score = outcome_quality_score / total_cost_usd

cost_per_successful_task = total_cost_usd / (request_count * success_rate)
```

These metrics are available in the cost analytics views and the Desktop dashboard.

---

## 11. Data Model

### ai_ros_budget_rules
Budget configuration. See `AI-RESOURCE-DATABASE.md`.

### ai_ros_cost_ledger
Append-only cost records.

### ai_ros_budget_reservations
In-flight pre-dispatch cost reservations (cleared on response receipt).

### ai_ros_budget_positions
Materialized view: current spend per budget rule per period.

---

## 12. Events Published

| Event | Payload |
|-------|---------|
| `budget.check.passed` | `request_id`, `estimated_cost_usd`, `available_usd` |
| `budget.check.failed` | `request_id`, `rule_id`, `estimated_cost_usd`, `available_usd` |
| `budget.cost_recorded` | `request_id`, `total_cost_usd`, `provider_id`, `model_id` |
| `budget.warning` | `rule_id`, `spent_pct`, `spent_usd`, `limit_usd` |
| `budget.critical` | `rule_id`, `spent_pct`, `spent_usd`, `limit_usd` |
| `budget.exhausted` | `rule_id`, `org_id` |
| `budget.forecast.over_budget` | `rule_id`, `projected_spend_usd`, `limit_usd` |
| `budget.approval_required` | `request_id`, `estimated_cost_usd`, `threshold_usd` |
| `budget.recommendation` | `org_id`, `recommendation_text`, `estimated_savings_usd` |
| `budget.period_reset` | `rule_id`, `previous_spend_usd` |

---

## 13. Security

- Budget rules writable only by admin roles; all changes audit-logged
- Cost ledger is append-only
- Chargeback reports contain cost data only, never request content or credentials

---

## 14. Scalability

- Cost calculation is synchronous on the response path (< 1ms for token math)
- Ledger writes are async (durability within 1 second)
- Position views materialized in Redis (same pattern as Quota Engine)
- Chargeback report generation is a background job; large reports paginated

---

## 15. Future Evolution

| Feature | Notes |
|---------|-------|
| Multi-currency support | Convert from USD at daily exchange rates |
| Department approval workflows | Department budget owner approval for over-threshold requests |
| Automated budget recommendations | ML-based spend pattern analysis |
| Cost anomaly detection | Alert on unusual spend spikes |
| Subscription ROI dashboard | Track actual savings from flat-rate plans |

---

## 16. Cross References

| Document | Relationship |
|----------|-------------|
| AI-RESOURCE-OS-BLUEPRINT.md | Parent system |
| AI-MODEL-CATALOG.md | Pricing source |
| AI-QUOTA-ENGINE.md | Parallel enforcement gate |
| AI-ACCOUNT-MANAGER.md | Subscription coverage check |
| AI-POLICY-ENGINE.md | Approval policy rules |
| AI-RESOURCE-EVENTS.md | Event schemas |
| AI-RESOURCE-DATABASE.md | Table schemas |
| AI-RESOURCE-DESKTOP.md | Budget dashboard |
