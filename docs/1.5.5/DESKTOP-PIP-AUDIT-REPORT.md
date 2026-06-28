# Desktop pip-audit Security Scan Report — AI Studio Desktop v1.5.5

**Generated:** 2026-06-27T13:17:33Z  
**Tool:** pip-audit 2.10.1  
**Component:** ai-studio-desktop  
**Virtual Environment:** `E:\UserData\MyData\Content\DEV\ai-studio-desktop\.venv`  
**Python:** 3.13.14

---

## Summary

| Metric | Value |
|--------|-------|
| Packages Scanned | 74 |
| Vulnerabilities Found | **0** |
| High/Critical CVEs | **0** |
| Scan Exit Code | **0** |
| Scan Duration | ~12s |
| Database | OSV (Open Source Vulnerabilities) |

**Result: PASS — No known vulnerabilities found**

---

## Execution Evidence (VERIFIED)

```
$ python -m pip_audit --format json -o desktop-pip-audit.json

No known vulnerabilities found
Exit code: 0
Duration: 12s (2026-06-27T13:17:21Z → 2026-06-27T13:17:33Z)
Output: desktop-pip-audit.json (74 packages scanned)
```

---

## Packages Scanned (74)

| Package | Version | Known CVEs |
|---------|---------|------------|
| altgraph | 0.17.5 | 0 |
| anyio | 4.14.1 | 0 |
| arrow | 1.4.0 | 0 |
| attrs | 26.1.0 | 0 |
| boolean-py | 5.0 | 0 |
| cachecontrol | 0.14.4 | 0 |
| certifi | 2026.6.17 | 0 |
| chardet | 5.2.0 | 0 |
| charset-normalizer | 3.4.7 | 0 |
| colorama | 0.4.6 | 0 |
| coverage | 7.14.3 | 0 |
| cyclonedx-bom | 7.3.0 | 0 |
| cyclonedx-python-lib | 11.11.0 | 0 |
| defusedxml | 0.7.1 | 0 |
| filelock | 3.29.4 | 0 |
| flake8 | 7.3.0 | 0 |
| fqdn | 1.5.1 | 0 |
| h11 | 0.16.0 | 0 |
| httpcore | 1.0.9 | 0 |
| httpx | 0.28.1 | 0 |
| idna | 3.18 | 0 |
| iniconfig | 2.3.0 | 0 |
| isoduration | 20.11.0 | 0 |
| jsonpointer | 3.1.1 | 0 |
| jsonschema | 4.26.0 | 0 |
| jsonschema-specifications | 2025.9.1 | 0 |
| lark | 1.3.1 | 0 |
| license-expression | 30.4.4 | 0 |
| lxml | 6.1.1 | 0 |
| markdown-it-py | 4.2.0 | 0 |
| mccabe | 0.7.0 | 0 |
| mdurl | 0.1.2 | 0 |
| msgpack | 1.2.1 | 0 |
| packageurl-python | 0.17.6 | 0 |
| packaging | 26.2 | 0 |
| pefile | 2024.8.26 | 0 |
| pip | 26.1.2 | 0 |
| pip-api | 0.0.34 | 0 |
| pip-audit | 2.10.1 | 0 |
| pip-requirements-parser | 32.0.1 | 0 |
| platformdirs | 4.10.0 | 0 |
| pluggy | 1.6.0 | 0 |
| py-serializable | 2.1.0 | 0 |
| pycodestyle | 2.14.0 | 0 |
| pyflakes | 3.4.0 | 0 |
| pygments | 2.20.0 | 0 |
| pyinstaller | 6.21.0 | 0 |
| pyinstaller-hooks-contrib | 2026.6 | 0 |
| pyparsing | 3.3.2 | 0 |
| **PySide6** | **6.11.1** | **0** |
| pyside6-addons | 6.11.1 | 0 |
| pyside6-essentials | 6.11.1 | 0 |
| pytest | 9.1.1 | 0 |
| pytest-cov | 7.1.0 | 0 |
| python-dateutil | 2.9.0.post0 | 0 |
| pywin32-ctypes | 0.2.3 | 0 |
| pyyaml | 6.0.3 | 0 |
| referencing | 0.37.0 | 0 |
| requests | 2.34.2 | 0 |
| rfc3339-validator | 0.1.4 | 0 |
| rfc3986-validator | 0.1.1 | 0 |
| rfc3987-syntax | 1.1.0 | 0 |
| rich | 15.0.0 | 0 |
| rpds-py | 2026.5.1 | 0 |
| setuptools | 82.0.1 | 0 |
| shiboken6 | 6.11.1 | 0 |
| six | 1.17.0 | 0 |
| sortedcontainers | 2.4.0 | 0 |
| tomli | 2.4.1 | 0 |
| tomli-w | 1.2.0 | 0 |
| tzdata | 2026.2 | 0 |
| uri-template | 1.3.0 | 0 |
| urllib3 | 2.7.0 | 0 |
| webcolors | 25.10.0 | 0 |

---

## Notable Dependency Notes

### PySide6 6.11.1 (Qt6 bindings)
- **License:** LGPL-3.0
- **CVEs:** 0 found by pip-audit (OSV database)
- **Note:** Qt CVEs are tracked at qt.io/security, not in PyPI/OSV. A separate Qt security advisory check is recommended for production environments.
- **Distribution compliance:** LGPL-3.0 compliant when distributed as a PyInstaller bundle (dynamic linking, _internal/ directory allows user relinking)

### httpx 0.28.1
- HTTP client used for auto-updater download
- No known CVEs at scan time (2026-06-27)

### requests 2.34.2
- High-level HTTP library
- No known CVEs at scan time

---

## Comparison: AISF vs Desktop

| Component | Packages | CVEs | Tool |
|-----------|----------|------|------|
| AISF (local) | 102 | 0 | pip-audit 2.10.1 |
| AISF (CI/GitHub Actions) | 87 | 0 | pip-audit 2.10.1 |
| Desktop (local) | 74 | 0 | pip-audit 2.10.1 |

Note: AISF local (102) vs AISF CI (87) package count difference is due to CI using a fresh venv with fewer dev/test dependencies.

---

## NOT VERIFIED

| Item | Reason |
|------|--------|
| Qt CVE advisory scan | Not in OSV/PyPI; requires manual check at qt.io/security |
| Desktop CI pip-audit | Not yet configured in CI workflow for Desktop component |
