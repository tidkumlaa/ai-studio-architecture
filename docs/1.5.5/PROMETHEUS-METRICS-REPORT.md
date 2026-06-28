# Prometheus Metrics Report — AI Studio Platform v1.5.4

**Date:** 2026-06-27
**Scope:** AISF + Content Factory Prometheus endpoints

---

## AISF — `GET /api/v1/metrics`

**URL:** `http://localhost:8088/api/v1/metrics`
**Content-Type:** `text/plain; version=0.0.4; charset=utf-8`
**Registry:** Default prometheus_client REGISTRY

### Sample Output (annotated)

```
# HELP aisf_http_requests_total Total HTTP requests handled
# TYPE aisf_http_requests_total counter
aisf_http_requests_total{method="GET",path="/health",status_code="200"} 42.0
aisf_http_requests_total{method="GET",path="/api/v1/workers",status_code="200"} 7.0
aisf_http_requests_total{method="POST",path="/api/v1/tasks/{id}/complete",status_code="200"} 3.0

# HELP aisf_http_request_duration_seconds HTTP request latency in seconds
# TYPE aisf_http_request_duration_seconds histogram
aisf_http_request_duration_seconds_bucket{method="GET",path="/health",le="0.001"} 38.0
aisf_http_request_duration_seconds_bucket{method="GET",path="/health",le="0.005"} 42.0
aisf_http_request_duration_seconds_bucket{method="GET",path="/health",le="+Inf"} 42.0
aisf_http_request_duration_seconds_sum{method="GET",path="/health"} 0.051
aisf_http_request_duration_seconds_count{method="GET",path="/health"} 42.0

# HELP aisf_workers_total Total number of registered agent workers
# TYPE aisf_workers_total gauge
aisf_workers_total 10.0

# HELP aisf_workers_running Number of agent workers currently alive
# TYPE aisf_workers_running gauge
aisf_workers_running 10.0

# HELP aisf_worker_restarts_total Total number of worker auto-restart events
# TYPE aisf_worker_restarts_total counter
aisf_worker_restarts_total{agent_key="CODE_GEN"} 0.0

# HELP aisf_worker_uptime_seconds Seconds since each worker last started
# TYPE aisf_worker_uptime_seconds gauge
aisf_worker_uptime_seconds{agent_key="CODE_GEN"} 3742.1
aisf_worker_uptime_seconds{agent_key="TEST_GEN"} 3741.8

# HELP aisf_tasks_active Tasks currently IN_PROGRESS
# TYPE aisf_tasks_active gauge
aisf_tasks_active 2.0

# HELP aisf_tasks_completed_total Tasks transitioned to REVIEW (success)
# TYPE aisf_tasks_completed_total counter
aisf_tasks_completed_total{agent="CODE_GEN"} 87.0
aisf_tasks_completed_total{agent="TEST_GEN"} 34.0

# HELP aisf_tasks_failed_total Tasks transitioned to STALLED (max retries exceeded)
# TYPE aisf_tasks_failed_total counter
aisf_tasks_failed_total{agent="CODE_GEN"} 0.0

# HELP aisf_tasks_retried_total Individual retry attempts
# TYPE aisf_tasks_retried_total counter
aisf_tasks_retried_total{agent="CODE_GEN"} 1.0

# HELP aisf_task_duration_seconds End-to-end task execution time in seconds
# TYPE aisf_task_duration_seconds histogram
aisf_task_duration_seconds_bucket{agent="CODE_GEN",le="1.0"} 2.0
aisf_task_duration_seconds_bucket{agent="CODE_GEN",le="5.0"} 28.0
aisf_task_duration_seconds_bucket{agent="CODE_GEN",le="30.0"} 85.0
aisf_task_duration_seconds_bucket{agent="CODE_GEN",le="+Inf"} 87.0
aisf_task_duration_seconds_sum{agent="CODE_GEN"} 1834.2
aisf_task_duration_seconds_count{agent="CODE_GEN"} 87.0

# HELP aisf_queue_length Number of tasks ready to be dispatched
# TYPE aisf_queue_length gauge
aisf_queue_length 5.0

# HELP aisf_orch_tick_duration_seconds Duration of a single orchestration tick
# TYPE aisf_orch_tick_duration_seconds histogram
aisf_orch_tick_duration_seconds_bucket{le="0.001"} 0.0
aisf_orch_tick_duration_seconds_bucket{le="0.01"} 312.0
aisf_orch_tick_duration_seconds_bucket{le="+Inf"} 374.0
aisf_orch_tick_duration_seconds_sum 18.74
aisf_orch_tick_duration_seconds_count 374.0

# HELP aisf_orch_tick_errors_total Exceptions raised during an orchestration tick
# TYPE aisf_orch_tick_errors_total counter
aisf_orch_tick_errors_total{step="dispatch"} 0.0

# HELP aisf_runtime_start_duration_seconds Time to bring orchestrator from cold start to ready
# TYPE aisf_runtime_start_duration_seconds gauge
aisf_runtime_start_duration_seconds 1.83

# HELP aisf_process_cpu_percent Process CPU utilisation
# TYPE aisf_process_cpu_percent gauge
aisf_process_cpu_percent 3.2

# HELP aisf_process_memory_bytes Process RSS memory in bytes
# TYPE aisf_process_memory_bytes gauge
aisf_process_memory_bytes 39845888.0

# HELP aisf_process_threads_total Number of OS threads in the process
# TYPE aisf_process_threads_total gauge
aisf_process_threads_total 24.0

# HELP aisf_process_uptime_seconds Process uptime in seconds
# TYPE aisf_process_uptime_seconds gauge
aisf_process_uptime_seconds 3742.1

# HELP aisf_disk_free_bytes Free disk space on the data volume in bytes
# TYPE aisf_disk_free_bytes gauge
aisf_disk_free_bytes{path="."} 52428800000.0

# HELP aisf_disk_used_bytes Used disk space on the data volume
# TYPE aisf_disk_used_bytes gauge
aisf_disk_used_bytes{path="."} 18874368000.0
```

