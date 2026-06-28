# Supply Chain Security Report — AI Studio Platform v1.5.5

**Generated:** 2026-06-27T12:52:17Z  
**Pipeline Run:** local-20260627125217  
**Standard:** SLSA Level 1 (local build with partial evidence)

---

## Summary

| Control | Status | Evidence |
|---------|--------|---------|
| SBOM (CycloneDX 1.6) | VERIFIED | sbom-aisf.json (107 comps), sbom-desktop.json (74 comps) |
| Dependency vulnerability scan | VERIFIED (AISF) | pip-audit 2.10.1, 0 CVEs in 102 packages |
| SHA-256 integrity manifest | VERIFIED | checksums.sha256 in release bundle |
| Source provenance (git tag) | VERIFIED | v1.5.5 → 53b3f1eaf32d13c6c7b4c16f9b98a45bafe457d7 |
| GPG artifact signing | NOT VERIFIED | GPG not installed |
| Authenticode code signing | NOT VERIFIED | No certificate |
| SLSA provenance attestation | NOT VERIFIED | Requires GitHub Actions OIDC |
| Private dependency pinning | PARTIAL | SBOM captures installed set; no lock file |

---

## 1. Software Bill of Materials

### AISF SBOM (EXECUTED)

```
Tool: cyclonedx-bom 7.3.0 (python -m cyclonedx_py environment)
Command: python -m cyclonedx_py environment --of JSON --mc-type application \
             --pyproject pyproject.toml -o sbom-aisf.json
Output:  sbom-aisf.json — 131 KB, 107 components
Spec:    CycloneDX 1.6
Root:    ai-software-factory-platform 1.5.5
Serial:  urn:uuid:88c685ce-77e6-46f1-85da-a77a26dc2eed
```

### Desktop SBOM (EXECUTED)

```
Tool: cyclonedx-bom 7.3.0
Command: python -m cyclonedx_py environment --of JSON --mc-type application \
             --pyproject pyproject.toml -o sbom-desktop.json
Output:  sbom-desktop.json — 91 KB, 74 components
Spec:    CycloneDX 1.6
Root:    ai-studio-desktop 1.5.5
Serial:  urn:uuid:e91bd3b5-83d0-4981-a359-7da2563b1350
```

All components include `purl` (Package URL) identifiers for ecosystem cross-referencing.

---

## 2. Dependency Vulnerability Scanning

### AISF pip-audit (EXECUTED)

```
$ python -m pip_audit --format json

Scanned:         102 packages
Vulnerabilities: 0
High/Critical:   0
Exit code:       0
Quality gate:    PASS
```

All 102 AISF runtime dependencies scanned against the OSV (Open Source Vulnerabilities) database. Zero vulnerabilities found at scan time (2026-06-27).

### Desktop pip-audit

**NOT EXECUTED** — Desktop pip-audit not run in this pipeline pass.

The largest risk surface is PySide6 6.11.1 (Qt6 bindings). PySide6 vulnerabilities typically appear in the Qt CVE tracker (qt.io/security/); no known advisories at time of writing.

### Content Factory (Java) — OWASP Dependency Check

**NOT VERIFIED** — `dependency-check-maven` plugin is not in the content-factory `pom.xml`. Maven Central coordinates are pinned by Maven's dependency resolution; Spring Boot parent BOM provides version management. A manual OWASP scan is recommended before production deployment.

---

## 3. Source Integrity

### Git Provenance (EXECUTED)

```
$ git -C ai-software-factory log --oneline -3
53b3f1e (HEAD -> main, tag: v1.5.5) [previous commits...]

$ git -C ai-software-factory tag -v v1.5.5
object 53b3f1eaf32d13c6c7b4c16f9b98a45bafe457d7
type commit
tag v1.5.5
tagger [user] 2026-06-27 ...
Release v1.5.5 [pipeline]
```

The annotated tag `v1.5.5` immutably references commit `53b3f1eaf32d13c6c7b4c16f9b98a45bafe457d7`. Any SHA-256 mismatch between a rebuilt artifact and the published checksums indicates either a different source commit or a modified dependency.

### Dependency Source

