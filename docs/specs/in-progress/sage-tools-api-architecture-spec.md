# Spec — `sage-tools-api` architecture hardening (tests first)

> **Status: in progress.** Phases 1 to 5 (folder moves, 2.8.1; hardening and one composition root, 2.8.2; one retry loop, a store interface and domain extraction, 2.8.3; delivery strategies and the `SyncService` split, 2.8.4; the `/v3` API, 3.0.0) are merged and
> deployed: Cloud Run serves 3.0.0 since 2026-10-06. Phase 6's code is merged:
> the site (Control Center, the scoresheet generator, every copy of the
> `ATTENDANCE CLIENT` and `SCORE CLIENT` blocks) calls `/v3`, and the repo's
> `sheets-sync.gs` calls `/v3`, checked by `verify-sheets-sync.mjs`. Left of
> Phase 6: pasting that script into the two masters, checking a fresh copy of
> each, and the production acceptance in §6.6. Phase 7's docs and root
> `CLAUDE.md` rules are done and the workspace is a local repo; the
> build-trigger filter and the workspace's GitHub repo are the owner's.
> Phase 8's decision record is written and awaits the owner's decision.
> Last revised 2026-10-06; before that 2026-10-05
> against `sage-tools-api` **2.8.0** (`main` at `daad5a4`, score entry), where
> `npm test` runs 957 tests, all passing. Earlier revisions were written
> against 2.5.0 (2026-10-01/02) and 2.7.0 (2026-10-03).
>
> **What the 2026-10-05 revision changed.** Attendance (2.6.0), team rosters
> (2.7.0), the series-final rule (2.7.1) and score entry (2.8.0) all landed
> after this spec was first written. The review in §2 was redone against
> 2.8.0, and these decisions changed:
>
> - The event registry (`SyncConfigStore`) serves sync, attendance and score
>   entry. It moves to `src/registry/`, not into `src/sync/`.
> - Every class that calls an outside service moves to `src/clients/`. Score
>   entry reuses attendance's `SheetsClient`, which already imports
>   `scores/scheduleGrid.mjs`, so today the two features import each other.
> - The auth checks are written three times (sync, attendance, scores). One
>   middleware module replaces all three.
> - The conflict-retry loop is written eight times, not six. `setScoreEntry`
>   added one, and Live/Hide has two of its own.
> - Error bodies can carry an extra field (`current` on a score conflict), so
>   the error handler passes extra fields through.
> - Publishing is two delivery strategies (`GitHubOnlyDelivery`,
>   `LiveFirstDelivery`) behind one `SnapshotPublisher` that only chooses
>   between them. Both stores share one error contract, so either can stand in
>   for the other (§4.6).
> - Every interface the code depends on is written down once, as a JSDoc
>   "port" (§4.4). Each consumer is typed against the narrowest port it uses.
> - `Server` receives its routers from the composition root instead of
>   importing every feature's routes.
> - `SyncConfigStore` keeps loading, caching and committing. Validation and
>   the three switch edits become pure functions in `src/registry/domain/`.
> - `SyncService` picks its CSV fetcher from a map keyed by method.
> - §4.0 names the architectural pattern, before and after, and §4.12 checks
>   the target against each SOLID principle.
> - The API's new surface is **`/v3`**, released as **3.0.0**. The URL version
>   matches the package's major version: there is no `/v2`. Every route,
>   including attendance's and score entry's (today under `/v1`), gets a
>   `/v3` twin. `/v1` joins the legacy URLs, frozen exactly as it is.
> - §4.11 sets REST conventions for `/v3`, covering naming, methods, status
>   codes, RFC 9457 problem-details errors, `405`/`415`, a `GET` for every
>   `PUT`, and RFC 9745 deprecation headers on every legacy twin. It lists the
>   few routes that keep an exception, with reasons, and a guard test
>   enforces the rules.
> - `createApp` is a composition function shared by `index.mjs` and the tests,
>   which today copy its wiring by hand (`test/helpers/buildApp.mjs`).
> - Apps Script IntelliSense is kept: the old `jsconfig.json` settings move to
>   `apps-script/jsconfig.json`.
> - Phase 1 deletes `verify-sync-merge.mjs` and `verify-facility-completion.mjs`,
>   because its moves would break their imports. Both are already ported to
>   `node:test`.
> - Every current client moves to `/v3` (new Phase 6): Control Center, the
>   scoresheet generator, and the `ATTENDANCE CLIENT` and `SCORE CLIENT`
>   blocks. Apps Script stays on its current URL.
> - Versions: Phases 1–4 are 2.8.1–2.8.4. Phase 5, the `/v3` API, is
>   **3.0.0**, a major release for the API's revamp.
>
> **Gate (open).** The test suite from its own spec,
> [`sage-tools-api-test-suite-spec.md`](../implemented/sage-tools-api-test-suite-spec.md),
> is merged to `main` and green. It was this spec's Phase 0. Every phase below
> has to leave that suite green.
>
> **When to start.** Never on an event day or in the few days before one,
> because merging deploys Cloud Run. Prefer to start after score entry's
> real-Google checks (C1–C8 in §11.2 of the
> [score entry spec](control-center-score-entry-spec.md)). Any
> fix those checks need then lands on `main` before Phase 1 moves the files.
> If `main` has moved past 2.8.0 by then, re-check §2's numbers and take the
> next free version for each phase.

This spec moves `sage-tools-api` from a feature-sliced layered service to a
modular monolith with a ports-and-adapters core (§4.0) that follows the SOLID
principles (§4.12). The result is a shape where:

- each module has one job;
- the conflict-retry logic is written once;
- errors are handled in one place;
- configuration is validated at startup;
- the secret comparison is constant-time;
- new endpoints are named the RESTful way;
- folders are laid out by what each one is for.

None of this changes behaviour a deployed client can see, except the few items
in §3.2. `/ping` stays exactly as it is.

---

## 0. Read this first (implementer orientation)

Assume no knowledge of this project beyond this page. Every file path, class
and function named below exists on `main` at 2.8.0 unless the text says a
phase creates it.

**Repos.** `D:\Personal\SAGE` is a plain folder holding four git repos. This
spec changes:

- `sage-tools-api/`: Node 22, Express 4, ESM `.mjs`, deployed to Google Cloud
  Run by a build trigger on every push to `main`;
- `sage-match-control.github.io/`: the static site, Phase 6 only (two pages);
- docs in `sage-docs/`, plus the root `D:\Personal\SAGE\CLAUDE.md` (not in any
  repo until Phase 7) and `CLAUDE.md`/README files in the other repos, where
  they name a moved path.

It does not change `event-data/` except one documentation path in
`event-data/config/README.md`.

**What the service does** (enough to follow this spec):

- **Sync.** A Google Apps Script trigger in each facility workbook calls
  `POST /sync/:day?facility=<name>`. The service reads that workbook's tabs
  from Google Sheets and merges them into the day's published snapshot. It
  publishes the snapshot to a Cloudflare Worker (the "live Worker", which
  pushes it to open pages over WebSockets), then commits it to the GitHub repo
  `event-data` as the archive. When live push is off or the Worker fails, the
  GitHub commit is the publish. Every write is optimistic: read a version,
  write with it, and on a conflict (HTTP 409) re-read, re-merge and retry, up
  to 3 attempts.
- **The registry.** `event-data/config/events.json` lists events, days and
  facility sheet IDs. `SyncConfigStore` fetches, validates and caches it, and
  writes three switches into it: a day's go-live override (`isLive`), the
  live-push switch (`livePush`) and an event's score-entry mode (`scoreEntry`).
- **Attendance** (`src/attendance/`) and **score entry** (`src/scores/`) write
  to facility workbooks as a Google service account. Score entry then
  publishes by calling `syncService.syncDay(...)`.
- **Scoresheets** (`src/scoresheets/`): CSV in, PDF out, through Chromium.
  This pipeline is imported lazily, on the first scoresheet request.
- **Auth** (`src/auth/AuthService.mjs`): one operator login that issues signed
  tokens. There are also desk tokens (attendance) and scorer tokens (score
  entry), each scoped to one day. Apps Script authenticates with the raw
  shared secret in the `X-Sync-Secret` header.

**Rules that apply to every change** (from the root `CLAUDE.md` and
`sage-tools-api/README.md`):

- Any code change in `sage-tools-api` bumps `package.json`'s version and adds
  a matching entry to the Changelog in `sage-tools-api/README.md`. §6 gives the
  version for each phase. Tests and docs alone do not bump it. Each Changelog
  entry ends with a **Tests:** line naming the test files it touched.
- Every change to `index.mjs`, `src/` or `templates/` changes `test/` in the
  same commit. Follow the table in the README's **Testing → Keeping the suite
  current** section. Never weaken or delete a test to make a change pass. A
  test changes only where §3.2 says the behaviour changes on purpose, or where
  only its wiring or an import path changes. Say which in the commit message.
- Cite a spec from outside `sage-docs` as
  `sage-docs/docs/specs/.../sage-tools-api-architecture-spec.md`: literally
  `...` where the status folder goes. Inside `sage-docs`, links use the real
  folder.
- Write documentation in the present tense, describing what the system does
  now.
- **Never deploy any of this on an event day, or in the few days before one.**
  Pushing `sage-tools-api` `main` deploys Cloud Run. Work on the branch
  `arch-refactor`, and merge only when the owner says so.
- Working copies use CRLF line endings. Keep them: an editor that rewrites a
  whole file to LF turns a small diff into a whole-file one. Use `git mv` for
  every move so history follows.
- No new runtime or dev dependency. The test runner is Node's built-in
  `node:test`.

**Tooling facts.**

- Node 22.23.3 or later locally. The `Dockerfile` pins `node:22-slim` and
  copies only `package*.json`, `index.mjs`, `src/` and `templates/` into the
  image.
- There is no linter and no TypeScript, by design. Phase 2 adds JSDoc types
  checked by `// @ts-check` in new files, which needs no build step.
- The four sync HTTP clients call the global `fetch`, and tests replace
  `globalThis.fetch` with an in-memory fake (`test/helpers/fakeWorld.mjs`).
  `SheetsClient` and `GoogleAccessToken` take a `fetchImpl` option instead.
- The production server is `http2.createServer({ allowHTTP1: true }, app)`.
  Cleartext HTTP/1.1 does not work on it, so tests mount `server.app` on
  `http.createServer(...)` at port 0 and use `fetch`.

**Commands** (run from `sage-tools-api/`; each exits non-zero on failure):

```bash
npm test
```

```bash
npm run verify
```

`npm run verify` runs `npm test`, the coverage thresholds
(`npm run test:coverage`) and the three Apps Script harnesses
(`npm run test:appscript`). Run `npm test` after every step and
`npm run verify` at the end of every phase.

**How to work.** One branch, `arch-refactor`, created from `main`, with one
commit per phase or a few small ones. If a test fails, the production change
is wrong until shown otherwise.

---

## 1. Goals, non-goals and constraints

**Goals**

1. `SyncService` split along its real responsibilities, with the
   conflict-retry loop written once.
2. One error-handling path, one auth-middleware module, one validated
   configuration module, one composition root shared with the tests, a
   constant-time secret comparison and a `jsconfig.json` that checks the
   code.
3. Folders laid out by purpose: Apps Script apart from the service, outbound
   clients in one folder, the registry shared rather than owned by sync, and
   each feature's pure logic in its own `domain/` folder. Dependency rules
   that a test enforces.
