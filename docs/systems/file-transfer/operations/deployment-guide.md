# Deployment Guide

This guide covers deploying FTS components to Azure Kubernetes Service (AKS).

## Prerequisites

### Azure Resources

- Azure Kubernetes Service (AKS) cluster
- Azure Container Registry (ACR)
- Azure Storage Account (Queues, Tables, Blobs, Files)
- Azure Event Hub namespace
- Azure Key Vault

### Tools

- kubectl configured for target cluster
- Azure CLI authenticated
- Docker for local builds

### Credentials

- ACR push/pull credentials
- Storage SAS tokens
- Event Hub connection strings
- Key Vault access

## Deployment Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                     AKS Cluster                               │
├──────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐  ┌─────────────────┐                    │
│  │ sftp-dev        │  │ sftp-stg        │  │ sftp-prod     │ │
│  │ namespace       │  │ namespace       │  │ namespace     │ │
│  ├─────────────────┤  ├─────────────────┤                    │
│  │ Deployments:    │  │ Deployments:    │                    │
│  │ - FileEvent...  │  │ - FileEvent...  │                    │
│  │ - FileJob...    │  │ - FileJob...    │                    │
│  │ - TMS handlers  │  │ - TMS handlers  │                    │
│  └─────────────────┘  └─────────────────┘                    │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │ Persistent Volumes (Azure File Shares)                   │ │
│  │ - sftp-internal-fs-{env}                                 │ │
│  │ - sftp-external-fs-{env}                                 │ │
│  └─────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

## Environment Setup

### Create Namespace

```bash
kubectl create namespace sftp-dev
kubectl create namespace sftp-stg
kubectl create namespace sftp-prod
```

### Create Persistent Volume Claims

```yaml
# sftp-internal-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: sftp-internal-fs-dev
  namespace: sftp-dev
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: azurefile
  resources:
    requests:
      storage: 100Gi
```

```bash
kubectl apply -f sftp-internal-pvc.yaml
kubectl apply -f sftp-external-pvc.yaml
```

## Configuration

### Application Settings

Create `appsettings.Production.json`:

```json
{
  "EventHub": {
    "EnterpriseFTP": {
      "ConnectionString": "${EVENTHUB_CONNECTION_STRING}",
      "EventHubName": "${EVENTHUB_NAME}"
    }
  },
  "Queues": {
    "jobs": {
      "Url": "${QUEUES_JOBS_URL}",
      "SASToken": "${QUEUES_JOBS_SAS_TOKEN}"
    },
    "NextCheck": 1000,
    "VisibilityTimeout": 120
  },
  "Tables": {
    "jobstatus": {
      "Url": "${TABLES_JOBS_STATUS_URL}",
      "SASToken": "${TABLES_JOBS_STATUS_SAS_TOKEN}",
      "TableName": "jobstatus"
    }
  },
  "KeyVaultSecrets": {
    "Enable": true,
    "Uri": "${KEY_VAULT_URI}",
    "TanentID": "${KEY_VAULT_TENANT_ID}"
  },
  "VirusScan": {
    "ClamAVServiceIP": "${VIRUS_SCAN_IP}",
    "ClamAVServicePort": "${VIRUS_SCAN_PORT}"
  },
  "AzureFiles": {
    "sftpintroot": "/mnt/sftpintroot",
    "sftpextroot": "/mnt/sftpextroot"
  }
}
```

### Create Kubernetes Secret

```bash
kubectl create secret -n sftp-dev generic \
  sftp-dev-file-job-handler-app-settings-secret \
  --from-file=appsettings.Production.json
```

## Deployment

### Deployment Template

