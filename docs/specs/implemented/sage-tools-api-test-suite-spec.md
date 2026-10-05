# Spec — `sage-tools-api` test suite (unit and integration tests first)

> **Status: implemented** (2026-10-03). Written 2026-10-02 against
> `sage-tools-api` 2.5.0 (live push, the operator switch, the Apps Script
> verify scripts), split out of the
> [architecture hardening spec](../in-progress/sage-tools-api-architecture-spec.md), whose
> Phase 0 this is. Revised 2026-10-03 against 2.7.0: attendance (2.6.0,
> 2.6.1) and team rosters (2.7.0) are on `main`, and this page covers both.
> Built the same day with no production file changed; §11 records where the
> built suite departs from the text below.
>
> **This spec comes first.** It builds a unit and integration test suite against
> the code **as it is today** and ends green, with no production file changed.
> The architecture spec's Phases 1 to 7 start only after this one's acceptance
> checklist (§7) is complete, and every one of them has to leave this suite
> green.
>
> **The suite is kept, not just built.** From the commit it lands in, every
> change to the API changes its tests in the same commit (§9). The suite
> carries guard tests that fail when a module or route has none, and the
> rule goes into the root `CLAUDE.md` and the README as part of this spec.
>
> **Part of `test/` already exists.** The
> [attendance spec](multi-event-attendance-spec.md) (2.6.0) created `test/`,
> the `test`/`test:unit`/`test:integration` scripts, `test/helpers/logger.mjs`
> and `test/helpers/fakeSheets.mjs`, in this spec's layout, and team rosters
> (2.7.0) added to them. At 2.7.0, `npm test` runs 192 tests in 13 files,
> all green:
>
> | File | Covers |
> |---|---|
> | `test/unit/attendance/*.test.mjs` (6 files) | every `src/attendance/` module except `routes.mjs` |
> | `test/integration/attendance-routes.test.mjs` | the three `/v1` attendance routes over HTTP |
> | `test/unit/auth/AuthService.test.mjs` | desk tokens only; login and the operator token are not tested yet |
> | `test/unit/sync/attendanceConfig.test.mjs` | the `attendance` setting in `events.json` |
> | `test/unit/sync/onFacilitiesSynced.test.mjs` | `SyncService`'s attendance hook |
> | `test/unit/sync/teamRoster.test.mjs` | `teamRoster.mjs`, and the roster cases of `SheetsCsvFetcher`, `sheetsFor` and the `SyncService` merge |
>
> Extend these files; do not recreate or rewrite them, and do not copy their
> cases into new files. What follows from them: `Access-Control-Allow-Methods`
> is `GET, POST, PUT, OPTIONS` (B3); §2.1 includes the three `/v1` routes;
> the OpenAPI path list has 11 paths and the route manifest 12 routes; and
> `src/attendance/routes.mjs` is the only attendance module in
> `coveredElsewhere.mjs`.

Pin what `sage-tools-api` does today, in tests, so the refactor in the
architecture spec can change its insides without anyone having to take it on
trust that nothing visible changed.

---

## 0. Read this first (implementer orientation)

Assume no knowledge of this project beyond this page.

**Repos.** `D:\Personal\SAGE` is a plain folder holding four git repos. This
spec changes `sage-tools-api/` (Node 22, Express, ESM `.mjs`, deployed to Google
Cloud Run by a build trigger on every push to `main`) and adds a **Testing**
section to its README. Outside the repos it adds the test rule to the root
`D:\Personal\SAGE\CLAUDE.md` (§9.4), which no repo tracks. It changes nothing in
the other repos.

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

- Node 22. The `Dockerfile` pins `node:22-slim`, which is the latest 22.x;
  work locally on **22.23.3 or later** (installed with nvm-windows:
  `nvm install 22.23.3`, then `nvm use 22.23.3` from an Administrator
  terminal). Anything below 22.12 is too old for puppeteer, and below 22.8 has
  no coverage thresholds. Checked on 22.23.3: `node --test` with a glob,
  `node:assert/strict`, `mock.fn`, `mock.timers` (including `Date`; it prints
  an `ExperimentalWarning`, which is expected), `t.assert.snapshot` with no
  flag, and `--experimental-test-coverage` with `--test-coverage-include` and
  `--test-coverage-lines`, which fails the run when a threshold is missed.
  Coverage still needs the `--experimental-` flag on Node 22.
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
npm test
npm test
npm test
node apps-script/verify-attendance.mjs
node apps-script/verify-standard-generator.mjs
node apps-script/verify-sheet-generator.mjs
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
4. Evidence the suite can fail: eleven deliberate breakages, each caught.
5. A suite that stays current: every later change to the API updates its tests
   in the same commit, and forgetting to fails a test (§9).

**Non-goals**

