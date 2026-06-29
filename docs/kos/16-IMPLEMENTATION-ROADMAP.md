---
knowledge_id: KNW-KOS-ARCH-016
title: "KOS Implementation Roadmap"
status: AUTHORITATIVE
phase: "3.0A"
version: "1.0.0"
created: "2026-06-30"
purpose: "Define the implementation sequence for KOS from Phase 3.0A through products"
canonical_source: "architecture/docs/kos/16-IMPLEMENTATION-ROADMAP.md"
dependencies:
  - "01-VISION.md"
  - "14-KOS-PLATFORM.md"
  - "15-FUTURE-SYSTEMS.md"
related_documents:
  - "17-ARCHITECTURE-FREEZE.md"
acceptance_criteria:
  - "All phases from 3.0A to products are listed"
  - "Each phase has a clear deliverable and stop condition"
  - "Dependencies between phases are explicit"
verification_checklist:
  - "[ ] All 9 phases listed"
  - "[ ] Each phase has a gate condition"
  - "[ ] No phase overlaps in scope"
future_extensions:
  - "Post-3.0I phases as ecosystem grows"
---

# KOS Implementation Roadmap

## Phase Overview

```
Phase 3.0A — Knowledge Vision (CURRENT)
    Architecture specification complete
    ↓
Phase 3.0B — Knowledge Object Model
    All object types implemented in Python
    ↓
Phase 3.0C — Knowledge Graph
    Graph database + edge management
    ↓
Phase 3.0D — Knowledge Compiler
    Compiler generates all 10 artifact types
    ↓
Phase 3.0E — Knowledge Runtime
    KOS boots as a platform runtime
    ↓
Phase 3.0F — Platform Migration
    All Phase 2.x objects registered in KOS
    ↓
Phase 3.0G — AI Runtime Migration
    All 29 AI modules registered; tests linked to KOS
    ↓
Phase 3.0H — Software Factory Migration
    Agents, tasks, workflows registered in KOS
    ↓
Phase 3.0I — Financial Runtime
    Financial objects, markets, strategies in KOS
    ↓
Products
    All products depend on KOS for specifications
```

---

## Phase 3.0A — Knowledge Vision

**Status:** CURRENT (this document)  
**Deliverable:** 20 architecture documents in `architecture/docs/kos/`  
**Stop Condition:** All 20 documents AUTHORITATIVE, Architecture Freeze FROZEN

**What is NOT done in 3.0A:**
- No Python code
- No knowledge objects created
- No registry built
- No compiler written

---

## Phase 3.0B — Knowledge Object Model

**Gate Condition:** Phase 3.0A Architecture Freeze is FROZEN  
**Deliverable:** Python implementation of all KnowledgeObject types

```
platform/
└── runtimes/
    └── knowledge_runtime/
        ├── objects/
        │   ├── base.py          — KnowledgeObject base class
        │   ├── platform.py      — PlatformObject, RuntimeObject, KernelObject
        │   ├── architecture.py  — ArchitectureObject, DecisionObject, RequirementObject
        │   ├── implementation.py — ModuleObject, ServiceObject, AlgorithmObject
        │   ├── interface.py     — APIObject, CapabilityObject, ProviderObject
        │   ├── quality.py       — TestObject, BenchmarkObject
        │   ├── product.py       — ProductObject, AgentObject, PromptObject
        │   ├── financial.py     — MarketObject, ForexObject
        │   └── __init__.py
        └── lifecycle.py         — KnowledgeLifecycleStatus
```

**Tests:** ≥ 50 unit tests covering all object types and validations  
**Stop Condition:** All objects serializable/deserializable; pydantic validation passes

---

## Phase 3.0C — Knowledge Graph

**Gate Condition:** Phase 3.0B complete  
**Deliverable:** Knowledge Graph engine with storage and traversal

```
platform/
└── runtimes/
    └── knowledge_runtime/
        ├── graph/
        │   ├── engine.py        — KnowledgeGraph implementation
        │   ├── edge.py          — Edge type definitions
        │   ├── traversal.py     — BFS/DFS/shortest-path algorithms
        │   ├── invariants.py    — GI-001 through GI-005 checks
        │   └── storage.py       — YAML-backed storage
        └── query/
            └── engine.py        — KnowledgeQueryEngine
```

**Tests:** ≥ 40 tests covering all edge types, traversal, and invariant checks  
**Stop Condition:** Graph validates, traversal returns correct paths, query engine answers all 8 query types

---

## Phase 3.0D — Knowledge Compiler

**Gate Condition:** Phase 3.0C complete  
**Deliverable:** Knowledge Compiler with all 10 artifact types

