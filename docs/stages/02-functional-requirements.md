# Stage 02 — Functional Requirements Specification

**Status:** Confirmed ✅ (Approved by User)  
**Upstream:** [01-requirement-analysis.md](01-requirement-analysis.md), MASTER.md business rules  
**Downstream:** [acceptance-criteria.md](acceptance-criteria.md), Stage 03 NFRs  
**Workflow tracker:** [../00-engineering-workflow.md](../00-engineering-workflow.md)

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
| **FR-01f** | All country codes, dial codes (`+91`), country names, flag emojis (`🇮🇳`), and phone validation regex patterns (`^[6-9]\d{9}$`) are stored in the database (`country_codes` table), seeded from reusable base JSON (`data/country-codes.json`), and served via `GET /api/v1/meta/country-codes`. Deleting, disabling, or modifying in-use country codes is strictly rejected (`ON DELETE RESTRICT` / HTTP 409). The frontend MUST NOT hardcode regex patterns. | MASTER Rule 10.3 |
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
| **FR-P07** | Upon Admin/Super Admin approval, the system generates a signed reset link (token valid for **15 minutes** as specified in [`docs/ttl-registry.md`](ttl-registry.md)) and emails it to the user. | Auth Update, docs/ttl-registry.md |
| **FR-P08** | Clicking the email link redirects the user to the frontend reset page (`/reset-password?token=...`), displaying the user's **Name** and **Email**. | Auth Update |
| **FR-P09** | User enters **New Password** + **Confirm New Password** (masked with toggle, regex validated). On success, the reset token expires immediately (single-use), all active sessions across all devices are invalidated, and the user is redirected to the Login page. | Auth Update |
| **FR-P09a** | Using a reset link token more than once or after 15 minutes returns `HTTP 400 TOKEN_EXPIRED` or `HTTP 400 TOKEN_ALREADY_USED`. | Auth Update |

#### 3. Fallback Workflow (No Registered Email — Admin, Landlord & Tenant)

| ID | Requirement | Source |
|----|---|---|
| **FR-P10** | **Admin / Fallback Temp Password Generation**: When a user has no registered email (or for direct Admin reset), Admin/Super Admin approval generates a secure **temporary password** (valid for **30 minutes** as specified in [`docs/ttl-registry.md`](ttl-registry.md)) displayed in a single-view Admin modal for secure manual share (in-person/SMS). | Auth Update, docs/ttl-registry.md |
| **FR-P11** | User logs in using the temporary password and is automatically redirected to the **"Create New Password" page** (`mustChangePassword = true`). | Auth Update |
| **FR-P11a** | The "Create New Password" page displays the user's **Name** (and Email if available). User enters **New Password** + **Confirm New Password**, validated against regex complexity rules with field-level error messages. | Auth Update |
| **FR-P12** | **Global Cross-Device Session Invalidation**: Upon any successful password update or reset, **ALL active sessions across ALL devices** for that user are immediately invalidated (`RefreshToken` records purged from DB). User must log in again on all devices. | Auth Update |

#### 4. Security Constraints on Passwords

| ID | Requirement | Source |
|----|---|---|
| **FR-P13** | All passwords MUST be stored as **Argon2id hashes** (memory: 64MB, 3 iterations, 4 parallelism threads). Plaintext passwords are never stored, logged, or transmitted. | BR-10.2, BR-11.4 |
| **FR-P14** | New passwords must meet minimum complexity: at least 8 characters, at least 1 uppercase, 1 lowercase, 1 digit, and 1 special character. | Auth Update |
| **FR-P15** | A user cannot reuse their last 3 passwords when setting a new one. | Auth Update |

#### 5. Country Code Management & Seeding

