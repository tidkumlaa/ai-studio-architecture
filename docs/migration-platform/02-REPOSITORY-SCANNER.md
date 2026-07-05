---
knowledge_id: KA-PLAT-003
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
  - id: KA-PLAT-002
    reason: "Scanner is Stage 1 of the migration pipeline"
---

# Repository Scanner

## File Discovery, Format Detection, and Inventory Generation

---

## 1. Purpose

The KnowledgeScanner performs non-destructive discovery of all files in a repository.
It produces a `RepositoryInventory` — the complete, structured record of every file
that subsequent pipeline stages will process. The scanner never modifies files.

---

## 2. Detected File Types

| Extension | Format | Description |
|-----------|--------|-------------|
| `.md` | Markdown | Primary knowledge documents |
| `.yaml`, `.yml` | YAML | Configuration, indexes, schemas |
| `.json` | JSON | Exchange formats, schemas |
| `.py` | Python | Implementation files |
| `.ts`, `.tsx` | TypeScript | Desktop implementation |
| `.sql` | SQL | Database schemas |
| `Dockerfile` | Docker | Infrastructure |
| `.toml` | TOML | Configuration |
| Other | Unknown | Recorded but not processed |

---

## 3. Inventory Record Structure

Each discovered file produces a `DocumentRecord`:

```yaml
DocumentRecord:
  path: Path                    # Absolute path
  relative_path: str            # Relative to root
  size_bytes: int               # File size
  line_count: int               # Line count (text files only)
  format: DocumentFormat        # Enum: markdown, yaml, json, python, ...
  has_frontmatter: bool         # True if YAML --- block present at line 1
  existing_id: str | None       # knowledge_id from frontmatter if present
  is_oversized: bool            # True if line_count > 500
  is_empty: bool                # True if size_bytes == 0
  last_modified: str            # ISO 8601 mtime
```

---

## 4. Scan Algorithm

```
SCAN(root_path):
  records = []
  FOR EACH path IN walk(root_path):
    IF path matches any exclude_pattern: SKIP
    IF path is directory:
      IF depth > max_depth: SKIP
      CONTINUE
    record = DocumentRecord(
      path        = path,
      relative_path = relative(path, root_path),
      size_bytes  = file_size(path),
      line_count  = count_lines(path) IF is_text(path) ELSE 0,
      format      = detect_format(path),
      has_frontmatter = check_frontmatter(path) IF format == MARKDOWN,
      existing_id = extract_id(path) IF has_frontmatter,
      is_oversized = line_count > MAX_LINES,
      is_empty    = size_bytes == 0,
      last_modified = mtime(path).isoformat()
    )
    records.append(record)
  RETURN RepositoryInventory(root=root_path, documents=records)
```

**Complexity:** O(D) where D = total file count. Single directory walk.

---

## 5. Frontmatter Detection

A file has frontmatter if:
1. First line is exactly `---`
2. A second `---` appears within the first 50 lines
3. Content between delimiters is valid YAML

The scanner extracts `knowledge_id` from frontmatter if present. This allows
incremental runs to skip already-migrated documents.

```
CHECK_FRONTMATTER(path):
  lines = read_first_50_lines(path)
  IF lines[0] != "---": RETURN False
  FOR i IN 1..len(lines):
    IF lines[i] == "---":
      yaml_block = join(lines[1:i])
      TRY: parsed = yaml.safe_load(yaml_block)
      EXCEPT: RETURN False
      RETURN True
  RETURN False
```

---

## 6. Exclude Patterns

Default exclude patterns (configurable):

```yaml
exclude_patterns:
  - ".git/**"
  - "node_modules/**"
  - "__pycache__/**"
  - "*.pyc"
  - ".venv/**"
  - "dist/**"
  - "*.min.js"
  - "*.min.css"
  - "archive/**"        # Optionally exclude archive (configurable)
```

---

## 7. Inventory Statistics

The `RepositoryInventory` includes aggregate statistics:

```yaml
RepositoryInventory:
  root_path: Path
  scan_timestamp: str               # ISO 8601
  total_files: int
  total_size_bytes: int
  documents: list[DocumentRecord]
  by_format:                        # Count per DocumentFormat
    markdown: int
    yaml: int
    json: int
    python: int
    other: int
  oversized_count: int              # line_count > 500
  with_frontmatter_count: int
  without_frontmatter_count: int
  already_migrated_count: int       # has existing_id
  empty_files_count: int
```

---

## 8. Output

```bash
python migrate.py scan /path/to/repo

# Console output (via rich):
Scanning: /path/to/repo
  Files found:      786
  Markdown:         612
  YAML/JSON:        94
  Other:            80
  With frontmatter: 12
  Oversized (>500): 23
  Already migrated: 12
  Elapsed:          0.4s

# Report file: reports/inventory.yaml
```

---

## References

- [01-MIGRATION-ENGINE-ARCHITECTURE.md](01-MIGRATION-ENGINE-ARCHITECTURE.md) — Pipeline context
- [03-DOCUMENT-CLASSIFIER.md](03-DOCUMENT-CLASSIFIER.md) — Next stage (consumes Inventory)
- [KA-SPEC-016](../knowledge-core/16-DATA-STRUCTURE-CATALOG.md) — DocumentRecord data structure
