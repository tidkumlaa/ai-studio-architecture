---
knowledge_id: KNW-PLAT-ARCH-019
title: "Platform State Machines"
status: AUTHORITATIVE
phase: "2.1D.0"
version: "1.0.0"
created: "2026-06-29"
purpose: "Define all platform state machines: runtime lifecycle, migration, provider health, and execution"
canonical_source: "architecture/docs/platform-consolidation/19-STATE-MACHINES.md"
dependencies:
  - "06-RUNTIME-MODEL.md"
  - "18-EVENT-MODEL.md"
related_documents:
  - "22-MIGRATION-ENGINE.md"
  - "24-ROLLBACK-ENGINE.md"
acceptance_criteria:
  - "Every state machine has a complete set of states, transitions, and guards"
  - "Terminal states are identified"
  - "Events emitted on each transition are specified"
verification_checklist:
  - "[ ] All state machines are deterministic"
  - "[ ] No unreachable states"
  - "[ ] Each transition specifies guard conditions"
future_extensions:
  - "State machine visualization in 28-DASHBOARD.md"
  - "State history replay for debugging"
---

# Platform State Machines

## 1. Runtime Lifecycle State Machine

### States

| State | Meaning |
|-------|---------|
| `UNINITIALIZED` | Not yet created |
| `INITIALIZING` | Boot sequence running |
| `READY` | Fully operational |
| `DEGRADED` | Operational but with errors |
| `STOPPING` | Shutdown in progress |
| `STOPPED` | Fully stopped |
| `FAILED` | Boot or runtime error; requires restart |

### Transitions

```
UNINITIALIZED  ──initialize()──▶  INITIALIZING
INITIALIZING   ──[success]──────▶  READY
INITIALIZING   ──[error]────────▶  FAILED
READY          ──degrade()──────▶  DEGRADED
DEGRADED       ──recover()──────▶  READY
DEGRADED       ──stop()─────────▶  STOPPING
READY          ──stop()─────────▶  STOPPING
STOPPING       ──[complete]─────▶  STOPPED
STOPPING       ──[error]────────▶  FAILED
FAILED         ──reset()────────▶  UNINITIALIZED
```

### Events Emitted

| Transition | Event |
|------------|-------|
| `INITIALIZING → READY` | `platform.kernel.runtime.started` |
| `READY → STOPPED` | `platform.kernel.runtime.stopped` |
| `* → FAILED` | `platform.kernel.runtime.failed` |

### Guard Conditions

| Transition | Guard |
|------------|-------|
| `UNINITIALIZED → INITIALIZING` | Kernel must be in READY state |
| `INITIALIZING → READY` | All required capabilities registered |
| `READY → STOPPING` | No active requests in flight (or timeout 30s) |

---

## 2. Migration State Machine

### States

| State | Meaning |
|-------|---------|
| `IDLE` | No migration in progress |
| `PLANNING` | Architecture phase — spec being written |
| `VALIDATED` | Architecture frozen and verified |
| `MIGRATING` | File moves / import rewrites in progress |
| `VERIFYING` | Post-migration checks running |
| `COMPLETE` | Migration successful |
| `ROLLBACK` | Reverting to previous state |
| `FAILED` | Migration failed and cannot auto-rollback |

### Transitions

```
IDLE        ──start_planning()──▶  PLANNING
PLANNING    ──freeze()──────────▶  VALIDATED
VALIDATED   ──begin()───────────▶  MIGRATING
MIGRATING   ──verify()──────────▶  VERIFYING
VERIFYING   ──[all_pass]────────▶  COMPLETE
VERIFYING   ──[any_fail]────────▶  ROLLBACK
MIGRATING   ──[error]───────────▶  ROLLBACK
ROLLBACK    ──[success]─────────▶  IDLE
ROLLBACK    ──[fail]────────────▶  FAILED
COMPLETE    ──reset()───────────▶  IDLE
```

### Guard Conditions

