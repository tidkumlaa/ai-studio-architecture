---
knowledge_id: KNW-PLAT-ARCH-031
title: "Platform Knowledge Integration"
status: AUTHORITATIVE
phase: "2.1D.0"
version: "1.0.0"
created: "2026-06-29"
purpose: "Specify how platform architecture knowledge is captured, linked, indexed, and consumed"
canonical_source: "architecture/docs/platform-consolidation/31-KNOWLEDGE-INTEGRATION.md"
dependencies:
  - "06-RUNTIME-MODEL.md"
  - "13-METADATA-STANDARD.md"
related_documents:
  - "17-MODULE-REGISTRY.md"
  - "15-CAPABILITY-REGISTRY.md"
  - "00-README.md"
acceptance_criteria:
  - "Knowledge ID format is machine-verifiable"
  - "All architecture documents are indexable by knowledge runtime"
  - "Cross-references are bidirectional"
verification_checklist:
  - "[ ] All documents have Knowledge IDs"
  - "[ ] Knowledge runtime can compile architecture docs"
  - "[ ] Cross-reference graph has no broken links"
future_extensions:
  - "Semantic search over architecture knowledge base"
  - "Knowledge graph visualization"
---

# Platform Knowledge Integration

## Knowledge ID Format

```
KNW-PLAT-ARCH-NNN
```

| Segment | Meaning |
|---------|---------|
| `KNW` | Knowledge artifact prefix |
| `PLAT` | Platform domain |
| `ARCH` | Architecture sub-domain |
| `NNN` | Three-digit sequence (000–999) |

Examples: `KNW-PLAT-ARCH-001`, `KNW-PLAT-ARCH-035`

All 37 architecture documents carry IDs `KNW-PLAT-ARCH-000` through `KNW-PLAT-ARCH-036`.

---

## Knowledge Document Schema

Every architecture document front-matter is a knowledge entry:

```yaml
knowledge_id: KNW-PLAT-ARCH-NNN
title: string
status: DRAFT | REVIEW | AUTHORITATIVE | FROZEN | SUPERSEDED
phase: string
version: semver
created: ISO date
purpose: string
canonical_source: path
dependencies: list[filename]      # documents this one depends on
related_documents: list[filename]
acceptance_criteria: list[string]
verification_checklist: list[string]
future_extensions: list[string]
```

---

## Knowledge Runtime Integration

The `knowledge_runtime` can compile all architecture documents:

```python
# knowledge_runtime compiles architecture docs into its graph
KnowledgeCompiler.compile(
    source_dir="architecture/docs/platform-consolidation/",
    pattern="*.md",
    id_field="knowledge_id",
    format="markdown+yaml-frontmatter",
)
```

After compilation:
- Each document becomes a node in the `KnowledgeGraph`
- `dependencies` and `related_documents` become edges
- The graph is searchable via `KnowledgeSearch`

---

## Cross-Reference Graph

All 37 documents form a directed dependency graph. Key relationships:

```
00-README ← (referenced by all)

01-VISION ← 02-OBJECT-MODEL
          ← 03-DIRECTORY-ARCHITECTURE

02-OBJECT-MODEL ← 05-MODULE-CATALOG
               ← 13-METADATA-STANDARD

06-RUNTIME-MODEL ← 07-KERNEL-MODEL
                ← 09-SERVICE-MODEL
                ← 10-DEPENDENCY-RULES
                ← 11-LAYER-RULES
                ← 18-EVENT-MODEL

10-DEPENDENCY-RULES ← 11-LAYER-RULES
                    ← 12-IMPORT-RULES

12-IMPORT-RULES ← 23-IMPORT-REWRITE

22-MIGRATION-ENGINE ← 23-IMPORT-REWRITE
                    ← 24-ROLLBACK-ENGINE
                    ← 25-VERIFICATION

35-ARCHITECTURE-FREEZE ← (references all preceding docs)
```

**Rule:** `dependencies` edges must be acyclic. A document may only depend on documents
with lower sequence numbers.

---

## Knowledge Search Queries

Once compiled, architecture knowledge is searchable:

```python
# Find all documents about import rules
results = knowledge_search.search("import rules")

# Find all FROZEN documents
results = knowledge_search.search(status="FROZEN")

# Find documents related to migration
results = knowledge_search.search("migration", domain="ARCH")

# Get document by Knowledge ID
doc = knowledge_graph.get("KNW-PLAT-ARCH-022")
```

---

## Metadata Linking

Architecture documents link to runtime entities:

| Document | Links to |
|----------|---------|
| 05-MODULE-CATALOG.md | Module IDs in `module.yaml` files |
| 06-RUNTIME-MODEL.md | Runtime IDs in `runtime.yaml` files |
| 09-SERVICE-MODEL.md | Service IDs in `service.yaml` files |
| 15-CAPABILITY-REGISTRY.md | Capability IDs in `capability.yaml` files |
| 14-PLATFORM-MANIFEST.md | `platform/MANIFEST.yaml` |

The knowledge runtime maintains these links and detects broken references.

---

## Knowledge Quality Rules

| Rule | Description | Severity |
|------|-------------|---------|
| KQ-001 | Every document has a unique Knowledge ID | ERROR |
| KQ-002 | No document references a non-existent Knowledge ID | ERROR |
| KQ-003 | Documents in AUTHORITATIVE status pass all acceptance criteria | ERROR |
| KQ-004 | Documents in FROZEN status have no open verification checklist items | ERROR |
| KQ-005 | Related document links are bidirectional | WARNING |
| KQ-006 | No document exceeds 400 lines | WARNING |

---

## Architecture Knowledge Index

The knowledge index is maintained in `architecture/docs/platform-consolidation/index.yaml`
(see separate `index.yaml` file at the root of the docs directory).

The index is the machine-readable counterpart to `00-README.md`.

```yaml
# index.yaml structure
schema_version: "1.0"
domain: "platform-consolidation"
document_count: 37
documents:
  - knowledge_id: KNW-PLAT-ARCH-000
    filename: "00-README.md"
    title: "Architecture Navigation Guide"
    status: AUTHORITATIVE
  # ... all 37 entries
```

---

## Knowledge Compilation CLI

```bash
# Compile architecture docs into knowledge graph
platform-knowledge compile --source architecture/docs/platform-consolidation/

# Validate cross-references
platform-knowledge validate --check-links

# Search knowledge base
platform-knowledge search "import rules"

# Show document graph
platform-knowledge graph --format dot > architecture.dot
```
