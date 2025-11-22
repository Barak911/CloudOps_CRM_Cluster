# Cluster Resources - Kubernetes Manifests

This repository stores **Kubernetes manifests** and future **Helm charts** for deploying the CRM application and supporting services to the EKS cluster.

## Repository Structure

```
.
├── app-deployment.yaml      # CRM Application Deployment
├── app-service.yaml         # CRM Application Service (LoadBalancer)
├── mongodb-deployment.yaml  # MongoDB StatefulSet
├── mongodb-service.yaml     # MongoDB Service (headless)
├── namespace.yaml           # Default namespace configuration
└── README.md               # This file
```

**Future structure:**
* `k8s/` – Raw YAML manifests (Deployment, Service, ConfigMap, etc.)
* `helm/` – Helm chart (to be created later)
* `.github/workflows/` – Pipeline that applies manifests when a new image tag is released

## Components

1. **namespace.yaml** - Default namespace configuration
2. **mongodb-deployment.yaml** - MongoDB StatefulSet
3. **mongodb-service.yaml** - MongoDB Service (headless for StatefulSet)
4. **app-deployment.yaml** - CRM Application Deployment
5. **app-service.yaml** - CRM Application Service (LoadBalancer)

## Prerequisites

- EKS cluster is running (infrastructure deployed via Terraform)
- kubectl is configured to connect to the cluster
- ECR repository exists and contains the application image

## Deployment Instructions

### 1. Connect to Cluster

```bash
aws eks update-kubeconfig --name develeap-eks-cluster --region us-east-1
```

### 2. Verify Connection

```bash
kubectl get nodes
```

### 3. Update Image URL

Edit `app-deployment.yaml` and replace `<AWS_ACCOUNT_ID>` with your actual AWS account ID:

```yaml
image: <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/crm-app:latest
```

### 4. Deploy MongoDB

```bash
kubectl apply -f mongodb-deployment.yaml
kubectl apply -f mongodb-service.yaml
```

Wait for MongoDB to be ready:

```bash
kubectl get pods -l app=mongodb -w
```

### 5. Deploy Application

```bash
kubectl apply -f app-deployment.yaml
kubectl apply -f app-service.yaml
```

### 6. Check Deployment Status

```bash
kubectl get all
```

### 7. Get Application URL

```bash
kubectl get service crm-app
```

The LoadBalancer will provide an external URL (takes a few minutes to provision).

## Architecture

- **MongoDB**: StatefulSet with persistent storage (5Gi)
  - Headless service for stable network identity
  - 1 replica for dev environment
  - Resource limits: 256Mi memory, 250m CPU

- **Application**: Deployment with 2 replicas
  - LoadBalancer service for external access
  - Resource limits: 256Mi memory, 200m CPU
  - Health checks via /health endpoint
  - Connects to MongoDB via internal DNS

## Testing the Deployment

### Check Pod Logs

```bash
# MongoDB logs
kubectl logs -l app=mongodb

# Application logs
kubectl logs -l app=crm-app
```

### Test Application

```bash
# Get the LoadBalancer URL
export APP_URL=$(kubectl get service crm-app -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

# Test health endpoint
curl http://$APP_URL/health

# Test API
curl http://$APP_URL/

# Add a person
curl -X POST http://$APP_URL/person/123 \
  -H "Content-Type: application/json" \
  -d '{"name":"John Doe","email":"john@example.com"}'

# Get all persons
curl http://$APP_URL/person
```

## Updating the Application

### Manual Update

```bash
# Update image tag in app-deployment.yaml, then:
kubectl apply -f app-deployment.yaml

# Check rollout status
kubectl rollout status deployment/crm-app
```

### Via CI/CD (Automated)

The GitHub Actions workflows in the application repository automatically build, test, and push images to ECR. This repository's pipeline will watch for new image tags and trigger `kubectl apply` / `helm upgrade` against the EKS cluster.

## Troubleshooting

### Pods Not Starting

```bash
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

### Cannot Pull Image

- Verify ECR repository exists
- Check IAM permissions for EKS nodes to pull from ECR
- Confirm image tag exists in ECR

### MongoDB Connection Issues

```bash
# Check MongoDB service
kubectl get service mongodb

# Test DNS resolution from app pod
kubectl exec -it <app-pod-name> -- nslookup mongodb.default.svc.cluster.local
```

### LoadBalancer Not Getting External IP

- Wait 3-5 minutes for AWS to provision
- Check AWS console for ELB/ALB creation
- Verify security group rules

## Cleanup

### Delete Application

```bash
kubectl delete -f app-service.yaml
kubectl delete -f app-deployment.yaml
```

### Delete MongoDB

```bash
kubectl delete -f mongodb-service.yaml
kubectl delete -f mongodb-deployment.yaml
```

### Delete Persistent Volumes

```bash
kubectl delete pvc mongodb-data-mongodb-0
```

## Important Notes

- MongoDB uses persistent storage - data survives pod restarts
- LoadBalancer creates an AWS ELB - additional cost (clean up when done!)
- Application has 2 replicas for basic high availability
- Health checks ensure traffic only goes to healthy pods
- Resource limits prevent pods from consuming too much CPU/memory
