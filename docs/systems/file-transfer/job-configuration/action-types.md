# Action Types

This document describes all 16 action types available in FTS job configurations.

## Action Type Values

**Important**: Use the exact values shown in the "Value" column in your configurations.

| Category | Value | Description |
|----------|-------|-------------|
| **File I/O** | `ReadFile` | Read file from source |
| | `WriteFile` | Write file to destination |
| | `DeleteFile` | Delete file from source |
| | `RenameFile` | Rename file on source/destination |
| **File Lists** | `GetFileList` | Get list of files matching pattern |
| | `ZipFileList` | Create ZIP from file list |
| | `ConcatenateList` | Concatenate files in list |
| | `DeleteFileList` | Delete files in list |
| **Encryption** | `PGP-Decrypt` | PGP decryption |
| | `PGP-Encrypt` | PGP encryption |
| | `RSA-Encrypt` | RSA encryption |
| **Processing** | `Scan` | ClamAV virus scanning |
| | `Encode` | Character encoding conversion |
| **Events** | `FireEvent` | Fire single CloudEvent |
| | `FireEventList` | Fire events for file list |
| | `Notification` | Send notification to APIM |

---

## ReadFile

Reads a file from the configured source.

### Configuration

```json
{
  "id": 1,
  "type": "ReadFile",
  "sourceFile": "{@JobSettings.source.path}/{@Message.fileName}"
}
```

### Fields

| Field | Required | Description |
|-------|----------|-------------|
| `id` | Yes | Action identifier |
| `type` | Yes | `"ReadFile"` |
| `sourceFile` | Yes | Path to file (supports placeholders) |

### Output
File content available as `{@JobSettings.actions.{id}.outputStream}`

---

## WriteFile

Writes file content to a destination.

### Configuration

```json
{
  "id": 3,
  "type": "WriteFile",
  "sourceStream": "{@JobSettings.actions.2.outputStream}",
  "outputFile": "{@JobSettings.destinations.1.path}/{@Message.fileName}"
}
```

### Fields

| Field | Required | Description |
|-------|----------|-------------|
| `id` | Yes | Action identifier |
| `type` | Yes | `"WriteFile"` |
| `sourceStream` | Yes | Reference to previous action output |
| `outputFile` | Yes | Destination path (must reference destination) |

---

## DeleteFile

Deletes a file from the source.

### Configuration

```json
{
  "id": 4,
  "type": "DeleteFile",
  "sourceFile": "{@JobSettings.source.path}/{@Message.fileName}"
}
```

### Fields

| Field | Required | Description |
|-------|----------|-------------|
| `id` | Yes | Action identifier |
| `type` | Yes | `"DeleteFile"` |
| `sourceFile` | Yes | Path to file to delete |

---

## RenameFile

Renames a file on source or destination.

### Configuration (Source)

```json
{
  "id": 5,
  "type": "RenameFile",
  "sourceFile": "{@JobSettings.source.path}/{@Message.fileName}",
  "outputFile": "{@JobSettings.source.path}/{@Message.fileName}.processed"
}
```

### Configuration (Destination)

```json
{
  "id": 5,
  "type": "RenameFile",
  "sourceFile": "{@JobSettings.destinations.1.path}/{@Message.fileName}",
  "outputFile": "{@JobSettings.destinations.1.path}/{@Message.fileName}.done"
}
```

### Fields

| Field | Required | Description |
|-------|----------|-------------|
| `id` | Yes | Action identifier |
| `type` | Yes | `"RenameFile"` |
| `sourceFile` | Yes | Current file path |
| `outputFile` | Yes | New file path |

---

## GetFileList

Gets a list of files matching a pattern.

### Configuration

```json
{
  "id": 1,
  "type": "GetFileList",
  "sourceFile": "{@JobSettings.source.path}/*.csv",
  "startWith": "report_",
  "endWith": ".csv",
  "numberOfFiles": 100,
  "sort": "name"
}
```

### Fields

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `id` | Yes | | Action identifier |
| `type` | Yes | | `"GetFileList"` |
| `sourceFile` | Yes | | Path pattern with wildcards |
| `startWith` | No | | Filter: filename prefix |
| `endWith` | No | | Filter: filename suffix |
| `numberOfFiles` | No | 1 | Maximum files to return |
| `sort` | No | | Sort order |

