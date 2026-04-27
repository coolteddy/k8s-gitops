# k8s-gitops

GitOps platform built on ArgoCD. All cluster state is managed from this repository.

## Stack

| Component | Purpose |
|-----------|---------|
| ArgoCD | GitOps continuous delivery |
| Argo Rollouts | Progressive delivery — blue/green, canary |
| Prometheus + Grafana | Metrics and dashboards |
| Fluent Bit + Loki | Log collection and aggregation |
| Istio | Service mesh — mTLS, traffic management |
| Cilium | eBPF-based CNI (kind cluster) |

## Cluster

- **Primary:** k8s-master2 (8GB/4CPU) + k8s-worker2 (6GB/4CPU), Kubernetes v1.34.7, Calico CNI
- **Cilium:** kind (local), separate cluster

## Repository Structure

```
argocd/apps/          # ArgoCD Application CRDs (app-of-apps)
apps/sample-app/      # Reference app used across all phases
rollouts/             # Argo Rollouts — blue/green and canary configs
monitoring/           # Prometheus + Grafana
logging/              # Fluent Bit + Loki
istio/                # Istio control plane + traffic policies
cilium/               # Cilium on kind
cluster-setup/        # Cluster bootstrap configs
```

## GitOps Workflow

```
git commit + push  →  ArgoCD detects change  →  syncs to cluster
```

ArgoCD polls this repo every 3 minutes. Manual sync available via UI or CLI.

## Bootstrap

ArgoCD is not managed by ArgoCD itself — it must be installed first before GitOps takes over.

### ArgoCD

Version: v3.3.8

```bash
kubectl create namespace argocd
kubectl apply --server-side --force-conflicts -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.3.8/manifests/install.yaml
```

Expose the UI:
```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
```

Initial admin password:
```bash
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d && echo
```
