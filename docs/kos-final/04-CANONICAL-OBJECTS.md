# KNW-FINAL-004 — Canonical Object Library

**Phase:** 3.0D.0.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Provides GOOD, BAD, and WHY examples for every KnowledgeObjectType. Every type has an authoritative canonical example and an anti-pattern with explanation.

---

## MODULE — GOOD Example

```yaml
knowledge_id: KNW-PLT-MOD-001
object_type: module
canonical_name: plt.module.quota-manager
name: Quota Manager
version: "1.0.0"
namespace: plt
domain: PLATFORM
lifecycle:
  state: CANONICAL
  version: "1.0.0"
  created_at: "2026-06-30"
  updated_at: "2026-06-30"
ownership:
  owner: team:platform
  team: team:platform
description: >
  Manages per-session and per-day resource consumption quotas
  across all AI providers. Enforces hard limits and emits
  QUOTAExceeded events when thresholds are crossed.
content:
  module_type: CORE
  responsibilities:
    - Enforce per-session token quotas
    - Track daily aggregate usage
    - Emit QuotaExceeded events
  interfaces:
    - IQuotaManager
  depends_on:
    - KNW-RT-RT-001   # AI Runtime
traceability:
  satisfies: [KNW-PLT-REQ-001]
  tested_by: [KNW-TEST-TST-001]
  benchmarked_by: [KNW-TEST-BENCH-001]
evidence:
  items:
    - evidence_type: EV-DOC
      source_id: KNW-ARCH-ARCH-001
      description: "Defined in architecture/docs/knowledge-core"
      weight: 0.60
      freshness_score: 1.0
tags: [quota, resource-management, platform-core]
```

---

## MODULE — BAD Example

```yaml
# BAD: Missing required fields, vague description, no traceability
knowledge_id: mod-001   # WRONG: does not follow KNW-{DOMAIN}-{TYPE}-{NNN} format
object_type: module
name: quota
# MISSING: canonical_name, namespace, domain, version, owner, description, traceability, evidence
```

**WHY this is wrong:**
- `knowledge_id` format violates KNW ID regex
- No `canonical_name` — cannot be discovered
- No `owner` — cannot be maintained
- No `traceability.satisfies` — requirement cannot be traced
- No `evidence` — quality score will be 0 for QD-3

---

## REQUIREMENT — GOOD Example

```yaml
knowledge_id: KNW-PLT-REQ-001
object_type: requirement
canonical_name: plt.requirement.quota-enforcement
name: Quota Enforcement Requirement
version: "1.0.0"
namespace: plt
domain: PLATFORM
lifecycle:
  state: CANONICAL
content:
  requirement_id: REQ-PLT-001
  requirement_text: >
    The platform MUST enforce per-session token quotas across all
    AI provider calls. Enforcement MUST be synchronous and MUST
    prevent calls that would exceed the quota.
  priority: MUST
  category: FUNCTIONAL
  acceptance_criteria:
    - QuotaManager returns QuotaExceeded when session limit reached
    - No AI provider call succeeds after quota exhausted
traceability:
  tested_by: [KNW-TEST-TST-001]
```

---

## ALGORITHM — GOOD Example

```yaml
knowledge_id: KNW-ALG-ALG-007
object_type: algorithm
canonical_name: alg.algorithm.bm25-text-ranker
name: BM25 Text Ranker
namespace: alg
domain: ALGORITHM
lifecycle:
  state: CANONICAL
content:
  algorithm_type: RANKING
  complexity_time: "O(N)"
  complexity_space: "O(N)"
  inputs: [query_terms, document_corpus, k1_param, b_param]
  outputs: [ranked_documents, bm25_scores]
  parameters:
    k1: {default: 1.5, range: [0.5, 2.0]}
    b: {default: 0.75, range: [0.0, 1.0]}
  formula: |
    score(D,Q) = Σ IDF(qi) × (f(qi,D)×(k1+1)) / (f(qi,D) + k1×(1-b+b×|D|/avgdl))
traceability:
  benchmarked_by: [KNW-TEST-BENCH-007]
evidence:
  items:
    - evidence_type: EV-RESEARCH
      description: "Robertson & Zaragoza 2009 — Probabilistic Relevance Framework"
      weight: 0.65
      freshness_score: 0.9
```

