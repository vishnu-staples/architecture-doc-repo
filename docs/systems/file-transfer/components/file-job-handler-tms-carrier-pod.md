# FileJobHandler-TMSCarrierPOD

Specialized handler for Transportation Management System (TMS) Carrier Proof of Delivery files.

## Repository
`FileJobHandler-TMSCarrierPOD/`

## Purpose

FileJobHandler-TMSCarrierPOD processes carrier POD (Proof of Delivery) files:
- Handles delivery confirmation documents
- Processes signature captures
- Updates delivery status in TMS

## Architecture

```
Azure Queue (jobtmscarrierpod)
       │
       ▼
┌─────────────────────────────────┐
│  FileJobHandler-TMSCarrierPOD   │
├─────────────────────────────────┤
│ QueueListener                   │
│ PODProcessor                    │
│ PODValidation                   │
│ ActionProcess                   │
└─────────────────────────────────┘
       │
       ├──▶ Carrier Sources
       ├──▶ POD Storage
       └──▶ TMS Systems
```

## Input Contract

**Queue Message**:
```json
{
  "specversion": "1.0",
  "type": "Succeeded",
  "source": "TMS.CarrierPOD",
  "subject": "POD Upload",
  "data": {
    "accountId": "CARRIER001",
    "fileName": "pod/delivery_pod_SHP123456.pdf",
    "correlationId": "pod-corr-789"
  }
}
```

## Output Contract

**Processed POD File**:
- POD document validated
- Linked to shipment record
- Stored in document management

**JobStatus Update**:
```json
{
  "Status": "Succeeded",
  "Queue": "jobtmscarrierpod",
  "Action": "PODProcess",
  "InvalidReason": "POD processed successfully"
}
```

## Business Logic

### POD Validation

- Validates POD document format (PDF, image)
- Extracts shipment reference
- Verifies carrier authentication

### POD Processing

- Links POD to shipment
- Extracts delivery information
- Updates TMS delivery status

## Configuration

### Routing Rule

```
PartitionKey: jobtmscarrierpod
EventType: Succeeded
EventSource: TMS.CarrierPOD
EventSubject: POD Upload
EventAccountID: CARRIER001
```

### Job Configuration

```json
{
  "feedName": "tms-carrier-pod-processing",
  "source": {
    "type": "Vendor-SFTP",
    "host": "sftp.carrier.com",
    "path": "/pod/uploads"
  },
  "destinations": [
    {
      "id": 1,
      "type": "BlobStore",
      "path": "tms/pod/documents"
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
      "type": "Scan",
      "sourceStream": "{@JobSettings.actions.1.outputStream}"
    },
    {
      "id": 2,
      "type": "WriteFile",
      "sourceStream": "{@JobSettings.actions.1.outputStream}",
      "outputFile": "{@JobSettings.destinations.1.path}/{@Message.fileName}"
    }
  ]
}
```

## Deployment

### Queue Assignment

Receives messages from `jobtmscarrierpod` queue.

### Kubernetes

```yaml
spec:
  containers:
    - name: file-job-handler-tms-carrier-pod
      image: enterprisenonpacr.azurecr.io/enterprise/file-job-handler-tms-carrier-pod:tag
```

## Dependencies

- FileTransferCommon
- TMS-specific libraries (if any)
- PDF processing libraries (if needed)

## Monitoring

### Key Metrics

- POD files processed
- Shipments confirmed
- Processing latency

### Alerts

- POD validation failures
- Unmatched shipment references

## Related Documentation

- [FileJobHandler](./file-job-handler.md)
- [Queue Architecture](../architecture/queue-architecture.md)
