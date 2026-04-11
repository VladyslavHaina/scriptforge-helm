# ScriptForge Helm Chart

Helm chart for deploying [ScriptForge](https://github.com/VladyslavHaina/ScriptForge) on Kubernetes.

## Architecture

```
+------------------------------------------------------------------+
|  Kubernetes Cluster                                              |
|                                                                  |
|  +-----------------------+    +-----------------------+          |
|  | API Deployment (2+)   |    | Controller Deployment |          |
|  | - REST API server     |    | - Job worker          |          |
|  | - Frontend SPA        |    | - Scheduler           |          |
|  | - Port 8080           |    | - Pod executor        |          |
|  +-----------+-----------+    +-----------+-----------+          |
|              |                            |                      |
|              v                            v                      |
|  +-----------+-----------+    +-----------+-----------+          |
|  | API Service           |    | PgBouncer (optional)  |          |
|  | ClusterIP :8080       |    | poolSize: 20          |          |
|  +-----------+-----------+    | maxClientConn: 100    |          |
|              |                +-----------+-----------+          |
|              v                            |                      |
|  +-----------+-----------+                v                      |
|  | Ingress (optional)    |    +-----------+-----------+          |
|  | scriptforge.local     |    | PostgreSQL StatefulSet |          |
|  +-----------------------+    | 10Gi PVC              |          |
|                               +-----------------------+          |
|                                                                  |
|  +--------------------+  +-------------------+                   |
|  | ServiceMonitor     |  | KEDA ScaledObject |                   |
|  | (Prometheus)       |  | (Autoscaling)     |                   |
|  +--------------------+  +-------------------+                   |
|                                                                  |
|  +------------------------------------------------------------+ |
|  | scriptforge-exec Namespace                                  | |
|  |  +-------+  +-------+  +-------+                           | |
|  |  | Pod   |  | Pod   |  | Pod   |  <- execution pods        | |
|  |  |alpine |  |python |  |tf     |     (ephemeral)           | |
|  |  +-------+  +-------+  +-------+                           | |
|  +------------------------------------------------------------+ |
+------------------------------------------------------------------+
```

## Quick Start

### Prerequisites
- Kubernetes 1.28+
- Helm 3.12+
- (Optional) KEDA for autoscaling
- (Optional) Prometheus Operator for monitoring

### Install

```bash
# Clone the chart
git clone https://github.com/VladyslavHaina/scriptforge-helm.git

# Install with generated encryption key
helm install scriptforge ./scriptforge-helm \
  --set encryption.key="$(openssl rand -hex 16)"

# With custom image tag
helm install scriptforge ./scriptforge-helm \
  --set encryption.key="$(openssl rand -hex 16)" \
  --set image.tag="v2.1.0"
```

### Verify

```bash
# Check pods are running
kubectl get pods -l app.kubernetes.io/name=scriptforge

# Port-forward to access the UI
kubectl port-forward svc/scriptforge-api 8080:8080

# Open http://localhost:8080
```

### Uninstall

```bash
helm uninstall scriptforge
```

## Configuration

### values.yaml

| Parameter | Default | Description |
|-----------|---------|-------------|
| `image.repository` | `vladyslavhaina/scriptforge` | Docker image |
| `image.tag` | `1.0.0` | Image tag |
| `image.pullPolicy` | `IfNotPresent` | Pull policy |
| `encryption.key` | `""` | **Required.** AES-256 encryption key (32 hex chars) |

### API Server

| Parameter | Default | Description |
|-----------|---------|-------------|
| `api.replicas` | `2` | Number of API replicas |
| `api.resources.requests.cpu` | `250m` | CPU request |
| `api.resources.requests.memory` | `256Mi` | Memory request |
| `api.resources.limits.cpu` | `1` | CPU limit |
| `api.resources.limits.memory` | `512Mi` | Memory limit |

### Controller (Worker + Scheduler)

| Parameter | Default | Description |
|-----------|---------|-------------|
| `controller.replicas` | `1` | Number of controller replicas |
| `controller.resources.requests.cpu` | `250m` | CPU request |
| `controller.resources.requests.memory` | `256Mi` | Memory request |
| `controller.resources.limits.cpu` | `1` | CPU limit |
| `controller.resources.limits.memory` | `512Mi` | Memory limit |

### PostgreSQL

| Parameter | Default | Description |
|-----------|---------|-------------|
| `postgres.enabled` | `true` | Deploy PostgreSQL StatefulSet |
| `postgres.storage` | `10Gi` | PVC size |
| `postgres.externalUrl` | `""` | External DB URL (when `enabled: false`) |
| `postgres.resources.requests.cpu` | `250m` | CPU request |
| `postgres.resources.limits.memory` | `1Gi` | Memory limit |

### PgBouncer

| Parameter | Default | Description |
|-----------|---------|-------------|
| `pgbouncer.enabled` | `true` | Deploy PgBouncer |
| `pgbouncer.poolSize` | `20` | Connection pool size |
| `pgbouncer.maxClientConn` | `100` | Max client connections |

### Ingress

| Parameter | Default | Description |
|-----------|---------|-------------|
| `ingress.enabled` | `true` | Create Ingress resource |
| `ingress.className` | `nginx` | Ingress class |
| `ingress.host` | `scriptforge.local` | Hostname |
| `ingress.annotations` | `{}` | Extra annotations |

### Runtime Images

| Parameter | Default | Description |
|-----------|---------|-------------|
| `runtimeImages.bash` | `alpine:3.21` | Bash runtime |
| `runtimeImages.python` | `python:3.13-alpine` | Python runtime |
| `runtimeImages.terraform` | `hashicorp/terraform:1.10` | Terraform runtime |
| `runtimeImages.powershell` | `mcr.microsoft.com/powershell:7.5-alpine-3.20` | PowerShell runtime |
| `runtimeImages.ansible` | `alpine:3.21` | Ansible runtime |

### Monitoring

| Parameter | Default | Description |
|-----------|---------|-------------|
| `monitoring.enabled` | `false` | Enable Prometheus metrics |
| `monitoring.serviceMonitor` | `false` | Create ServiceMonitor CR |

### Autoscaling (KEDA)

| Parameter | Default | Description |
|-----------|---------|-------------|
| `autoscaling.enabled` | `false` | Enable KEDA autoscaling |
| `autoscaling.minReplicas` | `1` | Minimum controller replicas |
| `autoscaling.maxReplicas` | `20` | Maximum controller replicas |
| `autoscaling.cooldownPeriod` | `300` | Cooldown after scale-down (seconds) |
| `autoscaling.queueThreshold` | `10` | Jobs in queue before scaling up |

### Execution

| Parameter | Default | Description |
|-----------|---------|-------------|
| `execution.podNamespace` | `scriptforge-exec` | Namespace for execution pods |

## Production Deployment

Use `values-production.yaml` for production:

```bash
helm install scriptforge ./scriptforge-helm \
  -f values-production.yaml \
  --set encryption.key="$(openssl rand -hex 16)"
```

### Production differences from default

| Setting | Default | Production |
|---------|---------|------------|
| API replicas | 2 | 3 |
| Controller replicas | 1 | 2 |
| API CPU limit | 1 | 2 |
| PostgreSQL storage | 10Gi | 50Gi |
| PostgreSQL CPU limit | 1 | 4 |
| PostgreSQL memory limit | 1Gi | 4Gi |
| PgBouncer pool | 20 | 40 |
| PgBouncer max connections | 100 | 200 |
| Autoscaling | disabled | enabled (2-50 replicas) |
| Monitoring | disabled | enabled + ServiceMonitor |

### External PostgreSQL

To use an external database instead of the built-in StatefulSet:

```yaml
postgres:
  enabled: false
  externalUrl: "postgres://user:password@your-rds.amazonaws.com:5432/scriptforge?sslmode=require"
```

### TLS / HTTPS

```yaml
ingress:
  enabled: true
  className: nginx
  host: scriptforge.example.com
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
```

## Templates

| Template | Resource | Description |
|----------|----------|-------------|
| `api-deployment.yaml` | Deployment | API server pods |
| `api-service.yaml` | Service | API ClusterIP service |
| `api-ingress.yaml` | Ingress | External access |
| `controller-deployment.yaml` | Deployment | Controller pods |
| `postgres-statefulset.yaml` | StatefulSet + Service | Database |
| `pgbouncer-deployment.yaml` | Deployment + Service | Connection pooler |
| `configmap.yaml` | ConfigMap | Non-sensitive config |
| `secret.yaml` | Secret | Encryption key, DB URL |
| `namespace.yaml` | Namespace | Execution pod namespace |
| `serviceaccount.yaml` | ServiceAccount | Pod identity |
| `role.yaml` | Role | K8s API permissions |
| `rolebinding.yaml` | RoleBinding | Bind role to SA |
| `servicemonitor.yaml` | ServiceMonitor | Prometheus scraping |
| `keda-scaledobject.yaml` | ScaledObject | KEDA autoscaling |
| `keda-triggerauth.yaml` | TriggerAuthentication | KEDA DB auth |

## RBAC

The chart creates a `Role` in the execution namespace with permissions to:
- Create, get, list, watch, delete **pods**
- Get, list **pods/log**
- Create, get, list, watch, delete **configmaps**

This allows the controller to create execution pods and stream their logs.

## License

MIT
