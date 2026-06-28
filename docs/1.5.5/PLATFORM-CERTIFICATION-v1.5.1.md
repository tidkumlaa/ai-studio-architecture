# AI Studio Platform — Production Integration & End-to-End Certification
## Audit Report v1.5.1 | Date: 2026-06-27

---

## Audit Scope

| Repository | Path |
|---|---|
| ai-studio-desktop | `E:\UserData\MyData\Content\DEV\ai-studio-desktop` |
| ai-software-factory | `E:\UserData\MyData\Content\DEV\ai-software-factory` |
| content-factory | `E:\UserData\MyData\Content\AgentAIDev\content-factory` |

All three repositories treated as one platform. Every finding is backed by code evidence with file:line references.

---

## Executive Summary

The platform is **NOT READY FOR PRODUCTION** in its current state. The bootstrap chain, AISF core API, and failure recovery are solidly implemented. However, seven blocking issues prevent a production certification:

1. `POST /runtime/update` endpoint missing from AISF — the Desktop Update button will always fail
2. All Desktop content management panels (Queue, Episodes, Publishing, Quality, Inventory) will receive HTTP 404 from AISF — the proxy routes do not exist
3. `desktop-config.yaml` contains a hardcoded developer machine path that will break installation on any other system
4. No authentication layer exists anywhere in the platform
5. All inter-service communication uses plain HTTP (no TLS)
6. Release pipeline artifact signing is a non-functional placeholder
7. SHA-256 checksum verification can be bypassed with a placeholder prefix

**Stated version (v1.5.1) does not match code version (1.0.0) in all three pyproject.toml files and platform_version.py.**

---

## Phase 1 — Platform Bootstrap Verification

### 1.1 Bootstrap Chain

```
start-desktop.bat
  └─ bootstrap/cli.py (start)
       └─ bootstrap/launcher.py
            ├─ venv_manager.py          — venv create / pip install
            ├─ dependency_checker.py    — Python / PySide6 / httpx / PyYAML
            ├─ aisf_checker.py          — health probe / auto-start AISF
            │    └─ [AISF_HOME]/start.bat → app.py → FastAPI lifespan
            │         ├─ create_tables() + seed_sprint0/sprint1()
            │         ├─ init_storage_service()
            │         ├─ init_asset_service() + InventoryNodeRegistry()
            │         ├─ WorkerSupervisor.start() (10 agent threads)
            │         ├─ Orchestration loop thread
            │         └─ 6 API routers registered on :8088
            └─ subprocess.Popen(main.py) — Qt process
                 └─ MainWindow.__init__
                      ├─ QTimer(0) → CapabilityController.fetch()
                      ├─ DashboardController.start_live_refresh(5000 ms)
                      └─ RuntimeController.start_live_refresh(3000 ms)
```

| Component | Status | Evidence |
|---|---|---|
| Desktop launches AISF automatically | **IMPLEMENTED** | `bootstrap/launcher.py:149–194` — probes health, calls `aisf.start(script_path)` + `wait_until_healthy()` if `auto_start_runtime: true` |
| AISF launches correctly | **IMPLEMENTED** | `app.py` lifespan registers all services and 6 routers |
| Content Factory starts correctly | **IMPLEMENTED** (via AISF control) | `POST /runtime/start` → `lib/start.py` starts CF Java process |
| Health endpoints become green | **IMPLEMENTED** | `/health`, `/health/ping`, `/health/metrics` all implemented |
| Desktop updates status automatically | **IMPLEMENTED** | Polling via QTimer: Dashboard 5000 ms, Runtime 3000 ms, with exponential backoff on failure |

**Phase 1 Result: PASS**

---

## Phase 2 — Runtime API Validation

All AISF runtime routes are in `api/runtime_routes.py`. Routes prefixed `/api/v1`.

| Endpoint | Method | HTTP Status | JSON Schema | Error Handling | Logging | Status |
|---|---|---|---|---|---|---|
| `/api/v1/health` | GET | 200/503 | `status, version, port, database, task_count, content_factory{}, ollama{}, version_info{}` | 503 on DB error | Yes — uvicorn | **IMPLEMENTED** |
| `/api/v1/health/ping` | GET | 200 | `ok, timestamp` | N/A | Minimal | **IMPLEMENTED** |
| `/api/v1/health/metrics` | GET | 200 | `cpu_percent, ram_percent, disk_used_gb, disk_total_gb, gpu_percent, gpu_vram_percent` | Defaults to 0.0 on psutil/nvidia-smi absence | None | **IMPLEMENTED** |
| `/api/v1/runtime/state` | GET | 200 | `status, aisf_running, cf_running, ollama_running, platform_version, last_started, last_stopped, ports{}` | Graceful on YAML missing | Yes | **IMPLEMENTED** |
| `/api/v1/capabilities` | GET | 200 | `contract_version, runtime{}, health{}, content{}, ai{}, storage, inventory, assets, plugins` | Returns False for absent libs | None | **IMPLEMENTED** |
| `/runtime/start` | POST | 200 | `ok, message, state{}` | Guards on `cf_running` | Yes | **IMPLEMENTED** |
| `/runtime/stop` | POST | 200 | `ok, returncode, output, error` (from `_run_lib`) | Timeout 60s | Yes | **IMPLEMENTED** |
| `/runtime/restart` | POST | 200 | `ok, message` | Background task | Yes | **IMPLEMENTED** |
| `/runtime/resume` | POST | 200 | `ok, returncode, output, error` | Timeout 60s | Yes | **IMPLEMENTED** |
| `/runtime/doctor` | POST | 200 | `ok, returncode, output, error` | Timeout 60s | Yes | **IMPLEMENTED** |
| `/runtime/repair` | POST | 200 | `ok, diagnostics{}, repair{}, state{}` | Timeout 60s | Yes | **IMPLEMENTED** |
| `/runtime/backup` | POST | 200 | `ok, returncode, output, error` | Timeout 120s | Yes | **IMPLEMENTED** |
| `/runtime/restore/{backup_id}` | POST | 200 | `ok, returncode, output, error` | Timeout 120s | Yes | **IMPLEMENTED** |
| `/runtime/update` | POST | **MISSING** | N/A | N/A | N/A | **NOT IMPLEMENTED** |

