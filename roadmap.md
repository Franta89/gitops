# Roadmap — "Listen" (text-to-speech) for Daily Dose of Tech

Prepared for the next session. **Goal:** add a **Listen** button that reads the news
aloud, per category and for the whole page, with a male and a female natural voice,
plus play/pause/stop and a progress bar. Use the latest Azure AI Foundry speech
offering, but **keep cost low** (the project has a monthly budget alert).

> Status: planning only — nothing implemented yet.

---

## 1. UX requirements (from the owner)

- A **Listen** button on **each category card** (Cloud Computing, AI Development,
  IT Security, Financial Markets, World News) → reads *that* category's summary.
- A **Listen** button at the **top of the page** → reads **all** categories.
- Clicking Listen opens a **popup with two voice options** (one male, one female).
  If the user doesn't pick, it plays in a **default voice**.
- Both voices must **sound natural/pleasant**.
- While reading: a **progress bar**; can be **paused / resumed / stopped** anytime.
- The Listen button **follows the page language toggle**: when the page is in
  **Czech** it reads with **Czech voices**; when in **English**, **English voices**.
  The male/female choice persists across both languages (it picks the same-gender
  voice in the active language).

---

## 2. Recommended solution (Azure AI Foundry → Speech, keyless)

**Service:** Azure AI **Speech** Neural Text-to-Speech, accessed through the
**existing** multi-service account `ais-ddot-dev-swc-001` (kind `AIServices` already
includes Speech, and it has a custom subdomain) — **no new Azure resource needed**.

**Auth:** keep the keyless pattern — Workload Identity. Add the role
**`Cognitive Services Speech User`** to the news-digest managed identity on the AI
Services account (Terraform: `infra-terraform/modules/azure-ai`). No keys in the app.

**Voices — native per language, chosen by (page language × gender):** the active
page language (EN/CZ toggle) selects the language; the popup selects the gender. So
the voice is a 2×2 matrix:

| | English page | Czech page |
| --- | --- | --- |
| **Female** (default) | `en-US-AvaMultilingualNeural` | `cs-CZ-VlastaNeural` |
| **Male** | `en-US-AndrewMultilingualNeural` | `cs-CZ-AntoninNeural` |

Using native `cs-CZ-*` voices for Czech gives the most natural Czech pronunciation
(the user explicitly wants Czech voices on the Czech page). The frontend passes the
current `lang` (from the toggle) + the persisted gender; the API maps them to the
voice above. (If you'd rather have one voice per gender for both languages, the
`en-US-*MultilingualNeural` voices can also speak Czech — a fallback, not the default.)

- **HD variants** (e.g. Ava HD) are available for higher fidelity at higher cost —
  see cost section; default to **Standard Neural** for cost.

**Synthesis:** build **SSML** per request (set `voice` + `xml:lang`, add short pauses
between articles, announce the category name) → call the Speech REST/SDK → get **MP3**.

---

## 3. Architecture — cost-optimal (cache, don't re-synthesize)

The digest is **static for the day** (generated 06:30 CET), so audio must be
**generated once and cached**, never re-synthesized per click. Recommended:

**Lazy synth + cache (primary):**
1. New API endpoint, e.g. `GET /api/audio?date=<YYYY-MM-DD>&category=<slug|all>&voice=<f|m>&lang=<en|cs>`.
2. Cache key = `date/category/voice/lang`. On **cache hit** → return the stored MP3.
   On **miss** → synthesize via Speech, store, then return.
3. Cache store: **Azure Blob Storage** container `audio` (the Foundry storage account
   already exists) — cheapest, simplest to serve (SAS URL or proxied by the API).
   Alternative: an Azure Files (RWX) PVC mounted by the API.
4. Frontend `<audio>` points at this endpoint (or the returned blob URL).

→ Each unique (category, voice, language) is synthesized **at most once per day**,
only for combinations users actually click. Old days' audio can be lifecycle-expired
from Blob after a few days.

**Optional pre-generation:** the daily aggregator CronJob could pre-synthesize the
default voice/language at digest time so the first click is instant. Trade-off: pays
for audio nobody may play. Recommend **lazy** first; add pre-gen only if first-play
latency is a problem.

---

## 4. Cost analysis & guardrails

Per-character billing (Mar 2026): **Standard Neural $15 / 1M chars**, **Neural HD
$22 / 1M chars**. Sources at the bottom.

