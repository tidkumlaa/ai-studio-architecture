# Platform Kernel — Architecture Diagrams
# Phase 2.0D.2.6A

All diagrams are ASCII. Seven diagram types as specified.

---

## Diagram 1 — Component Overview

```
╔══════════════════════════════════════════════════════════════════════════════╗
║                        AI STUDIO PLATFORM KERNEL                            ║
║                                                                              ║
║  ┌──────────────────────────────────────────────────────────────────────┐    ║
║  │                    KERNEL PUBLIC API SURFACE                         │    ║
║  └──────────────────────────────────────────────────────────────────────┘    ║
║           │              │              │              │                      ║
║           ▼              ▼              ▼              ▼                      ║
║  ┌──────────────┐ ┌────────────┐ ┌──────────────┐ ┌──────────────┐          ║
║  │   OBJECT     │ │   EVENT    │ │   SERVICE    │ │   SECURITY   │          ║
║  │  FOUNDATION  │ │   SYSTEM   │ │  INFRASTR.   │ │              │          ║
║  │              │ │            │ │              │ │ PlatformSec  │          ║
║  │ PlatObject   │ │ PlatEvent  │ │ PlatRegistry │ │ PlatPermiss. │          ║
║  │ PlatIdentity │ │ PlatEvtBus │ │ PlatPlugin   │ │ PlatFeature  │          ║
║  │ PlatLifecycl │ │            │ │ PlatSvcLoc   │ │ Flag         │          ║
║  │ PlatStateMch │ │            │ │ PlatModule   │ │              │          ║
║  │ PlatVersion  │ │            │ │ PlatManifest │ │              │          ║
║  │ PlatCapab.   │ │            │ │              │ │              │          ║
║  └──────┬───────┘ └──────┬─────┘ └──────┬───────┘ └──────┬───────┘          ║
║         │                │              │                │                   ║
║         └────────────────┴──────────────┴────────────────┘                   ║
║                                    │                                          ║
║           ┌────────────────────────┼─────────────────────────┐               ║
║           ▼                        ▼                          ▼               ║
║  ┌──────────────┐        ┌──────────────────┐       ┌──────────────────┐     ║
║  │   CONFIG     │        │   SCHEDULING     │       │   DIAGNOSTICS    │     ║
║  │              │        │                  │       │                  │     ║
║  │ PlatConfig   │        │ PlatScheduler    │       │ PlatMetrics      │     ║
║  │ PlatSettings │        │ PlatTask         │       │ PlatHealth       │     ║
║  │              │        │ PlatWorker       │       │ PlatDiagnostics  │     ║
║  │              │        │                  │       │ PlatAudit        │     ║
║  └──────────────┘        └──────────────────┘       └──────────────────┘     ║
║                                                                               ║
║           ┌──────────────┐        ┌────────────────────────────────────┐     ║
║           │   LOGGING    │        │    CONTEXT & WORKSPACE             │     ║
║           │ PlatLogger   │        │ PlatContext  PlatWorkspace         │     ║
║           └──────────────┘        │ PlatProject  PlatRepository        │     ║
║                                   │ PlatSession                        │     ║
║           ┌──────────────┐        └────────────────────────────────────┘     ║
║           │ EXECUTION    │        ┌────────────────────────────────────┐     ║
║           │ PlatExecution│        │    INFRASTRUCTURE                  │     ║
║           │ PlatCommand  │        │ PlatResource  PlatHook             │     ║
║           │ PlatQuery    │        │ PlatExtension PlatException        │     ║
║           │ PlatPipeline │        │ PlatResult                         │     ║
║           └──────────────┘        └────────────────────────────────────┘     ║
║                                                                               ║
║  ══════════════════════════════════════════════════════════════════════════   ║
║                        platform_sdk (frozen)                                  ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

---

## Diagram 2 — Dependency Graph (Module-to-Module)

Arrows = "depends on" (left imports from right/below)

```
Group A  ●──────────────────────────────────── no dependencies
(Object)  PlatObject  PlatIdentity  PlatLifecycle
          PlatStateMachine  PlatVersion  PlatCapability
                │
                │ (all groups depend on A)
                ▼
