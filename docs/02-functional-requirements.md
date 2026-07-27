# Stage 02 — Functional Requirements

**Status:** In Progress  
**Date locked:** —  
**Upstream:** [01-requirement-analysis.md](01-requirement-analysis.md), MASTER.md business rules  
**Downstream:** [acceptance-criteria.md](acceptance-criteria.md), Stage 03 NFRs  
**Workflow tracker:** [00-engineering-workflow.md](00-engineering-workflow.md)

---

## 1. Purpose

Define numbered, testable functional requirements (**FR-xx**) for the **My Room Ledger** platform. Each FR traces back to the Master business rules (Rule 1–10) or Stage 01 locked decisions. Acceptance criteria live in [acceptance-criteria.md](acceptance-criteria.md).

---

## 2. Stage 02 Locked Design Decisions

| # | Decision | Source |
|---|---|---|
| 1 | Electricity charges are **never** counted as landlord revenue or profit — pass-through model only | MASTER Rule 3 |
| 2 | Net Profit = `Rent Collected − Building Operating Expenses` (Electricity excluded) | MASTER Rule 3 |
| 3 | Billing cycle creation **locks an immutable Tenancy Snapshot** — historical bills never mutate when tenants move out | MASTER Rule 8 |
| 4 | Room Rent and Electricity Ledgers are **independent entities** — updating one MUST NEVER affect the other | MASTER Rule 3 |
| 5 | Revenue dashboards are **strictly blocked** for TENANT role — zero access at any level | MASTER Rule 1 |
| 6 | Super Admin can ONLY be created by another Super Admin — no other role can create a SUPER_ADMIN | MASTER Rule 1 |
| 7 | All dates and ranges for financial reporting follow Indian Financial Year: **April 1 – March 31** | MASTER Rule 4 |
| 8 | File uploads are processed backend-only (AES-256 GCM encrypted before R2 storage) | MASTER Rule 10 |
| 9 | Every API input (body, query, params) MUST pass Zod schema validation before reaching any service | MASTER Rule 10.1 |
| 10 | Rent cycle start date can be **any day of the month** — not fixed to calendar month | MASTER Rule 2 |
| 11 | Each device login generates an **independent session record** with its own refresh token — sessions never share tokens | Auth Update |
| 12 | Logout terminates **only the current session** — other active sessions on other devices remain valid | Auth Update |
| 13 | **First login** after account creation requires a mandatory password change before accessing any feature | Auth Update |
| 14 | Landlords and Tenants **cannot self-reset** passwords — they must raise a request approved by Admin/Super Admin | Auth Update |
| 15 | Password reset tokens are **time-bound (1 hour)** and **single-use** — reuse returns an error | Auth Update |

---

## 3. Functional Requirements

### 3.1 Authentication & Session Management

#### 3.1.1 Login & Token Issuance

| ID | Requirement | Source |
|----|---|---|
| **FR-01** | A user can log in using either **Email + Password** OR **Phone Number + Country Code + Password** via a unified secure login page. | MASTER Rule 10.3 |
| **FR-01e** | Phone login requires selecting a mandatory **Country Code** dropdown (default: `+91 🇮🇳 India`). The country code is mandatory and cannot be unselected. | MASTER Rule 10.3 |
| **FR-01f** | All country codes, dial codes (`+91`), country names, flag emojis (`🇮🇳`), and phone validation regex patterns (`^[6-9]\d{9}$`) are stored in the database (`country_codes` table) and served via `GET /api/v1/meta/country-codes`. The frontend MUST NOT hardcode regex patterns. | MASTER Rule 10.3 |
| **FR-01g** | Email input is validated against RFC 5322 regex. Backend (`ZodValidationPipe`) is the strict single source of truth for validation; Frontend executes matching Zod client validation for instant UX feedback. Phone numbers are normalized to E.164 format on the backend before DB query. | MASTER Rule 10.3 |
| **FR-02** | On successful login, the system issues a short-lived **JWT access token** (15-minute expiry). | MASTER Rule 10.2 |
| **FR-03** | A **refresh token** (7-day expiry) is issued, stored server-side in a `RefreshToken` database record, and delivered to the client in an `HttpOnly; Secure; SameSite=Strict` cookie. | MASTER Rule 10.2 |
| **FR-04** | All protected API routes must include a valid JWT; the acting user's identity and role are resolved from the token. | MASTER Rule 1 |
| **FR-05** | Missing, expired, or tampered JWT on a protected route is rejected with `HTTP 401 UNAUTHORIZED`. | MASTER Rule 10.2 |
| **FR-06** | Accessing a resource beyond the user's role scope is rejected with `HTTP 403 FORBIDDEN`. | MASTER Rule 1 |
| **FR-08** | After 5 consecutive failed login attempts from the same IP within 15 minutes, the auth endpoint returns HTTP 429 `TOO_MANY_REQUESTS`. | MASTER Rule 10.2 |

