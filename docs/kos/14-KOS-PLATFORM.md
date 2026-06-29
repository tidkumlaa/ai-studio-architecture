---
knowledge_id: KNW-KOS-ARCH-014
title: "KOS as Platform Owner"
status: AUTHORITATIVE
phase: "3.0A"
version: "1.0.0"
created: "2026-06-30"
purpose: "Define how KOS owns, governs, and coordinates the entire AI Studio platform hierarchy"
canonical_source: "architecture/docs/kos/14-KOS-PLATFORM.md"
dependencies:
  - "01-VISION.md"
  - "03-GOLDEN-RULES.md"
  - "04-OBJECT-MODEL.md"
related_documents:
  - "15-FUTURE-SYSTEMS.md"
  - "16-IMPLEMENTATION-ROADMAP.md"
acceptance_criteria:
  - "KOS ownership hierarchy is fully defined"
  - "No subsystem exists outside KOS scope"
  - "Integration points between KOS and each subsystem are specified"
verification_checklist:
  - "[ ] All 9 subsystems listed"
  - "[ ] KOS → subsystem integration pattern defined"
  - "[ ] Governance boundary is clear"
future_extensions:
  - "KOS as foundation for multi-tenant AI Studio instances"
---

# KOS as Platform Owner

## The Ownership Model

KOS is not a component of the platform. KOS **owns** the platform.

```
╔═══════════════════════════════════════════════════════════╗
║                                                           ║
║                Knowledge Operating System                 ║
║                                                           ║
║   Registry  Graph  Compiler  Query  Reason  Execute       ║
║   Governance  Quality  Lifecycle                          ║
║                                                           ║
╠═══════════════════════════════════════════════════════════╣
║                                                           ║
║  Platform Kernel  ←──────────────── knowledge/kernel/     ║
║  SDK              ←──────────────── knowledge/sdk/        ║
║  Services         ←──────────────── knowledge/services/   ║
║  AI Runtime       ←──────────────── knowledge/ai/         ║
║  Desktop          ←──────────────── knowledge/desktop/    ║
║  API              ←──────────────── knowledge/api/        ║
║  Products         ←──────────────── knowledge/products/   ║
║  Deployment       ←──────────────── knowledge/deployment/ ║
║  Agents           ←──────────────── knowledge/agents/     ║
║                                                           ║
╚═══════════════════════════════════════════════════════════╝
```

Every arrow `←` means: "this subsystem's architecture, modules, APIs, tests,
and deployments are Knowledge Objects in KOS."

---

## KOS Ownership by Subsystem

### 1. Platform Kernel

**What KOS owns:**
- DI container design → `KernelObject` knowledge objects
- Event bus specification → `CapabilityObject` + `AlgorithmObject`
- Config schema → `ConfigurationObject`
- Boot sequence → `ArchitectureObject`

**KOS integration:**
```python
# At kernel boot: assert all kernel knowledge objects are CANONICAL
kos_client.assert_domain_canonical("kernel")
```

**Knowledge namespace:** `KNW-KERN-*`

---

### 2. SDK

**What KOS owns:**
- All 8 public SDK modules → `ModuleObject` (is_public=True)
- Every public function → `APIObject` (type=SDK_FUNCTION)
- Stability contracts → `DecisionObject`
- Breaking change policy → `RequirementObject`

**KOS integration:**
- SDK documentation is generated from KOS (not hand-written)
- SDK changelog is computed from KOS version diffs

**Knowledge namespace:** `KNW-SDK-*`

---

### 3. Services

**What KOS owns:**
- All 6 service ABCs → `ServiceObject`
- Service dependency graph → graph edges
- Implementation registry → `ModuleObject` (each implementation)
- DI registration → `ConfigurationObject`

**KOS integration:**
```python
# ServiceFactory resolves implementations via KOS registry
service = kos_registry.get_canonical_implementation("service.auth")
```

**Knowledge namespace:** `KNW-SVC-*`

---

### 4. AI Runtime

