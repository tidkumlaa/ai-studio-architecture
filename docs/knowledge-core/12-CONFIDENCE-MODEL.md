# KNW-KC-ARCH-012 — Confidence Model

**Phase:** 3.0C  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

The Confidence Model defines how certainty is quantified, propagated, and decayed across Knowledge Objects and their relationships. Confidence is a first-class property — not a comment, not a metadata field, but a computed, evidence-backed score that drives lifecycle decisions.

---

## Confidence Dimensions

Every Knowledge Object carries three confidence values:

| Field | Description | Range |
|-------|-------------|-------|
| `declared_confidence` | Owner's subjective confidence at time of writing | 0.0–1.0 |
| `evidence_confidence` | Computed from evidence records (10-EVIDENCE-ENGINE) | 0.0–1.0 |
| `composite_confidence` | Weighted combination of declared and evidence | 0.0–1.0 |

```
composite_confidence =
  declared_confidence * 0.30
  + evidence_confidence * 0.70
```

---

## Confidence Levels

| Level | Range | Interpretation |
|-------|-------|---------------|
| SPECULATIVE | 0.00–0.30 | Hypothesis — not yet validated |
| TENTATIVE | 0.31–0.55 | Partially validated — use with caution |
| SUPPORTED | 0.56–0.75 | Evidence-backed — acceptable for VERIFIED |
| STRONG | 0.76–0.90 | Well-evidenced — acceptable for APPROVED |
| DEFINITIVE | 0.91–1.00 | Fully proven — acceptable for CANONICAL |

---

## Confidence Propagation

### Through Relationships
When object A `depends_on` object B:
```
A's dependency confidence = min(A.composite_confidence, B.composite_confidence)
```

When computing overall object confidence with dependencies:
```
object_confidence_with_deps =
  A.composite_confidence * 0.6
  + min(dep.composite_confidence for dep in A.hard_dependencies) * 0.4
```

### Transitive Propagation
For transitive relationships (dependency chains):
```
confidence(A→C via B) = confidence(A→B) * confidence(B→C)

Example:
  A depends_on B: confidence 0.90
  B depends_on C: confidence 0.80
  A→C transitive:  confidence 0.72
```

### Confidence Floor
No object may reach CANONICAL with `composite_confidence < 0.80`.

---

## Confidence Decay

Confidence decays when evidence becomes stale or when the world changes:

```
decayed_confidence(t) =
  composite_confidence
  * evidence_freshness_average
  * recency_factor(t)

recency_factor(t) =
  1.0                                    if updated_within_half_life
  exp(-0.693 * (age - half_life) / half_life)   otherwise

Domain half-lives:
  ARCHITECTURE:  365 days
  MODULE:        90 days
  TEST:          14 days
  MARKET:        1 hour
  FOREX:         15 minutes
```

---

## Confidence Inheritance Rules

### CI-001 — No Fabrication
`declared_confidence` must not exceed the maximum supported by submitted evidence.
```
declared_confidence ≤ max_evidenced_confidence + 0.10   (10% grace)
```
Violations are reported as `QA-007 OVERCONFIDENT`.

### CI-002 — Relationship Confidence Bound
The confidence of a relationship is bounded by the confidence of its two endpoints:
```
relation.confidence ≤ min(source.composite_confidence, target.composite_confidence)
```

### CI-003 — AI Approval Confidence Cap
Objects endorsed only by AI approval (`EV-AAPPROVAL`) may not exceed:
```
composite_confidence ≤ 0.75   (AI-only endorsement cap)
```
Human approval (`EV-HAPPROVAL`) is required to exceed 0.75.

### CI-004 — Production Confidence Boost
Objects with `EV-PROD` evidence receive a 0.05 confidence bonus (capped at 1.0).

---

## Confidence in the Knowledge Graph

Graph edges carry confidence. When reasoning across the graph:

```
PATH_CONFIDENCE(path = [node_1, edge_1, node_2, edge_2, ..., node_n]):
  = PRODUCT(edge.confidence for edge in path)
  * PRODUCT(node.composite_confidence for node in path)^(1/n)

This is the confidence that the entire chain of reasoning is valid.
```

---

## Confidence Reporting Schema

```yaml
confidence_report:
  knowledge_id: "KNW-KOS-ARCH-001"
  computed_at: datetime
  declared_confidence: 0.85
  evidence_confidence: 0.78
  composite_confidence: 0.801    # 0.85*0.30 + 0.78*0.70
  decayed_confidence: 0.791      # after freshness decay
  level: STRONG
  dependency_min_confidence: 0.72
  effective_confidence: 0.762    # composite * 0.6 + dep_min * 0.4
  alerts:
    - CI-001_OVERCONFIDENT: false
    - CI-003_AI_CAP_EXCEEDED: false
  trend: STABLE                  # RISING|STABLE|DECLINING|CRITICAL
```

---

## Confidence Engine Protocol

```python
class ConfidenceModel(Protocol):
    def compute_composite(self, object_id: str) -> float: ...
    def compute_decayed(self, object_id: str) -> float: ...
    def compute_path_confidence(self, path: list[str]) -> float: ...
    def propagate_through_deps(self, object_id: str) -> float: ...
    def get_level(self, confidence: float) -> ConfidenceLevel: ...
    def check_ci_rules(self, object_id: str) -> list[ConfidenceViolation]: ...
    def get_trend(self, object_id: str, days: int) -> list[float]: ...
```

---

## Confidence Alerts

| Code | Condition |
|------|-----------|
| `CI-001 OVERCONFIDENT` | declared > evidenced + 0.10 |
| `CI-002 REL_CONFIDENCE_EXCEEDED` | relation.confidence > min(source, target) |
| `CI-003 AI_CAP_EXCEEDED` | AI-only object with confidence > 0.75 |
| `CI-004 CRITICAL_DECAY` | decayed_confidence < 0.30 |
| `CI-005 DEP_CHAIN_BROKEN` | dependency chain composite < 0.40 |

---

## Cross-References

- Evidence confidence input → `10-EVIDENCE-ENGINE`
- Quality uses confidence → `11-QUALITY-ENGINE`
- Lifecycle gating → `34-STATE-MACHINES`
- Graph path confidence → `24-KNOWLEDGE-GRAPH-MODEL`
- Reasoning model uses confidence → `30-REASONING-MODEL`
