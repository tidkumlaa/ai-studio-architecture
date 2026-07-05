---
knowledge_id: KA-PLAT-004
version: "1.0.0"
status: approved
owner: Chief Knowledge Architect
phase: 2.0D.1
created: 2026-06-29
review_date: 2026-12-29
canonical: true
domain: DOM-KNOWLEDGE
capability: migration-platform
type: specification
depends_on:
  - id: KA-PLAT-003
    reason: "Classifier consumes RepositoryInventory from Scanner"
  - id: KA-SPEC-002
    reason: "Knowledge Type System defines all valid document types"
---

# Document Classifier

## Automatic Document Type Classification with Confidence Scoring

---

## 1. Purpose

The KnowledgeClassifier assigns every document in a RepositoryInventory to a
document type, domain, and capability. Classification uses multiple signal sources
and produces a confidence score. It never modifies files.

---

## 2. Document Types

| Type | Description |
|------|-------------|
| `capability` | Capability overview or README |
| `architecture` | Architectural design document |
| `specification` | Formal specification |
| `api` | API contract, endpoint catalog |
| `database` | Database schema, data model |
| `workflow` | Workflow or process definition |
| `prompt` | Prompt template or prompt engineering |
| `security` | Security model, auth, permissions |
| `desktop` | Desktop/UI component or panel |
| `provider` | AI/LLM provider integration |
| `runtime` | Runtime component specification |
| `decision` | Architecture Decision Record (ADR) |
| `reference` | Reference document, catalog |
| `guide` | How-to guide, tutorial |
| `index` | Index, registry, navigation file |
| `readme` | Repository or folder README |
| `history` | History, changelog, migration report |
| `other` | Unclassifiable |

---

## 3. Signal Sources

Classification uses four weighted signal sources:

| Source | Weight | Description |
|--------|--------|-------------|
| Filename pattern | 35% | Filename matches known patterns |
| Path pattern | 30% | Directory path indicates type |
| H1 heading | 20% | First heading content |
| Keyword density | 15% | Frequency of type-specific keywords |

---

## 4. Filename Signal Rules

| Pattern | Type | Confidence |
|---------|------|-----------|
| `README.md` | `readme` | 0.95 |
| `index.yaml`, `KNOWLEDGE-INDEX.md` | `index` | 0.95 |
| `*-ADR-*.md`, `adr/*.md` | `decision` | 0.95 |
| `architecture.md`, `*ARCHITECTURE*.md` | `architecture` | 0.85 |
| `api.md`, `*-API-*.md`, `API-*.md` | `api` | 0.85 |
| `database.md`, `*DATABASE*.md` | `database` | 0.85 |
| `specification.md`, `*-SPEC-*.md` | `specification` | 0.85 |
| `*SECURITY*.md` | `security` | 0.80 |
| `*WORKFLOW*.md` | `workflow` | 0.80 |
| `*PROMPT*.md` | `prompt` | 0.80 |
| `*PROVIDER*.md` | `provider` | 0.80 |
| `*DESKTOP*.md`, `*PANEL*.md` | `desktop` | 0.80 |
| `HISTORY.md`, `CHANGELOG.md` | `history` | 0.90 |
| `MIGRATION-REPORT*.md` | `history` | 0.90 |

---

## 5. Path Signal Rules

| Path Pattern | Type | Confidence |
|-------------|------|-----------|
| `adr/` in path | `decision` | 0.90 |
| `decisions/` in path | `decision` | 0.90 |
| `api/` in path | `api` | 0.80 |
| `prompts/` in path | `prompt` | 0.85 |
| `providers/` in path | `provider` | 0.85 |
| `desktop/` in path | `desktop` | 0.85 |
| `security/` in path | `security` | 0.80 |
| `workflows/` in path | `workflow` | 0.80 |
| `docs/` in path | `architecture` | 0.40 |
| `archive/` in path | `history` | 0.70 |

---

## 6. Capability Inference

The classifier infers the capability from the directory structure:

```
INFER_CAPABILITY(path, root):
  parts = relative(path, root).parts
  FOR known_capability IN capabilities_config:
    IF any(part.lower() in known_capability.aliases for part in parts):
      RETURN known_capability.name, confidence=0.80
  IF len(parts) >= 2:
    RETURN parts[-2].replace("-", "_"), confidence=0.50
  RETURN None, confidence=0.0
```

Capabilities are loaded from `domains.yaml` (injected via config), not hardcoded.

---

## 7. Classification Result

```yaml
ClassificationResult:
  document_type: DocumentType       # Best match type
  confidence: float                 # 0.0 - 1.0
  capability: str | None            # Inferred capability
  domain: str | None                # Inferred domain
  signals: list[str]                # Human-readable explanation
  alternative_types:                # Second and third best matches
    - type: str
      confidence: float
```

---

## 8. Low-Confidence Handling

Documents with confidence < 0.40 are classified as `other` and flagged for
manual review in the migration report. They are not blocked from processing —
they receive generic metadata with `confidence: low` in the frontmatter.

---

## 9. Algorithm

```
CLASSIFY(record, content):
  scores = defaultdict(float)
  
  # Signal 1: Filename
  fn_scores = score_by_filename(record.path.name)
  FOR type, score IN fn_scores:
    scores[type] += score * 0.35
    
  # Signal 2: Path
  path_scores = score_by_path(record.relative_path)
  FOR type, score IN path_scores:
    scores[type] += score * 0.30
    
  # Signal 3: H1 heading
  heading = extract_h1(content)
  IF heading:
    heading_scores = score_by_heading(heading)
    FOR type, score IN heading_scores:
      scores[type] += score * 0.20
      
  # Signal 4: Keywords
  kw_scores = score_by_keywords(content)
  FOR type, score IN kw_scores:
    scores[type] += score * 0.15
    
  best_type = argmax(scores)
  confidence = scores[best_type]
  
  RETURN ClassificationResult(
    document_type = best_type IF confidence >= 0.40 ELSE OTHER,
    confidence    = confidence,
    capability    = infer_capability(record),
    domain        = infer_domain(record),
    signals       = explain_signals(scores)
  )
```

**Complexity:** O(K) where K = document size in words.

---

## References

- [KA-SPEC-002](../knowledge-core/02-KNOWLEDGE-TYPE-SYSTEM.md) — Type system being mapped to
- [02-REPOSITORY-SCANNER.md](02-REPOSITORY-SCANNER.md) — Source of RepositoryInventory
- [05-METADATA-GENERATOR.md](05-METADATA-GENERATOR.md) — Consumes ClassificationResult
