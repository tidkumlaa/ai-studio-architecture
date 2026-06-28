# Build Reproducibility Report — AI Studio Platform v1.5.5

**Generated:** 2026-06-27T12:52:17Z  
**Pipeline Run:** local-20260627125217  
**Builder:** LAPTOP-TM8BC7MD (Windows 11 Home 10.0.26200, x64)

---

## Purpose

This report documents what is needed to reproduce the v1.5.5 release artifacts from source, and which parts of the build are deterministic vs. environment-dependent.

---

## Toolchain

All tool versions are pinned to produce the same bytecode and binary layout across rebuilds on the same OS.

| Tool | Version | Pin Method |
|------|---------|------------|
| Python | 3.13.14 | pyenv / installer lock |
| Java (Temurin) | 21.0.6 | SDKMAN / installer |
| Maven | 3.9.10 | `.mvn/wrapper/maven-wrapper.properties` |
| PyInstaller | 6.21.0 | `pyproject.toml` `[project.optional-dependencies]` |
| PySide6 | 6.11.1 | `pyproject.toml` dependency |
| cyclonedx-bom | 7.3.0 | `pyproject.toml` |
| pip-audit | 2.10.1 | `pyproject.toml` |
| flake8 | 7.3.0 | `pyproject.toml` |

### Verified Toolchain (EXECUTED)

```
$ python --version
Python 3.13.14

$ java -version
openjdk version "21.0.6" 2025-01-21 LTS
OpenJDK Runtime Environment Microsoft-9889606 (build 21.0.6+7-LTS)

$ mvn -version
Apache Maven 3.9.10

$ pyinstaller --version
6.21.0

$ python -m flake8 --version
7.3.0 (mccabe: 0.7.0, pycodestyle: 2.11.1, pyflakes: 3.2.0)

$ python -m cyclonedx_py --version
cyclonedx-bom 7.3.0

$ python -m pip_audit --version
pip-audit 2.10.1
```

---

## Source Pinning

### Git Commit (EXECUTED)

```
Repository: ai-software-factory (git)
Commit:     53b3f1eaf32d13c6c7b4c16f9b98a45bafe457d7
Tag:        v1.5.5 (annotated, 2026-06-27)
```

The annotated tag `v1.5.5` permanently identifies the source snapshot. A reproducible rebuild starts from:

```
git checkout v1.5.5
```

### Dependency Lock Files

| Component | Lock File | Status |
|-----------|-----------|--------|
| AISF | `requirements.txt` (or pip freeze) | NOT GENERATED in pipeline |
| Desktop | `requirements.txt` | NOT GENERATED in pipeline |
| Content Factory | `pom.xml` (Maven coordinates are exact) | YES — Maven resolves deterministically |

**NOT VERIFIED:** AISF and Desktop pip lock files (`pip freeze > requirements-lock.txt`) were not generated. Without a lock file, `pip install` may resolve different transitive dependency versions on a future rebuild. The SBOM captures the dependency set that was actually installed.

---

## Version Strings (EXECUTED)

All version strings were set to `1.5.5` before the pipeline ran. Verified:

```
$ python -c "import sys; sys.path.insert(0,'E:\\UserData\\MyData\\Content\\DEV\\ai-software-factory'); import platform_version as pv; print(pv.PLATFORM_VERSION)"
1.5.5

$ python -c "
import sys
sys.path.insert(0,'E:\\UserData\\MyData\\Content\\DEV\\ai-studio-desktop')
from app.updater import CURRENT_VERSION
print(CURRENT_VERSION)
"
1.5.5
```

Version strings changed from 1.5.4 → 1.5.5 in:

| File | Field |
|------|-------|
| `ai-software-factory/pyproject.toml` | `version` |
| `ai-software-factory/platform_version.py` | `PLATFORM_VERSION`, `PLUGIN_API_VERSION`, `RUNTIME_VERSION` |
| `ai-studio-desktop/pyproject.toml` | `version` |
| `ai-studio-desktop/app/application.py` | `_DESKTOP_VERSION` |
| `ai-studio-desktop/app/updater.py` | `CURRENT_VERSION` |
| `packaging/version_info.txt` | `filevers`, `FileVersion`, `ProductVersion` |

