# AI Studio Platform โ€” Interface Contracts

**Document ID:** PLATFORM-CONTRACTS  
**Version:** 1.0.0  
**Date:** 2026-06-28  
**Status:** RATIFIED  
**Classification:** INTERNAL โ€” ARCHITECTURE GOVERNANCE  
**Authority:** Chief Software Architect  
**Predecessor:** AI-STUDIO-PLATFORM-REFACTORING-BLUEPRINT.md, PLATFORM-STANDARDS.md  
**Review Cycle:** Aligned with SDK major version releases

---

## Contract Authority Notice

> This document defines the binding public interface contracts between all platform layers.  
> These contracts are frozen upon ratification. No change may be made to a contract without:  
> 1. A new contract version published in this document  
> 2. A backward compatibility statement  
> 3. A migration guide for all consuming parties  
> 4. CTO approval  
>  
> All consumers โ€” products, the desktop, AI agents, and future services โ€” rely on these contracts.  
> Breaking a contract without the above process is a platform incident, not a code change.

---

## How to Read This Document

Each interface is specified across 25 dimensions. Read the full specification before implementing any consumer of an interface. The specification is the truth โ€” if implementation diverges from specification, the implementation is wrong.

**Notation conventions:**

| Symbol | Meaning |
|--------|---------|
| `โ’` | Method returns |
| `*` | Required field |
| `?` | Optional field |
| `[...]` | List type |
| `\|` | Union type |
| `enum(...)` | Enumeration of permitted values |
| `UUID` | RFC 4122 UUID string |
| `ISO8601` | UTC datetime string `YYYY-MM-DDTHH:MM:SS.sssZ` |

---

## Table of Contents

1. Contract Principles
2. AIClient โ€” AI Resource Operating System Interface
3. WorkflowClient โ€” Workflow Runtime Interface
4. PromptClient โ€” Prompt OS Interface
5. BrainClient โ€” Knowledge and Intelligence Interface
6. WorkspaceClient โ€” Workspace Registry Interface
7. ProviderClient โ€” AI Provider Adapter Interface
8. SecurityClient โ€” Security Platform Interface
9. KnowledgeClient โ€” Knowledge Graph Interface
10. EventBus โ€” Platform Messaging Interface
11. PluginAPI โ€” Plugin Extension Interface
12. Cross-Contract Standards
13. Appendix A โ€” Shared DTO Catalog
14. Appendix B โ€” Shared Error Codes
15. Appendix C โ€” Contract Versioning Log

---

---

# 1. Contract Principles

## 1.1 Layering Rule

Every contract in this document defines a boundary that flows in one direction only:

```
[Products / Desktop]
        โ”
        โ” consumes via SDK contracts
        โ–ผ
[Platform Contracts Defined Here]
        โ”
        โ” consumes provider adapters via ProviderClient
        โ–ผ
[External AI Providers / Infrastructure]
```

No contract defined here may reference a product namespace. No product may call an external AI provider directly.

## 1.2 Contract Stability Tiers

| Tier | Contract | Stability |
|------|---------|-----------|
| STABLE | AIClient, WorkflowClient, PromptClient, SecurityClient | No breaking changes within a major version |
| BETA | BrainClient, KnowledgeClient, EventBus | Breaking changes with 2-sprint notice |
| EXPERIMENTAL | WorkspaceClient, PluginAPI | Interface may evolve rapidly |

Tier is marked on each contract header.

## 1.3 Universal Contract Properties

Every contract in this document obeys these properties unless explicitly overridden:

| Property | Default |
|----------|---------|
| Transport | In-process (same deployment); HTTP/JSON when cross-service |
| Content-Type | `application/json` |
| Authentication | JWT or API key via `AuthContext` injected by SecurityMiddleware |
| Correlation | Every call propagates `correlation_id` |
| Idempotency | Reads are always idempotent; writes require explicit idempotency key |
| Error format | `PlatformError` envelope (see Section 13, Appendix B) |
| Timeout default | 30 seconds per contract call |
| Retry default | 3 attempts with exponential backoff, 2ร— multiplier, base 500ms |

## 1.4 DTO Conventions

All DTOs in this document are design specifications. They map to Pydantic v2 models in implementation. Field names are snake_case. All IDs are UUID. All timestamps are ISO8601 UTC.

---

---

# 2. AIClient โ€” AI Resource Operating System Interface

**Stability Tier:** STABLE  
**SDK Package:** `aistudio.sdk.ai`  
**Platform Module:** `aistudio.platform.ai_ros`  
**Contract Version:** 1.0

---

## 2.1 Purpose

AIClient is the single gateway through which all products, agents, and platform modules access AI language model capabilities. No consumer may call an AI provider directly. AIClient abstracts provider selection, cost enforcement, quota management, circuit breaking, and streaming.

## 2.2 Responsibilities

AIClient is responsible for:
1. Authenticating the caller and enforcing budget and quota limits before any provider call
2. Selecting the appropriate provider based on capability requirements and routing strategy
3. Executing the request against the selected provider with retry and circuit breaker
4. Tracking token usage and cost for every execution
5. Managing multi-turn conversation sessions
6. Streaming responses to callers in real time
7. Publishing execution events to the platform event bus
8. Providing pre-execution cost estimates without calling a provider

