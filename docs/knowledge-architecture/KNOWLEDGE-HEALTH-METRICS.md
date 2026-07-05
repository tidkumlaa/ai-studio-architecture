---
knowledge_id: KA-STD-006
version: "1.0.0"
status: approved
owner: Chief Knowledge Architect
phase: 2.0D.0
created: 2026-06-29
review_date: 2026-12-29
canonical: true
implements:
  - KA-VISION-001
---

# Knowledge Health Metrics

## Measuring the Health of the Knowledge System

---

## 1. Why Health Metrics Matter

Without metrics, documentation health is a matter of opinion. With metrics, it is a matter of fact.

Health metrics answer:
- Which capabilities are well-documented and which are at risk?
- Which documents are stale and by how long?
- Where are the coverage gaps?
- What is the knowledge debt?
- Which changes are safe to make right now?

The health system produces **computable scores** from **machine-readable metadata**. No human judgment is required to generate a health report.

---

## 2. Health Score Architecture

### 2.1 Score Hierarchy

```
Repository Health Score (0–100)
├── Domain Health Score (0–100) × N domains
│   ├── Capability Health Score (0–100) × N capabilities
│   │   └── Document Health Score (0–100) × N documents
```

A document's score rolls up to its capability. A capability's average rolls up to its domain. Domains average to the repository score.

### 2.2 Weighting

Document health scores use **dimension weights**:

| Dimension | Weight | What It Measures |
|-----------|--------|-----------------|
| Structural completeness | 25% | Required fields in frontmatter |
| Relationship completeness | 20% | Declared relationships |
| Freshness | 20% | Distance from review_date |
| Evidence | 15% | Implementation verification |
| Coverage | 10% | Coverage percentages in frontmatter |
| Consistency | 10% | No broken references, no duplicate canonicals |

---

## 3. Document Health Score (0–100)

### 3.1 Dimension 1: Structural Completeness (25 points)

Measures whether all required frontmatter fields are present and valid.

| Field | Points |
|-------|--------|
| `knowledge_id` present and unique | 4 |
| `version` present and valid semver | 2 |
| `status` present and valid | 2 |
| `canonical` declared | 2 |
| `domain` present and valid | 2 |
| `capability` present and valid | 2 |
| `type` present and valid | 2 |
| `owner` present | 3 |
| `created` present and valid date | 2 |
| `review_date` present and valid date | 2 |
| `phase` present | 2 |

**Penalty:** -1 per additional missing optional field from the full schema.

### 3.2 Dimension 2: Relationship Completeness (20 points)

Measures whether the document declares its connections to other knowledge objects.

| Condition | Points |
|-----------|--------|
| `implements` declared (if applicable) | 4 |
| `depends_on` declared (if applicable) | 4 |
| `implemented_by` declared (for architecture docs) | 4 |
| `related_adr` declared (for architecture docs) | 4 |
| `consumed_by` declared (for capability indexes) | 4 |

**Partial credit:** Each relationship type is worth its full points only if at least one entry is declared.

### 3.3 Dimension 3: Freshness (20 points)

Measures whether the document has been reviewed within its scheduled review window.

| Condition | Score |
|-----------|-------|
| `review_date` is in the future | 20 |
| `review_date` is 0–30 days past | 14 |
| `review_date` is 31–60 days past | 8 |
| `review_date` is 61–90 days past | 4 |
| `review_date` is 91–180 days past | 1 |
| `review_date` is more than 180 days past | 0 |

**ADR exception:** ADRs are immutable and have no `review_date`. They always score 20 on freshness.
**History exception:** History documents are append-only. They always score 20 on freshness.

### 3.4 Dimension 4: Evidence (15 points)

Measures whether architecture documents have verifiable implementation evidence.

| Condition | Points |
|-----------|--------|
| At least one `evidence` entry declared | 5 |
| At least one evidence entry with `verified: true` | 5 |
| Evidence `last_verified` within 90 days | 5 |

**Type exception:** Vision, ADR, roadmap, history, and references documents are not scored on evidence.
**Confidence alignment:** If `confidence: high` but no evidence, score is capped at 8/15.

