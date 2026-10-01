# Not-started specs

Specs with **nothing built**. These are proposals and plans: the reasoning is
worth keeping, but none of it describes code that exists.

Read them for design intent, not as documentation. Every cell reference, file
path and function name in here is a *proposal* — do not go looking for it in
the repos.

Index and status-change procedure: [`../README.md`](../README.md).

| Spec | Proposes | Blocked on |
| --- | --- | --- |
| [Attendance for every event](multi-event-attendance-spec.md) | Idea only. Attendance writes move to `sage-tools-api` via a service account, keyed by `events.json`, with swaps reconciled on every sync. No per-workbook Apps Script | The open questions in its §5 |
| [sage-tools-api test suite](sage-tools-api-test-suite-spec.md) | Unit tests for every module and integration tests for every route and the whole sync pipeline (against an in-memory fake of GitHub, Google Sheets and the live Worker, with deterministic race tests), an opt-in PDF end-to-end, ten sabotage checks proving the suite can fail. Pins today's behaviour; no production change. The gate for the architecture spec. Then kept current: every API change updates its tests in the same commit, enforced by guard tests, a pre-push hook and a rule in the root `CLAUDE.md` | The live push code on `main`. Nothing is built |
| [sage-tools-api architecture hardening](sage-tools-api-architecture-spec.md) | After the test suite: the conflict retry written once, a snapshot-store interface and publishing strategies so `SyncService` stops knowing about GitHub and the Worker, one error handler and a validated config module, a constant-time secret check, `/v1` resource-style routes with every current URL kept, `apps-script/` split from `scripts/`, and a Cloud Build file filter. `/ping` stays exactly as it is | The [test suite spec](sage-tools-api-test-suite-spec.md) built and green. Nothing is built |
| [Site test suite](site-test-suite-spec.md) | `_tests/` in the site repo (never published): consistency checks on page text (the `LIVE CHANNEL` block, shared constants, leftover tokens), parity tests that run every copy of a hand-copied rule (played/BYE, team-event rules, pair labels, go-live) in its own page and compare answers, and Playwright page tests of Control Center and the current event pages and templates on fixture data, with the live Worker and clock faked. No page changes. Owns the runbook checks the dry run reuses | Nothing. Not before the 3 October events. Nothing is built |
| [Automated dry run](automated-dry-run-spec.md) | Idea only. A `verify-event-dry-run.mjs` script: preflight checks, rendering checks against a generated fixture, and a Puppeteer rehearsal that edits the real facility sheet | The spikes in its §4, starting with Google sign-in from an automated browser |
| [Fast data delivery](fast-data-delivery-spec.md) | **Superseded** by [Live push delivery](../in-progress/durable-object-push-spec.md), which was built instead. Cloudflare R2 in the live data path plus pointer polling, cutting edit-to-visible from ~35–40s to ~5–7s. Not going to be built | — |
| [Fast data delivery — explainer](fast-data-delivery-explainer.md) | The same, in plain language. No instructions to implement | — |