#### 3.1.2 Multi-Device Session Support

| ID | Requirement | Source |
|----|---|---|
| **FR-01a** | A single user can log in from **multiple devices simultaneously**. Each login creates an independent session with its own refresh token record. | Auth Update |
| **FR-01b** | Sessions across devices are fully **isolated** — tokens for Device A are not valid on Device B, and vice versa. | Auth Update |
| **FR-01c** | A user can view all active sessions (device name/hint, last-used timestamp) from their profile. | Auth Update |
| **FR-01d** | A user can **terminate any specific session** (other than their current one) from the active sessions list. | Auth Update |

#### 3.1.3 Logout Behaviour

| ID | Requirement | Source |
|----|---|---|
| **FR-07** | On logout, the **current session only** is terminated: the server deletes the specific `RefreshToken` record and the client clears its access token and cookie. | Auth Update |
| **FR-07a** | Logging out from Device A must **not** affect any active sessions on Device B, C, etc. | Auth Update |
| **FR-07b** | The frontend clears **all** user-related state on logout: access token, cached profile data, local/session storage. | Auth Update |

---

### 3.1.4 Password Management & Recovery Workflows

#### 1. Logged-in User Change Password

| ID | Requirement | Source |
|----|---|---|
| **FR-P00a** | A logged-in user (Tenant, Landlord, Admin, Super Admin) can change their password from the profile/settings page or login page by providing: **Old Password**, **New Password**, and **Confirm New Password**. | Auth Update |
| **FR-P00b** | **Password Masking UI Toggle**: All password input fields across the entire application (login, reset, change) MUST feature a UI toggle control (show/hide eye icon) to mask or reveal entered characters. | Auth Update |
| **FR-P01** | Accounts created by Admins or reset via temporary password are flagged `mustChangePassword = true`. The user is **forced to a password change screen** before accessing any other feature. | Auth Update |
| **FR-P02** | Any API request (other than the change-password endpoint itself) made while `mustChangePassword = true` is rejected with `HTTP 403 MUST_CHANGE_PASSWORD`. | Auth Update |

#### 2. Forgot Password Workflow (With Registered Email)

| ID | Requirement | Source |
|----|---|---|
| **FR-P05** | User (Tenant, Landlord, Admin) submits a "Forgot Password" request from the login page by providing their registered email. | Auth Update |
| **FR-P06** | Raising a reset request triggers a real-time notification badge in the **Admin / Super Admin frontend header notification panel** (header bell icon). | Auth Update |
| **FR-P07** | Upon Admin/Super Admin approval, the system generates a signed reset link (token valid for **15 minutes**) and emails it to the user. | Auth Update |
| **FR-P08** | Clicking the email link redirects the user to the frontend reset page (`/reset-password?token=...`), displaying the user's **Name** and **Email**. | Auth Update |
| **FR-P09** | User enters **New Password** + **Confirm New Password** (masked with toggle, regex validated). On success, the reset token expires immediately (single-use), all active sessions across all devices are invalidated, and the user is redirected to the Login page. | Auth Update |
| **FR-P09a** | Using a reset link token more than once or after 15 minutes returns `HTTP 400 TOKEN_EXPIRED` or `HTTP 400 TOKEN_ALREADY_USED`. | Auth Update |

#### 3. Fallback Workflow (No Registered Email — Admin, Landlord & Tenant)

