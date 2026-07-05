---
knowledge_id: KA-PLAT-001
version: "1.0.0"
status: approved
owner: Chief Knowledge Architect
phase: 2.0D.1
created: 2026-06-29
review_date: 2026-12-29
canonical: true
domain: DOM-KNOWLEDGE
capability: migration-platform
type: index
---

# Knowledge Migration Platform

> A reusable engine for transforming any repository into the Knowledge Architecture
> defined by Phase 2.0D.0 and Phase 2.0D.0.5.

**Phase:** 2.0D.1 | **Owner:** Chief Knowledge Architect | **Status:** Approved

---

## The 12 Specifications

| # | Document | Description |
|---|----------|-------------|
| 01 | [MIGRATION-ENGINE-ARCHITECTURE.md](01-MIGRATION-ENGINE-ARCHITECTURE.md) | Overall engine design — components, pipeline, data flow |
| 02 | [REPOSITORY-SCANNER.md](02-REPOSITORY-SCANNER.md) | File discovery, format detection, inventory generation |
| 03 | [DOCUMENT-CLASSIFIER.md](03-DOCUMENT-CLASSIFIER.md) | Automatic document type classification with confidence scoring |
| 04 | [KNOWLEDGE-SPLITTER.md](04-KNOWLEDGE-SPLITTER.md) | Oversized document detection and section-level splitting |
| 05 | [METADATA-GENERATOR.md](05-METADATA-GENERATOR.md) | YAML frontmatter generation per KA-SPEC-001 through 020 |
| 06 | [RELATIONSHIP-RESOLVER.md](06-RELATIONSHIP-RESOLVER.md) | Automatic dependency and relationship discovery |
| 07 | [CANONICAL-RESOLVER.md](07-CANONICAL-RESOLVER.md) | Duplicate, orphan, and dead document detection |
| 08 | [INDEX-BUILDER.md](08-INDEX-BUILDER.md) | Repository, capability, and knowledge graph index generation |
| 09 | [HEALTH-ANALYZER.md](09-HEALTH-ANALYZER.md) | 6-dimension health scoring and knowledge debt calculation |
| 10 | [MIGRATION-PLANNER.md](10-MIGRATION-PLANNER.md) | Execution order, risk analysis, rollback planning |
| 11 | [DESKTOP-PLANNING.md](11-DESKTOP-PLANNING.md) | Future dashboard designs (architecture only, no implementation) |
| 12 | [VERIFICATION.md](12-VERIFICATION.md) | Migration reports, inventory, health, canonical, relationship |

---

## Implementation

All components are implemented in `tools/migration/`.

```
tools/
├── migrate.py                     # CLI entry point
├── requirements.txt
└── migration/
    ├── engine.py                  # KnowledgeMigrationEngine (orchestrator)
    ├── models.py                  # All dataclasses and enums
    ├── scanner.py                 # KnowledgeScanner
    ├── classifier.py              # KnowledgeClassifier
    ├── splitter.py                # KnowledgeSplitter
    ├── metadata_generator.py      # MetadataGenerator
    ├── relationship_resolver.py   # RelationshipResolver
    ├── canonical_resolver.py      # CanonicalResolver
    ├── index_builder.py           # IndexBuilder
    ├── health_analyzer.py         # HealthAnalyzer
    ├── migration_planner.py       # MigrationPlanner
    ├── rollback_manager.py        # RollbackManager
    └── reporter.py                # KnowledgeReporter
```

---

## Quick Start

```bash
# Scan a repository (dry run)
python tools/migrate.py scan /path/to/repo

# Run full migration plan (dry run)
python tools/migrate.py plan /path/to/repo --output reports/

# Execute migration
python tools/migrate.py migrate /path/to/repo --dry-run
python tools/migrate.py migrate /path/to/repo --apply

# Generate health report
python tools/migrate.py health /path/to/repo --output HEALTH-REPORT.md
```

---

## Design Principles

1. **Product-independent** — no hardcoded paths, no AI Runtime dependencies
2. **Reusable** — works for AI Studio, Content Factory, Mythic Realms, and any future product
3. **Non-destructive** — dry-run mode by default; all mutations require `--apply`
4. **Observable** — every decision is logged and explained
5. **Rollback-safe** — git checkpoints before every phase
