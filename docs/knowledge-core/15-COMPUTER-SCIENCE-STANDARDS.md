---
knowledge_id: KA-SPEC-015
version: "1.0.0"
status: approved
owner: Chief Knowledge Architect
phase: 2.0D.0.5
created: 2026-06-29
review_date: 2026-12-29
canonical: true
domain: DOM-KNOWLEDGE
capability: knowledge-core
type: specification
implements:
  - KA-SPEC-001
---

# Computer Science Standards

## Foundational CS Standards for the Knowledge System Implementation

---

## 1. Purpose

The Knowledge System is a software system. Like any software system, it requires foundational computer science standards that govern how the implementation tools are designed: naming, data structures, error handling, concurrency, immutability, and correctness guarantees.

This document establishes the CS-level standards that all knowledge system tools (`audit.py`, `generate-indexes.py`, `ka-compile`, `knowledge-query`, etc.) must follow.

---

## 2. Language Standard

All knowledge system tools are implemented in **Python 3.11+**.

**Rationale:**
- Python is already the platform language
- Excellent YAML and Markdown processing ecosystem
- Strong graph library support (networkx)
- Type annotations via `typing` module
- Rapid development for architectural tooling

**Prohibited:**
- Python < 3.11
- Mixing languages within a single tool
- Shell scripts as primary logic (acceptable as thin wrappers only)

---

## 3. Naming Conventions

### 3.1 Python Naming

| Element | Convention | Example |
|---------|-----------|---------|
| Module | `snake_case` | `knowledge_index.py` |
| Class | `PascalCase` | `KnowledgeObject` |
| Function | `snake_case` | `compute_health_score` |
| Constant | `UPPER_SNAKE_CASE` | `MAX_DOCUMENT_LINES` |
| Variable | `snake_case` | `knowledge_id` |
| Private | `_leading_underscore` | `_validate_id` |
| Type alias | `PascalCase` | `KnowledgeID = str` |

### 3.2 Knowledge System Naming

| Element | Convention | Example |
|---------|-----------|---------|
| Knowledge ID | `KA-[TYPE]-[NNN]` | `KA-ARCH-001` |
| Capability name | `kebab-case` | `workflow-runtime` |
| Domain ID | `DOM-[UPPER]` | `DOM-WORKFLOW` |
| Event name | `domain.entity.action` | `knowledge.object.approved` |
| CLI command | `kebab-case` | `ka-compile`, `ka-query` |
| Config key | `snake_case` | `max_document_lines` |

### 3.3 File Naming

| File Type | Convention | Example |
|-----------|-----------|---------|
| Knowledge document | `lowercase-with-hyphens.md` or standard names | `overview.md`, `architecture.md` |
| Python tool | `snake_case.py` | `generate_indexes.py` |
| Configuration | `kebab-case.yaml` | `audit-config.yaml` |
| Output file | `UPPER-KEBAB-CASE.md` or `lower-kebab.yaml` | `HEALTH-REPORT.md`, `health-report.yaml` |

---

## 4. Type System Standards

All tools use Python type annotations throughout:

```python
from typing import Optional, List, Dict, Set, Tuple
from dataclasses import dataclass, field
from enum import Enum

# Use dataclasses for data-only structures
@dataclass
class KnowledgeObject:
    knowledge_id: str
    version: str
    status: LifecycleState
    canonical: bool
    domain: str
    capability: str
    type: KnowledgeType
    owner: str
    created: date
    phase: str
    confidence: ConfidenceLevel
    # Optional fields
    review_date: Optional[date] = None
    deprecated_date: Optional[date] = None
    superseded_by: Optional[str] = None
    tags: List[str] = field(default_factory=list)
    relationships: Relationships = field(default_factory=Relationships)
    evidence: List[EvidenceRecord] = field(default_factory=list)

# Use Enum for closed sets
class LifecycleState(Enum):
    CREATED = "created"
    DRAFT = "draft"
    REVIEW = "review"
    APPROVED = "approved"
    DEPRECATED = "deprecated"
    ARCHIVED = "archived"

class KnowledgeType(Enum):
    VISION = "vision"
    OVERVIEW = "overview"
    ARCHITECTURE = "architecture"
    # ... all types
```

**Prohibited:**
- `dict` for structured data that should be a dataclass
- `str` for values that should be an Enum
- `Any` type annotation (except in generic utility functions)
- Mutable default arguments

---

## 5. Error Handling Standards

### 5.1 Exception Hierarchy

