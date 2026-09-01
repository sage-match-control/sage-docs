# Architecture

S.A.G.E. is three independent git repos, not one monorepo.

| Repo | What it is |
| --- | --- |
| [`sage-tools-api`](https://github.com/sage-match-control/sage-tools-api) | Node/Express backend on Google Cloud Run. Two features: scoresheet PDF generation and the Google Sheets → GitHub live-data sync. |
| [`sage-match-control.github.io`](https://github.com/sage-match-control/sage-match-control.github.io) | GitHub Pages static site. Self-contained HTML pages (no build step, no framework) for the public tools and per-event pages. |
| [`event-data`](https://github.com/sage-match-control/event-data) | Shared GitHub Pages target every event's sync writes snapshots to, and the runtime-fetched event/day/facility registry. |

## Data flow

```
Facility Google Sheet  (one per venue per tournament day)
   |  installable onEdit trigger, debounced  (scripts/sheets-sync.gs)
   v
POST /sync/:day?facility=<name>   on Cloud Run   (X-Sync-Secret header)
   |  SheetsCsvFetcher (default) or GvizCsvFetcher (?method=csv fallback)
   |  merge with published snapshot — never drop a facility on failure
   v
GitHub Contents API commit -> event-data : <event-key>/data/<day>.json
   |  GitHub Pages redeploys on push (no cache-purge step)
   v
Event page fetches https://sage-match-control.github.io/event-data/<event-key>/data/<day>.json
```

Full detail on each hop: [Sync pipeline](sync-pipeline.md).

The scoresheet feature is independent of all of the above: the site's
`tools/scoresheet-generator.html` posts a CSV to
`/scoresheets/generate/stream` and gets NDJSON progress lines plus a base64
PDF back. See [Scoresheet pipeline](scoresheet-pipeline.md).

## Why the registry lives in `event-data`, not in code

Originally, the event/day/facility registry (which spreadsheets belong to
which day, display labels, etc.) was a hardcoded object in
`sage-tools-api`'s source. That meant adding an event or fixing a wrong
sheet ID required a full Cloud Build image rebuild — `gcloud run deploy
--source .`, which rebuilds a Dockerfile that installs Chromium.

It now lives at `event-data/config/events.json`, fetched by
`sage-tools-api` at runtime and cached with a TTL (`SyncConfigStore`). Adding
an event or fixing a sheet ID is now a commit to `event-data`, live within
about a minute, with **no redeploy**. See [event registry
schema](event-data-config.md) for the shape, and [sync
pipeline](sync-pipeline.md) for how the store fetches/caches/validates it.

## Things that must be kept in sync by hand

There's no single source of truth enforcing these — they're conventions,
not code:

- `event-data/config/events.json`'s `events[<event>].days` ↔ the `DAYS`
  array in that event's page ↔ each of its spreadsheets' `DAY_KEY` /
  `FACILITY_NAME` in `sheets-sync.gs`. Facility names are compared exactly
  and are case-sensitive.
- Adding a scoresheet type = a new `templates/<name>.{html,css}` pair in
  `sage-tools-api` **and** an entry in `ScoresheetConfig.mjs`. Nothing else
  needs touching.

## Versioning

`sage-tools-api`'s `package.json` version is bumped for *any* code change,
however small — patch for a fix, minor for a new endpoint/feature, major for
a breaking change. It's the only way to tell what's actually running on
Cloud Run: `GET /ping`'s `X-App-Version` header reads straight from it.
See [Deployment](deployment.md).
