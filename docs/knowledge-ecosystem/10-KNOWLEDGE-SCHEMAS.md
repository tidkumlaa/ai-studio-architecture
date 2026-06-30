# KNW-KE-ARCH-010 — Knowledge Schemas

**Phase:** 3.0C.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

JSON Schemas validate every Knowledge Object file at authoring time and in CI. One schema per object type. All schemas share a `base-schema.json`.

---

## Schema File Layout

```
knowledge/schemas/
├── base-schema.json          # shared base (all 16 Universal Schema sections)
├── by-type/
│   ├── module-schema.json
│   ├── service-schema.json
│   ├── algorithm-schema.json
│   ├── configuration-schema.json
│   ├── api-schema.json
│   ├── capability-schema.json
│   ├── provider-schema.json
│   ├── test-schema.json
│   ├── benchmark-schema.json
│   ├── requirement-schema.json
│   ├── decision-schema.json
│   ├── architecture-schema.json
│   ├── pattern-schema.json
│   ├── specification-schema.json
│   ├── deployment-schema.json
│   ├── product-schema.json
│   ├── agent-schema.json
│   ├── prompt-schema.json
│   ├── conversation-schema.json
│   ├── task-schema.json
│   ├── strategy-schema.json
│   ├── execution-plan-schema.json
│   ├── market-schema.json
│   ├── forex-schema.json
│   ├── news-schema.json
│   ├── dataset-schema.json
│   ├── runtime-schema.json
│   ├── kernel-schema.json
│   ├── sdk-schema.json
│   ├── package-schema.json
│   ├── document-schema.json
│   ├── knowledge-base-schema.json
│   └── platform-schema.json
└── schema-registry.yaml      # maps object_type → schema file
```

---

## Base Schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "kos/schemas/base-schema.json",
  "title": "KnowledgeObject Base Schema",
  "type": "object",
  "required": [
    "knowledge_id", "object_type", "name", "version",
    "status", "owner", "description", "tags"
  ],
  "properties": {
    "knowledge_id": {
      "type": "string",
      "pattern": "^KNW-[A-Z][A-Z0-9]*-[A-Z][A-Z0-9]*-[A-Za-z0-9_-]+$"
    },
    "object_type": {
      "type": "string",
      "enum": [
        "platform", "runtime", "kernel", "sdk", "service", "module", "package",
        "architecture", "specification", "decision", "requirement", "pattern", "algorithm",
        "api", "capability", "provider",
        "test", "benchmark",
        "deployment", "configuration",
        "product", "agent", "prompt", "conversation", "task",
        "strategy", "execution_plan",
        "dataset", "market", "forex", "news",
        "document", "knowledge_base"
      ]
    },
    "name": {
      "type": "string",
      "minLength": 1,
      "maxLength": 80
    },
    "version": {
      "type": "string",
      "pattern": "^\\d+\\.\\d+\\.\\d+$"
    },
    "status": {
      "type": "string",
      "enum": ["DRAFT", "REVIEW", "VERIFIED", "APPROVED", "CANONICAL", "DEPRECATED", "ARCHIVED"]
    },
    "owner": {
      "type": "string",
      "pattern": "^(team:|user:|system:).+"
    },
    "description": {
      "type": "string",
      "minLength": 20,
      "maxLength": 500
    },
    "tags": {
      "type": "array",
      "minItems": 1,
      "maxItems": 10,
      "items": {
        "type": "string",
        "pattern": "^[a-z][a-z0-9-]*$"
      },
      "uniqueItems": true
    },
    "classification": {
      "$ref": "#/$defs/classification"
    },
    "evidence": {
      "$ref": "#/$defs/evidence_section"
    },
    "confidence": {
      "$ref": "#/$defs/confidence_section"
    },
    "traceability": {
      "$ref": "#/$defs/traceability_section"
    },
    "metadata": {
      "$ref": "#/$defs/metadata_section"
    }
  },
  "$defs": {
    "classification": {
      "type": "object",
      "properties": {
        "domain": {"type": "string"},
        "layer": {"type": "integer", "minimum": 0, "maximum": 6},
        "category": {"type": "string"}
      }
    },
    "evidence_section": {
      "type": "object",
      "properties": {
        "items": {"type": "array"},
        "evidence_score": {"type": "number", "minimum": 0.0, "maximum": 1.0}
      }
    },
    "confidence_section": {
      "type": "object",
      "properties": {
        "declared": {"type": "number", "minimum": 0.0, "maximum": 1.0},
        "composite": {"type": ["number", "null"]}
      }
    },
    "traceability_section": {
      "type": "object",
      "properties": {
        "satisfies": {"type": "array", "items": {"type": "string"}},
        "implements": {"type": "array", "items": {"type": "string"}},
        "tests": {"type": "array", "items": {"type": "string"}}
      }
    },
    "metadata_section": {
      "type": "object",
      "properties": {
        "created_by": {"type": "string"},
        "created_at": {"type": "string", "format": "date-time"},
        "source_doc": {"type": "string"}
      }
    }
  }
}
```

---

## Type-Specific Extension (example: module)

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "kos/schemas/module-schema.json",
  "allOf": [{"$ref": "base-schema.json"}],
  "required": ["module_id", "runtime_id", "python_module"],
  "properties": {
    "object_type": {"const": "module"},
    "module_id": {
      "type": "string",
      "pattern": "^module:.+:.+"
    },
    "runtime_id": {"type": "string", "minLength": 1},
    "python_module": {
      "type": "string",
      "pattern": "^[a-zA-Z_][a-zA-Z0-9_.]*$"
    },
    "is_public": {"type": "boolean"},
    "module_status": {
      "type": "string",
      "enum": ["EXISTING", "PLANNED", "MOVED", "DEPRECATED"]
    }
  }
}
```

---

## Schema Registry

```yaml
# knowledge/schemas/schema-registry.yaml
version: "1.0.0"
schemas:
  module:         schemas/by-type/module-schema.json
  service:        schemas/by-type/service-schema.json
  algorithm:      schemas/by-type/algorithm-schema.json
  # ... all 33 types
```

---

## Schema Versioning

| Schema version | Policy |
|---------------|--------|
| PATCH bump | Add optional fields |
| MINOR bump | Add required fields (migration required) |
| MAJOR bump | Remove or rename fields (breaking) |

Schema version is embedded in `$id`:
`kos/schemas/v1/module-schema.json`

---

## Cross-References

- Templates reference these schemas → `03-KNOWLEDGE-TEMPLATES`
- CI validates against schemas → `08-KNOWLEDGE-CI`
- Python models in Phase 3.0B enforce same constraints → `platform/knowledge_runtime/objects/`
