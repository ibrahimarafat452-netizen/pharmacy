# Feature Specification: Authentication & Permissions

**Feature Branch**: `002-auth-permissions`

**Created**: 2026-10-08

**Status**: Draft

**Input**: User description: "Authentication & Permissions module — detailed specification based on existing V1 spec (User Story 5, FR-031 through FR-036), Constitution Principle X, and Architecture Discovery Section 8."

**Constitution Reference**: v1.0.0 (Ratified 2026-10-07) — Principle X (Security & Permissions)

**Architecture Reference**: Architecture Discovery Section 8 (Security & Permissions)

**Parent Specification**: specs/001-pharmacy-management-v1/spec.md — User Story 5, FR-031 through FR-036, SC-011, SC-012

---

## User Scenarios & Testing

### User Story 1 — Login & Logout (Priority: P1)

A user launches the application and is presented with a login screen. They enter their username and password. If the credentials are valid and the account is active, the system authenticates them and shows the interface appropriate to their role. When the user is done, they log out, returning to the login screen so another user can log in on the same machine.

**Why this priority**: Nothing in the system is accessible without authentication. This is the gateway to every other feature.

**Independent Test**: Can be tested by creating a user account, launching the application, entering valid and invalid credentials, verifying access is granted or denied, and logging out.

**Acceptance Scenarios**:

1. **Given** the application starts, **When** a user enters a valid username and correct password for an active account, **Then** the system authenticates them and displays the main interface appropriate to their role.
2. **Given** the login screen, **When** a user enters an invalid username, **Then** the system denies access and displays a generic error message that does not reveal whether the username or password was incorrect.
3. **Given** the login screen, **When** a user enters a valid username but incorrect password, **Then** the system denies access and displays the same generic error message (indistinguishable from an invalid username).
4. **Given** the login screen, **When** a user enters credentials for a deactivated account, **Then** the system denies access and displays a message indicating the account is inactive.
5. **Given** an authenticated session, **When** the user selects logout, **Then** the system ends the session and returns to the login screen, ready for another user.
6. **Given** the application is freshly installed with no user accounts in the database, **When** the application starts for the first time, **Then** the system creates a default Admin account and presents the login screen.
7. **Given** the default Admin account exists, **When** the Admin logs in with the default credentials, **Then** the system authenticates them and prompts them to change the default password before proceeding.

---

### User Story 2 — Role-Based Permission Enforcement (Priority: P2)

The system enforces access control based on the user's assigned role (Admin, Pharmacist, or Cashier). Each role has a defined set of permitted operations. Permission checks are enforced at the business logic layer, not only at the UI layer, so that hidden or direct access attempts are also blocked.

**Why this priority**: Without permission enforcement, authentication alone provides no security. Roles and permissions are what prevent unauthorized access to sensitive operations.

**Independent Test**: Can be tested by logging in as each role, attempting all system operations, and verifying that permitted actions succeed and restricted actions are denied with an appropriate message.

**Acceptance Scenarios**:

1. **Given** a user logged in as Cashier, **When** they attempt to access inventory management, purchase management, user administration, reports, expense tracking, or system configuration, **Then** the system denies access and displays a permission denied message.
2. **Given** a user logged in as Cashier, **When** they create a sale, accept a payment, process a sale return, or receive a customer debt payment, **Then** the system allows these actions.
3. **Given** a user logged in as Pharmacist, **When** they perform a sale, view or manage inventory, record a purchase or purchase return, manage products, perform a stock adjustment, conduct stocktaking, generate a report, record an expense, or view AI recommendations, **Then** the system allows these actions.
4. **Given** a user logged in as Pharmacist, **When** they attempt to access user management, system configuration, backup/restore, or the audit log, **Then** the system denies access.
5. **Given** a user logged in as Admin, **When** they attempt any operation in the system, **Then** the system allows the action (full access).
6. **Given** any role, **When** a permission-restricted action is attempted through any means (UI navigation, keyboard shortcut, or direct service call), **Then** the business logic layer enforces the permission check regardless of how the request was initiated.
7. **Given** a user logged in as Cashier, **When** they attempt to void a sale, **Then** the system denies the action (void sale requires Pharmacist or Admin).

