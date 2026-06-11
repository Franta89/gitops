# Repo: gitops

## What this repo is

The GitOps source of truth for everything that runs INSIDE the AKS cluster:
the Strimzi/Kafka messaging layer and the **Daily Dose of Tech** (ddot) web
application. Argo CD watches this repo and syncs it into the cluster.
**Dev/test sandbox only — NOT production.**

## Companion repo

The cluster itself (AKS, network, Azure AI services, Argo CD install) is
provisioned by a SEPARATE repo: `infra-terraform`. Nothing here creates Azure
resources. Run `terraform apply` there first; Argo CD and the AI services must
exist before the ddot app can run.

## How it flows

1. `infra-terraform` provisions AKS, Azure OpenAI, Bing Search, Managed Identity, and installs Argo CD.
2. Copy two Terraform outputs into gitops manifests (see placeholders in `manifests/news-digest/`).
3. `kubectl apply -f bootstrap/root-app.yaml` registers the app-of-apps.
4. Argo CD syncs `apps/` in wave order: Strimzi (0) → Kafka (1) → cert-manager + ingress-nginx (2) → monitoring + postgresql (3) → config (4) → ddot app (5).

## Applications

| App                 | Wave | Namespace     | Purpose                                        |
| ------------------- | ---- | ------------- | ---------------------------------------------- |
| strimzi-operator    | 0    | kafka         | Strimzi CRDs + operator                        |
| kafka-cluster       | 1    | kafka         | KRaft Kafka cluster                            |
| cert-manager        | 2    | cert-manager  | TLS certificate management                     |
| ingress-nginx       | 2    | ingress-nginx | Public NGINX ingress controller                |
| sealed-secrets      | 2    | kube-system   | SealedSecrets operator (decrypt SealedSecrets) |
| monitoring          | 3    | monitoring    | Prometheus + Grafana                           |
| postgresql          | 3    | puzzle        | PostgreSQL for puzzle app                      |
| cert-manager-config | 4    | cert-manager  | Let's Encrypt ClusterIssuer                    |
| monitoring-config   | 4    | monitoring    | PodMonitors + dashboards                       |
| news-digest (ddot)  | 5    | news-digest   | Daily Dose of Tech web app                     |

## Daily Dose of Tech (ddot)

Public URL: **<https://dailydoseoftech.org>** and **<https://www.dailydoseoftech.org>**

DNS is live and proxied through **Cloudflare** (orange cloud enabled).
TLS certificates issued by Let's Encrypt via **DNS-01 challenge** (Cloudflare API token).
Cloudflare SSL/TLS mode must be set to **Full (strict)**.

The app fetches news via Bing Search API, uses Azure OpenAI (GPT-4o-mini) to
pick and summarise the top 3 articles per area, and serves results on a web page.

Areas: Azure Cloud · AI Development · World News · IT Security

Schedule: aggregator CronJob at 06:10 CET daily; monthly summary on the 1st.

Auth: Azure Workload Identity (keyless) for OpenAI. Bing API key stored in
`bing-secret` (Bing Search v7 has no RBAC support — documented exception).

## Key decisions / constraints

- Kafka runs in **KRaft mode only**. No ZooKeeper.
- Default topology = 1 dual-role node, replication factors 1 (minimal test).
- Public images only. No private registry.
- Scripts for ddot are mounted via ConfigMap into `python:3.12-slim` pods with
  pinned pip versions. Acceptable for dev/test; a production version would use
  custom images in ACR.
- NetworkPolicy restricts postgres access to API and worker pods only.

## Conventions

- One Argo CD Application per concern under `apps/`.
- Raw K8s manifests under `manifests/`.
- Sync waves order dependencies. Never skip a wave without a documented reason.
- **No plaintext secrets committed to git.** Real-value secret files are in `.gitignore`.
  Each secret has a `*.yaml.example` template (committed, placeholder values) and a
  `*sealedsecret.yaml` file (committed, kubeseal-encrypted — safe to store in git).
  SealedSecrets operator is deployed at wave 2 (`apps/sealed-secrets.yaml`).
- All containers: `allowPrivilegeEscalation: false`, `capabilities.drop: ALL`,
  `seccompProfile: RuntimeDefault`. CPU and memory limits set on every container.

## Sealing a secret (workflow)

```text
# 1. Fetch the cluster public cert (once per cluster lifetime, don't commit it):
kubeseal --fetch-cert \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system \
  > pub-cert.pem

# 2. For each secret — copy the .yaml.example, fill in real values, then seal:
cp manifests/cert-manager/cloudflare-api-token-secret.yaml.example \
   manifests/cert-manager/cloudflare-api-token-secret.yaml
# edit cloudflare-api-token-secret.yaml with the real token, then:
kubeseal --cert pub-cert.pem --format yaml \
  < manifests/cert-manager/cloudflare-api-token-secret.yaml \
  > manifests/cert-manager/cloudflare-api-token-sealedsecret.yaml

# 3. Repeat for the other two secrets (see their .yaml.example files for exact paths).

# 4. Commit the *sealedsecret.yaml files. Delete the plaintext .yaml files from disk.
```

## Repo layout

```text
bootstrap/root-app.yaml       app-of-apps root Application (apply once)
apps/                         Argo CD Applications
manifests/
  kafka/                      KafkaNodePool + Kafka CRs
  cert-manager/               ClusterIssuer
  monitoring/                 PodMonitor + Grafana dashboards
  news-digest/                Daily Dose of Tech application
    namespace.yaml            Namespace with workload-identity label
    serviceaccount.yaml       ServiceAccount with Managed Identity annotation (fill after tf apply)
    networkpolicy.yaml        Restrict postgres, api, and frontend ingress
    ingress.yaml              dailydoseoftech.org + www, DNS-01 TLS
    postgres/                 StatefulSet, Service, Secret (dev placeholder)
    config/                   ConfigMaps: settings, scripts (aggregator.py, api.py), frontend HTML/CSS/JS
    api/                      FastAPI Deployment + Service
    frontend/                 nginx Deployment + Service
    cronjobs/                 daily (06:10 CET) + monthly (1st of month)
```

## TODO for Claude Code

- [x] Replace `<ORG>` in bootstrap/root-app.yaml and apps/kafka-cluster.yaml. (Franta89)
- [x] Daily Dose of Tech app added: manifests/news-digest/, apps/news-digest.yaml.
- [x] SealedSecrets operator added: apps/sealed-secrets.yaml (wave 2). Secret templates in *.yaml.example files.
- [ ] After `terraform apply` in infra-terraform: fill placeholders in serviceaccount.yaml and settings-configmap.yaml.
- [ ] Seal all three secrets using kubeseal (see "Sealing a secret" workflow above). Commit the *sealedsecret.yaml files.
- [ ] In Cloudflare dashboard: SSL/TLS → set mode to Full (strict), then enable orange-cloud proxy on both A records.
- [ ] Confirm latest Strimzi chart version in apps/strimzi-operator.yaml.
- [ ] Confirm the Kafka `version` and `metadataVersion` in manifests/kafka/kafka.yaml.
- [ ] Verify sealed-secrets chart version in apps/sealed-secrets.yaml matches latest release.
