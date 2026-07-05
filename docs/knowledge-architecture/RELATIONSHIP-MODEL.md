---
knowledge_id: KA-STD-004
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

# Relationship Model

## Every Knowledge Object Must Define Its Connections

---

## 1. Why Relationships Matter

A document without declared relationships is an island.

Islands accumulate silently. They are not maintained because no one knows they depend on something that changed. They are not found because nothing points to them. They are not trusted because no one can trace their origins.

The relationship model transforms the architecture from a collection of files into a **navigable knowledge graph**. Every relationship declared in frontmatter is an edge in that graph. The graph enables:

- Forward navigation (what does this affect?)
- Reverse navigation (what depends on this?)
- Traceability (what decision produced this?)
- Coverage analysis (what is not yet designed?)
- Impact analysis (if I change X, what else changes?)

---

## 2. Knowledge Object Types

All nodes in the relationship graph are typed. Every node has a unique identity.

| Node Type | Namespace | Examples |
|-----------|-----------|---------|
| Document | `KA-[TYPE]-NNN` | `KA-ARCH-001`, `KA-API-007` |
| Capability | `CAP-[name]` | `CAP-workflow-runtime` |
| Module | `MOD-[name]` | `MOD-executor`, `MOD-scheduler` |
| API | `API-[name]` | `API-workflow-execution-v1` |
| Database | `DB-[name]` | `DB-workflow-state` |
| Workflow | `WF-[NNN]` | `WF-001` |
| Prompt | `PROMPT-[name]` | `PROMPT-execution-planner` |
| Desktop | `UI-[name]` | `UI-workflow-panel` |
| ADR | `KA-ADR-NNN` | `KA-ADR-012` |
| Decision | `DEC-[name]` | `DEC-retry-strategy` |
| Provider | `PROV-[name]` | `PROV-anthropic`, `PROV-openai` |
| Agent | `AGENT-[name]` | `AGENT-execution-planner` |
| Runtime | `RUNTIME-[name]` | `RUNTIME-ai-os` |
| Product | `PROD-[name]` | `PROD-content-factory` |
| Standard | `KA-STD-NNN` | `KA-STD-001` |

---

## 3. Relationship Types

### 3.1 Core Relationship Types

| Relationship | Direction | Meaning |
|-------------|-----------|---------|
| `implements` | A → B | A implements the standard/contract B |
| `depends_on` | A → B | A requires B to function |
| `supersedes` | A → B | A replaces the older B |
| `references` | A → B | A refers to B for context (no dependency) |
| `extends` | A → B | A adds to B without modifying it |
| `constrains` | A → B | A imposes rules or limits on B |

### 3.2 Artifact-Linking Relationships

| Relationship | Direction | Meaning |
|-------------|-----------|---------|
| `implemented_by` | DOC → CODE | Architecture document has implementation in code |
| `tested_by` | DOC → TEST | Architecture document has tests that verify it |
| `documented_by` | CODE → DOC | Code artifact is documented by this document |
| `specified_by` | API → DOC | API specification is defined by this document |
| `stores_in` | CAP → DB | Capability persists data in this database |
| `produces_event` | CAP → EVT | Capability emits this event |
| `consumes_event` | CAP → EVT | Capability reacts to this event |

### 3.3 Decision Relationships

| Relationship | Direction | Meaning |
|-------------|-----------|---------|
| `decided_by` | DOC → ADR | This document's design was decided by this ADR |
| `motivates` | ADR → DOC | This ADR motivated the creation of this document |
| `overrides` | ADR → ADR | This ADR supersedes the previous decision |

### 3.4 Product Relationships

| Relationship | Direction | Meaning |
|-------------|-----------|---------|
| `consumed_by` | CAP → PROD | Capability is used by product |
| `owns` | PROD → CAP | Product is the primary owner of this capability |
| `integrates` | PROD → API | Product integrates via this API |

### 3.5 Agent and Provider Relationships

| Relationship | Direction | Meaning |
|-------------|-----------|---------|
| `executes_with` | WORKFLOW → AGENT | Workflow is executed by this agent |
| `routes_to` | DOC → PROV | Routing document targets this provider |
| `wraps` | CAP → PROV | Capability wraps this provider |

---

## 4. Relationship Declaration — Full Schema

