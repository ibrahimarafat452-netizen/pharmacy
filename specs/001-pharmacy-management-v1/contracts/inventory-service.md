# Contract: Inventory Services

**Module**: Product Catalog & Inventory (Phase 3)
**Plan**: [../plan.md](../plan.md) | **Data Model**: [../data-model.md](../data-model.md)

---

## IProductService

Manages product catalog (CRUD, search, import). Permission: Pharmacist+ for mutations, all roles for read.

| Method | Input | Output | Notes |
|--------|-------|--------|-------|
| CreateProduct | name, barcode?, categoryId, unitType, stripsPerBox, sellingPrice | Product | Validates uniqueness (barcode). Audit: price change if relevant |
| UpdateProduct | id, fields, expectedVersion | Product | Optimistic concurrency. Price change → audit log |
| DeactivateProduct | id | void | Sets status=inactive. Does not delete |
| GetProduct | id | Product | |
| GetProductByBarcode | barcode | Product? | Returns null if not found |
| SearchByName | partialName | List\<Product\> | Arabic partial match (LIKE %term%) |
| ListProducts | filters?, pagination | PagedList\<Product\> | By category, status, etc. |
| ImportFromCsv | filePath | ImportResult | Returns success/error per row. One-time catalog setup |

---

## ICategoryService

Manages product categories. Permission: Admin for mutations.

| Method | Input | Output | Notes |
|--------|-------|--------|-------|
| CreateCategory | name | Category | Unique name check |
| UpdateCategory | id, name | Category | |
| DeleteCategory | id | void | Blocked if products reference it |
| ListCategories | — | List\<Category\> | Ordered by SortOrder |
| SeedDefaults | — | void | Creates system default categories on first run |

---

## IBatchService

Manages inventory batches. Typically called through Purchase or Stock services, not directly from UI.

| Method | Input | Output | Notes |
|--------|-------|--------|-------|
| CreateBatch | productId, batchNumber, quantity, purchasePrice, expiryDate, supplierId? | Batch | Quantity in strips |
| UpdateBatchQuantity | id, newQuantity, expectedVersion | Batch | Optimistic concurrency |
| GetBatchesForProduct | productId | List\<Batch\> | All batches (including expired) |
| GetAvailableBatches | productId | List\<Batch\> | Non-expired, quantity > 0, ordered by ExpiryDate ASC (FEFO) |
| GetAvailableStock | productId | int | Sum of available batch quantities (strips) |
| FlagExpiredBatches | — | int | Bulk update IsExpired for past-expiry batches. Run on startup + periodically |

---

## IStockService

Orchestrates inventory operations (deduction, restock, adjustment). All operations within atomic transactions.

| Method | Input | Output | Notes |
|--------|-------|--------|-------|
| DeductStock | productId, quantity | List\<BatchDeduction\> | FEFO order. Fails if insufficient stock. Quantity in strips |
| RestockBatch | batchId, quantity | void | Returns stock to a specific batch (for voids/returns) |
| AdjustStock | productId, batchId, newQuantity, reason, userId | StockAdjustment | Mandatory reason. Creates audit log entry |
| GetStockSummary | productId | StockSummary | Total available, batch breakdown, expiry info |

**StockSummary**: totalAvailableStrips, batches (id, batchNumber, quantity, expiryDate, isExpired), nearestExpiry

---

## IInventoryAuditService

Records all inventory changes. Called internally by Stock/Sale/Purchase services.

| Method | Input | Output | Notes |
|--------|-------|--------|-------|
| RecordChange | changeType, productId, batchId, quantity, userId, details | void | Types: sale, purchase, adjustment, return, stocktaking |
| GetHistory | productId?, dateRange?, changeType? | List\<InventoryAuditEntry\> | Filterable query |
