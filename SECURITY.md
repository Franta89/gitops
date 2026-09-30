# Security posture — Daily Dose of Tech / AKS

How this cluster is hardened, layer by layer. **Dev/test sandbox** (not production),
but hardened with production patterns. Azure-side controls live in the companion
`infra-terraform` repo; in-cluster controls live here (gitops). Forward-looking
backlog is tracked in `infra-terraform/whatnext.md`.

## Defense in depth (summary)

| Layer | Control | Where |
| ----- | ------- | ----- |
| Edge | Cloudflare proxy (orange-cloud), SSL/TLS **Full (strict)** | Cloudflare dashboard |
| Edge | **WAF** — Cloudflare managed rules at the proxy | Cloudflare dashboard |
| Edge | **Origin lock** — the ingress public IP accepts only Cloudflare edge ranges, enforced as NSG rules AKS renders from `loadBalancerSourceRanges` | `manifests/envoy-gateway/gatewayclass.yaml` |
| Ingress | **Gateway API** via **Envoy Gateway**, in-cluster (AGC retired 2026-08-07; ingress-nginx before it) | `apps/envoy-gateway.yaml`, `manifests/envoy-gateway/`, `manifests/news-digest/gateway.yaml` |
| Ingress | TLS — Let's Encrypt via **DNS-01** (Cloudflare), HTTP→HTTPS 301 redirect | `manifests/news-digest/certificate.yaml`, `httproute.yaml` |
| Network | **NetworkPolicy enforced** (Cilium dataplane) | `infra-terraform` AKS module (`network_policy=cilium`) |
| Network | Postgres reachable only from API/worker pods; API only from frontend (identity-based) | `manifests/news-digest/networkpolicy.yaml` |
| Network | NSG: public inbound to nodes restricted to **Cloudflare ranges only**, on the AKS-managed NIC NSG (both subnet and NIC NSGs must allow, so the tighter one wins) | rendered from `loadBalancerSourceRanges`; subnet NSG in `infra-terraform` network module |
| Identity | **Workload Identity** (keyless) — no static keys/API keys anywhere; key-based auth is **disabled at the resource**: `local_auth_enabled=false` on AI Services, `shared_access_key_enabled=false` on the Foundry storage account | `infra-terraform` azure-ai module |
| Identity | **Least privilege split**: app identity (OpenAI + AI Developer + KV) vs secrets-only identity (KV read) for secret-sync | azure-ai module + monitoring/cert-manager/puzzle SA + SecretProviderClass |
| Identity | Automation account scoped to the **AKS cluster** with a custom start/stop-only role (not `Contributor`, which grants `listClusterAdminCredential`) | `infra-terraform/automation.tf` |
| Secrets | All secrets in **Azure Key Vault**, materialised via AKV CSI driver; no plaintext in git | `*/secret-provider-class.yaml`, `.gitignore` |
| Patching | AKS `automatic_upgrade_channel=patch` + `node_os_upgrade_channel=NodeImage`, Sunday maintenance windows inside the running period | `infra-terraform` aks module |
| Network | API and Postgres pods have **egress** policies (default-deny outbound; DNS, Postgres and public HTTPS only, RFC1918 + link-local excluded) | `manifests/news-digest/networkpolicy.yaml` |
| Workload | Pods run **non-root** (uid 1000), `allowPrivilegeEscalation:false`, `capabilities.drop:[ALL]`, `seccompProfile:RuntimeDefault`, CPU/mem limits | all Deployments/StatefulSets/CronJobs |
| Workload | **HTTP security headers** on every response: CSP, HSTS, X-Frame-Options, X-Content-Type-Options, Referrer-Policy, Permissions-Policy | `manifests/news-digest/config/frontend-configmap.yaml` (frontend-nginx) |
| App | Article URLs validated at **both** ends — the aggregator drops any URL the model did not take from the candidate pool, the frontend allows only `http(s)` in an `href` | `config/aggregator-configmap.yaml`, `config/frontend-configmap.yaml` |
| Cost | Paid TTS path guarded: single-flight on cache misses, global synthesis semaphore, per-client budget on `/api/audio` | `config/api-configmap.yaml` |
| GitOps | Argo CD runs `--insecure` behind Gateway TLS; UI at `/argocd`, edge-protected by Cloudflare + WAF | `infra-terraform` argocd module, `manifests/argocd/` |

