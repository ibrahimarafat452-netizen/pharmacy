# Research: Authentication & Permissions

**Date**: 2026-10-08
**Feature**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md)

---

## Research Question 1: Password Hashing Algorithm

**Context**: Architecture Discovery Section 8 specifies "bcrypt or PBKDF2" for password storage. Need to determine which is appropriate for .NET Framework 4.8 on Windows 7 32-bit with no external NuGet dependency risk.

**Decision**: PBKDF2 via `System.Security.Cryptography.Rfc2898DeriveBytes`

**Rationale**:
- Built into .NET Framework 4.8 — zero external dependency
- 32-bit compatible by definition (ships with the runtime)
- Well-established security standard (NIST SP 800-132)
- No NuGet package to vet for 32-bit compatibility
- Performance is predictable and tunable via iteration count
- Produces a salted hash — meets FR-034 (per-user salt requirement)

**Alternatives Considered**:
- **bcrypt (BCrypt.Net-Next NuGet)**: Strong algorithm, widely used. Rejected because it adds an external NuGet dependency that must be verified for 32-bit compatibility (Constitution III). PBKDF2 achieves the same security goal with zero dependency risk.
- **Argon2 (Konscious.Security.Cryptography)**: Memory-hard algorithm, stronger than PBKDF2 against GPU attacks. Rejected because it requires .NET Standard 2.0+ NuGet package, adds memory overhead on weak hardware (Constitution III/XIV), and is overkill for a local-only system where the attacker would need physical access to the database file.

**Implementation Notes**:
- Use SHA-256 as the PRF (pseudo-random function) — `HashAlgorithmName.SHA256`
- Iteration count: 100,000 (OWASP 2023 recommendation for PBKDF2-SHA256). Can be reduced if performance testing on target hardware shows unacceptable delay (> 1 second per hash).
- Salt: 16 bytes, generated via `RNGCryptoServiceProvider`
- Output hash: 32 bytes
- Storage format: `iterations:salt_base64:hash_base64` (single string column)

---

## Research Question 2: Optimistic Concurrency for User Records

**Context**: Architecture Discovery Section 5 specifies "version column on entities" for optimistic concurrency control. Need to confirm the pattern for user account records.

**Decision**: `RowVersion INTEGER` column on the Users table, incremented on each UPDATE

**Rationale**:
- Consistent with the system-wide concurrency strategy (Architecture Discovery Section 5)
- Simple to implement with Dapper: include `RowVersion = @ExpectedVersion` in WHERE clause, check affected rows count
- SQLite handles INTEGER columns efficiently
- Prevents silent overwrites when two Admins edit the same user (edge case from spec)

**Alternatives Considered**:
- **Last-modified timestamp**: Less reliable (two updates in the same millisecond). Rejected.
- **SQLite built-in rowid**: Not a version counter — doesn't change on UPDATE. Rejected.

**Implementation Notes**:
- On UPDATE: `SET RowVersion = RowVersion + 1 WHERE Id = @Id AND RowVersion = @ExpectedVersion`
- If affected rows = 0: throw a concurrency conflict exception
- Service layer catches the exception and returns a user-friendly "record was modified by another user" message

---

## Research Question 3: Audit Log Append-Only Enforcement

**Context**: FR-026 requires audit log to be append-only. Need to determine how to enforce this in SQLite.

**Decision**: Dual enforcement — application layer (no UPDATE/DELETE methods on IAuditLogRepository) + SQLite triggers as defense-in-depth

**Rationale**:
- Primary enforcement: The `IAuditLogRepository` interface exposes only `Insert` and `Query` methods. No `Update` or `Delete` methods exist. This is the architectural guarantee.
- Defense-in-depth: SQLite BEFORE UPDATE and BEFORE DELETE triggers on the `AuditLog` table that raise errors. These catch any direct SQL access that bypasses the application layer.
- Combined approach provides both compile-time safety (interface design) and runtime safety (triggers)

**Alternatives Considered**:
- **Application layer only**: Sufficient but no defense against direct database manipulation. The trigger adds negligible overhead and significant safety.
- **Separate audit database file**: Adds deployment complexity for no meaningful security gain (anyone with file access can modify either file). Rejected per Constitution XIV (Simplicity).