### Issue P2-01 — BLOCKER

| Field | Value |
|---|---|
| **Severity** | BLOCKER |
| **Finding** | `POST /runtime/update` is absent from AISF. The Desktop's `RuntimeClient.update()` (`services/runtime_client.py:38`) calls this path. The Update button (`ui/runtime_center.py:68,140`) will always produce a REST error at runtime. |
| **Root cause** | `runtime/lib/update.py` exists but is a Platform Rule validator (checks repo rule compliance), not a software-update script. It was never registered as an API route. |
| **Affected files** | `api/runtime_routes.py` (absent), `services/runtime_client.py:38`, `ui/runtime_center.py:68,140` |
| **Recommended fix** | Either (a) add a `POST /runtime/update` route in `runtime_routes.py` that calls `_run_lib("update.py")` or a proper updater script, or (b) remove the Update button and `RuntimeClient.update()` until the endpoint is built |
| **Estimated effort** | 2 hours (wire existing route pattern) + separate effort for update.py implementation |

**Phase 2 Result: FAIL — 1 blocker**

---

## Phase 3 — Capability Discovery

### 3.1 AISF Capability Response (from `api/capabilities_routes.py:65–96`)

```json
{
  "contract_version": "1.0",
  "runtime": {
    "start": true,   "stop": true,    "restart": true,
    "doctor": true,  "repair": true,  "backup": true,  "restore": true
  },
  "health": { "ping": true, "metrics": true },
  "content": {
    "available": <port-probe>,
    "episodes": false,  "queue": false,   "publishing": false,
    "quality": false,   "timeline": false
  },
  "ai":      { "copilot": false, "commands": true },
  "storage": true,
  "inventory": true,
  "assets": true,
  "plugins": false
}
```

### 3.2 Desktop Capability Handling (from `ui/runtime_center.py:108–132`)

| Capability | AISF emits key? | Desktop gates on it? | Result |
|---|---|---|---|
| Runtime: start/stop/restart/doctor/repair/backup/restore | Yes | Yes | CORRECT — all 7 gated individually |
| Runtime: resume | No (`resume` key absent from AISF response) | Gates on `restart` key — BUG | INCORRECT |
| Runtime: update | No | Not gated at all | INCORRECT — always enabled despite missing endpoint |
| Content: episodes/queue/publishing/quality/timeline | Yes (all False) | Not gated in `apply_capabilities()` | NOT VERIFIED — panels exist but capability gates not wired |
| AI: copilot | Yes (False) | Not gated | NOT VERIFIED |
| Plugins | Yes (False) | Not gated | NOT VERIFIED |

### Issue P3-01 — HIGH

| Field | Value |
|---|---|
| **Severity** | HIGH |
| **Finding** | Resume button is gated on the `restart` capability key instead of a `resume` key. |
| **Root cause** | `ui/runtime_center.py:122`: `self._resume_btn.setEnabled(rt.get("restart", True))` — copy-paste error |
| **Affected files** | `ui/runtime_center.py:122`, `api/capabilities_routes.py` (missing `resume` key) |
| **Recommended fix** | Add `"resume": _has_lib("resume.py")` to AISF capabilities response; change `runtime_center.py:122` to `rt.get("resume", True)` |
| **Estimated effort** | 30 minutes |

### Issue P3-02 — HIGH

| Field | Value |
|---|---|
| **Severity** | HIGH |
| **Finding** | Update button is never passed through `apply_capabilities()` and is always enabled, even though the endpoint is missing. |
| **Root cause** | No entry for `_update_btn` in `apply_capabilities()` in `ui/runtime_center.py` |
| **Affected files** | `ui/runtime_center.py` |
| **Recommended fix** | Add Update button to `apply_capabilities()` gated on an `"update"` capability key; add `"update": _has_lib("update_software.py")` in AISF, or disable the button unconditionally until the endpoint exists |
| **Estimated effort** | 30 minutes |

**Phase 3 Result: FAIL — 2 issues**

---

## Phase 4 — Desktop Functional Test

### 4.1 Runtime Center Buttons (`ui/runtime_center.py`)

| Button | Handler | AISF Endpoint | Status |
|---|---|---|---|
| Start | `on_start()` → `RuntimeController.start()` | `POST /runtime/start` | **IMPLEMENTED** |
| Stop | `on_stop()` → `RuntimeController.stop()` | `POST /runtime/stop` | **IMPLEMENTED** |
| Restart | `on_restart()` → `RuntimeController.restart()` | `POST /runtime/restart` | **IMPLEMENTED** |
| Resume | `on_resume()` → `RuntimeController.resume()` | `POST /runtime/resume` | **IMPLEMENTED** (backend); capability bug (see P3-01) |
| Doctor | `on_doctor()` → `RuntimeController.doctor()` | `POST /runtime/doctor` | **IMPLEMENTED** |
| Repair | `on_repair()` → `RuntimeController.repair()` | `POST /runtime/repair` | **IMPLEMENTED** |
| Update | `on_update()` → `RuntimeController.update()` | `POST /runtime/update` | **BROKEN** — endpoint missing (see P2-01) |
| Backup | `on_backup()` → `RuntimeController.backup()` | `POST /runtime/backup` | **IMPLEMENTED** |
| Restore | `on_restore()` → `RuntimeController.restore("latest")` | `POST /runtime/restore/latest` | **IMPLEMENTED** |

### 4.2 Settings Panel Buttons (`ui/settings_panel.py`)

| Button | Handler | Status |
|---|---|---|
| Apply | `_on_apply()` — saves QSettings, applies theme, syncs AISF home to YAML | **IMPLEMENTED** |
| Browse AISF | `_browse_aisf_home()` — `QFileDialog.getExistingDirectory` | **IMPLEMENTED** |

