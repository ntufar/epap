# EPAP — Improvement Ideas & New Features (Technical Specifications)

**Scope:** the ΕΠΑΠ platform — Flask backend (`app.py`), Vercel serverless handler (`api/index.py`), web PWA, Flutter mobile app (`epap_mobile/`), and browser extension (`browser-extension/`).
**Date:** July 2026
**Status:** proposal / backlog

This document is organized by area. Each idea includes a technical specification, effort estimate (S/M/L/XL), and priority (P0 = foundational, P1 = high value, P2 = nice to have).

---

## 1. Core Analysis Engine

### 1.1 Structured JSON output instead of markdown + regex parsing — **P0, M**

**Problem.** The LLM returns free-form Greek markdown; the overall score is recovered by regex (`ScoreExtractor` in Flutter, similar logic in web/extension). Any prompt or model change silently breaks every client, and per-criterion scores are not machine-readable at all.

**Spec.**
- New endpoint `POST /api/v2/analyze` returning a versioned JSON document. Keep `/analyze` (v1) as a compatibility shim that renders v2 into the current markdown.
- Use Mistral's JSON mode / structured output (`response_format={"type": "json_object"}` or the newer schema-constrained mode) with a schema like:

```json
{
  "schema_version": "2.0",
  "overall_score": 42,
  "classification": "medium",            // excellent | high | medium | low
  "confidence": 0.78,
  "criteria": [
    {
      "id": "emotional_manipulation",
      "label_el": "Συναισθηματική Χειραγώγηση",
      "score": 35,
      "summary_el": "…",
      "evidence": [
        {"quote": "…", "start": 812, "end": 878, "technique": "loaded_language"}
      ]
    }
    // bias_indicators, fact_vs_opinion, source_reliability,
    // language_analysis, logical_fallacies, recommendation
  ],
  "techniques_detected": ["loaded_language", "appeal_to_fear"],
  "article": {"title": "…", "source_domain": "documentonews.gr", "published_at": "…", "language": "el"},
  "meta": {"model": "mistral-large-latest", "prompt_version": "3", "cached": false, "latency_ms": 4210}
}
```

- Validate the LLM output server-side with `pydantic`; on validation failure, retry once with a repair prompt, then fall back to markdown-only with `"degraded": true`.
- Character offsets (`start`/`end`) refer to the extracted article text returned alongside the result — this is what enables inline highlighting (see 5.1 and 6.2).
- Clients migrate incrementally: web first, then extension, then Flutter (`AnalysisResult.fromJson` replaces `ScoreExtractor`).

### 1.2 Provider abstraction & model router — **P0, M**

**Problem.** The code is hard-wired to Mistral; the README still says Gemini. There is no fallback when the provider errors or rate-limits, and no way to A/B model quality/cost.

**Spec.**
- `services/llm.py` with a `Provider` protocol: `complete(prompt, schema, timeout) -> AnalysisResult`. Implementations: `MistralProvider`, `GeminiProvider`, `AnthropicProvider`, `OpenAIProvider` (or route everything through one gateway/OpenRouter-style endpoint to avoid N SDKs).
- Config via env: `LLM_PRIMARY=mistral:mistral-large-latest`, `LLM_FALLBACKS=gemini:gemini-2.5-flash,...`, `LLM_TIMEOUT_S=45`.
- Fallback chain: on timeout / 429 / 5xx / schema-validation failure, try the next provider; annotate `meta.model` with what actually served the request.
- Record per-request token counts and cost (each SDK exposes usage) into the analytics table (see 3.4).
- Fix the README to describe the actual provider setup.

### 1.3 Prompt versioning + evaluation harness — **P0, M**

**Problem.** Prompt changes ship blind. There is no way to know whether a new prompt/model scores Greek propaganda better or worse.

