# Stage 03 — Non-Functional Requirements (NFRs)

**Status:** Confirmed & Locked ✅ (Approved by User on 2026-08-09)  
**Upstream:** [02-functional-requirements.md](02-functional-requirements.md) (Confirmed & Locked ✅), MASTER.md business rules  
**Downstream:** Stage 04 (Domain Model & State Machines)  
**Workflow tracker:** [00-engineering-workflow.md](00-engineering-workflow.md)

---

**Authoritative Baseline:** All non-functional requirements (NFRs) are grounded in [`docs/business-rules.md`](business-rules.md) and derived from Functional Requirements (`FR-01` through `FR-112`) in [`docs/02-functional-requirements.md`](02-functional-requirements.md).

---

## 1. Security, Authorization & Session Resilience

| NFR ID | Requirement | Metric / Constraint Target | Verification Method | Source Rule |
|---|---|---|---|---|
| **NFR-01** | **JWT & Token Security** | Signed JWTs (RS256 or HS256 with $\ge 256$-bit key). Access Token TTL = **10 minutes**; Refresh Token TTL = **7 days**. All token lifecycles governed by [`docs/ttl-registry.md`](ttl-registry.md). Protected routes require `@UseGuards(JwtAuthGuard, RolesGuard)`. | Security Audit & Unit Test assertions | BR-10.1, BR-01.1, docs/ttl-registry.md |
| **NFR-02** | **Password Hashing & Storage** | Passwords MUST be hashed exclusively using **Argon2id** (memory: 64MB / $65,536\text{ KiB}$, time cost: 3 iterations, parallelism: 4 threads) with unique per-user salt. Zero plaintext or bcrypt passwords stored or logged. | DB Column Inspection & Auth Service Unit Tests | BR-10.2, BR-11.4 |
| **NFR-03** | **Encrypted Cloud Object Storage** | Files uploaded to Cloudflare R2 MUST be encrypted at-rest using **AES-256 GCM**. File access served strictly via backend-generated signed URLs with **15-minute expiration** as defined in [`docs/ttl-registry.md`](ttl-registry.md). | API Integration Test & Storage Header Audit | BR-10.3, BR-08, docs/ttl-registry.md |
| **NFR-04** | **Session Revocation SLA** | Administrative force logout (`FORCE_LOGOUT_USER` / `FORCE_LOGOUT_ROLE`) MUST purge user active session records from PostgreSQL within **$\le 500\text{ ms}$**, immediately blocking token refresh attempts. | Performance & Integration Test | BR-01.5, BR-16.2 |
| **NFR-05** | **Input Sanitization & Injection Defense** | 100% of API endpoints validate request payloads via `ZodValidationPipe`. Database access via Prisma ORM parameterized queries to eliminate SQL injection, XSS, and parameter pollution. | Static Code Analysis & DAST Vulnerability Scan | BR-10.4, BR-15.3 |

---

## 2. Financial Data Immutability & Audit Integrity

| NFR ID | Requirement | Metric / Constraint Target | Verification Method | Source Rule |
|---|---|---|---|---|
| **NFR-06** | **Immutable Audit Trail** | `audit_logs` table is strictly insert-only. Database triggers and application layers MUST reject `UPDATE` or `DELETE` commands across ALL user roles (`HTTP 403` / SQL Exception). | Database Permission Audit & Integration Tests | BR-16.1, BR-16.3 |
| **NFR-07** | **Historical Snapshot Immutability** | `TenancyHistory`, `BuildingPowerConnection`, and settled `SupplierMasterBill` records are immutable. Historical tenant bills NEVER re-evaluate or mutate upon move-out or room modification. | Ledger Regression Tests | BR-08, BR-13.3 |
| **NFR-08** | **ACID Transaction Safety** | Financial ledger updates, payment transactions, and reconciliation status recalculations MUST execute within PostgreSQL serializable transactions (`prisma.$transaction`) with optimistic locks. | Concurrency & Race Condition Load Tests | BR-14.1, BR-14.7 |
| **NFR-09** | **Pass-Through Segregation Invariant** | DB queries and API aggregation services MUST enforce strict mathematical segregation: Rental Net Profit calculations MUST NEVER include electricity collections, surplus, or deficit amounts. | P&L Unit Test & SQL Verification | BR-03.1, BR-03.4 |

---

## 3. System Performance & Low Latency SLAs

| NFR ID | Requirement | Metric / Constraint Target | Verification Method | Source Rule |
|---|---|---|---|---|
| **NFR-10** | **API Response Time Targets** | Standard CRUD API endpoints $p95 \le \mathbf{200\text{ ms}}$; P&L financial aggregation and electricity reconciliation endpoints $p95 \le \mathbf{400\text{ ms}}$ under nominal load. | k6 Load Test Suite | BR-04.1 |
| **NFR-11** | **Database Indexing & UUIDv7 Efficiency** | All primary keys across all tables MUST use **UUIDv7** (time-ordered 128-bit identifiers leveraging sequential B-Tree index locality to prevent page splits). All foreign keys and search filters indexed. Zero unindexed sequential scans allowed. | PostgreSQL `EXPLAIN ANALYZE` Inspection | BR-10.4, BR-10.5 |
| **NFR-12** | **PWA Frontend Performance** | Next.js PWA targets: First Contentful Paint (FCP) $\le \mathbf{1.2\text{ s}}$, Largest Contentful Paint (LCP) $\le \mathbf{2.0\text{ s}}$, Time to Interactive (TTI) $\le \mathbf{2.5\text{ s}}$ on standard 4G networks. | Lighthouse PWA Audit | BR-10.6 |
| **NFR-13** | **Payload Size & Compression** | Gzip/Brotli HTTP compression enabled. File upload size capped at **10MB** per document with client-side WebP compression for UI avatars. | Network Profiling | BR-10.3 |

