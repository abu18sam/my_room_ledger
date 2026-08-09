# Stage 02 — Acceptance Criteria

**Status:** In Progress  
**Date locked:** —  
**Upstream:** [02-functional-requirements.md](02-functional-requirements.md)  
**Downstream:** Stages 06 (API Design), 13–14 (Implementation & Tests)  
**Workflow tracker:** [00-engineering-workflow.md](00-engineering-workflow.md)

Each criterion is independently testable. Integration tests (Stage 14) must cite AC IDs. All ACs are binding — any implementation deviating from an AC is a defect.

---

## AC-01 — Login, Token Issuance & RBAC

**Requirements:** FR-01, FR-02, FR-03, FR-04, FR-05, FR-06, FR-08

| ID | Criterion |
|----|---|
| **AC-01.1** | Given valid email and password, login returns HTTP 200 with a signed JWT access token. A `RefreshToken` record is created in the database and delivered as an `HttpOnly; Secure; SameSite=Strict` cookie. |
| **AC-01.2** | Given wrong password or unknown email, login returns HTTP 401 `INVALID_CREDENTIALS`; no token is issued and no cookie is set. |
| **AC-01.3** | Given a valid JWT on a protected request, the request is authorised; the acting user identity and role resolve from the token payload. |
| **AC-01.4** | Given a missing JWT on a protected request, the API returns HTTP 401 `UNAUTHORIZED`. |
| **AC-01.5** | Given an expired JWT access token, the API returns HTTP 401 `TOKEN_EXPIRED`. |
| **AC-01.6** | Given a tampered or malformed JWT, the API returns HTTP 401 `INVALID_TOKEN`. |
| **AC-01.7** | Accessing a resource outside the user's role scope returns HTTP 403 `FORBIDDEN` regardless of a valid JWT. |
| **AC-01.8** | After 5 failed login attempts from the same IP within 15 minutes, the endpoint returns HTTP 429 `TOO_MANY_REQUESTS`. Subsequent attempts within that window are also rejected. |
| **AC-01.9** | A request to `GET /api/v1/meta/country-codes` returns all active country codes with dialCode, flagEmoji, minLength, maxLength, and phoneRegexPattern. India (`+91`) is flagged `isDefault: true`. |
| **AC-01.10** | Login with valid phone number + country code `+91` + correct password succeeds and returns HTTP 200 with a signed JWT access token. |
| **AC-01.11** | Login with a phone number that fails the regex pattern stored in DB for the selected dial code returns HTTP 400 `VALIDATION_ERROR` with field `phoneNumber`. |
| **AC-01.12** | Login with an invalid email format (failing RFC 5322 regex) returns HTTP 400 `VALIDATION_ERROR` with field `email`. |
| **AC-01.13** | The backend normalizes all incoming phone login credentials to E.164 format (`+<dialCode><phoneNumber>`) before executing database queries. |

---

## AC-01-sessions — Multi-Device Session Management

**Requirements:** FR-01a, FR-01b, FR-01c, FR-01d

| ID | Criterion |
|----|---|
| **AC-01-S.1** | A user can log in from Device A and Device B simultaneously; both sessions return valid, distinct access tokens and have distinct `RefreshToken` records in the database. |
| **AC-01-S.2** | The refresh token cookie from Device A is not accepted as valid when used from Device B's session context — each session's token is scoped to its own record. |
| **AC-01-S.3** | A user can retrieve a list of their active sessions showing device hint (e.g., browser/OS) and last-used timestamp. |
| **AC-01-S.4** | A user can terminate a specific session (not the current one) from the active sessions list; that session's `RefreshToken` record is deleted and its token is rejected on next use. |

---

## AC-01-logout — Logout & Session Isolation

**Requirements:** FR-07, FR-07a, FR-07b

| ID | Criterion |
|----|---|
| **AC-01-L.1** | On logout from Device A, the server deletes **only** the `RefreshToken` record associated with Device A's session. |
| **AC-01-L.2** | After logout from Device A, Device B's session remains valid — its refresh token is still accepted and a new access token can be obtained. |
| **AC-01-L.3** | After logout, the client clears: the access token (in-memory), the refresh token cookie, cached user profile data, and any local/session storage entries. |
| **AC-01-L.4** | Attempting to use the refresh token from a logged-out session to obtain a new access token returns HTTP 401 `INVALID_TOKEN`. |

---

## AC-P00 — Change Password (Logged-in User)

**Requirements:** FR-P00a, FR-P00b