### Output
`listFileNames` populated with matching files.

---

## ZipFileList

Creates a ZIP archive from files in list.

### Configuration

```json
{
  "id": 3,
  "type": "ZipFileList",
  "sourceStream": "{@JobSettings.actions.1.outputStream}",
  "outputFile": "{@JobSettings.source.path}/archive.zip"
}
```

### Fields

| Field | Required | Description |
|-------|----------|-------------|
| `id` | Yes | Action identifier |
| `type` | Yes | `"ZipFileList"` |
| `sourceStream` | Yes | Reference to GetFileList action (provides `listFileNames`) |
| `outputFile` | Yes | Output ZIP file path (writes directly to disk) |

### Behavior

> **Important**: ZipFileList writes the ZIP archive **directly to disk** at the specified `outputFile` path. It does NOT populate `outputStream`. To transfer the created ZIP to a remote destination, you must:
> 1. Use ZipFileList to create the archive locally (on the source system)
> 2. Use ReadFile to read the created ZIP file
> 3. Use WriteFile to upload to the remote destination

### Example: Create ZIP and Upload to Blob

```json
{
  "actions": [
    { "id": 1, "type": "GetFileList", "sourceFile": "{@JobSettings.source.path}/*.csv" },
    { "id": 2, "type": "ZipFileList", "sourceStream": "{@JobSettings.actions.1.outputStream}", "outputFile": "{@JobSettings.source.path}/temp.zip" },
    { "id": 3, "type": "ReadFile", "sourceFile": "{@JobSettings.source.path}/temp.zip" },
    { "id": 4, "type": "WriteFile", "sourceStream": "{@JobSettings.actions.3.outputStream}", "outputFile": "{@JobSettings.destinations.1.path}/archive.zip" },
    { "id": 5, "type": "DeleteFile", "sourceFile": "{@JobSettings.source.path}/temp.zip" }
  ]
}
```

---

## ConcatenateList

Concatenates files in list into single file(s).

### Configuration

```json
{
  "id": 2,
  "type": "ConcatenateList",
  "sourceStream": "{@JobSettings.actions.1.outputStream}",
  "outputFile": "{@JobSettings.source.path}/consolidated.csv",
  "concateWith": "\n",
  "numberOfFiles": 1000,
  "startWith": "",
  "endWith": ""
}
```

### Fields

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `id` | Yes | | Action identifier |
| `type` | Yes | | `"ConcatenateList"` |
| `sourceStream` | Yes | | Reference to GetFileList action |
| `outputFile` | Yes | | Output file path (writes directly to disk) |
| `concateWith` | No | `""` | Separator between file contents |
| `numberOfFiles` | No | 1 | Max files per output (splits if exceeded) |
| `startWith` | No | `""` | Content to prepend |
| `endWith` | No | `""` | Content to append |

### Output

- Creates concatenated file(s) on disk at `outputFile`
- Populates `listConcateFileNames` with output file paths
- If `numberOfFiles` causes splits, files are named: `{name}_0.ext`, `{name}_1.ext`, etc.

### Behavior

> **Note**: ConcatenateList writes directly to disk. The `listConcateFileNames` can be used by FireEventList to fire events for each concatenated file.

---

## DeleteFileList

Deletes all files in list.

### Configuration

```json
{
  "id": 4,
  "type": "DeleteFileList",
  "sourceStream": "{@JobSettings.actions.1.outputStream}"
}
```

### Fields

| Field | Required | Description |
|-------|----------|-------------|
| `id` | Yes | Action identifier |
| `type` | Yes | `"DeleteFileList"` |
| `sourceStream` | Yes | Reference to GetFileList action |

---

## PGP-Decrypt

Decrypts PGP-encrypted content.

### Configuration

```json
{
  "id": 1,
  "type": "PGP-Decrypt",
  "sourceStream": "{@JobSettings.actions.1.outputStream}",
  "keyFile": "{@AppSettings.jobsettingsfolder}/private.key",
  "keypassword": "{@JobSecrets.pgpPassword}"
}
```

### Fields