| ID | Requirement | Source |
|----|---|---|
| **FR-P10** | **Admin / Fallback Temp Password Generation**: When a user has no registered email (or for direct Admin reset), Admin/Super Admin approval generates a secure **temporary password** (valid for **30 minutes**) displayed in a single-view Admin modal for secure manual share (in-person/SMS). | Auth Update |
| **FR-P11** | User logs in using the temporary password and is automatically redirected to the **"Create New Password" page** (`mustChangePassword = true`). | Auth Update |
| **FR-P11a** | The "Create New Password" page displays the user's **Name** (and Email if available). User enters **New Password** + **Confirm New Password**, validated against regex complexity rules with field-level error messages. | Auth Update |
| **FR-P12** | **Global Cross-Device Session Invalidation**: Upon any successful password update or reset, **ALL active sessions across ALL devices** for that user are immediately invalidated (`RefreshToken` records purged from DB). User must log in again on all devices. | Auth Update |

#### 4. Security Constraints on Passwords

| ID | Requirement | Source |
|----|---|---|
| **FR-P13** | All passwords are stored as **bcrypt hashes** (minimum 12 salt rounds). Plaintext passwords are never stored, logged, or transmitted. | MASTER Rule 10.2 |
| **FR-P14** | New passwords must meet minimum complexity: at least 8 characters, at least 1 uppercase, 1 lowercase, 1 digit, and 1 special character. | Auth Update |
| **FR-P15** | A user cannot reuse their last 3 passwords when setting a new one. | Auth Update |

---

### 3.2 Super Admin Management

| ID | Requirement | Source |
|----|---|---|
| **FR-09** | A **Super Admin** can create a new **Admin** user (providing full name, email, phone, password). | MASTER Rule 1 |
| **FR-10** | A Super Admin can update an Admin user's profile details. | MASTER Rule 1 |
| **FR-11** | A Super Admin can deactivate or delete an Admin user. | MASTER Rule 1 |
| **FR-12** | A Super Admin can view a list of all Admins in the system. | MASTER Rule 1 |
| **FR-13** | A Super Admin can view system-wide platform metrics: total landlords, total buildings, total tenants, total revenue. | MASTER Rule 1 |
| **FR-14** | A new Super Admin can **only** be created by an **existing Super Admin**. Any other role attempting this is rejected with HTTP 403. | MASTER Rule 1 |

---

### 3.3 Admin Management

| ID | Requirement | Source |
|----|---|---|
| **FR-15** | An **Admin** can create a new **Landlord** account (providing full name, email, phone, password). | MASTER Rule 1 |
| **FR-16** | An Admin can update a Landlord's profile details. | MASTER Rule 1 |
| **FR-17** | An Admin can deactivate a Landlord account. | MASTER Rule 1 |
| **FR-18** | An Admin can view a list of all Landlords registered in the system. | MASTER Rule 1 |
| **FR-19** | An Admin can view all buildings and tenants associated with any Landlord. | MASTER Rule 1 |
| **FR-20** | An Admin can view system-wide financial metrics and landlord performance breakdowns. | MASTER Rule 4 |
| **FR-21** | An Admin **cannot** create, modify, or delete a Super Admin user — attempts are rejected with HTTP 403. | MASTER Rule 1 |

---

### 3.4 Building & Asset Hierarchy Management

| ID | Requirement | Source |
|----|---|---|
| **FR-22** | A **Landlord** can create a new **Building** (name, address, city, state, pincode, power supplier). | MASTER Rule 5 |
| **FR-23** | A Landlord can update their own Building's details. | MASTER Rule 5 |
| **FR-24** | A Landlord can add **Floors** to a Building (floor number, label). | MASTER Rule 5 |
| **FR-25** | A Landlord can add **Rooms** to a Floor, specifying: room label, max occupancy, monthly rent amount, and bathroom type (`PRIVATE_ATTACHED` or `SHARED_FLOOR`). | MASTER Rule 5, 6 |
| **FR-26** | A Landlord can add **Shared Bathrooms** and **Shared Toilets** to a Floor with standardized labels (`Bath XY`, `Toilet XY`). | MASTER Rule 6, 7 |
| **FR-27** | A Landlord can only view, modify, and manage **their own** buildings. Accessing another landlord's building is rejected with HTTP 403. | MASTER Rule 1 |
| **FR-28** | A Landlord can view the full asset hierarchy: Building → Floor → Room / Shared Facilities. | MASTER Rule 5 |
| **FR-29** | Room numbering follows the scheme `Room XY` where `X` = floor number (0 for Ground) and `Y` = room index on that floor. | MASTER Rule 7 |