| ID | Criterion |
|----|---|
| **AC-P00.1** | A logged-in user can change password by providing Old Password, New Password, and Confirm New Password. Passing an incorrect Old Password returns HTTP 400 `INVALID_CURRENT_PASSWORD`. |
| **AC-P00.2** | All password input fields in the frontend UI (Login, Change Password, Reset Password) render a mask toggle icon (show/hide eye icon) to toggle character visibility. |
| **AC-P00.3** | Mismatch between New Password and Confirm New Password is caught on frontend (validation message) and rejected on backend with HTTP 400 `PASSWORD_MISMATCH`. |

---

## AC-P01 — Forced Password Change Flow

**Requirements:** FR-P01, FR-P02

| ID | Criterion |
|----|---|
| **AC-P01.1** | Accounts flagged `mustChangePassword = true` (newly created or reset via temp password) return HTTP 200 with `mustChangePassword: true` on login. |
| **AC-P01.2** | Frontend immediately redirects users with `mustChangePassword: true` to the "Create New Password" page displaying their Name (and Email if available). No other navigation is allowed. |
| **AC-P01.3** | Any API request to protected routes while `mustChangePassword = true` (except `POST /api/v1/auth/change-password`) returns HTTP 403 `MUST_CHANGE_PASSWORD`. |

---

## AC-P05 — Forgot Password & Recovery Flows

**Requirements:** FR-P05, FR-P06, FR-P07, FR-P08, FR-P09, FR-P09a, FR-P10, FR-P11, FR-P12

| ID | Criterion |
|----|---|
| **AC-P05.1** | Submitting a "Forgot Password" request from login page creates a `PENDING` request and triggers a real-time notification in the Admin/Super Admin frontend header bell panel. |
| **AC-P05.2** | **Email Flow (Registered Email)**: Upon Admin approval, system sends an email with a signed reset link valid for **15 minutes**. |
| **AC-P05.3** | Clicking the reset link redirects to `/reset-password?token=...` displaying the user's **Name** and **Email**. User enters New Password + Confirm New Password. |
| **AC-P05.4** | On successful reset via link: reset link token expires immediately (single-use), **ALL active sessions on all devices are purged**, and user is redirected to Login page. |
| **AC-P05.5** | Attempting to use an expired (> 15 mins) or already-used reset link token returns HTTP 400 `TOKEN_EXPIRED` or `TOKEN_ALREADY_USED`. |
| **AC-P05.6** | **No-Email Fallback Flow**: Given a user without a registered email (or direct Admin reset), Admin approval generates a **temporary password valid for 30 minutes** displayed in an Admin single-view modal for manual share. |
| **AC-P05.7** | Target user logging in with temporary password is automatically redirected to the "Create New Password" page (`mustChangePassword = true`) showing their Name (and Email if available). |
| **AC-P05.8** | Setting new password successfully sets `mustChangePassword = false`, invalidates the temporary password, purges all active sessions across all devices, and redirects user to Login page. |

---

## AC-P13 — Password Security Constraints

**Requirements:** FR-P13, FR-P14, FR-P15

| ID | Criterion |
|----|---|
| **AC-P13.1** | No plaintext password appears in any database column, API response, or application log at any point. |
| **AC-P13.2** | The password column in the database stores a bcrypt hash (12 salt rounds). Verified by checking the hash prefix (`$2b$12$`). |
| **AC-P13.3** | A password that fails complexity rules (< 8 chars, missing uppercase, lowercase, digit, or special character) returns HTTP 400 `VALIDATION_ERROR` with specific field feedback. |
| **AC-P13.4** | A user attempting to set a new password that matches any of their last 3 password hashes receives HTTP 400 `PASSWORD_REUSE_NOT_ALLOWED`. |

---


## AC-09 — Super Admin Management

**Requirements:** FR-09, FR-10, FR-11, FR-12, FR-13, FR-14

| ID | Criterion |
|----|---|
| **AC-09.1** | A Super Admin can create a new Admin user with full name, email, phone, and password; the response confirms the new admin's ID and role. |
| **AC-09.2** | Creating an Admin with a duplicate email returns HTTP 409 `CONFLICT`. |
| **AC-09.3** | A Super Admin can update an Admin's full name and phone number successfully. |
| **AC-09.4** | A Super Admin can deactivate an Admin account; the deactivated admin cannot log in. |
| **AC-09.5** | A Super Admin can list all Admins in the system. |
| **AC-09.6** | A Super Admin can view system-wide metrics: total landlords count, total buildings count, total active tenants, total rent collected. |
| **AC-09.7** | Any attempt to create a SUPER_ADMIN by a non-Super-Admin role (Admin, Landlord, Tenant) returns HTTP 403 `FORBIDDEN`. |
| **AC-09.8** | A Super Admin creating another Super Admin succeeds only when the requesting token role is `SUPER_ADMIN`. |

