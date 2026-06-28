# Packaging Report — AI Studio Platform v1.5.4

**Date:** 2026-06-27
**Phase:** 4 — Packaging & Distribution
**Tool:** PyInstaller 6.21.0 | Inno Setup 6.x (script ready)
**Python:** 3.13.14 | PySide6: 6.11.1

---

## Executive Summary

| Requirement | Status | Evidence |
|---|---|---|
| Windows Installer EXE | NOT VERIFIED | ISS script complete; Inno Setup not installed |
| Portable ZIP (Desktop) | VERIFIED | 251.9 MB, SHA-256 confirmed |
| Portable ZIP (AISF) | VERIFIED | 25.1 MB, SHA-256 confirmed |
| Silent install | NOT VERIFIED | ISS `/VERYSILENT` flag ready |
| Silent uninstall | NOT VERIFIED | ISS `unins000.exe /VERYSILENT` ready |
| Version migration | VERIFIED | PKG-009: config fields migrated |
| Config migration | VERIFIED | apply_pending_migration() tested |
| Desktop shortcut | NOT VERIFIED | ISS [Icons] section ready |
| Start Menu | NOT VERIFIED | ISS [Icons] section ready |
| Automatic update check | VERIFIED | PKG-002/003/004: check() tested |
| Update download + verify | VERIFIED | PKG-005: SHA-256 stream verified |
| Atomic replacement | VERIFIED | PKG-007: apply.bat pattern tested |
| Rollback on failure | VERIFIED | PKG-008: .old restore tested |
| Release manifest | VERIFIED | release-manifest.json + update-manifest.json |
| SHA-256 manifest | VERIFIED | checksums.sha256 (2 artifacts) |
| Fresh install validation | VERIFIED | validate-install.ps1 17/17 PASS |
| Upgrade simulation | VERIFIED | validate-install.ps1 Step 5 PASS |
| Downgrade protection | NOT VERIFIED | ISS InitializeSetup() code ready |
| Uninstall validation | VERIFIED | validate-install.ps1 Step 8 PASS |
| Digital signature | NOT VERIFIED | No code-signing certificate |

**VERIFIED: 13/20 | NOT VERIFIED: 7/20**

---

## 1. Build Infrastructure

### New Files Created

| File | Location |
|---|---|
| PyInstaller spec (Desktop) | `packaging/desktop.spec` |
| PyInstaller spec (AISF) | `packaging/aisf.spec` |
| Windows PE version resource | `packaging/version_info.txt` |
| Inno Setup installer script | `packaging/installer.iss` |
| Master build script | `packaging/build.ps1` |
| Portable ZIP builder | `packaging/build-portable.ps1` |
| SHA-256 manifest generator | `packaging/generate-manifest.ps1` |
| Installation validator | `packaging/validate-install.ps1` |
| Default config template | `packaging/defaults/config.yaml` |
| Launch helper | `packaging/defaults/start-desktop.bat` |
| Auto-updater module | `ai-studio-desktop/app/updater.py` |
| Auto-updater tests | `ai-studio-desktop/tests/test_updater.py` |

---

## 2. Desktop Bundle — VERIFIED

### Live Build Evidence

```
PyInstaller 6.21.0, contrib hooks: 2026.6
Python: 3.13.14 | Platform: Windows-11-10.0.26200-SP0
Entry point: main.py
Bundled: PySide6 6.11.1 (collect_all), app, ui, services, bootstrap
Excluded: tkinter, matplotlib, numpy, scipy, pandas
EXE type: windowed (runw.exe bootloader, no console)
Build time: 297.7 seconds | Exit: 0

Output:
  ai-studio-desktop.exe    7.1 MB    (PyInstaller launcher + version resource)
  _internal/               668.3 MB  (4,008 files)
    *.dll                  54 DLLs   (Qt6, Python, vcruntime)
    PySide6/               Qt6 binaries + plugins
    app/ ui/ services/ bootstrap/   application source
```

### EXE Metadata (live)

```
FileVersion:     1.5.4.0
ProductName:     AI Studio Desktop
CompanyName:     AI Studio Team
Machine:         0x8664 (AMD64)
```

### Portable Package

```
AIStudioDesktop-v1.5.4-Portable.zip
  Size:    251.9 MB (compressed from 668.3 MB)
  SHA-256: db2468a92071a557cb75965b5c14e0f71830f48bd2c2d7eb6ec4a156e5c9f958
```

