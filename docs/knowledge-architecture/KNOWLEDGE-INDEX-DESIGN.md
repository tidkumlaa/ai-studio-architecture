---
knowledge_id: KA-IDX-001
version: "1.0.0"
status: approved
owner: Chief Knowledge Architect
phase: 2.0D.0
created: 2026-06-29
review_date: 2026-12-29
canonical: true
implements:
  - KA-VISION-001
  - KA-STD-002
---

# Knowledge Index Design

## Everything Must Be Discoverable Without Scanning Directories

---

## 1. The Discovery Problem

786 markdown files. No index. No registry. No capability map.

The only discovery mechanism is `find . -name "*.md"` followed by reading filenames and hoping. This is archaeology, not navigation.

The knowledge index system solves this with seven complementary indexes that together make every knowledge object discoverable from multiple entry points — by domain, capability, technology, product, relationship, or document type — without reading a single file or traversing a directory tree.

---

## 2. Index Architecture

### 2.1 Index Hierarchy

```
KNOWLEDGE-INDEX.md                ← Master registry (human-readable entry point)
index.yaml                        ← Machine-readable master registry

Domain Index                      ← By domain (DOM-WORKFLOW, DOM-AI, etc.)
Capability Index                  ← By capability (workflow-runtime, prompt-os, etc.)
Technology Index                  ← By technology (python, sqlite, claude, etc.)
Product Index                     ← By product (content-factory, claude-cli, etc.)
Document Index                    ← By document type (architecture, api, adr, etc.)
Relationship Index                ← By relationship type (implements, depends_on, etc.)
Knowledge Registry                ← Flat list of all knowledge IDs
```

### 2.2 Index Principles

- **No index is authoritative alone.** Each index is a view. The knowledge objects themselves are authoritative.
- **Indexes are generated, not maintained.** The audit tool regenerates all indexes from frontmatter. Manual index editing is forbidden.
- **Indexes are read-only for humans.** Humans navigate indexes; they do not edit them.
- **Indexes degrade gracefully.** If a knowledge object is missing from an index, the audit tool flags it. The object is not lost.

---

## 3. Master Knowledge Index

### 3.1 KNOWLEDGE-INDEX.md

The master human-readable index. Lives at `architecture/KNOWLEDGE-INDEX.md`.

**Structure:**

```markdown
# AI Studio Knowledge Index

> Generated: 2026-06-29 | Documents: 786 | Capabilities: 12 | Health Score: 67/100

## Quick Navigation

| I want to understand... | Go to |
|------------------------|-------|
| How workflows execute | [Workflow Runtime](capabilities/workflow-runtime/overview.md) |
| How prompts are managed | [Prompt OS](capabilities/prompt-os/overview.md) |
| How providers are selected | [Intelligent Routing](capabilities/intelligent-routing/overview.md) |
| How conversations work | [Conversation Intelligence](capabilities/conversation-intelligence/overview.md) |
| How the platform is structured | [Platform Architecture](capabilities/platform-core/architecture.md) |
| All architecture decisions | [ADR Index](adr/README.md) |

## By Domain

### DOM-WORKFLOW — Workflow Runtime
- [Workflow Runtime](capabilities/workflow-runtime/) — [KA-OVW-001]
  - architecture.md [KA-ARCH-001] ✓ approved
  - api.md [KA-API-001] ✓ approved
  - ...

### DOM-AI — AI Runtime OS
...

## By Capability

[Full capability listing — generated]

## Recently Updated

[Last 10 modified documents — generated]

## Stale Documents

[Documents past review_date — generated]

## Health Dashboard

[Health scores by domain — generated]
```

### 3.2 index.yaml (Master)

Machine-readable master index at `architecture/index.yaml`.

```yaml
meta:
  generated: 2026-06-29T00:00:00Z
  generator_version: "1.0.0"
  total_documents: 786
  total_capabilities: 12
  total_domains: 10
  health_score: 67

domains:
  DOM-WORKFLOW:
    name: "Workflow Runtime"
    path: capabilities/workflow-runtime/
    owner: "Capability Architect"
    document_count: 14
    health_score: 82
    capabilities:
      - workflow-runtime

  DOM-AI:
    name: "AI Runtime OS"
    path: capabilities/ai-runtime-os/
    ...

capabilities:
  workflow-runtime:
    domain: DOM-WORKFLOW
    path: capabilities/workflow-runtime/
    index: capabilities/workflow-runtime/index.yaml
    knowledge_ids:
      - KA-OVW-001
      - KA-ARCH-001
      - KA-API-001
      - KA-DB-001
      - KA-EVT-001
      - KA-SEC-001
      - KA-IMPL-001
      - KA-TEST-001
      - KA-ROAD-001
      - KA-HIST-001
      - KA-REF-001

knowledge_registry:
  KA-OVW-001:
    title: "Workflow Runtime Overview"
    file: capabilities/workflow-runtime/overview.md
    type: overview
    status: approved
    domain: DOM-WORKFLOW
    capability: workflow-runtime
    owner: "Capability Architect"
    version: "1.2.0"
    review_date: "2026-12-29"
    health_score: 88

  KA-ARCH-001:
    title: "Workflow Runtime Architecture"
    file: capabilities/workflow-runtime/architecture.md
    ...
```

