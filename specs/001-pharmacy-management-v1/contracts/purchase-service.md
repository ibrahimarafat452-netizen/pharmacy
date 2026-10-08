# Contract: Purchase Services

**Module**: Purchase Management & Suppliers (Phase 4)
**Plan**: [../plan.md](../plan.md) | **Data Model**: [../data-model.md](../data-model.md)

---

## IPurchaseService

Manages purchase recording and goods receipt. Permission: Pharmacist+.

| Method | Input | Output | Notes |
|--------|-------|--------|-------|
| CreatePurchase | supplierId, date, lineItems[] | Purchase | Each lineItem: productId, qty, unitCost, batchNumber, expiryDate. Auto-creates products if barcode-based entry (FR-011a). Creates batches via IBatchService |
| GetPurchase | id | Purchase (with line items) | |
| ListPurchases | supplierId?, dateRange?, pagination | PagedList\<Purchase\> | |
| CreatePurchaseReturn | originalPurchaseId, returnItems[] | PurchaseReturn | Each returnItem: lineItemId, returnQty. Removes stock from batch, reduces supplier balance, creates audit entry |

**Transaction scope**: Purchase creation wraps line items + batch creation + supplier balance update in single transaction.

---

## ISupplierService

Manages supplier records and balances. Permission: Pharmacist+.

| Method | Input | Output | Notes |
|--------|-------|--------|-------|
| CreateSupplier | name, phone?, address?, notes? | Supplier | |
| UpdateSupplier | id, fields, expectedVersion | Supplier | Optimistic concurrency |
| GetSupplier | id | Supplier | |
| ListSuppliers | pagination | PagedList\<Supplier\> | |
| RecordPayment | supplierId, amount, notes? | SupplierPayment | Updates OutstandingBalance. Records balanceBefore/After |
| GetPaymentHistory | supplierId, dateRange? | List\<SupplierPayment\> | |
| GetAccountSummary | supplierId | SupplierAccountSummary | All purchases, payments, returns, current balance |
