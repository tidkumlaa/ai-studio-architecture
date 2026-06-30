# KNW-KE-ARCH-004 — Knowledge Style Guide

**Phase:** 3.0C.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

The Style Guide defines how to write Knowledge Objects. Consistent style makes objects readable, comparable, and searchable. Violations are linter errors.

---

## Naming

### Object Names (`name` field)

```
Rule SG-N-01: Title Case. Every word capitalised.
  GOOD: "Quota Manager"
  BAD:  "quota manager", "QUOTA MANAGER", "quota_manager"

Rule SG-N-02: No articles (a, an, the) unless semantically required.
  GOOD: "Registry Architecture"
  BAD:  "The Registry Architecture"

Rule SG-N-03: Max 80 characters.

Rule SG-N-04: No special characters except hyphen and parentheses.
  GOOD: "BM25 Scorer (Text)"
  BAD:  "BM25/Scorer", "BM25:Scorer"

Rule SG-N-05: Acronyms fully uppercase: API, SDK, KQL, BFS.

Rule SG-N-06: Versions in names only when object type mandates it.
  GOOD (for MODULE): "Quota Manager v2"    -- version in module_id
  BAD:               "Quota Manager 2.0.0" -- version belongs in version field
```

---

## Descriptions (`description` field)

```
Rule SG-D-01: Imperative mood for MODULES, SERVICES, APIs.
  GOOD: "Manages per-session and per-day resource quotas."
  BAD:  "A module that manages quotas."

Rule SG-D-02: Declarative for REQUIREMENTS, DECISIONS, ARCHITECTURE.
  GOOD: "The platform must limit each session to 100,000 input tokens."
  BAD:  "Limit sessions."

Rule SG-D-03: Minimum 20 characters. Maximum 500 characters.

Rule SG-D-04: Single sentence preferred. Two sentences maximum.

Rule SG-D-05: No "TODO", "TBD", "placeholder", "see above", "as above".

Rule SG-D-06: No referring to the object by name in its own description.
  BAD: "The Quota Manager manages quotas."
  GOOD: "Manages per-session and per-day resource quotas for the AI runtime."
```

---

## Tags (`tags` field)

```
Rule SG-T-01: Minimum 1 tag. Maximum 10 tags.

Rule SG-T-02: All tags lowercase, hyphen-separated.
  GOOD: ["platform", "quota", "resource-management"]
  BAD:  ["Platform", "Quota Management", "resource_management"]

Rule SG-T-03: First tag must be the package namespace.
  GOOD (platform module): tags: [platform, ...]
  BAD:                     tags: [quota, platform, ...]

Rule SG-T-04: No duplicate tags.

Rule SG-T-05: No generic filler tags: "object", "knowledge", "kos", "item".

Rule SG-T-06: Use the canonical tag list from 09-KNOWLEDGE-CATALOG.
```

---

## Evidence (`evidence.items`)

```
Rule SG-E-01: Every VERIFIED+ object must have at least one evidence record.

Rule SG-E-02: Every CANONICAL object must have EV-CODE or EV-TEST evidence.

Rule SG-E-03: Evidence source_url must be a valid URI or file path.

Rule SG-E-04: Evidence description must be ≥ 10 characters.

Rule SG-E-05: Evidence weight must match the type's canonical weight
              from 10-EVIDENCE-ENGINE.
```

---

## Relationships

```
Rule SG-R-01: Relationships are defined in the package's relationships/ directory,
              not inline in the object file.

Rule SG-R-02: A relationship file must reference valid knowledge_ids for both
              source and target — linter checks at CI time.

Rule SG-R-03: Relationship type must be one of the 24 canonical types
              from 07-RELATIONSHIP-TYPES.

Rule SG-R-04: No bidirectional duplicates. If A DEPENDS_ON B exists,
              B DEPENDS_ON A is redundant — use the traversal engine instead.
```

---

## Metadata (`metadata` section)

```
Rule SG-M-01: created_by must be "team:name" or "user:email".

Rule SG-M-02: source_doc is the file path or URI where the knowledge was
              originally defined.

Rule SG-M-03: metadata may not contain PII (email addresses, passwords,
              personal identifiers).
```

---

## Formatting

### YAML files

```
Rule SG-F-01: 2-space indentation. No tabs.

Rule SG-F-02: Strings with special characters use double-quotes.

Rule SG-F-03: Multiline strings use block scalar (|).

Rule SG-F-04: Lists with 0 items: [] (inline)
              Lists with 1+ items: block style (one item per line).

Rule SG-F-05: Keys in snake_case. No camelCase.

Rule SG-F-06: Top-level section order follows Universal Schema:
              knowledge_id, object_type, name, version, status, owner,
              description, classification, identity, tags, evidence,
              quality, confidence, traceability, lifecycle, relationships,
              metadata, [type-specific fields].
```

### Markdown files (in docs/ directories)

```
Rule SG-F-07: Headings use ATX style (#, ##, ###). No underline style.

Rule SG-F-08: One blank line before and after each heading.

Rule SG-F-09: Code blocks use triple backticks with language tag.

Rule SG-F-10: Line length ≤ 120 characters (prose), no limit (code blocks).
```

---

## Writing Rules

```
Rule SG-W-01: Write for your successor, not for yourself.
              Assume the reader has never seen this object before.

Rule SG-W-02: Avoid jargon unless defined in the Glossary (22-KNOWLEDGE-REFERENCE-LIBRARY).

Rule SG-W-03: No marketing language: "best", "fastest", "unique", "revolutionary".

Rule SG-W-04: Quantify performance claims. Not "fast" but "< 5ms P99".

Rule SG-W-05: If you cite a requirement, use its knowledge_id.
              Not "the platform needs this" but "see KNW-ARCH-REQ-001".
```

---

## Review Rules

```
Rule SG-RV-01: Every PR touching CANONICAL objects requires two reviewers.

Rule SG-RV-02: Reviewer must check: naming, description, tags, evidence,
               relationships, and that no CANONICAL rules are violated.

Rule SG-RV-03: Author cannot be their own reviewer.

Rule SG-RV-04: Review comments must reference the rule number they cite.
               Not "bad name" but "SG-N-02: remove 'the'".
```

---

## Cross-References

- Naming rules for IDs/URIs → `05-KNOWLEDGE-NAMING-STANDARD`
- Linter enforces all SG-* rules → `06-KNOWLEDGE-LINTER`
- Formatter auto-fixes SG-F-* rules → `07-KNOWLEDGE-FORMATTER`
- Tag registry (canonical tag list) → `09-KNOWLEDGE-CATALOG`
