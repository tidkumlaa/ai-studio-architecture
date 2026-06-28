# AI Model Catalog — Canonical Model Registry

**Version:** 1.0.0  
**Status:** Approved for Implementation  
**Phase:** 1A  
**Date:** 2026-06-29  
**Classification:** Internal Architecture

---

## 1. Purpose

The AI Model Catalog is the single source of truth for every AI model available through AI ROS. It establishes canonical model identities that are independent of provider naming, normalizes capability declarations, and provides the pricing and performance data that the Routing Engine and Budget Engine require.

Products never reference provider model IDs. They reference canonical model IDs or capability profiles. AI ROS resolves these to provider-specific identifiers at routing time.

---

## 2. Responsibilities

- Maintain canonical model registry across all providers
- Map canonical IDs to provider-specific model names
- Track capability flags per model
- Store pricing tiers (input tokens, output tokens, cached tokens)
- Classify models by latency class and use-case suitability
- Track model availability, deprecation, and successor relationships

---

## 3. Architecture

```
Products / Routing Requests
         │
         │  canonical_model_id  OR  CapabilityProfile
         ▼
┌────────────────────────────────────────┐
│           Model Registry               │
│                                        │
│  resolve(canonical_id) → ModelRecord   │
│  query(profile) → List[ModelRecord]    │
│  rank(records, criteria) → ordered     │
└──────────┬─────────────────────────────┘
           │
    ┌──────┴──────┐
    ▼             ▼
Provider      Capability
Mapping       Registry
Table         (flags index)
```

---

## 4. Canonical Model ID Format

```
{provider_slug}/{family}/{generation}

Examples:
  anthropic/claude/claude-opus-4-8
  anthropic/claude/claude-sonnet-4-6
  anthropic/claude/claude-haiku-4-5
  anthropic/claude/claude-fable-5
  openai/gpt/gpt-4o
  openai/gpt/gpt-4o-mini
  openai/o/o3
  openai/o/o4-mini
  openai/dall-e/dall-e-3
  openai/whisper/whisper-1
  openai/text-embedding/text-embedding-3-large
  google/gemini/gemini-2-5-pro
  google/gemini/gemini-2-5-flash
  meta/llama/llama-3-3-70b
  deepseek/deepseek/deepseek-v3
  qwen/qwen/qwen-2-5-72b
  local/ollama/{user-defined-tag}
  local/lm-studio/{user-defined-tag}
```

Canonical IDs are stable and version-pinned. When a provider retires a model, the canonical ID remains in the catalog with `status: DEPRECATED` and a `successor_canonical_id`.

---

## 5. Model Record Schema

| Field | Type | Description |
|-------|------|-------------|
| `canonical_id` | `str` | Primary key, stable forever |
| `display_name` | `str` | Human-readable name |
| `provider_id` | `str` | Which provider serves this model |
| `provider_model_id` | `str` | Provider's native model identifier |
| `family` | `str` | Model family (claude, gpt, gemini, llama, etc.) |
| `generation` | `str` | Release generation label |
| `status` | `ModelStatus` | `ACTIVE` / `DEPRECATED` / `BETA` / `PREVIEW` |
| `successor_canonical_id` | `str \| None` | Replacement model if deprecated |
| `context_window_tokens` | `int` | Maximum total context (input + output) |
| `max_output_tokens` | `int` | Maximum response tokens |
| `capabilities` | `CapabilityFlags` | See AI-PROVIDER-CONTRACTS.md |
| `pricing` | `ModelPricing` | See section 6 |
| `latency_class` | `LatencyClass` | `ULTRA_LOW` / `LOW` / `MEDIUM` / `HIGH` |
| `quality_tier` | `QualityTier` | `FRONTIER` / `STANDARD` / `EFFICIENT` / `ECONOMY` |
| `recommended_use_cases` | `List[str]` | Routing hints |
| `not_recommended_for` | `List[str]` | Anti-pattern hints |
| `knowledge_cutoff` | `date \| None` | Training data cutoff |
| `available_regions` | `List[str]` | ISO region codes (for data residency routing) |
| `release_date` | `date` | When model became available |
| `deprecation_date` | `date \| None` | When model will be retired |
| `benchmark_scores` | `BenchmarkScores` | Seeded from Benchmark Engine |

---

## 6. Pricing Schema

### ModelPricing

