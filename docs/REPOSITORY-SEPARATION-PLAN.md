# AI Studio Platform — Repository Separation Plan

**Document ID:** REPOSITORY-SEPARATION-PLAN  
**Version:** 1.0.0  
**Date:** 2026-06-28  
**Status:** PROPOSED — Pending Architect Approval  
**Classification:** INTERNAL  
**Authority:** Chief Software Architect / CTO  
**Prerequisite:** Phase 0 architecture documents (BLUEPRINT, STANDARDS, CONTRACTS, EVENT-CATALOG, API-CATALOG, DATABASE-CATALOG, SECURITY-MODEL) — all RATIFIED  
**Implements:** AI-STUDIO-PLATFORM-REFACTORING-BLUEPRINT.md, Section 8 (Migration Strategy)

---

## Executive Summary

The AI Studio platform currently lives in four repositories with mixed ownership, duplicated code, and tight product-platform coupling. The existing `ai-software-factory` repository contains both the platform runtime and Mythic Realms-specific product workers. The `mythic-realms` repository contains a forked copy of the platform SDK. The `content-factory` repository is a separate product with no platform dependency — yet it is treated as if it were infrastructure.

This plan separates **Platform** from **Products**, giving each a clear boundary, a clean dependency graph, and an independent release cadence. It does not move a single file. It produces the approved plan that a subsequent implementation phase will execute.

**Current state:** 4 repositories, mixed ownership, coupled releases, security score 2/15  
**Target state:** 7 directories (platform, products, desktop, sdk, tools, examples, marketplace), clean dependency graph, security score 15/15 design-compliant

---

## 1. Current Repository Inventory

### 1.1 ai-software-factory

| Property | Value |
|----------|-------|
| Type | Platform + AISF Product + SDK (monolithic) |
| Language | Python 3.13 |
| Runtime | FastAPI / uvicorn |
| Port | 8088 |
| Files | ~280 Python files (excluding build/, .venv) |
| Database | SQLite (dev) / PostgreSQL (prod) |
| Event Bus | NATS JetStream (optional) |
| Config prefix | `ORCH_` (legacy — must migrate to `AISF_`) |
| Auth | Disabled by default (`api_key: str = ""`) — **security violation** |
| Tests | 33 test files, 233 passing tests |

**Top-level structure:**
```
ai-software-factory/
├── api/            # 40 route modules — ALL platform API endpoints
├── db/             # Database models (mixed platform + product)
├── engine/         # 43 modules — platform core + AISF product logic + MR workers
├── engine/prompt_os/   # 11 modules — Prompt OS platform
├── factory/        # SDK, providers, logging, metrics, tracing, storage
├── worker/         # 11 worker modules — MIXED (platform + Mythic Realms product)
├── runtime/        # Claude executor, git manager, build runner
├── installer/      # Desktop installer packaging
├── dashboard/      # Dashboard HTML builder
├── alembic/        # 12 database migrations
├── app.py          # Platform entrypoint
├── config.py       # ORCH_* settings (legacy prefix)
└── supervisor.py   # Worker supervisor
```

**Critical finding:** The `worker/` directory contains `battle_worker.py`, `economy_worker.py`, `collection_worker.py`, `liveops_worker.py`, and `player_worker.py`. These are Mythic Realms product workers embedded in the platform repository. This is the primary coupling violation.

### 1.2 ai-studio-desktop

| Property | Value |
|----------|-------|
| Type | Desktop client (correct placement) |
| Language | Python 3.13 / PySide6 |
| Communication | HTTP REST to platform API (no direct import coupling) |
| Files | ~210 Python files |
| Tests | 28 test files |

**Assessment:** CORRECTLY POSITIONED. The desktop has no Python import coupling to the platform — all communication is over HTTP via `services/api_client.py`. It reads `workspace.yaml` for service discovery. This repository's extraction is low-risk.

### 1.3 content-factory

| Property | Value |
|----------|-------|
| Type | Product (AI video production pipeline) |
| Language | Java 21 / Spring Boot 3.4 |
| Modules | 40 `cf-*` modules (Maven multi-module) |
| Files | 1,286 Java files |
| Platform dependency | None (standalone Java application) |
| Communication | Independent REST API on port 8090 |

