# Data Model: Authentication & Permissions

**Date**: 2026-10-08
**Feature**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md)

---

## Entity: User

**Source**: Spec FR-001–FR-023, Key Entities

A system operator account. Supports login, role-based access, deactivation (never deletion), and password management.

### Fields

| Field | Type | Constraints | Description |
|---|---|---|---|
| Id | INTEGER | PRIMARY KEY, AUTOINCREMENT | Unique identifier |
| Username | TEXT | NOT NULL, UNIQUE (case-insensitive via COLLATE NOCASE) | Login identifier |
| PasswordHash | TEXT | NOT NULL | PBKDF2 hash string: `iterations:salt_base64:hash_base64` |
| Role | TEXT | NOT NULL, CHECK(Role IN ('Admin','Pharmacist','Cashier')) | One of three roles (FR-008) |
| IsActive | INTEGER | NOT NULL, DEFAULT 1 | 1 = active, 0 = deactivated (FR-019) |
| MustChangePassword | INTEGER | NOT NULL, DEFAULT 0 | 1 = force password change on next login (FR-007) |
| RowVersion | INTEGER | NOT NULL, DEFAULT 1 | Optimistic concurrency control (Arch Discovery Section 5) |
| CreatedAt | TEXT | NOT NULL | ISO 8601 timestamp of account creation |
| UpdatedAt | TEXT | NOT NULL | ISO 8601 timestamp of last modification |

### Indexes

| Index | Columns | Purpose |
|---|---|---|
| UQ_Users_Username | Username (UNIQUE) | Enforce unique usernames (FR-022) |

### Validation Rules

- Username: required, non-empty, unique case-insensitive, no leading/trailing whitespace
- PasswordHash: required, non-empty, valid format (`iterations:salt:hash`)
- Role: must be one of `Admin`, `Pharmacist`, `Cashier`
- IsActive: boolean (0 or 1)
- MustChangePassword: boolean (0 or 1)
- RowVersion: positive integer, auto-incremented on update

### Business Rules

- Users are never physically deleted from the database (FR-023). Deactivation sets `IsActive = 0`.
- The system must prevent deactivating the last active Admin (FR-021): check `SELECT COUNT(*) FROM Users WHERE Role = 'Admin' AND IsActive = 1` before deactivation.
- Username uniqueness is case-insensitive (FR-022): `admin`, `Admin`, and `ADMIN` are the same username.
- Role changes take effect at next login, not during an active session (FR-012).
- The `MustChangePassword` flag is set to 1 only for the default Admin on first run (FR-007). After the password is changed, it is set to 0.

### State Transitions

```
[Created] → (Active, MustChangePassword=0)
                ↓ Admin deactivates (FR-019)
            (Inactive)
                ↓ Admin reactivates (FR-020)
            (Active)

Default Admin only:
[Created] → (Active, MustChangePassword=1)
                ↓ Password changed on first login (FR-007)
            (Active, MustChangePassword=0)
```

---

## Entity: AuditLogEntry

**Source**: Spec FR-024–FR-029, Key Entities

An immutable record of a sensitive action. Append-only — no UPDATE or DELETE operations are permitted.

### Fields

| Field | Type | Constraints | Description |
|---|---|---|---|
| Id | INTEGER | PRIMARY KEY, AUTOINCREMENT | Unique entry identifier |
| ActionType | TEXT | NOT NULL | Type of sensitive action (see enum below) |
| UserId | INTEGER | NOT NULL, FOREIGN KEY → Users(Id) | User who performed the action |
| Timestamp | TEXT | NOT NULL | ISO 8601 timestamp of the action |
| EntityType | TEXT | NULL | Type of entity affected (e.g., 'User', 'Sale', 'Product') |
| EntityId | TEXT | NULL | Identifier of the entity affected |
| OldValue | TEXT | NULL | Previous value (JSON or text, for changes) |
| NewValue | TEXT | NULL | New value (JSON or text, for changes) |
| Details | TEXT | NULL | Free-text description with additional context |

### Action Types (FR-024)

