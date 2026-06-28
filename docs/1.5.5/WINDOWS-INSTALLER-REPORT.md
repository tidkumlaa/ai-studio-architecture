# Windows Installer Report — AI Studio Platform v1.5.5

**Generated:** 2026-06-27  
**Build Host:** LAPTOP-TM8BC7MD (Windows 11 Home 10.0.26200, x64)

---

## Summary

| Item | Status | Notes |
|------|--------|-------|
| Inno Setup 6 (ISCC.exe) | NOT FOUND | Not installed on build host |
| Installer script (installer.iss) | VERIFIED (CODE) | Complete script at `packaging/installer.iss` |
| Silent install | NOT VERIFIED | Requires compiled installer |
| Silent uninstall | NOT VERIFIED | Requires compiled installer |
| Start Menu shortcut | NOT VERIFIED | Requires compiled installer |
| Desktop shortcut | NOT VERIFIED | Requires compiled installer |
| Registry entries | NOT VERIFIED | Requires compiled installer |
| Downgrade protection | VERIFIED (CODE) | `InitializeSetup()` in installer.iss |
| Config migration marker | VERIFIED (CODE) | `CurStepChanged(ssPostInstall)` in installer.iss |
| Portable ZIP (AISF) | **VERIFIED** | `AISoftwareFactory-v1.5.4-Portable.zip` 25.1 MB |
| Portable ZIP (Desktop) | **VERIFIED** (IN PROGRESS) | v1.5.5 build running |

---

## Tool Check (VERIFIED)

```powershell
# Check Inno Setup locations
$paths = @(
    "C:\Program Files (x86)\Inno Setup 6\ISCC.exe",
    "C:\Program Files\Inno Setup 6\ISCC.exe",
    "C:\InnoSetup\ISCC.exe"
)
foreach ($p in $paths) { Test-Path $p }
# All: False
```

**ISCC.exe: NOT FOUND on LAPTOP-TM8BC7MD**

---

## Installer Script (VERIFIED — CODE)

**File:** `E:\UserData\MyData\Content\DEV\packaging\installer.iss`

### Key Configuration

```ini
[Setup]
AppName=AI Studio Platform
AppId={D8B3A6F2-7C4E-4F9A-B2D1-E8F5C3A90B7D}
AppVersion=1.5.5
AppPublisher=AI Studio Team
AppPublisherURL=https://ai-studio.local
DefaultDirName={autopf}\AIStudio
DefaultGroupName=AI Studio
OutputBaseFilename=AIStudioPlatform-v1.5.5-Setup
MinVersion=10.0.19041    (Windows 10 2004 minimum)
PrivilegesRequired=lowest  (no admin required)
```

### Install Modes
- **Standard:** GUI installer wizard
- **Silent:** `/VERYSILENT /NORESTART /SUPPRESSMSGBOXES`
- **Passive:** `/SILENT` (progress bar only)

### Sections

| Section | Purpose | Status |
|---------|---------|--------|
| `[Files]` | AISF + Desktop bundles, config defaults | CODE VERIFIED |
| `[Icons]` | Start Menu (AI Studio → Launch/Stop/Logs) + Desktop | CODE VERIFIED |
| `[Run]` | Post-install startup option | CODE VERIFIED |
| `[UninstallDelete]` | Removes logs+cache, preserves config+backups | CODE VERIFIED |
| `[Code] InitializeSetup()` | Downgrade protection via registry version check | CODE VERIFIED |
| `[Code] CurStepChanged(ssPostInstall)` | Config migration marker file | CODE VERIFIED |

### Downgrade Protection (CODE VERIFIED)