---

### 3.5 Tenant Lifecycle Management

| ID | Requirement | Source |
|----|---|---|
| **FR-30** | A Landlord can **onboard a new Tenant** to a specific room: name, phone, email, government ID metadata, move-in date. | MASTER Rule 8 |
| **FR-31** | A Landlord can **check out a Tenant** from a room, recording move-out date and noting any outstanding balances. | MASTER Rule 8 |
| **FR-32** | Each tenancy is stored as a history record — multiple tenants can occupy the same room across different time periods. | MASTER Rule 8 |
| **FR-33** | A Landlord can upload a **Government ID document** for a tenant (Aadhaar, PAN, Passport). The file is AES-256 GCM encrypted before Cloudflare R2 storage. | MASTER Rule 10.2 |
| **FR-34** | A Tenant can view the history of their own room stays and payment records (rent + electricity). Read-only. | MASTER Rule 1 |
| **FR-35** | A Tenant has **zero access** to other tenants' data, other rooms, or any revenue / P&L dashboards. | MASTER Rule 1 |

---

### 3.6 Rent Cycle & Room Rent Ledger

| ID | Requirement | Source |
|----|---|---|
| **FR-36** | A Landlord can configure a **Rent Cycle** for a building: cycle start day (1–28) and rent due day. | MASTER Rule 2 |
| **FR-37** | Rent cycle start day can be **any day of the month** (e.g., 5th to 4th of the following month). | MASTER Rule 2 |
| **FR-38** | A Landlord can **generate a Billing Cycle** for a room, providing cycle start and end dates. | MASTER Rule 8 |
| **FR-39** | On billing cycle generation, the system **locks an immutable Tenancy Snapshot** capturing all active tenants in that room. Subsequent tenant check-outs do not alter historical snapshots. | MASTER Rule 8 |
| **FR-40** | Each billing cycle produces an independent **Room Rent Ledger** entry per room with status `UNPAID`. | MASTER Rule 3 |
| **FR-41** | A Landlord can update Room Rent Ledger payment status: `UNPAID` → `PARTIALLY_PAID` → `PAID` or `OVERDUE`. | MASTER Rule 3 |
| **FR-42** | Updating Room Rent status **must never** alter the Electricity Ledger status for the same cycle — ledgers are fully independent. | MASTER Rule 3 |
| **FR-43** | A Landlord can record payment details: amount paid, payment date, method (`UPI`, `CASH`, `BANK_TRANSFER`), and transaction reference. | Stage 01 scope |
| **FR-44** | A Tenant can view their own Room Rent Ledger history (read-only). | MASTER Rule 1 |

---

### 3.7 Electricity Billing — Pass-Through Model

| ID | Requirement | Source |
|----|---|---|
| **FR-45** | Each room has an independent **electricity submeter**. The Landlord inputs: units consumed, billing cycle dates, and rate per unit (₹/unit). | MASTER Rule 3 |
| **FR-46** | The system **auto-calculates** Total Electricity Bill: `Total Bill = Units Consumed × Rate per Unit`. | MASTER Rule 3 |
| **FR-47** | Each billing cycle produces an independent **Electricity Ledger** entry per room with status `UNPAID`. | MASTER Rule 3 |
| **FR-48** | A Landlord can update Electricity Ledger payment status: `UNPAID` → `PARTIALLY_PAID` → `PAID` or `OVERDUE`. | MASTER Rule 3 |
| **FR-49** | Updating Electricity Ledger status **must never** alter Room Rent Ledger status — fully isolated. | MASTER Rule 3 |
| **FR-50** | A Landlord can record the **Power Supplier (UPCL) Master Bill** per billing period: supplier name, bill cycle dates, total amount, due date, payment status. | MASTER Rule 3 |
| **FR-51** | A Landlord can upload the **digital power bill (PDF/Image)** for the supplier master bill. Encrypted AES-256 GCM before Cloudflare R2 storage. | MASTER Rule 3, 10 |
| **FR-52** | Electricity collections from tenants are **never counted** as landlord revenue or profit. They appear in pass-through tracking only. | MASTER Rule 3 |
| **FR-52a** | The system must track **monthly and yearly (IFY) total electricity collected from tenants** across all rooms vs. total master bill amount payable to the power company per billing cycle. | MASTER Rule 3, 4 |
| **FR-52b** | The system must auto-calculate **Electricity Variance**: `Electricity Variance = Tenant Electricity Collected − Power Supplier Master Bill Amount`. | MASTER Rule 3 |
| **FR-52c** | The system must categorize and display reconciliation status as **SURPLUS** (extra collected funds), **DEFICIT** (under-collected, landlord paying out-of-pocket), or **BALANCED** (exact match between collections and master bill). | MASTER Rule 3 |
| **FR-53** | A Tenant can view their own Electricity Ledger entries (read-only). | MASTER Rule 1 |

