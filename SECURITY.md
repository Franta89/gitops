# Security posture — Daily Dose of Tech / AKS

How this cluster is hardened, layer by layer. **Dev/test sandbox** (not production),
but hardened with production patterns. Azure-side controls live in the companion
`infra-terraform` repo; in-cluster controls live here (gitops). Forward-looking
backlog is tracked in `infra-terraform/whatnext.md`.

## Defense in depth (summary)

| Layer | Control | Where |
| ----- | ------- | ----- |
| Edge | Cloudflare proxy (orange-cloud), SSL/TLS **Full (strict)** | Cloudflare dashboard |
| Edge | **WAF** — Azure WAF policy, OWASP 3.2, **Prevention** mode | `infra-terraform` module `app-gateway-containers` + `manifests/news-digest/waf.yaml` |
| Edge | **Origin lock** — AGC accepts only Cloudflare edge IPs (WAF custom rule) | same WAF policy |
| Ingress | **Gateway API** via Azure Application Gateway for Containers (ingress-nginx retired) | `apps/alb-controller.yaml`, `manifests/news-digest/gateway.yaml` |
| Ingress | TLS — Let's Encrypt via **DNS-01** (Cloudflare), HTTP→HTTPS 301 redirect | `manifests/news-digest/certificate.yaml`, `httproute.yaml` |
| Network | **NetworkPolicy enforced** (Cilium dataplane) | `infra-terraform` AKS module (`network_policy=cilium`) |
| Network | Postgres reachable only from API/worker pods; API only from frontend (identity-based) | `manifests/news-digest/networkpolicy.yaml` |
| Network | NSG: **no public inbound** to nodes (default DenyAllInBound); AGC enters via delegated subnet over the VNet | `infra-terraform` network module |
| Identity | **Workload Identity** (keyless) — no static keys/API keys anywhere | `infra-terraform` azure-ai module |
| Identity | **Least privilege split**: app identity (OpenAI + AI Developer + KV) vs secrets-only identity (KV read) for secret-sync | azure-ai module + monitoring/cert-manager SA + SecretProviderClass |
| Identity | Automation account scoped to the **AKS cluster**, not the resource group | `infra-terraform/automation.tf` |
| Secrets | All secrets in **Azure Key Vault**, materialised via AKV CSI driver; no plaintext in git | `*/secret-provider-class.yaml`, `.gitignore` |
| Workload | Pods run **non-root** (uid 1000), `allowPrivilegeEscalation:false`, `capabilities.drop:[ALL]`, `seccompProfile:RuntimeDefault`, CPU/mem limits | all Deployments/StatefulSets/CronJobs |
| Workload | **HTTP security headers** on every response: CSP, HSTS, X-Frame-Options, X-Content-Type-Options, Referrer-Policy, Permissions-Policy | `manifests/news-digest/config/frontend-configmap.yaml` (frontend-nginx) |
| GitOps | Argo CD runs `--insecure` behind Gateway TLS; UI at `/argocd`, edge-protected by Cloudflare + WAF | `infra-terraform` argocd module, `manifests/argocd/` |

## Notes & intentional exceptions

- **Frontend is public by design.** It is the app's web entry point and has no
  NetworkPolicy; it is protected at the edge (Cloudflare proxy + AGC WAF origin
  lock). An in-cluster ipBlock to AGC's source proved unreliable under Cilium, so
  the sensitive tiers (Postgres, API) carry the identity-based policies instead.
- **NetworkPolicy enforcement requires the Cilium dataplane.** Without it (the
  pre-2026-06-16 state) every NetworkPolicy is a silent no-op. Verify with
  `az aks show … --query networkProfile.networkPolicy` (should be `cilium`).
- **Argo CD local `admin`** is still enabled (disabling needs SSO → Entra, deferred).
  An admin-IP allowlist, if wanted, belongs at **Cloudflare** (it sees the real
  client IP; AGC only sees the Cloudflare edge IP).

## Verify the posture

```bash
az aks show -g rg-kafka-dev-swc-001 -n aks-kafka-dev-swc-001 --query networkProfile.networkPolicy -o tsv   # cilium
kubectl get networkpolicy -n news-digest                 # postgres-ingress, api-ingress
kubectl get webapplicationfirewallpolicy -n news-digest  # ddot-waf
kubectl get certificate -n news-digest                   # dailydoseoftech-tls Ready
curl -sI http://dailydoseoftech.org | head -1            # 301 -> https
```

## Not yet hardened (backlog — see `infra-terraform/whatnext.md`)

- Key Vault network ACLs + purge protection (#7)
- AKS diagnostic/audit logs → Log Analytics (#12)
- Prebuilt, scanned images in ACR (digest-pinned) + `readOnlyRootFilesystem` (#10)
- PostgreSQL TLS (`sslmode=require`) (#13)
- AKS API-server authorized IP ranges / Entra RBAC + `disableLocalAccounts` (needs Entra)
- Defender for Containers + Azure Policy (Pod Security Standards: restricted)
