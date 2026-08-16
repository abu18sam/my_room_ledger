# Flexible Billing Cycles & Landlord Electricity Reconciliation Engine

## System: My Room Ledger (NestJS Backend API)

---

## 1. Architectural Philosophy & Billing Flexibility

1. **Independent Per-Room Billing Cycles**:
   - Billing cycles for individual rooms within a building are **completely independent**.
   - Billing cycles are **non-calendar** and can start and end on **any day of the month** ($1..31$).
   - Within a single room, the **Room Rent Billing Cycle** and the **Electricity Billing Cycle** can operate on different start/end dates.
2. **Independent Building Master Electricity Cycle**:
   - Each building has a **master electricity billing cycle** set by the power supply company (e.g. UPCL, UPPCL, Reliance, Adani).
   - The building master bill cycle range (e.g. 24th of Month $N$ to 24th of Month $N+1$) is **completely decoupled** from individual room billing cycles.
3. **Multi-Cycle Overlap & Reconciliation**:
   - System allows multiple overlapping room billing cycles to co-exist without forcing alignment.
   - Landlord electricity reconciliation computes actual cash collected from tenants across overlapping room electricity ledgers during the building master bill's date window.
4. **Pass-Through Isolation Invariant**:
   - Tenant electricity payments collected are pass-through funds held to settle external utility provider bills.
   - Electricity collections, surplus, and deficits are **strictly segregated from Net Room Rent Profit**:
     $$\text{Net Profit} = (\text{Total Room Rent Collected}) - (\text{Building Operating Expenses})$$

---

## 2. 3-Step Landlord Electricity Reconciliation Engine

When a landlord settles a building's `SupplierMasterBill` for a specific billing cycle `[billCycleStart, billCycleEnd]`, the system automatically executes the following 3-step reconciliation algorithm:

```
[Supplier Master Bill Paid]
           │
           ▼
┌────────────────────────────────────────────────────────┐
│ Step 1: Identify Overlapping Room Electricity Ledgers  │
│         & Aggregate Tenant Cash Collected (amountPaid) │
└──────────────────────────┬─────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│ Step 2: Calculate Electricity Variance                 │
│         Variance = Tenant Collection - Master Bill Paid│
└──────────────────────────┬─────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│ Step 3: Classify Outcome & Record Financial Variance   │
│         ├─ Variance > 0 ──→ SURPLUS (surplusAmount)    │
│         ├─ Variance < 0 ──→ DEFICIT (deficitAmount)    │
│         └─ Variance = 0 ──→ BREAK_EVEN                 │
└────────────────────────────────────────────────────────┘
```

### Mathematical Formulas

#### Step 1: Tenant Collection Aggregation ($\text{TenantCollected}$)
$$\text{TenantCollected} = \sum_{i \in \text{Overlapping Ledgers}} \text{PaymentTransaction.amountPaid}_i$$
*Where an overlapping ledger is any `ElectricityLedger` whose `BillingCycle` intersects with `[billCycleStart, billCycleEnd]`.*

#### Step 2: Variance Calculation
$$\text{Variance} = \text{TenantCollected} - \text{MasterBillAmountPaid}$$

#### Step 3: Categorization Logic
$$\text{Reconciliation Outcome} = \begin{cases} \text{SURPLUS} & \text{if } \text{Variance} > 0 \\ \text{DEFICIT} & \text{if } \text{Variance} < 0 \\ \text{BREAK\_EVEN} & \text{if } \text{Variance} = 0 \end{cases}$$

---

## 3. Concrete Numerical Walkthrough

### Scenario Configuration
- **Building**: Sunshine Heights (Dehradun)
- **Power Company**: Uttarakhand Power Corporation Limited (UPCL)
- **Supplier Master Bill Cycle**: **Feb 24, 2026 → Mar 24, 2026**
- **Master Bill Amount Paid to UPCL**: **₹10,000.00**

#### Room 01 Configuration & Ledgers
- **Room Rent Cycle**: Feb 4, 2026 → Mar 4, 2026 (Rent: ₹8,000)
- **Electricity Cycle**: Feb 4, 2026 → Mar 4, 2026
  - Units Consumed: 300 kWh @ ₹8.00/unit = ₹2,400.00
  - Tenant Paid: **₹2,400.00** (Full Payment)

#### Room 02 Configuration & Ledgers
- **Room Rent Cycle**: Feb 10, 2026 → Mar 10, 2026 (Rent: ₹10,000)
- **Electricity Cycle**: Feb 12, 2026 → Mar 12, 2026
  - Units Consumed: 450 kWh @ ₹8.00/unit = ₹3,600.00
  - Tenant Paid: **₹3,600.00** (Full Payment)

