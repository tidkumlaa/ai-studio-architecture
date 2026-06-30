# KNW-KE-ARCH-006 — Knowledge Linter

**Phase:** 3.0C.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

`kos lint` is the machine-executable enforcer of all Knowledge Ecosystem rules. A repository with lint errors may not be merged to main or released.

---

## Command Interface

```bash
kos lint [PATH]                  # lint file, directory, or package
kos lint --package {name}        # lint a specific package
kos lint --fix                   # auto-fix fixable violations
kos lint --severity error        # show only errors (not warnings/info)
kos lint --format json           # machine-readable output
kos lint --rule KL-007           # run only one check
```

**Exit codes:**
```
0  — no violations
1  — warnings only
2  — at least one ERROR
3  — linter itself failed (bug)
```

---

## Output Format

```
[ERROR]   KNW-PLT-MOD-001  KL-001  knowledge_id format invalid: missing DOMAIN segment
[ERROR]   KNW-PLT-MOD-002  KL-005  duplicate name "Quota Manager" already at KNW-PLT-MOD-001
[WARNING] KNW-PLT-MOD-003  KL-015  low confidence (0.40); consider adding evidence
[INFO]    KNW-PLT-MOD-004  KL-028  tags could be more specific (only 1 tag)

Summary: 2 errors, 1 warning, 1 info  (4 objects scanned)
```

---

## Linter Rules

### Category: Identity (KL-001–KL-006)

| Rule | Severity | Description | Auto-fix |
|------|----------|-------------|---------|
| KL-001 | ERROR | `knowledge_id` does not match `^KNW-[A-Z]...-[A-Za-z0-9_-]+$` | No |
| KL-002 | ERROR | `knowledge_id` domain code does not match declaring package | No |
| KL-003 | ERROR | Duplicate `knowledge_id` found in repository | No |
| KL-004 | ERROR | `canonical_name` does not match computed value | Yes |
| KL-005 | ERROR | Duplicate `name` within the same namespace | No |
| KL-006 | ERROR | `knowledge_uri` does not match computed value | Yes |

---

### Category: Required Fields (KL-007–KL-014)

| Rule | Severity | Description | Auto-fix |
|------|----------|-------------|---------|
| KL-007 | ERROR | `name` is empty or whitespace-only | No |
| KL-008 | ERROR | `description` is shorter than 20 characters | No |
| KL-009 | ERROR | `owner` is empty or not in `team:*` / `user:*` / `system:*` format | No |
| KL-010 | ERROR | `version` does not match semver `MAJOR.MINOR.PATCH` | No |
| KL-011 | ERROR | `status` is not a valid KnowledgeLifecycleStatus value | No |
| KL-012 | ERROR | `tags` list is empty | No |
| KL-013 | ERROR | Type-specific required field is missing or empty | No |
| KL-014 | ERROR | `object_type` does not match the template type | No |

---

### Category: References (KL-015–KL-020)

| Rule | Severity | Description | Auto-fix |
|------|----------|-------------|---------|
| KL-015 | ERROR | Relationship references `source_id` that does not exist in repository | No |
| KL-016 | ERROR | Relationship references `target_id` that does not exist in repository | No |
| KL-017 | ERROR | `traceability.satisfies` references non-existent knowledge_id | No |
| KL-018 | ERROR | `traceability.implements` references non-existent knowledge_id | No |
| KL-019 | WARNING | `traceability.tests` is empty for a non-DRAFT MODULE or SERVICE | No |
| KL-020 | WARNING | Evidence `source_url` is unreachable (checked only in CI, not local lint) | No |

---

### Category: Lifecycle (KL-021–KL-026)

| Rule | Severity | Description | Auto-fix |
|------|----------|-------------|---------|
| KL-021 | ERROR | CANONICAL object has `approved_by` empty | No |
| KL-022 | ERROR | VERIFIED+ object has zero evidence records | No |
| KL-023 | ERROR | CANONICAL object has `quality.overall_score` < 0.80 | No |
| KL-024 | ERROR | DEPRECATED object has no `lifecycle.successor_id` | No |
| KL-025 | WARNING | REVIEW object has been in REVIEW state > 30 days | No |
| KL-026 | ERROR | Relationship `status` is ACTIVE but source or target is ARCHIVED | No |

---

### Category: Style (KL-027–KL-032)

| Rule | Severity | Description | Auto-fix |
|------|----------|-------------|---------|
| KL-027 | WARNING | `name` is not Title Case (SG-N-01) | Yes |
| KL-028 | WARNING | `tags` list has fewer than 2 entries (SG-T-01) | No |
| KL-029 | WARNING | `tags` first entry does not match package namespace (SG-T-03) | Yes |
| KL-030 | WARNING | `description` contains "TODO", "TBD", or "placeholder" (SG-D-05) | No |
| KL-031 | INFO | `description` is longer than 500 characters (SG-D-03) | No |
| KL-032 | WARNING | Tag name not in canonical tag list (SG-T-06) | No |

---

## Linter Rule File Format

Each rule is defined as a YAML file in `lint/rules/`:

```yaml
# lint/rules/KL-001.yaml
id: KL-001
category: identity
severity: ERROR
title: "Invalid knowledge_id format"
description: |
  The knowledge_id field must match the canonical regex:
  ^KNW-[A-Z][A-Z0-9]*-[A-Z][A-Z0-9]*-[A-Za-z0-9_-]+$
check_field: knowledge_id
check_type: regex
regex: "^KNW-[A-Z][A-Z0-9]*-[A-Z][A-Z0-9]*-[A-Za-z0-9_-]+$"
auto_fix: false
references: ["05-KNOWLEDGE-NAMING-STANDARD"]
```

---

## Linter Configuration

```yaml
# lint/linter-config.yaml
version: "1.0.0"
disabled_rules: []          # override in team configs; empty by default
severity_overrides: {}      # e.g., {KL-028: INFO}
scan_paths:
  - knowledge/packages/
  - knowledge/reference/
exclude_paths:
  - knowledge/packages/*/examples/  # examples may be illustrative, not complete
check_external_refs: false  # set true in CI only
```

---

## Integration with CI

From `08-KNOWLEDGE-CI`:
- Pre-commit: run `kos lint --severity error` on staged files
- PR check: run full `kos lint` on all changed packages
- Main merge gate: `kos lint` exit code must be 0

---

## Cross-References

- Style rules being checked → `04-KNOWLEDGE-STYLE-GUIDE`
- Naming rules being checked → `05-KNOWLEDGE-NAMING-STANDARD`
- CI integration → `08-KNOWLEDGE-CI`
- Auto-fix is the formatter → `07-KNOWLEDGE-FORMATTER`
