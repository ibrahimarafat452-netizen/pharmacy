# Implementation Plan: Authentication & Permissions

**Branch**: `002-auth-permissions` | **Date**: 2026-10-08 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/002-auth-permissions/spec.md`

## Summary

Implement the Authentication & Permissions module: user login/logout, three-role RBAC (Admin/Pharmacist/Cashier), user account management, an append-only audit log for sensitive actions, and LAN authentication sequence. This module is foundational — every other module depends on it for permission checks and audit logging. The technical approach uses .NET Framework 4.8 built-in password hashing (PBKDF2 via Rfc2898DeriveBytes), Dapper for data access against SQLite, and a service-layer permission enforcement pattern that other modules will consume.

## Technical Context

**Language/Version**: C# (.NET Framework 4.8)

**Primary Dependencies**: Windows Forms (UI), Dapper (data access), System.Data.SQLite (database driver)

**Storage**: SQLite with WAL mode (embedded). User accounts and audit log entries stored in the same database as all other pharmacy data.

**Testing**: NUnit + Moq

**Target Platform**: Windows 7 SP1+ (32-bit compatible), weak hardware (2GB RAM, HDD)

**Project Type**: Desktop application (WinForms)

**Performance Goals**: Login < 5 seconds (LAN), user management operations < 3 seconds, application startup (including first-run bootstrap) < 15 seconds

**Constraints**: Fully offline, no cloud/internet dependency, Arabic RTL UI, LAN for multi-PC (prototype-dependent), keyboard-first navigation

**Scale/Scope**: Up to 50 user accounts per pharmacy (practical ceiling), 500 daily transactions generating audit entries, audit log retention indefinite (never pruned)

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Gate | Status | Notes |
|---|---|---|---|
| I. Product First | Feature has clear business value | PASS | Security is non-negotiable for financial system |
| II. Offline-First | No cloud/internet dependency | PASS | All auth is local; no online authentication |
| III. Legacy Hardware | Win7/32-bit/2GB RAM compatible | PASS | Uses .NET Framework 4.8 built-in crypto; no heavy external deps |
| IV. Local Data Ownership | Data stays in pharmacy's local DB | PASS | Users and audit log in local SQLite |
| V. LAN Support | Multi-PC auth addressed | PASS | FR-030–033; prototype-dependent (Arch Discovery 10.1) |
| VI. Data Integrity | Atomic transactions | PASS | FR-029: audit entry in same transaction as action |
| X. Security & Permissions | Roles and audit log | PASS | This is the implementing module |
| XI. Backup & Recovery | No business data deletion | PASS | FR-023: users deactivated, never deleted; FR-026: audit append-only |
| XIII. Product Architecture | No pharmacy-specific code | PASS | Roles are config, not hard-coded per pharmacy |
| XIV. Simplicity | Simple UX, minimal clicks | PASS | Standard login, no MFA, no complex auth flows |
| XV. Testing & Definition of Done | Tests required | PASS | Unit + integration testing strategy defined |
| XVI. Agent Governance | Following workflow | PASS | Spec → Plan → Tasks → Implementation |
| XVII. Scope Control | No scope creep | PASS | Excluded: session timeout, lockout, custom roles, self-service reset |

**Result**: All gates pass. No violations. No justifications needed.

## Project Structure

### Documentation (this feature)

```text
specs/002-auth-permissions/
├── plan.md              # This file
├── research.md          # Phase 0 output — technology decisions
├── data-model.md        # Phase 1 output — entity schemas
├── contracts/           # Phase 1 output — service interfaces
│   ├── auth-service.md
│   ├── permission-service.md
│   ├── user-service.md
│   └── audit-log-service.md
├── quickstart.md        # Phase 1 output — validation guide
└── tasks.md             # Phase 2 output (/speckit-tasks — NOT created by /speckit-plan)
```

### Source Code (repository root)

```text
src/
  PharmacyApp/
    Models/
      User.cs                    # User entity
      AuditLogEntry.cs           # Audit log entry entity
      Role.cs                    # Role enum (Admin, Pharmacist, Cashier)
      Permission.cs              # Permission enum
    Services/
      IAuthService.cs            # Authentication contract
      AuthService.cs             # Login, logout, session
      IUserService.cs            # User management contract
      UserService.cs             # CRUD, password reset, deactivation
      IPermissionService.cs      # Permission checking contract
      PermissionService.cs       # Role → permissions mapping, enforcement
      IAuditLogService.cs        # Audit logging contract
      AuditLogService.cs         # Append-only logging, querying
    Repositories/
      IUserRepository.cs         # User data access contract
      UserRepository.cs          # Dapper + SQLite implementation
      IAuditLogRepository.cs     # Audit log data access contract
      AuditLogRepository.cs      # Dapper + SQLite, no update/delete
    Forms/
      LoginForm.cs               # Login screen (first screen on startup)
      ChangePasswordForm.cs      # Forced password change on first login
      UserManagementForm.cs      # Admin: user CRUD
      AuditLogForm.cs            # Admin: audit log viewer with filters
    Security/
      PasswordHasher.cs          # PBKDF2 hashing via Rfc2898DeriveBytes
    Infrastructure/
      DatabaseMigrations.cs      # Schema creation (Users, AuditLog tables)
  PharmacyApp.sln

tests/
  PharmacyApp.Tests/
    Unit/
      Services/
        AuthServiceTests.cs
        UserServiceTests.cs
        PermissionServiceTests.cs
        AuditLogServiceTests.cs
      Security/
        PasswordHasherTests.cs
    Integration/
      AuthIntegrationTests.cs
      UserManagementIntegrationTests.cs
      AuditLogIntegrationTests.cs
      PermissionEnforcementTests.cs
```

**Structure Decision**: Standard layered architecture — Models (entities), Services (business logic with permission enforcement), Repositories (data access via Dapper), Forms (WinForms UI), Security (cross-cutting crypto). Permission checks enforced at the Service layer per FR-014. Repository pattern aligns with Architecture Discovery Section 4 (two implementations: `SqliteRepository` for single-PC/server, `NetworkRepository` for LAN clients).

## Dependencies & Risks

### Dependencies

| Dependency | Type | Impact on Auth Module |
|---|---|---|
| SQLite database + Dapper setup | Infrastructure | Auth tables require an initialized database and migration runner |
| WinForms application shell | Infrastructure | Login form requires the main application window to host it |
| LAN prototype validation (Arch 10.1) | Prototype | LAN auth (FR-030–033) blocked until prototype passes |
| Repository pattern (single/network) | Architecture | Auth repository must implement both SqliteRepository and NetworkRepository interfaces when LAN is added |

### Risks

| Risk | Severity | Mitigation |
|---|---|---|
| Auth module is first to be implemented — no existing project scaffolding | Low | Module plan includes initial project setup as prerequisite |
| LAN prototype may fail, changing auth architecture | Medium | LAN auth is isolated (FR-030–033); single-PC auth works independently |
| Audit log table growth over years | Low | SQLite handles millions of rows; indexed queries on action_type, user_id, timestamp |
| Password hashing performance on very old hardware | Low | PBKDF2 iteration count can be tuned; default .NET settings are reasonable |

## Complexity Tracking

No Constitution violations. No justifications needed. Table intentionally empty.
