##setting UP KEDA 
Before setup KEDA autoscalled POD as the runner in Github we need  to Install the KEDA incluster

## Status
- [x] Namepsace for KEDA
- [x] Installing the KEDA using the documenation via HELM
- [ ] Making connection between github and KEDA  
- [ ] Pipeline Sample Run


## steps to install helm in the cluster
```helm repo add kedacore https://kedacore.github.io/charts  
helm repo update
helm install keda kedacore/keda --namespace keda --create-namespace```

## Setting up Namespace for Runners
```k create ns gh-runner
   k create secret generic github-auth \
  --namespace gh-runner \
  --from-literal=github_token=github_pat_TOKEN
```
## Setting up Actions Runner Controller (ARC):
we dont need the ARC for ADO where as we do need ARC in github because, It correctly registering, unregistering, and cleaning up GitHub runners at scale
A Kubernetes controller that translates GitHub runners into Kubernetes Pods
This includes:
Generating runner registration tokens
Handling ephemeral runners
De-registering runners after job completion
Avoiding “ghost runners” in GitHub UI
Handling crashes & retries safely

ARC needs certmanager as the dependency we will isntall using the helm
Actions Runner Controller depends on cert-manager for webhook TLS certificates. If cert-manager CRDs aren’t installed, Helm fails because Kubernetes doesn’t recognize Certificate and Issuer kinds
ARC uses webhooks so Kubernetes cannot accept invalid Runner objects that would break GitHub runner lifecycle.
ARC needs cert-manager because ARC runs a Kubernetes admission webhook, and admission webhooks must use TLS certificates.
ARC injects this image automatically.
We can  use our own image

cert-manager:
Issues certificates
Manages CA trust
Rotates certs automatically
Injects CA bundle into webhook config

```helm repo add actions-runner-controller \
  https://actions-runner-controller.github.io/actions-runner-controller
  helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install arc actions-runner-controller/actions-runner-controller \
  --namespace gh-runner \
  --set authSecret.create=false \
  --set authSecret.name=github-auth```


ARC installed:
CRDs:
Runner
RunnerDeployment
RunnerSet

A controller that:
Watches these CRDs
Talks to GitHub API
Creates runner pods
kubectl get pods -n gh-runners


Why RunnerDeployment exists (problem it solves)
GitHub runners are:
External to Kubernetes
Stateful (registered in GitHub)
Security-sensitive

A normal Deployment cannot:
Register runners
Deregister runners
Handle runner identity safely
So ARC introduces RunnerDeployment.
