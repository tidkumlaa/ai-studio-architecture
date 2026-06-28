# Release Checklist — AI Studio Platform v1.5.5

**Release Date:** 2026-06-27  
**Pipeline Run:** local-20260627125217  
**Git Tag:** v1.5.5 → commit 53b3f1eaf32d13c6c7b4c16f9b98a45bafe457d7

Status key: `[x]` = VERIFIED (executed) | `[ ]` = NOT VERIFIED | `[-]` = SKIP

---

## Pre-Release

### Version Bump

- [x] `ai-software-factory/pyproject.toml` → `version = "1.5.5"`
- [x] `ai-software-factory/platform_version.py` → `PLATFORM_VERSION = "1.5.5"`
- [x] `ai-software-factory/platform_version.py` → `PLUGIN_API_VERSION = "1.5.5"`
- [x] `ai-software-factory/platform_version.py` → `RUNTIME_VERSION = "1.5.5"`
- [x] `ai-studio-desktop/pyproject.toml` → `version = "1.5.5"`
- [x] `ai-studio-desktop/app/application.py` → `_DESKTOP_VERSION = "1.5.5"`
- [x] `ai-studio-desktop/app/updater.py` → `CURRENT_VERSION = "1.5.5"`
- [x] `packaging/version_info.txt` → `filevers=(1,5,5,0)` / `FileVersion='1.5.5.0'`
- [ ] CHANGELOG.md updated with v1.5.5 release notes
- [ ] Content Factory pom.xml version bumped (if applicable)

### Static Analysis

- [x] AISF `flake8 . --count` = 0 critical errors
- [x] Desktop `flake8 . --count` = 0 critical errors
- [x] AISF F811 duplicate import fixed (`api/routes.py:23`)

---

## Testing

### Unit Tests

- [x] AISF pytest: **41/41 PASS** (3.47s)
- [x] AISF coverage: **72.93%** on factory.logging + factory.metrics (gate: ≥60%)
- [x] `test_fac_011_platform_version` asserts PLATFORM_VERSION == "1.5.5" — PASS
- [x] Desktop pytest: **10/10 PASS** (0.63s)
- [x] PKG-001–PKG-010 (updater) all pass
- [x] CF mvn test: **182/182 PASS** (410.8s, exit 0)

### Security Scan

- [x] AISF pip-audit: **0 CVEs** in 102 packages
- [ ] Desktop pip-audit (NOT RUN)
- [ ] Content Factory OWASP Dependency Check (NOT RUN)

### Integration Tests

- [x] Chaos scenarios: **6/6 PASS** (local, in-memory SQLite)
- [ ] Infrastructure integration (RabbitMQ, PostgreSQL, network partition) — NOT VERIFIED
- [ ] End-to-end Desktop → AISF → CF workflow — NOT VERIFIED

---

## Build & Packaging

### Packaging

- [x] AISF Portable ZIP built: `AISoftwareFactory-v1.5.4-Portable.zip` (25.1 MB)
- [x] Desktop Portable ZIP built: `AIStudioDesktop-v1.5.4-Portable.zip` (251.9 MB)
- [ ] v1.5.5 PyInstaller rebuild (AISF) — NOT RUN
- [ ] v1.5.5 PyInstaller rebuild (Desktop) — NOT RUN
- [-] Windows Installer build (Inno Setup not installed — SKIP)

### Validation

- [x] `validate-install.ps1 -Mode portable`: **17/17 PASS, 0 FAIL, 5 SKIP**
- [-] Silent install test (`/VERYSILENT`) — SKIP (installer not built)
- [-] Start Menu shortcut verification — SKIP
- [-] Desktop icon verification — SKIP
- [-] Downgrade protection test — SKIP

---

## Release Artifacts

### SBOM

- [x] `sbom-aisf.json` generated (CycloneDX 1.6, 107 components, 131 KB)
- [x] `sbom-desktop.json` generated (CycloneDX 1.6, 74 components, 91 KB)
- [x] SBOM metadata.component.version = "1.5.5" for both
- [ ] Content Factory SBOM (cyclonedx-maven-plugin) — NOT CONFIGURED
- [ ] SBOM schema validation (`cyclonedx validate`) — NOT RUN

### Checksums & Manifest

- [x] `checksums.sha256` generated (SHA-256 for both ZIPs)
- [x] `checksums.sha256` verified (`sha256sum -c` exit 0)
- [x] `release-manifest.json` generated (version=1.5.5, released_at=2026-06-27T12:52:17Z)
- [x] `update-manifest.json` generated
- [x] `build-metadata.json` generated

### Signing

- [-] GPG detached signatures (`.asc`) — SKIP (GPG not installed)
- [-] Authenticode code signing — SKIP (no certificate)

---

## Release Bundle

- [x] Bundle assembled at `pipeline-output\v1.5.5\release\`
- [x] Bundle: 8 files, 277.2 MB
  - [x] `AISoftwareFactory-v1.5.4-Portable.zip` (25.1 MB)
  - [x] `AIStudioDesktop-v1.5.4-Portable.zip` (251.9 MB)
  - [x] `checksums.sha256`
  - [x] `release-manifest.json`
  - [x] `update-manifest.json`
  - [x] `sbom-aisf.json`
  - [x] `sbom-desktop.json`
  - [x] `build-metadata.json`

---

## Source Control

- [x] Git tag `v1.5.5` created (annotated)
- [x] Tag points to: `53b3f1eaf32d13c6c7b4c16f9b98a45bafe457d7`
- [ ] Tag pushed to remote (`git push origin v1.5.5`) — NOT PUSHED (no remote configured)
- [ ] GitHub Release created — NOT CREATED

---

## Post-Release

### Distribution

- [ ] GitHub Release created with bundle artifacts attached
- [ ] `update-manifest.json` deployed to `https://releases.ai-studio.local/v1.5.5/`
- [ ] Update channel tested (Desktop auto-update check → detects v1.5.5)
- [ ] CHANGELOG.md published

### Verification on Clean Machine

- [ ] Fresh Windows 10/11 install
- [ ] Extract `AIStudioDesktop-v1.5.5-Portable.zip` to `C:\AIStudio`
- [ ] Run `desktop\desktop.exe`
- [ ] Verify launch without errors
- [ ] Start AISF server; Desktop → AISF health check returns 200
- [ ] Auto-update check hits configured manifest URL

---

## Known Gaps (action items for v1.5.6)

| Gap | Priority | Action |
|-----|----------|--------|
| v1.5.5 PyInstaller rebuild | HIGH | Run `pyinstaller packaging\aisf.spec` + `desktop.spec` with updated version_info.txt |
| pip lock files | HIGH | `pip freeze > requirements-lock.txt` for AISF and Desktop venvs |
| Desktop pip-audit | HIGH | Run `python -m pip_audit` from Desktop venv |
| CF OWASP scan | MEDIUM | Add `dependency-check-maven` to content-factory pom.xml |
| Content Factory SBOM | MEDIUM | Add `cyclonedx-maven-plugin` to pom.xml |
| GPG artifact signing | MEDIUM | Install GPG; add private key to GitHub Actions secret |
| GitHub Actions execution | MEDIUM | Push to GitHub remote with Actions enabled |
| Windows Installer | LOW | Install Inno Setup 6; run `packaging\build.ps1` |
| SLSA Level 2 | LOW | Run pipeline on GitHub Actions with OIDC provenance |

---

## Sign-Off

| Role | Name | Date | Signed |
|------|------|------|--------|
| Build Engineer | (auto — pipeline) | 2026-06-27 | pipeline.ps1 |
| QA Lead | | | |
| Release Manager | | | |
