# 🔌 REST API CONTRACTS SPECIFICATION
## System: My Room Ledger (Security-First JSON RESTful APIs)

---

## 0. Global API Standards & Security Rules (Applies to ALL Endpoints)

### 0.1 — Mandatory Zod Schema Validation

> **STRICT RULE**: Every endpoint enforces input validation via `ZodValidationPipe` globally registered in NestJS bootstrap. No request body, query param, or route param ever reaches a Controller or Service without first passing its Zod schema.

| Input Type | Enforcement |
|---|---|
| `req.body` | Validated by endpoint-specific Zod schema |
| `req.query` | Validated by query Zod schema (unknown keys stripped) |
| `req.params` | Route params coerced & validated (e.g., `z.coerce.number()` for IDs) |
| File uploads | MIME type + size validated before any processing |

### 0.2 — Standard Validation Error Response (HTTP 400)

All validation failures return this exact shape:
```json
{
  "statusCode": 400,
  "error": "VALIDATION_ERROR",
  "message": "Input validation failed",
  "details": [
    { "field": "email", "message": "Invalid email address" },
    { "field": "password", "message": "Password must be at least 8 characters" }
  ]
}
```

### 0.3 — Standard Authentication Error Responses

| HTTP Code | Error Code | Trigger |
|---|---|---|
| `401` | `UNAUTHORIZED` | Missing, expired, or invalid JWT access token |
| `403` | `FORBIDDEN` | Valid JWT but insufficient role for the resource |
| `429` | `TOO_MANY_REQUESTS` | Rate limit exceeded (100 req/15min; auth: 5/15min) |

### 0.4 — Global Security Headers (Helmet.js)

All responses include:
- `Content-Security-Policy`
- `X-Frame-Options: DENY`
- `X-Content-Type-Options: nosniff`
- `Strict-Transport-Security: max-age=31536000; includeSubDomains`
- `Referrer-Policy: no-referrer`

### 0.5 — Request Pipeline (Every Endpoint)

```
Request → Helmet → CORS → Throttler → JWT Auth Guard → Roles Guard → ZodValidationPipe → Controller → Service → Prisma
```

---

## 1. Authentication & Role Management Endpoints


### `GET /api/v1/meta/country-codes`
- **Access**: Public (Cached on client / CDN)
- **Response (200 OK)**:
  ```json
  [
    {
      "id": "1",
      "countryCode": "IN",
      "dialCode": "+91",
      "countryName": "India",
      "flagEmoji": "🇮🇳",
      "phoneRegexPattern": "^[6-9]\\d{9}$",
      "minLength": 10,
      "maxLength": 10,
      "isDefault": true
    },
    {
      "id": "2",
      "countryCode": "US",
      "dialCode": "+1",
      "countryName": "United States",
      "flagEmoji": "🇺🇸",
      "phoneRegexPattern": "^[2-9]\\d{9}$",
      "minLength": 10,
      "maxLength": 10,
      "isDefault": false
    }
  ]
  ```

### `POST /api/v1/auth/login`
- **Access**: Public
- **Request Body (Variant A: Email + Password)**:
  ```json
  {
    "loginType": "EMAIL",
    "email": "landlord@myroomledger.com",
    "password": "SecurePassword123!"
  }
  ```
- **Request Body (Variant B: Phone + Country Code + Password)**:
  ```json
  {
    "loginType": "PHONE",
    "dialCode": "+91",
    "phoneNumber": "9876543210",
    "password": "SecurePassword123!"
  }
  ```
- **Response (200 OK)**:
  ```json
  {
    "token": "eyJhbGciOiJIUzI1NiJ9...",
    "role": "LANDLORD",
    "userId": "10",
    "fullName": "Rajesh Kumar",
    "mustChangePassword": false
  }
  ```

### `POST /api/v1/auth/forgot-password-request`
- **Access**: Public (Tenants, Landlords, Admins)
- **Request Body**:
  ```json
  {
    "emailOrPhone": "+919876543210"
  }
  ```
- **Response (201 Created)**:
  ```json
  {
    "requestId": "req_99812",
    "status": "PENDING",
    "message": "Password reset request submitted for Admin review. You will receive your temporary password upon approval."
  }
  ```

### `GET /api/v1/admin/password-reset-requests`
- **Access**: `SUPER_ADMIN` | `ADMIN`
- **Query Parameters**: `status=PENDING`
- **Response (200 OK)**:
  ```json
  [
    {
      "requestId": "req_99812",
      "userId": "45",
      "userName": "Ramesh Kumar",
      "userRole": "TENANT",
      "email": "ramesh@example.com",
      "phoneNumber": "+919876543210",
      "hasRegisteredEmail": true,
      "requestedAt": "2026-07-27T19:40:00Z"
    }
  ]
  ```

