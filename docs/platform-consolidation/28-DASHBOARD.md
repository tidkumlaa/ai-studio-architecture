---
knowledge_id: KNW-PLAT-ARCH-028
title: "Platform Dashboard Architecture"
status: AUTHORITATIVE
phase: "2.1D.0"
version: "1.0.0"
created: "2026-06-29"
purpose: "Specify the desktop-only platform monitoring dashboard: panels, data sources, and refresh rates"
canonical_source: "architecture/docs/platform-consolidation/28-DASHBOARD.md"
dependencies:
  - "06-RUNTIME-MODEL.md"
  - "09-SERVICE-MODEL.md"
  - "27-PERFORMANCE-BUDGET.md"
related_documents:
  - "29-CLI.md"
  - "30-API.md"
acceptance_criteria:
  - "Dashboard is desktop-only (not web or mobile)"
  - "Every panel specifies its data source"
  - "Refresh rates do not exceed performance budgets"
verification_checklist:
  - "[ ] All panels have a data source"
  - "[ ] Desktop-only constraint enforced"
  - "[ ] No dashboard calls external APIs directly"
future_extensions:
  - "Custom panel plugins"
  - "Dashboard export to PDF for reports"
---

# Platform Dashboard Architecture

## Constraint: Desktop Only

The platform dashboard runs exclusively in the **desktop application**.
It is NOT a web dashboard, NOT accessible via browser, NOT deployed as a service.

Data flows:
```
platform runtime → MetricsService → desktop app → dashboard panels
```

The dashboard reads from `MetricsService`, `EventBus`, and `capability:ai:dashboard.snapshot:v1`.
It does NOT call external provider APIs.

---

## Dashboard Layout

```
╔══════════════════════════════════════════════════════════════╗
║  Platform Monitor                      [Refresh: 5s]  [■■□] ║
╠══════════════════╦═══════════════════╦══════════════════════╣
║  RUNTIME HEALTH  ║   RESOURCE USAGE  ║   INTELLIGENCE STATS ║
║  ─────────────── ║   ──────────────  ║   ─────────────────  ║
║  event    ● READY║   Quota: 72%      ║   Accuracy: 91.2%    ║
║  resource ● READY║   Budget: $4.23   ║   Trend: improving   ║
║  provider ● READY║   Tokens: 1.2M    ║   Predictions: 8,421 ║
║  knowledge● READY║   Cost today:$0.8 ║   Provider: Anthropic║
║  ai       ● READY║                   ║                      ║
╠══════════════════╩═══════════════════╩══════════════════════╣
║  ACTIVE EXECUTIONS                                          ║
║  ─────────────────────────────────────────────────────────  ║
║  [plan-001] RUNNING  multi-agent  est. $0.12  ETA: 45s     ║
║  [plan-002] QUEUED   single-agent est. $0.04  Wait: 12s    ║
╠═════════════════════════════════════════════════════════════╣
║  PROVIDER HEALTH                                            ║
║  ─────────────────────────────────────────────────────────  ║
║  anthropic  ● HEALTHY  p99: 2.1s  errors: 0.1%            ║
║  openai     ● HEALTHY  p99: 1.8s  errors: 0.2%            ║
║  google     ● HEALTHY  p99: 3.2s  errors: 0.0%            ║
╠═════════════════════════════════════════════════════════════╣
║  RECENT EVENTS                                    [clear]   ║
║  12:01:05 platform.ai.quota.consumed account=acc-001        ║
║  12:01:03 platform.ai.execution.completed plan=plan-000     ║
║  12:00:58 platform.provider.health.degraded provider=local  ║
╚═════════════════════════════════════════════════════════════╝
```

---

## Panel Specifications

### Panel 1 — Runtime Health
| Property | Value |
|----------|-------|
| Data source | `RuntimeHealth` dataclass per runtime |
| Refresh rate | 5 seconds |
| Display | State badge: ● READY (green) / ● DEGRADED (yellow) / ● FAILED (red) |
| API call | `capability:kernel:lifecycle:v1` → `get_runtime_health(runtime_id)` |

### Panel 2 — Resource Usage
| Property | Value |
|----------|-------|
| Data source | `capability:ai:dashboard.snapshot:v1` |
| Refresh rate | 10 seconds |
| Metrics | Quota %, monthly budget used, tokens today, cost today |
| API call | `ResourceIntelligenceDashboard.get_snapshot()` |

### Panel 3 — Intelligence Stats
| Property | Value |
|----------|-------|
| Data source | `capability:ai:learning.record:v1` → `LearningEngine.get_stats()` |
| Refresh rate | 30 seconds |
| Metrics | Overall accuracy, trend, total predictions, top provider |

### Panel 4 — Active Executions
| Property | Value |
|----------|-------|
| Data source | Execution plan state machine |
| Refresh rate | 2 seconds |
| Display | Plan ID, status, strategy, estimated cost, ETA |
| Events | Subscribe to `platform.ai.execution.*` |

### Panel 5 — Provider Health
| Property | Value |
|----------|-------|
| Data source | `capability:provider:health:v1` |
| Refresh rate | 15 seconds |
| Display | Health state, p99 latency, error rate |
| Alert | Visual alert when any provider enters DEGRADED |

### Panel 6 — Recent Events
| Property | Value |
|----------|-------|
| Data source | EventBus subscription to `platform.#` |
| Refresh rate | Real-time (event-driven) |
| Buffer | Last 100 events |
| Display | Timestamp, topic, key payload fields |

---

## Dashboard Data Service Interface

```python
class DashboardDataService(Protocol):
    def get_runtime_health(self) -> list[RuntimeHealth]: ...
    def get_resource_snapshot(self) -> ResourceSnapshot: ...
    def get_intelligence_stats(self) -> LearningStats: ...
    def get_active_executions(self) -> list[ExecutionPlanStatus]: ...
    def get_provider_health(self) -> list[ProviderHealth]: ...
    def get_recent_events(self, limit: int = 100) -> list[PlatformEvent]: ...
```

---

## ResourceSnapshot (dashboard-specific)

```python
@dataclass(frozen=True)
class ResourceSnapshot:
    quota_used_pct: float
    budget_used_usd: Decimal
    tokens_today: int
    cost_today_usd: Decimal
    top_account_id: str
    active_subscriptions: int
    captured_at: datetime
```

---

## Refresh Cadences

| Panel | Cadence | Mechanism |
|-------|---------|-----------|
| Runtime Health | 5s poll | Timer |
| Resource Usage | 10s poll | Timer |
| Intelligence Stats | 30s poll | Timer |
| Active Executions | 2s poll + event | Timer + EventBus |
| Provider Health | 15s poll | Timer |
| Recent Events | Real-time | EventBus subscription |

---

## Desktop Integration Notes

- Dashboard is a native desktop component (not HTML/JS)
- Rendered using the desktop app's UI framework
- `DashboardDataService` is injected via DI at desktop app boot
- Dashboard panels are independent — one panel failure does not crash others
- Dashboard respects `PlatformConfig.enable_dashboard` flag
