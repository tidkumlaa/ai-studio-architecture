# AI Studio โ€” Platform Refactoring Blueprint

**Document ID:** AI-STUDIO-PLATFORM-REFACTORING-BLUEPRINT  
**Version:** 1.0.0  
**Date:** 2026-06-28  
**Status:** DRAFT โ€” ARCHITECTURE REVIEW  
**Classification:** INTERNAL โ€” ARCHITECTURE GOVERNANCE  
**Authors:** Chief Software Architect, Enterprise Platform Architect, Principal Engineer, CTO  
**Predecessor:** AI-STUDIO-ENTERPRISE-ARCHITECTURE-REVIEW-V2.md (score: 51/100)

---

## Architecture Freeze Notice

> This document defines the platform architecture freeze for AI Studio 2.x, 3.x, and all future products.
>
> No production code shall be written until this blueprint is reviewed and approved.
>
> All implementation phases โ€” Security, Workflow, Prompt, AI Resource, Persistence, Messaging, Observability โ€” are blocked until this freeze is approved.
>
> Approved by: _________________________ Date: _____________

---

## Table of Contents

1. Executive Summary
2. Platform Vision
3. Strategic Context
4. Current Module Classification
5. Target Repository Layout
6. Platform Layer Design
7. Product Layer Design
8. Shared SDK Design
9. AI Resource Operating System (AI ROS)
10. Provider Interface Design
11. Workspace Architecture
12. Dependency Rules
13. Architecture Diagrams
14. Architecture Decision Records (ADR-001 through ADR-010)
15. Migration Strategy
16. Dependency Graph Recommendations
17. Risk Assessment
18. Implementation Phases
19. Architecture Freeze Checklist
20. Success Criteria

---

---

# 1. Executive Summary

## 1.1 The Problem

AI Studio 2.0 was built product-first. The result is a platform where:

- **ProductFactory** directly instantiates AIExecutionEngine, DecisionEngine, PromptRuntime, and WorkflowRuntime inside its own `__init__`
- **Prompt rendering** is implemented in four places with no canonical path
- **Central Brain** is owned by the `ai-software-factory` product, not by a shared platform layer
- **Every future product** (Content Factory, Mythic Realms, Vocal Coach, ERP Generator) would need to re-implement or copy workflow dispatch, AI routing, prompt governance, and knowledge management

This is not a flaw in the engineers who built it. It is the natural consequence of building one product before the platform that should underpin all products.

## 1.2 The Solution

This blueprint redesigns AI Studio as a **two-layer architecture**:

```
โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
โ”                    PRODUCTS LAYER                           โ”
โ”  ai-software-factory  content-factory  mythic-realms  ...  โ”
โ”  (business logic only โ€” no platform internals)             โ”
โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”ฌโ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
                            โ” consumes via SDK
โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ–ผโ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
โ”                    PLATFORM LAYER                           โ”
โ”  AI ROS  ยท  Workflow Runtime  ยท  Prompt OS  ยท  Brain       โ”
โ”  Security  ยท  Messaging  ยท  Knowledge  ยท  Memory           โ”
โ”  (reusable โ€” no product logic)                             โ”
โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
```

## 1.3 Key Decisions (Summary)

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Repository model | Monorepo with clear layer boundaries | Shared infrastructure, single CI, cross-cutting changes |
| AI abstraction | AI Resource Operating System (AI ROS) | Single broker between products and all AI providers |
| Provider model | Abstract interface + registry | New providers require zero workflow changes |
| Prompt platform | Prompt OS owns all rendering | 4 current implementations collapse to 1 |
| Brain ownership | Platform (not product) | Every product benefits from compounding intelligence |
| Workflow ownership | Platform | Products describe workflow; platform executes it |
| SDK model | Per-domain SDKs (Brain, Workflow, Prompt, AI) | Thin, typed, versioned interfaces |
| Security | Platform-level middleware | Products cannot bypass auth or rate limiting |
| Product boundaries | Business logic only | Products have no direct AI provider access |

## 1.4 Effort Estimate

| Phase | Effort | Scope |
|-------|--------|-------|
| Phase 0A: Architecture Freeze | 1 week | This document + ADRs |
| Phase 0B: Repository Restructure | 2 weeks | Create new layout, move without changing code |
| Phase 0C: SDK Extraction | 4 weeks | Extract interfaces; wrap existing implementations |
| Phase A: Security | 2 weeks | Implement Phase A on new structure |
| Phase B: Workflow Dispatch | 6 weeks | ProductFactory โ’ real WorkflowRuntime |
| Phase C: Prompt Consolidation | 3 weeks | 4 renderers โ’ Prompt OS |
| Phase D: AI ROS | 6 weeks | Full AI Resource Operating System |
| Phase E: Persistence | 3 weeks | PostgreSQL, pooling, pagination |
| Phase F: Messaging | 4 weeks | Full NATS event coverage |
| **Total to Enterprise GA** | **~31 weeks** | |

---

---

# 2. Platform Vision

## 2.1 Five-Year Target

AI Studio evolves from a single-product orchestrator into an **AI-native platform** that:

1. **Any AI product can be built on top of it** โ€” Content Factory, Mythic Realms, Vocal Coach, ERP Generator, and products not yet conceived all run on the same platform layer.

2. **Intelligence compounds across all products** โ€” The Central Brain learns from every execution across all products. A better prompt discovered in Content Factory improves Mythic Realms automatically.

3. **AI providers are interchangeable** โ€” A new AI provider can be added in a single afternoon. Products never know which provider runs underneath.

4. **Developers build products, not infrastructure** โ€” A new product team writes business logic. They call `workflow.execute(plan)`, `brain.recommend(context)`, `prompt.render(template, vars)`. They never touch providers, scheduling, or security.

5. **The platform enforces correctness** โ€” Security, rate limiting, cost budgets, prompt validation, and audit logging happen at the platform layer. Products cannot accidentally bypass them.

## 2.2 Guiding Principles

| Principle | Statement |
|-----------|-----------|
| Platform owns infrastructure | Workflow, AI routing, Prompt OS, Brain, Security โ€” platform, not products |
| Products own business logic | Domain models, business rules, domain events โ€” products, not platform |
| Interfaces define contracts | Products talk to platform through typed SDKs, not internal imports |
| Intelligence is shared | Central Brain is platform-wide, not per-product |
| Providers are pluggable | New AI provider = new adapter class, no workflow changes |
| Security is non-bypassable | Auth, RBAC, rate limiting applied before any product code runs |
| Everything is observable | Every platform operation produces metrics, traces, and structured logs |
| Backward compatibility | Existing product code migrates incrementally; no big-bang rewrites |

## 2.3 Platform Anatomy

```
โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
โ”                     AI STUDIO PLATFORM                                   โ”
โ”                                                                          โ”
โ”  โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ” โ”
โ”  โ”                 AI RESOURCE OPERATING SYSTEM (AI ROS)               โ” โ”
โ”  โ”                                                                     โ” โ”
โ”  โ”  Providers โ”€โ”€โ–บ ProviderRegistry โ”€โ”€โ–บ LoadBalancer โ”€โ”€โ–บ SchedulerQ    โ” โ”
โ”  โ”                     โ”                   โ”                           โ” โ”
โ”  โ”              CapabilityMatrix    CircuitBreaker                     โ” โ”
โ”  โ”                     โ”                   โ”                           โ” โ”
โ”  โ”              CostTracker โ—โ”€โ”€โ”€โ”€ BudgetManager โ—โ”€โ”€โ”€โ”€ QuotaManager    โ” โ”
โ”  โ”                     โ”                                               โ” โ”
โ”  โ”              ConversationManager โ”€โ”€โ–บ StreamingManager               โ” โ”
โ”  โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”ฌโ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ” โ”
โ”                                 โ”                                        โ”
โ”  โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”  โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ–ผโ”€โ”€โ”€โ”€โ”€โ”€โ”  โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”  โ”
โ”  โ”  Workflow    โ”  โ”   Prompt OS       โ”  โ”   Central Brain /        โ”  โ”
โ”  โ”  Runtime     โ”  โ”  (render engine)  โ”  โ”   Knowledge Graph        โ”  โ”
โ”  โ”              โ”  โ”                   โ”  โ”                          โ”  โ”
โ”  โ”  DAG Exec    โ”  โ”  Template Engine  โ”  โ”  SimilarityEngine        โ”  โ”
โ”  โ”  Scheduler   โ”  โ”  Governance FSM   โ”  โ”  PatternDiscovery        โ”  โ”
โ”  โ”  Checkpoint  โ”  โ”  HMAC Security    โ”  โ”  RecommendationEngine    โ”  โ”
โ”  โ”  Agent Pool  โ”  โ”  Registry         โ”  โ”  ExperienceGraph         โ”  โ”
โ”  โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”  โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”  โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”  โ”
โ”                                                                          โ”
โ”  โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”    โ”
โ”  โ”  Security Platform  โ”  Messaging (NATS)  โ”  Observability        โ”    โ”
โ”  โ”  Auth / RBAC / RL   โ”  Event Bus         โ”  OTel / Prometheus    โ”    โ”
โ”  โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”    โ”
โ”                                                                          โ”
โ”  โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”    โ”
โ”  โ”  Persistence Platform                                            โ”    โ”
โ”  โ”  PostgreSQL / Connection Pool / Migrations / Repository Layer    โ”    โ”
โ”  โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”    โ”
โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
          โ–ฒ                    โ–ฒ                    โ–ฒ
          โ” SDK                โ” SDK                โ” SDK
โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”ดโ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”  โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”ดโ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”  โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”ดโ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
โ” ai-software-     โ”  โ” content-factory โ”  โ” mythic-realms    โ”
โ” factory          โ”  โ” (product)       โ”  โ” (product)        โ”
โ” (product)        โ”  โ”                 โ”  โ”                  โ”
โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”  โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”  โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
```

---

---

# 3. Strategic Context

## 3.1 Current Product Portfolio

| Product | Repository | Status | Platform Coupling |
|---------|-----------|--------|-----------------|
| AI Software Factory | source/ai-software-factory | GA certified (84/100) | OWNS platform code |
| Content Factory | source/content-factory | Separate service (Java + Python) | REST-only integration |
| Mythic Realms | source/mythic-realms | Phase 1 complete | Zero platform coupling |
| AI Vocal Coach | Not yet created | Planned | Will need workflow, AI, prompts |
| ERP Generator | Not yet created | Planned | Will need workflow, AI, prompts |
| Marketplace | Not yet created | Planned | Will need platform services |

## 3.2 The Coupling Problem

Every planned future product needs the same core services:

- **Workflow orchestration** โ€” to manage multi-step AI processes
- **AI provider access** โ€” to call Claude, GPT-4, Gemini
- **Prompt management** โ€” to version, govern, and render prompts
- **Knowledge integration** โ€” to learn from past executions
- **Security** โ€” to enforce auth, RBAC, and rate limits
- **Cost tracking** โ€” to prevent runaway AI spend

If each product reimplements these, AI Studio becomes a collection of separate silos rather than a platform. The Content Factory (Java) currently integrates via REST โ€” it has no access to the Brain, Prompt OS, or Workflow Runtime at all.

## 3.3 Opportunity

The `factory/providers/` directory already contains the seed of a provider abstraction:
- `factory/providers/base.py:ProviderResult` โ€” clean result type
- `factory/providers/anthropic_provider.py` โ€” Anthropic implementation
- `factory/providers/openai_provider.py` โ€” OpenAI implementation
- `factory/providers/gemini_provider.py` โ€” Google implementation
- `factory/providers/ollama_provider.py` โ€” Ollama implementation
- `factory/providers/claude_code_provider.py` โ€” Claude Code CLI provider
- `factory/providers/router.py` โ€” routing logic (precursor to AI ROS)

This is the correct architectural instinct. The blueprint formalizes and extends it into a full AI Resource Operating System.

---

---

# 4. Current Module Classification

Every module in the current `ai-software-factory` codebase is classified below.

## 4.1 Classification Legend

| Class | Definition |
|-------|-----------|
| **PLATFORM** | Core infrastructure reusable by all products โ€” must move to platform layer |
| **SHARED_SVC** | Service shared across products but with product-specific configuration |
| **PRODUCT** | Business logic specific to ai-software-factory โ€” stays in product |
| **DESKTOP** | UI component โ€” belongs in desktop layer |
| **INFRA** | Infrastructure glue (DB engine, migrations, config) |
| **SDK** | Interface/contract definition โ€” becomes part of shared SDK |
| **UTILITY** | Pure utility with no business logic |
| **LEGACY** | Pre-2.0 component; review for deprecation |

## 4.2 Engine Modules

| Module | Current Path | Classification | Target Location |
|--------|-------------|---------------|----------------|
| `ai_execution.py` | engine/ | PLATFORM | platform/ai-ros/core |
| `ai_router.py` | engine/ | PLATFORM | platform/ai-ros/routing |
| `model_registry.py` | engine/ | PLATFORM | platform/ai-ros/registry |
| `tool_runtime.py` | engine/ | PLATFORM | platform/ai-ros/tools |
| `conversation_engine.py` | engine/ | PLATFORM | platform/ai-ros/conversation |
| `context_engine.py` | engine/ | PLATFORM | platform/ai-ros/context |
| `prompt_runtime.py` | engine/ | PLATFORM | platform/prompt-os/ (merge with prompt_os) |
| `prompt_intelligence.py` | engine/ | PLATFORM | platform/prompt-os/ |
| `prompt_os/` (12 files) | engine/ | PLATFORM | platform/prompt-os/ |
| `workflow_runtime.py` | engine/ | PLATFORM | platform/workflow-runtime/ |
| `workflow_engine.py` | engine/ | PLATFORM | platform/workflow-runtime/ |
| `workflow_supervisor.py` | engine/ | PLATFORM | platform/workflow-runtime/ |
| `dispatcher.py` | engine/ | PLATFORM | platform/workflow-runtime/ |
| `review_router.py` | engine/ | PLATFORM | platform/workflow-runtime/ |
| `merge_manager.py` | engine/ | PLATFORM | platform/workflow-runtime/ |
| `central_brain.py` | engine/ | PLATFORM | platform/knowledge/brain |
| `decision_engine.py` | engine/ | PLATFORM | platform/knowledge/decision |
| `memory_os.py` | engine/ | PLATFORM | platform/knowledge/memory |
| `experience_recorder.py` | engine/ | PLATFORM | platform/knowledge/experience |
| `event_bus.py` | engine/ | PLATFORM | platform/messaging/ |
| `nats_client.py` | engine/ | PLATFORM | platform/messaging/ |
| `event_schemas.py` | engine/ | SDK | sdk/messaging/ |
| `security.py` | engine/ | PLATFORM | platform/security/ |
| `approval_service.py` | engine/ | PLATFORM | platform/workflow-runtime/gates |
| `sla_monitor.py` | engine/ | PLATFORM | platform/workflow-runtime/monitoring |
| `blocker_monitor.py` | engine/ | PLATFORM | platform/workflow-runtime/monitoring |
| `root_cause_engine.py` | engine/ | PLATFORM | platform/workflow-runtime/monitoring |
| `cost_engine.py` | engine/ | PLATFORM | platform/ai-ros/cost |
| `brain.py` | engine/ | PLATFORM | platform/knowledge/brain (merge with central_brain) |
| `outcome_collector.py` | engine/ | PLATFORM | platform/knowledge/experience |
| `failure_analyzer.py` | engine/ | PLATFORM | platform/knowledge/experience |
| `learning_agent.py` | engine/ | PLATFORM | platform/knowledge/experience |
| `docker_service.py` | engine/ | SHARED_SVC | platform/ai-ros/tools/docker |
| `self_improvement.py` | engine/ | SHARED_SVC | platform/knowledge/improvement |
| `product_intake.py` | engine/ | PRODUCT | products/ai-software-factory/intake |
| `business_analyst.py` | engine/ | PRODUCT | products/ai-software-factory/agents |
| `architect_agent.py` | engine/ | PRODUCT | products/ai-software-factory/agents |
| `planner_engine.py` | engine/ | PRODUCT | products/ai-software-factory/planning |
| `artifact_generator.py` | engine/ | PRODUCT | products/ai-software-factory/generation |
| `product_factory.py` | engine/ | PRODUCT | products/ai-software-factory/factory |
| `org_engine.py` | engine/ | PRODUCT | products/ai-software-factory/org |
| `employee_engine.py` | engine/ | PRODUCT | products/ai-software-factory/employees |
| `dependency_resolver.py` | engine/ | PRODUCT | products/ai-software-factory/planning |
| `infra_patterns.py` | engine/ | PRODUCT | products/ai-software-factory/patterns |

## 4.3 Factory Modules

