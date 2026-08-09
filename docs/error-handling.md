# Backend Error Handling Specification & Code Registry

## System: My Room Ledger (NestJS Backend API)

---

## 1. Global Error Handling Philosophy & Security Principles

1. **Universal Envelope**: Every non-2xx API response (excluding raw file streams) returns a standardized JSON structure.
2. **Predictable & Machine-Readable**: Client application logic inspects `error` (SCREAMING_SNAKE_CASE code) and `statusCode` (integer HTTP code) to determine UI state, modal triggers, or navigation.
3. **Human-Readable & Actionable**: The `message` string clearly explains what failed AND what action the user or developer should take.
4. **Contextual Detail**: Structural context (e.g. affected buildings, linked entity IDs, field validation failures) is placed in `metadata` or `details` — never embedded as unparsed text inside `message`.
5. **Zero Information Leakage**: Stack traces, raw SQL queries, internal filesystem paths, and database constraint names are strictly stripped by NestJS global Exception Filters and loggers in production.

---

## 2. Universal Error Response Envelopes

### 2.1 Standard Non-Validation Error Response
Used for authentication failures, authorization blocks, state conflicts, resource missing errors, rate limiting, and internal server errors:

```json
{
  "statusCode": 409,
  "error": "COMPANY_IN_USE",
  "message": "Cannot delete 'Uttarakhand Power Corporation Limited (UPCL)' because 3 buildings are currently linked to it. Reassign these buildings to another power company first.",
  "metadata": {
    "companyId": "11111111-1111-4111-8111-111111111111",
    "companyName": "Uttarakhand Power Corporation Limited (UPCL)",
    "linkedBuildingCount": 3,
    "affectedBuildings": [
      {
        "buildingId": "b1111111-1111-4111-8111-111111111111",
        "buildingName": "Sunshine Heights",
        "landlord": {
          "landlordId": "u1000000-0000-4000-8000-000000000010",
          "fullName": "Rajesh Kumar",
          "email": "rajesh@example.com"
        }
      }
    ]
  }
}
```

> **Rules**:
> - `statusCode`: Matches the HTTP status code in the response header.
> - `error`: Unique, immutable SCREAMING_SNAKE_CASE string registered in Section 3.
> - `message`: Clear, user-facing actionable string.
> - `metadata`: JSON object containing contextual data. If no context exists, `metadata` defaults to `{}` (never `null` or omitted).

---

### 2.2 Validation Error Response (HTTP 400 `VALIDATION_ERROR`)
Triggered when request payload (`req.body`), URL parameters (`req.params`), or query strings (`req.query`) fail Zod schema validation (`ZodValidationPipe`):

```json
{
  "statusCode": 400,
  "error": "VALIDATION_ERROR",
  "message": "Input validation failed. Please correct the highlighted fields.",
  "details": [
    {
      "field": "phoneNumber",
      "message": "Phone number must be 10 digits starting with 6, 7, 8, or 9 for dial code +91 (India)"
    },
    {
      "field": "password",
      "message": "Password must be at least 8 characters long and contain at least 1 uppercase letter, 1 lowercase letter, 1 digit, and 1 special character"
    }
  ]
}
```

> **Rules**:
> - `details`: An array of field-level failure objects.
> - Each object contains `field` (JSON key name or path) and `message` (actionable validation rule description).

---

## 3. Complete Error Code Registry

| Error Code | HTTP Status | Category | Description & Trigger Scenario |
|---|---|---|---|
| `VALIDATION_ERROR` | `400` | Input | One or more fields in request body, query params, or route parameters failed Zod validation. Returns `details[]`. |
| `INVALID_CURRENT_PASSWORD` | `400` | Auth | Change password request supplied an incorrect old password. |
| `PASSWORD_MISMATCH` | `400` | Auth | New password and confirm password fields do not match. |
| `PASSWORD_REUSE_NOT_ALLOWED` | `400` | Auth | New password matches one of the user's last 3 password hashes. |
| `TOKEN_ALREADY_USED` | `400` | Auth | Password reset link token has already been consumed. |
| `PAYMENT_EXCEEDS_BALANCE` | `400` | Ledger | Recorded payment amount exceeds the total remaining balance of the ledger entry. |
| `INVALID_BILLING_CYCLE_DATES` | `400` | Billing | Billing cycle start date is on or after end date, or overlaps an existing cycle for the room. |
| `UNAUTHORIZED` | `401` | Auth | Request is missing a JWT access token, or token signature is invalid. |
| `INVALID_CREDENTIALS` | `401` | Auth | Login failed due to wrong email/phone or incorrect password. |
| `TOKEN_EXPIRED` | `401` | Auth | JWT access token or password reset link token has expired. |
| `INVALID_TOKEN` | `401` | Auth | JWT access token or refresh token is malformed, revoked, or tampered with. |
| `FORBIDDEN` | `403` | Auth | User is authenticated but lacks the required role or ownership permissions for the target resource. |
| `MUST_CHANGE_PASSWORD` | `403` | Auth | Account is flagged `mustChangePassword = true`. User must change password before accessing endpoints. |
| `ROLE_HIERARCHY_VIOLATION` | `403` | Auth | Lower role user attempted an administrative or session force-logout operation on an equal or higher role account. |
| `NOT_FOUND` | `404` | Resource | Target resource ID does not exist in the database. |
| `DUPLICATE_ENTRY` | `409` | Conflict | Resource creation failed due to unique constraint violation (e.g. duplicate email, phone, or bill serial number). |
| `COMPANY_IN_USE` | `409` | Conflict | Attempted to delete or deactivate a `PowerSupplyCompany` linked to 1 or more buildings. Returns `metadata.affectedBuildings[]`. |
| `PENDING_SUPPLIER_BILLS_EXIST` | `409` | Conflict | Attempted to switch building power supplier while open (`UNPAID` / `OVERDUE`) master bills exist for current supplier. Returns `metadata.pendingBills[]`. |
| `RESOURCE_ALREADY_PAID` | `409` | Business | Attempted to mark an already `PAID` supplier master bill as `PAID`. |
| `LEDGER_ALREADY_EXISTS` | `409` | Business | Billing cycle generation attempted for a room that already has a ledger for that cycle. |
| `TOO_MANY_REQUESTS` | `429` | Security | Rate limit exceeded (e.g. >5 failed logins from same IP in 15 minutes). |
| `INTERNAL_SERVER_ERROR` | `500` | System | Unhandled server exception. Generic message returned; stack trace logged internally. |

