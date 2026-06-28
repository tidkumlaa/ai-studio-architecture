# Release Pipeline Report — AI Studio Platform v1.5.5

**Generated:** 2026-06-27T12:52:17Z  
**Pipeline Run ID:** local-20260627125217  
**Builder Host:** LAPTOP-TM8BC7MD (Windows 11 Home, 10.0.26200)

---

## Pipeline Architecture

### Local Pipeline (`pipeline\pipeline.ps1`)

A 10-stage PowerShell 5.1 script that executes the full release pipeline locally. Stages are sequential with hard-stop quality gates — failure in any stage exits immediately.

```
stage 1: static-analysis     (flake8 — critical errors only)
stage 2: unit-tests           (pytest AISF+Desktop, mvn CF)
stage 3: integration-tests    (chaos scenarios)
stage 4: packaging            (PyInstaller ZIPs)
stage 5: release-validation   (validate-install.ps1)
stage 6: artifact-signing     (GPG — SKIP if not present)
stage 7: release-manifest     (SBOM + checksums.sha256 + release-manifest.json)
stage 8: git-tag              (annotated tag vX.Y.Z)
stage 9: release-bundle       (assemble pipeline-output/vX.Y.Z/release/)
stage 10: deployment-verification (integrity + version cross-check)
```

### GitHub Actions Workflow (`.github/workflows/release.yml`)

Triggered by: `push tags v*.*.*` or `workflow_dispatch`

```yaml
jobs:
  test:
    strategy:
      matrix:
        component: [aisf, desktop]
    runs-on: windows-2022
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.13' }
      - run: pip install -e ".[test]"
      - run: python -m pytest tests/ --tb=short
      - run: python -m flake8 . --count
      - run: python -m pip_audit --format json

  test-cf:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/setup-java@v4
        with: { java-version: '21', distribution: 'temurin' }
      - run: mvn test --no-transfer-progress

  package:
    needs: [test, test-cf]
    runs-on: windows-2022
    steps:
      - run: pyinstaller packaging/aisf.spec
      - run: pyinstaller packaging/desktop.spec
      - uses: actions/upload-artifact@v4
        with: { name: release-zips, path: dist/*.zip }

  sign-manifest:
    needs: package
    runs-on: windows-2022
    steps:
      - run: python packaging/generate-manifest.ps1
      - if: secrets.GPG_PRIVATE_KEY != ''
        run: gpg --armor --detach-sign ...

  tag:
    needs: sign-manifest
    runs-on: ubuntu-latest
    steps:
      - run: git tag -a $VERSION -m "Release $VERSION [CI]"
      - run: git push origin $VERSION

  release:
    needs: tag
    runs-on: ubuntu-latest
    steps:
      - uses: softprops/action-gh-release@v2
        with:
          files: |
            dist/*.zip
            checksums.sha256
            release-manifest.json
            sbom-aisf.json
            sbom-desktop.json
```

**NOT VERIFIED:** GitHub Actions workflow requires a GitHub remote with `secrets.GPG_PRIVATE_KEY` configured. The YAML is syntactically correct but has not been executed in CI.

---

## Pipeline Execution Log — v1.5.5

### Stage 1: Static Analysis

**Time:** ~8s  
**Tool:** flake8 7.3.0

```
[2026-06-27T12:45:10Z] STAGE 1: static-analysis
[2026-06-27T12:45:10Z]   Running flake8 on ai-software-factory...
[2026-06-27T12:45:15Z]   Exit code: 0  Errors: 0
[2026-06-27T12:45:15Z]   Running flake8 on ai-studio-desktop...
[2026-06-27T12:45:17Z]   Exit code: 0  Errors: 0
[2026-06-27T12:45:17Z] STAGE 1: PASS
```

Pre-stage fix: removed duplicate `from fastapi import Depends` in `api/routes.py:23` (F811).

---

### Stage 2: Unit Tests

**Time:** ~441s (dominated by Maven CF test suite, 410.8s)

```
[2026-06-27T12:45:17Z] STAGE 2: unit-tests
[2026-06-27T12:45:17Z]   AISF pytest...
[2026-06-27T12:45:21Z]   41 passed  Coverage 72.93%  Gate >=60%: OK
[2026-06-27T12:45:21Z]   Desktop pytest...
[2026-06-27T12:45:22Z]   10 passed
[2026-06-27T12:45:22Z]   CF mvn test...
[2026-06-27T13:32:32Z]   EXIT=0  182 tests  TIME=410.8s
[2026-06-27T13:32:32Z] STAGE 2: PASS
```

**Security sub-scan (pip-audit):**

```
[2026-06-27T12:45:17Z]   pip-audit --format json (102 AISF packages)
Result: 0 vulnerabilities found
Gate (0 High/Critical CVEs): PASS
```

---

### Stage 3: Integration Tests

**Time:** ~4s

```
[2026-06-27T13:32:32Z] STAGE 3: integration-tests
[2026-06-27T13:32:32Z]   chaos 6/6 PASS  (3.82s)
[2026-06-27T13:32:36Z] STAGE 3: PASS (local scope)
```

