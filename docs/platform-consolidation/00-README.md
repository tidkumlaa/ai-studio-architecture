---
knowledge_id: KNW-PLAT-ARCH-000
title: "Platform Consolidation Architecture — Navigation Guide"
status: AUTHORITATIVE
phase: "2.1D.0"
version: "1.0.0"
created: "2026-06-29"
purpose: "Entry point and navigation guide for the complete Platform Consolidation Architecture package"
canonical_source: "architecture/docs/platform-consolidation/00-README.md"
dependencies: []
related_documents:
  - "01-VISION.md"
  - "35-ARCHITECTURE-FREEZE.md"
  - "index.yaml"
acceptance_criteria:
  - "Every document listed in this guide exists and is valid"
  - "index.yaml is consistent with this README"
  - "All 37 documents are under 400 lines"
verification_checklist:
  - "[ ] All document files present"
  - "[ ] All YAML front-matter valid"
  - "[ ] No broken cross-references"
  - "[ ] index.yaml machine-parseable"
future_extensions:
  - "Auto-generate HTML from this package"
  - "Publish to internal docs portal"
---

# Platform Consolidation Architecture

## What This Package Is

This package contains the **complete architecture specification** for consolidating the AI Studio
Platform into a single canonical repository. It answers every architectural question before a
single line of implementation code is written.

Phase: **2.1D.0 — Architecture Only**
Status: **AUTHORITATIVE**
Implementation phase: **2.1D.1** (not yet started)

---

## Reading Order

### Tier 1 — Start Here (Strategic)
| Doc | Title | Purpose |
|-----|-------|---------|
| 01 | VISION | Why we consolidate; what the end state looks like |
| 02 | OBJECT-MODEL | Every platform object and its specification |
| 03 | DIRECTORY-ARCHITECTURE | Canonical target layout |
| 34 | ARCHITECTURE-DECISIONS | ADRs explaining every major choice |
| 35 | ARCHITECTURE-FREEZE | Frozen commitments; what is locked |

### Tier 2 — Design (Structural)
| Doc | Title | Purpose |
|-----|-------|---------|
| 04 | PACKAGE-MODEL | Python packaging strategy |
| 05 | MODULE-CATALOG | Every module in every runtime |
| 06 | RUNTIME-MODEL | Six runtimes and their contracts |
| 07 | KERNEL-MODEL | Platform kernel specification |
| 08 | SDK-MODEL | Platform SDK specification |
| 09 | SERVICE-MODEL | Service layer specification |
| 10 | DEPENDENCY-RULES | Allowed and forbidden dependencies |
| 11 | LAYER-RULES | Layer boundaries and flow directions |
| 12 | IMPORT-RULES | Import conventions and rewrite rules |

### Tier 3 — Registries and Metadata
| Doc | Title | Purpose |
|-----|-------|---------|
| 13 | METADATA-STANDARD | YAML schemas for module/runtime/service |
| 14 | PLATFORM-MANIFEST | Top-level manifest structure |
| 15 | CAPABILITY-REGISTRY | All platform capabilities |
| 16 | SERVICE-REGISTRY | All registered services |
| 17 | MODULE-REGISTRY | All registered modules |

### Tier 4 — Behaviour (Dynamic)
| Doc | Title | Purpose |
|-----|-------|---------|
| 18 | EVENT-MODEL | All platform events and payloads |
| 19 | STATE-MACHINES | Lifecycle state machines |
| 20 | DATA-STRUCTURES | Core data structures with complexity |
| 21 | ALGORITHMS | Core algorithms with complexity |

### Tier 5 — Migration Engine
| Doc | Title | Purpose |
|-----|-------|---------|
| 22 | MIGRATION-ENGINE | Migration engine design |
| 23 | IMPORT-REWRITE | Import rewrite engine |
| 24 | ROLLBACK-ENGINE | Rollback and recovery design |
| 25 | VERIFICATION | Verification engine design |
| 26 | TEST-STRATEGY | Test strategy across all layers |
| 27 | PERFORMANCE-BUDGET | Performance targets and limits |

