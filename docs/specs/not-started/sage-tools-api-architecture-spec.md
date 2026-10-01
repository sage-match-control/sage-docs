# Spec — `sage-tools-api` architecture hardening (tests first)

> **Status: not started.** Nothing here is built. Written 2026-10-01 against
> `sage-tools-api` 2.5.0 (live push 2.4.0 plus the operator switch 2.5.0, both
> in the working tree) and the review recorded in §2.
>
> **Prerequisite:** the [Live push delivery](../in-progress/durable-object-push-spec.md)
> code is committed to `main` of `sage-tools-api` and its 70-check
> `scripts/verify-sync-merge.mjs` passes. This spec refactors that code, so
> start from it, not before it.
>
> **Order is the point.** Phase 0 builds a unit and integration test suite
> against the code **as it is today** and ends green. No production file
> changes until it does. Every later phase has to leave that suite green.

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

- Node 22 (the `Dockerfile` pins `node:22-slim`). `node --test` with a glob,
  `node:assert/strict`, `mock.fn`, `mock.timers` and
  `--experimental-test-coverage` all work on Node 22.1.
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
   before any refactor, and kept green throughout.
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
  `/sync/config`, `/auth/login`. The scoresheet generator calls
  `/scoresheets/generate/stream`. None of those can be renamed without
  breaking something already deployed, so they are aliases forever.
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
| F12 | No test runner; checks are ad hoc scripts | `scripts/verify-*.mjs` | 0 |
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

For every existing route, the **status code, response body, and the response
headers listed here** are identical before and after the refactor. Phase 0 pins
them; later phases may not move them.

| Route | Pinned |
|---|---|
| `GET /ping` | `200`, body exactly `PONG!`, `X-App-Version` equal to `package.json`'s version, `X-Sync-Config` as `<7-char sha>/remote`, `seed/fallback`, or absent when nothing is cached. No config fetch. Remains in `Server.mjs`. |
| `GET /openapi.json` | `200`, a valid OpenAPI 3.0.3 document, `servers[0].url` built from the request host |
| `POST /auth/login` | `400` for a missing field, `401` for bad credentials or missing server config, `200 { token, expiresAt }` |
| `POST /sync/:day` | the sync response shape: `day, label, method, facilitiesSynced, facilitiesFailed, facilitiesStale, commitSha, attempts, live?, archive?, timing` (`timing`: `editToRequestMs, fetchMs, publishMs, editToPublishedMs, liveMs, archiveMs`); `X-Edit-At` accepted only within the last hour and at most 5 s in the future; `?facility=`, `?method=csv`; auth by `X-Sync-Secret` **or** a bearer token |
| `POST /sync/:day/live` | bearer token only (the secret is refused); body `{ isLive: true \| false \| "auto" }`; `400` otherwise; `{ day, label, isLive, republished, live?, archive? }` |
| `POST /sync/live-push` | bearer token only; body `{ enabled: boolean }`; `{ enabled, changed, available }` |
| `GET /sync/config` | secret or token; `{ sha, source, loadedAt, ageMs, events, days, live: { enabled, baseUrl, switch, active } }`, never the secret |
| `POST /scoresheets/generate` | multipart `csv` + `evt`, `type`, `out`, `blanks`; `200` PDF with `Content-Type: application/pdf` and `Content-Disposition: attachment; filename="<name>.pdf"` |
| `POST /scoresheets/generate/stream` | `400 { error }` before streaming for a missing field; otherwise `200 application/x-ndjson` lines `parsing`, `rendering`, `merging`, then `done` (with `pdfBase64`) or `error` |
| every response | `X-App-Version`, `Access-Control-Allow-Origin`, `Access-Control-Expose-Headers: X-App-Version, X-Sync-Config`; `OPTIONS` answers `204` |
| error statuses | validation `400`, unauthorized `401`, unknown day `400`, upstream failure `502`, config unavailable `503`, anything else `500` |

Sync semantics that are pinned by the existing `verify-sync-merge.mjs` (70
checks) are part of the contract: facility merge and carry-forward,
`completedAt`, `lastEditAt` survival across a full resync, live-first
publishing with GitHub archive, fallback to GitHub on any Worker failure, the
`409` retries, the operator switch, the stale-object rule (merge into the
newer of the Worker's and GitHub's copy by `publishedAt`).

### 3.2 What changes on purpose

Nothing else changes. These do, and each has a test that is updated in the
phase named.

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
  templates/                     # scoresheet HTML/CSS, unchanged
  test/                          # Phase 0
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
## 5. Test strategy

### 5.1 Layers

