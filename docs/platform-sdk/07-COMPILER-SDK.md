---
knowledge_id: KA-SDK-008
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
  - id: KA-SDK-007
    reason: "AST from Parser SDK is the compiler frontend input"
  - id: KA-SDK-002
    reason: "IR node collections"
---

# Compiler SDK

## Frontend → IR → Pass Pipeline → Backend

---

## 1. Purpose

The Compiler SDK provides a reusable compilation infrastructure for any
transformation that maps a structured input (AST, schema, model) to one or
more outputs (code, documents, configs, binary formats). It generalizes the
Knowledge Compiler (Phase 2.0D.2) into a universal platform component that
can be used by the Universal Product Factory, Knowledge Operating System,
and any code-generation feature.

---

## 2. Compiler Architecture

```
Input (AST | Model | Schema)
    │
    ▼ Frontend
    IR (Intermediate Representation)
    │
    ├──▶ Pass 1: Validation
    ├──▶ Pass 2: Optimization
    ├──▶ Pass 3: Lowering
    └──▶ Pass N: (pluggable)
         │
    ▼ Backend Dispatcher
    ├──▶ Backend A (e.g. Python codegen)
    ├──▶ Backend B (e.g. Markdown)
    ├──▶ Backend C (e.g. JSON Schema)
    └──▶ Backend D (pluggable)
         │
    ▼ CompilerResult
```

---

## 3. Interfaces

### 3.1 IR (Intermediate Representation)

```python
@dataclass
class IRNode:
    kind:     str
    id:       str
    children: Sequence["IRNode"]
    attrs:    ImmutableMap[str, Any]
    source:   Any | None    # Original input node (for error reporting)

@dataclass
class IRProgram:
    root:        IRNode
    symbol_table: SymbolTable
    diagnostics:  Sequence[IRDiagnostic]
    metadata:     ImmutableMap[str, Any]

@dataclass(frozen=True)
class IRDiagnostic:
    level:   str    # "error" | "warning" | "info"
    message: str
    node_id: str | None
    code:    str | None
```

### 3.2 SymbolTable

```python
class SymbolTable(Protocol):
    def define(self, name: str, symbol: "Symbol") -> None
    def lookup(self, name: str) -> "Symbol | None"
    def lookup_scoped(self, scope: str, name: str) -> "Symbol | None"
    def enter_scope(self, scope: str) -> None
    def exit_scope(self) -> None
    def all_symbols(self) -> Sequence["Symbol"]

@dataclass(frozen=True)
class Symbol:
    name:       str
    kind:       str      # "type" | "function" | "variable" | "constant"
    type_name:  str | None
    scope:      str
    defined_at: str | None
```

### 3.3 Frontend

```python
class Frontend(Protocol[I]):
    """Translates input type I to an IRProgram."""
    def compile(self, input: I) -> IRProgram  # O(|input|)
    def input_type(self) -> type
    def ir_version(self) -> str
```

### 3.4 Pass

```python
class Pass(Protocol):
    """Transforms IRProgram → IRProgram. May add diagnostics."""
    name:     str
    order:    int       # Lower runs first
    required: bool      # If True, failure is fatal

    def run(self, program: IRProgram) -> IRProgram
    def complexity(self) -> str     # e.g. "O(N)"

# Built-in passes:
# ValidationPass     — checks IR integrity, required=True
# DeadNodePass       — removes unreachable IR nodes
# InliningPass       — inlines single-use nodes
# NormalizationPass  — canonical form for deterministic backends
# OptimizationPass   — target-specific optimizations
```

### 3.5 PassPipeline

```python
class PassPipeline(Protocol):
    def add(self, pass_: Pass) -> None
    def remove(self, name: str) -> None
    def reorder(self, name: str, new_order: int) -> None
    def run(self, program: IRProgram) -> IRProgram
    def passes(self) -> Sequence[Pass]          # Sorted by order

class PassPipelineBuilder(Protocol):
    def with_pass(self, pass_: Pass) -> "PassPipelineBuilder": ...
    def without(self, name: str) -> "PassPipelineBuilder": ...
    def build(self) -> PassPipeline: ...
```

### 3.6 Backend

```python
class Backend(Protocol):
    """Translates IRProgram to output of type O."""
    name:   str
    format: str    # "python" | "markdown" | "json" | "jsonl" | "html" | ...

    def emit(self, program: IRProgram,
             output: "OutputSink") -> CompilerBackendResult
    def can_emit(self, node_kind: str) -> bool
    def supported_ir_versions(self) -> Sequence[str]

class OutputSink(Protocol):
    """Abstraction over output destination."""
    def write(self, content: str) -> None
    def write_file(self, filename: str, content: str) -> None
    def write_bytes(self, filename: str, data: bytes) -> None
    def flush(self) -> None

@dataclass
class CompilerBackendResult:
    backend:      str
    files_written: Sequence[str]
    token_count:  int | None
    elapsed_ms:   float
    warnings:     Sequence[str]
```

