# Role-Based Access Control (RBAC) Matrix & Permission Specification

## System: My Room Ledger (NestJS Backend API)

---

## 1. Architectural Philosophy & Security Principles

1. **Backend as Single Source of Truth**: All authorization checks are enforced on the server side via NestJS `@Roles(...)` decorator + `JwtAuthGuard` + `RolesGuard`. The frontend UI renders permission-based views dynamically for UX, but is never treated as a security boundary.
2. **Row-Level Data Isolation**: Multi-tenant data isolation is strictly enforced at the database query layer (Prisma ORM `where` clauses filtering by `landlordId` for Landlords and `userId` for Tenants).
3. **Role Hierarchy**:
   - `SUPER_ADMIN`: System-level operational authority over users and platform settings.
   - `ADMIN`: Operational management over Landlord and Tenant onboarding and reporting.
   - `LANDLORD`: Asset and financial management strictly scope-bound to own owned properties.
   - `TENANT`: Self-service access strictly scope-bound to own assigned room, ledger, and complaints.
4. **Special Operational Restrictions**: Immutable logs (`AuditLog`, `TenancyHistory`, `BuildingPowerConnection`) cannot be updated or deleted by ANY role (including Super Admin). Payment transactions (`PaymentTransaction`) have strict mutation restrictions (Landlords may update the latest transaction only; Super Admin/Admin/Tenant have read-only or zero mutation access).

---

## 2. Comprehensive Entity Permissions Matrix

### Legend:
- **C** = Create (Allowed)
- **R** = Read (Allowed within scope)
- **U** = Update (Allowed within scope)
- **D** = Delete (Allowed within scope)
- **L** = Latest Only (Update allowed ONLY for the chronologically latest record)
- **—** = Forbidden (No access permitted)

| Entity / Asset | SUPER_ADMIN | ADMIN | LANDLORD | TENANT | Scope & Operational Conditions |
|---|:---:|:---:|:---:|:---:|---|
| **Super Admin Accounts** | C / R / U / D | — | — | — | Super Admin users can ONLY be created/managed by another Super Admin. |
| **Admin Accounts** | C / R / U / D | — | — | — | Super Admin manages Admin accounts. |
| **Landlord Accounts** | C / R / U / D | C / R / U / D | — | — | Admins & Super Admins onboard & manage Landlords. |
| **Tenant Accounts** | C / R / U / D | C / R / U / D | C / R / U | — | Landlords onboard tenants to own rooms; Admins manage globally. |
| **User Sessions (`UserSession`)** | R / D (All) | R / D (Landlord/Tenant) | R / D (Self) | R / D (Self) | Force-logout subject to role hierarchy rules (BR-01.5). |
| **Country Code Metadata** | C / R / U / D | C / R / U / D | R | R | Read-only for public/auth; managed by Admin & Super Admin via `/api/v1/admin/country-codes`. Update/Disable/Delete blocked if referenced by ≥ 1 user (`COUNTRY_CODE_IN_USE`). |
| **Power Supply Company** | C / R / U / D | C / R / U / D | R | — | Deletion blocked if company linked to ≥1 building (`COMPANY_IN_USE`). |
| **Building Infrastructure** | C / R / U / D | C / R / U / D | C / R / U / D (Own) | — | Landlord strictly limited to own registered buildings. |
| **Power Supplier Switch** | C / U | C / U | C / U (Own) | — | Requires zero open master bills (`PENDING_SUPPLIER_BILLS_EXIST`). |
| **Building Power Connections (History)** | R | R | R | — | Inserted automatically on supplier switch; **Insert-Only (No U/D)**. |
| **Floors, Rooms & Facilities** | C / R / U / D | C / R / U / D | C / R / U / D (Own) | R (Assigned Room) | Tenants view assigned room infrastructure read-only. |
| **Tenancy History (Snapshots)** | R | R | R (Own) | R (Self) | Immutable billing snapshots; **Insert-Only (No U/D)**. |
| **Rent Cycle Configurations** | C / R / U | C / R / U | C / R / U (Own) | R (Assigned Room) | Landlords configure billing rules for own rooms. |
| **Billing Cycles** | C / R / U | C / R / U | C / R / U (Own) | R (Assigned Room) | Generated per room cycle; locks tenancy snapshot. |
| **Room Rent Ledgers** | C / R / U | C / R / U | C / R / U (Own) | R (Self) | Status updated automatically based on payments and due date. |
| **Electricity Ledgers** | C / R / U | C / R / U | C / R / U (Own) | R (Self) | Driven by submeter unit readings & pass-through tariff. |
| **Payment Transactions** | R (View Only) | R (View Only) | C / R / **L** (Latest Only) | R (Self View) | **Special Rule**: Landlord can update ONLY the latest payment transaction. |
| **Supplier Master Bills** | C / R / U | C / R / U | C / R / U (Own) | — | Single lump-sum settlement (`PAID`); no partial payments. |
| **Building Operating Expenses** | C / R / U / D | C / R / U / D | C / R / U / D (Own) | — | Landlord logs expenses for own buildings. |
| **Document Metadata (Cloudflare R2)** | C / R / D | C / R / D | C / R / D (Own) | C / R (Own ID/Receipt) | Files encrypted AES-256 GCM; access via 15-min signed URLs. |
| **Tenant Complaints** | R / U | R / U | R / U (Own Buildings) | C / R (Self) | Tenants log tickets; Landlords update resolution status/notes. |
| **Password Reset Requests** | R / U | R / U | C (Self Request) | C (Self Request) | Admin approves email reset link or generates temp password. |
| **Audit Logs (`AuditLog`)** | R (View Only) | R (View Only) | — | — | **Absolute Immutability**: Insert-Only across all roles (No U/D). |
| **Analytics & P&L Dashboards** | R (System-Wide) | R (System-Wide) | R (Own Portfolio) | — | Multi-level financial reporting (FY April 1 – March 31). |

