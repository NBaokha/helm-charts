# RabbitMQ Cluster Operator - Deployment Guide

## Completed Setup

### Structure Created
```
apps/
├── rabbitmq-operator/
│   └── applicationset.yaml (Operator installation from GitHub)
└── rabbitmq/
    └── applicationset.yaml (RabbitmqCluster deployment)

charts/rabbitmq/
├── README.md (Main documentation)
├── DEPLOYMENT.md (This file)
├── DEVELOPER_GUIDE.md (For application developers)
├── stage/
│   ├── rabbitmq-cluster.yaml (1 replica, 5Gi, 512Mi-1Gi)
│   ├── infisical-secret.yaml
│   ├── ingress.yaml
│   └── monitoring-annotations.yaml
└── production/
    ├── rabbitmq-cluster.yaml (3 replicas HA, 20Gi, 2-4Gi, PDB)
    ├── infisical-secret.yaml
    ├── ingress.yaml
    └── monitoring-annotations.yaml 
```

## Deployment Steps

### 1. Create Secrets in Infisical

**IMPORTANT:** Environment name in Infisical is `staging` (not `stage`)

**Stage Environment**:
- Infisical URL: `https://infisical.prod.saoad.dk`
- Project: `rabbitmq`
- Path: `/`
- Environment: `staging` ⚠️
- Secrets:
  ```
  RABBITMQ_ADMIN_USERNAME = admin
  RABBITMQ_ADMIN_PASSWORD = <generate strong password>
  ```

**Production Environment**:
- Infisical URL: `https://infisical.prod.saoad.dk`
- Project: `rabbitmq`
- Path: `/`
- Environment: `production`
- Secrets:
  ```
  RABBITMQ_ADMIN_USERNAME = admin
  RABBITMQ_ADMIN_PASSWORD = <generate strong password>
  ```

Generate password:
```bash
openssl rand -base64 32
```
or use Infisical's password generator.
### 2. Monitor Deployment

**Operator Installation:**
```bash
# Watch operator installation
kubectl get pods -n rabbitmq-system -w

# Check operator logs
kubectl logs -n rabbitmq-system -l app.kubernetes.io/name=rabbitmq-cluster-operator -f
```

**RabbitMQ Cluster:**
```bash
# Watch RabbitMQ cluster creation
kubectl get rabbitmqcluster -n rabbitmq -w
kubectl get pods -n rabbitmq -w

# Check cluster status
kubectl exec -n rabbitmq rabbitmq-server-0 -- rabbitmqctl cluster_status
```

**ArgoCD:**
```bash
# Watch ArgoCD sync
argocd app list | grep rabbitmq
argocd app get rabbitmq-operator
argocd app get rabbitmq
```

### 3. Verify Installation

**Check Operator:**
```bash
kubectl get deployment -n rabbitmq-system
kubectl get pods -n rabbitmq-system
```

**Check RabbitMQ:**
```bash
kubectl get rabbitmqcluster -n rabbitmq
kubectl get statefulset -n rabbitmq
kubectl get pods -n rabbitmq
kubectl get svc -n rabbitmq
```

**Check Secrets:**
```bash
kubectl get secret -n rabbitmq rabbitmq-admin-credentials
kubectl get infisicalsecret -n rabbitmq
kubectl describe infisicalsecret rabbitmq-admin-credentials -n rabbitmq
```

**Test Connection:**
```bash
# Via Ingress (production method)
# Stage: https://rabbitmq.stage.saoad.dk
# Production: https://rabbitmq.prod.saoad.dk

# Or port-forward Management UI (for testing)
kubectl port-forward -n rabbitmq svc/rabbitmq 15672:15672

# Open browser: http://localhost:15672
# Get credentials:
kubectl get secret rabbitmq-admin-credentials -n rabbitmq -o jsonpath='{.data.RABBITMQ_ADMIN_USERNAME}' | base64 -d
kubectl get secret rabbitmq-admin-credentials -n rabbitmq -o jsonpath='{.data.RABBITMQ_ADMIN_PASSWORD}' | base64 -d
```

**Check Metrics:**
```bash
# Port-forward Prometheus metrics
kubectl port-forward -n rabbitmq svc/rabbitmq 15692:15692

# Fetch metrics
curl http://localhost:15692/metrics
```

## Configuration

### Stage Environment
- **Namespace**: `rabbitmq`
- **Replicas**: 1
- **CPU**: 250m-500m
- **Memory**: 512Mi-1Gi
- **Storage**: 5Gi (Longhorn)
- **Plugins**: Essential plugins enabled by operator (management, prometheus, peer_discovery_k8s) + shovel, shovel_management
- **Ingress**: https://rabbitmq.stage.saoad.dk

