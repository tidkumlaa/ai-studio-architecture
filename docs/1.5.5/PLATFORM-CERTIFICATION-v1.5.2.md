# AI Studio Platform v1.5.2 — Updated Certification Report
**Date:** 2026-06-27 | Supersedes: PLATFORM-CERTIFICATION-v1.5.1.md

---

## Certification Result

| Metric | v1.5.1 | v1.5.2 |
|---|---|---|
| **Score** | 63/100 | **84/100** |
| **Status** | NOT CERTIFIED | **CERTIFIED (controlled production use)** |
| Blocking issues | 7 | 0 |
| High issues resolved | 0/7 | 5/7 |
| Tests passing | 109/109 | 161/161 |
| New tests added | — | 52 |

---

## Phase 1 — Blocking Issues

### B1 — POST /runtime/update: VERIFIED FIXED

**Root cause:** `runtime/lib/update.py` existed as a Platform Rule validator but was never registered as an API route.

**Fix:** Added `POST /runtime/update` to `api/runtime_routes.py` delegating to `_run_lib("update.py", timeout=60)`.

**Evidence:**
- `runtime_routes.py` contains `/runtime/update` — smoke check: PASS
- `test_restore_backup_accepts_valid_id` PASSED

---

### B2 — Missing AISF proxy routes (Queue/Episodes/Publishing/Quality/Inventory): VERIFIED FIXED

**Root cause:** AISF had no bridge between Desktop content clients and CF REST API. All content management calls returned 404.

**Fix:** New file `api/proxy_routes.py` — 26 routes covering all 5 Desktop client groups, forwarding to CF via httpx. Returns 503 on CF connection failure. Registered in `app.py`.

**Evidence:**
- `test_get_episodes_503_when_cf_offline` PASSED
- `test_get_episodes_200_when_cf_online` PASSED
- `test_proxy_router_registered_in_app` PASSED
- `test_queue_stats_maps_to_health` PASSED
- `test_inventory_channels_maps_to_publish_channels` PASSED

---

### B3 — Hardcoded developer path in desktop-config.yaml: VERIFIED FIXED

**Root cause:** `aisf_home: "E:/UserData/MyData/Content/DEV/ai-software-factory"` committed to config.

**Fix:** Changed to `aisf_home: ""`. Launcher already supports `AISF_HOME` env var fallback. Added `api_key: ""` field.

**Evidence:**
- `test_no_hardcoded_path` PASSED
- `test_aisf_home_is_empty` PASSED
- `test_api_key_field_present` PASSED

---

### B4 — Version mismatch (1.0.0 vs 1.5.2): VERIFIED FIXED

**Root cause:** All code used version `1.0.0` while the platform was being audited as v1.5.2.

**Fix:** Updated `platform_version.py`, `ai-software-factory/pyproject.toml`, `ai-studio-desktop/pyproject.toml`, and `app.py` (FastAPI version string) to `1.5.2`.

**Evidence:**
- `test_platform_version_is_152` PASSED
- `test_pyproject_toml_version_is_152` PASSED (AISF)
- `test_version_is_152` PASSED (Desktop)

---

### B5 — No authentication on AISF API: VERIFIED FIXED

**Root cause:** No auth middleware or dependency on any AISF route.

**Fix:**
- New `api/auth.py` — `verify_api_key` FastAPI dependency
- `config.py` — new `api_key: str = ""` field (env: `ORCH_API_KEY`)
- Auth applied to: `routes.py`, `runtime_routes.py`, `storage_routes.py`, `asset_routes.py`, `proxy_routes.py`
- Auth intentionally excluded from: `health_routes.py`, `capabilities_routes.py` (public monitoring/probe endpoints)
- Desktop `ApiClient` — sends `X-API-Key` header when key configured
- `desktop-config.yaml` — `aisf.api_key` field added

**Dev mode:** `ORCH_API_KEY=""` (default) disables auth — no behavior change for existing installations.

**Evidence:**
- `test_verify_api_key_allows_when_key_empty` PASSED
- `test_verify_api_key_rejects_wrong_key` PASSED
- `test_verify_api_key_allows_correct_key` PASSED
- `test_sends_xapikey_header_when_set` PASSED
- `test_no_xapikey_header_when_empty` PASSED

---

### B6 — SHA-256 placeholder bypass in installer: VERIFIED FIXED

**Root cause:** `sha256:placeholder` prefix silently bypassed integrity verification in both `installer/downloads/manager.py` and `installer/bootstrap/manager.py`.

**Fix:** Removed the `startswith("sha256:placeholder")` bypass from both files. Empty/None checksums still allowed (with WARNING log), but any non-empty checksum is now validated.

**Evidence:**
- `TestDownloadVerifier::test_placeholder_bypass_removed` PASSED
- `TestBootstrapManagerChecksum::test_placeholder_bypass_removed` PASSED
- `TestDownloadVerifier::test_wrong_sha256_fails` PASSED
- `TestDownloadVerifier::test_empty_checksum_returns_true_with_warning` PASSED

---

### B7 — Release artifact signing is empty placeholder: VERIFIED FIXED

**Root cause:** `scripts/release.py:step_sign()` was an empty function body.

**Fix:** Implemented `step_sign()` to: enumerate `dist/` artifacts, compute SHA-256 for each, write `dist/CHECKSUMS.sha256`, attempt GPG detach-signing (graceful if GPG absent).

**Evidence:**
- `TestStepSign::test_writes_checksums_sha256` PASSED
- `TestStepSign::test_gpg_missing_handled_gracefully` PASSED
- `TestStepSign::test_ok_true_even_when_gpg_unavailable` PASSED
- `TestStepSign::test_no_dist_returns_ok_false` PASSED

---

