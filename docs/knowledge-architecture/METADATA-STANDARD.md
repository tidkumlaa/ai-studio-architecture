---
knowledge_id: KA-STD-002
version: "1.0.0"
status: approved
owner: Chief Knowledge Architect
phase: 2.0D.0
created: 2026-06-29
review_date: 2026-12-29
canonical: true
implements: KA-MICRO-001
---

# Metadata Standard

## The Knowledge Document Frontmatter Schema

---

## 1. Purpose

Metadata is what transforms a Markdown file into a knowledge object.

Without metadata:
- A document has a filename, not an identity
- A document has content, not relationships
- A document has a modification date, not a review state
- A document cannot participate in health scoring, index building, or relationship traversal

Every knowledge document must contain a YAML frontmatter block that satisfies this standard. Documents without valid frontmatter are treated as **drafts** and excluded from all indexes.

---

## 2. YAML Frontmatter Schema

The following is the complete, authoritative schema for all knowledge documents.

### 2.1 Full Schema

```yaml
---
# ─── IDENTITY ────────────────────────────────────────────────────────────────
knowledge_id: "KA-ARCH-001"           # Required. Unique stable identifier.
version: "1.0.0"                       # Required. Semantic version.
status: approved                       # Required. Lifecycle state.
canonical: true                        # Required. True if this is the canonical source.

# ─── CLASSIFICATION ──────────────────────────────────────────────────────────
domain: DOM-WORKFLOW                   # Required. Domain this document belongs to.
capability: workflow-runtime           # Required. Capability folder name.
type: architecture                     # Required. Document type.

# ─── OWNERSHIP ───────────────────────────────────────────────────────────────
owner: Capability Architect            # Required. Role-level owner.
created: 2026-06-29                    # Required. ISO 8601 creation date.
review_date: 2026-12-29                # Required. Next review deadline.

# ─── LIFECYCLE ───────────────────────────────────────────────────────────────
deprecated_date: ""                    # Set when status = deprecated.
superseded_by: ""                      # Knowledge ID of replacement document.
archived_date: ""                      # Set when status = archived.

# ─── RELATIONSHIPS ───────────────────────────────────────────────────────────
implements:                            # Standards this document implements.
  - KA-STD-001
  - KA-STD-002

depends_on:                            # Knowledge objects this document depends on.
  - id: KA-OVW-002
    reason: "Builds on provider framework overview"

implemented_by:                        # What implements the design in this document.
  - type: module
    ref: "platform/workflow-runtime/src"
    confidence: high

related_apis:                          # API documents related to this capability.
  - id: KA-API-001
    title: "Workflow Execution API"

related_database:                      # Database documents related to this capability.
  - id: KA-DB-001
    title: "Workflow Execution Schema"

related_events:                        # Event documents related to this capability.
  - id: KA-EVT-001
    title: "Workflow Events"

related_desktop:                       # Desktop documents related to this capability.
  - id: KA-DSK-001
    title: "Workflow Runtime Desktop"

related_tests:                         # Test documents or suites.
  - id: KA-TEST-001
    title: "Workflow Runtime Tests"

related_adr:                           # Architecture Decision Records.
  - id: KA-ADR-012
    title: "Workflow Executor Selection"

related_products:                      # Products that use this capability.
  - product: content-factory
    role: consumer
  - product: claude-cli
    role: consumer

related_workflows:                     # Workflows that involve this capability.
  - id: WF-001
    title: "Content Generation Workflow"

# ─── EVIDENCE ────────────────────────────────────────────────────────────────
evidence:                              # Evidence of implementation.
  - type: source_file
    ref: "platform/workflow-runtime/src/executor.py"
    line: 1
    verified: true
  - type: test
    ref: "platform/workflow-runtime/tests/test_executor.py"
    verified: true

# ─── QUALITY ─────────────────────────────────────────────────────────────────
phase: 2.0D.0                          # Phase when this document was created.
confidence: high                       # Confidence level: high | medium | low.
coverage:                              # Coverage dimensions (0–100).
  architecture: 90
  implementation: 75
  testing: 60
  desktop: 0                           # 0 if not applicable.

# ─── TAGS ────────────────────────────────────────────────────────────────────
tags:                                  # Searchable tags.
  - workflow
  - execution
  - orchestration
---
```

---