| ID | Requirement | Source |
|----|---|---|
| **FR-35a** | The system MUST seed the `country_codes` database table during initial deployment using a reusable base JSON file (`data/country-codes.json`) pre-populated with authoritative data for 7 initial countries: India (`+91`), United States (`+1`), Canada (`+1`), United Kingdom (`+44`), United Arab Emirates (`+971`), Nepal (`+977`), and Sri Lanka (`+94`). | BR-12.2 |
| **FR-35b** | Super Admin (`SUPER_ADMIN`) and Admin (`ADMIN`) roles can access `/api/v1/admin/country-codes` to view all country codes (active and inactive) along with total user usage counts. | BR-12.4 |
| **FR-35c** | Super Admin and Admin roles can create new country codes (`POST /api/v1/admin/country-codes`) with unique `countryCode`, `dialCode`, `countryName`, `flagEmoji`, `phoneRegexPattern`, and min/max length parameters. | BR-12.4 |
| **FR-35d** | Super Admin and Admin roles can update unused country codes (`PUT /api/v1/admin/country-codes/:id`). If a country code is currently referenced by any user, updates (changing dial code, country code, country name, or phone regex) MUST be rejected with `HTTP 409 Conflict` (`COUNTRY_CODE_IN_USE`). | BR-12.3, BR-12.4 |
| **FR-35e** | Super Admin and Admin roles can disable or enable unused country codes (`isActive = false`). Disabling a country code referenced by active users MUST be rejected with `HTTP 409 Conflict` (`COUNTRY_CODE_IN_USE`). | BR-12.3 |
| **FR-35f** | Deleting a country code (`DELETE /api/v1/admin/country-codes/:id`) referenced by any user MUST be rejected at database (`ON DELETE RESTRICT`) and API layers with `HTTP 409 Conflict` (`COUNTRY_CODE_IN_USE`). Only unused country codes (0 referenced users) can be deleted. | BR-12.3 |

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
| **FR-22** | A **Landlord** can create a new **Building** by specifying: name, address, city, state, pincode, selected **Power Supply Company** (`powerCompanyId`), and mandatory active **Connection Number** (`connectionNumber`). | MASTER Rule 5, BR-13.5 |
| **FR-22a** | A **Landlord** can execute a **Power Supplier Switch** (`POST /api/v1/buildings/{buildingId}/switch-power-supplier`), providing `newPowerCompanyId`, `newConnectionNumber`, `effectiveDate`, and optional `notes`. | BR-13.8, BR-13.9 |
| **FR-22b** | The system enforces a **Zero Open Dues Pre-Condition** before allowing a power supplier switch. If any `UNPAID` or `OVERDUE` `SupplierMasterBill` records exist for the current supplier, the endpoint rejects the switch with `HTTP 409 PENDING_SUPPLIER_BILLS_EXIST` and returns a list of open bills in `metadata`. | BR-13.8 |
| **FR-22c** | A **Landlord** can view a building's full historical audit log of power supplier connections (`GET /api/v1/buildings/{buildingId}/power-supplier-history`), returning active and past connections with start and end dates. | BR-13.9 |
| **FR-23** | A Landlord can update their own Building's details (name, address, city, state, pincode). Power supplier changes MUST be performed via the dedicated switch endpoint (FR-22a). | MASTER Rule 5, BR-13 |
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
| **FR-41** | A Landlord can record a **payment transaction** against a Room Rent Ledger (partial or full), supplying: `amountPaid`, optional `paymentDate` (defaults to server UTC timestamp), `paymentMethod` (`UPI`, `CASH`, `BANK_TRANSFER`), optional `transactionReference`, and optional `notes`. | BR-14.1 |
| **FR-41a** | After each payment transaction, the system auto-updates `RoomRentLedger.amountPaid` (running sum) and transitions `status`: `UNPAID` / `PARTIALLY_PAID` → `PARTIALLY_PAID` (if 0 < paid < amount) or `PAID` (if paid ≥ amount). | BR-14.2 |
| **FR-41b** | System transitions a Room Rent Ledger to `OVERDUE` when `currentDate > BillingCycle.cycleEndDate` and status is `UNPAID` or `PARTIALLY_PAID`. No grace period. | BR-14.3 |
| **FR-41c** | API provides a **total outstanding balance** endpoint for a room: `SUM(amount − amountPaid)` across all non-`PAID` ledger entries (Rent + Electricity), with a per-cycle breakdown. | BR-14.4 |
| **FR-42** | Updating Room Rent status **must never** alter the Electricity Ledger status for the same cycle — ledgers are fully independent. | MASTER Rule 3 |
| **FR-43** | *(Superseded by FR-41)* Payment metadata (`paymentMethod`, `transactionReference`, `paymentDate`) is now recorded per `PaymentTransaction` row, not stored directly on the ledger. | BR-14.1 |
| **FR-44** | A Tenant can view their own Room Rent Ledger history and individual payment transactions (read-only). | MASTER Rule 1 |

---

### 3.7 Power Supply Companies Management

| ID | Requirement | Source |
|----|---|---|
| **FR-50a** | System database pre-seeds a central registry of **Power Supply Companies** (`power_supply_companies` table): UPCL, UPPCL, Reliance Power Ltd, Adani Power Ltd, TPCL, NTPC. | BR-13.1 |
| **FR-50b** | An API endpoint (`GET /api/v1/meta/power-companies`) serves the list of active power supply companies to populate frontend Building creation/edit dropdowns. | BR-13.1 |
| **FR-50c** | An **Admin** or **Super Admin** can create a new power company, update existing company names, or toggle status (`ACTIVE` / `INACTIVE`). | BR-13.3 |
| **FR-50d** | An Admin/Super Admin can **delete or deactivate** a power supply company **ONLY IF** zero buildings are linked to it. | BR-13.3 |
| **FR-50e** | If an Admin attempts to delete or deactivate a power company linked to $\ge 1$ building, the request is rejected with `HTTP 400 COMPANY_IN_USE` or `HTTP 409 CONFLICT`. | BR-13.3 |

---

### 3.8 Electricity Billing — Pass-Through Model & Rate Differential