---

### User Story 3 — User Account Management (Priority: P3)

An Admin manages user accounts for the pharmacy. They can create new accounts with an assigned role, edit existing accounts (change role, reset password), and deactivate accounts that are no longer needed. Deactivation preserves the user's audit history while preventing future login.

**Why this priority**: User management is required to set up and maintain the workforce that uses the system. Without it, only the default Admin account exists.

**Independent Test**: Can be tested by logging in as Admin, creating users with each role, editing their roles, resetting their passwords, deactivating an account, and verifying the deactivated account cannot log in.

**Acceptance Scenarios**:

1. **Given** an Admin is logged in, **When** they create a new user account with username, initial password, and role (Admin, Pharmacist, or Cashier), **Then** the account is created and the action is recorded in the audit log.
2. **Given** an Admin is logged in, **When** they change an existing user's role, **Then** the role is updated, the change is recorded in the audit log, and the new permissions take effect at the user's next login.
3. **Given** an Admin is logged in, **When** they reset a user's password, **Then** the password is updated (stored as a one-way hash), the action is recorded in the audit log, and the user must use the new password at their next login.
4. **Given** an Admin is logged in, **When** they deactivate a user account, **Then** the account is marked inactive, the action is recorded in the audit log, and the user can no longer log in.
5. **Given** only one active Admin account exists in the system, **When** an Admin attempts to deactivate that account, **Then** the system prevents the deactivation and displays a message explaining that at least one active Admin must exist.
6. **Given** a deactivated user account, **When** an Admin reactivates it, **Then** the account becomes active again and the user can log in with their existing credentials.
7. **Given** any user account operation (create, edit role, reset password, deactivate, reactivate), **When** the operation completes, **Then** the audit log records the action type, the Admin who performed it, the affected user, the timestamp, and relevant details (e.g., old role → new role).
8. **Given** an Admin creates a new user, **When** they assign a username, **Then** the system enforces that the username is unique (case-insensitive).

---

### User Story 4 — Audit Log for Sensitive Actions (Priority: P4)

The system records every sensitive action in an audit log that captures who did what, when, and the relevant details. The audit log is append-only and viewable only by Admin users. It provides accountability and traceability for operations that affect financial data, inventory, user accounts, or system configuration.

**Why this priority**: Accountability is a foundational requirement for a system handling financial transactions. The audit log is referenced by every other module that performs sensitive actions.

**Independent Test**: Can be tested by performing each type of sensitive action, then logging in as Admin and verifying that the audit log contains accurate, complete entries for each action.

**Acceptance Scenarios**:

1. **Given** a Pharmacist or Admin voids a completed sale, **When** the void is processed, **Then** the audit log records: action type (void sale), user who authorized it, timestamp, sale reference, sale amount, and reason.
2. **Given** a Pharmacist performs a stock adjustment, **When** the adjustment is saved, **Then** the audit log records: action type (stock adjustment), user, timestamp, product, batch, old quantity, new quantity, and reason.
3. **Given** a Pharmacist or Admin changes a product's selling price, **When** the change is saved, **Then** the audit log records: action type (price change), user, timestamp, product, old price, and new price.
4. **Given** an Admin performs a user management action (create, edit, deactivate, reactivate, password reset), **When** the action completes, **Then** the audit log records the action type, the Admin, the affected user, timestamp, and relevant details.
5. **Given** a Pharmacist or Admin processes a purchase return, **When** the return is recorded, **Then** the audit log records: action type (purchase return), user, timestamp, supplier, products and quantities returned, and financial impact.
6. **Given** an Admin changes a system configuration setting, **When** the change is saved, **Then** the audit log records: action type (configuration change), user, timestamp, setting name, old value, and new value.
7. **Given** an Admin navigates to the audit log, **When** the log is displayed, **Then** all entries are shown in reverse chronological order with filtering options for action type, user, and date range.
8. **Given** any audit log entry, **When** it is created, **Then** it cannot be modified or deleted by any user (append-only).

