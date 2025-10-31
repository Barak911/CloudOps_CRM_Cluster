# Monitoring Stack Documentation

This document describes the monitoring infrastructure deployed for the CRM application using Prometheus and Grafana.

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    EKS Cluster                          │
│                                                         │
│  ┌──────────────┐      ┌─────────────────────────┐    │
│  │              │      │  Monitoring Namespace   │    │
│  │  CRM App     │◄─────┤                         │    │
│  │  /metrics    │      │  ┌────────────────┐     │    │
│  │  endpoint    │      │  │   Prometheus   │     │    │
│  │              │      │  │   (scraping)   │     │    │
│  └──────────────┘      │  └────────┬───────┘     │    │
│                        │           │             │    │
│  ┌──────────────┐      │  ┌────────▼───────┐     │    │
│  │   MongoDB    │      │  │    Grafana     │     │    │
│  │              │      │  │  (dashboards)  │     │    │
│  └──────────────┘      │  └────────────────┘     │    │
│                        │                         │    │
│  ┌──────────────┐      │  ┌────────────────┐     │    │
│  │ Node Exporter│◄─────┤  │ Alert Manager  │     │    │
│  └──────────────┘      │  └────────────────┘     │    │
│                        └─────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
                                │
                                │ LoadBalancer
                                ▼
                         External Access
                    (Grafana Dashboard UI)
```

## Components

### 1. Prometheus
**Purpose:** Metrics collection, storage, and querying

**Configuration:**
- **Namespace:** monitoring
- **Storage:** 10Gi persistent volume
- **Retention:** 7 days
- **Scrape Interval:** 30 seconds
- **Resources:**
  - CPU: 200m request / 500m limit
  - Memory: 512Mi request / 1Gi limit

**Metrics Collected:**
- Application metrics from `/metrics` endpoint
- Kubernetes cluster metrics (nodes, pods, deployments)
- Node-level metrics (CPU, memory, disk, network)

### 2. Grafana
**Purpose:** Visualization and dashboarding

**Configuration:**
- **Namespace:** monitoring
- **Access:** LoadBalancer (external)
- **Persistence:** 5Gi for dashboards
- **Resources:**
  - CPU: 100m request / 300m limit
  - Memory: 256Mi request / 512Mi limit

**Credentials:**
- **Username:** admin
- **Password:** admin (change in production!)

**Pre-installed Dashboards:**
- Kubernetes Cluster Monitoring
- Node Exporter Full
- CRM Application Metrics (custom)

### 3. AlertManager
**Purpose:** Alert routing and management

**Configuration:**
- **Namespace:** monitoring
- **Resources:**
  - CPU: 50m request / 100m limit
  - Memory: 128Mi request / 256Mi limit

### 4. Node Exporter
**Purpose:** Host-level metrics collection

**Configuration:**
- **Type:** DaemonSet (runs on all nodes)
- **Resources:**
  - CPU: 50m request / 100m limit
  - Memory: 64Mi request / 128Mi limit

### 5. Kube State Metrics
**Purpose:** Kubernetes object state metrics

**Configuration:**
- **Namespace:** monitoring
- **Resources:**
  - CPU: 50m request / 100m limit
  - Memory: 128Mi request / 256Mi limit

## Application Metrics

The CRM application exposes Prometheus metrics at the `/metrics` endpoint.

### Metrics Available

#### Request Metrics
```
flask_http_request_total
  - Counter: Total HTTP requests
  - Labels: method, status, path

flask_http_request_duration_seconds
  - Histogram: Request duration in seconds
  - Labels: method, path
  - Buckets: 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0

flask_http_request_exceptions_total
  - Counter: Total unhandled exceptions
  - Labels: method, status
```

#### Application Info
```
crm_app_info
  - Gauge: Application version and environment information
  - Labels: version, environment
```

### Example Queries

**Request rate by endpoint:**
```promql
rate(flask_http_request_total{job="crm-app"}[5m])
```

**95th percentile request duration:**
```promql
histogram_quantile(0.95, rate(flask_http_request_duration_seconds_bucket{job="crm-app"}[5m]))
```

**Error rate (5xx responses):**
```promql
rate(flask_http_request_total{job="crm-app",status=~"5.."}[5m])
```

**Total requests by status code:**
```promql
sum by (status) (flask_http_request_total{job="crm-app"})
```

## ServiceMonitor Configuration

The CRM application is monitored using a Prometheus ServiceMonitor custom resource:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: crm-app-metrics
  namespace: default
  labels:
    app: crm-app
    release: prometheus
spec:
  selector:
    matchLabels:
      app: crm-app
  endpoints:
  - port: http
    path: /metrics
    interval: 30s
    scrapeTimeout: 10s
```

**Key Points:**
- Label selector matches CRM app service
- Scrapes `/metrics` endpoint every 30 seconds
- 10-second timeout for scrape operations
- Must have `release: prometheus` label for discovery

## Accessing Grafana

### 1. Get LoadBalancer URL
```bash
kubectl get svc prometheus-grafana -n monitoring -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

**Current URL:**
```
http://ac221706449b74ea490fc8f807c3eb9b-1321017938.us-east-1.elb.amazonaws.com
```

### 2. Login
- **Username:** admin
- **Password:** admin

### 3. Explore Dashboards
Navigate to: **Dashboards** → **Browse**

Available dashboards:
- **Kubernetes / Compute Resources / Cluster**
- **Kubernetes / Compute Resources / Namespace (Pods)**
- **Node Exporter / Nodes**
- **CRM Application Metrics** (custom)

## Importing the CRM Dashboard

### Method 1: Via Grafana UI
1. Login to Grafana
2. Click **+** (Create) → **Import**
3. Upload `grafana-crm-dashboard.json` file
4. Click **Import**

### Method 2: Via kubectl ConfigMap
```bash
kubectl create configmap crm-dashboard \
  --from-file=grafana-crm-dashboard.json \
  -n monitoring \
  --dry-run=client -o yaml | kubectl apply -f -
