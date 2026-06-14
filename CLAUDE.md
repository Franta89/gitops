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

1. `infra-terraform` provisions AKS, Azure AI Services (GPT-4.1-mini), AI Foundry Hub, Key Vault, Managed Identity, and installs Argo CD.
2. Copy two Terraform outputs into gitops manifests (see placeholders in `manifests/news-digest/`).
3. `kubectl apply -f bootstrap/root-app.yaml` registers the app-of-apps.
4. Argo CD syncs `apps/` in wave order: Strimzi (0) → Kafka (1) → cert-manager + ingress-nginx (2) → monitoring + postgresql (3) → config (4) → ddot app (5).

## Applications

| App                 | Wave | Namespace     | Purpose                           |
| ------------------- | ---- | ------------- | --------------------------------- |
| strimzi-operator    | 0    | kafka         | Strimzi CRDs + operator           |
| kafka-cluster       | 1    | kafka         | KRaft Kafka cluster               |
| cert-manager        | 2    | cert-manager  | TLS certificate management        |
| ingress-nginx       | 2    | ingress-nginx | Public NGINX ingress controller   |
| monitoring          | 3    | monitoring    | Prometheus + Grafana              |
| postgresql          | 3    | puzzle        | PostgreSQL for puzzle app         |
| cert-manager-config | 4    | cert-manager  | Let's Encrypt ClusterIssuer       |
| monitoring-config   | 4    | monitoring    | PodMonitors + dashboards          |
| news-digest (ddot)  | 5    | news-digest   | Daily Dose of Tech web app        |

## Daily Dose of Tech (ddot)

Public URL: **<https://dailydoseoftech.org>** and **<https://www.dailydoseoftech.org>**

DNS is live and proxied through **Cloudflare** (orange cloud enabled).
TLS certificates issued by Let's Encrypt via **DNS-01 challenge** (Cloudflare API token).
Cloudflare SSL/TLS mode must be set to **Full (strict)**.

The app gathers news in real time from authoritative **RSS/Atom feeds** (per-area
lists in `aggregator.py`: BBC, The Guardian, CNBC, The Register, BleepingComputer,
Al Jazeera, and more), uses Azure AI Services (GPT-4.1-mini) via the AI Foundry Hub
to pick and summarise the top 3–5 articles per area, and serves results on a web page.

Areas: Cloud Computing · AI Development · IT Security · Financial Markets · World News

Schedule: aggregator CronJob at 06:30 CET daily; monthly summary on the 1st.

News retrieval is keyless: RSS feeds need no API key, removing the previous NewsAPI
dependency (whose free tier was 24h-delayed and forbade production use). See
`docs/runbook.md` for the source list, look-back windows, and selection logic.

Auth: Azure Workload Identity (keyless) for Azure AI Services via AI Foundry Hub.
All secrets in Azure Key Vault via the AKV CSI driver — no key-based external APIs.

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
- **No plaintext secrets committed to git.** All secrets live in Azure Key Vault;
  the AKV CSI driver materialises them as standard K8s Secrets at pod startup.
  Plaintext secret file paths are listed in `.gitignore` as a safety net.
- All containers: `allowPrivilegeEscalation: false`, `capabilities.drop: ALL`,
  `seccompProfile: RuntimeDefault`. CPU and memory limits set on every container.

## Azure Key Vault secret workflow

All secrets live in Azure Key Vault (`kv-ddot-dev-swc-001`). The AKV CSI driver
(enabled as an AKS add-on) pulls them into pods at startup via `SecretProviderClass`
resources, creating standard K8s Secret objects that `envFrom: secretRef` consumes.

```text
# After terraform apply — paste the three outputs into both SecretProviderClass
# files and both secret-sa.yaml files:
terraform output key_vault_name          # → keyvaultName in SecretProviderClass
terraform output tenant_id               # → tenantId in SecretProviderClass
terraform output managed_identity_client_id  # → clientID in SecretProviderClass + SA annotation

# Secret values are stored by Terraform (in terraform.tfvars, which is gitignored).
# To rotate a secret: update the value in Azure Portal → AKV → secret → new version.
# The CSI driver picks up the new version within 2 minutes (rotation interval).
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
- [x] AKV CSI driver enabled on AKS; Key Vault, SecretProviderClass, and secret-sync resources added.
- [ ] After `terraform apply` in infra-terraform: fill AKV placeholders in SecretProviderClass files and secret-sa.yaml files.
- [ ] After `terraform apply` in infra-terraform: fill placeholders in serviceaccount.yaml and settings-configmap.yaml.
- [ ] After `terraform apply`: connect Azure OpenAI resource to the AI Foundry Hub in the Azure portal:
      AI Foundry Hub → Settings → Connected resources → + New connection → Azure OpenAI → select `oai-ddot-*`.
      Then replace `OPENAI_ENDPOINT` in `manifests/news-digest/config/settings-configmap.yaml`
      with the value from `terraform output foundry_hub_endpoint`.
- [ ] In Cloudflare dashboard: SSL/TLS → set mode to Full (strict), then enable orange-cloud proxy on both A records.
- [ ] Confirm latest Strimzi chart version in apps/strimzi-operator.yaml.
- [ ] Confirm the Kafka `version` and `metadataVersion` in manifests/kafka/kafka.yaml.
