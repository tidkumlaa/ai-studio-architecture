---
knowledge_id: KA-STD-001
version: "1.0.0"
status: approved
owner: Chief Knowledge Architect
phase: 2.0D.0
created: 2026-06-29
review_date: 2026-12-29
canonical: true
implements: KA-MICRO-001
---

# Document Folder Standards

## The Law of Predictable Structure

---

## 1. Purpose

A folder standard eliminates the question "where do I put this?"

When structure is predictable, discovery is instant. When every capability folder contains the same files in the same positions, navigation requires no learning — only orientation to the domain.

This document defines the exact rules for every folder and every file in the architecture repository.

---

## 2. Folder Naming Rules

### 2.1 Capability Folders

```
Format:   [capability-name]
Case:     kebab-case (lowercase, hyphens only)
Length:   2–40 characters
Pattern:  ^[a-z][a-z0-9-]*[a-z0-9]$

Valid:     workflow-runtime
           prompt-os
           provider-framework
           context-intelligence

Invalid:   WorkflowRuntime       (PascalCase)
           PROMPT_OS             (SCREAMING_SNAKE)
           provider framework    (spaces)
           wf                    (too short, not descriptive)
```

### 2.2 Document Filenames

```
Format:   [type].md
Case:     lowercase only

Fixed names (all capabilities):
  README.md
  index.yaml
  overview.md
  architecture.md
  implementation.md
  testing.md
  roadmap.md
  history.md
  references.md

Conditional names (when applicable):
  api.md
  database.md
  events.md
  security.md
  desktop.md
```

**No other filenames are valid at the root of a capability folder.**

Specialized sub-documents use a sub-folder:

```
capabilities/workflow-runtime/
├── api/
│   ├── execution-api.md
│   ├── scheduling-api.md
│   └── index.yaml
├── database/
│   ├── execution-schema.md
│   └── index.yaml
```

### 2.3 Sub-Folder Naming

Sub-folders follow the same kebab-case rule as capability folders.

Sub-folders are permitted **only** when:
- The content exceeds the document size limit
- The content has independent consumers
- The content can be independently versioned

Maximum sub-folder depth: **2 levels below the capability root**.

---

## 3. Required Files — Definitions

### 3.1 README.md

**Purpose:** Navigation entry point. First file a human or AI reads in any folder.

**Required content:**

```markdown
# [Capability Name]

> [One sentence: what this capability does]

## Documents

| Document | Description |
|----------|-------------|
| [overview.md](overview.md) | What this capability is |
| [architecture.md](architecture.md) | How it works |
| [implementation.md](implementation.md) | How to implement it |
| [testing.md](testing.md) | How to test it |
| [roadmap.md](roadmap.md) | What's next |
| [history.md](history.md) | How it evolved |
| [references.md](references.md) | External references |

## Quick Reference

- **Domain:** [domain name]
- **Knowledge ID:** [KA-OVW-NNN]
- **Owner:** [owner role]
- **Status:** [status]

## Related Capabilities

- [capability-name](../capability-name/) — [relationship description]
```

**Rules:**
- No original content. Navigation only.
- No headings below H2.
- Maximum 150 lines.
- No frontmatter required (README is a navigation aid, not a knowledge object).

### 3.2 index.yaml

**Purpose:** Machine-readable index of all documents in this folder. Required by the Knowledge Index system.

**Required schema:**

```yaml
capability: workflow-runtime
domain: DOM-WORKFLOW
knowledge_id: KA-OVW-[NNN]
version: "1.0.0"
status: approved
owner: Capability Architect
last_updated: 2026-06-29

documents:
  - id: KA-OVW-[NNN]
    type: overview
    file: overview.md
    title: "Workflow Runtime Overview"
    status: approved
    lines: 187

  - id: KA-ARCH-[NNN]
    type: architecture
    file: architecture.md
    title: "Workflow Runtime Architecture"
    status: approved
    lines: 342

  # ... all documents in the folder

relationships:
  depends_on:
    - capability: provider-framework
      reason: "Workflow steps delegate to providers"
  implements:
    - standard: KA-STD-001
  related:
    - capability: prompt-os
      reason: "Workflows execute prompt steps"

health:
  score: 0          # Computed by audit tool
  last_computed: "" # Computed by audit tool
```

**Rules:**
- Every document in the folder must appear in `documents:`
- `lines` field must be accurate (enforced by audit)
- `health.score` is computed by the audit tool, never manually set
- `index.yaml` is itself not a knowledge object — no knowledge ID

### 3.3 overview.md

**Purpose:** Explain what this capability is to someone who has never seen it.

**Required sections:**
```
# [Capability Name] Overview
## Purpose
## Scope
## Key Concepts
## Position in the Platform
## Quick Start Navigation
```

**Rules:**
- No implementation detail
- No API signatures
- No database schemas
- Links to all other documents for detail
- Maximum 200 lines

### 3.4 architecture.md

**Purpose:** Explain how this capability works — its design, decisions, and structure.

**Required sections:**
```
# [Capability Name] Architecture
## Design Goals
## Core Design
## Component Model
## Data Flow
## Key Decisions
## Constraints and Trade-offs
## References
```

**Rules:**
- Contains design rationale, not implementation code
- References ADRs for every significant decision
- Maximum 400 lines
- Diagrams referenced from `../../diagrams/` — not embedded inline

### 3.5 implementation.md

**Purpose:** Guide implementers through building or modifying this capability.

**Required sections:**
```
# [Capability Name] Implementation Guide
## Prerequisites
## Architecture Contract
## Implementation Steps
## Extension Points
## Common Mistakes
## Validation
```

