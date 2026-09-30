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
| [Automated dry run](automated-dry-run-spec.md) | Idea only. A `verify-event-dry-run.mjs` script: preflight checks, rendering checks against a generated fixture, and a Puppeteer rehearsal that edits the real facility sheet | The spikes in its §4, starting with Google sign-in from an automated browser |
| [Fast data delivery](fast-data-delivery-spec.md) | Cloudflare R2 in the live data path plus pointer polling, cutting edit-to-visible from 40–60s to ~5–7s. The 3s Apps Script debounce (step 16) is already in `sheets-sync.gs`; nothing else is built | Nothing technical. A Cloudflare account and a custom domain (§9.4) |
| [Fast data delivery — explainer](fast-data-delivery-explainer.md) | The same, in plain language. No instructions to implement | — |
