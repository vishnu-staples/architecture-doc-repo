# FileJobHandler-TMSStatus

Specialized handler for Transportation Management System (TMS) status files.

## Repository
`FileJobHandler-TMSStatus/`

## Purpose

FileJobHandler-TMSStatus processes TMS status update files:
- Handles shipment status updates
- Processes tracking information
- Updates TMS systems with delivery status

## Architecture

```
Azure Queue (jobstmsstatus)
       │
       ▼
┌─────────────────────────────┐
│  FileJobHandler-TMSStatus   │
├─────────────────────────────┤
│ QueueListener               │
│ TMSStatusProcessor          │
│ StatusValidation            │
│ ActionProcess               │
└─────────────────────────────┘
       │
       ├──▶ Status Sources
       ├──▶ Status Destinations
       └──▶ TMS Systems
```

## Input Contract

**Queue Message**:
```json
{
  "specversion": "1.0",
  "type": "Succeeded",
  "source": "TMS.Status",
  "subject": "Status Update",
  "data": {
    "accountId": "TRANSPORT001",
    "fileName": "status/shipment_status_20240115.csv",
    "correlationId": "status-corr-456"
  }
}
```

## Output Contract

**Processed Status File**:
- Status records validated
- Tracking updates extracted
- Delivered to TMS systems

**JobStatus Update**:
```json
{
  "Status": "Succeeded",
  "Queue": "jobstmsstatus",
  "Action": "TMSStatusProcess",
  "InvalidReason": "Status processed successfully"
}
```

## Business Logic

### Status Validation

- Validates status file format
- Checks shipment references
- Verifies status codes

### Status Processing

- Parses status records
- Maps to standard status codes
- Prepares update payloads

## Configuration

### Routing Rule

```
PartitionKey: jobstmsstatus
EventType: Succeeded
EventSource: TMS.Status
EventSubject: Status Update
EventAccountID: TRANSPORT001
```

### Job Configuration

```json
{
  "feedName": "tms-status-processing",
  "source": {
    "type": "Vendor-SFTP",
    "host": "sftp.carrier.com",
    "path": "/status/outbound"
  },
  "destinations": [
    {
      "id": 1,
      "type": "BlobStore",
      "path": "tms/status/processed"
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

Receives messages from `jobstmsstatus` queue.

### Kubernetes

```yaml
spec:
  containers:
    - name: file-job-handler-tms-status
      image: enterprisenonpacr.azurecr.io/enterprise/file-job-handler-tms-status:tag
```

## Dependencies

- FileTransferCommon
- TMS-specific libraries (if any)

## Monitoring

### Key Metrics

- Status files processed
- Shipment updates applied
- Processing latency

### Alerts

- Status processing failures
- Invalid status format rate

## Related Documentation

- [FileJobHandler](./file-job-handler.md)
- [Queue Architecture](../architecture/queue-architecture.md)
