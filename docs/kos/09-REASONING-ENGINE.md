---
knowledge_id: KNW-KOS-ARCH-009
title: "KOS Knowledge Reasoning Engine"
status: AUTHORITATIVE
phase: "3.0A"
version: "1.0.0"
created: "2026-06-30"
purpose: "Specify the Knowledge Reasoning Engine: answering why/how/impact/risk/dependency questions"
canonical_source: "architecture/docs/kos/09-REASONING-ENGINE.md"
dependencies:
  - "05-KNOWLEDGE-GRAPH.md"
  - "08-QUERY-ENGINE.md"
related_documents:
  - "10-EXECUTION-ENGINE.md"
  - "12-QUALITY.md"
acceptance_criteria:
  - "All 10 reasoning query types are specified"
  - "Reasoning pipeline is defined"
  - "Evidence-based answers are required (not opinion)"
verification_checklist:
  - "[ ] Why, How, Impact, Risk, Dependency covered"
  - "[ ] Answers reference knowledge_ids as evidence"
  - "[ ] Confidence scoring defined"
future_extensions:
  - "Continuous reasoning: monitor knowledge changes and proactively surface impacts"
  - "Counterargument reasoning: surface objections to decisions"
---

# KOS Knowledge Reasoning Engine

## Purpose

The Knowledge Reasoning Engine answers questions that require **interpretation**
of the Knowledge Graph — not just retrieval. Where the Query Engine retrieves
objects, the Reasoning Engine derives meaning from their relationships.

---

## Reasoning Query Types

### R1 — WHY

**Question:** Why does X exist? Why was this decision made?  
**Method:** Traverse `MOTIVATED_BY` and `DECIDED_IN` edges from the object to Requirements and Decisions.

```python
engine.why("KNW-AI-MOD-quota_manager")
# Answer: "quota_manager exists because:
#   - KNW-REQ-AI-001 requires resource quota control (MUST)
#   - KNW-ADR-009 decided Decimal for monetary values (prevents float errors)
#   - Platform Team approved 2026-06-29"
```

### R2 — HOW

**Question:** How does X work? How is this requirement fulfilled?  
**Method:** Traverse `IMPLEMENTS`, `EXPOSES`, `PROVIDES` edges; include algorithm references.

```python
engine.how("KNW-AI-MOD-quota_manager")
# Answer: "quota_manager works by:
#   - Implementing ALG-quota-check-001 (O(1) in-memory check)
#   - Providing capability:ai:quota.check:v1 and quota.consume:v1
#   - Depending on usage_collector for consumption history"
```

### R3 — IMPACT

**Question:** If X changes, what is the blast radius?  
**Method:** Traverse `DEPENDED_ON_BY` edges transitively. Compute reachability set.

```python
engine.impact("KNW-AI-MOD-quota_manager")
# Answer: ImpactReport(
#   direct_dependents=["routing_optimizer", "resource_scheduler"],
#   transitive_dependents=["execution_planner", "recommendation_center", ...],
#   affected_apis=["POST /api/v1/resources/quota/check"],
#   affected_tests=["test_quota_manager.py", "test_routing_optimizer.py"],
#   risk_level="HIGH"  # 12+ transitive dependents
# )
```

### R4 — RISK

**Question:** What are the risks of X?  
**Method:** Analyze `quality_score`, `confidence`, `evidence` fields.
Cross-reference with benchmarks for performance risk. Check lifecycle status.

```python
engine.risk("KNW-AI-MOD-quota_manager")
# Answer: RiskReport(
#   quality_score=0.91,
#   confidence=0.95,
#   risks=[
#     Risk("Coverage: 3 edge cases not in tests", severity="LOW"),
#     Risk("No benchmark for concurrent access under load", severity="MEDIUM"),
#   ],
#   overall="LOW_RISK"
# )
```

### R5 — DEPENDENCY

**Question:** What does X depend on? What is the full dependency chain?  
**Method:** Traverse `DEPENDS_ON` edges transitively. Produce layered dependency tree.

```python
engine.dependency("KNW-AI-MOD-execution_planner")
# Answer: DependencyTree showing all transitive deps in boot order
```

### R6 — HISTORY

**Question:** How has X changed over time?  
**Method:** Query version history from Knowledge Registry. Diff knowledge object versions.

```python
engine.history("KNW-AI-MOD-quota_manager")
# Answer: list of VersionSnapshot(version, changed_at, changed_by, diff_summary)
```

### R7 — ALTERNATIVES

