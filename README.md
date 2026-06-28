# AI Studio — Architecture Index

**Workspace Standard:** v1.0  
**Created:** 2026-06-27  
**Maintained by:** Platform Engineering

---

## Workspace Map

```
AIStudio/
├── architecture/           <- You are here
│   ├── docs/               <- All versioned architecture documents
│   │   ├── 1.5.5/          <- Production platform (current)
│   │   ├── 2.0/            <- Next generation (specification phase)
│   │   ├── 3.0/            <- Vision documents
│   │   ├── 3.5/            <- Universal Product Factory + Enterprise Engine
│   │   ├── implementation/ <- Engineering implementation guides
│   │   ├── standards/      <- Coding and API standards
│   │   └── roadmap/        <- Roadmap documents
│   ├── diagrams/           <- Architecture diagrams (PNG, SVG, draw.io)
│   ├── adr/                <- Architecture Decision Records
│   ├── decisions/          <- Design decision logs (lightweight)
│   └── assets/             <- Shared architecture assets
│
├── source/                 <- Source code repositories (not yet migrated)
├── assets/                 <- Platform assets (prompts, blueprints, workflows)
├── examples/               <- Working examples and demos
├── playground/             <- Experimental and scratch work
├── releases/               <- Release artifacts and changelogs
├── scripts/                <- Maintenance and automation scripts
└── archive/                <- Superseded documents and legacy artifacts
```

---

## Document Index

### v1.5.5 — Production Platform

| Document | Description |
|----------|-------------|
| [AI-STUDIO-1.5.5-REVERSE-ENGINEERING-SPEC.md](docs/1.5.5/AI-STUDIO-1.5.5-REVERSE-ENGINEERING-SPEC.md) | Complete reverse-engineered specification of the v1.5.5 platform. 1,818 lines. VERIFIED/PARTIALLY/TEMPLATE/NOT_IMPL classification for every feature. Canonical pre-2.0 reference. |
| [PLATFORM-CERTIFICATION-v1.5.1.md](docs/1.5.5/PLATFORM-CERTIFICATION-v1.5.1.md) | Platform certification evidence for v1.5.1 |
| [PLATFORM-CERTIFICATION-v1.5.2.md](docs/1.5.5/PLATFORM-CERTIFICATION-v1.5.2.md) | Platform certification evidence for v1.5.2 |
| [PLATFORM-CERTIFICATION-v1.5.3.md](docs/1.5.5/PLATFORM-CERTIFICATION-v1.5.3.md) | Platform certification evidence for v1.5.3 |
| [GA-CERTIFICATION.md](docs/1.5.5/GA-CERTIFICATION.md) | General Availability certification report |
| [CI-CD-REPORT-v1.5.5.md](docs/1.5.5/CI-CD-REPORT-v1.5.5.md) | CI/CD pipeline report for v1.5.5 |
| [OBSERVABILITY-REPORT-v1.5.4.md](docs/1.5.5/OBSERVABILITY-REPORT-v1.5.4.md) | Observability stack validation |
| [LOGGING-IMPLEMENTATION-v1.5.4.md](docs/1.5.5/LOGGING-IMPLEMENTATION-v1.5.4.md) | Structured logging implementation |
| [RELIABILITY-REPORT-v1.5.4.md](docs/1.5.5/RELIABILITY-REPORT-v1.5.4.md) | Reliability and uptime validation |
| [SECURITY-REPORT-v1.5.2.md](docs/1.5.5/SECURITY-REPORT-v1.5.2.md) | Security audit report v1.5.2 |
| [SECURITY-VALIDATION-REPORT-v1.5.3.md](docs/1.5.5/SECURITY-VALIDATION-REPORT-v1.5.3.md) | Security validation v1.5.3 |
| [SUPPLY-CHAIN-SECURITY.md](docs/1.5.5/SUPPLY-CHAIN-SECURITY.md) | Supply chain security audit |
| [PERFORMANCE-BASELINE.md](docs/1.5.5/PERFORMANCE-BASELINE.md) | Performance baseline measurements |
| [PERFORMANCE-BENCHMARK-REPORT-v1.5.3.md](docs/1.5.5/PERFORMANCE-BENCHMARK-REPORT-v1.5.3.md) | Benchmark results v1.5.3 |
| [DEPLOYMENT-GUIDE-v1.5.3.md](docs/1.5.5/DEPLOYMENT-GUIDE-v1.5.3.md) | Deployment procedures v1.5.3 |
| [DISASTER-RECOVERY-GUIDE-v1.5.3.md](docs/1.5.5/DISASTER-RECOVERY-GUIDE-v1.5.3.md) | Disaster recovery playbook |
| [OPERATIONAL-RUNBOOK-v1.5.3.md](docs/1.5.5/OPERATIONAL-RUNBOOK-v1.5.3.md) | Day-2 operations runbook |
| [RELEASE-CHECKLIST.md](docs/1.5.5/RELEASE-CHECKLIST.md) | Release gate checklist |
| [SBOM-REPORT.md](docs/1.5.5/SBOM-REPORT.md) | Software Bill of Materials |

