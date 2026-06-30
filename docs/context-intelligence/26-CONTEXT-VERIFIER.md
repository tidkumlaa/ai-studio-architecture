# UICE-DOC-026 — Context Verifier

**Phase:** 3.0D.2
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Context Verifier performs the final pre-emission check on every assembled
context pack. It verifies compliance with all 10 CAE design invariants
(CI-01 through CI-10) and all 10 UICE invariants (UI-01 through UI-10).
Any invariant violation causes the pack to be rejected — assembly must retry
with corrections, or the pack is emitted with an error annotation.

---

## Verification Checks (20 total)

### CAE Invariant Checks (CI-01 through CI-10)

| Check | Invariant | Verification |
|-------|-----------|-------------|
| VCI-01 | Primary always L3+ | primary.token_count ≥ L3_FLOOR_TOKENS AND fields ⊇ L3_FLOOR_FIELDS |
| VCI-02 | Budget is hard limit | pack.total_token_count ≤ constraints.max_tokens |
| VCI-03 | AH Guard always runs for PromptPack | guard is not None AND guard.token_count > 0 |
| VCI-04 | Validator must pass | quality_score ≥ 0.70 (inherited from CAE; UICE floor is 0.80) |
| VCI-05 | Stateless | verified structurally (same inputs → same outputs; checked by replay) |
| VCI-06 | context_confidence = min | pack.context_confidence == MIN(all confidence scores in pack) |
| VCI-07 | Guard always L2 | guard.token_count ≤ L2_TOKEN_LIMIT (50 tokens) |
| VCI-08 | AIRS ≥ 0.60 for PromptPack | primary_kil.ai_readiness_score ≥ 0.60 when pack_type == PROMPT_PACK |
| VCI-09 | Unique pack_id | pack.pack_id not in RECENT_PACK_IDS (last 10,000) |
| VCI-10 | Stages in order | plan.execution_log stages are in correct pipeline order |

### UICE Invariant Checks (UI-01 through UI-10)

| Check | Invariant | Verification |
|-------|-----------|-------------|
| VUI-01 | Every token justified by IntentProfile | pack.intent_profile is not None |
| VUI-02 | Delta requires Memory entry | if any(s.delta_mode for s in pack.all_slots): session_id is set AND memory.contains(s.knowledge_id) |
| VUI-03 | Model Adapter ran before rendering | pack.model_profile_id is set AND model_adapted == True |
| VUI-04 | Semantic Cache lookup before traversal | profile.cache_profile.semantic_lookup_attempted == True |
| VUI-05 | Deduplicator ran after assembly | compression_stats.dedup_pass_completed == True |
| VUI-06 | quality_score ≥ 0.80 | quality_report.quality_score ≥ 0.80 OR quality_report.quality_warning is set |
| VUI-07 | Memory session cap | memory.entry_count ≤ 10,000 (if session active) |
| VUI-08 | Guard always generated for PromptPack | same as VCI-03 (redundant guard) |
| VUI-09 | Budget Manager validated pack | budget_allocation.all_ok == True |
| VUI-10 | Learning proposals not auto-applied | no FieldWeightAdjustment with status == APPLIED was used in this assembly |

---

## Verification Algorithm

```
VERIFY(pack, quality_report, budget_allocation, profile, constraints) → VerificationResult:

  failures = []

  // Run all 20 checks
  for check in ALL_CHECKS:
    result = check.evaluate(pack, quality_report, budget_allocation, profile, constraints)
    if not result.passed:
      failures.append(VerificationFailure(
        check_id  = check.id,
        invariant = check.invariant,
        message   = result.failure_message,
        severity  = check.severity,
      ))

  critical_failures = [f for f in failures if f.severity == CRITICAL]
  warning_failures  = [f for f in failures if f.severity == WARNING]

  if critical_failures:
    return VerificationResult(
      passed            = False,
      critical_failures = critical_failures,
      warning_failures  = warning_failures,
      action            = REJECT,  // pack must not be emitted
    )

  if warning_failures:
    // Emit with warnings annotated in pack metadata
    pack.verification_warnings = [f.message for f in warning_failures]
    return VerificationResult(
      passed           = True,
      warning_failures = warning_failures,
      action           = EMIT_WITH_WARNINGS,
    )

  return VerificationResult(passed=True, action=EMIT_CLEAN)
```

---

## Severity Classification

```
CRITICAL (REJECT if failed):
  VCI-01  (primary L3+ floor)
  VCI-02  (budget not exceeded)
  VCI-03  (guard present for PromptPack)
  VCI-06  (confidence = minimum)
  VCI-08  (AIRS ≥ 0.60 for PromptPack)
  VUI-02  (delta without memory)
  VUI-09  (budget not validated)

WARNING (EMIT_WITH_WARNINGS if failed):
  All others
```

---

## VerificationResult Schema

```yaml
VerificationResult:
  pack_id: string
  passed: bool
  action: EMIT_CLEAN | EMIT_WITH_WARNINGS | REJECT

  critical_failures: list[VerificationFailure]
  warning_failures: list[VerificationFailure]

  checks_run: int          # always 20
  checks_passed: int
  verification_ms: float

  VerificationFailure:
    check_id: string       # VCI-01, VUI-02, etc.
    invariant: string
    message: string        # human-readable failure description
    severity: CRITICAL | WARNING
```

---

## Rules

| Rule | Statement |
|------|-----------|
| UICE-126 | Context Verifier runs on every pack, always — no bypass, no sampling |
| UICE-127 | CRITICAL check failures cause immediate REJECT; pack must not be emitted |
| UICE-128 | WARNING failures result in EMIT_WITH_WARNINGS; the pack is emitted but annotated |
| UICE-129 | All 20 checks must run; partial verification is INVALID |
| UICE-130 | Verification result must be recorded in ProfileReport for every pack |

---

## Cross-References

- All CAE invariants → Phase 3.0D.1 `03-CAE-INVARIANTS` through `28-CAE-FREEZE`
- UICE invariants → `README.md`
- Quality Optimizer → `25-QUALITY-OPTIMIZER`
- Context Profiler → `21-CONTEXT-PROFILER`
