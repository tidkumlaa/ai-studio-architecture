---
knowledge_id: KA-SPEC-012
version: "1.0.0"
status: approved
owner: Chief Knowledge Architect
phase: 2.0D.0.5
created: 2026-06-29
review_date: 2026-12-29
canonical: true
domain: DOM-KNOWLEDGE
capability: knowledge-core
type: specification
implements:
  - KA-SPEC-001
depends_on:
  - id: KA-SPEC-011
    reason: "Evidence terminates the traceability chain"
---

# Knowledge Evidence Model

## Proof That Architecture Exists in Reality

---

## 1. Purpose

Evidence is the bridge between architecture and reality.

An architecture document without evidence is a plan. An architecture document with verified evidence is a fact. The Evidence Model defines what constitutes valid evidence, how evidence is collected and validated, how confidence is derived from evidence, and how evidence degrades over time.

---

## 2. Evidence Types

| Type | Description | Example |
|------|-------------|---------|
| `source_file` | A platform source code file that implements the architecture | `platform/workflow-runtime/src/executor.py` |
| `test` | A test file that verifies the implementation | `platform/workflow-runtime/tests/test_executor.py` |
| `schema` | A database schema file that implements the data model | `platform/workflow-runtime/db/schema.sql` |
| `config` | A configuration file that demonstrates deployment | `platform/workflow-runtime/config/default.yaml` |
| `log` | A validation or test run log | `platform/workflow-runtime/test-results/coverage.xml` |
| `adr` | An Architecture Decision Record that justifies the design | `architecture/adr/KA-ADR-012.md` |

---

## 3. Evidence Record Schema

```yaml
evidence:
  - type: source_file                    # Required
    ref: platform/workflow-runtime/src/executor.py  # Required — relative path
    line: 42                             # Optional — specific line
    verified: true                       # Required
    verified_by: "Capability Architect"  # Optional — role
    last_verified: 2026-06-28            # Optional but recommended
    notes: "AsyncExecutor class at line 42"  # Optional
    confidence: high                     # Optional — inferred if not set
    expiry_days: 90                      # Optional — evidence expiry window
```

---

## 4. Evidence Lifecycle

```
proposed → collected → validated → verified → expired
                                      ↑           ↓
                                      └── re-verification
```

| State | Description | Transitions |
|-------|-------------|------------|
| `proposed` | Referenced but not yet checked | → collected when ref confirmed to exist |
| `collected` | File/artifact exists | → validated when content is confirmed |
| `validated` | Content matches architecture claim | → verified when automated check passes |
| `verified` | Automated tool confirmed match | → expired when expiry_days elapsed |
| `expired` | Past expiry — needs re-verification | → verified on re-check |

---

## 5. Evidence Confidence Mapping

Evidence drives the `confidence` field of the owning KnowledgeObject:

| Evidence State | Count | Resulting Confidence |
|---------------|-------|---------------------|
| No evidence | — | `low` |
| 1+ collected (not verified) | — | `medium` |
| 1+ verified source_file | ≥ 1 | `high` |
| 1+ verified source_file + 1+ verified test | ≥ 2 | `high` (full) |
| Any verified evidence expired | — | Downgrade to `medium` |
| All evidence expired | — | Downgrade to `low` |

**Rule:** If `confidence: high` is declared but no verified evidence exists, the audit tool issues a warning (RULE-M009B).

---

## 6. Evidence Validation Protocol

### 6.1 Existence Validation

The audit tool performs existence validation for all evidence references:

```
FOR each evidence record:
  IF evidence.type in [source_file, test, schema, config, log]:
    IF NOT file_exists(evidence.ref):
      REPORT BROKEN_EVIDENCE(obj.id, evidence.ref)
      SET evidence.verified = false
      SET evidence.state = broken
```

### 6.2 Content Validation

Content validation checks whether the referenced artifact actually implements what the architecture claims. This is a semantic check, not syntactic.

**Level 1 — Structural match** (automated):
- Architecture claims class `AsyncExecutor` exists → grep for `class AsyncExecutor` in source_file
- Architecture claims function `execute_step` exists → grep for `def execute_step`
- Architecture claims event `workflow.started` is emitted → grep for event name in source

**Level 2 — Semantic match** (AI-assisted, Phase 2.0D.2+):
- AI reads the architecture section and the implementation file
- AI determines whether the implementation matches the design intent
- AI produces a match score (0–100) and a summary of differences

### 6.3 Test Validation

Test validation connects a test file to the coverage percentage:

```
FOR each evidence record WHERE type = 'test':
  IF test_results_available:
    actual_coverage = parse_coverage_report(evidence.ref)
    IF actual_coverage != expected_coverage:
      REPORT COVERAGE_MISMATCH(obj.id, expected, actual)
  SET evidence.coverage_percent = actual_coverage
  SET evidence.verified = true
  SET evidence.last_verified = today()
```

---

## 7. Evidence Expiry

Evidence expires because implementations change. An evidence record that was verified in January may no longer be valid in June if the implementation was refactored.

### 7.1 Default Expiry Windows

| Evidence Type | Default Expiry |
|---------------|---------------|
| `source_file` | 90 days |
| `test` | 30 days |
| `schema` | 180 days |
| `config` | 180 days |
| `log` | 30 days |
| `adr` | Never (ADRs are immutable) |

### 7.2 Expiry Impact on Health Score

When evidence expires, the health score of the owning KnowledgeObject degrades:

```
IF evidence.last_verified + evidence.expiry_days < today():
  evidence.state = expired
  health_score.evidence_dimension -= 5 per expired record
  IF all evidence expired:
    confidence = downgrade(confidence)
```

### 7.3 Expiry Re-verification

Re-verification is triggered automatically when:
- A pre-commit hook detects changes to referenced files
- The audit tool runs in full mode (`--verify-evidence`)
- The owner manually re-verifies via the Documentation Intelligence Platform

---

## 8. Evidence Registry

All evidence records across the repository are aggregated in `architecture/evidence-registry.yaml`:

```yaml
generated: 2026-06-29
total_evidence_records: 247
  verified: 183
  unverified: 41
  expired: 23
  broken: 0

by_type:
  source_file: 120
  test: 89
  schema: 24
  config: 14

registry:
  KA-ARCH-001:
    evidence:
      - type: source_file
        ref: platform/workflow-runtime/src/executor.py
        state: verified
        last_verified: 2026-06-28
        confidence_contribution: high

  KA-ARCH-005:
    evidence:
      - type: source_file
        ref: platform/provider-framework/src/adapter.py
        state: expired
        last_verified: 2026-03-01
        confidence_contribution: degraded
```

---

## 9. Evidence in the Graph

Evidence records create edges in the knowledge graph:

```
KA-ARCH-001 ──[implemented_by]──▶ platform/workflow-runtime/src/executor.py
                                    (EdgeProperties: verified=true, confidence=high)

KA-ARCH-001 ──[tested_by]──▶ platform/workflow-runtime/tests/test_executor.py
                                (EdgeProperties: coverage_percent=78, verified=true)
```

These edges connect the architecture graph to the implementation world, enabling full traceability traversal.

---

## References

- [11-KNOWLEDGE-TRACEABILITY-MODEL.md](11-KNOWLEDGE-TRACEABILITY-MODEL.md) — Evidence terminates traceability chains
- [13-KNOWLEDGE-COVERAGE-MODEL.md](13-KNOWLEDGE-COVERAGE-MODEL.md) — Coverage derived from evidence
- [KA-STD-006](../knowledge-architecture/KNOWLEDGE-HEALTH-METRICS.md) — Health scoring using evidence