**Implementation Notes**:
- SQLite trigger: `CREATE TRIGGER IF NOT EXISTS prevent_audit_update BEFORE UPDATE ON AuditLog BEGIN SELECT RAISE(ABORT, 'Audit log entries cannot be modified'); END;`
- SQLite trigger: `CREATE TRIGGER IF NOT EXISTS prevent_audit_delete BEFORE DELETE ON AuditLog BEGIN SELECT RAISE(ABORT, 'Audit log entries cannot be deleted'); END;`
- These triggers are created during database migration/initialization

---

## Research Question 4: Default Admin Account Bootstrap

**Context**: FR-006 and FR-007 require a default Admin account on first run with forced password change. Need to determine the approach.

**Decision**: Database migration checks for empty Users table and inserts a default Admin record

**Rationale**:
- Simple and deterministic: if `SELECT COUNT(*) FROM Users = 0`, insert default admin
- Password stored as a PBKDF2 hash of the default password (not plaintext in code)
- A `MustChangePassword` boolean column on the Users table flags the forced change requirement
- Login flow checks this flag and redirects to ChangePasswordForm before granting access

**Alternatives Considered**:
- **Installer creates the admin account**: Couples auth to installer logic. Rejected — the application should be self-bootstrapping.
- **First-run wizard**: Adds UI complexity for a one-time event. Rejected per Constitution XIV. A forced password change on the standard login flow is simpler.

**Implementation Notes**:
- Default username: `admin`
- Default password: `admin` (hashed with PBKDF2 before storage)
- `MustChangePassword` column: set to `true` for default admin, `false` for all other accounts
- After successful password change, `MustChangePassword` is set to `false`
- This flag is also useful if an Admin wants to force a user to change their password on next login (future enhancement, not in V1 scope)

---

## Research Question 5: Audit Log Query Performance at Scale

**Context**: SC-007 requires the audit log to remain queryable after 3 years at 500 daily transactions. Need to estimate volume and confirm SQLite handles it.

**Decision**: SQLite handles the projected volume comfortably with proper indexing

**Rationale**:
- 500 transactions/day × 365 days × 3 years = ~547,500 rows
- Not every transaction generates an audit entry — only sensitive actions (a subset). Realistic estimate: ~50-100 audit entries/day → ~55,000-110,000 rows over 3 years
- SQLite handles millions of rows efficiently with proper indexes
- Indexed columns: `Timestamp` (range queries), `ActionType` (filter), `UserId` (filter)
- A composite index on `(ActionType, Timestamp)` covers the most common query pattern

**Alternatives Considered**:
- **Partitioned audit tables by year**: Adds complexity for no meaningful benefit at this scale. Rejected.
- **Archival/export mechanism**: Out of V1 scope per spec. Could be added later if needed.

**Implementation Notes**:
- Index: `CREATE INDEX IX_AuditLog_Timestamp ON AuditLog(Timestamp DESC);`
- Index: `CREATE INDEX IX_AuditLog_ActionType_Timestamp ON AuditLog(ActionType, Timestamp DESC);`
- Index: `CREATE INDEX IX_AuditLog_UserId ON AuditLog(UserId);`

---

## Research Question 6: LAN Authentication — Pharmacy Secret

**Context**: FR-030 requires a shared pharmacy secret for LAN connection authentication before user login. Need to understand the authentication flow without inventing requirements beyond what the spec defines.

**Decision**: Pharmacy secret is a pre-shared key exchanged during LAN connection handshake, before the user login screen appears

**Rationale**:
- Architecture Discovery Section 8: "the LAN connection itself requires a shared pharmacy secret"
- V1 Spec FR-071: "LAN connection MUST require a shared pharmacy secret before user authentication"
- The pharmacy secret is a configuration value set by Admin (spec assumption: managed through system configuration module, not this module)
- It authenticates the *connection*, not the *user* — it proves the client belongs to this pharmacy's network
- Implementation deferred to LAN module; this module's scope is documenting the auth sequence

**Implementation Notes**:
- Auth flow on client PC: (1) Enter/load pharmacy secret → (2) TCP handshake with server → (3) Server validates secret → (4) Connection established → (5) User login screen appears → (6) Credentials sent to server for validation against shared DB
- Auth flow on server PC: (1) Application starts → (2) User login screen appears → (3) Credentials validated against local DB
- The pharmacy secret value is stored in local configuration (not in the database — the database is on the server, but the client needs the secret before connecting to it)
- Detailed LAN protocol design belongs in the LAN module plan, not here
