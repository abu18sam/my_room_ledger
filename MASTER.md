# 🚀 MASTER PRODUCT SPECIFICATION — ROOM & RENT LEDGER SYSTEM

**Project Name:** My Room Ledger / RentAway  
**Architecture Style:** Decoupled Security-First Monolith (Node.js/TypeScript **NestJS 10** Backend + Prisma ORM + PostgreSQL + Next.js TypeScript PWA Frontend + Cloudflare R2 Encrypted Storage)  
**Engineering Methodology:** Stage-Wise Guided Development ([docs/00-engineering-workflow.md](docs/00-engineering-workflow.md))  
**Authoritative Business Rules:** [docs/governance/business-rules.md](docs/governance/business-rules.md)

---

## 1. Product Overview & Vision

**My Room Ledger** is a landlord-centric multi-building rental management platform specialized for urban shared accommodations (PGs, co-living spaces, apartment blocks). It provides total financial governance to landlords over multi-building properties, flexible infrastructure (private & shared bathrooms), dynamic rent cycles, and strictly uncoupled financial ledgers (Room Rent vs. Electricity Bills), while offering tenants read-only financial transparency and integrated maintenance complaint reporting.

---

## 2. Core Business Rules Summary

> 📖 **Full Specification:** Detailed domain specifications, formulas, edge cases, and security constraints live in [docs/governance/business-rules.md](docs/governance/business-rules.md).

