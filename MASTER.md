# 🚀 MASTER PRODUCT SPECIFICATION — ROOM & RENT LEDGER SYSTEM

**Project Name:** My Room Ledger / RentAway  
**Architecture Style:** Decoupled Security-First Monolith (Node.js/TypeScript **NestJS** Backend + Prisma ORM + PostgreSQL + Next.js TypeScript PWA Frontend + Cloudflare R2 Encrypted Storage)  
**Engineering Methodology:** Stage-Wise Guided Development (`docs/00-engineering-workflow.md`)

---

## 1. Product Overview & Vision

**My Room Ledger** is a landlord-centric multi-building rental management platform specialized for urban shared accommodations (PGs, co-living spaces, apartment blocks). It provides total financial governance to landlords over multi-building properties, flexible infrastructure (private & shared bathrooms), dynamic rent cycles, and strictly uncoupled financial ledgers (Room Rent vs. Electricity Bills), while offering tenants read-only financial transparency and integrated maintenance complaint reporting.

---

## 2. Core Business Rules (Critical Requirements)

### Rule 1: Strict Multi-Tier Role Hierarchy & Security Governance
- **Super Admin** (Highest Authority):
  - Full system-level CRUD permissions across all entities (Buildings, Rooms, Tenants, Landlords, Billing, Complaints, Revenue).
  - Can Create, Update, and Delete `ADMIN` users.
  - *Creation Constraint*: Super Admins can ONLY be created by another Super Admin.
- **Admin** (Operational Management):
  - Operational authority to manage Landlords, Tenants, Buildings, Rooms, and Billing entries.
  - Can create and manage `LANDLORD` and `TENANT` accounts.
  - Access to platform-wide Revenue Dashboard and system metrics.
  - *Restriction*: ❌ Cannot create, modify, or delete Super Admin users.
- **Landlord** (Property Management):
  - Full control strictly over their own assigned properties, rooms, tenants, rent cycles, and billing.
  - Access to building-level Revenue & P&L Insights for their own properties.
  - CANNOT access or view other landlords' buildings or system-wide admin settings.
- **Tenant** (Occupant Access):
  - Strictly scoped read-only access for assigned room stay history and payment statuses (Rent + Electricity).
  - Can submit and track complaint tickets.
  - **Strict Security Constraint**: 🚫 Zero access to Revenue Dashboard, financial P&L, or admin settings.

### Rule 2: Dynamic & Non-Calendar Rent Cycles
- Rent cycles are flexible and can start on any day of the month (e.g., 5th to 4th, 15th to 14th).
- Landlords can modify cycle start dates mid-tenancy. Cycle history must be preserved.

### Rule 3: Strict Independent Payment Separation & Electricity Pass-Through Model (CRITICAL)
- **Room Rent** and **Electricity Bill** are **independent financial entities**.
- **Pass-Through Cost Model**: Electricity payments are **NOT revenue or profit**. All collections are ultimately paid to the external power supplier (e.g., UPCL).
- **Landlord Profit Formula**:
  $$\text{Net Profit} = (\text{Room Rent Collected}) - (\text{Building Operating Expenses})$$
  *(Electricity collections are excluded from net profit calculations).*
- **Dual Flow Payment Tracking**:
  - **Flow 1 (Tenant $\rightarrow$ Landlord)**: Room submeter collection. Formula: $\text{Total Bill} = \text{Units Consumed} \times \text{Rate per Unit}$. Status: `UNPAID`, `PARTIALLY_PAID`, `PAID`, `OVERDUE`.
  - **Flow 2 (Landlord $\rightarrow$ Power Supplier)**: Master building bill (e.g., UPCL). Stores supplier name, total master bill amount, payment status, and digital bill attachment (PDF/Image).
- **Electricity Reconciliation & Variance Audit (CRITICAL)**:
  - System tracks and compares **Tenant Electricity Collected** vs. **Power Supplier Master Bill Amount** for each billing cycle, monthly, and yearly (Indian Financial Year: April 1 – March 31).
  - **Reconciliation Formula**:
    $$\text{Electricity Variance} = (\text{Tenant Electricity Collected}) - (\text{Supplier Master Bill Amount})$$
  - **Surplus ($\text{Variance} > 0$)**: Extra funds collected from tenants over the master bill.
  - **Deficit / Shortfall ($\text{Variance} < 0$)**: Under-collected from tenants; landlord paying out-of-pocket.
  - **Balanced ($\text{Variance} = 0$)**: Perfect match between collections and supplier billing.
- **Invariant Rule**: Updating Room Rent status to `PAID` MUST NEVER alter Electricity Bill status, and vice versa.

