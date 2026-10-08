# Contract: IAuthService

**Source**: Spec FR-001–FR-007 (Authentication, First-Run Bootstrap)

The authentication service handles login, logout, session state, and first-run bootstrap. It is the entry point for the application — no other service is accessible until authentication succeeds.

---

## Interface

### Methods

**Login(username, password) → AuthResult**

Authenticates a user with the given credentials.

- Input: username (string), password (string)
- Returns: `AuthResult` containing success/failure, authenticated User (on success), or error message (on failure)
- Behavior:
  - Validates that username and password are non-empty (FR-001)
  - Looks up user by username (case-insensitive)
  - If user not found: returns generic failure message ("اسم المستخدم أو كلمة المرور غير صحيحة") (FR-003)
  - If user found but password hash doesn't match: returns same generic failure message (FR-003)
  - If user found and password matches but account is inactive: returns "هذا الحساب غير نشط" (distinct from credential error) (Spec AS 1.4)
  - If user found, password matches, and account is active: returns success with User object
  - Sets the current session's authenticated user and role
  - Checks `MustChangePassword` flag — if true, result indicates password change required (FR-007)

**Logout() → void**

Ends the current authenticated session.

- Clears the session state
- Returns the application to the login screen (FR-005)
- No audit log entry for logout (not a sensitive action per FR-024)

**GetCurrentUser() → User or null**

Returns the currently authenticated user, or null if no session is active.

- Used by services to determine who is performing an action
- Used by UI to display the current user's name and role

**EnsureDefaultAdmin() → void**

Creates the default Admin account if no users exist in the database.

- Called during application startup, before the login screen appears
- If `Users` table has 0 rows: inserts a default Admin with username "admin", hashed default password, `MustChangePassword = true` (FR-006, FR-007)
- If `Users` table has ≥ 1 row: does nothing (idempotent)

---

## AuthResult Structure

| Field | Type | Description |
|---|---|---|
| Success | bool | Whether authentication succeeded |
| User | User | The authenticated user (null on failure) |
| RequiresPasswordChange | bool | True if MustChangePassword flag is set |
| ErrorMessage | string | Arabic error message (null on success) |

---

## Dependencies

- `IUserRepository` — to look up users and check credentials
- `PasswordHasher` — to verify password against stored hash

## Consumed By

- `LoginForm` — calls Login, Logout, EnsureDefaultAdmin
- `ChangePasswordForm` — checks RequiresPasswordChange
- All services — call GetCurrentUser for audit log entries and permission checks
