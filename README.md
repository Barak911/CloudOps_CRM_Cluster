# Cluster Resources - Kubernetes Helm Charts

This repository contains **Helm charts** for deploying the CRM application stack and observability infrastructure to the EKS cluster.

## Repository Structure

```
.
├── crm-stack/                    # Main CRM application Helm chart (umbrella)
│   ├── Chart.yaml               # Chart metadata and dependencies
│   ├── values.yaml              # Unified configuration for all components
│   └── charts/
│       ├── mongodb/             # MongoDB StatefulSet subchart
│       │   ├── Chart.yaml
│       │   └── templates/
│       │       ├── statefulset.yaml
│       │       └── service.yaml
│       └── crm-app/             # CRM application Deployment subchart
│           ├── Chart.yaml
│           └── templates/
│               ├── deployment.yaml
│               └── service.yaml
│
├── logging-stack/               # EFK logging infrastructure Helm chart
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│
├── crm-servicemonitor.yaml      # Prometheus ServiceMonitor for metrics
├── kibana-standalone-values.yaml # Standalone Kibana configuration
└── README.md                    # This file
```

## Components

### 1. CRM Stack (Umbrella Chart)

The main `crm-stack` chart includes:

- **MongoDB** - StatefulSet with persistent storage
  - Mongo 7.0 image
  - 5Gi persistent volume
  - Health checks with mongosh
  - Resource limits: 512Mi memory, 500m CPU

- **CRM Application** - Deployment with ClusterIP service
  - Configurable replica count (default: 1)
  - ECR image with dynamic tag override
  - Health checks via /health endpoint
  - Resource limits: 256Mi memory, 200m CPU
  - ClusterIP service (accessed via Nginx Ingress)

- **EFK Stack** (Optional) - Logging infrastructure
  - Elasticsearch 8.5.1
  - Kibana 8.5.1
  - Fluentd DaemonSet
  - Disabled by default for initial deployment

### 2. Nginx Ingress Controller

Single entry point for all services via AWS Network Load Balancer:
- Deployed to `ingress-nginx` namespace
- Path-based routing to services:
  - `/` → CRM Application
  - `/kibana` → Kibana Dashboard
  - `/grafana` → Grafana (when Prometheus enabled)
- Configuration: `nginx-ingress-values.yaml`

### 3. Logging Stack

Separate Helm chart for comprehensive EFK (Elasticsearch, Fluentd, Kibana) logging infrastructure.

## Prerequisites

- EKS cluster is running (infrastructure deployed via Terraform)
- kubectl configured to connect to the cluster
- Helm 3.x installed
- ECR repository exists and contains the application image
- AWS credentials configured

## Deployment Instructions

### Quick Start - Deploy CRM Application Only

```bash
# 1. Connect to cluster
aws eks update-kubeconfig --name develeap-eks-cluster --region us-east-1

# 2. Build Helm dependencies
cd crm-stack
helm dependency build

# 3. Deploy CRM stack (MongoDB + Application)
helm upgrade --install crm-stack . \
  --namespace default \
  --set crm-app.enabled=true \
  --set crm-app.image.tag=latest \
  --set mongodb.enabled=true \
  --set elasticsearch.enabled=false \
  --set kibana.enabled=false \
  --set fluentd.enabled=false \
  --wait \
  --timeout 10m
```

### Deployment with Custom Image Tag

```bash
# From CI/CD pipeline
ECR_REGISTRY="253490775265.dkr.ecr.us-east-1.amazonaws.com"
IMAGE_TAG="abc123def"

helm upgrade --install crm-stack ./crm-stack \
  --namespace default \
  --set crm-app.image.repository=$ECR_REGISTRY/crm-app \
  --set crm-app.image.tag=$IMAGE_TAG \
  --set crm-app.enabled=true \
  --set mongodb.enabled=true \
  --set elasticsearch.enabled=false \
  --set kibana.enabled=false \
  --set fluentd.enabled=false \
  --wait
```

### Deploy with Full Observability (EFK Stack)

```bash
helm upgrade --install crm-stack ./crm-stack \
  --namespace default \
  --set crm-app.enabled=true \
  --set mongodb.enabled=true \
  --set elasticsearch.enabled=true \
  --set kibana.enabled=true \
  --set fluentd.enabled=true \
  --wait \
  --timeout 15m
```

## Configuration

### Key Values (values.yaml)

