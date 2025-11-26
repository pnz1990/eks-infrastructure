# Grafana Dashboards

Dashboard JSON files for Amazon Managed Grafana workspace `g-8f648e108c`.

## Dashboards

| Dashboard | File | Description |
|-----------|------|-------------|
| EKS Infrastructure Monitoring | `infrastructure-monitoring.json` | Monitors Grafana Alloy, Fluent Bit, and Grafana Operator |

## Import to Managed Grafana

**Note:** The Grafana Operator cannot automatically sync to Amazon Managed Grafana (AMG) because AMG uses AWS SSO authentication, which the operator doesn't support. Dashboards must be imported manually.

### Via Grafana UI (Recommended)

1. Open AMG workspace: https://g-8f648e108c.grafana-workspace.ap-northeast-2.amazonaws.com
2. Navigate to **Dashboards** → **New** → **Import**
3. Click **Upload JSON file**
4. Select `infrastructure-monitoring.json`
5. Select **Prometheus** datasource (your AMP datasource)
6. Click **Import**

### Via Grafana API

```bash
WORKSPACE_ID="g-8f648e108c"
REGION="ap-northeast-2"

# Get Grafana endpoint
GRAFANA_URL="https://g-8f648e108c.grafana-workspace.ap-northeast-2.amazonaws.com"

# Create API key in Grafana UI first (Configuration → API Keys)
API_KEY="<your-api-key>"

# Import dashboard
curl -X POST "${GRAFANA_URL}/api/dashboards/db" \
  -H "Authorization: Bearer ${API_KEY}" \
  -H "Content-Type: application/json" \
  -d @infrastructure-monitoring.json
```

## Dashboard Panels

### Grafana Alloy
- **CPU Usage**: `rate(process_cpu_seconds_total{namespace="monitoring",pod=~"grafana-alloy.*"}[5m])`
- **Memory Usage**: `process_resident_memory_bytes{namespace="monitoring",pod=~"grafana-alloy.*"}`

### Fluent Bit
- **Records Processed**: `rate(fluentbit_input_records_total{namespace="monitoring"}[5m])`
- **Bytes Processed**: `rate(fluentbit_input_bytes_total{namespace="monitoring"}[5m])`

### Grafana Operator
- **Reconciliation Rate**: `rate(controller_runtime_reconcile_total{namespace="grafana-operator"}[5m])`
- **Memory Usage**: `process_resident_memory_bytes{namespace="grafana-operator"}`

### Infrastructure Pods
- Table showing all pods in monitoring and grafana-operator namespaces

## GitOps Deployment

The dashboard is also deployed as a `GrafanaDashboard` CR via GitOps in `base/dashboards/infrastructure-dashboard.yaml`. While it won't auto-sync to AMG, it serves as:
- Source of truth for the dashboard configuration
- Version control for dashboard changes
- Easy export to JSON for manual import

To export the latest dashboard JSON from the CR:
```bash
kubectl get grafanadashboard infrastructure-monitoring -n ack-system -o jsonpath='{.spec.json}' > infrastructure-monitoring.json
```
