# Knowledge Architecture

> The foundational design for transforming AI Studio's architecture repository from a collection of documents into a navigable, composable, machine-readable knowledge system.

**Phase:** 2.0D.0 | **Owner:** Chief Knowledge Architect | **Status:** Approved

---

## Phase 2.0D.0 Deliverables

This folder contains all Phase 2.0D.0 architecture specifications. No runtime code is produced in this phase.

| Document | Description |
|----------|-------------|
| [KNOWLEDGE-ARCHITECTURE-VISION.md](KNOWLEDGE-ARCHITECTURE-VISION.md) | Philosophy, principles, objectives, constraints, and roadmap |
| [MICRO-KNOWLEDGE-ARCHITECTURE.md](MICRO-KNOWLEDGE-ARCHITECTURE.md) | Document size limits, composition rules, lifecycle, and folder hierarchy |
| [DOCUMENT-FOLDER-STANDARDS.md](DOCUMENT-FOLDER-STANDARDS.md) | Exact folder structure and required file definitions for every capability |
| [METADATA-STANDARD.md](METADATA-STANDARD.md) | Complete YAML frontmatter schema for all knowledge documents |
| [KNOWLEDGE-INDEX-DESIGN.md](KNOWLEDGE-INDEX-DESIGN.md) | Design for all seven index types — the discovery system |
| [CANONICAL-SOURCE-STRATEGY.md](CANONICAL-SOURCE-STRATEGY.md) | One concept, one source — duplicate, orphan, and dead document detection |
| [RELATIONSHIP-MODEL.md](RELATIONSHIP-MODEL.md) | All relationship types, declaration syntax, and graph invariants |
| [KNOWLEDGE-NAVIGATION.md](KNOWLEDGE-NAVIGATION.md) | Capability-centric navigation patterns for humans and AI assistants |
| [KNOWLEDGE-HEALTH-METRICS.md](KNOWLEDGE-HEALTH-METRICS.md) | Health scoring dimensions, thresholds, and phase gate criteria |
| [KNOWLEDGE-EVOLUTION-STRATEGY.md](KNOWLEDGE-EVOLUTION-STRATEGY.md) | Versioning, deprecation, migration, and architecture timeline |
| [REPOSITORY-REFACTORING-PLAN.md](REPOSITORY-REFACTORING-PLAN.md) | Phase 2.0D.1 migration plan with rollout phases and rollback strategy |
| [REPOSITORY-AUDIT-RULES.md](REPOSITORY-AUDIT-RULES.md) | Complete automated audit rule set (34 rules) for enforcement |

---

## Quick Reference

- **Next Phase:** 2.0D.1 — Repository Refactoring (apply this architecture)
- **Phase After:** 2.0D.2 — Documentation Intelligence Platform (build on top of this)
- **Phase Gate Criteria:** Repository health score ≥ 80 before starting 2.0D.2

## Reading Order

For first-time readers, read in this order:
1. KNOWLEDGE-ARCHITECTURE-VISION.md — understand the why
2. MICRO-KNOWLEDGE-ARCHITECTURE.md — understand the what
3. DOCUMENT-FOLDER-STANDARDS.md — understand the structure
4. METADATA-STANDARD.md — understand the schema
5. Remaining documents as needed by role
