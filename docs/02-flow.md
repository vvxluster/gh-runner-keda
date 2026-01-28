##Components you need
KEDA: Event-based autoscaling
GitHub Actions:	CI jobs
GitHub PAT:	Auth to GitHub API
Kubernetes namespace:	For Runners (Isolation)

## The full flow 
1. Developer pushes code to GitHub
   ↓
2. GitHub workflow triggered with runs-on: [self-hosted, keda-runner]
   ↓
3. GitHub marks job as "queued"
   ↓
4. KEDA polls GitHub API (every 30 seconds) ()
   ↓
5. KEDA detects: 1 queued job with  label that we mentioned the scaled object manifest file
   ↓
6. KEDA calculates: need 1 runner (1 job ÷ targetWorkflowQueueLength)
   ↓
7. KEDA updates HPA: set desired replicas = 1
   ↓
8. Kubernetes creates 1 pod using github-runner deployment manifest file
   ↓
9. Pod starts, container registers with GitHub as runner (taken care by myoung base container)
   ↓
10. GitHub assigns queued job to new runner 
   ↓
11. Runner executes workflow
   ↓
12. Job completes, no more queued jobs
   ↓
13. KEDA detects: 0 queued jobs
   ↓
14. After cooldownPeriod (5 min), KEDA scales to 0
   ↓
15. Pod terminates, runner deregisters from GitHub
