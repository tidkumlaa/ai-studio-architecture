---
knowledge_id: KNW-PLAT-ARCH-020
title: "Platform Data Structures"
status: AUTHORITATIVE
phase: "2.1D.0"
version: "1.0.0"
created: "2026-06-29"
purpose: "Define all canonical platform data structures: enums, dataclasses, Pydantic models, Protocols"
canonical_source: "architecture/docs/platform-consolidation/20-DATA-STRUCTURES.md"
dependencies:
  - "02-OBJECT-MODEL.md"
  - "08-SDK-MODEL.md"
related_documents:
  - "21-ALGORITHMS.md"
  - "07-KERNEL-MODEL.md"
acceptance_criteria:
  - "Every data structure has a Python type signature"
  - "Immutability decisions are explicit"
  - "Pydantic models specify protected_namespaces where needed"
verification_checklist:
  - "[ ] All structures use Pydantic or dataclass (no plain dicts)"
  - "[ ] Enums are exhaustive"
  - "[ ] No circular type references"
future_extensions:
  - "JSON Schema export from Pydantic models"
  - "API serialization schemas"
---

# Platform Data Structures

## Design Rules

| Rule | Rationale |
|------|-----------|
| Pydantic `BaseModel` for all external/API data | Validation + serialization |
| `@dataclass(frozen=True)` for immutable domain events | Prevents mutation after creation |
| Python `Protocol` for structural interfaces | Avoids import coupling |
| Python `ABC` for service contracts | Enforces implementation completeness |
| Enums for all finite value sets | Type safety, no magic strings |

---

## Kernel Data Structures

### PlatformConfig
```python
from pydantic_settings import BaseSettings

class PlatformConfig(BaseSettings):
    environment: str = "development"
    log_level: str = "INFO"
    event_bus_backend: str = "memory"
    cache_backend: str = "memory"
    metrics_backend: str = "memory"
    enable_tracing: bool = False

    model_config = SettingsConfigDict(
        env_prefix="PLATFORM_",
        env_file=".env",
    )
```

### RuntimeHealth
```python
@dataclass(frozen=True)
class RuntimeHealth:
    runtime_id: str
    state: str           # "READY" | "DEGRADED" | "FAILED"
    uptime_seconds: float
    error_count: int
    last_error: str | None
    checked_at: datetime
```

### PlatformEvent
```python
@dataclass(frozen=True)
class PlatformEvent:
    topic: str
    event_id: str
    occurred_at: datetime
    source_runtime: str
    correlation_id: str
    payload: dict
```

---

## Common Enums

### WorkloadType
```python
class WorkloadType(str, Enum):
    ARCHITECTURE = "architecture"
    SPECIFICATION = "specification"
    CODING = "coding"
    REFACTORING = "refactoring"
    TESTING = "testing"
    DOCUMENTATION = "documentation"
    REVIEW = "review"
    RESEARCH = "research"
    KNOWLEDGE = "knowledge"
    TRANSLATION = "translation"
    CONVERSATION = "conversation"
    CREATIVE = "creative"
    MUSIC = "music"
    VISION = "vision"
    VIDEO = "video"
    REASONING = "reasoning"
    TOOL_USE = "tool_use"
    REVERSE_ENGINEERING = "reverse_engineering"
    UNKNOWN = "unknown"
```

### ComplexityLevel
```python
class ComplexityLevel(str, Enum):
    TRIVIAL = "trivial"
    SIMPLE = "simple"
    MODERATE = "moderate"
    COMPLEX = "complex"
    EXPERT = "expert"
```

### IntelligenceLevel
```python
class IntelligenceLevel(str, Enum):
    BASIC = "basic"
    STANDARD = "standard"
    ADVANCED = "advanced"
    EXPERT = "expert"
    FRONTIER = "frontier"
```

### ExecutionStrategy
```python
class ExecutionStrategy(str, Enum):
    SINGLE_AGENT = "single_agent"
    MULTI_AGENT_PARALLEL = "multi_agent_parallel"
    MULTI_AGENT_SEQUENTIAL = "multi_agent_sequential"
    HYBRID = "hybrid"
```

---

## AI Runtime Data Structures (Phase 2.1A)

### ResourceQuota
```python
class ResourceQuota(BaseModel):
    account_id: str
    model_id: str
    daily_token_limit: int
    monthly_cost_limit: Decimal
    tokens_used_today: int = 0
    cost_used_month: Decimal = Decimal("0")
    reset_at: datetime
```

