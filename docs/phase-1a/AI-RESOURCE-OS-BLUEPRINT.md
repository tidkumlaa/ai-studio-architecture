# AI Resource Operating System — Architecture Blueprint

**Version:** 1.0.0  
**Status:** Approved for Implementation  
**Phase:** 1A  
**Date:** 2026-06-29  
**Classification:** Internal Architecture — Single Source of Truth

---

## 1. Purpose

The AI Resource Operating System (AI ROS) is the centralized resource management kernel for all AI operations across the AI Studio platform. It sits between every product (AI Software Factory, Content Factory, Mythic Realms, ERP Generator, AI Vocal Coach) and every AI provider (Claude, OpenAI, Gemini, Ollama, Azure OpenAI, OpenRouter, LM Studio, DeepSeek, Qwen, and all future providers).

No product may communicate directly with any AI provider. Every AI operation — prompt execution, embedding generation, image synthesis, audio transcription, tool calling, streaming — passes through AI ROS.

AI ROS is not a proxy. It is an operating system: it owns scheduling, quota, budget, routing, conversation state, context management, policy enforcement, audit, and observability. Products submit AI work orders. AI ROS decides how, when, and where to execute them.

---

## 2. Design Principles

### P-01: Provider Opacity
Products never know which provider executes their request. Provider identity is an implementation detail of AI ROS. Changing providers requires zero product code changes.

### P-02: Plugin-First Architecture
Every provider is a plugin. Adding a new provider (DeepSeek, Qwen, a future local model) requires only: registering a plugin manifest, implementing the `AIProviderPlugin` interface, and hot-registering it. No core code changes.

### P-03: Policy Before Execution
Every request passes through the Policy Engine before any provider is contacted. Routing decisions, spend approval, compliance checks, and model allow/deny lists are enforced at this layer.

### P-04: Everything Is Auditable
Every request, response, routing decision, quota deduction, budget charge, policy evaluation, and error is written to the immutable audit log before the response is returned to the caller.

### P-05: Quota and Budget Are Hard Constraints
The scheduler will not dispatch a request that would breach a confirmed quota limit or exhaust a confirmed budget. Over-limit requests are queued (if configured) or rejected with a structured error.

### P-06: Conversation State Is Owned by AI ROS
Session history, branching, context compression, and memory integration are AI ROS responsibilities. Products receive session tokens. They do not manage context windows.

### P-07: Observability Is Not Optional
Every subsystem emits structured metrics, traces, and events. There is no subsystem that operates silently.

### P-08: High Availability by Default
AI ROS is designed for active-active deployment. No single point of failure. Circuit breakers protect against cascading provider failures. Failover is automatic.

### P-09: Multi-Tenancy From Day One
Every resource (quota, budget, conversation, audit record) is scoped to: organization → project → user. Isolation is enforced at the data layer, not the application layer.

### P-10: Future-Proof Capability Model
Model capabilities (vision, audio, reasoning, tool calling, structured output, embeddings) are registered in the Capability Registry and matched dynamically at routing time. New capability types are additive — no schema migration required.

---

## 3. System Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        AI Studio Products                                │
│  AI Software Factory │ Content Factory │ Mythic Realms │ ERP Generator  │
│                      │  AI Vocal Coach  │ Future Products                │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │ AI ROS Client SDK / REST / WebSocket
┌──────────────────────────────▼──────────────────────────────────────────┐
│                     AI Resource Operating System                          │
│                                                                           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │  Policy  │  │  Quota   │  │  Budget  │  │ Account  │  │  Audit   │  │
│  │  Engine  │  │  Engine  │  │  Engine  │  │ Manager  │  │  Layer   │  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  │
│       │              │              │              │              │        │
│  ┌────▼──────────────▼──────────────▼──────────────▼─────────────▼────┐  │
│  │                       Routing Engine                                 │  │
│  │   Capability Match → Policy Filter → Quota Check → Budget Check     │  │
│  │   → Benchmark Score → Load Balance → Circuit Break → Select         │  │
│  └─────────────────────────────┬────────────────────────────────────────┘  │
│                                 │                                           │
│  ┌──────────────────────────────▼────────────────────────────────────────┐  │
│  │                      Scheduling Engine                                  │  │
│  │    Priority Queue │ Rate Limit │ Retry │ Timeout │ Cancellation        │  │
│  └─────────────────────────────┬─────────────────────────────────────────┘  │
│                                 │                                            │
│  ┌──────────────────────────────▼────────────────────────────────────────┐  │
│  │                       Execution Engine                                  │  │
│  │    Prompt Runtime │ Tool Runtime │ MCP Runtime │ Streaming              │  │
│  └─────────────────────────────┬─────────────────────────────────────────┘  │
│                                 │                                            │
│  ┌──────────┐  ┌──────────┐  ┌─▼────────┐  ┌──────────┐  ┌──────────┐   │
│  │Conversat.│  │ Context  │  │ Provider │  │ Benchmark│  │  Cache   │   │
│  │  Engine  │  │  Engine  │  │  Health  │  │  Engine  │  │  Layer   │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
│                                                                            │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                      Provider Registry                                 │  │
│  │  Model Registry │ Capability Registry │ Circuit Breaker │ Load Bal.  │  │
│  └─────────────────────────────┬─────────────────────────────────────────┘  │
└─────────────────────────────────┼──────────────────────────────────────────┘
                                   │ Provider Plugin Interface (AIProviderPlugin)
