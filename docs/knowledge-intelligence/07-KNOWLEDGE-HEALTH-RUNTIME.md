---
knowledge_id: KA-KIP-008
version: "1.0.0"
status: approved
owner: Chief Knowledge Architect
phase: 2.0D.2
created: 2026-06-29
review_date: 2026-12-29
canonical: true
domain: DOM-KNOWLEDGE
capability: knowledge-intelligence
type: specification
depends_on:
  - id: KA-STD-006
    reason: "6-dimension health scoring definition"
  - id: KA-SPEC-017
    reason: "ALG-007 health computation algorithm"
---

# Knowledge Health Runtime

## Live 6-Dimension Scoring · Knowledge Debt · Dashboard Backend

---

## 1. Purpose

The Health Runtime is the real-time health computation engine. Unlike the batch
Health Analyzer in the Migration Platform, this component:

- Maintains a live `HealthIndex` updated on every registry change
- Computes health incrementally (O(1) per document update)
- Serves the health dashboard backend
- Detects knowledge debt and generates remediation priorities

---

## 2. HealthIndex Structure

```
HealthIndex:
  scores:        SortedList[(score, id)]  # O(log n) insert, O(1) min/max
  by_id:         HashMap<id, HealthScore>  # O(1) lookup
  by_capability: HashMap<cap, float>       # Aggregated per capability
  by_domain:     HashMap<domain, float>    # Aggregated per domain
  repo_score:    float                     # Repository-level aggregate
  last_updated:  str
```

SortedList enables O(1) retrieval of worst-scoring documents for dashboard display.

---

## 3. Health Dimensions (ALG-007 from KA-SPEC-017)

| Dimension | Weight | Max | Measured By |
|-----------|--------|-----|-------------|
| Structural | 25% | 25 | Frontmatter field presence |
| Relationship | 20% | 20 | Declared relationships + broken link count |
| Freshness | 20% | 20 | Days since review_date |
| Evidence | 15% | 15 | Evidence record existence + verified state |
| Coverage | 10% | 10 | Architecture/implementation/testing declared |
| Consistency | 10% | 10 | Version format, ID format, no multi-canonical |

---

## 4. Incremental Health Update

When a single document changes, only that document and its capability aggregate
need recomputation — not the full repository:

```
ON_DOCUMENT_CHANGE(id, updated_obj):
  new_score = compute_health(updated_obj, graph, evidence_registry)
  old_score = health_index.by_id.get(id)
  
  health_index.by_id[id] = new_score
  health_index.scores.remove((old_score.total, id))
  health_index.scores.add((new_score.total, id))
  
  # Recompute affected capability aggregate
  cap = updated_obj.capability
  cap_scores = [health_index.by_id[kid].total
                for kid in catalog.by_capability(cap)]
  health_index.by_capability[cap] = mean(cap_scores)
  
  # Recompute repository score
  health_index.repo_score = compute_repo_score(health_index)

Complexity: O(log D) where D = document count
```

---

## 5. Health Metrics

Beyond the 6 dimensions, the runtime tracks additional quality signals:

| Metric | Description | Target |
|--------|-------------|--------|
| Relationship Density | Avg edges per document | ≥ 3.0 |
| Broken References | Links to non-existent IDs | 0 |
| Duplicate Ratio | Exact content duplicates / total | ≤ 2% |
| Missing Metadata | Docs without frontmatter | 0 |
| Stale Reviews | review_date > 180 days ago | ≤ 5% |
| Orphan Ratio | Docs with 0 inbound references | ≤ 10% |
| Evidence Coverage | Docs with ≥ 1 verified evidence | ≥ 60% |

---

## 6. Knowledge Debt

Knowledge Debt quantifies remediation effort:

```python
@dataclass
class KnowledgeDebt:
    score:                    float   # 0-100 (inverse of health)
    documents_without_meta:   int
    documents_oversized:      int
    broken_links:             int
    orphans:                  int
    stale_reviews:            int
    missing_evidence:         int
    duplicate_canonicals:     int
    estimated_days:           float   # Person-days to remediate

    # Debt breakdown by capability
    by_capability: dict[str, float]
    # Prioritized remediation list
    priority_list: list[DebtItem]

@dataclass(frozen=True)
class DebtItem:
    knowledge_id: str
    issue:        str
    effort_days:  float
    impact:       float   # Health score gain if fixed
    priority:     int     # 1 = highest
```

---

## 7. Dashboard Backend Interface

The health runtime exposes these endpoints to the dashboard:

```python
class HealthRuntime:
    # Aggregate metrics
    def repo_score(self) -> float
    def capability_scores(self) -> dict[str, float]
    def domain_scores(self) -> dict[str, float]

    # Worst performers
    def worst_documents(self, n: int = 20) -> list[HealthScore]
    def worst_capabilities(self, n: int = 10) -> list[tuple[str, float]]

    # Debt
    def knowledge_debt(self) -> KnowledgeDebt
    def priority_list(self) -> list[DebtItem]

    # Live updates
    def on_change(self, id: str, obj: KnowledgeObject) -> None
    def recompute_all(self) -> None   # Full rebuild

    # Phase gate
    def phase_gate_met(self) -> bool  # score >= 80
    def phase_gate_gap(self) -> float

    # Trends (if historical snapshots available)
    def trend(self, days: int = 30) -> list[tuple[str, float]]  # (date, score)
```

---

## 8. Health Report Generation

```python
def generate_health_report(runtime: HealthRuntime) -> str:
    """Generate HEALTH-REPORT.md from runtime state."""
    # Sections:
    # 1. Executive summary (score, phase gate, debt)
    # 2. By-capability table (sorted worst to best)
    # 3. Top 20 worst documents
    # 4. Knowledge debt breakdown
    # 5. Priority remediation list
    # 6. Dimension breakdown chart (ASCII)
```

---

## References

- [KA-STD-006](../knowledge-architecture/KNOWLEDGE-HEALTH-METRICS.md) — Health scoring
- [KA-SPEC-017](../knowledge-core/17-ALGORITHMS-CATALOG.md) — ALG-007
- [08-EVIDENCE-ENGINE.md](08-EVIDENCE-ENGINE.md) — Evidence dimension input
