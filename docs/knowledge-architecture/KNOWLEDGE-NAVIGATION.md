---
knowledge_id: KA-STD-005
version: "1.0.0"
status: approved
owner: Chief Knowledge Architect
phase: 2.0D.0
created: 2026-06-29
review_date: 2026-12-29
canonical: true
implements:
  - KA-VISION-001
---

# Knowledge Navigation

## Navigate by Capability, Not by Filename

---

## 1. The Navigation Problem

The current repository has one navigation mechanism: knowing the filename.

If you know to look for `AI-STUDIO-MASTER-ARCHITECTURE.md`, you find 8,000 lines of everything. If you don't know the filename, you search — and search returns twenty results with no indication of which is canonical, current, or most relevant.

This is **filename-centric navigation**: the burden is on the human to know what to look for before they can find it.

The knowledge architecture inverts this:

**Capability-centric navigation**: You enter with a concept — "Workflow Runtime," "Provider Manager," "Prompt OS" — and the system delivers the complete, connected knowledge graph for that concept.

---

## 2. Navigation Entry Points

### 2.1 Concept Entry (Primary)

The user navigates from a concept name to its complete knowledge context.

```
User enters: "Workflow Runtime"
System delivers:
  → capabilities/workflow-runtime/README.md        (orientation)
  → capabilities/workflow-runtime/overview.md      (what it is)
  → capabilities/workflow-runtime/architecture.md  (how it works)
  → capabilities/workflow-runtime/api.md           (API surface)
  → capabilities/workflow-runtime/index.yaml       (machine-readable index)
  → Related: prompt-os, provider-framework, context-intelligence
  → ADRs: KA-ADR-012, KA-ADR-017
  → Products: content-factory (consumer), claude-cli (consumer)
```

### 2.2 Domain Entry

The user navigates from a domain to all capabilities within it.

```
User enters: "AI Domain" or "DOM-AI"
System delivers:
  → DOMAIN-INDEX.md#dom-ai
  → capabilities/ai-runtime-os/
  → capabilities/intelligent-routing/
  → Cross-domain dependencies: DOM-PROVIDER, DOM-WORKFLOW
  → Domain ADRs
  → Domain health score
```

### 2.3 Product Entry

The user navigates from a product to all capabilities, APIs, and workflows it uses.

```
User enters: "Content Factory"
System delivers:
  → products/content-factory/
  → Capabilities consumed: workflow-runtime, prompt-os, provider-framework, context-intelligence
  → APIs used: KA-API-001, KA-API-003
  → Events produced: KA-EVT-005
  → Release history: docs/1.5.5/
```

### 2.4 Decision Entry

The user navigates from an architecture decision to its full impact.

```
User enters: "ADR-012" or "executor selection"
System delivers:
  → adr/KA-ADR-012.md
  → Affected documents: KA-ARCH-001, KA-IMPL-001
  → Alternative options considered
  → Superseded by: nothing (current)
  → Related ADRs: KA-ADR-017 (retry strategy)
```

### 2.5 Technology Entry

The user navigates from a technology to all capabilities and documents that use it.

```
User enters: "SQLite" or "Python"
System delivers:
  → TECHNOLOGY-INDEX.md#sqlite
  → Capabilities: workflow-runtime, conversation-intelligence
  → Documents: KA-DB-001, KA-DB-002
  → Schema files in platform/
```

---

## 3. Navigation Patterns by Use Case

### 3.1 "I'm new to this system — where do I start?"

```
Entry: README.md (root)
Path:
  README.md
  → KNOWLEDGE-INDEX.md (master index)
  → DOMAIN-INDEX.md (choose a domain)
  → capabilities/[capability]/README.md (choose a capability)
  → capabilities/[capability]/overview.md (understand the capability)
  → capabilities/[capability]/architecture.md (understand the design)
```

### 3.2 "I need to implement [capability]"

```
Entry: capabilities/[capability]/
Path:
  overview.md      → Understand the concept
  architecture.md  → Understand the design
  api.md           → Understand the contracts
  implementation.md → Follow the implementation guide
  testing.md       → Know what tests to write
```

### 3.3 "I need to modify [capability] — what else will I break?"