**What KOS owns:**
- All 29 module definitions → `ModuleObject`
- All 32 capabilities → `CapabilityObject`
- All 10 algorithms (workload detection, EMA, routing, etc.) → `AlgorithmObject`
- All API endpoints → `APIObject`
- All 160+ tests → `TestObject`
- Performance budgets → `BenchmarkObject`

**KOS integration:**
- All tests are annotated with `@pytest.mark.kos(knowledge_id)`
- Knowledge Compiler generates runtime stubs from module knowledge objects

**Knowledge namespace:** `KNW-AI-*`

---

### 5. Desktop

**What KOS owns:**
- All 6 dashboard panels → `ProductObject` subtypes
- Panel data sources → graph edges to capability objects
- UI component specifications → `SpecificationObject`

**KOS integration:**
- Dashboard panel layouts generated from knowledge objects
- Panel data contracts validated against capability knowledge objects

**Knowledge namespace:** `KNW-DESK-*`

---

### 6. API

**What KOS owns:**
- Every HTTP endpoint → `APIObject`
- Request/response schemas → embedded in `APIObject`
- Authentication requirements → `RequirementObject`
- Rate limiting specs → `BenchmarkObject`

**KOS integration:**
- OpenAPI spec generated from KOS APIObjects
- API gateway validates routes against KOS registry at boot

**Knowledge namespace:** `KNW-API-*`

---

### 7. Products

**What KOS owns:**
- Each product → `ProductObject`
- Product capabilities → graph edges to runtime capabilities
- Product requirements → `RequirementObject`
- Product tests → `TestObject`

**KOS integration:**
- Products declare their knowledge dependencies at build time
- Product CI validates against KOS knowledge graph

**Knowledge namespace:** `KNW-PROD-{product_name}-*`

---

### 8. Deployment

**What KOS owns:**
- Deployment environments → `DeploymentObject`
- Deployment plans → `ExecutionPlan` (type=deployment)
- Pre-deployment checks → linked `TestObject` list
- Rollback procedures → `AlgorithmObject`

**KOS integration:**
- Deployment is gated on: `platform-kos validate --deployment` (GR-008 full traceability)
- Deployment manifest references all knowledge_ids of deployed artifacts

**Knowledge namespace:** `KNW-DEPLOY-*`

---

### 9. Agents

**What KOS owns:**
- Each agent definition → `AgentObject`
- Each prompt → `PromptObject`
- Agent capabilities → graph edges to capability objects
- Agent task types → `TaskObject`

**KOS integration:**
- Agents are instantiated from their KOS AgentObject definitions
- Prompts are versioned knowledge objects, not strings in code

**Knowledge namespace:** `KNW-AGENT-*`

---

## KOS Client Interface

Every subsystem that wants to integrate with KOS uses the `KOSClient`:

```python
class KOSClient:
    def __init__(self, config: PlatformConfig) -> None: ...

    # Registry operations
    def get(self, knowledge_id: str) -> KnowledgeObject | None: ...
    def search(self, query: str) -> list[KnowledgeObject]: ...
    def register(self, obj: KnowledgeObject) -> None: ...

    # Validation
    def assert_canonical(self, knowledge_id: str) -> None: ...
    def assert_domain_canonical(self, domain: str) -> None: ...
    # Raises KOSViolationError if any object in domain is not CANONICAL

    # Compiler
    def compile(self, targets: list[ArtifactType]) -> CompilationResult: ...

    # Quality
    def quality_score(self, knowledge_id: str) -> KnowledgeQualityScore: ...

    # Traceability
    def trace(self, knowledge_id: str) -> TraceabilityChain: ...
```

---

## Governance Boundary

**KOS governs:**
- What knowledge objects exist (registry)
- Their quality and lifecycle state
- Their relationships (graph)
- What they compile to (compiler)

**KOS does NOT govern:**
- Internal business logic within a module
- Provider-specific implementation details
- UI styling and layout details
- Runtime execution behavior

The boundary is: KOS owns the **specification** of everything. The **implementation** is owned by the respective subsystem teams, verified against the specification.
