# Specs

Design specs written before (or while) a feature was built — the reasoning
behind non-obvious decisions, worked examples, and the divergences section
each one keeps up to date when the built feature departs from what's written
here. These are reference material, not onboarding reading: start with
[Features & Usage](../features/README.md) or [Technical](../technical/README.md)
instead, and come here when you need the *why* behind a specific piece.

## How this folder is organised

By **build status**, one subfolder each:

| Folder | Means |
| --- | --- |
| [`implemented/`](implemented/README.md) | Built and in use. Read these to understand something that exists |
| [`in-progress/`](in-progress/README.md) | Partly built — some of it is running, some isn't. Read the spec's own status line for which |
| [`not-started/`](not-started/README.md) | Nothing built. These are proposals and plans |
| [`archived/`](archived/README.md) | Will not be built: superseded or dropped. Kept for their reasoning |

### Citing a spec

A spec's status folder is **not part of its identity.** Filenames are unique
across all four folders, and that is what lets a spec change status without
breaking every reference to it.

| Citing from | Write |
| --- | --- |
| Anywhere **outside** `sage-docs` — code comments, CLAUDE.md files, READMEs | `sage-docs/docs/specs/.../<name>-spec.md` |
| **Inside** `sage-docs` | The real path, e.g. `implemented/<name>-spec.md` — mkdocs has to resolve it |

The `...` stands in for whichever status folder the spec is in today. It keeps
the reference correct forever, still greps by filename, and avoids `*/`, which
would terminate the `/** */` block comments these references sit in.

### Moving a spec between folders

Because of the rule above, this is confined to `sage-docs`:

1. `git mv` it into the new folder.
2. Update its **own status line** — the folder is filing, the spec is the
   record.
3. Update this index, and the READMEs of both the folder it left and the one
   it joined.
4. Update `mkdocs.yml`'s nav.
5. Fix any real markdown link to it — `grep -rn "<name>-spec.md" docs/`. A link
   from another folder needs `../<folder>/`.

Nothing outside `sage-docs` should need touching. If a grep of the other three
repos turns up a folder-qualified path, that reference is the bug — rewrite it
to the `...` form rather than repointing it.

---

## Implemented

### Sync & live data

- **[Runtime-fetched sync config](implemented/sync-config-runtime-spec.md)** —
  why `event-data/config/events.json` is fetched at runtime instead of
  committed to `sage-tools-api`.
- **[Sync script configuration](implemented/sync-script-configuration-spec.md)**
  — why `sheets-sync.gs` keeps its per-workbook config in Script Properties,
  and the validation the setup dialog runs before saving it.
- **[sage-tools-api test suite](implemented/sage-tools-api-test-suite-spec.md)** —
  unit and integration tests (and an opt-in PDF end-to-end) that pin what the
  service does today, against an in-memory fake of GitHub, Google Sheets and the
  live Worker, built with no production change. Kept current: every API change
  updates its tests in the same commit, enforced by guard tests and a pre-push
  hook. The gate for the architecture spec, which is now open.
- **[Immediate sync](implemented/immediate-sync-spec.md)** — every sync
  measured from the edit, a sync that loses a GitHub commit race re-read and
  retried instead of dropped, and Apps Script syncing straight from the edit
  under a document lock instead of a delayed trigger. Both delivery specs
  build on it.

### Control Center

- **[Match Control console](implemented/match-control-console-spec.md)** — the
  central operator console (written before it was renamed Control Center).
- **[Awards tab](implemented/awards-podium-tab-spec.md)** — podium finishers
  and image export.
- **[Schedule screen](implemented/schedule-screen-spec.md)** — the venue wall
  display.
- **[Facility progress](implemented/facility-progress-spec.md)** — matches
  done, matches left and estimated finish per facility, on Live Matches and
  in Mission Control.

### Scoresheet Generator

- **[Event picker](implemented/scoresheet-event-picker-spec.md)** — sourcing
  the matches CSV from a published `event-data` snapshot instead of a file
  upload.

