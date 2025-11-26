# EKS Infrastructure

AWS managed services provisioned via ACK (AWS Controllers for Kubernetes) and GitOps for the [tictactoe-k8s](https://github.com/pnz1990/tictactoe-k8s) reference application.

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         GitHub Repository                                    │
│  eks-infrastructure/                                                        │
│  ├── base/                                                                  │
│  │   ├── prometheus/     (AMP Workspace via ACK)                            │
│  │   ├── monitoring/     (Grafana Alloy for metrics)                        │
│  │   └── logging/        (Fluent Bit for logs)                              │
│  └── overlays/                                                              │
│      ├── dev/                                                               │
│      └── prod/                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              ArgoCD                                          │
│  App: eks-infrastructure                                                    │
│  Auto-sync with self-healing and pruning                                    │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Kubernetes Cluster                                   │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │ namespace: ack-system                                                 │  │
│  │  └── Workspace CR (AMP) ─────────────────────────────────────────┐    │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │ namespace: monitoring                                                 │  │
│  │  ├── Grafana Alloy (metrics) ────────────────────────────────────┼──┐ │  │
│  │  └── Fluent Bit DaemonSet (logs) ────────────────────────────────┼──┼─│  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │ namespace: tictactoe-dev / tictactoe-staging / tictactoe-prod         │  │
│  │  └── App Pods (metrics :9113, logs stdout) ◄─────────────────────┘  │ │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            AWS Cloud                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ Amazon Managed Prometheus (AMP)                                     │    │
│  │  Workspace: ws-62f6ab4b-6a1c-4971-806e-dee13a1e1e95                 │    │
│  │  Receives metrics via remote_write from Grafana Alloy               │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ Amazon CloudWatch Logs                                              │    │
│  │  Log Group: /eks/tictactoe                                          │    │
│  │  Receives logs from Fluent Bit                                      │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ Amazon Managed Grafana (AMG)                                        │    │
│  │  Workspace: g-8f648e108c                                            │    │
│  │  Dashboards for metrics and logs visualization                      │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Components

| Component | Description | Namespace | IAM Role |
|-----------|-------------|-----------|----------|
| AMP Workspace | Amazon Managed Prometheus (via ACK) | ack-system | ACK controller role |
| Grafana Alloy | Metrics collector, scrapes pods → AMP | monitoring | GrafanaAgentAMPRole |
| Fluent Bit | Log collector DaemonSet → CloudWatch | monitoring | FluentBitRole |
| AMG Workspace | Grafana dashboards (created via AWS CLI) | N/A | AmazonGrafanaWorkspaceRole |

## AWS Resources

| Resource | ID/ARN | Region |
|----------|--------|--------|
| AMP Workspace | `ws-62f6ab4b-6a1c-4971-806e-dee13a1e1e95` | ap-northeast-2 |
| AMG Workspace | `g-8f648e108c` | ap-northeast-2 |
| CloudWatch Log Group | `/eks/tictactoe` | ap-northeast-2 |

## IAM Roles (IRSA)

| Role | Service Account | Namespace | Purpose |
|------|-----------------|-----------|---------|
| GrafanaAgentAMPRole | grafana-alloy | monitoring | Write metrics to AMP |
| FluentBitRole | fluent-bit | monitoring | Write logs to CloudWatch |
| AmazonGrafanaWorkspaceRole | N/A | N/A | AMG access to AMP & CloudWatch |

## Metrics Flow

```
┌──────────────────┐     scrape      ┌──────────────────┐    remote_write    ┌─────────┐
│ tictactoe pods   │ ◄────────────── │  Grafana Alloy   │ ─────────────────▶ │   AMP   │
│ (port 9113)      │                 │  (monitoring ns) │                    │         │
└──────────────────┘                 └──────────────────┘                    └─────────┘
                                                                                  │
                                                                                  ▼
                                                                            ┌─────────┐
                                                                            │   AMG   │
                                                                            │ (query) │
                                                                            └─────────┘
```

**Grafana Alloy Configuration:**
- Discovers pods with `prometheus.io/scrape: "true"` annotation
- Scrapes metrics from configured port (default: 9113)
- Adds labels: `namespace`, `pod`, `app`
- Remote writes to AMP with SigV4 authentication

## Logs Flow

```
┌──────────────────┐     tail        ┌──────────────────┐      ship         ┌────────────┐
│ tictactoe pods   │ ◄────────────── │   Fluent Bit     │ ─────────────────▶│ CloudWatch │
│ (stdout/stderr)  │                 │   (DaemonSet)    │                   │    Logs    │
└──────────────────┘                 └──────────────────┘                   └────────────┘
                                                                                  │
                                                                                  ▼
                                                                            ┌─────────┐
                                                                            │   AMG   │
                                                                            │ (query) │
                                                                            └─────────┘
```

**Fluent Bit Configuration:**
- Tails container logs from `/var/log/containers/*tictactoe*.log`
- Parses JSON structured logs
- Ships to CloudWatch Log Group `/eks/tictactoe`
- Log streams organized by namespace

## Grafana Dashboards

Pre-configured dashboards in AMG workspace `g-8f648e108c`:

| Dashboard | UID | Description |
|-----------|-----|-------------|
| Tic Tac Toe - DEV | `ff58o5r40bl6oa` | Dev environment metrics |
| Tic Tac Toe - STAGING | `bf58o6pd8jf9cd` | Staging environment metrics |
| Tic Tac Toe - PROD | `df58o7xz29hq8f` | Production environment metrics |
| Tic Tac Toe - Logs | `af58pcjfql7nkc` | CloudWatch logs viewer |

**Dashboard Panels:**
- Active connections gauge
- HTTP requests/sec rate
- Request latency
- Pod status
- Log stream (for logs dashboard)

## Directory Structure

```
eks-infrastructure/
├── base/
│   ├── prometheus/
│   │   ├── workspace.yaml        # AMP Workspace (ACK)
│   │   └── kustomization.yaml
│   ├── monitoring/
│   │   ├── grafana-alloy.yaml    # Alloy ConfigMap + Deployment
│   │   ├── fluent-bit.yaml       # Fluent Bit DaemonSet
│   │   └── kustomization.yaml
│   └── kustomization.yaml
├── overlays/
│   ├── dev/
│   │   └── kustomization.yaml
│   └── prod/
│       └── kustomization.yaml
└── README.md
```

## Quick Start

### Prerequisites
- EKS cluster with OIDC provider
- ACK Prometheus controller installed
- ArgoCD installed
- IAM roles created (see IAM Setup below)

### Deploy with ArgoCD

```bash
CLUSTER_SERVER="<your-cluster-arn>"

kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: eks-infrastructure
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/pnz1990/eks-infrastructure.git
    targetRevision: main
    path: overlays/dev
  destination:
    server: ${CLUSTER_SERVER}
    namespace: ack-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF
```

### Verify Deployment

```bash
# Check AMP workspace
kubectl get workspace -n ack-system

# Check Grafana Alloy
kubectl get pods -n monitoring -l app=grafana-alloy

# Check Fluent Bit
kubectl get pods -n monitoring -l app=fluent-bit

# View Alloy logs
kubectl logs -n monitoring -l app=grafana-alloy

# View Fluent Bit logs
kubectl logs -n monitoring -l app=fluent-bit
```

## IAM Setup

### 1. Create OIDC Provider (if not exists)

```bash
eksctl utils associate-iam-oidc-provider --cluster <cluster-name> --approve
```

### 2. GrafanaAgentAMPRole (for Grafana Alloy)

```bash
CLUSTER_NAME="<your-cluster>"
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
OIDC_PROVIDER=$(aws eks describe-cluster --name $CLUSTER_NAME --query "cluster.identity.oidc.issuer" --output text | sed 's|https://||')

# Create trust policy
cat > trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Federated": "arn:aws:iam::${ACCOUNT_ID}:oidc-provider/${OIDC_PROVIDER}"},
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "${OIDC_PROVIDER}:sub": "system:serviceaccount:monitoring:grafana-alloy",
        "${OIDC_PROVIDER}:aud": "sts.amazonaws.com"
      }
    }
  }]
}
EOF

# Create role
aws iam create-role --role-name GrafanaAgentAMPRole --assume-role-policy-document file://trust-policy.json

# Attach AMP write policy
aws iam attach-role-policy --role-name GrafanaAgentAMPRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonPrometheusRemoteWriteAccess
```

### 3. FluentBitRole (for Fluent Bit)

```bash
# Update trust policy for fluent-bit service account
cat > trust-policy-fb.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Federated": "arn:aws:iam::${ACCOUNT_ID}:oidc-provider/${OIDC_PROVIDER}"},
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "${OIDC_PROVIDER}:sub": "system:serviceaccount:monitoring:fluent-bit",
        "${OIDC_PROVIDER}:aud": "sts.amazonaws.com"
      }
    }
  }]
}
EOF

# Create role
aws iam create-role --role-name FluentBitRole --assume-role-policy-document file://trust-policy-fb.json

# Create and attach CloudWatch policy
cat > cloudwatch-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "logs:CreateLogGroup",
      "logs:CreateLogStream",
      "logs:PutLogEvents",
      "logs:DescribeLogGroups",
      "logs:DescribeLogStreams"
    ],
    "Resource": "*"
  }]
}
EOF

aws iam put-role-policy --role-name FluentBitRole --policy-name CloudWatchLogs --policy-document file://cloudwatch-policy.json
```

### 4. AmazonGrafanaWorkspaceRole (for AMG)

```bash
# Create role for AMG to access AMP and CloudWatch
aws iam create-role --role-name AmazonGrafanaWorkspaceRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "grafana.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

# Attach policies
aws iam attach-role-policy --role-name AmazonGrafanaWorkspaceRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonPrometheusQueryAccess
aws iam attach-role-policy --role-name AmazonGrafanaWorkspaceRole \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchLogsReadOnlyAccess
```

## AMG Workspace Setup

AMG workspace is created via AWS CLI (ACK controller not available):

```bash
# Create workspace
aws grafana create-workspace \
  --account-access-type CURRENT_ACCOUNT \
  --authentication-providers AWS_SSO \
  --permission-type SERVICE_MANAGED \
  --workspace-name tictactoe-grafana \
  --workspace-role-arn arn:aws:iam::${ACCOUNT_ID}:role/AmazonGrafanaWorkspaceRole \
  --workspace-data-sources PROMETHEUS CLOUDWATCH

# Add data sources via Grafana API
# 1. AMP data source for metrics
# 2. CloudWatch data source for logs
```

## Troubleshooting

### Grafana Alloy not scraping metrics

```bash
# Check Alloy config
kubectl get configmap -n monitoring grafana-alloy-config -o yaml

# Check Alloy logs
kubectl logs -n monitoring -l app=grafana-alloy

# Verify pod annotations
kubectl get pods -n tictactoe-dev -o yaml | grep -A5 annotations
```

### Fluent Bit not shipping logs

```bash
# Check Fluent Bit logs
kubectl logs -n monitoring -l app=fluent-bit

# Verify IAM role
kubectl describe sa fluent-bit -n monitoring

# Check CloudWatch log group
aws logs describe-log-groups --log-group-name-prefix /eks/tictactoe
```

### AMP workspace not created

```bash
# Check ACK controller logs
kubectl logs -n ack-system -l app.kubernetes.io/name=ack-prometheusservice-controller

# Check workspace status
kubectl describe workspace eks-metrics -n ack-system
```

## Related Repositories

- **[tictactoe-k8s](https://github.com/pnz1990/tictactoe-k8s)** - Reference application with CI/CD, security, and Kubernetes best practices

## License

MIT