### Rule 4: Multi-Level Financial Aggregation & Predictive Analytics (CRITICAL MODULE)
- **Multi-Level Aggregation Scope**:
  - **Level 1 (Single Building)**: Rent revenue, expenses, net profit, electricity pass-through & reconciliation audit (surplus/deficit), occupancy.
  - **Level 2 (Landlord Portfolio)**: Aggregated view across all buildings owned by a single landlord (portfolio rent, total operating expenses, portfolio net profit, portfolio electricity reconciliation surplus/deficit).
  - **Level 3 (Per-Landlord Admin View)**: Admin breakdown of individual landlord financial performance and electricity reconciliation state.
  - **Level 4 (System-Wide Platform View)**: Super Admin & Admin global platform metrics.
- **Building Expense Logging**: Category-wise tracking (water bills, maintenance, repairs, security, cleaning, property taxes, miscellaneous).
- **Time Window Filtering**: Monthly, Indian Financial Year (**April 1st to March 31st**), and Custom Date Ranges.
- **Predictive & Historical Analytics**:
  - **Historical Trends**: Multi-year rent revenue, expense trends, occupancy history, and electricity surplus/deficit history over time.
  - **Predictive Forecasting**: Statistical future revenue projections based on active tenancies and collection velocity.
  - **Risk Indicators**: Early warning alerts for mounting unpaid bills, declining occupancy rates, overdue spikes, and growing electricity deficits (out-of-pocket losses).
  - **Growth Suggestions**: Rent optimization insights, expense control recommendations, and submeter rate adjustments.

### Rule 5: Multi-Building & Asset Hierarchy Scaling
- Scales seamlessly from 2 buildings to $N$ buildings under a single landlord account.
- Hierarchy: `Platform` $\rightarrow$ `Landlord` $\rightarrow$ `Building` $\rightarrow$ `Floor` $\rightarrow$ `Room` / `Shared Bathroom` / `Shared Toilet`.

### Rule 6: Flexible Infrastructure Layout
- Rooms can feature **Private** attached bathrooms/toilets or utilize **Shared** floor-level facilities.
- **Hybrid Floors**: A single floor can host private-attached rooms alongside shared floor bathrooms simultaneously.

### Rule 7: Standardized Asset Numbering Scheme
- **Rooms**: `Room XY` where $X$ = floor number ($0$ for Ground) and $Y$ = room index on that floor (e.g., `Room 01`, `Room 11`, `Room 205`).
- **Shared Bathrooms**: `Bath XY` (e.g., `Bath 01`, `Bath 11`).
- **Shared Toilets**: `Toilet XY` (e.g., `Toilet 01`, `Toilet 11`).

### Rule 8: Tenancy History & Billing Snapshots
- Multiple tenants can reside in a single room.
- Full move-in and move-out audit trails are preserved.
- Generating a billing cycle locks an immutable **Tenancy Snapshot** of active tenants in that room during that cycle, ensuring historical bills never mutate when tenants change.

### Rule 9: Complaint & Maintenance System
- Tenants can log issues by category (Plumbing, Electrical, Cleanliness, Noise, Billing, Other) with description & severity.
- Landlords manage status (`OPEN` $\rightarrow$ `IN_PROGRESS` $\rightarrow$ `RESOLVED` / `REJECTED`) with resolution notes.

### Rule 10: Security-First Architecture, Strict Input Validation & Encrypted Cloudflare R2 Storage (MANDATORY)

#### 10.1 — Mandatory Zod Schema Validation on Every Input (STRICT)
- **Every single API endpoint input** — request body, query params, route params, and headers — MUST be validated through a **Zod schema** before reaching any business logic layer.
- **No raw unvalidated data** must ever enter a Controller, Service, or Prisma query.
- Validation is enforced globally via NestJS `ZodValidationPipe` registered at application bootstrap.
- All schemas are **defined in TypeScript** and co-located with their respective NestJS module DTOs (`dto/*.schema.ts`).
- **Validation failure response** must return `HTTP 400` with a structured error body:
  ```json
  {
    "statusCode": 400,
    "error": "VALIDATION_ERROR",
    "details": [{ "field": "email", "message": "Invalid email format" }]
  }
  ```

#### 10.2 — Strict Security Rules

**Session Management**
- **Multi-Device Login**: A single user may maintain **multiple concurrent sessions** across devices. Each login creates an independent `RefreshToken` database record scoped to that session.
- **Session Isolation**: Sessions are fully isolated — a token from Device A is not valid for Device B. Logout from one device terminates **only that session's** `RefreshToken` record. Other sessions remain unaffected.
- **Server-Side Session Store**: Refresh tokens are stored in the `RefreshToken` database table (not stateless). This enables precise revocation per-session and full logout-all capability.
- **JWT Security**: Access tokens expire in **15 minutes** (stored in-memory on client). Refresh tokens expire in **7 days** (stored in `HttpOnly; Secure; SameSite=Strict` cookies only).

