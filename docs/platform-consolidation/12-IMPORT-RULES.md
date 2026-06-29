---
knowledge_id: KNW-PLAT-ARCH-012
title: "Platform Import Rules"
status: AUTHORITATIVE
phase: "2.1D.0"
version: "1.0.0"
created: "2026-06-29"
purpose: "Define import conventions, absolute vs relative rules, rewrite rules for migration"
canonical_source: "architecture/docs/platform-consolidation/12-IMPORT-RULES.md"
dependencies:
  - "10-DEPENDENCY-RULES.md"
  - "11-LAYER-RULES.md"
related_documents:
  - "23-IMPORT-REWRITE.md"
  - "04-PACKAGE-MODEL.md"
acceptance_criteria:
  - "Every import form has an explicit allow/deny ruling"
  - "Rewrite rules are regex-transformable (no manual cases)"
  - "Generated import detection is unambiguous"
verification_checklist:
  - "[ ] All import forms covered (absolute, relative, star, TYPE_CHECKING)"
  - "[ ] Rewrite table covers all old → new path mappings"
  - "[ ] Import scanner can detect all forbidden forms"
future_extensions:
  - "Auto-generated __all__ from module.yaml"
  - "Import cycle visualiser"
---

# Platform Import Rules

## Import Form Classification

| Form | Example | Status |
|------|---------|--------|
| Absolute public | `from platform.sdk.ai import optimize_routing` | REQUIRED for cross-boundary |
| Absolute internal | `from platform.runtimes.ai_runtime.quota_manager import ...` | ALLOWED within same runtime |
| Relative (same package) | `from .models import UsageRecord` | ALLOWED within module dir |
| Relative (parent) | `from ..quota_manager import ResourceQuotaEngine` | ALLOWED within same runtime |
| Star import | `from platform.sdk import *` | FORBIDDEN always |
| `__future__` | `from __future__ import annotations` | REQUIRED in all files |
| TYPE_CHECKING | `from typing import TYPE_CHECKING` | ALLOWED for type hints only |
| Dynamic | `importlib.import_module(...)` | FORBIDDEN except in DI container |

---

## Absolute Import Rules

### IR-001 — Cross-boundary imports must be absolute
Any import that crosses a runtime, layer, or package boundary must use the full absolute path.

```python
# CORRECT — absolute across boundary
from platform.kernel.events.bus import EventBus
from platform.runtimes.ai_runtime.resource_intelligence import ResourceIntelligenceService

# INCORRECT — relative across boundary
from ....kernel.events.bus import EventBus  # how many dots? unreliable
```

### IR-002 — SDK public imports use only platform.sdk.*
```python
# product code
from platform.sdk.ai import optimize_routing     # CORRECT
from platform.sdk.resources import check_quota   # CORRECT
from platform.runtimes.ai_runtime import ...     # FORBIDDEN
```

---

## Relative Import Rules

### IR-003 — Relative imports allowed within a single module directory
```python
# Inside platform/runtimes/ai_runtime/resource_intelligence/quota_manager/engine.py
from .models import ResourceQuota, QuotaStatus   # CORRECT — same package
from ..forecast_engine.engine import ForecastEngine  # CORRECT — same runtime
from ....kernel.events import EventBus           # FORBIDDEN — crosses runtime boundary
```

### IR-004 — Maximum relative depth = 3
Relative imports deeper than `...` (three dots) indicate a directory structure problem.
```python
from ....something import X   # FORBIDDEN — restructure instead
```

---

## Star Import Rule

### IR-005 — Star imports are always forbidden
```python
from platform.sdk.ai import *      # FORBIDDEN
from platform.common.types import * # FORBIDDEN
```

Exception: `platform/sdk/__init__.py` may use star imports internally to re-export symbols.
This exception is controlled and not visible to consumers.

---

## TYPE_CHECKING Rule

### IR-006 — TYPE_CHECKING imports are allowed for type hints only
```python
from __future__ import annotations
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from platform.runtimes.knowledge_runtime.graph import KnowledgeGraph
```

This does NOT constitute a runtime dependency. The import never executes.
TYPE_CHECKING imports across runtime boundaries ARE ALLOWED.
They must not be used to call functions or instantiate objects.

---

## Future Annotations Rule

### IR-007 — `from __future__ import annotations` required in all platform files
Enables forward references and defers annotation evaluation.
```python
# Every platform Python file begins with:
from __future__ import annotations
```

---

## Old Path Rewrite Map

After consolidation, these old import paths must be rewritten:

| Old Path | New Path | Rule |
|----------|----------|------|
| `from platform_kernel.di import ...` | `from platform.kernel.di import ...` | DR-007 |
| `from platform_kernel.events import ...` | `from platform.kernel.events import ...` | DR-007 |
| `from platform_kernel.config import ...` | `from platform.kernel.config import ...` | DR-007 |
| `from platform_sdk.ai import ...` | `from platform.sdk.ai import ...` | DR-007 |
| `from platform_sdk.providers import ...` | `from platform.sdk.providers import ...` | DR-007 |
| `from ai_runtime.resource_intelligence ...` | `from platform.runtimes.ai_runtime.resource_intelligence ...` | M |
| `from platform.ai_runtime ...` | `from platform.runtimes.ai_runtime ...` | M |

Full rewrite specification: see 23-IMPORT-REWRITE.md.

---

## `__all__` Convention

### IR-008 — Public modules must define `__all__`
Every `__init__.py` that is part of the public surface must declare `__all__`.

```python
# platform/sdk/ai.py
__all__ = [
    "optimize_routing",
    "analyze_task",
    "plan_execution",
    "predict_cost",
    "get_recommendations",
    "record_execution_outcome",
]
```

Internal modules should NOT define `__all__` — this signals they are private.

---

## Generated Import Detection

Generated files must begin with the marker:
```python
# GENERATED — do not edit manually. Source: platform/tools/generate/...
```

The import scanner skips generated files for violation checks.
Generated imports are allowed to use any path (they are machine-controlled).

---

## Import Validation Rules for CI

```
IR-001: cross-boundary absolute                → ERROR if violated
IR-002: products use sdk only                  → ERROR if violated
IR-003: relative depth ≤ 3                    → WARNING
IR-004: no star imports (except sdk/__init__)  → ERROR
IR-005: TYPE_CHECKING runtime-ok               → INFO
IR-006: __future__ annotations present         → WARNING
IR-007: __all__ in public modules              → WARNING
DR-007: no old platform_kernel/platform_sdk    → ERROR
```
