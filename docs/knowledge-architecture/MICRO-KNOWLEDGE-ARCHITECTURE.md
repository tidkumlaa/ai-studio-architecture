---
knowledge_id: KA-MICRO-001
version: "1.0.0"
status: approved
owner: Chief Knowledge Architect
phase: 2.0D.0
created: 2026-06-29
review_date: 2026-12-29
canonical: true
implements: KA-VISION-001
---

# Micro Knowledge Architecture

## The Law of Small Documents

---

## 1. The Problem With Large Documents

An 8,000-line document is not one piece of knowledge. It is a filing cabinet with the drawers welded shut.

The symptoms of monolithic documentation:

- **Search replaces understanding.** Users `Ctrl+F` through documents because the mental model no longer fits in working memory.
- **Ownership is diffuse.** When one document covers twelve topics, no single person is responsible for any of them.
- **Updates create conflict.** Multi-topic documents require multi-reviewer changes for single-topic updates.
- **AI context is wasted.** An AI assistant that loads a 500KB document to answer a 200-token question wastes 90% of its context window on noise.
- **Staleness is invisible.** A stale section inside a large document is indistinguishable from fresh content.
- **Duplication is undetectable.** When the same concept appears in three large documents, there is no mechanism to detect or prevent it.

The solution is not better search. The solution is **smaller, focused, identifiable knowledge objects**.

---

## 2. Document Size Limits

### 2.1 Hard Limits

| Rule | Limit | Enforcement |
|------|-------|-------------|
| **Maximum document length** | 500 lines | Audit rule violation — blocks merge |
| **Maximum section depth** | H3 (three levels) | Audit rule violation |
| **Maximum frontmatter size** | 50 lines | Audit rule warning |

### 2.2 Recommended Targets

| Document Type | Target Length | Notes |
|---------------|--------------|-------|
| overview.md | 100–200 lines | High-level only; links to detail |
| architecture.md | 200–400 lines | Core design decisions |
| api.md | 150–300 lines | Per-endpoint detail lives in separate files |
| database.md | 100–250 lines | Schema overview; detail in schema files |
| events.md | 100–200 lines | Event catalog per capability |
| security.md | 100–200 lines | Security model and threat model |
| desktop.md | 100–200 lines | UI/UX for capability |
| implementation.md | 150–300 lines | Implementation guidance |
| testing.md | 100–200 lines | Test strategy and coverage |
| roadmap.md | 100–200 lines | Future work; links to ADRs |
| history.md | 100–300 lines | Decision history; links to decisions |
| references.md | 50–100 lines | Links only — no prose |
| README.md | 50–150 lines | Entry point; navigation only |
| index.yaml | 30–100 lines | Machine-readable index |

### 2.3 Split Rule

When a document approaches 400 lines, it must be reviewed for split candidacy. The split rule is:

> **If a section can stand alone with its own identity, it must become its own document.**

A section can stand alone if:
- It has a distinct concept or concern
- It has its own consumers (not always read together with siblings)
- It could have its own owner
- It could be versioned independently

---

## 3. The Knowledge Object

A **knowledge object** is the atomic unit of the knowledge system. Every file is a knowledge object.

### 3.1 Knowledge Object Properties

Every knowledge object has:

```
Identity     : A unique Knowledge ID (e.g., KA-ARCH-003)
Type         : The document type (overview, architecture, api, etc.)
Capability   : The capability it belongs to (e.g., workflow-runtime)
Domain       : The domain it belongs to (e.g., DOM-WORKFLOW)
Owner        : A declared responsible party
Status       : draft | review | approved | deprecated | archived
Version      : Semantic version (major.minor.patch)
Relationships: Explicit forward and reverse links
Health       : Computable score from metadata completeness
```

### 3.2 Knowledge Object Types