| ID | Requirement | Source |
|----|---|---|
| **FR-45** | Each room has an independent **electricity submeter**. Landlord inputs: units consumed, billing cycle dates, and landlord rate per unit (e.g. ₹8/unit). | MASTER Rule 3, BR-03.2 |
| **FR-46** | System auto-calculates Room Electricity Bill: `Room Elec Bill = Units Consumed × Landlord Rate per Unit`. | MASTER Rule 3 |
| **FR-47** | Each billing cycle produces an independent **Electricity Ledger** entry per room with status `UNPAID`. | MASTER Rule 3 |
| **FR-48** | A Landlord can record a **payment transaction** against an Electricity Ledger (partial or full), supplying: `amountPaid`, optional `paymentDate` (defaults to server UTC timestamp), `paymentMethod`, optional `transactionReference`, and optional `notes`. | BR-14.1 |
| **FR-48a** | After each electricity payment transaction, the system auto-updates `ElectricityLedger.amountPaid` and transitions `status` using the same state machine as Room Rent (BR-14.2). | BR-14.2 |
| **FR-49** | Updating Electricity Ledger status **must never** alter Room Rent Ledger status — fully isolated. | MASTER Rule 3 |
| **FR-50** | Landlord records a **Power Supplier Master Bill** per billing period: selected Power Supply Company, connection number, optional bill serial number, bill cycle dates (start + end), optional invoice date, optional total units consumed (kWh), total master bill amount, and due date. | BR-03.3, BR-13, BR-14.6 |
| **FR-50a** | Landlord can mark a `SupplierMasterBill` as `PAID`, optionally recording settlement `paymentMode` (`NEFT`, `UPI`, `CHEQUE`, `ONLINE_PORTAL`, `CASH`), `paymentReference` (UTR/cheque ID), `paidDate` (defaults to server UTC if omitted), and `notes`. Single lump-sum settlement only. | BR-14.6 |
| **FR-51** | Landlord can upload **digital power bill (PDF/Image)** for supplier bill. AES-256 GCM encrypted before Cloudflare R2 storage. | MASTER Rule 3, 10 |
| **FR-52** | Electricity collections from tenants are **never counted** as landlord revenue or profit. Excluded from Net Rental Profit formula. | BR-03.1 |
| **FR-52a** | System tracks **monthly and yearly (IFY) total tenant electricity collected** across all rooms vs. total master supplier bill amount payable per billing period. | BR-03.3, BR-04 |
| **FR-52b** | System auto-calculates **Electricity Variance**: `Electricity Variance = Tenant Electricity Collected − Supplier Master Bill Amount`. | BR-03.3 |
| **FR-52c** | System categorizes and displays 3 reconciliation outcomes: **SURPLUS** (Extra Savings when Collection > Master Bill), **DEFICIT** (Landlord Loss / Out-of-Pocket when Collection < Master Bill), and **BREAK_EVEN** (Collection = Master Bill). | BR-03.3 |
| **FR-52d** | Unmetered common electricity usage (common lighting, submersible water motor pumps) is paid from surplus or landlord rental income and tracked under Building Operating Expenses (`WATER_MOTOR_ELECTRICITY` / `COMMON_ELECTRICITY`). | BR-03.2 |
| **FR-53** | A Tenant can view their own Electricity Ledger entries and individual payment transactions (read-only). | MASTER Rule 1 |

---

### 3.9 Building Operating Expenses

| ID | Requirement | Source |
|----|---|---|
| **FR-54** | A Landlord can log a **Building Operating Expense**: category, title, amount (₹), expense date, and optional notes. | MASTER Rule 4 |
| **FR-55** | Expense categories: `WATER_BILL`, `COMMON_ELECTRICITY`, `WATER_MOTOR_ELECTRICITY`, `MAINTENANCE`, `REPAIRS`, `SECURITY`, `CLEANING`, `PROPERTY_TAX`, `MISCELLANEOUS`. | MASTER Rule 4, BR-03.2 |
| **FR-56** | A Landlord can view all expenses for a building, filterable by category and date range. | MASTER Rule 4 |
| **FR-57** | Operating Expenses are factored into the **Net Profit formula**: `Net Profit = Rent Collected − Operating Expenses`. | MASTER Rule 3, 4 |

---

### 3.10 Revenue Dashboard & P&L Analytics

