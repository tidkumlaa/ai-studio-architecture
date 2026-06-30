---
knowledge_id: KA-SPEC-020
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
depends_on:
  - id: KA-SPEC-009
    reason: "Runtime includes the Knowledge Compiler"
  - id: KA-SPEC-010
    reason: "Runtime includes the Knowledge Index Engine"
  - id: KA-SPEC-007
    reason: "Runtime includes the KQL Query Engine"
  - id: KA-SPEC-015
    reason: "Runtime is implemented per these CS standards"
  - id: KA-SPEC-019
    reason: "Runtime must meet the performance budget"
---

# Knowledge Runtime Architecture

## The Complete Runtime Design for the Knowledge System

---

## 1. Purpose

The Knowledge Runtime Architecture (KRA) synthesizes all preceding specifications into a coherent, deployable system design. It defines the components, their interactions, the data flows, deployment model, extension points, and operational model.

The KRA is the target architecture for Phase 2.0D.1 (tooling) and Phase 2.0D.2 (Documentation Intelligence Platform). It does not include implementation code — it is the blueprint.

---

## 2. Runtime Components

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Knowledge Runtime                               │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                      Core Engine Layer                       │   │
│  │                                                             │   │
│  │  ┌──────────────────┐    ┌──────────────────────────────┐  │   │
│  │  │  Knowledge Index │    │    Knowledge Graph Engine    │  │   │
│  │  │  Engine (KIE)    │    │    (Property Graph + SCC)    │  │   │
│  │  └──────────────────┘    └──────────────────────────────┘  │   │
│  │                                                             │   │
│  │  ┌──────────────────┐    ┌──────────────────────────────┐  │   │
│  │  │  KQL Query       │    │    Knowledge Compiler        │  │   │
│  │  │  Engine          │    │    (Multi-target)            │  │   │
│  │  └──────────────────┘    └──────────────────────────────┘  │   │
│  │                                                             │   │
│  │  ┌──────────────────┐    ┌──────────────────────────────┐  │   │
│  │  │  Schema          │    │    Audit Engine              │  │   │
│  │  │  Validator       │    │    (34 rules)                │  │   │
│  │  └──────────────────┘    └──────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                   Intelligence Layer                         │   │
│  │                                                             │   │
│  │  ┌──────────────────┐    ┌──────────────────────────────┐  │   │
│  │  │  Health Scorer   │    │    Traceability Engine       │  │   │
│  │  │                  │    │                              │  │   │
│  │  └──────────────────┘    └──────────────────────────────┘  │   │
│  │                                                             │   │
│  │  ┌──────────────────┐    ┌──────────────────────────────┐  │   │
│  │  │  Coverage        │    │    Evidence Validator        │  │   │
│  │  │  Analyzer        │    │                              │  │   │
│  │  └──────────────────┘    └──────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                   Storage Layer                              │   │
│  │                                                             │   │
│  │  Registry     CapabilityIndex   Graph      HealthIndex      │   │
│  │  (HashMap)    (HashMap²)        (Adj List) (SortedSet)      │   │
│  │                                                             │   │
│  │  KNOWLEDGE-REGISTRY.yaml    graph.json    health-report.yaml│   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                   Interface Layer                            │   │
│  │                                                             │   │
│  │  CLI Tools    Pre-commit Hook    REST API (Phase 2.0D.2)    │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Component Specifications

### 3.1 Knowledge Index Engine (KIE)

**Specification:** [10-KNOWLEDGE-INDEX-ENGINE.md](10-KNOWLEDGE-INDEX-ENGINE.md)

**Responsibilities:**
- Parse all Markdown + YAML frontmatter files
- Build and maintain all seven indexes
- Detect changes and perform incremental updates
- Serialize indexes to disk

**Interfaces:**
```
Input:  File system (architecture/**/*.md, **/index.yaml)
Output: In-memory indexes, disk-serialized files
API:    KnowledgeIndexEngine class (read-only query interface)
```

### 3.2 Knowledge Graph Engine

**Specification:** [08-KNOWLEDGE-GRAPH-MODEL.md](08-KNOWLEDGE-GRAPH-MODEL.md)

**Responsibilities:**
- Maintain the property graph
- Execute graph traversal algorithms (DFS, BFS, SCC)
- Detect cycles, orphans, and connected components
- Serve as the execution backend for TRAVERSE queries