**Assessment:** CORRECTLY POSITIONED as a standalone product. Has zero dependency on the Python platform. Extraction is organizational only — it moves from `source/` to `products/`.

### 1.4 mythic-realms

| Property | Value |
|----------|-------|
| Type | Product (AI trading card game) |
| Language | Java 21 (backend) + Python (orchestrator) |
| Backend | Spring Boot 3.x REST API |
| Orchestrator | `tools/orchestrator/` — forked copy of AISF platform |
| SDK | `ai-software-factory/` — vendored copy of factory/sdk |
| Workers | battle, economy, player, collection, liveops, auth, qa, security, architect, release |

**Critical findings:**
1. `mythic-realms/tools/orchestrator/` is a fork of the AISF platform (imports engine.*, db.*, config, supervisor)
2. `mythic-realms/ai-software-factory/` is a vendored/installed copy of the platform SDK
3. Workers in `mythic-realms/tools/orchestrator/worker/` and in `ai-software-factory/worker/` are the same Mythic Realms workers (duplicated)

---

## 2. Target Workspace Layout

```
E:/UserData/MyData/Content/AIStudio/
├── architecture/               # (existing — architecture docs, no change)
│   └── docs/
│       └── *.md               # All Phase 0 + Phase 0B documents
│
├── platform/                   # NEW: Platform runtime (extracted from AISF)
│   ├── ai-runtime/             # AI ROS: router, provider runtime, cost engine
│   ├── workflow-runtime/       # Workflow engine, DAG scheduler, dispatcher
│   ├── prompt-os/              # Prompt OS: governance, HMAC, marketplace
│   ├── decision-engine/        # Decision engine
│   ├── central-brain/          # Central Brain: experience, patterns, similarity
│   ├── security/               # Auth, RBAC, rate limiting
│   ├── knowledge/              # Knowledge graph (PostgreSQL → Kuzu)
│   ├── event-bus/              # NATS client, in-process event bus
│   ├── provider-runtime/       # Claude Code, OpenAI, Gemini, Ollama adapters
│   ├── plugin-runtime/         # Plugin sandbox, tool runtime
│   ├── metrics/                # Prometheus metrics, OpenTelemetry tracing
│   ├── observability/          # SLA monitor, alerting
│   ├── configuration/          # Settings (AISF_ prefix), feature flags
│   ├── workspace/              # Workspace manifest, product registry
│   ├── review-engine/          # Review router, merge manager, gate approval
│   ├── governance/             # Approval service, policy enforcement
│   ├── storage/                # Universal Storage Platform
│   └── api/                    # FastAPI REST layer, middlewares
│
├── products/                   # NEW: Products (extracted/moved)
│   ├── ai-software-factory/    # AISF product logic (extracted from engine/)
│   ├── content-factory/        # Content Factory (moved from source/)
│   └── mythic-realms/          # Mythic Realms (moved from source/ + cleaned)
│
├── desktop/                    # RENAMED: Desktop client (moved from source/)
│   └── ai-studio-desktop/      # PySide6 desktop app
│
├── sdk/                        # NEW: Platform SDKs
│   ├── python/                 # Python SDK (extracted from factory/sdk/)
│   ├── java/                   # Java SDK (stub for v1.1)
│   └── typescript/             # TypeScript SDK (stub for v2.0)
│
├── tools/                      # NEW: Platform tooling
│   ├── generator/              # factory/cli/generator.py → CLI tool
│   ├── architecture-linter/    # Validates module dependency rules
│   └── migration-tools/        # Database migration tooling
│
├── examples/                   # NEW: Example projects using the SDK
│   ├── python-project/
│   ├── java-project/
│   └── flutter-project/        # (from factory/templates/)
│
├── marketplace/                # NEW: Plugin marketplace
│   └── catalog/
│
└── playground/                 # NEW: Experimentation space
```

---

## 3. Migration Rationale

### 3.1 Why Separate Now?

**Problem 1: Mythic Realms workers embedded in platform**  
The `worker/` directory contains `battle_worker.py` and `economy_worker.py`. Any change to the platform risks breaking game-specific logic. Any game-specific change pollutes platform commit history.

**Problem 2: Forked platform in mythic-realms**  
`mythic-realms/tools/orchestrator/` is a copy of the platform. Two forks diverge over time. Bugs fixed in one are not fixed in the other. Features added to the platform must be manually merged.

