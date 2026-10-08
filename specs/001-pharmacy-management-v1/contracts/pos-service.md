# Contract: Point of Sale Services

**Module**: Point of Sale (Phase 5)
**Plan**: [../plan.md](../plan.md) | **Data Model**: [../data-model.md](../data-model.md)

---

## ISaleService

Core POS operations. Permission: all roles for sale creation; Pharmacist+ for void.

| Method | Input | Output | Notes |
|--------|-------|--------|-------|
| CreateSale | lineItems[], paidAmount, customerId? | Sale | Each lineItem: productId, quantity, unitPrice. Calls IStockService.DeductStock (FEFO). If paidAmount < total and customerId set → creates debt. Atomic transaction: sale + line items + batch deductions + payment + debt + audit |
| VoidSale | saleId, reason, userId | void | Pharmacist/Admin only. Restocks inventory (RestockBatch per BatchDeduction). Reverses financial entries. Reverses customer debt if applicable. Audit log entry |
| GetSale | id | Sale (with line items, batch deductions) | |
| ListSales | dateRange?, cashierId?, customerId?, pagination | PagedList\<Sale\> | |
| GetSaleForReceipt | saleId | ReceiptData | Structured data for receipt printing |

**Transaction atomicity**: CreateSale wraps ALL of the following in a single transaction: Sale record, SaleLineItems, BatchDeductions (via IStockService), SalePayment, Customer debt update (if credit), Inventory audit entries, Audit log entry (if applicable).

---

## IReceiptService

Generates and prints thermal receipts. Called after sale completion.

| Method | Input | Output | Notes |
|--------|-------|--------|-------|
| GenerateReceipt | receiptData | ReceiptDocument | Builds receipt layout from pharmacy config + sale data |
| PrintReceipt | receiptDocument | PrintResult | Sends to configured printer. Returns success/failure |
| ReprintReceipt | saleId | PrintResult | Reprints from stored sale data |

**ReceiptDocument contents**: Pharmacy name, logo, address, phone (from config). Sale items: name, qty, unit price, line total. Sale total, paid amount, change. Date/time, cashier name, sale number. Paper width from config (58mm/80mm).

**Error handling**: If printer not connected, sale is already complete. Return error; UI offers retry or skip.
