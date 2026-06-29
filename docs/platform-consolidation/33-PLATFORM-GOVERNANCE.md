---
knowledge_id: KNW-PLAT-ARCH-033
title: "Platform Governance"
status: AUTHORITATIVE
phase: "2.1D.0"
version: "1.0.0"
created: "2026-06-29"
purpose: "Define ownership, change control, review requirements, and decision authority for the platform"
canonical_source: "architecture/docs/platform-consolidation/33-PLATFORM-GOVERNANCE.md"
dependencies:
  - "01-VISION.md"
  - "10-DEPENDENCY-RULES.md"
  - "11-LAYER-RULES.md"
related_documents:
  - "34-ARCHITECTURE-DECISIONS.md"
  - "35-ARCHITECTURE-FREEZE.md"
acceptance_criteria:
  - "Every layer has an owner"
  - "Change approval requirements are documented per change type"
  - "Architecture decision process is defined"
verification_checklist:
  - "[ ] All ownership assignments documented"
  - "[ ] PR requirements per change type defined"
  - "[ ] Exception process defined"
future_extensions:
  - "Automated ownership checks in CI"
  - "Governance dashboard"
---

# Platform Governance

## Ownership Model

### Layer Owners

| Layer | Owner | Authority |
|-------|-------|-----------|
| STDLIB (0) | CPython / standard library | External |
| COMMON (1) | Platform Team | Approve all changes |
| KERNEL (2) | Kernel Team | Approve all kernel changes |
| SERVICES (3) | Services Team | Approve all service changes |
| RUNTIMES (4) | Per-runtime teams | Approve changes in their runtime |
| SDK (5) | SDK Team | Approve all public API changes |
| PRODUCTS (6) | Product Teams | Autonomous within their product dir |

### Runtime Owners

| Runtime | Owner Team | Responsibilities |
|---------|-----------|-----------------|
| `event_runtime` | Kernel Team | Event bus, pub/sub |
| `resource_runtime` | Platform Team | Quota, budget |
| `provider_runtime` | SDK Team | Provider adapters |
| `knowledge_runtime` | Knowledge Team | Knowledge graph, search |
| `ai_runtime` | AI Runtime Team | All AI resource intelligence |
| `workflow_runtime` | Workflow Team | Workflow engine (planned) |
| `orchestration_runtime` | Orchestration Team | Multi-agent (planned) |

---

## Change Types and Approval Requirements

### Type A — Architecture Change
Examples: adding a new runtime, changing layer boundaries, modifying dependency rules

| Requirement | Detail |
|-------------|--------|
| Owner approval | Kernel Team + Platform Team |
| Architecture review | Required |
| Architecture document update | Required (new document or update existing) |
| Migration impact analysis | Required |
| CI gates | All verification checks must pass |

### Type B — Public API Change (SDK or HTTP API)
Examples: adding/removing SDK function, changing API response schema

| Requirement | Detail |
|-------------|--------|
| Owner approval | SDK Team |
| Breaking change | New major version required |
| Deprecation notice | Minimum 2 releases before removal |
| Backwards compatibility | Required for minor changes |

### Type C — Runtime Module Change
Examples: new module in a runtime, changing a module's capabilities

| Requirement | Detail |
|-------------|--------|
| Owner approval | Runtime owner team |
| Module catalog update | `module.yaml` updated |
| Capability registry update | `capability.yaml` updated |
| Test coverage | ≥ 90% for STABLE modules |
| Manifest regeneration | `platform-verify manifest --generate` |

### Type D — Service Change
Examples: new service implementation, changing service interface

| Requirement | Detail |
|-------------|--------|
| Owner approval | Services Team |
| Interface compatibility | Existing implementations must still work |
| DI registration | Updated in kernel bootstrap |

### Type E — Internal Change
Examples: refactoring within a single module, test additions, documentation

| Requirement | Detail |
|-------------|--------|
| Owner approval | Module owner (usually just PR review) |
| Tests | Must not decrease coverage |
| No API changes | Any API change escalates to Type B/C |

---

## Dependency Rule Exception Process

If a team needs an exception to DR-001 through DR-010 or LR-001 through LR-011:

1. Create an architecture decision record (see 34-ARCHITECTURE-DECISIONS.md)
2. Get written approval from both layer owners
3. Document the exception with a time limit (exceptions are temporary)
4. Add the exception to `10-DEPENDENCY-RULES.md` exception table
5. Set a review date ≤ 6 months from approval

---

## Architecture Decision Authority

| Decision Type | Authority |
|---------------|-----------|
| New runtime | Platform Team + Kernel Team consensus |
| New service | Services Team approval |
| Breaking SDK change | SDK Team + product team notification (2+ weeks) |
| Dependency rule change | Kernel Team + Platform Team consensus |
| Architecture document update | Document author + section owner |
| Architecture Freeze | Platform Team unanimous |

---

## Review Requirements

### PR Review Requirements

| Change scope | Minimum reviewers | Special requirements |
|-------------|-------------------|----------------------|
| Kernel layer | 2 (Kernel Team) | Architecture review if new API |
| SDK layer | 2 (SDK Team) | Breaking change analysis |
| Runtime module | 1 (Runtime Team) | Coverage check |
| Service | 1 (Services Team) | Interface compatibility |
| Architecture doc | 1 (Document owner) | KQ checks pass |
| Test-only | 1 (any) | Coverage must not decrease |
| Documentation | 1 (any) | Spell check |

---

## Stability Levels and Contracts

| Stability | Contract |
|-----------|---------|
| STABLE | No breaking changes. Deprecated with 2-release notice. |
| BETA | May have breaking changes with 1-release notice. |
| EXPERIMENTAL | No stability guarantee. May be removed without notice. |
| PLANNED | Not yet implemented. No contract. |

---

## Governance Review Cadence

| Review | Frequency | Participants |
|--------|-----------|-------------|
| Architecture review | Per major change | All team leads |
| Dependency rule audit | Quarterly | Kernel Team |
| Ownership table update | On team change | Platform Team |
| Performance budget review | Per release | All teams |
| Knowledge quality audit | Monthly | Knowledge Team |
