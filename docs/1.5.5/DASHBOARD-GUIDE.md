# Dashboard Guide — AI Studio Platform v1.5.4

Generated: 2026-06-27
Status: COMPLETE

---

## Overview

Five standalone HTML dashboards are provided under `E:\UserData\MyData\Content\DEV\dashboards\`.
All dashboards auto-refresh every 5 seconds (Content Pipeline: 10 s) and require no server —
open any file directly in a browser.

| File | Purpose | Data Source |
|---|---|---|
| `platform-overview.html` | KPIs + HTTP/CPU/Memory/Worker charts | AISF `/api/v1/metrics/snapshot` |
| `workers-dashboard.html` | Worker status table + restart tracking | AISF snapshot + `/api/v1/workers/status` |
| `runtime-dashboard.html` | CPU, memory, disk, thread details | AISF snapshot |
| `ai-inference-dashboard.html` | Task counters, active vs completed | AISF snapshot |
| `content-pipeline-dashboard.html` | CF health + HTTP metrics | CF `/actuator/health`, `/actuator/metrics/*` |
| `desktop-dashboard.html` | AISF connection from Desktop perspective | AISF snapshot |

---

## How to Use

### Prerequisites

| Service | URL | Start Command |
|---|---|---|
| AISF | `http://localhost:8088` | `cd ai-software-factory && python app.py` |
| Content Factory | `http://localhost:8080` | `cd content-factory && mvn -pl cf-api spring-boot:run` |

### Open Dashboards

1. Open a dashboard HTML file in any modern browser (Chrome, Edge, Firefox).
2. The dashboard immediately polls the backend and starts rendering charts.
3. A green dot in the header means the backend is reachable; red means offline.
4. Charts accumulate up to 60 data points (5 min of history at 5 s intervals).

---

## Dashboard Details

### Platform Overview (`platform-overview.html`)

**8 KPI cards:**
- HTTP Requests Total — cumulative counter across all endpoints
- Active Workers — `workers_running / workers_total`
- CPU % — AISF process CPU (psutil)
- Memory RSS — AISF process memory
- Active Tasks — tasks currently in flight
- Queue Depth — pending tasks
- Uptime — AISF process uptime since startup
- Disk Free — storage partition available space

**4 live charts (5 s interval):**
- HTTP Request Rate (requests/second, computed from delta)
- Workers Running
- CPU %
- Memory MB

### Workers Dashboard (`workers-dashboard.html`)

**3 KPI cards:** Running count, Total Restarts, Active Tasks

**Worker Status Table** — calls `/api/v1/workers/status` and renders:
- Agent key
- Status (RUNNING / DEAD)
- Uptime (time since last start)
- Restart count (start_count - 1)

**2 charts:** Workers running over time, Active tasks over time

### Runtime Dashboard (`runtime-dashboard.html`)

**8 KPI cards:** CPU, Memory RSS, Threads, Uptime, Disk Used, Disk Free, Boot time (ms), Last stop time (ms)

**4 charts:** CPU %, Memory MB, Thread count, Disk Used MB

### AI Inference Dashboard (`ai-inference-dashboard.html`)

**4 KPI cards:** Total tasks completed, Active tasks, Failed, Retried

**2 charts:** Tasks completed (cumulative), Active tasks

**Note:** Latency histograms (p50/p95/p99 per agent) are in Prometheus format at `GET /api/v1/metrics` — use Grafana with a Prometheus datasource for quantile queries (`histogram_quantile`).

### Content Pipeline Dashboard (`content-pipeline-dashboard.html`)

Polls Content Factory `/actuator/health` and `/actuator/metrics/http.server.requests`.

**CF Metrics available at `/actuator/prometheus`:**

```
# Pipeline step timing (per step, per success)
cf_pipeline_step_duration_seconds_count{step, success}
cf_pipeline_step_duration_seconds_sum{step, success}

# AI inference (per provider, model, success)
cf_ai_inference_duration_seconds_count{provider, model, success}
cf_ai_inference_total_total{provider, model, success}

# Episode lifecycle
cf_episodes_active
cf_episode_duration_seconds_count{success}
cf_episodes_completed_total_total{success}
cf_episodes_started_total_total

# Publishing
cf_publishing_duration_seconds_count{platform, success}

# Reliability
cf_retries_total_total{step}
cf_failures_total_total{step, error_type}

# Worker pool
cf_workers_active
```

### Desktop Dashboard (`desktop-dashboard.html`)

Shows AISF connectivity and key counters from the Desktop's perspective.

**Desktop in-process metrics** (CPU, memory, thread count, API latency p50/p95, startup time) are displayed inside the Desktop application itself — **Runtime Center → Live Process Metrics** section. They update every 5 seconds via `QTimer`. They are not served over HTTP.

---

## Prometheus Scrape Configuration (Reference)

To connect Grafana + Prometheus:

```yaml
# prometheus.yml
scrape_configs:
  - job_name: aisf
    static_configs:
      - targets: ['localhost:8088']
    metrics_path: /api/v1/metrics

  - job_name: content_factory
    static_configs:
      - targets: ['localhost:8080']
    metrics_path: /actuator/prometheus
```

Then in Grafana:
- Add Prometheus datasource pointing to `http://localhost:9090`
- Use `aisf_*` metrics for AISF panels
- Use `cf_*` and `jvm_*` for Content Factory panels
- Histogram quantile: `histogram_quantile(0.95, rate(aisf_task_duration_seconds_bucket[5m]))`

---

## Metric Field Reference (JSON Snapshot)

`GET http://localhost:8088/api/v1/metrics/snapshot` returns:

```json
{
  "service": "ai-software-factory",
  "version": "1.5.4",
  "http_requests_total": 1042,
  "workers_total": 10,
  "workers_running": 10,
  "worker_restarts_total": 0,
  "tasks_active": 2,
  "tasks_completed_total": 387,
  "tasks_failed_total": 0,
  "tasks_retried_total": 1,
  "queue_length": 5,
  "cpu_percent": 3.2,
  "memory_bytes": 42991616,
  "threads_total": 24,
  "uptime_seconds": 3742.1,
  "disk_free_bytes": 52428800000,
  "disk_used_bytes": 18874368000,
  "disk_total_bytes": 107374182400,
  "runtime_start_duration_seconds": 1.83,
  "runtime_stop_duration_seconds": null
}
```

All values are real — sourced from psutil process data, Python counters, and orchestrator state.
No mocked or static values are present.