| Module | Current Path | Classification | Target Location |
|--------|-------------|---------------|----------------|
| `factory/providers/` (7 files) | factory/ | PLATFORM | platform/ai-ros/providers/ |
| `factory/storage/` (10 files) | factory/ | PLATFORM | platform/storage/ |
| `factory/assets/` (6 files) | factory/ | PLATFORM | platform/assets/ |
| `factory/memory/` | factory/ | PLATFORM | platform/knowledge/memory |
| `factory/metrics/` (6 files) | factory/ | PLATFORM | platform/observability/metrics |
| `factory/logging/` (4 files) | factory/ | PLATFORM | platform/observability/logging |
| `factory/tracing/` (3 files) | factory/ | PLATFORM | platform/observability/tracing |
| `factory/plugins/` (8 files) | factory/ | PRODUCT | products/ai-software-factory/plugins |
| `factory/sdk/` (3 files) | factory/ | SDK | sdk/plugin/ |
| `factory/hooks/` (2 files) | factory/ | SDK | sdk/hooks/ |
| `factory/versioning.py` | factory/ | UTILITY | sdk/common/ |
| `factory/cli/` | factory/ | UTILITY | sdk/cli/ |

## 4.4 Database and Infrastructure Modules

| Module | Current Path | Classification | Target Location |
|--------|-------------|---------------|----------------|
| `db/engine.py` | db/ | INFRA | platform/persistence/ |
| `db/models.py` | db/ | INFRA | platform/persistence/models |
| `db/init_db.py` | db/ | INFRA | platform/persistence/ |
| `db/migrator.py` | db/ | INFRA | platform/persistence/ |
| `db/pg_config.py` | db/ | INFRA | platform/persistence/ |
| `config.py` | root | INFRA | platform/config/ + product configs |
| `supervisor.py` | root | PLATFORM | platform/workflow-runtime/supervisor |
| `app.py` | root | PRODUCT | products/ai-software-factory/ |
| `dashboard/` | root | PRODUCT | products/ai-software-factory/dashboard |

## 4.5 API Modules

| Module | Current Path | Classification | Target Location |
|--------|-------------|---------------|----------------|
| `api/auth.py` | api/ | PLATFORM | platform/security/auth |
| `api/security_middleware.py` | api/ | PLATFORM | platform/security/ |
| `api/health_routes.py` | api/ | PLATFORM | platform/observability/ |
| `api/metrics_routes.py` | api/ | PLATFORM | platform/observability/ |
| `api/ws_routes.py` | api/ | PLATFORM | platform/messaging/websocket |
| `api/nats_routes.py` | api/ | PLATFORM | platform/messaging/ |
| `api/workflow_routes.py` | api/ | PRODUCT | products/ai-software-factory/api |
| `api/workflow_runtime_routes.py` | api/ | PRODUCT | products/ai-software-factory/api |
| `api/product_routes.py` | api/ | PRODUCT | products/ai-software-factory/api |
| `api/approval_routes.py` | api/ | PLATFORM | platform/workflow-runtime/api |
| `api/brain_routes.py` | api/ | PLATFORM | platform/knowledge/api |
| `api/central_brain_routes.py` | api/ | PLATFORM | platform/knowledge/api |
| `api/prompt_intelligence_routes.py` | api/ | PLATFORM | platform/prompt-os/api |
| `api/prompt_os_routes.py` | api/ | PLATFORM | platform/prompt-os/api |
| `api/ai_execution_routes.py` | api/ | PLATFORM | platform/ai-ros/api |
| `api/cost_routes.py` | api/ | PLATFORM | platform/ai-ros/api |
| `api/decision_routes.py` | api/ | PLATFORM | platform/knowledge/api |
| `api/memory_routes.py` | api/ | PLATFORM | platform/knowledge/api |
| `api/improvement_routes.py` | api/ | PRODUCT | products/ai-software-factory/api |
| `api/org_routes.py` | api/ | PRODUCT | products/ai-software-factory/api |
| `api/employee_routes.py` | api/ | PRODUCT | products/ai-software-factory/api |
| `api/experience_routes.py` | api/ | PLATFORM | platform/knowledge/api |
| `api/git_routes.py` | api/ | PLATFORM | platform/ai-ros/tools/api |
| `api/docker_routes.py` | api/ | PLATFORM | platform/ai-ros/tools/api |
| `api/agent_metrics_routes.py` | api/ | PLATFORM | platform/workflow-runtime/api |
| `api/security_routes.py` | api/ | PLATFORM | platform/security/api |
| `api/storage_routes.py` | api/ | PLATFORM | platform/storage/api |
| `api/asset_routes.py` | api/ | PLATFORM | platform/assets/api |
| `api/plugin_sdk_routes.py` | api/ | PRODUCT | products/ai-software-factory/api |
| `api/runtime_routes.py` | api/ | PLATFORM | platform/observability/api |
| `api/db_routes.py` | api/ | INFRA | platform/persistence/api |
| `api/proxy_routes.py` | api/ | PRODUCT | products/ai-software-factory/api |
| `api/capabilities_routes.py` | api/ | PLATFORM | platform/observability/api |

## 4.6 Classification Summary

| Classification | Count | Percentage |
|---------------|-------|-----------|
| PLATFORM | 68 | 61% |
| PRODUCT | 21 | 19% |
| SDK | 8 | 7% |
| INFRA | 8 | 7% |
| UTILITY | 4 | 4% |
| SHARED_SVC | 2 | 2% |
| LEGACY | 0 | 0% |

**Key finding:** 61% of the current codebase belongs in the platform layer, not the product layer. The product owns only 19% โ€” its business logic. This confirms the architectural coupling diagnosed in the review.

---

---

# 5. Target Repository Layout

## 5.1 New Monorepo Structure

```
AIStudio/                                    # Monorepo root
โ”
โ”โ”€โ”€ platform/                               # PLATFORM LAYER
โ”   โ”โ”€โ”€ ai-ros/                             # AI Resource Operating System
โ”   โ”   โ”โ”€โ”€ src/
โ”   โ”   โ”   โ”โ”€โ”€ providers/                  # Provider adapters
โ”   โ”   โ”   โ”   โ”โ”€โ”€ base.py                 # ProviderInterface (abstract)
โ”   โ”   โ”   โ”   โ”โ”€โ”€ anthropic_provider.py
โ”   โ”   โ”   โ”   โ”โ”€โ”€ openai_provider.py
โ”   โ”   โ”   โ”   โ”โ”€โ”€ gemini_provider.py
โ”   โ”   โ”   โ”   โ”โ”€โ”€ ollama_provider.py
โ”   โ”   โ”   โ”   โ”โ”€โ”€ claude_code_provider.py
โ”   โ”   โ”   โ”   โ””โ”€โ”€ registry.py
โ”   โ”   โ”   โ”โ”€โ”€ routing/                    # Load balancer, AI router
โ”   โ”   โ”   โ”โ”€โ”€ scheduling/                 # Request queue + scheduler
โ”   โ”   โ”   โ”โ”€โ”€ cost/                       # Cost tracking + budget
โ”   โ”   โ”   โ”โ”€โ”€ conversation/               # Multi-turn sessions
โ”   โ”   โ”   โ”โ”€โ”€ tools/                      # Tool runtime (12 tools)
โ”   โ”   โ”   โ”โ”€โ”€ circuit_breaker.py
โ”   โ”   โ”   โ”โ”€โ”€ retry_manager.py
โ”   โ”   โ”   โ”โ”€โ”€ health_monitor.py
โ”   โ”   โ”   โ””โ”€โ”€ streaming.py
โ”   โ”   โ”โ”€โ”€ tests/
โ”   โ”   โ””โ”€โ”€ pyproject.toml
โ”   โ”
โ”   โ”โ”€โ”€ workflow-runtime/                   # Workflow orchestration
โ”   โ”   โ”โ”€โ”€ src/
โ”   โ”   โ”   โ”โ”€โ”€ runtime/                    # WorkflowRuntime, DAG executor
โ”   โ”   โ”   โ”โ”€โ”€ scheduler/                  # WorkflowScheduler
โ”   โ”   โ”   โ”โ”€โ”€ gates/                      # Human approval gates
โ”   โ”   โ”   โ”โ”€โ”€ monitoring/                 # SLA, blockers, root cause
โ”   โ”   โ”   โ”โ”€โ”€ supervisor/                 # WorkerSupervisor
โ”   โ”   โ”   โ””โ”€โ”€ checkpoint/                 # Crash recovery
โ”   โ”   โ”โ”€โ”€ tests/
โ”   โ”   โ””โ”€โ”€ pyproject.toml
โ”   โ”
โ”   โ”โ”€โ”€ prompt-os/                          # Prompt Platform
โ”   โ”   โ”โ”€โ”€ src/
โ”   โ”   โ”   โ”โ”€โ”€ template/                   # Handlebars engine
โ”   โ”   โ”   โ”โ”€โ”€ governance/                 # FSM + audit
โ”   โ”   โ”   โ”โ”€โ”€ registry/                   # Prompt Intelligence
โ”   โ”   โ”   โ”โ”€โ”€ composition/                # Multi-part compositions
โ”   โ”   โ”   โ”โ”€โ”€ security/                   # HMAC + validator
โ”   โ”   โ”   โ”โ”€โ”€ ab_testing/                 # A/B test framework
โ”   โ”   โ”   โ””โ”€โ”€ brain_bridge.py             # Brain integration
โ”   โ”   โ”โ”€โ”€ tests/
โ”   โ”   โ””โ”€โ”€ pyproject.toml
โ”   โ”
โ”   โ”โ”€โ”€ knowledge/                          # Central Brain + Knowledge
โ”   โ”   โ”โ”€โ”€ src/
โ”   โ”   โ”   โ”โ”€โ”€ brain/                      # CentralBrain, BrainOrchestrator
โ”   โ”   โ”   โ”โ”€โ”€ decision/                   # DecisionEngine
โ”   โ”   โ”   โ”โ”€โ”€ similarity/                 # SimilarityEngine (Jaccard โ’ Qdrant)
โ”   โ”   โ”   โ”โ”€โ”€ patterns/                   # Pattern discovery
โ”   โ”   โ”   โ”โ”€โ”€ experience/                 # ExperienceRecorder (unified)
โ”   โ”   โ”   โ”โ”€โ”€ memory/                     # MemoryOS
โ”   โ”   โ”   โ”โ”€โ”€ improvement/                # SelfImprovement engine
โ”   โ”   โ”   โ””โ”€โ”€ graph/                      # Knowledge graph (SQL โ’ Kuzu migration)
โ”   โ”   โ”โ”€โ”€ tests/
โ”   โ”   โ””โ”€โ”€ pyproject.toml
โ”   โ”
โ”   โ”โ”€โ”€ security/                           # Security Platform
โ”   โ”   โ”โ”€โ”€ src/
โ”   โ”   โ”   โ”โ”€โ”€ auth/                       # API key, JWT, auth middleware
โ”   โ”   โ”   โ”โ”€โ”€ rbac/                       # RBACManager, permissions
โ”   โ”   โ”   โ”โ”€โ”€ rate_limiting/              # RateLimiter
โ”   โ”   โ”   โ”โ”€โ”€ prompt_validator/           # PromptValidator
โ”   โ”   โ”   โ””โ”€โ”€ audit/                      # Audit log
โ”   โ”   โ”โ”€โ”€ tests/
โ”   โ”   โ””โ”€โ”€ pyproject.toml
โ”   โ”
โ”   โ”โ”€โ”€ messaging/                          # Event Platform
โ”   โ”   โ”โ”€โ”€ src/
โ”   โ”   โ”   โ”โ”€โ”€ nats/                       # NATS JetStream client
โ”   โ”   โ”   โ”โ”€โ”€ event_bus/                  # In-process EventBus
โ”   โ”   โ”   โ”โ”€โ”€ schemas/                    # Event contracts (Pydantic)
โ”   โ”   โ”   โ””โ”€โ”€ websocket/                  # WebSocket broadcast
โ”   โ”   โ”โ”€โ”€ tests/
โ”   โ”   โ””โ”€โ”€ pyproject.toml
โ”   โ”
โ”   โ”โ”€โ”€ persistence/                        # Persistence Platform
โ”   โ”   โ”โ”€โ”€ src/
โ”   โ”   โ”   โ”โ”€โ”€ engine/                     # SQLAlchemy engine + pool config
โ”   โ”   โ”   โ”โ”€โ”€ models/                     # All 57 ORM models
โ”   โ”   โ”   โ”โ”€โ”€ migrations/                 # Alembic migrations (0001-NNNN)
โ”   โ”   โ”   โ””โ”€โ”€ repository/                 # Repository pattern (future)
โ”   โ”   โ”โ”€โ”€ tests/
โ”   โ”   โ””โ”€โ”€ pyproject.toml
โ”   โ”
โ”   โ”โ”€โ”€ storage/                            # Storage Platform
โ”   โ”   โ”โ”€โ”€ src/                            # factory/storage/ moved here
โ”   โ”   โ”โ”€โ”€ tests/
โ”   โ”   โ””โ”€โ”€ pyproject.toml
โ”   โ”
โ”   โ””โ”€โ”€ observability/                      # Observability Platform
โ”       โ”โ”€โ”€ src/
โ”       โ”   โ”โ”€โ”€ metrics/                    # Prometheus
โ”       โ”   โ”โ”€โ”€ tracing/                    # OpenTelemetry
โ”       โ”   โ””โ”€โ”€ logging/                    # Structured JSON logging
โ”       โ”โ”€โ”€ tests/
โ”       โ””โ”€โ”€ pyproject.toml
โ”
โ”โ”€โ”€ sdk/                                    # SHARED SDK LAYER
โ”   โ”โ”€โ”€ ai-sdk/                             # AI ROS client SDK
โ”   โ”   โ”โ”€โ”€ src/aistudio/sdk/ai/
โ”   โ”   โ””โ”€โ”€ pyproject.toml
โ”   โ”โ”€โ”€ workflow-sdk/                       # Workflow client SDK
โ”   โ”   โ”โ”€โ”€ src/aistudio/sdk/workflow/
โ”   โ”   โ””โ”€โ”€ pyproject.toml
โ”   โ”โ”€โ”€ prompt-sdk/                         # Prompt OS client SDK
โ”   โ”   โ”โ”€โ”€ src/aistudio/sdk/prompt/
โ”   โ”   โ””โ”€โ”€ pyproject.toml
โ”   โ”โ”€โ”€ brain-sdk/                          # Knowledge/Brain client SDK
โ”   โ”   โ”โ”€โ”€ src/aistation/sdk/brain/
โ”   โ”   โ””โ”€โ”€ pyproject.toml
โ”   โ”โ”€โ”€ knowledge-sdk/                      # Knowledge Graph client SDK
โ”   โ”   โ””โ”€โ”€ pyproject.toml
โ”   โ”โ”€โ”€ security-sdk/                       # Auth + permission check SDK
โ”   โ”   โ””โ”€โ”€ pyproject.toml
โ”   โ”โ”€โ”€ workspace-sdk/                      # Workspace awareness SDK
โ”   โ”   โ””โ”€โ”€ pyproject.toml
โ”   โ””โ”€โ”€ desktop-sdk/                        # Desktop PySide6 base classes
โ”       โ””โ”€โ”€ pyproject.toml
โ”
โ”โ”€โ”€ products/                               # PRODUCT LAYER
โ”   โ”โ”€โ”€ ai-software-factory/                # Current main product (refactored)
โ”   โ”   โ”โ”€โ”€ src/
โ”   โ”   โ”   โ”โ”€โ”€ intake/                     # product_intake.py
โ”   โ”   โ”   โ”โ”€โ”€ planning/                   # planner_engine, dependency_resolver
โ”   โ”   โ”   โ”โ”€โ”€ agents/                     # business_analyst, architect_agent
โ”   โ”   โ”   โ”โ”€โ”€ generation/                 # artifact_generator
โ”   โ”   โ”   โ”โ”€โ”€ factory/                    # product_factory (refactored)
โ”   โ”   โ”   โ”โ”€โ”€ org/                        # org_engine, employee_engine
โ”   โ”   โ”   โ”โ”€โ”€ plugins/                    # factory/plugins/
โ”   โ”   โ”   โ”โ”€โ”€ patterns/                   # infra_patterns
โ”   โ”   โ”   โ”โ”€โ”€ api/                        # Product-specific API routes
โ”   โ”   โ”   โ””โ”€โ”€ app.py                      # Product FastAPI app
โ”   โ”   โ”โ”€โ”€ tests/
โ”   โ”   โ””โ”€โ”€ pyproject.toml
โ”   โ”
โ”   โ”โ”€โ”€ content-factory/                    # Content Factory (existing, Java+Python)
โ”   โ”   โ”โ”€โ”€ java/                           # Existing Java implementation
โ”   โ”   โ”โ”€โ”€ python/                         # Python bridge + SDK consumer
โ”   โ”   โ””โ”€โ”€ pyproject.toml
โ”   โ”
โ”   โ”โ”€โ”€ mythic-realms/                      # Mythic Realms (existing)
โ”   โ”   โ””โ”€โ”€ (existing structure)
โ”   โ”
โ”   โ””โ”€โ”€ _template/                          # New product template
โ”       โ”โ”€โ”€ src/
โ”       โ”   โ”โ”€โ”€ domain/
โ”       โ”   โ”โ”€โ”€ api/
โ”       โ”   โ””โ”€โ”€ app.py
โ”       โ”โ”€โ”€ tests/
โ”       โ””โ”€โ”€ pyproject.toml
โ”
โ”โ”€โ”€ desktop/
โ”   โ””โ”€โ”€ ai-studio-desktop/                  # PySide6 desktop (existing, refactored)
โ”       โ”โ”€โ”€ src/
โ”       โ”   โ”โ”€โ”€ controllers/
โ”       โ”   โ”โ”€โ”€ ui/
โ”       โ”   โ”โ”€โ”€ api/                        # Uses desktop-sdk
โ”       โ”   โ””โ”€โ”€ main.py
โ”       โ”โ”€โ”€ tests/
โ”       โ””โ”€โ”€ pyproject.toml
โ”
โ”โ”€โ”€ marketplace/                            # Future: Plugin/product marketplace
โ”   โ””โ”€โ”€ (Phase 3.5)
โ”
โ”โ”€โ”€ docs/
โ”   โ”โ”€โ”€ architecture/                       # Architecture decision records
โ”   โ”โ”€โ”€ api/                                # Generated OpenAPI specs
โ”   โ”โ”€โ”€ sdk/                                # SDK documentation
โ”   โ””โ”€โ”€ runbooks/                           # Operational runbooks
โ”
โ”โ”€โ”€ tools/                                  # Development tools
โ”   โ”โ”€โ”€ scaffold/                           # New product scaffolding CLI
โ”   โ”โ”€โ”€ migrate/                            # Cross-product migration helpers
โ”   โ””โ”€โ”€ lint/                               # Architecture lint rules
โ”
โ”โ”€โ”€ .github/
โ”   โ””โ”€โ”€ workflows/                          # CI/CD pipelines
โ”
โ”โ”€โ”€ workspace.yaml                          # Workspace definition
โ””โ”€โ”€ pyproject.toml                          # Root workspace pyproject
```

