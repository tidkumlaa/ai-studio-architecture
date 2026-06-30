---
knowledge_id: KA-SPEC-002
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
  - KA-SPEC-001
---

# Knowledge Type System

## A Formal Type Hierarchy for All Knowledge Objects

---

## 1. Purpose

Every KnowledgeObject has a type. The type determines:
- Which metadata fields are required
- Which relationships are valid
- What index the object belongs to
- What health dimensions apply
- What the compiler can output from it

The type system is a closed enumeration at the top level, with extension points at the leaf level.

---

## 2. Type Hierarchy

```
KnowledgeObject (abstract root)
│
├── DocumentObject (abstract)
│   ├── VisionDocument          (type: vision)
│   ├── OverviewDocument        (type: overview)
│   ├── ArchitectureDocument    (type: architecture)
│   ├── APIDocument             (type: api)
│   ├── DatabaseDocument        (type: database)
│   ├── EventDocument           (type: event)
│   ├── SecurityDocument        (type: security)
│   ├── DesktopDocument         (type: desktop)
│   ├── ImplementationDocument  (type: implementation)
│   ├── TestingDocument         (type: testing)
│   ├── RoadmapDocument         (type: roadmap)
│   ├── HistoryDocument         (type: history)
│   ├── ReferenceDocument       (type: references)
│   ├── StandardDocument        (type: standard)
│   ├── SpecificationDocument   (type: specification)
│   └── ADRDocument             (type: adr)
│
├── StructuralObject (abstract)
│   ├── CapabilityObject        (type: capability)
│   ├── DomainObject            (type: domain)
│   └── ProductObject           (type: product)
│
├── ArtifactObject (abstract)
│   ├── ModuleObject            (type: module)
│   ├── APIEndpointObject       (type: api_endpoint)
│   ├── DatabaseSchemaObject    (type: db_schema)
│   ├── EventSchemaObject       (type: event_schema)
│   └── PromptObject            (type: prompt)
│
├── IntelligenceObject (abstract)
│   ├── AgentObject             (type: agent)
│   ├── WorkflowObject          (type: workflow)
│   └── ProviderObject          (type: provider)
│
└── MetaObject (abstract)
    ├── IndexObject             (type: index)
    ├── RegistryObject          (type: registry)
    └── DecisionObject          (type: decision)
```

---

## 3. Type Definitions

### 3.1 Document Types

| Type | Filename Convention | Required Fields | Health Dimensions |
|------|-------------------|-----------------|------------------|
| `vision` | `*VISION*.md` | All core fields | freshness, completeness |
| `overview` | `overview.md` | All core + capability | freshness, completeness |
| `architecture` | `architecture.md` | All core + evidence | all 6 dimensions |
| `api` | `api.md` | All core + related_apis | freshness, evidence |
| `database` | `database.md` | All core + related_database | freshness, evidence |
| `event` | `events.md` | All core + related_events | freshness, evidence |
| `security` | `security.md` | All core + confidence | freshness, completeness |
| `desktop` | `desktop.md` | All core + related_desktop | freshness |
| `implementation` | `implementation.md` | All core + evidence | freshness, evidence |
| `testing` | `testing.md` | All core + coverage.testing | freshness, coverage |
| `roadmap` | `roadmap.md` | All core | freshness |
| `history` | `history.md` | All core | none (append-only, always healthy) |
| `references` | `references.md` | All core | consistency (link validity) |
| `standard` | `*STANDARD*.md` | All core + implements | all 6 dimensions |
| `specification` | `*.md` in knowledge-core/ | All core | all 6 dimensions |
| `adr` | `KA-ADR-NNN-*.md` | ADR-specific schema | none (immutable) |

### 3.2 Structural Types

Structural types represent organizational nodes, not documents.

| Type | Represents | Indexed In |
|------|-----------|-----------|
| `capability` | A capability folder | CAPABILITY-INDEX |
| `domain` | An architectural domain | DOMAIN-INDEX |
| `product` | A product | PRODUCT-INDEX |

Structural types are declared in `index.yaml` files, not in standalone documents.

### 3.3 Artifact Types

Artifact types are references to implementation artifacts. They are referenced from DocumentObjects, not stored as documents themselves.

| Type | What It References |
|------|--------------------|
| `module` | A code module or package |
| `api_endpoint` | A single API endpoint |
| `db_schema` | A database schema file |
| `event_schema` | An event schema definition |
| `prompt` | A prompt template file |

### 3.4 Intelligence Types

Intelligence types represent AI-era entities that participate in the knowledge graph.

| Type | Description |
|------|-------------|
| `agent` | An AI agent with defined capabilities |
| `workflow` | An executable workflow definition |
| `provider` | An AI provider (Anthropic, OpenAI, etc.) |