### 3.3 KQL Query Engine

**Specification:** [07-KNOWLEDGE-QUERY-LANGUAGE.md](07-KNOWLEDGE-QUERY-LANGUAGE.md)

**Responsibilities:**
- Parse KQL text into an AST
- Validate and optimize queries
- Execute against the index store and graph engine
- Return typed result sets

### 3.4 Knowledge Compiler

**Specification:** [09-KNOWLEDGE-COMPILER-SPECIFICATION.md](09-KNOWLEDGE-COMPILER-SPECIFICATION.md)

**Responsibilities:**
- Parse all source documents into the IR
- Validate the IR against the schema
- Dispatch to output backends (Markdown, HTML, PromptPack, etc.)
- Write compiled artifacts to output directories

### 3.5 Schema Validator

**Specification:** [05-KNOWLEDGE-SCHEMA.md](05-KNOWLEDGE-SCHEMA.md)

**Responsibilities:**
- Validate individual KnowledgeObject frontmatter against JSON Schema
- Perform cross-reference validation in audit mode
- Produce typed `Diagnostic` objects

### 3.6 Audit Engine

**Specification:** [KA-STD-008 / KA-STD-008B](../knowledge-architecture/)

**Responsibilities:**
- Run all 34 audit rules
- Collect and categorize diagnostics
- Generate `audit-report.yaml` and `HEALTH-REPORT.md`
- Exit with code 1 if any error-severity rules are violated

### 3.7 Health Scorer

**Specification:** [KA-STD-006](../knowledge-architecture/KNOWLEDGE-HEALTH-METRICS.md), [ALG-007](17-ALGORITHMS-CATALOG.md)

**Responsibilities:**
- Compute health scores for all KnowledgeObjects
- Aggregate capability and domain health scores
- Maintain the `HealthIndex`

### 3.8 Traceability Engine

**Specification:** [11-KNOWLEDGE-TRACEABILITY-MODEL.md](11-KNOWLEDGE-TRACEABILITY-MODEL.md)

**Responsibilities:**
- Traverse the traceability chain (requirement → evidence)
- Detect traceability gaps
- Generate traceability matrix reports

### 3.9 Coverage Analyzer

**Specification:** [13-KNOWLEDGE-COVERAGE-MODEL.md](13-KNOWLEDGE-COVERAGE-MODEL.md)

**Responsibilities:**
- Compute coverage dimensions (architecture/implementation/testing)
- Override declared values with evidence-derived values
- Detect coverage gaps
- Generate coverage reports

### 3.10 Evidence Validator

**Specification:** [12-KNOWLEDGE-EVIDENCE-MODEL.md](12-KNOWLEDGE-EVIDENCE-MODEL.md)

**Responsibilities:**
- Check existence of all referenced evidence files
- Track evidence state and expiry
- Compute confidence levels from evidence
- Generate the `evidence-registry.yaml`

---

## 4. Component Interaction Model

```
File System
    │
    ▼ (read)
KIE ──────────────────────────────────────────────────────────┐
    │ Parsed objects                                           │
    ▼                                                          │
Schema Validator ──── diagnostics ──▶ Audit Engine            │
    │ valid objects                                            │
    ▼                                                          │
KnowledgeGraph Engine ─────────────────────────────────────┐  │
    │ enriched graph                                        │  │
    ▼                                                       │  │
Health Scorer ────────────────────────────────────────────┐ │  │
    │                                                      │ │  │
    ├── Evidence Validator ────────────────────────────┐   │ │  │
    │                                                  │   │ │  │
    ├── Coverage Analyzer ─────────────────────────────┤   │ │  │
    │                                                  │   │ │  │
    └── Traceability Engine ───────────────────────────┘   │ │  │
                                                           ▼ │  │
                                                     HealthIndex │
                                                           ▼ │  │
Indexes (Registry, Domain, Capability, Health) ────────────┘ │  │
    │                                                         │  │
    ├── KQL Query Engine ◀── Query requests                  │  │
    │                                                         │  │
    └── Knowledge Compiler ◀── Compile requests              │  │
         │                                                    │  │
         ├── Markdown Target ──────────────────▶ File System  │  │
         ├── HTML Target ──────────────────────▶ dist/html/   │  │
         ├── PromptPack Target ────────────────▶ dist/prompts/│  │
         ├── LLMMemory Target ─────────────────▶ dist/memory/ │  │
         └── Dataset Target ─────────────────── ▶ dist/data/  │  │
                                                              │  │
Audit Report ──────────────────────────────────────────────── ┘  │
    ▼                                                             │
Health Report ─────────────────────────────────────────────────── ┘
    ▼
File System (audit-report.yaml, HEALTH-REPORT.md, graph.json, index.yaml)
```