**Full list:** 48 documents covering certification, testing, operations, security, and release for v1.5.x cycle. See `docs/1.5.5/` for complete set.

---

### v2.0 — Next Generation (Specification Phase)

| Document | Description |
|----------|-------------|
| [AI-STUDIO-2.0-ARCHITECTURE.md](docs/2.0/AI-STUDIO-2.0-ARCHITECTURE.md) | Primary 2.0 system architecture. 1,708 lines. 7-layer platform, 18 agents, polyrepo strategy, full DDL, NATS event model. |
| [AI-STUDIO-2.0-EXTENSION.md](docs/2.0/AI-STUDIO-2.0-EXTENSION.md) | Architecture extension document. Full text (2,593 lines). |
| [AI-STUDIO-2.0-EXTENSION-PART1.md](docs/2.0/AI-STUDIO-2.0-EXTENSION-PART1.md) | Extension Part 1 of 3 |
| [AI-STUDIO-2.0-EXTENSION-PART2.md](docs/2.0/AI-STUDIO-2.0-EXTENSION-PART2.md) | Extension Part 2 of 3 |
| [AI-STUDIO-2.0-EXTENSION-PART3.md](docs/2.0/AI-STUDIO-2.0-EXTENSION-PART3.md) | Extension Part 3 of 3 |

---

### v3.0 — Vision

| Document | Description |
|----------|-------------|
| [AI-STUDIO-3.0-VISION.md](docs/3.0/AI-STUDIO-3.0-VISION.md) | Long-term platform vision. 628 lines. |

---

### v3.5 — Universal Product Factory + Enterprise Execution Engine

| Document | Description |
|----------|-------------|
| [AI-STUDIO-3.5-UNIVERSAL-PRODUCT-FACTORY.md](docs/3.5/AI-STUDIO-3.5-UNIVERSAL-PRODUCT-FACTORY.md) | Universal Product Factory architecture. 2,944 lines. One-sentence-in/full-product-out model. |
| [AI-STUDIO-3.5.1-ENTERPRISE-EXECUTION-ENGINE.md](docs/3.5/AI-STUDIO-3.5.1-ENTERPRISE-EXECUTION-ENGINE.md) | Enterprise Execution Engine. 4,425 lines. ExecutionContext, workspace layout, workflow compiler. |

---

### Master Architecture

| Document | Description |
|----------|-------------|
| [AI-STUDIO-MASTER-ARCHITECTURE.md](docs/AI-STUDIO-MASTER-ARCHITECTURE.md) | Consolidated master architecture document. 8,008 lines. Authoritative cross-version reference. |

---

### Implementation

| Document | Description |
|----------|-------------|
| [AI-STUDIO-2.0-IMPLEMENTATION-GUIDE.md](docs/implementation/AI-STUDIO-2.0-IMPLEMENTATION-GUIDE.md) | Engineering implementation guide. 5,757 lines, 198 KB. 20 parts covering all services, DDL, APIs, events, deployment, monitoring, testing. No placeholders. Ready for engineering team. |

---

## Reading Order

### For a new engineer joining the platform team:

