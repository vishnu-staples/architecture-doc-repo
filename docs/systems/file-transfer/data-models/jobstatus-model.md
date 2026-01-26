# JobStatus Model

The JobStatus model represents Azure Table entities that track the processing status of all jobs in the FTS platform.

## Source File
`FileEventHandler/FileEventHandler.Service/Models/JobStatusModel.cs`

## Schema

```csharp
public class JobStatusModel : TableEntity
{
    public string Message { get; set; }
    public string Status { get; set; }
    public string OffsetID { get; set; }
    public string Queue { get; set; }
    public string EventType { get; set; }
    public string EventSource { get; set; }
    public string EventSubject { get; set; }
    public string EventTime { get; set; }
    public string AccountID { get; set; }
    public string CorrelationID { get; set; }
    public string FlowType { get; set; }
    public string FileName { get; set; }
    public string InvalidReason { get; set; }
    public string Action { get; set; }
}
```

## Azure Table Entity Properties

### Inherited Properties (from TableEntity)

| Property | Type | Description |
|----------|------|-------------|
| `PartitionKey` | string | Composite key: `{source}-{accountId}-{fileName}` |
| `RowKey` | string | Unique job identifier |
| `Timestamp` | DateTimeOffset | Last modified time (auto-managed) |
| `ETag` | string | Entity tag for optimistic concurrency |

### Custom Properties

| Property | Type | Description |
|----------|------|-------------|
| `Message` | string | Original CloudEvent message (JSON) or error details |
| `Status` | string | Current job status |
| `OffsetID` | string | Event Hub offset ID: `{partitionId}.{offset}` |
| `Queue` | string | Assigned queue name |
| `EventType` | string | Event type from notification |
| `EventSource` | string | Event source from notification |
| `EventSubject` | string | Event subject from notification |
| `EventTime` | string | Event timestamp |
| `AccountID` | string | Account identifier |
| `CorrelationID` | string | Correlation ID for tracing |
| `FlowType` | string | Flow type identifier |
| `FileName` | string | File being processed |
| `InvalidReason` | string | Reason for failure/invalid status |
| `Action` | string | Current or last action executed |

## Status Values

| Status | Description | Set By | Action Field |
|--------|-------------|--------|--------------|
| `Failed` | Message validation failed at routing | FileEventHandler | `JobMessageValidation` |
| `Succeeded` | Successfully routed to queue | FileEventHandler | `AssignMessage` |
| `Invalid` | Job configuration invalid | FileJobHandler | varies |
| `ReadMessage` | Job being processed | FileJobHandler | current action |
| `Succeeded` | Job completed successfully | FileJobHandler | last action |
| `Failed` | Job execution failed | FileJobHandler | failed action |

> **Note**: The status value is reused across components. The `Action` field distinguishes the context:
> - `JobMessageValidation` + `Failed` = Message validation failed at FileEventHandler
> - `AssignMessage` + `Succeeded` = Successfully routed to queue
> - Other actions = FileJobHandler processing status

### Status Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        FileEventHandler                          │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                    ┌───────────┴───────────┐
                    ▼                       ▼
              ┌──────────┐           ┌───────────┐
              │  Failed  │           │ Succeeded │
              │(Validation│           │ (Routed   │
              │  failed) │           │  to queue)│
              └──────────┘           └─────┬─────┘
                                           │
┌──────────────────────────────────────────┼──────────────────────┐
│                        FileJobHandler                            │
└──────────────────────────────────────────┼──────────────────────┘
                                           │
                               ┌───────────┼───────────┐
                               ▼           │           ▼
                         ┌─────────┐       │     ┌──────────┐
                         │ Invalid │       │     │  Failed  │
                         │ (Config)│       │     │ (Error)  │
                         └─────────┘       │     └──────────┘
                                           │
                                    ┌──────┴──────┐
                                    ▼             │
                              ┌───────────┐       │
                              │ReadMessage│       │
                              └─────┬─────┘       │
                                    │             │
                                    ▼             │
                              ┌───────────┐       │
                              │ Succeeded │◄──────┘
                              └───────────┘
