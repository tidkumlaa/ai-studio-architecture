---
knowledge_id: KA-PLAT-012
version: "1.0.0"
status: approved
owner: Chief Knowledge Architect
phase: 2.0D.1
created: 2026-06-29
review_date: 2026-12-29
canonical: true
domain: DOM-KNOWLEDGE
capability: migration-platform
type: specification
---

# Desktop Planning

## Future Dashboard Designs — Architecture Only

> **Constraint:** This document describes dashboard architecture only.
> No implementation. Dashboards are Phase 2.0D.2.

---

## 1. Dashboard Inventory

Five dashboards are planned for Phase 2.0D.2:

| Dashboard | Purpose | Primary User |
|-----------|---------|-------------|
| Repository Explorer | Browse all documents by domain and capability | All engineers |
| Knowledge Explorer | Navigate the knowledge graph interactively | Architects |
| Migration Progress | Track Phase 2.0D.1 migration status | Chief Architect |
| Knowledge Health | Live health scores and trends | All engineers |
| Coverage Dashboard | Architecture/implementation/testing coverage | Tech leads |
| Relationship Viewer | Visualize document dependencies | Architects |

---

## 2. Repository Explorer

**Purpose:** File-browser-style navigation of the knowledge repository, organized
by domain and capability rather than by file system path.

**Data source:** Capability Index, Knowledge Registry

**Key views:**
- Domain tree (10 domains → N capabilities → documents)
- Filter by type, status, health score
- Quick preview of frontmatter and H1 heading
- Open document in editor or browser

**Layout:**

```
┌─────────────────────────┬────────────────────────────────────────┐
│ Domains                 │ Documents in: DOM-WORKFLOW / workflow-runtime │
│                         │                                        │
│ ▼ DOM-WORKFLOW          │ architecture.md     ✓ approved  72/100 │
│   ▶ workflow-runtime    │ specification.md    ○ draft     45/100 │
│   ▶ event-bus           │ implementation.md   ○ draft     38/100 │
│                         │ testing.md          ✗ missing         │
│ ▶ DOM-AI                │                                        │
│ ▶ DOM-KNOWLEDGE         │                                        │
└─────────────────────────┴────────────────────────────────────────┘
```

---

## 3. Knowledge Explorer

**Purpose:** Interactive graph visualization of the knowledge graph — nodes are
documents, edges are relationships.

**Data source:** graph.json (Knowledge Graph seed), Relationship Registry

**Key views:**
- Force-directed graph (D3 or Cytoscape.js)
- Filter by domain, capability, relationship type
- Click node → document preview
- Highlight dependency chains
- Orphan detection (disconnected nodes highlighted in red)

**Layout:**

```
┌──────────────────────────────────────────────────────────────────┐
│  [Filter: Domain ▾] [Filter: Type ▾] [Filter: Status ▾]         │
│                                                                  │
│          ○──────────○                                            │
│         /│           \        ● = approved                       │
│        / │            ○       ○ = draft                          │
│       ○  │       ○────●       ✗ = orphan                        │
│          │      /                                                │
│          ○─────○                                                 │
│                                                                  │
│  Selected: KA-ARCH-001 workflow-runtime/architecture.md          │
│  Implements: KA-SPEC-020  |  Depends on: KA-SPEC-001, KA-SPEC-008│
└──────────────────────────────────────────────────────────────────┘
```

---

## 4. Migration Progress Dashboard

**Purpose:** Track the status of the Phase 2.0D.1 migration in real time.

**Data source:** Migration Plan, per-phase completion logs

**Key views:**
- Phase progress bar (A → B → C → D → E → F)
- Document count: total / migrated / remaining / failed
- Current phase details and estimated completion
- Error log for failed documents
- Phase gate status (Health ≥ 80?)

**Layout:**

```
┌──────────────────────────────────────────────────────────────────┐
│ Migration Progress — Phase 2.0D.1                                │
│                                                                  │
│ [A ████] [B ████] [C ████] [D ░░░░] [E ░░░░] [F ░░░░]           │
│  Done     Done     Done    In Progress                           │
│                                                                  │
│ Phase D: Knowledge Splitter                                      │
│   Documents split:    18 / 23                                    │
│   Current:            AI-STUDIO-MASTER-ARCHITECTURE.md           │
│   Estimated remaining: 2h 15m                                    │
│                                                                  │
│ Overall: 412 / 786 documents migrated (52%)                      │
└──────────────────────────────────────────────────────────────────┘
```

---

## 5. Knowledge Health Dashboard

**Purpose:** Live health scores for all documents, capabilities, and domains.

**Data source:** Health Analyzer output, HealthIndex

**Key views:**
- Repository score with trend line
- Capability health heatmap
- Top 20 worst-scoring documents
- Dimension breakdown (structural / relationship / freshness / etc.)
- Phase gate progress (target: 80)

---

## 6. Coverage Dashboard

**Purpose:** Architecture, implementation, testing, and desktop coverage per capability.

**Data source:** Coverage Analyzer (Phase 2.0E)

**Key views:**
- Coverage matrix (capability × dimension)
- Gap detection — capabilities with missing coverage dimensions
- Coverage trend over phases

---

## 7. Relationship Viewer

**Purpose:** Drill into specific relationship chains — impact analysis, traceability.

**Data source:** Relationship Registry, Knowledge Graph

**Key views:**
- "What does this document depend on?" (forward chain)
- "What depends on this document?" (reverse chain)
- "What breaks if I change this?" (impact analysis)
- Traceability chain: requirement → decision → architecture → implementation → test

---

## 8. Technical Requirements (Phase 2.0D.2)

All dashboards share these requirements:

| Requirement | Value |
|-------------|-------|
| Data refresh | On file change (watch mode) or manual refresh |
| Backend | KnowledgeRuntimeService REST API |
| Rendering | Desktop application panel (existing panel system) |
| Auth | Local only — no external auth required |
| Performance | Initial load < 2s; refresh < 500ms |

---

## References

- [01-MIGRATION-ENGINE-ARCHITECTURE.md](01-MIGRATION-ENGINE-ARCHITECTURE.md) — Engine that feeds dashboard data
- [KA-SPEC-020](../knowledge-core/20-KNOWLEDGE-RUNTIME-ARCHITECTURE.md) — REST API specification
- Phase 2.0D.2 instructions — Full implementation specification (pending)
