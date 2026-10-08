# Data Model: Pharmacy Management System V1

**Date**: 2026-10-08
**Feature**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md) | **Research**: [research.md](research.md)

**Note**: Auth entities (User, AuditLogEntry) are fully detailed in `specs/002-auth-permissions/data-model.md`. This document defines all remaining entities and shows cross-entity relationships.

---

## Entity Overview

```
Product ──< Batch
Product ──< SaleLineItem
Product ──< PurchaseLineItem
Product ──< Recommendation
Product ──< StockAdjustment
Product >── Category

Sale ──< SaleLineItem
Sale ──< SalePayment
Sale >── Customer (nullable — walk-in)
Sale >── User (cashier)

Purchase ──< PurchaseLineItem
Purchase ──< PurchasePayment
Purchase >── Supplier

Customer ──< Sale (credit sales)
Customer ──< CustomerPayment

Supplier ──< Purchase
Supplier ──< SupplierPayment

Batch >── Product
Batch >── Supplier (reference)

SaleLineItem ──< BatchDeduction

Expense >── User (recorder)

StockAdjustment >── Product
StockAdjustment >── Batch
StockAdjustment >── User

User ──< AuditLogEntry (performer)
```

Key: `──<` = one-to-many, `>──` = many-to-one (FK)

---

## Entity Definitions

### Product

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| Id | INTEGER | PK, AUTOINCREMENT | |
| Name | TEXT | NOT NULL | Arabic product name |
| Barcode | TEXT | UNIQUE, nullable | EAN-13, Code 128, or null |
| CategoryId | INTEGER | FK → Category.Id, NOT NULL | |
| UnitType | TEXT | NOT NULL, CHECK('box','strip') | Selling unit |
| StripsPerBox | INTEGER | NOT NULL, DEFAULT 1, CHECK(> 0) | 1 for strip-only products |
| SellingPrice | INTEGER | NOT NULL, CHECK(>= 0) | In piasters (EGP × 100) to avoid floating point |
| Status | TEXT | NOT NULL, DEFAULT 'active', CHECK('active','inactive') | |
| RowVersion | INTEGER | NOT NULL, DEFAULT 1 | Optimistic concurrency |
| CreatedAt | TEXT | NOT NULL | ISO 8601 |
| UpdatedAt | TEXT | NOT NULL | ISO 8601 |

**Validation Rules**:
- Name is required (non-empty Arabic text)
- Barcode must be unique if provided; null is allowed (products without barcode)
- StripsPerBox must be >= 1
- SellingPrice stored as piasters (integer) to maintain financial accuracy (Constitution VIII)

**State Transitions**: active → inactive (Admin deactivates). Inactive products excluded from new sales but visible in historical records.

---

### Category

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| Id | INTEGER | PK, AUTOINCREMENT | |
| Name | TEXT | NOT NULL, UNIQUE | Arabic category name |
| IsSystem | INTEGER | NOT NULL, DEFAULT 0 | 1 = system default, 0 = user-created |
| SortOrder | INTEGER | NOT NULL, DEFAULT 0 | Display ordering |
| CreatedAt | TEXT | NOT NULL | ISO 8601 |

**Validation Rules**:
- Name must be unique (case-insensitive)
- System categories can be edited but deletion is prevented if products reference them
- Admin can add, edit, and remove user-created categories (FR-011)

---

### Batch

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| Id | INTEGER | PK, AUTOINCREMENT | |
| ProductId | INTEGER | FK → Product.Id, NOT NULL | |
| BatchNumber | TEXT | NOT NULL | Supplier's batch identifier |
| Quantity | INTEGER | NOT NULL, CHECK(>= 0) | In strips (smallest unit) |
| PurchasePrice | INTEGER | NOT NULL, CHECK(>= 0) | Per strip, in piasters |
| ExpiryDate | TEXT | NOT NULL | ISO 8601 date |
| SupplierId | INTEGER | FK → Supplier.Id, nullable | Which supplier provided this batch |
| PurchaseId | INTEGER | FK → Purchase.Id, nullable | Which purchase brought this batch |
| IsExpired | INTEGER | NOT NULL, DEFAULT 0 | Computed flag; set by expiry check |
| RowVersion | INTEGER | NOT NULL, DEFAULT 1 | Optimistic concurrency |
| CreatedAt | TEXT | NOT NULL | ISO 8601 |

**Validation Rules**:
- Quantity cannot go negative (FR-014)
- ExpiryDate is required; expired batches are excluded from sale (FR-016)
- BatchNumber + ProductId should be unique per supplier (prevent duplicate batch entries)
- Quantity is in strips regardless of product's selling unit

