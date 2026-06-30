# KNW-KE-ARCH-000 — Knowledge Ecosystem Navigation

**Phase:** 3.0C.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

The Knowledge Ecosystem is the canonical infrastructure that all runtimes, products, and agents consume. No object lives in two places. No documentation lives outside this ecosystem. Knowledge becomes reusable infrastructure.

---

## Reading Order

**For architects:** 01 → 02 → 27 → 26 → 30

**For contributors:** 04 → 05 → 03 → 06 → 07

**For SDK implementers:** 14 → 15 → 10 → 28

**For quality engineers:** 16 → 17 → 18 → 20 → 21

**For full study (6 tiers):** see tier map below

---

## Tier Map

### Tier 1 — Repository Foundation

| # | Document | Summary |
|---|----------|---------|
| 01 | [KNOWLEDGE-REPOSITORY](01-KNOWLEDGE-REPOSITORY.md) | Canonical repo design — single source of truth |
| 02 | [KNOWLEDGE-PACKAGES](02-KNOWLEDGE-PACKAGES.md) | Domain package system — 9 standard packages |
| 03 | [KNOWLEDGE-TEMPLATES](03-KNOWLEDGE-TEMPLATES.md) | Templates for all 33 object types |
| 04 | [KNOWLEDGE-STYLE-GUIDE](04-KNOWLEDGE-STYLE-GUIDE.md) | Writing, naming, formatting rules |
| 05 | [KNOWLEDGE-NAMING-STANDARD](05-KNOWLEDGE-NAMING-STANDARD.md) | ID, name, URI, tag, namespace naming |

### Tier 2 — Tooling

| # | Document | Summary |
|---|----------|---------|
| 06 | [KNOWLEDGE-LINTER](06-KNOWLEDGE-LINTER.md) | `kos lint` — 32 checks |
| 07 | [KNOWLEDGE-FORMATTER](07-KNOWLEDGE-FORMATTER.md) | `kos format` — stable canonical output |
| 08 | [KNOWLEDGE-CI](08-KNOWLEDGE-CI.md) | CI/CD pipeline for knowledge |
| 09 | [KNOWLEDGE-CATALOG](09-KNOWLEDGE-CATALOG.md) | Faceted discovery and search |
| 10 | [KNOWLEDGE-SCHEMAS](10-KNOWLEDGE-SCHEMAS.md) | JSON/YAML schemas for all types |

### Tier 3 — Content

| # | Document | Summary |
|---|----------|---------|
| 11 | [KNOWLEDGE-EXAMPLES](11-KNOWLEDGE-EXAMPLES.md) | Canonical examples per type |
| 12 | [KNOWLEDGE-GOLDEN-DATASET](12-KNOWLEDGE-GOLDEN-DATASET.md) | Reference + regression + AI eval dataset |
| 13 | [KNOWLEDGE-BENCHMARKS](13-KNOWLEDGE-BENCHMARKS.md) | Benchmark objects for all 19 algorithms |
| 14 | [KNOWLEDGE-REGISTRY](14-KNOWLEDGE-REGISTRY.md) | Registry protocol contracts |
| 15 | [KNOWLEDGE-SDK-CONTRACT](15-KNOWLEDGE-SDK-CONTRACT.md) | Language-neutral SDK interfaces |

### Tier 4 — Quality

| # | Document | Summary |
|---|----------|---------|
| 16 | [KNOWLEDGE-QUALITY](16-KNOWLEDGE-QUALITY.md) | 9 quality dimensions applied to the ecosystem |
| 17 | [KNOWLEDGE-SCORING](17-KNOWLEDGE-SCORING.md) | Detailed scoring formulas |
| 18 | [KNOWLEDGE-COVERAGE](18-KNOWLEDGE-COVERAGE.md) | Coverage dimensions and thresholds |
| 19 | [KNOWLEDGE-LIFECYCLE](19-KNOWLEDGE-LIFECYCLE.md) | Ecosystem object lifecycle |
| 20 | [KNOWLEDGE-VERIFICATION](20-KNOWLEDGE-VERIFICATION.md) | Verification suite — 50+ checks |

### Tier 5 — Governance

| # | Document | Summary |
|---|----------|---------|
| 21 | [KNOWLEDGE-ARCHITECTURE-TESTS](21-KNOWLEDGE-ARCHITECTURE-TESTS.md) | Architecture-level test suite |
| 22 | [KNOWLEDGE-REFERENCE-LIBRARY](22-KNOWLEDGE-REFERENCE-LIBRARY.md) | Standard objects and packages |
| 23 | [KNOWLEDGE-CANONICAL-SOURCES](23-KNOWLEDGE-CANONICAL-SOURCES.md) | Source hierarchy and conflict resolution |
| 24 | [KNOWLEDGE-IMPORT-EXPORT](24-KNOWLEDGE-IMPORT-EXPORT.md) | Import/export formats |
| 25 | [KNOWLEDGE-CHANGE-MANAGEMENT](25-KNOWLEDGE-CHANGE-MANAGEMENT.md) | Change process, deprecation, migration |

### Tier 6 — Package System

| # | Document | Summary |
|---|----------|---------|
| 26 | [KNOWLEDGE-PACKAGE-MANAGER](26-KNOWLEDGE-PACKAGE-MANAGER.md) | `kos install/remove/upgrade` |
| 27 | [KNOWLEDGE-DEPENDENCY-MODEL](27-KNOWLEDGE-DEPENDENCY-MODEL.md) | Package and object dependencies |
| 28 | [KNOWLEDGE-METADATA](28-KNOWLEDGE-METADATA.md) | Metadata categories and inheritance |
| 29 | [KNOWLEDGE-TRACEABILITY](29-KNOWLEDGE-TRACEABILITY.md) | Knowledge → runtime traceability |
| 30 | [KNOWLEDGE-FREEZE](30-KNOWLEDGE-FREEZE.md) | Architecture freeze declaration |

---

## Core Invariants

1. **One canonical location** — every object exists in exactly one package
2. **No duplicate knowledge** — linter enforces; duplicates are build errors
3. **Package owns its objects** — no object spans two packages
4. **Versioned** — every change creates a version record
5. **Traceable** — every runtime object links to its knowledge source
6. **Machine-verifiable** — every rule is checkable by `kos lint` or `kos verify`

---

## Key Numbers

| Metric | Value |
|--------|-------|
| Standard packages | 9 |
| Object templates | 33 (one per KnowledgeObjectType) |
| Linter checks | 32 |
| SDK languages | 3 (Python, Java, TypeScript) |
| Quality dimensions | 9 |
| Verification checks | 50+ |
| Freeze date | 2026-06-30 |