---

## 4. Reliability, Scalability & Infrastructure Boundaries

| NFR ID | Requirement | Metric / Constraint Target | Verification Method | Source Rule |
|---|---|---|---|---|
| **NFR-14** | **Low-Cost Infrastructure Footprint** | Backend micro-monolith memory footprint $\le \mathbf{256\text{ MB}}$ RAM. Operable on low-cost hosting ($0–$5/mo budget tier, single-vcpu VPS / Render / Railway). | Docker Container Resource Profiling | MASTER Sec 4 |
| **NFR-15** | **Scalability Capacity** | Architecture scales up to **10,000 active rooms**, **15,000 tenants**, and **500,000 annual payment transactions** without database schema restructuring. | Database Capacity Simulation | BR-05.1 |
| **NFR-16** | **Fault Tolerance & Resilience** | Database connection pool auto-reconnects with exponential backoff; Cloudflare R2 storage failures return structured `STORAGE_SERVICE_UNAVAILABLE` errors without backend process crash. | Chaos Testing & Service Interruption Mocking | BR-15.1 |

---

## 5. API Design & Universal Error Contracts

| NFR ID | Requirement | Metric / Constraint Target | Verification Method | Source Rule |
|---|---|---|---|---|
| **NFR-17** | **Uniform JSON Envelope** | 100% of API endpoints return standardized JSON envelopes (`statusCode`, `error`, `message`, `data`, `metadata`). Internal stack traces and raw SQL errors strictly stripped. | Global Exception Filter Unit Test | BR-15.1, BR-15.2 |
| **NFR-18** | **Rate Limiting & Anti-Abuse** | NestJS `ThrottlerModule` limits API requests: **100 req/min** per IP for standard routes; **5 req/min** per IP for authentication & password reset endpoints. See [`docs/ttl-registry.md`](ttl-registry.md) for auth token throttling rules. | Automated API Security Scan | BR-11, BR-12, docs/ttl-registry.md |

---

## 6. Maintainability & Code Quality

| NFR ID | Requirement | Metric / Constraint Target | Verification Method | Source Rule |
|---|---|---|---|---|
| **NFR-19** | **Test Coverage Target** | Minimum **85% line coverage** on backend services, controllers, financial calculation modules, and Zod validation pipes. | Jest Test Coverage Report | Workflow Rule |
| **NFR-20** | **Requirement Traceability** | 100% of backend services, API controllers, and frontend components cite explicit `FR-xx`, `NFR-xx`, and `AC-xx.y` identifiers in docstrings and commit messages. | Code Review & Audit Script | Workflow Rule |

---

## 7. Data Integrity & Seeding Performance

| NFR ID | Requirement | Metric / Constraint Target | Verification Method | Source Rule |
|---|---|---|---|---|
| **NFR-25** | **Country Code Integrity & Seeding Latency** | Database-level foreign key constraint (`ON DELETE RESTRICT`) guarantees 0 orphan phone records. Reusable seed execution from `data/country-codes.json` completes in $< 500\text{ ms}$ during DB deployment. Deleting, disabling, or updating in-use country codes guarantees strict HTTP 409 rejection. | Prisma Migration & Integration Test | BR-12.2, BR-12.3, FR-35 |

---

## 8. Traceability Map (NFR → Architectural Layer)

| NFR Range | Category | Target Architectural Layer |
|---|---|---|
| **NFR-01 – NFR-05** | Security & Auth | NestJS Guards, JwtModule, ZodPipes, Cloudflare R2 Adapter |
| **NFR-06 – NFR-09** | Immutability & Audit | PostgreSQL Triggers, Prisma Client, Financial Service Layer |
| **NFR-10 – NFR-13** | Performance & Latency | PostgreSQL B-Tree Indexes, Next.js PWA, Gzip Middleware |
| **NFR-14 – NFR-16** | Infrastructure & Scale | Docker Multi-Stage Build, Render/VPS Deployment, Prisma Pool |
| **NFR-17 – NFR-18** | Error Handling & Rates | Global NestJS Exception Filter, ThrottlerGuard |
| **NFR-19 – NFR-20** | Quality & Governance | Jest Unit/Integration Tests, Git Workflow |
| **NFR-25** | Data Integrity & Seeding | PostgreSQL FK Constraints (`ON DELETE RESTRICT`), Prisma Seed |

---

## Gate

**Stage 03 Confirmed & Locked ✅** (Approved by User on 2026-08-09)

**Next:** Stage 04 — Domain Model & State Machines → [`docs/04-domain-model.md`](04-domain-model.md)
