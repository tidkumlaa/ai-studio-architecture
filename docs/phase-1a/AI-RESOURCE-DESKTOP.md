# AI Resource Desktop — Dashboard and Management UI Specification

**Version:** 1.0.0  
**Status:** Approved for Implementation  
**Phase:** 1A  
**Date:** 2026-06-29  
**Classification:** Internal Architecture

---

## 1. Purpose

This document specifies all Desktop UI panels for AI ROS management and monitoring. The Desktop is the control plane for AI Studio administrators and power users: it provides visibility into provider health, quota and budget position, routing decisions, conversation exploration, benchmark performance, and credential management — all in real time.

The Desktop panels are implemented within the existing AI Studio Desktop application (Electron-based). They communicate with AI ROS exclusively through the AI Resource API (REST + WebSocket). No Desktop panel may call provider APIs directly.

---

## 2. Panel Inventory

| Panel | ID | Primary Users | Key Purpose |
|-------|----|--------------|-------------|
| AI ROS Overview | `ai-ros-overview` | All | System health at a glance |
| Provider Management | `ai-ros-providers` | Org Admin | Register, monitor, configure providers |
| Model Browser | `ai-ros-models` | All | Browse model catalog; compare capabilities |
| Quota Monitor | `ai-ros-quota` | Org Admin, Billing Admin | Real-time quota positions and forecasts |
| Budget Manager | `ai-ros-budget` | Billing Admin | Budgets, spend tracking, chargeback reports |
| Routing Monitor | `ai-ros-routing` | Org Admin | Live routing decisions; policy impact |
| Execution Monitor | `ai-ros-executions` | Org Admin | In-flight requests; history; failures |
| Conversation Explorer | `ai-ros-conversations` | Org Member | Browse, search, export sessions |
| Benchmark Dashboard | `ai-ros-benchmarks` | Org Admin | Model performance; regression alerts |
| Analytics | `ai-ros-analytics` | Billing Admin | Cost trends; usage patterns; ROI |
| Policy Manager | `ai-ros-policies` | Policy Admin | Create, manage, test policies |
| Credential Manager | `ai-ros-credentials` | Credential Admin | API keys; OAuth; rotation |
| Approval Queue | `ai-ros-approvals` | Approvers | Pending approval requests |
| Audit Log Viewer | `ai-ros-audit` | Org Admin | Tamper-evident audit trail |

---

## 3. Panel: AI ROS Overview

**Route:** `/ai-ros`  
**Refresh:** Live via WebSocket (no polling)

### 3.1 Layout

```
┌──────────────────────────────────────────────────────────────────────┐
│  AI Resource OS  v1.0.0      [Status: OPERATIONAL]    [Last sync: 2s] │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────┐ │
│  │  Providers   │  │  Requests    │  │  Today Spend │  │  Quota   │ │
│  │  4 / 4 OK    │  │  247 active  │  │   $12.84     │  │ 67% used │ │
│  │  [1 degraded]│  │  12 queued   │  │  of $50 limit│  │          │ │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────┘ │
│                                                                        │
│  Provider Health                          Active Routing               │
│  ┌────────────────────────────────┐    ┌─────────────────────────┐   │
│  │ ● anthropic    HEALTHY  423ms  │    │ claude-sonnet-4-6   64% │   │
│  │ ● openai       HEALTHY  612ms  │    │ claude-haiku-4-5    28% │   │
│  │ ◐ gemini       DEGRADED 2100ms │    │ gpt-4o-mini          8% │   │
│  │ ● ollama       HEALTHY   38ms  │    └─────────────────────────┘   │
│  └────────────────────────────────┘                                   │
│                                                                        │
│  Recent Alerts                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │ ⚠ 14:32  Gemini degraded — p99 latency 2100ms (baseline: 820ms) │ │
│  │ ℹ 14:15  Budget warning: project "content-gen" at 82% of monthly │ │
│  │ ℹ 13:58  Quota reset: anthropic INPUT_TOKENS PER_DAY             │ │
│  └───────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘
```

### 3.2 Data Sources