```
1. AI-STUDIO-1.5.5-REVERSE-ENGINEERING-SPEC.md
   Understand what is currently in production.

2. AI-STUDIO-2.0-ARCHITECTURE.md
   Understand what 2.0 is designed to be.

3. AI-STUDIO-2.0-IMPLEMENTATION-GUIDE.md (Parts 1-5)
   Repository layout, dependencies, runtime lifecycle, database, coordinator.

4. AI-STUDIO-2.0-IMPLEMENTATION-GUIDE.md (Parts 6-10)
   Agent runtime, event bus, memory, brain, REST API.

5. AI-STUDIO-2.0-IMPLEMENTATION-GUIDE.md (Parts 11-20)
   Desktop, web, mobile, marketplace, deployment, monitoring, testing,
   migration, performance, roadmap.
```

### For an architect reviewing the full vision:

```
1. AI-STUDIO-MASTER-ARCHITECTURE.md       (unified view)
2. AI-STUDIO-3.0-VISION.md               (where it goes)
3. AI-STUDIO-3.5-UNIVERSAL-PRODUCT-FACTORY.md  (product strategy)
4. AI-STUDIO-3.5.1-ENTERPRISE-EXECUTION-ENGINE.md  (execution model)
5. AI-STUDIO-2.0-ARCHITECTURE.md         (current spec)
```

### For a release engineer:

```
1. docs/1.5.5/RELEASE-CHECKLIST.md
2. docs/1.5.5/DEPLOYMENT-GUIDE-v1.5.3.md
3. docs/1.5.5/OPERATIONAL-RUNBOOK-v1.5.3.md
4. docs/1.5.5/DISASTER-RECOVERY-GUIDE-v1.5.3.md
```

---

## Architecture Evolution

```
v1.0 - v1.4   [legacy, not documented here]
     |
     v
v1.5.0         Single-process Python runtime, SQLite, polling orchestration
v1.5.1         Platform certification, PLATFORM-CERTIFICATION-v1.5.1.md
v1.5.2         Security hardening, API compatibility (v1.5.2 reports)
v1.5.3         Production hardening, deployment guides, disaster recovery
v1.5.4         Observability: structured logging, OpenTelemetry, Prometheus
v1.5.5 [PROD]  Current production. CI/CD certified. GA certified.
     |
     v
v2.0.0 [SPEC]  Distributed polyrepo. 9 services. NATS JetStream. 18 agents.
               PostgreSQL + Qdrant + Kuzu. Human approval gates.
     |
     v
v3.0   [VISION] Extended AI company model. Research + UX + PM agents.
     |
     v
v3.5   [VISION] Universal Product Factory. One-sentence-in/full-product-out.
v3.5.1 [VISION] Enterprise Execution Engine. Multi-tenant, audit, compliance.
```

---

## Document Ownership

| Domain | Owner | Documents |
|--------|-------|-----------|
| Platform Architecture | Platform Engineering Lead | 2.0-ARCHITECTURE, MASTER-ARCHITECTURE |
| Implementation Guide | Lead Platform Engineer | 2.0-IMPLEMENTATION-GUIDE |
| v1.5.5 Production | Release Engineering | All docs/1.5.5/* |
| Vision / Strategy | CTO / Product | 3.0-VISION, 3.5-*, 3.5.1-* |
| Security | Security Engineering | SECURITY-*, SUPPLY-CHAIN-*, CODE-SIGNING-* |
| Operations | Platform SRE | OPERATIONAL-RUNBOOK, DISASTER-RECOVERY, DEPLOYMENT-GUIDE |

---

## Version History

| Date | Change | Author |
|------|--------|--------|
| 2026-06-27 | Workspace Standard v1.0 created. 59 documents organized into AIStudio structure. | Platform Engineering |
| 2026-06-27 | AI-STUDIO-2.0-IMPLEMENTATION-GUIDE.md completed (5,757 lines). | Platform Engineering |
| 2026-06-27 | AI-STUDIO-1.5.5-REVERSE-ENGINEERING-SPEC.md completed (1,818 lines). | Platform Engineering |
