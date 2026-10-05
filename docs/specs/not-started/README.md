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
| [Calculator team format](calculator-team-format-spec.md) | A third **Format** option in the Tournament Calculator: one competition (no categories) planned in matchups: teams, groups, matches per matchup, how many play at once, how many reach the playoffs, an optional break. Counts matchups and matches, slots and the finish time (reproducing PickleDrive's published 4:05 PM), with its own CSV `format` value and `.xlsx` layout. Calculator only | Nothing; the generator handoff waits for the Team Tournament Master |
| [Team tournament event-site template](team-tournament-template-spec.md) | `_templates/team-tournament-template/` extracted from PickleDrive's pages, S.A.G.E.-themed like the standard and dual-meet templates: event values tokenised, the hard-coded pairs, stages, group count and advancing count generalised. A proposal with its open decisions listed | Nothing hard. The workbook recalculation fix is applied |
| [Team Tournament Master](team-tournament-master-spec.md) | The workbook generator for team events, like the dual-meet and standard generators: a master workbook carrying the named functions, and a `SAGE -> Generate event tabs` that builds `Teams`, `MatchUps`, `SCHEDULE`, `CSV`, `STANDINGSCSV` and the rest. Phased; a proposal with its open decisions listed | Nothing. [Team workbook recalculation](../implemented/team-workbook-stack-cache-spec.md) is applied; the master is built from that workbook |
| [Automated dry run](automated-dry-run-spec.md) | Idea only. A `verify-event-dry-run.mjs` script: preflight checks, rendering checks against a generated fixture, and a Puppeteer rehearsal that edits the real facility sheet | The spikes in its §4, starting with Google sign-in from an automated browser |
| [Court Control from Control Center](control-center-court-control-spec.md) | Outline only. A **Set match** action per court in Live Matches: the API writes that court's one match-number cell in `Court Control`, checked against what the console showed, then publishes the day. Later, the score dialog offers to put the next match on a court it frees | [Score entry](../in-progress/control-center-score-entry-spec.md)'s real-Google checks (C1–C8) |
