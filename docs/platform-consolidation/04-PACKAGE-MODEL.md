---
knowledge_id: KNW-PLAT-ARCH-004
title: "Platform Package Model"
status: AUTHORITATIVE
phase: "2.1D.0"
version: "1.0.0"
created: "2026-06-29"
purpose: "Define Python packaging strategy, pyproject.toml structure, namespace packages, and distribution"
canonical_source: "architecture/docs/platform-consolidation/04-PACKAGE-MODEL.md"
dependencies:
  - "03-DIRECTORY-ARCHITECTURE.md"
related_documents:
  - "10-DEPENDENCY-RULES.md"
  - "12-IMPORT-RULES.md"
  - "14-PLATFORM-MANIFEST.md"
acceptance_criteria:
  - "Single pyproject.toml governs the entire Platform"
  - "Namespace is 'platform' throughout"
  - "Editable install command is specified"
  - "All optional dependency groups are named"
verification_checklist:
  - "[ ] pyproject.toml schema is valid TOML"
  - "[ ] No sub-packages have their own pyproject.toml"
  - "[ ] Editable install tested in clean venv"
  - "[ ] All dependency groups resolve without conflicts"
future_extensions:
  - "Optional: split into platform-kernel and platform-sdk as separate distributions"
  - "PEP 420 namespace package exploration"
---

# Platform Package Model

## Distribution Strategy

The entire Platform ships as **one Python distribution package**.

| Property | Value |
|----------|-------|
| Distribution name | `ai-studio-platform` |
| Import namespace | `platform` |
| Single pyproject.toml | `platform/pyproject.toml` |
| Install mode | Editable (`pip install -e .`) |
| Python minimum | `>=3.11` |
| Build backend | `setuptools>=71` |

**Rationale for single distribution:**
- One `pip install` activates the entire platform
- Products don't manage per-runtime installs
- Internal refactoring doesn't break product installs
- Dependency conflicts resolved in one place

---

## pyproject.toml Canonical Schema

```toml
[build-system]
requires = ["setuptools>=71", "wheel"]
build-backend = "setuptools.backends.legacy:build"

[project]
name = "ai-studio-platform"
version = "2.0.0"
description = "AI Studio Platform — Kernel, SDK, Runtimes, Services"
requires-python = ">=3.11"
license = { text = "Proprietary" }

dependencies = [
    # Kernel (zero runtime deps — stdlib only beyond these)
    "pydantic>=2.7.0",
    "pydantic-settings>=2.3.0",
    "anyio>=4.4.0",
    # Services
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.30.0",
    "structlog>=24.1.0",
    "prometheus-client>=0.20.0",
    # Data
    "sqlalchemy>=2.0.30",
    "aiosqlite>=0.20.0",
    "alembic>=1.13.0",
    # Security
    "cryptography>=42.0.0",
    "python-jose[cryptography]>=3.3.0",
    # HTTP
    "httpx>=0.27.0",
    "python-multipart>=0.0.9",
    # AI providers (optional — see extras)
    "anthropic>=0.34.0",
    "tiktoken>=0.7.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.2.0",
    "pytest-asyncio>=0.23.0",
    "pytest-cov>=5.0.0",
    "ruff>=0.4.0",
    "mypy>=1.10.0",
]
desktop = [
    "PySide6>=6.7.0",
]
openai = [
    "openai>=1.30.0",
]
google = [
    "google-generativeai>=0.7.0",
]
knowledge = [
    "networkx>=3.3",
    "numpy>=1.26.0",
]
all = [
    "ai-studio-platform[dev,desktop,openai,google,knowledge]",
]

[project.scripts]
platform = "platform.tools.cli:main"
platform-migrate = "platform.tools.migrate.cli:main"
platform-verify = "platform.tools.verify.cli:main"

[tool.setuptools.packages.find]
where = ["."]
include = [
    "platform*",
    "platform.kernel*",
    "platform.sdk*",
    "platform.runtimes*",
    "platform.services*",
    "platform.api*",
    "platform.security*",
    "platform.common*",
    "platform.tools*",
]
exclude = [
    "platform.tests*",
    "platform.docs*",
]

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
addopts = "--tb=short -q"
pythonpath = ["."]
importmode = "importlib"

[tool.ruff]
line-length = 100
target-version = "py311"
[tool.ruff.lint]
select = ["E", "F", "W", "I", "UP", "B", "SIM"]
ignore = ["B008"]

[tool.mypy]
python_version = "3.11"
strict = true
ignore_missing_imports = true
exclude = ["tests/", "docs/"]
```

---

## Import Namespace Map

| Import Path | Maps To Directory | Visibility |
|-------------|------------------|-----------|
| `platform.kernel` | `kernel/` | INTERNAL (kernel team) |
| `platform.sdk` | `sdk/` | PUBLIC |
| `platform.runtimes.ai_runtime` | `runtimes/ai_runtime/` | INTERNAL (ai team) |
| `platform.runtimes.knowledge_runtime` | `runtimes/knowledge_runtime/` | INTERNAL |
| `platform.runtimes.provider_runtime` | `runtimes/provider_runtime/` | INTERNAL |
| `platform.runtimes.workflow_runtime` | `runtimes/workflow_runtime/` | INTERNAL |
| `platform.runtimes.orchestration_runtime` | `runtimes/orchestration_runtime/` | INTERNAL |
| `platform.runtimes.event_runtime` | `runtimes/event_runtime/` | INTERNAL |
| `platform.runtimes.resource_runtime` | `runtimes/resource_runtime/` | INTERNAL |
| `platform.services` | `services/` | INTERNAL via SDK |
| `platform.api` | `api/` | PUBLIC (HTTP) |
| `platform.common` | `common/` | INTERNAL |
| `platform.security` | `security/` | INTERNAL via services |
| `platform.tools` | `tools/` | DEV only |

---

## Editable Install

```bash
# From AIStudio/platform/
pip install -e ".[dev]"

# With optional extras
pip install -e ".[dev,knowledge,openai]"
```

After install, the following all resolve to the same directory:
```python
import platform.sdk                    # → platform/sdk/__init__.py
import platform.kernel                 # → platform/kernel/__init__.py
import platform.runtimes.ai_runtime   # → platform/runtimes/ai_runtime/__init__.py
```

---

## Namespace Package Rules

1. **No implicit namespace packages.** Every directory has `__init__.py`.
2. **The `platform` root namespace is exclusive** to this distribution.
   No other package may use the `platform.*` namespace.
3. **Relative imports** are allowed within a single runtime directory.
4. **Absolute imports** (`from platform.kernel...`) are required across directory boundaries.

---

## Dependency Groups

| Group | Purpose | Who Uses |
|-------|---------|---------|
| `[default]` | Core runtime dependencies | All deployments |
| `[dev]` | Testing, linting, type checking | CI/CD, developers |
| `[desktop]` | PySide6 for AI Studio Desktop | Desktop product only |
| `[openai]` | OpenAI provider plugin | Deployments using OpenAI |
| `[google]` | Google AI provider plugin | Deployments using Google |
| `[knowledge]` | NetworkX + NumPy for knowledge runtime | Knowledge-enabled deployments |
| `[all]` | Everything | Development environment |

---

## What Is NOT in the Platform Package

| Thing | Location | Reason |
|-------|----------|--------|
| Product code | `source/ai-software-factory/` | Products are separate |
| Content Factory | `source/content-factory/` | Product — depends on platform |
| AI Studio Desktop product code | `source/ai-studio-desktop/` | Product |
| Archive code | `archive/` | Deprecated — not packaged |
| Playground | `playground/` | Experimental — not packaged |
