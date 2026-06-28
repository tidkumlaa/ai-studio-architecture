# AI Scheduler — Request Scheduling and Execution Coordination

**Version:** 1.0.0  
**Status:** Approved for Implementation  
**Phase:** 1A  
**Date:** 2026-06-29  
**Classification:** Internal Architecture

---

## 1. Purpose

The AI Scheduler manages the complete lifecycle of AI execution requests from admission to completion: priority queuing, rate limiting, provider selection coordination, parallel execution, retry policies, timeout enforcement, and cancellation. It is the traffic controller between the Routing Engine and the Execution Engine.

---

## 2. Responsibilities

- Accept requests admitted by the Routing Engine (post quota/budget/policy checks)
- Prioritize requests using configurable priority queues
- Enforce rate limits per provider and per organization
- Coordinate parallel execution across providers
- Implement retry with exponential backoff and jitter
- Enforce per-request and per-session timeouts
- Handle cancellation (caller-initiated and system-initiated)
- Track request lifecycle events for observability
- Support multiple scheduling modes: immediate, queued, batch, scheduled, recurring

---

## 3. Architecture

```
Routing Engine
      │ ScheduleRequest(routed_request)
      ▼
┌──────────────────────────────────────────────────────────────────────┐
│                         AI Scheduler                                   │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                      Admission Control                             │  │
│  │  Mode check │ Rate limit pre-check │ Queue capacity check         │  │
│  └─────────────────────────────┬────────────────────────────────────┘  │
│                                 │                                       │
│  ┌──────────────┐  ┌────────────▼──────────────┐  ┌────────────────┐  │
│  │  Immediate   │  │    Priority Queue           │  │  Batch         │  │
│  │  Dispatcher  │  │    (Redis sorted set)       │  │  Collector     │  │
│  └──────┬───────┘  └────────────┬───────────────┘  └────────┬───────┘  │
│         │                       │                            │          │
│  ┌──────▼───────────────────────▼────────────────────────────▼───────┐  │
│  │                       Execution Coordinator                         │  │
│  │                                                                     │  │
│  │  dispatch_single(request) → Future[AIResponse]                     │  │
│  │  dispatch_parallel(requests) → List[Future[AIResponse]]            │  │
│  │  retry_with_backoff(request, policy) → AIResponse                  │  │
│  └─────────────────────────────┬───────────────────────────────────────┘  │
│                                 │                                        │
│  ┌──────────────┐  ┌────────────▼─────────┐  ┌───────────────────────┐  │
│  │  Rate Limit  │  │  Timeout Enforcer    │  │  Cancellation Manager │  │
│  │  Tracker     │  │                      │  │                       │  │
│  └──────────────┘  └──────────────────────┘  └───────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
         │
         ▼
  Execution Engine
```

---

## 4. Scheduling Modes

### 4.1 IMMEDIATE

Request must be executed now or fail. No queuing.

- Used for: interactive chat, real-time tool calls, streaming responses
- On provider unavailable: fail fast with structured error
- Timeout: per-request (typically 30–120 seconds)

### 4.2 QUEUED

Request is enqueued with priority and executed when capacity is available.

- Used for: background document processing, batch content generation, agent tasks
- Queue backed by Redis sorted set (score = priority + submission_time)
- Worker pool pulls from queue; configurable concurrency per provider
- Timeout: queue wait timeout + execution timeout (separate values)

### 4.3 BATCH

Multiple requests collected and submitted together to a provider batch API.

- Used for: embedding generation, classification at scale, offline evaluation
- Collector accumulates requests up to batch size limit or time window (whichever hits first)
- Submitted as single batch job; results polled or delivered via webhook
- Cost benefit: OpenAI Batch API offers 50% discount

### 4.4 SCHEDULED

Request executes at a specific future time.

- Used for: scheduled reports, overnight processing, time-zone-aware workflows
- Backed by a cron-style scheduler (database-persisted schedule)
- Supports one-time and recurring

### 4.5 RECURRING

Request repeats on a cron expression.

- Used for: daily benchmark runs, periodic summarization, subscription digest content
- Managed by the Scheduled Job Registry
- Each firing creates a new QUEUED request

---

## 5. Priority Queue

### 5.1 Priority Levels

| Level | Value | Typical Use |
|-------|-------|-------------|
| `CRITICAL` | 100 | System-critical (health checks, approval notifications) |
| `HIGH` | 75 | Interactive user requests, streaming responses |
| `STANDARD` | 50 | Default priority for most product requests |
| `LOW` | 25 | Background processing, batch jobs |
| `ECONOMY` | 10 | Scheduled overnight jobs, non-urgent analytics |

Products assign priority when submitting requests. Organizations can configure maximum allowed priority per project (preventing background jobs from claiming CRITICAL priority).

