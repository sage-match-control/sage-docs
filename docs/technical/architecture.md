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

## A fourth kind of code: bound Apps Script

`sage-tools-api/scripts/` holds Google Apps Script that is versioned in that
repo but is **not part of the service** and never runs on Cloud Run. It ships
by being pasted into a spreadsheet's own bound script project, so changing
one is not a deploy and doesn't bump the API version.

| File | Bound to | Does |
| --- | --- | --- |
| `sheets-sync.gs` | each facility spreadsheet | the debounced onEdit trigger that calls `POST /sync/:day` (the diagram above) |
| `sheet-generator.gs` | the SAGE Dual Meet Master workbook | builds a dual meet's category tabs from a Tournament Calculator CSV — see [Dual Meet Sheet Generator](dual-meet-sheet-generator.md) |

They sit at opposite ends: `sheets-sync.gs` is the *entry point* to the sync
pipeline, while `sheet-generator.gs` touches no server at all and only
prepares the workbook that pipeline will later read from.

## Why the registry lives in `event-data`, not in code

The event/day/facility registry (which spreadsheets belong to which day,
display labels, etc.) lives at `event-data/config/events.json`, fetched by
`sage-tools-api` at runtime and cached with a TTL (`SyncConfigStore`).

Holding it in `sage-tools-api`'s source instead would make adding an event or
fixing a wrong sheet ID a full Cloud Build image rebuild — `gcloud run deploy
--source .`, which rebuilds a Dockerfile that installs Chromium. As a config
fetch, it is a commit to `event-data`, live within about a minute, with **no
redeploy**. See [event registry schema](event-data-config.md) for the shape,
and [sync pipeline](sync-pipeline.md) for how the store fetches, caches and
validates it.

## Things that must be kept in sync by hand

There's no single source of truth enforcing these — they're conventions,
not code:

- `event-data/config/events.json`'s `events[<event>].days` ↔ the `DAYS`
  array in that event's page ↔ each spreadsheet's day key and facility name,
  set through that workbook's **SAGE → Set up live sync** menu item and
  stored in its own Script Properties (`sheets-sync.gs`'s source is
  identical in every workbook). Facility names are compared exactly and are
  case-sensitive — setup validates both against the registry before saving.
  The shared secret is the exception to "stored in Script Properties": it
  lives in developer metadata on the spreadsheet so that copies of the Dual
  Meet Master inherit it. See
  [sync pipeline](sync-pipeline.md#how-the-secret-reaches-a-workbook).
- Adding a scoresheet type = a new `templates/<name>.{html,css}` pair in
  `sage-tools-api` **and** an entry in `ScoresheetConfig.mjs`. Nothing else
  needs touching.

## Versioning

`sage-tools-api`'s `package.json` version is bumped for *any* code change,
however small — patch for a fix, minor for a new endpoint/feature, major for
a breaking change. It's the only way to tell what's actually running on
Cloud Run: `GET /ping`'s `X-App-Version` header reads straight from it.
See [Deployment](deployment.md).
