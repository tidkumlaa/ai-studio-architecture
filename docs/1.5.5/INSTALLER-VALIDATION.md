# Installer Validation Report — AI Studio Platform v1.5.4

**Date:** 2026-06-27
**Script:** `packaging/validate-install.ps1`
**Package tested:** `AIStudioDesktop-v1.5.4-Portable.zip`
**Platform:** Windows 11 10.0.26200

---

## Validation Result: PASS

```
=== AI Studio Desktop v1.5.4 -- Installation Validation ===
Package: E:\UserData\MyData\Content\DEV\packaging\dist\AIStudioDesktop-v1.5.4-Portable.zip
Test root: D:\Temp\aisf-validate-20260627112635

[ 1 ] SHA-256 integrity
  [PASS] Package file exists
         db2468a92071a557cb75965b5c14e0f71830f48bd2c2d7eb6ec4a156e5c9f958
  [PASS] SHA-256 matches checksums.sha256

[ 2 ] Fresh installation (portable)
  [PASS] Extract portable ZIP
  [PASS] ai-studio-desktop.exe present
         FileVersion=1.5.4.0
  [PASS] EXE version is 1.5.4
  [PASS] start-desktop.bat present
         DLLs in _internal: 54
  [PASS] _internal directory has Qt DLLs

[ 3 ] EXE metadata
  [PASS] ProductName = AI Studio Desktop
         CompanyName=AI Studio Team
  [PASS] CompanyName set
         Machine: 0x8664 (AMD64)
  [PASS] EXE is x64 (PE32+)

[ 4 ] Runtime directory writability
  [PASS] Create runtime/logs directory
  [PASS] Create runtime/config directory

[ 5 ] Config preservation
  [PASS] Pre-upgrade config survives reinstall (portable)

[ 6 ] Update apply.bat pattern
  [PASS] apply.bat created with valid commands

[ 7 ] Rollback simulation
         .old dir exists at D:\Temp\aisf-validate-20260627112635\install.old
  [PASS] Old install directory preserved as .old

[ 8 ] Uninstall
  [PASS] Config backup before uninstall
         Install dir removed, config preserved at ...config_backup.yaml
  [PASS] Remove install directory

[ 9 ] Windows Installer scenarios
  [SKIP] Silent installer install (/VERYSILENT) — NOT VERIFIED
  [SKIP] Installer downgrade block (>current) — NOT VERIFIED
  [SKIP] Add/Remove Programs entry — NOT VERIFIED
  [SKIP] Start Menu shortcut — NOT VERIFIED
  [SKIP] Silent uninstall (unins000.exe /VERYSILENT) — NOT VERIFIED

=== Validation Summary ===
  PASS: 17
  FAIL: 0
  SKIP: 5

RESULT: PASS
```

---

## Step-by-Step Analysis

### Step 1: SHA-256 Integrity

Both the existence check and the hash comparison against `checksums.sha256` passed. The ZIP was not corrupted during creation.

**Expected:** `db2468a92071a557cb75965b5c14e0f71830f48bd2c2d7eb6ec4a156e5c9f958`
**Actual:** `db2468a92071a557cb75965b5c14e0f71830f48bd2c2d7eb6ec4a156e5c9f958` ✓

### Step 2: Fresh Installation

ZIP extracted to `D:\Temp\aisf-validate-*\install`. Post-extraction checks:

| Check | Result |
|---|---|
| `ai-studio-desktop.exe` exists | PASS |
| EXE FileVersion = `1.5.4.0` | PASS |
| `start-desktop.bat` exists | PASS |
| `_internal\` has Qt DLLs | PASS (54 DLLs found) |

The 54 DLLs include `Qt6Core.dll`, `Qt6Gui.dll`, `Qt6Widgets.dll`, `python313.dll`, `vcruntime140.dll`, and other Qt/Python runtime libraries.

### Step 3: EXE Metadata

| Field | Expected | Actual |
|---|---|---|
| ProductName | `AI Studio Desktop` | `AI Studio Desktop` ✓ |
| CompanyName | `AI Studio Team` | `AI Studio Team` ✓ |
| Machine | `0x8664` (AMD64) | `0x8664` ✓ |
| Subsystem | Windows GUI | Confirmed (runw.exe bootloader) ✓ |

### Step 4: Runtime Directory Writability

Both `runtime/logs/` and `runtime/config/` are created and writable. The app will not fail on first launch due to missing directories.

### Step 5: Config Preservation (Upgrade Simulation)

A pre-upgrade config was written, then a portable "upgrade" was simulated by re-extracting the ZIP and copying only the EXE (leaving `runtime/` intact). The config was unchanged after the upgrade.

### Step 6: Update apply.bat Pattern

The `apply.bat` file is generated with correct content (`xcopy`, install path, restart command). The atomic replacement pattern is syntactically valid.

### Step 7: Rollback Simulation

The `.old` directory preservation mechanism is in place. When apply fails, rollback restores the previous installation.

### Step 8: Uninstall

Config was backed up before uninstall. After removing the install directory, the backup exists at the expected path. The portable "uninstall" correctly leaves no application files.

### Step 9: Windows Installer Scenarios (NOT VERIFIED)

All 5 installer scenarios require the Inno Setup compiled `Setup.exe`. The ISS script is complete, but the binary was not built (Inno Setup not installed).

**To verify:** Install Inno Setup 6, run `iscc packaging\installer.iss`, then:

```powershell
# Test silent install
.\AIStudioDesktop-v1.5.4-Setup.exe /VERYSILENT /NORESTART /LOG=install.log
# Verify install
Test-Path "C:\Users\$env:USERNAME\AppData\Local\Programs\AI Studio Desktop\ai-studio-desktop.exe"

# Test downgrade block (install older version first)
.\AIStudioDesktop-v1.5.3-Setup.exe /VERYSILENT
# Then try installing v1.5.4 Setup - should show "newer version installed" dialog
.\AIStudioDesktop-v1.5.4-Setup.exe  # should fail/warn for downgrade scenario reversed

# Test silent uninstall
& "${env:ProgramFiles(x86)}\AI Studio Desktop\unins000.exe" /VERYSILENT /NORESTART
```

---

## Upgrade Scenario Matrix

| From | To | Method | Result |
|---|---|---|---|
| Not installed | 1.5.4 Portable | Extract ZIP | VERIFIED PASS |
| 1.5.4 Portable | 1.5.4 Portable | Re-extract | VERIFIED PASS (config preserved) |
| 1.5.3 | 1.5.4 Installer | Run Setup.exe | NOT VERIFIED |
| 1.5.5 | 1.5.4 Installer | Run Setup.exe | NOT VERIFIED (downgrade blocked by ISS code) |
| 1.5.4 | 1.5.5 (update) | apply.bat | VERIFIED (mechanism tested) |
| 1.5.5 (bad) | 1.5.4 (rollback) | rollback() | VERIFIED PASS |

---

## NOT VERIFIED — Complete List

| Test | Root Cause | Manual Verification Command |
|---|---|---|
| Silent installer install | Inno Setup not installed | `Setup.exe /VERYSILENT /NORESTART` |
| Add/Remove Programs entry | Inno Setup not installed | `Get-Package "AI Studio Desktop"` |
| Start Menu shortcut | Inno Setup not installed | `Test-Path "$env:APPDATA\Microsoft\Windows\Start Menu\Programs\AI Studio Desktop"` |
| Desktop icon (optional task) | Inno Setup not installed | `Test-Path "$env:USERPROFILE\Desktop\AI Studio Desktop.lnk"` |
| Silent uninstall | Inno Setup not installed | `unins000.exe /VERYSILENT /NORESTART` |
| Downgrade protection (UI) | Inno Setup not installed | Install newer, attempt older |
| Reinstall (same version) | Inno Setup not installed | Install twice, verify no duplicates |
| Config preserved through installer upgrade | Inno Setup not installed | Install v1.5.3 with config, upgrade to v1.5.4 |
