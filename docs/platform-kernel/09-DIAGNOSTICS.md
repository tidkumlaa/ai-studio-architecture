# Platform Kernel — Diagnostics
# Phase 2.0D.2.6A

---

## 1. Overview

The kernel diagnostics layer provides a unified, always-on observability
surface covering:

- **Metrics** — counters, gauges, histograms for quantitative state
- **Health Checks** — qualitative component status (HEALTHY / DEGRADED / UNHEALTHY)
- **Distributed Tracing** — span-based request flow tracking
- **Structured Logging** — correlated, leveled, JSON-structured log records
- **Audit Log** — immutable, append-only record of significant actions
- **Performance Monitoring** — SLO tracking and degradation alerts
- **Debug Hooks** — custom diagnostic info injected at runtime

All diagnostic data is correlated via `correlation_id` and `trace_id`.

---

## 2. Metrics System

### 2.1 Metric Types

| Type | API | Use Case |
|------|-----|----------|
| Counter | `inc()`, `inc_by(n)` | Total events, errors, bytes |
| Gauge | `set(v)`, `inc()`, `dec()` | Current value, queue depth |
| Histogram | `observe(v)` + configurable buckets | Latency, size distributions |
| Summary | `observe(v)` + computed quantiles | Pre-aggregated p50/p95/p99 |
| Timer | context manager | Code block durations |

```python
# Counter
metrics.counter("kernel.tasks.completed.total", labels={"tier": "normal"}).inc()

# Gauge
metrics.gauge("kernel.scheduler.queue_depth").set(42)

# Histogram (latency in ms)
metrics.histogram(
    "kernel.tasks.duration_ms",
    buckets=[1, 5, 10, 25, 50, 100, 250, 500, 1000, 5000],
).observe(125.3)

# Timer (context manager)
with metrics.timer("kernel.plugin.load_duration_ms"):
    plugin = plugin_manager.load(manifest)
```

### 2.2 Labels

Labels add dimensions to metrics:
```python
metrics.counter(
    "kernel.events.published.total",
    labels={"event_type": event.event_type, "priority": event.priority.name},
).inc()
```

Label cardinality must stay bounded. Do not use unbounded values (object_ids,
user inputs) as label values.

### 2.3 Export Formats

```python
metrics.export_prometheus() → str     # Prometheus text format
metrics.export_json() → dict          # JSON for internal dashboards
metrics.snapshot() → list[MetricSample]
```

### 2.4 Metric Naming Conventions

```
{domain}.{subdomain}.{name}.{suffix}
  kernel.tasks.duration_ms             ← histogram (no suffix needed, unit in name)
  kernel.objects.created.total         ← counter (.total suffix)
  kernel.scheduler.queue_depth         ← gauge (no suffix)
  plugin.{plugin_id}.latency_ms        ← plugin-emitted
```

---

## 3. Health Check System

### 3.1 Health Status Model

```
HEALTHY   — Component operating normally, within SLOs
DEGRADED  — Component operating but with performance or reliability issues
UNHEALTHY — Component not operating; feature unavailable
UNKNOWN   — Health check has not yet run
```

Overall system health = worst status of all registered components.

### 3.2 Health Check Registration

```python
health.register_check(
    name="kernel.eventbus",
    check=lambda: _check_eventbus_health(event_bus),
    interval_ms=15_000,
)

def _check_eventbus_health(bus: PlatformEventBus) -> HealthCheckResult:
    dead_count = bus.dead_letter.count()
    if dead_count > 1000:
        return HealthCheckResult(
            component="kernel.eventbus",
            status=HealthStatus.UNHEALTHY,
            message=f"Dead letter queue overflowing: {dead_count} items",
            checked_at=datetime.now(timezone.utc),
            duration_ms=0.5,
            details={"dead_letter_count": dead_count},
        )
    elif dead_count > 100:
        return HealthCheckResult(..., status=HealthStatus.DEGRADED, ...)
    return HealthCheckResult(..., status=HealthStatus.HEALTHY, ...)
```

### 3.3 Built-In Kernel Health Checks

