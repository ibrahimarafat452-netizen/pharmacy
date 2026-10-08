# Quickstart Validation Guide: Pharmacy Management System V1

**Date**: 2026-10-08
**Plan**: [plan.md](plan.md) | **Data Model**: [data-model.md](data-model.md)

**Purpose**: Runnable validation scenarios that prove each phase works end-to-end. Execute after each phase is implemented to confirm it meets its acceptance criteria.

---

## Prerequisites

- Windows machine (Windows 7 SP1+ for final validation; Windows 10/11 acceptable for development)
- .NET Framework 4.8 installed
- Built application (`PharmacyApp.exe`)
- For printing tests: thermal receipt printer (58mm or 80mm) connected
- For LAN tests: 2+ PCs on the same local network

---

## Phase 1: Foundation — Validate Infrastructure

### Scenario 1.1: First Startup

1. Run `PharmacyApp.exe` for the first time with no existing database
2. **Expected**: Application creates `pharmacy.db` in the application directory
3. **Expected**: Database contains all tables (verify with SQLite browser or `PRAGMA table_list`)
4. **Expected**: `SchemaVersion` table shows migration version = latest
5. **Expected**: Application window appears with correct Arabic RTL layout

### Scenario 1.2: Crash Recovery

1. Start the application
2. Kill the process mid-operation (Task Manager → End Process)
3. Restart the application
4. **Expected**: Application starts normally, database is consistent (run `PRAGMA integrity_check`)

---

## Phase 2: Authentication — Validate Auth Module

*Detailed validation in `specs/002-auth-permissions/quickstart.md`*

### Scenario 2.1: First-Run Login

1. Start application (fresh database)
2. **Expected**: Login screen appears
3. Log in with `admin` / `admin`
4. **Expected**: Forced password change screen appears
5. Change password to a new value
6. **Expected**: Main interface appears with full Admin access

### Scenario 2.2: Role Enforcement

1. As Admin, create a Cashier user
2. Log out, log in as Cashier
3. Attempt to navigate to Inventory Management
4. **Expected**: Access denied message; navigation blocked

---

## Phase 3: Inventory — Validate Product & Stock

### Scenario 3.1: Product Creation

1. Log in as Pharmacist
2. Create a product: name = "بانادول" (Panadol), barcode = "6281022112115", category = "مسكنات", unit = box, strips/box = 10, price = 25.00 EGP
3. **Expected**: Product appears in product list with correct Arabic name and details

### Scenario 3.2: FEFO Deduction (Manual Verification)

1. Create two batches for "بانادول":
   - Batch A: qty = 20 strips, expiry = 2027-03-01
   - Batch B: qty = 30 strips, expiry = 2026-12-01 (expires sooner)
2. Perform a stock deduction of 15 strips (via a test or the adjustment mechanism)
3. **Expected**: 15 strips deducted from Batch B (earlier expiry). Batch B qty = 15, Batch A qty = 20

### Scenario 3.3: Expired Batch Exclusion

1. Create a batch with expiry = yesterday
2. Run expiry flagging
3. **Expected**: Batch is flagged as expired, excluded from available stock

### Scenario 3.4: Stock Adjustment

1. Select "بانادول", Batch B
2. Adjust quantity from 15 to 10, reason = "Damaged"
3. **Expected**: Quantity updated, audit trail entry shows old=15, new=10, reason="Damaged", user, timestamp

---

## Phase 4: Purchases — Validate Purchase Flow

### Scenario 4.1: Record a Purchase

1. Create supplier "شركة فاركو" (Pharco)
2. Create a purchase: supplier = Pharco, date = today, line items:
   - بانادول: qty = 50 strips, cost = 1.50 EGP/strip, batch = "B2024-001", expiry = 2027-06-01
3. **Expected**: Purchase recorded, new batch created with qty=50, supplier balance = purchase total

### Scenario 4.2: Supplier Payment

1. Pay 500 EGP to Pharco
2. **Expected**: OutstandingBalance reduced by 500, payment history shows date/amount/balance

### Scenario 4.3: Purchase Return

1. Return 10 strips of بانادول from the purchase
2. **Expected**: Batch qty reduced by 10, supplier balance reduced by return value, audit entry created

---

## Phase 5: POS — Validate Sale Workflow

### Scenario 5.1: Complete Sale (Keyboard-Only)

1. Log in as Cashier
2. Open POS screen
3. Scan barcode "6281022112115" (or type and press Enter)
4. **Expected**: بانادول appears with qty=1, price=25.00 EGP
5. Adjust quantity to 3
6. **Expected**: Line total = 75.00, sale total = 75.00
7. Press payment shortcut, enter 100.00 EGP
8. **Expected**: Change = 25.00. Sale completed. Inventory decremented by 3 strips (FEFO).
9. **Expected**: Receipt prints (or print dialog appears) with pharmacy name, items, totals, cashier name

### Scenario 5.2: Zero Stock Prevention

1. Deplete all stock of a product (sell or adjust to 0)
2. Attempt to add that product to a new sale
3. **Expected**: System prevents addition, shows "stock unavailable" message

### Scenario 5.3: Void Sale