**FEFO**: Batches selected for deduction by `ORDER BY ExpiryDate ASC WHERE Quantity > 0 AND IsExpired = 0`

---

### Sale

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| Id | INTEGER | PK, AUTOINCREMENT | |
| SaleNumber | TEXT | NOT NULL, UNIQUE | Sequential sale identifier |
| CustomerId | INTEGER | FK → Customer.Id, nullable | Null for walk-in customers |
| UserId | INTEGER | FK → User.Id, NOT NULL | Cashier who created the sale |
| SaleDate | TEXT | NOT NULL | ISO 8601 datetime |
| TotalAmount | INTEGER | NOT NULL, CHECK(>= 0) | In piasters |
| PaidAmount | INTEGER | NOT NULL, CHECK(>= 0) | In piasters |
| ChangeAmount | INTEGER | NOT NULL, DEFAULT 0 | In piasters (cash change returned) |
| DebtAmount | INTEGER | NOT NULL, DEFAULT 0 | In piasters (TotalAmount - PaidAmount, if credit) |
| Status | TEXT | NOT NULL, DEFAULT 'completed', CHECK('completed','voided') | |
| VoidedByUserId | INTEGER | FK → User.Id, nullable | Who voided (if voided) |
| VoidedAt | TEXT | nullable | When voided |
| VoidReason | TEXT | nullable | Why voided |
| RowVersion | INTEGER | NOT NULL, DEFAULT 1 | |
| CreatedAt | TEXT | NOT NULL | ISO 8601 |

**Validation Rules**:
- DebtAmount > 0 only when CustomerId is not null (walk-ins cannot have credit)
- Void requires Pharmacist or Admin role (FR-009)
- Voiding reverses all inventory deductions and financial entries in the same transaction

**State Transitions**: completed → voided (Pharmacist/Admin voids)

---

### SaleLineItem

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| Id | INTEGER | PK, AUTOINCREMENT | |
| SaleId | INTEGER | FK → Sale.Id, NOT NULL | |
| ProductId | INTEGER | FK → Product.Id, NOT NULL | |
| Quantity | INTEGER | NOT NULL, CHECK(> 0) | In strips |
| UnitPrice | INTEGER | NOT NULL, CHECK(>= 0) | Per selling unit, in piasters |
| LineTotal | INTEGER | NOT NULL, CHECK(>= 0) | In piasters |

**Validation Rules**:
- Quantity in strips (converted from selling unit at entry time)
- LineTotal = (Quantity / StripsPerBox or Quantity) × UnitPrice depending on selling unit

---

### BatchDeduction

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| Id | INTEGER | PK, AUTOINCREMENT | |
| SaleLineItemId | INTEGER | FK → SaleLineItem.Id, NOT NULL | |
| BatchId | INTEGER | FK → Batch.Id, NOT NULL | |
| Quantity | INTEGER | NOT NULL, CHECK(> 0) | Strips deducted from this batch |

**Purpose**: Records which batches were deducted for each sale line item. Enables accurate void/return stock restoration.

---

### SaleReturn

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| Id | INTEGER | PK, AUTOINCREMENT | |
| OriginalSaleId | INTEGER | FK → Sale.Id, NOT NULL | |
| CustomerId | INTEGER | FK → Customer.Id, nullable | |
| UserId | INTEGER | FK → User.Id, NOT NULL | Who processed the return |
| ReturnDate | TEXT | NOT NULL | ISO 8601 datetime |
| TotalRefundAmount | INTEGER | NOT NULL | In piasters |
| CreatedAt | TEXT | NOT NULL | ISO 8601 |

---

### SaleReturnLineItem

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| Id | INTEGER | PK, AUTOINCREMENT | |
| SaleReturnId | INTEGER | FK → SaleReturn.Id, NOT NULL | |
| OriginalLineItemId | INTEGER | FK → SaleLineItem.Id, NOT NULL | |
| ProductId | INTEGER | FK → Product.Id, NOT NULL | |
| ReturnQuantity | INTEGER | NOT NULL, CHECK(> 0) | In strips |
| RefundAmount | INTEGER | NOT NULL | In piasters |
| RestockedBatchId | INTEGER | FK → Batch.Id, NOT NULL | Batch where stock was returned |

**Validation Rules**:
- ReturnQuantity cannot exceed original sale line item quantity minus previously returned quantity
- Stock is restocked into the original batch (from BatchDeduction) or a return batch

