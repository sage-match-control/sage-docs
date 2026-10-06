# Technical

Architecture, code, and deployment for S.A.G.E. If you want to know what a
feature *does* rather than how it's built, its counterpart lives in
[Features & Usage](../features/README.md) and is linked from the bottom of
each page here.

## Start here

- **[Architecture](architecture.md)** — the three repos, the data flow
  between them, and how a score gets from a spreadsheet to a phone.

## Backend (`sage-tools-api`)

- **[Sync pipeline](sync-pipeline.md)** — Sheets → GitHub, the runtime-fetched
  event registry, the live-delivery path (the live push Worker, the
  GitHub archive and polling fallback, the operator switch), and how score
  entry writes a score and publishes it.
- **[API](api.md)** — the one REST surface, `/v3`, its conventions, errors and
  routes, and the frozen legacy and `/v1` URLs it replaces for new clients.
- **[Scoresheet pipeline](scoresheet-pipeline.md)** — CSV → Handlebars →
  Chromium → merged PDF.
- **[Auth](auth.md)** — the operator sign-in, how it interacts with the
  legacy shared-secret path, and the desk and scorer tokens.
- **[Deployment](deployment.md)** — Cloud Run setup, the version/changelog
  convention, environment variables.

## Bound Apps Script (in `sage-tools-api`, but not the API)

All four live in `sage-tools-api/apps-script/` for versioning, run inside a
Google Sheet, and ship by being pasted into that sheet's own script project —
changing any of them is not a deploy.

- **[Dual Meet Sheet Generator](dual-meet-sheet-generator.md)** — builds a
  dual meet's category tabs from a Tournament Calculator CSV.
  - **[Named Function library](named-function-library.md)** — the 23
    workbook-level `LAMBDA` definitions those generated formulas call. Not
    Apps Script: they live in the workbook itself, under
    Data → Named functions.
- **[Standard Tournament Generator](standard-tournament-generator.md)** —
  builds one facility-day's standard-tournament workbook from a calculator
  CSV.
- **[Event attendance](event-attendance.md)** — `src/attendance/` in
  `sage-tools-api` (roster, the `ATTENDANCE` tab, desk tokens, the `/v3`
  routes and their frozen `/v1` twins), the shared client block, and Pickle for
  Sight's earlier `attendance.gs` version.
- The sync trigger (`sheets-sync.gs`) is covered in
  [Sync pipeline](sync-pipeline.md).

## Frontend (`sage-match-control.github.io`)

- **[Control Center](control-center.md)** — the single-page operator
  console: config resolution, theming, the tabs (including the Attendance
  tab around the shared attendance client, score entry and its shared dialog,
  the Awards
  tab's podium derivation, bye/walkover handling, and Canvas 2D image
  export).
- **[Scorer page](scorer-page.md)** — the per-event page a scorer link opens:
  its template, start-up checks, token storage and data loading.
- **[Schedule board](schedule-board.md)** — the venue wall display.
- **[Tournament Calculator](tournament-calculator.md)** — the dual-meet math
  fixes and its PWA (installable, offline) setup.
- **[Bracket Generator](bracket-generator.md)** — the one-palette-source
  arrangement that keeps the canvas export in step with the page.

## Data (`event-data`)

- **[Event registry schema](event-data-config.md)** — `config/events.json`'s
  shape and validation rules.

## Operations

- **[Adding a new event](adding-a-new-event.md)** — instantiating a template,
  wiring the sync, the day-of rehearsal.
