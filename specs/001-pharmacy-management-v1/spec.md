# Feature Specification: Pharmacy Management System V1

**Feature Branch**: `001-pharmacy-management-v1`

**Created**: 2026-10-08

**Status**: Draft

**Input**: User description: "Pharmacy Management System V1 — full system specification covering all agreed features and business rules from Constitution v1.0.0 and Architecture Discovery"

**Constitution Reference**: v1.0.0 (Ratified 2026-10-07)

**Architecture Reference**: Architecture Discovery (Revised 2026-10-08)

---

## Clarifications

### Session 2026-10-08

- Q: Which roles are authorized to void a completed sale and process sale returns? → A: Void sales require Pharmacist or Admin. Sale returns can be performed by any role (Cashier, Pharmacist, Admin).
- Q: Does the system need to calculate and display tax or VAT on any products? → A: All prices are final (no separate tax calculation). Tax is out of scope for V1.
- Q: Are product categories predefined by the system or created freely by the pharmacy staff? → A: Hybrid — system ships with default categories, Admin can add, edit, and remove categories.
- Q: Should V1 reports be printable, or is on-screen display sufficient? → A: Screen + Print + Export — reports can be viewed on screen, printed, and exported to a file.
- Q: How will a pharmacy initially populate its product catalog? → A: Primary workflow is barcode-based entry through purchases. Basic spreadsheet import supported for initial catalog setup/migration only. No supplier catalog integration in V1.

---

## User Scenarios & Testing

### User Story 1 — Fast Point of Sale (Priority: P1)

A Cashier or Pharmacist opens the sales screen, scans product barcodes (or searches by name), builds a sale of one or more line items, accepts payment (cash or partial), prints a receipt, and moves to the next customer — all in seconds, primarily by keyboard.

**Why this priority**: This is the core daily operation of every pharmacy. Speed and accuracy here determine whether the system is adopted or rejected.

**Independent Test**: Can be tested by creating products, scanning/searching them into a sale, completing payment, and verifying the receipt prints and inventory decrements correctly.

**Acceptance Scenarios**:

1. **Given** products exist in inventory with available stock, **When** a Cashier scans a barcode, **Then** the product and its price appear on the sale screen with quantity 1.
2. **Given** a sale has one or more line items, **When** the Cashier presses the payment shortcut and enters the cash amount, **Then** the system records the sale, decrements inventory (FEFO), records the payment, and displays change due.
3. **Given** a completed sale, **When** the system processes payment, **Then** a thermal receipt is generated with pharmacy name/logo, sale items, quantities, unit prices, totals, date/time, and cashier name.
4. **Given** a product with multiple batches, **When** it is sold, **Then** the system automatically deducts from the batch with the earliest expiry date (FEFO).
5. **Given** a product is searched by partial Arabic name, **When** the Cashier types in the search field, **Then** matching products appear for selection.
6. **Given** a sale line item, **When** the Cashier adjusts the quantity, **Then** the line total and sale total update immediately.
7. **Given** a product with zero available stock, **When** a Cashier attempts to add it to a sale, **Then** the system prevents the addition and displays a stock unavailable message.
8. **Given** a sale in progress, **When** power failure occurs mid-transaction, **Then** the incomplete sale is not recorded and inventory remains unchanged (transaction rollback).

---

### User Story 2 — Inventory & Batch Management (Priority: P2)

A Pharmacist manages the pharmacy's inventory by receiving stock in batches (with expiry dates, box/strip units), viewing current stock levels, performing stock adjustments, and tracking all inventory changes through an audit trail.

**Why this priority**: Accurate inventory is the foundation for sales, purchasing decisions, and regulatory compliance. Without it, nothing else works correctly.

**Independent Test**: Can be tested by adding products with batches, verifying stock levels, performing adjustments, and confirming the audit trail records every change.

**Acceptance Scenarios**:

1. **Given** a new product, **When** a Pharmacist creates it, **Then** the system records the product name (Arabic), barcode, category, unit type (box/strip), strips-per-box ratio, and selling price.
2. **Given** an existing product, **When** stock is received, **Then** a new batch is created with batch number, quantity, purchase price, expiry date, and supplier reference.
3. **Given** multiple batches of the same product, **When** viewing inventory, **Then** the system shows total available quantity and individual batch details (quantity, expiry, batch number).
4. **Given** a batch with an expiry date, **When** the expiry date has passed, **Then** the system flags the batch as expired and excludes it from sale.
5. **Given** a stock discrepancy, **When** a Pharmacist performs a stock adjustment, **Then** the adjustment is recorded with reason, old quantity, new quantity, user, and timestamp in the audit log.
6. **Given** a product sold in strips, **When** a sale deducts 3 strips from a box of 10, **Then** the remaining quantity shows 7 strips (or the equivalent fractional box).
7. **Given** any inventory change (sale, purchase, adjustment, return), **When** the change is committed, **Then** an audit trail entry is created with the change type, quantity, user, and timestamp.