---

## AC-15 — Admin Management

**Requirements:** FR-15, FR-16, FR-17, FR-18, FR-19, FR-20, FR-21

| ID | Criterion |
|----|---|
| **AC-15.1** | An Admin can create a Landlord account with full name, email, phone, and password; the response confirms the landlord's ID. |
| **AC-15.2** | Creating a Landlord with a duplicate email returns HTTP 409 `CONFLICT`. |
| **AC-15.3** | An Admin can update a Landlord's profile details (full name, phone). |
| **AC-15.4** | An Admin can deactivate a Landlord account; the deactivated landlord cannot log in. |
| **AC-15.5** | An Admin can list all Landlords with their building counts. |
| **AC-15.6** | An Admin can view all buildings and tenants belonging to a specific Landlord. |
| **AC-15.7** | An Admin can view per-landlord financial metrics (Level 3 aggregation). |
| **AC-15.8** | Any attempt by an Admin to create, update, or delete a Super Admin returns HTTP 403 `FORBIDDEN`. |

---

## AC-22 — Building & Asset Hierarchy

**Requirements:** FR-22, FR-22a, FR-22b, FR-22c, FR-23, FR-24, FR-25, FR-26, FR-27, FR-28, FR-29

| ID | Criterion |
|----|---|
| **AC-22.1** | A Landlord can create a Building with name, address, city, state, pincode, selected `powerCompanyId` (Power Supply Company FK), and mandatory active `connectionNumber`. |
| **AC-22.2** | A Landlord can update their own Building's name, address, city, state, or pincode. |
| **AC-22.3** | A Landlord can add a Floor to their Building with a floor number and label. |
| **AC-22.4** | A Landlord can add a Room to a Floor specifying label (`Room XY` format), max occupancy, monthly rent, and bathroom type (`PRIVATE_ATTACHED` or `SHARED_FLOOR`). |
| **AC-22.5** | A Landlord can add Shared Bathrooms (`Bath XY`) and Shared Toilets (`Toilet XY`) to a Floor. |
| **AC-22.6** | A single floor can have a mix of private-attached rooms and shared facilities simultaneously. |
| **AC-22.7** | Attempting to access or modify another landlord's building returns HTTP 403 `FORBIDDEN`. |
| **AC-22.8** | A Landlord attempting to switch a building's power supply company (`POST /api/v1/buildings/{id}/switch-power-supplier`) while open (`UNPAID` or `OVERDUE`) master bills exist for the current supplier is rejected with `HTTP 409 PENDING_SUPPLIER_BILLS_EXIST`. Response `metadata.pendingBills[]` lists all open bill IDs, amounts, and statuses. |
| **AC-22.9** | A Landlord executing a power supplier switch when all master bills for the current supplier are `PAID` succeeds (HTTP 200 OK): current connection record is set to `TERMINATED` with `endDate`, new `BuildingPowerConnection` is created with `status = ACTIVE`, and `Building.powerCompanyId` + `Building.connectionNumber` are updated. |
| **AC-22.10** | `GET /api/v1/buildings/{buildingId}/power-supplier-history` returns the complete array of active and terminated connection history records for the building, ordered descending by `startDate`. |
| **AC-22.11** | A Landlord can retrieve the full hierarchy: building → floors → rooms → shared facilities. |

---

## AC-30 — Tenant Lifecycle

**Requirements:** FR-30, FR-31, FR-32, FR-33, FR-34, FR-35

| ID | Criterion |
|----|---|
| **AC-30.1** | A Landlord can onboard a Tenant to a specific room with name, phone, email, move-in date, and monthly rent. |
| **AC-30.2** | A Landlord can check out a Tenant, recording the move-out date. The tenancy record is preserved in history. |
| **AC-30.3** | Multiple tenants can occupy the same room across different time periods — each tenancy is a separate history record. |
| **AC-30.4** | A Landlord can upload a Government ID document for a tenant; the API confirms the document ID (not the raw file URL). |
| **AC-30.5** | The uploaded Government ID is encrypted (AES-256 GCM) before storage — direct R2 bucket access returns nothing. |
| **AC-30.6** | A Tenant can retrieve their own payment history (rent + electricity) — response is read-only. |
| **AC-30.7** | A Tenant attempting to access another tenant's data or any revenue endpoint receives HTTP 403 `FORBIDDEN`. |

---

