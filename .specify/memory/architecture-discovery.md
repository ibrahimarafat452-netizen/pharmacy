# Pharmacy Management System — Architecture Discovery

**Date**: 2026-10-07
**Revised**: 2026-10-08
**Status**: Proposed — pending prototype validation
**Constitution Reference**: v1.0.0

---

## 1. Project Constraints

| Constraint | Detail |
|------------|--------|
| Target market | Small/medium Egyptian pharmacies |
| Operating system | Windows 7 SP1+ (some machines 32-bit) |
| Hardware | Weak/old PCs, limited RAM and CPU |
| Network | Fully offline; LAN for multi-PC pharmacies |
| Language | Arabic-only UI (RTL) |
| Currency | Egyptian Pound (EGP) only |
| Peripherals | 58mm/80mm thermal receipt printers, barcode/QR scanners |
| POS workflow | Keyboard-first, minimal clicks, fast sales |
| Data | One authoritative local database per pharmacy |
| Deployment | Single Core Product; per-pharmacy config/license/logo |

---

## 2. Recommended Technology Stack

| Component | Technology |
|-----------|------------|
| Language | C# (.NET Framework 4.8) |
| UI | Windows Forms |
| Database | SQLite (WAL mode) |
| Data access | Dapper |
| LAN | Custom TCP server (application-level) |
| Printing | Windows driver + direct ESC/POS where appropriate |
| Testing | NUnit + Moq |
| Installer | WiX Toolset or NSIS |
| IDE | Visual Studio 2022 Community |

**Why this stack:** Selected primarily because it fits the project's
Windows 7 / 32-bit / legacy-hardware constraints. Alternatives were
evaluated and rejected:

- **Electron** — dropped Windows 7 and 32-bit support; 200+ MB RAM
  baseline. Disqualified.
- **Tauri** — requires WebView2 runtime on Windows 7 (fragile
  dependency). High risk.
- **Java Swing** — JVM overhead (100–200 MB RAM) on weak hardware.
- **Python + PyQt5** — Python 3.8 (last Win7-compatible version) is
  EOL. No compile-time type safety for financial code.
- **Local web app** — requires a modern browser; Windows 7 ships with
  IE11 at best.
- **C++ / Win32** — development complexity violates the Learning
  Principle (Constitution XVIII).
- **PostgreSQL** — no maintained 32-bit Windows builds (dropped after
  v10 EOL). Disqualified as primary database.

---

## 3. Database Architecture

**Decision:** SQLite with WAL mode (embedded).

**Rationale:**
- Zero deployment — embedded DLL, no database server to install
- Full ACID transactions with automatic crash recovery via WAL
- 32-bit fully supported
- Sufficient for pharmacy workloads (hundreds to low thousands of
  transactions per day)
- Most installations (~95%) are single-PC, where SQLite is ideal
- Simple backup — SQLite online backup API creates consistent snapshots
  without stopping the application

**Trade-off:** SQLite is not designed for multi-client network access.
LAN support requires a custom application-level server (see next
section). This trades one-time development cost for permanently simpler
deployment.

**Fallback:** If the LAN prototype (see Section 10) reveals
insurmountable concurrency issues, MariaDB (which has 32-bit Windows
builds) is the fallback database. This would add deployment complexity
but eliminate the need for a custom networking layer.

---

## 4. LAN Architecture

### Single-PC Mode (Default)

Application opens the SQLite database directly. No server, no network,
no configuration.

### Multi-PC Mode (LAN)

```
Server PC (Main)           Client PCs
┌────────────────┐         ┌──────────────┐
│ WinForms UI    │         │ WinForms UI  │
│ Business Logic │         │ Business Logic│
│ Data Layer     │         │ TCP Client ──┼──┐
│ TCP Server ────┼─ LAN ──►│              │  │
│ SQLite (WAL)   │         └──────────────┘  │
└────────────────┘         ┌──────────────┐  │
                           │ WinForms UI  │  │
                           │ Business Logic│  │
                           │ TCP Client ──┼──┘
                           └──────────────┘
```

**Key design decisions:**
- Same executable on all PCs — server mode is a configuration toggle
- Repository pattern with two implementations: `SqliteRepository`
  (direct, for server/single-PC) and `NetworkRepository` (TCP client)
- Server serializes all database writes; reads are concurrent (WAL)
- If the server PC goes offline, clients cannot transact — acceptable
  for 2–4 PCs in a small pharmacy
- No offline queuing on clients — split-brain inventory violates
  Constitution Principles VI and VIII

**LAN implementation is subject to successful prototype validation** —
see Section 10.

---

## 5. Data Integrity

All financial and inventory operations execute inside a single database
transaction. A sale, its payment, its inventory adjustment, and its
audit entry either all commit or none do.

**Power failure / crash recovery:** SQLite WAL guarantees the database
is always in a consistent state after any crash. Uncommitted
transactions are rolled back automatically on next open.

**Concurrent access (LAN):** The TCP server serializes write operations.
Optimistic concurrency control (version column on entities) prevents
silent overwrites when two PCs modify the same record.

