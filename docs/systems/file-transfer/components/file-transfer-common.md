# FileTransferCommon

Shared library providing common interfaces, models, and utilities used across all FTS components.

## Repository
`FileTransferCommon/`

## Purpose

FileTransferCommon is the foundation library that provides:
- Common interfaces for server operations
- Shared data models
- Azure service helpers
- Cryptographic utilities
- Logging infrastructure

## Key Interfaces

### IServersReadWrite

Primary interface for all server implementations.

```csharp
public interface IServersReadWrite
{
    bool ReadFile(Actions action, ref string invalidReason);
    bool WriteFile(Actions action, int destinationNumber, byte[] content, ref string invalidReason);
    bool DeleteFile(Actions action, ref string invalidReason);
    bool RenameFile(Actions action, int destinationNumber, ref string invalidReason);
    bool RenameFileSource(Actions action, ref string invalidReason);
    bool GetFileList(Actions action, ref string invalidReason);
    bool ZipFileList(Actions action, int actionNumberSource, ref string invalidReason);
    bool ConcatenateList(Actions action, int actionNumberSource, ref string invalidReason);
    bool DeleteFileList(Actions action, int actionNumberSource, ref string invalidReason);
    string GetEventFileName(Actions action, int destinationNumber, byte[] content, ref string invalidReason);

    int TryTimesRead { get; set; }
    int TryTimesWrite { get; set; }
    int TryTimesDelete { get; set; }
    int TryTimesRename { get; set; }
}
```

### IActionProcess

Interface for action processing pipeline.

```csharp
public interface IActionProcess
{
    bool Action(Actions action, out string invalidReason);
    CurrentMessageInfo currentMessageInfo { get; set; }
}
```

### IVirusScan

Interface for virus scanning service.

```csharp
public interface IVirusScan
{
    string VirusScanBytes(byte[] content);
}
```

## Data Models

### Notification

CloudEvents notification model used throughout the system.

```csharp
public class Notification
{
    public string specversion { get; set; }
    public string type { get; set; }
    public string source { get; set; }
    public string subject { get; set; }
    public string id { get; set; }
    public string time { get; set; }
    public string dataschema { get; set; }
    public DataInfo data { get; set; }
}
```

See [Notification Model](../data-models/notification-model.md) for details.

### CurrentMessageInfo

Runtime context carrying message and configuration information.

```csharp
public class CurrentMessageInfo
{
    public string MessageId { get; set; }
    public ConfigModel configModel { get; set; }
    public string AzureFile { get; set; }
    public string AzureFileToUpload { get; set; }
    public string AzureTargetFileToUpload { get; set; }
    public string sftpintroot { get; set; }
    public string sftpextroot { get; set; }
    public string jobsettingsfolder { get; set; }
    // ... additional context fields
}
```

## Azure Service Helpers

### QueueHelper

Azure Queue Storage operations.

```csharp
public class QueueHelper
{
    public async Task<bool> SendMessage(string message);
    public async Task<string> GetMessage();
    public async Task<bool> DeleteMessage(string messageId, string popReceipt);
}
```

### TableHelper

Azure Table Storage operations.

```csharp
public class TableHelper
{
    public List<T> GetEntities<T>() where T : TableEntity, new();
    public async Task<bool> InsertOrMerge<T>(T entity) where T : TableEntity;
    public string SearchConfigTable(string accountId, string source, string type, string subject, string offsetId);
}
```

### BlobStoreServer

Azure Blob Storage operations implementing `IServersReadWrite`.

## Cryptographic Utilities

### PgpStream

PGP encryption and decryption using BouncyCastle.

```csharp
public static class PgpStream
{
    public static byte[] Decrypt(byte[] encryptedData, Stream privateKeyStream, string passPhrase);
    public static byte[] Encrypt(byte[] clearData, byte[] publicKey, bool armor, bool integrity,
                                  SymmetricKeyAlgorithmTag algorithm, CompressionAlgorithmTag compression,
                                  string fileName, bool useSubkey);
}
```

### VirusScan

ClamAV integration for virus scanning.

```csharp
public class VirusScan : IVirusScan
{
    public VirusScan(string serverIp, string serverPort);
    public string VirusScanBytes(byte[] content);
}
```

## Logging

### Logger

NLog-based logging wrapper.

```csharp
public static class Logger
{
    public static void Info(string message);
    public static void Warning(string message);
    public static void Error(string message);
    public static void Error(Exception ex, string message = null);
}
```

## Server Implementations

### AccessToServers

Base class for all server implementations.

```csharp
public class AccessToServers : IServersReadWrite
{
    protected CurrentMessageInfo currentMessageInfo;

    public string ReplaceFileName(string path, bool removeBackSlash);
    public string ReplaceDateTime(string value);
    protected string GetDestinationPath(Actions action, int destinationNumber);
}
```

### Server Classes

| Class | Type | Description |
|-------|------|-------------|
| `InternalSFTPServer` | Internal-SFTP | Azure File Share mount |
| `ExternalSFTPServer` | External-SFTP | Azure File Share mount |
| `VendorSFTPServer` | Vendor-SFTP | Remote SFTP via SSH.NET |
| `VendorFTPServer` | Vendor-FTP | Remote FTP via FluentFTP |
| `BlobStoreServer` | BlobStore | Azure Blob Storage |
| `GCPServer` | GCP | Google Cloud Storage |
| `AmazonS3Server` | AmazonS3 | Amazon S3 |
| `Internal_APIM` | Internal-APIM | Azure API Management |
| `AzureFile_API` | AzureFile-API | Azure File Share API |

## Configuration

### AppConfiguration

Static configuration loaded from appsettings.json.

```csharp
public static class AppConfiguration
{
    // Queue configuration
    public static string FixQueuesName_Jobs;
    public static string FixQueuesName_Jobstmsroute;
    // ... more queue names

    // Action types
    public static string Actions_Types_ReadFile;
    public static string Actions_Types_WriteFile;
    // ... more action types

    // Source types
    public static string Actions_SourceTypes_Internal_Type;
    public static string Actions_SourceTypes_Vendor_SFTP;
    // ... more source types
}
```

## Dependencies

- **Azure.Storage.Blobs**: Azure Blob Storage SDK
- **Azure.Storage.Queues**: Azure Queue Storage SDK
- **Microsoft.Azure.Cosmos.Table**: Azure Table Storage SDK
- **SSH.NET**: SFTP client library
- **FluentFTP**: FTP client library
- **BouncyCastle**: Cryptography library
- **NLog**: Logging framework
- **Newtonsoft.Json**: JSON serialization
- **Google.Cloud.Storage.V1**: GCP Storage SDK
- **AWSSDK.S3**: AWS S3 SDK

## Build Output

**Assembly**: `FileTransferCommon.dll`

**NuGet Package**: Published to internal Azure Artifacts feed

## Usage

All FTS handlers reference FileTransferCommon:

```xml
<PackageReference Include="FileTransferCommon" Version="1.0.0" />
```

## Related Documentation

- [Multi-Cloud Storage](../architecture/multi-cloud-storage.md)
- [Notification Model](../data-models/notification-model.md)
- [Source/Destination Types](../job-configuration/source-destination-types.md)
