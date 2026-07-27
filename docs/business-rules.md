# 📜 CORE BUSINESS RULES & DOMAIN SPECIFICATION

**System:** My Room Ledger / RentAway  
**Status:** Confirmed & Active  
**Upstream:** [MASTER.md](../MASTER.md)  
**Downstream:** [01-requirement-analysis.md](01-requirement-analysis.md), [02-functional-requirements.md](02-functional-requirements.md), [acceptance-criteria.md](acceptance-criteria.md)  
**Workflow Tracker:** [00-engineering-workflow.md](00-engineering-workflow.md)

---

## 1. Purpose

This document serves as the **single authoritative source of truth for all domain rules, business logic, security constraints, and workflow invariants** governing the My Room Ledger platform. All functional requirements, API contracts, database schemas, and frontend UI components must conform to the rules specified herein.

---

## 2. Business Rules Catalog

### BR-01: Multi-Tier Role Hierarchy & Data Isolation Governance

#### BR-01.1 — Role Scope Definitions
- **SUPER_ADMIN** (Highest System Authority):
  - Full system-level CRUD access across all entities (Buildings, Rooms, Tenants, Landlords, Billing, Operating Expenses, Complaints, Global Analytics, Audit Logs).
  - Can Create, Update, Deactivate, and Delete `ADMIN` and `LANDLORD` accounts.
  - Can manage global system configurations and view system-wide logs.
  - *Creation Constraint*: A `SUPER_ADMIN` user can **ONLY** be created by another existing `SUPER_ADMIN`.
- **ADMIN** (Operational Management):
  - Administrative authority to onboard, update, and manage `LANDLORD` and `TENANT` accounts.
  - Access to platform-wide Revenue Dashboard, aggregated landlord performance metrics (Level 3), and system-wide reports.
  - *Restriction*: ❌ Cannot create, edit, deactivate, or delete `SUPER_ADMIN` accounts; cannot override Super Admin system configuration controls.
- **LANDLORD** (Property Management):
  - Access strictly gated to **their own assigned properties only**.
  - Can manage rooms, floors, tenant check-ins/check-outs, rent cycles, room submeter readings, and billing ledgers for their owned properties.
  - Access to revenue, P&L, expense logging, and electricity reconciliation dashboards for **their own properties only**.
  - *Restriction*: ❌ Cannot view or access other landlords' buildings, rooms, ledgers, or global admin settings under any condition.
- **TENANT** (Occupant Access):
  - Strictly limited to **own self-access data only**.
  - Can view assigned room details, tenancy history, and own payment records (Rent + Electricity).
  - Can submit and track status of personal maintenance complaints.
  - **Strict Security Constraint**: 🚫 ZERO access to revenue dashboards, P&L analytics, building operating expenses, or other tenants' data.

#### BR-01.2 — Technical Authorization & Data Isolation Rules
- **Backend as Source of Truth**: Authorization is enforced on the backend via NestJS `@Roles(...)` decorator + `JwtAuthGuard` + `RolesGuard`.
- **Row-Level Data Isolation**: Database queries enforce row-level tenant/landlord ownership filtering via Prisma ORM (`where: { landlordId: req.user.id }` for Landlords; `where: { userId: req.user.id }` for Tenants).
- **Frontend UI Permission Rendering**: Frontend UI components, navigation menus, and buttons dynamically render based on JWT role claims for clean UX, but the frontend NEVER acts as a security boundary.
- **Cross-Entity Lockdown**: Cross-role or cross-tenant data leakage is strictly prevented. Any unauthorized access attempt returns `HTTP 403 FORBIDDEN`.

---

### BR-02: Dynamic & Non-Calendar Rent Cycles
- Rent cycles are flexible and can start on any day of the month (e.g., 5th to 4th, 15th to 14th).
- Landlords can modify cycle start dates mid-tenancy. Cycle history must be preserved.

---

### BR-03: Strict Independent Payment Separation & Electricity Pass-Through Model

#### BR-03.1 — Payment Isolation & Net Profit Formula
- **Room Rent** and **Electricity Bill** are **independent financial entities**.
- **Pass-Through Cost Model**: Electricity payments are **NOT revenue or profit**. All collections are ultimately paid to the external power supplier (e.g., UPCL).
- **Landlord Profit Formula**:
  $$\text{Net Profit} = (\text{Room Rent Collected}) - (\text{Building Operating Expenses})$$
  *(Electricity collections are excluded from net profit calculations).*

