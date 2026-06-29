---
knowledge_id: KNW-KOS-ARCH-004
title: "KOS Knowledge Object Model"
status: AUTHORITATIVE
phase: "3.0A"
version: "1.0.0"
created: "2026-06-30"
purpose: "Define the complete taxonomy of Knowledge Objects — every entity type in the ecosystem"
canonical_source: "architecture/docs/kos/04-OBJECT-MODEL.md"
dependencies:
  - "02-CORE-PHILOSOPHY.md"
  - "03-GOLDEN-RULES.md"
related_documents:
  - "05-KNOWLEDGE-GRAPH.md"
  - "07-KNOWLEDGE-REGISTRY.md"
  - "11-GOVERNANCE.md"
acceptance_criteria:
  - "All object types have a Python type signature"
  - "All objects share a common base"
  - "Every object in the spec is represented"
verification_checklist:
  - "[ ] Base KnowledgeObject defined"
  - "[ ] All object types from spec covered"
  - "[ ] Type hierarchy is acyclic"
future_extensions:
  - "Domain-specific object subtypes for Financial Runtime and Forex"
  - "Agent-generated object types"
---

# KOS Knowledge Object Model

## Base Knowledge Object

Every object in KOS inherits from `KnowledgeObject`:

```python
from __future__ import annotations
from pydantic import BaseModel, Field
from datetime import datetime
from enum import Enum

class KnowledgeObjectType(str, Enum):
    # Platform
    PLATFORM       = "platform"
    RUNTIME        = "runtime"
    KERNEL         = "kernel"
    SDK            = "sdk"
    SERVICE        = "service"
    MODULE         = "module"
    PACKAGE        = "package"
    # Architecture
    ARCHITECTURE   = "architecture"
    SPECIFICATION  = "specification"
    DECISION       = "decision"
    REQUIREMENT    = "requirement"
    PATTERN        = "pattern"
    ALGORITHM      = "algorithm"
    # Interface
    API            = "api"
    CAPABILITY     = "capability"
    PROVIDER       = "provider"
    # Quality
    TEST           = "test"
    BENCHMARK      = "benchmark"
    # Deployment
    DEPLOYMENT     = "deployment"
    CONFIGURATION  = "configuration"
    # Product
    PRODUCT        = "product"
    AGENT          = "agent"
    PROMPT         = "prompt"
    CONVERSATION   = "conversation"
    TASK           = "task"
    # Strategy
    STRATEGY       = "strategy"
    EXECUTION_PLAN = "execution_plan"
    # Data
    DATASET        = "dataset"
    # Financial
    MARKET         = "market"
    FOREX          = "forex"
    NEWS           = "news"
    # Meta
    DOCUMENT       = "document"
    KNOWLEDGE_BASE = "knowledge_base"

class KnowledgeObject(BaseModel):
    knowledge_id: str                    # "KNW-{domain}-{type}-{NNN}"
    object_type: KnowledgeObjectType
    name: str
    version: str                         # semver
    status: KnowledgeLifecycleStatus
    owner: str
    created_at: datetime
    updated_at: datetime
    description: str
    tags: list[str] = Field(default_factory=list)
    metadata: dict = Field(default_factory=dict)

    # Governance
    evidence: str = ""
    confidence: float = 0.0              # 0.0–1.0
    review_status: str = "PENDING"
    approved_by: str = ""
    approved_at: datetime | None = None

    # Relationships (populated by KnowledgeGraph)
    dependencies: list[str] = Field(default_factory=list)   # knowledge_ids
    dependents: list[str] = Field(default_factory=list)     # knowledge_ids (reverse)
    related: list[str] = Field(default_factory=list)        # knowledge_ids

    # Quality
    quality_score: float | None = None   # computed by KnowledgeQualityEngine
```

---

## Platform Object Types

### Platform
```python
class PlatformObject(KnowledgeObject):
    object_type: Literal[KnowledgeObjectType.PLATFORM] = KnowledgeObjectType.PLATFORM
    platform_id: str              # "platform:ai-studio:v2"
    canonical_path: str           # "platform/"
    runtime_ids: list[str]
    service_ids: list[str]
    package_id: str
```

### Runtime
```python
class RuntimeObject(KnowledgeObject):
    object_type: Literal[KnowledgeObjectType.RUNTIME] = KnowledgeObjectType.RUNTIME
    runtime_id: str               # "runtime:ai:v2"
    boot_order: int
    platform_id: str
    module_ids: list[str]
    capability_ids: list[str]
    python_path: str              # "platform.runtimes.ai_runtime"
```

### Kernel
```python
class KernelObject(KnowledgeObject):
    object_type: Literal[KnowledgeObjectType.KERNEL] = KnowledgeObjectType.KERNEL
    kernel_id: str
    subsystems: list[str]         # di, events, config, lifecycle, registry
    zero_external_deps: bool = True
```

### Service
```python
class ServiceObject(KnowledgeObject):
    object_type: Literal[KnowledgeObjectType.SERVICE] = KnowledgeObjectType.SERVICE
    service_id: str               # "service.auth"
    interface_path: str           # Python ABC dotted path
    implementations: list[str]
    stability: str
```

### Module
```python
class ModuleObject(KnowledgeObject):
    object_type: Literal[KnowledgeObjectType.MODULE] = KnowledgeObjectType.MODULE
    module_id: str
    runtime_id: str
    python_module: str
    capability_ids: list[str]
    is_public: bool
```

---

