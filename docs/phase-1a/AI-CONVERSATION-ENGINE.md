# AI Conversation Engine — Session Management and Context

**Version:** 1.0.0  
**Status:** Approved for Implementation  
**Phase:** 1A  
**Date:** 2026-06-29  
**Classification:** Internal Architecture

---

## 1. Purpose

The Conversation Engine owns the complete lifecycle of all AI conversations across AI Studio: session creation, message history, context window management, conversation branching, shared context, history compression, and memory integration. Products interact with AI ROS through session tokens — they never manage conversation history directly.

---

## 2. Responsibilities

- Session creation, validation, and expiry
- Persistent message history storage and retrieval
- Context window budget management for each execution
- Automatic history compression when approaching model context limits
- Conversation branching (fork at any point to explore alternatives)
- Shared context injection (system-level context shared across turns)
- Cross-session memory injection via Memory Integration
- Conversation search and archival
- Export formats (JSON, Markdown, PDF)

---

## 3. Architecture

```
Product (via AI ROS Client SDK)
        │ session_id
        ▼
┌──────────────────────────────────────────────────────────────────────┐
│                      Conversation Engine                               │
│                                                                        │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐   │
│  │  Session Manager │  │  History Store   │  │  Context Engine  │   │
│  │                  │  │                  │  │  (assembler)     │   │
│  └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘   │
│           │                     │                      │              │
│  ┌────────▼─────────────────────▼──────────────────────▼───────────┐  │
│  │                    Conversation Core                               │  │
│  │                                                                    │  │
│  │  create_session(org_id, product_id, config) → session_id         │  │
│  │  add_message(session_id, message) → void                         │  │
│  │  assemble_context(session_id, model_id) → AssembledContext       │  │
│  │  branch_session(session_id, from_message_id) → new_session_id   │  │
│  │  compress_history(session_id) → void                             │  │
│  │  archive_session(session_id) → void                              │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐   │
│  │  Compression     │  │  Branch Manager  │  │  Memory          │   │
│  │  Engine          │  │                  │  │  Integration     │   │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 4. Data Models

### 4.1 ConversationSession

| Field | Type | Description |
|-------|------|-------------|
| `session_id` | `UUID` | Primary key; also the public session token |
| `org_id` | `UUID` | |
| `project_id` | `UUID \| None` | |
| `user_id` | `UUID \| None` | |
| `product_id` | `str` | Which AI Studio product owns this session |
| `status` | `SessionStatus` | `ACTIVE` / `IDLE` / `ARCHIVED` / `EXPIRED` |
| `parent_session_id` | `UUID \| None` | Set for branched sessions |
| `branch_point_message_id` | `UUID \| None` | Message ID where branch was taken |
| `title` | `str \| None` | Auto-generated or user-set |
| `system_prompt` | `str \| None` | Shared system prompt for all turns |
| `config` | `SessionConfig` | Session-level configuration |
| `message_count` | `int` | Cached count |
| `total_input_tokens` | `int` | Cumulative tokens |
| `total_output_tokens` | `int` | |
| `total_cost_usd` | `Decimal` | Cumulative cost |
| `created_at` | `datetime` | |
| `last_active_at` | `datetime` | Updated on every message |
| `expires_at` | `datetime \| None` | Auto-expiry time |
| `archived_at` | `datetime \| None` | |
| `metadata` | `dict` | Product-specific metadata (opaque to Conversation Engine) |

### 4.2 SessionConfig

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `max_history_messages` | `int \| None` | 200 | Hard cap on message history |
| `compression_trigger_pct` | `float` | 0.85 | Compress when context is 85% full |
| `compression_target_pct` | `float` | 0.50 | Compress down to 50% of context window |
| `compression_strategy` | `CompressionStrategy` | `ROLLING_SUMMARY` | How to compress |
| `memory_injection_enabled` | `bool` | True | Inject relevant memories before each turn |
| `memory_injection_max_tokens` | `int` | 2000 | Token budget for memory injection |
| `auto_title` | `bool` | True | Auto-generate session title from first message |
| `idle_expiry_hours` | `int \| None` | 72 | Auto-expire after N hours of inactivity |
| `export_format` | `str` | `json` | Default export format |

### 4.3 Message

| Field | Type | Description |
|-------|------|-------------|
| `message_id` | `UUID` | Primary key |
| `session_id` | `UUID` | |
| `sequence_number` | `int` | Ordering within session (monotonically increasing) |
| `role` | `MessageRole` | `USER` / `ASSISTANT` / `TOOL_RESULT` / `SYSTEM` |
| `content` | `List[ContentBlock]` | Multimodal content (see AI-PROVIDER-CONTRACTS.md) |
| `token_count` | `int \| None` | Counted at storage time |
| `linked_request_id` | `UUID \| None` | AI ROS request that generated this message |
| `model_used` | `str \| None` | Which model generated this (for assistant messages) |
| `latency_ms` | `int \| None` | Generation latency (assistant messages only) |
| `is_compressed_summary` | `bool` | True if this message is a compression artifact |
| `compressed_from_range` | `dict \| None` | `{start_seq, end_seq}` if compression artifact |
| `created_at` | `datetime` | |
| `metadata` | `dict` | Application metadata |

---

## 5. Session Lifecycle

```
create_session()
      │
      ▼
  ACTIVE ──── add_message() (repeating)
      │             │
      │             ▼
      │    assemble_context()  →  Execution Engine
      │             │
      │    (when approaching context limit)
      │             ▼
      │    compress_history()
      │
      ├── branch_session() → new ACTIVE session
      │
      ├── (no activity for idle_expiry_hours)
      ▼
  IDLE → auto-archive → ARCHIVED
      │
      └── (manual expire / TTL)
          ▼
      EXPIRED (deleted after retention period)