---

## DECISION — GOOD Example (ADR)

```yaml
knowledge_id: KNW-META-DEC-001
object_type: decision
canonical_name: meta.decision.use-python-runtime
name: Use Python for Knowledge Runtime
namespace: meta
domain: META
lifecycle:
  state: CANONICAL
content:
  decision_id: ADR-001
  status: ACCEPTED
  context: >
    We need a language for the Knowledge Runtime. Options are
    Python, Go, and Rust.
  decision: >
    Use Python 3.13 with Pydantic v2 for the Knowledge Runtime.
  rationale: >
    Team expertise, Pydantic v2 type safety, and the existing
    platform codebase in Python.
  consequences:
    positive:
      - Type-safe models with Pydantic v2
      - Fast iteration
    negative:
      - GIL limits true parallelism (mitigated by async)
  alternatives_considered:
    - Go: better performance, less team expertise
    - Rust: best performance, highest learning curve
traceability:
  satisfies: [KNW-META-REQ-001]
```

---

## TEST — GOOD Example

```yaml
knowledge_id: KNW-TEST-TST-001
object_type: test
canonical_name: test.test.quota-enforcement-test
name: Quota Enforcement Test
namespace: test
lifecycle:
  state: CANONICAL
content:
  test_type: INTEGRATION
  target: KNW-PLT-MOD-001
  test_cases:
    - id: TC-001
      description: "Quota exceeded returns error"
      given: "session with 100 token limit"
      when: "101st token requested"
      then: "QuotaExceededError raised"
    - id: TC-002
      description: "Quota reset on new session"
      given: "expired session with full quota"
      when: "new session created"
      then: "quota counter = 0"
traceability:
  tests: [KNW-PLT-MOD-001]
  satisfies: [KNW-PLT-REQ-001]
```

---

## BENCHMARK — GOOD Example

```yaml
knowledge_id: KNW-TEST-BENCH-001
object_type: benchmark
canonical_name: test.benchmark.identity-engine-perf
name: Identity Engine Performance Benchmark
namespace: test
lifecycle:
  state: CANONICAL
content:
  benchmark_type: PERFORMANCE
  target: KNW-RT-ENGINE-001
  dataset_size: 100000
  operations:
    - operation: generate_id
      iterations: 100000
      p50_ms: 0.1
      p99_ms: 0.5
    - operation: compute_checksum
      iterations: 100000
      p50_ms: 0.8
      p99_ms: 2.0
  pass_criteria:
    generate_id_p99_ms: 1.0
    compute_checksum_p99_ms: 5.0
traceability:
  tests: [KNW-TEST-TST-001]
```

---

## Anti-Pattern Summary

| Anti-Pattern | Rule Violated | Fix |
|-------------|---------------|-----|
| ID without KNW prefix | KL-001 | Use `KNW-{DOMAIN}-{TYPE}-{NNN}` |
| Missing owner on CANONICAL | KL-012 | Add `ownership.owner: team:X` |
| No traceability on MODULE | TR-001 | Add `traceability.satisfies` |
| No evidence on CANONICAL | CS-001 | Add ≥ 1 evidence item |
| Description > 500 words | SG-D-01 | Trim to essential meaning |
| Duplicate canonical_name | MC-005 | Rename with unique slug |
| lifecycle.state not in enum | MC-011 | Use one of 6 valid states |

---

## Cross-References

- All 33 type schemas → Phase 3.0C.5 `10-KNOWLEDGE-SCHEMAS`
- Lint rules → Phase 3.0C.5 `06-KNOWLEDGE-LINTER`
- Style guide → Phase 3.0C.5 `04-KNOWLEDGE-STYLE-GUIDE`
- Full examples (5 types) → Phase 3.0C.5 `11-KNOWLEDGE-EXAMPLES`
