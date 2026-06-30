---
knowledge_id: KA-SPEC-001
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
  - KA-VIS-001
  - KA-STD-002
---

# Knowledge Object Specification

## The Atomic Unit of the Knowledge System

---

## 1. Purpose

A KnowledgeObject is the fundamental runtime entity of the AI Studio Knowledge System. Everything that the knowledge system stores, indexes, queries, validates, and relates is a KnowledgeObject.

This specification defines what a KnowledgeObject is, what properties it carries, how it behaves, and what invariants it must satisfy at all times.

---

## 2. Definition

> **A KnowledgeObject is a self-describing, identifiable, versioned, typed, lifecycled, relatable piece of knowledge with a computable health state.**

The operative words:

| Word | Meaning |
|------|---------|
| Self-describing | Carries all metadata needed to interpret it without external context |
| Identifiable | Has a unique, stable, permanent identity |
| Versioned | Has a semantic version that changes with its content |
| Typed | Belongs to exactly one type in the Knowledge Type System |
| Lifecycled | Has an explicit, machine-readable lifecycle state |
| Relatable | Can declare typed relationships to other KnowledgeObjects |
| Computable health | Health score is derived from declared properties — not subjective |

---

## 3. KnowledgeObject Structure

### 3.1 Logical Structure

```
KnowledgeObject
├── Identity
│   ├── id: KnowledgeID              # Permanent, unique, never reused
│   ├── uri: KnowledgeURI            # Addressable URI
│   └── version: SemanticVersion     # Semantic version
│
├── Classification
│   ├── type: KnowledgeType          # From the Type System
│   ├── domain: DomainID             # From the Domain Registry
│   └── capability: CapabilityID     # From the Capability Registry
│
├── Ownership
│   ├── owner: OwnerRole             # Role-level owner
│   ├── created: Date                # Creation timestamp
│   └── review_date: Date            # Next required review
│
├── Content
│   ├── title: String                # Human-readable title
│   ├── body: MarkdownContent        # The actual knowledge
│   └── canonical: Boolean           # Is this the canonical source?
│
├── Lifecycle
│   ├── status: LifecycleState       # Current state
│   ├── deprecated_date?: Date       # Set on deprecation
│   ├── superseded_by?: KnowledgeID  # Set on supersession
│   └── archived_date?: Date         # Set on archival
│
├── Relationships
│   ├── implements: KnowledgeID[]
│   ├── depends_on: Dependency[]
│   ├── supersedes?: KnowledgeID
│   ├── related_apis: Reference[]
│   ├── related_database: Reference[]
│   ├── related_events: Reference[]
│   ├── related_adr: Reference[]
│   ├── related_products: ProductRef[]
│   └── implemented_by: ArtifactRef[]
│
├── Evidence
│   └── evidence: EvidenceRecord[]
│
├── Quality
│   ├── confidence: ConfidenceLevel
│   ├── coverage: CoverageMap
│   └── phase: PhaseID
│
├── Tags
│   └── tags: String[]
│
└── Health (computed — never declared)
    ├── health_score: Integer (0–100)
    └── health_computed_at: Timestamp
```

### 3.2 Minimal Valid KnowledgeObject

A KnowledgeObject is valid when the following fields are present and valid:

```yaml
id: KA-ARCH-001          # Required
version: "1.0.0"         # Required
status: draft            # Required
type: architecture       # Required
domain: DOM-WORKFLOW     # Required
capability: workflow-runtime  # Required
owner: "Capability Architect"  # Required
created: 2026-06-29      # Required
review_date: 2026-12-29  # Required (unless type is adr or history)
canonical: true          # Required for status=approved
phase: 2.0D.0.5          # Required
confidence: low          # Required
```

A KnowledgeObject with fewer fields than this is a **draft stub** — it exists in the registry but is not indexed.

---

## 4. KnowledgeObject Invariants

These invariants must hold at all times. The audit system enforces them.

| Invariant | Rule |
|-----------|------|
| **I1 — Unique Identity** | No two KnowledgeObjects share the same `id` |
| **I2 — Stable Identity** | A `id` never changes after first assignment |
| **I3 — No Reuse** | A retired `id` is never reassigned |
| **I4 — Type Immutability** | The `type` of a KnowledgeObject never changes |
| **I5 — Single Canonical** | For any (capability, type) pair, at most one KnowledgeObject has `canonical: true` with `status: approved` |
| **I6 — Lifecycle Monotonicity** | Status transitions follow the declared lifecycle FSM (see [KA-SPEC-014](14-KNOWLEDGE-LIFECYCLE.md)) |
| **I7 — Reference Integrity** | All IDs referenced in relationships exist in the registry |
| **I8 — Owner Required** | No approved KnowledgeObject has an empty `owner` |
| **I9 — No Self-Reference** | A KnowledgeObject cannot depend on itself |
| **I10 — Health is Derived** | `health_score` is never declared — always computed |

