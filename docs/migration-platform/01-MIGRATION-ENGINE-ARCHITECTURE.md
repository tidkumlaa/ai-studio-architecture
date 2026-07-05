---
knowledge_id: KA-PLAT-002
version: "1.0.0"
status: approved
owner: Chief Knowledge Architect
phase: 2.0D.1
created: 2026-06-29
review_date: 2026-12-29
canonical: true
domain: DOM-KNOWLEDGE
capability: migration-platform
type: architecture
implements:
  - KA-ROAD-001
depends_on:
  - id: KA-SPEC-020
    reason: "Runtime architecture that migration tooling must produce artifacts for"
  - id: KA-STD-008
    reason: "34 audit rules that migration must satisfy"
---

# Migration Engine Architecture

## Component Design for the Knowledge Migration Platform

---

## 1. Purpose

The Knowledge Migration Platform (KMP) transforms arbitrary repositories into the
Knowledge Architecture defined by Phase 2.0D.0 and Phase 2.0D.0.5. It operates as
a batch-mode pipeline engine: scan → classify → split → enrich → index → report.

The engine is product-independent. It receives a root path as input and produces
a fully-structured knowledge repository as output. It knows nothing about AI Studio,
Content Factory, or any specific product.

---

## 2. Component Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                   KnowledgeMigrationEngine                       │
│                       (Orchestrator)                             │
│                                                                  │
│  ┌──────────────┐   ┌───────────────┐   ┌──────────────────┐   │
│  │  Knowledge   │   │  Knowledge    │   │   Knowledge      │   │
│  │  Scanner     │──▶│  Classifier   │──▶│   Splitter       │   │
│  └──────────────┘   └───────────────┘   └──────────────────┘   │
│         │                                         │              │
│         │           ┌───────────────┐             │              │
│         │           │   Metadata    │◀────────────┘              │
│         └──────────▶│   Generator   │                            │
│                     └───────────────┘                            │
│                             │                                    │
│         ┌───────────────────┼────────────────────┐              │
│         ▼                   ▼                    ▼              │
│  ┌──────────────┐   ┌───────────────┐   ┌──────────────────┐   │
│  │ Relationship │   │   Canonical   │   │  Index Builder   │   │
│  │  Resolver    │   │   Resolver    │   │                  │   │
│  └──────────────┘   └───────────────┘   └──────────────────┘   │
│         │                   │                    │              │
│         └───────────────────┼────────────────────┘              │
│                             ▼                                    │
│                    ┌────────────────┐                            │
│                    │ Health Analyzer│                            │
│                    └────────────────┘                            │
│                             │                                    │
│              ┌──────────────┼──────────────┐                    │
│              ▼              ▼              ▼                    │
│      ┌────────────┐ ┌──────────────┐ ┌──────────────┐         │
│      │ Migration  │ │   Rollback   │ │  Knowledge   │         │
│      │  Planner   │ │   Manager    │ │  Reporter    │         │
│      └────────────┘ └──────────────┘ └──────────────┘         │
└──────────────────────────────────────────────────────────────────┘
```

---

## 3. Pipeline Stages

The engine executes in nine sequential stages. Each stage is independently
invocable for partial runs.

| Stage | Component | Input | Output | Mutations |
|-------|-----------|-------|--------|-----------|
| 1 | KnowledgeScanner | File system | RepositoryInventory | None |
| 2 | KnowledgeClassifier | Inventory | ClassifiedInventory | None |
| 3 | KnowledgeSplitter | ClassifiedInventory | SplitPlan list | Files (apply only) |
| 4 | MetadataGenerator | ClassifiedInventory | MetadataMap | Files (apply only) |
| 5 | RelationshipResolver | Inventory + MetadataMap | Relationship list | None |
| 6 | CanonicalResolver | Inventory + ClassifiedInventory | CanonicalMap | None |
| 7 | IndexBuilder | All above | BuiltIndexes | YAML files (apply only) |
| 8 | HealthAnalyzer | All above | HealthReport | None |
| 9 | MigrationPlanner | Inventory + HealthReport | MigrationPlan | None |

---

## 4. Configuration Model

Every component is configured via a typed config dataclass. No global state.
No environment variables. All paths are injected, never hardcoded.

```
EngineConfig
├── ScanConfig       — include/exclude patterns, max depth
├── ClassifierConfig — confidence threshold, custom rules
├── SplitterConfig   — line limit (default 500), heading levels
├── MetadataConfig   — default owner, default status, ID registry path
├── RelationshipConfig — link patterns, confidence thresholds
├── CanonicalConfig  — similarity threshold, orphan detection rules
├── IndexConfig      — output path, index types to build
├── HealthConfig     — scoring weights (match KA-STD-006)
├── PlannerConfig    — phase definitions, risk thresholds
└── ReporterConfig   — output path, report formats
```

---

## 5. Dry-Run Guarantee

Every mutation in the pipeline is guarded by a `dry_run: bool` flag.

When `dry_run=True` (default):
- No files are created, modified, or deleted
- All planned mutations are logged to stdout
- Reports describe what would happen

When `dry_run=False` (explicit `--apply`):
- The RollbackManager creates a git checkpoint first
- Mutations execute in phase order
- Each phase is tagged in git before proceeding

---

## 6. Error Handling

All components collect errors before reporting (fail-slow, not fail-fast).

```
KnowledgeMigrationError (base)
├── ScanError
├── ClassificationError
├── SplitError
├── MetadataError
├── RelationshipError
├── IndexBuildError
├── HealthAnalysisError
└── RollbackError
```

The engine collects all errors, continues processing, and reports at the end.
A phase that produces errors does not block subsequent read-only phases.

---

## 7. Interfaces

### Engine Entry Point

```python
engine = KnowledgeMigrationEngine(config=EngineConfig(...))
result = engine.migrate(root_path=Path("/repo"), dry_run=True)
```

### CLI

```bash
python migrate.py scan   <root>                  # Stage 1 only
python migrate.py plan   <root> [--output PATH]  # Stages 1-9, read-only
python migrate.py migrate <root> [--dry-run]     # Full pipeline
python migrate.py migrate <root> --apply         # Full pipeline with mutations
python migrate.py health  <root>                 # Stages 1-8, health report
python migrate.py report  <root> --output PATH   # Generate all reports
```

---

## 8. Reusability Contracts

1. No import from any product module (AI Studio, Content Factory, Mythic Realms)
2. All root paths injected via config — no `os.getcwd()` defaults
3. All output paths injected — no assumptions about output location
4. Classification rules are data-driven — loaded from config, not hardcoded
5. Domain and capability names loaded from `domains.yaml` — not hardcoded

---

## References

- [KA-ROAD-001](../knowledge-architecture/REPOSITORY-REFACTORING-PLAN.md) — The plan this engine executes
- [KA-SPEC-020](../knowledge-core/20-KNOWLEDGE-RUNTIME-ARCHITECTURE.md) — Runtime architecture being bootstrapped
- [KA-STD-008](../knowledge-architecture/REPOSITORY-AUDIT-RULES.md) — Rules the engine enforces
- [02 through 12](.) — Per-component specifications