| ID | Requirement | Source |
|----|---|---|
| **FR-58** | A **Landlord** can view a Revenue Dashboard scoped strictly to **their own buildings**. | MASTER Rule 1, 4 |
| **FR-59** | The dashboard displays distinct financial KPI cards: **Total Rent Collected**, **Total Operating Expenses**, **Net Rental Profit**, **Total Tenant Electricity Collected**, **Total Supplier Utility Bill**, **Electricity Surplus (Extra Savings)**, **Electricity Deficit (Landlord Out-of-Pocket)**, Pending Rent Dues, Occupancy Rate %. | BR-03.3, BR-04 |
| **FR-60** | The dashboard supports time window filters: **Monthly**, **Indian Financial Year (April 1 – March 31)**, and **Custom Date Range**. | MASTER Rule 4 |
| **FR-61** | The dashboard supports **granularity**: Single Building view, or full Landlord Portfolio (all buildings) rolled up. | MASTER Rule 4 |
| **FR-62** | Multi-level aggregation: **Level 1** (single building), **Level 2** (landlord portfolio), **Level 3** (per-landlord admin view), **Level 4** (system-wide super admin). | MASTER Rule 4 |
| **FR-63** | **Historical trend charts** show multi-year rent revenue, expense trends, occupancy history, and separate **Electricity Surplus vs. Deficit trend lines** (Recharts). | BR-03.3, BR-04 |
| **FR-64** | **Risk indicators** surface early warnings: mounting unpaid dues, declining occupancy, overdue spikes, and **growing electricity deficits (out-of-pocket losses)**. | BR-03.3, BR-04 |
| **FR-65** | A **Tenant** has **zero access** to any Revenue Dashboard or P&L data — blocked at both API (HTTP 403) and UI layers. | MASTER Rule 1 |
| **FR-66** | An **Admin** can view per-landlord financial breakdowns (Level 3 aggregation). | MASTER Rule 1, 4 |
| **FR-67** | A **Super Admin** can view system-wide platform financial metrics (Level 4 aggregation). | MASTER Rule 1, 4 |

---

### 3.10 Secure Document Management

| ID | Requirement | Source |
|----|---|---|
| **FR-68** | File uploads are processed through the **backend API only** — clients never upload directly to Cloudflare R2. | MASTER Rule 10.2 |
| **FR-69** | The backend **encrypts every uploaded file** using AES-256 GCM before writing to Cloudflare R2. The encryption key, IV, and auth tag are stored in PostgreSQL `DocumentMetadata`. | MASTER Rule 10.2 |
| **FR-70** | File access is provided **only via short-lived backend-generated signed URLs** (15-minute expiry as defined in [`docs/ttl-registry.md`](ttl-registry.md)). The backend decrypts and streams the binary to the authorised client. | MASTER Rule 10.2, docs/ttl-registry.md |
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

### 3.12 Backend Error Handling & Input Validation Standards

| ID | Requirement | Source |
|----|---|---|
| **FR-78** | **Every** API endpoint — body, query parameters, and route parameters — is validated through a **Zod schema** via `ZodValidationPipe` before reaching any Controller or Service. Route parameter entity IDs are coerced and validated as RFC 4122 / draft UUIDv7 strings (`z.string().uuid()`). | MASTER Rule 10.1, BR-10.4 |
| **FR-79** | Validation failures return `HTTP 400 VALIDATION_ERROR` with structured field-level error details (`details: [{ field, message }]`). No partial processing of invalid input occurs. | MASTER Rule 10.1, BR-15.2 |
| **FR-80** | All string inputs are sanitized: `.trim()`, minimum length enforced, and pattern-matched where applicable (e.g., phone numbers, email format, UUID format). | MASTER Rule 10.1 |
| **FR-81** | All date inputs are validated as ISO 8601. Indian Financial Year boundary rules (Apr 1 – Mar 31) are enforced for all date range queries. | MASTER Rule 4, 10.1 |
| **FR-82** | All non-2xx API responses across the system MUST conform to the universal error response envelope (`{ statusCode, error, message, metadata }`) defined in `docs/error-handling.md`. | BR-15.1 |
| **FR-83** | Deleting or deactivating a `PowerSupplyCompany` linked to ≥1 buildings MUST be rejected with `HTTP 409 COMPANY_IN_USE` accompanied by `metadata.affectedBuildings[]` listing linked building names and landlord details. | BR-13.3, BR-15.4 |
| **FR-84** | Attempting to record a `PaymentTransaction` where `amountPaid` + existing `ledger.amountPaid` > `ledger.amount` MUST be rejected with `HTTP 400 PAYMENT_EXCEEDS_BALANCE`. | BR-14.2, BR-15.5 |
| **FR-85** | Requesting a resource ID that does not exist in the database MUST return `HTTP 404 NOT_FOUND`. Requesting a resource owned by another user MUST return `HTTP 403 FORBIDDEN`. | BR-15.5 |
| **FR-86** | Creating a user (Admin, Landlord, Tenant) with an email or phone number that is already registered MUST return `HTTP 409 DUPLICATE_ENTRY`. | BR-15.5 |
| **FR-87** | Unhandled server exceptions MUST return `HTTP 500 INTERNAL_SERVER_ERROR` with a generic user message. Stack traces, database constraint names, and internal code paths must never be exposed. | BR-15.3 |
| **FR-88** | All database entity primary keys and foreign keys MUST use **UUIDv7** time-ordered 128-bit identifiers (`uuidv7()` in PostgreSQL / Prisma ORM). Numeric auto-increment integer IDs (`BIGINT AUTOINCREMENT`) are strictly prohibited system-wide. | BR-10.4 |

---

### 3.13 Session Management & Centralized Audit Trail Standards

