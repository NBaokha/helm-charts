# RabbitMQ with Cluster Operator

This repository contains RabbitMQ deployment using the official **RabbitMQ Cluster Operator**.

## Why Cluster Operator?

We chose the RabbitMQ Cluster Operator over Bitnami Helm charts for the following reasons:

### Advantages

1. **No Licensing Concerns**
   - Official RabbitMQ image (MPL 2.0 license)
   - No dependency on Bitnami/Broadcom licensing
   - Long-term sustainability

2. **Better Kubernetes Integration**
   - CRD-based declarative management
   - Native Kubernetes resource management
   - Operator handles all StatefulSet operations

3. **Automatic Operations**
   - Rolling upgrades managed by operator
   - Automatic failure recovery
   - Built-in clustering support

4. **Official Support**
   - Maintained by RabbitMQ team at VMware/Broadcom
   - Better documentation and community support
   - Regular updates and security patches

5. **Flexibility**
   - Easy configuration via `additionalConfig`
   - Plugin management without pod restarts
   - No Helm chart lock-in

## Architecture

```
ArgoCD
├── apps/rabbitmq-operator/ (Operator installation)
└── apps/rabbitmq/ (RabbitmqCluster CRDs)
    ├── stage/
    │   ├── rabbitmq-cluster.yaml (1 replica)
    │   ├── infisical-secret.yaml
    │   ├── ingress.yaml
    │   └── monitoring-annotations.yaml
    └── production/
        ├── rabbitmq-cluster.yaml (3 replicas, HA)
        ├── infisical-secret.yaml
        ├── ingress.yaml
        └── monitoring-annotations.yaml
```

## Components

### 1. RabbitMQ Cluster Operator
- Installed in `rabbitmq-system` namespace
- Watches `RabbitmqCluster` CRDs across all namespaces
- Manages StatefulSets, Services, ConfigMaps, Secrets

### 2. RabbitmqCluster CRD
- Custom resource defining RabbitMQ cluster
- Declarative configuration
- Automatically creates all required Kubernetes resources

### 3. Infisical Integration
- Admin credentials stored in Infisical
- Synced to Kubernetes via InfisicalSecret CRD
- Project: `rabbitmq`, Environments: `staging` (stage cluster) and `production`
- Secret name in K8s: `rabbitmq-admin-credentials`
- Keys: `RABBITMQ_ADMIN_USERNAME`, `RABBITMQ_ADMIN_PASSWORD`

### 4. Monitoring
- Prometheus metrics via annotation-based service discovery
- Compatible with standard Prometheus chart (no Prometheus Operator required)
- Metrics exposed on port 15692 via dedicated Service
- Grafana dashboards available

### 5. Ingress
- Management UI accessible via HTTPS
- Stage: https://rabbitmq.stage.saoad.dk
- Production: https://rabbitmq.prod.saoad.dk
- Automatic SSL certificates via cert-manager

## Environments

### Stage
- **Location**: `charts/rabbitmq/stage/`
- **Replicas**: 1 (single node)
- **Resources**: 250m-500m CPU, 512Mi-1Gi RAM
- **Storage**: 5Gi
- **Purpose**: Development and testing

### Production
- **Location**: `charts/rabbitmq/production/`
- **Replicas**: 3 (HA cluster)
- **Resources**: 1-2 CPU, 2-4Gi RAM per node
- **Storage**: 20Gi per node
- **HA**: Pod anti-affinity, PodDisruptionBudget (min 2 available)
- **Purpose**: Production workloads

## Deployment

### Prerequisites

1. **Install Cluster Operator**
   The operator is installed automatically via ArgoCD from `apps/rabbitmq-operator/`.

2. **Create Secrets in Infisical**
   
   For **stage** environment:
   - Project: `rabbitmq`
   - Path: `/`
   - Environment: `staging` (note: staging, not stage)
   - Secrets:
     ```
     RABBITMQ_ADMIN_USERNAME = admin
     RABBITMQ_ADMIN_PASSWORD = <generate with: openssl rand -base64 32>
     ```
   
   For **production** environment:
   - Project: `rabbitmq`
   - Path: `/`
   - Environment: `production`
   - Secrets:
     ```
     RABBITMQ_ADMIN_USERNAME = admin
     RABBITMQ_ADMIN_PASSWORD = <generate with: openssl rand -base64 32>
     ```

3. **Infisical API**
   - Internal Infisical: `https://infisical.prod.saoad.dk`
   - NOT `https://eu.infisical.com`

### Deploy Stage

ArgoCD will automatically:
1. Install the operator in `rabbitmq-system` namespace (from upstream GitHub repo)
2. Create `rabbitmq` namespace
3. Sync secrets from Infisical to `rabbitmq-admin-credentials`
4. Deploy RabbitmqCluster CRD
5. Operator creates StatefulSet, Services, ConfigMaps
6. Create Ingress for Management UI access

### Deploy Production

Update ArgoCD ApplicationSet to include production cluster by editing `apps/rabbitmq/applicationset.yaml` selector.

Ensure DNS records and HAProxies are configured

## Verification

### Check Operator
```bash
kubectl get pods -n rabbitmq-system
kubectl logs -n rabbitmq-system -l app.kubernetes.io/name=rabbitmq-cluster-operator -f
```