**Problem 3: Vendored SDK**  
`mythic-realms/ai-software-factory/` is a vendored copy of `factory/sdk/`. When the SDK changes, Mythic Realms does not get the update automatically. The two copies will drift.

**Problem 4: Config prefix mismatch**  
All architecture documents mandate `AISF_` prefix. The current codebase uses `ORCH_`. This creates a gap between the specification and the implementation.

**Problem 5: Auth disabled by default**  
`config.py` has `api_key: str = ""` with a comment that empty string disables auth. This is the root cause of the 2/15 security score. Separation creates the right moment to fix this.

### 3.2 What This Plan Does NOT Do

- Does **not** move files (implementation phase does that)
- Does **not** change any Python imports (implementation phase does that)
- Does **not** change runtime behavior
- Does **not** rewrite business logic
- Does **not** implement the security architecture (separate phase)
- Does **not** implement AI ROS (separate phase)

---

## 4. Governing Principles

**GP-1: Platform does not depend on products.** Platform modules may not import from any product (AISF product, Content Factory, Mythic Realms). Platform is the foundation; products build on it.

**GP-2: Products depend on the Platform SDK.** Products import from the SDK Python package (or Java SDK). They do not import from platform internal modules. The SDK is the only stable interface.

**GP-3: Desktop depends on Platform API only.** The desktop communicates over HTTP. It has no Python import coupling to the platform. This must remain true after separation.

**GP-4: SDK has no runtime dependencies.** The SDK package (`sdk/python/`) must install with only standard library dependencies. It may not import from `engine/`, `db/`, or `api/`.

**GP-5: One SDK, no vendoring.** The Mythic Realms vendored copy of `factory/sdk/` is removed. Mythic Realms installs the SDK as a package.

**GP-6: No duplicate workers.** Workers that exist in both `ai-software-factory/worker/` and `mythic-realms/tools/orchestrator/worker/` must exist only in `mythic-realms`.

**GP-7: Config prefix is AISF_.** The `ORCH_` prefix is a migration target. All platform settings migrate to `AISF_` prefix per PLATFORM-STANDARDS.md.

---

## 5. Platform Module Classification Summary

The complete classification is in REPOSITORY-EXTRACTION-MATRIX.md. Summary by category:

| Category | Count | Current Location | Target |
|----------|-------|-----------------|--------|
| PLATFORM — core runtime | 18 modules | `engine/` | `platform/*/` |
| PLATFORM — prompt-os | 11 modules | `engine/prompt_os/` | `platform/prompt-os/` |
| PLATFORM — providers | 7 modules | `factory/providers/` | `platform/provider-runtime/` |
| PLATFORM — event bus | 3 modules | `engine/event_bus.py`, `nats_client.py`, `event_schemas.py` | `platform/event-bus/` |
| PLATFORM — observability | 5 modules | `factory/logging/`, `factory/metrics/`, `factory/tracing/` | `platform/metrics/` |
| PLATFORM — storage | 8 modules | `factory/storage/` | `platform/storage/` |
| PLATFORM — API | 40 modules | `api/` | `platform/api/` |
| PLATFORM — persistence | 4 modules | `db/` | `platform/api/` |
| SDK | 2 modules | `factory/sdk/` | `sdk/python/` |
| SDK plugins | 7 modules | `factory/plugins/` | `sdk/python/plugins/` |
| AISF PRODUCT | 9 modules | `engine/product_factory.py` etc. | `products/ai-software-factory/` |
| MR PRODUCT (misplaced) | 5 modules | `worker/battle*.py` etc. | `products/mythic-realms/workers/` |
| INFRA | 6 modules | `runtime/`, `installer/` | `platform/` + `tools/` |
| UTILITY | 3 modules | `factory/cli/`, `factory/hooks/` | `tools/generator/` |

---

## 6. Coupling Severity Summary

Full analysis in COUPLING-ANALYSIS.md. Critical items:

