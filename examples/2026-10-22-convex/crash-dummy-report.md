# crash-dummy report: convex/convex-backend (self-hosted, cold-start)

Date: 2026-10-22 · Mode: cold-start, real app · Dummy: `acme-tasks` (Next.js 16 + convex SDK latest) · images `ghcr.io/convex/convex-backend:latest`, `convex-dashboard:latest`

## TTFS

clone → first successful call through the dummy (`npx convex run tasks:get` returning the imported rows): **3m 17s**
Split:
- **product setup** (clone → `/version` responds): **1m 15s** — 51s of that is cloning the 181 MB repo, which the docs don't actually ask for (they say download one compose file); `docker compose up` itself: 24s
- **scaffold** (`create-next-app` + npm install + `npm i convex`, not theirs): 17s
- **integration** (env + functions + `convex dev --once` + import + first query): **1m 31s**

Total wall clone → teardown: 7m 03s. No source read. Not `[assisted]`.
Benchmark line: common vendor target is first call in under 5–30 min (Twilio, Axway, Ably); no cross-vendor standard exists.

## Verdict

Convex self-host is the cleanest cold start of the three products tested today: prebuilt images, a 24-second `docker compose up`, a one-line admin key, and the Next.js quickstart runs verbatim against the local backend with live updates working on the first try — the only friction is the dashboard forgetting the admin key on every page load.

## Findings

| Grade | Finding | Evidence | Suggested fix |
|---|---|---|---|
| F-med | **Dashboard drops the admin key on every full page load.** Login page says "The admin key is required every time you open the dashboard"; navigating directly to `/data?table=tasks` bounced to login. Operators keep a 90-char key in their clipboard all day. | `shots/02-dashboard-login.jpg`; navigation to `http://localhost:6791/data?table=tasks` → login form | Persist the key in `localStorage`/session cookie for the deployment URL (opt-out for shared machines). |
| F-low | **`GET /version` on the `:latest` image returns `unknown`.** The compose healthcheck uses this endpoint; it passes, but tells you nothing about which backend you got. | `curl http://127.0.0.1:3210/version` → `unknown` | Stamp the image version at build time. |
| F-low | **Self-hosted README assumes you clone nothing but gives no fetch command for the compose file.** "First download the docker-compose.yml file" links to the GitHub tree view; a reader who clones the repo pulls 181 MB and 6,309 files for one 1 KB file. | `self-hosted/README.md` "Docker configuration" | Add `curl -O https://raw.githubusercontent.com/convex/convex-backend/main/self-hosted/docker/docker-compose.yml`. |
| F-low | **Generated `convex/_generated/api.js` has no module type**, so importing it from a plain Node script prints `MODULE_TYPELESS_PACKAGE_JSON` and reparses. Only bites non-bundler consumers (scripts, tests). | `node --env-file=.env.local dummy.mjs` warning | Emit `api.mjs` alongside, or document `"type": "module"`. |
| F-low | **Self-host docs say Convex Auth "requires manual setup since the CLI doesn't yet support self-hosted deployments"** and stop there — no link to what the manual setup is. | `self-hosted/README.md` "Other notes" | Link the manual steps or mark auth unsupported on self-host. |

## What worked (+)

- (+) `docker compose up` on prebuilt images: 24s to healthy. Only 3210/3211/6791 published; no collisions.
- (+) `docker compose exec backend ./generate_admin_key.sh` — one command, prints the key, done.
- (+) `npx convex dev --once` against `CONVEX_SELF_HOSTED_URL` pushed functions in 574 ms and *wrote* `NEXT_PUBLIC_CONVEX_URL` into `.env.local` for me.
- (+) The cloud Next.js quickstart runs verbatim on self-host: `sampleData.jsonl`, `convex import --table tasks`, `convex/tasks.ts`, `ConvexClientProvider`, `useQuery`. Zero edits.
- (+) Error quality is the best seen today: wrong arg type → `ArgumentValidationError` with `Path: .text`, `Value: 42.0`, `Validator: v.string()`, plus a request ID; missing function → `Could not find public function for 'tasks:nope'`.
- (+) Config change: a document added by the operator in the dashboard appeared in the running dummy's `useQuery` list with no reload (`7 tasks` → `8 tasks`).
- (+) Dashboard Add-document panel blocks invalid JSON inline ("Please fix the errors above to continue", Save disabled). Health tab already showed per-function call counts, failure rate and cache hit rate for the dummy's traffic.
- (+) `ConvexHttpClient` from `convex/browser` works for headless scripts with the same generated `api` object.

## Scenarios

| # | Scenario | Result | Evidence |
|---|---|---|---|
| 1 | Happy path: push functions, import 3 docs, query via CLI, 3 mutations + query via HTTP client, add via UI | PASS | CLI output, `dummy.mjs` transcript, `shots/01`, `shots/07` |
| 2 | Limits: wrong arg type, missing function, wrong URL | PASS (usable errors) | `dummy.mjs` transcript |
| 3 | Config change: operator adds a doc in the dashboard, live app updates | PASS | `shots/05`, `shots/06`, `shots/07` |
| 4 | Catalog CRUD: dashboard login, Data view, Add doc, invalid JSON | PASS | `shots/03`, `shots/04`, `shots/08` |
| — | Schema validation, Convex Auth, HTTP actions on :3211, file storage | NOT TESTED | — |

## Environmental deviations (not friction)

- Cloned the whole repo (skill rule) instead of downloading only `docker-compose.yml`; counted in setup but called out above.
- Browser window was stuck at 620×232 for the first dashboard pass; one Add-document attempt closed without saving in that state and could not be reproduced at full size — treated as a tool artefact, not a finding.

## The dummy

the `acme-tasks` dummy (`~/crash-dummies/acme-tasks/` locally) — Next.js 16, `convex/tasks.ts` (query + mutation), `app/ConvexClientProvider.tsx`, `app/page.tsx` (live list + add), `sampleData.jsonl`, `dummy.mjs` (HTTP-client flows + limits). Config via `.env.example`. Clean; works against cloud by swapping the two self-host vars for `npx convex dev`'s own login.

## Share block

"Fresh clone, docs only: self-hosted Convex up in 24 seconds, the stock Next.js quickstart running against it with live updates in 3m 17s total, and an operator edit in the dashboard showing up in the app without a reload. Best cold start of the day. One ask: remember my admin key."
