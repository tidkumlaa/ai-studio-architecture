# KNW-FINAL-012 — Canonical Traceability

**Phase:** 3.0D.0.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Provides canonical complete traceability chains — from Requirement through all 9 levels to Monitoring. These are the ground-truth chains used for traceability certification.

---

## Canonical Chain 1 — Quota Enforcement

```
KNW-PLT-REQ-001  (Requirement)
  The platform MUST enforce per-session token quotas.
    ↓ satisfies
KNW-ARCH-ARCH-001  (Architecture)
  KOS Platform Architecture — Quota section
    ↓ implements
KNW-META-DEC-001  (Decision — ADR-001)
  Use Python for Knowledge Runtime
    ↓ specifies
KNW-PLT-SPEC-001  (Specification)
  Quota Manager Interface Specification
    ↓ implemented_by
KNW-PLT-MOD-001  (Module)
  Quota Manager
    ↓ exposed_by
KNW-PLT-SVC-001  (Service)
  Platform API Service
    ↓ contracted_by
KNW-API-API-001  (API)
  Knowledge Query API
    ↓ verified_by
KNW-TEST-TST-001  (Test)
  Quota Enforcement Test
    ↓ deployed_by
KNW-PLT-DEP-001  (Deployment)
  Platform Production Deployment
    ↓ monitored_by
KNW-PLT-MON-001  (Monitoring)
  Platform Quota Monitoring
```

---

## Canonical Chain 2 — AI Routing

```
KNW-PLT-REQ-002  (Requirement)
  The platform MUST route AI requests to the optimal provider.
    ↓ satisfies
KNW-ARCH-ARCH-002  (Architecture)
  KOS Routing Architecture
    ↓ implements
KNW-META-DEC-003  (Decision — ADR-003)
  Use BM25 for context ranking
    ↓ specifies
KNW-PLT-SPEC-002  (Specification)
  Routing Engine Interface Specification
    ↓ implemented_by
KNW-PLT-MOD-004  (Module)
  Routing Engine
    ↓ exposed_by
KNW-RT-SVC-007  (Service)
  AI Router Service
    ↓ contracted_by
KNW-API-API-003  (API)
  Registry API
    ↓ verified_by
KNW-TEST-TST-005  (Test)
  AI Routing Test
    ↓ deployed_by
KNW-RT-DEP-001  (Deployment)
  Runtime Deployment
    ↓ monitored_by
KNW-RT-MON-001  (Monitoring)
  Runtime Latency Monitoring
```

---

## Canonical Chain 3 — Provider Registry

```
KNW-PLT-REQ-003  (Requirement)
  The platform MUST maintain a registry of all AI providers.
    ↓ satisfies
KNW-ARCH-ARCH-003  (Architecture)
  KOS Provider Architecture
    ↓ implements
KNW-META-DEC-005  (Decision)
  Use file-based registry as primary store
    ↓ specifies
KNW-PROV-SPEC-001  (Specification)
  Provider Registry Specification
    ↓ implemented_by
KNW-PROV-SVC-001  (Service)
  Provider Registry Service
    ↓ exposed_by
KNW-PROV-SVC-001  (Service — same, provides API directly)
    ↓ contracted_by
KNW-API-API-002  (API)
  Provider API
    ↓ verified_by
KNW-TEST-TST-010  (Test)
  Provider Registry Test
    ↓ deployed_by
KNW-PROV-DEP-001  (Deployment)
  Provider Deployment
    ↓ monitored_by
KNW-PROV-MON-001  (Monitoring)
  Provider Health Monitoring
```

---

## Traceability Matrix (Sample — 10 Requirements)

```
REQ              ARCH    DEC     SPEC    MOD/SVC  SVC     API     TST     DEP     MON     COMPLETE
KNW-PLT-REQ-001  ✓       ✓       ✓       ✓        ✓       ✓       ✓       ✓       ✓       YES
KNW-PLT-REQ-002  ✓       ✓       ✓       ✓        ✓       ✓       ✓       ✓       ✓       YES
KNW-PLT-REQ-003  ✓       ✓       ✓       ✓        ✓       ✓       ✓       ✓       ✓       YES
KNW-PLT-REQ-004  ✓       ✓       ✓       ✓        ✓       ✗       ✗       ✗       ✗       NO
KNW-PLT-REQ-005  ✓       ✓       ✓       ✓        ✓       ✓       ✓       ✓       ✓       YES
KNW-RT-REQ-001   ✓       ✓       ✓       ✓        ✓       ✓       ✓       ✗       ✓       NO
KNW-RT-REQ-002   ✓       ✓       ✗       ✓        ✗       ✗       ✓       ✗       ✗       NO
KNW-PROV-REQ-001 ✓       ✓       ✓       ✓        ✓       ✓       ✓       ✓       ✓       YES
KNW-ALG-REQ-001  ✓       ✓       ✓       ✓        ✗       ✗       ✓       ✗       ✗       NO
KNW-META-REQ-001 ✓       ✓       ✓       ✓        ✓       ✓       ✓       ✓       ✓       YES
```

---

## Bidirectionality Verification

Every forward link must have a matching backward link:

```
Chain 1 forward: REQ-001 → ARCH-001 → DEC-001 → SPEC-001 → MOD-001 → ...
Chain 1 backward: MON-001 ← DEP-001 ← TST-001 ← API-001 ← SVC-001 ← MOD-001 ← ...

For each pair (A, B) where A satisfies B:
  B.traceability.tested_by must include A    (or appropriate inverse)
  AND A.traceability.satisfies must include B
```

---

## Incomplete Chains (Known Gaps for Testing)

50 requirements with intentionally incomplete chains are included in the traceability dataset for gap testing:

```
Gap types:
  - Missing DEPLOYMENT (30 cases) — Deployment object not yet created
  - Missing MONITORING (15 cases) — Monitoring not yet wired
  - Missing API (5 cases)  — API contract not formalised
```

---

## Cross-References

- Traceability model → Phase 3.0C.5 `29-KNOWLEDGE-TRACEABILITY`
- Traceability certification → Phase 3.0D.0 `05-TRACEABILITY-CERTIFICATION`
- Dataset traceability chains → `03-GOLDEN-DATASET`