All relationships are declared in document frontmatter:

```yaml
# ─── FORWARD RELATIONSHIPS (this document → others) ──────────────────────────

implements:
  - KA-STD-001          # Simple: just the ID
  - KA-STD-002

depends_on:
  - id: KA-ARCH-005
    type: depends_on
    reason: "Workflow executor requires provider framework"
    criticality: required   # required | optional | enhances
    verified: true

extends:
  - id: KA-ARCH-000
    type: extends
    reason: "Adds scheduling layer to base workflow design"

references:
  - id: KA-ADR-012
    type: references
    context: "The executor selection decision"

# ─── ARTIFACT LINKS ──────────────────────────────────────────────────────────

implemented_by:
  - type: module
    ref: platform/workflow-runtime/src/executor.py
    namespace: MOD-executor
    confidence: high
    verified: true
    last_verified: 2026-06-29

  - type: module
    ref: platform/workflow-runtime/src/scheduler.py
    namespace: MOD-scheduler
    confidence: high
    verified: true

tested_by:
  - type: test_suite
    ref: platform/workflow-runtime/tests/
    coverage_percent: 78
    last_run: 2026-06-28

produces_event:
  - id: KA-EVT-001
    event: workflow.execution.started
  - id: KA-EVT-002
    event: workflow.execution.completed

stores_in:
  - id: KA-DB-001
    database: workflow-state
    tables: [workflow_runs, workflow_steps]

# ─── DECISION LINKS ──────────────────────────────────────────────────────────

decided_by:
  - id: KA-ADR-012
    decision: "Selected async executor over sync"
  - id: KA-ADR-017
    decision: "Exponential backoff retry strategy"

# ─── PRODUCT LINKS ───────────────────────────────────────────────────────────

consumed_by:
  - product: content-factory
    integration: KA-API-001
    criticality: required
  - product: claude-cli
    integration: KA-API-001
    criticality: required

# ─── REVERSE RELATIONSHIPS (declared for traceability, auto-computed by index) ─
# Do not declare reverse relationships manually — they are inferred and indexed.
# Declaring them here would create maintenance burden and divergence risk.
```

---

## 5. Entity-Specific Relationship Rules

### 5.1 Document → Document

Required declarations:
- `implements` — if this document follows a standard
- `depends_on` — if understanding this document requires first understanding another
- `supersedes` — if this document replaces an older document

Forbidden:
- Declaring `consumed_by` on a document (only capabilities use this)
- Circular `implements` chains

### 5.2 Capability → Capability

Each capability's `index.yaml` declares:

```yaml
relationships:
  depends_on:
    - capability: provider-framework
      reason: "Workflow steps delegate to providers"
      type: runtime_dependency

  consumed_by:
    - capability: content-factory
      relationship: product_consumer

  shares_contract_with:
    - capability: prompt-os
      contract: execution-interface
      contract_doc: KA-API-003
```

### 5.3 Capability → API

An API document is always owned by exactly one capability.

```yaml
# In api.md frontmatter:
owned_by: workflow-runtime   # The one capability that owns this API
```

Other capabilities that use this API declare:

```yaml
# In their architecture.md or index.yaml:
integrates:
  - api: KA-API-001
    role: consumer
    version: "v1"
```

### 5.4 Capability → Database

A database is always owned by exactly one capability (the one that creates and migrates it).

```yaml
# In database.md frontmatter:
owned_by: workflow-runtime
```

Other capabilities that read this database (not write) declare:

```yaml
reads_from:
  - db: KA-DB-001
    tables: [workflow_runs]
    access_type: read_only
```

### 5.5 Capability → Events

Events have producers and consumers. Both must be declared.

```yaml
# In the producing capability's events.md:
events_produced:
  - id: KA-EVT-001
    name: workflow.execution.started
    schema: workflow-execution-event-v1
    consumers_known:
      - context-intelligence
      - conversation-intelligence

# In the consuming capability's events.md:
events_consumed:
  - id: KA-EVT-001
    from_capability: workflow-runtime
    handler: workflow_started_handler
```

### 5.6 ADR Relationships

Every ADR must declare:

```yaml
# What it was deciding between:
options_considered:
  - name: "Sync executor"
    id: OPT-001
  - name: "Async executor"
    id: OPT-002

# What it decided:
decision: OPT-002

# What it affects:
affects:
  - id: KA-ARCH-001
    impact: "Changes execution model in section 3.2"
  - id: KA-IMPL-001
    impact: "Changes implementation pattern"

# What it supersedes:
supersedes: KA-ADR-008   # Prior decision, if any
```

### 5.7 Provider Relationships

```yaml
# In provider-framework/architecture.md:
wraps_providers:
  - id: PROV-anthropic
    name: Anthropic
    models: ["claude-sonnet-4-6", "claude-opus-4-8", "claude-haiku-4-5"]
    contract: KA-API-005
  - id: PROV-openai
    name: OpenAI
    models: ["gpt-4o", "gpt-4o-mini"]
    contract: KA-API-006
```

### 5.8 Agent Relationships

```yaml
# In an agent-related document:
agent:
  id: AGENT-execution-planner
  type: orchestrator
  uses_runtime: RUNTIME-ai-os
  uses_providers:
    - PROV-anthropic
  uses_prompts:
    - PROMPT-execution-planner
  produces_workflows:
    - WF-001
```

---

## 6. Relationship Graph Invariants

These invariants are enforced by the audit tool:

| Invariant | Rule |
|-----------|------|
| No dangling references | Every `id` in a relationship must exist in the registry |
| No self-references | A document cannot depend on itself |
| No circular dependencies | Cycles in `depends_on` chains are violations |
| Single ownership | Every capability, API, DB, and event has exactly one owner |
| Producer-consumer parity | Every `events_produced` must match a `events_consumed` declaration |
| ADR always affects something | Every accepted ADR must declare at least one `affects` entry |
| Implementation must verify | `implemented_by` with `confidence: high` requires `verified: true` |

---

## 7. Reverse Relationship Inference

Reverse relationships are never declared manually. They are computed by the index generator from forward declarations.

| Forward (declared) | Reverse (inferred) |
|-------------------|-------------------|
| A `depends_on` B | B `depended_on_by` A |
| A `implements` B | B `implemented_by` A |
| A `supersedes` B | B `superseded_by` A |
| A `consumed_by` P | P `consumes` A |
| A `produces_event` E | E `produced_by` A |
| A `stores_in` D | D `used_by` A |
| A `decided_by` ADR | ADR `motivated` A |

The reverse relationship index lives in `architecture/relationship-index.yaml` (see [KNOWLEDGE-INDEX-DESIGN.md](KNOWLEDGE-INDEX-DESIGN.md)).

---

## 8. Relationship Visualization

The relationship model is designed to support graph visualization in the Documentation Intelligence Platform (Phase 2.0D.2).

Node types map to visual representations:

| Node Type | Shape | Color |
|-----------|-------|-------|
| Document (architecture) | Rectangle | Blue |
| Document (overview) | Rounded rect | Light blue |
| Capability | Hexagon | Green |
| API | Diamond | Orange |
| Database | Cylinder | Purple |
| Event | Circle | Yellow |
| ADR | Scroll | Gray |
| Product | Star | Red |
| Provider | Cloud | Teal |
| Agent | Person | Pink |

Edge types map to visual line styles:

| Relationship | Line Style |
|-------------|-----------|
| `implements` | Solid, thin |
| `depends_on` | Solid, thick, arrow |
| `supersedes` | Dashed, arrow |
| `implemented_by` | Dotted, arrow |
| `consumed_by` | Solid, double-headed |
| `produces_event` | Wavy, arrow |
| `decided_by` | Dashed, no arrow |

---

## References

- [METADATA-STANDARD.md](METADATA-STANDARD.md) — Relationship declaration syntax
- [KNOWLEDGE-INDEX-DESIGN.md](KNOWLEDGE-INDEX-DESIGN.md) — Relationship Index structure
- [CANONICAL-SOURCE-STRATEGY.md](CANONICAL-SOURCE-STRATEGY.md) — Canonical references
- [KNOWLEDGE-NAVIGATION.md](KNOWLEDGE-NAVIGATION.md) — How to navigate the graph
- [KNOWLEDGE-HEALTH-METRICS.md](KNOWLEDGE-HEALTH-METRICS.md) — Relationship completeness scoring
