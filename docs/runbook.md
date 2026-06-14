# Runbook — Daily Dose of Tech (news-digest)

Operational guide for the news aggregator that powers
<https://dailydoseoftech.org>. Covers how news is sourced and selected, the daily
schedule, manual reruns, and troubleshooting.

> **Dev/test sandbox.** Not production. Content is AI-generated; verify with
> primary sources.

---

## 1. Pipeline overview

```
RSS/Atom feeds  ->  rss_fetch()  ->  dedup  ->  diversify_by_source  ->  AI select+summarise  ->  PostgreSQL  ->  API  ->  frontend
   (per area)       per-feed         by URL     cap per source         GPT-4.1-mini             daily_digests / area_summaries / news_items
```

All logic lives in `manifests/news-digest/config/aggregator-configmap.yaml`
(`aggregator.py`, mounted into a `python:3.12-slim` CronJob pod). The API
(`api-configmap.yaml`) and frontend (`frontend-configmap.yaml`) only read from
PostgreSQL — they never fetch news.

Five areas, processed independently and in this order:
`cloud_computing`, `ai_development`, `it_security`, `financial_markets`, `world_news`.

---

## 2. News sources (RSS/Atom)

News is pulled **directly from authoritative RSS/Atom feeds** — real-time,
keyless, $0, and compliant (public syndication). Feed lists are in `AREA_FEEDS`
in `aggregator.py`. Current sources:

| Area | Feeds |
| --- | --- |
| Cloud Computing | The New Stack, The Register, TechCrunch, ZDNet, Ars Technica |
| AI Development | VentureBeat, TechCrunch (AI), The Verge, Ars Technica, Wired |
| IT Security | BleepingComputer, The Hacker News, Krebs on Security, SecurityWeek, Dark Reading |
| Financial Markets | CNBC, MarketWatch, Yahoo Finance, Investing.com, BBC Business, Guardian Business, Seeking Alpha |
| World News | BBC World, Al Jazeera, Guardian World, NPR, DW, France24, CNN World |

World News and Financial Markets get **wider lists** (7 feeds) because strong
stories there can break anywhere globally.

**Look-back window** (`WINDOW_HOURS`, default `DEFAULT_WINDOW_HOURS = 48`):
`cloud_computing` and `it_security` use **72h** (lower-volume areas need more
candidates); the rest use **48h**. Cross-day dedup (`known_urls`, every URL ever
stored) prevents already-published stories from repeating, so a wider window only
adds genuinely new candidates.

### Adding / changing a feed

1. Edit `AREA_FEEDS` in `aggregator-configmap.yaml`.
2. Confirm the feed URL returns real RSS/Atom (not an HTML page) and recent items.
3. Commit + push to `main`; Argo CD syncs the ConfigMap. New feeds take effect on
   the next CronJob run (or a manual rerun, see §5).

A broken/stale feed is **logged and skipped** — it never aborts an area. Look for
`RSS <feed> failed` warnings in the logs.

---

## 3. AI selection & quality controls

For each area the AI (`summarize_area`) receives the deduped, source-diversified
candidate pool and returns 3–5 ranked items + an area overview (EN and CS). Guard
rails, in order:

- **Source diversity (`diversify_by_source`)** — at most **2 articles per domain**
  (1 per package-index/changelog domain). Stops any single busy feed from
  dominating an area.
- **Prompt diversity rule** — the model is told to prefer ≥3 distinct sources,
  ≤1 item per source, and to skip routine release/changelog posts.
- **Empty-area retry** — if the model returns 0 items, it retries once in
  **relaxed mode** (no dedup, no diversity gate) to surface the single best
  available story. If still 0, the area renders with its overview but no items
  (logged `Still 0 items ... after relaxed retry`).
- **Per-area isolation** — each area runs in its own `try/except`; a failure
  rolls back and continues to the next area (one bad area never kills the run).
- **Rank clamp** — item `rank` is clamped to the DB range `1..5` before insert.

