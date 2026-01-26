# FileJobHandler-PriceProposal

Specialized handler for pricing proposal files from LCP (Logistics Cost Platform) and related systems.

## Repository
`FileJobHandler-PriceProposal/`

## Purpose

FileJobHandler-PriceProposal processes pricing files:
- Handles price proposal uploads
- Processes LCP pricing data
- Manages pricing update workflows

## Architecture

```
Azure Queue (jobsbylcppricing)
       │
       ▼
┌────────────────────────────────────┐
│  FileJobHandler-PriceProposal      │
├────────────────────────────────────┤
│ QueueListener                      │
│ PriceProposalProcessor             │
│ PriceValidation                    │
│ ActionProcess                      │
└────────────────────────────────────┘
       │
       ├──▶ Pricing Sources
       ├──▶ Pricing Storage
       └──▶ Pricing Systems
```

## Input Contract

**Queue Message**:
```json
{
  "specversion": "1.0",
  "type": "Succeeded",
  "source": "Pricing.LCP",
  "subject": "Price Update",
  "data": {
    "accountId": "PRICING001",
    "fileName": "pricing/proposal_2024Q1.xlsx",
    "correlationId": "price-corr-101"
  }
}
```

## Output Contract

**Processed Pricing File**:
- Price data validated
- Format normalized
- Delivered to pricing systems

**JobStatus Update**:
```json
{
  "Status": "Succeeded",
  "Queue": "jobsbylcppricing",
  "Action": "PriceProposalProcess",
  "InvalidReason": "Price proposal processed successfully"
}
```

## Business Logic

### Price Validation

- Validates pricing file format
- Checks required pricing fields
- Verifies price ranges and units

### Price Processing

- Parses pricing data
- Normalizes price formats
- Applies business rules

## Configuration

### Routing Rule

```
PartitionKey: jobsbylcppricing
EventType: Succeeded
EventSource: Pricing.LCP
EventSubject: Price Update
EventAccountID: PRICING001
```

### Job Configuration

```json
{
  "feedName": "price-proposal-processing",
  "source": {
    "type": "Vendor-SFTP",
    "host": "sftp.lcpvendor.com",
    "path": "/pricing/proposals"
  },
  "destinations": [
    {
      "id": 1,
      "type": "BlobStore",
      "path": "pricing/processed"
    },
    {
      "id": 1,
      "type": "Internal-APIM",
      "TargetUrl": "https://api.example.com/v1/pricing/import"
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
    },
    {
      "id": 2,
      "type": "Notification",
      "sourceFile": "{@JobSettings.destinations.1.path}/{@Message.fileName}",
      "outputFile": "{@JobSettings.destinations.1.path}",
      "eventAccountId": "{@Message.accountID}"
    }
  ]
}
```

## Deployment

### Queue Assignment

Receives messages from `jobsbylcppricing` queue.

### Kubernetes

```yaml
spec:
  containers:
    - name: file-job-handler-price-proposal
      image: enterprisenonpacr.azurecr.io/enterprise/file-job-handler-price-proposal:tag
```

## Dependencies

- FileTransferCommon
- Pricing-specific libraries (if any)
- Excel processing libraries (if needed)

## Monitoring

### Key Metrics

- Pricing files processed
- Price updates applied
- Processing latency

### Alerts

- Price validation failures
- Invalid pricing format rate
- API notification failures

## Related Documentation

- [FileJobHandler](./file-job-handler.md)
- [Queue Architecture](../architecture/queue-architecture.md)