### 4.3 Window-Level Actions

| Action | Implementation | Status |
|---|---|---|
| Exit (File > Quit, Ctrl+Q) | `closeEvent()` — saves geometry, stops timers, closes httpx | **IMPLEMENTED** |
| Settings (File > Settings, Ctrl+,) | Navigate to settings panel in stacked widget | **IMPLEMENTED** |

### Issue P4-01 — MEDIUM

| Field | Value |
|---|---|
| **Severity** | MEDIUM |
| **Finding** | System tray "Quit" action has no connected signal handler (`app/notifications.py:29`: `menu.addAction("Quit")` — no `.triggered.connect()`). Clicking it does nothing. |
| **Root cause** | Incomplete wiring during notifications implementation |
| **Affected files** | `app/notifications.py:29` |
| **Recommended fix** | `action = menu.addAction("Quit"); action.triggered.connect(QApplication.quit)` |
| **Estimated effort** | 15 minutes |

**Phase 4 Result: FAIL — 1 blocker (Update button), 1 medium issue**

---

## Phase 5 — Content Factory Integration

### 5.1 Content Factory Service State

Content Factory is a real, multi-module Java/Spring Boot application located at `E:\UserData\MyData\Content\AgentAIDev\content-factory`. It is **not documentation only**.

| Module | Status | Notes |
|---|---|---|
| `cf-worker` | **IMPLEMENTED** | 24+ worker classes: ContentPipelineWorker, VideoGenerationWorker, StoryGenerationWorker, ThumbnailWorker, CriticWorker, PublishingWorker, etc. Full SPI interface layer. |
| `cf-api` | **IMPLEMENTED** | EpisodeController, OrchestrationController, WorkflowController, PublishController, UniverseController, ProjectController |
| `cf-workflow` | **IMPLEMENTED** | Full entity + service layer (WorkflowService, CheckpointService, StartupRecoveryService) |
| `cf-publish` | **IMPLEMENTED** | Full YouTube stack (YouTubePublishingService, ChannelTokenManager, CredentialEncryptionService) |
| `cf-quality` | **IMPLEMENTED** | SPI critics for Audio, Image, Metadata, Story, Subtitle, Thumbnail, Video |
| `cf-timeline` | **PARTIAL** | No dedicated timeline module or REST endpoint. Step tracking is embedded in cf-workflow (WorkflowCheckpoint, WorkflowStepInstance). |
| `cf-rabbitmq` | **PARTIAL** | Config and topology only (2 files: RabbitMQConfig.java, RabbitTopologyProperties.java). No queue management API. |
| `cf-assets` | **PARTIAL** | 5 files only (AssetEntry, AssetKey, AssetKind, AssetMemoryStore, FileSystemAssetMemoryStore). No REST controller. In-memory/filesystem only. |

### 5.2 AISF → Content Factory Connection

**IMPLEMENTED** via `factory/plugins/content_factory_plugin.py`.

`ContentFactoryPlugin` (full REST adapter):
- `health()` → `GET /api/integration/v1/health`
- `build()` → `POST /api/integration/v1/episodes` + polls to terminal state (max 10 min)
- `test()` → validates ≥13 workers at `GET /api/integration/v1/workers`
- `review()` → `GET /api/integration/v1/episodes/{id}/dag`
- `deploy()` → `POST /api/integration/v1/episodes/{id}/{action}`

### 5.3 Desktop → CF Live Data

**NOT IMPLEMENTED — CRITICAL GAP.**

The Desktop's `PlatformApiClient` contains clients for: `queue`, `episodes`, `publishing`, `quality`, `inventory`, `worker_monitor` — all directed to `http://localhost:8088/api/v1` (AISF).

AISF registers exactly 6 routers: `routes`, `storage_routes`, `asset_routes`, `health_routes`, `runtime_routes`, `capabilities_routes`. None of these expose `/queue`, `/episodes`, `/publishing`, `/quality`, or `/inventory`.

**All content management UI panels in the Desktop will receive HTTP 404 from AISF.**

The only CF data the Desktop actually receives is: `content_factory: {status, port}` embedded in the `/health` response.

### Issue P5-01 — BLOCKER

| Field | Value |
|---|---|
| **Severity** | BLOCKER |
| **Finding** | Desktop content clients (`QueueClient`, `EpisodeClient`, `PublishingClient`, `QualityClient`, `InventoryClient`, `WorkerMonitorClient`) call AISF routes that do not exist. All content management UI panels will fail with 404. |
| **Root cause** | No proxy/bridge layer has been implemented in AISF to forward Desktop requests to CF. The architecture assumes AISF is a gateway, but the gateway routes were never built. |
| **Affected files** | `api/` (missing routes), `services/queue_client.py`, `services/episode_client.py`, `services/publishing_client.py`, `services/quality_client.py`, `services/inventory_client.py`, `services/worker_monitor_client.py` |
| **Recommended fix** | Add proxy routes in AISF that forward requests to CF: `GET /api/v1/queue` → `http://localhost:8090/api/queue`, etc. Alternatively, point Desktop content clients directly at CF (port 8090) for production. |
| **Estimated effort** | 2–4 days (build all proxy routes + integration tests) |

### Issue P5-02 — HIGH

| Field | Value |
|---|---|
| **Severity** | HIGH |
| **Finding** | CF Timeline: no dedicated timeline API endpoint in CF or AISF. Desktop `EpisodeClient.get_timeline()` calls `/episodes/{id}/timeline` — this route is missing from both CF (`cf-api`) and AISF. |
| **Root cause** | cf-timeline module was not built as a standalone REST-accessible module |
| **Affected files** | `services/episode_client.py` (Desktop), `cf-api` (missing endpoint) |
| **Recommended fix** | Add timeline endpoint to CF `EpisodeController` reading from `WorkflowCheckpoint`/`WorkflowStepInstance` entities |
| **Estimated effort** | 1 day |