---

### User Story 5 — LAN Authentication (Priority: P5)

In multi-PC mode (LAN), the application on a client PC first establishes a network connection to the server PC using a shared pharmacy secret. Only after the LAN connection is authenticated does the user login screen appear, authenticating the user against the shared database on the server.

**Why this priority**: LAN authentication is essential for multi-PC pharmacies but depends on the LAN feature (User Story 11 in the V1 spec) and its successful prototype validation. It does not block single-PC operation.

**Independent Test**: Can be tested by configuring two PCs on a LAN, entering the correct and incorrect pharmacy secret, and verifying that user authentication only proceeds after a successful LAN connection.

**Acceptance Scenarios**:

1. **Given** a client PC configured for LAN mode, **When** the application starts, **Then** it first prompts for (or uses the configured) shared pharmacy secret to establish a connection to the server PC before showing the user login screen.
2. **Given** a client PC provides the correct pharmacy secret, **When** the LAN connection is established, **Then** the user login screen appears and authentication occurs against the shared database on the server.
3. **Given** a client PC provides an incorrect pharmacy secret, **When** the connection attempt fails, **Then** the system displays a connection error and does not show the user login screen.
4. **Given** a client PC in LAN mode, **When** the server PC is unreachable (powered off, network disconnected), **Then** the system displays a server unreachable message and does not allow user authentication.
5. **Given** a user authenticated on a client PC, **When** the LAN connection drops, **Then** the system notifies the user that the server is unreachable and blocks any transactions until the connection is restored.
6. **Given** the server PC in LAN mode, **When** a user logs in directly on the server PC, **Then** authentication uses the local shared database directly (no network hop).

---

### Edge Cases

- What happens when the application is installed for the first time with an empty database? The system MUST create a default Admin account and require a password change on first login.
- What happens when the only active Admin account is about to be deactivated? The system MUST prevent the deactivation and display an explanatory message.
- What happens when an Admin resets their own password? The system MUST allow it and record the action in the audit log. The new password takes effect immediately (no re-login required for the active session).
- What happens when a user's role is changed while they are logged in on another PC (LAN)? The new role takes effect at the user's next login. The current session retains the original role's permissions until logout.
- What happens when two Admins attempt to edit the same user account simultaneously (LAN)? Optimistic concurrency control MUST detect the conflict and prevent a silent overwrite, consistent with the system's concurrency strategy (V1 spec FR-069).
- What happens when a user enters an empty username or empty password? The system MUST reject the login attempt with a validation message and not attempt authentication.
- What happens when the audit log grows very large over years of operation? The audit log MUST remain queryable. Filtering by date range, action type, and user MUST remain functional. The audit log is never pruned or deleted (Constitution Principle XI — business data is never automatically deleted).
- What happens when a Pharmacist is downgraded to Cashier while they have open workflows (e.g., a stocktake in progress)? The role change takes effect at next login. Any open workflow that requires Pharmacist permissions will be denied when resumed under the Cashier role.
- What happens during a power failure while the audit log entry is being written as part of a sensitive action? The entire operation (including the audit log entry) is within a single atomic transaction. Either both the action and its audit entry commit, or neither does (Constitution Principle VI — data integrity).

---

## Requirements

### Functional Requirements

**Authentication**

- **FR-001**: System MUST require user authentication (username and password) before granting access to any system function.
- **FR-002**: System MUST store all passwords using a strong one-way hash. Plaintext passwords MUST NOT be stored or logged anywhere in the system.
- **FR-003**: System MUST NOT reveal whether a failed login was due to an incorrect username or an incorrect password. The error message MUST be generic and identical in both cases.
- **FR-004**: System MUST display the interface appropriate to the authenticated user's role after successful login.
- **FR-005**: System MUST provide a logout function that ends the current session and returns to the login screen.

**First-Run Bootstrap**

- **FR-006**: System MUST create a default Admin account on first run when no user accounts exist in the database.
- **FR-007**: System MUST require the default Admin to change the default password on first login before accessing any other function.

