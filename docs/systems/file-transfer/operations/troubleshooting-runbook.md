# Troubleshooting Runbook

This runbook provides systematic approaches for diagnosing and resolving FTS issues.

## Triage Flow

```
┌────────────────────────────────────────────────────────────┐
│                      Issue Reported                         │
└─────────────────────────────┬──────────────────────────────┘
                              │
                              ▼
              ┌───────────────────────────────┐
              │    Check jobstatus Table      │
              │    Query by PartitionKey      │
              └───────────────┬───────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
    ┌───────────┐       ┌───────────┐       ┌───────────┐
    │  Invalid  │       │ NoAssigned│       │  Failed   │
    └─────┬─────┘       └─────┬─────┘       └─────┬─────┘
          │                   │                   │
          ▼                   ▼                   ▼
    Check Message       Check Routing       Check Action
    Validation          Configuration       Execution
```

## Common Issues

### 1. Jobs Not Processing

**Symptoms**:
- Files not transferred
- No status updates
- Queue backing up

**Diagnostic Steps**:

1. **Check Event Hub ingestion**:
   ```
   # Datadog query
   service:file-event-handler "Event received"
   ```

2. **Check jobstatus for recent entries**:
   ```
   # Azure Table query
   Timestamp ge datetime'2024-01-15T00:00:00Z'
   ```

3. **Check handler pods**:
   ```bash
   kubectl get pods -n sftp-{env} -l app=file-job-handler
   kubectl logs -n sftp-{env} -l app=file-job-handler --tail=100
   ```

4. **Check queue depth**:
   ```bash
   az storage queue peek \
     --account-name enterprise{env}sftpapps \
     --queue-name jobs \
     --num-messages 10
   ```

**Resolution**:
- Restart handler pods if crashed
- Scale handlers if queue backed up
- Check Azure service health

---

### 2. Invalid Status

**Symptoms**:
- jobstatus shows `Invalid`
- `InvalidReason` contains validation error

**Common Causes**:

| InvalidReason | Cause | Fix |
|---------------|-------|-----|
| "Message is not json format" | Malformed event | Fix event publisher |
| "Type,Subject,Source,AccountID not provided" | Missing fields | Add required fields |
| "Garbage message" | Unrecognized format | Check event source |

**Diagnostic Steps**:

1. **Find Invalid records**:
   ```
   Status eq 'Invalid'
   ```

2. **Examine Message field** (contains original event)

3. **Verify event format** against CloudEvents spec

**Resolution**:
- Fix event publisher to send valid CloudEvents
- Ensure all required fields present

---

### 3. NoAssigned Status

**Symptoms**:
- jobstatus shows `NoAssigned`
- Event received but not routed

**Cause**: No matching rule in `jobconfig` table

**Diagnostic Steps**:

1. **Find NoAssigned records**:
   ```
   Status eq 'NoAssigned'
   ```

2. **Extract event attributes** from record:
   - EventType
   - EventSource
   - EventSubject
   - AccountID

3. **Check jobconfig table** for matching rule

**Resolution**:

Add routing rule to jobconfig table:
```bash
az storage entity insert \
  --account-name enterprise{env}sftpapps \
  --table-name jobconfig \
  --entity \
    PartitionKey=jobs \
    RowKey=new-rule \
    EventType=Succeeded \
    EventSource=FTP.File.ExternalService \
    EventSubject="File Upload" \
    EventAccountID=ACCOUNTID
```

---

### 4. Failed Status

**Symptoms**:
- jobstatus shows `Failed`
- `InvalidReason` contains error details

**Common Failure Patterns**:

| InvalidReason Pattern | Cause | Fix |
|-----------------------|-------|-----|
| "Try X: ... File not found" | Source file missing | Verify file exists |
| "Try X: ... Connection refused" | Server unreachable | Check network/credentials |
| "Virus Scan Failed" | Virus detected | Quarantine file |
| "PGP_Decryption method failed" | Wrong key/password | Update credentials |
| "Couldn't recognise source.type" | Invalid type value | Fix jobsettings.json |
| "Couldn't find jobsettings.json" | Missing config | Create configuration |

**Diagnostic Steps**:

1. **Find Failed record**:
   ```
   Status eq 'Failed' and AccountID eq 'ACCOUNTID'
   ```

2. **Examine InvalidReason** for error details