| Type | ID Prefix | Purpose |
|------|-----------|---------|
| Vision | KA-VIS | Strategic vision and philosophy |
| Architecture | KA-ARCH | Core architectural design |
| Overview | KA-OVW | Capability entry point |
| API | KA-API | API specification reference |
| Database | KA-DB | Database schema reference |
| Event | KA-EVT | Event catalog entry |
| Security | KA-SEC | Security model |
| Desktop | KA-DSK | Desktop/UI specification |
| Implementation | KA-IMPL | Implementation guidance |
| Testing | KA-TEST | Test strategy |
| Roadmap | KA-ROAD | Forward roadmap |
| History | KA-HIST | Decision history |
| Reference | KA-REF | Reference/link collection |
| ADR | KA-ADR | Architecture Decision Record |
| Standard | KA-STD | Architecture standard |
| Index | KA-IDX | Index document |

---

## 4. Folder Hierarchy

### 4.1 Repository Root Structure

```
architecture/
├── README.md                        # Repository entry point
├── KNOWLEDGE-INDEX.md               # Master knowledge index
├── DOMAIN-INDEX.md                  # Domain registry
├── index.yaml                       # Machine-readable repository index
│
├── knowledge-architecture/          # This domain
│   ├── README.md
│   ├── index.yaml
│   └── [knowledge documents]
│
├── capabilities/                    # All capability knowledge
│   ├── README.md
│   ├── index.yaml
│   ├── workflow-runtime/
│   ├── prompt-os/
│   ├── provider-framework/
│   ├── provider-management/
│   ├── conversation-intelligence/
│   ├── context-intelligence/
│   ├── intelligent-routing/
│   ├── ai-runtime-os/
│   └── [future capabilities]
│
├── platform/                        # Platform-wide concerns
│   ├── security/
│   ├── contracts/
│   ├── standards/
│   └── infrastructure/
│
├── products/                        # Product-level knowledge
│   ├── content-factory/
│   ├── claude-cli/
│   └── [future products]
│
├── adr/                             # Architecture Decision Records
│   ├── README.md
│   ├── index.yaml
│   └── [ADR documents]
│
├── decisions/                       # Decision registry
│   ├── README.md
│   └── index.yaml
│
├── archive/                         # Deprecated and superseded documents
│   ├── README.md
│   ├── index.yaml
│   └── [by version: v1.5.5/, v2.0/, etc.]
│
└── history/                         # Architecture timeline
    ├── README.md
    └── ARCHITECTURE-TIMELINE.md
```

### 4.2 Capability Folder Structure

Every capability folder follows this exact structure:

```
capabilities/[capability-name]/
├── README.md           # Navigation entry point (required)
├── index.yaml          # Machine-readable index (required)
├── overview.md         # What this capability is (required)
├── architecture.md     # How it works — design and decisions (required)
├── api.md              # External API surface (if applicable)
├── database.md         # Database schema (if applicable)
├── events.md           # Events produced and consumed (if applicable)
├── security.md         # Security model (if applicable)
├── desktop.md          # Desktop UI specification (if applicable)
├── implementation.md   # Implementation guidance (required)
├── testing.md          # Test strategy and coverage (required)
├── roadmap.md          # Future work (required)
├── history.md          # Decision history (required)
└── references.md       # External references (required)
```

**Required files** — must exist for every capability, even if minimal.
**Optional files** — created only when the capability has that concern.

### 4.3 Section Hierarchy Rules

| Level | Tag | Use |
|-------|-----|-----|
| H1 (`#`) | Document title | One per document only |
| H2 (`##`) | Major section | Top-level divisions |
| H3 (`###`) | Sub-section | Detail within a section |
| H4+ | **Forbidden** | Extract to a sub-document instead |

The H4 rule is absolute. If content requires four levels of heading hierarchy, it requires its own document.

---

## 5. Knowledge Ownership

### 5.1 Ownership Hierarchy

Every knowledge object has exactly one declared owner at the **role level** (not person level). Role-level ownership survives personnel changes.

Ownership roles:

| Role | Scope |
|------|-------|
| Chief Knowledge Architect | Knowledge architecture domain |
| Chief Enterprise Architect | Cross-domain architecture |
| Capability Architect | Individual capability |
| Platform Architect | Platform-wide concerns |
| Product Architect | Product-level knowledge |
| Security Architect | Security knowledge |

### 5.2 Ownership Rules