- Content size ≈ 2k chars/category × 5 ≈ 10k chars; "all" ≈ 10k chars.
- **Worst case** (every category + "all", both voices, both languages, every day,
  no cache): ≈ 80k chars/day ≈ **2.4M chars/month ≈ ~$36/mo** (Standard) / ~$53 (HD).
- **With lazy caching** (synthesize once per listened combo/day): typically a small
  fraction of that — realistically **well under $10/mo** at modest traffic.

**Guardrails to bake in:**
- Default to **Standard Neural** voices (HD only if quality demands it).
- **Cache aggressively** (synth once per day per combo) — the single biggest lever.
- Synthesize **only the requested** voice/language on demand (don't fan out).
- Read **summaries only**, not full article bodies.
- Blob **lifecycle rule** to delete audio older than ~7 days.
- Watch the existing monthly budget alert; TTS adds to the ddot AI spend.

---

## 5. Implementation outline (files to touch)

**infra-terraform** (`modules/azure-ai/main.tf`):
- Add role assignment `Cognitive Services Speech User` for the news-digest identity
  on `azurerm_cognitive_account.ai_services`.
- (If Blob cache) a `audio` container on the existing storage account + lifecycle rule.

**gitops — API** (`manifests/news-digest/config/api-configmap.yaml` + `api/`):
- New `/api/audio` endpoint: build SSML, call Speech (keyless via Workload Identity),
  cache to Blob, stream/redirect MP3. Add Speech endpoint + voice names to
  `settings-configmap.yaml`.
- Pin `azure-cognitiveservices-speech` (or use the REST API to avoid the SDK/native
  deps in the `python:3.12-slim` ConfigMap-mounted model).

**gitops — Frontend** (`manifests/news-digest/config/frontend-configmap.yaml`):
- Per-card **Listen ▾** split button: main button plays the default voice; the ▾
  opens the male/female popup; persist choice in `localStorage`.
- Global **Listen all** button in the header (next to Monthly Summary / About).
- Audio **player UI**: HTML5 `<audio>` + custom **play/pause/stop** + **progress bar**
  (bind to `timeupdate`/`duration`, seekable). Only one stream at a time (starting a
  new one stops the previous). Respect the EN/CZ toggle for `lang`.
- Bump `app.js?v=N`.

---

## 6. Suggested phases

1. **Backend MVP** — Speech role (TF) + `/api/audio` with Blob cache, EN + default
   (female) voice, SSML. Verify it returns playable MP3 and caches.
2. **Frontend per-category Listen** + player (play/pause/stop + progress bar).
3. **Voice popup** (male/female) + persistence + global **Listen all**.
4. **Czech** support wired to the CZ toggle (multilingual voice or cs-CZ voices).
5. **Optional**: pre-generate default audio at digest time; offer HD voices.

---

## 7. Open decisions (confirm at implementation)

- Standard Neural vs HD (cost vs fidelity). Default: Standard.
- Voice mapping: **resolved** — native per-language voices (`en-US-*` / `cs-CZ-*`)
  selected by the page toggle × gender (see the matrix). Multilingual single-voice
  is only a fallback.
- Cache store: Azure Blob (recommended) vs Azure Files PVC.
- Lazy synth vs pre-generate at digest time.
- Default voice gender (proposed: female / Ava).
- Speech SDK vs REST from the ConfigMap-mounted `python:3.12-slim` pod (REST avoids
  native SDK deps — likely simpler here).

## 8. Verification (when built)

- `curl /api/audio?...` returns MP3; second call is a cache hit (fast, no synth).
- Browser: Listen per category + Listen all; pause/resume/stop; progress bar seeks.
- CZ toggle reads Czech in the chosen voice.
- Cost: confirm synth count ≈ listened combos/day (caching working); check budget.

---

## Sources
- Azure Speech Neural HD pricing update (Mar 2026): <https://techcommunity.microsoft.com/blog/azure-ai-foundry-blog/azure-speech-%E2%80%93-neural-hd-text-to-speech-recent-voice-updates/4505380>
- Azure Speech (Foundry Tools) pricing: <https://azure.microsoft.com/en-us/pricing/details/cognitive-services/speech-services/>
- Azure Speech TTS model in Foundry catalog: <https://ai.azure.com/catalog/models/Azure-Speech-Text-to-speech>
