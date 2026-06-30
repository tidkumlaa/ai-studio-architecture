# KNW-KC-ARCH-030 — Reasoning Model

**Phase:** 3.0C  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

The Reasoning Model defines how KOS answers higher-order questions about the knowledge graph — not just "find X" but "why X", "how X", "what if X changes". Ten reasoning types are specified.

---

## Reasoning Types (10)

| Code | Type | Question Template | Algorithm |
|------|------|------------------|-----------|
| R-1 | **Why** | Why does X exist? Why was this decision made? | Evidence traversal + Decision chain |
| R-2 | **How** | How does X work? How is X implemented? | Inheritance traversal + Implementation trace |
| R-3 | **Impact** | What is impacted if X changes? | ALG-KC-016 (Impact graph, reverse traversal) |
| R-4 | **Risk** | What risks does X carry? | Evidence gaps + confidence decay + impact size |
| R-5 | **Dependency** | What does X depend on? | ALG-KC-016 (Dependency closure) |
| R-6 | **History** | How has X evolved? | Version Engine + snapshot diff |
| R-7 | **Alternatives** | What are the alternatives to X? | Semantic similarity + conflicts_with edges |
| R-8 | **Tradeoffs** | What are the tradeoffs of X vs Y? | Compare evidence, quality, confidence |
| R-9 | **Decisions** | What decisions govern X? | Decision chain traversal |
| R-10 | **Plans** | What execution plan achieves X? | Strategy → ExecutionPlan traversal |

---

## Reasoning Pipeline

```
REASON(object_id, reasoning_type, context):
  1. LOAD
     Load object from registry (full Universal Schema)
     Load related objects (depth 2 by default)

  2. GATHER
     Pull evidence records (10-EVIDENCE-ENGINE)
     Pull version history (08-VERSION-ENGINE)
     Pull graph neighbourhood (24-KNOWLEDGE-GRAPH-MODEL)

  3. COMPUTE
     Apply reasoning-type-specific algorithm (see below)

  4. SCORE
     confidence = path_confidence(reasoning_chain)
     completeness = gathered_evidence / required_evidence

  5. FORMAT
     Return: ReasoningAnswer{
       question, object_id, reasoning_type,
       answer_text, supporting_evidence,
       chain, confidence, completeness
     }
```

---

## Reasoning Answer Schema

```python
@dataclass
class ReasoningAnswer:
    question: str
    object_id: str
    reasoning_type: ReasoningType
    answer_text: str                   # human-readable narrative
    chain: list[ReasoningStep]         # step-by-step reasoning path
    supporting_evidence: list[str]     # evidence_ids
    referenced_objects: list[str]      # knowledge_ids
    confidence: float                  # overall confidence in answer
    completeness: float                # how much evidence was found
    alternatives: list[str]            # alternative conclusions, if any
    caveats: list[str]                 # known limitations
    generated_at: datetime

@dataclass
class ReasoningStep:
    step_number: int
    description: str
    object_ids: list[str]
    relation_type: str | None
    confidence: float
```

---

## R-1 Why Reasoning

```
WHY(object_id):
  1. Traverse: object ← [documents] ← Decision
  2. Traverse: object ← [implements] ← Requirement
  3. For each Decision found: read context, rationale, evidence
  4. For each Requirement found: read priority, source, acceptance_criteria
  5. Answer: "X exists because Requirement REQ-NNN (priority: MUST, source: S)
              governs it. Decision ADR-NNN chose this approach because: {rationale}"
  CONFIDENCE = min(decision.confidence, requirement.confidence)
```

---

## R-3 Impact Reasoning

```
IMPACT(object_id):
  1. ANCESTORS = all objects with depends_on → ... → object_id
  2. Group by object_type and namespace
  3. Compute blast_radius = len(ANCESTORS)
  4. Classify severity:
     CRITICAL  if blast_radius >= 26
     HIGH      if blast_radius >= 11
     MEDIUM    if blast_radius >= 4
     LOW       if blast_radius >= 1
  5. Return ImpactReport{
       object_id, blast_radius, severity,
       affected_by_type: dict, affected_by_namespace: dict,
       critical_path: list[str]   # highest-weight path
     }
```

---

## R-4 Risk Reasoning

```
RISK(object_id):
  Score components:
    evidence_gap    = 1.0 - evidence_score           (weight 0.30)
    confidence_risk = 1.0 - composite_confidence     (weight 0.25)
    freshness_risk  = 1.0 - average_freshness        (weight 0.20)
    impact_risk     = normalized(blast_radius / 26)  (weight 0.25)

  risk_score = Σ(component * weight)

  Risk levels:
    CRITICAL: risk_score >= 0.80
    HIGH:     risk_score >= 0.60
    MEDIUM:   risk_score >= 0.40
    LOW:      risk_score < 0.40
```

---

## R-7 Alternatives Reasoning

```
ALTERNATIVES(object_id):
  1. Find objects related_to or conflicts_with object_id
  2. Find semantically similar objects (similarity > 0.75)
  3. Find objects in same namespace with same object_type
  4. Filter: only include VERIFIED or above
  5. Rank by quality_score DESC
  6. Return top 5 with comparison table:
     {field: [object_A_value, object_B_value], ...}
```

---

## R-8 Tradeoffs Reasoning

```
TRADEOFFS(object_a_id, object_b_id):
  Load both objects
  Compare across dimensions:
    - quality scores (all 9 QD)
    - evidence counts and types
    - confidence levels
    - relationship counts (usage)
    - lifecycle maturity
    - performance benchmarks (if benchmarks exist)
  Return: ComparisonMatrix{
    dimensions: list[str],
    a_values: list, b_values: list,
    winner_per_dimension: dict,
    recommendation: str,
    reasoning: str
  }
```

---

## Reasoning Engine Protocol

```python
class KnowledgeReasoningEngine(Protocol):
    def why(self, object_id: str) -> ReasoningAnswer: ...
    def how(self, object_id: str) -> ReasoningAnswer: ...
    def impact(self, object_id: str) -> ImpactReport: ...
    def risk(self, object_id: str) -> RiskReport: ...
    def dependencies(self, object_id: str) -> DependencyReport: ...
    def history(self, object_id: str, since: datetime | None) -> HistoryReport: ...
    def alternatives(self, object_id: str) -> list[KnowledgeObject]: ...
    def tradeoffs(self, id_a: str, id_b: str) -> ComparisonMatrix: ...
    def decisions(self, object_id: str) -> list[DecisionObject]: ...
    def plans(self, object_id: str) -> list[ExecutionPlanObject]: ...
    def reason(self, object_id: str, reasoning_type: ReasoningType) -> ReasoningAnswer: ...
```

---

## CLI Integration

```bash
platform-kos why KNW-KOS-ARCH-001
platform-kos impact KNW-PLAT-SVC-001
platform-kos risk KNW-AI-MOD-quota_manager
platform-kos tradeoffs KNW-AI-MOD-001 KNW-AI-MOD-002
platform-kos history KNW-KOS-DEC-001 --since 2026-01-01
```

---

## Cross-References

- Evidence consumed by R-1, R-4 → `10-EVIDENCE-ENGINE`
- Impact algorithm (R-3) → `25-GRAPH-ALGORITHMS`
- Version history (R-6) → `08-VERSION-ENGINE`
- Semantic similarity for R-7 → `29-SEMANTIC-LAYER`
- Confidence model for scoring → `12-CONFIDENCE-MODEL`