| ID | Requirement | Source |
|----|---|---|
| **FR-89** | A **SUPER_ADMIN** can forcefully terminate all active multi-device sessions of any `ADMIN`, `LANDLORD`, or `TENANT` user account (`POST /api/v1/sessions/force-logout/user/{targetUserId}`). | BR-01.5 |
| **FR-90** | A **SUPER_ADMIN** can forcefully terminate all active multi-device sessions system-wide for an entire target role scope (`POST /api/v1/sessions/force-logout/role/{targetRole}`). | BR-01.5 |
| **FR-91** | An **ADMIN** can forcefully terminate all active multi-device sessions of any `LANDLORD` or `TENANT` account. Attempting to force-logout a `SUPER_ADMIN` or another `ADMIN` MUST be rejected with `HTTP 403 FORBIDDEN` (error: `ROLE_HIERARCHY_VIOLATION`). | BR-01.5 |
| **FR-92** | Forcefully terminating user sessions immediately purges all corresponding `UserSession` records from the database and revokes associated refresh tokens. | BR-01.5, BR-10.3 |
| **FR-93** | The backend system MUST automatically record an immutable `AuditLog` entry in the database for every state-changing API operation (`POST`, `PATCH`, `PUT`, `DELETE`) and security/session event. | BR-16.1 |
| **FR-94** | `SUPER_ADMIN` and `ADMIN` roles can query, search, and paginate system audit logs (`GET /api/v1/admin/audit-logs`) filtered by `actionType`, `category`, `performedByUserId`, `targetEntityId`, `targetEntityType`, and date range. | BR-16.1 |
| **FR-95** | `AuditLog` database records are strictly insert-only and read-only. No API endpoint or database operation exists to modify (`UPDATE`) or erase (`DELETE`) audit log entries. | BR-16.2 |
| **FR-96** | Every force-logout event MUST create an `AuditLog` entry with `actionType = "FORCE_LOGOUT_USER"` or `"FORCE_LOGOUT_ROLE"`, recording the acting user, target user/role, and reason notes in `metadata`. | BR-16.1, docs/audit-logging.md |

---

### 3.14 Centralized RBAC Enforcement & Payment Transaction Update Rules

| ID | Requirement | Source |
|----|---|---|
| **FR-97** | User roles and permissions across all system resources and database entities MUST strictly conform to the permissions matrix defined in [`docs/rbac-matrix.md`](rbac-matrix.md). | BR-01.1, docs/rbac-matrix.md |
| **FR-98** | A **LANDLORD** can update a payment transaction (`PATCH /api/v1/ledgers/payments/{transactionId}`) ONLY IF it is the chronologically latest transaction for its parent ledger. Updating the latest transaction automatically recalculates the parent ledger's `amountPaid` sum and status (`UNPAID` / `PARTIALLY_PAID` / `PAID` / `OVERDUE`) and emits an `AuditLog` entry (`actionType = "UPDATE_PAYMENT_TRANSACTION"`). | BR-14.7, docs/audit-logging.md |
| **FR-99** | Attempting to update any prior (non-latest) payment transaction MUST be rejected with `HTTP 409 NON_LAST_TRANSACTION_UPDATE_RESTRICTED`. Frontend UI MUST hide or disable update controls for non-latest records. | BR-14.7 |
| **FR-100** | **SUPER_ADMIN** and **ADMIN** roles possess READ-ONLY access to landlord payment transactions. Attempting to insert, edit, or delete payment transactions by Super Admin or Admin MUST be rejected with `HTTP 403 FORBIDDEN`. | BR-14.8, docs/rbac-matrix.md |

---

### 3.15 Flexible Billing Cycles & Landlord Electricity Reconciliation Engine

| ID | Requirement | Source |
|----|---|---|
| **FR-101** | The system MUST support independent, non-calendar billing cycle start/end dates ($1..31$) for every room within a building, allowing Room Rent and Electricity cycles to have different cycle dates. | BR-02, docs/billing-and-reconciliation.md |
| **FR-102** | The system MUST support independent building-level master electricity billing cycles (`[billCycleStart, billCycleEnd]`) configured by external power supply companies (e.g., UPCL, UPPCL). | BR-03.3, docs/billing-and-reconciliation.md |
| **FR-103** | Upon settlement of a building master bill, the system MUST execute the 3-step reconciliation algorithm to aggregate actual tenant cash collected (`amountPaid`) across all overlapping room electricity ledgers. | BR-03.5, docs/billing-and-reconciliation.md |
| **FR-104** | The system MUST compare total tenant collections against the master bill paid amount and categorize the outcome into `SURPLUS` (non-zero `surplusAmount`), `DEFICIT` (non-zero `deficitAmount`), or `BREAK_EVEN`. | BR-03.3, docs/billing-and-reconciliation.md |
| **FR-105** | Unpaid or overdue tenant ledgers contribute ONLY actual collected cash towards reconciliation; late payments by tenants MUST dynamically re-evaluate and update the period's reconciliation status. | BR-03.5, docs/billing-and-reconciliation.md |
| **FR-106** | Pass-through electricity collections, surplus amounts, and deficit losses MUST be strictly segregated from Net Room Rent Profit calculations across all financial dashboards. | BR-03.1, BR-03.4 |

---

### 3.16 Tenant Electricity Payment Cutoff & Master Cycle Allocation Rules

