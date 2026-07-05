---
knowledge_id: KA-ROAD-001
version: "1.0.0"
status: approved
owner: Chief Knowledge Architect
phase: 2.0D.0
created: 2026-06-29
review_date: 2026-09-29
canonical: true
implements:
  - KA-VISION-001
---

# Repository Refactoring Plan

## Phase 2.0D.1 — From Document Repository to Knowledge Repository

---

## 1. Current State Assessment

### 1.1 Repository Inventory

| Item | Count | Notes |
|------|-------|-------|
| Total markdown files | 786 | Across all subdirectories |
| Architecture docs (docs/) | ~90 | Primary refactoring target |
| Historical docs (1.5.5/) | 49 | Mostly reportage — archive candidates |
| Phase-1a capability docs | 17 | Need restructuring to new standard |
| ADRs | Unknown | Need audit and registration |
| Decisions | Unknown | Need audit and registration |

### 1.2 Known Problems

| Problem | Severity | Estimated Count |
|---------|----------|----------------|
| Documents with no metadata | Critical | ~786 |
| Documents with no knowledge ID | Critical | ~786 |
| Documents over 500 lines | High | 10–20 |
| Duplicate canonical concepts | High | Unknown |
| Broken internal links | High | Unknown |
| Orphaned documents | Medium | 50+ (historical) |
| Missing required files per standard | High | 100% of capabilities |
| No capability folder structure | High | 100% |

### 1.3 Refactoring Scope Statement

**In scope for Phase 2.0D.1:**
- All documents under `architecture/docs/`
- All documents under `architecture/adr/`
- All documents under `architecture/decisions/`
- Creating the new capability folder structure
- Applying metadata to all documents
- Registering all knowledge IDs
- Building all indexes

**Out of scope for Phase 2.0D.1:**
- Documents under `platform/` (platform code repository)
- Documents under `archive/` (already historical)
- Creating new architecture content (only refactoring existing content)

---

## 2. Migration Strategy

### 2.1 Core Migration Principle

**No content is deleted. Every document either:**
1. Becomes a canonical knowledge object in the new structure, or
2. Is deprecated and archived with a reference to its successor

### 2.2 Migration Classification

Every existing document is classified into one of these categories:

| Category | Action | Outcome |
|----------|--------|---------|
| **Active Canonical** | Restructure in place | Moves to capability folder, gets metadata |
| **Active Non-Canonical** | Convert to reference | Content stripped, reference link added |
| **Historical Report** | Archive | Moved to `archive/v[phase]/`, read-only |
| **Superseded Architecture** | Deprecate + Archive | Deprecated in place, moved to archive |
| **Orphaned** | Triage | Either classify as active or archive |
| **Duplicate** | Merge or deprecate | One becomes canonical, others deprecated |

---

## 3. Rollout Phases

### Phase 2.0D.1-A — Foundation (Weeks 1–2)

**Goal:** Create the structural skeleton without touching existing documents.

**Tasks:**
1. Create `architecture/capabilities/` directory
2. Create all 10 capability folders with stub files
3. Create `archive/` structure
4. Create `platform/` structure
5. Create `products/` structure
6. Create `history/` structure
7. Register all knowledge IDs for new stub documents
8. Create `KNOWLEDGE-REGISTRY.yaml` (initially containing stubs only)
9. Create all index files (initially empty)
10. Validate: No existing files were modified

**Success criteria:**
- New folder structure exists
- All required stub files are in place
- No existing documents were changed
- `KNOWLEDGE-REGISTRY.yaml` exists

**Estimated effort:** 3–5 days

---

### Phase 2.0D.1-B — Document Classification (Weeks 2–3)

**Goal:** Classify every existing document without moving any files.

**Tasks:**
1. Audit all documents in `docs/` — assign classification
2. Audit all documents in `docs/1.5.5/` — classify as archive candidates
3. Audit all documents in `docs/phase-1a/` — classify as capability docs
4. Audit all documents in `docs/2.0/`, `docs/3.0/`, `docs/3.5/` — classify as historical
5. Audit `adr/` — register in KNOWLEDGE-REGISTRY.yaml
6. Generate Classification Report

**Classification Report format:**
```yaml
classification_report:
  generated: 2026-07-15
  total_documents: 786

  active_canonical:
    count: 17
    documents:
      - file: docs/phase-1a/AI-CONVERSATION-ENGINE.md
        target_capability: conversation-intelligence
        target_type: architecture
        estimated_split: false

  active_non_canonical:
    count: 12
    documents:
      - file: docs/API-CATALOG.md
        reason: "Content distributed across capability api.md files"
        target: references.md in each capability

  historical_report:
    count: 49
    documents:
      - file: docs/1.5.5/CHAOS-TEST-REPORT.md
        archive_path: archive/v1.5.5/

  superseded_architecture:
    count: 8
    documents:
      - file: docs/2.0/AI-STUDIO-2.0-ARCHITECTURE.md
        superseded_by: docs/AI-STUDIO-MASTER-ARCHITECTURE.md

  orphaned:
    count: 23
    documents:
      - file: docs/PHASE-0C-MIGRATION-REPORT.md
        triage: archive
```

