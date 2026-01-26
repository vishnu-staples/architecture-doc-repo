# FileJobHandler

Generic job processing handler that executes file transfer operations based on job configurations.

## Repository
`FileJobHandler/`

## Purpose

FileJobHandler is the primary workhorse of FTS:
- Polls Azure Queue for job messages
- Loads job configurations from file share
- Executes action sequences
- Updates job status

## Architecture

```
Azure Queue (jobs)
       │
       ▼
┌─────────────────────────┐
│    FileJobHandler       │
├─────────────────────────┤
│ QueueListener           │ ◄── Polls queue
│ JobProcessor            │ ◄── Orchestrates job
│ SettingConfiguration    │ ◄── Loads config
│ ActionProcess           │ ◄── Executes actions
│ Server Implementations  │ ◄── File operations
└─────────────────────────┘
       │
       ├──▶ Sources (SFTP, Blob, S3, GCP, etc.)
       ├──▶ Destinations (SFTP, Blob, FTP, etc.)
       └──▶ Azure Table (status updates)
```

## Core Components

### QueueListener

Polls the queue for new messages.

```csharp
public class QueueListener
{
    public async Task StartListening();
    public async Task<QueueMessage> GetNextMessage();
    public async Task DeleteMessage(QueueMessage message);
}
```

### JobProcessor

Orchestrates job execution.

```csharp
public class JobProcessor
{
    public async Task<bool> ProcessJob(Notification notification);
    private ConfigModel LoadConfiguration(string source, string accountId, string fileName);
    private bool ExecuteActions(ConfigModel config, CurrentMessageInfo context);
}
```

### SettingConfiguration

Manages configuration loading and placeholder resolution.

```csharp
public class SettingConfiguration
{
    public ConfigModel LoadJobSettings(string folderPath);
    public Dictionary<string, string> LoadJobSecrets(string folderPath);
    public string ResolvePlaceholders(string value, CurrentMessageInfo context);
    public int FindNumberToDirect(string pattern, string regex, string messageId);
    public string FindSecretValue(string placeholder, CurrentMessageInfo context);
}
```

### ActionProcess

Executes individual actions.

```csharp
public class ActionProcess : IActionProcess
{
    public bool Action(ConfigModel.Actions action, out string invalidReason);

    private bool ReadFile(Actions action, out string invalidReason);
    private bool WriteFile(Actions action, out string invalidReason);
    private bool DeleteFile(Actions action, out string invalidReason);
    private bool RenameFile(Actions action, out string invalidReason);
    private bool GetFileList(Actions action, out string invalidReason);
    private bool ZipFileList(Actions action, out string invalidReason);
    private bool ConcatenateList(Actions action, out string invalidReason);
    private bool DeleteFileList(Actions action, out string invalidReason);
    private bool PGP_Decryption(Actions action, out string invalidReason);
    private bool PGP_Encryption(Actions action, out string invalidReason);
    private bool RSA_Encryption(Actions action, out string invalidReason);
    private bool Scan(Actions action, out string invalidReason);
    private bool Encoding(Actions action, out string invalidReason);
    private bool FireEvent(Actions action, out string invalidReason);
    private bool FireEventList(Actions action, out string invalidReason);
    private bool NotifyServer(Actions action, out string invalidReason);
}
```

## Message Flow

### Input Contract

**Queue Message** (CloudEvent):
```json
{
  "specversion": "1.0",
  "type": "Succeeded",
  "source": "FTP.File.ExternalService",
  "subject": "File Upload",
  "data": {
    "accountId": "ACCOUNT001",
    "fileName": "incoming/data.csv",
    "correlationId": "corr-123"
  }
}
```

### Output Contract

**JobStatus Update**:
```json
{
  "Status": "Succeeded",
  "InvalidReason": "File Successfully write to destination",
  "Action": "WriteFile"
}
```

**FireEvent Output** (new CloudEvent):
```json
{
  "specversion": "1.0",
  "type": "Succeeded",
  "source": "FTS.JobHandler",
  "subject": "File Ready",
  "data": {
    "accountId": "ACCOUNT001",
    "fileName": "processed/data.csv",
    "correlationId": "corr-123"
  }
}
```

## Job Processing Flow

```
1. Dequeue Message
       │
       ▼
2. Parse CloudEvent
       │
       ▼
3. Build Configuration Path
   /mnt/jobconfig/{source}/{accountID}/{subFolder}/
       │
       ▼
4. Load jobsettings.json
       │
       ▼
5. Load jobsecrets.json (if needed)
       │
       ▼
6. Resolve Placeholders
       │
       ▼
7. Execute Actions (sequentially)
   ┌─► ReadFile
   │      │
   │      ▼
   │   PGP-Decrypt
   │      │
   │      ▼
   │   Scan
   │      │
   │      ▼
   │   WriteFile
   │      │
   │      ▼
   └── DeleteFile
       │
       ▼
8. Update JobStatus (Succeeded/Failed)
       │
       ▼
9. Delete Queue Message
```

