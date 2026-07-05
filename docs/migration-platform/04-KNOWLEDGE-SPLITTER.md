---
knowledge_id: KA-PLAT-005
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
    reason: "Splitter uses ClassificationResult to name output files"
  - id: KA-MICRO-001
    reason: "Splitting rules derived from Micro Knowledge Architecture"
---

# Knowledge Splitter

## Oversized Document Detection and Section-Level Splitting

---

## 1. Purpose

The KnowledgeSplitter detects documents exceeding the 500-line limit defined by
KA-MICRO-001 and produces a `SplitPlan` describing how to divide them. Plans are
executed only when `--apply` is used; by default the splitter is read-only.

---

## 2. Split Thresholds

| Condition | Action |
|-----------|--------|
| `line_count <= 400` | No split needed |
| `400 < line_count <= 500` | Warning only — document in warning zone |
| `line_count > 500` | Split required — generate SplitPlan |

---

## 3. Split Point Detection

The splitter identifies natural split points by scanning for headings at the
configured heading levels. Default: H2 and H3 headings (`##`, `###`).

```
FIND_SPLIT_POINTS(content, max_lines=500):
  sections = []
  current_start = 0
  current_heading = "preamble"
  current_level = 0
  
  FOR i, line IN enumerate(content.lines):
    IF line matches HEADING_PATTERN:
      level = count_leading_hashes(line)
      IF level <= split_heading_level:
        IF current section > max_lines:
          # Force split here regardless
          MARK as forced split
        sections.append(SectionRecord(
          heading      = current_heading,
          heading_level = current_level,
          line_start   = current_start,
          line_end     = i - 1,
          line_count   = i - current_start
        ))
        current_start   = i
        current_heading = extract_heading_text(line)
        current_level   = level
        
  # Last section
  sections.append(SectionRecord(..., line_end=last_line))
  RETURN sections
```

---

## 4. Output Filename Generation

Each section becomes a separate file named from the section heading:

```
GENERATE_FILENAME(heading, parent_path, classification):
  slug = to_kebab_case(heading)
  slug = truncate(slug, max=60)
  base = parent_path.stem  # original filename without extension
  RETURN f"{base}-{slug}.md"
```

Examples:
- `AI-STUDIO-MASTER-ARCHITECTURE.md` → Section "Workflow Runtime" →
  `AI-STUDIO-MASTER-ARCHITECTURE-workflow-runtime.md`

---

## 5. Link Preservation

When a document is split, all internal links must be updated. The splitter:

1. Collects all heading anchors in the source document
2. Maps each anchor to its new file location
3. Rewrites all `[text](#anchor)` links to `[text](new-file.md#anchor)`
4. Rewrites all `[text](same-file.md#anchor)` cross-references

```
REWRITE_LINKS(content, anchor_map):
  FOR match IN find_markdown_links(content):
    IF match.target starts with "#":
      anchor = match.target[1:]
      IF anchor IN anchor_map:
        new_target = anchor_map[anchor]  # "new-file.md#anchor"
        content = replace(match, new_target)
  RETURN content
```

---

## 6. SplitPlan Structure

```yaml
SplitPlan:
  source_path: Path
  total_lines: int
  split_reason: str           # "exceeds 500 lines" or "forced: section exceeds limit"
  estimated_output_count: int
  sections:
    - heading: str
      heading_level: int
      line_start: int
      line_end: int
      line_count: int
      output_filename: str
      anchors: list[str]      # Heading anchors within this section
  anchor_map:                 # anchor -> "output_filename#anchor"
    heading-text: new-file.md#heading-text
```

---

## 7. Execution

```
EXECUTE_SPLIT(plan, content, dry_run=True):
  IF dry_run:
    LOG all planned files
    RETURN []
    
  RollbackManager.checkpoint("pre-split")
  
  output_files = []
  FOR section IN plan.sections:
    section_content = content[section.line_start : section.line_end + 1]
    section_content = prepend_heading(section_content, section.heading)
    section_content = rewrite_links(section_content, plan.anchor_map)
    output_path = plan.source_path.parent / section.output_filename
    write(output_path, section_content)
    output_files.append(output_path)
    
  # Mark source as deprecated, add pointer to split files
  mark_split_source(plan.source_path, output_files)
  RETURN output_files
```

---

## 8. Special Cases

**Preamble sections** (content before the first heading) are prepended to the first
section or written to a `-overview.md` file if they exceed 50 lines.

**Code blocks** are never split mid-block, even if a heading appears inside a
fenced code block. The splitter tracks code fence state.

**Tables** are never split mid-table.

**Frontmatter** from the source document is propagated to all output files with
`superseded_by` pointing to the next file in sequence.

---

## References

- [KA-MICRO-001](../knowledge-architecture/MICRO-KNOWLEDGE-ARCHITECTURE.md) — Line limits that trigger splitting
- [05-METADATA-GENERATOR.md](05-METADATA-GENERATOR.md) — Generates frontmatter for split files
- [06-RELATIONSHIP-RESOLVER.md](06-RELATIONSHIP-RESOLVER.md) — Updates relationship links after splits