| ID | Requirement | Source |
|----|---|---|
| **FR-107** | Tenant electricity payment allocation to building master billing cycles MUST be determined strictly by `PaymentTransaction.paymentDate`, independent of room billing cycle start/end dates. | BR-03.5, docs/billing-and-reconciliation.md |
| **FR-108** | Payments received on or before `masterCycleEndDate` MUST be allocated to the current building master billing cycle (`paymentDate <= masterCycleEndDate`). | BR-03.5, docs/billing-and-reconciliation.md |
| **FR-109** | Payments received after `masterCycleEndDate` MUST be excluded from the previous master cycle and allocated to the next building master billing cycle (`paymentDate > masterCycleEndDate`). | BR-03.5, docs/billing-and-reconciliation.md |
| **FR-110** | Early payments received before a room cycle ends MUST be allocated to the building master cycle window containing `paymentDate`. | BR-03.5, docs/billing-and-reconciliation.md |
| **FR-111** | Partial payments against a single room electricity ledger MUST be evaluated independently, allocating each `PaymentTransaction` to the master cycle corresponding to its own `paymentDate`. | BR-03.5, BR-14.7 |
| **FR-112** | Unpaid or overdue ledgers contribute $₹0.00$ to tenant collections for a master cycle until actual cash payment transactions are recorded. | BR-03.5, docs/billing-and-reconciliation.md |

---

### 3.17 Centralized Session Validation & Revocation Enforcement

| ID | Requirement | Source |
|----|---|---|
| **FR-113** | Centralized security middleware (`SessionValidationGuard`) MUST validate JWT signature/expiration AND verify active database session status (`user_sessions.isRevoked = false`) before every protected API execution. | BR-16.5, docs/error-handling.md |
| **FR-114** | Revoking a session in the database MUST immediately block subsequent API requests using tokens issued for that session, overriding remaining access token TTL. | BR-16.5, BR-01.5 |
| **FR-115** | Administrative force logout (`FORCE_LOGOUT_USER` / `FORCE_LOGOUT_ROLE`) MUST purge or mark revoked all active `user_sessions` for target user(s) with sub-500ms propagation. | BR-01.5, BR-16.2 |
| **FR-116** | API requests made using tokens of a revoked session MUST be rejected with `HTTP 401 SESSION_REVOKED` (code: `ERR-1002`) returning structured metadata (`timestamp`, `requestId`, `sessionId`, `revokedAt`). | BR-16.5, docs/error-handling.md |
| **FR-117** | Frontend PWA upon receiving `HTTP 401 SESSION_REVOKED` MUST immediately purge all local token storage, alert the user, and redirect to `/login?session_revoked=true`. | BR-16.5 |
| **FR-118** | Concurrent API calls made on a revoked session MUST all be rejected simultaneously. | BR-16.5, BR-14.1 |

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
| FR-22 – FR-29, FR-22a–c | AC-22 | Building, Asset Hierarchy & Power Supplier Switch |
| FR-30 – FR-35 | AC-30 | Tenant Lifecycle |
| FR-36 – FR-44 | AC-36 | Rent Cycle, Room Rent Ledger & Payment Transactions |
| FR-45 – FR-53, FR-50a | AC-45 | Electricity Pass-Through Model & Master Bills |
| FR-54 – FR-57 | AC-54 | Building Operating Expenses |
| FR-58 – FR-67 | AC-58 | Revenue Dashboard & P&L Analytics |
| FR-68 – FR-72 | AC-68 | Secure Document Management |
| FR-73 – FR-77 | AC-73 | Complaints & Maintenance |
| FR-78 – FR-81 | AC-78 | Input Validation & System Rules |
| FR-82 – FR-88 | AC-82 | Backend Error Handling Standards & Universal Envelopes |
| FR-89 – FR-96 | AC-89 | Session Force-Logout & Centralized Audit Trail |
| FR-97 – FR-100 | AC-97 | RBAC Matrix & Payment Transaction Update Rules |
| FR-101 – FR-106 | AC-101 | Flexible Billing Cycles & Electricity Reconciliation Engine |
| FR-107 – FR-112 | AC-107 | Tenant Electricity Payment Cutoff & Master Cycle Allocation Rules |
| FR-113 – FR-118 | AC-113 | Centralized Session Validation & Revocation Enforcement |
| FR-119 – FR-130 | AC-119 | Building Occupancy & Stacked Navigation Architecture |

---

### 3.8 Building Occupancy & Stacked Navigation Architecture

#### 3.8.1 Occupancy Derivation & Aggregation Rules