**Role Definitions**

- **FR-008**: System MUST support exactly three roles: Admin, Pharmacist, and Cashier.
- **FR-009**: Admin role MUST have full access to all system functions.
- **FR-010**: Pharmacist role MUST have access to: sales, sale returns, void sales, payments (customer and supplier), inventory management, stock adjustments, stocktaking, purchase management, purchase returns, product management (including price changes), customer and supplier management, expense recording, report generation, and AI recommendation viewing.
- **FR-011**: Cashier role MUST have access to: creating sales, accepting payments, processing sale returns, and receiving customer debt payments. All other functions MUST be denied.
- **FR-012**: Role changes MUST take effect at the affected user's next login, not during an active session.

**Permissions Matrix**

- **FR-013**: System MUST enforce the following permissions matrix:

| Capability | Admin | Pharmacist | Cashier |
|---|---|---|---|
| Login / Logout | Yes | Yes | Yes |
| Create sale | Yes | Yes | Yes |
| Accept payment (sale) | Yes | Yes | Yes |
| Process sale return | Yes | Yes | Yes |
| Receive customer debt payment | Yes | Yes | Yes |
| Void completed sale | Yes | Yes | No |
| View / manage inventory | Yes | Yes | No |
| Stock adjustment | Yes | Yes | No |
| Conduct stocktaking | Yes | Yes | No |
| Create / manage purchases | Yes | Yes | No |
| Process purchase return | Yes | Yes | No |
| Manage products (incl. price change) | Yes | Yes | No |
| Manage customer records | Yes | Yes | No |
| Manage supplier records | Yes | Yes | No |
| Record expenses | Yes | Yes | No |
| Generate reports | Yes | Yes | No |
| View AI recommendations | Yes | Yes | No |
| Configure recommendation thresholds | Yes | No | No |
| User management (create, edit, deactivate) | Yes | No | No |
| System configuration | Yes | No | No |
| Backup and restore | Yes | No | No |
| View audit log | Yes | No | No |

**Permission Enforcement**

- **FR-014**: System MUST enforce permission checks at the business logic layer (service layer), not only at the UI layer.
- **FR-015**: UI elements for restricted functions MUST be hidden or visually disabled for unauthorized roles, but enforcement MUST NOT rely solely on UI hiding.

**User Management**

- **FR-016**: Admin MUST be able to create new user accounts with: username, initial password, and assigned role.
- **FR-017**: Admin MUST be able to change a user's assigned role.
- **FR-018**: Admin MUST be able to reset a user's password.
- **FR-019**: Admin MUST be able to deactivate a user account, preventing future login while preserving all historical data and audit trail entries associated with that user.
- **FR-020**: Admin MUST be able to reactivate a previously deactivated user account.
- **FR-021**: System MUST prevent deactivating the last remaining active Admin account.
- **FR-022**: Usernames MUST be unique (case-insensitive). The system MUST reject duplicate usernames at creation time.
- **FR-023**: User accounts MUST NOT be permanently deleted. Deactivation is the only mechanism for disabling access (preserves referential integrity with audit log and transaction history).

**Audit Log**

- **FR-024**: System MUST record the following sensitive actions in the audit log: void sale, stock adjustment, price change, user management (create, edit role, reset password, deactivate, reactivate), purchase return, and configuration changes.
- **FR-025**: Each audit log entry MUST include: action type, user who performed the action, timestamp, entity affected, and relevant details (including old and new values where applicable).
- **FR-026**: Audit log MUST be append-only. No user, including Admin, may modify or delete audit log entries.
- **FR-027**: Audit log MUST be viewable only by Admin role.
- **FR-028**: Audit log MUST support filtering by: action type, user, and date range.
- **FR-029**: The audit log entry for a sensitive action MUST be recorded within the same atomic transaction as the action itself. If the transaction fails, neither the action nor the audit entry is persisted.

**LAN Authentication**