### Issue P5-03 — HIGH

| Field | Value |
|---|---|
| **Severity** | HIGH |
| **Finding** | CF Assets (`cf-assets`) has 5 files and no REST controller or service layer. The Desktop `AssetsClient` calls AISF `/assets` which AISF proxies to CF, but CF has no assets API surface. |
| **Root cause** | cf-assets module is a data-model stub, not a production module |
| **Affected files** | `cf-assets/` (5 files, no controller), `services/assets_client.py` (Desktop) |
| **Recommended fix** | Implement CF AssetController + AssetService backed by FileSystemAssetMemoryStore |
| **Estimated effort** | 2–3 days |

**Phase 5 Result: FAIL — 1 blocker, 2 high severity issues**

---

## Phase 6 — Failure Recovery

| Scenario | Status | Evidence |
|---|---|---|
| AISF offline | **IMPLEMENTED** | `base_controller.py:75–88` — detects connection error, emits "Backend offline — retrying…", enters backoff |
| Exponential backoff | **IMPLEMENTED (partial)** | Doubles interval on first offline event; capped at 60s. Subsequent ticks while offline are dropped silently — interval does not re-double per tick by design |
| AISF restart recovery | **IMPLEMENTED** | Auto-heals on next successful poll: resets `_offline=False`, restores `_base_interval_ms` |
| Content Factory offline | **IMPLEMENTED** | AISF `/health` parallel-probes CF with 0.8s timeout; returns `{status: "down"}` gracefully |
| Ollama offline | **IMPLEMENTED** | Same parallel-probe pattern; returns `{status: "down"}` gracefully |
| Network timeout (Desktop→AISF) | **IMPLEMENTED (hardcoded)** | `ApiClient` default 10s timeout; not user-configurable via config file |
| CF poll timeout (AISF→CF) | **IMPLEMENTED (hardcoded)** | `_PROBE_TIMEOUT = 0.8s` in `routes.py` and `runtime_routes.py` |
| Desktop restart recovery | **IMPLEMENTED** | `main.py:16–36` installs faulthandler + sys.excepthook + threading.excepthook → `runtime/crash.log` |
| Worker crash recovery | **IMPLEMENTED** | `supervisor.py:237–244` — exponential backoff: 5s, 10s, 20s, 40s, 80s, 120s (capped) |
| No polling floods | **IMPLEMENTED** | `_offline=True` suppresses all subsequent error logs; only one error emitted per offline episode |

### Issue P6-01 — MEDIUM

| Field | Value |
|---|---|
| **Severity** | MEDIUM |
| **Finding** | Network timeouts (Desktop→AISF: 10s, AISF→CF probe: 0.8s, CF episode poll: 30s per attempt × 120 = 10 min max) are hardcoded constants. There is no mechanism to tune these without code changes. |
| **Root cause** | No configuration surface for timeout values beyond `ORCH_*` env vars in `config.py` |
| **Affected files** | `services/api_client.py:26`, `api/routes.py` (`_PROBE_TIMEOUT`), `factory/plugins/content_factory_plugin.py` |
| **Recommended fix** | Add `api_timeout_seconds`, `probe_timeout_seconds`, `cf_poll_timeout_seconds` to `config.py` |
| **Estimated effort** | 2 hours |

**Phase 6 Result: PASS with notes**

---

## Phase 7 — Performance

| Metric | Status | Endpoint/Location |
|---|---|---|
| Desktop startup time | **NOT MEASURED** | No instrumentation |
| AISF startup time | **NOT MEASURED** | Logs "Orchestrator ready" but no duration |
| CF startup time | **NOT MEASURED** | Spring Boot logs start but not surfaced to AISF or Desktop |
| Health discovery latency | **NOT MEASURED** | `_probe_http()` returns bool, no latency value |
| Capability discovery latency | **NOT MEASURED** | No instrumentation |
| CPU usage | **IMPLEMENTED** | `GET /api/v1/health/metrics` — psutil `cpu_percent(interval=0.1)` |
| RAM usage | **IMPLEMENTED** | `GET /api/v1/health/metrics` — psutil `ram_percent` |
| Disk usage | **IMPLEMENTED** | `GET /api/v1/health/metrics` — `disk_used_gb`, `disk_total_gb` |
| GPU usage | **IMPLEMENTED** | `GET /api/v1/health/metrics` — nvidia-smi (graceful fallback) |
| Thread count | **PARTIAL** | Supervisor tracks `running_count`/`total_count` but not exposed in `/health/metrics` |
| Worker count | **IMPLEMENTED** | `GET /api/v1/workers` — supervisor `get_status()` |
| APM / Observability | **MISSING** | No Prometheus, OTel, Micrometer, or StatsD integration |

### Issue P7-01 — MEDIUM

| Field | Value |
|---|---|
| **Severity** | MEDIUM |
| **Finding** | No startup time measurement across any component. No health-check latency tracking. Performance baselines cannot be established for production SLA monitoring. |
| **Root cause** | Performance instrumentation was not included in the implementation scope |
| **Affected files** | `app.py` (AISF startup), `bootstrap/launcher.py` (Desktop startup), `api/routes.py` (`_probe_http`) |
| **Recommended fix** | Add `time.perf_counter()` timing around lifespan startup; add latency field to `_probe_http()` return; expose `thread_count` in `/health/metrics` |
| **Estimated effort** | 4 hours |

**Phase 7 Result: PARTIAL — metrics exposed, latency and startup timing missing**

---

## Phase 8 — Code Audit

### 8.1 TODO / FIXME

| File | Line | Severity | Content |
|---|---|---|---|
| `factory/cli/generator.py` | 706 | INFO | `# TODO: add sprint-1 tasks here` — propagates into every generated project's `db/seed_sprint1.py` |
| `engine/root_cause_engine.py` | 96 | INFO | `# TODO: implement mock execution` — in a generated regression test stub string |
| No FIXME found | — | — | — |

