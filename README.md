# k8s-gitops

GitOps platform built on ArgoCD. All cluster state is managed from this repository.

## Stack

| Component | Purpose | Notes |
|-----------|---------|-------|
| ArgoCD | GitOps continuous delivery | [notes/argocd.md](notes/argocd.md) |
| Prometheus + Grafana | Metrics and dashboards | [notes/prometheus.md](notes/prometheus.md) |
| Loki | Log aggregation | [notes/loki.md](notes/loki.md) |
| Fluent Bit | Log collection | [notes/fluent-bit.md](notes/fluent-bit.md) |
| Argo Rollouts | Progressive delivery — blue/green, canary | [notes/rollouts.md](notes/rollouts.md) (pending) |
| Istio | Service mesh — mTLS, traffic management | [notes/istio.md](notes/istio.md) (pending) |
| Cilium | eBPF-based CNI (kind cluster) | [notes/cilium.md](notes/cilium.md) (pending) |

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
notes/                # Per-tool setup, internals, and troubleshooting deep dives
```

## GitOps Workflow

```
git commit + push  →  ArgoCD detects change  →  syncs to cluster
```

ArgoCD polls this repo every 3 minutes. Manual sync available via UI or CLI.

**Critical rule:** push to Git before applying or syncing an ArgoCD `Application` — ArgoCD reads from Git, not the local filesystem.

## Setup

ArgoCD is not managed by ArgoCD itself — it must be installed first, before GitOps can take over everything else. Full install, verify, and internals for every tool live in its `notes/` file linked in the Stack table above. Start with [notes/argocd.md](notes/argocd.md).

Access reference:

| Service | Access | URL |
|---------|--------|-----|
| ArgoCD UI | NodePort | `http://<node-ip>:<nodeport>` |
| Grafana | NodePort 31689 | `http://10.124.224.145:31689` |
| Grafana (remote via SSH tunnel) | `ssh -L 3000:10.124.224.145:31689 dell` | `http://localhost:3000` |
