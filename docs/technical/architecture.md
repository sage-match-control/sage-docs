# Architecture

S.A.G.E. is three independent git repos, not one monorepo, plus a Cloudflare
Worker that lives inside one of them.

| Repo | What it is |
| --- | --- |
| [`sage-tools-api`](https://github.com/sage-match-control/sage-tools-api) | Node/Express backend on Google Cloud Run. Four features: scoresheet PDF generation, the Google Sheets → GitHub live-data sync, event attendance, and score entry (`src/scores/`). Its `live-worker/` folder holds the Cloudflare Worker that pushes snapshots to open pages; that folder is not part of the Cloud Run service. |
| [`sage-match-control.github.io`](https://github.com/sage-match-control/sage-match-control.github.io) | GitHub Pages static site. Self-contained HTML pages (no build step, no framework) for the public tools and per-event pages. |
| [`event-data`](https://github.com/sage-match-control/event-data) | Shared GitHub Pages target every event's sync writes snapshots to, and the runtime-fetched event/day/facility registry. |

## Data flow

```
Facility Google Sheet  (one per venue per tournament day)
   |  installable onEdit trigger, document-locked, syncs from the edit  (apps-script/sheets-sync.gs)
   v
POST /sync/:day?facility=<name>   on Cloud Run   (X-Sync-Secret header)
   |  SheetsCsvFetcher (default) or GvizCsvFetcher (?method=csv fallback)
   |  merge with published snapshot — never drop a facility on failure
   v
Cloudflare Worker "sage-live" (live-worker/): one Durable Object per <event>/<day>
   |  stores the snapshot, pushes it to every open page over a WebSocket
   |  wss://.../live/<event>/<day>
   v
GitHub Contents API commit -> event-data : <event-key>/data/<day>.json   (archive + fallback)
   |  GitHub Pages redeploys on push (no cache-purge step)
   v
A page with no open socket fetches https://sage-match-control.github.io/event-data/<event-key>/data/<day>.json
```

Full detail on each hop: [Sync pipeline](sync-pipeline.md). GitHub stays the
archive and the fallback read path: with `LIVE_PUSH_URL` unset, or when the
Worker cannot be reached, a sync publishes through GitHub alone and pages poll
it.

The scoresheet feature is independent of all of the above: the site's
`tools/scoresheet-generator.html` posts a CSV to
`/scoresheets/generate/stream` and gets NDJSON progress lines plus a base64
PDF back. See [Scoresheet pipeline](scoresheet-pipeline.md).

Attendance is the one feature that writes into the facility workbooks. After
every sync, `sage-tools-api` brings each workbook's `ATTENDANCE` tab up to date
with its roster, and Control Center and the desk pages mark people through
`PUT /v1/days/:day/facilities/:facility/attendance/:key`. It writes as its own
service account, and attendance only ever inside `ATTENDANCE!A:G`. See
[event attendance](event-attendance.md).

Score entry (`src/scores/`) is the second: `PUT
/v1/days/:day/facilities/:facility/matches/:matchNumber/score` writes one
match's two score cells in a workbook's `SCHEDULE` tab as the same service
account, then publishes the facility itself, because an API write fires no onEdit
trigger. Operators call it from Control Center and scorer staff from the scorer
page with a scorer link. See [sync pipeline § Score entry
writes](sync-pipeline.md#score-entry-writes) and [scorer page](scorer-page.md).

## A fourth kind of code: bound Apps Script

`sage-tools-api/apps-script/` holds Google Apps Script that is versioned in that
repo but is **not part of the service** and never runs on Cloud Run. It ships
by being pasted into a spreadsheet's own bound script project, so changing
one is not a deploy and doesn't bump the API version.

| File | Bound to | Does |
| --- | --- | --- |
| `sheets-sync.gs` | each facility spreadsheet | the lock-based onEdit trigger that calls `POST /sync/:day` (the diagram above) |
| `sheet-generator.gs` | the SAGE Dual Meet Master workbook | builds a dual meet's category tabs from a Tournament Calculator CSV — see [Dual Meet Sheet Generator](dual-meet-sheet-generator.md) |
| `standard-generator.gs` | the SAGE Standard Tournament Master workbook | builds one venue-day's standard-tournament workbook from a Tournament Calculator CSV — see [Standard Tournament Generator](standard-tournament-generator.md) |
| `attendance.gs` | Pickle for Sight's live workbooks | the earlier, per-workbook attendance web app — see [Event attendance](event-attendance.md#earlier-version-pickle-for-sight). Newer events need no script for attendance |

They sit at opposite ends: `sheets-sync.gs` is the *entry point* to the sync
pipeline, while the two generators touch no server at all and only prepare
the workbook that pipeline will later read from. Both generators also add
an empty `ATTENDANCE` tab in the format the API fills. A generated workbook
carries `sheets-sync.gs` and one of the two generators, never both.

## How `sage-tools-api` is put together

### The pattern

A **modular monolith with a ports-and-adapters core, applied lightly.** One
process, sliced by feature (`src/sync/`, `src/attendance/`, `src/scores/`,
`src/scoresheets/`, `src/auth/`), with the code every feature shares in its own
folders and one place where everything is wired.

```
 Apps Script, Control Center, event pages, desk and scorer pages
                    │ HTTP
                    ▼
 DRIVING ADAPTERS   routes + syncController + auth/middleware + errorHandler
                    │ call
                    ▼
 APPLICATION        services (one use case or one small group each)
                    │ use                              │ depend only on
                    ▼                                  ▼
 DOMAIN             */domain/ (pure functions)      PORTS (JSDoc interfaces)
                                                       ▲ implemented by
                                                       │
 DRIVEN ADAPTERS    clients/, sync/publishing/ stores and deliveries, registry/
                    │ HTTP
                    ▼
                    GitHub (event-data), the live Worker, Google Sheets

 COMPOSITION ROOT   src/app.mjs builds the adapters and plugs them into the ports
```

Before the restructuring it was a feature-sliced, layered monolith with a service
layer and hand-wired constructor injection: each service was coded against the
concrete object it received, routes did their own auth and error mapping, and
`Server` imported every feature. The patterns inside it now:

- **Strategy** — the two snapshot deliveries.
- **Facade with selection** — `SnapshotPublisher`.
- **Adapter** — each snapshot store wraps one client.
- **Repository** — `SyncConfigStore`.
- **Higher-order retry policy** — `withConflictRetry` and `mergeIntoStore`.
- **Observer** — `onFacilitiesSynced`.
- **Chain of responsibility** — the Express middleware, ending in `errorHandler`.

Deliberate trade-offs: ports are JSDoc typedefs, not runtime interfaces (`//
@ts-check` checks them in the editor, and the contract tests check them at run
time); error classes carry their HTTP `statusCode` instead of a second mapping
table; `src/scoresheets/` keeps its internal shape; there is no DI container
(`createApp` is plain code).

### Parts

| Where | What |
| --- | --- |
| `index.mjs` | Reads the version, loads the config, calls `createApp`, starts the server. Nothing else. |
| `src/app.mjs` | `createApp(config, { logger, version })` builds every client, store and service, and `createRouters` builds every router and says where it is mounted. The tests call the same functions, so the wiring is written once. The scoresheet pipeline is still imported lazily, on the first scoresheet request. |
| `src/config/loadConfig.mjs` | The only reader of the environment: it turns it into a validated, frozen config. |
| `src/server/` | `Server` (middleware order, `/ping`, `/openapi.json`, mounting the routers it is given, the error handler last), `cors`, `errorHandler`, `asyncHandler`. It imports no feature. |
| `src/auth/middleware.mjs` | Every Express auth check (see [Auth](auth.md#where-the-checks-live)). |
| `src/clients/` | Every class that calls an outside service: GitHub, the live Worker, Google Sheets. |
| `src/registry/` | The event registry (`events.json`): loading, caching, validating it and writing its three switches. Sync, attendance and score entry all read it. |
| `src/<feature>/domain/` | Each feature's pure functions: no I/O, no Express, no clients. |
| `src/shared/` | `Logger`, `ConcurrencyPool`, the error classes, `parseCsv`, the constant-time `safeEqual`, and `ports.mjs`, the interfaces the parts depend on, written as JSDoc typedefs. |

**Errors.** Every error is an `AppError` carrying the HTTP status it answers
with and a stable machine-readable `code`. A route handler throws; the one
`errorHandler` turns the error into `{ error, code }` (plus any extra member
the error carries, such as a score conflict's `current`) with that status,
logs it once, and adds `WWW-Authenticate` to a 401. A foreign error with a 4xx
status, such as body-parser's malformed-JSON error, answers `400 bad_request`;
anything else is `500 internal_error`. Once a streaming route has sent its
headers the error handler writes nothing, because the route already wrote its
own error line.

**Configuration.** `loadConfig` reads every variable as trimmed text, with a
blank value counting as unset. A number outside its range, or a URL that is not
`http` or `https`, stops the service at startup with one message naming every
bad variable. An unset secret does not: it turns its feature off, and a single
startup log line names what is unset. [Deployment](deployment.md#local-development)
lists the rules.

**Ports.** Every dependency a service, delivery or store receives is typed
against a **port**: a JSDoc typedef naming only the methods that consumer calls
(`src/shared/ports.mjs`, and `src/sync/publishing/ports.mjs` for the snapshot
stores and deliveries). At run time the same full object is passed as ever; a
port documents and type-checks the dependency without wrapping it.

| Port | Implemented by | Consumed by |
| --- | --- | --- |
| `FacilityFetcher` | `SheetsCsvFetcher`, `GvizCsvFetcher` | `SyncService` |
| `JsonDocumentStore` | `GitHubPublisher` | `SyncConfigStore`, `GitHubSnapshotStore` |
| `LiveChannel`, `LiveAvailability` | `LivePublisher` | `LiveSnapshotStore`; `SnapshotPublisher`, `SyncSettingsService` |
| `RegistryReader` | `SyncConfigStore` | every service |
| `DayVisibilitySwitch`, `LivePushSwitch`, `ScoreEntrySwitch` | `SyncConfigStore` | `DayVisibilityService`, `SyncSettingsService`, `ScoreService` |
| `DayPublisher` | `SyncService` | `ScoreService` |
| `SyncedHook` | `AttendanceService.reconcileAfterSync` | `SyncService` |
| `TokenVerifier`, `Authenticator`, `DeskTokenIssuer`, `ScorerTokenIssuer` | `AuthService` | `auth/middleware`, `auth/routes`, `AttendanceService`, `ScoreService` |
| `AttendanceSheets`, `ScoreSheets` | `SheetsClient` | `AttendanceService`, `ScoreService` |
| `SnapshotPublishing` | `SnapshotPublisher` | `SyncService`, `DayVisibilityService`, `SyncSettingsService` |
| `SnapshotStore` | `GitHubSnapshotStore`, `LiveSnapshotStore` | `mergeIntoStore`, both deliveries |
| `SnapshotDelivery` | `GitHubOnlyDelivery`, `LiveFirstDelivery` | `SnapshotPublisher`; `LiveFirstDelivery` (its fallback) |

**Dependency rules.** A guard test (`test/unit/guards/dependency-rules.test.mjs`)
fails the build when a module breaks one. It strips comments first, so a JSDoc
`import("…")` type reference to a port is never an import.

| Rule | Files | The rule |
| --- | --- | --- |
| R0 | the two `ports.mjs` | JSDoc typedefs and one `export {};`, no other code |
| R1 | `*/domain/*.mjs` | import only other domain files, `shared/errors.mjs` and `shared/csv.mjs`: no `node:` modules, no packages |
| R2 | `src/clients/` | `shared/*`, domain files and built-ins; never `registry/`, services, routes or `express` |
| R3 | `src/registry/` | `shared/*` and domain files; never `clients/` (it is given a `JsonDocumentStore`) or `express` |
| R4 | `src/sync/publishing/` | `shared/*`, `sync/domain/*`, its own folder; never `clients/`, `registry/` or `express`. A delivery never imports a store class: stores are injected |
| R5 | `*Service.mjs` | `shared/*`, domain files, built-ins; never `express`, `clients/`, `registry/` or `sync/publishing/` |
| R6 | routes, `syncController`, `auth/middleware` | `express`, `multer`, `shared/*`, the `server/` middleware and their own feature's domain; never `clients/`, `registry/`, a service or `AuthService` |
| R7 | a feature folder | another feature's `domain/` files only |
| R8 | everything but `app.mjs` and `scoresheets/index.mjs` | never `new` a class from `clients/`, `registry/`, `sync/publishing/`, a service or `Server`, outside its own folder |
| R9 | everything but `config/loadConfig.mjs` | never reads `process.env` |
| R10 | `src/server/` | `express`, `http2`, `shared/*`, `docs/openapiSpec.mjs`; never a feature, `auth/`, `registry/` or `clients/` |

**SOLID, as it holds here.**

- **Single responsibility.** `SyncService` changes only with how one day is synced;
  `DayVisibilityService` and `SyncSettingsService` are one small file each;
  `syncController` only parses and writes HTTP; routes only map URLs to handlers;
  each delivery is one path and `SnapshotPublisher` is the choice between them;
  `SyncConfigStore` loads, caches and commits, while validation and the three
  switch edits are pure modules in `registry/domain/`. `AttendanceService`,
  `ScoreService` and `AuthService` each group the use cases of one small feature
  behind one registry and one port, and are split when one use case outgrows the
  rest of its file.
- **Open for extension.** A delivery target, a CSV fetch method, an error type, a
  feature with routes, a registry switch, a token scope or a reaction after a sync
  is each new code plus one line in the composition root (or a scope map), with no
  edit to `SyncService`, `Server`, `errorHandler` or the existing deliveries.
- **Liskov substitution.** Both stores honour one `SnapshotStore` contract, failure
  contract included, and one contract test runs against both; the deliveries take
  stores by role, never by type. Both deliveries return the same result shapes,
  and `LiveFirstDelivery` returns exactly its fallback's result plus `live` when
  the primary fails.
- **Interface segregation.** Each consumer is typed against only the port it calls,
  even where one class implements several (`SyncConfigStore` implements four).
- **Dependency inversion.** Services, deliveries and `mergeIntoStore` import no
  adapter; stores depend on the client ports, not on `GitHubPublisher` or
  `LivePublisher`; `Server` depends on a list of `{ path, router }`; only `app.mjs`
  chooses concrete classes, and only `loadConfig` reads the environment.

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
- The `LIVE CHANNEL` block of JavaScript in `tools/control-center.html` and
  both templates' and every unfinished event's `index.html` and
  `schedule.html`, and the scorer template and every unfinished event's
  `scorer.html`, must stay byte-identical in every page that carries it, and
  its `LIVE_BASE_URL` constant points at the deployed Worker. A finished
  event's pages keep the block with `LIVE_BASE_URL` empty. See
  [sync pipeline](sync-pipeline.md#pages).
- The `ATTENDANCE CLIENT` block of JavaScript in `tools/control-center.html`,
  `_templates/attendance/attendance.html` and every `events/<key>/attendance.html`
  must stay byte-identical. See [event attendance](event-attendance.md#the-client).
- The `SCORE CLIENT` block of JavaScript in `tools/control-center.html`,
  `_templates/scorer/scorer.html` and every `events/<key>/scorer.html` must stay
  byte-identical. See [Control Center § Score entry](control-center.md#score-entry).
- Adding a scoresheet type = a new `templates/<name>.{html,css}` pair in
  `sage-tools-api` **and** an entry in `ScoresheetConfig.mjs`. Nothing else
  needs touching.

## Versioning

`sage-tools-api`'s `package.json` version is bumped for *any* code change,
however small — patch for a fix, minor for a new endpoint/feature, major for
a breaking change. It's the only way to tell what's actually running on
Cloud Run: `GET /ping`'s `X-App-Version` header reads straight from it.
See [Deployment](deployment.md).