#### BR-03.2 — Dual-Flow Payment Tracking
- **Flow 1 (Tenant $\rightarrow$ Landlord)**: Room submeter collection. Formula: $\text{Total Bill} = \text{Units Consumed} \times \text{Rate per Unit}$. Status: `UNPAID`, `PARTIALLY_PAID`, `PAID`, `OVERDUE`.
- **Flow 2 (Landlord $\rightarrow$ Power Supplier)**: Master building bill (e.g., UPCL). Stores supplier name, total master bill amount, payment status, and digital bill attachment (PDF/Image).

#### BR-03.3 — Electricity Reconciliation & Variance Audit
- System tracks and compares **Tenant Electricity Collected** vs. **Power Supplier Master Bill Amount** for each billing cycle, monthly, and yearly (Indian Financial Year: April 1 – March 31).
- **Reconciliation Formula**:
  $$\text{Electricity Variance} = (\text{Tenant Electricity Collected}) - (\text{Supplier Master Bill Amount})$$
- **Surplus ($\text{Variance} > 0$)**: Extra funds collected from tenants over the master bill.
- **Deficit / Shortfall ($\text{Variance} < 0$)**: Under-collected from tenants; landlord paying out-of-pocket.
- **Balanced ($\text{Variance} = 0$)**: Perfect match between collections and supplier billing.
- **Invariant Rule**: Updating Room Rent status to `PAID` MUST NEVER alter Electricity Bill status, and vice versa.

---

### BR-04: Multi-Level Financial Aggregation & Analytics

#### BR-04.1 — Multi-Level Aggregation Scope
- **Level 1 (Single Building)**: Rent revenue, expenses, net profit, electricity pass-through & reconciliation audit (surplus/deficit), occupancy.
- **Level 2 (Landlord Portfolio)**: Aggregated view across all buildings owned by a single landlord (portfolio rent, total operating expenses, portfolio net profit, portfolio electricity reconciliation surplus/deficit).
- **Level 3 (Per-Landlord Admin View)**: Admin breakdown of individual landlord financial performance and electricity reconciliation state.
- **Level 4 (System-Wide Platform View)**: Super Admin & Admin global platform metrics.

#### BR-04.2 — Building Expense Logging & Reporting
- Category-wise expense tracking: `WATER_BILL`, `MAINTENANCE`, `REPAIRS`, `SECURITY`, `CLEANING`, `PROPERTY_TAX`, `MISCELLANEOUS`.
- Time Window Filtering: **Monthly**, **Indian Financial Year (April 1st to March 31st)**, and **Custom Date Ranges**.
- Predictive & Historical Analytics: Multi-year trends, revenue forecasting, risk indicators (mounting unpaid dues, declining occupancy, overdue spikes, growing electricity deficits).

---

### BR-05: Multi-Building & Asset Hierarchy Scaling
- Scales seamlessly from 2 buildings to $N$ buildings under a single landlord account.
- Hierarchy: `Platform` $\rightarrow$ `Landlord` $\rightarrow$ `Building` $\rightarrow$ `Floor` $\rightarrow$ `Room` / `Shared Bathroom` / `Shared Toilet`.

---

### BR-06: Flexible Infrastructure Layout
- Rooms can feature **Private** attached bathrooms/toilets or utilize **Shared** floor-level facilities.
- **Hybrid Floors**: A single floor can host private-attached rooms alongside shared floor bathrooms simultaneously.

---

### BR-07: Standardized Asset Numbering Scheme
- **Rooms**: `Room XY` where $X$ = floor number ($0$ for Ground) and $Y$ = room index on that floor (e.g., `Room 01`, `Room 11`, `Room 205`).
- **Shared Bathrooms**: `Bath XY` (e.g., `Bath 01`, `Bath 11`).
- **Shared Toilets**: `Toilet XY` (e.g., `Toilet 01`, `Toilet 11`).

---

### BR-08: Tenancy History & Billing Snapshots
- Multiple tenants can reside in a single room.
- Full move-in and move-out audit trails are preserved.
- Generating a billing cycle locks an immutable **Tenancy Snapshot** of active tenants in that room during that cycle, ensuring historical bills never mutate when tenants change.

---

