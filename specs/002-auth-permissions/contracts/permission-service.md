# Contract: IPermissionService

**Source**: Spec FR-008–FR-015 (Role Definitions, Permissions Matrix, Permission Enforcement)

The permission service is the single source of truth for role-based access control. Every service that performs a restricted operation calls this service to check whether the current user is authorized. Permission checks happen at the service layer (FR-014), not at the UI layer.

---

## Interface

### Methods

**HasPermission(user, permission) → bool**

Checks whether the given user's role grants the specified permission.

- Input: User object (with Role property), Permission enum value
- Returns: true if the user's role includes the permission, false otherwise
- Behavior: Pure lookup against the permissions matrix (FR-013). No side effects, no database access.

**CheckPermission(user, permission) → void (throws on denial)**

Asserts that the user has the specified permission. Throws an `UnauthorizedAccessException` if denied.

- Input: User object, Permission enum value
- Behavior: Calls `HasPermission`; if false, throws with a descriptive message
- Used by services as a guard at the top of restricted methods

**GetPermissionsForRole(role) → list of Permission**

Returns all permissions granted to the specified role.

- Input: Role enum value (Admin, Pharmacist, Cashier)
- Returns: List of Permission enum values
- Used by the UI to determine which menu items/buttons to show or hide (FR-015)

---

## Permission Enum

Derived from the permissions matrix in FR-013:

| Permission Value | Admin | Pharmacist | Cashier |
|---|---|---|---|
| `CreateSale` | Yes | Yes | Yes |
| `AcceptPayment` | Yes | Yes | Yes |
| `ProcessSaleReturn` | Yes | Yes | Yes |
| `ReceiveDebtPayment` | Yes | Yes | Yes |
| `VoidSale` | Yes | Yes | No |
| `ManageInventory` | Yes | Yes | No |
| `AdjustStock` | Yes | Yes | No |
| `ConductStocktaking` | Yes | Yes | No |
| `ManagePurchases` | Yes | Yes | No |
| `ProcessPurchaseReturn` | Yes | Yes | No |
| `ManageProducts` | Yes | Yes | No |
| `ManageCustomers` | Yes | Yes | No |
| `ManageSuppliers` | Yes | Yes | No |
| `RecordExpenses` | Yes | Yes | No |
| `GenerateReports` | Yes | Yes | No |
| `ViewRecommendations` | Yes | Yes | No |
| `ConfigureRecommendations` | Yes | No | No |
| `ManageUsers` | Yes | No | No |
| `SystemConfiguration` | Yes | No | No |
| `BackupRestore` | Yes | No | No |
| `ViewAuditLog` | Yes | No | No |

---

## Design Notes

- The permissions matrix is defined in code (not in the database). Roles are fixed at three (FR-008) and permissions are not user-configurable in V1.
- Admin has all permissions — implementation can short-circuit: `if (user.Role == Role.Admin) return true;`
- The service is stateless and thread-safe. It performs no I/O.
- UI uses `GetPermissionsForRole` at login time to configure visible menu items and buttons. This is a convenience — it does NOT replace service-layer enforcement.

## Dependencies

- None (pure logic, no repository needed)

## Consumed By

- Every service that performs a restricted operation (e.g., SaleService.VoidSale calls CheckPermission(user, VoidSale) before proceeding)
- MainForm — calls GetPermissionsForRole to configure the UI after login
- All module services — as a cross-cutting dependency
