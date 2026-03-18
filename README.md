# gitops-config

GitOps config repo — the single source of truth for all Kubernetes workloads.
ArgoCD watches this repo and reconciles the cluster to match what's committed here.
**Never run `kubectl apply` manually in any environment.**

## Repo structure

```
gitops-config/
├── argocd/
│   ├── appsets/
│   │   └── app-appset.yaml     # ApplicationSet — generates one ArgoCD App per env
│   └── apps/
│       └── root-app.yaml       # "App of Apps" — bootstraps ArgoCD itself
├── charts/
│   └── app/                    # Helm chart shared by all environments
│       ├── Chart.yaml
│       ├── values.yaml         # base defaults
│       └── templates/
│           ├── deployment.yaml       # or rollout.yaml in prod
│           ├── service.yaml
│           ├── ingress.yaml
│           ├── hpa.yaml
│           └── analysistemplate.yaml # Prometheus metric gates for canary
└── envs/
    ├── dev/
    │   └── values.yaml         # overrides for dev (auto-sync, debug logging)
    ├── staging/
    │   └── values.yaml         # overrides for staging (manual sync)
    └── prod/
        └── values.yaml         # overrides for prod (canary rollout, HPA, TLS)
```

## How a deploy flows

```
CI pushes sha-abc123 to ECR
    └─► CI opens PR: bump envs/dev/values.yaml image.tag = sha-abc123
            └─► PR merged to main
                    └─► ArgoCD detects diff in config repo (polling every 3 min)
                            └─► Auto-sync fires for dev namespace
                                    └─► ArgoCD applies Helm chart with new tag
                                            └─► Rollout/Deployment updates pods
```

Staging and prod require a manual sync click in the ArgoCD UI (or `argocd app sync`).

## Bootstrapping a fresh cluster

```bash
# 1. Create cluster
kind create cluster --name gitops-demo

# 2. Install ArgoCD
kubectl create namespace argocd
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd -n argocd --wait

# 3. Install Argo Rollouts
kubectl create namespace argo-rollouts
helm install argo-rollouts argo/argo-rollouts -n argo-rollouts --wait

# 4. Apply the root App-of-Apps (this registers the ApplicationSet with ArgoCD)
kubectl apply -f argocd/apps/root-app.yaml

# ArgoCD now manages itself and all environments from Git.
```

## Promoting between environments

Dev auto-promotes on every merge to main (CI bumps the tag).

For staging:
```bash
# Copy the image tag that passed dev
yq e -i '.image.tag = "sha-abc123"' envs/staging/values.yaml
git add envs/staging/values.yaml
git commit -m "chore(staging): promote sha-abc123"
git push
# Then manually sync in ArgoCD UI or:
argocd app sync gitops-app-staging
```

For prod: same flow, then ArgoCD Rollouts handles the canary progression automatically.