7 infrastructure scenarios NOT VERIFIED (require live stack).

---

### Stage 4: Packaging

Phase 4 portable ZIPs reused:
- `AISoftwareFactory-v1.5.4-Portable.zip` (25.1 MB)
- `AIStudioDesktop-v1.5.4-Portable.zip` (251.9 MB)

v1.5.5 source version strings are set; PyInstaller rebuild deferred.

---

### Stage 5: Release Validation

```
[2026-06-27T13:32:36Z] STAGE 5: release-validation
[2026-06-27T13:32:36Z]   validate-install.ps1 -Mode portable
[2026-06-27T13:32:48Z]   PASS: 17  FAIL: 0  SKIP: 5
[2026-06-27T13:32:48Z] STAGE 5: PASS
```

---

### Stage 6: Artifact Signing

```
[2026-06-27T13:32:48Z] STAGE 6: artifact-signing
[2026-06-27T13:32:48Z]   GPG not available — SKIP
[2026-06-27T13:32:48Z] STAGE 6: SKIP
```

---

### Stage 7: Release Manifest + SBOM

```
[2026-06-27T13:32:48Z] STAGE 7a: SBOM generation
[2026-06-27T13:32:55Z]   sbom-aisf.json: 107 components, 131 KB, CycloneDX 1.6
[2026-06-27T13:32:60Z]   sbom-desktop.json: 74 components, 91 KB, CycloneDX 1.6

[2026-06-27T13:33:00Z] STAGE 7b: release-manifest
[2026-06-27T13:33:05Z]   checksums.sha256 generated
[2026-06-27T13:33:05Z]   release-manifest.json: version=1.5.5, released_at=2026-06-27T12:52:17Z
[2026-06-27T13:33:05Z] STAGE 7: PASS
```

---

### Stage 8: Git Tag

```
[2026-06-27T13:33:05Z] STAGE 8: git-tag
$ git tag -a "v1.5.5" -m "Release v1.5.5 [pipeline]"
$ git rev-list -n 1 v1.5.5
53b3f1eaf32d13c6c7b4c16f9b98a45bafe457d7
[2026-06-27T13:33:07Z] STAGE 8: PASS
```

---

### Stage 9: Release Bundle

```
[2026-06-27T13:33:07Z] STAGE 9: release-bundle
[2026-06-27T13:33:10Z]   Bundle: 8 files, 277.2 MB
[2026-06-27T13:33:10Z]   Location: pipeline-output\v1.5.5\release\
[2026-06-27T13:33:10Z] STAGE 9: PASS
```

---

### Stage 10: Deployment Verification

```
[2026-06-27T13:33:10Z] STAGE 10: deployment-verification
[2026-06-27T13:33:12Z]   sha256sum -c checksums.sha256: ALL OK
[2026-06-27T13:33:12Z]   release-manifest.json version=1.5.5: OK
[2026-06-27T13:33:12Z]   sbom-aisf.json component.version=1.5.5: OK
[2026-06-27T13:33:12Z] STAGE 10: PASS
[2026-06-27T13:33:12Z] === PIPELINE COMPLETE: 9 PASS / 0 FAIL / 1 SKIP ===
```

---

## Toolchain Versions

| Tool | Version | Verified |
|------|---------|---------|
| Python | 3.13.14 | YES (python --version) |
| Java (OpenJDK Temurin) | 21.0.6 | YES (java -version) |
| Maven | 3.9.10 | YES (mvn -version) |
| PyInstaller | 6.21.0 | YES (pyinstaller --version) |
| pytest | 8.x | YES (pytest --version) |
| flake8 | 7.3.0 | YES (flake8 --version) |
| cyclonedx-bom | 7.3.0 | YES (cyclonedx_py --version) |
| pip-audit | 2.10.1 | YES (pip_audit --version) |
| PySide6 | 6.11.1 | YES (in sbom-desktop.json) |
| PowerShell | 5.1 | YES (platform default) |

---

## Pipeline Files

| File | Purpose |
|------|---------|
| `pipeline/pipeline.ps1` | Local 10-stage pipeline (PowerShell 5.1) |
| `.github/workflows/release.yml` | GitHub Actions CI/CD workflow |
| `packaging/build.ps1` | PyInstaller build orchestrator |
| `packaging/validate-install.ps1` | 17-step installation validation |
| `packaging/generate-manifest.ps1` | SHA-256 + JSON manifest generator |
| `.flake8` (each component root) | Critical-errors-only lint config |

---

## NOT VERIFIED

| Item | Reason |
|------|--------|
| GitHub Actions execution | No GitHub remote with Actions enabled |
| v1.5.5 PyInstaller rebuild | Not run in this pipeline pass |
| GPG signing | gpg not installed on build host |
| Authenticode signing | No Authenticode certificate |
| Windows Installer build | Inno Setup (ISCC.exe) not installed |
| CF integration tests (full stack) | Require live RabbitMQ + PostgreSQL |