```

---

## 6. Context Assembly

`assemble_context(session_id, model_id) → AssembledContext`

The Context Engine assembles the full context for an execution in a defined priority order:

```
1. System prompt             (from session.system_prompt or product default)
2. Memory injections         (from Memory Integration, highest-relevance first)
3. Tool definitions          (from Tool Runtime, if any)
4. Compression summaries     (earliest summaries, if history was compressed)
5. Recent message history    (most recent messages, newest last)
6. [slot reserved for current user message — not yet received]

Total must fit within: model.context_window_tokens - model.max_output_tokens - safety_margin
```

### 6.1 Token Budget Enforcement

```
available_for_history = context_window - system_tokens - memory_tokens - tool_tokens - output_reserve - safety_margin

safety_margin = 1000 tokens  (buffer for tokenizer inaccuracy)
output_reserve = min(requested_max_tokens, model.max_output_tokens)
```

If `available_for_history < 0`, the Context Engine reduces in this order:
1. Trim memory injections (least relevant first)
2. Trigger `compress_history()` if not already compressed
3. Remove oldest message pairs (keep at least 2 turns)
4. Return error if still insufficient after all trims

### 6.2 AssembledContext

| Field | Type | Description |
|-------|------|-------------|
| `messages` | `List[Message]` | Final ordered message list |
| `system_prompt` | `str \| None` | |
| `tool_definitions` | `List[ToolDefinition]` | |
| `total_estimated_tokens` | `int` | |
| `context_fill_pct` | `float` | How full the context window is |
| `messages_included` | `int` | How many messages made it |
| `messages_excluded` | `int` | How many were trimmed |
| `memory_tokens_used` | `int` | |

---

## 7. History Compression

When the context window is `compression_trigger_pct` full, the Compression Engine summarizes older history.

### 7.1 Compression Strategies

| Strategy | Behavior |
|----------|---------|
| `ROLLING_SUMMARY` | Summarize the oldest 50% of messages into a single summary message |
| `HIERARCHICAL` | Build a tree of summaries (oldest → chapter summary → arc summary) |
| `SELECTIVE_DROP` | Drop messages below a relevance threshold (preserves verbatim recent context) |
| `PROVIDER_CACHE` | For Claude: use prompt caching to reduce cost of long histories without compressing |

### 7.2 Summary Generation

Summary is generated by a lightweight AI call (typically Claude Haiku or GPT-4o mini):

```
System: "You are summarizing a conversation for a context-limited AI assistant.
        Preserve: all decisions made, key facts, user preferences, open questions.
        Omit: repetitions, pleasantries, exploratory messages that were superseded."