- **FR-030**: In LAN (multi-PC) mode, the client application MUST authenticate the LAN connection using a shared pharmacy secret before presenting the user login screen.
- **FR-031**: User authentication in LAN mode MUST occur against the shared database on the server PC.
- **FR-032**: If the LAN connection cannot be established (server unreachable or incorrect pharmacy secret), the system MUST NOT allow user authentication.
- **FR-033**: On the server PC itself, authentication MUST use the local database directly without a network connection.

**Password Security**

- **FR-034**: Passwords MUST be stored using a strong one-way hash with a per-user salt. Plaintext, reversible encryption, and unsalted hashes are prohibited.
- **FR-035**: The system MUST NOT log, display, or transmit passwords in plaintext at any point during authentication or user management operations.

### Key Entities

- **User**: A system operator account. Has username (unique, case-insensitive), hashed password, role (Admin / Pharmacist / Cashier), active status (active / inactive), created date, and last modified date.
- **Audit Log Entry**: A record of a sensitive action. Has entry ID, action type (void sale / stock adjustment / price change / user management / purchase return / configuration change), user who performed the action, timestamp, entity type affected, entity ID affected, old value, new value, and details (free-text description). Append-only — cannot be modified or deleted.
- **Pharmacy Secret** (LAN only): A shared credential used to authenticate the LAN connection between client and server PCs. Configured by Admin. Not a user-level credential.

---

## Success Criteria

### Measurable Outcomes

- **SC-001**: Unauthorized users cannot access functions outside their role — 100% of permission-restricted actions are enforced at the business logic layer, not only at the UI.
- **SC-002**: Audit log captures 100% of defined sensitive actions with complete details (action type, user, timestamp, entity affected, old/new values).
- **SC-003**: A failed login attempt reveals no information about which credential (username or password) was incorrect — error messages are identical and indistinguishable.
- **SC-004**: On first installation, the system is usable within 2 minutes: default Admin account is created automatically, the Admin changes the default password, and can immediately create additional user accounts.
- **SC-005**: User account operations (create, edit role, reset password, deactivate, reactivate) complete in under 3 seconds and are reflected in the audit log immediately.
- **SC-006**: In LAN mode, user authentication against the server database completes in under 5 seconds on a local network.
- **SC-007**: The audit log remains queryable with filtering (by action type, user, and date range) even after 3 years of continuous operation at the target transaction volume (500 daily transactions).
- **SC-008**: No password is stored, logged, or transmitted in plaintext at any point in the system — verifiable through code review and data inspection.

---

## Assumptions

- **Roles are fixed at three**: Admin, Pharmacist, and Cashier. Custom roles or granular permission editing are out of scope for V1.
- **No self-service password reset**: Password resets are Admin-initiated only. There is no email or SMS recovery mechanism (consistent with offline-first architecture).
- **No session timeout**: The user remains logged in until they explicitly log out or the application is closed. Session timeout is out of scope for V1.
- **No account lockout**: Failed login attempts are not counted and do not trigger account lockout. This is acceptable for a LAN-only system with physical access control (pharmacy premises). Account lockout could be added in a future version.
- **Single active session per user is not enforced**: In LAN mode, the same user account may be logged in on multiple PCs simultaneously. Each session operates independently with the role's permissions.
- **Default Admin credentials**: The default Admin account uses a well-known username (e.g., "admin") and a well-known initial password. The system forces a password change on first login. The specific default credentials are an implementation decision.
- **Password minimum length**: A minimum password length is assumed as a basic security measure. The specific minimum (e.g., 6 or 8 characters) is an implementation decision.
- **LAN authentication is prototype-dependent**: The LAN authentication flow (User Story 5, FR-030 through FR-033) depends on the successful validation of the LAN prototype (Architecture Discovery Section 10.1). If the LAN prototype fails and the architecture falls back to MariaDB, the LAN authentication mechanism will adapt accordingly.
- **Audit log storage**: The audit log is stored in the same database as other application data. It is included in backups and restores. It is never pruned or deleted.
- **Pharmacy secret management**: The shared pharmacy secret for LAN mode is configured by the Admin through system configuration. The mechanism for setting and changing the pharmacy secret is part of system configuration, not this module.
