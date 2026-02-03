# Security & Secrets Management

This guide covers security practices and secrets management for FTS.

## Security Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      Security Layers                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │ Transport       │  │ Authentication  │  │ Encryption      │  │
│  │ Security        │  │                 │  │                 │  │
│  ├─────────────────┤  ├─────────────────┤  ├─────────────────┤  │
│  │ - TLS 1.2+      │  │ - SAS Tokens    │  │ - PGP           │  │
│  │ - SFTP/SSH      │  │ - SSH Keys      │  │ - RSA           │  │
│  │ - FTPS          │  │ - OAuth 2.0     │  │ - AES           │  │
│  │ - HTTPS         │  │ - Managed ID    │  │                 │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                    Secrets Storage                           ││
│  │  Azure Key Vault │ jobsecrets.json │ K8s Secrets            ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

## Secrets Storage Options

### Option 1: jobsecrets.json (File-based)

**Location**: Same directory as jobsettings.json

```
/mnt/jobconfig/{source}/{accountID}/{folder}/
├── jobsettings.json
├── jobsecrets.json    ◄── Secrets here
└── private.key
```

**Format**:
```json
{
  "vendorUser": "actual-username",
  "vendorPass": "actual-password",
  "pgpPassword": "key-passphrase",
  "blobConnection": "DefaultEndpointsProtocol=https;..."
}
```

**Usage in jobsettings.json**:
```json
{
  "credentials": {
    "username": "{@JobSecrets.vendorUser}",
    "password": "{@JobSecrets.vendorPass}"
  }
}
```

**Pros**:
- Simple deployment
- No external dependencies
- Job-specific isolation

**Cons**:
- Secrets in file system
- Manual rotation
- No audit trail

### Option 2: Azure Key Vault

**Configuration** (appsettings.json):
```json
{
  "KeyVaultSecrets": {
    "Enable": true,
    "Uri": "https://fts-{env}-key-vault.vault.azure.net/",
    "TanentID": "tenant-guid"
  }
}
```

**How It Works**:

Key Vault secrets must contain a JSON object (not individual values):

1. Create a Key Vault secret with a JSON blob as the value:
   ```json
   {
     "vendorUser": "actual-username",
     "vendorPass": "actual-password",
     "pgpPassword": "key-passphrase"
   }
   ```

2. Reference the secret name in jobsettings.json via the `jobsecret` field:
   ```json
   {
     "feedName": "vendor-intake",
     "jobsecret": "VENDOR001-secrets",
     "source": {
       "credentials": {
         "username": "{@JobSecrets.vendorUser}",
         "password": "{@JobSecrets.vendorPass}"
       }
     }
   }
   ```

**Resolution Flow**:
1. If `KeyVaultSecrets.Enable` is true AND `jobsecret` field is set:
   - Retrieve the Key Vault secret by the name in `jobsecret`
   - Deserialize the secret value as JSON dictionary
2. If Key Vault lookup fails or is disabled:
   - Fall back to `jobsecrets.json` file
3. `{@JobSecrets.*}` placeholders look up keys from the loaded dictionary

> **Important**: The `{@KeyVault.*}` placeholder syntax is NOT supported. Always use `{@JobSecrets.*}` with the `jobsecret` field.

**Pros**:
- Centralized management
- Audit logging
- Automatic rotation support
- Access policies

**Cons**:
- Additional dependency
- Network latency
- Cost per operation

### Option 3: Kubernetes Secrets

Used for application settings:

```bash
kubectl create secret -n sftp-dev generic \
  sftp-dev-app-settings-secret \
  --from-file=appsettings.Production.json
```

**Mounted in pod**:
```yaml
volumeMounts:
  - name: appsettings
    mountPath: /app/appsettings.Production.json
    subPath: appsettings.Production.json
```

## Encryption Keys

### PGP Keys

**Storage Location**:
```
/mnt/jobconfig/{source}/{accountID}/{folder}/
├── private.key     # PGP private key for decryption
└── public.key      # PGP public key for encryption
```

**Key Generation** (outside FTS):
```bash
# Generate key pair
gpg --gen-key

# Export private key
gpg --export-secret-keys -a "key-id" > private.key

# Export public key
gpg --export -a "key-id" > public.key
```

**Configuration Reference**:
```json
{
  "actions": [
    {
      "type": "PGP-Decrypt",
      "keyFile": "{@AppSettings.jobsettingsfolder}/private.key",
      "keypassword": "{@JobSecrets.pgpPassword}"
    }
  ]
}
```