### `POST /api/v1/admin/password-reset-requests/{id}/approve`
- **Access**: `SUPER_ADMIN` | `ADMIN`
- **Response (200 OK - Email Case)**:
  ```json
  {
    "requestId": "req_99812",
    "status": "APPROVED",
    "deliveryChannel": "EMAIL",
    "tokenExpiryMinutes": 15,
    "message": "Signed reset link (valid for 15 mins) emailed to ramesh@example.com."
  }
  ```
- **Response (200 OK - No Email Fallback Case)**:
  ```json
  {
    "requestId": "req_99813",
    "status": "APPROVED",
    "deliveryChannel": "ADMIN_MODAL_DISPLAY",
    "temporaryPassword": "TempPassword#8821",
    "tempPasswordExpiryMinutes": 30,
    "message": "Temporary password generated (valid for 30 mins). Share securely with user."
  }
  ```

### `POST /api/v1/auth/reset-password-with-link-token`
- **Access**: Public (Email reset link flow)
- **Request Body**:
  ```json
  {
    "token": "signed_reset_token_xyz123",
    "newPassword": "MyNewSecurePassword123!"
  }
  ```
- **Response (200 OK)**:
  ```json
  {
    "message": "Password reset successfully. Reset link token invalidated. All active sessions purged. Please log in with your new password.",
    "redirectUrl": "/login"
  }
  ```

### `POST /api/v1/auth/change-password`
- **Access**: Authenticated (or user logged in with temp password / `mustChangePassword = true`)
- **Request Body**:
  ```json
  {
    "oldPassword": "TempPassword#8821",
    "newPassword": "NewPermanentSecurePass123!"
  }
  ```
- **Response (200 OK)**:
  ```json
  {
    "message": "Password updated successfully. All active sessions invalidated. Account unlocked.",
    "mustChangePassword": false,
    "redirectUrl": "/login"
  }
  ```

### `POST /api/v1/admins`
- **Access**: `SUPER_ADMIN` Only
- **Request Body**:
  ```json
  {
    "fullName": "Vikram Singh",
    "email": "vikram.admin@myroomledger.com",
    "phoneNumber": "+919876543210",
    "password": "AdminPassword123!"
  }
  ```
- **Response (201 Created)**:
  ```json
  {
    "adminId": "2",
    "fullName": "Vikram Singh",
    "email": "vikram.admin@myroomledger.com",
    "role": "ADMIN",
    "createdAt": "2026-07-27T10:00:00Z"
  }
  ```

### `POST /api/v1/landlords`
- **Access**: `SUPER_ADMIN` | `ADMIN`
- **Request Body**:
  ```json
  {
    "fullName": "Rajesh Kumar",
    "email": "rajesh.landlord@example.com",
    "phoneNumber": "+919812345678",
    "password": "LandlordSecret123!"
  }
  ```

---

## 2. Multi-Level Revenue & P&L Aggregation Endpoints

### `GET /api/v1/analytics/revenue`
- **Access**: `SUPER_ADMIN` | `ADMIN` (System-Wide) | `LANDLORD` (Own Buildings Only)
- **Query Parameters**:
  - `granularity`: `SINGLE_BUILDING` | `LANDLORD_PORTFOLIO` | `PER_LANDLORD` | `SYSTEM_WIDE`
  - `buildingId`: optional BigInt
  - `landlordId`: optional BigInt
  - `year`: e.g. `2025` (For Indian Financial Year: April 1, 2025 – March 31, 2026)
  - `month`: optional Int (1..12)
- **Response (200 OK)**:
  ```json
  {
    "granularity": "LANDLORD_PORTFOLIO",
    "financialYear": "FY 2025-26",
    "summary": {
      "totalRentCollected": 450000.00,
      "totalOperatingExpenses": 65000.00,
      "netLandlordProfit": 385000.00,
      "electricityReconciliation": {
        "totalTenantElectricityCollected": 38000.00,
        "totalSupplierMasterBillAmount": 40000.00,
        "variance": -2000.00,
        "status": "DEFICIT",
        "description": "Under-collected by ₹2,000.00 (Landlord out-of-pocket loss)"
      },
      "pendingRentDues": 15000.00,
      "pendingElectricityDues": 2500.00,
      "occupancyRatePercentage": 92.5,
      "paymentComplianceRatePercentage": 96.0
    },
    "buildingBreakdown": [
      {
        "buildingId": "101",
        "buildingName": "Sunshine Heights",
        "rentCollected": 250000.00,
        "operatingExpenses": 35000.00,
        "netProfit": 215000.00,
        "tenantElectricityCollected": 22000.00,
        "supplierBillAmount": 20000.00,
        "electricityVariance": 2000.00,
        "electricityStatus": "SURPLUS",
        "occupancyRate": 95.0
      }
    ]
  }
  ```

