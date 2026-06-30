# KNW-KE-ARCH-002 — Knowledge Packages

**Phase:** 3.0C.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Every domain becomes a Knowledge Package. A package is the unit of ownership, versioning, and distribution for Knowledge Objects.

---

## Package Manifest Format

```yaml
# knowledge/packages/{name}/package.yaml
id: kos.platform.package
name: Platform Package
version: 1.0.0
namespace: plt
domain_code: PLT
status: CANONICAL
owner: team:platform
description: All Knowledge Objects for the KOS Platform layer.
license: internal
repository: knowledge/packages/platform/

dependencies:
  - id: kos.meta.package
    version: ">=1.0.0"

exports:
  object_types: [MODULE, SERVICE, CONFIGURATION, DEPLOYMENT]
  namespaces: [plt]
  tags: [platform]

contents:
  objects: 0        # computed by `kos stats`
  relationships: 0
  examples: 0
  tests: 0
  benchmarks: 0

quality:
  min_score: 0.75
  min_evidence_score: 0.55
```

---

## Standard Packages

### PKG-01: Platform Package

```
id:          kos.platform.package
namespace:   plt
domain_code: PLT
owner:       team:platform
object_types: [MODULE, SERVICE, CONFIGURATION, DEPLOYMENT]
```

Contains: all platform layer modules, services, configurations, deployment objects.

### PKG-02: Runtime Package

```
id:          kos.runtime.package
namespace:   rt
domain_code: RT
owner:       team:runtime
object_types: [RUNTIME, KERNEL, SDK, PACKAGE]
```

Contains: AI runtime, kernels, SDKs, and runtime sub-packages.

### PKG-03: Provider Package

```
id:          kos.provider.package
namespace:   prov
domain_code: PROV
owner:       team:provider
object_types: [PROVIDER, CAPABILITY]
```

Contains: AI provider definitions, model capabilities, auth configurations.

### PKG-04: Algorithm Package

```
id:          kos.algorithm.package
namespace:   alg
domain_code: ALG
owner:       team:core
object_types: [ALGORITHM]
```

Contains: all 19 algorithms (A-01–A-12, graph algorithms DFS–GraphDiff).

### PKG-05: Pattern Package

```
id:          kos.pattern.package
namespace:   pat
domain_code: PAT
owner:       team:architecture
object_types: [PATTERN, ARCHITECTURE, DECISION, REQUIREMENT, SPECIFICATION]
```

Contains: architectural patterns, ADRs, requirements, specifications.

### PKG-06: API Package

```
id:          kos.api.package
namespace:   api
domain_code: API
owner:       team:platform
object_types: [API]
```

Contains: all API family definitions (REST, gRPC, MCP, CLI, Python, WebSocket, Admin, Internal).

### PKG-07: Test Package

```
id:          kos.test.package
namespace:   test
domain_code: TEST
owner:       team:quality
object_types: [TEST, BENCHMARK]
```

Contains: all test objects and benchmark objects. References objects in other packages.

### PKG-08: Financial Package

```
id:          kos.financial.package
namespace:   fin
domain_code: FIN
owner:       team:financial
object_types: [MARKET, FOREX, NEWS, DATASET]
```

Contains: market knowledge objects, forex pairs, news sources, financial datasets.

### PKG-09: Meta Package

```
id:          kos.meta.package
namespace:   meta
domain_code: META
owner:       architecture-board
object_types: [DOCUMENT, KNOWLEDGE_BASE, PRODUCT, AGENT, PROMPT, CONVERSATION, TASK, STRATEGY, EXECUTION_PLAN]
```

Contains: platform documents, knowledge bases, product definitions, AI agents.

---

## Package Directory Layout

```
knowledge/packages/{name}/
├── package.yaml          # manifest
├── objects/              # YAML files, one per object
│   ├── {slug}.yaml
│   └── ...
├── relationships/        # relationship definitions
│   └── {source}-{type}-{target}.yaml
├── evidence/             # evidence records
│   └── {knowledge_id}-{ev_type}.yaml
├── tests/                # test references (not implementation)
│   └── {test_id}.yaml
├── examples/             # worked examples
│   └── {slug}-example.yaml
├── benchmarks/           # benchmark objects
│   └── {bench_id}.yaml
├── docs/                 # package documentation
│   ├── README.md
│   └── CHANGELOG.md
└── index.yaml            # generated object index
```

---

## Package Rules

| Rule | Description |
|------|-------------|
| PKG-001 | Every object belongs to exactly one package |
| PKG-002 | A package may depend on another package but may not directly include its objects |
| PKG-003 | Circular package dependencies are a build error |
| PKG-004 | Every package must have a `package.yaml` at its root |
| PKG-005 | Package `namespace` is globally unique |
| PKG-006 | Package `domain_code` is globally unique |
| PKG-007 | A package owns all `domain_code`-prefixed knowledge IDs |
| PKG-008 | `kos verify package {name}` must pass before any release |

---

## Package Versioning

Packages follow SemVer (`major.minor.patch`):

| Change | Version bump |
|--------|-------------|
| New object added | MINOR |
| Object removed or renamed | MAJOR |
| Field changed on existing object | MAJOR if breaking, MINOR if additive |
| Documentation / evidence update | PATCH |
| Dependency version change | MINOR or MAJOR (depends on whether breaking) |

---

## Package Lock File

```yaml
# knowledge/packages/{name}/package-lock.yaml
locked_at: "2026-06-30T00:00:00Z"
dependencies:
  kos.meta.package:
    version: "1.0.0"
    checksum: sha256:abc123
    source: "knowledge/packages/meta"
```

---

## Cross-References

- Dependency resolution → `27-KNOWLEDGE-DEPENDENCY-MODEL`
- Package manager commands → `26-KNOWLEDGE-PACKAGE-MANAGER`
- Repository layout → `01-KNOWLEDGE-REPOSITORY`
- Package quality gates → `08-KNOWLEDGE-CI`
