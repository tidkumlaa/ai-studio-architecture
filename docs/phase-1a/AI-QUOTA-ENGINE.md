# AI Quota Engine — Usage Tracking and Limit Enforcement

**Version:** 1.0.0  
**Status:** Approved for Implementation  
**Phase:** 1A  
**Date:** 2026-06-29  
**Classification:** Internal Architecture

---

## 1. Purpose

The Quota Engine tracks all AI resource consumption against provider-imposed limits and organization-configured quotas. It enforces these limits before any request is dispatched to a provider, provides real-time position reporting, forecasts depletion, and triggers notifications and automated fallbacks when limits approach.

---

## 2. Responsibilities

- Real-time tracking of tokens, requests, images, and audio minutes consumed
- Provider quota enforcement (rate limits, daily/monthly limits)
- Organization quota configuration and enforcement
- Pre-dispatch quota availability check (hard gate before scheduling)
- Quota position forecasting and depletion ETA
- Notification triggers (warning at configurable thresholds)
- Emergency fallback provider activation when primary quota is exhausted
- Quota ledger (immutable deduction records)
- Reset schedule management (minute, hourly, daily, weekly, monthly)

---

## 3. Architecture

```
Routing Engine (pre-dispatch)
        │
        │ check_quota(org_id, provider_id, model_id, estimated_tokens)
        ▼
┌───────────────────────────────────────────────────────────────────┐
│                        Quota Engine                                │
│                                                                    │
│  ┌──────────────────┐   ┌──────────────────┐   ┌──────────────┐  │
│  │  Quota Registry  │   │  Ledger Writer   │   │  Forecast    │  │
│  │  (config store)  │   │  (append-only)   │   │  Engine      │  │
│  └────────┬─────────┘   └────────┬─────────┘   └──────┬───────┘  │
│           │                      │                     │           │
│  ┌────────▼──────────────────────▼─────────────────────▼────────┐  │
│  │                      Quota Core                                │  │
│  │                                                                │  │
│  │  check(key, amount) → QuotaCheckResult                        │  │
│  │  deduct(key, amount, request_id)                              │  │
│  │  refund(key, amount, request_id)  ← on cancellation/error     │  │
│  │  position(key) → QuotaPosition                                │  │
│  │  forecast(key) → DepletionForecast                            │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌──────────────────┐   ┌──────────────────────────────────────┐  │
│  │  Reset           │   │  Alert Manager                        │  │
│  │  Scheduler       │   │  (warning / critical / exhausted)     │  │
│  └──────────────────┘   └──────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────┘
         │                          │
         ▼                          ▼
  QuotaLedger table          Event Bus
  (PostgreSQL)               (notifications)
```

---

## 4. Quota Dimensions

### 4.1 Quota Key

Every quota is identified by a hierarchical key:

```
QuotaKey:
  scope:       ORG | PROJECT | USER | PROVIDER
  org_id:      UUID
  project_id:  UUID | None
  user_id:     UUID | None
  provider_id: str         (e.g., "anthropic")
  model_id:    str | None  (e.g., "anthropic/claude/claude-opus-4-8"; None = all models)
  resource:    QuotaResource
  window:      QuotaWindow
```

### 4.2 QuotaResource

| Resource | Unit | Tracked Per |
|----------|------|------------|
| `INPUT_TOKENS` | tokens | request |
| `OUTPUT_TOKENS` | tokens | response |
| `TOTAL_TOKENS` | tokens | request + response |
| `REQUESTS` | count | request |
| `IMAGE_GENERATIONS` | count | request |
| `AUDIO_MINUTES` | seconds (stored as int) | request |
| `EMBEDDINGS_TOKENS` | tokens | request |

### 4.3 QuotaWindow

| Window | Reset Trigger |
|--------|--------------|
| `PER_MINUTE` | Rolling 60-second window |
| `PER_HOUR` | Rolling 60-minute window |
| `PER_DAY` | Resets at 00:00 UTC |
| `PER_WEEK` | Resets Monday 00:00 UTC |
| `PER_MONTH` | Resets 1st of month 00:00 UTC |
| `PER_BILLING_PERIOD` | Resets at subscription billing date |
| `LIFETIME` | Never resets (allocation caps) |

---

## 5. Quota Configuration

### 5.1 QuotaRule