---

### User Story 3 — Purchase Management (Priority: P3)

A Pharmacist records purchases from suppliers, receives goods into inventory with batch and expiry details, and tracks supplier balances and payment status.

**Why this priority**: Purchasing is the primary way stock enters the system. Accurate purchase records are essential for supplier balances and cost tracking.

**Independent Test**: Can be tested by creating a supplier, recording a purchase, receiving goods into inventory, and verifying supplier balance updates.

**Acceptance Scenarios**:

1. **Given** a supplier exists, **When** a Pharmacist creates a purchase, **Then** the system records the supplier, purchase date, line items (product, quantity, unit cost, batch number, expiry date), and total amount.
2. **Given** a purchase is recorded, **When** goods are received, **Then** inventory batches are created or updated with the received quantities and expiry dates.
3. **Given** a purchase with a total of 5000 EGP, **When** the Pharmacist pays 3000 EGP, **Then** the supplier balance shows 2000 EGP outstanding.
4. **Given** a purchase with defective items, **When** a Pharmacist creates a purchase return, **Then** the returned quantity is removed from inventory, the supplier balance is reduced, and the return is recorded in the audit log.
5. **Given** multiple purchases from a supplier, **When** viewing the supplier account, **Then** the system shows all purchases, payments, returns, and the current outstanding balance.

---

### User Story 4 — Customer & Debt Management (Priority: P4)

A Pharmacist or Cashier creates sales on credit for known customers, tracks outstanding debts, accepts partial payments over time, and maintains a complete payment history per customer.

**Why this priority**: Many Egyptian pharmacies extend credit to regular customers. Accurate debt tracking prevents financial disputes and losses.

**Independent Test**: Can be tested by creating a customer, making a credit sale, recording partial payments, and verifying the debt balance updates correctly with full transaction history.

**Acceptance Scenarios**:

1. **Given** a registered customer, **When** a sale is completed on credit (no immediate payment or partial payment), **Then** the unpaid amount is recorded as a debt against the customer.
2. **Given** a customer with an outstanding debt of 500 EGP, **When** the customer pays 200 EGP, **Then** the remaining debt shows 300 EGP and the payment is recorded with date, amount, and receiving user.
3. **Given** a customer with multiple credit sales, **When** viewing the customer account, **Then** all credit sales, all payments, and the current total outstanding debt are displayed.
4. **Given** a customer makes a payment, **When** the payment is recorded, **Then** the payment transaction includes timestamp, amount, user who received it, and the debt balance before and after.
5. **Given** a completed sale, **When** the customer returns items, **Then** a sale return is created, inventory is restocked (into the original batch or a return batch), and the customer's debt or payment is adjusted accordingly.

---

### User Story 5 — User Authentication & Role-Based Access (Priority: P5)

An Admin manages user accounts with assigned roles (Admin, Pharmacist, Cashier). Users log in at application startup. The system enforces permissions based on role and records sensitive actions in an audit log.

**Why this priority**: Security and accountability are non-negotiable for a system handling financial transactions and controlled substances.

**Independent Test**: Can be tested by creating users with different roles, logging in as each, verifying permitted/denied actions, and checking audit log entries.

**Acceptance Scenarios**:

1. **Given** the application starts, **When** a user enters valid credentials, **Then** the system authenticates them and shows the interface appropriate to their role.
2. **Given** invalid credentials, **When** a user attempts to log in, **Then** the system denies access and does not reveal which field (username or password) was incorrect.
3. **Given** a Cashier role, **When** the Cashier attempts to access inventory management or user administration, **Then** the system denies access.
4. **Given** a Pharmacist role, **When** the Pharmacist performs a sale, views inventory, or records a purchase, **Then** the system allows these actions.
5. **Given** an Admin role, **When** the Admin creates a new user, assigns a role, or resets a password, **Then** the action succeeds and is recorded in the audit log.
6. **Given** a sensitive action (void sale, stock adjustment, price change, user management), **When** any user performs it, **Then** the audit log records the action, user, timestamp, and relevant details.
7. **Given** a user account, **When** the Admin changes the user's role, **Then** the new permissions take effect at the user's next login.