### Check RabbitMQ Cluster
```bash
kubectl get rabbitmqcluster -n rabbitmq
kubectl get pods -n rabbitmq
kubectl logs -n rabbitmq -l app.kubernetes.io/name=rabbitmq -f
```

### Check Cluster Status
```bash
kubectl exec -n rabbitmq rabbitmq-server-0 -- rabbitmqctl cluster_status
```

### Access Management UI

**Via Ingress (production method):**
- Stage: https://rabbitmq.stage.saoad.dk
- Production: https://rabbitmq.prod.saoad.dk

**Via Port Forward (for testing/debugging):**
```bash
kubectl port-forward -n rabbitmq svc/rabbitmq 15672:15672
```
Open: http://localhost:15672

**Get credentials:**

Credentials are located in Infisical project "rabbitmq".

## Configuration

### RabbitmqCluster CRD Options

```yaml
spec:
  replicas: 3  # Number of nodes
  image: rabbitmq:4.1.3-management
  
  resources:
    requests:
      cpu: 1000m
      memory: 2Gi
    limits:
      cpu: 2000m
      memory: 4Gi
  
  persistence:
    storageClassName: longhorn
    storage: 20Gi
  
  rabbitmq:
    additionalConfig: |
      # Custom rabbitmq.conf configuration
    additionalPlugins:
      - rabbitmq_management
      - rabbitmq_prometheus
```

### Plugin Management

To add/remove plugins, edit `additionalPlugins` and apply:
```bash
kubectl apply -f charts/rabbitmq/stage/rabbitmq-cluster.yaml
```

The operator will enable/disable plugins **without restarting** pods.

### Scaling

Edit `spec.replicas` in the CRD:
```yaml
spec:
  replicas: 5  # Scale to 5 nodes
```

Apply the change:
```bash
kubectl apply -f charts/rabbitmq/production/rabbitmq-cluster.yaml
```

The operator handles rolling upgrade automatically.

## Monitoring

### Prometheus Setup
RabbitMQ metrics are exposed using annotation-based service discovery, compatible with the standard Prometheus chart:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: rabbitmq-metrics
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "15692"
    prometheus.io/path: "/metrics"
```

Prometheus will automatically discover and scrape this service using kubernetes_sd_configs.

### Manual Metrics Check
```bash
kubectl port-forward -n rabbitmq svc/rabbitmq-metrics 15692:15692
curl http://localhost:15692/metrics
```

### Verify Prometheus Target
```bash
kubectl port-forward -n prometheus svc/prometheus-server 9090:80
# Visit http://localhost:9090/targets and look for rabbitmq-metrics
```

### Grafana Dashboards
Import official RabbitMQ dashboards:
- RabbitMQ Overview: ID 10991
- RabbitMQ Cluster: ID 11340

## Troubleshooting

### Operator not creating resources
```bash
kubectl logs -n rabbitmq-system -l app.kubernetes.io/name=rabbitmq-cluster-operator
kubectl describe rabbitmqcluster -n rabbitmq
```

### Pod crash loop
```bash
kubectl logs -n rabbitmq rabbitmq-server-0
kubectl describe pod -n rabbitmq rabbitmq-server-0
```

### Network issues
```bash
kubectl exec -n rabbitmq rabbitmq-server-0 -- rabbitmq-diagnostics ping
kubectl exec -n rabbitmq rabbitmq-server-0 -- rabbitmq-diagnostics check_port_connectivity
```

### Secret not syncing
```bash
kubectl get infisicalsecret -n rabbitmq
kubectl describe infisicalsecret -n rabbitmq rabbitmq-admin-credentials

# Check Infisical operator logs
kubectl logs -n infisical-operator-system -l app.kubernetes.io/name=infisical-operator
```

**Common issues:**
- Environment name: Use `staging` not `stage` in Infisical
- Infisical URL: Use `https://infisical.prod.saoad.dk` not `eu.infisical.com`
- Project doesn't exist or machine identity lacks access

## Useful Commands

### Cluster Status
```bash
kubectl exec -n rabbitmq rabbitmq-server-0 -- rabbitmqctl cluster_status
```

### List Queues
```bash
kubectl exec -n rabbitmq rabbitmq-server-0 -- rabbitmqctl list_queues name messages
```

### List Users
```bash
kubectl exec -n rabbitmq rabbitmq-server-0 -- rabbitmqctl list_users
```

### Enable Feature Flags
```bash
kubectl exec -n rabbitmq rabbitmq-server-0 -- rabbitmqctl enable_feature_flag all
```

### Reset Node
```bash
kubectl exec -n rabbitmq rabbitmq-server-0 -- rabbitmqctl stop_app
kubectl exec -n rabbitmq rabbitmq-server-0 -- rabbitmqctl reset
kubectl exec -n rabbitmq rabbitmq-server-0 -- rabbitmqctl start_app
```

## References

- [RabbitMQ Cluster Operator Documentation](https://www.rabbitmq.com/kubernetes/operator/operator-overview)
- [RabbitMQ Cluster Operator GitHub](https://github.com/rabbitmq/cluster-operator)
- [RabbitMQ Configuration](https://www.rabbitmq.com/configure.html)
- [RabbitMQ Monitoring](https://www.rabbitmq.com/monitoring.html)

## Support

For issues related to:
- **Operator**: https://github.com/rabbitmq/cluster-operator/issues
- **RabbitMQ**: https://github.com/rabbitmq/rabbitmq-server/discussions
- **DAO Infrastructure**: Contact DevOps team
