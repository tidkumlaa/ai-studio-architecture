---
knowledge_id: KA-SDK-015
version: "1.0.0"
status: approved
owner: Chief Platform Architect
phase: 2.0D.2.5A
created: 2026-06-29
review_date: 2026-12-29
canonical: true
domain: DOM-PLATFORM
capability: platform-sdk
type: specification
depends_on:
  - id: KA-SDK-002
    reason: "ValidationResult collections"
  - id: KA-SDK-003
    reason: "Pipeline composition for validation chains"
---

# Validation SDK

## Schema · Type · Constraint · Semantic · Pipeline · Reporting

---

## 1. Purpose

The Validation SDK provides a composable, declarative validation framework
for all AI Studio boundaries: API inputs, knowledge object frontmatter,
configuration, rule definitions, IR nodes, and migration artifacts. It
separates validation rules (what is valid) from validation reporting (what
failed and why), and supports progressive validation (fail-fast vs. collect-all).

---

## 2. Validation Taxonomy

```
Validator[T]
├── TypeValidator      — runtime type conformance
├── SchemaValidator    — structural schema (JSON Schema, YAML schema)
├── ConstraintValidator — business rules and invariants
├── SemanticValidator  — cross-field, cross-object consistency
├── PipelineValidator  — chains multiple validators in order
└── CompositeValidator — parallel AND / OR of validators
```

---

## 3. Interfaces

### 3.1 ValidationResult

```python
@dataclass(frozen=True)
class ValidationViolation:
    path:     str        # JSONPath-style: "$.field.subfield[0]"
    code:     str        # Machine-readable code: "REQUIRED_FIELD_MISSING"
    message:  str        # Human-readable description
    severity: str        # "error" | "warning" | "info"
    context:  ImmutableMap[str, Any]  # Additional diagnostic data

@dataclass
class ValidationResult:
    valid:       bool
    violations:  Sequence[ValidationViolation]
    warnings:    Sequence[ValidationViolation]
    infos:       Sequence[ValidationViolation]
    elapsed_ms:  float

    def error_count(self) -> int
    def warning_count(self) -> int
    def merge(self, other: "ValidationResult") -> "ValidationResult"
    def filter_by_severity(self, severity: str) -> "ValidationResult"
    def to_report(self) -> str   # Formatted markdown or text
```

### 3.2 Validator[T]

```python
class Validator(Protocol[T]):
    def validate(self, value: T,
                 context: "ValidationContext | None" = None) -> ValidationResult
    def is_valid(self, value: T) -> bool    # Shorthand — true if no errors
    def name(self) -> str
    def description(self) -> str

@dataclass
class ValidationContext:
    mode:        str     # "fail_fast" | "collect_all"
    root_path:   str     # JSONPath prefix for nested validation
    metadata:    ImmutableMap[str, Any]
    depth:       int     # Current nesting depth (guards against infinite recursion)
```

### 3.3 TypeValidator

```python
class TypeValidator(Validator[Any]):
    """Validates that a value conforms to a Python type or Protocol."""
    def __init__(self, expected_type: type | tuple[type, ...])
    def validate(self, value: Any, ...) -> ValidationResult
    # Produces: TYPE_MISMATCH violation if isinstance fails

class NullabilityValidator(Validator[Any]):
    """Validates presence or absence of None."""
    def required(self) -> "NullabilityValidator"   # None → error
    def optional(self) -> "NullabilityValidator"   # None → allowed
```

### 3.4 SchemaValidator

```python
class SchemaValidator(Validator[dict]):
    """Validates dict/object against a declared schema."""
    def from_dict(self, schema: dict) -> "SchemaValidator"
    def from_json_schema(self, json_schema: dict) -> "SchemaValidator"
    def from_dataclass(self, cls: type) -> "SchemaValidator"

    # Schema language:
    #   {
    #     "fields": {
    #       "knowledge_id": {"type": "str", "required": true, "pattern": "KA-[A-Z]+-\\d{3}"},
    #       "version": {"type": "str", "required": true},
    #       "health_score": {"type": "float", "min": 0.0, "max": 100.0},
    #       "tags": {"type": "list", "items": {"type": "str"}}
    #     }
    #   }
```

### 3.5 ConstraintValidator

```python
class ConstraintValidator(Validator[T]):
    """Business rule validators — preconditions, invariants, postconditions."""

    def add_rule(self, name: str,
                 predicate: Callable[[T], bool],
                 message: str,
                 severity: str = "error") -> "ConstraintValidator[T]"

    def add_cross_field_rule(self, name: str,
                              fn: Callable[[T], bool],
                              message: str) -> "ConstraintValidator[T]"

    def validate(self, value: T, ...) -> ValidationResult

# Example:
# validator = ConstraintValidator()
#   .add_rule("non_empty_title", lambda obj: len(obj.title) > 0, "Title must not be empty")
#   .add_rule("valid_status", lambda obj: obj.status in VALID_STATUSES, "Invalid status")
#   .add_cross_field_rule("review_after_created",
#                          lambda obj: obj.review_date >= obj.created,
#                          "Review date must be after created date")
```