```yaml
# CRM Application
crm-app:
  enabled: true
  replicaCount: 1
  image:
    repository: 253490775265.dkr.ecr.us-east-1.amazonaws.com/crm-app
    tag: latest
    pullPolicy: Always
  service:
    type: LoadBalancer
    port: 80

# MongoDB
mongodb:
  enabled: true
  replicaCount: 1
  persistence:
    enabled: true
    size: 5Gi

# Logging Stack (Optional)
elasticsearch:
  enabled: false
kibana:
  enabled: false
fluentd:
  enabled: false
```

## Verification

### Check Deployment Status

```bash
# Helm release status
helm list -n default
helm status crm-stack -n default

# Kubernetes resources
kubectl get all -n default

# Check pods
kubectl get pods -l app=mongodb
kubectl get pods -l app=crm-app

# Check services
kubectl get svc -l app=crm-app
```

### Get Application URL (via Nginx Ingress)

```bash
# Get Ingress LoadBalancer URL
kubectl get service ingress-nginx-controller -n ingress-nginx

# Or with jsonpath
export INGRESS_URL=$(kubectl get service ingress-nginx-controller -n ingress-nginx -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "CRM Application: http://$INGRESS_URL/"
echo "Kibana:          http://$INGRESS_URL/kibana"
echo "Grafana:         http://$INGRESS_URL/grafana"
```

## Testing the Deployment

### Check Pod Logs

```bash
# MongoDB logs
kubectl logs -l app=mongodb --tail=50

# Application logs
kubectl logs -l app=crm-app --tail=50
```

### Test Application Endpoints

```bash
# Get Ingress URL
INGRESS_URL=$(kubectl get service ingress-nginx-controller -n ingress-nginx -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

# Test health endpoint
curl http://$INGRESS_URL/health

# Test root endpoint
curl http://$INGRESS_URL/

# Add a person
curl -X POST http://$INGRESS_URL/person/123 \
  -H "Content-Type: application/json" \
  -d '{"name":"John Doe","email":"john@example.com","phone":"555-1234"}'

# Get all persons
curl http://$INGRESS_URL/person

# Get person by ID
curl http://$INGRESS_URL/person/123

# Update person
curl -X PUT http://$INGRESS_URL/person/123 \
  -H "Content-Type: application/json" \
  -d '{"name":"John Updated","email":"updated@example.com","phone":"555-9999"}'

# Delete person
curl -X DELETE http://$INGRESS_URL/person/123
```

## Updating the Application

### Update to New Image Tag

```bash
# Update with new image tag
helm upgrade crm-stack ./crm-stack \
  --namespace default \
  --set crm-app.image.tag=new-tag-xyz \
  --reuse-values

# Check rollout status
kubectl rollout status deployment/crm-stack-crm-app -n default
```

### Update Configuration Values

```bash
# Edit values.yaml, then upgrade
helm upgrade crm-stack ./crm-stack \
  --namespace default \
  --values values.yaml

# Or use --set for specific values
helm upgrade crm-stack ./crm-stack \
  --namespace default \
  --set crm-app.replicaCount=3 \
  --reuse-values
```

### Rollback to Previous Version

```bash
# List release history
helm history crm-stack -n default

# Rollback to previous revision
helm rollback crm-stack -n default

# Rollback to specific revision
helm rollback crm-stack 2 -n default
```

## Monitoring and Observability

### Deploy Prometheus ServiceMonitor

```bash
kubectl apply -f crm-servicemonitor.yaml
```

### Access Kibana (if EFK enabled)

```bash
# Get Ingress URL
INGRESS_URL=$(kubectl get service ingress-nginx-controller -n ingress-nginx -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

# Access Kibana via Ingress
echo "Kibana URL: http://$INGRESS_URL/kibana"

# Default credentials: elastic / <password from elasticsearch-master-credentials secret>
```

### View Elasticsearch Password

```bash
kubectl get secret elasticsearch-master-credentials -n default -o jsonpath='{.data.password}' | base64 -d
```

## Troubleshooting

### Pods Not Starting

```bash
# Describe pod for events
kubectl describe pod <pod-name>

# Check logs
kubectl logs <pod-name>

# Check previous pod logs (if crashed)
kubectl logs <pod-name> --previous
```

### Helm Chart Issues

```bash
# Dry-run to validate chart
helm install crm-stack ./crm-stack --dry-run --debug

# Template rendering
helm template crm-stack ./crm-stack > rendered.yaml

# List all resources created by release
kubectl get all -l app.kubernetes.io/instance=crm-stack
```

