# Domain Index

> All architectural domains in the AI Studio knowledge system.
> See [KNOWLEDGE-INDEX-DESIGN.md](docs/knowledge-architecture/KNOWLEDGE-INDEX-DESIGN.md) for the index design specification.

---

## Domain Registry

| Domain ID | Domain Name | Status | Capabilities | Owner |
|-----------|-------------|--------|-------------|-------|
| DOM-AI | AI Runtime OS | Active | ai-runtime-os | Chief Enterprise Architect |
| DOM-PROMPT | Prompt OS | Active | prompt-os | Capability Architect |
| DOM-WORKFLOW | Workflow Runtime | Active | workflow-runtime | Capability Architect |
| DOM-CONV | Conversation Intelligence | Active | conversation-intelligence | Capability Architect |
| DOM-CTX | Context Intelligence | Active | context-intelligence | Capability Architect |
| DOM-ROUTE | Intelligent Routing | Active | intelligent-routing | Capability Architect |
| DOM-PROVIDER | Provider Framework | Active | provider-framework, provider-management | Platform Architect |
| DOM-PLATFORM | Platform Core | Active | platform-security, platform-contracts, platform-standards | Platform Architect |
| DOM-KNOWLEDGE | Knowledge Architecture | Active | knowledge-architecture | Chief Knowledge Architect |
| DOM-PRODUCT | Product Layer | Active | content-factory, claude-cli | Product Architect |

---

## Domain Descriptions

### DOM-AI — AI Runtime OS

The foundational AI execution runtime. Provides the operating environment for all AI capabilities — resource management, process isolation, capability lifecycle, and the desktop runtime.

**Phase established:** 1.0
**Current phase:** 2.0D.0
**Key documents:** `docs/phase-1a/AI-RESOURCE-OS-BLUEPRINT.md` (pre-migration)

---

### DOM-PROMPT — Prompt OS

Prompt management, versioning, templating, and execution. The system that treats prompts as first-class managed artifacts rather than inline strings.

**Phase established:** 2.0
**Key documents:** (pending Phase 2.0D.1 migration)

---

### DOM-WORKFLOW — Workflow Runtime

Workflow definition, execution, orchestration, step management, and scheduling. The runtime that executes multi-step AI workflows.

**Phase established:** 2.0
**Key documents:** `docs/phase-1a/AI-SCHEDULER.md` (pre-migration)

---

### DOM-CONV — Conversation Intelligence

Conversation history management, session handling, turn management, and conversation context. The system that makes AI Studio conversational.

**Phase established:** 2.0
**Key documents:** `docs/phase-1a/AI-CONVERSATION-ENGINE.md` (pre-migration)

---

### DOM-CTX — Context Intelligence

Context window management, token budget optimization, relevance scoring, and context compression. The system that ensures AI requests use context optimally.

**Phase established:** 2.0
**Key documents:** (pending Phase 2.0D.1 migration)

---

### DOM-ROUTE — Intelligent Routing

Request routing, provider selection, load balancing, fallback chains, and routing policies. The system that decides which provider handles each request.

**Phase established:** 2.0
**Key documents:** (pending Phase 2.0D.1 migration)

---

### DOM-PROVIDER — Provider Framework

Provider abstraction, contracts, adapter pattern, API normalization, and provider lifecycle management. The system that makes AI Studio provider-agnostic.

**Phase established:** 2.0
**Key documents:** `docs/phase-1a/AI-PROVIDER-CONTRACTS.md`, `docs/phase-1a/AI-ACCOUNT-MANAGER.md` (pre-migration)

---

### DOM-PLATFORM — Platform Core

Cross-cutting platform concerns: security model, shared contracts, platform standards, infrastructure.

**Phase established:** 1.0
**Key documents:** `docs/SECURITY-MODEL.md`, `docs/PLATFORM-CONTRACTS.md`, `docs/PLATFORM-STANDARDS.md`

---

### DOM-KNOWLEDGE — Knowledge Architecture

The meta-domain. Architecture of the architecture system itself. Knowledge management, documentation intelligence, and repository standards.

**Phase established:** 2.0D.0
**Key documents:** `docs/knowledge-architecture/` (this folder)

---

### DOM-PRODUCT — Product Layer

Product-specific knowledge: Content Factory, Claude CLI, and future products. Each product is a consumer of platform capabilities.

**Phase established:** 1.5.5
**Key documents:** (pending Phase 2.0D.1 migration)

---

## Cross-Domain Dependencies

```
DOM-PRODUCT
    └── consumes → DOM-AI, DOM-WORKFLOW, DOM-PROMPT, DOM-CONV, DOM-CTX, DOM-ROUTE

DOM-WORKFLOW
    └── depends on → DOM-AI, DOM-PROMPT, DOM-PROVIDER, DOM-CTX

DOM-CONV
    └── depends on → DOM-AI, DOM-CTX

DOM-ROUTE
    └── depends on → DOM-PROVIDER, DOM-AI

DOM-PROMPT
    └── depends on → DOM-AI

DOM-PROVIDER
    └── depends on → DOM-PLATFORM (contracts)

DOM-AI
    └── depends on → DOM-PLATFORM (security, infrastructure)

DOM-KNOWLEDGE
    └── governs → all domains
```

---

*Domain index is manually maintained during Phase 2.0D.0. Auto-generated after Phase 2.0D.1 completion.*
