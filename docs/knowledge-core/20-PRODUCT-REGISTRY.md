# KNW-KC-ARCH-020 — Product Registry

**Phase:** 3.0C  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

The Product Registry catalogs all user-facing products built on the AI Studio Platform. Every product declares its runtime dependencies, entry points, and capability requirements before deployment.

---

## Registered Products

| Product ID | Name | Type | Runtimes | Status |
|-----------|------|------|----------|--------|
| `product:desktop:v1` | AI Studio Desktop | application | kernel, sdk, ai | PLANNED |
| `product:api-server:v1` | AI Studio API Server | api | kernel, sdk, ai | PLANNED |
| `product:cli:v1` | Platform CLI | cli | kernel, sdk | CANONICAL |
| `product:kos-cli:v1` | KOS CLI | cli | kernel, sdk, knowledge | BETA |
| `product:mythic-realms:v1` | Mythic Realms | application | kernel, sdk, ai, content | PLANNED |

---

## Product Entry Format

```yaml
# knowledge/registry/products/product.desktop.v1.yaml
identity:
  knowledge_id: "KNW-PROD-APP-desktop-v1"
  canonical_name: "product.desktop.v1"
  knowledge_uri: "knw://product/desktop/v1"
  namespace: "product"
  version: "1.0.0"
  owner: "team:product"

product_spec:
  product_id: "product:desktop:v1"
  product_type: application
  runtime_ids:
    - "runtime:kernel:v1"
    - "runtime:sdk:v2"
    - "runtime:ai:v2"
  entry_point: "desktop.app:main"
  launch_command: "ai-studio"
  build_artifact: "dist/ai-studio-{version}-{platform}.exe"

  required_capabilities:
    - "capability:ai:model_selection:v2"
    - "capability:ai:workload_routing:v2"
    - "capability:platform:auth:v2"

  configuration:
    config_schema: "KNW-PROD-CFG-desktop"
    required_env_vars:
      - "ANTHROPIC_API_KEY"
      - "OPENAI_API_KEY"

  target_platforms:
    - windows: ">=10"
    - macos: ">=13"

lifecycle:
  status: PLANNED
```

---

## Product Registry Rules

### PR-001 — Runtime Pre-registration
All `runtime_ids` must be registered in the Runtime Registry with status ≥ BETA before the product may reach APPROVED.

### PR-002 — Capability Declaration
All `required_capabilities` must be registered in the Capability Registry before the product may reach VERIFIED.

### PR-003 — No Hardcoded Paths
Products reference runtimes and capabilities by knowledge_id, not by file path or Python import path.

### PR-004 — Entry Point Stability
`entry_point` and `launch_command` are immutable after CANONICAL.

### PR-005 — Configuration Schema
Every product must have a registered `ConfigurationObject` before reaching APPROVED.

---

## Product Registry Protocol

```python
class ProductRegistry(KnowledgeRegistryProtocol):
    def get_runtime_deps(self, product_id: str) -> list[str]: ...
    def get_capabilities_required(self, product_id: str) -> list[str]: ...
    def get_by_type(self, product_type: str) -> list[ProductObject]: ...
    def get_entry_point(self, product_id: str) -> str: ...
    def validate_deps_available(self, product_id: str) -> ValidationResult: ...
    def get_build_config(self, product_id: str) -> BuildConfig: ...
```

---

## Cross-References

- Registry base contract → `14-REGISTRY-ARCHITECTURE`
- Product objects → Phase 3.0B `knowledge_runtime/objects/product.py`
- Runtime dependencies → `19-RUNTIME-REGISTRY`
- Capability registry → Phase 2.1D.0 `15-CAPABILITY-REGISTRY`
