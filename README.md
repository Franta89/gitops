# gitops

Argo CD source of truth for Strimzi + Kafka (KRaft) running on AKS.
The cluster is provisioned by the companion **infra-terraform** repo.

## Order
1. Bring up the cluster + Argo CD via **infra-terraform**.
2. `kubectl apply -f bootstrap/root-app.yaml`
3. Watch: `kubectl -n kafka get pods,kafka,kafkanodepool`

## Argo CD UI
`kubectl -n argocd port-forward svc/argocd-server 8080:443`
Initial admin password:
`kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d`

## Daily Dose of Tech (news app)

Public site: <https://dailydoseoftech.org>. A daily AI-curated news digest across
six areas — Cloud Computing, AI Development, IT Security, Financial Markets, World
News, and **Euro News** (Europe-only). Deployed in sync wave 5 under
`manifests/news-digest/`. **Dev/test sandbox**; content is AI-generated — verify
with primary sources.

### How it works

```text
RSS feeds (by family) → gather/dedup → route_tech() + classify_articles() → per-area select+summarise → PostgreSQL → API → frontend
```

All logic lives in `manifests/news-digest/config/aggregator-configmap.yaml`
(`aggregator.py`), mounted into a `python:3.12-slim` CronJob pod. The API and
frontend only read from PostgreSQL.

**Categorization is content-based, not feed-based** — a story's area is decided by
what it is *about*, so the same feed can feed any area and stories aren't missed
because they sat in the "wrong" feed list. The tech pool (all tech/security feeds +
vendor newsrooms) is routed deterministically by entity — security incidents →
IT Security, valuations/IPOs → Financial Markets, named AI companies/models (Anthropic,
OpenAI, Mistral, Gemini, Cursor…) → AI Development, named cloud providers/infra
(AWS, Azure, Google Cloud, OCI, Kubernetes…) → Cloud Computing — and only the
genuinely ambiguous remainder goes to one `classify_articles()` AI call. Finance,
World, and Euro feeds map to their areas directly.

Areas are processed `it_security → ai_development → cloud_computing →
financial_markets → euro_news → world_news`; a story can appear in only one area
(cross-area + within-area URL/title dedup). World News is global **excluding**
Europe; Euro News is its Europe-only counterpart. In the four topic areas each item
is tagged `is_european` and the frontend lists global stories first, then a "Europe"
divider, then the European ones.

### News sources (RSS/Atom)

News is pulled directly from authoritative RSS/Atom feeds — real-time, keyless,
$0, and compliant (public syndication). Feeds are grouped into **families** (not
per-area lists); routing then assigns each story to an area by content:

| Family | Feeds |
| --- | --- |
| Tech (→ Cloud / AI / Security) | The New Stack, The Register, TechCrunch (+AI), ZDNet, Ars Technica, The Verge, Wired, VentureBeat, CNBC Tech, Silicon UK, Computer Weekly, Sifted, EU-Startups, BleepingComputer, The Hacker News, Krebs, SecurityWeek, Dark Reading, Infosecurity Magazine, **AWS / Google Cloud / Azure / OpenAI newsrooms** |
| Finance (→ Financial Markets) | CNBC (Top + Markets), MarketWatch, Yahoo Finance, Investing.com, BBC, Guardian, Seeking Alpha |
| World (→ World News, global ex-Europe) | BBC, Al Jazeera, Guardian, NPR, CNN |
| Euro (→ Euro News, continental) | Euronews, DW Europe, France24 Europe, Politico Europe, Guardian Europe, BBC Europe |

Vendor newsrooms route deterministically by domain (`DOMAIN_ROUTE`). Look-back:
**72h for the tech pool, 48h for news**. Cross-day dedup (`known_urls`, every URL
ever stored) prevents repeats, so a wider window only adds new candidates. A
broken/stale feed is logged and skipped, never aborting a family.

> Migrated off NewsAPI (June 2026): the free tier was 24h-delayed and its terms
> forbade production/staging use. RSS is real-time, keyless, and compliant.

### AI — how it works

The AI does **not** fetch news (RSS does); its job is selection + writing. Model:
**Azure AI Services GPT-5.4-mini**, called **directly against the AI Services
account's v1 API** (`*.openai.azure.com/openai/v1/`, deployment `gpt-5.4-mini`) —
**not** routed through the Foundry Hub. Uses the stock `OpenAI` client with no
dated `api-version` (the v1 API drops it — avoids dated-route breakage), authenticated
**keylessly** with an Entra ID bearer token from `WorkloadIdentityCredential` (no API key).
Endpoint/deployment are in `config/settings-configmap.yaml`; the Azure resources and
the identity/role chain are provisioned in the **infra-terraform** repo (see its
README → "Azure AI"). The Foundry Hub is provisioned but not in the inference path.

AI calls in `aggregator.py`:

- **Daily — `classify_articles()`** (one call). Resolves only the *ambiguous* tech
  articles that `route_tech()` couldn't place by entity (deterministic routing
  handles the clear majority). Returns each article's area; titles only, so it's cheap.
