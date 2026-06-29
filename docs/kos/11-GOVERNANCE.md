---
knowledge_id: KNW-KOS-ARCH-011
title: "KOS Knowledge Governance"
status: AUTHORITATIVE
phase: "3.0A"
version: "1.0.0"
created: "2026-06-30"
purpose: "Define governance rules for all Knowledge Objects: identity, versioning, ownership, evidence, and approval"
canonical_source: "architecture/docs/kos/11-GOVERNANCE.md"
dependencies:
  - "03-GOLDEN-RULES.md"
  - "04-OBJECT-MODEL.md"
  - "13-LIFECYCLE.md"
related_documents:
  - "12-QUALITY.md"
  - "07-KNOWLEDGE-REGISTRY.md"
acceptance_criteria:
  - "All 14 governance fields are specified"
  - "Approval process is defined"
  - "Governance rules are machine-verifiable"
verification_checklist:
  - "[ ] Identity requirements defined"
  - "[ ] Versioning rules specified"
  - "[ ] Ownership structure documented"
future_extensions:
  - "Cryptographic signing of approved objects"
  - "Automated governance scoring"
---

# KOS Knowledge Governance

## Governance Fields

Every Knowledge Object must contain the following governance fields (GR-001 through GR-010 basis):

| Field | Required | Rules |
|-------|----------|-------|
| `knowledge_id` | REQUIRED | Unique, format `KNW-{domain}-{type}-{NNN}` |
| `version` | REQUIRED | Valid semver; increment on any material change |
| `owner` | REQUIRED | Named team or individual |
| `dependencies` | REQUIRED | List of knowledge_ids this object depends on |
| `evidence` | REQUIRED for decisions | Non-empty string; context + rationale |
| `confidence` | REQUIRED | Float 0.0–1.0 |
| `review_status` | REQUIRED | `PENDING` \| `IN_REVIEW` \| `APPROVED` \| `REJECTED` |
| `approved_by` | REQUIRED for CANONICAL | Named approver |
| `coverage` | COMPUTED | % of acceptance_criteria with linked tests |
| `freshness` | COMPUTED | Age since last `updated_at` |
| `relationships` | REQUIRED | At least one edge in the Knowledge Graph |
| `lifecycle` | REQUIRED | Current lifecycle status |
| `tags` | REQUIRED | Minimum 1 tag |
| `metadata` | OPTIONAL | Arbitrary key-value pairs |

---

## Identity Rules

### Knowledge ID Format

```
KNW-{DOMAIN}-{TYPE_CODE}-{NNN}
```

| Component | Rules |
|-----------|-------|
| `KNW` | Literal prefix; always present |
| `DOMAIN` | Uppercase; domain short code (PLAT, KOS, AI, FIN, PROD, ALG, ADR, REQ) |
| `TYPE_CODE` | Uppercase abbreviation; ARCH, MOD, SVC, API, ALG, REQ, ADR, TEST, etc. |
| `NNN` | Three-digit zero-padded sequence; unique within domain+type |

**Examples:**
- `KNW-PLAT-ARCH-001` — platform architecture document #1
- `KNW-KOS-ARCH-011` — KOS governance document
- `KNW-AI-MOD-quota_manager` — AI runtime module
- `KNW-ADR-009` — Architecture Decision #9

**Rules:**
- IDs are assigned at creation time and never change
- If an object is superseded, the new object gets a new ID; the old ID is retired

---

## Versioning Rules

| Rule | Specification |
|------|--------------|
| V-001 | Version follows semver: `MAJOR.MINOR.PATCH` |
| V-002 | Increment MAJOR for breaking changes (interface change, field removal) |
| V-003 | Increment MINOR for additive changes (new field, new edge) |
| V-004 | Increment PATCH for corrections (typo fix, clarification) |
| V-005 | Version history is immutable; old versions are archived, not overwritten |
| V-006 | `updated_at` must change with every version increment |
| V-007 | Version must differ from previous on any content change |

---

## Ownership Structure

| Tier | Owner | Responsibility |
|------|-------|----------------|
| Kernel objects | Kernel Team | Kernel, DI, Events, Config |
| Platform objects | Platform Team | Runtime model, layer rules, SDK |
| AI Runtime objects | AI Runtime Team | All ai_runtime modules |
| Knowledge objects | Knowledge Team | KOS, Registry, Compiler |
| Product objects | Product Teams | Individual products |
| Financial objects | Financial Team | Market, Forex, Trading |

---

## Evidence Requirements

For **Decision objects** (ADRs): evidence must contain all four elements:

```yaml
evidence: |
  Context: [Why was this decision needed?]
  Options Considered:
    - Option A: [description, pros, cons]
    - Option B: [description, pros, cons]
  Chosen: Option B
  Rationale: [Why option B was chosen over A]
  Expected Consequences: [positive] / [negative]
```

For **Requirement objects**: evidence must name the source:

```yaml
evidence: |
  Source: [Who raised this requirement, when]
  Business Justification: [Why this is needed]
  Priority: MUST / SHOULD / MAY
```

For **Module objects**: evidence is a brief explanation:

```yaml
evidence: |
  Fulfills: [requirement knowledge_ids]
  Approach: [brief description of implementation strategy]
```

---

## Approval Process

### DRAFT → REVIEW
```
1. Author completes all required governance fields
2. Author runs: platform-kos governance --check KNW-{id}
   (validates all required fields are present and valid)
3. Author submits for review: platform-kos governance --submit-review KNW-{id}
4. Status changes: DRAFT → REVIEW
```

### REVIEW → APPROVED
```
1. Reviewer inspects object content, evidence, relationships
2. Reviewer checks: does this object have knowledge_score ≥ 0.70?
3. If approved: platform-kos governance --approve KNW-{id} --approver "Team Name"
4. Status changes: REVIEW → APPROVED
```

### APPROVED → CANONICAL
```
1. Platform Team verifies the object is referenced by at least one higher-level object
2. All dependencies are CANONICAL or APPROVED
3. At least one Test object exists with COVERS edge to this object
4. platform-kos governance --canonicalize KNW-{id}
5. Status changes: APPROVED → CANONICAL
6. Event emitted: platform.kos.object.status_changed
```

---

## Governance Validation

```bash
# Check a single object's governance fields
platform-kos governance --check KNW-AI-MOD-quota_manager

# Check all objects in a domain
platform-kos governance --check --domain ai

# List objects missing evidence
platform-kos governance --missing evidence

# List unapproved objects in CANONICAL state (invalid)
platform-kos governance --violations
```

---

## Governance Events

| Event | Trigger |
|-------|---------|
| `platform.kos.governance.review_requested` | Object submitted for review |
| `platform.kos.governance.approved` | Object approved |
| `platform.kos.governance.rejected` | Object review rejected |
| `platform.kos.governance.canonicalized` | Object reached CANONICAL |
| `platform.kos.governance.violation` | Governance rule violated |