### Cannot Pull ECR Image

- Verify ECR repository exists: `aws ecr describe-repositories --region us-east-1`
- Check IAM permissions for EKS node role to pull from ECR
- Confirm image tag exists: `aws ecr list-images --repository-name crm-app --region us-east-1`

### MongoDB Connection Issues

```bash
# Check MongoDB service
kubectl get service -l app=mongodb

# Test DNS resolution from app pod
POD=$(kubectl get pod -l app=crm-app -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $POD -- nslookup mongodb.default.svc.cluster.local

# Check MongoDB connectivity
kubectl exec -it $POD -- curl -v telnet://mongodb:27017
```

### LoadBalancer Not Getting External IP

- Wait 3-5 minutes for AWS to provision ELB
- Check AWS console for ELB/CLB creation
- Verify VPC has Internet Gateway attached
- Check security group rules allow traffic

### Helm Dependencies Not Found

```bash
# Update Helm repositories
helm repo add elastic https://helm.elastic.co
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update

# Rebuild dependencies
cd crm-stack
helm dependency build
```

## Cleanup

### Uninstall Helm Release

```bash
# Uninstall the release
helm uninstall crm-stack -n default

# Verify removal
helm list -n default
kubectl get all -n default
```

### Delete Persistent Volumes

```bash
# List PVCs
kubectl get pvc -n default

# Delete MongoDB PVC (data will be lost!)
kubectl delete pvc mongodb-crm-stack-mongodb-data-crm-stack-mongodb-0 -n default

# Delete all PVCs
kubectl delete pvc --all -n default
```

### Complete Cleanup

```bash
# Delete Helm release
helm uninstall crm-stack -n default

# Delete standalone resources
kubectl delete -f crm-servicemonitor.yaml

# Delete PVCs
kubectl delete pvc --all -n default

# Verify cleanup
kubectl get all,pvc -n default
```

## CI/CD Integration

This repository is designed to work with the GitHub Actions workflow in the application repository:
- **Workflow**: `.github/workflows/initial-deploy.yml`
- **Repository**: `Barak911/CloudOps_CRM`

The workflow:
1. Builds and tests the application
2. Pushes Docker image to ECR
3. Clones this cluster-resources repository
4. Deploys using Helm with the new image tag
5. Runs integration tests
6. Generates deployment report

## Important Notes

- MongoDB uses persistent storage - data survives pod restarts and redeployments
- PersistentVolumeClaims must be manually deleted when uninstalling
- Single Nginx Ingress LoadBalancer serves all services (reduces AWS costs)
- Helm release name `crm-stack` prefixes all resource names
- Resource names: `crm-stack-crm-app`, `crm-stack-mongodb`
- Health checks ensure traffic only routes to healthy pods
- Resource limits prevent pods from consuming excessive CPU/memory
- EFK stack is resource-intensive - only enable when needed

## Architecture

**CRM Application Stack (with Nginx Ingress):**
```
┌─────────────────────────────────────────────────────────┐
│            AWS Network Load Balancer (NLB)              │
│        (single LoadBalancer for all services)           │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│           Nginx Ingress Controller                      │
│              (ingress-nginx namespace)                  │
└────────────────────────┬────────────────────────────────┘
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
   ┌─────────┐    ┌───────────┐    ┌──────────┐
   │ /       │    │ /kibana   │    │ /grafana │
   │ CRM App │    │ Kibana    │    │ Grafana  │
   │ :80     │    │ :5601     │    │ :80      │
   └────┬────┘    └───────────┘    └──────────┘
        │         (ClusterIP)      (ClusterIP)
        ▼
┌─────────────────────────────────────────────────────────┐
│  CRM Application Pods (crm-stack-crm-app)               │
│  - Deployment (1+ replicas)                             │
│  - Health checks on /health                             │
│  - Connects to MongoDB via internal DNS                 │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│  MongoDB StatefulSet (crm-stack-mongodb)                │
│  - 1 replica with persistent storage                    │
│  - Headless service for stable identity                 │
│  - 5Gi PersistentVolume                                 │
└─────────────────────────────────────────────────────────┘
```

## Support

For issues or questions:
- Check the troubleshooting section above
- Review Helm chart values: `helm get values crm-stack -n default`
- Inspect rendered templates: `helm template crm-stack ./crm-stack`
- Check pod events: `kubectl describe pod <pod-name>`
