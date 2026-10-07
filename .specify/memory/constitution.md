<!--
## Sync Impact Report
- Version change: (none — raw template) → 1.0.0
- Modified principles: N/A (initial adoption — all placeholders replaced)
- Added sections:
  - 18 Core Principles (I through XVIII)
  - Operational Standards (Security, Backup, Licensing, Architecture)
  - Development Standards (Simplicity, Testing, Agent Governance, Scope, Learning)
  - Governance (amendment procedure, versioning, compliance)
- Removed sections: None
- Deferred TODOs: None
-->

# Pharmacy Management System Constitution

## Core Principles

### I. Product First

The system MUST solve real pharmacy operational problems. Features MUST have
clear business value and MUST NOT be added only for complexity or appearance.

### II. Offline-First

The system MUST operate fully without Internet access.

No core business operation may depend on:
- Cloud services
- External APIs
- Online authentication
- Online AI services

### III. Legacy Hardware Compatibility

The product MUST target weak pharmacy computers and support the defined
Windows 7 environment, including 32-bit where technically feasible.

Performance and resource usage are first-class requirements.

### IV. Local Data Ownership

Each pharmacy has an independent local database.

A pharmacy's Sales, Purchases, Inventory, Debts, Customers, Suppliers,
Users, and Reports MUST remain isolated from other pharmacies.

### V. LAN Support

The system MUST support multiple computers inside the same pharmacy through
a local network while sharing the pharmacy's authoritative database.

Internet MUST NOT be required for LAN operation.

### VI. Data Integrity

Financial and inventory operations MUST be atomic and consistent.

Unexpected shutdowns, application crashes, or power failures MUST NOT leave
sales, payments, inventory, or debt records in an inconsistent state.

### VII. Inventory Accuracy

Inventory MUST support:
- Batch tracking
- Expiry dates
- FEFO (First Expiry, First Out)
- Box/Strip units
- Stocktaking
- Stock adjustments
- Audit history

Inventory changes MUST be traceable.

### VIII. Financial Accuracy

Sales, purchases, payments, debts, returns, expenses, and supplier balances
MUST be accurately recorded.

Partial payments MUST be supported and every payment MUST retain its
transaction history.

### IX. Explainable AI Recommendation Engine

The system may use the name **AI Recommendation Engine** for its intelligent
rule-based analysis.

It MUST operate locally without external AI APIs.

Recommendations MUST be explainable and include, where applicable:
- Score
- Priority
- Reason
- Recommended Action

The engine covers:
- Reorder Recommendation
- Sales Trend & Demand Forecast
- Expiry Risk
- Slow-Moving & Dead Stock
- Overstock Risk
- Stockout Risk

The system MUST NOT present a recommendation as unexplained fact.

## Operational Standards

### X. Security & Permissions

The system MUST provide role-based permissions for:
- Admin
- Pharmacist
- Cashier

Sensitive actions MUST respect permissions and be recorded in the Audit Log.

### XI. Backup & Recovery

Backup and recovery are core functionality.

The system MUST support:
- Automatic backups
- Multiple backup versions
- User-selected backup location
- Restore
- Protection against unexpected shutdowns

Business data MUST NOT be automatically deleted as part of routine cleanup.

### XII. Licensing

Every pharmacy installation MUST have its own license.

The product MUST prevent unauthorized copying to another pharmacy while
providing a controlled reactivation mechanism for legitimate hardware changes.

### XIII. Product Architecture

There MUST be one maintained Core Product.

Customer-specific Pharmacy name, Logo, Configuration, and License MUST NOT
require modifying the Core code.

Customer-specific features MUST be isolated and MUST NOT compromise the
Core Product.

## Development Standards

### XIV. Simplicity & Performance

The user experience MUST prioritize:
- Fast sales
- Minimal clicks
- Keyboard efficiency
- Clear Arabic UI
- Simple workflows
- Low resource consumption

### XV. Testing & Definition of Done

A feature is not complete when the code compiles.

A feature is complete only when:
- Requirements are satisfied
- Acceptance Criteria pass
- Relevant tests pass
- Error/edge cases are handled
- Data integrity is verified
- Performance is acceptable
- Documentation is updated when necessary

### XVI. Agent Governance

AI coding agents MUST NOT invent business requirements.

Workflow: Requirement → Specification → Plan → Tasks → Implementation →
Tests → Review

Agent responsibilities:

**Claude Opus 4.6** — Architecture, complex technical decisions, difficult
implementation, code review, refactoring.

**Sonnet** — Medium-complexity implementation tasks.

**Haiku** — Simple implementation and maintenance tasks.

Agents MUST follow the Constitution and existing specifications.

### XVII. Scope Control

Every proposed feature MUST be classified as:
- Must Have
- Should Have
- Could Have
- Future

Features outside the approved scope require explicit product approval.

### XVIII. Learning Principle

The project MUST remain understandable to the Product Owner/Developer.

Major technical decisions MUST include:
- Why the decision was made
- Alternatives considered
- Trade-offs
- Impact on future development

The goal is to build the product while developing the owner's ability to
plan and build future software products.

## Governance

This Constitution is the authoritative governance document for the Pharmacy
Management System. All development, review, and operational decisions MUST
comply with its principles.

**Amendment Procedure:**
1. Proposed amendments MUST be documented with rationale before adoption.
2. Amendments MUST NOT weaken Data Integrity (VI), Financial Accuracy (VIII),
   or Offline-First (II) guarantees without explicit justification.
3. Every amendment MUST update the version number per semantic versioning:
   - MAJOR: Principle removal, redefinition, or backward-incompatible
     governance change.
   - MINOR: New principle added or existing principle materially expanded.
   - PATCH: Clarification, wording, or non-semantic refinement.

**Compliance Review:**
All code changes, specifications, and plans MUST be verifiable against this
Constitution. Non-compliance MUST be flagged during review.

**Version**: 1.0.0 | **Ratified**: 2026-10-07 | **Last Amended**: 2026-10-07
