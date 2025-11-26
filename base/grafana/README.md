# Amazon Managed Grafana (AMG)

> **Note**: ACK controller for AMG is not yet available. This workspace is created via AWS CLI.

## Workspace Details

| Property | Value |
|----------|-------|
| Workspace ID | `g-8f648e108c` |
| Endpoint | https://g-8f648e108c.grafana-workspace.ap-northeast-2.amazonaws.com |
| Region | ap-northeast-2 |
| Authentication | AWS IAM Identity Center (SSO) |
| Data Sources | Amazon Managed Prometheus |
| Status | ✅ ACTIVE |

## Data Sources

| Name | Type | Endpoint |
|------|------|----------|
| Amazon Managed Prometheus | prometheus | `https://aps-workspaces.ap-northeast-2.amazonaws.com/workspaces/ws-62f6ab4b-6a1c-4971-806e-dee13a1e1e95` |

## Dashboards

| Dashboard | Description |
|-----------|-------------|
| Tic Tac Toe Application | Nginx metrics: connections, requests, uptime |

## Access

### Via AWS IAM Identity Center (SSO)
1. Configure AWS IAM Identity Center
2. Assign users/groups to the Grafana workspace:
   ```bash
   aws grafana update-permissions \
     --workspace-id g-8f648e108c \
     --update-instruction-batch '[{
       "action": "ADD",
       "role": "ADMIN",
       "users": [{"id": "<SSO_USER_ID>", "type": "SSO_USER"}]
     }]' \
     --region ap-northeast-2
   ```
3. Access: https://g-8f648e108c.grafana-workspace.ap-northeast-2.amazonaws.com

## IAM Role

The workspace uses `AmazonGrafanaWorkspaceRole` with permissions to:
- Query AMP workspaces
- Read metrics, labels, and series

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Amazon Managed Grafana                        │
│  Workspace: eks-grafana-dev                                     │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Data Source: Amazon Managed Prometheus                   │    │
│  │ URL: aps-workspaces.ap-northeast-2.amazonaws.com/...    │    │
│  │ Auth: SigV4 (IAM Role)                                  │    │
│  └─────────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Dashboard: Tic Tac Toe Application                       │    │
│  │ - Active Connections                                     │    │
│  │ - Total HTTP Requests                                    │    │
│  │ - Nginx Up/Down Status                                   │    │
│  │ - Connections Over Time                                  │    │
│  │ - HTTP Requests Rate                                     │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                 Amazon Managed Prometheus                        │
│  Workspace: ws-62f6ab4b-6a1c-4971-806e-dee13a1e1e95            │
│  Metrics from: tictactoe-dev, tictactoe-staging, tictactoe-prod │
└─────────────────────────────────────────────────────────────────┘
```

## CLI Commands Reference

### Create Workspace
```bash
aws grafana create-workspace \
  --account-access-type CURRENT_ACCOUNT \
  --authentication-providers AWS_SSO \
  --permission-type SERVICE_MANAGED \
  --workspace-name eks-grafana-dev \
  --workspace-data-sources PROMETHEUS \
  --workspace-role-arn "arn:aws:iam::569190534191:role/AmazonGrafanaWorkspaceRole" \
  --region ap-northeast-2
```

### Create API Key
```bash
aws grafana create-workspace-api-key \
  --workspace-id g-8f648e108c \
  --key-name "setup-key" \
  --key-role ADMIN \
  --seconds-to-live 3600 \
  --region ap-northeast-2
```

### Check Status
```bash
aws grafana describe-workspace \
  --workspace-id g-8f648e108c \
  --region ap-northeast-2
```