| Field | Type | Description |
|-------|------|-------------|
| `rule_id` | `UUID` | Primary key |
| `org_id` | `UUID` | Scope |
| `project_id` | `UUID \| None` | Optional project scope |
| `user_id` | `UUID \| None` | Optional user scope |
| `provider_id` | `str` | Provider this rule applies to |
| `model_id` | `str \| None` | Model-specific rule (or all models) |
| `resource` | `QuotaResource` | What is being tracked |
| `window` | `QuotaWindow` | Reset period |
| `limit` | `int` | Hard cap |
| `warning_threshold_pct` | `int` | Percent at which to send warning (default: 80) |
| `critical_threshold_pct` | `int` | Percent at which to send critical alert (default: 95) |
| `on_exhaustion` | `ExhaustionAction` | What to do when limit hit |
| `priority` | `int` | Lower number = checked first in rule chain |
| `source` | `QuotaSource` | `PROVIDER_IMPOSED` / `ORG_CONFIGURED` / `SYSTEM_DEFAULT` |

### 5.2 ExhaustionAction

| Action | Behavior |
|--------|---------|
| `REJECT` | Return 429 to caller immediately |
| `QUEUE` | Place request in deferred queue until reset |
| `FALLBACK_PROVIDER` | Route to designated fallback provider |
| `FALLBACK_MODEL` | Route to lower-tier model in same provider |
| `ALERT_ONLY` | Allow request but send alert (for soft limits) |

---

## 6. Core Operations

### 6.1 check() — Pre-Dispatch Gate

Called by the Routing Engine before any request is scheduled.

```
check(
  org_id: UUID,
  provider_id: str,
  model_id: str,
  resources: Dict[QuotaResource, int],
  project_id: UUID | None = None,
  user_id: UUID | None = None
) → QuotaCheckResult
```

**QuotaCheckResult:**

| Field | Type | Description |
|-------|------|-------------|
| `allowed` | `bool` | Whether the request may proceed |
| `blocking_rules` | `List[QuotaViolation]` | Which rules would be violated |
| `positions` | `List[QuotaPosition]` | Current position for all checked rules |
| `suggested_action` | `ExhaustionAction` | What to do if not allowed |
| `retry_after_seconds` | `int \| None` | For QUEUE action: estimated wait |

The check is atomic — it either passes all rules or fails. Partial pass is not possible.

### 6.2 deduct() — Post-Dispatch Accounting

Called by the Execution Engine after receiving the actual token counts from the provider response.

```
deduct(
  org_id: UUID,
  provider_id: str,
  model_id: str,
  request_id: UUID,
  resources: Dict[QuotaResource, int],
  project_id: UUID | None = None,
  user_id: UUID | None = None
)
```

Writes an immutable entry to the quota ledger. Updates rolling counters.

**Important:** The pre-dispatch check uses an *estimated* token count (based on input length). The deduction uses *actual* counts from the response. Any delta (estimate vs. actual) is corrected in the ledger.

### 6.3 refund() — Error Recovery

Called when a dispatched request fails, is cancelled, or times out before incurring actual usage.

```
refund(
  org_id: UUID,
  provider_id: str,
  model_id: str,
  request_id: UUID,
  resources: Dict[QuotaResource, int]
)
```

Writes a negative ledger entry. Rolling counters are decremented.

---

## 7. Provider-Imposed Quota Tracking

Provider quotas are imported from the Account Manager and updated when providers return rate limit headers:

| Provider | Rate Limit Headers |
|---------|-------------------|
| Anthropic | `x-ratelimit-limit-requests`, `x-ratelimit-limit-tokens`, `x-ratelimit-remaining-requests`, `x-ratelimit-remaining-tokens`, `x-ratelimit-reset-requests`, `x-ratelimit-reset-tokens`, `retry-after` |
| OpenAI | `x-ratelimit-limit-requests`, `x-ratelimit-limit-tokens`, `x-ratelimit-remaining-requests`, `x-ratelimit-remaining-tokens`, `x-ratelimit-reset-requests`, `x-ratelimit-reset-tokens` |
| Gemini | `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` |

On receiving a 429 response, the Quota Engine:
1. Updates `remaining` counters to the values from headers
2. Sets `reset_at` from the reset header
3. Returns `RATE_LIMITED` to the Routing Engine
4. The Routing Engine reroutes to an alternative provider

---

## 8. QuotaPosition

Real-time snapshot of a quota rule's state.

| Field | Type | Description |
|-------|------|-------------|
| `rule_id` | `UUID` | |
| `window` | `QuotaWindow` | |
| `limit` | `int` | |
| `used` | `int` | Current usage in this window |
| `remaining` | `int` | `limit - used` |
| `used_pct` | `float` | 0.0–100.0 |
| `reset_at` | `datetime` | When this window resets |
| `alert_level` | `AlertLevel` | `OK` / `WARNING` / `CRITICAL` / `EXHAUSTED` |

---

## 9. Forecast Engine

### 9.1 DepletionForecast

