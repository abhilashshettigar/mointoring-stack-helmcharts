# Monitoring Helm Charts

## Overview
This directory contains Helm chart values files for a Kubernetes monitoring stack consisting of:
- **OpenTelemetry Collector** (OTEL)
- **Prometheus**
- **Grafana**

## Architecture

### Data Flow
1. **OTEL Collector** receives metrics via:
   - OTLP protocol (gRPC on port 4317, HTTP on port 4318)
   - Prometheus scraping from YugabyteDB (yb-master:7000, yb-tserver:9000, ysql:13000)
2. Metrics are processed with memory limiter and batch processors
3. Metrics exported to Prometheus format on port 8889
4. **Prometheus** scrapes metrics from OTEL Collector every 15 seconds
5. **Grafana** queries Prometheus as its default datasource and visualizes metrics

## Files

### otel-values.yaml
OpenTelemetry Collector configuration:
- **Image**: `otel/opentelemetry-collector-contrib:0.104.0`
- **Mode**: Deployment
- **RBAC**: ServiceAccount and RBAC enabled for Kubernetes API access
- **Receivers**: 
  - OTLP (gRPC:4317, HTTP:4318)
  - Prometheus scraper for YugabyteDB:
    - yb-master metrics (port 7000)
    - yb-tserver metrics (port 9000)
    - YSQL metrics (port 13000)
- **Exporters**: Prometheus endpoint on port 8889
- **Processors**: 
  - Memory limiter (400 MiB limit)
  - Batch processor (1000 records, 10s timeout)
- **Resources**: 
  - Limits: 500m CPU, 512Mi memory
  - Requests: 100m CPU, 128Mi memory
- **Telemetry**: Internal metrics disabled to avoid port conflicts

### prometheus-values.yaml
Prometheus configuration:
- **Retention**: 15 days
- **Service**: ClusterIP on port 9090
- **Scrape Config**: Polls `otel-collector-opentelemetry-collector:8889` every 15s
- **Monitoring**: ServiceMonitor and PodMonitor selectors enabled
- **Grafana**: Disabled (installed separately)
- **Node Exporter**: Disabled (Docker Desktop mount limitation)

### grafana-values.yaml
Grafana configuration:
- **Admin Password**: `admin`
- **Datasource**: Prometheus at `http://prometheus-kube-prometheus-prometheus:9090`
- **Dashboards**: 
  - OTEL metrics dashboard (ID: 12114)
  - Auto-refresh every 10 seconds
- **Service**: ClusterIP on port 3000

## Deployment Commands

### Initial Installation

#### 1. Install OTEL Collector
```bash
helm install otel-collector opentelemetry-collector \
  --namespace monitoring \
  --create-namespace \
  -f otel-values.yaml \
  --repo https://open-telemetry.github.io/opentelemetry-helm-charts
```

#### 2. Install Prometheus
```bash
helm install prometheus kube-prometheus-stack \
  --namespace monitoring \
  -f prometheus-values.yaml \
  --repo https://prometheus-community.github.io/helm-charts
```

#### 3. Install Grafana
```bash
helm install grafana grafana \
  --namespace monitoring \
  -f grafana-values.yaml \
  --repo https://grafana.github.io/helm-charts
```

### Upgrade Commands

#### Upgrade OTEL Collector
```bash
helm upgrade otel-collector opentelemetry-collector \
  --namespace monitoring \
  -f otel-values.yaml \
  --repo https://open-telemetry.github.io/opentelemetry-helm-charts
```

#### Upgrade Prometheus
```bash
helm upgrade prometheus kube-prometheus-stack \
  --namespace monitoring \
  -f prometheus-values.yaml \
  --repo https://prometheus-community.github.io/helm-charts
```

#### Upgrade Grafana
```bash
helm upgrade grafana grafana \
  --namespace monitoring \
  -f grafana-values.yaml \
  --repo https://grafana.github.io/helm-charts
```

### Access Grafana
```bash
# Port-forward to access Grafana UI
kubectl port-forward -n monitoring svc/grafana 3000:80

# Access at: http://localhost:3000
# Username: admin

# Get the auto-generated password:
kubectl get secret -n monitoring grafana -o jsonpath="{.data.admin-password}" | base64 -d && echo
```

### Import YugabyteDB Production Dashboard

A comprehensive production-level dashboard is available in `grafana-dashboards/yugabytedb-production-dashboard.json`

**Dashboard Features:**
- CPU & Memory Usage
- Cluster Health (Master/TServer nodes up)
- Error & Warning Message Rates
- Read/Write Latency & Throughput
- RocksDB Memory Usage
- RPC Connections
- CQL Query Performance
- WAL Metrics
- Namespace, Service, and Pod filtering

