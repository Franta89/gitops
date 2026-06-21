# AI Services

How the **Daily Dose of Tech** (ddot) app uses Azure AI in this cluster: which
service, how it is configured, how it authenticates, and where it sits in the
overall pipeline. This is the AI counterpart to [`SECURITY.md`](SECURITY.md) and
[`MONITORING.md`](MONITORING.md).

> **Dev/test sandbox only — NOT production.** The cluster, the AI Services
> account, the Key Vault and the managed identity are all provisioned by the
> companion `infra-terraform` repo; nothing here creates Azure resources.

## What we use

| Item | Value |
| --- | --- |
| Service | **Azure AI Services** account (`ais-ddot-dev-swc-001`) |
| Model / deployment | **GPT-5.4-mini** (`gpt-5.4-mini`) |
| API surface | Azure OpenAI **v1 API** — `https://ais-ddot-dev-swc-001.openai.azure.com/openai/v1/` |
| Client library | stock `openai` Python SDK (`openai~=1.57.4`), `OpenAI` class |
| Auth | Azure **Workload Identity** (keyless) → Entra ID bearer token |
| Provisioned but NOT in the inference path | AI Foundry Hub |

The AI Foundry Hub exists (Terraform creates it) but the app **does not** call
through it. All inference goes straight to the AI Services account's v1 API.
There is no Hub connection, no model-router, and no key-based access.

## Why these choices

- **v1 API, not a dated `api-version`.** The base URL ends in `/openai/v1/`. The
  v1 API drops the dated `api-version` query parameter, which avoids
  preview/dated-route breakage when Azure rotates API versions. As a result
  there is no `OPENAI_API_VERSION` anywhere in the config.
- **Stock `OpenAI` client, not `AzureOpenAI`.** Because we target the v1 base URL
  and pass an Entra bearer token as the `api_key`, the plain `OpenAI` client
  works directly — no Azure-specific client wrapper needed.
- **Keyless.** No API keys are stored or rotated for AI. The pod's managed
  identity mints a short-lived Entra token per run. This matches the repo's
  "no key-based external APIs" constraint (see [`CLAUDE.md`](CLAUDE.md) →
  Key decisions).
- **GPT-5.4-mini.** A small, fast, inexpensive model is sufficient: the workload
  is classification and bounded summarisation over a few hundred headlines once
  a day, not interactive chat.

## What the AI does

The AI is used **only by the aggregator** (`manifests/news-digest/config/aggregator-configmap.yaml`),
which runs as two CronJobs. The FastAPI backend and the nginx frontend do **not**
call AI — they only read pre-computed results from PostgreSQL.

There are **three distinct AI call sites**, all `chat.completions.create` against
`gpt-5.4-mini`:

1. **`classify_articles()` — ambiguous tech routing (daily).**
   Most tech stories are routed to an area (Cloud / AI / Security / Financial) in
   pure Python by entity regex and vendor-domain rules (`route_tech()`), with **no
   AI call**. Only the leftover ambiguous items (matched both/neither AI and cloud
   patterns) are sent to the model in **one batched call** that returns a single
   area per headline as JSON. `temperature=0.0` for deterministic routing.

2. **`summarize_area()` — per-area selection + summary (daily).**
   For each of the six areas the model receives the de-duplicated, source-diversified
   candidate pool and an area-specific prompt (persona, what to select, what to
   strictly exclude, how to summarise, the European angle). It returns ~5 ranked
   articles with English **and** Czech summaries, an overview paragraph (EN + CS),
   and an `is_european` flag per item. `temperature=0.2`, `max_completion_tokens=4000`.
   This is the bulk of the AI usage: roughly one call per area per day.

3. **`generate_monthly_summary()` — monthly roll-up (1st of month).**
   Aggregates the month's stored digests and asks the model for one cross-area
   technical summary. `temperature=0.4`, `max_completion_tokens=2000`.

### Areas

Cloud Computing · AI Development · IT Security · Financial Markets · World News ·
Euro News. Categorisation depends on what a story is **about** (entity routing +
the classifier call), not which feed carried it. Full prompt and routing rationale
live in `aggregator.py` and the project `README.md`.

## Configuration

Two settings drive the AI path, both in
[`manifests/news-digest/config/settings-configmap.yaml`](manifests/news-digest/config/settings-configmap.yaml)
(`ConfigMap` `news-digest-settings`), injected into the aggregator pods via
`envFrom`:

```yaml
OPENAI_ENDPOINT:   "https://ais-ddot-dev-swc-001.openai.azure.com/openai/v1/"
OPENAI_DEPLOYMENT: "gpt-5.4-mini"
```

- `OPENAI_ENDPOINT` — the v1 API base URL (must end in `/openai/v1/`). Passed as
  `base_url` to the `OpenAI` client.
- `OPENAI_DEPLOYMENT` — the deployment name, passed as `model=` on every call.

Both values come from Terraform outputs after `terraform apply` in
`infra-terraform` (see the TODO checklist in [`CLAUDE.md`](CLAUDE.md)). There is
**no** `OPENAI_API_VERSION` and **no** AI API key — by design.

## Authentication (keyless, Workload Identity)

```text
CronJob pod (label azure.workload.identity/use: "true")
  └─ ServiceAccount  news-digest-sa
       annotation azure.workload.identity/client-id: <managed_identity_client_id>
          └─ federated credential trusts the AKS OIDC issuer
               └─ WorkloadIdentityCredential mints an Entra token for
                    scope  https://cognitiveservices.azure.com/.default
                      └─ token passed as api_key to the OpenAI client
                           └─ AI Services account authorises via Azure RBAC
```