| Element | API Call | WebSocket Topic |
|---------|----------|----------------|
| Provider health tiles | `GET /providers` | `provider.health.*` |
| Active requests count | `GET /requests?status=in_flight` | `execution.submitted`, `execution.completed` |
| Today spend | `GET /budget` | `budget.cost_recorded` |
| Quota % | `GET /quota` | `quota.deducted` |
| Routing distribution | `GET /benchmarks/scores?window_days=1` | — |
| Recent alerts | `GET /audit-log?event_type=quota.warning,budget.warning,provider.health.*&limit=10` | All alert event types |

---

## 4. Panel: Provider Management

**Route:** `/ai-ros/providers`  
**Permissions:** `org_admin`

### 4.1 Provider List View

Each provider shown as a card with:
- Status indicator (green/amber/red) + real-time health
- Latency gauge (p50, p95, p99 from last 24h)
- Circuit breaker state badge (CLOSED / OPEN / HALF_OPEN)
- Model count
- Credentials registered indicator (yes/no; never shows key values)
- Actions: Edit | Test Connection | Disable | View Models

### 4.2 Register Provider Dialog

Fields:
- Provider name (select from known providers OR "Custom")
- Adapter class (auto-filled for known providers)
- Credential type (API Key / OAuth / None)
- API key input (masked; stored immediately in vault on save)
- Test connection button (runs `validate_credential` before saving)
- Display name, notes

### 4.3 Provider Health Detail Page

For each provider:
- Latency time series (24h, 7d, 30d)
- Error rate time series
- Circuit breaker event log
- Model availability matrix (model × time)
- Recent quota header extractions from provider responses

### 4.4 Model Browser (sub-view)

For a selected provider:
- Table: Model name | Context window | Quality tier | Latency class | Input cost | Output cost | Capabilities (chip set) | Status
- Filter by: capability, quality tier, status
- Click model → model detail page with benchmark scores

---

## 5. Panel: Quota Monitor

**Route:** `/ai-ros/quota`  
**Refresh:** Live via WebSocket

### 5.1 Layout

```
┌──────────────────────────────────────────────────────────────────────┐
│  Quota Monitor                             [Filter: Provider ▾] [Org ▾]│
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Anthropic Claude                                                      │
│  INPUT_TOKENS · PER_DAY                                               │
│  ████████████████████░░░░░░░░░░░░   3.2M / 10M  (32%)                │
│  Reset in: 9h 41m   ▶ Forecast: will NOT exhaust today               │
│                                                                        │
│  REQUESTS · PER_MINUTE                                                 │
│  ████████░░░░░░░░░░░░░░░░░░░░░░░░   8 / 60  (13%)   ↑ live          │
│                                                                        │
│  OpenAI GPT                                                            │
│  INPUT_TOKENS · PER_DAY                                               │
│  ████████████████████████████████   9.8M / 10M  (98%) ⚠ CRITICAL    │
│  Reset in: 9h 41m   ▶ Forecast: EXHAUSTION in 1h 12m               │
│                                                                        │
│  History                                                               │
│  [7-day stacked area chart: usage per provider per day]               │
│                                                                        │
│  Quota Rules  [+ Add Rule]                                             │
│  [Table: Rule | Provider | Resource | Window | Limit | Source | Actions] │
└──────────────────────────────────────────────────────────────────────┘
```

### 5.2 Interactions

- Click on any quota bar → expand to show per-project breakdown
- "Add Rule" → dialog to create a new quota rule (org_admin only)
- "Edit" on existing rule → update limit, thresholds, exhaustion action

### 5.3 Alert Callouts

- `WARNING` (≥80%): amber bar, amber text
- `CRITICAL` (≥95%): red bar, red text, pulsing alert dot
- `EXHAUSTED`: red fill, hard stop icon, next reset countdown

---

## 6. Panel: Budget Manager

**Route:** `/ai-ros/budget`  
**Permissions:** `billing_admin`

### 6.1 Budget Overview

