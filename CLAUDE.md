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

1. `infra-terraform` provisions AKS, Azure AI Services (GPT-5.4-mini), AI Foundry Hub, Key Vault, Managed Identity, and installs Argo CD.
2. Copy two Terraform outputs into gitops manifests (see placeholders in `manifests/news-digest/`).
3. `kubectl apply -f bootstrap/root-app.yaml` registers the app-of-apps.
4. Argo CD syncs `apps/` in wave order: Strimzi (0) → Kafka (1) → cert-manager + alb-controller (2) → monitoring + postgresql (3) → config (4) → ddot app (5).

## Applications

| App                 | Wave | Namespace        | Purpose                            |
| ------------------- | ---- | ---------------- | ---------------------------------- |
| strimzi-operator    | 0    | kafka            | Strimzi CRDs + operator            |
| kafka-cluster       | 1    | kafka            | KRaft Kafka cluster                |
| cert-manager        | 2    | cert-manager     | TLS certificate management         |
| alb-controller      | 2    | azure-alb-system | Gateway API controller (Azure AGC) |
| monitoring          | 3    | monitoring       | Prometheus + Grafana               |
| postgresql          | 3    | puzzle           | PostgreSQL for puzzle app          |
| cert-manager-config | 4    | cert-manager     | Let's Encrypt ClusterIssuer        |
| monitoring-config   | 4    | monitoring       | PodMonitors + dashboards           |
| news-digest (ddot)  | 5    | news-digest      | Daily Dose of Tech web app         |

## Daily Dose of Tech (ddot)

Public URL: **<https://dailydoseoftech.org>** and **<https://www.dailydoseoftech.org>**

DNS is live and proxied through **Cloudflare** (orange cloud enabled). Because the
public entry point is now **Azure Application Gateway for Containers (AGC)**, which
exposes a generated FQDN rather than a static IP, the Cloudflare records for both
hosts are **CNAMEs → the AGC FQDN** (the Gateway's runtime address — see below); Cloudflare
CNAME-flattening makes this work at the apex. TLS certificates issued by Let's Encrypt
via **DNS-01 challenge** (Cloudflare API token) — an explicit cert-manager
`Certificate` (`manifests/news-digest/certificate.yaml`) writes the `dailydoseoftech-tls`
Secret that the Gateway's HTTPS listener references. Cloudflare SSL/TLS mode must be
set to **Full (strict)**.

### Routing — Gateway API (not Ingress)

Traffic enters via the **Gateway API** (`ingress-nginx` was retired; it is now
upstream maintenance-only). The `alb-controller` app installs the Gateway API CRDs
and the `azure-alb-external` GatewayClass and programs the Terraform-provisioned AGC
(BYO model). A single shared `Gateway` (`ddot-gateway` in `news-digest`) terminates
TLS; `HTTPRoute`s attach to it: the app at `/` (`manifests/news-digest/httproute.yaml`)
and Grafana at `/grafana` (`manifests/monitoring/grafana-httproute.yaml`, cross-namespace).
An HTTP→HTTPS 301 redirect route replaces the old nginx force-ssl-redirect annotation.

The app gathers news in real time from authoritative **RSS/Atom feeds** (grouped by
**family** in `aggregator.py` — tech/finance/world/euro — plus vendor newsrooms:
BBC, The Guardian, CNBC, The Register, BleepingComputer, Al Jazeera, AWS, OpenAI,
and more), then **categorizes each story by content** (`route_tech()` entity
routing plus one `classify_articles()` AI call) so the area depends on what the
story is about, not which feed carried it. It uses Azure AI Services (GPT-5.4-mini) — called directly
against the AI Services account's **v1 API** (`*.openai.azure.com/openai/v1/`,
stock `OpenAI` client + Entra bearer token, no dated api-version), not via the
Foundry Hub — to pick and summarise the top ~5 articles per area, and serves
results on a web page.

Areas: Cloud Computing · AI Development · IT Security · Financial Markets · World News ·
Euro News (Europe-only). The four topic areas tag each item European/global and show
a "Europe" group; World News is global-excluding-Europe, Euro News its counterpart.

Schedule: aggregator CronJob at 06:30 CET daily; monthly summary on the 1st.

News retrieval is keyless: RSS feeds need no API key, removing the previous NewsAPI
dependency (whose free tier was 24h-delayed and forbade production use). See
`README.md` → "Daily Dose of Tech" for the source list, look-back windows, AI
selection logic, operations, and troubleshooting.

Auth: Azure Workload Identity (keyless) for Azure AI Services — direct v1 API
(`*.openai.azure.com/openai/v1/`); the AI Foundry Hub is provisioned but not
in the inference path. All secrets in Azure Key Vault via the AKV CSI driver — no
key-based external APIs.

## Key decisions / constraints

- Kafka runs in **KRaft mode only**. No ZooKeeper.
- Default topology = 1 dual-role node, replication factors 1 (minimal test).
- Public images only. No private registry.
- Scripts for ddot are mounted via ConfigMap into `python:3.12-slim` pods with
  pinned pip versions. Acceptable for dev/test; a production version would use
  custom images in ACR.
- NetworkPolicy restricts postgres access to API and worker pods only. Public
  ingress to the frontend is allowed from the **AGC delegated subnet CIDR**
  (`ipBlock`, not a namespaceSelector) because AGC delivers traffic from
  `snet-alb`, not from an in-cluster controller pod. Keep the CIDR in
  `networkpolicy.yaml` in sync with `snet-alb` in infra-terraform.