---

### User Story 6 — Backup & Recovery (Priority: P6)

An Admin configures automatic backups to a chosen location. The system creates backups at regular intervals, maintains multiple backup versions with rotation, verifies backup integrity, and supports restore with safety confirmation. The system automatically recovers from unexpected shutdowns.

**Why this priority**: Data loss is catastrophic for a pharmacy. Backup and crash recovery protect the business from hardware failures and power outages.

**Independent Test**: Can be tested by configuring backup settings, triggering a backup, verifying the backup file, simulating a restore, and verifying crash recovery leaves no partial transactions.

**Acceptance Scenarios**:

1. **Given** an Admin configures backup settings, **When** the backup interval elapses, **Then** the system creates a backup at the configured location without interrupting normal operation.
2. **Given** a backup completes, **When** the system verifies it, **Then** the integrity check confirms the backup is valid and complete.
3. **Given** multiple backups exist beyond the retention count, **When** a new backup is created, **Then** the oldest backup beyond the retention limit is removed (rotation), but business data in the active database is never deleted.
4. **Given** an Admin selects a backup to restore, **When** they confirm the restore, **Then** the system creates a safety backup of the current state before restoring and then restores the selected backup.
5. **Given** the application crashes or power fails unexpectedly, **When** the application restarts, **Then** the database automatically recovers to the last consistent state with no partial transactions.
6. **Given** an integrity check fails on startup, **When** the system detects corruption, **Then** it offers to restore from the most recent valid backup.
7. **Given** backup location options, **When** an Admin configures the location, **Then** the system supports local drives, USB drives, and external drives as backup destinations.

---

### User Story 7 — AI Recommendation Engine (Priority: P7)

A Pharmacist views intelligent recommendations about inventory — when to reorder, what is at risk of expiry, what is slow-moving or overstocked, and what might run out. Every recommendation includes a score, priority, reason, and suggested action. The engine operates entirely locally using the pharmacy's own data.

**Why this priority**: Recommendations prevent financial losses from expired stock, stockouts, and overstocking. They transform data into actionable decisions.

**Independent Test**: Can be tested by populating sales and inventory history, triggering recommendation calculations, and verifying each recommendation type produces explainable results with score, priority, reason, and action.

**Acceptance Scenarios**:

1. **Given** a product with declining stock and steady sales history, **When** the Pharmacist views recommendations, **Then** a Reorder Recommendation appears with score (0-100), priority (Critical/High/Medium/Low), reason explaining the calculation, and recommended order quantity.
2. **Given** a product with rising sales over the past months, **When** the Sales Trend analysis runs, **Then** it identifies the upward trend, estimates future demand, and recommends increased reorder quantities.
3. **Given** a batch expiring within the configured threshold, **When** the Expiry Risk analysis runs, **Then** it shows the product, batch, expiry date, quantity at risk, financial value at risk, and suggests actions (e.g., discount, prioritize sales).
4. **Given** a product with no sales in 90+ days, **When** the Slow-Moving/Dead Stock analysis runs, **Then** it identifies the product, calculates capital tied up, and suggests actions.
5. **Given** a product with stock exceeding projected demand, **When** the Overstock Risk analysis runs, **Then** it flags the product with estimated months of surplus and capital at risk.
6. **Given** a product with stock projected to deplete before the next reorder can arrive, **When** the Stockout Risk analysis runs, **Then** it shows the predicted depletion date, urgency level, and recommended action.
7. **Given** a newly added product with fewer than 30 days of sales history, **When** recommendations are requested, **Then** the system shows "collecting data" with a confidence indicator instead of unreliable recommendations.
8. **Given** Admin access to recommendation settings, **When** the Admin adjusts thresholds (safety stock days, lead times, overstock limits), **Then** future recommendations use the updated thresholds.
9. **Given** any recommendation, **When** it is displayed, **Then** it always includes Score (0-100), Priority, Reason (text explanation), and Recommended Action — never an unexplained fact.

---

### User Story 8 — Expense Tracking (Priority: P8)

A Pharmacist or Admin records pharmacy operational expenses (rent, utilities, supplies, etc.) with date, amount, category, and description for accurate financial tracking.