```

## CRM Application Dashboard

The custom dashboard includes:

### Panels

1. **HTTP Request Rate**
   - Time series graph
   - Requests per second by method and status
   - 5-minute rate

2. **HTTP Request Duration (p95)**
   - 95th percentile latency
   - Grouped by method and path
   - Helps identify slow endpoints

3. **Total HTTP Requests by Status Code**
   - Stat panel
   - Shows distribution of 2xx, 4xx, 5xx responses

4. **Error Rate (5xx)**
   - Stat panel with thresholds
   - Green: 0 errors/sec
   - Yellow: > 0.1 errors/sec
   - Red: > 1 error/sec

5. **Request Count by Endpoint**
   - Horizontal stat panel
   - Shows which endpoints are most used

6. **HTTP Methods Distribution**
   - Pie chart
   - GET, POST, PUT, DELETE distribution

7. **Average Request Size**
   - Time series graph
   - Average request duration by endpoint

8. **Application Info**
   - Static panel
   - Shows application version and environment

## Alerting

### Recommended Alerts

Create alerts in Prometheus AlertManager for:

**High Error Rate:**
```yaml
- alert: HighErrorRate
  expr: rate(flask_http_request_total{status=~"5.."}[5m]) > 0.1
  for: 5m
  annotations:
    summary: "High error rate detected"
    description: "Error rate is {{ $value }} requests/sec"
```

**High Latency:**
```yaml
- alert: HighLatency
  expr: histogram_quantile(0.95, rate(flask_http_request_duration_seconds_bucket[5m])) > 2
  for: 5m
  annotations:
    summary: "High request latency detected"
    description: "P95 latency is {{ $value }} seconds"
```

**Application Down:**
```yaml
- alert: ApplicationDown
  expr: up{job="crm-app"} == 0
  for: 1m
  annotations:
    summary: "CRM application is down"
    description: "Cannot scrape metrics from CRM app"
```

## Troubleshooting

### Prometheus Not Scraping Metrics

**Check ServiceMonitor:**
```bash
kubectl get servicemonitor crm-app-metrics -n default
```

**Check Prometheus targets:**
1. Port-forward Prometheus:
   ```bash
   kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090
   ```
2. Open http://localhost:9090/targets
3. Look for "crm-app" job
4. Check if status is "UP"

**Common Issues:**
- ServiceMonitor label mismatch
- Service selector not matching pods
- Network policy blocking scraping
- Application `/metrics` endpoint not responding

### Grafana Dashboard Not Showing Data

**Check Prometheus datasource:**
1. Login to Grafana
2. **Configuration** → **Data Sources**
3. Verify Prometheus URL: `http://prometheus-kube-prometheus-prometheus.monitoring:9090`
4. Click **Test** button

**Verify metrics exist in Prometheus:**
1. Port-forward Prometheus (see above)
2. Run query: `flask_http_request_total`
3. Should return results with CRM app metrics

### No Metrics from Application

**Test /metrics endpoint:**
```bash
# From outside cluster
CRM_URL=$(kubectl get svc crm-app -n default -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl http://$CRM_URL/metrics

# From inside cluster
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- \
  curl http://crm-app.default.svc.cluster.local/metrics
```

**Expected output:**
```
# HELP flask_http_request_total Total HTTP requests
# TYPE flask_http_request_total counter
flask_http_request_total{method="GET",status="200",path="/"} 42
...
```

## Cost Optimization

**Resource Configuration:**
- All components use resource requests and limits
- Single node deployment optimized for t3a.medium
- 7-day retention (instead of default 15 days)
- Single NAT gateway
- Storage: 15Gi total (10Gi Prometheus + 5Gi Grafana)

**Estimated Cost Impact:**
- Storage: ~$1.50/month (EBS gp3)
- LoadBalancer (Grafana): ~$16/month
- Compute: No additional cost (same node)

**Total:** ~$17.50/month additional cost

## Cleanup

To remove the monitoring stack:

```bash
# Uninstall Helm release
helm uninstall prometheus -n monitoring

# Remove namespace
kubectl delete namespace monitoring

# Remove ServiceMonitor
kubectl delete servicemonitor crm-app-metrics -n default
```

## Best Practices

1. **Change Default Passwords**
   - Update Grafana admin password immediately
   - Store in secrets manager

2. **Enable HTTPS**
   - Use Ingress with TLS for Grafana
   - Protect metrics endpoint

3. **Configure Alerts**
   - Set up AlertManager notifications (email, Slack)
   - Define SLOs and alert on violations

4. **Regular Backups**
   - Export Grafana dashboards to Git
   - Backup Prometheus data if long-term retention needed

5. **Monitor the Monitors**
   - Set alerts for Prometheus/Grafana downtime
   - Monitor disk usage for persistence volumes

## Next Steps

- [ ] Change Grafana admin password
- [ ] Configure AlertManager (email/Slack)
- [ ] Set up alerts for critical metrics
- [ ] Deploy Nginx Ingress for HTTPS access
- [ ] Create additional custom dashboards
- [ ] Implement log aggregation (ELK/EFK stack)
- [ ] Add application-specific business metrics

## References

- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)
- [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
- [Prometheus Flask Exporter](https://github.com/rycus86/prometheus_flask_exporter)
