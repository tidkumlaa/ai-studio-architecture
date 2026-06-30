# KNW-KE-ARCH-011 — Knowledge Examples

**Phase:** 3.0C.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Canonical worked examples for the most important object types. Every example passes the linter and formatter. Examples live in `knowledge/examples/`.

---

## Example: MODULE

```yaml
# knowledge/examples/module-quota-manager.yaml
knowledge_id: KNW-PLT-MOD-001
object_type: module
name: Quota Manager
version: 1.0.0
status: CANONICAL
owner: team:platform
description: Manages per-session and per-day resource consumption quotas across all AI providers.
tags: [platform, quota, resource-management, infrastructure]

classification:
  domain: PLATFORM
  layer: 3
  category: INFRASTRUCTURE

identity:
  canonical_name: plt.module.quota-manager
  knowledge_uri: knw://plt/module/quota-manager@1.0.0

evidence:
  items:
    - evidence_type: EV-CODE
      source_url: "platform/ai_runtime/quota/engine.py"
      description: "Production implementation in ai_runtime"
      weight: 0.60
      freshness_score: 1.0
  evidence_score: 0.60

confidence:
  declared: 0.90
  composite: 0.85

traceability:
  satisfies: [KNW-ARCH-REQ-001]
  implements: [KNW-RT-SVC-003]
  tests: [KNW-TEST-TST-001]

lifecycle:
  review_status: APPROVED
  approved_by: team:architecture
  approved_at: "2026-06-30T00:00:00Z"

metadata:
  created_by: team:platform
  created_at: "2026-06-29T00:00:00Z"
  source_doc: "architecture/docs/knowledge-core/19-RUNTIME-REGISTRY"

module_id: module:platform:quota_manager:v1
runtime_id: ai-runtime
python_module: ai_runtime.quota.engine
is_public: true
module_status: EXISTING
```

---

## Example: REQUIREMENT

```yaml
# knowledge/examples/requirement-sub-second-lookup.yaml
knowledge_id: KNW-ARCH-REQ-001
object_type: requirement
name: Sub-second Registry Lookup
version: 1.0.0
status: CANONICAL
owner: team:architecture
description: The registry must return any object by knowledge_id in under 1 second P99 at 1M objects.
tags: [architecture, performance, registry, requirement]

classification:
  domain: ARCHITECTURE
  layer: 0
  category: NON_FUNCTIONAL

identity:
  canonical_name: arch.requirement.sub-second-registry-lookup
  knowledge_uri: knw://arch/requirement/sub-second-registry-lookup@1.0.0

evidence:
  items:
    - evidence_type: EV-HAPPROVAL
      source_url: "architecture/docs/knowledge-core/37-PERFORMANCE-BUDGET"
      description: "Defined in performance budget with P99 < 5ms for registry get by ID"
      weight: 0.80
      freshness_score: 1.0
  evidence_score: 0.80

confidence:
  declared: 0.95

traceability:
  satisfies: []
  implements: []
  tests: [KNW-TEST-TST-101]

lifecycle:
  review_status: APPROVED
  approved_by: architecture-board
  approved_at: "2026-06-30T00:00:00Z"

metadata:
  created_by: team:architecture
  created_at: "2026-06-29T00:00:00Z"
  source_doc: "architecture/docs/knowledge-core/37-PERFORMANCE-BUDGET"

requirement_id: REQ-PLT-001
priority: MUST
source: architecture-board
acceptance_criteria:
  - "P99 < 1s at 1M objects measured by benchmark suite"
  - "No fallback to full scan — index must be used"
fulfilled_by: [KNW-PLT-MOD-042]
```

---

## Example: ALGORITHM

```yaml
# knowledge/examples/algorithm-bm25.yaml
knowledge_id: KNW-ALG-ALG-007
object_type: algorithm
name: BM25 Text Ranker
version: 1.0.0
status: CANONICAL
owner: team:core
description: Ranks documents against a query using BM25 with k1=1.2, b=0.75.
tags: [algorithm, search, text-ranking, bm25]

classification:
  domain: ALGORITHM
  layer: 0
  category: SEARCH

identity:
  canonical_name: alg.algorithm.bm25-text-ranker
  knowledge_uri: knw://alg/algorithm/bm25-text-ranker@1.0.0

evidence:
  items:
    - evidence_type: EV-DOC
      source_url: "architecture/docs/knowledge-core/28-SEARCH-ENGINE"
      description: "Specified in search engine architecture"
      weight: 0.60
  evidence_score: 0.60

confidence:
  declared: 0.95

traceability:
  satisfies: []
  implements: []
  tests: [KNW-TEST-TST-200]

metadata:
  created_by: team:core
  created_at: "2026-06-30T00:00:00Z"
  source_doc: "architecture/docs/knowledge-core/36-ALGORITHMS"

algorithm_id: ALG-007
time_complexity: "O(T * log N)"
space_complexity: "O(N)"
```