### Tournament Calculator

- **[Dual-meet fixes](implemented/calculator-dual-meet-spec.md)** — the
  calculator's dual-meet-specific timing math.
- **[PWA](implemented/calculator-pwa-spec.md)** — installable, offline setup.

### Bracket Generator

- **[Bracket Generator](implemented/bracket-generator-spec.md)** — promoting
  the per-event bracket draw page into one evergreen tool, with an optional
  event name.
- **[Verifiable draw](implemented/bracket-generator-verifiable-draw-spec.md)**
  — proving a draw wasn't rigged: a seed, a hash sort anyone can re-check on
  any SHA-256 site, and a plain-language *How it works* dialog.
- **[Bracket draw name import](implemented/bracket-draw-name-import-spec.md)**
  — uploading the tool's text exports into a generated workbook, so each
  category's rosters and its `STEP 3` codes come from the verifiable draw
  instead of being typed and hand-shuffled. Also carries the workbook's menu
  route into the tool (§11) and a dual meet's in-sheet roster shuffle (§12).
  Replaces the retired Bracket Generator workbook handoff spec.
- **[Keep-apart groups](implemented/bracket-generator-keep-apart-spec.md)** —
  keeping chosen pairs out of the same bracket while the draw stays checkable
  from the seed.

### Dual Meet Sheet Generator

- **[Sheet generator](implemented/dual-meet-sheet-generator-spec.md)** —
  Phase 1: category tabs, `Variables`, `Title`, `Reference for Players`.
- **[Schedule generator](implemented/dual-meet-schedule-generator-spec.md)** —
  Phase 2: the `SCHEDULE` tab.
- **[Readouts generator](implemented/dual-meet-readouts-generator-spec.md)** —
  Phase 3: `Court Control`, `Timeline`, `CSV`, `STANDINGSCSV`.

### Standard Tournament Generator

- **[Standard Tournament Master](implemented/standard-tournament-master-spec.md)**
  — the dual-meet generator's counterpart for open-entry tournaments: one
  workbook per facility per day, uneven round-robin brackets, and a
  forward-propagating single-elimination ladder.

### Events & templates

- **[Event site templates](implemented/event-templates-spec.md)** — the
  dual-meet and standard-tournament templates new events are instantiated
  from.
- **[Pickle & Friends × 1Bataan United Picklers dual meet](implemented/pnf-x-bup-dual-meet-spec.md)**
  — the event that drove the dual-meet template's first real run.
- **[Event attendance](implemented/event-attendance-spec.md)** — a
  staff check-in page writing each player's arrival into the venue
  workbook through an Apps Script web app; Pickle for Sight first.
- **[Attendance for every event](implemented/multi-event-attendance-spec.md)** —
  staff check-in for any event, marked in Control Center or on a desk page, one
  check-in per person. `sage-tools-api` writes each workbook's `ATTENDANCE` tab
  as a service account and keeps the roster current after every sync. In use at
  Piggleball and PickleDrive.
- **[Team tournament](implemented/pickledrive-club-anniversary-team-tournament-spec.md)** —
  a third event type for team events: named teams, four-match matchups, group
  stage then playoffs. PickleDrive Club One Year Celebration ran on 3 October 2026 on
  the hand-built prototype. The template and the workbook generator are separate specs
  under Not started.
- **[Team workbook recalculation](implemented/team-workbook-stack-cache-spec.md)**
  — why PickleDrive's workbook took 10–58 s to read on event day, and the
  by-hand fix applied on 2026-10-05: a hidden, non-volatile `StackCache` tab the
  named functions read instead of rebuilding `SCHEDULE`'s stacks thousands of
  times per edit, a matchup family that reads `MatchLookup`'s rows, and the
  unused functions removed.
- **[CLSO Pickle for Sight](implemented/pickle-for-sight-spec.md)** — the
  first standard-template event site: one day across two venues (PCPH Main
  and Annex), three divisions × three events. Ran 27 September 2026.