**Password Management**
- **First-Login & Reset Mandatory Change**: Accounts created by Admins or reset via temporary password are flagged `isFirstLogin = true`. The user is **forced to a change-password screen** on login before accessing any feature. Any other API request while this flag is `true` returns `HTTP 403 MUST_CHANGE_PASSWORD`.
- **Admin-Controlled Reset Workflow (Tenants, Landlords, Admins)**:
  - Users **cannot self-reset** passwords without approval.
  - User raises a "Forgot / Reset Password" request via login screen or profile.
  - Request triggers a real-time notification to Admin / Super Admin in the **frontend header notification panel**.
  - Admin or Super Admin reviews and **approves** the request.
  - System auto-generates a secure **temporary password** and sets `isFirstLogin = true`.
  - System sends the temporary password to the user's registered email.
- **Edge Case (No Registered Email)**:
  - If a user (e.g. Tenant/Landlord onboarded via phone) has no registered email, Admin approval displays a **secure single-view temporary password modal** in the Admin portal (valid for 15 mins, recorded in audit logs) allowing Admin to securely share it via SMS or in-person. The user is still forced to change password upon login.
- **Global Cross-Device Session Invalidation (MANDATORY)**:
  - Upon password reset approval/completion, **ALL active sessions across ALL devices** for that user are immediately invalidated (`RefreshToken` records deleted from database).
  - User must log in again on every device using the temporary password.
- **Password Complexity**: Minimum 8 characters, at least 1 uppercase, 1 lowercase, 1 digit, 1 special character.
- **Password Reuse Prevention**: Users cannot reuse any of their last 3 passwords.
- **Password Storage**: All passwords stored as **bcrypt hashes** (minimum 12 salt rounds). Plaintext passwords are never stored, logged, or transmitted.

**File & API Security**
- **Backend File Encryption**: All sensitive files (Government IDs, payment receipts, supplier bills) MUST be encrypted at the backend level (AES-256 GCM) BEFORE being uploaded to Cloudflare R2 object storage.
- **Zero Public URL Access**: Objects in Cloudflare R2 are strictly private. Direct client storage URLs are strictly prohibited.
- **Expiring Signed URLs**: File access is authorized ONLY via backend-generated short-lived expiring signed URLs following strict RBAC validation.
- **Rate Limiting**: All endpoints protected via `@nestjs/throttler` (100 requests / 15 minutes per IP). Auth endpoints have stricter limits (5 attempts / 15 minutes).
- **HTTP Security Headers**: `Helmet.js` applied globally at bootstrap — enforces `Content-Security-Policy`, `X-Frame-Options`, `X-Content-Type-Options`, `Strict-Transport-Security`.
- **CORS Lockdown**: Only the `NEXT_PUBLIC_FRONTEND_URL` origin is whitelisted. All other origins are rejected.
- **Data Protection & Input Sanitization**: No raw unsanitized user input ever reaches business logic. Sanitization is enforced via Zod `.trim()` + `.min()` + regex pattern matching on string fields.
#### 10.3 — Dual Login Methods & Dynamic Country Code Validation Architecture (MANDATORY)

**Supported Login Credentials**
1. **Email + Password**: Email format validated against RFC 5322 regex standard.
2. **Phone Number + Password**: Requires mandatory `Country Code` dropdown selection (default `+91 🇮🇳 India`) + local phone number input + password.

**Validation Architecture & Source of Truth**
- **Backend Source of Truth**: The Backend (NestJS + Zod) is the ultimate security boundary. All authentication inputs are validated at request entry via `ZodValidationPipe`. No client-side validation is trusted implicitly.
- **Frontend UX Validation**: The Frontend executes matching Zod client-side validation for instant user feedback (preventing unnecessary network roundtrips).

**DB-Managed Country Code & Regex System**
- **Zero Frontend Hardcoding**: Frontend MUST NOT hardcode or store phone regex patterns or country lists.
- **Database Table (`country_codes`)**: All supported country codes, dial codes (e.g. `+91`), country names, flag emojis (e.g. `🇮🇳`), and regex validation patterns (e.g. `^[6-9]\d{9}$`) are stored and managed in the backend database.
- **Dynamic Metadata Fetching**: On application load/login render, the frontend fetches active country codes via `GET /api/v1/meta/country-codes` (cached client-side).
- **Mandatory Default**: `+91` (India) is set as the default selection and cannot be unselected (must always have a valid country code picked).
- **E.164 Phone Normalization**: Backend normalizes all phone numbers to E.164 format (`+<dialCode><number>`) before database querying and persistence.



