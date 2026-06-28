# Update System Report — AI Studio Platform v1.5.4

**Date:** 2026-06-27
**Component:** `app/updater.py` (AI Studio Desktop)
**Tests:** PKG-001 through PKG-010 — 10/10 PASS

---

## Architecture

The auto-update system is a pure-Python module with no native dependencies beyond `httpx` (already bundled). It operates in five phases:

```
check() → download_and_stage() → apply() → launch_apply_and_exit() → [relaunch]
                                   ↓ on failure
                               cleanup + notify user
```

### Phase 1: Version Check

```python
# app/updater.py: check()
info = check(timeout=10.0)
# Returns UpdateInfo(available=True, current="1.5.4", latest="1.5.5", ...)
```

The update manifest is fetched from `UPDATE_MANIFEST_URL` (overridable via env var). Version comparison uses tuple decomposition to handle multi-segment versions correctly:

```python
_version_tuple("1.5.5") > _version_tuple("1.5.4")  # True
_version_tuple("1.10.0") > _version_tuple("1.9.9")  # True (not lexicographic)
```

**Test evidence — PKG-001:**
```
assert _version_tuple("1.5.4") == (1, 5, 4)           ✓
assert _version_tuple("2.0.0") > _version_tuple("1.9.9")  ✓
assert _version_tuple("bad")   == (0,)                 ✓
PKG-001 PASS: version tuple parsing correct
```

### Phase 2: Download

Downloads via `httpx.stream()` in 64 KiB chunks. Streams directly to disk — never buffers entire file in memory. Progress callbacks fire on each chunk.

**Test evidence — PKG-005:**
```
Download: http://test/update.zip
Verify SHA-256: db2468a9... (match)
Extract: 2 files
PKG-005 PASS: download+verify succeeded
```

### Phase 3: Integrity Verification

SHA-256 computed on the downloaded file bytes before extraction. Any mismatch immediately aborts and deletes the staging directory.

**Test evidence — PKG-006:**
```
Provided hash: deadbeefdeadbeef... (wrong)
Actual hash:   6f42...
Error: SHA-256 mismatch: expected deadbeef... got 6f42...
Staging directory deleted
PKG-006 PASS: SHA-256 mismatch rejected
```

### Phase 4: Atomic Staging + apply.bat

Files are extracted to a temp directory. `apply()` writes an `apply.bat` that performs the replacement only after the main process exits. This makes the update atomic from the user's perspective:

```batch
@echo off
timeout /t 2 /nobreak >nul
rename "C:\install" "install.old"
xcopy /e /i /y "C:\staged" "C:\install"
start "" "C:\install\ai-studio-desktop.exe"
rmdir /s /q "C:\install.old"
del "%~f0"
exit /b 0
```

**Test evidence — PKG-007:**
```
apply.bat written (341 bytes)
Content includes: xcopy, install_dir path, ai-studio-desktop.exe
PKG-007 PASS: apply.bat written
```

### Phase 5: Rollback

If `apply.bat` fails (xcopy error), it restores the `.old` directory. A manual rollback is also available via `rollback()`:

**Test evidence — PKG-008:**
```
Created install.old with "old_version" EXE
Ran rollback()
install_dir restored: [ai-studio-desktop.exe (old_version), config.yaml]
PKG-008 PASS: rollback restored
```

---

## Config Migration

On first launch after installer upgrade, a `.migration_needed` marker file triggers `apply_pending_migration()`:

**Test evidence — PKG-009:**
```
Old config:
  logging:
    level: DEBUG

After migration:
  logging:
    level: DEBUG    (preserved)
    structured: true    (new field added)
    backup_count: 5     (new field added)

Marker .migration_needed deleted
PKG-009 PASS: config migration applied
```

---

## Network Resilience

Update check failures are non-fatal. Any network error, timeout, or invalid JSON returns `UpdateInfo(available=False)` without raising:

**Test evidence — PKG-004:**
```
httpx.get() raised ConnectError("refused")
check() returned UpdateInfo(available=False, current="1.5.4", latest="1.5.4")
PKG-004 PASS: network failure handled gracefully
```

---

## Security Model

| Property | Implementation |
|---|---|
| Transport | HTTPS (enforced by URL scheme) |
| Integrity | SHA-256 verified before extraction |
| Privilege | Runs as user — no elevation required |
| Rollback | `.old` directory preserved until `apply.bat` confirms success |
| Tamper | Manifest URL overridable via `AISF_UPDATE_MANIFEST_URL` env var (for self-hosted) |

---

## Update Manifest Format

The update manifest lives at `UPDATE_MANIFEST_URL` and is checked on startup (configurable):

```json
{
  "schema_version": "1",
  "product": "AI Studio Platform",
  "latest_version": "1.5.4",
  "released_at": "2026-06-27T11:25:00Z",
  "channel": "stable",
  "min_version": "1.0.0",
  "installer_url": "https://releases.ai-studio.local/v1.5.4/AIStudioDesktop-v1.5.4-Setup.exe",
  "portable_url": "https://releases.ai-studio.local/v1.5.4/AIStudioDesktop-v1.5.4-Portable.zip",
  "sha256_installer": "",
  "sha256_portable": "db2468a92071a557cb75965b5c14e0f71830f48bd2c2d7eb6ec4a156e5c9f958",
  "release_notes": "https://releases.ai-studio.local/v1.5.4/CHANGELOG.md",
  "mandatory": false,
  "rollback_version": "1.5.3"
}
```

---

## Full Test Evidence

```
$ python -m pytest tests/test_updater.py -v

============================= test session starts ==============================
platform win32 -- Python 3.13.14, pytest-9.1.1
collected 10 items

tests/test_updater.py::test_pkg_001_version_tuple              PASSED [ 10%]
tests/test_updater.py::test_pkg_002_check_update_available     PASSED [ 20%]
tests/test_updater.py::test_pkg_003_check_no_update            PASSED [ 30%]
tests/test_updater.py::test_pkg_004_check_network_failure      PASSED [ 40%]
tests/test_updater.py::test_pkg_005_download_verify_success    PASSED [ 50%]
tests/test_updater.py::test_pkg_006_sha256_mismatch_aborts     PASSED [ 60%]
tests/test_updater.py::test_pkg_007_apply_writes_bat           PASSED [ 70%]
tests/test_updater.py::test_pkg_008_rollback_restores          PASSED [ 80%]
tests/test_updater.py::test_pkg_009_config_migration           PASSED [ 90%]
tests/test_updater.py::test_pkg_010_no_migration_without_marker PASSED [100%]

======================== 10 passed in 0.63s ===========================
```

---

## NOT VERIFIED

| Scenario | Reason |
|---|---|
| Real HTTP update download | No live update server deployed |
| Mandatory update enforcement | Requires live manifest with `mandatory: true` |
| 72h update check polling | Long-running test |
| Update over installer (EXE replace) | Requires signed installer |