3. **Check Datadog logs** for full stack trace:
   ```
   service:file-job-handler @messageId:MESSAGE_ID status:error
   ```

4. **Review action that failed** (in `Action` field)

---

### 5. Connection Failures

**Symptoms**:
- "Connection refused"
- "Authentication failed"
- "Timeout"

**Diagnostic Steps**:

1. **Identify target server** from jobsettings.json

2. **Test connectivity** from pod:
   ```bash
   kubectl exec -it <pod> -n sftp-{env} -- nc -zv hostname port
   ```

3. **Verify credentials**:
   - Check jobsecrets.json values
   - Verify Key Vault secrets

4. **Check firewall rules** for target server

**Resolution**:
- Update credentials
- Whitelist pod IP addresses
- Check VPN/ExpressRoute connectivity

---

### 6. PGP/Encryption Failures

**Symptoms**:
- "PGP_Decryption method failed"
- "Key/Vault Private key not found"

**Diagnostic Steps**:

1. **Verify key file exists**:
   ```bash
   kubectl exec -it <pod> -n sftp-{env} -- ls -la /mnt/jobconfig/.../
   ```

2. **Check key file path** in jobsettings.json matches actual location

3. **Verify passphrase** in jobsecrets.json or Key Vault

4. **Test key manually** (if appropriate):
   ```bash
   gpg --decrypt --batch --passphrase "password" file.gpg
   ```

**Resolution**:
- Deploy correct key files
- Update passphrase secrets
- Ensure key type matches (encryption vs signing)

---

### 7. Virus Scan Failures

**Symptoms**:
- "Virus Scan Failed ErrorMessage=..."

**Diagnostic Steps**:

1. **Check ClamAV service**:
   ```bash
   nc -zv CLAMAV_IP CLAMAV_PORT
   ```

2. **Review virus details** in InvalidReason

3. **Quarantine file** if legitimate virus

**Resolution**:
- If false positive: Update ClamAV signatures
- If true positive: Notify source, quarantine file

---

### 8. Queue Backup

**Symptoms**:
- Queue depth increasing
- Processing latency high

**Diagnostic Steps**:

1. **Check queue depth**:
   ```bash
   az storage queue show \
     --account-name enterprise{env}sftpapps \
     --queue-name jobs \
     --query "approximateMessageCount"
   ```

2. **Check handler pod count**:
   ```bash
   kubectl get deployment file-job-handler -n sftp-{env}
   ```

3. **Check for stuck jobs** (ReadMessage status for extended period)

**Resolution**:

1. **Scale handlers**:
   ```bash
   kubectl scale deployment file-job-handler -n sftp-{env} --replicas=5
   ```

2. **Clear poison messages** if necessary:
   ```bash
   az storage message clear \
     --account-name enterprise{env}sftpapps \
     --queue-name jobs-poison
   ```

---

## Azure Table Query Patterns

### Find by Account

```
AccountID eq 'ACCOUNT001'
```

### Find by Status

```
Status eq 'Failed' and Timestamp ge datetime'2024-01-15T00:00:00Z'
```

### Find by Time Range

```
Timestamp ge datetime'2024-01-15T10:00:00Z' and Timestamp lt datetime'2024-01-15T11:00:00Z'
```

### Find by Event Source

```
EventSource eq 'TMS.Route'
```

### Find Recent Failures

```
Status eq 'Failed' and Timestamp ge datetime'2024-01-15T00:00:00Z'
```

## Datadog Log Queries

### Find Errors for Account

```
service:file-job-handler @accountId:ACCOUNT001 status:error
```

### Find Retry Patterns

```
service:file-job-handler "Sleep for * Second" @accountId:ACCOUNT001
```

### Find Slow Operations

```
service:file-job-handler @duration:>60000
```

### Trace Single Job

```
service:file-* @correlationId:CORRELATION_ID
```

## Escalation Path

1. **L1 - Operations**:
   - Check jobstatus table
   - Review Datadog dashboards
   - Basic troubleshooting

2. **L2 - Engineering**:
   - Code-level debugging
   - Configuration review
   - Infrastructure issues

3. **L3 - Architecture**:
   - System design issues
   - Performance optimization
   - Cross-team coordination

## Related Documentation

- [Monitoring & Logging](./monitoring-logging.md)
- [JobStatus Model](../data-models/jobstatus-model.md)
- [Queue Architecture](../architecture/queue-architecture.md)
