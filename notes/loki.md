# Loki

## What It Is

Loki is Grafana's log aggregation system. Unlike most log databases, it
does **not** index the full text of log lines — it only indexes a small
set of labels (like `job`, `namespace`, `pod`) and stores the actual log
content in compressed chunks. Queries filter by label first (cheap, like
a Prometheus query) and only then grep through the matching chunks'
content (LogQL, Loki's query language, looks deliberately like PromQL).

## Why We Use It Here

It's the logs half of observability, pairing with Prometheus (metrics)
and eventually Tempo (traces) inside one Grafana UI. It was chosen over
`loki-stack` specifically — see the deprecation note below — and over
a full ELK-style stack because it's designed to run cheap: no full-text
indexing means far less storage and CPU than Elasticsearch for the same
log volume, which matters on a cluster with no PersistentVolumes at all.

## How It Works — Internals

Loki's architecture has four logical components — **distributor**,
**ingester**, **querier**, and **compactor** — plus an index and a chunk
store. In full "microservices" deployment mode these run as separate
Deployments that can scale independently. This repo runs
**`deploymentMode: SingleBinary`**, which collapses all of them into one
process/pod. Understanding the split still matters because it explains
what each config key controls:

- **Write path:** a log line arrives (from Fluent Bit) → the
  **distributor** hashes it by its label set and routes it to the owning
  **ingester** → the ingester buffers recent logs in memory, periodically
  flushing them as compressed **chunks** to the object store (here:
  the local filesystem, not S3/GCS — see `storage.type: filesystem`) →
  it also writes an **index** entry mapping labels → chunk location.
- **Read path:** a query first hits the index to find which chunks
  contain matching labels within the queried time range, then fetches
  and greps only those chunks. This is why a query like
  `{job="fluent-bit"}` is fast even over large log volumes, but a query
  with no label filter — or a label that doesn't exist, like `namespace`
  in this current setup — degrades to scanning everything.
- **Index format — `tsdb`:** the `schemaConfig` says `store: tsdb`. TSDB
  here reuses Prometheus's TSDB storage engine as Loki's index format
  (an index of label→chunk mappings, not metrics samples). It replaced
  the older `boltdb-shipper` format; both work the same way
  conceptually — chunk index sharded into periods — but `tsdb` has better
  query performance. `index.period: 24h` means a new index shard starts
  every 24 hours.
- **`replication_factor: 1`** — normally Loki replicates each log line to
  multiple ingesters for durability. Set to 1 here (single replica, no
  redundancy) because there's a single ingester pod anyway
  (`singleBinary.replicas: 1`) — replicating to yourself buys nothing.
- **Compactor** (not explicitly configured here, runs as part of the
  single binary) periodically merges small index files and applies
  retention. With no retention config set, Loki keeps data until it's
  evicted for other reasons (like the emptyDir workaround below wiping
  it on restart).

## Setup

`loki-stack` (deprecated, do not use) bundles Loki 2.6.1, which is
incompatible with Grafana 11.x's health check — hence the standalone
`grafana/loki` chart instead.

Deploy via ArgoCD — push to Git first:
```bash
git add logging/ argocd/apps/logging.yaml argocd/apps/fluent-bit.yaml
git commit -m "Add Loki and Fluent Bit"
git push
kubectl apply -f argocd/apps/logging.yaml -f argocd/apps/fluent-bit.yaml
argocd app sync loki && argocd app sync fluent-bit
```

Add Loki as a datasource in Grafana:
- Connections > Data sources > Add > Loki
- URL: `http://loki.logging.svc.cluster.local:3100`
- Save & test — should show green

## Configuration Deep Dive

`logging/loki/values.yaml`, key by key:

```yaml
deploymentMode: SingleBinary
```
Collapses distributor/ingester/querier/compactor into one process. The
right choice for a small cluster with no need to scale ingestion and
queries independently — full microservices mode would mean many more
pods for no benefit here.

```yaml
loki:
  auth_enabled: false
  commonConfig:
    replication_factor: 1
  storage:
    type: filesystem
```
- `auth_enabled: false` — disables Loki's multi-tenant auth (tenant ID
  header requirement). Single-tenant use, no need for it.
- `storage.type: filesystem` — chunks are written to local disk instead
  of an object store (S3/GCS/MinIO). This repo also explicitly disables
  `minio` and `gateway` below, confirming there's no object storage
  backend at all — everything lives on the pod's local filesystem.