---

## 5. Deployment Model

### 5.1 Phase 2.0D.1 — Local Tools

In Phase 2.0D.1, the runtime is deployed as a set of local CLI tools:

```
tools/
├── audit.py               # Audit engine CLI
├── generate_indexes.py    # KIE CLI (full build)
├── ka_compile.py          # Compiler CLI
├── ka_query.py            # KQL CLI
├── health_report.py       # Health report generator
└── requirements.txt       # Python dependencies
```

**Integration with git:**
```
.git/hooks/pre-commit:
  python tools/audit.py --ci
  python tools/generate_indexes.py --incremental
```

### 5.2 Phase 2.0D.2 — Background Service

In Phase 2.0D.2, the runtime also runs as a background service that powers the Documentation Intelligence Platform:

```
KnowledgeRuntimeService (daemon)
  ├── Watches file system for changes
  ├── Runs incremental updates (< 200ms per change)
  ├── Exposes REST API
  │   ├── GET /api/v1/objects/{id}
  │   ├── GET /api/v1/capabilities/{name}
  │   ├── POST /api/v1/query (KQL)
  │   └── GET /api/v1/health
  └── Exposes WebSocket for real-time health updates
```

---

## 6. Extension Points

The runtime is designed for extension at the following points:

| Extension Point | How | Phase |
|----------------|-----|-------|
| New output target | Implement `CompilerBackend` interface | 2.0D.2+ |
| New audit rule | Add `AuditRule` to audit engine config | 2.0D.1+ |
| New relationship type | Add to schema enum + type compatibility matrix | 2.0D.1+ |
| New knowledge type | Via type extension protocol (KA-SPEC-002) | 2.0D.1+ |
| New index type | Implement `KnowledgeIndex` interface | 2.0E+ |
| External data source | Implement `KnowledgeSource` interface | 2.0E+ |
| AI-powered analysis | Add `IntelligencePlugin` to KQL engine | 2.0D.2+ |

---

## 7. Operational Model

### 7.1 Development Mode

```bash
ka-watch             # Start file watcher + incremental updates
ka-query --repl      # Interactive KQL REPL
ka-health --live     # Live health dashboard (refreshes on change)
```

### 7.2 CI Mode

```bash
ka-compile --ci      # Fail on schema errors
ka-audit --ci        # Fail on rule violations
ka-report            # Generate health and coverage reports
```

### 7.3 Release Mode

```bash
ka-compile --targets html,dataset,promptpack  # Full compile
ka-report --full                              # Comprehensive reports
ka-snapshot --phase 2.0D.1                    # Create phase snapshot
```

---

## 8. Runtime Contract Summary

The Knowledge Runtime provides the following guarantees:

| Contract | Guarantee |
|---------|-----------|
| **Identity** | Every KnowledgeObject has exactly one ID; IDs are permanent |
| **Canonicality** | At most one approved canonical per (capability, type) |
| **Integrity** | All references in indexes point to existing objects |
| **Freshness** | Indexes are regenerated within 200ms of any source change (watch mode) |
| **Correctness** | Health scores are computed from declared metadata, never from prose |
| **Completeness** | All approved documents appear in at least one index |
| **Traceability** | All relationship chains are traversable without file system access |
| **Safety** | KQL is read-only; no mutations via query interface |

---

## References

- [01 through 19](.) — All preceding specifications that this architecture synthesizes
- [KA-VIS-001](../knowledge-architecture/KNOWLEDGE-ARCHITECTURE-VISION.md) — Vision that this runtime realizes
- [KA-ROAD-001](../knowledge-architecture/REPOSITORY-REFACTORING-PLAN.md) — Phase 2.0D.1 implementation plan
