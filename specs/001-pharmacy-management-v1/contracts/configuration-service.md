# Contract: Configuration Service

**Module**: System Configuration (Phase 10)
**Plan**: [../plan.md](../plan.md) | **Data Model**: [../data-model.md](../data-model.md)

---

## IConfigurationService

Manages pharmacy-wide configuration. Permission: Admin for mutations, all services for read.

| Method | Input | Output | Notes |
|--------|-------|--------|-------|
| GetValue | key | string? | Returns null if not set |
| GetValueOrDefault | key, defaultValue | string | |
| SetValue | key, value, userId | void | Stores in PharmacyConfig table. Creates audit log entry for config changes (FR-036) |
| GetPharmacyIdentity | — | PharmacyIdentity | Name, logo path, address, phone |
| SetPharmacyIdentity | name, logoPath?, address?, phone?, userId | void | Audit logged |
| GetReceiptSettings | — | ReceiptSettings | Paper width, content preferences, printer name |
| SetReceiptSettings | settings, userId | void | Audit logged |
| GetBackupSettings | — | BackupSettings | Interval, location, retention count |
| SetBackupSettings | settings, userId | void | Audit logged |
| GetRecommendationThresholds | — | RecommendationThresholds | Safety stock days, lead times, overstock limit, etc. |
| SetRecommendationThresholds | thresholds, userId | void | Audit logged |
| GetLanSettings | — | LanSettings | Mode (single/server/client), server address, pharmacy secret |
| SetLanSettings | settings, userId | void | Audit logged |

---

## Data Structures

**PharmacyIdentity**: name, logoPath, address, phone

**ReceiptSettings**: paperWidthMm (58 or 80), printerName, showLogo, showAddress, showPhone

**BackupSettings**: intervalMinutes, locationPath, retentionCount

**RecommendationThresholds**: safetyStockDays, leadTimeDays, overstockLimitMonths, slowMovingDays, expiryWarningDays

**LanSettings**: mode (single/server/client), serverAddress, pharmacySecret
