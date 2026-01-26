# FileEventHandler

Event Hub listener service responsible for event ingestion, validation, and routing to processing queues.

## Repository
`FileEventHandler/`

## Purpose

FileEventHandler is the entry point for all FTS events:
- Listens to Azure Event Hub for CloudEvents
- Validates incoming messages
- Routes events to appropriate processing queues
- Records job status in Azure Table Storage

## Architecture

```
Azure Event Hub
       │
       ▼
┌─────────────────────────┐
│   FileEventHandler      │
├─────────────────────────┤
│ EventHubListener        │ ◄── Consumes events
│ MessageHandler          │ ◄── Validates & routes
│ JobStatusManipulate     │ ◄── Records status
│ TableHelper             │ ◄── Config lookup
│ QueueHelper             │ ◄── Queue operations
└─────────────────────────┘
       │
       ▼
Azure Queue Storage
```

## Core Components

### EventHubListener

Manages Event Hub connection and event consumption.

```csharp
public class EventHubListener
{
    public async Task StartListening();
    public async Task ProcessEvent(EventData eventData);
}
```

**Responsibilities**:
- Establish Event Hub connection
- Manage consumer group
- Track partition offsets
- Handle connection failures

### MessageHandler

Processes individual messages.

```csharp
public class MessageHandler : IMessageHandler
{
    public async Task<enumProcessDataResult> Process(string data, string partitionId, long offsetId);
    private Notification ValidateData(string data, string partitionId, long offsetId);
    private async Task<enumProcessDataResult> SendMessageToQueue(Notification notify, string data, string partitionId, long offsetId);
}
```

**Responsibilities**:
- Parse CloudEvent JSON
- Validate required fields
- Query routing table
- Send to appropriate queue

### JobStatusManipulate

Manages job status records.

```csharp
public class JobStatusManipulate
{
    public async Task<bool> InsertInvalid(string data, string partitionKey, string partitionId, long offsetId, string reason);
    public async Task<bool> InsertAssigned(string data, Notification notify, string partitionId, long offsetId, string queue);
    public async Task<bool> InsertNotAssigned(string data, Notification notify, string partitionId, long offsetId, string reason);
}
```

## Message Flow

### Input Contract

**CloudEvent (from Event Hub)**:
```json
{
  "specversion": "1.0",
  "type": "Succeeded",
  "source": "FTP.File.ExternalService",
  "subject": "File Upload",
  "id": "event-uuid",
  "time": "2024-01-15T10:30:00+0000",
  "data": {
    "accountId": "ACCOUNT001",
    "fileName": "incoming/file.csv",
    "correlationId": "corr-123"
  }
}
```

### Output Contract

**Queue Message** (same CloudEvent forwarded):
```json
{
  "specversion": "1.0",
  "type": "Succeeded",
  "source": "FTP.File.ExternalService",
  "subject": "File Upload",
  "id": "event-uuid",
  "time": "2024-01-15T10:30:00+0000",
  "data": {
    "accountId": "ACCOUNT001",
    "fileName": "incoming/file.csv",
    "correlationId": "corr-123"
  }
}
```

**JobStatus Record**:
```json
{
  "PartitionKey": "FTP.File.ExternalService-ACCOUNT001-file.csv",
  "RowKey": "unique-id",
  "Status": "Assigned",
  "Queue": "jobs",
  "EventType": "Succeeded",
  "EventSource": "FTP.File.ExternalService",
  "EventSubject": "File Upload",
  "AccountID": "ACCOUNT001",
  "FileName": "incoming/file.csv"
}
```

## Validation Logic

### Required Fields Check

```csharp
if (string.IsNullOrEmpty(Notify.type) ||
    string.IsNullOrEmpty(Notify.subject) ||
    string.IsNullOrEmpty(Notify.source) ||
    string.IsNullOrEmpty(Notify.data.accountId))
{
    return null; // Invalid message
}
```

### Process Results

```csharp
public enum enumProcessDataResult
{
    Processed,    // Successfully routed
    GarbageMsg,   // Invalid or unroutable
    AzureIssue    // Azure service error
}
```

## Routing Logic

### Config Table Query

The routing lookup uses the `jobconfig` Azure Table:

```csharp
PartitionKey = _tableHelper.SearchConfigTable(
    Notify.data.accountId,  // EventAccountID
    Notify.source,          // EventSource
    Notify.type,            // EventType
    Notify.subject,         // EventSubject
    OriginalOffsetId
);
```

### Queue Selection

```csharp
if (PartitionKey == "jobs")
    await _ProjectInitialize.queuejobs.SendMessage(Data);
else if (PartitionKey == "jobstmsroute")
    await _ProjectInitialize.queueTmsRoute.SendMessage(Data);
else if (PartitionKey == "jobstmsstatus")
    await _ProjectInitialize.queueTmsStatus.SendMessage(Data);
// ... additional queue mappings
```

## Configuration

### appsettings.json

```json
{
  "EventHub": {
    "EnterpriseFTP": {
      "ConnectionString": "Endpoint=sb://...",
      "EventHubName": "enterprise-{env}-ftp-ehb"
    }
  },
  "Queues": {
    "jobs": {
      "Url": "https://{storage}.queue.core.windows.net/jobs",
      "SASToken": "..."
    }
  },
  "Tables": {
    "jobstatus": {
      "Url": "https://{storage}.table.core.windows.net/",
      "SASToken": "...",
      "TableName": "jobstatus"
    },
    "jobconfig": {
      "Url": "https://{storage}.table.core.windows.net/",
      "SASToken": "...",
      "TableName": "jobconfig"
    }
  }
}
```

## Error Handling

### Invalid Messages

- Logged with full message content
- Recorded in `jobstatus` with `Invalid` status
- Event offset committed (no retry)

### No Route Found

- Logged as warning
- Recorded as `NoAssigned`
- Event offset committed (no retry)

### Azure Service Errors

- Logged as error
- Returns `AzureIssue` result
- Event may be retried based on consumer group behavior

## Deployment

### Kubernetes

Single pod deployment listening to Event Hub.

### Volumes

- No persistent storage required
- Configuration via Kubernetes secrets

### Scaling

- Single instance recommended (consumer group manages partitions)
- Multiple instances share partition load

## Dependencies

- FileTransferCommon (shared library)
- Azure.Messaging.EventHubs
- Microsoft.Azure.Cosmos.Table
- Azure.Storage.Queues
- NLog

## Monitoring

### Key Metrics

- Events processed per second
- Invalid message rate
- Routing success rate
- Azure service latency

### Datadog Integration

```csharp
Logger.Info($"Message processed: PartitionID={partitionId}, OffsetID={offsetId}");
```

## Related Documentation

- [Event-Driven Architecture](../architecture/event-driven-architecture.md)
- [Queue Architecture](../architecture/queue-architecture.md)
- [Routing ConfigModel](../data-models/routing-config-model.md)
- [JobStatus Model](../data-models/jobstatus-model.md)
