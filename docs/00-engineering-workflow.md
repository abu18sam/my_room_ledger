# Engineering Workflow & Stage Tracker

**Project:** Room & Rent Ledger Management System (*My Room Ledger*)  
**Purpose:** Single source of truth for *how* we proceed, *what* each stage produces, and *where* we are — so progress remains traceable across chat sessions.

---

## How We Proceed

1. Work **one stage at a time** in the order below. Do not skip ahead.
2. At each stage: analyze $\rightarrow$ document reasoning/tradeoffs $\rightarrow$ **wait for user confirmation**.
3. After confirmation, write (or finalize) that stage's markdown under `/docs`.
4. Only then move to the next stage.
5. **Never invent requirements.** If something is ambiguous, stop and ask.
6. **Traceability is mandatory:** every implementation must cite requirement IDs, acceptance criteria, tests, and any confirmed assumptions.
7. **Modular Documentation Allocation:** New business rules, functional requirements, acceptance criteria, or technical specs MUST be placed into their dedicated document with explicit cross-referencing (`BR-xx` in `docs/business-rules.md`, `FR-xx` in `docs/02-functional-requirements.md`, `AC-xx` in `docs/acceptance-criteria.md`). Never bloat `MASTER.md`.
8. **Mandatory Glossary Maintenance:** [`docs/glossary.md`](glossary.md) MUST be updated whenever new terms, abbreviations, domain concepts, or document references are introduced or used anywhere in the workspace.

### Principles

- Think before coding. Challenge assumptions. Explain tradeoffs.
- Prefer maintainability over shortcuts; keep modular and scalable without overengineering.
- Complete bidirectional traceability:

```text
Requirement (FR-xx/NFR-xx) → Acceptance criteria (AC-xx.y) → Design decision → Implementation → Test(s)
```

---

## Document Map

| File | Stage | Status |
| :--- | :--- | :--- |
| [00-engineering-workflow.md](00-engineering-workflow.md) | Meta (this file) | Active |
| [business-rules.md](business-rules.md) | Meta (authoritative business rules) | Active — single source of truth for domain logic |
| [glossary.md](glossary.md) | Meta (abbreviations & terms) | Active — update when new terms appear |
| [deployment-strategy.md](deployment-strategy.md) | Meta (living deployment plan) | Active — continuously refined across stages |
| [01-requirement-analysis.md](01-requirement-analysis.md) | Stage 01 | Confirmed |
| [02-functional-requirements.md](02-functional-requirements.md) | Stage 02 | Confirmed |
| [acceptance-criteria.md](acceptance-criteria.md) | Stage 02 (companion) | Confirmed |
| [03-non-functional-requirements.md](03-non-functional-requirements.md) | Stage 03 | Confirmed |
| [04-domain-model.md](04-domain-model.md) | Stage 04 | Under Review (Pending Confirmation) |
| [05-database-design.md](05-database-design.md) | Stage 05 | Pending |
| [06-api-design.md](06-api-design.md) | Stage 06 | Pending |
| [07-frontend-architecture.md](07-frontend-architecture.md) | Stage 07 | Pending |
| [08-backend-architecture.md](08-backend-architecture.md) | Stage 08 | Pending |
| [09-folder-structure.md](09-folder-structure.md) | Stage 09 | Pending |
| [10-technical-decisions.md](10-technical-decisions.md) | Stage 10 | Pending |
| [11-development-plan.md](11-development-plan.md) | Stage 11 | Pending |
| [12-task-breakdown.md](12-task-breakdown.md) | Stage 12 | Pending |
| [13-implementation.md](13-implementation.md) | Stage 13 | Pending |
| [14-testing.md](14-testing.md) | Stage 14 | Pending |
| [15-debugging.md](15-debugging.md) | Stage 15 | Pending |
| [16-code-review.md](16-code-review.md) | Stage 16 | Pending |
| [17-documentation.md](17-documentation.md) | Stage 17 | Pending |
| [18-repo-restructure.md](18-repo-restructure.md) | Stage 18 | Pending |

**Current Position:** **Stage 04 Under Review** — Domain Model & State Machines (Pending User Confirmation).


---

## Stages (What We Do in Each)

### Stage 01 — Requirement Analysis
**Output:** `docs/01-requirement-analysis.md`  
Clarify problem domain, multi-building scope, locked technical stack, in/out of scope boundaries, and open business questions. No code.

### Stage 02 — Functional Requirements
**Output:** `docs/02-functional-requirements.md` + `docs/acceptance-criteria.md`  
Break capabilities into numbered **FR-xx** requirements and testable **AC-xx.y** criteria (Building hierarchy, Tenant onboarding, Rent cycles, Independent Rent vs Electricity ledgers, Complaints).

### Stage 03 — Non-Functional Requirements
**Output:** `docs/03-non-functional-requirements.md`  
Capture **NFR-xx**: Financial data immutability, query performance targets, security (JWT, RBAC), error contracts, and auditability.

### Stage 04 — Domain Model
**Output:** `docs/04-domain-model.md`  
Entities, fields, relationships, enums, independent financial state machines (Rent vs Electricity), tenancy lifecycle, complaint ticket workflow.

### Stage 05 — Database Design
**Output:** `docs/05-database-design.md`  
Tables, columns, foreign keys, indexes, Flyway migration scripts, seed strategy. Map tables to domain entities.

### Stage 06 — API Design
**Output:** `docs/06-api-design.md`  
RESTful endpoints, JSON request/response shapes, authentication headers, HTTP status codes, error shapes.

### Stage 07 — Frontend Architecture
**Output:** `docs/07-frontend-architecture.md`  
React 18 SPA structure, page routes, Landlord Control Dashboard, Tenant Portal UI, component hierarchy, state management.

### Stage 08 — Backend Architecture
**Output:** `docs/08-backend-architecture.md`  
Node.js (TypeScript) **NestJS 10** modular API architecture — Modules, Controllers, Services, Guards (JWT RBAC), Interceptors, Pipes (Zod validation), Filters (error handling), Prisma data access layer, Pino logging.

### Stage 09 — Folder Structure
**Output:** `docs/09-folder-structure.md`  
Complete repository layout for backend, frontend, database scripts, and documentation.

### Stage 10 — Technical Decisions
**Output:** `docs/10-technical-decisions.md`  
Architecture Decision Records (ADRs) explaining technical tradeoffs, schema choices, snapshotting strategies.

### Stage 11 — Development Plan
**Output:** `docs/11-development-plan.md`  
Phased implementation roadmap, module dependencies, sprint milestones.

### Stage 12 — Task Breakdown
**Output:** `docs/12-task-breakdown.md`  
Granular task items linked directly to requirement IDs (`FR-xx`) and acceptance criteria (`AC-xx.y`).

### Stage 13 — Implementation
**Output:** `docs/13-implementation.md`  
Source code execution tracking across backend entities, repositories, services, controllers, and frontend views.

### Stage 14 — Testing
**Output:** `docs/14-testing.md`  
Unit test cases, integration tests, ledger financial independence assertions, frontend component validation.

### Stage 15 — Debugging
**Output:** `docs/15-debugging.md`  
Defect tracking log, root-cause analyses, fix verifications.

### Stage 16 — Code Review
**Output:** `docs/16-code-review.md`  
Code quality review notes, static analysis, refactoring tasks.

### Stage 17 — Documentation
**Output:** `docs/17-documentation.md`  
User manuals for Landlords and Tenants, local setup guide, environment configuration.

### Stage 18 — Repository Restructure
**Output:** `docs/18-repo-restructure.md`  
Root cleanup, production build configuration, deployment readiness.
