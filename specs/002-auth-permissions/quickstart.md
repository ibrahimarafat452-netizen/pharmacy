# Quickstart Validation Guide: Authentication & Permissions

**Date**: 2026-10-08
**Feature**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md)

---

## Prerequisites

- Visual Studio 2022 Community with .NET Framework 4.8 targeting pack
- Solution built successfully with no errors
- SQLite database file created (WAL mode)
- NUnit test adapter installed

---

## Validation Scenario 1: First-Run Bootstrap (FR-006, FR-007)

**Goal**: Verify that the system creates a default Admin account on first launch and forces a password change.

### Steps

1. Delete the database file (or start with a fresh/empty database)
2. Launch the application
3. Verify: Login screen appears with username and password fields
4. Enter username `admin`, password `admin`
5. Verify: System authenticates and immediately shows the Change Password form (not the main interface)
6. Enter a new password (e.g., `pharmacy2026`)
7. Verify: Password change succeeds, main interface appears with Admin-level access

### Expected Outcome

- Default Admin account auto-created on first run
- Login with default credentials succeeds
- Password change is mandatory before accessing any other feature
- After password change, full Admin interface is accessible

---

## Validation Scenario 2: Login & Logout (FR-001–FR-005)

**Goal**: Verify authentication accepts valid credentials, rejects invalid ones with generic messages, and logout works.

### Steps

1. From the login screen, enter a valid username and correct password
2. Verify: Main interface appears, showing the user's name and role
3. Click Logout
4. Verify: Login screen reappears
5. Enter a valid username with an incorrect password
6. Verify: Error message appears — note the exact text
7. Enter a non-existent username with any password
8. Verify: Error message appears — compare with step 6 (must be identical, FR-003)
9. Enter empty username or empty password
10. Verify: Validation message appears without attempting authentication

### Expected Outcome

- Valid credentials → authenticated, role-appropriate interface
- Invalid password → generic error (no hint about which field was wrong)
- Invalid username → same generic error (indistinguishable from invalid password)
- Logout → return to login screen
- Empty fields → input validation message

---

## Validation Scenario 3: Role-Based Permission Enforcement (FR-008–FR-015)

**Goal**: Verify each role can only access its permitted functions, and enforcement is at the service layer.

### Steps

1. Log in as Admin → verify all menu items/functions are accessible
2. Create a Pharmacist user and a Cashier user (via User Management)
3. Log out, log in as Pharmacist:
   - Verify: Can access sales, inventory, purchases, reports, expenses, recommendations
   - Verify: Cannot access user management, system configuration, backup/restore, audit log
   - Verify: Can void a sale
4. Log out, log in as Cashier:
   - Verify: Can access sales and payments only
   - Verify: Cannot access inventory, purchases, reports, expenses, user management, configuration
   - Verify: Cannot void a sale (requires Pharmacist or Admin)
   - Verify: Can process a sale return and receive a debt payment

### Expected Outcome

- Admin: full access (22 permissions)
- Pharmacist: all except ManageUsers, SystemConfiguration, BackupRestore, ViewAuditLog, ConfigureRecommendations
- Cashier: CreateSale, AcceptPayment, ProcessSaleReturn, ReceiveDebtPayment only
- Restricted functions are hidden/disabled in UI AND enforced at service layer

---

## Validation Scenario 4: User Account Management (FR-016–FR-023)

**Goal**: Verify Admin can create, edit, deactivate, and reactivate user accounts with proper audit logging.

### Steps

1. Log in as Admin
2. Create a new user: username=`pharmacist1`, role=Pharmacist
3. Verify: User appears in the user list
4. Create another user with username=`PHARMACIST1` (uppercase)
5. Verify: System rejects with duplicate username error (case-insensitive, FR-022)
6. Change `pharmacist1`'s role to Cashier
7. Verify: Role is updated in the user list
8. Reset `pharmacist1`'s password
9. Log out, log in as `pharmacist1` with the new password
10. Verify: Login succeeds, interface shows Cashier-level access (role change from step 6 took effect at this login)
11. Log back in as Admin
12. Deactivate `pharmacist1`
13. Log out, attempt to log in as `pharmacist1`
14. Verify: Login fails with "account inactive" message
15. Log in as Admin, reactivate `pharmacist1`
16. Log out, log in as `pharmacist1`
17. Verify: Login succeeds again