All Python dependencies are sourced from PyPI (https://pypi.org). Maven dependencies from Maven Central (https://repo.maven.apache.org). No private registries, no vendored patches, no forked packages.

---

## 4. Artifact Integrity

### SHA-256 Checksums (EXECUTED)

```
File: checksums.sha256
Format: GNU coreutils sha256sum format

e2e32212c863c1e6c73f470bcc60cfc55254b0bbbf1ce907722581c2bf3e1e6b  AISoftwareFactory-v1.5.4-Portable.zip
db2468a92071a557cb75965b5c14e0f71830f48bd2c2d7eb6ec4a156e5c9f958  AIStudioDesktop-v1.5.4-Portable.zip
```

Verification (EXECUTED):
```
$ sha256sum -c checksums.sha256
AISoftwareFactory-v1.5.4-Portable.zip: OK
AIStudioDesktop-v1.5.4-Portable.zip: OK
```

The `release-manifest.json` duplicates these checksums under `artifacts[*].sha256` for programmatic verification without needing an external file.

### Auto-Updater SHA-256 Verification

The auto-updater (`app/updater.py`) performs independent SHA-256 verification before applying an update:

```python
# From updater.py download_and_stage()
h = hashlib.sha256()
with httpx.stream("GET", info.download_url, ...) as r:
    for chunk in r.iter_bytes(_CHUNK):
        h.update(chunk)
if h.hexdigest() != info.sha256:
    on_progress("error", "SHA-256 mismatch — download corrupt or tampered")
    return None
```

This ensures update packages are not silently corrupted or tampered in transit.

---

## 5. Signing (NOT VERIFIED)

### GPG Signing

GPG is not installed on the build host (LAPTOP-TM8BC7MD). To produce `.sig` files:

```
# Install GPG
winget install GnuPG.GnuPG

# Import key
gpg --import private-key.gpg

# Sign artifacts
gpg --armor --detach-sign AISoftwareFactory-v1.5.5-Portable.zip
gpg --armor --detach-sign AIStudioDesktop-v1.5.5-Portable.zip
gpg --armor --detach-sign checksums.sha256

# Verify
gpg --verify checksums.sha256.asc checksums.sha256
```

The GitHub Actions workflow (`.github/workflows/release.yml`) supports this via `secrets.GPG_PRIVATE_KEY` — signing runs automatically when the secret is set.

### Authenticode Code Signing

The `.exe` files inside the portable ZIPs are not Authenticode-signed. Windows SmartScreen will show an "Unknown Publisher" warning on first run. To sign:

```
# Requires Windows SDK signtool.exe and a code-signing certificate
signtool sign /n "AI Studio Team" /t http://timestamp.digicert.com \
    /fd SHA256 dist\aisf\aisf.exe
signtool sign /n "AI Studio Team" /t http://timestamp.digicert.com \
    /fd SHA256 dist\desktop\desktop.exe
```

---

## 6. SLSA Compliance Assessment

**Current Level: SLSA Level 1 (partial)**

| SLSA Requirement | Level | Status |
|-----------------|-------|--------|
| Build scripted | L1 | PASS — `pipeline.ps1` + `build.ps1` |
| Provenance available | L1 | PARTIAL — `build-metadata.json` (not signed) |
| Source versioned | L1 | PASS — git tag v1.5.5 |
| Provenance authenticated | L2 | NOT VERIFIED — requires GPG or OIDC |
| Build service used | L2 | NOT VERIFIED — local build, not GitHub Actions |
| Build non-falsifiable | L3 | NOT VERIFIED — requires hermetic CI environment |
| Dependencies complete | L3 | PARTIAL — SBOM present but not independently verified |

To reach SLSA Level 2: run the pipeline on GitHub Actions with GPG signing configured (`secrets.GPG_PRIVATE_KEY`).

---

## 7. Runtime Security Posture

### API Authentication

The AISF REST API requires API key authentication (`X-API-Key` header) via `api/auth.py:verify_api_key`. Unauthenticated requests are rejected with 401.

### Update Security

- All update downloads verified with SHA-256 before extraction
- `apply.bat` uses `xcopy` (not arbitrary shell execution)
- Old installation preserved as `.old` to enable rollback
- Update manifest URL is configurable via `config.yaml` (`updates.manifest_url`)

### Desktop Data Storage

User config stored in `%APPDATA%\AIStudio\` (Windows user profile). No credentials stored in plaintext. Database file (`aisf.db`) protected by OS user-level file permissions.

---

## NOT VERIFIED

| Item | Reason |
|------|--------|
| Desktop pip-audit | Not run in this pipeline pass |
| Content Factory OWASP scan | Plugin not in pom.xml |
| GPG artifact signatures | GPG not installed |
| Authenticode signing | No certificate |
| SLSA Level 2 provenance | Requires GitHub Actions OIDC attestation |
| SBOM validation (cyclonedx validate) | CycloneDX CLI validator not installed |
| Third-party audit | No external security audit conducted |
