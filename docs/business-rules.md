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

#### BR-01.1 — Role Scope Definitions & Centralized RBAC Matrix
> **Centralized Authority**: Full CRUD permissions per entity across all system roles are formally defined in the centralized [`docs/rbac-matrix.md`](rbac-matrix.md) document.

- **SUPER_ADMIN** (Highest System Authority):
  - Full system-level CRUD access across all entities (Buildings, Rooms, Tenants, Landlords, Billing, Operating Expenses, Complaints, Global Analytics, Audit Logs).
  - Can Create, Update, Deactivate, and Delete `ADMIN` and `LANDLORD` accounts.
  - Can manage global system configurations and view system-wide logs.
  - *Creation Constraint*: A `SUPER_ADMIN` user can **ONLY** be created by another existing `SUPER_ADMIN`.
  - *Operational Restrictions*: ❌ Cannot edit or delete `AuditLog` records, `TenancyHistory` logs, or `PaymentTransaction` records. View-only access for financial transaction logs.
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

#### BR-01.5 — Hierarchical Session Force-Logout Invariant
- **`SUPER_ADMIN`**: Can forcefully terminate all active sessions of `ADMIN`, `LANDLORD`, and `TENANT` users.
- **`ADMIN`**: Can forcefully terminate all active sessions of `LANDLORD` and `TENANT` users.
- **Role Hierarchy Enforcement**: An `ADMIN` user is **strictly blocked** from forcefully terminating sessions of a `SUPER_ADMIN` or another `ADMIN` account. Any attempt is rejected with `HTTP 403 FORBIDDEN` (error code: `ROLE_HIERARCHY_VIOLATION`).
- **Landlords & Tenants**: Can only view and terminate their own individual device sessions (`UserSession` records).
- **Cross-Entity Lockdown**: Cross-role or cross-tenant data leakage is strictly prevented. Any unauthorized access attempt returns `HTTP 403 FORBIDDEN`.

---

### BR-02: Dynamic & Non-Calendar Rent Cycles
> **Single Source of Truth**: Detailed mathematical calculations, overlap rules, and edge cases for flexible non-calendar billing cycles are indexed in [`docs/billing-and-reconciliation.md`](billing-and-reconciliation.md).

- Rent cycles are flexible and can start on any day of the month (e.g., 5th to 4th, 15th to 14th).
- Each room within a building can operate on an independent billing cycle schedule.
- Room Rent and Electricity cycles within the same room can have different start/end dates.
- Landlords can modify cycle start dates mid-tenancy. Cycle history must be preserved.

---

### BR-03: Strict Independent Payment Separation & Electricity Pass-Through Model

#### BR-03.1 — Payment Isolation & Net Profit Formula
- **Room Rent** and **Electricity Bill** are **independent financial entities**.
- **Pass-Through Cost Model**: Electricity payments collected from tenants are **NOT revenue or profit**. All tenant electricity collections are held in a pass-through ledger intended to offset external power supplier master bills (e.g., UPCL, UPPCL).
- **Landlord Net Profit Invariant**:
  $$\text{Net Profit} = (\text{Room Rent Collected}) - (\text{Building Operating Expenses})$$
  *(Electricity collections, electricity surplus, and electricity deficits are strictly excluded from net rental profit calculations).*

#### BR-03.2 — Per-Unit Rate Differential & Common Electricity Load
- **Per-Unit Rate Differential**: Landlords charge tenants a per-unit electricity rate (e.g., ₹8/unit on room submeters) that may differ from the utility company's master tariff rate (e.g., ₹7/unit).
- **Common Area Electricity Usage**:
  - Unmetered or common building electricity consumption (e.g., hallway/stairwell lighting, common area sanitation lighting, submersible water motors/pumps) is **NOT billed to individual tenants**.
  - Common electricity costs are paid by the landlord from:
    1. **Electricity Collection Surplus** (if collections exceed supplier master bill), OR
    2. **Landlord Rental Income** (if collections result in a deficit), logged as a `WATER_MOTOR_ELECTRICITY` or `COMMON_ELECTRICITY` Building Operating Expense.

#### BR-03.3 — Dynamic Billing Period & 3-Scenario Reconciliation
> **Full Engine Specification**: See [`docs/billing-and-reconciliation.md`](billing-and-reconciliation.md) for step-by-step algorithms, pro-rata formulas, and concrete numerical examples.