| Layer | What it is | What is real | What is faked |
|---|---|---|---|
| **Unit** | one module in isolation | the module | its collaborators (hand-written fakes), or `globalThis.fetch` for the four HTTP clients |
| **Integration** | the real wiring, driven over HTTP | `Server`, routes, middleware, `SyncService`, `SyncConfigStore`, publishers, fetchers | the three outside services (GitHub, Google Sheets, the live Worker), replaced at `globalThis.fetch` by one in-memory `FakeWorld` (§5.4) |
| **End-to-end (opt-in)** | real Chromium renders a real PDF | the scoresheet pipeline | nothing; skipped unless `RUN_PDF_E2E=1` |

The Apps Script harnesses (`verify-attendance`, `verify-standard-generator`,
`verify-sheet-generator`) and the live Worker's `smoke.mjs` stay as they are;
they test code that is not the Cloud Run service. `npm run verify` runs them
beside the new suite.

### 5.2 Tooling and scripts

`node:test` and `node:assert/strict`. No test dependency is added.

```jsonc
// package.json "scripts" — added in Phase 0; "start" is unchanged
{
  "start": "node index.mjs",
  "test": "node --test \"test/unit/**/*.test.mjs\" \"test/integration/**/*.test.mjs\"",
  "test:unit": "node --test \"test/unit/**/*.test.mjs\"",
  "test:integration": "node --test \"test/integration/**/*.test.mjs\"",
  "test:e2e": "node --test \"test/e2e/**/*.test.mjs\"",
  "test:coverage": "node --test --experimental-test-coverage \"test/unit/**/*.test.mjs\" \"test/integration/**/*.test.mjs\"",
  "test:appscript": "node scripts/run-appscript-verifies.mjs",
  "verify": "npm test && npm run test:appscript"
}
```

`scripts/run-appscript-verifies.mjs` is a ten-line script that runs the three
Apps Script verify scripts in turn and exits non-zero if any does. (After
Phase 1 it points at `apps-script/`.) The two `verify-*` scripts that cover
`src/` (`verify-sync-merge`, `verify-facility-completion`) are ported to
`node:test` in Phase 0 and removed in Phase 6.

Coverage is reported, not enforced (Node 22.1 has no threshold flag). Targets
for the owner to eyeball in the report: `src/sync` ≥ 90 % of lines,
`src/auth` ≥ 95 %, `src/server` ≥ 90 %, `src/config` ≥ 95 %.

### 5.3 Layout and naming

```
test/
  helpers/
    logger.mjs         # silentLogger, capturingLogger()
    builders.mjs       # snapshot, facility, config builders (§5.4)
    fakes.mjs          # FakePublisher, FakeLivePublisher (moved from verify-sync-merge.mjs)
    fakeWorld.mjs      # in-memory GitHub + Sheets + Worker behind globalThis.fetch
    http.mjs           # startApp(), multipart()
  unit/<area>/<Module>.test.mjs
  integration/<feature>.test.mjs
  e2e/pdf.e2e.test.mjs
```

- A test file mirrors the module it covers; after Phase 1 and 4 moves it
  moves with it (`git mv`).
- `describe(<module or route>)` then `it("<observable behaviour in plain words>")`.
  Names read as a specification: `it("answers 401 when neither a secret nor a token is sent")`.
- One behaviour per `it`. Arrange, act, assert, in that order, with no logic
  in the assertion.

### 5.4 Helpers (specified so every test builds on the same ones)

**`logger.mjs`**

```js
export const silentLogger = { info() {}, warn() {}, error() {}, child() { return silentLogger; } };
export function capturingLogger() {
    const lines = [];
    const log = {
        lines,
        info: m => lines.push(["info", m]), warn: m => lines.push(["warn", m]),
        error: (m, e) => lines.push(["error", m, e]), child: () => log,
    };
    return log;
}
```

**`builders.mjs`**: `facilityRow(name, csv, extra)`, `snapshot({ day, facilities, publishedAt, isLive })`,
`registryConfig({ livePush })` returning a valid `events.json` object with one
event `evt`, days `day1` (facilities `A`, `B`) and `day2`, and `configSnapshot(raw)`
wrapping it in `SyncConfigSnapshot`. Defaults match the fixtures in
`verify-sync-merge.mjs` (`PATH = "evt/data/day1.json"`, facility CSVs
`A-old`/`B-old`, fresh `A-new`/`B-new`).

**`http.mjs`**

