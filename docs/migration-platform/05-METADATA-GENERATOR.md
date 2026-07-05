---
knowledge_id: KA-PLAT-006
version: "1.0.0"
status: approved
owner: Chief Knowledge Architect
phase: 2.0D.1
created: 2026-06-29
review_date: 2026-12-29
canonical: true
domain: DOM-KNOWLEDGE
capability: migration-platform
type: specification
depends_on:
  - id: KA-PLAT-004
    reason: "Consumes ClassificationResult for type and capability"
  - id: KA-SPEC-003
    reason: "ID generation protocol"
  - id: KA-SPEC-005
    reason: "Schema that generated frontmatter must satisfy"
  - id: KA-STD-002
    reason: "Metadata standard defining all required fields"
---

# Metadata Generator

## YAML Frontmatter Generation per Knowledge Architecture Standard

---

## 1. Purpose

The MetadataGenerator produces complete, schema-valid YAML frontmatter for every
document that lacks it, and repairs incomplete frontmatter for documents that have
partial metadata. It implements the ID generation protocol from KA-SPEC-003.

---

## 2. Metadata Fields Generated

All 12 required fields from KA-STD-002:

| Field | Source | Notes |
|-------|--------|-------|
| `knowledge_id` | Registry + ALG-001 | Format: `KA-{TYPE}-{NNN}` |
| `version` | Inferred from content | Default `"1.0.0"` |
| `status` | Inferred from content | Default `draft` |
| `canonical` | Canonical resolver | Default `true` for new docs |
| `domain` | Classifier result | From capability→domain map |
| `capability` | Classifier result | Directory-inferred |
| `type` | Classifier result | Mapped to KA-SPEC-002 types |
| `owner` | Config default | e.g. `Chief Knowledge Architect` |
| `created` | File mtime or today | ISO 8601 date |
| `review_date` | Created + 6 months | ISO 8601 date |
| `phase` | Config default | Current phase |
| `confidence` | Classifier confidence | `high` / `medium` / `low` |

---

## 3. Knowledge ID Assignment

ID assignment follows ALG-001 from KA-SPEC-017:

```
ASSIGN_ID(doc_type, registry):
  type_code = TYPE_CODE_MAP[doc_type]  # e.g. ARCH, SPEC, ADR
  
  IF existing_id IN registry: RETURN existing_id  # Skip already-assigned
  
  sequence = registry.next_available[type_code]
  candidate = f"KA-{type_code}-{sequence:03d}"
  
  WHILE candidate IN registry.all_ids:  # Collision check
    sequence += 1
    candidate = f"KA-{type_code}-{sequence:03d}"
    
  registry.next_available[type_code] = sequence + 1
  registry.register(candidate, doc_path)
  RETURN candidate
```

---

## 4. Type Code Mapping

| Document Type | Type Code | Example ID |
|---------------|-----------|-----------|
| `architecture` | `ARCH` | `KA-ARCH-001` |
| `specification` | `SPEC` | `KA-SPEC-021` |
| `decision` | `ADR` | `KA-ADR-001` |
| `api` | `API` | `KA-API-001` |
| `database` | `DB` | `KA-DB-001` |
| `workflow` | `WF` | `KA-WF-001` |
| `prompt` | `PROM` | `KA-PROM-001` |
| `security` | `SEC` | `KA-SEC-001` |
| `desktop` | `DESK` | `KA-DESK-001` |
| `provider` | `PROV` | `KA-PROV-001` |
| `runtime` | `RT` | `KA-RT-001` |
| `reference` | `REF` | `KA-REF-001` |
| `guide` | `GUIDE` | `KA-GUIDE-001` |
| `capability` | `CAP` | `KA-CAP-001` |
| `index` | `IDX` | `KA-IDX-001` |
| `history` | `HIST` | `KA-HIST-001` |
| `other` | `DOC` | `KA-DOC-001` |

---

## 5. Status Inference

```
INFER_STATUS(content, path):
  IF "deprecated" IN content (case-insensitive, in first 20 lines):
    RETURN "deprecated"
  IF "archived" IN path.parts:
    RETURN "archived"
  IF "draft" IN content (first 20 lines):
    RETURN "draft"
  IF line_count < 30:          # Stub documents
    RETURN "draft"
  RETURN "draft"               # Conservative default
```

All generated metadata defaults to `draft`. Upgrade to `approved` is a manual step.

---

## 6. Generated Frontmatter Template

```yaml
---
knowledge_id: KA-ARCH-042
version: "1.0.0"
status: draft
canonical: true
domain: DOM-WORKFLOW
capability: workflow-runtime
type: architecture
owner: Chief Knowledge Architect
phase: 2.0D.1
created: 2026-06-29
review_date: 2026-12-29
confidence: medium
# generated_by: MetadataGenerator 1.0.0
# migrated_from: <original relative path>
---
```

---

## 7. Repair Mode

When a document already has frontmatter but is missing required fields:

```
REPAIR_METADATA(existing_yaml, classification, registry):
  errors = validate(existing_yaml)  # Against KA-SPEC-005 schema
  
  FOR missing_field IN errors.missing_required:
    IF missing_field == "knowledge_id":
      existing_yaml["knowledge_id"] = assign_id(...)
    ELIF missing_field == "domain":
      existing_yaml["domain"] = classification.domain
    # ... etc
    
  RETURN repaired_yaml
```

Repair preserves all existing field values. It only fills gaps.

---

## 8. Apply Mode

```
APPLY_METADATA(path, metadata, dry_run=True):
  IF dry_run:
    LOG f"Would write frontmatter to {path}"
    RETURN
    
  content = read(path)
  IF has_frontmatter(content):
    content = replace_frontmatter(content, to_yaml(metadata))
  ELSE:
    content = prepend_frontmatter(content, to_yaml(metadata))
  write(path, content)
```

---

## References

- [KA-SPEC-003](../knowledge-core/03-KNOWLEDGE-IDENTITY-SPECIFICATION.md) — ID generation protocol
- [KA-SPEC-005](../knowledge-core/05-KNOWLEDGE-SCHEMA.md) — Schema all frontmatter must satisfy
- [KA-STD-002](../knowledge-architecture/METADATA-STANDARD.md) — Required metadata fields
- [03-DOCUMENT-CLASSIFIER.md](03-DOCUMENT-CLASSIFIER.md) — ClassificationResult source
