# Implementation Plan: Pharmacy Management System V1

**Branch**: `001-pharmacy-management-v1` | **Date**: 2026-10-08 | **Spec**: [spec.md](spec.md)

**Input**: Full-system specification from `specs/001-pharmacy-management-v1/spec.md`

**Constitution Reference**: v1.0.0 (Ratified 2026-10-07)

**Architecture Reference**: Architecture Discovery (Revised 2026-10-08)

**Module Specifications**: `specs/002-auth-permissions/` (detailed spec + plan complete)

---

## Summary

Full-system implementation plan for the Pharmacy Management System V1: a C# WinForms desktop application with SQLite (WAL mode) for Egyptian pharmacies. The system covers 13 user stories (POS, Inventory, Purchasing, Customer Debts, Auth, Backup, AI Recommendations, Expenses, Reports, Stocktaking, LAN, Licensing, Configuration) organized into 12 implementation phases based on dependency order. Foundational modules (infrastructure, auth, inventory) are built first; data-consuming modules (reports, AI engine) come after their data sources exist; LAN multi-PC is last due to cross-cutting integration and prototype dependency.

---

## Technical Context

**Language/Version**: C# (.NET Framework 4.8)

**Primary Dependencies**: Windows Forms (UI), Dapper (data access), System.Data.SQLite (database driver), NUnit + Moq (testing)

**Storage**: SQLite with WAL mode (embedded). Single database file per pharmacy containing all entities.

**Testing**: NUnit + Moq. Unit tests for services/logic, integration tests against SQLite for repositories and cross-module flows.

**Target Platform**: Windows 7 SP1+ (32-bit compatible), weak hardware (2GB RAM, HDD)

**Project Type**: Desktop application (WinForms)

**Performance Goals**: Application startup < 15 seconds on target hardware. Sale completion (3 items + payment + receipt) < 60 seconds keyboard-only. 5,000 products and 500 daily transactions without noticeable delay.

**Constraints**: Fully offline (no cloud/internet dependency), Arabic-only UI with RTL layout, Egyptian Pound (EGP) only, keyboard-first navigation, LAN for multi-PC (prototype-dependent)

**Scale/Scope**: Small/medium Egyptian pharmacies, 1–4 PCs, up to 5,000 products, 500 daily transactions, indefinite data retention

---

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| # | Principle | Gate | Status | Notes |
|---|-----------|------|--------|-------|
| I | Product First | Every feature has clear business value | PASS | All 13 user stories trace to pharmacy operations |
| II | Offline-First | No cloud/internet dependency | PASS | All operations local; LAN is local network only |
| III | Legacy Hardware | Win7/32-bit/2GB RAM compatible | PASS | .NET Framework 4.8 + SQLite; no heavy dependencies |
| IV | Local Data Ownership | Data stays in pharmacy's local DB | PASS | Single SQLite file per pharmacy; no cross-pharmacy data |
| V | LAN Support | Multi-PC addressed | PASS | Phase 12 + Prototype 10.1; fallback to MariaDB documented |
| VI | Data Integrity | Atomic transactions | PASS | All financial/inventory operations in single DB transaction |
| VII | Inventory Accuracy | Batch/FEFO/units/stocktaking/audit | PASS | Phase 3 (Inventory) + Phase 7 (Stocktaking) |
| VIII | Financial Accuracy | Sales/purchases/payments/debts/expenses | PASS | Phases 4–7 cover all financial operations |
| IX | Explainable AI | Score/Priority/Reason/Action | PASS | Phase 9 (AI Engine); no external AI dependency |
| X | Security & Permissions | Roles and audit log | PASS | Phase 2 (Auth module — already planned in detail) |
| XI | Backup & Recovery | No business data deletion | PASS | Phase 10; audit log append-only; users deactivated not deleted |
| XII | Licensing | Per-pharmacy license | PASS | Phase 11 + Prototype 10.3 |
| XIII | Product Architecture | One Core Product, config not code | PASS | Phase 10 (Configuration); pharmacy identity in config |
| XIV | Simplicity & Performance | Fast/keyboard/Arabic/simple | PASS | WinForms keyboard-first; performance targets defined |
| XV | Testing & DoD | Tests required per feature | PASS | NUnit + Moq; each phase includes test requirements |
| XVI | Agent Governance | Spec → Plan → Tasks → Impl | PASS | This plan follows the mandated workflow |
| XVII | Scope Control | No scope creep | PASS | All features from approved V1 spec only |
| XVIII | Learning Principle | Decisions include rationale | PASS | Research.md documents all decisions with alternatives |

