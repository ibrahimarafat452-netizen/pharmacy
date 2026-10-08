# Contract: IUserService

**Source**: Spec FR-016–FR-023 (User Management)

The user service provides user account management for Admin users: creating, editing, deactivating, reactivating accounts, and resetting passwords. All operations are restricted to Admin role and generate audit log entries.

---

## Interface

### Methods

**CreateUser(username, password, role) → User**

Creates a new user account.

- Input: username (string), password (string), role (Role enum)
- Returns: the created User entity
- Permission: `ManageUsers` (Admin only)
- Validation:
  - Username is non-empty and has no leading/trailing whitespace
  - Username is unique (case-insensitive) — throws if duplicate exists (FR-022)
  - Password meets minimum length requirement
  - Role is a valid enum value (Admin, Pharmacist, or Cashier)
- Behavior:
  - Hashes the password with PBKDF2 (FR-034)
  - Sets IsActive = true, MustChangePassword = false, RowVersion = 1
  - Writes audit log entry: ActionType=`UserCreated`, details include username and role (FR-024)
  - Both user creation and audit entry within the same transaction (FR-029)

**ChangeRole(userId, newRole, expectedRowVersion) → User**

Changes a user's assigned role.

- Input: userId (int), newRole (Role enum), expectedRowVersion (int)
- Returns: the updated User entity
- Permission: `ManageUsers` (Admin only)
- Validation: newRole is valid; user exists and is not being concurrently modified (optimistic concurrency)
- Behavior:
  - Updates Role column, increments RowVersion
  - Writes audit log entry: ActionType=`UserRoleChanged`, OldValue=previous role, NewValue=new role
  - Role change takes effect at the user's next login (FR-012)

**ResetPassword(userId, newPassword, expectedRowVersion) → void**

Resets a user's password.

- Input: userId (int), newPassword (string), expectedRowVersion (int)
- Permission: `ManageUsers` (Admin only)
- Validation: newPassword meets minimum length
- Behavior:
  - Hashes the new password (FR-034)
  - Updates PasswordHash, increments RowVersion
  - Writes audit log entry: ActionType=`UserPasswordReset` — never logs the password itself (FR-035)
  - If the Admin resets their own password, it takes effect immediately in the current session (edge case from spec)

**ChangeOwnPassword(userId, currentPassword, newPassword) → void**

Allows any user to change their own password (used during forced password change on first login).

- Input: userId (int), currentPassword (string), newPassword (string)
- Permission: any authenticated user (no ManageUsers requirement)
- Validation:
  - Current password must match stored hash
  - New password meets minimum length and differs from current password
- Behavior:
  - Verifies current password
  - Hashes new password, updates PasswordHash
  - Sets MustChangePassword = false (if it was true)
  - Increments RowVersion

**DeactivateUser(userId, expectedRowVersion) → void**

Deactivates a user account.

- Input: userId (int), expectedRowVersion (int)
- Permission: `ManageUsers` (Admin only)
- Validation:
  - User exists
  - User is currently active
  - User is NOT the last active Admin (FR-021): check count of active Admins before deactivation
- Behavior:
  - Sets IsActive = false, increments RowVersion
  - Writes audit log entry: ActionType=`UserDeactivated`
  - User can no longer log in (FR-019)

**ReactivateUser(userId, expectedRowVersion) → void**

Reactivates a previously deactivated user account.

- Input: userId (int), expectedRowVersion (int)
- Permission: `ManageUsers` (Admin only)
- Behavior:
  - Sets IsActive = true, increments RowVersion
  - Writes audit log entry: ActionType=`UserReactivated`

**GetAllUsers() → list of User**

Returns all user accounts (active and inactive) for the Admin management screen.

- Permission: `ManageUsers` (Admin only)
- Returns: list of User entities (without PasswordHash for display)

**GetUserById(userId) → User**

Returns a single user by ID.

- Permission: `ManageUsers` (Admin only) or the user themselves (for own profile)

---

## Dependencies

- `IPermissionService` — checks ManageUsers permission on every operation
- `IUserRepository` — data access
- `IAuditLogService` — writes audit entries for all user management actions
- `PasswordHasher` — hashes new passwords

## Consumed By

- `UserManagementForm` — Admin user CRUD interface
- `ChangePasswordForm` — calls ChangeOwnPassword for forced password change