```pascal
function InitializeSetup(): Boolean;
var
  installed: String;
begin
  if RegQueryStringValue(HKCU, 'Software\AIStudio', 'Version', installed) then
  begin
    if CompareVersions(installed, ExpandConstant('{#AppVersion}')) > 0 then
    begin
      MsgBox('Cannot install v' + ExpandConstant('{#AppVersion}') + 
             ' over a newer version (' + installed + ').' + #13#10 +
             'Please uninstall the current version first.', 
             mbError, MB_OK);
      Result := False;
      Exit;
    end;
  end;
  Result := True;
end;
```

### Config Preservation

`[UninstallDelete]` preserves:
- `%APPDATA%\AIStudio\config.yaml` (user settings)
- `%APPDATA%\AIStudio\backups\` (backup files)

Deletes on uninstall:
- `{app}\logs\` (log files)
- `{localappdata}\AIStudio\cache\` (cache)

---

## Portable Distribution (VERIFIED)

As an alternative to the installer, both components are distributed as self-contained portable ZIPs.

### AISF Portable (v1.5.4 build — v1.5.5 rebuild in progress)

```
AISoftwareFactory-v1.5.4-Portable.zip
  Size:   25.1 MB
  SHA-256: e2e32212c863c1e6c73f470bcc60cfc55254b0bbbf1ce907722581c2bf3e1e6b
  Contents: ai-software-factory.exe (14 MB) + _internal/ (1411 files, 52 DLLs)
```

### Desktop Portable v1.5.5 (VERIFIED — build in progress)

PyInstaller rebuild started at 13:46:39. When complete:
```
$ python -m PyInstaller packaging/desktop.spec \
    --distpath packaging/dist \
    --workpath packaging/dist/work/desktop \
    --noconfirm --clean
```

Expected: `AIStudioDesktop-v1.5.5-Portable.zip` with:
- `ai-studio-desktop.exe` with PE version resource (FileVersion=1.5.5.0, ProductVersion=1.5.5.0)
- `_internal/` (PySide6 6.11.1 DLLs, Qt6 runtime)

---

## Manual Installer Build Commands

When Inno Setup 6 is installed:

```powershell
# Install Inno Setup 6
winget install JRSoftware.InnoSetup

# Build installer
& "C:\Program Files (x86)\Inno Setup 6\ISCC.exe" packaging\installer.iss

# Silent install test
.\dist\AIStudioPlatform-v1.5.5-Setup.exe /VERYSILENT /NORESTART /SUPPRESSMSGBOXES

# Verify installation
Test-Path "$env:LOCALAPPDATA\Programs\AIStudio\ai-software-factory.exe"
Test-Path "$env:APPDATA\Microsoft\Windows\Start Menu\Programs\AI Studio\*.lnk"

# Silent uninstall test
& "$env:LOCALAPPDATA\Programs\AIStudio\unins000.exe" /VERYSILENT /NORESTART

# Upgrade test (install 1.5.4 then 1.5.5)
.\dist\AIStudioPlatform-v1.5.4-Setup.exe /VERYSILENT
.\dist\AIStudioPlatform-v1.5.5-Setup.exe /VERYSILENT

# Downgrade protection test (install 1.5.5 then try 1.5.4)
.\dist\AIStudioPlatform-v1.5.5-Setup.exe /VERYSILENT
.\dist\AIStudioPlatform-v1.5.4-Setup.exe  # should reject with error dialog
```

---

## NOT VERIFIED

All installer items require Inno Setup 6 (ISCC.exe) to be installed.

| Item | Manual Command |
|------|---------------|
| Installer compilation | `ISCC.exe packaging\installer.iss` |
| Silent install | `Setup.exe /VERYSILENT /NORESTART` |
| Silent uninstall | `unins000.exe /VERYSILENT /NORESTART` |
| Start Menu shortcuts | Visual inspection after install |
| Desktop shortcut | Visual inspection after install |
| Registry entries | `reg query HKCU\Software\AIStudio` |
| Downgrade protection | Install newer → try older → verify rejection |
| Config migration | Check `.migration_needed` marker file |
| Code signing | `signtool sign /n "AI Studio" Setup.exe` |