4. One REST surface, `/v3` (released as 3.0.0), holding a resource-named twin
   of every route (the RPC-style legacy routes, and attendance's and score
   entry's `/v1` routes). Every existing URL, `/v1` included, is kept as a
   permanent, frozen alias. Every current client moves to `/v3`.
5. A Cloud Build trigger that does not redeploy the service for changes that
   are not part of it, and the workspace `CLAUDE.md` under version control.

**Non-goals**

- Rewriting the scoresheet pipeline, the Apps Script files, the live Worker,
  `AttendanceService` or `ScoreService`.
- TypeScript, a bundler, a framework, a linter or an ORM.
- Changing what any deployed client sends or receives, beyond §3.2.
- Renaming or removing `/ping`, `/sync/...`, `/scoresheets/...` or
  `/auth/...`.

**Constraints**

- `sheets-sync.gs` is pasted into live Google Sheets workbooks, so every
  copy in a workbook keeps calling what it calls today, whatever the repo's
  copy says. It makes two calls, both with `X-Sync-Secret`:
  - `POST /sync/:day?facility=<name>` (with `X-Edit-At`) on every edit, on
    **Sync now**, and as **SAGE → Set up live sync**'s test sync. It shows
    operators `Sync failed (<status>): <body>`;
  - `GET /sync/config` during **Set up live sync**. A `401` means "wrong
    secret"; it also reads the JSON body and reports any other status as
    unreachable.

  Both URLs, their auth, their status codes and their bodies never change.
  Phase 1 only moves the `.gs` files and updates their header comments.
  Phase 6 moves the script to `/v3` **in the two master workbooks only**, so
  workbooks made from them afterwards call `/v3`, while every existing
  workbook keeps the copy it has. The Apps Script generators and
  `attendance.gs` make no call to this API.
- Control Center (`tools/control-center.html`) reads specific fields:
  - `live.published`, `republished` and `archive` from Live/Hide;
  - `live.enabled`, `live.switch` and `live.baseUrl` from `/sync/config`. A
    service without `live.switch` is reported as "older than 2.5.0";
  - `enabled` and `available` from the live-push switch;
  - `token` and `expiresAt` from login. It checks `res.ok`, not
    `status === 200`;
  - on a score `409`, `json.current`.
- The scoresheet generator reads the NDJSON lines from
  `/scoresheets/generate/stream`.
- The attendance desk pages, the scorer page and Control Center call the
  `/v1` attendance and score routes (in the `ATTENDANCE CLIENT` and
  `SCORE CLIENT` blocks, and Mission Control). Copies already in browsers keep
  calling `/v1`, so `/v1` is frozen as it is, like the other legacy URLs.
- Archived event pages and any cached copy of Control Center
  (`tools/sw.js` caches it) keep calling the old URLs, so the old URLs are
  aliases forever.

---

## 2. What this spec answers

The architecture review, redone on 2026-10-05 against 2.8.0. Each finding maps
to a phase.

| # | Finding | Evidence at 2.8.0 | Phase |
|---|---|---|---|
| F1 | `SyncService` has five responsibilities: fetch orchestration, merge policy, publishing with fallback and archive, the Live/Hide override, the live-push switch | `src/sync/SyncService.mjs` is 486 lines with 7 constructor dependencies, and `syncDay` is about 145 lines | 4 |
| F2 | Deciding between live push and GitHub alone is spread across `SyncService` and the routes | `#liveActive` in `syncDay` and `setLiveOverride`; `GET /sync/config` reads `syncService.livePublisher` directly in `src/sync/routes.mjs` | 4 |
| F3 | `GitHubPublisher` and `LivePublisher` do the same job behind different shapes | `fetchExisting`/`publish(sha)` against `read`/`publish(version)` | 3 |
| F4 | The read-modify-write-retry-on-409 loop is written eight times | `SyncService`: `#publishViaGitHub`, `#publishViaLive`, `#archiveToGitHub`, `setLiveOverride`'s GitHub loop, `#overrideViaLive`. `SyncConfigStore`: `setIsLive`, `setLivePush`, `setScoreEntry` | 3, 4 |
| F5 | The shared secret is compared with `===` | `hasValidSyncSecret` in `src/sync/routes.mjs`. `AuthService` and the Worker use `timingSafeEqual` | 2 |
| F6 | Error handling is copy-pasted into every handler, and error bodies have no machine-readable code | `res.status(err.statusCode ?? 500)` in 10 handlers (4 sync, 3 attendance, 1 auth, 1 scoresheets, 1 in scores' `fail` helper used by 3), plus `/openapi.json`'s own body. Express's default HTML error page answers a malformed JSON body | 2 |
| F7 | `POST /sync/live-push` works only because it is registered before `POST /sync/:day`. Day keys are slugs, so a day called `live-push` or `config` would collide | Route order in `src/sync/routes.mjs`. `SLUG_RE` in `SyncConfigStore` allows both | 2, 5 |
| F8 | The auth checks are written three times | Sync: `requireAuthToken`, `requireSyncSecretOrAuthToken`. Attendance: `bearerToken`, `requireOperatorOrDesk`, `requireOperator`. Scores: `bearerToken`, `requireOperatorOrScorer`, `requireOperator` | 2 |
| F9 | `index.mjs` parses environment variables ad hoc, with no validation | An empty `SHEETS_FETCH_TIMEOUT_MS=` becomes `0` and disables the timeout. An empty `AUTH_TOKEN_TTL_MS=` becomes a zero-length token life. `SCORESHEET_CONCURRENCY=abc` silently becomes `8` | 2 |
| F10 | `jsconfig.json` checks nothing in the service | `include: ["**/*.js", "**/*.gs"]` (the code is `.mjs`), target ES2015, `checkJs: false` | 2 |
| F11 | The server comment says the HTTP/1.1 fallback helps; over cleartext it does not work | `Server.start()` | 2 |
| F12 | *(Resolved: the test suite exists.)* | — | — |
| F13 | Four route groups are RPC-style (verbs in paths, `POST` to set state), while attendance and score entry are already resources under `/v1`. Control Center calls both styles, and neither surface follows written conventions (`attendance/{key}` is not a plural collection; errors are not problem details) | `/sync/:day/live`, `/sync/live-push`, `/scoresheets/generate`, `/auth/login` | 5, 6 |
| F14 | CORS allows `GET, POST, PUT, OPTIONS` only | `Server.mjs` | 2 |
| F15 | Three conflicting GitHub commits answer `500`. `409` is the accurate status | The raw GitHub error has `status` but no `statusCode` | 3, 4 |
| F16 | `scripts/` mixes four things: Apps Script sources, their test harness and fixtures, two superseded verify scripts, a dev tool | `scripts/` | 1 |
| F17 | `src/sync/` mixes routes, services, HTTP clients, pure logic and the registry that two other features use | A flat folder of 11 files | 1 |
| F18 | Any push rebuilds Cloud Run, including a `.gs`, Worker or markdown change | The build trigger has no file filter | 7 |
| F19 | The workspace `CLAUDE.md` is not under version control | `D:\Personal\SAGE` is not a repo | 7 |
| F20 | The site repeats code blocks by hand | `LIVE CHANNEL` in 12 files, `ATTENDANCE CLIENT` and `SCORE CLIENT` in 4 each, the played/BYE/series rules in 4 places, team rules and team rosters in 2 each | 8 (decision) |
| F21 | Features import each other's internals | `attendance/SheetsClient` imports `scores/scheduleGrid`; `index.mjs` hands attendance's `SheetsClient` to score entry; `attendance/AttendanceService` imports `parseCsv` from `sync/facilityCompletion`; `attendance/roster` imports `sync/teamRoster` | 1 |
| F22 | The composition is written twice | `test/helpers/buildApp.mjs` is a hand copy of `index.mjs`'s wiring. Score entry had to edit both | 2 |
| F23 | `src/attendance/` and `src/scores/` have no coverage threshold (they are at 98 % lines or more) | `package.json` `test:coverage` covers `src/sync`, `src/auth` and `src/server` only | 1 |
| F24 | No interface is written down. Every service is coded against whatever concrete object it receives, with every method of it visible (`ScoreService` gets all of `AuthService`, `SyncService` and `SheetsClient` to use one or three methods of each) | constructor comments name classes, not contracts | 2, 4 |
| F25 | `Server` imports all five route modules, so a new feature edits it. `SyncConfigStore` mixes loading and caching with validation and three feature-specific edits. `SyncService` chooses its fetcher with an `if` | `Server.mjs`, `SyncConfigStore.mjs` (332 lines), `syncDay` | 2, 3, 4 |

---

## 3. Behaviour contract

### 3.1 What must not change

For every existing route, the status code, the response body and the response
headers that deployed clients read stay identical before and after the
refactor, except for the items in §3.2. The contract is defined in the
[test-suite spec §2.1](../implemented/sage-tools-api-test-suite-spec.md#21-what-must-not-change),
extended by the attendance and score-entry tests, and the suite enforces it.

`GET /ping` stays exactly as it is: `200`, body `PONG!`, the `X-App-Version`
and `X-Sync-Config` headers, no config fetch. It stays in `Server.mjs` and
gets no alias.

Log lines that operators search for keep their text. These are the conflict
lines (`commit conflict (attempt n/3) — re-reading and re-merging`,
`live version conflict (attempt n/3) …`,
`archive commit conflict (attempt n/3) …`), the `timing …` line, the
`done — …` line and `live publish failed, publishing via GitHub instead: …`.
§4.5 and §4.6 list them exactly. Which logger child emits a line may change.

### 3.2 What changes on purpose

Each item is pinned as today's behaviour by tests marked
`// CHARACTERIZATION B<n>` (see the README's Testing section). That marker is
edited to the new expectation in the phase named, and the commit message names
the B-number.

| # | Change | Phase |
|---|---|---|
| B1 | Three conflicting writes in a row answer `409` with `{ error, code: "conflict" }` instead of `500`. `error` keeps today's message (the last GitHub error's text). This covers the three `SyncConfigStore` setters (Live/Hide's config write, the live-push switch, the score-entry switch) in Phase 3, and `syncDay`'s and Live/Hide's GitHub publish in Phase 4 | 3, 4 |
| B2 | Every error body gains a stable `code` next to `error` (§4.7). `error` is unchanged. A score conflict keeps its `current` field | 2 |
| B3 | CORS `Access-Control-Allow-Methods` becomes `GET, POST, PUT, PATCH, DELETE, OPTIONS` | 2 |
| B4 | Config validation rejects a day key of `config` or `live-push` | 2 |
| B5 | `/v3` exists (§4.11): a twin of every legacy and `/v1` route, plus three `GET`s. `/v1` itself does not change | 5 |
| B6 | Process start fails with a clear message when an environment variable is set but unusable (§4.8). An empty variable now counts as unset: `SHEETS_FETCH_TIMEOUT_MS=` and `SYNC_CONFIG_TTL_MS=` use their defaults instead of becoming `0`, and `AUTH_TOKEN_TTL_MS=` uses 12 hours. An unset secret still only disables its feature | 2 |
| B7 | A malformed JSON request body answers `400 { error, code: "bad_request" }` instead of Express's HTML page (Phase 2). An unknown path answers `404 { error, code: "not_found" }` instead of Express's HTML `Cannot GET …` (Phase 5) | 2, 5 |
| B8 | The `/v3` twins of attendance and score entry differ from their `/v1` originals, by design: problem-details errors (still carrying `error`, `code` and `current`), `404` for an unknown day, event or facility in the path (`/v1` answers `400`), `405` with `Allow`, `415` for a non-JSON body, the attendance path `…/people/{personKey}/attendance`, `{ scoreEntry }` in the score-entry body, and reconciliations taking `{ facility }` in a body | 5 |
| B9 | Additive headers. Every `401` carries `WWW-Authenticate: Bearer realm="sage-tools-api"` (Phase 2). Every legacy and `/v1` route with a `/v3` twin carries `Deprecation` and `Link: <…>; rel="successor-version"` (Phase 5). No body changes | 2, 5 |

One change is not client-visible: request-failure log lines become one generic
line per error (§4.7) instead of each handler's own text (`Sync failed for
day=…`, `Login failed for username=…`).

---

## 4. Target architecture

### 4.0 Architectural pattern: today and target

**Today: a feature-sliced, layered monolith with a service layer and manual
dependency injection.**

- **One process, sliced by feature.** One Express process holds every
  feature, and each feature has its own folder (`sync/`, `attendance/`,
  `scores/`, `scoresheets/`, `auth/`).
- **Layered.** Inside a feature, the layers are routes → service → HTTP
  client.
- **Service layer, in the transaction-script style.** Each service method
  runs one use case from top to bottom.
- **Constructor injection, wired by hand** in a composition root
  (`index.mjs`), with no container.

Where it falls short:

- **Dependencies are injected, but nothing names an interface.** Each
  service is written against the concrete shape of the client it happens to
  receive. `SyncService` knows GitHub's sha, the Worker's version, the
  archive and the fallback (F1–F3).
- **Routes do more than routing.** They also do auth, validation and error
  mapping, each feature its own way (F6, F8).
- **`Server` imports every feature.**
- **The features import each other's internals** (F21).

A few patterns already exist in small form:

- a repository with a cache and a fallback (`SyncConfigStore`);
- a facade with lazy loading (`scoresheets/index.mjs` behind
  `getScoresheetService`);
- an observer hook (`onFacilitiesSynced`);
- optimistic concurrency on every write.

**Target: a modular monolith with a ports-and-adapters (hexagonal) core,
applied lightly.**

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
 DOMAIN             */domain/ (pure functions)      PORTS (JSDoc interfaces, §4.4)
                                                       ▲ implemented by
                                                       │
 DRIVEN ADAPTERS    clients/, sync/publishing/ stores and deliveries, registry/
                    │ HTTP
                    ▼
                    GitHub (event-data), the live Worker, Google Sheets

 COMPOSITION ROOT   src/app.mjs builds the adapters and plugs them into the ports
```

| Layer | Folders | May depend on |
|---|---|---|
| Domain | `*/domain/`, `shared/csv.mjs`, `shared/errors.mjs` | nothing but other domain code |
| Ports | `shared/ports.mjs`, `sync/publishing/ports.mjs` (JSDoc only) | domain types |
| Application | `*Service.mjs`, `sync/publishing/mergeIntoStore.mjs`, the deliveries and `SnapshotPublisher` | domain, ports |
| Driven adapters | `clients/`, `registry/`, the two snapshot stores | domain, ports |
| Driving adapters | `*/routes.mjs`, `syncController.mjs`, `auth/middleware.mjs`, `server/` | application (injected), domain for request parsing |
| Composition root | `src/app.mjs`, `index.mjs` | everything |

Design patterns used inside it:

- **Strategy:** the two snapshot deliveries.
- **Facade with selection:** `SnapshotPublisher`.
- **Adapter:** each snapshot store wraps one client.
- **Repository:** `SyncConfigStore`.
- **Higher-order retry policy:** `withConflictRetry` and `mergeIntoStore`.
- **Observer:** `onFacilitiesSynced`.
- **Chain of responsibility:** the Express middleware, ending in
  `errorHandler`.

The §4.3 guard test enforces the dependency direction.

It is applied **lightly**, and these are deliberate trade-offs:

- **Ports are JSDoc typedefs, not runtime interfaces.** `// @ts-check` in new
  files checks them in the editor, and the contract tests (§5.2) check them
  at run time.
- **Error classes carry their HTTP `statusCode`.** In strict hexagonal style,
  a driving adapter would map domain errors to HTTP statuses. That would mean
  a second table to keep in step with `errors.mjs` for no gain in a service
  with one transport.
- **`src/scoresheets/` keeps its internal shape.** Only its routes join the
  pattern.
- **There is no DI container.** `createApp` is plain code.

### 4.1 Layout

The shape after Phase 5. The phase that creates or moves each item is in
brackets.

```
sage-tools-api/
  index.mjs                    # loadConfig, createApp, start. Nothing else  [2]
  package.json                 # coverage scripts per folder               [1, 2]
  jsconfig.json                # .mjs, ES2022                               [2]
  apps-script/                 # NOT part of the Cloud Run service          [1]
    jsconfig.json              # the Apps Script settings of today's root jsconfig
    sheets-sync.gs  sheet-generator.gs  standard-generator.gs  attendance.gs
    mock-apps-script.mjs
    verify-attendance.mjs  verify-sheet-generator.mjs  verify-standard-generator.mjs
    fixtures/
  live-worker/  spikes/  templates/                                         # unchanged
  scripts/
    hash-password.mjs
    run-appscript-verifies.mjs # points at apps-script/                     [1]
  test/
    helpers/  e2e/  integration/
    unit/                      # mirrors src/, folder for folder            [1]
  src/
    app.mjs                    # createApp, createRouters: construct everything [2]
    config/
      loadConfig.mjs           # env -> validated, frozen config            [2]
    server/
      Server.mjs               # middleware order, mounts the routers it is given, /ping, /openapi.json
      cors.mjs  errorHandler.mjs  asyncHandler.mjs                          [2]
      requireJson.mjs  methodNotAllowed.mjs  deprecation.mjs                [5]
    auth/
      AuthService.mjs
      middleware.mjs           # every auth check                           [2]
      routes.mjs               # /auth/login and /v3/sessions               [5]
    shared/
      Logger.mjs  ConcurrencyPool.mjs  errors.mjs
      ports.mjs                # JSDoc-only interfaces shared across folders [2]
      csv.mjs                  # parseCsv, moved out of facilityCompletion  [1]
      safeEqual.mjs                                                         [2]
      conflictRetry.mjs        # COMMIT_ATTEMPTS, withConflictRetry         [3]
    clients/                   # every class that calls an outside service  [1]
      GitHubPublisher.mjs  LivePublisher.mjs
      SheetsCsvFetcher.mjs  GvizCsvFetcher.mjs
      SheetsClient.mjs  GoogleAccessToken.mjs
    registry/                  # event-data/config/events.json              [1]
      SyncConfigStore.mjs      # load, cache, fall back, commit an edit
      SyncConfigSnapshot.mjs  events.seed.json
      domain/                                                               [3]
        validateRegistry.mjs   # validate(), SLUG_RE, RESERVED_DAY_KEYS
        registryEdits.mjs      # the three switch edits, as pure functions
    sync/
      SyncService.mjs          # one sync: fetch, merge, publish, report    [4]
      DayVisibilityService.mjs # the Live/Hide override                     [4]
      SyncSettingsService.mjs  # the live-push switch and GET diagnostics   [4]
      syncController.mjs       # HTTP in, service call, HTTP out            [4]
      routes.mjs               # syncRoutes (/sync) and syncV3Routes (/v3)  [4, 5]
      domain/                                                               [1, 3]
        facilityCompletion.mjs  teamRoster.mjs
        mergeSnapshot.mjs  snapshotStamp.mjs  syncTiming.mjs  editAt.mjs
      publishing/                                                           [3, 4]
        ports.mjs              # SnapshotStore, SnapshotDelivery (JSDoc only)
        StoreUnavailableError.mjs
        GitHubSnapshotStore.mjs  LiveSnapshotStore.mjs                      # adapters
        mergeIntoStore.mjs                                                  # the one read-merge-write loop
        GitHubOnlyDelivery.mjs  LiveFirstDelivery.mjs                       # strategies
        SnapshotPublisher.mjs                                               # chooses a strategy, nothing else
    attendance/
      AttendanceService.mjs  routes.mjs
      domain/  attendanceTab.mjs  personKey.mjs  roster.mjs                 [1]
    scores/
      ScoreService.mjs  routes.mjs
      domain/  scheduleGrid.mjs  scoreRequest.mjs                           [1]
    scoresheets/               # unchanged except routes.mjs                [2, 5]
    docs/
      openapiSpec.mjs
```

### 4.2 Responsibilities

| Module | Its one job | Gets from outside (constructor or arguments) |
|---|---|---|
| `index.mjs` | read the version, load config, call `createApp`, start the server | — |
| `app.mjs` | construct every object and every router, in one place | config, logger |
| `config/loadConfig` | turn the environment into a validated, frozen config object | — |
| `server/Server` | middleware order, mounting the routers it is given, `/ping`, `/openapi.json`, the error handler last | `routers: { path, router }[]`, `peekConfig`, logger, CORS origin, version |
| `server/errorHandler` | map any error to a status and `{ error, code, ...extra }` | logger |
| `auth/middleware` | every Express auth check | `TokenVerifier`, the shared secret |
| `clients/*` | talk HTTP to GitHub, the live Worker or Google; nothing else | credentials, timeouts, logger |
| `registry/SyncConfigStore` | load, cache and fall back; commit one edit with conflict retry | `JsonDocumentStore` |
| `registry/domain/*` | validate `events.json`; apply one switch edit to it | arguments only |
| `*/domain/*` | pure functions: no I/O, no Express, no clients | arguments only |
| `publishing/GitHubSnapshotStore`, `LiveSnapshotStore` | adapt one client to `SnapshotStore` | `JsonDocumentStore` or `LiveChannel` |
| `publishing/mergeIntoStore` | read, build, stamp, write, retry on conflict, against any one store | a `SnapshotStore` |
| `publishing/GitHubOnlyDelivery` | deliver through one store, with no fallback | a `SnapshotStore` |
| `publishing/LiveFirstDelivery` | deliver to a primary store, archive to a second, hand over to a fallback when the primary fails | two `SnapshotStore`s, a `SnapshotDelivery` |
| `publishing/SnapshotPublisher` | choose the delivery for this config; nothing else | two `SnapshotDelivery`s, `LiveAvailability` |
| `sync/SyncService` | one sync: resolve the day, fetch, merge, publish, time, report, call the hook | `fetchers` map of `FacilityFetcher`s, `SnapshotPublishing`, `RegistryReader`, `SyncedHook` |
| `sync/DayVisibilityService` | the Live/Hide override | `DayVisibilitySwitch`, `RegistryReader`, `SnapshotPublishing` |
| `sync/SyncSettingsService` | the live-push switch and the `GET /sync/config` payload | `LivePushSwitch`, `RegistryReader`, `LiveAvailability`, `SnapshotPublishing` |
| `sync/syncController` | parse the request, call a service, write the response | the three sync services |
| `*/routes` | map URLs to handlers, attach middleware, carry the `@openapi` blocks | controller or service, auth middleware |

### 4.3 Dependency rules

A guard test (`test/unit/guards/dependency-rules.test.mjs`, Phase 4) enforces
these.

- **"Imports"** means a static `import … from "…"` or a dynamic `import("…")`
  of a relative path, found **after stripping comments**. JSDoc type
  references such as `@param {import("../shared/ports.mjs").RegistryReader}`
  live in comments and are always allowed. That is how every layer refers to
  a port without depending on an implementation.
- Node built-ins and npm packages are listed where they are allowed.

| # | Files | May import | Never |
|---|---|---|---|
| R0 | `src/shared/ports.mjs`, `src/sync/publishing/ports.mjs` | nothing. Each file is JSDoc typedefs and one `export {};`, with no other code | any runtime code |
| R1 | `src/**/domain/*.mjs` | other `domain/` files, `shared/errors.mjs`, `shared/csv.mjs` | anything else: no `node:` modules, no packages |
| R2 | `src/clients/*.mjs` | `shared/*`, any `domain/` file, `node:` built-ins | `registry/`, services, routes, `express` |
| R3 | `src/registry/**/*.mjs` | `shared/*`, `domain/` files | `clients/` (gets a `JsonDocumentStore` injected), `express` |
| R4 | `src/sync/publishing/*.mjs` | `shared/*`, `sync/domain/*`, its own folder | `clients/`, `registry/`, `express`. A delivery never imports a store class: stores are injected |
| R5 | `src/**/*Service.mjs` | `shared/*`, `domain/` files, `node:` built-ins | `express`, `clients/`, `registry/`, `sync/publishing/` (instances are injected) |
| R6 | `src/**/routes.mjs`, `src/sync/syncController.mjs`, `src/auth/middleware.mjs` | `express`, `multer`, `shared/*`, `server/asyncHandler.mjs`, `server/requireJson.mjs`, `server/methodNotAllowed.mjs`, `server/deprecation.mjs`, `domain/` files of its own feature | `clients/`, `registry/`, `*Service.mjs`, `auth/AuthService.mjs` |
| R7 | a feature folder (`sync/`, `attendance/`, `scores/`, `scoresheets/`) | another feature's `domain/` files only | another feature's services, routes or clients |
| R8 | only `src/app.mjs` and `src/scoresheets/index.mjs` | `new` on a class from `clients/`, `registry/`, `sync/publishing/`, a `*Service.mjs` or `Server` | — |
| R9 | only `src/config/loadConfig.mjs` | reads `process.env` | — |
| R10 | `src/server/*.mjs` | `express`, `http2`, `http2-express`, `shared/*`, `docs/openapiSpec.mjs`, its own folder | any feature folder, `auth/`, `registry/`, `clients/`: routers and `peekConfig` are injected |

`src/scoresheets/` keeps its own internal structure (it already meets these
rules apart from R6's `routes.mjs`, which Phase 2 fixes).

### 4.4 Ports (the interfaces) and the snapshot stores

Every dependency a service, delivery or store receives is typed against a
**port**: a JSDoc typedef naming only the methods that consumer calls. Ports
live in two typedef-only files: `src/shared/ports.mjs` (Phase 2) and
`src/sync/publishing/ports.mjs` (Phase 3). A consumer names the port in its
constructor's JSDoc, for example:

```js
/**
 * @param {{ configStore: import("../shared/ports.mjs").RegistryReader
 *                      & import("../shared/ports.mjs").ScoreEntrySwitch,
 *           sheets: import("../shared/ports.mjs").ScoreSheets,
 *           syncService: import("../shared/ports.mjs").DayPublisher,
 *           authService: import("../shared/ports.mjs").ScorerTokenIssuer,
 *           logger: import("../shared/Logger.mjs").Logger, now?: () => Date }} deps
 */
```

At run time the same full object is passed as today. A port documents and
type-checks the dependency; it does not wrap the object.

| Port | Members | Implemented by | Consumed by |
|---|---|---|---|
| `FacilityFetcher` | `fetchFacility(facility, sheets) → Promise<{ name, matchesCsv, standingsCsv, rosterCsv? }>` | `SheetsCsvFetcher`, `GvizCsvFetcher` | `SyncService` (as `fetchers: { sheets, csv }`) |
| `JsonDocumentStore` | `fetchExisting(path) → { json, sha }`, `publish(path, json, message, sha?) → { commitSha }`; throws errors with `status` | `GitHubPublisher` | `SyncConfigStore`, `GitHubSnapshotStore` |
| `LiveChannel` | `read(event, day) → { version, snapshot }`, `publish(event, day, snapshot, expectedVersion) → { ok, version }` | `LivePublisher` | `LiveSnapshotStore` |
| `LiveAvailability` | `enabled: boolean`, `baseUrl: string` | `LivePublisher` | `SnapshotPublisher`, `SyncSettingsService` |
| `RegistryReader` | `get() → Promise<SyncConfigSnapshot>` | `SyncConfigStore` | every service |
| `DayVisibilitySwitch` | `setIsLive(day, isLive) → { event, label }` | `SyncConfigStore` | `DayVisibilityService` |
| `LivePushSwitch` | `setLivePush(enabled) → { enabled, changed }` | `SyncConfigStore` | `SyncSettingsService` |
| `ScoreEntrySwitch` | `setScoreEntry(event, mode) → { event, scoreEntry, changed }` | `SyncConfigStore` | `ScoreService` |
| `DayPublisher` | `syncDay(day, { facilityName, method, editAt }) → SyncResult` | `SyncService` | `ScoreService` |
| `SyncedHook` | `({ day, event, facilities }) → Promise<void>` | `AttendanceService.reconcileAfterSync` (bound) | `SyncService` (`onFacilitiesSynced`) |
| `TokenVerifier` | `verify`, `verifyDeskToken`, `verifyScorerToken` | `AuthService` | `auth/middleware` |
| `Authenticator` | `login(username, password)` | `AuthService` | `auth/routes` |
| `DeskTokenIssuer` | `issueDeskToken({ day, expiresAt })` | `AuthService` | `AttendanceService` |
| `ScorerTokenIssuer` | `issueScorerToken({ day, expiresAt })` | `AuthService` | `ScoreService` |
| `AttendanceSheets` | `readValues`, `updateValues`, `createAttendanceTab` | `SheetsClient` | `AttendanceService` |
| `ScoreSheets` | `readValues`, `writeScores`, `clearScores` | `SheetsClient` | `ScoreService` |
| `SnapshotPublishing` | `publish({ config, ...PublishArgs })`, `republish({ config, ...RepublishArgs })`, `liveActive(config)` | `SnapshotPublisher` | `SyncService`, `DayVisibilityService`, `SyncSettingsService` |
| `SnapshotStore` | `read`, `write` (below) | `GitHubSnapshotStore`, `LiveSnapshotStore` | `mergeIntoStore`, both deliveries |
| `SnapshotDelivery` | `publish(PublishArgs) → PublishResult`, `republish(RepublishArgs) → RepublishResult` (§4.6) | `GitHubOnlyDelivery`, `LiveFirstDelivery` | `SnapshotPublisher`; `LiveFirstDelivery` (its `fallback`) |

The first sixteen rows go in `src/shared/ports.mjs`. `SnapshotStore` and
`SnapshotDelivery`, with their argument and result types, go in
`src/sync/publishing/ports.mjs`. `SnapshotPublishing` sits in
`shared/ports.mjs` and refers to those types.

**The snapshot store.** Both places a snapshot lives (GitHub and the live
Worker) are optimistic stores. You read a value and a **token**, write back
with that token, and are told if someone else wrote first. The token is the
file's `sha` for GitHub and the `version` for the Worker.

```js
// src/sync/publishing/ports.mjs
/**
 * @typedef {{ event: string, day: string, path: string }} SnapshotRef
 *   path is the event-data repo path, config.repoPathFor(event, day).
 * @typedef {{ snapshot: object|null, token: string|number|null }} StoreRead
 * @typedef {{ ok: true, token: string|number|null, commitSha?: string|null }
 *         | { ok: false, conflict: true, error?: Error }} StoreWrite
 *
 * @typedef {object} SnapshotStore
 * @property {(ref: SnapshotRef) => Promise<StoreRead>} read
 *   Resolves { snapshot: null, token: <empty token> } when nothing is stored.
 * @property {(ref: SnapshotRef, snapshot: object,
 *             opts: { token: string|number|null, message?: string }) => Promise<StoreWrite>} write
 *   Resolves { ok: false, conflict: true } when `token` is stale.
 *
 * Failure contract, the same for every store: any other failure of read or
 * write (network, timeout, auth, 4xx other than the conflict, 5xx) is thrown
 * as a StoreUnavailableError whose message is the original error's message,
 * whose `cause` is the original, whose `store` is the store instance that
 * threw, and whose `status` is copied from the original when it has one.
 */
export {};
```

`StoreUnavailableError` (`src/sync/publishing/StoreUnavailableError.mjs`) is a
plain `Error` subclass, not an `AppError`. It never sets `statusCode`, so a
GitHub failure that reaches a route still answers `500` with the same message
as today.

- **`GitHubSnapshotStore(jsonDocumentStore)`.**
  - `read` calls `fetchExisting(ref.path)` and returns
    `{ snapshot: json, token: sha }`. Both are `null` when the file does not
    exist.
  - `write` calls `publish(ref.path, snapshot, opts.message, opts.token)` and
    returns `{ ok: true, token: null, commitSha }`. A thrown error with
    `status === 409` becomes `{ ok: false, conflict: true, error }`. Any other
    error becomes a `StoreUnavailableError`.
  - With `token: null`, the publisher looks the sha up itself, which is what
    the archive relies on.
- **`LiveSnapshotStore(liveChannel)`.**
  - `read` calls `read(ref.event, ref.day)` and returns
    `{ snapshot, token: version }`.
  - `write` calls `publish(ref.event, ref.day, snapshot, opts.token)` and
    returns `{ ok: true, token: version }`, or `{ ok: false, conflict: true }`
    on a 409.
  - Any thrown error becomes a `StoreUnavailableError`.

Because both stores honour one contract, a delivery takes them as roles
("primary", "archive") and never asks which is which. That is what makes them
substitutable (§4.12, L). One contract test (§5.2) runs against both.

### 4.5 The retry loop, once

```js
// src/shared/conflictRetry.mjs
import { ConflictError } from "./errors.mjs";

export const COMMIT_ATTEMPTS = 3;

/**
 * Runs attemptFn until it succeeds, up to `attempts` times.
 * @template T
 * @param {(attempt: number) => Promise<{ ok: true, value: T } | { ok: false, error?: Error }>} attemptFn
 *        Return { ok: false } for a conflict (optionally with the error that
 *        signalled it); throw for anything else, which propagates unretried.
 * @param {object} [opts]
 * @param {number} [opts.attempts=COMMIT_ATTEMPTS]
 * @param {string} [opts.message="update: 3 conflicting updates in a row"]
 *        The ConflictError's message when the last conflict carried no error.
 * @param {(attempt: number, max: number) => void|Promise<void>} [opts.onConflict]
 *        Awaited after EVERY conflicting attempt, including the last.
 * @returns {Promise<{ value: T, attempts: number }>}
 * @throws {ConflictError} after `attempts` conflicts. Its message is the last
 *         conflict's error.message if it had one, else opts.message.
 */
export async function withConflictRetry(attemptFn, { attempts = COMMIT_ATTEMPTS, message, onConflict } = {}) { /* … */ }
```

`ConflictError` lives in `shared/errors.mjs` (§4.7). It sets `statusCode`,
`status` (kept, because callers and tests read `err.status`) and `code`.

`onConflict` fires after every conflict, so each caller reproduces today's
logging exactly. GitHub-path callers log only `if (n < max)`; live-path
callers always log.

| Caller | Log line (logger) | When |
|---|---|---|
| GitHub publish of a sync | `commit conflict (attempt ${n}/${max}) — re-reading and re-merging` | `n < max` |
| Live publish of a sync | `live version conflict (attempt ${n}/${max}) — re-reading and re-merging` | always |
| GitHub republish (Live/Hide) | `commit conflict (attempt ${n}/${max}) — re-reading and re-applying` | `n < max` |
| Live republish (Live/Hide) | `live version conflict (attempt ${n}/${max}) — re-reading and re-applying` | always |
| Archive commit | `archive commit conflict (attempt ${n}/${max}) — retrying with the live snapshot`, then re-read the live store (§4.6) | `n < max` |
| `SyncConfigStore` setters | `commit conflict (attempt ${n}/${max}) — re-reading and re-applying` | `n < max` |

```js
// src/sync/publishing/mergeIntoStore.mjs
/**
 * Read a base, build the result from it, stamp publishedAt, write it with the
 * base's token, and go round again on a conflict.
 *
 * @param {object} p
 * @param {SnapshotStore} p.store                 where the write goes
 * @param {SnapshotRef} p.ref
 * @param {(ref: SnapshotRef) => Promise<StoreRead>} [p.readBase]
 *        Defaults to ref => store.read(ref). LiveFirstDelivery passes one that
 *        reads both stores and returns the newer (§4.6).
 * @param {(base: object|null) => { snapshot: object, stale: string[] } | null} p.build
 *        null means "nothing to write" (a republish with no snapshot yet).
 *        A throw (e.g. SyncUpstreamError) propagates unretried.
 * @param {(stale: string[]) => string} p.message  commit message (GitHub only)
 * @param {(n: number, max: number) => void} [p.onConflict]
 * @param {string} [p.conflictMessage]               passed to withConflictRetry as `message`
 * @returns {Promise<{ written: false, attempts: number }
 *                 | { written: true, snapshot: object, stale: string[],
 *                     token: any, commitSha: string|null, attempts: number }>}
 */
export async function mergeIntoStore(p) { /* … */ }
```

Each attempt does the following, in this order:

1. `base = await readBase(ref)`.
2. `built = build(base.snapshot)`. If it is `null`, resolve
   `{ written: false, attempts: n }`.
3. Set `built.snapshot.publishedAt = new Date().toISOString()`.
4. `res = await store.write(ref, built.snapshot, { token: base.token, message: message(built.stale) })`.
5. If `res.ok`, resolve. Otherwise report the conflict to `withConflictRetry`
   as `{ ok: false, error: res.error }`.

### 4.6 Delivering a snapshot: two strategies and a selector

Three classes, one job each:

- **`GitHubOnlyDelivery`** delivers through one store.
- **`LiveFirstDelivery`** delivers to a primary store, archives to a second
  store, and hands over to a fallback delivery when the primary fails.
- **`SnapshotPublisher`** decides which delivery a config gets. It is the
  **only** place that decides whether live push is used.

`SyncService` and `DayVisibilityService` depend on `SnapshotPublishing` and
never learn which delivery ran.

```js
// src/sync/publishing/ports.mjs (continued)
/**
 * @typedef {{ ref: SnapshotRef,
 *             build: (base: object|null) => { snapshot: object, stale: string[] },
 *             message: (stale: string[]) => string,
 *             log: Logger }} PublishArgs
 * @typedef {{ ref: SnapshotRef,
 *             mutate: (snapshot: object) => object,   // publishedAt is stamped for you
 *             message: string, label: string, log: Logger }} RepublishArgs
 *
 * @typedef {object} PublishResult
 * @property {object} snapshot
 * @property {string[]} stale
 * @property {number} attempts        primary attempts if the primary publish succeeded, else fallback attempts
 * @property {number} publishedAtMs   Date.now() when the publish that counts finished
 * @property {string|null} commitSha  the GitHub commit; null if the archive failed
 * @property {{ published: boolean, version?: number, error?: string }} [live]   absent unless LiveFirstDelivery ran
 * @property {{ committed: boolean, commitSha?: string, error?: string }} [archive]  present only after a primary publish succeeded
 * @property {number|null} liveMs     time spent on the primary attempt (success or failure); null for GitHubOnlyDelivery
 * @property {number|null} archiveMs  null unless the primary publish succeeded
 *
 * @typedef {{ republished: boolean, live?: object, archive?: object }} RepublishResult
 *
 * @typedef {object} SnapshotDelivery
 * @property {(a: PublishArgs) => Promise<PublishResult>} publish
 * @property {(a: RepublishArgs) => Promise<RepublishResult>} republish
 */
```

**`GitHubOnlyDelivery({ store })`**, where `store` is a `SnapshotStore`:

- `publish(a)`: `mergeIntoStore` over `store` with `a.build` and `a.message`.
  It logs on conflict, when `n < max`:
  `commit conflict (attempt n/max) — re-reading and re-merging`. It returns
  `{ snapshot, stale, attempts, publishedAtMs: Date.now(), commitSha, liveMs: null, archiveMs: null }`.
  Three conflicts throw the `ConflictError` (B1).
- `republish(a)`: `mergeIntoStore` with
  `build = base => base ? { snapshot: a.mutate(base), stale: [] } : null`. It
  logs on conflict, when `n < max`:
  `commit conflict (attempt n/max) — re-reading and re-applying`. It returns
  `{ republished: true }` when it wrote and `{ republished: false }` when there
  was nothing to republish.

**`LiveFirstDelivery({ primary, archive, fallback })`**: two `SnapshotStore`s
and a `SnapshotDelivery`. `app.mjs` passes the live store as `primary`, the
GitHub store as `archive`, and the `GitHubOnlyDelivery` as `fallback`. The
class never imports or names a store class. Its log lines say "live" and
"GitHub" because operators search for those words.

- **"The primary failed"** means one of two things:
  - a `StoreUnavailableError` whose `store === this.primary`;
  - a `ConflictError` from the `mergeIntoStore` over the primary.

  Every other error propagates. That includes a `StoreUnavailableError` from
  the archive store (e.g. GitHub unreadable while the live object is empty)
  and `SyncUpstreamError` from `build`, which both behave this way today.
- **`publish(a)`:**
  1. Start `liveMs`. Run `mergeIntoStore` over `primary` with
     `readBase = ref => this.#newerBase(ref, a.log)`. Use
     `conflictMessage: "live publish: 3 version conflicts"`, because today's
     response carries that exact text as `live.error`. On every conflict, log
     `live version conflict (attempt n/max) — re-reading and re-merging`.
  2. On success, stop `liveMs` and set `publishedAtMs = Date.now()`. Run
     `#archive` (below), timed as `archiveMs`. Return
     `live: { published: true, version: <token> }`, `archive`, and
     `commitSha: archive.committed ? archive.commitSha : null`.
  3. If the primary failed: stop `liveMs` and log `warn`
     `live publish failed, publishing via GitHub instead: <message>`. Return
     `{ ...(await this.fallback.publish(a)), live: { published: false, error: <message> }, liveMs }`.
- **`republish(a)`:** the same, with
  `build = base => base ? { snapshot: a.mutate(base), stale: [] } : null`. On
  every conflict, log
  `live version conflict (attempt n/max) — re-reading and re-applying`.
  - Nothing to republish: return `{ republished: false }`.
  - Written: archive, then return
    `{ republished: true, live: { published: true, version }, archive }`.
  - Primary failed: log `warn`
    `${a.label}: live republish failed, republishing via GitHub instead: <message>`,
    then `r = await this.fallback.republish(a)`. Return
    `r.republished ? { ...r, live: { published: false, error } } : { republished: false }`.
    No `live` key when there was nothing to republish, as today.
- **`#archive({ ref, snapshot, message, log })`** never throws. It runs
  `withConflictRetry` around
  `archive.write(ref, snapshot, { token: null, message })`.
  - On a conflict with `n < max`: log
    `archive commit conflict (attempt n/max) — retrying with the live snapshot`,
    then `primary.read(ref)`. If that returns a snapshot, use it for the next
    attempt. If it throws, log `warn`
    `could not re-read the live snapshot for the archive retry: <message>`.
  - On success, return `{ committed: true, commitSha }`.
  - On **any** error, including `ConflictError`: log `error`
    `archive commit failed (live copy is published): <message>` and return
    `{ committed: false, error: <message> }`.
- **`#newerBase(ref, log)`** is the stale-object rule, unchanged. It reads
  `primary` and `archive` together (`Promise.allSettled`):
  - primary rejected: rethrow (a `StoreUnavailableError` from the primary, so
    the delivery falls back);
  - archive rejected: if the primary's snapshot is `null`, rethrow (it
    propagates). Otherwise log `warn`
    `could not read GitHub's copy to compare with the live object's: <message>`
    and use the primary's snapshot;
  - both read: take `newer(primary.snapshot, archive.snapshot)` from
    `sync/domain/snapshotStamp.mjs` (ties go to the primary). When the
    archive's copy wins over a non-null primary one, log `info`
    `GitHub's snapshot is newer than the live object's — merging into GitHub's`.

  It returns `{ snapshot: <chosen>, token: <primary's token> }`.

**`SnapshotPublisher({ liveFirst, githubOnly, liveAvailability })`**
implements `SnapshotPublishing`:

```js
liveActive(config) { return this.liveAvailability.enabled && config.livePushOn; }
#select(config)    { return this.liveActive(config) ? this.liveFirst : this.githubOnly; }
publish({ config, ...a })   { return this.#select(config).publish(a); }
republish({ config, ...a }) { return this.#select(config).republish(a); }
```

The republish results must equal today's `setLiveOverride` in each case. The
integration tests pin these shapes:

| Case | Returns |
|---|---|
| Live active, live write succeeds | `{ republished: true, live: { published: true, version }, archive }` |
| Live active, newer base is `null` | `{ republished: false }`, with **no** `live` key |
| Live failed, GitHub write succeeds | `{ republished: true, live: { published: false, error } }` |
| Live failed, GitHub holds nothing | `{ republished: false }`, with **no** `live` key |
| Live not active, GitHub write succeeds | `{ republished: true }` |
| Live not active, GitHub holds nothing | `{ republished: false }` |
| Three GitHub conflicts | throws `ConflictError` (B1) |

Adding a third delivery target means a new store adapter, a new
`SnapshotDelivery` (or a reuse of `LiveFirstDelivery` with different stores),
and one line in `SnapshotPublisher.#select` and in `app.mjs`. `SyncService`,
`DayVisibilityService` and both existing deliveries stay untouched.

### 4.7 Errors

`AppError` gains a `code` string and an optional `extra` object. The pattern
for every class: call `super(...)`, then set `this.code` (and `this.extra`
where listed). Subclasses of subclasses overwrite `code` after their own
`super`.

| Class (in `src/shared/errors.mjs`) | `statusCode` | `code` | `extra` |
|---|---|---|---|
| `AppError` (base) | 500 | `internal_error` | — |
| `ValidationError` | 400 | `validation_error` | — |
| `UnauthorizedError` | 401 | `unauthorized` | — |
| `UnknownScoresheetTypeError` | 400 | `unknown_scoresheet_type` | — |
| `UnknownSyncDayError` | 400 | `unknown_day` | — |
| `SyncUpstreamError` | 502 | `upstream_failure` | — |
| `SyncConfigUnavailableError` | 503 | `config_unavailable` | — |
| `ForbiddenError` | 403 | `forbidden` | — |
| `NotFoundError` | 404 | `not_found` | — |
| `UnknownEventError` | 404 | `unknown_event` | — |
| `AttendanceLayoutError` | 409 | `attendance_layout` | — |
| `UpstreamError` | 502 | `upstream_failure` | — |
| `ServiceBusyError` | 503 | `service_busy` | — |
| `ScoreEntryOffError` | 403 | `score_entry_off` | — |
| `ScheduleLayoutError` | 422 | `schedule_layout` | — |
| `ScoreConflictError` | 409 | `score_conflict` | `{ current }` (it also keeps `this.current`) |
| `ConflictError` (new, Phase 2; used from Phase 3) | 409 | `conflict` | — (also sets `this.status = 409`) |
| `NotAcceptableError` (new, Phase 5) | 406 | `not_acceptable` | — |
| `MethodNotAllowedError` (new, Phase 5) | 405 | `method_not_allowed` | — (carries `this.allow`, e.g. `["GET", "HEAD", "PUT"]`, for the `Allow` header) |
| `UnsupportedMediaTypeError` (new, Phase 5) | 415 | `unsupported_media_type` | — |
| `UnknownFacilityError` (new, Phase 2; extends `ValidationError`) | 400 (404 on `/v3`, Phase 5) | `unknown_facility` | — (thrown by `SyncService`, `AttendanceService` and `ScoreService`, each with its message of today, so the legacy and `/v1` bodies keep their text. It replaces the two identical `unknownFacility` helpers. On `/v3` the facility is always a path segment, hence `404`) |
| `ConfigError` (new, Phase 2; startup only, never an HTTP response) | — | `config_error` | — |

```js
// src/server/errorHandler.mjs — registered last, after every route
export function errorHandler(logger) {
    return (err, req, res, next) => {
        if (res.headersSent) return next(err);   // streaming: the route already wrote its own error line
        const isApp = err instanceof AppError;
        const passthrough = Number.isInteger(err?.statusCode) && err.statusCode >= 400 && err.statusCode < 600;
        const status = isApp ? err.statusCode : passthrough ? err.statusCode : 500;
        const code = isApp ? err.code : status < 500 ? "bad_request" : "internal_error";
        if (status >= 500) logger.error(`${req.method} ${req.originalUrl} failed (${status})`, err);
        else logger.warn(`${req.method} ${req.originalUrl} failed (${status}): ${err?.message}`);   // Logger.warn takes one argument
        if (status === 401) res.set("WWW-Authenticate", 'Bearer realm="sage-tools-api"');   // B9
        res.status(status).json({ error: err?.message ?? String(err), code, ...(isApp ? err.extra : undefined) });
    };
}
```

- `code` comes only from an `AppError`. A Node system error's own `code`
  (`ECONNRESET` and the like) must never reach the body.
- `passthrough` covers body-parser's errors (`express.json()` on a malformed
  body sets `statusCode: 400`): B7.
- `error` is still `err.message`, including for `500`s, because Apps Script
  shows it to operators.
- The request body is never logged, since login carries a password.

**Phase 5 extends the handler** for the REST conventions (§4.11). The final
shape is:

```js
// On /v3, a day, event or facility is always named in the path, so not finding it is 404.
// The same errors keep their 2.x status everywhere else, so /v1 and the legacy routes do not change.
const V3_STATUS = { unknown_day: 404, unknown_facility: 404 };

export function errorHandler(logger) {
    return (err, req, res, next) => {
        if (res.headersSent) return next(err);
        const isApp = err instanceof AppError;
        const isV3 = req.path === "/v3" || req.path.startsWith("/v3/");
        const passthrough = Number.isInteger(err?.statusCode) && err.statusCode >= 400 && err.statusCode < 600;
        const code = isApp ? err.code : (passthrough && err.statusCode < 500) ? "bad_request" : "internal_error";
        const status = (isV3 && V3_STATUS[code]) || (isApp ? err.statusCode : passthrough ? err.statusCode : 500);
        const message = err?.message ?? String(err);
        if (status >= 500) logger.error(`${req.method} ${req.originalUrl} failed (${status})`, err);
        else logger.warn(`${req.method} ${req.originalUrl} failed (${status}): ${message}`);

        if (status === 401) res.set("WWW-Authenticate", 'Bearer realm="sage-tools-api"');
        if (status === 405 && err.allow) res.set("Allow", err.allow.join(", "));
        const extra = isApp ? err.extra : undefined;
        if (!isV3) return res.status(status).json({ error: message, code, ...extra });
        res.status(status).type("application/problem+json").send(JSON.stringify({
            type: "about:blank", title: http.STATUS_CODES[status], status, detail: message,
            instance: req.originalUrl.split("?")[0], code, error: message, ...extra,
        }));
    };
}
```

- `WWW-Authenticate` is set on every `401`, legacy routes included. It is an
  added header and changes no body.
- `components.schemas.Problem` in `openapiSpec.mjs` describes the `/v3` body:
  `type`, `title`, `status`, `detail`, `instance`, `code` and `error`
  required, plus `additionalProperties: true` for `current`.

```js
// src/server/asyncHandler.mjs
export const asyncHandler = fn => (req, res, next) => Promise.resolve(fn(req, res, next)).catch(next);
```

Express 4 does not forward a rejected promise by itself. Every async route
handler is wrapped in `asyncHandler`.

### 4.8 Configuration

```js
// src/config/loadConfig.mjs
/** @returns {Readonly<AppConfig>}  @throws {ConfigError} listing every problem at once */
export function loadConfig(env = process.env) {}
/** @returns {string|null} one warning line naming unset secrets, or null */
export function unsetWarning(config) {}
```

Rules:

- A value is read as `String(v).trim()`. `undefined` and `""` both mean
  **unset**.
- An integer must match `/^\d+$/` and fall within its bounds. Anything else
  (`abc`, `1.5`, `-5`, out of range) is a problem, and every problem is
  collected. The `ConfigError` message is `Invalid environment:` followed by
  one `  - NAME: "<value>" <why>` line per problem.
- An unset value becomes the default in the table, or `undefined` so the
  receiving constructor's own default parameter applies. **Never `null`**: a
  default parameter does not apply to `null`.
- The result is frozen, nested objects included.

| Variable | Rule | Property | Unset |
|---|---|---|---|
| `PORT` | integer 1–65535 | `port` | `8080` |
| `SCORESHEET_CONCURRENCY` | integer ≥ 1 | `scoresheetConcurrency` | `8` |
| `CORS_ORIGIN` | string | `corsOrigin` | `"*"` |
| `GITHUB_OWNER`, `GITHUB_REPO` | string | `github.owner`, `github.repo` | `undefined`, warned |
| `GITHUB_BRANCH` | string | `github.branch` | `"main"` |
| `GITHUB_TOKEN` | secret | `github.token` | `undefined`, warned |
| `SYNC_SHARED_SECRET` | secret | `sync.sharedSecret` | `undefined`, warned |
| `SYNC_CONFIG_TTL_MS` | integer ≥ 0 | `sync.configTtlMs` | `undefined` (store default 60000) |
| `SHEETS_FETCH_TIMEOUT_MS` | integer ≥ 0; **`0` disables the timeout** | `sync.sheetsFetchTimeoutMs` | `undefined` (fetchers' default 8000) |
| `GOOGLE_SHEETS_API_KEY` | secret | `sheets.apiKey` | `undefined`, warned |
| `GOOGLE_ACCESS_TOKEN` | secret, local development only | `sheets.accessTokenOverride` | `undefined`, not warned (normal on Cloud Run) |
| `AUTH_PASSWORD_HASH` | secret | `auth.passwordHash` | `undefined`, warned |
| `AUTH_TOKEN_SECRET` | secret | `auth.tokenSecret` | `undefined`, warned |
| `AUTH_TOKEN_TTL_MS` | integer ≥ 1 | `auth.tokenTtlMs` | `undefined` (12 hours) |
| `LIVE_PUSH_URL` | an `http:` or `https:` URL (`new URL()` parses it and the protocol matches) | `live.url` | `undefined` (live push off, not warned) |
| `LIVE_PUSH_SECRET` | secret | `live.secret` | `undefined` |
| `LIVE_PUSH_TIMEOUT_MS` | integer ≥ 1 | `live.timeoutMs` | `undefined` (4000) |

`unsetWarning(config)` returns one line, or `null` when there is nothing to
report. The line names every unset "warned" variable. It also says
`live push is off: LIVE_PUSH_SECRET is not set` when only the URL is set, and
the reverse. `.env.example` documents the same rules: update its comments in
Phase 2.

### 4.9 Auth middleware

```js
// src/auth/middleware.mjs
/**
 * @param {{ authService: import("../shared/ports.mjs").TokenVerifier, syncSharedSecret?: string }} deps
 * @returns {{
 *   requireOperator: import("express").RequestHandler,
 *   requireOperatorOrSyncSecret: import("express").RequestHandler,
 *   requireOperatorOr: (kind: "desk" | "scorer") => import("express").RequestHandler,
 * }}
 */
export function createAuthMiddleware({ authService, syncSharedSecret }) {}
```

- `bearerToken(req)`: `Authorization: Bearer <token>`, or `null`.
- `requireOperator`: `authService.verify(token)` passes and sets
  `req.actor = { kind: "operator" }`. Otherwise it calls
  `next(new UnauthorizedError())`. It replaces sync's `requireAuthToken` and
  attendance's and scores' `requireOperator`.
- `requireOperatorOrSyncSecret`: passes on a valid operator token (setting
  `req.actor` as above), or when `safeEqual(req.get("X-Sync-Secret"), syncSharedSecret)`
  holds. A missing or empty configured secret never matches, not even an
  empty header. It replaces `requireSyncSecretOrAuthToken`.
- `requireOperatorOr("desk")`: an operator token, or
  `authService.verifyDeskToken(token)` returning `{ day }`, which sets
  `req.actor = { kind: "desk", day }`. It replaces `requireOperatorOrDesk`.
- `requireOperatorOr("scorer")`: the same with `verifyScorerToken`, and
  `req.actor = { kind: "scorer", day }`. It replaces `requireOperatorOrScorer`.
- The scoped kinds come from one map at the top of the file,
  `const SCOPED_VERIFIERS = { desk: "verifyDeskToken", scorer: "verifyScorerToken" }`.
  `requireOperatorOr(kind)` throws at startup (when the routes are built) for
  an unknown kind. A new token scope is one entry here.

`safeEqual(a, b)` (`src/shared/safeEqual.mjs`) returns `false` unless both are
non-empty strings. It then compares the SHA-256 digests of both with
`crypto.timingSafeEqual`, so the length does not leak.

### 4.10 `createApp`

```js
// src/app.mjs
/**
 * Builds every object the service needs, in one place. index.mjs and the
 * tests' buildApp both call it, so the wiring is never copied.
 * @param {Readonly<AppConfig>} config  from loadConfig
 * @param {{ logger: Logger, version?: string|null, templatesDir?: string,
 *           getScoresheetService?: () => Promise<ScoresheetService> }} opts
 *   getScoresheetService defaults to the lazy import of src/scoresheets/index.mjs
 *   (memoized, retried after a failure), exactly as index.mjs does today.
 * @returns {{ server, syncService, syncConfigStore, authService, attendanceService,
 *             scoreService, githubPublisher, livePublisher,
 *             // from Phase 4:
 *             snapshotPublisher, dayVisibilityService, syncSettingsService }}
 */
export function createApp(config, opts) {}

/**
 * Builds every router and says where it is mounted. createApp calls it with
 * the real services; the tests' startApp calls it with stub services.
 * @param {{ services: object, auth: ReturnType<typeof createAuthMiddleware>,
 *           getScoresheetService: Function, logger: Logger }} deps
 * @returns {{ path: string, router: import("express").Router }[]}
 *   In mount order: /scoresheets, /sync, /auth, /v1 (attendance and scores,
 *   frozen), then (Phase 5) /v3 for sync, auth, scoresheets, attendance and
 *   scores.
 */
export function createRouters(deps) {}
```

- `Server` is constructed as
  `new Server({ routers, peekConfig: () => syncConfigStore.peek(), logger, port, corsOrigin, version })`.
  It mounts each router at its `path` in order and imports no feature module
  (R10).
- `SyncService` receives `fetchers: { sheets: <SheetsCsvFetcher>, csv: <GvizCsvFetcher> }`
  and picks `this.fetchers[method]`. It no longer takes `sheetsApiFetcher` and
  `gvizFetcher`.
- From Phase 4 the publishing graph is built like this:

  ```js
  const githubStore = new GitHubSnapshotStore(githubPublisher);
  const liveStore = new LiveSnapshotStore(livePublisher);
  const githubOnly = new GitHubOnlyDelivery({ store: githubStore });
  const liveFirst = new LiveFirstDelivery({ primary: liveStore, archive: githubStore, fallback: githubOnly });
  const snapshotPublisher = new SnapshotPublisher({ liveFirst, githubOnly, liveAvailability: livePublisher });
  ```

- It reads `events.seed.json` relative to itself:
  `new URL("./registry/events.seed.json", import.meta.url)`.
- It keeps today's logger children (`sync`, `sync/github`, `sync/config`,
  `sync/live`, `sync/fetch`, `sync/fetch-csv`, `auth`, `attendance`, `scores`,
  `http`).
- It keeps the two `SheetsClient` instances: 5 s for attendance, 60 s for
  score entry.
- The scoresheet pipeline is still imported lazily, inside
  `getScoresheetService`. **Never import `src/scoresheets/index.mjs` at the
  top of any file.**

### 4.11 The `/v3` API and its REST conventions

`/v3` is the API's one REST surface, released as 3.0.0. Phase 5 builds it:

- a resource-named twin of every route a client calls today, legacy and
  `/v1` alike;
- three `GET`s;
- every route following the conventions below.

Every older URL is frozen exactly as it is. That means `/sync/...`,
`/scoresheets/...` and `/auth/login`, and also `/v1/...`: attendance's and
score entry's routes from 2.6.0 and 2.8.0. None of them is held to these
conventions, and none is ever removed. `/ping` stays outside any version.

**Why `/v3` when there is no `/v2`.** The URL version names the API's major
version, and the API's major version is the package's major version. 3.0.0 is
the release that introduces this surface, so it is `/v3`. `/v1` was the
partial REST surface of the 2.x releases. No `/v2` ever existed, and nothing
answers there.

#### Conventions

The rules follow RFC 9110 (HTTP semantics), RFC 9457 (problem details) and
RFC 9745 (deprecation). The naming rules are common to the Microsoft, Google
and Zalando API guidelines. Every `/v3` route follows them. The few
departures are listed under [Exceptions](#exceptions), with their reasons.

**URLs**

- **N1. Nouns, never verbs.** A path names a resource. An action becomes a
  noun for its result: `syncs`, `reconciliations`, `sessions`. There is no
  `generate`, `login`, `run` or `set` in a `/v3` path.
- **N2. Plural collections.** A path segment directly in front of an
  identifier is a plural noun: `days/{day}`, `facilities/{facility}`,
  `matches/{matchNumber}`, `events/{event}`.
- **N3. Singular sub-resources** for one-per-parent things: a day's
  `visibility`, a match's `score`, an event's `score-entry`, the global
  `settings/live-push`, the `diagnostics/sync` report.
- **N4. Lowercase kebab-case** literal segments (`desk-links`,
  `score-entry`). Path parameters are camelCase (`{matchNumber}`,
  `{personKey}`). No trailing slash and no file extension.
- **N5. Shallow nesting.** Nest only for real containment, and start from the
  shortest unique parent. Day keys are unique across events, so a day sits at
  `/v3/days/{day}`, not under its event.
- **N6. Query strings filter or shape a `GET`.** A `POST` or `PUT` takes its
  input in a JSON body.

**Methods**

- **M1. `GET`** reads, and is safe and idempotent. Express answers `HEAD` for
  every `GET` automatically.
- **M2. `PUT`** replaces a resource's state with the body, and is idempotent:
  the same body twice gives the same state. Every `PUT` resource also has a
  `GET` with the same representation, unless an exception says where it is
  read instead.
- **M3. `POST`** to a collection creates something, or runs an action whose
  result is a report. Not idempotent.
- **M4.** `PATCH` and `DELETE` are allowed by CORS (B3) but no route uses them
  yet.
- **M5. Method not allowed.** A request to a known `/v3` path with a method
  it does not support answers `405` with an `Allow` header. It never answers
  `404`.

**Bodies and representations**

- **B-1. JSON** with camelCase member names. A request with a body must send
  `Content-Type: application/json`, or the API answers `415`.
- **B-2. No envelopes.** A response is the resource or the report itself,
  never wrapped in `{ data: … }`.
- **B-3. The `PUT` body and the `GET` response use the same member names.** A
  `PUT` response is that representation, plus side-effect members where the
  write did more than store a value (`changed`, `republished`, `live`,
  `archive`).
- **B-4. Enumerations are lowercase strings** (`"console"`, `"links"`,
  `"sheets"`, `"csv"`, `"auto"`).
- **B-5. Timestamps are ISO 8601 UTC strings**, except where an exception
  says otherwise.
- **B-6. Absent means unknown, `null` means "none".** A response never omits
  a member that its schema lists as always present.

**Status codes**

| Status | When |
|---|---|
| `200 OK` | a successful `GET`, `PUT`, or action `POST` that returns a report |
| `201 Created` | a `POST` that creates something: a session, a desk link, a scorer link. These are issued credentials with no URL of their own, so there is no `Location` header (see the exceptions) |
| `400 Bad Request` | malformed JSON, or a body or query value that fails validation |
| `401 Unauthorized` | no credentials, or credentials that are invalid or expired. Always with `WWW-Authenticate: Bearer realm="sage-tools-api"` |
| `403 Forbidden` | valid credentials that are not allowed this action (wrong day, feature switched off) |
| `404 Not Found` | an unknown path, or an unknown resource named **in the path** (`unknown_day`, `unknown_event`, `unknown_facility`, or a match number not in `SCHEDULE`) |
| `405 Method Not Allowed` | a known path with an unsupported method, with `Allow` |
| `406 Not Acceptable` | an `Accept` header the route cannot satisfy |
| `409 Conflict` | the target's current state conflicts with the request: conflicting writes (`conflict`), a score that changed (`score_conflict`), an `ATTENDANCE` tab in the wrong shape (`attendance_layout`) |
| `415 Unsupported Media Type` | a body that is not `application/json` (or `multipart/form-data` for scoresheets) |
| `422 Unprocessable Content` | a well-formed request the workbook's state cannot take (`schedule_layout`) |
| `500` / `502` / `503` | our failure / Google's or GitHub's failure / temporarily unavailable |

**Errors: RFC 9457 problem details**

Every `/v3` error response has `Content-Type: application/problem+json` and
this body:

```json
{
  "type": "about:blank",
  "title": "Conflict",
  "status": 409,
  "detail": "The sheet changed since this match was opened",
  "instance": "/v3/days/pd-day1/facilities/Main/matches/12/score",
  "code": "score_conflict",
  "error": "The sheet changed since this match was opened",
  "current": { "teamCode1": "A", "teamCode2": "B", "team1Score": 11, "team2Score": 9 }
}
```

- `type` is `about:blank`, so `title` is the status's standard reason phrase,
  `http.STATUS_CODES[status]`.
- `detail` is the error's message. `instance` is `req.originalUrl` without
  its query string.
- `code` is the stable machine-readable code from §4.7.
- `error` repeats `detail`. It is an extension member kept so that the desk
  pages, the scorer page and Control Center, which all read `body.error`,
  keep working unchanged.
- An error's `extra` members (`current`) follow.

Legacy routes keep the plain `{ error, code }` body with
`Content-Type: application/json` (§4.7), because Apps Script prints that body
to operators.

**Versioning and deprecation**

- **The version is in the path, and it equals the package's major version.**
  - Adding a route, an optional request member or a response member is not
    breaking: it is a minor release, and stays on `/v3`.
  - Breaking changes are renaming or removing a route or member, changing a
    type, or changing the status code for the same outcome. One is allowed
    only as a new major release with a new URL surface (4.0.0 and `/v4`),
    which serves alongside `/v3`.
  - A major release never ships without its new URL surface, so the two
    numbers stay equal.
- Every legacy or `/v1` route that has a `/v3` twin answers with two extra
  headers:
  - `Deprecation: @<unix seconds>` (RFC 9745). The value is
    `LEGACY_DEPRECATED_AT`, a constant set to the Unix time of the 3.0.0
    release commit.
  - `Link: <{the /v3 path, with this request's parameters filled in}>; rel="successor-version"`.

  No `Sunset` header is sent, because these routes never go away. OpenAPI
  marks them `deprecated: true` as well.

**Documentation**

- Every `/v3` operation has an `@openapi` block with an `operationId` in
  camelCase verb-noun form (`getDayVisibility`, `updateDayVisibility`,
  `createSession`).
- Each block documents every parameter and every status the operation can
  return. The error responses use `$ref: '#/components/schemas/Problem'`.

#### Routes

Every `/v3` route is new in Phase 5. "Twin of" is the frozen route it
replaces for clients.

| Method and path | What it does | Auth | Request | Success | Twin of |
|---|---|---|---|---|---|
| `POST /v3/days/{day}/syncs` | sync every facility of the day | `X-Sync-Secret` or operator | optional JSON `{ "method"?: "sheets" \| "csv" }`; no body = `sheets` | `200`, the sync report (same body as the twin) | `POST /sync/:day` |
| `POST /v3/days/{day}/facilities/{facility}/syncs` | sync one facility (what Apps Script calls on an edit) | `X-Sync-Secret` or operator | optional JSON `{ "method"?: "sheets" \| "csv", "editedAt"?: string }`; `editedAt` is the ISO 8601 time of the edit behind the sync | `200`, the sync report (same body as the twin) | `POST /sync/:day?facility=` |
| `GET /v3/days/{day}/visibility` | read the day's go-live setting | operator | — | `200 { day, label, isLive }` | — |
| `PUT /v3/days/{day}/visibility` | set it, and republish | operator | `{ "isLive": true \| false \| "auto" }` | `200 { day, label, isLive, republished, live?, archive? }` (same as the twin) | `POST /sync/:day/live` |
| `GET /v3/settings/live-push` | read the live-push switch | operator | — | `200 { enabled, available }` | — |
| `PUT /v3/settings/live-push` | set it | operator | `{ "enabled": boolean }` | `200 { enabled, changed, available }` (same as the twin) | `POST /sync/live-push` |
| `GET /v3/diagnostics/sync` | sync config and live-push diagnostics | `X-Sync-Secret` or operator | — | `200`, same body as the twin | `GET /sync/config` |
| `POST /v3/scoresheets` | generate a scoresheet PDF | none | `multipart/form-data`, as the twin; `Accept: application/pdf` (or absent, or `*/*`) or `application/x-ndjson` | `200`, a PDF or NDJSON lines, same as the twins | `POST /scoresheets/generate`, `…/generate/stream` |
| `POST /v3/sessions` | sign in | none | `{ "username": string, "password": string }` | **`201`** `{ token, expiresAt }` | `POST /auth/login` (`200`) |
| `GET /v3/events/{event}/score-entry` | read the event's score-entry mode | operator | — | `200 { event, scoreEntry }`; `scoreEntry` is `"console"`, `"links"` or `null` (off) | — |
| `PUT /v3/events/{event}/score-entry` | set it | operator | `{ "scoreEntry": "console" \| "links" }` | `200 { event, scoreEntry, changed }` | `PUT /v1/events/:event/score-entry` (body `{ mode }`) |
| `PUT /v3/days/{day}/facilities/{facility}/people/{personKey}/attendance` | mark one person present or not | operator or desk | `{ "present": boolean }` | `200`, same body as the twin | `PUT /v1/days/:day/facilities/:facility/attendance/:key` |
| `POST /v3/days/{day}/attendance/desk-links` | issue a desk link | operator | — | `201`, same body as the twin | `POST /v1/days/:day/attendance/desk-links` |
| `POST /v3/days/{day}/attendance/reconciliations` | update the `ATTENDANCE` roster | operator | optional JSON `{ "facility"?: string }`; no body = every facility | `200`, same body as the twin | `POST /v1/days/:day/attendance/reconciliations` (`?facility=`) |
| `PUT /v3/days/{day}/facilities/{facility}/matches/{matchNumber}/score` | enter, correct or clear a score | operator or scorer | as the twin | `200`, same body as the twin | `PUT /v1/days/:day/facilities/:facility/matches/:matchNumber/score` |
| `POST /v3/days/{day}/scores/scorer-links` | issue a scorer link | operator | — | `201`, same body as the twin | `POST /v1/days/:day/scores/scorer-links` |

That is sixteen routes on thirteen paths.

- **The sync routes.** A facility is a sub-resource of its day, so a
  one-facility sync is `POST …/facilities/{facility}/syncs`, and an unknown
  facility there is a `404` (`unknown_facility`), like every other name in a
  `/v3` path.
  - The edit time travels as the body member `editedAt` (ISO 8601, B-5)
    instead of the `X-Edit-At` header: custom `X-` headers are deprecated
    by RFC 6648. The controller converts it with `Date.parse` and applies
    the same `parseEditAt` window. An unparseable `editedAt` is a `400`;
    one outside the window is ignored, as today.
  - The day-level route takes no `editedAt`: today an edit time only counts
    on a facility-scoped sync.
  - Both reject a `method` other than `"sheets"` or `"csv"` with `400`;
    the legacy route silently uses `sheets`.
- **Scoresheet negotiation.** `POST /v3/scoresheets` negotiates with
  `req.accepts(["application/pdf", "application/x-ndjson"])` **before**
  multer parses the upload. If the result is `false`, it throws
  `NotAcceptableError("Accept must be application/pdf or application/x-ndjson")`.
- **The attendance path.** A person's attendance is a singular sub-resource
  of that person (N3): `…/people/{personKey}/attendance`. This replaces
  `/v1`'s `…/attendance/{key}`, whose segment before the identifier was not a
  plural (N2).
- **How the `/v3` twins of attendance and score entry differ from `/v1`**
  (B8): problem-details errors; `404` for an unknown day, event or facility
  named in the path (`/v1` answers `400`); `405`; `415`; the attendance path;
  `{ scoreEntry }`; reconciliations' body. `/v1` itself is unchanged apart
  from the B9 headers.

#### Exceptions

These are documented here and in each route's OpenAPI description.
`rest-conventions.test.mjs` (§5.2) carries the same list as its allow-list.

| Route | Rule it departs from | Why |
|---|---|---|
| `PUT …/people/{personKey}/attendance` and `PUT …/matches/{matchNumber}/score` | M2 (no `GET`) | the state is read from the published day snapshot and the `ATTENDANCE` tab's published CSV, which pages already poll. A Sheets read per `GET` would spend Google quota for no reader |
| `PUT …/matches/{matchNumber}/score` | concurrency through `If-Match`/`412` | the precondition is the values a person saw in two cells, not a version the API issues, so there is no `ETag` to give. `expected` travels in the body, and the `409` carries `current` so the dialog can show both |
| `201` without `Location` (sessions, desk links, scorer links) | RFC 9110 §15.3.2's `Location` for a created resource | the result is a signed credential, not a stored resource, so there is no URL to point at |
| `expiresAt` in the three token responses | B-5 (ISO 8601) | it is epoch milliseconds in every surface. Control Center, the desk pages and the scorer page compare it with `Date.now()`, and their token code is not otherwise touched. Every OpenAPI schema says `integer, format: int64, epoch milliseconds` |
| `isLive: true \| false \| "auto"` | B-4 (an enum, not a mixed type) | it is the same field, with the same values, as `events.json` and the published snapshot every page reads. One name and one shape across three repos beats a translation layer |
| `422` for `schedule_layout` | — (stated so nobody "fixes" it to `409`) | the request is fine; the workbook's content cannot take it |

### 4.12 SOLID: how the target meets each principle

This is the review's checklist. A phase is not done while a row that names it
fails. "Checked by" says how a reviewer, or a test, can tell.

**S — Single responsibility: one reason to change per module.**

| Module | Its one reason to change | Checked by |
|---|---|---|
| `SyncService` | how one day is synced (which facilities, how results merge, what the response reports) | no GitHub, Worker, archive, fallback or HTTP code in it (Phase 4 acceptance) |
| `DayVisibilityService`, `SyncSettingsService` | the Live/Hide override; the live-push switch and its diagnostics | one small file each |
| `syncController` | the HTTP shape of the sync routes | no service logic; every branch is request parsing or response writing |
| `*/routes.mjs` | which URL maps to which handler, with which auth, and its OpenAPI text | no `try/catch`, no auth logic, no validation beyond calling a domain parser |
| `GitHubOnlyDelivery`, `LiveFirstDelivery`, `SnapshotPublisher` | one delivery path each; the choice between them | §4.6 |
| `GitHubSnapshotStore`, `LiveSnapshotStore` | the shape of one client | each under 80 lines |
| `mergeIntoStore`, `withConflictRetry` | the read-merge-write loop; the retry policy | written once (§4.5) |
| `SyncConfigStore` | loading, caching, falling back and committing `events.json` | validation and the switch edits are pure modules in `registry/domain/` |
| `errorHandler`, `cors`, `auth/middleware`, `loadConfig`, `createApp` | one cross-cutting concern each | §4.7–§4.10 |

Accepted as they are, and out of scope (§1):

- **`AttendanceService`** (mark, reconcile, desk links) and **`ScoreService`**
  (submit, scorer link, mode switch) each group the use cases of one small
  feature behind one registry and one Sheets port.
- **`AuthService`** issues and verifies three kinds of token that share one
  signing scheme, so a change to the scheme is its one reason to change.

Split any of them when one use case grows past the rest of its file.

**O — Open for extension, closed for modification.**

| To add… | You write | You do not touch |
|---|---|---|
| a delivery target | a store adapter, a delivery (or reuse `LiveFirstDelivery`), one line in `SnapshotPublisher.#select` and `app.mjs` | `SyncService`, `DayVisibilityService`, the existing deliveries |
| a CSV fetch method | a `FacilityFetcher` and one entry in the `fetchers` map in `app.mjs` | `SyncService` (it does `this.fetchers[method]`) |
| an error type | an `AppError` subclass with `statusCode` and `code` | `errorHandler` |
| a feature with routes | its routes factory, and one entry in `createRouters` | `Server` |
| a registry switch | a pure edit in `registry/domain/registryEdits.mjs`, a one-line `SyncConfigStore` method | `#update`, the load and cache code |
| a token scope | an `AuthService` issue/verify pair and one entry in the middleware's scope map | the other middleware |
| a reaction after a sync | a `SyncedHook` composed in `app.mjs` | `SyncService` |

**L — Liskov substitution: implementations of a port are interchangeable.**

- **The two stores.** Both honour one `SnapshotStore` contract, failure
  contract included (§4.4). `test/helpers/snapshotStoreContract.mjs` holds the
  shared cases, and both store test files run them (§5.2). The deliveries
  take stores by role, never by type.
- **The two deliveries.** Both return the `PublishResult` and
  `RepublishResult` shapes. When the primary fails, `LiveFirstDelivery`
  returns exactly its fallback's result plus `live` (and `liveMs`).
- **The two fetchers.** Both resolve
  `{ name, matchesCsv, standingsCsv, rosterCsv? }`. Their existing unit tests
  pin this.
- **The test fakes** (`FakePublisher`, `FakeLivePublisher`, `fakeSheets`,
  `FakeWorld`) implement the same ports as the clients. A test that passes
  with a fake but fails against the real client's stubbed `fetch` points to
  the fake.

**I — Interface segregation: each consumer depends only on what it calls.**

The port table in §4.4 gives each consumer its own narrow port, even where one
class implements several:

- `SyncConfigStore` implements `RegistryReader`, `DayVisibilitySwitch`,
  `LivePushSwitch` and `ScoreEntrySwitch`;
- `AuthService` implements `TokenVerifier`, `Authenticator`, `DeskTokenIssuer`
  and `ScorerTokenIssuer`;
- `SheetsClient` implements `AttendanceSheets` and `ScoreSheets`;
- `SyncService` implements `DayPublisher` for score entry.

`// @ts-check` in the new modules flags a call outside the declared port. For
`AttendanceService` and `ScoreService`, Phase 4 adds the port types to their
constructor JSDoc as a comment-only change, without `// @ts-check`.

**D — Dependency inversion: policy depends on abstractions, and the
composition root picks the details.**

- Services, deliveries and `mergeIntoStore` import no adapter (R4, R5). They
  are typed against ports and receive instances.
- Stores depend on the client ports (`JsonDocumentStore`, `LiveChannel`), not
  on `GitHubPublisher` or `LivePublisher`.
- `Server` depends on a list of `{ path, router }` and a `peekConfig()`
  function, not on any feature (R10).
- Configuration reaches objects through `createApp`. Nothing else reads the
  environment (R9).
- Only `app.mjs` (and the lazy `scoresheets/index.mjs`) chooses concrete
  classes (R8).
- Allowed concrete dependencies, because they are stable and free of side
  effects: `shared/errors.mjs`, `shared/conflictRetry.mjs`, `shared/csv.mjs`
  and `domain/` functions.

---

## 5. Tests

### 5.1 The suite that already exists

`test/` (957 tests at 2.8.0) is described in the README's **Testing**
section. Its helpers are what new tests are written with: `buildApp` and
`startApp`, `createFakeWorld`, the builders and the fakes (`FakePublisher`,
`FakeLivePublisher`, `fakeSheets`). The guards
`every-module-tested.test.mjs` and `every-route-tested.test.mjs` fail when a
new module or route has no test.

### 5.2 Tests for the modules this spec creates

Write each module's tests **first, in the phase that creates it**. Watch them
fail because the module is missing, then implement. Each lives at
`test/unit/<path under src>/<Module>.test.mjs`.

| Module | Cases |
|---|---|
| `shared/csv.mjs` (Phase 1) | the `parseCsv` cases moved out of `facilityCompletion.test.mjs`, unchanged |
| `shared/safeEqual.mjs` (2) | equal strings → true; different strings of equal and of unequal length → false; `undefined`, `null`, a number, `""` on either side → false; never throws |
| `config/loadConfig.mjs` (2) | every row of §4.8; `""` and `"  "` are unset; `abc`, `1.5`, `-5`, `0` (where ≥ 1) and `70000` (`PORT`) throw; `SHEETS_FETCH_TIMEOUT_MS=0` gives `0`; `LIVE_PUSH_URL=ftp://x` and `not a url` throw; three bad values give one `ConfigError` naming all three; numbers passed instead of strings work; the result and its nested objects are frozen; `unsetWarning` names exactly the unset warned variables, flags a URL without a secret and the reverse, and returns `null` when all are set |
| `server/errorHandler.mjs` (2) | each class in §4.7 gives its status, `code` and `extra`; a `ScoreConflictError` body has `current`; a plain `Error` is `500 internal_error` with its message; an `Error` with `code: "ECONNRESET"` still says `internal_error`; `{ statusCode: 400 }` from body-parser is `400 bad_request`; `res.headersSent` delegates to `next` and writes nothing; 4xx logs `warn`, 5xx logs `error`, once each; a `401` sets `WWW-Authenticate: Bearer realm="sage-tools-api"`. Phase 5 adds: on a `/v3` path the body is problem details (every member of §4.11), `unknown_day` becomes `404`, a `405` sets `Allow`, and a legacy path keeps `{ error, code }` |
| `server/asyncHandler.mjs` (2) | a resolved handler does not call `next`; a rejected one calls `next(err)`; a synchronous throw calls `next(err)` |
| `server/cors.mjs` (2) | the four headers on a normal response; `OPTIONS` answers `204` and does not call `next` |
| `auth/middleware.mjs` (2) | each function in §4.9: passes and sets `req.actor`; missing, garbled or expired token → `next(UnauthorizedError)`; a desk token on `requireOperatorOr("scorer")` and a scorer token on `requireOperatorOr("desk")` → 401; a desk or scorer token on `requireOperator` → 401; secret matches; wrong secret → 401; unset configured secret with an empty header → 401 |
| `app.mjs` (2) | `createApp(loadConfig(defaultEnv()), { logger })` returns every named object; `server.app` is a function; no scoresheet import happened (`getScoresheetService` was not called: pass a spy) |
| `shared/conflictRetry.mjs` (3) | first success → `attempts: 1`; `{ ok: false }` retries and calls `onConflict(n, max)` after every conflict including the last; throws `ConflictError` (`statusCode`, `status` 409, `code: "conflict"`) after the limit, with the last conflict's error message or else `opts.message`; a throw from `attemptFn` propagates without retry or `onConflict`; a custom `attempts` is honoured; an async `onConflict` is awaited |
| `sync/domain/snapshotStamp.mjs` (3) | `stamp(s)` prefers `publishedAt` over `generatedAt`; neither or `null` gives `0`; `newer(a, b)` picks the later, prefers `a` on a tie, handles `null` on either side and on both |
| `sync/domain/syncTiming.mjs` (3) | `buildTiming({ t0, editAt, fetchMs, publishMs, publishedAtMs, liveMs, archiveMs })`: `editToRequestMs`/`editToPublishedMs` are `null` without `editAt`; the log line prints `n/a` for `null` and otherwise `<n>ms` in the order `edit→request=… fetch=… publish=… live=… archive=… edit→published=…` (exactly today's text) |
| `sync/domain/editAt.mjs` (3) | `parseEditAt(header, now)`: inside the last hour and at most 5 s ahead → the number; older than an hour, more than 5 s ahead, `NaN`, empty, missing → `null` (today's inline rule in `handleSync`) |
| `sync/domain/mergeSnapshot.mjs` (3) | on the pure function, the merge cases of `SyncService.test.mjs`: fresh wins; untargeted is carried forward; a failed facility is carried forward and listed `stale`; a facility never seen and failed is omitted; nothing to publish throws `SyncUpstreamError`; `completedAt` is stamped once and carried; `lastEditAt` comes from the edit or is carried; a fresh `rosterCsv` replaces the old one and a fetch without one keeps it; `publishedAt` starts as `now` |
| `test/helpers/snapshotStoreContract.mjs` (3) | **the shared store contract** (LSP). `runSnapshotStoreContract(describeName, makeStore)` takes a factory returning `{ store, seed(ref, snapshot), failNextWrite(kind), failNextRead() }`, where `kind` is `"conflict"` or `"down"`, and asserts: an empty read is `{ snapshot: null }`; a written snapshot reads back; a write with a stale token resolves `{ ok: false, conflict: true }`; a write after `failNextWrite("down")` and a read after `failNextRead()` reject with a `StoreUnavailableError` whose `store` is the store and whose message is the underlying one. Both store test files call it |
| `sync/publishing/StoreUnavailableError.mjs` (3) | keeps the message; sets `cause`, `store`, and `status` when the cause has one; never sets `statusCode`; is not an `AppError` |
| `sync/publishing/GitHubSnapshotStore.mjs` (3) | runs the contract against `FakePublisher`; also: write passes the token as `knownSha`; `token: null` lets the publisher look up the sha; a 409 carries the GitHub error as `error`; a 403 becomes `StoreUnavailableError` with `status` 403 |
| `sync/publishing/LiveSnapshotStore.mjs` (3) | runs the contract against `FakeLivePublisher`; also: read and write map as §4.4 |
| `shared/ports.mjs`, `sync/publishing/ports.mjs` (2, 3) | typedef-only, so an entry each in `test/helpers/coveredElsewhere.mjs` naming `dependency-rules.test.mjs`, which checks R0 |
| `registry/domain/validateRegistry.mjs` (3) | moved out of `SyncConfigStore.mjs` unchanged: the validation cases already in `SyncConfigStore.test.mjs`'s "validation" block, called directly on `validate(raw)`, plus the reserved day keys. Leave the store's own tests where they are |
| `registry/domain/registryEdits.mjs` (3) | `setDayIsLive(raw, day, isLive)`, `setLivePushSwitch(raw, enabled)`, `setEventScoreEntry(raw, eventKey, mode)`: each edits `raw` and returns `{ result, message }`, or `{ result }` with no `message` when nothing changes; unknown day → `UnknownSyncDayError`; unknown event → `UnknownEventError`; an event without `scoreEntry`, or a bad mode → `ValidationError` with today's messages; the result values equal today's setters' return values |
| `sync/publishing/mergeIntoStore.mjs` (3) | success first try; a conflict then success re-reads and re-builds against the new base; `readBase` override used; `build` returning `null` → `written: false`, nothing written; `publishedAt` stamped before each write; `ConflictError` after three; a non-conflict error propagates; `build` throwing propagates unretried |
| `registry/SyncConfigStore` reserved keys (2) | a day key `config` or `live-push` is rejected with a message naming it (flips the `CHARACTERIZATION B4` pin) |
| `sync/publishing/GitHubOnlyDelivery.mjs` (4) | with an in-memory `SnapshotStore` fake: publish returns the `PublishResult` with `liveMs`/`archiveMs` `null` and no `live`/`archive`; a conflict then success re-merges; three conflicts → `ConflictError`; republish with and without a stored snapshot; the two log lines, logged only while `n < max` |
| `sync/publishing/LiveFirstDelivery.mjs` (4) | with two in-memory stores and a spy fallback: primary success with archive; primary conflict ×3 → fallback, with `live.error` `live publish: 3 version conflicts` and the fallback's result otherwise unchanged; primary read or write down → fallback; archive store down while the primary is empty → throws, no fallback; `SyncUpstreamError` from `build` → throws, no fallback; the stale-object rule both ways and the tie; an archive conflict re-reads the primary; an archive failure → `{ committed: false }` while the publish still succeeds; every row of the republish table that involves live; every log line of §4.5 and §4.6 with its exact text. It never imports a store class (the guard checks this) |
| `sync/publishing/SnapshotPublisher.mjs` (4) | `liveActive` for the four combinations of `enabled` × `livePushOn`; `publish` and `republish` go to `liveFirst` exactly when `liveActive`, with `config` stripped from the arguments; the result is returned untouched |
| `sync/DayVisibilityService.mjs` (4) | writes the config first, then republishes; result is `{ day, label, isLive, ...republish result }`; the three log lines (`isLive set to …` with `(no published snapshot yet to republish)`, `, republished live`, `, republished`) |
| `sync/SyncSettingsService.mjs` (4) | `setLivePush` → `{ enabled, changed, available }`, with `available` from `livePublisher.enabled`; `describe()` gives today's `/sync/config` payload, `active` from `snapshotPublisher.liveActive` |
| `sync/syncController.mjs` (4) | an entry in `test/helpers/coveredElsewhere.mjs` naming `test/integration/sync-routes.test.mjs` |
| dependency rules (4) | `test/unit/guards/dependency-rules.test.mjs`: strip comments (`/* … */` and `// …`, outside strings) from every `src/**/*.mjs`, then scan the rest for static and dynamic imports, `process.env` and `new <Class>(`, and check R0–R10 of §4.3. Keep the rule table as data at the top of the test. A failure names the file, the import and the rule. Add one case that feeds it a JSDoc `import("…")` inside a comment and expects no violation |
| `server/requireJson.mjs` (5) | JSON body passes; `text/plain` with a body → `UnsupportedMediaTypeError`; no body passes in both modes; `application/json; charset=utf-8` passes |
| `server/methodNotAllowed.mjs` (5) | a route with `get` and `put` gives `Allow` `GET, HEAD, PUT`; one with `put` gives `PUT`; `_all` is never listed |
| `server/deprecation.mjs` (5) | sets `Deprecation: @<LEGACY_DEPRECATED_AT>` and the `Link` header built from `req.params`; calls `next` |
| REST conventions (5) | `test/unit/guards/rest-conventions.test.mjs`, specified in §6.5 step 2 |

The `/v3` parity tests are specified in §6.5.

### 5.3 Pins resolved along the way

`grep -rn "CHARACTERIZATION B" test/` lists what is still pending. Flipping a
pin means editing the assertion to the new expectation and deleting its
`CHARACTERIZATION B<n>` comment. After Phase 4 the grep finds nothing.

The attendance and score-entry tests were written after the markers existed,
and they assert exact error bodies **without** a marker. Examples are
`{ error: "Unauthorized" }` and, in `score-routes.test.mjs`,
`{ error: "The sheet changed since this match was opened", current }`. They
change under B2 in Phase 2 too: find them with
`grep -rn "{ error: " test/integration test/unit/helpers` and add each one's
`code`. A marker with no number
(`// CHARACTERIZATION:` in `AuthService.test.mjs`) pins an unscheduled defect
and stays.

`test/unit/sync/SyncConfigStore.test.mjs`'s `setScoreEntry` "stops retrying
after three 409s" test has no marker but is part of B1. It changes in Phase 3
too.

---

## 6. Phases

Each phase ends with `npm run verify` green, a version bump where stated, a
Changelog entry with its **Tests:** line, the docs it affects (§7), and one
commit on `arch-refactor`. Phases are strictly ordered: do not start one
before the previous one is green.

| Phase | What | Version | Production code touched |
|---|---|---|---|
| 1 | Folder moves | 2.8.1 | paths and imports only, plus `parseCsv` extracted |
| 2 | Hardening and one composition root | 2.8.2 | config, `createApp`, errors, auth middleware, CORS, secret compare, reserved keys, jsconfig |
| 3 | One retry loop, store interface, domain extraction | 2.8.3 | `shared/`, `sync/` internals, `registry/SyncConfigStore` |
| 4 | Delivery strategies and the `SyncService` split | 2.8.4 | `sync/`, `app.mjs`, JSDoc on two services |
| 5 | The `/v3` API and its REST conventions | 3.0.0 | routes, controller, three small `server/` middlewares, the error handler's `/v3` branch, OpenAPI, 404 handler |
| 6 | Every client on `/v3` | none (site repo, `apps-script/`) | Control Center, the scoresheet generator, the `ATTENDANCE CLIENT` and `SCORE CLIENT` blocks, the masters' `sheets-sync.gs` |
| 7 | Build trigger, docs, workspace | none | none |
| 8 | Site shared-JS decision | n/a | none |

### 6.1 Phase 1 — folder moves (2.8.1)

Mechanical, with the test suite as the guard. Do every move with `git mv`,
update imports, and run `npm test` after each group. Each test file moves with
`git mv` to mirror its module's new path.

**1a. Apps Script out of `scripts/`.**

| From | To |
|---|---|
| `scripts/sheets-sync.gs`, `sheet-generator.gs`, `standard-generator.gs`, `attendance.gs` | `apps-script/` |
| `scripts/mock-apps-script.mjs` | `apps-script/` |
| `scripts/verify-attendance.mjs`, `verify-sheet-generator.mjs`, `verify-standard-generator.mjs` | `apps-script/` |
| `scripts/fixtures/` | `apps-script/fixtures/` |
| `scripts/verify-sync-merge.mjs`, `scripts/verify-facility-completion.mjs` | **deleted** (`git rm`). They import `src/sync/*` paths this phase moves, and both are ported to `node:test` (`test/unit/sync/SyncService.test.mjs`, `test/unit/sync/facilityCompletion.test.mjs`) |
| `scripts/hash-password.mjs`, `scripts/run-appscript-verifies.mjs` | stay; the runner now points at `apps-script/` |
| the `.gs`-related settings of `jsconfig.json` (`include: ["**/*.gs"]` and the `google-apps-script` `typeAcquisition`) | a new `apps-script/jsconfig.json` with `include: ["*.gs"]`, the same `typeAcquisition`, and `compilerOptions` `{ "target": "ES2019", "lib": ["ES2019"], "checkJs": false }`. The root `jsconfig.json` loses its `.gs` entry here; Phase 2 rewrites the rest of it |

The verify scripts locate the `.gs` files, `mock-apps-script.mjs`, fixtures,
and (`verify-attendance.mjs`, the two generator verifies)
`src/attendance/attendanceTab.mjs` relative to themselves. Fix every relative
path: `apps-script/` is one level below the repo root, just as `scripts/` was,
but `attendanceTab.mjs` moves in 1c. Confirm with `npm run test:appscript`.

**1b. Outbound clients and the registry out of the feature folders.**

| From | To |
|---|---|
| `src/sync/GitHubPublisher.mjs`, `LivePublisher.mjs`, `SheetsCsvFetcher.mjs`, `GvizCsvFetcher.mjs` | `src/clients/` |
| `src/attendance/SheetsClient.mjs`, `GoogleAccessToken.mjs` | `src/clients/` |
| `src/sync/SyncConfigStore.mjs`, `SyncConfigSnapshot.mjs`, `events.seed.json` | `src/registry/` |
| `test/unit/sync/attendanceConfig.test.mjs`, `scoreEntryConfig.test.mjs` | `test/unit/registry/` (they test `SyncConfigStore`/`SyncConfigSnapshot`) |

`COMMIT_ATTEMPTS` stays exported from `clients/GitHubPublisher.mjs` until
Phase 3.

**1c. Pure logic into each feature's `domain/`.**

| From | To |
|---|---|
| `src/sync/facilityCompletion.mjs`, `teamRoster.mjs` | `src/sync/domain/` |
| `src/attendance/attendanceTab.mjs`, `personKey.mjs`, `roster.mjs` | `src/attendance/domain/` |
| `src/scores/scheduleGrid.mjs`, `scoreRequest.mjs` | `src/scores/domain/` |
| `parseCsv` (a function inside `facilityCompletion.mjs`) | `src/shared/csv.mjs`. `facilityCompletion.mjs` and `AttendanceService.mjs` import it from there, with no re-export. Its tests move to `test/unit/shared/csv.test.mjs` |

**1d. Update every reference:**

- every `import` in `src/`, `test/` and `index.mjs`;
- `index.mjs`'s seed path (`src/registry/events.seed.json`) and the same path
  in `test/helpers/buildApp.mjs`;
- `src/docs/openapiSpec.mjs`'s `apis` list, which is unchanged because no
  route file moves (confirm it);
- `test/helpers/coveredElsewhere.mjs` entries, if any name a moved file;
- `package.json`'s `test:appscript`, if it names a path.

Check that `every-module-tested.test.mjs` maps a nested folder
(`src/sync/domain/x.mjs` → `test/unit/sync/domain/x.test.mjs`). If it only
handles one level, fix the guard first, as a test-only change.

**1e. Coverage for every folder** (F23). Add these to `package.json`, each
with `--test-coverage-lines=90` and the same test globs as the existing
scripts, and chain them into `test:coverage`:

- `test:coverage:clients` (`src/clients/**`)
- `test:coverage:registry` (`src/registry/**`)
- `test:coverage:attendance` (`src/attendance/**`)
- `test:coverage:scores` (`src/scores/**`)

`test:coverage:sync` keeps `src/sync/**`. All of these measured 97 % lines or
more at 2.8.0.

**1f. Documentation paths.** Replace every mention of a moved path in
markdown and comments. Find them with:

```bash
grep -rIn --exclude-dir=node_modules --exclude-dir=.git \
  -e "scripts/sheets-sync" -e "scripts/sheet-generator" -e "scripts/standard-generator" \
  -e "scripts/attendance.gs" -e "scripts/mock-apps-script" -e "scripts/verify-" -e "scripts/fixtures" \
  -e "src/sync/facilityCompletion" -e "src/sync/teamRoster" -e "src/sync/GitHubPublisher" \
  -e "src/sync/LivePublisher" -e "src/sync/SheetsCsvFetcher" -e "src/sync/GvizCsvFetcher" \
  -e "src/sync/SyncConfig" -e "src/sync/events.seed" \
  -e "src/attendance/SheetsClient" -e "src/attendance/GoogleAccessToken" \
  -e "src/attendance/attendanceTab" -e "src/attendance/personKey" -e "src/attendance/roster" \
  -e "src/scores/scheduleGrid" -e "src/scores/scoreRequest" \
  /d/Personal/SAGE/sage-tools-api /d/Personal/SAGE/sage-docs \
  /d/Personal/SAGE/sage-match-control.github.io /d/Personal/SAGE/CLAUDE.md \
  /d/Personal/SAGE/event-data/config/README.md
```

- Replace each match with the new path. Historical specs keep their prose;
  only the path text changes.
- Where a doc tells someone to *run* `verify-sync-merge.mjs` or
  `verify-facility-completion.mjs`, replace the instruction with `npm test`.
  This covers the root `CLAUDE.md`'s `sage-tools-api` section, which lists
  both as commands, and its "Things that must be kept in sync by hand" item,
  which names `verify-facility-completion.mjs`.
- The `.gs` header comments that say "paste `scripts/…`" change too. A `.gs`
  comment change does not bump the version.
- `sage-tools-api/README.md`'s "Bound Apps Script in `scripts/`" section
  becomes "… in `apps-script/`".

**Acceptance.**

- `npm run verify` is green, with the four new coverage scripts.
- The grep above returns nothing.
- `node index.mjs` starts, and `GET /ping` answers over h2c
  (`curl --http2-prior-knowledge http://localhost:8080/ping`).
- `docker build .` succeeds, if Docker is available. Otherwise state in the
  commit that it was not run.
- `git log --follow` on a moved file shows its history.
- `grep -rn "\.\./attendance/\|\.\./scores/\|\.\./sync/" src/attendance src/scores src/sync src/clients`
  finds only imports of a `domain/` file.

### 6.2 Phase 2 — hardening and one composition root (2.8.2)

Do these steps in order, and write the failing test first for each one.

1. **`shared/safeEqual.mjs`** (F5), per §4.9.
2. **`shared/ports.mjs`**: every row of §4.4's port table except
   `SnapshotStore` and `SnapshotDelivery`, as JSDoc typedefs plus one
   `export {};`. `SnapshotPublishing` refers to
   `import("../sync/publishing/ports.mjs")` types, which Phase 3 creates;
   until then, write those argument types as `object`. Add its
   `coveredElsewhere.mjs` entry. From here on, every new constructor's JSDoc
   names the port it receives (§4.12 I).
3. **Errors** (F6, B2, B7), per §4.7:
   - Add `code` and `extra` to `AppError` and every subclass, and add
     `ConflictError`, `ConfigError` and `UnknownFacilityError`. Switch
     `SyncService`, `AttendanceService` and `ScoreService` to
     `UnknownFacilityError`, keeping each one's message, so only `code`
     changes in their bodies (`unknown_facility`).
   - Write `server/errorHandler.mjs` and `server/asyncHandler.mjs`.
   - Extend `test/unit/shared/errors.test.mjs` with every class's `code`.
4. **`config/loadConfig.mjs`** (F9, B6), per §4.8. Add
   `test:coverage:config` (`src/config/**`, lines ≥ 95) to `package.json` and
   to `test:coverage`. Update `.env.example`'s comments to the §4.8 rules.
5. **`auth/middleware.mjs`** (F8), per §4.9.
6. **`server/cors.mjs`** (F14, B3). Move the CORS middleware out of `Server`
   and change the methods to `GET, POST, PUT, PATCH, DELETE, OPTIONS`. The
   other headers stay byte-for-byte the same.
7. **`src/app.mjs`** (F22), per §4.10. Then:
   - `index.mjs` shrinks to: read `APP_VERSION` from `package.json`; create
     `new Logger("scoresheet")`; `loadConfig()` inside a `try`. On a
     `ConfigError` it calls `logger.error(err.message)` and
     `process.exit(1)` before anything listens. It logs `unsetWarning`'s line
     as `warn` if there is one, then calls
     `createApp(config, { logger, version, templatesDir: path.join(process.cwd(), "templates") })`
     and `server.start()`.
   - `test/helpers/buildApp.mjs` builds its env as today
     (`{ ...defaultEnv(), ...env }`), calls `loadConfig` on it, then
     `createApp(config, { logger: silentLogger, version, getScoresheetService: <today's throwing stub> })`.
     It returns the same fields it returns today, so no test that uses it
     changes.
8. **`Server` and the route files.**
   - `Server`'s constructor becomes
     `{ routers, peekConfig, logger, port, corsOrigin, version }` (§4.10). It
     no longer imports `scoresheets/routes.mjs`, `sync/routes.mjs`,
     `auth/routes.mjs`, `attendance/routes.mjs` or `scores/routes.mjs`. Those
     move into `createRouters` in `src/app.mjs`, which passes each factory
     the `auth` object from `createAuthMiddleware` and the services it needs.
     `/ping` calls `peekConfig()` where it called `syncConfigStore.peek()`.
   - `Server` registers `cors`, then the version header, then
     `express.json()`, then each `{ path, router }` in order, then
     `errorHandler(logger)` **last**.
   - Every route factory takes `auth` and uses its functions. Delete the local
     `bearerToken`, `hasValidSyncSecret`, `hasValidAuthToken`,
     `requireAuthToken`, `requireSyncSecretOrAuthToken`, `requireOperator`,
     `requireOperatorOrDesk` and `requireOperatorOrScorer`, and scores' `fail`.
   - Every async handler is wrapped in `asyncHandler`, throws instead of
     catching, and loses its `try/catch` and its own `logger.error` line. The
     route-level failure logs become the handler's generic line (§3.2).
   - The **streaming** scoresheet handler keeps its own `catch`, because it
     must write an NDJSON error line after the headers are sent. The handler's
     `headersSent` check covers anything that escapes it.
   - `/openapi.json` logs the original error, then throws
     `new AppError("Could not generate OpenAPI spec.")`.
   - `test/helpers/http.mjs`'s `startApp` keeps its signature and its
     default stub services. As today, it merges `services` over them into
     `s`. It then builds
     `auth = createAuthMiddleware({ authService: s.authService, syncSharedSecret })`
     and `routers = createRouters({ services: s, auth, getScoresheetService: s.getScoresheetService, logger: silentLogger })`.
     It constructs `Server` with those and
     `peekConfig: () => s.syncConfigStore.peek()`. No integration test's
     expectations change.
9. **OpenAPI.** In `src/docs/openapiSpec.mjs`'s definition, add
   `components.schemas.Error`: `{ type: object, required: [error, code],
   properties: { error: { type: string }, code: { type: string } } }`.
   Replace every inline error schema in the `@openapi` blocks with
   `$ref: '#/components/schemas/Error'`. The score route's 409 keeps its own
   schema with `current`, and gains `code`. Then run
   `npm run test:snapshots` and review the snapshot diff.
10. **Reserved day keys** (F7, B4). `SyncConfigStore`'s `validate` rejects a
   day key in `RESERVED_DAY_KEYS = ["config", "live-push"]` with
   `day key "<key>" (event "<event>") is reserved`. Flip the B4 marker.
11. **`Server.start()`** (F11). Replace the comment with the truth: Cloud Run
    talks h2c to the container, and cleartext HTTP/1.1 clients cannot connect.
    `start()` returns the listening server.
12. **`jsconfig.json`** (F10). Set:
    - `compilerOptions`: `target` and `lib` `ES2022`, `module` and
      `moduleResolution` `NodeNext`, `checkJs: false`;
    - `include: ["index.mjs", "src/**/*.mjs", "test/**/*.mjs"]`;
    - `exclude: ["node_modules", "apps-script", "live-worker", "spikes"]`.

    Add `// @ts-check` as the first line of every module this phase creates
    (`safeEqual`, `loadConfig`, `errorHandler`, `asyncHandler`, `cors`,
    `middleware`, `app`), and of every module Phases 3–4 create. Make them
    clean: open each in VS Code and see no red squiggles. If VS Code is not
    available, run `npx -y -p typescript tsc --noEmit -p jsconfig.json` once
    as a check. It downloads TypeScript temporarily and adds no dependency.
    Errors it reports in files without `// @ts-check` are expected and
    ignored. Do not turn `checkJs` on globally.

Update the B2 markers (every error body now has `code`), B3 and B4 in the same
commit. Add the B7 test for a malformed JSON body (`POST /auth/login` with body
`{` → `400`, `code: "bad_request"`) to `test/integration/headers.test.mjs`.

**Acceptance.**

- The suite is green, with test edits limited to B2, B3, B4, B7, wiring and
  the snapshot.
- `SHEETS_FETCH_TIMEOUT_MS=abc node index.mjs` exits with status 1, prints
  `Invalid environment:` and names the variable.
- `LIVE_PUSH_TIMEOUT_MS= node index.mjs` starts.
- `grep -rn "statusCode ?? 500\|res.status(401)" src` finds nothing.
- `grep -rn "process.env" src` finds only `src/config/loadConfig.mjs`.
- `grep -rln "new GitHubPublisher\|new SyncService\|new Server" src index.mjs`
  finds only `src/app.mjs`.
- `Server.mjs` imports no route module and nothing from `registry/`,
  `attendance/`, `scores/`, `sync/` or `scoresheets/` (R10).

### 6.3 Phase 3 — one retry loop, one store interface, domain extraction (2.8.3)

Write each new module's tests from §5.2 **first**.

1. **`shared/conflictRetry.mjs`** (§4.5). Move `COMMIT_ATTEMPTS` here.
   `clients/GitHubPublisher.mjs` stops exporting it, and every importer
   switches.
2. **The registry, split by responsibility** (F4, B1, §4.12 S).
   - **`registry/domain/validateRegistry.mjs`**: move `validate`,
     `SUPPORTED_VERSION`, `SLUG_RE` and `RESERVED_DAY_KEYS` out of
     `SyncConfigStore.mjs` unchanged, and export `validate`. The store and
     `SyncConfigSnapshot` (if it validates) import it.
   - **`registry/domain/registryEdits.mjs`**: three pure functions, each
     taking the validated `raw` object, editing it in place, and returning
     `{ result, message }` (commit) or `{ result }` (unchanged, nothing to
     commit). Their bodies are today's loop bodies, without the I/O:
     - `setDayIsLive(raw, day, isLive)` → `result: { event, label }`,
       `message: \`set ${day} isLive=${isLive}\``. It throws
       `UnknownSyncDayError` as today. It always commits, as today.
     - `setLivePushSwitch(raw, enabled)` → `result: { enabled, changed }`,
       `message: \`set livePush=${enabled}\``.
     - `setEventScoreEntry(raw, eventKey, mode)` →
       `result: { event, scoreEntry, changed }`,
       `message: \`set ${eventKey} scoreEntry=${mode}\``, with today's
       `UnknownEventError` and `ValidationError`s. The mode check stays
       before any I/O, in the `SyncConfigStore.setScoreEntry` wrapper, as
       today.
   - **`SyncConfigStore`** keeps loading, caching, the seed fallback and one
     private helper that commits an edit with conflict retry:

     ```js
     // edit(raw) -> { result, message } to commit, or { result } to return without committing
     async #update(edit) {
         const { value } = await withConflictRetry(async () => {
             const fetched = await this.publisher.fetchExisting(this.path);
             if (!fetched.json) throw new Error(`config not found at ${this.path}`);
             const raw = validate(fetched.json);
             const { result, message } = edit(raw);   // throws UnknownSyncDayError etc. unretried
             if (!message) return { ok: true, value: result };
             try {
                 await this.publisher.publish(this.path, raw, message, fetched.sha);
             } catch (err) {
                 if (err.status === 409) return { ok: false, error: err };
                 throw err;
             }
             this.cached = null;
             this.cachedAt = 0;
             return { ok: true, value: result };
         }, { onConflict: (n, max) => { if (n < max) this.logger.info(`commit conflict (attempt ${n}/${max}) — re-reading and re-applying`); } });
         return value;
     }

     setIsLive(day, isLive)        { return this.#update(raw => setDayIsLive(raw, day, isLive)); }
     setLivePush(enabled)          { return this.#update(raw => setLivePushSwitch(raw, enabled)); }
     async setScoreEntry(event, mode) {
         if (mode !== "console" && mode !== "links") throw new ValidationError('mode must be "console" or "links"');
         return this.#update(raw => setEventScoreEntry(raw, event, mode));
     }
     ```

     The three public methods keep their names, arguments, validation,
     commit messages and return values.
   - **B1:** three conflicts now throw `ConflictError` (`statusCode` 409).
     - Edit the three B1 tests in `SyncConfigStore.test.mjs` (including
       `setScoreEntry`'s unmarked one) to expect
       `e instanceof ConflictError && e.statusCode === 409 && e.status === 409`.
     - Edit `sync-routes.test.mjs`'s "raw GitHub 409" case so the stub throws
       `new ConflictError("GitHub commit failed: HTTP 409 does not match")`
       and expects `409 { error, code: "conflict" }`.
3. **The snapshot stores** (F3, §4.4): `sync/publishing/ports.mjs` (the
   `SnapshotStore` typedefs only, for now), `StoreUnavailableError.mjs`,
   `test/helpers/snapshotStoreContract.mjs`, `GitHubSnapshotStore.mjs` and
   `LiveSnapshotStore.mjs`, each with its unit test (§5.2).
4. **`sync/publishing/mergeIntoStore.mjs`** (§4.5) and its tests.
5. **Extract the pure parts of `SyncService`** into `sync/domain/`.
   `SyncService` calls them, with no behaviour change: the existing
   `SyncService` tests stay green untouched.
   - `mergeSnapshot.mjs`: `#buildSnapshot`, same inputs and output, now taking
     the base snapshot (`object|null`) instead of `existing`.
   - `snapshotStamp.mjs`: `stamp` and `newer`, from `#readLiveBase`.
   - `syncTiming.mjs`: the timing object and its log line.
   - `editAt.mjs`: `parseEditAt`, from the window rule inline in
     `src/sync/routes.mjs`'s `handleSync`, which then calls it.
6. `SyncService`'s own loops are **not** converted here. Phase 4 replaces them
   wholesale.

**Acceptance.**

- The suite is green.
- `grep -rn "COMMIT_ATTEMPTS" src` finds the definition in
  `shared/conflictRetry.mjs`, plus uses in `SyncService` (until Phase 4).
- `grep -n "for (let attempt\|^function validate" src/registry/SyncConfigStore.mjs`
  finds nothing.
- `grep -rn "CHARACTERIZATION B" test` finds only the `syncDay` B1 test in
  `sync-pipeline.test.mjs`.

### 6.4 Phase 4 — delivery strategies and the `SyncService` split (2.8.4)

Goal: `SyncService` knows nothing about GitHub, the Worker, the archive or the
fallback, and every class in `sync/` has one reason to change (§4.12).

1. **Delivery** (F1, F2, §4.6), test first for each:
   - add the `PublishArgs`, `RepublishArgs`, `PublishResult`,
     `RepublishResult` and `SnapshotDelivery` typedefs to
     `sync/publishing/ports.mjs`, and replace the placeholder `object` types
     in `shared/ports.mjs`'s `SnapshotPublishing`;
   - `GitHubOnlyDelivery.mjs`;
   - `LiveFirstDelivery.mjs`;
   - `SnapshotPublisher.mjs`.
2. **`sync/SyncService.mjs`.** Its constructor takes
   `{ fetchers, snapshotPublisher, configStore, logger, onFacilitiesSynced }`,
   where `fetchers` is `{ sheets: FacilityFetcher, csv: FacilityFetcher }`.
   **`syncDay(day, { facilityName, method, editAt })` keeps its exact
   signature and return value: `ScoreService` calls it.** The body:
   1. Resolve the day and the target facilities, with today's two
      `ValidationError`s and messages.
   2. Pick `this.fetchers[method]` (`method` is already `"sheets"` or
      `"csv"`). Fetch in parallel and build `freshByName`, `failed` and
      `lastEditAt` exactly as today. The fetch log line keeps its text,
      including `via Sheets API` / `via gviz CSV export (fallback)`.
   3. Call `snapshotPublisher.publish({ config, ref, build, message, log })`,
      where `build = base => mergeSnapshot({ ..., base })` and `message` is
      today's commit message function. Measure `publishMs` around this call.
   4. Call `buildTiming` and log the `timing` and `done` lines.
   5. Await `onFacilitiesSynced` exactly as today: after publish and archive,
      only when something was fetched fresh, with a throw logged and
      swallowed.
   6. Return today's body:
      `{ day, label, method, facilitiesSynced, facilitiesFailed, facilitiesStale, commitSha, attempts, ...(live), ...(archive), timing }`.

   `setLiveOverride`, `setLivePush`, `#liveActive`, `#readLiveBase`,
   `#publishViaGitHub`, `#publishViaLive`, `#archiveToGitHub` and
   `#overrideViaLive` are all removed from it.
3. **`sync/DayVisibilityService.mjs`**:
   `constructor({ configStore, snapshotPublisher, logger })` and
   `setVisibility(day, isLive)`. It calls `configStore.setIsLive`, then
   `configStore.get()`, then `snapshotPublisher.republish` with
   `mutate = s => ({ ...s, isLive })`, `message: \`set ${day} isLive=${isLive}\``
   and `label`. It logs the matching line and returns
   `{ day, label, isLive, ...result }`.
4. **`sync/SyncSettingsService.mjs`**:
   `constructor({ configStore, liveAvailability, snapshotPublisher, logger })`,
   where `app.mjs` passes `livePublisher` as `liveAvailability`. It has:
   - `setLivePush(enabled)`: today's `SyncService.setLivePush`, log line
     included, with `available` from `liveAvailability.enabled`;
   - `describe()`: today's body of `handleConfigDiagnostics`, with
     `enabled`/`baseUrl` from `liveAvailability` and `active` from
     `snapshotPublisher.liveActive(snap)`.
5. **`sync/syncController.mjs`**:
   `createSyncController({ syncService, dayVisibilityService, syncSettingsService })`
   returns four handlers, each `async (req, res)`:
   - `syncDay`: parses `facility` and `method` as today, and `X-Edit-At`
     through `parseEditAt(req.get("X-Edit-At"), Date.now())`;
   - `setDayVisibility`: validates `isLive` with today's `ValidationError`
     message;
   - `setLivePush`: validates `enabled` with today's message;
   - `getDiagnostics`.

   Each answers `200` with the service's result. `src/sync/routes.mjs` shrinks
   to URL mapping, `auth` middleware, `asyncHandler` and the `@openapi`
   blocks. `syncRoutes` takes `{ controller, auth }` and no longer receives
   `syncService` or `syncConfigStore`.
6. **`app.mjs`** builds the publishing graph as in §4.10, the `fetchers` map,
   the three sync services and the controller, and passes the controller to
   `createRouters`. It passes `syncService` to `ScoreService` and attendance's
   hook as today, and returns the new objects too.
7. **Ports on the untouched services** (§4.12 I). Add the port types from
   §4.4 to the constructor JSDoc of `AttendanceService`, `ScoreService`,
   `SyncConfigStore`, `AuthService`'s consumers and the two snapshot stores.
   This is a comment-only change to `AttendanceService` and `ScoreService`:
   no `// @ts-check` is added to them, and no line of code changes.
8. **Tests.**
   - `test/unit/sync/SyncService.test.mjs` and `onFacilitiesSynced.test.mjs`
     construct `SyncService` directly with `FakePublisher` and
     `FakeLivePublisher`. Add `test/helpers/makeSyncService.mjs`. It takes
     today's constructor arguments (`sheetsApiFetcher`, `gvizFetcher`,
     `publisher`, `livePublisher`, `configStore`, `onFacilitiesSynced`,
     `logger`) and builds the stores, both deliveries, the
     `SnapshotPublisher`, the `fetchers` map and the three services exactly
     as `app.mjs` does. It returns
     `{ syncService, dayVisibilityService, syncSettingsService }`.
   - Switch both files to `makeSyncService`. Tests that called
     `syncService.setLiveOverride` or `.setLivePush` call
     `dayVisibilityService.setVisibility` or `syncSettingsService.setLivePush`.
     **No expected value changes**, except the B1 `syncDay` case.
   - `test/integration/sync-routes.test.mjs` stubs services: give its stub the
     new shape (`dayVisibilityService`, `syncSettingsService`) through
     `startApp`'s `services`. Its expected bodies do not change.
   - `sync-pipeline.test.mjs`'s B1 test: three GitHub conflicts on the
     GitHub-only path now answer `409`, `code: "conflict"`, with `error` still
     matching `/^GitHub commit failed: HTTP 409/`. Remove its marker.
   - Add `test/unit/guards/dependency-rules.test.mjs` (§4.3, §5.2) and fix
     whatever it finds.
9. Update `sage-docs/docs/technical/sync-pipeline.md` (the ports, stores,
   `mergeIntoStore`, the two deliveries and `SnapshotPublisher`) and
   `technical/architecture.md` (§4.0 and §4.12 in prose).

**Acceptance.**

- Every test passes, with edits limited to wiring and B1.
- `grep -rn "CHARACTERIZATION B" test` finds nothing.
- `SyncService.mjs` is under 200 lines, and `syncDay` under 80.
- `grep -n "livePublisher\|GitHubPublisher\|COMMIT_ATTEMPTS\|livePushOn\|gviz\|Gviz" src/sync/SyncService.mjs`
  finds only the fetch log line's `gviz CSV export (fallback)` text.
- `grep -rn "livePushOn" src` finds only the registry (which defines it),
  `SnapshotPublisher.liveActive` and `SyncSettingsService.describe` (the
  `switch` field).
- `grep -n "GitHub\|Live\|import" src/sync/publishing/LiveFirstDelivery.mjs`
  shows imports only of `mergeIntoStore`, `StoreUnavailableError`,
  `conflictRetry`, `shared/errors` and `sync/domain/snapshotStamp`. Mentions
  of GitHub and live are log text only.
- The dependency guard is green.
- Every row of §4.12 holds. The Changelog says which files a third delivery
  target would touch: a store adapter, a delivery or a reuse of
  `LiveFirstDelivery`, `SnapshotPublisher.#select` and `app.mjs`.

### 6.5 Phase 5 — the `/v3` API (3.0.0)

A major release: `package.json` goes from 2.8.4 to **3.0.0**. Its Changelog
entry opens with "Breaking: none for existing clients", followed by what is
new: `/v3`, its conventions, and every older route kept and frozen. Write the
tests in steps 1 and 2 first.

1. **Parity and resource tests:** `test/integration/v3.test.mjs`,
   table-driven over §4.11's routes table.
   - **Twins.** For each row with a twin, the twin and the `/v3` request
     produce the same success status (except `201` for sessions) and the same
     success body, for equivalent inputs:
     - the legacy `?facility=A&method=csv` with `X-Edit-At` and `/v3`
       `…/facilities/A/syncs` with body `{ "method": "csv", "editedAt": <the same time, ISO> }`.
       The report and the published `lastEditAt` are identical;
     - the legacy route without `?facility=` and `/v3` `…/days/day1/syncs`;
     - `/v1` `…/attendance/P1` and `/v3` `…/people/P1/attendance`;
     - `/v1` `{ mode }` and `/v3` `{ scoreEntry }`;
     - `/v1` reconciliations `?facility=A` and `/v3` body
       `{ "facility": "A" }`.
   - **Each new `GET`** returns the representation its `PUT` takes. After
     `PUT /v3/days/day1/visibility { isLive: false }`, the `GET` returns
     `isLive: false`; the same holds for live-push and score-entry.
   - **Idempotent `PUT`s.** `PUT` visibility twice gives the same state.
     `PUT` live-push twice gives `changed: false` the second time.
   - **Scoresheets.** `POST /v3/scoresheets` with
     `Accept: application/x-ndjson` streams; `application/pdf`, `*/*` or no
     `Accept` returns a PDF; `text/html` gives `406`,
     `code: "not_acceptable"`.
   - **Sessions and syncs.** `POST /v3/sessions` answers `201`.
     Both sync routes accept `X-Sync-Secret`, and reject
     `{ "method": "xml" }` and `{ "editedAt": "yesterday-ish" }` with
     `400`. A facility sync for an unknown facility answers `404`,
     `code: "unknown_facility"`, with the same `detail` text as the legacy
     route's `error`.
   - **Errors.** On `/v3`, `Content-Type: application/problem+json`, with
     `type`, `title`, `status`, `detail`, `instance`, `code` and `error`. A
     score conflict also carries `current`.
   - **404s.** An unknown day in the path (`/v3/days/zzz/visibility`), an
     unknown event, an unknown facility (attendance, score) and an unknown
     path (`GET /v3/nope`) all answer `404`. The same unknown day or facility
     on `/v1` still answers `400` with `{ error, code }`. `GET /nope` and
     `GET /v2/anything` answer the legacy-shaped `404`.
   - **405.** `GET …/matches/1/score` answers `405` with `Allow: PUT`, and
     `DELETE /v3/settings/live-push` with `Allow: GET, HEAD, PUT`.
   - **415.** `PUT /v3/settings/live-push` with `Content-Type: text/plain`
     answers `415`. A body-less `POST /v3/days/day1/syncs` is fine.
   - **401.** It carries `WWW-Authenticate` on every surface.
   - **Deprecation.** Every legacy and `/v1` twin carries
     `Deprecation: @<LEGACY_DEPRECATED_AT>` and a `Link` whose target is its
     `/v3` path with this request's parameters filled in. `/ping` carries
     neither, and has no `/v3` alias (`GET /v3/ping` → 404).
   - **`/v1` is frozen.** `attendance-routes.test.mjs` and
     `score-routes.test.mjs` keep testing `/v1` and pass **unchanged**,
     apart from the B9 headers if a test pins an exact header set.
2. **`test/unit/guards/rest-conventions.test.mjs`.** It walks every
   registered `/v3` route (with `registeredRoutes`) and checks N1–N4 and M2.
   - Every literal segment matches `/^[a-z]+(-[a-z]+)*$/` and is not in a
     verb list (`generate`, `create`, `update`, `delete`, `get`, `set`,
     `login`, `logout`, `run`, `do`, `send`).
   - Every parameter matches `/^:[a-z][a-zA-Z0-9]*$/`.
   - A segment directly before a parameter ends in `s`.
   - Every `PUT` path also has a `GET`.

   §4.11's exceptions table is its allow-list, written as data with the
   reason beside each entry. Also check that every `/v3` operation in the
   OpenAPI document has an `operationId` matching `/^[a-z]+[A-Z][A-Za-z]*$/`
   and documents a `4xx` response that refers to `Problem`. A last case
   checks that nothing is registered under `/v2`.
3. **Errors and middleware.**
   - Add `NotAcceptableError`, `MethodNotAllowedError` and
     `UnsupportedMediaTypeError` to `errors.mjs`, with their
     `errors.test.mjs` cases. (`UnknownFacilityError` exists from Phase 2;
     `V3_STATUS` is what makes it `404` on `/v3`.)
   - Extend `errorHandler` as in §4.7 (problem details on `/v3`, `V3_STATUS`,
     `Allow`) and add its unit cases, including that a `/v1` path gets the
     plain body and the 2.x status.
   - `server/requireJson.mjs` exports `requireJson({ optional = false } = {})`.
     It throws `UnsupportedMediaTypeError("Content-Type must be application/json")`
     when the request has a body (`req.headers["content-length"] > 0` or
     `transfer-encoding` is set) and `!req.is("application/json")`. When
     `optional` is false, a missing body is left for validation to reject
     with `400`.
   - `server/methodNotAllowed.mjs` exports `methodNotAllowed(req, _res, next)`.
     It reads `req.route.methods`, upper-cases the keys except `_all`, adds
     `HEAD` when `GET` is present, and calls
     `next(new MethodNotAllowedError(allow))`.
   - `server/deprecation.mjs` exports `deprecated(successorPath)`. It is
     middleware that sets `Deprecation: @${LEGACY_DEPRECATED_AT}` and
     `Link: <${successorPath(req)}>; rel="successor-version"`.
     `LEGACY_DEPRECATED_AT` is a constant in the same file.

   Each new module gets a unit test.
4. **Routes.** Each feature's `routes.mjs` exports one factory per surface.
   The handlers are written once, as functions in that file (or in
   `syncController`), and every factory calls the same ones. No handler logic
   is duplicated.
   - **Frozen factories**, unchanged apart from `deprecated(...)` in front of
     each handler: `syncRoutes` (`/sync`), `scoresheetsRoutes`
     (`/scoresheets`), `authRoutes` (`/auth`), and the two `/v1` factories,
     renamed `attendanceV1Routes` and `scoreV1Routes`. The `/v1` factories
     keep today's paths, `:key`, `{ mode }` and `?facility=`. Each
     `deprecated(...)` builds the `/v3` path from `req.params`.
   - **`/v3` factories.** Every route is declared with `router.route(path)`,
     chaining its methods and then `.all(methodNotAllowed)`, e.g.
     `router.route("/days/:day/visibility").get(...).put(requireJson(), ...).all(methodNotAllowed)`.
     - `syncV3Routes({ controller, auth })`: the two `syncs` routes, `visibility` (`GET` and
       `PUT`), `settings/live-push` (`GET` and `PUT`) and `diagnostics/sync`.
       `syncController` gains:
       - `syncDayV3`: builds the same `{ facilityName, method, editAt }`
         from `req.params.facility` (absent on the day route), `body.method`
         and `body.editedAt`, and calls the same service method;
       - `getDayVisibility`: `{ day, label, isLive }` from
         `configStore.get()`;
       - `getLivePush`: `{ enabled: snap.livePushOn, available }`, served
         from `SyncSettingsService.getLivePush()`.
     - `authV3Routes`: `POST /sessions`, sharing one `login(req)` helper with
       the legacy route, which answers `200` where `/v3` answers `201`.
     - `scoresheetsV3Routes`: one `POST /scoresheets` that negotiates
       `Accept` (§4.11), then calls today's `handleGenerate` or
       `handleGenerateStream`.
     - `scoreV3Routes`: the score `PUT`, scorer links, and score-entry `GET`
       and `PUT`. The `GET` is served by `ScoreService.getScoreEntryMode(event)`
       → `{ event, scoreEntry }`. The `PUT` reads `body.scoreEntry` through a
       `parseScoreEntry` beside today's `parseScoreEntryMode`, which `/v1`
       keeps using.
     - `attendanceV3Routes`: `…/people/:personKey/attendance` calling the
       same mark handler (it reads `req.params.personKey` where `/v1` reads
       `req.params.key`), desk links, and reconciliations reading
       `req.body?.facility`.

   `createRouters` mounts, in order: `/scoresheets`, `/sync`, `/auth`, `/v1`
   (`attendanceV1Routes`, `scoreV1Routes`), then `/v3` (all five `/v3`
   factories).
5. **404.** After every router and before `errorHandler`, add
   `app.use((req, _res, next) => next(new NotFoundError(\`No route for ${req.method} ${req.path}\`)))`.
   It is registered with `app.use`, not as a route, so `registeredRoutes`
   does not count it. Check that `registeredRoutes` (`test/helpers/routes.mjs`)
   skips the `_all` method that `.all(methodNotAllowed)` adds. If it does
   not, teach it to, as a test-only change. The route count in
   `test/unit/helpers/routes.test.mjs` becomes **31**: today's 15 plus the
   16 `/v3` routes.
6. **OpenAPI.**
   - Set `info.version` to the package version (it already reads it), and
     add `components.schemas.Problem` (§4.7).
   - Add an `@openapi` block for each `/v3` operation, following §4.11's
     documentation rule, with operationIds `runDaySync`, `runFacilitySync`, `getDayVisibility`,
     `updateDayVisibility`, `getLivePushSetting`, `updateLivePushSetting`,
     `getSyncDiagnostics`, `createScoresheet`, `createSession`,
     `getScoreEntry`, `updateScoreEntry`, `updatePersonAttendance`,
     `createDeskLink`, `createReconciliation`, `updateMatchScore` and
     `createScorerLink`. Each existing legacy and `/v1` operation keeps its
     operationId.
   - Mark every legacy and `/v1` operation that has a twin
     `deprecated: true`, with the description "Kept for Apps Script, archived
     pages and cached copies of the site; new clients use `<the /v3 path>`."
   - Document `POST /v3/scoresheets`'s two `200` media types and its `406`.
   - Update `openapiSpec.test.mjs` to the new path list: today's 14 paths
     plus the 13 `/v3` paths, **27** in all. Update `openapi.test.mjs`'s
     probe list, run `npm run test:snapshots` and review the diff.
7. **Route manifest.** Add a `test/helpers/routeManifest.mjs` line per `/v3`
   route, pointing at `test/integration/v3.test.mjs`.
8. **Docs.** Add a new page, `sage-docs/docs/technical/api.md`, covering:
   - §4.11's conventions, versioning rule, routes table and exceptions;
   - the frozen surfaces (legacy and `/v1`) with their `/v3` twins, and the
     rule "a new client uses `/v3`; the old URLs never go away";
   - both error shapes, with the code list from §4.7;
   - the auth each route takes.

   Link it from `technical/README.md` and `mkdocs.yml`, and from its features
   counterpart, `features/control-center.md`, if that page mentions the API.
   Root `CLAUDE.md`'s endpoint list moves to `/v3`, with `/v1` and the legacy
   routes listed as frozen twins, plus a one-line pointer: "new endpoints
   follow the REST conventions in `sage-docs/docs/technical/api.md`".

**Acceptance.**

- `v3.test.mjs` and `rest-conventions.test.mjs` are green.
- Every legacy and `/v1` test passes, with edits limited to the B9 headers if
  a test pins an exact header set.
- `/openapi.json` imports into Postman (**Import → Link**) with no validation
  warnings.
- `GET /ping` is unchanged, and its `X-App-Version` reads `3.0.0`.

### 6.6 Phase 6 — every client calls `/v3` (site repo and the master workbooks)

**Gate:** 3.0.0 is deployed. `GET /ping` on the Cloud Run URL shows
`X-App-Version` 3.0.0 or later. Do not edit the site before then. Work on a
branch of `sage-match-control.github.io` and merge when the owner says so,
never near an event. No `sage-tools-api` version changes.

**Mission Control and the rest of `tools/control-center.html`.** Find the
call sites with:

```bash
grep -n "CLOUD_RUN_BASE_URL}/sync\|CLOUD_RUN_BASE_URL}/auth\|CLOUD_RUN_BASE_URL}/v1" tools/control-center.html
```

| Today | Becomes |
|---|---|
| `` `${CLOUD_RUN_BASE_URL}/sync/${day.key}` + `?facility=…&method=…` `` (Resync, `POST`, no body) | `` `${CLOUD_RUN_BASE_URL}/v3/days/${encodeURIComponent(day.key)}/syncs` `` for the whole day, or `` …/facilities/${encodeURIComponent(facility)}/syncs `` for one facility; `POST`, headers gain `'Content-Type': 'application/json'`, and `method` moves into a body, `JSON.stringify({ method })`, sent only when set |
| `` `${CLOUD_RUN_BASE_URL}/sync/${day.key}/live` ``, `POST` | `` `${CLOUD_RUN_BASE_URL}/v3/days/${encodeURIComponent(day.key)}/visibility` ``, **`PUT`** |
| `` `${CLOUD_RUN_BASE_URL}/sync/config` `` (two places) | `` `${CLOUD_RUN_BASE_URL}/v3/diagnostics/sync` `` |
| `` `${CLOUD_RUN_BASE_URL}/sync/live-push` ``, `POST` | `` `${CLOUD_RUN_BASE_URL}/v3/settings/live-push` ``, **`PUT`** |
| `` `${CLOUD_RUN_BASE_URL}/auth/login` `` | `` `${CLOUD_RUN_BASE_URL}/v3/sessions` `` (it already checks `res.ok`, so `201` needs no change) |
| `` `${CLOUD_RUN_BASE_URL}/v1/events/…/score-entry` ``, body `{ mode }` | `` `${CLOUD_RUN_BASE_URL}/v3/events/…/score-entry` ``, body `{ scoreEntry: mode }` |
| `` `${CLOUD_RUN_BASE_URL}/v1/days/…/scores/scorer-links` `` | `` `${CLOUD_RUN_BASE_URL}/v3/days/…/scores/scorer-links` `` |
| `` `${CLOUD_RUN_BASE_URL}/v1/days/…/attendance/desk-links` `` | `` `${CLOUD_RUN_BASE_URL}/v3/days/…/attendance/desk-links` `` |
| `` `${CLOUD_RUN_BASE_URL}/v1/days/…/attendance/reconciliations` `` + `?facility=` | `` `${CLOUD_RUN_BASE_URL}/v3/days/…/attendance/reconciliations` ``, with `'Content-Type': 'application/json'` and body `{ facility }` when one is chosen (no body for the whole day) |
| the two comments naming `PUT /v1/events/:event/score-entry` | `PUT /v3/events/{event}/score-entry` |

**The shared blocks.** Each must stay byte-identical across its copies (root
`CLAUDE.md`), so change one copy, paste it into the others, and compare them
with `diff`.

- **`ATTENDANCE CLIENT`**, `markPerson`: the URL becomes
  `` `${apiBase}/v3/days/${encodeURIComponent(day)}/facilities/${encodeURIComponent(facility)}/people/${encodeURIComponent(key)}/attendance` ``.
  The copies are `tools/control-center.html`,
  `_templates/attendance/attendance.html`,
  `events/piggleball-2026/attendance.html` and
  `events/pickledrive-anniversary-2026/attendance.html`. The two event copies
  belong to finished events. They change too, because the block must stay
  identical, and `/v3` serves them as well as `/v1` did.
- **`SCORE CLIENT`**: `/v1/days/` becomes `/v3/days/` in the score URL. The
  copies are `tools/control-center.html` and `_templates/scorer/scorer.html`;
  no event has a `scorer.html` yet. Untracked `scorer-beta.html` scratch
  copies are not part of the rule.

Nothing else in either block changes. Their error handling reads
`body.error`, `body.current` and `res.status` (`409`, `401`, `403`), all of
which `/v3` keeps. The `404` that `/v3` gives for an unknown facility only
changes which message a caller shows for a request a correct page never
sends.

**The rest:**

- **Check connection** reports a service without `live.switch` as "older than
  2.5.0". Make a `404` from `/v3/diagnostics/sync` read as "older than 3.0.0"
  instead.
- In `tools/scoresheet-generator.html`, `API_URL` becomes
  `…/v3/scoresheets`, and its request adds the header
  `Accept: application/x-ndjson`.
- Response handling stays as it is everywhere: every success body is
  unchanged, and error bodies still carry `error`.

**Apps Script: the masters' `sheets-sync.gs` moves to `/v3`.** A workbook
made from a master gets a copy of the master's bound script. So updating the
script in the masters moves every **future** workbook to `/v3`. Workbooks that
already exist keep their own copy, and with it the legacy URLs, which keep
working. Nobody re-pastes into an existing workbook.

1. **Edit `apps-script/sheets-sync.gs`** in `sage-tools-api`. It is not part
   of the Cloud Run service, so there is no version bump; the commit message
   says "sheets-sync.gs: call /v3 (masters only)".
   - **`triggerSync_`'s sync** (today's `POST /sync/${config.dayKey}?facility=…` with
     `X-Edit-At`) becomes
     `POST ${CLOUD_RUN_BASE_URL}/v3/days/${encodeURIComponent(dayKey)}/facilities/${encodeURIComponent(facilityName)}/syncs`.
     It sends `contentType: 'application/json'` and
     `payload: JSON.stringify(editAt > 0 ? { editedAt: new Date(editAt).toISOString() } : {})`,
     with `X-Sync-Secret` unchanged and **no** `X-Edit-At`.
   - **Log lines (`triggerSync_`).** On failure, the log reads
     `Sync failed (${status} ${code}): ${detail}` when the body parses as
     JSON with `detail`, and today's `Sync failed (${status}): ${body}`
     otherwise. A shared helper `describeFailure_(status, body)` builds the
     text for both this line and setup's messages. On success the log line
     stays `Sync OK (${status}): ${body}`.
   - **`checkSecretAndDayKey_`** calls `GET ${CLOUD_RUN_BASE_URL}/v3/diagnostics/sync`.
     The `401` check and the `days` list read stay as they are; the body is
     the same. The two `GET /sync/config` texts in its warnings name the new
     path.
   - **`runTestSync_`** uses the same facility-scoped `POST` as the sync,
     with an empty JSON body. "Unknown facility" is detected by
     `status === 404 && parsed.code === 'unknown_facility'` instead of
     `body.includes('unknown facility')`, and the message shown is
     `parsed.detail`. Other non-2xx answers stay "unreachable", with the
     new path in the warning.
   - **The header comment** says the script calls `/v3`, and that workbooks
     made before 3.0.0 hold an older copy that calls `/sync/...`, which keeps
     working.
   - **No top-level name is added** that `sheet-generator.gs`,
     `standard-generator.gs` or `attendance.gs` declares: they share a
     script project. `npm run test:appscript` catches a clash for the
     generators and `attendance.gs`.
2. **`apps-script/verify-sheets-sync.mjs`** (new), the harness for these
   calls. Add `UrlFetchApp` (recording each call and answering from a
   per-test table) and `PropertiesService` (an in-memory store) to
   `apps-script/mock-apps-script.mjs` if they are not there, then load
   `sheets-sync.gs` and assert:
   - the sync's method, URL (day and facility encoded), headers (secret, no
     `X-Edit-At`), content type and body (with and without an edit time);
   - the setup check's URL and its three outcomes (`401` → blocking, `200`
     with and without the day key, `503` → unreachable);
   - the test sync's `404 unknown_facility` → blocking with `detail`, and
     `502` → unreachable;
   - both shapes of the failure log line.

   Add the script to `scripts/run-appscript-verifies.mjs`, so
   `npm run test:appscript` and `npm run verify` run it.
3. **Paste into each master.** For each master workbook:
   - open **Extensions → Apps Script**, select the `sheets-sync.gs` file
     (named `sheets-sync` or similar in the editor), and replace its whole contents with the
     repo copy;
   - save, then reload the workbook and check that the **SAGE** menu
     appears.

   Do not run **Set up live sync** in a master: it has no day key. The
   masters are:
   - **SAGE Dual Meet Master**, for `"dual-meet"` events;
   - **SAGE Standard Tournament Master**, for `"standard"` events.

   Their Drive file IDs are in `GENERATOR_MASTERS` in
   `tools/tournament-calculator.html`.

   There is no team master yet. Until one exists, a team event's workbook is
   copied from the previous team workbook, so paste the new script into the
   next team workbook by hand once it is made. The
   [Team Tournament Master spec](../not-started/team-tournament-master-spec.md) carries the
   `/v3` script from the start.
4. **Check it.** Make a fresh copy of each master the way an operator does
   (the calculator's generator handoff), on a test day key registered in
   `events.json`. Then:
   - run **SAGE → Set up live sync** and confirm the secret check and the
     test sync pass;
   - edit a watched cell and confirm the execution log shows
     `Sync OK (200)` and Cloud Run's request log shows
     `POST /v3/days/<day>/facilities/<facility>/syncs`;
   - confirm the published snapshot's `lastEditAt` matches the edit;
   - set up with a wrong facility name and confirm the blocking message
     names the valid facilities.

   Delete the test copy afterwards and remove the test day key from
   `events.json`.

Do not touch `events/archives/`, and do not re-paste the script into an
existing event workbook.

**Acceptance.**

- Open Control Center locally against production (`?fixture=` is not enough,
  because these calls go to Cloud Run). Sign in, press **Check connection**,
  run **Resync this day now** on a day that is not live, toggle
  **Force hidden** and back on a day that is not live, and flip
  **Sync method** off and on.
- Generate one scoresheet PDF in the scoresheet generator.
- On an event whose workbooks are safe to write to (a test event, or a
  finished one), do each of the following:
  - in Mission Control, flip **Scorer links** both ways, and issue a scorer
    link and a desk link;
  - run **Update roster**;
  - mark and unmark one person, from Control Center's Attendance tab and
    from the desk page;
  - save and then clear one score, from Match Finder and from the scorer
    template opened with the scorer link.
- Each works, and Cloud Run's request log shows only `/v3` paths for these
  actions.
- The `diff` of every copy of each shared block shows no difference.
- `grep -rn "/sync/\|/auth/login\|/scoresheets/generate\|/v1/" tools/ _templates/ events/ --include=*.html | grep -v "events/archives/\|-beta.html"`
  finds only comments.

### 6.7 Phase 7 — build trigger, docs and workspace

1. **Cloud Build trigger (F18).** The owner runs this once. Document it in
   `sage-docs/docs/technical/deployment.md`:

   ```bash
   gcloud builds triggers list --project=sage-tools-api
   ```

   ```bash
   gcloud builds triggers update github <TRIGGER_NAME> --project=sage-tools-api --ignored-files="apps-script/**,live-worker/**,spikes/**,test/**,scripts/**,.githooks/**,**/*.md,.env.example,jsconfig.json"
   ```

   To verify, push a README-only commit to a branch the trigger watches and
   confirm no build starts. `package.json`, `package-lock.json`, `index.mjs`,
   `src/**`, `templates/**` and the `Dockerfile` still trigger a build.
2. **Workspace `CLAUDE.md` (F19).** Run `git init` in `D:\Personal\SAGE`,
   with a `.gitignore` listing `sage-tools-api/`,
   `sage-match-control.github.io/`, `sage-docs/`, `event-data/` and
   `.claude/`. Commit only `CLAUDE.md`. The owner creates the GitHub repo
   (`sage-match-control/sage-workspace`) and pushes. The file stays at the
   same path, so Claude Code keeps finding it.
3. **Root `CLAUDE.md` rules.** Add:
   - the dependency rules of §4.3, in one short table;
   - "no production change without a failing test first";
   - the API rule: a new endpoint goes under the current version (`/v3`)
     and follows the REST conventions in `sage-docs/docs/technical/api.md`,
     or adds a reasoned row to its exceptions table and to
     `rest-conventions.test.mjs`. An old URL is never removed. A breaking
     change is a new major version **and** a new URL surface together
     (4.0.0 with `/v4`), so the package major and the URL version stay
     equal;
   - that `apps-script/` is not part of the service;
   - that `src/app.mjs` is the only place objects are constructed;
   - that `src/config/loadConfig.mjs` is the only reader of `process.env`, so
     a new environment variable is added there, in `.env.example` and in its
     test.
4. **Docs** (§7), for anything an earlier phase left.

**Acceptance.**

- A markdown-only push does not rebuild Cloud Run.
- The root `CLAUDE.md` is tracked.
- Phase 1's grep still returns nothing.

### 6.8 Phase 8 — the site's duplicated code (decision only)

The site repeats code by hand:

- the `LIVE CHANNEL` block in 12 files;
- the `ATTENDANCE CLIENT` and `SCORE CLIENT` blocks in 4 each;
- the played/BYE/series rules in `facilityCompletion.mjs`, Control Center,
  the templates' `index.html`/`schedule.html` and the scorer template;
- the team-event rules and team rosters in 2 files each.

This is the largest maintenance risk in the system, but fixing it changes the
site's "one self-contained file per page" rule. That is a decision for the
owner, not an implementation detail.

This phase produces a one-page decision record in
`sage-docs/docs/specs/not-started/`, comparing these three options, and
nothing else:

1. **Shared files** at root-absolute `/assets/js/*.js`, loaded with
   `<script src>`. No build step, and no byte-identical copies to keep in
   step. Costs: a page is no longer one file; `tools/sw.js` caching and
   cache-busting (`?v=`) need care; an archived page that keeps loading a
   shared file can break when that file changes.
2. **A generation script** that stamps each shared block into its pages.
   Pages stay self-contained. Costs: a script to run and a diff check to
   keep.
3. **Keep the copies** and rely on the
   [site test suite](../not-started/site-test-suite-spec.md)'s consistency and parity checks
   to catch drift. Nothing changes in the pages. Costs: the copies still have
   to be edited by hand, and that spec has to be built first.

Recommendation to evaluate: option 1 for the live channel, the attendance
client and the score client, leaving archived events untouched; option 3 for
the rule copies that also live in `sage-tools-api`.

---

## 7. Documentation to update

Write in the present tense, in the same commit as the phase that changes the
thing.

| File | Change | Phase |
|---|---|---|
| `sage-tools-api/README.md` | Changelog entry per versioned phase; layout; the "Bound Apps Script" section's path; Testing section (the new coverage scripts, the dependency guard, `makeSyncService`, and that the two verify scripts are gone) | 1–5 |
| root `CLAUDE.md` | the `sage-tools-api` layout (`apps-script/`, `src/clients/`, `src/registry/`, the `domain/` folders, `src/app.mjs`, `src/config/`), the removed verify commands, the moved paths in "Things that must be kept in sync by hand", the endpoint list (the `/v3` twins, the 404 body) | 1, 2, 5, 7 |
| `sage-match-control.github.io/_templates/CLAUDE.md` | the `sheets-sync.gs` path | 1 |
| `event-data/config/README.md` | the `SyncConfigStore` path | 1 |
| `sage-docs/docs/technical/architecture.md` | the pattern (§4.0), the layout and dependency rules (§4.1–4.3), the port table (§4.4), `createApp`, and the SOLID mapping (§4.12) | 2, 4 |
| `sage-docs/docs/technical/sync-pipeline.md` | the ports, the stores, `mergeIntoStore`, the two deliveries and `SnapshotPublisher` replace the `SyncService` description; file paths | 1, 4 |
| `sage-docs/docs/technical/auth.md` | `src/auth/middleware.mjs` and its four checks | 2 |
| `sage-docs/docs/technical/deployment.md` | the startup validation and its error; the Cloud Build file filter; the test commands | 2, 7 |
| `sage-docs/docs/technical/api.md` (new) | both API surfaces, the alias rule, error codes | 5 |
| `sage-docs/docs/technical/control-center.md`, `event-attendance.md`, `scorer-page.md`, `auth.md` | the `/v3` paths each page names, with `/v1` mentioned once as the frozen twin | 6 |
| `sage-match-control.github.io/_templates/CLAUDE.md`, root `CLAUDE.md`'s site section | any `/v1` path a page calls becomes its `/v3` path | 6 |
| `sage-docs/docs/technical/sync-pipeline.md`, `architecture.md`, `deployment.md`, `event-data-config.md`; root `CLAUDE.md`'s data-flow diagram and `sheets-sync.gs` notes | Apps Script in a workbook made from a master calls `POST /v3/days/{day}/facilities/{facility}/syncs` with `editedAt`, and `GET /v3/diagnostics/sync` at setup. Older workbooks call the legacy `POST /sync/:day` and `GET /sync/config`, which keep working | 6 |
| `sage-docs/docs/technical/README.md`, `mkdocs.yml` | index and nav for `api.md` | 5 |
| `sage-docs/docs/specs/README.md`, folder READMEs, `mkdocs.yml` | this spec's status as it moves (`not-started` → `in-progress` at Phase 1, → `implemented` after Phase 7) | 1, 7 |
| every file the Phase 1 grep finds | new paths | 1 |

---

## 8. Acceptance checklist

**Phase 1**

- [ ] `apps-script/` holds the `.gs` files, their harness, fixtures and
      `jsconfig.json`; `scripts/` holds only `hash-password.mjs` and the
      runner.
- [ ] `src/clients/` holds the six clients; `src/registry/` holds the
      registry; each feature's pure modules are in its `domain/`;
      `parseCsv` is in `src/shared/csv.mjs`.
- [ ] Coverage thresholds exist for `clients`, `registry`, `attendance` and
      `scores`.
- [ ] The Phase 1 grep returns nothing; `npm run verify` is green; the service
      starts.

**Phase 2**

- [ ] `X-Sync-Secret` is compared in constant time.
- [ ] A bad environment value stops startup with a clear message; an unset
      secret does not; an empty value is unset.
- [ ] Every error response is `{ error, code }` (plus `current` on a score
      conflict); no handler builds its own error response; a malformed JSON
      body gets JSON.
- [ ] Every auth check lives in `src/auth/middleware.mjs`.
- [ ] `index.mjs` and `buildApp` both use `createApp`; `Server` mounts the
      routers `createRouters` gives it and imports no feature.
- [ ] `src/shared/ports.mjs` declares the ports of §4.4.
- [ ] CORS allows `PUT`, `PATCH` and `DELETE`.
- [ ] Day keys `config` and `live-push` are rejected.
- [ ] `jsconfig.json` covers `.mjs`; the new modules pass `// @ts-check`.

**Phase 3**

- [ ] The conflict retry is written once (`withConflictRetry`).
- [ ] The three registry setters share `#update`, and three conflicts answer
      `409`; validation and the switch edits are pure modules in
      `registry/domain/`.
- [ ] The store interface, `mergeIntoStore` and the extracted domain modules
      exist, with tests.

**Phase 4**

- [ ] `SyncService` has no GitHub, Worker, archive or fallback code;
      `syncDay`'s signature and result are unchanged.
- [ ] `GitHubOnlyDelivery`, `LiveFirstDelivery`, `SnapshotPublisher`,
      `DayVisibilityService`, `SyncSettingsService` and the controller exist;
      `SyncService` takes a `fetchers` map.
- [ ] Both snapshot stores pass the shared contract test.
- [ ] Every row of §4.12 (SOLID) holds.
- [ ] Three GitHub conflicts on a sync answer `409`; no `CHARACTERIZATION B`
      marker is left.
- [ ] The dependency guard is green.

**Phase 5**

- [ ] `package.json` is 3.0.0, and `GET /ping` reports it.
- [ ] All sixteen `/v3` routes exist (a twin of every legacy and `/v1`
      route, plus three `GET`s), and the parity tests are green.
- [ ] Every `/v3` route follows §4.11 or is in its exceptions table;
      `rest-conventions.test.mjs` is green.
- [ ] `/v3` errors are RFC 9457 problem details that still carry `error`
      and `code`; `405` carries `Allow`; `415` guards JSON bodies; `401`
      carries `WWW-Authenticate`.
- [ ] Every legacy and `/v1` URL answers as before (plus the B9 headers),
      `/ping` is untouched, and nothing answers under `/v2`.
- [ ] Unknown paths answer JSON `404`.
- [ ] OpenAPI documents all twenty-seven paths, with every legacy and `/v1`
      twin marked deprecated.

**Phase 6**

- [ ] Control Center, the scoresheet generator and every copy of the
      `ATTENDANCE CLIENT` and `SCORE CLIENT` blocks call only `/v3` routes,
      the copies are identical, and each action was exercised against
      production.
- [ ] Both masters carry the `/v3` `sheets-sync.gs`;
      `verify-sheets-sync.mjs` is green; a fresh copy of each master set up
      and synced through `/v3`.

**Phase 7**

- [ ] A markdown-only push does not rebuild Cloud Run.
- [ ] The workspace `CLAUDE.md` is tracked and carries the new rules.
- [ ] Docs describe the new structure.

**Always**

- [ ] `GET /ping` returns `200 PONG!` with `X-App-Version` and
      `X-Sync-Config`.
- [ ] Nothing was deployed before the owner said so, and nothing near an
      event.

---

## 9. Rollout and rollback

- Everything happens on `arch-refactor`. **Nothing merges to `main` until the
  owner agrees**, and never during or in the few days before an event,
  because merging deploys Cloud Run.
- Merge phase by phase if the owner prefers. Each phase is independently
  green and independently revertible with `git revert <phase commit>`.
  Phase 6 is a separate branch in the site repo, and merges only after 3.0.0
  is deployed.
- After each API deploy:
  - check that `GET /ping` shows the new version;
  - press **Sync now** in a workbook, and run **SAGE → Set up live sync** in
    a copy of a master. Both are Apps Script, on the legacy URLs; setup also
    exercises `GET /sync/config`;
  - in Control Center, press **Check connection**, run one
    **Resync this day now**, and save one score from Match Finder on an
    event with `scoreEntry` set (score entry calls `syncDay`);
  - mark one person on an attendance desk page.
- After deploying 2.8.2, also check the Cloud Run logs for the one-line
  `unsetWarning` and confirm it names nothing unexpected.
- Rollback is a revert and a push. No URL a client uses is removed, so no
  client needs touching. Revert Phase 6 before reverting Phase 5.

## 10. Out of scope

- Re-pasting `sheets-sync.gs` into workbooks that already exist, and moving
  the archived pages (`events/archives/`) to new URLs. Both keep the legacy
  URLs, which never go away.
- Tests for the live Worker beyond `smoke.mjs`.
- Replacing the shared-secret and one-operator-login model.
- Merging the two Google Sheets readers (`SheetsCsvFetcher` with an API key,
  `SheetsClient` with an API key or service account) into one. They now sit
  side by side in `src/clients/`, which makes that a later, separate
  decision.
- Rate limiting, request IDs, structured (JSON) logging, metrics.
- REST features no client needs yet: `ETag`/`If-Match` concurrency (score
  entry's body precondition stays, §4.11), pagination and filtering of
  collections (no `/v3` route lists anything), hypermedia links in bodies,
  `Retry-After` on `503`, and a `/v2`.
- The site's shared code (Phase 8 is a decision only).