### 5.2 Queue Score Calculation

```
score = (MAX_PRIORITY - priority_level) + (submission_timestamp_ns / 1e12)
```

Lower score = dequeued first. This ensures:
- Higher priority requests always execute before lower priority
- Within same priority level: FIFO ordering by submission time

### 5.3 Queue Capacity Limits

| Config | Default | Description |
|--------|---------|-------------|
| `max_queue_depth_per_org` | 1,000 | Per-org queue depth limit |
| `max_queue_depth_total` | 50,000 | System-wide queue depth limit |
| `max_queue_wait_seconds` | 3,600 | Request fails if not dispatched within this window |

When queue is full for an org, new QUEUED requests are rejected with `QUEUE_FULL` error.

---

## 6. Rate Limiting

The Scheduler enforces AI ROS-side rate limits per provider (separate from provider-imposed quota, which the Quota Engine tracks).

### 6.1 Rate Limit Tiers

Per-provider, per-AI-ROS-instance rate limits prevent overloading providers even before provider-side 429s:

| Tier | Requests/sec | Tokens/sec | Use |
|------|-------------|-----------|-----|
| Conservative | 2 | 5,000 | During provider degradation |
| Standard | 10 | 50,000 | Normal operation |
| Aggressive | 50 | 200,000 | High-quota enterprise accounts |

Active tier per provider is set dynamically by the Provider Health Monitor.

### 6.2 Rate Limit Algorithm

Token bucket algorithm per (org_id, provider_id):
- Bucket refills at configured rate
- Each request consumes tokens proportional to estimated input tokens
- If bucket empty: IMMEDIATE requests fail; QUEUED requests wait

---

## 7. Provider Selection for Scheduling

After the Routing Engine has identified candidate providers ranked by score, the Scheduler makes the final selection based on current runtime state:

1. **Concurrency check**: Is this provider's concurrent request slot available?
2. **Rate limit check**: Does the token bucket have sufficient capacity?
3. **Circuit breaker**: Is the circuit CLOSED (healthy)?
4. If primary provider fails all checks: try next candidate
5. If all candidates unavailable: return `PROVIDER_UNAVAILABLE` error

---

## 8. Parallel Execution

### 8.1 Parallel Dispatch

The Scheduler supports racing multiple providers and returning the first successful response:

```
dispatch_parallel(
  requests: List[RoutedRequest],
  strategy: ParallelStrategy
) → AIResponse
```

**ParallelStrategy options:**

| Strategy | Behavior |
|----------|---------|
| `RACE` | Dispatch all simultaneously; return first success; cancel losers |
| `HEDGE` | Start with primary; after hedge_delay_ms, start backup if primary hasn't returned |
| `ALL` | Dispatch all; return all results (for benchmark comparison) |

### 8.2 Hedged Requests

Hedging reduces p99 latency by starting a backup request after a configurable delay if the primary hasn't responded:

```
hedge_delay_ms: 2000  # Start backup if primary doesn't return within 2 seconds
```

On primary success: cancel backup (if provider supports cancellation) or absorb the cost.

---

## 9. Retry Policy

### 9.1 RetryPolicy

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `max_attempts` | `int` | 3 | Total attempts including initial |
| `initial_delay_ms` | `int` | 1,000 | Wait before first retry |
| `max_delay_ms` | `int` | 30,000 | Maximum wait between retries |
| `backoff_multiplier` | `float` | 2.0 | Exponential backoff factor |
| `jitter_factor` | `float` | 0.25 | Random jitter to spread retries |
| `retry_on` | `List[ProviderErrorCode]` | `[RATE_LIMITED, TIMEOUT, SERVICE_UNAVAILABLE, CAPACITY_OVERLOADED, STREAM_INTERRUPTED]` | Error codes that trigger retry |
| `change_provider_on_retry` | `bool` | `False` | If True: retry on next candidate provider |

### 9.2 Backoff Calculation

```
delay = min(initial_delay_ms * (multiplier ^ attempt), max_delay_ms)
jitter = delay * jitter_factor * random(-1, 1)
actual_delay = delay + jitter

# Honor provider Retry-After header if present:
actual_delay = max(actual_delay, retry_after_seconds * 1000)
```

### 9.3 Retry Budget

Each organization has a retry budget per minute to prevent retries from amplifying load during provider degradation:

```
max_retries_per_minute_per_org: 100  (default)
```

If the retry budget is exhausted, retries are dropped and the request fails immediately.

---

## 10. Timeout Enforcement

### 10.1 Timeout Hierarchy