┌──────────────────────────────────▼──────────────────────────────────────────┐
│                          Provider Runtime Layer                               │
│                                                                               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐     │
│  │  Claude  │  │  OpenAI  │  │  Gemini  │  │  Ollama  │  │ Azure OAI│     │
│  │   API    │  │          │  │          │  │          │  │          │     │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────┘     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐     │
│  │OpenRouter│  │ LM Studio│  │ DeepSeek │  │  Qwen    │  │ Future.. │     │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────┘     │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Major Subsystems

### 4.1 Provider Registry
Central catalog of all registered AI providers. Manages provider manifests, health status, capability declarations, and connection configurations. Supports hot registration — new providers can be registered without restart.

**Key responsibilities:**
- Accept and validate provider manifests
- Maintain provider health status (from Health Monitor)
- Expose provider capability matrix to the Routing Engine
- Support version-aware provider registration

### 4.2 Model Registry
Canonical catalog of every known AI model across all providers. Normalized capability descriptions, pricing, context windows, latency classifications, and benchmark scores.

**Key responsibilities:**
- Maintain canonical model identities independent of provider naming conventions
- Map canonical model IDs to provider-specific model IDs
- Store capability flags (vision, audio, tool_calling, structured_output, reasoning)
- Expose model selection API for routing decisions

### 4.3 Capability Registry
Normalized index of what each model can do. Decouples capability queries from provider and model specifics.

**Capability dimensions:**
- Text generation (chat, completion, instruction-following)
- Vision (image input, video input)
- Audio (transcription, speech synthesis, audio input)
- Embeddings (text, image, multimodal)
- Tool calling (function calling, parallel tools)
- Structured output (JSON mode, schema enforcement)
- Reasoning (chain-of-thought, extended thinking)
- Code execution (sandboxed interpreter)
- Web browsing
- File handling (PDF, DOCX, images)

### 4.4 Account Manager
Manages all provider credentials, subscription states, OAuth tokens, and API keys across all providers and organizations.

**Key responsibilities:**
- Secure storage of API keys (encrypted at rest, never logged)
- OAuth 2.0 flows for providers that support it
- Subscription state tracking (Claude Max, Claude Pro, OpenAI Plus, etc.)
- Key rotation scheduling and enforcement
- Per-organization credential isolation

### 4.5 Quota Engine
Tracks all usage against provider-imposed and organization-configured quotas. Enforces limits before dispatch. Forecasts depletion and triggers notifications.

**Quota dimensions:**
- Requests per minute / hour / day
- Tokens per minute / hour / day / month
- Images per day
- Audio minutes per day
- Subscription-tier token limits

### 4.6 Budget Engine
Owns all cost accounting for AI operations. Projects, departments, and organizations have budgets. Every execution is costed and charged back.

**Key responsibilities:**
- Real-time cost computation from token usage and pricing catalog
- Project and department budget enforcement
- Cost forecast and burn rate calculation
- Chargeback reports
- ROI analytics

### 4.7 Routing Engine
The intelligence layer that selects provider and model for each request. Runs a multi-stage selection pipeline.

**Selection pipeline:**
1. Capability filter (must support required capabilities)
2. Policy filter (organization routing rules, allow/deny lists)
3. Quota filter (must have remaining quota)
4. Budget filter (must have remaining budget)
5. Circuit breaker filter (must be healthy)
6. Benchmark score ranking (quality-weighted by task category)
7. Cost ranking (minimize cost within quality band)
8. Load balance (distribute across equivalent choices)
9. Final selection with rationale record

### 4.8 Scheduling Engine
Manages request queues, priorities, rate limiting, parallel execution, retry, timeout, and cancellation.

**Scheduling modes:**
- Immediate (synchronous): execute now or fail
- Queued (async): enqueue with priority, execute when capacity available
- Batch: collect and execute as group
- Scheduled: execute at specific time
- Recurring: repeat on schedule

