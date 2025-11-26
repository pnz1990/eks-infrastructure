# Grafana Dashboards

Dashboard JSON files for Amazon Managed Grafana workspace `g-8f648e108c`.

## Dashboards

| Dashboard | File | Description |
|-----------|------|-------------|
| EKS Infrastructure Monitoring | `infrastructure-monitoring.json` | Monitors Grafana Alloy, Fluent Bit, and Grafana Operator |

## Import to Managed Grafana

### Via AWS CLI

```bash
WORKSPACE_ID="g-8f648e108c"
REGION="ap-northeast-2"

# Get Grafana endpoint
GRAFANA_URL=$(aws grafana describe-workspace --workspace-id $WORKSPACE_ID --region $REGION --query 'workspace.endpoint' --output text)

# Import dashboard (requires Grafana API key)
curl -X POST "https://${GRAFANA_URL}/api/dashboards/db" \
  -H "Authorization: Bearer <API_KEY>" \
  -H "Content-Type: application/json" \
  -d @infrastructure-monitoring.json
```

### Via Grafana UI

1. Open AMG workspace: https://g-8f648e108c.grafana-workspace.ap-northeast-2.amazonaws.com
2. Navigate to Dashboards → Import
3. Upload `infrastructure-monitoring.json`
4. Select Prometheus datasource (AMP)
5. Click Import

## Dashboard Panels

### Grafana Alloy
- CPU Usage: `rate(process_cpu_seconds_total[5m])`
- Memory Usage: `process_resident_memory_bytes`

### Fluent Bit
- Records Processed: `rate(fluentbit_input_records_total[5m])`
- Bytes Processed: `rate(fluentbit_input_bytes_total[5m])`

### Grafana Operator
- Reconciliation Rate: `rate(controller_runtime_reconcile_total[5m])`
- Memory Usage: `process_resident_memory_bytes`

### Infrastructure Pods
- Table showing all pods in monitoring and grafana-operator namespaces
