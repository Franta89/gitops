# Monitoring — Daily Dose of Tech / AKS

How the cluster and the **Daily Dose of Tech** (ddot) app are observed, layer by
layer. **Dev/test sandbox** (not production), but wired with production patterns.
Everything here is GitOps — Argo CD syncs it from this repo; nothing is clicked in
by hand. Azure-side resources (AKS, Key Vault, managed identities) come from the
companion `infra-terraform` repo. Security posture is documented separately in
[`SECURITY.md`](SECURITY.md).

## Stack at a glance

| Component | Source | Namespace | Purpose |
| --------- | ------ | --------- | ------- |
| Prometheus | `kube-prometheus-stack` chart (`monitoring` app, wave 3) | `monitoring` | Metrics store + scraping |
| Grafana | same chart | `monitoring` | Dashboards + visualisation |
| node-exporter | same chart | `monitoring` | Node-level CPU/mem/disk metrics |
| kube-state-metrics | same chart | `monitoring` | K8s object state (pods, jobs, requests…) |
| Pushgateway | `prom/pushgateway` ([pushgateway.yaml](manifests/monitoring/pushgateway.yaml)) | `monitoring` | Sink for the batch `cf-analytics` job |
| cf-analytics | CronJob ([cf-analytics-cronjob.yaml](manifests/monitoring/cf-analytics-cronjob.yaml)) | `monitoring` | Pulls Cloudflare site stats → Pushgateway |
| Dashboards | ConfigMaps ([dashboard-cluster.yaml](manifests/monitoring/dashboard-cluster.yaml), [dashboard-app.yaml](manifests/monitoring/dashboard-app.yaml)) | `monitoring` | Provisioned Grafana dashboards |

Two Argo CD apps split install from config so CRDs exist before resources that use
them:

- **`monitoring`** (wave 3, [apps/monitoring.yaml](apps/monitoring.yaml)) — the
  Helm stack (Prometheus, Grafana, exporters, CRDs).
- **`monitoring-config`** (wave 4, [apps/monitoring-config.yaml](apps/monitoring-config.yaml))
  — everything under [`manifests/monitoring/`](manifests/monitoring/): dashboards,
  the Pushgateway + its ServiceMonitor, the cf-analytics CronJob, the Grafana
  HTTPRoute/HealthCheckPolicy, and secret plumbing.

## Metrics collection

Prometheus is configured to **scrape every `PodMonitor` and `ServiceMonitor` in
all namespaces** — the selectors are deliberately wide open
([apps/monitoring.yaml](apps/monitoring.yaml)):

```yaml
podMonitorSelectorNilUsesHelmValues: false
serviceMonitorSelectorNilUsesHelmValues: false
podMonitorNamespaceSelector: {}
serviceMonitorNamespaceSelector: {}
```

So adding a `ServiceMonitor`/`PodMonitor` anywhere is enough to get scraped — no
allow-list to edit. Notable scrape targets:

- **Cluster/node** — node-exporter, kubelet, kube-state-metrics, API server,
  scheduler, controller-manager, etcd, CoreDNS (all from the stack's own
  ServiceMonitors).
- **cert-manager** — its chart enables `prometheus.servicemonitor`
  ([apps/cert-manager.yaml](apps/cert-manager.yaml)), exposing
  `certmanager_certificate_expiration_timestamp_seconds` used by the TLS-expiry
  panel.
- **Pushgateway** — a `ServiceMonitor` in [pushgateway.yaml](manifests/monitoring/pushgateway.yaml)
  scrapes the Cloudflare metrics (see below).

**Retention:** Prometheus keeps **3 days** on a **5Gi** PVC. Grafana persists its
DB on a **1Gi** PVC. Both are intentionally small for a sandbox.

## Cloudflare site-traffic pipeline

Real visitor metrics are pull-based via a batch job, since Cloudflare data isn't a
Prometheus exporter:

```
cf-analytics CronJob (every 15 min)  →  Pushgateway  →  ServiceMonitor  →  Prometheus  →  Grafana
   queries Cloudflare GraphQL API        :9091           scrape              ddot_cloudflare_*
```

- [cf-analytics-cronjob.yaml](manifests/monitoring/cf-analytics-cronjob.yaml) runs
  `*/15 * * * *`, calls the Cloudflare GraphQL Analytics API for the zone, and
  pushes gauges to the Pushgateway: `ddot_cloudflare_uniques`, `_requests`,
  `_pageviews`, `_bytes`, `_threats`, `_cached_requests`, plus per-day and
  per-threat-type series.
- Auth: `CF_ZONE_ID` + `CF_API_TOKEN` (from the `cloudflare-exporter-token`
  secret; the token has **Zone:Read + Analytics:Read**).
- Verify manually:
  ```bash
  kubectl create job --from=cronjob/cf-analytics t -n monitoring
  kubectl logs job/t -n monitoring   # expect "pushed cloudflare metrics …"
  ```

## Dashboards

Dashboards are **provisioned as ConfigMaps** labelled `grafana_dashboard: "1"`; the
Grafana sidecar auto-loads them. Edits propagate on the next sidecar reload — **no
pod restart needed**, and **no manual edits in the Grafana UI** (they aren't
persisted; this repo is the source of truth).