---

### Purchase

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| Id | INTEGER | PK, AUTOINCREMENT | |
| PurchaseNumber | TEXT | NOT NULL, UNIQUE | Sequential purchase identifier |
| SupplierId | INTEGER | FK → Supplier.Id, NOT NULL | |
| UserId | INTEGER | FK → User.Id, NOT NULL | Who recorded the purchase |
| PurchaseDate | TEXT | NOT NULL | ISO 8601 date |
| TotalAmount | INTEGER | NOT NULL, CHECK(>= 0) | In piasters |
| PaidAmount | INTEGER | NOT NULL, DEFAULT 0 | In piasters |
| Status | TEXT | NOT NULL, DEFAULT 'received', CHECK('received','partial_return','full_return') | |
| RowVersion | INTEGER | NOT NULL, DEFAULT 1 | |
| CreatedAt | TEXT | NOT NULL | ISO 8601 |

---

### PurchaseLineItem

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| Id | INTEGER | PK, AUTOINCREMENT | |
| PurchaseId | INTEGER | FK → Purchase.Id, NOT NULL | |
| ProductId | INTEGER | FK → Product.Id, NOT NULL | |
| Quantity | INTEGER | NOT NULL, CHECK(> 0) | In strips |
| UnitCost | INTEGER | NOT NULL, CHECK(>= 0) | Per strip, in piasters |
| LineTotal | INTEGER | NOT NULL | In piasters |
| BatchNumber | TEXT | NOT NULL | Supplier's batch identifier |
| ExpiryDate | TEXT | NOT NULL | ISO 8601 date |
| BatchId | INTEGER | FK → Batch.Id, nullable | Created batch (set after goods receipt) |

---

### PurchaseReturn

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| Id | INTEGER | PK, AUTOINCREMENT | |
| OriginalPurchaseId | INTEGER | FK → Purchase.Id, NOT NULL | |
| SupplierId | INTEGER | FK → Supplier.Id, NOT NULL | |
| UserId | INTEGER | FK → User.Id, NOT NULL | Who processed the return |
| ReturnDate | TEXT | NOT NULL | ISO 8601 datetime |
| TotalReturnAmount | INTEGER | NOT NULL | In piasters |
| CreatedAt | TEXT | NOT NULL | ISO 8601 |

---

### PurchaseReturnLineItem

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| Id | INTEGER | PK, AUTOINCREMENT | |
| PurchaseReturnId | INTEGER | FK → PurchaseReturn.Id, NOT NULL | |
| OriginalLineItemId | INTEGER | FK → PurchaseLineItem.Id, NOT NULL | |
| ProductId | INTEGER | FK → Product.Id, NOT NULL | |
| ReturnQuantity | INTEGER | NOT NULL, CHECK(> 0) | In strips |
| ReturnAmount | INTEGER | NOT NULL | In piasters |
| BatchId | INTEGER | FK → Batch.Id, NOT NULL | Batch from which stock was removed |

---

### Customer

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| Id | INTEGER | PK, AUTOINCREMENT | |
| Name | TEXT | NOT NULL | Arabic customer name |
| Phone | TEXT | nullable | |
| Address | TEXT | nullable | |
| Notes | TEXT | nullable | |
| OutstandingDebt | INTEGER | NOT NULL, DEFAULT 0 | In piasters; denormalized running total |
| RowVersion | INTEGER | NOT NULL, DEFAULT 1 | |
| CreatedAt | TEXT | NOT NULL | ISO 8601 |
| UpdatedAt | TEXT | NOT NULL | ISO 8601 |

**Validation Rules**:
- OutstandingDebt is a denormalized sum (credit sale debts - payments). Maintained within transactions.
- No arbitrary debt limit (edge case from spec) — system allows credit but displays balance prominently

---

### CustomerPayment

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| Id | INTEGER | PK, AUTOINCREMENT | |
| CustomerId | INTEGER | FK → Customer.Id, NOT NULL | |
| UserId | INTEGER | FK → User.Id, NOT NULL | Who received the payment |
| PaymentDate | TEXT | NOT NULL | ISO 8601 datetime |
| Amount | INTEGER | NOT NULL, CHECK(> 0) | In piasters |
| BalanceBefore | INTEGER | NOT NULL | In piasters |
| BalanceAfter | INTEGER | NOT NULL | In piasters |
| Notes | TEXT | nullable | |
| CreatedAt | TEXT | NOT NULL | ISO 8601 |

---

