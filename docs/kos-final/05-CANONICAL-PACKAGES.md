# KNW-FINAL-005 — Canonical Packages

**Phase:** 3.0D.0.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Defines the 9 canonical KOS packages — their namespaces, domain codes, responsibilities, and expected object composition.

---

## Package Registry

| # | Package ID | Namespace | Domain Code | Owner |
|---|------------|-----------|-------------|-------|
| 1 | kos.platform.package | plt | PLT | team:platform |
| 2 | kos.runtime.package | rt | RT | team:runtime |
| 3 | kos.provider.package | prov | PROV | team:platform |
| 4 | kos.algorithm.package | alg | ALG | team:ai |
| 5 | kos.pattern.package | pat | PAT | team:architecture |
| 6 | kos.api.package | api | API | team:platform |
| 7 | kos.test.package | test | TEST | team:qa |
| 8 | kos.financial.package | fin | FIN | team:finance |
| 9 | kos.meta.package | meta | META | team:architecture-board |

---

## 1 — Platform Package (kos.platform.package)

**Namespace:** `plt` | **Domain:** PLATFORM

Responsibilities: Core platform modules, services, configurations, deployments, and requirements that define the KOS platform layer.

Expected objects:
- 100+ MODULE objects (quota, routing, scheduling, etc.)
- 50+ SERVICE objects
- 100+ REQUIREMENT objects
- 50+ CONFIGURATION objects
- 50+ DECISION objects

Key objects:
```
KNW-PLT-MOD-001   Quota Manager
KNW-PLT-MOD-002   AI Router
KNW-PLT-MOD-003   Session Manager
KNW-PLT-MOD-004   Routing Engine
KNW-PLT-SVC-001   Platform API Service
KNW-PLT-REQ-001   Quota Enforcement Requirement
```

Dependencies: `kos.meta.package >=1.0.0`, `kos.algorithm.package >=1.0.0` (optional)

---

## 2 — Runtime Package (kos.runtime.package)

**Namespace:** `rt` | **Domain:** RUNTIME

Responsibilities: AI Runtime engine, execution planners, context managers, and runtime-level services.

Expected objects:
- 80+ MODULE objects
- 50+ SERVICE objects
- 30+ RUNTIME objects
- 50+ REQUIREMENT objects

Key objects:
```
KNW-RT-RT-001     AI Runtime
KNW-RT-SVC-001    Context Manager Service
KNW-RT-SVC-007    AI Router Service
KNW-RT-MOD-001    Execution Planner
```

Dependencies: `kos.platform.package >=1.0.0`, `kos.provider.package >=1.0.0`

---

## 3 — Provider Package (kos.provider.package)

**Namespace:** `prov` | **Domain:** PROVIDER

Responsibilities: AI provider adapters, provider registry, capability declarations, and provider contracts.

Expected objects:
- 30+ PROVIDER objects (one per AI provider)
- 10+ SERVICE objects
- 20+ CAPABILITY objects
- 20+ CONFIGURATION objects

Key objects:
```
KNW-PROV-PROV-001  OpenAI Provider
KNW-PROV-PROV-002  Anthropic Provider
KNW-PROV-SVC-001   Provider Registry Service
```

Dependencies: `kos.meta.package >=1.0.0`

---

## 4 — Algorithm Package (kos.algorithm.package)

**Namespace:** `alg` | **Domain:** ALGORITHM

Responsibilities: All search, ranking, scoring, graph, and ML algorithms used by the platform.

Expected objects:
- 50+ ALGORITHM objects (19 canonical benchmarked)
- 20+ BENCHMARK objects
- 30+ TEST objects

Key objects:
```
KNW-ALG-ALG-001   BFS
KNW-ALG-ALG-002   DFS
KNW-ALG-ALG-003   Token Bucket Rate Limiter
KNW-ALG-ALG-007   BM25 Text Ranker
KNW-ALG-ALG-009   Tarjan SCC
```

Dependencies: none (leaf package)

---

## 5 — Pattern Package (kos.pattern.package)

**Namespace:** `pat` | **Domain:** PATTERN

Responsibilities: Architectural and design patterns used across the platform.

Expected objects:
- 50+ PATTERN objects
- 20+ DECISION objects (pattern selection ADRs)

Key objects:
```
KNW-PAT-PAT-001   Registry Pattern
KNW-PAT-PAT-002   Circuit Breaker Pattern
KNW-PAT-PAT-003   Event Sourcing Pattern
KNW-PAT-PAT-004   CQRS Pattern
```

Dependencies: `kos.meta.package >=1.0.0`

---

## 6 — API Package (kos.api.package)

**Namespace:** `api` | **Domain:** API

Responsibilities: API contracts, endpoint definitions, request/response schemas, and API versioning.

Expected objects:
- 50+ API objects
- 30+ SERVICE objects
- 20+ CONFIGURATION objects

Key objects:
```
KNW-API-API-001   Knowledge Query API
KNW-API-API-002   Provider API
KNW-API-API-003   Registry API
```

Dependencies: `kos.platform.package >=1.0.0`

---

## 7 — Test Package (kos.test.package)

**Namespace:** `test` | **Domain:** TEST

Responsibilities: All TEST and BENCHMARK objects for the platform. Imports objects from all other packages to verify them.

Expected objects:
- 100+ TEST objects
- 19+ BENCHMARK objects (canonical set)
- Test fixture objects

Key objects:
```
KNW-TEST-TST-001   Quota Enforcement Test
KNW-TEST-BENCH-001 Identity Engine Benchmark
...
KNW-TEST-BENCH-019 Coverage Benchmark
```

Dependencies: `kos.platform.package >=1.0.0`, `kos.meta.package >=1.0.0`

---

## 8 — Financial Package (kos.financial.package)

**Namespace:** `fin` | **Domain:** FINANCIAL

Responsibilities: Billing models, pricing rules, cost tracking, and financial reporting knowledge.

Expected objects:
- 30+ FINANCIAL objects
- 20+ REQUIREMENT objects
- 20+ CONFIGURATION objects

Dependencies: `kos.meta.package >=1.0.0`

---

## 9 — Meta Package (kos.meta.package)

**Namespace:** `meta` | **Domain:** META

Responsibilities: Cross-cutting concerns — architecture decisions (ADRs), glossary, standards, and meta-knowledge about the KOS itself.

Expected objects:
- 50+ DECISION objects (all ADRs)
- 30+ STANDARD objects
- 20+ GLOSSARY objects
- 10+ WORKFLOW objects

Key objects:
```
KNW-META-DEC-001  Use Python for Runtime (ADR-001)
KNW-META-DEC-002  Use Pydantic v2 (ADR-002)
KNW-META-STD-001  KNW ID Standard
```

Dependencies: none (root package — no dependencies)

---

## Package Dependency Graph

```
kos.meta.package          (no deps)
kos.algorithm.package     (no deps)
kos.pattern.package    → meta
kos.provider.package   → meta
kos.financial.package  → meta
kos.platform.package   → meta, algorithm(optional)
kos.api.package        → platform
kos.test.package       → platform, meta
kos.runtime.package    → platform, provider
```

---

## Cross-References

- Package manifest format → Phase 3.0C.5 `02-KNOWLEDGE-PACKAGES`
- Dependency model → Phase 3.0C.5 `27-KNOWLEDGE-DEPENDENCY-MODEL`
- Namespace definitions → `15-CANONICAL-NAMESPACES`
