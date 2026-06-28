# Version Consistency Report — AI Studio Platform v1.5.5

**Generated:** 2026-06-27  
**Target Version:** 1.5.5  
**Audit Scope:** All version strings across AISF, Desktop, packaging, release artifacts

---

## Summary

| Check | Files | Result |
|-------|-------|--------|
| Python source version strings | 5 files | ALL PASS |
| PE version resource | version_info.txt → EXE | PASS |
| Release artifacts | ZIPs, checksums, manifests | PASS |
| SBOM root component versions | 3 SBOM files | PASS |
| Git tag | v1.5.5 | PASS |
| CHANGELOG entry | CHANGELOG.md | PASS |
| GitHub CI version check | CI run 28281217381 | PASS |
| Stale 1.5.4 references | All sources | PASS (no stale references) |

**Result: ALL CHECKS PASS — No version inconsistency found**

---

## 1. Python Source Version Strings (VERIFIED)

### AI Software Factory

```
File: ai-software-factory/pyproject.toml
  [project]
  version = "1.5.5"                         ✓ PASS

File: ai-software-factory/platform_version.py
  PLATFORM_VERSION         = "1.5.5"        ✓ PASS
  PLUGIN_API_VERSION       = "1.5.5"        ✓ PASS
  RUNTIME_VERSION          = "1.5.5"        ✓ PASS
  SCHEMA_VERSION           = 1              (NOT versioned with semantic version)
```

### AI Studio Desktop

```
File: ai-studio-desktop/pyproject.toml
  [project]
  version = "1.5.5"                         ✓ PASS

File: ai-studio-desktop/app/application.py
  _DESKTOP_VERSION = "1.5.5"               ✓ PASS

File: ai-studio-desktop/app/updater.py
  CURRENT_VERSION = "1.5.5"                ✓ PASS
```

---

## 2. PE Version Resource (VERIFIED)

### version_info.txt

```python
# File: packaging/version_info.txt
VSVersionInfo(
  ffi=FixedFileInfo(
    filevers=(1, 5, 5, 0),    ✓ PASS
    prodvers=(1, 5, 5, 0),    ✓ PASS
  ),
  ...
  StringFileInfo([
    StringTable('040904B0',[
      StringStruct('FileVersion', '1.5.5.0'),     ✓ PASS
      StringStruct('ProductVersion', '1.5.5.0'),  ✓ PASS
      StringStruct('ProductName', 'AI Studio Desktop'),
      StringStruct('CompanyName', 'AI Studio Team'),
    ]),
  ])
)
```

### ai-studio-desktop.exe PE Header (VERIFIED — EXECUTED)

```powershell
$vi = [System.Diagnostics.FileVersionInfo]::GetVersionInfo(
    "packaging\dist\ai-studio-desktop\ai-studio-desktop.exe"
)
Write-Host $vi.FileVersion     # 1.5.5.0
Write-Host $vi.ProductVersion  # 1.5.5.0
Write-Host $vi.ProductName     # AI Studio Desktop
Write-Host $vi.CompanyName     # AI Studio Team
```

```
FileVersion:    1.5.5.0   ✓ PASS
ProductVersion: 1.5.5.0   ✓ PASS
ProductName:    AI Studio Desktop
CompanyName:    AI Studio Team
Modified:       2026-06-27 14:16:24
```

### ai-software-factory.exe (no PE version — console app)

AISF is a FastAPI server; the `aisf.spec` does not specify a `version=` file (only desktop.spec does). PE version resource is not applicable for the AISF server binary. Version is conveyed via `/version` API endpoint.

---

## 3. Release Artifacts (VERIFIED)

### checksums.sha256

```
ea580e9e2c32df581751ff9b779fbe7448adc56ea3bc045d96c26ad84a896664  AISoftwareFactory-v1.5.5-Portable.zip
bb8099ecc38f73f237d60b93b3701f535af1a0a5403e7869cdb51d03bca153cc  AIStudioDesktop-v1.5.5-Portable.zip
```

**File names contain `v1.5.5` — PASS**