- **Dynamic Utility Billing Cycle**: Billing periods for electricity pass-through are dynamic, following the power supply company's bill cycle dates (not restricted to calendar months).
- **3-Scenario Reconciliation Ledger**:
  $$\text{Electricity Variance} = (\text{Tenant Electricity Collected}) - (\text{Supplier Master Bill Amount})$$
  1. **Deficit Case (Loss / Landlord Contribution, $\text{Variance} < 0$)**:
     - Occurs when tenant submeter collections are lower than the master bill (e.g., Collections ₹4,000 vs. Utility Bill ₹5,000 $\rightarrow$ Deficit ₹1,000).
     - Deficit is paid by the landlord and recorded as a landlord out-of-pocket contribution.
     - **Strict Invariant**: Tracked separately and **NEVER merged into Net Rental Profit**.
  2. **Surplus Case (Extra Savings, $\text{Variance} > 0$)**:
     - Occurs when tenant submeter collections exceed the master bill (e.g., Collections ₹4,500 vs. Utility Bill ₹3,500 $\rightarrow$ Surplus ₹1,000).
     - Recorded as "Extra Savings" for the building/landlord.
     - **Strict Invariant**: Tracked separately and **NEVER added to Net Rental Profit**.
  3. **Break-Even Case ($\text{Variance} = 0$)**:
     - Tenant collections exactly match the master bill. No surplus or deficit recorded.

#### BR-03.4 — Strict Separation Invariants
- Updating Room Rent status to `PAID` MUST NEVER alter Electricity Bill status, and vice versa.
- All dashboards, reports, and API responses MUST present Rental Income, Electricity Surplus, and Electricity Deficit as distinct, un-merged financial metrics.

#### BR-03.5 — Multi-Cycle Overlap & Payment Date Cutoff Allocation Rule
- Landlord electricity reconciliation is computed dynamically upon settling a `SupplierMasterBill`.
- **Payment Date-Based Allocation**: Allocation of tenant electricity collections to a building master billing cycle is strictly driven by the **actual payment date (`PaymentTransaction.paymentDate`)**, NOT by the room billing cycle start/end dates.
- **Allocation Rule**:
  - If `paymentDate <= masterCycleEndDate`: The payment is allocated to the **Current Master Cycle**.
  - If `paymentDate > masterCycleEndDate`: The payment is excluded from the previous cycle and allocated to the **Next Master Cycle** (or master cycle window containing `paymentDate`).