Group B  ●──── A
(Events)  PlatEvent  PlatEventBus
                │
                ├────────────────────────────┐
                ▼                            ▼
Group C  ●──── A, B               Group D  ●──── A, B
(Services)                        (Config)
PlatRegistry  PlatPlugin          PlatConfiguration
PlatSvcLoc    PlatModule          PlatSettings
PlatManifest
                │                            │
                └───────────────┬────────────┘
                                ▼
Group E  ●──── A, B, C, D    Group F  ●──── A, B, C
(Scheduling)                  (Diagnostics)
PlatScheduler                 PlatMetrics  PlatHealth
PlatTask                      PlatDiagnostics  PlatAudit
PlatWorker
                                             │
                                             ▼
                              Group G  ●──── A, B, F
                              (Logging)
                              PlatLogger
                │
                │   ┌──────────────────────┐
                ▼   ▼                      │
Group H  ●──── A, B, C, D, F              │
(Security)                                │
PlatSecurity  PlatPermission              │
PlatFeatureFlag                           │
                │                         │
                ▼                         │
Group I  ●──── A, B, C, D, E, F, H       │
(Context & Workspace)                     │
PlatContext  PlatWorkspace                │
PlatProject  PlatRepository               │
PlatSession                               │
                │                         │
                ▼                         │
Group J  ●──── A, B, C, D, E, F, H, I   │
(Execution)                               │
PlatExecution  PlatCommand                │
PlatQuery  PlatPipeline                   │
                                          │
Group K  ●──── A, B  ────────────────────┘
(Infrastructure — minimal deps)
PlatResource  PlatHook
PlatExtension PlatException
PlatResult
```

---

## Diagram 3 — Lifecycle State Machine

```
  ┌─────────────────┐
  │  UNINITIALIZED  │
  └────────┬────────┘
           │ initialize()
           ▼
  ┌─────────────────┐
  │  INITIALIZED    │◄──────────────────────────────┐
  └────────┬────────┘                               │ (re-init not shown;
           │ start()                                │  stop → start is the path)
           ▼
  ┌─────────────────┐
  │    STARTING     │──────── fail() ──────────────►┐
  └────────┬────────┘                               │
           │ [ready signal]                         │
           ▼                                        │
  ┌─────────────────┐ ◄──── resume() ─────────┐    │
  │    RUNNING      │──── pause() ────────────►│    │
  └────────┬────────┘                    ┌─────┴─┐  │
           │                             │PAUSING│  │
           │ fail()                      └───┬───┘  │
           │                                │       │
           │             [drained]          │       │
           │                      ┌─────────┴──┐    │
           │                      │   PAUSED   │    │
           │                      └─────┬──────┘    │
           │                            │ stop()     │
           │ stop()            ┌────────┴──────────┐ │
           │                   │      STOPPING     │ │
           └──────────────────►│                   │ │
                               └────────┬──────────┘ │
                                        │ [done]      │
                                        ▼             │
                               ┌──────────────────┐  │
                               │    STOPPED       │  │
                               └────────┬─────────┘  │
                                        │ start()     │
                                        └────────────►┘ (back to INITIALIZED)

                               ┌──────────────────┐
    ◄──── fail() ─────────────►│     FAILED       │
                               └────────┬─────────┘
                                        │ recover()
                                        ▼
                               ┌──────────────────┐
                               │   RECOVERING     │──► RUNNING (success)
                               └──────────────────┘──► FAILED  (fail again)

    Any State (except DISPOSED)
           │ dispose()
           ▼
  ┌─────────────────┐
  │    DISPOSING    │
  └────────┬────────┘
           │ [done]
           ▼
  ┌─────────────────┐
  │    DISPOSED     │  ← terminal; no further transitions
  └─────────────────┘
