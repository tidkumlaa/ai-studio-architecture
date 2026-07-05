---
knowledge_id: KA-PLAT-008
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
    reason: "Consumes ClassificationResult to find docs of the same type+capability"
  - id: KA-STD-003
    reason: "Canonical Source Strategy defines duplicate and orphan rules"
---

# Canonical Resolver

## Duplicate, Orphan, and Dead Document Detection

---

## 1. Purpose

The CanonicalResolver enforces the canonical source rule: "One concept. One
canonical document. Unlimited references. Zero copies." It detects duplicate
documents, orphans, dead documents, and multiple canonical sources, and produces
a `CanonicalMap` with recommended resolutions.

---

## 2. Detection Categories

| Category | Definition | Rule |
|----------|-----------|------|
| Duplicate | Two documents covering the same concept | D1-D4 |
| Obsolete | Document superseded by a newer version | OB1-OB5 |
| Orphan | Document with no inbound references | KA-STD-003 |
| Dead | Document linked-to from nowhere and links nowhere | KA-STD-003 |
| Multi-canonical | Two documents both declare `canonical: true` for same type+cap | RULE-C001 |

---

## 3. Duplicate Detection Algorithms

### D1 — Title Similarity

Documents with H1 headings that are >85% similar (normalized Levenshtein) are
flagged as potential duplicates:

```
FOR EACH pair (a, b) IN same_capability_docs:
  sim = normalized_levenshtein(a.h1_heading, b.h1_heading)
  IF sim > 0.85:
    FLAG as duplicate, confidence = sim
```

### D2 — Same Filename in Different Directories

Files with identical names in different directories may be copies:

```
filename_groups = group_by(inventory, key=lambda r: r.path.name)
FOR EACH group WITH len > 1:
  IF any two in group have same type classification:
    FLAG as duplicate, confidence = 0.70
```

### D3 — Content Hash Match

Files with identical content (after stripping frontmatter) are definite duplicates:

```
content_hash_map = {}
FOR EACH doc IN inventory:
  body = strip_frontmatter(read(doc.path))
  h = sha256(body)
  IF h IN content_hash_map:
    FLAG both as exact duplicate, confidence = 1.0
  ELSE:
    content_hash_map[h] = doc
```

### D4 — Section Overlap

A document whose sections are a subset of another document's sections indicates
a split that was not cleaned up:

```
FOR EACH pair (a, b) IN same_capability_docs:
  a_headings = extract_headings(a)
  b_headings = extract_headings(b)
  overlap = intersection(a_headings, b_headings)
  IF len(overlap) / min(len(a_headings), len(b_headings)) > 0.70:
    FLAG smaller as partial_duplicate of larger
```

---

## 4. Orphan Detection

```
DETECT_ORPHANS(inventory, relationships):
  all_targets = {r.target_path for r in relationships}
  all_paths   = {doc.relative_path for doc in inventory.documents}
  
  FOR EACH path IN all_paths:
    IF path NOT IN all_targets:
      IF path is not a root README or index:
        FLAG as orphan
```

---

## 5. Dead Document Detection

A dead document:
1. Has no inbound references (orphan)
2. Has no outbound references (links nowhere)
3. Was not modified in the last 90 days (configurable)
4. Status is not `approved` or `review`

```
DETECT_DEAD(inventory, relationships, threshold_days=90):
  inbound  = {r.target_path for r in relationships}
  outbound = {r.source_path for r in relationships}
  
  FOR EACH doc IN inventory.documents:
    path = doc.relative_path
    age  = days_since(doc.last_modified)
    IF path NOT IN inbound AND path NOT IN outbound AND age > threshold_days:
      FLAG as dead
```

---

## 6. Multi-Canonical Detection

```
DETECT_MULTI_CANONICAL(classified, metadata_map):
  canonical_by_key = {}  # key = (capability, type)
  
  FOR EACH (path, meta) IN metadata_map:
    IF meta.canonical == True:
      key = (meta.capability, meta.document_type)
      IF key IN canonical_by_key:
        FLAG both as multi-canonical violation (RULE-C001)
      ELSE:
        canonical_by_key[key] = path
```

---

## 7. Canonical Map Output

```yaml
CanonicalMap:
  generated: "2026-06-29"
  root_path: str
  canonical_entries:
    - concept: "workflow-runtime-architecture"
      canonical_path: "docs/workflow/architecture.md"
      duplicates: ["docs/old-workflow/arch-v1.md"]
      resolution: "archive_duplicates"
      confidence: 0.92
  orphans:
    - path: "docs/misc/forgotten-doc.md"
      age_days: 180
      recommendation: "archive or delete"
  dead_documents:
    - path: "docs/old/experiment.md"
      age_days: 340
      recommendation: "archive"
  multi_canonical_violations:
    - capability: "workflow-runtime"
      type: "architecture"
      paths: ["path/a.md", "path/b.md"]
      recommendation: "set canonical: false on b.md"
```

---

## References

- [KA-STD-003](../knowledge-architecture/CANONICAL-SOURCE-STRATEGY.md) — Canonical strategy
- [07-CANONICAL-RESOLVER.md](07-CANONICAL-RESOLVER.md) — This document
- [08-INDEX-BUILDER.md](08-INDEX-BUILDER.md) — CanonicalMap feeds canonical index
- [12-VERIFICATION.md](12-VERIFICATION.md) — Canonical report output
