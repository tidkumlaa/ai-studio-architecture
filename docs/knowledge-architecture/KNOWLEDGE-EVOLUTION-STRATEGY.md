---
knowledge_id: KA-STD-007
version: "1.0.0"
status: approved
owner: Chief Knowledge Architect
phase: 2.0D.0
created: 2026-06-29
review_date: 2026-12-29
canonical: true
implements:
  - KA-VISION-001
---

# Knowledge Evolution Strategy

## Versioning, Deprecation, Migration, and the Architecture Timeline

---

## 1. Knowledge Changes Over Time

Architecture is not written once and frozen. It evolves:
- Decisions are reversed by new ADRs
- Capabilities are redesigned or replaced
- Products are deprecated or merged
- Implementation reveals gaps in design
- New requirements invalidate old assumptions

The knowledge system must accommodate this evolution without losing history. Every change must be:

1. **Traceable** — what changed, when, why, who decided
2. **Reversible** — old knowledge is preserved in archive, never deleted
3. **Discoverable** — the current state is always findable; the history is always accessible
4. **Navigable** — supersession chains are traversable

---

## 2. Versioning Strategy

### 2.1 Document-Level Versioning

Every knowledge document uses semantic versioning.

```
MAJOR.MINOR.PATCH

MAJOR: The concept described has fundamentally changed
       - A new architecture replaces the old one
       - A contract breaking change was made
       - The scope of the document significantly changed

MINOR: New information was added without breaking existing understanding
       - New sections added
       - New relationships declared
       - Roadmap updated

PATCH: Corrections that do not change the information
       - Typo fixes
       - Broken link repairs
       - Metadata corrections
```

**Version bump rules:**

| Change Type | Version Bump | Example |
|-------------|-------------|---------|
| New architectural pattern | MAJOR | 1.2.3 → 2.0.0 |
| New capability section added | MINOR | 1.2.3 → 1.3.0 |
| New ADR reference added | MINOR | 1.2.3 → 1.3.0 |
| New relationship declared | MINOR | 1.2.3 → 1.3.0 |
| Typo corrected | PATCH | 1.2.3 → 1.2.4 |
| Link fixed | PATCH | 1.2.3 → 1.2.4 |
| review_date updated | PATCH | 1.2.3 → 1.2.4 |

### 2.2 Capability-Level Versioning

Each capability's `index.yaml` declares a capability version distinct from any individual document version.

```yaml
version: "2.0.0"   # Capability version
```

The capability version follows the same semver rules but reflects the maturity of the **capability as a whole**.

| Phase | Version |
|-------|---------|
| First documented | 0.1.0 |
| First approved design | 1.0.0 |
| Major redesign | 2.0.0 |
| Stable, fully covered | 2.x.x |

### 2.3 Repository-Level Versioning

The repository has an epoch version tracked in `architecture/index.yaml`:

```yaml
meta:
  epoch: "2.0D.0"      # Phase that last made structural changes
  schema_version: "1.0.0"  # Knowledge schema version
```

When the knowledge architecture schema itself changes (e.g., new required metadata fields), the `schema_version` increments.

---

## 3. Deprecation Strategy

### 3.1 Deprecation Triggers

A knowledge document is deprecated when:

| Trigger | Description |
|---------|-------------|
| **Superseded** | A newer document replaces this one |
| **Invalidated** | An ADR decision invalidates the design described |
| **Merged** | Content was consolidated into another document |
| **Capability Retired** | The capability this document describes was retired |
| **Product Retired** | The product this document covers is no longer active |
| **Standard Replaced** | This standard was replaced by a newer standard |

### 3.2 Deprecation Process

**Step 1 — Declare Deprecation**

Update frontmatter:
```yaml
status: deprecated
deprecated_date: 2026-06-29
superseded_by: KA-ARCH-042    # ID of replacement, if exists
```

**Step 2 — Add Deprecation Banner**

Add to the top of the document body:

```markdown
> **DEPRECATED** — This document was deprecated on 2026-06-29.
> It has been superseded by [Architecture v2.0](../architecture.md) [KA-ARCH-042].
> This document is preserved for historical reference only.
> Do not use for implementation decisions.
```

**Step 3 — Update All Incoming References**

