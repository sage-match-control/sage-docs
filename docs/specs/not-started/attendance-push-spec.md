# Spec — Push for attendance

When someone is marked in at one desk, every other desk and Control Center's
Attendance tab show it within a second or two, instead of within about 20 seconds.
Pages stop polling: they read the list when they open, when they come back into view,
when a nudge arrives, and when someone presses **Refresh**.

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

`D:\Personal\SAGE` is a plain folder holding four git repos. Elsewhere (a cloud
checkout, say), it is whichever folder holds the four repos side by side, and the
workspace `CLAUDE.md` this spec calls `D:\Personal\SAGE\CLAUDE.md` is the one in the
`sage-workspace` checkout beside them. Every path below is relative to that folder.

| Path | What you change there |
|---|---|
| `sage-tools-api/live-worker/` | §3: two routes on the Worker, the `DayChannel` object, `smoke.mjs`, `README.md` |
| `sage-tools-api/` | §4: `LivePublisher.nudge`, `AttendanceService` sends nudges, a port, wiring, tests, version, Changelog |
| `sage-match-control.github.io/` | §5: a new `lib/v1/data/attendance-signal.js`, `attendance-view.js`, `lib/v1/css/attendance.css`, `attendance-desk.js`, `tools/control-center.html`, the attendance template, `_tests/` |
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
| `LivePublisher` has `enabled` (URL and secret both set), `read`, `publish`, a private `#request(method, kind, event, day, body = null)` that sends `X-Publish-Secret`, times out at `timeoutMs` (4000) and **derives the word in its error messages from the method** (`GET` → `read`, anything else → `publish`), and `#failure(what, res)` → `Error("live Worker <what> failed: HTTP <status> <body>")` | `src/clients/LivePublisher.mjs` |
| `smoke.mjs` uses **one** throwaway `key`, `` `smoke-test/run-${Date.now()}` ``, which is the event and the day together (event `smoke-test`, day `run-<ms>`), so a URL is `` `${base}/live/${key}` ``. It has helpers `report(step, ok, detail)` and `nextMessage(ws, ms)`, numbered steps 1–6, and `ws.close()` after step 6 | `live-worker/smoke.mjs` |
| `config.livePushOn` is `raw.livePush !== false` (the **Sync method** switch); `SnapshotPublisher.liveActive(config)` is `livePublisher.enabled && config.livePushOn` | `src/registry/SyncConfigSnapshot.mjs`, `src/sync/publishing/SnapshotPublisher.mjs` |
| `AttendanceService.mark` writes only when `planMark(…).changed`, then `this.peopleCache.delete(facility.sheetId)`. `#reconcileFacility` deletes the entry after `createAttendanceTab` and after `updateValues` with rows. `listPeople` → `#cachedPeople` reuses a read for `PEOPLE_CACHE_MS` (10 000). `this.now()` returns a `Date` | `src/attendance/AttendanceService.mjs` |
| `#reconcileFacility({ config, day, type, facility, standingsRows, force })` is **not** given the event. Both callers have it: `reconcileAfterSync({ day, event, facilities })` as an argument, `reconcileDay` from `config.getDay(day)`. `reconcileAfterSync` waits for all facilities at most `RECONCILE_BUDGET_MS` (6000) | `src/attendance/AttendanceService.mjs` |
| `src/app.mjs` constructs one `LivePublisher` (`livePublisher`) and `new AttendanceService({ configStore, sheets, authService, logger })`; `test/helpers/fakes.mjs` has `FakeLivePublisher({ enabled, snapshot })`. `test/unit/app.test.mjs` checks what `createApp` returns (`app.attendanceService instanceof AttendanceService`) | `src/app.mjs`, `test/helpers/fakes.mjs`, `test/unit/app.test.mjs` |
| `AttendanceService.test.mjs` builds the service with `make(extra)` (a fresh `FakeSheets`, which records `writes`, `reads` and `created`; a `capturingLogger()`, whose `lines` holds `["warn", message]` pairs; `now: () => NOW`), and `...extra` overrides any dependency | `test/unit/attendance/AttendanceService.test.mjs` |
| `attendance-routes.test.mjs` does **not** go through `createApp`: it serves the real routes with `startApp({ services })` (`test/helpers/http.mjs`). Its last block, *a desk link issued to someone, end to end*, builds a real `AttendanceService` over `FakeSheets` and marks through `PUT /v3/…/attendance` | `test/integration/attendance-routes.test.mjs` |
| `createAttendanceView(opts)` polls with `setInterval(() => { if(!document.hidden) load(true); }, opts.pollMs)` and loads on `visibilitychange`; it returns `{ refresh, setShowWithdrawn, destroy }`. `load(quiet)` fetches every facility in console mode and the chosen venue in desk mode, and drops its reply when `st.loadSeq` has moved on. A successful mark already calls `load(true)`. The skeleton is `box.append(venuesEl, bar, countEl, catbar, listEl)`, where `bar` holds the search box and **Refresh** | `lib/v1/views/attendance-view.js` |
| `mountAttendanceDesk(settings)` accepts only `KNOWN_SETTINGS = ['eventKey']`; any other name logs `unknown setting` and **returns without rendering**. It discards what `createAttendanceView` returns, and has `EVENT_KEY`, `FIXTURE` and `info.day` in scope | `lib/v1/apps/attendance-desk.js` |
| The attendance template's shell has no `LIVE_BASE_URL`; Control Center does (`'wss://sage-live.sagematchcontrol.workers.dev'`) and already runs `createLiveChannel` for the day's snapshot. Its Attendance tab is built in **`attConsoleMount()`** (`attConsoleView = createAttendanceView({…, event: CURRENT_EVENT_KEY, day: day.key, …})`) and torn down only in **`attConsoleDestroy()`**, which every exit calls (another tab, a day or event change through `syncAttendanceTab`, a remount). `ATTENDANCE_POLL_MS` is imported from `platform.js` and used only as that `pollMs` | `_templates/attendance/attendance.html`, `tools/control-center.html` |
| The finished events' desk pages (Pickle for Sight, PickleDrive, Piggleball) are inline copies that do not import the engine: PickleDrive's and Piggleball's carry their own inline `createAttendanceView` and poll with their own `ATTENDANCE_POLL_MS`. No live page imports `attendance-desk.js` today; only the template does | `events/*/attendance.html` |
| Site test rules: `_tests/unit/guards.test.mjs` lets `views/` import only `views/` and `domain/`, and `data/` import `platform.js`, `domain/` and `data/`, so **the view never imports the signal**; the app and Control Center wire the two together. `_tests/unit/views/attendance-view.test.mjs` has **no DOM** and covers only `attendanceCsvUrl`, `fetchPeople` and `markPerson`. `_tests/unit/apps/settings.test.mjs` checks each mount's unknown and required settings. `_tests/unit/data/data-modules.test.mjs` has a `FakeSocket` class and a `liveSetup(t)` that installs it as `globalThis.WebSocket` under `mock.timers` | `_tests/unit/` |
| The comparison harness answers every `sage-live` socket by closing it at once (`page.routeWebSocket(/sage-live/, ws => ws.close())` in `_tests/helpers/browser.mjs`) and installs a fake clock, so no timer fires unless a step moves it. Its attendance cases are `attendance/<event>/…` (the template's desk page, not in fixture mode), `cc/<event>/attendance/…` (signed out, already accepted as row 28) and `cc-signed-in/<event>/attendance/…` | `_tests/helpers/browser.mjs`, `_tests/engine/cases.mjs`, `_tests/engine/accepted.mjs` |
| `npm test` in `sage-tools-api`, in a Linux checkout on 2026-10-12: 4 failures, all the HTML snapshots in `test/unit/scoresheets/TemplateService.test.mjs`, unrelated to this spec (§6 step 1 says what to do with them). The site's `_tests` `npm test`: all pass | — |

---

## 1. Decisions (settled with the owner, 2026-10-12)

| # | Decision |
|---|---|
| D1 | **The nudge carries no data.** The message is exactly `{"type":"attendance","facility":"<name>"}`. No person, key, name, state, time or count. A facility's name is already public in `events.json`. Pages re-read the list from the API with their own token |
| D2 | **Its own object, not the day's snapshot channel.** The Worker gives attendance a separate object, `idFromName(\`attendance:${event}/${day}\`)`, of the **same** `DayChannel` class (no migration). It stores nothing. Public viewers never receive a nudge, a desk never receives a snapshot, and a fault here can't reach the public feed. Event and day keys match `[a-z0-9-]`, so the `attendance:` prefix can't collide with a snapshot object's name |
| D3 | **Two Worker routes.** `GET /attendance/<event>/<day>` is the socket: `Upgrade: websocket`, the same `Origin` rule as `/live`, no secret, nothing sent on connect. `POST /nudge/<event>/<day>` with body `{ facility }` needs `X-Publish-Secret`, broadcasts the D1 message to that object's sockets and answers `200 {clients}`. A `facility` that isn't a 1–100 character string is `400` |
| D4 | **The API nudges after every write that changes the tab**, and only then: a mark whose plan `changed`, a reconcile that created the tab, and a reconcile that wrote rows. One nudge per write, after the write and the cache drop |
| D5 | **Best effort, awaited, short.** The nudge is awaited before the response, because Cloud Run throttles CPU once a response is sent, with a **2-second** timeout. A failure or timeout logs one warning and never changes the answer or throws |
| D6 | **Nudges follow the environment, not the Sync method switch.** The API nudges whenever `LivePublisher.enabled` (URL and secret set), whatever `config.livePushOn` says. With no polling (D8), a nudge is the only way other pages hear of a mark, and the switch is about how snapshots are delivered. After a failed nudge the service sends none for **60 s** (per instance), so a Worker outage costs a mark at most one 2-second wait a minute |
| D7 | **Pages coalesce.** A nudge for a facility the page shows re-reads that facility only, at most once every 2 s (trailing). **And once more 11 s after the last nudge**, because another Cloud Run instance can hold a list up to 10 s old (`PEOPLE_CACHE_MS`), so the follow-up is never stale. Nudges for a facility the page doesn't show are ignored. While the page is hidden, nudges are ignored; the existing `visibilitychange` load catches up |
| D8 | **No polling.** The owner's call: no timer re-reads the list. A page reads it when it opens, when it comes back into view (`visibilitychange`: a phone that slept missed nudges), when its socket opens again after a drop (it missed nudges meanwhile), on a nudge (D7), and when someone presses the existing **Refresh** button. When the socket isn't open, the view says so in a line of its own: *Live updates are off. Press Refresh to see marks from other devices.* |
| D9 | **Where it runs.** The desk page (desks mode, from the template) and Control Center's Attendance tab. Off in fixture mode. A page with no Worker address gets no nudges and shows the D8 line. The finished events' inline desk pages are untouched and keep their own polling |
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
- **API requests:** none while nothing changes, instead of six a minute per page. At
  most one re-read per 2 s per page during a burst of marks, plus one follow-up.
- **Worker:** one nudge per write that changed the tab. Each nudge goes to that day's
  attendance sockets only.

---

## 3. The Worker (`sage-tools-api/live-worker/`)

### 3.1 `src/index.js`, `fetch`

- Widen `ROUTE` to `^/(live|snapshot|publish|attendance|nudge)/(${KEY})/(${KEY})$`.
- Name the object by kind: `attendance` and `nudge` use
  `` `attendance:${event}/${day}` ``, the rest `` `${event}/${day}` `` as now. Same
  `locationHint`.
- `attendance`: the same checks as `live` (`426` without the upgrade header, `403` for a
  disallowed `Origin`), then `stub.fetch(request)`. Today everything that isn't `live`
  falls through to the secret check, so widen that branch's condition to
  `kind === "live" || kind === "attendance"` rather than adding a second copy.
- `nudge`: the same secret check as `publish` (`401`), `POST` only (else `405`), then
  `stub.fetch(request)`. That is one more `|| (kind === "nudge" && request.method === "POST")`
  in the existing method check.

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

Add these as steps 7–12, after step 6 and before the existing `ws.close()`, on the same
`key` (so the URLs are `` `${wsBase}/attendance/${key}` `` and `` `${base}/nudge/${key}` ``).
`ws` is the step-2 `/live` socket, still open, its greeting already read. Add one helper
beside `nextMessage`: `noMessage(ws, ms)`, which resolves `true` if no message arrives
within `ms` and `false` if one does.

7. Open `` const att = new WebSocket(`${wsBase}/attendance/${key}`) ``, wait for its `open`
   event, then: no message within 1 s (nothing is sent on connect).
8. `POST /nudge/${key}` with the secret and `{ "facility": "Smoke Court" }` answers `200`
   with `clients ≥ 1`. Start listening on both sockets **before** sending it.
9. `att` receives exactly `{"type":"attendance","facility":"Smoke Court"}` within 2 s.
10. `ws` (the `/live` socket) receives nothing within 2 s of that nudge.
11. `POST /nudge/${key}` without the secret is `401`; with the secret and
    `{ "facility": "" }` it is `400`; a `GET` with the secret is `405`.
12. `att.send("ping")` gets `pong` within 3 s. Then `att.close()`.

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
`timeoutMs`. Don't duplicate `#request`: give it a sixth, optional argument
`{ what, timeoutMs } = {}`, where `what` defaults to what it derives from the method today
and `timeoutMs` to `this.timeoutMs`; `read` and `publish` keep calling it as they do. So
`nudge` calls `this.#request("POST", "nudge", event, day, { facility }, { what: "nudge", timeoutMs: 2000 })`,
throws `await this.#failure("nudge", res)` for any non-`ok` answer, and returns
`{ clients: body.clients }`. The messages are then `live Worker nudge timed out after 2000ms`
and `live Worker nudge failed: …`. Unlike `publish`, it logs nothing on success (one
line per mark would flood the log).

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
- The constructor stores `this.signals = signals` and
  `this.nudgeFailedAt = 0` (ms since the epoch of the last failed nudge; per instance,
  in memory, like the caches).
- A private `async #nudge(event, day, facilityName)`:
  - Return at once unless `this.signals?.enabled` (D6). The registry's `livePushOn` is
    **not** consulted, so `#nudge` takes no `config`.
  - Return at once if `this.now().getTime() - this.nudgeFailedAt < NUDGE_BACKOFF_MS`
    (60 000, a constant beside `PEOPLE_CACHE_MS`).
  - Otherwise `await this.signals.nudge(event, day, facilityName)` inside `try`. On any
    error, set `this.nudgeFailedAt = this.now().getTime()`, log
    `` this.logger.warn(`attendance ${day}/${facilityName}: live nudge failed: ${err?.message ?? err}`) ``
    and return. Never throw.
- `mark`: inside `if (plan.changed)`, after `updateValues` and the cache delete, call
  `await this.#nudge(event, day, facilityName)` (`event` is already in scope from
  `config.getDay(day)`). The return value is unchanged.
- `#reconcileFacility`: add `event` to its argument object, and pass it from both
  callers (`reconcileAfterSync` has it as an argument, `reconcileDay` from
  `config.getDay(day)`). Track whether this run called `createAttendanceTab` or called
  `updateValues` (only done when `data.length > 0`). If either, call
  `await this.#nudge(event, day, name)` **once**, after the race re-check and its
  logging, whether or not it raced (the tab changed either way), before the `return`.
  A skipped reconcile (same fingerprint) and a run with nothing to write send nothing.
  A reconcile after a sync runs inside `RECONCILE_BUDGET_MS`. A failing nudge adds at
  most 2 s, once a minute; if that ever tips a run past the budget,
  `reconcileAfterSync` logs and moves on as it already does. Change nothing there.

### 4.4 Wiring (`src/app.mjs`)

Pass `signals: livePublisher` to `new AttendanceService({ … })`. Nothing else is
constructed. `livePublisher` is constructed above `attendanceService` already, so no
lines move.

### 4.5 Test helpers

`FakeLivePublisher` (`test/helpers/fakes.mjs`) gains `nudges` (an array of
`{ event, day, facility }`), a `nudge(event, day, facility)` that records one, and
`nudgeFails` (an `Error`, default `null`) that makes `nudge` reject after recording.

### 4.6 New tests (write each first; watch it fail)

| File | Cases |
|---|---|
| `test/unit/clients/LivePublisher.test.mjs` | `nudge`: `POST` to `…/nudge/<event>/<day>` (URL-encoded), the secret header, the JSON body `{ facility }`; `200` resolves to `{ clients }`; `401` and `500` reject with `live Worker nudge failed: HTTP …`; a request that never answers rejects with `timed out after 2000ms` (use the file's existing timer approach), even when `timeoutMs` is 4000 |
| `test/unit/attendance/AttendanceService.test.mjs` | Build with `make({ signals: new FakeLivePublisher() })`. A changed mark nudges once with the event, the day and the facility name, **after** `updateValues` (a `nudge` that records `sheets.writes.length` when called sees the write already there). An unchanged mark sends no nudge. `enabled: false` sends none. An event in a registry with top-level `livePush: false` **still** nudges (build a second `SyncConfigSnapshot` from `{ ...RAW, livePush: false }`). A rejecting nudge leaves the mark's return value as it was, logs exactly one warning containing `live nudge failed`, and doesn't throw. After it, a mark 59 s later sends no nudge, and one 60 s later does (pass `now: () => current` and move `current`). A reconcile that writes rows nudges once; one that creates the tab nudges once (not twice when it also writes rows); a skipped reconcile (the same roster again with `force: false`) and a forced one with nothing to write send none. A service built **without** `signals` marks and reconciles as before. Use the file's existing `reconcileAfterSync`/`reconcileDay` cases as the pattern for setting up each tab |
| `test/integration/attendance-routes.test.mjs` | In a new `describe` modelled on *a desk link issued to someone, end to end* (its own `RAW`, `FakeSheets` and `startApp`), with a real `AttendanceService` given `signals: new FakeLivePublisher()`: an operator's `PUT /v3/…/attendance` that changes someone records one nudge `{ event, day, facility }`. The same `PUT` again records none. With `nudgeFails` set, the `PUT` still answers `200` with the same body as without it |
| `test/unit/app.test.mjs` | `createApp` gives the attendance service the app's `LivePublisher`: `app.attendanceService.signals === app.livePublisher` |

### 4.7 Version and Changelog

`package.json` **3.4.0**. A `### 3.4.0` entry at the top of `README.md`'s Changelog in
its usual shape, saying what pages now get and that nothing private crosses the socket.
It ends with a **Tests:** line naming the four test files and `test/helpers/fakes.mjs`.

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

/**
 * @param {{ baseUrl: string, enabled: boolean, event: string, day: string,
 *           onChange: (facility: string) => void,
 *           onStatus?: (open: boolean) => void }} options
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
- **Status:** `onStatus(true)` when the socket opens; `onStatus(false)` when it closes or
  a connection attempt fails, once per change (not once per retry). The state starts
  unknown, so the first open or the first failure always reports. With
  `enabled: false`, never create a socket and call `onStatus(false)` once, on the next
  tick (`setTimeout(…, 0)`, so the caller has its view by then).
- `isOpen()` is `!!ws && ws.readyState === WebSocket.OPEN`, as in `createLiveChannel`.
- After `close()`, no callback fires again: clear the ping, pong, retry, gap and settle
  timers, and detach the socket's handlers before closing it.
- No pause on a hidden page: the view ignores nudges while hidden (§5.2), and a socket
  that died while a phone slept reconnects through the normal backoff.

### 5.2 `lib/v1/views/attendance-view.js` — push A

- **No timer without `pollMs`.** Today the interval is set whenever the view isn't in
  fixture mode, and `setInterval(fn, undefined)` would fire constantly. Set it only when
  `opts.pollMs` is a positive number. The apps stop passing it in push B (D8). Until
  then, a cached older app passes it and keeps polling, which is harmless.
- The returned object gains `changed(facility)`:
  - If the page is hidden, or the facility isn't one this view loads (desk mode: the
    chosen venue; console: any of `facilities`), do nothing.
  - Otherwise do a quiet load of **that facility only**. Give `load` an optional second
    argument naming one facility: `load(quiet, only)`, whose targets are
    `[that facility]` when `only` is given and what they are today otherwise.
  - The `loadSeq` rule must not drop the other facilities' data: only a newer load
    **of the same facility** supersedes it. Do it this way: keep the global counter,
    add `st.facilitySeq = new Map()` (facility name → the seq of the newest load that
    included it); a load records its seq for each of its targets before fetching, and
    on reply applies only the results whose `st.facilitySeq.get(name) === seq`. If
    none apply, it returns without rendering, as a superseded load does today. A full
    load records every target, so it supersedes any single-facility load still in
    flight, and vice versa for that one facility.
  - Someone this device is still saving keeps their local state, as a load already
    does.
- It also gains `setLive(open)`:
  - `false` shows the D8 line, `true` hides it. The line is its own element,
    `el('div', 'att-live-off', 'Live updates are off. Press Refresh to see marks from other devices.')`,
    placed straight after `bar` (`box.append(venuesEl, bar, liveOffEl, countEl, catbar, listEl)`)
    and shown or hidden with its `hidden` property. Add `.att-live-off` to
    `lib/v1/css/attendance.css` beside `.att-status`, in that file's one-line style:
    the same size as `.att-status`, `color: var(--att-red)`, a small top margin.
    Adding a class is compatible. It isn't a `say()` message, because in Control
    Center those become toasts and this line has to stay.
  - When `setLive(true)` follows a `setLive(false)`, do one quiet load of everything the
    view shows (`load(true)`), to catch up on nudges missed while disconnected. The
    first `setLive(true)` loads nothing: the view's own first load covers it.
  - Before the first call the line is hidden.
- A successful mark keeps calling `load(true)`. The device that marked also gets its
  own nudge; the second read is served from the API's cache and costs nothing.
- Update the header comment (`returns { refresh(), destroy(), setShowWithdrawn(v) }`
  and the `pollMs` line) to describe `changed`, `setLive`, and `pollMs` as optional.
- Don't change the behaviour of `refresh`, the **Refresh** button, `destroy`,
  `visibilitychange` or optimistic marking.

### 5.3 `lib/v1/apps/attendance-desk.js` — push B

- `KNOWN_SETTINGS` becomes `['eventKey', 'liveBaseUrl']` (optional, default `''`).
- After `createAttendanceView`, keep the view it returns. Then
  `createAttendanceSignal({ baseUrl: settings.liveBaseUrl, enabled: !!settings.liveBaseUrl && !FIXTURE, event: EVENT_KEY, day: info.day, onChange: f => view.changed && view.changed(f), onStatus: open => !FIXTURE && view.setLive && view.setLive(open) })`.
  - Stop passing `pollMs` to `createAttendanceView` (D8). Drop `ATTENDANCE_POLL_MS`
    from the import if nothing else uses it. The export stays in `platform.js`, since
    nothing is removed from `lib/v1`.
  - `view.changed &&` and `view.setLive &&` guard against a cached push-A-less view,
    which is harmless.
  - Update the header comment's usage line to
    `mountAttendanceDesk({ eventKey, liveBaseUrl })` and describe `liveBaseUrl`
    (optional; `''` or absent: no nudges, and the page says live updates are off).

### 5.4 `tools/control-center.html` — push B

- Beside `let attConsoleView = null;` add `let attConsoleSignal = null;`.
- In `attConsoleMount()`, after `attConsoleView = createAttendanceView({…})`, set
  `attConsoleSignal = createAttendanceSignal({ baseUrl: LIVE_BASE_URL, enabled: !!LIVE_BASE_URL && !FIXTURE, event: CURRENT_EVENT_KEY, day: day.key, onChange: f => attConsoleView && attConsoleView.changed && attConsoleView.changed(f), onStatus: open => !FIXTURE && attConsoleView && attConsoleView.setLive && attConsoleView.setLive(open) })`.
  Stop passing `pollMs` (D8), and drop `ATTENDANCE_POLL_MS` from the `platform.js`
  import, since nothing else in the page uses it (the export stays).
- In `attConsoleDestroy()`, close the signal before the view:
  `if(attConsoleSignal){ attConsoleSignal.close(); attConsoleSignal = null; }`. Every
  teardown goes through it, so nothing else needs to close it. Signing out doesn't
  tear the tab down: the signal stays open, a nudge's load finds no token and the view
  shows *Sign in* as it does today.
- Import `createAttendanceSignal` from `/lib/v1/data/attendance-signal.js`, beside the
  `live-channel.js` import.
- The day's existing `liveChannel` is untouched.

### 5.5 `_templates/attendance/attendance.html` — push C

Add `const LIVE_BASE_URL = 'wss://sage-live.sagematchcontrol.workers.dev';` beside
`EVENT_KEY`, the same literal as Control Center: a literal, not a `{{TOKEN}}`. Pass
`liveBaseUrl: LIVE_BASE_URL` to `mountAttendanceDesk`. The finished events' desk pages
aren't changed (D9). Jekyll doesn't publish `_templates/`, so this reaches a live page
only when an event is next instantiated from the template; the wait in D10 protects
that page.

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
  - `onStatus` fires once per change of state, not once per retry, and once with
    `false` when `enabled` is false;
  - `isOpen()`.
  Put them in a new `// ---- attendance signal ----` section after the live-channel
  tests, reusing `FakeSocket` with a setup function like `liveSetup` that calls
  `createAttendanceSignal` and records `onChange` and `onStatus` calls.
- `unit/apps/settings.test.mjs`: one case in the style of *mountScheduleBoard knows
  "type"*: `mountAttendanceDesk({ eventKey: 'e', liveBaseUrl: 'wss://x' })` logs no
  `unknown setting` error. Past the settings check the mount reads `location`, which
  Node lacks, so it rejects with `location is not defined`: call it as
  `errorsFrom(() => mountAttendanceDesk({…}).catch(() => {}))` and assert only that no
  error mentions `unknown setting`.
- `unit/views/attendance-view.test.mjs` has no DOM, so `changed`, `setLive`, the
  one-facility load and the absent interval can't be unit tested there. Add no DOM
  library (no new dependency). The harness (below) and §6 step 4 cover them; say so in
  your report.
- `unit/guards.test.mjs`, `unit/live-pages.test.mjs` and `unit/no-copies.test.mjs` must
  still pass: the view imports nothing from `data/`, and no page defines its own copy
  of the signal.
- `npm run verify`: the comparison harness must show no difference on any page, apart
  from the D8 line. The harness closes every `sage-live` socket at once (§0.4), so on
  the branch the line shows on every non-fixture attendance case: the desk page
  (`attendance/…`) and Control Center's tab signed in (`cc-signed-in/…/attendance/…`).
  `cc/…/attendance/…` is already accepted in full (row 28). Add one entry per new
  pattern to `ACCEPTED` in `_tests/engine/accepted.mjs`, under a heading comment
  `// ---- push for attendance (sage-docs/docs/specs/.../attendance-push-spec.md §1 D8) ----`,
  shaped like the rows there, for example
  `{ case: /^attendance\//, kind: 'both', row: 'attendance-push D8', reason: 'with the live Worker unreachable the list says live updates are off' }`.
  Accept only the cases that actually differ, and list them in your report. If a case
  relied on a poll to show a change, report it rather than adding a timer back.
  Fixture mode turns the signal off and shows no line.

---

## 6. Checking it yourself

1. `npm run verify` in `sage-tools-api`: passes, apart from any failure that also fails
   on `main` in your checkout. Name those.
2. `npm run verify` in `sage-match-control.github.io/_tests`: passes.
3. In `sage-tools-api/live-worker`: `npm install` (installs the existing `wrangler`
   devDependency from its lockfile; not a new dependency), create `.dev.vars` holding
   `PUBLISH_SECRET=local-test-secret` (git-ignored), run `npx wrangler dev`, then
   `node smoke.mjs http://localhost:8787 local-test-secret`: every check passes, the old
   and the §3.3 ones. If `wrangler dev` can't run where you are (no network for its
   runtime, say), report that and the smoke output you could get.
4. This one needs a local API (`npm start` in `sage-tools-api` with a `.env` that has
   `LIVE_PUSH_URL=http://localhost:8787`, `LIVE_PUSH_SECRET=local-test-secret`, sign-in
   settings, and Google access to a scratch facility workbook) and the local Worker.
   If you don't have Google access or a scratch workbook, don't improvise one: say so
   in your report, and the owner's P3, P6 and P7 cover it. Otherwise serve the site from
   its root (`python3 -m http.server 8000`, or any static server; imports are
   root-absolute), and point the pages at the local services with **temporary** edits
   that you revert before reporting: `CLOUD_RUN_BASE_URL` in `lib/v1/platform.js`
   becomes `http://localhost:8080`, and `LIVE_BASE_URL` in `tools/control-center.html`
   and in a scratch copy of the desk page becomes `ws://localhost:8787` (name the copy
   `attendance-beta.html`: the site's `**/*beta*` rule keeps it out of git). Open two desk
   pages and Control Center's Attendance tab for one day.
   - Mark someone on one desk. The other desk and the console show it within about
     2 s, and each makes one more read about 11 s later.
   - With no marks, no page reads the list again, however long you wait.
   - Stop the Worker: each page shows *Live updates are off…*, and **Refresh** still
     works. Restart it: the pages reconnect, the line goes, and each page reads its list
     once.
   - Hide a desk page, mark someone on another, show it again: it reads the list once.
5. `grep -rn "attendance-signal" sage-match-control.github.io/events`: no matches.
   Finished desk pages don't use it.

---

## 7. Documentation

| Page | What it says |
|---|---|
| `sage-docs/docs/technical/event-attendance.md` | Replace *The live Worker is not used: its channel is public* with the nudge: the separate object, the data-free message, D4–D8, and that nothing polls. Add the nudge to the flow diagram. A **Push** subsection after *Reading the list* |
| `sage-docs/docs/technical/sync-pipeline.md`, *Live push delivery* | One paragraph: the Worker also relays attendance nudges on a separate object per day; the snapshot channel is unchanged |
| `sage-docs/docs/technical/api.md` | No route changes. Under attendance, one sentence that a write that changes the tab nudges the Worker |
| `sage-docs/docs/technical/control-center.md` | The Attendance tab's attendance signal and when it is closed |
| `sage-docs/docs/features/event-attendance.md` | Marks made on other devices appear within a second or two. If the page says *Live updates are off*, press **Refresh** to see them |
| `sage-docs/docs/usage/desk-handout.md` | *Marks made on other phones appear within about 10 seconds* becomes *within a second or two*; if the page says *Live updates are off*, press **Refresh** |
| `sage-docs/docs/usage/run-the-day.md` | Under *What to watch*, the **Live updates misbehaving** bullet (the one that says to switch **Sync method** to **GitHub only**): one sentence that the switch doesn't change attendance; desks still get live updates while the Worker is up |
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
| Polling as a fallback (every 10 s, or 30 s while connected) | The owner's call: no polling. The **Refresh** button, a re-read when the page comes back into view or reconnects, and the *Live updates are off* line cover the same gap |
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
| P4 | Switch **Sync method** to **GitHub only** | marks still save, and other pages still show them within about 2 s; switch back |
| P5 | 10 minutes after B, push C; instantiate the template for a scratch event | its desk page connects and updates live |
| P6 | With two desks open, watch Cloud Run's requests to `…/people` for ten quiet minutes | none after each page's first load |
| P7 | Block the Worker on one desk's network (or stop it) | that desk shows *Live updates are off*; **Refresh** shows other desks' marks |
