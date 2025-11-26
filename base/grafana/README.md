# Amazon Managed Grafana (AMG)

> **Note**: ACK controller for AMG is not yet available. This workspace is created via AWS CLI.

## Workspace Details

| Property | Value |
|----------|-------|
| Workspace ID | `g-8f648e108c` |
| Endpoint | `https://g-8f648e108c.grafana-workspace.ap-northeast-2.amazonaws.com` |
| Region | ap-northeast-2 |
| Authentication | AWS IAM Identity Center (SSO) |
| Data Sources | Amazon Managed Prometheus |

## IAM Role

The workspace uses `AmazonGrafanaWorkspaceRole` with permissions to:
- Query AMP workspaces
- Read metrics, labels, and series

## Setup Commands

### Create Workspace (already done)
```bash
aws grafana create-workspace \
  --account-access-type CURRENT_ACCOUNT \
  --authentication-providers AWS_SSO \
  --permission-type SERVICE_MANAGED \
  --workspace-name eks-grafana-dev \
  --workspace-description "Grafana for EKS monitoring" \
  --workspace-data-sources PROMETHEUS \
  --workspace-role-arn "arn:aws:iam::569190534191:role/AmazonGrafanaWorkspaceRole" \
  --region ap-northeast-2
```

### Add AMP Data Source
```bash
# Get AMP workspace endpoint
AMP_WORKSPACE_ID="ws-62f6ab4b-6a1c-4971-806e-dee13a1e1e95"
AMP_ENDPOINT="https://aps-workspaces.ap-northeast-2.amazonaws.com/workspaces/${AMP_WORKSPACE_ID}"

# Create data source via Grafana API (requires API key)
curl -X POST https://g-8f648e108c.grafana-workspace.ap-northeast-2.amazonaws.com/api/datasources \
  -H "Authorization: Bearer <API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Amazon Managed Prometheus",
    "type": "prometheus",
    "url": "'${AMP_ENDPOINT}'",
    "access": "proxy",
    "jsonData": {
      "httpMethod": "POST",
      "sigV4Auth": true,
      "sigV4AuthType": "default",
      "sigV4Region": "ap-northeast-2"
    }
  }'
```

## Access

1. Configure AWS IAM Identity Center (SSO)
2. Assign users/groups to the Grafana workspace
3. Access via: https://g-8f648e108c.grafana-workspace.ap-northeast-2.amazonaws.com