**Spec.**
- Move prompts out of code into `prompts/analyze_v{N}.md` with front-matter (version, model targets, changelog). The active version is config-driven; `meta.prompt_version` is stored with every result.
- Build a golden dataset: 100–300 Greek articles labeled by humans (start with the team; later crowdsource — see 8.2). Each item: text, expected score band (±15), expected techniques, notes. Store in `eval/dataset.jsonl`.
- Eval runner (`eval/run.py`, or promptfoo config): runs the candidate prompt × model matrix against the dataset, reports MAE on scores, precision/recall on technique detection, and schema-validity rate. Runs in CI on any change under `prompts/`.
- Acceptance gate: a prompt change may not regress score MAE by more than X points on the golden set.

### 1.4 Propaganda technique taxonomy with span-level detection — **P1, M**

**Spec.**
- Adopt an established taxonomy rather than an ad-hoc list — the SemEval-2020 Task 11 catalogue of 18 techniques (loaded language, name calling, exaggeration/minimisation, appeal to fear, flag-waving, doubt, whataboutism, red herring, bandwagon, etc.) plus Greek-specific additions.
- The v2 schema (1.1) already carries `technique` + span offsets per evidence quote. Add a static registry `taxonomy.json` with per-technique: id, Greek name, plain-language explanation in Greek, canonical example. Served at `GET /api/v2/taxonomy` so all clients render consistent tooltips.
- This powers inline highlighting (5.1, 6.2) and the education mode (5.6).

### 1.5 Two-stage analysis: cheap triage + deep analysis — **P1, M**

**Spec.**
- Stage 1 (triage): a small/cheap model (`mistral-small`, `gemini-flash-lite`) classifies: is this news content at all? language? obvious satire/opinion label? rough risk band. Cost: ~1/20 of the large model.
- Stage 2 (deep): the full structured analysis, only for content that passes triage. Non-news content returns a friendly "this doesn't look like a news article" response instead of a fabricated score.
- Optional ensemble mode (research flag): run 2 models, if scores diverge > 25 points set `confidence` low and surface "τα μοντέλα διαφωνούν" in the UI.

### 1.6 Claim extraction & fact-check cross-referencing — **P1, L**

**Spec.**
- Add a claim-extraction step to the prompt: list of check-worthy factual claims with spans.
- Cross-reference each claim against:
  - **Google Fact Check Tools API** (`claims:search`, ClaimReview markup) — free, covers AFP Ελλάδα and others.
  - A curated index of Greek fact-checkers (Ellinika Hoaxes, Greece Fact Check, AFP Ελλάδα) built by ingesting their RSS/sitemaps into a small embeddings index (pgvector on Neon; `text-embedding` via the same gateway). Match claims by cosine similarity ≥ threshold.
- Response addition: `"fact_checks": [{"claim": "…", "verdict": "false", "publisher": "Ellinika Hoaxes", "url": "…"}]`.
- UI: a "Σχετικοί έλεγχοι γεγονότων" section with links. This materially raises credibility versus a pure LLM opinion.

### 1.7 Source reputation database + automatic source detection — **P1, M**

**Problem.** "Source" is a free-text field the user types; the LLM judges `source_reliability` from its priors.

**Spec.**
- Table `sources` (see 3.4): domain, canonical name, ownership/parent company, funding notes, political-lean notes (sourced, with citations), rolling average EPAP score, analysis count.
- Auto-detect from the URL's registrable domain (`tldextract`); drop the free-text field for URL mode.
- Feed the source's own rolling stats into the prompt ("this outlet's last 90-day average score is 44 across 120 articles") — grounding the source-reliability criterion in data instead of model vibes.
- Public per-source pages `/source/<domain>` (see 8.3).

### 1.8 Cross-outlet story comparison — **P2, L**

**Spec.**
- Given an analyzed article, extract title/entities, query Google News RSS (`news.google.com/rss/search?q=…&hl=el&gl=GR&ceid=GR:el`) for the same story in other Greek outlets.
- Fetch + analyze 2–4 alternatives (respecting cache), and render a comparison: same event, different framing, per-outlet scores.
- Expensive — gate behind an explicit user action ("Σύγκριση κάλυψης") and cache aggressively by story cluster (embedding similarity of titles).