| Check Name | HEALTHY | DEGRADED | UNHEALTHY |
|------------|---------|----------|-----------|
| `kernel.eventbus` | dead letter < 100 | < 1000 | >= 1000 |
| `kernel.scheduler` | queue < 1000 | < 5000 | >= 5000 |
| `kernel.scheduler.stuck` | 0 stuck tasks | 1–5 stuck | > 5 stuck |
| `kernel.registry` | all registered objects healthy | any DEGRADED | any FAILED |
| `kernel.plugins` | 0 failed plugins | 1–2 failed | > 2 failed |
| `kernel.resources.memory` | < 80% quota | 80–95% | > 95% |
| `kernel.resources.ai_tokens` | < 80% session budget | 80–95% | > 95% |
| `kernel.config` | reload succeeded | last reload > 1h ago | reload failed |

### 3.4 Health Change Events

When a component's health status changes, PlatformEventBus receives:
```
kernel.health.status.changed
  payload:
    component: "kernel.scheduler"
    old_status: "healthy"
    new_status: "degraded"
    message: "Queue depth is 1500"
    details: { queue_depth: 1500, threshold: 1000 }
```

---

## 4. Distributed Tracing

### 4.1 Span Model

```python
class TraceSpan:
    span_id: str         # unique per span
    trace_id: str        # shared across all spans in a trace
    parent_id: str | None  # parent span_id for nesting
    operation: str
    started_at: datetime
    ended_at: datetime | None
    status: str          # "ok" | "error" | "unset"
    attributes: dict[str, Any]
    events: list[dict]   # named points in time within the span
```

### 4.2 Creating Spans

```python
# Using context manager
with diagnostics.trace("analyze-repository", correlation_id=ctx.correlation_id) as span:
    span.set_attribute("repository_id", repo.repository_id)
    span.set_attribute("path", repo.path)
    result = run_analysis(repo)
    span.set_attribute("symbols_found", result.symbol_count)

# Nested spans
with diagnostics.trace("outer-operation") as outer:
    with diagnostics.trace("inner-operation") as inner:
        inner.set_attribute("step", 1)
        do_inner_work()
```

### 4.3 Trace Propagation

Spans inherit trace_id from PlatformContext:
```python
ctx = kernel.create_context(correlation_id="flow-42")
# All spans created using ctx carry the same trace_id
```

When a task is dispatched across threads, the context (and its trace_id)
is captured and propagated into the new thread automatically.

### 4.4 Span Export

Spans are written to PlatformAudit with `action="trace.span"`.
They can also be exported to an external tracing backend via sink adapters.

---

## 5. Structured Logging

### 5.1 Log Record Structure

Every log record is a structured dictionary:
```json
{
  "timestamp": "2026-06-29T13:00:00.000000Z",
  "level": "INFO",
  "logger": "kernel.scheduler",
  "message": "Task submitted",
  "correlation_id": "flow-42",
  "trace_id": "trace-abc",
  "span_id": "span-xyz",
  "task_id": "task-001",
  "priority": 5,
  "source_file": "platform/kernel/scheduler.py",
  "source_line": 142
}
```

### 5.2 Log Levels and When to Use

| Level | Use |
|-------|-----|
| TRACE | Per-item processing loops, ultra-verbose internals |
| DEBUG | Component state changes, configuration values |
| INFO | Major lifecycle events, user-visible operations |
| WARNING | Recoverable issues, degraded performance, approaching limits |
| ERROR | Operation failed; caller can recover |
| CRITICAL | Kernel invariant violated; shutdown imminent |

### 5.3 Logger Hierarchy

```
kernel                          (root kernel logger)
  kernel.scheduler
  kernel.eventbus
  kernel.registry
  kernel.security
  kernel.plugins
    kernel.plugins.{plugin_id}
  kernel.resources
```

Child loggers inherit the parent's effective level unless overridden.

### 5.4 Log Sinks

```python
class LogSink(ABC):
    @abstractmethod
    def emit(self, record: LogRecord) -> None: ...

class ConsoleSink(LogSink):
    # Text output to stdout, formatted per format string

class JsonFileSink(LogSink):
    # Rotating JSON log file

class EventBusSink(LogSink):
    # Emits log records as kernel.log.* events (for real-time monitoring)
    # Only for WARNING and above to avoid feedback loops
```

