# KNW-KC-ARCH-039 — Test Strategy

**Phase:** 3.0C  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

This document defines the test strategy for the Knowledge Core. It covers what to test, how to test it, and what coverage is required. Implementation of these tests belongs to Phase 3.0D and beyond.

---

## Test Tiers

| Tier | Name | Scope | Coverage Target |
|------|------|-------|----------------|
| T1 | Unit | Single object / algorithm in isolation | 90% line |
| T2 | Integration | Engine-to-engine interactions | 80% path |
| T3 | System | Full Knowledge Core end-to-end | 70% scenario |
| T4 | Performance | Operations vs. budget from `37` | 100% of P99 budgets |
| T5 | Verification | Architecture invariants | 100% of checks in `38` |

---

## T1: Unit Tests

### Object Model (Phase 3.0B — complete)

| Module | Target Count | Status |
|--------|-------------|--------|
| `test_kos_lifecycle.py` | ≥ 20 | DONE (21) |
| `test_kos_base_object.py` | ≥ 20 | DONE (24) |
| `test_kos_platform_objects.py` | ≥ 14 | DONE (14) |
| `test_kos_architecture_objects.py` | ≥ 19 | DONE (19) |
| `test_kos_implementation_objects.py` | ≥ 21 | DONE (21) |
| `test_kos_interface_objects.py` | ≥ 18 | DONE (18) |
| `test_kos_quality_objects.py` | ≥ 16 | DONE (16) |
| `test_kos_product_objects.py` | ≥ 26 | DONE (26) |
| `test_kos_financial_objects.py` | ≥ 29 | DONE (29) |

### Algorithm Unit Tests (Phase 3.0D)

| Algorithm | Test Cases |
|-----------|-----------|
| A-01 ID Generation | format, uniqueness, sequence |
| A-02 Checksum | determinism, collision resistance, exclusions |
| A-03 Fingerprint | determinism, discriminates domain/type |
| A-04 Quality Score | each dimension, weighted sum, health penalty |
| A-05 Evidence Freshness | fresh/degrading/stale/expired transitions |
| A-06 Confidence Decay | half-life, evidence multiplier, floor |
| A-07 BM25 | term frequency, IDF, field weights |
| A-08 Hybrid Score | weight sum = 1.0, ranking order |
| A-09 Traceability Coverage | full chain, partial chain, no chain |
| A-10 Path Confidence | single-node, multi-hop, zero-confidence edge |
| A-11 Cosine Similarity | identical, orthogonal, normalised vectors |
| A-12 Bulk Validation | all pass, partial fail, schema errors |

---

## T2: Integration Tests

### Identity + Registry

```
test_identity_registration:
  create object → register → verify in registry → resolve by URI
  create duplicate → detect by fingerprint → reject

test_alias_resolution:
  register alias → resolve via alias → get canonical object

test_bulk_registration:
  bulk 100 objects → verify all indexed → query all back
```

### Relationship Engine

```
test_relationship_creation:
  create source + target → add IMPLEMENTS relation → traverse graph

test_cycle_prevention:
  create A DEPENDS_ON B → B DEPENDS_ON A → expect cycle error

test_relationship_propagation:
  A DEPENDS_ON B, B DEPENDS_ON C → query transitive deps of A
```

### Evidence + Quality + Confidence

```
test_evidence_lifecycle:
  add EV-CODE evidence → verify freshness 1.0 → advance time → verify decay

test_quality_gates:
  object at DRAFT → add evidence → compute quality → verify REVIEW transition allowed

test_confidence_propagation:
  set A.confidence=0.9 → A IMPLEMENTS B → verify B.path_confidence influenced by A
```

### Versioning

```
test_version_bump:
  change MINOR field → bump version → verify version record created

test_version_immutability:
  publish version → attempt to mutate → expect rejection

test_version_compatibility:
  check MAJOR change breaks compatibility contract
```

---

## T3: System Tests

### Scenario S-1: Object Creation to CANONICAL

```
GIVEN: empty registry
WHEN:  create RequirementObject with all required fields
       submit for REVIEW
       add EV-CODE and EV-TEST evidence
       reviewer approves → VERIFIED
       approver signs → APPROVED
       architecture board votes → CANONICAL
THEN:  object is CANONICAL, version locked, identity stable
```

