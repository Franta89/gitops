# Security & robustness findings — 2026-09-03

Full-stack review of both repos (`gitops` + `infra-terraform`): Terraform/Azure,
Kubernetes manifests, and the application code embedded in ConfigMaps.

Findings already tracked in [`SECURITY.md`](SECURITY.md) and
`infra-terraform/whatnext.md` are **not** repeated here — this document lists only
what those two did not already cover.

Severity: 🔴 critical · 🟠 high · 🟡 medium · ⚪ low

| # | Sev | Location | Issue | Status |
| - | --- | -------- | ----- | ------ |
| 1 | 🔴 | `infra-terraform/terraform.tfstate` (+2 backups) | Unencrypted local state holds live secrets | fixed |
| 2 | 🔴 | `apps/postgresql.yaml` | Hardcoded Postgres password in git | fixed |
| 3 | 🔴 | `manifests/news-digest/postgres/` | Postgres on Azure Files + `reclaimPolicy: Delete` + no backup | prepared |
| 4 | 🟠 | `config/aggregator-configmap.yaml`, `config/frontend-configmap.yaml` | Unvalidated LLM/RSS URL rendered into `href` → stored XSS | fixed |
| 5 | 🟠 | `infra-terraform/automation.tf` | Automation identity is cluster Contributor → cluster-admin | fixed |
| 6 | 🟠 | `infra-terraform/modules/azure-ai/main.tf` | Key-based auth still enabled on AI Services + storage | fixed |
| 7 | 🟠 | `infra-terraform/main.tf` | No budget on `rg-ddot` (the AI spend) | fixed |
| 8 | 🟠 | `config/api-configmap.yaml` | Unauthenticated paid TTS endpoints, no rate limit | fixed |
| 9 | 🟡 | `infra-terraform/modules/aks/main.tf` | No node-OS / cluster auto-upgrade channel | fixed |
| 10 | 🟡 | `config/aggregator-configmap.yaml` | No timeout on OpenAI calls | fixed |
| 11 | 🟡 | `apps/*.yaml`, `bootstrap/root-app.yaml` | Everything in the `default` AppProject | drafted |
| 12 | 🟡 | `manifests/kafka/kafka.yaml` | Plaintext listener, no authn/authz, RF=1 | accepted |
| 13 | ⚪ | `manifests/news-digest/networkpolicy.yaml` | Egress unrestricted | fixed |
| 14 | ⚪ | `infra-terraform/versions.tf` | Floating provider versions | fixed |
| 15 | ⚪ | `postgres/statefulset.yaml`, `terraform.tfvars.example` | No liveness probe; example missing a variable | fixed |

Verified clean: no secrets in git history (`bing-secret.yaml` was a placeholder),
SQL is parameterized, no `eval`/`shell=True`/`pickle`, no `verify=False`,
CORS scoped to the two real origins.

---

## 1. 🔴 Unencrypted local Terraform state containing live secrets

`infra-terraform/backend.tf` still has the `azurerm` backend commented out, so state
is local. Three files exist:

```text
terraform.tfstate
terraform.tfstate.backup
terraform.tfstate.1781271633.backup
```

They contain, in plaintext: `postgres_password`, `cloudflare_api_token`,
`newsapi_key`, `grafana_admin_password`, and the AKS `kube_config` **client key**
(cluster-admin credential).

They are correctly gitignored and *not* in git history (verified), but they sit
unencrypted on a workstation with no locking, no versioning, and no recovery path
if the disk is lost.

**Status: fixed and verified.** State now lives in Azure Blob Storage
(`rg-tfstate-dev-swc-001` / `stkafkastatedevswc001`, container `tfstate`).

The state account is deliberately harder than a default one, because this
particular blob holds a cluster-admin credential:

- **shared key access disabled** — the only way in is an Entra identity holding
  `Storage Blob Data Contributor`. This is why `backend.tf` sets
  `use_azuread_auth = true` and `providers.tf` sets `storage_use_azuread = true`.
- **blob versioning on** — a truncated or corrupted state can be rolled back.
- **30-day soft delete** on blobs and containers — an accidental delete is
  recoverable, which plain local state never was.
- TLS 1.2 minimum, HTTPS-only, no public blob access.