## AC-36 — Rent Cycle & Room Rent Ledger

**Requirements:** FR-36, FR-37, FR-38, FR-39, FR-40, FR-41, FR-41a, FR-41b, FR-41c, FR-42, FR-43, FR-44

| ID | Criterion |
|----|---|
| **AC-36.1** | A Landlord can configure a Rent Cycle with a start day between 1 and 28 (inclusive). |
| **AC-36.2** | Rent cycle start is not restricted to the 1st of the month — any valid day is accepted. |
| **AC-36.3** | A Landlord can generate a Billing Cycle for a room with valid start and end dates. |
| **AC-36.4** | On billing cycle generation, a Tenancy Snapshot is created capturing all active tenants at that point in time. |
| **AC-36.5** | Checking out a tenant after a billing cycle is generated does not alter the Tenancy Snapshot on that historical cycle. |
| **AC-36.6** | A generated Billing Cycle produces a Room Rent Ledger entry with status `UNPAID`. |
| **AC-36.7** | A Landlord can transition Rent Ledger status: `UNPAID` → `PARTIALLY_PAID` → `PAID` or `OVERDUE`. |
| **AC-36.8** | Updating a Rent Ledger status does not change the Electricity Ledger for the same cycle — verified by checking both records before and after. |
| **AC-36.9** | A Landlord can record payment method (`UPI`, `CASH`, `BANK_TRANSFER`) and transaction reference on a PaymentTransaction entry. |
| **AC-36.10** | A Tenant can retrieve their own Rent Ledger entries and associated payment transactions; all data is read-only. |
| **AC-36.11** | Recording a payment of ₹1,000 against a ₹2,500 rent ledger transitions status to `PARTIALLY_PAID` and sets `amountPaid = 1000.00`. |
| **AC-36.12** | Recording a second payment of ₹1,500 against the same ledger transitions status to `PAID`, sets `amountPaid = 2500.00`, and populates `paidDate` with current UTC timestamp. |
| **AC-36.13** | Attempting to record a payment where `amountPaid ≤ 0` returns HTTP 400 `VALIDATION_ERROR`. |
| **AC-36.14** | If `paymentDate` is omitted from the payment request body, the system defaults it to the UTC timestamp of the API call (`recordedAt`). |
| **AC-36.15** | Given `cycleEndDate = 2026-08-05`, a ledger with status `UNPAID` or `PARTIALLY_PAID` transitions to `OVERDUE` when `currentDate > 2026-08-05`. |
| **AC-36.16** | An `OVERDUE` ledger that subsequently receives a full payment (`amountPaid ≥ amount`) transitions to `PAID`. |
| **AC-36.17** | `GET /api/v1/rooms/{roomId}/outstanding-balance` returns a `totalOutstanding` field equal to `SUM(amount − amountPaid)` across all non-`PAID` ledger entries, plus a per-cycle breakdown. |
| **AC-36.18** | `GET /api/v1/ledgers/room-rent/{ledgerId}/payments` returns all payment transactions for the ledger in ascending `paymentDate` order. |

---

## AC-50 — Power Supply Companies Management

**Requirements:** FR-50a, FR-50b, FR-50c, FR-50d, FR-50e

| ID | Criterion |
|----|---|
| **AC-50.1** | `GET /api/v1/meta/power-companies` returns all active power companies pre-seeded in DB (UPCL, UPPCL, Reliance, Adani, TPCL, NTPC). |
| **AC-50.2** | An Admin or Super Admin can create a new power company payload (`name`, `status`), returning HTTP 201 `CREATED`. |
| **AC-50.3** | An Admin or Super Admin can update an existing power company's name or toggle status between `ACTIVE` and `INACTIVE`. |
| **AC-50.4** | Deleting or deactivating an unused power company (0 linked buildings) succeeds with HTTP 200/240. |
| **AC-50.5** | Attempting to delete or deactivate a power company linked to $\ge 1$ building is rejected with `HTTP 409 COMPANY_IN_USE`. Response `metadata.affectedBuildings[]` contains the building names and landlord details for all linked buildings. |

---

## AC-45 — Electricity Pass-Through Model & Rate Differential

**Requirements:** FR-45, FR-46, FR-47, FR-48, FR-48a, FR-49, FR-50, FR-51, FR-52, FR-52a, FR-52b, FR-52c, FR-52d, FR-53

