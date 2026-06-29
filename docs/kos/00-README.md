---
knowledge_id: KNW-KOS-ARCH-000
title: "KOS Architecture Navigation Guide"
status: AUTHORITATIVE
phase: "3.0A"
version: "1.0.0"
created: "2026-06-30"
purpose: "Navigation index for all Phase 3.0A Knowledge Operating System architecture documents"
canonical_source: "architecture/docs/kos/00-README.md"
dependencies: []
related_documents:
  - "01-VISION.md"
  - "17-ARCHITECTURE-FREEZE.md"
acceptance_criteria:
  - "All 20 documents listed"
  - "Reading order defined"
  - "Knowledge ID format documented"
verification_checklist:
  - "[ ] All documents exist"
  - "[ ] All Knowledge IDs follow KNW-KOS-ARCH-NNN"
future_extensions: []
---

# KOS Architecture Navigation Guide

## What This Is

Phase 3.0A — Knowledge Operating System (KOS) Vision & Master Architecture.

This directory contains the complete architecture for the Knowledge Operating System.
KOS is the permanent, foundational layer of the entire AI Studio ecosystem.
Every future system depends on KOS. Nothing may exist outside the Knowledge System.

**Architecture only. No implementation. No runtime code.**

---

## Knowledge ID Format

```
KNW-KOS-ARCH-NNN
```

| Segment | Meaning |
|---------|---------|
| `KNW` | Knowledge artifact |
| `KOS` | Knowledge Operating System domain |
| `ARCH` | Architecture sub-domain |
| `NNN` | Three-digit sequence (000–017) |

---

## Reading Order

### Tier 0 — Orientation
| Doc | Title |
|-----|-------|
| [00-README.md](00-README.md) | This file — navigation |

### Tier 1 — Foundation
| Doc | Title |
|-----|-------|
| [01-VISION.md](01-VISION.md) | KOS Vision & Mission |
| [02-CORE-PHILOSOPHY.md](02-CORE-PHILOSOPHY.md) | Core Philosophy — Knowledge as Origin |
| [03-GOLDEN-RULES.md](03-GOLDEN-RULES.md) | The 10 Golden Rules |

### Tier 2 — Knowledge Architecture
| Doc | Title |
|-----|-------|
| [04-OBJECT-MODEL.md](04-OBJECT-MODEL.md) | Knowledge Object Model |
| [05-KNOWLEDGE-GRAPH.md](05-KNOWLEDGE-GRAPH.md) | Knowledge Graph |
| [06-KNOWLEDGE-COMPILER.md](06-KNOWLEDGE-COMPILER.md) | Knowledge Compiler |
| [07-KNOWLEDGE-REGISTRY.md](07-KNOWLEDGE-REGISTRY.md) | Knowledge Registry |

### Tier 3 — Knowledge Engines
| Doc | Title |
|-----|-------|
| [08-QUERY-ENGINE.md](08-QUERY-ENGINE.md) | Knowledge Query Engine |
| [09-REASONING-ENGINE.md](09-REASONING-ENGINE.md) | Knowledge Reasoning Engine |
| [10-EXECUTION-ENGINE.md](10-EXECUTION-ENGINE.md) | Knowledge Execution Engine |

### Tier 4 — Governance & Quality
| Doc | Title |
|-----|-------|
| [11-GOVERNANCE.md](11-GOVERNANCE.md) | Knowledge Governance |
| [12-QUALITY.md](12-QUALITY.md) | Knowledge Quality |
| [13-LIFECYCLE.md](13-LIFECYCLE.md) | Knowledge Lifecycle |

### Tier 5 — Platform & Systems
| Doc | Title |
|-----|-------|
| [14-KOS-PLATFORM.md](14-KOS-PLATFORM.md) | KOS as Platform Owner |
| [15-FUTURE-SYSTEMS.md](15-FUTURE-SYSTEMS.md) | Future Systems Dependency |
| [16-IMPLEMENTATION-ROADMAP.md](16-IMPLEMENTATION-ROADMAP.md) | Implementation Roadmap (3.0A–3.0I) |

### Tier 6 — Closure
| Doc | Title |
|-----|-------|
| [17-ARCHITECTURE-FREEZE.md](17-ARCHITECTURE-FREEZE.md) | Architecture Freeze |

---

## Document Status Values

| Status | Meaning |
|--------|---------|
| DRAFT | Being written |
| REVIEW | Under peer review |
| AUTHORITATIVE | Approved, in effect |
| FROZEN | Cannot be changed without new ADR |
| SUPERSEDED | Replaced by newer document |

---

## Document Summary

| # | Document | Knowledge ID | Purpose |
|---|----------|--------------|---------|
| 00 | README | KNW-KOS-ARCH-000 | Navigation |
| 01 | VISION | KNW-KOS-ARCH-001 | Why KOS exists |
| 02 | CORE-PHILOSOPHY | KNW-KOS-ARCH-002 | Knowledge-as-origin paradigm |
| 03 | GOLDEN-RULES | KNW-KOS-ARCH-003 | 10 inviolable rules |
| 04 | OBJECT-MODEL | KNW-KOS-ARCH-004 | All knowledge objects |
| 05 | KNOWLEDGE-GRAPH | KNW-KOS-ARCH-005 | Graph model and relationships |
| 06 | KNOWLEDGE-COMPILER | KNW-KOS-ARCH-006 | Compiler design |
| 07 | KNOWLEDGE-REGISTRY | KNW-KOS-ARCH-007 | Central registry |
| 08 | QUERY-ENGINE | KNW-KOS-ARCH-008 | Query and search |
| 09 | REASONING-ENGINE | KNW-KOS-ARCH-009 | Reasoning and answers |
| 10 | EXECUTION-ENGINE | KNW-KOS-ARCH-010 | Execution plan generation |
| 11 | GOVERNANCE | KNW-KOS-ARCH-011 | Governance rules |
| 12 | QUALITY | KNW-KOS-ARCH-012 | Quality scoring |
| 13 | LIFECYCLE | KNW-KOS-ARCH-013 | State machine |
| 14 | KOS-PLATFORM | KNW-KOS-ARCH-014 | KOS ownership model |
| 15 | FUTURE-SYSTEMS | KNW-KOS-ARCH-015 | All systems depend on KOS |
| 16 | IMPLEMENTATION-ROADMAP | KNW-KOS-ARCH-016 | Phase 3.0A through Products |
| 17 | ARCHITECTURE-FREEZE | KNW-KOS-ARCH-017 | Freeze declaration |
| — | index.yaml | — | Machine-readable index |
| — | README.md | — | Human entry point |
