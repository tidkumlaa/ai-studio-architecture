# KNW-FINAL-002 — Reference Repository

**Phase:** 3.0D.0.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Defines the canonical directory layout of the `knowledge/` reference repository — the single source of truth for all KOS Knowledge Objects.

---

## Repository Root Layout

```
knowledge/
├── README.md                  # Repository overview
├── index.yaml                 # Repository manifest
├── workspace.yaml             # Package workspace config
├── package-lock.yaml          # Locked dependency graph
│
├── packages/                  # All 9 standard packages
│   ├── platform/              # kos.platform.package (PLT)
│   ├── runtime/               # kos.runtime.package (RT)
│   ├── provider/              # kos.provider.package (PROV)
│   ├── algorithm/             # kos.algorithm.package (ALG)
│   ├── pattern/               # kos.pattern.package (PAT)
│   ├── api/                   # kos.api.package (API)
│   ├── test/                  # kos.test.package (TEST)
│   ├── financial/             # kos.financial.package (FIN)
│   └── meta/                  # kos.meta.package (META)
│
├── reference/                 # Reference objects (algorithm/pattern library)
│   ├── algorithms/
│   ├── patterns/
│   └── glossary.yaml
│
├── templates/                 # One YAML template per KnowledgeObjectType
│   └── {type}.template.yaml  # 33 templates
│
├── examples/                  # Canonical GOOD/BAD/WHY examples
│   ├── good/                  # 33 good examples
│   ├── bad/                   # 33 bad examples
│   └── migration/             # Before/after migration examples
│
├── schemas/                   # JSON Schema validation files
│   ├── base-schema.json
│   └── by-type/               # 33 type-specific schemas
│
├── lint/                      # Lint rule definitions
│   └── rules/                 # KL-001.yaml through KL-032.yaml
│
├── formatter/                 # Formatter configuration
│   └── formatter-config.yaml
│
├── datasets/                  # Certification and golden datasets
│   ├── golden/                # 100K object golden dataset
│   ├── regression/            # Pinned regression subset
│   ├── performance/           # Synthetic scale datasets
│   ├── traceability/          # 2K traceability chains
│   ├── adversarial/           # 500 adversarial cases
│   └── manifest.yaml          # Dataset manifest with checksums
│
├── benchmarks/                # 19 benchmark objects (KNW-TEST-BENCH-NNN)
│   └── {bench-name}.yaml
│
├── registry/                  # File-based registry
│   ├── index.yaml             # Primary object index
│   ├── aliases.yaml           # ID → canonical_name aliases
│   ├── cross-refs.yaml        # Cross-package references
│   └── stats.yaml             # Aggregate statistics
│
├── catalog/                   # Object catalog
│   ├── catalog.yaml           # Full catalog
│   └── tags.yaml              # Canonical tag registry
│
├── contracts/                 # SDK Protocol contracts
│   ├── python/
│   ├── java/
│   └── typescript/
│
├── verification/              # Verification check results
│   └── reports/
│
├── reports/                   # Quality and coverage reports
│   ├── quality/
│   ├── coverage/
│   └── certification/
│
├── acceptance/                # KAT-001 through KAT-010
│   ├── KAT-001.yaml
│   └── ...
│
└── playground/                # Interactive playground config
    ├── config.yaml
    └── scenarios/
```

---

## Package Internal Layout

Every package follows this structure:

```
knowledge/packages/{namespace}/
├── package.yaml               # Package manifest
├── README.md                  # Package overview
├── CHANGELOG.md               # Version history
│
├── objects/                   # Knowledge Object YAML files
│   ├── modules/               # MODULE objects
│   ├── services/              # SERVICE objects
│   ├── requirements/          # REQUIREMENT objects
│   ├── decisions/             # DECISION objects
│   ├── tests/                 # TEST objects
│   ├── benchmarks/            # BENCHMARK objects
│   └── ...                    # (one subdir per object type)
│
├── relationships/             # Relationship files
│   └── relationships.yaml
│
├── evidence/                  # Evidence records
│   └── evidence.yaml
│
├── docs/                      # Package documentation
│   └── ...
│
└── reports/                   # Package quality reports
    └── quality.yaml
```

---

## Object File Naming Convention

```
{knowledge_id_lower}.yaml

Examples:
  knw-plt-mod-001.yaml           → KNW-PLT-MOD-001 (Quota Manager)
  knw-alg-alg-007.yaml           → KNW-ALG-ALG-007 (BM25 Text Ranker)
  knw-test-bench-001.yaml        → KNW-TEST-BENCH-001 (Identity Bench)
```

---

## Repository Invariants

| # | Rule |
|---|------|
| RR-001 | Every object exists exactly once in the repository |
| RR-002 | Object files are named by lowercase knowledge_id |
| RR-003 | Object files are grouped by object_type within each package |
| RR-004 | Every package has package.yaml, README.md, CHANGELOG.md |
| RR-005 | Registry index.yaml is the authoritative object list |
| RR-006 | Schemas directory always contains base-schema.json + 33 type schemas |
| RR-007 | Dataset directory always contains manifest.yaml with checksums |

---

## Cross-References

- Package manifest format → Phase 3.0C.5 `02-KNOWLEDGE-PACKAGES`
- Registry format → Phase 3.0C.5 `14-KNOWLEDGE-REGISTRY`
- Dataset manifest → Phase 3.0D.0 `21-DATASET-CERTIFICATION`
- Namespace definitions → `15-CANONICAL-NAMESPACES`
