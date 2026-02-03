# Job ConfigModel

The Job ConfigModel defines the structure of job configuration files (`jobsettings.json`) that specify file transfer operations.

## Source File
`FileJobHandler/FileJobHandler.Service/Model/ConfigModel.cs`

## Schema Overview

```
ConfigModel
├── feedName: string (required)
├── version: string
├── jobsecret: string
├── source: ServersAttributes
│   ├── name, type, host, port, path
│   ├── ssl: bool? (default: false)
│   ├── endpoint (BlobStore)
│   ├── bucketName (Google, S3)
│   ├── GCP fields: servicetype, projectID, privateKeyID, clientEmail,
│   │   clientID, authUri, tokenUri, authProvider, clientCertUrl, universeDomain
│   ├── S3 fields: region
│   ├── APIM fields: TargetUrl, TimeoutMs, RepeatAPI
│   └── credentials: Credentials
├── destinations[]: Destinations (extends ServersAttributes)
│   ├── id: int
│   └── removeFirstBackSlash: int? (Readnet specific)
└── actions[]: Actions
    ├── id, type, sourceFile, outputStream, outputFile
    ├── keyFile, keypassword, sourceStream, source, target
    ├── destination: int
    ├── FireEvent fields: eventType, eventSource, eventSubject, eventAccountId,
    │   eventId, eventCorrelationId, eventFileName
    ├── Encryption fields: encryptArmor, encryptIntegirity, encryptSymmetricKey,
    │   encryptCompressed, encryptFilename, encryptSubkey
    ├── FileList fields: listFileNames, pathlistFile, startWith, endWith,
    │   concateWith, numberOfFiles, listConcateFileNames
    ├── sort: string
    └── ActualOutPutStream: byte[] (runtime only)
```

## Root Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `feedName` | string | Yes | Job/feed identifier |
| `version` | string | No | Configuration version |
| `jobsecret` | string | No | Reference to secrets file |
| `source` | ServersAttributes | Yes | Source server configuration |
| `destinations` | Destinations[] | Yes | Array of destination configurations |
| `actions` | Actions[] | Yes | Array of actions to execute |

## ServersAttributes

Base class for source and destination configuration.

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `name` | string | | Server display name |
| `type` | string | | Server type (see [Source/Destination Types](../job-configuration/source-destination-types.md)) |
| `host` | string | | Server hostname |
| `port` | string | | Server port |
| `path` | string | | Base path on server |
| `ssl` | bool? | false | Enable SSL/TLS |
| `endpoint` | string | | BlobStore endpoint URL |
| `bucketName` | string | | GCP/S3 bucket name |
| `credentials` | Credentials | | Authentication credentials |

### GCP-Specific Fields

| Property | Type | Description |
|----------|------|-------------|
| `servicetype` | string | Service type (e.g., "service_account") |
| `projectID` | string | GCP project identifier |
| `privateKeyID` | string | Private key identifier |
| `clientEmail` | string | Service account email |
| `clientID` | string | Client identifier |
| `authUri` | string | OAuth authorization URI |
| `tokenUri` | string | OAuth token URI |
| `authProvider` | string | Auth provider certificate URL |
| `clientCertUrl` | string | Client certificate URL |
| `universeDomain` | string | Universe domain (default: googleapis.com) |

### S3-Specific Fields

| Property | Type | Description |
|----------|------|-------------|
| `region` | string | AWS region (e.g., "us-east-1") |

### APIM-Specific Fields

| Property | Type | Description |
|----------|------|-------------|
| `TargetUrl` | string | API endpoint URL |
| `TimeoutMs` | string | Request timeout in milliseconds |
| `RepeatAPI` | string | Number of retry attempts |

## Destinations

Extends ServersAttributes with additional properties.

| Property | Type | Description |
|----------|------|-------------|
| `id` | int | Destination identifier (referenced by actions) |
| `removeFirstBackSlash` | int? | Readnet-specific: remove leading backslash |

## Credentials

Authentication configuration.

