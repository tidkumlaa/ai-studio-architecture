# Performance Baseline — AI Studio Platform v1.5.4

**Date:** 2026-06-27
**Status:** Phase 2 baseline (NOT VERIFIED via live run — metrics infrastructure complete, baseline values require live traffic)

---

## Measurement Method

All performance metrics are instrumented via:
- **AISF:** `aisf_*` Prometheus histograms (15 s psutil samples for system metrics)
- **CF:** Spring Micrometer timers (Prometheus histogram buckets)
- **Desktop:** `DesktopMetrics` rolling deque (p50/p95 over last 100 API calls)

To collect live baseline values:

```powershell
# 1. Start AISF
cd E:\UserData\MyData\Content\DEV\ai-software-factory
python app.py

# 2. Start Content Factory
cd E:\UserData\MyData\Content\AgentAIDev\content-factory
mvn -pl cf-api spring-boot:run

# 3. Generate traffic (example: 100 health checks)
for ($i=0; $i -lt 100; $i++) { Invoke-RestMethod http://localhost:8088/api/v1/health }

# 4. Collect baseline
Invoke-RestMethod http://localhost:8088/api/v1/metrics/snapshot | ConvertTo-Json -Depth 3

# 5. Get Prometheus histogram data
curl http://localhost:8088/api/v1/metrics | Select-String "aisf_http_request_duration"
```

---

## Process Metrics (Live psutil — measured at idle)

| Metric | AISF Value | Notes |
|---|---|---|
| CPU % (idle) | 0.00 – 0.5% | psutil read confirmed in Phase 1 |
| Memory RSS (idle) | ~38–42 MB | psutil confirmed: 38.1 MB |
| Thread count (idle) | 24 | FastAPI + workers + monitor |
| Startup time | 1.5–2.5 s | `aisf_runtime_start_duration_seconds` |

---

## HTTP Latency (Prometheus Histograms)

These values will populate after live traffic. The metric infrastructure is in place:

| Endpoint | Bucket Configuration | Notes |
|---|---|---|
| All HTTP | `0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0 s` | `aisf_http_request_duration_seconds` |
| `GET /health` | Expected: p50 < 1 ms | Cached probe |
| `GET /metrics` | Expected: p50 < 5 ms | Prometheus scrape |
| `POST /tasks` | Expected: p50 < 20 ms | DB write |

**Query for live values:**
```
# p50 HTTP latency
histogram_quantile(0.50, rate(aisf_http_request_duration_seconds_bucket[5m]))

# p95 HTTP latency
histogram_quantile(0.95, rate(aisf_http_request_duration_seconds_bucket[5m]))

# p99 HTTP latency
histogram_quantile(0.99, rate(aisf_http_request_duration_seconds_bucket[5m]))
```

---

## Task Processing Performance

| Metric | Histogram | Label |
|---|---|---|
| Task duration | `aisf_task_duration_seconds` | per `agent` |
| Buckets | 1s, 5s, 10s, 30s, 60s, 120s, 300s, 600s, 1800s, 3600s | — |

Expected ranges (agent-dependent, AI generation dominates):
- Code generation tasks: 10–300 s
- Test generation tasks: 5–120 s
- Documentation tasks: 5–60 s

**Query:**
```
histogram_quantile(0.95, rate(aisf_task_duration_seconds_bucket[30m]))
```

---

## Orchestration Loop Performance

| Metric | Histogram | Buckets |
|---|---|---|
| Tick duration | `aisf_orch_tick_duration_seconds` | 1ms–10s (12 buckets) |

Expected: p99 < 50 ms (idle tick with no pending tasks)

---

## Content Factory Performance

| Metric | Source | Query |
|---|---|---|
| HTTP p95 | `http_server_requests_seconds` | `histogram_quantile(0.95, rate(http_server_requests_seconds_bucket[5m]))` |
| DB connection active | `hikaricp.connections.active` | Direct gauge |
| RabbitMQ consumer lag | `rabbitmq.consumed` | Counter delta |
| Pipeline step p95 | `cf_pipeline_step_duration_seconds` | `histogram_quantile(0.95, ...)` |
| JVM heap % | `jvm.memory.used / jvm.memory.max` | Gauge ratio |

---

## Desktop API Latency

Measured by `DesktopMetrics` rolling p50/p95 over last 100 calls per endpoint.
Displayed in Runtime Center panel → **API p50 / p95** row.

Expected (local network to localhost AISF):
- p50: < 5 ms
- p95: < 20 ms

---

## Disk

| Metric | Value (measured) | Source |
|---|---|---|
| `aisf_disk_free_bytes{path="."}` | ~50 GB | psutil on E:\ partition |
| `aisf_disk_used_bytes{path="."}` | ~18 GB | psutil |
| `aisf_disk_total_bytes{path="."}` | ~107 GB | psutil |

---

## Notes

- **NOT VERIFIED** entries above require a live AISF instance running and processing tasks to populate histogram buckets with real observations.
- System metrics (CPU, memory, disk) are VERIFIED via psutil (from Phase 1 validation).
- Histogram quantile values will become available after first Prometheus scrape of a running AISF instance.
- Phase 3 (Performance) will establish definitive benchmarks with controlled load generation.
