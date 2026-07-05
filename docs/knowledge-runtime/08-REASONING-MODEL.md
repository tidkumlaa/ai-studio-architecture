---
knowledge_id: KR-ARCH-008
version: "1.0.0"
status: approved
owner: Chief Knowledge Architect
phase: 2.0D.3A
created: 2026-06-29
review_date: 2027-06-29
canonical: true
domain: DOM-KNOWLEDGE
capability: knowledge-runtime
type: architecture
depends_on:
  - id: KA-KIP-006
    reason: "Reasoning spec defines inference rule schema and forward-chaining algorithm"
  - id: KA-SPEC-012
    reason: "Evidence model defines evidence states consumed by coverage rules"
  - id: KA-SPEC-013
    reason: "Coverage model defines dimensions and gap thresholds"
  - id: KA-KIP-003
    reason: "Graph traversal used for transitive dependency and impact propagation"
---

# Knowledge Runtime — Reasoning Model

## Forward-Chaining Rule Engine · Dependency Propagation · Impact Analysis · Coverage Inference · Consistency Checking

---

## 1. Reasoning Overview

The `KnowledgeReasoningEngine` applies **forward-chaining rule-based reasoning** to derive new knowledge facts from the existing set of declared knowledge objects and relationships. Derived facts are not stored permanently in the KnowledgeStore; they reside in a **working memory** that is recomputed on each reasoning pass and cached for query access.

Reasoning serves three purposes:

| Purpose | Description |
|---------|-------------|
| **Enrichment** | Derive facts that would be tedious or impossible for authors to declare manually (transitive dependencies, impact chains) |
| **Gap detection** | Identify coverage gaps, orphaned objects, stale evidence, and structural holes |
| **Consistency enforcement** | Detect contradictions, duplicates, and lifecycle violations that schema validation alone cannot catch |

### 1.1 Invocation Triggers

The reasoning engine runs under three triggers:

| Trigger | Scope | Typical Duration |
|---------|-------|-----------------|
| **Boot** (Phase 3 of boot sequence) | Full pass — all objects and rules | 2–30 s depending on corpus size |
| **Incremental** (`knowledge.store.written` event) | Affected objects and their graph neighbourhood (radius 3) | < 500 ms |
| **On-demand** (KQL `REASON` clause) | Objects returned by the preceding SELECT | < 200 ms |

Incremental reasoning uses a **dirty set**: when a `knowledge.store.written` event arrives, all objects within 3 graph hops of the written object are added to the dirty set and re-reasoned on the next cycle (default: 100 ms debounce).

### 1.2 Architectural Position

```
KnowledgeStore ──► KnowledgeGraphRuntime ──► KnowledgeReasoningEngine
                                                      │
                        ┌─────────────────────────────┤
                        │                             │
                        ▼                             ▼
              WorkingMemory (facts)     KnowledgeRecommendationEngine
                        │
                        ▼
              KnowledgeQueryEngine (REASON clause resolved here)
```

---

## 2. Rule Engine

### 2.1 Forward-Chaining Algorithm

The engine executes the **Rete-inspired forward-chaining** algorithm:

```
REASONING_PASS(objects, graph, rules):

  1. LOAD active rules ordered by priority DESC
  2. INITIALISE working_memory ← assert_base_facts(objects, graph)
  3. REPEAT
       new_facts ← []
       FOR rule IN rules WHERE rule.enabled:
         matches ← pattern_match(working_memory, rule.condition)
         FOR match IN matches:
           fact ← apply(rule.conclusion, match)
           IF fact NOT IN working_memory:
             new_facts.append(fact)
       IF new_facts IS EMPTY:
         BREAK                          -- fixpoint reached
       working_memory.add_all(new_facts)
  4. RETURN working_memory
```

The algorithm terminates because:
- Derived facts are monotonically added (never removed mid-pass).
- Each rule is bounded by `max_depth` limits on transitive closures.
- The total number of possible distinct facts is finite (bounded by object count squared for relationship facts).

### 2.2 Rule Structure

```
ReasoningRule:
  rule_id:    str          # Unique identifier, e.g. "DEP-PROP-001"
  name:       str          # Human-readable label
  category:   str          # DEPENDENCY | IMPACT | COVERAGE | CONSISTENCY
  condition:  KQLExpression  # Pattern in working memory that triggers the rule
  conclusion: DerivedFact  # What to assert when condition matches
  priority:   int          # Evaluation order; higher = evaluated first
  max_depth:  int | None   # Recursion limit for transitive rules; None = not transitive
  enabled:    bool         # Runtime toggle; disabled rules are skipped entirely
```

### 2.3 Derived Fact Types

