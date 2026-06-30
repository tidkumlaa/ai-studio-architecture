---
knowledge_id: KA-SPEC-004
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
depends_on:
  - id: KA-SPEC-003
    reason: "URIs are a representation of Knowledge IDs"
---

# Knowledge URI Specification

## Universal Resource Identifiers for Knowledge Objects

---

## 1. Purpose

A Knowledge ID (`KA-ARCH-001`) is the permanent identity of a KnowledgeObject. A Knowledge URI is the **addressable** form of that identity — a string that carries enough information to locate, retrieve, and dereference any KnowledgeObject in the system.

Knowledge URIs enable:
- Deep linking into the knowledge graph from external tools
- Unambiguous cross-tool references
- REST API addressing in the Documentation Intelligence Platform
- Command-line tool arguments
- Browser-navigable knowledge graph URLs

---

## 2. URI Scheme

### 2.1 Formal Definition

```
KnowledgeURI ::= "ka://" authority "/" capability "/" type "/" id [ "#" fragment ]

authority    ::= domain-id                         (e.g., "dom-workflow")
capability   ::= capability-name                   (e.g., "workflow-runtime")
type         ::= type-code-lowercase               (e.g., "architecture")
id           ::= knowledge-id                      (e.g., "KA-ARCH-001")
fragment     ::= section-slug                      (e.g., "design-goals")

Full example:
  ka://dom-workflow/workflow-runtime/architecture/KA-ARCH-001#design-goals
```

### 2.2 URI Components

| Component | Description | Example |
|-----------|-------------|---------|
| Scheme | Always `ka` | `ka://` |
| Authority | Domain ID, lowercase | `dom-workflow` |
| Capability | Capability folder name | `workflow-runtime` |
| Type | Document type, lowercase | `architecture` |
| ID | Knowledge ID | `KA-ARCH-001` |
| Fragment | Section anchor (optional) | `#design-goals` |

---

## 3. URI Variants

### 3.1 Full URI (Canonical)

References a specific KnowledgeObject at a specific version:

```
ka://dom-workflow/workflow-runtime/architecture/KA-ARCH-001
```

### 3.2 Versioned URI

References a specific version of a KnowledgeObject:

```
ka://dom-workflow/workflow-runtime/architecture/KA-ARCH-001@1.2.0
```

The `@version` suffix pins to a specific version. Without it, the URI resolves to the current (latest approved) version.

### 3.3 Section URI (Fragment)

References a section within a KnowledgeObject:

```
ka://dom-workflow/workflow-runtime/architecture/KA-ARCH-001#component-model
```

Fragment identifiers are derived from heading text: lowercase, spaces replaced with hyphens, non-alphanumeric removed.

```
## Component Model  →  #component-model
### Data Flow       →  #data-flow
```

### 3.4 Capability URI

References a capability as a whole (not a specific document):

```
ka://dom-workflow/workflow-runtime
```

Resolves to the capability's `index.yaml` entry.

### 3.5 Domain URI

References a domain:

```
ka://dom-workflow
```

Resolves to the domain's entry in `DOMAIN-INDEX.yaml`.

### 3.6 Type URI (Collection)

References all KnowledgeObjects of a type within a capability:

```
ka://dom-workflow/workflow-runtime/architecture/*
```

Resolves to a collection. Used in query contexts.

### 3.7 Abbreviated URI

For internal use and human-readable contexts, the authority and capability may be inferred from context:

```
/workflow-runtime/architecture/KA-ARCH-001    (domain inferred from context)
architecture/KA-ARCH-001                       (capability and domain inferred)
KA-ARCH-001                                    (full context inferred — registry lookup)
```

The shortest form `KA-ARCH-001` is the Knowledge ID itself, and always resolves via the registry without needing URI components.

---

## 4. URI Resolution Algorithm

### 4.1 Full URI Resolution

```
FUNCTION resolve(uri: KnowledgeURI) → KnowledgeObject:
  
  1. Parse uri into (scheme, authority, capability, type, id, fragment)
  2. Validate scheme == "ka"
  3. Validate authority in DOMAIN-INDEX
  4. Validate capability in CAPABILITY-INDEX[authority]
  5. Validate type is a known KnowledgeType
  6. Look up id in KNOWLEDGE-REGISTRY
     IF not found: RAISE KnowledgeNotFoundError(id)
  7. Retrieve KnowledgeObject from store by id
  8. IF fragment provided:
       Resolve to specific section within object
  9. RETURN KnowledgeObject (or section)
```

