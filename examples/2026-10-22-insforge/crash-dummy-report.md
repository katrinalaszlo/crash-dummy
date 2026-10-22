# crash-dummy report: InsForge/InsForge (self-hosted, cold-start)

Date: 2026-10-22 · Mode: cold-start, real app · Dummy: `acme-notes` (Next.js 16 + @insforge/sdk 1.5.2) · InsForge OSS backend 2.3.4

## TTFS

clone → first successful call through the dummy (`auth.signUp` from `dummy.mjs`): **10m 01s**
Total wall time clone → teardown: 13m 33s. No source read. Not `[assisted]`.

Breakdown: setup script 2s · `docker compose up -d` 1m 27s (image pulls) · the rest was reading docs and one
dead end on the admin login page.

## Verdict

The self-hosted stack comes up clean in under two minutes, but the first screen a self-hoster sees lies to them
(prefilled wrong admin password, CLI tab that only works on cloud), and a table created the only way a self-hoster
can create one (SQL Editor) is wide open to the public anon key — contradicting the docs' "RLS on by default".

## Findings

| Grade | Finding | Evidence | Suggested fix |
|---|---|---|---|
| F-high | **Table created in the dashboard SQL Editor has RLS off and the anon key can SELECT, INSERT and DELETE every row.** FAQ says "New tables have RLS on by default" — true only for the API/MCP create-table path. Self-hosters without MCP are pointed at the SQL Editor; migrations CLI is cloud-only. The table view and Policies tab show no "RLS disabled" warning. | `dummy.mjs` steps `ANON insert notes` → 1 row, `ANON delete note` → no error. `pg_class.relrowsecurity = f`; `anon` has INSERT/SELECT/UPDATE/DELETE grants. `shots/18-policies-none-no-rls-warning.jpg` | Either apply the same `rlsEnabled: true` default to tables created via SQL Editor / raw DDL (event trigger), or show a red "RLS off — anon key has full access" banner on the table and Policies views. Docs: state plainly which creation paths enable RLS. |
| F-med | **Admin login page is prefilled with a stale default password** (`cha…`, 20 chars) while `deploy/setup.sh` generates a random 24-char one. First click → "Invalid username or password". Footer says "Use the credentials configured in your .env file" but nothing tells you the field is wrong. | `http://localhost:7130/dashboard/login`, `shots/01-login-prefilled-fails.jpg` | Don't prefill; or prefill only when `ROOT_ADMIN_PASSWORD` equals the published default. |
| F-med | **Dashboard "Install InsForge" defaults to the CLI path, which is cloud-only.** On self-host it shows `npx @insforge/cli link --project-id <project-id>` and the Claude Code prompt contains literal `<placeholder>` and `<project id>`. README line 51 says CLI is "(cloud only)". | `/dashboard/install`, `shots/03-install-page.jpg`, `shots/05-claude-code-cli-tab-placeholder.jpg` | Hide the CLI tab/command when `API_BASE_URL` isn't an insforge.app host, or default to the MCP tab (which is correct and complete). |
| F-med | **Backend prints the API key and anon key in plain text to container logs on first boot.** `docker compose logs` shows `API key generated: ik_…` and `Anon key generated: anon_…`. Logs are routinely shipped/shared. | `docker compose logs insforge` lines "API key generated", "Anon key generated" | Print "keys written to .env / visible in Dashboard → API Keys" instead of the values. |
| F-med | **Missing table returns HTTP 404 with body `{}`; SDK `error` is an empty object.** No message, no code, no `nextActions`. Compare auth errors, which carry all four. | `POST /api/database/records/nope_table` → 404 `{}`; `dummy.mjs` step `LIMIT missing table` → `error: {}` | Return `{error:"DATABASE_TABLE_NOT_FOUND", message:"table public.nope_table does not exist", statusCode:404}`. |
| F-med | **No doc page covers self-hosted project setup after `docker compose up`.** `docs.insforge.dev/quickstart` and `mcp-setup` are cloud-only; the framework guides hardcode `https://your-project.us-east.insforge.app` and never mention env vars, where the anon key lives on self-host, or how to create a table without the CLI. I found the SQL Editor by clicking around. | `docs.insforge.dev/quickstart.md`, `examples/framework-guides/nextjs.md`, `core-concepts/database/migrations.md` ("doesn't say whether this works on self-hosted") | One "Self-hosted quickstart" page: login → API Keys → create table (SQL Editor or MCP) → bucket → SDK env vars. |
| F-low | **Boot log says `Dashboard: http://localhost:7131`; README says open `http://localhost:7130`.** 7130 is right (7131 is the auth port, bound to 127.0.0.1). | `docker compose logs insforge` "Setup complete" block vs README line 170 | Fix the log string. |
| F-low | **Create Bucket with an empty name does nothing** — no toast, no inline error, dialog stays open. (Invalid characters are surfaced nicely.) | `/dashboard/storage` → Create Bucket → submit empty, `shots/08-bucket-empty-name-no-error.jpg` | Show the same inline error as for invalid names. |
| F-low | **SDK TypeScript storage docs don't say how to create a bucket** (SDK? dashboard? MCP?) and don't document public vs private or that the saved `url` from `upload()` stops working once the bucket goes private. | `sdks/typescript/storage.md` | Add a "Create a bucket" section and a note under `upload()` about private buckets needing `download()`/signed URLs. |
| F-low | **`signUp` with email verification off returns a session with `Email Verified: False`.** Fine for local dev, but nothing in the auth docs says verification is off by default on self-host. | `shots/17-auth-users-two.jpg`, `dummy.mjs` step `signUp` → `verify:false` | One sentence in authentication/overview. |