Migration was verified before anything was deleted: 50 resources in, 50 resources
out, and the uploaded blob matched the local file byte-for-byte (155,873 bytes).
Only then were `terraform.tfstate`, both backups, the saved plan files and the
temporary copies **overwritten with random bytes and unlinked**. `terraform state
list` now reads 54 resources straight from the blob with no local state present.

`bootstrap.sh` was rewritten to reproduce this exact account, including the RBAC
grant and the propagation retry loop it needs. `.gitignore` also gained
`*.tfplan` / `tfplan*.binary` — saved plans embed full resource values including
secrets and were previously not ignored.

## 2. 🔴 Hardcoded Postgres password in git

`apps/postgresql.yaml` passed `auth.password: puzzle-dev-changeme` as a Helm value.
Argo CD rendered it into a live Secret, and anyone with repo read access had the
credential.

**Fixed:** the chart now consumes `auth.existingSecret`, pointing at a Secret
materialised from Azure Key Vault by the CSI driver, matching how every other
secret in this repo is handled. New objects: `manifests/puzzle/secret-provider-class.yaml`,
`manifests/puzzle/secret-sa.yaml`, `manifests/puzzle/secret-sync-deployment.yaml`
and `apps/puzzle-secrets.yaml` (sync-wave 2, ahead of postgresql at wave 3);
`puzzle_postgres_password` is a new Terraform variable stored in Key Vault.

> **Rotation caveat.** The bitnami chart only sets a password when it *initialises*
> the data directory. Pointing an existing database at a new secret does not change
> the role, so the app will fail to authenticate. Either run
> `ALTER USER postgres WITH PASSWORD '<new>';` (and the same for the app role)
> against the running instance to match the Key Vault value, or delete the PVC and
> let the chart re-initialise. The old value `puzzle-dev-changeme` is in git history
> and must be treated as compromised regardless.

## 3. 🔴 Postgres data on Azure Files, deletable, unbacked

`manifests/news-digest/postgres/storageclass.yaml` provisioned the database volume
with `file.csi.azure.com` (SMB). PostgreSQL is not supported on SMB — its fsync and
file-locking assumptions do not hold, which risks silent corruption. The class also
used `reclaimPolicy: Delete`, so with Argo's `prune: true` a bad sync that removes
the PVC destroys the database irrecoverably, and no backup exists.

**Fixed in code, cutover pending.** A replacement class
`manifests/news-digest/postgres/storageclass-managed.yaml` provisions Azure Disk
(`disk.csi.azure.com`, StandardSSD_LRS — the correct backing store for a
single-writer database) with `reclaimPolicy: Retain` and
`volumeBindingMode: WaitForFirstConsumer`. The old class is marked DEPRECATED.

The StatefulSet still points at the old class **on purpose**: both StorageClasses
and `volumeClaimTemplates` are immutable, so editing the reference in place makes
Argo fail the sync rather than migrate the data. The switch is a `pg_dump` →
delete PVC → restore sequence; the six steps are in the header of
`storageclass-managed.yaml`. Do it with the cluster up and a dump verified
off-cluster first.

## 4. 🟠 Stored XSS via unvalidated article URLs

The aggregator accepted the `url` field straight from the model's JSON output
without checking it against the candidate articles or validating its scheme, then
persisted it. The frontend rendered it as
`'<a href="' + escHtml(item.url) + '">'`, and `escHtml()` escapes HTML
metacharacters but not `javascript:`. A poisoned RSS item (prompt injection) could
therefore land an executable link on the public site.

**Fixed** at both ends:

- aggregator: every returned item must map back to a URL that was actually offered
  as a candidate, and the URL must be `http(s)`; anything else is dropped.
- frontend: a `safeUrl()` helper enforces an `https:`/`http:` allowlist and falls
  back to `#` for anything else.

## 5. 🟠 Automation identity effectively cluster-admin

`automation.tf` granted `Contributor` on the AKS cluster resource so a runbook could
start and stop it. `Contributor` includes
`Microsoft.ContainerService/managedClusters/listClusterAdminCredential/action`,
so that identity could mint an admin kubeconfig and take over the cluster.

**Fixed:** a custom role definition (`AKS Start Stop Operator`) grants exactly
`.../managedClusters/start/action`, `.../stop/action` and `read`, and the role
assignment now uses it.