**Why this priority**: The constitution requires expenses to be accurately recorded alongside sales, purchases, and debts for complete financial visibility.

**Independent Test**: Can be tested by recording expenses in different categories, viewing expense history, and verifying totals.

**Acceptance Scenarios**:

1. **Given** the expense entry screen, **When** a Pharmacist records an expense with date, amount (EGP), category, and description, **Then** the expense is saved and reflected in financial records.
2. **Given** multiple expenses recorded, **When** viewing expenses for a date range, **Then** all expenses in that range are displayed with totals per category and overall total.
3. **Given** an expense record, **When** it is created, **Then** the audit log records the entry with user, timestamp, and amount.

---

### User Story 9 — Reports (Priority: P9)

A Pharmacist or Admin generates reports to understand the pharmacy's financial and operational status — daily sales, inventory status, expiry alerts, debt summaries, purchase history, and profit overview.

**Why this priority**: Reports turn raw data into business decisions. They are essential for daily operations and periodic financial review.

**Independent Test**: Can be tested by populating data across sales, inventory, purchases, and debts, then generating each report and verifying accuracy.

**Acceptance Scenarios**:

1. **Given** sales data exists, **When** a Pharmacist generates a sales report for a date range, **Then** the report shows total sales, number of transactions, breakdown by product, and payment method summary.
2. **Given** inventory data exists, **When** a Pharmacist generates an inventory status report, **Then** it shows all products with current stock levels, batch details, and expiry dates.
3. **Given** batches with upcoming expiry dates, **When** an expiry report is generated, **Then** it lists products expiring within a configurable period, sorted by urgency, with quantities and financial values.
4. **Given** customers with outstanding debts, **When** a debt report is generated, **Then** it shows all customers with outstanding balances, sorted by amount, with aging information.
5. **Given** purchases recorded over a period, **When** a purchase report is generated, **Then** it shows total purchases by supplier, amounts paid vs. outstanding, and product breakdown.
6. **Given** sales revenue, purchase costs, and expenses, **When** a profit overview report is generated, **Then** it shows revenue, cost of goods sold, expenses, and net profit for the selected period.

---

### User Story 10 — Stocktaking (Priority: P10)

A Pharmacist conducts a physical inventory count, enters the actual quantities found, and the system calculates discrepancies against recorded stock, allowing the Pharmacist to approve adjustments.

**Why this priority**: Regular stocktaking is essential for inventory accuracy and is required by the constitution's inventory accuracy principle.

**Independent Test**: Can be tested by initiating a stocktake, entering counted quantities, reviewing discrepancies, approving adjustments, and verifying the audit trail.

**Acceptance Scenarios**:

1. **Given** a Pharmacist initiates stocktaking, **When** they select products (all or by category), **Then** the system presents a list showing product name, recorded quantity, and a field for entering counted quantity.
2. **Given** counted quantities are entered, **When** the Pharmacist reviews discrepancies, **Then** the system shows each product where counted quantity differs from recorded quantity, with the difference highlighted.
3. **Given** discrepancies are reviewed, **When** the Pharmacist approves the adjustment, **Then** stock levels are updated to match the physical count and every adjustment is recorded in the audit log with reason "Stocktaking," old quantity, new quantity, user, and timestamp.

---

### User Story 11 — LAN Multi-PC Operation (Priority: P11)

An Admin configures one PC as the server (holding the authoritative database) and connects additional PCs in the pharmacy over the local network. All PCs share the same data. The system operates without Internet.

**Why this priority**: Multi-PC support is required by Constitution Principle V. Many pharmacies have separate POS and back-office stations.

**Independent Test**: Can be tested by running the application on multiple PCs on the same LAN, performing concurrent operations, and verifying data consistency. (Requires prototype validation per Architecture Discovery Section 10.1.)

**Acceptance Scenarios**:

1. **Given** a PC is configured as server, **When** a client PC connects over LAN, **Then** the client accesses the same data as the server without Internet.
2. **Given** two PCs are connected, **When** a sale is completed on one PC, **Then** the inventory change is immediately visible on the other PC.
3. **Given** a client PC loses network connection, **When** it attempts a transaction, **Then** the system informs the user that the server is unreachable and prevents the transaction (no offline queuing — per Constitution Principles VI and VIII).
4. **Given** the server PC restarts, **When** it comes back online, **Then** client PCs automatically reconnect and resume operation.
5. **Given** concurrent writes from two PCs targeting the same record, **When** the second write arrives, **Then** optimistic concurrency control detects the conflict and prevents a silent overwrite.
6. **Given** LAN mode is active, **When** a user logs in on a client PC, **Then** authentication occurs after the LAN connection is established, using the shared database.