---

## 2. Content Extraction

### 2.1 Replace hand-rolled BeautifulSoup extraction with `trafilatura` — **P0, S**

**Problem.** The current selector-list approach (`main`, `article`, `.content`…) misses many Greek news CMSes and falls back to whole-`body` text, polluting the analysis with menus and cookie banners.

**Spec.**
- Use `trafilatura.extract(html, include_comments=False, target_language="el")` — state of the art, pure Python, Vercel-compatible. Keep the current logic only as a last-resort fallback.
- Also extract metadata via `trafilatura.metadata`: title, author, date, canonical URL, sitename — fill `article.*` in the v2 schema and the `sources` table.
- Return structured errors (`{"error_code": "EXTRACTION_FAILED", …}`) instead of the current `"Error …"` string sentinel that callers detect with `startswith("Error")`.

### 2.2 SSRF hardening of URL fetching — **P0, S** *(security)*

**Problem.** `extract_text_from_url` fetches any user-supplied URL from the server. An attacker can point it at `http://169.254.169.254/…` (cloud metadata), internal services, or use it as a request proxy.

**Spec.**
- Resolve the hostname first; reject private/reserved ranges (RFC 1918, loopback, link-local, ULA) — validate again after each redirect (max 3 redirects).
- Enforce: scheme ∈ {http, https}, port ∈ {80, 443}, response size cap (2 MB streamed), content-type `text/html`, total fetch budget 10 s.
- Consider an allowlist-first mode for known Greek news domains with a monitored fallback for others.

### 2.3 Smarter handling of long articles — **P1, S**

**Problem.** Extraction truncates at 3,000 chars and the prompt at 2,000 — long investigative pieces are judged on their intro only, which biases scores.

**Spec.**
- `mistral-large` has a 128k context; raise the limit to ~20,000 chars directly (cost impact is modest with caching, see 10.1).
- Above that, map-reduce: chunk → per-chunk mini-analysis with the cheap model → final judgment prompt over the summaries. Mark `meta.truncated`/`meta.chunked` in the response.

### 2.4 Fallbacks for JS-rendered and blocked pages — **P2, M**

**Spec.**
- Fallback chain when extraction yields < 100 chars: ① AMP variant (`<link rel="amphtml">`), ② `r.jina.ai/<url>` reader proxy, ③ Wayback Machine latest snapshot API.
- If all fail, return a specific error telling the user to paste the text (current behaviour, but with an explanatory error code the clients can render nicely).
- Note: do not build paywall circumvention; only use publicly served variants.

---

## 3. Backend Architecture

### 3.1 Unify `app.py` and `api/index.py` into one codebase — **P0, L**

**Problem.** `api/index.py` is a 1,675-line `BaseHTTPRequestHandler` that re-implements the Flask app (routes, extraction, prompt, cache) by hand. Every feature must be written twice; they have already drifted.

**Spec.**
- Vercel's Python runtime (Fluid Compute) runs WSGI apps natively: point `vercel.json` at the Flask app (`app.py` exposing `app`) and delete the duplicated handler. Templates/static resolve from one place.
- Keep `api/` only as the Vercel entry shim (`api/index.py` → `from app import app`).
- Acceptance: the full existing test suite (`test_app.py`) passes against the deployed preview; parity checklist for every current route (`/`, `/analyze`, `/health`, `/status`, `/about`, `/privacy`, SEO files, PWA files).
- This is the highest-leverage refactor in the repo — nearly every idea below costs half as much after it.

### 3.2 Persistent shared cache (Redis) — **P0, S**

**Problem.** `analysis_cache = {}` is per-instance memory: useless across serverless instances, lost on every cold start, and unbounded locally. The MD5 key uses only the first 1,000 chars, so different articles can collide.