```

---

## Diagram 4 — Event Flow

```
  PUBLISHER                  PLATFORM EVENT BUS                   SUBSCRIBERS
  (any kernel                                                      (any kernel
   component)                                                       component)
       │                                                                │
       │  publish(event)                                                │
       ├──────────────►┌────────────────────────────────────────────┐  │
       │               │ 1. Validate event (non-null fields)         │  │
       │               │ 2. Assign sequence number                   │  │
       │               │ 3. Write to replay log (if persistent)      │  │
       │               │ 4. Determine priority                       │  │
       │               │                                             │  │
       │               │    CRITICAL ──► dispatch INLINE             │  │
       │               │    HIGH ──────► dedicated dispatch thread   │  │
       │               │    NORMAL ────► thread pool                 │  │
       │               │    LOW ────────► thread pool (low-pri)      │  │
       │               │    BACKGROUND ► idle thread                 │  │
       │               │                                             │  │
       │               │ 5. Match event_type against all patterns    │  │
       │               │ 6. For each matching subscription:          │  │
       │               │    a. Apply subscriber filter (if any)      │  │
       │               │    b. Enqueue to subscriber's queue         │  │
       │               │    c. Signal subscriber thread              │  │
       │               └────────────────────────────────────────────┘  │
       │                                                                │
       │                                                  ┌─────────────┘
       │                                                  │ deliver(event)
       │                                                  ▼
       │                                          ┌───────────────┐
       │                                          │  SUBSCRIBER   │
       │                                          │  HANDLER      │
       │                                          │               │
       │                                          │  try:         │
       │                                          │    handler(e) │
       │                                          │  except:      │
       │                                          │    retry up   │
       │                                          │    to N times │
       │                                          │    then →     │
       │                                          │    dead letter│
       │                                          └───────────────┘
       │
       │               DEAD LETTER QUEUE
       │               ┌─────────────────────────────────────────┐
       │               │ Events that could not be delivered      │
       │               │ after all retries exhausted.            │
       │               │                                         │
       │               │ Operations: peek() requeue() discard()  │
       │               └─────────────────────────────────────────┘
```

---

## Diagram 5 — Kernel Initialization Flow

```
  CALLER (Desktop / Runtime)
       │
       │  KernelContainer(settings)
       ▼
  ┌─────────────────────────────────────────────────────────────────────┐
  │ BOOT PHASE 1 — Core Infrastructure (no event bus yet)              │
  │                                                                     │
  │   [1] PlatformAudit.initialize()       — append-only store ready   │
  │   [2] PlatformMetrics.initialize()     — metric registry ready     │
  │   [3] LoggerFactory.configure()        — logger factory ready      │
  │   [4] PlatformConfiguration.load()     — config from all sources   │
  │                                                                     │
  └─────────────────────────────────────────────────────────────────────┘
       │
       ▼
  ┌─────────────────────────────────────────────────────────────────────┐
  │ BOOT PHASE 2 — Event Infrastructure                                │
  │                                                                     │
  │   [5] PlatformEventBus.initialize()    — bus ready, no subs yet    │
  │   [6] PlatformRegistry.initialize()   — registry ready            │
  │   [7] Replay recent persistent events from PlatformAudit           │
  │                                                                     │
  └─────────────────────────────────────────────────────────────────────┘
       │
       ▼
  ┌─────────────────────────────────────────────────────────────────────┐
  │ BOOT PHASE 3 — Security & Governance                               │
  │                                                                     │
  │   [8]  PlatformSecurity.initialize()  — auth + capability system   │
  │   [9]  FeatureFlagManager.load()      — feature flags ready        │
  │   [10] QuotaRegistry.load()           — resource quotas loaded     │
  │                                                                     │
  └─────────────────────────────────────────────────────────────────────┘
       │
       ▼
  ┌─────────────────────────────────────────────────────────────────────┐
  │ BOOT PHASE 4 — Execution Infrastructure                            │
  │                                                                     │
  │   [11] PlatformScheduler.start()     — thread pools started        │
  │   [12] PlatformHealth.start()        — health check loop started   │
  │   [13] ResourceManager.initialize()  — resource accounting ready   │
  │   [14] PlatformServiceLocator.init() — locator ready               │
  │                                                                     │
  └─────────────────────────────────────────────────────────────────────┘
       │
       ▼
  ┌─────────────────────────────────────────────────────────────────────┐
  │ BOOT PHASE 5 — Plugin System                                       │
  │                                                                     │
  │   [15] PluginManager.initialize()                                  │
  │   [16] PluginManager.discover(search_paths)                        │
  │   [17] For each manifest (in dependency order):                    │
  │         a. Validate manifest                                       │
  │         b. Verify capabilities                                     │
  │         c. Load plugin                                             │
  │         d. Register in PlatformRegistry                            │
  │                                                                     │
  └─────────────────────────────────────────────────────────────────────┘
       │
       ▼
  ┌─────────────────────────────────────────────────────────────────────┐
  │ BOOT COMPLETE                                                       │
  │                                                                     │
  │   [18] kernel.boot.complete event emitted (CRITICAL, persistent)   │
  │   [19] KernelContainer.is_ready = True                             │
  │   [20] Control returned to caller                                  │
  │                                                                     │
  └─────────────────────────────────────────────────────────────────────┘
