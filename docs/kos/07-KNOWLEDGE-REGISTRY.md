---
knowledge_id: KNW-KOS-ARCH-007
title: "KOS Knowledge Registry"
status: AUTHORITATIVE
phase: "3.0A"
version: "1.0.0"
created: "2026-06-30"
purpose: "Specify the Knowledge Registry: central index of all knowledge objects, storage, and access patterns"
canonical_source: "architecture/docs/kos/07-KNOWLEDGE-REGISTRY.md"
dependencies:
  - "04-OBJECT-MODEL.md"
  - "05-KNOWLEDGE-GRAPH.md"
related_documents:
  - "08-QUERY-ENGINE.md"
  - "11-GOVERNANCE.md"
  - "13-LIFECYCLE.md"
acceptance_criteria:
  - "Registry contains all object types"
  - "CRUD operations are specified"
  - "Storage format is portable"
verification_checklist:
  - "[ ] All 9 content domains listed"
  - "[ ] Registration process documented"
  - "[ ] Search interface specified"
future_extensions:
  - "Distributed registry for multi-team deployments"
  - "Registry federation across AI Studio instances"
---

# KOS Knowledge Registry

## Purpose

The Knowledge Registry is the **single authoritative index** of everything the
AI Studio ecosystem knows about itself. It is the physical home of all Knowledge Objects.

**Central registry contains:**
- All Objects
- All Services
- All APIs
- All Events
- All Tests
- All Modules
- All Algorithms
- All Documents
- All Ownership
- All Capabilities
- All Runtimes
- All Products

---

## Registry Storage Layout

```
knowledge/
├── registry/
│   ├── index.yaml              — master index of all knowledge_ids
│   ├── objects/
│   │   ├── platform/           — PlatformObject files
│   │   ├── runtime/            — RuntimeObject files
│   │   ├── module/             — ModuleObject files
│   │   ├── service/            — ServiceObject files
│   │   ├── api/                — APIObject files
│   │   ├── algorithm/          — AlgorithmObject files
│   │   ├── requirement/        — RequirementObject files
│   │   ├── architecture/       — ArchitectureObject files
│   │   ├── decision/           — DecisionObject files
│   │   ├── test/               — TestObject files
│   │   ├── benchmark/          — BenchmarkObject files
│   │   ├── product/            — ProductObject files
│   │   ├── agent/              — AgentObject files
│   │   ├── prompt/             — PromptObject files
│   │   ├── strategy/           — StrategyObject files
│   │   ├── deployment/         — DeploymentObject files
│   │   ├── market/             — MarketObject files
│   │   ├── forex/              — ForexObject files
│   │   └── news/               — NewsObject files
│   └── graph/
│       └── edges/              — Edge definition files per node
├── compiler/
│   └── templates/              — Jinja2 templates
└── quality/
    └── scores/                 — Quality score snapshots
```

---

## Storage Format — Knowledge Object File

Each object stored as YAML:

```yaml
# knowledge/registry/objects/module/KNW-AI-MOD-quota_manager.yaml

knowledge_id: KNW-AI-MOD-quota_manager
object_type: module
name: "quota_manager"
version: "1.0.0"
status: CANONICAL
owner: "AI Runtime Team"
created_at: "2026-06-01T00:00:00Z"
updated_at: "2026-06-30T00:00:00Z"
description: "Resource quota enforcement for AI model calls"
tags:
  - "quota"
  - "resource"
  - "ai-runtime"

# Governance
evidence: "Required by GR-003 (no runtime without knowledge). Quota control is a P1 requirement from REQ-AI-001."
confidence: 0.95
review_status: "APPROVED"
approved_by: "Platform Team"
approved_at: "2026-06-29T00:00:00Z"

# Relationships
dependencies:
  - KNW-AI-MOD-usage_collector
related:
  - KNW-AI-MOD-api_budget_manager

# Module-specific fields
module_id: "module:ai:quota_manager:v1"
runtime_id: "runtime:ai:v2"
python_module: "platform.runtimes.ai_runtime.resource_intelligence.quota_manager"
capability_ids:
  - "capability:ai:quota.check:v1"
  - "capability:ai:quota.consume:v1"
is_public: false

# Quality (populated by KnowledgeQualityEngine)
quality_score: 0.91
```

---

## Registry Master Index

```yaml
# knowledge/registry/index.yaml
schema_version: "1.0"
generated_at: "2026-06-30T00:00:00Z"
total_objects: 247
objects_by_type:
  module: 58
  capability: 66
  service: 6
  runtime: 7
  api: 24
  algorithm: 10
  requirement: 12
  architecture: 37
  decision: 10
  test: 160
  benchmark: 8
  product: 4
  agent: 5
  prompt: 8
  # ...

canonical_count: 130
approved_count: 80
draft_count: 37
```

---

## Registry Interface

```python
class KnowledgeRegistry(Protocol):
    # Registration
    def register(self, obj: KnowledgeObject) -> None: ...
    def update(self, knowledge_id: str, obj: KnowledgeObject) -> None: ...
    def deregister(self, knowledge_id: str, reason: str) -> None: ...

    # Lookup
    def get(self, knowledge_id: str) -> KnowledgeObject | None: ...
    def exists(self, knowledge_id: str) -> bool: ...
    def list_by_type(self, object_type: KnowledgeObjectType) -> list[KnowledgeObject]: ...
    def list_by_status(self, status: KnowledgeLifecycleStatus) -> list[KnowledgeObject]: ...
    def list_by_owner(self, owner: str) -> list[KnowledgeObject]: ...

    # Validation
    def validate_id_unique(self, knowledge_id: str) -> bool: ...
    def validate_version(self, knowledge_id: str, new_version: str) -> bool: ...

    # Index
    def count(self) -> dict[str, int]: ...  # counts by type
    def checksum(self) -> str: ...         # deterministic checksum of all IDs
    def export(self, path: str) -> None: ...
```

---

## Registration Process

```
1. Author creates KnowledgeObject YAML
2. Author runs: platform-kos register --file path/to/object.yaml
3. Registry validates:
   a. knowledge_id is globally unique
   b. version is valid semver
   c. required fields present
   d. object_type is valid
   e. dependencies exist in registry (or PLANNED status)
4. If DRAFT → object added with status DRAFT
5. If requesting CANONICAL → must pass governance review first
6. Registry assigns created_at timestamp
7. Registry updates index.yaml
8. Registry emits event: platform.kos.object.registered
```

---

## Registry CLI

```bash
# Register a new knowledge object
platform-kos register --file knowledge/registry/objects/module/KNW-AI-MOD-new.yaml

# Get an object
platform-kos get KNW-AI-MOD-quota_manager

# List all modules
platform-kos list --type module

# List all CANONICAL objects
platform-kos list --status CANONICAL

# Count by type
platform-kos count

# Export full registry
platform-kos export --output knowledge-export.tar.gz

# Validate all objects in registry
platform-kos validate --registry
```

---

## Registry Events

| Event | Topic |
|-------|-------|
| Object registered | `platform.kos.object.registered` |
| Object updated | `platform.kos.object.updated` |
| Object status changed | `platform.kos.object.status_changed` |
| Object deregistered | `platform.kos.object.deregistered` |
| Registry rebuilt | `platform.kos.registry.rebuilt` |

All events carry `knowledge_id`, `object_type`, `old_status`, `new_status`.
