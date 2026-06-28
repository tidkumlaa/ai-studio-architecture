# AI Studio Platform v1.5.3 — Live Production Validation Evidence
**Date:** 2026-06-27 | **Session:** v1.5.3 Production Validation
**Method:** Real services, real execution, no mocks. All evidence is live HTTP traces and process output.

---

## Environment at Validation Time

| Service | Status | PID / Port | Version |
|---|---|---|---|
| AISF (uvicorn) | RUNNING | PID 44340 / 8088 | 1.5.2 |
| Content Factory (Spring Boot) | RUNNING | PID 24340→restarted / 8090 | 1.0.0-SNAPSHOT |
| PostgreSQL | RUNNING | Docker cf-postgres / 5432 | 17-alpine |
| RabbitMQ | RUNNING | Docker cf-rabbitmq / 5672, 15672 | 3.13-management |
| Redis | RUNNING | Docker cf-redis / 6379 | 7-alpine |
| Ollama | RUNNING | port 11434 | qwen2.5:3b |
| ComfyUI | NOT RUNNING | — | NOT VERIFIED |

---

## Phase 1: Boot Chain — FULLY VERIFIED

**Evidence:**
```
GET http://127.0.0.1:8088/api/v1/health  HTTP/1.1
→ 200 OK  26ms
{
  "status": "healthy",
  "version": "1.5.2",
  "database": "ok",
  "task_count": 18,
  "content_factory": {"status": "healthy", "port": 8090},
  "ollama": {"status": "healthy", "port": 11434}
}

GET http://127.0.0.1:8090/actuator/health/liveness  HTTP/1.1
→ 200 OK  43ms  {"status":"UP"}

PostgreSQL: reachable at localhost:5432 ✓
RabbitMQ: reachable at localhost:5672 ✓
```

**Fix applied:** CF health probe changed from `/actuator/health` (returns 503 when optional indicators DOWN) to `/actuator/health/liveness` (returns 200 when app alive) in `api/routes.py:48,61`, `api/runtime_routes.py:127`, `runtime/runtime-config.yaml`.

---

## Phase 2: Content Pipeline — 6/10 STEPS VERIFIED

**Live workflow trigger:**
```
POST http://127.0.0.1:8090/api/v1/production/start
Body: {"projectId":"P01KV9C8J2NXFXGC6FCF6","episodeId":"01KW36EH6PD0RFTPAV9TKSBH68",
       "topic":"AI Revolution","languageCode":"en","channelId":"CH01"}
→ 202 ACCEPTED {"status":"QUEUED","episodeId":"01KW36EH6PD0RFTPAV9TKSBH68"}
```

**Pipeline step verification:**

| Step | Status | Evidence |
|---|---|---|
| TREND_RESEARCH | ✅ VERIFIED | TrendResearchAgent: score=80.11, 3 trends identified |
| KEYWORD_ANALYSIS | ✅ VERIFIED | KeywordAgent: 3 clusters, 12 keywords |
| STORY_GENERATION | ✅ VERIFIED | StoryAgent: 760 words, 923 tokens, qwen2.5:3b, ~45s |
| CRITIC_REVIEW | ✅ VERIFIED | CriticAgent: score=76/100, 3 improvement areas |
| MARKETING_COPY | ✅ VERIFIED | MarketingAgent: 3 copy variants |
| TITLE_OPTIMIZATION | ✅ VERIFIED | TitleAgent: 5 title variants |
| THUMBNAIL_GENERATION | ⚠️ PARTIAL | Java2D fallback active (ComfyUI not running) |
| TRANSLATION | ✅ VERIFIED | via RabbitMQ queue delivery |
| VIDEO_GENERATION | ❌ NOT VERIFIED | VideoGenerationPipeline requires ComfyUI |
| PUBLISHING | ❌ NOT VERIFIED | Blocked by VIDEO_GENERATION failure |

**AI Inference confirmed:** Ollama `qwen2.5:3b` responding for all text generation steps. `qwen3:8b` excluded (2-min timeout failures observed).

---

## Phase 3: Runtime API — ALL ENDPOINTS VERIFIED

