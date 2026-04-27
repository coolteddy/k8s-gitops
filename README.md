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

Connect repo and deploy sample app:
```bash
argocd login <node-ip>:<nodeport> --username admin --password <password> --insecure
argocd repo add https://github.com/coolteddy/k8s-gitops.git
kubectl apply -f argocd/apps/sample-app.yaml
```

### ArgoCD Verify

```bash
# All 7 ArgoCD pods running
kubectl get pods -n argocd

# Sample app deployed by ArgoCD
kubectl get pods -n sample-app

# App sync status
argocd app get sample-app

# GitOps loop test — change replicas in apps/sample-app/deployment.yaml, push, then:
argocd app sync sample-app
kubectl get pods -n sample-app -w

# Revert test
git revert HEAD --no-edit && git push
argocd app sync sample-app
kubectl get pods -n sample-app -w
```

---

### Prometheus + Grafana

Chart: kube-prometheus-stack v70.4.2
Values: `monitoring/prometheus/values.yaml`

Deploy via ArgoCD (push to Git first — ArgoCD reads values.yaml from the repo):
```bash
git add monitoring/prometheus/values.yaml argocd/apps/monitoring.yaml
git commit -m "Add kube-prometheus-stack"
git push
kubectl apply -f argocd/apps/monitoring.yaml
argocd app sync monitoring
```

Get Grafana NodePort:
```bash
kubectl get svc -n monitoring | grep grafana
```

**Local machine** (same network as cluster) — open directly:
```
http://10.124.224.145:31689
```

**Remote machine** (SSH tunnel) — run on remote machine, then open browser:
```bash
ssh -L 3000:10.124.224.145:31689 dell
# Open: http://localhost:3000
```

Login: `admin / gitops1234`

### Prometheus + Grafana Verify

```bash
# All pods running
kubectl get pods -n monitoring

# Grafana and Prometheus services with NodePorts
kubectl get svc -n monitoring

# Check Prometheus targets (port-forward Prometheus UI)
kubectl port-forward svc/monitoring-kube-prometheus-prometheus -n monitoring 9090:9090
# Open: http://localhost:9090/targets — all targets should be UP

# In Grafana:
# Dashboards > Browse > Node Exporter / Nodes — live CPU, RAM, disk per node
# Dashboards > Browse > Kubernetes / Pods — pod-level metrics
# Explore > Prometheus > run query: up
```

---

### Loki + Fluent Bit

- Chart: `grafana/loki` 6.55.0 (Loki 3.6.7) — values: `logging/loki/values.yaml`
- Chart: `fluent/fluent-bit` 0.57.3 — values: `logging/fluent-bit/values.yaml`

Note: `loki-stack` (deprecated) bundles Loki 2.6.1 which is incompatible with Grafana 11.x. Use standalone charts above.

Deploy via ArgoCD (push to Git first):
```bash
git add logging/ argocd/apps/logging.yaml argocd/apps/fluent-bit.yaml
git commit -m "Add Loki and Fluent Bit"
git push
kubectl apply -f argocd/apps/logging.yaml -f argocd/apps/fluent-bit.yaml
argocd app sync loki && argocd app sync fluent-bit
```

Add Loki datasource in Grafana:
- Connections > Data sources > Add > Loki
- URL: `http://loki.logging.svc.cluster.local:3100`
- Save & test — should show green

### Loki + Fluent Bit Verify

```bash
# All pods running
kubectl get pods -n logging
# Expected: loki-0 (2/2), fluent-bit-* (1/1) per node

# Confirm Loki has labels
kubectl exec -n monitoring deployment/monitoring-grafana -c grafana -- \
  wget -qO- "http://loki.logging.svc.cluster.local:3100/loki/api/v1/labels"

# In Grafana Explore > Loki:
# Label filter: job = fluent-bit → Run query → logs visible
```

Known gap: namespace label not yet exposed — fix pending in Fluent Bit output config.
