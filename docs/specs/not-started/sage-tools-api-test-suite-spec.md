# Spec — `sage-tools-api` test suite (unit and integration tests first)

> **Status: not started.** Nothing here is built. Written 2026-10-02 against
> `sage-tools-api` 2.5.0 (live push, the operator switch, the Apps Script
> verify scripts), split out of the
> [architecture hardening spec](sage-tools-api-architecture-spec.md), whose
> Phase 0 this is.
>
> **This spec comes first.** It builds a unit and integration test suite against
> the code **as it is today** and ends green, with no production file changed.
> The architecture spec's Phases 1 to 7 start only after this one's acceptance
> checklist (§7) is complete, and every one of them has to leave this suite
> green.

Pin what `sage-tools-api` does today, in tests, so the refactor in the
architecture spec can change its insides without anyone having to take it on
trust that nothing visible changed.

---

## 0. Read this first (implementer orientation)

Assume no knowledge of this project beyond this page.

**Repos.** `D:\Personal\SAGE` is a plain folder holding four git repos. This
spec changes `sage-tools-api/` (Node 22, Express, ESM `.mjs`, deployed to Google
Cloud Run by a build trigger on every push to `main`) and adds a **Testing**
section to its README. It changes nothing in the other repos.

**Rules that apply to every change** (from the root `CLAUDE.md`):

- Tests and README text do not bump `package.json`'s version, because nothing
  that runs on Cloud Run changes. The only edit to `package.json` is its
  `scripts`.
- Cite a spec from outside `sage-docs` as
  `sage-docs/docs/specs/.../sage-tools-api-test-suite-spec.md`, literally `...`
  where the status folder goes. Inside `sage-docs`, links carry the real folder.
- Documentation is written in the present tense.
- **Do not merge to `main` on an event day, or in the few days before one.**
  Pushing `sage-tools-api` `main` deploys Cloud Run. Work on a branch
  (`test-suite`); a tests-only branch is safe to deploy, but the owner decides.
- Working copies use CRLF line endings. Keep them.
- No new **runtime** dependency, and no dev dependency either: the runner is
  Node's built-in `node:test`.

**Tooling facts.**

- Node 22 (the `Dockerfile` pins `node:22-slim`). `node --test` with a glob,
  `node:assert/strict`, `mock.fn`, `mock.timers` and
  `--experimental-test-coverage` all work on Node 22.1.
- `GitHubPublisher`, `LivePublisher` and both CSV fetchers call the global
  `fetch`, so tests replace `globalThis.fetch`. They have no injection point
  today and this spec does not add one.
- The HTTP server is `http2.createServer({ allowHTTP1: true }, app)`. Cleartext
  HTTP/1.1 does not work on it (a plain `fetch` to it fails with
  `HPE_INVALID_CONSTANT`), but `server.app` is an ordinary Express app, so tests
  mount it on `http.createServer(server.app)` on port 0 and use `fetch`. This was
  checked: `/ping`, `/sync/config` and `/openapi.json` all answer.

**Commands that exist today** (run from `sage-tools-api/`, each exits non-zero
on failure):

```bash
node scripts/verify-sync-merge.mjs
node scripts/verify-facility-completion.mjs
node scripts/verify-attendance.mjs
node scripts/verify-standard-generator.mjs
node scripts/verify-sheet-generator.mjs
```

**How to work.** One branch, `test-suite`, one commit per step in §6. If a test
fails, the production code is right until proven otherwise (this suite describes
it); fix the test, not the code. If a behaviour looks wrong, pin it as it is,
mark it `// CHARACTERIZATION` (§3.6), and tell the owner.

---

## 1. Goals, non-goals and constraints

**Goals**

1. A unit test for every module that exists today, and an integration test for
   every route, so a later change to any of them fails a test if it changes what
   a deployed client can see.
2. Integration tests of the whole sync pipeline against an in-memory fake of
   GitHub, Google Sheets and the live Worker, including deterministic race
   conditions.
3. The two ad hoc `verify-*` scripts that cover `src/` ported to the same
   runner, with no loss of coverage.
4. Evidence the suite can fail: ten deliberate breakages, each caught.

**Non-goals**

- Changing production code. No refactor, no seam, no new option on any class.
- Tests for the Apps Script files or the live Worker (they keep their own
  harnesses, §3.1).
- Enforcing a coverage threshold.
- Tests for modules the architecture spec creates; those are written in the
  phase that creates them (architecture spec §5.2).

**Constraint.** The suite pins today's behaviour **including known defects**.
It must not "fix" anything to make a test pass or look right.