### Supplier

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| Id | INTEGER | PK, AUTOINCREMENT | |
| Name | TEXT | NOT NULL | Supplier name |
| Phone | TEXT | nullable | |
| Address | TEXT | nullable | |
| Notes | TEXT | nullable | |
| OutstandingBalance | INTEGER | NOT NULL, DEFAULT 0 | In piasters; denormalized (purchases - payments - returns) |
| RowVersion | INTEGER | NOT NULL, DEFAULT 1 | |
| CreatedAt | TEXT | NOT NULL | ISO 8601 |
| UpdatedAt | TEXT | NOT NULL | ISO 8601 |

---

### SupplierPayment

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| Id | INTEGER | PK, AUTOINCREMENT | |
| SupplierId | INTEGER | FK → Supplier.Id, NOT NULL | |
| UserId | INTEGER | FK → User.Id, NOT NULL | Who recorded the payment |
| PaymentDate | TEXT | NOT NULL | ISO 8601 datetime |
| Amount | INTEGER | NOT NULL, CHECK(> 0) | In piasters |
| BalanceBefore | INTEGER | NOT NULL | In piasters |
| BalanceAfter | INTEGER | NOT NULL | In piasters |
| Notes | TEXT | nullable | |
| CreatedAt | TEXT | NOT NULL | ISO 8601 |

---

### Expense

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| Id | INTEGER | PK, AUTOINCREMENT | |
| ExpenseDate | TEXT | NOT NULL | ISO 8601 date |
| Amount | INTEGER | NOT NULL, CHECK(> 0) | In piasters |
| Category | TEXT | NOT NULL | User-defined (rent, utilities, supplies, misc) |
| Description | TEXT | nullable | |
| UserId | INTEGER | FK → User.Id, NOT NULL | Who recorded the expense |
| CreatedAt | TEXT | NOT NULL | ISO 8601 |

---

### StockAdjustment

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| Id | INTEGER | PK, AUTOINCREMENT | |
| ProductId | INTEGER | FK → Product.Id, NOT NULL | |
| BatchId | INTEGER | FK → Batch.Id, NOT NULL | |
| OldQuantity | INTEGER | NOT NULL | In strips |
| NewQuantity | INTEGER | NOT NULL, CHECK(>= 0) | In strips |
| Reason | TEXT | NOT NULL | Mandatory reason (FR-017) |
| UserId | INTEGER | FK → User.Id, NOT NULL | |
| AdjustmentType | TEXT | NOT NULL, CHECK('manual','stocktaking') | |
| CreatedAt | TEXT | NOT NULL | ISO 8601 |

---

### Stocktake

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| Id | INTEGER | PK, AUTOINCREMENT | |
| InitiatedByUserId | INTEGER | FK → User.Id, NOT NULL | |
| Status | TEXT | NOT NULL, DEFAULT 'in_progress', CHECK('in_progress','completed','cancelled') | |
| Scope | TEXT | NOT NULL | 'all' or category IDs |
| StartedAt | TEXT | NOT NULL | ISO 8601 |
| CompletedAt | TEXT | nullable | ISO 8601 |

---

### StocktakeItem

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| Id | INTEGER | PK, AUTOINCREMENT | |
| StocktakeId | INTEGER | FK → Stocktake.Id, NOT NULL | |
| ProductId | INTEGER | FK → Product.Id, NOT NULL | |
| BatchId | INTEGER | FK → Batch.Id, NOT NULL | |
| RecordedQuantity | INTEGER | NOT NULL | System quantity at time of stocktake |
| CountedQuantity | INTEGER | nullable | Null until counted |
| Discrepancy | INTEGER | nullable | CountedQuantity - RecordedQuantity |
| AdjustmentApproved | INTEGER | NOT NULL, DEFAULT 0 | |

---

### Recommendation

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| Id | INTEGER | PK, AUTOINCREMENT | |
| ProductId | INTEGER | FK → Product.Id, NOT NULL | |
| Type | TEXT | NOT NULL | reorder, trend, expiry_risk, slow_moving, overstock, stockout |
| Score | INTEGER | NOT NULL, CHECK(0–100) | |
| Priority | TEXT | NOT NULL | Critical, High, Medium, Low |
| Reason | TEXT | NOT NULL | Human-readable explanation |
| RecommendedAction | TEXT | NOT NULL | Suggested action text |
| ConfidenceLevel | TEXT | NOT NULL | sufficient, collecting_data |
| CalculatedAt | TEXT | NOT NULL | ISO 8601 datetime |