| Field | Required | Description |
|-------|----------|-------------|
| `id` | Yes | Action identifier |
| `type` | Yes | `"PGP-Decrypt"` |
| `sourceStream` | Yes | Reference to encrypted content |
| `keyFile` | Yes | Path to PGP private key |
| `keypassword` | No | Key passphrase (from secrets) |

---

## PGP-Encrypt

Encrypts content using PGP.

### Configuration

```json
{
  "id": 1,
  "type": "PGP-Encrypt",
  "sourceStream": "{@JobSettings.actions.1.outputStream}",
  "keyFile": "{@AppSettings.jobsettingsfolder}/public.key",
  "encryptArmor": true,
  "encryptIntegirity": true,
  "encryptSymmetricKey": "Cast5",
  "encryptCompressed": "Uncompressed",
  "encryptFilename": "{@Message.fileName.name}.pgp",
  "encryptSubkey": false
}
```

### Fields

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `id` | Yes | | Action identifier |
| `type` | Yes | | `"PGP-Encrypt"` |
| `sourceStream` | Yes | | Reference to content to encrypt |
| `keyFile` | Yes | | Path to PGP public key |
| `encryptArmor` | No | `true` | ASCII armor output |
| `encryptIntegirity` | No | `true` | Include integrity check |
| `encryptSymmetricKey` | No | `"Cast5"` | Symmetric algorithm |
| `encryptCompressed` | No | `"Uncompressed"` | Compression algorithm |
| `encryptFilename` | No | `""` | Filename in encrypted packet |
| `encryptSubkey` | No | `false` | Use encryption subkey |

### Symmetric Key Algorithms
`Cast5`, `Aes128`, `Aes192`, `Aes256`, `Twofish`, `Blowfish`, `TripleDes`

### Compression Algorithms
`Uncompressed`, `Zip`, `ZLib`, `BZip2`

---

## RSA-Encrypt

Encrypts content using RSA.

### Configuration

```json
{
  "id": 1,
  "type": "RSA-Encrypt",
  "sourceStream": "{@JobSettings.actions.1.outputStream}",
  "keyFile": "{@AppSettings.jobsettingsfolder}/rsa_public.pem"
}
```

### Fields

| Field | Required | Description |
|-------|----------|-------------|
| `id` | Yes | Action identifier |
| `type` | Yes | `"RSA-Encrypt"` |
| `sourceStream` | Yes | Reference to content to encrypt |
| `keyFile` | Yes | Path to RSA public key |

---

## Scan

Performs ClamAV virus scanning.

### Configuration

```json
{
  "id": 2,
  "type": "Scan",
  "sourceStream": "{@JobSettings.actions.1.outputStream}"
}
```

### Fields

| Field | Required | Description |
|-------|----------|-------------|
| `id` | Yes | Action identifier |
| `type` | Yes | `"Scan"` |
| `sourceStream` | Yes | Reference to content to scan |

### Behavior
- If clean: Passes content through unchanged
- If infected: Action fails with virus details in error message

---

## Encode

Converts character encoding.

### Configuration

```json
{
  "id": 2,
  "type": "Encode",
  "sourceStream": "{@JobSettings.actions.1.outputStream}",
  "source": "UTF-8",
  "target": "ASCII"
}
```

### Fields

| Field | Required | Description |
|-------|----------|-------------|
| `id` | Yes | Action identifier |
| `type` | Yes | `"Encode"` |
| `sourceStream` | Yes | Reference to content to encode |
| `source` | Yes | Source encoding |
| `target` | Yes | Target encoding |

### Supported Encodings
`TEXT`, `UNICODE`, `ASCII`, `UTF-8`, `UTF-7`, `UTF-32`

---

## FireEvent

Fires a CloudEvent notification.

### Configuration

```json
{
  "id": 5,
  "type": "FireEvent",
  "sourceStream": "{@JobSettings.actions.3.outputStream}",
  "outputFile": "{@JobSettings.destinations.1.path}/{@Message.fileName}",
  "eventType": "Succeeded",
  "eventSource": "FTS.JobHandler",
  "eventSubject": "File Ready",
  "eventAccountId": "{@Message.accountID}",
  "eventId": "",
  "eventCorrelationId": "{@Message.correlationId}",
  "eventFileName": "{@Message.fileName}"
}
```

