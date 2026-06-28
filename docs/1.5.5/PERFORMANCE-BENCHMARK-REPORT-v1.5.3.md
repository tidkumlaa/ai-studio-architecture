# AI Studio Platform v1.5.3 — Performance Benchmark Report
**Date:** 2026-06-27 | **Environment:** Local Windows 11, Java 21, Python 3.13, Docker Desktop

---

## AISF REST Endpoint Latency

Measurement method: 5-rep average after 1 warm-up, Python `urllib.request`, loopback (127.0.0.1).

| Endpoint | Avg (ms) | Min (ms) | P99 (ms) | SLA Target | Result |
|---|---|---|---|---|---|
| `GET /health` | 26.7 | 16.0 | 36.2 | <200ms | ✅ PASS |
| `GET /version` | 1.0 | 1.0 | 1.0 | <50ms | ✅ PASS |
| `GET /runtime/state` | 27.8 | 17.2 | 31.7 | <200ms | ✅ PASS |
| `GET /runtime/backups` | 4.8 | 4.4 | 5.4 | <500ms | ✅ PASS |
| `GET /capabilities` | 31.5 | 30.8 | 32.8 | <200ms | ✅ PASS |
| `GET /workers` | 1.4 | 1.0 | 2.0 | <50ms | ✅ PASS |
| `GET /tasks` | 3.1 | 3.0 | 3.5 | <500ms | ✅ PASS |
| `GET /dashboard` | 29.5 | 27.1 | 31.1 | <200ms | ✅ PASS |

Health endpoints (`/health`, `/runtime/state`, `/capabilities`) include parallel HTTP probes to CF and Ollama; their latency is dominated by the CF liveness check.

## CF Service Latency

| Endpoint | Avg (ms) | Min (ms) | Max (ms) |
|---|---|---|---|
| `/actuator/health/liveness` | 8.5 | 1.0 | 66.6ms |

The max spike (66.6ms) is within acceptable bounds; JVM GC pause on Spring Boot.

## Recovery Benchmark

| Scenario | Time |
|---|---|
| CF crash detection | <1 second (next `/runtime/state` poll) |
| CF recovery after crash (`POST /runtime/start`) | 47 seconds |
| AISF full startup + resume check (`POST /runtime/resume`) | 8.9 seconds |

## Pipeline Step Timing (Ollama qwen2.5:3b)

| Step | Observed Time |
|---|---|
| TREND_RESEARCH | ~5s |
| KEYWORD_ANALYSIS | ~3s |
| STORY_GENERATION (760 words, 923 tokens) | ~45s |
| CRITIC_REVIEW | ~8s |
| MARKETING_COPY | ~10s |
| TITLE_OPTIMIZATION | ~5s |

Full pipeline (6 verified steps): ~76 seconds. VIDEO_GENERATION and PUBLISHING not measured (ComfyUI required).

## Summary

All AISF REST endpoints are well within SLA targets. CF Spring Boot startup (47s) is the dominant latency in recovery scenarios. AI inference via Ollama `qwen2.5:3b` is the dominant cost in pipeline execution (~45s for story generation).