```python
class KnowledgeError(Exception):
    """Base exception for all knowledge system errors."""
    pass

class KnowledgeNotFoundError(KnowledgeError):
    """Knowledge ID not found in registry."""
    def __init__(self, knowledge_id: str):
        self.knowledge_id = knowledge_id
        super().__init__(f"Knowledge object not found: {knowledge_id}")

class KnowledgeValidationError(KnowledgeError):
    """Schema validation failure."""
    def __init__(self, knowledge_id: str, violations: List[str]):
        self.violations = violations
        super().__init__(f"Validation failed for {knowledge_id}: {violations}")

class KnowledgeIntegrityError(KnowledgeError):
    """Registry or graph integrity violation."""
    pass

class KnowledgeCycleError(KnowledgeIntegrityError):
    """Cycle detected in relationship graph."""
    def __init__(self, cycle: List[str]):
        self.cycle = cycle
        super().__init__(f"Cycle detected: {' → '.join(cycle)}")
```

### 5.2 Error Handling Rules

- Functions that can fail return `Result[T, E]` or raise typed exceptions — never return `None` for failure
- Every exception carries context (which ID failed, which rule violated)
- Tools collect all errors before reporting — no early exit on first error
- Diagnostic severity levels: `ERROR | WARNING | INFO`

---

## 6. Immutability Standards

### 6.1 Immutable Types

The following types are immutable once created:

| Type | Immutability Rule |
|------|-----------------|
| `KnowledgeID` | Never changes after assignment |
| `KnowledgeType` | Never changes after first assignment |
| `ADRDocument` | Never modified after acceptance |
| `HistoryDocument.entries` | Append-only |
| `ArchiveEntry` | Never modified after archival |

### 6.2 Implementation Pattern

```python
from dataclasses import dataclass

@dataclass(frozen=True)   # frozen=True makes all fields immutable
class KnowledgeID:
    value: str

    def __post_init__(self):
        import re
        if not re.match(r'^KA-[A-Z]{2,5}-\d{3,}$', self.value):
            raise ValueError(f"Invalid Knowledge ID format: {self.value}")
```

---

## 7. Concurrency Standards

The knowledge system tools are designed for:
- **Single-threaded operation**: audit and index generation run sequentially
- **No shared mutable state**: each tool run builds fresh state from disk
- **Idempotent operations**: running a tool twice produces the same result

For the Documentation Intelligence Platform (Phase 2.0D.2+):
- Read operations (queries) are concurrent-safe (immutable indexes)
- Write operations (rebuilds) use exclusive file locking
- The KIE rebuild acquires a lock file before writing indexes

---

## 8. Correctness Standards

### 8.1 Invariant Assertions

All critical invariants are checked at runtime with assertions:

```python
def add_to_registry(obj: KnowledgeObject) -> None:
    assert obj.knowledge_id not in self._registry, \
        f"Invariant I1 violated: duplicate ID {obj.knowledge_id}"
    assert obj.type not in MUTABLE_TYPES or not obj.archived, \
        f"Invariant L2 violated: archived object modified"
    self._registry[obj.knowledge_id] = obj
```

### 8.2 Test Coverage Requirements

All knowledge system tools must achieve:

| Coverage Type | Minimum |
|--------------|---------|
| Line coverage | 80% |
| Branch coverage | 70% |
| Critical path coverage | 100% (health scoring, index build, schema validation) |

### 8.3 Property-Based Testing

The following properties must be tested via property-based tests (Hypothesis library):

| Property | Description |
|----------|-------------|
| ID uniqueness | No two generated IDs are equal |
| Schema round-trip | YAML → KnowledgeObject → YAML is lossless |
| Health score bounds | Health score is always in [0, 100] |
| Graph invariants | Graph built from valid documents satisfies all invariants |
| Lifecycle FSM | No transition violates the FSM definition |

---

## 9. Dependency Standards

### 9.1 Approved Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `pyyaml` | ≥ 6.0 | YAML parsing |
| `markdown-it-py` | ≥ 3.0 | Markdown parsing |
| `networkx` | ≥ 3.0 | Graph operations |
| `jsonschema` | ≥ 4.0 | Schema validation |
| `click` | ≥ 8.0 | CLI interfaces |
| `rich` | ≥ 13.0 | Terminal output |
| `hypothesis` | ≥ 6.0 | Property-based testing |
| `pytest` | ≥ 7.0 | Test framework |

### 9.2 Dependency Rules

- No dependencies with licenses incompatible with the project license
- No runtime dependencies on AI/LLM services (tools must work offline)
- Dependency versions pinned in `tools/requirements.lock`
- New dependencies require explicit approval

---

## References

- [16-DATA-STRUCTURE-CATALOG.md](16-DATA-STRUCTURE-CATALOG.md) — Data structures that follow these standards
- [17-ALGORITHMS-CATALOG.md](17-ALGORITHMS-CATALOG.md) — Algorithms that follow these standards
- [18-COMPLEXITY-GUIDE.md](18-COMPLEXITY-GUIDE.md) — Complexity analysis governed by these standards
- [20-KNOWLEDGE-RUNTIME-ARCHITECTURE.md](20-KNOWLEDGE-RUNTIME-ARCHITECTURE.md) — Runtime that implements these standards
