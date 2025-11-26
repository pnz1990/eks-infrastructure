# EKS Infrastructure

AWS managed services provisioned via ACK (AWS Controllers for Kubernetes) using GitOps.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    GitHub Repository                             │
│  eks-infrastructure/                                            │
│  ├── base/                                                      │
│  │   ├── prometheus/    (AMP Workspace)                         │
│  │   └── grafana/       (AMG Workspace) [future]                │
│  └── overlays/                                                  │
│      ├── dev/                                                   │
│      └── prod/                                                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                         ArgoCD                                   │
│  App: eks-infrastructure                                        │
│  Syncs ACK resources to cluster                                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                            │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ ACK Controllers (managed)                                │    │
│  │  - prometheusservice-controller                          │    │
│  │  - grafana-controller [future]                           │    │
│  └─────────────────────────────────────────────────────────┘    │
│                              │                                   │
│                              ▼                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ ACK Custom Resources                                     │    │
│  │  - Workspace (AMP)                                       │    │
│  │  - Workspace (AMG) [future]                              │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      AWS Cloud                                   │
│  ┌──────────────────┐    ┌──────────────────┐                   │
│  │ Amazon Managed   │    │ Amazon Managed   │                   │
│  │ Prometheus (AMP) │    │ Grafana (AMG)    │                   │
│  └──────────────────┘    └──────────────────┘                   │
└─────────────────────────────────────────────────────────────────┘
```

## Services

| Service | ACK Controller | Status |
|---------|---------------|--------|
| Amazon Managed Prometheus | prometheusservice | ✅ Ready |
| Amazon Managed Grafana | grafana | 🔜 Planned |

## Directory Structure

```
eks-infrastructure/
├── base/
│   ├── prometheus/
│   │   ├── workspace.yaml      # AMP Workspace
│   │   └── kustomization.yaml
│   ├── grafana/                # Future
│   └── kustomization.yaml
├── overlays/
│   ├── dev/
│   │   └── kustomization.yaml
│   └── prod/
│       └── kustomization.yaml
└── README.md
```

## Quick Start

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

### Verify AMP Workspace

```bash
# Check ACK resource status
kubectl get workspace -n ack-system

# Get workspace details
kubectl describe workspace eks-metrics -n ack-system
```

## Connecting Prometheus Agent

Once the AMP workspace is created, configure your Prometheus agent to remote write:

```yaml
# Get the workspace ID and endpoint from the ACK resource status
WORKSPACE_ID=$(kubectl get workspace eks-metrics -n ack-system -o jsonpath='{.status.workspaceID}')
REGION=ap-northeast-2

# Remote write URL
# https://aps-workspaces.${REGION}.amazonaws.com/workspaces/${WORKSPACE_ID}/api/v1/remote_write
```

### Using ADOT Collector

```yaml
exporters:
  prometheusremotewrite:
    endpoint: https://aps-workspaces.ap-northeast-2.amazonaws.com/workspaces/${WORKSPACE_ID}/api/v1/remote_write
    auth:
      authenticator: sigv4auth
```

### Using Grafana Agent

```yaml
prometheus:
  configs:
    - name: default
      remote_write:
        - url: https://aps-workspaces.ap-northeast-2.amazonaws.com/workspaces/${WORKSPACE_ID}/api/v1/remote_write
          sigv4:
            region: ap-northeast-2
```

## IAM Requirements

The ACK controller needs IAM permissions to create AMP resources. Ensure your IRSA role has:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "aps:CreateWorkspace",
        "aps:DeleteWorkspace",
        "aps:DescribeWorkspace",
        "aps:UpdateWorkspaceAlias",
        "aps:TagResource",
        "aps:UntagResource",
        "aps:ListTagsForResource"
      ],
      "Resource": "*"
    }
  ]
}
```