In code (`aggregator.py`):

```python
class _TokenProvider:
    def __init__(self):
        self._cred = WorkloadIdentityCredential()
    def __call__(self) -> str:
        return self._cred.get_token(
            "https://cognitiveservices.azure.com/.default"
        ).token

client = OpenAI(base_url=OPENAI_ENDPOINT, api_key=_TokenProvider()())
```

The token is fetched once at job start and is valid for the lifetime of the
short-lived job. The managed identity (`managed_identity_client_id`) must hold a
data-plane role (e.g. **Cognitive Services OpenAI User**) on the AI Services
account — granted in `infra-terraform`.

> Note: the AI auth uses the **managed identity**, not an API key, so it does NOT
> flow through the Azure Key Vault CSI driver. Key Vault is only used for the
> Postgres password / database URL (`news-digest-akv` `SecretProviderClass`).
>
> **Which identity?** The workload identity is a **dedicated user-assigned
> managed identity** (`id-ddot-dev-swc-001`, client ID `a3f25662-…`) created in
> `infra-terraform` for this app alone — it is **not** the AKS cluster's own
> kubelet/node-pool identity (`aks-kafka-dev-swc-001-agentpool`). This is the
> point of Workload Identity: ddot pods authenticate with their own
> least-privilege identity (scoped to AI Services + Key Vault) instead of
> inheriting the broad cluster identity. The `id-` prefix vs `…-agentpool` is how
> Azure distinguishes a purpose-built workload identity from the auto-generated
> cluster one. A **federated credential** on `id-ddot-dev-swc-001` trusts the AKS
> OIDC issuer for subject `system:serviceaccount:news-digest:news-digest-sa`, so
> only pods running as `news-digest-sa` can mint its token.

## Where it fits in the overall solution

```text
RSS/Atom feeds ──▶ aggregator CronJob (python:3.12-slim, news-digest ns)
                     │  1. fetch + dedup + source-diversify candidates
                     │  2. route_tech()         ← pure Python, no AI
                     │  3. classify_articles()  ← AI call (ambiguous only)
                     │  4. summarize_area() x6   ← AI calls (Azure AI Services)
                     ▼
                 PostgreSQL (puzzle/news-digest)  ← stores digests + summaries
                     ▲
              FastAPI api  ── reads ──┘   (NO AI calls)
                     ▲
              nginx frontend ── serves ──┘  (static assets, NO AI calls)
                     ▲
        Gateway API (AGC) / Cloudflare ──▶ https://dailydoseoftech.org
```

- AI runs **once a day** (daily digest) plus **once a month** (monthly summary),
  in detached, short-lived batch jobs — never on the request path.
- The user-facing API and frontend only read AI-produced rows from Postgres, so a
  transient AI outage never takes the site down; the previous day's digest stays
  served.
- Each area is isolated: a failure in one area's AI call is logged and skipped
  without aborting the others (see the per-area `try/except` in `main()`).

## Cost & footprint

- Model: GPT-5.4-mini (small/cheap tier).
- Volume: ~1 classification call + 6 summarisation calls per day, plus 1 monthly
  call. `max_completion_tokens` capped at 4000 (daily) / 2000 (monthly).
- Resource footprint of the job itself: requests 100m CPU / 256Mi, limits
  500m CPU / 512Mi (see `cronjobs/daily.yaml`). Note the single-node CPU pressure
  documented in project memory when scheduling alongside other workloads.

## Operations & troubleshooting

Run the daily aggregator on demand and watch the AI calls:

```bash
kubectl create job --from=cronjob/news-digest-daily ddot-manual -n news-digest
kubectl logs job/ddot-manual -n news-digest -f
```

What to look for in the logs:

- `Tech pool N: routed {...}, ambiguous M` — routing before any AI call.
- `After classify: {...}` — the `classify_articles()` AI call landed items into areas.
- `Area <name> done — N items written` — `summarize_area()` succeeded for that area.
- `AI response for <area> hit the token limit` — JSON may be truncated; lower the
  candidate count or raise `max_completion_tokens`.
- `Classification call failed` / `Area <area> failed` — inspect the stack trace.

Common issues:

- **401 / token errors** → the managed identity lacks the data-plane RBAC role on
  the AI Services account, or the federated credential / SA annotation
  (`azure.workload.identity/client-id`) is wrong. Confirm `news-digest-sa` is
  annotated and the pod carries `azure.workload.identity/use: "true"`.
- **404 / wrong route** → `OPENAI_ENDPOINT` is not the v1 base URL (must end in
  `/openai/v1/`), or `OPENAI_DEPLOYMENT` doesn't match the deployed model name.
- **Empty digest** → check RSS retrieval logs first; an area with zero candidates
  is skipped before any AI call is made.

## Related docs

- [`CLAUDE.md`](CLAUDE.md) — repo overview, AI auth & endpoint decisions, TODO checklist.
- [`README.md`](README.md) → "Daily Dose of Tech" — feed list, look-back windows, AI selection logic.
- `infra-terraform` — provisions the AI Services account, GPT-5.4-mini deployment, AI Foundry Hub, managed identity, and the RBAC role assignment.
</content>
</invoke>