1. Log in as Pharmacist
2. Void the sale from Scenario 5.1 with reason "Customer returned"
3. **Expected**: Sale status = voided. Inventory restocked (3 strips returned to original batches). Audit entry created.

### Scenario 5.4: Credit Sale

1. Create customer "أحمد محمد"
2. Create a sale of 50.00 EGP for أحمد, pay 30.00 EGP
3. **Expected**: Sale completed, debt of 20.00 EGP recorded against customer

---

## Phase 6: Customer & Debt — Validate Debt Tracking

### Scenario 6.1: Debt Payment

1. أحمد has 20.00 EGP debt (from Phase 5)
2. Record a payment of 10.00 EGP from أحمد
3. **Expected**: Debt = 10.00 EGP. Payment history shows: amount=10, balanceBefore=20, balanceAfter=10

### Scenario 6.2: Sale Return

1. أحمد returns 1 item from the credit sale
2. **Expected**: Inventory restocked. Debt adjusted (reduced by the return amount). Audit entry created.

---

## Phase 7: Expenses & Stocktaking

### Scenario 7.1: Record Expense

1. Log in as Pharmacist
2. Record expense: date = today, amount = 2000 EGP, category = "إيجار" (Rent), description = "إيجار أكتوبر"
3. **Expected**: Expense saved. Audit entry created.

### Scenario 7.2: Stocktaking

1. Initiate stocktake for all products
2. **Expected**: List of products with recorded quantities and entry fields
3. Enter counted quantities (some matching, some different)
4. Review discrepancies
5. **Expected**: Only mismatched items shown with differences highlighted
6. Approve adjustments
7. **Expected**: Stock levels updated to counted values. Audit entries for each adjustment with reason "Stocktaking"

---

## Phase 8: Reports — Validate Each Report

### Scenario 8.1: Sales Report

1. Generate sales report for today's date range
2. **Expected**: Shows total sales, transaction count, product breakdown, payment summary
3. Print the report
4. **Expected**: Report prints correctly (Arabic, RTL)
5. Export to PDF
6. **Expected**: PDF file created with correct content

### Scenario 8.2: Profit Overview

1. Generate profit report for this month
2. **Expected**: Revenue (from sales), COGS (from purchase prices), Expenses (from recorded expenses), Net Profit calculated

*Repeat for Inventory, Expiry, Debt, and Purchase reports*

---

## Phase 9: AI Recommendations — Validate Engine

### Scenario 9.1: Reorder Recommendation

1. Create a product with steady sales (e.g., 10 strips/day for 30+ days) and low stock (20 strips)
2. Run recommendation calculation
3. **Expected**: Reorder recommendation with Score, Priority, Reason (explains calculation), and recommended order qty

### Scenario 9.2: Collecting Data Status

1. Create a new product with < 30 days of history
2. Request recommendations
3. **Expected**: Shows "collecting data" with confidence indicator, not a score-based recommendation

### Scenario 9.3: Expiry Risk

1. Create a batch expiring in 15 days with 100 strips
2. Run recommendations
3. **Expected**: Expiry risk alert with score, value at risk, and suggested action

---

## Phase 10: Configuration & Backup

### Scenario 10.1: Pharmacy Identity

1. Set pharmacy name = "صيدلية النور", address, phone
2. Complete a sale and print a receipt
3. **Expected**: Receipt shows the configured pharmacy name, address, phone

### Scenario 10.2: Backup & Restore

1. Configure backup: interval = 5 min, location = a test folder, retention = 3
2. Wait for automatic backup
3. **Expected**: Backup file created in configured location. Integrity check = passed.
4. Create some test data after the backup
5. Restore from the backup
6. **Expected**: Safety backup created first. Data reverts to backup state. Post-backup test data is gone.

---

## Phase 11: Licensing

### Scenario 11.1: Valid License

1. Generate a test license file for the current hardware
2. Place it in the application directory
3. Start the application
4. **Expected**: License verified, application starts normally

### Scenario 11.2: Missing License

1. Remove or rename the license file
2. Start the application
3. **Expected**: "Activation required" screen. Hardware fingerprint displayed for reactivation contact.

---

## Phase 12: LAN — Validate Multi-PC

*Prototype P-10.1 must pass before this phase*

### Scenario 12.1: Server + Client Connection

1. Configure PC-A as server
2. Configure PC-B as client with pharmacy secret
3. Start both
4. **Expected**: PC-B connects to PC-A. User login screen appears on PC-B.

### Scenario 12.2: Data Consistency

1. Create a sale on PC-A
2. Check inventory on PC-B
3. **Expected**: Inventory change visible on PC-B within 5 seconds (SC-008)

### Scenario 12.3: Server Unavailable

1. Shut down PC-A (server)
2. Attempt a sale on PC-B (client)
3. **Expected**: "Server unreachable" message. Transaction blocked. No offline queuing.

### Scenario 12.4: Concurrent Write Conflict

1. On PC-A and PC-B simultaneously, edit the same product's price
2. Second save arrives after the first
3. **Expected**: Optimistic concurrency detects conflict. Second user gets "record modified by another user" message.
