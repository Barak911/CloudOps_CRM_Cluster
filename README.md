# Cluster Resources Repository

Stores **Kubernetes manifests** and future **Helm charts** for deploying the application and supporting micro-services.

Structure:
* `k8s/` – Raw YAML manifests (Deployment, Service, ConfigMap, etc.)
* `helm/` – Helm chart (to be created later)
* `.github/workflows/` – Pipeline that applies manifests when a new image tag is released

---

> The pipeline will watch the `application` repo’s container registry tags and trigger `kubectl apply` / `helm upgrade` against the EKS cluster.
