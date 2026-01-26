# Credentials Model

The Credentials class defines authentication configuration for source and destination servers in job configurations.

## Source File
`FileJobHandler/FileJobHandler.Service/Model/ConfigModel.cs` (nested class)

## Schema

```csharp
public class Credentials
{
    public string type { get; set; }
    public string username { get; set; }
    public string password { get; set; }
    public string sshkey { get; set; }
    public string containerName { get; set; }
    public string sasToken { get; set; }
    public string usernameValue { get; set; }
    public string passwordValue { get; set; }
    public string sshkeyValue { get; set; }
    public string sshkeyContent { get; set; }
    public string connectionString { get; set; }
    public string shareName { get; set; }
    public string OAuthScope { get; set; }
    public string privateKey { get; set; }
    public string accessKey { get; set; }
    public string secretKey { get; set; }
}
```

## Property Reference

### Authentication Type

| Property | Type | Description |
|----------|------|-------------|
| `type` | string | Authentication method: `BasicAuth`, `SSHKey` |

### Username/Password Authentication

| Property | Type | Description |
|----------|------|-------------|
| `username` | string | Username (supports placeholders) |
| `password` | string | Password (supports placeholders) |
| `usernameValue` | string | Resolved username value |
| `passwordValue` | string | Resolved password value |

### SSH Key Authentication

| Property | Type | Description |
|----------|------|-------------|
| `sshkey` | string | Path to SSH key file |
| `sshkeyValue` | string | Resolved SSH key content |
| `sshkeyContent` | string | Alternative SSH key content field |

### Azure Blob Storage

| Property | Type | Description |
|----------|------|-------------|
| `containerName` | string | Blob container name |
| `connectionString` | string | Storage connection string |
| `sasToken` | string | SAS token for authentication |

### Azure File Share

| Property | Type | Description |
|----------|------|-------------|
| `shareName` | string | File share name |
| `connectionString` | string | Storage connection string |

### Azure API Management

| Property | Type | Description |
|----------|------|-------------|
| `OAuthScope` | string | OAuth 2.0 scope for token requests |

### Google Cloud Platform

| Property | Type | Description |
|----------|------|-------------|
| `privateKey` | string | GCP service account private key |

### Amazon S3

| Property | Type | Description |
|----------|------|-------------|
| `accessKey` | string | AWS access key ID |
| `secretKey` | string | AWS secret access key |

## Credential Types by Server Type

### Vendor-SFTP / External-SFTP

**BasicAuth**:
```json
{
  "credentials": {
    "type": "BasicAuth",
    "username": "{@JobSecrets.sftpUser}",
    "password": "{@JobSecrets.sftpPass}"
  }
}
```

**SSHKey**:
```json
{
  "credentials": {
    "type": "SSHKey",
    "username": "{@JobSecrets.sftpUser}",
    "sshkey": "{@AppSettings.jobsettingsfolder}/id_rsa"
  }
}
```

### Vendor-FTP

```json
{
  "credentials": {
    "username": "{@JobSecrets.ftpUser}",
    "password": "{@JobSecrets.ftpPass}"
  }
}
```

### BlobStore

**Connection String**:
```json
{
  "credentials": {
    "containerName": "files",
    "connectionString": "{@JobSecrets.blobConnection}"
  }
}
```

**SAS Token**:
```json
{
  "credentials": {
    "containerName": "files",
    "sasToken": "{@JobSecrets.blobSasToken}"
  }
}
```

### AzureFile-API

```json
{
  "credentials": {
    "shareName": "fileshare",
    "connectionString": "{@JobSecrets.fileShareConnection}"
  }
}
```

### Internal-APIM

```json
{
  "credentials": {
    "OAuthScope": "api://my-api/.default"
  }
}
```

### GCP

```json
{
  "credentials": {
    "privateKey": "{@JobSecrets.gcpPrivateKey}"
  }
}
```

### AmazonS3

```json
{
  "credentials": {
    "accessKey": "{@JobSecrets.s3AccessKey}",
    "secretKey": "{@JobSecrets.s3SecretKey}"
  }
}
```

## Security Best Practices

### Use Job Secrets

Always reference credentials through `jobsecrets.json`:

```json
// jobsettings.json
{
  "credentials": {
    "username": "{@JobSecrets.vendorUser}",
    "password": "{@JobSecrets.vendorPass}"
  }
}

// jobsecrets.json
{
  "vendorUser": "actual-username",
  "vendorPass": "actual-password"
}
```

### Use Key Vault

For production, use Azure Key Vault via the `jobsecret` field in the root configuration:

```json
{
  "feedName": "secure-feed",
  "jobsecret": "vendor-secrets",
  "source": {
    "credentials": {
      "password": "{@JobSecrets.vendorPass}"
    }
  }
}
```

The `jobsecret` field references a Key Vault secret name. The secret value must be a JSON object:
```json
{
  "vendorUser": "actual-username",
  "vendorPass": "actual-password"
}
```

> **Note**: The `{@KeyVault.*}` placeholder syntax is NOT supported. Always use `{@JobSecrets.*}` with the `jobsecret` field for Key Vault integration.

### SSH Key Storage

Store SSH keys in the job configuration folder:

```
/mnt/jobconfig/{source}/{accountID}/{folder}/
├── jobsettings.json
├── jobsecrets.json
└── id_rsa           # SSH private key
```

Reference in configuration:
```json
{
  "credentials": {
    "type": "SSHKey",
    "username": "sftp-user",
    "sshkey": "{@AppSettings.jobsettingsfolder}/id_rsa"
  }
}
```

## Credential Resolution Flow

1. **Placeholder Detection**: System identifies `{@JobSecrets.xxx}` patterns
2. **Secrets File Load**: `jobsecrets.json` loaded from same directory
3. **Value Substitution**: Placeholders replaced with actual values
4. **Key Vault Fallback**: If file not found, check Azure Key Vault
5. **Credential Usage**: Resolved values used for authentication

## Validation Rules

The system validates credentials based on server type:

| Server Type | Required Credentials |
|-------------|---------------------|
| `Vendor-SFTP` (BasicAuth) | `username`, `password` |
| `Vendor-SFTP` (SSHKey) | `username`, `sshkey` |
| `Vendor-FTP` | `username`, `password` |
| `BlobStore` | `containerName`, `connectionString` or `sasToken` |
| `AzureFile-API` | `shareName`, `connectionString` |
| `Internal-APIM` | `OAuthScope` |
| `GCP` | `privateKey` |
| `AmazonS3` | `accessKey`, `secretKey` |

## Related Documentation

- [Job ConfigModel](./job-config-model.md)
- [Source/Destination Types](../job-configuration/source-destination-types.md)
- [Security & Secrets](../operations/security-secrets.md)
- [Placeholders Reference](../job-configuration/placeholders-reference.md)
