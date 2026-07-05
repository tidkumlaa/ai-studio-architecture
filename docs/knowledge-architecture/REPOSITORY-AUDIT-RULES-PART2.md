---
knowledge_id: KA-STD-008B
version: "1.0.0"
status: approved
owner: Chief Knowledge Architect
phase: 2.0D.0
created: 2026-06-29
review_date: 2026-12-29
canonical: true
implements:
  - KA-VISION-001
related:
  - id: KA-STD-008
    title: "Repository Audit Rules — Structure and Metadata"
    file: REPOSITORY-AUDIT-RULES.md
---

# Repository Audit Rules — Part 2

## Size, Reference, Canonical, Health, Ownership Rules and Execution

---

## 1. Size Rules

### RULE-SZ001 — Maximum Document Length (Hard Limit)

```yaml
rule_id: RULE-SZ001
category: size
severity: error
scope: document
auto_fix: false
blocks_merge: true
description: No knowledge document may exceed 500 lines
check: |
  FOR each .md file in capabilities/:
    IF count_lines(file) > 500: VIOLATION(file, lines)
exception: Files in archive/ are exempt (historical, read-only)
remediation: Split per MICRO-KNOWLEDGE-ARCHITECTURE.md
```

### RULE-SZ002 — Document Approaching Limit (Warning)

```yaml
rule_id: RULE-SZ002
category: size
severity: warning
scope: document
auto_fix: false
blocks_merge: false
description: Documents 400–500 lines should be reviewed for split candidacy
check: |
  FOR each .md file in capabilities/:
    lines = count_lines(file)
    IF lines >= 400 AND lines <= 500: VIOLATION(file, lines)
```

### RULE-SZ003 — Maximum Heading Depth

```yaml
rule_id: RULE-SZ003
category: size
severity: error
scope: document
auto_fix: false
blocks_merge: true
description: Documents must not contain headings below H3
check: |
  FOR each .md file:
    FOR each line:
      IF line starts with "#### ": VIOLATION(file, line_number)
remediation: Extract H4+ content to a sub-document
```

### RULE-SZ004 — README Size Limit

```yaml
rule_id: RULE-SZ004
category: size
severity: warning
scope: document
auto_fix: false
blocks_merge: false
description: README.md files should not exceed 150 lines
check: |
  FOR each README.md:
    IF count_lines(README.md) > 150: VIOLATION
```

---

## 2. Reference Rules

### RULE-R001 — No Broken Internal Links

```yaml
rule_id: RULE-R001
category: reference
severity: error
scope: repository
auto_fix: false
blocks_merge: true
description: All internal markdown links must resolve to existing files
check: |
  FOR each .md file:
    FOR each [text](link) where link is a relative path:
      resolved = resolve(link, relative_to=file)
      IF NOT file_exists(resolved): VIOLATION(file, link, line_number)
remediation: Update the link or remove it
```

### RULE-R002 — Knowledge ID References Must Exist

```yaml
rule_id: RULE-R002
category: reference
severity: error
scope: repository
auto_fix: false
blocks_merge: true
description: All knowledge IDs referenced in frontmatter must exist in KNOWLEDGE-REGISTRY.yaml
check: |
  registry = load KNOWLEDGE-REGISTRY.yaml
  FOR each document:
    FOR each ID in [implements, depends_on, superseded_by, related_*]:
      IF ID not in registry: VIOLATION(document, ID)
remediation: Register the ID, or remove the broken reference
```

### RULE-R003 — index.yaml Line Counts Accurate

```yaml
rule_id: RULE-R003
category: reference
severity: warning
scope: capability
auto_fix: true
blocks_merge: false
check: |
  FOR each capability:
    FOR each document entry in index.yaml:
      IF entry.lines != count_lines(entry.file): VIOLATION
auto_fix_action: Update entry.lines in index.yaml to match actual count
```

### RULE-R004 — superseded_by Chain Must Terminate

```yaml
rule_id: RULE-R004
category: reference
severity: error
scope: repository
auto_fix: false
blocks_merge: true
description: Supersession chains must not form cycles
check: |
  FOR each document with superseded_by:
    follow chain until terminal or cycle detected
    IF cycle: VIOLATION
remediation: Break the cycle; ensure chain terminates at a current canonical document
```

---

## 3. Canonical Rules

