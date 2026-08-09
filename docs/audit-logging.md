# Centralized Immutable Audit Logging Registry

## System: My Room Ledger (NestJS Backend API)

---

## 1. System Invariants & Immutability Rules

1. **Insert-Only Operations**: Audit logs are strictly immutable. Once an `AuditLog` row is created, it **CANNOT be updated, altered, or deleted** by any application endpoint, service, or database user.
2. **Synchronous/Transactional Logging**: All state-changing API operations (`POST`, `PATCH`, `PUT`, `DELETE`) and critical security events MUST emit an `AuditLog` entry in the same database transaction context.
3. **Structured Context**: Every audit entry captures:
   - `actionType`: A standardized SCREAMING_SNAKE_CASE action identifier from Section 3.
   - `category`: The functional module identifier (`SESSION`, `AUTH`, `USER_MANAGEMENT`, `ASSET_MANAGEMENT`, `TENANT_MANAGEMENT`, `FINANCIAL`, `SYSTEM`).
   - `performedByUserId`: UUID of the acting user.
   - `performedByUserRole`: Role of the acting user (`SUPER_ADMIN`, `ADMIN`, `LANDLORD`, `TENANT`).
   - `targetEntityId`: UUID or unique identifier of the target resource (if applicable).
   - `targetEntityType`: Entity discriminator (`USER`, `BUILDING`, `TENANT`, `ROOM`, `LEDGER`, `POWER_COMPANY`, `DOCUMENT`).
   - `ipAddress` & `userAgent`: Client environment details.
   - `metadata`: JSON payload containing structured contextual details (before/after states, parameters, amounts, reason notes).
4. **Central Registry Synchronization**: As new features and actions are added to the platform, this document MUST be updated with the new `actionType` string, category, and metadata payload specification.

---

## 2. Database Schema Reference (`AuditLog` Model)

```prisma
model AuditLog {
  id                  String   @id @default(uuid()) @db.Uuid
  actionType          String   @map("action_type") @db.VarChar(100)
  category            String   @db.VarChar(50)
  performedByUserId   String   @map("performed_by_user_id") @db.Uuid
  performedByUserRole UserRole @map("performed_by_user_role")
  targetEntityId      String?  @map("target_entity_id") @db.VarChar(100)
  targetEntityType    String?  @map("target_entity_type") @db.VarChar(50)
  ipAddress           String?  @map("ip_address") @db.VarChar(45)
  userAgent           String?  @map("user_agent") @db.Text
  metadata            Json?    @db.JsonB
  createdAt           DateTime @default(now()) @map("created_at")

  performedByUser     User     @relation("AuditLogsPerformed", fields: [performedByUserId], references: [id], onDelete: Restrict)

  @@index([actionType])
  @@index([category])
  @@index([performedByUserId])
  @@index([targetEntityId])
  @@index([createdAt])
  @@map("audit_logs")
}
```

---

## 3. Categorized Audit Action Type Registry

### 3.1 `SESSION` Category (Session Control & Force Logout)

| Action Type | Description & Trigger | Target Entity Type | Expected Metadata Payload Keys |
|---|---|---|---|
| `FORCE_LOGOUT_USER` | Admin or Super Admin forcefully terminates all active sessions of a specific user. | `USER` | `targetUserId`, `targetUserRole`, `targetUserName`, `terminatedSessionCount`, `reason` |
| `FORCE_LOGOUT_ROLE` | Super Admin forcefully terminates all active sessions for an entire role scope. | `ROLE` | `targetRole`, `terminatedUserCount`, `terminatedSessionCount`, `reason` |
| `TERMINATE_SESSION` | User or Admin terminates a specific device session. | `SESSION` | `sessionId`, `deviceName`, `isSelfTerminated` |
| `PASSWORD_RESET_SESSION_REVOCATION` | System purges all sessions on password update or reset. | `USER` | `targetUserId`, `revokedSessionCount`, `trigger` |

---

### 3.2 `AUTH` Category (Authentication & Credentials)

| Action Type | Description & Trigger | Target Entity Type | Expected Metadata Payload Keys |
|---|---|---|---|
| `USER_LOGIN` | User completes login via Email or Phone. | `USER` | `loginType`, `dialCode`, `deviceId`, `mustChangePassword` |
| `USER_LOGOUT` | User explicitly logs out of current device session. | `USER` | `sessionId`, `deviceName` |
| `CHANGE_PASSWORD` | User updates password from profile or first-login force screen. | `USER` | `mustChangePasswordReset` |
| `PASSWORD_RESET_REQUEST` | User submits password reset request for Admin review. | `PASSWORD_RESET_REQUEST` | `requestId`, `deliveryChannel` |
| `PASSWORD_RESET_APPROVE` | Admin approves password reset request. | `PASSWORD_RESET_REQUEST` | `requestId`, `deliveryChannel`, `tokenExpiryMinutes` |

---

### 3.3 `USER_MANAGEMENT` Category (User Account Lifecycle)

