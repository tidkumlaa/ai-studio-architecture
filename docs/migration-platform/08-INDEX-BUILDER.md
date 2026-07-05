---
knowledge_id: KA-PLAT-009
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
depends_on:
  - id: KA-PLAT-006
    reason: "Consumes MetadataMap"
  - id: KA-PLAT-007
    reason: "Consumes Relationship list"
  - id: KA-IDX-001
    reason: "7 index types defined in Knowledge Index Design"
  - id: KA-SPEC-010
    reason: "Knowledge Index Engine specification"
---

# Index Builder

## Repository, Capability, and Knowledge Graph Index Generation

---

## 1. Purpose

The IndexBuilder generates all index artifacts defined by KA-IDX-001. It consumes
the outputs of the Metadata Generator, Relationship Resolver, and Canonical Resolver
to produce both human-readable and machine-readable indexes.

---

## 2. Indexes Generated

| Index | Output File | Format | Consumer |
|-------|------------|--------|---------|
| Knowledge Registry | `KNOWLEDGE-REGISTRY.yaml` | YAML | KIE, IDG |
| Domain Index | `DOMAIN-INDEX.md` + `domain-index.yaml` | MD + YAML | Navigation |
| Capability Index | Per-capability `index.yaml` | YAML | KIE |
| Technology Index | `technology-index.yaml` | YAML | Navigation |
| Relationship Registry | `relationship-registry.yaml` | YAML | KIE, Graph |
| Navigation Tree | `KNOWLEDGE-INDEX.md` | MD | Humans |
| Graph Seed | `graph.json` | JSON | KGE (Phase 2.0D.2) |

---

## 3. Knowledge Registry Structure

```yaml
# KNOWLEDGE-REGISTRY.yaml
meta:
  version: "1.0.0"
  generated: "2026-06-29T00:00:00Z"
  generator: "IndexBuilder 1.0.0"
  root_path: "/path/to/repo"     # Stored as relative reference

next_available:
  ARCH: 43
  SPEC: 21
  ADR:  12
  # ... all type codes

registry:
  KA-ARCH-001:
    path: "architecture/docs/knowledge-core/01-KNOWLEDGE-OBJECT-SPECIFICATION.md"
    capability: "knowledge-core"
    domain: "DOM-KNOWLEDGE"
    type: "specification"
    status: "approved"
    canonical: true
    version: "1.0.0"
  # ...
```

---

## 4. Capability Index Structure

Each capability directory receives an `index.yaml`:

```yaml
# <capability>/index.yaml
capability: workflow-runtime
domain: DOM-WORKFLOW
version: "1.0.0"
status: active
generated: "2026-06-29"

documents:
  - id: KA-ARCH-042
    type: architecture
    file: architecture.md
    status: approved
    canonical: true
  - id: KA-SPEC-022
    type: specification
    file: specification.md
    status: draft
    canonical: true

required_present:
  architecture: true
  specification: false     # Missing — flagged
  implementation: true
  testing: false           # Missing — flagged

health_score: 72
```

---

## 5. Relationship Registry Structure

```yaml
# relationship-registry.yaml
generated: "2026-06-29"

relationships:
  - source: "docs/workflow/architecture.md"
    target: "docs/knowledge-core/01-KNOWLEDGE-OBJECT-SPECIFICATION.md"
    type: "implements"
    confidence: 0.95
    inferred: false

  - source: "docs/knowledge-core/01-KNOWLEDGE-OBJECT-SPECIFICATION.md"
    target: "docs/workflow/architecture.md"
    type: "implemented_by"
    confidence: 0.95
    inferred: true          # Reverse-inferred

broken_links:
  - source: "docs/old/doc.md"
    target: "docs/missing.md"
    line: 42
```

---

## 6. Graph Seed Structure

The graph seed provides the initial state for the Knowledge Graph Engine:

```json
{
  "vertices": [
    {
      "id": "KA-ARCH-001",
      "path": "...",
      "type": "architecture",
      "capability": "knowledge-core",
      "domain": "DOM-KNOWLEDGE",
      "status": "approved"
    }
  ],
  "edges": [
    {
      "from": "KA-ARCH-001",
      "to": "KA-SPEC-001",
      "type": "implements",
      "confidence": 0.95,
      "inferred": false
    }
  ]
}
```

---

## 7. Navigation Tree

`KNOWLEDGE-INDEX.md` is the human-readable master navigation document:

```markdown
# Knowledge Index

## By Domain

### DOM-WORKFLOW — Workflow Runtime
- **workflow-runtime** — [Architecture](docs/workflow/architecture.md) · [Spec](...)
  - Health: 72/100

### DOM-KNOWLEDGE — Knowledge System
- **knowledge-core** — 20 specifications — Health: 0/100 (audit not yet run)

## Recently Modified
...
## Low Health (< 50)
...
```

---

## 8. Build Algorithm

```
BUILD(inventory, metadata_map, relationships, canonical_map):
  registry = build_registry(metadata_map)
  capability_indexes = build_capability_indexes(metadata_map)
  rel_registry = build_relationship_registry(relationships)
  nav_tree = build_navigation_tree(metadata_map, capability_indexes)
  graph_seed = build_graph_seed(registry, rel_registry)
  
  RETURN BuiltIndexes(
    registry=registry,
    capability_indexes=capability_indexes,
    rel_registry=rel_registry,
    nav_tree=nav_tree,
    graph_seed=graph_seed
  )
```

**Complexity:** O(D + E) — linear in documents and edges.

---

## References

- [KA-IDX-001](../knowledge-architecture/KNOWLEDGE-INDEX-DESIGN.md) — 7 index types
- [KA-SPEC-010](../knowledge-core/10-KNOWLEDGE-INDEX-ENGINE.md) — Index engine specification
- [09-HEALTH-ANALYZER.md](09-HEALTH-ANALYZER.md) — Health scores feed into capability indexes
