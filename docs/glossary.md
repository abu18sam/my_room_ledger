# 📚 GLOSSARY OF TERMS, ABBREVIATIONS & DOCUMENT REFERENCES

**System:** My Room Ledger / RentAway  
**Purpose:** Single authoritative index for domain vocabulary, technical abbreviations, system concepts, and document cross-references across all development stages.

---

## 1. Document Index & Identifier Prefix Map

| Prefix / File | Full Name / Scope | Primary Purpose | Authoritative Path |
| :--- | :--- | :--- | :--- |
| **AC-xx** | Acceptance Criteria | Testable condition for verifying functional requirements | [`docs/acceptance-criteria.md`](acceptance-criteria.md) |
| **API** | API Contracts | RESTful JSON endpoints, request/response DTO contracts | [`api-contracts.md`](../api-contracts.md) |
| **ARCH** | System Architecture | Technical design, request lifecycle, data flow diagrams | [`architecture.md`](../architecture.md) |
| **BR-xx** | Business Rule | Authoritative domain logic, formulas, and security constraints | [`docs/business-rules.md`](business-rules.md) |
| **FR-xx** | Functional Requirement | Specific capability or behavior required by the system | [`docs/02-functional-requirements.md`](02-functional-requirements.md) |
| **MASTER** | Master Product Specification | High-level executive product summary & rule index | [`MASTER.md`](../MASTER.md) |
| **NFR-xx** | Non-Functional Requirement | System quality attribute (performance, security, reliability) | [`docs/03-non-functional-requirements.md`](03-non-functional-requirements.md) |
| **SCHEMA** | Database Schema | PostgreSQL DDL & Prisma ORM model definitions | [`database-schema.md`](../database-schema.md) |

---

## 2. Technical Abbreviations & Concepts

| Abbreviation | Full Name | Definition & System Context | Referenced In |
| :--- | :--- | :--- | :--- |
| **AES-256 GCM** | Advanced Encryption Standard (GCM Mode) | Authenticated symmetric encryption applied at NestJS backend level to all files before Cloudflare R2 upload. | BR-10.2, FR-69, AC-68.4 |
| **DTO** | Data Transfer Object | Strongly-typed Zod schema structure validating API request payloads and defining responses. | API §0.1, ARCH §1b |
| **E.164** | International Phone Standard | Standardized phone number format (`+<dialCode><number>`, e.g. `+919876543210`) used for DB normalization. | BR-12.2, FR-01g, AC-01.13 |
| **IFY** | Indian Financial Year | Accounting calendar period running from **April 1st to March 31st** of the following year for tax compliance. | BR-04, FR-60, AC-58.5 |
| **JWT** | JSON Web Token | Signed authentication token (10-min access token in-memory; 7-day refresh token in HttpOnly cookie) as specified in [`docs/ttl-registry.md`](ttl-registry.md). | BR-10.2, FR-02–05, AC-01, docs/ttl-registry.md |
| **PWA** | Progressive Web App | Installable client application built with Next.js + `next-pwa` supporting offline caching and mobile-first UX. | MASTER §1, ARCH §1 |
| **RBAC** | Role-Based Access Control | Permission enforcement separating `SUPER_ADMIN`, `ADMIN`, `LANDLORD`, and `TENANT` access boundaries. | BR-01, FR-04–06, AC-01 |
| **TTL** | Time To Live | Expiration duration for all system tokens, links, and temporary access mechanisms. Single source of truth is [`docs/ttl-registry.md`](ttl-registry.md). | BR-10.2, BR-11, FR-P07, docs/ttl-registry.md |
| **UPCL** | Uttarakhand Power Corporation Limited | External power utility provider issuing master building electricity bills. | BR-03, FR-50, AC-45.6 |
| **Zod** | TypeScript Validation Schema | Library used via NestJS `ZodValidationPipe` for strict input validation on all API endpoints. | BR-10.1, FR-78, AC-78 |

---

## 3. Domain Terms & Business Vocabulary

