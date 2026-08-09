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
| `req.params` | Route params coerced & validated as RFC 4122 UUID strings (`z.string().uuid()`) |
| File uploads | MIME type + size validated before any processing |

### 0.2 — Universal Error Response Envelopes

All non-2xx API responses (excluding raw file streams) MUST strictly conform to the universal error envelope specification detailed in [`docs/error-handling.md`](file:///Users/abdulsamad/Desktop/Projects/my_room_ledger/docs/error-handling.md).

#### Standard Error Response Envelope:
```json
{
  "statusCode": 409,
  "error": "COMPANY_IN_USE",
  "message": "Cannot delete 'Uttarakhand Power Corporation Limited (UPCL)' because 3 buildings are currently linked to it.",
  "metadata": {
    "companyId": "1",
    "linkedBuildingCount": 3,
    "affectedBuildings": []
  }
}
```

#### Field-Level Validation Error Response (HTTP 400 `VALIDATION_ERROR`):
All validation failures (`ZodValidationPipe`) return field-level details:
```json
{
  "statusCode": 400,
  "error": "VALIDATION_ERROR",
  "message": "Input validation failed. Please correct the highlighted fields.",
  "details": [
    { "field": "email", "message": "Invalid email address format" },
    { "field": "phoneNumber", "message": "Phone number must be 10 digits starting with 6, 7, 8, or 9 for dial code +91 (India)" }
  ]
}
```

### 0.3 — Standard Authentication & System Error Matrix

| HTTP Code | Error Code | Trigger | Response Metadata |
|---|---|---|---|
| `400` | `VALIDATION_ERROR` | Request body, params, or query fail Zod schema | `details[]` array |
| `400` | `PAYMENT_EXCEEDS_BALANCE` | Recorded payment exceeds remaining due | `remainingBalance`, `attemptedPayment` |
| `401` | `UNAUTHORIZED` | Missing, invalid, or expired JWT access token | `{}` |
| `401` | `INVALID_CREDENTIALS` | Incorrect email/phone or password on login | `{}` |
| `401` | `TOKEN_EXPIRED` | Expired access token or password reset token | `{}` |
| `403` | `FORBIDDEN` | Valid JWT but insufficient role or ownership | `{}` |
| `403` | `MUST_CHANGE_PASSWORD` | Account flagged `mustChangePassword = true` | `{}` |
| `404` | `NOT_FOUND` | Target entity ID does not exist | `resourceId` |
| `409` | `DUPLICATE_ENTRY` | Unique constraint violation (email, phone, serial) | `field` |
| `409` | `COMPANY_IN_USE` | Deleting power company linked to ≥1 buildings | `affectedBuildings[]` |
| `409` | `PENDING_SUPPLIER_BILLS_EXIST` | Switching supplier with open master bills | `pendingBills[]` |
| `429` | `TOO_MANY_REQUESTS` | Rate limit exceeded (e.g. 5 failed logins / 15m) | `{}` |
| `500` | `INTERNAL_SERVER_ERROR` | Unhandled server exception (sanitized) | `{}` |

### 0.6 — Central Error Registry Reference
> See [`docs/error-handling.md`](file:///Users/abdulsamad/Desktop/Projects/my_room_ledger/docs/error-handling.md) for complete documentation of all 21 error codes, metadata schemas, and status mappings.

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
      "id": "a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11",
      "countryCode": "IN",
      "dialCode": "+91",
      "countryName": "India",
      "flagEmoji": "🇮🇳",
      "phoneRegexPattern": "^[6-9]\\d{9}$",
      "minLength": 10,
      "maxLength": 10,
      "isDefault": true
    }
  ]
  ```

### `GET /api/v1/meta/power-companies`
- **Access**: Public / Authenticated (Populates FE dropdowns)
- **Response (200 OK)**:
  ```json
  [
    { "id": "11111111-1111-4111-8111-111111111111", "name": "Uttarakhand Power Corporation Limited (UPCL)", "status": "ACTIVE" },
    { "id": "22222222-2222-4222-8222-222222222222", "name": "Uttar Pradesh Power Corporation Limited (UPPCL)", "status": "ACTIVE" },
    { "id": "33333333-3333-4333-8333-333333333333", "name": "Reliance Power Ltd", "status": "ACTIVE" },
    { "id": "44444444-4444-4444-8444-444444444444", "name": "Adani Power Ltd", "status": "ACTIVE" },
    { "id": "55555555-5555-4555-8555-555555555555", "name": "Tata Power Company Limited (TPCL)", "status": "ACTIVE" },
    { "id": "66666666-6666-4666-8666-666666666666", "name": "National Thermal Power Corporation (NTPC)", "status": "ACTIVE" }
  ]
  ```

### `POST /api/v1/admin/power-companies`
- **Access**: `SUPER_ADMIN` | `ADMIN`
- **Request Body**:
  ```json
  {
    "name": "Torrent Power Ltd",
    "status": "ACTIVE"
  }
  ```
- **Response (201 Created)**:
  ```json
  {
    "id": "77777777-7777-4777-8777-777777777777",
    "name": "Torrent Power Ltd",
    "status": "ACTIVE",
    "createdAt": "2026-08-08T20:00:00Z"
  }
  ```

### `DELETE /api/v1/admin/power-companies/{id}`
- **Access**: `SUPER_ADMIN` | `ADMIN`
- **Response (200 OK - When 0 linked buildings)**:
  ```json
  {
    "message": "Power supply company deleted successfully."
  }
  ```
- **Error Response (409 Conflict - When linked to ≥ 1 building)**:
  ```json
  {
    "statusCode": 409,
    "error": "COMPANY_IN_USE",
    "message": "Cannot delete or deactivate 'Uttarakhand Power Corporation Limited (UPCL)' because 2 buildings are currently linked to it. Reassign or remove these buildings first.",
    "metadata": {
      "companyId": "11111111-1111-4111-8111-111111111111",
      "companyName": "Uttarakhand Power Corporation Limited (UPCL)",
      "linkedBuildingCount": 2,
      "affectedBuildings": [
        {
          "buildingId": "b1111111-1111-4111-8111-111111111111",
          "buildingName": "Sunshine Heights",
          "landlord": {
            "landlordId": "u1000000-0000-4000-8000-000000000010",
            "fullName": "Rajesh Kumar",
            "email": "rajesh.landlord@example.com"
          }
        }
      ]
    }
  }
  ```

---

## 4. Power Supplier Master Bill Endpoints

### `POST /api/v1/buildings/{buildingId}/supplier-master-bills`
- **Access**: `LANDLORD` (Own Buildings) | `ADMIN` | `SUPER_ADMIN`
- **Request Body**:
  ```json
  {
    "powerCompanyId": "1",
    "billCycleStart": "2026-06-01",
    "billCycleEnd": "2026-06-30",
    "masterBillAmount": 28500.00,
    "tariffRatePerUnit": 7.00,
    "dueDate": "2026-07-20",
    "digitalBillDocId": "501"
  }
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
    "userId": "u1000000-0000-4000-8000-000000000010",
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
      "requestId": "req_99812-4444-8888-9999",
      "userId": "u4500000-0000-4000-8000-000000000045",
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
    "adminId": "a2000000-0000-4000-8000-000000000002",
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

