# KNW-KC-ARCH-032 — Metadata Standard

**Phase:** 3.0C  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

The Metadata Standard defines the mandatory and optional metadata fields every Knowledge Object carries. Metadata is separate from identity and content — it describes how the object is managed, classified, and consumed.

---

## Metadata Categories

### Category M1 — Governance Metadata
```yaml
governance:
  knowledge_id: string           # from Identity Engine
  owner: OwnerRef
  reviewers: list[OwnerRef]
  steward: string                # long-term responsible party
  sla_tier: CRITICAL|HIGH|NORMAL|LOW
  change_authority: string       # who can approve MAJOR changes
  review_cadence_days: int       # how often this object should be reviewed
  last_reviewed_at: datetime | null
  next_review_due: datetime | null
```

### Category M2 — Classification Metadata
```yaml
classification:
  object_type: KnowledgeObjectType
  sub_type: string
  category: string
  domain: string
  tier: 0..7
  maturity: EXPERIMENTAL|BETA|STABLE|CANONICAL
  sensitivity: PUBLIC|INTERNAL|CONFIDENTIAL|SECRET
  external_facing: bool
```

### Category M3 — Discovery Metadata
```yaml
discovery:
  tags: list[string]
  keywords: list[string]
  aliases: list[string]
  search_boost: float            # 0.5..2.0 — weights this object in search results
  related_external_ids: dict     # {"jira": "PLAT-123", "notion": "abc123"}
  doc_url: string | null
  source_repo: string | null
  source_file: string | null
  source_line: int | null
```

### Category M4 — Operational Metadata
```yaml
operational:
  created_at: datetime
  updated_at: datetime
  created_by: string
  updated_by: string
  read_count: int
  write_count: int
  last_read_at: datetime | null
  clone_count: int               # how many forks exist
  reference_count: int           # how many other objects link to this
  import_count: int              # how many compilers used this
```

### Category M5 — Extension Metadata
```yaml
extensions:
  domain_fields: dict            # typed by domain owner
  plugin_data: dict              # populated by registered plugins
  experimental: dict             # pre-promotion fields
  custom_validators: list[str]   # knowledge_ids of CustomValidatorObjects
```

---

## Metadata Validation Rules

| Rule | Description |
|------|-------------|
| MV-001 | `owner` must resolve to a registered team or user |
| MV-002 | `sla_tier: CRITICAL` requires `change_authority` to be non-empty |
| MV-003 | `sensitivity: CONFIDENTIAL` requires `reviewers` to be non-empty |
| MV-004 | `external_facing: true` requires status ≥ APPROVED |
| MV-005 | `review_cadence_days` must be > 0; CRITICAL objects: ≤ 30 days |
| MV-006 | `search_boost` outside 0.5..2.0 is rejected |
| MV-007 | `tags` are normalised to lowercase; max 20 tags per object |
| MV-008 | `domain_fields` must conform to the domain's registered extension schema |

---

## SLA Tier Definitions

| Tier | Description | Review Cadence | Change Authority |
|------|-------------|----------------|-----------------|
| CRITICAL | Platform kernel, security, identity | ≤ 30 days | Architecture Board |
| HIGH | Runtimes, APIs, core services | ≤ 90 days | Domain Lead |
| NORMAL | Modules, patterns, algorithms | ≤ 180 days | Team Lead |
| LOW | Drafts, experiments, ephemeral | ≤ 365 days | Owner |

---

## Metadata in File Storage

Every object file includes a `_metadata` section:
```yaml
_metadata:
  governance:
    owner: "team:architecture-board"
    steward: "team:architecture-board"
    sla_tier: CRITICAL
    review_cadence_days: 30
    last_reviewed_at: "2026-06-01T00:00:00Z"
    next_review_due: "2026-07-01T00:00:00Z"
  classification:
    tier: 0
    maturity: CANONICAL
    sensitivity: INTERNAL
    external_facing: false
  discovery:
    tags: ["kos", "architecture", "core"]
    keywords: ["knowledge", "operating", "system"]
    search_boost: 1.5
  operational:
    created_by: "arch-board"
    updated_by: "arch-board"
    reference_count: 0
```

---

## Metadata Propagation Rules

| Scenario | Propagation |
|----------|-------------|
| Fork | `owner` and `created_by` updated to fork creator; `sensitivity` inherited |
| Merge | Strictest `sensitivity` wins; `sla_tier` inherits highest tier |
| Deprecation | `sensitivity` preserved; `sla_tier` reduced to LOW after 90 days |

---

## Metadata Review Protocol

```
METADATA_REVIEW_DUE(object):
  if now() > next_review_due:
    emit: governance.review.due event
    notify: owner, steward
    flag: review_overdue = true

  REVIEW_ACTION:
    CONFIRM: update last_reviewed_at, extend next_review_due
    UPDATE: modify metadata fields
    ESCALATE: involve change_authority
```

---

## Metadata Protocol

```python
class MetadataStandard(Protocol):
    def validate(self, obj: KnowledgeObject) -> MetadataValidationResult: ...
    def compute_next_review_date(self, obj: KnowledgeObject) -> datetime: ...
    def get_overdue_reviews(self, namespace: str) -> list[str]: ...
    def propagate(
        self, source: KnowledgeObject, target: KnowledgeObject, event: str
    ) -> dict: ...
```

---

## Cross-References

- Governance metadata drives review cadence → `33-LIFECYCLE-EXTENSIONS`
- SLA tier affects alerts → `11-QUALITY-ENGINE`
- Sensitivity in ExecutionContext → `31-EXECUTION-CONTEXT`
- Tags used in search → `28-SEARCH-ENGINE`
