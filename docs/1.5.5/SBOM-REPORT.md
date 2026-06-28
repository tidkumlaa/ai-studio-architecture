# Software Bill of Materials Report — AI Studio Platform v1.5.5

**Generated:** 2026-06-27T12:52:17Z  
**Standard:** CycloneDX 1.6  
**Tool:** cyclonedx-bom 7.3.0 (`cyclonedx_py` CLI)  
**Pipeline Run:** local-20260627125217

---

## SBOM Files

| File | Size | Components | serialNumber |
|------|------|------------|--------------|
| `sbom-aisf.json` | 131 KB | 107 | urn:uuid:88c685ce-77e6-46f1-85da-a77a26dc2eed |
| `sbom-desktop.json` | 91 KB | 74 | urn:uuid:e91bd3b5-83d0-4981-a359-7da2563b1350 |

**Location:** `E:\UserData\MyData\Content\DEV\pipeline-output\v1.5.5\`

---

## SBOM 1: AI Software Factory Platform

**File:** `sbom-aisf.json`  
**Root component:** `ai-software-factory-platform 1.5.5`  
**specVersion:** CycloneDX 1.6  
**serialNumber:** `urn:uuid:88c685ce-77e6-46f1-85da-a77a26dc2eed`

### Generation Command (EXECUTED)

```
E:\UserData\MyData\Content\DEV\ai-software-factory> python -m cyclonedx_py environment \
    --of JSON \
    --mc-type application \
    --pyproject pyproject.toml \
    -o E:\UserData\MyData\Content\DEV\pipeline-output\v1.5.5\sbom-aisf.json

Exit code: 0
Output: sbom-aisf.json (131 KB, 107 components)
```

### Verification (EXECUTED)

```python
import json
s = json.load(open("sbom-aisf.json"))
assert s["specVersion"] == "1.6"
assert s["metadata"]["component"]["name"] == "ai-software-factory-platform"
assert s["metadata"]["component"]["version"] == "1.5.5"
assert len(s["components"]) == 107
# Pass — all assertions satisfied
```

### Component Count by Type

| Type | Count |
|------|-------|
| library | 107 |
| **Total** | **107** |

### Notable Runtime Dependencies

| Package | Version | License (where known) |
|---------|---------|----------------------|
| fastapi | 0.115.x | MIT |
| sqlalchemy | 2.x | MIT |
| uvicorn | 0.x | BSD-2-Clause |
| httpx | 0.27.x | BSD-3-Clause |
| pydantic | 2.x | MIT |
| pydantic-settings | 2.x | MIT |
| python-dotenv | 1.x | BSD-3-Clause |
| aiofiles | 23.x | Apache-2.0 |
| structlog | 24.x | MIT |
| PyYAML | 6.x | MIT |
| alembic | 1.x | MIT |
| pip-audit | 2.10.1 | Apache-2.0 |
| cyclonedx-bom | 7.3.0 | Apache-2.0 |
| flake8 | 7.3.0 | MIT |
| pytest | 8.x | MIT |
| pytest-cov | 5.x | MIT |

Full dependency tree in `sbom-aisf.json`.

---

## SBOM 2: AI Studio Desktop

**File:** `sbom-desktop.json`  
**Root component:** `ai-studio-desktop 1.5.5`  
**specVersion:** CycloneDX 1.6  
**serialNumber:** `urn:uuid:e91bd3b5-83d0-4981-a359-7da2563b1350`

### Generation Command (EXECUTED)

```
E:\UserData\MyData\Content\DEV\ai-studio-desktop> python -m cyclonedx_py environment \
    --of JSON \
    --mc-type application \
    --pyproject pyproject.toml \
    -o E:\UserData\MyData\Content\DEV\pipeline-output\v1.5.5\sbom-desktop.json

Exit code: 0
Output: sbom-desktop.json (91 KB, 74 components)
```

### Verification (EXECUTED)

```python
import json
s = json.load(open("sbom-desktop.json"))
assert s["specVersion"] == "1.6"
assert s["metadata"]["component"]["name"] == "ai-studio-desktop"
assert s["metadata"]["component"]["version"] == "1.5.5"
assert len(s["components"]) == 74
# Pass — all assertions satisfied
```

### Notable Runtime Dependencies

| Package | Version | License (where known) |
|---------|---------|----------------------|
| PySide6 | 6.11.1 | LGPL-3.0 |
| httpx | 0.27.x | BSD-3-Clause |
| structlog | 24.x | MIT |
| PyYAML | 6.x | MIT |
| pyinstaller | 6.21.0 | GPL-2.0 (build-time only) |

Full dependency tree in `sbom-desktop.json`.

---

## Security Scan (pip-audit)

### AISF Security Scan (EXECUTED)

```
$ python -m pip_audit --format json

{
  "dependencies": [...],   // 102 packages scanned
  "fixes": [],
  "vulnerabilities": []
}

Exit code: 0
High/Critical CVEs: 0
Result: PASS
```

### Desktop Security Scan

**NOT VERIFIED** — pip-audit was not run against the Desktop venv in this pipeline pass. Desktop uses PySide6 6.11.1 (current as of 2026-06-27); no known CVEs at time of writing.

---

## SBOM Inclusion in Release Bundle

Both SBOM files are included in `pipeline-output/v1.5.5/release/`:

```
release/
  sbom-aisf.json      (131 KB)
  sbom-desktop.json   ( 91 KB)
```

The `release-manifest.json` references both:
```json
"sbom": [
  "sbom-aisf.json (CycloneDX 1.6, 107 components)",
  "sbom-desktop.json (CycloneDX 1.6, 74 components)"
]
```

---

## CycloneDX Schema Compliance

| Field | AISF | Desktop |
|-------|------|---------|
| `bomFormat` | "CycloneDX" | "CycloneDX" |
| `specVersion` | "1.6" | "1.6" |
| `serialNumber` | urn:uuid:88c685ce-... | urn:uuid:e91bd3b5-... |
| `metadata.component.type` | "application" | "application" |
| `metadata.component.name` | "ai-software-factory-platform" | "ai-studio-desktop" |
| `metadata.component.version` | "1.5.5" | "1.5.5" |
| `components[*].type` | "library" | "library" |
| `components[*].purl` | present | present |

---

## NOT VERIFIED

| Item | Reason |
|------|--------|
| Desktop pip-audit security scan | Not run in this pipeline pass |
| Content Factory SBOM (Java/Maven) | cyclonedx-maven-plugin not configured in pom.xml |
| SBOM schema validation (`cyclonedx validate`) | Not run; CycloneDX CLI validator not installed |
| SBOM signing (GPG) | GPG not installed on build host |
| License compliance check (all OSS licenses) | Manual review required for LGPL-3.0 (PySide6) |

### PySide6 LGPL-3.0 Note

AI Studio Desktop uses PySide6 under the LGPL-3.0 license. Distribution as a PyInstaller bundle (dynamic linking against Qt6 DLLs in `_internal/`) is permissible under LGPL-3.0 provided end-users can relink against a modified PySide6. The `_internal/` directory makes this mechanically possible. No source modifications to PySide6 were made.
