---
knowledge_id: KNW-PLAT-ARCH-002
title: "Platform Object Model"
status: AUTHORITATIVE
phase: "2.1D.0"
version: "1.0.0"
created: "2026-06-29"
purpose: "Define every platform object with identity, lifecycle, capabilities, and relationships"
canonical_source: "architecture/docs/platform-consolidation/02-OBJECT-MODEL.md"
dependencies:
  - "01-VISION.md"
related_documents:
  - "13-METADATA-STANDARD.md"
  - "15-CAPABILITY-REGISTRY.md"
  - "18-EVENT-MODEL.md"
  - "19-STATE-MACHINES.md"
acceptance_criteria:
  - "Every object has an identity scheme"
  - "Every object has a defined lifecycle"
  - "Every relationship is bidirectional and named"
  - "No object is left without an owner"
verification_checklist:
  - "[ ] All 16 objects defined"
  - "[ ] Every object has identity, lifecycle, owner, capabilities"
  - "[ ] Relationship graph is acyclic at the object level"
  - "[ ] Matches metadata schemas in 13-METADATA-STANDARD.md"
future_extensions:
  - "Versioned object model (semver per object type)"
  - "Object serialisation to Knowledge Graph nodes"
---

# Platform Object Model

## Object Hierarchy

```
Platform
├── PlatformKernel
├── PlatformSDK
├── PlatformRuntime  (7 instances)
│   └── PlatformModule  (N per runtime)
├── PlatformService  (N instances)
├── PlatformPackage
├── PlatformExtension
├── PlatformCapability
├── PlatformEvent
├── PlatformAPI
├── PlatformResource
├── PlatformDocument
├── PlatformRegistry (4 instances)
└── PlatformManifest
```

---

## Object Specifications

### Platform
**Identity:** `platform:ai-studio:v{major}`
**Owner:** Architecture Team
**Lifecycle:** INITIALIZING → RUNNING → DEGRADED → SHUTDOWN
**Capabilities:** hosts all runtimes, services, kernel, SDK
**Metadata fields:**
```yaml
id: string          # "platform:ai-studio:v2"
version: semver     # "2.0.0"
runtimes: list[str] # runtime IDs
services: list[str] # service IDs
state: enum         # INITIALIZING | RUNNING | DEGRADED | SHUTDOWN
kernel_version: str
sdk_version: str
```
**Relationships:**
- owns 1 PlatformKernel
- owns 1 PlatformSDK
- owns N PlatformRuntime
- owns N PlatformService
- owns 1 PlatformManifest

---

### PlatformKernel
**Identity:** `kernel:{version}`
**Owner:** Kernel Team
**Lifecycle:** COLD → BOOTING → READY → SHUTDOWN
**Capabilities:** dependency_injection, event_bus, lifecycle_management, configuration, service_locator
**Constraints:** ZERO external runtime dependencies; imports stdlib + `platform.common` only
**Metadata fields:**
```yaml
id: string
version: semver
capabilities: list[str]
boot_order: int        # 0 — boots first
dependencies: []       # always empty
```
**Relationships:**
- provides services to all PlatformRuntime
- provides event bus to all PlatformRuntime
- owns no PlatformModule directly

---

### PlatformSDK
**Identity:** `sdk:{version}`
**Owner:** SDK Team
**Lifecycle:** INACTIVE → ACTIVE → DEPRECATED
**Capabilities:** public_api, type_facade, versioned_contracts, backwards_compatibility
**Constraints:** imports from `platform.kernel` and `platform.services` only; never from runtimes directly
**Metadata fields:**
```yaml
id: string
version: semver
public_namespaces: list[str]
stability: enum    # STABLE | BETA | EXPERIMENTAL
deprecated_since: str | null
```
**Relationships:**
- wraps PlatformService (provides stable facades)
- consumed by Products only

---

### PlatformRuntime
**Identity:** `runtime:{name}:{version}`
**Owner:** Runtime Team (per runtime)
**Lifecycle:** INACTIVE → LOADING → READY → PAUSED → SHUTDOWN → FAILED
**Capabilities:** runtime-specific (see 06-RUNTIME-MODEL.md)
**Metadata fields:**
```yaml
id: string           # "runtime:ai:v2"
name: string         # "ai_runtime"
version: semver
modules: list[str]   # module IDs owned
depends_on: list[str]  # kernel capabilities only
exposes: list[str]   # capability IDs
state: enum
```
**Relationships:**
- owns N PlatformModule
- communicates with other runtimes via PlatformEvent only
- registers capabilities with PlatformRegistry

---

### PlatformModule
**Identity:** `module:{runtime}:{name}:{version}`
**Owner:** Runtime Team (inherits from parent runtime)
**Lifecycle:** REGISTERED → LOADING → ACTIVE → DISABLED → UNLOADED
**Capabilities:** module-specific
**Metadata fields:**
```yaml
id: string
runtime_id: string
name: string
version: semver
python_module: str   # importable dotted path
public: bool         # visible in SDK?
capabilities: list[str]
dependencies: list[str]  # module IDs within same runtime
```
**Relationships:**
- member of exactly one PlatformRuntime
- may depend on modules within same runtime
- never depends on modules in other runtimes

---

### PlatformService
**Identity:** `service:{name}:{version}`
**Owner:** Services Team
**Lifecycle:** INACTIVE → STARTING → RUNNING → STOPPING → STOPPED
**Capabilities:** service-specific (auth, audit, metrics, tracing)
**Metadata fields:**
```yaml
id: string
name: string
version: semver
interface: str    # Python ABC dotted path
implementations: list[str]
depends_on: list[str]   # kernel capabilities + other services
```
**Relationships:**
- depends on PlatformKernel
- may depend on other PlatformService (acyclic)
- exposed via PlatformSDK