---

## 2. Behaviour the suite pins

### 2.1 What must not change

For every existing route, the **status code, response body, and the response
headers listed here** are identical before and after the refactor. This suite
pins them; the architecture spec's later phases may not move them.

| Route | Pinned |
|---|---|
| `GET /ping` | `200`, body exactly `PONG!`, `X-App-Version` equal to `package.json`'s version, `X-Sync-Config` as `<7-char sha>/remote`, `seed/fallback`, or absent when nothing is cached. No config fetch. Remains in `Server.mjs`. |
| `GET /openapi.json` | `200`, a valid OpenAPI 3.0.3 document, `servers[0].url` built from the request host |
| `POST /auth/login` | `400` for a missing field, `401` for bad credentials or missing server config, `200 { token, expiresAt }` |
| `POST /sync/:day` | the sync response shape: `day, label, method, facilitiesSynced, facilitiesFailed, facilitiesStale, commitSha, attempts, live?, archive?, timing` (`timing`: `editToRequestMs, fetchMs, publishMs, editToPublishedMs, liveMs, archiveMs`); `X-Edit-At` accepted only within the last hour and at most 5 s in the future; `?facility=`, `?method=csv`; auth by `X-Sync-Secret` **or** a bearer token |
| `POST /sync/:day/live` | bearer token only (the secret is refused); body `{ isLive: true \| false \| "auto" }`; `400` otherwise; `{ day, label, isLive, republished, live?, archive? }` |
| `POST /sync/live-push` | bearer token only; body `{ enabled: boolean }`; `{ enabled, changed, available }` |
| `GET /sync/config` | secret or token; `{ sha, source, loadedAt, ageMs, events, days, live: { enabled, baseUrl, switch, active } }`, never the secret. Control Center reads `live.enabled`, `live.switch` (`"on"`/`"off"`) and `live.baseUrl` |
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

### 2.2 Behaviour pinned now that a later phase changes on purpose

The architecture spec (§3.2) changes four visible behaviours. Each is pinned
here as it is **today**, in a test marked `// CHARACTERIZATION B<n>` (§3.6), so
the phase that changes it edits one named test in the same commit.

| # | Today (what the marked test asserts) | Changed in architecture-spec phase | Marked test |
|---|---|---|---|
| B1 | Three conflicting GitHub commits in a row surface as HTTP `500` (the raw GitHub error has no `statusCode`), for `syncDay`'s GitHub path, `setIsLive` and `setLivePush` | 3 | the pipeline case "three conflicts on GitHub" and the `SyncConfigStore` "three 409s throw" cases |
| B2 | Error bodies are exactly `{ "error": "<message>" }` | 2 | every whole-body assertion on an error response |
| B3 | `Access-Control-Allow-Methods` is exactly `GET, POST, OPTIONS` | 2 | `headers.test.mjs` |
| B4 | A day key of `config` or `live-push` passes config validation | 2 | the `SyncConfigStore` validation cases |

Everything else in §2.1 must still pass unchanged after every later phase.

---

## 3. Strategy

### 3.1 Layers

| Layer | What it is | What is real | What is faked |
|---|---|---|---|
| **Unit** | one module in isolation | the module | its collaborators (hand-written fakes), or `globalThis.fetch` for the four HTTP clients |
| **Integration** | the real wiring, driven over HTTP | `Server`, routes, middleware, `SyncService`, `SyncConfigStore`, publishers, fetchers | the three outside services (GitHub, Google Sheets, the live Worker), replaced at `globalThis.fetch` by one in-memory `FakeWorld` (§5.4) |
| **End-to-end (opt-in)** | real Chromium renders a real PDF | the scoresheet pipeline | nothing; skipped unless `RUN_PDF_E2E=1` |

The Apps Script harnesses (`verify-attendance`, `verify-standard-generator`,
`verify-sheet-generator`) and the live Worker's `smoke.mjs` stay as they are;
they test code that is not the Cloud Run service. `npm run verify` runs them
beside the new suite.

### 3.2 Tooling and scripts

`node:test` and `node:assert/strict`. No test dependency is added.

