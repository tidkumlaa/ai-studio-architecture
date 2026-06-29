---
knowledge_id: KNW-KOS-ARCH-006
title: "KOS Knowledge Compiler"
status: AUTHORITATIVE
phase: "3.0A"
version: "1.0.0"
created: "2026-06-30"
purpose: "Specify the Knowledge Compiler design: inputs, outputs, compilation stages, and artifact types"
canonical_source: "architecture/docs/kos/06-KNOWLEDGE-COMPILER.md"
dependencies:
  - "04-OBJECT-MODEL.md"
  - "05-KNOWLEDGE-GRAPH.md"
related_documents:
  - "07-KNOWLEDGE-REGISTRY.md"
  - "10-EXECUTION-ENGINE.md"
acceptance_criteria:
  - "All 10 output artifact types are specified"
  - "Compilation pipeline has defined stages"
  - "Compiler is idempotent"
verification_checklist:
  - "[ ] All artifact types listed"
  - "[ ] Compilation is deterministic"
  - "[ ] Compiler does not modify Knowledge Objects (read-only input)"
future_extensions:
  - "Incremental compilation (only recompile changed subgraph)"
  - "Parallel compilation for independent subgraphs"
---

# KOS Knowledge Compiler

## Purpose

The Knowledge Compiler transforms Knowledge Objects and the Knowledge Graph into
executable artifacts. It is the mechanism by which **knowledge becomes reality**.

The compiler is:
- **Deterministic:** Same knowledge graph → same artifacts, always
- **Idempotent:** Running twice produces identical output
- **Read-only on inputs:** Never modifies Knowledge Objects
- **Verifiable:** Every output is traceable to its input knowledge_id

---

## Compilation Pipeline

```
Knowledge Objects
        │
        ▼
Stage 1: LOAD
  Load all CANONICAL/APPROVED objects from registry
  Resolve all graph edges
  Validate graph invariants (GI-001 through GI-005)
        │
        ▼
Stage 2: ANALYZE
  Topological sort (dependency order)
  Detect cycles (error if any found)
  Compute compilation order
        │
        ▼
Stage 3: VALIDATE
  Validate all object schemas
  Check GR-001 through GR-010 compliance
  Report any violations (compiler does not proceed if errors)
        │
        ▼
Stage 4: GENERATE
  For each target artifact type:
    Select relevant knowledge objects
    Apply template
    Generate artifact
        │
        ▼
Stage 5: ANNOTATE
  Embed knowledge_id provenance in every output file
  Write compilation manifest
        │
        ▼
Stage 6: VERIFY
  Verify generated artifacts import/parse correctly
  Run smoke tests on generated code
  Compute artifact checksums
        │
        ▼
Compilation Output
```

---

## Output Artifact Types

### 1. Platform Code
Generated Python modules and class stubs from `Module`, `Service`, `Algorithm` objects.

```python
# Generated from KNW-AI-MOD-quota_manager
# GENERATED — do not edit manually. Source: KNW-AI-MOD-quota_manager

from __future__ import annotations
from platform.kernel.di import inject
from platform.runtimes.resource_runtime.quota_engine import QuotaEngine

class QuotaManager:
    """Generated from KNW-AI-MOD-quota_manager v1.0.0"""
    ...
```

### 2. Runtime Configuration
Generated `runtime.yaml` and `module.yaml` files from `Runtime` and `Module` objects.

### 3. SDK Facades
Generated `platform.sdk.*` functions from `Capability` and `Service` objects.
Every public capability becomes a typed SDK function.

### 4. Test Suites
Generated pytest test files from `Test` and `Benchmark` objects.
Tests linked to `Requirement` objects via `@pytest.mark.kos(knowledge_id)`.

### 5. Desktop Components
Generated PySide6 panel stubs from `Product` + `Dashboard` knowledge objects.

### 6. Documentation
Generated Markdown documentation from all object `description` fields + graph traversal.
Every module's README is generated, not hand-written.

### 7. Configuration
Generated `.env`, `pyproject.toml`, `runtime.yaml`, `capability.yaml` files
from `Platform`, `Package`, `Runtime`, `Capability` objects.

### 8. Validation Rules
Generated CI check scripts from `GoldenRule` objects (GR-001 through GR-010).
Rules are code, not prose.

### 9. Reports
Generated compliance reports: coverage, quality scores, traceability matrices,
dependency graphs. Consumed by the Dashboard.

### 10. Execution Context
Generated `ExecutionContext` object containing the complete compiled platform state.
Used by runtimes at boot to verify they match their knowledge specification.

---

## Compiler Interface

```python
class KnowledgeCompiler:
    def __init__(
        self,
        registry: KnowledgeRegistry,
        graph: KnowledgeGraph,
        output_dir: str,
    ) -> None: ...

    def compile(
        self,
        targets: list[ArtifactType] | None = None,  # None = all
        dry_run: bool = False,
    ) -> CompilationResult: ...

    def validate(self) -> list[CompilationError]: ...
    # Run stages 1–3 only; return errors without generating

    def incremental(self, changed_ids: list[str]) -> CompilationResult: ...
    # Recompile only artifacts affected by changed knowledge objects
```

---

## CompilationResult

```python
@dataclass(frozen=True)
class CompilationResult:
    success: bool
    artifacts_generated: int
    artifacts_by_type: dict[ArtifactType, int]
    errors: list[CompilationError]
    warnings: list[CompilationWarning]
    duration_ms: int
    manifest_path: str
    checksum: str
```

---

## Compilation Manifest

Every compilation produces a manifest:

```yaml
# .kos/compilation-manifest.yaml
# GENERATED — do not edit manually

schema_version: "1.0"
compiled_at: "2026-06-30T00:00:00Z"
knowledge_graph_checksum: "sha256:abc123..."
artifacts:
  - type: platform_code
    count: 29
    checksum: "sha256:def456..."
  - type: test_suites
    count: 12
    checksum: "sha256:ghi789..."
  - type: documentation
    count: 37
    checksum: "sha256:jkl012..."
  # ...
errors: 0
warnings: 2
```

---

## Compiler CLI

```bash
# Compile all artifacts
platform-kos compile --all

# Compile only tests
platform-kos compile --target tests

# Dry run (validate only, no output)
platform-kos compile --dry-run

# Incremental compile after knowledge changes
platform-kos compile --incremental --changed KNW-AI-MOD-quota_manager

# Verify compilation manifest
platform-kos compile --verify-manifest
```

---

## Template System

Each artifact type has a **Jinja2 template** that transforms knowledge objects into output:

```
platform/tools/kos/templates/
  platform_code/
    module.py.j2       — generates Python module skeleton
    service.py.j2      — generates Service ABC implementation
    algorithm.py.j2    — generates Algorithm class
  test_suites/
    unit_test.py.j2    — generates pytest unit test file
    arch_test.py.j2    — generates architecture compliance test
  documentation/
    module_readme.md.j2
    runtime_readme.md.j2
  configuration/
    module_yaml.j2
    runtime_yaml.j2
    pyproject_toml.j2
```

Templates are themselves Knowledge Objects (type `CONFIGURATION`) and versioned.