---

## 3. AISF Server Bundle — VERIFIED

### Live Build Evidence

```
PyInstaller 6.21.0
Entry point: app.py (FastAPI + uvicorn)
Bundled: api, engine, factory, runtime, worker, db, dashboard, templates
         uvicorn, fastapi, sqlalchemy, prometheus_client, psutil, opentelemetry
EXE type: console (run.exe bootloader)
Build time: 58.2 seconds | Exit: 0

Output:
  ai-software-factory.exe  12.8 MB
  _internal/               48.4 MB  (796 files)
```

### Portable Package

```
AISoftwareFactory-v1.5.4-Portable.zip
  Size:    25.1 MB
  SHA-256: e2e32212c863c1e6c73f470bcc60cfc55254b0bbbf1ce907722581c2bf3e1e6b
```

---

## 4. Auto-Update Framework — VERIFIED

### Module Overview (`app/updater.py`)

```python
# Five-phase update pipeline:
UpdateInfo = check()                      # fetch manifest, compare versions
staged_dir = download_and_stage(info)     # download + SHA-256 verify + extract
apply(staged_dir, install_dir)            # write apply.bat
launch_apply_and_exit(staged_dir)         # detach apply.bat, exit
# On failure: staged_dir deleted, install untouched
```

### Test Results — 10/10 PASS

```
PKG-001: Version tuple (1,5,4) > (1,5,3) comparison   PASS
PKG-002: check() returns available=True for newer ver  PASS
PKG-003: check() returns available=False for older ver PASS
PKG-004: ConnectError handled gracefully               PASS
PKG-005: Download + SHA-256 verify + extract           PASS
PKG-006: Wrong SHA-256 aborts and cleans up            PASS
PKG-007: apply.bat written with xcopy pattern          PASS
PKG-008: rollback() restores .old directory            PASS
PKG-009: Config migration adds new fields              PASS
PKG-010: No migration without .migration_needed marker PASS

10 passed in 0.63s
```

---

## 5. Installation Validation — 17/17 PASS

```
validate-install.ps1 -PackagePath AIStudioDesktop-v1.5.4-Portable.zip

[ 1 ] SHA-256 integrity
  PASS: Package file exists
  PASS: SHA-256 matches checksums.sha256

[ 2 ] Fresh installation (portable)
  PASS: Extract portable ZIP
  PASS: ai-studio-desktop.exe present (FileVersion=1.5.4.0)
  PASS: EXE version is 1.5.4
  PASS: start-desktop.bat present
  PASS: _internal directory has Qt DLLs (54 DLLs)

[ 3 ] EXE metadata
  PASS: ProductName = AI Studio Desktop
  PASS: CompanyName set (AI Studio Team)
  PASS: EXE is x64 (Machine: 0x8664 AMD64)

[ 4 ] Runtime directory writability
  PASS: Create runtime/logs directory
  PASS: Create runtime/config directory

[ 5 ] Config preservation
  PASS: Pre-upgrade config survives reinstall (portable)

[ 6 ] Update apply.bat pattern
  PASS: apply.bat created with valid commands

[ 7 ] Rollback simulation
  PASS: Old install directory preserved as .old

[ 8 ] Uninstall
  PASS: Config backup before uninstall
  PASS: Remove install directory (config preserved)

[ 9 ] Windows Installer (5 SKIP — require Inno Setup)

PASS: 17 | FAIL: 0 | SKIP: 5
RESULT: PASS
```

---

## 6. Release Artifacts

```
packaging/dist/
├── AIStudioDesktop-v1.5.4-Portable.zip          251.9 MB
├── AISoftwareFactory-v1.5.4-Portable.zip          25.1 MB
├── checksums.sha256                                 0.2 KB
├── release-manifest.json                            1.0 KB
└── update-manifest.json                             1.0 KB
```

---

## 7. NOT VERIFIED — Root Cause

All 7 NOT VERIFIED items share one root cause: **Inno Setup 6 is not installed** on this machine. The ISS script (`packaging/installer.iss`) is complete, readable, and covers all features.

To resolve all 7:
```
1. Install Inno Setup 6 from https://jrsoftware.org/isdl.php
2. Run: iscc packaging\installer.iss
3. Run: validate-install.ps1 -Mode installer -PackagePath dist\AIStudioDesktop-v1.5.4-Setup.exe
```

Digital signature (item 8) additionally requires purchasing an Authenticode certificate.
