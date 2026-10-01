# Spec — `sage-tools-api` architecture hardening (tests first)

> **Status: not started.** Nothing here is built. Written 2026-10-01 and
> revised 2026-10-02 against `sage-tools-api` 2.5.0 (live push 2.4.0 plus the
> operator switch 2.5.0) and the review recorded in §2.
>
> **Prerequisite, met:** the [Live push delivery](../in-progress/durable-object-push-spec.md)
> code is committed to `main` of `sage-tools-api`, deployed to Cloud Run
> (2.5.0) and running against the `sage-live` Worker, and its 70-check
> `scripts/verify-sync-merge.mjs` passes. This spec refactors that code, so
> start from it, not before it. The test suite describes the code **as it is on
> `main` when it is written**; if the code has moved since this revision, the
> numbers in §2 and in the test-suite spec's §2 (line counts, route list, the 70
> checks) are the first thing to re-check.
>
> **Order is the point.** A unit and integration test suite is built first,
> from its own spec, [`sage-tools-api-test-suite-spec.md`](sage-tools-api-test-suite-spec.md)
> (this spec's Phase 0), against the code **as it is today**, and ends green.
> No production file changes until it does. Every later phase has to leave that
> suite green.

Bring `sage-tools-api` to a shape that follows SOLID, has no duplicated
conflict-retry logic, handles errors in one place, is configured and validated
at startup, keeps its secrets comparison constant-time, names new endpoints the
RESTful way, and is laid out by what each folder is for. Do it without changing
any behaviour a deployed client can see, except the few listed in §3.2. `/ping`
stays exactly as it is.

---

## 0. Read this first (implementer orientation)

Assume no knowledge of this project beyond this page.

**Repos.** `D:\Personal\SAGE` is a plain folder holding four git repos. This
spec changes `sage-tools-api/` (Node 22, Express, ESM `.mjs`, deployed to
Google Cloud Run by a build trigger on every push to `main`), edits docs in
`sage-docs/`, and (Phase 6) `CLAUDE.md` files in the other repos. It does not
change `sage-match-control.github.io/` or `event-data/` except documentation
paths.

**Rules that apply to every change** (from the root `CLAUDE.md`):

- Any code change in `sage-tools-api` bumps `package.json`'s version and adds a
  matching entry to the Changelog in `sage-tools-api/README.md`. §6 gives the
  version for each phase. Tests and docs alone do not bump it.
- Cite a spec from outside `sage-docs` as
  `sage-docs/docs/specs/.../sage-tools-api-architecture-spec.md`, literally
  `...` where the status folder goes. Inside `sage-docs`, links carry the real
  folder.
- Documentation is written in the present tense: what the system does now.
- **Never deploy any of this on an event day, or in the few days before one.**
  Pushing `sage-tools-api` `main` deploys Cloud Run. Work on a branch
  (`arch-refactor`) and merge only when the owner says so.
- Working copies use CRLF line endings. Keep them: an editor that rewrites a
  whole file to LF turns a small diff into a whole-file one. Use `git mv` for
  every move so history follows.
- No new **runtime** dependency. The test runner is Node's built-in
  `node:test`. The `Dockerfile` installs with `npm install --omit=dev`, so a
  dev-only dependency would never reach production, but this spec needs none.

**Tooling facts.**

- Node 22 (the `Dockerfile` pins `node:22-slim`, the latest 22.x); work
  locally on 22.23.3 or later. The test-suite spec §0 lists what was checked
  on it, including enforced coverage thresholds and snapshot assertions.
- There is no linter and no TypeScript, by design. Phase 2 adds JSDoc types
  checked with `// @ts-check`, which needs no build step.
- `GitHubPublisher`, `LivePublisher` and both CSV fetchers call the global
  `fetch`, so tests replace `globalThis.fetch`. They have no injection point
  today and Phase 0 does not add one.
- The HTTP server is `http2.createServer({ allowHTTP1: true }, app)`. Cleartext
  HTTP/1.1 does not work on it (a plain `fetch` to it fails with
  `HPE_INVALID_CONSTANT`), but `server.app` is an ordinary Express app, so
  tests mount it on `http.createServer(server.app)` on port 0 and use `fetch`.
  This was checked: `/ping`, `/sync/config` and `/openapi.json` all answer.

**Commands that exist today** (run from `sage-tools-api/`, each exits non-zero
on failure):

```bash
node scripts/verify-sync-merge.mjs
node scripts/verify-facility-completion.mjs
node scripts/verify-attendance.mjs
node scripts/verify-standard-generator.mjs
node scripts/verify-sheet-generator.mjs
```

**How to work.** One branch, `arch-refactor`, one commit per phase (or a few
small ones). After every step run the whole suite. If a test fails, the
production change is wrong until proven otherwise; change a test only where
§3.2 says the behaviour changes on purpose, and say so in the commit message.

---

## 1. Goals, non-goals and constraints

**Goals**

1. A unit and integration test suite that pins today's behaviour, written
   before any refactor (its own spec, [§5](#5-tests)), and kept green throughout.
2. `SyncService` split along its real responsibilities, with the conflict-retry
   loop written once.
3. One error-handling path, one auth-middleware module, one validated
   configuration module, a constant-time secret comparison, a correct
   `jsconfig.json`.
4. New endpoints named as resources under `/v1`, with every existing URL kept
   as a permanent alias.
5. Folders laid out by purpose: Apps Script separated from the service,
   `src/sync/` layered, the unversioned workspace `CLAUDE.md` put under git.
6. A Cloud Build trigger that does not redeploy the service for changes that
   are not part of it.

**Non-goals**

- Rewriting the scoresheet pipeline, the Apps Script files, the live Worker or
  the site. The site's shared-JS question is recorded in §6.7 as a decision,
  not a build.
- TypeScript, a bundler, a framework, a linter or an ORM.
- Changing what any deployed client sends or receives (§3).
- Renaming or removing `/ping`, `/sync/...`, `/scoresheets/...` or `/auth/...`.

**Constraints that rule out the obvious clean-up**

- `scripts/sheets-sync.gs` is pasted into live Google Sheets workbooks and
  calls `POST /sync/:day?facility=<name>` with `X-Sync-Secret` and
  `X-Edit-At`. Control Center calls `/sync/:day`, `/sync/:day/live`,
  `/sync/live-push`, `/sync/config`, `/auth/login`, and reads specific
  fields from them: `live.published` in the Live/Hide response (its
  confirmation message), and `live.enabled`, `live.switch` and `live.baseUrl`
  in `/sync/config` (the Sync method switch and **Check connection**; a service
  without `live.switch` is reported as "older than 2.5.0"). The scoresheet
  generator calls `/scoresheets/generate/stream`. None of those can be renamed
  or reshaped without breaking something already deployed, so they are aliases
  forever and their fields are part of the contract (test-suite spec §2.1).
- Operators read error bodies: Apps Script's **Sync now** shows
  `Sync failed (<status>): <body>`. Error messages stay as they are.

---

## 2. What this spec answers

The findings of the architecture review (2026-10-01). Each maps to a phase.

| # | Finding | Evidence | Phase |
|---|---|---|---|
| F1 | `SyncService` has five responsibilities: fetch orchestration, merge policy, publishing with fallback, the Live/Hide override, the live-push switch | 467 lines, 6 dependencies, `syncDay` about 130 lines | 4 |
| F2 | Adding a publish target means editing `SyncService` | `if (this.livePublisher?.enabled)` branches in three methods | 4 |
| F3 | `GitHubPublisher` and `LivePublisher` do the same job behind different shapes, so they cannot be swapped | `fetchExisting`/`publish(sha)` vs `read`/`publish(version)` | 3 |
| F4 | The read-modify-write-retry-on-409 loop exists six times | `syncDay`, live loop, `setLiveOverride`, `setIsLive`, `setLivePush`, archive | 3 |
| F5 | The shared secret is compared with `===` | `src/sync/routes.mjs` `hasValidSyncSecret`; `AuthService` and the Worker use `timingSafeEqual` | 2 |
| F6 | Error handling is copy-pasted into every handler | `res.status(err.statusCode ?? 500).json({ error })` in 6 handlers; no error code | 2 |
| F7 | `POST /sync/live-push` only works because it is registered before `POST /sync/:day`; a day key is a slug, so a day called `live-push` or `config` would collide | route order in `sync/routes.mjs`; `SLUG_RE` allows both | 2, 5 |
| F8 | Auth middleware lives inside the sync routes factory | `requireAuthToken`, `requireSyncSecretOrAuthToken` | 2 |
| F9 | `index.mjs` parses environment variables ad hoc with no validation | five `Number(process.env…)`; an empty `LIVE_PUSH_TIMEOUT_MS` would become `0` | 2 |
| F10 | `jsconfig.json` checks nothing | includes `**/*.js` (the code is `.mjs`), target ES2015, `checkJs: false` | 2 |
| F11 | The server comment and docs say HTTP/1.1 fallback works over cleartext; it does not | `http2.createServer({ allowHTTP1: true })` | 2 |
| F12 | No test runner; checks are ad hoc scripts | `scripts/verify-*.mjs` | 0 (test-suite spec) |
| F13 | The API is RPC-style: verbs in paths, `POST` used to set state, no versioning | `/scoresheets/generate`, `/sync/:day/live`, `/sync/live-push`, `/auth/login` | 5 |
| F14 | CORS allows `GET, POST, OPTIONS` only | `Server.mjs` | 2, 5 |
| F15 | Three failed GitHub commits return `500`; `409` is the accurate status | `SyncService` rethrows the raw GitHub error, which has no `statusCode` | 3 |
| F16 | `scripts/` mixes four things: Apps Script sources, their test harness, fixtures, a dev tool | `scripts/` | 1 |
| F17 | `src/sync/` mixes routes, services, infrastructure clients and domain logic | flat folder of 11 files | 1 |
| F18 | Any push, including a `.gs`, Worker or markdown change, rebuilds Cloud Run | build trigger has no file filter | 6 |
| F19 | The workspace `CLAUDE.md` is not under version control | `D:\Personal\SAGE` is not a repo | 6 |
| F20 | The site repeats code blocks by hand (live channel ×9, match rules ×2, team rules ×2) | `tools/control-center.html` is 7,292 lines | 7 (decision) |

---

## 3. Behaviour contract

### 3.1 What must not change

For every existing route, the **status code, response body and the response
headers that deployed clients read** are identical before and after the
refactor. That contract is defined once, in the
[test-suite spec §2.1](sage-tools-api-test-suite-spec.md#21-what-must-not-change),
and enforced by that suite. Nothing in it changes except the items in §3.2.
`GET /ping` stays exactly as it is: `200`, body `PONG!`, `X-App-Version` and
`X-Sync-Config`, no config fetch, in `Server.mjs`, with no alias.

### 3.2 What changes on purpose

Nothing else changes. These do, and each is pinned as today's behaviour by a
test marked `// CHARACTERIZATION B<n>` in the test suite
([test-suite spec §2.2](sage-tools-api-test-suite-spec.md#22-behaviour-pinned-now-that-a-later-phase-changes-on-purpose)),
which is edited in the phase named.

| # | Change | Phase |
|---|---|---|
| B1 | Three conflicting GitHub commits in a row (`syncDay` on the GitHub path, `setIsLive`, `setLivePush`) answer `409` with `{ error, code: "conflict" }` instead of `500` | 3 |
| B2 | Every error body gains a stable `code` next to `error` (§4.6). `error` is unchanged | 2 |
| B3 | CORS `Access-Control-Allow-Methods` becomes `GET, POST, PUT, PATCH, DELETE, OPTIONS` | 2 |
| B4 | A day key of `config` or `live-push` is rejected by config validation | 2 |
| B5 | New `/v1` routes exist (§4.8) | 5 |
| B6 | Process start fails fast with a clear message when an environment variable is set but unusable (§4.7). An **unset** secret still only disables the feature, as today | 2 |

---

## 4. Target architecture

### 4.1 Layout

```
sage-tools-api/
  index.mjs                      # composition root only
  package.json                   # scripts: start, test, test:unit, test:integration, test:e2e, verify
  jsconfig.json                  # fixed (Phase 2)
  apps-script/                   # NOT part of the Cloud Run service (Phase 1)
    sheets-sync.gs  sheet-generator.gs  standard-generator.gs  attendance.gs
    mock-apps-script.mjs
    verify-attendance.mjs  verify-sheet-generator.mjs  verify-standard-generator.mjs
    fixtures/
  live-worker/                   # unchanged
  scripts/
    hash-password.mjs            # the only dev tool left here
    run-appscript-verifies.mjs   # runs the three Apps Script verify scripts (test-suite spec)
  templates/                     # scoresheet HTML/CSS, unchanged
  test/                          # built by the test-suite spec
    helpers/  unit/  integration/  e2e/
  src/
    config/
      loadConfig.mjs             # env -> validated, frozen config (Phase 2)
    server/
      Server.mjs                 # middleware + mounting + /ping + /openapi.json
      errorHandler.mjs           # AppError -> status + { error, code } (Phase 2)
      asyncHandler.mjs
      cors.mjs
    auth/
      AuthService.mjs
      middleware.mjs             # requireAuthToken, requireSyncSecretOrAuthToken (Phase 2)
      routes.mjs                 # /auth/login and /v1/sessions
    shared/
      Logger.mjs  ConcurrencyPool.mjs  errors.mjs
      safeEqual.mjs              # constant-time string compare (Phase 2)
      conflictRetry.mjs          # COMMIT_ATTEMPTS, ConflictError, withConflictRetry (Phase 3)
    sync/
      index.mjs                  # composes the sync module (Phase 4)
      SyncService.mjs            # orchestrates one sync
      GoLiveService.mjs          # the Live/Hide override
      LivePushSettings.mjs       # the operator switch
      SyncDiagnostics.mjs        # what GET /sync/config returns
      syncController.mjs         # HTTP handlers, no routing
      legacyRoutes.mjs           # /sync/... (permanent aliases)
      v1Routes.mjs               # /v1/... (Phase 5)
      domain/
        mergeSnapshot.mjs  facilityCompletion.mjs  snapshotStamp.mjs  syncTiming.mjs  editAt.mjs
      publishing/
        SnapshotStore.mjs        # the interface, as JSDoc typedefs
        GitHubSnapshotStore.mjs  LiveSnapshotStore.mjs
        mergeIntoStore.mjs       # the one read-merge-write-retry loop
        GitHubOnlyPublishing.mjs  LiveFirstPublishing.mjs  PublishingSelector.mjs
      infra/
        GitHubPublisher.mjs  LivePublisher.mjs  SheetsCsvFetcher.mjs  GvizCsvFetcher.mjs
      config/
        SyncConfigStore.mjs  SyncConfigSnapshot.mjs  events.seed.json
    scoresheets/                 # unchanged except routes (Phase 5)
    docs/
      openapiSpec.mjs
```

### 4.2 Responsibilities

| Module | One job | Depends on |
|---|---|---|
| `index.mjs` | read config, build objects, start the server | everything, nothing depends on it |
| `config/loadConfig` | turn `process.env` into a validated config object | `shared/errors` |
| `server/Server` | CORS, version header, JSON body, mounting, `/ping`, `/openapi.json`, error handler | route modules |
| `server/errorHandler` | map any error to `{ error, code }` and a status | `shared/errors` |
| `auth/middleware` | two Express middlewares | `AuthService`, `safeEqual` |
| `sync/domain/*` | pure functions: merge, completion, stamps, timing, `X-Edit-At` parsing | nothing |
| `sync/publishing/*` | getting a merged snapshot to GitHub, and to the live Worker first when it is on | stores, `shared/conflictRetry` |
| `sync/SyncService` | one sync: resolve day, fetch, merge, publish, time, respond | fetchers, `PublishingSelector`, config store, `domain/*` |
| `sync/GoLiveService` | the Live/Hide override | config store, `PublishingSelector` |
| `sync/LivePushSettings` | the operator switch | config store, live publisher |
| `sync/SyncDiagnostics` | the `GET /sync/config` payload | config store, live publisher |
| `sync/syncController` | request parsing and response writing | the four services |
| `sync/*Routes` | URL to handler mapping and OpenAPI blocks | controller, `auth/middleware` |

### 4.3 Dependency rules

1. `domain/` imports nothing outside `domain/` and `shared/errors`.
2. `publishing/` imports `domain/` and `shared/`; never `infra/` directly. Stores
   wrap `infra/` clients and are the only bridge.
3. Services import `domain/`, `publishing/`, `config/` and `shared/`; never
   Express.
4. Controllers and routes import services and `server/` helpers; never `infra/`.
5. Only `index.mjs` and `sync/index.mjs` construct concrete classes.
6. No module reads `process.env` except `config/loadConfig.mjs`.

### 4.4 The snapshot store interface

Both places a snapshot lives (GitHub, the live Worker) are optimistic-concurrency
stores: read a value and a **token**, write back with that token, and be told
if someone else got there first. The token is a file `sha` for GitHub and a
`version` for the Worker.

```js
// src/sync/publishing/SnapshotStore.mjs
/**
 * @typedef {{ event: string, day: string, path: string }} SnapshotRef
 * @typedef {{ snapshot: object|null, token: string|number|null }} StoreRead
 * @typedef {{ ok: true, token: string|number|null, commitSha?: string }
 *         | { ok: false, conflict: true }} StoreWrite
 *
 * @typedef {object} SnapshotStore
 * @property {(ref: SnapshotRef) => Promise<StoreRead>} read
 *   Resolves { snapshot: null, token: <empty token> } when nothing is stored.
 *   Throws on network errors, timeouts, auth failures and 5xx.
 * @property {(ref: SnapshotRef, snapshot: object,
 *             opts: { token: string|number|null, message?: string }) => Promise<StoreWrite>} write
 *   Resolves { ok: false, conflict: true } when `token` is stale. Throws on
 *   every other failure, with `err.status` set where the remote gave one.
 */
export {};
```

- `GitHubSnapshotStore.read` calls `publisher.fetchExisting(ref.path)`; its
  token is the sha (`null` when the file does not exist). `write` calls
  `publisher.publish(ref.path, snapshot, message, token)` and maps an error
  with `status === 409` to `{ ok: false, conflict: true }`. With `token: null`
  the publisher looks the sha up itself, which is what the archive step relies
  on.
- `LiveSnapshotStore.read` calls `livePublisher.read(event, day)`; its token is
  the version (`0` for an empty object). `write` calls
  `livePublisher.publish(event, day, snapshot, token)`, which already returns
  the right shape.

### 4.5 The retry loop, once

```js
// src/sync/publishing/mergeIntoStore.mjs
/**
 * Read a base snapshot, build the merged result from it, stamp it, write it
 * with the base's token, and go round again on a conflict.
 *
 * @param {object} p
 * @param {import('./SnapshotStore.mjs').SnapshotStore} p.store
 * @param {import('./SnapshotStore.mjs').SnapshotRef} p.ref
 * @param {(ref) => Promise<import('./SnapshotStore.mjs').StoreRead>} [p.readBase]
 *        defaults to store.read(ref); LiveFirstPublishing passes one that also
 *        reads GitHub and returns the newer copy (the stale-object rule)
 * @param {(base: object|null) => { snapshot: object, stale: string[] }} p.build
 * @param {(stale: string[]) => string} p.message
 * @returns {Promise<{ snapshot: object, stale: string[], attempts: number,
 *                     token: any, commitSha: string|null }>}
 * @throws ConflictError after COMMIT_ATTEMPTS conflicts (status 409)
 */
export async function mergeIntoStore({ store, ref, readBase, build, message, log }) { /* … */ }
```

`withConflictRetry` (in `shared/conflictRetry.mjs`) is the generic loop it uses,
and `SyncConfigStore.setIsLive` / `setLivePush` use it directly:

```js
export const COMMIT_ATTEMPTS = 3;

export class ConflictError extends AppError {
    constructor(what, attempts) {
        super(`${what}: ${attempts} conflicting updates in a row`, 409);
        this.status = 409;            // kept: callers and tests read err.status
        this.code = "conflict";
    }
}

/**
 * @template T
 * @param {(attempt: number) => Promise<{ ok: true, value: T } | { ok: false }>} attemptFn
 *        return { ok: false } for a conflict; throw for anything else
 * @returns {Promise<{ value: T, attempts: number }>}
 */
export async function withConflictRetry(attemptFn, { attempts = COMMIT_ATTEMPTS, what = "update", onConflict } = {}) {
    for (let n = 1; n <= attempts; n++) {
        const out = await attemptFn(n);
        if (out.ok) return { value: out.value, attempts: n };
        onConflict?.(n, attempts);
    }
    throw new ConflictError(what, attempts);
}
```

Log text stays as it is today (`commit conflict (attempt n/3) — re-reading and
re-merging`, `live version conflict (attempt n/3) …`) so existing operator log
searches keep working.

### 4.6 Errors

`AppError` gains a `code` string; every subclass sets one:

| Class | `statusCode` | `code` |
|---|---|---|
| `ValidationError` | 400 | `validation_error` |
| `UnauthorizedError` | 401 | `unauthorized` |
| `UnknownScoresheetTypeError` | 400 | `unknown_scoresheet_type` |
| `UnknownSyncDayError` | 400 | `unknown_day` |
| `SyncUpstreamError` | 502 | `upstream_failure` |
| `SyncConfigUnavailableError` | 503 | `config_unavailable` |
| `ConflictError` (new) | 409 | `conflict` |
| `NotFoundError` (new, Phase 5) | 404 | `not_found` |
| `ConfigError` (new, startup only) | n/a | `config_error` |
| anything else | 500 | `internal_error` |

```js
// src/server/errorHandler.mjs — registered last, after every route
export function errorHandler(logger) {
    return (err, req, res, next) => {
        if (res.headersSent) return next(err);          // streaming: the route already wrote its own error line
        const status = err.statusCode ?? 500;
        logger.error(`${req.method} ${req.originalUrl} failed`, err);
        res.status(status).json({ error: err.message, code: err.code ?? "internal_error" });
    };
}
```

`error` is still `err.message`, including for `500`s: Apps Script shows it to
operators. Route handlers stop catching and `throw`; a tiny `asyncHandler(fn)`
forwards rejections (Express 4 does not).

### 4.7 Configuration

```js
// src/config/loadConfig.mjs
/** @returns {Readonly<AppConfig>} or throws ConfigError listing every problem at once */
export function loadConfig(env = process.env) { /* … */ }
```

Rules, applied to every variable currently read in `index.mjs`:

- An **empty or unset** variable is "not set". `NaN`, a negative number, or a
  non-integer where an integer is required throws `ConfigError`.
- Integer variables and their bounds: `PORT` 1–65535 (default 8080),
  `SCORESHEET_CONCURRENCY` ≥ 1 (default 8), `SHEETS_FETCH_TIMEOUT_MS` ≥ 0 (**`0`
  disables the timeout**, as today), `SYNC_CONFIG_TTL_MS` ≥ 0,
  `AUTH_TOKEN_TTL_MS` ≥ 1, `LIVE_PUSH_TIMEOUT_MS` ≥ 1.
- Unset secrets (`GITHUB_TOKEN`, `SYNC_SHARED_SECRET`, `GOOGLE_SHEETS_API_KEY`,
  `AUTH_PASSWORD_HASH`, `AUTH_TOKEN_SECRET`, `LIVE_PUSH_SECRET`) never throw:
  the feature fails closed as it does today. `loadConfig` returns them as
  `undefined`, and `index.mjs` logs one `warn` line naming which are unset.
- `LIVE_PUSH_URL` set but not an `http(s)` URL throws.
- The returned object is `Object.freeze`d, nested.

### 4.8 The `/v1` API

New endpoints, named as resources, mounted under `/v1`. Each existing URL stays,
served by the same controller, marked `deprecated: true` in OpenAPI with the
note "kept for deployed Apps Script and Control Center".

| Legacy (kept forever) | New | Method and semantics | Success |
|---|---|---|---|
| `POST /sync/:day` | `POST /v1/days/{day}/syncs` | create a sync; query `facility`, `method` | `200` same body |
| `POST /sync/:day/live` | `PUT /v1/days/{day}/visibility` | set the day's visibility; body `{ isLive }` | `200` same body |
| `POST /sync/live-push` | `PUT /v1/settings/live-push` | set the switch; body `{ enabled }` | `200` same body |
| `GET /sync/config` | `GET /v1/diagnostics/sync` | read diagnostics | `200` same body |
| `POST /scoresheets/generate` | `POST /v1/scoresheets` | create a scoresheet; `Accept: application/pdf` (default) | `200` PDF, same headers |
| `POST /scoresheets/generate/stream` | `POST /v1/scoresheets` with `Accept: application/x-ndjson` | same, streamed | `200` NDJSON, same lines |
| `POST /auth/login` | `POST /v1/sessions` | create a session | `201 { token, expiresAt }` |
| `GET /ping` | **kept as is, no alias** | health | `200 PONG!` |

Notes:

- `PUT` is idempotent, which is the honest verb for "set this value". Calling
  it twice with the same body gives the same result (`changed: false` the
  second time for the switch).
- A nested path (`/v1/days/{day}/…`) removes the `/sync/:day` versus
  `/sync/live-push` collision; the legacy collision is handled by B4.
- `POST /v1/scoresheets` with an `Accept` it cannot satisfy answers `406`.
- Status `201` is used only for `POST /v1/sessions`, where a resource really is
  created. The sync keeps `200` because it returns a report, not a stored
  resource, and the legacy route returns `200`.
- All `/v1` errors use the same `{ error, code }` body.

The site and Apps Script keep calling the legacy URLs. Moving Control Center to
`/v1` is a separate, later change in the site repo and is not part of this
spec.

---
## 5. Tests

### 5.1 The suite that comes first

The unit and integration suite for everything that exists today (helpers, the
`FakeWorld`, the characterization policy, the coverage matrix, the ten sabotage
checks, the build steps and acceptance checklist) is its own spec:
[`sage-tools-api-test-suite-spec.md`](sage-tools-api-test-suite-spec.md). It is
this spec's Phase 0 and a **gate**: Phase 1 does not start until that spec's
acceptance checklist is complete and `npm run verify` is green. After that, every
phase here leaves it green. Its helpers (`startApp`, `createFakeWorld`, builders, the
fakes) are what the tests in §5.2 are written with.

### 5.2 Tests for the modules this spec creates

These modules do not exist when the suite is built, so their tests are written
**first, in the phase that creates the module**: write them, watch them fail
(the module is missing), then implement. They follow the test-suite spec's
rules (§3.5) and layout (§3.3), and live under `test/unit/` mirroring `src/`.

| Module | Cases |
|---|---|
| `safeEqual.mjs` (Phase 2) | equal strings true; different strings of equal and of unequal length false; `undefined` or non-string false; does not throw on empty strings |
| `conflictRetry.mjs` (Phase 3) | returns on first success with `attempts: 1`; retries on `{ ok: false }`, calls `onConflict(n, max)`; throws `ConflictError` (status and `statusCode` 409, code `conflict`) after the limit; a thrown error from `attemptFn` propagates unretried; custom `attempts` honoured |
| `middleware.mjs` (Phase 2) | `requireAuthToken`: valid bearer passes, missing/garbled/expired `401 { error: "Unauthorized", code: "unauthorized" }`; `requireSyncSecretOrAuthToken`: secret passes, token passes, neither `401`, an empty configured secret never matches an empty header |
| `mergeSnapshot.mjs` (Phase 3) | the merge cases in the test-suite spec §4 (`SyncService` merge) exercised directly on the pure function: fresh wins; untargeted carried forward; failed facility carried forward and listed `stale`; facility never seen and failed is omitted; nothing to publish throws `SyncUpstreamError`; `completedAt` stamped once and carried; `lastEditAt` from the edit or carried; `publishedAt` starts as `now` |
| `snapshotStamp.mjs` (Phase 3) | `publishedAt` preferred over `generatedAt`; neither gives `0`; `newer(a, b)` picks the later, prefers `a` on a tie, handles `null` on either side |
| `syncTiming.mjs` (Phase 3) | `editToRequestMs`/`editToPublishedMs` are `null` without an edit time; `liveMs`/`archiveMs` `null` when that step did not run; the log line prints `n/a` for `null` and otherwise `<n>ms` in the order `edit→request=… fetch=… publish=… live=… archive=… edit→published=…` |
| `editAt.mjs` (Phase 3) | `parseEditAt(header, now)`: a value within the last hour and at most 5 s ahead is returned; older than an hour, more than 5 s ahead, `NaN`, empty, negative and missing are `null` (today's inline rule in `handleSync`) |
| `loadConfig.mjs` (Phase 2) | each rule in §4.7: empty means unset, `NaN`/negative/fractional throw, `SHEETS_FETCH_TIMEOUT_MS=0` allowed, bad `LIVE_PUSH_URL` throws, all problems reported in one error, unset secrets are `undefined` and listed by the helper that builds the warning, the result is frozen |
| `errorHandler.mjs` (Phase 2) | each class in §4.6 gives its status and `{ error, code }`; an unknown `Error` is `500 internal_error` with its message; `res.headersSent` delegates and writes nothing; the error is logged once |
| `SyncConfigStore` reserved day keys (Phase 2) | a day key of `config` or `live-push` is rejected with a message naming it (flips the `CHARACTERIZATION B4` pin) |

The `/v1` parity tests are specified in §6.5. The store, `mergeIntoStore` and
publishing-strategy tests are specified in §6.3 and §6.4.

### 5.3 Pins resolved along the way

`grep -rn "CHARACTERIZATION B" test/` lists what is still pending. Each marker
is edited to the new expectation in the commit of the phase named in §3.2, and
the commit message names the B-number. After Phase 3 the grep returns nothing.

---

## 6. Phases

Each phase ends with `npm run verify` green, a version bump where stated, a
Changelog entry, and one commit on `arch-refactor`. Phases are strictly
ordered; do not start one before the previous is green.

| Phase | What | Version | Production code touched |
|---|---|---|---|
| 0 | The test suite, **its own spec** ([test-suite spec](sage-tools-api-test-suite-spec.md)) | none | none (only `package.json` scripts, `test/`, one script, the pre-push hook, README) |
| 1 | Folder moves | 2.6.1 | paths and imports only |
| 2 | Hardening | 2.6.2 | secret compare, errors, config, auth middleware, CORS, jsconfig |
| 3 | One retry loop, store interface, domain extraction | 2.6.3 | `sync/` internals |
| 4 | Publishing strategies and the `SyncService` split | 2.6.4 | `sync/` internals |
| 5 | The `/v1` API | 2.7.0 | routes, controllers, OpenAPI |
| 6 | Build, docs and workspace | none | none |
| 7 | Site shared-JS decision | n/a | none |

### 6.0 Phase 0 — tests before code (a separate spec)

Built from [`sage-tools-api-test-suite-spec.md`](sage-tools-api-test-suite-spec.md)
on its own branch (`test-suite`), with no production change. **Gate:** its §7
acceptance checklist is complete, `npm test` and `npm run verify` are green, and
`arch-refactor` is created from that branch. The version and Changelog are not
touched. If the code on `main` has moved since that spec was written, bring it
up to date first (its §2 lists the contract it pins).

### 6.1 Phase 1 — folder moves (2.6.1)

Mechanical, with the test suite as the guard. Do all moves with `git mv`, update
imports, and run `npm run verify` after each group.

**1a. Apps Script out of `scripts/`.**

| From | To |
|---|---|
| `scripts/sheets-sync.gs`, `sheet-generator.gs`, `standard-generator.gs`, `attendance.gs` | `apps-script/` |
| `scripts/mock-apps-script.mjs` | `apps-script/mock-apps-script.mjs` |
| `scripts/verify-attendance.mjs`, `verify-sheet-generator.mjs`, `verify-standard-generator.mjs` | `apps-script/` |
| `scripts/fixtures/` | `apps-script/fixtures/` |
| `scripts/hash-password.mjs` | stays |
| `scripts/run-appscript-verifies.mjs` | stays; points at `apps-script/` |

The verify scripts locate the `.gs` files and fixtures relative to
themselves; confirm each still finds them. Update `package.json`'s
`test:appscript`.

**1b. `src/sync/` into layers.**

| From | To |
|---|---|
| `facilityCompletion.mjs` | `sync/domain/facilityCompletion.mjs` |
| `GitHubPublisher.mjs`, `LivePublisher.mjs`, `SheetsCsvFetcher.mjs`, `GvizCsvFetcher.mjs` | `sync/infra/` |
| `SyncConfigStore.mjs`, `SyncConfigSnapshot.mjs`, `events.seed.json` | `sync/config/` |
| `SyncService.mjs`, `routes.mjs` | stay in `sync/` |

Update: every import; `index.mjs`'s seed path
(`src/sync/config/events.seed.json`); `src/docs/openapiSpec.mjs` globs; the
tests' import paths (the test files move to mirror `src/`).

**1c. Documentation paths.** Every mention of the moved files in markdown and
comments. Find them:

```bash
grep -rIn --exclude-dir=node_modules --exclude-dir=.git --exclude-dir=event-data \
  -e "scripts/sheets-sync" -e "scripts/sheet-generator" -e "scripts/standard-generator" \
  -e "scripts/attendance" -e "scripts/mock-apps-script" -e "scripts/verify-" \
  -e "scripts/fixtures" -e "src/sync/facilityCompletion" -e "src/sync/GitHubPublisher" \
  -e "src/sync/LivePublisher" -e "src/sync/Sheets" -e "src/sync/Gviz" \
  -e "src/sync/SyncConfig" -e "src/sync/events.seed" \
  /d/Personal/SAGE/sage-tools-api /d/Personal/SAGE/sage-docs \
  /d/Personal/SAGE/sage-match-control.github.io /d/Personal/SAGE/CLAUDE.md
```

About 40 files match, mostly specs in `sage-docs`. Replace each with the new path.
Historical specs keep their prose; only the path text changes. The `.gs` header
comments that say "paste `scripts/…`" are updated too (a `.gs` change does not
bump the version and ships by pasting, so only the comment moves).

**Acceptance.** `npm run verify` green; the grep above returns nothing;
`node index.mjs` starts and `GET /ping` answers; `docker build .` (or the
owner's build trigger on the branch) succeeds; `git log --follow` on a moved
file shows its history.

### 6.2 Phase 2 — hardening (2.6.2)

Do these in order; for each, write the failing test first.

1. **`shared/safeEqual.mjs`** (F5): hash both strings with SHA-256 and compare
   with `timingSafeEqual`, so length does not leak. Use it for `X-Sync-Secret`.
2. **`config/loadConfig.mjs`** (F9, B6) per §4.7; `index.mjs` calls it once, passes
   the pieces on, and logs the one warning line for unset secrets. A
   `ConfigError` prints every problem and exits with code 1 before listening.
   Add `test:coverage:config` (`src/config/**`, lines ≥ 95) to `package.json`
   and to `test:coverage` in the same commit (test-suite spec §3.2).
3. **Errors** (F6, B2): add `code` to `AppError` and each subclass (§4.6), add
   `ConflictError` and `ConfigError`, write `server/errorHandler.mjs` and
   `server/asyncHandler.mjs`, register the handler last in `Server`. Convert
   every handler in `sync/routes.mjs`, `scoresheets/routes.mjs` and
   `auth/routes.mjs` to throw (wrapped in `asyncHandler`) and delete their
   `try/catch`. The streaming handler keeps its own `catch`, because it must
   write an NDJSON error line after headers are sent; the handler's
   `headersSent` check covers anything that escapes it. Update the integration
   tests' expected bodies to include `code` (B2).
4. **`auth/middleware.mjs`** (F8): move `requireAuthToken` and
   `requireSyncSecretOrAuthToken` out of the sync routes factory into factories
   taking `authService` (and the shared secret). Routes import them. The
   [attendance spec](multi-event-attendance-spec.md)'s operator-or-desk check
   in `src/attendance/routes.mjs` moves here too, as
   `requireAuthTokenOrDeskToken`.
5. **`server/cors.mjs`** (F14, B3): move the CORS middleware out of `Server`;
   methods become `GET, POST, PUT, PATCH, DELETE, OPTIONS`. Fix the comment on
   `start()` (F11): cleartext HTTP/1.1 does not work; Cloud Run talks h2c.
   `start()` returns the listening server so a test can close it.
6. **Reserved day keys** (F7, B4): `SyncConfigStore.validate` rejects a day key
   in `RESERVED_DAY_KEYS = ["config", "live-push"]` with a message naming it.
   Add the test and flip its `CHARACTERIZATION` marker.
7. **`jsconfig.json`** (F10): `include: ["src/**/*.mjs", "index.mjs", "test/**/*.mjs"]`,
   `target`/`lib` `ES2022`, `module`/`moduleResolution` `NodeNext`, `checkJs:
   false` globally. Add `// @ts-check` to the new modules from this phase on
   (`safeEqual`, `loadConfig`, `errorHandler`, `middleware`, `cors`) and make
   them clean in the editor. Do not turn `checkJs` on for existing files here.

**Acceptance.** Suite green with only the B2/B3/B4 test edits; `curl` against a
local instance with `SHEETS_FETCH_TIMEOUT_MS=abc` exits 1 with a readable
message; with `LIVE_PUSH_TIMEOUT_MS=` (empty) it starts and uses the default;
no handler contains `res.status(err.statusCode`.

### 6.3 Phase 3 — one retry loop, one store interface, domain extraction (2.6.3)

Write the unit tests for each new module from §5.2 **first**, watch them fail
(the module does not exist), then implement.

1. `shared/conflictRetry.mjs` (§4.5). Move `COMMIT_ATTEMPTS` here; keep
   re-exporting it from `infra/GitHubPublisher.mjs` for one release so nothing
   else breaks, then delete the re-export in Phase 4.
2. `sync/publishing/SnapshotStore.mjs`, `GitHubSnapshotStore.mjs`,
   `LiveSnapshotStore.mjs` (§4.4), each with a unit test that drives it against
   the existing `FakePublisher`/`FakeLivePublisher`.
3. `sync/publishing/mergeIntoStore.mjs` (§4.5) with tests: success first try;
   conflict then success (re-reads and re-builds against the new base);
   `readBase` override used; `ConflictError` after three; non-conflict errors
   propagate; `build` throwing (`SyncUpstreamError`) propagates unretried.
4. Extract the pure parts of `SyncService` into `sync/domain/`:
   `mergeSnapshot.mjs` (`#buildSnapshot`, same inputs and outputs),
   `snapshotStamp.mjs` (the `stamp` helper in `#readLiveBase`), `syncTiming.mjs`
   (the `timing` object and the log line), `editAt.mjs` (the inline window rule
   in `handleSync`). `SyncService` calls them. No behaviour change: the ported
   scenarios stay green untouched.
5. `SyncConfigStore.setIsLive` and `setLivePush` use `withConflictRetry`
   (B1: three conflicts now throw `ConflictError`, `409`). `SyncService`'s
   GitHub-path loop, the live loop and the archive loop are **not** converted
   yet: Phase 4 replaces them wholesale.
6. Update the B1 characterization tests.

**Acceptance.** Suite green; `grep -rn "COMMIT_ATTEMPTS" src` finds the constant
defined once and used by `conflictRetry` and the stores;
`SyncConfigStore` has no hand-written retry loop left.

### 6.4 Phase 4 — publishing strategies and the `SyncService` split (2.6.4)

Goal: `SyncService` knows nothing about GitHub, the Worker, the archive or the
fallback.

1. **Contract.** A publishing strategy has two methods:

```js
/**
 * @typedef {object} PublishResult
 * @property {object} snapshot             what was published
 * @property {string[]} stale              facilities carried forward after a failed fetch
 * @property {number} attempts
 * @property {number} publishedAtMs        Date.now() when it became visible to viewers
 * @property {string|null} commitSha       the GitHub commit, or null when the archive failed
 * @property {{ published: boolean, version?: number, error?: string }} [live]   absent when live push is not in use
 * @property {{ committed: boolean, commitSha?: string, error?: string }} [archive]  present only after a live publish
 * @property {number|null} liveMs
 * @property {number|null} archiveMs
 *
 * @typedef {object} PublishingStrategy
 * @property {(a: { ref, build, message, log }) => Promise<PublishResult>} publish
 * @property {(a: { ref, mutate, message, log }) =>
 *            Promise<{ republished: boolean, live?: object, archive?: object }>} republish
 */
```

   `build(base) → { snapshot, stale }` is the merge, supplied by `SyncService`.
   `mutate(snapshot) → snapshot` is the change a Live/Hide applies.

2. **`GitHubOnlyPublishing`**: `publish` is `mergeIntoStore` over
   `GitHubSnapshotStore`; `republish` reads, applies `mutate`, stamps
   `publishedAt`, writes with retry, and returns `republished: false` when there
   is no file.
3. **`LiveFirstPublishing`** (constructor takes `liveStore`, `githubStore`,
   `fallback` = a `GitHubOnlyPublishing`): `publish` is `mergeIntoStore` over
   `liveStore` with a `readBase` that reads the Worker and GitHub **together**
   and returns the newer by `snapshotStamp` (the stale-object rule, unchanged),
   then the archive (`githubStore.write` with `token: null`, retried on conflict
   by re-reading the Worker's current snapshot, never throwing, returning
   `{ committed, commitSha | error }`). Any Worker read or publish failure, or
   `ConflictError`, hands over to `fallback.publish` and returns its result with
   `live: { published: false, error }`. `republish` mirrors it. Every log line
   keeps today's text.
4. **`PublishingSelector`**: constructed with the two strategies and the
   `LivePublisher` (only to ask `enabled`); `select(config)` returns
   `LiveFirstPublishing` when `livePublisher.enabled && config.livePushOn`,
   else the GitHub-only one. This is the only place that decision lives.
5. **Services.**
   - `SyncService({ sheetsApiFetcher, gvizFetcher, publishing, configStore, logger })`:
     resolve the day, fetch, build `freshByName`/`failed`/`lastEditAt`, call
     `publishing.select(config).publish(...)` with `build = base =>
     mergeSnapshot({...})`, then `buildTiming`, log, and return the response. It
     does not import `GitHubPublisher`, `LivePublisher` or `COMMIT_ATTEMPTS`.
   - `GoLiveService({ configStore, publishing, logger }).setLiveOverride(day, isLive)`.
   - `LivePushSettings({ configStore, livePublisher, logger })`:
     `set(enabled) → { enabled, changed, available }`.
   - `SyncDiagnostics({ configStore, livePublisher }).describe()` returns the
     `GET /sync/config` payload; `syncRoutes` no longer receives `syncConfigStore`.
6. **`syncController.mjs`** exposes `syncDay`, `setDayVisibility`, `setLivePush`,
   `getDiagnostics`: request in, service call, `res.json`. It parses `facility`,
   `method` and the `X-Edit-At` header (via `editAt`), and validates bodies,
   throwing `ValidationError`. `sync/routes.mjs` is renamed `legacyRoutes.mjs`
   and shrinks to URL mapping, middleware and the `@openapi` blocks.
7. **`sync/index.mjs`** exports `createSyncModule({ config, logger })` returning
   `{ router(s), services }`. `index.mjs` calls it. The integration test helper
   `buildApp` is replaced by this same function.
8. Delete the Phase 3 re-export of `COMMIT_ATTEMPTS`.

**Acceptance.** Every test from the test suite passes unchanged except import paths and
constructor wiring; `SyncService.mjs` is under 150 lines and
`SyncService.syncDay` under 60; `grep -n "livePublisher\|GitHubPublisher"
src/sync/SyncService.mjs` finds nothing; there is exactly one place that reads
`livePushOn`; adding a hypothetical third strategy requires no edit to
`SyncService` (state which files a new strategy would touch in the Changelog).

### 6.5 Phase 5 — the `/v1` API (2.7.0)

1. Write the parity tests first (`test/integration/v1.test.mjs`). They are
   table-driven: for each row of §4.8, the legacy and the `/v1` request produce
   the same status (except `201` for sessions), the same body and the same
   headers listed in the test-suite spec §2.1. Plus: `PUT` visibility twice is idempotent;
   `PUT` live-push twice gives `changed: false` the second time; `POST
   /v1/scoresheets` with `Accept: application/x-ndjson` streams, with
   `application/pdf` or none returns a PDF, with `text/html` is `406`;
   `POST /v1/sessions` is `201`; an unknown `/v1/...` path is `404 { error,
   code: "not_found" }`; `/ping` has no `/v1` alias.
   The `/v1` prefix already exists: the
   [attendance spec](multi-event-attendance-spec.md) (2.6.0) mounts its three
   routes there. They and their tests stay as they are; this phase adds
   routes beside them.
2. `sync/v1Routes.mjs`, `auth` and `scoresheets` get `/v1` routers calling the
   **same** controller functions as the legacy routes. No handler logic is
   duplicated. Mount in `Server`: legacy at `/sync`, `/scoresheets`, `/auth`;
   new at `/v1`.
3. Add a catch-all `404` handler that throws `NotFoundError` (new, status 404,
   code `not_found`) so unknown paths get the same body as every other error.
4. OpenAPI: add `@openapi` blocks for each new path in the route files; mark
   the legacy duplicates `deprecated: true` with the description note from
   §4.8; add `servers` unchanged; extend `apis` globs to the new files. Update
   `openapiSpec.test.mjs` to the 14 paths and the route-versus-document test.
5. Docs: a `docs/technical/api.md` page listing both surfaces and the rule "a new
   client uses `/v1`; the old URLs never go away".

**Acceptance.** Parity tests green; the OpenAPI document validates (import it
into Postman: **Import → Link** on `/openapi.json`); every legacy test from the
test suite still green untouched.

### 6.6 Phase 6 — build, docs and workspace

1. **Cloud Build trigger (F18).** The owner runs this once; the implementer
   documents it in `docs/technical/deployment.md`:

   ```bash
   gcloud builds triggers list --project=sage-tools-api
   gcloud builds triggers update github <TRIGGER_NAME> --project=sage-tools-api \
     --ignored-files='live-worker/**,apps-script/**,test/**,.githooks/**,scripts/run-appscript-verifies.mjs,**/*.md'
   ```

   Verify by pushing a README-only commit to a branch the trigger watches and
   confirming no build starts. `package.json`, `src/**`, `index.mjs`,
   `templates/**` and the `Dockerfile` still trigger a build.
2. **Remove** `scripts/verify-sync-merge.mjs` and
   `scripts/verify-facility-completion.mjs` (superseded by `test/`), and update
   every doc that names them (`README.md`, root `CLAUDE.md`, `sage-docs`).
3. **Workspace `CLAUDE.md` (F19).** `git init` in `D:\Personal\SAGE`, with a
   `.gitignore` listing `sage-tools-api/`, `sage-match-control.github.io/`,
   `sage-docs/`, `event-data/` and `.claude/`; commit only `CLAUDE.md`. The
   owner creates the GitHub repo (`sage-match-control/sage-workspace`) and pushes.
   The file stays at the same path, so Claude Code keeps finding it.
4. **Docs**, present tense (§7).
5. **CLAUDE.md rules.** The test-maintenance rule is already there (test-suite
   spec §9.4). Extend it with "no production change without a failing test
   first", and add the `/v1` rule, the dependency rules of §4.3, and that
   `apps-script/` is not part of the service.

**Acceptance.** A markdown-only push does not rebuild Cloud Run; the root
`CLAUDE.md` is tracked; the doc grep in Phase 1 still returns nothing.

### 6.7 Phase 7 — the site's duplicated code (decision only)

The site repeats code by hand: the live-channel block in nine pages, the
played/BYE rules in `control-center.html` and the server, the team-event rules
in two files. That is the largest maintenance risk in the system, but fixing it
changes the site's "one self-contained file per page" rule, which is a
decision for the owner, not an implementation detail.

This phase produces a one-page decision record in `sage-docs/docs/specs/not-started/`
comparing two options, and nothing else:

1. **Shared files** at root-absolute `/assets/js/*.js` (the same convention the
   site already uses for images), loaded with `<script src>`. No build step.
   Removes the byte-identical-copy hazard. Costs: a page is no longer one file,
   `tools/sw.js` caching and cache-busting (`?v=`) need care, and an archived
   page that keeps loading a shared file can break when that file changes.
2. **A generation script** that stamps the shared block into each page. Keeps
   self-contained pages. Costs: a script to run and a diff check to keep.

Recommendation to evaluate: option 1 for the live channel and the match rules,
leaving archived events untouched.

---

## 7. Documentation to update

Present tense, in the same commit as the phase that changes the thing.

| File | Change |
|---|---|
| `sage-tools-api/README.md` | Changelog entry per versioned phase; **Testing** section (added by the test-suite spec); updated layout |
| root `CLAUDE.md` | `sage-tools-api` layout (`apps-script/`, `test/`, the new `src/` tree), the verify and test commands, the dependency rules, the `/v1` rule, the deprecated-alias note, the live-push entries that name moved files |
| `sage-docs/docs/technical/architecture.md` | the layering and dependency rules (§4.1–4.3) |
| `sage-docs/docs/technical/sync-pipeline.md` | the publishing strategies and the store interface replace the `SyncService` description; file paths |
| `sage-docs/docs/technical/deployment.md` | the Cloud Build file filter; the test commands |
| `sage-docs/docs/technical/api.md` (new, Phase 5) | both API surfaces and the alias rule |
| `sage-docs/docs/technical/README.md`, `mkdocs.yml` | index and nav for the new page and this spec |
| `sage-docs/docs/specs/README.md`, folder READMEs | this spec's entry, then its move as it progresses |
| every file the Phase 1 grep finds | new paths |

---

## 8. Acceptance checklist

**Phase 0** (the test-suite spec)

- [ ] The [test-suite spec](sage-tools-api-test-suite-spec.md)'s §7 checklist is complete, and `npm run verify` is green.

**Phase 1**

- [ ] `apps-script/` holds the `.gs` files, their harness and fixtures; `scripts/` holds `hash-password.mjs` and the runner.
- [ ] `src/sync/` has `domain/`, `infra/`, `config/`.
- [ ] The Phase 1 grep returns nothing; `npm run verify` green; the service starts.

**Phase 2**

- [ ] `X-Sync-Secret` is compared in constant time.
- [ ] A bad environment value stops startup with a clear message; an unset secret does not.
- [ ] Every error response has `{ error, code }`; no handler builds its own error response.
- [ ] Auth middleware lives in `src/auth/middleware.mjs`.
- [ ] CORS allows `PUT`, `PATCH`, `DELETE`.
- [ ] Day keys `config` and `live-push` are rejected.
- [ ] `jsconfig.json` covers `.mjs`; the new modules pass `// @ts-check`.

**Phase 3**

- [ ] The conflict retry is written once (`withConflictRetry`) and used by the stores and `mergeIntoStore`.
- [ ] Three GitHub conflicts answer `409` (B1).
- [ ] The pure sync logic lives in `sync/domain/` with its own unit tests.

**Phase 4**

- [ ] `SyncService` has no GitHub, Worker, archive or fallback code.
- [ ] Publishing strategies, the selector, the three small services and the controller exist.
- [ ] Every Phase 0 test passes unchanged apart from wiring.

**Phase 5**

- [ ] The six new `/v1` routes exist; parity tests green.
- [ ] Every legacy URL still answers exactly as before; `/ping` untouched.
- [ ] OpenAPI documents all fourteen paths, legacy ones marked deprecated.

**Phase 6**

- [ ] A markdown-only push does not rebuild Cloud Run.
- [ ] The old verify scripts are removed; the workspace `CLAUDE.md` is tracked.
- [ ] Docs describe the new structure.

**Always**

- [ ] `GET /ping` returns `200 PONG!` with `X-App-Version` and `X-Sync-Config`.
- [ ] Nothing was deployed before the owner said so.

---

## 9. Rollout and rollback

- Everything happens on `arch-refactor`. **Nothing merges to `main` until the
  owner agrees**, and not in the few days before or during an event, because
  merging deploys Cloud Run.
- Merge phase by phase if the owner prefers: each phase is independently
  green and independently revertible with `git revert <phase commit>`.
- After deploying, check `GET /ping` for the new version, press **Sync now** in
  a workbook, press **Check connection** in Control Center, and run one
  **Resync this day now**. These exercise the legacy routes the deployed
  clients use.
- Rollback is a revert and a push; the old URLs never changed, so no client
  needs touching.

## 10. Out of scope

- Moving Control Center, the scoresheet generator or Apps Script to `/v1`.
- Tests for the live Worker beyond `smoke.mjs`.
- Replacing the shared-secret and one-operator-login model.
- Rate limiting, request IDs, structured (JSON) logging, metrics.
- The site's shared code (§6.7 is a decision only).