| Field | Type | Description |
|-------|------|-------------|
| `currency` | `str` | Always `USD` |
| `input_per_million_tokens` | `Decimal` | Cost per 1M input tokens |
| `output_per_million_tokens` | `Decimal` | Cost per 1M output tokens |
| `cache_write_per_million_tokens` | `Decimal \| None` | Prompt cache write cost |
| `cache_read_per_million_tokens` | `Decimal \| None` | Prompt cache read cost |
| `reasoning_per_million_tokens` | `Decimal \| None` | Reasoning/thinking tokens (if billed separately) |
| `image_input_per_image` | `Decimal \| None` | Cost per image in input |
| `image_output_per_image` | `Decimal \| None` | Cost per generated image |
| `audio_input_per_minute` | `Decimal \| None` | Cost per audio input minute |
| `audio_output_per_minute` | `Decimal \| None` | Cost per audio output minute |
| `embedding_per_million_tokens` | `Decimal \| None` | Cost per 1M tokens for embedding models |
| `subscription_tier_override` | `str \| None` | If null cost (Claude Max subscription), note here |
| `pricing_updated_at` | `datetime` | When pricing was last verified |

---

## 7. Latency Classes

| Class | p50 Target | p99 Target | Typical Use Cases |
|-------|-----------|-----------|------------------|
| `ULTRA_LOW` | < 500ms | < 2s | Real-time chat, voice, autocomplete |
| `LOW` | < 2s | < 8s | Interactive assistant, code completion |
| `MEDIUM` | < 10s | < 30s | Document analysis, complex reasoning |
| `HIGH` | < 60s | < 300s | Long document processing, batch jobs |

---

## 8. Quality Tiers

| Tier | Description | Examples |
|------|-------------|---------|
| `FRONTIER` | Best-in-class, highest capability | Claude Opus 4.8, GPT-4o, Gemini 2.5 Pro |
| `STANDARD` | Strong general capability | Claude Sonnet 4.6, GPT-4o mini, Gemini 2.5 Flash |
| `EFFICIENT` | Good capability, optimized for speed/cost | Claude Haiku 4.5, GPT-3.5-turbo equivalent |
| `ECONOMY` | Maximum cost efficiency, basic tasks | Small local models, embedding-only models |

---

## 9. Catalog: Anthropic Claude

### claude-opus-4-8

| Field | Value |
|-------|-------|
| Canonical ID | `anthropic/claude/claude-opus-4-8` |
| Provider Model ID | `claude-opus-4-8` |
| Status | `ACTIVE` |
| Context Window | 200,000 tokens |
| Max Output | 32,000 tokens |
| Quality Tier | `FRONTIER` |
| Latency Class | `MEDIUM` |
| Capabilities | text_generation, chat, vision_image, tool_calling, parallel_tool_calling, structured_output, extended_thinking, file_handling |
| Input Pricing | $15.00 / 1M tokens |
| Output Pricing | $75.00 / 1M tokens |
| Cache Write | $18.75 / 1M tokens |
| Cache Read | $1.50 / 1M tokens |
| Recommended For | Complex reasoning, agentic tasks, long-document analysis, code architecture |
| Subscription Override | Included in Claude Max (null cost) |

### claude-sonnet-4-6

| Field | Value |
|-------|-------|
| Canonical ID | `anthropic/claude/claude-sonnet-4-6` |
| Provider Model ID | `claude-sonnet-4-6` |
| Status | `ACTIVE` |
| Context Window | 200,000 tokens |
| Max Output | 16,000 tokens |
| Quality Tier | `STANDARD` |
| Latency Class | `LOW` |
| Capabilities | text_generation, chat, vision_image, tool_calling, parallel_tool_calling, structured_output, extended_thinking, file_handling |
| Input Pricing | $3.00 / 1M tokens |
| Output Pricing | $15.00 / 1M tokens |
| Cache Write | $3.75 / 1M tokens |
| Cache Read | $0.30 / 1M tokens |
| Recommended For | Balanced quality/cost for most tasks, software engineering agents |
| Subscription Override | Included in Claude Pro (null cost) |

### claude-haiku-4-5

| Field | Value |
|-------|-------|
| Canonical ID | `anthropic/claude/claude-haiku-4-5` |
| Provider Model ID | `claude-haiku-4-5-20251001` |
| Status | `ACTIVE` |
| Context Window | 200,000 tokens |
| Max Output | 8,192 tokens |
| Quality Tier | `EFFICIENT` |
| Latency Class | `ULTRA_LOW` |
| Capabilities | text_generation, chat, vision_image, tool_calling, parallel_tool_calling, structured_output, file_handling |
| Input Pricing | $0.80 / 1M tokens |
| Output Pricing | $4.00 / 1M tokens |
| Recommended For | High-volume classification, real-time interactions, routing decisions |

