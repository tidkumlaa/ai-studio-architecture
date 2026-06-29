---
knowledge_id: KNW-PLAT-ARCH-011
title: "Platform Layer Rules"
status: AUTHORITATIVE
phase: "2.1D.0"
version: "1.0.0"
created: "2026-06-29"
purpose: "Define platform layers, their boundaries, allowed flow directions, and enforcement"
canonical_source: "architecture/docs/platform-consolidation/11-LAYER-RULES.md"
dependencies:
  - "10-DEPENDENCY-RULES.md"
related_documents:
  - "12-IMPORT-RULES.md"
  - "25-VERIFICATION.md"
acceptance_criteria:
  - "Every layer has a numeric level"
  - "Allowed/forbidden flow directions are exhaustive"
  - "Layer boundaries are path-detectable (no heuristics)"
verification_checklist:
  - "[ ] Layer levels are unique"
  - "[ ] Boundary detection rules are regex-based"
  - "[ ] All 7 layers defined with examples"
future_extensions:
  - "Layer visualisation in 28-DASHBOARD.md"
  - "Per-layer SLO definitions"
---

# Platform Layer Rules

## Layer Definitions

| Level | Layer | Directory Pattern | Owner |
|-------|-------|------------------|-------|
| 0 | **STDLIB** | Python built-in | CPython |
| 1 | **COMMON** | `platform/common/` | Platform Team |
| 2 | **KERNEL** | `platform/kernel/` | Kernel Team |
| 3 | **SERVICES** | `platform/services/` | Services Team |
| 4 | **RUNTIMES** | `platform/runtimes/*/` | Runtime Teams |
| 5 | **SDK** | `platform/sdk/` | SDK Team |
| 6 | **PRODUCTS** | `source/*/` | Product Teams |

---

## Allowed Flow Directions

```
PRODUCTS (6)
    │  may import from
    ▼
SDK (5)
    │  may import from
    ▼
SERVICES (3)  ←── (also imports KERNEL)
    │  may import from
    ▼
RUNTIMES (4)
    │  may import from
    ▼
KERNEL (2)
    │  may import from
    ▼
COMMON (1)
    │  may import from
    ▼
STDLIB (0)
```

**Rule: Lower layers never import from higher layers.**
**Rule: Lateral imports between runtimes are forbidden.**

---

## Boundary Detection Rules

The layer of a Python file is determined by its path:

```python
def detect_layer(file_path: str) -> int:
    if "/sdk/" in file_path:
        return 5
    if "/runtimes/" in file_path:
        return 4
    if "/services/" in file_path:
        return 3
    if "/kernel/" in file_path:
        return 2
    if "/common/" in file_path:
        return 1
    if is_product_path(file_path):  # source/*/
        return 6
    return 0  # stdlib or external
```

An import from file at layer A to module at layer B is:
- **ALLOWED** if B < A (importing from a lower layer)
- **ALLOWED** if B == A and same runtime/service (lateral within container)
- **FORBIDDEN** if B > A (upward import)
- **FORBIDDEN** if B == 4 and file_path is in different runtime (lateral across runtimes)

---

## Layer Rules Table

| Rule | Description | Severity |
|------|-------------|---------|
| LR-001 | COMMON imports STDLIB only | ERROR |
| LR-002 | KERNEL imports COMMON and STDLIB only | ERROR |
| LR-003 | SERVICES import KERNEL, COMMON, STDLIB | ERROR |
| LR-004 | RUNTIMES import KERNEL, SERVICES, COMMON, STDLIB | ERROR |
| LR-005 | SDK imports SERVICES, KERNEL, COMMON, STDLIB | ERROR |
| LR-006 | PRODUCTS import SDK only (from platform.*) | ERROR |
| LR-007 | No runtime imports another runtime directly | ERROR |
| LR-008 | No service imports a runtime | ERROR |
| LR-009 | No kernel imports a service | ERROR |
| LR-010 | Tests are exempt from visibility rules | INFO |
| LR-011 | Tools import KERNEL, COMMON, STDLIB, TOOLS only | WARNING |

---

## Allowed Lateral Dependencies

**Within SERVICES (layer 3):** Services may depend on other services acyclically.
```
service.audit → service.auth       ✓ (audit needs auth context)
service.notifications → service.audit ✓ (notifications write audit log)
service.auth → service.audit       ✗ FORBIDDEN (cycle)
```

**Within RUNTIMES (layer 4):** A runtime may NOT import another runtime.
Communication via events only (see 10-DEPENDENCY-RULES.md DR-002).

**Within a single runtime:** Modules within the same runtime may import each other.
Dependency graph within a runtime must be acyclic (no cycles between modules).

---

## Layer Violation Examples

```python
# LR-001 VIOLATION — common imports kernel
# platform/common/utils.py
from platform.kernel.di import Container   # ← ERROR: common cannot import kernel

# LR-002 VIOLATION — kernel imports service
# platform/kernel/bootstrap.py
from platform.services.auth import AuthService  # ← ERROR: kernel cannot import service

# LR-004 VIOLATION — runtime imports other runtime
# platform/runtimes/ai_runtime/router.py
from platform.runtimes.knowledge_runtime.graph import KnowledgeGraph  # ← ERROR

# LR-006 VIOLATION — product imports runtime
# source/ai-software-factory/agent.py
from platform.runtimes.ai_runtime.routing_optimizer import RoutingOptimizer  # ← ERROR
```

---

## Enforcement

Layer rules are enforced by three mechanisms:

### 1. Import Linter (CI — mandatory)
```bash
platform-verify layers --strict
# Exits non-zero if any ERROR-level rule is violated
```

### 2. Pre-commit hook (development — recommended)
```yaml
# .pre-commit-config.yaml
- repo: local
  hooks:
    - id: platform-layer-check
      name: Platform Layer Rules
      entry: platform-verify layers
      language: python
      pass_filenames: false
```

### 3. Runtime assertion (optional — development only)
```python
# kernel/bootstrap.py
if __debug__:
    from platform.tools.verify import LayerVerifier
    LayerVerifier().assert_clean()
```

---

## Layer Metrics

| Metric | Target | Alert threshold |
|--------|--------|----------------|
| Layer violations | 0 | > 0 |
| Upward dependencies | 0 | > 0 |
| Cross-runtime imports | 0 | > 0 |
| Cyclic service dependencies | 0 | > 0 |
