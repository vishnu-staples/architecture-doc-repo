# Placeholders Reference

FTS job configurations support dynamic placeholders that are resolved at runtime. This document describes all available placeholder patterns.

## Placeholder Syntax

Placeholders follow the pattern: `{@Category.property}` or `{@Category.property.subProperty}`

## Message Placeholders

Values from the incoming CloudEvent notification.

| Placeholder | Description | Example Value |
|-------------|-------------|---------------|
| `{@Message.fileName}` | Full file path from event | `incoming/data/file.csv` |
| `{@Message.targetFileName}` | Target file path | `processed/file.csv` |
| `{@Message.source}` | Event source | `FTP.File.ExternalService` |
| `{@Message.accountID}` | Account identifier | `ACCOUNT001` |
| `{@Message.correlationId}` | Correlation ID | `corr-123456` |
| `{@Message.fileList}` | File list (for batch operations) | (array) |

### Filename Components

| Placeholder | Description | Example |
|-------------|-------------|---------|
| `{@Message.fileName.name}` | Filename without extension | `report` |
| `{@Message.fileName.extension}` | File extension | `csv` |
| `{@Message.fileName.path}` | Directory path | `incoming/data` |
| `{@Message.fileName.subFolder}` | Subfolder path | `data` |
| `{@Message.fileName.firstSubFolder}` | First subfolder | `incoming` |

### Example

For `{@Message.fileName}` = `incoming/reports/daily_report.csv`:

| Placeholder | Resolved Value |
|-------------|----------------|
| `{@Message.fileName}` | `incoming/reports/daily_report.csv` |
| `{@Message.fileName.name}` | `daily_report` |
| `{@Message.fileName.extension}` | `csv` |
| `{@Message.fileName.path}` | `incoming/reports` |
| `{@Message.fileName.subFolder}` | `reports` |

---

## AppSettings Placeholders

Values from application configuration (`appsettings.json`).

| Placeholder | Description | Example Value |
|-------------|-------------|---------------|
| `{@AppSettings.sftpintroot}` | Internal SFTP mount | `/mnt/sftpintroot` |
| `{@AppSettings.sftpextroot}` | External SFTP mount | `/mnt/sftpextroot` |
| `{@AppSettings.sftpintfolder}` | Internal folder (with account) | `/mnt/sftpintroot/ACCOUNT001` |
| `{@AppSettings.sftpextfolder}` | External folder (with account) | `/mnt/sftpextroot/ACCOUNT001` |
| `{@AppSettings.sftpintfile}` | Internal file path | `/mnt/sftpintroot/ACCOUNT001/file.csv` |
| `{@AppSettings.sftpextfile}` | External file path | `/mnt/sftpextroot/ACCOUNT001/file.csv` |
| `{@AppSettings.jobsettingsfolder}` | Job config folder | `/mnt/jobconfig/FTP.File.ExternalService/ACCOUNT001/data` |

### AppSettings Resolution

AppSettings placeholders can be nested:

```json
{
  "AzureFiles": {
    "sftpintroot": "/mnt/sftpintroot",
    "sftpintfolder": "{@AppSettings.sftpintroot}/{@Message.accountID}",
    "jobsettingsfolder": "/mnt/jobconfig/{@Message.source}/{@Message.accountID}/{@Message.fileName.subFolder}"
  }
}
```

---

## JobSettings Placeholders

References to values within the job configuration itself.

### Source References

| Placeholder | Description |
|-------------|-------------|
| `{@JobSettings.source.path}` | Source server path |
| `{@JobSettings.source.host}` | Source server hostname |
| `{@JobSettings.source.type}` | Source server type |

### Destination References

| Placeholder | Description |
|-------------|-------------|
| `{@JobSettings.destinations.{n}.path}` | Destination path by position (1-based) |
| `{@JobSettings.destinations.{n}.host}` | Destination hostname by position (1-based) |
| `{@JobSettings.destinations.{n}.type}` | Destination type by position (1-based) |

> **Important**: Destination indices are **1-based**. The first destination is referenced as `destinations.1`, not `destinations.0`.

### Action References

| Placeholder | Description |
|-------------|-------------|
| `{@JobSettings.actions.{n}.outputStream}` | Action output stream (1-based) |
| `{@JobSettings.action.{n}.filename}` | Action output filename (1-based) |

> **Important**: Action indices are **1-based**. The first action is referenced as `actions.1`, not `actions.0`.

### Example

```json
{
  "actions": [
    {
      "id": 1,
      "type": "ReadFile",
      "sourceFile": "{@JobSettings.source.path}/{@Message.fileName}"
    },
    {
      "id": 2,
      "type": "WriteFile",
      "sourceStream": "{@JobSettings.actions.1.outputStream}",
      "outputFile": "{@JobSettings.destinations.1.path}/{@Message.fileName}"
    }
  ]
}
```