**Best Practices**:
- Use strong passphrases
- Rotate keys annually
- Separate encryption and signing keys
- Store backup copies securely

### SSH Keys

**Storage Location**:
```
/mnt/jobconfig/{source}/{accountID}/{folder}/
└── id_rsa          # SSH private key
```

**Key Generation**:
```bash
ssh-keygen -t rsa -b 4096 -f id_rsa -N ""
```

**Configuration Reference**:
```json
{
  "credentials": {
    "type": "SSHKey",
    "username": "sftp-user",
    "sshkey": "{@AppSettings.jobsettingsfolder}/id_rsa"
  }
}
```

**Best Practices**:
- Use RSA 4096-bit or Ed25519
- Rotate keys annually
- Register public key with vendor
- Restrict file permissions (600)

### RSA Keys

**Storage Location**:
```
/mnt/jobconfig/{source}/{accountID}/{folder}/
└── rsa_public.pem  # RSA public key for encryption
```

**Configuration Reference**:
```json
{
  "actions": [
    {
      "type": "RSA-Encrypt",
      "keyFile": "{@AppSettings.jobsettingsfolder}/rsa_public.pem"
    }
  ]
}
```

## Credential Rotation

### Password Rotation

1. **Update target system** with new password
2. **Update FTS credentials**:
   - If using jobsecrets.json: Update file
   - If using Key Vault: Update secret
3. **No service restart required** - next job uses new credentials

### SSH Key Rotation

1. **Generate new key pair**
2. **Register new public key** with vendor
3. **Deploy new private key** to FTS
4. **Test connectivity**
5. **Remove old public key** from vendor

### PGP Key Rotation

1. **Generate new key pair**
2. **Exchange public keys** with partner
3. **Deploy new private key** to FTS
4. **Coordinate cutover date**
5. **Archive old keys**

### SAS Token Rotation

1. **Generate new SAS token** in Azure Portal
2. **Update appsettings.json** with new token
3. **Redeploy Kubernetes secret**
4. **Rolling restart** of pods

```bash
# Update secret
kubectl delete secret -n sftp-dev sftp-dev-app-settings-secret
kubectl create secret -n sftp-dev generic \
  sftp-dev-app-settings-secret \
  --from-file=appsettings.Production.json

# Rolling restart
kubectl rollout restart deployment/file-job-handler -n sftp-dev
```

## Access Control

### Azure Role Assignments

| Resource | Role | Principal |
|----------|------|-----------|
| Key Vault | Key Vault Secrets User | FTS Managed Identity |
| Storage Account | Storage Blob Data Contributor | FTS Managed Identity |
| Event Hub | Azure Event Hubs Data Receiver | FileEventHandler |
| Queue | Storage Queue Data Contributor | FileJobHandler |

### Kubernetes RBAC

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: fts-role
  namespace: sftp-dev
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list"]
```

### File Share Permissions

```
/mnt/jobconfig/
├── {source}/              # 755 (read for all)
│   └── {accountId}/       # 750 (no others)
│       ├── jobsettings.json   # 640
│       ├── jobsecrets.json    # 600 (owner only)
│       └── private.key        # 600 (owner only)
```

## Security Checklist

### Job Configuration

- [ ] Credentials use `{@JobSecrets.xxx}` placeholders
- [ ] No plaintext secrets in jobsettings.json
- [ ] Key files have restricted permissions
- [ ] PGP keys have strong passphrases

### Infrastructure

- [ ] TLS enabled for all Azure connections
- [ ] SFTP/FTPS for vendor connections
- [ ] Managed identities where possible
- [ ] SAS tokens with minimal scope

### Operations

- [ ] Credential rotation documented
- [ ] Key backup procedures in place
- [ ] Access audit logging enabled
- [ ] Security alerts configured

## Compliance Considerations

### Data Classification

| Data Type | Classification | Handling |
|-----------|----------------|----------|
| Credentials | Confidential | Key Vault / encrypted |
| File content | Varies | Encrypt in transit |
| Job configs | Internal | Access controlled |
| Logs | Internal | No PII |

### Audit Requirements

- Key Vault access logs
- Azure Activity logs
- Datadog trace data
- jobstatus table (retain per policy)

### Encryption Standards

- PGP: AES-256 symmetric key
- RSA: 2048-bit minimum (4096 recommended)
- TLS: 1.2 minimum
- SSH: RSA 4096 or Ed25519

## Related Documentation

- [Credentials Model](../data-models/credentials-model.md)
- [Jobsettings Storage](../job-configuration/jobsettings-storage.md)
- [Configuration Management](./configuration-management.md)