```jsonc
// package.json "scripts" — added by this spec; "start" is unchanged
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
the architecture spec's Phase 1 it points at `apps-script/`.) The two `verify-*`
scripts that cover `src/` (`verify-sync-merge`, `verify-facility-completion`) are
ported to `node:test` here; the originals are removed in the architecture spec's
Phase 6.

Coverage is reported, not enforced (Node 22.1 has no threshold flag). Targets
for the owner to eyeball in the report: `src/sync` ≥ 90 % of lines,
`src/auth` ≥ 95 %, `src/server` ≥ 90 %, `src/config` ≥ 95 %.

### 3.3 Layout and naming

```
test/
  helpers/
    logger.mjs         # silentLogger, capturingLogger()
    builders.mjs       # snapshot, facility, config builders (§3.4)
    fakes.mjs          # FakePublisher, FakeLivePublisher (moved from verify-sync-merge.mjs)
    fakeWorld.mjs      # in-memory GitHub + Sheets + Worker behind globalThis.fetch
    http.mjs           # startApp(), multipart()
  unit/<area>/<Module>.test.mjs
  integration/<feature>.test.mjs
  e2e/pdf.e2e.test.mjs
```

- A test file mirrors the module it covers; when the architecture
  spec's Phase 1 or 4 moves a module, its test file moves with it (`git mv`).
- `describe(<module or route>)` then `it("<observable behaviour in plain words>")`.
  Names read as a specification: `it("answers 401 when neither a secret nor a token is sent")`.
- One behaviour per `it`. Arrange, act, assert, in that order, with no logic
  in the assertion.

### 3.4 Helpers (specified so every test builds on the same ones)

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

The architecture spec's Phases 2 and 4 change the `Server` constructor to take the composed modules rather than
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

### 3.5 Rules for every test

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
   log lines, architecture spec §4.5).
5. **Failure paths get as many tests as success paths.** Every `throw` in a
   module has a test that reaches it.
6. **Secrets never appear in test output.** Use obviously fake values.

### 3.6 Characterization policy

These tests describe **what the code does today**, including behaviour that
is a known defect. A test that pins a behaviour the architecture spec's §3.2 will change is written as
it passes today and carries a marker comment:

```js
// CHARACTERIZATION B1 (architecture spec, Phase 3): today three GitHub 409s surface as HTTP 500.
```

In the phase that changes it, the test is edited to the new expectation in the
same commit, and the commit message names the B-number. `grep -rn
"CHARACTERIZATION B" test/` lists what is still pending; after the architecture
spec's Phase 3 that grep returns nothing.


## 4. Coverage matrix: what each module's tests must reach

Every case below is required. Cases for modules that do not exist yet
(`safeEqual`, `conflictRetry`, the auth middleware, `mergeSnapshot` and the other
extracted modules, `loadConfig`, `errorHandler`, the `/v1` routes) are in the
architecture spec §5.2, because they are written first in the phase that creates
the module.

**Unit: shared**

| Module | Cases |
|---|---|
| `errors.mjs` | each class's `statusCode`, `name` and message; `UnknownSyncDayError` lists the valid days; `SyncUpstreamError` joins reasons with `; `; default `AppError` is `500`; subclass `instanceof` chain |
| `Logger.mjs` | `info`→`console.log`, `warn`→`console.warn`, `error`→`console.error` with the error as second argument; line is `<ISO timestamp> :: [<scope>] <msg>`; `child("b")` of `a` has scope `a:b`; a `null` message prints empty |
| `ConcurrencyPool.mjs` | never more than `limit` tasks in flight; results keep input order; a rejection propagates; `limit` of 1 is sequential |

**Unit: auth**

| Module | Cases |
|---|---|
| `AuthService.mjs` | `login`: valid `username:password` against a hash produced the way `scripts/hash-password.mjs` does (scrypt, 16-byte salt, 64-byte key, `<saltHex>:<hashHex>`) returns `{ token, expiresAt }` with `expiresAt = now + ttl`; wrong password, wrong username, malformed hash and a missing `passwordHash` or `tokenSecret` all return `null` (the last logs an error); `verify`: a fresh token true, an expired token false, a tampered payload or signature false, a token with no `.` false, a non-string false, a token signed with another secret false; default TTL is 12 h; the username is never stored (changing it breaks login) |

**Unit: sync domain**

| Module | Cases |
|---|---|
| `facilityCompletion.mjs` | every case in `scripts/verify-facility-completion.mjs`, 1:1: all non-BYE matches scored, partial, BYE by team code, by either player name, case-insensitive `bye`, stamp kept once set, cleared when a score is removed, an empty CSV |
| `SyncService` merge | every scenario in `scripts/verify-sync-merge.mjs`, 1:1 (the 70 checks): scoped sync, carry-forward, conflict retry, three 409s, non-409, `setLiveOverride` retry, `lastEditAt` survival, branch-level conflict, live disabled, empty live object, live conflict, live read/publish throws, three live conflicts, archive 409 and 403, `setLiveOverride` through the Worker and its fallbacks, switch off, stale live object, newer live object, GitHub read failing |

**Unit: sync config**

| Module | Cases |
|---|---|
| `SyncConfigSnapshot.mjs` | `getDay` returns only facilities with a non-blank `sheetId`, `isLive` defaults to `"auto"`, includes `event`; unknown day throws `UnknownSyncDayError`; `repoPathFor` is `<event>/data/<day>.json`; `sheetsFor` falls back day → `defaults` → `CSV`/`STANDINGSCSV`; `knownDays`, `eventKeys`; `livePushOn` is `false` only for an explicit `false` |
| `SyncConfigStore.mjs` | one case per validation rule: wrong `version`; no `events`; event key not a slug; event with no days; day key not a slug; day key declared by two events; missing/blank label; `facilities` not an array; invalid `isLive`; duplicate facility name; blank facility name; non-boolean `livePush`. Day keys `config` and `live-push` are **accepted today** (pinned as `// CHARACTERIZATION B4`). Caching: a second `get` inside the TTL does not fetch; after the TTL it does; concurrent `get`s share one fetch. Failure: a remote failure serves the last good config and renews its TTL; no cache and a failure serves the bundled seed (`source: "fallback"`, `sha: "seed"`); no cache and no seed throws `SyncConfigUnavailableError`; an invalid remote keeps the last good config. `peek()` never fetches. `setIsLive`: sets the value and returns `{ event, label }`; unknown day throws; a 409 is retried keeping a concurrent change to another day; three 409s throw; success clears the cache. `setLivePush`: commits `livePush`, an unchanged value commits nothing, a 409 is retried, a non-boolean in the file is rejected |

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