## Notes & intentional exceptions

- **Frontend is public by design.** It is the app's web entry point and has no
  NetworkPolicy; it is protected at the edge (Cloudflare proxy + the origin lock
  above). An in-cluster ipBlock to the ingress source proved unreliable under
  Cilium, so the sensitive tiers (Postgres, API) carry the identity-based policies
  instead. Now that Envoy enters from inside the pod network, a podSelector policy
  for the frontend is feasible if ever wanted.
- **The Azure WAF was never in force.** Until 2026-08-07 this document claimed an
  Azure WAF policy (OWASP 3.2, Prevention) plus an AGC origin lock. The policy
  existed in Azure and the `WebApplicationFirewallPolicy` CR targeted the Gateway
  correctly, but the ALB controller never programmed it — the CR sat at
  `Deployment=False` / `reason=NoDeployment` for 52 days, and no WAF meter ever
  appeared on the bill. Neither the ruleset nor the origin lock was ever applied.
  Both roles now sit at Cloudflare and in the NSG-backed source ranges, which are
  verifiable: `az network nsg rule list` shows the Cloudflare prefixes, and a
  request to the origin IP from a non-Cloudflare address gets no response.
- **NetworkPolicy enforcement requires the Cilium dataplane.** Without it (the
  pre-2026-06-16 state) every NetworkPolicy is a silent no-op. Verify with
  `az aks show … --query networkProfile.networkPolicy` (should be `cilium`).
- **Argo CD local `admin`** is still enabled (disabling needs SSO → Entra, deferred).
  An admin-IP allowlist, if wanted, belongs at **Cloudflare** (it sees the real
  client IP; the origin only ever sees the Cloudflare edge IP).

## Verify the posture

```bash
az aks show -g rg-kafka-dev-swc-002 -n aks-kafka-dev-swc-002 --query networkProfile.networkPolicy -o tsv   # cilium
kubectl get networkpolicy -n news-digest                 # postgres-ingress, api-ingress
curl -s -m 8 --resolve dailydoseoftech.org:443:4.165.129.12 https://dailydoseoftech.org/ || echo "origin locked"   # direct, non-Cloudflare: no answer
kubectl get certificate -n news-digest                   # dailydoseoftech-tls Ready
curl -sI http://dailydoseoftech.org | head -1            # 301 -> https
```

## Not yet hardened (backlog — see `infra-terraform/whatnext.md`)

