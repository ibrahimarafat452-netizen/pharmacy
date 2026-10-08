# Contract: Report Service

**Module**: Reports (Phase 8)
**Plan**: [../plan.md](../plan.md) | **Data Model**: [../data-model.md](../data-model.md)

---

## IReportService

Generates business reports. Permission: Pharmacist+. All reports support screen view, print, and PDF export.

| Method | Input | Output | Notes |
|--------|-------|--------|-------|
| GenerateSalesReport | dateRange | SalesReport | Total sales, transaction count, product breakdown, payment summary (FR-054) |
| GenerateInventoryReport | — | InventoryReport | All products, stock levels, batch details, expiry dates (FR-055) |
| GenerateExpiryReport | withinDays | ExpiryReport | Products expiring within period, sorted by urgency, qty + values (FR-056) |
| GenerateDebtReport | — | DebtReport | Customers with balances, sorted by amount, aging info (FR-057) |
| GeneratePurchaseReport | dateRange | PurchaseReport | By supplier, paid vs outstanding, product breakdown (FR-058) |
| GenerateProfitReport | dateRange | ProfitReport | Revenue, COGS, expenses, net profit (FR-059) |
| PrintReport | report, printerName? | PrintResult | Sends to printer. Uses configured default if no name |
| ExportReport | report, filePath | void | Exports to PDF file |

---

## Report Data Structures

**SalesReport**: totalRevenue (piasters), transactionCount, productBreakdown (product, qty, revenue), paymentSummary (cash, credit amounts)

**InventoryReport**: products[] (name, category, totalStock, batches[] (batchNumber, qty, expiryDate, expired))

**ExpiryReport**: items[] (product, batch, expiryDate, daysUntilExpiry, quantityAtRisk, valueAtRisk), sorted by daysUntilExpiry ASC

**DebtReport**: customers[] (name, phone, outstandingDebt, oldestUnpaidDate, agingBuckets (30/60/90+ days))

**PurchaseReport**: suppliers[] (name, totalPurchases, totalPaid, outstanding, products[] (name, qty, cost))

**ProfitReport**: totalRevenue, costOfGoodsSold, totalExpenses, netProfit, expensesByCategory