| Fact Type | Meaning | Consumers |
|-----------|---------|-----------|
| `TransitiveEdge(type, from, to, weight, depth)` | Inferred graph relationship | QueryEngine, ImpactAnalyzer |
| `ObjectAnnotation(object_id, key, value)` | Attribute derived by rule (e.g. `dependency_risk`) | QueryEngine, HealthScorer |
| `INCONSISTENCY(type, rule_id, objects)` | Structural contradiction | RecommendationEngine, EventBus |
| `COVERAGE_GAP(object_id, dimension, score, threshold)` | Coverage below threshold | RecommendationEngine, CoverageEngine |
| `RECOMMENDATION(action, targets, confidence)` | Actionable suggestion | RecommendationEngine |

---

## 3. Dependency Reasoning

Category `DEPENDENCY`. Propagates facts through `DEPENDS_ON` and `IMPLEMENTS` relationship chains.

### DEP-PROP-001 — Transitive Dependency

| Field | Value |
|-------|-------|
| Condition | `A DEPENDS_ON B` AND `B DEPENDS_ON C` AND `A ≠ C` |
| Conclusion | `TransitiveEdge(DEPENDS_ON, A, C, weight=0.5, depth=current+1)` |
| Max depth | 5 (prevents combinatorial explosion in dense dependency graphs) |
| Priority | 90 |

The weight of 0.5 distinguishes transitive edges from declared direct edges (weight 1.0). Query results can filter by weight to control transitivity exposure.

### DEP-PROP-002 — Lifecycle Propagation

| Field | Value |
|-------|-------|
| Condition | `A DEPENDS_ON B` AND `B.lifecycle_state = DEPRECATED` |
| Conclusion | `ObjectAnnotation(A, "reasoning.dependency_risk", "HIGH")` |
| Priority | 80 |
| Notes | Applies to both direct and transitive DEPENDS_ON edges |

### DEP-PROP-003 — Circular Dependency Warning

| Field | Value |
|-------|-------|
| Condition | `A DEPENDS_ON B` AND `B DEPENDS_ON A` (direct cycle, length 2) |
| Conclusion | `INCONSISTENCY(CIRCULAR_DEPENDENCY, "DEP-PROP-003", [A, B])` |
| Priority | 100 |
| Notes | Longer cycles (length > 2) are detected by `KnowledgeCycleDetector` as a graph-level check and emitted as `knowledge.graph.cycle.detected` events |

### DEP-PROP-004 — Missing Dependency (Text Reference)

| Field | Value |
|-------|-------|
| Condition | `A` references `B.knowledge_id` in body text AND no `DEPENDS_ON` edge exists between A and B |
| Conclusion | `RECOMMENDATION(ADD_DEPENDENCY, from=A, to=B, confidence=0.60)` |
| Priority | 20 |
| Notes | Text reference detection requires a full index scan — **expensive**. This rule runs in the overnight batch pass only; it is disabled in incremental mode. Confidence 0.60 reflects false-positive risk (IDs can appear in code examples). |

---

## 4. Impact Reasoning

Category `IMPACT`. Derives the change-impact graph from the dependency graph by inverting dependency edges.

### IMP-RULE-001 — Direct Impact

| Field | Value |
|-------|-------|
| Condition | `B DEPENDS_ON A` |
| Conclusion | `TransitiveEdge(IMPACTS, A, B, weight=1.0, depth=1)` |
| Priority | 85 |
| Notes | The `IMPACTS` edge is the semantic inverse of `DEPENDS_ON`. It is always derived, never declared by authors. |

### IMP-RULE-002 — Transitive Impact

| Field | Value |
|-------|-------|
| Condition | `A IMPACTS B` AND `B IMPACTS C` AND `A ≠ C` |
| Conclusion | `TransitiveEdge(IMPACTS, A, C, weight=0.6, depth=current+1)` |
| Max depth | 10 |
| Priority | 75 |
| Notes | max_depth=10 is intentionally deeper than DEP-PROP-001 because understanding blast radius is the primary use case for impact analysis |

### IMP-RULE-003 — Critical Path

| Field | Value |
|-------|-------|
| Condition | `A` lies on the unique shortest path from a graph root to any leaf (no alternate route exists) |
| Conclusion | `ObjectAnnotation(A, "reasoning.is_critical", True)` |
| Priority | 60 |
| Algorithm | Computed via single-source shortest path (Dijkstra) over the dependency graph; objects with no bypass path are flagged critical |

---

## 5. Coverage Reasoning

Category `COVERAGE`. Infers coverage scores from evidence facts and propagates them through the traceability chain.