### 8.2 Placeholder / Stub Code

| File | Line | Severity | Content |
|---|---|---|---|
| `scripts/release.py` | 417 | HIGH | `step_sign()` is empty — release artifacts ship unsigned |
| `installer/downloads/manager.py` | 95–96 | HIGH | `sha256:placeholder` prefix silently bypasses checksum verification |
| `installer/bootstrap/manager.py` | 150–151 | HIGH | Same `sha256:placeholder` bypass |
| `ui/production/timeline_panel.py` | 124 | MEDIUM | `if item and item.spacerItem(): pass` — spacer item detected but never removed, loop incomplete |

### 8.3 NotImplementedError

| File | Line | Severity | Content |
|---|---|---|---|
| `installer/tasks.py` | 111 | INFO | `raise NotImplementedError(...)` in abstract base class — correct pattern, not a bug |

### 8.4 Debug print() in Production Code

| File | Lines | Severity | Content |
|---|---|---|---|
| `production/sprint1_runner.py` | 114–489 (multiple) | MEDIUM | Extensive `print()` throughout runnable script; should use `logging` |

### 8.5 sleep() in Request Handlers

| File | Line | Severity | Content |
|---|---|---|---|
| `supervisor.py` | 165 | MEDIUM | `time.sleep(0.5)` inside `restart()` which is called from `POST /workers/{agent}/restart` — blocks request thread 500ms |

### 8.6 Hardcoded Paths

| File | Line | Severity | Content |
|---|---|---|---|
| `desktop-config.yaml` | 19 | **CRITICAL** | `aisf_home: "E:/UserData/MyData/Content/DEV/ai-software-factory"` — absolute developer path |
| `tests/test_runtime_consolidation.py` | 281 | LOW | `r"C:\Users\smart\AppData\Roaming\npm\claude.CMD"` — developer username in test |
| `factory/plugins/oracle_plugin.py` | 82 | MEDIUM | Default Oracle connection string with `app_pass` credential in code |

### 8.7 Hardcoded Ports Outside Config Files

| File | Lines | Severity | Content |
|---|---|---|---|
| `api/routes.py` | 45–48 | LOW | `cf_port = 8090`, `ollama_port = 11434` fallback before reading runtime config |
| `api/runtime_routes.py` | 124–125 | LOW | Same fallback pattern |
| `services/api_client.py` | 26 | MEDIUM | `"http://localhost:8088/api/v1"` hardcoded default |
| `ui/config/platform_panel.py` | 26, 77 | MEDIUM | `"http://localhost:8088/api/v1"` in widget reset path |
| `ui/settings_panel.py` | 137 | MEDIUM | Same `localhost:8088` in settings reset |

### 8.8 Dead Code / Unused Imports

| File | Line | Severity | Content |
|---|---|---|---|
| `services/api_client.py` | 7 | LOW | `import json` unused — all JSON handled by httpx |
| `scripts/release.py` | 420 | HIGH | `step_sign()` function body is entirely empty |

### Issue P8-01 — CRITICAL

| Field | Value |
|---|---|
| **Severity** | CRITICAL |
| **Finding** | `desktop-config.yaml:19` contains an absolute path `E:/UserData/MyData/Content/DEV/ai-software-factory`. Every new installation will have the wrong AISF home and auto-launch will fail silently. |
| **Root cause** | Developer committed local config without stripping the machine-specific path |
| **Affected files** | `desktop-config.yaml:19` |
| **Recommended fix** | Replace with `aisf_home: ""` or a placeholder like `"~/.aisf"`. The launcher already reads `AISF_HOME` env var as fallback. |
| **Estimated effort** | 5 minutes |

### Issue P8-02 — HIGH

| Field | Value |
|---|---|
| **Severity** | HIGH |
| **Finding** | `scripts/release.py:step_sign()` function body is empty. All release artifacts ship without any cryptographic signature. |
| **Root cause** | The signing step was scaffolded as a placeholder and never implemented |
| **Affected files** | `scripts/release.py:417–420` |
| **Recommended fix** | Implement using sigstore/cosign as noted in the comment, or remove the step entirely rather than leaving a false-positive in the pipeline |
| **Estimated effort** | 4–8 hours (depending on signing infrastructure) |

### Issue P8-03 — HIGH

| Field | Value |
|---|---|
| **Severity** | HIGH |
| **Finding** | `installer/downloads/manager.py:95–96` and `installer/bootstrap/manager.py:150–151` both contain logic that silently skips SHA-256 verification for any component whose checksum begins with `sha256:placeholder`. An attacker who can substitute a component can bypass integrity checking if the placeholder is in the manifest. |
| **Root cause** | Placeholder checksums were left in the installer manifest to unblock development |
| **Affected files** | `installer/downloads/manager.py:95–96`, `installer/bootstrap/manager.py:150–151` |
| **Recommended fix** | Remove the `sha256:placeholder` bypass. Populate all checksums in the manifest before release. |
| **Estimated effort** | 2 hours (compute real checksums) + build pipeline time |

**Phase 8 Result: FAIL — 1 critical, 3 high severity issues**

---

## Phase 9 — Integration Tests

### 9.1 Test Suite Created

**File:** `E:\UserData\MyData\Content\DEV\ai-studio-desktop\tests\test_integration_e2e.py`

**Result: 109 tests — 109 PASSED, 0 FAILED** (execution time: 2.43 seconds)

| Test Class | Count | Covers |
|---|---|---|
| `TestDesktopStartup` | 10 | Bootstrap imports, config loading, PID manager, crash hooks |
| `TestAisfStartup` | 14 | AISF module imports, all 6 routers registered, lifespan hooks |
| `TestCapabilityDiscovery` | 9 | Capability response keys, `_has_lib()`, `_port_open()` |
| `TestRuntimeApiShape` | 11 | All 9 runtime route handlers exist with correct signatures |
| `TestHealthApiShape` | 7 | Health, ping, metrics endpoints and response fields |
| `TestContentFactoryIntegration` | 14 | ContentFactoryPlugin import, health/build/test/review/deploy methods |
| `TestFailureRecovery` | 21 | Offline detection, exponential backoff, auto-recovery, error swallowing |
| `TestShutdown` | 8 | PID clean-stop, httpx session close, timer teardown |
| `TestRestart` | 15 | AisfChecker start/stop signatures, wait_until_healthy contract |

