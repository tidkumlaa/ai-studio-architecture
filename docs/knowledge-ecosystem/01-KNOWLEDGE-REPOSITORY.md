# KNW-KE-ARCH-001 — Knowledge Repository

**Phase:** 3.0C.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

The Knowledge Repository is the single source of truth for all Knowledge Objects in the KOS ecosystem. Everything that every runtime, product, and agent depends upon lives here and only here.

---

## Repository Root Layout

```
knowledge/
├── packages/           # domain packages (one dir per package)
│   ├── platform/
│   ├── runtime/
│   ├── provider/
│   ├── algorithm/
│   ├── pattern/
│   ├── api/
│   ├── test/
│   └── financial/
├── reference/          # canonical reference objects (read-only)
├── templates/          # object templates for authoring
├── examples/           # worked examples (pass the linter)
├── schemas/            # JSON/YAML schemas
├── style/              # style guide source
├── lint/               # linter rule definitions
├── formatter/          # formatter rule definitions
├── dataset/            # golden dataset
├── benchmarks/         # performance benchmark objects
├── registry/           # registry index and catalog files
├── catalog/            # discovery metadata
├── contracts/          # SDK contracts
├── verification/       # verification scripts and reports
├── reports/            # generated quality reports
├── README.md           # human navigation
└── index.yaml          # machine navigation
```

---

## Repository Rules

| Rule | Description |
|------|-------------|
| KR-001 | Every Knowledge Object lives in exactly one package |
| KR-002 | No object may be copied between packages — use a dependency reference |
| KR-003 | Every object must have a canonical `knowledge_id` matching KNW-{DOMAIN}-{TYPE}-{NNN} |
| KR-004 | `reference/` is read-only — objects promoted there cannot be edited without ADR |
| KR-005 | Every directory contains an `index.yaml` listing its contents |
| KR-006 | All changes are versioned (git-based; no direct pushes to main without CI pass) |
| KR-007 | Aliases are defined in `registry/aliases.yaml` — never by copying an object |
| KR-008 | No DRAFT objects in `reference/` — minimum status is VERIFIED |

---

## Object Storage Format

Each Knowledge Object is stored as a YAML file:

```yaml
# knowledge/packages/platform/objects/quota-manager.yaml
knowledge_id: KNW-PLT-MOD-001
object_type: module
name: Quota Manager
version: 1.0.0
status: CANONICAL
owner: team:platform
description: Manages per-session and per-day resource consumption quotas.
tags: [platform, quota, resource-management]

# Phase 3.0C identity fields
canonical_name: plt.module.quota-manager
knowledge_uri: knw://plt/module/quota-manager@1.0.0

# Phase 3.0C Universal Schema sections
classification:
  domain: PLATFORM
  layer: 3
  category: INFRASTRUCTURE

evidence:
  items: []
  evidence_score: 0.0

quality:
  overall_score: null
  computed_at: null

confidence:
  declared: 0.80
  composite: null

# Domain-specific fields
module_id: module:platform:quota_manager:v1
runtime_id: ai-runtime
python_module: ai_runtime.quota.engine
is_public: true
```

---

## Versioning

The repository uses git as its version store:

| Operation | Policy |
|-----------|--------|
| New object | PR to `main`; linter must pass |
| Update DRAFT | Direct commit allowed |
| Update VERIFIED+ | PR required; reviewer must approve |
| Update CANONICAL | PR + Architecture Board sign-off |
| Rename / move | ADR required; old ID becomes alias |
| Delete CANONICAL | Never — deprecate instead |

**Branch strategy:**
- `main` — CANONICAL objects only; protected
- `develop` — VERIFIED and below; CI enforced
- `feature/*` — DRAFT work; no CI requirement

---

## Namespace Structure

Each package owns a namespace segment:

| Package | Namespace | Domain Codes |
|---------|-----------|-------------|
| platform | `plt` | PLT |
| runtime | `rt` | RT |
| provider | `prov` | PROV |
| algorithm | `alg` | ALG |
| pattern | `pat` | PAT |
| api | `api` | API |
| test | `test` | TEST |
| financial | `fin` | FIN |
| meta | `meta` | META |

---

## Discovery

```yaml
# knowledge/registry/search-index.yaml
# Machine-generated; do not edit manually
version: "1.0.0"
generated_at: "2026-06-30T00:00:00Z"
entries:
  - knowledge_id: KNW-PLT-MOD-001
    canonical_name: plt.module.quota-manager
    knowledge_uri: knw://plt/module/quota-manager@1.0.0
    object_type: module
    namespace: plt
    status: CANONICAL
    tags: [platform, quota, resource-management]
    quality_score: 0.92
```

---

## Alias System

```yaml
# knowledge/registry/aliases.yaml
aliases:
  - alias: knw://plt/module/quota@1.0.0
    canonical_uri: knw://plt/module/quota-manager@1.0.0
    reason: "Short form for internal tooling"
    created_at: "2026-06-30"
    expires_at: null
```

---

## Repository Health Metrics

| Metric | Healthy | Warning | Critical |
|--------|---------|---------|---------|
| Lint errors | 0 | 1–5 | > 5 |
| Broken references | 0 | 1–3 | > 3 |
| Orphan objects | 0 | 1–10 | > 10 |
| Objects without owner | 0 | 1–5 | > 5 |
| DRAFT objects in main | 0 | any | — |
| Quality score avg | ≥ 0.80 | 0.65–0.80 | < 0.65 |

---

## Cross-References

- Package contents → `02-KNOWLEDGE-PACKAGES`
- Object file format → `10-KNOWLEDGE-SCHEMAS`
- Linter enforces KR-001–KR-008 → `06-KNOWLEDGE-LINTER`
- Alias resolution → `14-KNOWLEDGE-REGISTRY`
- CI enforces versioning rules → `08-KNOWLEDGE-CI`