```yaml
  schemaConfig:
    configs:
      - from: "2024-01-01"
        store: tsdb
        object_store: filesystem
        schema: v13
        index:
          prefix: loki_index_
          period: 24h
```
- `from: "2024-01-01"` — the date this schema version becomes active.
  Loki supports schema migrations over time; changing schema config for
  existing data requires adding a *new* dated entry, never editing this
  one in place.
- `schema: v13` — the index schema version (encodes how label/chunk
  references are laid out internally). `v13` is current as of this
  chart version.

```yaml
singleBinary:
  replicas: 1
  persistence:
    enabled: false
  extraVolumes:
    - name: loki-data
      emptyDir: {}
  extraVolumeMounts:
    - name: loki-data
      mountPath: /var/loki

backend:
  replicas: 0
read:
  replicas: 0
write:
  replicas: 0

minio:
  enabled: false
gateway:
  enabled: false
lokiCanary:
  enabled: false
test:
  enabled: false
chunksCache:
  enabled: false
resultsCache:
  enabled: false
```
- **`singleBinary.persistence.enabled: false` is load-bearing and
  non-obvious.** Setting `persistence.enabled: false` at the chart's
  top level does *not* disable persistence for `SingleBinary` mode — it
  has to be set specifically under `singleBinary.persistence`. Get this
  wrong and the chart tries to create a PVC, which fails outright since
  this cluster has no StorageClass/PV provisioner.
- `extraVolumes`/`extraVolumeMounts` mounting an `emptyDir` at
  `/var/loki` is the actual storage workaround: Loki still needs *some*
  writable disk at that path (it's hardcoded as its working directory
  for WAL, chunks-in-flight, etc.), so an ephemeral emptyDir stands in
  for a real PV. **Tradeoff: all log data is lost on pod restart.**
  Acceptable here since this is not a durability-critical setup.
  - **If you need to change persistence settings later**, note the repo's
    hard-won lesson: a StatefulSet's `volumeClaimTemplate` is immutable.
    Changing persistence on an *existing* Loki StatefulSet requires
    deleting the ArgoCD app (cascade), force-deleting the PVC and
    StatefulSet, then recreating the app — editing in place will fail.
- `backend`/`read`/`write` all forced to `replicas: 0` — these are the
  Deployments used only in microservices/simple-scalable mode. In
  `SingleBinary` mode they're irrelevant, so they're zeroed out rather
  than left at chart defaults (which would otherwise deploy unused
  pods).
- `minio`/`gateway`/`lokiCanary`/`test`/`chunksCache`/`resultsCache` all
  disabled — these are optional chart extras (bundled object storage,
  a query gateway/load-balancer, a synthetic log-write canary for
  self-testing, chart test hooks, and Redis-backed query caches) that
  add pods and complexity this small single-binary setup doesn't need.

## Verify

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
The `/loki/api/v1/labels` call hits Loki's own API to list every label
key it currently has an index for — a quick way to confirm ingestion is
actually happening without going through Grafana at all. If it returns
an empty list, no logs have been ingested yet (check Fluent Bit).

## What To Look For / Troubleshooting

- **Known gap — no `namespace` label.** `Auto_Kubernetes_Labels On` in
  Fluent Bit maps *pod* labels, not the namespace itself, into Loki
  labels. Today the only real workaround is filtering by
  `{job="fluent-bit"}` and grepping log content for namespace strings.
  The real fix is Fluent Bit's config, not Loki's — see `fluent-bit.md`.
- **PVC creation error on deploy** → `persistence.enabled` was set at
  the wrong level. It must be under `singleBinary.persistence`, not the
  chart's top-level `persistence` key.
- **All logs gone after a Loki pod restart** → expected. This is the
  `emptyDir` tradeoff, not a bug — there's no real disk here.
- **Changed persistence config, StatefulSet failed to update** → you hit
  the `volumeClaimTemplate` immutability issue. Delete the ArgoCD app
  (cascade), force-delete the PVC/StatefulSet, recreate the app.
- **Query with only a text filter, no label filter, is very slow** →
  expected given Loki's architecture — always filter by label first
  (`{job="..."}`) before adding a `|=` text filter; an unfiltered query
  scans every chunk in the time range.
