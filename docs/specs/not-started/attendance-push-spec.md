# Spec — Push for attendance

When someone is marked in at one desk, every other desk and Control Center's
Attendance tab show it within a second or two, instead of within about 20 seconds.
Pages poll a third as often while it works.

The live Worker pushes a **nudge**, a message that says only *this facility's list
changed*. It carries no names and no check-in state, so it can travel on a public
socket. Each page that gets one re-reads the list from the API with its own token, as
it does today. The API's reads from Google stay as they are.

> **Status: not started.** Written 2026-10-12 against `sage-tools-api` 3.3.0, the live
> Worker `sage-live` and `sage-match-control.github.io` at `lib/v1`, as of that date.
> **Every decision is settled (§1).** There are no open questions: do not stop to ask
> about anything §1 covers.
>
> Builds on [Live push delivery](../implemented/durable-object-push-spec.md) and
> [Attendance for every event](../implemented/multi-event-attendance-spec.md), whose D4
> kept attendance off the live channel because the channel is public. This spec keeps
> that rule: nothing private goes over the socket. Read their §0 if a term here is
> unfamiliar.

---

## 0. Read this first

You are implementing this with no other context. Everything you need is in this page
and the files it names. Read §0 to §2 in full before writing code, then do §3 to §6 in
order. §10 is what to report when you finish.

### 0.1 The system in one paragraph

Each venue on each day has a Google Sheet (a *facility workbook*) with an `ATTENDANCE`
tab. `sage-tools-api` (Node/Express on Cloud Run) reads and writes that tab as its
service account. Desk staff open a **desk link** on a phone (the desk page,
`events/<key>/attendance.html`) and operators use Control Center's **Attendance** tab.
Both render `lib/v1/views/attendance-view.js`, which reads one facility's people with
`GET /v3/days/{day}/facilities/{facility}/people` every 10 s while the page is visible,
and marks someone with `PUT …/people/{personKey}/attendance`. The API caches the list
for 10 s per workbook, per instance. Separately, the Cloudflare Worker `sage-live`
holds one Durable Object per `<event>/<day>` that stores the day's public snapshot and
pushes it to every open page over `wss://…/live/<event>/<day>`. Cloud Run publishes to
it with a shared secret.

### 0.2 Where things are

`D:\Personal\SAGE` is a plain folder holding four git repos:

| Path | What you change there |
|---|---|
| `sage-tools-api/live-worker/` | §3: two routes on the Worker, the `DayChannel` object, `smoke.mjs`, `README.md` |
| `sage-tools-api/` | §4: `LivePublisher.nudge`, `AttendanceService` sends nudges, a port, wiring, tests, version, Changelog |
| `sage-match-control.github.io/` | §5: a new `lib/v1/data/attendance-signal.js`, `attendance-view.js`, `attendance-desk.js`, `tools/control-center.html`, the attendance template, `_tests/` |
| `sage-docs/` | §7: feature and technical pages |
| `D:\Personal\SAGE\CLAUDE.md` | §7: three sentences (not in any repo) |

`event-data/`, every `apps-script/*.gs` file and the finished events' desk pages are
**not** touched (§8).

### 0.3 Hard rules

1. **Leave every change uncommitted.** No `git add`, `commit`, `push`, branch, switch or
   stash. The owner reviews, commits and deploys.
2. **Never deploy the Worker** (`npx wrangler deploy`), never call the production API or
   the production Worker, and never touch the real `events.json`. Run the Worker with
   `npx wrangler dev`; tests use fakes. Deploys are the owner's (§10.2).
3. **Write each test first and watch it fail**, then make it pass. **Never weaken or
   delete a test.** No existing assertion should need to change; if one does, stop and
   report it.
4. `sage-tools-api`: ESM `.mjs`, classes with constructor injection, **no new
   dependency**, no TypeScript, 4-space indent, double quotes, comments that say why.
   Obey the dependency rules in `D:\Personal\SAGE\CLAUDE.md` (enforced by
   `test/unit/guards/dependency-rules.test.mjs`): `AttendanceService` reaches the Worker
   only through the port injected in `src/app.mjs`.