---

### 3.8 Building Operating Expenses

| ID | Requirement | Source |
|----|---|---|
| **FR-54** | A Landlord can log a **Building Operating Expense**: category, title, amount (₹), expense date, and optional notes. | MASTER Rule 4 |
| **FR-55** | Expense categories: `WATER_BILL`, `MAINTENANCE`, `REPAIRS`, `SECURITY`, `CLEANING`, `PROPERTY_TAX`, `MISCELLANEOUS`. | MASTER Rule 4 |
| **FR-56** | A Landlord can view all expenses for a building, filterable by category and date range. | MASTER Rule 4 |
| **FR-57** | Operating Expenses are factored into the **Net Profit formula**: `Net Profit = Rent Collected − Operating Expenses`. | MASTER Rule 3, 4 |

---

### 3.9 Revenue Dashboard & P&L Analytics

| ID | Requirement | Source |
|----|---|---|
| **FR-58** | A **Landlord** can view a Revenue Dashboard scoped strictly to **their own buildings**. | MASTER Rule 1, 4 |
| **FR-59** | The dashboard displays: Total Rent Collected, Total Operating Expenses, Net Profit, Total Electricity Collected, Total Supplier Bill Amount, **Electricity Variance & Reconciliation Status (Surplus / Deficit / Balanced)**, Pending Rent Dues, Occupancy Rate %. | MASTER Rule 3, 4 |
| **FR-60** | The dashboard supports time window filters: **Monthly**, **Indian Financial Year (April 1 – March 31)**, and **Custom Date Range**. | MASTER Rule 4 |
| **FR-61** | The dashboard supports **granularity**: Single Building view, or full Landlord Portfolio (all buildings) rolled up. | MASTER Rule 4 |
| **FR-62** | Multi-level aggregation: **Level 1** (single building), **Level 2** (landlord portfolio), **Level 3** (per-landlord admin view), **Level 4** (system-wide super admin). | MASTER Rule 4 |
| **FR-63** | **Historical trend charts** show multi-year rent revenue, expense trends, occupancy history, and electricity reconciliation surplus/deficit history (Recharts). | MASTER Rule 4 |
| **FR-64** | **Risk indicators** surface early warnings: mounting unpaid dues, declining occupancy, overdue spikes, and **growing electricity deficits (out-of-pocket losses)**. | MASTER Rule 4 |
| **FR-65** | A **Tenant** has **zero access** to any Revenue Dashboard or P&L data — blocked at both API (HTTP 403) and UI layers. | MASTER Rule 1 |
| **FR-66** | An **Admin** can view per-landlord financial breakdowns (Level 3 aggregation). | MASTER Rule 1, 4 |
| **FR-67** | A **Super Admin** can view system-wide platform financial metrics (Level 4 aggregation). | MASTER Rule 1, 4 |

---

### 3.10 Secure Document Management

| ID | Requirement | Source |
|----|---|---|
| **FR-68** | File uploads are processed through the **backend API only** — clients never upload directly to Cloudflare R2. | MASTER Rule 10.2 |
| **FR-69** | The backend **encrypts every uploaded file** using AES-256 GCM before writing to Cloudflare R2. The encryption key, IV, and auth tag are stored in PostgreSQL `DocumentMetadata`. | MASTER Rule 10.2 |
| **FR-70** | File access is provided **only via short-lived backend-generated signed URLs** (15-minute expiry). The backend decrypts and streams the binary to the authorised client. | MASTER Rule 10.2 |
| **FR-71** | Allowed file types: `.pdf`, `.jpg`, `.jpeg`, `.png`. Maximum file size: **10 MB**. Backend enforces both limits before processing. | Stage 01 EC-ELE-05 |
| **FR-72** | File access is gated by role and resource ownership — a user can only retrieve documents they are authorised for. | MASTER Rule 1, 10.2 |

