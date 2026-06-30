# KNW-FINAL-010 — Canonical Search Cases

**Phase:** 3.0D.0.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Defines 50 canonical search test cases with queries, ground-truth top results, and pass criteria. Used to verify search accuracy and edge-case handling.

---

## Exact Match Cases (SC-EXACT-001–010)

| ID | Query | Expected Top1 | Pass Criteria |
|----|-------|--------------|---------------|
| SC-EXACT-001 | "Quota Manager" | KNW-PLT-MOD-001 | Top1 = exact match |
| SC-EXACT-002 | "BM25 Text Ranker" | KNW-ALG-ALG-007 | Top1 = exact match |
| SC-EXACT-003 | "AI Runtime" | KNW-RT-RT-001 | Top1 = exact match |
| SC-EXACT-004 | "Tarjan SCC" | KNW-ALG-ALG-009 | Top1 = exact match |
| SC-EXACT-005 | "plt.module.quota-manager" | KNW-PLT-MOD-001 | canonical_name match |
| SC-EXACT-006 | "KNW-PLT-MOD-001" | KNW-PLT-MOD-001 | ID match |
| SC-EXACT-007 | "Quota Enforcement Test" | KNW-TEST-TST-001 | Exact name match |
| SC-EXACT-008 | "Identity Engine Performance Benchmark" | KNW-TEST-BENCH-001 | Exact match |
| SC-EXACT-009 | "Use Python for Runtime" | KNW-META-DEC-001 | ADR match |
| SC-EXACT-010 | "Anthropic Provider" | KNW-PROV-PROV-002 | Provider match |

---

## Concept Search Cases (SC-CONCEPT-001–010)

| ID | Query | Expected Top5 includes | Notes |
|----|-------|----------------------|-------|
| SC-CONCEPT-001 | "rate limiting" | KNW-PLT-MOD-001, KNW-ALG-ALG-003 | Token bucket algorithm |
| SC-CONCEPT-002 | "text ranking" | KNW-ALG-ALG-007, KNW-ALG-ALG-008 | BM25 + TF-IDF |
| SC-CONCEPT-003 | "dependency resolution" | KNW-PROV-SVC-001, kos.algorithm.* | Graph deps |
| SC-CONCEPT-004 | "strongly connected" | KNW-ALG-ALG-009 | Tarjan SCC |
| SC-CONCEPT-005 | "execution planning" | KNW-RT-MOD-001 | Execution Planner |
| SC-CONCEPT-006 | "provider adapter" | KNW-PROV-PROV-001, KNW-PROV-PROV-002 | All providers |
| SC-CONCEPT-007 | "quality scoring" | Phase 3.0C.5 `17-KNOWLEDGE-SCORING` | Scoring algorithm |
| SC-CONCEPT-008 | "circuit breaker" | KNW-PAT-PAT-002 | Pattern |
| SC-CONCEPT-009 | "event sourcing" | KNW-PAT-PAT-003 | Pattern |
| SC-CONCEPT-010 | "financial billing" | kos.financial.package objects | FIN namespace |

---

## Namespace-Qualified Cases (SC-NS-001–005)

| ID | Query | Expected Results |
|----|-------|----------------|
| SC-NS-001 | "plt:quota" | Only plt namespace objects matching "quota" |
| SC-NS-002 | "alg:ranking" | Only alg namespace ranking algorithms |
| SC-NS-003 | "test:benchmark" | Only TEST-type BENCHMARK objects |
| SC-NS-004 | "rt:*" | All objects in rt namespace |
| SC-NS-005 | "meta:decision" | All ADR (DECISION) objects in meta |

---

## Typo Tolerance Cases (SC-TYPO-001–010)

| ID | Typo Query | Expected Top1 | Edit Distance |
|----|------------|--------------|---------------|
| SC-TYPO-001 | "quota managor" | KNW-PLT-MOD-001 | 1 |
| SC-TYPO-002 | "bm 25 ranker" | KNW-ALG-ALG-007 | space insertion |
| SC-TYPO-003 | "algoritm" | alg namespace objects | 1 (missing 'h') |
| SC-TYPO-004 | "tracsability" | traceability objects | 2 |
| SC-TYPO-005 | "exection planner" | KNW-RT-MOD-001 | 1 ('cu' → 'c') |
| SC-TYPO-006 | "tarjan scc algoritm" | KNW-ALG-ALG-009 | 1 |
| SC-TYPO-007 | "antrohpic provider" | KNW-PROV-PROV-002 | 2 |
| SC-TYPO-008 | "registary" | registry objects | 1 |
| SC-TYPO-009 | "knowldge runtime" | KNW-RT-RT-001 | 1 |
| SC-TYPO-010 | "platfrom modules" | plt MODULE objects | 1 |

---

## Edge Cases (SC-EDGE-001–010)

| ID | Query | Expected Behaviour |
|----|-------|--------------------|
| SC-EDGE-001 | "" (empty) | Returns [] — no crash |
| SC-EDGE-002 | " " (whitespace only) | Returns [] — trimmed |
| SC-EDGE-003 | 10,000-character query | Returns [] or truncates at 512 |
| SC-EDGE-004 | "SELECT * FROM objects" | Returns [] — no SQL execution |
| SC-EDGE-005 | "<script>alert(1)</script>" | Returns [] — sanitised |
| SC-EDGE-006 | "คำค้นหาภาษาไทย" (Thai) | Handles gracefully; returns best match |
| SC-EDGE-007 | "搜索算法" (CJK) | Handles gracefully |
| SC-EDGE-008 | "quota*" (wildcard) | Returns all objects matching prefix |
| SC-EDGE-009 | "quota AND manager" | Returns intersection |
| SC-EDGE-010 | Archived object name | NOT returned in default results |

---

## Alias Cases (SC-ALIAS-001–005)

| ID | Query (alias) | Expected Top1 | Notes |
|----|--------------|--------------|-------|
| SC-ALIAS-001 | "knw-plt-mod-001" (lowercase ID) | KNW-PLT-MOD-001 | Case-insensitive ID |
| SC-ALIAS-002 | Old ID after rename | New canonical object | Alias resolution |
| SC-ALIAS-003 | "plt.module.quota-manager" | KNW-PLT-MOD-001 | canonical_name as query |
| SC-ALIAS-004 | "QuotaManager" (no space) | KNW-PLT-MOD-001 | Word-join normalisation |
| SC-ALIAS-005 | "Quota_Manager" (underscore) | KNW-PLT-MOD-001 | Underscore normalisation |

---

## Ground Truth Format

```yaml
# knowledge/datasets/golden/ground-truth/search-queries.yaml
- query_id: SC-EXACT-001
  query: "Quota Manager"
  top1: KNW-PLT-MOD-001
  top5:
    - KNW-PLT-MOD-001
    - KNW-PLT-REQ-001
    - KNW-TEST-TST-001
    - KNW-PLT-MOD-003
    - KNW-TEST-BENCH-001
  relevance_set: [KNW-PLT-MOD-001, KNW-PLT-REQ-001, KNW-TEST-TST-001]
```

---

## Cross-References

- Search certification → Phase 3.0D.0 `02-SEARCH-CERTIFICATION`
- Golden dataset (10K queries) → `03-GOLDEN-DATASET`
- Alias handling → Phase 3.0C.5 `14-KNOWLEDGE-REGISTRY`