### 9.2 Pre-existing Tests

Both repos have extensive existing test coverage:

**Desktop** (`tests/`): test_api_clients, test_bootstrap_config, test_bootstrap_health, test_bootstrap_launcher, test_bootstrap_regression, test_command_catalog, test_command_client, test_config_models, test_config_panels, test_config_rest, test_config_widgets, test_copilot_ui, test_episode_wizard, test_models, test_preview_widget, test_production_rest, test_quality_publishing, test_queue_widget, test_regression, test_timeline_widget, test_v12_regression, test_v13_regression, test_widgets, test_worker_monitor

**AISF** (`tests/`): test_assets, test_autonomous_execution, test_claude_integration, test_installer, test_phase21b_autonomous_sprint, test_phase21b_chaos, test_phase22_load, test_phase22_reliability, test_phase22_security, test_phase23_factory, test_phase25_multi_project, test_phase26_bootstrap, test_phase27_release, test_phase28_adoption, test_runtime_consolidation, test_runtime_manager, test_runtime_validation, test_stall_recovery, test_storage, test_workflow_supervisor

**Phase 9 Result: PASS — 109/109 new integration tests pass**

---

## Phase 10 — Production Certification

### 10.1 Architecture Validation

| Layer | Component | Architecture Status |
|---|---|---|
| Presentation | Desktop Qt app (PySide6 ≥6.7) | **SOUND** — MVC separation, controller/service/widget layers clean |
| API Gateway | AISF FastAPI on :8088 | **SOUND** — lifespan, routers, DB, supervisor all correctly structured |
| Content Engine | CF Java/Spring Boot on :8090 | **SOUND** — modular multimodule Maven project, clean SPI interfaces |
| AI Runtime | Ollama on :11434 | **NOT AUDITED** — external dependency |
| Storage | SQLite (dev) / configurable | **SOUND** — SQLAlchemy ORM, migration-ready |
| Worker Pool | AISF supervisor (10 threads) + CF RabbitMQ workers | **SOUND** — supervisor has backoff, health tracking |

**Version Discrepancy (BLOCKING for certification):** The platform is called v1.5.1 in this audit request. All authoritative code sources report `1.0.0`:
- `platform_version.py`: `PLATFORM_VERSION = "1.0.0"`
- `ai-software-factory/pyproject.toml`: `version = "1.0.0"`
- `ai-studio-desktop/pyproject.toml`: `version = "1.0.0"`

### 10.2 API Compatibility Matrix (Complete)

| Endpoint | Method | AISF | Desktop Client | Desktop Calls? | Compat |
|---|---|---|---|---|---|
| `/api/v1/health` | GET | ✅ | HealthClient | ✅ | ✅ |
| `/api/v1/health/ping` | GET | ✅ | HealthClient | ✅ | ✅ |
| `/api/v1/health/metrics` | GET | ✅ | HealthClient | ✅ | ✅ |
| `/api/v1/runtime/state` | GET | ✅ | RuntimeClient | ✅ | ✅ |
| `/api/v1/capabilities` | GET | ✅ | CapabilitiesClient | ✅ | ✅ |
| `/api/v1/runtime/start` | POST | ✅ | RuntimeClient | ✅ | ✅ |
| `/api/v1/runtime/stop` | POST | ✅ | RuntimeClient | ✅ | ✅ |
| `/api/v1/runtime/restart` | POST | ✅ | RuntimeClient | ✅ | ✅ |
| `/api/v1/runtime/resume` | POST | ✅ | RuntimeClient | ✅ | ✅ |
| `/api/v1/runtime/doctor` | POST | ✅ | RuntimeClient | ✅ | ✅ |
| `/api/v1/runtime/repair` | POST | ✅ | RuntimeClient | ✅ | ✅ |
| `/api/v1/runtime/update` | POST | ❌ MISSING | RuntimeClient | ✅ | ❌ |
| `/api/v1/runtime/backup` | POST | ✅ | RuntimeClient | ✅ | ✅ |
| `/api/v1/runtime/restore/{id}` | POST | ✅ | RuntimeClient | ✅ | ✅ |
| `/api/v1/runtime/backups` | GET | ✅ | RuntimeClient | ✅ | ✅ |
| `/api/v1/queue/*` | ANY | ❌ MISSING | QueueClient | ✅ | ❌ |
| `/api/v1/episodes/*` | ANY | ❌ MISSING | EpisodeClient | ✅ | ❌ |
| `/api/v1/publishing/*` | ANY | ❌ MISSING | PublishingClient | ✅ | ❌ |
| `/api/v1/quality/*` | ANY | ❌ MISSING | QualityClient | ✅ | ❌ |
| `/api/v1/inventory/*` | ANY | ❌ MISSING | InventoryClient | ✅ | ❌ |
| `/api/v1/workers` | GET | ✅ | WorkerMonitorClient | ✅ | ✅ |
| `/api/v1/dashboard` | GET | ✅ | DashboardClient | ✅ | ✅ |
| `/api/v1/metrics` | GET | ✅ | MetricsClient | ✅ | ✅ |

**Compatible: 17/23 (74%). Broken: 6/23 (26%).**

### 10.3 Integration Matrix