| ID | Criterion |
|----|---|
| **AC-45.1** | A Landlord can create an Electricity Ledger entry for a room with units consumed, landlord rate per unit (e.g. ₹8/unit), and billing cycle dates. |
| **AC-45.2** | The system auto-calculates `Room Bill = Units Consumed × Landlord Rate per Unit`; client-supplied total is overwritten. |
| **AC-45.3** | A generated Electricity Ledger entry has initial status `UNPAID`. |
| **AC-45.4** | A Landlord can transition Electricity Ledger status: `UNPAID` → `PARTIALLY_PAID` → `PAID` or `OVERDUE`. |
| **AC-45.5** | Updating an Electricity Ledger status does not change the Room Rent Ledger for the same cycle. |
| **AC-45.6** | A Landlord can create a Supplier Master Bill entry linked to a selected Power Supply Company, connection number, cycle dates, total master bill amount, due date, optional bill serial number, optional invoice date, and optional total units consumed (kWh). |
| **AC-45.6a** | Entering a duplicate `billSerialNumber` for the same `powerCompanyId` is rejected with `HTTP 409 DUPLICATE_ENTRY`. |
| **AC-45.6b** | Entering a second master bill for the same `buildingId` + `powerCompanyId` covering an identical `billCycleStart` + `billCycleEnd` range is rejected with `HTTP 409 DUPLICATE_ENTRY`. |
| **AC-45.6c** | Marking a `SupplierMasterBill` as `PAID` allows specifying `paymentMode` (`NEFT`, `UPI`, `CHEQUE`, `ONLINE_PORTAL`, `CASH`) and `paymentReference` (UTR/cheque ID), which are persisted and returned in GET responses. |
| **AC-45.7** | A Landlord can upload a digital power bill (PDF/Image) linked to a Supplier Master Bill; file is encrypted before R2 storage. |
| **AC-45.8** | Electricity totals collected from tenants are strictly excluded from Net Rental Profit calculations. |
| **AC-45.9** | A Tenant can retrieve their own Electricity Ledger entries and payment transactions; read-only. |
| **AC-45.10** | Given tenant electricity collections $C$ and power supplier master bill amount $B$ for a billing period, system computes `Variance = C - B`. |
| **AC-45.11** | Given `Variance > 0` (e.g., Collection ₹4,500 vs Bill ₹3,500 $\rightarrow$ Surplus ₹1,000), status is `SURPLUS` ("Extra Savings"), reported separately from Net Rental Profit. |
| **AC-45.12** | Given `Variance < 0` (e.g., Collection ₹4,000 vs Bill ₹5,000 $\rightarrow$ Deficit ₹1,000), status is `DEFICIT` ("Landlord Out-of-Pocket Contribution"), reported separately from Net Rental Profit. Given `Variance == 0`, status is `BALANCED`. |
| **AC-45.13** | Unmetered common electricity consumption (common lighting, submersible water motors) is recorded as a `WATER_MOTOR_ELECTRICITY` or `COMMON_ELECTRICITY` operating expense. |
| **AC-45.14** | Recording a partial electricity payment of ₹800 against a ₹1,200 bill transitions status to `PARTIALLY_PAID` and sets `amountPaid = 800.00`. |
| **AC-45.15** | Recording a subsequent ₹400 electricity payment transitions status to `PAID` and sets `amountPaid = 1200.00`, and populates `paidDate`. |
| **AC-45.16** | A Landlord can mark a `SupplierMasterBill` as `PAID`; if `paidDate` is omitted from the request, the system records the current UTC timestamp. |
| **AC-45.17** | Attempting to re-mark an already-`PAID` `SupplierMasterBill` as `PAID` is rejected with HTTP 409 `CONFLICT` (idempotency guard). |

---

## AC-54 — Building Operating Expenses

**Requirements:** FR-54, FR-55, FR-56, FR-57

| ID | Criterion |
|----|---|
| **AC-54.1** | A Landlord can log an expense with category, title, amount, and expense date. |
| **AC-54.2** | Expense category must be one of: `WATER_BILL`, `COMMON_ELECTRICITY`, `WATER_MOTOR_ELECTRICITY`, `MAINTENANCE`, `REPAIRS`, `SECURITY`, `CLEANING`, `PROPERTY_TAX`, `MISCELLANEOUS`. Invalid category returns HTTP 400 `VALIDATION_ERROR`. |
| **AC-54.3** | A Landlord can retrieve all expenses for a building, optionally filtered by category and/or date range. |
| **AC-54.4** | Building expenses are included in the Net Profit calculation: `Net Profit = Rent Collected − Operating Expenses`. |

---

## AC-58 — Revenue Dashboard & P&L Analytics

**Requirements:** FR-58, FR-59, FR-60, FR-61, FR-62, FR-63, FR-64, FR-65, FR-66, FR-67

