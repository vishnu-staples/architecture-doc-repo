# Configuration Management

This guide covers managing FTS configuration across environments.

## Configuration Layers

```
┌─────────────────────────────────────────────┐
│ Application Settings (appsettings.json)     │ ◄── K8s Secret
├─────────────────────────────────────────────┤
│ Job Configuration (jobsettings.json)        │ ◄── Azure File Share
├─────────────────────────────────────────────┤
│ Job Secrets (jobsecrets.json)               │ ◄── Azure File Share / Key Vault
├─────────────────────────────────────────────┤
│ Routing Configuration (jobconfig table)     │ ◄── Azure Table Storage
└─────────────────────────────────────────────┘
```

## Application Settings

### File Location

Application settings are deployed as Kubernetes secrets mounted at:
```
/app/appsettings.Production.json
```

### Configuration Sections

#### Event Hub

```json
{
  "EventHub": {
    "EnterpriseFTP": {
      "ConnectionString": "Endpoint=sb://...",
      "EventHubName": "enterprise-{env}-ftp-ehb"
    }
  }
}
```

#### Queues

```json
{
  "Queues": {
    "jobs": {
      "Url": "https://{storage}.queue.core.windows.net/jobs",
      "SASToken": "sv=2018-03-28&si=..."
    },
    "NextCheck": 1000,
    "NextWriteForLogs": 60,
    "VisibilityTimeout": 120
  }
}
```

#### Tables

```json
{
  "Tables": {
    "jobstatus": {
      "Url": "https://{storage}.table.core.windows.net/",
      "SASToken": "sv=2017-04-17&si=...",
      "TableName": "jobstatus"
    },
    "jobconfig": {
      "Url": "https://{storage}.table.core.windows.net/",
      "SASToken": "sv=2017-04-17&si=...",
      "TableName": "jobconfig"
    }
  }
}
```

#### Key Vault

```json
{
  "KeyVaultSecrets": {
    "Enable": true,
    "Uri": "https://fts-{env}-key-vault.vault.azure.net/",
    "TanentID": "tenant-guid"
  }
}
```

#### Azure Files

```json
{
  "AzureFiles": {
    "sftpintroot": "/mnt/sftpintroot",
    "sftpextroot": "/mnt/sftpextroot",
    "jobsettingsfolder": "/mnt/jobconfig/{@Message.source}/{@Message.accountID}/{@Message.fileName.subFolder}",
    "ReplaceText": {
      "Message_fileName": "{@Message.fileName}",
      "Message_accountID": "{@Message.accountID}"
    }
  }
}
```

#### Actions

```json
{
  "Actions": {
    "SourceTypes": {
      "Internal_Type": "Internal-SFTP",
      "External-SFTP": "External-SFTP",
      "Vendor-SFTP": "Vendor-SFTP",
      "BlobStore": "BlobStore"
    },
    "Types": {
      "ReadFile": "ReadFile",
      "WriteFile": "WriteFile",
      "PGP-Decryption": "PGP-Decrypt"
    },
    "RetryFailureSource": 3,
    "RetryFailureDestination": 3,
    "RetryFailureSourceDelay": 5,
    "RetryFailureDestinationDelay": 5
  }
}
```

#### Virus Scan

```json
{
  "VirusScan": {
    "ClamAVServiceIP": "10.21.129.142",
    "ClamAVServicePort": "3310"
  }
}
```

### Environment Differences

| Setting | Dev | Stg | Prod |
|---------|-----|-----|------|
| Event Hub | enterprise-dev-ftp-ehb | enterprise-stg-ftp-ehb | enterprise-prod-ftp-ehb |
| Storage | enterprisedevsftpapps | enterprisestgsftpapps | enterpriseprodsftpapps |
| Key Vault | fts-dev-key-vault | fts-stg-key-vault | fts-prod-key-vault |
| ACR | enterprisenonpacr | enterprisenonpacr | enterpriseprodacr |

## Job Configuration

### File Structure

```
/mnt/jobconfig/
└── {EventSource}/
    └── {AccountID}/
        └── {SubFolder}/
            ├── jobsettings.json
            ├── jobsecrets.json
            └── [key files]
```

### Creating New Job Configuration

1. **Determine Path**:
   ```
   /mnt/jobconfig/FTP.File.ExternalService/NEWACCOUNT/incoming/
   ```

