# EFK Logging Stack

This Helm umbrella chart deploys the EFK (Elasticsearch, Fluentd, Kibana) stack for centralized logging in the CRM Kubernetes cluster.

## Architecture

- **Elasticsearch**: Stores and indexes log data
- **Fluentd**: Collects logs from all pods and forwards to Elasticsearch
- **Kibana**: Provides web UI for log visualization and analysis

## Components

### Elasticsearch
- Single-node deployment optimized for t3.medium
- 512Mi memory, 5Gi persistent storage
- Health check accepts yellow status (normal for single-node)

### Fluentd
- DaemonSet runs on every node
- Automatically collects logs from all pods
- Parses JSON logs from CRM application
- Forwards to Elasticsearch with Logstash format

### Kibana
- LoadBalancer service for external access
- Connected to Elasticsearch with authentication
- Pre-configured for Elasticsearch 8.5.1

## Prerequisites

1. Kubernetes cluster with at least 1 node
2. kubectl configured to access the cluster
3. Helm 3.x installed
4. At least 1.5Gi memory available on the node

## Installation

### 1. Add Helm Repositories

```bash
helm repo add elastic https://helm.elastic.co
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update
```

### 2. Update Chart Dependencies

```bash
cd logging-stack
helm dependency update
```

This will download the Elasticsearch, Kibana, and Fluentd charts into the `charts/` directory.

### 3. Create Logging Namespace

```bash
kubectl create namespace logging
```

### 4. Deploy the Logging Stack

```bash
helm install logging-stack . \
  --namespace logging \
  --timeout 10m \
  --wait
```

### 5. Verify Deployment

```bash
# Check all pods are running
kubectl get pods -n logging

# Expected output:
# NAME                                      READY   STATUS    RESTARTS   AGE
# logging-stack-elasticsearch-master-0      1/1     Running   0          5m
# logging-stack-kibana-xxxxx                1/1     Running   0          5m
# logging-stack-fluentd-xxxxx               1/1     Running   0          5m

# Check services
kubectl get svc -n logging
```

### 6. Get Elasticsearch Password

```bash
kubectl get secrets --namespace=logging \
  logging-stack-elasticsearch-master-credentials \
  -ojsonpath='{.data.password}' | base64 -d
echo
```

Save this password - you'll need it to log into Kibana.

### 7. Access Kibana

Get the Kibana LoadBalancer URL:

```bash
kubectl get svc -n logging logging-stack-kibana
```

Wait for the EXTERNAL-IP to be assigned, then access Kibana at:
```
http://<EXTERNAL-IP>:5601
```

Login credentials:
- **Username**: `elastic`
- **Password**: (from step 6)

## Post-Installation Configuration

### Create Index Pattern in Kibana

1. Open Kibana in your browser
2. Navigate to **Management** → **Stack Management** → **Index Patterns**
3. Click **Create index pattern**
4. Enter index pattern: `kubernetes-*`
5. Select time field: `@timestamp`
6. Click **Create index pattern**

### View Logs

1. Navigate to **Discover** in the left menu
2. Select the `kubernetes-*` index pattern
3. You should see logs from all pods including the CRM application

### Filter CRM Application Logs

To see only CRM application logs:
1. In the Discover view, add a filter:
   - Field: `kubernetes.pod_name`
   - Operator: `contains`
   - Value: `crm-app`

Or use KQL (Kibana Query Language):
```
kubernetes.pod_name: crm-app*
```

## Resource Usage

Optimized for single-node t3.medium (2 vCPU, 4GB RAM):

| Component      | CPU Request | Memory Request | CPU Limit | Memory Limit |
|----------------|-------------|----------------|-----------|--------------|
| Elasticsearch  | 200m        | 512Mi          | 500m      | 1Gi          |
| Kibana         | 100m        | 256Mi          | 250m      | 512Mi        |
| Fluentd        | 100m        | 256Mi          | 200m      | 512Mi        |
| **Total**      | **400m**    | **1024Mi**     | **950m**  | **2Gi**      |

## Troubleshooting

### Elasticsearch Pod Stuck in Pending

Check if there's enough memory:
```bash
kubectl describe pod -n logging logging-stack-elasticsearch-master-0
```

If memory is insufficient, scale down other deployments temporarily.

### No Logs in Kibana

1. Check Fluentd is running:
```bash
kubectl get pods -n logging -l app=fluentd
kubectl logs -n logging -l app=fluentd
```

2. Verify Elasticsearch is accessible:
```bash
kubectl exec -n logging logging-stack-elasticsearch-master-0 -- \
  curl -u elastic:<password> http://localhost:9200/_cat/indices
```

### Kibana Can't Connect to Elasticsearch

Check the Elasticsearch password secret:
```bash
kubectl get secret -n logging \
  logging-stack-elasticsearch-master-credentials -o yaml
```

## Uninstall

```bash
helm uninstall logging-stack -n logging
kubectl delete namespace logging
```

Note: This will delete all log data stored in Elasticsearch.

## Upgrading

To upgrade the stack with new configuration:

```bash
helm upgrade logging-stack . \
  --namespace logging \
  --timeout 10m \
  --wait
```

## Integration with CRM Application

The CRM application outputs JSON-formatted logs that include:
- `timestamp`: ISO 8601 format
- `level`: DEBUG, INFO, WARNING, ERROR
- `message`: Log message
- `correlation_id`: Request tracing ID
- `request_method`: HTTP method
- `request_path`: API endpoint
- `status_code`: HTTP response code

These structured logs are automatically parsed by Fluentd and indexed in Elasticsearch.

## Further Reading

- [Elasticsearch Documentation](https://www.elastic.co/guide/en/elasticsearch/reference/8.5/index.html)
- [Fluentd Documentation](https://docs.fluentd.org/)
- [Kibana Documentation](https://www.elastic.co/guide/en/kibana/8.5/index.html)