### 3.5 Dimension 5: Coverage (10 points)

Measures the `coverage` object in frontmatter.

| Condition | Points |
|-----------|--------|
| `coverage.architecture` ≥ 80 | 3 |
| `coverage.architecture` ≥ 50 | 1 |
| `coverage.implementation` ≥ 70 | 3 |
| `coverage.implementation` ≥ 40 | 1 |
| `coverage.testing` ≥ 60 | 4 |
| `coverage.testing` ≥ 30 | 2 |

**Applicable to:** architecture.md, implementation.md, testing.md only.
**Not applicable to:** overview, api, events, security, desktop, roadmap, history, references, adr.

### 3.6 Dimension 6: Consistency (10 points)

Measures internal consistency and reference integrity.

| Condition | Points Deducted |
|-----------|----------------|
| Broken internal reference (link target does not exist) | -2 per broken link |
| `canonical: true` conflict (two docs claim canonical for same concept) | -5 |
| `superseded_by` points to non-existent ID | -3 |
| `depends_on` references non-existent ID | -2 per broken dep |
| Document is over the 500-line limit | -5 |
| Document has headings below H3 | -2 |

Consistency starts at 10 and deducts from there (floor: 0).

---

## 4. Capability Health Score

The capability health score aggregates document scores with structural bonuses.

```
Capability Score = (Average of all document scores)
                 × (1 - penalty_factor)
                 + structural_bonus
```

### 4.1 Structural Bonus (up to +10 points)

| Condition | Bonus |
|-----------|-------|
| All required files present | +5 |
| All required files are approved (not draft) | +3 |
| index.yaml is up-to-date | +2 |

### 4.2 Penalty Factor

| Condition | Penalty |
|-----------|---------|
| Missing required file (per file) | 5% per missing file |
| File over 500 lines | 3% per oversized file |
| More than 3 stale files | 5% |
| Duplicate canonical declared | 15% |

---

## 5. Metric Definitions

### 5.1 Coverage

**Architecture Coverage (0–100)**
Percentage of the capability's design that is documented. Declared by owner in frontmatter.

**Implementation Coverage (0–100)**
Percentage of the architecture that is implemented. Derived from `implemented_by` evidence entries with `verified: true`.

**Test Coverage (0–100)**
Percentage of implementation covered by tests. Derived from `tested_by` entries with `coverage_percent`.

**Desktop Coverage (0–100)**
Percentage of the UI that is documented. 0 if capability has no desktop component.

### 5.2 Freshness

**Freshness Score (0–100)**
How recently a document was reviewed vs. its scheduled review frequency.

**Stale Document Count**
Number of documents past their `review_date`.

**Overdue Days (average and maximum)**
Average and maximum days past `review_date` across all stale documents.

### 5.3 Completeness

**Metadata Completeness (0–100)**
Percentage of required metadata fields that are present across all documents.

**Relationship Completeness (0–100)**
Percentage of expected relationships that are declared.

**File Completeness (0–100)**
Percentage of required capability files that exist.

### 5.4 Consistency

**Evidence Score (0–100)**
Percentage of architecture documents that have verified implementation evidence.

**Broken Reference Count**
Number of internal links pointing to non-existent files or IDs.

**Duplicate Ratio (0–1.0)**
Ratio of non-canonical concept declarations to total concept declarations.

**Knowledge Confidence (high/medium/low)**
Distribution of `confidence` levels across all architecture documents.

### 5.5 Debt Metrics

**Documentation Debt**
Sum of effort required to bring all documents to a health score ≥ 80.

```
Documentation Debt = Σ(80 - document_health_score) for all docs where score < 80
```

**Knowledge Debt**
Missing documents: capabilities that exist in code but lack architecture documentation.

**Orphan Count**
Documents with no inbound or outbound references.

**Dead Document Count**
Documents unreachable through any index.

---

## 6. Health Thresholds

### 6.1 Document-Level Thresholds

