# AI Studio Platform v1.5.3 — Deployment Guide
**Date:** 2026-06-27

---

## Prerequisites

| Component | Version | Notes |
|---|---|---|
| Java | 21 (Eclipse Adoptium recommended) | Required for Content Factory |
| Python | 3.11+ (3.13 validated) | Required for AISF |
| Docker Desktop | any | Runs PostgreSQL, RabbitMQ, Redis |
| Ollama | any | Models: `qwen2.5:3b` (required), others optional |
| FFmpeg | any | For media processing |

## Repository Layout

```
E:\UserData\MyData\Content\DEV\
├── ai-software-factory/          ← AISF (FastAPI) — port 8088
│   ├── app.py
│   ├── runtime/
│   │   ├── runtime-config.yaml  ← Single source of truth (Platform Rule #001)
│   │   ├── lib/                 ← start.py, stop.py, doctor.py, backup.py, ...
│   │   ├── logs/
│   │   └── pids/
│   ├── api/
│   └── tests/
└── ai-studio-desktop/            ← Desktop UI (PySide6)

E:\UserData\MyData\Content\AgentAIDev\content-factory\  ← CF (Spring Boot) — port 8090
├── cf-api/target/cf-api-1.0.0-SNAPSHOT.jar
├── docker-compose.local.yml      ← PostgreSQL, RabbitMQ, Redis
└── scripts/
```

## Fresh Deployment Steps

### 1. Start Infrastructure (Docker)

```powershell
Set-Location "E:\UserData\MyData\Content\AgentAIDev\content-factory"
docker-compose -f docker-compose.local.yml up -d
# Starts: cf-postgres (5432), cf-rabbitmq (5672/15672), cf-redis (6379) -- wait 10s
```

### 2. Verify Infrastructure

```powershell
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
# Expected: cf-postgres Up, cf-rabbitmq Up, cf-redis Up
```

### 3. Install Python Dependencies

```powershell
Set-Location "E:\UserData\MyData\Content\DEV\ai-software-factory"
pip install -e ".[dev]"
```

### 4. Start the Platform

```powershell
python runtime/lib/start.py --config runtime/runtime-config.yaml
```

Expected output: 12-step sequence ending in `PLATFORM READY`.

### 5. Verify

```powershell
Invoke-RestMethod "http://localhost:8088/api/v1/health"
# Expected: {"status":"healthy","database":"ok","content_factory":{"status":"healthy"},...}

Invoke-RestMethod "http://localhost:8088/api/v1/runtime/state"
# Expected: {"cf_running":true,"aisf_running":true,"ollama_running":true,"status":"running"}
```

## Configuration

All configuration lives in `runtime/runtime-config.yaml` (Platform Rule #001 — never commit stale config).

### Key settings to review before production:

```yaml
ollama:
  default_model: "qwen2.5:3b"   # DO NOT change to qwen3:8b — times out

content_factory:
  health_url: "http://localhost:8090/actuator/health/liveness"  # Must be /liveness, not /health
```

### Environment Variables (production)

```bash
ORCH_API_KEY=<strong-random-key>   # Required: enables API authentication
```

Without `ORCH_API_KEY`, all routes are unauthenticated. Only acceptable in local dev.

## Building CF JAR (if needed)

```powershell
Set-Location "E:\UserData\MyData\Content\AgentAIDev\content-factory"
$env:JAVA_HOME = "D:\Program Files\Eclipse Adoptium\jdk-21.0.6.7-hotspot"
mvn package -DskipTests -pl cf-api -am
```

Output: `cf-api/target/cf-api-1.0.0-SNAPSHOT.jar`

## Port Reference

| Port | Service | Notes |
|---|---|---|
| 8088 | AISF (FastAPI) | Primary API |
| 8090 | Content Factory | DO NOT use 8080 — svchost occupies it permanently on Windows |
| 5432 | PostgreSQL | Docker |
| 5672 | RabbitMQ AMQP | Docker |
| 15672 | RabbitMQ Management UI | Docker |
| 6379 | Redis | Docker |
| 11434 | Ollama | Local process |
| 8188 | ComfyUI | Optional; Java2D fallback when absent |

## Ollama Model Setup

```powershell
ollama pull qwen2.5:3b    # Required — primary model
ollama pull llama3.2:3b   # Optional — fallback
# Do NOT pull qwen3:8b for production use (thinking model, times out)
```

## Desktop Application

```powershell
Set-Location "E:\UserData\MyData\Content\DEV\ai-studio-desktop"
pip install -e ".[dev]"
python main.py
```

Desktop reads `desktop-config.yaml`. Ensure `aisf.host` and `aisf.port` are correct. If `api_key` is set, the desktop sends `X-API-Key` header automatically.

## Smoke Test After Deployment

```powershell
$h = @{"X-API-Key"=$env:ORCH_API_KEY}  # empty string if no key set

# 1. Platform health
Invoke-RestMethod "http://localhost:8088/api/v1/health" -Headers $h

# 2. Runtime state (all three should be true)
$s = Invoke-RestMethod "http://localhost:8088/api/v1/runtime/state" -Headers $h
if ($s.aisf_running -and $s.cf_running -and $s.ollama_running) {
    Write-Host "✅ Platform ready"
} else {
    Write-Host "❌ Platform incomplete: $($s | ConvertTo-Json)"
}

# 3. Run doctor
Invoke-RestMethod -Method POST "http://localhost:8088/api/v1/runtime/doctor" -Headers $h
```