| Field | Type | Description |
|-------|------|-------------|
| `rule_id` | `UUID` | |
| `current_burn_rate` | `float` | Units per hour (rolling 24h average) |
| `projected_exhaustion_at` | `datetime \| None` | Null if burn rate would not exhaust limit |
| `confidence` | `float` | 0.0–1.0 |
| `basis_hours` | `int` | How many hours of history used |
| `recommendation` | `str` | Human-readable action suggestion |

### 9.2 Forecast Alerts

| Trigger | Action |
|---------|--------|
| Projected exhaustion < 24 hours | `quota.forecast.exhaustion_24h` event |
| Projected exhaustion < 6 hours | `quota.forecast.exhaustion_6h` event |
| Actual usage > `warning_threshold_pct` | `quota.warning` event |
| Actual usage > `critical_threshold_pct` | `quota.critical` event |
| Actual exhaustion | `quota.exhausted` event + trigger ExhaustionAction |

---

## 10. Reset Schedule

The Reset Scheduler runs a cron job per window type:

| Window | Cron Schedule |
|--------|--------------|
| PER_MINUTE | Every minute (rolling; Redis-based) |
| PER_HOUR | Top of every hour |
| PER_DAY | 00:00 UTC daily |
| PER_WEEK | Monday 00:00 UTC |
| PER_MONTH | 1st of month 00:00 UTC |
| PER_BILLING_PERIOD | Per-subscription date |

On reset:
1. Archive current window counters to history table
2. Zero rolling counters
3. Publish `quota.reset` event

For `PER_MINUTE` and `PER_HOUR` windows, a Redis sliding window (sorted set by timestamp) is used instead of resetting — this avoids thundering-herd behavior at reset boundaries.

---

## 11. Data Model

### ai_ros_quota_rules
Primary configuration table. See `AI-RESOURCE-DATABASE.md`.

### ai_ros_quota_ledger
Append-only ledger of all deductions and refunds.

| Column | Type | Notes |
|--------|------|-------|
| `ledger_id` | UUID | PK |
| `rule_id` | UUID | FK to quota_rules |
| `request_id` | UUID | AI ROS request ID |
| `entry_type` | enum | `DEDUCTION` / `REFUND` / `RESET` |
| `amount` | int | Units deducted (negative for refunds) |
| `recorded_at` | timestamptz | |
| `window_start` | timestamptz | Window this entry belongs to |

### ai_ros_quota_positions (materialized / Redis cache)
Current real-time position per rule. Rebuilt from ledger on startup; maintained in Redis for sub-millisecond reads.

---

## 12. Events Published

| Event | Trigger |
|-------|---------|
| `quota.check.passed` | Request cleared quota check |
| `quota.check.failed` | Request blocked by quota |
| `quota.deducted` | Usage recorded post-execution |
| `quota.refunded` | Usage refunded on error |
| `quota.warning` | Usage > warning threshold |
| `quota.critical` | Usage > critical threshold |
| `quota.exhausted` | Limit reached |
| `quota.reset` | Window reset |
| `quota.forecast.exhaustion_24h` | Forecast: exhaustion within 24h |
| `quota.forecast.exhaustion_6h` | Forecast: exhaustion within 6h |
| `quota.rule.created` | New quota rule configured |
| `quota.rule.modified` | Quota rule changed |

---

## 13. Security

- Quota rules are modified only by admin roles; all changes are audit-logged
- Quota ledger entries are immutable (append-only table, no UPDATE/DELETE grants)
- Cross-organization quota isolation enforced at query level (`WHERE org_id = ?`)

---

## 14. Scalability

- Redis sorted sets for sub-millisecond rate limit checks on PER_MINUTE and PER_HOUR windows
- PostgreSQL for PER_DAY and longer windows (lower frequency, bulk analytics)
- Quota check is on the critical path of every request; p99 must be < 5ms
- Ledger writes are async (fire-and-forget from Execution Engine perspective); durability guarantee within 1 second

---

## 15. Future Evolution

| Feature | Target |
|---------|--------|
| Token estimate improvement (smarter pre-dispatch estimation) | Phase 1A.3 |
| Dynamic quota increase request workflow | Phase 1A.4 |
| Cross-provider quota pooling | Phase 1A.4 |
| Real-time quota dashboard streaming via WebSocket | Phase 1A.3 |

---

## 16. Cross References

| Document | Relationship |
|----------|-------------|
| AI-RESOURCE-OS-BLUEPRINT.md | Parent system |
| AI-SCHEDULER.md | Quota check is prerequisite to scheduling |
| AI-BUDGET-ENGINE.md | Parallel enforcement gate alongside quota |
| AI-RESOURCE-EVENTS.md | Event schemas |
| AI-RESOURCE-DATABASE.md | Table schemas |
| AI-RESOURCE-DESKTOP.md | Quota dashboard |
