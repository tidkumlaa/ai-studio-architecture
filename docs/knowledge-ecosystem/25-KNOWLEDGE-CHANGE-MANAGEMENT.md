# KNW-KE-ARCH-025 — Knowledge Change Management

**Phase:** 3.0C.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Governs how changes to Knowledge Objects, packages, and ecosystem infrastructure are proposed, reviewed, and applied. Prevents uncontrolled drift.

---

## Change Classification

| Class | Definition | Examples |
|-------|------------|---------|
| PATCH | Non-breaking: fix typo, add tag, update evidence | Fix description, add evidence record, update freshness |
| MINOR | Non-breaking addition: new optional field, new object | Add new module, add optional metadata field |
| MAJOR | Breaking: rename, remove, change required field, change ID | Rename canonical_name, change knowledge_id format |
| EMERGENCY | Critical production fix, bypasses normal process | Security fix requiring immediate CANONICAL change |

---

## Change Process

### PATCH changes

```
1. Author edits YAML file
2. Run kos format + kos lint
3. Open PR with title "patch(package): description"
4. 1 reviewer approval
5. CI passes
6. Merge
```

### MINOR changes

```
1. Author edits/creates YAML file(s)
2. Run kos format + kos lint + kos verify
3. Open PR with title "feat(package): description"
4. 1 reviewer approval
5. CI passes (quality gate ≥ 0.70)
6. Merge
7. Bump package version MINOR
```

### MAJOR changes

```
1. Author creates ADR (Architecture Decision Record) — KNW-ARCH-ADR-NNN
2. ADR is reviewed and approved
3. Migration guide written (see Migration Guide format below)
4. Open PR with title "MAJOR(package): description"
5. Architecture Board sign-off
6. CI passes (quality gate ≥ 0.80 for all changed objects)
7. Merge
8. Bump package version MAJOR
9. Publish migration guide to documentation
```

### EMERGENCY changes

From Phase 3.0C `33-LIFECYCLE-EXTENSIONS` bypass policy:
```
Authority: Architecture Board (unanimous vote)
Conditions: Production incident OR legal/compliance deadline
Requirements:
  - Bypass reason documented in object
  - Post-incident evidence filed within 7 days
  - Bypass audit log entry created
Max duration: 90 days — reach target state or revert
```

---

## Deprecation Process

When a Knowledge Object becomes obsolete:

```
Step 1: Create successor object (if applicable)
Step 2: Add DEPRECATION_NOTICE to original object:
  lifecycle.deprecation_reason: "Replaced by KNW-PLT-MOD-002"
  lifecycle.successor_id: KNW-PLT-MOD-002
  lifecycle.deprecated_at: "2026-06-30"
  lifecycle.sunset_at: "2026-09-30"   # deprecated_at + 90 days

Step 3: Run: kos lifecycle deprecate KNW-PLT-MOD-001 --successor KNW-PLT-MOD-002
Step 4: Notify all dependent packages (kos notify --deprecated KNW-PLT-MOD-001)
Step 5: After sunset_at: kos lifecycle archive KNW-PLT-MOD-001
```

---

## Renaming Process

Renaming is a MAJOR change. Renames must:

```
1. Create ADR for the rename
2. Assign new knowledge_id (old ID never reused)
3. Add alias: old ID → new ID in registry/aliases.yaml
4. Update all references in other objects (kos refactor --rename)
5. Deprecate old object with new as successor
6. Keep alias active for ≥ 90 days
```

---

## Migration Guide Format

```yaml
# knowledge/migration/migration-{from_version}-to-{to_version}.yaml
from_version: "1.0.0"
to_version: "2.0.0"
package: kos.platform.package
breaking_changes:
  - field: module_status
    old_values: [EXISTING, PLANNED]
    new_values: [ACTIVE, PLANNED]
    migration: |
      Replace EXISTING with ACTIVE in all objects.
      Command: kos migrate --rule module-status-rename

migrations:
  - id: M-001
    description: "Rename module_status EXISTING → ACTIVE"
    type: field_value_rename
    target_field: module_status
    old_value: EXISTING
    new_value: ACTIVE
    reversible: true
    command: "kos migrate --rule module-status-rename"
```

---

## Change Log Format

```yaml
# knowledge/packages/{name}/docs/CHANGELOG.md (YAML front matter)
---
package: kos.platform.package
---

# Changelog

## [2.0.0] — 2026-09-30
### Breaking Changes
- `module_status` renamed: `EXISTING` → `ACTIVE`
  See migration guide: migration/migration-1.0.0-to-2.0.0.yaml

## [1.1.0] — 2026-07-30
### Added
- `KNW-PLT-MOD-010`: New Analytics Engine module

## [1.0.0] — 2026-06-30
### Initial Release
```

---

## Change Notification

When a MAJOR or MINOR change affects objects used by other packages:

```bash
kos notify --changed KNW-PLT-MOD-001 \
           --message "module_status field renamed" \
           --affected-packages kos.runtime.package,kos.test.package
```

Emits event: `knowledge.object.status_changed` to all registered consumers.

---

## Cross-References

- Lifecycle transitions → `19-KNOWLEDGE-LIFECYCLE`
- ADR format → `11-KNOWLEDGE-EXAMPLES` (DECISION example)
- Package versioning → `02-KNOWLEDGE-PACKAGES`
- CI enforces change process → `08-KNOWLEDGE-CI`
- Emergency bypass → Phase 3.0C `33-LIFECYCLE-EXTENSIONS`