### 4.9 Execution Engine
Executes AI operations against the selected provider through the Provider Plugin interface.

**Handles:**
- Text generation (streaming and non-streaming)
- Tool call round-trips
- MCP server invocation
- Image/audio generation
- Embedding generation
- Retry on transient failure
- Timeout enforcement

### 4.10 Conversation Engine
Owns all session and conversation lifecycle: creation, history, branching, sharing, compression, and archival.

**Key responsibilities:**
- Session token issuance and validation
- Message history storage and retrieval
- Conversation branching (fork at any point)
- Context window management
- Automatic history compression when nearing limit
- Cross-session memory injection (via Memory Integration)

### 4.11 Context Engine
Manages the context window budget for each execution. Decides what fits, what to compress, and what to retrieve from memory.

**Context assembly pipeline:**
1. System prompt (from Prompt Runtime)
2. Memory injections (from Memory Integration)
3. Tool definitions (from Tool Runtime)
4. Conversation history (recent + compressed)
5. Current user message
6. Budget enforcement (trim to fit model limit)

### 4.12 Prompt Runtime
Template engine and prompt security layer.

**Key responsibilities:**
- Handlebars-style template rendering (consistent with Prompt OS)
- Variable injection with type validation
- Injection attack detection (OWASP prompt injection patterns)
- Secret masking in logs
- Governance integration (approved prompts only in restricted mode)

### 4.13 Policy Engine
Evaluates all routing, compliance, and security policies before execution.

**Policy types:**
- Provider routing rules (force to specific provider, exclude providers)
- Model allow/deny lists per organization
- Regional data residency restrictions
- Content moderation requirements
- Spend approval thresholds
- Human approval triggers

### 4.14 Approval Engine
Human-in-the-loop for high-stakes AI operations.

**Triggers:**
- Execution cost exceeds threshold
- Sensitive data detected in prompt
- Model capability risk score exceeds threshold
- Organization policy requires approval for operation type

### 4.15 Tool Runtime
Manages tool definitions, tool call execution, and tool result injection.

**Key responsibilities:**
- Tool schema registry (JSON Schema definitions)
- Tool call routing (local function, HTTP endpoint, MCP server)
- Result validation
- Tool call audit logging

### 4.16 MCP Runtime
Model Context Protocol server management.

**Key responsibilities:**
- MCP server registration and health monitoring
- Protocol negotiation (MCP v1, future versions)
- Tool and resource exposition via MCP
- Security sandboxing for MCP servers

### 4.17 Memory Integration
Bridges AI ROS with the Memory OS layer.

**Provides:**
- Relevant memory retrieval for context assembly
- Experience recording after execution
- Long-term pattern learning feed

### 4.18 Benchmark Engine
Tracks and scores provider/model performance by task category.

**Benchmark dimensions:**
- Task success rate
- Response quality score (human and automated)
- Latency (p50, p95, p99)
- Token efficiency
- Cost per successful outcome

### 4.19 Circuit Breaker
Protects the system from cascading provider failures.

**States:** CLOSED → OPEN → HALF_OPEN  
**Transitions:** failure rate threshold, success rate recovery

### 4.20 Provider Health Monitor
Proactively monitors all registered providers.

**Checks:**
- API endpoint reachability
- Latency probe (synthetic request)
- Error rate from recent executions
- Quota headroom
- Certificate validity

### 4.21 Cache Layer
Reduces cost and latency by caching deterministic responses.

**Cache types:**
- Exact match cache (identical prompt + model)
- Semantic similarity cache (configurable threshold)
- Embedding cache (deduplication)

### 4.22 Analytics and Forecast Engine
Usage analytics, trend analysis, and spend forecasting.

### 4.23 Audit Layer
Immutable append-only log of every significant event in AI ROS.

---

## 5. Data Architecture

AI ROS uses a CQRS pattern with separated write models and read models.

### Write Models (Command Side)
- `ai_ros_requests` — every execution request
- `ai_ros_executions` — execution results
- `ai_ros_quota_ledger` — quota deduction events
- `ai_ros_budget_ledger` — cost charge events
- `ai_ros_routing_decisions` — routing rationale
- `ai_ros_audit_log` — immutable event log
- `ai_ros_provider_health` — health check results
- `ai_ros_conversations` — session state
- `ai_ros_messages` — message history

### Read Models (Query Side)
- `ai_ros_quota_summaries` — aggregated quota position
- `ai_ros_budget_summaries` — aggregated spend position
- `ai_ros_provider_scorecards` — provider performance view
- `ai_ros_model_scorecards` — model performance view
- `ai_ros_usage_by_project` — project-level usage view
- `ai_ros_cost_by_org` — organization spend view