| ID | Coupling | Severity | Risk |
|----|----------|----------|------|
| C-001 | MR product workers in platform `worker/` | CRITICAL | Platform releases coupled to game development |
| C-002 | MR vendored SDK in `mythic-realms/ai-software-factory/` | HIGH | SDK drift, dual maintenance |
| C-003 | MR forked orchestrator in `tools/orchestrator/` | HIGH | Platform divergence, bug duplication |
| C-004 | Config uses `ORCH_` prefix (not `AISF_`) | MEDIUM | Architecture compliance |
| C-005 | Auth disabled by default (`api_key: str = ""`) | CRITICAL | Security score 2/15 |
| C-006 | All DB models in one flat file (`db/models.py`) | MEDIUM | Schema ownership violation |
| C-007 | `engine/` mixes platform + AISF product logic | HIGH | Hard to extract independently |
| C-008 | `runtime/git_manager.py` directly calls git (no abstraction) | MEDIUM | Platform-product boundary unclear |

---

## 7. Extraction Order

Extraction must follow dependency order: extract what has no internal dependencies first.

**Phase 1 — Foundation (Zero Dependencies)**  
Extract configuration, logging, metrics, tracing, versioning. These modules have no imports from other platform modules. They are the leaf nodes of the dependency graph.

**Phase 2 — Infrastructure Services**  
Extract event bus, security primitives, storage. These depend only on Phase 1 outputs and standard library.

**Phase 3 — Core Platform Services**  
Extract workflow runtime, prompt OS, provider runtime. These depend on Phase 1 + Phase 2.

**Phase 4 — Intelligence Services**  
Extract central brain, decision engine, knowledge, AI ROS. These depend on Phase 1 + Phase 2 + Phase 3.

**Phase 5 — Products and Desktop**  
Extract AISF product logic, Mythic Realms workers. Reorganize product directories. Move desktop.

**Phase 6 — SDK and Tools**  
Extract SDK to its own directory. Extract CLI to tools. Clean up vendored copies.

Full extraction roadmap with detailed steps in PLATFORM-EXTRACTION-ROADMAP.md.

---

## 8. Risk Summary

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| Import path changes break tests | HIGH | HIGH | Run full test suite after each phase; fix before proceeding |
| Mythic Realms orchestrator becomes broken during SDK extraction | MEDIUM | HIGH | SDK extraction before removing MR vendored copy |
| Configuration migration (`ORCH_` → `AISF_`) breaks existing deployments | HIGH | MEDIUM | Dual-prefix support during transition; deprecation warning on old prefix |
| Git history fragmentation | LOW | MEDIUM | Use `git subtree` to preserve history; never `cp` + delete |
| Circular imports discovered during extraction | MEDIUM | MEDIUM | Run `pyflakes` + `import-linter` before each phase |

---

## 9. Success Criteria

Phase 0B (this plan) is complete when:
- [ ] All 7 documents are written and reviewed
- [ ] Architecture compliance rules are defined (see REPOSITORY-DEPENDENCY-RULES.md)
- [ ] Every current module has a target location in REPOSITORY-EXTRACTION-MATRIX.md
- [ ] Coupling report identifies all violations with evidence (see COUPLING-ANALYSIS.md)
- [ ] Extraction phases are sequenced by dependency order (see PLATFORM-EXTRACTION-ROADMAP.md)
- [ ] Migration checklist is complete and actionable (see REPOSITORY-MIGRATION-CHECKLIST.md)
- [ ] Future repository tree is fully specified (see REPOSITORY-TREE-V2.md)
- [ ] CTO has reviewed and approved for implementation

Implementation phase begins only after CTO approval of this plan.

---

## 10. Document Map

| Document | Purpose | Key Consumer |
|----------|---------|-------------|
| **This document** | Master plan, rationale, phases | CTO, Architects |
| REPOSITORY-EXTRACTION-MATRIX.md | Row-by-row module disposition | Platform Engineer |
| REPOSITORY-DEPENDENCY-RULES.md | Formal dependency rules | All engineers |
| COUPLING-ANALYSIS.md | Evidence-based coupling report | Security, Architects |
| PLATFORM-EXTRACTION-ROADMAP.md | Phase-by-phase execution | Platform Engineer |
| REPOSITORY-MIGRATION-CHECKLIST.md | Actionable checklist | Platform Engineer |
| REPOSITORY-TREE-V2.md | Future directory trees | All engineers |

---

*End of REPOSITORY-SEPARATION-PLAN.md*  
*Version 1.0.0 | Status: PROPOSED | Authority: Chief Software Architect*
