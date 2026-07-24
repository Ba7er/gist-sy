# Syria News Aggregator (Arabic) — System Design

## Context

Building a news aggregator focused exclusively on Syria, in Arabic, that pulls articles from a pre-defined set of source websites, scrapes them, and displays only the headline + reference link to the original article (filterable/sortable) on an HTML page. This document is the living design reference — implementation work should follow it, and it will be extended (frontend section, admin page, etc.) in later sessions.

**Status: backend/data-layer design locked in. Frontend design, admin page details, and a few ops items are still open (see Open Items).**

## Architecture Overview

Three independently deployed codebases, each its own git repo (on GitHub), living side-by-side in one parent directory on each VPS:

1. **`scraper`** (Python) — fetches sources, extracts articles, applies the Syria-relevance rule, writes to Postgres. Runs on-demand, not as a daemon.
2. **`backend`** (Node.js + Express + TypeScript) — read API over Postgres, serves JSON to the frontend.
3. **`frontend`** (vanilla JS/HTML/CSS, no framework) — consumes the backend API, renders the article list with filters/sorting/infinite scroll. Must have full **RTL support** (Arabic-first UI). *(Design details deferred to a later session.)*

All three share one **PostgreSQL** database as the integration point (scraper writes, backend reads).

## Infrastructure & Deployment

- **Hosting**: VPS-based, two environments — **dev** and **prod** — each running the full stack independently.
- **Containerization**: containerd + **nerdctl** as the CLI, one `compose.yaml` per environment (nerdctl compose, docker-compose-compatible spec).
  - `postgres` — persistent, named volume for data.
  - `backend` — persistent, Express API.
  - `scraper` — **not** a long-running service; invoked on demand.
  - `frontend` — reserved service slot, defined when frontend design is finalized.
- **Scheduling**: Linux **cron on the VPS host** (not in-container) triggers the scraper every **15 minutes** via `nerdctl compose run --rm scraper`, which runs to completion and exits.
- **Deployment process (v1)**: manual — SSH into the VPS, `git pull` in each of the 3 repo directories, `nerdctl compose up --build -d`. CI/CD is not in scope for v1.
- **DB access**: single shared Postgres role/user for both scraper and backend in v1. Least-privilege separation (separate read-only role for backend, write-only for scraper) is deferred — revisit post-v1.

## Database Schema (PostgreSQL)

### `sources`
| column | type | notes |
|---|---|---|
| id | serial PK | |
| name | text | e.g. "SANA", "Enab Baladi" |
| base_url | text | |
| scrape_method | text | `'rss'` \| `'html'` |
| config | jsonb | feed URL or listing URL + CSS selectors + source-specific category-label mapping; kept in DB (not code) so it's editable later via the admin page |
| is_active | boolean | default true |
| created_at, updated_at | timestamptz | |

### `categories`
| column | type | notes |
|---|---|---|
| id | serial PK | |
| name | text | controlled taxonomy, e.g. سياسة، عسكري، اقتصاد، إنساني، مجتمع، أخرى |
| sort_order | int | controls display order (e.g. keep "أخرى" last) instead of relying on Arabic alphabetical sort |

### `relevance_keywords`
| column | type | notes |
|---|---|---|
| id | serial PK | |
| term | text | e.g. سوريا، دمشق، حلب، الجيش السوري, etc. |
| is_active | boolean | editable later via admin page |

Seed list (draft, tunable later):
- Country forms: سوريا، سورية، السورية، السوري، السوريين
- Cities/regions: دمشق، حلب، حمص، حماة، اللاذقية، دير الزور، الرقة، إدلب، درعا، السويداء، القامشلي، القنيطرة، طرطوس، عفرين، منبج
- Institutions/entities: الحكومة السورية، الجيش السوري، النظام السوري، المعارضة السورية، قسد، الأمم المتحدة في سوريا

### `articles`
| column | type | notes |
|---|---|---|
| id | serial PK | |
| source_id | FK → sources.id | |
| category_id | FK → categories.id, nullable | mapped from source's own category label via `sources.config` |
| title | text | Arabic headline |
| excerpt | text | source-provided summary if scraper finds one, else auto-truncate `content` to ~200 chars + "…" |
| content | text | full article body, **plain text** (tags stripped) — stored for future use, never displayed on the page in v1 |
| url | text, **unique** | canonical link; the dedup key |
| status | text | `'published'` \| `'rejected'` — rejected articles (failed the relevance rule) are kept, not deleted, so they can be reviewed to judge/tune the relevance rule; `/api/articles` only ever returns `status = 'published'` |
| matched_keywords | text[] / jsonb, nullable | which `relevance_keywords` terms matched (null for rejected articles) — for debugging the relevance rule |
| published_at | timestamptz, nullable | parsed from source when available; unreliable/absent for many sources |
| scraped_at | timestamptz, default now() | **always populated** — used as the primary sort/pagination field instead of `published_at` |

Full-text search (`search_vector` tsvector column, Postgres `'arabic'` text-search config, weighted title > content) is a **planned but deferred** feature — schema will support adding it later without re-scraping, since `title`/`content` are already captured. v1 search is a simple `ILIKE` on `title`/`content` via the `q` param.