| Property | Type | Description |
|----------|------|-------------|
| `type` | string | Auth type: `BasicAuth`, `SSHKey` |
| `username` | string | Username |
| `password` | string | Password |
| `sshkey` | string | SSH key file path |
| `sshkeyValue` | string | SSH key content (inline) |
| `sshkeyContent` | string | SSH key content (alternative) |
| `usernameValue` | string | Username from secrets |
| `passwordValue` | string | Password from secrets |
| `containerName` | string | BlobStore container name |
| `sasToken` | string | SAS token |
| `connectionString` | string | Connection string |
| `shareName` | string | Azure File Share name |
| `OAuthScope` | string | OAuth scope for APIM |
| `privateKey` | string | GCP private key |
| `accessKey` | string | S3 access key |
| `secretKey` | string | S3 secret key |

## Actions

Action configuration for job execution.

### Common Properties

| Property | Type | Description |
|----------|------|-------------|
| `id` | int | Action identifier (execution order) |
| `type` | string | Action type (see [Action Types](../job-configuration/action-types.md)) |
| `sourceFile` | string | Source file path |
| `sourceStream` | string | Reference to previous action output |
| `outputStream` | string | Output identifier for chaining |
| `outputFile` | string | Output file path |
| `destination` | int | Destination ID for write operations |

### FireEvent Properties

| Property | Type | Description |
|----------|------|-------------|
| `eventType` | string | Event type to emit |
| `eventSource` | string | Event source identifier |
| `eventSubject` | string | Event subject |
| `eventAccountId` | string | Target account ID |
| `eventId` | string | Event ID (or auto-generated) |
| `eventCorrelationId` | string | Correlation ID |
| `eventFileName` | string | Filename for event |

### Encryption Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `keyFile` | string | | Path to encryption key file |
| `keypassword` | string | | Key password (from secrets) |
| `encryptArmor` | bool | true | ASCII armor output |
| `encryptIntegirity` | bool | true | Include integrity check |
| `encryptSymmetricKey` | string | "Cast5" | Symmetric algorithm |
| `encryptCompressed` | string | "Uncompressed" | Compression algorithm |
| `encryptFilename` | string | "" | Encrypted filename |
| `encryptSubkey` | bool | false | Use subkey for encryption |

### File List Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `listFileNames` | List\<string\> | | List of files to process |
| `pathlistFile` | string | | Path to file list |
| `startWith` | string | | Filter: filename starts with |
| `endWith` | string | | Filter: filename ends with |
| `concateWith` | string | | Concatenation delimiter |
| `numberOfFiles` | int | 1 | Max files to process |
| `listConcateFileNames` | List\<string\> | | Concatenated file list |
| `sort` | string | | Sort order for file list |

### Runtime Properties

| Property | Type | Description |
|----------|------|-------------|
| `ActualOutPutStream` | byte[] | Runtime: processed file content (do not serialize) |

## Example Configuration

```json
{
  "feedName": "vendor-data-import",
  "version": "1.0",
  "source": {
    "name": "Vendor SFTP",
    "type": "Vendor-SFTP",
    "host": "sftp.vendor.com",
    "port": "22",
    "path": "/outbound",
    "credentials": {
      "type": "BasicAuth",
      "username": "{@JobSecrets.vendorUser}",
      "password": "{@JobSecrets.vendorPass}"
    }
  },
  "destinations": [
    {
      "id": 1,
      "name": "Internal Storage",
      "type": "BlobStore",
      "endpoint": "https://storage.blob.core.windows.net",
      "credentials": {
        "containerName": "imports",
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
      "outputFile": "{@JobSettings.destinations.1.path}/processed/{@Message.fileName.name}.csv"
    },
    {
      "id": 4,
      "type": "DeleteFile",
      "sourceFile": "{@JobSettings.source.path}/{@Message.fileName}"
    }
  ]
}
```

## Related Documentation

- [Job Configuration Specification](../job-configuration/specification.md)
- [Source/Destination Types](../job-configuration/source-destination-types.md)
- [Action Types](../job-configuration/action-types.md)
- [Placeholders Reference](../job-configuration/placeholders-reference.md)
- [Credentials Model](./credentials-model.md)