| # | Business Rule Name | Summary | Full Specs |
|---|---|---|---|
| **BR-01** | **Multi-Tier Role Hierarchy & Data Isolation** | 4-tier hierarchy (`SUPER_ADMIN` → `ADMIN` → `LANDLORD` → `TENANT`). Centralized permissions matrix in [`docs/governance/rbac-matrix.md`](docs/governance/rbac-matrix.md). Strict row-level data isolation — landlords see only owned properties; tenants see only own room. | [BR-01](docs/governance/business-rules.md#br-01-multi-tier-role-hierarchy--data-isolation-governance) \| [`docs/governance/rbac-matrix.md`](docs/governance/rbac-matrix.md) |
| **BR-02** | **Dynamic Non-Calendar Rent Cycles** | Rent cycles start on any day of the month ($1..31$). Independent per room and per utility. Fully specified in [`docs/domain/billing-and-reconciliation.md`](docs/domain/billing-and-reconciliation.md). | [BR-02](docs/governance/business-rules.md#br-02-dynamic--non-calendar-rent-cycles) \| [`docs/domain/billing-and-reconciliation.md`](docs/domain/billing-and-reconciliation.md) |
| **BR-03** | **Independent Ledgers, Reconciliation Engine & Payment Cutoff Rules** | Rent & Electricity are independent entities. 3-step reconciliation engine compares tenant collections vs supplier master bill paid (Surplus / Deficit / Break-even). Master cycle allocation strictly driven by `paymentDate` cutoff (`paymentDate <= masterCycleEnd`). Pass-through funds strictly excluded from Net Profit. Fully specified in [`docs/domain/billing-and-reconciliation.md`](docs/domain/billing-and-reconciliation.md). | [BR-03](docs/governance/business-rules.md#br-03-strict-independent-payment-separation--electricity-pass-through-model) \| [`docs/domain/billing-and-reconciliation.md`](docs/domain/billing-and-reconciliation.md) |
| **BR-04** | **Multi-Level Financial Aggregation** | 4-level reporting (Building → Portfolio → Per-Landlord → Platform). Aligned to Indian Financial Year (April 1 – March 31). | [BR-04](docs/governance/business-rules.md#br-04-multi-level-financial-aggregation--analytics) |
| **BR-05** | **Multi-Building Asset Hierarchy** | Scales seamlessly across $N$ buildings under a landlord account (`Landlord` → `Building` → `Floor` → `Room`/`Facilities`). | [BR-05](docs/governance/business-rules.md#br-05-multi-building--asset-hierarchy-scaling) |
| **BR-06** | **Flexible Infrastructure Layout** | Supports Private attached bathrooms and Shared floor bathrooms/toilets simultaneously on hybrid floors. | [BR-06](docs/governance/business-rules.md#br-06-flexible-infrastructure-layout) |
| **BR-07** | **Standardized Asset Numbering** | Floor-prefixed numbering scheme (`Room XY`, `Bath XY`, `Toilet XY` where $X$ = floor number). | [BR-07](docs/governance/business-rules.md#br-07-standardized-asset-numbering-scheme) |
| **BR-08** | **Tenancy History & Immutable Snapshots** | Billing cycle generation locks an immutable Tenancy Snapshot of active tenants. Historical bills never mutate on checkout. | [BR-08](docs/governance/business-rules.md#br-08-tenancy-history--billing-snapshots) |
| **BR-09** | **Complaint & Maintenance Lifecycle** | Categorized ticket logging (`PLUMBING`, `ELECTRICAL`, etc.) with severity and landlord resolution status workflow. | [BR-09](docs/governance/business-rules.md#br-09-complaint--maintenance-system) |
| **BR-10** | **Security-First Architecture, Sessions, Argon2id & UUIDv7 Strategy** | Universal **UUIDv7** primary keys (`uuidv7()` / time-ordered sequential B-Tree index locality) across all 24 entity tables. Zero auto-increment integers. Mandatory **Argon2id** password hashing (64MB RAM, 3 iterations, 4 parallelism). Mandatory Zod validation (`ZodValidationPipe`), AES-256 GCM encrypted R2 files, expiring signed URLs, multi-device session tracking. All token lifecycles governed by [`docs/governance/ttl-registry.md`](docs/governance/ttl-registry.md). | [BR-10](docs/governance/business-rules.md#br-10-security-first-architecture--session-management) \| [`docs/governance/ttl-registry.md`](docs/governance/ttl-registry.md) |
| **BR-11** | **Password Workflows & Recovery System** | Logged-in change (with UI masking toggle), Email Reset Link (15-min TTL), No-Email Temp Password Fallback (30-min TTL), global session revocation. All TTLs governed by [`docs/governance/ttl-registry.md`](docs/governance/ttl-registry.md). | [BR-11](docs/governance/business-rules.md#br-11-password-workflows--recovery-system) \| [`docs/governance/ttl-registry.md`](docs/governance/ttl-registry.md) |
| **BR-12** | **Dual Login & Dynamic Country Code Management System** | Email or Phone login (`+91` default). Central DB table (`country_codes`), seeded from base JSON (`data/country-codes.json`) with 7 countries. Admin CRUD controls (`/api/v1/admin/country-codes`). Strict DB referential integrity (`ON DELETE RESTRICT`) & in-use immutability (`HTTP 409 COUNTRY_CODE_IN_USE` on update/disable/delete if referenced). Zero FE hardcoding. | [BR-12](docs/governance/business-rules.md#br-12-dual-login--dynamic-country-code-management-architecture) |
| **BR-13** | **Power Supply Company Registry Governance & Connection Management** | Central pre-seeded DB registry (UPCL, UPPCL, Reliance, Adani, TPCL, NTPC). Mandatory building FK + active `connectionNumber`. Unpaid master bill block (`HTTP 409 PENDING_SUPPLIER_BILLS_EXIST`) on supplier switch. Audited supplier connections (`BuildingPowerConnection`). Deletion blocked if company in-use (`HTTP 409 COMPANY_IN_USE` with `affectedBuildings[]` metadata). | [BR-13](docs/governance/business-rules.md#br-13-power-supply-company-management--referential-integrity-governance) |
| **BR-14** | **Partial Payment Tracking, Balance Accumulation & Last-Transaction Update Rule** | Payment transaction sequence per ledger. Status machine: `UNPAID → PARTIALLY_PAID → PAID` or `OVERDUE`. Landlord can update latest transaction only (auto-recalculating ledger balance & status). Older transactions & Super Admin edits strictly locked (`HTTP 409 NON_LAST_TRANSACTION_UPDATE_RESTRICTED`). Supplier bill: single lump-sum settlement only. | [BR-14](docs/governance/business-rules.md#br-14-partial-payment-tracking--multi-cycle-balance-accumulation) |
| **BR-15** | **Backend Error Handling Standards & Universal Envelopes** | Universal JSON error envelope (`statusCode`, `error`, `message`, `metadata`). 21-code error registry with standardized HTTP statuses. Actionable user messages; internal stack traces and SQL queries strictly hidden. | [BR-15](docs/governance/business-rules.md#br-15-backend-error-handling-standards--uniform-response-envelopes) |
| **BR-16** | **Centralized Immutable Audit Logging & Session Control Governance** | Role-based force logout hierarchy (Super Admin > Admin > Landlord/Tenant). Mandatory active DB session check on every API call (`SessionValidationGuard`); session revocation immediately overrides JWT access token TTL (BR-16.5). Immutable, insert-only audit trail (`audit_logs`) tracking all state-changing actions. Indexed in [`docs/domain/audit-logging.md`](docs/domain/audit-logging.md). | [BR-16](docs/governance/business-rules.md#br-16-audit-logging--immutability-governance) \| [`docs/domain/audit-logging.md`](docs/domain/audit-logging.md) |
| **BR-17** | **Building, Floor, Room Occupancy Engine & Stacked Navigation Architecture** | Dynamic room/floor/building occupancy derived strictly from active tenant assignments (`Tenant.status = ACTIVE`). Historical move-outs isolated. Stacked visual navigation (`Building Details` → `Floor Stack` → `Floor Details` → `Room Blocks` → `Room Details` → `Tenant Cards`). Responsive UX across Mobile ($<640\text{px}$), Tablet ($640-1024\text{px}$), and Desktop ($>1024\text{px}$). Strict tenant isolation (`TenantRoomAccessGuard` / `HTTP 403`). Single source of truth in [`docs/domain/building-occupancy.md`](docs/domain/building-occupancy.md) (Domain rules) and [`docs/domain/frontend-navigation.md`](docs/domain/frontend-navigation.md) (UI & Navigation). | [BR-17](docs/governance/business-rules.md#br-17-building-floor-room-occupancy-engine--stacked-navigation-system-architecture) \| [`docs/domain/building-occupancy.md`](docs/domain/building-occupancy.md) \| [`docs/domain/frontend-navigation.md`](docs/domain/frontend-navigation.md) |
| **BR-18** | **Global File-Upload Policy & Security-First Storage Lifecycle Governance** | Enforces universal **5 MB file ceiling** across all upload features. Dual validation: frontend UX pre-check + backend gatekeeper (`HTTP 413 MAX_FILE_SIZE_EXCEEDED`). Direct-to-R2 presigned upload URLs (15-min TTL). Security-first cost optimization: WebP compression (~92% byte reduction), SHA-256 deduplication, R2 auto-tiering (Standard $\rightarrow$ Infrequent Access at 90d), and 30-day soft-deleted purge. Zero-compromise security (AES-256 GCM encryption at rest, RBAC, audit logs). Single source of truth in [`docs/domain/file-storage-and-upload-policy.md`](docs/domain/file-storage-and-upload-policy.md). | [BR-18](docs/governance/business-rules.md#br-18-global-file-upload-policy--security-first-storage-lifecycle-governance) \| [`docs/domain/file-storage-and-upload-policy.md`](docs/domain/file-storage-and-upload-policy.md) |
| **BR-19** | **Dual-Layer Validation Architecture & Defense-in-Depth Security Policy** | Enforces strict responsibility separation between Frontend and Backend. FE pre-validates inputs/forms client-side for immediate UX feedback and network efficiency (`BR-19.1`). BE independently performs 100% of all authentication, authorization/RBAC, schema validation (`ZodValidationPipe`), data ownership, and business rule checks for every request, acting as the sole authoritative security boundary (`BR-19.2`). Zero-trust client principle protects against direct API bypass (curl/Postman). Standardized JSON error envelopes with `fieldErrors[]` (`BR-19.5`). | [BR-19](docs/governance/business-rules.md#br-19-dual-layer-validation-architecture--defense-in-depth-security-policy) |
| **BR-20** | **Frontend API Communication, Server-State & Financial Idempotency Governance** | Standardizes Next.js $\rightarrow$ React $\rightarrow$ TanStack Query v5 $\rightarrow$ Orval Services $\rightarrow$ Axios Interceptor Pipeline. Central Axios client (`lib/api/api-client.ts`). Single-refresh queue mutex (`isRefreshing` flag, `failedQueue[]`) handles 10-min Access Token expiry without component awareness (`BR-20.2`). Immediate user ejection on `SESSION_REVOKED` / `ERR-1002` (`BR-20.3`). TanStack Query manages server caching (`staleTime: 5m`) and targeted query invalidation on mutation (`BR-20.4`). Financial mutations attach UUIDv7 `Idempotency-Key` headers (`BR-20.5`). | [BR-20](docs/governance/business-rules.md#br-20-frontend-api-communication-server-state--financial-idempotency-governance) |

---

## 3. System Scope

### In-Scope (Phase 1–6)
- Multi-building, floor, room, and shared facility CRUD operations.
- Flexible room configuration (occupancy, attached vs shared facilities).
- Tenant onboarding, KYC metadata, move-in/move-out history logs.
- Dynamic rent cycle configuration & history tracking.
- Independent ledger generation for Room Rent & Electricity Bills.
- **Electricity Reconciliation & Variance Monitoring** (Surplus / Deficit vs supplier master bill).
- **Security-First Cloudflare R2 Storage** with Backend-Side AES-256 Encryption & Expiring Signed URLs.
- **Installable Progressive Web App** (Next.js PWA + ShadCN UI + Tailwind CSS + Recharts).
- Manual payment status updates (UPI, Cash, Bank Transfer reference logging).
- Tenant complaint lifecycle management.
- **Indian Financial Year (April 1 – March 31)** Landlord Dashboard & P&L Revenue Tracking.
- Dual Login (Email or Phone + Country Code) with Zod validation.
- Admin-approved password reset workflows (Email reset link + No-email temp password fallback).

### Out-of-Scope (Future Enhancements)
- Automated online gateway payment processing (Razorpay/Stripe API integration).
- Automated SMS/WhatsApp OTP login gateways (simulated via phone/email auth in initial phase).

---

## 4. Key Specifications & Architectural References

| Document | Purpose |
|---|---|
| 📜 [docs/governance/business-rules.md](docs/governance/business-rules.md) | Complete business rules, logic formulas, edge cases & security constraints |
| 🏗️ [docs/technical/architecture.md](docs/technical/architecture.md) | System architecture, request lifecycle pipeline & data flow diagrams |
| 🗄️ [docs/technical/database-schema.md](docs/technical/database-schema.md) | Full PostgreSQL database schema (Prisma DDL) |
| 🔌 [docs/technical/api-contracts.md](docs/technical/api-contracts.md) | REST API endpoints, request/response contracts & validation rules |
| 🚀 [docs/technical/deployment-strategy.md](docs/technical/deployment-strategy.md) | Living deployment strategy ($0–$5/mo budget hosting & Docker multi-stage) |
| 📋 [docs/00-engineering-workflow.md](docs/00-engineering-workflow.md) | Stage tracker and development governance guidelines |

---

## 5. Stage Alignment & Governance

All engineering artifacts, schemas, APIs, and implementations strictly follow the stage-wise roadmap documented in [`docs/00-engineering-workflow.md`](docs/00-engineering-workflow.md):

- **Stage 01**: Requirement Analysis ([docs/stages/01-requirement-analysis.md](docs/stages/01-requirement-analysis.md)) — Confirmed ✅
- **Stage 02**: Functional Requirements & ACs ([docs/stages/02-functional-requirements.md](docs/stages/02-functional-requirements.md), [docs/stages/acceptance-criteria.md](docs/stages/acceptance-criteria.md)) — Confirmed ✅
- **Stage 03**: Non-Functional Requirements ([docs/stages/03-non-functional-requirements.md](docs/stages/03-non-functional-requirements.md)) — Confirmed ✅
- **Stage 04**: Domain Model & State Machines ([docs/stages/04-domain-model.md](docs/stages/04-domain-model.md)) — In Progress ⏳
- **Stage 05**: Database Design & DDL ([docs/stages/05-database-design.md](docs/stages/05-database-design.md))
- **Stage 06**: API Design & Contracts ([docs/stages/06-api-design.md](docs/stages/06-api-design.md))
- **Stage 07–18**: Frontend, Backend, Testing, and Production Readiness.
