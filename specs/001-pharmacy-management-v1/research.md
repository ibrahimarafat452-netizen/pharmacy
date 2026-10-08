# Research: Pharmacy Management System V1

**Date**: 2026-10-08
**Feature**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md)

**Note**: Auth-specific research (password hashing, optimistic concurrency, audit log enforcement, default admin bootstrap, audit log scaling, LAN pharmacy secret) is fully documented in `specs/002-auth-permissions/research.md`. This document covers system-wide and remaining module-specific research questions.

---

## Research Question 1: Database Migration Strategy

**Context**: The system needs schema versioning to create tables on first run and support future upgrades. Must work with SQLite and be simple enough for the project's scale.

**Decision**: Manual migration runner with numbered SQL scripts

**Rationale**:
- Simple and transparent — each migration is a numbered `.sql` file or embedded resource
- No dependency on heavy migration frameworks (Entity Framework Migrations, FluentMigrator)
- A `SchemaVersion` table tracks which migrations have been applied
- On startup: read current version, execute all unapplied migrations in order, within a transaction
- Aligns with Constitution XIV (Simplicity) and XVIII (Learning Principle)

**Alternatives Considered**:
- **Entity Framework Migrations**: Requires EF dependency; overkill for Dapper-based project. Rejected.
- **FluentMigrator**: Mature library but adds NuGet dependency and abstraction layer. Rejected — the system has ~15 tables; hand-written SQL is manageable and educational.
- **DbUp**: Lightweight, runs SQL scripts. Viable but adds unnecessary dependency for the same pattern we can implement in ~50 lines. Rejected.

**Implementation Notes**:
- Migration table: `CREATE TABLE IF NOT EXISTS SchemaVersion (Version INTEGER PRIMARY KEY, AppliedAt TEXT NOT NULL)`
- Migrations stored as embedded resources: `Migrations/001_initial_schema.sql`, `002_add_indexes.sql`, etc.
- Migration runner executes at application startup before any business logic
- Each migration runs in its own transaction; if one fails, the database stays at the last successful version

---

## Research Question 2: Box/Strip Unit Conversion Model

**Context**: FR-015 requires box and strip units with automatic conversion. A box of 10 strips sold as 3 strips should show 7 strips remaining. Need to determine how to store and calculate fractional units.

**Decision**: Store all quantities in strips (smallest unit). Display in box+strip format using the strips-per-box ratio.

**Rationale**:
- Eliminates floating-point issues — all arithmetic is on integers (strip count)
- A "box" is a display convenience: `boxes = strips / strips_per_box`, `remaining = strips % strips_per_box`
- Products with `unit_type = box` have `strips_per_box > 1`; products sold only as whole units have `strips_per_box = 1`
- FEFO deductions, stock checks, and adjustments all operate on strip counts
- Simple and unambiguous — no decimal rounding surprises on financial/inventory data

**Alternatives Considered**:
- **Store as decimal boxes (e.g., 0.7 boxes)**: Introduces floating-point arithmetic for inventory. Rejected — financial/inventory accuracy (Constitution VI, VIII) demands integer precision.
- **Dual columns (boxes + strips)**: Adds complexity to every query and calculation. Rejected — single integer strip count is simpler and equivalent.

**Implementation Notes**:
- Product table: `StripsPerBox INTEGER NOT NULL DEFAULT 1`
- Batch table: `Quantity INTEGER NOT NULL` (in strips)
- Display helper: `FormatQuantity(strips, stripsPerBox)` → e.g., "2 boxes, 3 strips" or "23 strips"
- Sale input: User enters quantity in the product's selling unit (box or strip). If selling unit is "box," multiply by strips_per_box for internal deduction.
- When a product is sold in strips, deduction is 1:1. When sold as a box, deduction = 1 × strips_per_box.

---

## Research Question 3: FEFO (First Expiry, First Out) Implementation

**Context**: FR-004, FR-013 require automatic FEFO for all inventory deductions. Need to determine the batch selection algorithm.

**Decision**: Query batches ordered by expiry date ascending, skip expired batches, deduct sequentially until the required quantity is fulfilled.