### 3.6 SemanticValidator

```python
class SemanticValidator(Validator[T]):
    """Cross-object semantic consistency. Requires a context store."""
    def validate_with_context(self, value: T,
                               store: Any,
                               validation_ctx: ValidationContext | None = None) -> ValidationResult

# Examples of semantic validation:
# - knowledge_id in frontmatter matches the one in the registry
# - depends_on IDs all exist in the knowledge base
# - capability name is registered
# - domain is in the approved domain list
```

### 3.7 PipelineValidator

```python
class PipelineValidator(Validator[T]):
    """Chains validators sequentially. Fails fast or collects all."""
    def then(self, validator: Validator[T]) -> "PipelineValidator[T]"
    def validate(self, value: T, context: ValidationContext | None = None) -> ValidationResult
    # In fail_fast mode: stops at first error
    # In collect_all mode: runs all validators, merges results
```

### 3.8 CompositeValidator

```python
class AndValidator(Validator[T]):
    """All validators must pass. Equivalent to logical AND."""
    def __init__(self, *validators: Validator[T])

class OrValidator(Validator[T]):
    """At least one validator must pass. Equivalent to logical OR."""
    def __init__(self, *validators: Validator[T])

class NotValidator(Validator[T]):
    """Inverts validation result. Errors become passes, passes become errors."""
    def __init__(self, validator: Validator[T])
```

### 3.9 ValidatorRegistry

```python
class ValidatorRegistry(Protocol):
    """Catalogs named validators for reuse."""
    def register(self, name: str, validator: Validator) -> None
    def get(self, name: str) -> Validator | None
    def validate_by_name(self, name: str, value: Any) -> ValidationResult
    def list(self) -> Sequence[str]
    def compose(self, names: Sequence[str]) -> PipelineValidator
```

---

## 4. Contracts

| ID | Contract |
|----|----------|
| C-VAL-001 | `validate(valid_input).valid == True` and `violations == []`. |
| C-VAL-002 | `validate(invalid_input).valid == False` and `violations` non-empty. |
| C-VAL-003 | `fail_fast` mode: `violations` contains at most 1 error. |
| C-VAL-004 | `collect_all` mode: all violations are collected before returning. |
| C-VAL-005 | `ValidationViolation.path` is a valid JSONPath expression. |
| C-VAL-006 | `ValidationResult.merge(other).valid == True` only if both are valid. |
| C-VAL-007 | `NotValidator(v).validate(x).valid == True` iff `v.validate(x).valid == False`. |
| C-VAL-008 | `OrValidator(A, B).validate(x).valid == True` if A passes or B passes. |
| C-VAL-009 | `SchemaValidator` on unknown extra fields: configurable — error or ignore. |
| C-VAL-010 | `PipelineValidator.then` adds to end of chain; original is not modified. |

---

## 5. Complexity Table

| Validator | Complexity | Notes |
|-----------|-----------|-------|
| TypeValidator | O(1) | `isinstance` check |
| SchemaValidator | O(F) | F = field count |
| ConstraintValidator | O(R) | R = rule count |
| SemanticValidator | O(R × L) | L = lookup cost |
| PipelineValidator | O(sum of stages) | Sequential |
| CompositeValidator | O(max of members) | Parallel logical |

---

## 6. Dependencies

- Collections SDK (KA-SDK-002) — violation lists, context metadata
- Algorithms SDK (KA-SDK-003) — pipeline composition

---

## 7. Extension Points

```python
class ValidationRule(Protocol[T]):
    """Single validation predicate, pluggable into ConstraintValidator."""
    def name(self) -> str: ...
    def severity(self) -> str: ...
    def evaluate(self, value: T) -> tuple[bool, str | None]: ...
    # Returns (passed, optional_message)

class ValidationReporter(Protocol):
    """Custom report format for ValidationResult."""
    def report(self, result: ValidationResult) -> str: ...
    # Built-in: MarkdownReporter, JSONReporter, PlainTextReporter

class SchemaLoader(Protocol):
    """Loads schemas from external sources."""
    def load(self, uri: str) -> dict: ...
    # e.g. from file system, registry, URL
```

---

## 8. Verification

| Check | Criterion |
|-------|-----------|
| V-VAL-001 | Valid input → `result.valid == True`, `violations == []` |
| V-VAL-002 | Required field missing → `REQUIRED_FIELD_MISSING` violation |
| V-VAL-003 | `fail_fast` mode: exactly 1 violation on multi-error input |
| V-VAL-004 | `collect_all` mode: all violations present |
| V-VAL-005 | `PipelineValidator.then` → new pipeline, original unchanged |
| V-VAL-006 | `AndValidator`: fails if any member fails |
| V-VAL-007 | `OrValidator`: passes if any member passes |
| V-VAL-008 | `NotValidator(v).validate(x).valid = !v.validate(x).valid` |
| V-VAL-009 | Schema pattern violation: `path` points to offending field |
| V-VAL-010 | `ValidatorRegistry.compose([A, B])` = `PipelineValidator(A, then=B)` |