User: [The messages to be summarized, as JSON]
```

The generated summary is stored as a `Message` with `is_compressed_summary=True` and `compressed_from_range={start_seq, end_seq}`.

Original messages are retained in the database but marked with `is_archived=True` — they are excluded from context assembly but available for history browsing and export.

---

## 8. Conversation Branching

`branch_session(session_id, from_message_id) → new_session_id`

Creates a new session that shares all history up to and including `from_message_id`. The branched session is fully independent from that point forward.

Use cases:
- Explore an alternative response
- Try a different instruction without losing the original thread
- A/B test different prompting approaches on the same conversation
- Create a "what-if" branch for simulation

### Branch Data Model

The branched session has:
- `parent_session_id = original_session_id`
- `branch_point_message_id = from_message_id`
- Messages with `sequence_number ≤ from_message.sequence_number` are shared via reference (not duplicated)
- Messages added after branching are stored separately per session

---

## 9. Shared Context

Products can inject shared context that persists across all turns of a session without appearing in the message history:

| Type | Description | Example |
|------|-------------|---------|
| `SYSTEM_PROMPT` | Persistent instruction set | "You are the Mythic Realms Economy Agent..." |
| `CONTEXT_DOCUMENT` | Background document injected at session start | Project requirements, game rulebook |
| `TOOL_DEFINITIONS` | Available tools for this session | MythicRealms API tools |

Shared context is injected before message history in context assembly.

---

## 10. Memory Integration

Before each turn, the Memory Integration layer retrieves relevant memories:

```
Conversation Engine → Memory Integration:
  retrieve_relevant_memories(
    session_id,
    current_user_message,
    max_tokens=config.memory_injection_max_tokens
  ) → List[MemoryFragment]
```

Returned memories are injected into the system prompt or as a special `MEMORY` role message (configurable).

After each turn:
```
Conversation Engine → Memory Integration:
  record_experience(
    session_id,
    exchange = {user_message, assistant_response},
    outcome_metadata
  )
```

---

## 11. APIs

### Session Management

```
create_session(
  org_id, product_id, config: SessionConfig,
  project_id=None, user_id=None, system_prompt=None,
  metadata={}
) → session_id: UUID

get_session(session_id) → ConversationSession

list_sessions(org_id, project_id=None, user_id=None,
              status=ACTIVE, page=0, page_size=50) → List[SessionSummary]

update_session(session_id, title=None, config=None) → void

archive_session(session_id) → void

delete_session(session_id) → void  # admin only; permanent

branch_session(session_id, from_message_id) → new_session_id

export_session(session_id, format: "json"|"markdown"|"pdf") → bytes
```

### Message Operations

```
add_user_message(session_id, content: List[ContentBlock]) → message_id: UUID

get_messages(session_id, after_sequence=0, limit=50) → List[Message]

delete_message(session_id, message_id) → void  # admin only
```

### Context Operations

```
assemble_context(session_id, model_id, requested_max_tokens=None) → AssembledContext

compress_history(session_id) → void  # manual trigger

get_context_position(session_id, model_id) → ContextPosition
```

---

## 12. Events Published

| Event | Trigger |
|-------|---------|
| `conversation.session.created` | New session |
| `conversation.session.archived` | Session archived |
| `conversation.session.expired` | Session expired |
| `conversation.session.branched` | Branch created |
| `conversation.message.added` | User message added |
| `conversation.response.recorded` | Assistant response stored |
| `conversation.history.compressed` | Compression ran |
| `conversation.context.overflow` | Context window would overflow; trimming applied |

---

## 13. Data Model

Tables: `ai_ros_conversations`, `ai_ros_messages`. Full schema in `AI-RESOURCE-DATABASE.md`.

Message content stored as JSONB. Large content blocks (images, audio) stored in object storage (S3-compatible); JSONB stores reference URL only.

---

## 14. Security

- Sessions are scoped to `(org_id, user_id)` — users cannot access other users' sessions
- Session tokens (UUIDs) are not guessable; additional MAC validation for sensitive products
- Message content is encrypted at rest (column-level encryption for sensitive orgs)
- Exported sessions include all content; export is audit-logged

---

## 15. Scalability

- Hot sessions (active in last hour) cached in Redis (session metadata + last 20 messages)
- Cold sessions fetched from PostgreSQL on demand
- Message JSONB content indexed for full-text search (pg_trgm)
- Session archival runs nightly for sessions idle > `idle_expiry_hours`

---

## 16. Future Evolution

| Feature | Notes |
|---------|-------|
| Multi-participant sessions | Multiple users + agents in one conversation |
| Real-time collaborative editing | Live session sharing (WebSocket) |
| Cross-session knowledge graph | Link related sessions by topic |
| Automatic session titling | AI-generated title from first exchange |
| Conversation analytics | Topic clustering, sentiment, outcome tracking |

---

## 17. Cross References

| Document | Relationship |
|----------|-------------|
| AI-RESOURCE-OS-BLUEPRINT.md | Parent system |
| AI-PROVIDER-CONTRACTS.md | Message and ContentBlock types |
| AI-RESOURCE-DATABASE.md | Table schemas |
| AI-RESOURCE-EVENTS.md | Event schemas |
| AI-RESOURCE-DESKTOP.md | Conversation Explorer UI |
