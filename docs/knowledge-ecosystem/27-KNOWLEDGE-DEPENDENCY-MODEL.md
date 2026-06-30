# KNW-KE-ARCH-027 — Knowledge Dependency Model

**Phase:** 3.0C.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Defines how packages and objects express, resolve, and validate dependencies. No runtime code — specification only.

---

## Dependency Types

### Package Dependencies

A package depends on another package when it references objects from it.

```yaml
# In package.yaml:
dependencies:
  kos.meta.package:
    version: ">=1.0.0,<2.0.0"   # semver range
    optional: false               # HARD dependency
  kos.algorithm.package:
    version: ">=1.0.0"
    optional: true                # SOFT dependency (can function without it)
```

### Object Dependencies

An object depends on another when it references it in:
- `traceability.satisfies` — satisfies a requirement in another package
- `traceability.implements` — implements an object in another package
- `traceability.tests` — tests an object in another package
- Relationship `DEPENDS_ON`, `USES`, `EXTENDS`

Object dependencies may be:
- **HARD** — `DEPENDS_ON` or `IMPLEMENTS` — the object cannot function without the target
- **SOFT** — `USES` — the object can degrade gracefully if the target is absent
- **OPTIONAL** — `REFERENCES` or `RELATED_TO` — informational only

---

## Dependency Constraints

```yaml
# Object-level dependency in relationship file:
- relation_id: REL-001
  relation_type: DEPENDS_ON
  source_id: KNW-PLT-MOD-001   # Quota Manager
  target_id: KNW-RT-RT-001      # AI Runtime (in different package)
  strength: HARD
  version_constraint: ">=1.0.0"
  optional: false
  reason: "Quota Manager requires AI Runtime for context propagation"
```

---

## Version Constraint Syntax

```
Exact:        ==1.2.3
Minimum:      >=1.0.0
Range:        >=1.0.0,<2.0.0
Wildcard:     1.x.x    (any patch+minor)
Any:          *
```

---

## Dependency Resolution Algorithm

```
RESOLVE(package, constraints) → resolved_versions

  1. Build dependency graph G
     G = {package: [direct dependencies]}

  2. Topological sort G → installation_order
     If cycle detected → error (DM-001)

  3. For each package in installation_order:
     a. Find all version constraints from dependent packages
     b. Compute satisfying version (highest compatible)
     c. If no version satisfies all constraints → error (DM-002)

  4. Return resolved_versions map

COMPLEXITY: O(P² * V) where P = packages, V = versions per package
```

---

## Dependency Rules

| Rule | Description |
|------|-------------|
| DM-001 | Circular package dependencies are a build error |
| DM-002 | Unsatisfiable version constraints are a build error |
| DM-003 | Objects may not reference objects from packages not listed in their package's dependencies |
| DM-004 | Optional dependencies must degrade gracefully (object must function without them) |
| DM-005 | Version pins in lock file override range constraints |
| DM-006 | No diamond dependency conflicts allowed (use lock file to resolve) |

---

## Dependency Graph

```
kos.platform.package@1.0.0
  └── kos.meta.package@1.0.0        [HARD, >=1.0.0,<2.0.0]
  └── kos.algorithm.package@1.0.0   [SOFT, >=1.0.0]

kos.test.package@1.0.0
  └── kos.platform.package@1.0.0   [HARD, >=1.0.0]
  └── kos.meta.package@1.0.0       [HARD, >=1.0.0]

kos.financial.package@1.0.0
  └── kos.meta.package@1.0.0       [HARD, >=1.0.0]

kos.algorithm.package@1.0.0
  └── (no dependencies)

kos.meta.package@1.0.0
  └── (no dependencies)

kos.pattern.package@1.0.0
  └── kos.meta.package@1.0.0       [HARD, >=1.0.0]

kos.provider.package@1.0.0
  └── kos.meta.package@1.0.0       [HARD, >=1.0.0]

kos.api.package@1.0.0
  └── kos.platform.package@1.0.0   [HARD, >=1.0.0]

kos.runtime.package@1.0.0
  └── kos.platform.package@1.0.0   [HARD, >=1.0.0]
  └── kos.provider.package@1.0.0   [HARD, >=1.0.0]
```

---

## Cross-Package Reference Check

The linter checks cross-package references at PR time:

```bash
kos verify dependencies             # check all cross-package references
kos verify dependencies --package kos.platform.package
```

This checks that:
1. Every object reference in `traceability.*` refers to an object in an installed package
2. Every relationship `target_id` is in an installed package
3. No package declares a dependency it doesn't actually use
4. No package uses objects from packages not in its dependency list

---

## Dependency Commands

```bash
kos deps list {package}             # list all dependencies
kos deps tree {package}             # show dependency tree
kos deps check                      # check for conflicts
kos deps why {knowledge_id}         # why is this object a dependency?
kos deps unused                     # find declared-but-unused dependencies
kos deps missing                    # find used-but-undeclared dependencies
```

---

## Cross-References

- Package manifest with dependencies → `02-KNOWLEDGE-PACKAGES`
- Package manager uses dependency resolver → `26-KNOWLEDGE-PACKAGE-MANAGER`
- Relationship engine traverses deps → Phase 3.0C `06-RELATIONSHIP-ENGINE`
- Phase 3.0C dependency closure algorithm → Phase 3.0C `25-GRAPH-ALGORITHMS` G-06