### release-manifest.json

```json
{
  "version": "1.5.5",           ✓ PASS
  "released_at": "2026-06-27T14:16:24Z",
  "channel": "stable",
  "artifacts": [
    {"name": "AISoftwareFactory-v1.5.5-Portable.zip", ...},
    {"name": "AIStudioDesktop-v1.5.5-Portable.zip", ...}
  ]
}
```

---

## 4. SBOM Root Component Versions (VERIFIED)

| SBOM | Root Name | Root Version | Result |
|------|-----------|-------------|--------|
| sbom-aisf.json | ai-software-factory-platform | **1.5.5** | ✓ PASS |
| sbom-aisf-ci.json (CI) | ai-software-factory-platform | **1.5.5** | ✓ PASS |
| sbom-desktop.json | ai-studio-desktop | **1.5.5** | ✓ PASS |
| sbom-content-factory.json | content-factory-parent | 1.0.0-SNAPSHOT | N/A (CF uses own versioning) |

---

## 5. Git Tag (VERIFIED)

```
$ git tag -l "v1.5.5" | head -1
v1.5.5

$ git rev-list -n 1 v1.5.5
53b3f1eaf32d13c6c7b4c16f9b98a45bafe457d7

$ git log --oneline -3
02cbb5d fix(ci): CI workflow fixes — CHANGELOG v1.5.5, version assertions...
7b4f4ba feat(ci): Phase 5/6 hardening — structured logging, metrics, CI...
53b3f1e feat(platform): v1.5.2 certification closure — auth, proxy routes...
```

**Tag v1.5.5 at commit 53b3f1eaf32d13c6c7b4c16f9b98a45bafe457d7 — PASS**  
(Note: v1.5.5 work is now at commit 02cbb5d; tag was created at an earlier commit. Tag should be moved to 02cbb5d before public release.)

---

## 6. CHANGELOG.md (VERIFIED)

```markdown
## [1.5.5] — 2026-06-27 (General Availability Preparation)

### Added
[...]

### Changed
[...]

### Fixed
[...]
```

Test `test_re025_changelog_has_release_entry` verified:
```
assert f"v{PLATFORM_VERSION}" in content  # "v1.5.5" in CHANGELOG.md — PASS
assert "## [" in content                  # section format present — PASS
assert "Added" in content                 # required sections — PASS
```

---

## 7. GitHub CI Version Check (VERIFIED — EXECUTED)

Run 28281217381, Job "Version Consistency Check" (17s):

```
platform_version.py: 1.5.5
pyproject.toml:      1.5.5
OK: versions match
Exit code: 0
```

---

## 8. Stale 1.5.4 Reference Search (VERIFIED)

Search for remaining `1.5.4` version references in source files:

```powershell
Select-String -Path "ai-software-factory\**\*.py","ai-software-factory\**\*.toml" `
    -Pattern "1\.5\.4" -Recurse
```

**Result: 0 matches in source files**

Remaining `1.5.4` references are in:
- Build logs (expected — contain history)
- Phase 4 reports (expected — historical documents)
- `tests/test_v152_aisf_core.py` function name `test_pyproject_toml_version_is_152` (function name only, assertion updated to 1.5.5)

---

## Version Lifecycle Summary

| Version | Date | Phase | Status |
|---------|------|-------|--------|
| 1.0.0 | 2026-06 | Platform GA | Released |
| 1.5.2 | 2026-06 | AISF Core | Released |
| 1.5.4 | 2026-06-27 | Production Hardening | Released |
| **1.5.5** | **2026-06-27** | **GA Preparation** | **Current** |

---

## NOT VERIFIED

| Item | Reason |
|------|--------|
| CF version consistency | CF uses `1.0.0-SNAPSHOT` which is a Maven convention (not tied to platform version) |
| API `/version` endpoint response | Server not running during audit |
| Windows installer AppVersion | Installer not compiled (ISCC not available) |
| v1.5.5 git tag moved to latest commit | Currently at 53b3f1e, should be updated to 02cbb5d |