**Inventory rules:** Stock MUST NOT go negative. FEFO is enforced
within the transaction. All stock adjustments are audit-trailed.

---

## 6. Backup & Recovery

**Architecture requirements:**
- Automatic local backups at configurable intervals
- Backup location configurable by Admin — prefer non-system drive
  (USB, external, or ) when available
- Backup rotation with configurable retention count
- Integrity verification after each backup
- Restore with pre-restore safety backup and Admin confirmation
- Automatic crash recovery via SQLite WAL (no user action required)
- Business data is NEVER automatically deleted (Constitution XI) —
  only backup files are rotated

**Recovery after unexpected shutdown:** SQLite WAL replays committed
entries and discards uncommitted ones on next startup. If integrity
check fails (rare), the application offers to restore from the most
recent valid backup.

---

## 7. Licensing Architecture

**Decision:** Signed offline license with hardware-bound activation.

**Design:**
- Pharmacy-specific license file signed with the vendor's private key
  (RSA or equivalent asymmetric signature)
- Application embeds the vendor's public key and verifies signature
  at startup
- License is bound to a hardware fingerprint of the target machine
- Controlled reactivation process for legitimate hardware changes
  (pharmacist contacts vendor for a new license file)
- No server dependency for daily operation

**Hardware fingerprint strategy must be validated through a prototype**
before final implementation. Hardware identifiers (CPU serial, disk
serial, MAC address, motherboard serial, etc.) vary in availability
and stability across old Windows machines. The specific fingerprint
formula and match tolerance will be determined during prototyping.

**Security position:** Offline licensing provides strong deterrence
against casual/opportunistic copying (the primary piracy vector for
small business software). It is NOT designed to be unbreakable — any
offline license can be reverse-engineered given sufficient effort.
Additional mitigations (code obfuscation, distributed checks) raise
the difficulty but do not eliminate the risk. This is an accepted
trade-off for the offline-first constraint.

---

## 8. Security & Permissions

**Role-based access control:**
- Admin — full access, user management, configuration, audit log
- Pharmacist — sales, purchases, inventory, reports
- Cashier — sales and payments only

**Architecture requirements:**
- Authentication at application startup (username + hashed password)
- Permission checks enforced at the service layer, not just UI
- Sensitive actions (void sale, stock adjustment, price change, user
  management) recorded in the Audit Log with timestamp, user, and
  details
- Password storage using a strong one-way hash (bcrypt or PBKDF2)
- In LAN mode, user authentication happens after LAN connection is
  established; the LAN connection itself requires a shared pharmacy
  secret

---

## 9. AI Recommendation Engine

**Decision:** Local, deterministic, rule-based scoring engine.

**Architectural constraints:**
- Runs entirely on the local machine — no cloud AI, no LLM, no
  external API
- Deterministic — same inputs always produce the same output
- Explainable — every recommendation includes Score (0–100),
  Priority (Critical/High/Medium/Low), Reason (text), and
  Recommended Action (text)
- Operates on the pharmacy's own historical data (sales, inventory,
  suppliers)

**V1 capabilities:**
1. **Reorder Recommendation** — when and how much to order based on
   sales rate, current stock, and supplier lead time
2. **Sales Trend & Demand Forecast** — rising/falling demand,
   seasonal detection, simple future projection
3. **Expiry Risk** — near-expiry products, quantity at risk,
   suggested actions
4. **Slow-Moving & Dead Stock** — stagnant inventory, capital tied
5. **Overstock Risk** — inventory exceeding expected future demand
6. **Stockout Risk** — predicted depletion date and urgency

**Data quality handling:** Recommendations carry a confidence level
based on available history. New products with insufficient sales data
show "collecting data" instead of unreliable recommendations.

**All thresholds are Admin-configurable** (safety stock days, lead
times, overstock limits, etc.).

Detailed formulas, scoring weights, and business rules belong in a
dedicated AI Recommendation Engine specification — not in this
architecture document.

---

## 10. Required Prototypes

The following decisions MUST be validated with working prototypes
before committing to full implementation.

### 10.1 LAN Prototype (Critical)

SQLite + custom TCP server is a non-standard LAN architecture. The
prototype must verify:

- Multiple PCs (3–4) connecting simultaneously
- Concurrent reads and writes (sale on one PC while another reads
  inventory)
- Database locking behavior under concurrent operations
- Network interruption handling and automatic reconnection
- Server PC restart — clients reconnect and resume
- Client PC restart — no data corruption on server
- Recovery after connection loss mid-transaction
- Data integrity under all concurrent scenarios

**If the prototype fails:** Fall back to MariaDB as the database
(eliminates the custom networking layer but adds deployment complexity).

### 10.2 Arabic Thermal Printing Prototype

ESC/POS Arabic encoding varies between printer models. The prototype
must verify correct Arabic receipt output on 2–3 common Egyptian
pharmacy printer models. Windows driver fallback must also be tested.

### 10.3 Hardware Fingerprint Prototype