```js
import http from "node:http";
import { Server } from "../../src/server/Server.mjs";   // path updates when Server moves

/** Builds the real Server with fakes for whatever is passed, listens on an ephemeral port. */
export async function startApp({ services = {}, version = "9.9.9", corsOrigin = "*", syncSharedSecret = "test-secret" } = {}) {
    const server = new Server({
        getScoresheetService: async () => { throw new Error("scoresheet service not faked"); },
        syncService: {}, syncConfigStore: { peek: () => null, get: async () => { throw new Error("no config"); } },
        authService: { verify: () => false, login: () => null },
        logger: silentLogger, port: 0, corsOrigin, syncSharedSecret, version,
        ...services,
    });
    const listener = http.createServer(server.app);
    await new Promise(r => listener.listen(0, "127.0.0.1", r));
    const baseUrl = `http://127.0.0.1:${listener.address().port}`;
    return { baseUrl, close: () => new Promise(r => listener.close(r)), server };
}
```

After Phase 2/4 the `Server` constructor takes the composed modules rather than
these arguments; `startApp` is updated in the same commit so tests do not change.
`multipart(fields, file)` builds a `FormData` with a `Blob` for the `csv` part.

**`fakeWorld.mjs`**: one object that replaces `globalThis.fetch` and behaves like
the three outside services. It is the riskiest helper, so its contract is exact:

```js
export function createFakeWorld({
    github = { owner: "o", repo: "r", branch: "main" },
    sheets = {},                 // { "<sheetId>": { CSV: [[...]], STANDINGSCSV: [[...]] } }  (tab name -> 2D values)
    worker = { baseUrl: "https://worker.test", secret: "pub-secret" },
} = {}) { /* returns world */ }

world.install()   // sets globalThis.fetch, returns an uninstall function; call in before/after
world.github.files        // Map<path, { json, sha }>   — seed with world.github.set(path, json)
world.worker.objects      // Map<"event/day", { version, snapshot }>
world.calls               // [{ service: "github"|"sheets"|"gviz"|"worker", method, key, status }]
world.fail(service, { method, status = 500, times = 1, mutate })   // next N matching calls answer `status` after running mutate(world)
world.worker.down = true  // every Worker call rejects with TypeError("fetch failed")
world.hold(service, { method })  // the next matching call waits: returns { release() } and, once called, proceeds
```

Behaviour it must reproduce, each pinned by its own test in
`test/unit/helpers/fakeWorld.test.mjs` so a bug in the helper cannot hide a bug
in the service:

- **GitHub Contents API** at
  `https://api.github.com/repos/<owner>/<repo>/contents/<path>`:
  `GET ?ref=<branch>` returns `200 { content: <base64 of the JSON text>, sha }`
  or `404`; `PUT { message, content, branch, sha? }` returns `409` when the file
  exists and `sha` is missing or differs, `422` when `sha` is given but the file
  does not exist, otherwise stores the file with a new sha and returns
  `200 { commit: { sha }, content: { html_url } }`. Requires an
  `Authorization: Bearer` header, else `401`.
- **Google Sheets** `GET https://sheets.googleapis.com/v4/spreadsheets/<id>/values:batchGet`
  with repeated `ranges=` and `key=`: `{ valueRanges: [{ values }, …] }` in the
  order asked; `400` with no `key`; `404` for an unknown id; fewer ranges than
  asked when a tab is missing.
- **Google gviz** `GET https://docs.google.com/spreadsheets/d/<id>/gviz/tq?tqx=out:csv&sheet=<tab>`:
  the tab as CSV text.
- **Live Worker** `GET <base>/snapshot/<event>/<day>` → `200 { version, snapshot }`
  or `404 { version: 0 }`; `POST <base>/publish/<event>/<day>` with
  `{ expectedVersion, snapshot }` → `200 { version, clients: 0 }`, `409 { error:
  "version conflict", version }` when `expectedVersion` is stale, `400` on a bad
  body. Both `401` without the right `X-Publish-Secret`.
- `hold` is what makes races deterministic: start sync A, let it reach its
  GitHub `PUT` (held), run sync B to completion, then `release()` A and observe A
  get `409` and retry. No test may rely on timing or `setTimeout` to interleave.

### 5.5 Rules for every test

1. **No real network, no real clock waits.** Tests that exercise the fetchers'
   retry back-off (1 s, then 2 s) use `mock.timers.enable({ apis: ["setTimeout"] })`
   and `tick`. Nothing sleeps for more than 50 ms.
2. **Independent and order-free.** Each test builds its own fixtures. Anything
   that installs `globalThis.fetch` restores it in `afterEach`.
3. **Deterministic.** No `Math.random`, no dependence on `Date.now()` ordering
   without `mock.timers`'s `Date` control (`apis: ["Date"]`) or a tolerance
   stated in the test.
4. **Assert exactly.** Compare whole response bodies with `assert.deepEqual`.
   `JSON.stringify` equality for contract tests is fine. Do not assert on log
   text except where the log line is itself the contract (the three conflict
   log lines, §4.5).