## 4. Building, Connection & Power Supplier Master Bill Endpoints

### `POST /api/v1/buildings`
- **Access**: `LANDLORD` | `ADMIN` | `SUPER_ADMIN`
- **Request Body**:
  ```json
  {
    "name": "Sunshine Heights",
    "addressLine": "12 Rajpur Road",
    "city": "Dehradun",
    "state": "Uttarakhand",
    "pincode": "248001",
    "totalFloors": 4,
    "powerCompanyId": "11111111-1111-4111-8111-111111111111",
    "connectionNumber": "UPCL-CONN-10023"
  }
  ```
- **Response (201 Created)**:
  ```json
  {
    "buildingId": "b1111111-1111-4111-8111-111111111111",
    "name": "Sunshine Heights",
    "powerCompanyId": "11111111-1111-4111-8111-111111111111",
    "powerCompanyName": "Uttarakhand Power Corporation Limited (UPCL)",
    "connectionNumber": "UPCL-CONN-10023",
    "createdAt": "2026-08-08T20:00:00Z"
  }
  ```

### `POST /api/v1/buildings/{buildingId}/switch-power-supplier`
- **Access**: `LANDLORD` (Own Buildings) | `ADMIN` | `SUPER_ADMIN`
- **Purpose**: Switch a building's power supply company and connection number. Enforces Zero Open Dues rule.
- **Request Body**:
  ```json
  {
    "newPowerCompanyId": "2",
    "newConnectionNumber": "UPPCL-CONN-998822",
    "effectiveDate": "2026-09-01",
    "notes": "Switched from UPCL to UPPCL due to commercial tariff revision"
  }
  ```