```

## Table Structure

**Table Name**: `jobstatus`

**Storage Account**: Environment-specific

### Example Records

| PartitionKey | RowKey | Status | Queue | AccountID | FileName |
|--------------|--------|--------|-------|-----------|----------|
| FTP.File.ExternalService-ACC001-file1.csv | evt-123 | Succeeded | jobs | ACC001 | file1.csv |
| FTP.File.ExternalService-ACC002-data.txt | evt-456 | Failed | jobs | ACC002 | data.txt |
| TMS.Route-TMS001-route.xml | evt-789 | Assigned | jobstmsroute | TMS001 | route.xml |

## Query Patterns

### Find Jobs by Account

```
PartitionKey ge 'source-ACCOUNTID' and PartitionKey lt 'source-ACCOUNTIE'
```

### Find Failed Jobs

```
Status eq 'Failed'
```

### Find Jobs by Time Range

```
EventTime ge '2024-01-15T00:00:00' and EventTime lt '2024-01-16T00:00:00'
```

### Find Jobs by Queue

```
Queue eq 'jobstmsroute' and Status eq 'Succeeded'
```

## Status Transitions

### FileEventHandler

1. **Message Received**:
   - If validation fails → `Failed` (Action: `JobMessageValidation`)
   - If route found → `Succeeded` (Action: `AssignMessage`)
   - Note: `NoAssigned` status is defined but not currently used in code

2. **Validation Failed Record** (sets Status=`Failed`):
   ```csharp
   await _jobStatusManipulate.InsertInvalid(Data, PartitionKey, PartitionID, OffsetID, InvalidReason);
   // table.Status = AppConfiguration.FixData_Status_Failed; // "Failed"
   // table.Action = AppConfiguration.FixData_Action_Validation; // "JobMessageValidation"
   ```

3. **Successfully Routed Record** (sets Status=`Succeeded`):
   ```csharp
   await _jobStatusManipulate.InsertAssigned(Data, Notify, PartitionID, OffsetID, PartitionKey);
   // table.Status = AppConfiguration.FixData_Status_Succeeded; // "Succeeded"
   // table.Action = AppConfiguration.FixData_Action_Assign; // "AssignMessage"
   ```

### FileJobHandler

1. **Processing Start**:
   - Update to `ReadMessage`

2. **Processing Complete**:
   - On success → `Succeeded`
   - On failure → `Failed` with `InvalidReason`

## InvalidReason Values

Common reasons stored in the `InvalidReason` field:

| Reason Pattern | Description |
|----------------|-------------|
| `Message is not json format` | Invalid JSON in event |
| `Type,Subject,Source,AccountID not provided` | Missing required fields |
| `Couldn't find Record in Config Table` | No routing rule matched |
| `Try X: {error}. Sleep for Y Second.` | Retry attempts |
| `File Successfully read` | Read operation succeeded |
| `File Successfully write to destination` | Write operation succeeded |
| `Virus Scan Failed ErrorMessage={msg}` | Virus detected |
| `PGP_Decryption method failed` | Decryption error |
| `Couldn't recognise source.type` | Unknown server type |

## Configuration

### appsettings.json

```json
{
  "Tables": {
    "jobstatus": {
      "Url": "https://enterprisedevsftpapps.table.core.windows.net/",
      "SASToken": "sv=2017-04-17&si=full-acl&tn=jobstatus&sig=...",
      "TableName": "jobstatus"
    }
  },
  "FixData": {
    "InvalidStatus": "Invalid",
    "ReadMessageStatus": "ReadMessage",
    "StatusSuccess": "Succeeded",
    "StatusFailed": "Failed"
  }
}
```

## Monitoring and Alerting

### Key Metrics

1. **Failed Job Rate**: Count of `Status eq 'Failed'` per time period
2. **NoAssigned Rate**: Events without routing rules
3. **Processing Latency**: Time between `Assigned` and `Succeeded`
4. **Retry Rate**: Records with multiple "Try X" in `InvalidReason`

### Datadog Queries

```
# Failed jobs in last hour
sum:fts.jobstatus.count{status:failed}.as_count()

# Average processing time
avg:fts.job.processing_time{*}
```

## Troubleshooting

### Finding Stuck Jobs

Jobs stuck in `ReadMessage` for extended periods:

```
Status eq 'ReadMessage' and Timestamp lt datetime'2024-01-15T10:00:00'
```

### Investigating Failures

1. Query for specific job by RowKey
2. Examine `InvalidReason` for error details
3. Check `Message` field for original event
4. Review `Action` for last executed action

### Common Issues

| Symptom | Possible Cause | Resolution |
|---------|----------------|------------|
| Many `NoAssigned` | Missing routing rules | Add entry to jobconfig table |
| Many `Invalid` | Malformed events | Check event publisher |
| Stuck `ReadMessage` | Handler crashed | Check pod logs, restart |
| Repeated `Failed` | Configuration error | Review jobsettings.json |

## Related Documentation

- [Queue Architecture](../architecture/queue-architecture.md)
- [Routing ConfigModel](./routing-config-model.md)
- [Troubleshooting Runbook](../operations/troubleshooting-runbook.md)
- [Monitoring & Logging](../operations/monitoring-logging.md)