## 5.2 Migration: Current โ’ Target

During the transition, the current `ai-software-factory` repository is preserved intact. The new structure is built alongside it:

```
Phase 0B: Create new directory skeleton (no code moved)
Phase 0C: Extract platform interfaces (platform/* directories created, code copied not moved)
Phase A+: Implementation phases work on new structure
Final:    ai-software-factory becomes products/ai-software-factory
```

There is no single "move day" โ€” the migration is incremental and backward-compatible throughout.

## 5.3 Import Path Strategy

Current imports like:
```python
from engine.central_brain import get_brain_orchestrator
from engine.workflow_runtime import get_workflow_runtime
```

Migrate to:
```python
from aistudio.sdk.brain import get_brain_client
from aistation.sdk.workflow import get_workflow_client
```

SDK clients are thin wrappers โ€” they delegate to the platform services but provide typed interfaces. During transition, both imports coexist via re-export shims in `engine/__init__.py`.

---

---

# 6. Platform Layer Design

## 6.1 Platform Boundaries

Each platform module has hard-coded boundaries:

| Rule | Platform modules | Enforcement |
|------|----------------|-------------|
| No product imports | Platform code cannot import from `products/` | Architecture linter |
| No direct DB in AI ROS | AI ROS uses Repository layer, not raw ORM | Code review |
| No product-specific config | Platform uses only platform-scoped config | Config schema validation |
| All requests audited | Every platform operation emits an audit event | Platform middleware |
| All errors structured | Platform raises typed exceptions, not raw strings | Exception hierarchy |

## 6.2 Platform Startup Contract

Every platform module must implement:

```python
class PlatformModule(ABC):
    @abstractmethod
    async def startup(self, config: PlatformConfig) -> None:
        """Initialize the module. Called during application startup."""

    @abstractmethod
    async def shutdown(self) -> None:
        """Clean up resources. Called during application shutdown."""

    @abstractmethod
    def health(self) -> ModuleHealth:
        """Return current health status for the /health endpoint."""
```

The platform `app.py` (or product `app.py`) calls `startup()` on all registered platform modules in dependency order. The `/health` endpoint aggregates all module health statuses.

## 6.3 Platform Configuration

Each platform module owns its configuration section. The root config is assembled by the platform config loader:

```python
# platform/config/schema.py
class PlatformConfig(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="AISF_", env_file=".env")

    # AI ROS
    ai_ros: AIROSConfig = Field(default_factory=AIROSConfig)

    # Workflow Runtime
    workflow: WorkflowConfig = Field(default_factory=WorkflowConfig)

    # Prompt OS
    prompt_os: PromptOSConfig = Field(default_factory=PromptOSConfig)

    # Knowledge / Brain
    knowledge: KnowledgeConfig = Field(default_factory=KnowledgeConfig)

    # Security
    security: SecurityConfig = Field(default_factory=SecurityConfig)

    # Messaging
    messaging: MessagingConfig = Field(default_factory=MessagingConfig)

    # Persistence
    persistence: PersistenceConfig = Field(default_factory=PersistenceConfig)

    # Observability
    observability: ObservabilityConfig = Field(default_factory=ObservabilityConfig)
```

Products extend this with their own section:
```python
class AISFConfig(PlatformConfig):
    aisf: AISFProductConfig = Field(default_factory=AISFProductConfig)
```

## 6.4 Platform Event Contract

Every platform action emits a typed domain event via the messaging layer:

```python
# sdk/messaging/events.py
class PlatformEvent(BaseModel):
    event_id: UUID = Field(default_factory=uuid4)
    event_type: str          # e.g. "workflow.task.completed"
    source: str              # "platform/workflow-runtime"
    correlation_id: UUID | None = None
    causation_id: UUID | None = None
    timestamp: datetime = Field(default_factory=utcnow)
    payload: dict[str, Any]
    schema_version: str = "1.0"
```

All 17 NATS subjects defined in the existing architecture are implemented as typed event classes.

---

---

# 7. Product Layer Design

## 7.1 What Products Own

Products own exactly three things:

1. **Domain models** โ€” The entities specific to their business domain. For `ai-software-factory`: `Product`, `Blueprint`, `Artifact`. For `mythic-realms`: `Character`, `Realm`, `Quest`.

2. **Business rules** โ€” The logic that makes the product valuable. For `ai-software-factory`: intake interpretation, blueprint generation, phase gate logic. For `mythic-realms`: game balance rules, narrative generation.

3. **Product API** โ€” The HTTP routes and WebSocket endpoints specific to that product.

## 7.2 What Products Must NOT Own

| Forbidden | Correct Alternative |
|-----------|-------------------|
| Direct AI provider calls | `airostudio.sdk.ai.execute(request)` |
| Prompt template strings in code | Register in Prompt OS; call by ID |
| Workflow scheduling logic | `airostudio.sdk.workflow.submit(plan)` |
| Database sessions created directly | Repository passed via dependency injection |
| Authentication logic | Platform SecurityMiddleware handles it |
| Rate limiting | Platform SecurityMiddleware handles it |
| NATS publish calls | `airostudio.sdk.messaging.publish(event)` |
| Brain queries | `airostudio.sdk.brain.recommend(context)` |

## 7.3 Product Interface Contract

Every product exposes a `ProductManifest` that the platform uses for registration:

```python
# sdk/common/manifest.py
class ProductManifest(BaseModel):
    product_id: str                     # "ai-software-factory"
    product_name: str                   # "AI Software Factory"
    version: str                        # "2.0.0"
    platform_min_version: str           # "1.0.0"
    required_platform_modules: list[str] # ["ai-ros", "workflow-runtime", "prompt-os"]
    api_prefix: str                     # "/api/v1/aisf"
    event_namespace: str                # "aisf.*"
    capabilities: list[str]             # ["product_creation", "agent_execution"]
```

The platform validates the manifest at startup and ensures all required platform modules are available before the product can serve requests.

## 7.4 ai-software-factory Product Structure (Post-Refactor)

After refactoring, the `ai-software-factory` product directory contains only business logic:

```
products/ai-software-factory/src/
โ”โ”€โ”€ domain/
โ”   โ”โ”€โ”€ product.py              # Product domain entity
โ”   โ”โ”€โ”€ blueprint.py            # Blueprint domain entity
โ”   โ”โ”€โ”€ phase.py                # Phase definitions
โ”   โ””โ”€โ”€ artifact.py             # Artifact domain entity
โ”โ”€โ”€ intake/
โ”   โ””โ”€โ”€ intake_service.py       # Interpret NL spec โ’ structured ProductSpec
โ”                                # Uses: AI SDK, Prompt SDK
โ”โ”€โ”€ planning/
โ”   โ”โ”€โ”€ blueprint_generator.py  # Generate implementation blueprint
โ”   โ”โ”€โ”€ planner.py              # Create workflow plan from blueprint
โ”   โ””โ”€โ”€ dependency_resolver.py  # Resolve task dependencies
โ”                                # Uses: Brain SDK (for patterns), Workflow SDK
โ”โ”€โ”€ agents/
โ”   โ”โ”€โ”€ business_analyst.py     # Business analysis agent
โ”   โ”โ”€โ”€ architect_agent.py      # Architecture agent
โ”   โ””โ”€โ”€ agent_base.py           # Product-level base (extends SDK WorkflowAgent)
โ”                                # Uses: AI SDK, Prompt SDK, Workflow SDK
โ”โ”€โ”€ generation/
โ”   โ””โ”€โ”€ artifact_generator.py   # Generate code, docs, tests
โ”                                # Uses: AI SDK (not direct providers)
โ”โ”€โ”€ factory/
โ”   โ””โ”€โ”€ product_factory.py      # Orchestrates full pipeline
โ”                                # Uses: Workflow SDK (submits a plan, does NOT simulate)
โ”โ”€โ”€ org/
โ”   โ”โ”€โ”€ org_service.py
โ”   โ””โ”€โ”€ employee_service.py
โ”โ”€โ”€ plugins/
โ”   โ””โ”€โ”€ (factory/plugins/ moved here)
โ”โ”€โ”€ api/
โ”   โ”โ”€โ”€ product_routes.py
โ”   โ”โ”€โ”€ workflow_routes.py      # AISF-specific workflow views
โ”   โ”โ”€โ”€ improvement_routes.py
โ”   โ”โ”€โ”€ org_routes.py
โ”   โ””โ”€โ”€ employee_routes.py
โ””โ”€โ”€ app.py                      # Product FastAPI app (mounts platform routes + product routes)
```

**The critical difference:** `product_factory.py` no longer contains `_simulate_task_execution()`. It calls `workflow_sdk.submit(plan)` and the platform's WorkflowRuntime executes it via real agent dispatch.

---

---

# 8. Shared SDK Design

## 8.1 SDK Philosophy

SDKs are **thin typed wrappers** over platform HTTP APIs or in-process platform calls. They:
- Provide IDE autocompletion and type safety
- Abstract the transport (in-process call vs. HTTP call vs. gRPC)
- Version independently from the platform
- Never contain business logic

## 8.2 AI SDK (`sdk/ai-sdk`)

```python
# airostudio/sdk/ai/__init__.py

class ExecutionRequest(BaseModel):
    prompt: str
    system_prompt: str | None = None
    prompt_id: str | None = None          # Use Prompt OS template
    model_preference: ModelPreference = ModelPreference.BALANCED
    max_tokens: int | None = None
    temperature: float = 0.7
    cost_budget_usd: float = 5.0
    tools: list[str] = []                 # Tool names from ToolRuntime
    stream: bool = False
    context: dict[str, Any] = {}

class ExecutionResult(BaseModel):
    execution_id: UUID
    content: str
    model_used: str
    provider_used: str
    prompt_tokens: int
    completion_tokens: int
    cost_usd: float
    duration_ms: int
    tool_calls: list[ToolCallResult] = []

class AIClient:
    """SDK client for the AI Resource Operating System."""

    def execute(self, request: ExecutionRequest) -> ExecutionResult: ...
    async def stream(self, request: ExecutionRequest) -> AsyncGenerator[str, None]: ...
    def estimate_cost(self, request: ExecutionRequest) -> CostEstimate: ...
    def get_model_capabilities(self) -> list[ModelCapability]: ...
```

## 8.3 Workflow SDK (`sdk/workflow-sdk`)

```python
# airostudio/sdk/workflow/__init__.py

class WorkflowPlan(BaseModel):
    plan_id: UUID = Field(default_factory=uuid4)
    product_id: str
    tasks: list[TaskDefinition]
    dag_edges: list[tuple[str, str]]      # (task_id, depends_on_task_id)
    gates: list[GateDefinition] = []      # Human approval checkpoints
    metadata: dict[str, Any] = {}

class TaskDefinition(BaseModel):
    task_id: str
    task_type: str                        # "code_generation", "test_writing", etc.
    agent_type: str                       # "architect", "developer", "reviewer"
    prompt_id: str | None = None
    inputs: dict[str, Any] = {}
    priority: int = 5
    timeout_seconds: int = 300

class WorkflowClient:
    """SDK client for the Workflow Runtime."""

    def submit(self, plan: WorkflowPlan) -> WorkflowHandle: ...
    def get_status(self, workflow_id: UUID) -> WorkflowStatus: ...
    def pause(self, workflow_id: UUID) -> None: ...
    def resume(self, workflow_id: UUID) -> None: ...
    def cancel(self, workflow_id: UUID) -> None: ...
    def get_checkpoint(self, workflow_id: UUID) -> WorkflowCheckpoint: ...
```

## 8.4 Prompt SDK (`sdk/prompt-sdk`)

```python
# airostudio/sdk/prompt/__init__.py

class PromptClient:
    """SDK client for Prompt OS."""

    def render(self, template_id: str, variables: dict[str, Any]) -> str: ...
    def compose(self, part_ids: list[str], variables: dict[str, Any]) -> str: ...
    def register(self, template: TemplateDefinition) -> str: ...
    def validate(self, content: str) -> ValidationResult: ...
    def get_version(self, template_id: str, version: int | None = None) -> TemplateVersion: ...
```

## 8.5 Brain SDK (`sdk/brain-sdk`)

```python
# airostudio/sdk/brain/__init__.py

class BrainClient:
    """SDK client for Central Brain / Knowledge platform."""

    def recommend(self, context: RecommendationContext) -> list[Recommendation]: ...
    def find_similar(self, project_id: str, threshold: float = 0.30) -> list[SimilarProject]: ...
    def record_experience(self, experience: ExperienceRecord) -> None: ...
    def discover_patterns(self, domain: str) -> list[Pattern]: ...
    def query_graph(self, query: GraphQuery) -> GraphResult: ...
```

## 8.6 Desktop SDK (`sdk/desktop-sdk`)

```python
# airostudio/sdk/desktop/__init__.py

# Re-exports the BaseController, ApiWorker, WorkerSignals patterns
# so the desktop does not import from platform directly

class BaseController(QObject):
    """Base controller with backoff, GC anchor, auto-refresh."""
    ...

class ApiWorker(QRunnable):
    """Thread-safe Qt worker with RuntimeError protection."""
    ...

class PlatformApiClient:
    """Typed HTTP client for all platform service endpoints."""
    ...
```

## 8.7 Security SDK (`sdk/security-sdk`)

```python
# airostudio/sdk/security/__init__.py

# FastAPI dependency factories that products can use for per-route RBAC
def require_permission(permission: Permission) -> Depends: ...

# AuthContext available from request.state after middleware runs
class AuthContext(BaseModel):
    identity: str
    role: Role
    method: Literal["apikey", "jwt", "dev"]
    is_admin: bool
    permissions: frozenset[Permission]
```

---

---

# 9. AI Resource Operating System (AI ROS)

## 9.1 Overview

AI ROS is the central operating system for all AI resource management. Every product that needs AI capability calls AI ROS. No product calls a provider directly.

```
Product calls AIClient.execute(request)
    โ”
    โ–ผ
AI ROS: SchedulingEngine (queues + prioritizes)
    โ”
    โ”โ”€โ–บ BudgetManager.check(request) โ€” abort if budget exceeded
    โ”โ”€โ–บ QuotaManager.check(client_id) โ€” abort if quota exceeded
    โ”
    โ–ผ
CapabilityMatrix.route(request) โ’ candidate providers
    โ”
    โ–ผ
LoadBalancer.select(candidates) โ’ specific provider instance
    โ”
    โ”โ”€โ–บ CircuitBreaker.allow(provider)? โ€” skip degraded providers
    โ”
    โ–ผ
RetryManager.execute(provider, request)
    โ”
    โ”โ”€โ–บ provider.complete(request) โ’ CompletionResponse
    โ”โ”€โ–บ [on failure] โ’ try next provider in fallback chain
    โ”
    โ–ผ
CostTracker.record(response) โ€” accumulate usage
    โ”
    โ–ผ
ConversationManager.append(session_id, message) โ€” maintain context
    โ”
    โ–ผ
StreamingManager.emit(response) โ€” SSE/WebSocket if streaming=True
    โ”
    โ–ผ
Return ExecutionResult to product
```

