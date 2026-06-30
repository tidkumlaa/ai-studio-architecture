# KNW-KE-ARCH-028 — Knowledge Metadata

**Phase:** 3.0C.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Defines all metadata fields on Knowledge Objects — their semantics, constraints, and usage. Metadata is the envelope around the object's domain content.

---

## Metadata Categories

Knowledge Object metadata is divided into five categories:

| Category | Fields | Set by |
|----------|--------|--------|
| Identity | `knowledge_id`, `canonical_name`, `object_uri` | `kos` / identity engine |
| Classification | `object_type`, `domain`, `namespace`, `tags`, `labels` | Author |
| Lifecycle | `lifecycle.state`, `lifecycle.version`, `lifecycle.created_at`, … | `kos lifecycle` |
| Ownership | `ownership.owner`, `ownership.team`, `ownership.maintainers`, `ownership.stakeholders` | Author |
| Operational | `quality_score`, `created_at`, `updated_at`, `checksum` | System |

---

## Identity Metadata

```yaml
knowledge_id: KNW-PLT-MOD-001       # canonical ID — immutable after assignment
canonical_name: plt.module.quota-manager   # {namespace}.{type_lower}.{slug}
object_uri: knw://plt/module/quota-manager@1.0.0
```

Rules:
- `knowledge_id` is assigned once by the identity engine; never changed, never reused
- `canonical_name` changes only on MAJOR rename (with alias preserved ≥ 90 days)
- `object_uri` version component mirrors `lifecycle.version`

---

## Classification Metadata

```yaml
object_type: module
domain: PLATFORM
namespace: plt
tags:
  - quota
  - resource-management
  - platform-core
labels:
  team: platform
  criticality: high
  feature-flag: quota-v2
```

Rules:
- `object_type` is immutable after assignment
- `domain` is derived from `knowledge_id` prefix; must match
- `namespace` is derived from `domain`; must match package namespace
- `tags` — lowercase, hyphen-delimited, from canonical tag registry (09-KNOWLEDGE-CATALOG)
- `labels` — free-form key:value pairs, not indexed

---

## Lifecycle Metadata

Full lifecycle metadata schema:

```yaml
lifecycle:
  state: CANONICAL            # DRAFT | PROPOSED | VERIFIED | CANONICAL | DEPRECATED | ARCHIVED
  version: "1.0.0"            # semver — bumped by kos lifecycle commands
  created_at: "2026-06-30"
  updated_at: "2026-06-30"
  deprecated_at: null
  sunset_at: null
  archived_at: null
  deprecation_reason: null
  successor_id: null          # KNW ID of replacement object, if deprecated
  lifecycle_events:           # append-only event log
    - event: PROMOTED_TO_CANONICAL
      at: "2026-06-30T00:00:00Z"
      by: "team:architecture-board"
      reason: "All gates passed"
```

---

## Ownership Metadata

```yaml
ownership:
  owner: team:platform           # primary responsible team or person
  team: team:platform
  maintainers:
    - user:jane.doe
    - user:john.smith
  stakeholders:
    - team:runtime
    - team:ai-products
  review_required_from:          # approvers required for MAJOR changes
    - team:architecture-board
```

Rules:
- `owner` is required for all non-DRAFT objects
- `owner` must be a valid team or user identifier from the identity registry
- `maintainers` is the set of people who can merge MINOR changes
- `review_required_from` is set automatically for CANONICAL objects

---

## Operational Metadata

Operational metadata is written by the system, not the author:

```yaml
quality_score: 0.87        # computed by scoring engine; read-only
checksum: "sha256:abc..."  # SHA-256 of canonical content (excludes this field)
created_at: "2026-06-30T00:00:00Z"   # first registered
updated_at: "2026-06-30T00:00:00Z"   # last modified
```

Rules:
- Authors must not set `quality_score`, `checksum`, `created_at`, `updated_at`
- `kos format` strips these fields from author-written YAML
- These fields are injected by the registry on import

---

## Metadata Validation Rules

| Rule | Description |
|------|-------------|
| MD-001 | `knowledge_id` must match regex `^KNW-[A-Z][A-Z0-9]*-[A-Z][A-Z0-9]*-[A-Za-z0-9_-]+$` |
| MD-002 | `canonical_name` must match `^[a-z][a-z0-9]*\.[a-z][a-z0-9]*\.[a-z][a-z0-9-]*$` |
| MD-003 | `object_type` must be one of the 33 canonical types |
| MD-004 | `namespace` must match the package's declared namespace |
| MD-005 | All tags must appear in the canonical tag registry |
| MD-006 | `owner` must be non-empty for VERIFIED+ objects |
| MD-007 | `lifecycle.state` must follow valid transitions (see 19-KNOWLEDGE-LIFECYCLE) |
| MD-008 | `lifecycle.version` must be semver format |

---

## Metadata Search Index

The following metadata fields are indexed for fast search:

```yaml
indexed_fields:
  - knowledge_id         # exact match
  - canonical_name       # exact + prefix
  - object_type          # exact
  - namespace            # exact
  - lifecycle.state      # exact
  - ownership.owner      # exact
  - ownership.team       # exact
  - tags                 # multi-value exact
  - quality_score        # range
```

Non-indexed fields (full-scan only): `labels`, `description`, `lifecycle_events`.

---

## Metadata Commands

```bash
kos meta show KNW-PLT-MOD-001              # show all metadata
kos meta set KNW-PLT-MOD-001 --tag quota   # add a tag
kos meta set KNW-PLT-MOD-001 --owner team:runtime
kos meta validate KNW-PLT-MOD-001          # validate all metadata rules
kos meta audit --field owner               # find objects missing owner
```

---

## Cross-References

- Identity assignment → Phase 3.0C `01-IDENTITY-ENGINE`
- Lifecycle states → `19-KNOWLEDGE-LIFECYCLE`
- Tag registry → `09-KNOWLEDGE-CATALOG`
- Quality score computation → `17-KNOWLEDGE-SCORING`
- Linter metadata rules → `06-KNOWLEDGE-LINTER` (KL-001–KL-005)