- **[Piggleball Chairman's Cup](implemented/piggleball-chairmans-cup-spec.md)**
  — the event site for NATFED's 1st Piggleball Chairman's Cup, part of the
  Pig Sports Festival: one venue (Centro Atletico, 3 courts), Novice
  Open Doubles plus Intermediate Men's and Mixed Doubles. Standard
  template, re-skinned from the pubmat. Ran 3 October 2026.

---

## In progress

- **[Live push delivery](in-progress/durable-object-push-spec.md)** — a
  Cloudflare Durable Object that pushes each snapshot to open pages over
  WebSockets, with GitHub kept as archive and fallback. Deployed and switched
  on; the real-workbook checks are still open.
  - **[Plain-language explainer](in-progress/durable-object-push-explainer.md)**
    — the same, without the implementation detail.
- **[Score entry and scorer links](in-progress/control-center-score-entry-spec.md)**
  — click a match in Match Finder, enter the scores, review (the winner named in large
  text), save. The API writes the match's two `SCHEDULE` score cells, refuses if the
  sheet changed meanwhile, and publishes at once. Scorer staff use the same dialog on a
  per-event scorer page, through 24-hour links that a Mission Control switch can stop.
  Built and tested against fakes; the real-workbook checks (§11.2) are still open.

---

## Not started

- **[sage-tools-api architecture hardening](not-started/sage-tools-api-architecture-spec.md)** —
  revised against 2.8.0. The service moves to ports-and-adapters, checked
  against SOLID. The conflict retry is written once, behind two delivery
  strategies and a `SnapshotPublisher` selector. There is one error handler, one auth-middleware module,
  and a validated config with a composition root that the tests share.
  Outbound clients and the event registry get their own folders. Release
  3.0.0 adds one REST surface, `/v3`, with a twin of every route (legacy and
  `/v1`), and every client moves onto it. Every existing URL is kept, frozen.
  `/ping` is unchanged.
- **[Site test suite](not-started/site-test-suite-spec.md)** — Control
  Center, the current event pages and the attendance desk pages tested in a
  real browser on fixture data, with every hand-copied rule (the live
  channel and attendance client blocks, played/BYE, the team-event rules,
  team rosters, go-live) checked across its copies. No page changes. It
  provides the dry run's rendering layer.
- **[Calculator team format](not-started/calculator-team-format-spec.md)** — a third Format option
  in the Tournament Calculator: one team competition planned in matchups, with the group stage,
  quarterfinal-to-final playoffs, slots and finish time worked out from teams, groups and courts.
- **[Team tournament event-site template](not-started/team-tournament-template-spec.md)**
  — extract `_templates/team-tournament-template/` from PickleDrive's pages, S.A.G.E.-themed
  like the other two templates, with the event's constants generalised.
- **[Team Tournament Master](not-started/team-tournament-master-spec.md)** — the
  workbook generator for team events, phased, starting from the optimised workbook.
- **[Automated dry run](not-started/automated-dry-run-spec.md)** — idea
  only: run the Control Center runbook's rehearsal with one command,
  including Puppeteer editing the real facility sheet so the onEdit trigger
  is tested too. Its rendering checks come from the site test suite.
- **[Court Control from Control Center](not-started/control-center-court-control-spec.md)**
  — outline only: an operator puts a match on a court, or clears it, from Live
  Matches, through the same write-then-publish path as score entry. Waits for
  score entry's real-workbook checks.

---

## Archived

- **[Fast data delivery](archived/fast-data-delivery-spec.md)** — replacing
  the GitHub Pages build step in the live data path with Cloudflare R2, and
  full-payload polling with pointer polling. **Superseded** by Live push
  delivery, which was built instead; it will not be built.
  - **[Plain-language explainer](archived/fast-data-delivery-explainer.md)**
    — the same, without the implementation detail.
