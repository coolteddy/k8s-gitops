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
argocd/install/       # Pinned ArgoCD install manifest
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