5. The Worker: plain JS, the style of `live-worker/src/index.js`. **No new Durable Object
   class**, so no new entry in `wrangler.jsonc`'s `migrations` (§1 D2).
6. Site: no new dependency, 2-space indent, single quotes, match the surrounding style.
   **`lib/v1` changes must be compatible** (`_templates/CLAUDE.md` §8): add, never remove
   or rename. A browser can hold a 10-minute-old copy of one file beside a new copy of
   another, so the site change ships in **three pushes** (§1 D10). Write the code so each
   push is safe on its own.
7. Find functions **by name** (`grep -n "function name("`), not by line number.
8. Line endings: keep whatever each file has today (the files named here are LF).
9. Cite this spec from code comments as
   `sage-docs/docs/specs/.../attendance-push-spec.md`, with literally `...`.
10. Documentation is present tense: what the system does, not what changed.
11. If the code contradicts this spec in a way that blocks you, stop and report it
    (§10). Don't work around it.

### 0.4 Facts checked against the code (2026-10-12)

| Fact | Where |
|---|---|
| The Worker routes `/live/<event>/<day>`, `/snapshot/…` and `/publish/…` (one `ROUTE` regex); the object is `idFromName(\`${event}/${day}\`)` with `locationHint: "apac-se"`. `live` needs `Upgrade: websocket` and an allowed `Origin` (`originAllowed`); `snapshot` and `publish` need `X-Publish-Secret` (`secretOk`, constant-time) | `live-worker/src/index.js`, `fetch` |
| `DayChannel.fetch` dispatches on the path's first segment. `#connect` accepts a hibernatable socket and sends `{type:"snapshot",…}` or `{type:"empty"}` at once. `#publish` broadcasts to `this.ctx.getWebSockets()`. The constructor sets a `ping`→`pong` auto-response | `live-worker/src/index.js`, `DayChannel` |
| `wrangler.jsonc` binds `DAY_CHANNEL` to `DayChannel`, migration tag `v1` | `live-worker/wrangler.jsonc` |
| Every WebSocket client ignores a message whose `type` isn't `snapshot` (`if(msg.type !== 'snapshot' …) return;`) in `lib/v1/data/live-channel.js` and in the inline copies in `events/pickledrive-anniversary-2026/{index,schedule}.html` and `events/piggleball-2026/{index,schedule}.html` | those files |
| `live-channel.js` exports `LIVE_PING_MS` (50000), `LIVE_PONG_TIMEOUT_MS` (10000), `LIVE_SAFETY_POLL_MS`, `LIVE_RETRY_MAX_MS` (30000) and `createLiveChannel` | `lib/v1/data/live-channel.js` |
| `LivePublisher` has `enabled` (URL and secret both set), `read`, `publish`, and a private `#request(method, kind, event, day, body)` that sends `X-Publish-Secret` and times out at `timeoutMs` (4000) | `src/clients/LivePublisher.mjs` |
| `config.livePushOn` is `raw.livePush !== false` (the **Sync method** switch); `SnapshotPublisher.liveActive(config)` is `livePublisher.enabled && config.livePushOn` | `src/registry/SyncConfigSnapshot.mjs`, `src/sync/publishing/SnapshotPublisher.mjs` |
| `AttendanceService.mark` writes only when `planMark(…).changed`, then `this.peopleCache.delete(facility.sheetId)`. `#reconcileFacility` deletes the entry after `createAttendanceTab` and after `updateValues` with rows. `listPeople` → `#cachedPeople` reuses a read for `PEOPLE_CACHE_MS` (10 000) | `src/attendance/AttendanceService.mjs` |
| `src/app.mjs` constructs one `LivePublisher` (`livePublisher`) and `new AttendanceService({ configStore, sheets, authService, logger })`; `test/helpers/fakes.mjs` has `FakeLivePublisher({ enabled, snapshot })` | `src/app.mjs`, `test/helpers/fakes.mjs` |
| `createAttendanceView(opts)` polls with `setInterval(() => { if(!document.hidden) load(true); }, opts.pollMs)` and loads on `visibilitychange`; it returns `{ refresh, setShowWithdrawn, destroy }`. `load(quiet)` fetches every facility in console mode and the chosen venue in desk mode | `lib/v1/views/attendance-view.js` |
| `mountAttendanceDesk(settings)` accepts only `KNOWN_SETTINGS = ['eventKey']`; any other name logs `unknown setting` and **returns without rendering** | `lib/v1/apps/attendance-desk.js` |
| The attendance template's shell has no `LIVE_BASE_URL`; Control Center does (`'wss://sage-live.sagematchcontrol.workers.dev'`) and already runs `createLiveChannel` for the day's snapshot | `_templates/attendance/attendance.html`, `tools/control-center.html` |
| The finished events' desk pages (Pickle for Sight, PickleDrive, Piggleball) are inline copies that do not use the engine | `events/*/attendance.html` |

