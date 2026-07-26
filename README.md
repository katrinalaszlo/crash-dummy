# 🚗 crash-dummy

**Take your repo for a test drive.**

It builds a test user with a real application, integrates your product the way
your docs teach, and runs the flows your customers will actually run. Every
stumble gets logged with the exact command or doc line that caused it.

## How it works

1. **Cold clone:** your public repo, in a fresh temp workspace
2. **Docs only:** reading your source is off-limits; getting stuck is itself a finding
3. **Build a client:** a real application (real framework, real UI, real env config) that integrates your product exactly as your docs describe
4. **Run the flows:** the happy path, the enforced limit (quota, denial, auth), and a server-side config change mid-run
5. **Report:** a self-contained `crash-report.html` with the headline metric, per-flow results and screenshots, and findings ranked by severity, plus a markdown twin for CI

## Example report

<img src="assets/sample-report.png" alt="Sample crash report: 4m21s, all scenarios green, evidence screenshots of the test client" width="820">

From a regression run against [tanso-oss](https://github.com/tansohq/tanso-oss):
all flows green in **4m 21s**. An earlier cold-start run on the same repo found
an undocumented second Docker stack at the repo root that hijacked
`docker compose up`. Fixed the same day. Full artifacts:
[HTML report](examples/sample-crash-report.html) ·
[markdown twin](examples/sample-crash-report.md).

## Modes

- **Cold-start** (default): builds a new client from your docs alone and
  measures TTFS, the time from clone to first success. Run it when docs,
  onboarding, or SDKs change.
- **Regression** (`--regression <client>`): reuses a client from previous runs
  (kept in `~/crash-dummies/`) against a fresh stack. Fast enough to run every
  release.
- **PR mode** (`--pr <number>`): posts the results as a comment on your PR,
  with the TTFS delta vs the previous run and a link to the full report.

It states the estimated cost (time and tokens) before starting.

## Install

Claude Code:

```bash
git clone https://github.com/katrinalaszlo/crash-dummy.git ~/.claude/skills/crash-dummy
```

Then: `/crash-dummy your-org/your-repo`

Any agent that reads skill files works the same way. The whole skill is one
markdown file, [`SKILL.md`](SKILL.md), with no dependencies.

## Safety

The cloned repo is treated as untrusted code. Instructions embedded in target
docs are ignored and reported. Anything hosted or billable requires one
explicit up-front consent, and only throwaway credentials are ever used.

## License

MIT
