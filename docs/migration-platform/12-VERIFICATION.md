---
knowledge_id: KA-PLAT-013
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
  - id: KA-PLAT-009
    reason: "Verification consumes all indexes built by IndexBuilder"
  - id: KA-PLAT-010
    reason: "Verification consumes HealthReport"
  - id: KA-PLAT-008
    reason: "Verification consumes CanonicalMap"
---

# Verification

## Migration Reports and Acceptance Criteria

---

## 1. Purpose

Verification is the final pipeline stage. It produces the report suite that
confirms whether the migration succeeded and whether the repository health
phase gate (≥ 80/100) has been met.

---

## 2. Report Suite

| Report | File | Audience |
|--------|------|---------|
| Migration Report | `MIGRATION-REPORT.md` | Chief Architect |
| Knowledge Inventory | `reports/inventory.yaml` | All engineers |
| Health Report | `HEALTH-REPORT.md` | All engineers |
| Canonical Report | `reports/canonical-report.md` | Knowledge Architect |
| Relationship Report | `reports/relationship-report.md` | Architects |
| Coverage Report | `reports/coverage-report.md` | Tech leads |
| Audit Report | `reports/audit-report.yaml` | CI/CD |

---

## 3. Migration Report

The Migration Report summarizes what the engine did and whether it succeeded:

```markdown
# Migration Report — Phase 2.0D.1

**Date:** 2026-06-29
**Repository:** /path/to/repo
**Engine:** KnowledgeMigrationEngine 1.0.0

## Summary

| Metric | Before | After |
|--------|--------|-------|
| Total documents | 786 | 823 |
| With frontmatter | 12 | 823 |
| Oversized (>500 lines) | 23 | 0 |
| Broken links | 47 | 3 |
| Orphan documents | 31 | 8 |
| Health score | 15.2 | 82.1 |

## Phase Gate

Health score: **82.1 / 100** ✓ PASS (required: 80)
Phase 2.0D.2 is UNLOCKED.

## Phases Completed

| Phase | Status | Duration | Issues |
|-------|--------|---------|--------|
| A: Foundation | ✓ | 2m | None |
| B: Classification | ✓ | 8m | 14 low-confidence |
| C: Archive | ✓ | 3m | None |
| D: Split | ✓ | 24m | 2 manual required |
| E: Metadata | ✓ | 47m | 3 failed |
| F: Index | ✓ | 6m | None |

## Failures Requiring Manual Resolution
- docs/legacy/MASTER.md: Section boundaries ambiguous — manual split required
- docs/old-api/catalog.md: No detectable H2 headings — cannot split
- docs/unknown/X.md: Classification confidence 0.21 (< 0.40) — manual type required
```

---

## 4. Knowledge Inventory

Machine-readable inventory for CI and downstream tools:

```yaml
# reports/inventory.yaml
generated: "2026-06-29"
repository: "/path/to/repo"
summary:
  total_documents: 823
  by_type:
    architecture: 42
    specification: 87
    decision: 23
    api: 18
    guide: 31
    other: 12
  by_status:
    approved: 156
    draft: 589
    deprecated: 61
    archived: 17
  by_domain:
    DOM-WORKFLOW: 94
    DOM-AI: 87
    # ...
  by_health:
    excellent (85-100): 23
    good (70-84): 87
    fair (50-69): 156
    poor (25-49): 412
    critical (0-24): 145
```

---

## 5. Acceptance Criteria

Migration is accepted when ALL of the following pass:

| Criterion | Pass Condition |
|-----------|--------------|
| Health gate | Repository health ≥ 80/100 |
| No broken indexes | All index.yaml files are valid YAML |
| Registry complete | KNOWLEDGE-REGISTRY.yaml has entry for every approved document |
| No duplicate canonicals | Zero RULE-C001 violations |
| No oversized documents | Zero documents > 500 lines |
| Metadata coverage | ≥ 95% of documents have valid frontmatter |
| Broken links | Broken link count < 5% of total links |

---

## 6. Audit Report

The audit report is a YAML file consumed by CI:

```yaml
# reports/audit-report.yaml
run_date: "2026-06-29"
rules_run: 34
violations:
  - rule_id: RULE-M001
    severity: error
    count: 3
    documents:
      - "docs/legacy/old-doc.md"
warnings:
  - rule_id: RULE-SZ003
    severity: warning
    count: 8
passed:
  count: 24
exit_code: 1  # 0 if no errors
```

---

## 7. CLI Commands

```bash
# Generate all reports
python migrate.py report /path/to/repo --output reports/

# Verify phase gate only
python migrate.py verify /path/to/repo

# Generate single report
python migrate.py report /path/to/repo --type health --output HEALTH-REPORT.md
python migrate.py report /path/to/repo --type canonical --output reports/
python migrate.py report /path/to/repo --type inventory --output reports/
```

---

## References

- [01-MIGRATION-ENGINE-ARCHITECTURE.md](01-MIGRATION-ENGINE-ARCHITECTURE.md) — Pipeline context
- [KA-STD-006](../knowledge-architecture/KNOWLEDGE-HEALTH-METRICS.md) — Health score definition
- [KA-STD-008](../knowledge-architecture/REPOSITORY-AUDIT-RULES.md) — 34 audit rules
- [KA-VIS-001](../knowledge-architecture/KNOWLEDGE-ARCHITECTURE-VISION.md) — Phase gate requirement
