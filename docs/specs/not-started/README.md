# Not-started specs

Specs with **nothing built**. These are proposals and plans: the reasoning is
worth keeping, but none of it describes code that exists.

Read them for design intent, not as documentation. Every cell reference, file
path and function name in here is a *proposal* — do not go looking for it in
the repos.

Index and status-change procedure: [`../README.md`](../README.md).

| Spec | Proposes | Blocked on |
| --- | --- | --- |
| [sage-tools-api architecture hardening](sage-tools-api-architecture-spec.md) | After the test suite: the conflict retry written once, a snapshot-store interface and publishing strategies so `SyncService` stops knowing about GitHub and the Worker, one error handler and a validated config module, a constant-time secret check, `/v1` resource-style routes beside attendance's, with every current URL kept, `apps-script/` split from `scripts/`, and a Cloud Build file filter. `/ping` stays exactly as it is | The [test suite spec](../implemented/sage-tools-api-test-suite-spec.md) is built and green (the gate is open). Nothing here is built |
| [Site test suite](site-test-suite-spec.md) | `_tests/` in the site repo (never published): consistency checks on page text (the `LIVE CHANNEL` and `ATTENDANCE CLIENT` blocks, shared constants, leftover tokens), parity tests that run every copy of a hand-copied rule (played/BYE, team-event rules, team rosters, pair labels, go-live) in its own page and compare answers, and Playwright page tests of Control Center, the current event pages, the attendance desk pages and the templates on fixture data, with the live Worker and clock faked. No page changes. Owns the runbook checks the dry run reuses | Nothing. Not before the 3 October events. Nothing is built |
| [Team tournament event-site template](team-tournament-template-spec.md) | `_templates/team-tournament-template/` extracted from PickleDrive's pages, S.A.G.E.-themed like the standard and dual-meet templates: event values tokenised, the hard-coded pairs, stages, group count and advancing count generalised. A proposal with its open decisions listed | Nothing hard; better after the workbook recalculation fix |
| [Team Tournament Master](team-tournament-master-spec.md) | The workbook generator for team events, like the dual-meet and standard generators: a master workbook carrying the named functions, and a `SAGE -> Generate event tabs` that builds `Teams`, `MatchUps`, `SCHEDULE`, `CSV`, `STANDINGSCSV` and the rest. Phased; a proposal with its open decisions listed | [Team workbook recalculation](team-workbook-stack-cache-spec.md), which is its phase 1 |
| [Team workbook recalculation](team-workbook-stack-cache-spec.md) | By hand in the PickleDrive workbook: build `SCHEDULE`'s five stacked columns once in a hidden `StackCache` tab and point the named functions at it (about 9,200 `STACKBLOCKS` rebuilds per edit down to 5), de-duplicate the matchup family's lookups, remove the 13 unused named functions. Verified by comparing values against a named version and by one sync's `fetch` time | Nothing. Before the team template is extracted |
| [Automated dry run](automated-dry-run-spec.md) | Idea only. A `verify-event-dry-run.mjs` script: preflight checks, rendering checks against a generated fixture, and a Puppeteer rehearsal that edits the real facility sheet | The spikes in its §4, starting with Google sign-in from an automated browser |
| [Fast data delivery](fast-data-delivery-spec.md) | **Superseded** by [Live push delivery](../in-progress/durable-object-push-spec.md), which was built instead. Cloudflare R2 in the live data path plus pointer polling, cutting edit-to-visible from ~35–40s to ~5–7s. Not going to be built | — |
| [Fast data delivery — explainer](fast-data-delivery-explainer.md) | The same, in plain language. No instructions to implement | — |