Find all documents that link to this document:
```
grep -r "KA-ARCH-001" architecture/ --include="*.md" --include="*.yaml"
```

Update each reference to point to the canonical successor, or remove the reference.

**Step 4 — Update Indexes**

The audit tool regenerates all indexes. The deprecated document appears in indexes with a `(deprecated)` marker. It is removed from active navigation but remains accessible via archive navigation.

**Step 5 — Start the 90-Day Clock**

After 90 days in `deprecated` state, the document is eligible for archival.

### 3.3 Deprecation Exceptions

| Case | Rule |
|------|------|
| ADR | ADRs are never deprecated. They are superseded by newer ADRs. The supersession chain preserves history. |
| History documents | History documents are never deprecated. They are append-only records. |
| Archive documents | Already archived — not subject to deprecation. |

---

## 4. Migration Strategy

### 4.1 Document Migration

When a document is split into multiple smaller documents:

**Before:**
```
capabilities/workflow-runtime/LARGE-DOCUMENT.md    (2,000 lines)
```

**After:**
```
capabilities/workflow-runtime/architecture.md      (350 lines)
capabilities/workflow-runtime/api.md               (280 lines)
capabilities/workflow-runtime/database.md          (200 lines)
capabilities/workflow-runtime/events.md            (180 lines)
```

Migration steps:
1. Create target documents with content extracted from the large document
2. Assign knowledge IDs to each new document
3. Deprecate the large document with `superseded_by` pointing to the primary successor
4. Add a migration note in the deprecated document listing all successors
5. Update all inbound references to point to the appropriate successor

**Migration deprecation note:**
```markdown
> **DEPRECATED — SPLIT** — This document was split into multiple focused documents on 2026-06-29:
>
> - Architecture → [architecture.md](architecture.md) [KA-ARCH-001]
> - API → [api.md](api.md) [KA-API-001]
> - Database → [database.md](database.md) [KA-DB-001]
> - Events → [events.md](events.md) [KA-EVT-001]
```

### 4.2 Capability Migration

When a capability is refactored, renamed, or split:

**Case: Rename**
```yaml
# In new folder's history.md:
history:
  - date: 2026-06-29
    event: renamed
    from: "ai-provider-manager"
    to: "provider-management"
    reason: "Aligned with platform naming conventions"
    decided_by: KA-ADR-028
```

**Case: Split**
```yaml
# In each new capability's history.md:
history:
  - date: 2026-06-29
    event: split_from
    source: "provider-framework"
    reason: "Provider management concerns separated from provider contract concerns"
    decided_by: KA-ADR-029
```

**Case: Merge**
```yaml
# In the surviving capability's history.md:
history:
  - date: 2026-06-29
    event: merged_from
    sources:
      - "ai-account-manager"
      - "ai-quota-engine"
    reason: "Budget and quota management are the same concern"
    decided_by: KA-ADR-030
```

---

## 5. Refactoring Strategy

### 5.1 When to Refactor

Knowledge refactoring is warranted when:

| Signal | Threshold | Action |
|--------|-----------|--------|
| Document health score | < 60 for 60+ days | Refactor document |
| Capability health score | < 70 for 30+ days | Refactor capability |
| Stale document accumulation | > 20% of documents stale | Capability-wide review sprint |
| Duplicate canonical conflicts | Any | Immediate resolution required |
| Broken reference count | > 5 | Broken reference sprint |
| Repository health score | < 70 | Repository-wide refactoring sprint |

### 5.2 Refactoring Scope Levels

| Level | Scope | Triggers | Effort |
|-------|-------|---------|--------|
| Document | Single document | Document health < 60 | Hours |
| Capability | One capability folder | Capability health < 70 | Days |
| Domain | All capabilities in domain | Domain health < 65 | Weeks |
| Repository | All documents | Repository health < 70 | Months |

### 5.3 Refactoring Protocol

1. **Snapshot** — Record the current health score baseline
2. **Audit** — Run full audit to identify all violations
3. **Triage** — Prioritize: Critical → High → Medium
4. **Resolve Critical** — Duplicate canonicals, broken references, missing required files
5. **Resolve High** — Stale documents, missing metadata, missing relationships
6. **Resolve Medium** — Incomplete coverage, low confidence docs, weak evidence
7. **Verify** — Re-run audit; confirm score improvement
8. **Document** — Record refactoring in `history.md`