5. **Failure paths get as many tests as success paths.** Every `throw` in a
   module has a test that reaches it.
6. **Secrets never appear in test output.** Use obviously fake values.

### 5.6 Characterization policy

Phase 0 tests describe **what the code does today**, including behaviour that
is a known defect. A test that pins a behaviour §3.2 will change is written as
it passes today and carries a marker comment:

```js
// CHARACTERIZATION B1 (Phase 3): today three GitHub 409s surface as HTTP 500.
```

In the phase that changes it, the test is edited to the new expectation in the
same commit, and the commit message names the B-number. `grep -rn
"CHARACTERIZATION B" test/` lists what is still pending; after Phase 3 that
grep returns nothing.

### 5.7 Coverage matrix: what each module's tests must reach

`✔` means a test for that behaviour is required in Phase 0 (or, for modules
that do not exist yet, in the phase that creates them, **written before the
module**).

**Unit: shared**

| Module | Cases |
|---|---|
| `errors.mjs` | each class's `statusCode`, `name` and message; `UnknownSyncDayError` lists the valid days; `SyncUpstreamError` joins reasons with `; `; default `AppError` is `500`; subclass `instanceof` chain |
| `Logger.mjs` | `info`→`console.log`, `warn`→`console.warn`, `error`→`console.error` with the error as second argument; line is `<ISO timestamp> :: [<scope>] <msg>`; `child("b")` of `a` has scope `a:b`; a `null` message prints empty |
| `ConcurrencyPool.mjs` | never more than `limit` tasks in flight; results keep input order; a rejection propagates; `limit` of 1 is sequential |
| `safeEqual.mjs` (Phase 2) | equal strings true; different strings of equal and of unequal length false; `undefined` or non-string false; does not throw on empty strings |
| `conflictRetry.mjs` (Phase 3) | returns on first success with `attempts: 1`; retries on `{ ok: false }`, calls `onConflict(n, max)`; throws `ConflictError` (status and `statusCode` 409, code `conflict`) after the limit; a thrown error from `attemptFn` propagates unretried; custom `attempts` honoured |

**Unit: auth**

| Module | Cases |
|---|---|
| `AuthService.mjs` | `login`: valid `username:password` against a hash produced the way `scripts/hash-password.mjs` does (scrypt, 16-byte salt, 64-byte key, `<saltHex>:<hashHex>`) returns `{ token, expiresAt }` with `expiresAt = now + ttl`; wrong password, wrong username, malformed hash and a missing `passwordHash` or `tokenSecret` all return `null` (the last logs an error); `verify`: a fresh token true, an expired token false, a tampered payload or signature false, a token with no `.` false, a non-string false, a token signed with another secret false; default TTL is 12 h; the username is never stored (changing it breaks login) |
| `middleware.mjs` (Phase 2) | `requireAuthToken`: valid bearer passes, missing/garbled/expired `401 { error: "Unauthorized", code: "unauthorized" }`; `requireSyncSecretOrAuthToken`: secret passes, token passes, neither `401`, an empty configured secret never matches an empty header |

**Unit: sync domain** (existing code, then the extracted modules)

