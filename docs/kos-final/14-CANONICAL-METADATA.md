# KNW-FINAL-014 — Canonical Metadata

**Phase:** 3.0D.0.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Provides canonical metadata patterns, anti-patterns, and the authoritative list of valid values for all enumerated metadata fields.

---

## Complete Metadata Block (Golden Pattern)

```yaml
# Identity
knowledge_id: KNW-PLT-MOD-001
object_type: module
canonical_name: plt.module.quota-manager
object_uri: knw://plt/module/quota-manager@1.0.0

# Classification
domain: PLATFORM
namespace: plt
tags:
  - quota
  - resource-management
  - platform-core
labels:
  team: platform
  criticality: high

# Lifecycle
lifecycle:
  state: CANONICAL
  version: "1.0.0"
  created_at: "2026-06-30"
  updated_at: "2026-06-30"
  deprecated_at: null
  sunset_at: null
  archived_at: null
  deprecation_reason: null
  successor_id: null

# Ownership
ownership:
  owner: team:platform
  team: team:platform
  maintainers:
    - user:jane.doe
  stakeholders:
    - team:runtime

# Operational (system-set — do not author these)
# quality_score: 0.87    ← set by scoring engine
# checksum: sha256:abc   ← set by identity engine
# created_at: ...        ← set by registry
# updated_at: ...        ← set by registry
```

---

## Valid Enum Values

### object_type (33 values)

```
MODULE, SERVICE, REQUIREMENT, DECISION, SPECIFICATION,
ALGORITHM, PATTERN, API, TEST, BENCHMARK, CONFIGURATION,
DEPLOYMENT, MONITORING, PROVIDER, RUNTIME, WORKFLOW,
DATASET, FINANCIAL, METRIC, EVENT, SCHEMA, CONTRACT,
TEMPLATE, GLOSSARY, STANDARD, POLICY, CAPABILITY,
INTEGRATION, PRODUCT, REPORT, RESEARCH, MIGRATION, META
```

### lifecycle.state (6 values)

```
DRAFT → PROPOSED → VERIFIED → CANONICAL → DEPRECATED → ARCHIVED
```

### domain (9 values mapped from domain_code)

| domain_code | domain |
|-------------|--------|
| PLT | PLATFORM |
| RT | RUNTIME |
| PROV | PROVIDER |
| ALG | ALGORITHM |
| PAT | PATTERN |
| API | API |
| TEST | TEST |
| FIN | FINANCIAL |
| META | META |

### evidence_type (10 values)

```
EV-HAPPROVAL, EV-DOC, EV-BENCHMARK, EV-TEST, EV-CODE,
EV-MONITORING, EV-PROPOSAL, EV-USER, EV-STANDARD, EV-RESEARCH
```

---

## ID Format Rules

```
knowledge_id regex: ^KNW-[A-Z][A-Z0-9]*-[A-Z][A-Z0-9]*-[A-Za-z0-9_-]+$

Examples:
  VALID:    KNW-PLT-MOD-001
  VALID:    KNW-ALG-ALG-007
  VALID:    KNW-TEST-BENCH-001
  INVALID:  mod-001           (no KNW prefix)
  INVALID:  KNW-plt-mod-001   (lowercase domain)
  INVALID:  KNW-PLT-MOD       (no sequence number)
```

```
canonical_name regex: ^[a-z][a-z0-9]*\.[a-z][a-z0-9]*\.[a-z][a-z0-9-]*$

Examples:
  VALID:   plt.module.quota-manager
  VALID:   alg.algorithm.bm25-text-ranker
  INVALID: PLT.module.quota-manager   (uppercase)
  INVALID: plt.module.quota_manager   (underscore, not hyphen)
  INVALID: plt.mod.quota              (type must be full word)
```

---

## Tag Registry (Canonical Tags)

```yaml
# Excerpt from knowledge/catalog/tags.yaml
tags:
  # Platform tags
  - quota
  - resource-management
  - platform-core
  - routing
  - session-management
  - scheduling

  # Algorithm tags
  - ranking
  - search
  - graph
  - ml
  - nlp
  - embedding

  # Provider tags
  - llm
  - provider
  - openai
  - anthropic

  # Quality tags
  - high-quality
  - experimental
  - deprecated-soon
  - benchmark-critical

  # Lifecycle tags
  - needs-evidence
  - needs-test
  - review-required
```

---

## Metadata Anti-Patterns

| Field | Anti-Pattern | Error | Fix |
|-------|-------------|-------|-----|
| `knowledge_id` | `mod-001` | MD-001 | Use `KNW-PLT-MOD-001` |
| `canonical_name` | `PLT.Module.QuotaManager` | MD-002 | Use `plt.module.quota-manager` |
| `object_type` | `CORE_MODULE` | MD-003 | Use one of 33 valid types |
| `namespace` | `platform` | MD-004 | Use `plt` (3-5 char code) |
| `tags` | `["QUOTA", "Resource"]` | MD-005 | Use lowercase hyphens from registry |
| `lifecycle.state` | `APPROVED` | MD-011 | Use one of 6 valid states |
| `lifecycle.version` | `v1` | MD-012 | Use semver `1.0.0` |
| `quality_score` (authored) | Any value | MD-018 (system field) | Remove from YAML; set by system |
| `owner` | `Jane Doe` | MD-017 | Use `team:X` or `user:X` format |

---

## Metadata Completeness Requirements by State

| Field | DRAFT | PROPOSED | VERIFIED | CANONICAL |
|-------|-------|----------|----------|-----------|
| `knowledge_id` | Required | Required | Required | Required |
| `object_type` | Required | Required | Required | Required |
| `canonical_name` | Required | Required | Required | Required |
| `name` | Required | Required | Required | Required |
| `description` | Optional | Required | Required | Required |
| `owner` | Optional | Required | Required | Required |
| `lifecycle.version` | Required | Required | Required | Required |
| `tags` | Optional | Optional | Required | Required |
| `traceability.satisfies` | Optional | Optional | Optional | Required (MODULE) |
| `evidence.items` | Optional | Optional | Optional | Required (≥1) |

---

## Cross-References

- Metadata spec → Phase 3.0C.5 `28-KNOWLEDGE-METADATA`
- Metadata certification → Phase 3.0D.0 `06-METADATA-CERTIFICATION`
- Naming standard → Phase 3.0C.5 `05-KNOWLEDGE-NAMING-STANDARD`
- Tag registry → Phase 3.0C.5 `09-KNOWLEDGE-CATALOG`
