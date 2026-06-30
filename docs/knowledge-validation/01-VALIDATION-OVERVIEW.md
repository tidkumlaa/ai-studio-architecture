# KVF-DOC-001 — Validation Overview

**Phase:** 3.0D.1.5
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

This document defines the 7 validation domains, the overall framework
structure, the output schema, and the relationship between validation
and the KOS architecture that is being validated.

---

## The Validation Problem

KOS specifies how knowledge should be structured, indexed, compressed,
and assembled into context. A Runtime that claims to implement KOS can
fail in many ways:

```
Failure Mode 1: Schema violations
  Objects missing required fields; invalid states; broken invariants.

Failure Mode 2: Context degradation
  Context produced is too large, too compressed, or irrelevant.

Failure Mode 3: Hallucination amplification
  AI using KOS context produces more errors than without it.

Failure Mode 4: Coverage gaps
  Tasks requiring object X fail because X is never retrieved.

Failure Mode 5: Search regression
  Search finds the wrong objects; correct objects ranked too low.

Failure Mode 6: Performance failure
  P99 latency breaches SLO; memory grows unboundedly at scale.

Failure Mode 7: Cross-agent inconsistency
  Agent A and Agent B disagree about the same KOS object.
```

The KVF detects all 7 failure modes with reproducible, machine-readable evidence.

---

## 7 Validation Domains

```
Domain 1: CONFORMANCE
  Does the implementation conform to the KOS schema and rules?
  Checks: KnowledgeObject fields, invariants, state machines
  Documents: 02, 03

Domain 2: CONTEXT QUALITY
  Does the implementation produce high-quality context?
  Checks: relevance, completeness, safety, efficiency, correctness
  Documents: 04, 05, 06, 11

Domain 3: KNOWLEDGE COVERAGE
  Does the implementation retrieve what tasks require?
  Checks: coverage %, missing objects, extra objects
  Documents: 07, 14, 15, 16, 17

Domain 4: SEARCH AND REASONING
  Does search find the right objects? Does reasoning use them correctly?
  Checks: Top-1/5/MRR/NDCG; hallucination rate; reasoning depth
  Documents: 08, 09, 10, 18

Domain 5: AI CAPABILITY
  Can a new AI using only KOS complete real tasks?
  Checks: task success rate; consistency; cold-start; long context
  Documents: 19, 20, 21, 22

Domain 6: PERFORMANCE AND SCALE
  Does the implementation meet SLOs at all scales?
  Checks: latency/memory/CPU at 1K/10K/100K/1M objects
  Documents: 23, 24, 25, 26

Domain 7: CERTIFICATION
  What level of KOS conformance has this implementation achieved?
  Output: Bronze / Silver / Gold / Enterprise / Research
  Documents: 27, 28, 29
```

---

## Validation Run Schema

```yaml
validation_run:
  run_id: string
  run_type: FULL | DOMAIN | REGRESSION
  started_at: string
  completed_at: string
  implementation_id: string
  implementation_version: string
  kos_schema_version: string

  domains_run: [string]               # domain names from list above

  results:
    conformance: ConformanceResult | null
    context_quality: ContextQualityResult | null
    knowledge_coverage: CoverageResult | null
    search_reasoning: SearchReasoningResult | null
    ai_capability: AICapabilityResult | null
    performance: PerformanceResult | null
    certification: CertificationResult | null

  summary:
    overall_score: float               # 0.0–1.0
    certification_level: string        # NONE | BRONZE | SILVER | GOLD | ENTERPRISE | RESEARCH
    domains_passed: integer
    domains_failed: integer
    critical_failures: [string]        # domain names with critical failures

  output_formats:                      # what was emitted
    - format: MARKDOWN
      path: string
    - format: JSON
      path: string
    - format: CSV
      path: string
    - format: HTML
      path: string
```

---

## Validation Levels

```
FULL:
  All 7 domains
  All 32 document checks
  All 140 rules
  Takes: 1–4 hours depending on corpus size

DOMAIN:
  One domain only (specified in run_type_config)
  Used for targeted regression after a specific change
  Takes: 5–30 minutes

REGRESSION:
  Compares current implementation vs. baseline snapshot
  Runs domains 4 and 6 only (search/reasoning + performance)
  Takes: 30–60 minutes

SMOKE:
  Conformance only (domain 1)
  Used in CI pipeline on every commit
  Takes: < 5 minutes
```

---

## Evidence Requirements

Every validation check must produce evidence:

```yaml
validation_evidence:
  check_id: string
  domain: string
  input_hash: string                   # hash of test input for reproducibility
  output_hash: string                  # hash of implementation output
  expected: string                     # expected value or range
  actual: string                       # actual value from implementation
  passed: boolean
  evidence_artifact: string | null     # path to detailed artifact (JSONL, CSV, etc.)
```

Evidence is mandatory — "passed" without artifact is not accepted for
Gold, Enterprise, or Research certification.

---

## Canonical Test Subject

The canonical validation object throughout this framework is:

```
KNW-PLT-MOD-001 — Quota Manager
Namespace: platform
Package: platform.core
AIRS: 0.91 (AI-NATIVE)
```

All worked examples in KVF documents use this object.

---

## Rules

| Rule | Statement |
|------|-----------|
| KVF-001 | Every validation check must produce machine-readable evidence |
| KVF-002 | A FULL run must execute all 7 domains — partial runs are not FULL |
| KVF-003 | Certification level cannot exceed what the evidence supports |
| KVF-004 | All evidence artifacts must be reproducible given the same implementation + corpus |
| KVF-005 | Critical failures in any domain block advancement past BRONZE |

---

## Cross-References

- KOS architecture total → `architecture/docs/context-assembly-engine/28-CAE-FREEZE.md`
- Certification scoring → `27-CERTIFICATION-SCORING`
- Architecture freeze → `29-ARCHITECTURE-FREEZE`
