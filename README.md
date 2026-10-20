# TGFI — Trilateral Geoeconomic Fragmentation Index

An unattended LLM pipeline that monitors 11 think-tank and research sources, extracts structured claims about
China–US, China–EU and US–EU relations, detects when sources flip position or disagree, and publishes
analyst-style briefs to an editorial dashboard.

Built solo for the HKUST Center for Macro-Finance & Geoeconomics (Feb–May 2026). TypeScript / Next.js 16 on
Vercel, Neon serverless PostgreSQL, Claude Haiku 4.5 + Sonnet 4.5.

---

## What it does

Every 30 minutes the system:

1. **Ingests** new articles from 11 RSS feeds (NBER, BIS, Bruegel, PIIE, MERICS, Rhodium Group, ECFR,
   Atlantic Council Global China Hub, RAND, CF40 Research, ECIPE).
2. **Extracts claims** with Claude Haiku 4.5: each claim gets a type (`QUANTITATIVE | CAUSAL | INTERPRETIVE |
   PREDICTIVE`), a direction score in `[-100, +100]` (fragmentation ↔ cooperation), a verbatim supporting quote,
   bucket weights across 6 dimensions, and the bilateral pairs it concerns.
3. **Detects patterns** twice a day across the last 180 days of claims:
   temporal flips, source disagreements, and cross-bucket divergences.
4. **Writes briefs** for the top 5 signals with Claude Sonnet 4.5, in the style of a forward-looking analyst note.
5. **Serves a dashboard** where researchers browse signals, dissenting evidence, 90-day sparklines, and the
   methodology.

The six dimensions ("buckets") are **trade, investment, technology, finance, leverage, policy**.
The three bilateral pairs are **CN-US, CN-EU, US-EU**.

---

## Architecture

```
                 Vercel cron                              Neon PostgreSQL
┌──────────────────────────────────────────┐        ┌──────────────────────┐
│ */30 * * * *    GET /api/ingest          │──────▶ │ articles  (pending)  │
│ 5,35 * * * *    GET /api/extract         │◀─────▶ │ articles  (5 states) │
│                   └─ Claude Haiku 4.5    │──────▶ │ claims               │
│ 0 0,12 * * *    GET /api/detect?analyze  │◀─────▶ │ signals              │
│                   ├─ lib/detector.ts     │        │ ingest_log           │
│                   └─ Claude Sonnet 4.5   │        └──────────────────────┘
└──────────────────────────────────────────┘                  ▲
                                                              │
┌──────────────────────────────────────────┐                  │
│ Next.js App Router (password-gated)      │                  │
│   /            signals + bucket matrix   │◀─────────────────┘
│   /sources     per-source coverage       │
│   /methodology rules, thresholds, scores │
│   /login                                 │
└──────────────────────────────────────────┘
```

### Pipeline stages

