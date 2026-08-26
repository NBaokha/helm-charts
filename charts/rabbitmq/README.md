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
└── apps/rabbitmq/ (this chart)
    ├── Chart.yaml
    ├── definitions.yaml (users, vhosts, exchanges, queues, bindings)
    └── templates/
        ├── rabbitmq-cluster.yaml (RabbitmqCluster + PodDisruptionBudget)
        ├── config-map.yaml (definitions.yaml rendered to JSON)
        ├── monitoring-annotations.yaml (Prometheus scrape Service)
        └── reload-config.yaml (PostSync definitions reload)
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

### 3. Credentials
- Static, defined in `definitions.yaml` (user `guest`, password `guest`)
- Loaded by the broker at startup via `management.load_definitions`
- Intended for local and test clusters only — these are not secrets
- No external secret manager is involved

### 4. Monitoring
- Prometheus metrics via annotation-based service discovery
- Compatible with standard Prometheus chart (no Prometheus Operator required)
- Metrics exposed on port 15692 via dedicated Service
- Grafana dashboards available

### 5. Ingress
- Management UI accessible via HTTPS
- Stage: https://rabbitmq.stage.saoad.dk
- Automatic SSL certificates via cert-manager
- Note: no Ingress is rendered by this chart; the Service is ClusterIP only.

## Environment

The chart carries a single configuration, targeting stage:

- **Location**: `charts/rabbitmq/`
- **Replicas**: 1 (single node)
- **Resources**: 1-2 CPU, 2-4Gi RAM
- **Storage**: 5Gi on storage class `testing`
- **Purpose**: Development and testing

The separate `production/` manifests (3 replicas, HA, 20Gi, plus an externally
managed secret and Ingress) were removed when this became a single chart.
Recover them from git history if needed:

```bash
git log --diff-filter=D -- 'charts/rabbitmq/prod/*'
```

## Deployment

### Prerequisites

1. **Install Cluster Operator**
   The operator is installed automatically via ArgoCD from `apps/rabbitmq-operator/`.

2. **Credentials**
   None to provision. The broker's users come from `definitions.yaml` and are
   static. Edit that file to change them.

### Deploy

```bash
helm template rabbitmq charts/rabbitmq   # render locally
helm install  rabbitmq charts/rabbitmq -n rabbitmq --create-namespace
```

Under ArgoCD this happens automatically:
1. Install the operator in `rabbitmq-system` namespace (from upstream GitHub repo)
2. Create `rabbitmq` namespace
3. Deploy the RabbitmqCluster CRD
4. Operator creates StatefulSet, Services, ConfigMaps
5. PostSync job reloads the definitions from `definitions.yaml`

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

Static, from `definitions.yaml`: user `guest`, password `guest`.

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
    storage: 20Gi
  
  rabbitmq:
    additionalConfig: |
      # Custom rabbitmq.conf configuration
    additionalPlugins:
      - rabbitmq_management
      - rabbitmq_prometheus
```

### Plugin Management

To add/remove plugins, edit `additionalPlugins` in
`templates/rabbitmq-cluster.yaml` and re-apply:
```bash
helm upgrade rabbitmq charts/rabbitmq -n rabbitmq
```

The operator will enable/disable plugins **without restarting** pods.

### Scaling

Edit `spec.replicas` in `templates/rabbitmq-cluster.yaml`:
```yaml
spec:
  replicas: 5  # Scale to 5 nodes
```

Apply the change:
```bash
helm upgrade rabbitmq charts/rabbitmq -n rabbitmq
```

The operator handles rolling upgrade automatically.

Scaling past 1 also requires revisiting the PodDisruptionBudget and the
`required` pod anti-affinity rule, both in the same file — the anti-affinity
rule leaves extra pods Pending on a single-node cluster.

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

### Definitions not applied
```bash
# The definitions ConfigMap the broker loads at startup
kubectl get configmap -n rabbitmq rabbitmq-definitions -o yaml

# The PostSync job that re-applies them to a running broker
kubectl logs -n rabbitmq job/rabbitmq-reload-definitions
```

**Common issues:**
- `management.load_definitions` only runs at startup; use the reload job for a
  running broker, or restart the pod
- Malformed `definitions.yaml` leaves the broker up but the topology missing

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