---

## 6. Audit Log

Full specification in `03-MODULE-CATALOG.md § F4`.

### 6.1 What Must Be Audited

All of the following must produce an AuditRecord:

| Category | Action Examples |
|----------|----------------|
| Authentication | login, logout, token_refresh |
| Authorization | access_granted, access_denied |
| Capability | capability_granted, capability_revoked |
| Plugin | plugin_loaded, plugin_unloaded, plugin_failed |
| Lifecycle | object_created, object_disposed, state_transitioned |
| Configuration | config_changed, config_reloaded |
| Resource | quota_exceeded, budget_exhausted |
| Security | secret_accessed, sandbox_violation |
| Data | workspace_created, workspace_deleted, project_deleted |

### 6.2 Audit Query Examples

```python
# All access denials in the last hour
audit.query(
    action_pattern="kernel.security.access.*",
    outcome="denied",
    since=datetime.now() - timedelta(hours=1),
)

# Everything a specific session did
audit.query(session_id="sess-42")

# All plugin loads ever
audit.query(action_pattern="kernel.plugin.loaded")
```

---

## 7. Performance Monitoring

### 7.1 SLO Tracking

The kernel declares SLOs (Service Level Objectives) for its own operations:

```python
@dataclass
class KernelSLO:
    operation: str
    target_ms: float
    warning_threshold_ms: float   # emit warning if exceeded
    error_threshold_ms: float     # emit error event if exceeded

KERNEL_SLOS: list[KernelSLO] = [
    KernelSLO("object.creation",        target_ms=1.0,   warning=2.0,   error=10.0),
    KernelSLO("event.dispatch",         target_ms=0.1,   warning=0.5,   error=5.0),
    KernelSLO("service.lookup",         target_ms=0.05,  warning=0.2,   error=1.0),
    KernelSLO("lifecycle.transition",   target_ms=0.5,   warning=2.0,   error=10.0),
    KernelSLO("plugin.load",            target_ms=50.0,  warning=100.0, error=500.0),
    KernelSLO("config.read",            target_ms=0.1,   warning=0.5,   error=5.0),
    KernelSLO("permission.check",       target_ms=0.1,   warning=0.5,   error=5.0),
    KernelSLO("metric.emit",            target_ms=0.01,  warning=0.05,  error=0.5),
    KernelSLO("log.write",              target_ms=0.05,  warning=0.2,   error=2.0),
]
```

When an operation exceeds a threshold:
```
kernel.performance.degraded
  payload:
    operation: "event.dispatch"
    target_ms: 0.1
    actual_ms: 3.2
    severity: "error"
    correlation_id: "flow-42"
```

### 7.2 Debug Hooks

```python
# Register a custom debug hook
diagnostics.register_debug_hook(
    name="my-plugin.state",
    hook=lambda: {
        "queue_depth": my_plugin.queue.depth,
        "last_processed": my_plugin.last_processed_at.isoformat(),
        "error_count": my_plugin.error_count,
    },
)

# The hook output is included in diagnostic reports
report = diagnostics.diagnostic_report()
report.custom_sections["my-plugin.state"]  # → dict above
```

---

## 8. DiagnosticReport

```python
@dataclass
class DiagnosticReport:
    generated_at: datetime
    
    # System identity
    kernel_version: str
    platform_id: str
    
    # Metrics snapshot
    metrics: list[MetricSample]
    
    # Health summary
    overall_health: HealthStatus
    health_checks: dict[str, HealthCheckResult]
    
    # Active traces
    active_spans: list[TraceSpan]
    
    # Lifecycle overview
    lifecycle_counts: dict[str, int]   # state → count
    
    # Recent errors (last 100)
    recent_errors: list[AuditRecord]
    
    # Resource usage
    resource_usage: dict[str, float]
    
    # Custom sections from debug hooks
    custom_sections: dict[str, dict]
    
    def to_json(self) -> str: ...
    def to_text(self) -> str: ...   # human-readable summary
```
