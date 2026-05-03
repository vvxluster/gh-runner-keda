# GitHub Runner Helm Chart Pipeline Flow

This document explains the complete flow of how your Helm chart gets packaged and published to GitHub Container Registry (GHCR), and how it enables KEDA-based autoscaling for GitHub self-hosted runners.

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Pipeline Flow](#pipeline-flow)
- [How KEDA Autoscaling Works](#how-keda-autoscaling-works)
- [End-to-End Workflow](#end-to-end-workflow)
- [Component Interactions](#component-interactions)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        GitHub Actions Pipeline                       │
│  ┌────────────┐    ┌────────────┐    ┌────────────┐    ┌─────────┐ │
│  │  Checkout  │ -> │    Lint    │ -> │  Package   │ -> │  Push   │ │
│  │    Code    │    │    Chart   │    │   Chart    │    │  GHCR   │ │
│  └────────────┘    └────────────┘    └────────────┘    └─────────┘ │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
                    ┌─────────────────────────────┐
                    │   GitHub Container Registry  │
                    │  (ghcr.io/vvxluster/        │
                    │   gh-runner-chart)           │
                    └─────────────────────────────┘
                                  │
                                  ▼
                    ┌─────────────────────────────┐
                    │    Kubernetes Cluster        │
                    │                              │
                    │  ┌──────────────────────┐   │
                    │  │   KEDA Operator      │   │
                    │  └──────────────────────┘   │
                    │           │                  │
                    │           ▼                  │
                    │  ┌──────────────────────┐   │
                    │  │  ScaledObject        │   │
                    │  │  (Monitors GitHub)   │   │
                    │  └──────────────────────┘   │
                    │           │                  │
                    │           ▼                  │
                    │  ┌──────────────────────┐   │
                    │  │  Runner Deployment   │   │
                    │  │  (Scales 0-N)        │   │
                    │  └──────────────────────┘   │
                    └─────────────────────────────┘
                                  │
                                  ▼
                    ┌─────────────────────────────┐
                    │      GitHub Actions          │
                    │   (Workflow Execution)       │
                    └─────────────────────────────┘
```

---

## Pipeline Flow

### 1. Developer Makes Changes

```
Developer updates chart files
   │
   ├── Chart.yaml (version, metadata)
   ├── values.yaml (configuration)
   ├── templates/ (Kubernetes manifests)
   │   ├── deployment.yaml
   │   ├── scaledobject.yaml
   │   └── _helpers.tpl
   │
   └── Commits and pushes to main branch
```

### 2. GitHub Actions Workflow Triggers

**Trigger conditions:**
```yaml
on:
  push:
    branches:
      - main
    paths:
      - "gh-runner-chart/**"  # Only when chart files change
```

**The workflow won't run if:**
- Changes are only in README or other files
- Push is to a different branch
- It's a pull request (only validates, doesn't publish)

### 3. Pipeline Steps Execute

#### Step 1: Checkout Code
```bash
actions/checkout@v4
```
- Clones the repository
- Makes chart files available to the runner

#### Step 2: Set up Helm
```bash
azure/setup-helm@v4
version: v3.16.3
```
- Installs Helm CLI on the runner
- Ensures consistent Helm version

#### Step 3: Lint the Chart
```bash
helm lint gh-runner-chart
```
- Validates Chart.yaml structure
- Checks template syntax
- Verifies best practices
- **Fails the pipeline if errors found**

Example output:
```
==> Linting gh-runner-chart
[INFO] Chart.yaml: icon is recommended
1 chart(s) linted, 0 chart(s) failed
```

#### Step 4: Package the Chart
```bash
helm package gh-runner-chart
```
- Creates a versioned `.tgz` file
- Filename: `gh-runner-chart-0.1.2.tgz`
- Contains all chart files in compressed format

Example output:
```
Successfully packaged chart and saved it to: /path/gh-runner-chart-0.1.2.tgz
```

#### Step 5: Login to GHCR
```bash
echo "${{ secrets.GHCR_TOKEN }}" | \
  helm registry login ghcr.io -u ${{ github.actor }} --password-stdin
```
- Authenticates with GitHub Container Registry
- Uses Personal Access Token (PAT) from repository secrets
- Required for pushing packages

#### Step 6: Push to GHCR
```bash
CHART_VERSION=$(helm show chart gh-runner-chart | grep '^version:' | awk '{print $2}')
CHART_NAME=$(helm show chart gh-runner-chart | grep '^name:' | awk '{print $2}')
helm push ${CHART_NAME}-${CHART_VERSION}.tgz oci://ghcr.io/vvxluster
```
- Extracts version from Chart.yaml
- Pushes packaged chart to OCI registry
- Creates immutable artifact at: `oci://ghcr.io/vvxluster/gh-runner-chart:0.1.2`

Example output:
```
Pushed: ghcr.io/vvxluster/gh-runner-chart:0.1.2
Digest: sha256:1234567890abcdef...
```

---

## How KEDA Autoscaling Works

### Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                    GitHub Actions Workflow                          │
│  Developer triggers workflow with label: "gh-keda-runner"           │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        GitHub API                                    │
│  Queue: 1 workflow waiting for "gh-keda-runner"                     │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼ (Polls every 30s)
┌─────────────────────────────────────────────────────────────────────┐
│                    KEDA ScaledObject                                 │
│  - Queries GitHub API                                                │
│  - Checks queue length for label "gh-keda-runner"                   │
│  - Compares to targetWorkflowQueueLength: "1"                       │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼ (Queue > 0)
┌─────────────────────────────────────────────────────────────────────┐
│                  KEDA Creates HPA (Horizontal Pod Autoscaler)       │
│  Desired replicas = queue length / targetWorkflowQueueLength        │
│  Example: 1 workflow / 1 target = 1 replica                         │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│              Kubernetes Deployment Scales Up                         │
│  - Creates new pod(s)                                                │
│  - Pod registers with GitHub as self-hosted runner                  │
│  - Runner picks up queued workflow                                  │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   Workflow Executes                                  │
│  - Runner executes job steps                                         │
│  - Workflow completes                                                │
│  - Queue becomes empty                                               │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼ (After cooldownPeriod: 300s)
┌─────────────────────────────────────────────────────────────────────┐
│              Kubernetes Deployment Scales Down                       │
│  - KEDA detects queue = 0                                            │
│  - Waits 300 seconds (cooldown)                                      │
│  - Scales deployment to minReplicas: 0                              │
│  - Pods are terminated                                               │
└─────────────────────────────────────────────────────────────────────┘
```

### Key Configuration Parameters

```yaml
# From ScaledObject
spec:
  minReplicaCount: 0           # Scale to zero when idle
  maxReplicaCount: 5           # Maximum concurrent runners
  pollingInterval: 30          # Check GitHub API every 30 seconds
  cooldownPeriod: 300          # Wait 5 minutes before scaling down
  
  triggers:
  - type: github-runner
    metadata:
      owner: "vvxluster"                    # GitHub org/user
      runnerScope: "org"                    # org or repo
      labels: "gh-keda-runner"              # Runner label to monitor
      targetWorkflowQueueLength: "1"        # 1 runner per queued workflow
```

### Scaling Logic

**Scale Up:**
```
desired_replicas = ceiling(queue_length / targetWorkflowQueueLength)

Example:
- 3 workflows queued, target = 1
- desired = 3 / 1 = 3 runners
- But max is 5, so won't exceed that
```

**Scale Down:**
```
- If queue_length = 0
- Wait cooldownPeriod (300s)
- Scale to minReplicaCount (0)
```

---

## End-to-End Workflow

### Complete Flow from Code to Execution

```
┌──────────────────────────────────────────────────────────────────┐
│ 1. Developer Updates Chart                                       │
│    - Increment version in Chart.yaml: 0.1.1 -> 0.1.2            │
│    - Update templates or values                                  │
│    - Commit and push to main                                     │
└──────────────────────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────────────────┐
│ 2. GitHub Actions Pipeline Runs                                  │
│    - Lints chart                                                 │
│    - Packages to .tgz                                            │
│    - Pushes to ghcr.io/vvxluster/gh-runner-chart:0.1.2         │
└──────────────────────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────────────────┐
│ 3. Ops Team Installs/Upgrades Chart                             │
│    helm upgrade gh-runner \                                      │
│      oci://ghcr.io/vvxluster/gh-runner-chart \                  │
│      --version 0.1.2 -n gh-runner                               │
└──────────────────────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────────────────┐
│ 4. Kubernetes Resources Created                                  │
│    - Deployment: github-runner (replicas: 0)                    │
│    - ScaledObject: github-runner-scaler                         │
│    - TriggerAuthentication: github-auth-trigger                 │
└──────────────────────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────────────────┐
│ 5. KEDA Starts Monitoring                                        │
│    - Every 30s: Query GitHub API                                 │
│    - Check queue for "gh-keda-runner" label                     │
│    - Initially: queue = 0, replicas stay at 0                   │
└──────────────────────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────────────────┐
│ 6. Developer Triggers Workflow                                   │
│    .github/workflows/example.yml:                                │
│      runs-on: [self-hosted, gh-keda-runner]                     │
└──────────────────────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────────────────┐
│ 7. KEDA Detects Queue                                            │
│    - Next poll: queue = 1                                        │
│    - Calculates: need 1 runner                                   │
│    - Updates HPA: desiredReplicas = 1                           │
└──────────────────────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────────────────┐
│ 8. Kubernetes Scales Deployment                                  │
│    - Creates 1 pod                                               │
│    - Pod starts runner container                                 │
│    - Runner registers with GitHub                                │
│    - Becomes available in ~30-60 seconds                        │
└──────────────────────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────────────────┐
│ 9. Workflow Executes                                             │
│    - Runner picks up job                                         │
│    - Executes workflow steps                                     │
│    - Job completes                                               │
│    - Queue becomes empty                                         │
└──────────────────────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────────────────┐
│ 10. KEDA Scales Down                                             │
│    - Detects queue = 0                                           │
│    - Waits 300s (cooldownPeriod)                                │
│    - Scales deployment to 0                                      │
│    - Pod terminates, runner de-registers                        │
└──────────────────────────────────────────────────────────────────┘
```

---

## Component Interactions

### GitHub Actions Pipeline Components

```yaml
┌─────────────────────────────────────────────────────────────┐
│                   Workflow File                              │
│  .github/workflows/helm-chart-pipeline.yml                   │
├─────────────────────────────────────────────────────────────┤
│  Triggers:                                                   │
│    - push to main                                            │
│    - paths: gh-runner-chart/**                              │
├─────────────────────────────────────────────────────────────┤
│  Permissions:                                                │
│    - contents: read                                          │
│    - packages: write  ← Critical for GHCR push              │
├─────────────────────────────────────────────────────────────┤
│  Secrets Used:                                               │
│    - GHCR_TOKEN (Personal Access Token)                     │
│    - GITHUB_TOKEN (auto-provided, not used currently)       │
└─────────────────────────────────────────────────────────────┘
```

### Helm Chart Components

```
gh-runner-chart/
├── Chart.yaml                  # Metadata and version
│   ├── apiVersion: v2
│   ├── name: gh-runner-chart
│   ├── version: 0.1.2         ← Increment for each release
│   └── type: application
│
├── values.yaml                 # Default configuration
│   ├── github:
│   │   ├── owner: "vvxluster"
│   │   ├── runnerScope: "org"
│   │   ├── labels: "gh-keda-runner"
│   │   └── targetWorkflowQueueLength: "1"
│   └── autoscaling:
│       ├── minReplicas: 0
│       ├── maxReplicas: 5
│       ├── pollingInterval: 30
│       └── cooldownPeriod: 300
│
└── templates/                  # Kubernetes manifests
    ├── deployment.yaml         # Runner pods
    ├── scaledobject.yaml       # KEDA scaling config
    ├── _helpers.tpl            # Template helpers
    └── NOTES.txt               # Post-install instructions
```

### KEDA Components

```
┌─────────────────────────────────────────────────────────────┐
│                    KEDA Operator                             │
│  (Installed once per cluster in 'keda' namespace)           │
├─────────────────────────────────────────────────────────────┤
│  Responsibilities:                                           │
│    - Watch ScaledObject resources                           │
│    - Poll configured triggers (GitHub API)                   │
│    - Calculate desired replica count                        │
│    - Create/update HPA (Horizontal Pod Autoscaler)         │
│    - Manage scaling lifecycle                               │
└─────────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────────┐
│              ScaledObject (Your Chart Creates)               │
│  (In 'gh-runner' namespace)                                  │
├─────────────────────────────────────────────────────────────┤
│  Links:                                                      │
│    - Deployment: github-runner                              │
│    - Trigger: github-runner (type)                          │
│    - Auth: github-auth-trigger (TriggerAuthentication)      │
├─────────────────────────────────────────────────────────────┤
│  Behavior:                                                   │
│    - Every 30s: GET /orgs/vvxluster/actions/runners         │
│    - Filter by label: "gh-keda-runner"                      │
│    - Count queue length                                      │
│    - Update HPA if queue changes                            │
└─────────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────────┐
│            TriggerAuthentication                             │
│  (Stores GitHub credentials)                                 │
├─────────────────────────────────────────────────────────────┤
│  References Secret:                                          │
│    - name: github-auth                                       │
│    - key: personalAccessToken                               │
├─────────────────────────────────────────────────────────────┤
│  Used By:                                                    │
│    - KEDA to authenticate GitHub API calls                  │
└─────────────────────────────────────────────────────────────┘
```

### Authentication Flow

```
KEDA Operator needs GitHub API access
         ↓
ScaledObject references TriggerAuthentication
         ↓
TriggerAuthentication references Secret
         ↓
Secret contains PAT (Personal Access Token)
         ↓
KEDA uses PAT for GitHub API calls:
  GET https://api.github.com/orgs/vvxluster/actions/runners
  Authorization: Bearer ghp_xxxxxxxxxxxxx
```

---

## Version Management

### Chart Versioning Strategy

```
Chart.yaml version format: MAJOR.MINOR.PATCH

Examples:
  0.1.0  → Initial release
  0.1.1  → Bug fix (template syntax, value defaults)
  0.1.2  → Bug fix (metadata corrections)
  0.2.0  → New feature (add new template, new config option)
  1.0.0  → Production ready, stable API
```

### Publishing New Versions

```bash
# 1. Update Chart.yaml
vim gh-runner-chart/Chart.yaml
# Change: version: 0.1.1 → version: 0.1.2

# 2. Commit and push
git add gh-runner-chart/Chart.yaml
git commit -m "Bump chart version to 0.1.2"
git push origin main

# 3. Pipeline automatically:
#    - Packages gh-runner-chart-0.1.2.tgz
#    - Pushes to oci://ghcr.io/vvxluster/gh-runner-chart:0.1.2

# 4. Users can install specific version:
helm install gh-runner oci://ghcr.io/vvxluster/gh-runner-chart --version 0.1.2
```

---

## Monitoring and Observability

### Key Metrics to Watch

```bash
# 1. Runner Pod Count
kubectl get pods -n gh-runner -w

# 2. ScaledObject Status
kubectl get scaledobject -n gh-runner

# 3. HPA Status (created by KEDA)
kubectl get hpa -n gh-runner

# 4. KEDA Operator Logs
kubectl logs -n keda -l app.kubernetes.io/name=keda-operator -f

# 5. Runner Logs
kubectl logs -n gh-runner -l app=github-runner -f
```

### Debugging Commands

```bash
# View complete ScaledObject configuration
kubectl describe scaledobject github-runner-scaler -n gh-runner

# Check events (scaling activities)
kubectl get events -n gh-runner --sort-by='.lastTimestamp'

# View current metric value
kubectl get hpa -n gh-runner -o jsonpath='{.items[0].status.currentMetrics}'

# Manual scaling test
kubectl scale deployment github-runner --replicas=2 -n gh-runner
```

---

## Summary

This system provides:

✅ **Automated Packaging**: Every chart change is automatically packaged and published  
✅ **Version Control**: Immutable chart versions in GHCR  
✅ **Event-Driven Scaling**: Runners scale based on actual workflow demand  
✅ **Cost Efficiency**: Scale to zero when idle  
✅ **Fast Response**: New runners ready in ~30-60 seconds  
✅ **Enterprise Ready**: GitHub App authentication, namespace isolation  

The complete flow ensures your GitHub runners are always available when needed, but don't waste resources when idle.