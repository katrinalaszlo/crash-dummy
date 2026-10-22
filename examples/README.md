# Example readouts

Real cold-start runs, docs only, no source read. Each folder has the markdown readout and the
self-contained HTML report (open it in a browser).

| Date | Target | TTFS | Headline |
| --- | --- | --- | --- |
| 2026-10-22 | [convex/convex-backend](2026-10-22-convex/crash-dummy-report.md) · [html](2026-10-22-convex/crash-report.html) | 3m 17s | Cleanest run. Stock Next.js quickstart verbatim on self-host; dashboard forgets the admin key. |
| 2026-10-22 | [InsForge/InsForge](2026-10-22-insforge/crash-dummy-report.md) · [html](2026-10-22-insforge/crash-report.html) | 10m 01s | SQL-Editor tables get no RLS; anon key can delete rows. Filed as [InsForge#2153](https://github.com/InsForge/InsForge/issues/2153). |
| 2026-10-22 | [firecrawl/firecrawl](2026-10-22-firecrawl/crash-dummy-report.md) · [html](2026-10-22-firecrawl/crash-report.html) | 15m 09s | 14 of 15 minutes compiling Rust that a published ghcr image already contains. |

`sample-crash-report.*` is the template filled with illustrative data.