| ActionType Value | Trigger | Example Details |
|---|---|---|
| `VoidSale` | Pharmacist/Admin voids a completed sale | Sale reference, amount, reason |
| `StockAdjustment` | Pharmacist adjusts stock manually | Product, batch, old qty, new qty, reason |
| `PriceChange` | Pharmacist/Admin changes a product price | Product, old price, new price |
| `UserCreated` | Admin creates a new user account | New username, assigned role |
| `UserRoleChanged` | Admin changes a user's role | Username, old role, new role |
| `UserPasswordReset` | Admin resets a user's password | Username (never the password) |
| `UserDeactivated` | Admin deactivates a user account | Username |
| `UserReactivated` | Admin reactivates a user account | Username |
| `PurchaseReturn` | Pharmacist/Admin processes a purchase return | Supplier, products, quantities, financial impact |
| `ConfigurationChange` | Admin changes a system setting | Setting name, old value, new value |

### Indexes

| Index | Columns | Purpose |
|---|---|---|
| IX_AuditLog_Timestamp | Timestamp DESC | Date range queries (FR-028) |
| IX_AuditLog_ActionType_Timestamp | ActionType, Timestamp DESC | Filter by action type with date ordering |
| IX_AuditLog_UserId | UserId | Filter by user (FR-028) |

### Validation Rules

- ActionType: required, must be one of the defined action types
- UserId: required, must reference an existing User
- Timestamp: required, ISO 8601 format
- EntityType, EntityId, OldValue, NewValue, Details: optional (content varies by action type)

### Business Rules

- Append-only: no UPDATE or DELETE operations (FR-026). Enforced at both application layer (IAuditLogRepository interface) and database layer (SQLite triggers).
- Each audit entry must be written within the same atomic transaction as the action it records (FR-029). If the transaction rolls back, the audit entry is also rolled back.
- Viewable only by Admin role (FR-027). The service layer enforces this permission check.
- Supports filtering by ActionType, UserId, and date range (FR-028).

### Append-Only Triggers

```sql
CREATE TRIGGER IF NOT EXISTS prevent_audit_update
BEFORE UPDATE ON AuditLog
BEGIN
  SELECT RAISE(ABORT, 'Audit log entries cannot be modified');
END;

CREATE TRIGGER IF NOT EXISTS prevent_audit_delete
BEFORE DELETE ON AuditLog
BEGIN
  SELECT RAISE(ABORT, 'Audit log entries cannot be deleted');
END;
```

---

## Database Schema (DDL Summary)

```sql
CREATE TABLE IF NOT EXISTS Users (
    Id              INTEGER PRIMARY KEY AUTOINCREMENT,
    Username        TEXT    NOT NULL COLLATE NOCASE,
    PasswordHash    TEXT    NOT NULL,
    Role            TEXT    NOT NULL CHECK(Role IN ('Admin','Pharmacist','Cashier')),
    IsActive        INTEGER NOT NULL DEFAULT 1,
    MustChangePassword INTEGER NOT NULL DEFAULT 0,
    RowVersion      INTEGER NOT NULL DEFAULT 1,
    CreatedAt       TEXT    NOT NULL,
    UpdatedAt       TEXT    NOT NULL,
    CONSTRAINT UQ_Users_Username UNIQUE (Username)
);

CREATE TABLE IF NOT EXISTS AuditLog (
    Id              INTEGER PRIMARY KEY AUTOINCREMENT,
    ActionType      TEXT    NOT NULL,
    UserId          INTEGER NOT NULL,
    Timestamp       TEXT    NOT NULL,
    EntityType      TEXT,
    EntityId        TEXT,
    OldValue        TEXT,
    NewValue        TEXT,
    Details         TEXT,
    FOREIGN KEY (UserId) REFERENCES Users(Id)
);

CREATE INDEX IF NOT EXISTS IX_AuditLog_Timestamp
    ON AuditLog(Timestamp DESC);

CREATE INDEX IF NOT EXISTS IX_AuditLog_ActionType_Timestamp
    ON AuditLog(ActionType, Timestamp DESC);

CREATE INDEX IF NOT EXISTS IX_AuditLog_UserId
    ON AuditLog(UserId);
```

---

## Entity Relationships

```
Users (1) ──── (0..*) AuditLog
  │                      │
  │ UserId ──────────────┘
  │
  └── Referenced by other modules:
      Sales (CashierId → Users.Id)
      Payments (ReceivedByUserId → Users.Id)
      StockAdjustments (UserId → Users.Id)
      etc.
```

The Users table is a dependency for every module that records who performed an action. The AuditLog table is consumed by every module that performs sensitive actions (Sale void, stock adjustment, price change, purchase return, configuration change).