```
Entry: capabilities/[capability]/index.yaml
Path:
  index.yaml → relationships.consumed_by → products affected
  index.yaml → relationships.depends_on  → upstream dependencies
  RELATIONSHIP-INDEX.md → reverse dependencies (who depends on me)
  events.md → who consumes my events
  api.md    → who calls my API
```

### 3.4 "Why was this decision made?"

```
Entry: Document with a design decision
Path:
  architecture.md → decided_by: [ADR IDs]
  → adr/[ADR ID].md → Full decision context
  → ADR: options_considered, decision, rationale
  → ADR: affects → other impacted documents
```

### 3.5 "What's the current status of [capability]?"

```
Entry: capabilities/[capability]/index.yaml
Contents:
  health.score
  health.missing_files
  health.stale_files
  health.broken_references
  documents[*].status
```

### 3.6 "I'm an AI assistant — how do I load context about [capability]?"

```
Load sequence (minimizing context consumption):
  1. capabilities/[capability]/index.yaml       → structure and IDs (small)
  2. capabilities/[capability]/overview.md      → concept understanding
  3. capabilities/[capability]/architecture.md  → design understanding
  4. Load specific docs based on the query type:
     - API question → api.md
     - DB question  → database.md
     - Event question → events.md
     - Security question → security.md
```

---

## 4. Canonical Navigation Map

The following is the navigation map for known AI Studio capabilities.

### 4.1 Core Platform Capabilities

| User Intent | Entry Point | Primary Document |
|-------------|-------------|-----------------|
| Workflow execution and orchestration | `capabilities/workflow-runtime/` | `architecture.md` |
| Prompt management and versioning | `capabilities/prompt-os/` | `architecture.md` |
| Provider selection and routing | `capabilities/intelligent-routing/` | `architecture.md` |
| Provider abstraction and contracts | `capabilities/provider-framework/` | `architecture.md` |
| Provider lifecycle management | `capabilities/provider-management/` | `architecture.md` |
| Conversation history and management | `capabilities/conversation-intelligence/` | `architecture.md` |
| Context window optimization | `capabilities/context-intelligence/` | `architecture.md` |
| Core AI execution runtime | `capabilities/ai-runtime-os/` | `architecture.md` |

### 4.2 Knowledge Architecture Capabilities

| User Intent | Entry Point | Primary Document |
|-------------|-------------|-----------------|
| Knowledge architecture design | `capabilities/knowledge-architecture/` | This document |
| Document standards | `knowledge-architecture/DOCUMENT-FOLDER-STANDARDS.md` | Self |
| Metadata schema | `knowledge-architecture/METADATA-STANDARD.md` | Self |
| Index design | `knowledge-architecture/KNOWLEDGE-INDEX-DESIGN.md` | Self |

### 4.3 Products

| User Intent | Entry Point | Primary Document |
|-------------|-------------|-----------------|
| Content Factory product | `products/content-factory/` | `overview.md` |
| Claude CLI product | `products/claude-cli/` | `overview.md` |

### 4.4 Platform Standards

| User Intent | Entry Point | Primary Document |
|-------------|-------------|-----------------|
| All architecture decisions | `adr/` | `README.md` |
| Platform contracts | `platform/contracts/` | `overview.md` |
| Security model | `platform/security/` | `architecture.md` |
| Platform standards | `platform/standards/` | `overview.md` |

---

## 5. README Navigation Template

Every capability `README.md` implements this navigation template:

```markdown
# [Capability Name]

> [One sentence: what this capability does and why it exists]

**Domain:** [DOM-XXX] | **Owner:** [Role] | **Status:** [status]

---

## Start Here

| I want to... | Go to |
|-------------|-------|
| Understand what this capability is | [overview.md](overview.md) |
| Understand how it works | [architecture.md](architecture.md) |
| Call its API | [api.md](api.md) |
| Implement it | [implementation.md](implementation.md) |
| Write tests for it | [testing.md](testing.md) |
| See what's planned | [roadmap.md](roadmap.md) |
| Understand why it was designed this way | [history.md](history.md) |
| Find related links | [references.md](references.md) |

---

## This Capability Depends On

| Capability | Why |
|-----------|-----|
| [provider-framework](../provider-framework/) | [reason] |
| [prompt-os](../prompt-os/) | [reason] |

## This Capability Is Used By

| Consumer | How |
|---------|-----|
| [content-factory](../../products/content-factory/) | [role] |
| [claude-cli](../../products/claude-cli/) | [role] |

---

## Quick Facts

- **Knowledge ID:** [KA-OVW-NNN]
- **Version:** [X.Y.Z]
- **Phase:** [phase]
- **Health Score:** [computed by audit tool]
```

