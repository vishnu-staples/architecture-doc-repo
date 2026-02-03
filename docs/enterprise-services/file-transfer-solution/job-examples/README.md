# Job Examples

This directory contains example job configurations demonstrating common FTS use cases.

## Important: 1-Based Indexing

**All action and destination indices use 1-based indexing.** When referencing:
- Actions: Use `{@JobSettings.actions.1.outputStream}` for the first action
- Destinations: Use `{@JobSettings.destinations.1.path}` for the first destination

This applies to both the `id` field in configurations and placeholder references.

## Coverage Matrix

### Source Type Coverage

| Example | Source Type |
|---------|-------------|
| 1. sftp-to-sftp-basic | Internal-SFTP |
| 2. sftp-to-blob-pgp-decrypt | Vendor-SFTP |
| 3. blob-to-vendor-ftp | BlobStore |
| 4. gcp-to-azure-encrypted | GCP |
| 5. s3-to-sftp-virus-scan | AmazonS3 |
| 6. multi-file-processing | Internal-SFTP |
| 7. fire-event-chain | Internal-SFTP |
| 8. apim-notification | BlobStore |
| 9. internal-sftp-azurefile-api | Internal-SFTP |
| 10. rsa-encrypt-multi-dest | Vendor-SFTP |

### Destination Type Coverage

| Example | Destination Type(s) |
|---------|---------------------|
| 1. sftp-to-sftp-basic | External-SFTP |
| 2. sftp-to-blob-pgp-decrypt | BlobStore |
| 3. blob-to-vendor-ftp | Vendor-FTP |
| 4. gcp-to-azure-encrypted | BlobStore |
| 5. s3-to-sftp-virus-scan | Internal-SFTP |
| 6. multi-file-processing | BlobStore |
| 7. fire-event-chain | BlobStore |
| 8. apim-notification | Internal-APIM |
| 9. internal-sftp-azurefile-api | AzureFile-API |
| 10. rsa-encrypt-multi-dest | BlobStore, GCP |

### Action Type Coverage

| Action Type | Examples |
|-------------|----------|
| ReadFile | 1, 2, 3, 4, 5, 6, 8, 9, 10 |
| WriteFile | 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 |
| DeleteFile | 1 |
| RenameFile | 9 |
| GetFileList | 6, 7 |
| ZipFileList | 6 |
| ConcatenateList | 6, 7 |
| DeleteFileList | 6, 7 |
| PGP-Decrypt | 2 |
| PGP-Encrypt | 4 |
| RSA-Encrypt | 10 |
| Scan | 2, 5 |
| Encode | 3 |
| FireEvent | 2, 7 |
| FireEventList | 7 |
| Notification | 8 |

## Examples

### 1. [sftp-to-sftp-basic.json](./sftp-to-sftp-basic.json)

Basic file transfer from Internal SFTP to External SFTP.

**Flow**: ReadFile → WriteFile → DeleteFile

**Use Case**: Simple internal-to-external file delivery

---

### 2. [sftp-to-blob-pgp-decrypt.json](./sftp-to-blob-pgp-decrypt.json)

Receive encrypted file from vendor, decrypt, scan, and store.

**Flow**: ReadFile → PGP-Decrypt → Scan → WriteFile → FireEvent

**Use Case**: Secure vendor file intake with virus scanning

---

### 3. [blob-to-vendor-ftp.json](./blob-to-vendor-ftp.json)

Send file from Azure Blob to vendor FTP with encoding conversion.

**Flow**: ReadFile → Encode → WriteFile

**Use Case**: Outbound file delivery with character encoding

---

### 4. [gcp-to-azure-encrypted.json](./gcp-to-azure-encrypted.json)

Transfer from Google Cloud Storage to Azure with PGP encryption.

**Flow**: ReadFile → PGP-Encrypt → WriteFile

**Use Case**: Cross-cloud transfer with encryption at rest

---

### 5. [s3-to-sftp-virus-scan.json](./s3-to-sftp-virus-scan.json)

Pull from S3, virus scan, deliver to internal SFTP.

**Flow**: ReadFile → Scan → WriteFile → DeleteFile

**Use Case**: Inbound file intake from AWS with security scanning

---

### 6. [multi-file-processing.json](./multi-file-processing.json)

Process multiple files into a ZIP archive.

**Flow**: GetFileList → ConcatenateList → ZipFileList → ReadFile → WriteFile → DeleteFileList → DeleteFile

**Use Case**: Batch file aggregation and archiving

**Note**: ZipFileList writes directly to disk, so a ReadFile action is needed before WriteFile to transfer the zip to a remote destination.

---

### 7. [fire-event-chain.json](./fire-event-chain.json)

Batch process files, consolidate, and trigger downstream events.

**Flow**: GetFileList → ConcatenateList → WriteFile → FireEvent → FireEventList → DeleteFileList

**Use Case**: Event-driven workflow orchestration with batch processing

**Note**: FireEventList requires a GetFileList or ConcatenateList source to provide the list of files.

---

### 8. [apim-notification.json](./apim-notification.json)

Transfer file and notify downstream API.

**Flow**: ReadFile → WriteFile → Notification

**Use Case**: File delivery with API callback notification

---

### 9. [internal-sftp-azurefile-api.json](./internal-sftp-azurefile-api.json)

Transfer from SFTP to Azure File Share with rename.

**Flow**: ReadFile → WriteFile → RenameFile

**Use Case**: Internal file staging with status indicators

---

### 10. [rsa-encrypt-multi-dest.json](./rsa-encrypt-multi-dest.json)

Encrypt file and write to multiple destinations.

**Flow**: ReadFile → RSA-Encrypt → WriteFile(Blob) → WriteFile(GCP)

**Use Case**: Multi-cloud delivery with encryption

---

## Usage Guide

### 1. Choose an Example

Select the example that most closely matches your use case.

### 2. Copy and Modify

1. Copy the JSON file to your job configuration folder
2. Rename to `jobsettings.json`
3. Modify values for your environment:
   - Update server hostnames and paths
   - Change credential placeholders to your secrets
   - Adjust file patterns and filters

### 3. Create Secrets File

Create `jobsecrets.json` with actual credential values:

```json
{
  "yourSecretKey": "actual-value"
}
```

### 4. Deploy Keys (if needed)

Copy encryption keys to the configuration folder:
- PGP keys: `private.key`, `public.key`
- SSH keys: `id_rsa`
- RSA keys: `rsa_public.pem`

### 5. Configure Routing

Add entry to `jobconfig` Azure Table to route events to your handler.

## Validation

Before deploying, validate your configuration:

1. **JSON Syntax**: Use a JSON validator
2. **Placeholder References**: Verify all `{@...}` placeholders exist
3. **Destination IDs**: Ensure action `outputFile` references valid destination IDs
4. **Action Chain**: Verify `sourceStream` references point to previous action outputs

## Related Documentation

- [Job Configuration Specification](../job-configuration/specification.md)
- [Source/Destination Types](../job-configuration/source-destination-types.md)
- [Action Types](../job-configuration/action-types.md)
- [Placeholders Reference](../job-configuration/placeholders-reference.md)