**Integration: HTTP contract** (`startApp` with faked services; real routes and middleware)

| File | Cases |
|---|---|
| `ping.test.mjs` | `200` and body `PONG!`; `X-App-Version`; `X-Sync-Config` is `abc1234/remote`, `seed/fallback`, or absent (config store `peek()` returning those); `/ping` never calls `get()` |
| `headers.test.mjs` | CORS headers on every route, including a `401` and a `404`; `OPTIONS` on any path answers `204` with the allow headers; `Access-Control-Expose-Headers` lists `X-App-Version, X-Sync-Config`; a `corsOrigin` other than `*` is echoed |
| `openapi.test.mjs` | `GET /openapi.json` is `200` JSON; `servers[0].url` is the request's host; every documented path answers something other than `404` to an unauthenticated request (it exists); every route registered on the app is documented, except `/openapi.json` itself, found by walking `server.app._router.stack` (Express 4 internal; the helper has its own test that it finds the nine routes registered today: `/ping`, `/openapi.json`, the two `/scoresheets`, `/auth/login` and the four `/sync`) |
| `auth.test.mjs` | missing `username` or `password` or a non-string is `400`; wrong password `401`; unconfigured auth `401`; success `200 { token, expiresAt }` and the token then works on a token-only route |
| `sync-routes.test.mjs` | `POST /sync/:day` is `401` with neither header; works with the secret and with a token; a wrong secret is `401`; `?facility=` and `?method=csv` reach the service; `X-Edit-At` reaches it only inside the window (cases from `editAt`); the service result is returned as JSON `200`; an `UnknownSyncDayError` is `400`, `SyncUpstreamError` `502`, `SyncConfigUnavailableError` `503`, a plain `Error` `500` with its message. `POST /sync/:day/live`: the secret is refused (`401`), a token works, an invalid body is `400`, each of `true`/`false`/`"auto"` reaches the service and its result, including `live.published` (which Control Center reads), is returned unchanged. `POST /sync/live-push`: token only, body must be boolean, result passed through, **registered before `/:day`** (a test posts to it and expects the switch handler, not the sync handler). `GET /sync/config`: secret or token, `401` otherwise, payload as §2 with no secret in it |
| `scoresheets-routes.test.mjs` | `/generate` with a fake service: PDF headers and body, field mapping (`evt`, `type`, `out`, `blanks`), a service error is `err.statusCode ?? 500`; `/generate/stream`: a missing field is `400 { error }`, a good run writes the NDJSON lines in order and ends with `done` carrying base64, a failing service writes an `error` line after a `200` and ends the response |

**Integration: the sync pipeline** (`test/integration/sync-pipeline.test.mjs`; real services, `FakeWorld`)