---

## 6. Domain README Navigation Template

```markdown
# [Domain Name] Domain

> [One sentence: what architectural concern this domain addresses]

**Domain ID:** DOM-XXX | **Owner:** [Role]

---

## Capabilities in This Domain

| Capability | Description | Status |
|-----------|-------------|--------|
| [workflow-runtime](../capabilities/workflow-runtime/) | [description] | active |
| [intelligent-routing](../capabilities/intelligent-routing/) | [description] | active |

## Domain Dependencies

| Depends On | Reason |
|-----------|--------|
| [DOM-PROVIDER](../DOMAIN-INDEX.md#dom-provider) | [reason] |

## This Domain Is Used By

| Consumer | Role |
|---------|------|
| DOM-PRODUCT | All products consume this domain |

## Domain Health

- Documents: [N]
- Health Score: [N/100]
- Stale Documents: [N]
- Open ADRs: [N]
```

---

## 7. Navigation for AI Assistants

The knowledge architecture is explicitly designed for AI-assistant consumption. This section defines the recommended loading strategy for AI assistants working in this repository.

### 7.1 Query-Type to Load-Order Mapping

| Query Type | Load Order |
|-----------|-----------|
| "What is X?" | `overview.md` → `architecture.md` |
| "How does X work?" | `architecture.md` → `implementation.md` |
| "What API does X expose?" | `api.md` → `architecture.md` (if needed) |
| "What database does X use?" | `database.md` → `architecture.md` (if needed) |
| "What events does X produce?" | `events.md` |
| "Why was X designed this way?" | `history.md` → related `adr/` documents |
| "What depends on X?" | `index.yaml` (relationships) → `RELATIONSHIP-INDEX.md` |
| "What does X depend on?" | `index.yaml` (depends_on) → target `overview.md` |
| "Is X implemented?" | `index.yaml` (evidence, coverage) |
| "What's the status of X?" | `index.yaml` (health) |

### 7.2 Context Budget Optimization

For AI assistants with limited context windows:

**Tier 1 — Orientation (< 2K tokens):**
- Load `capabilities/[capability]/index.yaml`
- This gives: all document IDs, relationships, health, status

**Tier 2 — Understanding (< 8K tokens):**
- Load `overview.md` + `architecture.md`
- This gives: what it is and how it works

**Tier 3 — Deep Dive (< 20K tokens):**
- Load specific document based on query type
- Full architecture context

**Never load:**
- `archive/` — historical only, no current truth
- `docs/1.5.5/`, `docs/2.0/` — historical only
- `KNOWLEDGE-INDEX.md` as first step — too large; use `index.yaml` instead

---

## 8. Navigation Anti-Patterns

These navigation approaches are explicitly forbidden in the refactored repository:

| Anti-Pattern | Why Forbidden | Correct Approach |
|-------------|--------------|-----------------|
| Opening `AI-STUDIO-MASTER-ARCHITECTURE.md` | Monolithic, stale, unstructured | Navigate to specific capability |
| Searching for a term across all files | Returns too many results, no context | Use KNOWLEDGE-INDEX or TECHNOLOGY-INDEX |
| Reading `docs/2.0/` for current architecture | Historical only | Use `capabilities/` tree |
| Using filename to infer document type | Filenames are standardized — use `type` in frontmatter | Check `index.yaml` |
| Opening `README.md` expecting content | READMEs are navigation aids only | Follow links in README |

---

## References

- [KNOWLEDGE-ARCHITECTURE-VISION.md](KNOWLEDGE-ARCHITECTURE-VISION.md) — Discovery principle
- [KNOWLEDGE-INDEX-DESIGN.md](KNOWLEDGE-INDEX-DESIGN.md) — Index structures used in navigation
- [DOCUMENT-FOLDER-STANDARDS.md](DOCUMENT-FOLDER-STANDARDS.md) — Folder structure that makes navigation predictable
- [RELATIONSHIP-MODEL.md](RELATIONSHIP-MODEL.md) — Relationships traversed during navigation