```yaml
# file-job-handler-template.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: file-job-handler
  namespace: ${K8S_NAMESPACE}
  labels:
    app: file-job-handler
    project: sftp
    env: ${K8S_ENV}
spec:
  replicas: ${K8S_REPLICAS}
  selector:
    matchLabels:
      app: file-job-handler
  template:
    metadata:
      labels:
        app: file-job-handler
        version: ${K8S_VERSION}
    spec:
      containers:
        - name: file-job-handler
          image: ${K8S_IMAGE_URI}:${K8S_IMAGE_TAG}
          ports:
            - containerPort: 80
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "1000m"
          volumeMounts:
            - name: appsettings
              mountPath: /app/appsettings.Production.json
              subPath: appsettings.Production.json
            - name: sftpintroot
              mountPath: /mnt/sftpintroot
            - name: sftpextroot
              mountPath: /mnt/sftpextroot
      volumes:
        - name: appsettings
          secret:
            secretName: ${K8S_SECRET_NAME}
        - name: sftpintroot
          persistentVolumeClaim:
            claimName: ${K8S_INT_PVC}
        - name: sftpextroot
          persistentVolumeClaim:
            claimName: ${K8S_EXT_PVC}
```

### Apply Deployment

```bash
# Substitute variables
envsubst < file-job-handler-template.yaml > file-job-handler.yaml

# Apply to cluster
kubectl apply -n sftp-dev -f file-job-handler.yaml
```

### Verify Deployment

```bash
# Check deployment status
kubectl get deployments -n sftp-dev

# Check pod status
kubectl get pods -n sftp-dev

# View logs
kubectl logs -n sftp-dev -l app=file-job-handler
```

## Scaling

### Manual Scaling

```bash
kubectl scale deployment file-job-handler -n sftp-dev --replicas=3
```

### Horizontal Pod Autoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: file-job-handler-hpa
  namespace: sftp-dev
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: file-job-handler
  minReplicas: 1
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

## Updating

### Rolling Update

```bash
# Update image tag
kubectl set image deployment/file-job-handler \
  file-job-handler=enterprisenonpacr.azurecr.io/enterprise/file-job-handler:new-tag \
  -n sftp-dev

# Watch rollout
kubectl rollout status deployment/file-job-handler -n sftp-dev
```

### Rollback

```bash
# Rollback to previous version
kubectl rollout undo deployment/file-job-handler -n sftp-dev

# Rollback to specific revision
kubectl rollout undo deployment/file-job-handler -n sftp-dev --to-revision=2
```

## CI/CD Integration

### Azure DevOps Pipeline

The `azure-pipelines.yml` automates deployment:

1. **Build Stage**: Build and push Docker image
2. **Dev Deploy**: Deploy to development
3. **Stg Deploy**: Deploy to staging
4. **Prod Deploy**: Deploy to production

### Pipeline Variables

| Variable | Description |
|----------|-------------|
| `SECRET_EVENTHUB_CONNECTION_STRING` | Event Hub connection |
| `SECRET_QUEUES_SAS_TOKEN` | Queue SAS token |
| `SECRET_TABLES_SAS_TOKEN` | Table SAS token |
| `SECRET_KEY_VAULT_TENANT_ID` | Key Vault tenant |

## Troubleshooting

### Pod Not Starting

```bash
# Describe pod for events
kubectl describe pod <pod-name> -n sftp-dev

# Common issues:
# - Image pull errors: Check ACR credentials
# - Volume mount errors: Verify PVC exists
# - Secret errors: Verify secret exists
```

### Container Crashes

```bash
# View logs from crashed container
kubectl logs <pod-name> -n sftp-dev --previous

# Check container exit code
kubectl get pod <pod-name> -n sftp-dev -o jsonpath='{.status.containerStatuses[0].lastState}'
```

### Storage Issues

```bash
# Verify PVC is bound
kubectl get pvc -n sftp-dev

# Check Azure File Share mount
kubectl exec -it <pod-name> -n sftp-dev -- ls -la /mnt/sftpintroot
```

## Related Documentation

- [Configuration Management](./configuration-management.md)
- [Monitoring & Logging](./monitoring-logging.md)
- [Deployment Architecture](../architecture/deployment-architecture.md)