**Import via UI:**
1. Login to Grafana
2. Go to **Dashboards** → **Import**
3. Upload `grafana-dashboards/yugabytedb-production-dashboard.json`
4. Select Prometheus datasource
5. Click Import

See `grafana-dashboards/README.md` for detailed instructions and troubleshooting.

## YugabyteDB Metrics Collection

The YugabyteDB metrics flow through the following path:

1. **OTEL Collector** uses Kubernetes service discovery to find YugabyteDB pods
2. **OTEL Prometheus Receiver** scrapes three endpoints:
   - `yb-master` pods on port 7000 (`/prometheus-metrics`)
   - `yb-tserver` pods on port 9000 (`/prometheus-metrics`)
   - `ysql` endpoints on port 13000 (`/prometheus-metrics`)
3. **OTEL Prometheus Exporter** exposes all collected metrics on port 8889
4. **Prometheus** scrapes OTEL Collector on port 8889 every 15 seconds
5. Metrics are stored in Prometheus with these labels:
   - `job`: "otel-collector-metrics" (Prometheus scrape job)
   - `exported_job`: "yb-master", "yb-tserver", or "ysql" (original OTEL scrape job)
   - `pod`: YugabyteDB pod name
   - `service`: "yb-master", "yb-tserver", or "ysql"

### Example Metrics
- `handler_latency_*`: Request latency metrics
- `mem_tracker_*`: Memory usage metrics
- `rocksdb_*`: RocksDB storage metrics
- `rpc_*`: RPC connection metrics

## Deployment Notes
- All services use ClusterIP (internal cluster access only)
- OTEL Collector has ClusterRole with RBAC permissions for Kubernetes pod discovery
- OTEL scrapes YugabyteDB metrics directly from pods using Prometheus receiver
- Prometheus scrapes aggregated metrics from OTEL Collector on port 8889
- Grafana auto-provisions Prometheus datasource and OTEL dashboards
- Node-exporter disabled due to Docker Desktop mount limitations

## Troubleshooting

### Check Pod Status
```bash
kubectl get pods -n monitoring
```

### Common Issues

**OTEL Collector CrashLoopBackOff with "variable substitution" error:**
- Issue: Prometheus relabel config uses `$1` which OTEL interprets as env var
- Solution: Use `'$$1'` with single quotes in YAML to escape properly
- Example: `replacement: '$$1:7000'` instead of `replacement: $1:7000`

**Prometheus not scraping OTEL Collector:**
- Issue: Port 8889 not exposed in OTEL collector service
- Solution: Add `ports.prometheus` section in `otel-values.yaml`
- Use `additionalScrapeConfigs` (not `scrapeConfigs`) in `prometheus-values.yaml`

**Prometheus node-exporter CrashLoopBackOff:**
- Issue: Docker Desktop doesn't support required mount types
- Solution: Disable with `prometheus-node-exporter.enabled: false`
- Delete daemonset: `kubectl delete daemonset prometheus-prometheus-node-exporter -n monitoring`

### View Logs
```bash
# OTEL Collector
kubectl logs -n monitoring deployment/otel-collector-opentelemetry-collector

# Prometheus
kubectl logs -n monitoring prometheus-prometheus-kube-prometheus-prometheus-0 -c prometheus

# Grafana
kubectl logs -n monitoring deployment/grafana
```

### Verify Services
```bash
kubectl get svc -n monitoring
```

### Verify Prometheus is Scraping OTEL Collector
```bash
# Check if OTEL collector target is up
kubectl exec -n monitoring prometheus-prometheus-kube-prometheus-prometheus-0 -c prometheus -- \
  wget -qO- http://localhost:9090/api/v1/targets 2>/dev/null | \
  jq -r '.data.activeTargets[] | select(.scrapePool == "otel-collector-metrics") | {health, lastError}'

# Should show: {"health": "up", "lastError": ""}
```

### Query Metrics from Prometheus
```bash
# Check if OTEL collector is up
kubectl exec -n monitoring prometheus-prometheus-kube-prometheus-prometheus-0 -c prometheus -- \
  wget -qO- 'http://localhost:9090/api/v1/query?query=up{job="otel-collector-metrics"}' 2>/dev/null | \
  jq -r '.data.result[0].value[1]'

# Should return: "1"

# Check YugabyteDB metrics (scraped by OTEL, then by Prometheus)
kubectl exec -n monitoring prometheus-prometheus-kube-prometheus-prometheus-0 -c prometheus -- \
  wget -qO- 'http://localhost:9090/api/v1/query?query=handler_latency_yb_tserver_TabletServerService_Read' 2>/dev/null | \
  jq -r '.data.result[] | {exported_job: .metric.exported_job, pod: .metric.pod, service: .metric.service}'

# Should show metrics with exported_job="yb-tserver", exported_job="yb-master", etc.
```