## 6. 🟠 Key-based auth still enabled on AI Services and storage

`SECURITY.md` claims "no static keys/API keys anywhere", but the Azure AI Services
account still accepted its API keys and the Foundry storage account still accepted
its shared account keys — both bypass the Workload Identity RBAC model entirely.

**Fixed:** `local_auth_enabled = false` on the cognitive account;
`shared_access_key_enabled = false`, `https_traffic_only_enabled = true`,
`min_tls_version = "TLS1_2"`, and `allow_nested_items_to_be_public = false` on the
storage account.

## 7. 🟠 No budget on the resource group that actually spends

The only `azurerm_consumption_budget_resource_group` targeted `rg-kafka-*`. Every
variable-cost resource — AI Services inference, Foundry Hub, storage, Key Vault —
lives in `rg-ddot-*`, which had no budget and no alert at all. The AGC bill that
forced the 2026-08-07 migration was discovered the same way: too late.

**Fixed:** an equivalent budget with 80 % actual / 100 % forecast notifications now
covers `rg-ddot`, sized by the new `ddot_budget_alert_amount` variable.

## 8. 🟠 Unauthenticated, unthrottled paid TTS endpoints

`/api/audio` synthesises speech through a paid Azure service and writes to blob
storage. It is public, unauthenticated and unthrottled, so a script can drive
inference cost and storage writes at will. Caching helps only after the first hit;
concurrent misses still fan out.

**Fixed:** a lightweight in-process rate limiter caps audio requests per client IP,
and a global concurrency semaphore bounds simultaneous synthesis calls. Durable
protection still belongs at Cloudflare (it sees the real client IP).

## 9. 🟡 No automatic patching of the cluster

`modules/aks/main.tf` pinned nothing and enabled no upgrade channel, so node images
never received OS security patches and the control plane never moved off whatever
version it was created with.

**Fixed:** `node_os_upgrade_channel = "NodeImage"` plus
`automatic_upgrade_channel = "patch"`, with a maintenance window aligned to the
existing 06:00–20:00 start/stop schedule.

## 10. 🟡 No timeout on OpenAI calls

The `OpenAI` client was constructed without a timeout, so a stalled connection could
hang the daily CronJob indefinitely.

**Fixed:** explicit `timeout` and `max_retries` on the client.

## 11. 🟡 Every Argo CD Application in the `default` project

`bootstrap/root-app.yaml` and all of `apps/` use `project: default`, which permits
any repo, any destination and any resource kind. Combined with `prune: true`,
`selfHeal: true`, a mutable `targetRevision: main`, and an Argo UI that is publicly
reachable at `/argocd` with the local `admin` account, a single compromised commit
or session reaches the whole cluster.

**Status: drafted, not adopted.** `manifests/argocd/appproject.yaml` defines a `ddot`
project restricted to the six repos this stack actually deploys from and the seven
namespaces it actually targets. Resource kinds are left unrestricted on purpose:
cert-manager, Strimzi, Envoy Gateway and kube-prometheus-stack all legitimately
create `ClusterRoleBinding`s and admission webhooks, so a kind blacklist would break
syncs without stopping a real attacker. The restriction value is in `sourceRepos`
and `destinations`.

Nothing sets `project: ddot` yet — flipping every Application at once changes the
sync path for the whole cluster, so it should be done in a maintenance window with
the cluster up, one app at a time. Adoption steps are in the file header.

## 12. 🟡 Kafka has no TLS, no auth, and RF=1

`manifests/kafka/kafka.yaml` exposes a plaintext listener, configures no
`authentication` or `authorization`, and runs a single broker with
`min.insync.replicas: 1`.

**Status: accepted.** This is the documented sandbox topology (`CLAUDE.md`:
"Default topology = 1 dual-role node, replication factors 1"). It is recorded here
because it is a hard blocker for any promotion beyond dev/test, not because it is a
defect in the current context.

## 13. ⚪ NetworkPolicies restrict ingress only

`networkpolicy.yaml` set `policyTypes: [Ingress]`, leaving egress unrestricted for
Postgres and API pods — so a compromised pod could reach anything, including the
cluster's own control plane and arbitrary internet hosts.

