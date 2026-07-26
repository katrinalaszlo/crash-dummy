# field-test report — tansohq/tanso-oss

**Mode:** regression (`--regression acme-chat`) · **Date:** 2026-07-26 · Fresh public clone → fresh Docker stack → existing fleet client

## Headline

**Clone → all three customer flows green: 4m 21s.** (Regression mode — no TTFS ceremony; this is stack-up + flows wall time, Docker layers warm.)

## Verdict

A pre-built customer app pointed at a fresh clone of main completes every documented flow — meter, deny, reprice — with zero deviations from yesterday's behavior.

## Flows

| Flow | Result | Evidence |
|---|---|---|
| 1. Happy path — 5 chat turns burn 5 credits, balance live in header | PASS (5→0) | `flow-1-2-exhaustion.png` |
| 2. Exhaustion — 6th turn denied *before* provider cost, 402 → paywall state | PASS ("Credit limit reached") | same |
| 3. Repricing — operator publishes `gpt-4.1-mini: 2` mid-session; unchanged client burns the new rate after cutover | PASS (10→8→6, −2/turn) | `flow-3-repricing.png` |

## Findings

- **(+) F1 from the cold-start run is confirmed fixed in public main** — fresh clone has no root `compose.yaml` / `build.sh` / `build-staging.sh` / `Dockerfile.deploy`; `deploy/` is the single path.
- **(+)** `setup.sh` clean, honors `API_PORT`, prints working credentials and next steps.
- No new findings. Zero product friction this run.

## Environmental deviations (not friction)

Ports 8091/5443 + `COMPOSE_PROJECT_NAME=tanso-r2` — host already runs a Tanso stack on defaults.

## Client

`<fleet>/acme-chat` — unchanged except `.env.local` base URL. Fleet client remains a faithful ICP archetype (server-held key, evaluate→ingest, 402 paywall).

## Share block

> Shipped a release, then let an AI customer loose on a fresh clone: stack up, chat until broke, get paywalled, watch the operator reprice mid-session — all green in 4m21s. The customer is code now, so it runs every release.