**Result**: All 18 gates pass. No violations. No justifications needed.

---

## Required Prototypes

These prototypes MUST be validated before the phases that depend on them. Results may alter architecture decisions.

| ID | Prototype | Required Before | Fallback if Fails | Arch Ref |
|----|-----------|----------------|-------------------|----------|
| P-10.4 | WinForms Arabic RTL on Win7 32-bit | Phase 1 (Foundation) | Third-party UI controls | 10.4 |
| P-10.5 | SQLite Crash Recovery Verification | Phase 1 (Foundation) | Reconsider database choice | 10.5 |
| P-10.2 | Arabic Thermal Printing | Phase 5 (POS) | Windows driver only (reduced control) | 10.2 |
| P-10.3 | Hardware Fingerprint | Phase 11 (Licensing) | Simplified fingerprint or alternative binding | 10.3 |
| P-10.1 | LAN Multi-PC | Phase 12 (LAN) | MariaDB as database (eliminates custom networking) | 10.1 |

---

## Implementation Phases

### Phase 1: Foundation & Infrastructure

**Dependencies**: Prototypes P-10.4 (RTL) and P-10.5 (Crash Recovery) validated

**Scope**: Project scaffolding and shared infrastructure that all modules depend on.

| Item | Description | FR Coverage |
|------|-------------|-------------|
| Solution structure | `PharmacyApp.sln`, project layout (Models, Services, Repositories, Forms, Infrastructure) | — |
| SQLite setup | Database creation, WAL mode configuration, connection management | — |
| Migration framework | Schema versioning, table creation on first run, upgrade path | — |
| Base repository pattern | `IRepository<T>` → `SqliteRepository<T>` via Dapper; prepared for future `NetworkRepository<T>` | ADR-003 |
| Service layer pattern | Base service with permission check hook and audit log integration point | FR-014, FR-082 |
| Transaction management | `IUnitOfWork` wrapping SQLite transactions; atomic commit/rollback | FR-082 |
| Application shell | Main form, RTL layout, Arabic font configuration, navigation framework | FR-081 |
| Keyboard navigation | Tab order management, shortcut key framework, focus management | FR-010 |

**Output**: Compilable solution with empty main form, database initialization, migration runner, and base patterns.

---

### Phase 2: Authentication & Permissions

**Dependencies**: Phase 1

**Module Spec**: `specs/002-auth-permissions/spec.md` (complete)

**Module Plan**: `specs/002-auth-permissions/plan.md` (complete, with research + data-model + contracts + quickstart)

**Scope**: User Story 5 — FR-031 through FR-036.

| Item | Description | FR Coverage |
|------|-------------|-------------|
| Login / Logout | Username + hashed password authentication, session management | FR-031 |
| Password hashing | PBKDF2 via Rfc2898DeriveBytes (100K iterations, SHA-256, 16-byte salt) | FR-032 |
| Role-based permissions | Admin / Pharmacist / Cashier with permissions matrix | FR-033 |
| Service-layer enforcement | Permission checks in business logic, not just UI | FR-034 |
| Audit log | Append-only log for sensitive actions, same transaction as action | FR-035, FR-036 |
| User management | Admin creates/edits/deactivates users; no permanent deletion | FR-033 |
| First-run bootstrap | Default Admin account with forced password change | — |

**Output**: Working login, role enforcement across all service calls, audit logging, user management UI.

