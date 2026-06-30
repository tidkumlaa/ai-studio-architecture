# KNW-KE-ARCH-007 — Knowledge Formatter

**Phase:** 3.0C.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

`kos format` produces stable, canonical output for all Knowledge Object files. Idempotent — running twice produces the same result. Fixes all auto-fixable linter rules.

---

## Command Interface

```bash
kos format [PATH]               # format file or directory
kos format --package {name}     # format entire package
kos format --check              # check only; exit 1 if formatting needed
kos format --diff               # show diff without writing
kos format --skip-style         # skip style-only transforms
```

---

## YAML Field Ordering

Every object file is rewritten with fields in this canonical order:

```yaml
# Section 1: Core identity
knowledge_id:
object_type:
name:
version:
status:
owner:
description:

# Section 2: Classification
classification:
  domain:
  layer:
  category:

# Section 3: Computed identity (auto-generated)
identity:
  global_id:
  canonical_name:
  knowledge_uri:

# Section 4: Tags
tags:

# Section 5: Quality sub-sections
evidence:
  items:
  evidence_score:

quality:
  overall_score:
  computed_at:

confidence:
  declared:
  composite:

# Section 6: Traceability
traceability:
  satisfies:
  implements:
  tests:

# Section 7: Lifecycle
lifecycle:
  review_status:
  approved_by:
  approved_at:

# Section 8: Relationships (computed; do not edit manually)
relationships:

# Section 9: Metadata
metadata:
  created_by:
  created_at:
  source_doc:

# Section 10: Type-specific fields (alphabetical within section)
{type_specific_fields in alphabetical order}
```

---

## YAML Normalization Rules

| Rule | Transform |
|------|-----------|
| FR-001 | 2-space indentation (tabs → spaces) |
| FR-002 | Empty list → `[]` (inline) |
| FR-003 | Single-item list → block style |
| FR-004 | Multi-item list → block style |
| FR-005 | Strings with `:`, `{`, `}`, `[`, `]`, `#` → double-quoted |
| FR-006 | Boolean values → bare `true` / `false` |
| FR-007 | Null values → bare `null` |
| FR-008 | Trailing whitespace removed from all lines |
| FR-009 | Single blank line at end of file |
| FR-010 | No duplicate keys (last value wins; formatter warns) |

---

## String Normalization

| Rule | Transform |
|------|-----------|
| FR-S-01 | `name`: strip leading/trailing whitespace |
| FR-S-02 | `name`: collapse internal multiple spaces → single space |
| FR-S-03 | `description`: strip leading/trailing whitespace |
| FR-S-04 | `tags`: sort alphabetically; deduplicate |
| FR-S-05 | `tags`: lowercase all entries |
| FR-S-06 | `owner`: lowercase after `:` (e.g., `team:Platform` → `team:platform`) |

---

## Auto-Generated Field Update

The formatter recomputes and writes:

| Field | Computed From |
|-------|--------------|
| `identity.canonical_name` | `{namespace}.{type_lower}.{slugify(name)}` |
| `identity.knowledge_uri` | `knw://{namespace}/{type}/{slug}@{version}` |
| `tags[0]` | Set to package namespace if not already |

The formatter does **not** write:
- `identity.global_id` — assigned by registry at registration time
- `quality.*` — computed by Quality Engine
- `evidence.evidence_score` — computed by Evidence Engine
- `confidence.composite` — computed by Confidence Engine

---

## Reference Normalization

```
FR-R-01: Relationship files — source_id and target_id printed in
         knowledge_id sort order.

FR-R-02: Relationship files — fields printed in canonical order:
         relation_id, relation_type, source_id, target_id,
         strength, confidence, evidence, metadata.

FR-R-03: traceability.satisfies — sorted by knowledge_id.

FR-R-04: traceability.implements — sorted by knowledge_id.

FR-R-05: traceability.tests — sorted by knowledge_id.
```

---

## Markdown Normalization

For `.md` files in `docs/` directories:

| Rule | Transform |
|------|-----------|
| FR-M-01 | ATX headings (`#` style) — convert setext (`===`) |
| FR-M-02 | Single blank line before and after headings |
| FR-M-03 | Trailing whitespace removed |
| FR-M-04 | Link reference style → inline links |
| FR-M-05 | Single blank line at end of file |

---

## Idempotency Contract

`kos format X && kos format X` must produce identical output to `kos format X`.

This is enforced in CI by `kos format --check`. Any formatter change that breaks idempotency is a regression.

---

## Formatter Configuration

```yaml
# formatter/formatter-config.yaml
version: "1.0.0"
yaml_indent: 2
max_line_length: 120
sort_tags: true
normalize_style: true
recompute_canonical_name: true
recompute_knowledge_uri: true
update_tags_namespace: true
```

---

## Integration

- Pre-commit hook runs `kos format --check` on staged YAML files
- CI PR check: `kos format --check` fails if any file needs formatting
- Author may run `kos format` locally to fix before push

---

## Cross-References

- Style rules the formatter applies → `04-KNOWLEDGE-STYLE-GUIDE`
- Linter auto-fix calls formatter → `06-KNOWLEDGE-LINTER`
- CI pipeline → `08-KNOWLEDGE-CI`