- **Response (200 OK - Successful Switch)**:
  ```json
  {
    "buildingId": "101",
    "previousSupplier": {
      "powerCompanyId": "1",
      "companyName": "Uttarakhand Power Corporation Limited (UPCL)",
      "connectionNumber": "UPCL-CONN-10023",
      "terminatedAt": "2026-08-31"
    },
    "newSupplier": {
      "powerCompanyId": "2",
      "companyName": "Uttar Pradesh Power Corporation Limited (UPPCL)",
      "connectionNumber": "UPPCL-CONN-998822",
      "effectiveFrom": "2026-09-01",
      "status": "ACTIVE"
    }
  }
  ```
- **Error Response (409 Conflict - When Unpaid Bills Exist)**:
  ```json
  {
    "statusCode": 409,
    "error": "PENDING_SUPPLIER_BILLS_EXIST",
    "message": "Cannot switch power supplier. There are 2 unpaid supplier master bills for the current supplier 'UPCL'. All pending bills must be settled first.",
    "metadata": {
      "currentPowerCompanyId": "1",
      "currentPowerCompanyName": "Uttarakhand Power Corporation Limited (UPCL)",
      "pendingBillCount": 2,
      "pendingBills": [
        { "billId": "501", "masterBillAmount": 28500.00, "dueDate": "2026-07-20", "status": "OVERDUE" },
        { "billId": "502", "masterBillAmount": 29100.00, "dueDate": "2026-08-20", "status": "UNPAID" }
      ]
    }
  }
  ```

### `GET /api/v1/buildings/{buildingId}/power-supplier-history`
- **Access**: `LANDLORD` (Own Buildings) | `ADMIN` | `SUPER_ADMIN`
- **Purpose**: Retrieve full historical audit log of power supplier connections for a building.
- **Response (200 OK)**:
  ```json
  [
    {
      "connectionId": "12",
      "powerCompanyId": "2",
      "companyName": "Uttar Pradesh Power Corporation Limited (UPPCL)",
      "connectionNumber": "UPPCL-CONN-998822",
      "startDate": "2026-09-01",
      "endDate": null,
      "status": "ACTIVE"
    },
    {
      "connectionId": "1",
      "powerCompanyId": "1",
      "companyName": "Uttarakhand Power Corporation Limited (UPCL)",
      "connectionNumber": "UPCL-CONN-10023",
      "startDate": "2025-04-01",
      "endDate": "2026-08-31",
      "status": "TERMINATED"
    }
  ]
  ```

### `POST /api/v1/buildings/{buildingId}/supplier-master-bills`
- **Access**: `LANDLORD` (Own Buildings) | `ADMIN` | `SUPER_ADMIN`
- **Request Body**:
  ```json
  {
    "powerCompanyId": "11111111-1111-4111-8111-111111111111",
    "connectionNumber": "UPCL-CONN-10023",
    "billSerialNumber": "UPCL-2026-06-88192",
    "billCycleStart": "2026-06-01",
    "billCycleEnd": "2026-06-30",
    "invoiceDate": "2026-07-02",
    "totalUnitsConsumed": 4071.42,
    "tariffRatePerUnit": 7.00,
    "masterBillAmount": 28500.00,
    "dueDate": "2026-07-20",
    "notes": "June master bill for Sunshine Heights",
    "digitalBillDocId": "d5010000-0000-4000-8000-000000000501"
  }
  ```
- **Response (201 Created)**:
  ```json
  {
    "billId": "m5010000-0000-4000-8000-000000000501",
    "buildingId": "b1111111-1111-4111-8111-111111111111",
    "powerCompanyId": "11111111-1111-4111-8111-111111111111",
    "billSerialNumber": "UPCL-2026-06-88192",
    "connectionNumber": "UPCL-CONN-10023",
    "masterBillAmount": 28500.00,
    "status": "UNPAID",
    "createdAt": "2026-07-02T10:00:00Z"
  }
  ```

### `PATCH /api/v1/buildings/{buildingId}/supplier-master-bills/{billId}`
- **Access**: `LANDLORD` (Own Buildings) | `ADMIN` | `SUPER_ADMIN`
- **Purpose**: Mark supplier master bill as paid. Single lump-sum settlement — no partial payments.
- **Request Body**:
  ```json
  {
    "status": "PAID",
    "paidDate": "2026-08-10T14:00:00Z",
    "paymentMode": "NEFT",
    "paymentReference": "NEFT/N1238491029",
    "notes": "Paid via NEFT to UPCL for July bill"
  }
  ```