- Card per active budget rule showing: name, period, limit, spent, remaining, burn rate, projected exhaustion
- Spending velocity indicator (daily burn vs. expected daily burn to stay within budget)

### 6.2 Spend Breakdown

Interactive treemap or donut chart, group by:
- Project
- User
- Provider
- Model
- Task category

Date range picker (today / this week / this month / custom).

### 6.3 Cost Trend

Line chart showing daily spend over selected period, overlaid with:
- Budget limit line
- Projected spend (dotted)
- Budget exhaustion point (if applicable)

### 6.4 Chargeback Report

- Report builder: select date range, grouping dimensions, format (CSV / JSON)
- Download button
- Scheduled report: configure weekly/monthly email delivery

### 6.5 Budget Rules Management

Table: Rule | Scope | Period | Limit | Current Spend | Alert Level | Actions

"Add Budget Rule" dialog: select scope (org / project / department / user), period, limit, thresholds, exhaustion action.

### 6.6 Cost Optimization Recommendations

Card list showing AI-generated recommendations:
```
💡 Route summary tasks to claude-haiku-4-5
   Potential saving: $38/month · Confidence: high
   [View Details] [Dismiss]

💡 Enable prompt caching for Content Factory system prompts
   Potential saving: $22/month · Confidence: medium
   [View Details] [Dismiss]
```

---

## 7. Panel: Routing Monitor

**Route:** `/ai-ros/routing`  
**Refresh:** Live

### 7.1 Live Routing Feed

Real-time stream of routing decisions (last 100, auto-updating):

```
14:32:01  code.generation    claude-sonnet-4-6  (anthropic)  $0.031  1.8s  ✓
14:32:00  content.article    gpt-4o-mini        (openai)     $0.004  0.9s  ✓
14:31:58  game.card_gen      claude-haiku-4-5   (anthropic)  $0.001  0.4s  ✓
14:31:55  code.review        claude-opus-4-8    (anthropic)  $0.184  4.2s  ✓
14:31:52  [POLICY_DENIED]    —                  —            —       —     ✗
```

Click any row → detail panel showing:
- Full routing rationale
- Candidate models considered
- Scores for each candidate
- Policies evaluated
- Final selection reason

### 7.2 Policy Impact View

For the selected time window:
- How many requests hit each policy
- How many were denied vs. constrained vs. passed
- "Simulated impact" toggle: show what would happen if a draft policy were activated

### 7.3 Provider Load Distribution

Pie/donut chart of requests by provider (last hour / day / week).

Stacked area chart: provider distribution over time (shows transitions when provider health changes).

---

## 8. Panel: Execution Monitor

**Route:** `/ai-ros/executions`  
**Refresh:** Live

### 8.1 Active Requests

Table of in-flight and recently queued requests:

| Request ID | Product | Task | Model | Status | Queued | Dispatched | Age |
|-----------|---------|------|-------|--------|--------|-----------|-----|
| ...short... | aisf | code.gen | claude-sonnet | IN_FLIGHT | 14:31:58 | 14:32:00 | 3s |
| ...short... | content-factory | content.article | gpt-4o-mini | QUEUED #3 | 14:32:01 | — | 1s |

Click row → full request detail (prompts, policy evaluation, routing, response, cost).

### 8.2 Failure Analysis

Tab showing failed requests:
- Error code distribution (bar chart)
- Retry rate by provider
- Timeout rate by model
- Error trend over time

### 8.3 Queue Depth

Live gauge: current queue depth per priority level. Alert if queue depth > configurable threshold.

---

## 9. Panel: Conversation Explorer

**Route:** `/ai-ros/conversations`  
**Permissions:** `org_member` (own sessions); `org_admin` (all sessions)

### 9.1 Session List

Table with search and filters:
- Filter by: product, user, status, date range, cost range
- Sort by: last active, message count, total cost
- Full-text search on session title

### 9.2 Session Detail

Full conversation view:
- Messages in chronological order with role badges (USER / ASSISTANT)
- Tool call expandable blocks showing input/output
- Per-message metadata: model used, latency, token count, cost
- Context compression events shown inline ("History compressed at this point — 28 messages summarized")
- Branch indicator: if this session is a branch, link to parent