---

## 4. Domain Index

### 4.1 Purpose

Navigate the knowledge system by architectural domain. Each domain index is the authoritative entry point for everything within that domain.

### 4.2 Location

`architecture/DOMAIN-INDEX.md` (human-readable)
`architecture/domain-index.yaml` (machine-readable)

### 4.3 Domain Index Entry Schema

```yaml
domain:
  id: DOM-WORKFLOW
  name: "Workflow Runtime"
  description: "Workflow definition, execution, orchestration, and scheduling"
  owner: "Chief Enterprise Architect"
  status: active

  capabilities:
    - id: workflow-runtime
      path: capabilities/workflow-runtime/
      overview: KA-OVW-001
      architecture: KA-ARCH-001

  cross_domain_dependencies:
    - domain: DOM-AI
      reason: "Workflows call AI providers"
    - domain: DOM-PROMPT
      reason: "Workflow steps execute prompts"
    - domain: DOM-PROVIDER
      reason: "Workflows delegate to provider framework"

  products:
    - content-factory
    - claude-cli

  adr_count: 8
  total_documents: 48
  health_score: 82
```

---

## 5. Capability Index

### 5.1 Purpose

Navigate to any capability and see all of its knowledge objects in one view.

### 5.2 Location

Generated into each capability's `index.yaml`. Aggregated in `architecture/CAPABILITY-INDEX.md`.

### 5.3 Capability Index Entry Schema

```yaml
capability: workflow-runtime
domain: DOM-WORKFLOW
status: active
version: "2.0.0"
owner: "Capability Architect"
overview: KA-OVW-001

documents:
  core:
    overview: { id: KA-OVW-001, status: approved, lines: 187 }
    architecture: { id: KA-ARCH-001, status: approved, lines: 342 }
    implementation: { id: KA-IMPL-001, status: approved, lines: 289 }
    testing: { id: KA-TEST-001, status: approved, lines: 198 }
    roadmap: { id: KA-ROAD-001, status: approved, lines: 145 }
    history: { id: KA-HIST-001, status: approved, lines: 234 }
    references: { id: KA-REF-001, status: approved, lines: 67 }

  optional:
    api: { id: KA-API-001, status: approved, lines: 267 }
    database: { id: KA-DB-001, status: approved, lines: 198 }
    events: { id: KA-EVT-001, status: approved, lines: 156 }
    security: { id: KA-SEC-001, status: approved, lines: 178 }

  missing: []  # Files required by standards but not present

adrs:
  - { id: KA-ADR-012, title: "Executor Selection", status: accepted }
  - { id: KA-ADR-017, title: "Retry Strategy", status: accepted }

relationships:
  depends_on: [DOM-AI, DOM-PROMPT, DOM-PROVIDER]
  consumed_by: [content-factory, claude-cli]

health:
  score: 82
  missing_files: 0
  oversized_files: 0
  stale_files: 1
  broken_references: 0
  missing_evidence: 2
```

---

## 6. Technology Index

### 6.1 Purpose

Navigate to all knowledge objects that involve a specific technology.

### 6.2 Location

`architecture/TECHNOLOGY-INDEX.md`
`architecture/technology-index.yaml`

### 6.3 Technology Index Schema

```yaml
technologies:
  python:
    version: "3.11+"
    capabilities:
      - workflow-runtime
      - prompt-os
      - provider-framework
    documents:
      - id: KA-IMPL-001
        title: "Workflow Runtime Implementation"
        relevance: primary

  sqlite:
    capabilities:
      - workflow-runtime
      - conversation-intelligence
    documents:
      - id: KA-DB-001
        title: "Workflow Execution Schema"
        relevance: primary

  claude:
    provider: Anthropic
    models: ["claude-sonnet-4-6", "claude-opus-4-8", "claude-haiku-4-5"]
    capabilities:
      - provider-framework
      - intelligent-routing
    documents:
      - id: KA-ARCH-005
        title: "Provider Framework Architecture"
        relevance: primary

  openai:
    provider: OpenAI
    capabilities:
      - provider-framework
    documents:
      - id: KA-ARCH-005
        relevance: secondary

  tauri:
    capabilities:
      - ai-runtime-os
    documents:
      - id: KA-DSK-001
```

---

## 7. Product Index

### 7.1 Purpose

Navigate from a product to all capabilities, APIs, and knowledge objects it consumes.

### 7.2 Location

`architecture/PRODUCT-INDEX.md`
`architecture/product-index.yaml`

### 7.3 Product Index Entry Schema