| ID | Criterion |
|----|---|
| **AC-58.1** | A Landlord can retrieve their own Revenue Dashboard data for a single building. |
| **AC-58.2** | A Landlord can retrieve a Portfolio-level rollup across all their buildings. |
| **AC-58.3** | Dashboard response includes explicit separate fields for: total rent collected, operating expenses, **net rental profit**, total tenant electricity collected, total supplier utility bill, **electricity surplus (extra savings)**, **electricity deficit (landlord out-of-pocket)**, reconciliation status (`SURPLUS` \| `DEFICIT` \| `BALANCED`), pending dues, and occupancy rate. |
| **AC-58.4** | Dashboard results filtered by Monthly period return data for only that calendar month. |
| **AC-58.5** | Dashboard results filtered by Indian Financial Year return data for April 1 of year Y through March 31 of year Y+1. |
| **AC-58.6** | Dashboard results filtered by custom date range return data within those exact dates (inclusive). |
| **AC-58.7** | Historical trend data (multi-year) is available for revenue, expenses, occupancy, and separate electricity surplus/deficit trend lines. |
| **AC-58.8** | A Tenant attempting to access any dashboard endpoint receives HTTP 403 `FORBIDDEN` — no data is returned. |
| **AC-58.9** | An Admin can retrieve Level 3 (per-landlord) aggregated data for any landlord in the system. |
| **AC-58.10** | A Super Admin can retrieve Level 4 (system-wide) platform metrics. |

---

## AC-68 — Secure Document Management

**Requirements:** FR-68, FR-69, FR-70, FR-71, FR-72

| ID | Criterion |
|----|---|
| **AC-68.1** | A file upload request to `POST /api/v1/files/upload` succeeds with a valid authorised JWT and a supported file type ≤ 10 MB. Response returns `documentId` only — never a direct R2 URL. |
| **AC-68.2** | Uploading a file type outside `.pdf`, `.jpg`, `.jpeg`, `.png` returns HTTP 400 `VALIDATION_ERROR`. |
| **AC-68.3** | Uploading a file exceeding 10 MB returns HTTP 400 `VALIDATION_ERROR`. |
| **AC-68.4** | The uploaded file in Cloudflare R2 is not readable as plain binary — it is encrypted ciphertext. |
| **AC-68.5** | `GET /api/v1/files/:id/signed-url` returns a short-lived signed URL (valid ≤ 15 minutes) for authorised users. |
| **AC-68.6** | Accessing a file belonging to another user's resource returns HTTP 403 `FORBIDDEN`. |
| **AC-68.7** | A signed URL that has expired returns an error when accessed — the file is not served. |

---

## AC-73 — Complaints & Maintenance

**Requirements:** FR-73, FR-74, FR-75, FR-76, FR-77

| ID | Criterion |
|----|---|
| **AC-73.1** | A Tenant can submit a complaint with valid category, severity, title, and description. Response includes the complaint ID and initial status `OPEN`. |
| **AC-73.2** | Submitting a complaint with an invalid category or severity returns HTTP 400 `VALIDATION_ERROR`. |
| **AC-73.3** | A Landlord can list all complaints for their own buildings. |
| **AC-73.4** | A Landlord can update complaint status: `OPEN` → `IN_PROGRESS` → `RESOLVED` or `REJECTED`, and add resolution notes. |
| **AC-73.5** | A Landlord cannot view or manage complaints for buildings they do not own. |
| **AC-73.6** | A Tenant can retrieve the status history of their own complaints. |

---

## AC-78 — Input Validation & System Rules

**Requirements:** FR-78, FR-79, FR-80, FR-81

| ID | Criterion |
|----|---|
| **AC-78.1** | Any request with a missing required body field returns HTTP 400 `VALIDATION_ERROR` with the exact field name in `details`. |
| **AC-78.2** | Any request with an invalid enum value (e.g., wrong role, wrong payment status) returns HTTP 400 `VALIDATION_ERROR`. |
| **AC-78.3** | Any request with an invalid date format (non-ISO 8601) returns HTTP 400 `VALIDATION_ERROR`. |
| **AC-78.4** | String fields with leading/trailing whitespace are trimmed before processing — stored values contain no padding. |
| **AC-78.5** | An invalid email format in any request body returns HTTP 400 `VALIDATION_ERROR` with field `email`. |
| **AC-78.6** | A financial date range query where `startDate` is after `endDate` returns HTTP 400 `VALIDATION_ERROR`. |

---

## AC-82 — Backend Error Handling Standards & Universal Envelopes

