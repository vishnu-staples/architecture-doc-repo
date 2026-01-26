# Job Configuration Specification

This document provides the complete specification for FTS job configuration files (`jobsettings.json`).

## File Structure

Job configurations are stored in the Azure File Share at:
```
/mnt/jobconfig/{source}/{accountID}/{fileName.subFolder}/jobsettings.json
```

## JSON Schema

```json
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["feedName", "source", "destinations", "actions"],
  "properties": {
    "feedName": {
      "type": "string",
      "description": "Unique identifier for this job configuration"
    },
    "version": {
      "type": "string",
      "description": "Configuration version (optional)"
    },
    "jobsecret": {
      "type": "string",
      "description": "Path to secrets file (optional)"
    },
    "source": {
      "$ref": "#/definitions/serversAttributes"
    },
    "destinations": {
      "type": "array",
      "items": {
        "$ref": "#/definitions/destination"
      },
      "minItems": 1
    },
    "actions": {
      "type": "array",
      "items": {
        "$ref": "#/definitions/action"
      },
      "minItems": 1
    }
  }
}
```

## Configuration Sections

### Root Level

```json
{
  "feedName": "my-data-feed",
  "version": "2.0",
  "source": { ... },
  "destinations": [ ... ],
  "actions": [ ... ]
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `feedName` | Yes | Unique job identifier for logging and troubleshooting |
| `version` | No | Configuration version for change tracking |
| `jobsecret` | No | Reference to external secrets file |
| `source` | Yes | Source server configuration |
| `destinations` | Yes | Array of destination configurations |
| `actions` | Yes | Array of actions to execute in order |

### Source Configuration

The `source` object defines where files are read from.

```json
{
  "source": {
    "name": "Vendor SFTP Server",
    "type": "Vendor-SFTP",
    "host": "sftp.vendor.com",
    "port": "22",
    "path": "/outbound",
    "ssl": false,
    "credentials": {
      "type": "BasicAuth",
      "username": "{@JobSecrets.vendorUser}",
      "password": "{@JobSecrets.vendorPass}"
    }
  }
}
```

See [Source/Destination Types](./source-destination-types.md) for complete field reference.

### Destinations Configuration

The `destinations` array defines where files can be written.

```json
{
  "destinations": [
    {
      "id": 1,
      "name": "Azure Blob Storage",
      "type": "BlobStore",
      "endpoint": "https://storage.blob.core.windows.net",
      "credentials": {
        "containerName": "incoming",
        "connectionString": "{@JobSecrets.blobConnection}"
      }
    },
    {
      "id": 1,
      "name": "Internal SFTP",
      "type": "Internal-SFTP",
      "path": "{@AppSettings.sftpintroot}/{@Message.accountID}/processed"
    }
  ]
}
```

**Important**: The `id` field must be unique and is used by actions to reference destinations.

### Actions Configuration

The `actions` array defines operations to perform in sequence.

```json
{
  "actions": [
    {
      "id": 1,
      "type": "ReadFile",
      "sourceFile": "{@JobSettings.source.path}/{@Message.fileName}"
    },
    {
      "id": 1,
      "type": "Scan",
      "sourceStream": "{@JobSettings.actions.1.outputStream}"
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

See [Action Types](./action-types.md) for complete action reference.

## Action Chaining

Actions are linked through stream references:

1. **ReadFile** outputs to `actions.{id}.outputStream`
2. Subsequent actions reference via `sourceStream: "{@JobSettings.actions.1.outputStream}"`
3. Data flows through the action pipeline

```
ReadFile (id=0) ──> Scan (id=1) ──> WriteFile (id=2)
     │                   │
     └── outputStream ───┘── sourceStream
```

## Validation Rules

### Required Fields

| Section | Required Fields |
|---------|-----------------|
| Root | `feedName`, `source`, `destinations`, `actions` |
| Source | `type` |
| Destination | `id`, `type` |
| Action | `id`, `type` |

### Action-Specific Requirements

| Action Type | Required Fields |
|-------------|-----------------|
| ReadFile | `sourceFile` |
| WriteFile | `sourceStream`, `outputFile` |
| DeleteFile | `sourceFile` |
| PGP-Decrypt | `sourceStream`, `keyFile` |
| PGP-Encrypt | `sourceStream`, `keyFile` |
| Scan | `sourceStream` |
| FireEvent | `sourceStream`, `outputFile`, event fields |

### Destination Reference

WriteFile `outputFile` must reference a valid destination:
```
{@JobSettings.destinations.{id}.path}/...
```

## Complete Example

```json
{
  "feedName": "vendor-invoice-import",
  "version": "1.0",
  "source": {
    "name": "Vendor SFTP",
    "type": "Vendor-SFTP",
    "host": "sftp.vendor.com",
    "port": "22",
    "path": "/outbound/invoices",
    "credentials": {
      "type": "BasicAuth",
      "username": "{@JobSecrets.vendorUser}",
      "password": "{@JobSecrets.vendorPass}"
    }
  },
  "destinations": [
    {
      "id": 1,
      "name": "Azure Blob - Raw",
      "type": "BlobStore",
      "endpoint": "https://storage.blob.core.windows.net",
      "path": "raw/invoices",
      "credentials": {
        "containerName": "invoices",
        "connectionString": "{@JobSecrets.blobConnection}"
      }
    },
    {
      "id": 1,
      "name": "Azure Blob - Processed",
      "type": "BlobStore",
      "endpoint": "https://storage.blob.core.windows.net",
      "path": "processed/invoices",
      "credentials": {
        "containerName": "invoices",
        "connectionString": "{@JobSecrets.blobConnection}"
      }
    }
  ],
  "actions": [
    {
      "id": 1,
      "type": "ReadFile",
      "sourceFile": "{@JobSettings.source.path}/{@Message.fileName}"
    },
    {
      "id": 1,
      "type": "PGP-Decrypt",
      "sourceStream": "{@JobSettings.actions.1.outputStream}",
      "keyFile": "{@AppSettings.jobsettingsfolder}/private.key",
      "keypassword": "{@JobSecrets.pgpPassword}"
    },
    {
      "id": 2,
      "type": "Scan",
      "sourceStream": "{@JobSettings.actions.1.outputStream}"
    },
    {
      "id": 3,
      "type": "WriteFile",
      "sourceStream": "{@JobSettings.actions.2.outputStream}",
      "outputFile": "{@JobSettings.destinations.1.path}/{@Message.fileName.name}.csv"
    },
    {
      "id": 4,
      "type": "DeleteFile",
      "sourceFile": "{@JobSettings.source.path}/{@Message.fileName}"
    },
    {
      "id": 5,
      "type": "FireEvent",
      "sourceStream": "{@JobSettings.actions.2.outputStream}",
      "outputFile": "{@JobSettings.destinations.1.path}/{@Message.fileName.name}.csv",
      "eventType": "Succeeded",
      "eventSource": "FTS.InvoiceProcessor",
      "eventSubject": "Invoice Ready",
      "eventAccountId": "{@Message.accountID}"
    }
  ]
}
```

## Secrets File (jobsecrets.json)

Store sensitive credentials separately:

```json
{
  "vendorUser": "actual-username",
  "vendorPass": "actual-password",
  "pgpPassword": "key-passphrase",
  "blobConnection": "DefaultEndpointsProtocol=https;AccountName=..."
}
```

Referenced via `{@JobSecrets.keyName}` placeholders.

## Best Practices

1. **Use Meaningful Names**: Set descriptive `feedName` for easy identification
2. **Secure Credentials**: Always use `{@JobSecrets.xxx}` for sensitive data
3. **Sequential IDs**: Use sequential action IDs (0, 1, 2, ...) for clarity
4. **Delete After Success**: Include DeleteFile as final action when appropriate
5. **Fire Events**: Use FireEvent to notify downstream systems
6. **Version Control**: Track configuration changes with `version` field

## Related Documentation

- [Source/Destination Types](./source-destination-types.md)
- [Action Types](./action-types.md)
- [Placeholders Reference](./placeholders-reference.md)
- [Jobsettings Storage](./jobsettings-storage.md)
- [Job Examples](../job-examples/README.md)