```yaml
products:
  content-factory:
    name: "Content Factory"
    status: active
    phase: "v1.5.5"
    description: "AI-powered content generation product"

    capabilities_consumed:
      - capability: workflow-runtime
        role: primary-consumer
        integration_doc: KA-ARCH-001
      - capability: prompt-os
        role: primary-consumer
      - capability: provider-framework
        role: primary-consumer
      - capability: context-intelligence
        role: consumer

    apis_used:
      - id: KA-API-001
        title: "Workflow Execution API"
      - id: KA-API-003
        title: "Prompt OS API"

    events_produced:
      - id: KA-EVT-005
        title: "Content Generation Events"

    adrs:
      - id: KA-ADR-020
        title: "Content Factory Workflow Model"

  claude-cli:
    name: "Claude CLI"
    status: active
    ...
```

---

## 8. Document Index

### 8.1 Purpose

Find all documents of a specific type across all capabilities.

### 8.2 Location

`architecture/DOCUMENT-INDEX.md`
`architecture/document-index.yaml`

### 8.3 Document Index Schema

```yaml
by_type:
  architecture:
    count: 12
    documents:
      - id: KA-ARCH-001
        capability: workflow-runtime
        title: "Workflow Runtime Architecture"
        status: approved
        lines: 342
        health: 88

      - id: KA-ARCH-002
        capability: prompt-os
        ...

  api:
    count: 8
    documents:
      ...

  adr:
    count: 47
    documents:
      - id: KA-ADR-001
        title: "Initial Architecture Approach"
        status: accepted
        decision_date: 2025-01-15
      ...

by_status:
  approved: 156
  draft: 23
  deprecated: 47
  archived: 560   # Historical documents
```

---

## 9. Relationship Index

### 9.1 Purpose

Navigate the dependency and relationship graph without reading individual documents.

### 9.2 Location

`architecture/RELATIONSHIP-INDEX.md`
`architecture/relationship-index.yaml`

### 9.3 Relationship Index Schema

```yaml
relationships:
  implements:
    # Who implements what standard
    - from: KA-ARCH-001
      to: KA-STD-001
      type: implements

  depends_on:
    # Upstream dependencies
    - from: KA-ARCH-001
      to: KA-ARCH-005
      reason: "Workflow executor delegates to provider"

  supersedes:
    # Document evolution chain
    - from: KA-ARCH-001
      to: KA-ARCH-000
      deprecated_date: 2026-01-15

  implemented_by:
    # Architecture to implementation mapping
    - from: KA-ARCH-001
      to:
        type: module
        ref: platform/workflow-runtime/src/executor.py

capability_graph:
  workflow-runtime:
    depends_on:
      - provider-framework
      - prompt-os
      - context-intelligence
    consumed_by:
      - content-factory
      - claude-cli
    implements:
      - KA-STD-001

orphans:
  # Knowledge IDs with no relationships (health concern)
  - KA-ARCH-099
  - KA-OVW-022
```

---

## 10. Knowledge Registry

### 10.1 Purpose

The flat, authoritative list of every knowledge ID in the system. Used for ID collision detection, completeness checking, and audit validation.

### 10.2 Location

`architecture/KNOWLEDGE-REGISTRY.yaml`

### 10.3 Registry Schema

```yaml
meta:
  total: 786
  last_updated: 2026-06-29
  next_available:
    KA-ARCH: "KA-ARCH-013"
    KA-API: "KA-API-009"
    KA-ADR: "KA-ADR-048"
    # ... per type

registry:
  KA-VIS-001:
    title: "Knowledge Architecture Vision"
    file: docs/knowledge-architecture/KNOWLEDGE-ARCHITECTURE-VISION.md
    type: vision
    status: approved
    domain: DOM-KNOWLEDGE
    canonical: true
    version: "1.0.0"
    created: 2026-06-29

  KA-ARCH-001:
    title: "Workflow Runtime Architecture"
    file: capabilities/workflow-runtime/architecture.md
    ...

collisions: []       # Auto-detected by audit tool — should always be empty
missing_files: []    # IDs registered but file not found
orphaned_files: []   # Files with knowledge IDs not in registry
```

---

## 11. Index Generation Rules

All indexes are **generated artifacts**, never hand-written.

| Rule | Description |
|------|-------------|
| Source of truth | Knowledge object frontmatter |
| Generation trigger | Pre-commit hook, or manual `make indexes` |
| Generation tool | `tools/generate-indexes.py` (to be implemented in Phase 2.0D.1) |
| Failure behavior | Build fails if any index is stale |
| Conflict resolution | Frontmatter wins — index is always regenerated |
| Index staleness check | `index_generated_at` timestamp compared to any frontmatter `modified` date |

---

## References

- [METADATA-STANDARD.md](METADATA-STANDARD.md) — Frontmatter that feeds the indexes
- [KNOWLEDGE-ARCHITECTURE-VISION.md](KNOWLEDGE-ARCHITECTURE-VISION.md) — Why discovery matters
- [KNOWLEDGE-NAVIGATION.md](KNOWLEDGE-NAVIGATION.md) — How humans use these indexes
- [RELATIONSHIP-MODEL.md](RELATIONSHIP-MODEL.md) — Relationships captured in indexes
- [REPOSITORY-AUDIT-RULES.md](REPOSITORY-AUDIT-RULES.md) — Index freshness enforcement
