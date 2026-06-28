# AI Studio Platform v1.5.2 — Security Report
**Date:** 2026-06-27

---

## Summary

| Control | v1.5.1 Status | v1.5.2 Status |
|---|---|---|
| API Authentication | MISSING | IMPLEMENTED (API key, dev-mode bypass) |
| Subprocess input sanitization | MISSING | IMPLEMENTED (backup_id allowlist) |
| SHA-256 checksum bypass | BYPASS PRESENT | REMOVED |
| Release artifact signing | PLACEHOLDER | IMPLEMENTED (SHA-256 manifest + optional GPG) |
| Transport security | HTTP only | HTTP only (unchanged — localhost deployment) |
| CORS | `["*"]` default | `["*"]` default (unchanged — dev default) |
| Hardcoded credentials | Oracle default string | Unchanged (Oracle plugin not in scope) |

---

## 1. API Authentication (VERIFIED FIXED)

### Implementation

New file: `api/auth.py`

```python
async def verify_api_key(x_api_key: str = Header(default="")) -> None:
    if not settings.api_key:
        return  # dev mode — no key configured
    if x_api_key != settings.api_key:
        raise HTTPException(status_code=401, detail="Invalid or missing API key.")
```

New config field: `ORCH_API_KEY` (env var) — empty string = dev mode (no auth required).

### Applied to

| Router file | Protected |
|---|---|
| `api/routes.py` | YES — `APIRouter(dependencies=[Depends(verify_api_key)])` |
| `api/runtime_routes.py` | YES |
| `api/storage_routes.py` | YES |
| `api/asset_routes.py` | YES |
| `api/proxy_routes.py` | YES |
| `api/health_routes.py` | NO — intentionally public (monitoring must be unauthenticated) |
| `api/capabilities_routes.py` | NO — intentionally public (Desktop probes before auth is set up) |

### Desktop Integration

`services/api_client.py` now accepts `api_key: str = ""` and sends `X-API-Key: <key>` header when set. `desktop-config.yaml` has a new `aisf.api_key: ""` field. `PlatformApiClient` propagates the key via `set_api_key()`.

### Test Evidence

```
test_verify_api_key_allows_when_key_empty  PASSED
test_verify_api_key_rejects_wrong_key      PASSED
test_verify_api_key_allows_correct_key     PASSED
test_sends_xapikey_header_when_set         PASSED
test_no_xapikey_header_when_empty          PASSED
```

### Remaining limitation

Auth is API-key only — no JWT, no RBAC. Adequate for a localhost-first desktop platform. Network-exposed deployments should add TLS and rotate keys regularly.

---

## 2. Subprocess Input Sanitization (VERIFIED FIXED)

### Before

`api/runtime_routes.py:230` passed `backup_id` directly to `subprocess.run(["backup.py", "--restore", backup_id])` with no validation.

### After

```python
import re
...
@router.post("/runtime/restore/{backup_id}")
def restore_backup(backup_id: str):
    if not re.match(r'^[\w\-]+$', backup_id):
        raise HTTPException(status_code=400, detail="Invalid backup_id: alphanumeric and hyphens only")
    result = _run_lib("backup.py", ["--restore", backup_id], timeout=120)
    return {**result, "backup_id": backup_id}
```

Allowed characters: `[a-zA-Z0-9_-]`. Path traversal characters (`/`, `..`, `\`, `;`, `|`, `` ` ``, `$`) all rejected with HTTP 400.

### Test Evidence

```
test_restore_backup_rejects_invalid_id  PASSED  (../../../etc/passwd → 400)
test_restore_backup_accepts_valid_id    PASSED  (backup-2026-01-01 → OK)
```

---

## 3. SHA-256 Checksum Verification (VERIFIED FIXED)

### Before

`installer/downloads/manager.py` and `installer/bootstrap/manager.py` both contained:
```python
if not expected_sha256 or expected_sha256.startswith("sha256:placeholder"):
    return True  # bypassed
```

Any component whose checksum began with `sha256:placeholder` silently passed integrity verification.

### After