**Estimated effort:** 5–7 days

---

### Phase 2.0D.1-C — Historical Archive (Week 3)

**Goal:** Move all historical/report documents to `archive/`.

**Target documents:**
- All `docs/1.5.5/*.md` → `archive/v1.5.5/`
- All `docs/2.0/*.md` → `archive/v2.0/`
- All `docs/3.0/*.md` → `archive/v3.0/`
- All `docs/3.5/*.md` → `archive/v3.5/`
- All phase reports and one-time reports → `archive/reports/`

**Tasks:**
1. Create archive sub-directories
2. Move historical documents (git mv, preserving history)
3. Create `archive/index.yaml` with full inventory
4. Add deprecation banners to moved documents
5. Validate: archived documents have no broken incoming references

**Rollback:** `git revert` — all moves are in git history

**Estimated effort:** 2–3 days

---

### Phase 2.0D.1-D — Capability Document Migration (Weeks 3–5)

**Goal:** Extract capability knowledge from existing documents into the new capability folders.

**Document migration queue (priority order):**

| Source Document | Target Capability | Split? | Effort |
|----------------|------------------|--------|--------|
| `phase-1a/AI-CONVERSATION-ENGINE.md` | conversation-intelligence | No | M |
| `phase-1a/AI-PROVIDER-CONTRACTS.md` | provider-framework | No | M |
| `phase-1a/AI-RESOURCE-OS-BLUEPRINT.md` | ai-runtime-os | Yes | L |
| `phase-1a/AI-SCHEDULER.md` | workflow-runtime | No | M |
| `phase-1a/AI-ACCOUNT-MANAGER.md` | provider-management | No | M |
| `phase-1a/AI-BUDGET-ENGINE.md` | provider-management | Merge | S |
| `phase-1a/AI-QUOTA-ENGINE.md` | provider-management | Merge | S |
| `phase-1a/AI-POLICY-ENGINE.md` | provider-framework | No | M |
| `phase-1a/AI-MODEL-CATALOG.md` | provider-management | No | M |
| `phase-1a/AI-BENCHMARK-CENTER.md` | products/benchmark | No | M |
| `docs/AI-STUDIO-MASTER-ARCHITECTURE.md` | Multiple | Yes — major split | XL |
| `docs/API-CATALOG.md` | Multiple | Yes — distribute | L |
| `docs/DATABASE-CATALOG.md` | Multiple | Yes — distribute | L |
| `docs/EVENT-CATALOG.md` | Multiple | Yes — distribute | L |
| `docs/PLATFORM-STANDARDS.md` | platform/standards | No | S |
| `docs/SECURITY-MODEL.md` | platform/security | No | M |
| `docs/PLATFORM-CONTRACTS.md` | platform/contracts | No | M |

**Effort sizing:** S=1 day, M=2 days, L=3 days, XL=5+ days

**Migration process per document:**

```
1. Read source document
2. Identify section → capability → type mapping
3. Extract section content to target file
4. Add metadata frontmatter to target file
5. Assign knowledge ID
6. Register in KNOWLEDGE-REGISTRY.yaml
7. Add reference link in source document
8. Run audit to check for broken references
```

**Estimated effort:** 3–4 weeks

---

### Phase 2.0D.1-E — Metadata Application (Week 5–6)

**Goal:** Apply complete frontmatter metadata to all active documents.

**Tasks:**
1. For every document in capability folders, complete the full YAML frontmatter
2. Declare all relationships (`depends_on`, `implements`, `implemented_by`)
3. Set `confidence` and `coverage` values
4. Set `evidence` entries where implementation exists
5. Set `review_date` per document type schedule
6. Validate: audit tool reports 0 metadata errors

**Estimated effort:** 1–2 weeks

---

### Phase 2.0D.1-F — Index Generation (Week 6)

**Goal:** Generate all indexes and validate the complete knowledge graph.

**Tasks:**
1. Run `tools/generate-indexes.py` to generate all indexes
2. Validate `KNOWLEDGE-REGISTRY.yaml` has no collisions
3. Validate `relationship-index.yaml` has no dangling references
4. Validate `CANONICAL-SOURCE-MAP.yaml` has no conflicts
5. Run full health report
6. Achieve repository health score ≥ 80

**Success criteria:**
- All indexes generated without errors
- 0 broken references
- 0 duplicate canonicals
- Repository health score ≥ 80
- Phase gate criteria met (see [KNOWLEDGE-HEALTH-METRICS.md](KNOWLEDGE-HEALTH-METRICS.md))

**Estimated effort:** 3–5 days

---

## 4. Document Split Plan

The following documents require splitting (over 500 lines or multi-topic).

### 4.1 AI-STUDIO-MASTER-ARCHITECTURE.md (8,008 lines)

The largest refactoring target. Contains multi-domain content that must be distributed.

**Split mapping:**

