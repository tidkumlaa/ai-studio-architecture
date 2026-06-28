# AI Provider Contracts — Interface Specification

**Version:** 1.0.0  
**Status:** Approved for Implementation  
**Phase:** 1A  
**Date:** 2026-06-29  
**Classification:** Internal Architecture

---

## 1. Purpose

This document specifies the complete contract that every AI provider plugin must implement to participate in AI ROS. A provider plugin is the only code that is allowed to know provider-specific API details. All other AI ROS subsystems communicate exclusively through these interfaces.

---

## 2. Responsibilities

- Define the `AIProviderPlugin` interface that all provider adapters must implement
- Specify request and response data models (provider-neutral)
- Define streaming protocols
- Specify error taxonomy and retry contracts
- Define capability declaration format
- Establish version compatibility rules

---

## 3. Architecture

```
AI ROS Execution Engine
        │
        │  AIProviderPlugin interface (this spec)
        ▼
┌───────────────────────────┐
│    Provider Adapter        │
│  (one per AI provider)    │
│                           │
│  implements:              │
│    execute()              │
│    stream()               │
│    embed()                │
│    transcribe()           │
│    generate_image()       │
│    health_check()         │
│    get_capabilities()     │
└───────────┬───────────────┘
            │
            │  Provider-specific HTTP/SDK calls
            ▼
   [Claude API / OpenAI / Gemini / ...]
```

---

## 4. Core Interface

### 4.1 AIProviderPlugin

The root interface that every provider adapter must implement.

```
AIProviderPlugin
  ├── METADATA: ProviderMetadata          (static, declared at class level)
  ├── execute(request: AIRequest) → AIResponse
  ├── stream(request: AIRequest) → Iterator[AIStreamChunk]
  ├── embed(request: EmbedRequest) → EmbedResponse
  ├── transcribe(request: TranscribeRequest) → TranscribeResponse
  ├── generate_image(request: ImageRequest) → ImageResponse
  ├── health_check() → ProviderHealth
  └── get_capabilities() → ProviderCapabilities
```

### 4.2 ProviderMetadata

Static declaration attached to every provider class.

| Field | Type | Description |
|-------|------|-------------|
| `provider_id` | `str` | Unique, stable identifier (e.g., `anthropic`, `openai`) |
| `display_name` | `str` | Human-readable name |
| `version` | `str` | Adapter version (semver) |
| `protocol_version` | `str` | AI ROS protocol version this adapter implements |
| `homepage_url` | `str` | Provider documentation URL |
| `requires_api_key` | `bool` | Whether an API key credential is required |
| `supports_local_deployment` | `bool` | Whether provider can run locally (Ollama, LM Studio) |

### 4.3 ProviderCapabilities

Declared by each provider to feed the Capability Registry.

| Field | Type | Description |
|-------|------|-------------|
| `models` | `List[ModelCapabilityDeclaration]` | All models offered by this provider |
| `max_concurrent_requests` | `int` | Max parallel requests the adapter should receive |
| `supports_streaming` | `bool` | Provider offers streaming responses |
| `supports_batch` | `bool` | Provider offers batch API |

### 4.4 ModelCapabilityDeclaration

Per-model capability declaration within ProviderCapabilities.

| Field | Type | Description |
|-------|------|-------------|
| `provider_model_id` | `str` | Provider's native model identifier |
| `canonical_model_id` | `str` | AI ROS canonical model ID (see AI-MODEL-CATALOG.md) |
| `context_window_tokens` | `int` | Maximum context window in tokens |
| `max_output_tokens` | `int` | Maximum tokens per response |
| `capabilities` | `CapabilityFlags` | Bitmask of supported capabilities |
| `pricing` | `ModelPricing` | Cost per token (input/output/cached) |
| `latency_class` | `LatencyClass` | `ULTRA_LOW` / `LOW` / `MEDIUM` / `HIGH` |
| `is_available` | `bool` | Currently available for routing |

### 4.5 CapabilityFlags

```
text_generation       — basic text completion and instruction following
chat                  — multi-turn conversation
vision_image          — accepts image inputs
vision_video          — accepts video inputs
audio_input           — accepts audio inputs
audio_output          — generates audio output
embeddings            — generates vector embeddings
tool_calling          — supports function/tool calling
parallel_tool_calling — can call multiple tools in one turn
structured_output     — enforces JSON schema on output
extended_thinking     — reasoning / chain-of-thought mode
code_execution        — sandboxed code interpreter
web_search            — live web browsing
file_handling         — accepts file uploads (PDF, DOCX, etc.)
image_generation      — generates images
```

---

## 5. Request Models