### 3.7 Compiler

```python
class Compiler(Protocol[I]):
    """Orchestrates Frontend → PassPipeline → Backends."""
    def compile(
        self,
        input: I,
        backends: Sequence[str],
        output: OutputSink,
        options: CompilerOptions | None = None,
    ) -> CompilerResult

    def add_backend(self, backend: Backend) -> None
    def remove_backend(self, name: str) -> None
    def add_pass(self, pass_: Pass) -> None
    def explain(self, input: I) -> CompilerPlan  # Dry-run plan

@dataclass(frozen=True)
class CompilerOptions:
    optimize:      bool = True
    max_warnings:  int = 100
    fail_on_warn:  bool = False
    ir_dump:       bool = False     # Emit IR for debugging

@dataclass
class CompilerResult:
    ok:               bool
    backends_run:     Sequence[str]
    passes_run:       Sequence[str]
    diagnostics:      Sequence[IRDiagnostic]
    elapsed_ms:       float
    ir_node_count:    int

@dataclass
class CompilerPlan:
    frontend:  str
    passes:    Sequence[str]
    backends:  Sequence[str]
    estimated_nodes: int
```

---

## 4. Contracts

| ID | Contract |
|----|----------|
| C-CMP-001 | Frontend must not mutate its input. |
| C-CMP-002 | Passes run in `order` sequence. A required Pass that fails halts the pipeline. |
| C-CMP-003 | `CompilerResult.ok` is False if any `IRDiagnostic` has level="error". |
| C-CMP-004 | Backend `emit` is idempotent: calling twice with same IR produces same output. |
| C-CMP-005 | `SymbolTable` is scoped: `lookup` searches current scope then parent scopes. |
| C-CMP-006 | `OutputSink.write_file` with same filename twice overwrites the first. |
| C-CMP-007 | `PassPipeline.run` returns a new `IRProgram`; original is not mutated. |
| C-CMP-008 | `explain()` (dry-run) does not write to the `OutputSink`. |
| C-CMP-009 | `DeadNodePass` does not remove any node reachable from IR root. |
| C-CMP-010 | IR version must be checked by Backend before emitting. |

---

## 5. Complexity Table

| Operation | Complexity | Notes |
|-----------|-----------|-------|
| Frontend.compile | O(|input|) | Linear in input size |
| ValidationPass | O(N) | N = IR nodes |
| DeadNodePass | O(N + E) | Graph reachability |
| OptimizationPass | O(N · P) | P = pattern rules |
| Backend.emit | O(N) | One pass over IR |
| PassPipeline.run (k passes) | O(k · N) | Sequential |
| SymbolTable.lookup | O(D) | D = scope depth |

---

## 6. Dependencies

- Parser SDK (KA-SDK-007) — AST input for language compilers
- Collections SDK (KA-SDK-002) — IRNode children, symbol table entries
- Algorithms SDK (KA-SDK-003) — DAG reachability for dead-node elimination

---

## 7. Extension Points

```python
class FrontendPlugin(Protocol[I]):
    """Registers a new Frontend for a new input type."""
    def input_type_name(self) -> str: ...
    def create_frontend(self) -> Frontend[I]: ...

class BackendPlugin(Protocol):
    """Registers a new Backend for a new output format."""
    def format_name(self) -> str: ...
    def create_backend(self, config: dict) -> Backend: ...

class PassPlugin(Protocol):
    """Registers a custom IR transform pass."""
    def pass_name(self) -> str: ...
    def create_pass(self, config: dict) -> Pass: ...

class IRValidator(Protocol):
    """Custom IR invariant checker (used by ValidationPass)."""
    def validate(self, program: IRProgram) -> Sequence[IRDiagnostic]: ...
```

---

## 8. Verification

| Check | Criterion |
|-------|-----------|
| V-CMP-001 | Frontend on valid input: `IRProgram.diagnostics` empty |
| V-CMP-002 | Passes run in ascending `order` |
| V-CMP-003 | Required pass failure → `CompilerResult.ok == False` |
| V-CMP-004 | Backend `emit` twice produces identical output |
| V-CMP-005 | `explain()` does not write to OutputSink |
| V-CMP-006 | `SymbolTable.lookup` after `enter_scope("X")` finds symbols defined in X |
| V-CMP-007 | `DeadNodePass` removes only unreachable nodes |
| V-CMP-008 | `PassPipeline.run` returns new program; original unchanged |
| V-CMP-009 | Backend rejects IR version it does not support |
| V-CMP-010 | `CompilerResult.ok == True` only if no error-level diagnostics |
