# KNW-FINAL-009 — Canonical Reasoning Cases

**Phase:** 3.0D.0.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Defines 50 canonical reasoning questions with ground-truth answers. These are the authoritative cases for evaluating AI reasoning quality over the knowledge base.

---

## Identity Questions (RI-001–RI-010)

| ID | Question | Ground Truth Answer |
|----|----------|---------------------|
| RI-001 | What is KNW-PLT-MOD-001? | Quota Manager — manages per-session and per-day resource consumption quotas |
| RI-002 | What namespace does the Quota Manager belong to? | plt (PLATFORM) |
| RI-003 | Who owns KNW-PLT-MOD-001? | team:platform |
| RI-004 | What version is KNW-PLT-MOD-001? | 1.0.0 |
| RI-005 | What is the lifecycle state of BM25 Text Ranker? | CANONICAL |
| RI-006 | What is the canonical_name of KNW-ALG-ALG-007? | alg.algorithm.bm25-text-ranker |
| RI-007 | What object type is KNW-TEST-BENCH-001? | BENCHMARK |
| RI-008 | What algorithm computes SCC? | Tarjan SCC (KNW-ALG-ALG-009) |
| RI-009 | What algorithm implements BFS? | KNW-ALG-ALG-001 |
| RI-010 | What is the URI of KNW-PLT-MOD-001? | knw://plt/module/quota-manager@1.0.0 |

---

## Relationship Questions (RR-001–RR-010)

| ID | Question | Ground Truth Answer |
|----|----------|---------------------|
| RR-001 | What does Quota Manager depend on? | AI Runtime (KNW-RT-RT-001) — DEPENDS_ON, HARD |
| RR-002 | What implements Routing Engine? | AI Router Service (KNW-RT-SVC-007) — IMPLEMENTS |
| RR-003 | What tests KNW-PLT-MOD-001? | Quota Enforcement Test (KNW-TEST-TST-001) |
| RR-004 | What does AI Router use? | BM25 Text Ranker (KNW-ALG-ALG-007) — USES, optional |
| RR-005 | What is KNW-PLT-MOD-002 related to? | BM25 and TF-IDF algorithms (RELATED_TO) |
| RR-006 | What does Provider Registry provide? | Provider API (KNW-API-API-002) — PROVIDES |
| RR-007 | What extends the Base Provider? | Anthropic Provider (KNW-PROV-PROV-002) |
| RR-008 | What supersedes the legacy Router? | AI Router v2 (KNW-PLT-MOD-002) |
| RR-009 | What does the Quota Manager test? | None — Quota Manager is a MODULE, not a TEST |
| RR-010 | What requirements does KNW-PLT-MOD-001 satisfy? | KNW-PLT-REQ-001 (Quota Enforcement Requirement) |

---

## Dependency Questions (RD-001–RD-010)

| ID | Question | Ground Truth Answer |
|----|----------|---------------------|
| RD-001 | What depends on Provider Registry? | AI Runtime (KNW-RT-RT-001) depends on Provider Registry |
| RD-002 | What are the leaf packages (no dependencies)? | kos.meta.package, kos.algorithm.package |
| RD-003 | What is the dependency closure of kos.platform.package? | meta, algorithm (transitively) |
| RD-004 | Does kos.test.package depend on kos.runtime.package? | No — test → platform, meta only |
| RD-005 | Can a PLT object reference an RT object without declaring a dependency? | No — DM-003 violation |
| RD-006 | What happens if kos.meta.package is removed? | All 7 dependent packages break (DM-001) |
| RD-007 | What is the optional dependency of kos.platform.package? | kos.algorithm.package — optional |
| RD-008 | What package has zero dependencies? | kos.meta.package and kos.algorithm.package |
| RD-009 | What is the dependency order (topo sort)? | meta/alg → pattern/provider/financial → platform → api/test → runtime |
| RD-010 | What version of kos.meta.package does kos.platform.package require? | >=1.0.0,<2.0.0 |

---

## Architecture Questions (RA-001–RA-010)

| ID | Question | Ground Truth Answer |
|----|----------|---------------------|
| RA-001 | What are the 9 standard KOS packages? | platform/runtime/provider/algorithm/pattern/api/test/financial/meta |
| RA-002 | How many object types exist in KOS v1.0? | 33 |
| RA-003 | What is the KNW ID format? | KNW-{DOMAIN}-{TYPE}-{NNN} |
| RA-004 | What are the 6 lifecycle states? | DRAFT/PROPOSED/VERIFIED/CANONICAL/DEPRECATED/ARCHIVED |
| RA-005 | What quality score is required for CANONICAL? | ≥ 0.80 overall + ≥ 0.55 evidence |
| RA-006 | How many lint rules exist? | 32 (KL-001 through KL-032) |
| RA-007 | What does `kos format` guarantee? | Idempotency: format(format(x)) == format(x) |
| RA-008 | What is the minimum quality for VERIFIED? | ≥ 0.60 overall |
| RA-009 | What CI stage runs the full certification? | Merge gate (2 hours) |
| RA-010 | What is the Gold certification threshold? | Overall score ≥ 0.80 |

---

## Impact Questions (IM-001–IM-005)

| ID | Question | Ground Truth Answer |
|----|----------|---------------------|
| IM-001 | What breaks if KNW-RT-RT-001 (AI Runtime) is removed? | kos.runtime.package cannot function; all HARD dependents break |
| IM-002 | What breaks if kos.meta.package is removed? | All 7 other packages break (diamond dependency) |
| IM-003 | What breaks if BM25 ranker is removed? | AI Router degrades to fallback (SOFT dependency — graceful) |
| IM-004 | What breaks if Provider Registry is deprecated? | AI Runtime loses provider resolution; fallback required |
| IM-005 | What is the minimum object set needed to run the platform? | kos.meta.package + kos.platform.package + kos.runtime.package + kos.provider.package |

---

## Hallucination-Prone Questions (Non-answerable)

| ID | Question | Correct Behaviour |
|----|----------|-------------------|
| HAL-001 | What is KNW-PLT-MOD-999? | "Unknown — object does not exist" |
| HAL-002 | What algorithm uses quantum computing? | "Unknown — no quantum algorithm in knowledge base" |
| HAL-003 | What is the P99 latency of the Registry at 10B objects? | "Unknown — 10B objects not in performance spec" |
| HAL-004 | Who invented BM25? | Hedge: "External research question; Robertson & Zaragoza are cited but I cannot confirm" |
| HAL-005 | What is the current production status? | "Unknown — production state is not in the knowledge base" |

---

## Cross-References

- Reasoning certification → Phase 3.0D.0 `13-REASONING-CERTIFICATION`
- Hallucination certification → Phase 3.0D.0 `23-HALLUCINATION-CERTIFICATION`
- Golden dataset (5K reasoning Q&A) → `03-GOLDEN-DATASET`