### BR-09: Complaint & Maintenance System
- Tenants log issues by category (`PLUMBING`, `ELECTRICAL`, `CLEANLINESS`, `NOISE`, `BILLING`, `OTHER`) with description & severity (`LOW`, `MEDIUM`, `HIGH`, `URGENT`).
- Landlords manage status (`OPEN` $\rightarrow$ `IN_PROGRESS` $\rightarrow$ `RESOLVED` / `REJECTED`) with resolution notes.

---

### BR-10: Security-First Architecture & Session Management

#### BR-10.1 — Mandatory Zod Input Validation
- **Every API endpoint input** (body, query, params, headers) MUST be validated through a **Zod schema** via global `ZodValidationPipe`. No raw unvalidated data enters Controllers, Services, or Prisma.
- Returns `HTTP 400 VALIDATION_ERROR` with structured field-level error details.

#### BR-10.2 — Session & Storage Security
- **Multi-Device Login**: Users can log in from multiple devices simultaneously. Each session maintains an independent `RefreshToken` record in DB.
- **Session Isolation & Logout**: Sessions are isolated — logout terminates only the specific device's session.
- **JWT Security**: Access tokens expire in 15 mins (in-memory). Refresh tokens expire in 7 days (`HttpOnly; Secure; SameSite=Strict` cookie).
- **Backend File Encryption**: Sensitive files (Govt IDs, receipts, supplier bills) encrypted AES-256 GCM before storage in private Cloudflare R2 bucket.
- **Zero Public File Access**: File access strictly via short-lived backend-generated expiring signed URLs (15-min expiry).
- **Rate Limiting & Security Headers**: Helmet.js enabled globally. `@nestjs/throttler` (100 req/15min globally; 5 req/15min auth). CORS locked down to `NEXT_PUBLIC_FRONTEND_URL`.

---

### BR-11: Password Workflows & Recovery System

#### BR-11.1 — Logged-in User Change Password
- Accessible from profile/settings or login screen.
- Inputs required: **Old Password**, **New Password**, **Confirm New Password**.
- UI Requirement: All password inputs feature **password masking with toggle (show/hide eye icon)**.

#### BR-11.2 — Forgot Password Workflow (With Registered Email)
- User submits request from login page $\rightarrow$ Real-time notification in Admin header bell panel.
- On Admin approval, system emails a signed reset link (token valid for **15 minutes**).
- Clicking link redirects to `/reset-password?token=...` displaying User Name & Email. User enters New Password + Confirm New Password.
- On success: password updated, link token expires immediately (single-use), **ALL active sessions on all devices invalidated**, user redirected to Login page.

#### BR-11.3 — Fallback Workflow (No Registered Email — Admin, Landlord & Tenant)
- User raises reset request $\rightarrow$ Admin approves and generates a secure **temporary password (valid for 30 minutes)** displayed in Admin single-view modal.
- Admin shares temporary password securely (in-person/SMS).
- User logs in with temporary password $\rightarrow$ Forced redirection to **"Create New Password" page** (`mustChangePassword = true`).
- Displays User Name (and Email if available). User enters New Password + Confirm New Password.
- On success: password updated, `mustChangePassword` reset to `false`, **ALL active sessions on all devices invalidated**, user redirected to Login page.

#### BR-11.4 — Security & Complexity Constraints
- Password Complexity: Minimum 8 characters, 1 uppercase, 1 lowercase, 1 digit, 1 special character.
- Password Reuse Prevention: Users cannot reuse any of their last 3 passwords.
- Passwords hashed with `bcrypt` (minimum 12 salt rounds). Plaintext passwords never stored or logged.

---

### BR-12: Dual Login & Dynamic Country Code Validation Architecture

#### BR-12.1 — Supported Credentials
- **Email + Password**: Email format validated against RFC 5322 regex standard.
- **Phone Number + Password**: Requires mandatory `Country Code` dropdown selection (default `+91 🇮🇳 India`) + local phone number input + password.

#### BR-12.2 — Validation & Metadata System
- **Backend Source of Truth**: NestJS + Zod (`ZodValidationPipe`) is the ultimate security boundary. Phone numbers normalized to E.164 (`+919876543210`).
- **Frontend UX Validation**: Frontend runs matching Zod client validation for instant visual feedback.
- **DB-Managed Country Codes**: All country codes, dial codes (`+91`), flags (`🇮🇳`), and regex patterns (`^[6-9]\d{9}$`) stored in `country_codes` database table and served via `GET /api/v1/meta/country-codes`.
- **Zero Frontend Hardcoding**: Frontend MUST NOT hardcode regex patterns or country lists.
