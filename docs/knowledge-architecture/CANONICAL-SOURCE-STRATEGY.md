---
knowledge_id: KA-STD-003
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

# Canonical Source Strategy

## One Concept. One Source. Everything Else References It.

---

## 1. The Duplication Problem

Every duplicated concept is a time bomb.

When the definition of "Workflow Executor" exists in five documents, five maintainers are responsible for keeping five copies synchronized. In practice, none of them are. Each copy drifts. In six months, no one can answer "what is the current, authoritative definition of Workflow Executor?" with confidence.

This is not a documentation hygiene problem. It is an architectural problem: the knowledge system has no concept of identity, and without identity, there can be no canonicality.

The canonical source strategy solves this at the structural level.

---

## 2. The Canonical Rule

> **One concept. One canonical document. Unlimited references. Zero copies.**

A "concept" is any named, distinct piece of knowledge:
- An architectural pattern
- A capability definition
- An API contract
- A data schema
- An event specification
- An architecture decision
- A standard or rule

The canonical document is the **single authoritative source** for that concept. All other documents that mention the concept must **reference** the canonical source — never copy its content.

---

## 3. How to Declare Canonicality

### 3.1 In the Canonical Document

```yaml
canonical: true
```

A document with `canonical: true` declares itself as the single authoritative source for its primary concept.

### 3.2 In a Referencing Document

```yaml
canonical: false
```

Or simply omit the `canonical` field (defaults to `false`).

### 3.3 Canonical Conflict Rule

If two documents both declare `canonical: true` for the same concept, the audit tool flags this as a **Critical** violation. Resolution requires:
1. Determining which document is actually authoritative
2. Setting `canonical: false` on the other
3. Adding `superseded_by` if one replaced the other
4. Adding a reference link in the non-canonical document

---

## 4. Duplicate Detection

### 4.1 What Constitutes a Duplicate

A duplicate exists when:

| Condition | Duplicate Type |
|-----------|---------------|
| Two documents have the same `capability` + `type` combination with `canonical: true` | Hard duplicate |
| Two documents describe the same named concept in prose without one referencing the other | Soft duplicate |
| One document contains a section that is substantively identical to content in another document | Content duplicate |
| One document has a title that is identical or near-identical to another | Title duplicate |

### 4.2 Duplicate Detection Algorithm

The audit tool runs the following checks:

**Rule D1 — Multiple Canonicals per Concept:**
```
SELECT * FROM knowledge_objects
WHERE canonical = true
GROUP BY capability, type
HAVING COUNT(*) > 1
```

**Rule D2 — Title Collision:**
```
SELECT * FROM knowledge_objects
WHERE LOWER(title) = LOWER(other.title)
AND id != other.id
```

**Rule D3 — Capability-Type Uniqueness:**
```
SELECT capability, type, COUNT(*)
FROM knowledge_objects
WHERE canonical = true
GROUP BY capability, type
HAVING COUNT(*) > 1
```

**Rule D4 — Orphan Detection (Concept Exists, No Canonical):**
```
SELECT * FROM knowledge_objects
WHERE capability IN (known_capabilities)
AND type IN (required_types)
AND canonical = false
AND NOT EXISTS (
  SELECT 1 FROM knowledge_objects k2
  WHERE k2.capability = knowledge_objects.capability
  AND k2.type = knowledge_objects.type
  AND k2.canonical = true
)
```

### 4.3 Duplicate Resolution Process

When a duplicate is detected:

1. **Identify the authoritative document** — which document is more complete, more recent, more cited?
2. **Merge unique content** into the authoritative document
3. **Set the loser to** `canonical: false` and `superseded_by: [winner ID]`
4. **Add a banner** to the non-canonical document
5. **Update all references** pointing to the non-canonical document
6. **Start the deprecation clock** on the non-canonical document

---

## 5. Obsolete Document Detection

### 5.1 Obsolete Conditions

A document is **obsolete** when:

| Condition | Detection Rule |
|-----------|---------------|
| Its `review_date` is more than 6 months in the past with no update | Rule OB1 |
| It has `canonical: true` but zero inbound references and zero outbound links | Rule OB2 |
| Its `status` is `approved` but it references only deprecated or archived documents | Rule OB3 |
| Its `version` has not changed in 18+ months despite active development in the domain | Rule OB4 |
| The capability it belongs to has been renamed or merged | Rule OB5 |

### 5.2 Obsolete vs. Deprecated

| State | Meaning |
|-------|---------|
| Obsolete | May still be correct, but hasn't been reviewed — **uncertain validity** |
| Deprecated | Actively known to be replaced — **known invalid, being retired** |

Obsolete documents are flagged for human review. Deprecated documents follow the lifecycle process.

### 5.3 Obsolete Document Handling

When a document is flagged as obsolete:
1. Owner is notified (via health report)
2. A 30-day review window opens
3. If owner reviews and confirms accuracy, `review_date` is updated
4. If owner confirms it's no longer valid, deprecation is initiated
5. If no action in 30 days, document health score drops to `0`

---

## 6. Superseded Document Detection

### 6.1 What Supersedes a Document

A document is superseded when:
- A newer version of the document explicitly replaces it
- An ADR decision invalidates its design
- A capability refactoring renders it structurally obsolete

