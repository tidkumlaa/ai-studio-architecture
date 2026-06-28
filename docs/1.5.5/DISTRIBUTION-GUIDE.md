# Distribution Guide — AI Studio Platform v1.5.4

**Date:** 2026-06-27

---

## Artifacts Overview

| Artifact | Size | Purpose |
|---|---|---|
| `AIStudioDesktop-v1.5.4-Portable.zip` | 251.9 MB | Extract-and-run, no installer |
| `AISoftwareFactory-v1.5.4-Portable.zip` | 25.1 MB | AISF backend, standalone |
| `AIStudioDesktop-v1.5.4-Setup.exe` | NOT VERIFIED | Windows installer (Inno Setup) |
| `checksums.sha256` | 1 KB | SHA-256 for all artifacts |
| `release-manifest.json` | 1 KB | Full release metadata |
| `update-manifest.json` | 1 KB | Auto-updater pointer manifest |

**Location:** `packaging/dist/`

---

## Quick Start

### Option A: Portable (Recommended for testing)

```
1. Download AIStudioDesktop-v1.5.4-Portable.zip
2. Verify SHA-256:
   (PowerShell) (Get-FileHash .\AIStudioDesktop-v1.5.4-Portable.zip -Algorithm SHA256).Hash
   Expected:    DB2468A92071A557CB75965B5C14E0F71830F48BD2C2D7EB6EC4A156E5C9F958
3. Extract to any folder (e.g. C:\ai-studio-desktop\)
4. Run start-desktop.bat  OR  ai-studio-desktop.exe directly
```

### Option B: Windows Installer (NOT VERIFIED — requires Inno Setup build)

```
1. Download AIStudioDesktop-v1.5.4-Setup.exe
2. Verify SHA-256 from checksums.sha256
3. Double-click to run with UI wizard  OR
   Silent install: Setup.exe /VERYSILENT /NORESTART /LOG=install.log
4. Desktop shortcut + Start Menu created automatically
5. Uninstall via Control Panel > Programs or:
   Silent uninstall: unins000.exe /VERYSILENT /NORESTART
```

---

## Building the Installer

### Prerequisites

- Python 3.11+ with venvs configured (already present)
- PyInstaller 6.21.0 (installed in both venvs)
- Inno Setup 6.x: Download from https://jrsoftware.org/isdl.php

### Build Steps

```powershell
# 1. Full build (Desktop + AISF + ZIPs + manifest)
cd E:\UserData\MyData\Content\DEV\packaging
.\build.ps1

# 2. Desktop only
.\build.ps1 -Target desktop

# 3. Skip installer (no Inno Setup)
.\build.ps1 -SkipInstaller

# 4. Clean rebuild
.\build.ps1 -Clean
```

### Build Times (measured)

| Component | Build Time | Output Size |
|---|---|---|
| Desktop (PyInstaller) | 297.7 s | 668.3 MB (bundle), 251.9 MB (ZIP) |
| AISF (PyInstaller) | 58.2 s | 48.4 MB (bundle), 25.1 MB (ZIP) |
| Portable ZIPs | ~2 min (Desktop ZIP) | — |
| SHA-256 manifest | < 1 s | — |
| Inno Setup installer | ~30 s (estimated) | ~200 MB (NOT VERIFIED) |

---

## Directory Structure (Portable Package)

```
AIStudioDesktop-v1.5.4-Portable\
├── ai-studio-desktop.exe        (7.1 MB — PyInstaller launcher)
├── start-desktop.bat            (launches the EXE)
└── _internal\                   (668 MB total bundle)
    ├── ai-studio-desktop.pkg    (Python bytecode archive)
    ├── base_library.zip         (stdlib)
    ├── *.dll                    (Qt6, Python, system DLLs)
    ├── PySide6\                 (Qt6 binaries, plugins, QML)
    ├── app\                     (application source)
    ├── ui\                      (UI source)
    ├── services\                (API client layer)
    └── bootstrap\               (startup modules)
```

Runtime directories (created on first launch):
```
<install-root>\runtime\
├── logs\         desktop.log (rotating, 10 MB × 5 backups)
├── config\       config.yaml (first-launch defaults)
├── backups\      configuration backups
└── crash.log     fault handler output
```

---

## Version Migration

### Portable Upgrade

