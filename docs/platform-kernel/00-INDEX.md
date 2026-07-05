# Platform Kernel Architecture — Document Index
# Phase 2.0D.2.6A
# Status: ARCHITECTURE FREEZE CANDIDATE

---

## Document Set

| # | File | Contents |
|---|------|----------|
| 00 | `00-INDEX.md` | This index |
| 01 | `01-OVERVIEW.md` | Vision, principles, layer model, kernel position |
| 02 | `02-OBJECT-MODEL.md` | PlatformObject hierarchy, identity, metadata, ownership |
| 03 | `03-MODULE-CATALOG.md` | All 40 kernel modules fully specified |
| 04 | `04-EVENT-SYSTEM.md` | PlatformEvent, PlatformEventBus, correlation, replay |
| 05 | `05-LIFECYCLE.md` | 10-state lifecycle, state machine, transitions |
| 06 | `06-SERVICES.md` | DI, registry, plugin discovery, configuration, capability |
| 07 | `07-SCHEDULING.md` | Task scheduler, worker, queues, cancellation, retry |
| 08 | `08-RESOURCE-MODEL.md` | CPU, memory, network, AI/token budget, storage |
| 09 | `09-DIAGNOSTICS.md` | Metrics, tracing, health, logging, audit |
| 10 | `10-SECURITY.md` | Identity, auth, permissions, secrets, sandbox |
| 11 | `11-GOVERNANCE.md` | Architecture rules, extension rules, upgrade rules |
| 12 | `12-DIAGRAMS.md` | Component, dependency, lifecycle, event flow, init, exec, plugin |
| 13 | `13-VERIFICATION.md` | Dependency matrix, interface catalog, event catalog, readiness |

---

## Phase Context

```
Phase 2.0D.2.5A  Platform SDK Architecture         COMPLETE / FROZEN
Phase 2.0D.2.5B  Platform SDK Implementation        COMPLETE / FROZEN
Phase 2.0D.2.5C  Platform SDK Integration & Facade  COMPLETE / FROZEN
Phase 2.0D.2.5D  Platform SDK Governance            COMPLETE / FROZEN
Phase 2.0D.2.6A  Platform Kernel Architecture       THIS PHASE
```

---

## Kernel Position in the Stack

```
  AI Agents / Products / Runtimes
          |
  Platform Kernel   <-- THIS ARCHITECTURE
          |
  Platform SDK (frozen)
          |
  Platform Algorithms / Data Structures (frozen)
```

---

## Definition of Done

- [ ] All 40 kernel modules specified with full architectural detail
- [ ] Object model hierarchy defined
- [ ] Event system fully specified
- [ ] Lifecycle state machine frozen
- [ ] Security model reviewed
- [ ] Dependency matrix complete
- [ ] Interface catalog complete
- [ ] No implementation code written
- [ ] Architecture frozen for implementation phase