- **Response (200 OK)**:
  ```json
  {
    "billId": "501",
    "status": "PAID",
    "paidDate": "2026-08-10T14:00:00Z",
    "paymentMode": "NEFT",
    "paymentReference": "NEFT/N1238491029",
    "updatedAt": "2026-08-10T14:02:33Z"
  }
  ```
- **Error**: HTTP 409 `RESOURCE_ALREADY_PAID` if bill is already `PAID`.

### `GET /api/v1/buildings/{buildingId}/electricity-reconciliation/{masterBillId}`
- **Access**: `LANDLORD` (Own Buildings) | `ADMIN` | `SUPER_ADMIN`
- **Purpose**: Retrieve detailed audit reconciliation breakdown comparing tenant electricity collections vs master bill paid.
- **Response (200 OK)**:
  ```json
  {
    "masterBillId": "m5010000-0000-4000-8000-000000000501",
    "buildingId": "b1111111-1111-4111-8111-111111111111",
    "buildingName": "Sunshine Heights",
    "powerCompanyName": "Uttarakhand Power Corporation Limited (UPCL)",
    "connectionNumber": "UPCL-CONN-10023",
    "masterBillCycle": {
      "startDate": "2026-02-24",
      "endDate": "2026-03-24"
    },
    "masterBillAmountPaid": 10000.00,
    "aggregatedTenantCollections": 10400.00,
    "varianceAmount": 400.00,
    "reconciliationStatus": "SURPLUS",
    "surplusAmount": 400.00,
    "deficitAmount": 0.00,
    "overlappingRoomCycles": [
      {
        "roomId": "r1010000-0000-4000-8000-000000000101",
        "roomName": "Room 01",
        "cycleStartDate": "2026-02-04",
        "cycleEndDate": "2026-03-04",
        "unitsConsumed": 300.00,
        "ratePerUnit": 8.00,
        "totalBilled": 2400.00,
        "amountCollected": 2400.00,
        "ledgerStatus": "PAID"
      },
      {
        "roomId": "r1020000-0000-4000-8000-000000000102",
        "roomName": "Room 02",
        "cycleStartDate": "2026-02-12",
        "cycleEndDate": "2026-03-12",
        "unitsConsumed": 450.00,
        "ratePerUnit": 8.00,
        "totalBilled": 3600.00,
        "amountCollected": 3600.00,
        "ledgerStatus": "PAID"
      },
      {
        "roomId": "r1030000-0000-4000-8000-000000000103",
        "roomName": "Room 03",
        "cycleStartDate": "2026-02-20",
        "cycleEndDate": "2026-03-20",
        "unitsConsumed": 550.00,
        "ratePerUnit": 8.00,
        "totalBilled": 4400.00,
        "amountCollected": 4400.00,
        "ledgerStatus": "PAID"
      }
    ],
    "reconciledAt": "2026-03-25T11:00:00Z"
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
    "documentId": "d5010000-0000-4000-8000-000000000501",
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

### `POST /api/v1/ledgers/room-rent/{ledgerId}/payments`
- **Access**: `LANDLORD` (Own Buildings) | `ADMIN` | `SUPER_ADMIN`
- **Purpose**: Record a payment transaction (partial or full) against a Room Rent Ledger. Multiple payments may be recorded per ledger.
- **Request Body**:
  ```json
  {
    "amountPaid": 1000.00,
    "paymentDate": "2026-08-03T10:30:00Z",
    "paymentMethod": "UPI",
    "transactionReference": "UPI/329482910",
    "notes": "Partial payment for August cycle"
  }
  ```
  > `paymentDate` is optional. If omitted, defaults to server UTC timestamp.
- **Response (201 Created)**:
  ```json
  {
    "transactionId": "701",
    "ledgerId": "301",
    "amountPaid": 1000.00,
    "runningTotal": 1000.00,
    "remainingBalance": 1500.00,
    "ledgerStatus": "PARTIALLY_PAID",
    "paymentDate": "2026-08-03T10:30:00Z",
    "recordedAt": "2026-08-03T10:32:11Z"
  }
  ```

### `GET /api/v1/ledgers/room-rent/{ledgerId}/payments`
- **Access**: `LANDLORD` (Own Buildings) | `ADMIN` | `SUPER_ADMIN` | `TENANT` (Own Ledger)
- **Purpose**: Retrieve all payment transactions for a Room Rent Ledger, sorted ascending by `paymentDate`.
- **Response (200 OK)**:
  ```json
  {
    "ledgerId": "301",
    "totalAmount": 2500.00,
    "amountPaid": 1000.00,
    "remainingBalance": 1500.00,
    "status": "PARTIALLY_PAID",
    "transactions": [
      {
        "transactionId": "701",
        "amountPaid": 1000.00,
        "paymentDate": "2026-08-03T10:30:00Z",
        "paymentMethod": "UPI",
        "transactionReference": "UPI/329482910",
        "notes": "Partial payment for August cycle",
        "recordedAt": "2026-08-03T10:32:11Z"
      }
    ]
  }
  ```

### `POST /api/v1/ledgers/electricity/{ledgerId}/payments`
- **Access**: `LANDLORD` (Own Buildings) | `ADMIN` | `SUPER_ADMIN`
- **Purpose**: Record a payment transaction (partial or full) against an Electricity Ledger.
- **Request Body**: *(Same shape as Room Rent payment above.)*
- **Response (201 Created)**: *(Same shape as Room Rent payment response above.)*

### `GET /api/v1/ledgers/electricity/{ledgerId}/payments`
- **Access**: `LANDLORD` (Own Buildings) | `ADMIN` | `SUPER_ADMIN` | `TENANT` (Own Ledger)
- **Purpose**: Retrieve all payment transactions for an Electricity Ledger.
- **Response (200 OK)**: *(Same shape as Room Rent GET payments response above.)*

### `PATCH /api/v1/ledgers/payments/{transactionId}`
- **Access**: `LANDLORD` (Own Buildings Only)
- **Restriction**: `SUPER_ADMIN` & `ADMIN` are strictly read-only (`HTTP 403 FORBIDDEN`). Allowed ONLY if `transactionId` is the chronologically latest transaction for its parent ledger (`recordedAt` max).
- **Purpose**: Update fields of the latest payment transaction to correct entry mistakes. Automatically recalculates parent ledger `amountPaid` and `status`.
- **Request Body**:
  ```json
  {
    "amountPaid": 1200.00,
    "paymentDate": "2026-08-03T10:30:00Z",
    "paymentMethod": "UPI",
    "transactionReference": "UPI/329482910_CORRECTED",
    "notes": "Corrected payment amount from ₹1000 to ₹1200"
  }
  ```
- **Response (200 OK - Successful Update & Balance Recalculation)**:
  ```json
  {
    "transactionId": "t7010000-0000-4000-8000-000000000701",
    "ledgerId": "rl301000-0000-4000-8000-000000000301",
    "amountPaid": 1200.00,
    "paymentDate": "2026-08-03T10:30:00Z",
    "paymentMethod": "UPI",
    "transactionReference": "UPI/329482910_CORRECTED",
    "notes": "Corrected payment amount from ₹1000 to ₹1200",
    "recalculatedLedger": {
      "ledgerId": "rl301000-0000-4000-8000-000000000301",
      "totalAmount": 2500.00,
      "amountPaid": 1200.00,
      "remainingBalance": 1300.00,
      "status": "PARTIALLY_PAID"
    },
    "updatedAt": "2026-08-09T15:00:00Z"
  }
  ```
- **Error Response (409 Conflict - Non-Latest Transaction Update Attempt)**:
  ```json
  {
    "statusCode": 409,
    "error": "NON_LAST_TRANSACTION_UPDATE_RESTRICTED",
    "message": "Cannot update payment transaction. Only the chronologically latest payment transaction for a ledger can be modified.",
    "metadata": {
      "attemptedTransactionId": "t7000000-0000-4000-8000-000000000700",
      "latestTransactionId": "t7010000-0000-4000-8000-000000000701",
      "ledgerId": "rl301000-0000-4000-8000-000000000301"
    }
  }
  ```

### `GET /api/v1/rooms/{roomId}/outstanding-balance`
- **Access**: `LANDLORD` (Own Buildings) | `ADMIN` | `SUPER_ADMIN`
- **Purpose**: Returns total outstanding balance for a room: `SUM(amount − amountPaid)` across all non-`PAID` rent and electricity ledger entries, with per-cycle breakdown.
- **Response (200 OK)**:
  ```json
  {
    "roomId": "r1010000-0000-4000-8000-000000000101",
    "roomName": "Room 11",
    "totalOutstanding": 3200.00,
    "cycles": [
      {
        "billingCycleId": "bc201000-0000-4000-8000-000000000201",
        "cycleStart": "2026-06-01",
        "cycleEnd": "2026-06-30",
        "rent": {
          "ledgerId": "rl301000-0000-4000-8000-000000000301",
          "amount": 2500.00,
          "amountPaid": 1000.00,
          "pending": 1500.00,
          "status": "PARTIALLY_PAID"
        },
        "electricity": {
          "ledgerId": "el401000-0000-4000-8000-000000000401",
          "amount": 1700.00,
          "amountPaid": 0.00,
          "pending": 1700.00,
          "status": "OVERDUE"
        }
      }
    ]
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

---

## 8. Session Management & Centralized Audit Log Endpoints

### `POST /api/v1/sessions/force-logout/user/{targetUserId}`
- **Access**: `SUPER_ADMIN` (All roles) | `ADMIN` (Landlords & Tenants Only)
- **Purpose**: Forcefully terminate all active multi-device sessions for a specified target user account.
- **Request Body**:
  ```json
  {
    "reason": "Security compromise flagged by system admin"
  }
  ```
- **Response (200 OK - Successful Termination)**:
  ```json
  {
    "message": "All active sessions for user 'Rajesh Kumar' forcefully terminated.",
    "targetUserId": "u1000000-0000-4000-8000-000000000010",
    "targetUserRole": "LANDLORD",
    "terminatedSessionCount": 3,
    "auditLogId": "a9010000-0000-4000-8000-000000000901"
  }
  ```
- **Error Response (403 Forbidden - Role Hierarchy Violation)**:
  ```json
  {
    "statusCode": 403,
    "error": "ROLE_HIERARCHY_VIOLATION",
    "message": "Permission denied. Admins cannot forcefully terminate sessions of a Super Admin or fellow Admin user account.",
    "metadata": {
      "actingUserRole": "ADMIN",
      "targetUserRole": "SUPER_ADMIN",
      "targetUserId": "u0000000-0000-4000-8000-000000000001"
    }
  }
  ```

### `POST /api/v1/sessions/force-logout/role/{targetRole}`
- **Access**: `SUPER_ADMIN` Only
- **Purpose**: Bulk force-logout all active user sessions system-wide for an entire role scope (`ADMIN`, `LANDLORD`, or `TENANT`).
- **Request Body**:
  ```json
  {
    "reason": "Platform security compliance patch enforcement"
  }
  ```
- **Response (200 OK)**:
  ```json
  {
    "message": "Bulk force logout executed. All active sessions for role 'LANDLORD' purged.",
    "targetRole": "LANDLORD",
    "affectedUserCount": 42,
    "terminatedSessionCount": 87,
    "auditLogId": "a9020000-0000-4000-8000-000000000902"
  }
  ```

### `GET /api/v1/admin/audit-logs`
- **Access**: `SUPER_ADMIN` | `ADMIN`
- **Purpose**: Search, filter, and paginate immutable system audit logs.
- **Query Parameters**:
  - `page`: default `1`
  - `limit`: default `20` (max `100`)
  - `category`: optional (`SESSION` \| `AUTH` \| `USER_MANAGEMENT` \| `ASSET_MANAGEMENT` \| `TENANT_MANAGEMENT` \| `FINANCIAL` \| `SYSTEM`)
  - `actionType`: optional (e.g. `FORCE_LOGOUT_USER`, `SAVE_PAYMENT_TRANSACTION`)
  - `performedByUserId`: optional UUID
  - `targetEntityId`: optional UUID/String
  - `startDate`: optional ISO 8601
  - `endDate`: optional ISO 8601
- **Response (200 OK)**:
  ```json
  {
    "data": [
      {
        "id": "a9010000-0000-4000-8000-000000000901",
        "actionType": "FORCE_LOGOUT_USER",
        "category": "SESSION",
        "performedBy": {
          "userId": "a2000000-0000-4000-8000-000000000002",
          "fullName": "Vikram Singh",
          "role": "ADMIN"
        },
        "targetEntity": {
          "id": "u1000000-0000-4000-8000-000000000010",
          "type": "USER"
        },
        "ipAddress": "103.21.124.8",
        "userAgent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)",
        "metadata": {
          "targetUserRole": "LANDLORD",
          "targetUserName": "Rajesh Kumar",
          "terminatedSessionCount": 3,
          "reason": "Security compromise flagged by system admin"
        },
        "createdAt": "2026-08-09T14:30:00Z"
      }
    ],
    "meta": {
      "currentPage": 1,
      "limit": 20,
      "totalRecords": 142,
      "totalPages": 8
    }
  }
  ```

