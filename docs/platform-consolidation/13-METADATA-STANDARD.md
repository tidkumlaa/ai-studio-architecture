---
knowledge_id: KNW-PLAT-ARCH-013
title: "Platform Metadata Standard"
status: AUTHORITATIVE
phase: "2.1D.0"
version: "1.0.0"
created: "2026-06-29"
purpose: "Define YAML metadata schemas for module, runtime, service, capability, and manifest files"
canonical_source: "architecture/docs/platform-consolidation/13-METADATA-STANDARD.md"
dependencies:
  - "02-OBJECT-MODEL.md"
related_documents:
  - "14-PLATFORM-MANIFEST.md"
  - "15-CAPABILITY-REGISTRY.md"
  - "17-MODULE-REGISTRY.md"
acceptance_criteria:
  - "Every schema is valid JSON Schema"
  - "Every required field is documented"
  - "Example files are provided for all 6 schema types"
verification_checklist:
  - "[ ] All schemas are parseable with PyYAML"
  - "[ ] Required fields validated on CI"
  - "[ ] Examples match schemas"
future_extensions:
  - "JSON Schema registry for IDE autocomplete"
  - "Schema versioning with migration rules"
---

# Platform Metadata Standard

## Schema Files Overview

| File Name | Location | Purpose |
|-----------|----------|---------|
| `module.yaml` | Every module directory | Module identity and capabilities |
| `runtime.yaml` | Every runtime directory | Runtime configuration |
| `service.yaml` | Every service directory | Service interface and dependencies |
| `capability.yaml` | Capability definitions | What the platform can do |
| `manifest.yaml` | `platform/` root | Complete platform manifest |
| `ownership.yaml` | Every directory | Ownership and governance |

---

## module.yaml Schema

```yaml
# Required in every platform module directory
# JSON Schema: schemas/module.schema.json

id: string                    # required: "module:ai:resource_intelligence:v2"
name: string                  # required: "resource_intelligence"
version: string               # required: semver "2.1.0"
runtime_id: string            # required: "runtime:ai:v2"
python_module: string         # required: importable dotted path
status: enum                  # required: EXISTING | PLANNED | MOVED | DEPRECATED
public: boolean               # required: true = in SDK
capabilities:                 # required: list of capability IDs
  - "capability:ai:routing:v1"
dependencies:                 # optional: module IDs within same runtime
  - "module:ai:quota_manager:v1"
description: string           # required: one-line description
owner: string                 # required: team name
created: date                 # required: ISO 8601
```

**Example:**
```yaml
id: "module:ai:resource_intelligence:v2"
name: "resource_intelligence"
version: "2.1.0"
runtime_id: "runtime:ai:v2"
python_module: "platform.runtimes.ai_runtime.resource_intelligence"
status: EXISTING
public: false
capabilities:
  - "capability:ai:routing:v1"
  - "capability:ai:quota:v1"
  - "capability:ai:forecast:v1"
dependencies: []
description: "AI Resource Operating Center — quota, routing, intelligence"
owner: "AI Runtime Team"
created: "2026-06-01"
```

---

## runtime.yaml Schema

```yaml
# Required in every runtime directory

id: string                    # required: "runtime:ai:v2"
name: string                  # required: "ai_runtime"
version: string               # required: semver
boot_order: integer           # required: 1-7
status: enum                  # required: ACTIVE | PLANNED | DEPRECATED
capabilities:                 # required: capability IDs this runtime provides
  - "capability:ai:routing:v1"
depends_on:                   # required: kernel capability IDs consumed
  - "capability:kernel:events:v1"
  - "capability:kernel:di:v1"
modules:                      # required: module IDs owned
  - "module:ai:resource_intelligence:v2"
owner: string                 # required: team name
description: string           # required
```

**Example:**
```yaml
id: "runtime:ai:v2"
name: "ai_runtime"
version: "2.1.0"
boot_order: 5
status: ACTIVE
capabilities:
  - "capability:ai:routing:v1"
  - "capability:ai:quota:v1"
depends_on:
  - "capability:kernel:events:v1"
  - "capability:resource:quota:v1"
  - "capability:provider:registry:v1"
modules:
  - "module:ai:resource_intelligence:v2"
owner: "AI Runtime Team"
description: "AI execution, routing, and resource intelligence"
```

---

## service.yaml Schema

```yaml
# Required in every service directory

id: string                    # required: "service.auth"
name: string                  # required
version: string               # required: semver
interface: string             # required: Python ABC dotted path
implementations:              # required: list of implementation class paths
  - "platform.services.auth.JWTAuthService"
default_implementation: string # required
depends_on:                   # optional: other service IDs
  - "service.cache"
stability: enum               # required: STABLE | BETA | EXPERIMENTAL
owner: string                 # required
```

---

## capability.yaml Schema

```yaml
# Capability definition

id: string                    # required: "capability:ai:routing:v1"
domain: string                # required: "ai"
name: string                  # required: "routing"
version: string               # required: semver
interface: string             # required: Python Protocol dotted path
provided_by: string           # required: runtime or service ID
consumers: list[string]       # optional: known consumer IDs
description: string           # required
stability: enum               # required: STABLE | BETA | EXPERIMENTAL
```

---

## manifest.yaml Schema

```yaml
# platform/manifest.yaml — machine-generated, do not edit manually
# Generated by: platform-verify manifest --generate

platform_id: string           # required
platform_version: string      # required: semver
generated_at: datetime        # required
runtimes: list[string]        # runtime IDs
services: list[string]        # service IDs
capabilities: list[string]    # all capability IDs
packages: list[string]        # distribution package IDs
module_count: integer
capability_count: integer
checksum: string              # SHA-256 of sorted capability+module IDs
```

---

## ownership.yaml Schema

```yaml
# Required in key directories (optional in leaf directories)

directory: string             # required: relative path
owner: string                 # required: team name
visibility: enum              # required: PUBLIC | INTERNAL | PRIVATE | EXPERIMENTAL
write_access:                 # required: list of teams
  - "AI Runtime Team"
review_required: boolean      # required: true = PRs need team review
escalation_contact: string    # required: Slack channel or email
```

---

## Metadata Validation

All metadata files are validated by `platform-verify metadata`:
```bash
platform-verify metadata --strict
# Validates all *.yaml against their schemas
# Exits non-zero if any required field is missing or invalid
```

Validation rules:
- All required fields present
- ID uniqueness across registry
- version is valid semver
- python_module is importable (or PLANNED status)
- dependencies exist in registry
- capabilities declared in capability.yaml match module.yaml claims