---

### PlatformPackage
**Identity:** `package:{distribution_name}:{version}`
**Owner:** Platform Team
**Lifecycle:** UNPUBLISHED → PUBLISHED → YANKED
**Capabilities:** python_distribution
**Metadata fields:**
```yaml
id: string
distribution_name: str   # PyPI name e.g. "ai-studio-platform"
version: semver
pyproject_path: str
namespace: str           # Python namespace e.g. "platform"
includes: list[str]      # module glob patterns
```
**Relationships:**
- contains N PlatformModule
- published as single editable install

---

### PlatformExtension
**Identity:** `extension:{provider}:{name}:{version}`
**Owner:** Extension Author
**Lifecycle:** UNREGISTERED → REGISTERED → ACTIVE → DISABLED
**Capabilities:** extension-specific (provider plugins, tool adapters)
**Metadata fields:**
```yaml
id: string
provider_id: str
extension_type: enum   # PROVIDER | TOOL | ADAPTER | SINK
entry_point: str       # Python importable class
capabilities: list[str]
```
**Relationships:**
- registered in PlatformRegistry
- invoked by PlatformRuntime (provider_runtime)
- never imported directly by Platform core

---

### PlatformCapability
**Identity:** `capability:{domain}:{name}:{version}`
**Owner:** owning Runtime or Kernel
**Lifecycle:** DECLARED → REGISTERED → AVAILABLE → DEPRECATED
**Metadata fields:**
```yaml
id: string
domain: str    # "kernel" | "runtime:ai" | "service:auth" etc.
name: str
version: semver
interface: str   # Python Protocol or ABC dotted path
provided_by: str   # object ID
consumers: list[str]
```
**Relationships:**
- declared by PlatformKernel, PlatformRuntime, or PlatformService
- consumed by PlatformModule or PlatformSDK
- registered in PlatformRegistry (capability)

---

### PlatformEvent
**Identity:** `event:{domain}:{name}:{version}`
**Owner:** producing Runtime or Service
**Lifecycle:** PUBLISHED → DISPATCHED → CONSUMED | DEAD_LETTER
**Metadata fields:**
```yaml
id: string
domain: str
name: str
version: semver
producer: str     # object ID
consumers: list[str]   # object IDs
payload_schema: str    # JSON Schema path
ordering_guarantee: enum  # NONE | FIFO | CAUSAL
```
**Relationships:**
- produced by PlatformRuntime or PlatformService
- dispatched via PlatformKernel event bus
- consumed by PlatformRuntime or PlatformService

---

### PlatformAPI
**Identity:** `api:{version}:{resource}`
**Owner:** API Team
**Lifecycle:** DRAFT → STABLE → DEPRECATED → REMOVED
**Metadata fields:**
```yaml
id: string
version: str     # "v1", "v2"
resource: str    # REST resource path
methods: list[str]
stability: enum  # STABLE | BETA | EXPERIMENTAL
openapi_path: str
```
**Relationships:**
- backed by PlatformService
- versioned independently

---

### PlatformResource
**Identity:** `resource:{type}:{id}`
**Owner:** resource_runtime
**Lifecycle:** PENDING → ALLOCATED → IN_USE → RELEASING → RELEASED
**Metadata fields:**
```yaml
id: string
resource_type: enum   # TOKEN_QUOTA | COMPUTE_SLOT | MEMORY | CONNECTION
owner_org_id: str
limit: number
used: number
state: enum
```
**Relationships:**
- managed by resource_runtime
- allocated to PlatformModule on request

---

### PlatformDocument
**Identity:** `doc:KNW-PLAT-ARCH-{NNN}`
**Owner:** Architecture Team
**Lifecycle:** DRAFT → REVIEW → AUTHORITATIVE → FROZEN → SUPERSEDED
**Metadata fields:**
```yaml
knowledge_id: str
title: str
status: enum
version: semver
created: date
dependencies: list[str]
```
**Relationships:**
- cross-referenced in other PlatformDocument
- indexed in PlatformManifest
- registered in Knowledge Runtime

---

### PlatformRegistry
**Identity:** `registry:{type}`
**Owner:** Kernel Team
**Lifecycle:** EMPTY → LOADING → ACTIVE → LOCKED
**Types:** capability | service | module | extension
**Metadata fields:**
```yaml
id: string
registry_type: enum
entries: list[str]     # registered object IDs
version: int           # monotonic generation counter
```
**Relationships:**
- managed by PlatformKernel
- queried by PlatformRuntime and PlatformSDK

---

### PlatformManifest
**Identity:** `manifest:platform:{version}`
**Owner:** Platform Team
**Lifecycle:** DRAFT → VALIDATED → PUBLISHED
**Metadata fields:**
```yaml
id: string
platform_version: semver
runtimes: list[str]
services: list[str]
packages: list[str]
capabilities: list[str]
checksum: str
generated_at: datetime
```
**Relationships:**
- summarises entire Platform state
- consumed by Migration Engine and Verification Engine
- registered in Knowledge Runtime

---

## Relationship Matrix

| From \ To | Kernel | SDK | Runtime | Module | Service | Extension |
|-----------|--------|-----|---------|--------|---------|-----------|
| Platform | owns | owns | owns | — | owns | — |
| Kernel | — | provides | provides | provides | provides | — |
| SDK | uses | — | — | — | wraps | — |
| Runtime | uses | — | — | owns | uses | loads |
| Module | uses | — | — | uses* | uses | — |
| Service | uses | — | — | — | uses* | — |

*within same container only; * = acyclic