## 9.2 ProviderRegistry

The ProviderRegistry is the single source of truth for available providers:

```
ProviderRegistry
    โ”
    โ”โ”€ providers: dict[str, ProviderAdapter]
    โ”     "anthropic"    โ’ AnthropicProvider
    โ”     "openai"       โ’ OpenAIProvider
    โ”     "gemini"       โ’ GeminiProvider
    โ”     "ollama"       โ’ OllamaProvider
    โ”     "claude_code"  โ’ ClaudeCodeProvider
    โ”     "openrouter"   โ’ OpenRouterProvider (future)
    โ”     "lm_studio"    โ’ LMStudioProvider (future)
    โ”
    โ”โ”€ health: dict[str, ProviderHealth]
    โ”     Maintained by HealthMonitor (30s polling)
    โ”
    โ””โ”€ capabilities: dict[str, ProviderCapabilities]
          Maintained at startup + updated on health change
```

Adding a new provider:
1. Create `NewProvider(ProviderInterface)` in `platform/ai-ros/providers/`
2. Register in `ProviderRegistry.register("new_provider", NewProvider(config))`
3. Zero changes to any routing, scheduling, cost tracking, or product code

## 9.3 SchedulingEngine

The current system: requests execute immediately in the calling thread.

AI ROS target: 4-tier queue system (from 3.5.1 spec, implemented):

```
PriorityQueue     URGENT โ€” human approval gates, real-time UX responses
    โ“
StandardQueue     NORMAL โ€” regular product execution requests
    โ“
BatchQueue        BATCH  โ€” background tasks, analysis, brain updates
    โ“
DeadLetterQueue   DLQ    โ€” failed requests after MaxRetries exhausted
```

Each queue has:
- Configurable max depth (prevents unbounded growth)
- Per-queue worker threads (PriorityQueue: 2 dedicated threads)
- Timeout per tier (URGENT: 30s, NORMAL: 300s, BATCH: 3600s)

## 9.4 BudgetManager

```
BudgetPolicy
    โ”
    โ”โ”€ global_daily_limit_usd: float       # ORCH_MAX_AI_COST_USD / 365
    โ”โ”€ global_monthly_limit_usd: float
    โ”โ”€ per_product_daily_limit: dict[str, float]
    โ”โ”€ per_execution_max_usd: float        # 5.0 default
    โ”โ”€ alert_threshold_pct: float          # 80% โ’ warning event
    โ””โ”€ hard_stop_threshold_pct: float      # 100% โ’ reject request

BudgetManager.check(product_id, estimated_cost):
    โ’ Check global daily spend
    โ’ Check product daily spend
    โ’ Check per-execution cap
    โ’ If any limit exceeded: raise BudgetExceededError
    โ’ If alert threshold crossed: emit budget.threshold event
```

## 9.5 QuotaManager

QuotaManager enforces rate limits at the AI execution level (separate from HTTP rate limiting in SecurityMiddleware):

```
QuotaPolicy
    โ”
    โ”โ”€ per_product_rpm: int               # AI requests per minute per product
    โ”โ”€ per_product_tph: int               # Tokens per hour per product
    โ””โ”€ per_provider_rpm: int              # Max requests to any single provider
```

## 9.6 CapabilityMatrix

Maps task requirements to provider capabilities:

```
Task: "generate_code" with capability_requirements=["function_calling", "long_context"]
    โ“
CapabilityMatrix.find_providers(requirements)
    โ’ anthropic (claude-sonnet-4-6): function_calling=YES, long_context=YES (200k)
    โ’ openai (gpt-4o): function_calling=YES, long_context=YES (128k)
    โ’ gemini (gemini-1.5-pro): function_calling=YES, long_context=YES (1M)
    โ’ ollama (llama3.2): function_calling=NO  โ’ excluded
    โ’ claude_code: function_calling=YES, long_context=YES
```

## 9.7 CircuitBreaker

Per-provider circuit breaker with 3 states:

```
CLOSED (normal)
    โ”
    โ”โ”€ failure_count >= THRESHOLD โ’ OPEN
    โ”
OPEN (rejecting)
    โ”
    โ”โ”€ after TIMEOUT_SECONDS โ’ HALF_OPEN
    โ”
HALF_OPEN (testing)
    โ”
    โ”โ”€ success โ’ CLOSED
    โ””โ”€ failure โ’ OPEN
```

Configuration: `failure_threshold=5`, `timeout_seconds=60`, `success_threshold=2`

## 9.8 TokenForecaster

Before execution, AI ROS estimates token consumption:

```python
class TokenForecast(BaseModel):
    prompt_tokens_estimated: int
    completion_tokens_estimated: int
    total_tokens_estimated: int
    cost_usd_estimated: float
    confidence: float                # 0.0-1.0
    basis: str                       # "historical_average" | "template_analysis"
```

Historical averages from `ai_executions` table. Template analysis uses prompt length + variable expansion estimate. Products can request a forecast before committing to an execution.

## 9.9 Session and Conversation Management

```
SessionManager
    โ”
    โ”โ”€ create_session(product_id, context) โ’ session_id
    โ”โ”€ get_session(session_id) โ’ Session
    โ”โ”€ append_message(session_id, role, content) โ’ None
    โ”โ”€ get_context_window(session_id, max_tokens) โ’ ContextWindow
    โ””โ”€ close_session(session_id) โ’ None

Session
    โ”
    โ”โ”€ session_id: UUID
    โ”โ”€ product_id: str
    โ”โ”€ model_binding: str | None        # Pinned to specific model
    โ”โ”€ system_prompt: str | None
    โ”โ”€ messages: list[Message]          # Ordered conversation history
    โ”โ”€ total_tokens: int
    โ”โ”€ cost_usd: float
    โ””โ”€ created_at: datetime
```

Sessions are stored in `ai_conversations` table. Context windows are assembled by the ContextEngine (existing module, moved to platform).

---

---

# 10. Provider Interface Design

## 10.1 The Provider Contract

Every AI provider must implement one interface:

```python
# platform/ai-ros/providers/base.py

from abc import ABC, abstractmethod
from typing import AsyncGenerator
from pydantic import BaseModel

class CompletionRequest(BaseModel):
    """Normalized request โ€” provider-agnostic."""
    messages: list[Message]             # Assembled by SessionManager
    model: str                          # Provider-specific model name
    max_tokens: int = 4096
    temperature: float = 0.7
    top_p: float = 1.0
    stop_sequences: list[str] = []
    tools: list[ToolDefinition] = []
    stream: bool = False
    timeout_seconds: int = 300
    metadata: dict[str, str] = {}       # request_id, correlation_id, etc.

class CompletionResponse(BaseModel):
    """Normalized response โ€” provider-agnostic."""
    content: str
    role: str = "assistant"
    stop_reason: str                    # "end_turn", "max_tokens", "stop_sequence", "tool_use"
    usage: TokenUsage
    tool_calls: list[ToolCall] = []
    model_used: str
    provider_latency_ms: int
    raw_response: dict | None = None    # Original provider response for debugging

class ProviderCapabilities(BaseModel):
    provider_id: str
    models: list[ModelInfo]
    supports_streaming: bool
    supports_function_calling: bool
    supports_vision: bool
    max_context_tokens: int
    max_output_tokens: int
    supported_languages: list[str] = ["en"]

class ProviderHealth(BaseModel):
    provider_id: str
    status: Literal["healthy", "degraded", "offline"]
    latency_p50_ms: float
    latency_p99_ms: float
    error_rate_1m: float
    last_checked: datetime
    message: str | None = None

class CostEstimate(BaseModel):
    prompt_tokens: int
    completion_tokens_estimated: int
    cost_usd_estimated: float
    basis: str

class AIProvider(ABC):
    """Abstract base class all AI providers must implement."""

    @property
    @abstractmethod
    def provider_id(self) -> str:
        """Unique identifier for this provider. e.g. 'anthropic'"""

    @abstractmethod
    def complete(self, request: CompletionRequest) -> CompletionResponse:
        """Synchronous completion. Required for all providers."""

    @abstractmethod
    async def complete_async(self, request: CompletionRequest) -> CompletionResponse:
        """Async completion. Required for all providers."""

    @abstractmethod
    async def stream(self, request: CompletionRequest) -> AsyncGenerator[CompletionChunk, None]:
        """Streaming completion. Required for all providers."""

    @abstractmethod
    def health_check(self) -> ProviderHealth:
        """Return current health status. Called by HealthMonitor every 30s."""

    @abstractmethod
    def get_capabilities(self) -> ProviderCapabilities:
        """Return static capabilities. Called once at startup."""

    @abstractmethod
    def estimate_cost(self, request: CompletionRequest) -> CostEstimate:
        """Estimate cost before execution. Should not make API calls."""

    def get_headers(self) -> dict[str, str]:
        """Optional: return HTTP headers for observability correlation."""
        return {}
```

## 10.2 Claude Code as Provider #1

The existing `factory/providers/claude_code_provider.py` implements Claude Code CLI integration. In the new architecture, this becomes `ClaudeCodeProvider(AIProvider)` and is the reference implementation:

```
ClaudeCodeProvider
    โ”โ”€ provider_id = "claude_code"
    โ”โ”€ complete() โ€” calls Claude Code CLI via subprocess
    โ”โ”€ complete_async() โ€” async subprocess wrapper
    โ”โ”€ stream() โ€” streams Claude Code CLI output lines
    โ”โ”€ health_check() โ€” checks Claude Code binary availability
    โ”โ”€ get_capabilities() โ€” returns Claude Code supported models/features
    โ””โ”€ estimate_cost() โ€” uses Anthropic published pricing
```

**Key design decision:** Claude Code CLI is Provider #1 not because it is primary, but because it is the richest integration โ€” it supports tool use, multi-turn conversation, and code execution natively. Other providers are alternatives, not fallbacks.

## 10.3 Adding a New Provider (Zero-Workflow Example)

To add OpenRouter as a new provider:

```python
# platform/ai-ros/providers/openrouter_provider.py
class OpenRouterProvider(AIProvider):
    provider_id = "openrouter"

    def __init__(self, api_key: str, base_url: str = "https://openrouter.ai/api/v1"):
        self._client = httpx.Client(base_url=base_url, headers={"Authorization": f"Bearer {api_key}"})

    def complete(self, request: CompletionRequest) -> CompletionResponse:
        resp = self._client.post("/chat/completions", json=self._to_openai_format(request))
        return self._from_openai_format(resp.json())

    # ... implement all abstract methods
```

Then in provider registry initialization:
```python
if config.openrouter_api_key:
    registry.register("openrouter", OpenRouterProvider(api_key=config.openrouter_api_key))
```

Zero changes to WorkflowRuntime, ProductFactory, SchedulingEngine, CostTracker, or any product code.

---

---

# 11. Workspace Architecture

## 11.1 Workspace Definition

The workspace is the runtime context that understands what resources are available:

```yaml
# workspace.yaml (monorepo root)
workspace:
  id: "ai-studio-main"
  version: "2.0.0"

platform:
  modules:
    - ai-ros
    - workflow-runtime
    - prompt-os
    - knowledge
    - security
    - messaging
    - persistence
    - observability

products:
  - id: ai-software-factory
    path: products/ai-software-factory
    version: "2.0.0"
    enabled: true

  - id: content-factory
    path: products/content-factory
    version: "1.5.5"
    enabled: true
    integration: rest-only         # Java service โ€” REST bridge

  - id: mythic-realms
    path: products/mythic-realms
    version: "0.1.0"
    enabled: true

providers:
  anthropic:
    enabled: true
    key_env: AISF_ANTHROPIC_API_KEY
  openai:
    enabled: false
  gemini:
    enabled: false
  ollama:
    enabled: false
    base_url: http://localhost:11434
  claude_code:
    enabled: true                  # Default provider

knowledge_stores:
  graph:
    type: sql                      # sql | kuzu | neo4j
    connection: ${AISF_DATABASE_URL}
  vector:
    type: none                     # none | qdrant | pgvector
  memory:
    type: sqlite
    path: ${AISF_MEMORY_DB_PATH}

workspaces:
  - id: default
    repositories:
      - path: ./repositories/default
        type: git
```

## 11.2 WorkspaceManager

```python
class WorkspaceManager:
    """Central registry of all platform resources, products, and capabilities."""

    def get_product(self, product_id: str) -> ProductManifest: ...
    def get_provider(self, provider_id: str) -> ProviderCapabilities: ...
    def get_available_capabilities(self) -> list[str]: ...
    def get_knowledge_store(self, store_type: str) -> KnowledgeStoreConfig: ...
    def get_repository(self, workspace_id: str, repo_id: str) -> RepositoryInfo: ...
    def health(self) -> WorkspaceHealth: ...
```

## 11.3 Desktop Workspace Panel

The existing Desktop `WorkspacePanel` (already implemented in ai-studio-desktop) expands to display:
- All registered products (ai-software-factory, content-factory, mythic-realms)
- All platform module health statuses
- All active AI providers with health indicators
- All knowledge stores
- All active repositories

---

---

# 12. Dependency Rules

## 12.1 Layer Dependency Matrix

| From โ’ To | Platform | Product | SDK | Desktop | Infra |
|-----------|---------|---------|-----|---------|-------|
| **Platform** | ALLOWED (within layer) | FORBIDDEN | ALLOWED | FORBIDDEN | ALLOWED |
| **Product** | via SDK only | ALLOWED (within product) | ALLOWED | FORBIDDEN | ALLOWED |
| **SDK** | FORBIDDEN | FORBIDDEN | ALLOWED | FORBIDDEN | ALLOWED |
| **Desktop** | via SDK only | via SDK only | ALLOWED | ALLOWED | ALLOWED |
| **Infra** | FORBIDDEN | FORBIDDEN | FORBIDDEN | FORBIDDEN | ALLOWED |

**Rules in plain language:**
1. Products never import from `platform/` directly โ€” they use `sdk/`
2. SDK modules never import from `platform/` or `products/` โ€” they define interfaces only
3. Platform modules may import from other platform modules (within-layer) but must not form cycles
4. Desktop uses SDK clients (which call platform via HTTP) โ€” not direct platform imports
5. Infrastructure (persistence, config) may be imported by any layer

## 12.2 Circular Dependency Removal

Current circular dependencies identified:

| Cycle | Files Involved | Resolution |
|-------|---------------|-----------|
| Cycle A | central_brain.py โ” decision_engine.py | Extract shared interface to SDK; brain calls decision via interface |
| Cycle B | product_factory.py โ’ ai_execution.py โ’ experience_recorder.py โ’ product_factory.py | Experience recording moved to platform; product_factory receives callback |
| Cycle C | prompt_os/brain_bridge.py โ” central_brain.py | Brain bridge uses Brain SDK (one-direction) |
| Cycle D | workflow_runtime.py โ’ approval_service.py โ’ workflow_runtime.py | Approval service references workflow ID (string), not WorkflowRuntime instance |

## 12.3 Dependency Rules Enforcement

Architecture lint rules (enforced in CI):

```yaml
# tools/lint/arch-rules.yaml
rules:
  - name: no-product-in-platform
    description: Platform code must not import from products/
    pattern: "from products.*import"
    paths: ["platform/**/*.py"]
    action: fail

  - name: no-direct-platform-in-product
    description: Products must not import from platform/ (use SDK)
    pattern: "from platform.*import"
    paths: ["products/**/*.py"]
    action: fail

  - name: no-sdk-implementation
    description: SDK must not contain business logic
    pattern: "def .*:.*\n.*db\.query"
    paths: ["sdk/**/*.py"]
    action: fail

  - name: no-cross-product-import
    description: Products must not import from each other
    pattern: "from products\.(ai-software-factory|content-factory|mythic-realms)"
    paths: ["products/**/*.py"]
    action: fail
```

---

---

# 13. Architecture Diagrams

## 13.1 Current vs. Target: ProductFactory

