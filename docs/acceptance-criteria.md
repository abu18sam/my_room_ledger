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

## AC-P01 — First-Login Mandatory Password Change

**Requirements:** FR-P01, FR-P02, FR-P03, FR-P04

| ID | Criterion |
|----|---|
| **AC-P01.1** | A newly created Landlord or Tenant account has `isFirstLogin = true` in the database. |
| **AC-P01.2** | On first successful login, the API response includes a flag `mustChangePassword: true` and the frontend immediately redirects to the password-change screen. No other route is accessible. |
| **AC-P01.3** | Any API call to a protected endpoint (other than `POST /api/v1/auth/change-password`) while `isFirstLogin = true` returns HTTP 403 `MUST_CHANGE_PASSWORD`. |
| **AC-P01.4** | After the user successfully sets a new password on first login: `isFirstLogin` is set to `false`, the temporary password is invalidated, and the user is redirected to their dashboard. |
| **AC-P01.5** | The new password on first login must meet the complexity rules (FR-P14) — a weak password returns HTTP 400 `VALIDATION_ERROR` with field-level feedback. |

---

## AC-P05 — Admin-Controlled Password Reset Flow

**Requirements:** FR-P05, FR-P06, FR-P07, FR-P08, FR-P09, FR-P10, FR-P11, FR-P12

| ID | Criterion |
|----|---|
| **AC-P05.1** | A Tenant, Landlord, or Admin can raise a Password Reset Request by providing their registered email or phone number. Request status is created as `PENDING`. |
| **AC-P05.2** | There is **no** public self-service password reset bypass. Any attempt to update password without an active session or approved reset returns HTTP 403 `FORBIDDEN`. |
| **AC-P05.3** | Raising a Password Reset Request emits an immediate real-time notification to the Admin and Super Admin frontend header notification panel (bell icon badge). |
| **AC-P05.4** | Upon Admin/Super Admin approval, the system auto-generates a secure temporary password, flags the account `isFirstLogin = true`, and **deletes ALL `RefreshToken` records for that user across ALL devices**. |
| **AC-P05.5** | Given a user with a registered email, the system dispatches the generated temporary password via Email. |
| **AC-P05.6** | **Edge Case (No Registered Email)**: Given a user without a registered email, Admin approval displays a single-view temporary password modal in the Admin portal (valid for 15 minutes, logged in audit trail) for manual/SMS dispatch. |
| **AC-P05.7** | Logging in with the temporary password returns HTTP 200 with `mustChangePassword: true` and forces immediate redirection to the password change screen. Access to any other route returns HTTP 403 `MUST_CHANGE_PASSWORD`. |
| **AC-P05.8** | Setting the new password successfully sets `isFirstLogin = false` and invalidates the temporary password. |
| **AC-P05.9** | After password reset approval, any existing access token on any other device returns HTTP 401 `UNAUTHORIZED` on its next API call. |

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

**Requirements:** FR-22, FR-23, FR-24, FR-25, FR-26, FR-27, FR-28, FR-29

| ID | Criterion |
|----|---|
| **AC-22.1** | A Landlord can create a Building with name, address, city, state, pincode, and power supplier name. |
| **AC-22.2** | A Landlord can update their own Building's name or address. |
| **AC-22.3** | A Landlord can add a Floor to their Building with a floor number and label. |
| **AC-22.4** | A Landlord can add a Room to a Floor specifying label (`Room XY` format), max occupancy, monthly rent, and bathroom type (`PRIVATE_ATTACHED` or `SHARED_FLOOR`). |
| **AC-22.5** | A Landlord can add Shared Bathrooms (`Bath XY`) and Shared Toilets (`Toilet XY`) to a Floor. |
| **AC-22.6** | A single floor can have a mix of private-attached rooms and shared facilities simultaneously. |
| **AC-22.7** | Attempting to access or modify another landlord's building returns HTTP 403 `FORBIDDEN`. |
| **AC-22.8** | A Landlord can retrieve the full hierarchy: building → floors → rooms → shared facilities. |

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

**Requirements:** FR-36, FR-37, FR-38, FR-39, FR-40, FR-41, FR-42, FR-43, FR-44

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
| **AC-36.9** | A Landlord can record payment method (`UPI`, `CASH`, `BANK_TRANSFER`) and transaction reference on a Rent Ledger entry. |
| **AC-36.10** | A Tenant can retrieve their own Rent Ledger entries; status and amounts are read-only. |