---

## 4. Schedule

| Job | CronJob | Schedule (Europe/Prague) |
| --- | --- | --- |
| Daily digest | `news-digest-daily` | `30 6 * * *` (06:30 CET/CEST) |
| Monthly summary | `news-digest-monthly` | `0 6 1 * *` (06:00, 1st of month) |

`timeZone: Europe/Prague` handles DST automatically. The frontend advertises
"published daily at 07:00 CET".

---

## 5. Manual rerun (rebuild today's digest now)

```bash
JOB=news-digest-daily-manual-$(date +%H%M%S)
kubectl create job -n news-digest --from=cronjob/news-digest-daily "$JOB"
kubectl wait -n news-digest --for=condition=complete job/$JOB --timeout=360s

# Inspect
POD=$(kubectl get pods -n news-digest -l job-name=$JOB -o jsonpath='{.items[0].metadata.name}')
kubectl logs -n news-digest "$POD" | grep -E "RSS total|unique candidates|done —|retrying|Still 0|complete"
```

The daily run **upserts** today's digest, so a rerun replaces today's content.
Re-running repeatedly the same day shrinks the fresh pool (each run adds its URLs
to `known_urls`); the relaxed retry compensates.

> **GitOps gotcha:** if you change a ConfigMap, **push to `main` first** and let
> Argo CD sync it. A manual `kubectl apply` ahead of git is reverted by Argo
> self-heal within ~2 min — a manual rerun would then run the old code. Verify the
> live ConfigMap before triggering:
> `kubectl get cm -n news-digest aggregator-script -o jsonpath='{.data.aggregator\.py}' | grep -c rss_fetch`

---

## 6. Troubleshooting

| Symptom | Likely cause | Action |
| --- | --- | --- |
| An area card is empty (overview, no articles) | Thin pool / no qualifying story that day | Check logs for `Still 0 items`. Usually genuine; consider adding feeds (§2). |
| Whole area missing from the page | No `area_summaries` row written | Check logs for that area's `failed` / traceback; confirm its feeds returned items (`RSS <feed> -> N recent entries`). |
| All areas thin | Many feeds returned 0 in window | Widen `WINDOW_HOURS`, or check for a network/DNS issue from the pod. |
| Job `BackoffLimitExceeded` | Deterministic crash across all retries | `kubectl logs` the failed pod (logs GC after ~1h — grab them fast); check DB for which areas committed. |
| Stale content after a code change | Argo hasn't synced, or browser cache | See §5 GitOps gotcha; bump `app.js?v=` for frontend changes. |

Verify what's in the DB for a date:

```bash
kubectl exec -n news-digest postgres-0 -c postgres -- sh -c \
 'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -t -c \
  "SELECT a.area, count(n.id) FROM daily_digests d \
   JOIN area_summaries a ON a.digest_id=d.id \
   LEFT JOIN news_items n ON n.area_summary_id=a.id \
   WHERE d.date=CURRENT_DATE GROUP BY a.area ORDER BY a.area;"'
```

---

## 7. Change history

- **2026-06 — Migrated NewsAPI → RSS.** NewsAPI's free Developer tier was
  24h-delayed (a 25h window exposed only a ~1h sliver of available articles,
  starving narrow areas) and its terms forbade production/staging use. Replaced
  with authoritative RSS/Atom feeds: real-time, keyless, $0, compliant. Removed
  `NEWSAPI_KEY`, the `newsapi-secret` wiring from both CronJobs, and the secret
  from the SecretProviderClass; added `feedparser` to the daily job.
- **2026-06 — Reliability + quality fixes** (pre-migration, still in effect):
  per-area `try/except` isolation, source diversification, empty-area relaxed
  retry, rank clamp, wider look-back windows.

> Optional cleanup: the AKV `newsapi-key` secret still exists but is unreferenced.
> It is defined in the `infra-terraform` repo — remove it there for full tidiness.