**Note**: This module is fully specified and planned. Refer to `specs/002-auth-permissions/` for detailed research, data model, service contracts, and quickstart.

---

### Phase 3: Product Catalog & Inventory Core

**Dependencies**: Phase 2 (auth + audit log)

**Scope**: User Story 2 — FR-011 through FR-018, plus product/category foundation for all other modules.

| Item | Description | FR Coverage |
|------|-------------|-------------|
| Product entity | Name (Arabic), barcode, category, unit type (box/strip), strips-per-box ratio, selling price, status | FR-011 |
| Category management | Hybrid list: system defaults + Admin add/edit/remove | FR-011 |
| Batch tracking | Batch number, quantity, purchase price, expiry date, supplier reference; linked to Product | FR-012 |
| Unit conversion | Box ↔ strip conversion with strips-per-box ratio; fractional box display | FR-015 |
| FEFO enforcement | Automatic selection of earliest-expiry batch for all deductions | FR-013 |
| Expiry flagging | Expired batches flagged and excluded from sale operations | FR-016 |
| Negative stock prevention | Stock MUST NOT go below zero; validated in service layer | FR-014 |
| Stock adjustment | Manual adjustment with mandatory reason; old/new qty recorded | FR-017 |
| Inventory audit trail | Every inventory change (sale, purchase, adjustment, return, stocktaking) logged | FR-018 |
| Spreadsheet import | Basic import for initial catalog setup/migration | FR-011a |
| Product search | Partial Arabic name match for product lookup | FR-003 |

**Output**: Product CRUD, category management, batch management, stock operations with FEFO, audit trail, import utility.

---

### Phase 4: Purchase Management & Suppliers

**Dependencies**: Phase 3 (products + inventory)

**Scope**: User Story 3 — FR-019 through FR-024.

| Item | Description | FR Coverage |
|------|-------------|-------------|
| Supplier records | Name, phone, address, notes | FR-024 |
| Purchase recording | Supplier, date, line items (product, qty, unit cost, batch number, expiry), total | FR-019 |
| Goods receipt | Purchase line items → create/update inventory batches with received quantities + expiry dates | FR-020 |
| Barcode product creation | New products auto-created when entered through a purchase via barcode (primary workflow) | FR-011a |
| Supplier balance | Total purchases minus payments minus returns = outstanding balance | FR-021 |
| Supplier payments | Partial payments with full payment history | FR-022 |
| Purchase returns | Returned qty removed from inventory, supplier balance reduced, audit log entry | FR-023 |
| Supplier account view | All purchases, payments, returns, current balance per supplier | FR-021 |

**Output**: Supplier CRUD, purchase recording with batch creation, supplier balance management, purchase returns.

---

### Phase 5: Point of Sale

**Dependencies**: Phase 3 (inventory + FEFO), Phase 2 (auth). Prototype P-10.2 (Arabic printing) must be validated.

**Scope**: User Story 1 — FR-001 through FR-010.

| Item | Description | FR Coverage |
|------|-------------|-------------|
| Sale creation | One or more line items with product, quantity, unit price | FR-001 |
| Barcode input | Scanner input adds product to sale with quantity 1 | FR-002 |
| Arabic name search | Partial match search for products without barcode | FR-003 |
| FEFO deduction | Automatic earliest-expiry batch selection on sale | FR-004 |
| Stock validation | Prevent selling zero-stock or all-expired products | FR-005 |
| Real-time totals | Line totals, sale total, change due update immediately | FR-006 |
| Payment processing | Full cash + partial payment (credit → customer debt) | FR-007 |
| Receipt printing | Thermal receipt (58mm/80mm): pharmacy identity, items, totals, date/time, cashier | FR-008 |
| Void sale | Pharmacist/Admin only; reverses inventory + financial entries; audit logged | FR-009 |
| Keyboard navigation | All POS operations accessible via keyboard shortcuts | FR-010 |
| Transaction atomicity | Sale + payment + inventory deduction + audit entry in single transaction | FR-082 |
| Power failure safety | Incomplete sale not recorded; inventory unchanged on crash | FR-082 |