### 5.1 AIRequest (text generation)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `request_id` | `UUID` | Yes | Globally unique request ID from AI ROS |
| `canonical_model_id` | `str` | Yes | AI ROS canonical model ID |
| `messages` | `List[Message]` | Yes | Conversation history |
| `system_prompt` | `str \| None` | No | System prompt |
| `tools` | `List[ToolDefinition] \| None` | No | Tool schemas |
| `tool_choice` | `ToolChoice` | No | `AUTO` / `NONE` / `REQUIRED` / specific tool |
| `max_tokens` | `int` | No | Override max output tokens |
| `temperature` | `float` | No | 0.0–2.0 (default: provider default) |
| `top_p` | `float` | No | Nucleus sampling (default: provider default) |
| `stream` | `bool` | No | Request streaming response |
| `response_format` | `ResponseFormat \| None` | No | `JSON_MODE` / `JSON_SCHEMA` / `TEXT` |
| `json_schema` | `dict \| None` | No | Schema for JSON_SCHEMA mode |
| `stop_sequences` | `List[str]` | No | Stop token sequences |
| `metadata` | `dict` | No | Pass-through metadata (not sent to provider) |
| `thinking_budget` | `int \| None` | No | Token budget for extended thinking mode |

### 5.2 Message

| Field | Type | Description |
|-------|------|-------------|
| `role` | `MessageRole` | `USER` / `ASSISTANT` / `TOOL_RESULT` |
| `content` | `List[ContentBlock]` | Multimodal content blocks |

### 5.3 ContentBlock (union type)

| Variant | Fields | Description |
|---------|--------|-------------|
| `TextBlock` | `text: str` | Plain text content |
| `ImageBlock` | `url: str \| None`, `base64_data: str \| None`, `media_type: str` | Image (URL or base64) |
| `AudioBlock` | `base64_data: str`, `format: str` | Audio data |
| `DocumentBlock` | `base64_data: str`, `filename: str`, `media_type: str` | File upload |
| `ToolCallBlock` | `tool_call_id: str`, `name: str`, `input: dict` | Tool invocation from assistant |
| `ToolResultBlock` | `tool_call_id: str`, `content: str \| List[ContentBlock]` | Tool result injected as user |

### 5.4 ToolDefinition

| Field | Type | Description |
|-------|------|-------------|
| `name` | `str` | Tool name (unique within request) |
| `description` | `str` | Natural language description for the model |
| `input_schema` | `dict` | JSON Schema for tool input |
| `side_effects` | `bool` | Hint: does calling this tool have side effects? |

---

## 6. Response Models

### 6.1 AIResponse (text generation)

| Field | Type | Description |
|-------|------|-------------|
| `request_id` | `UUID` | Echoed from AIRequest |
| `response_id` | `UUID` | Unique ID for this response |
| `provider_id` | `str` | Which provider served this response |
| `provider_model_id` | `str` | Provider's native model ID used |
| `canonical_model_id` | `str` | AI ROS canonical model ID |
| `stop_reason` | `StopReason` | `END_TURN` / `MAX_TOKENS` / `TOOL_USE` / `STOP_SEQUENCE` |
| `content` | `List[ContentBlock]` | Response content (text + tool calls) |
| `usage` | `TokenUsage` | Token counts |
| `thinking` | `str \| None` | Extended thinking content (if requested) |
| `latency_ms` | `int` | Provider response time |
| `provider_request_id` | `str \| None` | Provider's own request ID (for support) |

### 6.2 TokenUsage

| Field | Type | Description |
|-------|------|-------------|
| `input_tokens` | `int` | Tokens in the prompt |
| `output_tokens` | `int` | Tokens generated |
| `cache_read_tokens` | `int` | Prompt cache hit tokens (Claude) |
| `cache_write_tokens` | `int` | Prompt cache write tokens (Claude) |
| `reasoning_tokens` | `int` | Thinking/reasoning tokens (if applicable) |

### 6.3 StopReason

| Value | Meaning |
|-------|---------|
| `END_TURN` | Model finished naturally |
| `MAX_TOKENS` | Hit max_tokens limit |
| `TOOL_USE` | Model is requesting a tool call |
| `STOP_SEQUENCE` | Hit a configured stop sequence |
| `CONTENT_FILTER` | Provider blocked the response |

---

## 7. Streaming Protocol

### 7.1 AIStreamChunk (union type)

| Variant | Fields | Description |
|---------|--------|-------------|
| `StreamStart` | `request_id`, `response_id`, `provider_id`, `canonical_model_id` | First chunk |
| `TextDelta` | `index: int`, `delta: str` | Incremental text |
| `ThinkingDelta` | `delta: str` | Incremental thinking content |
| `ToolCallStart` | `index: int`, `tool_call_id: str`, `name: str` | Tool call starting |
| `ToolCallInputDelta` | `index: int`, `tool_call_id: str`, `delta: str` | Tool call input (JSON fragment) |
| `ToolCallEnd` | `index: int`, `tool_call_id: str` | Tool call complete |
| `StreamEnd` | `stop_reason: StopReason`, `usage: TokenUsage`, `latency_ms: int` | Final chunk |
| `StreamError` | `error: ProviderError` | Error during streaming |

