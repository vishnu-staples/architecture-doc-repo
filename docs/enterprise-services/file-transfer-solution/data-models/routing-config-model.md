# Routing ConfigModel

The Routing ConfigModel (distinct from Job ConfigModel) defines Azure Table entities that map incoming events to processing queues.

## Source File
`FileEventHandler/FileEventHandler.Service/Models/ConfigModel.cs`

## Schema

```csharp
public class ConfigModel : TableEntity
{
    public string EventType { get; set; }
    public string EventSource { get; set; }
    public string EventSubject { get; set; }
    public string EventAccountID { get; set; }
}
```

## Azure Table Entity Properties

The ConfigModel extends `TableEntity`, inheriting standard Azure Table properties.

### Inherited Properties

| Property | Type | Description |
|----------|------|-------------|
| `PartitionKey` | string | Queue assignment key (e.g., "jobs", "jobstmsroute") |
| `RowKey` | string | Unique identifier for the routing rule |
| `Timestamp` | DateTimeOffset | Last modified time (auto-managed) |
| `ETag` | string | Entity tag for optimistic concurrency |

### Custom Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `EventType` | string | Yes | Event type to match (e.g., "Succeeded") |
| `EventSource` | string | Yes | Event source to match (e.g., "FTP.File.ExternalService") |
| `EventSubject` | string | Yes | Event subject to match (e.g., "File Upload") |
| `EventAccountID` | string | Yes | Account ID to match (e.g., "ACCOUNT001") |

## Table Structure

**Table Name**: `jobconfig`

**Storage Account**: Environment-specific (dev/stg/prod)

### Example Table Data

| PartitionKey | RowKey | EventType | EventSource | EventSubject | EventAccountID |
|--------------|--------|-----------|-------------|--------------|----------------|
| jobs | rule-001 | Succeeded | FTP.File.ExternalService | File Upload | ACCOUNT001 |
| jobs | rule-002 | Succeeded | FTP.File.ExternalService | File Upload | ACCOUNT002 |
| jobstmsroute | rule-003 | Succeeded | TMS.Route | File Upload | TMS001 |
| jobstmsstatus | rule-004 | Succeeded | TMS.Status | Status Update | TMS001 |
| jobsbylcppricing | rule-005 | Succeeded | Pricing.LCP | Price Update | PRICING001 |

## Routing Logic

The FileEventHandler queries the `jobconfig` table to determine where to route each event.

### Lookup Process

```csharp
// From MessageHandler.cs
PartitionKey = _tableHelper.SearchConfigTable(
    Notify.data.accountId,  // EventAccountID
    Notify.source,          // EventSource
    Notify.type,            // EventType
    Notify.subject,         // EventSubject
    OriginalOffsetId
);
```

### Matching Rules

1. All four attributes must match exactly
2. Matching is case-sensitive
3. First matching rule is used
4. If no match found, event is marked as `NoAssigned`

## PartitionKey Values (Queue Names)

The `PartitionKey` maps directly to queue names:

| PartitionKey | Queue | Handler |
|--------------|-------|---------|
| `jobs` | jobs | FileJobHandler |
| `jobstmsroute` | jobstmsroute | FileJobHandler-TMSRoute |
| `jobstmsstatus` | jobstmsstatus | FileJobHandler-TMSStatus |
| `jobtmscarrierpod` | jobtmscarrierpod | FileJobHandler-TMSCarrierPOD |
| `jobsbylcppricing` | jobsbylcppricing | FileJobHandler-PriceProposal |
| `jobsbellusermanagement` | jobsbellusermanagement | FileJobHandler |
| `jobsbellsrs` | jobsbellsrs | FileJobHandler |
| `jobsbellpricegrid` | jobsbellpricegrid | FileJobHandler |
| `jobscalgary` | jobscalgary | FileJobHandler |
| `jobsdigitalml` | jobsdigitalml | FileJobHandler |
| `jobspunchcard` | jobspunchcard | FileJobHandler |
| `jobtmsrateshop` | jobtmsrateshop | FileJobHandler |
| `jobspostalmate` | jobspostalmate | FileJobHandler |
| `jobhrvariancereport` | jobhrvariancereport | FileJobHandler |
| `jobmonerisupdater` | jobmonerisupdater | FileJobHandler |
| `jobstatfloklaviyo` | jobstatfloklaviyo | FileJobHandler |
| `jobamazontax` | jobamazontax | FileJobHandler |
| `jobamzskulist` | jobamzskulist | FileJobHandler |

## Configuration Management

### Adding a New Route

1. Determine the target queue (`PartitionKey`)
2. Create unique `RowKey`
3. Define matching criteria (EventType, EventSource, EventSubject, EventAccountID)
4. Insert entity into `jobconfig` table

### Azure Portal / Storage Explorer

Navigate to:
- Storage Account: `enterprise{env}sftpapps`
- Tables: `jobconfig`
- Insert/Edit entities

### Using Azure CLI

```bash
# Insert routing rule
az storage entity insert \
  --account-name enterprisedevsftpapps \
  --table-name jobconfig \
  --entity PartitionKey=jobs RowKey=rule-new \
    EventType=Succeeded \
    EventSource=FTP.File.ExternalService \
    EventSubject="File Upload" \
    EventAccountID=NEWACCOUNT
```

## Disambiguation Note

**Important**: There are two different `ConfigModel` classes in the codebase:

| Context | File | Purpose |
|---------|------|---------|
| **Job ConfigModel** | `FileJobHandler.Service/Model/ConfigModel.cs` | Job definition (jobsettings.json) |
| **Routing ConfigModel** | `FileEventHandler.Service/Models/ConfigModel.cs` | Event routing (jobconfig table) |

Throughout FTS documentation:
- "**Job ConfigModel**" refers to job definitions
- "**Routing ConfigModel**" refers to this routing table entity

## Example Scenarios

### Scenario 1: New Account Setup

To route events from a new account `ACME001` to the generic `jobs` queue:

```json
{
  "PartitionKey": "jobs",
  "RowKey": "acme-file-upload",
  "EventType": "Succeeded",
  "EventSource": "FTP.File.ExternalService",
  "EventSubject": "File Upload",
  "EventAccountID": "ACME001"
}
```

### Scenario 2: Specialized Handler

To route TMS route files from account `TRANSPORT001` to the TMS Route handler:

```json
{
  "PartitionKey": "jobstmsroute",
  "RowKey": "transport-tms-route",
  "EventType": "Succeeded",
  "EventSource": "TMS.Route",
  "EventSubject": "File Upload",
  "EventAccountID": "TRANSPORT001"
}
```

### Scenario 3: Multiple Event Types

An account may need different handlers for different event types:

```json
// Standard file uploads -> generic handler
{
  "PartitionKey": "jobs",
  "RowKey": "multi-standard",
  "EventType": "Succeeded",
  "EventSource": "FTP.File.ExternalService",
  "EventSubject": "File Upload",
  "EventAccountID": "MULTI001"
}

// Pricing updates -> pricing handler
{
  "PartitionKey": "jobsbylcppricing",
  "RowKey": "multi-pricing",
  "EventType": "Succeeded",
  "EventSource": "Pricing.LCP",
  "EventSubject": "Price Update",
  "EventAccountID": "MULTI001"
}
```

## Related Documentation

- [Queue Architecture](../architecture/queue-architecture.md)
- [Event-Driven Architecture](../architecture/event-driven-architecture.md)
- [Job ConfigModel](./job-config-model.md)