### `GET /api/v1/analytics/electricity-reconciliation`
- **Access**: `SUPER_ADMIN` | `ADMIN` (System-Wide) | `LANDLORD` (Own Buildings Only)
- **Query Parameters**:
  - `buildingId`: optional BigInt
  - `landlordId`: optional BigInt
  - `year`: e.g. `2025` (FY 2025-26)
  - `month`: optional Int (1..12)
- **Response (200 OK)**:
  ```json
  {
    "buildingId": "101",
    "buildingName": "Sunshine Heights",
    "period": "FY 2025-26",
    "totalTenantElectricityCollected": 38000.00,
    "totalSupplierMasterBillAmount": 40000.00,
    "varianceAmount": -2000.00,
    "status": "DEFICIT",
    "monthlyBreakdown": [
      {
        "month": "April 2025",
        "tenantCollected": 3500.00,
        "supplierBill": 3200.00,
        "variance": 300.00,
        "status": "SURPLUS"
      },
      {
        "month": "May 2025",
        "tenantCollected": 3000.00,
        "supplierBill": 3500.00,
        "variance": -500.00,
        "status": "DEFICIT"
      }
    ]
  }
  ```

---

## 3. Building Operating Expenses Endpoints

### `POST /api/v1/buildings/{buildingId}/expenses`
- **Access**: `LANDLORD` (Own Buildings) | `ADMIN` | `SUPER_ADMIN`
- **Request Body**:
  ```json
  {
    "category": "WATER_BILL",
    "title": "Municipal Water Tanker Charge - July",
    "amount": 4500.00,
    "expenseDate": "2026-07-15",
    "notes": "Emergency water tanker ordered during supply cut"
  }
  ```

---

## 4. Power Supplier (UPCL) Master Bill Endpoints

### `POST /api/v1/buildings/{buildingId}/supplier-master-bills`
- **Access**: `LANDLORD` (Own Buildings) | `ADMIN` | `SUPER_ADMIN`
- **Request Body**:
  ```json
  {
    "supplierName": "UPCL (Uttarakhand Power Corporation Limited)",
    "billCycleStart": "2026-06-01",
    "billCycleEnd": "2026-06-30",
    "masterBillAmount": 28500.00,
    "dueDate": "2026-07-20",
    "digitalBillDocId": "501"
  }
  ```

---

## 5. Security-First Encrypted File Storage Endpoints

### `POST /api/v1/files/upload`
- **Access**: `LANDLORD` | `ADMIN` | `SUPER_ADMIN` | `TENANT` (Own Govt ID / Receipts)
- **Request**: Multipart Form Data (`file`: PDF/Image, `category`: `GOVT_ID` | `SUPPLIER_BILL` | `RECEIPT`)
- **Backend Behavior**: Encrypts file buffer via AES-256 GCM before uploading to private Cloudflare R2 bucket.
- **Response (201 Created)**:
  ```json
  {
    "documentId": "501",
    "originalFileName": "upcl_july_bill.pdf",
    "mimeType": "application/pdf",
    "fileSizeBytes": 1048576,
    "uploadedAt": "2026-07-27T10:30:00Z"
  }
  ```

### `GET /api/v1/files/{documentId}/signed-url`
- **Access**: Authorized Role Scope Check
- **Response (200 OK)**:
  ```json
  {
    "signedUrl": "https://api.myroomledger.com/api/v1/files/stream?token=eyJhbGciOi...",
    "expiresInSeconds": 900
  }
  ```

---

## 6. Room Submeter & Rent Ledger Endpoints

### `POST /api/v1/rooms/{roomId}/billing-cycles/generate`
- **Access**: `LANDLORD` (Own Buildings) | `ADMIN` | `SUPER_ADMIN`
- **Request Body**:
  ```json
  {
    "cycleStartDate": "2026-08-01",
    "cycleEndDate": "2026-08-31",
    "submeterUnitsConsumed": 145.5,
    "electricityRatePerUnit": 8.50
  }
  ```

### `PATCH /api/v1/ledgers/room-rent/{ledgerId}`
- **Access**: `LANDLORD` (Own Buildings)
- **Request Body**:
  ```json
  {
    "status": "PAID",
    "amountPaid": 12000.00,
    "paidDate": "2026-08-03T10:30:00Z",
    "paymentMethod": "UPI",
    "transactionReference": "UPI/329482910"
  }
  ```

---

## 7. Tenant Complaint Endpoints

### `POST /api/v1/complaints`
- **Access**: `TENANT`
- **Request Body**:
  ```json
  {
    "category": "PLUMBING",
    "severity": "HIGH",
    "title": "Bathroom tap leaking in Bath 11",
    "description": "Continuous water leakage causing floor dampness since morning."
  }
  ```