- Changing production code. No refactor, no seam, no new option on any class.
- Tests for the Apps Script files or the live Worker (they keep their own
  harnesses, §3.1).
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
| `PUT /v1/days/:day/facilities/:facility/attendance/:key` | operator or desk token; body `{ present: boolean }`, `400` otherwise; `200 { key, player, present, timeIn, withdrawn }` ([attendance spec](multi-event-attendance-spec.md) §4.8) |
| `POST /v1/days/:day/attendance/desk-links` | operator token only (a desk token is `401`); `201 { token, expiresAt, day }` |
| `POST /v1/days/:day/attendance/reconciliations` | operator token only; optional `?facility=`; `200 { day, facilities: [result] }` |
| `POST /scoresheets/generate` | multipart `csv` + `evt`, `type`, `out`, `blanks`; `200` PDF with `Content-Type: application/pdf` and `Content-Disposition: attachment; filename="<name>.pdf"` |
| `POST /scoresheets/generate/stream` | `400 { error }` before streaming for a missing field; otherwise `200 application/x-ndjson` lines `parsing`, `rendering`, `merging`, then `done` (with `pdfBase64`) or `error` |
| every response | `X-App-Version`, `Access-Control-Allow-Origin`, `Access-Control-Expose-Headers: X-App-Version, X-Sync-Config`; `OPTIONS` answers `204` |
| error statuses | validation `400`, unauthorized `401`, unknown day `400`, upstream failure `502`, config unavailable `503`, anything else `500`; from attendance: forbidden `403`, not found and unknown event `404`, `ATTENDANCE` layout `409`, Google failure `502`, Google busy `503` |

Sync semantics that are pinned by the existing `verify-sync-merge.mjs` (70
checks) are part of the contract: facility merge and carry-forward,
`completedAt`, `lastEditAt` survival across a full resync, live-first
publishing with GitHub archive, fallback to GitHub on any Worker failure, the
`409` retries, the operator switch, the stale-object rule (merge into the
newer of the Worker's and GitHub's copy by `publishedAt`). So are two later
additions, pinned by `teamRoster.test.mjs` and `onFacilitiesSynced.test.mjs`:

- **Team rosters.** For an event of `type: "team"`, the snapshot's
  `facilities[]` carry `rosterCsv` (`teamCode,player,level,gender`, CRLF),
  read from the day's `rosterSheetName` tab (default `Teams`). A fetch that
  brings no roster (the `?method=csv` fallback, a workbook without the tab,
  a tab with no recognisable header) keeps the last published `rosterCsv`.
  Other event types never carry one.
- **The attendance hook.** After the publish and archive, `syncDay` awaits
  `onFacilitiesSynced({ day, event, facilities })` with the facilities
  fetched fresh. A throw from it is logged and never changes the sync's
  status or body. `setLiveOverride` does not call it.

### 2.2 Behaviour pinned now that a later phase changes on purpose

The architecture spec (§3.2) changes four visible behaviours. Each is pinned
here as it is **today**, in a test marked `// CHARACTERIZATION B<n>` (§3.6), so
the phase that changes it edits one named test in the same commit.