**Validation Rules**:
- Every recommendation MUST have Score, Priority, Reason, and RecommendedAction (FR-050, FR-051)
- Products with < 30 days history: ConfidenceLevel = 'collecting_data' (FR-052)
- Recommendations are recalculated periodically; old rows replaced

---

### BackupRecord

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| Id | INTEGER | PK, AUTOINCREMENT | |
| FilePath | TEXT | NOT NULL | Full path to backup file |
| BackupDate | TEXT | NOT NULL | ISO 8601 datetime |
| SizeBytes | INTEGER | NOT NULL | |
| IntegrityStatus | TEXT | NOT NULL | passed, failed |
| CreatedAt | TEXT | NOT NULL | ISO 8601 |

---

### PharmacyConfig

| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| Key | TEXT | PK | Setting key |
| Value | TEXT | NOT NULL | Setting value (JSON or plain text) |
| UpdatedAt | TEXT | NOT NULL | ISO 8601 |

**Known Keys**: `pharmacy_name`, `pharmacy_logo_path`, `pharmacy_address`, `pharmacy_phone`, `receipt_paper_width`, `printer_name`, `backup_interval_minutes`, `backup_location`, `backup_retention_count`, `recommendation_safety_stock_days`, `recommendation_lead_time_days`, `recommendation_overstock_limit`, `lan_mode` (single/server/client), `lan_pharmacy_secret`, `lan_server_address`

---

### License (stored as file, not in database)

| Field | Description |
|-------|-------------|
| PharmacyId | Unique pharmacy identifier |
| HardwareFingerprint | Bound hardware identifiers |
| IssuedAt | License creation date |
| ExpiresAt | License expiry (if applicable) |
| Signature | RSA signature of the above fields |

**Note**: License is a signed file, verified at startup. The vendor's public key is embedded in the application. The license file is NOT stored in the database — it is a standalone file alongside the application.

---

## Financial Amount Convention

All monetary amounts are stored as **INTEGER in piasters** (1 EGP = 100 piasters). This avoids floating-point precision issues and maintains financial accuracy per Constitution Principle VIII.

- Display: divide by 100 and format as EGP with 2 decimal places
- Input: multiply by 100 before storage
- Arithmetic: all calculations on integers; rounding only at display time

---

## Indexes

### Performance-Critical Indexes

```sql
-- Product lookup
CREATE INDEX IX_Product_Barcode ON Product(Barcode) WHERE Barcode IS NOT NULL;
CREATE INDEX IX_Product_CategoryId ON Product(CategoryId);
CREATE INDEX IX_Product_Name ON Product(Name);

-- Batch queries (FEFO, expiry)
CREATE INDEX IX_Batch_ProductId_ExpiryDate ON Batch(ProductId, ExpiryDate ASC)
  WHERE Quantity > 0 AND IsExpired = 0;
CREATE INDEX IX_Batch_ExpiryDate ON Batch(ExpiryDate ASC);

-- Sale queries
CREATE INDEX IX_Sale_SaleDate ON Sale(SaleDate DESC);
CREATE INDEX IX_Sale_CustomerId ON Sale(CustomerId) WHERE CustomerId IS NOT NULL;
CREATE INDEX IX_SaleLineItem_SaleId ON SaleLineItem(SaleId);
CREATE INDEX IX_SaleLineItem_ProductId ON SaleLineItem(ProductId);

-- Purchase queries
CREATE INDEX IX_Purchase_SupplierId ON Purchase(SupplierId);
CREATE INDEX IX_Purchase_PurchaseDate ON Purchase(PurchaseDate DESC);
CREATE INDEX IX_PurchaseLineItem_PurchaseId ON PurchaseLineItem(PurchaseId);

-- Payment queries
CREATE INDEX IX_CustomerPayment_CustomerId ON CustomerPayment(CustomerId);
CREATE INDEX IX_SupplierPayment_SupplierId ON SupplierPayment(SupplierId);

-- Audit log (see specs/002-auth-permissions/research.md)
CREATE INDEX IX_AuditLog_Timestamp ON AuditLog(Timestamp DESC);
CREATE INDEX IX_AuditLog_ActionType_Timestamp ON AuditLog(ActionType, Timestamp DESC);
CREATE INDEX IX_AuditLog_UserId ON AuditLog(UserId);

-- Recommendation queries
CREATE INDEX IX_Recommendation_ProductId ON Recommendation(ProductId);
CREATE INDEX IX_Recommendation_Type ON Recommendation(Type);

-- Expense queries
CREATE INDEX IX_Expense_ExpenseDate ON Expense(ExpenseDate DESC);
```