---

## 3. Special Operational Restrictions & Immutability Governance

### 3.1 Super Admin Operational Restrictions
While a `SUPER_ADMIN` possesses broad authority over user accounts and platform assets:
1. **Audit Logs (`AuditLog`)**: Super Admin **CANNOT modify or delete** any audit log entries under any circumstances. Audit logs are strictly insert-only.
2. **History Logs (`TenancyHistory`, `BuildingPowerConnection`)**: Super Admin **CANNOT edit or delete** historical tenancy snapshots or power connection audit logs.
3. **Payment Transactions (`PaymentTransaction`)**: Super Admin **CANNOT insert, edit, or delete** payment transaction records. Super Admin has **READ-ONLY** visibility to view landlord transaction history for audit compliance.

---

### 3.2 Landlord "Last Transaction Only" Payment Update Rule
To allow landlords to correct entry mistakes (e.g. typos in payment amount, date, reference, or notes) while preserving total chronological financial integrity:
1. **Creation**: Landlords can create payment transactions (`POST /api/v1/ledgers/room-rent/{id}/payments` and `POST /api/v1/ledgers/electricity/{id}/payments`).
2. **Latest Transaction Update**: A landlord can update a payment transaction (`PATCH /api/v1/ledgers/payments/{transactionId}`) **ONLY IF** that transaction is the **chronologically latest record** (highest `recordedAt` timestamp) associated with that ledger.
3. **Modifiable Fields**: `amountPaid`, `paymentDate`, `paymentMethod`, `transactionReference`, `notes`.
4. **Ledger Recalculation**: Updating the latest transaction automatically recalculates the parent ledger's total `amountPaid` sum (`SUM(PaymentTransaction.amountPaid)`) and updates the parent ledger status (`UNPAID` / `PARTIALLY_PAID` / `PAID` / `OVERDUE`).
5. **Non-Latest Lock**: Landlords **CANNOT update any prior (older) transactions**. Non-latest transaction edit attempts are rejected with `HTTP 409 NON_LAST_TRANSACTION_UPDATE_RESTRICTED`. UI edit controls are hidden for non-latest records.

---

### 3.3 Session Termination Hierarchy Rules
1. `SUPER_ADMIN` can terminate sessions of `ADMIN`, `LANDLORD`, and `TENANT`.
2. `ADMIN` can terminate sessions of `LANDLORD` and `TENANT`.
3. `ADMIN` attempting to terminate sessions of a `SUPER_ADMIN` or another `ADMIN` is strictly blocked (`HTTP 403 FORBIDDEN`, error: `ROLE_HIERARCHY_VIOLATION`).