### 9.3 Session Actions

- Export (JSON / Markdown / PDF)
- Branch from any message
- Archive
- Delete (admin only)

### 9.4 Context Position Indicator

Sidebar showing current context fill percentage and what's consuming tokens:
- System prompt: N tokens
- Memory injections: N tokens
- Tool definitions: N tokens
- Message history: N tokens / X messages
- Available for response: N tokens

---

## 10. Panel: Benchmark Dashboard

**Route:** `/ai-ros/benchmarks`

### 10.1 Score Matrix

Heatmap: models × task categories, colored by composite score (green = high, red = low).

Click any cell → score detail panel:
- Success rate, avg quality, p50/p95/p99 latency, avg cost, sample size
- 30-day trend chart

### 10.2 Regression Alerts

Alert list with severity badges:

```
🔴 CRITICAL  gemini-2-5-flash  code.generation  p95 latency +52%  since 2026-06-28
⚠  WARNING   gpt-4o            general.summary  quality -0.08     since 2026-06-27
```

Click → drill into the data points that triggered the alert; compare before/after.

### 10.3 Model Comparison

Side-by-side comparison of up to 4 models for a selected task category:
- Radar chart: quality, speed, cost, success rate
- Cost-efficiency scatter plot: quality score vs. cost per request

### 10.4 Synthetic Benchmark Runner

- List of available benchmark suites
- Last run result per suite
- "Run Now" button (triggers `POST /benchmarks/suites/{id}/run`)
- Progress indicator during run; results appear inline when complete

### 10.5 Recommendations

Cards showing which model is recommended per task category:
```
code.generation    →  claude-sonnet-4-6  (score: 0.91)  Cost: $0.028/req
content.article    →  claude-haiku-4-5   (score: 0.84)  Cost: $0.006/req
general.embedding  →  text-embedding-3-large  (score: 0.97)  Cost: $0.0002/req
```

---

## 11. Panel: Policy Manager

**Route:** `/ai-ros/policies`  
**Permissions:** `policy_admin`

### 11.1 Policy List

Table: Name | Type | Status | Priority | Matched (last 24h) | Denied (last 24h) | Actions

Toggle test_mode per policy without going into edit flow.

### 11.2 Policy Editor

Form-based policy builder:
- Policy type selector
- Condition builder (field + operator + value, add multiple conditions)
- Action selector (with parameter fields based on selected action)
- Priority field
- Effective from / until datetime pickers
- "Test Mode" toggle

### 11.3 Policy Impact Simulator

Before activating, test the policy against recent traffic:
- Feed last N hours of request history through the policy
- Show: how many would have matched, how many denied, which requests

### 11.4 Policy Evaluation History

For a selected policy: show every evaluation (match/no-match/denied) over time. Filter by time range. Export to CSV.

---

## 12. Panel: Credential Manager

**Route:** `/ai-ros/credentials`  
**Permissions:** `credential_admin`

### 12.1 Credential List

Table: Alias | Provider | Type | Status | Last Used | Expiry | Last Validated | Actions

No credential values shown anywhere in this panel.

Color coding: green (active, valid), amber (expiry < 30 days or not validated recently), red (expired or revoked).

### 12.2 Register Credential Dialog

- Provider selector
- Credential type (API Key / OAuth)
- For API Key: masked input field + "Test Connection" button
- For OAuth: "Connect via OAuth" button → opens provider OAuth flow in browser
- Alias (human label)
- Rotation policy configuration

### 12.3 Subscription Management

Sub-section for subscription state:
- Provider, Plan, Status, Billing period, Renewal date
- Included models list
- "Register Subscription" dialog

### 12.4 Rotation Schedule

Calendar view showing upcoming rotation events. "Rotate Now" button for manual trigger.

---

## 13. Panel: Approval Queue

**Route:** `/ai-ros/approvals`  
**Permissions:** Approver roles (configurable per policy)

### 13.1 Pending Approvals

Cards showing each pending approval request:

```
┌─────────────────────────────────────────────────────────┐
│ Approval Required                    Submitted: 2m ago   │
│                                                          │
│ Product: AI Software Factory                            │
│ Task: code.architecture                                 │
│ Estimated Cost: $8.42                                   │
│ Policy: "Require approval above $5"                     │
│                                                          │
│ Summary:                                                 │
│ Requesting full architecture review for new microservice│
│ design. 200K token context window request.              │
│                                                          │
│ Expires in: 47 minutes                                  │
│                                                          │
│  [Approve]    [Deny]    [View Details]                  │
└─────────────────────────────────────────────────────────┘
```

### 13.2 Approval Decision

- Approve: request proceeds immediately
- Deny: request fails with `APPROVAL_DENIED` error returned to product
- "View Details" shows assembled context summary (not full prompt content by default; requires explicit "Show Prompt" action which is audit-logged)

---

## 14. Panel: Audit Log Viewer

**Route:** `/ai-ros/audit`  
**Permissions:** `org_admin`

### 14.1 Log View

Table with filters: date range, event type, actor, resource type, outcome.

Each row: Timestamp | Event Type | Actor | Resource | Outcome | Details

### 14.2 Export

Export filtered log as CSV or JSON for compliance reporting.

### 14.3 Chain Integrity Status

Banner showing: "Audit chain verified ✓ — last verified 6h ago" or "⚠ Chain verification failed — contact security team".

---

## 15. Desktop Integration Requirements

### 15.1 Navigation

AI ROS panels are accessible from the Desktop sidebar under the "AI Resource OS" group. Sidebar shows:
- Live health indicator (green/amber/red dot)
- Unread alert count badge

### 15.2 Notifications

The Desktop notification system receives events from the WebSocket `ai-ros-alerts` topic:
- `quota.critical` → toast notification (amber)
- `quota.exhausted` → persistent alert (red)
- `budget.critical` → toast notification (amber)
- `provider.circuit_breaker.opened` → toast notification
- `credential.expiry_warning` → toast notification
- `benchmark.regression.detected` (CRITICAL) → toast notification

### 15.3 Deep Linking

Each panel and sub-view supports deep linking:
- `/ai-ros/providers/anthropic` → Provider detail
- `/ai-ros/quota?provider=anthropic` → Quota filtered to Anthropic
- `/ai-ros/conversations/{session_id}` → Session detail
- `/ai-ros/approvals/{approval_id}` → Specific approval card

### 15.4 Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `Cmd/Ctrl+Shift+A` | Open AI ROS Overview |
| `Cmd/Ctrl+Shift+Q` | Open Quota Monitor |
| `R` (in Execution Monitor) | Refresh execution list |

---

## 16. Implementation Roadmap

### Phase 1A.1 (Week 3–4)
- AI ROS Overview panel
- Provider Management panel (list + health)
- Credential Manager (register, list, status)

### Phase 1A.2 (Week 5–6)
- Quota Monitor (live)
- Budget Manager (positions, rules)
- Execution Monitor (active + history)

### Phase 1A.3 (Week 7–8)
- Benchmark Dashboard
- Routing Monitor
- Conversation Explorer

### Phase 1A.4 (Week 9–10)
- Policy Manager (full editor + simulator)
- Approval Queue
- Audit Log Viewer
- Cost optimization recommendations

---

## 17. Cross References

| Document | Relationship |
|----------|-------------|
| AI-RESOURCE-OS-BLUEPRINT.md | Parent system |
| AI-RESOURCE-API.md | All panels consume this API |
| AI-RESOURCE-EVENTS.md | WebSocket events driving live updates |
| AI-QUOTA-ENGINE.md | Quota panel data |
| AI-BUDGET-ENGINE.md | Budget panel data |
| AI-BENCHMARK-CENTER.md | Benchmark panel data |
| AI-POLICY-ENGINE.md | Policy panel data |
| AI-ACCOUNT-MANAGER.md | Credential panel data |
| AI-CONVERSATION-ENGINE.md | Conversation explorer data |