---

## 6. Historical Snapshots

### 6.1 What Is Preserved

Historical knowledge is preserved at version boundaries.

Every time the architecture enters a new major phase, a historical snapshot is taken:
```
archive/
├── v1.5.5/     ← Snapshot of all active documents at Phase 1.5.5 completion
├── v2.0/       ← Snapshot at Phase 2.0 completion
├── v2.0D.0/    ← Snapshot at Phase 2.0D.0 completion
```

Historical snapshots are read-only. They are the authoritative record of what was known at a given phase.

### 6.2 Snapshot Process

At phase completion:
1. Run full audit to get health score
2. Copy all `approved` documents to `archive/v[phase]/`
3. Generate `archive/v[phase]/index.yaml` with full document inventory
4. Record snapshot in `history/ARCHITECTURE-TIMELINE.md`
5. Tag the git commit with the phase version

### 6.3 Snapshot Accessibility

Historical snapshots are accessible but not indexed in the active knowledge system:
- Active `KNOWLEDGE-INDEX.md` does not include archive documents
- Archive documents have their own `archive/index.yaml`
- Historical queries (e.g., "what was the architecture at v1.5.5?") use the archive index

---

## 7. Architecture Timeline

The architecture timeline is the permanent, append-only record of all major knowledge events.

**Location:** `history/ARCHITECTURE-TIMELINE.md`

**Structure:**

```markdown
# Architecture Timeline

## 2026-06-29 — Phase 2.0D.0: Knowledge Architecture Foundation

**Event:** Knowledge Architecture Phase initiated
**Scope:** Repository-wide structural redesign
**ADRs:** [KA-ADR-040](../adr/KA-ADR-040.md) — Knowledge Architecture Approach
**Health Before:** 23/100
**Deliverables:**
- KNOWLEDGE-ARCHITECTURE-VISION.md
- MICRO-KNOWLEDGE-ARCHITECTURE.md
- [... all Phase 2.0D.0 documents]

---

## 2025-12-15 — Phase 2.0C.0: Repository Separation

**Event:** Architecture repository separated from platform repository
**Scope:** Repository structure
**ADRs:** [KA-ADR-035](../adr/KA-ADR-035.md)

---

## 2025-01-01 — Phase 1.0: Initial Architecture

**Event:** First architecture documents created
...
```

---

## 8. Decision History

Architecture decisions have their own evolution model in the ADR system.

### 8.1 ADR Lifecycle

```
proposed → accepted
         ↓
      deprecated → superseded
```

ADRs are **immutable once accepted**. They are never edited. If a decision is reversed:
1. A new ADR is created documenting the new decision
2. The new ADR declares `supersedes: KA-ADR-[old]`
3. The old ADR is updated to `status: superseded` and `superseded_by: KA-ADR-[new]`
4. The old ADR content is never modified

### 8.2 Decision Chain Navigation

```
KA-ADR-008 (superseded)
    ↓ superseded_by
KA-ADR-015 (superseded)
    ↓ superseded_by
KA-ADR-028 (accepted — current)
```

The current decision is always the terminal node. Prior decisions are preserved in the chain.

---

## 9. Knowledge Lifecycle Summary

```
            [Created as draft]
                    ↓
            [Review period]
                    ↓
            [Approved — Active]
             /           \
    [Updated → same ID]  [Superseded → Deprecated]
    [Version bump]               ↓
                         [90-day hold period]
                                 ↓
                           [Archived]
                                 ↓
                         [Snapshot preservation]
                           (permanent read-only)
```

---

## References

- [KNOWLEDGE-ARCHITECTURE-VISION.md](KNOWLEDGE-ARCHITECTURE-VISION.md) — Lifecycle as a design principle
- [METADATA-STANDARD.md](METADATA-STANDARD.md) — Version and lifecycle fields
- [KNOWLEDGE-HEALTH-METRICS.md](KNOWLEDGE-HEALTH-METRICS.md) — Health triggers for refactoring
- [REPOSITORY-REFACTORING-PLAN.md](REPOSITORY-REFACTORING-PLAN.md) — Concrete migration plan
- [CANONICAL-SOURCE-STRATEGY.md](CANONICAL-SOURCE-STRATEGY.md) — Deprecation and supersession