---

### User Story 12 — Licensing & Activation (Priority: P12)

The vendor provides a signed license file bound to the pharmacy's hardware. The application verifies the license at startup. Hardware changes require the pharmacist to contact the vendor for reactivation.

**Why this priority**: Licensing protects the vendor's business model and is required by Constitution Principle XII.

**Independent Test**: Can be tested by installing the application, activating with a valid license, verifying startup validation, and testing rejection of invalid/tampered licenses.

**Acceptance Scenarios**:

1. **Given** a valid signed license file matching the hardware, **When** the application starts, **Then** it verifies the license and starts normally.
2. **Given** no license file or an invalid/tampered license, **When** the application starts, **Then** it displays a clear activation required message and does not allow access to the system.
3. **Given** the hardware changes (e.g., new hard drive), **When** the application starts, **Then** it detects the hardware mismatch and prompts the user to contact the vendor for reactivation.
4. **Given** a license for Pharmacy A, **When** it is copied to Pharmacy B's machine, **Then** the hardware fingerprint mismatch prevents activation on Pharmacy B.

---

### User Story 13 — System Configuration (Priority: P13)

An Admin configures the pharmacy's identity (name, logo, address, phone), system preferences, receipt layout, backup settings, recommendation thresholds, and user accounts — all without modifying the core application code.

**Why this priority**: Per Constitution Principle XIII, customer-specific configuration must not require code changes. This enables one Core Product serving many pharmacies.

**Independent Test**: Can be tested by configuring pharmacy details, verifying they appear on receipts and the UI, and confirming settings persist across restarts.

**Acceptance Scenarios**:

1. **Given** the Admin opens system settings, **When** they enter pharmacy name, logo, address, and phone number, **Then** these appear on receipts and the application title.
2. **Given** system preferences, **When** the Admin configures currency display format, receipt content, and printer selection, **Then** the preferences apply immediately and persist.
3. **Given** AI recommendation thresholds, **When** the Admin adjusts safety stock days, supplier lead times, or overstock limits, **Then** future recommendations use the new thresholds.

---

### Edge Cases

- What happens when a sale is attempted for a product with all batches expired? System MUST prevent the sale and display an expiry warning.
- What happens when the last strip of a box is sold? System MUST correctly track zero remaining and handle unit conversions.
- What happens when a return quantity exceeds the original sale quantity? System MUST reject the return.
- What happens when multiple users attempt to adjust the same product's stock simultaneously (LAN)? Optimistic concurrency MUST detect the conflict and prevent silent overwrite.
- What happens when the backup location is unavailable (USB removed)? System MUST notify the Admin and retain the backup locally until the location is available.
- What happens when a customer's total debt exceeds a reasonable limit? System MUST allow the sale on credit (no arbitrary limit) but display the total outstanding balance prominently.
- What happens when a product has no barcode? System MUST allow searching by Arabic name or product code.
- What happens when the thermal printer is not connected? System MUST complete the sale and offer to retry printing or skip.
- What happens when a sale is voided? System MUST reverse inventory changes (restock), reverse financial entries, and record the void in the audit log with the authorizing user.

---

## Requirements

### Functional Requirements

**Point of Sale**

- **FR-001**: System MUST support creating sales with one or more line items, each referencing a product, quantity, and unit price.
- **FR-002**: System MUST support barcode scanning input to add products to a sale.
- **FR-003**: System MUST support searching products by Arabic name (partial match).
- **FR-004**: System MUST automatically apply FEFO (First Expiry, First Out) when deducting inventory for a sale.
- **FR-005**: System MUST prevent selling products with zero available stock or only expired batches.
- **FR-006**: System MUST calculate sale totals, line totals, and change due in real time.
- **FR-007**: System MUST support full cash payment and partial payment (creating a customer debt for the remainder).
- **FR-008**: System MUST generate and print thermal receipts (58mm/80mm) with pharmacy identity, sale details, date/time, and cashier name.
- **FR-009**: System MUST support voiding a completed sale, restricted to Pharmacist or Admin roles, reversing inventory and financial entries.
- **FR-010**: System MUST support keyboard-first navigation for all POS operations with minimal mouse dependency.