### claude-fable-5

| Field | Value |
|-------|-------|
| Canonical ID | `anthropic/claude/claude-fable-5` |
| Provider Model ID | `claude-fable-5` |
| Status | `ACTIVE` |
| Context Window | 200,000 tokens |
| Max Output | 32,000 tokens |
| Quality Tier | `FRONTIER` |
| Latency Class | `LOW` |
| Capabilities | text_generation, chat, vision_image, tool_calling, parallel_tool_calling, structured_output, extended_thinking, file_handling |
| Recommended For | Fast frontier-quality tasks, creative writing |

---

## 10. Catalog: OpenAI

### gpt-4o

| Field | Value |
|-------|-------|
| Canonical ID | `openai/gpt/gpt-4o` |
| Provider Model ID | `gpt-4o` |
| Status | `ACTIVE` |
| Context Window | 128,000 tokens |
| Max Output | 16,384 tokens |
| Quality Tier | `FRONTIER` |
| Latency Class | `LOW` |
| Capabilities | text_generation, chat, vision_image, tool_calling, parallel_tool_calling, structured_output, file_handling |
| Input Pricing | $2.50 / 1M tokens |
| Output Pricing | $10.00 / 1M tokens |

### gpt-4o-mini

| Field | Value |
|-------|-------|
| Canonical ID | `openai/gpt/gpt-4o-mini` |
| Context Window | 128,000 tokens |
| Quality Tier | `EFFICIENT` |
| Latency Class | `ULTRA_LOW` |
| Input Pricing | $0.15 / 1M tokens |
| Output Pricing | $0.60 / 1M tokens |

### o3

| Field | Value |
|-------|-------|
| Canonical ID | `openai/o/o3` |
| Quality Tier | `FRONTIER` |
| Latency Class | `HIGH` |
| Capabilities | text_generation, chat, tool_calling, structured_output, extended_thinking |
| Recommended For | Hard math, science, coding competitions, deep reasoning |

### o4-mini

| Field | Value |
|-------|-------|
| Canonical ID | `openai/o/o4-mini` |
| Quality Tier | `STANDARD` |
| Latency Class | `MEDIUM` |
| Capabilities | text_generation, chat, tool_calling, structured_output, extended_thinking |

### text-embedding-3-large

| Field | Value |
|-------|-------|
| Canonical ID | `openai/text-embedding/text-embedding-3-large` |
| Quality Tier | `FRONTIER` |
| Capabilities | embeddings |
| Dimensions | 3,072 (configurable down to 256) |
| Pricing | $0.13 / 1M tokens |

### dall-e-3

| Field | Value |
|-------|-------|
| Canonical ID | `openai/dall-e/dall-e-3` |
| Quality Tier | `FRONTIER` |
| Capabilities | image_generation |
| Pricing | $0.04–$0.12 per image (size/quality dependent) |

### whisper-1

| Field | Value |
|-------|-------|
| Canonical ID | `openai/whisper/whisper-1` |
| Quality Tier | `STANDARD` |
| Capabilities | audio_input (transcription) |
| Pricing | $0.006 / minute |

---

## 11. Catalog: Google Gemini

### gemini-2-5-pro

| Field | Value |
|-------|-------|
| Canonical ID | `google/gemini/gemini-2-5-pro` |
| Context Window | 1,000,000 tokens |
| Quality Tier | `FRONTIER` |
| Latency Class | `MEDIUM` |
| Capabilities | text_generation, chat, vision_image, vision_video, audio_input, tool_calling, structured_output, extended_thinking, file_handling, code_execution, web_search |

### gemini-2-5-flash

| Field | Value |
|-------|-------|
| Canonical ID | `google/gemini/gemini-2-5-flash` |
| Context Window | 1,000,000 tokens |
| Quality Tier | `STANDARD` |
| Latency Class | `LOW` |
| Capabilities | text_generation, chat, vision_image, audio_input, tool_calling, structured_output, extended_thinking |

---

## 12. Catalog: Local Providers

### Ollama (Generic Entry)

