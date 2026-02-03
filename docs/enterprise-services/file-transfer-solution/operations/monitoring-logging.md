# Monitoring & Logging

This guide covers monitoring FTS components using Datadog APM and NLog.

## Monitoring Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     FTS Components                           │
├─────────────────────────────────────────────────────────────┤
│ FileEventHandler │ FileJobHandler │ Specialized Handlers    │
└────────┬─────────┴────────┬───────┴────────────┬────────────┘
         │                  │                    │
         ▼                  ▼                    ▼
    ┌─────────────────────────────────────────────────────┐
    │                   Datadog Agent                      │
    │                 (APM + Logs)                         │
    └─────────────────────────┬───────────────────────────┘
                              │
                              ▼
    ┌─────────────────────────────────────────────────────┐
    │                   Datadog Cloud                      │
    │    Dashboards │ Alerts │ Traces │ Log Explorer      │
    └─────────────────────────────────────────────────────┘
```

## Datadog APM

### Integration

FTS components include Datadog APM via build arguments:

```dockerfile
ARG ARG_APM_VENDOR=datadog
```

### Trace Context

Traces propagate through:
- Event Hub messages
- Queue messages
- HTTP calls to APIs

### Key Traces

| Service | Operation | Description |
|---------|-----------|-------------|
| FileEventHandler | EventHub.Receive | Event ingestion |
| FileEventHandler | Table.Query | Routing lookup |
| FileEventHandler | Queue.Send | Message routing |
| FileJobHandler | Queue.Receive | Job pickup |
| FileJobHandler | Action.Execute | Action processing |
| FileJobHandler | Server.ReadFile | File read |
| FileJobHandler | Server.WriteFile | File write |

## NLog Configuration

### Log Levels

| Level | Usage |
|-------|-------|
| Debug | Detailed debugging (disabled in prod) |
| Info | Normal operation flow |
| Warning | Recoverable issues |
| Error | Operation failures |

### Log Format

```csharp
Logger.Info($"Start ActionProcess Action method ActionID={action.id}, ActionType={action.type}. MessageId={currentMessageInfo.MessageId}");
```

### Example Logs

**Job Start**:
```
INFO: Start Process MessageId=abc-123, Source=FTP.File.ExternalService, AccountID=ACCOUNT001
```

**Action Execution**:
```
INFO: Start Action ReadFile by ActionID=0, ActionType=ReadFile. MessageId=abc-123
INFO: ReadFile method ended. File Successfully read
```

**Action Failure**:
```
ERROR: Action_Read method failed. ActionID=0, ActionType=ReadFile. MessageId=abc-123
```

## Key Metrics

### FileEventHandler

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| Events received/sec | Event ingestion rate | < 10% of normal |
| Invalid message rate | Validation failures | > 5% |
| Routing success rate | Messages routed | < 95% |
| Queue send latency | Message publish time | > 1000ms |

### FileJobHandler

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| Jobs processed/min | Processing throughput | < 50% of normal |
| Job success rate | Completed successfully | < 95% |
| Action failure rate | Individual actions | > 10% |
| Processing latency | End-to-end time | > 60s |
| Retry rate | Operations retried | > 20% |

### Infrastructure

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| Pod CPU usage | Container CPU | > 80% |
| Pod memory usage | Container memory | > 80% |
| Queue depth | Messages pending | > 1000 |
| Storage latency | Azure File ops | > 500ms |

## Dashboards

### Operations Dashboard

Key widgets:
- Job success/failure rate (time series)
- Events processed (counter)
- Queue depth (gauge)
- Error rate by action type (pie chart)
- Top failing accounts (table)

### Health Dashboard

Key widgets:
- Service uptime (SLO)
- Pod status (status map)
- Resource utilization (heatmap)
- Azure service health (status)

## Alerts

### Critical Alerts

| Alert | Condition | Action |
|-------|-----------|--------|
| Service Down | No events in 5 min | Page on-call |
| High Error Rate | > 10% failures | Notify team |
| Queue Backup | > 10000 messages | Scale handlers |
| Pod CrashLoop | > 3 restarts/5min | Page on-call |

### Warning Alerts

| Alert | Condition | Action |
|-------|-----------|--------|
| Elevated Latency | p95 > 30s | Monitor |
| Retry Rate High | > 20% | Investigate |
| Memory Pressure | > 70% | Plan scaling |

### Alert Configuration (Datadog)

```yaml
# Example monitor
name: FTS Job Failure Rate
type: metric alert
query: |
  sum(last_5m):sum:fts.jobs.failed{*}.as_count() /
  sum:fts.jobs.total{*}.as_count() * 100 > 10
message: |
  Job failure rate exceeded 10% in the last 5 minutes.

  Check Datadog logs for error details.
  Review jobstatus table for failed jobs.

  @slack-fts-alerts
options:
  thresholds:
    critical: 10
    warning: 5
```

## Log Queries

### Find Failed Jobs

```
service:file-job-handler status:error @level:ERROR
```

### Find Jobs by Account

```
service:file-job-handler @accountId:ACCOUNT001
```

### Find Slow Operations

```
service:file-job-handler @duration:>30000
```

### Find Retry Events

```
service:file-job-handler "Sleep for * Second"
```

## Troubleshooting with Logs

### Job Not Processing

1. Check FileEventHandler for routing:
   ```
   service:file-event-handler @accountId:ACCOUNT001
   ```

2. Check if message reached queue:
   ```
   service:file-event-handler "Message Sent to Queue"
   ```

3. Check handler pickup:
   ```
   service:file-job-handler "Start Process"
   ```

### Action Failures

1. Find the failed action:
   ```
   service:file-job-handler @messageId:abc-123 status:error
   ```

2. Check retry attempts:
   ```
   service:file-job-handler @messageId:abc-123 "Try *"
   ```

3. View full trace:
   - Click on error log
   - Navigate to associated trace
   - Review span details

### Performance Issues

1. Identify slow jobs:
   ```
   service:file-job-handler @duration:>60000
   ```

2. Break down by action:
   ```
   service:file-job-handler "method ended" @duration:*
   ```

3. Check external dependencies:
   ```
   service:file-job-handler ("Vendor-SFTP" OR "Vendor-FTP") @duration:*
   ```

## Log Retention

| Environment | Retention | Archive |
|-------------|-----------|---------|
| Development | 7 days | None |
| Staging | 14 days | None |
| Production | 30 days | S3 (1 year) |

## Related Documentation

- [Troubleshooting Runbook](./troubleshooting-runbook.md)
- [JobStatus Model](../data-models/jobstatus-model.md)
- [Deployment Guide](./deployment-guide.md)