1. Download new portable ZIP
2. Verify SHA-256
3. Extract to same folder (overwrites app files)
4. `runtime\` directory is NOT overwritten — all config and logs preserved
5. On first launch: migration module checks `.migration_needed` marker

### Installer Upgrade

The Inno Setup script:
- Blocks downgrades (`InitializeSetup` code section)
- Preserves `runtime\config\` (installed with `onlyifdoesntexist` flag)
- Writes `.migration_needed` marker in `ssPostInstall` step
- Migration runs on next app launch

### Rollback

1. The `apply.bat` update helper preserves `install_dir.old` until success is confirmed
2. If apply fails: `install_dir.old` is renamed back
3. Manual rollback: `app.updater.rollback(version, install_dir)` restores `.old` directory

---

## AISF Backend Distribution

The AISF backend (`ai-software-factory.exe`) is a standalone Windows executable:

```
AISoftwareFactory-v1.5.4-Portable\
├── ai-software-factory.exe      (12.8 MB — PyInstaller launcher)
├── start-aisf.bat               (launches on 0.0.0.0:8088)
└── _internal\                   (48.4 MB total)
    ├── ai-software-factory.pkg  (Python bytecode archive)
    ├── *.dll                    (Python, system DLLs)
    ├── api\                     (FastAPI routes)
    ├── factory\                 (logging, metrics, tracing)
    ├── engine\                  (orchestrator)
    └── templates\               (project templates)
```

### Install AISF as Windows Service (NOT VERIFIED)

```powershell
# Using NSSM (Non-Sucking Service Manager)
nssm install AISoftwareFactory "C:\aisf\ai-software-factory.exe"
nssm set AISoftwareFactory AppParameters "--host 0.0.0.0 --port 8088"
nssm set AISoftwareFactory Start SERVICE_AUTO_START
nssm start AISoftwareFactory
```

---

## SHA-256 Verification

```
# checksums.sha256 content:
db2468a92071a557cb75965b5c14e0f71830f48bd2c2d7eb6ec4a156e5c9f958  AIStudioDesktop-v1.5.4-Portable.zip
e2e32212c863c1e6c73f470bcc60cfc55254b0bbbf1ce907722581c2bf3e1e6b  AISoftwareFactory-v1.5.4-Portable.zip
```

Verify on Windows:
```powershell
Get-FileHash .\AIStudioDesktop-v1.5.4-Portable.zip -Algorithm SHA256
# Expected: DB2468A92071A557CB75965B5C14E0F71830F48BD2C2D7EB6EC4A156E5C9F958
```

Verify on Linux/Mac:
```bash
sha256sum AIStudioDesktop-v1.5.4-Portable.zip
# Expected: db2468a92071a557cb75965b5c14e0f71830f48bd2c2d7eb6ec4a156e5c9f958
```

---

## Code Signing (NOT VERIFIED)

The release is currently unsigned. To add Authenticode signatures:

```powershell
# Sign with a self-signed certificate (development only)
$cert = New-SelfSignedCertificate -Type CodeSigning -Subject "CN=AI Studio Team" `
    -CertStoreLocation "cert:\CurrentUser\My"
Set-AuthenticodeSignature -Certificate $cert `
    -FilePath "dist\ai-studio-desktop\ai-studio-desktop.exe"

# Sign with a trusted certificate (production)
signtool sign /f certificate.pfx /p password /t http://timestamp.digicert.com /fd sha256 `
    dist\AIStudioDesktop-v1.5.4-Setup.exe

# Sign the portable EXE before zipping
signtool sign /f certificate.pfx /p password /t http://timestamp.digicert.com /fd sha256 `
    dist\ai-studio-desktop\ai-studio-desktop.exe
```

Windows SmartScreen will flag unsigned executables. For distribution to end users, an EV code-signing certificate is recommended.

---

## System Requirements

| Requirement | Value |
|---|---|
| OS | Windows 10 2004 (Build 19041) or later |
| Architecture | x64 (AMD64) |
| RAM | 512 MB minimum, 2 GB recommended |
| Disk | 1 GB for portable package + runtime data |
| Network | Required for AISF connectivity; optional for offline/portable use |
| Python | Bundled (Python 3.13.14, no separate install required) |
| Qt | Bundled (PySide6 6.11.1, no separate install required) |
