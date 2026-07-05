---
knowledge_id: KA-PLAT-007
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
  - id: KA-PLAT-003
    reason: "Reads RepositoryInventory to resolve relative paths"
  - id: KA-STD-004
    reason: "Relationship Model defines valid types and compatibility rules"
---

# Relationship Resolver

## Automatic Dependency and Relationship Discovery

---

## 1. Purpose

The RelationshipResolver discovers relationships between documents by analyzing
Markdown links, frontmatter references, heading cross-references, and textual
citations. It produces a `Relationship` list consumed by the IndexBuilder and
HealthAnalyzer.

---

## 2. Relationship Types Discovered

| Type | Discovery Method |
|------|-----------------|
| `references` | Markdown links: `[text](path.md)` |
| `depends_on` | Frontmatter `depends_on:` list |
| `implements` | Frontmatter `implements:` list |
| `implemented_by` | Frontmatter `implemented_by:` list |
| `supersedes` | Frontmatter `supersedes:` field |
| `superseded_by` | Frontmatter `superseded_by:` field |
| `related_to` | Frontmatter `related:` list |
| `duplicates` | Title similarity > 0.85 (CanonicalResolver provides this) |

---

## 3. Markdown Link Extraction

```
EXTRACT_MARKDOWN_LINKS(content, source_path, root_path):
  links = []
  FOR match IN MARKDOWN_LINK_PATTERN.findall(content):
    text, target = match
    IF target starts with "http": SKIP  # External link
    IF target starts with "#": 
      links.append(Relationship(source=source, target=source, type=SELF_REF))
      CONTINUE
    
    # Resolve relative path
    resolved = resolve_path(target, source_path, root_path)
    IF resolved is None: 
      LOG warning: broken link
      CONTINUE
      
    rel_type = classify_link(text, target, context_around_match)
    links.append(Relationship(
      source_path   = relative(source_path, root_path),
      target_path   = relative(resolved, root_path),
      relationship_type = rel_type,
      confidence    = 0.70,
      line_number   = match.line,
      anchor        = target.split("#")[1] if "#" in target else None
    ))
  RETURN links
```

---

## 4. Frontmatter Reference Extraction

Frontmatter fields that carry explicit relationship declarations are extracted
with high confidence (0.95) since they are authored intentionally:

```yaml
# Frontmatter fields → Relationship types
depends_on:   → DEPENDS_ON   (0.95)
implements:   → IMPLEMENTS   (0.95)
implemented_by: → IMPLEMENTED_BY (0.95)
supersedes:   → SUPERSEDES   (0.95)
superseded_by: → SUPERSEDED_BY (0.95)
related:      → RELATED_TO   (0.90)
related_apis: → REFERENCES   (0.90)
related_tests: → REFERENCES  (0.90)
```

---

## 5. Link Classification

Link text and surrounding context is used to upgrade `references` to a more
specific type:

| Signal | Upgraded Type | Confidence boost |
|--------|--------------|-----------------|
| Link text contains "implements" | `implements` | +0.15 |
| Link text contains "depends on" | `depends_on` | +0.15 |
| Link text contains "see also" | `related_to` | +0.10 |
| Link text contains "supersedes" | `supersedes` | +0.15 |
| Target is in `adr/` directory | Preserves `references` | 0 |
| Source is `implementation.md` | `implements` toward arch doc | +0.20 |

---

## 6. Path Resolution

```
RESOLVE_PATH(link_target, source_path, root_path):
  # Strip anchors
  path_part = link_target.split("#")[0]
  IF path_part == "": RETURN source_path  # Self-anchor
  
  # Resolve relative to source file's directory
  candidate = source_path.parent / path_part
  candidate = candidate.resolve()
  
  IF candidate.exists(): RETURN candidate
  
  # Try root-relative
  candidate = root_path / path_part
  IF candidate.exists(): RETURN candidate
  
  RETURN None  # Broken link
```

---

## 7. Reverse Inference

Per KA-STD-004, reverse relationships are inferred automatically:

| Forward | Inferred Reverse |
|---------|-----------------|
| `A implements B` | `B implemented_by A` |
| `A depends_on B` | `B used_by A` |
| `A supersedes B` | `B superseded_by A` |

The `inferred: true` flag marks reverse-inferred relationships in the index,
distinguishing them from explicitly declared ones.

---

## 8. Relationship Record

```yaml
Relationship:
  source_path: str          # Relative to root
  target_path: str          # Relative to root
  relationship_type: str    # RelationshipType enum value
  confidence: float         # 0.0 - 1.0
  line_number: int | None   # Line in source where discovered
  anchor: str | None        # Fragment identifier
  inferred: bool            # True if reverse-inferred
  discovery_method: str     # "markdown_link" | "frontmatter" | "inferred"
```

---

## 9. Broken Link Detection

The resolver collects all broken links (targets that do not resolve) as
`BrokenLinkRecord` entries. These feed directly into:
- RULE-R001 (broken references) in the Audit Engine
- Health score dimension: relationship (deducted per broken link)
- The `relationship-report.md` output

---

## References

- [KA-STD-004](../knowledge-architecture/RELATIONSHIP-MODEL.md) — Relationship model and type definitions
- [08-INDEX-BUILDER.md](08-INDEX-BUILDER.md) — Consumes Relationship list
- [09-HEALTH-ANALYZER.md](09-HEALTH-ANALYZER.md) — Uses relationships in scoring