**Output**: Complete POS workflow — scan/search, build sale, pay, print receipt, void — all keyboard-first.

---

### Phase 6: Customer & Debt Management

**Dependencies**: Phase 5 (POS — credit sales create debt)

**Scope**: User Story 4 — FR-025 through FR-028.

| Item | Description | FR Coverage |
|------|-------------|-------------|
| Customer records | Name, phone, address, notes | FR-025 |
| Credit sales | Unpaid amount recorded as debt against customer | FR-026 |
| Debt payments | Partial payments with complete transaction history (date, amount, user, balance before/after) | FR-027 |
| Sale returns | Inventory restock + customer debt/payment adjustment; available to all roles | FR-028 |
| Walk-in customers | Sales to non-registered customers supported; debt features for registered only | Assumption |
| Customer account view | All credit sales, all payments, current outstanding debt | FR-026 |

**Output**: Customer CRUD, credit sale flow integrated with POS, debt payment tracking, sale returns.

---

### Phase 7: Expenses & Stocktaking

**Dependencies**: Phase 2 (auth), Phase 3 (inventory — for stocktaking). These two sub-modules can be developed in parallel.

**Scope**: User Story 8 (Expenses — FR-029, FR-030) + User Story 10 (Stocktaking — FR-060 through FR-063).

| Item | Description | FR Coverage |
|------|-------------|-------------|
| **Expenses** | | |
| Expense recording | Date, amount (EGP), category, description | FR-029 |
| Expense in financials | Expenses included in financial calculations and reports | FR-030 |
| Expense audit trail | Creation recorded with user, timestamp, amount | FR-029 |
| **Stocktaking** | | |
| Initiate stocktake | Select all or by category; display product name + recorded qty + entry field | FR-060 |
| Count entry | Enter counted quantities alongside recorded quantities | FR-061 |
| Discrepancy display | Show differences between recorded and counted, highlighted | FR-062 |
| Adjustment approval | Pharmacist approves; stock updated; audit log with reason "Stocktaking" | FR-063 |

**Output**: Expense entry and listing; stocktaking workflow with discrepancy review and approval.

---

### Phase 8: Reports

**Dependencies**: Phases 3–7 (all data-producing modules must be complete for accurate reports)

**Scope**: User Story 9 — FR-054 through FR-059a.

| Item | Description | FR Coverage |
|------|-------------|-------------|
| Sales report | Total sales, transaction count, product breakdown, payment summary for date range | FR-054 |
| Inventory status report | All products, current stock levels, batch details, expiry dates | FR-055 |
| Expiry report | Products expiring within configurable period, sorted by urgency, quantities + values | FR-056 |
| Debt report | Customers with outstanding balances, sorted by amount, with aging | FR-057 |
| Purchase report | Purchases by supplier, paid vs. outstanding, product breakdown | FR-058 |
| Profit overview | Revenue, COGS, expenses, net profit for selected period | FR-059 |
| Output modes | On-screen viewing, printing to any installed printer, file export | FR-059a |

**Output**: Six report types with screen/print/export support.

---

### Phase 9: AI Recommendation Engine

**Dependencies**: Phase 3 (inventory data), Phase 5 (sales history), Phase 4 (purchase/supplier data)

**Scope**: User Story 7 — FR-044 through FR-053.

| Item | Description | FR Coverage |
|------|-------------|-------------|
| Reorder recommendation | Sales rate + current stock + supplier lead time → order quantity | FR-044 |
| Sales trend & demand forecast | Rising/falling demand, seasonal patterns, future projection | FR-045 |
| Expiry risk | Near-expiry batches: qty at risk, financial value, suggested actions | FR-046 |
| Slow-moving & dead stock | No sales in 90+ days, capital tied up, suggested actions | FR-047 |
| Overstock risk | Stock exceeding projected demand, months of surplus, capital at risk | FR-048 |
| Stockout risk | Predicted depletion date, urgency level, recommended action | FR-049 |
| Recommendation format | Every recommendation: Score (0–100), Priority, Reason (text), Action (text) | FR-050 |
| No unexplained facts | Recommendations always include all four components | FR-051 |
| Data quality handling | "Collecting data" for products with < 30 days history | FR-052 |
| Configurable thresholds | Safety stock days, lead times, overstock limits — Admin-configurable | FR-053 |