## 3. Field Reference

### 3.1 Identity Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `knowledge_id` | string | Yes | Unique identifier. Format: `KA-[TYPE]-[NNN]`. Never changes. |
| `version` | string | Yes | Semantic version `MAJOR.MINOR.PATCH`. Increment on every substantive change. |
| `status` | enum | Yes | `draft \| review \| approved \| deprecated \| archived` |
| `canonical` | boolean | Yes | `true` if this is the single authoritative source for this concept. `false` if this is a summary or reference copy. |

**Knowledge ID Format:**

```
KA-[TYPE]-[NNN]

Where TYPE is:
  VIS  = Vision
  ARCH = Architecture
  OVW  = Overview
  API  = API Specification
  DB   = Database
  EVT  = Event
  SEC  = Security
  DSK  = Desktop
  IMPL = Implementation
  TEST = Testing
  ROAD = Roadmap
  HIST = History
  REF  = References
  ADR  = Architecture Decision Record
  STD  = Standard
  IDX  = Index

Where NNN is a zero-padded 3-digit sequence number per type.

Examples:
  KA-ARCH-001   First architecture document
  KA-ADR-042    42nd ADR
  KA-API-007    7th API document
```

**Version Rules:**
- `MAJOR` — Breaking change to the concept described (not just the document)
- `MINOR` — New content added without breaking existing understanding
- `PATCH` — Corrections, typo fixes, reference updates

### 3.2 Classification Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `domain` | enum | Yes | Domain identifier from the domain registry |
| `capability` | string | Yes | Folder name of the owning capability |
| `type` | enum | Yes | Document type matching the filename |

**Domain Values:**

```
DOM-AI        AI Runtime OS
DOM-PROMPT    Prompt OS
DOM-WORKFLOW  Workflow Runtime
DOM-CONV      Conversation Intelligence
DOM-CTX       Context Intelligence
DOM-ROUTE     Intelligent Routing
DOM-PROVIDER  Provider Framework
DOM-PLATFORM  Platform Core
DOM-KNOWLEDGE Knowledge Architecture
DOM-PRODUCT   Product Layer
```

**Type Values:**

```
vision          KNOWLEDGE-ARCHITECTURE-VISION.md
architecture    architecture.md
overview        overview.md
api             api.md
database        database.md
event           events.md
security        security.md
desktop         desktop.md
implementation  implementation.md
testing         testing.md
roadmap         roadmap.md
history         history.md
references      references.md
adr             adr/[NNN]-[slug].md
standard        knowledge-architecture/[name].md
index           index.yaml (not a knowledge object — no ID needed)
```

### 3.3 Ownership Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `owner` | string | Yes | Role title of the responsible party |
| `created` | date | Yes | ISO 8601 date of document creation |
| `review_date` | date | Yes | Date by which the owner must re-review this document |

**Review Schedule by Document Type:**

| Type | Review Frequency |
|------|-----------------|
| vision | 12 months |
| architecture | 6 months |
| overview | 6 months |
| api | 3 months |
| database | 6 months |
| events | 6 months |
| security | 3 months |
| desktop | 6 months |
| implementation | 6 months |
| testing | 6 months |
| roadmap | 3 months |
| history | Never (append-only) |
| references | 3 months |
| adr | Never (immutable after approval) |
| standard | 12 months |

### 3.4 Lifecycle Fields

| Field | Type | Condition | Description |
|-------|------|-----------|-------------|
| `deprecated_date` | date | When status=deprecated | Date deprecation was declared |
| `superseded_by` | string | When status=deprecated | Knowledge ID of the replacement |
| `archived_date` | date | When status=archived | Date document was archived |

### 3.5 Relationship Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `implements` | list[string] | If applicable | Knowledge IDs of standards this document implements |
| `depends_on` | list[object] | If applicable | Knowledge IDs of upstream dependencies |
| `implemented_by` | list[object] | If applicable | References to implementation artifacts |
| `related_apis` | list[object] | If applicable | Related API documents |
| `related_database` | list[object] | If applicable | Related database documents |
| `related_events` | list[object] | If applicable | Related event documents |
| `related_desktop` | list[object] | If applicable | Related desktop documents |
| `related_tests` | list[object] | If applicable | Related test documents |
| `related_adr` | list[object] | If applicable | Related ADR documents |
| `related_products` | list[object] | If applicable | Products that use this capability |
| `related_workflows` | list[object] | If applicable | Workflows that involve this capability |

