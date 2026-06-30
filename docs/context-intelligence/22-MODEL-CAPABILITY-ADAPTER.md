# UICE-DOC-022 — Model Capability Adapter

**Phase:** 3.0D.2
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Model Capability Adapter transforms a compiled context pack into a format
optimized for the specific LLM that will consume it. Different models have
different context windows, different preferred formats, different hallucination
risk profiles, and different token pricing. UICE adapts the pack layout to
maximize utility for each model family.

UICE invariant UI-03: Model Capability Adapter runs before final pack rendering.

---

## Supported Model Families (6)

```
CLAUDE:      Anthropic Claude models (Sonnet, Opus, Haiku)
             Format: YAML preferred; structured reasoning context
             Strengths: long context, reasoning, tool use
             Context window: 200K tokens
             Preferred field order: purpose → behavior → reasoning → examples

GPT:         OpenAI GPT models (GPT-4, GPT-4o, GPT-4o-mini)
             Format: Markdown preferred; numbered lists
             Strengths: instruction following, code generation
             Context window: 128K tokens
             Preferred field order: identity → capabilities → usage → errors

GEMINI:      Google Gemini models (Pro, Flash, Ultra)
             Format: structured text; tables for comparisons
             Strengths: multi-modal, long document processing
             Context window: 1M tokens (Flash)
             Preferred field order: overview → details → constraints

DEEPSEEK:    DeepSeek models (Chat, Coder, R1)
             Format: code-first; technical precision preferred
             Strengths: code generation, mathematical reasoning
             Context window: 64K tokens
             Preferred field order: interface → implementation → examples

QWEN:        Alibaba Qwen models (Qwen2, Qwen2.5)
             Format: bilingual-aware (Chinese + English)
             Strengths: multilingual, instruction following
             Context window: 128K tokens
             Preferred field order: identity → behavior → constraints

LOCAL:       Self-hosted models (LLaMA, Mistral, Phi, etc.)
             Format: minimal — reduce format overhead
             Strengths: privacy, no cost per token
             Context window: varies (8K–128K)
             Preferred field order: key facts only (L3 equivalent)
```

---

## ModelProfile Schema

```yaml
ModelProfile:
  model_id: string                    # e.g., "claude-sonnet-4-6"
  model_family: CLAUDE | GPT | GEMINI | DEEPSEEK | QWEN | LOCAL
  provider: string                    # "anthropic", "openai", etc.

  # Capabilities
  context_window_tokens: int
  supports_structured_output: bool
  supports_tool_use: bool
  supports_json_mode: bool
  reasoning_capability: BASIC | ADVANCED | EXPERT

  # Risk profile
  hallucination_risk: LOW | MEDIUM | HIGH
    # HIGH: local models, smaller models
    # MEDIUM: standard GPT-4, Gemini Flash
    # LOW: Claude Opus, GPT-4o, Gemini Ultra

  # Formatting preferences
  preferred_format: YAML | MARKDOWN | JSON | PLAIN
  preferred_field_order: list[str]    # ordered field keys
  max_guard_tokens: int               # model-specific guard budget

  # Pricing
  token_cost_per_1k_input: float      # USD
  token_cost_per_1k_output: float     # USD
```

---

## Adaptation Algorithm

```
ADAPT_FOR_MODEL(context_pack, model_profile) → ModelAdaptedPack:

  // Step 1: Context window check
  if context_pack.total_token_count > model_profile.context_window_tokens:
    // Force further compression to fit
    context_pack = FORCE_COMPRESS(context_pack, model_profile.context_window_tokens * 0.80)

  // Step 2: Format transformation
  adapted = TRANSFORM_FORMAT(context_pack, model_profile.preferred_format):
    YAML:     preserve YAML structure, add type annotations
    MARKDOWN: convert to markdown headers and lists
    JSON:     serialize to JSON with schema annotations
    PLAIN:    strip YAML/JSON syntax, use indented plain text

  // Step 3: Field reordering
  for slot in adapted.all_slots:
    slot.fields = REORDER_FIELDS(slot.fields, model_profile.preferred_field_order)

  // Step 4: Guard adjustment for hallucination risk
  if model_profile.hallucination_risk == HIGH:
    // Add extra guard warnings for high-risk models
    adapted.guard.intent_warnings.append(IntentWarning(
      warning_type = SCOPE_BOUNDARY,
      message = "This model has elevated hallucination risk. Cross-check all facts.",
      severity = HIGH,
    ))
    // Include more errors in guard (relax QAGB filter slightly)
    adapted.guard = EXPAND_GUARD(adapted.guard, model_profile.max_guard_tokens)

  elif model_profile.hallucination_risk == LOW:
    // Tighter guard for capable models (they handle nuance better)
    // No extra warnings needed beyond QAGB output

  // Step 5: Token budget re-validation
  if adapted.total_token_count > model_profile.context_window_tokens:
    RAISE_BUDGET_ERROR(model_profile.model_id, adapted.total_token_count)

  adapted.model_profile_id = model_profile.model_id
  return adapted
```

---

## Format Transformation Examples

```
YAML (Claude preferred):
  dna:
    identity: "platform.service.payment-gateway"
    purpose: "Process payment transactions"
  reasoning:
    requires: ["KNW-PLATFORM-DB-001", "KNW-PLATFORM-AUTH-001"]

MARKDOWN (GPT preferred):
  ## Payment Gateway
  **Purpose:** Process payment transactions
  **Requires:** DB Service, Auth Service

JSON (structured output):
  {"identity": "platform.service.payment-gateway",
   "purpose": "Process payment transactions",
   "requires": ["KNW-PLATFORM-DB-001", ...]}

PLAIN (Local model):
  Payment Gateway: processes payment transactions
  requires: DB Service, Auth Service
```

---

## LOCAL Model Special Handling

```
Local models often have:
  - Smaller context windows (8K–32K)
  - Higher hallucination risk
  - No JSON mode

Adaptations:
  - Force MINI or MICRO summary mode for all context slots
  - Use PLAIN format (minimal overhead)
  - Maximum guard block: 50 tokens
  - Add "DO NOT fabricate" statement to guard
  - Reduce max_objects to 3
```

---

## Rules

| Rule | Statement |
|------|-----------|
| UICE-106 | Model Capability Adapter runs before final rendering (UI-03) — adapted pack is what gets sent |
| UICE-107 | If context_pack.total_tokens > model_profile.context_window_tokens, adapter must compress further |
| UICE-108 | HIGH hallucination_risk models always receive expanded guard — QAGB relaxed for these models |
| UICE-109 | Format transformation must be lossless — no information may be dropped during format conversion |
| UICE-110 | LOCAL model profile must set max_objects ≤ 3 and preferred_format = PLAIN |

---

## Cross-References

- Token Budget Manager → `10-TOKEN-BUDGET-MANAGER`
- Adaptive Guard Generator → `09-ADAPTIVE-GUARD-GENERATOR`
- Cost Optimizer → `23-COST-OPTIMIZER`
- Context Verifier → `26-CONTEXT-VERIFIER`