### Last Admin Protection (FR-021)

18. Ensure only one active Admin exists
19. Attempt to deactivate that Admin
20. Verify: System prevents deactivation with explanatory message

### Expected Outcome

- Full CRUD lifecycle works (create, edit role, reset password, deactivate, reactivate)
- Case-insensitive username uniqueness enforced
- Deactivated accounts cannot log in
- Last Admin cannot be deactivated

---

## Validation Scenario 5: Audit Log (FR-024–FR-029)

**Goal**: Verify all sensitive actions are recorded and the audit log is queryable with filters.

### Steps

1. Perform the following actions as Admin:
   - Create a user (audit type: UserCreated)
   - Change a user's role (audit type: UserRoleChanged)
   - Reset a user's password (audit type: UserPasswordReset)
   - Deactivate a user (audit type: UserDeactivated)
   - Reactivate a user (audit type: UserReactivated)
2. Navigate to the Audit Log viewer
3. Verify: All 5 actions appear with correct action type, user (Admin), timestamp, and details
4. Verify: Each entry shows the affected entity and relevant details (e.g., old role → new role)
5. Filter by action type "UserCreated" → verify only creation entries appear
6. Filter by date range → verify only entries in range appear
7. Filter by user → verify only entries by that user appear
8. Log out, log in as Pharmacist
9. Attempt to access the Audit Log viewer
10. Verify: Access denied (FR-027)

### Append-Only Verification

11. Attempt to UPDATE an audit log entry via direct SQL (e.g., SQLite CLI)
12. Verify: SQLite trigger prevents the update with error message
13. Attempt to DELETE an audit log entry via direct SQL
14. Verify: SQLite trigger prevents the deletion

### Expected Outcome

- All user management actions generate audit entries
- Entries contain complete details (who, what, when, old/new values)
- Filtering works by action type, user, and date range
- Only Admin can view the audit log
- Database triggers prevent modification and deletion

---

## Validation Scenario 6: Password Security (FR-034, FR-035)

**Goal**: Verify passwords are hashed and never stored or displayed in plaintext.

### Steps

1. Create a user with a known password
2. Open the SQLite database file directly (e.g., with DB Browser for SQLite)
3. Inspect the Users table → PasswordHash column
4. Verify: Password is stored as `iterations:salt_base64:hash_base64` format (not plaintext)
5. Verify: Two users with the same password have different hashes (per-user salt, FR-034)
6. Check application log files (if any) for the password
7. Verify: Password does not appear in any log output (FR-035)

### Expected Outcome

- Passwords stored as PBKDF2 hashes with unique per-user salts
- Same password produces different hashes for different users
- No plaintext passwords anywhere in storage or logs

---

## Automated Test Coverage

Run the test suite to validate all acceptance scenarios programmatically:

```
dotnet test tests/PharmacyApp.Tests/ --filter "Category=Auth|Category=Permissions|Category=UserManagement|Category=AuditLog"
```

### Unit Tests

- `AuthServiceTests`: login success, login failure (wrong password, wrong username, inactive account), generic error messages, first-run bootstrap, forced password change
- `PermissionServiceTests`: each role × each permission (22 permissions × 3 roles = 66 assertions)
- `UserServiceTests`: create, change role, reset password, deactivate, reactivate, last-admin guard, duplicate username, optimistic concurrency
- `AuditLogServiceTests`: log entry creation, query with filters, pagination, admin-only access
- `PasswordHasherTests`: hash generation, hash verification, different salts per hash, invalid hash rejection

### Integration Tests

- `AuthIntegrationTests`: full login flow against SQLite database, first-run bootstrap with empty DB
- `UserManagementIntegrationTests`: user lifecycle (create → edit → deactivate → reactivate) with real database
- `AuditLogIntegrationTests`: action logging within transactions, rollback verification, append-only trigger enforcement
- `PermissionEnforcementTests`: service-layer permission checks for each restricted operation
