# AI Studio 2.0 — Autonomous AI Software Engineering Platform
# System Architecture & Design Specification

**Version:** 2.0.0-DRAFT  
**Date:** 2026-06-27  
**Status:** Architecture Proposal  
**Predecessor:** AI Studio v1.5.5 (in production)

---

## Table of Contents

1. [Complete System Architecture](#1-complete-system-architecture)
2. [Component Diagram](#2-component-diagram)
3. [Sequence Diagrams](#3-sequence-diagrams)
4. [Deployment Diagram](#4-deployment-diagram)
5. [Agent Architecture](#5-agent-architecture)
6. [Database Schema](#6-database-schema)
7. [Event Model](#7-event-model)
8. [API Specifications](#8-api-specifications)
9. [Plugin SDK](#9-plugin-sdk)
10. [Memory Architecture](#10-memory-architecture)
11. [Knowledge Graph Design](#11-knowledge-graph-design)
12. [Worker Runtime](#12-worker-runtime)
13. [Desktop UI Mockups](#13-desktop-ui-mockups)
14. [Technology Stack](#14-technology-stack)
15. [Development Roadmap](#15-development-roadmap)
16. [Migration Plan from v1.5.5](#16-migration-plan-from-v155)
17. [Risk Assessment](#17-risk-assessment)
18. [Testing Strategy](#18-testing-strategy)
19. [Performance Targets](#19-performance-targets)
20. [Release Plan](#20-release-plan)

---

## 1. Complete System Architecture

### 1.1 Architecture Overview

AI Studio 2.0 is a layered, event-driven autonomous software engineering platform. The v1.5.5 runtime is the proven foundation. All new systems are additive.

```
┌─────────────────────────────────────────────────────────────────────┐
│                        USER INTERACTION LAYER                        │
│  Desktop App (PySide6)  ·  Web UI (optional)  ·  CLI  ·  REST API  │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────┐
│                        COORDINATOR LAYER                             │
│         Session Manager  ·  Auth/RBAC  ·  Human Approval Gates      │
│         Requirement Intake  ·  Project Context  ·  Audit Logger     │
└──────┬─────────────────────────┬──────────────────────────┬─────────┘
       │                         │                          │
┌──────▼──────┐         ┌────────▼────────┐       ┌────────▼─────────┐
│ Task Planner│         │  Agent          │       │  Knowledge       │
│             │         │  Orchestrator   │       │  Graph           │
│ - Decompose │         │                 │       │  (Neo4j / Kuzu)  │
│ - Prioritize│         │ - Task Queue    │       │                  │
│ - Estimate  │◄───────►│ - Worker Pool   │◄─────►│ - Code entities  │
│ - Sequence  │         │ - Dependency    │       │ - Requirements   │
│ - Assign    │         │   Graph         │       │ - Decisions      │
└─────────────┘         │ - Retry Engine  │       │ - Relationships  │
                        │ - Checkpointing │       └──────────────────┘
                        └────────┬────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────┐
│                        MESSAGE BUS (NATS JetStream)                  │
│   Topics: tasks · events · agent.* · memory.* · tools.* · ci.*     │
└──┬──────────┬────────────┬──────────┬──────────┬────────────────────┘
   │          │            │          │          │
┌──▼──┐  ┌───▼──┐  ┌──────▼──┐  ┌───▼──┐  ┌───▼─────────────┐
│     │  │      │  │         │  │      │  │                 │
│AGENT│  │AGENT │  │  AGENT  │  │AGENT │  │   AGENT POOL N  │
│  1  │  │  2   │  │    3    │  │  4   │  │                 │
│Arch.│  │Back. │  │Frontend │  │ QA   │  │ Security/DevOps │
│     │  │      │  │         │  │      │  │ Docs/Reviewer   │
└──┬──┘  └──┬───┘  └─────┬───┘  └──┬───┘  └────────┬────────┘
   │        │             │         │               │
┌──▼────────▼─────────────▼─────────▼───────────────▼────────────────┐
│                        TOOL RUNTIME (sandboxed)                      │
│  File I/O · Build Runner · Test Runner · Linter · Git Ops · Shell   │
│  Browser Tools · DB Client · HTTP Client · Search · AST Parser      │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────────┐
│                        PERSISTENCE LAYER                             │
│                                                                      │
│  ┌─────────────────┐  ┌──────────────────┐  ┌────────────────────┐ │
│  │   PostgreSQL 15  │  │  Qdrant (Vector) │  │  Neo4j / Kuzu      │ │
│  │  (operational)   │  │  (embeddings)    │  │  (knowledge graph) │ │
│  └─────────────────┘  └──────────────────┘  └────────────────────┘ │
│  ┌─────────────────┐  ┌──────────────────┐  ┌────────────────────┐ │
│  │   Redis 7        │  │  S3 / MinIO      │  │  SQLite (local)    │ │
│  │  (cache/session) │  │  (artifacts)     │  │  (desktop mode)    │ │
│  └─────────────────┘  └──────────────────┘  └────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 Layered Responsibility Model

| Layer | Responsibility | Built On |
|-------|---------------|----------|
| Desktop | Human interaction, approval gates, real-time dashboards | PySide6 (v1.5.5 extended) |
| Coordinator | Session, auth, project context, human gates | New — Python FastAPI |
| Task Planner | SDLC decomposition, sequencing, estimation | New — Python |
| Agent Orchestrator | Scheduling, dependency resolution, retry, resource | New — Python |
| Message Bus | Decoupled async communication, durability | NATS JetStream (replaces RabbitMQ for agents) |
| Agent Pool | Stateless AI workers, parallel execution | New — Python workers |
| Tool Runtime | Sandboxed tool execution, capability registry | New — Python/subprocess |
| Repository Manager | Git operations, code analysis, PR management | New — Python/libgit2 |
| CI Runner | Build, test, lint, coverage, artifact management | New — integrates with GitHub Actions/local |
| Deployment Engine | Container build, K8s deploy, rollout | New — Python |
| Persistence | Operational DB, vector store, graph, cache, artifacts | PostgreSQL/Qdrant/Neo4j/Redis |
| v1.5.5 Runtime | AI model routing, content pipeline, AISF/Desktop/CF | **Unchanged** |

### 1.3 Design Invariants

1. **Human gates are not optional.** Every destructive action (commit, deploy, delete) requires explicit approval unless the project has explicitly configured unattended mode.
2. **All agents are stateless.** State lives in PostgreSQL + vector store + knowledge graph. An agent can be killed and restarted at any checkpoint.
3. **No vendor lock-in.** Every AI provider is behind an abstraction interface. Swap providers per-task.
4. **Local-first.** The full platform runs on a developer laptop with SQLite + local Qdrant + local Ollama. Cloud adds scale, not features.
5. **Audit everything.** Every tool call, agent decision, and model invocation is logged with full input/output, cost, latency, and outcome.

---

## 2. Component Diagram

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  DESKTOP SHELL (PySide6)                                                      │
│                                                                               │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │  Project    │  │  Chat /      │  │  Agent       │  │  Approval        │  │
│  │  Explorer   │  │  Requirement │  │  Timeline    │  │  Queue           │  │
│  │  Panel      │  │  Intake      │  │  Viewer      │  │  (Human Gates)   │  │
│  └─────────────┘  └──────────────┘  └──────────────┘  └──────────────────┘  │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │  Code       │  │  Test        │  │  Memory      │  │  Observability   │  │
│  │  Diff View  │  │  Results     │  │  Browser     │  │  Dashboards      │  │
│  └─────────────┘  └──────────────┘  └──────────────┘  └──────────────────┘  │
└───────────────────────────────────────┬──────────────────────────────────────┘
                                        │ WebSocket + REST
┌───────────────────────────────────────▼──────────────────────────────────────┐
│  COORDINATOR SERVICE (coordinator.py — FastAPI)                               │
│                                                                               │
│  ┌─────────────────┐  ┌──────────────────┐  ┌──────────────────────────────┐ │
│  │  Auth & RBAC    │  │  Session Manager  │  │  Approval Gate Controller   │ │
│  │  (OIDC/LDAP/   │  │                  │  │  (blocks until approved or   │ │
│  │   API key)      │  │                  │  │   timeout with rollback)     │ │
│  └─────────────────┘  └──────────────────┘  └──────────────────────────────┘ │
│  ┌─────────────────┐  ┌──────────────────┐  ┌──────────────────────────────┐ │
│  │  Audit Logger   │  │  Project Registry │  │  Secrets Vault (client)     │ │
│  └─────────────────┘  └──────────────────┘  └──────────────────────────────┘ │
└───────────────────────────────────────┬──────────────────────────────────────┘
                                        │
       ┌────────────────────────────────┼──────────────────────────────┐
       │                                │                              │
┌──────▼──────────┐  ┌─────────────────▼───────┐  ┌──────────────────▼──────┐
│  TASK PLANNER   │  │  AGENT ORCHESTRATOR      │  │  KNOWLEDGE ENGINE       │
│                 │  │                          │  │                         │
│  - NL→Tasks     │  │  - Priority Queue        │  │  - Graph Query API      │
│  - SDLC template│  │  - Dependency resolver   │  │  - Embedding search     │
│  - Estimation   │  │  - Parallel scheduler    │  │  - Context assembly     │
│  - Sequencing   │  │  - Retry w/ backoff      │  │  - Memory writer        │
│  - Checkpoint   │  │  - Resource limits       │  │  - Relationship mapper  │
│    planning     │  │  - Agent affinity        │  │                         │
│  - Conflict     │  │  - Load balancer         │  │  ┌───────────────────┐  │
│    detection    │  │  - Cancellation          │  │  │  Neo4j / Kuzu     │  │
└─────────────────┘  └──────────┬───────────────┘  │  │  (code graph)     │  │
                                │                   │  └───────────────────┘  │
                                │                   │  ┌───────────────────┐  │
                                │                   │  │  Qdrant           │  │
                                │                   │  │  (vector memory)  │  │
                                │                   │  └───────────────────┘  │
                                │                   └─────────────────────────┘
                                │
┌───────────────────────────────▼──────────────────────────────────────────────┐
│  NATS JETSTREAM MESSAGE BUS                                                   │
│                                                                               │
│  Streams: TASKS · EVENTS · AGENT_INBOX · AGENT_OUTBOX · MEMORY · CI · TOOLS  │
└──────┬────────┬────────┬───────┬───────┬────────┬──────────┬─────────────────┘
       │        │        │       │       │        │          │
┌──────▼─┐ ┌───▼────┐ ┌─▼──────┐ ┌─────▼──┐ ┌───▼────┐ ┌───▼────┐ ┌──────────┐
│Architect│ │Backend │ │Frontend│ │Database│ │Security│ │  QA    │ │Reviewer  │
│ Agent   │ │ Agent  │ │ Agent  │ │ Agent  │ │ Agent  │ │ Agent  │ │ Agent    │
├─────────┤ ├────────┤ ├────────┤ ├────────┤ ├────────┤ ├────────┤ ├──────────┤
│Planner  │ │DevOps  │ │Docs    │ │Release │ │ UX     │ │Research│ │Learning  │
│ Agent   │ │ Agent  │ │ Agent  │ │ Agent  │ │ Agent  │ │ Agent  │ │ Agent    │
└────┬────┘ └────┬───┘ └───┬────┘ └────┬───┘ └────┬───┘ └────┬───┘ └─────┬────┘
     └───────────┴──────────┴───────────┴──────────┴──────────┴───────────┘
                                        │
┌───────────────────────────────────────▼──────────────────────────────────────┐
│  TOOL RUNTIME                                                                 │
│                                                                               │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐   │
│  │  File    │ │  Build   │ │  Test    │ │  Linter  │ │  Git / VCS       │   │
│  │  System  │ │  Runner  │ │  Runner  │ │  & Fmt   │ │  Operations      │   │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────────────┘   │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐   │
│  │  Shell   │ │  HTTP    │ │  AST /   │ │  DB      │ │  Browser /        │   │
│  │ Executor │ │  Client  │ │  Search  │ │  Client  │ │  Screenshot       │   │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────────────┘   │
└───────────────────────────────────────┬──────────────────────────────────────┘
                                        │
┌───────────────────────────────────────▼──────────────────────────────────────┐
│  AI PROVIDER GATEWAY (extends v1.5.5 model routing)                           │
│                                                                               │
│  OpenAI · Claude · Gemini · Ollama · LM Studio · vLLM · OpenRouter · Custom  │
│                                                                               │
│  Features: cost tracking · token counting · fallback chain · model affinity  │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Sequence Diagrams

### 3.1 Autonomous Feature Development (Happy Path)

```
User          Desktop        Coordinator    Task          Agent         Tool
                                           Planner       Orchestrator  Runtime
  │               │               │            │               │           │
  │ "Add payment  │               │            │               │           │
  │  with Stripe" │               │            │               │           │
  ├──────────────►│               │            │               │           │
  │               ├──[POST /task]►│            │               │           │
  │               │               ├──[analyze]►│               │           │
  │               │               │            ├─[decompose]   │           │
  │               │               │            │  returns 8    │           │
  │               │               │            │  subtasks     │           │
  │               │               │            ├──────────────►│           │
  │               │               │            │               ├─[schedule]│
  │               │               │            │               │  parallel │
  │               │               │            │               │           │
  │               │◄──[APPROVAL   │            │               │           │
  │◄──[show plan]─┤    GATE]──────┤            │               │           │
  │               │               │            │               │           │
  ├─[APPROVE]────►│               │            │               │           │
  │               ├──[approve]───►│            │               │           │
  │               │               ├──[resume]─►│               │           │
  │               │               │            │               │           │
  │               │               │            │  ArchAgent    │           │
  │               │               │            │  ├──[read codebase]──────►│
  │               │               │            │  │◄──[AST + deps]─────────┤
  │               │               │            │  ├──[generate design]     │
  │               │               │            │  └──[emit ARCH_DONE]      │
  │               │               │            │               │           │
  │               │               │            │  BackendAgent (unblocked) │
  │               │               │            │  ├──[write stripe_service.py]
  │               │               │            │  ├──[write tests]─────────►│
  │               │               │            │  ├──[run tests]────────────►
  │               │               │            │  │◄──[2/5 FAIL]────────────┤
  │               │               │            │  ├──[fix + retry]──────────►
  │               │               │            │  │◄──[5/5 PASS]────────────┤
  │               │               │            │  └──[emit BACKEND_DONE]    │
  │               │               │            │               │           │
  │               │               │            │  QAAgent, SecurityAgent (parallel)
  │               │               │            │  ├──[generate extra tests] │
  │               │               │            │  ├──[SAST scan]────────────►
  │               │               │            │  └──[emit REVIEW_DONE]     │
  │               │               │            │               │           │
  │               │◄──[COMMIT     │            │               │           │
  │◄──[show diff]─┤    GATE]──────┤            │               │           │
  ├─[APPROVE]────►│               │            │               │           │
  │               │               │            │  ├──[git commit]──────────►│
  │               │               │            │  ├──[git push]             │
  │               │               │            │  ├──[CI triggered]         │
  │               │               │            │  └──[emit COMPLETE]        │
  │               │◄─[notify]─────┤            │               │           │
  │◄──[DONE]──────┤               │            │               │           │
```

### 3.2 Autonomous Bug Fix Cycle

```
Monitor       Coordinator   Planner       BackendAgent  QAAgent       CI
   │               │            │               │           │          │
   ├─[detect       │            │               │           │          │
   │  exception]   │            │               │           │          │
   ├──────────────►│            │               │           │          │
   │               ├─[classify: │            │               │          │
   │               │  severity] │            │               │          │
   │               ├─[create    │            │               │          │
   │               │  bug task]►│            │               │          │
   │               │            ├─[plan fix]►│               │          │
   │               │            │            ├─[read logs]   │          │
   │               │            │            ├─[read stack trace]       │
   │               │            │            ├─[search bug memory]      │
   │               │            │            ├─[similar past fix found] │
   │               │            │            ├─[apply patch] │          │
   │               │            │            ├─[run tests]──►│          │
   │               │            │            │           ├─[pass]       │
   │               │            │            ├─[commit]────────────────►│
   │               │            │            │               │      ├─[CI pass]
   │               │            │            │               │      │   │
   │               │◄─[APPROVAL for prod]    │               │      │   │
   │               ├─[deploy if approved]    │               │      │   │
   │               ├─[update bug memory]     │               │      │   │
```

### 3.3 Knowledge Graph Update Flow

```
Agent         Tool Runtime   KG Engine      Vector Store    PostgreSQL
  │               │               │               │              │
  ├─[write file]─►│               │               │              │
  │◄─[written]────┤               │               │              │
  ├─[parse AST]──►│               │               │              │
  │◄─[entities]───┤               │               │              │
  ├─[UPSERT nodes]────────────────►               │              │
  ├─[embed code]──────────────────────────────────►              │
  ├─[log event]────────────────────────────────────────────────-►│
  ├─[query: "does class X exist?"]─►              │              │
  │◄─────────────────────────────────────────── [yes/refs]       │
```

---

## 4. Deployment Diagram

### 4.1 Local Developer Mode (Laptop)

```
┌──────────────────────────────────────────────────────────────┐
│  DEVELOPER LAPTOP  (Windows / macOS / Linux)                 │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  AI Studio 2.0 Desktop (PySide6)                       │  │
│  └───────────────────────────┬────────────────────────────┘  │
│                              │ localhost:8090                 │
│  ┌───────────────────────────▼────────────────────────────┐  │
│  │  Coordinator + Orchestrator + Task Planner (FastAPI)   │  │
│  │  Port: 8090                                            │  │
│  └───┬──────────────────────┬──────────────────────┬──────┘  │
│      │ :4222                │ :6333                │ :7474    │
│  ┌───▼────┐  ┌──────────────▼──────┐  ┌───────────▼──────┐  │
│  │  NATS  │  │  Qdrant (vector)    │  │  Kuzu (embedded  │  │
│  │(local) │  │  Port: 6333         │  │  KG, file-based) │  │
│  └────────┘  └─────────────────────┘  └──────────────────┘  │
│                                                              │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐  │
│  │  PostgreSQL 15  │  │  Redis 7        │  │  Ollama     │  │
│  │  Port: 5432     │  │  Port: 6379     │  │  Port: 11434│  │
│  └─────────────────┘  └─────────────────┘  └─────────────┘  │
│                                                              │
│  Agent Pool (local threads — concurrency: 4 agents)         │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐        │
│  │  Agent 1 │ │  Agent 2 │ │  Agent 3 │ │  Agent 4 │        │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘        │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  v1.5.5 Services (unchanged)                           │  │
│  │  AISF :8088  ·  Desktop :8091  ·  CF :8092            │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

### 4.2 Team / Self-Hosted Mode (Docker Compose)

```
┌──────────────────────────────────────────────────────────────────────┐
│  SERVER / VM  (Linux x64, 32 GB RAM, 8 cores)                        │
│                                                                      │
│  docker-compose.yml                                                  │
│                                                                      │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────────────┐  │
│  │  coordinator   │  │  task-planner  │  │  orchestrator          │  │
│  │  :8090         │  │  (internal)    │  │  (internal)            │  │
│  └────────────────┘  └────────────────┘  └────────────────────────┘  │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────────────┐  │
│  │  agent-pool    │  │  tool-runtime  │  │  knowledge-engine      │  │
│  │  (8 workers)   │  │  (sandboxed)   │  │  (internal)            │  │
│  └────────────────┘  └────────────────┘  └────────────────────────┘  │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────────────┐  │
│  │  nats          │  │  qdrant        │  │  neo4j                 │  │
│  │  :4222/:8222   │  │  :6333         │  │  :7474/:7687           │  │
│  └────────────────┘  └────────────────┘  └────────────────────────┘  │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────────────┐  │
│  │  postgres      │  │  redis         │  │  minio (S3-compat)     │  │
│  │  :5432         │  │  :6379         │  │  :9000                 │  │
│  └────────────────┘  └────────────────┘  └────────────────────────┘  │
│  ┌────────────────┐  ┌────────────────┐                              │
│  │  nginx (proxy) │  │  v1.5.5 stack  │                              │
│  │  :443          │  │  (unchanged)   │                              │
│  └────────────────┘  └────────────────┘                              │
└──────────────────────────────────────────────────────────────────────┘
           │ HTTPS
    Clients connect via Desktop App or Web UI
```

### 4.3 Kubernetes Production Mode

```
┌─────────────────────────────────────────────────────────────────────┐
│  KUBERNETES CLUSTER                                                  │
│                                                                     │
│  Namespace: aistudio-system                                         │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  Ingress (nginx / Traefik)  →  TLS termination              │    │
│  └──────────────────┬──────────────────────────────────────────┘    │
│                     │                                               │
│  ┌──────────────────▼──────────────────────────────────────────┐    │
│  │  coordinator  Deployment  (replicas: 2, HPA: 2-8)           │    │
│  └──────────────────┬──────────────────────────────────────────┘    │
│                     │                                               │
│  ┌──────────────────▼──────────────────────────────────────────┐    │
│  │  orchestrator  Deployment  (replicas: 1, leader-election)    │    │
│  └──────────────────┬──────────────────────────────────────────┘    │
│                     │                                               │
│  ┌──────────────────▼──────────────────────────────────────────┐    │
│  │  agent-pool  Deployment  (replicas: 4-32, HPA by queue len) │    │
│  └──────────────────┬──────────────────────────────────────────┘    │
│                     │                                               │
│  ┌──────────────────▼──────────────────────────────────────────┐    │
│  │  tool-runtime  DaemonSet  (1 per node, sandboxed gVisor)    │    │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  StatefulSets: postgres-ha · nats-cluster · qdrant-cluster          │
│  Operators:    neo4j-operator · redis-operator                      │
│  Storage:      PVC (SSD) for PG/Qdrant/Neo4j  ·  S3 for artifacts   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 5. Agent Architecture

### 5.1 Agent Registry

| Agent | Role | Primary Tools | Parallelism |
|-------|------|--------------|-------------|
| **ArchitectAgent** | System design, ADRs, module boundaries | AST analyzer, knowledge graph reader | 1 per project |
| **PlannerAgent** | SDLC decomposition, task graphs, estimates | LLM planner, template library | 1 per task set |
| **BackendAgent** | API, services, business logic | File I/O, build, lint, test runner | N per project |
| **FrontendAgent** | UI components, state, routing | File I/O, browser, screenshot | N per project |
| **DatabaseAgent** | Schema design, migrations, queries | DB client, migration runner, schema diff | 1 per DB per project |
| **SecurityAgent** | SAST, dependency audit, secret scan | pip-audit, OWASP DC, semgrep, truffleHog | 1 per CI run |
| **QAAgent** | Test generation, coverage analysis, mutation | Test runner, coverage, mutation tester | N per module |
| **ReviewerAgent** | Code review, style, correctness, complexity | AST, diff reader, style linter | 1 per PR |
| **DocumentationAgent** | Docstrings, README, API docs, changelog | File I/O, AST reader | 1 per release |
| **ReleaseAgent** | Versioning, changelog, SBOM, packaging | Git, PyInstaller, Maven, SBOM tools | 1 per release |
| **DevOpsAgent** | CI config, Docker, K8s manifests, IaC | File I/O, kubectl, helm | 1 per environment |
| **RefactoringAgent** | Dead code, complexity reduction, patterns | AST, build, test runner | 1 per sprint |
| **MonitoringAgent** | Alert rules, dashboard config, SLOs | Prometheus, Grafana API | 1 per service |
| **LearningAgent** | Outcome collection, model improvement hints | PostgreSQL, metrics store | Background singleton |
| **ResearchAgent** | Web research, library comparison, PoC | HTTP client, search, sandbox | On-demand |
| **UXAgent** | Accessibility, usability, responsive | Browser, screenshot, axe-core | 1 per UI feature |
| **ProductManagerAgent** | Requirement clarification, story breakdown | LLM, knowledge graph | 1 per epic |
| **SupportAgent** | User question answering, debug guidance | Knowledge graph, log reader | N concurrent |

### 5.2 Agent Lifecycle

```
┌─────────┐   spawn   ┌──────────┐  claim_task  ┌───────────┐
│  IDLE   ├──────────►│  READY   ├─────────────►│  RUNNING  │
└────▲────┘           └──────────┘              └─────┬─────┘
     │                                                │
     │ release                         ┌──────────────┼──────────────┐
     │                                 │              │              │
     │                           ┌─────▼───┐  ┌──────▼──┐  ┌───────▼──┐
     │                           │ SUCCESS │  │ BLOCKED  │  │ FAILED   │
     │                           └─────┬───┘  └──────┬──┘  └───────┬──┘
     │                                 │              │             │
     │◄────────────────────────────────┘    emit      │   retry?    │
     │                                    BLOCKED     │             │
     │                                    event       ▼             │
     │                                           CHECKPOINT        │
     │                                           SAVED, wait       │
     │◄──────────────────────────────────────────────┘             │
     │                                                              │
     └──────────────────────────────────────────────────────────────┘
                              on max_retries exhausted →
                              escalate to human or abandon
```

### 5.3 Agent Base Class Contract

```python
class BaseAgent(ABC):
    agent_type: str
    capabilities: list[str]
    max_tokens_per_call: int
    preferred_models: list[str]       # ordered preference, falls back
    tool_access: list[ToolPermission]

    async def execute(self, task: Task, context: AgentContext) -> AgentResult:
        """Primary execution method. Must be idempotent."""
        ...

    async def checkpoint(self) -> CheckpointState:
        """Snapshot current progress for resume after failure."""
        ...

    async def resume(self, state: CheckpointState) -> AgentResult:
        """Resume from a saved checkpoint."""
        ...

    def estimate_cost(self, task: Task) -> CostEstimate:
        """Before execution: estimate token usage + wall-clock time."""
        ...

    def validate_task(self, task: Task) -> ValidationResult:
        """Check this agent can handle the task before claiming it."""
        ...
```

### 5.4 Agent Context Assembly

Every agent receives a `AgentContext` assembled from:

1. **Project snapshot** — files, structure, git log (last 20 commits)
2. **Task description** — what to do, acceptance criteria
3. **Dependency results** — outputs from predecessor tasks
4. **Relevant memories** — top-K from vector search on task description
5. **Knowledge graph slice** — nodes/edges relevant to affected modules
6. **Coding style guide** — extracted from existing code + STYLE_MEMORY
7. **Tool permissions** — what the agent is allowed to call

Context is assembled by the Knowledge Engine and streamed to the agent to stay within model context limits.

---

## 6. Database Schema

### 6.1 PostgreSQL — Operational Database

```sql
-- Projects
CREATE TABLE projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT UNIQUE NOT NULL,
    repo_url        TEXT,
    repo_path       TEXT,
    default_branch  TEXT DEFAULT 'main',
    ai_provider     TEXT DEFAULT 'claude',
    ai_model        TEXT,
    unattended_mode BOOLEAN DEFAULT FALSE,
    settings        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Tasks (the core work unit)
CREATE TABLE tasks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID REFERENCES projects(id),
    parent_id       UUID REFERENCES tasks(id),   -- subtask tree
    type            TEXT NOT NULL,               -- 'feature','bug','refactor','test','deploy'
    title           TEXT NOT NULL,
    description     TEXT,
    status          TEXT NOT NULL DEFAULT 'pending',
    -- pending|planning|ready|running|blocked|awaiting_approval|completed|failed|cancelled
    priority        INTEGER DEFAULT 50,          -- 0-100
    assigned_agent  TEXT,
    estimated_tokens INTEGER,
    estimated_seconds INTEGER,
    actual_tokens   INTEGER,
    actual_seconds  INTEGER,
    cost_usd        NUMERIC(10,6),
    attempt_count   INTEGER DEFAULT 0,
    max_attempts    INTEGER DEFAULT 3,
    checkpoint_data JSONB,
    result_data     JSONB,
    metadata        JSONB DEFAULT '{}',
    depends_on      UUID[],                      -- dependency list
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ
);
CREATE INDEX idx_tasks_project_status ON tasks(project_id, status);
CREATE INDEX idx_tasks_depends_on ON tasks USING GIN(depends_on);

-- Agent executions (immutable log)
CREATE TABLE agent_executions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id         UUID REFERENCES tasks(id),
    agent_type      TEXT NOT NULL,
    agent_instance  TEXT,
    model           TEXT NOT NULL,
    provider        TEXT NOT NULL,
    status          TEXT NOT NULL,
    input_tokens    INTEGER,
    output_tokens   INTEGER,
    cost_usd        NUMERIC(10,6),
    wall_seconds    NUMERIC(10,3),
    prompt_summary  TEXT,                        -- first 500 chars
    result_summary  TEXT,                        -- first 500 chars
    tool_calls      JSONB DEFAULT '[]',
    error           TEXT,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Tool calls (immutable audit)
CREATE TABLE tool_calls (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    execution_id    UUID REFERENCES agent_executions(id),
    tool_name       TEXT NOT NULL,
    tool_version    TEXT,
    input           JSONB NOT NULL,
    output          JSONB,
    exit_code       INTEGER,
    duration_ms     INTEGER,
    success         BOOLEAN,
    error           TEXT,
    called_at       TIMESTAMPTZ DEFAULT NOW()
);

-- Human approval gates
CREATE TABLE approval_gates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id         UUID REFERENCES tasks(id),
    gate_type       TEXT NOT NULL,  -- 'commit','deploy','delete','pr','cost_limit'
    title           TEXT NOT NULL,
    description     TEXT,
    diff            TEXT,           -- code diff or action summary
    status          TEXT DEFAULT 'pending',  -- pending|approved|rejected|expired
    approver_id     UUID REFERENCES users(id),
    expires_at      TIMESTAMPTZ,
    decided_at      TIMESTAMPTZ,
    metadata        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Memory records (text + metadata; vector stored separately in Qdrant)
CREATE TABLE memories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID REFERENCES projects(id),
    memory_type     TEXT NOT NULL,
    -- project|architecture|decision|requirement|bug|fix|api|db|style|test|release|meeting|prompt|conversation
    title           TEXT NOT NULL,
    content         TEXT NOT NULL,
    source          TEXT,           -- which agent/user created it
    source_task_id  UUID REFERENCES tasks(id),
    tags            TEXT[],
    qdrant_id       UUID,           -- corresponding vector in Qdrant
    importance      INTEGER DEFAULT 50,  -- 0-100 for retrieval ranking
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_memories_type ON memories(project_id, memory_type);
CREATE INDEX idx_memories_tags ON memories USING GIN(tags);

-- Users & RBAC
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           TEXT UNIQUE NOT NULL,
    display_name    TEXT,
    provider        TEXT DEFAULT 'local',  -- local|oidc|ldap
    provider_id     TEXT,
    role            TEXT DEFAULT 'developer',  -- admin|lead|developer|viewer
    api_key_hash    TEXT,
    last_login      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Plugins
CREATE TABLE plugins (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    version         TEXT NOT NULL,
    type            TEXT NOT NULL,  -- agent|tool|model|workflow|template
    manifest        JSONB NOT NULL,
    enabled         BOOLEAN DEFAULT TRUE,
    installed_at    TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(name, version)
);

-- Events (append-only event log)
CREATE TABLE events (
    id              BIGSERIAL PRIMARY KEY,
    project_id      UUID REFERENCES projects(id),
    task_id         UUID REFERENCES tasks(id),
    event_type      TEXT NOT NULL,
    payload         JSONB NOT NULL,
    source          TEXT,
    occurred_at     TIMESTAMPTZ DEFAULT NOW()
) PARTITION BY RANGE (occurred_at);
-- Monthly partitions: events_2026_06, events_2026_07 ...
```

### 6.2 Qdrant Collections (Vector Store)

```
Collection: project_memories
  payload fields: project_id, memory_type, memory_id, importance, created_at
  vector: 1536-dim (text-embedding-3-small) or 768-dim (local nomic-embed)
  index: HNSW, ef=128, m=16

Collection: code_chunks
  payload fields: project_id, file_path, language, chunk_type, start_line, end_line
  vector: 1536-dim
  index: HNSW

Collection: agent_prompts
  payload fields: agent_type, task_type, outcome (success|fail), model, tokens, cost
  vector: 768-dim (prompts only)
  index: HNSW
  purpose: find similar past prompts for few-shot learning
```

### 6.3 Neo4j / Kuzu Graph Schema

```cypher
// Node types
(:Project {id, name, slug})
(:Module {id, project_id, name, path, language})
(:File {id, module_id, path, hash, last_modified})
(:Class {id, file_id, name, line_start, line_end})
(:Method {id, class_id, name, signature, complexity})
(:Table {id, project_id, name, schema})
(:Column {id, table_id, name, type, nullable})
(:ApiEndpoint {id, project_id, path, method, handler})
(:Requirement {id, project_id, title, type})  // epic|story|task|bug
(:Task {id, project_id, title, status})
(:Agent {id, type, instance_id})
(:Model {id, provider, name, version})
(:Decision {id, project_id, title, rationale, date})

// Relationship types
(:File)-[:BELONGS_TO]->(:Module)
(:Class)-[:DEFINED_IN]->(:File)
(:Method)-[:DEFINED_IN]->(:Class)
(:Method)-[:CALLS]->(:Method)
(:Method)-[:READS]->(:Column)
(:Method)-[:WRITES]->(:Column)
(:ApiEndpoint)-[:HANDLED_BY]->(:Method)
(:Class)-[:DEPENDS_ON]->(:Class)
(:Module)-[:IMPORTS]->(:Module)
(:Task)-[:IMPLEMENTS]->(:Requirement)
(:Task)-[:MODIFIES]->(:File)
(:Task)-[:EXECUTED_BY]->(:Agent)
(:Agent)-[:USED_MODEL]->(:Model)
(:Requirement)-[:DEPENDS_ON]->(:Requirement)
(:Decision)-[:APPLIES_TO]->(:Module)
```

---

## 7. Event Model

### 7.1 NATS JetStream Streams

| Stream | Subjects | Retention | Purpose |
|--------|---------|-----------|---------|
| `TASKS` | `tasks.created`, `tasks.updated`, `tasks.completed`, `tasks.failed` | 7d | Task lifecycle |
| `AGENT` | `agent.{type}.inbox`, `agent.{type}.outbox`, `agent.heartbeat` | 1d | Agent messaging |
| `MEMORY` | `memory.write`, `memory.invalidate` | 3d | Memory updates |
| `TOOLS` | `tools.call`, `tools.result`, `tools.error` | 1d | Tool audit |
| `CI` | `ci.run.start`, `ci.run.complete`, `ci.test.result` | 7d | CI events |
| `APPROVAL` | `approval.requested`, `approval.granted`, `approval.rejected` | 30d | Gate audit |
| `EVENTS` | `events.*` | 90d | Full audit log |

### 7.2 Core Event Schemas

```json
// TaskCreated
{
  "event": "tasks.created",
  "task_id": "uuid",
  "project_id": "uuid",
  "type": "feature",
  "title": "Add Stripe payment",
  "priority": 80,
  "depends_on": [],
  "timestamp": "2026-06-27T14:00:00Z"
}

// AgentResultPublished
{
  "event": "agent.backend.outbox",
  "task_id": "uuid",
  "agent_type": "BackendAgent",
  "status": "success",
  "artifacts": [
    {"type": "file", "path": "services/payment.py"},
    {"type": "test", "path": "tests/test_payment.py"}
  ],
  "test_result": {"passed": 12, "failed": 0, "coverage": 0.87},
  "tokens_used": 8420,
  "cost_usd": 0.0252,
  "timestamp": "2026-06-27T14:03:22Z"
}

// ApprovalRequested
{
  "event": "approval.requested",
  "gate_id": "uuid",
  "task_id": "uuid",
  "gate_type": "commit",
  "title": "Commit: Add Stripe payment service",
  "diff_summary": "+312 lines, -4 lines across 3 files",
  "test_summary": "12/12 pass, 87% coverage",
  "expires_in_seconds": 3600,
  "timestamp": "2026-06-27T14:03:25Z"
}
```

---

## 8. API Specifications

### 8.1 Coordinator REST API (v2)

Base URL: `https://{host}/api/v2`

```
POST   /projects                     Create project
GET    /projects/{id}                Get project
GET    /projects/{id}/tasks          List tasks
POST   /projects/{id}/tasks          Submit requirement → auto-plan
GET    /tasks/{id}                   Get task + subtree
PATCH  /tasks/{id}                   Update task (cancel, pause)
GET    /tasks/{id}/executions        Execution history
GET    /tasks/{id}/artifacts         Generated files

POST   /approvals/{gate_id}/approve  Approve a gate
POST   /approvals/{gate_id}/reject   Reject a gate
GET    /approvals                    List pending approvals

GET    /memories                     Search memories (q=, type=, limit=)
POST   /memories                     Create memory
DELETE /memories/{id}                Delete memory

GET    /agents                       List active agents + status
GET    /agents/{id}/timeline         Agent execution timeline

GET    /plugins                      List installed plugins
POST   /plugins/install              Install from marketplace
DELETE /plugins/{name}               Uninstall

GET    /models                       Available AI models + status
POST   /models/test                  Test a provider/model

WS     /ws/projects/{id}/stream      Real-time event stream (task updates, agent output)
WS     /ws/approvals/stream          Real-time approval notifications
```

### 8.2 Internal Tool Runtime API

```
POST   /tools/call                   Execute a tool call (internal only)
GET    /tools                        List available tools + permissions
POST   /tools/sandbox/create         Create sandboxed execution environment
DELETE /tools/sandbox/{id}           Destroy sandbox
GET    /tools/sandbox/{id}/logs      Stream sandbox output
```

### 8.3 Plugin API Endpoints (injected per plugin)

Each plugin declares its routes in `plugin.yaml`. The coordinator mounts them at `/plugins/{plugin-name}/...`.

---

## 9. Plugin SDK

### 9.1 Plugin Types

| Type | What it provides | Example |
|------|-----------------|---------|
| `agent` | New agent with custom behavior | `SalesforceAgent`, `WordPressAgent` |
| `tool` | New tool the agent pool can call | `JiraTool`, `FigmaTool`, `SonarTool` |
| `model` | New AI provider adapter | `MistralProvider`, `CohereProvider` |
| `workflow` | Reusable task graph template | `django-app`, `react-fullstack` |
| `template` | Code/file templates | `FastAPI CRUD`, `React component` |
| `extension` | UI panel in Desktop | `PR dashboard`, `cost tracker` |

### 9.2 Plugin Manifest (`plugin.yaml`)

```yaml
name: sonar-security-agent
version: 1.0.0
type: agent
description: Runs SonarQube analysis and creates fix tasks
author: AI Studio Marketplace
license: MIT

agent:
  class: SonarSecurityAgent
  module: sonar_plugin.agent
  capabilities:
    - security_scan
    - code_quality
  tool_access:
    - shell_executor
    - http_client
    - file_reader
  preferred_models:
    - claude-sonnet-4-6
    - gpt-4o

tools_provided:
  - name: sonar_scan
    description: Run SonarQube scan on a project
    input_schema:
      type: object
      properties:
        project_key: {type: string}
        sonar_url: {type: string}

dependencies:
  python:
    - requests>=2.31
    - python-sonarqube-api>=1.3
```

### 9.3 Tool Plugin Contract

```python
class BaseTool(ABC):
    name: str
    description: str
    input_schema: dict      # JSON Schema
    requires_sandbox: bool
    timeout_seconds: int

    async def call(self, input: dict, context: ToolContext) -> ToolResult:
        ...

    def validate_input(self, input: dict) -> bool:
        ...

@dataclass
class ToolResult:
    success: bool
    output: Any
    exit_code: int | None
    error: str | None
    duration_ms: int
    metadata: dict = field(default_factory=dict)
```

### 9.4 SDK Installation

```bash
pip install aistudio-sdk

# Scaffold a new plugin
aistudio plugin new my-agent --type agent
aistudio plugin test my-agent/
aistudio plugin publish my-agent/   # to marketplace
```

---

## 10. Memory Architecture

### 10.1 Memory Hierarchy

```
┌──────────────────────────────────────────────────────────────────┐
│  TIER 1 — Context Window (ephemeral, per task)                   │
│  ~32K tokens, assembled by Knowledge Engine before each call     │
│  Discarded after task completes                                  │
└──────────────────────────────┬───────────────────────────────────┘
                               │ written back
┌──────────────────────────────▼───────────────────────────────────┐
│  TIER 2 — Session Memory (Redis, TTL: 24h)                       │
│  Active task state, in-progress decisions, recent tool outputs   │
│  Warm cache for fast retrieval during task execution             │
└──────────────────────────────┬───────────────────────────────────┘
                               │ persisted on significant events
┌──────────────────────────────▼───────────────────────────────────┐
│  TIER 3 — Project Memory (PostgreSQL + Qdrant)                   │
│  Persistent across sessions. 14 memory types.                    │
│  Searchable via text (PG full-text) + semantic (Qdrant vector)   │
└──────────────────────────────┬───────────────────────────────────┘
                               │ abstracted relationships
┌──────────────────────────────▼───────────────────────────────────┐
│  TIER 4 — Knowledge Graph (Neo4j / Kuzu)                         │
│  Structural knowledge: code entities, dependencies, ownership    │
│  Traversal queries: "all callers of this method"                 │
└──────────────────────────────────────────────────────────────────┘
```

### 10.2 Memory Types and Lifecycle

| Memory Type | Created By | Expires | Search Key |
|-------------|-----------|---------|-----------|
| `project` | User / PlannerAgent | Never | project_id |
| `architecture` | ArchitectAgent | On major refactor | module_name |
| `decision` | ArchitectAgent / User | Never (historical) | affected_modules |
| `requirement` | User / ProductManagerAgent | On completion | epic_id |
| `bug` | MonitoringAgent / User | 90 days post-fix | error_hash |
| `fix` | BackendAgent | 180 days | bug_id, file_path |
| `api` | BackendAgent / DocumentationAgent | On API change | endpoint_path |
| `db` | DatabaseAgent | On schema change | table_name |
| `style` | ReviewerAgent | On style guide update | project_id |
| `test` | QAAgent | On test deletion | test_id, file_path |
| `release` | ReleaseAgent | Never | version |
| `meeting` | User (manual) | Never | date, topic |
| `prompt` | LearningAgent | 30 days | agent_type, task_type |
| `conversation` | User / Coordinator | 7 days | session_id |

### 10.3 Memory Retrieval Pipeline

```
Query: task description
   │
   ├── 1. Exact match (PostgreSQL FTS): tag, title, source_task_id
   ├── 2. Semantic search (Qdrant): top-20 by cosine similarity
   ├── 3. Graph traversal (Neo4j): neighbors of affected nodes
   └── 4. Recency boost: importance score × recency factor
          │
          ▼
   Re-rank: cross-encoder (local model)
          │
          ▼
   Top-K memories (K = remaining context budget / avg_memory_tokens)
          │
          ▼
   Injected into agent context
```

### 10.4 Memory Compression

When project memory exceeds thresholds:
- Summaries replace individual conversation memories older than 30 days
- Bug+Fix pairs are merged into "fix recipe" memories
- Low-importance memories (importance < 20) are expired after 60 days
- LearningAgent periodically runs consolidation: `POST /internal/memory/consolidate`

---

## 11. Knowledge Graph Design

### 11.1 Repository Ingestion Pipeline

```
git clone / pull
      │
      ▼
Language Detection (by extension)
      │
      ├── Python → tree-sitter-python
      ├── Java → tree-sitter-java
      ├── TypeScript → tree-sitter-typescript
      ├── SQL → pganalyze/pg-query
      └── ...
      │
      ▼
AST Extraction → Class, Method, Import, Decorator nodes
      │
      ▼
Dependency Resolution → IMPORTS/CALLS/READS/WRITES edges
      │
      ▼
Complexity Metrics (McCabe CC, LOC, coupling)
      │
      ▼
Neo4j MERGE (idempotent upsert)
      │
      ▼
Vector Embedding → Qdrant (code_chunks collection)
      │
      ▼
Change Detection → evict stale nodes when files change
```

### 11.2 Graph Query Examples

```cypher
// Find all methods with complexity > 10 in a module
MATCH (m:Method)-[:DEFINED_IN]->(c:Class)-[:DEFINED_IN]->(f:File)-[:BELONGS_TO]->(mod:Module)
WHERE mod.name = 'payment' AND m.complexity > 10
RETURN f.path, c.name, m.name, m.complexity ORDER BY m.complexity DESC

// Find dead code (methods never called)
MATCH (m:Method)
WHERE NOT (m)<-[:CALLS]-()
AND m.name <> '__init__' AND m.name <> 'main'
RETURN m

// Impact analysis: what calls this method?
MATCH path = (caller:Method)-[:CALLS*1..5]->(m:Method {name: 'process_payment'})
RETURN path

// Which requirements touch the most files?
MATCH (r:Requirement)<-[:IMPLEMENTS]-(t:Task)-[:MODIFIES]->(f:File)
RETURN r.title, COUNT(DISTINCT f) AS file_count ORDER BY file_count DESC
```

### 11.3 Automatic Analysis Reports

ArchitectAgent runs these weekly and stores results in `architecture` memory:

- **Hotspot Report**: files changed most frequently in last 90 days × complexity
- **Coupling Report**: modules with highest afferent/efferent coupling
- **Dead Code Report**: unreachable methods/classes
- **Security Hotspot Report**: methods that handle auth, input parsing, DB writes
- **Debt Map**: estimated hours to bring each module to clean architecture

---

## 12. Worker Runtime

### 12.1 Tool Permission Matrix

| Tool | Agent Types Allowed | Sandbox Required | Timeout |
|------|--------------------|--------------------|---------|
| `file_reader` | All | No | 5s |
| `file_writer` | Backend, Frontend, DB, DevOps, Release, Refactoring | No | 10s |
| `shell_executor` | Build, Test, Lint operations | **Yes (gVisor)** | 300s |
| `git_ops` | Backend, Release, DevOps | No | 60s |
| `build_runner` | Backend, Frontend, DevOps, QA | **Yes** | 600s |
| `test_runner` | QA, Backend, Frontend | **Yes** | 300s |
| `http_client` | Research, Support, Security | No | 30s |
| `db_client` | Database, Backend | No (uses existing conn) | 30s |
| `browser` | UX, Research, Frontend | **Yes (headless)** | 120s |
| `ast_parser` | Architect, Reviewer, Refactoring | No | 30s |
| `search` | All | No | 10s |

### 12.2 Sandbox Implementation

```
Tool call arrives
      │
      ▼
PermissionCheck(agent_type, tool_name) → allow / deny
      │
      ▼
SandboxAllocator.get_or_create(task_id)
  - Linux: namespace isolation (pid, net, mount, user)
  - Windows: Job Object with restricted token + process tree limits
  - RAM limit: 2 GB per sandbox
  - CPU limit: 2 cores per sandbox
  - Network: blocked by default; allow-listed domains only
  - Filesystem: copy-on-write overlay; writes isolated to temp dir
      │
      ▼
Execute command inside sandbox
      │
      ▼
Collect stdout/stderr/exit_code + resource usage
      │
      ▼
Commit file writes if exit_code == 0, else discard
      │
      ▼
Return ToolResult; log to tool_calls table
```

### 12.3 Retry and Backoff

```python
RetryPolicy(
    max_attempts=3,
    initial_delay_s=5,
    backoff_multiplier=2.0,      # 5s → 10s → 20s
    retry_on=[BuildFailed, TestFailed, ToolTimeout],
    no_retry_on=[AuthError, QuotaExceeded, HumanRejected],
    checkpoint_before_retry=True,
    notify_human_on_exhaustion=True
)
```

### 12.4 Resource Scheduling

```
AgentSlot = {cpu: 0.5, ram_gb: 2, gpu_vram_gb: 0 (optional)}

NodeCapacity = total_cpus × 0.8 reserved for agents

Scheduler:
  - If task.priority > 80 → preempt lowest-priority running task
  - If agent_type in {'Security', 'QA'} → preferred_node = high-ram
  - If model requires GPU → schedule on GPU node or queue
  - Max concurrent agents per project: configurable (default 4)
  - Global max agents: configurable (default 16 per host)
```

---

## 13. Desktop UI Mockups

### 13.1 Main Application Layout

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  AI Studio 2.0  [Project: my-saas-app ▼]  [Status: 3 agents running]  [≡]  │
├─────────┬────────────────────────────────────────────────────┬───────────────┤
│         │                                                    │               │
│ SIDEBAR │  MAIN WORKSPACE                                    │  AGENT PANEL  │
│         │                                                    │               │
│ ◉ Chat  │  ┌─ REQUIREMENT INTAKE ──────────────────────┐    │ ┌───────────┐ │
│   & Ask │  │                                           │    │ │ArchAgent  │ │
│         │  │  Describe what you want to build:         │    │ │● Running  │ │
│ ○ Tasks │  │  ┌─────────────────────────────────────┐  │    │ │Task: ARD  │ │
│   & Plan│  │  │ Add Stripe payment with webhook      │  │    │ │60% ████▒ │ │
│         │  │  │ support and subscription management  │  │    │ └───────────┘ │
│ ○ Agent │  │  │ ▊                                    │  │    │ ┌───────────┐ │
│   Timeline    │  └─────────────────────────────────────┘  │    │ │BackAgent  │ │
│         │  │  [▶ Plan this]  [⚙ Settings]  [📎 Attach]  │    │ │● Running  │ │
│ ○ Code  │  └───────────────────────────────────────────┘    │ │Task: API  │ │
│   Diff  │                                                    │ │35% ██▒▒▒ │ │
│         │  ┌─ TASK PLAN ──────────────────────────────┐    │ └───────────┘ │
│ ○ Memory│  │                                           │    │ ┌───────────┐ │
│   Browser    │  ✓ 1. Architecture Review        [done]  │    │ │QA Agent   │ │
│         │  │  ● 2. Database Schema Design      [60%]   │    │ │○ Queued   │ │
│ ○ Appro-│  │  ◉ 3. Payment Service Backend     [35%]   │    │ │Waiting on │ │
│   vals  │  │  ○ 4. Stripe Webhook Handler      [wait]  │    │ │BackAgent  │ │
│         │  │  ○ 5. Frontend Payment UI         [wait]  │    │ └───────────┘ │
│ ○ Dash- │  │  ○ 6. Integration Tests           [wait]  │    │               │
│   boards│  │  ○ 7. Documentation               [wait]  │    │ ┌───────────┐ │
│         │  │  ○ 8. Security Audit              [wait]  │    │ │Cost Today │ │
│ ○ Market│  │                          [Cancel All]     │    │ │$0.42 USD  │ │
│   place │  └───────────────────────────────────────────┘    │ │Tokens: 82K│ │
│         │                                                    │ └───────────┘ │
└─────────┴────────────────────────────────────────────────────┴───────────────┘
```

### 13.2 Approval Gate Dialog

```
┌─────────────────────────────────────────────────────────────┐
│  ⚠ APPROVAL REQUIRED                               [×]      │
├─────────────────────────────────────────────────────────────┤
│  Action:  Commit to main branch                             │
│  Agent:   BackendAgent (task: Payment Service Backend)      │
│  Expires: in 58 minutes                                     │
├─────────────────────────────────────────────────────────────┤
│  SUMMARY                                                    │
│  + 312 lines added  − 4 lines removed  across 3 files       │
│  Tests: 12/12 PASS  ·  Coverage: 87%  ·  Lint: 0 errors     │
├─────────────────────────────────────────────────────────────┤
│  DIFF                                                       │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ + services/payment.py                    (+218 ln)  │    │
│  │ + tests/test_payment.py                  (+94 ln)   │    │
│  │ ~ requirements.txt                       (+2/-4 ln)  │    │
│  │                                                     │    │
│  │ [View full diff ▼]                                  │    │
│  └─────────────────────────────────────────────────────┘    │
├─────────────────────────────────────────────────────────────┤
│  NOTES (optional):                                          │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ ▊                                                   │    │
│  └─────────────────────────────────────────────────────┘    │
├─────────────────────────────────────────────────────────────┤
│         [✗ Reject]    [→ Request Changes]    [✓ Approve]    │
└─────────────────────────────────────────────────────────────┘
```

### 13.3 Agent Timeline View

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  AGENT TIMELINE — Task: Add Stripe Payment                         [⟳]  [↓] │
├─────────────────────────────────────────────────────────────────────────────┤
│  14:00    14:05    14:10    14:15    14:20    14:25    14:30    14:35        │
│  │        │        │        │        │        │        │        │           │
│  ArchitectAgent    ████████████░░░░░░  done                                 │
│  PlannerAgent      ███░░  done                                              │
│  DatabaseAgent              ████████████  done                              │
│  BackendAgent               ░░░░░████████████████████▓▓▓▓ running (60%)    │
│  SecurityAgent                              ░░░░░░░░░░ queued               │
│  QAAgent                                   ░░░░░░░░░░░░░░ queued           │
│  FrontendAgent                                      ░░░░░░░░░░░░ queued    │
│  DocumentationAgent                                         ░░░░░ queued   │
│  ⚠ APPROVAL GATE                                    14:22 ─────┤           │
│                                                                 (pending)  │
├─────────────────────────────────────────────────────────────────────────────┤
│  ■ Running  □ Queued  ░ Waiting on dependency  ▓ Awaiting approval  done   │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 13.4 Memory Browser

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  MEMORY BROWSER                                [Search: payment        🔍]  │
├─────────────┬───────────────────────────────────────────────────────────────┤
│  Type       │  Results (24)                                                 │
│             │                                                               │
│ ■ All       │  ┌──────────────────────────────────────────────────────┐     │
│ ○ Architecture   │  [architecture] Payment Module Design  •  2026-06-27   │     │
│ ○ Decision  │  │  "Stripe is the primary provider. Fallback to PayPal │     │
│ ○ Bug       │  │   for EU customers. Webhooks verified via HMAC-SHA256"│     │
│ ○ Fix       │  │  Source: ArchitectAgent  ·  Importance: 90           │     │
│ ○ API       │  └──────────────────────────────────────────────────────┘     │
│ ○ Test      │                                                               │
│ ○ Release   │  ┌──────────────────────────────────────────────────────┐     │
│ ○ Meeting   │  [fix] Payment webhook signature validation fix • 2026-06-15  │
│             │  │  "CVE in hmac.compare_digest — use secrets.compare    │     │
│             │  │   instead. Fixed in commit a1b2c3d"                  │     │
│             │  │  Source: SecurityAgent  ·  Importance: 85            │     │
│             │  └──────────────────────────────────────────────────────┘     │
│             │                                                               │
│             │  [+ Add Memory]                         [Export] [Purge Old]  │
└─────────────┴───────────────────────────────────────────────────────────────┘
```

---

## 14. Technology Stack

### 14.1 Core Stack

| Layer | Technology | Version | Rationale |
|-------|-----------|---------|-----------|
| **Coordinator** | Python + FastAPI | 3.12 / 0.115 | Async, type-safe, consistent with v1.5.5 |
| **Agent Runtime** | Python (asyncio) | 3.12 | Async parallel workers |
| **Message Bus** | NATS JetStream | 2.10 | Lightweight, embedded option, persistent streams, < 20 MB binary |
| **Task Scheduler** | Custom (Python) | — | Full control over agent affinity, dependency graph |
| **Knowledge Graph** | Kuzu (local) / Neo4j (server) | 0.7 / 5.x | Kuzu: zero-server embedded; Neo4j: production cluster |
| **Vector Store** | Qdrant | 1.10 | Local mode (no server), Rust performance, filter+vector |
| **Relational DB** | PostgreSQL 15 | 15.x | Existing v1.5.5 DB — same connection pool |
| **Cache** | Redis 7 | 7.2 | Session state, agent locks, rate limiting |
| **Artifact Store** | MinIO (S3-compat) | latest | Local-first S3; swap for real S3 in cloud mode |
| **Embedding** | nomic-embed-text (Ollama) / text-embedding-3-small (OpenAI) | — | Local-first fallback available |
| **AST Parsing** | tree-sitter | 0.23 | Language-agnostic, Python bindings, fast |
| **Sandbox** | gVisor (Linux) / Job Objects (Windows) | — | Existing platform compatibility |
| **Desktop** | PySide6 | 6.11 | Existing v1.5.5 — extend, don't replace |
| **AI Providers** | claude-sonnet-4-6 (default), gpt-4o, gemini-2.0, ollama | — | Provider-agnostic gateway |
| **Plugin Packaging** | Python wheel + plugin.yaml | — | pip-installable plugins |
| **Marketplace Backend** | FastAPI + PostgreSQL | — | Simple HTTP API, plugin registry |

### 14.2 Infrastructure Stack

| Purpose | Technology | Notes |
|---------|-----------|-------|
| Container runtime | Docker / containerd | Existing |
| Orchestration | Kubernetes 1.32 + Helm | K8s mode |
| CI/CD | GitHub Actions + local runner | Existing |
| IaC | Terraform + Helm charts | Cloud deployments |
| Secrets | HashiCorp Vault / KMS | Enterprise mode |
| Monitoring | Prometheus + Grafana (existing) | Extend with agent metrics |
| Tracing | OpenTelemetry → Jaeger / Tempo | Extend v1.5.5 OTel |
| Logging | ECS structured JSON → Loki | Extend v1.5.5 logging |
| TLS | Let's Encrypt / cert-manager | HTTPS everywhere |

### 14.3 Agent AI Models (Default Assignments)

| Agent Type | Default Model | Fallback | Reason |
|------------|--------------|---------|--------|
| Architect, Planner | claude-opus-4-8 | claude-sonnet-4-6 | Complex reasoning |
| Backend, Frontend | claude-sonnet-4-6 | gpt-4o | Code generation sweet spot |
| Security | claude-sonnet-4-6 | gemini-2.0-flash | Security reasoning |
| QA | claude-haiku-4-5 | gpt-4o-mini | Volume task, cost-sensitive |
| Reviewer | claude-sonnet-4-6 | — | Code review quality |
| Documentation | claude-haiku-4-5 | — | Low complexity, high volume |
| Research | claude-sonnet-4-6 + web search | — | Tool-use capability |
| Learning | ollama/qwen2.5:7b | claude-haiku-4-5 | Local, privacy-sensitive |

---

## 15. Development Roadmap

### Phase 1 — Foundation (Months 1–3)

**Goal:** Minimum viable orchestration on top of v1.5.5.

| # | Deliverable | Effort | Owner |
|---|------------|--------|-------|
| 1.1 | Coordinator service (FastAPI, auth, session) | 3w | Core |
| 1.2 | Task Planner (NL→task decomposition) | 2w | Core |
| 1.3 | NATS JetStream integration | 1w | Core |
| 1.4 | Agent base class + BackendAgent (MVP) | 3w | Agent |
| 1.5 | Tool Runtime: file_reader, file_writer, shell_executor | 2w | Tools |
| 1.6 | PostgreSQL schema (tasks, executions, tool_calls) | 1w | Core |
| 1.7 | Human approval gates (API + Desktop dialog) | 2w | Desktop |
| 1.8 | Basic Desktop integration: Chat intake → tasks panel | 2w | Desktop |
| 1.9 | Agent Timeline view (Desktop) | 1w | Desktop |

**Exit Criteria:** BackendAgent can take a text requirement, generate Python files, run tests, and present for human approval.

---

### Phase 2 — Multi-Agent (Months 4–6)

| # | Deliverable | Effort |
|---|------------|--------|
| 2.1 | All 18 agent types (stub → working) | 6w |
| 2.2 | Dependency graph scheduler + parallel execution | 3w |
| 2.3 | Qdrant vector memory + Knowledge Engine (retrieval) | 3w |
| 2.4 | Kuzu knowledge graph + AST ingestion pipeline | 3w |
| 2.5 | Git operations tool (commit, branch, PR) | 2w |
| 2.6 | Retry engine + checkpointing | 2w |
| 2.7 | Plugin SDK v1 + 3 reference plugins | 3w |
| 2.8 | Memory browser (Desktop) | 1w |

**Exit Criteria:** 5 agents run in parallel on a feature request; memory persists across sessions; plugin installed from disk.

---

### Phase 3 — Intelligence (Months 7–9)

| # | Deliverable | Effort |
|---|------------|--------|
| 3.1 | Repository Intelligence (hotspot, complexity, debt) | 3w |
| 3.2 | Autonomous test generation (QAAgent full) | 3w |
| 3.3 | LearningAgent + outcome collection | 2w |
| 3.4 | AI Provider Gateway (all providers + fallback chain) | 2w |
| 3.5 | Cost tracking + budget gates | 1w |
| 3.6 | Observability dashboards v2 (agent timeline, prompt history) | 2w |
| 3.7 | Marketplace backend + CLI | 3w |
| 3.8 | Desktop: Marketplace panel, Memory browser full | 2w |

**Exit Criteria:** Platform runs SDLC for a real feature end-to-end with 80%+ autonomous decisions; LearningAgent improves test pass rate.

---

### Phase 4 — Enterprise (Months 10–12)

| # | Deliverable | Effort |
|---|------------|--------|
| 4.1 | OIDC/LDAP/SSO authentication | 2w |
| 4.2 | RBAC (roles, project permissions) | 2w |
| 4.3 | Secrets Vault integration (HashiCorp) | 2w |
| 4.4 | Multi-user workspace (concurrent users, project locking) | 3w |
| 4.5 | Kubernetes Helm chart + operator | 3w |
| 4.6 | CI Runner integration (GitHub Actions, GitLab CI, Jenkins) | 3w |
| 4.7 | Deployment Engine (Docker + K8s rollout) | 3w |
| 4.8 | Audit log export (SIEM integration) | 1w |
| 4.9 | Web UI (Next.js thin client over existing API) | 4w |

**Exit Criteria:** Platform deployed on K8s, 5 concurrent users, LDAP auth, full audit trail.

---

### Phase 5 — GA + Marketplace (Month 13–15)

| # | Deliverable | Effort |
|---|------------|--------|
| 5.1 | Performance hardening (targets in §19) | 3w |
| 5.2 | Security audit + pen test | 2w |
| 5.3 | Marketplace public launch (100+ plugins) | 4w |
| 5.4 | SaaS cloud deployment (multi-tenant) | 4w |
| 5.5 | Windows/macOS native installers | 1w |
| 5.6 | Documentation site | 2w |
| 5.7 | GA release + pricing | 2w |

---

## 16. Migration Plan from v1.5.5

### 16.1 Migration Principles

1. **Non-breaking.** v1.5.5 continues running unchanged. New components are additive.
2. **Incremental.** Users opt into v2 features per project.
3. **Shared persistence.** Coordinator writes to the same PostgreSQL instance, new tables only.
4. **Shared AI gateway.** v2 reuses v1.5.5 model routing; no duplicate provider configuration.

### 16.2 Migration Steps

```
STEP 1 — Deploy Coordinator alongside v1.5.5
  ├── Coordinator runs on :8090 (v1.5.5 AISF stays on :8088)
  ├── Both share PostgreSQL — Coordinator runs migrations for NEW tables only
  ├── Desktop v2 connects to Coordinator + legacy AISF endpoints
  └── Feature flag: aistudio.v2.enabled=false (default)

STEP 2 — Migrate projects one at a time
  ├── User selects project in Desktop → "Upgrade to AI Studio 2.0"
  ├── Coordinator indexes the repo (AST → Kuzu, embeddings → Qdrant)
  ├── Existing AI sessions continue working unchanged
  └── New tasks routed through Coordinator/AgentOrchestrator

STEP 3 — Enable agent-driven workflows
  ├── User submits a new requirement via v2 Chat intake
  ├── Coordinator plans, schedules agents, shows approval gates
  └── Existing v1.5.5 content pipeline unchanged

STEP 4 — Deprecate legacy endpoints (Month 6+)
  ├── v1.5.5 AISF API endpoints get thin proxies in Coordinator
  ├── Legacy Desktop panels redirect to v2 equivalents
  └── v1.5.5 AISF transitions to internal service (not user-facing)

STEP 5 — Full v2 (Month 12)
  └── v1.5.5 AISF, Content Factory run as internal microservices
      behind Coordinator — invisible to user, accessible to agents
```

### 16.3 Data Migration

| Data | Action |
|------|--------|
| PostgreSQL schemas | New tables added; v1.5.5 tables untouched |
| Configuration files | v2 reads v1.5.5 config; writes v2 config to separate path |
| AI provider settings | Inherited from v1.5.5 provider config (no re-entry) |
| Existing logs | Indexed into memory store on project upgrade |
| Existing metrics | Prometheus namespaces extended; no migration needed |
| Release artifacts | v1.5.5 pipeline continues; v2 Release Agent supplements |

---

## 17. Risk Assessment

### 17.1 Risk Matrix

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| **LLM context length limits** — agents need more context than model allows | High | High | Hierarchical summarization, KG-guided selective context, chunked analysis |
| **Hallucinated code** — agent writes plausible but wrong code | High | High | Mandatory test gate (no commit without passing tests), ReviewerAgent, human approval |
| **Runaway costs** — agents consume $100s in tokens on a single task | Medium | High | Per-task token budget, pre-execution cost estimate, budget kill switch |
| **Infinite retry loops** — agent repeatedly fails + retries same bad approach | Medium | Medium | Max attempt limits, backoff, escalate-to-human after N failures |
| **Data loss in KG** — knowledge graph corrupted or stale | Low | High | Incremental ingestion (idempotent MERGE), repo re-index on demand, PG as source of truth |
| **Sandbox escape** — malicious plugin or prompt injection reaches host | Low | Critical | gVisor isolation, deny-list for network calls, plugin signing, content policy on tool inputs |
| **Multi-user conflicts** — two agents modify the same file | Medium | Medium | File-level advisory locks in Redis, conflict detection in git_ops tool, branch-per-task |
| **Provider outage** — primary AI provider down | Medium | Medium | Fallback chain (claude → gpt-4o → local Ollama), circuit breaker per provider |
| **Stale memory** — agents make decisions based on outdated architecture decisions | Medium | Medium | Memory TTLs, "last verified" timestamps, ReviewerAgent cross-checks KG |
| **Human approval fatigue** — too many gates → users approve blindly | High | Medium | Configurable gate thresholds, approval batching, risk-tiered gates (low risk = auto-approve) |
| **Plugin supply chain** — malicious plugin in marketplace | Low | Critical | Plugin signing (GPG), sandboxed execution, code review for verified publishers |
| **GDPR/compliance** — AI processes customer code containing PII | Medium | High | Data residency controls, local-only mode, PII detection before sending to external models |

### 17.2 Security Controls

| Control | Implementation |
|---------|---------------|
| Authentication | API key (dev), OIDC JWT (enterprise), LDAP bind |
| Authorization | RBAC with project-level permissions; agents inherit project role |
| Secret management | Vault integration; secrets never in plaintext in DB |
| Network isolation | Tool Runtime sandbox; no egress except allow-list |
| Audit trail | Immutable `events` table (append-only), exported to SIEM |
| Data encryption | TLS 1.3 in transit; AES-256 at rest (PostgreSQL, Qdrant, S3) |
| Plugin trust | Signed plugins from verified publishers; unsigned blocked by default |
| Prompt injection | Input sanitization before tool calls; no user-supplied tool names |
| Rate limiting | Per-user, per-project, per-model rate limits via Redis |

---

## 18. Testing Strategy

### 18.1 Test Pyramid

```
                        ┌───────────────┐
                        │  E2E Tests    │  10%
                        │  (real agents,│
                        │  real models) │
                       ┌┴───────────────┴┐
                       │  Integration    │  30%
                       │  (mocked AI,    │
                       │  real infra)    │
                      ┌┴─────────────────┴┐
                      │  Unit Tests        │  60%
                      │  (all components,  │
                      │  no I/O)           │
                      └────────────────────┘
```

### 18.2 Test Categories

| Category | Scope | Tools | Gate |
|----------|-------|-------|------|
| **Unit** | Task Planner logic, Agent base, Tool validation, Event schemas | pytest, unittest.mock | CI gate ≥80% |
| **Integration** | Coordinator + NATS + PostgreSQL, Agent + Tool Runtime, KG ingestion | pytest + testcontainers | CI gate |
| **Agent Contract** | Each agent produces valid artifacts for its task type | pytest + fixture tasks | CI gate |
| **E2E — Happy Path** | Full requirement → code → test → approval → commit flow | pytest + real Claude API | Pre-release |
| **E2E — Failure Scenarios** | Agent failure, retry, human rejection, checkpoint resume | pytest | Pre-release |
| **Chaos** | NATS partition, PostgreSQL restart, agent kill mid-task | chaostoolkit | Monthly |
| **Cost regression** | Same benchmark task should not exceed token budget by 20% | CI scheduled | Weekly |
| **Security** | Plugin sandbox escape, prompt injection, RBAC enforcement | semgrep, manual pen test | Pre-GA |
| **Performance** | Task throughput, agent scheduling latency, KG query time | Locust, k6 | Pre-GA |
| **Plugin API** | Plugin SDK contract compatibility | pytest | On SDK change |

### 18.3 Benchmark Task Suite

A reproducible set of 20 software tasks (stored in `tests/benchmarks/`) used to measure:
- Autonomous success rate (task completed without human intervention)
- Time to completion
- Cost per task
- Code quality score (test pass rate, lint score, reviewer score)
- Memory hit rate (relevant memory retrieved for similar past task)

Target: 90%+ success rate on benchmark suite by GA.

---

## 19. Performance Targets

### 19.1 Latency Targets

| Operation | P50 | P99 | Notes |
|-----------|-----|-----|-------|
| Requirement intake → task plan | <5s | <15s | LLM decomposition |
| Agent scheduler: claim task | <200ms | <1s | Queue + dependency check |
| KG query (single hop) | <50ms | <200ms | Kuzu / Neo4j |
| Vector search (top-10) | <100ms | <500ms | Qdrant HNSW |
| Memory context assembly | <2s | <5s | KG + vector + PG |
| Tool: file_writer | <10ms | <100ms | Local I/O |
| Tool: shell_executor (build) | <300s | <600s | Bounded by project size |
| Approval gate: UI notification | <500ms | <2s | WebSocket push |
| Plugin load | <1s | <3s | pip importlib |

### 19.2 Throughput Targets

| Metric | Target (laptop) | Target (8-core server) | Target (K8s 16-node) |
|--------|----------------|----------------------|---------------------|
| Concurrent agent workers | 4 | 16 | 256 |
| Tasks/hour (simple) | 20 | 80 | 1000+ |
| Tasks/hour (complex SDLC) | 2 | 8 | 100 |
| Concurrent users | 1 | 10 | 1000 |
| KG nodes per project | 50K | 500K | 5M |
| Vector memories per project | 10K | 100K | 1M |

### 19.3 Reliability Targets

| Metric | Target |
|--------|--------|
| Task checkpoint/resume success rate | ≥99% |
| Agent crash recovery time | <30s |
| Data durability (no task lost on pod restart) | 100% |
| NATS message delivery guarantee | at-least-once (JetStream) |
| Coordinator uptime | 99.9% (3-nines) |
| Approval gate notification delivery | 99.99% |

### 19.4 Quality Targets

| Metric | Target at GA |
|--------|-------------|
| Autonomous task success rate (benchmark suite) | ≥90% |
| Generated code: tests pass rate (first attempt) | ≥70% |
| Generated code: tests pass after retry (≤3 attempts) | ≥95% |
| Memory retrieval precision@10 | ≥0.75 |
| KG accuracy (compared to manual code analysis) | ≥95% |
| Average cost per "add a CRUD endpoint" task | <$0.50 USD |

---

## 20. Release Plan

### 20.1 Version Milestones

| Version | Target Date | Scope |
|---------|------------|-------|
| **2.0.0-alpha1** | Month 3 | Phase 1 complete — single BackendAgent, Desktop integration |
| **2.0.0-alpha2** | Month 6 | Phase 2 complete — all agents, vector memory, KG, parallel execution |
| **2.0.0-beta1** | Month 9 | Phase 3 complete — repo intelligence, test gen, marketplace alpha |
| **2.0.0-beta2** | Month 11 | Phase 4 partial — K8s, OIDC, multi-user |
| **2.0.0-rc1** | Month 13 | All phases complete — performance tuned, security audited |
| **2.0.0 GA** | Month 15 | Marketplace launch, SaaS, commercial licensing |

### 20.2 Release Channels

| Channel | Audience | Stability | Update frequency |
|---------|---------|-----------|-----------------|
| `nightly` | Internal developers | Unstable | Daily |
| `alpha` | Trusted early adopters | Expect breaking changes | Weekly |
| `beta` | Community testers | Feature complete, bugs possible | Bi-weekly |
| `stable` | Production users | Fully tested | Monthly (bugfix: as-needed) |
| `lts` | Enterprise | 18-month support window | Quarterly |

### 20.3 Pricing Model (Commercial)

| Tier | Target | Price | Includes |
|------|--------|-------|---------|
| **Community** | Individual devs | Free | Local mode, 4 agents, 10K memories, Ollama only |
| **Pro** | Professional devs | $49/mo | Cloud AI providers, 16 agents, 100K memories, marketplace |
| **Team** | Small teams (≤20) | $199/mo | Multi-user, SSO, 32 agents, priority queue |
| **Enterprise** | Large orgs | Custom | LDAP, Vault, K8s, SLA, on-prem, custom agents |
| **Marketplace** | Plugin authors | Revenue share 70/30 | Distribution, signing, hosting |

### 20.4 Launch Strategy

1. **Month 10** — Developer preview: open beta with Community tier, GitHub public repo
2. **Month 12** — Plugin marketplace opens: invite 20 partner plugin authors
3. **Month 14** — Pro tier launches: Stripe payment, cloud AI provider keys, usage dashboard
4. **Month 15** — GA: press release, Product Hunt launch, Enterprise outreach
5. **Month 18** — First LTS release (2.0 LTS): 18-month security support commitment

---

## Appendix A — Glossary

| Term | Definition |
|------|-----------|
| **Agent** | A stateless AI worker that executes a specific type of task |
| **Coordinator** | The central API service managing auth, session, and routing |
| **Gate** | A human approval checkpoint before a destructive action |
| **KG** | Knowledge Graph — structural representation of code and relationships |
| **Orchestrator** | The scheduler that assigns tasks to agents and resolves dependencies |
| **Tool** | A capability (file I/O, shell, HTTP) available to agents |
| **Memory** | Persistent, searchable information stored across sessions |
| **Task** | The atomic work unit: a requirement decomposed into an assignable item |
| **Plugin** | An installable extension: agent, tool, model, workflow, or template |
| **Sandbox** | Isolated execution environment for untrusted tool calls |

---

## Appendix B — ADRs (Architecture Decision Records)

**ADR-001: NATS JetStream over RabbitMQ for agent messaging**  
_Decision:_ Use NATS JetStream for the v2 agent message bus; keep RabbitMQ for v1.5.5 Content Factory workers.  
_Rationale:_ NATS has a single 20 MB binary, embedded mode for local dev, native K8s operator, and subject-based routing that maps cleanly to agent addressing. RabbitMQ migration would be a breaking change to v1.5.5.

**ADR-002: Kuzu for local KG, Neo4j for production**  
_Decision:_ Kuzu (embedded) for local/dev mode; Neo4j for team/K8s mode.  
_Rationale:_ Kuzu requires zero server setup — same philosophy as SQLite. Neo4j is the industry standard with full Cypher support and a managed cloud offering. Same Cypher queries run on both.

**ADR-003: Agents are stateless; all state in persistence layer**  
_Decision:_ No agent holds state in memory between tool calls.  
_Rationale:_ Enables kill-restart without data loss, horizontal scaling, and checkpoint/resume. Checkpoints are JSON-serializable snapshots stored in `tasks.checkpoint_data`.

**ADR-004: Human gates are mandatory by default**  
_Decision:_ commit, deploy, delete always require human approval unless `unattended_mode=true` per project.  
_Rationale:_ Prevents AI from making irreversible mistakes autonomously. Users can opt into unattended mode once they trust a specific agent/project combination.

**ADR-005: Plugin SDK wraps, not replaces, the agent base**  
_Decision:_ All plugin agents extend `BaseAgent`; tool plugins extend `BaseTool`.  
_Rationale:_ Ensures lifecycle, logging, cost tracking, and sandbox apply uniformly to all plugins.

---

*Document version: 2.0.0-DRAFT — AI Studio 2.0 Architecture*  
*Author: AI Studio Architecture Team*  
*Last updated: 2026-06-27*