**Spec.**
- Upstash Redis via Vercel Marketplace. Key: `analysis:v{schema}:{prompt_version}:{sha256(canonical_url or full_text)}`. TTL 30 days. Value: the v2 JSON.
- Same Redis backs `flask-limiter` (`storage_uri=os.environ["REDIS_URL"]`) — the current in-memory limiter provides no protection on serverless, where each instance counts separately.
- Add cache stats to `/status` (hit rate, size) from Redis `INFO`.

### 3.3 Streaming responses (SSE) — **P1, M**

**Problem.** Analyses take 5–20 s with a blank spinner; serverless timeout risk on slow model days.

**Spec.**
- `POST /api/v2/analyze/stream` returning `text/event-stream`: events `status` ("extracting", "analyzing"), `delta` (raw tokens for a live text preview), `result` (the final validated JSON), `error`.
- Mistral SDK supports streaming (`chat.stream`). Structured output + streaming: stream the raw text for UX, validate the complete document at the end and emit `result`.
- Clients: web uses `EventSource`/fetch-streams; Flutter uses `http` streamed request; extension popup uses fetch streams. All keep the non-streaming endpoint as fallback.

### 3.4 Database for analyses, sources, and analytics — **P0, M**

**Spec.** Neon Postgres (Vercel Marketplace). Minimal schema:

```sql
CREATE TABLE analyses (
  id            TEXT PRIMARY KEY,        -- nanoid, used in share URLs
  url           TEXT,
  url_hash      TEXT,
  source_domain TEXT REFERENCES sources(domain),
  overall_score INT NOT NULL,
  result        JSONB NOT NULL,          -- full v2 document
  prompt_version TEXT, model TEXT,
  tokens_in INT, tokens_out INT, cost_usd NUMERIC(8,5),
  created_at    TIMESTAMPTZ DEFAULT now()
);
CREATE TABLE sources (
  domain TEXT PRIMARY KEY, name TEXT, ownership TEXT, notes TEXT,
  avg_score_90d NUMERIC, analysis_count INT DEFAULT 0
);
CREATE TABLE feedback (
  analysis_id TEXT REFERENCES analyses(id),
  agrees BOOLEAN, comment TEXT, created_at TIMESTAMPTZ DEFAULT now()
);
```

- No PII: no user IDs, no raw IPs (rate limiting hashes the IP with a rotating salt).
- Unlocks: permalinks (5.3), source pages (8.3), the public dashboard (8.4), cost tracking (1.2), and the eval feedback loop (8.2).

### 3.5 Observability & error tracking — **P1, S**

**Spec.**
- Sentry SDK (Flask integration) for exceptions; structured JSON logs (request id, route, duration, cache hit, model, tokens) so Vercel log drains are queryable.
- `/health` extended with dependency probes (Redis ping, DB ping, provider reachability) returning per-dependency status — the mobile app and uptime monitors can use it.
- A simple daily cost report (cron route `/api/cron/daily-report`, Vercel Cron) summarizing request counts, cache hit rate, and LLM spend; e-mail or log output.

### 3.6 Security hygiene — **P0, S**

- **Remove `release.keystore` and `keystore.properties` from the repository root.** Signing keys must never be committed; move to CI secrets, rotate the keystore password, and rewrite history if the repo is public (`git filter-repo`), or at minimum invalidate/rotate.
- Add security headers on all HTML responses: CSP (self + required CDNs), `X-Content-Type-Options: nosniff`, `Referrer-Policy`, `Permissions-Policy`.
- CORS: explicit allowlist for the extension origins and the app domain instead of any implicit permissiveness.
- Dependabot / `pip-audit` in CI.

---

## 4. Public API & Integrations

### 4.1 Versioned public API with keys and quotas — **P1, M**

