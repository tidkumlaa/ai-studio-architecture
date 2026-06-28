# AI Studio Platform v1.5.3 — Disaster Recovery Guide
**Date:** 2026-06-27

---

## Backup Strategy

Backups are created via `POST /api/v1/runtime/backup` and stored in `runtime/backups/`.

**What's backed up:**
- `runtime/state/` — episode state, workflow state
- `storage/objects/` — content artifacts
- `orchestrator.db` — AISF SQLite database (tasks, heartbeats, gates, SLAs)
- `SPRINT1-PRODUCTION-REPORT.md`
- `factory.yaml`

**What's NOT backed up (external):**
- PostgreSQL data (CF domain data) — back up separately with `pg_dump`
- RabbitMQ messages in-flight
- Ollama models (re-downloadable)

### Create Backup
```powershell
Invoke-RestMethod -Method POST "http://localhost:8088/api/v1/runtime/backup"
# Returns: {"ok": true, "output": "Backup created: backup-20260627-HHMMSS"}
```

### List Backups
```powershell
Invoke-RestMethod "http://localhost:8088/api/v1/runtime/backups"
```

### Restore from Backup
```powershell
Invoke-RestMethod -Method POST "http://localhost:8088/api/v1/runtime/restore/backup-20260627-072519"
```

---

## Recovery Scenarios

### Scenario 1: CF Crash (most common)

1. AISF detects CF down via next `/runtime/state` poll (<1s)
2. Trigger recovery:
   ```powershell
   Invoke-RestMethod -Method POST "http://localhost:8088/api/v1/runtime/start"
   ```
3. Poll state every 15s:
   ```powershell
   while ($true) {
       $s = Invoke-RestMethod "http://localhost:8088/api/v1/runtime/state"
       Write-Host "cf_running=$($s.cf_running)"
       if ($s.cf_running) { break }
       Start-Sleep 15
   }
   ```
4. Expected recovery time: **47 seconds**

### Scenario 2: AISF Process Dead

1. Find and start AISF:
   ```powershell
   # Check if port 8088 is free
   Get-NetTCPConnection -LocalPort 8088 -State Listen -ErrorAction SilentlyContinue
   
   # Start AISF
   Set-Location "E:\UserData\MyData\Content\DEV\ai-software-factory"
   Start-Process python -ArgumentList "-m uvicorn app:app --host 0.0.0.0 --port 8088" -PassThru
   ```
2. Wait 5s; verify: `Invoke-RestMethod "http://localhost:8088/api/v1/health"`

### Scenario 3: PostgreSQL Down

CF will fail to start if PostgreSQL is unavailable (HikariCP connection pool fails at boot).

1. Start Docker Desktop
2. Verify Docker containers: `docker ps` — check `cf-postgres` is running
3. If container stopped: `docker start cf-postgres`
4. Restart CF: `Invoke-RestMethod -Method POST "http://localhost:8088/api/v1/runtime/start"`

### Scenario 4: Full Platform Recovery

```powershell
Set-Location "E:\UserData\MyData\Content\DEV\ai-software-factory"
# Run repair — combines doctor + start.py
Invoke-RestMethod -Method POST "http://localhost:8088/api/v1/runtime/repair"
# Expected time: ~15-20 seconds (checks all deps, starts what's missing)
```

### Scenario 5: Port Conflict (old process on 8088)

If AISF restart leaves old process holding port 8088:
```powershell
$conn = Get-NetTCPConnection -LocalPort 8088 -State Listen
$oldPid = $conn.OwningProcess
Stop-Process -Id $oldPid -Force
Start-Sleep 2
# Now start fresh AISF
```

### Scenario 6: Restore from Backup

```powershell
# List available backups
(Invoke-RestMethod "http://localhost:8088/api/v1/runtime/backups").backups | Select id, timestamp, size_bytes

# Restore (AISF must be running; CF can be stopped)
Invoke-RestMethod -Method POST "http://localhost:8088/api/v1/runtime/restore/backup-20260627-072519"

# Resume after restore to reload episode state
Invoke-RestMethod -Method POST "http://localhost:8088/api/v1/runtime/resume"
```

---

## PostgreSQL Backup (CF Domain Data)

The CF PostgreSQL database (`contentfactory` DB) is not covered by the platform backup tool.

```powershell
# Dump CF database
docker exec cf-postgres pg_dump -U contentfactory contentfactory > cf-backup-$(Get-Date -Format 'yyyyMMdd').sql

# Restore
Get-Content cf-backup-20260627.sql | docker exec -i cf-postgres psql -U contentfactory contentfactory
```

---

## RTO / RPO Targets

| Component | RTO | RPO |
|---|---|---|
| CF crash recovery | 60s (actual: 47s) | 0 (in-flight work re-queued via RabbitMQ) |
| Full platform recovery | 120s | Last backup |
| AISF crash | 30s | 0 (stateless HTTP) |
| PostgreSQL loss | 120s + DB restore | Last pg_dump |