**Inventory Management**

- **FR-011**: System MUST track products with: name (Arabic), barcode, category, unit type (box/strip), strips-per-box ratio, and selling price. Categories are managed from a hybrid list: the system ships with default categories, and the Admin can add, edit, and remove categories.
- **FR-011a**: System MUST support creating products automatically when entered through a purchase (barcode-based, primary workflow) and through a basic spreadsheet import for initial catalog setup and migration.
- **FR-012**: System MUST support batch tracking with: batch number, quantity, purchase price, expiry date, and supplier reference.
- **FR-013**: System MUST enforce FEFO across all inventory operations (sales, returns).
- **FR-014**: System MUST prevent stock from going negative.
- **FR-015**: System MUST support box and strip units with automatic conversion between them.
- **FR-016**: System MUST flag expired batches and exclude them from sale operations.
- **FR-017**: System MUST support manual stock adjustments with mandatory reason, recording old and new quantities.
- **FR-018**: System MUST maintain a complete audit trail for all inventory changes (sale, purchase, adjustment, return, stocktaking).

**Purchasing & Suppliers**

- **FR-019**: System MUST support recording purchases with: supplier, date, line items (product, quantity, unit cost, batch number, expiry date), and total amount.
- **FR-020**: System MUST create or update inventory batches when a purchase is recorded.
- **FR-021**: System MUST track supplier balances (total purchases minus total payments minus returns).
- **FR-022**: System MUST support partial payments to suppliers with full payment history.
- **FR-023**: System MUST support purchase returns with inventory and supplier balance adjustments.
- **FR-024**: System MUST maintain supplier records with: name, phone, address, and notes.

**Customer & Debt Management**

- **FR-025**: System MUST support customer records with: name, phone, address, and notes.
- **FR-026**: System MUST support credit sales that create or increase a customer's outstanding debt.
- **FR-027**: System MUST support partial debt payments with complete payment transaction history (date, amount, receiving user, balance before/after).
- **FR-028**: System MUST support sale returns (available to Cashier, Pharmacist, and Admin) with inventory restock and customer debt/payment adjustment.

**Expenses**

- **FR-029**: System MUST support recording expenses with: date, amount (EGP), category, and description.
- **FR-030**: System MUST include expenses in financial calculations and reports.

**User Authentication & Security**

- **FR-031**: System MUST require user authentication (username + password) at application startup.
- **FR-032**: System MUST store passwords using a strong one-way hash.
- **FR-033**: System MUST enforce role-based permissions: Admin (full access), Pharmacist (sales, purchases, inventory, reports), Cashier (sales and payments only).
- **FR-034**: System MUST enforce permission checks at the business logic layer, not only at the UI layer.
- **FR-035**: System MUST record sensitive actions in an audit log with: action type, user, timestamp, and relevant details.
- **FR-036**: Sensitive actions include: void sale, stock adjustment, price change, user management, purchase return, and configuration changes.

**Backup & Recovery**

- **FR-037**: System MUST support automatic backups at configurable intervals.
- **FR-038**: System MUST support configurable backup location (local, USB, external drives).
- **FR-039**: System MUST maintain multiple backup versions with configurable retention count and rotation.
- **FR-040**: System MUST verify backup integrity after each backup.
- **FR-041**: System MUST create a safety backup before performing a restore.
- **FR-042**: System MUST automatically recover from unexpected shutdowns with no partial transactions.
- **FR-043**: System MUST never automatically delete business data; only backup files are subject to rotation.

**AI Recommendation Engine**

- **FR-044**: System MUST provide Reorder Recommendations based on sales rate, current stock, and supplier lead time.
- **FR-045**: System MUST provide Sales Trend & Demand Forecast identifying rising/falling demand and seasonal patterns.
- **FR-046**: System MUST provide Expiry Risk alerts for near-expiry batches with quantity and financial value at risk.
- **FR-047**: System MUST provide Slow-Moving & Dead Stock identification with capital tied up.
- **FR-048**: System MUST provide Overstock Risk alerts for inventory exceeding projected demand.
- **FR-049**: System MUST provide Stockout Risk alerts with predicted depletion date and urgency.
- **FR-050**: Every recommendation MUST include: Score (0-100), Priority (Critical/High/Medium/Low), Reason (text), and Recommended Action (text).
- **FR-051**: System MUST NOT present recommendations as unexplained facts.
- **FR-052**: System MUST show "collecting data" for products with insufficient sales history instead of unreliable recommendations.
- **FR-053**: Recommendation thresholds (safety stock days, lead times, overstock limits) MUST be Admin-configurable.

