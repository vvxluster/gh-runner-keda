# Setting Up KEDA for GitHub Self-Hosted Runners

This guide walks you through setting up KEDA (Kubernetes Event Driven Autoscaling) to automatically scale GitHub self-hosted runners based on workflow queue length.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Step 1: Install KEDA](#step-1-install-keda)
- [Step 2: Create GitHub Authentication](#step-2-create-github-authentication)
- [Step 3: Install Your GitHub Runner Chart](#step-3-install-your-github-runner-chart)
- [Step 4: Verify the Setup](#step-4-verify-the-setup)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

Before you begin, ensure you have:

- ✅ A Kubernetes cluster (v1.19+)
- ✅ `kubectl` configured to access your cluster
- ✅ Helm 3.8+ installed
- ✅ A GitHub organization or repository
- ✅ GitHub Personal Access Token (PAT) or GitHub App credentials

### Check Your Environment

```bash
# Verify kubectl access
kubectl cluster-info

# Check Helm version
helm version

# Verify you can create resources
kubectl auth can-i create deployments
```

---

## Step 1: Install KEDA

KEDA needs to be installed **once per cluster** and manages autoscaling for all your applications.

### Install KEDA Using Helm

```bash
# Add the KEDA Helm repository
helm repo add kedacore https://kedacore.github.io/charts

# Update your Helm repositories
helm repo update

# Create a namespace for KEDA
kubectl create namespace keda

# Install KEDA
helm install keda kedacore/keda \
  --namespace keda \
  --version 2.18.3

# Verify installation
kubectl get pods -n keda
```

**Expected output:**
```
NAME                                      READY   STATUS    RESTARTS   AGE
keda-admission-webhooks-xxxxxxxxxx-xxxxx  1/1     Running   0          1m
keda-operator-xxxxxxxxxx-xxxxx            1/1     Running   0          1m
keda-operator-metrics-apiserver-xxxxx     1/1     Running   0          1m
```

### Verify KEDA CRDs are Installed

```bash
# Check for KEDA Custom Resource Definitions
kubectl get crds | grep keda

# You should see:
# cloudeventsources.eventing.keda.sh
# scaledobjects.keda.sh
# scaledjobs.keda.sh
# triggerauthentications.keda.sh
```

---

## Step 2: Create GitHub Authentication

KEDA needs credentials to query GitHub's API for pending workflow runs.

### Option A: Using Personal Access Token (PAT) - Simpler

#### Create a GitHub PAT

1. Go to: https://github.com/settings/tokens
2. Click **"Generate new token (classic)"**
3. Give it a name: `KEDA GitHub Runner Scaler`
4. Set expiration (recommended: 90 days or 1 year)
5. Select scopes:
   - ✅ `repo` (Full control of private repositories)
   - ✅ `admin:org` → `read:org` (Read org and team membership)
6. Click **"Generate token"**
7. **Copy the token** (starts with `ghp_...`)

#### Create Kubernetes Secret

```bash
# Create namespace for your runners
kubectl create namespace gh-runner

# Create secret with your PAT
kubectl create secret generic github-auth \
  --from-literal=personalAccessToken=ghp_YOUR_TOKEN_HERE \
  --namespace gh-runner
```

#### Create TriggerAuthentication

Create a file `github-trigger-auth.yaml`:

```yaml
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: github-auth-trigger
  namespace: gh-runner
spec:
  secretTargetRef:
    - parameter: personalAccessToken
      name: github-auth
      key: personalAccessToken
```

Apply it:

```bash
kubectl apply -f github-trigger-auth.yaml
```

### Option B: Using GitHub App - More Secure (Recommended for Production)

#### Create a GitHub App

1. Go to your organization settings: `https://github.com/organizations/YOUR_ORG/settings/apps`
2. Click **"New GitHub App"**
3. Fill in:
   - **Name**: `KEDA Runner Scaler`
   - **Homepage URL**: `https://github.com/YOUR_ORG`
   - **Webhook**: Uncheck "Active"
4. Set **Repository permissions**:
   - Actions: Read-only
   - Administration: Read-only
5. Set **Organization permissions**:
   - Self-hosted runners: Read-only
6. Click **"Create GitHub App"**
7. Note down:
   - **App ID**
   - **Installation ID** (install the app to your org first)
8. Generate and download **Private Key**

#### Create Kubernetes Secret

```bash
# Create secret with GitHub App credentials
kubectl create secret generic github-app-auth \
  --from-literal=appID=YOUR_APP_ID \
  --from-literal=installationID=YOUR_INSTALLATION_ID \
  --from-file=privateKey=path/to/your-app-private-key.pem \
  --namespace gh-runner
```

#### Create TriggerAuthentication

Create a file `github-app-trigger-auth.yaml`:

```yaml
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: github-auth-trigger
  namespace: gh-runner
spec:
  secretTargetRef:
    - parameter: appKey
      name: github-app-auth
      key: privateKey
    - parameter: appID
      name: github-app-auth
      key: appID
    - parameter: installationID
      name: github-app-auth
      key: installationID
```

Apply it:

```bash
kubectl apply -f github-app-trigger-auth.yaml
```

### Verify Authentication

```bash
# Check the secret
kubectl get secret github-auth -n gh-runner

# Check the TriggerAuthentication
kubectl get triggerauthentication -n gh-runner
```

---

## Step 3: Install Your GitHub Runner Chart

Now that KEDA and authentication are set up, install your runner chart.

### From GitHub Container Registry (GHCR)

```bash
# If the package is private, login first
echo $GITHUB_TOKEN | helm registry login ghcr.io -u YOUR_USERNAME --password-stdin

# Install the chart
helm install gh-runner oci://ghcr.io/vvxluster/gh-runner-chart \
  --version 0.1.2 \
  --namespace gh-runner \
  --create-namespace
```

### From Local Chart

```bash
# Navigate to your chart directory
cd /path/to/gh-runner-chart

# Install
helm install gh-runner . \
  --namespace gh-runner \
  --create-namespace
```

### With Custom Values

Create a `custom-values.yaml`:

```yaml
github:
  owner: "vvxluster"
  runnerScope: "org"
  labels: "gh-keda-runner"
  targetWorkflowQueueLength: "1"

autoscaling:
  minReplicas: 0
  maxReplicas: 10
  pollingInterval: 30
  cooldownPeriod: 300

image:
  repository: ghcr.io/actions/actions-runner
  tag: latest
  pullPolicy: IfNotPresent
```

Install with custom values:

```bash
helm install gh-runner oci://ghcr.io/vvxluster/gh-runner-chart \
  --version 0.1.2 \
  --namespace gh-runner \
  --values custom-values.yaml
```

---

## Step 4: Verify the Setup

### Check KEDA Components

```bash
# Check KEDA pods
kubectl get pods -n keda

# Check ScaledObject
kubectl get scaledobject -n gh-runner

# Describe ScaledObject for details
kubectl describe scaledobject github-runner-scaler -n gh-runner
```

### Check Runner Deployment

```bash
# Check deployment
kubectl get deployment -n gh-runner

# Check pods (should be 0 initially if minReplicas=0)
kubectl get pods -n gh-runner

# Watch pods scale
kubectl get pods -n gh-runner -w
```

### Check KEDA Metrics

```bash
# View HPA created by KEDA
kubectl get hpa -n gh-runner

# Check KEDA operator logs
kubectl logs -n keda -l app.kubernetes.io/name=keda-operator -f

# Check metrics server
kubectl get --raw /apis/external.metrics.k8s.io/v1beta1 | jq .
```

### Test Scaling

1. **Trigger a GitHub workflow** that uses your self-hosted runner label (`gh-keda-runner`)
2. Watch the pods scale up:

```bash
kubectl get pods -n gh-runner -w
```

3. After the workflow completes, watch pods scale down (after cooldown period)

---

## Troubleshooting

### KEDA Not Scaling

**Check ScaledObject status:**

```bash
kubectl describe scaledobject github-runner-scaler -n gh-runner
```

Look for events and conditions at the bottom.

**Check KEDA operator logs:**

```bash
kubectl logs -n keda -l app.kubernetes.io/name=keda-operator --tail=100 -f
```

### Authentication Issues

**Verify secret exists:**

```bash
kubectl get secret github-auth -n gh-runner -o yaml
```

**Check TriggerAuthentication:**

```bash
kubectl describe triggerauthentication github-auth-trigger -n gh-runner
```

**Test GitHub API access manually:**

```bash
# Using PAT
curl -H "Authorization: token YOUR_PAT" \
  https://api.github.com/orgs/vvxluster/actions/runners

# Should return list of runners
```

### Pods Not Starting

**Check deployment:**

```bash
kubectl describe deployment github-runner -n gh-runner
```

**Check pod logs:**

```bash
kubectl logs -n gh-runner -l app=github-runner
```

**Common issues:**
- Image pull errors (check image repository and credentials)
- Missing GitHub runner token
- Insufficient permissions

### ScaledObject Errors

**Error: `targetWorkflowQueueLength` must be string**

Your template needs to use the `quote` filter:

```yaml
targetWorkflowQueueLength: {{ .Values.github.targetWorkflowQueueLength | quote }}
```

**Error: CRD already exists**

KEDA is already installed. Remove KEDA from chart dependencies.

### View All KEDA Resources

```bash
# Get all KEDA-related resources
kubectl api-resources | grep keda

# Get all ScaledObjects in all namespaces
kubectl get scaledobject -A

# Get all TriggerAuthentications
kubectl get triggerauthentication -A
```

---

## Upgrading Your Chart

### Update the Chart

```bash
# Pull latest version
helm repo update

# Upgrade
helm upgrade gh-runner oci://ghcr.io/vvxluster/gh-runner-chart \
  --version 0.2.0 \
  --namespace gh-runner \
  --reuse-values
```

### Upgrade KEDA

```bash
helm upgrade keda kedacore/keda \
  --namespace keda \
  --version 2.18.3
```

---

## Uninstalling

### Remove GitHub Runner Chart

```bash
helm uninstall gh-runner --namespace gh-runner
```

### Remove KEDA (Optional - affects all autoscaling in cluster)

```bash
helm uninstall keda --namespace keda
kubectl delete namespace keda
```

---

## Additional Resources

- **KEDA Documentation**: https://keda.sh/docs/
- **KEDA GitHub Runner Scaler**: https://keda.sh/docs/latest/scalers/github-runner/
- **GitHub Actions Documentation**: https://docs.github.com/en/actions
- **Your Chart Repository**: https://github.com/vvxluster/gh-runner-keda

---

## Quick Reference Commands

```bash
# View runner pods
kubectl get pods -n gh-runner -w

# View ScaledObject
kubectl get scaledobject -n gh-runner

# View KEDA logs
kubectl logs -n keda -l app.kubernetes.io/name=keda-operator -f

# View runner logs
kubectl logs -n gh-runner -l app=github-runner -f

# Scale manually (for testing)
kubectl scale deployment github-runner --replicas=3 -n gh-runner

# Delete and reinstall
helm uninstall gh-runner -n gh-runner
helm install gh-runner oci://ghcr.io/vvxluster/gh-runner-chart --version 0.1.2 -n gh-runner
```