**Output**: Local rule-based scoring engine with six recommendation types, configurable thresholds, and data-quality awareness.

---

### Phase 10: System Configuration & Backup

**Dependencies**: Phase 2 (admin auth), database infrastructure. Can begin after Phase 2; backup testing benefits from having data-producing modules.

**Scope**: User Story 6 (Backup — FR-037 through FR-043) + User Story 13 (Configuration — FR-077 through FR-079).

| Item | Description | FR Coverage |
|------|-------------|-------------|
| **Configuration** | | |
| Pharmacy identity | Name, logo, address, phone — appears on receipts and UI | FR-077 |
| Receipt layout | Content and layout configuration; printer selection | FR-078 |
| Settings storage | All customer-specific settings in config, not code | FR-079 |
| **Backup & Recovery** | | |
| Automatic backup | Configurable interval, no interruption to normal operation | FR-037 |
| Backup location | Local, USB, external drives — configurable by Admin | FR-038 |
| Backup rotation | Multiple versions with configurable retention count | FR-039 |
| Integrity verification | Verify backup validity after each backup | FR-040 |
| Restore with safety backup | Create safety backup of current state before restoring | FR-041 |
| Crash recovery | SQLite WAL automatic recovery; no partial transactions | FR-042 |
| No business data deletion | Only backup files subject to rotation | FR-043 |

**Output**: Configuration UI, backup scheduler, backup/restore workflow, integrity checking.

---

### Phase 11: Licensing & Activation

**Dependencies**: Application shell. Prototype P-10.3 (Hardware Fingerprint) must be validated.

**Scope**: User Story 12 — FR-072 through FR-076.

| Item | Description | FR Coverage |
|------|-------------|-------------|
| License file verification | Signed license file checked at startup | FR-072 |
| Hardware binding | License bound to machine's hardware fingerprint | FR-073 |
| Access denial | Missing/invalid/tampered/mismatched license blocks access | FR-074 |
| Reactivation | Controlled process for legitimate hardware changes | FR-075 |
| Offline verification | No internet required for license check | FR-076 |

**Output**: License verification at startup, hardware fingerprint generation, reactivation flow.

---

### Phase 12: LAN Multi-PC

**Dependencies**: ALL prior phases (must integrate with every module). Prototype P-10.1 (LAN) MUST pass — this is the critical prototype.

**Scope**: User Story 11 — FR-064 through FR-071, plus LAN auth from `specs/002-auth-permissions/` (FR-030 through FR-033).

| Item | Description | FR Coverage |
|------|-------------|-------------|
| Single-PC mode | Default; direct SQLite access, no network, no configuration | FR-064 |
| Server mode | One PC hosts SQLite + TCP server; config toggle | FR-065, FR-066 |
| Client mode | Other PCs connect via TCP client; `NetworkRepository` implementation | FR-065, FR-066 |
| No offline queuing | Server unreachable → transactions blocked | FR-067 |
| Write serialization | Server serializes all writes; concurrent reads via WAL | FR-068 |
| Optimistic concurrency | Version column prevents silent overwrites on concurrent edits | FR-069 |
| Auto-reconnection | Clients reconnect after server/network interruption | FR-070 |
| Pharmacy secret | Shared secret required before user authentication | FR-071 |
| LAN auth flow | Secret → TCP connect → user login against shared DB | Auth FR-030–033 |

**Output**: Dual-mode application (single/LAN), TCP server/client, NetworkRepository, concurrency control.

**Fallback**: If Prototype P-10.1 fails, fall back to MariaDB as database (Architecture Discovery Section 3). This eliminates the custom networking layer but adds deployment complexity (MariaDB server install).