| Timeout | Default | Scope |
|---------|---------|-------|
| `connect_timeout_ms` | 5,000 | TCP connection to provider |
| `request_timeout_ms` | 60,000 | Full response (non-streaming) |
| `stream_first_token_ms` | 10,000 | Time to first token (streaming) |
| `stream_idle_timeout_ms` | 30,000 | Max gap between chunks (streaming) |
| `queue_wait_timeout_ms` | 3,600,000 | Max time in QUEUED mode |
| `total_session_timeout_ms` | None | Optional: cap entire multi-turn session |

All timeouts are configurable per request and per organization policy.

### 10.2 Timeout Handling

On timeout:
1. Cancel the in-flight request (if provider supports cancellation)
2. Return `TIMEOUT` error to caller
3. Publish `execution.timeout` event
4. Refund quota and budget reservations (via Quota Engine and Budget Engine)
5. Increment provider timeout metric (feeds Circuit Breaker)

---

## 11. Cancellation

### 11.1 Caller-Initiated Cancellation

```
cancel_request(request_id: UUID, reason: str) → void
```

- QUEUED requests: remove from queue immediately; no quota/budget deduction
- IN_FLIGHT requests: send cancellation to provider (best-effort); record partial usage
- STREAMING requests: close stream; record tokens received so far

### 11.2 System-Initiated Cancellation

Triggered by:
- Organization budget exhaustion (cancel all in-flight requests for that org)
- Provider circuit breaker opening (cancel requests routed to that provider)
- Graceful shutdown (drain queue; cancel in-flight with `SYSTEM_SHUTDOWN` reason)

---

## 12. Request Lifecycle

```
SUBMITTED → ROUTING → QUEUED → DISPATCHING → IN_FLIGHT → COMPLETED
                                                        → FAILED
                                                        → TIMED_OUT
                                                        → CANCELLED
              → FAILED (quota/budget/policy rejection)
```

Each state transition is recorded in `ai_ros_requests` and emits an event.

---

## 13. Data Model

### ai_ros_requests

| Column | Type | Description |
|--------|------|-------------|
| `request_id` | UUID | PK |
| `org_id` | UUID | |
| `project_id` | UUID | |
| `user_id` | UUID | |
| `session_id` | UUID \| None | Conversation session |
| `mode` | enum | `IMMEDIATE` / `QUEUED` / `BATCH` / `SCHEDULED` |
| `priority` | int | |
| `status` | enum | See lifecycle above |
| `provider_id` | str \| None | Assigned provider |
| `canonical_model_id` | str \| None | Assigned model |
| `submitted_at` | timestamptz | |
| `dispatched_at` | timestamptz \| None | |
| `completed_at` | timestamptz \| None | |
| `attempts` | int | Retry count |
| `error_code` | str \| None | |
| `queue_wait_ms` | int \| None | Time from submit to dispatch |
| `execution_ms` | int \| None | Provider latency |

---

## 14. Events Published

| Event | Trigger |
|-------|---------|
| `execution.submitted` | Request admitted to scheduler |
| `execution.queued` | Request placed in priority queue |
| `execution.dispatched` | Request sent to Execution Engine |
| `execution.completed` | Successful response received |
| `execution.failed` | Terminal failure after all retries |
| `execution.timeout` | Request timed out |
| `execution.cancelled` | Request cancelled |
| `execution.retry` | Retry attempt initiated |
| `execution.queue_full` | Queue capacity exceeded, request rejected |

---

## 15. Security

- Priority levels are enforced per organization policy; products cannot set CRITICAL priority without explicit grant
- Cancellation requires same org_id as original submission
- Queue inspection (list in-flight requests) requires admin role

---

## 16. Scalability

- Priority queue backed by Redis sorted sets (sub-millisecond enqueue/dequeue)
- Execution Coordinator uses asyncio / thread pool for concurrent dispatches
- Queue worker pool scales horizontally; each worker is stateless
- Rate limit token buckets stored in Redis (atomic INCR + EXPIRE)

---

## 17. Future Evolution

| Feature | Notes |
|---------|-------|
| Priority boost for expiring requests | Auto-boost priority as `queue_wait_timeout_ms` approaches |
| Cross-instance queue visibility | All AI ROS instances share one Redis queue |
| Predictive scheduling | ML-based provider selection accounting for predicted availability |
| Provider warmup requests | Pre-warm slow providers before batch jobs |

---

## 18. Cross References

| Document | Relationship |
|----------|-------------|
| AI-RESOURCE-OS-BLUEPRINT.md | Parent system |
| AI-QUOTA-ENGINE.md | Pre-dispatch quota gate |
| AI-BUDGET-ENGINE.md | Pre-dispatch budget gate |
| AI-POLICY-ENGINE.md | Pre-dispatch policy gate |
| AI-PROVIDER-CONTRACTS.md | Execution targets |
| AI-RESOURCE-EVENTS.md | Event schemas |
| AI-RESOURCE-DATABASE.md | ai_ros_requests table |