Hardware identifiers vary in availability and stability across old
PCs. Test fingerprint generation on 5–10 different hardware
configurations to verify consistency across reboots and minor system
changes. Determine which identifiers are reliable and what match
tolerance to use.

### 10.4 WinForms Arabic RTL Prototype

Verify correct Arabic rendering, RTL layout, keyboard navigation,
and barcode scanner input in WinForms on a Windows 7 32-bit machine.

### 10.5 SQLite Crash Recovery Verification

Simulate power failure (kill process) during a multi-step sale
transaction. Verify the database recovers to a consistent state with
no partial transactions.

---

## 11. Main Technical Risks

| Risk | Severity | Mitigation |
|------|----------|------------|
| SQLite + custom TCP for LAN is unproven | Medium | LAN prototype before implementation; MariaDB fallback |
| Arabic ESC/POS encoding varies by printer model | Medium | Printing prototype; Windows driver fallback |
| Hardware fingerprint instability on old PCs | Medium | Fingerprint prototype; flexible match strategy |
| Custom networking code introduces bugs | Medium | Keep protocol simple; comprehensive tests |
| Third-party NuGet library 32-bit compatibility | Medium | Verify every package on 32-bit before adoption |
| WinForms limitations for complex Arabic layouts | Low | RTL prototype; third-party controls if needed |

---

## 12. Architecture Decision Records

### ADR-001: Application Framework

- **Decision:** C# / .NET Framework 4.8 / Windows Forms
- **Reason:** Only option that meets all hard constraints (Win7,
  32-bit, low resources, Arabic RTL, printing, barcode scanners)
  without compromise.
- **Main tradeoff:** WinForms UI is functional but not visually
  modern. Acceptable for a POS/business application.
- **Validation:** Arabic RTL prototype on Windows 7 32-bit (10.4).

### ADR-002: Database Engine

- **Decision:** SQLite with WAL mode (embedded)
- **Reason:** Zero deployment complexity, full ACID, crash recovery,
  32-bit support, ideal for the common single-PC case.
- **Main tradeoff:** Requires a custom application-level server for
  LAN — more development, but simpler deployment forever.
- **Validation:** LAN prototype (10.1) and crash recovery test (10.5).

### ADR-003: LAN Architecture

- **Decision:** Application-level TCP server with Repository pattern
- **Reason:** Same executable on all PCs; no database server install;
  simple deployment.
- **Main tradeoff:** Custom networking code; server PC is a single
  point of failure.
- **Validation:** LAN prototype (10.1) — this decision is conditional
  on successful prototype results.

### ADR-004: Licensing

- **Decision:** Signed offline license with hardware-bound activation
- **Reason:** Fully offline; deters casual copying; no daily server
  dependency.
- **Main tradeoff:** Offline license is a deterrent, not unbreakable.
  Hardware changes require manual reactivation.
- **Validation:** Hardware fingerprint prototype (10.3).

### ADR-005: Backup & Recovery

- **Decision:** SQLite online backup API with rotation and integrity
  verification
- **Reason:** Creates consistent snapshots without downtime; simple
  file-based restore.
- **Main tradeoff:** Local backup only — theft or disk failure can
  lose both original and backup. Mitigated by supporting external
  backup locations.

### ADR-006: AI Recommendation Engine

- **Decision:** Local rule-based scoring system
- **Reason:** Fits offline constraint, runs on weak hardware, produces
  explainable results, works immediately with basic data.
- **Main tradeoff:** Cannot detect complex patterns that ML could
  find. Acceptable for the target scale (hundreds of products).

### ADR-007: Printing

- **Decision:** Windows driver support + direct ESC/POS where
  appropriate
- **Reason:** ESC/POS gives fine-grained control for thermal receipt
  formatting; Windows driver provides a fallback.
- **Main tradeoff:** Must handle printer-specific Arabic encoding
  quirks.
- **Validation:** Arabic thermal printing prototype (10.2).

---

## 13. Architecture Status

### Confirmed Decisions

- **Application framework:** C# / .NET Framework 4.8 / WinForms
- **Database:** SQLite with WAL mode
- **Offline-first:** No cloud services, no external APIs
- **Arabic-only UI, EGP only**
- **AI Recommendation Engine:** Local rule-based scoring (V1 scope
  defined above)
- **Backup:** SQLite online backup with rotation and integrity checks
- **Licensing:** Signed offline license with hardware binding

### Decisions Requiring Prototype Validation

| Decision | Prototype | Fallback if fails |
|----------|-----------|-------------------|
| SQLite + TCP server for LAN | 10.1 LAN Prototype | MariaDB as database |
| Arabic thermal printing | 10.2 Printing Prototype | Windows driver only (reduced control) |
| Hardware fingerprint formula | 10.3 Fingerprint Prototype | Simplified fingerprint or alternative binding |
| WinForms Arabic RTL on Win7 32-bit | 10.4 RTL Prototype | Third-party UI controls |
| SQLite crash recovery | 10.5 Crash Recovery Test | Expected to pass; failure would reconsider database choice |

### Next Step

This document moves to `/speckit-specify` for detailed feature
specifications based on the confirmed architecture and prototype plan.
