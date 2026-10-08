# Contract: Expense Service

**Module**: Expense Tracking (Phase 7)
**Plan**: [../plan.md](../plan.md) | **Data Model**: [../data-model.md](../data-model.md)

---

## IExpenseService

Records and queries pharmacy operational expenses. Permission: Pharmacist+.

| Method | Input | Output | Notes |
|--------|-------|--------|-------|
| RecordExpense | date, amount, category, description?, userId | Expense | Creates audit log entry |
| GetExpense | id | Expense | |
| ListExpenses | dateRange?, category?, pagination | PagedList\<Expense\> | |
| GetTotalByCategory | dateRange | Dictionary\<string, int\> | Category → total piasters |
| GetTotal | dateRange | int | Overall total in piasters |
| GetCategories | — | List\<string\> | Distinct expense categories used |