| Desktop → | AISF → | CF | Status |
|---|---|---|---|
| RuntimeClient | `/runtime/*` | lib/start.py, stop.py, etc. | ✅ 13/14 routes work |
| HealthClient | `/health`, `/health/ping`, `/health/metrics` | Probed at `/actuator/health` | ✅ Full chain |
| CapabilitiesClient | `/capabilities` | Port-probed for liveness | ✅ Works |
| DashboardController | `/dashboard`, `/metrics` | N/A (AISF-internal) | ✅ Works |
| WorkerMonitorClient | `/workers` | Supervisor + CF `/workers` | ✅ Works |
| QueueClient | MISSING ROUTE | CF queue via RabbitMQ | ❌ 404 |
| EpisodeClient | MISSING ROUTE | CF `/api/integration/v1/episodes` | ❌ 404 |
| PublishingClient | MISSING ROUTE | CF `/api/integration/v1/publish` | ❌ 404 |
| QualityClient | MISSING ROUTE | CF quality reports | ❌ 404 |
| InventoryClient | MISSING ROUTE | CF `/api/integration/v1/*` | ❌ 404 |
| ContentFactoryPlugin | CF REST directly | CF `/api/integration/v1/*` | ✅ Full AISF→CF chain |

### 10.4 Performance Report

| Measurement | Value | Method |
|---|---|---|
| AISF health probe timeout | 0.8s | Hardcoded `_PROBE_TIMEOUT` |
| Desktop→AISF request timeout | 10s | `ApiClient` default |
| CF episode build timeout | 10 min max | 120 × 5s poll |
| Worker backoff max | 120s | Supervisor `RESTART_BACKOFF_MAX` |
| Desktop offline backoff max | 60s | `base_controller.py:82` |
| Desktop health refresh interval | 5000 ms | DashboardController |
| Desktop runtime state refresh | 3000 ms | RuntimeController |
| CPU metrics | Yes (psutil) | `/health/metrics` |
| RAM metrics | Yes (psutil) | `/health/metrics` |
| GPU metrics | Yes (nvidia-smi) | `/health/metrics` |
| Startup time measurement | **NOT MEASURED** | No instrumentation |
| Health check latency | **NOT MEASURED** | No instrumentation |
| Prometheus / OTel | **NOT PRESENT** | — |

**Bottleneck:** The Desktop polls AISF every 3–5 seconds. Under 0.8s AISF→CF probe + DB query, each health poll completes well within the interval. No flood risk under normal conditions.

### 10.5 Security Review

| Control | Status | Finding |
|---|---|---|
| Authentication | ❌ NONE | No auth on any AISF or CF endpoint. Any process on localhost can control the platform. |
| Transport security | ❌ HTTP only | All inter-service communication is plain HTTP. No TLS anywhere. |
| Input validation — runtime routes | ⚠️ PARTIAL | `backup_id` path parameter passed unsanitized to subprocess command line |
| Input validation — task routes | ✅ | Enum validation + 404/409 guards |
| Credential storage | ⚠️ | Oracle default credential in `oracle_plugin.py:82`. YouTube credentials delegated to CF (not audited in full). |
| CORS | ⚠️ | `cors_origins: ["*"]` default — explicitly flagged in config.py comment as dev-only but no production enforcement |
| Artifact signing | ❌ PLACEHOLDER | `step_sign()` in release pipeline is empty |
| Checksum verification | ❌ BYPASS | `sha256:placeholder` prefix skips integrity verification |
| Secrets in config | ❌ | `desktop-config.yaml` committed with absolute developer path |

### Issue P10-SEC-01 — CRITICAL

| Field | Value |
|---|---|
| **Severity** | CRITICAL |
| **Finding** | The AISF REST API has no authentication. Any unauthenticated caller can start/stop/restart the Content Factory, trigger backups/restores, and (via the `backup_id` path parameter) attempt command injection against the subprocess. |
| **Root cause** | Authentication was not in scope for the initial implementation |
| **Affected files** | All AISF route files |
| **Recommended fix** | Add a FastAPI dependency (`Depends(verify_api_key)`) to all sensitive routes. Minimally: require a shared secret API key in `X-API-Key` header configured via `ORCH_API_KEY` env var. |
| **Estimated effort** | 4 hours (add auth middleware) |

### Issue P10-SEC-02 — HIGH

| Field | Value |
|---|---|
| **Severity** | HIGH |
| **Finding** | `backup_id` in `POST /runtime/restore/{backup_id}` is passed directly as a subprocess argument without sanitization: `_run_lib("backup.py", ["--restore", backup_id])`. Without authentication, this is a trivially reachable injection vector. |
| **Root cause** | No input sanitization on path parameters that reach subprocess calls |
| **Affected files** | `api/runtime_routes.py:230` |
| **Recommended fix** | Validate `backup_id` against a strict allowlist (alphanumeric + hyphens only): `if not re.match(r'^[\w\-]+$', backup_id): raise HTTPException(400)` |
| **Estimated effort** | 30 minutes |

### 10.6 Deployment Checklist

- [ ] **REQUIRED** Remove hardcoded developer path from `desktop-config.yaml:19`
- [ ] **REQUIRED** Implement `POST /runtime/update` in AISF or remove Update button
- [ ] **REQUIRED** Implement AISF proxy routes for Queue, Episodes, Publishing, Quality, Inventory
- [ ] **REQUIRED** Populate real SHA-256 checksums in installer manifests
- [ ] **REQUIRED** Fix system tray "Quit" action (connect handler)
- [ ] **REQUIRED** Align version number: update `platform_version.py`, both `pyproject.toml` to `1.5.1`
- [ ] RECOMMENDED — Add API key authentication to AISF
- [ ] RECOMMENDED — Sanitize `backup_id` path parameter
- [ ] RECOMMENDED — Fix Resume capability gating (use `"resume"` key)
- [ ] RECOMMENDED — Gate Update button on capability key
- [ ] RECOMMENDED — Change `cors_origins` default from `["*"]` to localhost-only
- [ ] RECOMMENDED — Implement `step_sign()` in release pipeline
- [ ] NICE-TO-HAVE — Add startup time instrumentation
- [ ] NICE-TO-HAVE — Expose thread count in `/health/metrics`
- [ ] NICE-TO-HAVE — Replace `print()` in `sprint1_runner.py` with logging
- [ ] NICE-TO-HAVE — Fix `timeline_panel.py:124` spacer removal loop