- Every document must declare an `owner` in frontmatter
- Ownership transfers are recorded in `history.md`
- Orphaned documents (no owner, owner role eliminated) are flagged as health violations
- Owner must review document before `review_date` or health score degrades

### 5.3 Collective Ownership

Some documents are owned by a domain, not a role:

```yaml
owner: platform-team        # Team ownership
owner: governance-board     # Committee ownership
```

Collective ownership requires a named delegate who is responsible for review scheduling.

---

## 6. Knowledge Lifecycle

### 6.1 Lifecycle States

```
draft → review → approved → deprecated → archived
                     ↑           ↓
                     └── revision loop
```

| State | Meaning | Index Visible | Navigable |
|-------|---------|--------------|-----------|
| `draft` | Work in progress | No | No |
| `review` | Pending approval | No | No |
| `approved` | Active, authoritative | Yes | Yes |
| `deprecated` | Replaced or retired | Yes (with warning) | Yes (with warning) |
| `archived` | Permanently removed from active use | Archive index only | No |

### 6.2 Lifecycle Transitions

| Transition | Requires | Creates |
|-----------|---------|--------|
| draft → review | Author decision | Review request |
| review → approved | Owner approval | Index entry, health tracking |
| approved → deprecated | Owner decision + deprecation notice | Superseded-by link |
| deprecated → archived | 90-day hold + confirmation | Archive entry |
| approved → revision → approved | Minor change | Version increment |

### 6.3 Deprecation Rules

When a knowledge object is deprecated:

1. `status` changes to `deprecated`
2. `deprecated_date` is added to frontmatter
3. `superseded_by` link is added (if a replacement exists)
4. A `deprecated` banner is added to the document body
5. The canonical index is updated with `(deprecated)` marker
6. The document is NOT deleted until archived

### 6.4 Archival Rules

A document may be archived only after:
- 90 days in `deprecated` state
- Owner confirmation
- All references updated to point to successor (or removed)
- Archive entry created in `archive/index.yaml`

---

## 7. Composition Patterns

### 7.1 Reference Pattern

When Document A needs to reference concept C that lives in Document B:

```markdown
See [Concept C](../capability-b/architecture.md#concept-c) for the canonical definition.
```

Never copy the definition. Only reference it.

### 7.2 Summary Pattern

When Document A needs to summarize concept C for context:

```markdown
**Concept C** (defined in [architecture.md](../capability-b/architecture.md)) is the mechanism 
responsible for X. In the context of this document, the relevant behavior is Y.
```

A one-sentence summary is acceptable. A paragraph is a copy risk.

### 7.3 Overview Pattern

`overview.md` is an entry point, not a reference. It:
- Describes what the capability is (not how it works)
- Links to all detailed documents
- Contains no technical detail that is duplicated in other documents
- Is never more than 200 lines

### 7.4 README Navigation Pattern

`README.md` is a navigation panel, not a document. It:
- Contains no original content
- Links to all other documents in the folder
- Includes a one-sentence description of each linked document
- Is never more than 150 lines

---

## 8. Migration Strategy for Existing Documents

Existing large documents are split using the following algorithm:

1. **Classify** — Identify which capability and type each section belongs to
2. **Extract** — Move each section to its target document type
3. **Link** — Replace extracted content with a reference link
4. **Redirect** — Original large document becomes a `references.md` with links only
5. **Archive** — If the original document has no remaining unique content, deprecate it

See [REPOSITORY-REFACTORING-PLAN.md](REPOSITORY-REFACTORING-PLAN.md) for the full migration plan.

---

## References

- [KNOWLEDGE-ARCHITECTURE-VISION.md](KNOWLEDGE-ARCHITECTURE-VISION.md) — Philosophy and objectives
- [DOCUMENT-FOLDER-STANDARDS.md](DOCUMENT-FOLDER-STANDARDS.md) — Exact folder rules
- [METADATA-STANDARD.md](METADATA-STANDARD.md) — Frontmatter schema
- [REPOSITORY-AUDIT-RULES.md](REPOSITORY-AUDIT-RULES.md) — Enforcement rules