---

### 3.11 Complaints & Maintenance System

| ID | Requirement | Source |
|----|---|---|
| **FR-73** | A **Tenant** can submit a maintenance complaint: category, severity, title, and description. | MASTER Rule 9 |
| **FR-74** | Complaint categories: `PLUMBING`, `ELECTRICAL`, `CLEANLINESS`, `NOISE`, `BILLING`, `OTHER`. | MASTER Rule 9 |
| **FR-75** | Complaint severity levels: `LOW`, `MEDIUM`, `HIGH`, `URGENT`. | MASTER Rule 9 |
| **FR-76** | A **Landlord** can view all complaints for their own buildings, update status (`OPEN` → `IN_PROGRESS` → `RESOLVED` / `REJECTED`), and add resolution notes. | MASTER Rule 9 |
| **FR-77** | A Tenant can track the status history of their own submitted complaints (read-only). | MASTER Rule 9 |

---

### 3.12 Input Validation & System-Wide Rules

| ID | Requirement | Source |
|----|---|---|
| **FR-78** | **Every** API endpoint — body, query parameters, and route parameters — is validated through a **Zod schema** via `ZodValidationPipe` before reaching any Controller or Service. | MASTER Rule 10.1 |
| **FR-79** | Validation failures return `HTTP 400 VALIDATION_ERROR` with structured field-level error details. No partial processing of invalid input occurs. | MASTER Rule 10.1 |
| **FR-80** | All string inputs are sanitized: `.trim()`, minimum length enforced, and pattern-matched where applicable (e.g., phone numbers, email format). | MASTER Rule 10.1 |
| **FR-81** | All date inputs are validated as ISO 8601. Indian Financial Year boundary rules (Apr 1 – Mar 31) are enforced for all date range queries. | MASTER Rule 4, 10.1 |

---

## 4. Traceability Map (FR → AC Group)

| FR IDs | AC Group | Domain |
|---|---|---|
| FR-01, FR-02, FR-03, FR-04, FR-05, FR-06, FR-08 | AC-01 | Login, Tokens & RBAC |
| FR-01a – FR-01d | AC-01-sessions | Multi-Device Session Management |
| FR-07, FR-07a, FR-07b | AC-01-logout | Logout & Session Isolation |
| FR-P01 – FR-P04 | AC-P01 | First-Login Mandatory Password Change |
| FR-P05 – FR-P12 | AC-P05 | Admin-Controlled Password Reset Flow |
| FR-P13 – FR-P15 | AC-P13 | Password Security Constraints |
| FR-09 – FR-14 | AC-09 | Super Admin Management |
| FR-15 – FR-21 | AC-15 | Admin Management |
| FR-22 – FR-29 | AC-22 | Building & Asset Hierarchy |
| FR-30 – FR-35 | AC-30 | Tenant Lifecycle |
| FR-36 – FR-44 | AC-36 | Rent Cycle & Room Rent Ledger |
| FR-45 – FR-53 | AC-45 | Electricity Pass-Through Model |
| FR-54 – FR-57 | AC-54 | Building Operating Expenses |
| FR-58 – FR-67 | AC-58 | Revenue Dashboard & P&L Analytics |
| FR-68 – FR-72 | AC-68 | Secure Document Management |
| FR-73 – FR-77 | AC-73 | Complaints & Maintenance |
| FR-78 – FR-81 | AC-78 | Input Validation & System Rules |

---

## 5. Explicitly Out of Scope (Reaffirmed for Stage 02)

- Automated online payment gateway (Razorpay / Stripe) — deferred to future phase
- Automated SMS / WhatsApp OTP login
- Tenant-to-tenant messaging
- Automated late-fee calculation and penalty engine
- Multi-currency support

---

## 6. Open Items Deferred

| Item | Defer to |
|---|---|
| Exact seed user list and sample data | Stage 05 / 11 |
| Predictive forecasting algorithm specification | Stage 08 |
| Email delivery infrastructure (SMTP / transactional email provider) | Stage 08 / 11 |
| Push notification system | Future phase |

---

## 7. Gate

**Awaiting user confirmation.**

**Next:** Stage 03 — Non-Functional Requirements → `docs/03-non-functional-requirements.md`