| Score Range | Status | Action Required |
|-------------|--------|----------------|
| 90–100 | Excellent | None |
| 75–89 | Good | Monitor |
| 60–74 | Fair | Schedule improvement sprint |
| 40–59 | Poor | Escalate to owner |
| 20–39 | Critical | Immediate remediation required |
| 0–19 | Invalid | Document is effectively unusable |

### 6.2 Capability-Level Thresholds

| Score Range | Status | Action Required |
|-------------|--------|----------------|
| 85–100 | Certified | Eligible for Phase gate |
| 70–84 | Active | No gate risk |
| 50–69 | Degraded | Owner must present remediation plan |
| 30–49 | At Risk | Capability is not safe to extend |
| 0–29 | Blocked | No new features until remediated |

### 6.3 Repository-Level Thresholds

| Score Range | Status |
|-------------|--------|
| 80+ | Ready for Documentation Intelligence Platform |
| 60–79 | Documentation refactoring in progress |
| 40–59 | Foundation phase incomplete |
| < 40 | Critical knowledge debt — all other work blocked |

---

## 7. Health Report Structure

The health report is generated by the audit tool and published as:
- `architecture/HEALTH-REPORT.md` (human-readable)
- `architecture/health-report.yaml` (machine-readable)

### 7.1 Report Sections

```markdown
# Knowledge Health Report

Generated: 2026-06-29
Repository Score: 67/100 (Fair)

## Summary

| Metric | Value | Threshold | Status |
|--------|-------|-----------|--------|
| Total Documents | 786 | — | — |
| Documents Approved | 234 | — | — |
| Documents Stale | 47 | 0 | Warning |
| Broken References | 12 | 0 | Critical |
| Duplicate Canonicals | 3 | 0 | Critical |
| Documentation Debt | 4,120 pts | < 500 | Critical |

## Domain Scorecard

| Domain | Score | Trend | Issues |
|--------|-------|-------|--------|
| DOM-WORKFLOW | 82/100 | ↑ | 1 stale |
| DOM-AI | 71/100 | → | 3 stale, 1 broken ref |
| DOM-PROMPT | 68/100 | ↓ | 2 missing files |
...

## Critical Issues (0 days to resolve)

1. [CRIT-001] Duplicate canonical: KA-ARCH-001 and KA-ARCH-008 both claim canonical for workflow-execution
2. [CRIT-002] Broken reference in KA-IMPL-003: points to KA-API-015 which does not exist
3. [CRIT-003] Duplicate canonical: ...

## High Priority (30 days to resolve)

...

## Stale Documents (owner action required)

| Document | Owner | Days Overdue |
|----------|-------|-------------|
| KA-ARCH-007 | Capability Architect | 45 days |
...
```

---

## 8. Health Tracking

### 8.1 Trend Tracking

Health scores are recorded in `architecture/health-history.yaml` at each audit run.

```yaml
history:
  - date: 2026-06-29
    repository_score: 67
    domains:
      DOM-WORKFLOW: 82
      DOM-AI: 71
    critical_issues: 3
    stale_documents: 47

  - date: 2026-05-29
    repository_score: 59
    ...
```

### 8.2 Phase Gate Criteria

Before proceeding to Phase 2.0D.2 (Documentation Intelligence Platform):

| Gate Criterion | Required Value |
|---------------|---------------|
| Repository health score | ≥ 80 |
| Broken references | 0 |
| Duplicate canonicals | 0 |
| Documents missing knowledge_id | 0 |
| Capabilities missing required files | 0 |
| Documents over 500 lines | 0 |

---

## References

- [KNOWLEDGE-ARCHITECTURE-VISION.md](KNOWLEDGE-ARCHITECTURE-VISION.md) — Health as a success criterion
- [METADATA-STANDARD.md](METADATA-STANDARD.md) — Metadata that drives health scoring
- [CANONICAL-SOURCE-STRATEGY.md](CANONICAL-SOURCE-STRATEGY.md) — Duplication metrics
- [REPOSITORY-AUDIT-RULES.md](REPOSITORY-AUDIT-RULES.md) — Automated health enforcement
- [KNOWLEDGE-EVOLUTION-STRATEGY.md](KNOWLEDGE-EVOLUTION-STRATEGY.md) — Health over time
