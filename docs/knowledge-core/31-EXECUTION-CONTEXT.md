# KNW-KC-ARCH-031 — Execution Context

**Phase:** 3.0C  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

An Execution Context is the primary output of the Knowledge Compiler. It packages everything needed to execute a plan, run a service, deploy a product, or conduct research — derived entirely from compiled Knowledge Objects.

---

## Execution Context Schema

```python
@dataclass
class ExecutionContext:
    # Identity
    context_id: str                          # "CTX-{plan_type}-{timestamp}-{hash}"
    plan_type: PlanType                      # SOFTWARE|AGENT|TESTING|DEPLOYMENT|RESEARCH|TRADING
    knowledge_ids: list[str]                 # all KO IDs that contributed to this context
    compiled_at: datetime
    compiler_version: str                    # KOS compiler version
    confidence: float                        # min(contributing_object.confidence)

    # Knowledge Content
    requirements: list[RequirementObject]
    architecture: list[ArchitectureObject]
    decisions: list[DecisionObject]
    services: list[ServiceObject]
    interfaces: list[APIObject | CapabilityObject]
    policies: list[str]                      # governance rules that apply
    security: SecurityContext
    knowledge_refs: list[str]                # URIs of all referenced KOs

    # Operational
    evidence: list[EvidenceRecord]
    tests: list[TestObject]
    benchmarks: list[BenchmarkObject]
    constraints: list[str]                   # must-not-violate rules
    risks: list[RiskItem]
    dependencies: list[DependencyRef]

    # Quality
    quality_scores: dict[str, float]         # knowledge_id → quality_score
    traceability_coverage: float
    evidence_coverage: float

    # Execution-Specific
    execution_plan: ExecutionPlanObject | None
    tasks: list[TaskObject]
    agent_configs: list[AgentObject]
    strategy: StrategyObject | None
    dataset_refs: list[str]
```

---

## Security Context

```python
@dataclass
class SecurityContext:
    classification: str                      # PUBLIC|INTERNAL|CONFIDENTIAL|SECRET
    required_roles: list[str]
    forbidden_operations: list[str]
    encryption_required: bool
    audit_required: bool
    secrets_required: list[str]             # env var names (not values)
    access_policy: str                      # knowledge_id of AccessPolicyObject
```

---

## Risk Item

```python
@dataclass
class RiskItem:
    risk_id: str
    description: str
    severity: str                           # LOW|MEDIUM|HIGH|CRITICAL
    object_id: str                          # which KO carries this risk
    risk_score: float
    mitigation: str
```

---

## Execution Context by Plan Type

### SOFTWARE Context
```
Required Knowledge:
  requirements: ≥ 1 MUST-priority requirement
  architecture: ≥ 1 CANONICAL architecture
  decisions: all applicable ADRs
  services: all services the software calls
  interfaces: all APIs exposed
  tests: all test objects
  dependencies: all hard module dependencies
```

### AGENT Context
```
Required Knowledge:
  agent_configs: ≥ 1 AgentObject
  prompts: all PromptObjects for this agent
  capabilities: all capability requirements
  policies: agent safety policies
  constraints: rate limits, token budgets, permission boundaries
```

### TESTING Context
```
Required Knowledge:
  tests: full test inventory for scope
  benchmarks: all performance benchmarks
  datasets: test fixtures and evaluation datasets
  modules: all modules under test
  requirements: requirements being verified
```

### DEPLOYMENT Context
```
Required Knowledge:
  product: ProductObject being deployed
  runtime_configs: all runtime registry entries
  configuration: ConfigurationObject
  security: full SecurityContext
  rollback_plan: Snapshot reference for rollback
```

### RESEARCH Context
```
Required Knowledge:
  datasets: research datasets
  algorithms: algorithms to apply
  strategy: research strategy
  evidence: existing evidence to build on
  related_work: references
```

### TRADING Context
```
Required Knowledge:
  forex/market objects: instruments to trade
  strategy: trading strategy
  algorithms: signal algorithms
  datasets: historical data
  constraints: risk limits
  evidence: backtesting evidence
```

---

## Compilation Rules

### CR-001 — No Unverified Requirements
All RequirementObjects in the context must be status ≥ VERIFIED.

### CR-002 — No Unresolved Dependencies
All dependency knowledge_ids must resolve in the registry.

### CR-003 — Minimum Context Confidence
`context.confidence = min(obj.composite_confidence for obj in all contributing objects)`  
Contexts with confidence < 0.50 are flagged as `LOW_CONFIDENCE`.

### CR-004 — Evidence Required
At least one EvidenceRecord must exist for each CANONICAL requirement in scope.

### CR-005 — Test Coverage Gate
For SOFTWARE and DEPLOYMENT contexts: `traceability_coverage ≥ 0.80` required.

### CR-006 — Security Consistency
If any contributing object has `classification: CONFIDENTIAL`, the entire context is classified CONFIDENTIAL.

---

## Execution Context Lifecycle

```
DRAFT        # context assembled, not yet validated
VALIDATED    # all compilation rules passed
PUBLISHED    # context made available for consumers
EXPIRED      # TTL exceeded or source objects changed
REVOKED      # security issue or quality degradation of source objects
```

---

## Context Invalidation

A published context is automatically EXPIRED when:
- Any contributing `KnowledgeObject` is updated (version bump)
- Any contributing evidence record becomes stale
- `traceability_coverage` drops below threshold
- Security classification of source objects changes

Consumers must re-compile the context on expiry.

---

## Execution Context Protocol

```python
class ExecutionContextEngine(Protocol):
    def compile(
        self,
        plan_type: PlanType,
        scope: list[str],
        filters: ContextFilters,
    ) -> ExecutionContext: ...

    def validate(self, context: ExecutionContext) -> ContextValidationResult: ...

    def publish(self, context: ExecutionContext) -> str: ...       # context_id

    def get(self, context_id: str) -> ExecutionContext: ...

    def invalidate(self, context_id: str, reason: str) -> None: ...

    def get_active_contexts(self, plan_type: PlanType | None) -> list[ExecutionContext]: ...
```

---

## Cross-References

- Input: all 33 object types from → Phase 3.0B
- Evidence from → `10-EVIDENCE-ENGINE`
- Quality from → `11-QUALITY-ENGINE`
- Risks computed by → `30-REASONING-MODEL`
- Tests from → `39-TEST-STRATEGY`
- Security context from → `32-METADATA-STANDARD`