### `scrape_runs`
| column | type | notes |
|---|---|---|
| id | serial PK | |
| source_id | FK → sources.id | |
| started_at, finished_at | timestamptz | |
| status | text | `'success'` \| `'failure'` \| `'partial'` |
| articles_found | int | total items seen this run |
| articles_relevant | int | passed the relevance rule |
| articles_new | int | actually inserted (not already present by `url`) |
| error_message | text, nullable | |

### Duplicate handling
`url` UNIQUE constraint; inserts use `ON CONFLICT (url) DO NOTHING`, so re-scraping the same article on every 15-min run is a no-op. (To be tested once the scraper is built.)

### Sorting & pagination
- Primary sort field: **`scraped_at`** (not `published_at`, which is frequently null/unparseable). `published_at` is stored for display and for date-range filtering via `COALESCE(published_at, scraped_at)`.
- Pagination: **infinite scroll**, implemented via **keyset/cursor pagination** — `WHERE (scraped_at, id) < (cursor.scraped_at, cursor.id) ORDER BY scraped_at DESC, id DESC LIMIT N` — avoids OFFSET-based paging issues while new rows are actively being inserted by the scraper.

## API Contract (Express backend)

**`GET /api/articles`**

Query params:
| param | type | notes |
|---|---|---|
| `source_id` | int, optional | |
| `category_id` | int, optional | |
| `date_from`, `date_to` | `YYYY-MM-DD`, optional | filters on `COALESCE(published_at, scraped_at)` |
| `q` | string, optional | `ILIKE` on `title` + `content` |
| `sort` | `newest` \| `oldest`, default `newest` | maps to `scraped_at`/`id` ordering |
| `cursor` | opaque base64 string, optional | omit for first page; echo back `next_cursor` from prior response |
| `limit` | int, default 20, max 50 | |

Response `200`:
```json
{
  "articles": [
    {
      "id": 123,
      "title": "...",
      "excerpt": "...",
      "url": "https://...",
      "source": { "id": 1, "name": "SANA" },
      "category": { "id": 2, "name": "سياسة" },
      "published_at": "2026-07-20T10:00:00Z",
      "scraped_at": "2026-07-20T10:15:00Z"
    }
  ],
  "next_cursor": "opaque-string-or-null"
}
```
Invalid params → `400 { "error": "..." }`. Unexpected failures → generic `500`, details logged server-side only.

**`GET /api/sources`** → `[{ id, name }]`, active sources only.

**`GET /api/categories`** → `[{ id, name }]`, ordered by `sort_order`.

**`GET /health`** → `{ "status": "ok" }` (200) or 503 if DB unreachable — for container orchestration.

## Scraper Architecture (Python)

- **Config-driven adapter pattern**: a common interface (`fetch_articles() → raw articles`) with a per-source implementation behind it, driven by each source's `sources.config` row (no hardcoded per-source logic requiring redeploy for config-only changes).
- **Method preference**: use each source's **RSS/Atom feed** when available (structured, reliable — via `feedparser`); fall back to **HTML scraping** (`requests` + `BeautifulSoup`) only for sources without a feed. Avoid `Playwright`/`Selenium` (full browser automation) unless a specific source turns out to require JS rendering.
- **Politeness/robustness**: respect `robots.txt`, set a proper `User-Agent`, add delay between requests to the same site, use request timeouts, retry transient failures a couple times. Each source is wrapped in its own try/except — one source failing must not stop the others; failures are logged to `scrape_runs`.
- **Per-run pipeline**: read active `sources` + `relevance_keywords` from DB → for each source, fetch → parse/normalize (title, content as plain text, excerpt, category via source's label→category mapping, published_at parsed if present) → **relevance check** → upsert into `articles` → log one `scrape_runs` row per source.

### Syria-relevance rule (two layers)
1. **Source-level filtering**: where a source exposes a Syria-specific section/feed (vs. a general "international" one), the adapter config points only at the Syria-specific feed/section.
2. **Keyword safety net** (applies regardless of source): article must contain ≥1 active term from `relevance_keywords` in `title` or `content`. Articles that pass are stored with `status = 'published'` and `matched_keywords` populated; articles that fail are still stored, with `status = 'rejected'`, so they can be reviewed later to judge/tune the rule's accuracy — they are never returned by `/api/articles`.

## Open Items (to design in later sessions)

- **Frontend design** — page/component structure, RTL-specific layout/typography concerns, infinite-scroll implementation, filter/sort UI. Explicitly deferred until backend is fully settled.
- **Admin page** (Phase 2) — auth-protected UI to: add/edit `sources`, manage `relevance_keywords`, review `rejected` articles. Not part of v1 core.
- **Initial source list** — proposed candidates (pending final confirmation): SANA (sana.sy), Enab Baladi (enabbaladi.net), Syrian Observatory for Human Rights (syriahr.com), North Press Agency (npasyria.com), Zaman Al Wasl (zamanalwsl.net). Per-source scrape config (RSS vs HTML, selectors) to be determined during scraper implementation.
- **Full-text search** — `search_vector` (Postgres `'arabic'` config) planned, not yet implemented.
- **DB access hardening** — split single shared role into least-privilege scraper/backend roles, post-v1.
- **CI/CD** — none in v1 (manual git pull + compose rebuild); revisit if needed.