### Path Normalization

`PrometheusMiddleware` (`factory/metrics/http_middleware.py`) normalizes request paths before using them as metric labels:

| Raw path | Normalized label |
|---|---|
| `/api/v1/tasks/01HXA3...ULID` | `/api/v1/tasks/{id}` |
| `/api/v1/episodes/550e8400-...UUID` | `/api/v1/episodes/{id}` |
| `/api/v1/workers/42` | `/api/v1/workers/{id}` |

This prevents cardinality explosion when each unique task/episode ID would otherwise become a distinct label value.

---

## Content Factory — `GET /actuator/prometheus`

**URL:** `http://localhost:8080/actuator/prometheus`
**Configuration:** `cf-api/src/main/resources/application.yml`
**Registry:** Spring Boot Micrometer `SimpleMeterRegistry` → `PrometheusMeterRegistry`

### Custom CF Metrics (ContentFactoryMetrics + PipelineMetricsAspect)

```
# TYPE cf_pipeline_step_duration_seconds_active gauge
# TYPE cf_pipeline_step_duration_seconds summary
cf_pipeline_step_duration_seconds_count{application="content-factory",step="StoryGenerationWorker.onMessage",success="true",} 52.0
cf_pipeline_step_duration_seconds_sum{application="content-factory",step="StoryGenerationWorker.onMessage",success="true",} 4821.4

# TYPE cf_pipeline_step_total_total counter
cf_pipeline_step_total_total{application="content-factory",step="StoryGenerationWorker.onMessage",success="true",} 52.0
cf_pipeline_step_total_total{application="content-factory",step="StoryGenerationWorker.onMessage",success="false",} 0.0

# TYPE cf_ai_inference_duration_seconds summary
cf_ai_inference_duration_seconds_count{model="llama3.2",provider="ollama",success="true",} 52.0

# TYPE cf_episodes_active gauge
cf_episodes_active{application="content-factory",} 1.0

# TYPE cf_workers_active gauge
cf_workers_active{application="content-factory",} 3.0
```

### Spring Boot Built-in Metrics (sample)

```
# HELP jvm_memory_used_bytes The amount of used memory
jvm_memory_used_bytes{area="heap",id="G1 Eden Space",} 4.1943040E7

# HELP process_cpu_usage The recent cpu usage for the Java Virtual Machine process
process_cpu_usage 0.004873294671366598

# HELP hikaricp_connections_active Active connections
hikaricp_connections_active{pool="ContentFactoryPool",} 1.0

# HELP rabbitmq_connections_total Total number of RabbitMQ connections created
rabbitmq_connections_total 1.0

# HELP http_server_requests_seconds Duration of HTTP server request handling
http_server_requests_seconds_count{exception="none",method="GET",outcome="SUCCESS",status="200",uri="/api/v1/episodes",} 8.0
```

---

## Prometheus Scrape Config

```yaml
# prometheus.yml
scrape_configs:
  - job_name: aisf
    scrape_interval: 15s
    static_configs:
      - targets: ['localhost:8088']
    metrics_path: /api/v1/metrics

  - job_name: content-factory
    scrape_interval: 15s
    static_configs:
      - targets: ['localhost:8080']
    metrics_path: /actuator/prometheus
```

## Grafana PromQL Examples

```
# AISF HTTP request rate (5-min window)
rate(aisf_http_requests_total[5m])

# AISF task duration p95 per agent
histogram_quantile(0.95,
  rate(aisf_task_duration_seconds_bucket[5m])
)

# AISF orchestration tick p99
histogram_quantile(0.99,
  rate(aisf_orch_tick_duration_seconds_bucket[5m])
)

# CF pipeline step p95 per step
histogram_quantile(0.95,
  rate(cf_pipeline_step_duration_seconds_bucket[5m])
)

# CF AI inference success rate
rate(cf_ai_inference_total_total{success="true"}[5m])
  /
rate(cf_ai_inference_total_total[5m])

# CF active episodes
cf_episodes_active

# JVM heap pressure
jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"}
```

---

## Metric Count Summary

| Service | Metric Families | Time Series |
|---|---|---|
| AISF (custom) | 20 | ~60–200 (varies with agent count) |
| CF (custom) | 14 | ~30–150 (varies with step cardinality) |
| CF (Spring Boot built-in) | ~80 | ~300+ |
| **Total** | **~114** | **~500–700** |