- **Daily — `summarize_area()`** (one call per area). Given the diversified
  candidate pool (title, source, snippet, url), the model: (1) selects ~5
  most important stories, reserving roughly one slot for the **European** angle
  (continental, not UK-only) and tagging each item `is_european`, (2) writes a 4–6
  sentence summary of each, (3) writes a synthesis overview, (4) translates summaries +
  overview into Czech — returning strict JSON
  `{items:[{rank,title,url,summary,summary_cs,is_european}], overview, overview_cs}`.
  Each area uses a domain-expert **persona** (`AREA_PROMPTS`) — e.g. Security cites
  CVE IDs/CVSS, Financial cites figures and ignores the tech itself. `temperature=0.2`.
- **Monthly — `generate_monthly_summary()`** (1st of month). Aggregates the
  month's stored items per area into a multi-paragraph technical retrospective.
  `temperature=0.4`, `max_tokens=2000`.

**Quality guardrails:** deterministic **entity routing** keeps security incidents,
valuations/IPOs, AI-company, and cloud-provider stories in their correct areas
regardless of the model; source-diversity cap (≤2 articles/domain); a prompt
diversity rule (prefer ≥3 sources, skip routine release posts); cross-area and
within-area dedup (by URL and normalised title); an **empty-area relaxed retry**
(drops the diversity gate so an area never goes blank); per-area `try/except`
isolation; and Europe-aware re-ranking (global first, then European; ranks 1..n).

### Schedule

| Job | CronJob | Schedule (Europe/Prague) |
| --- | --- | --- |
| Daily digest | `news-digest-daily` | `30 6 * * *` (06:30 CET/CEST) |
| Monthly summary | `news-digest-monthly` | `0 6 1 * *` (06:00, 1st of month) |

### Operations

**Manual rerun (rebuild today's digest now):**

```bash
JOB=news-digest-daily-manual-$(date +%H%M%S)
kubectl create job -n news-digest --from=cronjob/news-digest-daily "$JOB"
kubectl wait -n news-digest --for=condition=complete job/$JOB --timeout=360s
```

The daily run **upserts** today's digest, so a rerun replaces today's content.
Re-running repeatedly the same day shrinks the fresh pool (dedup); the relaxed
retry compensates.

> **GitOps gotcha:** if you change a ConfigMap, **push to `main` first** and let
> Argo CD sync it. A manual `kubectl apply` ahead of git is reverted by Argo
> self-heal within ~2 min — a manual rerun would then run the old code.

**Frontend changes:** the frontend assets are two ConfigMaps (`frontend-assets`,
`frontend-nginx`) mounted as **directories** (not subPath). **Static** changes
(`index.html`, `style.css`, `app.js`) propagate to the running pod automatically —
nginx reads those per request, no restart needed. **nginx config** changes
(`frontend-nginx` / `default.conf`, e.g. the security headers) only take effect
after `kubectl rollout restart deployment/frontend -n news-digest`, since nginx
reads its config only at startup. Still bump `app.js?v=N` on `app.js` changes for
browser/CDN cache-busting. The About-page copy and the ASCII architecture diagram
live in `app.js` (`T.en` / `T.cs`, `ARCH_DIAGRAM`) — keep both languages in sync.

**Troubleshooting:**

| Symptom | Likely cause | Action |
| --- | --- | --- |
| Area card empty (overview, no articles) | Thin pool that day | Check logs for `Still 0 items`; usually genuine. Consider adding feeds. |
| Whole area missing from page | No `area_summaries` row | Check logs for that area's failure/traceback; confirm feeds returned items. |
| Job `BackoffLimitExceeded` | Deterministic crash | `kubectl logs` the failed pod (GC'd ~1h); check DB for committed areas. |
| Stale page after a change | Argo not synced / browser cache | Wait for sync; bump `app.js?v=` and hard-refresh. |

Check the DB for a date:

```bash
kubectl exec -n news-digest postgres-0 -c postgres -- sh -c \
 'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -t -c \
  "SELECT a.area, count(n.id) FROM daily_digests d \
   JOIN area_summaries a ON a.digest_id=d.id \
   LEFT JOIN news_items n ON n.area_summary_id=a.id \
   WHERE d.date=CURRENT_DATE GROUP BY a.area ORDER BY a.area;"'
```

### Change history

- **2026-06 — Content-based categorization + Euro News.** Replaced the feed-siloed
  design (each area saw only its own feeds) with topic routing: feeds grouped into
  families, `route_tech()` + `classify_articles()` assign each story to an area by
  what it's about (fixing mis-routing like Anthropic→Cloud and surfacing missed
  stories). Added the **Euro News** area (Europe-only) and segregated World News to
  global-ex-Europe; the four topic areas tag `is_european` and group a "Europe"
  section. Added vendor newsroom feeds (AWS, Google Cloud, Azure, OpenAI). Schema:
  `news_items.is_european`, rank check widened to 1–20.
- **2026-06 — Migrated NewsAPI → RSS.** NewsAPI's free tier was 24h-delayed (a 25h
  window exposed only a ~1h sliver of articles, starving narrow areas) and forbade
  production use. Replaced with authoritative RSS/Atom feeds: real-time, keyless,
  $0, compliant. Removed `NEWSAPI_KEY`, the `newsapi-secret` wiring, and the
  secret from the SecretProviderClass; added `feedparser`.
- **2026-06 — Reliability + quality fixes:** per-area isolation, source
  diversification, empty-area relaxed retry, rank clamp, wider look-back windows,
  and directory-mounted frontend ConfigMaps (auto-propagate, no restart).

## Note
Strimzi chart version and the Kafka/metadataVersion fields are marked TODO —
verify before syncing. See CLAUDE.md.