## What worked (+)

- (+) `curl … setup.sh | sh -s <dir>` took 2s, generated all secrets with mode 600, and printed exactly the next two commands. `API_BASE_URL` already defaulted to `http://localhost:7130`, so the "edit .env" step was a no-op for local.
- (+) `.env` carries `COMPOSE_FILE`, so plain `docker compose up -d` works. Changing `POSTGRES_PORT` and `COMPOSE_PROJECT_NAME` to dodge a local collision worked first try.
- (+) Changing `ROOT_ADMIN_PASSWORD` in `.env` and re-running `docker compose up -d` was picked up with no volume reset.
- (+) MCP tab on the Install page is correct for self-host: real key, real base URL, verify prompt.
- (+) SQL Editor: typed DDL, Run, green toast, done. Database view showed the table and rows immediately.
- (+) SDK front-page examples (`signUp`, `signInWithPassword`, `getCurrentUser`, `from().insert().select()`, `order`, `update().eq()`, `delete().eq()`, `storage.upload/download/remove`) all ran verbatim with no edits. Upload returned both `url` and `key` as documented.
- (+) Auth error objects are good: `statusCode`, `error` code, `message`, `nextActions`. Wrong password → 401 `AUTH_UNAUTHORIZED`; duplicate email → 409 `AUTH_EMAIL_EXISTS`; missing bucket → 404 `STORAGE_NOT_FOUND`; no key at all → 401 with "login to get a new token".
- (+) Postgres constraint errors surface through the SDK with `code`, `details`, `hint`, `message`.
- (+) Config change mid-flight: flipping the bucket to private in the dashboard took effect instantly — the running dummy's `<img>` went 401, anon `download()` → 403 `FORBIDDEN`, signed-in user `download()` still returned bytes.
- (+) Bucket name validation (lowercase/digits/hyphen/underscore) is surfaced inline and as a toast.

## Scenarios

| # | Scenario | Result | Evidence |
|---|---|---|---|
| 1 | Happy path: sign up → 3 inserts → select → update → upload → public fetch → download | PASS | `dummy.mjs` transcript, `shots/12`, `shots/13` |
| 2 | Limits: wrong password, duplicate signup, missing table, missing bucket, no key | PASS (usable errors) except missing table → `{}` | `dummy.mjs` transcript |
| 3 | Config change: bucket public → private while dummy runs | PASS | `shots/14`, `shots/15`, `probe2` transcript |
| 4 | Catalog CRUD (operator): create table via SQL Editor, create/edit bucket, invalid bucket name, empty name | PASS with 1 F-low (empty name silent) | `shots/07`–`10` |
| 5 | Anon-key isolation on a SQL-Editor table | **FAIL** — anon can insert and delete | `dummy.mjs` `ANON *` steps, `shots/18` |
| — | MCP install (`npx @insforge/install --client claude-code …`) | NOT TESTED — writes to the host's global agent config | — |
| — | Edge Functions, Realtime, Model Gateway (needs provider key), Compute, Sites | NOT TESTED | — |

## Environmental deviations (not friction)

- `POSTGRES_PORT=5433` and `COMPOSE_PROJECT_NAME=crashdummy-insforge` — host already runs Postgres on 5432.
- `ROOT_ADMIN_PASSWORD` replaced with a known test value because this sandbox can't read generated secrets into the session. A real user just opens `.env`.
- Browser file-upload used a JS-injected `File`; the automation tool can't pick files from the workspace.

## The dummy

the `acme-notes` dummy (`~/crash-dummies/acme-notes/` locally) — Next.js 16 app, Composition: `lib/insforge.ts`, `app/page.tsx` (sign up / sign in),
`app/notes/page.tsx` (list, insert with image upload, delete), `dummy.mjs` (headless flows incl. limits and anon
isolation). Config via `.env.example`. Clean enough to offer as an `examples/nextjs-notes` candidate once the anon
finding is settled (the example would currently demo an open table).

## Share block

Not offered. The run is honest, not shareable-good: the core promise (auth-scoped data) didn't hold on the
documented self-host path.
