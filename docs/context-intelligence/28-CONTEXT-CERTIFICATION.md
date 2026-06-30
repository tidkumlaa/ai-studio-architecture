# UICE-DOC-028 — Context Certification

**Phase:** 3.0D.2
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Context Certification defines the formal process for certifying a UICE
implementation. Four levels: UICE-PROTO, UICE-STANDARD, UICE-ADVANCED, and
UICE-RESEARCH. Certification is based on benchmark results, compliance checks,
and operational evidence.

---

## 4 Certification Levels

```
UICE-PROTO:
  Minimum viable UICE — basic FLA + QAGB working.
  Suitable for: development, internal testing.
  Does not require benchmark run.

UICE-STANDARD:
  All 30 UICE modules implemented and verified.
  STANDARD benchmark passed (1,000 queries).
  Suitable for: team production use, non-critical applications.

UICE-ADVANCED:
  LARGE benchmark passed (10,000 queries).
  All 50 certification checks passed.
  Conversation savings > 90% demonstrated.
  Suitable for: company-wide production, external products.

UICE-RESEARCH:
  FULL benchmark passed (50,000 queries).
  All 50 checks passed with RESEARCH-level thresholds.
  Hallucination reduction > 30% independently verified.
  Suitable for: AI training data, academic benchmarks.
```

---

## 50 Certification Checks (UICE-CC-01 through UICE-CC-50)

### Domain 1: Pipeline Completeness (UICE-CC-01 through UICE-CC-10)

| Check | Requirement | PROTO | STANDARD | ADVANCED | RESEARCH |
|-------|-------------|-------|----------|----------|----------|
| UICE-CC-01 | Intent Analyzer returns IntentProfile | YES | YES | YES | YES |
| UICE-CC-02 | Query Classifier returns QueryProfile | YES | YES | YES | YES |
| UICE-CC-03 | FLA FIELD_MATRIX has 120 combinations | YES | YES | YES | YES |
| UICE-CC-04 | L3 floor enforced for every primary | YES | YES | YES | YES |
| UICE-CC-05 | Shared Context Optimizer triggers at ≥ 2 objects same namespace | — | YES | YES | YES |
| UICE-CC-06 | Delta Generator active with Conversation Memory | — | YES | YES | YES |
| UICE-CC-07 | Deduplicator eliminates 100% TYPE-2 duplicates | — | YES | YES | YES |
| UICE-CC-08 | All 6 model families have ModelProfile | — | YES | YES | YES |
| UICE-CC-09 | Semantic Cache lookup before graph traversal | — | YES | YES | YES |
| UICE-CC-10 | Context Verifier runs 20 checks on every pack | — | YES | YES | YES |

### Domain 2: Token Efficiency (UICE-CC-11 through UICE-CC-20)

| Check | Requirement | PROTO | STANDARD | ADVANCED | RESEARCH |
|-------|-------------|-------|----------|----------|----------|
| UICE-CC-11 | token_reduction_pct ≥ 90% | YES | YES | YES | YES |
| UICE-CC-12 | token_reduction_pct ≥ 97% | — | YES | YES | YES |
| UICE-CC-13 | token_reduction_pct ≥ 99% | — | — | YES | YES |
| UICE-CC-14 | Shared context savings > 50% | — | YES | YES | YES |
| UICE-CC-15 | Conversation savings > 90% | — | — | YES | YES |
| UICE-CC-16 | Duplicate elimination 100% (TYPE-2) | — | YES | YES | YES |
| UICE-CC-17 | Field precision > 95% | — | YES | YES | YES |
| UICE-CC-18 | Field precision > 99% | — | — | YES | YES |
| UICE-CC-19 | OMIT mode active (conversation queries 2+) | — | YES | YES | YES |
| UICE-CC-20 | Budget never exceeded (0 violations) | YES | YES | YES | YES |

### Domain 3: Quality (UICE-CC-21 through UICE-CC-30)