**Question:** What alternatives were considered for X?  
**Method:** Query `DECIDED_IN` edges → Decision objects → `options_considered` field.

```python
engine.alternatives("KNW-ADR-009")
# Answer: "For monetary values, alternatives were:
#   - float: rejected (precision errors documented)
#   - int (cents): rejected (complexity, edge cases)
#   - Decimal: chosen (exact, standard library)"
```

### R8 — TRADEOFFS

**Question:** What are the tradeoffs of X?  
**Method:** Query Decision objects for `consequences_positive` and `consequences_negative`.

```python
engine.tradeoffs("KNW-ADR-002")
# Answer: Tradeoffs(
#   positive=["decoupled boot", "auditable events", ...],
#   negative=["eventual consistency", "debugging complexity", ...]
# )
```

### R9 — ARCHITECTURE DECISIONS

**Question:** What architecture decisions affect X?  
**Method:** Traverse `DECIDED_IN` and `MOTIVATED_BY` edges from the object to all reachable Decisions.

```python
engine.decisions("KNW-AI-MOD-provider_selector")
# Answer: [ADR-006 (Provider Plugin), ADR-009 (Decimal), ADR-004 (Protocol interfaces)]
```

### R10 — EXECUTION PLANS

**Question:** What is the plan to implement X?  
**Method:** Delegated to Execution Engine (10-EXECUTION-ENGINE.md).
The Reasoning Engine prepares the context; the Execution Engine generates the plan.

---

## Reasoning Pipeline

```
Question
    │
    ▼
1. Parse question → QueryIntent (classify R1–R10)
    │
    ▼
2. Extract subjects (knowledge_ids or search terms)
    │
    ▼
3. Fetch relevant subgraph via QueryEngine
    │
    ▼
4. Apply reasoning strategy for this query type
    │
    ▼
5. Collect evidence (knowledge_ids supporting the answer)
    │
    ▼
6. Score confidence (0.0–1.0 based on evidence completeness)
    │
    ▼
7. Format answer (structured + natural language)
    │
    ▼
ReasoningAnswer
```

---

## ReasoningAnswer Schema

```python
@dataclass(frozen=True)
class ReasoningAnswer:
    question: str
    query_type: str              # "WHY" | "HOW" | "IMPACT" | ...
    answer_text: str             # Natural language answer
    evidence: list[str]          # knowledge_ids supporting the answer
    confidence: float            # 0.0–1.0
    graph_path: list[str]        # traversal path taken
    computed_at: datetime
    warning: str | None          # if confidence < 0.5
```

---

## Reasoning Engine Interface

```python
class KnowledgeReasoningEngine(Protocol):
    def why(self, knowledge_id: str) -> ReasoningAnswer: ...
    def how(self, knowledge_id: str) -> ReasoningAnswer: ...
    def impact(self, knowledge_id: str) -> ImpactReport: ...
    def risk(self, knowledge_id: str) -> RiskReport: ...
    def dependency(self, knowledge_id: str) -> DependencyTree: ...
    def history(self, knowledge_id: str) -> list[VersionSnapshot]: ...
    def alternatives(self, knowledge_id: str) -> ReasoningAnswer: ...
    def tradeoffs(self, knowledge_id: str) -> ReasoningAnswer: ...
    def decisions(self, knowledge_id: str) -> list[DecisionObject]: ...
    def ask(self, question: str) -> ReasoningAnswer: ...  # free-form
```

---

## ImpactReport

```python
@dataclass(frozen=True)
class ImpactReport:
    subject_id: str
    direct_dependents: list[str]
    transitive_dependents: list[str]
    affected_apis: list[str]
    affected_tests: list[str]
    affected_products: list[str]
    risk_level: str              # "LOW" | "MEDIUM" | "HIGH" | "CRITICAL"
    blast_radius: int            # total count of affected objects
```

**Risk level thresholds:**

| Blast Radius | Risk Level |
|-------------|------------|
| 0–3 objects | LOW |
| 4–10 objects | MEDIUM |
| 11–25 objects | HIGH |
| 26+ objects | CRITICAL |

---

## CLI

```bash
platform-kos reason why KNW-AI-MOD-quota_manager
platform-kos reason how KNW-AI-MOD-execution_planner
platform-kos reason impact KNW-AI-MOD-quota_manager
platform-kos reason risk KNW-AI-MOD-quota_manager
platform-kos reason alternatives KNW-ADR-009
platform-kos ask "What would break if the event bus was replaced?"
```