---

## 1. Decisions (settled with the owner, 2026-10-12)

| # | Decision |
|---|---|
| D1 | **The nudge carries no data.** The message is exactly `{"type":"attendance","facility":"<name>"}`. No person, key, name, state, time or count. A facility's name is already public in `events.json`. Pages re-read the list from the API with their own token |
| D2 | **Its own object, not the day's snapshot channel.** The Worker gives attendance a separate object, `idFromName(\`attendance:${event}/${day}\`)`, of the **same** `DayChannel` class (no migration). It stores nothing. Public viewers never receive a nudge, a desk never receives a snapshot, and a fault here can't reach the public feed. Event and day keys match `[a-z0-9-]`, so the `attendance:` prefix can't collide with a snapshot object's name |
| D3 | **Two Worker routes.** `GET /attendance/<event>/<day>` is the socket: `Upgrade: websocket`, the same `Origin` rule as `/live`, no secret, nothing sent on connect. `POST /nudge/<event>/<day>` with body `{ facility }` needs `X-Publish-Secret`, broadcasts the D1 message to that object's sockets and answers `200 {clients}`. A `facility` that isn't a 1–100 character string is `400` |
| D4 | **The API nudges after every write that changes the tab**, and only then: a mark whose plan `changed`, a reconcile that created the tab, and a reconcile that wrote rows. One nudge per write, after the write and the cache drop |
| D5 | **Best effort, awaited, short.** The nudge is awaited before the response, because Cloud Run throttles CPU once a response is sent, with a **2-second** timeout. A failure or timeout logs one warning and never changes the answer or throws |
| D6 | **The same switches as live push.** The API nudges only when `LivePublisher.enabled` (URL and secret set) **and** `config.livePushOn` (the **Sync method** switch is not *GitHub only*). Otherwise it sends nothing, and pages fall back to their poll (D8) |
| D7 | **Pages coalesce.** A nudge for a facility the page shows re-reads that facility only, at most once every 2 s (trailing). **And once more 11 s after the last nudge**, because another Cloud Run instance can hold a list up to 10 s old (`PEOPLE_CACHE_MS`), so the follow-up is never stale. Nudges for a facility the page doesn't show are ignored. While the page is hidden, nudges are ignored; the existing `visibilitychange` load catches up |
| D8 | **Polling stays as the fallback, slower while connected.** With the socket closed, the page polls every 10 s as today. With it open, every 30 s, which covers a nudge path that has quietly stopped (the API's switch off, a failed nudge) |
| D9 | **Where it runs.** The desk page (desks mode, from the template) and Control Center's Attendance tab. Off in fixture mode, and off when the page has no Worker address. The finished events' inline desk pages are untouched and keep polling |
| D10 | **Rollout order.** (1) The Worker, deployed by hand, never within 3 days of an event. (2) The API: harmless before any page listens. (3) Site push A: `attendance-signal.js` (new) and `attendance-view.js`. (4) Site push B, at least 10 minutes after A: `attendance-desk.js` and `tools/control-center.html` start using them. (5) Site push C, at least 10 minutes after B: the attendance template passes the Worker address. A cached older `attendance-desk.js` refuses an unknown setting and renders nothing, which is why C waits |
| D11 | **No read changes.** `GET …/people`, its 10-second cache and its response stay exactly as they are. Google reads per facility don't go up |
| D12 | **Versions.** `sage-tools-api` **3.4.0** (minor: new behaviour, no new route). The Worker has no version; its `README.md` route table is its record |

---

## 2. Overview

```
desk A marks someone
   PUT …/people/{personKey}/attendance  ──>  sage-tools-api  ──>  ATTENDANCE tab (write)
                                                  │  drop the list cache
                                                  │  POST /nudge/<event>/<day> {facility}   (2 s, best effort)
                                                  v
                                      sage-live: object "attendance:<event>/<day>"
                                                  │  {"type":"attendance","facility":"Gym"}
                         ┌────────────────────────┼─────────────────────────┐
                         v                        v                         v
                      desk A                   desk B                 Control Center
               (re-reads Gym:            (re-reads Gym now,        (re-reads Gym now,
                sees its own mark)        and again in 11 s)        and again in 11 s)
                         └──── GET …/facilities/Gym/people with its own token ───┘
```

The cost is easy to see:

- **Google reads:** unchanged. Every re-read goes through the existing 10 s cache.
- **API requests:** fewer when quiet, since pages poll every 30 s instead of 10 s. At
  most one re-read per 2 s per page during a burst of marks, plus one follow-up.
- **Worker:** one nudge per write that changed the tab. Each nudge goes to that day's
  attendance sockets only.

---

## 3. The Worker (`sage-tools-api/live-worker/`)

### 3.1 `src/index.js`, `fetch`

- Widen `ROUTE` to `^/(live|snapshot|publish|attendance|nudge)/(KEY)/(KEY)$`.
- Name the object by kind: `attendance` and `nudge` use
  `` `attendance:${event}/${day}` ``, the rest `` `${event}/${day}` `` as now. Same
  `locationHint`.
- `attendance`: the same checks as `live` (`426` without the upgrade header, `403` for a
  disallowed `Origin`), then `stub.fetch(request)`.
- `nudge`: the same secret check as `publish` (`401`), `POST` only (else `405`), then
  `stub.fetch(request)`.

### 3.2 `DayChannel.fetch`

- `attendance` → a new `#listen()`: like `#connect` (create the pair,
  `this.ctx.acceptWebSocket(server)`, answer `101`) but **send nothing**.
- `nudge` → a new `#nudge(request)`. Parse the JSON body (`400 bad json` on failure).
  `facility` must be a string of 1–100 characters, else `400`. Build
  `JSON.stringify({ type: "attendance", facility })` and send it to every socket, as
  `#publish` does, ignoring a socket that throws. Answer `Response.json({ clients })`.
  Write nothing to storage.

`#connect`, `#read` and `#publish` are unchanged. Comment the two new methods with this
spec's path.

### 3.3 `smoke.mjs`

Add these checks, on the same throwaway keys it already uses:

1. Open `/attendance/<key>/<day>` and `/live/<key>/<day>`.
2. `POST /nudge/<key>/<day>` `{ "facility": "Smoke Court" }` answers `200` with
   `clients ≥ 1`.
3. The attendance socket receives exactly `{"type":"attendance","facility":"Smoke Court"}`.
4. The live socket receives nothing within 2 s.
5. `POST /nudge/…` without the secret is `401`. With `{ "facility": "" }` it is `400`.
6. Attendance sockets connect with nothing sent first, and a `ping` gets `pong`.

### 3.4 `README.md`

Add the two routes to the route table, with the rules in D1–D3, in the README's own
style.

---

## 4. `sage-tools-api`

### 4.1 `LivePublisher.nudge` (`src/clients/LivePublisher.mjs`)

```js
// One data-free nudge: "this facility's attendance list changed"
// (sage-docs/docs/specs/.../attendance-push-spec.md). -> { clients }.
// Throws on network errors, a 2-second timeout, 401, 400 and 5xx.
async nudge(event, day, facility) { … }
```

`POST {baseUrl}/nudge/{event}/{day}` with `X-Publish-Secret`, `Content-Type:
application/json` and the body `{ facility }`. It times out after **2000 ms**, not
`timeoutMs`. Give `#request` an optional timeout argument rather than duplicating it.
The timeout message reads `live Worker nudge timed out after 2000ms`, and failures read
`live Worker nudge failed: …`, in the shape `read` and `publish` use.

### 4.2 The port (`src/shared/ports.mjs`)

```js
/**
 * What AttendanceService needs to tell open pages a list changed
 * (sage-docs/docs/specs/.../attendance-push-spec.md).
 * @typedef {object} AttendanceSignals
 * @property {boolean} enabled
 * @property {(event: string, day: string, facility: string) => Promise<unknown>} nudge
 */
```

### 4.3 `AttendanceService` (`src/attendance/AttendanceService.mjs`)

- The constructor takes an optional `signals` (`AttendanceSignals`, default `null`).
  Type it in the JSDoc like the other dependencies.
- A private `#nudge(config, event, day, facilityName)`:
  - Return at once unless `this.signals?.enabled && config.livePushOn` (D6).
  - Otherwise `await this.signals.nudge(event, day, facilityName)` inside `try`. On any
    error, log `this.logger.warn(\`attendance ${day}/${facilityName}: live nudge failed:
    ${err.message}\`)` and return. Never throw.
- `mark`: after `updateValues` and the cache delete, inside `if (plan.changed)`, call
  `await this.#nudge(config, event, day, facilityName)`. The return value is unchanged.
- `#reconcileFacility`: if this run called `createAttendanceTab` or wrote rows with
  `updateValues`, call `#nudge` **once**, after the race re-check, for that facility.
  It needs the event: take it from `config.getDay(day)` as `mark` does. A skipped
  reconcile (same fingerprint) and a run with nothing to write send nothing.

### 4.4 Wiring (`src/app.mjs`)

Pass `signals: livePublisher` to `new AttendanceService({ … })`. Nothing else is
constructed.

### 4.5 Test helpers

`FakeLivePublisher` (`test/helpers/fakes.mjs`) gains `nudges` (an array of
`{ event, day, facility }`), a `nudge(event, day, facility)` that records one, and
`nudgeFails` (an `Error`, default `null`) that makes `nudge` reject after recording.

### 4.6 New tests (write each first; watch it fail)

| File | Cases |
|---|---|
| `test/unit/clients/LivePublisher.test.mjs` | `nudge`: `POST` to `…/nudge/<event>/<day>` (URL-encoded), the secret header, the JSON body `{ facility }`; `200` resolves to `{ clients }`; `401` and `500` reject with `live Worker nudge failed: HTTP …`; a request that never answers rejects with `timed out after 2000ms` (use the file's existing timer approach), even when `timeoutMs` is 4000 |
| `test/unit/attendance/AttendanceService.test.mjs` | A changed mark nudges once with the event, the day and the facility name, **after** `updateValues`. An unchanged mark sends no nudge. `livePushOn: false` and `enabled: false` each send none. A rejecting nudge leaves the mark's return value as it was, logs one `live nudge failed` warning, and doesn't throw. A reconcile that writes rows nudges once; one that creates the tab nudges once; a skipped reconcile and one with nothing to write send none. A service built **without** `signals` marks and reconciles as before |
| `test/integration/attendance-routes.test.mjs` | Through the real app with `FakeLivePublisher`: `PUT /v3/…/attendance` that changes someone records one nudge. The same `PUT` again records none. A nudge that rejects still answers `200` with the same body |

### 4.7 Version and Changelog

`package.json` **3.4.0**. A `### 3.4.0` entry at the top of `README.md`'s Changelog in
its usual shape, saying what pages now get and that nothing private crosses the socket.
It ends with a **Tests:** line naming the three test files and `test/helpers/fakes.mjs`.

---

## 5. The site (`sage-match-control.github.io`)

### 5.1 `lib/v1/data/attendance-signal.js` (new) — push A

```js
// The attendance nudge (sage-docs/docs/specs/.../attendance-push-spec.md).
// One WebSocket per page to the Worker's attendance object for a day. A message says
// only which facility's list changed; the page re-reads it from the API.
import { LIVE_PING_MS, LIVE_PONG_TIMEOUT_MS, LIVE_RETRY_MAX_MS } from './live-channel.js';

export const ATTENDANCE_NUDGE_GAP_MS = 2000;   // at most one re-read per facility this often
export const ATTENDANCE_SETTLE_MS = 11000;     // sage-tools-api's PEOPLE_CACHE_MS (10 s) + 1 s
export const ATTENDANCE_LIVE_POLL_MS = 30000;  // the view's poll while the socket is open

/**
 * @param {{ baseUrl: string, enabled: boolean, event: string, day: string,
 *           onChange: (facility: string) => void }} options
 * @returns {{ isOpen(): boolean, close(): void }}
 */
export function createAttendanceSignal(options){ … }
```

These imports are of exports that already exist, and the file is new, so it's safe
in push A.

- **The socket:** connect to `` `${baseUrl}/attendance/${event}/${day}` `` (both
  URL-encoded) when `enabled`. Mirror `createLiveChannel` for upkeep: `ping` every
  `LIVE_PING_MS`, close if no `pong` within `LIVE_PONG_TIMEOUT_MS`, reconnect with the
  same backoff (`1000 · 2^retries`, capped at `LIVE_RETRY_MAX_MS`, plus up to 1 s of
  jitter). `close()` stops everything for good.
- **Messages:** ignore `pong` and anything that isn't JSON with `type === 'attendance'`
  and a string `facility`.
- **Coalescing (D7), per facility:**
  - Call `onChange(facility)` at once if the last call for that facility was at least
    `ATTENDANCE_NUDGE_GAP_MS` ago. Otherwise call it once when the gap ends.
  - After every nudge, (re)start that facility's settle timer: `onChange(facility)` again
    `ATTENDANCE_SETTLE_MS` after the **last** nudge.
- `isOpen()` is `ws.readyState === WebSocket.OPEN`.

### 5.2 `lib/v1/views/attendance-view.js` — push A

- New optional options:
  - `isLive`, `() => boolean`. When it returns true, the interval skips a tick unless the
    last load started at least `livePollMs` ago.
  - `livePollMs`, default `30000`, defined as a literal in this file, not imported, since
    a new import would break the rule in §0.3 rule 6.
  - Without `isLive`, nothing changes.
- The returned object gains `changed(facility)`:
  - If the page is hidden, or the facility isn't one this view loads (desk mode: the
    chosen venue; console: any of `facilities`), do nothing.
  - Otherwise do a quiet load of **that facility only**. Give `load` an optional second
    argument naming one facility. The `loadSeq` rule must not drop the other facilities'
    data: only a newer load **of the same facility** supersedes it.
  - Someone this device is still saving keeps their local state, as in a poll.
- Don't change the existing behaviour of `refresh`, `destroy`, the interval with
  `isLive` absent, `visibilitychange` or optimistic marking.

### 5.3 `lib/v1/apps/attendance-desk.js` — push B

- `KNOWN_SETTINGS` becomes `['eventKey', 'liveBaseUrl']` (optional, default `''`).
- After `createAttendanceView`, keep the view it returns. Then
  `createAttendanceSignal({ baseUrl: settings.liveBaseUrl, enabled: !!settings.liveBaseUrl && !FIXTURE, event: EVENT_KEY, day: info.day, onChange: f => view.changed && view.changed(f) })`,
  and pass the view `isLive: () => signal.isOpen()`.
  - Create the signal first and give the view a function that reads it, so neither
    needs the other at construction.
  - `view.changed &&` guards against a cached push-A-less view, which is harmless.

### 5.4 `tools/control-center.html` — push B

- Where the Attendance tab creates its view (`attConsoleView = createAttendanceView({…})`),
  also create an attendance signal for the event and day, with `baseUrl: LIVE_BASE_URL`
  and `enabled: !!LIVE_BASE_URL && !FIXTURE`, and pass `isLive`.
  `onChange` calls `attConsoleView.changed && attConsoleView.changed(f)`.
- Close the signal wherever that view is destroyed or replaced: a day or event change,
  sign-out, leaving the tab if the view is destroyed then.
- Import `createAttendanceSignal` from `/lib/v1/data/attendance-signal.js`.
- The day's existing `liveChannel` is untouched.

### 5.5 `_templates/attendance/attendance.html` — push C

Add `const LIVE_BASE_URL = 'wss://sage-live.sagematchcontrol.workers.dev';` beside
`EVENT_KEY`, the same literal as Control Center: a literal, not a `{{TOKEN}}`. Pass
`liveBaseUrl: LIVE_BASE_URL` to `mountAttendanceDesk`. The finished events' desk pages
aren't changed (D9).

### 5.6 `_tests/`

- `unit/data/data-modules.test.mjs`, in the style of its `createLiveChannel` tests
  (the `FakeSocket` stub and `mock.timers`), for `createAttendanceSignal`:
  - the URL;
  - `enabled: false` never connects;
  - a nudge calls `onChange` at once;
  - a second nudge for the same facility within 2 s calls it once more, when the gap
    ends;
  - two facilities don't share a gap;
  - the settle call comes 11 s after the **last** nudge, and a nudge in between pushes
    it back;
  - junk, `pong` and other `type`s are ignored;
  - reconnect backoff after a close;
  - `close()` stops reconnecting and pending calls;
  - `isOpen()`.
- `unit/views/attendance-view.test.mjs`: if `changed`, the one-facility load and the
  `isLive` interval can be driven without a DOM by the file's existing means, add cases;
  otherwise say so in your report, and §6 step 4 covers them.
- `unit/live-pages.test.mjs` and `unit/no-copies.test.mjs` must still pass: no page
  defines its own copy of the signal.
- `npm run verify`: the comparison harness must show **no difference** on any page.
  Fixture mode turns the signal off.

---

## 6. Checking it yourself

1. `npm run verify` in `sage-tools-api`: passes, apart from any failure that also fails
   on `main` in your checkout. Name those.
2. `npm run verify` in `sage-match-control.github.io/_tests`: passes.
3. `cd sage-tools-api/live-worker && npx wrangler dev` with a `.dev.vars` secret, then
   `node smoke.mjs http://localhost:8787 <secret>`: every check passes, the old and the
   §3.3 ones.
4. Serve the site locally, with a local API and the local Worker. Open two desk pages and
   Control Center's Attendance tab for one day, each pointed at `ws://localhost:8787`.
   - Mark someone on one desk. The other desk and the console show it within about
     2 s, and each makes one more read about 11 s later.
   - With no marks, each page reads the list every 30 s, not 10 s.
   - Stop the Worker: polling returns to every 10 s. Restart it: the pages reconnect.
5. `grep -rn "attendance-signal" sage-match-control.github.io/events`: no matches.
   Finished desk pages don't use it.

---

## 7. Documentation

| Page | What it says |
|---|---|
| `sage-docs/docs/technical/event-attendance.md` | Replace *The live Worker is not used: its channel is public* with the nudge: the separate object, the data-free message, D4–D8, and the 30 s fallback poll. Add the nudge to the flow diagram. A **Push** subsection after *Reading the list* |
| `sage-docs/docs/technical/sync-pipeline.md`, *Live push delivery* | One paragraph: the Worker also relays attendance nudges on a separate object per day; the snapshot channel is unchanged |
| `sage-docs/docs/technical/api.md` | No route changes. Under attendance, one sentence that a write that changes the tab nudges the Worker |
| `sage-docs/docs/technical/control-center.md` | The Attendance tab's attendance signal and when it is closed |
| `sage-docs/docs/features/event-attendance.md` | Marks made on other devices appear within a second or two; within about 30 seconds if live updates are down, or press **Refresh** |
| `sage-docs/docs/usage/desk-handout.md` | *Marks made on other phones appear within about 10 seconds* becomes *within a second or two* |
| `sage-docs/docs/changelog.md`, `mkdocs.yml` | A MINOR entry and the version, per the changelog's rules |
| `D:\Personal\SAGE\CLAUDE.md` | The data-flow note: the Worker also relays data-free attendance nudges (`/attendance`, `/nudge`). `live-worker/`'s layout entry: the two routes. `src/attendance/`: it nudges the Worker after a write |

---

## 8. What doesn't change

- `GET …/people`, its cache, its response and its headers (D11). No route is added or
  removed, and `/v1` is frozen as always.
- The day's snapshot channel: `/live`, `/snapshot`, `/publish`, `#connect`, `#publish`,
  `createLiveChannel`, and every page that uses them.
- The finished events' desk pages, `event-data/`, every `apps-script/*.gs`, and
  `wrangler.jsonc` (no migration, D2).
- Google reads (D11).

---

## 9. Alternatives considered (for the record; don't build these)

| Alternative | Why not |
|---|---|
| Push the list itself over the socket | The socket is public (only `Origin` is checked, and that is trivially forged). Check-in state and shirt sizes are staff data |
| An authenticated attendance socket, the Worker verifying desk and operator tokens | The Worker would need the token keys, a second copy of `AuthService`'s rules in another runtime, and their expiry rules kept in step. The data-free nudge needs none of it |
| The nudge on the day's existing snapshot channel | Every public viewer would receive every nudge, which also reveals when check-in activity happens. A fault in the new code would share an object with the public feed. It saves one socket per staff page |
| Server-Sent Events from Cloud Run | A long request per page keeps instances billed and busy, and a write on one instance can't reach a page connected to another |
| A shorter poll | More API requests with no fresher data: the 10 s cache sets the floor |
| A `?since=` on `GET …/people` that forces a fresh read | A token holder could force a Google read per request; the 11 s follow-up gets the same freshness from the existing cache |

---

## 10. Finishing

### 10.1 Report back

1. `git status --short` from `sage-tools-api`, the site and `sage-docs`, and the
   `CLAUDE.md` lines you changed.
2. `npm run verify`'s summary from `sage-tools-api` and from the site's `_tests/`.
3. `smoke.mjs`'s output against `wrangler dev`.
4. §6's checks, each ticked or explained.
5. Which site files belong to push A, B and C (D10).
6. Anything in this spec you found wrong or had to interpret, with what you did.

### 10.2 The owner's checks (not yours)

| # | Check | Pass |
|---|---|---|
| P1 | Deploy the Worker by hand (not within 3 days of an event); run `smoke.mjs` against it | every check passes; the public Hub still updates live |
| P2 | Deploy the API; on a scratch event, mark someone from Control Center with no desk page open | the mark works; the log shows no `live nudge failed` |
| P3 | Push A; 10 minutes later push B; open a desk link and Control Center's Attendance tab | a mark on one shows on the other within about 2 s |
| P4 | Switch **Sync method** to **GitHub only** | marks still save; other pages catch up within about 30 s; switch back |
| P5 | 10 minutes after B, push C; instantiate the template for a scratch event | its desk page connects and updates live |
| P6 | With two desks open, watch Cloud Run's requests to `…/people` for ten quiet minutes | about 2 a minute per page, not 6 |