### 3.5 Meta Types

Meta types represent the knowledge system's own artifacts.

| Type | Description |
|------|-------------|
| `index` | An index file (index.yaml) |
| `registry` | A registry file (KNOWLEDGE-REGISTRY.yaml) |
| `decision` | A decision not captured as a formal ADR |

---

## 4. Type Rules

### 4.1 Type Immutability

Once assigned, a KnowledgeObject's type never changes. If a document changes its fundamental nature (e.g., was `overview` and must become `architecture`), a new KnowledgeObject is created and the old one is superseded.

**Rationale:** Type changes break index entries, relationship declarations, and health scoring baselines. They are destructive changes that must be explicit.

### 4.2 Type-Capability Cardinality

For DocumentTypes within a capability:

| Cardinality | Types |
|-------------|-------|
| **0 or 1** (optional) | api, database, event, security, desktop |
| **Exactly 1** (required) | overview, architecture, implementation, testing, roadmap, history, references |
| **0..N** (multiple allowed) | adr, specification |
| **1 per repository** | vision, standard (per topic) |

A capability may not have two `architecture` documents both with `canonical: true`. This would violate I5 (Single Canonical Invariant).

### 4.3 Type-Index Membership

Each type belongs to exactly one primary index:

| Type | Primary Index |
|------|--------------|
| architecture, overview, api, database, event, security, desktop, implementation, testing, roadmap | CAPABILITY-INDEX |
| adr | ADR-INDEX |
| standard, specification, vision | STANDARD-INDEX |
| history, references | CAPABILITY-INDEX (non-health-scored) |
| capability | CAPABILITY-INDEX (structural entry) |
| domain | DOMAIN-INDEX |
| product | PRODUCT-INDEX |
| index, registry | META-INDEX |

### 4.4 Type Compatibility in Relationships

Not all relationship types are valid between all KnowledgeObject types.

| Relationship | From Types | To Types |
|-------------|-----------|---------|
| `implements` | Any DocumentObject | standard, specification |
| `depends_on` | Any DocumentObject | Any DocumentObject |
| `supersedes` | Any DocumentObject | Same type only |
| `implemented_by` | architecture, specification | module, api_endpoint, db_schema |
| `tested_by` | architecture, specification, implementation | module (test type) |
| `produces_event` | capability | event_schema |
| `consumes_event` | capability | event_schema |
| `decided_by` | architecture, specification | adr |
| `stores_in` | capability, architecture | db_schema |
| `owned_by` | api, database, event | capability |

Relationship type violations are audit errors (RULE-R005 — to be added in Phase 2.0D.1).

---

## 5. Type Extension Protocol

The type system is closed at the abstract level but extensible at the leaf level. New leaf types may be added through the following protocol:

1. Propose the new type in an ADR
2. Define the required fields in the ADR
3. Define the health dimensions in the ADR
4. Define the type's index membership in the ADR
5. ADR is approved by Chief Knowledge Architect
6. Type is added to this specification (MINOR version bump)
7. Schema updated (see [KA-SPEC-005](05-KNOWLEDGE-SCHEMA.md))
8. Audit rules updated for the new type

New abstract parent types require a MAJOR version bump and full review.

---

## 6. Type Inference

When a document does not declare its `type` in frontmatter, the index engine attempts to infer it from the filename:

| Filename | Inferred Type |
|----------|--------------|
| `overview.md` | `overview` |
| `architecture.md` | `architecture` |
| `api.md` | `api` |
| `database.md` | `database` |
| `events.md` | `event` |
| `security.md` | `security` |
| `desktop.md` | `desktop` |
| `implementation.md` | `implementation` |
| `testing.md` | `testing` |
| `roadmap.md` | `roadmap` |
| `history.md` | `history` |
| `references.md` | `references` |
| `KA-ADR-*.md` | `adr` |
| `*VISION*.md` | `vision` |
| `*STANDARD*.md` | `standard` |
| Other | `unknown` — flagged as RULE-M003 violation |

Type inference is a fallback only. Explicit declaration is required for approved documents.

---

## References

- [01-KNOWLEDGE-OBJECT-SPECIFICATION.md](01-KNOWLEDGE-OBJECT-SPECIFICATION.md) — KnowledgeObject that types apply to
- [05-KNOWLEDGE-SCHEMA.md](05-KNOWLEDGE-SCHEMA.md) — JSON Schema per type
- [14-KNOWLEDGE-LIFECYCLE.md](14-KNOWLEDGE-LIFECYCLE.md) — Lifecycle rules vary by type
- [KA-STD-001](../knowledge-architecture/DOCUMENT-FOLDER-STANDARDS.md) — Document folder standards (type → filename mapping)