### 7.2 Back-pressure

The adapter must respect back-pressure from the Execution Engine. If the consumer is not consuming chunks fast enough, the adapter must block the generator rather than buffer unboundedly.

---

## 8. Embedding Interface

### 8.1 EmbedRequest

| Field | Type | Description |
|-------|------|-------------|
| `request_id` | `UUID` | |
| `canonical_model_id` | `str` | Embedding model |
| `inputs` | `List[str]` | Text strings to embed |
| `input_type` | `EmbedInputType` | `QUERY` / `DOCUMENT` (for asymmetric models) |
| `dimensions` | `int \| None` | Override output dimensions (if supported) |

### 8.2 EmbedResponse

| Field | Type | Description |
|-------|------|-------------|
| `request_id` | `UUID` | |
| `embeddings` | `List[List[float]]` | One vector per input |
| `model` | `str` | Model actually used |
| `usage` | `TokenUsage` | Token counts |

---

## 9. Audio Interface

### 9.1 TranscribeRequest

| Field | Type | Description |
|-------|------|-------------|
| `request_id` | `UUID` | |
| `canonical_model_id` | `str` | Transcription model |
| `audio_data` | `bytes` | Raw audio bytes |
| `audio_format` | `str` | `mp3` / `wav` / `webm` / `ogg` / `flac` |
| `language` | `str \| None` | ISO 639-1 language code (auto-detect if None) |
| `prompt` | `str \| None` | Context hint for transcription |
| `timestamp_granularity` | `str` | `none` / `word` / `segment` |

### 9.2 TranscribeResponse

| Field | Type | Description |
|-------|------|-------------|
| `request_id` | `UUID` | |
| `text` | `str` | Full transcript |
| `language` | `str` | Detected or specified language |
| `duration_seconds` | `float` | Audio duration |
| `segments` | `List[TranscriptSegment]` | Time-aligned segments (if requested) |
| `words` | `List[TranscriptWord]` | Word-level timestamps (if requested) |

---

## 10. Image Generation Interface

### 10.1 ImageRequest

| Field | Type | Description |
|-------|------|-------------|
| `request_id` | `UUID` | |
| `canonical_model_id` | `str` | Image generation model |
| `prompt` | `str` | Image description |
| `negative_prompt` | `str \| None` | What to avoid |
| `n` | `int` | Number of images (default: 1) |
| `size` | `str` | `256x256` / `512x512` / `1024x1024` / `1792x1024` / `1024x1792` |
| `quality` | `str` | `standard` / `hd` |
| `style` | `str \| None` | `vivid` / `natural` (provider-specific) |
| `response_format` | `str` | `url` / `base64` |

### 10.2 ImageResponse

| Field | Type | Description |
|-------|------|-------------|
| `request_id` | `UUID` | |
| `images` | `List[GeneratedImage]` | Generated images |
| `usage` | `dict` | Provider-specific usage info |

---

## 11. Error Taxonomy

### 11.1 ProviderError

| Field | Type | Description |
|-------|------|-------------|
| `error_code` | `ProviderErrorCode` | Normalized error code |
| `message` | `str` | Human-readable description |
| `provider_error_code` | `str \| None` | Raw error code from provider |
| `retryable` | `bool` | Whether AI ROS should retry this request |
| `retry_after_seconds` | `int \| None` | Minimum wait before retry (from Retry-After header) |
| `http_status` | `int \| None` | HTTP status code |

### 11.2 ProviderErrorCode (normalized taxonomy)

| Code | Meaning | Retryable |
|------|---------|-----------|
| `AUTH_FAILED` | Invalid or expired API key | No |
| `RATE_LIMITED` | Provider rate limit hit | Yes |
| `QUOTA_EXCEEDED` | Account quota exhausted | No |
| `MODEL_UNAVAILABLE` | Requested model not available | Maybe |
| `CONTEXT_TOO_LONG` | Input exceeds model context window | No |
| `CONTENT_FILTERED` | Content policy violation | No |
| `SERVICE_UNAVAILABLE` | Provider is down | Yes |
| `TIMEOUT` | Request timed out | Yes |
| `INVALID_REQUEST` | Malformed request parameters | No |
| `CAPACITY_OVERLOADED` | Provider overloaded | Yes |
| `STREAM_INTERRUPTED` | Streaming was interrupted mid-response | Yes |
| `UNKNOWN` | Unmapped provider error | Maybe |

---

## 12. Retry Contract

The adapter is **not responsible** for retrying. The Execution Engine owns retry logic.

The adapter must:
1. Raise `ProviderError` with accurate `retryable` flag and `retry_after_seconds`
2. Not swallow errors and return partial/empty responses
3. Not retry internally (causes double-counting in the Quota Engine)