### COV-RULE-001 — Evidence Lifts Testing Coverage

| Field | Value |
|-------|-------|
| Condition | `A EVIDENCED_BY test_B` AND `test_B.evidence_state = VERIFIED` |
| Conclusion | `ObjectAnnotation(A, "coverage.testing", max(A.coverage.testing, 1.0))` |
| Priority | 70 |
| Notes | Evidence-derived coverage overrides declared coverage when higher. This prevents authors from claiming 100% testing coverage without verified evidence. |

### COV-RULE-002 — Child Coverage Lifts Parent

| Field | Value |
|-------|-------|
| Condition | `A TRACES_TO B` AND `A.coverage.architecture >= 0.8` |
| Conclusion | `ObjectAnnotation(B, "coverage.architecture", max(B.coverage.architecture, 0.5))` |
| Priority | 50 |
| Notes | A parent receives partial credit (0.5) when a child has high architecture coverage. The 0.5 coefficient reflects that one child does not fully represent the parent. Multiple qualifying children can raise the parent further through multiple rule firings. |

### COV-RULE-003 — Coverage Gap Detection

| Field | Value |
|-------|-------|
| Condition | `A.coverage.testing < 0.6` AND `A.lifecycle_state = APPROVED` |
| Conclusion | `COVERAGE_GAP(A, dimension=testing, score=A.coverage.testing, threshold=0.6)` |
| Priority | 65 |
| Trigger | Each `COVERAGE_GAP` fact fires `KnowledgeRecommendationEngine` to generate a `FILL_GAP` recommendation |

### COV-RULE-004 — Stale Coverage

| Field | Value |
|-------|-------|
| Condition | `A.coverage.testing > 0` AND no `EVIDENCED_BY` edge to a `VERIFIED` evidence record with `last_verified` within the past 90 days |
| Conclusion | `RECOMMENDATION(REFRESH_EVIDENCE, target=A, confidence=0.90)` |
| Priority | 55 |
| Notes | The 90-day window is configurable via `KnowledgeRuntimeConfig.evidence_expiry_days` |

---

## 6. Consistency Checking

Category `CONSISTENCY`. Detects contradictions that cannot be caught by per-object schema validation.

### CON-CHECK-001 — Canonical Conflict

| Field | Value |
|-------|-------|
| Condition | `A.canonical = True` AND `B.canonical = True` AND `A.capability = B.capability` AND `A.type = B.type` AND `A ≠ B` |
| Conclusion | `INCONSISTENCY(DUPLICATE_CANONICAL, "CON-CHECK-001", [A, B])` |
| Priority | 95 |
| Notes | A capability must have exactly one canonical document per type. Two canonical documents for the same slot is always an error. |

### CON-CHECK-002 — Version Inconsistency

| Field | Value |
|-------|-------|
| Condition | `A.version < B.version` AND `A SUPERSEDES B` |
| Conclusion | `INCONSISTENCY(SUPERSESSION_VERSION_ERROR, "CON-CHECK-002", [A, B])` |
| Priority | 90 |
| Notes | A document that supersedes another must carry a strictly higher semantic version number. |

### CON-CHECK-003 — Orphan Object

| Field | Value |
|-------|-------|
| Condition | `A` has no incoming or outgoing edges in the knowledge graph AND `A.type NOT IN root_types` |
| Conclusion | `RECOMMENDATION(LINK_ORPHAN, target=A, confidence=0.85)` |
| Priority | 40 |
| Notes | `root_types` is the set of types that are allowed to be graph roots (e.g. domain definitions, top-level requirements). All other objects must participate in at least one relationship. |

### CON-CHECK-004 — Lifecycle Contradiction

| Field | Value |
|-------|-------|
| Condition | `A.lifecycle_state = APPROVED` AND `A IMPLEMENTS B` AND `B.lifecycle_state IN [DRAFT, DEPRECATED, ARCHIVED]` |
| Conclusion | `INCONSISTENCY(LIFECYCLE_MISMATCH, "CON-CHECK-004", [A, B])` |
| Priority | 85 |
| Notes | An approved implementation cannot implement a document that is not also approved (or at least REVIEW). |

---

## 7. Reasoning Pass Lifecycle