- **Partial Payment Allocation**: Multiple payments against a single room ledger are evaluated independently based on each transaction's own `paymentDate`.
- Unpaid or overdue tenant ledgers contribute $₹0.00$ to tenant collections for a cycle window until actual cash payments are recorded. See [`docs/billing-and-reconciliation.md`](billing-and-reconciliation.md#5-tenant-electricity-payment-cutoff--master-cycle-allocation-rules) for full specifications.

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
> **Single Source of Truth**: All token lifecycles, expiration rules, and TTL governance are specified in [`docs/ttl-registry.md`](ttl-registry.md).

- **Multi-Device Login**: Users can log in from multiple devices simultaneously. Each session maintains an independent `RefreshToken` record in DB.
- **Session Isolation & Logout**: Sessions are isolated — logout terminates only the specific device's session.
- **JWT Security**: Access tokens expire in **10 minutes** (in-memory). Refresh tokens expire in 7 days (`HttpOnly; Secure; SameSite=Strict` cookie). All TTLs governed by [`docs/ttl-registry.md`](ttl-registry.md).
- **Backend File Encryption**: Sensitive files (Govt IDs, receipts, supplier bills) encrypted AES-256 GCM before storage in private Cloudflare R2 bucket.
- **Zero Public File Access**: File access strictly via short-lived backend-generated expiring signed URLs (15-min expiry as defined in [`docs/ttl-registry.md`](ttl-registry.md)).
- **Rate Limiting & Security Headers**: Helmet.js enabled globally. `@nestjs/throttler` (100 req/min globally; 5 req/min auth). CORS locked down to `NEXT_PUBLIC_FRONTEND_URL`.

#### BR-10.4 — Universal UUIDv7 Primary Key Strategy
- All primary keys across all database tables MUST be generated using **UUIDv7** (time-ordered 128-bit globally unique identifiers via `uuidv7()` in PostgreSQL / Prisma ORM).
- **Time-Ordered Index Locality**: UUIDv7 embeds a 48-bit millisecond timestamp in high-order bits, providing sequential monotonicity. This eliminates B-Tree index page splits, reduces fragmentation, and optimizes high-throughput PostgreSQL INSERT performance while preserving global uniqueness.
- Sequential auto-increment integer primary keys (`BIGINT AUTOINCREMENT`) are **strictly prohibited** across all models.
- All foreign key relationships, route parameters (`z.string().uuid()`), API payloads, and NestJS pipes (`ParseUUIDPipe`) strictly enforce 36-character hyphenated UUIDv7 strings (`8-4-4-4-12` format).

---

### BR-11: Password Workflows & Recovery System

#### BR-11.1 — Logged-in User Change Password
- Accessible from profile/settings or login screen.
- Inputs required: **Old Password**, **New Password**, **Confirm New Password**.
- UI Requirement: All password inputs feature **password masking with toggle (show/hide eye icon)**.

#### BR-11.2 — Forgot Password Workflow (With Registered Email)
- User submits request from login page $\rightarrow$ Real-time notification in Admin header bell panel.
- On Admin approval, system emails a signed reset link (token valid for **15 minutes** as specified in [`docs/ttl-registry.md`](ttl-registry.md)).
- Single-use token: Invalidated immediately upon password update (`usedAt = now()`).

#### BR-11.3 — Fallback Workflow (No Registered Email — Admin, Landlord & Tenant)
- User raises reset request $\rightarrow$ Admin approves and generates a secure **temporary password (valid for 30 minutes)** displayed in Admin single-view modal (governed by [`docs/ttl-registry.md`](ttl-registry.md)).
- Admin shares temporary password securely (in-person/SMS).
- User logs in with temporary password $\rightarrow$ Forced redirection to **"Create New Password" page** (`mustChangePassword = true`).
- Displays User Name (and Email if available). User enters New Password + Confirm New Password.
- On success: password updated, `mustChangePassword` reset to `false`, **ALL active sessions on all devices invalidated**, user redirected to Login page.

#### BR-11.4 — Security & Complexity Constraints
- Password Complexity: Minimum 8 characters, 1 uppercase, 1 lowercase, 1 digit, 1 special character.
- Password Reuse Prevention: Users cannot reuse any of their last 3 passwords.
- **Mandatory Argon2id Hashing**: Passwords MUST be hashed exclusively using **Argon2id** (memory cost: 64 MB / $65,536\text{ KiB}$, time cost: 3 iterations, parallelism: 4 threads, unique per-user salt). Plaintext passwords are NEVER stored, logged, or transmitted. Encoded hashes start with `$argon2id$`. Automatic transparent rehashing supported on login upon parameter upgrades.

---

### BR-12: Dual Login & Dynamic Country Code Management Architecture

#### BR-12.1 — Supported Credentials
- **Email + Password**: Email format validated against RFC 5322 regex standard.
- **Phone Number + Password**: Requires mandatory `Country Code` dropdown selection (default `+91 🇮🇳 India`) + local phone number input + password.

#### BR-12.2 — Centralized Database Registry & Reusable Base JSON Seeding
- **Database Table**: Dedicated, independent `country_codes` database table storing ISO alpha-2 codes (`IN`, `US`, `CA`, `GB`, `AE`, `NP`, `LK`), dial codes (`+91`), flag emojis (`🇮🇳`), phone validation regex patterns (`^[6-9]\d{9}$`), min/max lengths, default flag (`isDefault`), and active status (`isActive`).
- **Reusable Base JSON Seed**: Database seeded during initial setup via a standalone, reusable JSON artifact (`data/country-codes.json`) pre-populated with authoritative data for 7 initial countries: India (`+91`), United States (`+1`), Canada (`+1`), United Kingdom (`+44`), United Arab Emirates (`+971`), Nepal (`+977`), and Sri Lanka (`+94`). India (`+91`) is flagged `isDefault: true`.

#### BR-12.3 — Strict Data Integrity & In-Use Immutability Governance
- **Database Referential Integrity (`ON DELETE RESTRICT`)**: Foreign key `countryCodeId` on `users` table referencing `country_codes(id)` is configured with `onDelete: Restrict` at backend and PostgreSQL database levels.
- **In-Use Protection Invariant**: If a `CountryCode` record is referenced by any `User` (or other system entity):
  - ❌ **No Hard Deletion**: `DELETE` operations are strictly blocked by DB constraints and API validation (HTTP `409 Conflict`, error code `COUNTRY_CODE_IN_USE`).
  - ❌ **No Disabling**: Setting `isActive = false` on a referenced country code is strictly rejected (HTTP `409 Conflict`, error code `COUNTRY_CODE_IN_USE`) to prevent breaking existing user logins or data integrity.
  - ❌ **No Modification**: Updating dial code, country code, country name, or phone regex pattern of a referenced country code is strictly rejected (HTTP `409 Conflict`, error code `COUNTRY_CODE_IN_USE`) to prevent corrupting historical user data validation.
- **Unused Records**: Only country codes referencing zero (`0`) users can be updated, disabled, or deleted.

#### BR-12.4 — Admin Configuration Controls & CRUD Endpoints
- **Role Permissions**: `SUPER_ADMIN` and `ADMIN` roles have configuration controls via `/api/v1/admin/country-codes` to create new country codes, view all entries (active/inactive) with user usage counts, and update/delete unused entries.
- **Public Metadata Access**: `GET /api/v1/meta/country-codes` serves active country codes (`isActive = true`) for login/registration dropdowns.

#### BR-12.5 — Validation & Zero Frontend Hardcoding
- **Dynamic Server Validation**: Backend NestJS + Zod (`ZodValidationPipe`) dynamically validates local phone numbers against the `phoneRegexPattern` and `minLength`/`maxLength` of the selected `CountryCode` entity.
- **Zero Frontend Hardcoding**: Frontend MUST NOT hardcode country lists or regex validation rules.

---

### BR-13: Power Supply Company Management & Referential Integrity Governance

#### BR-13.1 — Database-Driven Power Companies Registry & Seed Data
- All power utility companies are managed dynamically in the PostgreSQL database (`power_supply_companies` table).
- **Initial Pre-Seeded Companies**:
  1. Uttarakhand Power Corporation Limited (UPCL)
  2. Uttar Pradesh Power Corporation Limited (UPPCL)
  3. Reliance Power Ltd
  4. Adani Power Ltd
  5. Tata Power Company Limited (TPCL)
  6. National Thermal Power Corporation (NTPC)

#### BR-13.2 — Mandatory Building Linkage
- When registering or modifying a Building, the landlord **MUST select a Power Supply Company** from the active company list via foreign key (`power_company_id`).
- Different buildings under the same or different landlords can be linked to different power supply companies.


#### BR-13.3 — Admin Management & Deletion Constraint
- **Admin/Super Admin Management**: Authorized Admins and Super Admins can add new power companies, edit company names, or toggle company status (`ACTIVE` vs. `INACTIVE`).
- **Referential Integrity Constraint**:
  - A Power Supply Company **CANNOT be deleted or deactivated** if one or more buildings are linked to it in the database (`onDelete: Restrict`).
  - Attempting to delete or deactivate an in-use company is rejected by the backend with `HTTP 409 COMPANY_IN_USE` accompanied by a metadata payload listing all linked building names and landlord details.

#### BR-13.4 — Unique Bill Serial Number per Power Company
- A `billSerialNumber` printed on a utility invoice is unique per power supply company (`@@unique([powerCompanyId, billSerialNumber])`).
- Attempting to enter a duplicate serial number for the same power company is rejected with `HTTP 409 DUPLICATE_ENTRY`. Null values are ignored.

#### BR-13.5 — Building Connection Number Governance
- Every building must have an active `connectionNumber` (consumer account / K-number) assigned by its power supply company upon building registration (`Building.connectionNumber`).
- The connection number remains **constant across all billing cycles** for that power supplier.

#### BR-13.6 — Master Consumption & Room Submeter Cross-Verification
- A master bill records `totalUnitsConsumed` (building kWh).
- System allows landlords to cross-verify `Σ(room submeter units consumed) ≤ totalUnitsConsumed`. Any gap represents unmetered common area electricity usage.

#### BR-13.7 — Connection Number Consistency Rule
- Each building has a **unique connection number** assigned by the power supply company that remains constant across billing cycles for that supplier.
- The connection number is strictly tied to the specific power supplier.

#### BR-13.8 — Zero Open Dues Invariant on Power Supplier Switch
- A landlord **CANNOT switch** a building's power supply company if there are any `UNPAID` or `OVERDUE` `SupplierMasterBill` records associated with the current supplier.
- All pending dues for the previous supplier must be settled (`status = PAID`) or cleared in the system prior to switching.
- Attempting a switch with open dues is rejected with `HTTP 409 PENDING_SUPPLIER_BILLS_EXIST`, returning metadata containing the list of open bills.

#### BR-13.9 — Supplier Connection Transition & Historical Data Isolation
- Executing a power supplier switch (`POST /api/v1/buildings/{buildingId}/switch-power-supplier`) triggers:
  1. The current connection history record (`BuildingPowerConnection`) is closed with `status = TERMINATED` and `endDate = switchDate - 1 day`.
  2. A new connection history record is created with `status = ACTIVE`, `startDate = switchDate`, the new `powerCompanyId`, and the new `connectionNumber`.
  3. `Building.powerCompanyId` and `Building.connectionNumber` are updated to the active new supplier.
- **Strict Data Isolation**: Past `SupplierMasterBill` records remain permanently locked to their original `powerCompanyId` and original `connectionNumber`. Future master bills automatically inherit the new supplier and new connection number. No data bleeding occurs across transitions.

---

### BR-14: Partial Payment Tracking & Multi-Cycle Balance Accumulation

#### BR-14.1 — Payment Transaction Audit Log
- Each payment event (partial or full) against a `RoomRentLedger` or `ElectricityLedger` is recorded as a `PaymentTransaction` row.
- The ledger stores `amountPaid` = running sum of all associated `PaymentTransaction.amountPaid` values for that ledger.
- **Chronological Sequence**: Payment transactions are ordered by creation timestamp (`recordedAt`). Older transactions are locked and immutable. Corrections to the latest entry follow BR-14.7.

#### BR-14.2 — Ledger Status State Machine
```
UNPAID
  ├─(partial payment: 0 < amountPaid < amount)──→ PARTIALLY_PAID
  └─(full payment: amountPaid ≥ amount)──────────→ PAID

PARTIALLY_PAID
  └─(additional payment: amountPaid ≥ amount)────→ PAID

UNPAID / PARTIALLY_PAID
  └─(currentDate > dueDate)──────────────────────→ OVERDUE

OVERDUE
  └─(full settlement: amountPaid ≥ amount)───────→ PAID  (retroactive)
```
- **`dueDate`** = `BillingCycle.cycleEndDate` for both Room Rent and Electricity ledgers.
- **Strict Invariant**: A landlord cannot manually force status to `PAID` while `amountPaid < amount`. Status is driven exclusively by payment math and the due date rule.

#### BR-14.3 — OVERDUE Definition (Simplified)
- A ledger transitions to `OVERDUE` when `currentDate > BillingCycle.cycleEndDate` and its status is still `UNPAID` or `PARTIALLY_PAID`.
- **No grace period is tracked in the system.** Grace arrangements between landlord and tenant are verbal contracts and outside system scope.
- An `OVERDUE` ledger can still receive further payments. On full settlement (`amountPaid ≥ amount`), status transitions to `PAID`.

#### BR-14.4 — Multi-Cycle Balance Accumulation
- Pending balances are **NOT physically carried forward** as new ledger rows. Each billing cycle's `RoomRentLedger` / `ElectricityLedger` independently tracks its own `amount` and `amountPaid`.
- **Total outstanding** for a room is computed dynamically:
  $$\text{Total Outstanding} = \sum_{\text{non-PAID cycles}} (\text{amount} - \text{amountPaid})$$
- The API and UI must surface both **per-cycle** pending amounts and the **total accumulated outstanding** balance.

#### BR-14.5 — Payment Date Recording
- Every `PaymentTransaction` has a `paymentDate` field representing the actual date/time the payment occurred.
- If the landlord supplies an explicit date/time, that value is stored.
- If omitted, the system defaults `paymentDate` to the server's current UTC timestamp at the moment the API call is processed.
- `recordedAt` is always the server creation timestamp and is immutable regardless of user input.

#### BR-14.6 — Supplier Master Bill Settlement (Single Payment)
- When a landlord settles a `SupplierMasterBill`, they record: `status = PAID` and an optional `paidDate` (defaults to server UTC timestamp if omitted).
- **No partial payments** are supported for supplier master bills in this phase. The bill is either `UNPAID`, `OVERDUE`, or `PAID`.
- A `SupplierMasterBill` transitions to `OVERDUE` when `currentDate > SupplierMasterBill.dueDate` and status is `UNPAID`.

#### BR-14.7 — Landlord "Last Transaction Only" Update Rule
- A landlord can update a payment transaction (`PATCH /api/v1/ledgers/payments/{transactionId}`) **ONLY IF** that transaction is the **chronologically latest record** (highest `recordedAt` timestamp) for its parent ledger.
- **Modifiable Fields**: `amountPaid`, `paymentDate`, `paymentMethod`, `transactionReference`, `notes`.
- **Automatic Recalculation**: Updating the latest transaction automatically recalculates the parent ledger's `amountPaid` sum (`SUM(PaymentTransaction.amountPaid)`) and updates the parent ledger status (`UNPAID` / `PARTIALLY_PAID` / `PAID` / `OVERDUE`).
- **Prior Transaction Lock**: Landlords **CANNOT update any previous (non-latest) transactions**. Attempting to update a non-latest transaction is strictly rejected with `HTTP 409 NON_LAST_TRANSACTION_UPDATE_RESTRICTED`. UI hides/disables edit controls for non-latest records.

#### BR-14.8 — Super Admin & Admin Payment Transaction Read-Only Lock
- Super Admin and Admin roles are granted **READ-ONLY** visibility over landlord payment transaction history.
- Super Admin and Admin users **CANNOT insert, update, or delete** payment transaction records under any circumstances (`HTTP 403 FORBIDDEN`). All payment transactions are maintained exclusively by landlords.

---

### BR-15: Backend Error Handling Standards & Uniform Response Envelopes

#### BR-15.1 — Universal Error Envelope Requirement
- All non-2xx API responses across the entire system MUST conform strictly to the universal error envelope: `{ statusCode, error, message, metadata }`.
- The `error` field must be a machine-readable SCREAMING_SNAKE_CASE string registered in `docs/error-handling.md`.
- `metadata` must be a JSON object containing structured contextual details. If no additional detail is required, `metadata` must default to `{}`.

#### BR-15.2 — Field-Level Validation Response Structure
- Input validation failures (`HTTP 400 VALIDATION_ERROR`) MUST replace `metadata` with a `details` array of field error objects (`[{ field, message }]`).
- Validation error responses must identify exact failing field names and state the precise validation constraint violated.

#### BR-15.3 — Actionable Error Messages & Security Masking
- Error `message` strings must clearly state what failed AND what actionable step the client/user should take to resolve the error.
- Internal stack traces, raw database error strings, SQL queries, or internal filesystem paths must **NEVER** be exposed in API responses.

#### BR-15.4 — Enhanced Conflict Metadata
- Conflict errors (`HTTP 409`) must include actionable metadata:
  - `COMPANY_IN_USE`: Contains `affectedBuildings[]` listing linked building IDs, names, and landlord contact details.
  - `PENDING_SUPPLIER_BILLS_EXIST`: Contains `pendingBills[]` listing open bill IDs, cycle dates, amounts, and statuses blocking a supplier switch.

#### BR-15.5 — HTTP Status Code Uniformity
- Backend endpoints must adhere strictly to the HTTP status mapping matrix defined in `docs/error-handling.md §5` (`200 OK`, `201 Created`, `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `409 Conflict`, `429 Too Many Requests`, `500 Internal Server Error`).

---

### BR-16: Audit Logging & Immutability Governance

#### BR-16.1 — Mandatory Real-Time Audit Trail
- All state-changing API actions (`POST`, `PATCH`, `PUT`, `DELETE`) and security/session events (logins, logouts, force logouts, password resets) MUST emit an `AuditLog` entry in the database.
- The `AuditLog` entry MUST record: `actionType`, `category`, `performedByUserId`, `performedByUserRole`, `targetEntityId`, `targetEntityType`, client `ipAddress`, client `userAgent`, and structured JSON `metadata`.

#### BR-16.2 — Absolute Immutability Invariant
- The `AuditLog` database table (`audit_logs`) is **strictly insert-only**.
- No application controller, background service, or database user possesses permissions to execute `UPDATE` or `DELETE` queries on `audit_logs`. Any modification attempt is rejected.

#### BR-16.3 — Audit Metadata Standard
- Contextual details (e.g. before/after states, updated fields, payment transaction amounts, force logout target user details) MUST be stored in the structured `metadata` JsonB column.
- Textual log `message` strings are never used for machine-readable audit context.

#### BR-16.4 — Centralized Audit Registry Synchronization
- All recognized audit `actionType` strings and categories are formally indexed in [`docs/audit-logging.md`](file:///Users/abdulsamad/Desktop/Projects/my_room_ledger/docs/audit-logging.md).
- Whenever a new feature or state-changing action is added to the system, `docs/audit-logging.md` MUST be updated first with its `actionType` specification before backend code implementation.

#### BR-16.5 — Centralized Session Validation & Revocation Enforcement Pipeline
- **Every API Request Session Check**: Every protected API request carrying an authentication bearer token MUST execute a DB/Redis active session validation (`SessionValidationGuard`) BEFORE entering business logic controllers.
- **Revocation Overrides Access Token TTL**: Even if a 10-minute access token signature and expiration (`exp`) are valid, if the underlying database session (`user_sessions.isRevoked = true` or record purged) has been revoked or terminated, the request MUST be rejected immediately with `HTTP 401 UNAUTHORIZED` (`error: "SESSION_REVOKED"`, code: `ERR-1002`).
- **Instant Sub-500ms Revocation Propagation**: When an administrative force logout (`FORCE_LOGOUT_USER` / `FORCE_LOGOUT_ROLE`) is executed, ALL active session records for target user(s) are revoked/purged within $\le 500\text{ ms}$.
- **Frontend Auto-Logout Reaction**: Upon receiving `HTTP 401 SESSION_REVOKED`, frontend PWA client interceptors MUST immediately clear all local token storage, alert the user ("Your session has been terminated by an administrator. Please log in again."), and redirect to `/login?session_revoked=true`.

---

### BR-17: Building, Floor, Room Occupancy Engine & Stacked Navigation System Architecture
Authoritative governance rules for calculating structural occupancy, executing stacked UI navigation, responsive viewports, and enforcing tenant RBAC isolation. Detailed in [`docs/building-occupancy.md`](building-occupancy.md) (Domain occupancy rules source of truth) and [`docs/frontend-navigation.md`](frontend-navigation.md) (UI presentation & responsive navigation source of truth).

#### BR-17.1 — Active Tenant Occupancy Derivation Rule
- **Active Assignment Invariant**: A room's occupancy status is derived **strictly and exclusively** from active tenant assignments (`Tenant.status = ACTIVE` AND `Tenant.currentRoomId = room.id`).
- **Historical Data Isolation**: Past tenants who have checked out (`status = MOVED_OUT` in `TenancyHistory`) do NOT make a room occupied. A room with 0 active assigned tenants is strictly `VACANT` regardless of past tenant count.

#### BR-17.2 — Hierarchical Occupancy Aggregation Formulas
- **Room Status**: `OCCUPIED` if active tenants $\ge 1$; `VACANT` if active tenants $= 0$.
- **Floor Status**: `OCCUPIED` if $\ge 1$ room on that floor is `OCCUPIED`; `VACANT` if ALL rooms on that floor are `VACANT`. Exposes `totalRooms`, `occupiedRooms`, `vacantRooms`, and `totalActiveTenants`.
- **Building Status**: `OCCUPIED` if $\ge 1$ room in any floor is `OCCUPIED`; `VACANT` if ALL rooms across ALL floors are `VACANT`. Exposes `totalFloors`, `totalRooms`, `occupiedRooms`, `vacantRooms`, `occupiedFloors`, `vacantFloors`, and `totalActiveTenants`.

#### BR-17.3 — Backend Dynamic Aggregation Source of Truth
- Occupancy metrics are calculated dynamically via database queries (`COUNT` with index `@@index([currentRoomId, status])`) during API execution to eliminate state drift.
- Any tenant room assignment, move-out, deactivation, or room creation immediately updates query results.

#### BR-17.4 — Stacked UI Navigation Hierarchy
- Navigation follows a strict 6-tier hierarchy: `Building Details (Floor Stack)` $\rightarrow$ `Floor Details (Room Block Cards)` $\rightarrow$ `Room Details` $\rightarrow$ `Active Tenant Cards` $\rightarrow$ `Individual Tenant Details`.

#### BR-17.5 — Strict Tenant RBAC Access Isolation
- `TENANT` users can ONLY access the Room Details page for their currently assigned room (`Tenant.currentRoomId`).
- Browsing unrelated buildings, floors, rooms, or tenant profiles outside their room is strictly forbidden and rejected backend-side by `TenantRoomAccessGuard` (`HTTP 403 FORBIDDEN`, `ROOM_ACCESS_DENIED`). Co-tenants assigned to the same room can view basic profile cards of active room-mates.





