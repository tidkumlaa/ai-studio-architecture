# KNW-KE-ARCH-024 — Knowledge Import / Export

**Phase:** 3.0C.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Defines the supported formats for importing knowledge from external sources and exporting knowledge for consumption by runtimes, products, and external tools.

---

## Import Formats

### 1. YAML (canonical)

The repository's native format. Every object stored as YAML.

```bash
kos import --format yaml --file objects.yaml
kos import --format yaml --dir knowledge/packages/platform/objects/
```

Rules:
- Must validate against `base-schema.json` and type schema
- Must pass `kos lint`
- Duplicate IDs are rejected unless `--allow-update` is set

### 2. JSON

JSON equivalent of the YAML format. Used for API imports.

```bash
kos import --format json --file objects.json
```

Conversion: `json → yaml (canonical form) → lint → register`

### 3. CSV (bulk metadata import)

For importing object metadata without full schemas. CSV maps to base fields only.

```
knowledge_id,object_type,name,version,status,owner,description
KNW-PLT-MOD-001,module,Quota Manager,1.0.0,DRAFT,team:platform,"Manages quotas"
```

```bash
kos import --format csv --file objects.csv --type module
```

Limitations: CSV import creates DRAFT objects only. Type-specific fields must be added manually.

### 4. Markdown (documentation import)

Import from existing documentation. Extracts metadata from front matter:

```markdown
---
knowledge_id: KNW-PLT-MOD-001
name: Quota Manager
object_type: module
---

## Description
Manages per-session and per-day resource consumption quotas.
```

```bash
kos import --format markdown --dir platform/docs/
```

### 5. External Source (migration)

Import from legacy systems. Requires a migration adapter:

```bash
kos import --format external \
           --adapter confluence \
           --source "https://wiki.example.com/pages" \
           --mapping migration/confluence-mapping.yaml
```

Migration adapters are defined in `knowledge/migration-platform/` (not in this phase).

---

## Export Formats

### 1. YAML (canonical export)

```bash
kos export --format yaml --package plt --output dist/
kos export --format yaml --namespace plt --status CANONICAL
```

Output: One file per object, same format as repository.

### 2. JSON

```bash
kos export --format json --package plt --output dist/platform.json
```

Output: Single JSON file with array of objects.

### 3. PromptPack (AI consumption)

Structured format for LLM prompts. Each object becomes a prompt-ready text block.

```bash
kos export --format promptpack \
           --package plt \
           --output dist/platform-promptpack.yaml
```

```yaml
# dist/platform-promptpack.yaml
format: promptpack
version: "1.0.0"
package: kos.platform.package
generated_at: "2026-06-30T00:00:00Z"
objects:
  - knowledge_id: KNW-PLT-MOD-001
    prompt_text: |
      ## Quota Manager (Module)
      Manages per-session and per-day resource consumption quotas across all AI providers.
      Status: CANONICAL | Owner: team:platform | Confidence: 0.90
      Implements: [KNW-RT-SVC-003] | Tests: [KNW-TEST-TST-001]
```

### 4. LLMMemory (embedding format)

For vector store ingestion:

```bash
kos export --format llmmemory \
           --package plt \
           --output dist/platform-embeddings.jsonl
```

```jsonl
{"id": "KNW-PLT-MOD-001", "text": "Quota Manager ...", "metadata": {...}}
{"id": "KNW-ALG-ALG-007", "text": "BM25 Text Ranker ...", "metadata": {...}}
```

### 5. Dataset (ML training format)

```bash
kos export --format dataset --namespace all --output dist/training-data/
```

Outputs Parquet files suitable for ML training pipelines.

---

## Export Rules

| Rule | Description |
|------|-------------|
| EX-001 | Default export includes only CANONICAL and VERIFIED objects |
| EX-002 | Export never includes ARCHIVED objects |
| EX-003 | `--include-draft` flag required to export DRAFT objects |
| EX-004 | Exported files are signed with checksum in manifest |
| EX-005 | Export manifest is always generated alongside the export |

---

## Export Manifest

```yaml
# dist/export-manifest.yaml
generated_at: "2026-06-30T00:00:00Z"
format: yaml
package: kos.platform.package
objects_exported: 0
min_status: VERIFIED
checksum: sha256:abc123...
files:
  - file: objects/quota-manager.yaml
    knowledge_id: KNW-PLT-MOD-001
    checksum: sha256:def456...
```

---

## Round-Trip Invariant

```
object → export(YAML) → import(YAML) → object' 
object.to_dict() == object'.to_dict()
```

This round-trip property is tested in the verification suite (KV-001).

---

## Cross-References

- Package structure → `02-KNOWLEDGE-PACKAGES`
- Schemas for validation → `10-KNOWLEDGE-SCHEMAS`
- PromptPack used by agents → Phase 3.0C `31-EXECUTION-CONTEXT`
- LLMMemory used by semantic layer → Phase 3.0C `29-SEMANTIC-LAYER`