```
CURRENT (BROKEN)
โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€
ProductFactory.__init__()
    โ”โ”€โ”€ self._ai_engine = AIExecutionEngine(db)        # Direct instantiation
    โ”โ”€โ”€ self._decision_engine = DecisionEngine()        # Direct instantiation
    โ”โ”€โ”€ self._workflow_engine = WorkflowEngine()        # Direct instantiation
    โ”โ”€โ”€ self._blueprint_engine = BlueprintEngine()     # Direct instantiation
    โ”โ”€โ”€ self._experience_recorder = ExperienceRecorder() # Direct instantiation
    โ””โ”€โ”€ self._artifact_engine = ArtifactEngine()       # Direct instantiation

ProductFactory._simulate_task_execution()
    โ”โ”€โ”€ time.sleep(random.uniform(1,3))   # FAKE execution
    โ”โ”€โ”€ ai_result = self._ai_engine.execute(...)
    โ””โ”€โ”€ task.status = TaskStatus.MERGED   # FAKE completion


TARGET (CORRECT)
โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€
ProductFactory.__init__(
    workflow_client: WorkflowClient,    # Injected via DI
    ai_client: AIClient,                # Injected via DI
    prompt_client: PromptClient,        # Injected via DI
    brain_client: BrainClient,          # Injected via DI
)

ProductFactory.create(product_id, spec):
    โ”โ”€โ”€ plan = self._planner.build_plan(spec, brain_client.get_patterns())
    โ”โ”€โ”€ handle = workflow_client.submit(plan)   # REAL dispatch
    โ””โ”€โ”€ return ProductHandle(product_id, handle.workflow_id)

# WorkflowRuntime (platform) dispatches tasks to agents
# Agents call AIClient.execute() via AI ROS
# No simulation. No sleep(). No fake status updates.
```

## 13.2 Request Flow: Product โ’ AI ROS โ’ Provider

```
Product (ai-software-factory)
    โ”
    โ”  ai_client.execute(ExecutionRequest(
    โ”      prompt="Design a user authentication module",
    โ”      model_preference=ModelPreference.CAPABILITY,
    โ”      cost_budget_usd=2.0,
    โ”      tools=["file_write", "shell_exec"]
    โ”  ))
    โ”
    โ–ผ
AI ROS: SchedulingEngine
    โ”  priority=NORMAL; enqueue
    โ–ผ
AI ROS: BudgetManager.check(product_id="aisf", estimated_cost=~$0.05)
    โ”  daily spend: $12.50 / $50.00 โ’ OK
    โ–ผ
AI ROS: QuotaManager.check(client_id="aisf")
    โ”  requests this minute: 8 / 20 โ’ OK
    โ–ผ
AI ROS: CapabilityMatrix.route(requires=["function_calling"])
    โ”  candidates: [anthropic, openai, claude_code]
    โ–ผ
AI ROS: LoadBalancer.select(candidates, strategy=CAPABILITY)
    โ”  scores: anthropic=0.92, openai=0.88, claude_code=0.91
    โ”  selected: anthropic
    โ–ผ
AI ROS: CircuitBreaker.allow("anthropic")
    โ”  state: CLOSED โ’ allowed
    โ–ผ
AI ROS: RetryManager.execute(AnthropicProvider, request)
    โ”
    โ”โ”€โ–บ AnthropicProvider.complete(request) [attempt 1]
    โ”       POST api.anthropic.com/v1/messages
    โ”       โ 200 OK, 1247 tokens, 0.9s
    โ”
    โ–ผ
AI ROS: CostTracker.record(product="aisf", tokens=1247, cost=$0.0037)
    โ”
    โ–ผ
AI ROS: ConversationManager.append(session_id, message)
    โ”
    โ–ผ
ExecutionResult(
    content="Here is the authentication module design...",
    model_used="claude-sonnet-4-6",
    provider_used="anthropic",
    tokens=1247,
    cost_usd=0.0037,
    duration_ms=923
)
    โ”
    โ–ผ
Product receives result โ€” proceeds with business logic
```

## 13.3 Event Flow: Platform Events

```
WorkflowRuntime dispatches task
    โ”
    โ–ผ
WorkflowRuntime publishes: PlatformEvent(
    event_type="workflows.task.started",   # NOTE: plural "workflows"
    source="platform/workflow-runtime",
    payload={"workflow_id": "...", "task_id": "...", "agent_type": "developer"}
)
    โ”
    โ”โ”€โ–บ NATS JetStream WORKFLOWS stream (subject: workflows.task.started)
    โ”       โ””โ”€โ–บ Consumer: workflow_monitor (logs + metrics)
    โ”       โ””โ”€โ–บ Consumer: brain_learner (records experience)
    โ”
    โ””โ”€โ–บ In-process EventBus (for WebSocket real-time delivery to desktop)
            โ””โ”€โ–บ Desktop controllers receive event โ’ update UI
```

## 13.4 Security Flow

```
HTTP Request โ’ FastAPI
    โ”
    โ–ผ
SecurityMiddleware (platform/security)
    โ”
    โ”โ”€ path in PUBLIC_PATHS? โ’ pass through
    โ”
    โ”โ”€ Extract: X-API-Key OR Authorization: Bearer <jwt>
    โ”
    โ”โ”€ Validate credential โ’ AuthContext(identity, role, method)
    โ”   attach to request.state
    โ”
    โ”โ”€ RateLimiter.check(identity, path) โ’ 429 if exceeded
    โ”
    โ–ผ
CORSMiddleware
    โ”
    โ–ผ
Product route handler
    โ”
    โ”โ”€ [optional] require_permission(Permission.WORKFLOW_CREATE)
    โ”     reads request.state.auth_context
    โ”     RBACManager.has_permission(role, permission)
    โ”     403 if denied
    โ”
    โ–ผ
Product business logic (via SDK clients)
```

---



---

---

# 14. Architecture Decision Records

---

## ADR-001 โ€” Platform Separation: AI Studio as a Two-Layer Architecture

**Status:** PROPOSED  
**Date:** 2026-06-28  
**Deciders:** CTO, Chief Architect, Principal Engineer

### Context

AI Studio currently has no formal separation between platform infrastructure and product business logic. All code lives in a single repository (`ai-software-factory`). The Central Brain, Workflow Runtime, Prompt OS, and AI routing logic are all co-located with the product code that builds AI software factories.

The planned product portfolio includes:
- ai-software-factory (current)
- content-factory (separate Java service, partial integration)
- mythic-realms (no platform integration)
- vocal-coach (planned)
- erp-generator (planned)
- marketplace (planned)

Without platform separation, each new product must either:
1. Copy platform code (diverges, accumulates bugs independently)
2. Import from `ai-software-factory` (creates hard coupling to a specific product)
3. Build from scratch (expensive, no shared intelligence)

### Decision

We will organize AI Studio as a **two-layer architecture**:

1. **Platform Layer** โ€” Reusable infrastructure shared by all products. No product-specific logic. Versioned independently.
2. **Product Layer** โ€” Business logic for each product. Consumes platform via typed SDK clients. No direct platform imports.

The Platform Layer contains: AI ROS, Workflow Runtime, Prompt OS, Central Brain, Security, Messaging, Persistence, Storage, Observability.

The Product Layer contains: ai-software-factory, content-factory, mythic-realms, and future products.

Products consume platform services via the Shared SDK Layer โ€” a set of typed client libraries that abstract transport and provide IDE autocompletion.

### Consequences

**Positive:**
- New products (vocal-coach, erp-generator) can be built in days, not months, using SDK
- Platform improvements (better AI routing, better Brain) benefit all products automatically
- Products can be deployed and tested independently
- Security and observability enforced at platform level โ€” products cannot bypass

**Negative:**
- Initial migration effort to extract platform from current ai-software-factory code
- SDK interfaces must be designed carefully โ€” a breaking SDK change affects all products
- Platform and product versioning must be managed independently

**Neutral:**
- Existing code does not need to be rewritten โ€” refactored incrementally into the new structure

### Rejected Alternatives

| Alternative | Reason Rejected |
|-------------|----------------|
| Keep monolithic structure | Prevents new product development without code copying |
| Microservices (one service per module) | Premature at current scale; adds network latency for tight inner loops |
| Per-product platform copies | Intelligence does not compound; bugs fixed in one copy persist in others |

---

## ADR-002 โ€” AI Resource Operating System: Single Broker for All AI Access

**Status:** PROPOSED  
**Date:** 2026-06-28  
**Deciders:** CTO, Chief Architect

### Context

The current codebase has AI access scattered across multiple locations:
- `engine/ai_execution.py` โ€” AIExecutionEngine (some routes use this)
- `engine/ai_router.py` โ€” AIRouter (routing strategy)
- `engine/central_brain.py` โ€” direct provider calls
- `engine/product_factory.py` โ€” direct provider calls  
- `engine/artifact_generator.py` โ€” direct provider calls
- `engine/business_analyst.py` โ€” direct provider calls

The `factory/providers/` directory has a clean provider abstraction (`base.py:ProviderResult`), but it is not used consistently โ€” 60% of AI calls bypass it.

This means: no consistent cost tracking, no consistent circuit breaking, no consistent fallback, no consistent rate limiting at the AI level.

### Decision

We will implement an **AI Resource Operating System (AI ROS)** as the single broker between all AI consumers (products, platform modules) and all AI providers (Anthropic, OpenAI, Gemini, Ollama, Claude Code).

**All AI access must flow through AI ROS.** No module, product, or background job may call a provider directly.

AI ROS responsibilities:
1. Priority-based scheduling (URGENT/STANDARD/BATCH queues)
2. Budget enforcement (per-product daily limits)
3. Quota enforcement (AI requests per minute)
4. Capability-based routing (match task requirements to provider capabilities)
5. Load balancing across providers
6. Circuit breaking (per-provider failure isolation)
7. Retry with exponential backoff
8. Cost tracking (every token counted)
9. Conversation/session management
10. Token forecasting (pre-execution cost estimate)

### Consequences

**Positive:**
- Complete AI spend visibility across all products
- New providers added in one place; zero workflow changes
- Circuit breaker prevents cascade failure when one provider goes down
- Cost budgets enforced before any AI call
- Consistent observability (every AI call logged and traced)

**Negative:**
- AI ROS becomes a critical single point of failure โ€” requires HA deployment in production
- All current direct provider calls must be migrated to AI ROS client
- Adds latency (~1ms scheduling overhead) to every AI call

**Neutral:**
- The existing `factory/providers/` structure is the correct foundation; AI ROS builds on it

### Rejected Alternatives

| Alternative | Reason Rejected |
|-------------|----------------|
| Keep current scatter | No consistent budget, routing, or observability |
| Provider-per-product | Products must manage circuit breaking, budgets, rate limits โ€” expensive to build N times |
| External LLM gateway (LiteLLM, etc.) | Adds external dependency; loses platform-level cost attribution per product |

---

## ADR-003 โ€” Provider Interface: Strict Abstract Contract

**Status:** PROPOSED  
**Date:** 2026-06-28  
**Deciders:** Chief Architect, Principal Engineer

### Context

The existing `factory/providers/base.py` defines a `ProviderResult` dataclass and some provider-specific response parsing, but does not define a strict abstract interface. Each provider implementation varies:
- `anthropic_provider.py` โ€” uses `anthropic` SDK
- `openai_provider.py` โ€” uses `openai` SDK
- `claude_code_provider.py` โ€” subprocess call to Claude Code CLI

There is no contract enforcing what all providers must implement. New provider authors have no guidance. AI ROS cannot safely call providers without a stable interface.

### Decision

We will define a strict `AIProvider` abstract base class with:
- `complete(request: CompletionRequest) -> CompletionResponse` โ€” required, synchronous
- `complete_async(request: CompletionRequest) -> CompletionResponse` โ€” required, async
- `stream(request: CompletionRequest) -> AsyncGenerator[CompletionChunk, None]` โ€” required
- `health_check() -> ProviderHealth` โ€” required, called every 30s
- `get_capabilities() -> ProviderCapabilities` โ€” required, called once at startup
- `estimate_cost(request: CompletionRequest) -> CostEstimate` โ€” required, no API calls
- `get_headers() -> dict[str, str]` โ€” optional, observability correlation headers

All inputs and outputs are normalized Pydantic models โ€” no provider-specific structures leak out.

Claude Code (`ClaudeCodeProvider`) is the **reference implementation** and Provider #1. All other providers follow its pattern. Its implementation serves as the example for all future provider contributors.

### Consequences

**Positive:**
- New provider = implement one class with 6 required methods; zero other changes
- AI ROS can treat all providers identically
- Test mocking is trivial โ€” mock `AIProvider` interface
- Type-checked at import time by Python's ABC mechanism

