---
knowledge_id: KA-KIP-009
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
  - id: KA-SPEC-012
    reason: "Evidence model specification"
---

# Evidence Engine

## Registry · Ranking · Verification · Aggregation · Confidence

---

## 1. Evidence Types

| Type | Description | Auto-detectable |
|------|-------------|----------------|
| `source_file` | Implementation file at declared path | Yes |
| `test` | Test file with coverage threshold | Yes |
| `schema` | Schema file (JSON/YAML) | Yes |
| `config` | Configuration file | Yes |
| `log` | CI/CD log showing passing test | No |
| `adr` | Architecture Decision Record | Yes |

---

## 2. Evidence Record

```python
@dataclass
class EvidenceRecord:
    evidence_id:   str        # EV-001, EV-002, ...
    knowledge_id:  str        # KA-ARCH-001
    evidence_type: str        # source_file, test, schema, ...
    source_path:   str        # Relative file path or URL
    description:   str
    state:         str        # proposed → collected → validated → verified → expired
    confidence:    float      # 0.0 - 1.0
    created:       str
    verified_date: str | None
    expires_days:  int        # 90 for tests, 180 for source, 365 for ADR
    metadata:      dict
```

---

## 3. Evidence Lifecycle FSM

```
proposed → collected → validated → verified → expired
                                       ↑
                               (re-verify on expiry)
```

| Transition | Guard | Action |
|-----------|-------|--------|
| proposed → collected | File exists | Set state=collected |
| collected → validated | Content check passes | Set confidence |
| validated → verified | Manual or CI approval | Set verified_date |
| verified → expired | expires_days elapsed | Set state=expired |
| expired → collected | File still exists | Reset for re-validation |

---

## 4. Evidence Registry

```python
class EvidenceRegistry:
    # HashMap<knowledge_id, list[EvidenceRecord]>

    def register(self, record: EvidenceRecord) -> None
    def get(self, knowledge_id: str) -> list[EvidenceRecord]
    def get_by_id(self, evidence_id: str) -> EvidenceRecord | None
    def all(self) -> list[EvidenceRecord]
    def by_state(self, state: str) -> list[EvidenceRecord]
    def by_type(self, evidence_type: str) -> list[EvidenceRecord]
    def expired(self) -> list[EvidenceRecord]
    def next_id(self) -> str
```

---

## 5. Evidence Verification

Two levels of verification:

**Level 1 — Existence check (automated):**
```
VERIFY_EXISTENCE(record):
  IF record.evidence_type in {source_file, test, schema, config}:
    IF Path(record.source_path).exists(): record.state = "collected"
    ELSE: record.state = "proposed"; ADD warning
```

**Level 2 — Content check (automated):**
```
VERIFY_CONTENT(record):
  IF record.evidence_type == "test":
    content = read(record.source_path)
    IF "def test_" in content OR "@pytest" in content:
      coverage = extract_coverage_percent(content)
      record.confidence = min(coverage / 100.0, 1.0)
      record.state = "validated"
  
  IF record.evidence_type == "source_file":
    content = read(record.source_path)
    IF knowledge_id_mentioned(content, record.knowledge_id):
      record.confidence = 0.80
      record.state = "validated"
```

---

## 6. Evidence Ranking

Evidence items are ranked by confidence for display and aggregation:

```
RANK_EVIDENCE(records):
  # Sort by: (state_priority, confidence, recency)
  state_priority = {verified: 4, validated: 3, collected: 2, proposed: 1, expired: 0}
  
  RETURN sorted(records, key=lambda r: (
    state_priority[r.state],
    r.confidence,
    r.verified_date or ""
  ), reverse=True)
```

---

## 7. Confidence Aggregation

The confidence of a knowledge object is derived from its evidence:

```
AGGREGATE_CONFIDENCE(knowledge_id, evidence_registry):
  records = evidence_registry.get(knowledge_id)
  IF NOT records: RETURN 0.0
  
  # Use only non-expired evidence
  active = [r for r in records if r.state != "expired"]
  IF NOT active: RETURN 0.05  # Tiny residual — was once evidenced
  
  # Weighted average: higher weight for verified and higher-confidence items
  weights = [r.confidence * state_weight[r.state] for r in active]
  total_weight = sum(weights)
  IF total_weight == 0: RETURN 0.0
  
  weighted_conf = sum(r.confidence * w for r, w in zip(active, weights))
  RETURN min(weighted_conf / total_weight, 1.0)
```

---

## 8. Interface

```python
class EvidenceEngine:
    def register(self, record: EvidenceRecord) -> None
    def get(self, knowledge_id: str) -> list[EvidenceRecord]
    def verify_all(self, root_path: Path) -> VerificationReport
    def verify_one(self, evidence_id: str, root_path: Path) -> VerificationResult
    def rank(self, knowledge_id: str) -> list[EvidenceRecord]
    def confidence(self, knowledge_id: str) -> float
    def aggregate_for_capability(self, capability: str) -> float
    def expired(self) -> list[EvidenceRecord]
    def missing(self, registry: KnowledgeRegistry) -> list[str]
        # Returns knowledge_ids with approved status but no evidence

    def load_from_yaml(self, path: Path) -> None
    def save_to_yaml(self, path: Path) -> None
    def report(self) -> str  # Markdown evidence report
```

---

## References

- [KA-SPEC-012](../knowledge-core/12-KNOWLEDGE-EVIDENCE-MODEL.md) — Evidence model
- [05-KNOWLEDGE-REASONING.md](05-KNOWLEDGE-REASONING.md) — Evidence propagation
- [07-KNOWLEDGE-HEALTH-RUNTIME.md](07-KNOWLEDGE-HEALTH-RUNTIME.md) — Evidence dimension
