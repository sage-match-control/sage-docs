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
  event registry, and the live-delivery path (poll interval, staleness
  guards).
- **[Scoresheet pipeline](scoresheet-pipeline.md)** — CSV → Handlebars →
  Chromium → merged PDF.
- **[Auth](auth.md)** — the operator sign-in and how it interacts with the
  legacy shared-secret path.
- **[Deployment](deployment.md)** — Cloud Run setup, the version/changelog
  convention, environment variables.

## Frontend (`sage-match-control.github.io`)

- **[Control Center](control-center.md)** — the single-page operator
  console: config resolution, theming, the five tabs (including the Awards
  tab's podium derivation, bye/walkover handling, and Canvas 2D image
  export).
- **[Schedule board](schedule-board.md)** — the venue wall display.
- **[Tournament Calculator](tournament-calculator.md)** — the dual-meet math
  fixes and its PWA (installable, offline) setup.

## Data (`event-data`)

- **[Event registry schema](event-data-config.md)** — `config/events.json`'s
  shape and validation rules.

## Operations

- **[Adding a new event](adding-a-new-event.md)** — instantiating a template,
  wiring the sync, the day-of rehearsal.