### 10.7 Operational Checklist

- [ ] Set `AISF_HOME` environment variable or update `desktop-config.yaml` with correct path on target machine
- [ ] Set `ORCH_API_PORT` if 8088 is unavailable
- [ ] Set `CONTENT_FACTORY_BASE_URL` if CF is not on localhost:8080
- [ ] Configure `ORCH_CORS_ORIGINS` for production deployment
- [ ] Set `ORCH_DATABASE_URL` if not using SQLite default
- [ ] Verify Java is installed (required for Content Factory)
- [ ] Verify Ollama is running if AI features are required
- [ ] Verify `nvidia-smi` is available if GPU metrics are needed
- [ ] Run `bootstrap doctor` after installation to validate all dependencies
- [ ] Confirm all `sha256:placeholder` entries in installer manifests are replaced before distributing

### 10.8 Risk Assessment

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| All content panels broken on first run | HIGH | HIGH | Implement AISF proxy routes (P5-01) before any user testing |
| Update button triggers uncaught error on click | CERTAIN | MEDIUM | Disable button or implement endpoint (P2-01) |
| Wrong AISF path on any non-developer machine | HIGH | HIGH | Fix desktop-config.yaml (P8-01) immediately |
| Unauthenticated API access | HIGH (localhost-only service) | HIGH | Add API key auth before any network-exposed deployment |
| Installer bypass via placeholder checksums | LOW (internal use) | HIGH | Replace placeholders before external distribution |
| Timeline / Assets UI broken | HIGH | MEDIUM | Implement CF timeline endpoint and assets REST layer |
| System tray Quit non-functional | CERTAIN | LOW | 15-minute fix (P4-01) |

### 10.9 Remaining Work

| Priority | Item | Estimated Effort |
|---|---|---|
| P0 | Fix `desktop-config.yaml` hardcoded path | 5 minutes |
| P0 | Implement AISF proxy routes (Queue, Episodes, Publishing, Quality, Inventory) | 3–4 days |
| P0 | Implement `POST /runtime/update` or remove Update button | 2–4 hours |
| P0 | Align version number across all three repositories | 30 minutes |
| P1 | Add API key authentication to AISF routes | 4 hours |
| P1 | Sanitize `backup_id` subprocess parameter | 30 minutes |
| P1 | Populate real SHA-256 checksums; remove placeholder bypass | 2 hours |
| P1 | Implement `step_sign()` in release pipeline | 4–8 hours |
| P2 | Fix Resume capability key bug | 30 minutes |
| P2 | Gate Update button on capability key | 30 minutes |
| P2 | Fix system tray Quit handler | 15 minutes |
| P2 | Implement CF Timeline REST endpoint | 1 day |
| P2 | Implement CF Assets REST controller | 2–3 days |
| P3 | Add startup time / latency instrumentation | 4 hours |
| P3 | Replace print() with logging in sprint1_runner.py | 2 hours |
| P3 | Change CORS default to localhost-only | 30 minutes |
| P3 | Make network timeouts configurable | 2 hours |

**Total estimated remaining work before production: ~3 weeks for a single developer, ~1 week with 3 developers working in parallel on the P0/P1 items.**

---

## Production Readiness Score

| Category | Weight | Score | Notes |
|---|---|---|---|
| Platform Bootstrap | 10% | 95/100 | Chain complete; non-fatal AISF failure is correct behavior |
| Runtime API | 10% | 85/100 | 13/14 endpoints work; Update button broken |
| Capability Discovery | 10% | 70/100 | Core works; resume key bug, update not gated |
| Desktop UI | 10% | 80/100 | 8/9 buttons work; tray Quit missing handler |
| Content Factory Integration | 20% | 30/100 | AISF→CF works; Desktop→CF proxy entirely missing |
| Failure Recovery | 10% | 85/100 | Solid; timeout not configurable |
| Performance | 5% | 50/100 | Metrics present; no latency/startup instrumentation |
| Code Quality | 10% | 65/100 | Dev path in config, placeholder signing, checksum bypass |
| Security | 10% | 20/100 | No auth, HTTP only, injection risk, unsigned artifacts |
| Test Coverage | 5% | 90/100 | Extensive existing tests + 109 new integration tests pass |

### **Overall Score: 63/100 — NOT READY FOR PRODUCTION**

**Minimum viable score for production: 80/100**

The platform is production-ready for the Runtime control and Health monitoring surfaces (start/stop/restart/doctor/repair/backup/restore + metrics dashboard). It is **not production-ready** for Content Factory management, and must not be deployed to an externally-accessible network without authentication.

---

## Certification Decision

**CERTIFICATION STATUS: NOT CERTIFIED**

**Certifiable with fixes:** The architecture is sound. The blocking issues are implementation gaps, not design flaws. Once the P0 items are resolved (estimated 1 week), a re-audit of Phases 2, 3, 4, and 5 is recommended before re-certifying.

**Items that are production-ready today:**
- Platform bootstrap chain
- AISF health monitoring (CPU, RAM, GPU, service status)
- Runtime control (start, stop, restart, resume, doctor, repair, backup, restore)
- Failure recovery and exponential backoff
- Desktop status synchronization
- Worker supervisor and crash recovery
- Integration test suite (109/109 passing)

**Items that require fixes before any production use:**
- Content Factory panel integration (all 5 proxy route groups)
- `POST /runtime/update` endpoint
- Hardcoded developer path in `desktop-config.yaml`
- Version number alignment
- Authentication layer
- Checksum verification bypass removal

---

*Audit performed: 2026-06-27*
*Integration test file: `ai-studio-desktop/tests/test_integration_e2e.py` (109 tests, 0 failures)*
*Report generated from static code analysis + test execution. Live service runtime was NOT verified — marked NOT VERIFIED where applicable.*
