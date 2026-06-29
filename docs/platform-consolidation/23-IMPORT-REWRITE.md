---
knowledge_id: KNW-PLAT-ARCH-023
title: "Import Rewrite Specification"
status: AUTHORITATIVE
phase: "2.1D.0"
version: "1.0.0"
created: "2026-06-29"
purpose: "Complete specification of all import path rewrites needed during platform consolidation"
canonical_source: "architecture/docs/platform-consolidation/23-IMPORT-REWRITE.md"
dependencies:
  - "12-IMPORT-RULES.md"
  - "22-MIGRATION-ENGINE.md"
  - "04-PACKAGE-MODEL.md"
related_documents:
  - "24-ROLLBACK-ENGINE.md"
  - "25-VERIFICATION.md"
acceptance_criteria:
  - "All rewrite rules are regex-transformable (no special cases)"
  - "Rules are ordered by specificity (most specific first)"
  - "Edge cases (string literals, comments) are handled"
verification_checklist:
  - "[ ] Each rule has a before/after example"
  - "[ ] Rules cover all old package namespaces"
  - "[ ] Comment and string literal handling defined"
future_extensions:
  - "AST-based rewriting for complex cases"
  - "Type stub update rules"
---

# Import Rewrite Specification

## Rewrite Scope

Import rewrites apply ONLY to files in:
- `platform/` (moved or updated modules)
- `source/*/` (product code updating SDK paths)

Import rewrites do NOT apply to:
- `architecture/` (documentation)
- `tests/` that are OUTSIDE platform/ (use whatever paths they use)
- Files marked `# GENERATED`

---

## Rule Priority

Rules are applied in order. Most-specific patterns first.

---

## Ruleset R1 — platform_kernel Namespace

**Applies when:** `platform_kernel/` is merged into `platform/kernel/`  
**Rule ID:** DR-007 (from 10-DEPENDENCY-RULES.md)

| Before | After |
|--------|-------|
| `from platform_kernel.di import` | `from platform.kernel.di import` |
| `from platform_kernel.events import` | `from platform.kernel.events import` |
| `from platform_kernel.events.bus import` | `from platform.kernel.events.bus import` |
| `from platform_kernel.config import` | `from platform.kernel.config import` |
| `from platform_kernel.lifecycle import` | `from platform.kernel.lifecycle import` |
| `from platform_kernel.registry import` | `from platform.kernel.registry import` |
| `import platform_kernel.di` | `import platform.kernel.di` |
| `import platform_kernel` | `import platform.kernel` |

**Regex pattern:**
```python
RULE_R1 = [
    ImportRewrite(
        old_pattern=r"from platform_kernel\.(\w+(?:\.\w+)*) import",
        new_pattern=r"from platform.kernel.\1 import",
        scope="all",
        rule_id="DR-007",
    ),
    ImportRewrite(
        old_pattern=r"import platform_kernel\.(\w+(?:\.\w+)*)",
        new_pattern=r"import platform.kernel.\1",
        scope="all",
        rule_id="DR-007",
    ),
]
```

---

## Ruleset R2 — platform_sdk Namespace

**Applies when:** `platform_sdk/` is merged into `platform/sdk/`  
**Rule ID:** DR-007

| Before | After |
|--------|-------|
| `from platform_sdk.ai import` | `from platform.sdk.ai import` |
| `from platform_sdk.providers import` | `from platform.sdk.providers import` |
| `from platform_sdk.knowledge import` | `from platform.sdk.knowledge import` |
| `from platform_sdk.workflows import` | `from platform.sdk.workflows import` |
| `from platform_sdk.resources import` | `from platform.sdk.resources import` |
| `from platform_sdk.events import` | `from platform.sdk.events import` |
| `from platform_sdk.types import` | `from platform.sdk.types import` |
| `from platform_sdk.exceptions import` | `from platform.sdk.exceptions import` |
| `import platform_sdk` | `import platform.sdk` |

**Regex pattern:**
```python
RULE_R2 = [
    ImportRewrite(
        old_pattern=r"from platform_sdk\.(\w+(?:\.\w+)*) import",
        new_pattern=r"from platform.sdk.\1 import",
        scope="all",
        rule_id="DR-007",
    ),
]
```

---

## Ruleset R3 — AI Runtime Path

**Applies when:** `platform/ai_runtime/` moves to `platform/runtimes/ai_runtime/`

| Before | After |
|--------|-------|
| `from platform.ai_runtime.resource_intelligence` | `from platform.runtimes.ai_runtime.resource_intelligence` |
| `from platform.ai_runtime import` | `from platform.runtimes.ai_runtime import` |
| `from ai_runtime.resource_intelligence` | `from platform.runtimes.ai_runtime.resource_intelligence` |
| `import platform.ai_runtime` | `import platform.runtimes.ai_runtime` |

**Regex pattern:**
```python
RULE_R3 = [
    ImportRewrite(
        old_pattern=r"from platform\.ai_runtime\.([\w.]+) import",
        new_pattern=r"from platform.runtimes.ai_runtime.\1 import",
        scope="all",
        rule_id="M",
    ),
    ImportRewrite(
        old_pattern=r"from ai_runtime\.([\w.]+) import",
        new_pattern=r"from platform.runtimes.ai_runtime.\1 import",
        scope="all",
        rule_id="M",
    ),
]
```

---

## Edge Case Handling

### String Literals
Rewrites operate on import statements only. String literals that happen to contain
old paths (e.g., in log messages or docstrings) are NOT rewritten.

**Detection:** Only lines that match `^\s*(from|import)\s+` are candidates.

```python
def is_import_line(line: str) -> bool:
    stripped = line.lstrip()
    return stripped.startswith("from ") or stripped.startswith("import ")
```

### Comments
Lines that start with `#` after stripping are skipped entirely.

### Multi-line Imports
```python
from platform_kernel.di import (
    Container,
    Lifetime,
)
```
Only the first line (the `from ... import` line) is rewritten. The continuation
lines are untouched.

### TYPE_CHECKING Blocks
```python
if TYPE_CHECKING:
    from platform_kernel.events import EventBus  # must also be rewritten
```
The rewriter applies to all lines regardless of `TYPE_CHECKING` context.

---

## Rewriter Implementation Sketch

```python
def rewrite_file(path: str, rules: list[ImportRewrite]) -> int:
    """Returns count of lines changed."""
    lines = Path(path).read_text(encoding="utf-8").splitlines(keepends=True)
    changed = 0
    for i, line in enumerate(lines):
        if not is_import_line(line):
            continue
        for rule in rules:
            new_line, n = re.subn(rule.old_pattern, rule.new_pattern, line)
            if n > 0:
                lines[i] = new_line
                changed += 1
                break  # apply only first matching rule per line
    if changed:
        Path(path).write_text("".join(lines), encoding="utf-8")
    return changed
```

---

## Rewrite Verification

After rewriting, verify no old paths remain:

```bash
# Should return zero matches
grep -r "from platform_kernel" platform/ source/
grep -r "import platform_kernel" platform/ source/
grep -r "from platform_sdk" platform/ source/
grep -r "from platform\.ai_runtime" platform/ source/
grep -r "from ai_runtime" platform/ source/
```

Any remaining matches are migration failures → trigger rollback.

---

## File Scope Table

| Rule | Applies to files in |
|------|---------------------|
| R1 (platform_kernel) | `platform/`, `source/*/` |
| R2 (platform_sdk) | `platform/`, `source/*/` |
| R3 (ai_runtime path) | `platform/` only |