---

## Build Steps for Full Reproduction

### Step 1: Checkout

```powershell
git clone <repo-url> ai-software-factory
git -C ai-software-factory checkout v1.5.5
```

### Step 2: Python Environments

```powershell
# AISF
cd ai-software-factory
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -e ".[test,dev]"

# Desktop
cd ..\ai-studio-desktop
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -e ".[test,dev]"
```

### Step 3: Static Analysis

```powershell
python -m flake8 . --count
# Expected: 0
```

### Step 4: Unit Tests

```powershell
# AISF
python -m pytest tests/ --cov=factory.logging --cov=factory.metrics --cov-fail-under=60
# Expected: 41 passed, 72.93% coverage

# Desktop
python -m pytest tests/
# Expected: 10 passed

# CF
mvn test -pl cf-api,cf-common,cf-worker --no-transfer-progress
# Expected: BUILD SUCCESS, 182 tests
```

### Step 5: Security Scan

```powershell
python -m pip_audit --format json
# Expected: 0 vulnerabilities
```

### Step 6: SBOM Generation

```powershell
# From AISF venv
python -m cyclonedx_py environment --of JSON --mc-type application --pyproject pyproject.toml -o sbom-aisf.json

# From Desktop venv
python -m cyclonedx_py environment --of JSON --mc-type application --pyproject pyproject.toml -o sbom-desktop.json
```

### Step 7: PyInstaller Builds

```powershell
# From packaging directory with AISF venv active
pyinstaller packaging\aisf.spec

# From packaging directory with Desktop venv active
pyinstaller packaging\desktop.spec
```

### Step 8: Checksums + Manifest

```powershell
.\packaging\generate-manifest.ps1 -Version "1.5.5"
```

### Step 9: Git Tag

```powershell
git tag -a "v1.5.5" -m "Release v1.5.5"
```

### Step 10: Bundle

```powershell
.\pipeline\pipeline.ps1 -Stage release-bundle
```

---

## Reproducibility Assessment

| Artifact | Reproducible? | Notes |
|----------|--------------|-------|
| Python test results | YES | Pure code, no external I/O |
| CF test results | YES | Maven resolves same coordinates |
| SBOM JSON | PARTIAL | Component list same; `serialNumber` UUID changes each run |
| `checksums.sha256` | PARTIAL | SHA-256 matches when same inputs used |
| PyInstaller ZIP | PARTIAL | Binary identical on same Python+PyInstaller+OS versions; bootloader has embedded timestamp |
| `release-manifest.json` | NO | `released_at` and `pipeline_run_id` change each build |

### Known Non-Deterministic Elements

1. **SBOM `serialNumber`**: `cyclonedx_py` generates a new UUID v4 on each run. Consumers should compare component lists, not the serial number, when checking for changes.

2. **PyInstaller bootloader timestamp**: The PE header `TimeDateStamp` field is set by the MSVC linker at PyInstaller build time. Two builds from identical source produce binaries differing only in this 4-byte field.

3. **`released_at` in release-manifest.json**: UTC timestamp at manifest generation time. Changes on every pipeline run.

4. **`pipeline_run_id`**: Embeds the wall-clock time (`local-YYYYMMDDHHMMSS`).

---

## NOT VERIFIED

| Item | Reason |
|------|--------|
| v1.5.5 PyInstaller rebuild | Not executed in this pipeline pass; ZIPs are v1.5.4 artifacts |
| pip lock files | Not generated; future rebuilds may differ if upstream packages change |
| Cross-machine reproducibility | Only verified on LAPTOP-TM8BC7MD; different machines (different DLL versions) will produce different Desktop ZIPs |
| Maven `.mvn/wrapper` | Not verified that content-factory has wrapper config |