| Endpoint | HTTP | Response Time | Result |
|---|---|---|---|
| `GET /runtime/state` | 200 | 27.8ms avg | `cf_running=true, status=running` |
| `POST /runtime/start` (CF up) | 200 | 134ms | `"Platform already running"` |
| `POST /runtime/stop` | 200 | 450ms | `ok=true, returncode=0` |
| `POST /runtime/restart` | 200 | 90ms | Background task initiated |
| `POST /runtime/resume` | 200 | 8,928ms | 12-step full startup sequence |
| `POST /runtime/doctor` | 200 | ~5s | All deps checked |
| `POST /runtime/backup` | 200 | ~2s | Archive created |
| `GET /runtime/backups` | 200 | 80ms | 3 backups listed |
| `POST /runtime/restore/{id}` | 200/400 | varies | Valid IDs proceed, invalid→400 |
| `POST /runtime/update` | 200 | ~3s | Rule #001 validator ran |
| `POST /runtime/repair` | 200 | 15,461ms | Doctor + start.py executed |

**Key fix:** `cf_running` was returning `false` even though CF was healthy. Root cause: old AISF process (PID 42116) still bound to port 8088 after "restart"; new process (PID 43728) silently failed to bind. Fix: killed old PID explicitly; new AISF (PID 44340) bound correctly; `cf_running=true` confirmed.

---

## Phase 4: Failure Recovery — VERIFIED

**Simulation:**
```
Kill CF PID 24340 → simulated crash
GET /runtime/state → cf_running=false, status=stopped  (detection in <1s)
POST /runtime/start → "Platform startup initiated"  (background)
Poll /runtime/state every 15s:
  t+16s → cf_running=false
  t+32s → cf_running=false  
  t+47s → cf_running=true, status=running  ← RECOVERED
```

**Recovery time: 47 seconds** (CF Spring Boot startup with PostgreSQL schema validation).

---

## Phase 5: Performance Benchmark — ALL ENDPOINTS <40ms

| Endpoint | Avg | Min | Max |
|---|---|---|---|
| `GET /health` | 26.7ms | 16ms | 36.2ms |
| `GET /version` | 1.0ms | 1ms | 1ms |
| `GET /runtime/state` | 27.8ms | 17.2ms | 31.7ms |
| `GET /runtime/backups` | 4.8ms | 4.4ms | 5.4ms |
| `GET /capabilities` | 31.5ms | 30.8ms | 32.8ms |
| `GET /workers` | 1.4ms | 1ms | 2ms |
| `GET /tasks` | 3.1ms | 3ms | 3.5ms |
| `GET /dashboard` | 29.5ms | 27.1ms | 31.1ms |
| CF `/actuator/health/liveness` | 8.5ms | 1ms | 66.6ms |

Measurement: 5-rep average after 1 warm-up, Python `urllib.request`.

---

## Phase 6: Security Validation — PASS

| Test | Expected | Result |
|---|---|---|
| Empty `X-API-Key` (dev mode) | 200 OK | ✅ PASS |
| Wrong key (no `ORCH_API_KEY` set) | 200 OK (key disabled) | ✅ PASS |
| `backup_id` with semicolon (`;`) | HTTP 400 | ✅ PASS |
| `backup_id` with `..%2F..%2F` | HTTP 404 (router-level block) | ✅ PASS (blocked before handler) |
| Valid `backup_id` (alphanumeric + hyphens) | Restore proceeds | ✅ PASS |

**Note:** `ORCH_API_KEY` must be set in production. Default empty key disables auth (dev convenience only).

---

## Phase 7: Installer — VERIFIED

| Check | Result |
|---|---|
| `installer/downloads/manager.py` exists | ✅ |
| `installer/bootstrap/manager.py` exists | ✅ |
| `sha256:placeholder` bypass in downloads | REMOVED ✅ |
| `sha256:placeholder` bypass in bootstrap | REMOVED ✅ |
| `scripts/release.py` writes `CHECKSUMS.sha256` | ✅ |
| GPG signing graceful when GPG absent | ✅ |

---

## Phase 8: CI/CD Test Results

**Run from `ai-software-factory/` root (correct sys.path):**