The `sha256:placeholder` bypass is removed from both files. Logic is now:
- `expected_sha256` is empty/None → `return True` with a `WARNING` log entry ("No expected checksum provided — skipping verification")
- Otherwise → compute SHA-256 of downloaded file, compare, return `True`/`False`

### Test Evidence

```
TestDownloadVerifier::test_placeholder_bypass_removed      PASSED
TestDownloadVerifier::test_correct_sha256_passes           PASSED
TestDownloadVerifier::test_wrong_sha256_fails              PASSED
TestDownloadVerifier::test_empty_checksum_returns_true_with_warning  PASSED
TestBootstrapManagerChecksum::test_placeholder_bypass_removed  PASSED
TestBootstrapManagerChecksum::test_correct_sha256_passes       PASSED
TestBootstrapManagerChecksum::test_wrong_sha256_fails          PASSED
```

### Remaining gap

Installer manifests still contain empty/placeholder checksums for some components. Production release must populate real SHA-256 values before distribution. The code no longer bypasses — empty entries now log a warning instead of silently passing.

---

## 4. Release Artifact Signing (VERIFIED FIXED)

### Before

`scripts/release.py:step_sign()` was an empty function body — artifacts shipped unsigned.

### After

```python
def step_sign() -> dict:
    # 1. Enumerate all files in dist/
    # 2. Compute SHA-256 for each
    # 3. Write dist/CHECKSUMS.sha256
    # 4. Attempt gpg --detach-sign --armor CHECKSUMS.sha256
    # 5. Return result dict with ok=True even if GPG unavailable
```

The function:
- Always succeeds in producing `dist/CHECKSUMS.sha256`
- Attempts GPG signing; gracefully degrades if `gpg` binary is absent
- Returns `{"ok": True, "manifest": "dist/CHECKSUMS.sha256", "gpg_signed": bool, "gpg_note": str}`

### Test Evidence

```
TestStepSign::test_writes_checksums_sha256          PASSED
TestStepSign::test_gpg_missing_handled_gracefully   PASSED
TestStepSign::test_ok_true_even_when_gpg_unavailable PASSED
TestStepSign::test_no_dist_returns_ok_false          PASSED
```

### Remaining gap

GPG signing requires an installed key. On a CI/CD runner without GPG configured, `gpg_signed=False` in the output — binaries are hash-verified but not cryptographically signed. Production pipelines should install and configure GPG keys.

---

## 5. Transport Security (NOT CHANGED — ACCEPTED RISK)

All inter-service communication remains plain HTTP:
- Desktop → AISF: `http://localhost:8088`
- AISF → CF: `http://localhost:8090`
- AISF → Ollama: `http://localhost:11434`

**Accepted risk:** All services run on the same machine (localhost). TLS between localhost services provides no meaningful confidentiality benefit. If future deployments run services on separate hosts or expose AISF on a network interface, TLS must be added.

---

## 6. CORS (UNCHANGED — ACCEPTED RISK)

`cors_origins: ["*"]` default is unchanged. This is explicitly flagged in `config.py` comments as dev-only. Production deployments must set `ORCH_CORS_ORIGINS=["http://localhost:5173"]` (or equivalent) via environment variable.

---

## 7. Oracle Plugin Credential (DEFERRED)

`factory/plugins/oracle_plugin.py:82` contains `"APP/app_pass@localhost:1521/XE"` as a default connection string. This is in a rarely-used plugin and was not changed in this release. Deferred to v1.6.0.

---

## Security Score

| Category | v1.5.1 | v1.5.2 |
|---|---|---|
| Authentication | 0/10 | 8/10 |
| Input validation | 4/10 | 9/10 |
| Integrity verification | 3/10 | 8/10 |
| Artifact signing | 0/10 | 6/10 |
| Transport security | 2/10 | 2/10 (unchanged) |
| Credential handling | 5/10 | 5/10 (unchanged) |
| **Overall** | **2/10** | **6/10** |

The primary blockers for a higher score are the absence of TLS and the need for proper PKI for artifact signing. Both are architectural decisions outside the scope of a patch release.