```
platform/
└── tools/
    └── kos/
        ├── compiler/
        │   ├── engine.py        — KnowledgeCompiler
        │   ├── stages.py        — Load, Analyze, Validate, Generate, Annotate, Verify
        │   └── artifacts/
        │       ├── platform_code.py
        │       ├── runtime_config.py
        │       ├── sdk_facades.py
        │       ├── test_suites.py
        │       ├── desktop_components.py
        │       ├── documentation.py
        │       ├── configuration.py
        │       ├── validation_rules.py
        │       ├── reports.py
        │       └── execution_context.py
        └── templates/           — Jinja2 templates per artifact type
```

**Tests:** Compiler generates valid Python for at least 5 known modules  
**Stop Condition:** All 10 artifact types generated; generated code passes import check

---

## Phase 3.0E — Knowledge Runtime

**Gate Condition:** Phase 3.0D complete  
**Deliverable:** KOS boots as `runtime:knowledge:v2` with all engines

```
platform/runtimes/knowledge_runtime/
├── runtime.py           — KnowledgeRuntime (boot_order=4, extended)
├── objects/             — (from 3.0B)
├── graph/               — (from 3.0C)
├── query/               — QueryEngine
├── reasoning/           — ReasoningEngine
├── execution/           — ExecutionEngine
├── registry/            — KnowledgeRegistry
├── governance/          — GovernanceEngine
├── quality/             — QualityEngine
├── lifecycle/           — LifecycleManager
└── compiler/            — (from 3.0D, integrated into runtime)
```

**Tests:** ≥ 100 tests; runtime boots cleanly; all engines respond  
**Stop Condition:** `platform-kos validate --all` exits 0

---

## Phase 3.0F — Platform Migration

**Gate Condition:** Phase 3.0E complete  
**Deliverable:** All Phase 2.x platform knowledge registered in KOS

**Work:**
- Register all 37 Phase 2.1D.0 architecture documents as `ArchitectureObject`
- Register all 10 ADRs as `DecisionObject`
- Register all 58 modules as `ModuleObject`
- Register all 66 capabilities as `CapabilityObject`
- Register all 6 services as `ServiceObject`
- Compute initial quality scores for all objects

**Stop Condition:** `platform-kos quality dashboard` shows average health ≥ 0.70 for all domains

---

## Phase 3.0G — AI Runtime Migration

**Gate Condition:** Phase 3.0F complete  
**Deliverable:** All AI runtime artifacts traced to KOS knowledge objects

**Work:**
- Register all 29 AI runtime modules in KOS
- Link all 160+ existing tests to their knowledge objects
- Register all 10 algorithms (EMA, workload detection, etc.) as `AlgorithmObject`
- Annotate all tests with `@pytest.mark.kos(knowledge_id)`
- Run Knowledge Compiler to generate AI runtime docs from KOS

**Stop Condition:** `platform-kos validate --system ai-runtime` exits 0; pytest coverage ≥ 90%

---

## Phase 3.0H — Software Factory Migration

**Gate Condition:** Phase 3.0G complete  
**Deliverable:** All Software Factory agents, tasks, workflows in KOS

**Work:**
- Register all Software Factory modules as `ModuleObject`
- Register all agent definitions as `AgentObject`
- Register all prompt templates as `PromptObject`
- Register all task types as `TaskObject`
- Generate agent documentation from KOS

**Stop Condition:** `platform-kos validate --system software-factory` exits 0

---

## Phase 3.0I — Financial Runtime

**Gate Condition:** Phase 3.0H complete  
**Deliverable:** Financial Runtime designed and specified in KOS before implementation

**Work:**
- Design all financial knowledge object types
- Register market, forex, trading strategy specifications
- Design Financial Runtime architecture as `ArchitectureObject`
- Generate Financial Runtime skeleton from Knowledge Compiler

**Stop Condition:** Financial Runtime architecture CANONICAL; skeleton compiles

---

## Products

**Gate Condition:** Phase 3.0I complete  
**All products rebuilt with KOS as foundation:**
- Mythic Realms game mechanics → KOS objects
- Content Factory pipeline → KOS objects
- Desktop panels → generated from KOS
- API routes → generated from KOS APIObjects

---

## Milestone Summary

| Phase | Key Output | Gate |
|-------|-----------|------|
| 3.0A | Architecture docs (20 files) | Freeze FROZEN |
| 3.0B | KnowledgeObject Python types | All types serialize |
| 3.0C | Knowledge Graph engine | Graph invariants pass |
| 3.0D | Knowledge Compiler (10 artifact types) | Generated code compiles |
| 3.0E | KOS Runtime boots | `platform-kos validate --all` |
| 3.0F | Platform 2.x objects in KOS | Domain health ≥ 0.70 |
| 3.0G | AI Runtime objects in KOS | Test coverage ≥ 90% |
| 3.0H | Software Factory in KOS | Factory CI passes |
| 3.0I | Financial Runtime designed | Architecture CANONICAL |
| Products | All products on KOS | Per-product CI |