---

## Cross-Cutting Concerns (All Phases)

These requirements apply across every phase and are not isolated to a single module.

| Concern | Requirement | FR |
|---------|-------------|-----|
| Currency | All amounts in EGP; prices are final (no separate tax) | FR-080 |
| Arabic UI | Entire UI in Arabic with correct RTL layout | FR-081 |
| Atomicity | All financial/inventory operations in atomic transactions | FR-082 |
| Offline operation | All core operations work without Internet | FR-083 |
| Audit trail | Sensitive actions logged with action, user, timestamp, details | FR-035, FR-036 |
| Permission enforcement | Service-layer checks, not just UI hiding | FR-034 |

---

## Phase Dependency Graph

```
Prototypes P-10.4, P-10.5
        │
        ▼
    Phase 1: Foundation
        │
        ▼
    Phase 2: Auth & Permissions
        │
        ├──────────────────────┐
        ▼                      ▼
    Phase 3: Inventory     Phase 10: Config & Backup
        │                      │
        ├───────────┐          │ (can start after Phase 2)
        ▼           ▼          │
    Phase 4:    Phase 5:       │
    Purchases   POS ◄── P-10.2│
        │           │          │
        │           ▼          │
        │       Phase 6:       │
        │       Customer/Debt  │
        │           │          │
        ├───────────┤          │
        ▼           ▼          │
    Phase 7: Expenses &        │
    Stocktaking                │
        │                      │
        ▼                      │
    Phase 8: Reports ◄─────────┘
        │
        ▼
    Phase 9: AI Engine
        │
        ▼
    Phase 11: Licensing ◄── P-10.3
        │
        ▼
    Phase 12: LAN ◄── P-10.1
```

**Parallelization Opportunities**:
- Phase 10 (Config & Backup) can develop in parallel with Phases 3–7
- Phase 7 sub-modules (Expenses and Stocktaking) can develop in parallel
- Phase 4 (Purchases) and Phase 5 (POS) can develop in parallel after Phase 3
- Phase 11 (Licensing) can develop in parallel with Phases 8–9 (after prototype P-10.3)

---

## Project Structure

### Documentation

```text
specs/001-pharmacy-management-v1/
├── spec.md                  # V1 system specification (approved)
├── plan.md                  # This file — full-system implementation plan
├── research.md              # Phase 0 — consolidated research decisions
├── data-model.md            # Phase 1 — full entity model
├── contracts/               # Phase 1 — service interface contracts by module
│   ├── inventory-service.md
│   ├── purchase-service.md
│   ├── pos-service.md
│   ├── customer-debt-service.md
│   ├── expense-service.md
│   ├── stocktaking-service.md
│   ├── report-service.md
│   ├── recommendation-service.md
│   ├── configuration-service.md
│   ├── backup-service.md
│   └── license-service.md
├── quickstart.md            # Phase 1 — validation guide
└── checklists/
    └── requirements.md      # Specification quality checklist (passed)

specs/002-auth-permissions/   # Module-level detail (complete)
├── spec.md
├── plan.md
├── research.md
├── data-model.md
├── contracts/
│   ├── auth-service.md
│   ├── permission-service.md
│   ├── user-service.md
│   └── audit-log-service.md
└── quickstart.md
```

### Source Code (repository root)