| Transition | Guard |
|------------|-------|
| `PLANNING → VALIDATED` | All 37 architecture documents complete |
| `VALIDATED → MIGRATING` | Git working tree clean; snapshot taken |
| `VERIFYING → COMPLETE` | All 25+ verification checks pass |
| `VERIFYING → ROLLBACK` | Any verification check fails |

### Phase Mapping

| Migration State | Phase |
|-----------------|-------|
| `IDLE` | Pre-migration |
| `PLANNING` | Phase 2.1D.0 (current) |
| `VALIDATED` | Phase 2.1D.0 → 2.1D.1 boundary |
| `MIGRATING` | Phase 2.1D.1 |
| `VERIFYING` | Phase 2.1D.1 post-step |
| `COMPLETE` | Phase 2.1D.2+ |

---

## 3. Provider Health State Machine

### States

| State | Meaning |
|-------|---------|
| `HEALTHY` | Normal operation |
| `DEGRADED` | Elevated error rate or latency |
| `CIRCUIT_OPEN` | Requests blocked; recovery probes only |
| `RECOVERING` | Probe successful; warming up |
| `UNAVAILABLE` | No response; fully blocked |

### Transitions

```
HEALTHY     ──[error_rate > 5%]───▶  DEGRADED
HEALTHY     ──[latency > 10s]──────▶  DEGRADED
DEGRADED    ──[error_rate > 20%]───▶  CIRCUIT_OPEN
DEGRADED    ──[error_rate < 2%]────▶  HEALTHY
CIRCUIT_OPEN──[probe_success]───────▶  RECOVERING
CIRCUIT_OPEN──[timeout 60s]────────▶  UNAVAILABLE
RECOVERING  ──[3 consecutive ok]───▶  HEALTHY
RECOVERING  ──[probe_fail]──────────▶  CIRCUIT_OPEN
UNAVAILABLE ──[probe_success]───────▶  RECOVERING
```

### Events Emitted

| Transition | Event |
|------------|-------|
| `HEALTHY → DEGRADED` | `platform.provider.health.degraded` |
| `* → CIRCUIT_OPEN` | `platform.provider.health.degraded` |
| `RECOVERING → HEALTHY` | `platform.provider.health.restored` |

---

## 4. Execution Plan State Machine

### States

| State | Meaning |
|-------|---------|
| `CREATED` | Plan object created |
| `QUEUED` | Awaiting resource slot |
| `RUNNING` | Agents actively executing |
| `COMPLETED` | All agents finished successfully |
| `PARTIAL` | Some agents completed, some failed |
| `FAILED` | All agents failed |
| `CANCELLED` | Cancelled before completion |

### Transitions

```
CREATED    ──submit()───────▶  QUEUED
QUEUED     ──dispatch()─────▶  RUNNING
RUNNING    ──[all_ok]────────▶  COMPLETED
RUNNING    ──[some_fail]─────▶  PARTIAL
RUNNING    ──[all_fail]──────▶  FAILED
RUNNING    ──cancel()────────▶  CANCELLED
QUEUED     ──cancel()────────▶  CANCELLED
PARTIAL    ──retry()─────────▶  RUNNING
FAILED     ──retry()─────────▶  RUNNING
```

### Guard Conditions

| Transition | Guard |
|------------|-------|
| `QUEUED → RUNNING` | Quota check passes; budget check passes |
| `FAILED → retry` | Retry count < `RetryStrategy.max_retries` |

---

## 5. Knowledge Document State Machine

### States

| State | Meaning |
|-------|---------|
| `DRAFT` | Being authored |
| `REVIEW` | Under peer review |
| `AUTHORITATIVE` | Approved, in effect |
| `FROZEN` | No further changes allowed |
| `SUPERSEDED` | Replaced by newer document |

### Transitions

```
DRAFT         ──submit()────▶  REVIEW
REVIEW        ──approve()───▶  AUTHORITATIVE
REVIEW        ──reject()────▶  DRAFT
AUTHORITATIVE ──freeze()────▶  FROZEN
AUTHORITATIVE ──supersede()─▶  SUPERSEDED
FROZEN        ──supersede()─▶  SUPERSEDED
```

This state machine governs all documents in `architecture/docs/platform-consolidation/`.