---

## 3. System Scope

### In-Scope (Phase 1–6)
- Multi-building, floor, and room CRUD operations.
- Flexible room configuration (occupancy, attached vs shared facilities).
- Tenant onboarding, KYC metadata, move-in/move-out history logs.
- Dynamic rent cycle configuration & history tracking.
- Independent ledger generation for Room Rent & Electricity Bills.
- **Security-First Cloudflare R2 Storage with Backend-Side AES-256 Encryption & Expiring Signed URLs.**
- **Installable Progressive Web App (Next.js PWA + ShadCN UI + Tailwind CSS).**
- Manual payment status updates (UPI, Cash, Bank Transfer reference logging).
- Tenant complaint lifecycle management.
- **Indian Financial Year (April 1 – March 31) Landlord Dashboard & P&L Revenue Tracking.**
- Landlord & Tenant dashboards with JWT authentication.

### Out-of-Scope (Future Enhancements)
- Automated online gateway payment processing (Razorpay/Stripe API integration).
- Automated SMS/WhatsApp OTP login gateways (simulated via phone/email auth in initial phase).

---

## 4. Multi-Tier RBAC Permission Matrix

| System Module / Action | Super Admin | Admin | Landlord | Tenant |
| :--- | :---: | :---: | :---: | :---: |
| Create / Manage Admin Users & Assign Roles | ✅ Full | ❌ Blocked | ❌ Blocked | ❌ Blocked |
| Register / Onboard New Landlords | ✅ Full | ✅ Allowed | ❌ Blocked | ❌ Blocked |
| Update Landlord Profile Details | ✅ Full | ✅ Allowed | ❌ Blocked | ❌ Blocked |
| View System-Wide Analytics (Landlords count, Total Buildings) | ✅ Full | ✅ Allowed | ❌ Blocked | ❌ Blocked |
| Create / Manage Own Buildings & Floor/Room Structure | 🔍 System Audit | 🔍 System Audit | ✅ Own Buildings | ❌ Blocked |
| View Yearly IFY Revenue & P&L Dashboard | 🔍 System Audit | 🔍 System Audit | ✅ Own Buildings Only | 🚫 STRICTLY BLOCKED |
| Manage Rent Cycles & Generate Billing Ledgers | 🔍 System Audit | 🔍 System Audit | ✅ Own Buildings Only | ❌ Blocked |
| Update Room Rent & Electricity Ledger Status | 🔍 System Audit | 🔍 System Audit | ✅ Own Buildings Only | ❌ Blocked |
| View Assigned Room Payment History (Rent + Elec) | 🔍 System Audit | 🔍 System Audit | ✅ Own Buildings Only | 🔍 Own Assigned Room Only |
| Raise / Track Complaint Tickets | 🔍 Audit | 🔍 Audit | 🔍 Audit / Resolve | ✅ Log & Track Own Complaints |

---

## 5. Primary Entities Overview

```
+---------------+      (1:N)     +---------------+      (1:N)     +---------------+
|     USERS     +--------------->|   BUILDINGS   +--------------->|    FLOORS     |
| (Landlord/Ten)|                +---------------+                +-------+-------+
+---------------+                                                         |
                                 +----------------------------------------+
                                 |
                                 +---------> (1:N) Rooms
                                 +---------> (1:N) Shared Bathrooms
                                 +---------> (1:N) Shared Toilets

+---------------+      (1:N)     +---------------+      (1:1)     +-----------------------+
|     ROOMS     +--------------->|BILLING_CYCLES +--------------->|   ROOM_RENT_LEDGERS   |
+-------+-------+                +-------+-------+                +-----------------------+
        |                                |
        | (1:N)                          +----------------------->|  ELECTRICITY_LEDGERS  |
        v                                               (1:1)     +-----------------------+
+---------------+
|    TENANTS    |
+---------------+
```

---

## 5. Stage Alignment & Governance

All engineering artifacts, schemas, APIs, and implementations strictly follow the stage-wise roadmap documented in [`docs/00-engineering-workflow.md`](docs/00-engineering-workflow.md):

- **Stage 01**: Requirement Analysis (`docs/01-requirement-analysis.md`)
- **Stage 02**: Functional Requirements & ACs (`docs/02-functional-requirements.md`, `docs/acceptance-criteria.md`)
- **Stage 03**: Non-Functional Requirements (`docs/03-non-functional-requirements.md`)
- **Stage 04**: Domain Model & State Machines (`docs/04-domain-model.md`)
- **Stage 05**: Database Design & DDL (`docs/05-database-design.md`)
- **Stage 06**: API Design & Contracts (`docs/06-api-design.md`)
- **Stage 07–18**: Frontend, Backend, Testing, and Production Readiness.
