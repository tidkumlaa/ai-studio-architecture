# KNW-KC-ARCH-005 — Object Inheritance

**Phase:** 3.0C  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Object Inheritance defines how Knowledge Object types relate to each other. Every concrete type derives from `KnowledgeObject` (the root abstract type) and may extend one or more intermediate abstract types. No type may bypass the universal schema.

---

## Inheritance Hierarchy

```
KnowledgeObject                              (abstract root — 02-UNIVERSAL-SCHEMA)
│
├── ArchitecturalObject                      (abstract — documents + decisions)
│   ├── ArchitectureObject                   (Phase 3.0B)
│   ├── DecisionObject                       (Phase 3.0B)
│   ├── RequirementObject                    (Phase 3.0B)
│   ├── PatternObject                        (Phase 3.0B)
│   └── SpecificationObject                  (Phase 3.0B)
│
├── PlatformObject                           (abstract — platform components)
│   ├── RuntimeObject                        (Phase 3.0B)
│   ├── KernelObject                         (Phase 3.0B)
│   ├── SDKObject                            (Phase 3.0B)
│   ├── PackageObject                        (Phase 3.0B)
│   └── PlatformRootObject                   (Phase 3.0B — the platform itself)
│
├── ImplementationObject                     (abstract — code units)
│   ├── ModuleObject                         (Phase 3.0B)
│   ├── ServiceObject                        (Phase 3.0B)
│   ├── AlgorithmObject                      (Phase 3.0B)
│   └── ConfigurationObject                  (Phase 3.0B)
│
├── InterfaceObject                          (abstract — contracts)
│   ├── APIObject                            (Phase 3.0B)
│   ├── CapabilityObject                     (Phase 3.0B)
│   └── ProviderObject                       (Phase 3.0B)
│
├── QualityObject                            (abstract — verification)
│   ├── TestObject                           (Phase 3.0B)
│   └── BenchmarkObject                      (Phase 3.0B)
│
├── ProductObject                            (abstract — user-facing)
│   ├── ProductRootObject                    (Phase 3.0B)
│   ├── AgentObject                          (Phase 3.0B)
│   ├── PromptObject                         (Phase 3.0B)
│   ├── ConversationObject                   (Phase 3.0B)
│   ├── TaskObject                           (Phase 3.0B)
│   ├── StrategyObject                       (Phase 3.0B)
│   └── ExecutionPlanObject                  (Phase 3.0B)
│
├── FinancialObject                          (abstract — financial domain)
│   ├── MarketObject                         (Phase 3.0B)
│   ├── ForexObject                          (Phase 3.0B)
│   ├── NewsObject                           (Phase 3.0B)
│   └── DatasetObject                        (Phase 3.0B)
│
└── MetaObject                               (abstract — knowledge about knowledge)
    ├── DocumentObject                       (future phase)
    └── KnowledgeBaseObject                  (future phase)
```

---

## Intermediate Abstract Types

### ArchitecturalObject
Additional required fields beyond `KnowledgeObject`:
```yaml
document_path: string        # path to the source document
phase: string                # e.g. "3.0C"
covers_requirements: list    # list[RequirementRef]
```

### PlatformObject (abstract)
Additional required fields:
```yaml
layer: int                   # 0–7 platform layer
stability: STABLE|BETA|EXPERIMENTAL
platform_id: string          # parent platform knowledge_id
```

### ImplementationObject
Additional required fields:
```yaml
python_path: string          # importable dotted path
runtime_id: string           # owning runtime
is_public: bool
```

### InterfaceObject
Additional required fields:
```yaml
contract_path: string        # Protocol / ABC dotted path
stability: STABLE|BETA|EXPERIMENTAL
versioned: bool
```

### QualityObject
Additional required fields:
```yaml
tests_target: string         # knowledge_id of tested object
automated: bool
ci_enforced: bool
```

### ProductObject (abstract)
Additional required fields:
```yaml
runtime_ids: list[string]
launch_command: string
```

### FinancialObject
Additional required fields:
```yaml
data_source: string
currency: string
```

---

## Inheritance Rules

### OI-001 — Single Root
Every concrete type has exactly one path to `KnowledgeObject`. No multiple inheritance of abstract types.

### OI-002 — Universal Schema Compliance
Every object — regardless of depth in the hierarchy — must satisfy all 16 sections of `02-UNIVERSAL-SCHEMA`. Fields from parent abstract types are merged, not replaced.

### OI-003 — Field Override Policy
A subtype may narrow (restrict) a field inherited from its parent type. A subtype may not widen (relax) a validation rule.

```
Example:
  KnowledgeObject.confidence: float 0.0..1.0
  TestObject.confidence: float 0.5..1.0    ← ALLOWED (narrowed)
  TestObject.confidence: float 0.0..2.0    ← FORBIDDEN (widened)
```

### OI-004 — No Lateral Inheritance
`ImplementationObject` may not inherit from `InterfaceObject`. All cross-type relationships are modelled as graph edges, not inheritance.

### OI-005 — Abstract Type Registration
Abstract types are registered in the schema registry with `is_abstract: true`. They may not be instantiated as objects.

### OI-006 — Extension Point
New concrete types must inherit from exactly one registered abstract type. New abstract types require Architecture Board approval.

---

## Type Resolution Protocol

```
TYPE_CHECK(object):
  1. Read object_type field
  2. Look up in TypeRegistry
  3. Walk inheritance chain to KnowledgeObject
  4. Validate each abstract type's required fields
  5. Validate Universal Schema (02)
  6. Return VALID | INVALID(reasons)
```

---

## Type Catalog Summary

| Category | Abstract Type | Concrete Types | Count |
|----------|--------------|----------------|-------|
| Architecture | ArchitecturalObject | Architecture, Decision, Requirement, Pattern, Specification | 5 |
| Platform | PlatformObject | Runtime, Kernel, SDK, Package, Platform | 5 |
| Implementation | ImplementationObject | Module, Service, Algorithm, Configuration | 4 |
| Interface | InterfaceObject | API, Capability, Provider | 3 |
| Quality | QualityObject | Test, Benchmark | 2 |
| Product | ProductObject | Product, Agent, Prompt, Conversation, Task, Strategy, ExecutionPlan | 7 |
| Financial | FinancialObject | Market, Forex, News, Dataset | 4 |
| Meta | MetaObject | Document, KnowledgeBase | 2 |
| **Total** | **8 abstract** | **32 concrete** | **33 types** |

---

## Cross-References

- Universal Schema → `02-UNIVERSAL-SCHEMA`
- Phase 3.0B objects → `platform/knowledge_runtime/objects/`
- Object Registry → `15-OBJECT-REGISTRY`
- Type codes → `03-NAMESPACE-SYSTEM`