| ID | Requirement | Source |
|----|---|---|
| **FR-119** | A room's occupancy status is derived **strictly from active tenant assignments** (`Tenant.status = ACTIVE` AND `Tenant.currentRoomId = room.id`). A room with $\ge 1$ active tenant is `OCCUPIED`; a room with 0 active tenants is `VACANT`. | BR-17.1, docs/building-occupancy.md |
| **FR-120** | A floor's occupancy status is `OCCUPIED` if $\ge 1$ room on that floor is `OCCUPIED`; `VACANT` if ALL rooms on that floor are `VACANT`. The backend exposes `totalRooms`, `occupiedRooms`, `vacantRooms`, and `totalActiveTenants` for each floor. | BR-17.2, docs/building-occupancy.md |
| **FR-121** | A building's occupancy status is `OCCUPIED` if $\ge 1$ room across any floor is `OCCUPIED`; `VACANT` if ALL rooms across ALL floors are `VACANT`. Exposes `totalFloors`, `totalRooms`, `occupiedRooms`, `vacantRooms`, `occupiedFloors`, `vacantFloors`, and `totalActiveTenants`. | BR-17.2, docs/building-occupancy.md |
| **FR-122** | Historical tenants who have checked out (`status = MOVED_OUT` in `TenancyHistory`) MUST NOT keep a room marked as occupied. Historical tenant logs remain accessible for audit, billing, and room history only. | BR-17.1, docs/building-occupancy.md |

#### 3.8.2 Stacked UI Navigation & Page Hierarchy

| ID | Requirement | Source |
|----|---|---|
| **FR-123** | The **Building Details Page** MUST render a visual stacked representation of floors (`Floor Stack`) from top floor down to Ground Floor, displaying floor numbers, occupancy badges (`OCCUPIED` \| `VACANT`), and room count pills. Responsive across Mobile ($<640\text{px}$ stack), Tablet ($640\text{px}-1024\text{px}$ grid), and Desktop ($>1024\text{px}$ multi-column). | BR-17.4, docs/frontend-navigation.md |
| **FR-124** | Clicking a floor card in the Building Stack navigates to the **Floor Details Page**, displaying building header, selected floor number, floor occupancy badge, KPI metric cards, and horizontal room block cards (`[ Room 01 \| 2 Tenants ]`). | BR-17.4, docs/frontend-navigation.md |
| **FR-125** | Room block cards on the Floor Details page MUST support hover tooltips (bottom-sheet popover on mobile) displaying max capacity, base rent, and active tenant names preview (for Landlord/Admin roles). | BR-17.4, docs/frontend-navigation.md |
| **FR-126** | Clicking a room block card navigates to the **Room Details Page**, displaying building, floor, room number, occupancy status, facilities, kitchen, bathroom/toilet config, electricity ledgers, room rent ledgers, payment history, and current active tenant cards. | BR-17.4, docs/frontend-navigation.md |
| **FR-127** | Active tenants on the Room Details page are displayed as profile cards (photo, full name, check-in date). Clicking a card navigates to the Individual Tenant Details page. | BR-17.4, docs/frontend-navigation.md |

#### 3.8.3 Tenant Role Access Isolation

| ID | Requirement | Source |
|----|---|---|
| **FR-128** | A `TENANT` role user can ONLY access the Room Details page for their currently assigned room (`Tenant.currentRoomId`). | BR-17.5, docs/frontend-navigation.md |
| **FR-129** | `TENANT` users CANNOT browse other buildings, floors, rooms, or tenant details outside their assigned room. Multiple tenants assigned to the same room can view basic profile cards of active room-mates. | BR-17.5, docs/frontend-navigation.md |
| **FR-130** | Any unauthorized request by a tenant to access unassigned rooms, floors, or buildings MUST be intercepted and rejected by NestJS `TenantRoomAccessGuard` with `HTTP 403 FORBIDDEN` (`ROOM_ACCESS_DENIED`). | BR-17.5, docs/frontend-navigation.md |

---

### 3.9 Global File-Upload Policy & Storage Lifecycle Governance

#### 3.9.1 Size Enforcement & Direct Upload Rules

| ID | Requirement | Source |
|----|---|---|
| **FR-131** | All document upload features across the application (utility bills, receipts, tenant IDs, complaint photos, user avatars, reports) MUST enforce a strict **maximum file size ceiling of 5 MB ($5,242,880\text{ bytes}$)** per file and MIME allowlist (`image/jpeg`, `image/png`, `image/webp`, `application/pdf`). | BR-18.1, docs/file-storage-and-upload-policy.md |
| **FR-132** | The backend MUST act as the authoritative gatekeeper. Any file upload metadata or payload exceeding 5 MB MUST be rejected with `HTTP 413 PAYLOAD_TOO_LARGE` / `HTTP 400 BAD_REQUEST` (`error: "MAX_FILE_SIZE_EXCEEDED"`) returning a clear field-level error message. | BR-18.2, docs/file-storage-and-upload-policy.md |
| **FR-133** | The client PWA MUST pre-validate file sizes and MIME types before sending requests, displaying field-level inline validation errors below file inputs if a file exceeds 5 MB. | BR-18.2, docs/file-storage-and-upload-policy.md |
| **FR-134** | File uploads MUST use **Direct-to-R2 Presigned URLs** (`POST /api/v1/documents/presigned-upload-url`). The client transfers files directly to Cloudflare R2 object storage, bypassing API server memory and bandwidth proxying. | BR-18.3, docs/file-storage-and-upload-policy.md |

#### 3.9.2 Cost Optimization & Security Invariants