### DDoT — Cluster Health (`uid: ddot-cluster`)

[dashboard-cluster.yaml](manifests/monitoring/dashboard-cluster.yaml). Cluster
capacity + news-digest app health.

- **Cluster Health row:** CPU Utilisation (live usage), **CPU Reserved (Requests)**
  (scheduler reservation = `sum(pod cpu requests) / node allocatable`), Memory
  Utilisation, Running Pods, Pending / Failed Pods, and a CPU & Memory — Node
  timeseries (usage vs reserved vs memory).
- **Application Health — news-digest row:** Pod Restarts (24h), Container Memory,
  Container CPU, PostgreSQL PVC Used, and **Aggregator CronJob — Last Run**
  (completion time / failures for the daily digest job).

> **Why both CPU gauges?** Kubernetes schedules by **requests, not usage**. On the
> single 2-vCPU node, reservation runs ~94% while live usage is ~22% — which is why
> a pod can sit `Pending` ("Insufficient cpu") on an apparently idle node. The
> "CPU Reserved (Requests)" gauge surfaces that gap; see the node-capacity note
> under Operations.

### DDoT — Application & Traffic (`uid: ddot-app`)

[dashboard-app.yaml](manifests/monitoring/dashboard-app.yaml). Public-site traffic
from the Cloudflare pipeline plus the TLS-expiry panel.

- **Site Traffic — Cloudflare row:** Unique Visitors / Total Requests / Bandwidth /
  Threats / Page Views (today), 30-day trends (requests, page views, unique
  visitors, threats, threats-by-type), and **TLS Cert — time to expiry**
  (cert-manager metric for `dailydoseoftech-tls`).

## Access

- **URL:** <https://dailydoseoftech.org/grafana> — exposed via **Gateway API**
  (not Ingress). The kube-prometheus-stack nginx Ingress is disabled; an
  `HTTPRoute` ([grafana-httproute.yaml](manifests/monitoring/grafana-httproute.yaml))
  attaches cross-namespace to `ddot-gateway` in `news-digest` on the `/grafana`
  PathPrefix. Grafana runs with `serve_from_sub_path` + `root_url=/grafana`, so the
  prefix is preserved (no rewrite).
- **Backend health:** AGC probes `/` by default, which Grafana 403s; a
  `HealthCheckPolicy` ([grafana-healthcheck.yaml](manifests/monitoring/grafana-healthcheck.yaml))
  points the probe at `/api/health` (returns 200) so the backend stays healthy.
- **Login:** `admin` / password from Key Vault (see Secrets) — not committed.

## Secrets

No plaintext secrets in git. Grafana admin creds and the Cloudflare token live in
Azure Key Vault (`kv-ddot-dev-swc-001`) and are materialised by the **AKV CSI
driver**:

- [secret-provider-class.yaml](manifests/monitoring/secret-provider-class.yaml) —
  `SecretProviderClass monitoring-akv` maps `grafana-admin-user`,
  `grafana-admin-password`, and `cloudflare-api-token` from AKV; it syncs the
  `grafana-admin-secret` K8s Secret that the Grafana chart reads via
  `existingSecret`.
- [secret-sync-deployment.yaml](manifests/monitoring/secret-sync-deployment.yaml) —
  a tiny `pause` pod that mounts the CSI volume so the Secret object is created and
  kept refreshed even when no other consumer is mounting it.
- [service-account.yaml](manifests/monitoring/service-account.yaml) —
  `monitoring-secret-sa` carries the **secrets-only** workload-identity client ID
  (separate from the news-digest app identity; see `SECURITY.md`).

## Operations

- **Add a scrape target:** create a `ServiceMonitor`/`PodMonitor` anywhere — the
  wide-open selectors pick it up. No Prometheus config change.
- **Add / edit a dashboard:** edit (or add) a ConfigMap with the
  `grafana_dashboard: "1"` label under `manifests/monitoring/`; commit. Argo syncs,
  the sidecar reloads — no restart.
- **Node capacity:** the single 2-vCPU node runs near 100% CPU **requests**. Before
  adding workloads or bumping replicas/requests, check the **CPU Reserved** gauge or
  `kubectl describe node | grep -A3 'Allocated resources'`. A `Pending`/preempted
  pod here almost always means request pressure, not real load.
- **Cloudflare metrics missing/stale:** run the manual `cf-analytics` job above and
  check its logs; confirm Pushgateway is up (`kubectl get deploy pushgateway -n monitoring`)
  and its `ServiceMonitor` is being scraped (Prometheus → Targets).

## Key decisions / constraints

- **Alerting is disabled.** Alertmanager is turned off
  ([apps/monitoring.yaml](apps/monitoring.yaml)); this is observability-only for a
  sandbox. Enabling Alertmanager + `PrometheusRule`s is a known next step.
- **Short retention, small PVCs** (Prometheus 3d/5Gi, Grafana 1Gi) — sized for
  dev/test, not long-term capacity planning.
- **Gateway API, not Ingress** — consistent with the rest of the cluster after
  ingress-nginx was retired.
- **GitOps is the source of truth** — dashboards and config are reconciled from
  this repo; UI edits don't persist. Make changes here.