### UsageRecord
```python
@dataclass(frozen=True)
class UsageRecord:
    usage_id: str
    account_id: str
    model_id: str
    provider_id: str
    tokens_input: int
    tokens_output: int
    cost_usd: Decimal
    workload_type: str
    latency_ms: int
    recorded_at: datetime
```

### RoutingDecision
```python
class RoutingDecision(BaseModel):
    model_config = {"protected_namespaces": ()}
    decision_id: str
    model_id: str
    provider_id: str
    account_id: str
    estimated_cost: Decimal
    estimated_latency_ms: int
    strategy: str
    reason: str
```

---

## AI Runtime Data Structures (Phase 2.1B)

### WorkloadProfile
```python
class WorkloadProfile(BaseModel):
    workload_type: WorkloadType
    prompt_length: int
    language: str
    requires_vision: bool
    requires_tools: bool
    requires_reasoning: bool
    confidence: float
```

### ComplexityProfile
```python
class ComplexityProfile(BaseModel):
    level: ComplexityLevel
    score: float                   # 0.0–1.0
    parallelizable: bool
    estimated_subtasks: int
    recommended_retries: int
    estimated_duration_minutes: float
    requires_coordination: bool
    context_depth: int
    technical_depth: int
```

### TokenPrediction
```python
class TokenPrediction(BaseModel):
    model_config = {"protected_namespaces": ()}
    model_id: str
    predicted_input_tokens: int
    predicted_output_tokens: int
    confidence: float
    basis: str
```

### CostPrediction
```python
class CostPrediction(BaseModel):
    model_config = {"protected_namespaces": ()}
    model_id: str
    provider_id: str
    best_case_usd: Decimal
    expected_usd: Decimal
    worst_case_usd: Decimal
    currency: str = "USD"
```

### ExecutionPlan
```python
class ExecutionPlan(BaseModel):
    model_config = {"protected_namespaces": ()}
    plan_id: str
    strategy: ExecutionStrategy
    model_id: str
    provider_id: str
    account_id: str
    estimated_tokens: int
    estimated_cost_usd: Decimal
    estimated_latency_ms: int
    agents: list[AgentTask]
    retry_strategy: RetryStrategy
    fallback_chain: FallbackChain
```

### AgentTask
```python
class AgentTask(BaseModel):
    task_id: str
    description: str
    estimated_tokens: int
    dependencies: list[str]     # task_ids that must complete first
    parallel_group: int | None  # tasks in same group run concurrently
```

### RetryStrategy
```python
class RetryStrategy(BaseModel):
    max_retries: int
    backoff_seconds: float
    retry_on: list[str]         # error types
```

### FallbackChain
```python
class FallbackChain(BaseModel):
    model_config = {"protected_namespaces": ()}
    primary_model_id: str
    fallbacks: list[str]        # model IDs in priority order
```

### LearningStats
```python
class LearningStats(BaseModel):
    total_predictions: int
    mean_token_error: float
    accuracy_trend: str         # "improving" | "stable" | "degrading"
    provider_scores: dict[str, float]
    workload_accuracy: dict[str, float]
```

---

## Provider Runtime Data Structures

### ProviderSpec
```python
@dataclass(frozen=True)
class ProviderSpec:
    provider_id: str
    display_name: str
    capabilities: frozenset[str]
    base_url: str | None
    auth_type: str              # "api_key" | "oauth" | "none"
```

### ProviderHealth
```python
@dataclass(frozen=True)
class ProviderHealth:
    provider_id: str
    state: str                  # matches Provider Health State Machine
    error_rate: float
    p50_latency_ms: float
    p99_latency_ms: float
    checked_at: datetime
```

---

## Migration Data Structures

### MigrationPlan
```python
class MigrationPlan(BaseModel):
    plan_id: str
    phase: str
    source_paths: list[str]
    target_paths: list[str]
    import_rewrites: list[ImportRewrite]
    verification_checks: list[str]
    rollback_snapshot: str      # git commit SHA
    estimated_files_changed: int
```

### ImportRewrite
```python
@dataclass(frozen=True)
class ImportRewrite:
    old_pattern: str            # regex
    new_pattern: str            # replacement string
    scope: str                  # "all" | "runtime:ai" | specific path
    rule_id: str                # "DR-007" etc.
```