```

---

## Diagram 6 — Execution Flow

```
  SESSION                     EXECUTION                    PIPELINE
  (PlatformSession)           (PlatformExecution)          (PlatformPipeline)
       │                            │                            │
       │ create_execution(ctx)      │                            │
       ├───────────────────────────►│                            │
       │                            │                            │
       │                            │ CommandBus.dispatch(cmd)   │
       │                            ├────────────────────────────►
       │                            │                            │
       │                            │                    ┌───────┴──────┐
       │                            │                    │   STEP 1     │
       │                            │                    │  (validate)  │
       │                            │                    └───────┬──────┘
       │                            │                            │
       │                            │                    ┌───────┴──────┐
       │                            │                    │   STEP 2     │
       │                            │                    │  (authorize) │
       │                            │                    └───────┬──────┘
       │                            │                            │
       │                            │                    ┌───────┴──────┐
       │                            │                    │   STEP N     │
       │                            │                    │  (execute)   │
       │                            │                    └───────┬──────┘
       │                            │                            │
       │                            │◄───── PlatformResult ──────┘
       │                            │
       │                            │ emit kernel.execution.completed
       │                            │ record to PlatformAudit
       │                            │ update ResourceBudget consumed
       │                            │
       │◄───── PlatformResult ──────┤
       │                            │
  [session stores result]    [execution STOPPED]
```

---

## Diagram 7 — Plugin Flow

```
  PLUGIN MANAGER             MANIFEST              PLUGIN SANDBOX         KERNEL SERVICES
       │                       │                        │                      │
       │ discover(path)        │                        │                      │
       ├──────────────────────►│                        │                      │
       │                       │ validate()             │                      │
       │◄─ list[manifests] ───┤                        │                      │
       │                                                │                      │
       │ For each manifest:                             │                      │
       │                                                │                      │
       │ 1. Verify capabilities ─────────────────────────────────────────────►│
       │◄──── allowed? ─────────────────────────────────────────────────────── │
       │                                                │                      │
       │ 2. Create sandbox ────────────────────────────►│                      │
       │◄── PluginSandbox ─────────────────────────────┤                      │
       │                                                │                      │
       │ 3. import entry_module (inside sandbox)        │                      │
       │                                                │                      │
       │ 4. Instantiate entry_class ─────────────────────────────────────────►│
       │    (inject kernel services)◄───────────────────────────────────────── │
       │                                                │                      │
       │ 5. plugin.initialize()                         │                      │
       │                                                │                      │
       │ 6. Register in PlatformRegistry ──────────────────────────────────── ►│
       │                                                │                      │
       │ 7. Emit kernel.plugin.loaded ──────────────────────────────────────── ►│
       │                                                │                      │
       │ [PLUGIN NOW RUNNING]                           │                      │
       │                                                │                      │
       │ Plugin makes a kernel API call:                │                      │
       │    plugin ──► sandbox.check("kernel.event.publish") ◄── ALLOW/DENY    │
       │           ──► event_bus.publish(event)                                │
       │                                                │                      │
       │ [ON UNLOAD]                                    │                      │
       │                                                │                      │
       │ 1. plugin.stop()                               │                      │
       │ 2. PlatformRegistry.unregister(plugin_id)      │                      │
       │ 3. sandbox.exit() ────────────────────────────►│                      │
       │ 4. del sys.modules[entry_module]               │                      │
       │ 5. Emit kernel.plugin.unloaded                                        │
```
