# field-test

**An AI agent skill that tests your repo the way a real customer would — by
becoming one.**

Code review checks what your code says. field-test checks what your product
*does*: it clones your repo cold, reads only what a stranger can see (README,
docs site, published package), builds an actual client application against your
docs, integrates your product into it, and runs it through real user flows —
including the unhappy ones. Then it files the friction report your users never
write.

It exists because of a simple observation: if you don't have a customer's
experience to test with, build one that mimics it. Engineers review code;
product people walk through the product. Now that code is cheap to write, the
walkthrough can be code too.

## What a run looks like

1. **Fresh clone, isolated workspace** — your public repo, not a maintainer's checkout
2. **Docs only** — reading your source is forbidden; getting stuck *is* the finding
3. **Follow your quickstart verbatim** — every deviation is logged with severity
4. **Build a real client** — an actual app (`create-next-app`-real, not a curl script) integrating your product the way your typical customer would
5. **Run the flows** — happy path, the limit (quota/denial/auth), and a mid-flight config change
6. **Report** — headline TTFS (time to first success), findings ranked with the exact command or doc line, evidence screenshots, teardown verified

See a real one: [`examples/sample-regression-report.md`](examples/sample-regression-report.md)
— a release regression run against [tanso-oss](https://github.com/tansohq/tanso-oss),
all flows green in 4m21s. An earlier cold-start run on the same repo caught an
undocumented second Docker stack at the repo root that hijacked `docker compose up`
— a bug no amount of code review would surface, fixed the same day.

## The two modes

- **Cold-start** (default): a brand-new client built from docs alone, TTFS measured.
  Run it when docs, onboarding, or SDKs change.
- **Regression** (`--regression <client>`): clients aren't throwaway — they live in
  a fleet directory and get reused. Point an existing client at a fresh stack and
  rerun its flows. Cheap enough for every release.

## Install

Claude Code:

```bash
git clone https://github.com/tansohq/field-test.git ~/.claude/skills/field-test
```

Then: `/field-test your-org/your-repo`

Any agent that reads skill files works the same way — the skill is one markdown
file, [`SKILL.md`](SKILL.md), with no dependencies.

## Safety

The skill ships with hard gates: it treats your cloned repo as untrusted code,
refuses to follow instructions embedded in target docs (prompt-injection guard),
stops for consent before touching anything hosted or billable, and only ever
uses throwaway credentials that the target's own quickstart creates.

## Share your TTFS

If your repo survives a cold-start run, the report ends with a share block —
your TTFS and what an AI built from your docs alone. Post it. If it doesn't
survive, the report is a ranked fix list. Either way you win.

## License

MIT