### Scenario S-2: Traceability Chain

```
GIVEN: RequirementObject CANONICAL
WHEN:  create ArchitectureObject → link SATISFIES requirement
       create ModuleObject → link IMPLEMENTS architecture
       create TestObject → link TESTS module
THEN:  forward trace: req → arch → module → test
       traceability coverage = 1.0
```

### Scenario S-3: Impact Analysis

```
GIVEN: 20-node dependency graph
WHEN:  deprecate a node with 5 direct dependents
THEN:  blast radius = expected count
       CRITICAL/HIGH/MEDIUM severity breakdown correct
       critical path identified
```

### Scenario S-4: Knowledge Query

```
GIVEN: 1000 objects in registry
WHEN:  execute KQL: SELECT objects WHERE type = "MODULE" AND quality > 0.80
THEN:  result count correct
       P99 < 50ms
       query plan uses MODULE_TYPE index
```

### Scenario S-5: Snapshot + Restore

```
GIVEN: 100 objects in CANONICAL state
WHEN:  create NAMESPACE snapshot
       modify 10 objects
       restore to snapshot
THEN:  all 10 modified objects reverted
       snapshot checksum matches
```

---

## T4: Performance Tests

All tests run against 1M objects, 10M relationships (loaded from fixture).

| Test | Budget (P99) | Measured At |
|------|-------------|-------------|
| ID generation × 10000 | < 5ms each | after cold start |
| Registry get by ID × 10000 | < 5ms each | warm cache |
| 3-hop graph traversal × 1000 | < 50ms each | warm graph |
| Dependency closure × 100 | < 200ms each | warm graph |
| Full-text search × 100 | < 500ms each | warm index |
| Semantic search × 100 | < 200ms each | warm index |
| Quality computation × 100 | < 50ms each | cold |
| Snapshot namespace (1000 objects) | < 2s | cold |

---

## T5: Architecture Verification Tests

Automated checks against the architecture documents themselves:

```
test_all_33_types_in_registry:
  load 15-OBJECT-REGISTRY → extract type codes → compare to KnowledgeObjectType enum

test_all_24_relation_types_have_constraints:
  load 07-RELATIONSHIP-TYPES → verify source/target constraints for all 24

test_quality_weights_sum_to_one:
  load 11-QUALITY-ENGINE → sum all 9 weights → assert == 1.0

test_hybrid_search_weights_sum_to_one:
  load 28-SEARCH-ENGINE → sum text+semantic+quality weights → assert == 1.0

test_no_orphan_cross_references:
  scan all Cross-Reference sections → verify all referenced doc numbers exist

test_performance_budgets_cover_all_operations:
  load 37-PERFORMANCE-BUDGET → verify at least one budget per engine
```

---

## Test Coverage Requirements

| Component | Line Coverage | Path Coverage |
|-----------|--------------|--------------|
| Object model (all 33 types) | ≥ 90% | ≥ 80% |
| Identity engine | ≥ 90% | ≥ 80% |
| Quality engine | ≥ 85% | ≥ 75% |
| Evidence engine | ≥ 85% | ≥ 75% |
| Registry engine | ≥ 85% | ≥ 75% |
| Graph engine | ≥ 80% | ≥ 70% |
| Search engine | ≥ 80% | ≥ 70% |
| Query engine | ≥ 80% | ≥ 70% |

---

## Test File Naming Convention

```
platform/tests/
  unit/
    test_kos_<component>.py       # T1 unit tests
  integration/
    test_kos_<engine>_integration.py
  system/
    test_kos_scenario_<N>.py
  performance/
    bench_<component>.py
  verification/
    test_architecture_<check>.py  # T5 verification
```

---

## Cross-References

- Performance budgets under test → `37-PERFORMANCE-BUDGET`
- Verification checks under test → `38-VERIFICATION`
- Architecture freeze → `40-ARCHITECTURE-FREEZE`
- Phase 3.0B unit tests (T1 complete) → `platform/tests/unit/`