---

## Example: BENCHMARK

```yaml
# knowledge/examples/benchmark-registry-lookup.yaml
knowledge_id: KNW-TEST-BENCH-001
object_type: benchmark
name: Registry Get by ID Benchmark
version: 1.0.0
status: VERIFIED
owner: team:quality
description: Measures P50 and P99 latency for registry.get() at 1M objects.
tags: [test, benchmark, registry, performance]

classification:
  domain: TEST
  layer: 0
  category: PERFORMANCE

identity:
  canonical_name: test.benchmark.registry-get-by-id-benchmark
  knowledge_uri: knw://test/benchmark/registry-get-by-id-benchmark@1.0.0

evidence:
  items: []
  evidence_score: 0.0

confidence:
  declared: 0.80

traceability:
  satisfies: [KNW-ARCH-REQ-001]
  implements: []
  tests: []

metadata:
  created_by: team:quality
  created_at: "2026-06-30T00:00:00Z"
  source_doc: "architecture/docs/knowledge-core/37-PERFORMANCE-BUDGET"

benchmark_id: BENCH-001
operation: "registry.get(knowledge_id)"
p50_ms: 1.0
p99_ms: 5.0
```

---

## Example: DECISION

```yaml
# knowledge/examples/decision-python-pydantic-v2.yaml
knowledge_id: KNW-ARCH-ADR-001
object_type: decision
name: Use Pydantic v2 for Knowledge Object Model
version: 1.0.0
status: CANONICAL
owner: team:architecture
description: Pydantic v2 is the validation and serialisation layer for all Knowledge Objects.
tags: [architecture, python, pydantic, decision]

classification:
  domain: ARCHITECTURE
  layer: 0
  category: TECHNOLOGY_CHOICE

identity:
  canonical_name: arch.decision.use-pydantic-v2-for-knowledge-object-model
  knowledge_uri: knw://arch/decision/use-pydantic-v2-for-knowledge-object-model@1.0.0

evidence:
  items:
    - evidence_type: EV-CODE
      source_url: "platform/knowledge_runtime/objects/base.py"
      description: "Implemented using pydantic BaseModel with field_validator"
      weight: 0.60
  evidence_score: 0.60

confidence:
  declared: 0.95

lifecycle:
  review_status: APPROVED
  approved_by: architecture-board
  approved_at: "2026-06-30T00:00:00Z"

metadata:
  created_by: team:architecture
  created_at: "2026-06-29T00:00:00Z"
  source_doc: "architecture/docs/knowledge-core/05-OBJECT-INHERITANCE"

decision_id: ADR-001
context: "We need a Python data validation library for all 33 Knowledge Object types."
options_considered:
  - "Pydantic v2 — fast, type-safe, JSON native"
  - "attrs + cattrs — lighter but less ecosystem integration"
  - "dataclasses — no validation built-in"
chosen_option: "Pydantic v2"
rationale: >
  Pydantic v2 provides discriminated unions, model_config, strict validation,
  and JSON serialisation with no extra code. The Python 3.13 StrEnum integration
  requires .value instead of str(), which is an acceptable trade-off.
consequences_positive:
  - "Strong type checking at runtime"
  - "JSON serialisation via model_dump(mode='json')"
consequences_negative:
  - "Namespace conflicts for fields starting with 'model_' (mitigated via protected_namespaces=())"
```

---

## Cross-References

- Templates used to create these examples → `03-KNOWLEDGE-TEMPLATES`
- Style rules followed → `04-KNOWLEDGE-STYLE-GUIDE`
- Examples must pass linter → `06-KNOWLEDGE-LINTER`
- Golden dataset built from best examples → `12-KNOWLEDGE-GOLDEN-DATASET`
