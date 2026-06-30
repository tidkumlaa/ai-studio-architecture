# KNW-FINAL-006 — Canonical Relationships

**Phase:** 3.0D.0.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Provides canonical examples, anti-patterns, and usage rules for all 10 Knowledge Relationship types.

---

## Relationship Type Reference

| Type | Direction | Inverse label | Strength |
|------|-----------|---------------|----------|
| DEPENDS_ON | A → B | DEPENDENCY_OF | HARD |
| IMPLEMENTS | A → B | IMPLEMENTED_BY | HARD |
| USES | A → B | USED_BY | SOFT |
| EXTENDS | A → B | EXTENDED_BY | HARD |
| TESTS | A → B | TESTED_BY | — |
| PROVIDES | A → B | PROVIDED_BY | — |
| RELATED_TO | A ↔ B | RELATED_TO | — |
| REFERENCES | A → B | REFERENCED_BY | OPTIONAL |
| SUPERSEDES | A → B | SUPERSEDED_BY | — |
| OWNED_BY | A → B | OWNS | — |

---

## 1. DEPENDS_ON — Canonical Example

```yaml
# KNW-PLT-MOD-001 (Quota Manager) DEPENDS_ON KNW-RT-RT-001 (AI Runtime)
- relation_id: REL-PLT-001
  relation_type: DEPENDS_ON
  source_id: KNW-PLT-MOD-001
  target_id: KNW-RT-RT-001
  strength: HARD
  version_constraint: ">=1.0.0"
  optional: false
  reason: "Quota Manager requires AI Runtime for context propagation"
```

**Rule:** Use DEPENDS_ON when the source cannot function without the target.  
**Anti-pattern:** Using DEPENDS_ON when USES is correct (source degrades gracefully without target).

---

## 2. IMPLEMENTS — Canonical Example

```yaml
# KNW-RT-SVC-007 (AI Router Service) IMPLEMENTS KNW-PLT-MOD-004 (Routing Engine)
- relation_id: REL-RT-001
  relation_type: IMPLEMENTS
  source_id: KNW-RT-SVC-007
  target_id: KNW-PLT-MOD-004
  reason: "AI Router Service is the runtime implementation of the Routing Engine module"
```

**Rule:** Source is the concrete implementation; target is the abstract specification/module.  
**Anti-pattern:** Implementing in reverse (specification IMPLEMENTS the concrete class).

---

## 3. USES — Canonical Example

```yaml
# KNW-PLT-MOD-002 (AI Router) USES KNW-ALG-ALG-007 (BM25 Ranker)
- relation_id: REL-PLT-002
  relation_type: USES
  source_id: KNW-PLT-MOD-002
  target_id: KNW-ALG-ALG-007
  optional: true
  reason: "Router uses BM25 for context ranking; falls back to TF-IDF if BM25 unavailable"
```

**Rule:** Use USES when the source degrades gracefully without the target (optional dependency).

---

## 4. EXTENDS — Canonical Example

```yaml
# KNW-PROV-PROV-002 (Anthropic Provider) EXTENDS KNW-META-STD-001 (Base Provider)
- relation_id: REL-PROV-001
  relation_type: EXTENDS
  source_id: KNW-PROV-PROV-002
  target_id: KNW-META-STD-001
  reason: "Anthropic Provider extends the Base Provider interface"
```

**Rule:** Source adds or specialises behaviour from the target.

---

## 5. TESTS — Canonical Example

```yaml
# KNW-TEST-TST-001 (Quota Enforcement Test) TESTS KNW-PLT-MOD-001 (Quota Manager)
- relation_id: REL-TEST-001
  relation_type: TESTS
  source_id: KNW-TEST-TST-001
  target_id: KNW-PLT-MOD-001
  coverage_type: INTEGRATION
  reason: "Tests all quota enforcement scenarios for the Quota Manager"
```

**Rule:** Only TEST objects may be the source of TESTS relationships.

---

## 6. PROVIDES — Canonical Example

```yaml
# KNW-PROV-SVC-001 (Provider Registry) PROVIDES KNW-API-API-002 (Provider API)
- relation_id: REL-PROV-002
  relation_type: PROVIDES
  source_id: KNW-PROV-SVC-001
  target_id: KNW-API-API-002
  reason: "Provider Registry Service provides the Provider API endpoint"
```

---

## 7. RELATED_TO — Canonical Example

```yaml
# KNW-ALG-ALG-007 (BM25) RELATED_TO KNW-ALG-ALG-008 (TF-IDF)
- relation_id: REL-ALG-001
  relation_type: RELATED_TO
  source_id: KNW-ALG-ALG-007
  target_id: KNW-ALG-ALG-008
  reason: "BM25 and TF-IDF are both term-frequency ranking algorithms"
```

**Rule:** RELATED_TO is undirected — stored in both directions automatically.

---

## 8. REFERENCES — Canonical Example

```yaml
# KNW-META-DEC-001 REFERENCES KNW-ALG-ALG-007
- relation_id: REL-META-001
  relation_type: REFERENCES
  source_id: KNW-META-DEC-001
  target_id: KNW-ALG-ALG-007
  reason: "ADR-001 references BM25 as the search algorithm choice"
```

**Rule:** Informational only — does not imply dependency.

---

## 9. SUPERSEDES — Canonical Example

```yaml
# KNW-PLT-MOD-002 (AI Router v2) SUPERSEDES KNW-PLT-MOD-001-LEGACY
- relation_id: REL-PLT-003
  relation_type: SUPERSEDES
  source_id: KNW-PLT-MOD-002
  target_id: KNW-PLT-MOD-000
  reason: "AI Router v2 replaces the legacy Router with streaming support"
```

**Rule:** The target should be in DEPRECATED state when SUPERSEDES is added.

---

## 10. OWNED_BY — Canonical Example

```yaml
# KNW-PLT-MOD-001 OWNED_BY team:platform
- relation_id: REL-OWN-001
  relation_type: OWNED_BY
  source_id: KNW-PLT-MOD-001
  target_id: team:platform
  reason: "Platform team is the primary maintainer"
```

**Note:** Ownership is usually captured in `ownership.owner` field. OWNED_BY relationship is used when ownership needs to be traversable in the graph.

---

## Relationship Anti-Patterns

| Anti-Pattern | Error | Fix |
|-------------|-------|-----|
| `A DEPENDS_ON B; B DEPENDS_ON A` | Circular dependency | Break cycle; one should USES the other |
| `A IMPLEMENTS A` | Self-reference | Remove — objects cannot implement themselves |
| TEST object using DEPENDS_ON | Wrong type | Use TESTS relationship |
| RELATED_TO with `optional: false` | RELATED_TO is always optional | Remove optional field |
| SUPERSEDES without DEPRECATED target | Inconsistent state | Deprecate target first |

---

## Cross-References

- Relationship engine → Phase 3.0C `06-RELATIONSHIP-ENGINE`
- Relationship certification → Phase 3.0D.0 `08-RELATIONSHIP-CERTIFICATION`
- Traceability links → Phase 3.0C.5 `29-KNOWLEDGE-TRACEABILITY`
