# Specs

Design specs written before (or while) a feature was built — the reasoning
behind non-obvious decisions, worked examples, and the divergences section
each one keeps up to date when the built feature departs from what's written
here. These are reference material, not onboarding reading: start with
[Features & Usage](../features/README.md) or [Technical](../technical/README.md)
instead, and come here when you need the *why* behind a specific piece.

## Sync & live data

- **[Fast data delivery](fast-data-delivery-spec.md)** — the sync pipeline's
  latency budget and staleness guards.
  - **[Plain-language explainer](fast-data-delivery-explainer.md)** — the
    same, without the implementation detail.
- **[Runtime-fetched sync config](sync-config-runtime-spec.md)** — why
  `event-data/config/events.json` is fetched at runtime instead of committed
  to `sage-tools-api`.
- **[Sync script configuration](sync-script-configuration-spec.md)** — why
  `sheets-sync.gs` keeps its per-workbook config in Script Properties, and the
  validation the setup dialog runs before saving it.

## Control Center

- **[Match Control console](match-control-console-spec.md)** — the central
  operator console (written before it was renamed Control Center).
- **[Awards tab](awards-podium-tab-spec.md)** — podium finishers and image
  export.
- **[Schedule screen](schedule-screen-spec.md)** — the venue wall display.

## Scoresheet Generator

- **[Event picker](scoresheet-event-picker-spec.md)** — sourcing the matches
  CSV from a published `event-data` snapshot instead of a file upload.

## Tournament Calculator

- **[Dual-meet fixes](calculator-dual-meet-spec.md)** — the calculator's
  dual-meet-specific timing math.
- **[PWA](calculator-pwa-spec.md)** — installable, offline setup.

## Bracket Generator

- **[Bracket Generator](bracket-generator-spec.md)** — promoting the per-event
  bracket draw page into one evergreen tool, with an optional event name.

## Dual Meet Sheet Generator

- **[Sheet generator](dual-meet-sheet-generator-spec.md)** — Phase 1: category
  tabs, `Variables`, `Title`, `Reference for Players`.
- **[Schedule generator](dual-meet-schedule-generator-spec.md)** — Phase 2:
  the `SCHEDULE` tab.
- **[Readouts generator](dual-meet-readouts-generator-spec.md)** — Phase 3:
  `Court Control`, `Timeline`, `CSV`, `STANDINGSCSV`.

## Events & templates

- **[Event site templates](event-templates-spec.md)** — the dual-meet and
  standard-tournament templates new events are instantiated from.
- **[Pickle & Friends × 1Bataan United Picklers dual meet](pnf-x-bup-dual-meet-spec.md)**
  — the event that drove the dual-meet template's first real run.