| # | Today (what the marked test asserts) | Changed in architecture-spec phase | Marked test |
|---|---|---|---|
| B1 | Three conflicting GitHub commits in a row surface as HTTP `500` (the raw GitHub error has no `statusCode`), for `syncDay`'s GitHub path, `setIsLive` and `setLivePush` | 3 | the pipeline case "three conflicts on GitHub" and the `SyncConfigStore` "three 409s throw" cases |
| B2 | Error bodies are exactly `{ "error": "<message>" }` | 2 | every whole-body assertion on an error response |
| B3 | `Access-Control-Allow-Methods` is exactly `GET, POST, PUT, OPTIONS` (`PUT` since the attendance spec) | 2 | `headers.test.mjs` |
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
  "test:snapshots": "node --test --test-update-snapshots \"test/unit/**/*.test.mjs\" \"test/integration/**/*.test.mjs\"",
  "test:coverage": "npm run test:coverage:sync && npm run test:coverage:auth && npm run test:coverage:server",
  "test:coverage:sync": "node --test --experimental-test-coverage --test-coverage-include=\"src/sync/**\" --test-coverage-lines=90 \"test/unit/**/*.test.mjs\" \"test/integration/**/*.test.mjs\"",
  "test:coverage:auth": "node --test --experimental-test-coverage --test-coverage-include=\"src/auth/**\" --test-coverage-lines=95 \"test/unit/**/*.test.mjs\" \"test/integration/**/*.test.mjs\"",
  "test:coverage:server": "node --test --experimental-test-coverage --test-coverage-include=\"src/server/**\" --test-coverage-lines=90 \"test/unit/**/*.test.mjs\" \"test/integration/**/*.test.mjs\"",
  "test:appscript": "node scripts/run-appscript-verifies.mjs",
  "verify": "npm test && npm run test:coverage && npm run test:appscript"
}
```

`scripts/run-appscript-verifies.mjs` is a ten-line script that runs the three
Apps Script verify scripts in turn and exits non-zero if any does. (After
the architecture spec's Phase 1 it points at `apps-script/`.) The two `verify-*`
scripts that cover `src/` (`verify-sync-merge`, `verify-facility-completion`) are
ported to `node:test` here; the originals are removed in the architecture spec's
Phase 6.

Coverage is **enforced** by line, per folder: `src/sync` ≥ 90 %, `src/auth`
≥ 95 %, `src/server` ≥ 90 %. A threshold applies to everything one run
includes, so each folder is its own run of the whole suite (the suite is fast;
§3.5 rule 1). `npm run verify` includes it, so a push to `main` that follows
the rule in §9.4 cannot lower coverage below the floor. Other folders are
reported but have no threshold: `src/scoresheets` is mostly Chromium, covered by
the opt-in E2E, and `src/shared` and `src/docs` are small. The architecture
spec's Phase 2 creates `src/config` and adds `test:coverage:config` at ≥ 95 %
in the same commit (§9.2). If a run falls short, write the missing tests.
Never lower a threshold to make a change pass without the owner agreeing.

`test:snapshots` rewrites the snapshot files (§3.5 rule 7). It is the only
script that changes files under `test/`.

### 3.3 Layout and naming

```
test/
  helpers/
    logger.mjs         # silentLogger, capturingLogger()
    builders.mjs       # snapshot, facility, config builders (§3.4)
    fakes.mjs          # FakePublisher, FakeLivePublisher (moved from verify-sync-merge.mjs)
    fakeWorld.mjs      # in-memory GitHub + Sheets + Worker behind globalThis.fetch
    http.mjs           # startApp(), multipart()
    routes.mjs         # registeredRoutes(app): walks server.app._router.stack
    coveredElsewhere.mjs  # modules with no unit test file, and where they are tested (§9.3)
    routeManifest.mjs  # every route and the integration test file that covers it (§9.3)
  unit/<area>/<Module>.test.mjs
  unit/guards/         # the suite-maintenance guards (§9.3)
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
event `evt`, days `day1` (facilities `A`, `B`) and `day2`, plus a second
event `team` of `type: "team"` with one day `tday1` (facility `T`), and
`configSnapshot(raw)` wrapping it in `SyncConfigSnapshot`. Neither event has
an `attendance` setting, so the attendance hook does nothing in the pipeline
tests; attendance has its own integration test. Defaults match the fixtures in
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
  with repeated `ranges=` (two, or three for a team event's roster tab) and
  `key=`: `{ valueRanges: [{ values }, …] }` in the order asked; `400` with no
  `key`; `404` for an unknown id; `400` with a body containing
  `Unable to parse range: <tab>` when an asked-for tab does not exist, which
  is what the real API answers and what `SheetsCsvFetcher`'s roster fallback
  looks for. (The fetcher's "fewer than two ranges" error is reached in its
  unit test with a stubbed `fetch`, not through `FakeWorld`.)
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
7. **Snapshots only for large generated output.** `t.assert.snapshot` is for
   output too big to write out by hand and meant to change only on purpose:
   the OpenAPI document and each scoresheet type's rendered HTML (§4). Small
   things, like response bodies, status codes and headers, are asserted
   explicitly (rule 4), so the expectation can be read in the test.
   Snapshot input must be deterministic, with a fixed version (`"9.9.9"`),
   fixture and clock. The `.snapshot` files sit beside their tests, are
   committed, and are never edited by hand. When output changes on purpose,
   run `npm run test:snapshots`, read the `.snapshot` diff, and commit it with
   the change. A snapshot diff nobody read is a test nobody ran.

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
| `AuthService.mjs` | `login`: valid `username:password` against a hash produced the way `scripts/hash-password.mjs` does (scrypt, 16-byte salt, 64-byte key, `<saltHex>:<hashHex>`) returns `{ token, expiresAt }` with `expiresAt = now + ttl`; wrong password, wrong username, malformed hash and a missing `passwordHash` or `tokenSecret` all return `null` (the last logs an error); `verify`: a fresh token true, an expired token false, a tampered payload or signature false, a token with no `.` false, a non-string false, a token signed with another secret false; default TTL is 12 h; the username is never stored (changing it breaks login). These go in the existing `AuthService.test.mjs`, beside its desk-token cases |

**Unit: sync domain**

| Module | Cases |
|---|---|
| `facilityCompletion.mjs` | every case in `verify-facility-completion.mjs`, 1:1: all non-BYE matches scored, partial, BYE by team code, by either player name, case-insensitive `bye`, stamp kept once set, cleared when a score is removed, an empty CSV |
| `SyncService` merge | every scenario in `verify-sync-merge.mjs`, 1:1 (the 70 checks): scoped sync, carry-forward, conflict retry, three 409s, non-409, `setLiveOverride` retry, `lastEditAt` survival, branch-level conflict, live disabled, empty live object, live conflict, live read/publish throws, three live conflicts, archive 409 and 403, `setLiveOverride` through the Worker and its fallbacks, switch off, stale live object, newer live object, GitHub read failing. The roster merge (a fresh `rosterCsv` replaces the old one; a fetch without one keeps it) is already in `teamRoster.test.mjs`, and the hook in `onFacilitiesSynced.test.mjs` |
| `teamRoster.mjs` | already tested by `teamRoster.test.mjs`; nothing to add |

**Unit: sync config**

| Module | Cases |
|---|---|
| `SyncConfigSnapshot.mjs` | `getDay` returns only facilities with a non-blank `sheetId`, `isLive` defaults to `"auto"`, includes `event`; unknown day throws `UnknownSyncDayError`; `repoPathFor` is `<event>/data/<day>.json`; `sheetsFor` falls back day → `defaults` → `CSV`/`STANDINGSCSV`; `knownDays`, `eventKeys`; `livePushOn` is `false` only for an explicit `false`. `sheetsFor`'s `rosterSheetName` (the day's value or `Teams` for a team event, `null` otherwise) is already in `teamRoster.test.mjs` |
| `SyncConfigStore.mjs` | one case per validation rule: wrong `version`; no `events`; event key not a slug; event with no days; day key not a slug; day key declared by two events; missing/blank label; `facilities` not an array; invalid `isLive`; duplicate facility name; blank facility name; non-boolean `livePush`; a `rosterSheetName` that is not a non-blank string. (The `attendance` setting's rules are already in `attendanceConfig.test.mjs`.) Day keys `config` and `live-push` are **accepted today** (pinned as `// CHARACTERIZATION B4`). Caching: a second `get` inside the TTL does not fetch; after the TTL it does; concurrent `get`s share one fetch. Failure: a remote failure serves the last good config and renews its TTL; no cache and a failure serves the bundled seed (`source: "fallback"`, `sha: "seed"`); no cache and no seed throws `SyncConfigUnavailableError`; an invalid remote keeps the last good config. `peek()` never fetches. `setIsLive`: sets the value and returns `{ event, label }`; unknown day throws; a 409 is retried keeping a concurrent change to another day; three 409s throw; success clears the cache. `setLivePush`: commits `livePush`, an unchanged value commits nothing, a 409 is retried, a non-boolean in the file is rejected |

**Unit: sync infrastructure** (stub `globalThis.fetch`)

| Module | Cases |
|---|---|
| `GitHubPublisher.mjs` | `publish`: `PUT` to `…/contents/<path>` with `Authorization: Bearer`, `Accept: application/vnd.github+json`, `X-GitHub-Api-Version: 2022-11-28`, a `User-Agent`; body has `message`, `branch`, `content` (base64 of `JSON.stringify(json, null, 2)`) and `sha` only when known; a `null` sha triggers a `GET ?ref=<branch>` lookup first, a `404` lookup sends no sha; failure throws `GitHub commit failed: HTTP <status> <detail>` with `err.status`; success returns `{ committed: true, commitSha, htmlUrl }`. `fetchExisting`: `404` gives `{ json: null, sha: null }`; success decodes and parses; another failure throws `GitHub lookup failed: HTTP …` |
| `LivePublisher.mjs` | `enabled` needs both URL and secret; a trailing slash on the URL is dropped; `read`: `404` is `{ version: 0, snapshot: null }`, `200` is `{ version, snapshot }`, `401` and `5xx` throw `live Worker read failed: HTTP <status> <body>`, a network error throws `live Worker read failed: <message>`, a stalled call throws `live Worker read timed out after <n>ms`; `publish`: `200` is `{ ok: true, version }`, `409` is `{ ok: false, conflict: true, version }`, other statuses and network errors throw; event and day are URL-encoded; `X-Publish-Secret` on every call, `Content-Type` only on `POST` |
| `SheetsCsvFetcher.mjs` | URL has two `ranges` (matches tab then standings tab; the roster tab third when `rosterSheetName` is set, cases already in `teamRoster.test.mjs`), `key`, `valueRenderOption=FORMATTED_VALUE`; no API key throws before any request; values become CSV with rows padded to the header width and fields quoted when they contain `,`, `"` or a newline, `null` as empty; HTTP error message `<facility>: HTTP <status> <detail>`; fewer than two ranges throws the tab-name hint; a timeout throws `<facility>: timed out after <n>ms`; three attempts in total with 1 s then 2 s waits (mock timers) and the last error thrown; `timeoutMs: 0` sends no abort signal |
| `GvizCsvFetcher.mjs` | the export URL it builds for each tab; both tabs fetched in parallel; same retry and timeout rules as above; same `{ name, matchesCsv, standingsCsv }` shape, never with a `rosterCsv` (a team event's CSV fallback keeps the last published roster) |

**Unit: scoresheets** (read the file, then pin it)

`CsvService`, `TemplateService`, `PageChunkBuilder`, `ScoresheetConfig` (registry,
unknown type throws `UnknownScoresheetTypeError` naming the valid types) and
`ScoresheetService.generate` with fakes for the browser, renderer and merger:
the progress events in order (`parsing`, `rendering` with `completed`/`total`,
`merging`), blank-row padding, a missing CSV or unknown type is a
`ValidationError`, concurrency is passed through. `TemplateService` also
renders every type registered in `ScoresheetConfig` from one fixed fixture CSV
and snapshots the HTML (§3.5 rule 7). This is the only check on `templates/`
short of the opt-in E2E, so a template edit shows up as a snapshot diff.

**Unit: other**

| Module | Cases |
|---|---|
| `openapiSpec.mjs` | the spec is OpenAPI `3.0.3`, has the five tags (`health`, `scoresheets`, `sync`, `auth`, `attendance`) and two security schemes, and documents exactly these 11 paths today: `/ping`, `/scoresheets/generate`, `/scoresheets/generate/stream`, `/sync/{day}`, `/sync/{day}/live`, `/sync/live-push`, `/sync/config`, `/auth/login`, `/v1/days/{day}/facilities/{facility}/attendance/{key}`, `/v1/days/{day}/attendance/desk-links`, `/v1/days/{day}/attendance/reconciliations`; it is cached after the first build; `info.version` is the version passed in; the whole document from `getOpenApiSpec("9.9.9")` (its `servers` is empty; `Server` fills it in per request) matches its snapshot (§3.5 rule 7) |
| every `src/**/*.mjs` | one test imports each module dynamically, so a broken import path after a move fails here first (`test/unit/imports.test.mjs`) |

**Unit: guards** (`test/unit/guards/`; what they check is in §9.3)

| File | Cases |
|---|---|
| `every-module-tested.test.mjs` | every `src/**/*.mjs` has a mirrored unit test file or an entry in `coveredElsewhere.mjs`; every entry names a module that exists and a test file that exists; no module has both |
| `every-route-tested.test.mjs` | the routes `registeredRoutes(server.app)` finds equal the routes in `routeManifest.mjs`, as sets; each manifest entry's test file exists and contains the route's path text |

**Integration: HTTP contract** (`startApp` with faked services; real routes and middleware)

| File | Cases |
|---|---|
| `ping.test.mjs` | `200` and body `PONG!`; `X-App-Version`; `X-Sync-Config` is `abc1234/remote`, `seed/fallback`, or absent (config store `peek()` returning those); `/ping` never calls `get()` |
| `headers.test.mjs` | CORS headers on every route, including a `401` and a `404`; `OPTIONS` on any path answers `204` with the allow headers; `Access-Control-Expose-Headers` lists `X-App-Version, X-Sync-Config`; a `corsOrigin` other than `*` is echoed |
| `openapi.test.mjs` | `GET /openapi.json` is `200` JSON; `servers[0].url` is the request's host; every documented path answers something other than `404` to an unauthenticated request (it exists); every route registered on the app is documented, except `/openapi.json` itself, found by `registeredRoutes(server.app)` from `test/helpers/routes.mjs`, which walks `server.app._router.stack` (Express 4 internal; the helper has its own test that it finds the twelve routes registered today: `/ping`, `/openapi.json`, the two `/scoresheets`, `/auth/login`, the four `/sync` and the three `/v1` attendance routes) |
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
16. Team roster: a sync of `team`/`tday1` asks Sheets for three ranges and
    publishes `rosterCsv` on facility `T`; a later `?method=csv` sync keeps
    that `rosterCsv` unchanged; a workbook with no `Teams` tab (the fake
    answers `400 Unable to parse range: Teams`) still syncs, with two Sheets
    calls and no `rosterCsv`; a sync of `evt`/`day1` asks for two ranges and
    publishes none.

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
11. In `SyncService`'s merge, drop the line that keeps the prior `rosterCsv`
    when a fetch brings none.

---

## 6. Build steps

**Goal.** A suite that describes today's behaviour, green on the unmodified
production code.

**Rule.** The only non-test files this spec may touch are `package.json`
(scripts), the new `scripts/run-appscript-verifies.mjs`, the new
`.githooks/pre-push`, `README.md`, and (outside the repo) the root
`CLAUDE.md`. If a
module cannot be tested without changing it, stop and note it in the hand-off
report (§8). Today none needs it (`Server.app` is public, the clients use the
global `fetch`, services take their dependencies in their constructors).

Steps:

1. Create `test/` per §3.3 and add the `package.json` scripts from §3.2.
2. Write the helpers (§3.4), including `fakeWorld.test.mjs` for the helper itself.
3. Move the fakes out of `verify-sync-merge.mjs` into `test/helpers/fakes.mjs`
   unchanged.
4. Port `verify-sync-merge.mjs` and `verify-facility-completion.mjs` to
   `node:test`, 1:1: same scenarios, same assertions, one `it` per `check`. The
   old scripts stay in place and keep passing; they are removed in the architecture spec's Phase 6.
5. Write every unit test in §4 for modules that exist today.
6. Write the integration tests in §4 (HTTP contract, then the pipeline).
7. Write the opt-in E2E: with `RUN_PDF_E2E=1`, post a three-row CSV to the real
   pipeline (`createScoresheetService`) and assert the response starts with
   `%PDF-` and is more than 1 KB. Without the variable the test is `skip`ped.
8. Write the guards (§9.3): `coveredElsewhere.mjs`, `routeManifest.mjs` and
   the two tests in `test/unit/guards/`, and confirm each fails when a module's
   test file is renamed away or a route's manifest line is deleted.
9. Run the sabotage list (§5).
10. Add `.githooks/pre-push` (§9.4) and run `git config core.hooksPath
    .githooks` in this clone; confirm a push with a failing test is refused.
11. README: add a **Testing** section (commands, layout, the characterization
    marker, the maintenance rule and table from §9.1–9.2, the hook setup line).
    Do not bump the version.
12. Root `CLAUDE.md`: add the rule from §9.4. This is the last step, so the
    rule never names tests that do not exist yet.

---

## 7. Acceptance checklist

- [ ] `npm test` and `npm run verify` pass on unmodified production code, on
      Node 22.23.3 or later (`verify` includes the per-folder coverage
      thresholds of §3.2).
- [ ] The OpenAPI and template snapshot files are committed, and
      `npm test` passes without `--test-update-snapshots`.
- [ ] Every case in §4 exists, for every module that exists today.
- [ ] The ported sync-merge suite has at least the 70 original assertions, and the
      ported facility-completion suite every original assertion.
- [ ] `test/helpers/fakeWorld.mjs` has its own passing tests (§3.4).
- [ ] All eleven sabotage breakages were caught (§5).
- [ ] `git diff --stat` against `main` shows changes only under `test/`,
      `package.json` (scripts only), `scripts/run-appscript-verifies.mjs`,
      `.githooks/pre-push` and `README.md`.
- [ ] Both guards in `test/unit/guards/` pass, and each was seen to fail when
      its rule was broken (§6 step 8).
- [ ] The pre-push hook refuses a push with a failing test (§6 step 10).
- [ ] The README's **Testing** section and the root `CLAUDE.md` carry the
      maintenance rule (§9).
- [ ] `grep -rn "CHARACTERIZATION B" test/` lists B1 to B4 and nothing else.
- [ ] `npm run test:e2e` with `RUN_PDF_E2E=1` passes on a machine with Chromium;
      without the variable it is skipped, not failed.

## 8. Hand-off to the architecture spec

When §7 is complete, report to the owner: the test and assertion counts, the
coverage report's per-folder lines, anything pinned as a defect that the owner
should know about, and any module that could not be tested without a production
change (none is expected). The architecture spec's Phase 1 starts from this
branch. This spec then moves to `implemented/` per `docs/specs/README.md`.

## 9. Keeping the suite current

Building the suite once is half the job. It only stays a net if every later
change to the API changes its tests too. This section is the standing rule from
the commit the suite lands in onward, for the owner, for anyone else, and for
Claude Code sessions alike.

### 9.1 The rule

**A change to the API is not finished until its tests are.** The test change
goes in the **same commit** as the code change, never a follow-up. It is the
same kind of rule as the version bump: a commit that touches what runs on
Cloud Run without touching `test/` is incomplete, unless it is one of the
exceptions below.

"The API" here means everything that ships to Cloud Run: `index.mjs`, every
file under `src/`, `templates/`, and `package.json`'s dependencies. It does
**not** mean the Apps Script files, the live Worker or the site, which keep
their own harnesses (§3.1). A change to a `.gs` file updates its own `verify-*`
script in the same way, but that is outside this suite.

The only changes that may skip a test edit are ones no test can observe: a
comment, a log line that is not part of the contract (§3.5 rule 4), a
dependency patch bump. The Changelog entry says so (§9.5).

### 9.2 What each kind of change requires

| Change | Tests in the same commit |
|---|---|
| New module | its mirrored unit test file `test/unit/<area>/<Module>.test.mjs`, with every `throw` reached (§3.5 rule 5). If it is only testable through HTTP or Chromium, an entry in `coveredElsewhere.mjs` naming that test instead |
| New route | its cases in the matching `test/integration/*-routes.test.mjs` (auth, success, every error status), a `routeManifest.mjs` line, its `@openapi` block, and the route count in the `registeredRoutes` helper test and in `openapiSpec.test.mjs` |
| New behaviour in an existing module or route | a new `it` describing it |
| Changed behaviour that a deployed client can see (status, body, a §2.1 header) | the pinned test edited to the new expectation, and the Changelog entry says which client-visible thing changed. If §2.1 lists it, update §2.1 too: the table describes the API as it is |
| Bug fix | a test that fails on the code before the fix and passes after it. If a test pinned the bug (`// CHARACTERIZATION`), that test is the one that changes |
| Removed behaviour, module or route | its tests, manifest line or `coveredElsewhere.mjs` entry removed in the same commit |
| Moved or renamed module | its test file moved with `git mv` to mirror the new path |
| A new call to GitHub, Google Sheets or the Worker, or a change in how one is called | `fakeWorld.mjs` taught the new behaviour, with its own case in `fakeWorld.test.mjs`, before the pipeline test that uses it |
| A new `throw` or error class | a test that reaches it, and its status in `errors.test.mjs` |
| A change to the "played" or "BYE" rule | `facilityCompletion.test.mjs`, and the console's copy in `control-center.html` (root `CLAUDE.md`, "kept in sync by hand"); the [site test suite](../not-started/site-test-suite-spec.md)'s `played-bye` parity test runs both copies on the same cases |
| A change to a template, a scoresheet type or an `@openapi` block | `npm run test:snapshots`, and the reviewed `.snapshot` diff committed with it (§3.5 rule 7) |
| A new folder under `src/` that the §3.2 thresholds should cover | its `test:coverage:<folder>` script, added to `test:coverage` |
| Coverage drops below a threshold | more tests, not a lower threshold |

### 9.3 The guards: tests that fail when a test is missing

Two guard tests make the most common omissions fail `npm test`, so the rule
does not rest on memory alone.

- **`every-module-tested.test.mjs`.** Lists `src/**/*.mjs` and, for each,
  expects `test/unit/<path under src>/<Name>.test.mjs`. A module with no unit
  test file must be in `test/helpers/coveredElsewhere.mjs`, a map from module
  path to the test file that covers it and a one-line reason:

  ```js
  export const coveredElsewhere = {
      "src/server/Server.mjs":               ["test/integration/headers.test.mjs", "middleware and mounting; only observable over HTTP"],
      "src/sync/routes.mjs":                 ["test/integration/sync-routes.test.mjs", "route handlers"],
      "src/auth/routes.mjs":                 ["test/integration/auth.test.mjs", "route handlers"],
      "src/scoresheets/routes.mjs":          ["test/integration/scoresheets-routes.test.mjs", "route handlers"],
      "src/attendance/routes.mjs":           ["test/integration/attendance-routes.test.mjs", "route handlers"],
      "src/scoresheets/index.mjs":           ["test/e2e/pdf.e2e.test.mjs", "composes the Chromium pipeline"],
      "src/scoresheets/BrowserManager.mjs":  ["test/e2e/pdf.e2e.test.mjs", "drives real Chromium"],
      "src/scoresheets/PdfRenderer.mjs":     ["test/e2e/pdf.e2e.test.mjs", "drives real Chromium"],
      "src/scoresheets/PdfMergerService.mjs":["test/e2e/pdf.e2e.test.mjs", "merges real PDFs"],
      "src/scoresheets/Workspace.mjs":       ["test/e2e/pdf.e2e.test.mjs", "temp-directory lifecycle of a real run"],
  };
  ```

  (The implementer confirms each entry while writing it; a module that turns
  out to be unit-testable gets a unit test instead.) The guard also fails on a
  stale entry: a module that no longer exists, a test file that does not
  exist, or a module that has both an entry and a unit test file.

- **`every-route-tested.test.mjs`.** Builds the app with `startApp()` and
  compares `registeredRoutes(server.app)` with `test/helpers/routeManifest.mjs`
  as sets of `"<METHOD> <path>"`:

  ```js
  export const routeManifest = [
      { route: "GET /ping",                         test: "test/integration/ping.test.mjs" },
      { route: "GET /openapi.json",                 test: "test/integration/openapi.test.mjs" },
      { route: "POST /auth/login",                  test: "test/integration/auth.test.mjs" },
      { route: "POST /sync/live-push",              test: "test/integration/sync-routes.test.mjs" },
      { route: "POST /sync/:day",                   test: "test/integration/sync-routes.test.mjs" },
      { route: "POST /sync/:day/live",              test: "test/integration/sync-routes.test.mjs" },
      { route: "GET /sync/config",                  test: "test/integration/sync-routes.test.mjs" },
      { route: "POST /scoresheets/generate",        test: "test/integration/scoresheets-routes.test.mjs" },
      { route: "POST /scoresheets/generate/stream", test: "test/integration/scoresheets-routes.test.mjs" },
      { route: "PUT /v1/days/:day/facilities/:facility/attendance/:key", test: "test/integration/attendance-routes.test.mjs" },
      { route: "POST /v1/days/:day/attendance/desk-links",               test: "test/integration/attendance-routes.test.mjs" },
      { route: "POST /v1/days/:day/attendance/reconciliations",          test: "test/integration/attendance-routes.test.mjs" },
  ];
  ```

  A route on the app and not in the manifest fails; so does a manifest line
  for a route that no longer exists, or whose test file is missing or never
  mentions the route's path text. Together with `openapi.test.mjs` (every
  route documented), adding a route without its tests and its documentation
  cannot pass.

The guards check that a test **exists**, not that it is good. What a test must
cover is §9.2, and review checks that.

### 9.4 Where the rule lives

- **Root `CLAUDE.md`**, in the `sage-tools-api` section beside the version-bump
  rule (§6 step 12), so every Claude Code session in the workspace sees it.
  The text to add:

  > Every change to `index.mjs`, `src/` or `templates/` changes `test/` in the
  > same commit: a new module gets its mirrored unit test file, a new route
  > its integration cases and a `test/helpers/routeManifest.mjs` line, a bug
  > fix a test that failed before it, a client-visible change its pinned test
  > edited. The full table is in the README's **Testing** section. Run
  > `npm test` before every commit; `npm run verify` before every push to
  > `main`. Never weaken or delete a test to make a change pass. A failing
  > test means the change is wrong until shown otherwise; if the test is the
  > one that is wrong, say so in the commit message.

- **`sage-tools-api/README.md`**, in the **Testing** section (§6 step 11):
  the rule, the §9.2 table, and the hook setup line.

- **`.githooks/pre-push`**, a POSIX `sh` script that runs `npm test` and
  refuses the push if it fails (Git for Windows runs it too; commit it with
  `git update-index --chmod=+x .githooks/pre-push` so it is executable on a
  Mac or Linux clone). Git does not
  enable a versioned hook by itself, so each clone runs this once, and the
  README says so:

  ```bash
  git config core.hooksPath .githooks
  ```

  The hook is the last check before a push to `main` deploys to Cloud Run. It
  can be bypassed with `git push --no-verify`. Do that only when the owner
  says so, for an emergency fix during an event, and follow it with the
  missing tests before anything else is pushed.

### 9.5 The Changelog shows it

Every Changelog entry for a code change ends with a **Tests:** line naming
the test files added or changed, for example
`Tests: sync-routes.test.mjs (X-Edit-At window), editAt.test.mjs (new)`. An
entry that changes no test says why: `Tests: none, comment-only`. A version
bump whose entry has no **Tests:** line is the visible sign that the rule was
skipped.

### 9.6 The architecture spec's phases follow it too

Every phase of the architecture spec is a change to the API, so the rule
applies there unchanged. The architecture spec's own "write the failing test
first" instruction is stricter and takes precedence. When a phase moves or
adds modules or routes, `coveredElsewhere.mjs` and `routeManifest.mjs` change
in that phase's commit. Phase 5's `/v1` routes each get a manifest line
pointing at `test/integration/v1.test.mjs`.

## 10. Out of scope

- Any production-code change, including adding injection points.
- Tests for the Apps Script files, the live Worker, or the site.
- Coverage thresholds for `src/scoresheets`, `src/shared` and `src/docs`
  (§3.2), mutation-testing tools, load tests.
- Running the suite in the Cloud Run build (a `RUN npm test` in the
  `Dockerfile`) or in GitHub Actions. That would change the deploy pipeline,
  which this spec does not touch. The pre-push hook (§9.4) is the gate
  instead.
- Tests for the modules and routes the architecture spec adds (architecture spec
  §5.2 and Phase 5).

## 11. As built

Where the suite as built departs from the text above. Everything else is as
written.

- **Size.** 777 tests in 43 test files under `test/unit/` and
  `test/integration/`, plus the opt-in end-to-end test. Line coverage is 99.6 %
  for `src/sync`, 99.0 % for `src/auth` and 92.1 % for `src/server`.
- **Two modules are unit-tested instead of listed in `coveredElsewhere.mjs`.**
  `Workspace.mjs` (a real temp directory) and `PdfRenderer.mjs` (a fake page)
  turned out to be unit-testable, which §9.3 says to prefer, and the guard fails
  on a module that has both. `coveredElsewhere.mjs` lists eight modules: the
  five route files and `Server.mjs` over HTTP, and `BrowserManager`,
  `PdfMergerService` and `scoresheets/index.mjs` through the end-to-end test.
- **Helpers beyond §3.4.** `buildApp.mjs` (the pipeline's copy of `index.mjs`'s
  composition), `auth.mjs` (a real `AuthService` with token shortcuts) and
  `timers.mjs` (`withTimers`, `settle`, `waitFor`). `FakeWorld`'s `hold()`
  also returns a `reached` promise, so a test can wait for the held call to
  arrive, and a Sheets entry in `world.calls` records its `ranges`.
  `http.mjs`'s `call()` keeps a reference to the real `fetch`, because a
  `FakeWorld` replaces `globalThis.fetch` and would otherwise intercept the test's
  own requests to the app.
- **Retry delays in the pipeline tests.** Case 14 releases the fetchers' waits
  with `mock.timers.tick` only after `waitFor` has seen the next Sheets call
  arrive, because the request crosses a real socket and a free-running tick would
  race it. Case 15 mocks `Date` for the config's TTL.
- **Snapshots.** The scoresheet templates inline their logo as base64, so the
  template snapshot collapses each data URI to a hash of its bytes: a changed
  image still changes the snapshot, which stays about 150 KB.
- **Calls are listed in invocation order.** In the GitHub race case the held
  `PUT` is the first entry (it answered 409), then the second sync's, then the
  retry.
- **A test the checklist did not ask for.** The sabotage run found that nothing
  caught dropping `completedAt`'s carry-forward in the merge (breakage 9), so
  `SyncService.test.mjs` gained a `completedAt` group.
- **The hook runs `node --test` directly, not `npm test`.** From GitHub Desktop
  on Windows, `npm` resolves to its shell-script wrapper, which runs through WSL
  bash and fails on a `D:\` path. The command in `.githooks/pre-push` has to be
  kept the same as the `test` script.
- **A defect, pinned and not fixed.** `AuthService.login` accepts any username and
  password when `AUTH_PASSWORD_HASH`'s key part is not hex (for example
  `zz:zz`): `Buffer.from("zz", "hex")` is empty, scrypt of a zero-length key is
  empty, and two empty buffers compare equal. The test is marked
  `// CHARACTERIZATION` with no B-number. It needs its own change, with a version
  bump.