| Field | Value |
|-------|-------|
| Canonical ID | `local/ollama/{tag}` |
| Provider ID | `ollama` |
| Status | `ACTIVE` (if Ollama service reachable) |
| Quality Tier | Inherits from base model (e.g., llama-3-3-70b → `STANDARD`) |
| Latency Class | Hardware-dependent (registered at discovery time) |
| Capabilities | text_generation, chat, embeddings (model-dependent) |
| Pricing | $0.00 (local compute) |
| Notes | Canonical ID suffix is the Ollama tag (e.g., `llama3.3:70b`) |

### LM Studio (Generic Entry)

| Field | Value |
|-------|-------|
| Canonical ID | `local/lm-studio/{tag}` |
| Provider ID | `lm-studio` |
| Notes | OpenAI-compatible API; auto-discovered via LM Studio server endpoint |

---

## 13. Catalog: DeepSeek

### deepseek-v3

| Field | Value |
|-------|-------|
| Canonical ID | `deepseek/deepseek/deepseek-v3` |
| Context Window | 128,000 tokens |
| Quality Tier | `FRONTIER` |
| Capabilities | text_generation, chat, tool_calling, structured_output |
| Input Pricing | $0.27 / 1M tokens (cache hit) |
| Output Pricing | $1.10 / 1M tokens |

### deepseek-r1

| Field | Value |
|-------|-------|
| Canonical ID | `deepseek/deepseek/deepseek-r1` |
| Quality Tier | `FRONTIER` |
| Capabilities | text_generation, chat, extended_thinking |

---

## 14. Catalog: Qwen

### qwen-2-5-72b

| Field | Value |
|-------|-------|
| Canonical ID | `qwen/qwen/qwen-2-5-72b` |
| Context Window | 131,072 tokens |
| Quality Tier | `STANDARD` |
| Capabilities | text_generation, chat, tool_calling, structured_output |

---

## 15. Capability Profile Queries

Products and the Routing Engine can query the catalog by capability profile rather than canonical ID:

```
CapabilityProfile:
  required_capabilities: [vision_image, tool_calling]
  minimum_context_window: 100000
  maximum_latency_class: LOW
  maximum_cost_per_million_input_tokens: 5.00
  quality_tier_minimum: STANDARD
  excluded_providers: [openai]      # optional
  required_regions: [eu-west-1]    # optional, for data residency
```

Query returns a ranked list of matching `ModelRecord` objects.

---

## 16. Model Deprecation Protocol

1. Set `status: DEPRECATED` and `deprecation_date` at least 90 days in advance
2. Set `successor_canonical_id` to replacement model
3. AI ROS logs a deprecation warning for any request targeting the deprecated model
4. At `deprecation_date`, model moves to `status: RETIRED`; routing automatically redirects to `successor_canonical_id`
5. Product teams notified via event `model.deprecated` at T-90, T-30, T-7, and T-0

---

## 17. Data Model

Primary table: `ai_ros_models` — see full schema in `AI-RESOURCE-DATABASE.md`.

---

## 18. Events Published

| Event | Trigger |
|-------|---------|
| `model.registered` | New model added to catalog |
| `model.capability.updated` | Capability flags changed |
| `model.pricing.updated` | Pricing updated |
| `model.deprecated` | Model marked deprecated |
| `model.retired` | Model no longer available |

---

## 19. Security

- Pricing data is sensitive; only internal services may update it
- Model catalog read access is unrestricted within AI ROS
- Pricing updates require admin role; all changes are audit-logged

---

## 20. Future Evolution

| Feature | Target |
|---------|--------|
| Automatic pricing sync from provider APIs | Phase 1A.3 |
| Model benchmark auto-population from Benchmark Engine | Phase 1A.3 |
| Provider-reported capability discovery (no manual registration) | Phase 1A.5 |
| Multi-region model routing based on available_regions | Phase 1A.4 |
| Model cost estimation API for budget pre-flight checks | Phase 1A.2 |

---

## 21. Cross References

| Document | Relationship |
|----------|-------------|
| AI-RESOURCE-OS-BLUEPRINT.md | Parent system |
| AI-PROVIDER-CONTRACTS.md | ModelCapabilityDeclaration fed from here |
| AI-BUDGET-ENGINE.md | Uses pricing from this catalog |
| AI-SCHEDULER.md | Uses latency_class for scheduling decisions |
| AI-BENCHMARK-CENTER.md | Writes benchmark_scores back to this catalog |
| AI-RESOURCE-DATABASE.md | Table schema for ai_ros_models |