| Term | Definition & System Scope | Referenced In |
| :--- | :--- | :--- |
| **ACID Transaction Lock** | PostgreSQL serializable transaction boundary (`prisma.$transaction`) ensuring concurrent payment writes and ledger balance recalculations execute safely without race conditions. | BR-14.1, NFR-08, docs/03-non-functional-requirements.md |
| **Action Type** | Standardized SCREAMING_SNAKE_CASE string identifying a specific auditable action (e.g. `FORCE_LOGOUT_USER`, `SAVE_PAYMENT_TRANSACTION`, `SWITCH_POWER_SUPPLIER`). Indexed in `docs/audit-logging.md`. | BR-16.4, FR-94, AC-89.5, docs/audit-logging.md |
| **Actionable Error Message** | User-facing error message describing what failed AND what actionable step to take next. Internal stack traces, raw database error strings, and SQL queries are strictly hidden. | BR-15.3, FR-87, AC-82.11 |
| **Active Session Check** | Database/Redis query executed prior to API logic to ensure the underlying user session has not been forcefully terminated. | BR-16.5, FR-113, AC-113.1 |
| **Admin** | Operations Manager. Administrative user who onboards Landlords/Tenants and views Level 3 aggregated reports. | BR-01.1, FR-15–21, AC-15 |
| **Argon2id Password Hashing** | Mandatory system-wide memory-hard key derivation function (64MB RAM, 3 iterations, 4 parallelism threads) used exclusively to hash user credentials. | BR-10.2, BR-11.4, NFR-02, FR-P13, AC-P13.2 |
| **Audit Category** | High-level grouping of audit log action types (`SESSION`, `AUTH`, `USER_MANAGEMENT`, `ASSET_MANAGEMENT`, `TENANT_MANAGEMENT`, `FINANCIAL`, `SYSTEM`). | BR-16.1, FR-94, docs/audit-logging.md |
| **AuditLog** | An immutable, insert-only database table (`audit_logs`) recording every state-changing API operation and security event with acting user, target entity, timestamp, IP, User-Agent, and metadata. | BR-16.1, FR-93, AC-89.5, docs/audit-logging.md |
| **Balanced** | State when tenant electricity collections exactly match master supplier bill ($\text{Variance} = 0$). | BR-03.3, FR-52c, AC-45.12 |
| **Bill Serial Number** | Unique invoice/bill number printed on a physical utility bill issued by a power company. Unique per power company (`@@unique([powerCompanyId, billSerialNumber])`). | BR-13.4, FR-50, AC-45.6a |
| **Building Occupancy Stack** | Stacked vertical visual representation of building floors (top floor down to Ground Floor) in the Building Details page showing floor numbers, occupancy badges (`OCCUPIED` \| `VACANT`), and room count summaries. | BR-17.4, FR-123, AC-119.4, docs/frontend-navigation.md |
| **BuildingPowerConnection** | Historical audit model tracking every power supplier association for a building over time, with `startDate`, `endDate`, and `status` (`ACTIVE` \| `TERMINATED`). | BR-13.9, FR-22c, AC-22.10 |
| **Carry-Forward Balance** | Informal term for unpaid ledger amounts persisting across billing cycles. Not physically transferred — computed dynamically as `SUM(amount − amountPaid)` across all non-PAID cycles for a room. | BR-14.4, FR-41c, AC-36.17 |
| **Common Electricity Load** | Electricity consumed by building common areas (lights, submersible water pumps) paid by landlord from surplus or rental income. | BR-03.2, FR-52d, AC-45.13 |
| **Connection Number** | Unique consumer account or K-number assigned by the power supply company to a building's electricity meter. Mandatory on building creation (`Building.connectionNumber`) and constant across billing cycles for that supplier. | BR-13.5, BR-13.7, FR-22, AC-22.1 |
| **Country Code Registry** | Centrally managed database table (`country_codes`), seeded from base JSON (`data/country-codes.json`), storing ISO alpha-2 codes, dial codes, flag emojis, and phone validation regex patterns. Protected by `ON DELETE RESTRICT` and in-use immutability rules. | BR-12.2, FR-35a, AC-35.1, database-schema.md |
| **Deficit** | State when tenant electricity collections fall short of master supplier bill ($\text{Variance} < 0$), causing out-of-pocket loss. | BR-03.3, FR-52c, AC-45.12 |
| **Electricity Reconciliation Engine** | 3-step calculation algorithm executing upon supplier master bill payment to compare aggregated tenant collections against supplier bill paid and classify outcome into Surplus, Deficit, or Break-even. | BR-03.3, BR-03.5, FR-103, AC-101.2, docs/billing-and-reconciliation.md |
| **Electricity Variance** | Difference between tenant collections ($C$) and power supplier master bill ($B$): $\text{Variance} = C - B$. | BR-03.3, FR-52b, AC-45.10 |
| **Error Code** | Machine-readable SCREAMING_SNAKE_CASE string uniquely identifying an error condition (e.g. `COMPANY_IN_USE`, `PENDING_SUPPLIER_BILLS_EXIST`). Enables precise client error routing. | BR-15.1, FR-82, AC-82.1, docs/error-handling.md |
| **Error Envelope** | Standardized JSON structure returned for all non-2xx API responses: `{ statusCode, error, message, metadata }` (or `details` for validation errors). | BR-15.1, FR-82, AC-82.1, docs/error-handling.md |
| **Flexible Billing Cycle** | Non-calendar billing cycle structure allowing each room to start/end on any day ($1..31$), with room rent, room electricity, and building master bill cycles operating on independent schedules. | BR-02, FR-101, AC-101.1, docs/billing-and-reconciliation.md |
| **Floor Occupancy State** | Aggregated occupancy status of a floor (`OCCUPIED` if $\ge 1$ room is occupied; `VACANT` if ALL rooms are vacant). | BR-17.2, FR-120, AC-119.2, docs/building-occupancy.md |
| **Force Logout** | Administrative action allowing a Super Admin or Admin to forcefully invalidate active user sessions (`UserSession` records purged from DB) based on role hierarchy. | BR-01.5, FR-89–91, AC-89.1–4 |
| **Frontend Responsive Navigation** | Frontend presentation architecture adapting Building Stack, Floor Details, Room Blocks, and Tenant Cards across Mobile ($<640\text{px}$), Tablet ($640-1024\text{px}$), and Desktop ($>1024\text{px}$) viewports. | BR-17.4, FR-123–127, NFR-26, docs/frontend-navigation.md |
| **Header Notification Panel** | Admin frontend top-header bell icon displaying real-time alerts for pending password reset requests. | BR-11.2, FR-P06, AC-P05.1 |
| **Hybrid Floor** | Floor layout containing both rooms with private attached bathrooms and rooms utilizing shared floor bathrooms. | BR-06, AC-22.6 |
| **Invitation Link Token** | Single-use 30-minute token sent to new users for onboarding registration, invalidated immediately upon use (`usedAt = now()`). | BR-11, FR-P10, docs/ttl-registry.md |
| **Landlord** | Property Owner / Lessor. Manages owned buildings, rooms, tenants, rent cycles, P&L, and expenses (Level 1–2). | BR-01.1, FR-22–29, AC-22 |
| **Last Transaction Update Rule** | Domain rule allowing landlords to update (`PATCH`) ONLY the chronologically latest payment transaction for a ledger cycle (`recordedAt` max), automatically triggering parent ledger balance & status recalculation. | BR-14.7, FR-98, AC-97.1, docs/rbac-matrix.md |
| **Master Cycle Allocation** | The derived assignment of a tenant payment transaction to a specific building master electricity billing cycle window based on payment date timestamp. | BR-03.5, FR-107, AC-107.1, docs/billing-and-reconciliation.md |
| **Non-Functional Requirement** | System quality constraint (`NFR-01` to `NFR-20` in `docs/03-non-functional-requirements.md`) governing security, immutability, performance SLAs, scalability, and error contracts. | docs/03-non-functional-requirements.md |
| **Non-Last Transaction Lock** | Security restriction (`HTTP 409 NON_LAST_TRANSACTION_UPDATE_RESTRICTED`) blocking edits to older (non-latest) payment transaction records to maintain financial ledger history integrity. | BR-14.7, FR-99, AC-97.2, docs/error-handling.md |
| **Occupied Status** | Occupancy classification indicating $\ge 1$ active assigned tenant (`Tenant.status = ACTIVE`) currently living in a room, floor, or building. | BR-17.1, FR-119, AC-119.1, docs/building-occupancy.md |
| **One-Time Token Invalidation** | Security rule requiring single-use tokens (invitations, password resets, temp passwords) to be immediately invalidated upon consumption to prevent reuse. | BR-11, NFR-01, docs/ttl-registry.md |
| **OVERDUE** | A `PaymentStatus` enum value assigned when `currentDate > BillingCycle.cycleEndDate` and the ledger is still `UNPAID` or `PARTIALLY_PAID`. No grace period applies. An OVERDUE ledger can still receive payments and transition to `PAID` upon full settlement. | BR-14.3, FR-41b, AC-36.15 |
| **Pass-Through Cost** | Electricity payments collected from tenants and remitted to power supplier. Excluded from landlord net profit. | BR-03.1, FR-52, AC-45.8 |
| **Password Masking Toggle** | UI control (show/hide eye icon) on password input fields allowing users to mask or reveal entered characters. | BR-11.1, FR-P00b, AC-P00.2 |
| **Payment Cutoff Rule** | Mandatory allocation logic assigning tenant electricity payments to building master cycles based strictly on `PaymentTransaction.paymentDate` ($\le \text{masterCycleEndDate} \rightarrow$ Current Cycle; $> \text{masterCycleEndDate} \rightarrow$ Next Cycle), independent of room billing cycle dates. | BR-03.5, FR-107–109, AC-107.1–2, docs/billing-and-reconciliation.md |
| **Payment Date Allocation** | Allocation strategy prioritizing actual cash receipt date (`paymentDate`) over room billing period start/end dates for reconciliation accounting. | BR-03.5, FR-107, AC-107.1, docs/billing-and-reconciliation.md |
| **PaymentTransaction** | An immutable record of a single payment event (partial or full) made against a `RoomRentLedger` or `ElectricityLedger`. Multiple transactions may exist per ledger cycle. Stores `amountPaid`, `paymentDate`, `paymentMethod`, and optional `transactionReference`. | BR-14.1, FR-41, AC-36.11 |
| **Per-Unit Rate Differential** | Variance between per-unit rate charged by landlord to tenants (e.g. ₹8/unit) vs utility company tariff rate (e.g. ₹7/unit). | BR-03.2, FR-45, AC-45.1 |
| **Phone Regex Pattern** | Database-driven regular expression pattern stored per `CountryCode` entity used for server-side Zod phone number validation. | BR-12.5, FR-01f, AC-01.11 |
| **Power Supplier Transition** | The controlled operation of switching a building's assigned power supply company and connection number (`POST /api/v1/buildings/{id}/switch-power-supplier`). Blocked if open bills exist (`HTTP 409 PENDING_SUPPLIER_BILLS_EXIST`). Archives previous connection and activates new connection. | BR-13.8, BR-13.9, FR-22a–b, AC-22.8–9 |
| **Power Supply Company** | External electricity provider entity (e.g. UPCL, UPPCL, Reliance, Adani, TPCL, NTPC) stored in database and linked to buildings. | BR-13, FR-50a–e, AC-50 |
| **RBAC Matrix** | Centralized single-source-of-truth document (`docs/rbac-matrix.md`) mapping CRUD capabilities across 4 roles (`SUPER_ADMIN`, `ADMIN`, `LANDLORD`, `TENANT`) for all database entities and special operational restrictions. | BR-01.1, FR-97, AC-97.4, docs/rbac-matrix.md |
| **Referential Integrity (RESTRICT)** | PostgreSQL foreign key constraint (`ON DELETE RESTRICT`) preventing deletion, disabling, or modification of `CountryCode` records referenced by active `User` entities. | BR-12.3, NFR-25, AC-35.6, database-schema.md |
| **Response Time SLA (p95)** | Performance target requiring 95% of standard API requests to respond within $\le 200\text{ ms}$ and complex P&L aggregations within $\le 400\text{ ms}$. | NFR-10, docs/03-non-functional-requirements.md |
| **Revocation Overrides TTL** | Invariant guaranteeing that an active DB session revocation immediately invalidates any bearer access token issued for that session, regardless of remaining TTL. | BR-16.5, FR-114, AC-113.3 |
| **Role Hierarchy Violation** | Authorization block (`HTTP 403 FORBIDDEN`, error code: `ROLE_HIERARCHY_VIOLATION`) triggered when an Admin attempts to execute force-logout or administrative operations on a Super Admin or fellow Admin user. | BR-01.5, FR-91, AC-89.3, docs/error-handling.md |
| **Room Block Card** | Horizontal interactive UI card on the Floor Details page rendering room label, occupancy status, tenant count badge, and hover tooltips. | BR-17.4, FR-124, AC-119.5, docs/frontend-navigation.md |
| **Session Revocation Enforcement** | Security mechanism (`SessionValidationGuard`) verifying database active session status (`user_sessions.isRevoked = false`) on every protected API call. | BR-16.5, FR-113–118, AC-113 |
| **SESSION_REVOKED Error** | Standardized `HTTP 401 UNAUTHORIZED` error envelope (`ERR-1002`) returned when an API request is attempted on a revoked session. | BR-16.5, docs/error-handling.md §4.5, API §0.3 |
| **Shared Bathroom / Toilet** | Floor-level shared sanitation facility using standardized numbering (`Bath XY`, `Toilet XY`). | BR-06, BR-07, FR-26, AC-22.5 |
| **Signed R2 URL** | Time-bound (15-minute TTL) encrypted URL generated by NestJS backend for secure document downloads from Cloudflare R2 bucket. | BR-10.3, NFR-03, docs/03-non-functional-requirements.md |
| **Single-View Modal** | Secure Admin portal modal displaying generated temporary password for manual share (valid 15-30 mins, logged in audit). | BR-11.3, FR-P10, AC-P05.6 |
| **Super Admin** | Platform Owner / Role Administrator. Highest authority capable of creating Admins and viewing Level 4 platform metrics. | BR-01.1, FR-09–14, AC-09 |
| **Surplus** | State when tenant electricity collections exceed master supplier bill ($\text{Variance} > 0$). | BR-03.3, FR-52c, AC-45.11 |
| **Surplus/Deficit Variance** | The financial difference ($\text{TenantCollections} - \text{MasterBillPaid}$). Positive variance creates a Surplus (`surplusAmount`); negative variance creates a Landlord Deficit (`deficitAmount`). Pass-through funds strictly excluded from Net Profit. | BR-03.1, BR-03.3, FR-104, AC-101.3, docs/billing-and-reconciliation.md |
| **Tenancy Snapshot** | Immutable historical record locking active room tenants during a billing cycle, preventing bill mutation on move-out. | BR-08, FR-39, AC-36.4 |
| **Tenant** | Resident / Occupant. Strictly read-only user accessing own room stay history, payment ledgers, and complaint submission. | BR-01.1, FR-30–35, AC-30 |
| **Tenant Access Isolation** | Backend security rule (`TenantRoomAccessGuard`) restricting `TENANT` role users to access ONLY the Room Details page of their assigned room (`Tenant.currentRoomId`). | BR-17.5, FR-128–130, AC-119.10, docs/frontend-navigation.md |
| **Tenant Collection Aggregation** | The sum of actual cash collected (`PaymentTransaction.amountPaid`) from tenants across all room electricity ledgers whose cycle intersects the building master bill window `[billCycleStart, billCycleEnd]`. | BR-03.5, FR-103, AC-101.2, docs/billing-and-reconciliation.md |
| **Token Lifecycle Governance** | Policy requiring all token-based features to define TTL, transport mechanism, and one-time invalidation rules in `docs/ttl-registry.md` before implementation. | docs/ttl-registry.md |
| **Total Units Consumed** | Total building-level electricity consumption (kWh) recorded on a `SupplierMasterBill`. Used to cross-verify against the sum of room submeter consumption readings. | BR-13.6, FR-50, AC-45.6 |
| **TTL Registry** | Dedicated single-source-of-truth governance specification (`docs/ttl-registry.md`) standardizing all 8 token lifecycles, TTL durations, and invalidation rules across the application. | docs/ttl-registry.md |
| **UUID Primary Key Strategy** | The mandatory system-wide identifier policy replacing sequential integer auto-increment keys with 128-bit RFC 4122 non-sequential UUID strings (`gen_random_uuid()` / `@default(uuid()) @db.Uuid`) across all 22 database entities. Prevents resource enumeration and IDOR attacks. | BR-10.4, FR-88, AC-82.12 |
| **UUIDv7 Primary Key Strategy** | Mandatory primary key standard across all 24 database entities, embedding a 48-bit millisecond timestamp in high-order bits for time-ordered B-Tree index locality and high-throughput INSERT efficiency. | BR-10.4, NFR-11, FR-88, AC-82.12 |
| **Vacant Status** | Occupancy classification indicating zero (`0`) active assigned tenants currently living in a room, floor, or building. | BR-17.1, FR-119, AC-119.1, docs/building-occupancy.md |