### Fields

| Field | Required | Description |
|-------|----------|-------------|
| `id` | Yes | Action identifier |
| `type` | Yes | `"FireEvent"` |
| `sourceStream` | Yes | Reference to file content |
| `outputFile` | Yes | File path for event |
| `eventType` | Yes | Event type to emit |
| `eventSource` | Yes | Event source identifier |
| `eventSubject` | Yes | Event subject |
| `eventAccountId` | Yes | Target account ID |
| `eventId` | No | Event ID (auto-generated if empty) |
| `eventCorrelationId` | No | Correlation ID |
| `eventFileName` | No | Filename for event |

---

## FireEventList

Fires events for each file in a list.

### Configuration

```json
{
  "id": 4,
  "type": "FireEventList",
  "sourceStream": "{@JobSettings.actions.1.outputStream}",
  "outputFile": "{@JobSettings.destinations.1.path}/{@Message.fileList}",
  "eventType": "Succeeded",
  "eventSource": "FTS.BatchProcessor",
  "eventSubject": "Batch File Ready",
  "eventAccountId": "{@Message.accountID}"
}
```

### Fields
Same as FireEvent, but fires one event per file in the list.

---

## Notification

Sends notification to APIM endpoint.

> **Important**: The `outputFile` must reference an `Internal-APIM` destination. The Notification action only executes for destinations with type `Internal-APIM`.

### Configuration

```json
{
  "id": 4,
  "type": "Notification",
  "sourceFile": "{@JobSettings.destinations.1.path}/{@Message.fileName}",
  "outputFile": "{@JobSettings.destinations.2.path}",
  "eventType": "Succeeded",
  "eventAccountId": "{@Message.accountID}",
  "eventCorrelationId": "{@Message.correlationId}"
}
```

Where destination 2 is configured as:
```json
{
  "id": 2,
  "type": "Internal-APIM",
  "TargetUrl": "https://api.example.com/notify",
  "TimeoutMs": "30000"
}
```

### Fields

| Field | Required | Description |
|-------|----------|-------------|
| `id` | Yes | Action identifier |
| `type` | Yes | `"Notification"` |
| `sourceFile` | Yes | Source file reference |
| `outputFile` | Yes | Must reference an `Internal-APIM` destination |
| `eventType` | No | Event type for notification |
| `eventAccountId` | Yes | Account identifier |
| `eventCorrelationId` | No | Correlation ID |

### Behavior
Sends CloudEvent-formatted notification to configured APIM endpoint.

---

## Action Chaining Reference

```
Action 1 (ReadFile)
    └── outputStream ──> Action 2 (PGP-Decrypt)
                              └── outputStream ──> Action 3 (Scan)
                                                        └── outputStream ──> Action 4 (WriteFile)
```

Reference pattern: `{@JobSettings.actions.{n}.outputStream}` (1-based indexing)

---

## Action Support Matrix by Source Type

Not all actions are supported by all source types. Below is the compatibility matrix:

| Action | Internal-SFTP | External-SFTP | Vendor-SFTP | Vendor-FTP | BlobStore | GCP | AmazonS3 | Internal-APIM | AzureFile-API |
|--------|:-------------:|:-------------:|:-----------:|:----------:|:---------:|:---:|:--------:|:-------------:|:-------------:|
| ReadFile | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| WriteFile | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | - | ✓ | ✓ |
| DeleteFile | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | - | - | ✓ |
| RenameFile | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | - | - | - |
| GetFileList | ✓ | ✓ | - | - | - | - | - | - | - |
| DeleteFileList | ✓ | ✓ | - | - | - | - | - | - | - |

**Notes:**
- `✓` = Supported
- `-` = Not supported
- Actions not listed (PGP-Decrypt, PGP-Encrypt, RSA-Encrypt, Scan, Encode, FireEvent, FireEventList, Notification) operate on in-memory streams and are source-type independent

## Related Documentation

- [Job Configuration Specification](./specification.md)
- [Placeholders Reference](./placeholders-reference.md)
- [Job Examples](../job-examples/README.md)
