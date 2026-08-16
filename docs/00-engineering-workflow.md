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
7. **Modular Documentation Allocation:** New business rules, functional requirements, acceptance criteria, or technical specs MUST be placed into their dedicated document with explicit cross-referencing (`BR-xx` in `docs/governance/business-rules.md`, `FR-xx` in `docs/stages/02-functional-requirements.md`, `AC-xx` in `docs/stages/acceptance-criteria.md`). Never bloat `MASTER.md`.
8. **Mandatory Glossary Maintenance:** [`docs/governance/glossary.md`](governance/glossary.md) MUST be updated whenever new terms, abbreviations, domain concepts, or document references are introduced or used anywhere in the workspace.
9. **Document Minimization & Modular Consolidation Rule:** Before creating any new document, review all existing project documentation to determine whether the new requirement, rule, decision, or information can be appropriately incorporated into an existing document. Do NOT create a new document for every new requirement or update. Prefer updating an existing relevant document when it provides an appropriate place. Avoid document duplication and fragmentation. Only create a standalone document if the requirement represents a distinct concern that genuinely provides value as a dedicated file.

### Principles

- Think before coding. Challenge assumptions. Explain tradeoffs.
- Prefer maintainability over shortcuts; keep modular and scalable without overengineering.
- Complete bidirectional traceability:

```text
Requirement (FR-xx/NFR-xx) → Acceptance criteria (AC-xx.y) → Design decision → Implementation → Test(s)
```

---

## Document Map

| File | Subdirectory / Scope | Status |
| :--- | :--- | :--- |
| [00-engineering-workflow.md](00-engineering-workflow.md) | Top-Level Entry Map | Active |
| [governance/business-rules.md](governance/business-rules.md) | Governance (authoritative rules) | Active — single source of truth for domain logic |
| [governance/glossary.md](governance/glossary.md) | Governance (abbreviations & terms) | Active — update when new terms appear |
| [governance/rbac-matrix.md](governance/rbac-matrix.md) | Governance (RBAC matrix & security guards) | Active — single source of truth for authorization |
| [governance/ttl-registry.md](governance/ttl-registry.md) | Governance (token lifecycles & TTLs) | Active — single source of truth for TTLs |
| [stages/01-requirement-analysis.md](stages/01-requirement-analysis.md) | Stages (Stage 01) | Confirmed |
| [stages/02-functional-requirements.md](stages/02-functional-requirements.md) | Stages (Stage 02) | Confirmed |
| [stages/acceptance-criteria.md](stages/acceptance-criteria.md) | Stages (Stage 02 companion) | Confirmed |
| [stages/03-non-functional-requirements.md](stages/03-non-functional-requirements.md) | Stages (Stage 03) | Confirmed |
| [stages/04-domain-model.md](stages/04-domain-model.md) | Stages (Stage 04) | Under Review (Pending Confirmation) |
| [domain/audit-logging.md](domain/audit-logging.md) | Domain (BR-16 Audit Trail) | Active — single source of truth for audit logs |
| [domain/billing-and-reconciliation.md](domain/billing-and-reconciliation.md) | Domain (BR-02/03 Billing Engine) | Active — single source of truth for billing |
| [domain/building-occupancy.md](domain/building-occupancy.md) | Domain (BR-17 Occupancy Engine) | Active — single source of truth for occupancy |
| [domain/file-storage-and-upload-policy.md](domain/file-storage-and-upload-policy.md) | Domain (BR-18 File Upload & R2) | Active — single source of truth for storage policy |
| [domain/frontend-navigation.md](domain/frontend-navigation.md) | Domain (BR-17 Responsive UX) | Active — single source of truth for navigation |
| [technical/api-contracts.md](technical/api-contracts.md) | Technical (REST Endpoints) | Active — single source of truth for API contracts |
| [technical/architecture.md](technical/architecture.md) | Technical (System Architecture) | Active — single source of truth for system design |
| [technical/database-schema.md](technical/database-schema.md) | Technical (PostgreSQL & Prisma) | Active — single source of truth for schema DDL |
| [technical/deployment-strategy.md](technical/deployment-strategy.md) | Technical (Deployment & Hosting) | Active — living deployment plan |
| [technical/error-handling.md](technical/error-handling.md) | Technical (Error Codes & Envelopes)| Active — single source of truth for errors |
| [stages/05-database-design.md](stages/05-database-design.md) | Stages (Stage 05) | Pending |
| [stages/06-api-design.md](stages/06-api-design.md) | Stages (Stage 06) | Pending |
| [stages/07-frontend-architecture.md](stages/07-frontend-architecture.md) | Stages (Stage 07) | Pending |
| [stages/08-backend-architecture.md](stages/08-backend-architecture.md) | Stages (Stage 08) | Pending |
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
**Output:** `docs/stages/01-requirement-analysis.md`  
Clarify problem domain, multi-building scope, locked technical stack, in/out of scope boundaries, and open business questions. No code.

### Stage 02 — Functional Requirements
**Output:** `docs/stages/02-functional-requirements.md` + `docs/stages/acceptance-criteria.md`  
Break capabilities into numbered **FR-xx** requirements and testable **AC-xx.y** criteria (Building hierarchy, Tenant onboarding, Rent cycles, Independent Rent vs Electricity ledgers, Complaints).

### Stage 03 — Non-Functional Requirements
**Output:** `docs/stages/03-non-functional-requirements.md`  
Capture **NFR-xx**: Financial data immutability, query performance targets, security (JWT, RBAC), error contracts, and auditability.

### Stage 04 — Domain Model
**Output:** `docs/stages/04-domain-model.md`  
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