**Rationale**:
- Straightforward SQL: `SELECT * FROM Batches WHERE ProductId = @id AND Quantity > 0 AND ExpiryDate > @today ORDER BY ExpiryDate ASC`
- Iterate in application code, deducting from each batch until total quantity is met
- All within a single database transaction — if any step fails, everything rolls back
- Handles partial batch deductions (e.g., need 15 strips, batch A has 10, batch B has 20 → take 10 from A, 5 from B)

**Alternatives Considered**:
- **Database-level procedure**: SQLite doesn't support stored procedures. Rejected.
- **Deduct from single batch only**: Doesn't handle cases where one batch has insufficient stock. Rejected.

**Implementation Notes**:
- Implemented in `IStockService.DeductStock(productId, quantity)` which returns a `List<BatchDeduction>` recording which batches were used
- Each `BatchDeduction` is stored on the `SaleLineItem` for traceability
- Stock validation happens before deduction: total available (non-expired) stock must be >= requested quantity
- Expired batches (ExpiryDate <= today) are excluded from the query

---

## Research Question 4: Report Export Format

**Context**: FR-059a requires reports to support "exporting to file." The spec says export format is "to be determined during planning."

**Decision**: PDF export (primary) using a lightweight .NET PDF library

**Rationale**:
- PDF is universally viewable, preserves formatting, and supports Arabic/RTL text
- Customers (pharmacy staff) are familiar with PDF files
- A single export format keeps the implementation simple (Constitution XIV)
- PDF files are print-ready, matching the report's screen/print appearance

**Alternatives Considered**:
- **Excel (.xlsx)**: Useful for data analysis but adds a heavy dependency (ClosedXML or EPPlus) and is overkill for fixed reports. Could be added in a future version.
- **CSV**: Loses formatting, doesn't support Arabic well in all environments (encoding issues with Excel on Arabic Windows). Rejected as primary format.
- **HTML**: Requires a browser to view; doesn't feel like a "file" to end users. Rejected.

**Implementation Notes**:
- Library candidates: **PDFsharp** (MIT license, .NET Framework compatible, supports Unicode/Arabic). Must verify 32-bit compatibility and Arabic text rendering.
- If PDFsharp Arabic support is insufficient, fallback: use the Windows print driver to "print to PDF" using a built-in PDF printer (available on Win10+; Win7 would need a third-party PDF printer).
- Report rendering: Reports are designed as WinForms `PrintDocument` objects. The same rendering code drives screen preview, physical printing, and PDF export.

---

## Research Question 5: Barcode Scanner Integration

**Context**: FR-002 requires barcode scanning to add products to a sale. Need to determine how barcode scanners integrate with WinForms.

**Decision**: Barcode scanners act as keyboard input — no special driver or API integration needed.

**Rationale**:
- Standard USB/Bluetooth barcode scanners emulate keyboard input: they "type" the barcode characters followed by Enter
- WinForms receives this as regular `KeyPress` events on the focused text field
- The POS barcode field receives the string, looks up the product, and adds it to the sale
- This is the industry-standard approach for POS barcode scanning
- No special SDK, driver, or library needed — any HID-compliant scanner works out of the box

**Alternatives Considered**:
- **Serial port / COM communication**: Only needed for legacy serial scanners. Not worth the complexity for modern USB scanners. Rejected for V1.
- **Scanner SDK**: Vendor-specific SDKs (Honeywell, Zebra) are unnecessary for basic barcode reading. Rejected.