## Architecture Object Types

### Requirement
```python
class RequirementObject(KnowledgeObject):
    object_type: Literal[KnowledgeObjectType.REQUIREMENT] = KnowledgeObjectType.REQUIREMENT
    requirement_id: str           # "REQ-{domain}-{NNN}"
    priority: str                 # "MUST" | "SHOULD" | "MAY"
    source: str                   # who raised it
    acceptance_criteria: list[str]
    fulfilled_by: list[str]       # knowledge_ids of implementations
```

### Architecture
```python
class ArchitectureObject(KnowledgeObject):
    object_type: Literal[KnowledgeObjectType.ARCHITECTURE] = KnowledgeObjectType.ARCHITECTURE
    document_path: str
    phase: str
    covers_requirements: list[str]
    layer: int                    # 0–6 (STDLIB to PRODUCTS)
```

### Decision (ADR)
```python
class DecisionObject(KnowledgeObject):
    object_type: Literal[KnowledgeObjectType.DECISION] = KnowledgeObjectType.DECISION
    decision_id: str              # "ADR-NNN"
    context: str
    options_considered: list[str]
    chosen_option: str
    rationale: str
    consequences_positive: list[str]
    consequences_negative: list[str]
    review_date: datetime | None  # when to re-evaluate
```

### Algorithm
```python
class AlgorithmObject(KnowledgeObject):
    object_type: Literal[KnowledgeObjectType.ALGORITHM] = KnowledgeObjectType.ALGORITHM
    algorithm_id: str             # "ALG-{domain}-{NNN}"
    time_complexity: str          # "O(N log N)"
    space_complexity: str
    inputs: list[str]
    outputs: list[str]
    pseudocode: str
    implemented_in: list[str]     # module knowledge_ids
```

---

## Interface Object Types

### API
```python
class APIObject(KnowledgeObject):
    object_type: Literal[KnowledgeObjectType.API] = KnowledgeObjectType.API
    method: str                   # "GET" | "POST" | "PUT" | "DELETE"
    path: str                     # "/api/v1/intelligence/analyze"
    request_schema: dict
    response_schema: dict
    auth_required: bool
    served_by: str                # module knowledge_id
```

### Provider
```python
class ProviderObject(KnowledgeObject):
    object_type: Literal[KnowledgeObjectType.PROVIDER] = KnowledgeObjectType.PROVIDER
    provider_id: str              # "anthropic"
    capabilities: list[str]
    auth_type: str
    pricing_per_1m_input: str     # Decimal string
    pricing_per_1m_output: str
```

---

## Quality Object Types

### Test
```python
class TestObject(KnowledgeObject):
    object_type: Literal[KnowledgeObjectType.TEST] = KnowledgeObjectType.TEST
    test_type: str                # "unit" | "integration" | "e2e" | "architecture"
    tests_knowledge_id: str       # what this test verifies
    generated: bool               # True = generated by KnowledgeCompiler
    file_path: str
    assertion_count: int
```

### Benchmark
```python
class BenchmarkObject(KnowledgeObject):
    object_type: Literal[KnowledgeObjectType.BENCHMARK] = KnowledgeObjectType.BENCHMARK
    benchmarks_knowledge_id: str
    metric: str                   # "latency_ms_p99" | "throughput_rps"
    target_value: float
    actual_value: float | None
    measured_at: datetime | None
```

---

## Product & Agent Object Types

### Product
```python
class ProductObject(KnowledgeObject):
    object_type: Literal[KnowledgeObjectType.PRODUCT] = KnowledgeObjectType.PRODUCT
    product_id: str
    repository_path: str
    consumes_runtimes: list[str]  # runtime knowledge_ids
    consumes_apis: list[str]
```

### Agent
```python
class AgentObject(KnowledgeObject):
    object_type: Literal[KnowledgeObjectType.AGENT] = KnowledgeObjectType.AGENT
    agent_id: str
    capabilities: list[str]
    prompt_ids: list[str]
    model_id: str
    max_context_tokens: int
```

### Prompt
```python
class PromptObject(KnowledgeObject):
    object_type: Literal[KnowledgeObjectType.PROMPT] = KnowledgeObjectType.PROMPT
    prompt_id: str
    system_prompt: str
    used_by_agents: list[str]     # agent knowledge_ids
    workload_types: list[str]
```

---

## Financial Object Types

### Market
```python
class MarketObject(KnowledgeObject):
    object_type: Literal[KnowledgeObjectType.MARKET] = KnowledgeObjectType.MARKET
    market_id: str
    asset_class: str              # "forex" | "equity" | "crypto"
    symbols: list[str]
```

### Forex
```python
class ForexObject(KnowledgeObject):
    object_type: Literal[KnowledgeObjectType.FOREX] = KnowledgeObjectType.FOREX
    pair: str                     # "EUR/USD"
    base_currency: str
    quote_currency: str
    data_sources: list[str]
```

---

## Object ID Convention

```
KNW-{DOMAIN}-{TYPE}-{NNN}
```

| Domain | Prefix |
|--------|--------|
| Platform architecture | `KNW-PLAT-ARCH` |
| KOS architecture | `KNW-KOS-ARCH` |
| AI runtime | `KNW-AI-MOD` |
| Product | `KNW-PROD-{name}` |
| Financial | `KNW-FIN-{market}` |
| Algorithm | `KNW-ALG` |
| Decision | `KNW-ADR` |
| Requirement | `KNW-REQ` |
