# Clean Machine Validation Report — AI Studio Platform v1.5.5

**Generated:** 2026-06-27  
**Status: NOT VERIFIED — No isolated VM available on build host**

---

## Summary

Clean machine validation requires a fresh Windows 10/11 environment with no pre-installed Python, Java, or AI Studio dependencies. This environment is not available on the current build host (LAPTOP-TM8BC7MD).

| Validation Item | Status | Notes |
|----------------|--------|-------|
| Fresh Windows 11 VM | NOT VERIFIED | No VM infrastructure available |
| First launch (AISF portable) | NOT VERIFIED | Requires clean environment |
| First launch (Desktop portable) | NOT VERIFIED | Requires clean environment |
| Auto-update check | NOT VERIFIED | Requires network + update server |
| Shutdown/restart | NOT VERIFIED | Requires live deployment |
| Recovery after crash | NOT VERIFIED | Requires fault injection |
| No pre-installed dependencies | NOT VERIFIED | Requires clean OS image |

---

## What Was Partially Validated (on Developer Machine)

The developer machine (LAPTOP-TM8BC7MD) has Python, Java, and various development tools installed. The following was validated on this machine:

### Portable ZIP Extraction + Structure (VERIFIED)

```powershell
# AISF
Expand-Archive AISoftwareFactory-v1.5.4-Portable.zip -DestinationPath C:\test-aisf
Test-Path C:\test-aisf\ai-software-factory\ai-software-factory.exe  # True
Test-Path C:\test-aisf\ai-software-factory\_internal                 # True

# Desktop  
Expand-Archive AIStudioDesktop-v1.5.4-Portable.zip -DestinationPath C:\test-desktop
Test-Path C:\test-desktop\ai-studio-desktop\ai-studio-desktop.exe   # True
Test-Path C:\test-desktop\ai-studio-desktop\_internal               # True
```

### EXE PE32+ Header Validation (VERIFIED)

```powershell
# Read PE header
$bytes = [System.IO.File]::ReadAllBytes("ai-software-factory\ai-software-factory.exe")
$pe_offset = [BitConverter]::ToInt32($bytes, 0x3c)
$machine = [BitConverter]::ToUInt16($bytes, $pe_offset + 4)
Write-Host "Machine type: 0x$($machine.ToString('X4'))"  # 0x8664 = AMD64 (PE32+)
```

Both EXEs are PE32+ (x64), Windows-compatible.

### validate-install.ps1 (VERIFIED)

```
PASS: 17 | FAIL: 0 | SKIP: 5
RESULT: PASS
```

---

## Clean Machine Validation Protocol

For future GA sign-off, use this protocol:

### Environment Setup

1. Fresh Windows 11 Home/Pro 22H2+ (clean install, no updates beyond critical security)
2. Standard user account (no admin rights — tests `PrivilegesRequired=lowest`)
3. No Python, Java, Node, or development tools installed
4. Microsoft Defender active (tests SmartScreen behavior with unsigned EXEs)

### Validation Steps

#### Step 1: AISF Portable

```powershell
# Extract
Expand-Archive AISoftwareFactory-v1.5.5-Portable.zip -DestinationPath "C:\AIStudioTest\aisf"

# Start server
cd "C:\AIStudioTest\aisf\ai-software-factory"
.\ai-software-factory.exe

# Expected: FastAPI starts, logs to console
# Verify: http://127.0.0.1:8765/health returns 200

# Health check
(Invoke-WebRequest http://127.0.0.1:8765/health).StatusCode  # 200
(Invoke-WebRequest http://127.0.0.1:8765/version).Content    # {"platform":"1.5.5",...}
```

#### Step 2: Desktop Portable

```powershell
# Extract
Expand-Archive AIStudioDesktop-v1.5.5-Portable.zip -DestinationPath "C:\AIStudioTest\desktop"

# Launch
cd "C:\AIStudioTest\desktop\ai-studio-desktop"
.\ai-studio-desktop.exe

# Expected:
# - AI Studio Desktop window opens
# - Runtime Center panel shows services
# - Desktop connects to AISF (if running)
```

#### Step 3: Auto-Update Check

```powershell
# Configure update manifest URL in config.yaml
# updates.manifest_url = https://releases.ai-studio.local/update-manifest.json

# Launch Desktop → Help → Check for Updates
# Expected: Either "up to date" or update dialog
```

#### Step 4: Shutdown + Restart

```powershell
# Stop AISF
Stop-Process -Name "ai-software-factory" -Force

# Restart — data should persist
.\ai-software-factory.exe

# Verify DB state preserved
(Invoke-WebRequest http://127.0.0.1:8765/tasks).Content  # same tasks as before
```

#### Step 5: Crash Recovery

```powershell
# Kill AISF mid-operation
Stop-Process -Name "ai-software-factory" -Force -Confirm:$false

# Restart
.\ai-software-factory.exe

# Verify: starts cleanly, DB not corrupted
```

---

## Screenshots/Evidence Collection

When running on a clean machine, collect:

1. `screenshot_first_launch_desktop.png` — Desktop UI on first launch
2. `screenshot_aisf_health.png` — Browser showing `/health` response
3. `screenshot_update_check.png` — Update check dialog
4. `screenshot_runtime_center.png` — Runtime Center panel
5. `screenshot_smartscreen.png` — SmartScreen warning (if unsigned)
6. `aisf.log` — First 100 lines of startup log
7. `validate-install.ps1` output

---

## NOT VERIFIED — Complete List

| Item | Justification |
|------|---------------|
| Fresh VM | No VM hypervisor (Hyper-V/VMware) configured on build host |
| First launch | Developer machine has pre-installed dependencies that mask startup issues |
| Auto-update test | No update server deployed at configured URL |
| SmartScreen behavior | Developer machine trusts local EXEs (bypassed) |
| Crash recovery | Not tested under controlled conditions |
| Multi-user scenario | Not tested (single user on developer machine) |
| Disk full scenario | Not tested |
| Low RAM scenario | Not tested |

---

## Recommendation for GA

Before public GA release:

1. **Provision a Windows 11 VM** (Azure, AWS, or local Hyper-V)
2. **Run the clean machine validation protocol above**
3. **Capture all screenshots and logs**
4. **Verify SmartScreen behavior** — if EXE is unsigned, document the user experience
5. **Test auto-update flow** end-to-end with a real v1.5.4 → v1.5.5 update

Estimated time: 2 hours on a clean Windows 11 VM.