- Key Vault network ACLs + purge protection (#7)
- AKS diagnostic/audit logs → Log Analytics (#12)
- Prebuilt, scanned images in ACR (digest-pinned) + `readOnlyRootFilesystem` (#10)
- PostgreSQL TLS (`sslmode=require`) (#13)
- AKS API-server authorized IP ranges / Entra RBAC + `disableLocalAccounts` — see S1 below
- Defender for Containers + Azure Policy (Pod Security Standards: restricted) — see S5 below

## Open from the 2026-09-30 review

Live review after the move to the CNS DEV FROZ subscription, checked against the
running deployment (not just the manifests). Items already listed above are not
repeated.

| # | Sev | Finding | Fix | Status |
| - | --- | ------- | --- | ------ |
| S1 | 🔴 | **AKS API server open to the internet with a local admin certificate.** No authorized IP ranges, no Entra ID integration, `disableLocalAccounts=false`. The cluster-admin client cert lives in Terraform state and `~/.kube/config` and cannot be revoked short of rotating cluster certificates. | Quick: `api_server_authorized_ip_ranges` (infra-terraform). Proper: Entra ID + Azure RBAC for Kubernetes, local accounts disabled, Terraform Helm/Kubernetes providers via kubelogin. | **quick fix done 2026-09-30** — allowlisted to the operator egress IP (`217.16.110.58/32`); Entra step blocked (no Entra ID access) |
| S2 | 🔴 | **Argo CD login is public, admin-only, initial password never rotated.** `argocd-initial-admin-secret` still present, no MFA. Argo CD access is effectively cluster-admin. | Rotate admin password + delete the initial secret; **Cloudflare Access** (Zero Trust, free tier) in front of `/argocd` and `/grafana` for SSO + MFA at the edge. | **rotation done 2026-09-30** — new password in Key Vault `argocd-admin-password`, initial secret deleted, old password verified rejected; Cloudflare Access open (dashboard) |
| S3 | 🔴 | **Cloudflare API token over-privileged and over-distributed.** The DNS-edit token (needed only by cert-manager DNS-01) is also mounted into `monitoring` for the `cf-analytics` job, which needs Analytics:Read only. A compromised monitoring pod could repoint `dailydoseoftech.org` or issue certificates for it. | Separate read-only analytics token in Key Vault for `cf-analytics`; DNS-edit token stays in cert-manager only. | open (needs a new token from the Cloudflare dashboard) |
| S4 | 🟠 | **`/grafana/metrics` publicly readable** — ~130 KB of Grafana internals, unauthenticated. | Gateway rule answering 404 on that path. Prometheus scrapes Grafana in-cluster via its ServiceMonitor, so monitoring is unaffected. | **fixed 2026-09-30** (`grafana-httproute.yaml`). Also fixed the Grafana ServiceMonitor, which scraped `/metrics` and was always down (403) — Grafana serves it under `/grafana` |
| S5 | 🟠 | **No detection or audit trail.** No AKS diagnostic settings / audit logs, no Defender for Containers, no Azure Policy add-on. An exploit of S1–S3 would leave no record. | AKS `kube-audit-admin` → storage account (cheap); Defender for Containers (~EUR 13/month for this node); Azure Policy add-on (PSS restricted, audit mode first). | open (cost decision) |
| S6 | 🟠 | **Postgres volume on an AKS-auto-created storage account** (`f92424a5…` in the node RG) with shared-key access and public network access enabled — Azure Files (SMB) requires the key. Extends the Azure Files finding above. | Resolved by the Azure Disk cutover, which is blocked by the node's 4-data-disk limit; alternatively restrict the account's network access to the AKS subnet. | open |
| S7 | 🟡 | **Supply chain:** 45 image references not digest-pinned; API + both aggregator CronJobs `pip install` from PyPI on every start (version-pinned, not hash-pinned). | Prebuilt images in ACR, digest-pinned (backlog #10); interim: `pip install --require-hashes`. | open |
| S8 | ⚪ | Unused `newsapi-key` still stored in Key Vault; `terraform.tfvars` on the operator workstation holds every secret in plaintext (gitignored). | Remove the unused secret; keep tfvars out of synced folders/backups. | open |
| S9 | ⚪ | **Unverified:** Cloudflare SSL/TLS mode (Full strict) and WAF rules — the API token cannot read zone settings. | Check in the Cloudflare dashboard. | open |
| S10 | ⚪ | No NetworkPolicies in `monitoring`, `argocd`, `cert-manager`, `envoy-gateway-system`. | Default-deny + explicit allows per namespace. | open |

## Open from the 2026-09-03 review

Full write-up in [`20260903_security_findings.md`](20260903_security_findings.md).
The Terraform side is **applied and verified** against Azure; state has been moved
to an Entra-only blob backend with versioning and soft delete, and the local
plaintext copies were shredded.

Remaining, both in this repo and both deliberately staged rather than auto-synced:

- **Postgres runs on Azure Files (SMB)**, which PostgreSQL does not support, with
  `reclaimPolicy: Delete` and no backup. Replacement class and cutover procedure
  are prepared in `manifests/news-digest/postgres/storageclass-managed.yaml`; the
  migration itself is a manual dump/restore.
- **All Argo Applications still use the `default` AppProject.** A scoped project is
  drafted in `manifests/argocd/appproject.yaml` with adoption steps; switching the
  Applications over should be done with the cluster up.
