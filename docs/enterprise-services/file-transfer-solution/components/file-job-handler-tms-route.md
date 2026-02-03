# FileJobHandler-TMSRoute

Specialized handler for Transportation Management System (TMS) routing files.

## Repository
`FileJobHandler-TMSRoute/`

## Purpose

FileJobHandler-TMSRoute processes TMS routing files with domain-specific business logic:
- Handles TMS route file formats
- Applies routing-specific transformations
- Integrates with TMS systems

## Architecture

```
Azure Queue (jobstmsroute)
       │
       ▼
┌─────────────────────────────┐
│  FileJobHandler-TMSRoute    │
├─────────────────────────────┤
│ QueueListener               │
│ TMSRouteProcessor           │
│ RouteValidation             │
│ ActionProcess               │
└─────────────────────────────┘
       │
       ├──▶ TMS Sources
       ├──▶ Route Destinations
       └──▶ TMS Systems
```

## Input Contract

**Queue Message**:
```json
{
  "specversion": "1.0",
  "type": "Succeeded",
  "source": "TMS.Route",
  "subject": "File Upload",
  "data": {
    "accountId": "TRANSPORT001",
    "fileName": "routes/daily/route_20240115.xml",
    "correlationId": "route-corr-123"
  }
}
```

## Output Contract

**Processed Route File**:
- Route data validated and transformed
- Delivered to configured destinations
- TMS system notified

**JobStatus Update**:
```json
{
  "Status": "Succeeded",
  "Queue": "jobstmsroute",
  "Action": "TMSRouteProcess",
  "InvalidReason": "Route processed successfully"
}
```

## Business Logic

### Route Validation

- Validates route file structure
- Checks required route fields
- Verifies carrier information

### Route Transformation

- Normalizes route data format
- Applies business rules
- Enriches with reference data

## Configuration

### Routing Rule

```
PartitionKey: jobstmsroute
EventType: Succeeded
EventSource: TMS.Route
EventSubject: File Upload
EventAccountID: TRANSPORT001
```

### Job Configuration

```json
{
  "feedName": "tms-route-processing",
  "source": {
    "type": "Internal-SFTP",
    "path": "{@AppSettings.sftpintroot}/tms/{@Message.accountID}/routes"
  },
  "destinations": [
    {
      "id": 1,
      "type": "BlobStore",
      "path": "tms/processed/routes"
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
      "type": "WriteFile",
      "sourceStream": "{@JobSettings.actions.1.outputStream}",
      "outputFile": "{@JobSettings.destinations.1.path}/{@Message.fileName}"
    }
  ]
}
```

## Deployment

### Queue Assignment

Receives messages from `jobstmsroute` queue.

### Kubernetes

```yaml
spec:
  containers:
    - name: file-job-handler-tms-route
      image: enterprisenonpacr.azurecr.io/enterprise/file-job-handler-tms-route:tag
```

## Dependencies

- FileTransferCommon
- TMS-specific libraries (if any)

## Monitoring

### Key Metrics

- Routes processed per hour
- Route validation failures
- Processing latency

### Alerts

- Route validation failure rate > threshold
- Processing queue backlog

## Related Documentation

- [FileJobHandler](./file-job-handler.md)
- [Queue Architecture](../architecture/queue-architecture.md)
