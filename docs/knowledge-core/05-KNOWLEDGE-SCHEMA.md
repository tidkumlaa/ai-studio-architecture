---
knowledge_id: KA-SPEC-005
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
  - KA-SPEC-002
depends_on:
  - id: KA-SPEC-003
    reason: "Schema includes identity fields"
---

# Knowledge Schema

## The Formal Schema for All Knowledge Objects

---

## 1. Purpose

The Knowledge Schema is the machine-enforceable contract for every KnowledgeObject in the system. It defines required fields, types, formats, constraints, and validation rules in a form that can be executed by tooling without human judgment.

This schema is implemented in JSON Schema Draft 2020-12 for runtime validation, and in YAML frontmatter format for authoring.

---

## 2. Base Schema

The base schema applies to ALL KnowledgeObject types:

```yaml
# knowledge-object-base.schema.yaml
$schema: "https://json-schema.org/draft/2020-12/schema"
$id: "ka://dom-knowledge/knowledge-core/schema/base"
title: "KnowledgeObject Base Schema"

type: object
required:
  - knowledge_id
  - version
  - status
  - canonical
  - domain
  - capability
  - type
  - owner
  - created
  - phase
  - confidence

properties:

  # ── IDENTITY ──────────────────────────────────────────────────────────────
  knowledge_id:
    type: string
    pattern: "^KA-[A-Z]{2,5}-[0-9]{3,}$"
    description: "Unique Knowledge ID. Format: KA-[TYPE]-[NNN]"

  version:
    type: string
    pattern: "^[0-9]+\\.[0-9]+\\.[0-9]+$"
    description: "Semantic version MAJOR.MINOR.PATCH"

  status:
    type: string
    enum: [draft, review, approved, deprecated, archived]

  canonical:
    type: boolean
    description: "True if this is the single authoritative source for this concept"

  # ── CLASSIFICATION ────────────────────────────────────────────────────────
  domain:
    type: string
    pattern: "^DOM-[A-Z]+$"
    description: "Domain ID from DOMAIN-INDEX"

  capability:
    type: string
    pattern: "^[a-z][a-z0-9-]*[a-z0-9]$"
    description: "Capability folder name (kebab-case)"

  type:
    type: string
    enum:
      - vision
      - overview
      - architecture
      - api
      - database
      - event
      - security
      - desktop
      - implementation
      - testing
      - roadmap
      - history
      - references
      - standard
      - specification
      - adr
      - index
      - capability
      - domain
      - product

  # ── OWNERSHIP ─────────────────────────────────────────────────────────────
  owner:
    type: string
    minLength: 3
    description: "Role-level owner (not person name)"

  created:
    type: string
    format: date
    description: "ISO 8601 creation date YYYY-MM-DD"

  review_date:
    type: string
    format: date
    description: "ISO 8601 next review date"

  # ── QUALITY ───────────────────────────────────────────────────────────────
  phase:
    type: string
    pattern: "^[0-9]+\\.[0-9]+[A-Z]?(\\.[0-9]+)?(\\.[0-9]+)?$"
    description: "Phase ID (e.g., 2.0D.0.5)"

  confidence:
    type: string
    enum: [high, medium, low]

  # ── LIFECYCLE (optional fields) ────────────────────────────────────────────
  deprecated_date:
    type: string
    format: date

  superseded_by:
    type: string
    pattern: "^KA-[A-Z]{2,5}-[0-9]{3,}$"

  archived_date:
    type: string
    format: date

  # ── RELATIONSHIPS (optional) ───────────────────────────────────────────────
  implements:
    type: array
    items:
      type: string
      pattern: "^KA-[A-Z]{2,5}-[0-9]{3,}$"

  depends_on:
    type: array
    items:
      oneOf:
        - type: string
          pattern: "^KA-[A-Z]{2,5}-[0-9]{3,}$"
        - type: object
          required: [id]
          properties:
            id:
              type: string
              pattern: "^KA-[A-Z]{2,5}-[0-9]{3,}$"
            reason:
              type: string
            criticality:
              type: string
              enum: [required, optional, enhances]

  implemented_by:
    type: array
    items:
      type: object
      required: [type, ref]
      properties:
        type:
          type: string
          enum: [module, service, test, script, config, schema]
        ref:
          type: string
        confidence:
          type: string
          enum: [high, medium, low]
        verified:
          type: boolean
        last_verified:
          type: string
          format: date

  related_apis:
    type: array
    items:
      $ref: "#/definitions/RelatedRef"

  related_database:
    type: array
    items:
      $ref: "#/definitions/RelatedRef"

  related_events:
    type: array
    items:
      $ref: "#/definitions/RelatedRef"

  related_adr:
    type: array
    items:
      $ref: "#/definitions/RelatedRef"

  related_products:
    type: array
    items:
      type: object
      required: [product]
      properties:
        product:
          type: string
        role:
          type: string
          enum: [consumer, producer, owner]

  # ── COVERAGE (optional) ────────────────────────────────────────────────────
  coverage:
    type: object
    properties:
      architecture:
        type: integer
        minimum: 0
        maximum: 100
      implementation:
        type: integer
        minimum: 0
        maximum: 100
      testing:
        type: integer
        minimum: 0
        maximum: 100
      desktop:
        type: integer
        minimum: 0
        maximum: 100

  # ── EVIDENCE (optional) ────────────────────────────────────────────────────
  evidence:
    type: array
    items:
      type: object
      required: [type, ref]
      properties:
        type:
          type: string
          enum: [source_file, test, schema, config, log, adr]
        ref:
          type: string
        verified:
          type: boolean
        last_verified:
          type: string
          format: date

  # ── TAGS (optional) ────────────────────────────────────────────────────────
  tags:
    type: array
    items:
      type: string

definitions:
  RelatedRef:
    type: object
    required: [id]
    properties:
      id:
        type: string
        pattern: "^KA-[A-Z]{2,5}-[0-9]{3,}$"
      title:
        type: string
      reason:
        type: string
```

