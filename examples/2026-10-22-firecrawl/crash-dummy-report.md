# crash-dummy report: firecrawl/firecrawl (self-hosted v2.11.162, cold-start)

Date: 2026-10-22 · Mode: cold-start, real app · Dummy: `acme-research` (Next.js 16 + firecrawl SDK 4.51.0)

## TTFS

clone → first successful call through the dummy (`scrape example.com` in `dummy.mjs`): **15m 09s**
Split:
- **product setup** (clone → `/v0/health/readiness` ok): **14m 56s**, of which `docker compose up --build` = 13m 52s
- **scaffold** (`create-next-app` + npm install, not theirs): 50s — ran *during* the build, so it overlaps
- **integration** (stack ready → dummy's first successful scrape): **13s** headless, **77s** to the same call through the Next.js route. Caveat: the route and `dummy.mjs` were written while the build ran, so the authoring part of integration (~5 min) overlapped setup and isn't in the 13s.

Total wall clone → teardown: 17m 27s. No source read. Not `[assisted]`.
Benchmark line: common vendor target is first call in under 5–30 min (Twilio, Axway, Ably); no cross-vendor standard exists.

## Verdict

Once the stack is up, Firecrawl is the easiest integration tested so far — every SDK call worked first try and errors are specific — but the documented self-host quickstart spends 14 of its 15 minutes compiling Rust from source while a prebuilt image sits one commented line away in the same compose file.

## Findings

| Grade | Finding | Evidence | Suggested fix |
|---|---|---|---|
| F-high | **Self-host quickstart builds from source (13m 52s) when a prebuilt image exists.** `docs.firecrawl.dev/contributing/self-host` step 3 is `docker compose up --build -d`. `docker-compose.yaml` line 6 has `# image: ghcr.io/firecrawl/firecrawl` commented out with `build: apps/api` active; `docker manifest inspect ghcr.io/firecrawl/firecrawl:latest` → exists. The api image compiles Rust native deps (`maud`, `zstd`, `pdf-inspector`…) inside pnpm install. | `compose-up.log`: `docker compose up --build -d … 13:51.78 total` | Default compose to `image:`, keep `build:` behind a `docker-compose.dev.yaml` or a `--profile build`. Docs quickstart becomes `docker compose up -d`. |
| F-med | **Two self-host docs, two different `.env` templates.** Repo `SELF_HOST.md` says `PORT=3002 HOST=0.0.0.0 … BULL_AUTH_KEY=CHANGEME` and `POSTGRES_USER=firecrawl`; docs site says 4 vars with `POSTGRES_USER=postgres`, "leave `BULL_AUTH_KEY` unset", keep `POSTGRES_DB=postgres` for `pg_cron`. SELF_HOST.md step numbering jumps 3 → 5 and its curl test hits `/v1/crawl` while the site uses `/v2/scrape`. README links both. | `SELF_HOST.md` lines ~60–140 vs `docs.firecrawl.dev/contributing/self-host` | Make `SELF_HOST.md` a one-line pointer to the docs page. |
| F-med | **No doc anywhere says how to point the SDK at a self-hosted instance.** Node SDK page, npm README, self-host page: none mention `apiUrl`. The self-host page only covers CLI (`--api-url`) and MCP. I guessed `new Firecrawl({ apiUrl })` and it worked. | `docs.firecrawl.dev/sdks/node`, `npm view firecrawl readme | grep -ci apiurl` → 0 | Add a "Self-hosted" snippet to each SDK page: `new Firecrawl({ apiUrl: 'http://localhost:3002' })`, no key needed. |
| F-med | **`json` format without an LLM key returns a generic 500 "check your logs".** Root cause in api logs: `OPENAI_BASE_URL` is passed as an empty string by compose, so the AI SDK posts to `/responses` → `Failed to parse URL`. The HTTP response is `An error occurred. Please check your logs for more details. Error ID: …`. | `dummy.mjs` step `LIMIT json format w/o LLM key`; `docker compose logs api` "LLM extraction failed … Failed to parse URL from /responses" | Validate at request time: 400 `LLM_PROVIDER_NOT_CONFIGURED` "set OPENAI_API_KEY or OLLAMA_BASE_URL". Treat empty env strings as unset. |
| F-med | **`screenshot` format is silently dropped on self-host.** Request succeeds with `success: true` and no `screenshot` field; only a server-side warn says the engine doesn't support it. The docs' feature table does say screenshots need Fire-engine, but the API gives the caller no signal. | `dummy.mjs` step `LIMIT screenshot` → `{shot:false}`; api log "Request had format: screenshot … but there was no screenshot field" | Return a `warning` field in the response (the SDK already has a place for it) or 422 when none of the requested formats can be produced. |
| F-low | **46 "variable is not set" warnings on every `docker compose` command.** Docs pre-apologise for it ("warnings about unset optional variables are expected"). | `compose-up.log` | Use `${VAR:-}` in compose so Compose stops warning. |
| F-low | **`map` on a single-page site returns 0 links**, not even the root URL. `map('https://example.com')` → `links: []`, while `map('https://firecrawl.dev')` → 20. | `shots/02-dummy-map-empty.jpg` | Include the seed URL in `links`, or document that map only returns discovered links. |
| F-low | **Shallow clone can't follow the docs.** Docs say `git clone … && git checkout v2.11.162`; a `--depth 1` clone needs `git fetch --depth 1 origin tag v2.11.162` first. Minor, but the repo is 88 MB shallow. | my own `git checkout` error | Add `--branch v2.11.162 --depth 1` to the clone line. |

## What worked (+)

- (+) Docs-site self-host page is excellent: pinned release, honest "trusted-network quickstart, not production", a heartbeat curl *and* a functional smoke curl with expected output, feature-support table, troubleshooting by symptom.
- (+) `.env` is four lines. No secrets to generate, no admin login.
- (+) `USE_DB_AUTHENTICATION=false` means zero auth ceremony; SDK constructor with no key works as documented on the SDK page.
- (+) Every documented SDK call worked verbatim on the first try: `scrape` (274ms example.com, 1.8s firecrawl.dev), `map`, `crawl` (blocking helper), `search` — search worked with no SearXNG and no key (direct Google).
- (+) Error quality: bad URL → 400 with three specific validation messages; 404 page → `metadata.statusCode: 404`, `metadata.error: "Not Found"`; wrong `apiUrl` → `FirecrawlSdkError` wrapping ECONNREFUSED; localhost target rejected with a clear message.
- (+) Compose keeps Postgres, Redis, RabbitMQ unpublished on the host; only 3002 is exposed. No port collisions on a machine already running Postgres and Redis.

## Scenarios

| # | Scenario | Result | Evidence |
|---|---|---|---|
| 1 | Happy path: scrape ×3, map, crawl, search via SDK; scrape + map via Next.js route | PASS | `dummy.mjs` transcript, `shots/01`, `shots/02` |
| 2 | Limits: bad URL, 404 target, localhost target, wrong apiUrl | PASS (usable errors) | `dummy.mjs` transcript |
| 2b | Unsupported-on-self-host formats: screenshot, json | FAIL (silent drop / generic 500) | `dummy.mjs`, api logs |
| 3 | Config change mid-run | NOT TESTED — no admin/config surface in the default stack; queue UI needs `BULL_AUTH_KEY`, which docs say to leave unset (`/admin/CHANGEME/queues` → 404) | — |
| 4 | Catalog CRUD | N/A — no admin console | — |

## Environmental deviations (not friction)

- `COMPOSE_PROJECT_NAME=crashdummy-firecrawl` added to `.env`.
- Browser file/screenshot capture via Claude in Chrome; map/scrape outputs captured as JSON in the page.

## The dummy

the `acme-research` dummy (`~/crash-dummies/acme-research/` locally) — Next.js 16, `lib/firecrawl.ts`, `app/api/scrape/route.ts`, `app/api/map/route.ts`, `app/page.tsx`, `dummy.mjs` (6 happy-path + 6 limit flows). Config via `.env.example` (`FIRECRAWL_API_URL`). Works against cloud by adding `FIRECRAWL_API_KEY`. Clean; candidate for `examples/nextjs-self-hosted`.

## Share block

"Fresh clone, docs only: a Next.js app scraping, mapping and crawling through self-hosted Firecrawl in 15 minutes — 14 of them compiling Rust that a prebuilt image on ghcr.io already contains. Flip one comment in docker-compose.yaml and this is a sub-2-minute quickstart."