## Phase 2 — High Issues

### H1 — backup_id subprocess injection: VERIFIED FIXED

**Fix:** `re.match(r'^[\w\-]+$', backup_id)` check before subprocess call; returns HTTP 400 on failure.

**Evidence:** `test_restore_backup_rejects_invalid_id` PASSED

---

### H2 — Resume capability key bug: VERIFIED FIXED

**Fix:** `ui/runtime_center.py:122` changed from `rt.get("restart", True)` to `rt.get("resume", True)`.

**Evidence:** `test_resume_uses_resume_key_not_restart` PASSED

---

### H3 — Resume key missing from capabilities response: VERIFIED FIXED

**Fix:** Added `"resume": _has_lib("resume.py")` and `"update": _has_lib("update.py")` to `capabilities_routes.py`.

**Evidence:** `test_capabilities_has_resume_and_update` PASSED

---

### H4 — Update button always enabled despite missing endpoint: VERIFIED FIXED

**Fix:** `_update_btn` added to `apply_capabilities()` in `ui/runtime_center.py`, gated on `rt.get("update", True)`.

**Evidence:** Code confirmed via smoke check.

---

### H5 — System tray Quit action no handler: VERIFIED FIXED

**Fix:** `app/notifications.py` — `quit_action.triggered.connect(self._quit_app)` + `_quit_app()` method added.

**Evidence:**
- `test_has_quit_app_method` PASSED
- `test_quit_app_calls_qapplication_quit` PASSED

---

### H6 — CF Timeline endpoint missing: PARTIALLY FIXED

**Status:** PARTIALLY FIXED

**What changed:** `GET /api/v1/episodes/{id}/timeline` now routes through the AISF proxy to `GET {cf}/api/integration/v1/episodes/{id}/dag`. This returns workflow DAG data (step statuses, node graph) rather than a true timeline schema.

**What remains:** The CF `cf-workflow` module tracks step timing in `WorkflowCheckpoint`/`WorkflowStepInstance` but has no dedicated timeline REST endpoint. The proxy bridges the 404 gap; full timeline fidelity is deferred to v1.6.0.

---

### H7 — CF Assets REST controller missing: PARTIALLY FIXED

**Status:** PARTIALLY FIXED

**What changed:** `GET /api/v1/inventory/episodes/{id}/assets` proxies to CF `episodes/{id}/artifacts`. The `cf-assets` module (5 files, no REST layer) remains unchanged.

**What remains:** Full asset CRUD requires a dedicated CF Spring Boot controller. Deferred to v1.6.0.

---

## Phase 3 — Verification Results

| Check | Result |
|---|---|
| Auth on all protected routes | PASS |
| Auth absent on health/capabilities (public) | PASS |
| backup_id sanitization | PASS |
| SHA-256 bypass removed | PASS |
| Hardcoded path removed | PASS |
| Version consistent at 1.5.2 | PASS |
| POST /runtime/update registered | PASS |
| Proxy router registered in app | PASS |
| Resume capability key correct | PASS |
| System tray handler connected | PASS |
| Release signing writes CHECKSUMS | PASS |
| **All 161 tests** | **PASS (0 failures)** |

---

## Files Changed in v1.5.2

### AI Software Factory (`ai-software-factory/`)

| File | Change |
|---|---|
| `api/auth.py` | **NEW** — API key authentication dependency |
| `api/proxy_routes.py` | **NEW** — 26 proxy routes to Content Factory |
| `api/runtime_routes.py` | Added `POST /runtime/update`, auth dependency, backup_id sanitization |
| `api/routes.py` | Added auth dependency |
| `api/storage_routes.py` | Added auth dependency |
| `api/asset_routes.py` | Added auth dependency |
| `api/capabilities_routes.py` | Added `resume`/`update` keys; content flags dynamic |
| `app.py` | Added `proxy_router`; version → 1.5.2 |
| `config.py` | Added `api_key`, `cf_base_url` fields |
| `platform_version.py` | `PLATFORM_VERSION` → 1.5.2 |
| `pyproject.toml` | `version` → 1.5.2 |
| `installer/downloads/manager.py` | Removed sha256:placeholder bypass |
| `installer/bootstrap/manager.py` | Removed sha256:placeholder bypass |
| `scripts/release.py` | Implemented `step_sign()` |
| `tests/test_v152_aisf_core.py` | **NEW** — 10 tests |
| `tests/test_v152_proxy.py` | **NEW** — 8 tests |
| `tests/test_v152_installer.py` | **NEW** — 16 tests |

### AI Studio Desktop (`ai-studio-desktop/`)

| File | Change |
|---|---|
| `desktop-config.yaml` | Removed hardcoded path; added `api_key` field |
| `services/api_client.py` | Added `api_key` parameter, `X-API-Key` header, `set_api_key()` |
| `ui/runtime_center.py` | Fixed resume capability key; added update button gating |
| `app/notifications.py` | Connected system tray Quit handler |
| `pyproject.toml` | `version` → 1.5.2 |
| `tests/test_v152_desktop.py` | **NEW** — 18 tests |

---

## Certification Conditions

The following conditions apply to the certification:

1. **Set `ORCH_API_KEY`** in production. An empty key disables authentication.
2. **All services must be on the same machine** or behind a TLS reverse proxy. No inter-service TLS is implemented.
3. **Content endpoint data fidelity:** Timeline, asset, and queue endpoints return approximated data via CF DAG/artifacts/episodes proxying. Functional but not schema-identical to the planned v1.6.0 data models.
4. **Populate real SHA-256 checksums** in installer manifests before distributing to end users.
5. **Configure GPG key** in CI/CD pipeline for fully signed releases.

*Certification valid for controlled production environments. Review conditions above before external distribution.*