**Spec.**
- `/api/v2/*` documented with an OpenAPI 3.1 spec (`flask-smorest` or hand-written `openapi.yaml` served at `/api/docs`).
- API keys (`X-API-Key`) stored hashed in Postgres with per-key daily quotas; anonymous access keeps the current low rate limits. Target users: Greek newsrooms, researchers, media-literacy NGOs.
- Batch endpoint `POST /api/v2/analyze/batch` (≤ 20 URLs, async): returns a job id; results retrievable at `GET /api/v2/jobs/<id>`. Backed by a simple Postgres job table + Vercel Cron worker (or Vercel Queues when convenient).

### 4.2 MCP server — **P2, S**

**Spec.**
- Expose an MCP server (`/api/mcp`, streamable HTTP) with tools `analyze_article(url|text)` and `get_source_stats(domain)`, so Claude/other assistants can call EPAP directly. Python `mcp` SDK; thin wrapper over the v2 service layer. Great demo material for the Open Innovation Challenge.

### 4.3 Monitoring & digests (RSS-driven) — **P2, L**

**Spec.**
- Ingest the RSS feeds of the top ~20 Greek outlets on a Vercel Cron (hourly); analyze new items through the cached pipeline (cheap-model triage first, 1.5).
- Products on top: daily "most manipulative headlines" digest page, weekly per-source trend report, optional e-mail newsletter (Resend/Postmark).
- Budget guard: hard daily token budget; skip analysis when exhausted (log and continue).

---

## 5. Web Frontend / UX

### 5.1 Inline evidence highlighting — **P1, M**

**Spec.**
- Using v2 span offsets, render the extracted article text with `<mark data-technique="loaded_language">` spans, color-coded by technique family; tooltip shows the technique's Greek name + explanation from the taxonomy endpoint (1.4).
- This transforms the product from "a score" into "a teacher" — the single most differentiating UX idea in this list.

### 5.2 Score visualization: gauge + radar + source context — **P1, S**