**Reports**

- **FR-054**: System MUST provide a Sales Report with: total sales, transaction count, product breakdown, and payment summary for a date range.
- **FR-055**: System MUST provide an Inventory Status Report with: all products, current stock levels, batch details, and expiry dates.
- **FR-056**: System MUST provide an Expiry Report with: products expiring within a configurable period, quantities, and financial values.
- **FR-057**: System MUST provide a Debt Report with: all customers with outstanding balances, sorted by amount, with aging.
- **FR-058**: System MUST provide a Purchase Report with: purchases by supplier, amounts paid vs. outstanding.
- **FR-059**: System MUST provide a Profit Overview with: revenue, cost of goods sold, expenses, and net profit for a period.
- **FR-059a**: All reports MUST support on-screen viewing, printing to any installed printer, and exporting to file.

**Stocktaking**

- **FR-060**: System MUST support initiating a physical inventory count for all or selected products.
- **FR-061**: System MUST display recorded quantities alongside fields for entering counted quantities.
- **FR-062**: System MUST calculate and display discrepancies between recorded and counted quantities.
- **FR-063**: System MUST adjust stock levels upon Pharmacist approval and record all adjustments in the audit log.

**LAN Multi-PC**

- **FR-064**: System MUST support single-PC mode (default) with direct database access and no network configuration.
- **FR-065**: System MUST support multi-PC mode where one PC acts as server and others connect as clients over LAN.
- **FR-066**: Same application executable MUST run on all PCs; server mode is a configuration toggle.
- **FR-067**: Client PCs MUST NOT queue transactions offline; if the server is unreachable, transactions MUST be blocked.
- **FR-068**: Server MUST serialize all database writes; reads MUST be concurrent.
- **FR-069**: System MUST use optimistic concurrency control to prevent silent overwrites on concurrent edits.
- **FR-070**: Client PCs MUST automatically reconnect after server or network interruption.
- **FR-071**: LAN connection MUST require a shared pharmacy secret before user authentication.

**Licensing**

- **FR-072**: System MUST verify a signed license file at application startup.
- **FR-073**: License MUST be bound to the machine's hardware fingerprint.
- **FR-074**: System MUST deny access when the license is missing, invalid, tampered, or hardware-mismatched.
- **FR-075**: System MUST support a controlled reactivation process for legitimate hardware changes.
- **FR-076**: License verification MUST NOT require Internet access.

**System Configuration**

- **FR-077**: System MUST support configuring pharmacy identity (name, logo, address, phone) without code changes.
- **FR-078**: System MUST support configuring receipt layout and printer selection.
- **FR-079**: All customer-specific settings MUST be stored in configuration, not in core application code.

**Cross-Cutting**

- **FR-080**: All financial amounts MUST be in Egyptian Pound (EGP). All selling prices are final amounts (tax-inclusive or tax-exempt); separate tax/VAT calculation is out of scope for V1.
- **FR-081**: The entire UI MUST be in Arabic with correct RTL layout.
- **FR-082**: All financial and inventory operations MUST execute within atomic database transactions.
- **FR-083**: System MUST operate fully without Internet access for all core business operations.

### Key Entities

- **Product**: A medicine or pharmacy item. Has name (Arabic), barcode, category, unit type (box/strip), strips-per-box ratio, selling price, and status.
- **Batch**: A specific lot of a product. Has batch number, quantity, purchase price, expiry date, supplier reference. Linked to one Product.
- **Sale**: A transaction selling products to a customer or walk-in. Has sale number, date/time, line items, total amount, payment details, cashier, and status.
- **Sale Line Item**: One product entry within a sale. Has product reference, quantity, unit price, line total, and batch allocation.
- **Purchase**: A transaction buying products from a supplier. Has purchase number, supplier, date, line items, total amount, and payment status.
- **Purchase Line Item**: One product entry within a purchase. Has product, quantity, unit cost, batch number, and expiry date.
- **Customer**: A person who buys from the pharmacy. Has name, phone, address, notes, and outstanding debt balance.
- **Supplier**: A company or person who sells to the pharmacy. Has name, phone, address, notes, and outstanding balance.
- **Payment**: A financial transaction (customer debt payment or supplier payment). Has date, amount, type, related entity, receiving user, and balance snapshot.
- **Expense**: A pharmacy operational cost. Has date, amount, category, description, and recording user.
- **User**: A system operator. Has username, hashed password, role (Admin/Pharmacist/Cashier), and active status.
- **Audit Log Entry**: A record of a sensitive action. Has action type, user, timestamp, entity affected, old value, new value, and details.
- **Backup**: A snapshot of the database. Has timestamp, file path, integrity status, and size.
- **License**: The activation record. Has pharmacy identity, hardware fingerprint, signature, and expiry (if applicable).
- **Recommendation**: An AI engine output. Has product reference, recommendation type, score (0-100), priority, reason, recommended action, confidence level, and calculation date.
- **Stock Adjustment**: A manual inventory change. Has product, batch, old quantity, new quantity, reason, user, and timestamp.