---

## 5. KnowledgeObject Serialization

### 5.1 Primary Format — Markdown with YAML Frontmatter

The primary serialization format is a Markdown file with a YAML frontmatter block. This format is:
- Human-readable and editable
- AI-parseable without special tooling
- Version-controllable with meaningful diffs
- Rendereable in any Markdown viewer

```
---
[YAML frontmatter — all metadata fields]
---
[Markdown body — the knowledge content]
```

### 5.2 Exchange Format — JSON

For the Knowledge Index Engine, Knowledge Query Engine, and all runtime tools, KnowledgeObjects are exchanged as JSON:

```json
{
  "id": "KA-ARCH-001",
  "uri": "ka://dom-workflow/workflow-runtime/architecture/KA-ARCH-001",
  "version": "1.2.0",
  "type": "architecture",
  "domain": "DOM-WORKFLOW",
  "capability": "workflow-runtime",
  "owner": "Capability Architect",
  "status": "approved",
  "canonical": true,
  "created": "2026-06-29",
  "review_date": "2026-12-29",
  "confidence": "high",
  "coverage": {
    "architecture": 90,
    "implementation": 75,
    "testing": 60
  },
  "relationships": {
    "implements": ["KA-STD-001"],
    "depends_on": [
      { "id": "KA-ARCH-005", "reason": "Delegates provider selection" }
    ]
  },
  "evidence": [
    {
      "type": "source_file",
      "ref": "platform/workflow-runtime/src/executor.py",
      "verified": true
    }
  ],
  "health_score": 82,
  "health_computed_at": "2026-06-29T14:00:00Z"
}
```

### 5.3 Compact Format — PromptPack

For AI assistant consumption, a KnowledgeObject is serialized as a compact PromptPack (see [KA-SPEC-009](09-KNOWLEDGE-COMPILER-SPECIFICATION.md)):

```
[KA-ARCH-001 v1.2.0 | workflow-runtime | architecture | approved | health:82]
Title: Workflow Runtime Architecture
Domain: DOM-WORKFLOW | Owner: Capability Architect | Reviewed: 2026-06-29
Depends: KA-ARCH-005 (provider-framework)
Implements: KA-STD-001

[Content follows]
```

---

## 6. KnowledgeObject Identity vs. Content

A critical distinction:

| Aspect | Rule |
|--------|------|
| **Identity** | Permanent. The `id` is assigned once and never changes. |
| **Content** | Mutable. The `body` can be updated; `version` is incremented. |
| **Relationships** | Mutable. Can be added (version bump) or removed (version bump). |
| **Type** | Immutable. Once set, never changes. |
| **Status** | Transitions follow the lifecycle FSM. Cannot jump arbitrarily. |

When a KnowledgeObject is fundamentally redesigned (MAJOR version bump), it retains its identity. When it is superseded by a completely different design, a new KnowledgeObject is created and the old one declares `superseded_by`.

---

## 7. KnowledgeObject Collections

KnowledgeObjects are organized into collections:

| Collection | Contents | Indexed By |
|-----------|---------|-----------|
| **CapabilityKnowledgeSet** | All KnowledgeObjects for one capability | capability |
| **DomainKnowledgeSet** | All KnowledgeObjects for one domain | domain |
| **TypeKnowledgeSet** | All KnowledgeObjects of one type | type |
| **KnowledgeRepository** | All KnowledgeObjects in the system | id, uri |

Collections are not containers — they are views. A KnowledgeObject is not stored "in" a collection; collections are queries over the registry.

---

## References

- [02-KNOWLEDGE-TYPE-SYSTEM.md](02-KNOWLEDGE-TYPE-SYSTEM.md) — Type system for KnowledgeObjects
- [03-KNOWLEDGE-IDENTITY-SPECIFICATION.md](03-KNOWLEDGE-IDENTITY-SPECIFICATION.md) — Identity rules
- [04-KNOWLEDGE-URI-SPECIFICATION.md](04-KNOWLEDGE-URI-SPECIFICATION.md) — URI addressing
- [05-KNOWLEDGE-SCHEMA.md](05-KNOWLEDGE-SCHEMA.md) — Formal schema
- [14-KNOWLEDGE-LIFECYCLE.md](14-KNOWLEDGE-LIFECYCLE.md) — Lifecycle FSM
- [KA-STD-002](../knowledge-architecture/METADATA-STANDARD.md) — Metadata standard (implemented by this spec)