**Spec.**
- Keep the overall gauge; add a 7-axis radar chart of criterion scores (lightweight SVG, no chart library needed) and a "source context" line: this article vs the outlet's 90-day average (from `sources`).
- Follow accessible color rules (don't encode meaning in color alone; add labels).

### 5.3 Public shareable result pages with OG images — **P1, M**

**Spec.**
- Every analysis gets `/a/<id>` (from 3.4): server-rendered result page, canonical, indexable.
- Dynamic OG image (`/a/<id>/og.png`): score card with gauge, title, source — Pillow-rendered PNG (or Vercel OG if a Node function is acceptable). This makes every shared analysis an advertisement for the platform.
- "Κοινοποίηση" buttons (X, Facebook, Viber — Viber is huge in Greece) + copy-link.

### 5.4 Modernize the frontend incrementally with htmx — **P2, M**

**Problem.** `templates/index.html` is 749 lines of inline HTML/CSS/JS.

**Spec.**
- Stay server-rendered (Flask + Jinja) but split into partials, extract CSS/JS to static files, and use htmx for the analyze flow (`hx-post`, SSE extension for streaming). Avoid a SPA rewrite — the app is fundamentally one form and one result view.
- Add dark mode via `prefers-color-scheme` + toggle (the mobile app already has it; the web doesn't).
- Accessibility pass: WCAG 2.1 AA — labels, focus states, `aria-live` on the streaming result region, contrast.

### 5.5 History on web (localStorage) — **P2, S**

**Spec.**
- Mirror the mobile app's history: persist the last ~50 analyses (id, url, score, date) in `localStorage`; render a "Ιστορικό" drawer. No accounts needed; links resolve to `/a/<id>` permalinks.

### 5.6 Education / media-literacy mode — **P1, M**

**Spec.**
- `/learn`: one page per taxonomy technique (from 1.4) with definition, real (anonymized) Greek examples harvested from high-scoring detections, and a "spot the technique" mini-quiz (static JSON question bank to start).
- "Γιατί αυτή η βαθμολογία;" expandable explainer on every result linking each criterion to its `/learn` page.
- This aligns EPAP with EU media-literacy funding streams (EDMO / MedDMO hub for Greece-Cyprus-Malta) and strengthens the Open Innovation Challenge story: not just a detector, a teaching tool.

---

## 6. Browser Extension

### 6.1 One-click analysis of the current tab with toolbar badge — **P1, M**

**Spec.**
- Popup button "Ανάλυση σελίδας" sends the tab URL to the API; on completion, set `chrome.action.setBadgeText` to the score with the classification color. Cache per-tab in `chrome.storage.session`.
- Content script extracts the article text locally (Readability.js vendored) and POSTs text instead of URL when possible — avoids server-side fetch failures on cookie-walled pages and cuts server bandwidth.

### 6.2 In-page highlighting overlay — **P2, L**

**Spec.**
- After analysis, the content script maps evidence quotes back into the live DOM (string search with normalization; spans that fail to match are listed in the popup instead) and wraps them in highlight marks with tooltips — the 5.1 experience on the publisher's own page.

### 6.3 Context menu + link-hover affordances — **P2, S**

**Spec.**
- `contextMenus`: "Ανάλυση με ΕΠΑΠ" on links and selected text (selection ≥ 50 chars posts as text).
- Optional: badge known scores next to links on `news.google.com`/social feeds via a `GET /api/v2/scores?urls=…` bulk-lookup endpoint that only returns cached/known results (no fresh analyses — keeps it fast and free).

### 6.4 Restore the publishing guide — **P0, S**

`docs/BROWSER_EXTENSION_PUBLISHING.md` currently contains an unrelated telematics business-case document — the actual store-publishing guide was overwritten (visible as the uncommitted modification). Restore it via `git checkout -- docs/BROWSER_EXTENSION_PUBLISHING.md` after saving the telematics text elsewhere, then finish the Chrome Web Store / AMO / Edge Add-ons submissions it describes.

---

## 7. Mobile App (Flutter)

### 7.1 Migrate to API v2 — **P1, S**
Replace `ScoreExtractor` regex parsing with `AnalysisResult.fromJson`; render criteria cards from structured data; keep markdown rendering only for the free-text summary fields.

### 7.2 Streaming + richer result screen — **P2, M**
Consume the SSE endpoint (3.3) for progressive display; add the radar chart and technique highlights (spans over the returned extracted text).

### 7.3 iOS share extension & release — **P1, M**
Finish the iOS share-sheet integration noted as "future" in the implementation plan (native Share Extension target posting to a shared App Group, read by `receive_sharing_intent`); TestFlight then App Store release.

### 7.4 Offline & sync-light history — **P2, S**
Keep history local (sqflite) but store the permalink id so "άνοιγμα στο web" works; queue analyses made while offline and submit on reconnect.

### 7.5 Home-screen widget — **P2, M**
Android app widget + iOS WidgetKit showing the day's "most manipulative headline" from the digest API (4.3).

---

## 8. Data, Community & Accounts

### 8.1 Optional accounts — **P2, L**
Only when a concrete need exists (cross-device history, researcher API keys, saved sources). Magic-link e-mail auth (no passwords); store minimal profile. Do not gate core analysis behind login — anonymity is a feature for this product category.

### 8.2 User feedback loop feeding the eval set — **P1, S**

**Spec.**
- On every result: "Συμφωνείτε με την αξιολόγηση;" 👍/👎 + optional comment → `feedback` table (3.4).
- Weekly job surfaces high-disagreement analyses for human review; confirmed misjudgments are added (with corrected labels) to the golden eval dataset (1.3). This closes the quality loop with zero ML infrastructure.

### 8.3 Public per-source pages — **P1, M**

**Spec.**
- `/source/<domain>`: rolling average score with trend sparkline, score distribution histogram, criterion breakdown averages, recent public analyses, ownership/notes from `sources`.
- **Legal guardrails (required):** publish the methodology page first (9.1); show stats only when `analysis_count ≥ 30`; label everything as "AI-assisted assessment of specific articles", never "this outlet is propaganda"; provide a corrections contact (9.2).

### 8.4 Public dashboard — **P2, M**
`/dashboard`: platform-wide stats (analyses/day, average scores by category, technique frequency over time, most-analyzed stories). Server-rendered charts from nightly-materialized aggregates; a strong artifact for press and the Innovation Challenge jury.

### 8.5 Researcher data export — **P2, S**
`GET /api/v2/export?from=&to=` (API-key gated, researcher tier): JSONL of analyses with URLs, scores, techniques — no user data to strip because none is stored. CC-BY-style license note.

---

## 9. Trust, Ethics & Legal

### 9.1 Methodology & limitations page — **P0, S**
`/methodology` in Greek and English: how scoring works, which model(s), prompt criteria, known failure modes (satire, opinion pieces, short texts), the meaning of confidence, and an explicit statement that scores assess *articles, not journalists*, and are AI-generated opinions, not findings of fact. Link it from every result. This is both an ethical requirement and the primary defamation-risk mitigation for 8.3.

### 9.2 Corrections / appeal channel — **P1, S**
A form (`/report`) where an outlet or reader can flag a wrong analysis; flagged items get re-run with the ensemble mode (1.5) and human review; resolved cases annotate the public permalink ("επανεξετάστηκε").

### 9.3 GDPR posture — **P0, S**
Already light (no accounts); formalize it: no raw IP storage (salted hash for rate limits, salt rotated daily), 30-day log retention, privacy page updated to reflect Redis/Postgres processors and the LLM provider (article text is sent to the model provider — must be disclosed).

### 9.4 EDMO / fact-checker partnerships — **P2, ongoing**
Approach MedDMO (the Greece/Cyprus/Malta EDMO hub) and Ellinika Hoaxes for data partnership (1.6) and credibility; their endorsement matters more than any feature for adoption by schools and newsrooms.

---

## 10. Performance & Cost

### 10.1 Prompt caching — **P1, S**
The ~1,200-token instruction block is identical on every call; put the static instructions first and the article last, and enable the provider's prompt/context caching where available — typical 50–90 % input-token discount on the static part.

### 10.2 Semantic deduplication — **P2, M**
Greek outlets republish wire copy (ΑΠΕ-ΜΠΕ) nearly verbatim; exact-hash caching misses trivial edits. Compute an embedding of the first ~1,500 chars, store in pgvector, and serve the cached analysis when cosine similarity ≥ 0.97 (mark `meta.near_duplicate_of`).

### 10.3 Edge caching of public pages — **P2, S**
Permalinks (5.3), source pages (8.3), and the dashboard are immutable or slowly-changing: `Cache-Control: public, s-maxage=3600, stale-while-revalidate=86400` and let Vercel's CDN absorb the traffic.

---

## 11. Suggested Sequencing

**Phase 1 — Foundations (unblocks everything):**
3.6 security hygiene → 2.2 SSRF fix → 3.1 codebase unification → 3.2 Redis cache/limits → 1.1 structured v2 API → 3.4 database → 2.1 trafilatura → 9.1/9.3 methodology & GDPR → 6.4 restore the publishing doc.

**Phase 2 — Differentiation:**
1.2 provider router → 1.3 eval harness → 1.4 taxonomy → 5.1 inline highlighting → 5.3 permalinks + OG images → 3.3 streaming → 7.1 mobile v2 migration → 8.2 feedback loop → 10.1 prompt caching.

**Phase 3 — Platform & community:**
1.6 fact-check integration → 1.7/8.3 source database & pages → 5.6 education mode → 4.1 public API → 4.3 digests → 6.1 extension badge → 7.3 iOS release → 8.4 dashboard.

**Phase 4 — Reach:**
1.8 cross-outlet comparison → 4.2 MCP server → 6.2 in-page highlighting → 8.5 researcher export → 10.2 semantic dedup → 9.4 partnerships.
