# KNW-KC-ARCH-040 — Architecture Freeze

**Phase:** 3.0C  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0  
**Freeze Date:** 2026-06-30

---

## Declaration

The Knowledge Core Architecture (Phase 3.0C) is hereby frozen.

This freeze covers all 43 documents in `architecture/docs/knowledge-core/`, the 33-type Knowledge Object Model in `platform/knowledge_runtime/`, and all algorithms, data structures, state machines, registries, and protocols defined therein.

**No structural changes may be made to this architecture without an Architecture Decision Record (ADR) reviewed and approved by the Architecture Board.**

---

## What Is Frozen

| Category | Frozen Elements |
|----------|----------------|
| Identity | 14 identity fields; KNW ID format; fingerprint algorithm |
| Object Model | 33 KnowledgeObjectType values; 16-section Universal Schema |
| Namespaces | 14 top-level domains; 32 type codes; canonical name formula |
| URIs | `knw://` scheme; 4 version channels; resolution protocol |
| Relationships | 24 relationship types; source/target constraints; cycle policies |
| Lifecycle | 7 base states; SM-1 through SM-7; all gates and thresholds |
| Evidence | 10 evidence types; weights; freshness decay parameters |
| Quality | 9 dimensions; weights; health penalty formula |
| Confidence | Composite formula; 5 confidence levels; half-life by domain |
| Traceability | Canonical chain; 4 traversal types; coverage formula |
| Registries | 9 domain registries; master hierarchy; write protocol |
| Graph | Property graph definition; 5 GI invariants; edge weight formula |
| Algorithms | A-01 through A-12; graph algorithms in `25` |
| Data Structures | DS-01 through DS-15 |
| Query Language | 8 KQL query types; SELECT/WHERE/TRAVERSE/LIMIT syntax |
| Search | BM25 parameters (k1=1.2, b=0.75); HNSW (768 dims); hybrid weights |
| Semantic | 5-step NL→KQL pipeline; 768-dim embeddings; cosine thresholds |
| Reasoning | R-1 through R-10; impact severity thresholds |
| Execution Context | 6 plan types; required knowledge lists; lifecycle |
| Performance | All P50/P99 budgets in `37`; all scale targets |
| State Machines | 7 machines with all transitions, guards, and effects |

---

## What Is NOT Frozen

| Category | Notes |
|----------|-------|
| Implementation | All runtime code; compiler; graph database engine |
| UI | No UI has been specified |
| Test code | Test implementation belongs to Phase 3.0D |
| Algorithm implementations | Pseudocode only; implementation is Phase 3.0D |
| Registry storage backend | Storage technology choice is Phase 3.0D |
| Search backend | Elasticsearch vs. Typesense vs. custom is Phase 3.0D |
| Vector store | HNSW library choice is Phase 3.0D |

---

## Freeze Conditions Met

All 50 verification checks in `38-VERIFICATION` pass:

| Area | Checks | Result |
|------|--------|--------|
| V-1 Identity | 6 | PASS |
| V-2 Completeness | 10 | PASS |
| V-3 Consistency | 10 | PASS |
| V-4 Relationships | 5 | PASS |
| V-5 Lifecycle | 6 | PASS |
| V-6 Graph | 5 | PASS |
| V-7 Registry | 5 | PASS |
| V-8 Circular Ownership | 3 | PASS |

---

## Document Inventory

43 canonical architecture documents:

| Range | Documents |
|-------|-----------|
| 00 | README (navigation) |
| 01–05 | Identity, Schema, Namespaces, URIs, Inheritance |
| 06–09 | Relationships, Relationship Types, Versioning, Snapshots |
| 10–13 | Evidence, Quality, Confidence, Traceability |
| 14–23 | Registry Architecture + 9 Domain Registries |
| 24–27 | Graph Model, Graph Algorithms, Graph Indexes, Query Language |
| 28–31 | Search, Semantic Layer, Reasoning, Execution Context |
| 32–36 | Metadata, Lifecycle Extensions, State Machines, Data Structures, Algorithms |
| 37–39 | Performance Budget, Verification, Test Strategy |
| 40 | Architecture Freeze (this document) |
| — | `index.yaml`, `README.md` |

---

## Change Process After Freeze

Changes are classified as:

| Change Class | Definition | Process |
|-------------|------------|---------|
| PATCH | Clarification, typo, cross-ref fix | Single reviewer approval |
| MINOR | Add non-breaking extension or optional field | 2 Architecture Board members |
| MAJOR | Breaking change to any frozen element | Full Architecture Board unanimous vote + ADR |
| EMERGENCY | Production incident requiring immediate change | Bypass policy in `33-LIFECYCLE-EXTENSIONS` |

All changes after freeze must:
1. Create an ADR (Architecture Decision Record) in `architecture/decisions/`
2. Bump the document version (`1.0.0` → `1.x.y`)
3. Update `CHANGE-LOG` section of the modified document
4. Re-run verification checks in `38-VERIFICATION`

---

## Phase Boundary

| Phase | Scope | Status |
|-------|-------|--------|
| 3.0A | Platform Object Model (12 runtimes) | COMPLETE |
| 3.0B | Knowledge Object Model (33 types, 184 tests) | COMPLETE |
| **3.0C** | **Knowledge Core Architecture (43 documents)** | **FROZEN** |
| 3.0D | Implementation (runtime engines, compiler, graph DB) | NEXT |
| 3.1+ | Integration, deployment, UI | FUTURE |

---

## Successor Phase

Phase 3.0D may begin immediately. It must implement exactly what this architecture specifies. Any deviation found during implementation must be surfaced as a change request against this architecture — the architecture is not silently amended by implementation choices.

---

## Architecture Board Sign-off

This freeze document supersedes all prior drafts, sketches, or notes about Knowledge Core Architecture from prior phases.

```
Frozen by:  architecture-board
Date:       2026-06-30
Version:    1.0.0
Documents:  43
Checks:     50 / 50 PASS
```

---

## Cross-References

- All documents in this set
- Verification → `38-VERIFICATION`
- Test strategy → `39-TEST-STRATEGY`
- Performance contracts → `37-PERFORMANCE-BUDGET`
- Phase 3.0B object model → `platform/knowledge_runtime/`