#### Room 03 Configuration & Ledgers
- **Room Rent Cycle**: Feb 20, 2026 → Mar 20, 2026 (Rent: ₹12,000)
- **Electricity Cycle**: Feb 20, 2026 → Mar 20, 2026
  - Units Consumed: 550 kWh @ ₹8.00/unit = ₹4,400.00
  - Tenant Paid: **₹4,400.00** (Full Payment)

---

### Step-by-Step Reconciliation Calculation

1. **Step 1: Aggregate Overlapping Tenant Collections**:
   $$\text{TenantCollected} = \text{Room 01 (₹2,400)} + \text{Room 02 (₹3,600)} + \text{Room 03 (₹4,400)} = \mathbf{₹10,400.00}$$

2. **Step 2: Compare with Master Bill**:
   $$\text{Variance} = \text{TenantCollected (₹10,400.00)} - \text{Master Bill Paid (₹10,000.00)} = \mathbf{+₹400.00}$$

3. **Step 3: Financial Classification**:
   - $\text{Variance} > 0 \rightarrow$ **Outcome: SURPLUS**
   - `surplusAmount` = **₹400.00**
   - `deficitAmount` = **₹0.00**
   - **Interpretation**: Landlord saved ₹400.00 due to submeter tariff rate differential (₹8/unit submeter vs ₹7.20/unit UPCL commercial rate). This ₹400.00 is stored as `surplusAmount` on the `SupplierMasterBill` record and tracked under Electricity Reconciliation Savings (excluded from rental net profit).

---

## 4. Edge Cases & System Handling Rules

### 4.1 Partial Overlaps Between Room Cycles & Master Bill Cycle
- **Scenario**: A room electricity cycle (e.g. Feb 4 – Mar 4) only partially overlaps with the supplier bill cycle (Feb 24 – Mar 24).
- **Rule**:
  1. **Full Cash Collection Mode (Default)**: Sums all actual tenant payment transactions (`recordedAt` / `paymentDate`) falling within or linked to cycle ledgers active during the master bill window.
  2. **Pro-Rata Weighted Mode (Advanced Analytics)**: If enabled, calculates pro-rata daily consumption:
     $$\text{Pro-Rata Contribution} = \text{TenantCollected} \times \left( \frac{\text{Overlap Days}}{\text{Total Cycle Days}} \right)$$

---

### 4.2 Unpaid or Overdue Tenant Ledgers
- **Scenario**: Tenant in Room 02 has an electricity ledger of ₹3,600 for Feb 12 – Mar 12, but has **NOT paid yet** (`amountPaid = ₹0.00`, status `OVERDUE`).
- **Rule**:
  1. **Cash Accounting Principle**: Only **actual collected cash (`amountPaid`)** is summed into `TenantCollected` during reconciliation. Unpaid dues (`amount - amountPaid`) are **NOT** assumed as collected.
  2. **Temporary Deficit**: If Room 02 has paid ₹0.00, `TenantCollected` = ₹2,400 (Room 01) + ₹4,400 (Room 03) = ₹6,800. Variance = ₹6,800 - ₹10,000 = **-₹3,200 (DEFICIT)**.
  3. **Dynamic Re-Evaluation**: When the Room 02 tenant later pays ₹3,600, the system updates `TenantCollected` to ₹10,400 and dynamically transitions the master bill reconciliation status from **DEFICIT (-₹3,200)** to **SURPLUS (+₹400)**.

---

### 4.3 Mid-Cycle Tenant Checkouts
- **Scenario**: A tenant checks out on Feb 18 mid-billing cycle.
- **Rule**:
  1. Upon checkout, the system generates a final pro-rated `ElectricityLedger` up to Feb 18 based on submeter reading at move-out.
  2. Any payment collected against this final ledger is included in `TenantCollected` for the corresponding master bill cycle.
  3. Historical `TenancyHistory` snapshot preserves the immutable tenant record.

---

### 4.4 Rate Differential & Common Area Electricity Load
- **Rate Differential**: Submeter rate (e.g. ₹8.00/unit) vs. Utility rate (e.g. ₹7.00/unit). The margin covers common area lighting, water pump motors, and line losses.
- **Common Area Motors/Pumps**: Common electricity (submersible water motor) is not billed to individual tenants. If common usage causes `TenantCollected < Master Bill Paid`, the resulting `deficitAmount` is logged as a Landlord Out-of-Pocket Expense under `COMMON_ELECTRICITY`.

---

## 5. Tenant Electricity Payment Cutoff & Master Cycle Allocation Rules