---

## Success Criteria

### Measurable Outcomes

- **SC-001**: A trained Cashier can complete a typical sale (scan 3 items, accept payment, print receipt) in under 60 seconds using keyboard only.
- **SC-002**: System supports a pharmacy with up to 5,000 products and 500 daily transactions without noticeable delay on target hardware.
- **SC-003**: 100% of financial and inventory operations are atomic — no partial transactions survive a crash or power failure.
- **SC-004**: Every recommendation displayed includes all four components: Score, Priority, Reason, and Recommended Action — zero unexplained recommendations.
- **SC-005**: System operates fully for all core business operations with no Internet connection.
- **SC-006**: A Pharmacist with 1 hour of training can complete the primary workflows (sale, purchase, inventory check, debt payment) without external assistance.
- **SC-007**: Backup and restore operations complete without data loss, verified by integrity check after each backup.
- **SC-008**: In LAN mode, data changes on one PC are visible on other connected PCs within 5 seconds.
- **SC-009**: All UI text is in Arabic with correct RTL layout; no untranslated strings appear during normal operation.
- **SC-010**: The application starts and is ready for the first sale within 15 seconds on target hardware (Windows 7, 2GB RAM, HDD).
- **SC-011**: Unauthorized users cannot access functions outside their role — 100% of permission-restricted actions are enforced at the business logic layer.
- **SC-012**: Audit log captures 100% of defined sensitive actions with complete details (user, timestamp, action, affected entity).

---

## Assumptions

- **Target users**: Small and medium Egyptian pharmacies with 1-4 PCs, operated by pharmacy staff with basic computer skills.
- **Hardware**: Target machines run Windows 7 SP1+ (some 32-bit), with limited RAM (2-4 GB) and HDD storage. Peripherals include 58mm/80mm thermal receipt printers and barcode/QR scanners.
- **Single currency**: All financial operations use Egyptian Pound (EGP) only. Multi-currency is out of scope.
- **Single language**: Arabic-only UI. Multi-language support is out of scope for V1.
- **No cloud features**: No online sync, cloud backup, cloud AI, or remote access in V1.
- **LAN scope**: LAN supports 2-4 PCs within the same physical pharmacy. WAN/VPN multi-location is out of scope.
- **LAN availability**: LAN feature is contingent on successful prototype validation (Architecture Discovery Section 10.1). If the prototype fails, MariaDB fallback architecture applies.
- **Expense categories**: Basic user-defined categories (rent, utilities, supplies, miscellaneous). No approval workflow for expenses.
- **Returns**: Both sale returns (customer → pharmacy) and purchase returns (pharmacy → supplier) are supported. Returns are full or partial line-item returns against the original transaction.
- **Walk-in customers**: Sales to non-registered customers (walk-ins) are supported. Debt features apply only to registered customers.
- **Reports format**: Reports can be viewed on screen, printed on any installed printer, and exported to file. Export format to be determined during planning (e.g., PDF, Excel).
- **AI Engine data requirement**: Recommendations require a minimum sales history (assumed 30 days) before producing reliable results. New products show "collecting data" status.
- **Printer compatibility**: Receipt printing targets common Egyptian pharmacy thermal printers (58mm/80mm). Specific printer models and Arabic encoding compatibility are subject to prototype validation (Architecture Discovery Section 10.2).
- **Hardware fingerprint**: The specific hardware identifiers and match tolerance for licensing are subject to prototype validation (Architecture Discovery Section 10.3).
- **Barcode format**: System supports standard 1D barcodes (EAN-13, Code 128) commonly found on Egyptian pharmaceutical products. QR code scanning is supported where scanner hardware provides it.