---

## 3. Type-Specific Schema Extensions

Each KnowledgeObject type extends the base schema with type-specific constraints.

### 3.1 Architecture Document Schema Extension

```yaml
$id: "ka://dom-knowledge/knowledge-core/schema/architecture"
allOf:
  - $ref: "#base"
if:
  properties:
    status: { const: approved }
then:
  required:
    - review_date
    - evidence          # Architecture docs require evidence when approved
    - implemented_by    # Required when confidence is high
```

### 3.2 ADR Document Schema Extension

```yaml
$id: "ka://dom-knowledge/knowledge-core/schema/adr"
allOf:
  - $ref: "#base"

properties:
  decision_date:
    type: string
    format: date
    description: "Date the decision was made"

  participants:
    type: array
    items:
      type: string

  options_considered:
    type: array
    items:
      type: object
      required: [name]
      properties:
        name:
          type: string
        pros:
          type: array
          items: { type: string }
        cons:
          type: array
          items: { type: string }

  decision:
    type: string
    description: "The name of the option that was chosen"

  affects:
    type: array
    items:
      type: object
      required: [id]
      properties:
        id:
          type: string
          pattern: "^KA-[A-Z]{2,5}-[0-9]{3,}$"
        impact:
          type: string

  supersedes:
    type: string
    pattern: "^KA-ADR-[0-9]{3,}$"

required_if_status_accepted:
  - decision_date
  - decision

# ADR-specific overrides:
not_required:
  - review_date   # ADRs have no review date — immutable once accepted
  - coverage      # ADRs have no coverage
  - confidence    # ADRs are facts, not estimates
```

### 3.3 History Document Schema Extension

```yaml
$id: "ka://dom-knowledge/knowledge-core/schema/history"
allOf:
  - $ref: "#base"

# History documents are append-only and never stale
not_required:
  - review_date   # Exempt from review cycle
  - evidence      # Not implementation docs
  - coverage      # Not applicable
```

---

## 4. Schema Validation Modes

| Mode | Description | When Used |
|------|-------------|----------|
| **Strict** | All required fields, all type constraints | CI/CD gate |
| **Lenient** | Required fields only, no type constraints | Developer save |
| **Draft** | Only `knowledge_id` required | New document creation |
| **Audit** | Full validation + cross-reference checks | Audit tool |

### 4.1 Cross-Reference Validation (Audit Mode Only)

Cross-reference validation checks relationships against the live registry:

```
For each ID in [implements, depends_on, related_*, superseded_by]:
  IF NOT exists in KNOWLEDGE-REGISTRY.yaml:
    RAISE SchemaValidationError("Broken reference: " + id)

For each (capability, type) pair:
  IF count(canonical=true AND status=approved) > 1:
    RAISE SchemaValidationError("Duplicate canonical: " + capability + "/" + type)
```

---

## 5. Schema Evolution Rules

Schema changes follow a compatibility matrix:

| Change Type | Compatibility | Version Bump |
|-------------|--------------|-------------|
| Add optional field | Backward compatible | MINOR |
| Add required field | Breaking | MAJOR |
| Remove optional field | Forward breaking | MAJOR |
| Change enum values (add) | Backward compatible | MINOR |
| Change enum values (remove) | Breaking | MAJOR |
| Change pattern constraint (relax) | Backward compatible | MINOR |
| Change pattern constraint (restrict) | Breaking | MAJOR |

MAJOR schema version bumps require:
1. Migration tool to update all existing documents
2. ADR documenting the change rationale
3. 30-day deprecation notice before enforcement

---

## References

- [01-KNOWLEDGE-OBJECT-SPECIFICATION.md](01-KNOWLEDGE-OBJECT-SPECIFICATION.md) — What this schema validates
- [02-KNOWLEDGE-TYPE-SYSTEM.md](02-KNOWLEDGE-TYPE-SYSTEM.md) — Type enum values used in schema
- [03-KNOWLEDGE-IDENTITY-SPECIFICATION.md](03-KNOWLEDGE-IDENTITY-SPECIFICATION.md) — ID pattern defined here
- [KA-STD-002](../knowledge-architecture/METADATA-STANDARD.md) — Human-authored YAML this schema validates
