# KNW-FINAL-015 — Canonical Namespaces

**Phase:** 3.0D.0.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Defines all 9 canonical KOS namespaces — their codes, domains, ID prefixes, object counts, and ownership. These are permanently fixed in KOS v1.0.

---

## Namespace Registry

| Namespace | Domain Code | Domain | Package | Owner | Objects (target) |
|-----------|-------------|--------|---------|-------|-----------------|
| `plt` | PLT | PLATFORM | kos.platform.package | team:platform | 20,000 |
| `rt` | RT | RUNTIME | kos.runtime.package | team:runtime | 18,000 |
| `prov` | PROV | PROVIDER | kos.provider.package | team:platform | 12,000 |
| `alg` | ALG | ALGORITHM | kos.algorithm.package | team:ai | 15,000 |
| `pat` | PAT | PATTERN | kos.pattern.package | team:architecture | 10,000 |
| `api` | API | API | kos.api.package | team:platform | 8,000 |
| `test` | TEST | TEST | kos.test.package | team:qa | 8,000 |
| `fin` | FIN | FINANCIAL | kos.financial.package | team:finance | 5,000 |
| `meta` | META | META | kos.meta.package | team:architecture-board | 4,000 |

---

## Namespace 1: plt — PLATFORM

**Purpose:** All platform-level modules, services, requirements, configurations, and decisions.

ID range: `KNW-PLT-{TYPE}-{NNN}`

```
KNW-PLT-MOD-NNN   → Modules (Quota Manager, AI Router, etc.)
KNW-PLT-SVC-NNN   → Services
KNW-PLT-REQ-NNN   → Requirements
KNW-PLT-CONF-NNN  → Configurations
KNW-PLT-DEC-NNN   → Decisions
KNW-PLT-DEP-NNN   → Deployments
KNW-PLT-MON-NNN   → Monitoring
```

Canonical objects in this namespace own the platform behaviour that all other namespaces depend upon.

---

## Namespace 2: rt — RUNTIME

**Purpose:** AI Runtime engine, execution planners, context management, and runtime-level services.

ID range: `KNW-RT-{TYPE}-{NNN}`

```
KNW-RT-RT-NNN     → Runtime objects
KNW-RT-SVC-NNN    → Services (AI Router Service, etc.)
KNW-RT-MOD-NNN    → Modules
KNW-RT-DEP-NNN    → Deployments
```

---

## Namespace 3: prov — PROVIDER

**Purpose:** AI provider adapters, provider registry, capability declarations.

ID range: `KNW-PROV-{TYPE}-{NNN}`

```
KNW-PROV-PROV-NNN → Provider objects (OpenAI, Anthropic, etc.)
KNW-PROV-SVC-NNN  → Provider Registry Service
KNW-PROV-CAP-NNN  → Capability declarations
```

---

## Namespace 4: alg — ALGORITHM

**Purpose:** All algorithms — search, ranking, graph, ML, scheduling.

ID range: `KNW-ALG-{TYPE}-{NNN}`

```
KNW-ALG-ALG-NNN   → Algorithm objects
KNW-ALG-ALG-001   → BFS
KNW-ALG-ALG-002   → DFS
KNW-ALG-ALG-003   → Token Bucket Rate Limiter
...
KNW-ALG-ALG-019   → Last canonical algorithm
```

Algorithms have zero dependencies — this is the leaf package.

---

## Namespace 5: pat — PATTERN

**Purpose:** Architectural and design patterns.

ID range: `KNW-PAT-{TYPE}-{NNN}`

```
KNW-PAT-PAT-NNN   → Pattern objects
KNW-PAT-PAT-001   → Registry Pattern
KNW-PAT-PAT-002   → Circuit Breaker
KNW-PAT-PAT-003   → Event Sourcing
KNW-PAT-PAT-004   → CQRS
```

---

## Namespace 6: api — API

**Purpose:** API contracts, endpoint definitions, request/response schemas.

ID range: `KNW-API-{TYPE}-{NNN}`

```
KNW-API-API-NNN   → API objects
KNW-API-API-001   → Knowledge Query API
KNW-API-API-002   → Provider API
KNW-API-API-003   → Registry API
```

---

## Namespace 7: test — TEST

**Purpose:** Test objects (TEST type) and Benchmark objects (BENCHMARK type).

ID range: `KNW-TEST-{TYPE}-{NNN}`

```
KNW-TEST-TST-NNN   → Test objects
KNW-TEST-BENCH-NNN → Benchmark objects
KNW-TEST-BENCH-001 through KNW-TEST-BENCH-019 are canonical benchmarks
```

---

## Namespace 8: fin — FINANCIAL

**Purpose:** Billing models, pricing rules, cost tracking.

ID range: `KNW-FIN-{TYPE}-{NNN}`

```
KNW-FIN-FIN-NNN   → Financial objects
KNW-FIN-REQ-NNN   → Financial requirements
KNW-FIN-CONF-NNN  → Pricing configurations
```

---

## Namespace 9: meta — META

**Purpose:** Cross-cutting architecture decisions, standards, and glossary.

ID range: `KNW-META-{TYPE}-{NNN}`

```
KNW-META-DEC-NNN  → Architecture Decision Records (ADRs)
KNW-META-STD-NNN  → Standards
KNW-META-GLOSS-NNN → Glossary entries
KNW-META-WF-NNN   → Workflow objects
```

The meta namespace is the root — it has no dependencies.

---

## Cross-Namespace Reference Rules

| Rule | Description |
|------|-------------|
| NS-001 | An object may only use its own namespace in `canonical_name` |
| NS-002 | Cross-namespace references are allowed only if the target's package is declared as a dependency |
| NS-003 | The `plt` namespace may reference `alg` and `meta` |
| NS-004 | The `rt` namespace may reference `plt`, `prov`, and `meta` |
| NS-005 | The `meta` namespace has no cross-namespace dependencies |
| NS-006 | The `test` namespace may reference any namespace (tests all packages) |
| NS-007 | No namespace may create a circular dependency with another |

---

## Namespace Addition Policy

New namespaces may only be added via:
1. ADR documenting the new domain
2. Architecture Board unanimous vote
3. New phase designation (minimum KOS v1.1)

KOS v1.0 is permanently frozen at 9 namespaces.

---

## Cross-References

- Package definitions → `05-CANONICAL-PACKAGES`
- Naming standard → Phase 3.0C.5 `05-KNOWLEDGE-NAMING-STANDARD`
- Dependency model → Phase 3.0C.5 `27-KNOWLEDGE-DEPENDENCY-MODEL`