### 5.1 Core Allocation Principle
Tenant electricity payment allocation to building master billing cycles is strictly driven by the **actual payment date (`PaymentTransaction.paymentDate`)**, NOT by the room billing cycle start/end dates.

$$\text{Target Master Cycle} = \begin{cases} \text{Current Master Cycle } [T_{\text{start}}, T_{\text{end}}] & \text{if } \text{paymentDate} \le T_{\text{end}} \\ \text{Next Master Cycle } [T_{\text{end}}, T_{\text{next\_end}}] & \text{if } \text{paymentDate} > T_{\text{end}} \end{cases}$$

---

### 5.2 Core Scenario & Payment Cutoff Cases

#### System Scenario Baseline
- **Building Master Electricity Cycle**: **24 Feb 2026 → 24 Mar 2026**
- **Room 02 Electricity Billing Cycle**: **20 Feb 2026 → 20 Mar 2026**

#### Case 1: Payment Received Within Master Cycle Window
- **Tenant Payment Date**: **21 Mar 2026**
- **Condition Check**: `paymentDate (21 Mar)` $\le$ `masterCycleEndDate (24 Mar)` $\rightarrow$ **TRUE**
- **Allocation Result**:
  - ✅ Included in **Current Master Billing Cycle (24 Feb → 24 Mar)**.
  - Summed into `TenantCollected` for the 24 Feb → 24 Mar supplier bill reconciliation.

#### Case 2: Late Payment Received After Master Cycle Cutoff
- **Tenant Payment Date**: **25 Mar 2026**
- **Condition Check**: `paymentDate (25 Mar)` $>$ `masterCycleEndDate (24 Mar)` $\rightarrow$ **TRUE**
- **Allocation Result**:
  - ❌ **EXCLUDED** from the previous master cycle (24 Feb → 24 Mar).
  - ✅ Included in **Next Master Billing Cycle (24 Mar → 24 Apr)**.
  - Prevents retroactive reopening of settled supplier bills.

---

### 5.3 Detailed Edge Case Allocation Matrix

| Edge Case | Scenario Description | System Handling & Allocation Rule |
|---|---|---|
| **1. Early Payments** | Tenant pays on **15 Feb 2026** before room cycle ends (20 Mar). | Allocated to the building master cycle window containing `15 Feb 2026` (e.g. 24 Jan → 24 Feb). |
| **2. Very Late Payments Across Multiple Cycles** | Tenant pays on **28 Apr 2026** for a February room ledger (20 Feb → 20 Mar). | Allocated strictly to the building master cycle window containing `28 Apr 2026` (i.e. **24 Apr → 24 May** master cycle). Never re-opens past closed cycles. |
| **3. Partial Payments Across Multiple Dates** | Tenant pays ₹1,000 on **20 Mar 2026** and ₹2,600 on **26 Mar 2026** against one ledger. | Each `PaymentTransaction` is evaluated **independently**: <br>• Transaction 1 (₹1,000 on 20 Mar) $\rightarrow$ Allocated to **24 Feb → 24 Mar** Master Cycle. <br>• Transaction 2 (₹2,600 on 26 Mar) $\rightarrow$ Allocated to **24 Mar → 24 Apr** Master Cycle. |
| **4. Unpaid Bills** | Room ledger remains `UNPAID` / `OVERDUE` during cycle. | ❌ Contributes **₹0.00** to tenant collections for that cycle. Only actual collected cash (`amountPaid`) is summed. |

---

## 6. System Data Linkages & ERD Mapping

```
[Building] ──1:N──> [SupplierMasterBill] (billCycleStart, billCycleEnd, masterBillAmount, surplusAmount, deficitAmount)
    │
    └──1:N──> [Room] ──1:N──> [BillingCycle] (cycleStartDate, cycleEndDate)
                                   │
                                   ├──1:1──> [RoomRentLedger] (amount, amountPaid, status)
                                   └──1:1──> [ElectricityLedger] (amount, amountPaid, status)
                                                   │
                                                   └──1:N──> [PaymentTransaction] (amountPaid, paymentDate)
```

---

## 7. Future Enhancements & Tracking Roadmap

1. **Automated Utility API Integration**: Direct API sync with state power utilities (UPCL, UPPCL, TPDDL) to auto-fetch master bill amounts and due dates.
2. **Automated Tenant Reminder Engine**: Scheduled SMS/WhatsApp push notifications triggered when electricity submeter readings are recorded.
3. **AI-Powered Deficit Forecasting**: Machine learning models analyzing historical submeter trends to alert landlords of projected monthly electricity deficits 10 days before master bill due dates.