---

## AC-45 — Electricity Pass-Through Model

**Requirements:** FR-45, FR-46, FR-47, FR-48, FR-49, FR-50, FR-51, FR-52, FR-53

| ID | Criterion |
|----|---|
| **AC-45.1** | A Landlord can create an Electricity Ledger entry for a room with units consumed, rate per unit, and billing cycle dates. |
| **AC-45.2** | The system auto-calculates `Total Bill = Units Consumed × Rate per Unit`; the client-supplied total (if any) is ignored and overwritten by the calculation. |
| **AC-45.3** | A generated Electricity Ledger entry has initial status `UNPAID`. |
| **AC-45.4** | A Landlord can transition Electricity Ledger status: `UNPAID` → `PARTIALLY_PAID` → `PAID` or `OVERDUE`. |
| **AC-45.5** | Updating an Electricity Ledger status does not change the Room Rent Ledger for the same cycle — both records verified before and after. |
| **AC-45.6** | A Landlord can create a Supplier Master Bill entry with supplier name, cycle dates, total amount, and due date. |
| **AC-45.7** | A Landlord can upload a digital power bill (PDF/Image) linked to a Supplier Master Bill; the file is encrypted before R2 storage. |
| **AC-45.8** | Electricity totals collected from tenants do NOT appear in the Net Profit or revenue calculations. |
| **AC-45.9** | A Tenant can retrieve their own Electricity Ledger entries; read-only. |
| **AC-45.10** | Given tenant electricity collections $C$ and power supplier master bill amount $B$ for a billing period/month/year, the system computes `Variance = C - B`. |
| **AC-45.11** | Given `Variance > 0`, the system categorizes the reconciliation status as `SURPLUS` and reports the exact over-collected surplus amount. |
| **AC-45.12** | Given `Variance < 0`, the system categorizes the reconciliation status as `DEFICIT` and reports the exact under-collected deficit amount (landlord out-of-pocket loss). Given `Variance == 0`, status is `BALANCED`. |

---

## AC-54 — Building Operating Expenses

**Requirements:** FR-54, FR-55, FR-56, FR-57

| ID | Criterion |
|----|---|
| **AC-54.1** | A Landlord can log an expense with category, title, amount, and expense date. |
| **AC-54.2** | Expense category must be one of: `WATER_BILL`, `MAINTENANCE`, `REPAIRS`, `SECURITY`, `CLEANING`, `PROPERTY_TAX`, `MISCELLANEOUS`. Invalid category returns HTTP 400 `VALIDATION_ERROR`. |
| **AC-54.3** | A Landlord can retrieve all expenses for a building, optionally filtered by category and/or date range. |
| **AC-54.4** | Building expenses are included in the Net Profit calculation: `Net Profit = Rent Collected − Operating Expenses`. |

---

## AC-58 — Revenue Dashboard & P&L Analytics

**Requirements:** FR-58, FR-59, FR-60, FR-61, FR-62, FR-63, FR-64, FR-65, FR-66, FR-67

| ID | Criterion |
|----|---|
| **AC-58.1** | A Landlord can retrieve their own Revenue Dashboard data for a single building. |
| **AC-58.2** | A Landlord can retrieve a Portfolio-level rollup across all their buildings. |
| **AC-58.3** | Dashboard response includes: total rent collected, operating expenses, net profit, total electricity collected from tenants, total supplier bill amount, **electricity reconciliation variance**, **reconciliation status (`SURPLUS` | `DEFICIT` | `BALANCED`)**, pending dues, and occupancy rate. |
| **AC-58.4** | Dashboard results filtered by Monthly period return data for only that calendar month. |
| **AC-58.5** | Dashboard results filtered by Indian Financial Year return data for April 1 of year Y through March 31 of year Y+1. |
| **AC-58.6** | Dashboard results filtered by custom date range return data within those exact dates (inclusive). |
| **AC-58.7** | Historical trend data (multi-year) is available for revenue, expenses, and occupancy. |
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

## Gate

**Awaiting user confirmation.**

**Next:** Stage 03 — Non-Functional Requirements → `docs/03-non-functional-requirements.md`