AIClient is **not** responsible for:
- Prompt template rendering (that is PromptClient's responsibility)
- Business logic within the prompt content
- Storing artifacts or outputs (that is the caller's responsibility)

## 2.3 Ownership

**Platform Owner:** AI ROS team  
**Interface Owner:** Chief Architect  
**Consumer Teams:** All products, Workflow Runtime (task dispatch), Brain (pattern analysis)

## 2.4 Methods

---

### Method: `execute`

**Description:** Execute a single, non-streaming AI completion request. Blocks until the response is available or the timeout is reached.

**Input DTO: `ExecutionRequest`**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `prompt` | string | * | The human turn prompt. Mutually exclusive with `prompt_id`. |
| `prompt_id` | UUID | ? | Prompt OS template ID. Renders the template before execution. Mutually exclusive with `prompt`. |
| `prompt_variables` | map[string, any] | ? | Variables for `prompt_id` template rendering. Ignored if `prompt` is used. |
| `system_prompt` | string | ? | System prompt override. If omitted and `prompt_id` is set, Prompt OS provides the system prompt. |
| `session_id` | UUID | ? | If provided, this execution is appended to an existing conversation session. |
| `model_preference` | enum(COST, SPEED, CAPABILITY, RELIABILITY, BALANCED) | ? | Default: BALANCED |
| `provider_hint` | string | ? | Preferred provider ID (e.g., `"anthropic"`). Treated as a hint, not a requirement โ€” AI ROS may route elsewhere if the provider is degraded. |
| `max_tokens` | integer | ? | Maximum completion tokens. Default: provider default. |
| `temperature` | float [0.0, 2.0] | ? | Default: 0.7 |
| `stop_sequences` | [string] | ? | Sequences that stop generation. |
| `tools` | [string] | ? | Tool names from ToolRuntime to make available. |
| `cost_budget_usd` | float | ? | Per-execution cost cap. Default: from platform config. Request is rejected before execution if estimate exceeds this. |
| `priority` | enum(URGENT, STANDARD, BATCH) | ? | Default: STANDARD |
| `stream` | boolean | ? | Default: false. If true, use `stream()` method instead. |
| `correlation_id` | UUID | ? | Propagated from the calling request context. Auto-populated if not provided. |
| `metadata` | map[string, string] | ? | Caller-provided key-value pairs logged alongside the execution. |

**Output DTO: `ExecutionResult`**

| Field | Type | Description |
|-------|------|-------------|
| `execution_id` | UUID | Unique identifier for this execution. Stable for idempotency. |
| `session_id` | UUID \| null | Session this execution was appended to, or null if session-less. |
| `content` | string | The AI-generated completion text. |
| `stop_reason` | enum(END_TURN, MAX_TOKENS, STOP_SEQUENCE, TOOL_USE) | Why generation stopped. |
| `tool_calls` | [ToolCallResult] | Tool calls made during execution, in order. |
| `model_used` | string | The exact model that served this request (e.g., `claude-sonnet-4-6`). |
| `provider_used` | string | Provider ID that served this request (e.g., `anthropic`). |
| `prompt_tokens` | integer | Tokens consumed in the prompt. |
| `completion_tokens` | integer | Tokens generated in the completion. |
| `total_tokens` | integer | Sum of prompt and completion tokens. |
| `cost_usd` | float | Actual cost for this execution. |
| `duration_ms` | integer | Wall-clock time from request submission to response receipt. |
| `provider_latency_ms` | integer | Time spent in the provider API call only. |
| `correlation_id` | UUID | Echoed from request for trace correlation. |
| `created_at` | ISO8601 | Execution start timestamp. |

---

### Method: `stream`

**Description:** Execute an AI completion and stream the response as it is generated. Returns an async iterator of chunks.

**Input DTO:** Same as `ExecutionRequest` with `stream: true` (enforced).

**Output DTO: `ExecutionStreamChunk`** โ€” emitted zero or more times, then `ExecutionStreamEnd`

| Field | Type | Description |
|-------|------|-------------|
| `execution_id` | UUID | Same execution_id throughout the stream. |
| `chunk_index` | integer | Zero-based sequence number of this chunk. |
| `delta` | string | Incremental content in this chunk. |
| `tool_call_delta` | ToolCallDelta \| null | Incremental tool call data if a tool call is in progress. |
| `correlation_id` | UUID | Trace correlation. |

**Terminal DTO: `ExecutionStreamEnd`**

| Field | Type | Description |
|-------|------|-------------|
| `execution_id` | UUID | |
| `stop_reason` | enum(END_TURN, MAX_TOKENS, STOP_SEQUENCE, TOOL_USE) | |
| `model_used` | string | |
| `provider_used` | string | |
| `total_tokens` | integer | |
| `cost_usd` | float | |
| `duration_ms` | integer | |

---

### Method: `estimate_cost`

**Description:** Return a cost estimate for a request without executing it. Makes no provider API calls. Used by callers to decide whether to proceed.

**Input DTO: `CostEstimateRequest`**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `prompt` | string | * | The prompt text to estimate. |
| `model_preference` | enum(...) | ? | Default: BALANCED |
| `max_tokens` | integer | ? | Assumed completion tokens if not provided: provider average. |

**Output DTO: `CostEstimate`**

| Field | Type | Description |
|-------|------|-------------|
| `provider_id` | string | The provider estimated for. |
| `model_id` | string | The specific model estimated. |
| `prompt_tokens_estimated` | integer | Token count estimate for the prompt. |
| `completion_tokens_estimated` | integer | Token count estimate for completion. |
| `cost_usd_estimated` | float | Estimated total cost in USD. |
| `confidence` | float [0.0, 1.0] | Estimation confidence. Low if prompt variables are unknown. |
| `basis` | enum(HISTORICAL_AVERAGE, TEMPLATE_ANALYSIS, TIKTOKEN) | How the estimate was produced. |

---

### Method: `create_session`

**Description:** Create a new multi-turn conversation session. All subsequent executions that reference the session_id append to this conversation context.

**Input DTO: `CreateSessionRequest`**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `product_id` | string | * | The product creating this session. |
| `model_binding` | string | ? | Pin this session to a specific model. If not set, model may vary per execution. |
| `system_prompt` | string | ? | Session-level system prompt applied to all executions. |
| `metadata` | map[string, string] | ? | |

**Output DTO: `SessionHandle`**

| Field | Type | Description |
|-------|------|-------------|
| `session_id` | UUID | Use in subsequent `execute()` calls to continue conversation. |
| `product_id` | string | |
| `model_binding` | string \| null | |
| `created_at` | ISO8601 | |
| `expires_at` | ISO8601 | Session TTL. Default: 24 hours of inactivity. |

---

### Method: `close_session`

**Description:** Explicitly close a session and release its resources. Sessions also expire automatically per TTL.

**Input:** `session_id: UUID`  
**Output:** `SessionCloseResult { session_id, final_token_count, final_cost_usd, closed_at }`

---

### Method: `get_provider_catalog`

**Description:** Return the list of all registered providers and their current health and capabilities.

**Input:** None (or optional `product_id` for product-scoped view)  
**Output:** `ProviderCatalog { providers: [ProviderStatus], retrieved_at: ISO8601 }`

**ProviderStatus fields:** `provider_id`, `display_name`, `health` (enum: HEALTHY, DEGRADED, OFFLINE), `models: [ModelInfo]`, `supports_streaming`, `supports_function_calling`, `circuit_breaker_state` (enum: CLOSED, OPEN, HALF_OPEN)

---

## 2.5 Error Model

| Error Code | HTTP | Trigger |
|-----------|------|---------|
| `AI_BUDGET_DAILY_LIMIT_EXCEEDED` | 429 | Product has exceeded its daily spend limit |
| `AI_QUOTA_RATE_LIMIT_EXCEEDED` | 429 | Product has exceeded its per-minute request quota |
| `AI_NO_SUITABLE_PROVIDER` | 503 | No provider can serve the request (all circuit-open or capability mismatch) |
| `AI_PROVIDER_TIMEOUT` | 504 | Provider did not respond within the execution timeout |
| `AI_TOKEN_LIMIT_EXCEEDED` | 400 | Request exceeds the model's context window |
| `AI_PROMPT_VALIDATION_FAILED` | 400 | PromptValidator rejected the prompt (risk_score >= 0.75) |
| `AI_SESSION_NOT_FOUND` | 404 | Provided session_id does not exist or has expired |
| `AI_COST_ESTIMATE_EXCEEDED` | 400 | Estimated cost exceeds `cost_budget_usd` โ€” execution not started |
| `AI_INVALID_TOOL` | 400 | One or more requested tool names are not registered |
| `AI_EXECUTION_FAILED` | 500 | Provider returned an error after all retries exhausted |

## 2.6 Authentication

All calls to AIClient require a valid `AuthContext` in the request state (populated by SecurityMiddleware). The `product_id` for cost and quota tracking is derived from `AuthContext.identity`. Direct calls without an `AuthContext` raise `AUTH_MISSING_CONTEXT`.

## 2.7 Authorization

| Method | Required Permission |
|--------|-------------------|
| `execute` | `AI_EXECUTE` |
| `stream` | `AI_EXECUTE` |
| `estimate_cost` | `AI_READ` |
| `create_session` | `AI_EXECUTE` |
| `close_session` | `AI_EXECUTE` |
| `get_provider_catalog` | `AI_READ` |

## 2.8 Thread Safety

AIClient is fully thread-safe and async-safe. Multiple coroutines may call `execute()` concurrently. The SchedulingEngine serializes access to the priority queues internally using asyncio primitives โ€” callers do not need external synchronization.

Session state is isolated per `session_id`. Concurrent calls to the same `session_id` are serialized by the SessionManager โ€” the second call blocks until the first completes.

## 2.9 Lifecycle

```
Application startup
    โ’ AI ROS initializes ProviderRegistry
    โ’ Each provider: health_check() called
    โ’ CapabilityMatrix populated from provider capabilities
    โ’ SchedulingEngine starts queue workers
    โ’ BudgetManager loads current daily spend from DB
    โ’ AIClient ready to accept calls

Application shutdown
    โ’ SchedulingEngine drains URGENT queue (max 30s)
    โ’ STANDARD and BATCH queues: in-flight requests complete; new requests rejected
    โ’ All provider connections closed
    โ’ Final spend totals persisted
```

## 2.10 Versioning

Contract version 1.0. All fields marked `?` (optional) may be added as required fields only in a MAJOR version increment. New optional fields may be added in MINOR versions. Method signatures (method name, required input fields, output fields) are frozen until MAJOR v2.

## 2.11 Backward Compatibility

- New optional fields in input DTOs: MINOR increment โ€” existing callers unaffected
- New fields in output DTOs: MINOR increment โ€” existing callers ignore unknown fields
- New error codes: MINOR increment โ€” callers that do not handle them receive the base `AIROSError`
- Removing a field (input or output): MAJOR increment
- Changing a field type: MAJOR increment
- Removing a method: MAJOR increment

## 2.12 Performance Expectations

| Operation | P50 target | P99 target | Measurement point |
|-----------|-----------|-----------|-------------------|
| `estimate_cost` | < 5ms | < 20ms | AIClient boundary |
| `execute` (excluding provider) | < 10ms overhead | < 50ms overhead | AIClient overhead only |
| `create_session` | < 10ms | < 30ms | AIClient boundary |
| `get_provider_catalog` (cached) | < 2ms | < 5ms | AIClient boundary |

Provider latency is excluded โ€” it is measured separately and reported in `ExecutionResult.provider_latency_ms`.

## 2.13 Timeout

| Operation | Default timeout | Configurable? |
|-----------|----------------|---------------|
| `execute` (URGENT priority) | 30 seconds | Yes โ€” `AISF_AI_ROS__URGENT_TIMEOUT_SECONDS` |
| `execute` (STANDARD priority) | 300 seconds | Yes โ€” `AISF_AI_ROS__STANDARD_TIMEOUT_SECONDS` |
| `execute` (BATCH priority) | 3600 seconds | Yes โ€” `AISF_AI_ROS__BATCH_TIMEOUT_SECONDS` |
| `estimate_cost` | 2 seconds | No |
| `create_session` | 5 seconds | No |

Timeout is measured from the moment the request enters the scheduling queue, not from when the provider call starts. Queue wait time is included.

## 2.14 Retry Policy

AIClient retries internally before returning an error to the caller. Callers do **not** implement their own retry logic.

| Failure type | Retries | Backoff | Retry on same provider? |
|-------------|---------|---------|------------------------|
| Provider timeout | 2 | Exponential: 500ms, 1000ms | No โ€” try next provider |
| Provider rate limit (429) | 3 | Respect Retry-After header | Same provider |
| Provider server error (5xx) | 2 | Exponential: 500ms, 1000ms | No โ€” try next provider |
| Network error | 2 | Exponential: 200ms, 400ms | No โ€” try next provider |
| Budget exceeded | 0 | N/A โ€” immediate rejection | N/A |
| Validation failure | 0 | N/A โ€” immediate rejection | N/A |

After all retries are exhausted across all eligible providers, `AI_EXECUTION_FAILED` is raised.

## 2.15 Circuit Breaker

Per-provider circuit breaker with three states. Does not apply to `estimate_cost` or `get_provider_catalog`.

| Parameter | Value | Configurable? |
|-----------|-------|--------------|
| Failure threshold | 5 failures | Yes |
| Failure window | 60 seconds | Yes |
| Open timeout | 60 seconds | Yes |
| Half-open probe count | 1 | No |
| Success threshold to close | 2 consecutive | No |

When a provider's circuit is OPEN, that provider is excluded from candidate selection. If all providers are OPEN, `AI_NO_SUITABLE_PROVIDER` is returned immediately (no retry).

## 2.16 Caching

`get_provider_catalog` is cached for 30 seconds. All other methods are not cached โ€” AI executions are inherently non-deterministic and must not be cached.

Cost estimates may be cached per `(prompt_hash, model_preference, max_tokens)` tuple for 60 seconds.

## 2.17 Idempotency

`execute` is not idempotent by design โ€” calling it twice produces two executions and two billing events.

Callers that require idempotent execution (e.g., retry-safe workflow task dispatch) must:
1. Check `execution_id` in their own state before calling `execute`
2. Store the `ExecutionResult` keyed on the caller's idempotency key immediately after receiving it

The platform does not provide execution deduplication.

## 2.18 Events Published

| Event | When |
|-------|------|
| `ai.execution.started` | Request dequeued and provider call initiated |
| `ai.execution.completed` | Provider returned successfully |
| `ai.execution.failed` | All retries exhausted |
| `ai.provider.circuit_opened` | Circuit breaker trips |
| `ai.provider.circuit_closed` | Circuit breaker recovers |
| `ai.budget.threshold_reached` | Product spend crosses 80% of daily limit |
| `ai.budget.limit_exceeded` | Product spend reaches 100% of daily limit |
| `ai.quota.rate_limit_exceeded` | Product exceeds per-minute request quota |

## 2.19 Events Consumed

| Event | Action |
|-------|--------|
| None | AIClient does not consume external events |

## 2.20 Sequence Diagram

```
Caller                  AIClient              SchedulingEngine         BudgetManager
  โ”                        โ”                        โ”                       โ”
  โ”โ”€โ”€ execute(request) โ”€โ”€โ–บ โ”                        โ”                       โ”
  โ”                        โ”โ”€โ”€ enqueue(request) โ”€โ”€โ–บ โ”                       โ”
  โ”                        โ”                        โ”โ”€โ”€ check(product_id)โ”€โ”€โ–บ โ”
  โ”                        โ”                        โ” โ—โ”€โ”€ OK โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
  โ”                        โ”                        โ”                       โ”
  โ”                        โ”                  CapabilityMatrix
  โ”                        โ”                  LoadBalancer
  โ”                        โ”                        โ”โ”€โ”€ select_provider() โ”€โ”€โ–บโ”
  โ”                        โ”                        โ” โ—โ”€โ”€ provider_id โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
  โ”                        โ”                        โ”                       โ”
  โ”                        โ”              CircuitBreaker    Provider
  โ”                        โ”                        โ”โ”€โ”€ allow(provider)? โ”€โ”€โ–บ โ”
  โ”                        โ”                        โ” โ—โ”€โ”€ CLOSED โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
  โ”                        โ”                        โ”โ”€โ”€ provider.complete() โ”€โ–บโ”
  โ”                        โ”                        โ” โ—โ”€โ”€ CompletionResponse โ”€โ”
  โ”                        โ”                        โ”                       โ”
  โ”                        โ”              CostTracker
  โ”                        โ”                        โ”โ”€โ”€ record(usage) โ”€โ”€โ”€โ”€โ”€โ”€โ–บโ”
  โ”                        โ”                        โ”โ”€โ”€ publish(ai.execution.completed)
  โ” โ—โ”€โ”€ ExecutionResult โ”€โ”€โ”€โ”                        โ”
```

## 2.21 State Diagram

```
                    โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
                    โ”           EXECUTION STATES              โ”
                    โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”

  execute() called
        โ”
        โ–ผ
  โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
  โ”   QUEUED     โ” โ”€โ”€โ”€โ”€ budget exceeded โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ–บ REJECTED
  โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”ฌโ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
         โ” dequeued
         โ–ผ
  โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
  โ”  VALIDATING  โ” โ”€โ”€โ”€โ”€ prompt risk โฅ 0.75 โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ–บ REJECTED
  โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”ฌโ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
         โ” validation passed
         โ–ผ
  โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
  โ”  ROUTING     โ” โ”€โ”€โ”€โ”€ no provider available โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ–บ FAILED
  โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”ฌโ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
         โ” provider selected
         โ–ผ
  โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”   timeout/error              โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
  โ”  EXECUTING   โ” โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ–บ โ”   RETRYING   โ”
  โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”ฌโ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”                              โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”ฌโ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
         โ” success                    retries exhaustedโ”
         โ–ผ                                             โ–ผ
  โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”                            โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
  โ”  COMPLETED   โ”                            โ”    FAILED    โ”
  โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”                            โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
```

## 2.22 Future Extension Points

| Extension | Description | Target version |
|-----------|-------------|---------------|
| Multi-modal inputs | Accept image/audio alongside text prompt | v1.2 |
| Batch execution | Submit N prompts as one batch, receive results when all complete | v1.1 |
| Provider-pinned sessions | Pin a session to a specific provider + model permanently | v1.1 |
| Cost forecasting API | Estimate cost for a WorkflowPlan before submission | v1.2 |
| Async callback | Register a webhook/event for long-running BATCH executions | v2.0 |

---

---

# 3. WorkflowClient โ€” Workflow Runtime Interface

**Stability Tier:** STABLE  
**SDK Package:** `aistudio.sdk.workflow`  
**Platform Module:** `aistudio.platform.workflow_runtime`  
**Contract Version:** 1.0

---

## 3.1 Purpose

WorkflowClient is the interface through which products submit, monitor, and control multi-step AI workflow executions. The Workflow Runtime orchestrates task dispatch, agent assignment, human approval gates, checkpoint/resume, and SLA monitoring. Products submit a plan and receive events โ€” they never directly manage task execution.

## 3.2 Responsibilities

WorkflowClient is responsible for:
1. Accepting `WorkflowPlan` submissions and returning a handle for monitoring
2. Providing real-time status of running workflows and individual tasks
3. Accepting human approval gate decisions
4. Providing checkpoint data for crash recovery
5. Accepting cancel, pause, and resume commands
6. Publishing lifecycle events for all state transitions

WorkflowClient is **not** responsible for:
- Executing AI calls within tasks (that is AIClient's responsibility)
- Rendering prompts for tasks (that is PromptClient's responsibility)
- Recording learned outcomes (the Brain subscribes to workflow events independently)

## 3.3 Ownership

**Platform Owner:** Workflow Runtime team  
**Interface Owner:** Principal Engineer  
**Consumer Teams:** All products that create multi-step AI pipelines

## 3.4 Methods

---

### Method: `submit`

**Description:** Submit a workflow plan for execution. Returns immediately with a handle. Execution is asynchronous.

**Input DTO: `WorkflowSubmitRequest`**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `product_id` | string | * | Product submitting this workflow. |
| `plan_name` | string | * | Human-readable name for this workflow. |
| `tasks` | [TaskDefinition] | * | Ordered list of task definitions. Minimum 1. |
| `dag_edges` | [[string, string]] | ? | Dependency pairs `[task_id, depends_on_task_id]`. If empty, tasks execute in submission order. |
| `gates` | [GateDefinition] | ? | Human approval checkpoints between specified task IDs. |
| `priority` | enum(URGENT, STANDARD, BATCH) | ? | Default: STANDARD. Propagated to AI executions within the workflow. |
| `metadata` | map[string, string] | ? | Product-defined metadata. Returned in status and events. |
| `idempotency_key` | string | ? | If provided, duplicate submissions with the same key within 24 hours return the same WorkflowHandle without creating a new workflow. |

**TaskDefinition fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `task_id` | string | * | Caller-assigned unique ID within this plan. Used in dag_edges. |
| `task_type` | string | * | Type label (e.g., `code_generation`, `test_writing`, `review`). Used for routing to agent type. |
| `agent_type` | string | * | The agent role that executes this task (e.g., `architect`, `developer`, `reviewer`). |
| `prompt_id` | UUID | ? | Prompt OS template ID. Mutually exclusive with `prompt_text`. |
| `prompt_text` | string | ? | Direct prompt text. Mutually exclusive with `prompt_id`. Using `prompt_id` is strongly preferred. |
| `prompt_variables` | map[string, any] | ? | Variables for `prompt_id` template rendering. |
| `inputs` | map[string, any] | ? | Task inputs. Referenced outputs from dependency tasks are merged in at runtime. |
| `priority` | integer [1, 10] | ? | Default: 5. Higher number = higher priority within the workflow. |
| `timeout_seconds` | integer | ? | Default: 300. Task-level timeout. |
| `max_retries` | integer | ? | Default: 2. Number of times to retry a failed task before marking FAILED. |

**GateDefinition fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `gate_id` | string | * | Caller-assigned ID. |
| `after_task_id` | string | * | Workflow pauses for approval after this task completes. |
| `before_task_id` | string | * | Workflow resumes and starts this task after approval. |
| `approver_role` | enum(ADMIN, DEVELOPER, REVIEWER) | * | Minimum role required to approve. |
| `description` | string | ? | Shown to the human approver in the desktop UI. |
| `auto_approve_after_seconds` | integer | ? | If set, gate auto-approves after this many seconds. MUST NOT be used in production without explicit CTO approval. Currently: forbidden (see Architecture Review finding). |

**Output DTO: `WorkflowHandle`**

| Field | Type | Description |
|-------|------|-------------|
| `workflow_id` | UUID | Stable identifier for this workflow. Use for all subsequent calls. |
| `product_id` | string | |
| `status` | enum(QUEUED, ACTIVE, PAUSED, COMPLETED, FAILED, CANCELLED) | |
| `task_count` | integer | Total tasks in the plan. |
| `created_at` | ISO8601 | |
| `idempotency_key` | string \| null | Echoed if provided. |

---

### Method: `get_status`

**Description:** Return the current status of a workflow and all its tasks.

**Input:** `workflow_id: UUID`

**Output DTO: `WorkflowStatus`**

| Field | Type | Description |
|-------|------|-------------|
| `workflow_id` | UUID | |
| `product_id` | string | |
| `plan_name` | string | |
| `status` | enum(QUEUED, ACTIVE, PAUSED, COMPLETED, FAILED, CANCELLED) | |
| `tasks` | [TaskStatus] | Status of every task in the workflow. |
| `current_gate` | GateStatus \| null | If PAUSED, the gate awaiting approval. |
| `started_at` | ISO8601 \| null | |
| `completed_at` | ISO8601 \| null | |
| `duration_ms` | integer \| null | Wall-clock time from start to completion/failure. Null if still running. |
| `total_cost_usd` | float | Sum of AI execution costs across all tasks. |
| `metadata` | map[string, string] | |

**TaskStatus fields:** `task_id`, `task_type`, `agent_type`, `status` (enum: PENDING, READY, IN_PROGRESS, REVIEW, APPROVED, STALLED, FAILED, CANCELLED, COMPLETED), `started_at`, `completed_at`, `duration_ms`, `retry_count`, `error_message`

---

### Method: `get_task_output`

**Description:** Return the output produced by a completed task.

**Input:** `workflow_id: UUID`, `task_id: string`

**Output DTO: `TaskOutput`**

| Field | Type | Description |
|-------|------|-------------|
| `task_id` | string | |
| `workflow_id` | UUID | |
| `status` | enum | |
| `output` | map[string, any] | Task-produced output. Schema is task_type-specific. |
| `ai_execution_ids` | [UUID] | AIClient execution IDs that produced this output. |
| `tool_calls` | [ToolCallSummary] | Summary of tools called during this task. |
| `completed_at` | ISO8601 \| null | |

---

### Method: `approve_gate`

**Description:** Approve a human approval gate, allowing the workflow to continue.

**Input DTO: `GateApprovalRequest`**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `workflow_id` | UUID | * | |
| `gate_id` | string | * | |
| `decision` | enum(APPROVED, REJECTED) | * | |
| `comment` | string | ? | Reviewer comment stored in audit log. |
| `approver_identity` | string | * | Identity from AuthContext โ€” set by SecurityMiddleware, not by caller. |

**Output DTO: `GateDecisionResult`**

| Field | Type | Description |
|-------|------|-------------|
| `gate_id` | string | |
| `decision` | enum(APPROVED, REJECTED) | |
| `workflow_status` | enum(ACTIVE, FAILED) | ACTIVE if approved; FAILED if rejected. |
| `next_task_id` | string \| null | The task that will now execute (if approved). |
| `decided_at` | ISO8601 | |

---

### Method: `cancel`

**Description:** Cancel a running or queued workflow. In-progress tasks are allowed to complete; new tasks are not started.

**Input:** `workflow_id: UUID`, `reason: string`  
**Output:** `CancelResult { workflow_id, status: CANCELLED, cancelled_at, tasks_completed, tasks_cancelled }`

---

### Method: `pause`

**Description:** Pause a running workflow after the current in-progress tasks complete. No new tasks are dispatched.

**Input:** `workflow_id: UUID`  
**Output:** `PauseResult { workflow_id, status: PAUSED, paused_at, next_task_id }`

---

### Method: `resume`

**Description:** Resume a paused workflow from the next scheduled task.

**Input:** `workflow_id: UUID`  
**Output:** `ResumeResult { workflow_id, status: ACTIVE, resumed_at, next_task_id }`

---

### Method: `get_checkpoint`

**Description:** Return the checkpoint state of a workflow for crash recovery. Includes all completed task outputs and current execution state.

**Input:** `workflow_id: UUID`  
**Output DTO: `WorkflowCheckpoint`**

| Field | Type | Description |
|-------|------|-------------|
| `workflow_id` | UUID | |
| `plan` | WorkflowPlan | The original submitted plan. |
| `completed_task_outputs` | map[string, TaskOutput] | Outputs of all COMPLETED tasks. |
| `last_completed_task_id` | string \| null | |
| `checkpoint_at` | ISO8601 | |
| `can_resume` | boolean | True if there is a recoverable state to resume from. |

---

### Method: `list_workflows`

**Description:** List workflows for a product with filtering and cursor-based pagination.

**Input DTO: `WorkflowListQuery`**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `product_id` | string | * | |
| `status` | [enum] | ? | Filter by status. Multiple values = OR. |
| `created_after` | ISO8601 | ? | |
| `created_before` | ISO8601 | ? | |
| `page_size` | integer [1, 100] | ? | Default: 20 |
| `page_cursor` | string | ? | Opaque cursor from previous response. |

**Output DTO: `WorkflowListResult`**

| Field | Type | Description |
|-------|------|-------------|
| `items` | [WorkflowSummary] | Abbreviated workflow records. |
| `total` | integer | Total matching workflows (regardless of pagination). |
| `page_size` | integer | |
| `next_cursor` | string \| null | Null if last page. |
| `has_more` | boolean | |

---

## 3.5 Error Model

| Error Code | HTTP | Trigger |
|-----------|------|---------|
| `WORKFLOW_NOT_FOUND` | 404 | Workflow ID does not exist |
| `WORKFLOW_ALREADY_COMPLETED` | 409 | Operation on a terminal-state workflow |
| `WORKFLOW_ALREADY_CANCELLED` | 409 | Cancel on already-cancelled workflow |
| `WORKFLOW_TASK_NOT_FOUND` | 404 | Task ID not found in workflow |
| `WORKFLOW_GATE_NOT_FOUND` | 404 | Gate ID not in current workflow state |
| `WORKFLOW_GATE_NOT_PENDING` | 409 | Gate already decided |
| `WORKFLOW_INVALID_PLAN` | 400 | DAG has cycles or references nonexistent task IDs |
| `WORKFLOW_EMPTY_PLAN` | 400 | Submitted plan has zero tasks |
| `WORKFLOW_UNAUTHORIZED_APPROVER` | 403 | Caller's role is below `approver_role` required by gate |
| `WORKFLOW_IDEMPOTENCY_CONFLICT` | 409 | Same idempotency_key submitted with different plan |

## 3.6 Authentication

Requires `AuthContext` from SecurityMiddleware. `product_id` is cross-validated against `AuthContext.identity` โ€” a product may only submit workflows for its own `product_id`.

## 3.7 Authorization

| Method | Required Permission |
|--------|-------------------|
| `submit` | `WORKFLOW_CREATE` |
| `get_status` | `WORKFLOW_READ` |
| `get_task_output` | `WORKFLOW_READ` |
| `approve_gate` | `WORKFLOW_APPROVE` (and `approver_role` check) |
| `cancel` | `WORKFLOW_MANAGE` |
| `pause` | `WORKFLOW_MANAGE` |
| `resume` | `WORKFLOW_MANAGE` |
| `get_checkpoint` | `WORKFLOW_READ` |
| `list_workflows` | `WORKFLOW_READ` |

## 3.8 Thread Safety

WorkflowClient is fully thread-safe. Concurrent `submit()` calls for different workflows execute independently. Concurrent operations on the same `workflow_id` are serialized by the WorkflowRuntime โ€” the second operation blocks until the first completes and the state is consistent.

## 3.9 Lifecycle

```
Platform startup
    โ’ WorkflowRuntime loads all ACTIVE workflows from DB
    โ’ Incomplete tasks are re-enqueued for dispatch
    โ’ STALLED tasks are identified and root-cause analyzed
    โ’ WorkflowClient ready to accept calls

Platform shutdown
    โ’ All in-progress tasks receive a cancellation signal
    โ’ Completed task outputs are checkpointed to DB
    โ’ WorkflowRuntime persists all workflow states
    โ’ Graceful shutdown timeout: 60 seconds (then force shutdown)
```

## 3.10 Versioning

Contract version 1.0. The `auto_approve_after_seconds` field in GateDefinition exists in the schema for forward compatibility but is rejected (HTTP 400) if set to any non-null value until explicitly enabled by a platform release.

## 3.11 Performance Expectations

| Operation | P50 target | P99 target |
|-----------|-----------|-----------|
| `submit` | < 50ms | < 200ms |
| `get_status` | < 20ms | < 100ms |
| `get_task_output` | < 20ms | < 100ms |
| `approve_gate` | < 50ms | < 200ms |
| `cancel` / `pause` / `resume` | < 100ms | < 500ms |
| `list_workflows` (page_size=20) | < 50ms | < 200ms |

## 3.12 Timeout

`submit()` returns immediately (async). All other methods: 10 second default timeout.

## 3.13 Retry Policy

Callers may retry `get_status`, `get_checkpoint`, and `list_workflows` safely (they are read-only and idempotent). Callers must not retry `submit()` without an `idempotency_key` โ€” duplicate submissions create duplicate workflows. `approve_gate`, `cancel`, `pause`, `resume` are idempotent only if the workflow is still in a compatible state.

## 3.14 Idempotency

`submit()` with `idempotency_key`: identical plans with the same key within 24 hours return the original `WorkflowHandle` without creating a new workflow. Different plans with the same key return `WORKFLOW_IDEMPOTENCY_CONFLICT`.

`cancel()`, `pause()`, `resume()` are naturally idempotent โ€” calling them on an already-cancelled/paused/active workflow is a no-op with the appropriate current status returned.

## 3.15 Events Published

| Event | When |
|-------|------|
| `workflows.workflow.created` | Workflow accepted and queued |
| `workflows.workflow.completed` | All tasks completed successfully |
| `workflows.workflow.cancelled` | Workflow cancelled |
| `workflows.workflow.failed` | Workflow failed (task failed after max retries) |
| `workflows.task.started` | Task dispatched to agent |
| `workflows.task.completed` | Task completed successfully |
| `workflows.task.failed` | Task failed (will retry or escalate) |
| `workflows.task.stalled` | Task in IN_PROGRESS beyond stall threshold |
| `workflows.task.approved` | Human gate approved |
| `workflows.task.rejected` | Human gate rejected |

## 3.16 Events Consumed

WorkflowRuntime (internal) consumes:
- `ai.execution.completed` โ’ update task status with AI execution result
- `ai.execution.failed` โ’ trigger task retry or task failure

WorkflowClient (the SDK) does not consume external events.

## 3.17 Sequence Diagram

```
Product              WorkflowClient      WorkflowRuntime     AIClient
  โ”                       โ”                    โ”                โ”
  โ”โ”€โ”€submit(plan) โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ–บโ”                    โ”                โ”
  โ”                       โ”โ”€โ”€create_workflow()โ”€โ–บโ”                โ”
  โ”                       โ”                    โ”โ”€โ”€persistโ”€โ”€โ”€โ”€โ”€โ”€โ”€โ–บDB
  โ”โ—โ”€โ”€WorkflowHandle โ”€โ”€โ”€โ”€โ”€โ”                    โ”                โ”
  โ”                       โ”                    โ”โ”€โ”€dispatch_task()
  โ”                       โ”                    โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ–บ Agent
  โ”                       โ”                    โ”                โ”
  โ”                       โ”                    โ” Agent calls:   โ”
  โ”                       โ”                    โ”โ—โ”€โ”€ execute() โ”€โ”€โ”
  โ”                       โ”                    โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ–บโ”
  โ”                       โ”                    โ”โ—โ”€โ”€ result โ”€โ”€โ”€โ”€ โ”
  โ”                       โ”                    โ”                โ”
  โ”โ”€โ”€get_status() โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ–บโ”                    โ”                โ”
  โ”โ—โ”€โ”€WorkflowStatus โ”€โ”€โ”€โ”€โ”€โ”                    โ”                โ”
  โ”                       โ”                    โ”                โ”
  โ” [gate reached]         โ”                    โ”                โ”
  โ”โ—โ”€โ”€(event: task.approved_required)          โ”                โ”
  โ”โ”€โ”€approve_gate() โ”€โ”€โ”€โ”€โ”€โ”€โ–บโ”                    โ”                โ”
  โ”                       โ”โ”€โ”€record_decision()โ”€โ–บโ”                โ”
  โ”                       โ”                    โ”โ”€โ”€resume_dispatch
```

## 3.18 State Diagram

```
                    โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
                    โ”          WORKFLOW STATES                 โ”
                    โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”

  submit() called
        โ”
        โ–ผ
  โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
  โ”   QUEUED     โ” โ”€โ”€โ”€โ”€ cancel() โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€ โ–บ CANCELLED
  โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”ฌโ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
         โ” task dispatched
         โ–ผ
  โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”        โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
  โ”   ACTIVE     โ” โ”€โ”€โ”€โ”€โ”€โ”€โ–บโ”   PAUSED     โ” โ”€โ”€pause()โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
  โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”ฌโ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”        โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”ฌโ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”                                โ”
         โ”               resume()โ”                                         โ”
         โ”โ—โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”                                         โ”
         โ”                                                                 โ”
         โ”โ”€โ”€โ”€โ”€ cancel() โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ–บ CANCELLED  โ”
         โ”                                                                 โ”
         โ”โ”€โ”€โ”€โ”€ gate reached โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ–บ PAUSED โ—โ”€โ”€โ”
         โ”   (gate approval โ”€โ”€โ–บ ACTIVE)
         โ”   (gate rejection โ”€โ”€โ–บ FAILED)
         โ”
         โ”โ”€โ”€โ”€โ”€ all tasks complete โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ–บ COMPLETED
         โ”
         โ””โ”€โ”€โ”€โ”€ task failed (retries exhausted) โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ–บ FAILED
```

## 3.19 Future Extension Points

| Extension | Description | Target version |
|-----------|-------------|---------------|
| Parallel sub-workflows | A task can spawn child workflows | v1.1 |
| Workflow templates | Reusable plan templates registered in platform | v1.2 |
| Webhook callbacks | Notify external URLs on state transitions | v1.2 |
| Distributed execution | Tasks dispatched to remote agent workers via NATS | v2.0 |
| Workflow versioning | Re-run a workflow with a different plan version | v2.0 |

---

---

# 4. PromptClient โ€” Prompt OS Interface

**Stability Tier:** STABLE  
**SDK Package:** `aistudio.sdk.prompt`  
**Platform Module:** `aistudio.platform.prompt_os`  
**Contract Version:** 1.0

---

## 4.1 Purpose

PromptClient is the single interface through which all products and platform modules access prompt templates. No prompt string may exist in product code. All prompts are registered in Prompt OS and referenced by ID. PromptClient provides rendering, versioning, governance state management, and A/B test management.

## 4.2 Responsibilities

PromptClient is responsible for:
1. Rendering registered prompt templates with caller-provided variables
2. Assembling multi-part prompt compositions
3. Validating rendered prompts via the security validator
4. Managing template lifecycle (DRAFT โ’ REVIEW โ’ ACTIVE โ’ DEPRECATED โ’ ARCHIVED)
5. Providing version history and diff between versions
6. Managing A/B tests between prompt variants
7. Recording render metrics (latency, token counts, cost) for analytics

PromptClient is **not** responsible for:
- Executing the rendered prompt (that is AIClient's responsibility)
- Storing prompt outputs (that is the caller's responsibility)
- Designing the prompt (that is the product team's responsibility)

## 4.3 Ownership

**Platform Owner:** Prompt OS team  
**Interface Owner:** Chief Architect  
**Consumer Teams:** All products; AIClient (auto-renders when `prompt_id` is specified)

## 4.4 Methods

---

### Method: `render`

**Description:** Render a registered prompt template with provided variables. Returns the fully rendered prompt string. Verifies the template's HMAC signature before rendering โ€” a tampered template is rejected.

**Input DTO: `PromptRenderRequest`**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `template_id` | UUID | * | Registered template ID in Prompt OS. |
| `variables` | map[string, any] | ? | Variable values for `{{variable}}` placeholders. Missing variables use template defaults; if no default, render fails. |
| `version` | integer | ? | Specific version to render. Default: the currently ACTIVE version. |
| `environment` | string | ? | Deployment environment (`local`, `staging`, `production`). Used to select environment-specific deployments. Default: from platform config. |
| `ab_test_assignment` | string | ? | If this render is part of an A/B test, the variant identifier (`"A"` or `"B"`). If omitted, the assigned variant is auto-selected by Prompt OS. |

**Output DTO: `PromptRenderResult`**

| Field | Type | Description |
|-------|------|-------------|
| `render_id` | UUID | Unique identifier for this render. Used for A/B result recording. |
| `template_id` | UUID | |
| `version` | integer | The version that was rendered. |
| `variant` | string \| null | A/B test variant used, or null if no A/B test active. |
| `rendered_prompt` | string | The fully rendered prompt text. |
| `rendered_system_prompt` | string \| null | Rendered system prompt if the template includes one. |
| `variable_count` | integer | Number of variables substituted. |
| `estimated_tokens` | integer | Approximate token count of the rendered output. |
| `render_duration_ms` | integer | |
| `signature_verified` | boolean | Always true (render fails if signature is invalid). |

---

### Method: `compose`

**Description:** Render multiple templates and concatenate them in order. Each template renders independently with its own variables. Used for constructing large prompts from modular parts.

**Input DTO: `PromptComposeRequest`**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `parts` | [ComposePart] | * | Ordered list of templates to render and concatenate. |
| `separator` | string | ? | String inserted between rendered parts. Default: `"\n\n"` |

**ComposePart fields:** `template_id`, `variables`, `version` (all same as `render`)

**Output DTO: `PromptComposeResult`**

| Field | Type | Description |
|-------|------|-------------|
| `compose_id` | UUID | |
| `rendered_prompt` | string | Concatenated result of all parts. |
| `parts` | [PromptRenderResult] | Individual render results for each part. |
| `total_estimated_tokens` | integer | Sum of all parts' token estimates. |

---

### Method: `register`

**Description:** Register a new prompt template. New templates are created in DRAFT state. Requires PROMPT_MANAGE permission.

**Input DTO: `PromptRegisterRequest`**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `template_id` | string | ? | Caller-chosen slug (e.g., `"aisf/architect/blueprint_v2"`). Auto-generated UUID if not provided. |
| `display_name` | string | * | Human-readable name. |
| `description` | string | ? | Purpose and usage notes. |
| `content` | string | * | The Handlebars template content with `{{variable}}` placeholders. |
| `system_prompt_content` | string | ? | Optional system prompt portion of the template. |
| `variables` | [VariableDefinition] | ? | Declared variables with types and default values. |
| `tags` | [string] | ? | Classification tags. |
| `owner` | string | * | Team or product that owns this template. |

**VariableDefinition fields:** `name` (string, required), `type` (enum: string, integer, float, boolean, list, map), `required` (boolean), `default` (any), `description` (string)

**Output DTO: `PromptRegistration`**

| Field | Type | Description |
|-------|------|-------------|
| `template_id` | UUID | Assigned (or confirmed) template ID. |
| `version` | integer | Always 1 for new registrations. |
| `state` | enum(DRAFT) | New templates are always DRAFT. |
| `hmac_signature` | string | HMAC-SHA256 of template content. Stored with template. |
| `created_at` | ISO8601 | |

---

### Method: `update`

**Description:** Create a new version of an existing template. Does not affect the currently ACTIVE version until `activate()` is called.

**Input DTO: `PromptUpdateRequest`**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `template_id` | UUID | * | |
| `content` | string | ? | New template content. If null, content is unchanged. |
| `system_prompt_content` | string | ? | New system prompt. |
| `description` | string | ? | |
| `variables` | [VariableDefinition] | ? | |
| `change_notes` | string | * | Required description of what changed and why. |

**Output DTO: `PromptVersion`**

| Field | Type | Description |
|-------|------|-------------|
| `template_id` | UUID | |
| `version` | integer | Incremented from previous version. |
| `state` | enum(DRAFT) | New versions are always DRAFT. |
| `parent_version` | integer | |
| `change_notes` | string | |
| `hmac_signature` | string | |
| `created_at` | ISO8601 | |

---

### Method: `transition`

**Description:** Advance a template through the governance state machine.

**Input DTO: `GovernanceTransitionRequest`**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `template_id` | UUID | * | |
| `version` | integer | * | |
| `target_state` | enum(REVIEW, ACTIVE, DEPRECATED, ARCHIVED, SUSPENDED) | * | Must be a valid transition from current state (see state diagram). |
| `comment` | string | ? | Required for SUSPENDED and ARCHIVED transitions. |

**Output DTO: `GovernanceTransitionResult`**

| Field | Type | Description |
|-------|------|-------------|
| `template_id` | UUID | |
| `version` | integer | |
| `previous_state` | enum | |
| `current_state` | enum | |
| `transitioned_at` | ISO8601 | |
| `transitioned_by` | string | Identity from AuthContext. |

---

### Method: `get_diff`

**Description:** Return a character-level diff between two versions of a template.

**Input:** `template_id: UUID`, `version_a: integer`, `version_b: integer`  
**Output DTO: `PromptDiff`** โ€” `{ template_id, version_a, version_b, diff_lines: [DiffLine], added_chars, removed_chars }`

---

### Method: `record_ab_result`

**Description:** Record the outcome of an A/B test render. Called by the consuming product after the AI execution completes.

**Input DTO: `ABTestResultRequest`**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `render_id` | UUID | * | From `PromptRenderResult.render_id`. |
| `execution_id` | UUID | * | From `AIClient.ExecutionResult.execution_id`. |
| `outcome` | enum(SUCCESS, FAILURE, PARTIAL) | * | |
| `quality_score` | float [0.0, 1.0] | ? | Caller-assessed quality of the AI output. |
| `notes` | string | ? | |

**Output:** `ABTestResultAck { render_id, recorded_at }`

---

## 4.5 Error Model

| Error Code | HTTP | Trigger |
|-----------|------|---------|
| `PROMPT_TEMPLATE_NOT_FOUND` | 404 | Template ID does not exist |
| `PROMPT_VERSION_NOT_FOUND` | 404 | Requested version does not exist |
| `PROMPT_NOT_ACTIVE` | 409 | Template is not in ACTIVE state and no version specified |
| `PROMPT_SIGNATURE_INVALID` | 422 | HMAC signature mismatch โ€” template may have been tampered |
| `PROMPT_VARIABLE_MISSING` | 400 | Required variable not provided and no default defined |
| `PROMPT_VARIABLE_TYPE_ERROR` | 400 | Variable value does not match declared type |
| `PROMPT_INVALID_TRANSITION` | 409 | Target governance state is not reachable from current state |
| `PROMPT_RENDER_FAILED` | 500 | Template engine error during rendering |
| `PROMPT_VALIDATION_FAILED` | 400 | Rendered content failed PromptValidator risk check |
| `PROMPT_TEMPLATE_ID_CONFLICT` | 409 | Caller-provided template_id already exists |

## 4.6 Authentication

Requires `AuthContext`. Renders (read operations) require `PROMPT_READ`. Management operations (register, update, transition) require `PROMPT_MANAGE`.

## 4.7 Thread Safety

PromptClient is fully thread-safe. Template rendering is stateless โ€” concurrent renders of the same template are independent. The governance state machine uses optimistic locking โ€” concurrent transitions to the same target state return `PROMPT_INVALID_TRANSITION` for the second caller.

## 4.8 Lifecycle

```
Platform startup
    โ’ Prompt OS loads ACTIVE template index into memory (cache)
    โ’ HMAC signature keys loaded from SecretConfig
    โ’ TemplateEngine initialized
    โ’ PromptClient ready

Template cache refresh: every 60 seconds (configurable)
Template cache invalidation: on governance state transition event
```

## 4.9 Performance Expectations

| Operation | P50 target | P99 target |
|-----------|-----------|-----------|
| `render` (cached template) | < 5ms | < 20ms |
| `render` (cache miss) | < 20ms | < 50ms |
| `compose` (3 parts, cached) | < 15ms | < 60ms |
| `register` / `update` | < 100ms | < 300ms |
| `transition` | < 50ms | < 200ms |

## 4.10 Caching

ACTIVE template content is cached in memory. Cache key: `(template_id, version)`. TTL: 60 seconds (refreshed on access). On governance transition event (`prompts.template.activated`), the cache entry for the affected template is invalidated immediately.

Renders are not cached โ€” variables make each render potentially unique.

## 4.11 Events Published

| Event | When |
|-------|------|
| `prompts.template.registered` | New template created |
| `prompts.template.updated` | New version created |
| `prompts.template.activated` | Template moved to ACTIVE state |
| `prompts.template.deprecated` | Template deprecated |
| `prompts.template.archived` | Template archived |
| `prompts.template.suspended` | Template emergency-suspended |

## 4.12 State Diagram

```
                    โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
                    โ”      PROMPT GOVERNANCE STATES           โ”
                    โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”

  register() called
        โ”
        โ–ผ
  โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
  โ”    DRAFT     โ”โ”€โ”€โ”€โ”€ transition(REVIEW) โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ–บโ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
  โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”                                            โ”    REVIEW    โ”
                                                              โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”ฌโ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
                                            transition(ACTIVE)โ”€โ”€โ”€โ”€โ”€โ”€โ–บโ”
                                                                     โ”
                                            โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ—โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
                                            โ”   ACTIVE     โ”
                                            โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”ฌโ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
                                                   โ”
                              transition(DEPRECATED)โ”
                                                   โ–ผ
                                            โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
                                            โ”  DEPRECATED  โ”โ”€โ”€โ–บ transition(ARCHIVED) โ”€โ”€โ–บ ARCHIVED
                                            โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”

                    SUSPENDED (emergency, from ACTIVE only):
                    ACTIVE โ”€โ”€โ–บ SUSPENDED โ”€โ”€โ–บ ACTIVE (re-enable)
                                        โ”€โ”€โ–บ ARCHIVED (retire)

  Notes:
  - REVIEW โ’ REJECTED is not a state; rejected template returns to DRAFT
  - ARCHIVED is terminal; no transitions out
  - A new version of an ARCHIVED template starts as DRAFT (new version, new lifecycle)
```

## 4.13 Future Extension Points

| Extension | Description | Target version |
|-----------|-------------|---------------|
| Prompt branching | Fork a template to a named branch | v1.1 |
| Multi-language templates | Same template_id, locale-specific content | v1.2 |
| Prompt marketplace | Community template registry | v2.0 |
| Semantic search | Find templates by meaning, not ID | v1.2 |
| Auto-optimization | Brain-driven A/B test winner selection | v1.3 |

---

---

# 5. BrainClient โ€” Knowledge and Intelligence Interface

**Stability Tier:** BETA  
**SDK Package:** `aistudio.sdk.brain`  
**Platform Module:** `aistudio.platform.knowledge`  
**Contract Version:** 0.9 (BETA โ€” breaking changes with 2-sprint notice)

---

## 5.1 Purpose

BrainClient provides access to the Central Brain โ€” the cross-product intelligence layer that accumulates knowledge from every AI execution across all products. Products consult the Brain for recommendations before making decisions, and record experiences after outcomes are known. The Brain compounds intelligence across all products over time.

## 5.2 Responsibilities

BrainClient is responsible for:
1. Providing recommendations based on historical patterns and current context
2. Finding similar past projects or executions for a given context
3. Accepting experience records after task or workflow completion
4. Providing discovered patterns for a given domain or task type
5. Generating self-improvement proposals based on accumulated patterns

BrainClient is **not** responsible for:
- Storing raw artifacts (that is the caller's responsibility)
- Executing AI calls (that is AIClient's responsibility)
- Querying structured knowledge graphs directly (that is KnowledgeClient's responsibility)

## 5.3 Ownership

**Platform Owner:** Knowledge team  
**Interface Owner:** Chief Architect  
**Consumer Teams:** All products; WorkflowRuntime (task planning); PromptClient (template recommendation)

## 5.4 Methods

---

### Method: `recommend`

**Description:** Request recommendations relevant to the current context. The Brain synthesizes patterns from past executions across all products to produce actionable recommendations.

**Input DTO: `RecommendationRequest`**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `product_id` | string | * | |
| `task_type` | string | * | The type of task for which recommendations are needed (e.g., `"blueprint_generation"`, `"test_writing"`). |
| `context` | map[string, any] | * | Relevant context for the recommendation (e.g., language, framework, domain). |
| `max_recommendations` | integer [1, 20] | ? | Default: 5 |
| `min_confidence` | float [0.0, 1.0] | ? | Minimum recommendation confidence score. Default: 0.5 |
| `include_similar_projects` | boolean | ? | Default: true โ€” include similar past projects in recommendations. |

**Output DTO: `RecommendationResult`**

| Field | Type | Description |
|-------|------|-------------|
| `request_id` | UUID | |
| `recommendations` | [Recommendation] | Ordered by confidence descending. |
| `similar_projects` | [SimilarProject] | Past projects with similar characteristics. |
| `pattern_count` | integer | Number of patterns consulted. |
| `generated_at` | ISO8601 | |

**Recommendation fields:** `recommendation_id`, `type` (enum: PROMPT_TEMPLATE, AGENT_STRATEGY, TOOL_SEQUENCE, ARCHITECTURE_PATTERN, AVOID), `content` (string), `rationale` (string), `confidence` (float), `based_on_executions` (integer)

**SimilarProject fields:** `project_id`, `product_id`, `similarity_score` (float), `task_type`, `outcome` (enum: SUCCESS, FAILURE, PARTIAL), `key_attributes` (map[string, any])

---

### Method: `record_experience`

**Description:** Record the outcome of a completed task or workflow execution. This is how the Brain learns. Should be called after every significant AI execution.

**Input DTO: `ExperienceRecord`**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `product_id` | string | * | |
| `execution_id` | UUID | ? | AIClient execution ID, if applicable. |
| `workflow_id` | UUID | ? | WorkflowClient workflow ID, if applicable. |
| `task_type` | string | * | |
| `context` | map[string, any] | * | Same context as was passed to `recommend()`. |
| `outcome` | enum(SUCCESS, FAILURE, PARTIAL) | * | |
| `quality_score` | float [0.0, 1.0] | ? | Caller-assessed quality. |
| `duration_ms` | integer | ? | |
| `cost_usd` | float | ? | |
| `prompt_id` | UUID | ? | The prompt template used, if any. |
| `lessons_learned` | [string] | ? | Explicit observations from the caller. |
| `metadata` | map[string, any] | ? | Additional context for future pattern matching. |

**Output DTO: `ExperienceRecordAck`**

| Field | Type | Description |
|-------|------|-------------|
| `experience_id` | UUID | |
| `product_id` | string | |
| `pattern_updated` | boolean | True if this experience updated an existing pattern. |
| `new_pattern_discovered` | boolean | True if this experience triggered new pattern discovery. |
| `recorded_at` | ISO8601 | |

---

### Method: `find_similar`

**Description:** Find past projects or executions with similar characteristics to the provided context.

**Input DTO: `SimilaritySearchRequest`**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `context` | map[string, any] | * | Context to match against. |
| `task_type` | string | ? | Restrict to this task type. |
| `product_id` | string | ? | Restrict to a specific product. |
| `min_similarity` | float [0.0, 1.0] | ? | Default: 0.30 (configurable; current implementation: Jaccard โฅ 0.08 โ€” too low, see improvement plan). |
| `max_results` | integer [1, 50] | ? | Default: 10 |
| `outcome_filter` | [enum] | ? | Restrict to specific outcomes (e.g., `[SUCCESS]` for success-only patterns). |

**Output DTO: `SimilaritySearchResult`**

| Field | Type | Description |
|-------|------|-------------|
| `results` | [SimilarProject] | Ordered by similarity score descending. |
| `algorithm` | enum(JACCARD, VECTOR, HYBRID) | Algorithm used for this search. |
| `search_duration_ms` | integer | |

---

### Method: `get_patterns`

**Description:** Return discovered patterns for a given domain. Patterns are distilled from many individual experiences and represent generalizable insights.

**Input DTO: `PatternQuery`**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `domain` | string | * | Domain to query (e.g., `"web_api"`, `"data_pipeline"`, `"game_design"`). |
| `task_type` | string | ? | Restrict to patterns relevant to this task type. |
| `min_evidence_count` | integer | ? | Minimum number of supporting experiences. Default: 3. |
| `max_results` | integer | ? | Default: 20 |

**Output DTO: `PatternQueryResult`**

| Field | Type | Description |
|-------|------|-------------|
| `patterns` | [Pattern] | |
| `total` | integer | |

**Pattern fields:** `pattern_id`, `domain`, `pattern_type` (enum: DO, AVOID, CONSIDER), `description` (string), `confidence` (float), `evidence_count` (integer), `first_discovered` (ISO8601), `last_updated` (ISO8601)

---

### Method: `propose_improvements`

**Description:** Request improvement proposals based on accumulated patterns and observed performance data. The Brain analyzes patterns to suggest concrete improvements to prompts, agent strategies, or workflow designs.

**Input DTO: `ImprovementProposalRequest`**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `product_id` | string | * | |
| `scope` | enum(PROMPTS, AGENT_STRATEGIES, WORKFLOW_DESIGNS, ALL) | ? | Default: ALL |
| `max_proposals` | integer | ? | Default: 5 |
| `min_confidence` | float | ? | Default: 0.7 |

**Output DTO: `ImprovementProposalResult`**

| Field | Type | Description |
|-------|------|-------------|
| `proposals` | [ImprovementProposal] | |

**ImprovementProposal fields:** `proposal_id`, `type` (enum: PROMPT_UPDATE, STRATEGY_CHANGE, WORKFLOW_REDESIGN), `description` (string), `expected_improvement` (string), `confidence` (float), `supporting_evidence_count` (integer), `affected_template_id` (UUID \| null)

---

## 5.5 Error Model

| Error Code | HTTP | Trigger |
|-----------|------|---------|
| `BRAIN_UNAVAILABLE` | 503 | Central Brain service not initialized |
| `BRAIN_CONTEXT_INVALID` | 400 | Context map is empty or contains invalid types |
| `BRAIN_EXPERIENCE_INCOMPLETE` | 400 | Experience record missing required fields |
| `BRAIN_NO_PATTERNS` | 200 (empty result) | No patterns found (not an error โ€” empty result) |
| `BRAIN_SEARCH_TIMEOUT` | 504 | Similarity search timed out (common at scale with O(nยฒ) Jaccard) |

## 5.6 Performance Expectations

| Operation | P50 target | P99 target | Notes |
|-----------|-----------|-----------|-------|
| `recommend` | < 100ms | < 500ms | Depends on pattern count |
| `record_experience` | < 30ms | < 100ms | Async internally |
| `find_similar` | < 200ms | < 2000ms | Degrades O(nยฒ) with Jaccard at scale |
| `get_patterns` | < 50ms | < 200ms | |
| `propose_improvements` | < 500ms | < 2000ms | AI-assisted analysis |

**Known limitation:** `find_similar` with Jaccard algorithm degrades significantly beyond 1,000 projects. Qdrant or pgvector migration is planned for v1.1 to achieve O(log n) performance.

## 5.7 Events Published

| Event | When |
|-------|------|
| `brain.experience.recorded` | Experience record accepted |
| `brain.pattern.discovered` | New pattern identified from accumulated experiences |
| `brain.pattern.updated` | Existing pattern updated with new evidence |

## 5.8 Events Consumed

| Event | Action |
|-------|--------|
| `workflows.workflow.completed` | Auto-record workflow completion experience |
| `workflows.task.failed` | Auto-record failure experience |
| `ai.execution.completed` | Update cost and performance patterns |

## 5.9 Future Extension Points

| Extension | Description | Target version |
|-----------|-------------|---------------|
| Vector search | Replace Jaccard with Qdrant/pgvector for semantic similarity | v1.1 |
| Cross-product learning | Brain explicitly learns from Content Factory and Mythic Realms | v1.1 |
| Experiment tracking | Track A/B test results as experiences | v1.2 |
| Auto-apply improvements | With approval, auto-create new prompt versions from proposals | v2.0 |

---



---

---

# 6. WorkspaceClient โ€” Workspace Registry Interface

**Stability Tier:** EXPERIMENTAL  
**SDK Package:** `aistudio.sdk.workspace`  
**Platform Module:** `aistudio.platform.workspace`  
**Contract Version:** 0.8 (EXPERIMENTAL โ€” interface evolves rapidly)

---

## 6.1 Purpose

WorkspaceClient provides a registry of all resources active in the current deployment โ€” registered products, available AI providers, platform module health, knowledge stores, and workspace repositories. It is the single source of truth for "what is available right now." The desktop Workspace panel consumes WorkspaceClient to render its live status view.

## 6.2 Responsibilities

WorkspaceClient is responsible for:
1. Returning the list of registered products and their health status
2. Returning the list of registered AI providers and their capabilities
3. Returning the current health of all platform modules
4. Returning active workspace repositories and their metadata
5. Broadcasting real-time workspace state changes via events

WorkspaceClient is **not** responsible for:
- Starting or stopping products or providers (that is the platform bootstrapper's responsibility)
- Configuring providers (that is the configuration layer's responsibility)
- Filesystem operations on repositories (that is git tooling's responsibility)

## 6.3 Ownership

**Platform Owner:** Platform team  
**Interface Owner:** Principal Engineer  
**Consumer Teams:** Desktop (WorkspacePanel); monitoring tools; product bootstrap checks

## 6.4 Methods

---

### Method: `get_workspace`

**Description:** Return the full workspace state snapshot โ€” products, providers, platform health, repositories.

**Input:** None (or optional `detail_level: enum(SUMMARY, FULL)`)

**Output DTO: `WorkspaceSnapshot`**

| Field | Type | Description |
|-------|------|-------------|
| `workspace_id` | string | From `workspace.yaml`. |
| `platform_version` | string | Running platform version. |
| `products` | [ProductRegistration] | All registered products. |
| `providers` | [ProviderStatus] | All registered AI providers with health. |
| `platform_modules` | [ModuleHealth] | Health of all platform modules. |
| `repositories` | [RepositoryInfo] | Workspace repositories from workspace.yaml. |
| `knowledge_stores` | [KnowledgeStoreStatus] | DB, vector, memory stores. |
| `snapshot_at` | ISO8601 | When this snapshot was taken. |
| `overall_health` | enum(HEALTHY, DEGRADED, UNHEALTHY) | Aggregate status. |

**ProductRegistration fields:** `product_id`, `product_name`, `version`, `status` (enum: RUNNING, STOPPED, ERROR), `api_prefix`, `capabilities: [string]`, `started_at`

**ModuleHealth fields:** `module_id`, `status` (enum: HEALTHY, DEGRADED, OFFLINE), `last_checked`, `message`

---

### Method: `get_product`

**Description:** Return the registration and live status of a specific product.

**Input:** `product_id: string`  
**Output:** `ProductRegistration` (extended with error details if status is ERROR)

---

### Method: `get_capabilities`

**Description:** Return the flat list of all capabilities available in this workspace. Used by consumers to determine what the platform can do before calling.

**Input:** `product_id: string | null` (if provided, restrict to capabilities of that product; if null, return all platform capabilities)  
**Output:** `CapabilityList { capabilities: [string], providers: [string], product_ids: [string] }`

---

### Method: `register_product`

**Description:** Register a product manifest with the workspace. Called by product startup code.

**Input DTO: `ProductManifest`**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `product_id` | string | * | Unique product identifier. |
| `product_name` | string | * | Human-readable name. |
| `version` | string | * | SemVer version of this product. |
| `platform_min_version` | string | * | Minimum platform version required. |
| `required_platform_modules` | [string] | * | Platform modules this product depends on. |
| `api_prefix` | string | * | This product's API route prefix. |
| `event_namespace` | string | * | NATS event namespace (e.g., `"aisf.*"`). |
| `capabilities` | [string] | * | Declared capabilities (e.g., `"product_creation"`). |

**Output:** `ProductRegistrationResult { product_id, registered_at, platform_version_compatible: boolean }`

---

### Method: `deregister_product`

**Description:** Remove a product from the workspace registry. Called by product shutdown code.

**Input:** `product_id: string`  
**Output:** `DeregistrationResult { product_id, deregistered_at }`

---

## 6.5 Error Model

| Error Code | HTTP | Trigger |
|-----------|------|---------|
| `WORKSPACE_PRODUCT_NOT_FOUND` | 404 | Product ID not registered |
| `WORKSPACE_PRODUCT_ALREADY_REGISTERED` | 409 | Product ID already registered (use `deregister` + `register`) |
| `WORKSPACE_PLATFORM_VERSION_INCOMPATIBLE` | 422 | Product requires higher platform version |
| `WORKSPACE_MODULE_UNAVAILABLE` | 503 | Required platform module not healthy |

## 6.6 Performance Expectations

`get_workspace` (full snapshot): P50 < 50ms, P99 < 200ms โ€” cached; refreshed every 30 seconds.  
`get_capabilities`: P50 < 5ms, P99 < 20ms โ€” in-memory.  
`register_product` / `deregister_product`: < 100ms โ€” called at startup/shutdown only.

## 6.7 Caching

`get_workspace` returns a cached snapshot. Cache TTL: 30 seconds. Force-refresh via `detail_level=FULL` or by calling with a `no_cache: true` flag. The desktop panel polls `get_workspace` every 30 seconds.

## 6.8 Events Published

| Event | When |
|-------|------|
| `workspace.product.registered` | New product registered |
| `workspace.product.deregistered` | Product removed |
| `workspace.health.degraded` | A platform module transitions to DEGRADED |
| `workspace.health.offline` | A platform module transitions to OFFLINE |

## 6.9 Future Extension Points

| Extension | Description | Target version |
|-----------|-------------|---------------|
| Multi-workspace | Multiple named workspaces in one deployment | v1.0 |
| Remote workspace | Connect to a remote platform instance | v1.1 |
| Workspace templates | Pre-defined workspace configurations | v1.2 |
| Resource quotas | Per-product resource limits enforced at workspace level | v2.0 |

---

---

# 7. ProviderClient โ€” AI Provider Adapter Interface

**Stability Tier:** STABLE  
**SDK Package:** N/A โ€” internal platform interface only (not exposed to products)  
**Platform Module:** `aistudio.platform.ai_ros.providers`  
**Contract Version:** 1.0

---

## 7.1 Purpose

ProviderClient is the abstract contract that every AI provider adapter must implement. It defines the boundary between the AI ROS (which routes, schedules, and tracks) and the AI providers (Anthropic, OpenAI, Gemini, Ollama, Claude Code, and future providers). Products never see this interface โ€” they use AIClient. Only AI ROS uses ProviderClient.

Adding a new provider requires implementing this interface and registering the implementation. Zero other changes are required.

## 7.2 Responsibilities

Each ProviderClient implementation is responsible for:
1. Translating `CompletionRequest` (normalized) to the provider's API format
2. Executing the provider API call
3. Translating the provider's response to `CompletionResponse` (normalized)
4. Implementing streaming that emits `CompletionChunk` objects
5. Implementing a health check (no side effects, no billing)
6. Returning static capability declarations at startup
7. Providing per-request cost estimates without API calls

ProviderClient is **not** responsible for:
- Retry logic (that is AI ROS's responsibility)
- Circuit breaking (that is AI ROS's responsibility)
- Cost tracking (that is AI ROS's responsibility)
- Caching (that is AI ROS's responsibility)

## 7.3 Ownership

**Platform Owner:** AI ROS team  
**Interface Owner:** Principal Engineer  
**Implementors:** Any engineer adding a new AI provider adapter

## 7.4 Methods

---

### Method: `complete`

**Description:** Execute a synchronous AI completion. Blocks until the provider returns.

**Input DTO: `CompletionRequest`** (AI ROS internal)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `messages` | [Message] | * | Full conversation history assembled by SessionManager. |
| `model` | string | * | Provider-specific model name (e.g., `"claude-sonnet-4-6"`). |
| `max_tokens` | integer | * | Maximum completion tokens. |
| `temperature` | float [0.0, 2.0] | ? | Default: 0.7 |
| `top_p` | float [0.0, 1.0] | ? | Default: 1.0 |
| `stop_sequences` | [string] | ? | |
| `tools` | [ToolDefinition] | ? | Tool schemas in the provider's expected format. |
| `stream` | boolean | * | Always false for `complete()`. |
| `timeout_seconds` | integer | * | Hard timeout. Provider must honor this. |
| `metadata` | map[string, string] | ? | `request_id`, `correlation_id` for provider-side logging. |

**Message DTO:**

| Field | Type | Description |
|-------|------|-------------|
| `role` | enum(user, assistant, system, tool) | |
| `content` | string \| [ContentBlock] | String for text-only; ContentBlock list for multi-modal. |
| `tool_call_id` | string \| null | For role=tool, the ID of the tool call this is responding to. |

**Output DTO: `CompletionResponse`** (AI ROS internal)

| Field | Type | Description |
|-------|------|-------------|
| `content` | string | Generated text. |
| `role` | string | Always `"assistant"`. |
| `stop_reason` | enum(END_TURN, MAX_TOKENS, STOP_SEQUENCE, TOOL_USE) | |
| `usage` | TokenUsage | `{ prompt_tokens, completion_tokens, cache_read_tokens, cache_write_tokens }` |
| `tool_calls` | [ToolCall] | Tool calls the model wants to make. |
| `model_used` | string | The exact model that responded. |
| `provider_latency_ms` | integer | Time for the provider API call only. |
| `raw_response` | map | The original provider response object. Stored for debugging. Never logged. |

---

### Method: `complete_async`

**Description:** Async version of `complete`. Required even if the provider's SDK is synchronous โ€” the adapter wraps it in `asyncio.to_thread`.

**Input / Output:** Same as `complete`.

---

### Method: `stream`

**Description:** Execute an AI completion and yield chunks as they arrive from the provider.

**Input:** Same as `CompletionRequest`.

**Output:** Async generator yielding `CompletionChunk`, then `CompletionStreamEnd`.

**CompletionChunk fields:** `chunk_index`, `delta` (string), `tool_call_delta` (ToolCallDelta \| null)  
**CompletionStreamEnd fields:** `stop_reason`, `usage`, `model_used`, `provider_latency_ms`

---

### Method: `health_check`

**Description:** Verify the provider is reachable and functional. Called every 30 seconds by the HealthMonitor. Must complete within 5 seconds. Must not bill the API account (use a minimal API check if provider supports it).

**Input:** None  
**Output DTO: `ProviderHealth`**

| Field | Type | Description |
|-------|------|-------------|
| `provider_id` | string | |
| `status` | enum(HEALTHY, DEGRADED, OFFLINE) | |
| `latency_ms` | integer | |
| `message` | string \| null | Human-readable status detail. |
| `checked_at` | ISO8601 | |

---

### Method: `get_capabilities`

**Description:** Return static capability declarations. Called once at startup. Must not make API calls.

**Output DTO: `ProviderCapabilities`**

| Field | Type | Description |
|-------|------|-------------|
| `provider_id` | string | Unique, stable provider identifier. |
| `display_name` | string | |
| `models` | [ModelInfo] | All supported models. |
| `supports_streaming` | boolean | |
| `supports_function_calling` | boolean | |
| `supports_vision` | boolean | |
| `supports_prompt_caching` | boolean | |
| `max_context_tokens` | integer | Across all supported models; per-model in `models`. |
| `max_output_tokens` | integer | |
| `supported_languages` | [string] | ISO 639-1 language codes. |

**ModelInfo fields:** `model_id`, `display_name`, `context_window`, `max_output_tokens`, `cost_per_1k_input_tokens`, `cost_per_1k_output_tokens`, `supports_function_calling`, `supports_vision`, `knowledge_cutoff`

---

### Method: `estimate_cost`

**Description:** Estimate the cost of a completion request. Must not make API calls. Uses token count estimates and published pricing.

**Input:** `CompletionRequest` (messages and max_tokens are the key inputs)  
**Output DTO: `ProviderCostEstimate`**

| Field | Type | Description |
|-------|------|-------------|
| `provider_id` | string | |
| `model_id` | string | |
| `prompt_tokens_estimated` | integer | |
| `completion_tokens_estimated` | integer | |
| `cost_usd_estimated` | float | |
| `pricing_basis` | string | Price per 1k tokens used for estimate (e.g., `"$3.00/1M input, $15.00/1M output"`). |

---

## 7.5 Error Handling Contract

Provider adapters translate provider-specific exceptions to the following normalized types before throwing. AI ROS catches these and decides retry/circuit-breaker behavior.

| Normalized exception | When to raise |
|---------------------|--------------|
| `ProviderTimeoutError` | Provider did not respond within `timeout_seconds` |
| `ProviderRateLimitError` | Provider returned HTTP 429; include `retry_after_seconds` if available |
| `ProviderServerError` | Provider returned HTTP 5xx |
| `ProviderAuthenticationError` | Provider returned HTTP 401 (API key issue) โ€” not retried |
| `ProviderCapabilityError` | Request uses a feature the model does not support |
| `ProviderContextLengthError` | Prompt exceeds model's context window |
| `ProviderNetworkError` | Connection-level failure (DNS, TCP) |

No provider-SDK-specific exception (`anthropic.APIError`, `openai.OpenAIError`) is thrown past the adapter boundary.

## 7.6 Reference Implementation: Claude Code Provider

| Attribute | Specification |
|-----------|--------------|
| `provider_id` | `"claude_code"` |
| Transport | subprocess call to `claude` CLI binary |
| `complete()` | `subprocess.run(["claude", ...args], capture_output=True, timeout=timeout_seconds)` |
| `complete_async()` | `asyncio.create_subprocess_exec(["claude", ...args])` with `asyncio.wait_for` |
| `stream()` | `asyncio.create_subprocess_exec`, read stdout line by line |
| `health_check()` | `subprocess.run(["claude", "--version"], timeout=5)` |
| `get_capabilities()` | Static โ€” defined from known Claude model specs |
| `estimate_cost()` | Tiktoken approximation + Anthropic published pricing |
| Binary discovery | `shutil.which("claude")` at startup; fail fast if not found |

## 7.7 Provider Registration Contract

Providers are registered in the ProviderRegistry at startup. The registry entry:

| Field | Type | Description |
|-------|------|-------------|
| `provider_id` | string | Must match `ProviderCapabilities.provider_id`. |
| `adapter` | ProviderClient | Instance of the adapter class. |
| `enabled` | boolean | From workspace.yaml provider config. |
| `priority` | integer | Used by LoadBalancer when capability scores are equal. |
| `health` | ProviderHealth | Updated by HealthMonitor every 30 seconds. |
| `circuit_breaker` | CircuitBreakerState | Per-provider circuit breaker state. |

## 7.8 Thread Safety

ProviderClient implementations must be thread-safe. Multiple concurrent `complete()` calls on the same adapter instance must not share state. The adapter holds no per-request state โ€” all state is in the `CompletionRequest` and `CompletionResponse`.

## 7.9 New Provider Checklist

A new provider is complete when:
- [ ] `complete()` returns correct `CompletionResponse` with all fields populated
- [ ] `complete_async()` returns the same result as `complete()` for identical input
- [ ] `stream()` yields chunks in order with correct `delta` accumulation
- [ ] `health_check()` returns OFFLINE when the provider binary/API is unavailable
- [ ] `health_check()` completes within 5 seconds under normal conditions
- [ ] `get_capabilities()` returns accurate pricing and context window for all declared models
- [ ] `estimate_cost()` estimates within ยฑ20% of actual cost for typical requests
- [ ] All provider-specific exceptions are wrapped in normalized exception types
- [ ] Unit tests cover: success, timeout, rate limit, server error, context length exceeded
- [ ] Integration test: end-to-end `complete()` against the real provider API (in a dedicated CI environment)

## 7.10 Future Extension Points

| Extension | Description | Target version |
|-----------|-------------|---------------|
| `embed()` method | Generate vector embeddings for Knowledge Graph | v1.1 |
| `moderate()` method | Content moderation check before completion | v1.2 |
| Multi-modal `complete()` | Accept image/audio in `ContentBlock` list | v1.2 |
| Provider-specific telemetry | Per-provider latency histograms | v1.1 |

---

---

# 8. SecurityClient โ€” Security Platform Interface

**Stability Tier:** STABLE  
**SDK Package:** `aistudio.sdk.security`  
**Platform Module:** `aistudio.platform.security`  
**Contract Version:** 1.0

---

## 8.1 Purpose

SecurityClient exposes the security platform to product code for per-operation permission checks, API key management, and audit log access. Most security enforcement happens automatically in SecurityMiddleware โ€” products only use SecurityClient when they need to check permissions programmatically or manage API keys.

## 8.2 Responsibilities

SecurityClient is responsible for:
1. Providing `require_permission()` as a FastAPI dependency factory for per-route RBAC
2. Providing programmatic permission checks for complex authorization logic
3. Managing API key lifecycle (create, list, revoke)
4. Providing access to security audit log entries
5. Exposing rate limit status for monitoring

SecurityClient is **not** responsible for:
- Issuing JWT tokens (that is the SecurityMiddleware's responsibility at login)
- Enforcing authentication (that is SecurityMiddleware's responsibility)
- Storing API keys in plaintext (keys are stored as SHA-256 hashes only)

## 8.3 Ownership

**Platform Owner:** Security team  
**Interface Owner:** Chief Architect  
**Consumer Teams:** All products (for `require_permission()` in routes); Admin users (for API key management)

## 8.4 Methods

---

### Method: `require_permission` (FastAPI dependency factory)

**Description:** Returns a FastAPI `Depends()` that enforces the specified permission. The dependency reads `AuthContext` from `request.state` (set by SecurityMiddleware) and raises `HTTP 403` if the permission is not satisfied.

This is not a method call โ€” it is a decorator/dependency used in route definitions.

**Usage pattern:**
```
@router.post("/workflows")
async def create_workflow(
    _: AuthContext = Depends(require_permission(Permission.WORKFLOW_CREATE))
)
```

**Input:** `permission: Permission` (enum value)  
**Output:** The `AuthContext` of the authenticated caller (if permission is granted)  
**On denial:** Raises `HTTP 403` with error code `AUTH_PERMISSION_DENIED`

---

### Method: `check_permission`

**Description:** Programmatic permission check. Use when authorization logic is more complex than a single permission (e.g., check permission AND ownership of the resource).

**Input DTO: `PermissionCheckRequest`**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `identity` | string | * | The identity to check (from AuthContext). |
| `permission` | Permission | * | The permission to verify. |
| `resource_id` | UUID | ? | If provided, also checks resource-level ownership where applicable. |

**Output DTO: `PermissionCheckResult`**

| Field | Type | Description |
|-------|------|-------------|
| `identity` | string | |
| `permission` | Permission | |
| `granted` | boolean | |
| `role` | Role | The role that granted (or denied) this permission. |
| `denial_reason` | string \| null | Human-readable reason if denied. |

---

### Method: `create_api_key`

**Description:** Create a new API key. The plaintext key is returned exactly once โ€” it is never retrievable again.

**Input DTO: `CreateApiKeyRequest`**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `label` | string | * | Human-readable label (e.g., `"CI/CD pipeline key"`). |
| `role` | Role | * | Role to assign to this key. |
| `expires_at` | ISO8601 | ? | Expiry date. Default: never expires. |
| `allowed_ips` | [string] | ? | CIDR blocks. If set, only requests from these IPs are authenticated with this key. |
| `rate_limit_per_minute` | integer | ? | Override the platform default rate limit for this key. |

**Output DTO: `ApiKeyCreationResult`**

| Field | Type | Description |
|-------|------|-------------|
| `key_id` | UUID | Stable identifier for this key. Use for revocation. |
| `api_key` | string | The plaintext key. **Shown once. Never retrievable again.** |
| `label` | string | |
| `role` | Role | |
| `created_at` | ISO8601 | |
| `expires_at` | ISO8601 \| null | |

---

### Method: `list_api_keys`

**Description:** List all API keys (without plaintext โ€” only metadata).

**Output DTO: `ApiKeyList { items: [ApiKeyMetadata], total: integer }`**

**ApiKeyMetadata fields:** `key_id`, `label`, `role`, `created_at`, `expires_at`, `last_used_at`, `status` (enum: ACTIVE, EXPIRED, REVOKED), `allowed_ips`

---

### Method: `revoke_api_key`

**Description:** Immediately revoke an API key. Revoked keys fail authentication instantly.

**Input:** `key_id: UUID`, `reason: string`  
**Output:** `RevokeResult { key_id, revoked_at, reason }`

---

### Method: `get_rate_limit_status`

**Description:** Return current rate limit consumption for an identity or API key.

**Input:** `identity: string | null` (null returns current caller's status from AuthContext)  
**Output DTO: `RateLimitStatus`**

| Field | Type | Description |
|-------|------|-------------|
| `identity` | string | |
| `requests_this_minute` | integer | |
| `limit_per_minute` | integer | |
| `requests_remaining` | integer | |
| `reset_at` | ISO8601 | When the current window resets. |
| `is_throttled` | boolean | True if currently rate-limited. |

---

### Method: `get_audit_log`

**Description:** Retrieve security audit log entries. Restricted to ADMIN role.

**Input DTO: `AuditLogQuery`**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `identity` | string | ? | Filter by specific identity. |
| `event_type` | [string] | ? | Filter by event types (e.g., `["AUTH_FAILED", "PERMISSION_DENIED"]`). |
| `after` | ISO8601 | ? | |
| `before` | ISO8601 | ? | |
| `page_size` | integer | ? | Default: 50, max: 500 |
| `page_cursor` | string | ? | |

**Output DTO: `AuditLogResult { items: [AuditEvent], total, next_cursor, has_more }`**

**AuditEvent fields:** `event_id`, `event_type`, `identity`, `ip_address`, `resource_id`, `outcome` (enum: SUCCESS, FAILURE), `timestamp`, `details` (map)

---

## 8.5 Permission Catalog

| Permission | Description | Minimum Role |
|-----------|-------------|-------------|
| `AI_READ` | Read AI execution metadata | VIEWER |
| `AI_EXECUTE` | Execute AI completions | DEVELOPER |
| `WORKFLOW_READ` | Read workflow status | VIEWER |
| `WORKFLOW_CREATE` | Submit workflow plans | DEVELOPER |
| `WORKFLOW_MANAGE` | Cancel, pause, resume workflows | DEVELOPER |
| `WORKFLOW_APPROVE` | Approve human gates | REVIEWER |
| `PROMPT_READ` | Read and render prompt templates | VIEWER |
| `PROMPT_MANAGE` | Register, update, transition templates | DEVELOPER |
| `BRAIN_READ` | Request recommendations, search patterns | VIEWER |
| `BRAIN_RECORD` | Record experiences, update patterns | DEVELOPER |
| `KNOWLEDGE_READ` | Query knowledge graph | VIEWER |
| `KNOWLEDGE_WRITE` | Add/update knowledge graph nodes | DEVELOPER |
| `SECURITY_READ` | View audit log, rate limit status | REVIEWER |
| `SECURITY_MANAGE` | Create/revoke API keys | ADMIN |
| `WORKSPACE_READ` | View workspace snapshot | VIEWER |
| `PLUGIN_READ` | List and view plugins | VIEWER |
| `PLUGIN_INSTALL` | Install plugins | ADMIN |
| `PLUGIN_EXECUTE` | Execute installed plugins | DEVELOPER |
| `ADMIN` | All permissions | ADMIN |

## 8.6 Role Hierarchy

```
ADMIN
  โ”โ”€โ”€ All REVIEWER permissions
  โ”โ”€โ”€ SECURITY_MANAGE
  โ””โ”€โ”€ PLUGIN_INSTALL

REVIEWER
  โ”โ”€โ”€ All DEVELOPER permissions
  โ”โ”€โ”€ WORKFLOW_APPROVE
  โ””โ”€โ”€ SECURITY_READ

DEVELOPER
  โ”โ”€โ”€ All VIEWER permissions
  โ”โ”€โ”€ AI_EXECUTE
  โ”โ”€โ”€ WORKFLOW_CREATE
  โ”โ”€โ”€ WORKFLOW_MANAGE
  โ”โ”€โ”€ PROMPT_MANAGE
  โ”โ”€โ”€ BRAIN_RECORD
  โ”โ”€โ”€ KNOWLEDGE_WRITE
  โ””โ”€โ”€ PLUGIN_EXECUTE

VIEWER
  โ”โ”€โ”€ AI_READ
  โ”โ”€โ”€ WORKFLOW_READ
  โ”โ”€โ”€ PROMPT_READ
  โ”โ”€โ”€ BRAIN_READ
  โ”โ”€โ”€ KNOWLEDGE_READ
  โ””โ”€โ”€ WORKSPACE_READ
```

## 8.7 Error Model

| Error Code | HTTP | Trigger |
|-----------|------|---------|
| `AUTH_MISSING_CREDENTIALS` | 401 | No API key or JWT in request |
| `AUTH_INVALID_API_KEY` | 401 | API key not found or hash mismatch |
| `AUTH_EXPIRED_API_KEY` | 401 | API key past its expiry date |
| `AUTH_REVOKED_API_KEY` | 401 | API key has been revoked |
| `AUTH_INVALID_JWT` | 401 | JWT signature invalid or malformed |
| `AUTH_EXPIRED_JWT` | 401 | JWT past its expiry |
| `AUTH_PERMISSION_DENIED` | 403 | Valid credentials but insufficient role |
| `AUTH_IP_NOT_ALLOWED` | 403 | Caller IP not in API key's allowed_ips |
| `AUTH_RATE_LIMIT_EXCEEDED` | 429 | Caller exceeded per-minute request limit |
| `AUTH_API_KEY_NOT_FOUND` | 404 | key_id does not exist (for management operations) |
| `AUTH_MISSING_CONTEXT` | 500 | AuthContext not populated (middleware misconfiguration) |

## 8.8 Authentication Context DTO

The `AuthContext` is the output of SecurityMiddleware. It is injected into `request.state.auth_context` and available to all route handlers and services.

| Field | Type | Description |
|-------|------|-------------|
| `identity` | string | Stable identity string. For API keys: the key_id as string. For JWT: the `sub` claim. |
| `role` | Role | Enum: VIEWER, DEVELOPER, REVIEWER, ADMIN |
| `method` | enum(API_KEY, JWT, DEV) | How the caller authenticated. DEV mode only on local environment. |
| `is_admin` | boolean | Convenience flag. Equivalent to `role == ADMIN`. |
| `permissions` | frozenset[Permission] | Effective permissions for this role. |
| `product_id` | string \| null | If the identity is a product service account, its product_id. |
| `authenticated_at` | ISO8601 | Time of this authentication. |
| `ip_address` | string | Caller IP address. |

## 8.9 SecurityMiddleware Request Flow

```
HTTP Request arrives
        โ”
        โ–ผ
SecurityMiddleware
        โ”
        โ”โ”€โ”€ path in PUBLIC_PATHS? โ’ pass through (no AuthContext set)
        โ”
        โ”โ”€โ”€ Extract credentials:
        โ”       X-API-Key header present? โ’ ApiKey path
        โ”       Authorization: Bearer present? โ’ JWT path
        โ”       Neither? โ’ AUTH_MISSING_CREDENTIALS (401)
        โ”
        โ”โ”€โ”€ [ApiKey path]
        โ”       hash = SHA-256(provided_key)
        โ”       stored_hash = db.api_keys.get(hash)
        โ”       hmac.compare_digest(hash, stored_hash)? โ’ AuthContext(role from db)
        โ”       No match? โ’ AUTH_INVALID_API_KEY (401)
        โ”
        โ”โ”€โ”€ [JWT path]
        โ”       jwt.decode(token, secret, algorithms=["HS256"])
        โ”       signature valid and not expired? โ’ AuthContext(role from claims)
        โ”       Invalid? โ’ AUTH_INVALID_JWT (401)
        โ”
        โ”โ”€โ”€ RateLimiter.check(identity, endpoint)?
        โ”       Exceeded? โ’ AUTH_RATE_LIMIT_EXCEEDED (429)
        โ”
        โ”โ”€โ”€ request.state.auth_context = AuthContext(...)
        โ”
        โ–ผ
CORSMiddleware โ’ Route Handler
```

## 8.10 Events Published

| Event | When |
|-------|------|
| `security.auth.failed` | Authentication failure |
| `security.auth.succeeded` | Authentication success (sampled โ€” not every request) |
| `security.permission.denied` | Authorization failure |
| `security.rate_limit.exceeded` | Rate limit triggered |
| `security.api_key.created` | New API key created |
| `security.api_key.revoked` | API key revoked |

## 8.11 Future Extension Points

| Extension | Description | Target version |
|-----------|-------------|---------------|
| JWT issuance endpoint | Login endpoint that issues JWT from credentials | v1.1 |
| OAuth 2.0 / OIDC | Enterprise SSO integration | v2.0 |
| SAML 2.0 | Enterprise SSO via SAML | v2.0 |
| Fine-grained resource RBAC | `require_permission(WORKFLOW_READ, resource_id=workflow_id)` | v1.2 |
| Multi-tenancy | tenant_id in AuthContext | v2.0 |

---

---

# 9. KnowledgeClient โ€” Knowledge Graph Interface

**Stability Tier:** BETA  
**SDK Package:** `aistudio.sdk.knowledge`  
**Platform Module:** `aistudio.platform.knowledge`  
**Contract Version:** 0.9 (BETA)

---

## 9.1 Purpose

KnowledgeClient provides access to the platform's structured knowledge graph. Where BrainClient provides high-level intelligence recommendations, KnowledgeClient provides direct access to the graph structure โ€” nodes, relationships, and queries. KnowledgeClient is the lower-level interface for products that need to contribute to or query the knowledge graph directly.

## 9.2 Responsibilities

KnowledgeClient is responsible for:
1. Adding and updating knowledge nodes (concepts, entities, facts)
2. Adding relationships between nodes
3. Querying the graph for paths, neighbors, and subgraphs
4. Providing full-text and attribute search across nodes
5. Retrieving memory records (cross-session persistent facts)

KnowledgeClient is **not** responsible for:
- Pattern discovery (that is BrainClient's responsibility)
- AI-powered reasoning over the graph (that is BrainClient's responsibility)
- AI execution (that is AIClient's responsibility)

## 9.3 Ownership

**Platform Owner:** Knowledge team  
**Interface Owner:** Principal Engineer  
**Consumer Teams:** Products that contribute domain-specific knowledge; BrainClient (internal consumer)

## 9.4 Methods

---

### Method: `add_node`

**Description:** Add a knowledge node to the graph. If a node with the same `node_id` already exists, it is updated.

**Input DTO: `AddNodeRequest`**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `node_id` | string | ? | Caller-assigned stable ID. Auto-generated if not provided. |
| `node_type` | string | * | Semantic type (e.g., `"project"`, `"pattern"`, `"technology"`, `"outcome"`). |
| `label` | string | * | Human-readable label. |
| `attributes` | map[string, any] | ? | Typed key-value attributes specific to this node type. |
| `product_id` | string | ? | Product that created this node. |
| `embedding` | [float] | ? | Vector embedding. Required if vector search is enabled. |

**Output DTO: `NodeResult`** โ€” `{ node_id, node_type, label, attributes, created_at, updated_at }`

---

### Method: `add_relationship`

**Description:** Create a directed relationship between two nodes.

**Input DTO: `AddRelationshipRequest`**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `from_node_id` | string | * | |
| `to_node_id` | string | * | |
| `relationship_type` | string | * | Semantic label (e.g., `"USES_TECHNOLOGY"`, `"CAUSED_BY"`, `"SIMILAR_TO"`). |
| `weight` | float [0.0, 1.0] | ? | Relationship strength. Default: 1.0 |
| `attributes` | map[string, any] | ? | |

**Output DTO: `RelationshipResult`** โ€” `{ relationship_id, from_node_id, to_node_id, relationship_type, weight, created_at }`

---

### Method: `query`

**Description:** Execute a structured graph query. The query language is a simplified path expression โ€” not full Cypher or SPARQL, which would create a vendor dependency.

**Input DTO: `GraphQuery`**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `start_node_id` | string | ? | Start traversal from this node. |
| `node_type` | string | ? | Filter results to this node type. |
| `relationship_type` | string | ? | Follow only relationships of this type. |
| `direction` | enum(OUTGOING, INCOMING, BOTH) | ? | Default: OUTGOING |
| `max_depth` | integer [1, 5] | ? | Default: 1 |
| `attribute_filters` | [AttributeFilter] | ? | Filter results by attribute values. |
| `max_results` | integer [1, 100] | ? | Default: 20 |

**AttributeFilter:** `{ attribute, operator: enum(EQ, GT, LT, CONTAINS), value }`

**Output DTO: `GraphQueryResult`**

| Field | Type | Description |
|-------|------|-------------|
| `nodes` | [NodeResult] | Nodes matching the query. |
| `relationships` | [RelationshipResult] | Relationships traversed. |
| `paths` | [GraphPath] | Paths from start_node_id to each result node. |
| `query_duration_ms` | integer | |

---

### Method: `search`

**Description:** Full-text and attribute search across all knowledge nodes.

**Input DTO: `KnowledgeSearchRequest`**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `query` | string | * | Full-text search query. |
| `node_types` | [string] | ? | Restrict to these node types. |
| `product_id` | string | ? | Restrict to this product's nodes. |
| `max_results` | integer | ? | Default: 20 |

**Output DTO: `KnowledgeSearchResult { results: [NodeResult], total, search_duration_ms }`**

---

### Method: `get_memory`

**Description:** Retrieve persistent memory records for a product or session. Memory records are facts that survive across sessions.

**Input DTO: `MemoryQuery`**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `product_id` | string | * | |
| `session_id` | UUID | ? | Restrict to memories from a specific session. |
| `key` | string | ? | Exact memory key lookup. |
| `tag` | string | ? | Filter by memory tag. |
| `max_results` | integer | ? | Default: 50 |

**Output DTO: `MemoryQueryResult { memories: [MemoryRecord], total }`**

**MemoryRecord fields:** `memory_id`, `product_id`, `session_id`, `key`, `value` (any), `tags: [string]`, `created_at`, `updated_at`, `expires_at`

---

### Method: `set_memory`

**Description:** Create or update a persistent memory record.

**Input DTO: `SetMemoryRequest`**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `product_id` | string | * | |
| `session_id` | UUID | ? | |
| `key` | string | * | Stable key for this memory. |
| `value` | any | * | The value to store. |
| `tags` | [string] | ? | Classification tags. |
| `expires_at` | ISO8601 | ? | TTL for this memory record. |

**Output DTO: `MemorySetResult { memory_id, key, updated_at }`**

---

## 9.5 Error Model

| Error Code | HTTP | Trigger |
|-----------|------|---------|
| `KNOWLEDGE_NODE_NOT_FOUND` | 404 | Node ID does not exist |
| `KNOWLEDGE_RELATIONSHIP_NOT_FOUND` | 404 | Relationship does not exist |
| `KNOWLEDGE_QUERY_DEPTH_EXCEEDED` | 400 | max_depth > 5 |
| `KNOWLEDGE_QUERY_TIMEOUT` | 504 | Graph query exceeded timeout |
| `KNOWLEDGE_MEMORY_NOT_FOUND` | 404 | Memory key does not exist |
| `KNOWLEDGE_STORE_UNAVAILABLE` | 503 | Underlying graph store not accessible |

## 9.6 Performance Expectations

| Operation | P50 target | P99 target | Notes |
|-----------|-----------|-----------|-------|
| `add_node` | < 30ms | < 100ms | |
| `add_relationship` | < 30ms | < 100ms | |
| `query` (depth=1) | < 50ms | < 200ms | |
| `query` (depth=3) | < 200ms | < 1000ms | Degrades with depth ร— node degree |
| `search` | < 100ms | < 500ms | |
| `get_memory` / `set_memory` | < 20ms | < 80ms | |

## 9.7 Storage Backends

| Backend | Status | Use case |
|---------|--------|---------|
| SQL (SQLAlchemy + adjacency list tables) | CURRENT | Default; no additional infrastructure |
| Kuzu (embedded graph DB) | PLANNED v1.1 | Better graph traversal performance |
| Neo4j | PLANNED v2.0 | Full enterprise graph queries |

The interface is stable regardless of the backend. Backend changes are transparent to KnowledgeClient consumers.

## 9.8 Events Published

| Event | When |
|-------|------|
| `knowledge.node.added` | Node created |
| `knowledge.node.updated` | Node attributes updated |
| `knowledge.relationship.added` | Relationship created |
| `knowledge.pattern.indexed` | Pattern indexed into knowledge graph |

---

---

# 10. EventBus โ€” Platform Messaging Interface

**Stability Tier:** STABLE  
**SDK Package:** `aistudio.sdk.messaging`  
**Platform Module:** `aistudio.platform.messaging`  
**Contract Version:** 1.0

---

## 10.1 Purpose

EventBus is the platform's messaging backbone. It provides two transports: in-process (for tight inner loops within a single deployment) and NATS JetStream (for durable, at-least-once delivery across services). Products and platform modules use EventBus to publish events and subscribe to them. No module communicates directly with NATS โ€” all NATS access goes through EventBus.

## 10.2 Responsibilities

EventBus is responsible for:
1. Publishing platform events to the in-process bus and NATS JetStream
2. Dispatching events to registered in-process subscribers
3. Managing NATS stream configuration (creating streams if not present)
4. Providing at-least-once delivery guarantees for events published to NATS
5. Providing real-time event delivery to WebSocket subscribers for desktop live updates
6. Managing consumer groups and acknowledgment

EventBus is **not** responsible for:
- Event schema validation (callers provide typed event models)
- Dead-letter queue handling for business logic (that is the subscriber's responsibility)
- Event sourcing or event store capabilities (NATS JetStream provides this)

## 10.3 Ownership

**Platform Owner:** Messaging team  
**Interface Owner:** Principal Engineer  
**Consumer Teams:** All platform modules; all products; desktop (via WebSocket)

## 10.4 Methods

---

### Method: `publish`

**Description:** Publish an event to the platform event bus. Delivers to in-process subscribers immediately. If NATS is enabled, also publishes to the appropriate JetStream.

**Input DTO: `PublishRequest`**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `event` | PlatformEvent | * | The event to publish. |
| `durable` | boolean | ? | Default: true. If true, event is also published to NATS JetStream. If false, in-process only. |

**PlatformEvent DTO:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `event_id` | UUID | * | Auto-generated. |
| `event_type` | string | * | Full NATS subject (e.g., `"workflows.task.completed"`). Must match the catalog. |
| `source` | string | * | Publishing module path (e.g., `"platform/workflow-runtime"`). |
| `correlation_id` | UUID | ? | Propagated from the triggering request. |
| `causation_id` | UUID | ? | The event_id of the event that caused this event. |
| `timestamp` | ISO8601 | * | Auto-populated. |
| `schema_version` | string | * | Payload schema version (e.g., `"1.0"`). |
| `payload` | map[string, any] | * | Event-specific payload (validated against the event type's schema). |

**Output DTO: `PublishResult`**

| Field | Type | Description |
|-------|------|-------------|
| `event_id` | UUID | |
| `delivered_in_process` | integer | Number of in-process subscribers notified. |
| `nats_sequence` | integer \| null | NATS JetStream sequence number. Null if NATS disabled or `durable=false`. |
| `published_at` | ISO8601 | |

---

### Method: `subscribe`

**Description:** Register an in-process subscriber for events matching a subject pattern. The handler is called synchronously in the publish call for in-process events; NATS events are delivered to the handler by a background consumer.

**Input DTO: `SubscribeRequest`**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `subject_pattern` | string | * | NATS-style wildcard pattern. `"workflows.task.*"` matches any task event. `">"` matches all events. |
| `handler` | EventHandler | * | Async callable that receives `PlatformEvent`. |
| `consumer_group` | string | ? | NATS durable consumer name. If set, events are delivered once per group, not once per instance. Required for NATS subscriptions. |
| `ack_timeout_seconds` | integer | ? | NATS ack timeout. Default: 30 |
| `deliver_from` | enum(NEW, LAST, FIRST, SEQUENCE) | ? | Where to start consuming from NATS JetStream. Default: NEW. |
| `start_sequence` | integer | ? | Required if `deliver_from=SEQUENCE`. |

**Output DTO: `SubscriptionHandle`**

| Field | Type | Description |
|-------|------|-------------|
| `subscription_id` | UUID | Use to unsubscribe. |
| `subject_pattern` | string | |
| `consumer_group` | string \| null | |
| `subscribed_at` | ISO8601 | |

---

### Method: `unsubscribe`

**Description:** Remove an in-process or NATS subscription.

**Input:** `subscription_id: UUID`  
**Output:** `UnsubscribeResult { subscription_id, unsubscribed_at }`

---

### Method: `get_stream_status`

**Description:** Return the current status of all NATS JetStream streams. Used by monitoring and health checks.

**Output DTO: `StreamStatusResult`**

| Field | Type | Description |
|-------|------|-------------|
| `nats_connected` | boolean | |
| `streams` | [StreamStatus] | One per defined stream. |
| `retrieved_at` | ISO8601 | |

**StreamStatus fields:** `stream_name`, `subjects: [string]`, `message_count`, `byte_count`, `consumer_count`, `last_sequence`, `created_at`

---

### Method: `replay`

**Description:** Replay events from NATS JetStream history for a subscriber. Used for event sourcing and crash recovery.

**Input DTO: `ReplayRequest`**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `subject_pattern` | string | * | |
| `from_sequence` | integer | ? | Start from this sequence number. Default: beginning. |
| `from_timestamp` | ISO8601 | ? | Start from this timestamp. |
| `handler` | EventHandler | * | Handler to receive replayed events. |
| `max_events` | integer | ? | Default: 1000 |

**Output DTO: `ReplayResult { events_replayed, last_sequence, completed_at }`**

---

## 10.5 NATS Stream Configuration

The following streams are pre-configured. All subjects listed must be published to the correct stream.

| Stream Name | Subjects | Retention | Max Age |
|-------------|---------|-----------|---------|
| WORKFLOWS | `workflows.>` | limits | 7 days |
| AI | `ai.>` | limits | 3 days |
| PROMPTS | `prompts.>` | limits | 30 days |
| BRAIN | `brain.>` | limits | 30 days |
| SECURITY | `security.>` | limits | 90 days |
| WORKSPACE | `workspace.>` | limits | 1 day |
| KNOWLEDGE | `knowledge.>` | limits | 30 days |

**Critical note:** The subject prefix `workflows.>` (plural, with period, with wildcard) is the correct NATS subject filter. Prior bug used `workflow.*` (singular) โ€” this is fixed in the stream configuration. Any code publishing to `workflow.*` (singular) will not be delivered to the WORKFLOWS stream.

## 10.6 Error Model

| Error Code | HTTP | Trigger |
|-----------|------|---------|
| `MESSAGING_NATS_UNAVAILABLE` | 503 | NATS server not reachable |
| `MESSAGING_UNKNOWN_SUBJECT` | 400 | Event type not in catalog |
| `MESSAGING_SUBSCRIPTION_NOT_FOUND` | 404 | Subscription ID not registered |
| `MESSAGING_CONSUMER_GROUP_REQUIRED` | 400 | NATS subscription without consumer_group |
| `MESSAGING_PUBLISH_TIMEOUT` | 504 | NATS publish ACK not received within timeout |

## 10.7 Thread Safety

EventBus is fully thread-safe and async-safe. In-process subscriber handlers are called in the same coroutine context as the publisher. Handlers must not block โ€” they must be `async def` and complete within 100ms. Long-running handlers must offload work to a background task.

## 10.8 Delivery Guarantees

| Scenario | Guarantee |
|----------|-----------|
| In-process publish (NATS disabled) | At-most-once |
| In-process publish (NATS enabled, `durable=true`) | At-least-once via NATS |
| NATS consumer group subscription | At-least-once per group (one delivery per group) |
| In-process subscriber during crash | No delivery (in-memory only) |
| NATS subscriber after crash (consumer_group set) | Delivery on reconnect |

## 10.9 WebSocket Delivery

EventBus bridges events to connected WebSocket clients (desktop) via the WebSocket manager. All events published to the in-process bus are also delivered to WebSocket subscribers. WebSocket delivery is best-effort (connected clients only, no replay on disconnect).

## 10.10 Sequence Diagram

```
Publisher              EventBus           In-Process Sub   NATS JetStream
    โ”                      โ”                     โ”               โ”
    โ”โ”€โ”€ publish(event) โ”€โ”€โ–บ โ”                     โ”               โ”
    โ”                      โ”โ”€โ”€ dispatch(event) โ”€โ”€โ–บโ”               โ”
    โ”                      โ”   (sync, inline)    โ”               โ”
    โ”                      โ”                     โ”               โ”
    โ”                      โ”โ”€โ”€ nats.publish(subj, payload) โ”€โ”€โ”€โ”€โ”€โ–บโ”
    โ”                      โ” โ—โ”€โ”€ ACK (seq=1234) โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€ โ”
    โ” โ—โ”€โ”€ PublishResult โ”€โ”€ โ”                     โ”               โ”
    โ”                      โ”                     โ”               โ”
    โ”                      โ”     [NATS consumer loop]            โ”
    โ”                      โ”โ—โ”€โ”€ delivery(event) โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€ โ”
    โ”                      โ”โ”€โ”€ dispatch to NATS subscribers โ”€โ”€โ–บ โ” (consumer groups)
```

## 10.11 State Diagram

```
NATS CONNECTION STATES

  startup
    โ”
    โ–ผ
  โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
  โ” CONNECTING   โ” โ”€โ”€โ”€โ”€ timeout/error โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ–บ OFFLINE
  โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”ฌโ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”                                       โ”
         โ” connected                                      โ” retry (30s interval)
         โ–ผ                                               โ”
  โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”                                       โ”
  โ”   CONNECTED  โ” โ—โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
  โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”ฌโ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
         โ” network error
         โ–ผ
  โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
  โ” RECONNECTING โ” โ”€โ”€โ”€โ”€ max_retries exceeded โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ–บ OFFLINE
  โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”ฌโ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
         โ” reconnected
         โ–ผ
  โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
  โ”   CONNECTED  โ”
  โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”

Note: OFFLINE state โ’ in-process bus still works; durable publishes buffered in memory
      (max buffer: 10,000 events). Buffer overflow drops oldest events with WARNING log.
```

## 10.12 Events Published

EventBus itself publishes:
| Event | When |
|-------|------|
| `workspace.health.degraded` | NATS disconnects (platform degraded to in-process only) |
| `workspace.health.offline` | NATS connection lost entirely |

## 10.13 Future Extension Points

| Extension | Description | Target version |
|-----------|-------------|---------------|
| Event schema registry | Validate payloads against registered schemas on publish | v1.1 |
| Dead-letter queue API | Expose DLQ messages for manual inspection/replay | v1.1 |
| WebSocket subscription filtering | Desktop subscribes to specific subjects, not all events | v1.1 |
| Multi-region NATS | Cross-datacenter event delivery | v2.0 |

---

---

# 11. PluginAPI โ€” Plugin Extension Interface

**Stability Tier:** EXPERIMENTAL  
**SDK Package:** `aistudio.sdk.plugin`  
**Platform Module:** `aistudio.platform.plugin_runtime`  
**Contract Version:** 0.7 (EXPERIMENTAL)

---

## 11.1 Purpose

PluginAPI defines the contract between the AI Studio platform and external plugins. Plugins extend the platform's capabilities without modifying platform code. The initial plugin types are Worker plugins (extend the agent pool with new worker implementations) and Tool plugins (add new tools to the ToolRuntime).

## 11.2 Plugin Types

| Type | Description | What it contributes |
|------|-------------|-------------------|
| `worker` | A new agent worker implementation | New task types that the WorkflowRuntime can dispatch |
| `tool` | A new tool for agents to call | New tool available in `ExecutionRequest.tools` |
| `prompt` | A bundle of registered prompt templates | Templates registered in Prompt OS on install |
| `knowledge` | Domain-specific knowledge nodes | Nodes added to KnowledgeClient graph on install |

## 11.3 Ownership

**Platform Owner:** Platform team  
**Interface Owner:** Principal Engineer  
**Consumer Teams:** Product teams adding new task types; community plugin authors (future marketplace)

## 11.4 Plugin Manifest

Every plugin provides a manifest that declares its contract:

**PluginManifest DTO:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `plugin_id` | string | * | Unique, stable plugin identifier (e.g., `"aisf/java-worker"`). |
| `plugin_name` | string | * | Human-readable name. |
| `version` | string | * | SemVer plugin version. |
| `plugin_type` | enum(worker, tool, prompt, knowledge) | * | |
| `platform_min_version` | string | * | Minimum platform version. |
| `sdk_version` | string | * | Plugin SDK version used. |
| `author` | string | * | |
| `description` | string | * | |
| `capabilities` | [string] | * | What this plugin adds to the platform. |
| `permissions_required` | [Permission] | * | Permissions the plugin needs to call platform APIs. |
| `configuration_schema` | map | ? | JSON Schema for plugin configuration. |

---

## 11.5 Methods (Platform manages plugins)

---

### Method: `install`

**Description:** Install a plugin into the platform. Admin only. Validates the manifest, verifies compatibility, executes the plugin's `on_install` hook.

**Input:** `plugin_manifest: PluginManifest`, `configuration: map[string, any]`  
**Output DTO: `PluginInstallResult`** โ€” `{ plugin_id, version, installed_at, status: enum(INSTALLED, FAILED), message }`

---

### Method: `uninstall`

**Description:** Remove an installed plugin. Executes the plugin's `on_uninstall` hook first.

**Input:** `plugin_id: string`  
**Output DTO: `PluginUninstallResult`** โ€” `{ plugin_id, uninstalled_at }`

---

### Method: `list`

**Description:** List all installed plugins with their status.

**Output DTO: `PluginList { plugins: [InstalledPlugin], total }`**

**InstalledPlugin fields:** `plugin_id`, `plugin_name`, `version`, `plugin_type`, `status` (enum: ACTIVE, DISABLED, ERROR), `installed_at`, `capabilities`

---

### Method: `get`

**Description:** Return details and current status of a specific installed plugin.

**Input:** `plugin_id: string`  
**Output DTO: `InstalledPlugin`** (extended with configuration and error details)

---

### Method: `enable` / `disable`

**Description:** Enable or disable an installed plugin without uninstalling it.

**Input:** `plugin_id: string`  
**Output:** `PluginStatusResult { plugin_id, status, updated_at }`

---

## 11.6 Worker Plugin Contract

Worker plugins extend the agent pool. A worker plugin must implement:

| Method | Description |
|--------|-------------|
| `on_install(config)` | Initialize resources needed by the worker. |
| `on_uninstall()` | Clean up resources. |
| `get_task_types()` โ’ [string] | Declare task types this worker can handle. |
| `can_handle(task: TaskDefinition)` โ’ boolean | Runtime check for specific task. |
| `execute(task: TaskDefinition, ai_client: AIClient, prompt_client: PromptClient)` โ’ TaskOutput | Execute the task. |
| `health()` โ’ WorkerHealth | Report health to the platform. |

**Worker constraints:**
- Workers must complete within the task's `timeout_seconds`.
- Workers use `AIClient` and `PromptClient` via injected SDK clients โ€” they never call providers directly.
- Workers must not import from `platform/` directly โ€” only from `aistudio.sdk.*`.
- Workers must emit structured logs via the platform logger.

---

## 11.7 Tool Plugin Contract

Tool plugins add new tools to the ToolRuntime. A tool plugin must implement:

| Method | Description |
|--------|-------------|
| `get_name()` โ’ string | Unique tool name (e.g., `"slack_notify"`). |
| `get_schema()` โ’ ToolSchema | JSON Schema for tool input. |
| `execute(input: dict, context: ToolContext)` โ’ ToolResult | Execute the tool. |
| `health()` โ’ ToolHealth | Report health. |

**Tool constraints:**
- Tool execution must complete within 60 seconds.
- Tools must not start subprocesses without explicit sandboxing.
- Tools that make HTTP requests must honor the SSRF allowlist configuration.
- Tool names must be globally unique โ€” the platform rejects registration if the name is taken.

---

## 11.8 Error Model

| Error Code | HTTP | Trigger |
|-----------|------|---------|
| `PLUGIN_NOT_FOUND` | 404 | Plugin ID not installed |
| `PLUGIN_ALREADY_INSTALLED` | 409 | Same plugin_id already installed |
| `PLUGIN_INCOMPATIBLE_VERSION` | 422 | Plugin requires higher platform version |
| `PLUGIN_PERMISSION_DENIED` | 403 | Plugin requests permissions above what ADMIN has granted |
| `PLUGIN_INSTALL_FAILED` | 500 | `on_install` hook raised an exception |
| `PLUGIN_TASK_TYPE_CONFLICT` | 409 | Plugin declares a task type already registered by another plugin |
| `PLUGIN_TOOL_NAME_CONFLICT` | 409 | Plugin registers a tool name already taken |

## 11.9 Security Model for Plugins

Plugins run in the same process as the platform in the current implementation. This means:
- Plugins have access to the platform's memory space
- Plugins can call platform SDK clients (with the permissions declared in the manifest)
- Plugins **cannot** access platform internal state (they have no access to the `platform/` namespace)

Future (v2.0): Plugins run in isolated subprocesses with a gRPC boundary, providing true isolation.

Current risk mitigation:
- Only ADMIN can install plugins
- Plugin manifest permissions are validated against a maximum grant policy
- Plugin code is reviewed before installation (no auto-install from marketplace)

## 11.10 Events Published

| Event | When |
|-------|------|
| `workspace.plugin.installed` | Plugin installed successfully |
| `workspace.plugin.uninstalled` | Plugin removed |
| `workspace.plugin.disabled` | Plugin disabled |
| `workspace.plugin.error` | Plugin health check fails |

## 11.11 Future Extension Points

| Extension | Description | Target version |
|-----------|-------------|---------------|
| Subprocess isolation | Run plugin workers in isolated subprocesses | v2.0 |
| Plugin marketplace | Registry of community-contributed plugins | v2.0 |
| Plugin versioning | Install and run multiple versions side-by-side | v1.1 |
| Prompt plugin type | Bundles of Prompt OS templates installed as a unit | v1.0 |

---



---

---

# 12. Cross-Contract Standards

These standards apply to every contract in this document. They resolve ambiguity when a contract's specific section is silent on a topic.

## 12.1 Correlation and Tracing

Every SDK call propagates a `correlation_id`. When a product calls `WorkflowClient.submit()`, that `correlation_id` flows into every `AIClient.execute()` call made by the workflow's tasks, every event published, every log line written, and every database row inserted.

**Propagation chain:**
```
HTTP Request (X-Correlation-ID: req-abc-123)
    โ”
    โ–ผ SecurityMiddleware extracts/generates correlation_id
WorkflowClient.submit(... correlation_id=req-abc-123)
    โ”
    โ–ผ WorkflowRuntime passes correlation_id to task dispatch
AIClient.execute(... correlation_id=req-abc-123)
    โ”
    โ–ผ AI ROS passes correlation_id to provider
AnthropicProvider.complete(metadata={"correlation_id": "req-abc-123"})
    โ”
    โ–ผ All platform events
PlatformEvent(correlation_id=req-abc-123)
    โ”
    โ–ผ All log lines
{"correlation_id": "req-abc-123", ...}
```

The `correlation_id` is generated by SecurityMiddleware if not present in the incoming request. All SDK methods that accept `correlation_id` propagate it automatically when called within a request context.

## 12.2 Pagination Contract

All list methods follow a consistent cursor-based pagination contract:

**Input parameters (all list methods):**
- `page_size: integer [1, 100]` โ€” Default: 20. Maximum: 100.
- `page_cursor: string` โ€” Opaque cursor from previous response. Absent on first call.

**Output fields (all list methods):**
- `items: [T]` โ€” The page of results.
- `total: integer` โ€” Total matching items (across all pages).
- `page_size: integer` โ€” Effective page size (may be less than requested on last page).
- `next_cursor: string | null` โ€” Null if this is the last page.
- `has_more: boolean` โ€” Equivalent to `next_cursor != null`.

**Cursor properties:**
- Opaque โ€” consumers must not parse or construct cursors.
- Stable within a result set โ€” same filter + same cursor always returns the same page.
- Does not guarantee stability across filter changes.
- Expires after 24 hours.

## 12.3 Idempotency Key Standard

For all mutating operations that accept an `idempotency_key`:
- Key is a string of max 256 characters.
- Key is scoped to the `product_id` of the caller.
- Duplicate calls with the same key within 24 hours return the original response.
- The original response is returned regardless of the current state (even if the resource has since changed).
- After 24 hours, the key may be reused and will create a new resource.

## 12.4 Timeout Propagation

When a caller sets a timeout on an SDK method call, that timeout propagates to all downstream calls made on behalf of that call:

```
WorkflowClient.submit(timeout=30s)
    โ”
    โ””โ”€โ”€ WorkflowRuntime dispatches task
            โ””โ”€โ”€ AIClient.execute(timeout=min(task.timeout, remaining))
                    โ””โ”€โ”€ AnthropicProvider.complete(timeout=min(task.timeout, remaining))
```

The effective timeout for any downstream call is `min(parent_timeout_remaining, method_default_timeout)`. A downstream call never has a longer timeout than its caller.

## 12.5 Error Response Envelope

All errors across all contracts use the same response envelope. HTTP errors from the platform API always return this body:

```
{
    "error": {
        "code":       string (SCREAMING_SNAKE_CASE, domain-prefixed),
        "message":    string (human-readable, no sensitive data),
        "request_id": string (UUID โ€” the correlation_id of the failing request),
        "timestamp":  string (ISO8601),
        "details":    object (optional, error-specific context fields),
        "retry_after": integer | null (seconds, for 429 and 503 responses)
    }
}
```

SDK method exceptions follow the same structure โ€” all exceptions in the hierarchy carry `error_code`, `message`, `request_id`, `timestamp`, and a `details` dict.

## 12.6 Backward Compatibility Decision Tree

When a contract change is proposed:

```
Is any existing consumer broken by this change?
    โ”
    โ”โ”€โ”€ NO โ’ MINOR increment. Ship in next minor release.
    โ”
    โ””โ”€โ”€ YES โ’
            Does the old behavior still work alongside the new?
                โ”
                โ”โ”€โ”€ YES โ’ Ship new behavior as additive MINOR increment.
                โ”          Mark old behavior as DEPRECATED with removal timeline.
                โ”          Remove old behavior in next MAJOR (with migration guide).
                โ”
                โ””โ”€โ”€ NO โ’ MAJOR increment required.
                           Write migration guide before shipping.
                           Announce to all consumer teams.
                           Provide compatibility period (minimum 2 minor releases).
```

## 12.7 SDK Method Stability Guarantees

| Change type | Version increment | Consumer action required |
|-------------|-----------------|------------------------|
| New optional input field | MINOR | None โ€” existing calls unchanged |
| New output field | MINOR | None โ€” consumers ignore unknown fields |
| New method | MINOR | None |
| New error code | MINOR | Consider handling the new code |
| New enum value | MINOR | Handle unknown enum values gracefully |
| New permission required | MINOR | Callers with the role automatically get it |
| Remove input field | MAJOR | Update call sites to remove the field |
| Remove output field | MAJOR | Update callers that read the field |
| Remove method | MAJOR | Migrate to replacement before MAJOR version |
| Required field added | MAJOR | Update all call sites to provide the field |
| Field type changed | MAJOR | Update all call sites and response handlers |
| Method renamed | MAJOR | Update all call sites |
| Error code renamed | MAJOR | Update all error handlers |
| Permission renamed | MAJOR | Update all RBAC configurations |

## 12.8 Contract Versioning Log

| Version | Date | Contract | Change | Breaking? |
|---------|------|---------|--------|-----------|
| 1.0.0 | 2026-06-28 | All | Initial ratification | N/A |

---

---

# Appendix A โ€” Shared DTO Catalog

These DTOs are used across multiple contracts. They are defined once here and referenced by name in the contract definitions above.

---

### A.1 ToolCallResult

Produced by AIClient when a model makes a tool call.

| Field | Type | Description |
|-------|------|-------------|
| `tool_call_id` | string | Provider-assigned ID for this tool call. |
| `tool_name` | string | Name of the tool called. |
| `tool_input` | map[string, any] | Arguments provided to the tool. |
| `tool_output` | map[string, any] \| null | Result from ToolRuntime. Null if tool execution is still pending. |
| `tool_status` | enum(PENDING, SUCCESS, FAILED, TIMEOUT) | |
| `execution_duration_ms` | integer \| null | |

---

### A.2 ToolDefinition

Describes a tool available for AI model use. Provided to AIClient in `ExecutionRequest.tools` by name; resolved to full definitions by AI ROS via ToolRuntime.

| Field | Type | Description |
|-------|------|-------------|
| `tool_name` | string | Unique tool identifier. |
| `description` | string | What the tool does (shown to the AI model). |
| `input_schema` | map | JSON Schema for tool inputs. |
| `output_schema` | map | JSON Schema for tool outputs. |

---

### A.3 TokenUsage

Token consumption reported by providers.

| Field | Type | Description |
|-------|------|-------------|
| `prompt_tokens` | integer | |
| `completion_tokens` | integer | |
| `cache_read_tokens` | integer | Prompt cache hits (Anthropic prompt caching). |
| `cache_write_tokens` | integer | Prompt cache writes. |
| `total_tokens` | integer | Sum of prompt + completion. |

---

### A.4 WorkflowSummary

Abbreviated workflow record for list responses.

| Field | Type | Description |
|-------|------|-------------|
| `workflow_id` | UUID | |
| `product_id` | string | |
| `plan_name` | string | |
| `status` | enum(QUEUED, ACTIVE, PAUSED, COMPLETED, FAILED, CANCELLED) | |
| `task_count` | integer | |
| `completed_task_count` | integer | |
| `created_at` | ISO8601 | |
| `updated_at` | ISO8601 | |
| `total_cost_usd` | float | |

---

### A.5 RepositoryInfo

Workspace repository metadata.

| Field | Type | Description |
|-------|------|-------------|
| `repository_id` | string | |
| `path` | string | Filesystem path (relative to workspace root). |
| `type` | enum(git) | |
| `current_branch` | string | |
| `last_commit_sha` | string | |
| `last_commit_at` | ISO8601 | |
| `is_dirty` | boolean | True if there are uncommitted changes. |

---

### A.6 ToolContext

Context injected into Tool plugin `execute()` calls.

| Field | Type | Description |
|-------|------|-------------|
| `execution_id` | UUID | The AIClient execution_id that triggered this tool call. |
| `workflow_id` | UUID \| null | The workflow this tool call is part of, if any. |
| `product_id` | string | The product that owns this execution. |
| `correlation_id` | UUID | |
| `working_directory` | string | The sandboxed working directory for file operations. |
| `allowed_hosts` | [string] | Allowed external hostnames for HTTP tools. |

---

### A.7 GraphPath

A path in the knowledge graph from start node to a result node.

| Field | Type | Description |
|-------|------|-------------|
| `nodes` | [NodeResult] | Ordered list of nodes in the path. |
| `relationships` | [RelationshipResult] | Ordered list of relationships connecting nodes. |
| `length` | integer | Number of hops. |
| `total_weight` | float | Sum of relationship weights along the path. |

---

---

# Appendix B โ€” Shared Error Codes

Error codes not specific to a single contract:

| Error Code | HTTP | Description |
|-----------|------|-------------|
| `INTERNAL_ERROR` | 500 | Unhandled platform error |
| `SERVICE_UNAVAILABLE` | 503 | Platform module unavailable; Retry-After header present |
| `REQUEST_TIMEOUT` | 504 | Operation exceeded timeout |
| `INVALID_REQUEST` | 400 | Generic request validation failure (use specific codes where available) |
| `NOT_FOUND` | 404 | Generic not-found (use specific codes where available) |
| `CONFLICT` | 409 | Generic conflict (use specific codes where available) |
| `RATE_LIMITED` | 429 | Generic rate limit; Retry-After header present |
| `UNSUPPORTED_API_VERSION` | 400 | API version not supported |
| `MISSING_CORRELATION_ID` | 400 | Correlation ID required but not provided |
| `SCHEMA_VERSION_UNSUPPORTED` | 400 | Event schema version not supported by consumer |

---

---

# Appendix C โ€” Contract Interaction Map

This diagram shows which contracts interact with each other:

```
โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
โ”                        PRODUCTS / DESKTOP                               โ”
โ”            (consume via SDK โ€” never access internals)                   โ”
โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”ฌโ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”ฌโ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”ฌโ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
         โ”AIClient      โ”WorkflowClient   โ”PromptClient  BrainClient
         โ”              โ”                 โ”              KnowledgeClient
         โ”              โ”                 โ”              WorkspaceClient
         โ”              โ”                 โ”              SecurityClient
         โ–ผ              โ–ผ                 โ–ผ              PluginAPI
โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
โ”                      PLATFORM CONTRACTS                                 โ”
โ”                                                                         โ”
โ”  โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”  โ”
โ”  โ”  AIClient  โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ–บ ProviderClient (abstract)           โ”  โ”
โ”  โ”      โ”                              โ”                             โ”  โ”
โ”  โ”      โ” uses                    Anthropic โ” OpenAI โ” Ollama โ” etc. โ”  โ”
โ”  โ”      โ–ผ                                                            โ”  โ”
โ”  โ”  PromptClient (auto-render when prompt_id provided)               โ”  โ”
โ”  โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”  โ”
โ”                                                                         โ”
โ”  โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”  โ”
โ”  โ”  WorkflowClient โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ–บ AIClient (task execution)           โ”  โ”
โ”  โ”      โ”              โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ–บ PromptClient (task rendering)       โ”  โ”
โ”  โ”      โ”              โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ–บ SecurityClient (gate authorization) โ”  โ”
โ”  โ”      โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€ BrainClient (pattern lookup)        โ”  โ”
โ”  โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”  โ”
โ”                                                                         โ”
โ”  โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”  โ”
โ”  โ”  BrainClient โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ–บ KnowledgeClient (graph queries)     โ”  โ”
โ”  โ”      โ”                โ”€โ”€โ”€โ”€โ”€โ”€โ–บ AIClient (pattern analysis)         โ”  โ”
โ”  โ”      โ””โ”€โ”€ Consumes events from: WorkflowClient, AIClient           โ”  โ”
โ”  โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”  โ”
โ”                                                                         โ”
โ”  โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”  โ”
โ”  โ”  EventBus โ”€โ”€โ”€ all contracts publish through EventBus              โ”  โ”
โ”  โ”               all contracts that consume events use EventBus.sub  โ”  โ”
โ”  โ”               WebSocket delivery to desktop                       โ”  โ”
โ”  โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”  โ”
โ”                                                                         โ”
โ”  โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”  โ”
โ”  โ”  SecurityClient โ”€โ”€โ”€ all contracts check permissions via this      โ”  โ”
โ”  โ”                     SecurityMiddleware wraps all contract calls    โ”  โ”
โ”  โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”  โ”
โ”                                                                         โ”
โ”  โ”โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”  โ”
โ”  โ”  WorkspaceClient โ”€โ”€ read-only registry; consulted at startup      โ”  โ”
โ”  โ”  PluginAPI โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€ extends AIClient tools and WorkflowRuntime   โ”  โ”
โ”  โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”  โ”
โ””โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”€โ”
```

---

---

# Appendix D โ€” Event Payload Schemas

Complete payload schemas for every event in the catalog.

---

### D.1 `ai.execution.completed`

```
{
    execution_id:    UUID,
    product_id:      string,
    workflow_id:     UUID | null,
    task_id:         string | null,
    provider_id:     string,
    model_id:        string,
    prompt_tokens:   integer,
    completion_tokens: integer,
    cost_usd:        float,
    duration_ms:     integer,
    stop_reason:     enum(END_TURN, MAX_TOKENS, STOP_SEQUENCE, TOOL_USE),
    tool_calls_count: integer
}
```

### D.2 `workflows.task.completed`

```
{
    workflow_id:      UUID,
    task_id:          string,
    task_type:        string,
    agent_type:       string,
    product_id:       string,
    status:           "COMPLETED",
    duration_ms:      integer,
    ai_executions:    [UUID],
    output_keys:      [string],    // keys present in TaskOutput.output
    retry_count:      integer
}
```

### D.3 `workflows.task.failed`

```
{
    workflow_id:      UUID,
    task_id:          string,
    task_type:        string,
    product_id:       string,
    status:           "FAILED",
    error_code:       string,
    error_message:    string,
    retry_count:      integer,
    max_retries:      integer,
    final_failure:    boolean     // true if no more retries
}
```

### D.4 `brain.pattern.discovered`

```
{
    pattern_id:        UUID,
    domain:            string,
    pattern_type:      enum(DO, AVOID, CONSIDER),
    description:       string,
    confidence:        float,
    evidence_count:    integer,
    source_product_ids: [string],
    discovered_at:     ISO8601
}
```

### D.5 `security.auth.failed`

```
{
    attempt_method:   enum(API_KEY, JWT),
    ip_address:       string,
    reason:           enum(INVALID_KEY, EXPIRED_KEY, REVOKED_KEY, INVALID_JWT, EXPIRED_JWT),
    path:             string,      // endpoint attempted
    user_agent:       string | null
}
```

### D.6 `prompts.template.activated`

```
{
    template_id:      UUID,
    version:          integer,
    previous_version: integer | null,
    display_name:     string,
    owner:            string,
    activated_by:     string,       // identity from AuthContext
    activated_at:     ISO8601
}
```

### D.7 `ai.budget.limit_exceeded`

```
{
    product_id:         string,
    daily_limit_usd:    float,
    daily_spend_usd:    float,
    excess_usd:         float,
    rejected_execution_id: UUID | null,   // the execution that was rejected
    reset_at:           ISO8601           // when the daily limit resets
}
```

### D.8 `ai.provider.circuit_opened`

```
{
    provider_id:        string,
    failure_count:      integer,
    failure_window_secs: integer,
    open_until:         ISO8601,
    last_error:         string      // sanitized โ€” no provider-internal details
}
```

---

---

# Appendix E โ€” Contract Compliance Checklist

When implementing a consumer of any contract defined in this document, verify:

### For all consumers

- [ ] I am importing from `aistudio.sdk.*` only โ€” not from `aistudio.platform.*`
- [ ] I am not calling any AI provider directly โ€” all AI calls go through `AIClient`
- [ ] I am not rendering prompt strings inline โ€” all prompts use `PromptClient.render(template_id, variables)`
- [ ] I am propagating `correlation_id` from my request context into all SDK calls
- [ ] I handle all documented error codes, not just the success case
- [ ] I do not retry `submit()` / mutating calls without an `idempotency_key`
- [ ] I use cursor-based pagination (`page_cursor`) on all list method calls
- [ ] I call `BrainClient.record_experience()` after every significant AI execution

### For WorkflowClient consumers

- [ ] My `WorkflowPlan` has no cycles in `dag_edges`
- [ ] All `dag_edges` reference valid `task_id` values in `tasks`
- [ ] I use `prompt_id` (not `prompt_text`) in `TaskDefinition` wherever possible
- [ ] I do not set `auto_approve_after_seconds` on any `GateDefinition`
- [ ] I handle the `PAUSED` status in my polling/event handling logic

### For AIClient consumers

- [ ] I call `estimate_cost()` before large executions and check against my budget
- [ ] I use `session_id` for multi-turn conversations (not sending full history in prompt)
- [ ] I record the `execution_id` from `ExecutionResult` for tracing and A/B testing
- [ ] I handle `AI_BUDGET_DAILY_LIMIT_EXCEEDED` gracefully (queue for later, not crash)
- [ ] I do not implement my own retry logic (AI ROS retries internally)

### For ProviderClient implementors

- [ ] `health_check()` completes within 5 seconds
- [ ] `health_check()` does not incur billing
- [ ] All provider-SDK exceptions are wrapped in normalized exception types
- [ ] `estimate_cost()` makes no API calls
- [ ] `complete()` and `complete_async()` return identical results for identical inputs
- [ ] The adapter is stateless per-request (no instance-level request state)

### For PluginAPI consumers (plugin authors)

- [ ] My plugin manifest declares all permissions it needs (no more)
- [ ] My worker plugin calls AI via the injected `AIClient`, not directly
- [ ] My tool plugin does not start un-sandboxed subprocesses
- [ ] My `on_install()` is idempotent โ€” calling it twice leaves the platform in the same state
- [ ] My `health()` returns OFFLINE if my plugin is not functional

---

---

# Appendix F โ€” Sequence: New Product Request-to-Response

Complete trace of a `POST /api/v1/products` request through all contracts:

```
HTTP Client
    โ”
    โ” POST /api/v1/products
    โ” X-API-Key: <key>
    โ” Content-Type: application/json
    โ” Body: { "name": "My Product", "description": "..." }
    โ”
    โ–ผ
SecurityClient (SecurityMiddleware)
    โ” hash(api_key) โ’ db lookup โ’ AuthContext(role=DEVELOPER)
    โ” RateLimiter.check(identity, "/api/v1/products") โ’ OK
    โ” request.state.auth_context = AuthContext(...)
    โ”
    โ–ผ
ai-software-factory product route handler
    โ” require_permission(PRODUCT_CREATE) โ’ check AuthContext โ’ granted
    โ” CreateProductRequest.model_validate(body) โ’ valid
    โ”
    โ–ผ
ProductCreationService (product business logic)
    โ”
    โ”โ”€โ”€ BrainClient.recommend(context={name, description})
    โ”       โ’ [Recommendation: "Use REST API pattern", "TypeScript recommended"]
    โ”
    โ”โ”€โ”€ PromptClient.render(template_id="aisf/intake/product_spec_v3",
    โ”                       variables={name, description, recommendations})
    โ”       โ’ PromptRenderResult(rendered_prompt="...", render_id=uuid)
    โ”
    โ”โ”€โ”€ AIClient.execute(ExecutionRequest(
    โ”       prompt=rendered_prompt,
    โ”       model_preference=CAPABILITY,
    โ”       tools=["file_write"],
    โ”       cost_budget_usd=2.0
    โ”   ))
    โ”       โ’ AI ROS: BudgetManager OK โ’ route โ’ AnthropicProvider
    โ”       โ’ ExecutionResult(content="Blueprint: ...", cost_usd=0.043)
    โ”
    โ”โ”€โ”€ BrainClient.record_experience(
    โ”       task_type="product_intake",
    โ”       outcome=SUCCESS,
    โ”       cost_usd=0.043
    โ”   )
    โ”       โ’ ExperienceRecordAck(experience_id=uuid)
    โ”
    โ”โ”€โ”€ WorkflowClient.submit(WorkflowSubmitRequest(
    โ”       product_id="my-product-id",
    โ”       plan_name="My Product โ€” Build Phase",
    โ”       tasks=[architecture_task, implementation_task, test_task],
    โ”       dag_edges=[["impl", "arch"], ["test", "impl"]],
    โ”       gates=[GateDefinition(after_task_id="arch", before_task_id="impl")]
    โ”   ))
    โ”       โ’ WorkflowHandle(workflow_id=uuid, status=QUEUED)
    โ”
    โ””โ”€โ”€ EventBus.publish(PlatformEvent(
            event_type="aisf.product.created",
            payload={product_id, workflow_id, name}
        ))
    โ”
    โ–ผ
HTTP Response 201 Created
    Location: /api/v1/products/uuid
    Body: { "product_id": "uuid", "workflow_id": "uuid", "status": "workflow_queued" }
```

---

*End of document.*

---

**AI Studio Platform โ€” Interface Contracts**  
**Document ID:** PLATFORM-CONTRACTS  
**Version:** 1.0.0  
**Date:** 2026-06-28  
**Status:** RATIFIED  
**Next Review:** Aligned with SDK v2.0 planning  
**Document Owner:** Chief Software Architect  

*Contracts defined in this document are binding on all consumers. Deviations constitute platform violations and must be resolved before merge.*

