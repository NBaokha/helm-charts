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
├── Chart.yaml
├── definitions.yaml (users, vhosts, exchanges, queues, bindings)
└── templates/
    ├── rabbitmq-cluster.yaml (1 replica, 5Gi, RabbitmqCluster + PDB)
    ├── config-map.yaml (definitions.yaml rendered to JSON)
    ├── monitoring-annotations.yaml (Prometheus scrape Service)
    └── reload-config.yaml (PostSync definitions reload)
```

The separate `production/` manifests were removed when this became a single
chart; recover them from git history if needed.

## Deployment Steps

### 1. Credentials

Nothing to provision. Users, vhosts and permissions are static and live in
`definitions.yaml`, which the broker loads at startup:

```yaml
users:
  - name: guest
    password: guest
    tags: [administrator]
```

These are throwaway values for a local/test cluster, not secrets. To change
them, edit `definitions.yaml` and re-sync — the PostSync job re-applies the
definitions to a running broker.
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

**Check Definitions:**
```bash
kubectl get configmap -n rabbitmq rabbitmq-definitions -o yaml
kubectl logs -n rabbitmq job/rabbitmq-reload-definitions
```

**Test Connection:**
```bash
# The chart renders no Ingress, so port-forward the Management UI
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

The chart ships a single configuration, matching what the templates actually
declare:

- **Namespace**: `rabbitmq`
- **Replicas**: 1
- **CPU**: 1-2 cores
- **Memory**: 2-4Gi
- **Storage**: 5Gi on storage class `testing`
- **Plugins**: Essential plugins enabled by operator (management, prometheus, peer_discovery_k8s) + shovel, shovel_management, federation, federation_management
- **HA**: Pod anti-affinity (required), PDB (minAvailable 1)
- **Ingress**: none rendered by this chart; the Service is ClusterIP only

The production variant (3 replicas, 20Gi, NetworkPolicy, Ingress) was removed
when this became a single chart.

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
- Non-root containers (official RabbitMQ image)
- Pod security contexts

### High Availability
The manifests declare pod anti-affinity and a PodDisruptionBudget, but with
`replicas: 1` neither provides real availability — and `minAvailable: 1` against
a single replica will block a node drain. Scale up before relying on these.

## Important Notes

1. **Operator Namespace**: The operator is installed in `rabbitmq-system` namespace from upstream GitHub repo (rabbitmq/cluster-operator)
2. **Operator Version**: Pinned to `v2.16.1` (tagged release, not main branch)
3. **Cluster Namespace**: RabbitMQ clusters are deployed in `rabbitmq` namespace
4. **Credentials**: Static, in `definitions.yaml` (`guest`/`guest`). No external
   secret manager — suitable for local/test clusters only.
5. **Image**: Official `rabbitmq:4.1.3-management` (not Bitnami)
6. **License**: MPL 2.0 (Mozilla Public License)
7. **Ingress**: None rendered by this chart; port-forward the Management UI
8. **Application Access**: Internal URL `rabbitmq.rabbitmq.svc.cluster.local:5672` (AMQP)

## Next Steps

1. Test AMQP connection from application
2. Verify monitoring in Prometheus/Grafana
3. Document any custom configuration needed

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

### Definitions Issues
```bash
kubectl get configmap -n rabbitmq rabbitmq-definitions -o yaml
kubectl logs -n rabbitmq job/rabbitmq-reload-definitions
```

**Common issues:**
- `management.load_definitions` only applies at startup; the PostSync reload job
  handles a running broker, or restart the pod
- Malformed `definitions.yaml` leaves the broker up but the topology missing
- Login failures usually mean `definitions.yaml` was edited without a reload

## Support

- Operator Issues: https://github.com/rabbitmq/cluster-operator/issues
- RabbitMQ Issues: https://github.com/rabbitmq/rabbitmq-server/discussions
- DAO DevOps: Internal team

