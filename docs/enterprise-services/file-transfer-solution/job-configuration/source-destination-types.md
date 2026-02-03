# Source/Destination Types

This document describes all supported server types for sources and destinations in FTS job configurations.

## Type Values

| Type Value | Description | Supported As |
|------------|-------------|--------------|
| `Internal-SFTP` | Internal SFTP (Azure File Share mounted) | Source, Destination |
| `External-SFTP` | External SFTP (Azure File Share mounted) | Source, Destination |
| `Vendor-SFTP` | Third-party vendor SFTP servers | Source, Destination |
| `Vendor-FTP` | Third-party vendor FTP/FTPS servers | Source, Destination |
| `BlobStore` | Azure Blob Storage | Source, Destination |
| `GCP` | Google Cloud Storage | Source, Destination |
| `AmazonS3` | Amazon S3 | Source, Destination |
| `Internal-APIM` | Azure API Management | Source, Destination |
| `AzureFile-API` | Azure File Share via API | Source, Destination |

## Internal-SFTP

Azure File Share mounted internally for SFTP access.

### Configuration

```json
{
  "type": "Internal-SFTP",
  "name": "Internal SFTP Server",
  "path": "{@AppSettings.sftpintroot}/{@Message.accountID}"
}
```

### Fields

| Field | Required | Description |
|-------|----------|-------------|
| `type` | Yes | `"Internal-SFTP"` |
| `name` | No | Display name |
| `path` | Yes | File path (supports placeholders) |

### Mount Path
`/mnt/sftpintroot`

---

## External-SFTP

Azure File Share mounted for external-facing SFTP access.

### Configuration

```json
{
  "type": "External-SFTP",
  "name": "External SFTP Server",
  "path": "{@AppSettings.sftpextroot}/{@Message.accountID}"
}
```

### Fields

| Field | Required | Description |
|-------|----------|-------------|
| `type` | Yes | `"External-SFTP"` |
| `name` | No | Display name |
| `path` | Yes | File path (supports placeholders) |

### Mount Path
`/mnt/sftpextroot`

---

## Vendor-SFTP

Third-party vendor SFTP servers accessed via SSH.

### Configuration (Basic Auth)

```json
{
  "type": "Vendor-SFTP",
  "name": "Vendor SFTP",
  "host": "sftp.vendor.com",
  "port": "22",
  "path": "/outbound",
  "credentials": {
    "type": "BasicAuth",
    "username": "{@JobSecrets.sftpUser}",
    "password": "{@JobSecrets.sftpPass}"
  }
}
```

### Configuration (SSH Key)

```json
{
  "type": "Vendor-SFTP",
  "name": "Vendor SFTP",
  "host": "sftp.vendor.com",
  "port": "22",
  "path": "/outbound",
  "credentials": {
    "type": "SSHKey",
    "username": "{@JobSecrets.sftpUser}",
    "sshkey": "{@AppSettings.jobsettingsfolder}/id_rsa"
  }
}
```

### Fields

| Field | Required | Description |
|-------|----------|-------------|
| `type` | Yes | `"Vendor-SFTP"` |
| `host` | Yes | SFTP server hostname |
| `port` | Yes | Port number (typically "22") |
| `path` | Yes | Base directory path |
| `credentials.type` | Yes | `"BasicAuth"` or `"SSHKey"` |
| `credentials.username` | Yes | SSH username |
| `credentials.password` | Conditional | Password (for BasicAuth) |
| `credentials.sshkey` | Conditional | Path to SSH private key (for SSHKey) |

---

## Vendor-FTP

Third-party vendor FTP/FTPS servers.

### Configuration

```json
{
  "type": "Vendor-FTP",
  "name": "Vendor FTP",
  "host": "ftp.vendor.com",
  "port": "21",
  "path": "/uploads",
  "ssl": true,
  "credentials": {
    "username": "{@JobSecrets.ftpUser}",
    "password": "{@JobSecrets.ftpPass}"
  }
}
```

### Fields

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `type` | Yes | | `"Vendor-FTP"` |
| `host` | Yes | | FTP server hostname |
| `port` | Yes | | Port number (typically "21") |
| `path` | Yes | | Base directory path |
| `ssl` | No | `false` | Enable FTPS (explicit TLS) |
| `credentials.username` | Yes | | FTP username |
| `credentials.password` | Yes | | FTP password |

---

## BlobStore

Azure Blob Storage.

### Configuration

```json
{
  "type": "BlobStore",
  "name": "Azure Blob Storage",
  "endpoint": "https://storageaccount.blob.core.windows.net",
  "path": "incoming/files",
  "credentials": {
    "containerName": "data",
    "connectionString": "{@JobSecrets.blobConnection}"
  }
}
```

### Alternative (SAS Token)

```json
{
  "type": "BlobStore",
  "endpoint": "https://storageaccount.blob.core.windows.net",
  "credentials": {
    "containerName": "data",
    "sasToken": "{@JobSecrets.blobSasToken}"
  }
}
```

### Fields

