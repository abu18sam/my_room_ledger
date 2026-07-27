# 📚 GLOSSARY OF TERMS, ABBREVIATIONS & DOCUMENT REFERENCES

**System:** My Room Ledger / RentAway  
**Purpose:** Single authoritative index for domain vocabulary, technical abbreviations, system concepts, and document cross-references across all development stages.

---

## 1. Document Index & Identifier Prefix Map

| Prefix / File | Full Name / Scope | Primary Purpose | Authoritative Path |
| :--- | :--- | :--- | :--- |
| **BR-xx** | Business Rule | Authoritative domain logic, formulas, and security constraints | [`docs/business-rules.md`](business-rules.md) |
| **FR-xx** | Functional Requirement | Specific capability or behavior required by the system | [`docs/02-functional-requirements.md`](02-functional-requirements.md) |
| **AC-xx** | Acceptance Criteria | Testable condition for verifying functional requirements | [`docs/acceptance-criteria.md`](acceptance-criteria.md) |
| **NFR-xx** | Non-Functional Requirement | System quality attribute (performance, security, reliability) | [`docs/03-non-functional-requirements.md`](03-non-functional-requirements.md) |
| **MASTER** | Master Product Specification | High-level executive product summary & rule index | [`MASTER.md`](../MASTER.md) |
| **ARCH** | System Architecture | Technical design, request lifecycle, data flow diagrams | [`architecture.md`](../architecture.md) |
| **SCHEMA** | Database Schema | PostgreSQL DDL & Prisma ORM model definitions | [`database-schema.md`](../database-schema.md) |
| **API** | API Contracts | RESTful JSON endpoints, request/response DTO contracts | [`api-contracts.md`](../api-contracts.md) |

---

## 2. Technical Abbreviations & Concepts

| Abbreviation | Full Name | Definition & System Context | Referenced In |
| :--- | :--- | :--- | :--- |
| **RBAC** | Role-Based Access Control | Permission enforcement separating `SUPER_ADMIN`, `ADMIN`, `LANDLORD`, and `TENANT` access boundaries. | BR-01, FR-04–06, AC-01 |
| **PWA** | Progressive Web App | Installable client application built with Next.js + `next-pwa` supporting offline caching and mobile-first UX. | MASTER §1, ARCH §1 |
| **IFY** | Indian Financial Year | Accounting calendar period running from **April 1st to March 31st** of the following year for tax compliance. | BR-04, FR-60, AC-58.5 |
| **UPCL** | Uttarakhand Power Corporation Limited | External power utility provider issuing master building electricity bills. | BR-03, FR-50, AC-45.6 |
| **Zod** | TypeScript Validation Schema | Library used via NestJS `ZodValidationPipe` for strict input validation on all API endpoints. | BR-10.1, FR-78, AC-78 |
| **JWT** | JSON Web Token | Signed authentication token (15-min access token in-memory; 7-day refresh token in HttpOnly cookie). | BR-10.2, FR-02–05, AC-01 |
| **AES-256 GCM** | Advanced Encryption Standard (GCM Mode) | Authenticated symmetric encryption applied at NestJS backend level to all files before Cloudflare R2 upload. | BR-10.2, FR-69, AC-68.4 |
| **TTL** | Time To Live | Expiration duration for signed URLs (15 mins), password reset link tokens (15 mins), and temp passwords (30 mins). | BR-10.2, BR-11, FR-P07 |
| **E.164** | International Phone Standard | Standardized phone number format (`+<dialCode><number>`, e.g. `+919876543210`) used for DB normalization. | BR-12.2, FR-01g, AC-01.13 |
| **DTO** | Data Transfer Object | Strongly-typed Zod schema structure validating API request payloads and defining responses. | API §0.1, ARCH §1b |

---

## 3. Domain Terms & Business Vocabulary

| Term | Definition & System Scope | Referenced In |
| :--- | :--- | :--- |
| **Super Admin** | Platform Owner / Role Administrator. Highest authority capable of creating Admins and viewing Level 4 platform metrics. | BR-01.1, FR-09–14, AC-09 |
| **Admin** | Operations Manager. Administrative user who onboards Landlords/Tenants and views Level 3 aggregated reports. | BR-01.1, FR-15–21, AC-15 |
| **Landlord** | Property Owner / Lessor. Manages owned buildings, rooms, tenants, rent cycles, P&L, and expenses (Level 1–2). | BR-01.1, FR-22–29, AC-22 |
| **Tenant** | Resident / Occupant. Strictly read-only user accessing own room stay history, payment ledgers, and complaint submission. | BR-01.1, FR-30–35, AC-30 |
| **Pass-Through Cost** | Electricity payments collected from tenants and remitted to power supplier. Excluded from landlord net profit. | BR-03.1, FR-52, AC-45.8 |
| **Electricity Variance** | Difference between tenant collections ($C$) and power supplier master bill ($B$): $\text{Variance} = C - B$. | BR-03.3, FR-52b, AC-45.10 |
| **Surplus** | State when tenant electricity collections exceed master supplier bill ($\text{Variance} > 0$). | BR-03.3, FR-52c, AC-45.11 |
| **Deficit** | State when tenant electricity collections fall short of master supplier bill ($\text{Variance} < 0$), causing out-of-pocket loss. | BR-03.3, FR-52c, AC-45.12 |
| **Balanced** | State when tenant electricity collections exactly match master supplier bill ($\text{Variance} = 0$). | BR-03.3, FR-52c, AC-45.12 |
| **Tenancy Snapshot** | Immutable historical record locking active room tenants during a billing cycle, preventing bill mutation on move-out. | BR-08, FR-39, AC-36.4 |
| **Shared Bathroom / Toilet** | Floor-level shared sanitation facility using standardized numbering (`Bath XY`, `Toilet XY`). | BR-06, BR-07, FR-26, AC-22.5 |
| **Hybrid Floor** | Floor layout containing both rooms with private attached bathrooms and rooms utilizing shared floor bathrooms. | BR-06, AC-22.6 |
| **Password Masking Toggle** | UI control (show/hide eye icon) on password input fields allowing users to mask or reveal entered characters. | BR-11.1, FR-P00b, AC-P00.2 |
| **Header Notification Panel** | Admin frontend top-header bell icon displaying real-time alerts for pending password reset requests. | BR-11.2, FR-P06, AC-P05.1 |
| **Single-View Modal** | Secure Admin portal modal displaying generated temporary password for manual share (valid 15-30 mins, logged in audit). | BR-11.3, FR-P10, AC-P05.6 |