2. **Create jobsettings.json**:
   ```json
   {
     "feedName": "new-account-feed",
     "version": "1.0",
     "source": { ... },
     "destinations": [ ... ],
     "actions": [ ... ]
   }
   ```

3. **Create jobsecrets.json**:
   ```json
   {
     "username": "actual-username",
     "password": "actual-password"
   }
   ```

4. **Deploy Key Files** (if needed):
   - PGP keys: `private.key`, `public.key`
   - SSH keys: `id_rsa`

### Updating Job Configuration

1. **Access Azure File Share**:
   - Via Azure Portal
   - Via Azure Storage Explorer
   - Via mounted share on pods

2. **Edit Configuration**:
   - Modify JSON file
   - Save changes

3. **Changes Take Effect**:
   - Immediately on next job execution
   - No service restart required

## Routing Configuration

### jobconfig Table Schema

| Column | Type | Description |
|--------|------|-------------|
| PartitionKey | string | Queue name (e.g., "jobs") |
| RowKey | string | Unique rule identifier |
| EventType | string | Event type to match |
| EventSource | string | Event source to match |
| EventSubject | string | Event subject to match |
| EventAccountID | string | Account ID to match |

### Adding Routing Rule

#### Azure Portal

1. Navigate to Storage Account → Tables → jobconfig
2. Add entity with required fields
3. Set PartitionKey to target queue

#### Azure CLI

```bash
az storage entity insert \
  --account-name enterprisedevsftpapps \
  --table-name jobconfig \
  --entity \
    PartitionKey=jobs \
    RowKey=new-account-rule \
    EventType=Succeeded \
    EventSource=FTP.File.ExternalService \
    EventSubject="File Upload" \
    EventAccountID=NEWACCOUNT
```

#### Azure Storage Explorer

1. Connect to storage account
2. Navigate to Tables → jobconfig
3. Click "Add Entity"
4. Fill in required fields

### Queue Name Reference

| PartitionKey | Handler |
|--------------|---------|
| jobs | FileJobHandler |
| jobstmsroute | FileJobHandler-TMSRoute |
| jobstmsstatus | FileJobHandler-TMSStatus |
| jobtmscarrierpod | FileJobHandler-TMSCarrierPOD |
| jobsbylcppricing | FileJobHandler-PriceProposal |

## Secrets Management

### Option 1: jobsecrets.json

Store secrets in file alongside jobsettings.json:

```json
{
  "vendorUser": "username",
  "vendorPass": "password",
  "pgpPassword": "key-passphrase"
}
```

Reference in jobsettings.json:
```json
{
  "credentials": {
    "username": "{@JobSecrets.vendorUser}",
    "password": "{@JobSecrets.vendorPass}"
  }
}
```

### Option 2: Azure Key Vault

Enable Key Vault in appsettings:
```json
{
  "KeyVaultSecrets": {
    "Enable": true,
    "Uri": "https://fts-dev-key-vault.vault.azure.net/"
  }
}
```

Store secrets in Key Vault:
```bash
az keyvault secret set \
  --vault-name fts-dev-key-vault \
  --name vendorPass \
  --value "actual-password"
```

Reference in jobsettings.json:
```json
{
  "credentials": {
    "password": "{@JobSecrets.vendorPass}"
  }
}
```

## Configuration Validation

### JSON Syntax

```bash
# Validate JSON syntax
cat jobsettings.json | jq .
```

### Required Fields

Verify presence of:
- `feedName`
- `source.type`
- `destinations[].id`
- `destinations[].type`
- `actions[].id`
- `actions[].type`

### Placeholder References

Ensure all placeholders resolve:
- `{@JobSettings.xxx}` - References within config
- `{@JobSecrets.xxx}` - References to secrets file
- `{@Message.xxx}` - References to event fields
- `{@AppSettings.xxx}` - References to app settings

## Best Practices

1. **Version Control**: Track configurations in git (excluding secrets)
2. **Environment Parity**: Maintain similar structures across environments
3. **Secret Rotation**: Plan for credential rotation
4. **Documentation**: Comment complex configurations
5. **Testing**: Test in dev before promoting to stg/prod

## Related Documentation

- [Deployment Guide](./deployment-guide.md)
- [Security & Secrets](./security-secrets.md)
- [Jobsettings Storage](../job-configuration/jobsettings-storage.md)
- [Routing ConfigModel](../data-models/routing-config-model.md)