**Rules:**
- References `architecture.md` for design context — does not repeat it
- No API or database content (link to those documents)
- Maximum 300 lines

### 3.6 testing.md

**Purpose:** Define the test strategy and coverage requirements for this capability.

**Required sections:**
```
# [Capability Name] Testing
## Test Strategy
## Unit Test Coverage
## Integration Test Coverage
## Contract Tests
## Performance Tests
## Test Data
## Coverage Targets
```

**Rules:**
- References actual test locations in `platform/` (not architecture repo)
- Maximum 200 lines

### 3.7 roadmap.md

**Purpose:** Define future work, enhancements, and known gaps.

**Required sections:**
```
# [Capability Name] Roadmap
## Current State
## Short-term (Next Phase)
## Medium-term (2–3 Phases)
## Long-term
## Known Gaps
## Blocked Items
```

**Rules:**
- All dates are explicit (not "next quarter")
- Blocked items reference the blocking dependency
- Maximum 200 lines

### 3.8 history.md

**Purpose:** Record the evolution of this capability over time.

**Required sections:**
```
# [Capability Name] History
## Decision Log
## Version History
## Ownership History
## Migration Notes
```

**Rules:**
- Entries are chronological (newest first)
- Each entry includes date and rationale
- References ADRs and decisions by ID
- Maximum 300 lines

### 3.9 references.md

**Purpose:** Collect all external and cross-capability references.

**Required sections:**
```
# [Capability Name] References
## Related Capabilities
## Related Standards
## ADRs
## External Resources
## Superseded Documents
```

**Rules:**
- Links only — no prose beyond one-sentence descriptions
- All links are validated by audit tool
- Maximum 100 lines

---

## 4. Conditional Files — Definitions

### 4.1 api.md

Required when the capability exposes an API surface (REST, gRPC, IPC, or internal).

**Required sections:**
```
# [Capability Name] API
## API Overview
## Endpoints / Operations
## Request/Response Models
## Error Codes
## Authentication
## Rate Limits
## Versioning
```

### 4.2 database.md

Required when the capability owns a database schema.

**Required sections:**
```
# [Capability Name] Database
## Schema Overview
## Tables / Collections
## Indexes
## Relationships
## Migration Strategy
## Query Patterns
```

### 4.3 events.md

Required when the capability produces or consumes events.

**Required sections:**
```
# [Capability Name] Events
## Event Overview
## Events Produced
## Events Consumed
## Event Schema
## Ordering Guarantees
## Replay Strategy
```

### 4.4 security.md

Required when the capability has security-sensitive behavior.

**Required sections:**
```
# [Capability Name] Security
## Threat Model
## Authentication
## Authorization
## Data Classification
## Audit Logging
## Compliance Requirements
```

### 4.5 desktop.md

Required when the capability has a desktop UI representation.

**Required sections:**
```
# [Capability Name] Desktop
## UI Overview
## Panel Definitions
## User Interactions
## State Management
## Accessibility
## Design System References
```

---

## 5. File Presence Matrix

| File | Capability with No UI | Capability with UI | Platform Standard |
|------|--------------------|-------------------|-------------------|
| README.md | Required | Required | Required |
| index.yaml | Required | Required | Required |
| overview.md | Required | Required | Required |
| architecture.md | Required | Required | Required |
| implementation.md | Required | Required | Required |
| testing.md | Required | Required | Required |
| roadmap.md | Required | Required | Required |
| history.md | Required | Required | Required |
| references.md | Required | Required | Required |
| api.md | If has API | If has API | If has API |
| database.md | If has DB | If has DB | If has DB |
| events.md | If has events | If has events | If has events |
| security.md | If security-sensitive | If security-sensitive | If security-sensitive |
| desktop.md | Not applicable | Required | Not applicable |

---

## 6. Folder Validation Rules

The following conditions are audit violations:

| Rule | Violation Type |
|------|---------------|
| Missing README.md | Error |
| Missing index.yaml | Error |
| Missing overview.md | Error |
| Missing architecture.md | Error |
| Missing implementation.md | Error |
| Missing testing.md | Error |
| Missing roadmap.md | Error |
| Missing history.md | Error |
| Missing references.md | Error |
| File not in index.yaml | Error |
| Filename not in approved list | Warning |
| Sub-folder depth > 2 | Error |
| README.md contains original content | Warning |
| Any .md file > 500 lines | Error |
| index.yaml lines count mismatch | Warning |

---

## 7. Archive Folder Structure

```
archive/
├── README.md
├── index.yaml           # Registry of all archived documents
├── v1.5.5/              # Version-scoped archives
│   └── [original files preserved]
├── v2.0/
│   └── [original files preserved]
├── superseded/          # Documents replaced by canonical sources
│   └── [with redirect notes]
└── orphaned/            # Documents that had no capability mapping
    └── [with triage notes]
```

Archive documents are **read-only**. No content modifications after archival. If an archived concept is revived, a new document is created in the active capability tree.

---

## References

- [KNOWLEDGE-ARCHITECTURE-VISION.md](KNOWLEDGE-ARCHITECTURE-VISION.md) — Strategic context
- [MICRO-KNOWLEDGE-ARCHITECTURE.md](MICRO-KNOWLEDGE-ARCHITECTURE.md) — Size limits and composition rules
- [METADATA-STANDARD.md](METADATA-STANDARD.md) — Frontmatter schema for all files
- [REPOSITORY-AUDIT-RULES.md](REPOSITORY-AUDIT-RULES.md) — Automated enforcement