The Execution Engine will apply exponential backoff with jitter, respecting `retry_after_seconds`.

---

## 13. Health Check Interface

### 13.1 health_check() → ProviderHealth

| Field | Type | Description |
|-------|------|-------------|
| `provider_id` | `str` | |
| `status` | `HealthStatus` | `HEALTHY` / `DEGRADED` / `UNAVAILABLE` |
| `latency_ms` | `int` | Round-trip latency for probe request |
| `checked_at` | `datetime` | Timestamp of check |
| `error_message` | `str \| None` | Present if not HEALTHY |
| `models_available` | `List[str]` | Canonical model IDs currently available |

Health checks must be lightweight: a minimal API call (e.g., list models, or a single-token generation). They must not consume meaningful quota.

---

## 14. Provider Registration

Providers are registered via a manifest file rather than code-level coupling:

```yaml
provider_id: anthropic
adapter_class: "ai_ros.providers.anthropic.AnthropicAdapter"
display_name: "Anthropic Claude"
version: "1.0.0"
protocol_version: "1.0"
requires_api_key: true
supports_local_deployment: false
credential_key: anthropic_api_key       # key in Account Manager
health_check_interval_seconds: 30
```

AI ROS loads provider manifests from a configured directory at startup and on `SIGHUP`. New providers are hot-registered without restart.

---

## 15. Version Compatibility

### Protocol Versions

| Protocol Version | Status | Notes |
|-----------------|--------|-------|
| `1.0` | Current | Initial AI ROS protocol |

Adapters declare their supported protocol version in `ProviderMetadata.protocol_version`. If an adapter declares an incompatible protocol version, AI ROS will refuse to load it and log a structured error.

### Adapter Versioning

Provider adapters follow semver. AI ROS logs adapter version at startup. Breaking changes in the AIProviderPlugin interface increment the protocol version; the old interface continues to be supported for one major AI ROS version.

---

## 16. Data Model

See `AI-RESOURCE-DATABASE.md` tables: `ai_ros_providers`, `ai_ros_provider_models`, `ai_ros_provider_health`.

---

## 17. Events Published

| Event | Trigger |
|-------|---------|
| `provider.registered` | New provider loaded |
| `provider.health.changed` | Provider health status changed |
| `provider.model.available` | Model came online |
| `provider.model.unavailable` | Model went offline |

Full event schemas in `AI-RESOURCE-EVENTS.md`.

---

## 18. Security

- Adapters receive credentials from the Account Manager at instantiation time; they must not cache credentials beyond the lifetime of the adapter instance
- Adapters must not log credential values or API responses that contain credential echoes
- Adapters must enforce TLS for all provider communication (no HTTP allowed)
- Adapters must validate response signatures where the provider supports them (e.g., webhooks)

---

## 19. Scalability

- Adapters are stateless; multiple instances can be created by the Execution Engine
- Adapters must be thread-safe (or document if they are not)
- Connection pooling should be implemented within the adapter using the provider's official SDK if available
- Adapters should implement connection keep-alive and respect provider-recommended connection limits

---

## 20. Future Evolution

| Capability | Status | Notes |
|-----------|--------|-------|
| Agent protocol (Agents-as-providers) | Planned | Protocol 1.1 |
| Provider webhooks for async responses | Planned | Protocol 1.1 |
| Batch API (async file in/out) | Planned | Protocol 1.2 |
| Cross-provider prompt caching | Research | Protocol 2.0 |

---

## 21. Implementation Roadmap

### Week 1–2: Interface Foundation
- Define all dataclasses and enums
- Implement `AIProviderPlugin` ABC
- Write provider contract test suite (adapter must pass all tests)

### Week 3–4: First Adapters
- Anthropic Claude adapter (streaming, tools, extended thinking)
- Ollama adapter (local models)
- Adapter contract test suite automated in CI

### Week 5–8: Additional Adapters
- OpenAI (GPT-4o, o-series reasoning, embeddings, DALL-E, Whisper)
- Gemini (Google AI Studio + Vertex AI)
- Azure OpenAI
- OpenRouter (meta-provider wrapping many backends)
- LM Studio (local deployment, OpenAI-compatible API)
- DeepSeek
- Qwen (Alibaba)

---

## 22. Cross References

| Document | Relationship |
|----------|-------------|
| AI-RESOURCE-OS-BLUEPRINT.md | Parent system overview |
| AI-MODEL-CATALOG.md | Canonical model IDs used in contracts |
| AI-ACCOUNT-MANAGER.md | Credential injection into adapters |
| AI-RESOURCE-EVENTS.md | Events published by provider layer |
| AI-RESOURCE-SECURITY.md | TLS and credential handling requirements |
| AI-BENCHMARK-CENTER.md | Performance data collected per adapter |