```text
src/
  PharmacyApp/
    Models/
      Product.cs, Batch.cs, Category.cs
      Sale.cs, SaleLineItem.cs
      Purchase.cs, PurchaseLineItem.cs
      Customer.cs, Supplier.cs
      Payment.cs, Expense.cs
      User.cs, AuditLogEntry.cs
      Role.cs, Permission.cs
      Recommendation.cs, StockAdjustment.cs
      License.cs, BackupRecord.cs
      PharmacyConfig.cs
    Services/
      Auth/          (IAuthService, IPermissionService, IUserService, IAuditLogService)
      Inventory/     (IProductService, IBatchService, ICategoryService, IStockService)
      Purchasing/    (IPurchaseService, ISupplierService)
      Sales/         (ISaleService, IReceiptService)
      Customers/     (ICustomerService, IDebtService)
      Expenses/      (IExpenseService)
      Stocktaking/   (IStocktakeService)
      Reports/       (IReportService)
      AI/            (IRecommendationService)
      Config/        (IConfigurationService)
      Backup/        (IBackupService)
      Licensing/     (ILicenseService)
    Repositories/
      IRepository.cs           # Generic base
      SqliteRepository.cs      # Direct SQLite (single-PC / server)
      NetworkRepository.cs     # TCP client (LAN — Phase 12)
      [Per-entity repositories]
    Forms/
      LoginForm.cs
      MainForm.cs              # Navigation shell
      POS/                     # Sale screen, receipt preview
      Inventory/               # Product management, batch view, stock adjustment
      Purchasing/              # Purchase entry, supplier management
      Customers/               # Customer management, debt view
      Expenses/                # Expense entry
      Stocktaking/             # Physical count workflow
      Reports/                 # Report viewer with print/export
      AI/                      # Recommendation dashboard
      Admin/                   # User management, audit log, config, backup
    Infrastructure/
      DatabaseMigrations.cs
      TransactionManager.cs
      UnitOfWork.cs
    Security/
      PasswordHasher.cs
      LicenseVerifier.cs
    Networking/                # Phase 12
      TcpServer.cs
      TcpClient.cs
      Protocol.cs
    Printing/
      ReceiptBuilder.cs
      EscPosDriver.cs
      WindowsPrintDriver.cs
  PharmacyApp.sln

tests/
  PharmacyApp.Tests/
    Unit/
      Services/    (per-service test files)
      Security/
    Integration/
      Auth/, Inventory/, POS/, Purchasing/, Customers/,
      Expenses/, Stocktaking/, Reports/, AI/, Backup/,
      Licensing/, LAN/
```

**Structure Decision**: Standard layered architecture — Models (entities), Services (business logic with permission enforcement and audit logging), Repositories (data access via Dapper), Forms (WinForms UI), Infrastructure (database, transactions). Repository pattern supports two implementations: `SqliteRepository` (direct, for single-PC and server) and `NetworkRepository` (TCP client, for LAN). Permission checks enforced at the Service layer. Services grouped by business domain.

---

## Dependencies & Risks

### External Dependencies

| Dependency | Type | Impact |
|------------|------|--------|
| .NET Framework 4.8 | Runtime | Must be installed on target machines; ships with Win10+, manual install on Win7 |
| System.Data.SQLite | NuGet | Must verify 32-bit compatibility; has official x86 builds |
| Dapper | NuGet | Must verify .NET Framework 4.8 compatibility; well-established |
| NUnit + Moq | NuGet (dev) | Test-time only; no runtime impact |

### Risks

| Risk | Severity | Mitigation | Phase |
|------|----------|------------|-------|
| LAN prototype fails | High | MariaDB fallback architecture; single-PC works regardless | 12 |
| Arabic thermal printing inconsistencies | Medium | Windows driver fallback; prototype P-10.2 validation | 5 |
| Hardware fingerprint instability on old PCs | Medium | Flexible match tolerance; prototype P-10.3 validation | 11 |
| WinForms Arabic RTL limitations | Medium | RTL prototype P-10.4; third-party controls as fallback | 1 |
| 32-bit NuGet package incompatibility | Medium | Verify every package on x86 before adoption | 1 |
| Audit log table growth over years | Low | SQLite handles millions of rows; proper indexing | 2 |
| Report performance with large datasets | Low | Indexed queries; pagination for display | 8 |
| AI engine accuracy with limited data | Low | "Collecting data" status for insufficient history | 9 |

---

## Complexity Tracking

No Constitution violations. No justifications needed.

All architectural decisions align with confirmed decisions from Architecture Discovery. The Repository pattern (two implementations) is the prescribed LAN architecture from ADR-003, not an unnecessary abstraction.