**Requirements:** FR-82, FR-83, FR-84, FR-85, FR-86, FR-87, FR-88

| ID | Criterion |
|----|---|
| **AC-82.1** | Every non-2xx API response contains exactly the top-level keys: `statusCode`, `error`, `message`, `metadata` (or `details` for validation errors). No unhandled exception leaks internal paths or stack traces. |
| **AC-82.2** | Validation failures (`HTTP 400 VALIDATION_ERROR`) include a `details` array containing `{ field, message }` objects for every invalid payload field. |
| **AC-82.3** | A phone number failing DB regex for the selected dial code returns `HTTP 400 VALIDATION_ERROR` with `details[].field = "phoneNumber"` and an actionable message specifying the dial code format required. |
| **AC-82.4** | A password failing complexity rules (< 8 chars, missing uppercase, missing digit, or missing special character) returns `HTTP 400 VALIDATION_ERROR` with `details[].field = "password"` specifying the unmet requirement. |
| **AC-82.5** | `DELETE /api/v1/admin/power-companies/{id}` on an in-use power company returns `HTTP 409 COMPANY_IN_USE`; `metadata.linkedBuildingCount` matches DB count; `metadata.affectedBuildings` lists each building ID, name, and landlord email. |
| **AC-82.6** | Deleting an unused power company (0 linked buildings) succeeds with `HTTP 200 OK`. |
| **AC-82.7** | `POST /api/v1/ledgers/room-rent/{id}/payments` where `amountPaid` > remaining due returns `HTTP 400 PAYMENT_EXCEEDS_BALANCE` with `metadata.remainingBalance` and `metadata.attemptedPayment`. |
| **AC-82.8** | Requesting a resource belonging to another landlord returns `HTTP 403 FORBIDDEN` — access is denied regardless of whether the resource exists. |
| **AC-82.9** | Requesting a non-existent resource ID returns `HTTP 404 NOT_FOUND` with `metadata.resourceId`. |
| **AC-82.10** | Registering a user with an existing email or phone returns `HTTP 409 DUPLICATE_ENTRY` with `metadata.field`. |
| **AC-82.11** | Unhandled 500 server errors return `HTTP 500 INTERNAL_SERVER_ERROR` with generic message "An unexpected internal server error occurred. Please contact support."; no stack trace or SQL text is present. |
| **AC-82.12** | Every entity primary key and foreign key generated and returned by the API is a valid 36-character hyphenated RFC 4122 UUID string (`8-4-4-4-12` hex). Numeric integer IDs passed in request payloads return `HTTP 400 VALIDATION_ERROR`. |

---

## AC-89 — Session Force-Logout & Centralized Audit Trail

**Requirements:** FR-89, FR-90, FR-91, FR-92, FR-93, FR-94, FR-95, FR-96

| ID | Criterion |
|----|---|
| **AC-89.1** | `POST /api/v1/sessions/force-logout/user/{targetUserId}` executed by a `SUPER_ADMIN` on an Admin, Landlord, or Tenant account returns HTTP 200 `OK`, purges all active `UserSession` records for that user, and creates an `AuditLog` entry with `actionType = "FORCE_LOGOUT_USER"`. |
| **AC-89.2** | `POST /api/v1/sessions/force-logout/user/{targetUserId}` executed by an `ADMIN` on a Landlord or Tenant account succeeds (HTTP 200) and creates an `AuditLog` entry. |
| **AC-89.3** | `POST /api/v1/sessions/force-logout/user/{targetUserId}` executed by an `ADMIN` targeting a `SUPER_ADMIN` or another `ADMIN` account is rejected with `HTTP 403 FORBIDDEN` (error code: `ROLE_HIERARCHY_VIOLATION`). |
| **AC-89.4** | `POST /api/v1/sessions/force-logout/role/{targetRole}` executed by a `SUPER_ADMIN` forcefully terminates all active sessions for all users of that role scope and creates an `AuditLog` entry (`actionType = "FORCE_LOGOUT_ROLE"`). |
| **AC-89.5** | Every state-changing API call (`POST`, `PATCH`, `PUT`, `DELETE`) automatically creates an `AuditLog` row with non-null `actionType`, `category`, `performedByUserId`, `performedByUserRole`, `ipAddress`, `userAgent`, and `metadata`. |
| **AC-89.6** | `GET /api/v1/admin/audit-logs` executed by `SUPER_ADMIN` or `ADMIN` returns paginated audit records and supports filtering by `actionType`, `category`, `performedByUserId`, `targetEntityId`, `targetEntityType`, and date range. |
| **AC-89.7** | Any HTTP request or database query attempting to `UPDATE` or `DELETE` records in `audit_logs` is rejected with an error — audit trail remains immutable. |