### RULE-C001 — Single Canonical Per Concept

```yaml
rule_id: RULE-C001
category: canonical
severity: error
scope: repository
auto_fix: false
blocks_merge: true
description: No two approved documents with the same capability+type may both declare canonical: true
check: |
  FOR each (capability, type) combination:
    canonical_docs = documents WHERE canonical=true AND status='approved'
    IF count > 1: VIOLATION(capability, type, canonical_docs)
remediation: Set canonical: false on all but one; add superseded_by references
```

### RULE-C002 — Deprecated Documents Not Canonical

```yaml
rule_id: RULE-C002
category: canonical
severity: error
scope: document
auto_fix: true
blocks_merge: true
check: |
  FOR each document WHERE status == 'deprecated':
    IF frontmatter.canonical == true: VIOLATION
auto_fix_action: Set canonical: false on deprecated documents
```

### RULE-C003 — Canonical Source Map Integrity

```yaml
rule_id: RULE-C003
category: canonical
severity: error
scope: repository
auto_fix: true
blocks_merge: false
description: CANONICAL-SOURCE-MAP.yaml must be current with all canonical documents
check: |
  canonical_docs = all documents WHERE canonical = true
  map = load CANONICAL-SOURCE-MAP.yaml
  FOR each doc in canonical_docs:
    IF doc.knowledge_id not in map.concepts: VIOLATION
auto_fix_action: Regenerate CANONICAL-SOURCE-MAP.yaml from frontmatter
```

---

## 4. Health Rules

### RULE-H001 — Stale Document Warning (30 days)

```yaml
rule_id: RULE-H001
category: health
severity: warning
scope: document
auto_fix: false
blocks_merge: false
check: |
  FOR each document WHERE status == 'approved':
    IF today - review_date > 30 days: VIOLATION(document, days_overdue)
remediation: Owner must review document and update review_date
exception: ADR and history documents are exempt
```

### RULE-H002 — Stale Document Error (90 days)

```yaml
rule_id: RULE-H002
category: health
severity: error
scope: document
auto_fix: false
blocks_merge: true
check: |
  FOR each document WHERE status == 'approved':
    IF today - review_date > 90 days: VIOLATION(document, days_overdue)
remediation: Owner must immediately review or deprecate the document
exception: ADR and history documents are exempt
```

### RULE-H003 — Low Health Score Warning

```yaml
rule_id: RULE-H003
category: health
severity: warning
scope: document
auto_fix: false
blocks_merge: false
check: |
  FOR each document:
    score = compute_health_score(document)
    IF score < 60: VIOLATION(document, score)
remediation: Improve per KNOWLEDGE-HEALTH-METRICS.md
```

### RULE-H004 — Orphan Document Detection

```yaml
rule_id: RULE-H004
category: health
severity: warning
scope: repository
auto_fix: false
blocks_merge: false
description: Documents with no inbound or outbound references
check: |
  FOR each document:
    outbound = count references in frontmatter
    inbound = count in relationship-index.yaml
    IF outbound == 0 AND inbound == 0: VIOLATION
exception: overview.md and root-level README.md are exempt
```

### RULE-H005 — Repository Health Score Gate

```yaml
rule_id: RULE-H005
category: health
severity: info
scope: repository
auto_fix: false
blocks_merge: false
description: Health score is tracked; gate at 80 for Phase 2.0D.2
check: |
  score = compute_repository_health_score()
  IF score < 80: REPORT(score, "Below Phase 2.0D.2 gate threshold")
```

---

## 5. Ownership Rules

### RULE-O001 — Ownerless Approved Documents

```yaml
rule_id: RULE-O001
category: metadata
severity: error
scope: document
auto_fix: false
blocks_merge: true
check: |
  FOR each document WHERE status == 'approved':
    IF frontmatter.owner is missing or empty: VIOLATION
```

### RULE-O002 — Review Schedule Declared

```yaml
rule_id: RULE-O002
category: metadata
severity: error
scope: document
auto_fix: false
blocks_merge: true
check: |
  FOR each document WHERE status == 'approved' AND type NOT IN ['adr', 'history']:
    IF frontmatter.review_date is missing: VIOLATION
```

---

## 6. Audit Execution

### 6.1 Invocation

