# 🗄️ PRODUCTION DATABASE SCHEMA & PRISMA SPECIFICATION
## System: My Room Ledger (PostgreSQL + Prisma ORM Engine)

---

## 1. ER Diagram & Relational Overview

```
                         +-------------------+
                         |       USERS       | (SUPER_ADMIN, ADMIN, LANDLORD, TENANT)
                         +---------+---------+
                                   |
              +--------------------+--------------------+
              | (Admin -> Landlord)                     | (Landlord -> Buildings)
              v                                         v
   +--------------------+                     +-------------------+
   |  ADMIN_AUDIT_LOGS  |                     |     BUILDINGS     |
   +--------------------+                     +----+----+----+----+
                                                   |    |    |
                      +----------------------------+    |    +-------------------------+
                      | (1:N)                           | (1:N)                        | (1:N)
                      v                                 v                              v
            +-------------------+             +-------------------+          +-------------------+
            | BUILDING_EXPENSES |             |SUPPLIER_MASTER_BIL|          |      FLOORS       |
            +-------------------+             +-------------------+          +----+----+----+----+
                                                                                  |    |    |
                                      +-------------------------------------------+    |    +--------------------+
                                      | (1:N)                                          | (1:N)                   | (1:N)
                                      v                                                v                         v
                              +---------------+                               +------------------+     +-------------------+
                              |     ROOMS     |                               | SHARED_BATHROOMS |     |  SHARED_TOILETS   |
                              +-------+-------+                               +------------------+     +-------------------+
                                      |
                     +----------------+----------------+
                     | (1:N)                           | (1:N)
                     v                                 v
             +---------------+               +-------------------+
             |    TENANTS    |               | RENT_CYCLE_CONFIGS|
             +-------+-------+               +---------+---------+
                     |                                 |
                     | (Snapshot)                      | (1:N)
                     v                                 v
             +---------------+               +-------------------+
             |TENANCY_HISTORY|               |  BILLING_CYCLES   |
             +---------------+               +----+----+---------+
                                                  |    |
                  +-------------------------------+    +-------------------------------+
                  | (1:1)                                                              | (1:1)
                  v                                                                    v
     +-----------------------+                                            +-----------------------+
     |   ROOM_RENT_LEDGERS   |                                            |  ELECTRICITY_LEDGERS  |
     | (Independent Status)  |                                            | (Submeter Calculation)|
     +-----------------------+                                            +-----------------------+
```

---

## 2. Declarative Prisma Schema Definition (`schema.prisma`)

```prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

// ENUM DEFINITIONS
enum UserRole {
  SUPER_ADMIN
  ADMIN
  LANDLORD
  TENANT
}

enum OccupancyType {
  SINGLE
  DOUBLE
  TRIPLE
  DORMITORY
}

enum FacilityAccessType {
  PRIVATE
  SHARED
}

enum FacilityStatus {
  FUNCTIONAL
  UNDER_MAINTENANCE
  OUT_OF_SERVICE
}

enum TenantStatus {
  ACTIVE
  NOTICE_PERIOD
  MOVED_OUT
}

enum PaymentStatus {
  UNPAID
  PARTIALLY_PAID
  PAID
  OVERDUE
}

enum CompanyStatus {
  ACTIVE
  INACTIVE
}

enum LedgerType {
  ROOM_RENT
  ELECTRICITY
}

enum SupplierConnectionStatus {
  ACTIVE
  TERMINATED
}

enum ExpenseCategory {
  WATER_BILL
  COMMON_ELECTRICITY
  WATER_MOTOR_ELECTRICITY
  MAINTENANCE
  REPAIRS
  SECURITY
  CLEANING
  PROPERTY_TAX
  MISCELLANEOUS
}

enum ComplaintCategory {
  PLUMBING
  ELECTRICAL
  CLEANLINESS
  NOISE
  BILLING
  OTHER
}

enum ComplaintSeverity {
  LOW
  MEDIUM
  HIGH
  URGENT
}

enum RequestStatus {
  PENDING
  APPROVED
  REJECTED
}

enum ComplaintStatus {
  OPEN
  IN_PROGRESS
  RESOLVED
  REJECTED
}


// MODEL DEFINITIONS
// -----------------------------------------------------------------------------
// PRIMARY KEY STRATEGY: All models use UUIDv7 (@default(dbgenerated("uuidv7()")) / @default(uuid()))
// Time-ordered 128-bit identifiers optimize B-Tree index locality & INSERT performance.
// PASSWORD HASHING: passwordHash stores Argon2id encoded hashes ($argon2id$).
// -----------------------------------------------------------------------------

model PowerSupplyCompany {
  id        String        @id @default(uuid()) @db.Uuid // UUIDv7 time-ordered PK
  name      String        @unique @db.VarChar(150) // e.g. UPCL, UPPCL, Reliance Power Ltd
  status    CompanyStatus @default(ACTIVE)
  createdAt DateTime      @default(now()) @map("created_at")
  updatedAt DateTime      @updatedAt @map("updated_at")

  buildings        Building[]
  supplierBills    SupplierMasterBill[]
  powerConnections BuildingPowerConnection[]

  @@index([status])
  @@map("power_supply_companies")
}

model CountryCode {
  id                 String   @id @default(uuid()) @db.Uuid
  countryCode        String   @unique @map("country_code") @db.VarChar(5) // ISO alpha-2 e.g. "IN"
  dialCode           String   @unique @map("dial_code") @db.VarChar(10)   // e.g. "+91"
  countryName        String   @map("country_name") @db.VarChar(100)      // e.g. "India"
  flagEmoji          String   @map("flag_emoji") @db.VarChar(10)         // e.g. "🇮🇳"
  phoneRegexPattern  String   @map("phone_regex_pattern") @db.VarChar(255) // e.g. "^[6-9]\\d{9}$"
  minLength          Int      @default(10) @map("min_length")
  maxLength          Int      @default(10) @map("max_length")
  isDefault          Boolean  @default(false) @map("is_default")         // True for +91 India
  isActive           Boolean  @default(true) @map("is_active")
  createdAt          DateTime @default(now()) @map("created_at")
  updatedAt          DateTime @updatedAt @map("updated_at")

  users              User[]   @relation("UserCountryCode")

  @@index([dialCode])
  @@map("country_codes")
}

model User {
  id                 String      @id @default(uuid()) @db.Uuid // UUIDv7 time-ordered PK
  fullName           String      @map("full_name") @db.VarChar(120)
  email              String?     @unique @db.VarChar(150)
  countryCodeId      String      @map("country_code_id") @db.Uuid
  phoneNumber        String      @unique @map("phone_number") @db.VarChar(20)
  passwordHash       String      @map("password_hash") @db.VarChar(255) // Argon2id hash ($argon2id$)
  role               UserRole    @default(TENANT)
  mustChangePassword Boolean     @default(false) @map("must_change_password")
  isSystemActive     Boolean     @default(true) @map("is_system_active")
  createdAt          DateTime    @default(now()) @map("created_at")
  updatedAt          DateTime    @updatedAt @map("updated_at")

  // Relationships
  countryCode        CountryCode @relation("UserCountryCode", fields: [countryCodeId], references: [id], onDelete: Restrict)
  buildings          Building[]  @relation("LandlordBuildings")
  tenantProfile      Tenant?
  documentsUploaded  DocumentMetadata[] @relation("UploadedDocuments")
  complaints         Complaint[] @relation("TenantComplaints")
  resetRequests      PasswordResetRequest[] @relation("UserResetRequests")
  sessions           UserSession[]
  auditLogsPerformed AuditLog[]  @relation("AuditLogsPerformed")

  @@index([countryCodeId])
  @@map("users")
}

model PasswordResetRequest {
  id                 String        @id @default(uuid()) @db.Uuid
  userId             String        @map("user_id") @db.Uuid
  status             RequestStatus @default(PENDING)
  deliveryChannel    String        @map("delivery_channel") @db.VarChar(50) // EMAIL, ADMIN_MODAL_DISPLAY, SMS
  approvedByUserId   String?       @map("approved_by_user_id") @db.Uuid
  rejectionReason    String?       @map("rejection_reason") @db.Text
  createdAt          DateTime      @default(now()) @map("created_at")
  updatedAt          DateTime      @updatedAt @map("updated_at")

  user               User          @relation("UserResetRequests", fields: [userId], references: [id], onDelete: Cascade)

  @@index([status])
  @@index([userId])
  @@map("password_reset_requests")
}

model Building {
  id               String             @id @default(uuid()) @db.Uuid
  landlordId       String             @map("landlord_id") @db.Uuid
  powerCompanyId   String             @map("power_company_id") @db.Uuid
  connectionNumber String             @map("connection_number") @db.VarChar(100) // Currently active connection number
  name             String             @db.VarChar(150)
  addressLine      String             @map("address_line") @db.Text
  city             String             @db.VarChar(100)
  state            String             @db.VarChar(100)
  pincode          String             @db.VarChar(10)
  totalFloors      Int                @map("total_floors")
  createdAt        DateTime           @default(now()) @map("created_at")
  updatedAt        DateTime           @updatedAt @map("updated_at")

  // Relationships
  landlord         User                      @relation("LandlordBuildings", fields: [landlordId], references: [id], onDelete: Cascade)
  powerCompany     PowerSupplyCompany        @relation(fields: [powerCompanyId], references: [id], onDelete: Restrict)
  floors           Floor[]
  expenses         BuildingExpense[]
  supplierBills    SupplierMasterBill[]
  powerConnections BuildingPowerConnection[]

  @@index([landlordId])
  @@index([powerCompanyId])
  @@map("buildings")
}

model BuildingPowerConnection {
  id               String                   @id @default(uuid()) @db.Uuid
  buildingId       String                   @map("building_id") @db.Uuid
  powerCompanyId   String                   @map("power_company_id") @db.Uuid
  connectionNumber String                   @map("connection_number") @db.VarChar(100)
  startDate        DateTime                 @map("start_date") @db.Date
  endDate          DateTime?                @map("end_date") @db.Date
  status           SupplierConnectionStatus @default(ACTIVE)
  notes            String?                  @db.Text
  createdAt        DateTime                 @default(now()) @map("created_at")

  building         Building                 @relation(fields: [buildingId], references: [id], onDelete: Cascade)
  powerCompany     PowerSupplyCompany       @relation(fields: [powerCompanyId], references: [id], onDelete: Restrict)

  @@unique([powerCompanyId, connectionNumber])
  @@index([buildingId, status])
  @@index([powerCompanyId])
  @@map("building_power_connections")
}

model Floor {
  id              String      @id @default(uuid()) @db.Uuid
  buildingId      String      @map("building_id") @db.Uuid
  floorNumber     Int         @map("floor_number") // 0 = Ground Floor
  name            String      @db.VarChar(50)
  createdAt       DateTime    @default(now()) @map("created_at")

  // Relationships
  building        Building    @relation(fields: [buildingId], references: [id], onDelete: Cascade)
  rooms           Room[]
  sharedBathrooms SharedBathroom[]
  sharedToilets   SharedToilet[]

  @@unique([buildingId, floorNumber])
  @@index([buildingId])
  @@map("floors")
}

model Room {
  id              String             @id @default(uuid()) @db.Uuid
  floorId         String             @map("floor_id") @db.Uuid
  roomNumber      String             @map("room_number") @db.VarChar(20) // e.g. Room 01, Room 11
  occupancyType   OccupancyType      @default(SINGLE) @map("occupancy_type")
  baseRentAmount  Decimal            @map("base_rent_amount") @db.Decimal(10, 2)
  bathroomType    FacilityAccessType @default(SHARED) @map("bathroom_type")
  toiletType      FacilityAccessType @default(SHARED) @map("toilet_type")
  createdAt       DateTime           @default(now()) @map("created_at")
  updatedAt       DateTime           @updatedAt @map("updated_at")

  // Relationships
  floor           Floor              @relation(fields: [floorId], references: [id], onDelete: Cascade)
  tenants         Tenant[]
  tenancyHistory  TenancyHistory[]
  cycleConfigs    RentCycleConfig[]
  billingCycles   BillingCycle[]

  @@unique([floorId, roomNumber])
  @@index([floorId])
  @@map("rooms")
}

model SharedBathroom {
  id              String         @id @default(uuid()) @db.Uuid
  floorId         String         @map("floor_id") @db.Uuid
  bathNumber      String         @map("bath_number") @db.VarChar(20) // e.g. Bath 01, Bath 11
  status          FacilityStatus @default(FUNCTIONAL)
  createdAt       DateTime       @default(now()) @map("created_at")

  floor           Floor          @relation(fields: [floorId], references: [id], onDelete: Cascade)

  @@unique([floorId, bathNumber])
  @@map("shared_bathrooms")
}

model SharedToilet {
  id              String         @id @default(uuid()) @db.Uuid
  floorId         String         @map("floor_id") @db.Uuid
  toiletNumber    String         @map("toilet_number") @db.VarChar(20) // e.g. Toilet 01, Toilet 11
  status          FacilityStatus @default(FUNCTIONAL)
  createdAt       DateTime       @default(now()) @map("created_at")

  floor           Floor          @relation(fields: [floorId], references: [id], onDelete: Cascade)

  @@unique([floorId, toiletNumber])
  @@map("shared_toilets")
}

model Tenant {
  id               String       @id @default(uuid()) @db.Uuid
  userId           String       @unique @map("user_id") @db.Uuid
  currentRoomId    String?      @map("current_room_id") @db.Uuid
  emergencyContact String?      @map("emergency_contact") @db.VarChar(20)
  idProofType      String?      @map("id_proof_type") @db.VarChar(50)
  idProofNumber    String?      @map("id_proof_number") @db.VarChar(100)
  checkInDate      DateTime     @map("check_in_date") @db.Date
  status           TenantStatus @default(ACTIVE)
  createdAt        DateTime     @default(now()) @map("created_at")
  updatedAt        DateTime     @updatedAt @map("updated_at")

  // Relationships
  user             User         @relation(fields: [userId], references: [id], onDelete: Cascade)
  currentRoom      Room?        @relation(fields: [currentRoomId], references: [id], onDelete: SetNull)
  tenancyHistory   TenancyHistory[]
  billingSnapshots BillingCycleTenantsSnapshot[]

  @@index([currentRoomId])
  @@map("tenants")
}

model TenancyHistory {
  id            String    @id @default(uuid()) @db.Uuid
  tenantId      String    @map("tenant_id") @db.Uuid
  roomId        String    @map("room_id") @db.Uuid
  checkInDate   DateTime  @map("check_in_date") @db.Date
  checkOutDate  DateTime? @map("check_out_date") @db.Date
  moveOutReason String?   @map("move_out_reason") @db.Text
  createdAt     DateTime  @default(now()) @map("created_at")

  tenant        Tenant    @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  room          Room      @relation(fields: [roomId], references: [id], onDelete: Cascade)

  @@index([tenantId])
  @@index([roomId])
  @@map("tenancy_history")
}

model BuildingExpense {
  id          String          @id @default(uuid()) @db.Uuid
  buildingId  String          @map("building_id") @db.Uuid
  category    ExpenseCategory
  title       String          @db.VarChar(200)
  amount      Decimal         @db.Decimal(10, 2)
  expenseDate DateTime        @map("expense_date") @db.Date
  notes       String?         @db.Text
  createdAt   DateTime        @default(now()) @map("created_at")

  building    Building        @relation(fields: [buildingId], references: [id], onDelete: Cascade)

  @@index([buildingId, expenseDate])
  @@map("building_expenses")
}

model SupplierMasterBill {
  id                   String             @id @default(uuid()) @db.Uuid
  buildingId           String             @map("building_id") @db.Uuid
  powerCompanyId       String             @map("power_company_id") @db.Uuid

  // Identifiers
  billSerialNumber     String?            @map("bill_serial_number") @db.VarChar(100)   // Unique invoice/bill number from physical bill
  connectionNumber     String             @map("connection_number") @db.VarChar(100)    // Consumer account / K-number for this bill

  // Billing Cycle & Dates
  billCycleStart       DateTime           @map("bill_cycle_start") @db.Date
  billCycleEnd         DateTime           @map("bill_cycle_end") @db.Date
  invoiceDate          DateTime?          @map("invoice_date") @db.Date                 // Date utility company issued this bill

  // Consumption & Tariff Rate
  totalUnitsConsumed   Decimal?           @map("total_units_consumed") @db.Decimal(10, 2) // Total building kWh consumed
  tariffRatePerUnit    Decimal?           @map("tariff_rate_per_unit") @db.Decimal(8, 2)  // Utility company rate (e.g. ₹7.00/unit)

  // Financial Amounts & Variance
  masterBillAmount     Decimal            @map("master_bill_amount") @db.Decimal(10, 2)
  surplusAmount        Decimal?           @default(0.00) @map("surplus_amount") @db.Decimal(10, 2)
  deficitAmount        Decimal?           @default(0.00) @map("deficit_amount") @db.Decimal(10, 2)

  // Settlement Details
  amountPaid           Decimal            @default(0.00) @map("amount_paid") @db.Decimal(10, 2)
  status               PaymentStatus      @default(UNPAID)
  dueDate              DateTime           @map("due_date") @db.Date
  paidDate             DateTime?          @map("paid_date")
  paymentMode          String?            @map("payment_mode") @db.VarChar(50)          // NEFT | UPI | CHEQUE | ONLINE_PORTAL | CASH
  paymentReference     String?            @map("payment_reference") @db.VarChar(100)   // UTR / cheque number / transaction ID

  // Audit & Documents
  notes                String?            @db.Text
  digitalBillDocId     String?            @map("digital_bill_doc_id") @db.Uuid
  createdAt            DateTime           @default(now()) @map("created_at")

  building             Building           @relation(fields: [buildingId], references: [id], onDelete: Cascade)
  powerCompany         PowerSupplyCompany @relation(fields: [powerCompanyId], references: [id], onDelete: Restrict)
  digitalBillDoc       DocumentMetadata?  @relation(fields: [digitalBillDocId], references: [id])

  @@unique([powerCompanyId, billSerialNumber])                         // Serial number unique per power company
  @@unique([buildingId, powerCompanyId, billCycleStart, billCycleEnd]) // No duplicate cycle range per building+company
  @@index([buildingId, status])
  @@index([powerCompanyId])
  @@map("supplier_master_bills")
}

model RentCycleConfig {
  id             String    @id @default(uuid()) @db.Uuid
  roomId         String    @map("room_id") @db.Uuid
  cycleStartDay  Int       @map("cycle_start_day") // 1..31
  activeFromDate DateTime  @map("active_from_date") @db.Date
  activeToDate   DateTime? @map("active_to_date") @db.Date
  createdAt      DateTime  @default(now()) @map("created_at")

  room           Room      @relation(fields: [roomId], references: [id], onDelete: Cascade)
  billingCycles  BillingCycle[]

  @@index([roomId])
  @@map("rent_cycle_configs")
}

model BillingCycle {
  id                 String       @id @default(uuid()) @db.Uuid
  roomId             String       @map("room_id") @db.Uuid
  rentCycleConfigId  String       @map("rent_cycle_config_id") @db.Uuid
  cycleStartDate     DateTime     @map("cycle_start_date") @db.Date
  cycleEndDate       DateTime     @map("cycle_end_date") @db.Date
  createdAt          DateTime     @default(now()) @map("created_at")

  room               Room         @relation(fields: [roomId], references: [id], onDelete: Cascade)
  config             RentCycleConfig @relation(fields: [rentCycleConfigId], references: [id])
  tenantsSnapshot    BillingCycleTenantsSnapshot[]
  roomRentLedger     RoomRentLedger?
  electricityLedger  ElectricityLedger?

  @@unique([roomId, cycleStartDate, cycleEndDate])
  @@index([roomId])
  @@map("billing_cycles")
}

model BillingCycleTenantsSnapshot {
  id                 String       @id @default(uuid()) @db.Uuid
  billingCycleId     String       @map("billing_cycle_id") @db.Uuid
  tenantId           String       @map("tenant_id") @db.Uuid
  tenantNameSnapshot String       @map("tenant_name_snapshot") @db.VarChar(120)
  createdAt          DateTime     @default(now()) @map("created_at")

  billingCycle       BillingCycle @relation(fields: [billingCycleId], references: [id], onDelete: Cascade)
  tenant             Tenant       @relation(fields: [tenantId], references: [id], onDelete: Cascade)

  @@unique([billingCycleId, tenantId])
  @@map("billing_cycle_tenants_snapshot")
}

model RoomRentLedger {
  id                   String               @id @default(uuid()) @db.Uuid
  billingCycleId       String               @unique @map("billing_cycle_id") @db.Uuid
  amount               Decimal              @db.Decimal(10, 2)
  amountPaid           Decimal              @default(0.00) @map("amount_paid") @db.Decimal(10, 2) // Running sum of all PaymentTransactions
  status               PaymentStatus        @default(UNPAID)
  dueDate              DateTime             @map("due_date") @db.Date                              // = BillingCycle.cycleEndDate; OVERDUE when currentDate > dueDate
  paidDate             DateTime?            @map("paid_date")                                      // Set when status transitions to PAID
  notes                String?              @db.Text
  createdAt            DateTime             @default(now()) @map("created_at")

  billingCycle         BillingCycle         @relation(fields: [billingCycleId], references: [id], onDelete: Cascade)
  paymentTransactions  PaymentTransaction[]

  @@index([status])
  @@map("room_rent_ledgers")
}

model ElectricityLedger {
  id                   String               @id @default(uuid()) @db.Uuid
  billingCycleId       String               @unique @map("billing_cycle_id") @db.Uuid
  unitsConsumed        Decimal?             @map("units_consumed") @db.Decimal(10, 2)
  ratePerUnit          Decimal?             @map("rate_per_unit") @db.Decimal(8, 2)
  amount               Decimal              @db.Decimal(10, 2)                                      // Formula: Units * Rate
  amountPaid           Decimal              @default(0.00) @map("amount_paid") @db.Decimal(10, 2)  // Running sum of all PaymentTransactions
  status               PaymentStatus        @default(UNPAID)
  dueDate              DateTime             @map("due_date") @db.Date                               // = BillingCycle.cycleEndDate; OVERDUE when currentDate > dueDate
  paidDate             DateTime?            @map("paid_date")                                       // Set when status transitions to PAID
  notes                String?              @db.Text
  createdAt            DateTime             @default(now()) @map("created_at")

  billingCycle         BillingCycle         @relation(fields: [billingCycleId], references: [id], onDelete: Cascade)
  paymentTransactions  PaymentTransaction[]

  @@index([status])
  @@map("electricity_ledgers")
}

model PaymentTransaction {
  id                   String             @id @default(uuid()) @db.Uuid
  ledgerType           LedgerType         @map("ledger_type")                                        // Discriminator: ROOM_RENT | ELECTRICITY
  rentLedgerId         String?            @map("rent_ledger_id") @db.Uuid                            // Non-null when ledgerType = ROOM_RENT
  electricityLedgerId  String?            @map("electricity_ledger_id") @db.Uuid                     // Non-null when ledgerType = ELECTRICITY
  amountPaid           Decimal            @map("amount_paid") @db.Decimal(10, 2)                     // Must be > 0; validated at application layer
  paymentDate          DateTime           @map("payment_date")                                        // User-supplied or defaults to server UTC timestamp
  paymentMethod        String             @map("payment_method") @db.VarChar(50)                     // UPI | CASH | BANK_TRANSFER
  transactionReference String?            @map("transaction_reference") @db.VarChar(100)
  notes                String?            @db.Text
  recordedAt           DateTime           @default(now()) @map("recorded_at")                        // Immutable system creation timestamp

  rentLedger           RoomRentLedger?    @relation(fields: [rentLedgerId],        references: [id], onDelete: Cascade)
  electricityLedger    ElectricityLedger? @relation(fields: [electricityLedgerId], references: [id], onDelete: Cascade)

  // Application layer (Zod) enforces exactly one of rentLedgerId / electricityLedgerId is non-null, matching ledgerType
  @@index([rentLedgerId])
  @@index([electricityLedgerId])
  @@map("payment_transactions")
}

model DocumentMetadata {
  id                 String       @id @default(uuid()) @db.Uuid
  uploadedByUserId   String       @map("uploaded_by_user_id") @db.Uuid
  objectKey          String       @unique @map("object_key") @db.VarChar(255) // Cloudflare R2 object key
  encryptionIv       String       @map("encryption_iv") @db.VarChar(64)       // AES-256 GCM IV hex
  encryptionAuthTag  String       @map("encryption_auth_tag") @db.VarChar(64) // AES-256 Auth Tag
  originalFileName   String       @map("original_file_name") @db.VarChar(255)
  mimeType           String       @map("mime_type") @db.VarChar(100)
  fileSizeBytes      BigInt       @map("file_size_bytes")
  createdAt          DateTime     @default(now()) @map("created_at")

  uploadedByUser     User         @relation("UploadedDocuments", fields: [uploadedByUserId], references: [id])
  supplierBills      SupplierMasterBill[]

  @@map("document_metadata")
}

model Complaint {
  id             String            @id @default(uuid()) @db.Uuid
  tenantId       String            @map("tenant_id") @db.Uuid
  roomId         String            @map("room_id") @db.Uuid
  category       ComplaintCategory
  severity       ComplaintSeverity @default(MEDIUM)
  title          String            @db.VarChar(200)
  description    String            @db.Text
  status         ComplaintStatus   @default(OPEN)
  landlordNotes  String?           @map("landlord_notes") @db.Text
  resolvedAt     DateTime?         @map("resolved_at")
  createdAt      DateTime          @default(now()) @map("created_at")

  tenant         User              @relation("TenantComplaints", fields: [tenantId], references: [id], onDelete: Cascade)

  @@index([tenantId])
  @@index([status])
  @@map("complaints")
}

model UserSession {
  id           String   @id @default(uuid()) @db.Uuid
  userId       String   @map("user_id") @db.Uuid
  tokenHash    String   @unique @map("token_hash") @db.VarChar(255)
  deviceName   String?  @map("device_name") @db.VarChar(150)
  ipAddress    String?  @map("ip_address") @db.VarChar(45)
  userAgent    String?  @map("user_agent") @db.Text
  expiresAt    DateTime @map("expires_at")
  lastUsedAt   DateTime @default(now()) @map("last_used_at")
  createdAt    DateTime @default(now()) @map("created_at")

  user         User     @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId])
  @@index([expiresAt])
  @@map("user_sessions")
}

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