---

## AC-97 — RBAC Matrix & Payment Transaction Update Rules

**Requirements:** FR-97, FR-98, FR-99, FR-100

| ID | Criterion |
|----|---|
| **AC-97.1** | `PATCH /api/v1/ledgers/payments/{transactionId}` executed by a `LANDLORD` on the chronologically latest payment transaction for their room's ledger succeeds (HTTP 200); updates transaction fields (`amountPaid`, `paymentDate`, `paymentMethod`, `transactionReference`, `notes`), automatically recalculates parent ledger `amountPaid` and `status`, and creates an `AuditLog` entry (`actionType = "UPDATE_PAYMENT_TRANSACTION"`). |
| **AC-97.2** | `PATCH /api/v1/ledgers/payments/{transactionId}` executed by a `LANDLORD` on a non-latest (older) payment transaction for a ledger is rejected with `HTTP 409 NON_LAST_TRANSACTION_UPDATE_RESTRICTED`; parent ledger balance and transaction record remain completely unchanged. |
| **AC-97.3** | Any attempt by a `SUPER_ADMIN` or `ADMIN` to invoke payment transaction creation (`POST`), update (`PATCH`), or deletion (`DELETE`) is rejected with `HTTP 403 FORBIDDEN`. |
| **AC-97.4** | System authorization behavior strictly enforces the entity permissions matrix defined in `docs/rbac-matrix.md`; unauthorized cross-role or cross-tenant data mutation attempts return `HTTP 403 FORBIDDEN`. |

---

## AC-101 — Flexible Billing Cycles & Electricity Reconciliation Engine

**Requirements:** FR-101, FR-102, FR-103, FR-104, FR-105, FR-106

| ID | Criterion |
|----|---|
| **AC-101.1** | Rooms within the same building can be created and configured with independent, non-calendar billing cycles starting on any date ($1..31$), with Room Rent and Electricity cycles operating on distinct start/end dates. |
| **AC-101.2** | Settling a building's `SupplierMasterBill` executes the 3-step reconciliation engine, correctly aggregating actual tenant collections (`PaymentTransaction.amountPaid`) across all overlapping room electricity ledgers. |
| **AC-101.3** | When aggregated tenant collections exceed supplier master bill amount paid, `SupplierMasterBill` records `status = SURPLUS` and non-zero `surplusAmount`; when collections are less, it records `status = DEFICIT` and non-zero `deficitAmount`. |
| **AC-101.4** | Unpaid or overdue tenant ledgers contribute ONLY actual collected cash (`amountPaid`) toward current master bill reconciliation; recording subsequent tenant payments automatically recalculates the period's reconciliation status. |
| **AC-101.5** | Revenue and P&L dashboards strictly exclude electricity pass-through collections, surplus amounts, and deficit losses from Net Room Rent Profit calculations. |

---

## AC-107 — Tenant Electricity Payment Cutoff & Master Cycle Allocation Rules

**Requirements:** FR-107, FR-108, FR-109, FR-110, FR-111, FR-112

| ID | Criterion |
|----|---|
| **AC-107.1** | A tenant electricity payment received on or before `masterCycleEndDate` (e.g. 21 Mar for a 24 Feb → 24 Mar cycle) is allocated to the current building master cycle reconciliation. |
| **AC-107.2** | A tenant electricity payment received after `masterCycleEndDate` (e.g. 25 Mar for a 24 Feb → 24 Mar cycle) is strictly excluded from the previous cycle and allocated to the next building master cycle (24 Mar → 24 Apr). |
| **AC-107.3** | Early payments made before room cycle end date but within the active master cycle window (e.g. 15 Feb) are allocated to the master cycle window containing `paymentDate`. |
| **AC-107.4** | Multi-cycle delayed payments (e.g. paid 28 Apr for a Feb room ledger) are assigned strictly to the master cycle window containing `paymentDate` (24 Apr → 24 May). Past settled cycles are never re-opened. |
| **AC-107.5** | Multiple partial payments against a single room electricity ledger on different dates are split independently into their respective master cycle windows based on each transaction's `paymentDate`. |
| **AC-107.6** | Unpaid or overdue ledgers contribute $₹0.00$ to tenant collections for a master cycle until actual cash payment transactions are recorded. |

---

## Gate

**Awaiting user confirmation.**

**Next:** Stage 03 — Non-Functional Requirements → `docs/03-non-functional-requirements.md`
