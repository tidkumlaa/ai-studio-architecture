# KNW-KC-ARCH-010 — Evidence Engine

**Phase:** 3.0C  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Every Knowledge Object must carry verifiable evidence for its existence, correctness, and relevance. The Evidence Engine defines how evidence is recorded, scored, and used to gate lifecycle transitions.

---

## Evidence Types (10)

| Code | Type | Description | Weight |
|------|------|-------------|--------|
| `EV-DOC` | Document | Written specification, RFC, or design document | 0.60 |
| `EV-SPEC` | Specification | Formal spec (OpenAPI, JSON Schema, Protobuf, etc.) | 0.80 |
| `EV-CODE` | Code | Implemented and merged source code | 0.90 |
| `EV-TEST` | Test | Passing automated test suite | 0.85 |
| `EV-BENCH` | Benchmark | Performance measurement within budget | 0.80 |
| `EV-REVIEW` | Review | Human review record with reviewer identity | 0.70 |
| `EV-HAPPROVAL` | Human Approval | Named person approved this object | 0.95 |
| `EV-AAPPROVAL` | AI Approval | AI system endorsed with confidence ≥ 0.85 | 0.65 |
| `EV-RUNTIME` | Runtime Metrics | Live system measurements | 0.90 |
| `EV-PROD` | Production Observation | Behaviour observed in production for ≥ 30 days | 1.00 |

---

## Evidence Record Schema

```yaml
evidence_record:
  evidence_id: "EV-{type_code}-{knowledge_id}-{NNN}"
  evidence_type: EV-DOC|EV-SPEC|...
  object_id: string              # knowledge_id this evidence supports
  claim: string                  # what does this evidence prove?

  # Source
  source_ref: string             # URI, file path, or knowledge_id of the evidence artifact
  source_type: FILE|URL|TEST_RUN|REVIEW_RECORD|APPROVAL_RECORD|METRIC_REPORT
  source_captured_at: datetime
  captured_by: string

  # Scoring
  confidence: float              # 0.0..1.0 — how strongly does this prove the claim?
  freshness_score: float         # decays over time — see Freshness Model below
  verification_score: float      # computed from confidence * weight * freshness
  verified_by: string | null
  verified_at: datetime | null

  # Lifecycle
  status: PROPOSED|ACTIVE|STALE|EXPIRED|DISPUTED
  expires_at: datetime | null
  dispute_reason: string

  # Metadata
  notes: string
  tags: list[string]
```

---

## Evidence Score Computation

```
evidence_score(object) =
  Σ(evidence_record.verification_score * evidence_type.weight)
  ─────────────────────────────────────────────────────────────
  Σ(evidence_type.weight for all evidence records)

Where:
  verification_score = confidence * freshness_score
  freshness_score    = see Freshness Model below
```

---

## Freshness Model

Evidence decays over time unless refreshed:

```
freshness_score(evidence_record) =
  1.0                                          if age_days <= freshness_threshold
  max(0.0, 1.0 - (age_days - threshold) / decay_period)

Freshness thresholds by evidence type:
  EV-DOC:        fresh for 180 days, decays over 360 days
  EV-SPEC:       fresh for 365 days, decays over 365 days
  EV-CODE:       fresh for 90 days, decays over 180 days
  EV-TEST:       fresh for 7 days (CI must re-run), decays over 30 days
  EV-BENCH:      fresh for 30 days, decays over 60 days
  EV-REVIEW:     fresh for 90 days, decays over 180 days
  EV-HAPPROVAL:  fresh for 365 days, decays over 365 days
  EV-AAPPROVAL:  fresh for 30 days, decays over 60 days
  EV-RUNTIME:    fresh for 1 day, decays over 7 days
  EV-PROD:       fresh for 60 days, decays over 120 days
```

---

## Evidence Requirements by Lifecycle Status

| Status | Required Evidence | Minimum evidence_score |
|--------|-----------------|----------------------|
| DRAFT | None | 0.0 |
| REVIEW | ≥ 1 EV-DOC or EV-SPEC | 0.30 |
| VERIFIED | ≥ 1 EV-CODE or EV-TEST | 0.55 |
| APPROVED | ≥ 1 EV-HAPPROVAL | 0.70 |
| CANONICAL | ≥ 1 EV-HAPPROVAL + EV-TEST + EV-CODE | 0.80 |

---

## Evidence for Decisions (Special Case)

`DecisionObject` and `RequirementObject` have heightened evidence requirements:

```yaml
decision_evidence:
  mandatory:
    - context_document: EV-DOC     # the problem context
    - options_comparison: EV-DOC   # alternatives considered
    - rationale_document: EV-DOC   # reasoning
  for_canonical:
    - human_approval: EV-HAPPROVAL
    - review_record: EV-REVIEW
```

---

## Evidence Dispute Protocol

```
DISPUTE(evidence_id, reason, disputed_by):
  1. Set evidence_record.status = DISPUTED
  2. Set dispute_reason = reason
  3. Emit EVIDENCE_DISPUTED event
  4. If disputed evidence was load-bearing for CANONICAL status:
     a. Object status → APPROVED (one step back)
     b. Owner notified
  5. Resolution: REVIEW_PANEL decides:
     UPHELD → evidence removed, re-evaluate object quality
     DISMISSED → evidence restored, status unchanged
```

---

## Evidence Engine Protocol

```python
class EvidenceEngine(Protocol):
    def add_evidence(
        self, object_id: str, record: EvidenceRecord
    ) -> str: ...                                          # evidence_id

    def get_evidence(self, object_id: str) -> list[EvidenceRecord]: ...

    def compute_evidence_score(self, object_id: str) -> float: ...

    def compute_freshness(self, evidence_id: str) -> float: ...

    def check_requirements(
        self, object_id: str, target_status: LifecycleStatus
    ) -> EvidenceCheckResult: ...                          # PASS|FAIL(missing)

    def dispute(
        self, evidence_id: str, reason: str, disputed_by: str
    ) -> DisputeRecord: ...

    def refresh(self, evidence_id: str, new_source_ref: str) -> EvidenceRecord: ...

    def expire(self, evidence_id: str) -> None: ...
```

---

## Evidence Events

| Event | Trigger | Recipients |
|-------|---------|-----------|
| `evidence.added` | New record created | Object owner |
| `evidence.stale` | freshness_score < 0.3 | Object owner |
| `evidence.expired` | freshness_score = 0.0 | Object owner + reviewers |
| `evidence.disputed` | Dispute filed | Object owner + approvers |
| `evidence.resolved` | Dispute resolved | Object owner |

---

## Cross-References

- Quality computation uses evidence score → `11-QUALITY-ENGINE`
- Lifecycle transitions gate on evidence → `34-STATE-MACHINES`
- Evidence stored in Universal Schema → `02-UNIVERSAL-SCHEMA`
- Evidence for decisions → Phase 3.0A `11-GOVERNANCE`