---

## JobSecrets Placeholders

References to values in `jobsecrets.json`.

### Syntax
`{@JobSecrets.keyName}`

### Pattern
Regex: `{@JobSecrets.([a-zA-Z0-9]*)}`

### Example

**jobsettings.json:**
```json
{
  "source": {
    "credentials": {
      "username": "{@JobSecrets.vendorUser}",
      "password": "{@JobSecrets.vendorPass}"
    }
  }
}
```

**jobsecrets.json:**
```json
{
  "vendorUser": "actual_username",
  "vendorPass": "actual_password"
}
```

---

## DateTime Placeholders

Dynamic date/time values.

### Current DateTime

| Placeholder | Description | Example |
|-------------|-------------|---------|
| `{@DateTimeNow:yyyy}` | Current year | `2024` |
| `{@DateTimeNow:MM}` | Current month | `01` |
| `{@DateTimeNow:dd}` | Current day | `15` |
| `{@DateTimeNow:HH}` | Current hour (24h) | `14` |
| `{@DateTimeNow:mm}` | Current minute | `30` |
| `{@DateTimeNow:ss}` | Current second | `45` |
| `{@DateTimeNow:yyyyMMdd}` | Date stamp | `20240115` |
| `{@DateTimeNow:yyyyMMddHHmmss}` | Full timestamp | `20240115143045` |

### File DateTime

Based on file modification time:

| Placeholder | Description |
|-------------|-------------|
| `{@FileDateTime:yyyy}` | File year |
| `{@FileDateTime:MM}` | File month |
| `{@FileDateTime:dd}` | File day |
| `{@FileDateTime:yyyyMMdd}` | File date stamp |

### Example

```json
{
  "outputFile": "{@JobSettings.destinations.1.path}/archive_{@DateTimeNow:yyyyMMdd}_{@Message.fileName}"
}
```

Result: `archive_20240115_report.csv`

---

## Placeholder Resolution Order

1. **JobSecrets**: `{@JobSecrets.xxx}` resolved from jobsecrets.json
2. **AppSettings**: `{@AppSettings.xxx}` resolved from appsettings.json
3. **Message**: `{@Message.xxx}` resolved from CloudEvent
4. **JobSettings**: `{@JobSettings.xxx}` resolved from jobsettings.json
5. **DateTime**: `{@DateTimeNow:xxx}` and `{@FileDateTime:xxx}` resolved

## Nested Resolution

Placeholders can be nested:

```json
{
  "sftpintfolder": "{@AppSettings.sftpintroot}/{@Message.accountID}"
}
```

Resolution:
1. `{@AppSettings.sftpintroot}` → `/mnt/sftpintroot`
2. `{@Message.accountID}` → `ACCOUNT001`
3. Result: `/mnt/sftpintroot/ACCOUNT001`

---

## Common Patterns

### Standard File Paths

```json
// Read from source
"sourceFile": "{@JobSettings.source.path}/{@Message.fileName}"

// Write to destination (1-based index)
"outputFile": "{@JobSettings.destinations.1.path}/{@Message.fileName}"

// Write with new name
"outputFile": "{@JobSettings.destinations.1.path}/{@Message.fileName.name}_processed.{@Message.fileName.extension}"
```

### Dated Archive

```json
"outputFile": "{@JobSettings.destinations.1.path}/archive/{@DateTimeNow:yyyy}/{@DateTimeNow:MM}/{@Message.fileName}"
```

### Secrets in Credentials

```json
"credentials": {
  "username": "{@JobSecrets.user}",
  "password": "{@JobSecrets.pass}"
}
```

### Key File Paths

```json
"keyFile": "{@AppSettings.jobsettingsfolder}/keys/private.key",
"keypassword": "{@JobSecrets.keyPassword}"
```

---

## Troubleshooting

### Placeholder Not Resolved

**Symptom**: Literal `{@xxx}` appears in output

**Causes**:
1. Typo in placeholder name
2. Missing value in source (jobsecrets, appsettings)
3. Incorrect placeholder category

### Resolution Order Issues

**Symptom**: Unexpected value substitution

**Solution**: Ensure proper nesting order - outer placeholders resolve first

### File Not Found

**Symptom**: `{@AppSettings.jobsettingsfolder}` resolves to wrong path

**Cause**: Message fields don't match expected folder structure

**Solution**: Verify `{@Message.source}`, `{@Message.accountID}`, `{@Message.fileName.subFolder}`

## Related Documentation

- [Job Configuration Specification](./specification.md)
- [Jobsettings Storage](./jobsettings-storage.md)
- [Security & Secrets](../operations/security-secrets.md)
