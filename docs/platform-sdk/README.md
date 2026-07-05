---
knowledge_id: KA-SDK-001
version: "1.0.0"
status: approved
owner: Chief Platform Architect
phase: 2.0D.2.5A
created: 2026-06-29
review_date: 2026-12-29
canonical: true
domain: DOM-PLATFORM
capability: platform-sdk
type: specification
---

# Platform SDK

## The Reusable Primitive Layer

The Platform SDK is the foundational layer of all AI Studio components. Every
Runtime, Graph Engine, Compiler, and Intelligence module is built exclusively
from Platform SDK primitives. No product-specific code may exist at this layer.

---

## 1. SDK Catalog

| ID | Name | Core Abstraction | Primary Consumer |
|----|------|-----------------|-----------------|
| KA-SDK-002 | Collections SDK | Typed container hierarchy | All |
| KA-SDK-003 | Algorithms SDK | Composable algorithm pipeline | All |
| KA-SDK-004 | Graph SDK | Generic labeled property graph | Knowledge, Reasoning |
| KA-SDK-005 | Search SDK | Pluggable multi-strategy search | Knowledge, Intelligence |
| KA-SDK-006 | Index SDK | B+Tree, Hash, Inverted, Spatial | Graph, Knowledge |
| KA-SDK-007 | Parser SDK | Lexer + combinator + AST | Universal Reverse Engineering |
| KA-SDK-008 | Compiler SDK | Frontend → IR → Backend pipeline | Knowledge Compiler |
| KA-SDK-009 | Cache SDK | Multi-tier, pluggable eviction | All Runtimes |
| KA-SDK-010 | Optimization SDK | Combinatorial + constraint solver | Scheduling, Planning |
| KA-SDK-011 | Scheduling SDK | Priority, deadline, topology | Platform Kernel |
| KA-SDK-012 | Reasoning SDK | Rule engine, forward/backward chaining | Knowledge Reasoning |
| KA-SDK-013 | Statistics SDK | Descriptive, streaming, regression | Benchmark, Health |
| KA-SDK-014 | Similarity SDK | Distance metric suite | Search, Dedup |
| KA-SDK-015 | Validation SDK | Schema, type, constraint, semantic | All |
| KA-SDK-016 | Traversal SDK | Generic traversal protocol | Graph, Knowledge |

---

## 2. Mandatory SDK Contract

Every SDK must satisfy:

```
MUST:
  ✓ Be product-independent
  ✓ Be dependency-injected (no global state)
  ✓ Expose typed interfaces only
  ✓ Declare O-complexity for all operations
  ✓ Define preconditions and postconditions
  ✓ Provide extension points for specialization
  ✓ Be independently verifiable

MUST NOT:
  ✗ Import AI Runtime
  ✗ Import ProductFactory
  ✗ Import ContentFactory
  ✗ Hardcode domain names
  ✗ Hardcode file paths
  ✗ Use mutable global state
  ✗ Depend on another SDK implementation (only interfaces)
```

---

## 3. Layering Rules

```
Phase 3.0 — Autonomous Engineering
    │
Phase 2.3 — Universal Product Factory
    │
Phase 2.2 — Universal Reverse Engineering
    │
Phase 2.E — Knowledge Operating System
    │
Phase 2.0D.2.6 — Platform Kernel         ← Depends on SDK interfaces
    │
Phase 2.0D.2.5 — Platform SDK            ← This layer
    │
Python Standard Library + Approved packages
```

---

## 4. Design Invariants

1. **Interface-first**: Every SDK exposes a Protocol/ABC interface. Implementations are swappable.
2. **Complexity guaranteed**: Every operation states O-complexity in its contract.
3. **Immutable by default**: Data in, transformed data out. No mutation hidden in return types.
4. **No side effects at boundary**: SDK methods do not write to disk, network, or global state unless explicitly declared.
5. **Composable**: SDKs compose via protocol adapters, not inheritance.
6. **Verifiable**: Every SDK ships a `Verifier` that checks contract compliance at runtime.

---

## 5. Dependency Graph

```
Collections ←── Algorithms ←── Graph ←── Traversal
     │               │           │
     │               ▼           ▼
     │           Search ──────► Index
     │               │
     ▼               ▼
  Cache          Similarity
     │               │
     └───────────────┘
             │
             ▼
         Reasoning ←── Validation
             │
         Scheduling ←── Optimization
             │
         Statistics ←── Parser ←── Compiler
```

No SDK may create a dependency cycle.

---

## 6. Extension Protocol

Every SDK defines at least one extension point:

```python
class SDKExtensionPoint(Protocol):
    """Marker for SDK extension contracts."""
    def sdk_id(self) -> str: ...
    def sdk_version(self) -> str: ...
    def verify(self) -> VerificationResult: ...
```

---

## References

- [Phase 2.0D.2.6A: Platform Kernel](../platform-kernel/README.md) — Consumes SDK
- [Phase 2.0D.2 KA-KIP-001](../knowledge-intelligence/README.md) — First SDK consumer
- [Phase 2.0D.2.7: Architecture Constitution](../architecture-constitution/README.md) — Governs SDK