**Implementation Notes**:
- POS form has a barcode input field that is always focused by default
- When Enter is pressed (scanner's suffix), the field content is used to look up the product
- Manual barcode entry is supported (user types the number and presses Enter)
- Scanners must be configured to send Enter as suffix (most do by default)
- Product lookup: `SELECT * FROM Products WHERE Barcode = @barcode`

---

## Research Question 6: Arabic Search (Partial Name Match)

**Context**: FR-003 requires searching products by Arabic name with partial match. SQLite's default `LIKE` is case-sensitive and doesn't handle Arabic normalization. Need to determine the search approach.

**Decision**: SQLite `LIKE` with `%` wildcards — sufficient for Arabic pharmacy product names without normalization.

**Rationale**:
- Arabic product names in Egyptian pharmacies are typically written in a consistent form (no diacritics, no tashkeel on product names)
- `LIKE '%search_term%'` handles partial matching for Arabic text (SQLite stores and compares UTF-8 correctly)
- No need for full-text search (FTS) — the product catalog is small (up to 5,000 products) and LIKE with an index performs adequately
- Constitution XIV (Simplicity) — avoid unnecessary complexity for a simple search

**Alternatives Considered**:
- **SQLite FTS5 (Full-Text Search)**: Powerful but adds complexity. Arabic tokenization in FTS5 is not well-documented. Overkill for 5,000 products. Rejected for V1.
- **Application-level filtering**: Load all products into memory and filter. Works for 5,000 products but bypasses SQLite's query optimization. Rejected — keep data access in the database layer.
- **Arabic normalization (stripping tashkeel, normalizing hamza/alef)**: May be needed if users type with diacritics. Deferred — if testing shows issues, a simple normalization function can be added to the LIKE query.

**Implementation Notes**:
- Query: `SELECT * FROM Products WHERE Name LIKE @pattern ORDER BY Name` where `@pattern = '%' + searchTerm + '%'`
- Results displayed in a dropdown or list as the user types (debounced for performance)
- If the product list is loaded in memory for POS speed, application-level filtering is acceptable as an optimization

---

## Research Question 7: Receipt Printing Architecture

**Context**: FR-008 requires thermal receipt printing (58mm/80mm). Architecture Discovery ADR-007 decided "Windows driver + direct ESC/POS where appropriate." Prototype P-10.2 required.

**Decision**: Primary approach is Windows print driver via `PrintDocument`. ESC/POS direct printing as optional/fallback.

**Rationale**:
- Windows `PrintDocument` API is built into .NET Framework — zero external dependency
- Works with any printer that has a Windows driver installed (broad compatibility)
- Arabic text rendering through GDI+ (same as WinForms UI rendering)
- ESC/POS direct printing (via serial/USB port) gives finer control but Arabic encoding varies by printer model — requires prototype validation
- Start with Windows driver (reliable, universal); add ESC/POS as an enhancement if prototype P-10.2 shows consistent results

**Alternatives Considered**:
- **ESC/POS only**: Maximum control but Arabic encoding inconsistency across printer models is a known risk. Rejected as primary approach.
- **Third-party receipt library**: Adds dependency; most target modern frameworks. Rejected.

**Implementation Notes**:
- `IReceiptService.PrintReceipt(Sale)` generates receipt content and sends to printer
- Receipt content: Pharmacy name/logo (from config), items (name, qty, price), totals, date/time, cashier name, sale number
- `PrintDocument.PrintPage` event handler draws text using `Graphics.DrawString` with Arabic font
- Paper width: 58mm (32 chars) or 80mm (48 chars) — configurable in system settings
- If printer not connected: complete the sale, show error, offer retry or skip printing

---

## Research Question 8: AI Recommendation Engine — Scoring Approach

**Context**: FR-044–053 define six recommendation types. Architecture Discovery ADR-006 decided "local rule-based scoring." Need to determine the high-level scoring approach.

**Decision**: Each recommendation type has its own scoring formula producing a Score (0–100) and Priority (Critical/High/Medium/Low), with configurable thresholds.

**Rationale**:
- Deterministic and explainable — same inputs always produce the same output
- Each recommendation type has independent scoring logic — no complex inter-dependencies
- Thresholds (safety stock days, lead times, overstock limits) are Admin-configurable
- Priority derived from score: Critical (80–100), High (60–79), Medium (40–59), Low (0–39) — configurable
- Runs as a background calculation on demand or on a schedule; results cached until recalculated

**Alternatives Considered**:
- **Single unified scoring model**: Overcomplicates the implementation when each recommendation type has distinct inputs and logic. Rejected.
- **Machine learning**: Violates offline constraint (training requires significant compute); explainability is harder; insufficient data for small pharmacies. Rejected per ADR-006.

**Implementation Notes**:
- `IRecommendationService.CalculateRecommendations(productId?)` runs all recommendation types
- Each type is a separate calculator class: `ReorderCalculator`, `ExpiryRiskCalculator`, etc.
- Inputs: Product data, Batch data (quantities, expiry dates), Sales history (last 30/60/90 days), Supplier data (lead times)
- Output: `List<Recommendation>` with Score, Priority, Reason (text), RecommendedAction (text)
- Products with < 30 days sales history: return a recommendation with Status = "CollectingData" and a confidence indicator
- Detailed scoring formulas to be defined in the speckit-tasks phase for each calculator

---

## Research Question 9: Spreadsheet Import Format

**Context**: FR-011a supports "basic spreadsheet import for initial catalog setup/migration only." Need to determine the supported format.

**Decision**: CSV import (comma-separated or semicolon-separated, UTF-8 with BOM)

**Rationale**:
- CSV is the simplest format to parse — no external library required (`TextFieldParser` in .NET or simple string splitting)
- Users can create CSV files from Excel, Google Sheets, or any spreadsheet application
- UTF-8 with BOM ensures Arabic text is preserved when opened in Excel on Arabic Windows
- One-time import operation — does not need to be sophisticated

**Alternatives Considered**:
- **Excel (.xlsx)**: Requires a NuGet library (ClosedXML, EPPlus, or NPOI) for parsing. Adds dependency for a one-time feature. Rejected for V1.
- **JSON**: Not user-friendly for pharmacy staff. Rejected.

**Implementation Notes**:
- Expected columns: Product Name (Arabic), Barcode, Category, Unit Type (box/strip), Strips Per Box, Selling Price
- Import validates each row: required fields, barcode uniqueness, valid unit type, numeric fields
- Errors reported per row with line number and reason; valid rows imported, invalid rows skipped with error report
- Import available from Admin or Pharmacist role (Inventory Management screen)
- No batch import — batches are created through purchases (the primary workflow)

---

## Research Question 10: Dependency Injection in WinForms

**Context**: The project uses a service layer with repository pattern. Need to determine how to wire up services, repositories, and forms without a DI container framework.

**Decision**: Manual composition root — a single `AppBootstrapper` class that creates and wires all dependencies at startup.

**Rationale**:
- WinForms does not have built-in DI support like ASP.NET
- A full DI container (Autofac, Microsoft.Extensions.DependencyInjection) adds a dependency and learning curve
- The application has a fixed set of services (~15-20) that are created once at startup and live for the application lifetime
- Manual wiring in a composition root is transparent, debuggable, and educational (Constitution XVIII)
- Services receive their dependencies through constructor injection — the pattern is the same with or without a container

**Alternatives Considered**:
- **Microsoft.Extensions.DependencyInjection**: Industry standard but targets .NET Core/.NET 5+. The .NET Framework 4.8 compatible version exists but adds complexity. Deferred — can be adopted later if the manual approach becomes unwieldy.
- **Autofac**: Mature DI container. Adds NuGet dependency. Rejected for V1 — manual composition root is sufficient for ~15-20 services.
- **Service Locator pattern**: Anti-pattern that hides dependencies. Rejected.

**Implementation Notes**:
- `AppBootstrapper.Initialize()` called in `Program.Main()` before showing `LoginForm`
- Creates: database connection → repositories → services → forms
- All services are singletons (one instance per application run)
- Forms receive services through constructor parameters
- Example: `var userRepo = new UserRepository(connection); var authService = new AuthService(userRepo, passwordHasher); var loginForm = new LoginForm(authService);`

---

## Research Question 11: Backup Implementation via SQLite Online Backup API

**Context**: Architecture Discovery ADR-005 decided "SQLite online backup API." Need to confirm the approach for .NET Framework 4.8.

**Decision**: Use `System.Data.SQLite`'s `BackupDatabase` method (wraps SQLite's `sqlite3_backup_*` API)

**Rationale**:
- `SQLiteConnection.BackupDatabase()` creates a consistent snapshot of the database while it's in use
- No need to stop the application or close connections — WAL mode allows concurrent reads during backup
- Produces a single `.db` file that is a complete, valid SQLite database
- Integrity verification: open the backup file and run `PRAGMA integrity_check`

**Alternatives Considered**:
- **File copy**: Risk of copying an inconsistent file if writes are in progress. Rejected.
- **SQL dump and restore**: Adds complexity; text dumps are larger than binary copies. Rejected.

**Implementation Notes**:
- Backup runs on a background thread with configurable interval (e.g., every 30 minutes)
- Backup file naming: `pharmacy_backup_YYYYMMDD_HHMMSS.db`
- After backup: run `PRAGMA integrity_check` on the backup file; log result
- Rotation: keep N most recent backups (configurable), delete oldest when exceeded
- Backup location: path configurable by Admin; validate path exists and is writable
- Restore: (1) create safety backup of current database, (2) close all connections, (3) copy backup file over current database, (4) reopen and verify integrity