```
TRIGGER (boot / store.written / KQL REASON)
        │
        ▼
 1. LOAD RULES
    ├── Filter: enabled = True
    └── Sort: priority DESC

        │
        ▼
 2. INITIALISE WORKING MEMORY
    ├── Assert base facts from KnowledgeStore snapshot
    └── Assert base graph edges from KnowledgeGraphRuntime

        │
        ▼
 3. FORWARD CHAIN (iterate to fixpoint)
    ├── Evaluate each rule's condition against working memory
    ├── Collect new derived facts
    ├── Add to working memory
    └── Repeat until no new facts derived

        │
        ▼
 4. EMIT EVENTS
    ├── knowledge.reasoning.pass.completed
    ├── knowledge.reasoning.inconsistency.detected  (one per INCONSISTENCY fact)
    └── knowledge.coverage.gap.detected             (one per COVERAGE_GAP fact)

        │
        ▼
 5. NOTIFY RECOMMENDATION ENGINE
    └── KnowledgeRecommendationEngine.process(working_memory)
```

### 7.1 Pass Metrics

Each completed pass emits the following payload on `knowledge.reasoning.pass.completed`:

| Field | Type | Meaning |
|-------|------|---------|
| `duration_ms` | int | Wall-clock time for the full pass |
| `rules_fired` | int | Number of rule-match-and-apply operations |
| `facts_derived` | int | Number of new facts added to working memory |
| `iterations` | int | Number of forward-chain iterations until fixpoint |
| `inconsistencies_detected` | int | Count of INCONSISTENCY facts |
| `coverage_gaps_detected` | int | Count of COVERAGE_GAP facts |

---

## 8. Recommendation Generation

After each reasoning pass, `KnowledgeRecommendationEngine` processes all `COVERAGE_GAP`, `INCONSISTENCY`, and `RECOMMENDATION` facts in working memory and produces ranked `KnowledgeRecommendation` objects.

### 8.1 Scoring Formula

```
score(rec) = impact(rec) × urgency(rec) × (1 / effort_weight(rec.estimated_effort))

where:
  impact  = 1.0 (CRITICAL) | 0.75 (HIGH) | 0.5 (MEDIUM) | 0.25 (LOW)
  urgency = 1.0 if lifecycle_state = APPROVED else 0.6
  effort_weight = 1 (LOW) | 2 (MEDIUM) | 4 (HIGH)
```

### 8.2 Recommendation Schema

```
KnowledgeRecommendation:
  recommendation_id:    str     # KR-REC-{timestamp}-{seq}
  type:                 str     # FILL_GAP | ADD_EVIDENCE | LINK_ORPHAN |
                                #   RESOLVE_CONFLICT | REFRESH_EVIDENCE | ADD_DEPENDENCY
  priority:             int     # 1 (critical) → 5 (nice to have)
  affected_object_id:   str     # knowledge_id of the primary affected object
  action_description:   str     # Human-readable description of recommended action
  estimated_effort:     str     # LOW | MEDIUM | HIGH
  auto_fixable:         bool    # True if runtime can resolve without human input
  confidence:           float   # 0.0–1.0; inherited from triggering rule
  generated_at:         datetime
  expires_at:           datetime | None
```

### 8.3 Auto-Fixable Recommendations

The following recommendation types are auto-fixable when `KnowledgeRuntimeConfig.auto_fix = True`:

| Type | Auto-Fix Action |
|------|----------------|
| `REFRESH_EVIDENCE` | Re-verify evidence by re-running associated test command if recorded |
| `LINK_ORPHAN` | Propose link targets based on domain and capability match; requires human approval |

All other types require human review before application.

---

## 9. Performance Constraints

| Operation | Target | Notes |
|-----------|--------|-------|
| Full reasoning pass (10 K objects) | < 30 s | Acceptable at boot time |
| Incremental pass (dirty set ≤ 50 objects) | < 500 ms | Must complete before UI refresh |
| Rule pattern matching (per rule per iteration) | O(E) where E = graph edges | Rete partial match indexing reduces constant factor |
| Working memory footprint | < 200 MB for 50 K derived facts | Fact tuples are small; stored as interned strings |
| Fixpoint iterations | ≤ 20 in practice | max_depth limits prevent runaway transitive closure |

---

## 10. Cross-References

| Document | Relationship |
|----------|-------------|
| `KA-KIP-006` (05-KNOWLEDGE-REASONING.md) | Prior art: inference rule schema and Rete algorithm sketch |
| `KA-SPEC-012` (12-KNOWLEDGE-EVIDENCE-MODEL.md) | Defines `EVIDENCED_BY` edge and `evidence_state` values |
| `KA-SPEC-013` (13-KNOWLEDGE-COVERAGE-MODEL.md) | Defines coverage dimensions and gap thresholds |
| `KR-ARCH-009` (09-EVENT-MODEL.md) | Events emitted by this engine |
| `KR-ARCH-010` (10-DIAGRAMS.md) | Reasoning flow diagram (Section 5) |
| `KR-ARCH-011` (11-VERIFICATION.md) | Rule catalog verification table |
