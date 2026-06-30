# CAE-DOC-027 — CAE Testing Framework

**Phase:** 3.0D.1
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The CAE Testing Framework defines how to verify that an implementation of the
Context Assembly Engine is correct. It specifies the test categories, corpus
requirements, and the link between each CAE rule and its test case.

Testing verifies architecture correctness — not performance optimization.

---

## Test Categories

```
Category 1: Unit Tests (per stage, per rule)
  - Test each stage in isolation
  - Test each CAE rule with a minimal case
  - 120 rules → 120 minimum unit test cases

Category 2: Integration Tests (pipeline end-to-end)
  - Test the full S1→S9 pipeline with real KIL objects
  - Use canonical test corpus (100 objects across 5 packages)

Category 3: Contract Tests (API correctness)
  - Test AssemblyRequest → AssemblyResponse contract
  - Test all 6 error types are raised correctly
  - Test idempotency

Category 4: Property Tests (invariants)
  - Test the 10 Design Invariants (CI-01 through CI-10)
  - Use property-based testing (randomized inputs)

Category 5: Performance Tests (SLO verification)
  - P50, P95, P99 under load
  - Cache warm-up and hit rate
  - Degradation activation
```

---

## Test Corpus

```
CANONICAL TEST CORPUS:
  100 KnowledgeObjects across 5 packages
  20 objects per package
  Object types: SERVICE(25), MODULE(20), COMPONENT(20), INTERFACE(15), CONFIG(10), RESOURCE(10)
  Quality scores: 10% per quality band
  AIRS distribution: 20% AI-NATIVE, 40% AI-READY, 30% AI-CAPABLE, 10% AI-PARTIAL

Canonical test object: KNW-PLT-MOD-001 (Quota Manager)
  This object is used in all documentation examples and must be present.

Test relationships:
  Each object has minimum: 3 DEPENDS_ON, 1 IMPLEMENTS, 1 COMPONENT_OF
  At least 10 pairs have high semantic similarity (> 0.75) for anti-confusion tests
  At least 5 objects are DEPRECATED for status warning tests
```

---

## Critical Path Tests (Subset for CI)

The following tests must run in every CI pipeline:

```
CPT-01: Primary entity resolved for 10 standard queries
CPT-02: Budget exactly satisfied (no overflow) for 20 queries
CPT-03: Guard block present in all PromptPacks (100 samples)
CPT-04: AIRS < 0.60 raises AssemblyError (5 test objects)
CPT-05: P99 < 120ms under 10 concurrent requests (smoke test)
CPT-06: Validator rejects pack with quality < 0.70 (3 forced cases)
CPT-07: Cache hit after warm-up (L3 of primary object)
CPT-08: SearchPack returns ≤ 10 results
CPT-09: All 10 design invariants pass (property test)
CPT-10: API echoes request_id on both success and error responses
```

---

## Design Invariant Tests

| Invariant | Test |
|-----------|------|
| CI-01 Primary always L3+ | Force budget=200; assert primary.level in {L3,L4,L5} |
| CI-02 Budget is hard limit | Assert total_tokens ≤ budget for 1,000 assemblies |
| CI-03 AH Guard always runs | Intercept S8; assert it was called for 100 PromptPacks |
| CI-04 Validator must pass | Assert no pack emitted without ValidationResult.passed=true |
| CI-05 Stateless | Same request → same pack structure in 2 sequential calls |
| CI-06 context_confidence=min | Assert context_confidence=min(obj.confidence) for 100 packs |
| CI-07 Guard always L2 | Assert guard block ≤ 50 tokens × guard_count |
| CI-08 AIRS≥0.60 for PromptPack | Assert no PromptPack contains obj with AIRS<0.60 |
| CI-09 Unique pack_id | Assert 0 collisions in 10,000 assemblies |
| CI-10 Stages execute in order | Assert trace shows S1<S2<…<S9 timestamps |

---

## Rule Test Mapping

Each rule is covered by at least one test. Sample:

| Rule | Test Description |
|------|-----------------|
| CAE-001 | Context quality score formula: verify each component with known values |
| CAE-007 | Intent parser: 12 intent types classified correctly from sample queries |
| CAE-016 | Object selector: ALWAYS_INCLUDE refs always present in candidates |
| CAE-026 | Guard budget: assert guard_budget ≥ max(total×0.10, 50) for all budgets |
| CAE-031 | Primary minimum L3: assert no primary at L1/L2 even at budget=100 |
| CAE-041 | Missing KIL object: assembly continues with warning, not failure |
| CAE-046 | AIRS check: PromptPack rejects object with AIRS=0.55 |
| CAE-066 | AH Guard exemption: SearchPack emitted without guard_block |
| CAE-076 | Quality formula: verify all 4 components computed correctly |
| CAE-096 | P99 target: load test verifies < 120ms |
| CAE-111 | Learning: auto-apply blocked — proposals require approval |
| CAE-116 | API v1 stability: no breaking changes after freeze |

---

## Test Execution Order

```
Phase 1 (pre-build): Static analysis
  - No LLM inference in S1–S9 (CC-17, CC-18)
  - All 120 rules have corresponding test declarations

Phase 2 (unit): Per-rule tests
  - 120 unit tests
  - Must all pass before integration

Phase 3 (integration): CPT-01 through CPT-10
  - Run against canonical test corpus
  - Must all pass before performance tests

Phase 4 (performance): Load tests
  - P50, P95, P99 under load
  - Cache hit rate
  - Degradation activation

Phase 5 (certification): Full 50 CC checks
  - Run for every release candidate
  - CERTIFIED level required before production
```

---

## Test Fixtures

```
Fixture: minimal_query
  QueryRecord(query_id="T-001", raw_query="What is the Quota Manager?")
  ConsumerProfile(type=AI_AGENT)
  BudgetSpec(total_tokens=1000)

Fixture: budget_tight
  QueryRecord(query_id="T-002", raw_query="What is KNW-PLT-MOD-001?")
  ConsumerProfile(type=AI_AGENT)
  BudgetSpec(total_tokens=100)   # NANO — minimal budget

Fixture: search_query
  SearchRequest(query_text="quota", limit=10)

Fixture: deprecated_primary
  QueryRecord(query_id="T-003", raw_query="What is the old rate limiter?")
  # Primary resolves to a DEPRECATED object
  ConsumerProfile(type=AI_AGENT)
  BudgetSpec(total_tokens=1000)
  Expected: pack contains status_warning, not AssemblyError
```

---

## Cross-References

- CAE certification → `26-CAE-CERTIFICATION`
- CAE design invariants → `01-CAE-OVERVIEW`
- All rule definitions → docs 02–25
- CAE freeze → `28-CAE-FREEZE`
