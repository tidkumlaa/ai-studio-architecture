# AI Studio Platform v1.5.3 — Operational Runbook
**Date:** 2026-06-27

---

## Prerequisites

| Dependency | Required Version | Location |
|---|---|---|
| Java | 21 | `D:/Program Files/Eclipse Adoptium/jdk-21.0.6.7-hotspot` |
| Python | 3.13+ | system |
| Docker Desktop | latest | must be running for PostgreSQL/RabbitMQ/Redis |
| Ollama | any | port 11434, model `qwen2.5:3b` loaded |

## Starting the Platform

### Full start (all services via start.py)
```powershell
Set-Location "E:\UserData\MyData\Content\DEV\ai-software-factory"
python runtime/lib/start.py --config runtime/runtime-config.yaml
```

Start sequence: runtime dirs → Java → Python → PostgreSQL → RabbitMQ → Ollama → ComfyUI (optional) → Piper → FFmpeg → AISF → CF → health check.

### Manual AISF start (if start.py fails)
```powershell
Set-Location "E:\UserData\MyData\Content\DEV\ai-software-factory"
Start-Process python -ArgumentList "-m uvicorn app:app --host 0.0.0.0 --port 8088" -PassThru
```

### Manual CF start (with Ollama model override)
```powershell
$jar = "E:\UserData\MyData\Content\AgentAIDev\content-factory\cf-api\target\cf-api-1.0.0-SNAPSHOT.jar"
$javaHome = "D:\Program Files\Eclipse Adoptium\jdk-21.0.6.7-hotspot"
& "$javaHome\bin\java.exe" -Xmx2g -jar $jar `
    --spring.profiles.active=production-local `
    --contentfactory.ollama.default-model=qwen2.5:3b
```

**IMPORTANT:** Do NOT use `qwen3:8b` — it times out during story generation (2+ minutes). Always use `qwen2.5:3b`.

## Health Checks

```powershell
# Overall platform health
Invoke-RestMethod "http://localhost:8088/api/v1/health"

# Runtime state (cf_running, ollama_running, aisf_running)
Invoke-RestMethod "http://localhost:8088/api/v1/runtime/state"

# CF liveness directly (avoids 503 from non-critical indicators)
Invoke-RestMethod "http://localhost:8090/actuator/health/liveness"
```

**Key:** Always probe CF at `/actuator/health/liveness`, NOT `/actuator/health`. The aggregate endpoint returns 503 when optional indicators (YouTube, ComfyUI) are DOWN.

## Stopping the Platform

```powershell
# Via API
Invoke-RestMethod -Method POST "http://localhost:8088/api/v1/runtime/stop"

# Via script
python runtime/lib/stop.py --config runtime/runtime-config.yaml

# Manual (if no PID file exists — e.g. manually started CF)
Get-NetTCPConnection -LocalPort 8090 | Select OwningProcess
Stop-Process -Id <PID> -Force
```

**Note:** `stop.py` uses PID files written by `start.py`. If CF was started manually, kill by PID directly.

## Triggering Content Production

Content production requires:
1. A domain episode in `core.episode` (created via `POST /api/v1/episodes`)
2. A ULID episode ID (26-char Crockford Base32, NOT UUID)

```powershell
# Step 1: Create domain episode (get seasonId from existing season)
$episode = Invoke-RestMethod -Method POST "http://localhost:8090/api/v1/episodes" `
    -ContentType "application/json" `
    -Body '{"seasonId":"01KV9C8J2NXFXGC6FCF6BNW0JX","episodeNo":1,"title":"AI Revolution",
             "summary":"Exploring AI trends","targetDurationMin":10}'

# Step 2: Trigger production
Invoke-RestMethod -Method POST "http://localhost:8090/api/v1/production/start" `
    -ContentType "application/json" `
    -Body "{\"projectId\":\"P01KV9C8J2NXFXGC6FCF6\",\"episodeId\":\"$($episode.id)\",
           \"topic\":\"AI Revolution\",\"languageCode\":\"en\",\"channelId\":\"CH01\"}"
```

## Diagnosing Issues

```powershell
# Full dependency check
Invoke-RestMethod -Method POST "http://localhost:8088/api/v1/runtime/doctor"

# Auto-repair (doctor + start)
Invoke-RestMethod -Method POST "http://localhost:8088/api/v1/runtime/repair"

# Create backup before troubleshooting
Invoke-RestMethod -Method POST "http://localhost:8088/api/v1/runtime/backup"
```

## Worker Supervision

AISF runs 10 agent worker threads: ARCHITECT, AUTH, PLAYER, ECONOMY, COLLECTION, BATTLE, LIVEOPS, SECURITY, QA, RELEASE.

```powershell
# View worker status
Invoke-RestMethod "http://localhost:8088/api/v1/workers"

# Restart specific worker
Invoke-RestMethod -Method POST "http://localhost:8088/api/v1/workers/restart/ARCHITECT"
```

## CF RabbitMQ Management

RabbitMQ management UI: http://localhost:15672 (guest/guest, vhost=contentfactory)

Check queues if pipeline steps appear stuck. Messages route via `contentfactory.events` exchange with routing key `content.pipeline`.

## Common Issues

| Symptom | Cause | Fix |
|---|---|---|
| `cf_running=false` despite CF running | AISF probing `/actuator/health` (503) | Ensure AISF code has liveness fix; restart AISF |
| `cf_running=false` after AISF restart | Old AISF process still holds port 8088 | Find old PID: `Get-NetTCPConnection -LocalPort 8088`; kill it |
| CF startup fails | PostgreSQL not running | Start Docker Desktop; ensure cf-postgres container is running |
| Story generation timeout | `qwen3:8b` model | Use `qwen2.5:3b`; override via `--contentfactory.ollama.default-model=qwen2.5:3b` |
| Episode 400 on production start | Wrong ID format | Use ULID (26-char), not UUID (36-char) |
| Pipeline stalls at VIDEO_GENERATION | ComfyUI not running | Start ComfyUI or accept Java2D fallback for images only |