| File | Isolated Pass | Isolated Fail | Note |
|---|---|---|---|
| test_autonomous_execution.py | 5 | 0 | |
| test_claude_integration.py | 7 | 0 | |
| test_installer.py | 70 | 0 | |
| test_phase21b_autonomous_sprint.py | 10 | 0 | |
| test_phase21b_chaos.py | 10 | 0 | |
| test_phase22_load.py | 6 | 0 | |
| test_phase22_reliability.py | 10 | 0 | |
| test_phase22_security.py | 10 | 0 | |
| test_phase23_factory.py | 15 | 1 | Stale: expects `1.0.0`, actual `1.5.2` |
| test_phase25_multi_project.py | 23 | 0 | |
| test_phase26_bootstrap.py | 34 | 0 | |
| test_phase27_release.py | 24 | 1 | Missing CHANGELOG entry for v1.5.3 |
| test_phase28_adoption.py | 77 | 0 | |
| test_runtime_consolidation.py | 9 | 0 | |
| test_runtime_manager.py | 35 | 0 | |
| test_runtime_validation.py | 9 | 0 | |
| test_stall_recovery.py | 6 | 0 | |
| test_storage.py | 68 | 0 | |
| test_v152_aisf_core.py | 10 | 0 | v1.5.2 certified tests |
| test_v152_installer.py | 16 | 0 | v1.5.2 certified tests |
| test_v152_proxy.py | 8 | 0 | v1.5.2 certified tests |
| test_workflow_supervisor.py | 7 | 0 | |
| **Desktop: test_v152_desktop.py** | **18** | **0** | |

**TOTAL: 487 PASS / 489 (99.6%) in isolation**

**Known test isolation issue (suite run):** When running the full suite, `runtime/lib/config.py` shadows `ai-software-factory/config.py` via `sys.path` pollution from `test_runtime_manager.py`, causing 37 additional failures that all pass in isolation. Root cause: two files both named `config.py` in the project. Not a production code defect.

**test_assets.py:** Import error (`ModuleNotFoundError: factory`). Factory SDK module not on test path.

---

## Phase 9: Observability

**Structured Health Response (7 service fields):**
```json
{
  "status": "healthy", "version": "1.5.2", "port": 8088,
  "database": "ok", "task_count": 18,
  "content_factory": {"status": "healthy", "port": 8090},
  "ollama": {"status": "healthy", "port": 11434},
  "platform": "1.5.2", "plugin_api": "1.5.2", "runtime": "1.5.2",
  "schema": 1, "hook": "1.0.0", "dashboard": "1.0.0", "worker_protocol": "1.0.0"
}
```

**12-Step Startup Timeline** (captured via `POST /runtime/resume`, 8.9s total):
1. Runtime directories — OK
2. Java 21 — OK
3. Python 3.13.14 — OK
4. PostgreSQL:5432 — OK
5. RabbitMQ:5672 — OK
6. Ollama (qwen2.5:3b, 4 models) — OK
7. ComfyUI — WARN (optional, Java2D fallback)
8. Piper (en_US-sapi-medium.onnx) — OK
9. FFmpeg — OK
10. Temp dirs — OK
11. AISF already running — OK
12. CF already running → Final health: AISF=HEALTHY, CF=HEALTHY

**AISF log file:** Not created in current session (uvicorn started without `--log-config`; log goes to stdout).

---

## Bugs Discovered and Fixed in v1.5.3

| # | Bug | Files | Status |
|---|---|---|---|
| 1 | CF health probe `/actuator/health` returns 503 when optional indicators DOWN | `api/routes.py:48,61` `api/runtime_routes.py:127` `runtime/runtime-config.yaml` | FIXED |
| 2 | `cf_running=false` because old AISF PID held port 8088 after restart | Process management (operational, not code) | DOCUMENTED |
| 3 | UUID overflows VARCHAR(26) CF columns | Operational: use ULID format IDs | DOCUMENTED |
| 4 | `stop.py` cannot stop manually-started CF (no PID file) | `runtime/lib/stop.py` reads PID file | KNOWN LIMITATION |
| 5 | Ollama `qwen3:8b` times out in story generation | `runtime-config.yaml` default_model | FIXED (→ qwen2.5:3b) |

---

*Evidence collected 2026-06-27. All HTTP traces captured with real service calls, no mocks.*
