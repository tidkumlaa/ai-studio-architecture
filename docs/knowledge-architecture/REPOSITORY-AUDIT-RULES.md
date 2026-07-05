---
knowledge_id: KA-STD-008
version: "1.0.0"
status: approved
owner: Chief Knowledge Architect
phase: 2.0D.0
created: 2026-06-29
review_date: 2026-12-29
canonical: true
implements:
  - KA-VISION-001
  - KA-STD-001
  - KA-STD-002
related:
  - id: KA-STD-008B
    title: "Repository Audit Rules — Size, Reference, Canonical, Health"
    file: REPOSITORY-AUDIT-RULES-PART2.md
---

# Repository Audit Rules — Part 1

## Structure and Metadata Rules

---

## 1. Purpose

Audit rules are the enforcement mechanism of the knowledge architecture. Standards decay through neglect without automated enforcement.

The audit tool reads this rule set and produces:
- A machine-readable audit report (`health-report.yaml`)
- A human-readable summary (`HEALTH-REPORT.md`)
- Blocking errors that prevent merge
- Warnings that require acknowledgement

**The audit tool is implemented in Phase 2.0D.1.** This document and [REPOSITORY-AUDIT-RULES-PART2.md](REPOSITORY-AUDIT-RULES-PART2.md) form its complete specification.

---

## 2. Rule Structure

Every rule has the following attributes:

```yaml
rule_id: RULE-001
category: structure       # structure | metadata | size | reference | canonical | health
severity: error           # error | warning | info
scope: document           # document | folder | capability | repository
auto_fix: false           # Can the audit tool auto-fix this?
blocks_merge: true        # Does violation prevent merge to main?
description: "Rule description"
check: |
  Pseudocode for the check
remediation: |
  What the author must do to resolve it
```

---

## 3. Structure Rules

### RULE-S001 — README.md Required

```yaml
rule_id: RULE-S001
category: structure
severity: error
scope: folder
auto_fix: false
blocks_merge: true
description: Every capability folder must contain a README.md file
check: |
  FOR each directory in capabilities/:
    IF README.md does not exist: VIOLATION
remediation: Create README.md using the template in DOCUMENT-FOLDER-STANDARDS.md
```

### RULE-S002 — index.yaml Required

```yaml
rule_id: RULE-S002
category: structure
severity: error
scope: folder
auto_fix: false
blocks_merge: true
description: Every capability folder must contain an index.yaml file
check: |
  FOR each directory in capabilities/:
    IF index.yaml does not exist: VIOLATION
remediation: Create index.yaml using the schema in DOCUMENT-FOLDER-STANDARDS.md
```

### RULE-S003 — Required Files Present

```yaml
rule_id: RULE-S003
category: structure
severity: error
scope: capability
auto_fix: false
blocks_merge: true
description: |
  Every capability folder must contain: overview.md, architecture.md,
  implementation.md, testing.md, roadmap.md, history.md, references.md
check: |
  required = [overview.md, architecture.md, implementation.md,
               testing.md, roadmap.md, history.md, references.md]
  FOR each directory in capabilities/:
    FOR each file in required:
      IF file does not exist: VIOLATION(directory, file)
remediation: Create the missing file using the template in DOCUMENT-FOLDER-STANDARDS.md
```

### RULE-S004 — No Unauthorized Filenames

```yaml
rule_id: RULE-S004
category: structure
severity: warning
scope: folder
auto_fix: false
blocks_merge: false
description: Capability folders should only contain approved filenames.
check: |
  approved = [README.md, index.yaml, overview.md, architecture.md,
               api.md, database.md, events.md, security.md, desktop.md,
               implementation.md, testing.md, roadmap.md, history.md, references.md]
  FOR each file in capabilities/[cap]/ (excluding sub-folders):
    IF file not in approved: VIOLATION(file)
remediation: Rename file to approved name, or move to a sub-folder
```

### RULE-S005 — Maximum Sub-Folder Depth

```yaml
rule_id: RULE-S005
category: structure
severity: error
scope: repository
auto_fix: false
blocks_merge: true
description: Sub-folders must not exceed 2 levels below the capability root
check: |
  FOR each path in capabilities/:
    depth = path.split('/').length - base_depth
    IF depth > 2: VIOLATION(path)
remediation: Flatten sub-folder hierarchy; split into peer documents instead
```

### RULE-S006 — All Files in index.yaml

```yaml
rule_id: RULE-S006
category: structure
severity: warning
scope: capability
auto_fix: true
blocks_merge: false
description: Every .md file in a capability folder must be listed in index.yaml
check: |
  FOR each capability in capabilities/:
    index_files = parse(index.yaml).documents[*].file
    disk_files = glob(capability/*.md)
    FOR each file in disk_files:
      IF file not in index_files: VIOLATION(capability, file)
auto_fix_action: Add missing file entry to index.yaml with placeholder metadata
```

