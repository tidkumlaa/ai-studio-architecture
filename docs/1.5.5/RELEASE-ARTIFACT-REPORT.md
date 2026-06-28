# Release Artifact Report — AI Studio Platform v1.5.4

**Date:** 2026-06-27
**Build environment:** Windows 11 10.0.26200, Python 3.13.14, PyInstaller 6.21.0

---

## Build Evidence

### Desktop Build

```
PyInstaller 6.21.0, contrib hooks: 2026.6
Python: 3.13.14
Platform: Windows-11-10.0.26200-SP0
Python environment: ai-studio-desktop\.venv

Analysis:
  Entry point: main.py
  Source packages: app, ui, services, bootstrap
  Hidden imports: 25 (PySide6, psutil, httpx, yaml, logging.handlers, ...)
  PySide6 hooks: collect_all() — full Qt6 bundle
  Excluded: tkinter, matplotlib, numpy, scipy, pandas

Build output:
  PYZ (bytecode archive): OK
  PKG (CArchive):         OK
  EXE (bootloader):       OK — runw.exe (windowed, no console)
  COLLECT:                OK

Build time: 297.7 seconds
Exit code: 0
Warnings: 3 (missing optional SQL driver DLLs — not used)
```

### AISF Build

```
PyInstaller 6.21.0
Entry point: app.py
Source packages: api, engine, factory, runtime, worker, db, dashboard, templates
Hidden imports: uvicorn, fastapi, sqlalchemy.dialects.sqlite, pydantic,
                prometheus_client, factory.logging, factory.metrics, factory.tracing

Build output:
  PYZ: OK
  PKG: OK
  EXE: OK — run.exe (console, server mode)
  COLLECT: OK

Build time: 58.2 seconds
Exit code: 0
```

---

## Artifacts

### Desktop Portable ZIP

| Property | Value |
|---|---|
| File | `AIStudioDesktop-v1.5.4-Portable.zip` |
| Size | 251.9 MB |
| SHA-256 | `db2468a92071a557cb75965b5c14e0f71830f48bd2c2d7eb6ec4a156e5c9f958` |
| Python | 3.13.14 (bundled) |
| PySide6 | 6.11.1 (bundled) |
| Architecture | x64 (AMD64) |
| EXE size | 7.1 MB |
| Bundle files | 4,008 files, 668.3 MB (compressed to 251.9 MB) |

**EXE version info:**
```
FileVersion:   1.5.4.0
ProductVersion: 1.5.4.0
ProductName:   AI Studio Desktop
CompanyName:   AI Studio Team
FileDescription: AI Studio Desktop — Native Control Center for AI Software Factory
OriginalFilename: ai-studio-desktop.exe
LegalCopyright: Copyright (C) 2026 AI Studio Team
```

**SHA-256 verification (live):**
```powershell
(Get-FileHash AIStudioDesktop-v1.5.4-Portable.zip -Algorithm SHA256).Hash
DB2468A92071A557CB75965B5C14E0F71830F48BD2C2D7EB6EC4A156E5C9F958
```

### AISF Portable ZIP

| Property | Value |
|---|---|
| File | `AISoftwareFactory-v1.5.4-Portable.zip` |
| Size | 25.1 MB |
| SHA-256 | `e2e32212c863c1e6c73f470bcc60cfc55254b0bbbf1ce907722581c2bf3e1e6b` |
| Python | 3.13.14 (bundled) |
| Architecture | x64 (AMD64) |
| EXE size | 12.8 MB |
| Bundle files | 796 files, 48.4 MB |

### Manifests

```
release-manifest.json    (1.0 KB)  — full product release descriptor
update-manifest.json     (1.0 KB)  — auto-updater pointer
checksums.sha256         (0.2 KB)  — GNU sha256sum format
```

---

## Windows Installer (NOT VERIFIED)

The Inno Setup script (`packaging/installer.iss`) is complete and ready to build. Requires `iscc.exe` from Inno Setup 6.x.

**Installer features specified in ISS:**
- Silent install: `/VERYSILENT /NORESTART`
- Silent uninstall: `unins000.exe /VERYSILENT`
- Start Menu group: `AI Studio Desktop`
- Desktop shortcut: optional task
- Minimum OS: Windows 10 Build 19041
- Architecture: x64 only
- Privilege: user-level (no admin required)
- Downgrade protection: `InitializeSetup()` code blocks if installed > current
- Config migration marker: written in `ssPostInstall` step
- Log directory created: `{app}\runtime\logs`
- Config preserved on upgrade: `onlyifdoesntexist` flag

**Build command:**
```
iscc packaging\installer.iss
```

**Status: NOT VERIFIED — Inno Setup not installed on this machine.**

---

## Checksums File

```
db2468a92071a557cb75965b5c14e0f71830f48bd2c2d7eb6ec4a156e5c9f958  AIStudioDesktop-v1.5.4-Portable.zip
e2e32212c863c1e6c73f470bcc60cfc55254b0bbbf1ce907722581c2bf3e1e6b  AISoftwareFactory-v1.5.4-Portable.zip
```

---

## Packaging Source Files

All packaging scripts are in `E:\UserData\MyData\Content\DEV\packaging\`:

| File | Purpose |
|---|---|
| `desktop.spec` | PyInstaller spec — Desktop (PySide6, windowed) |
| `aisf.spec` | PyInstaller spec — AISF (FastAPI server, console) |
| `version_info.txt` | Windows PE version resource |
| `installer.iss` | Inno Setup 6 script |
| `build.ps1` | Master build orchestrator |
| `build-portable.ps1` | Portable ZIP creator |
| `generate-manifest.ps1` | SHA-256 + JSON manifest generator |
| `validate-install.ps1` | Installation validation harness |
| `defaults/config.yaml` | Default user configuration |
| `defaults/start-desktop.bat` | Launch helper |

---

## Digital Signature Status

**Status: NOT SIGNED**

The EXEs are unsigned. Windows SmartScreen will show a "Windows protected your PC" dialog on first launch for unsigned executables downloaded from the internet.

Production deployment should use an Authenticode certificate:
- EV Code Signing Certificate (eliminates SmartScreen warning immediately)
- Standard Code Signing Certificate (reputation builds over time)

**Signing command (when certificate available):**
```powershell
signtool sign /f cert.pfx /p $pass /t http://timestamp.digicert.com /fd sha256 `
    dist\ai-studio-desktop\ai-studio-desktop.exe
```

---

## Reproducibility

The build is reproducible given:
- Same Python venv (`python 3.13.14`, `PySide6 6.11.1`, `PyInstaller 6.21.0`)
- Same source code state
- Same `desktop.spec`

Binary content differs between runs due to PyInstaller bootloader timestamps. SHA-256 changes on each rebuild. Deploy from a single canonical build.
