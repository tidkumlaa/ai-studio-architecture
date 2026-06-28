# AI Studio Platform v1.5.3 — Runtime Operations Report
**Date:** 2026-06-27 | **Session Duration:** ~6 hours of live validation

---

## Service State Summary

All services verified operational during v1.5.3 live validation:

| Service | Port | Status | Notes |
|---|---|---|---|
| AISF (FastAPI/uvicorn) | 8088 | ✅ RUNNING | PID 44340, Python 3.13.14 |
| Content Factory (Spring Boot) | 8090 | ✅ RUNNING | Java 21, qwen2.5:3b |
| PostgreSQL | 5432 | ✅ RUNNING | Docker cf-postgres, contentfactory DB |
| RabbitMQ | 5672 / 15672 | ✅ RUNNING | Docker cf-rabbitmq, vhost=contentfactory |
| Redis | 6379 | ✅ RUNNING | Docker cf-redis |
| Ollama | 11434 | ✅ RUNNING | qwen2.5:3b (primary), qwen3:8b (available but excluded) |
| ComfyUI | 8188 | ❌ NOT RUNNING | Optional; Java2D fallback active for images |
| Piper TTS | — | ✅ AVAILABLE | en_US-sapi-medium.onnx via piper.bat |
| FFmpeg | — | ✅ AVAILABLE | WinGet installation |

## Worker Thread Status

AISF supervises 10 worker threads; all operational:

| Worker | Status | Protocol |
|---|---|---|
| ARCHITECT | Running | 1.0.0 |
| AUTH | Running | 1.0.0 |
| PLAYER | Running | 1.0.0 |
| ECONOMY | Running | 1.0.0 |
| COLLECTION | Running | 1.0.0 |
| BATTLE | Running | 1.0.0 |
| LIVEOPS | Running | 1.0.0 |
| SECURITY | Running | 1.0.0 |
| QA | Running | 1.0.0 |
| RELEASE | Running | 1.0.0 |

Verified via `GET /api/v1/workers`.

## Database State

- Tasks in AISF SQLite (`orchestrator.db`): **18 tasks**
- CF PostgreSQL: contentfactory schema, tables managed by Flyway migrations
- 3 backup archives available (sizes: 1.3MB, 1.3MB, 1.6MB)

## Content Pipeline Executions

One full live pipeline run executed during validation:

- **Episode:** `01KW36EH6PD0RFTPAV9TKSBH68` (Season 1, Universe `01KV9C8J2NXFXGC6FCF6BNW0JX`)
- **Topic:** "AI Revolution in Content Creation"
- **Language:** en
- **Steps completed:** TREND_RESEARCH → KEYWORD_ANALYSIS → STORY_GENERATION → CRITIC_REVIEW → MARKETING_COPY → TITLE_OPTIMIZATION
- **Final status:** Stalled at VIDEO_GENERATION (ComfyUI unavailable)
- **AI inference:** qwen2.5:3b via Ollama, all 6 steps successful

## Ollama Model Policy

| Model | Status | Reason |
|---|---|---|
| `qwen2.5:3b` | ✅ PRIMARY | 3B model, fast (~45s story gen) |
| `llama3.2:3b` | ✅ AVAILABLE | Fallback |
| `qwen3:8b` | ⚠️ EXCLUDED | Thinking model, times out in story generation (>2 min) |
| `qwen3:4b` | ⚠️ AVAILABLE (untested) | Thinking model, likely same issue |

**Production setting:** `runtime-config.yaml` → `ollama.default_model: "qwen2.5:3b"`

## CF Health Architecture Finding

CF Spring Boot Actuator endpoint behavior:
- `GET /actuator/health` → 503 when ANY health indicator is DOWN (including optional YouTube, ComfyUI)
- `GET /actuator/health/liveness` → 200 when Spring Boot app is alive (regardless of optional deps)
- `GET /actuator/health/readiness` → 200 when app is ready to serve

**Operational rule:** All health probes must use `/actuator/health/liveness`. Fixed in all 4 affected locations (routes.py ×2, runtime_routes.py, runtime-config.yaml).

## Incident Log

| Time | Incident | Resolution |
|---|---|---|
| Session start | CF health URL bug — AISF reporting CF=down despite CF running | Fixed 3 code locations; changed probe URL to /liveness |
| Session start | Old AISF process (PID 42116) holding port 8088 after restart | Killed old PID; new AISF (44340) bound correctly |
| Session start | UUID episode ID overflows VARCHAR(26) | Switched to ULID format IDs |
| Mid-session | qwen3:8b story generation timeout | Restarted CF with `--contentfactory.ollama.default-model=qwen2.5:3b` |
| Mid-session | CF crash simulation (failure recovery test) | Recovered via POST /runtime/start in 47s |

## Backup History

| Backup ID | Size | Created |
|---|---|---|
| backup-20260626-214133 | 1.3 MB | 2026-06-26 21:41 |
| backup-20260626-214358 | 1.3 MB | 2026-06-26 21:43 |
| backup-20260627-072519 | 1.6 MB | 2026-06-27 07:25 |