Wired exactly as `index.mjs` wires them (a `buildApp({ world, env })` helper
calls the same composition code `index.mjs` uses: a copy of those lines now, which
the architecture spec's Phase 4 replaces with the shared function).

1. GitHub only (no `LIVE_PUSH_*`): a scoped sync of facility A commits
   `evt/data/day1.json` with A fresh and B carried forward; response shape
   matches §2; `publishedAt` is stamped.
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
   answer `500` today (pinned as `// CHARACTERIZATION B1`; the architecture spec's
   Phase 3 makes it `409`).
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

---

## 5. Proof that the net works

With the suite green, before it is handed over, the implementer makes each of these
deliberate breakages one at a time, confirms **at least one test fails**, and
reverts it. The list goes in the commit message.

1. In `SyncService`, change the stale-object comparison `>` to `>=`.
2. In `handleSync`, widen the `X-Edit-At` window from `3600_000` to `36_000_000`.
3. In `GitHubPublisher`, drop the `sha` from the `PUT` body.
4. In `routes.mjs`, register `/live-push` after `/:day`.
5. In `Server.mjs`, remove `X-Sync-Config` from the exposed headers.
6. In `AuthService.verify`, skip the expiry check.
7. In `SyncConfigStore`, stop clearing the cache after `setIsLive`.
8. In `LivePublisher.publish`, treat `409` as success.
9. In `#buildSnapshot` stop carrying forward
   `completedAt`.
10. In `handleSetLive`, accept the shared secret.

---

## 6. Build steps

**Goal.** A suite that describes today's behaviour, green on the unmodified
production code.

**Rule.** The only non-test files this spec may touch are `package.json`
(scripts), the new `scripts/run-appscript-verifies.mjs` and `README.md`. If a
module cannot be tested without changing it, stop and note it in the hand-off
report (§8). Today none needs it (`Server.app` is public, the clients use the
global `fetch`, services take their dependencies in their constructors).

Steps:

1. Create `test/` per §3.3 and add the `package.json` scripts from §3.2.
2. Write the helpers (§3.4), including `fakeWorld.test.mjs` for the helper itself.
3. Move the fakes out of `scripts/verify-sync-merge.mjs` into `test/helpers/fakes.mjs`
   unchanged.
4. Port `verify-sync-merge.mjs` and `verify-facility-completion.mjs` to
   `node:test`, 1:1: same scenarios, same assertions, one `it` per `check`. The
   old scripts stay in place and keep passing; they are removed in the architecture spec's Phase 6.
5. Write every unit test in §4 for modules that exist today.
6. Write the integration tests in §4 (HTTP contract, then the pipeline).
7. Write the opt-in E2E: with `RUN_PDF_E2E=1`, post a three-row CSV to the real
   pipeline (`createScoresheetService`) and assert the response starts with
   `%PDF-` and is more than 1 KB. Without the variable the test is `skip`ped.
8. Run the sabotage list (§5).
9. README: add a **Testing** section (commands, layout, the characterization
   marker). Do not bump the version.

---

## 7. Acceptance checklist

- [ ] `npm test` and `npm run verify` pass on unmodified production code.
- [ ] Every case in §4 exists, for every module that exists today.
- [ ] The ported sync-merge suite has at least the 70 original assertions, and the
      ported facility-completion suite every original assertion.
- [ ] `test/helpers/fakeWorld.mjs` has its own passing tests (§3.4).
- [ ] All ten sabotage breakages were caught (§5).
- [ ] `git diff --stat` against `main` shows changes only under `test/`,
      `package.json` (scripts only), `scripts/run-appscript-verifies.mjs` and
      `README.md`.
- [ ] `grep -rn "CHARACTERIZATION B" test/` lists B1 to B4 and nothing else.
- [ ] `npm run test:e2e` with `RUN_PDF_E2E=1` passes on a machine with Chromium;
      without the variable it is skipped, not failed.

## 8. Hand-off to the architecture spec

When §7 is complete, report to the owner: the test and assertion counts, the
coverage report's per-folder lines, anything pinned as a defect that the owner
should know about, and any module that could not be tested without a production
change (none is expected). The architecture spec's Phase 1 starts from this
branch. This spec then moves to `implemented/` per `docs/specs/README.md`.

## 9. Out of scope

- Any production-code change, including adding injection points.
- Tests for the Apps Script files, the live Worker, or the site.
- Coverage thresholds, mutation-testing tools, load tests.
- Tests for the modules and routes the architecture spec adds (architecture spec
  §5.2 and Phase 5).