| Field | Required | Description |
|-------|----------|-------------|
| `type` | Yes | `"BlobStore"` |
| `endpoint` | Yes | Blob storage endpoint URL |
| `path` | No | Virtual directory path within container |
| `credentials.containerName` | Yes | Blob container name |
| `credentials.connectionString` | Conditional | Full connection string |
| `credentials.sasToken` | Conditional | SAS token (if not using connection string) |

---

## GCP (Google Cloud Storage)

Google Cloud Storage buckets.

### Configuration

```json
{
  "type": "GCP",
  "name": "Google Cloud Storage",
  "bucketName": "my-bucket",
  "path": "data/incoming",
  "servicetype": "service_account",
  "projectID": "my-gcp-project",
  "privateKeyID": "{@JobSecrets.gcpKeyId}",
  "clientEmail": "service@project.iam.gserviceaccount.com",
  "clientID": "123456789012345678901",
  "authUri": "https://accounts.google.com/o/oauth2/auth",
  "tokenUri": "https://oauth2.googleapis.com/token",
  "authProvider": "https://www.googleapis.com/oauth2/v1/certs",
  "clientCertUrl": "https://www.googleapis.com/robot/v1/metadata/x509/...",
  "universeDomain": "googleapis.com",
  "credentials": {
    "privateKey": "{@JobSecrets.gcpPrivateKey}"
  }
}
```

### Fields

| Field | Required | Description |
|-------|----------|-------------|
| `type` | Yes | `"GCP"` |
| `bucketName` | Yes | GCS bucket name |
| `path` | No | Object prefix path |
| `servicetype` | Yes | Authentication type |
| `projectID` | Yes | GCP project identifier |
| `clientEmail` | Yes | Service account email |
| `credentials.privateKey` | Yes | Service account private key |

---

## AmazonS3

Amazon S3 buckets.

### Configuration

```json
{
  "type": "AmazonS3",
  "name": "Amazon S3",
  "bucketName": "my-bucket",
  "region": "us-east-1",
  "path": "data/incoming",
  "credentials": {
    "accessKey": "{@JobSecrets.s3AccessKey}",
    "secretKey": "{@JobSecrets.s3SecretKey}"
  }
}
```

### Fields

| Field | Required | Description |
|-------|----------|-------------|
| `type` | Yes | `"AmazonS3"` |
| `bucketName` | Yes | S3 bucket name |
| `region` | Yes | AWS region (e.g., "us-east-1") |
| `path` | No | Object prefix path |
| `credentials.accessKey` | Yes | AWS access key ID |
| `credentials.secretKey` | Yes | AWS secret access key |

---

## Internal-APIM

Azure API Management for HTTP-based integrations.

### Configuration

```json
{
  "type": "Internal-APIM",
  "name": "Internal API",
  "TargetUrl": "https://api.example.com/v1/files",
  "TimeoutMs": "30000",
  "RepeatAPI": "3",
  "credentials": {
    "OAuthScope": "api://my-api/.default"
  }
}
```

### Fields

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `type` | Yes | | `"Internal-APIM"` |
| `TargetUrl` | Yes | | API endpoint URL |
| `TimeoutMs` | No | "30000" | Request timeout in milliseconds |
| `RepeatAPI` | No | "1" | Number of retry attempts |
| `credentials.OAuthScope` | Yes | | OAuth 2.0 scope for token |

### OAuth Flow
Uses Azure AD client credentials flow with APIM configuration from `appsettings.json`.

---

## AzureFile-API

Azure File Share accessed via Azure SDK.

### Configuration

```json
{
  "type": "AzureFile-API",
  "name": "Azure File Share",
  "path": "/share/folder",
  "credentials": {
    "shareName": "myfileshare",
    "connectionString": "{@JobSecrets.fileShareConnection}"
  }
}
```

### Fields

| Field | Required | Description |
|-------|----------|-------------|
| `type` | Yes | `"AzureFile-API"` |
| `path` | Yes | Directory path within share |
| `credentials.shareName` | Yes | Azure File Share name |
| `credentials.connectionString` | Yes | Storage connection string |

---

## Operation Support Matrix

| Type | ReadFile | WriteFile | DeleteFile | RenameFile | GetFileList |
|------|----------|-----------|------------|------------|-------------|
| Internal-SFTP | Yes | Yes | Yes | Yes | Yes |
| External-SFTP | Yes | Yes | Yes | Yes | Yes |
| Vendor-SFTP | Yes | Yes | Yes | Yes | Yes |
| Vendor-FTP | Yes | Yes | Yes | Yes | Yes |
| BlobStore | Yes | Yes | Yes | Yes | Yes |
| GCP | Yes | Yes | Yes | Yes | Yes |
| AmazonS3 | Yes | Yes | Yes | No | No |
| Internal-APIM | Yes | Yes | No | No | No |
| AzureFile-API | Yes | Yes | Yes | Yes | Yes |

## Related Documentation

- [Job Configuration Specification](./specification.md)
- [Credentials Model](../data-models/credentials-model.md)
- [Multi-Cloud Storage](../architecture/multi-cloud-storage.md)
