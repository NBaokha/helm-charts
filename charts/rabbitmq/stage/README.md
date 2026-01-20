# RabbitMQ Cluster Operator - Stage Environment

This directory contains the RabbitmqCluster CRD for the stage environment.

## Architecture

- **Operator**: RabbitMQ Cluster Operator v2.x (installed via ArgoCD)
- **CRD**: `RabbitmqCluster` custom resource
- **Image**: `rabbitmq:4.1.3-management` (official RabbitMQ image)
- **License**: MPL 2.0 (Mozilla Public License)

## Stage Configuration

- **Replicas**: 1 (single node)
- **Resources**:
  - CPU: 250m-500m
  - Memory: 512Mi-1Gi
- **Storage**: 5Gi (Longhorn)
- **Service**: ClusterIP

## Secrets

The cluster requires a secret `rabbitmq-default-user` with:
- `username`: RabbitMQ admin username
- `password`: RabbitMQ admin password

This secret is automatically synced from Infisical:
- **Project**: `rabbitmq`
- **Path**: `/`
- **Environment**: `stage`
- **Keys**: `RABBITMQ_ADMIN_USERNAME`, `RABBITMQ_ADMIN_PASSWORD`

Create secrets in Infisical:
1. Login to Infisical (https://eu.infisical.com)
2. Navigate to project `rabbitmq`
3. Select environment `stage`
4. Add secrets:
   - `RABBITMQ_ADMIN_USERNAME`: `admin`
   - `RABBITMQ_ADMIN_PASSWORD`: Generate strong password (`openssl rand -base64 32`)

## Features Enabled

- `rabbitmq_management`: Management UI
- `rabbitmq_prometheus`: Prometheus metrics
- `rabbitmq_peer_discovery_k8s`: Kubernetes discovery
- `rabbitmq_shovel`: Message shoveling
- `rabbitmq_shovel_management`: Shovel management

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

## Troubleshooting

### View logs
```bash
kubectl logs -n rabbitmq -l app.kubernetes.io/name=rabbitmq -f
```

### Cluster status
```bash
kubectl exec -n rabbitmq rabbitmq-server-0 -- rabbitmqctl cluster_status
```

### Check operator logs
```bash
kubectl logs -n rabbitmq-system -l app.kubernetes.io/name=rabbitmq-cluster-operator -f
```

## Migration from Bitnami

This setup uses the official RabbitMQ Cluster Operator instead of Bitnami Helm charts.

**Benefits**:
- CRD-based declarative management
- Official RabbitMQ image (no vendor lock-in)
- Better Kubernetes integration
- Automatic rolling upgrades
- Built-in Prometheus monitoring
- No licensing concerns

**Differences from Bitnami**:
- No Helm chart dependencies
- Different secret structure
- Operator manages StatefulSet automatically
- Different configuration approach (additionalConfig vs values.yaml)