### Production Environment (when deployed)
- **Namespace**: `rabbitmq`
- **Replicas**: 3 (HA)
- **CPU**: 1-2 cores
- **Memory**: 2-4Gi
- **Storage**: 20Gi per node (Longhorn)
- **Plugins**: Essential plugins + shovel, shovel_management, federation, federation_management
- **HA**: Pod anti-affinity (required), PDB (min 2 available)
- **Security**: NetworkPolicy
- **Ingress**: https://rabbitmq.prod.saoad.dk

## Key Features

### Operator Managed
- StatefulSet creation and management
- Service creation (ClusterIP for AMQP, Management, Prometheus)
- ConfigMap for RabbitMQ configuration
- Secret for Erlang cookie (auto-generated)
- Rolling upgrades
- Automatic failure recovery

### Monitoring
- Prometheus metrics via annotation-based service discovery
- Compatible with standard Prometheus chart (no Prometheus Operator/ServiceMonitor needed)
- Metrics on port 15692 via `rabbitmq-metrics` service
- Management UI on port 15672 via `rabbitmq` service
- AMQP on port 5672 via `rabbitmq` service

### Security
- Infisical secret management
- NetworkPolicy (production)
- Non-root containers (official RabbitMQ image)
- Pod security contexts

### High Availability (Production)
- 3 replicas
- Pod anti-affinity (spread across nodes)
- PodDisruptionBudget (min 2 available)
- Automatic partition healing

## Important Notes

1. **Operator Namespace**: The operator is installed in `rabbitmq-system` namespace from upstream GitHub repo (rabbitmq/cluster-operator)
2. **Operator Version**: Pinned to `v2.16.1` (tagged release, not main branch)
3. **Cluster Namespace**: RabbitMQ clusters are deployed in `rabbitmq` namespace
4. **Secret Names**: `rabbitmq-admin-credentials` for admin credentials (synced from Infisical)
5. **Infisical Configuration**: 
   - URL: `https://infisical.prod.saoad.dk` (internal, not eu.infisical.com)
   - Stage: Project `rabbitmq`, Path `/`, Environment `staging` ⚠️ (not "stage")
   - Production: Project `rabbitmq`, Path `/`, Environment `production`
   - Keys: `RABBITMQ_ADMIN_USERNAME`, `RABBITMQ_ADMIN_PASSWORD`
6. **Image**: Official `rabbitmq:4.1.3-management` (not Bitnami)
7. **License**: MPL 2.0 (Mozilla Public License)
8. **Ingress**: Management UI accessible via HTTPS with automatic SSL certificates
9. **Application Access**: Internal URL `rabbitmq.rabbitmq.svc.cluster.local:5672` (AMQP)

## Next Steps

### After Stage Deployment
1. Test AMQP connection from application
2. Verify monitoring in Prometheus/Grafana
3. Test secret rotation from Infisical
4. Document any custom configuration needed

### Production Deployment
1. Update ApplicationSet to include production cluster
2. Create secrets in Infisical (`/rabbitmq/production`)
3. Deploy via ArgoCD
4. Verify 3-node cluster formation
5. Test failover scenarios
6. Configure application connection strings

## Documentation

- **Main README**: `charts/rabbitmq/README.md` (architecture and overview)
- **Deployment Guide**: `charts/rabbitmq/DEPLOYMENT.md` (this file - for platform team)
- **Developer Guide**: `charts/rabbitmq/DEVELOPER_GUIDE.md` (for application developers)
- **Operator Docs**: https://www.rabbitmq.com/kubernetes/operator/operator-overview
- **Operator GitHub**: https://github.com/rabbitmq/cluster-operator

## Troubleshooting

### Operator Issues
```bash
kubectl logs -n rabbitmq-system -l app.kubernetes.io/name=rabbitmq-cluster-operator
kubectl describe deployment -n rabbitmq-system rabbitmq-cluster-operator
```

### Cluster Issues
```bash
kubectl describe rabbitmqcluster -n rabbitmq rabbitmq
kubectl logs -n rabbitmq rabbitmq-server-0
kubectl exec -n rabbitmq rabbitmq-server-0 -- rabbitmq-diagnostics status
```

### Secret Issues
```bash
kubectl get infisicalsecret -n rabbitmq rabbitmq-admin-credentials -o yaml
kubectl describe infisicalsecret -n rabbitmq rabbitmq-admin-credentials
kubectl logs -n infisical-operator-system -l app.kubernetes.io/name=infisical-operator
```

**Common Infisical issues:**
- Wrong environment: Use `staging` not `stage`
- Wrong URL: Use `https://infisical.prod.saoad.dk` not `eu.infisical.com`
- Project doesn't exist or machine identity lacks access
- Invalid credentials in `infisical-secrets` secret in `argocd` namespace

## Support

- Operator Issues: https://github.com/rabbitmq/cluster-operator/issues
- RabbitMQ Issues: https://github.com/rabbitmq/rabbitmq-server/discussions
- DAO DevOps: Internal team