### 6.2 Supersession Metadata

The superseding document declares:
```yaml
supersedes: KA-ARCH-000    # The old document being replaced
```

The superseded document declares:
```yaml
status: deprecated
deprecated_date: 2026-06-29
superseded_by: KA-ARCH-001  # The new document
```

### 6.3 Supersession Chain

Documents may form a supersession chain:
```
KA-ARCH-000 → KA-ARCH-001 → KA-ARCH-002  (current)
```

The chain is navigable through `history.md` for any capability. Only the terminal node (no `superseded_by`) is the current canonical source.

---

## 7. Dead Document Detection

### 7.1 What Is a Dead Document

A dead document is:
- **Unreachable** — no index entry, no inbound references, no README link
- **Disconnected** — `status: approved` but not in any domain or capability folder
- **Misplaced** — in the `docs/` root with no path to a capability

Dead documents are the most dangerous: they may be correct or incorrect, and no one is reading them to know.

### 7.2 Dead Document Detection Algorithm

```
Rule DEAD1: Document not in capability index AND not in archive index
Rule DEAD2: Document not referenced by any other document's frontmatter
Rule DEAD3: Document not listed in any README.md
Rule DEAD4: Document knowledge_id not in KNOWLEDGE-REGISTRY.yaml
```

A document failing any three of these four rules is classified as dead.

### 7.3 Dead Document Handling

1. **Triage** — Determine if the document has valid content
2. **Classify** — Assign to a capability and type
3. **Register** — Add to KNOWLEDGE-REGISTRY.yaml with a new ID if needed
4. **Integrate** — Move to the correct capability folder
5. **Archive** — If content is genuinely stale and valueless, archive directly

The `archive/orphaned/` folder receives dead documents pending triage.

---

## 8. Orphan Document Detection

### 8.1 What Is an Orphan

An orphan is a document that:
- Has valid content and a knowledge ID
- But belongs to no declared capability
- Or references capabilities or IDs that no longer exist

### 8.2 Orphan Detection

```
Rule ORPH1: knowledge_id declared but not in KNOWLEDGE-REGISTRY.yaml
Rule ORPH2: capability in frontmatter not in CAPABILITY-INDEX.yaml
Rule ORPH3: related_* references knowledge IDs that don't exist
Rule ORPH4: domain not in DOMAIN-INDEX.yaml
```

### 8.3 Orphan Resolution

| Rule Violated | Resolution |
|--------------|-----------|
| ORPH1 | Register in KNOWLEDGE-REGISTRY.yaml |
| ORPH2 | Create the capability folder, or reassign to correct capability |
| ORPH3 | Update references to current IDs, or remove broken references |
| ORPH4 | Add domain to domain registry, or reassign to correct domain |

---

## 9. The Canonical Source Map

The Canonical Source Map is a machine-generated file that lists every concept and its canonical document.

### 9.1 Location

`architecture/CANONICAL-SOURCE-MAP.yaml`

### 9.2 Structure

```yaml
generated: 2026-06-29
total_concepts: 247
total_canonical: 247
conflicts: 0

concepts:
  workflow-execution:
    canonical: KA-ARCH-001
    title: "Workflow Runtime Architecture"
    file: capabilities/workflow-runtime/architecture.md
    references:
      - id: KA-IMPL-001
        context: "References workflow execution model"
      - id: KA-API-001
        context: "Documents the execution API"
    duplicates: []

  provider-selection:
    canonical: KA-ARCH-005
    title: "Provider Framework Architecture"
    references:
      - id: KA-ARCH-001
        context: "Workflow delegates provider selection"
    duplicates:
      - id: KA-ARCH-008
        conflict: "partial — sections 3.2–3.4 overlap with canonical"
        resolution: pending
```

---

## 10. Reference Syntax

When a document references a canonical source, the reference syntax is:

### 10.1 In Prose

```markdown
The Workflow Executor (canonical definition: [KA-ARCH-001](../workflow-runtime/architecture.md)) 
receives a workflow definition and...
```

Or, shorter form:

```markdown
The [Workflow Executor](../workflow-runtime/architecture.md) receives a workflow definition and...
```

### 10.2 In Frontmatter

```yaml
depends_on:
  - id: KA-ARCH-001
    reason: "This document depends on the workflow execution model"
```

### 10.3 Cross-Repository References

When referencing a concept from a different repository:

```yaml
depends_on:
  - id: KA-ARCH-001
    repo: ai-studio-architecture
    file: capabilities/workflow-runtime/architecture.md
    reason: "External canonical source"
```

---

## References

- [KNOWLEDGE-ARCHITECTURE-VISION.md](KNOWLEDGE-ARCHITECTURE-VISION.md) — Single Source of Truth principle
- [METADATA-STANDARD.md](METADATA-STANDARD.md) — How canonicality is declared
- [RELATIONSHIP-MODEL.md](RELATIONSHIP-MODEL.md) — Reference relationship types
- [KNOWLEDGE-HEALTH-METRICS.md](KNOWLEDGE-HEALTH-METRICS.md) — How duplicates affect health
- [REPOSITORY-AUDIT-RULES.md](REPOSITORY-AUDIT-RULES.md) — Automated enforcement