- The **frontend** assets live in two ConfigMaps (`frontend-assets`,
  `frontend-nginx`) mounted as **directories** (not subPath), so edits propagate
  to the running pod automatically — no rollout restart needed. Still bump
  `app.js?v=N` on `app.js` changes for browser/CDN cache-busting. See
  `README.md` → "Daily Dose of Tech" → Operations → Frontend changes.

## Conventions

- One Argo CD Application per concern under `apps/`.
- Raw K8s manifests under `manifests/`.
- Sync waves order dependencies. Never skip a wave without a documented reason.
- **No plaintext secrets committed to git.** All secrets live in Azure Key Vault;
  the AKV CSI driver materialises them as standard K8s Secrets at pod startup.
  Plaintext secret file paths are listed in `.gitignore` as a safety net.
- All containers: `allowPrivilegeEscalation: false`, `capabilities.drop: ALL`,
  `seccompProfile: RuntimeDefault`. CPU and memory limits set on every container.
- **Full security posture (how the cluster is hardened) is documented in
  [`SECURITY.md`](SECURITY.md);** forward-looking backlog in `infra-terraform/whatnext.md`.
- **Monitoring setup (Prometheus/Grafana, dashboards, Cloudflare traffic pipeline)
  is documented in [`MONITORING.md`](MONITORING.md).**

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
    networkpolicy.yaml        Restrict postgres + api; allow frontend from AGC subnet
    gateway.yaml              Gateway API Gateway (AGC); fill alb-id after tf apply
    httproute.yaml            HTTPRoutes: app at / + HTTP→HTTPS redirect
    certificate.yaml          cert-manager Certificate → dailydoseoftech-tls Secret
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
- [x] After `terraform apply`: set `OPENAI_ENDPOINT` in `settings-configmap.yaml` to the
      AI Services **v1 API** base URL (`https://ais-ddot-dev-swc-001.openai.azure.com/openai/v1/`)
      and `OPENAI_DEPLOYMENT` to `gpt-5.4-mini`. The app calls AI Services directly; the
      Foundry Hub is provisioned but NOT in the inference path (no Hub connection needed).
- [ ] After `terraform apply`: paste `alb_controller_client_id` into `apps/alb-controller.yaml`
      (`albController.podIdentity.clientID`) and `alb_id` into `manifests/news-digest/gateway.yaml`
      (`alb.networking.azure.io/alb-id`). If `snet-alb` is not `10.0.2.0/24`, also update the
      `ipBlock` in `manifests/news-digest/networkpolicy.yaml` (`terraform output alb_subnet_cidr`).
- [ ] In Cloudflare dashboard: SSL/TLS → set mode to Full (strict). Point both records at the
      **Gateway's runtime FQDN** as a CNAME (CNAME-flattening handles the apex), then keep
      orange-cloud proxy enabled on both. Get the FQDN from the Gateway status (the ALB
      controller manages the frontend, so it is NOT a terraform output):
      `kubectl get gateway ddot-gateway -n news-digest -o jsonpath='{.status.addresses[0].value}'`
- [ ] After `terraform apply` (security pass): paste `secrets_sync_client_id` into the
      **monitoring + cert-manager** `secret-provider-class.yaml`, `service-account.yaml`,
      and `secret-sa.yaml` (replaces the old shared clientID — those pods now use a
      secrets-only identity). The news-digest SA/SPC keep `managed_identity_client_id`.
- [ ] After `terraform apply` (security pass): add `manifests/news-digest/waf.yaml`
      (`WebApplicationFirewallPolicy` targeting `ddot-gateway`) with
      `webApplicationFirewall.id = terraform output waf_policy_id`. Template + rationale
      in `infra-terraform/whatnext.md` (kept out of the repo until the policy exists so
      Argo never pushes an invalid id at the live Gateway).
- [x] Cloudflare `cloudflare-api-token` granted **Zone:Read + Analytics:Read** (token
      value unchanged). Site-traffic metrics now flow via the `cf-analytics` CronJob →
      Pushgateway → Prometheus (`ddot_cloudflare_*`), shown in the Grafana "Site Traffic
      — Cloudflare" row. Verify: `kubectl create job --from=cronjob/cf-analytics t -n monitoring`
      then `kubectl logs job/t -n monitoring` (should log "pushed cloudflare metrics …").
- [ ] After `terraform apply` ("Listen"/TTS pass): fill `SPEECH_RESOURCE_ID` and
      `AUDIO_STORAGE_ACCOUNT` in `manifests/news-digest/config/settings-configmap.yaml`
      from `terraform output speech_resource_id` and `terraform output
      audio_storage_account_name` (`SPEECH_ENDPOINT` is already the custom-domain URL).
      The news-digest identity gains `Cognitive Services Speech User` + `Storage Blob
      Data Contributor` (keyless); the `audio` blob container has a 7-day lifecycle.
      The `/api/audio` endpoint synthesizes (SSML → Speech REST) and Blob-caches the
      MP3 the Listen UI plays. See `roadmap.md`.
- [ ] Confirm latest Strimzi chart version in apps/strimzi-operator.yaml.
- [ ] Confirm the Kafka `version` and `metadataVersion` in manifests/kafka/kafka.yaml.