```bash
python tools/audit.py --all              # Full audit
python tools/audit.py --category metadata
python tools/audit.py --path capabilities/workflow-runtime/
python tools/audit.py --report           # Generate HEALTH-REPORT.md
python tools/audit.py --ci              # Exit 1 if any errors
```

### 6.2 Output Format

```yaml
# audit-report.yaml
generated: 2026-06-29T14:32:00Z
total_documents: 786
total_violations: 47
  errors: 12
  warnings: 35

violations:
  - rule_id: RULE-M001
    severity: error
    document: capabilities/workflow-runtime/api.md
    message: "knowledge_id is missing"
    remediation: "Add knowledge_id using ID format KA-API-NNN"

health_score: 67
phase_gate_2_0_D_2: FAIL
```

### 6.3 CI/CD Integration

```yaml
# .github/workflows/knowledge-audit.yml
name: Knowledge Audit
on: [push, pull_request]
jobs:
  audit:
    steps:
      - uses: actions/checkout@v4
      - run: python tools/audit.py --ci
      - run: python tools/generate-indexes.py
      - run: python tools/audit.py --report --output HEALTH-REPORT.md
```

Merge is blocked when any `blocks_merge: true` rule has a violation.

---

## 7. Complete Rule Summary

| Rule ID | Category | Severity | Blocks Merge | Auto-Fix |
|---------|----------|----------|-------------|---------|
| RULE-S001 | Structure | Error | Yes | No |
| RULE-S002 | Structure | Error | Yes | No |
| RULE-S003 | Structure | Error | Yes | No |
| RULE-S004 | Structure | Warning | No | No |
| RULE-S005 | Structure | Error | Yes | No |
| RULE-S006 | Structure | Warning | No | Yes |
| RULE-M001 | Metadata | Error | Yes | No |
| RULE-M002 | Metadata | Error | Yes | No |
| RULE-M003 | Metadata | Error | Yes | No |
| RULE-M004 | Metadata | Error | Yes | No |
| RULE-M005 | Metadata | Error | Yes | No |
| RULE-M006 | Metadata | Error | Yes | No |
| RULE-M007 | Metadata | Error | Yes | Yes |
| RULE-M008 | Metadata | Error | Yes | No |
| RULE-M009 | Metadata | Warning | No | No |
| RULE-M010 | Metadata | Warning | No | No |
| RULE-SZ001 | Size | Error | Yes | No |
| RULE-SZ002 | Size | Warning | No | No |
| RULE-SZ003 | Size | Error | Yes | No |
| RULE-SZ004 | Size | Warning | No | No |
| RULE-R001 | Reference | Error | Yes | No |
| RULE-R002 | Reference | Error | Yes | No |
| RULE-R003 | Reference | Warning | No | Yes |
| RULE-R004 | Reference | Error | Yes | No |
| RULE-C001 | Canonical | Error | Yes | No |
| RULE-C002 | Canonical | Error | Yes | Yes |
| RULE-C003 | Canonical | Error | No | Yes |
| RULE-H001 | Health | Warning | No | No |
| RULE-H002 | Health | Error | Yes | No |
| RULE-H003 | Health | Warning | No | No |
| RULE-H004 | Health | Warning | No | No |
| RULE-H005 | Health | Info | No | No |
| RULE-O001 | Ownership | Error | Yes | No |
| RULE-O002 | Ownership | Error | Yes | No |

**Totals:** 34 rules — 20 errors (19 block merge), 12 warnings, 1 info, 7 auto-fixable

---

## References

- [REPOSITORY-AUDIT-RULES.md](REPOSITORY-AUDIT-RULES.md) — Structure and metadata rules (Part 1)
- [METADATA-STANDARD.md](METADATA-STANDARD.md) — Source of metadata rules
- [DOCUMENT-FOLDER-STANDARDS.md](DOCUMENT-FOLDER-STANDARDS.md) — Source of structure rules
- [MICRO-KNOWLEDGE-ARCHITECTURE.md](MICRO-KNOWLEDGE-ARCHITECTURE.md) — Source of size rules
- [CANONICAL-SOURCE-STRATEGY.md](CANONICAL-SOURCE-STRATEGY.md) — Source of canonical rules
- [KNOWLEDGE-HEALTH-METRICS.md](KNOWLEDGE-HEALTH-METRICS.md) — Source of health rules