---

## 4. Specific Complex Error Examples

### 4.1 `COMPANY_IN_USE` (HTTP 409 Conflict)
Returned when an Admin attempts `DELETE /api/v1/admin/power-companies/{id}` or deactivation for a company linked to buildings:

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
      },
      {
        "buildingId": "b2222222-2222-4222-8222-222222222222",
        "buildingName": "Green Valley Residency",
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

### 4.2 `PENDING_SUPPLIER_BILLS_EXIST` (HTTP 409 Conflict)
Returned when a Landlord attempts `POST /api/v1/buildings/{id}/switch-power-supplier` while open bills exist for the current supplier:

```json
{
  "statusCode": 409,
  "error": "PENDING_SUPPLIER_BILLS_EXIST",
  "message": "Cannot switch power supplier. There are 2 unpaid supplier master bills for the current supplier 'UPCL'. All pending bills must be settled first.",
  "metadata": {
    "currentPowerCompanyId": "11111111-1111-4111-8111-111111111111",
    "currentPowerCompanyName": "Uttarakhand Power Corporation Limited (UPCL)",
    "pendingBillCount": 2,
    "pendingBills": [
      {
        "billId": "m5010000-0000-4000-8000-000000000501",
        "masterBillAmount": 28500.00,
        "amountPaid": 0.00,
        "dueDate": "2026-07-20",
        "status": "OVERDUE"
      },
      {
        "billId": "m5020000-0000-4000-8000-000000000502",
        "masterBillAmount": 29100.00,
        "amountPaid": 0.00,
        "dueDate": "2026-08-20",
        "status": "UNPAID"
      }
    ]
  }
}
```

---

### 4.3 `PAYMENT_EXCEEDS_BALANCE` (HTTP 400 Bad Request)
Returned when `POST /api/v1/ledgers/room-rent/{id}/payments` attempts to record an amount higher than remaining due:

```json
{
  "statusCode": 400,
  "error": "PAYMENT_EXCEEDS_BALANCE",
  "message": "Payment amount of ₹3,000.00 exceeds remaining balance of ₹1,500.00 for this ledger cycle.",
  "metadata": {
    "ledgerId": "rl301000-0000-4000-8000-000000000301",
    "totalAmount": 2500.00,
    "alreadyPaid": 1000.00,
    "remainingBalance": 1500.00,
    "attemptedPayment": 3000.00
  }
}
```

---

## 5. HTTP Status Code Mapping Matrix

| Status Code | Standard Usage Guidelines |
|---|---|
| `200 OK` | Successful retrieval (`GET`), update (`PATCH`/`PUT`), or non-creation execution (`POST approve`). |
| `201 Created` | Successful creation of a new entity (`POST`). |
| `400 Bad Request` | Syntactic failure, Zod schema validation failure, or pre-condition violation (e.g. over-payment). |
| `401 Unauthorized` | Missing, expired, tampered, or invalid authentication credentials. |
| `403 Forbidden` | Authenticated user lacks permission, or account restricted by security policy (`mustChangePassword`). |
| `404 Not Found` | Requested resource ID does not exist in the system. |
| `409 Conflict` | Resource state conflict (unique constraint violation, entity in use, pending bill block, duplicate bill serial). |
| `429 Too Many Requests` | Throttling limit exceeded. Client must pause before retrying. |
| `500 Internal Error` | Unexpected application fault. Response sanitized to prevent information exposure. |
