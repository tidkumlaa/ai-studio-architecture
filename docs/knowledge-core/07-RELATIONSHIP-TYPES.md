# KNW-KC-ARCH-007 — Relationship Types

**Phase:** 3.0C  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

This document is the canonical catalog of all 24 recognized relationship types. Every relationship in KOS must use one of these types. New types require Architecture Board approval.

---

## Type Catalog

### Dependency Relationships

| Type | Code | Direction | Transitive | Symmetric | Description |
|------|------|-----------|-----------|-----------|-------------|
| `depends_on` | DEP | Forward | Yes | No | A requires B to function. If B is missing, A cannot execute. |
| `imports` | IMP | Forward | No | No | A imports B's interface at compile/load time. |
| `uses` | USE | Forward | No | No | A calls or consumes B's functionality at runtime. |
| `consumes` | CON | Forward | No | No | A reads B's output data stream or resource. |
| `produces` | PRD | Forward | No | No | A produces output consumed by B. |

### Architecture Relationships

| Type | Code | Direction | Transitive | Symmetric | Description |
|------|------|-----------|-----------|-----------|-------------|
| `implements` | IMPL | Forward | No | No | A provides the concrete implementation of abstract B (Interface/Protocol). |
| `extends` | EXT | Forward | Yes | No | A inherits and expands B. Subtypes are reachable via this edge. |
| `inherits` | INH | Forward | Yes | No | A inherits fields/behaviour from B (class hierarchy). |
| `instantiates` | INST | Forward | No | No | A is a running instance of type B. |
| `configures` | CONF | Forward | No | No | A is a configuration that governs B's behaviour. |

### Containment Relationships

| Type | Code | Direction | Transitive | Symmetric | Description |
|------|------|-----------|-----------|-----------|-------------|
| `contains` | CTN | Forward | Yes | No | A is the logical parent/container of B. |
| `owns` | OWN | Forward | No | No | A has exclusive ownership and lifecycle control over B. |
| `exports` | EXP | Forward | No | No | A makes B available to consumers outside its boundary. |

### Knowledge Relationships

| Type | Code | Direction | Transitive | Symmetric | Description |
|------|------|-----------|-----------|-----------|-------------|
| `references` | REF | Forward | No | No | A cites B as a source, example, or context. |
| `documents` | DOC | Forward | No | No | A is the documentation or specification for B. |
| `calls` | CALL | Forward | No | No | A invokes B (function/method/API call). |
| `related_to` | REL | Bidirectional | No | Yes | A and B are associated without a more specific relationship. |

### Quality Relationships

| Type | Code | Direction | Transitive | Symmetric | Description |
|------|------|-----------|-----------|-----------|-------------|
| `tests` | TST | Forward | No | No | A is a test or test suite that verifies B. |
| `verifies` | VER | Forward | No | No | A is evidence or proof that B meets its specification. |
| `benchmarks` | BCH | Forward | No | No | A measures the performance of B. |
| `generates` | GEN | Forward | No | No | A produces B as output (code generation, report generation). |

### Lifecycle Relationships

| Type | Code | Direction | Transitive | Symmetric | Description |
|------|------|-----------|-----------|-----------|-------------|
| `deprecates` | DEPR | Forward | No | No | A is the newer replacement that renders B deprecated. |
| `supersedes` | SUP | Forward | No | No | A fully replaces B (stronger than deprecates). |
| `conflicts_with` | CONF | Bidirectional | No | Yes | A and B cannot coexist or have incompatible semantics. |

---

## Relationship Constraints by Object Type

| Source Type → Target Type | Allowed Relationship Types |
|--------------------------|---------------------------|
| Architecture → Requirement | `implements`, `references` |
| Decision → Architecture | `documents`, `configures` |
| Module → Service | `uses`, `calls`, `depends_on` |
| Service → Capability | `implements` |
| Test → Module | `tests`, `verifies` |
| Benchmark → Module | `benchmarks` |
| API → Service | `uses`, `calls` |
| Product → Runtime | `depends_on`, `uses` |
| Requirement → Requirement | `depends_on`, `extends`, `related_to` |
| Algorithm → Module | `instantiates` |
| Prompt → Agent | `configures` |
| ConversationObject → Agent | `uses` |
| ExecutionPlan → Task | `contains` |
| Dataset → Benchmark | `uses` |
| Forex → Market | `related_to` |

---

## Relationship Strength Semantics

| Strength | Description | Impact on Deletion |
|----------|-------------|-------------------|
| `WEAK` | Informational — loss has no functional impact | Target may be removed freely |
| `NORMAL` | Standard dependency — absence degrades quality | Target removal triggers WARNING |
| `STRONG` | Functional dependency — absence causes failure | Target removal blocked until resolved |
| `CRITICAL` | Safety-critical — absence causes system failure | Target removal requires full audit |

---

## Cycle Policy by Type

| Relationship Type | Cycles Allowed? |
|-------------------|----------------|
| `depends_on` | **NO** — hard reject at write time |
| `imports` | **NO** — hard reject |
| `extends` | **NO** — hard reject |
| `inherits` | **NO** — hard reject |
| `contains` | **NO** — hard reject |
| `owns` | **NO** — hard reject |
| `uses` | YES — allowed (A uses B uses A is valid at runtime) |
| `calls` | YES — allowed (recursive call patterns) |
| `related_to` | YES — symmetric, cycles expected |
| `references` | YES — cross-references are common |
| All lifecycle types | **NO** |

---

## Adding New Relationship Types

New types must satisfy:
1. Not expressible by composition of existing types
2. Architecture Board vote required (majority)
3. Cycle policy must be defined
4. Transitive/Symmetric behaviour must be defined
5. Minimum one concrete example across existing object types

---

## Cross-References

- Relationship engine → `06-RELATIONSHIP-ENGINE`
- Graph traversal using these types → `24-KNOWLEDGE-GRAPH-MODEL`
- Traceability chain uses subset → `13-TRACEABILITY`
- Query language references type codes → `27-QUERY-LANGUAGE`