**Negative:**
- Breaking change in interface signature requires all providers to update
- Provider-specific features (e.g., Anthropic's extended thinking) must be expressed through normalized request fields or provider-specific metadata

**Neutral:**
- Existing 5 providers will need minor refactoring to implement the strict interface

### Rejected Alternatives

| Alternative | Reason Rejected |
|-------------|----------------|
| Duck typing (no ABC) | No contract enforcement; runtime failures when provider is missing a method |
| Protocol (structural typing) | Less IDE support; no enforcement at class definition time |
| gRPC service contract | Overkill for in-process provider calls |

---

## ADR-004 โ€” Product Boundaries: Business Logic Only

**Status:** PROPOSED  
**Date:** 2026-06-28  
**Deciders:** CTO, Chief Architect, Principal Engineer

### Context

The current `product_factory.py` contains:
- `_simulate_task_execution()` โ€” fake AI execution with `time.sleep()`
- Direct instantiation of AIExecutionEngine, DecisionEngine, WorkflowEngine
- `_wait_for_approval()` โ€” hardcoded 300-second timeout
- Business logic (product pipeline orchestration)
- Platform logic (workflow dispatch, AI execution)

This violates the principle that products should own only business logic. It also means the core value proposition of AI Studio โ€” autonomous multi-agent execution โ€” is not actually working.

### Decision

We will enforce strict product boundaries. Products are **allowed** to own:
1. Domain entities (Product, Blueprint, Artifact, Character, Realm, etc.)
2. Business rules (phase gate logic, intake interpretation, domain validation)
3. Product-specific API routes
4. Product-specific plugins and patterns

Products are **explicitly forbidden** from owning:
1. Direct AI provider calls (use AI SDK)
2. Prompt template strings in code (register in Prompt OS; call by ID)
3. Workflow scheduling logic (use Workflow SDK)
4. Database session creation (use repository via DI)
5. Authentication logic (platform SecurityMiddleware)
6. Rate limiting (platform SecurityMiddleware)
7. NATS publish calls (use Messaging SDK)
8. Brain queries (use Brain SDK)
9. Direct health check of platform modules

**Enforcement mechanisms:**
1. Architecture lint rules (CI fails on forbidden imports)
2. ProductManifest registration check at startup
3. Code review requirement: all product PRs reviewed by platform architect

### Consequences

**Positive:**
- `ProductFactory._simulate_task_execution()` is deleted โ€” replaced by real `workflow_sdk.submit(plan)`
- The 300-second auto-approve is deleted โ€” real approval gate with no timeout
- Products can be tested without real AI providers (mock SDK clients)
- Product code is simpler โ€” pure business logic

**Negative:**
- `product_factory.py` must be substantially rewritten (not just refactored)
- All direct AI calls in product code must be migrated to AI SDK

**Neutral:**
- Product teams need SDK documentation to replace what they currently import directly

### Rejected Alternatives

| Alternative | Reason Rejected |
|-------------|----------------|
| Allow products to import specific platform modules | Slippery slope; hard to enforce limits |
| Shared "platform helper" module in product | Duplicates platform; diverges over time |
| Keep current structure, document guidelines | Guidelines without enforcement are not enforced |

---

## ADR-005 โ€” Shared SDK Design: Thin Typed Wrappers

**Status:** PROPOSED  
**Date:** 2026-06-28  
**Deciders:** Chief Architect

### Context

Products need to call platform services. The question is how. Options:
1. Direct in-process function calls (import from platform directly)
2. HTTP REST calls to platform services (treat platform as microservices)
3. Typed SDK clients (thin wrappers over in-process or HTTP calls)

### Decision

We will provide **per-domain typed SDK clients** as the official product-to-platform interface:
- `ai-sdk` โ€” AIClient for AI execution
- `workflow-sdk` โ€” WorkflowClient for workflow dispatch and monitoring
- `prompt-sdk` โ€” PromptClient for Prompt OS access
- `brain-sdk` โ€” BrainClient for knowledge and recommendations
- `security-sdk` โ€” `require_permission()` dependency factory and AuthContext
- `workspace-sdk` โ€” WorkspaceManager access
- `desktop-sdk` โ€” PySide6 base classes and platform HTTP client

SDK design principles:
1. **Thin** โ€” SDK clients delegate to platform; they contain no business logic
2. **Typed** โ€” All inputs and outputs are Pydantic models; IDE autocompletion works
3. **Transport-agnostic** โ€” SDK client works whether platform is in-process or over HTTP
4. **Versioned** โ€” SDKs version independently from platform
5. **Testable** โ€” SDK interfaces are easy to mock in product tests

### Consequences

**Positive:**
- Products can be tested with mock SDK clients โ€” no real platform required
- SDK provides stable contract โ€” platform can refactor internals without breaking products
- IDE autocompletion for all platform capabilities

**Negative:**
- SDK adds one indirection layer โ€” slightly more verbose than direct calls
- SDK versions must be managed (product declares minimum SDK version requirement)

**Neutral:**
- Initial SDK implementation is simple: thin wrappers over existing platform code

---

## ADR-006 โ€” Central Brain Ownership: Platform, Not Product

**Status:** PROPOSED  
**Date:** 2026-06-28  
**Deciders:** CTO, Chief Architect

### Context

The current `engine/central_brain.py` is co-located with `ai-software-factory` product code. It contains the BrainOrchestrator that aggregates intelligence across all projects. However, it is currently only accessible to the ai-software-factory product.

The knowledge that Central Brain accumulates โ€” project similarity, pattern discovery, self-improvement proposals โ€” is only valuable if it compounds across ALL products. Insights from a content-factory build should inform an ai-software-factory build and vice versa.

### Decision

Central Brain is **platform-owned**. It is the cross-product intelligence layer.

**Platform owns:**
- `central_brain.py` โ€” BrainOrchestrator with full recommendation engine
- `decision_engine.py` โ€” model selection and task planning intelligence
- `experience_recorder.py` โ€” unified experience logging (single canonical implementation)
- `memory_os.py` โ€” cross-session memory
- `learning_agent.py` โ€” pattern discovery
- `self_improvement.py` โ€” improvement proposal engine
- `knowledge_graph.py` (new) โ€” unified cross-product knowledge graph

**Products consume via Brain SDK:**
```python
brain = BrainClient()
recommendations = brain.recommend(RecommendationContext(
    product_id="mythic-realms",
    task_type="narrative_generation",
    constraints={"genre": "fantasy", "tone": "epic"},
))
```

**Brain events are platform-wide:**
- `brain.pattern.discovered` โ€” published to all product consumers
- `brain.recommendation.generated` โ€” published to requesting product
- `brain.experience.recorded` โ€” every product's execution feeds the brain

### Consequences

**Positive:**
- Intelligence compounds cross-product โ€” the unique value proposition of AI Studio is realized
- A new product gets brain recommendations from day 1 (learns from other products)
- Single `ExperienceRecorder` eliminates the name collision across 3 current modules

**Negative:**
- Brain becomes a platform dependency โ€” if Brain is unavailable, products degrade gracefully (return empty recommendations, not errors)
- Brain must be designed for multi-tenancy if products should have isolated knowledge spaces

**Rejected Alternatives**

| Alternative | Reason Rejected |
|-------------|----------------|
| Per-product brain | Intelligence is siloed; the key value proposition is lost |
| No shared brain | Products learn nothing from each other |
| External ML platform | Premature complexity; adds external dependency |

---

## ADR-007 โ€” Prompt OS Ownership: Platform Canonical Implementation

**Status:** PROPOSED  
**Date:** 2026-06-28  
**Deciders:** Chief Architect, Principal Engineer

### Context

Prompt rendering currently has 4 separate implementations:
1. `engine/prompt_runtime.py` โ€” `PromptRuntime.render()` (legacy)
2. `engine/prompt_os/template.py` โ€” `TemplateEngine.render()` (Prompt OS)
3. `engine/prompt_os/composition.py` โ€” `CompositionEngine.compose()` (multi-part)
4. `engine/prompt_intelligence.py` โ€” inline f-string rendering in some methods

All 4 are active in production. A change to prompt governance affects only Prompt OS โ€” not the other 3 renderers. A security patch to template injection detection must be applied in 4 places.

### Decision

**Prompt OS is the one canonical prompt implementation.** All other renderers are deprecated.

Migration path:
1. `prompt_runtime.py` โ€” delegates to `TemplateEngine` instead of its own implementation
2. `prompt_intelligence.py` โ€” inline f-strings replaced with Prompt OS `render_by_id()` calls
3. Legacy caller sites in product code โ€” migrated to `PromptClient.render()` SDK calls

**All prompts must be registered in Prompt OS** โ€” no prompt strings in product code. This is enforced by:
1. Architecture lint: detect multi-line f-strings containing `{task}`, `{context}` etc. in product code
2. Code review requirement
3. Product's `PromptClient.render(template_id, variables)` is the only allowed path

### Consequences

**Positive:**
- One renderer to fix, one renderer to secure, one renderer to A/B test
- All prompts versioned and governed automatically
- HMAC signing protects all prompts (not just those using Prompt OS)

**Negative:**
- All product code that has inline prompts must be migrated (effort: ~2-3 weeks for aisf product)
- Prompt registration adds a deploy step (register templates before deploying product)

**Neutral:**
- Prompt OS already implements the richest feature set โ€” no capability is lost

---

## ADR-008 โ€” Repository Structure: Monorepo with Clear Layer Boundaries

**Status:** PROPOSED  
**Date:** 2026-06-28  
**Deciders:** CTO, Chief Architect

### Context

Current state: 4 separate Git repositories:
1. `ai-software-factory` โ€” main product + all platform code
2. `ai-studio-desktop` โ€” PySide6 desktop app
3. `content-factory` โ€” Java + Python content service
4. `mythic-realms` โ€” game platform

Options:
1. Keep 4 repos (polyrepo) โ€” evolve each independently
2. Monorepo โ€” one repo, clear directory structure
3. Hybrid โ€” platform repo + product repos

### Decision

We adopt a **monorepo** under `AIStudio/source/` with the structure defined in Section 5.

All code (platform, products, SDK, desktop) lives in one repository. Clear directory conventions enforce layer separation (`platform/`, `products/`, `sdk/`, `desktop/`).

The existing `source/` directory already contains all 4 repos (per workspace.yaml migration in Phase 2). This is the natural home for the monorepo layout.

### Rationale

| Factor | Monorepo | Polyrepo |
|--------|---------|---------|
| Cross-cutting platform changes | One PR | Coordinated PRs across N repos |
| Product isolation | By directory | By repository boundary |
| CI/CD | Single pipeline, per-directory triggers | N pipelines |
| Dependency management | Single pyproject.toml | Cross-repo version pinning |
| Intelligence compounding | Trivial (same codebase) | Requires SDK versioning coordination |
| Onboarding | One clone | N clones |

At current team size (1-5 developers), monorepo wins on every dimension.

### Consequences

**Positive:**
- Single CI pipeline with path-based triggers
- Platform changes and product consumers in one PR
- No cross-repo version coordination

**Negative:**
- `git clone` includes all code (can be mitigated with sparse checkout)
- CI must be configured to only run relevant tests for changed directories

**Neutral:**
- Existing 4 repos become `products/` subdirectories; git history preserved via `git subtree`

---

## ADR-009 โ€” Dependency Rules: Architecture Linting in CI

**Status:** PROPOSED  
**Date:** 2026-06-28  
**Deciders:** Chief Architect, Principal Engineer

### Context

Layer separation defined in ADR-001 is only valuable if it is enforced. Guidelines without enforcement are not enforced. The history of software development demonstrates that without automated enforcement, forbidden imports accumulate over time.

### Decision

We will implement **architecture linting as a mandatory CI step** that fails the build if dependency rules are violated.

Rules enforced by linter:
1. `platform/*` cannot import from `products/*` โ€” platform must not know about products
2. `products/*` cannot import from `platform/*` โ€” must use `sdk/*` only
3. `sdk/*` cannot import from `platform/*` or `products/*` โ€” SDKs define contracts, not implementations
4. No cross-product imports โ€” `products/aisf` cannot import from `products/content-factory`
5. Product cannot import ORM models directly โ€” must use repository layer

Implementation: custom Python script using `ast.parse` to walk all imports; run as `pre-commit` hook and CI step.

### Consequences

**Positive:**
- Violations are caught before code review, not in production
- New team members cannot accidentally violate layer boundaries
- Architecture stays clean as the codebase grows

**Negative:**
- Initial setup effort for linting tool
- Some legitimate exceptions will need `# arch: allow` comments with justification

**Neutral:**
- Linter can be extended to catch other patterns (e.g., hardcoded provider names, raw SQL in product code)

---

## ADR-010 โ€” Future Platform Evolution: AI Studio 3.0 Target State

**Status:** PROPOSED  
**Date:** 2026-06-28  
**Deciders:** CTO

### Context

The current architecture decisions establish AI Studio 2.x platform foundations. This ADR documents the intended 3.0 target state so that 2.x decisions are made with 3.0 in mind.

### AI Studio 3.0 Target State

**Multi-Tenancy**
Every resource (products, workflows, executions, knowledge, prompts) is scoped by `tenant_id`. A single AI Studio deployment serves multiple organizations with complete data isolation.

**Horizontal Scaling**
The platform supports multiple instances behind a load balancer. Workflow state is shared via NATS JetStream (not in-process). Database is PostgreSQL with connection pooling. AI ROS scheduling is distributed via NATS queue groups.

**Vector Knowledge Graph**
Central Brain migrates from SQL-based Jaccard similarity to Qdrant (or pgvector) for O(log n) nearest-neighbor search. Enables semantic similarity, not just lexical matching.

**Plugin Marketplace**
A registry of community-contributed AI tools, prompt templates, workflow patterns, and product plugins. Products can install marketplace items at runtime.

**Full Observability**
Grafana dashboards for all platform KPIs. Prometheus alert rules for SLA breaches, budget thresholds, circuit breaker trips. SIEM integration for security events.

**Enterprise SSO**
SAML 2.0 and OIDC integration. JWT tokens issued by enterprise identity provider. RBAC roles mapped from IdP groups.

### Design Constraints for 2.x Decisions

All 2.x decisions must not foreclose 3.0 options:
1. Data model includes `tenant_id` (nullable now; required at 3.0)
2. WorkflowRuntime uses pluggable state backend (SQLite now; NATS KV at 3.0)
3. Provider registry is runtime-configurable (not startup-only)
4. Cost tracking per tenant from 2.x baseline
5. All platform APIs use `/api/v1/` prefix (versioned for 3.0 `/api/v2/` compatibility)

---

---

# 15. Migration Strategy

## 15.1 Migration Overview

The migration from the current single-repository monolith to the two-layer platform architecture follows a **strangler fig pattern** โ€” the new structure grows around the old structure until the old structure can be safely removed. No big-bang rewrites. No flag days.

```
Current State              โ’  Platform Refactored      โ’  AI Studio 3.0
โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€
ai-software-factory         platform/ + products/        Multi-tenant platform
  (all code mixed)            (clear boundaries)          (HA, scaled)

engine/*.py                 platform/ai-ros/            AI ROS v2
  (scattered AI calls)        (unified broker)           (distributed scheduler)

prompt_runtime (4 copies)   platform/prompt-os/         Prompt OS 2.0
                             (one implementation)         (vector search)

ProductFactory (simulated)  products/aisf/factory       Real execution
  _simulate_task_execution()  workflow_sdk.submit()      (agent pool)

SQLite                      PostgreSQL                  PostgreSQL HA
  (hardcoded path)            (config-driven, pooled)    (+ pgvector)

1 repo (ai-software-factory) Monorepo (4 products)      Monorepo (N products)
```

## 15.2 Migration Phase Sequence

### Phase 0A: Architecture Freeze (1 week โ€” THIS DOCUMENT)

Deliverables:
- [x] AI-STUDIO-PLATFORM-REFACTORING-BLUEPRINT.md (this document)
- [ ] Blueprint reviewed and approved by CTO
- [ ] ADR-001 through ADR-010 reviewed and approved
- [ ] Architecture freeze announced to all contributors

**Output:** Approved architectural target. No implementation begins until this is signed off.

---

### Phase 0B: Repository Restructure (2 weeks)

**Goal:** Create new directory layout with existing code in correct locations. No logic changes.

Steps:
1. Create monorepo directory skeleton (`platform/`, `products/`, `sdk/`, `desktop/`)
2. Move existing code to new locations using `git mv` (preserves history)
3. Update all import paths using automated refactoring script
4. Verify all tests pass in new structure
5. Update CI/CD to use new paths

**Files moved (not rewritten):**

| From | To | Method |
|------|----|--------|
| `engine/workflow_runtime.py` | `platform/workflow-runtime/src/runtime/` | `git mv` |
| `engine/central_brain.py` | `platform/knowledge/src/brain/` | `git mv` |
| `engine/prompt_os/` (12 files) | `platform/prompt-os/src/` | `git mv` |
| `engine/security.py` | `platform/security/src/` | `git mv` |
| `factory/providers/` | `platform/ai-ros/src/providers/` | `git mv` |
| `factory/storage/` | `platform/storage/src/` | `git mv` |
| `engine/product_factory.py` | `products/ai-software-factory/src/factory/` | `git mv` |
| `engine/business_analyst.py` | `products/ai-software-factory/src/agents/` | `git mv` |
| `api/product_routes.py` | `products/ai-software-factory/src/api/` | `git mv` |

**Test:** `python -m pytest` passes. Line count same. Zero logic changes.

---

### Phase 0C: SDK Extraction (4 weeks)

**Goal:** Define typed SDK interfaces and create thin wrapper clients over existing implementations.

Steps:
1. Define `AIProvider` abstract interface (`platform/ai-ros/src/providers/base.py`)
2. Refactor 5 existing providers to implement `AIProvider`
3. Define `WorkflowClient` interface and thin wrapper
4. Define `PromptClient` interface and thin wrapper
5. Define `BrainClient` interface and thin wrapper
6. Define `SecuritySDK` (`require_permission()` factory)
7. Create `sdk/` directory with per-domain pyprojects
8. Update product code to use SDK imports instead of direct platform imports
9. Architecture linter: create and run first pass (report only, not fail)

**Acceptance criteria:**
- All 5 existing provider files implement `AIProvider` ABC
- All existing platform calls from product code go through SDK clients
- Architecture linter reports 0 violations in `products/` directory

---

### Phase A: Security Foundation (2 weeks)

**Goal:** Implement complete security hardening per the Phase A plan (already designed).

**Prerequisite:** Phase 0C complete.

Key deliverables:
1. `SecurityMiddleware` โ€” single enforcement point
2. JWT + API key dual-path authentication
3. RBAC wired to all 31 routes
4. `hmac.compare_digest` replacement
5. `Lock โ’ RLock` in 6 singletons
6. CORS restricted from `*` to explicit allowlist
7. `PromptValidator` wired in AI execution path
8. `ApiKey` ORM model + migration 0013
9. 60+ security tests

**Test:** All security tests pass. Pre-deployment verification commands (from architecture review Appendix KK) all return expected results.

---

### Phase B: Workflow Dispatch (6 weeks)

**Goal:** Eliminate `_simulate_task_execution()`. Products submit real plans to WorkflowRuntime.

Sub-phases:
1. **B.1:** WorkflowRuntime DAG executor โ€” execute tasks in dependency order
2. **B.2:** Agent pool โ€” real agent spawning and dispatch
3. **B.3:** ProductFactory refactor โ€” remove simulation, add `workflow_sdk.submit(plan)` call
4. **B.4:** Real approval gates โ€” remove 300-second auto-approve
5. **B.5:** WorkflowRuntime NATS events โ€” all 7 streams fully implemented
6. **B.6:** End-to-end test: NL spec โ’ real agent execution โ’ artifact

**Acceptance criteria:**
- `grep -r "_simulate_task_execution" products/` returns 0 results
- `grep -r "elapsed < 300" platform/` returns 0 results
- E2E test: create product via API, observe workflow_id created, task agents dispatched, artifacts generated

---

### Phase C: Prompt Consolidation (3 weeks)

**Goal:** 4 prompt renderers โ’ 1 canonical Prompt OS.

Steps:
1. `prompt_runtime.py` โ€” delegate all rendering to `TemplateEngine`
2. `prompt_intelligence.py` โ€” replace inline f-strings with `PromptOS.render_by_id()`
3. Product code audit โ€” identify all inline prompt strings in product code
4. Register all product prompts in Prompt OS
5. Replace product inline strings with `prompt_sdk.render(template_id, variables)`
6. Deprecate `prompt_runtime.py` (keep for 1 release cycle, then remove)
7. Architecture linter: fail on multi-line f-strings with `{task}`, `{context}` patterns in products

**Acceptance criteria:**
- Architecture linter reports 0 prompt-in-code violations
- Prompt OS render logs show all AI executions using registered templates
- No change in AI output quality (verified by golden-set evaluation)

---

### Phase D: AI Resource Operating System (6 weeks)

**Goal:** Build full AI ROS with scheduling, budget, quota, capability routing, circuit breaking.

Sub-phases:
1. **D.1:** Scheduling queues (URGENT/STANDARD/BATCH/DLQ)
2. **D.2:** BudgetManager โ€” per-product daily limits
3. **D.3:** QuotaManager โ€” AI requests per minute
4. **D.4:** CapabilityMatrix โ€” provider capability index
5. **D.5:** CircuitBreaker โ€” per-provider failure isolation
6. **D.6:** ConversationManager โ€” multi-turn session management
7. **D.7:** TokenForecaster โ€” pre-execution cost estimates
8. **D.8:** All product code migrated to `AIClient.execute()` via SDK
9. **D.9:** Cost dashboard in desktop showing spend by product + provider

**Acceptance criteria:**
- 0 direct provider calls outside `platform/ai-ros/src/providers/`
- BudgetManager blocks execution at daily limit (tested with synthetic limit)
- CircuitBreaker trips after 5 failures, recovers after 60 seconds (unit tested)
- Cost tracking: all AI spend attributed to product + model + provider

---

### Phase E: Persistence Hardening (3 weeks)

**Goal:** Resolve 9 critical database issues from the architecture review.

Steps:
1. Migration 0013 โ€” 9 missing indexes
2. Migration 0014 โ€” `memory_database_url` config setting (remove hardcoded `memory.db`)
3. Connection pool configuration (pool_size=5, max_overflow=10, pool_pre_ping=True)
4. Pagination on all 11 list endpoints (cursor-based, page size max=100)
5. Fix cross-migration ALTER in 0012 (extract to migration 0015)
6. Repository pattern implementation (data access layer)
7. Slow query logging (log queries >100ms)

**Acceptance criteria:**
- `EXPLAIN ANALYZE` on top 5 queries shows index scan, not sequential scan
- All list endpoints return `next_cursor` and respect `limit` parameter
- DB engine created with explicit pool configuration (verified by grep)

---

### Phase F: Messaging Hardening (4 weeks)

**Goal:** All 17 NATS subjects fully implemented and consumed.

Steps:
1. Fix subject naming bug (`workflow.*` โ’ `workflows.*`)
2. Implement all 7 streams with correct subjects (15 remaining unimplemented subjects)
3. NATS consumer implementations for each subscriber group
4. Dead letter queue handling
5. At-least-once delivery verification tests
6. WebSocket broadcast verified for all platform events

**Acceptance criteria:**
- NATS subject audit: all 17 subjects mapped to stream consumers
- NATS publish/consume round-trip test passes for each stream
- Brain learner receives workflow events via NATS

---

### Phase G: Observability Platform (3 weeks)

**Goal:** Complete observability stack: metrics, traces, alerts, dashboards.

Steps:
1. Custom Prometheus metrics: task throughput, AI cost/model, queue depth, circuit breaker state
2. Structured JSON logging (replace plaintext log lines)
3. OTel trace propagation through SDK calls
4. Grafana dashboards: AI cost, workflow throughput, brain activity, security events
5. Prometheus alert rules: budget threshold, circuit breaker, SLA breach, error rate
6. SIEM export (optional: webhook to Slack, PagerDuty)

**Acceptance criteria:**
- Grafana AI cost dashboard shows spend by product/provider/model
- PagerDuty alert fires when error rate >5% (verified in staging)
- All AI executions have correlated trace across AI ROS โ’ provider โ’ product

---

### Future: AI Studio 2.5 (Post-Phase G)

After all 7 phases complete:
- Multi-tenancy (add `tenant_id` to all 57 tables + migration)
- Vector search integration (Qdrant or pgvector) replacing O(nยฒ) Jaccard
- Plugin marketplace
- CI/CD pipeline with automated deployment
- Performance testing baseline

---

### Future: AI Studio 3.0 (Post-2.5)

As defined in ADR-010:
- Horizontal scaling (multiple platform instances)
- Enterprise SSO (SAML 2.0 + OIDC)
- Distributed scheduler via NATS queue groups
- Full HA deployment

---

## 15.3 Migration Rollback Plan

Each phase has a rollback procedure. Because we use the strangler fig pattern (new code alongside old), rollback is always available:

| Phase | Rollback |
|-------|---------|
| 0B (directory restructure) | `git revert` all `git mv` commits โ€” import paths restore |
| 0C (SDK extraction) | Remove SDK imports, restore direct platform imports in products |
| A (security) | Feature flag: `AISF_SECURITY_ENABLED=false` disables SecurityMiddleware |
| B (workflow) | Feature flag: `AISF_WORKFLOW_REAL_DISPATCH=false` re-enables simulation |
| C (prompt) | Feature flag: `AISF_PROMPT_OS_CANONICAL=false` re-enables prompt_runtime |
| D (AI ROS) | Feature flag: `AISF_AI_ROS_ENABLED=false` bypasses scheduling to direct calls |
| E (persistence) | Migrations are forward-only; rollback DB version with downgrade scripts |
| F (messaging) | Feature flag: `NATS_ENABLED=false` already exists |
| G (observability) | Remove Grafana dashboards, alert rules; logging rollback not needed |

---

---

# 16. Dependency Graph Recommendations

## 16.1 Modules to Merge

| Module A | Module B | Recommendation | Rationale |
|---------|---------|---------------|---------|
| `engine/brain.py` | `engine/central_brain.py` | Merge into `platform/knowledge/src/brain/brain_orchestrator.py` | Both represent Central Brain; split was historical accident |
| `engine/prompt_runtime.py` | `engine/prompt_os/template.py` | Merge; `prompt_runtime.py` becomes delegate wrapper | 4 renderers โ’ 1 |
| `engine/experience_recorder.py` (3 copies) | (all three) | Merge into single `platform/knowledge/src/experience/` | Name collision is a design defect |
| `engine/outcome_collector.py` | `engine/experience_recorder.py` | Merge; outcome collection is part of experience | Functionally overlapping |
| `engine/workflow_engine.py` | `engine/workflow_runtime.py` | Merge into single `WorkflowRuntime` class | Artificial split; shared state |

## 16.2 Modules to Split

| Module | Split Into | Rationale |
|--------|-----------|---------|
| `engine/product_factory.py` (2,400+ lines) | `product_factory.py` (orchestrator) + `intake_service.py` + `planner.py` | SRP violation; too many responsibilities |
| `engine/central_brain.py` (if merging brain.py) | `brain_orchestrator.py` + `recommendation_engine.py` + `pattern_discovery.py` | Will become large; logical separation |
| `engine/ai_execution.py` | `execution_service.py` + `context_assembler.py` + `response_handler.py` | Separates request assembly from execution from response processing |

## 16.3 Modules to Move

See Section 4 (Classification Table) โ€” 68 modules classified as PLATFORM need to move from `engine/` and `factory/` into `platform/`.

## 16.4 Modules to Deprecate

| Module | Target | Deprecation Plan |
|--------|--------|-----------------|
| `engine/prompt_runtime.py` | Replaced by `platform/prompt-os/` | Phase C: delegate to TemplateEngine; Phase D: remove |
| `factory/sdk/worker_base.py` | Replaced by `sdk/workflow-sdk/` WorkflowAgent | Phase 0C: create new base; Phase B: migrate all workers |
| `factory/hooks/hook_registry.py` | Replaced by `sdk/plugin-sdk/` | Phase 0C: incorporate into plugin SDK |

## 16.5 Circular Dependency Removal

### Cycle A: central_brain โ” decision_engine

**Current:**
```
engine/central_brain.py imports โ’ engine/decision_engine.py
engine/decision_engine.py imports โ’ engine/central_brain.py (for model registry)
```

**Fix:**
```
platform/knowledge/src/brain/brain_orchestrator.py
    โ’ calls decision via BrainSDK interface (not direct import)

platform/knowledge/src/decision/decision_engine.py
    โ’ calls model_registry via AIROSClient (not central_brain)
```

The `_MODEL_CATALOG` hardcoded in `decision_engine.py` is replaced by a `model_registry_client.get_active_models()` call. This breaks the cycle and also fixes the hardcoded model catalog defect.

### Cycle B: product_factory โ’ experience_recorder โ’ product_factory

**Current:**
```
engine/product_factory.py creates tasks โ’ engine/experience_recorder.py records task completion
engine/experience_recorder.py imports product_factory.py (for Product model access)
```

**Fix:**
```
products/aisf/factory/product_factory.py
    โ’ emits "product.task.completed" event (no import of experience_recorder)

platform/knowledge/src/experience/experience_recorder.py
    โ’ listens to "product.task.completed" event (no import of product_factory)
```

Event-driven decoupling eliminates the cycle entirely.

### Cycle C: prompt_os/brain_bridge โ” central_brain

**Current:**
```
engine/prompt_os/brain_bridge.py imports โ’ engine/central_brain.py
engine/central_brain.py imports โ’ engine/prompt_os/orchestrator.py
```

**Fix:**
```
platform/prompt-os/src/brain_bridge.py
    โ’ calls brain via BrainSDK (one-direction dependency on interface)

platform/knowledge/src/brain/brain_orchestrator.py
    โ’ calls prompt_os via PromptSDK (one-direction dependency on interface)
```

Both depend on interfaces (SDK), not on each other's implementations.

### Cycle D: workflow_runtime โ” approval_service

**Current:**
```
engine/workflow_runtime.py imports โ’ engine/approval_service.py
engine/approval_service.py imports โ’ engine/workflow_runtime.py (to update workflow state)
```

**Fix:**
```
platform/workflow-runtime/src/runtime/workflow_runtime.py
    โ’ creates approval request (passes workflow_id: UUID, not runtime reference)

platform/workflow-runtime/src/gates/approval_service.py
    โ’ receives workflow_id, publishes "workflow.gate.approved" event
    โ’ WorkflowRuntime subscribes to event, updates state
```

Again, event-driven decoupling eliminates the cycle.

---

---

# 17. Risk Assessment

## 17.1 Risk Register

| ID | Risk | Probability | Impact | Mitigation |
|----|------|------------|--------|-----------|
| R-001 | ProductFactory simulation removal breaks dependent tests | HIGH | MEDIUM | Write new integration tests against real WorkflowRuntime before removing simulation |
| R-002 | SDK interface breaks require all products to update simultaneously | MEDIUM | HIGH | Semantic versioning; SDK v1.x is backward-compatible; v2.0 requires migration guide |
| R-003 | Brain consolidation loses cross-module learned patterns | LOW | HIGH | Export brain state before merge; restore after; run parallel for 1 sprint |
| R-004 | Directory restructure breaks import resolution in IDE | HIGH | LOW | Automated import rewrite script; immediate verification in CI |
| R-005 | Phase A security breaks existing integrations that call unprotected routes | HIGH | MEDIUM | Feature flag `AISF_SECURITY_ENABLED=false`; migration guide for API clients |
| R-006 | NATS subject fix breaks existing consumers reading `workflow.*` | MEDIUM | MEDIUM | Dual-publish to both `workflow.*` (legacy) and `workflows.*` (new) for 1 release |
| R-007 | AI ROS scheduling adds latency to interactive UI operations | MEDIUM | LOW | URGENT priority queue bypasses scheduling (5ms max overhead) |
| R-008 | PostgreSQL migration from SQLite loses data | MEDIUM | HIGH | Full SQLite export before migration; import script tested on copy |
| R-009 | Architecture linter generates false positives on legitimate imports | MEDIUM | LOW | `# arch: allow` escape hatch with mandatory review comment |
| R-010 | Phase sequence takes longer than 31 weeks | HIGH | MEDIUM | Each phase is independently deployable; partial wins are acceptable |

## 17.2 Critical Path Analysis

The critical path to AI Studio 3.0:

```
Phase 0A (architecture) โ’ Phase 0B (restructure) โ’ Phase 0C (SDK) โ’ Phase A (security)
                                                                        โ“
                                                              Phase B (workflow) โ CRITICAL
                                                                        โ“
                                                              Phase C (prompt)
                                                                        โ“
                                                              Phase D (AI ROS) โ CRITICAL
                                                                        โ“
                                                    Phase E (persistence) + Phase F (messaging)  [parallel]
                                                                        โ“
                                                              Phase G (observability)
                                                                        โ“
                                                              AI Studio 2.5 (multi-tenancy)
                                                                        โ“
                                                              AI Studio 3.0 (HA + SSO)
```

**Phase B is critical:** Real workflow dispatch is the core value proposition. No other phase delivers more business value.

**Phase D is critical:** AI ROS enables budget enforcement, which is a prerequisite for any multi-tenant or customer deployment.

**Phases E, F, G can be done in parallel** after Phase D โ€” they are independent of each other.

---

---

# 18. Implementation Phases

## 18.1 Phase Readiness Gates

Each phase has a readiness gate that must pass before the next phase begins:

| Phase | Gate Criteria |
|-------|-------------|
| 0A โ’ 0B | Blueprint approved (CTO signature). ADR-001โ€“010 approved. |
| 0B โ’ 0C | All tests pass in new directory structure. 0 import errors. |
| 0C โ’ A | Architecture linter 0 violations. All 5 providers implement AIProvider ABC. |
| A โ’ B | All 60+ security tests pass. Pre-deployment checklist Appendix KK items 2-4 green. |
| B โ’ C | E2E test: NL spec โ’ real agent โ’ artifact passes. 0 simulation code in product. |
| C โ’ D | All prompts in Prompt OS. Architecture linter 0 prompt-in-code violations. |
| D โ’ E | All AI calls via AI ROS. BudgetManager tests pass. CircuitBreaker tests pass. |
| E โ’ F | All 9 DB indexes present. Connection pool configured. All list endpoints paginated. |
| F โ’ G | All 17 NATS subjects implemented. NATS round-trip tests pass. |
| G โ’ 2.5 | Grafana dashboards live. Alert rules tested. OTel traces visible. |

## 18.2 Definition of Done (Per Phase)

A phase is **COMPLETE** when:
1. All code written, reviewed, and merged
2. All tests pass (including new tests written for this phase)
3. No new test failures introduced (regressions prohibited)
4. Architecture linter passes
5. Pre-deployment verification commands return expected results
6. Phase readiness gate criteria met
7. Documentation updated

"Complete" is not declared until all 7 criteria are met.

## 18.3 Team Assignments (Suggested)

| Role | Responsibility |
|------|---------------|
| Platform Architect | Owns platform/* and sdk/* โ€” ensures no product logic leaks in |
| Principal Engineer | Leads Phase B (Workflow) and Phase D (AI ROS) โ€” critical path |
| Security Engineer | Owns Phase A โ€” security hardening |
| Product Engineer | Owns products/ai-software-factory migration โ€” Phase 0C, B, C |
| Data Engineer | Owns Phase E (persistence) and F (messaging) |
| SRE | Owns Phase G (observability) and operational readiness |

---

---

# 19. Architecture Freeze Checklist

This checklist must be completed and verified before any implementation phase begins.

## 19.1 Blueprint Review Checklist

| # | Item | Status | Reviewer |
|---|------|--------|---------|
| 1 | Executive Summary reviewed and understood | โ | CTO |
| 2 | Module classification (Section 4) complete and accurate | โ | Chief Architect |
| 3 | Target repository layout (Section 5) approved | โ | CTO |
| 4 | Platform layer design (Section 6) reviewed | โ | Principal Engineer |
| 5 | Product layer boundaries (Section 7) agreed | โ | Product Team |
| 6 | Shared SDK design (Section 8) reviewed | โ | Principal Engineer |
| 7 | AI ROS design (Section 9) reviewed | โ | Chief Architect |
| 8 | Provider interface (Section 10) approved | โ | Principal Engineer |
| 9 | Workspace architecture (Section 11) reviewed | โ | Desktop Lead |
| 10 | Dependency rules (Section 12) agreed | โ | All Team |
| 11 | ADR-001 through ADR-010 reviewed and accepted | โ | CTO + Chief Architect |
| 12 | Migration phase sequence (Section 15) approved | โ | CTO |
| 13 | Risk register (Section 17) reviewed | โ | CTO |
| 14 | Implementation phase readiness gates (Section 18) agreed | โ | All Team |
| 15 | No production code written during Phase 0 | โ | Chief Architect |

## 19.2 Pre-Implementation Verification

Before any Phase 0B work begins, verify the following about the current codebase:

| # | Verification | Command | Expected Result |
|---|-------------|---------|----------------|
| 1 | All current tests pass | `pytest tests/ -x -q` | 193 pass, 39 fail (known) |
| 2 | No uncommitted changes | `git status` | Clean working tree |
| 3 | Engine module count | `ls engine/*.py \| wc -l` | 44 files |
| 4 | Factory module count | `find factory -name "*.py" \| wc -l` | 70+ files |
| 5 | Route file count | `ls api/*_routes.py \| wc -l` | 31 files |
| 6 | Simulation code present (to be removed later) | `grep -r "_simulate_task_execution" engine/` | Found in product_factory.py |
| 7 | Auth bypass present (to be removed in Phase A) | `grep -n "if not settings.api_key: return" api/auth.py` | Found at line ~15 |

These serve as the **baseline** that Phase 0B must not change.

---

---

# 20. Success Criteria

## 20.1 Phase 0 Success Criteria (This Blueprint)

| Criterion | Measurement | Target |
|-----------|------------|--------|
| All modules classified | Count classified / total modules | 100% |
| Platform modules identified | Count | 68 (61% of codebase) |
| ADRs complete | Count with Status: PROPOSED | 10/10 |
| Migration plan covers all phases | Count phases with readiness gates | 9 phases |
| Dependency cycles documented | Count cycles with resolutions | 4/4 |
| Risk register complete | Count risks with mitigations | 10 |
| Blueprint approved by CTO | Signature on freeze document | YES |

## 20.2 Phase A (Security) Success Criteria

| Criterion | Target |
|-----------|--------|
| Routes with auth | 31/31 (was: 20/31) |
| Routes intentionally open | 3 (health, capabilities, metrics) |
| CORS wildcards | 0 (was: 1) |
| Timing-unsafe comparisons | 0 (was: 1) |
| Security tests | 60+ passing |
| Auth bypass via empty api_key | IMPOSSIBLE (was: trivially bypassable) |

## 20.3 Phase B (Workflow) Success Criteria

| Criterion | Target |
|-----------|--------|
| `_simulate_task_execution` occurrences | 0 (was: 1 in product_factory.py) |
| 300-second auto-approve | Removed (was: hardcoded) |
| WorkflowInstance created per product | YES (was: 0) |
| Agent dispatched per task | YES (was: never) |
| E2E test: NL spec โ’ artifact | PASS (was: not implemented) |
| NATS subjects implemented | 17/17 (was: 2/17) |

## 20.4 Phase D (AI ROS) Success Criteria

| Criterion | Target |
|-----------|--------|
| Direct provider calls outside AI ROS | 0 (was: ~60% of AI calls) |
| AI spend tracked | 100% of executions (was: partial) |
| Budget enforcement | YES (was: none) |
| Circuit breaker per provider | YES (was: none) |
| Cost attributed per product | YES (was: not possible) |
| AI cost dashboard in desktop | YES (was: none) |

## 20.5 Enterprise GA Success Criteria (Post All Phases)

| Domain | Score | Target |
|--------|-------|--------|
| Architecture Quality | 10/15 | 14/15 |
| Security Posture | 2/15 | 13/15 |
| Database Design | 6/10 | 9/10 |
| API Quality | 5/10 | 9/10 |
| Observability | 7/10 | 10/10 |
| Testing Completeness | 5/10 | 9/10 |
| Operational Readiness | 4/10 | 9/10 |
| Code Quality | 7/10 | 9/10 |
| Scalability | 1/5 | 4/5 |
| Documentation | 4/5 | 5/5 |
| **TOTAL** | **51/100** | **91/100** |

**Enterprise GA threshold: 75/100.** Target 91/100 to be enterprise-grade with margin.

---

---

# Appendices

---

## Appendix A โ€” Module Inventory by Classification

### Platform Modules (68 total)

**AI ROS (16 modules)**
- engine/ai_execution.py โ’ platform/ai-ros/src/
- engine/ai_router.py โ’ platform/ai-ros/src/routing/
- engine/model_registry.py โ’ platform/ai-ros/src/registry/
- engine/tool_runtime.py โ’ platform/ai-ros/src/tools/
- engine/conversation_engine.py โ’ platform/ai-ros/src/conversation/
- engine/context_engine.py โ’ platform/ai-ros/src/context/
- engine/cost_engine.py โ’ platform/ai-ros/src/cost/
- engine/docker_service.py โ’ platform/ai-ros/src/tools/docker/
- factory/providers/base.py โ’ platform/ai-ros/src/providers/
- factory/providers/anthropic_provider.py โ’ platform/ai-ros/src/providers/
- factory/providers/openai_provider.py โ’ platform/ai-ros/src/providers/
- factory/providers/gemini_provider.py โ’ platform/ai-ros/src/providers/
- factory/providers/ollama_provider.py โ’ platform/ai-ros/src/providers/
- factory/providers/claude_code_provider.py โ’ platform/ai-ros/src/providers/
- factory/providers/router.py โ’ platform/ai-ros/src/routing/
- factory/providers/__init__.py โ’ platform/ai-ros/src/

**Workflow Runtime (11 modules)**
- engine/workflow_runtime.py โ’ platform/workflow-runtime/src/runtime/
- engine/workflow_engine.py โ’ platform/workflow-runtime/src/runtime/ (merge)
- engine/workflow_supervisor.py โ’ platform/workflow-runtime/src/supervisor/
- engine/dispatcher.py โ’ platform/workflow-runtime/src/runtime/
- engine/review_router.py โ’ platform/workflow-runtime/src/runtime/
- engine/merge_manager.py โ’ platform/workflow-runtime/src/runtime/
- engine/approval_service.py โ’ platform/workflow-runtime/src/gates/
- engine/sla_monitor.py โ’ platform/workflow-runtime/src/monitoring/
- engine/blocker_monitor.py โ’ platform/workflow-runtime/src/monitoring/
- engine/root_cause_engine.py โ’ platform/workflow-runtime/src/monitoring/
- supervisor.py (root) โ’ platform/workflow-runtime/src/supervisor/

**Prompt OS (14 modules)**
- engine/prompt_os/_db.py โ’ platform/prompt-os/src/
- engine/prompt_os/template.py โ’ platform/prompt-os/src/
- engine/prompt_os/governance.py โ’ platform/prompt-os/src/
- engine/prompt_os/security.py โ’ platform/prompt-os/src/
- engine/prompt_os/context.py โ’ platform/prompt-os/src/
- engine/prompt_os/composition.py โ’ platform/prompt-os/src/
- engine/prompt_os/marketplace.py โ’ platform/prompt-os/src/
- engine/prompt_os/execution.py โ’ platform/prompt-os/src/
- engine/prompt_os/metrics.py โ’ platform/prompt-os/src/
- engine/prompt_os/brain_bridge.py โ’ platform/prompt-os/src/
- engine/prompt_os/orchestrator.py โ’ platform/prompt-os/src/
- engine/prompt_os/__init__.py โ’ platform/prompt-os/src/
- engine/prompt_intelligence.py โ’ platform/prompt-os/src/registry/
- engine/prompt_runtime.py โ’ platform/prompt-os/src/ (deprecated delegate)

**Knowledge / Central Brain (10 modules)**
- engine/central_brain.py โ’ platform/knowledge/src/brain/
- engine/brain.py โ’ platform/knowledge/src/brain/ (merge)
- engine/decision_engine.py โ’ platform/knowledge/src/decision/
- engine/memory_os.py โ’ platform/knowledge/src/memory/
- engine/experience_recorder.py โ’ platform/knowledge/src/experience/ (unified)
- engine/outcome_collector.py โ’ platform/knowledge/src/experience/
- engine/failure_analyzer.py โ’ platform/knowledge/src/experience/
- engine/learning_agent.py โ’ platform/knowledge/src/experience/
- engine/self_improvement.py โ’ platform/knowledge/src/improvement/
- factory/memory/ โ’ platform/knowledge/src/memory/

**Security (4 modules)**
- engine/security.py โ’ platform/security/src/
- api/auth.py โ’ platform/security/src/auth/
- api/security_middleware.py โ’ platform/security/src/ (new in Phase A)
- api/security_routes.py โ’ platform/security/src/api/

**Messaging (6 modules)**
- engine/nats_client.py โ’ platform/messaging/src/nats/
- engine/event_bus.py โ’ platform/messaging/src/event_bus/
- engine/event_schemas.py โ’ platform/messaging/src/schemas/
- api/nats_routes.py โ’ platform/messaging/src/api/
- api/ws_routes.py โ’ platform/messaging/src/websocket/
- factory/logging/ โ’ platform/observability/src/logging/

**Storage (3 modules)**
- factory/storage/ (10 files) โ’ platform/storage/src/

**Observability (4 modules)**
- factory/metrics/ โ’ platform/observability/src/metrics/
- factory/tracing/ โ’ platform/observability/src/tracing/
- factory/logging/ โ’ platform/observability/src/logging/
- api/health_routes.py โ’ platform/observability/src/api/

### Product Modules (21 total)

**ai-software-factory product**
- engine/product_intake.py โ’ products/ai-software-factory/src/intake/
- engine/business_analyst.py โ’ products/ai-software-factory/src/agents/
- engine/architect_agent.py โ’ products/ai-software-factory/src/agents/
- engine/planner_engine.py โ’ products/ai-software-factory/src/planning/
- engine/artifact_generator.py โ’ products/ai-software-factory/src/generation/
- engine/product_factory.py โ’ products/ai-software-factory/src/factory/
- engine/org_engine.py โ’ products/ai-software-factory/src/org/
- engine/employee_engine.py โ’ products/ai-software-factory/src/employees/
- engine/dependency_resolver.py โ’ products/ai-software-factory/src/planning/
- engine/infra_patterns.py โ’ products/ai-software-factory/src/patterns/
- factory/plugins/ (8 files) โ’ products/ai-software-factory/src/plugins/
- api/product_routes.py โ’ products/ai-software-factory/src/api/
- api/workflow_routes.py โ’ products/ai-software-factory/src/api/
- api/workflow_runtime_routes.py โ’ products/ai-software-factory/src/api/
- api/improvement_routes.py โ’ products/ai-software-factory/src/api/
- api/org_routes.py โ’ products/ai-software-factory/src/api/
- api/employee_routes.py โ’ products/ai-software-factory/src/api/
- api/plugin_sdk_routes.py โ’ products/ai-software-factory/src/api/
- api/proxy_routes.py โ’ products/ai-software-factory/src/api/
- app.py โ’ products/ai-software-factory/src/
- dashboard/ โ’ products/ai-software-factory/src/dashboard/

---

## Appendix B โ€” Provider Interface Reference Implementation

The following is the **design specification** (not production code) for `ClaudeCodeProvider` as reference implementation:

```
ClaudeCodeProvider implements AIProvider:

    provider_id = "claude_code"

    Initialization:
        - Locate Claude Code binary (which claude / where claude)
        - Store binary path
        - Detect available models via `claude --list-models` or config

    complete(request: CompletionRequest) -> CompletionResponse:
        1. Serialize request to Claude Code CLI arguments
        2. subprocess.run(["claude", "--model", request.model, "--no-stream", ...])
        3. Parse stdout JSON response
        4. Return CompletionResponse with normalized fields
        5. Record latency in metrics

    complete_async(request) -> CompletionResponse:
        - asyncio.create_subprocess_exec equivalent of complete()

    stream(request) -> AsyncGenerator[CompletionChunk]:
        1. subprocess with stdout pipe
        2. Read line by line (Claude Code streams JSON lines)
        3. Parse each line as CompletionChunk
        4. yield CompletionChunk to caller

    health_check() -> ProviderHealth:
        1. subprocess.run(["claude", "--version"], timeout=5)
        2. If exit code 0: status="healthy"
        3. If timeout: status="degraded"
        4. If not found: status="offline"

    get_capabilities() -> ProviderCapabilities:
        1. Returns static ProviderCapabilities based on known Claude models
        2. supports_streaming=True
        3. supports_function_calling=True
        4. supports_vision=True (claude-3+)
        5. max_context_tokens=200000 (claude-3-5+ sonnet)

    estimate_cost(request) -> CostEstimate:
        1. Count prompt tokens via tiktoken approximation
        2. Estimate completion tokens from request.max_tokens
        3. Apply Anthropic published pricing
        4. Return CostEstimate (no API call made)
```

---

## Appendix C โ€” Workspace SDK Panel: Desktop Integration Design

The Desktop `WorkspacePanel` displays the live state of the workspace. This design extends the existing `WorkspacePanel` with platform-awareness.

```
โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
โ”  AI STUDIO WORKSPACE                               โ—Online  โ”
โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”ค
โ”                                                            โ”
โ”  PLATFORM HEALTH                                           โ”
โ”  โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”  โ”
โ”  โ”  ai-ros          โ— Healthy  (5 providers active)     โ”  โ”
โ”  โ”  workflow-rt     โ— Healthy  (12 workers running)     โ”  โ”
โ”  โ”  prompt-os       โ— Healthy  (247 templates)          โ”  โ”
โ”  โ”  knowledge       โ— Healthy  (brain active)           โ”  โ”
โ”  โ”  security        โ— Healthy  (auth enforced)          โ”  โ”
โ”  โ”  messaging       โ— Healthy  (NATS connected)         โ”  โ”
โ”  โ”  persistence     โ— Healthy  (PostgreSQL pooled)      โ”  โ”
โ”  โ”  observability   โ— Healthy  (metrics + traces)       โ”  โ”
โ”  โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”  โ”
โ”                                                            โ”
โ”  AI PROVIDERS                                              โ”
โ”  โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”  โ”
โ”  โ”  claude_code     โ— Active   claude-sonnet-4-6        โ”  โ”
โ”  โ”  anthropic       โ— Active   claude-sonnet-4-6        โ”  โ”
โ”  โ”  openai          โ— Disabled                          โ”  โ”
โ”  โ”  gemini          โ— Disabled                          โ”  โ”
โ”  โ”  ollama          โ— Active   llama3.2                 โ”  โ”
โ”  โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”  โ”
โ”                                                            โ”
โ”  PRODUCTS                                                  โ”
โ”  โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”  โ”
โ”  โ”  ai-software-factory  v2.0.0  โ— Running  port 8088   โ”  โ”
โ”  โ”  content-factory      v1.5.5  โ— Running  port 8090   โ”  โ”
โ”  โ”  mythic-realms        v0.1.0  โ— Stopped             โ”  โ”
โ”  โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”  โ”
โ”                                                            โ”
โ”  AI COST TODAY                                             โ”
โ”  โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”  โ”
โ”  โ”  ai-software-factory  $12.43 / $50.00  โ–โ–โ–โ–โ–โ–โ–‘โ–‘โ–‘โ–‘   โ”  โ”
โ”  โ”  content-factory      $0.23  / $10.00  โ–โ–โ–‘โ–‘โ–‘โ–‘โ–‘โ–‘โ–‘โ–‘   โ”  โ”
โ”  โ”  Total                $12.66 / $60.00              โ”  โ”
โ”  โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”  โ”
โ”                                                            โ”
โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
```

This panel refreshes every 30 seconds using the existing `BaseController` backoff/refresh pattern. Data comes from the Workspace SDK client calling the platform `/api/v1/health` and `/api/v1/cost/summary` endpoints.

---

## Appendix D โ€” Architecture Linter: Rule Specification

The architecture linter (`tools/lint/arch_lint.py`) enforces the dependency rules from ADR-009.

### Rule Specification

```
Rule: no-product-in-platform
    Scope:  platform/**/*.py
    Pattern: import.*products\. | from products\.
    Action: FAIL
    Escape: # arch: allow โ€” reason required

Rule: no-direct-platform-in-product
    Scope:  products/**/*.py
    Pattern: import.*platform\. | from platform\.
    Action: FAIL
    Escape: # arch: allow โ€” reason required

Rule: no-cross-product-import
    Scope:  products/*/src/**/*.py
    Pattern: from products\.(ai-software-factory|content-factory|mythic-realms|vocal-coach|erp-generator)
    Action: FAIL
    Escape: NONE (cross-product imports are never allowed; use events instead)

Rule: no-prompt-in-product-code
    Scope:  products/**/*.py
    Pattern: """[\s\S]{50,}You are a[\s\S]+?""" | f"""[\s\S]*?\{task\}[\s\S]*?"""
    Action: WARN (Phase C start) โ’ FAIL (Phase C complete)
    Escape: # arch: allow โ€” reason required

Rule: no-direct-db-session-in-product
    Scope:  products/**/*.py
    Pattern: SessionLocal\(\) | Session\(\) | engine\.connect\(\)
    Action: WARN
    Escape: # arch: allow โ€” reason required

Rule: sdk-no-implementation
    Scope:  sdk/**/*.py
    Pattern: def .*:\n.*db\.query | def .*:\n.*httpx\.(get|post|put)
    Action: FAIL
    Escape: NONE (SDKs never implement โ€” they delegate)
```

### Linter Invocation

```bash
# Run as pre-commit hook
python tools/lint/arch_lint.py --strict --fail-on warn

# Run in CI (report mode only during Phase 0B)
python tools/lint/arch_lint.py --report-only --output arch-violations.json

# Run in CI (strict mode from Phase A onward)
python tools/lint/arch_lint.py --strict
```

---

*Document end.*

---

**AI-STUDIO-PLATFORM-REFACTORING-BLUEPRINT.md**  
**Version:** 1.0.0  
**Date:** 2026-06-28  
**Status:** DRAFT โ€” PENDING ARCHITECTURE REVIEW  
**Total Sections:** 20 + 4 Appendices  

*This document constitutes the Phase 0 deliverable. No implementation work begins until the CTO and Chief Architect have reviewed and signed off on the Architecture Freeze Checklist in Section 19.*

---