### Tier 6 — Interfaces
| Doc | Title | Purpose |
|-----|-------|---------|
| 28 | DASHBOARD | Desktop dashboard panel design |
| 29 | CLI | Migration and verification CLI |
| 30 | API | REST API for architecture operations |
| 31 | KNOWLEDGE-INTEGRATION | Knowledge Runtime integration |
| 32 | RUNTIME-INTEGRATION | Cross-runtime integration rules |

### Tier 7 — Governance
| Doc | Title | Purpose |
|-----|-------|---------|
| 33 | PLATFORM-GOVERNANCE | Governance model and ownership |
| 34 | ARCHITECTURE-DECISIONS | Full ADR log |
| 35 | ARCHITECTURE-FREEZE | What is frozen and why |

---

## Document Conventions

### Every document contains:
- YAML front-matter (knowledge_id, title, status, dependencies, acceptance_criteria)
- Purpose statement
- Canonical definitions (not prose — structured data)
- Cross-references using document numbers
- Acceptance criteria (testable statements)
- Verification checklist (actionable checkboxes)
- Future extensions section

### Status values:
| Status | Meaning |
|--------|---------|
| DRAFT | Under construction — do not rely on |
| REVIEW | Ready for architecture review |
| AUTHORITATIVE | Approved — implementation may proceed against this |
| FROZEN | Locked — changes require ADR |
| SUPERSEDED | Replaced by a newer document |

### Knowledge ID format:
```
KNW-PLAT-ARCH-NNN
```
Where NNN is the document number (000–035).

---

## Critical Rules

1. **Architecture First.** No implementation until 35-ARCHITECTURE-FREEZE.md is signed.
2. **No Product Code.** Products depend on Platform; Platform never depends on Products.
3. **Every Object Specified.** Nothing left to interpretation during implementation.
4. **Under 400 Lines.** Every document. No exceptions.
5. **Cross-Reference Everything.** No document is an island.

---

## Package Files

```
architecture/docs/platform-consolidation/
├── 00-README.md          ← you are here
├── 01-VISION.md
├── 02-OBJECT-MODEL.md
├── 03-DIRECTORY-ARCHITECTURE.md
├── 04-PACKAGE-MODEL.md
├── 05-MODULE-CATALOG.md
├── 06-RUNTIME-MODEL.md
├── 07-KERNEL-MODEL.md
├── 08-SDK-MODEL.md
├── 09-SERVICE-MODEL.md
├── 10-DEPENDENCY-RULES.md
├── 11-LAYER-RULES.md
├── 12-IMPORT-RULES.md
├── 13-METADATA-STANDARD.md
├── 14-PLATFORM-MANIFEST.md
├── 15-CAPABILITY-REGISTRY.md
├── 16-SERVICE-REGISTRY.md
├── 17-MODULE-REGISTRY.md
├── 18-EVENT-MODEL.md
├── 19-STATE-MACHINES.md
├── 20-DATA-STRUCTURES.md
├── 21-ALGORITHMS.md
├── 22-MIGRATION-ENGINE.md
├── 23-IMPORT-REWRITE.md
├── 24-ROLLBACK-ENGINE.md
├── 25-VERIFICATION.md
├── 26-TEST-STRATEGY.md
├── 27-PERFORMANCE-BUDGET.md
├── 28-DASHBOARD.md
├── 29-CLI.md
├── 30-API.md
├── 31-KNOWLEDGE-INTEGRATION.md
├── 32-RUNTIME-INTEGRATION.md
├── 33-PLATFORM-GOVERNANCE.md
├── 34-ARCHITECTURE-DECISIONS.md
├── 35-ARCHITECTURE-FREEZE.md
├── index.yaml
└── README.md
```
