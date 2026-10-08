# Contract: Customer & Debt Services

**Module**: Customer & Debt Management (Phase 6)
**Plan**: [../plan.md](../plan.md) | **Data Model**: [../data-model.md](../data-model.md)

---

## ICustomerService

Manages customer records. Permission: Pharmacist+ for mutations, all roles for read (needed during POS).

| Method | Input | Output | Notes |
|--------|-------|--------|-------|
| CreateCustomer | name, phone?, address?, notes? | Customer | |
| UpdateCustomer | id, fields, expectedVersion | Customer | Optimistic concurrency |
| GetCustomer | id | Customer | |
| SearchCustomers | partialName | List\<Customer\> | Arabic partial match |
| ListCustomers | pagination | PagedList\<Customer\> | |
| GetAccountSummary | customerId | CustomerAccountSummary | All credit sales, all payments, returns, current debt |

---

## IDebtService

Manages customer debts and payments. Permission: all roles for receiving payments.

| Method | Input | Output | Notes |
|--------|-------|--------|-------|
| RecordDebt | customerId, saleId, amount | void | Called by ISaleService during credit sale. Updates OutstandingDebt |
| RecordPayment | customerId, amount, userId, notes? | CustomerPayment | Records balanceBefore/After. Updates OutstandingDebt. Amount must be > 0 and <= OutstandingDebt |
| GetPaymentHistory | customerId, dateRange? | List\<CustomerPayment\> | Full payment history with balance snapshots |
| GetOutstandingDebt | customerId | int | Current debt in piasters |

---

## ISaleReturnService

Processes sale returns. Permission: all roles (Cashier, Pharmacist, Admin per spec).

| Method | Input | Output | Notes |
|--------|-------|--------|-------|
| CreateSaleReturn | originalSaleId, returnItems[] | SaleReturn | Each returnItem: lineItemId, returnQty. Validates returnQty <= original - previously returned. Restocks inventory (into original batch or return batch). Adjusts customer debt/refund. Audit log entry. Atomic transaction |
| GetSaleReturn | id | SaleReturn (with line items) | |
| ListReturns | dateRange?, customerId?, pagination | PagedList\<SaleReturn\> | |
