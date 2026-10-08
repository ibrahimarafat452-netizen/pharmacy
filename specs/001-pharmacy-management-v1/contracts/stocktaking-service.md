# Contract: Stocktaking Service

**Module**: Stocktaking (Phase 7)
**Plan**: [../plan.md](../plan.md) | **Data Model**: [../data-model.md](../data-model.md)

---

## IStocktakeService

Manages physical inventory count workflow. Permission: Pharmacist+.

| Method | Input | Output | Notes |
|--------|-------|--------|-------|
| InitiateStocktake | scope (all or categoryIds[]), userId | Stocktake | Creates StocktakeItems for all products/batches in scope with RecordedQuantity snapshot |
| GetStocktake | id | Stocktake (with items) | |
| UpdateCountedQuantity | stocktakeId, itemId, countedQuantity | StocktakeItem | Calculates Discrepancy = counted - recorded |
| GetDiscrepancies | stocktakeId | List\<StocktakeItem\> | Only items where counted != recorded |
| ApproveAdjustments | stocktakeId, userId | void | For each discrepancy: calls IStockService.AdjustStock with reason "Stocktaking". Creates StockAdjustment and audit entries. Sets Stocktake.Status = completed |
| CancelStocktake | stocktakeId | void | Sets status = cancelled. No adjustments applied |
| ListStocktakes | pagination | PagedList\<Stocktake\> | |
