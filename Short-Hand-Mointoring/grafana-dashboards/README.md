# YugabyteDB Production Dashboard

## Overview
This dashboard provides comprehensive production-level monitoring for YugabyteDB clusters with:
- Real-time metrics visualization
- Namespace filtering
- Service and Pod selection
- 30-second auto-refresh

## Dashboard Panels

### System Resources
1. **CPU Usage (%)** - Shows CPU usage as percentage of available CPU
   - **User**: Application code execution time
   - **System**: Kernel/system calls time
   - **Total**: Combined user + system (most important metric)
   - Formula: `rate(cpu_time_ms[5m]) / 10` (converts ms/sec to %)
   - Reading: 0.5% = idle, 5-10% = light, 30-50% = moderate, 80%+ = heavy
   - Note: Can exceed 100% on multi-core systems (200% = 2 cores fully used)
2. **Memory Usage** - Total memory consumption per pod in bytes

### Cluster Health
3. **YB-Master Nodes Up** - Count of healthy master nodes (gauge)
4. **YB-TServer Nodes Up** - Count of healthy tablet server nodes (gauge)
5. **Error Messages Rate** - Rate of error log messages (with thresholds)
6. **Warning Messages Rate** - Rate of warning log messages (with thresholds)

### Performance Metrics
7. **TServer Read/Write Latency** - Average latency for read and write operations (ms)
8. **TServer Read/Write Throughput** - Operations per second for reads and writes
9. **YSQL Query Latency** - Average latency for SELECT, INSERT, UPDATE statements (ms)
10. **YSQL Query Throughput** - Queries per second by statement type (SELECT/INSERT/UPDATE/DELETE)

### Storage & Memory
11. **RocksDB Memory Usage** - MemTable and BlockCache memory consumption
12. **TServer Memory Breakdown** - Detailed memory usage: Total, MemTable, BlockCache, RPC Calls
13. **WAL Log Bytes** - Write-Ahead Log size tracking
14. **WAL Append Rate** - WAL append operations per second

### Network
14. **RPC Connections** - Active RPC connections per pod

## Dashboard Variables

The dashboard includes four template variables for flexible filtering:

1. **Datasource** - Select Prometheus datasource
2. **Namespace** - Filter by Kubernetes namespace (default: All)
3. **Service** - Filter by YugabyteDB service (yb-master, yb-tserver, ysql) - Multi-select
4. **Pod** - Filter by specific pods - Multi-select

## Import Instructions

### Method 1: Via Grafana UI
1. Access Grafana at `http://localhost:3000` (after port-forward)
2. Login with credentials:
   - Username: `admin`
   - Password: Run `kubectl get secret -n monitoring grafana -o jsonpath="{.data.admin-password}" | base64 -d`
3. Click **Dashboards** → **Import** (or use the + icon)
4. Click **Upload JSON file**
5. Select `yugabytedb-production-dashboard.json`
6. Select your Prometheus datasource
7. Click **Import**

### Method 2: Via ConfigMap (Auto-provisioning)

Add the dashboard to Grafana's values file for automatic provisioning:

```yaml
grafana:
  dashboardProviders:
    dashboardproviders.yaml:
      apiVersion: 1
      providers:
        - name: 'yugabytedb'
          orgId: 1
          folder: 'YugabyteDB'
          type: file
          disableDeletion: false
          updateIntervalSeconds: 10
          allowUpdates: true
          options:
            path: /var/lib/grafana/dashboards/yugabytedb

  dashboards:
    yugabytedb:
      yugabytedb-production:
        file: grafana-dashboards/yugabytedb-production-dashboard.json
```

Then upgrade Grafana:
```bash
helm upgrade grafana grafana \
  --namespace monitoring \
  -f grafana-values.yaml \
  --repo https://grafana.github.io/helm-charts
```

### Method 3: Using kubectl and ConfigMap

```bash
# Create a configmap with the dashboard
kubectl create configmap yugabytedb-dashboard \
  --from-file=yugabytedb-production-dashboard.json \
  -n monitoring

# Add appropriate labels for Grafana sidecar to pick it up
kubectl label configmap yugabytedb-dashboard \
  grafana_dashboard=1 \
  -n monitoring
```

## Accessing the Dashboard

1. Port-forward to Grafana:
   ```bash
   kubectl port-forward -n monitoring svc/grafana 3000:80
   ```

2. Open browser to: `http://localhost:3000`

3. Navigate to **Dashboards** → **Browse** → Select "YugabyteDB Production Monitoring"

## Key Metrics for Production Monitoring

### Critical Alerts
- **Master/TServer Nodes Down**: If gauge shows < expected count
- **Error Rate > 100**: High error message rate indicates issues
- **Read/Write Latency > 50ms**: Performance degradation
- **Memory Usage trending up**: Potential memory leak or heavy load

### Performance Tuning
- Monitor **RocksDB Memory** for cache efficiency
- Watch **WAL Append Rate** for write performance
- Track **RPC Connections** for connection pooling
- Check **CPU Usage** for resource constraints

### Capacity Planning
- **Memory Usage trends**: Plan for scaling
- **Throughput trends**: Understand growth patterns
- **Log bytes**: Storage capacity planning

## Customization

The dashboard JSON can be edited to:
- Add more panels for specific metrics
- Adjust threshold values
- Modify time ranges and refresh rates
- Add annotations for deployments/incidents
- Create alerts based on panel queries

## Troubleshooting

### Dashboard shows "No Data"
- Verify Prometheus datasource is configured
- Check that OTEL collector is scraping YugabyteDB pods
- Ensure namespace filter matches your deployment
- Run verification command:
  ```bash
  kubectl exec -n monitoring prometheus-prometheus-kube-prometheus-prometheus-0 -c prometheus -- \
    wget -qO- 'http://localhost:9090/api/v1/query?query=up{exported_job="yb-master"}' 2>/dev/null
  ```

### Variables not populating
- Check Prometheus datasource connection
- Verify metrics are being collected:
  ```bash
  kubectl exec -n monitoring prometheus-prometheus-kube-prometheus-prometheus-0 -c prometheus -- \
    wget -qO- 'http://localhost:9090/api/v1/label/__name__/values' 2>/dev/null | \
    jq -r '.data[]' | grep handler_latency
  ```

### Panels showing errors
- Some metrics may not be available depending on YugabyteDB workload
- YSQL metrics require active YSQL connections and queries
- Some panels may show "No data" until workload is applied
- Adjust queries based on your specific use case

## Metric Labels

All YugabyteDB metrics use these labels for filtering:
- `namespace` - Kubernetes namespace
- `exported_job` - Original scrape job (yb-master, yb-tserver, ysql)
- `job` - Prometheus scrape job (otel-collector-metrics)
- `pod` - Kubernetes pod name
- `service` - YugabyteDB service name

## Additional Resources

- [YugabyteDB Metrics Reference](https://docs.yugabyte.com/preview/explore/observability/)
- [Prometheus Query Language (PromQL)](https://prometheus.io/docs/prometheus/latest/querying/basics/)
- [Grafana Dashboard Best Practices](https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/best-practices/)
