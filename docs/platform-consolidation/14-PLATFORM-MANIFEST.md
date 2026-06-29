---
knowledge_id: KNW-PLAT-ARCH-014
title: "Platform Manifest"
status: AUTHORITATIVE
phase: "2.1D.0"
version: "1.0.0"
created: "2026-06-29"
purpose: "Define the top-level platform manifest structure, generation rules, and usage"
canonical_source: "architecture/docs/platform-consolidation/14-PLATFORM-MANIFEST.md"
dependencies:
  - "13-METADATA-STANDARD.md"
  - "02-OBJECT-MODEL.md"
related_documents:
  - "15-CAPABILITY-REGISTRY.md"
  - "16-SERVICE-REGISTRY.md"
  - "17-MODULE-REGISTRY.md"
  - "22-MIGRATION-ENGINE.md"
acceptance_criteria:
  - "Manifest is machine-generated (never hand-edited)"
  - "Manifest checksum detects tampering"
  - "Manifest consumed by migration engine and verification engine"
verification_checklist:
  - "[ ] Manifest generation is idempotent"
  - "[ ] Checksum algorithm documented"
  - "[ ] Manifest version scheme defined"
future_extensions:
  - "Manifest signing with platform private key"
  - "Manifest diff between versions"
---

# Platform Manifest

## Purpose

The Platform Manifest is the **single authoritative record** of everything the Platform contains
at a point in time. It is:
- **Generated** by `platform-verify manifest --generate`
- **Never hand-edited**
- **Validated** on every CI run
- **Consumed** by the Migration Engine and Verification Engine

---

## Manifest Structure

```yaml
# platform/MANIFEST.yaml
# GENERATED — do not edit manually
# Generator: platform.tools.verify.manifest

schema_version: "1.0"
platform_id: "platform:ai-studio:v2"
platform_version: "2.0.0"
generated_at: "2026-06-29T00:00:00Z"
generated_by: "platform-verify manifest --generate"
environment: "development"

# Integrity
checksum: "sha256:abc123..."
previous_checksum: "sha256:def456..."
checksum_algorithm: "sha256"

# Counts
module_count: 78
capability_count: 164
service_count: 6
runtime_count: 7
extension_count: 0

runtimes:
  - id: "runtime:event:v1"
    name: "event_runtime"
    version: "1.0.0"
    boot_order: 1
    state: "ACTIVE"
    module_count: 3
    capability_count: 3
  - id: "runtime:resource:v1"
    name: "resource_runtime"
    version: "1.0.0"
    boot_order: 2
    state: "ACTIVE"
    module_count: 3
    capability_count: 4
  - id: "runtime:provider:v1"
    name: "provider_runtime"
    version: "1.0.0"
    boot_order: 3
    state: "ACTIVE"
    module_count: 4
    capability_count: 3
  - id: "runtime:knowledge:v1"
    name: "knowledge_runtime"
    version: "1.0.0"
    boot_order: 4
    state: "ACTIVE"
    module_count: 5
    capability_count: 5
  - id: "runtime:ai:v2"
    name: "ai_runtime"
    version: "2.1.0"
    boot_order: 5
    state: "ACTIVE"
    module_count: 29
    capability_count: 130
  - id: "runtime:workflow:v1"
    name: "workflow_runtime"
    version: "1.0.0"
    boot_order: 6
    state: "PLANNED"
    module_count: 3
    capability_count: 3
  - id: "runtime:orchestration:v1"
    name: "orchestration_runtime"
    version: "1.0.0"
    boot_order: 7
    state: "PLANNED"
    module_count: 3
    capability_count: 3

services:
  - id: "service.auth"
    name: "Authentication"
    version: "1.0.0"
    stability: "STABLE"
  - id: "service.audit"
    name: "Audit"
    version: "1.0.0"
    stability: "STABLE"
  - id: "service.metrics"
    name: "Metrics"
    version: "1.0.0"
    stability: "STABLE"
  - id: "service.tracing"
    name: "Tracing"
    version: "1.0.0"
    stability: "BETA"
  - id: "service.cache"
    name: "Cache"
    version: "1.0.0"
    stability: "STABLE"
  - id: "service.notifications"
    name: "Notifications"
    version: "1.0.0"
    stability: "BETA"

packages:
  - id: "package:ai-studio-platform:2.0.0"
    distribution_name: "ai-studio-platform"
    version: "2.0.0"
    pyproject_path: "platform/pyproject.toml"

sdk:
  version: "2.0.0"
  stability: "STABLE"
  public_modules:
    - "platform.sdk.ai"
    - "platform.sdk.knowledge"
    - "platform.sdk.providers"
    - "platform.sdk.workflows"
    - "platform.sdk.resources"
    - "platform.sdk.events"
    - "platform.sdk.types"
    - "platform.sdk.exceptions"

migration_state:
  phase: "2.1D.0"
  status: "ARCHITECTURE"
  source_repositories:
    - path: "platform/"
      status: "CANONICAL"
    - path: "platform_kernel/"
      status: "TO_MERGE"
    - path: "platform_sdk/"
      status: "TO_MERGE"
  target_repository: "platform/"
```

---

## Checksum Algorithm

```python
def compute_checksum(manifest: dict) -> str:
    """
    Deterministic checksum over sorted module and capability IDs.
    Detects any module addition, removal, or renaming.
    """
    items = sorted(
        manifest["module_ids"] + manifest["capability_ids"] + manifest["service_ids"]
    )
    content = "\n".join(items).encode("utf-8")
    return "sha256:" + hashlib.sha256(content).hexdigest()
```

---

## Generation Command

```bash
# Generate from current directory state
platform-verify manifest --generate --output platform/MANIFEST.yaml

# Validate existing manifest
platform-verify manifest --validate

# Diff manifest against previous version
platform-verify manifest --diff platform/MANIFEST.yaml platform/MANIFEST.prev.yaml

# Show manifest summary
platform-verify manifest --summary
```

---

## CI Integration

```yaml
# .github/workflows/platform-ci.yml (or equivalent)
- name: Validate Platform Manifest
  run: platform-verify manifest --validate --strict
  # Fails if:
  # - manifest is stale (modules added/removed without regenerating)
  # - checksum mismatch
  # - required fields missing
```

---

## Manifest Lifecycle

```
DRAFT → VALIDATED → PUBLISHED → ARCHIVED
```

| State | Meaning |
|-------|---------|
| DRAFT | Generated but not validated |
| VALIDATED | Passed all verification checks |
| PUBLISHED | Used as reference in CI and migration engine |
| ARCHIVED | Superseded by newer manifest version |