| Action Type | Description & Trigger | Target Entity Type | Expected Metadata Payload Keys |
|---|---|---|---|
| `REGISTER_ADMIN` | Super Admin registers a new Admin account. | `USER` | `newUserId`, `fullName`, `email`, `role: "ADMIN"` |
| `REGISTER_LANDLORD` | Admin or Super Admin registers a new Landlord account. | `USER` | `newUserId`, `fullName`, `email`, `phoneNumber`, `role: "LANDLORD"` |
| `DEACTIVATE_USER` | Admin or Super Admin deactivates a user account. | `USER` | `targetUserId`, `targetRole`, `reason` |
| `UPDATE_USER_PROFILE` | Admin or user updates profile information. | `USER` | `targetUserId`, `updatedFields[]` |

---

### 3.4 `ASSET_MANAGEMENT` Category (Buildings, Infrastructure & Suppliers)

| Action Type | Description & Trigger | Target Entity Type | Expected Metadata Payload Keys |
|---|---|---|---|
| `REGISTER_BUILDING` | Landlord registers a new building with mandatory connection number. | `BUILDING` | `buildingId`, `buildingName`, `powerCompanyId`, `connectionNumber`, `totalFloors` |
| `UPDATE_BUILDING` | Landlord updates building details. | `BUILDING` | `buildingId`, `buildingName`, `updatedFields[]` |
| `SWITCH_POWER_SUPPLIER` | Landlord switches building power supplier & connection number. | `BUILDING` | `buildingId`, `previousPowerCompanyId`, `previousConnectionNumber`, `newPowerCompanyId`, `newConnectionNumber`, `effectiveDate` |
| `ADD_FLOOR` | Landlord adds a floor to a building. | `FLOOR` | `floorId`, `buildingId`, `floorNumber`, `name` |
| `ADD_ROOM` | Landlord adds a room to a floor. | `ROOM` | `roomId`, `floorId`, `roomNumber`, `baseRentAmount`, `bathroomType` |
| `ADD_SHARED_FACILITY` | Landlord adds shared bathroom/toilet to a floor. | `SHARED_FACILITY` | `facilityId`, `floorId`, `facilityType`, `label` |

---

### 3.5 `TENANT_MANAGEMENT` Category (Tenant Lifecycle & Occupancy)

| Action Type | Description & Trigger | Target Entity Type | Expected Metadata Payload Keys |
|---|---|---|---|
| `REGISTER_TENANT` | Landlord onboards a new tenant to a room. | `TENANT` | `tenantId`, `userId`, `roomId`, `fullName`, `checkInDate`, `rentAmount` |
| `CHECKOUT_TENANT` | Landlord checks out a tenant from a room. | `TENANT` | `tenantId`, `roomId`, `checkOutDate`, `outstandingBalanceAtCheckout` |
| `UPDATE_TENANT` | Landlord updates tenant contact or ID metadata. | `TENANT` | `tenantId`, `updatedFields[]` |

---

### 3.6 `FINANCIAL` Category (Rent, Electricity, Master Bills & Expenses)

| Action Type | Description & Trigger | Target Entity Type | Expected Metadata Payload Keys |
|---|---|---|---|
| `SAVE_PAYMENT_TRANSACTION` | Payment recorded against rent or electricity ledger. | `LEDGER` | `transactionId`, `ledgerId`, `ledgerType`, `amountPaid`, `paymentDate`, `paymentMethod`, `newStatus`, `remainingBalance` |
| `GENERATE_BILLING_CYCLE` | Rent & electricity billing cycle created for a room. | `BILLING_CYCLE` | `billingCycleId`, `roomId`, `cycleStart`, `cycleEnd`, `rentAmount`, `electricityAmount` |
| `CREATE_MASTER_BILL` | Landlord enters supplier master bill for a building. | `SUPPLIER_MASTER_BILL` | `billId`, `buildingId`, `powerCompanyId`, `billSerialNumber`, `connectionNumber`, `masterBillAmount`, `dueDate` |
| `SETTLE_MASTER_BILL` | Landlord marks supplier master bill as PAID. | `SUPPLIER_MASTER_BILL` | `billId`, `buildingId`, `masterBillAmount`, `paidDate`, `paymentMode`, `paymentReference` |
| `LOG_BUILDING_EXPENSE` | Landlord logs a building operating expense. | `BUILDING_EXPENSE` | `expenseId`, `buildingId`, `category`, `title`, `amount`, `expenseDate` |

---

### 3.7 `SYSTEM` Category (Platform Governance & Files)

| Action Type | Description & Trigger | Target Entity Type | Expected Metadata Payload Keys |
|---|---|---|---|
| `CREATE_POWER_COMPANY` | Admin creates a new power supply company. | `POWER_COMPANY` | `companyId`, `name`, `status` |
| `UPDATE_POWER_COMPANY` | Admin updates a power company's name or status. | `POWER_COMPANY` | `companyId`, `name`, `status` |
| `DELETE_POWER_COMPANY` | Admin deletes an unused power company. | `POWER_COMPANY` | `companyId`, `name` |
| `UPLOAD_FILE` | User uploads an encrypted file document. | `DOCUMENT` | `documentId`, `category`, `originalFileName`, `mimeType`, `fileSizeBytes` |