---

## 4. Metadata Rules

### RULE-M001 — knowledge_id Required

```yaml
rule_id: RULE-M001
category: metadata
severity: error
scope: document
auto_fix: false
blocks_merge: true
description: Every knowledge document must declare a knowledge_id
check: |
  FOR each .md file in capabilities/ or adr/ or knowledge-architecture/:
    IF frontmatter.knowledge_id is missing: VIOLATION
remediation: Add knowledge_id using the ID format from METADATA-STANDARD.md
```

### RULE-M002 — knowledge_id Unique

```yaml
rule_id: RULE-M002
category: metadata
severity: error
scope: repository
auto_fix: false
blocks_merge: true
description: Every knowledge_id must be unique across the entire repository
check: |
  ids = collect all frontmatter.knowledge_id values
  duplicates = ids where count > 1
  IF duplicates not empty: VIOLATION(each duplicate)
remediation: Assign a new unique ID to the duplicate; register in KNOWLEDGE-REGISTRY.yaml
```

### RULE-M003 — Status Required and Valid

```yaml
rule_id: RULE-M003
category: metadata
severity: error
scope: document
auto_fix: false
blocks_merge: true
valid_values: [draft, review, approved, deprecated, archived]
check: |
  FOR each document:
    IF frontmatter.status is missing OR not in valid_values: VIOLATION
```

### RULE-M004 — Owner Required

```yaml
rule_id: RULE-M004
category: metadata
severity: error
scope: document
auto_fix: false
blocks_merge: true
check: |
  FOR each document WHERE status == 'approved':
    IF frontmatter.owner is missing or empty: VIOLATION
```

### RULE-M005 — review_date Required for Approved Documents

```yaml
rule_id: RULE-M005
category: metadata
severity: error
scope: document
auto_fix: false
blocks_merge: true
exception: ADR documents (type == 'adr') are exempt
check: |
  FOR each document WHERE status == 'approved' AND type != 'adr':
    IF frontmatter.review_date is missing or empty: VIOLATION
```

### RULE-M006 — domain Required and Valid

```yaml
rule_id: RULE-M006
category: metadata
severity: error
scope: document
auto_fix: false
blocks_merge: true
check: |
  valid_domains = load DOMAIN-INDEX.yaml domains[*].id
  FOR each document:
    IF frontmatter.domain not in valid_domains: VIOLATION
```

### RULE-M007 — capability Required and Consistent

```yaml
rule_id: RULE-M007
category: metadata
severity: error
scope: document
auto_fix: true
blocks_merge: true
check: |
  FOR each document in capabilities/[cap]/:
    IF frontmatter.capability != cap: VIOLATION
auto_fix_action: Set frontmatter.capability to the containing folder name
```

### RULE-M008 — version is Valid Semver

```yaml
rule_id: RULE-M008
category: metadata
severity: error
scope: document
auto_fix: false
blocks_merge: true
check: |
  semver_pattern = /^\d+\.\d+\.\d+$/
  FOR each document:
    IF NOT semver_pattern.match(frontmatter.version): VIOLATION
```

### RULE-M009 — canonical Declared

```yaml
rule_id: RULE-M009
category: metadata
severity: warning
scope: document
auto_fix: false
blocks_merge: false
check: |
  FOR each document WHERE status == 'approved':
    IF frontmatter.canonical is missing: VIOLATION
```

### RULE-M010 — Deprecated Document Has superseded_by

```yaml
rule_id: RULE-M010
category: metadata
severity: warning
scope: document
auto_fix: false
blocks_merge: false
check: |
  FOR each document WHERE status == 'deprecated':
    IF frontmatter.superseded_by is missing or empty: VIOLATION
```

---

## 5. Continued in Part 2

Size, reference, canonical, health, and ownership rules, plus execution instructions and the full rule summary table, are in [REPOSITORY-AUDIT-RULES-PART2.md](REPOSITORY-AUDIT-RULES-PART2.md).

---

## References

- [REPOSITORY-AUDIT-RULES-PART2.md](REPOSITORY-AUDIT-RULES-PART2.md) — Size, reference, canonical, health, ownership rules
- [METADATA-STANDARD.md](METADATA-STANDARD.md) — Source of metadata rules
- [DOCUMENT-FOLDER-STANDARDS.md](DOCUMENT-FOLDER-STANDARDS.md) — Source of structure rules
- [MICRO-KNOWLEDGE-ARCHITECTURE.md](MICRO-KNOWLEDGE-ARCHITECTURE.md) — Source of size rules
- [KNOWLEDGE-HEALTH-METRICS.md](KNOWLEDGE-HEALTH-METRICS.md) — Source of health rules
