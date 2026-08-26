# ArgoCD

## What It Is

ArgoCD is a GitOps continuous delivery controller for Kubernetes. It runs
inside the cluster, watches one or more Git repositories, and continuously
reconciles live cluster state to match what's declared in Git. Every
resource it manages is described by an `Application` custom resource
(`argoproj.io/v1alpha1`), which tells ArgoCD: which repo/path/chart to
read, which cluster/namespace to deploy into, and how to keep them in sync.

## Why We Use It Here

It's the foundation the rest of this repo builds on — "GitOps" here
specifically means "ArgoCD reconciles this repo against the cluster."
Every other tool (Prometheus, Loki, Rollouts, Istio, Cilium) is installed
*as* an ArgoCD `Application`, not with raw `helm install`. That gives one
consistent story for every tool: push to Git, ArgoCD picks it up, drift
gets corrected automatically.

## How It Works

- ArgoCD's `application-controller` polls each `Application`'s source
  repo on an interval (default 3 minutes) and diffs the rendered
  manifests against live cluster state.
- `automated.selfHeal: true` means if someone runs `kubectl edit` directly
  against a resource ArgoCD manages, ArgoCD reverts it back to what's in
  Git on the next reconcile. Git — not the cluster — is the source of
  truth.
- `automated.prune: true` means removing a resource from Git (e.g.
  deleting a YAML file) causes ArgoCD to delete it from the cluster too.
  Without this, deleted-from-Git resources are just orphaned in the
  cluster.
- **Two-source pattern**: for Helm-based tools, one source points at the
  public Helm chart repo (for the chart itself), and a second source
  points at *this* Git repo with `ref: values`, supplying just the
  `values.yaml` via `$values/<path>`. This lets us pin a chart version
  from its upstream repo while keeping full config history and diffs for
  our own `values.yaml` in our own Git log. See `monitoring.yaml` for the
  reference example (`argocd/apps/monitoring.yaml`).
- Simple apps with no Helm chart (like `sample-app`) use the older
  single-`source` field instead, pointing straight at a path in this repo.

## Setup

ArgoCD is not managed by ArgoCD itself — it has to be installed first,
before GitOps can take over anything else.

```bash
kubectl create namespace argocd
kubectl apply --server-side --force-conflicts -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.3.8/manifests/install.yaml
```

**Why `--server-side --force-conflicts`:** ArgoCD's CRDs are large. Default
client-side apply fails with `metadata.annotations: Too long` because the
entire previous-state annotation kubectl stores client-side blows past the
Kubernetes annotation size limit. Server-side apply tracks field ownership
on the API server instead, avoiding that annotation entirely. This is a
hard requirement for large CRDs generally in this cluster, not just
ArgoCD — see the same flag used for `monitoring`'s `syncOptions`.

Expose the UI (no ingress controller in this cluster, so NodePort):
```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
```

Get the generated initial admin password:
```bash
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

Log in via CLI, register this repo, and deploy the first app
(`sample-app`, a static nginx reference app used to validate the loop
end-to-end):
```bash
argocd login <node-ip>:<nodeport> --username admin --password <password> --insecure
argocd repo add https://github.com/coolteddy/k8s-gitops.git
kubectl apply -f argocd/apps/sample-app.yaml
```

**Critical ordering rule:** always push to Git *before* applying the
`Application` manifest. ArgoCD clones from Git — it has no visibility into
your local uncommitted files. Applying an `Application` that points at a
path/values file that isn't pushed yet just gives you a sync error.

## Configuration Deep Dive

`argocd/apps/sample-app.yaml` — the minimal single-source shape:
```yaml
spec:
  source:
    repoURL: https://github.com/coolteddy/k8s-gitops.git
    targetRevision: main
    path: apps/sample-app
  destination:
    server: https://kubernetes.default.svc
    namespace: sample-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```
- `destination.server: https://kubernetes.default.svc` — deploys to the
  same cluster ArgoCD itself runs in (as opposed to a remote cluster
  registered separately).
- `syncOptions: [CreateNamespace=true]` — lets ArgoCD create the
  `sample-app` namespace itself instead of requiring it to pre-exist.

`argocd/apps/monitoring.yaml` — the two-source shape used for every
Helm-based tool from here on:
```yaml
spec:
  sources:
    - repoURL: https://prometheus-community.github.io/helm-charts
      chart: kube-prometheus-stack
      targetRevision: 70.4.2
      helm:
        valueFiles:
          - $values/monitoring/prometheus/values.yaml
    - repoURL: https://github.com/coolteddy/k8s-gitops.git
      targetRevision: main
      ref: values
  syncOptions:
    - CreateNamespace=true
    - ServerSideApply=true
```
- `ref: values` on the second source names it, so the first source's
  `$values/...` path can reference it.
- `targetRevision: 70.4.2` on the chart source pins the Helm chart
  version explicitly — chart upgrades are a deliberate Git change to this
  number, not something that happens implicitly.
- `ServerSideApply=true` — required here for the same CRD-size reason as
  the ArgoCD install itself (kube-prometheus-stack ships large CRDs).

## Verify

```bash
# All 7 ArgoCD pods running
kubectl get pods -n argocd

# Sample app deployed by ArgoCD
kubectl get pods -n sample-app

# App sync status — should show Synced / Healthy
argocd app get sample-app
```
A `Synced` status with no diff means live cluster state matches Git
exactly. `OutOfSync` means either a manual change was made (selfHeal will
correct it on the next reconcile) or Git has an unapplied change.

**GitOps loop test** — proves the whole reconcile pipeline end-to-end:
```bash
# change replicas in apps/sample-app/deployment.yaml, push, then:
argocd app sync sample-app
kubectl get pods -n sample-app -w

# revert test — confirms selfHeal/rollback works via Git, not kubectl
git revert HEAD --no-edit && git push
argocd app sync sample-app
kubectl get pods -n sample-app -w
```

## What To Look For / Troubleshooting

- **`metadata.annotations: Too long` on apply** → you used client-side
  apply on a resource with large CRDs. Re-run with `--server-side
  --force-conflicts`.
- **App stuck `OutOfSync` after a push** → check you actually pushed
  before applying/syncing; ArgoCD only sees Git, not your working
  directory. `git log origin/main -1` vs `git log -1` to confirm they
  match.
- **App shows changes you didn't make** → don't assume it's broken.
  Check `git log`/`git blame` on the affected path first — this repo is
  also used from other sessions against a separate EKS cluster, so
  externally-made commits are expected, not drift.
- **Manual `kubectl edit` doesn't stick** → this is `selfHeal` working as
  intended, not a bug. The fix is to change Git, not the cluster.