**Fixed:** the Postgres policy now denies egress except DNS. The API keeps broad
egress (it legitimately calls Azure AI, Speech and blob storage over the internet)
but is no longer permitted to reach the rest of the cluster implicitly.

## 14. ⚪ Floating provider versions

`versions.tf` used `~> 4.0` / `~> 2.13` / `~> 2.30`, contradicting the repo's own
standard ("Pin all provider versions in `versions.tf`. No floating versions.").

**Fixed:** pinned to the exact versions already recorded in `.terraform.lock.hcl`
(azurerm 4.76.0, helm 2.17.0, kubernetes 2.38.0).

## 15. ⚪ Smaller gaps

- `postgres/statefulset.yaml` had a readiness probe but no liveness probe, so a
  wedged postmaster would never be restarted. **Fixed.**
- `terraform.tfvars.example` was missing `grafana_admin_password`, so a fresh clone
  fails at plan time with an unhelpful prompt. **Fixed.**

---

## Addendum — issues found while applying (2026-09-03)

Running a real `terraform plan`/`apply` against Azure surfaced three defects that
static validation could not. All are fixed.

### A. Redundant `depends_on` forced needless role-assignment replacement

`module "azure_ai"` carried `depends_on = [module.aks]`. That is redundant — the
module already consumes `module.aks.oidc_issuer_url`, which orders it implicitly —
but it is not harmless: an explicit module-level `depends_on` defers **every data
source in the module** to apply time whenever anything in `module.aks` changes.

Because this review modified the AKS resource, `data.azurerm_client_config.current`
became unknown at plan time, so `object_id` and `tenant_id` were unknown, which
forced replacement of `azurerm_role_assignment.terraform_kv_admin` — the very grant
that authorises writing the new `puzzle-postgres-password` secret in the same
apply. That is a plausible mid-apply 403.

Removing the redundant `depends_on` took the plan from
`7 add / 4 change / 3 destroy` to `5 add / 3 change / 1 destroy`.

### B. Budget `start_date` goes stale and breaks fresh applies

Azure rejects a monthly budget whose `start_date` precedes the current month:

```text
400: Start date for monthly time grain should not be prior to current month.
```

`var.budget_start_date` is the fixed literal `2026-06-01T00:00:00Z`, so the new
`rg-ddot` budget failed to create. The existing `rg-kafka` budget only survives
because it was created back when that date was valid — any fresh apply of this
configuration would fail the same way.

The ddot budget now anchors to the first of the current month via
`formatdate("YYYY-MM-01'T'00:00:00'Z'", timestamp())` with
`lifecycle { ignore_changes = [time_period] }`, so it is valid on creation and does
not churn every month afterwards. The kafka budget was deliberately left alone —
it is already created and working, and rewriting its start date risks the same 400.

### C. Disabling storage shared keys broke the provider's own read-back

With `shared_access_key_enabled = false`, the apply succeeded but the post-apply
refresh failed:

```text
403 Key based authentication is not permitted on this storage account.
```

The azurerm provider was still using key auth to read queue/table properties.
Fixed with `storage_use_azuread = true` in the provider block. Note the setting had
already taken effect on the account — only the read-back failed — so this presented
as a confusing partial failure rather than a rollback.

### Post-apply verification

| Check | Result |
| --- | --- |
| `terraform plan -detailed-exitcode` | `0` — no drift |
| Role assignments on the AKS cluster | only `AKS Start Stop Operator (kafka-dev-swc-001)`; Contributor gone |
| AI Services `disableLocalAuth` | `true` |
| Foundry storage `allowSharedKeyAccess` | `false` |
| AKS upgrade channels | `patch` / `NodeImage`, cluster `Running` |
| Budgets | `budget-kafka-*` €50, `budget-ddot-*` €30 |
| `puzzle-postgres-password` in Key Vault | present, enabled |
| `GET /` | 200 |
| `GET /api/health`, `/api/digest/today` | 200, 54 KB of content |
| `GET /api/audio?...&category=ai_development` | **200, 1.8 MB `audio/mpeg`** |

The audio check is the meaningful one: it exercises Azure Speech synthesis *and*
the blob audio cache end-to-end, proving that disabling local auth on AI Services
and shared keys on the storage account did not break the keyless path.
