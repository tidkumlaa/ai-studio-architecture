# KNW-CERT-ARCH-030 — Knowledge Platform Certification — Architecture Freeze

**Phase:** 3.0D.0  
**Status:** FROZEN  
**Owner:** architecture-board  
**Version:** 1.0.0  
**Freeze date:** 2026-06-30

---

## Freeze Declaration

The Knowledge Platform Certification Suite architecture is hereby **FROZEN** as of 2026-06-30.

All 30 specification documents in `architecture/docs/knowledge-certification/` (00-README through 29-CI-INTEGRATION) are CANONICAL and immutable from this date forward.

Changes require an Architecture Board Decision (ADR) and a new phase designation.

---

## Scope of Freeze

| Area | Documents |
|------|-----------|
| Overview | 00-README, 01-CERTIFICATION-OVERVIEW |
| Functional suites | 02-SEARCH through 13-REASONING |
| Non-functional suites | 14-PERFORMANCE through 20-BACKUP |
| Data/AI suites | 21-DATASET through 23-HALLUCINATION |
| Evolution suites | 24-EVOLUTION through 26-COVERAGE |
| Scoring & reporting | 27-OVERALL-SCORING, 28-DASHBOARD, 29-CI |

---

## What Is NOT Covered by This Freeze

| Item | Phase |
|------|-------|
| kos-cert CLI implementation | Phase 3.0D |
| Dataset population (golden, regression, adversarial) | Phase 3.0D |
| Dashboard HTML implementation | Phase 3.0D |
| CI pipeline wiring | Phase 3.0D |

---

## Certification Invariants

The following invariants are established by this freeze and must not be violated in any implementation:

| # | Invariant |
|---|-----------|
| CI-01 | All domain scores ∈ [0.0, 1.0] |
| CI-02 | Domain weights sum to exactly 1.0 |
| CI-03 | Check severity is one of {CRITICAL, MAJOR, MINOR, INFO} |
| CI-04 | Every check has a unique ID and defined pass criteria |
| CI-05 | All benchmarks use fixed seed 42 unless overridden |
| CI-06 | Dataset version is pinned in every certification run |
| CI-07 | Reports are self-contained (no external deps) |
| CI-08 | Exit codes are stable (code 0 = pass, 1 = critical fail) |
| CI-09 | Hallucination definition is binary (not subjective) |
| CI-10 | Level gates are strictly numeric (no subjective override) |

---

## Key Numbers (Frozen)

| Metric | Value |
|--------|-------|
| Certification domains | 11 |
| Architecture documents | 32 |
| Certification levels | 5 (Bronze / Silver / Gold / Enterprise / Research) |
| Total named checks | 312+ |
| Search query sizes | 4 (500 / 1K / 5K / 10K) |
| Reasoning question sizes | 3 (500 / 1K / 5K) |
| Performance dataset sizes | 4 (1K / 10K / 100K / 1M) |
| Traceability chains tested | 500 |
| Regression smoke checks | 15 |
| Regression core checks | 75 |
| CI pipeline stages | 5 |
| Report formats | 6 |
| Exit codes | 8 |
| Hallucination categories | 6 |

---

## What Phase 3.0D Must Implement

Implementations in Phase 3.0D must provide:

1. `kos-cert verify` — full suite runner
2. `kos-cert benchmark` — performance benchmarks
3. `kos-cert search` — search domain runner
4. `kos-cert registry` — registry domain runner
5. `kos-cert graph` — graph domain runner
6. `kos-cert reasoning` — reasoning domain runner
7. `kos-cert report` — report generator (all 6 formats)
8. `kos-cert dashboard` — dashboard generator + viewer
9. `kos-cert evolution` — evolution comparison
10. `kos-cert regression` — regression suite runner

All commands must:
- Accept `--seed N` parameter
- Accept `--output path` parameter
- Produce machine-readable JSON output
- Return correct exit codes (0–7)
- Run reproducibly across environments

---

## Approval

| Role | Name | Date |
|------|------|------|
| Architecture Board | architecture-board | 2026-06-30 |
| Phase Author | Jessadaporn Jampakaew | 2026-06-30 |

---

## Next Phase

**Phase 3.0D — Knowledge Runtime + Certification Implementation**

Phase 3.0D implements:
1. Identity Engine + Registry Engine (deferred from 3.0D.1)
2. `kos-cert` certification tool (this specification)
3. Dataset population (golden, regression, adversarial)
4. Knowledge Graph engine

Phase 3.0D may not begin before this certification freeze is committed.

---

## Cross-References

- Phase 3.0C freeze → `architecture/docs/knowledge-core/40-ARCHITECTURE-FREEZE`
- Phase 3.0C.5 freeze → `architecture/docs/knowledge-ecosystem/30-KNOWLEDGE-FREEZE`
- Document index → `index.yaml`
- Navigation → `00-README`
