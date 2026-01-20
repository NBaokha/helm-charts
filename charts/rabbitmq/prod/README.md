# RabbitMQ Cluster Operator - Production Environment

This directory contains the RabbitmqCluster CRD for the production environment with High Availability.

## Architecture

- **Operator**: RabbitMQ Cluster Operator v2.x (installed via ArgoCD)
- **CRD**: `RabbitmqCluster` custom resource
- **Image**: `rabbitmq:4.1.3-management` (official RabbitMQ image)
- **License**: MPL 2.0 (Mozilla Public License)

## Production Configuration

- **Replicas**: 3 (HA cluster with quorum)
- **Resources**:
  - CPU: 1-2 cores per node
  - Memory: 2-4Gi per node
- **Storage**: 20Gi per node (Longhorn)
- **Service**: ClusterIP
- **PodDisruptionBudget**: Minimum 2 pods available

## High Availability Features

### Pod Anti-Affinity
Enforces pod distribution across different nodes using `requiredDuringSchedulingIgnoredDuringExecution`.
This ensures the cluster survives single node failures.

### PodDisruptionBudget
Ensures at least 2 out of 3 pods remain available during:
- Kubernetes upgrades
- Node maintenance
- Voluntary disruptions

### Cluster Configuration
- **Partition handling**: `autoheal` (automatically heals network partitions)
- **Queue master locator**: `min-masters` (distributes queue masters evenly)
- **Cluster keepalive**: 10 seconds

## Secrets

The cluster requires a secret `rabbitmq-default-user` with:
- `username`: RabbitMQ admin username
- `password`: RabbitMQ admin password

This secret is automatically synced from Infisical:
- **Project**: `rabbitmq`
- **Path**: `/`
- **Environment**: `production`
- **Keys**: `RABBITMQ_ADMIN_USERNAME`, `RABBITMQ_ADMIN_PASSWORD`

Create secrets in Infisical:
1. Login to Infisical (https://eu.infisical.com)
2. Navigate to project `rabbitmq`
3. Select environment `production`
4. Add secrets:
   - `RABBITMQ_ADMIN_USERNAME`: `admin`
   - `RABBITMQ_ADMIN_PASSWORD`: Generate strong password (`openssl rand -base64 32`)

## Features Enabled

- `rabbitmq_management`: Management UI
- `rabbitmq_prometheus`: Prometheus metrics
- `rabbitmq_peer_discovery_k8s`: Kubernetes discovery
- `rabbitmq_shovel`: Message shoveling
- `rabbitmq_shovel_management`: Shovel management
- `rabbitmq_federation`: Federation support
- `rabbitmq_federation_management`: Federation UI

## Access

### Management UI
```bash
kubectl port-forward -n rabbitmq svc/rabbitmq 15672:15672
```
Then open: http://localhost:15672

### AMQP Connection
```bash
kubectl port-forward -n rabbitmq svc/rabbitmq 5672:5672
```
Connection string: `amqp://username:password@localhost:5672/`

## Monitoring

Prometheus metrics are exposed on port 15692:
```bash
kubectl port-forward -n rabbitmq svc/rabbitmq 15692:15692
curl http://localhost:15692/metrics
```

## Operations

### Cluster Status
```bash
kubectl exec -n rabbitmq rabbitmq-server-0 -- rabbitmqctl cluster_status
```

### Node Health
```bash
for i in 0 1 2; do
  echo "=== Node $i ==="
  kubectl exec -n rabbitmq rabbitmq-server-$i -- rabbitmqctl node_health_check
done
```

### Quorum Queue Status
```bash
kubectl exec -n rabbitmq rabbitmq-server-0 -- rabbitmqctl list_queues name type members
```

### Rolling Restart (safe)
The operator handles rolling restarts automatically. If needed:
```bash
kubectl rollout restart -n rabbitmq statefulset/rabbitmq-server
```

## Scaling

To scale the cluster:
```yaml
spec:
  replicas: 5  # Change to desired number (odd numbers recommended)
```

**Important**: Always use odd numbers (3, 5, 7) for quorum-based decisions.

## Disaster Recovery

### Backup
RabbitMQ data is stored in PersistentVolumes. Use Longhorn snapshots for backup:
```bash
# Create snapshot via Longhorn UI or CLI
# Snapshots are stored according to Longhorn backup policy
```

### Restore
1. Scale down to 0 replicas
2. Restore PVCs from Longhorn snapshots
3. Scale back to 3 replicas

## Troubleshooting

### View logs from all pods
```bash
kubectl logs -n rabbitmq -l app.kubernetes.io/name=rabbitmq -f --max-log-requests=3
```

### Check operator logs
```bash
kubectl logs -n rabbitmq-system -l app.kubernetes.io/name=rabbitmq-cluster-operator -f
```

### Network partition
```bash
# Check for partitions
kubectl exec -n rabbitmq rabbitmq-server-0 -- rabbitmqctl cluster_status | grep -A 10 partitions

# If partition detected, operator will auto-heal due to 'autoheal' setting
```

### Pod not starting
```bash
# Describe pod
kubectl describe pod -n rabbitmq rabbitmq-server-0

# Check events
kubectl get events -n rabbitmq --sort-by='.lastTimestamp'
```

## Performance Tuning

### Memory
Current watermark is 60% (`vm_memory_high_watermark.relative = 0.6`).
With 2-4Gi memory, this provides ~1.2-2.4Gi usable memory per node.

### Disk
Free disk limit is 5GB (`disk_free_limit.absolute = 5GB`).
With 20Gi storage, alarms trigger when <5GB remains.

### Consumer Timeout
Set to 15 minutes (900000ms). Increase if you have long-running consumers.

## Migration from Bitnami

This setup uses the official RabbitMQ Cluster Operator instead of Bitnami Helm charts.

**Benefits**:
- CRD-based declarative management
- Official RabbitMQ image (no vendor lock-in)
- Better Kubernetes integration
- Automatic rolling upgrades
- Built-in Prometheus monitoring
- No licensing concerns
- Native HA support with Cluster Operator

**Differences from Bitnami**:
- No Helm chart dependencies
- Different secret structure
- Operator manages StatefulSet automatically
- Different configuration approach (additionalConfig vs values.yaml)
- Built-in quorum queue support
