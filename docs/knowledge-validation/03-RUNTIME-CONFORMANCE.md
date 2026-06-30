# KVF-DOC-003 — Runtime Conformance

**Phase:** 3.0D.1.5
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Runtime Conformance validates that a KOS Runtime implementation satisfies the
behavioral contracts defined in the KOS architecture. It does not validate the
KnowledgeObjects themselves (that is Knowledge Conformance, doc 02) — it
validates the behavior of the system that processes them.

---

## What Runtime Conformance Covers

```
RCG-1: Context Assembly Conformance
  Does the Runtime's CAE implementation follow the CAE specification?

RCG-2: Index Conformance
  Does the Runtime's index match the KIL index specification?

RCG-3: Query Resolution Conformance
  Does intent parsing produce correct results?

RCG-4: Budget Enforcement Conformance
  Does the Runtime enforce token budgets correctly?

RCG-5: Guard Block Conformance
  Does every PromptPack include a correct guard block?

RCG-6: Compression Conformance
  Does the Runtime produce correct compression at each level?

RCG-7: Validation Conformance
  Does the Runtime's validator reject invalid packs?

RCG-8: API Conformance
  Does the Runtime's API match the CAE API specification (doc 25)?
```

---

## RCG-1: Context Assembly Conformance

```
Test: For each of 1,000 test queries:

  Run the Runtime's assemble() with a known (Query, Consumer, Budget).
  Run the Reference Assembler with the same inputs.

  Compare:
    primary_object_id: must match (identical)
    pack_type: must match
    primary.compression_level: must be >= L3
    total_tokens: must be <= budget_declared
    guard_block.present: must be true for PromptPack

  RC-1.01: primary_object_id match rate >= 99%
  RC-1.02: pack_type match rate = 100%
  RC-1.03: primary compression never below L3 = 100%
  RC-1.04: total_tokens <= budget in 100% of packs
  RC-1.05: guard_block present in 100% of PromptPacks
```

---

## RCG-2: Index Conformance

```
Test: Query the Runtime's KIL index for 1,000 known object IDs.

  RC-2.01: Hit rate for CANONICAL objects = 100%
  RC-2.02: Hit rate for ACTIVE objects >= 99.9%
  RC-2.03: Returned quality_score matches object metadata.quality_score
  RC-2.04: Returned ai_readiness_score matches object intelligence.ai_readiness_score
  RC-2.05: Index lookup P99 latency < 5ms
```

---

## RCG-3: Query Resolution Conformance

```
Test: Submit 500 queries with known correct intent type.

  RC-3.01: Intent classification accuracy >= 95%
  RC-3.02: Entity resolution accuracy (entity is in top-3 candidates) >= 99%
  RC-3.03: Fast-path (keyword) resolution rate >= 30% of queries
  RC-3.04: AMBIGUOUS resolution triggers disambiguation, not silent guess
  RC-3.05: Unknown entity raises ENTITY_NOT_FOUND, not empty pack
```

---

## RCG-4: Budget Enforcement Conformance

```
Test: Submit 500 queries with tight budgets (NANO=100 and MICRO=300).

  RC-4.01: No emitted pack exceeds declared budget
  RC-4.02: Primary object always at L3 or above even when budget=100
  RC-4.03: When budget too small for minimum assembly, AssemblyError raised
  RC-4.04: Guard block minimum is max(budget×0.10, 50) tokens
  RC-4.05: Dropped objects are recorded in allocation_plan.dropped_objects
```

---

## RCG-5: Guard Block Conformance

```
Test: Inspect guard_block of 500 PromptPacks.

  RC-5.01: scope_boundaries is non-empty list in 100% of packs
  RC-5.02: confidence_hedge present in 100% of packs
  RC-5.03: confidence_hedge text matches object's confidence level
  RC-5.04: Deprecated primary triggers status_warning in 100% of cases
  RC-5.05: typical_mistakes list present (may be empty only if cortex absent)
```

---

## RCG-6: Compression Conformance

```
Test: Request rendering of 100 objects at each compression level.

  RC-6.01: L1 outputs have token count <= 15 for all 100 objects
  RC-6.02: L2 outputs have token count <= 50 for all 100 objects
  RC-6.03: L3 outputs have token count <= 200 for all 100 objects
  RC-6.04: L4 outputs have token count <= 500 for all 100 objects
  RC-6.05: L5 machine outputs are valid JSON for all 100 objects
```

---

## RCG-7: Validation Conformance

```
Test: Submit 50 deliberately invalid packs (one per validation check V-01..V-20)
and verify that each is rejected with the correct error code.

  RC-7.01: All 20 validation checks raise AssemblyError when violated
  RC-7.02: Error code in AssemblyError matches the violated check's code
  RC-7.03: No invalid pack passes validation
  RC-7.04: Validator rejects quality < 0.70 packs (V-15) — 0 false passes
  RC-7.05: Valid packs always pass — 0 false rejections on 1,000 valid packs
```

---

## RCG-8: API Conformance

```
Test: Exercise the CAE API (from doc 25) with conformance test suite.

  RC-8.01: AssemblyRequest with all fields returns valid AssemblyResponse
  RC-8.02: All 6 error codes can be triggered and are returned correctly
  RC-8.03: request_id is echoed in success and error responses
  RC-8.04: Feedback API accepts EXPLICIT and IMPLICIT sources
  RC-8.05: API version header X-CAE-API-Version: v1 present on all responses
```

---

## Runtime Conformance Result Schema

```yaml
runtime_conformance_result:
  total_checks: integer                # 40 RC checks
  checks_passed: integer
  checks_failed: integer
  critical_failures: [string]          # check IDs from RCG-1, RCG-4

  by_group:
    - group: string
      checks_passed: integer
      checks_failed: integer

  conformance_score: float
  api_conformant: boolean
  budget_enforcement_conformant: boolean
  guard_block_conformant: boolean
```

---

## Runtime Conformance Score Thresholds

| Score | Disposition |
|-------|-------------|
| 40/40 | FULLY CONFORMANT |
| 37–39 | MOSTLY CONFORMANT (eligible Silver+) |
| 34–36 | PARTIALLY CONFORMANT (eligible Bronze) |
| < 34 | NON-CONFORMANT (certification blocked) |

---

## Rules

| Rule | Statement |
|------|-----------|
| KVF-011 | RCG-4 budget enforcement must be tested with NANO budget — smallest real case |
| KVF-012 | RC-7.05 (0 false rejections) is mandatory for Gold+ certification |
| KVF-013 | RC-6.05 (L5 valid JSON) must use strict parser — lenient pass is disallowed |
| KVF-014 | RCG-1 comparison must use a Reference Assembler, not the same implementation |
| KVF-015 | Runtime conformance must be re-run after any semver change to the implementation |

---

## Cross-References

- Knowledge conformance → `02-KNOWLEDGE-CONFORMANCE`
- CAE rules → Phase 3.0D.1 `25-CAE-API`
- Validator spec → Phase 3.0D.1 `16-CONTEXT-VALIDATOR`
- Certification scoring → `27-CERTIFICATION-SCORING`