## Server Selection

The handler instantiates appropriate server class based on type:

```csharp
if (source.type == "Internal-SFTP")
    Server = new InternalSFTPServer(currentMessageInfo);
else if (source.type == "External-SFTP")
    Server = new ExternalSFTPServer(currentMessageInfo);
else if (source.type == "Vendor-SFTP")
    Server = new VendorSFTPServer(currentMessageInfo);
else if (source.type == "Vendor-FTP")
    Server = new VendorFTPServer(currentMessageInfo);
else if (source.type == "BlobStore")
    Server = new BlobStoreServer(currentMessageInfo);
else if (source.type == "GCP")
    Server = new GCPServer(currentMessageInfo);
else if (source.type == "AmazonS3")
    Server = new AmazonS3Server(currentMessageInfo);
else if (source.type == "Internal-APIM")
    Server = new Internal_APIM(currentMessageInfo);
else if (source.type == "AzureFile-API")
    Server = new AzureFile_API(currentMessageInfo);
```

## Retry Logic

### Source Operations

```csharp
for (int i = 1; i <= RetryFailureSource; i++)
{
    Thread.Sleep(RetryFailureSourceDelay * 1000);
    Result = Server.ReadFile(action, ref reason);
    if (Result) break;
}
```

### Destination Operations

```csharp
for (int i = 1; i <= RetryFailureDestination; i++)
{
    Thread.Sleep(RetryFailureDestinationDelay * 1000);
    Result = Server.WriteFile(action, destination, content, ref reason);
    if (Result) break;
}
```

### Configuration

```json
{
  "Actions": {
    "RetryFailureSource": 3,
    "RetryFailureDestination": 3,
    "RetryFailureSourceDelay": 5,
    "RetryFailureDestinationDelay": 5
  }
}
```

## Configuration

### appsettings.json

```json
{
  "Queues": {
    "jobs": {
      "Url": "https://{storage}.queue.core.windows.net/jobs",
      "SASToken": "..."
    },
    "NextCheck": 1000,
    "VisibilityTimeout": 120
  },
  "AzureFiles": {
    "sftpintroot": "/mnt/sftpintroot",
    "sftpextroot": "/mnt/sftpextroot",
    "jobsettingsfolder": "/mnt/jobconfig/{@Message.source}/{@Message.accountID}/{@Message.fileName.subFolder}"
  },
  "VirusScan": {
    "ClamAVServiceIP": "10.21.129.142",
    "ClamAVServicePort": "3310"
  },
  "KeyVaultSecrets": {
    "Enable": true,
    "Uri": "https://fts-{env}-key-vault.vault.azure.net/"
  }
}
```

## Deployment

### Kubernetes

```yaml
spec:
  containers:
    - name: file-job-handler
      image: enterprisenonpacr.azurecr.io/enterprise/file-job-handler:tag
      volumeMounts:
        - name: appsettings
          mountPath: /app/appsettings.Production.json
        - name: sftpintroot
          mountPath: /mnt/sftpintroot
        - name: sftpextroot
          mountPath: /mnt/sftpextroot
```

### Volumes

| Mount | Purpose |
|-------|---------|
| `/mnt/sftpintroot` | Internal SFTP files |
| `/mnt/sftpextroot` | External SFTP files |
| `/mnt/jobconfig` | Job configuration files |

## Error Handling

### Configuration Errors

- Missing `jobsettings.json`: Job fails immediately
- Invalid JSON: Job fails with parse error
- Missing secrets: Job fails with resolution error

### Action Errors

- Read failure: Retry, then fail job
- Write failure: Retry, then fail job
- Virus detected: Fail job with virus details
- Encryption error: Fail job with error message

## Dependencies

- FileTransferCommon
- Azure.Storage.Queues
- Azure.Storage.Blobs
- Microsoft.Azure.Cosmos.Table
- SSH.NET
- FluentFTP
- BouncyCastle
- Google.Cloud.Storage.V1
- AWSSDK.S3

## Monitoring

### Datadog APM

- Job processing duration
- Action execution times
- Success/failure rates
- Retry counts

### NLog

```csharp
Logger.Info($"Start ActionProcess Action method ActionID={action.id}, ActionType={action.type}");
Logger.Error(ex, $"Action failed. MessageId={messageId}");
```

## Related Documentation

- [Job ConfigModel](../data-models/job-config-model.md)
- [Action Types](../job-configuration/action-types.md)
- [Jobsettings Storage](../job-configuration/jobsettings-storage.md)
- [Multi-Cloud Storage](../architecture/multi-cloud-storage.md)