**Relationship Object Schema:**

```yaml
# Standard relationship
- id: KA-ARCH-002
  title: "Provider Framework Architecture"
  reason: "Optional human-readable reason for the relationship"

# Implementation reference
- type: module | service | test | script | config
  ref: "relative/path/to/artifact"
  line: 42          # Optional: specific line reference
  confidence: high | medium | low

# Product relationship
- product: content-factory | claude-cli | [name]
  role: consumer | producer | owner
```

### 3.6 Evidence Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `evidence` | list[object] | Recommended | References to implementation evidence |

**Evidence Types:**

| Type | Description |
|------|-------------|
| `source_file` | Reference to platform source code |
| `test` | Reference to a test file or test suite |
| `schema` | Reference to a database schema file |
| `config` | Reference to a configuration file |
| `log` | Reference to a validation log or report |
| `adr` | Reference to an ADR that decided this |

### 3.7 Quality Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `phase` | string | Yes | Phase when document was created (e.g., `2.0D.0`) |
| `confidence` | enum | Yes | `high \| medium \| low` — confidence in accuracy |
| `coverage` | object | Recommended | 0–100 score per dimension |

**Confidence Rules:**

| Level | Meaning |
|-------|---------|
| `high` | Document is verified against implementation. Evidence exists. |
| `medium` | Document is based on design intent. Implementation partially verified. |
| `low` | Document is speculative or based on incomplete information. |

---

## 4. Minimal Valid Frontmatter

The minimum required fields for a document to be indexed:

```yaml
---
knowledge_id: KA-ARCH-042
version: "1.0.0"
status: draft
canonical: true
domain: DOM-WORKFLOW
capability: workflow-runtime
type: architecture
owner: Capability Architect
created: 2026-06-29
review_date: 2026-12-29
phase: 2.0D.0
confidence: low
---
```

All other fields default to empty and are populated as the document matures.

---

## 5. Frontmatter Validation Rules

| Rule | Level | Description |
|------|-------|-------------|
| `knowledge_id` present | Error | Document not indexed without it |
| `knowledge_id` unique across repo | Error | Duplicates rejected by audit |
| `version` is valid semver | Error | |
| `status` is valid enum value | Error | |
| `owner` is present | Error | |
| `review_date` is not in the past (for approved docs) | Warning | Health score degraded |
| `review_date` is not in the past > 30 days | Error | Document flagged stale |
| `canonical: true` on duplicate concept | Error | Only one canonical per concept |
| `confidence: high` without evidence | Warning | |
| `implemented_by` references non-existent path | Warning | |
| `related_adr` references non-existent ADR ID | Warning | |

---

## 6. ADR Frontmatter Schema

Architecture Decision Records have a specialized frontmatter:

```yaml
---
knowledge_id: KA-ADR-042
version: "1.0.0"
status: accepted                       # proposed | accepted | deprecated | superseded
canonical: true
domain: DOM-WORKFLOW
capability: workflow-runtime
type: adr
owner: Chief Enterprise Architect
created: 2026-06-29
review_date: ""                        # ADRs are immutable — no review date

# ADR-specific fields
decision_date: 2026-06-29
participants:
  - Chief Enterprise Architect
  - Capability Architect
context_phase: 2.0D.0

superseded_by: ""                      # If this ADR is superseded
supersedes: ""                         # ADR this one replaces

tags:
  - workflow
  - execution-model
---
```

---

## References

- [KNOWLEDGE-ARCHITECTURE-VISION.md](KNOWLEDGE-ARCHITECTURE-VISION.md) — Why metadata matters
- [MICRO-KNOWLEDGE-ARCHITECTURE.md](MICRO-KNOWLEDGE-ARCHITECTURE.md) — Document structure context
- [KNOWLEDGE-INDEX-DESIGN.md](KNOWLEDGE-INDEX-DESIGN.md) — How metadata feeds the indexes
- [KNOWLEDGE-HEALTH-METRICS.md](KNOWLEDGE-HEALTH-METRICS.md) — How metadata drives health scoring
- [REPOSITORY-AUDIT-RULES.md](REPOSITORY-AUDIT-RULES.md) — Enforcement of this standard