### 4.2 ID-Only Resolution

```
FUNCTION resolve_id(id: KnowledgeID) → KnowledgeObject:
  
  1. Look up id in KNOWLEDGE-REGISTRY
     IF not found: RAISE KnowledgeNotFoundError(id)
  2. IF status == "retired":
       Retrieve archived object
       SET resolution_note = "This object is archived"
  3. IF status == "deprecated":
       Retrieve deprecated object
       SET resolution_note = "Superseded by: " + superseded_by
  4. RETURN KnowledgeObject
```

### 4.3 Resolution Fallback Chain

When a URI resolves to a deprecated or superseded object, the resolver returns the object but annotates the response:

```json
{
  "resolved": true,
  "object": { ... },
  "status": "deprecated",
  "superseded_by": "ka://dom-workflow/workflow-runtime/architecture/KA-ARCH-002",
  "message": "This object was deprecated on 2026-01-15. Consider using the successor."
}
```

---

## 5. URI in Markdown

### 5.1 Markdown Link Form

In Markdown documents, Knowledge URIs appear as standard links:

```markdown
See the [Workflow Runtime Architecture](ka://dom-workflow/workflow-runtime/architecture/KA-ARCH-001)
for the canonical design.
```

### 5.2 Relative File Link Form

For filesystem-based rendering (before the Documentation Intelligence Platform exists), Knowledge URIs degrade gracefully to relative filesystem paths:

```markdown
See [Workflow Runtime Architecture](../workflow-runtime/architecture.md)
```

### 5.3 ID Citation Form

For compact references in text:

```markdown
The executor design (KA-ARCH-001) defines the core execution model.
```

Renderers in Phase 2.0D.2+ resolve bare Knowledge IDs to hyperlinks automatically.

---

## 6. URI in YAML Frontmatter

```yaml
# Full URI form (for cross-tool use)
depends_on:
  - uri: "ka://dom-workflow/workflow-runtime/architecture/KA-ARCH-001"
    reason: "Depends on workflow execution model"

# ID form (equivalent — resolved via registry)
depends_on:
  - id: KA-ARCH-001
    reason: "Depends on workflow execution model"
```

Both forms are equivalent. The registry resolves ID form to URI form at index time.

---

## 7. URI Stability Guarantee

The Knowledge URI is as stable as the Knowledge ID it contains.

Stability breakdown:
- `ka://` — permanent (scheme never changes)
- `dom-workflow` — permanent (domain IDs never change)
- `workflow-runtime` — stable (capability names change only via ADR)
- `architecture` — permanent (type never changes per I4)
- `KA-ARCH-001` — permanent (IDs never change per I2)

The URI therefore has the same stability guarantee as the Knowledge ID itself: permanent.

If a capability is renamed, the old capability name is registered as an alias. All existing URIs continue to resolve.

---

## 8. REST API URI Mapping

In the Documentation Intelligence Platform (Phase 2.0D.2), Knowledge URIs map to REST endpoints:

```
Knowledge URI:                 REST API Path:
ka://dom-workflow              GET /api/v1/domains/dom-workflow
ka://dom-workflow/workflow-runtime            GET /api/v1/capabilities/workflow-runtime
ka://dom-workflow/workflow-runtime/architecture/KA-ARCH-001   GET /api/v1/objects/KA-ARCH-001
ka://dom-workflow/workflow-runtime/architecture/KA-ARCH-001@1.0.0  GET /api/v1/objects/KA-ARCH-001?version=1.0.0
```

The REST API is designed in Phase 2.0D.2. This section defines the mapping contract.

---

## References

- [03-KNOWLEDGE-IDENTITY-SPECIFICATION.md](03-KNOWLEDGE-IDENTITY-SPECIFICATION.md) — Knowledge IDs that form the core of URIs
- [01-KNOWLEDGE-OBJECT-SPECIFICATION.md](01-KNOWLEDGE-OBJECT-SPECIFICATION.md) — What URIs resolve to
- [10-KNOWLEDGE-INDEX-ENGINE.md](10-KNOWLEDGE-INDEX-ENGINE.md) — URI resolution implementation
- [07-KNOWLEDGE-QUERY-LANGUAGE.md](07-KNOWLEDGE-QUERY-LANGUAGE.md) — KQL uses URIs as operands
