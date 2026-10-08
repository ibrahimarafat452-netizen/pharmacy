# Contract: Backup Service

**Module**: Backup & Recovery (Phase 10)
**Plan**: [../plan.md](../plan.md) | **Data Model**: [../data-model.md](../data-model.md)

---

## IBackupService

Manages database backup, restore, and integrity verification. Permission: Admin.

| Method | Input | Output | Notes |
|--------|-------|--------|-------|
| CreateBackup | — | BackupRecord | Uses SQLite online backup API. Verifies integrity after backup. Records in BackupRecord table. Runs rotation (delete oldest beyond retention count). Does NOT interrupt normal operation |
| VerifyBackup | filePath | bool | Opens backup file, runs PRAGMA integrity_check |
| RestoreFromBackup | filePath, userId | void | (1) Create safety backup of current DB. (2) Close all connections. (3) Copy backup over current DB. (4) Reopen and verify integrity. Audit log entry after restore |
| ListBackups | — | List\<BackupRecord\> | Ordered by date DESC |
| GetLastBackupStatus | — | BackupRecord? | Most recent backup with integrity status |
| StartScheduler | intervalMinutes | void | Starts background timer for automatic backups |
| StopScheduler | — | void | Stops automatic backup timer |
| ValidateBackupLocation | path | LocationValidation | Checks path exists, is writable, has sufficient space |

**Rotation rules**: After successful backup, count backups at configured location. If count > retention, delete oldest files. ONLY backup files are deleted — never business data (Constitution XI).

**Crash recovery**: Not in this service — SQLite WAL handles it automatically on database open. Application startup runs `PRAGMA integrity_check`; if it fails, prompts Admin to restore from latest valid backup.
