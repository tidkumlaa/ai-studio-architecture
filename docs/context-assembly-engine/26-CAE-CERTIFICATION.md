# CAE-DOC-026 — CAE Certification Framework

**Phase:** 3.0D.1
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

CAE Certification defines the 50 checks required before any implementation of
the Context Assembly Engine may be declared production-ready. Certification is
required before the engine handles live traffic.

---

## Certification Levels

```
Level 1 — PROTO:     ≥ 30 of 50 checks pass; no CRITICAL failures
Level 2 — BETA:      ≥ 40 of 50 checks pass; no CRITICAL failures
Level 3 — RC:        ≥ 47 of 50 checks pass; no CRITICAL failures
Level 4 — CERTIFIED: all 50 checks pass

CRITICAL checks: CC-01 through CC-20
  All CRITICAL checks must pass for any level ≥ PROTO
```

---

## Certification Checks

### Domain A — Pipeline Correctness (CC-01 to CC-10) [CRITICAL]

| Check | Requirement |
|-------|-------------|
| CC-01 | S1 resolves intent for all 12 INT types with ≥ 95% accuracy on test corpus |
| CC-02 | S2 selector returns primary entity within top-3 candidates for IDENTITY queries |
| CC-03 | S3 relevance score differences of < 0.01 between runs on identical input = 0 |
| CC-04 | S4 budget allocation sums to exactly total_budget (no rounding errors) |
| CC-05 | S5 always assigns L3 or above to primary — verified on 1,000 random queries |
| CC-06 | S6 planning produces valid ordered sections (prerequisites before dependents) |
| CC-07 | S7 assembler never produces packs exceeding budget by more than 5% |
| CC-08 | S8 AH Guard runs on every PromptPack — 0 packs emitted without guard block |
| CC-09 | S9 validator rejects all packs with quality score < 0.70 — 0 false passes |
| CC-10 | E2E pipeline executes stages in fixed order S1→S2→…→S9 — verified via trace |

---

### Domain B — Performance (CC-11 to CC-20) [CRITICAL]

| Check | Requirement |
|-------|-------------|
| CC-11 | P99 E2E latency < 120ms under load test (1,000 concurrent requests) |
| CC-12 | P50 E2E latency < 50ms under standard load (100 rps) |
| CC-13 | Cache hit rate ≥ 85% after warm-up (1,000 queries) |
| CC-14 | Cache invalidation < 100ms after index update — verified 100 times |
| CC-15 | Prefetch tasks do not add > 5ms to pack emission latency |
| CC-16 | Degradation Level 1 activates within 10s of P99 > 100ms — verified |
| CC-17 | No blocking I/O (DB transactions, network) in S1–S9 — static analysis required |
| CC-18 | No LLM inference inside the pipeline — static analysis required |
| CC-19 | Memory usage stable under 1-hour sustained load — no memory leak |
| CC-20 | Stage budget violations logged and counted — verified: 0 silent violations |

---

### Domain C — Safety and Hallucination Prevention (CC-21 to CC-30)

| Check | Requirement |
|-------|-------------|
| CC-21 | PromptPack with object AIRS < 0.60 raises AssemblyError — 0 exceptions |
| CC-22 | Guard block always contains scope_boundaries — verified on 500 PromptPacks |
| CC-23 | Guard block confidence_hedge matches object's confidence level — 100% match |
| CC-24 | Deprecated primary triggers status_warning — 100% of test cases |
| CC-25 | Missing required context declared in guard block — verified on corpus |
| CC-26 | context_confidence = min (not mean) — algebraic property test |
| CC-27 | ANTI_CONFUSION objects labeled with "Do NOT confuse" — 100% of inclusions |
| CC-28 | SearchPack does not include guard block — verified on 200 SearchPack samples |
| CC-29 | Hallucination flag reaches maintenance queue within 1 hour — e2e test |
| CC-30 | Guard block render is idempotent — same guard for same inputs |

---

### Domain D — Output Pack Correctness (CC-31 to CC-40)

| Check | Requirement |
|-------|-------------|
| CC-31 | PromptPack pack_id is unique UUID — 0 collisions in 10,000 assemblies |
| CC-32 | HumanPack uses audience-specific explanation block — 100% of HUMAN packs |
| CC-33 | HumanPack fallback to developer explanation when audience block missing — verified |
| CC-34 | OperatorPack failure modes ordered by severity (CRITICAL first) — 100% |
| CC-35 | SearchPack result count ≤ 10 — 0 exceptions |
| CC-36 | SearchPack empty_reason set when results is empty — 100% |
| CC-37 | All pack content is valid UTF-8 — verified on corpus with non-ASCII objects |
| CC-38 | L5 machine content is valid JSON — parse succeeds on all L5 outputs |
| CC-39 | Quality report included in all emitted packs — 100% |
| CC-40 | budget_utilization ≤ 1.00 in all packs — 0 over-budget emissions |

---

### Domain E — API and Contract (CC-41 to CC-50)

| Check | Requirement |
|-------|-------------|
| CC-41 | assemble() returns AssemblyError with correct error_code for all 6 error types |
| CC-42 | request_id echoed in all responses (success and error) — 100% |
| CC-43 | Feedback API accepts EXPLICIT and IMPLICIT types — both functional |
| CC-44 | Feedback records are append-only — no update or delete accepted |
| CC-45 | Search API applies min_quality filter correctly — verified |
| CC-46 | API version header present on all responses — 100% |
| CC-47 | idempotency: same (query, consumer, budget) produces same pack structure — verified |
| CC-48 | Context metrics (all 30) observable with correct values — spot-check 10 |
| CC-49 | Degradation protocol activates at correct thresholds — integration test |
| CC-50 | All 120 CAE rules pass their corresponding test cases (from doc 27) |

---

## Certification Report Schema

```yaml
cae_certification_report:
  report_id: string
  assessed_at: string
  implementation_version: string
  certification_level: PROTO | BETA | RC | CERTIFIED | FAILED

  summary:
    total_checks: 50
    passed: integer
    failed: integer
    critical_failed: integer

  results:
    - check_id: string
      domain: string
      passed: boolean
      critical: boolean
      evidence: string
      failure_reason: string | null

  certifier: string
  next_review: string | null         # date of next required recertification
```

---

## Recertification

```
Recertification required when:
  - CAE version bumped (any semver change)
  - New pack type added
  - Guard block rules modified
  - Performance model targets changed

Recertification scope:
  - Run all 50 checks
  - Full CERTIFIED level required before production deployment
```

---

## Rules

| Rule | Statement |
|------|-----------|
| N/A — Certification is external | All 50 checks are tested in doc 27 |

---

## Cross-References

- CAE testing → `27-CAE-TESTING`
- CAE freeze → `28-CAE-FREEZE`
- All pipeline stages → docs 01–10
- All pack builders → docs 11–14
- Performance model → `21-PERFORMANCE-MODEL`