| Check | Requirement | PROTO | STANDARD | ADVANCED | RESEARCH |
|-------|-------------|-------|----------|----------|----------|
| UICE-CC-21 | quality_score_mean ≥ 0.80 | YES | YES | YES | YES |
| UICE-CC-22 | quality_score_mean ≥ 0.85 | — | YES | YES | YES |
| UICE-CC-23 | quality_score_mean ≥ 0.90 | — | — | YES | YES |
| UICE-CC-24 | Safety dimension ≥ 0.95 | YES | YES | YES | YES |
| UICE-CC-25 | Hallucination reduction > 20% vs CAE | — | YES | YES | YES |
| UICE-CC-26 | Hallucination reduction > 30% vs CAE | — | — | YES | YES |
| UICE-CC-27 | Type A hallucination rate < 2% | — | YES | YES | YES |
| UICE-CC-28 | Type A hallucination rate < 1% | — | — | YES | YES |
| UICE-CC-29 | Guard block active for 100% of PromptPacks | YES | YES | YES | YES |
| UICE-CC-30 | Adaptive guard verified (intent-specific warnings) | — | YES | YES | YES |

### Domain 4: Performance (UICE-CC-31 through UICE-CC-40)

| Check | Requirement | PROTO | STANDARD | ADVANCED | RESEARCH |
|-------|-------------|-------|----------|----------|----------|
| UICE-CC-31 | compilation_p99_ms < 120ms (CAE level) | YES | YES | YES | YES |
| UICE-CC-32 | compilation_p99_ms < 60ms (UICE target) | — | YES | YES | YES |
| UICE-CC-33 | Cache hit rate ≥ 60% | — | YES | YES | YES |
| UICE-CC-34 | Cache hit rate ≥ 85% | — | — | YES | YES |
| UICE-CC-35 | Semantic cache hit rate ≥ 40% | — | — | YES | YES |
| UICE-CC-36 | Memory leak = none | — | YES | YES | YES |
| UICE-CC-37 | 1-hour stability at 100 rps | — | — | YES | YES |
| UICE-CC-38 | 10K object scale tested | — | — | YES | YES |
| UICE-CC-39 | Cold start < 300s to 85% cache hit rate | — | — | YES | YES |
| UICE-CC-40 | Profiler overhead < 2ms per pack | — | YES | YES | YES |

### Domain 5: Compliance (UICE-CC-41 through UICE-CC-50)

| Check | Requirement | PROTO | STANDARD | ADVANCED | RESEARCH |
|-------|-------------|-------|----------|----------|----------|
| UICE-CC-41 | All CAE CI-01 through CI-10 compliant | YES | YES | YES | YES |
| UICE-CC-42 | All UICE UI-01 through UI-10 compliant | YES | YES | YES | YES |
| UICE-CC-43 | Learning proposals require human approval | — | YES | YES | YES |
| UICE-CC-44 | 50 UICE metrics tracked and reported | — | YES | YES | YES |
| UICE-CC-45 | ProfileReport generated for every pack | — | YES | YES | YES |
| UICE-CC-46 | Context Evolution invalidation < 5s delay | — | — | YES | YES |
| UICE-CC-47 | Benchmark run documented and archived | — | YES | YES | YES |
| UICE-CC-48 | Hallucination measurement independently verified | — | — | YES | YES |
| UICE-CC-49 | FULL benchmark (50K queries) completed | — | — | — | YES |
| UICE-CC-50 | All 50 checks at RESEARCH thresholds passed | — | — | — | YES |

---

## Certification Score Formula

```
certification_score =
  pipeline_completeness  × 0.20 +
  token_efficiency       × 0.30 +
  quality                × 0.25 +
  performance            × 0.15 +
  compliance             × 0.10

Minimum score per level:
  UICE-PROTO:    ≥ 0.50
  UICE-STANDARD: ≥ 0.70
  UICE-ADVANCED: ≥ 0.85
  UICE-RESEARCH: ≥ 0.97
```

---

## Rules

| Rule | Statement |
|------|-----------|
| UICE-136 | UICE-CC-20 (budget never exceeded) is a hard gate for all certification levels |
| UICE-137 | UICE-CC-41 and UICE-CC-42 (CAE+UICE invariants) are hard gates for all levels |
| UICE-138 | Certification validity is 1 year from issuance; re-certification required after 1 year |
| UICE-139 | UICE-RESEARCH requires independent third-party verification of hallucination reduction |
| UICE-140 | Certification level must match the lowest level met across all hard gate checks |

---

## Cross-References

- Context Benchmark → `27-CONTEXT-BENCHMARK`
- Context Metrics → `20-CONTEXT-METRICS`
- KVF Certification → Phase 3.0D.1.5 `27-CERTIFICATION-SCORING`
- Dashboard → `29-DASHBOARD`