| Module | Cases |
|---|---|
| `facilityCompletion.mjs` | every case in `scripts/verify-facility-completion.mjs`, 1:1: all non-BYE matches scored, partial, BYE by team code, by either player name, case-insensitive `bye`, stamp kept once set, cleared when a score is removed, an empty CSV |
| `SyncService` merge | every scenario in `scripts/verify-sync-merge.mjs`, 1:1 (the 70 checks): scoped sync, carry-forward, conflict retry, three 409s, non-409, `setLiveOverride` retry, `lastEditAt` survival, branch-level conflict, live disabled, empty live object, live conflict, live read/publish throws, three live conflicts, archive 409 and 403, `setLiveOverride` through the Worker and its fallbacks, switch off, stale live object, newer live object, GitHub read failing |
| `mergeSnapshot.mjs` (Phase 3) | the merge cases above exercised directly on the pure function: fresh wins; untargeted carried forward; failed facility carried forward and listed `stale`; facility never seen and failed is omitted; nothing to publish throws `SyncUpstreamError`; `completedAt` stamped once and carried; `lastEditAt` from the edit or carried; `publishedAt` starts as `now` |
| `snapshotStamp.mjs` (Phase 3) | `publishedAt` preferred over `generatedAt`; neither gives `0`; `newer(a, b)` picks the later, prefers `a` on a tie, handles `null` on either side |
| `syncTiming.mjs` (Phase 3) | `editToRequestMs`/`editToPublishedMs` are `null` without an edit time; `liveMs`/`archiveMs` `null` when that step did not run; the log line prints `n/a` for `null` and otherwise `<n>ms` in the order `edit→request=… fetch=… publish=… live=… archive=… edit→published=…` |
| `editAt.mjs` (Phase 3) | `parseEditAt(header, now)`: a value within the last hour and at most 5 s ahead is returned; older than an hour, more than 5 s ahead, `NaN`, empty, negative and missing are `null` (today's inline rule in `handleSync`) |

**Unit: sync config**

| Module | Cases |
|---|---|
| `SyncConfigSnapshot.mjs` | `getDay` returns only facilities with a non-blank `sheetId`, `isLive` defaults to `"auto"`, includes `event`; unknown day throws `UnknownSyncDayError`; `repoPathFor` is `<event>/data/<day>.json`; `sheetsFor` falls back day → `defaults` → `CSV`/`STANDINGSCSV`; `knownDays`, `eventKeys`; `livePushOn` is `false` only for an explicit `false` |
| `SyncConfigStore.mjs` | one case per validation rule: wrong `version`; no `events`; event key not a slug; event with no days; day key not a slug; day key declared by two events; missing/blank label; `facilities` not an array; invalid `isLive`; duplicate facility name; blank facility name; non-boolean `livePush`; reserved day keys (Phase 2). Caching: a second `get` inside the TTL does not fetch; after the TTL it does; concurrent `get`s share one fetch. Failure: a remote failure serves the last good config and renews its TTL; no cache and a failure serves the bundled seed (`source: "fallback"`, `sha: "seed"`); no cache and no seed throws `SyncConfigUnavailableError`; an invalid remote keeps the last good config. `peek()` never fetches. `setIsLive`: sets the value and returns `{ event, label }`; unknown day throws; a 409 is retried keeping a concurrent change to another day; three 409s throw; success clears the cache. `setLivePush`: commits `livePush`, an unchanged value commits nothing, a 409 is retried, a non-boolean in the file is rejected |

**Unit: sync infrastructure** (stub `globalThis.fetch`)

| Module | Cases |
|---|---|
| `GitHubPublisher.mjs` | `publish`: `PUT` to `…/contents/<path>` with `Authorization: Bearer`, `Accept: application/vnd.github+json`, `X-GitHub-Api-Version: 2022-11-28`, a `User-Agent`; body has `message`, `branch`, `content` (base64 of `JSON.stringify(json, null, 2)`) and `sha` only when known; a `null` sha triggers a `GET ?ref=<branch>` lookup first, a `404` lookup sends no sha; failure throws `GitHub commit failed: HTTP <status> <detail>` with `err.status`; success returns `{ committed: true, commitSha, htmlUrl }`. `fetchExisting`: `404` gives `{ json: null, sha: null }`; success decodes and parses; another failure throws `GitHub lookup failed: HTTP …` |
| `LivePublisher.mjs` | `enabled` needs both URL and secret; a trailing slash on the URL is dropped; `read`: `404` is `{ version: 0, snapshot: null }`, `200` is `{ version, snapshot }`, `401` and `5xx` throw `live Worker read failed: HTTP <status> <body>`, a network error throws `live Worker read failed: <message>`, a stalled call throws `live Worker read timed out after <n>ms`; `publish`: `200` is `{ ok: true, version }`, `409` is `{ ok: false, conflict: true, version }`, other statuses and network errors throw; event and day are URL-encoded; `X-Publish-Secret` on every call, `Content-Type` only on `POST` |
| `SheetsCsvFetcher.mjs` | URL has two `ranges` (matches tab then standings tab), `key`, `valueRenderOption=FORMATTED_VALUE`; no API key throws before any request; values become CSV with rows padded to the header width and fields quoted when they contain `,`, `"` or a newline, `null` as empty; HTTP error message `<facility>: HTTP <status> <detail>`; fewer than two ranges throws the tab-name hint; a timeout throws `<facility>: timed out after <n>ms`; three attempts in total with 1 s then 2 s waits (mock timers) and the last error thrown; `timeoutMs: 0` sends no abort signal |
| `GvizCsvFetcher.mjs` | the export URL it builds for each tab; both tabs fetched in parallel; same retry and timeout rules as above; same `{ name, matchesCsv, standingsCsv }` shape |

**Unit: scoresheets** (read the file, then pin it)

`CsvService`, `TemplateService`, `PageChunkBuilder`, `ScoresheetConfig` (registry,
unknown type throws `UnknownScoresheetTypeError` naming the valid types) and
`ScoresheetService.generate` with fakes for the browser, renderer and merger:
the progress events in order (`parsing`, `rendering` with `completed`/`total`,
`merging`), blank-row padding, a missing CSV or unknown type is a
`ValidationError`, concurrency is passed through.

**Unit: other**

| Module | Cases |
|---|---|
| `openapiSpec.mjs` | the spec is OpenAPI `3.0.3`, has the four tags and two security schemes, and documents exactly these paths today: `/ping`, `/scoresheets/generate`, `/scoresheets/generate/stream`, `/sync/{day}`, `/sync/{day}/live`, `/sync/live-push`, `/sync/config`, `/auth/login`; it is cached after the first build; `info.version` is the version passed in |
| every `src/**/*.mjs` | one test imports each module dynamically, so a broken import path after a move fails here first (`test/unit/imports.test.mjs`) |
| `loadConfig.mjs` (Phase 2) | each rule in §4.7: empty means unset, `NaN`/negative/fractional throw, `SHEETS_FETCH_TIMEOUT_MS=0` allowed, bad `LIVE_PUSH_URL` throws, all problems reported in one error, unset secrets are `undefined` and listed by the helper that builds the warning, the result is frozen |
| `errorHandler.mjs` (Phase 2) | each class in §4.6 gives its status and `{ error, code }`; an unknown `Error` is `500 internal_error` with its message; `res.headersSent` delegates and writes nothing; the error is logged once |

**Integration: HTTP contract** (`startApp` with faked services; real routes and middleware)

| File | Cases |
|---|---|
| `ping.test.mjs` | `200` and body `PONG!`; `X-App-Version`; `X-Sync-Config` is `abc1234/remote`, `seed/fallback`, or absent (config store `peek()` returning those); `/ping` never calls `get()` |
| `headers.test.mjs` | CORS headers on every route, including a `401` and a `404`; `OPTIONS` on any path answers `204` with the allow headers; `Access-Control-Expose-Headers` lists `X-App-Version, X-Sync-Config`; a `corsOrigin` other than `*` is echoed |
| `openapi.test.mjs` | `GET /openapi.json` is `200` JSON; `servers[0].url` is the request's host; every documented path answers something other than `404` to an unauthenticated request (it exists); every route registered on the app is documented, except `/openapi.json` itself, found by walking `server.app._router.stack` (Express 4 internal; the helper has its own test that it finds the nine routes registered today: `/ping`, `/openapi.json`, the two `/scoresheets`, `/auth/login` and the four `/sync`) |
| `auth.test.mjs` | missing `username` or `password` or a non-string is `400`; wrong password `401`; unconfigured auth `401`; success `200 { token, expiresAt }` and the token then works on a token-only route |
| `sync-routes.test.mjs` | `POST /sync/:day` is `401` with neither header; works with the secret and with a token; a wrong secret is `401`; `?facility=` and `?method=csv` reach the service; `X-Edit-At` reaches it only inside the window (cases from `editAt`); the service result is returned as JSON `200`; an `UnknownSyncDayError` is `400`, `SyncUpstreamError` `502`, `SyncConfigUnavailableError` `503`, a plain `Error` `500` with its message. `POST /sync/:day/live`: the secret is refused (`401`), a token works, an invalid body is `400`, each of `true`/`false`/`"auto"` reaches the service. `POST /sync/live-push`: token only, body must be boolean, result passed through, **registered before `/:day`** (a test posts to it and expects the switch handler, not the sync handler). `GET /sync/config`: secret or token, `401` otherwise, payload as §3.1 with no secret in it |
| `scoresheets-routes.test.mjs` | `/generate` with a fake service: PDF headers and body, field mapping (`evt`, `type`, `out`, `blanks`), a service error is `err.statusCode ?? 500`; `/generate/stream`: a missing field is `400 { error }`, a good run writes the NDJSON lines in order and ends with `done` carrying base64, a failing service writes an `error` line after a `200` and ends the response |

**Integration: the sync pipeline** (`test/integration/sync-pipeline.test.mjs`; real services, `FakeWorld`)

Wired exactly as `index.mjs` wires them (a `buildApp({ world, env })` helper
calls the same composition code `index.mjs` uses; until Phase 4 that is a
copy of those lines, and Phase 4 replaces it with the shared function).

1. GitHub only (no `LIVE_PUSH_*`): a scoped sync of facility A commits
   `evt/data/day1.json` with A fresh and B carried forward; response shape
   matches §3.1; `publishedAt` is stamped.
2. Live on: the Worker object gets the snapshot (version 1), then GitHub gets
   the same snapshot (archive); the response has `live.published: true` and an
   `archive.committed: true`; the order of `world.calls` is Sheets, Worker read,
   GitHub read, Worker publish, GitHub write.
3. Worker down: the sync still succeeds through GitHub, `live.published:
   false` with the reason; no archive field.
4. Worker returns `401` (wrong secret): same fallback, reason names the status.
5. Switch off through `POST /sync/live-push` (token): the next sync does not
   touch the Worker; `GET /sync/config` reports `switch: "off"`, `active:
   false`; the config file in the fake GitHub has `livePush: false`; switching
   back on restores live publishing.
6. Stale Worker object: sync A and B with the switch off, switch on, sync A:
   B's newer data survives (the stale-object rule), in the Worker and in GitHub.
7. **Race:** two facilities' syncs, GitHub `PUT` of the first held; the second
   completes; release; the first gets `409`, re-reads, re-merges, and the final
   file holds both facilities' new data; `attempts` is `2` on the first.
8. **Race on the Worker:** same, holding the Worker publish; both facilities
   survive; `live.version` is `2`.
9. Three conflicts in a row on the Worker fall back to GitHub; three on GitHub
   answer `409` (B1: `500` until Phase 3).
10. Archive fails with `403`: the request succeeds, `archive.committed: false`,
    `commitSha: null`, the Worker holds the data.
11. Live/Hide through the Worker: version increments, `isLive` set, `publishedAt`
    newer, archived; with the Worker down it goes through GitHub with `live.published: false`.
12. Live/Hide on a day never published: `republished: false`, config updated.
13. Full resync keeps `lastEditAt`; `X-Edit-At` inside the window is recorded.
14. A failed fetch (Sheets `500`, after the retry delays with mock timers)
    carries the facility's previous data forward and lists it in
    `facilitiesStale`; every facility failing on an empty day is `502`.
15. Config change takes effect: editing `config/events.json` in the fake GitHub
    (a new sheet id) is used by the next sync after the TTL (mock timers).

### 5.8 Proof that the net works

Before Phase 1, with the suite green, the implementer makes each of these
deliberate breakages one at a time, confirms **at least one test fails**, and
reverts it. The list goes in the Phase 0 commit message.

1. In `SyncService`, change the stale-object comparison `>` to `>=`.
2. In `handleSync`, widen the `X-Edit-At` window from `3600_000` to `36_000_000`.
3. In `GitHubPublisher`, drop the `sha` from the `PUT` body.
4. In `routes.mjs`, register `/live-push` after `/:day`.
5. In `Server.mjs`, remove `X-Sync-Config` from the exposed headers.
6. In `AuthService.verify`, skip the expiry check.
7. In `SyncConfigStore`, stop clearing the cache after `setIsLive`.
8. In `LivePublisher.publish`, treat `409` as success.
9. In `#buildSnapshot` (`mergeSnapshot` after Phase 3), stop carrying forward
   `completedAt`.
10. In `handleSetLive`, accept the shared secret.

---

## 6. Phases

Each phase ends with `npm run verify` green, a version bump where stated, a
Changelog entry, and one commit on `arch-refactor`. Phases are strictly
ordered; do not start one before the previous is green.

| Phase | What | Version | Production code touched |
|---|---|---|---|
| 0 | Test infrastructure and characterization suite | none | none (only `package.json` scripts, `test/`, and one script) |
| 1 | Folder moves | 2.5.1 | paths and imports only |
| 2 | Hardening | 2.5.2 | secret compare, errors, config, auth middleware, CORS, jsconfig |
| 3 | One retry loop, store interface, domain extraction | 2.5.3 | `sync/` internals |
| 4 | Publishing strategies and the `SyncService` split | 2.5.4 | `sync/` internals |
| 5 | The `/v1` API | 2.6.0 | routes, controllers, OpenAPI |
| 6 | Build, docs and workspace | none | none |
| 7 | Site shared-JS decision | n/a | none |

### 6.0 Phase 0 — tests before code

**Goal.** A suite that describes today's behaviour, green on the unmodified
production code.

**Rule.** The only non-test file this phase may touch is `package.json`
(scripts) and the new `scripts/run-appscript-verifies.mjs`. If a module cannot be
tested without changing it, stop and note it in §6.0's report: today none needs
it (`Server.app` is public, the clients use the global `fetch`, services take
their dependencies in their constructors).

Steps:

1. Create `test/` per §5.3 and add the `package.json` scripts from §5.2.
2. Write the helpers (§5.4), including `fakeWorld.test.mjs` for the helper itself.
3. Move the fakes out of `scripts/verify-sync-merge.mjs` into `test/helpers/fakes.mjs`
   unchanged.
4. Port `verify-sync-merge.mjs` and `verify-facility-completion.mjs` to
   `node:test`, 1:1: same scenarios, same assertions, one `it` per `check`. The
   old scripts stay in place and keep passing; they are removed in Phase 6.
5. Write every unit test in §5.7 for modules that exist today.
6. Write the integration tests in §5.7 (HTTP contract, then the pipeline).
7. Write the opt-in E2E: with `RUN_PDF_E2E=1`, post a three-row CSV to the real
   pipeline (`createScoresheetService`) and assert the response starts with
   `%PDF-` and is more than 1 KB. Without the variable the test is `skip`ped.
8. Run the sabotage list (§5.8).
9. README: add a **Testing** section (commands, layout, the characterization
   marker). Do not bump the version.

**Acceptance.**

- `npm test` passes, with every case in §5.7 present for the existing modules.
- `npm run verify` passes.
- The ported suites contain at least as many assertions as the two scripts
  they replace (70 for sync-merge).
- All ten sabotage breakages were caught.
- `git diff --stat` for the phase shows changes only under `test/`,
  `package.json`, `scripts/run-appscript-verifies.mjs` and `README.md`.
- `grep -rn "CHARACTERIZATION B" test/` lists the B1–B4 pins still to change.

### 6.1 Phase 1 — folder moves (2.5.1)

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

### 6.2 Phase 2 — hardening (2.5.2)

Do these in order; for each, write the failing test first.

1. **`shared/safeEqual.mjs`** (F5): hash both strings with SHA-256 and compare
   with `timingSafeEqual`, so length does not leak. Use it for `X-Sync-Secret`.
2. **`config/loadConfig.mjs`** (F9, B6) per §4.7; `index.mjs` calls it once, passes
   the pieces on, and logs the one warning line for unset secrets. A
   `ConfigError` prints every problem and exits with code 1 before listening.
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
   taking `authService` (and the shared secret). Routes import them.
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

### 6.3 Phase 3 — one retry loop, one store interface, domain extraction (2.5.3)

Write the unit tests for each new module from §5.7 **first**, watch them fail
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

### 6.4 Phase 4 — publishing strategies and the `SyncService` split (2.5.4)

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
4. **`PublishingSelector`**: `select(config)` returns `LiveFirstPublishing`
   when `livePublisher.enabled && config.livePushOn`, else the GitHub-only one.
   This is the only place that decision lives.
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

**Acceptance.** Every Phase 0 test passes unchanged except import paths and
constructor wiring; `SyncService.mjs` is under 150 lines and
`SyncService.syncDay` under 60; `grep -n "livePublisher\|GitHubPublisher"
src/sync/SyncService.mjs` finds nothing; there is exactly one place that reads
`livePushOn`; adding a hypothetical third strategy requires no edit to
`SyncService` (state which files a new strategy would touch in the Changelog).

### 6.5 Phase 5 — the `/v1` API (2.6.0)

1. Write the parity tests first (`test/integration/v1.test.mjs`). They are
   table-driven: for each row of §4.8, the legacy and the `/v1` request produce
   the same status (except `201` for sessions), the same body and the same
   headers listed in §3.1. Plus: `PUT` visibility twice is idempotent;
   `PUT` live-push twice gives `changed: false` the second time; `POST
   /v1/scoresheets` with `Accept: application/x-ndjson` streams, with
   `application/pdf` or none returns a PDF, with `text/html` is `406`;
   `POST /v1/sessions` is `201`; an unknown `/v1/...` path is `404 { error,
   code: "not_found" }`; `/ping` has no `/v1` alias.
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
into Postman: **Import → Link** on `/openapi.json`); every Phase 0 legacy test
still green untouched.

### 6.6 Phase 6 — build, docs and workspace

1. **Cloud Build trigger (F18).** The owner runs this once; the implementer
   documents it in `docs/technical/deployment.md`:

   ```bash
   gcloud builds triggers list --project=sage-tools-api
   gcloud builds triggers update github <TRIGGER_NAME> --project=sage-tools-api \
     --ignored-files='live-worker/**,apps-script/**,test/**,scripts/run-appscript-verifies.mjs,**/*.md'
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
5. **CLAUDE.md rules.** Add to the root `CLAUDE.md`: the test commands, "no
   production change without a failing test first", the `/v1` rule, the
   dependency rules of §4.3, and that `apps-script/` is not part of the service.

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
| `sage-tools-api/README.md` | Changelog entry per versioned phase; **Testing** section (Phase 0); updated layout |
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

**Phase 0**

- [ ] `npm test` and `npm run verify` pass on unmodified production code.
- [ ] Every §5.7 case exists for the modules that exist today.
- [ ] The ported sync-merge suite has at least the 70 original assertions.
- [ ] All ten sabotage breakages were caught.
- [ ] Only `test/`, `package.json`, `scripts/run-appscript-verifies.mjs` and `README.md` changed.

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