| ID | Requirement | Source |
|----|---|---|
| **FR-135** | Storage cost optimization MUST incorporate client-side WebP image compression, SHA-256 content deduplication, Cloudflare R2 auto-tiering (Standard $\rightarrow$ Infrequent Access at 90 days), and 30-day soft-deleted file purge pipelines. | BR-18.4, docs/file-storage-and-upload-policy.md |
| **FR-136** | Cost optimization mechanisms MUST NEVER weaken AES-256 GCM encryption at rest, 15-minute presigned URL access expiration, role-based access control, tenant privacy, or immutable audit logging (`AuditLog`). | BR-18.5, docs/file-storage-and-upload-policy.md |

---

### 3.10 Dual-Layer Validation Architecture & Defense-in-Depth Requirements

#### 3.10.1 Responsibilities & Defense-in-Depth Rules

| ID | Requirement | Source |
|----|---|---|
| **FR-137** | The Frontend (FE PWA) MUST perform client-side pre-validation on all forms and user inputs (required fields, format regex, password rules, range checks, file sizes) before initiating network requests to provide instant UX feedback and prevent redundant server traffic. | BR-19.1 |
| **FR-138** | The Backend (NestJS BE) MUST independently perform 100% of all required authentication, authorization/RBAC, schema validation, multi-tenant ownership, and business rule checks for every API request, acting as the sole authoritative security boundary. | BR-19.2 |
| **FR-139** | The Backend MUST operate under a **Zero-Trust Client Principle**, treating all incoming payloads as untrusted and tampered until independently verified by NestJS validation pipes and guards, protecting against direct curl/Postman API invocations. | BR-19.2, BR-19.3 |
| **FR-140** | Every incoming request MUST pass through an immutable 7-stage server execution chain: `ThrottlerGuard` $\rightarrow$ `SessionValidationGuard` $\rightarrow$ `JwtAuthGuard` $\rightarrow$ `RolesGuard` $\rightarrow$ `ResourceAccessGuard` $\rightarrow$ `ZodValidationPipe` $\rightarrow$ `DomainServiceInvariants`. | BR-19.4 |
| **FR-141** | The Backend MUST enforce strict multi-tenant resource data ownership (`LandlordBuildingGuard`, `TenantRoomAccessGuard`), rejecting unauthorized cross-tenant resource access requests with `HTTP 403 FORBIDDEN`. | BR-19.2, BR-17.5 |
| **FR-142** | Server-side validation failures MUST return standardized `HTTP 400 BAD_REQUEST` / `HTTP 422 UNPROCESSABLE_ENTITY` JSON envelopes with a `fieldErrors[]` array (`field`, `message`, `errorCode`), strictly hiding internal stack traces, DB exceptions, and SQL queries. | BR-19.5, BR-15.1 |

---

### 3.11 Frontend API Architecture, State Management & Idempotency Requirements

#### 3.11.1 Client Transport & Interceptor Rules

| ID | Requirement | Source |
|----|---|---|
| **FR-143** | All frontend API calls MUST route through a centralized Axios client instance (`lib/api/api-client.ts`) that enforces standard headers, authentication Bearer tokens, CSRF protection, and timeout limits (`15s`). | BR-20.1, architecture.md |
| **FR-144** | Upon receiving an `HTTP 401 UNAUTHORIZED` (`TOKEN_EXPIRED`) error, the response interceptor MUST execute a single-refresh mutex (`isRefreshing` flag). Parallel failing requests MUST be queued in `failedQueue[]`, resolved upon refresh completion, and retried without component awareness. | BR-20.2, api-contracts.md §0.7.2 |
| **FR-145** | Upon receiving an `HTTP 401 UNAUTHORIZED` with `error: "SESSION_REVOKED"` or `errorCode: "ERR-1002"`, the interceptor MUST bypass token refresh, purge client auth state (`useAuthStore`), cancel pending queries, redirect to `/login?reason=session_revoked`, and alert the user. | BR-20.3, api-contracts.md §0.7.3 |
| **FR-146** | Server-state management MUST use **TanStack Query v5**, governing caching (`staleTime: 5m`, `gcTime: 15m`), loading/error states, request deduplication, and automatic target query invalidation upon mutation. | BR-20.4, architecture.md |
| **FR-147** | All non-idempotent financial mutations (`POST`, `PUT`, `PATCH` for payments, ledgers, expenses) MUST inject a unique UUIDv7 `Idempotency-Key` header (`Idempotency-Key: idemp_<uuidv7>`). | BR-20.5, api-contracts.md §0.7.4 |
| **FR-148** | Automatic client-side HTTP retries MUST be strictly disabled for non-idempotent methods (`POST`, `PUT`, `PATCH`) unless an explicit `Idempotency-Key` header is attached, preventing duplicate financial transactions. | BR-20.5, api-contracts.md §0.7.4 |

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

**Stage 02 Confirmed & Locked ✅** (Approved by User)

**Next:** Stage 03 — Non-Functional Requirements → [`docs/03-non-functional-requirements.md`](03-non-functional-requirements.md)