| Stage | Where | Notes |
|---|---|---|
| Ingest | `app/api/ingest/route.ts` | `rss-parser`, 20 items per source per run, `ON CONFLICT (url) DO NOTHING` so re-runs are idempotent, one `ingest_log` row per run |
| Extract | `app/api/extract/route.ts`, `lib/extractor.ts` | Claims 10 `pending` articles atomically with `FOR UPDATE SKIP LOCKED`, resets anything stuck in `processing` for > 10 min, truncates article text at 12,000 chars, caps 30 claims per article, marks each article `extracted`, `skipped` (no in-scope claims) or `failed` |
| Detect | `app/api/detect/route.ts`, `lib/detector.ts` | Loads up to 5,000 claims from the last 180 days, runs 3 detectors, keeps signals scoring > 60, archives the previous signal set and stores the top 10 |
| Analyze | `lib/analyzer.ts` | Sonnet 4.5 writes a 3–4 sentence brief for the top 5 signals. Prompt is anchored to today's date; JSON-only retry on parse failure; falls back to the raw summary |
| Enrich | `lib/signal-enrichment.ts` | For each displayed signal: dissenting claims (±45 days, opposite direction, different source), σ-baseline (Δ ÷ the source's 365-day direction std-dev), and sparkline data. Three queries run in parallel |

### Job queue semantics

The `articles.status` column is a five-state machine: `pending → processing → extracted | skipped | failed`.
Workers claim rows with

```sql
UPDATE articles SET status = 'processing'
WHERE id IN (
  SELECT id FROM articles
  WHERE status = 'pending' AND full_text IS NOT NULL AND word_count > 50
  ORDER BY fetched_at ASC LIMIT 10
  FOR UPDATE SKIP LOCKED
)
RETURNING ...
```

so the cron worker and the manual batch scripts can run at the same time without double-processing,
and no external queue is needed.

### Detection rules

| Pattern | Fires when | Score |
|---|---|---|
| `TEMPORAL_FLIP` | Same source × pair, different articles, \|Δ direction\| ≥ 60 within 45 days, same topic | `min(100, \|Δ\| · (1 + 1/√(days+1)))` |
| `SOURCE_DISAGREEMENT` | Same pair × dominant bucket (weight > 30), one source > +20 and another < −20, same topic | `min(100, gap · √n / 8)` |
| `CROSS_BUCKET_DIVERGENCE` | Same pair, two buckets with ≥ 2 claims each, average-direction gap > 50 | `min(100, gap · √n / 10)` |

"Same topic" means keyword Jaccard ≥ 0.20, **or** bucket-weight cosine ≥ 0.85 with Jaccard ≥ 0.08.
Thresholds were tightened in May 2026 after false-positive flips between superficially related claims.
Temporal flips are deduplicated to one signal per (source, pair, bucket); an earlier version stored 11 copies of
a single CF40 flip.

Each displayed signal also carries a 0–5 credibility score built from five weighted factors (source diversity,
sample size, detection margin, and two more documented on `/methodology`).

### Other modules

- **Rubric classifier** (`lib/classifier/`, `POST /api/classify`, `POST /api/classify/batch`) — a 9-anchor
  trade-bucket rubric used as both the LLM system prompt and the human labelling standard, with agreement
  metrics for gold-label validation (≥ 100 labels, ≥ 30 per pair, ±15-point acceptable error). The batch
  endpoint runs with a 5-way concurrency cap and a 50-article limit.
- **Hard-data pipeline** (`lib/pipeline/`, `GET /api/pipeline/trade`) — pulls bilateral goods trade from the
  OECD SDMX API, computes YoY growth, and normalizes to `[-100, +100]` against an 8-quarter rolling baseline.
- **Backtest** (`lib/backtest/`) — aligns monthly index scores with ETF returns (Yahoo Finance) and computes
  lagged correlations, lead-lag IC curves, directional hit rates and event studies.

---

## API routes

| Route | Auth | Purpose |
|---|---|---|
| `GET /api/ingest` | `Bearer CRON_SECRET` | Pull RSS feeds, insert new articles |
| `GET /api/extract` | `Bearer CRON_SECRET` | Claim-extract a batch of pending articles |
| `GET /api/detect?analyze=true&limit=10` | `Bearer CRON_SECRET` | Run detectors, optionally analyze top 5, store signals |
| `GET /api/signals?status=SIGNAL&limit=10` | rate-limited public | Read stored signals (no recomputation) |
| `GET /api/status` | rate-limited public | Live counts, last ingest run, per-source coverage |
| `POST /api/classify` | `Bearer CRON_SECRET` | Classify one article against the rubric |
| `POST /api/classify/batch` | `Bearer CRON_SECRET` | Classify up to 50 articles, 5 at a time |
| `GET /api/pipeline/trade?freq=Q&start=2022-Q1&pair=CN-US` | public | Run the OECD trade pipeline |
| `GET /api/tgfi` | public | Composite index summary |
| `POST /api/auth` / `GET /api/auth` | — | Login (sets cookie) / logout |

Pages are gated by a cookie checked in `proxy.ts` with a timing-safe compare. Cron routes are exempt from the
cookie gate and protected by the bearer secret instead. Public read endpoints are rate-limited to 30 requests
per minute per IP.

---

## Getting started

### Prerequisites

- Node 20+
- A Neon (or any) PostgreSQL database
- An Anthropic API key

### Environment variables

Create `.env.local`:

```bash
DATABASE_URL=postgres://...
ANTHROPIC_API_KEY=sk-ant-...
CRON_SECRET=<random string; Vercel sends it as a bearer token>
SITE_PASSWORD=<password for the dashboard>

# optional overrides
ANTHROPIC_ANALYSIS_MODEL=claude-sonnet-4-5
ANTHROPIC_CLASSIFY_MODEL=claude-sonnet-4-5
```

### Install, create the schema, run

```bash
npm install
DATABASE_URL="..." npx tsx scripts/setup-db.ts
DATABASE_URL="..." npx tsx scripts/add-indexes.ts
npm run dev
```

Open <http://localhost:3000>, log in with `SITE_PASSWORD`.

### Trigger the pipeline locally

```bash
curl -H "Authorization: Bearer $CRON_SECRET" http://localhost:3000/api/ingest
curl -H "Authorization: Bearer $CRON_SECRET" http://localhost:3000/api/extract
curl -H "Authorization: Bearer $CRON_SECRET" "http://localhost:3000/api/detect?analyze=true"
```

### Deploy

Deploy to Vercel and set the same environment variables. `vercel.json` registers the three cron schedules;
Vercel calls them with the `CRON_SECRET` bearer token automatically.

---

## Scripts

All scripts read `DATABASE_URL` (and `ANTHROPIC_API_KEY` where needed) from the environment and exit with a
clear message if anything is missing.

| Script | What it does |
|---|---|
| `scripts/setup-db.ts` | Create tables and indexes from `lib/db/schema.sql` |
| `scripts/add-indexes.ts` | Add performance indexes (idempotent) |
| `scripts/backfill.ts [--source=NAME]` | Pull historical articles from each source's archive: paged WordPress RSS, the Substack archive API, or `cheerio` HTML scraping. 1.5 s delay between requests |
| `scripts/fetch-content.ts [--source=NAME] [--limit=N]` | Fetch full text for articles that arrived with title only |
| `scripts/extract-batch.ts [batch_size]` | Extract more articles per run than the cron does (default 30) — for catching up after a backfill |
| `scripts/extract-source.ts "Bruegel,BIS" [limit]` | Targeted extraction to rebalance claim distribution across sources |
| `scripts/seed-signals.ts` | Run detection + analysis once and store the results |
| `scripts/audit.ts` | Health check: per-source coverage, empty-article rate, claim distribution, pair coverage, recent cron runs. Prints a PASS / WARN / FAIL verdict |
| `scripts/test-pipeline.ts`, `test-full-pipeline.ts`, `test-detection.ts` | Local end-to-end checks |

---

## Operating notes

- **Throughput and cost.** The extract cron handles 10 articles per run, 48 runs a day, so 480 articles/day
  at most. At roughly $0.003 per article on Haiku that is about $1.44/day. Each `detect?analyze=true` run spends
  about $0.07 on Sonnet.
- **Failure handling.** An article whose extraction throws is marked `failed` and not retried automatically; use
  `extract-source.ts` to retry. A worker that dies mid-batch leaves rows in `processing`; the next run resets
  anything older than 10 minutes.
- **Sources that block fetches.** PIIE and RAND return 403 to programmatic content fetches, so they are
  RSS-description only. `audit.ts` labels them accordingly.
- **Model lifecycle.** The analyst model was moved from Sonnet 4 to Sonnet 4.5 ahead of the Sonnet 4 end-of-life.

---

## Data model

```
articles    id, source_name, source_url, title, url (UNIQUE), published_at, fetched_at,
            full_text, word_count, status
claims      id, article_id → articles, claim_text, claim_type, direction, verbatim_quote,
            bucket_trade … bucket_policy (6 REALs), pairs TEXT[] (GIN), extracted_at
signals     id, pattern_type, claim_ids TEXT[], score, status
            (SIGNAL | HYPOTHESIS | INSIGHT | THESIS | ARCHIVED | DISPROVEN),
            title, summary, evidence JSONB, detected_at, updated_at, archived_at
ingest_log  id, run_at, sources_checked, articles_found, articles_new, errors JSONB
```

---

## Project layout

```
app/
  api/            ingest, extract, detect, signals, status, classify, pipeline/trade, tgfi, auth
  page.tsx        dashboard (home-client.tsx holds the interactive view)
  sources/        per-source coverage
  methodology/    rules, thresholds, credibility scoring
  login/
components/       bucket matrix, sparkline, trilateral diagram, navbar
lib/
  db/             neon client + schema.sql
  extractor.ts    Haiku claim extraction
  detector.ts     the three pattern detectors
  analyzer.ts     Sonnet brief writer
  signal-enrichment.ts
  classifier/     rubric + agreement metrics
  pipeline/       OECD trade hard-data pipeline
  backtest/       ETF backtest
  sources.ts      the 11 RSS sources
scripts/          backfill, content fetch, batch extraction, audit, tests
proxy.ts          cookie auth gate + legacy redirects
vercel.json       cron schedules
```

---

## Status and roadmap

Shipped and demoed to the center's principal investigator in May 2026. Deferred after the demo:

- Multi-agent "debate" pass over top signals (budget-gated)
- Full backfill of Chinese-language sources (needs translation)
- Knowledge-graph construction over claims
- Signal lifecycle management (`HYPOTHESIS → INSIGHT → THESIS` promotion)

---

## Author

Thomas Tse — [GitHub](https://github.com/thomasyunghint) · [LinkedIn](https://www.linkedin.com/in/thomasyht/)