| Section(s) | Target | Knowledge Type |
|-----------|--------|---------------|
| Executive Summary | archive — superseded by README | N/A |
| AI Runtime OS | capabilities/ai-runtime-os/architecture.md | KA-ARCH |
| Prompt OS | capabilities/prompt-os/architecture.md | KA-ARCH |
| Workflow Runtime | capabilities/workflow-runtime/architecture.md | KA-ARCH |
| Provider Framework | capabilities/provider-framework/architecture.md | KA-ARCH |
| Conversation Intelligence | capabilities/conversation-intelligence/architecture.md | KA-ARCH |
| Context Intelligence | capabilities/context-intelligence/architecture.md | KA-ARCH |
| Intelligent Routing | capabilities/intelligent-routing/architecture.md | KA-ARCH |
| Platform Security | platform/security/architecture.md | KA-SEC |
| Platform Standards | platform/standards/overview.md | KA-OVW |
| Database Catalog | Each capability's database.md | KA-DB |
| API Catalog | Each capability's api.md | KA-API |
| Event Catalog | Each capability's events.md | KA-EVT |

**Post-split:** Original document deprecated with migration note pointing to all successors.

### 4.2 AI-RESOURCE-OS-BLUEPRINT.md

**Split mapping:**

| Section | Target |
|---------|--------|
| OS Architecture | capabilities/ai-runtime-os/architecture.md |
| Desktop Architecture | capabilities/ai-runtime-os/desktop.md |
| API Surface | capabilities/ai-runtime-os/api.md |
| Database Schema | capabilities/ai-runtime-os/database.md |
| Security Model | capabilities/ai-runtime-os/security.md |

### 4.3 API-CATALOG.md, DATABASE-CATALOG.md, EVENT-CATALOG.md

These cross-capability catalogs are distributed to individual capability folders. Each capability receives its own api.md, database.md, and events.md.

The catalog files themselves become `references.md` with links only.

---

## 5. Rollback Strategy

### 5.1 Rollback Triggers

Rollback is initiated if:
- Repository health score drops below baseline (23/100 pre-refactoring)
- Critical systems can no longer find their architecture documents
- Refactoring phase introduces more broken references than it resolves
- More than 5 documents are accidentally deleted or corrupted

### 5.2 Rollback Mechanism

The refactoring is conducted in a dedicated git branch:

```
main ──────────────────────────────────────── (stable)
         \
          knowledge-arch-refactoring ──────── (refactoring work)
```

Each sub-phase (A through F) is a separate commit with a clear tag:
```
v2.0D.1-A — Foundation structure
v2.0D.1-B — Classification complete
v2.0D.1-C — Archive complete
v2.0D.1-D — Capability migration complete
v2.0D.1-E — Metadata complete
v2.0D.1-F — Indexes complete + GATE
```

**Rollback to any phase:**
```
git revert [phase tag]
```

No content is ever hard-deleted. `git mv` is used for all file moves.

### 5.3 Rollback Test

Before merging each phase, verify:
1. `find architecture/ -name "*.md" | wc -l` has not decreased (no accidental deletes)
2. Existing documents can be found via the old paths or redirects
3. The audit tool produces no new Critical violations not present at phase start

---

## 6. Effort Estimate

| Phase | Work | Estimated Effort | Risk |
|-------|------|-----------------|------|
| 2.0D.1-A | Foundation structure | 3–5 days | Low |
| 2.0D.1-B | Classification | 5–7 days | Medium |
| 2.0D.1-C | Archive migration | 2–3 days | Low |
| 2.0D.1-D | Capability migration | 15–20 days | High |
| 2.0D.1-E | Metadata application | 7–10 days | Medium |
| 2.0D.1-F | Index generation | 3–5 days | Low |
| **Total** | | **35–50 days** | **Medium** |

**Risk factors:**
- Document content is more complex than estimated (HIGH risk for master document)
- Additional orphaned documents discovered during classification
- Implementation evidence is harder to verify than expected

---

## 7. Parallel Work Constraints

During Phase 2.0D.1:

| Constraint | Rule |
|-----------|------|
| No new platform features | Prevents architecture documents from going stale immediately after being created |
| No architecture document deletions | Only deprecation and archival — never delete |
| All new architecture documents | Must comply with new standards immediately |
| Audit tool must be usable by end of Phase A | Required for Phase B–F validation |

---

## References

- [KNOWLEDGE-ARCHITECTURE-VISION.md](KNOWLEDGE-ARCHITECTURE-VISION.md) — Why this refactoring is needed
- [DOCUMENT-FOLDER-STANDARDS.md](DOCUMENT-FOLDER-STANDARDS.md) — Target structure
- [KNOWLEDGE-HEALTH-METRICS.md](KNOWLEDGE-HEALTH-METRICS.md) — Phase gate criteria
- [KNOWLEDGE-EVOLUTION-STRATEGY.md](KNOWLEDGE-EVOLUTION-STRATEGY.md) — Migration and deprecation strategy
- [REPOSITORY-AUDIT-RULES.md](REPOSITORY-AUDIT-RULES.md) — Validation tool specification
