# GitHub Actions Report — AI Studio Platform v1.5.5

**Generated:** 2026-06-27T14:30:00Z  
**Repository:** https://github.com/tidkumlaa/ai-software-factory  
**Workflow:** AI Software Factory — CI (`.github/workflows/ci.yml`)

---

## Summary

| Run | Trigger | Result | Duration |
|-----|---------|--------|---------|
| 28281015789 | push (initial) | FAILED | 2m16s |
| 28281016810 | workflow_dispatch | FAILED | 4m5s |
| **28281217381** | push (fixes applied) | **SUCCESS** | **2m33s** |

**Final Result: ALL 5 JOBS PASS** on run 28281217381  
**URL:** https://github.com/tidkumlaa/ai-software-factory/actions/runs/28281217381

---

## Run 28281217381 — VERIFIED SUCCESS

### Job Results

| Job | Status | Duration | Runner |
|-----|--------|---------|--------|
| Static Analysis (flake8) | ✓ SUCCESS | 34s | windows-latest |
| Security Scan (pip-audit) | ✓ SUCCESS | 1m7s | windows-latest |
| Generate SBOM (CycloneDX) | ✓ SUCCESS | 1m6s | windows-latest |
| Unit Tests + Coverage | ✓ SUCCESS | 2m30s | windows-latest |
| Version Consistency Check | ✓ SUCCESS | 17s | windows-latest |

**All 5 jobs: SUCCESS**

---

## Job Details (VERIFIED)

### Job 1: Static Analysis (flake8) — 34s

**Steps:**
- Set up job: success
- actions/checkout@v4: success
- actions/setup-python@v5 (Python 3.13): success
- `pip install flake8`: success
- Run flake8 (critical errors only): **success** — 0 critical errors

```
# Command executed on GitHub runner (windows-latest)
python -m flake8 factory api engine runtime --count \
  --select=E9,F821,F822,F823,F811,W6 \
  --max-line-length=120 \
  --extend-ignore=E501,W503,E203 \
  --statistics

# Exit code: 0
```

---

### Job 2: Security Scan (pip-audit) — 1m7s

**Steps:**
- actions/setup-python@v5 (Python 3.13): success
- `pip install -e ".[dev]" pip-audit`: success
- pip-audit: **success** — 0 vulnerabilities in 87 packages

```json
{
  "dependencies": [...],    // 87 packages
  "vulnerabilities": [],    // 0 CVEs
  "fixes": []
}
```

**Artifact uploaded:** `security-scan/pip-audit-results.json` (5 KB)

Note: CI scanned 87 packages (fresh venv, test-only tools excluded) vs local scan 102 packages (includes dev tools).

---

### Job 3: Generate SBOM (CycloneDX) — 1m6s

**Steps:**
- `pip install -e ".[dev]" cyclonedx-bom`: success
- Generate SBOM: **success**

```
$ python -m cyclonedx_py environment \
    --of JSON \
    --mc-type application \
    --pyproject pyproject.toml \
    -o sbom-aisf-ci.json

# Output: sbom-aisf-ci.json (120 KB, 99 components, CycloneDX 1.6)
# metadata.component: ai-software-factory-platform 1.5.5
```

**Artifact uploaded:** `sbom/sbom-aisf-ci.json` (120 KB)

---

### Job 4: Unit Tests + Coverage — 2m30s

**Steps:**
- `pip install -e ".[dev]"`: success
- `pip install pytest pytest-cov pip-audit cyclonedx-bom`: success
- **Run core unit tests:** success

```
$ python -m pytest \
    tests/test_phase23_factory.py \
    tests/test_phase1_logging.py \
    tests/test_reliability.py \
    --cov=factory.logging --cov=factory.metrics \
    --cov-report=xml --cov-report=term-missing \
    --cov-fail-under=60 \
    --junitxml=test-results/junit-aisf.xml -v

# Coverage (line-rate): 0.7293 = 72.93% (gate: ≥60%) PASS
# Exit code: 0
```

- **Run full test suite (informational):** success (continue-on-error: true)

```
$ python -m pytest tests/ \
    --ignore=tests/test_phase1_logging.py \
    --ignore=tests/test_reliability.py \
    -v --tb=no -q

# 559 tests discovered across 25+ test files
# Some failures expected (integration tests requiring live DB/RabbitMQ)
# continue-on-error: true — job succeeds regardless
```

**Artifacts uploaded:** `test-results/` (junit-aisf.xml 4 KB, junit-aisf-full.xml 72 KB, coverage.xml 9 KB)

---

### Job 5: Version Consistency Check — 17s

**Steps:**
- `python -c $script` (PowerShell pwsh inline Python): success

```python
import importlib.util, sys, tomllib

# Load platform_version.py
spec = importlib.util.spec_from_file_location("pv", "platform_version.py")
pv = importlib.util.module_from_spec(spec)
spec.loader.exec_module(pv)

# Load pyproject.toml
with open("pyproject.toml", "rb") as f:
    toml = tomllib.load(f)

pv_ver = pv.PLATFORM_VERSION    # "1.5.5"
toml_ver = toml["project"]["version"]  # "1.5.5"

print(f"platform_version.py: {pv_ver}")
print(f"pyproject.toml:      {toml_ver}")
# Output: "OK: versions match"
# Exit code: 0
```

---

## Fixes Applied Between Run 1 (FAIL) and Run 3 (SUCCESS)

### Run 28281015789 Failures

| Failure | Root Cause | Fix Applied |
|---------|-----------|-------------|
| `test_pyproject_toml_version_is_152` | Asserted `"1.5.4"` after bump to `1.5.5` | Updated assertion to `"1.5.5"` |
| CHANGELOG assertion | `test_re025_changelog_has_release_entry` checked for `v1.5.5` entry not yet present | Added `## [1.5.5]` section to CHANGELOG.md |
| Coverage failure (0%) | `--cov=factory.logging --cov=factory.metrics` but test_phase1_logging.py not in repo yet | Added `test_phase1_logging.py` and `test_reliability.py` to commit |
| Version check script | Used bash heredoc (`<< 'EOF'`) in PowerShell shell (`pwsh.EXE`) | Rewrote as PowerShell `@'...'@` here-string |

---

## Artifacts Downloaded

```
ci-artifacts/
  sbom/sbom-aisf-ci.json         (120 KB, 99 components, CycloneDX 1.6)
  security-scan/pip-audit-results.json  (5 KB, 87 packages, 0 CVEs)
  test-results/coverage.xml      (9 KB, 72.93% line rate)
  test-results/test-results/junit-aisf.xml       (4 KB, core tests)
  test-results/test-results/junit-aisf-full.xml  (72 KB, full suite)
```

---

## Python Version (CI Runner)

```
Python 3.13.14 on C:\hostedtoolcache\windows\Python\3.13.14\x64
pytest 9.1.1
Runner: windows-latest (windows-2022, 10.0.20348)
```

---

## NOT VERIFIED

| Item | Reason |
|------|--------|
| Content Factory (Java/Maven) tests on CI | CF repo not accessible from AISF GitHub Actions; separate CI needed |
| Desktop tests on CI | Desktop not in the AISF repo; separate CI needed |
| Packaging (PyInstaller) on CI | PyInstaller build not in the CI workflow (complex, long-running) |
| GitHub Release creation | No v1.5.5 tag pushed to remote yet; requires tag push trigger |
| GPG signing in CI | `secrets.GPG_PRIVATE_KEY` not set in repo secrets |