Full schema in `AI-RESOURCE-DATABASE.md`.

---

## 6. API Architecture

AI ROS exposes three API surfaces:

### REST API (`/api/v1/ai-ros/`)
Standard request/response for: execute, queue, cancel, status, sessions, quota, budget, routing, models, providers.

### WebSocket API (`ws://ai-ros/stream`)
Streaming execution results, live routing events, real-time quota alerts.

### Event Bus
All significant state changes are published as typed events. Consumers subscribe by topic. Full catalog in `AI-RESOURCE-EVENTS.md`.

---

## 7. Security Architecture

- All provider credentials encrypted at rest (AES-256-GCM)
- Credentials never appear in logs, events, or API responses
- Per-organization isolation enforced at database row level
- Audit log is append-only and tamper-evident
- Prompt injection detection on all inputs
- TLS required for all provider communication

Full security model in `AI-RESOURCE-SECURITY.md`.

---

## 8. Deployment Architecture

### Minimum viable deployment (single node)
- AI ROS process (FastAPI)
- PostgreSQL (or SQLite for development)
- Redis (cache layer, scheduling queue)
- 1 GB RAM, 2 CPU cores

### Production deployment (clustered)
- AI ROS: 3+ instances behind load balancer (active-active)
- PostgreSQL: primary + replica, connection pooling via PgBouncer
- Redis: sentinel or cluster mode
- Provider plugins: loaded from shared plugin directory or registry

### High availability
- Each AI ROS instance maintains local circuit breaker state (shared via Redis)
- Scheduling queue is Redis-backed (persistent, recoverable)
- No in-process state that cannot survive instance restart

---

## 9. Implementation Roadmap

### Phase 1A.0 — Core Infrastructure (Week 1-2)
- Provider Registry
- Model Registry
- Capability Registry
- Account Manager (API key storage)
- AIProviderPlugin interface
- First two provider adapters (Claude API, Ollama)
- REST API skeleton

### Phase 1A.1 — Execution Pipeline (Week 3-4)
- Routing Engine (basic capability + health filter)
- Scheduling Engine (immediate mode)
- Execution Engine (text generation, streaming)
- Quota Engine (per-minute rate limiting)
- Budget Engine (real-time cost tracking)
- Audit Layer

### Phase 1A.2 — Conversation & Context (Week 5-6)
- Conversation Engine (sessions, history)
- Context Engine (window management, compression)
- Prompt Runtime (templates, injection detection)
- Policy Engine (routing rules, allow/deny lists)

### Phase 1A.3 — Intelligence Layer (Week 7-8)
- Benchmark Engine
- Recommendation Engine
- Circuit Breaker (full state machine)
- Cache Layer (exact match)
- Provider Health Monitor (proactive)

### Phase 1A.4 — Enterprise Features (Week 9-10)
- Approval Engine
- Tool Runtime
- MCP Runtime
- Memory Integration
- Forecast Engine
- Desktop dashboards

### Phase 1A.5 — Multi-Provider Expansion (Week 11-12)
- OpenAI adapter
- Gemini adapter
- Azure OpenAI adapter
- OpenRouter adapter
- LM Studio adapter
- DeepSeek adapter
- Qwen adapter

---

## 10. Cross References

| Document | Relationship |
|----------|-------------|
| AI-PROVIDER-CONTRACTS.md | Provider plugin interface specification |
| AI-MODEL-CATALOG.md | Canonical model registry |
| AI-ACCOUNT-MANAGER.md | Credential management detail |
| AI-QUOTA-ENGINE.md | Quota system detail |
| AI-BUDGET-ENGINE.md | Budget and cost system detail |
| AI-SCHEDULER.md | Scheduling algorithms detail |
| AI-POLICY-ENGINE.md | Policy evaluation detail |
| AI-CONVERSATION-ENGINE.md | Session management detail |
| AI-BENCHMARK-CENTER.md | Performance benchmarking detail |
| AI-RESOURCE-API.md | API surface specification |
| AI-RESOURCE-DATABASE.md | Database schema |
| AI-RESOURCE-EVENTS.md | Event catalog |
| AI-RESOURCE-SECURITY.md | Security model detail |
| AI-RESOURCE-DESKTOP.md | Desktop dashboard specification |
| PLATFORM-EXTRACTION-ROADMAP.md | Phase 0C prerequisite |
| PLATFORM-CONTRACTS.md | Platform-wide interface contracts |
| REPOSITORY-DEPENDENCY-RULES.md | Import rules AI ROS must follow |
