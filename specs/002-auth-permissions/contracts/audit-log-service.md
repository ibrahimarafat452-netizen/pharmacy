# Contract: IAuditLogService

**Source**: Spec FR-024–FR-029 (Audit Log)

The audit log service records sensitive actions and provides filtered querying for the Admin audit log viewer. It is consumed by every module that performs a sensitive action. The service enforces append-only semantics — no update or delete operations exist.

---

## Interface

### Methods

**LogAction(actionType, userId, entityType, entityId, oldValue, newValue, details, transaction) → void**

Records a sensitive action in the audit log.

- Input:
  - actionType (string): one of the defined action types (see data-model.md)
  - userId (int): the user who performed the action
  - entityType (string, optional): type of entity affected (e.g., "User", "Sale", "Product")
  - entityId (string, optional): identifier of the affected entity
  - oldValue (string, optional): previous value (JSON or text)
  - newValue (string, optional): new value (JSON or text)
  - details (string, optional): free-text description
  - transaction (IDbTransaction): the database transaction to participate in
- Returns: void
- Behavior:
  - Inserts a new AuditLogEntry with the current timestamp
  - The insert MUST use the provided transaction so that the audit entry commits or rolls back with the action it records (FR-029)
  - No permission check needed — this is an internal service called by other services, not directly by users
  - No validation on who can write — any service that performs a sensitive action writes through this method

**GetEntries(filter) → paginated list of AuditLogEntry**

Queries audit log entries with filtering.

- Input: `AuditLogFilter` object with optional fields:
  - actionType (string, optional): filter by action type
  - userId (int, optional): filter by user who performed the action
  - fromDate (DateTime, optional): start of date range
  - toDate (DateTime, optional): end of date range
  - pageNumber (int): 1-based page number
  - pageSize (int): entries per page
- Returns: paginated result with entries (reverse chronological) and total count
- Permission: `ViewAuditLog` (Admin only) — checked before returning results (FR-027)

---

## AuditLogFilter Structure

| Field | Type | Required | Description |
|---|---|---|---|
| ActionType | string | No | Filter to specific action type |
| UserId | int? | No | Filter to specific user |
| FromDate | DateTime? | No | Start of date range (inclusive) |
| ToDate | DateTime? | No | End of date range (inclusive) |
| PageNumber | int | Yes | 1-based page number |
| PageSize | int | Yes | Entries per page (default: 50) |

---

## Design Notes

- `LogAction` accepts the active `IDbTransaction` as a parameter. This is critical for FR-029: the audit entry must commit or rollback with the action. The caller (e.g., SaleService.VoidSale) opens a transaction, performs the business operation, calls `LogAction` within the same transaction, and then commits.
- The `IAuditLogRepository` interface exposes only `Insert` and `Query` methods — no `Update` or `Delete` (FR-026).
- Pagination is required because the audit log grows indefinitely (Constitution XI — never pruned).

## Dependencies

- `IAuditLogRepository` — data access (append and query only)
- `IPermissionService` — checks ViewAuditLog permission on GetEntries

## Consumed By

- `UserService` — logs UserCreated, UserRoleChanged, UserPasswordReset, UserDeactivated, UserReactivated
- `SaleService` (future) — logs VoidSale
- `InventoryService` (future) — logs StockAdjustment
- `ProductService` (future) — logs PriceChange
- `PurchaseService` (future) — logs PurchaseReturn
- `ConfigService` (future) — logs ConfigurationChange
- `AuditLogForm` — calls GetEntries for display
