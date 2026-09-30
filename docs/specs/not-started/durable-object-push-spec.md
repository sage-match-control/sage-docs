# Spec — Live push delivery (Durable Objects)

> **Status: not started.** Nothing here is built. Written 2026-10-01 against
> `sage-tools-api` 2.2.0 and the `sage-match-control.github.io` pages as of
> that date.
>
> **This is one of two alternative designs.** The other is
> [Fast data delivery](fast-data-delivery-spec.md) (Cloudflare R2 behind a
> CDN, pointer polling). Build one, not both.
>
> **Prerequisite:** the [Immediate sync](immediate-sync-spec.md) spec — timing instrumentation, a retry for
> syncs that lose a GitHub commit race, and an Apps Script trigger that
> syncs straight away. It is written separately because it is useful on
> its own and both delivery designs need it. Build it first.
>
> Plain-language version: [explainer](durable-object-push-explainer.md).

Make a score typed into a facility sheet appear on every open page within a
few seconds, for free, by pushing each new snapshot to open pages over a
WebSocket from a Cloudflare Durable Object, instead of waiting for GitHub
Pages to rebuild and for pages to poll. (Making the Apps Script trigger sync
immediately is the prerequisite spec's job.)

---

## 0. Read this first (implementer orientation)

Assume no knowledge of this project beyond this page. What you need:

**Repos** — `D:\Personal\SAGE` is a plain folder holding four git repos:

| Repo | Role here |
|---|---|
| `sage-tools-api/` | Node 22 / Express, ESM `.mjs`, no TypeScript, no build step, no test framework. Runs on Google Cloud Run (`us-central1`), deploys automatically on push to `main`. Holds `scripts/sheets-sync.gs` (Apps Script, **not** deployed with the service — pasted into each spreadsheet by hand). The new Worker goes in a subfolder of this repo (Phase 1) |
| `sage-match-control.github.io/` | Static HTML on GitHub Pages. Every page is one self-contained file (inline `<style>` and `<script>`), no bundler, no shared JS files. Commit to deploy |
| `event-data/` | Public GitHub Pages repo. Holds `config/events.json` (the event/day/facility registry) and every published snapshot at `<event-key>/data/<day>.json` |
| `sage-docs/` | Documentation (mkdocs). This spec lives here |

**Rules that apply to every change** (from the root `CLAUDE.md`):

- Any code change in `sage-tools-api` bumps `package.json`'s version (patch
  for a fix, minor for a feature) **and** adds a matching entry to the
  Changelog section of `sage-tools-api/README.md`. Changes to `scripts/*.gs`
  alone do not bump the version.
- Cite a spec from outside `sage-docs` as
  `sage-docs/docs/specs/.../durable-object-push-spec.md` — literally `...`
  where the status folder goes, never the real folder.
- Documentation is written in the present tense: what the system does now.
- **Never deploy any part of this during an event day.** It replaces the
  live data path. Nothing in this spec ships before 3 October 2026 (two
  events run that day).

**How the live data flows today:**

```
Facility Google Sheet (one per venue per tournament day)
  | installable onEdit trigger -> onEditInstallable (scripts/sheets-sync.gs)
  | schedules a one-shot time-based trigger ~3s later -> runIfSettled
  v
POST /sync/:day?facility=<name>   Cloud Run, X-Sync-Secret header
  | SyncService.syncDay: fetch that facility's CSV + STANDINGSCSV tabs
  | (Sheets API), read the published snapshot back from GitHub, merge
  v
GitHub Contents API commit -> event-data/<event-key>/data/<day>.json
  | GitHub Pages build + deploy
  v
Pages poll https://sage-match-control.github.io/event-data/<event-key>/data/<day>.json
every 10s (POLL_INTERVAL_MS), paused while the tab is hidden
```

Once the [Immediate sync](immediate-sync-spec.md) spec is built, the first
step is different: `onEditInstallable` syncs straight from the edit under a
document lock (`syncUntilSettled_`) instead of scheduling a time-based
trigger. Nothing else in the flow above changes.

Control Center also calls `POST /sync/:day` (no `?facility=`, a full
resync, operator bearer token) and `POST /sync/:day/live` (the Live/Hide
override: `SyncService.setLiveOverride` commits `config/events.json`, then
republishes the day's snapshot with only `isLive` changed).

**The snapshot** (`SyncService.syncDay` builds it; `lastEditAt` comes from
the prerequisite spec, and this spec adds `publishedAt`):

```js
{
  day, label,
  isLive,               // true | false | "auto"
  generatedAt,          // ISO, set by each syncDay
  publishedAt,          // NEW (Phase 2): ISO, set on every publish, including Live/Hide
  facilities: [{
    name, matchesCsv, standingsCsv,
    syncedAt,           // ISO, restamped whenever this facility is fetched
    completedAt,        // ISO or null — src/sync/facilityCompletion.mjs
    lastEditAt          // added by the Immediate sync spec: ISO or null — when the edit behind the latest sync was made
  }],
  failedFacilities: [], // "name: error" strings
  staleFacilities: []   // names carried forward because this attempt failed them
}
```

## 1. Why, in numbers

Measured at CLSO Pickle for Sight, 27 September 2026 (2 facilities):

| Leg | p50 | p90 | max | Source |
|---|---|---|---|---|
| Apps Script: edit → `runIfSettled` fires | ? | ? | ? | Not measured. The code says `.after()` can fire up to ~1 min late. The Immediate sync spec measures it and removes the delay |
| Cloud Run `POST /sync/:day` (264 calls) | 1.6s | 1.85s | 19s | Cloud Run request logs |
| GitHub commit → Pages deployed (287 builds, 46 cancelled by newer pushes) | 25s | 52s | 82s | `event-data` Actions runs |
| Page poll | 5s | 10s | 10s | `POLL_INTERVAL_MS = 10000` |

Estimated edit → screen today: ~35–40s typical, 90s+ worst, plus the unknown
Apps Script delay. Target after all phases: **~2–5s typical, ~8–10s worst**
(a Cloud Run cold start plus a slow trigger).

| Leg after this spec | Estimate |
|---|---|
| Edit → sync starts (Immediate sync spec) | 0.5–2s, plus a 1.5s settle |
| Cloud Run fetch + merge + push (Phase 2) | ~1.5–2.5s (Sheets fetch ~1s, plus two ~200ms round trips to the Durable Object) |
| Durable Object → every open page (Phase 1, 5) | < 0.5s |

**Concurrent writes already fail.** Every facility of a day writes into the
same `<event>/data/<day>.json`. Each sync reads it, merges its own facility
in and commits it back with the sha it read; GitHub rejects a commit whose
sha is stale with `409 Conflict`, and today the losing sync just returns
`500`. All four failed syncs at Pickle for Sight were this (Cloud Run logs,
UTC): two same-workbook double fires (04:15:22, 11:07:56 — the old
time-based trigger firing twice for one burst) and two console full
resyncs colliding with a facility sync (06:44:41, 09:58:22). None lost data
that day, but a collision **between two facilities** loses the loser's
update until someone edits that facility's sheet again — minutes, if it was
a match's final score. With three facilities, expect a handful a day, more
as syncs get faster. The [Immediate sync](immediate-sync-spec.md) spec fixes it for today's GitHub path (its
§3.3 and §4.3), and Phase 2 here keeps the same guarantee for the Durable
Object (§6.5).

## 2. Target architecture

```
Facility Google Sheet
  | installable onEdit -> onEditInstallable -> syncUntilSettled_ (Immediate sync spec)
  |   takes a document lock, waits 1.5s, syncs; catches up if more edits land
  v
POST /sync/:day?facility=<name>  + X-Edit-At header (Immediate sync spec)
  | SyncService.syncDay (Phase 2):
  |   1. fetch the facility's tabs (unchanged)
  |   2. read the current snapshot + version from the Durable Object
  |      (GitHub only if the object has none yet)
  |   3. merge (unchanged logic)
  |   4. POST /publish with expectedVersion  --409--> back to step 2 (max 3 tries)
  |   5. commit the same snapshot to GitHub (archive; off the visibility path)
  v
Cloudflare Worker "sage-live"  (live-worker/, Phase 1)
  |  one Durable Object per <event>/<day>: stores {version, snapshot},
  |  broadcasts every accepted publish to its WebSockets
  v
wss://.../live/<event>/<day>   every open page (Phase 3)
  |  receives the snapshot, renders it through the page's existing code
  |  fallback: today's GitHub Pages polling whenever the socket is down,
  |  plus a 60s GitHub safety check while it is up
```

GitHub stays: it is the archive, the fallback read path for pages, the seed
for a Durable Object that has no snapshot yet, and what
`tools/scoresheet-generator.html` and archived pages keep reading.

## 3. Phases

Build the prerequisite first, then these phases in order. Each one ships on
its own:

| Phase | What | Where | Ships as |
|---|---|---|---|
| — | **Prerequisite:** [Immediate sync](immediate-sync-spec.md) — timing, commit-conflict retry, lock-based Apps Script sync | `sage-tools-api`, `sheets-sync.gs`, `control-center.html` | Its own spec |
| 1 | Worker + Durable Object | `sage-tools-api/live-worker/` | `wrangler deploy` (manual) |
| 2 | Cloud Run publishes to the object | `sage-tools-api/src/` | Minor version bump |
| 3 | Pages subscribe | `sage-match-control.github.io` | Site commit |
| 4 | Documentation and runbooks | `sage-docs`, `CLAUDE.md`s, checklist template | Commits |

After the prerequisite ships, re-measure with its `edit→request` numbers at
an event or rehearsal before starting Phase 1.

---

## 4. Prerequisite — Immediate sync

Everything this spec builds on is in the
[Immediate sync](immediate-sync-spec.md) spec. Confirm each of these exists
in the code before starting Phase 1; if any is missing, build that spec
first:

| What | Where to look |
|---|---|
| `X-Edit-At` header sent by Apps Script and read by Cloud Run | `scripts/sheets-sync.gs` `triggerSync_`; `src/sync/routes.mjs` `handleSync` |
| `facilities[].lastEditAt` in the snapshot, and a `timing` object in the sync response | `src/sync/SyncService.mjs` `syncDay` |
| `GitHubPublisher.publish` throws errors carrying `status` | `src/sync/GitHubPublisher.mjs` |
| 3-attempt re-read-and-re-merge on `409` in `syncDay`, `setLiveOverride` and `SyncConfigStore.setIsLive` | `src/sync/SyncService.mjs`, `src/sync/SyncConfigStore.mjs` |
| A private `#buildSnapshot(...)` holding the merge logic | `src/sync/SyncService.mjs` |
| `scripts/verify-sync-merge.mjs` with its eight scenarios | `sage-tools-api/scripts/` |
| Lock-based `syncUntilSettled_` and `syncWithRetry_` | `scripts/sheets-sync.gs` |

Phase 2 below changes `syncDay` and `setLiveOverride` again, building on
that retry loop and on `#buildSnapshot`.

---

## 5. Phase 1 — The Worker and Durable Object

### 5.1 Prerequisites (owner does these)

1. A free Cloudflare account. Choose the `workers.dev` subdomain when
   prompted; the Worker's URL becomes
   `https://sage-live.<subdomain>.workers.dev`. No custom domain is needed.
2. A long random publish secret (e.g. `node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"`),
   stored as a Worker secret (below) and as a Cloud Run env var (Phase 2).

### 5.2 Files

A new folder `sage-tools-api/live-worker/`. The Cloud Run image never
contains it: the `Dockerfile` copies only `package*.json`, `index.mjs`,
`src/` and `templates/`.

```
live-worker/
  package.json        # { "private": true, "type": "module", "devDependencies": { "wrangler": "^4" } }
  wrangler.jsonc
  src/index.js        # Worker entry + DayChannel Durable Object, plain JS
  smoke.mjs           # end-to-end check against a running Worker (6.6)
  README.md           # deploy and secret commands from 6.5
```

`live-worker/node_modules/` is already ignored by the repo's `.gitignore`
(`node_modules/`), and `.wrangler/` must be added to it.

`wrangler.jsonc`:

```jsonc
{
  "name": "sage-live",
  "main": "src/index.js",
  "compatibility_date": "2026-09-01",
  "durable_objects": {
    "bindings": [{ "name": "DAY_CHANNEL", "class_name": "DayChannel" }]
  },
  // Free plan: Durable Objects must be SQLite-backed.
  "migrations": [{ "tag": "v1", "new_sqlite_classes": ["DayChannel"] }],
  "vars": {
    "ALLOWED_ORIGIN": "https://sage-match-control.github.io"
  }
}
```

### 5.3 HTTP and WebSocket contract

Event and day keys must match `^[a-z0-9][a-z0-9-]{0,63}$` (the same rule
`SyncConfigStore` enforces). Anything else is `404`.

| Request | Auth | Response |
|---|---|---|
| `GET /health` | none | `200 ok` |
| `GET /live/<event>/<day>` with `Upgrade: websocket` | `Origin`, when present, must equal `ALLOWED_ORIGIN` or be `http://localhost:<port>` / `http://127.0.0.1:<port>`. No `Origin` (non-browser clients) is allowed | `101`, then messages below. Wrong origin `403`; no upgrade header `426` |
| `GET /snapshot/<event>/<day>` | `X-Publish-Secret` | `200 { "version": n, "snapshot": {…} }`, or `404 { "version": 0 }` if nothing has been published |
| `POST /publish/<event>/<day>` body `{ "expectedVersion": n, "snapshot": {…} }` | `X-Publish-Secret` | `200 { "version": n+1, "clients": k }`; `409 { "error": "version conflict", "version": current }` if `expectedVersion` isn't the current version; `400` on a bad body |

Wrong or missing secret: `401`. Compare secrets in constant time.

**Messages server → client** (text frames, JSON):

```json
{ "type": "snapshot", "version": 42, "snapshot": { "...": "the full snapshot" } }
{ "type": "empty" }
```

One is sent immediately on connect (`empty` if nothing is published yet),
then a `snapshot` after every accepted publish.

**Client → server:** only the literal text `ping`, answered with `pong` by
Cloudflare's auto-response without waking the object. Anything else is
ignored.

### 5.4 `src/index.js`

```js
import { DurableObject } from "cloudflare:workers";

const KEY = "[a-z0-9][a-z0-9-]{0,63}";
const ROUTE = new RegExp(`^/(live|snapshot|publish)/(${KEY})/(${KEY})$`);

export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    if (url.pathname === "/health") return new Response("ok");

    const m = url.pathname.match(ROUTE);
    if (!m) return new Response("not found", { status: 404 });
    const [, kind, event, day] = m;

    const id = env.DAY_CHANNEL.idFromName(`${event}/${day}`);
    // Only the first get() for an object is placed by the hint. Viewers are
    // in the Philippines; Cloud Run, which usually calls first, is in the US.
    const stub = env.DAY_CHANNEL.get(id, { locationHint: "apac-se" });

    if (kind === "live") {
      if (request.headers.get("Upgrade") !== "websocket") {
        return new Response("expected websocket", { status: 426 });
      }
      if (!originAllowed(request.headers.get("Origin"), env)) {
        return new Response("forbidden", { status: 403 });
      }
      return stub.fetch(request); // the object routes on the path's first segment
    }

    if (!(await secretOk(request.headers.get("X-Publish-Secret"), env.PUBLISH_SECRET))) {
      return new Response("unauthorized", { status: 401 });
    }
    if ((kind === "snapshot" && request.method === "GET")
        || (kind === "publish" && request.method === "POST")) {
      return stub.fetch(request);
    }
    return new Response("method not allowed", { status: 405 });
  },
};

// Browsers always send Origin on a WebSocket upgrade, so this stops other
// websites from embedding the feed. Non-browser clients send none and are
// let through: an Origin header is trivially forged, so refusing them would
// protect nothing and only break smoke.mjs.
function originAllowed(origin, env) {
  if (!origin) return true;
  if (origin === env.ALLOWED_ORIGIN) return true;
  return /^http:\/\/(localhost|127\.0\.0\.1)(:\d+)?$/.test(origin);
}

async function secretOk(given, expected) {
  if (!given || !expected) return false;
  const enc = new TextEncoder();
  const a = enc.encode(given), b = enc.encode(expected);
  if (a.byteLength !== b.byteLength) return false;
  return crypto.subtle.timingSafeEqual(a, b);
}

export class DayChannel extends DurableObject {
  constructor(ctx, env) {
    super(ctx, env);
    // Answered by Cloudflare without waking a hibernated object.
    ctx.setWebSocketAutoResponse(new WebSocketRequestResponsePair("ping", "pong"));
  }

  async fetch(request) {
    // The Worker forwards the original request: /<kind>/<event>/<day>.
    const kind = new URL(request.url).pathname.split("/")[1];
    if (kind === "live") return this.#connect();
    if (kind === "snapshot") return this.#read();
    if (kind === "publish") return this.#publish(request);
    return new Response("not found", { status: 404 });
  }

  async #current() {
    return (await this.ctx.storage.get("current")) ?? null; // { version, snapshot, publishedAt }
  }

  async #connect() {
    const [client, server] = Object.values(new WebSocketPair());
    this.ctx.acceptWebSocket(server); // hibernatable: no duration billed while idle
    const cur = await this.#current();
    server.send(JSON.stringify(cur
      ? { type: "snapshot", version: cur.version, snapshot: cur.snapshot }
      : { type: "empty" }));
    return new Response(null, { status: 101, webSocket: client });
  }

  async #read() {
    const cur = await this.#current();
    return cur
      ? Response.json({ version: cur.version, snapshot: cur.snapshot })
      : Response.json({ version: 0 }, { status: 404 });
  }

  async #publish(request) {
    let body;
    try { body = await request.json(); } catch { return new Response("bad json", { status: 400 }); }
    const { expectedVersion, snapshot } = body ?? {};
    if (!Number.isInteger(expectedVersion) || !snapshot || typeof snapshot !== "object") {
      return new Response("expectedVersion (integer) and snapshot (object) required", { status: 400 });
    }

    // Storage calls keep the object's input gate closed, so no other
    // request runs between this read and the put below: the version check
    // and the write are atomic.
    const cur = await this.#current();
    const version = cur?.version ?? 0;
    if (expectedVersion !== version) {
      return Response.json({ error: "version conflict", version }, { status: 409 });
    }
    const next = { version: version + 1, snapshot, publishedAt: new Date().toISOString() };
    await this.ctx.storage.put("current", next);

    const msg = JSON.stringify({ type: "snapshot", version: next.version, snapshot });
    const sockets = this.ctx.getWebSockets();
    for (const ws of sockets) {
      try { ws.send(msg); } catch { /* closing socket — its client reconnects */ }
    }
    return Response.json({ version: next.version, clients: sockets.length });
  }

  webSocketMessage(_ws, _message) {
    // Clients only send "ping", which the auto-response answers. Ignore the rest.
  }

  webSocketClose(ws, code, _reason, _wasClean) {
    try { ws.close(code, "closing"); } catch { /* already closed */ }
  }

  webSocketError(_ws, _error) {}
}
```

Implementation notes to verify during Phase 1 (§10 checks 1–3):

- If `locationHint: "apac-se"` is rejected at deploy or runtime, use
  `"apac"`.
- Confirm the message sent before returning the `101` arrives. If it
  doesn't, send it from a `ctx.waitUntil` after returning, or have the
  client ask for it (add a `hello` message).
- Snapshots are ~5–45 KB; one stored value may be up to 2 MB.

### 5.5 Deploy

From `sage-tools-api/live-worker/`:

```
npm install
npx wrangler login
npx wrangler secret put PUBLISH_SECRET
npx wrangler deploy
```

Deploying restarts every Durable Object and drops every WebSocket. Clients
reconnect on their own, but **never deploy the Worker during an event.** It
does not deploy on push; it deploys only when someone runs
`wrangler deploy`.

For local testing, `npx wrangler dev` runs the Worker and Durable Object on
`http://localhost:8787`, with secrets read from
`live-worker/.dev.vars` (add `.dev.vars` to `.gitignore`).

### 5.6 `smoke.mjs`

`node live-worker/smoke.mjs <baseUrl> <secret>` (Node 22 has a global
`WebSocket`). It uses a throwaway key pair such as `smoke-test/run-<timestamp>`
so it never touches a real event:

1. `GET /snapshot/...` → expect `404`, `version: 0`.
2. Open `/live/...` (Node's `WebSocket` sends no `Origin`, which the Worker
   allows) → expect `{type:"empty"}`.
3. `POST /publish/...` with `expectedVersion: 0` → `200`, `version: 1`; the
   socket receives `{type:"snapshot", version:1}` within 2s.
4. Publish again with `expectedVersion: 0` → `409`, `version: 1`.
5. Publish with a wrong secret → `401`.
6. Send `ping` → receive `pong`.

Print PASS/FAIL per step; exit non-zero on any failure.

---

## 6. Phase 2 — Cloud Run publishes to the Durable Object

### 6.1 Configuration

New env vars, documented in `.env.example` in the same commenting style:

```
# ---- Live push (Cloud Run -> Cloudflare Worker "sage-live") ----
# Base URL of the live Worker, no trailing slash, e.g.
# https://sage-live.<subdomain>.workers.dev. Leave unset to disable live
# push entirely: syncs then behave exactly as before (GitHub only).
LIVE_PUSH_URL=
# Must equal the Worker's PUBLISH_SECRET (wrangler secret put PUBLISH_SECRET).
LIVE_PUSH_SECRET=
# Per-request timeout for calls to the Worker. Default 4000.
LIVE_PUSH_TIMEOUT_MS=
```

Unset `LIVE_PUSH_URL` is the rollback switch: set it empty on Cloud Run and
every sync goes back to today's path, with no redeploy of the site.

### 6.2 `src/sync/LivePublisher.mjs` (new)

```js
// Publishes day snapshots to the live Worker's Durable Object, which pushes
// them to every open page. See sage-docs/docs/specs/.../durable-object-push-spec.md.
export class LivePublisher {
    constructor({ baseUrl, secret, timeoutMs = 4000 }, logger) { /* store */ }

    get enabled() { return !!(this.baseUrl && this.secret); }

    // -> { version: number, snapshot: object|null }. 404 means nothing
    // published yet: { version: 0, snapshot: null }. Throws on network
    // errors, timeouts and 5xx.
    async read(event, day) {}

    // -> { ok: true, version } on 200, { ok: false, conflict: true, version }
    // on 409. Throws on network errors, timeouts, 401 and 5xx.
    async publish(event, day, snapshot, expectedVersion) {}
}
```

Both calls send `X-Publish-Secret` and abort after `timeoutMs` with an
`AbortController`. Errors thrown are plain `Error`s whose message names the
Worker and the status.

### 6.3 `GitHubPublisher.publish` — status on errors

Already done by the [Immediate sync](immediate-sync-spec.md) spec (its §3.3): a failed PUT throws an error carrying
`status`, so `409 Conflict` can be told apart from other failures.

### 6.4 `SyncService` — constructor

Switch the constructor to one options object (it is already five positional
arguments):

```js
constructor({ sheetsApiFetcher, gvizFetcher, publisher, livePublisher, configStore, logger })
```

`index.mjs` is the only call site. Construct `LivePublisher` there from the
three env vars and pass it in. Add `live: { enabled, baseUrl }` to the
`GET /sync/config` diagnostics response in `routes.mjs` (never the secret).

### 6.5 `syncDay` flow

The fetch step and the merge logic (the `allFacilities.forEach` that builds
`finalFacilities`, including `resolveCompletedAt`) do not change. What
changes is where "existing" comes from and where the result goes.

```
fetch facilities (unchanged)

if !livePublisher.enabled:
    exactly today's code path (fetchExisting from GitHub, merge, publish with sha)
    — plus publishedAt and the Immediate sync spec's fields
    return

liveError = null
for attempt in 1..3:
    try:
        live = await livePublisher.read(event, day)
    catch err:
        liveError = err; break                 # degrade: GitHub path below
    existingJson = live.snapshot
    if existingJson == null:                   # object empty: first sync since deploy, or new day
        existingJson = (await publisher.fetchExisting(path)).json   # seed from GitHub
    snapshot = merge(fresh, existingJson)      # unchanged logic
    snapshot.publishedAt = new Date().toISOString()
    try:
        res = await livePublisher.publish(event, day, snapshot, live.version)
    catch err:
        liveError = err; break
    if res.ok: livePublished = res.version; break
    # 409: another sync or a Live/Hide override won — re-read and re-merge
if no publish succeeded and no liveError after 3 attempts:
    liveError = new Error("live publish: 3 version conflicts")

if liveError:
    log a warning with the error
    run today's GitHub path instead (fetchExisting with sha, merge, publish(sha));
    a GitHub 409 there is retried exactly as in the Immediate sync spec (its §3.3)
    return result with live: { published: false, error: liveError.message }

# Live publish succeeded: archive to GitHub, off the visibility path.
archive = await archiveToGitHub(path, snapshot, message)
return result with live: { published: true, version }, archive
```

`archiveToGitHub(path, snapshot, message)`:

- `publisher.publish(path, snapshot, message, null)` (null: it looks up the
  sha itself).
- On an error with `status === 409` (another request committed between the
  sha lookup and the PUT), re-read the **Durable Object's** current snapshot
  (`livePublisher.read`) and commit that instead — it is the newest — up to
  3 attempts.
- Any other failure, or 3 conflicts: log an error and return
  `{ committed: false, error }`. **Do not fail the request**: the live copy
  is correct and already visible. The next sync's archive commit catches
  GitHub up. (If the day's final sync fails to archive, the end-of-day
  **Resync this day now** in the runbook fixes it.)

The response keeps its current fields (`day`, `label`, `method`,
`facilitiesSynced`, `facilitiesFailed`, `facilitiesStale`, `commitSha`) and
adds `live`, `archive` and the Immediate sync spec's `timing`. `commitSha` is the archive
commit's sha, or `null` if the archive failed.

That `timing` object gains `liveMs` (read + publish to the object, all
attempts) and `archiveMs`. `editToPublishedMs` is measured when the **live**
publish succeeds — that is when pages see it — not after the archive. The
log line gains `live=<liveMs>ms archive=<archiveMs>ms`.

Order matters: **the live publish comes before the GitHub commit**, and both
happen before the response is sent. Never move the archive into
fire-and-forget work after `res.json()`: Cloud Run throttles CPU after a
response, so background work there is unreliable.

### 6.6 `setLiveOverride` flow

`configStore.setIsLive(day, isLive)` (the `events.json` commit) is
unchanged and still comes first. Then, when live push is enabled:

```
for attempt in 1..3:
    live = await livePublisher.read(event, day)          # errors -> today's GitHub path
    existingJson = live.snapshot ?? (await publisher.fetchExisting(path)).json
    if !existingJson: return { day, label, isLive, republished: false }
    snapshot = { ...existingJson, isLive, publishedAt: new Date().toISOString() }
    res = await livePublisher.publish(event, day, snapshot, live.version)
    if res.ok: break
archive as in 7.5
return { day, label, isLive, republished: true, live, archive }
```

Without this, a Live/Hide click would only reach GitHub, and every page
connected by WebSocket would keep showing the old state until the next
score edit. **Force hidden must work through the push path.**

### 6.7 Extend `scripts/verify-sync-merge.mjs`

The [Immediate sync](immediate-sync-spec.md) spec created the script (its §3.3). Add a fake `LivePublisher` holding
`{version, snapshot}` that can be told to return `409` once or to throw,
keep its eight scenarios passing, and add:

1. Live disabled → identical behaviour to the Immediate sync spec (GitHub read, merge,
   publish with sha, `409` retry).
2. Empty live object → merge seeds from the GitHub file; live `version` 1.
3. A forced `409` on the first publish, with the fake's state changed to
   include another facility's fresh data → the retry's snapshot contains
   **both** facilities.
4. Live read throws → the GitHub path runs and the result says
   `live.published: false`.
5. Archive `409` once → retried with the live object's snapshot; succeeds.
6. `setLiveOverride(false)` → live version increments; the stored snapshot's
   `isLive` is `false`; `publishedAt` is newer than before.
7. `lastEditAt` survives a full resync (no `editAt`), with live enabled.

Bump the minor version in `package.json` with a Changelog entry. Deploy by pushing to `main`, with
`LIVE_PUSH_URL` **unset** first. Then set the two env vars on the Cloud Run
service (a new revision, no code push) when ready to turn it on.

---

## 7. Phase 3 — Pages subscribe

### 7.1 Files

| File | Day switching | Hook points |
|---|---|---|
| `tools/control-center.html` | Event **and** day (`selectEvent`, `selectDay`) | `fetchDaySnapshot`, `selectDay`, `selectEvent`, the `visibilitychange` handler at the bottom, `renderOrganizerStatus` |
| `_templates/standard-tournament-template/index.html` | Day (`selectDay`) | `fetchDaySnapshot`, `selectDay`, `visibilitychange` |
| `_templates/dual-meet-template/index.html` | same | same |
| `_templates/standard-tournament-template/schedule.html` | None (fixed `DAY_KEY`) | `fetchDaySnapshot`, startup, `visibilitychange` |
| `_templates/dual-meet-template/schedule.html` | same | same |
| `events/<key>/index.html`, `events/<key>/schedule.html` for each event still in use | as its template | as its template |

Events that exist as of this spec: `pickle-for-sight-2026`,
`piggleball-2026`, `pnf-x-bup-dual-meet` (instantiated from templates) and
`pickledrive-anniversary-2026` (hand-built, has its own `FIXTURE` mode).
Update only those still running events; finished ones stay on polling.

Not changed: `tools/scoresheet-generator.html`, everything under
`events/archives/`, `beta.html` files.

### 7.2 The live channel block

One block of code, **identical in every file above**, pasted directly after
each page's `CONFIGURATION` constants and marked with banner comments:

```js
// ==== LIVE CHANNEL — identical in every page that uses it; see ====
// ==== sage-docs/docs/specs/.../durable-object-push-spec.md §7   ====
// Receives each new snapshot the moment Cloud Run publishes it, over one
// WebSocket per page. GitHub Pages polling stays as the fallback: whenever
// the socket isn't open, fetchDaySnapshot polls GitHub exactly as before.
const LIVE_BASE_URL = 'wss://sage-live.<subdomain>.workers.dev'; // '' disables live push
const LIVE_PING_MS = 50000;          // under Cloudflare's idle-connection timeout
const LIVE_PONG_TIMEOUT_MS = 10000;  // no pong by then: treat the socket as dead
const LIVE_SAFETY_POLL_MS = 60000;   // while connected, still check GitHub this often
const LIVE_RETRY_MAX_MS = 30000;

function createLiveChannel({ enabled, onSnapshot }){
  let ws = null, key = null, latest = null, paused = false;
  let retries = 0, retryTimer = null, pingTimer = null, pongTimer = null;
  let lastSafetyPoll = 0;

  const stamp = s => Date.parse((s && (s.publishedAt || s.generatedAt)) || 0) || 0;

  function clearTimers(){
    clearTimeout(retryTimer); clearInterval(pingTimer); clearTimeout(pongTimer);
    retryTimer = pingTimer = pongTimer = null;
  }
  function close(){
    clearTimers();
    if(ws){ ws.onclose = null; try { ws.close(); } catch(e){} ws = null; }
  }
  function scheduleRetry(){
    if(paused || !key) return;
    const delay = Math.min(LIVE_RETRY_MAX_MS, 1000 * 2 ** retries) + Math.random() * 1000;
    retries++;
    retryTimer = setTimeout(open, delay);
  }
  function open(){
    close();
    if(!enabled || paused || !key) return;
    const myKey = key;
    ws = new WebSocket(`${LIVE_BASE_URL}/live/${myKey.event}/${myKey.day}`);
    ws.onopen = () => {
      retries = 0;
      pingTimer = setInterval(() => {
        try { ws.send('ping'); } catch(e){}
        clearTimeout(pongTimer);
        pongTimer = setTimeout(() => { try { ws.close(); } catch(e){} }, LIVE_PONG_TIMEOUT_MS);
      }, LIVE_PING_MS);
    };
    ws.onmessage = ev => {
      clearTimeout(pongTimer);
      if(ev.data === 'pong') return;
      let msg; try { msg = JSON.parse(ev.data); } catch(e){ return; }
      if(msg.type !== 'snapshot' || !msg.snapshot) return;
      if(!key || key.event !== myKey.event || key.day !== myKey.day) return; // switched away
      if(latest && latest.key === `${myKey.event}/${myKey.day}` && msg.version <= latest.version) return;
      latest = { key: `${myKey.event}/${myKey.day}`, version: msg.version, snapshot: msg.snapshot };
      onSnapshot();
    };
    ws.onclose = () => { clearTimers(); ws = null; scheduleRetry(); };
    ws.onerror = () => { /* onclose follows */ };
  }

  return {
    // Point the channel at a day (or at nothing with null). Reconnects.
    follow(event, day){
      const next = event && day ? { event, day } : null;
      if(next && key && next.event === key.event && next.day === key.day) return;
      key = next; retries = 0; open();
    },
    pause(){ paused = true; close(); },
    resume(){ paused = false; retries = 0; open(); },
    isOpen(){ return !!ws && ws.readyState === WebSocket.OPEN; },
    // What fetchDaySnapshot should return, or null to fetch from GitHub.
    cached(event, day){
      if(!this.isOpen() || !latest || latest.key !== `${event}/${day}`) return null;
      if(Date.now() - lastSafetyPoll > LIVE_SAFETY_POLL_MS) return null; // time for a GitHub check
      return latest.snapshot;
    },
    // Called with every snapshot fetched from GitHub: returns whichever of
    // it and the pushed one is newer. Catches a push path that has quietly
    // stopped while the socket stays open (e.g. Cloud Run's live publish failing).
    newer(event, day, fetched){
      lastSafetyPoll = Date.now();
      if(!latest || latest.key !== `${event}/${day}`) return fetched;
      return stamp(fetched) > stamp(latest.snapshot) ? fetched : latest.snapshot;
    },
  };
}
// ==== END LIVE CHANNEL ====
```

### 7.3 Wiring, per page

**Create the channel** once, after the block. `FIXTURE` pages (Control
Center and PickleDrive on localhost) must not connect:

```js
const liveChannel = createLiveChannel({
  enabled: !!LIVE_BASE_URL && !(typeof FIXTURE !== 'undefined' && FIXTURE),
  onSnapshot: () => loadLiveData(),        // schedule.html: () => loadSchedule(false)
});
```

**`fetchDaySnapshot`**: rename the existing function to
`fetchDaySnapshotFromPages` (body unchanged) and add a wrapper with the old
name, so every caller is untouched:

```js
// index.html (event key is the page constant EVENT_KEY)
async function fetchDaySnapshot(dayKey){
  const pushed = liveChannel.cached(EVENT_KEY, dayKey);
  if(pushed) return pushed;
  const fetched = await fetchDaySnapshotFromPages(dayKey);
  return liveChannel.newer(EVENT_KEY, dayKey, fetched);
}
```

Control Center uses `CURRENT_EVENT_KEY` instead of `EVENT_KEY`.
`schedule.html`'s `fetchDaySnapshot()` takes no argument; use `DAY_KEY`.

If the GitHub fetch throws (e.g. `404` before the day's first publish) but a
pushed snapshot exists for this day, return the pushed one instead of
rethrowing.

**Follow the current day:**

- `index.html`: at the top of `selectDay(idx, …)`, after `currentDayIndex`
  is set: `liveChannel.follow(EVENT_KEY, DAYS[idx].key);`
- Control Center: the same in `selectDay` with `CURRENT_EVENT_KEY`; in
  `selectEvent`, where `currentDayIndex = null` is set,
  `liveChannel.follow(null, null);`
- `schedule.html`: once, just before the first `loadSchedule(true)`:
  `liveChannel.follow(EVENT_KEY, DAY_KEY);`

**Visibility:** in each page's existing `visibilitychange` handler, call
`liveChannel.pause()` in the hidden branch and `liveChannel.resume()` in the
visible branch, alongside what is there. Keep the existing poll timer
exactly as it is: while the socket is open, the poll re-renders from the
pushed snapshot without touching the network, and every
`LIVE_SAFETY_POLL_MS` it does one real GitHub fetch.

**Nothing else changes.** Rendering, `computeDayIsLive(day, snapshot)` (so
Live/Hide works through the pushed `isLive`), the `dayIndexAtStart` stale
guard, scroll preservation on the wall board, and error handling all run on
the pushed snapshot exactly as they do on a fetched one.

**Control Center status line:** in `renderOrganizerStatus`, add one line to
Mission Control: `Live updates: push connected` when
`liveChannel.isOpen()`, otherwise `Live updates: polling GitHub (push not
connected)`. Re-render it on connect and disconnect (call
`renderOrganizerStatus()` from `onSnapshot` is enough for connect; for
disconnect, have the page poll's existing call cover it).

Also correct the stale comment above `POLL_INTERVAL_MS` in every file that
says Apps Script's debounce is ~10s.

### 7.4 Templates

`LIVE_BASE_URL` is platform-wide, like `GHPAGES_REPO`: a literal constant in
both templates, **not** a `{{TOKEN}}`. Note it in
`_templates/CLAUDE.md`'s list of hard-coded constants.

---

## 8. Phase 4 — Documentation and runbooks

Update, in the present tense:

- Root `CLAUDE.md`: the data-flow diagram, the `sage-tools-api` layout
  (`live-worker/`, `LivePublisher.mjs`), and the hard-coded constants
  (`LIVE_BASE_URL`).
- `sage-tools-api/README.md` (Changelog entry for Phase 2's version) and
  `.env.example`.
- `sage-docs/docs/technical/sync-pipeline.md` (replace the "Live delivery
  today, and a planned upgrade" section), `architecture.md`, and
  `deployment.md` (the Worker's deploy and secret commands).
- `sage-docs/docs/features/control-center.md`: the "Live updates" line.
- `_templates/dry-run-checklist-template.md` and each running event's
  `dry-run-checklist.md`: in §2.1 add "Mission Control reads **Live updates:
  push connected**"; in §2.4 add "If it reads *polling GitHub*, updates
  still arrive, just 30–60s slower — keep going and tell whoever maintains
  the system".
- Move this spec to `implemented/` per `docs/specs/README.md`, and mark the
  R2 spec as superseded there.

---

## 9. Limits and cost

Workers Free plan, as of this spec:

| Limit | Value | Expected use |
|---|---|---|
| Durable Object requests | 100,000 / day per account | See budget below |
| Incoming WebSocket messages | billed 20 : 1 as requests | Client pings |
| Outgoing WebSocket messages | not billed as requests | Every broadcast |
| Worker requests | 100,000 / day per account | One per connect, publish, read |
| Durable Object storage | 1 GB per object, 5 GB per account | One snapshot per day, ≤ 45 KB |
| One object's throughput | soft ~1,000 requests/s | A few syncs a minute |
| Duration while hibernated | not billed | Idle between broadcasts |

**Daily budget**, per viewer watching all day (10h): ~720 pings (50s) ÷ 20
= 36 requests, plus ~10 reconnects. Syncs add ~2 requests each (read +
publish).

| Viewers all day | ≈ Requests/day |
|---|---|
| 200 | ~10k |
| 500 | ~24k |
| 2,000 | ~93k — at the cap |

Assume auto-answered pings count toward requests (the docs say they avoid
*duration* charges; treat them as requests until the dashboard shows
otherwise, §10 check 4).

At the cap, further requests fail with an error — never a bill. Pages then
stay on GitHub polling: today's speed, not an outage. Limits reset at 00:00
UTC, which is **08:00 in Manila**, mid-morning on an event day; budget a
multi-day event by UTC day.

Other constraints:

- **Worker deploys and Cloudflare maintenance** drop every socket; clients
  reconnect with backoff and receive the current snapshot on connect.
- **Phones** drop sockets in the background; `pause()`/`resume()` on
  visibility handles it. Wall screens stay connected.
- **Some venue networks** block WebSockets; those clients stay on polling.
- **Anyone** can open a socket to any valid-looking key; unknown keys just
  get `{type:"empty"}` and cost a request each. The `Origin` check stops
  other websites embedding it, not scripts. Acceptable at this scale: the
  data is already public on GitHub Pages, and exhausting the quota only
  pushes pages back to polling.
- **Portability:** Durable Objects exist only on Cloudflare. Leaving means
  rewriting Phase 1; the GitHub fallback keeps the site working meanwhile.

## 10. Checks before and during the build

| # | Check | When | If it fails |
|---|---|---|---|
| 1 | `locationHint: "apac-se"` accepted | Phase 1 | Use `"apac"` |
| 2 | The on-connect message sent before the `101` arrives | Phase 1 (`smoke.mjs` step 2) | Send it after connect (§5.4 note) |
| 3 | Cloudflare keeps a socket open ≥ 60s with 50s pings | Phase 1, with a browser tab left open 10 min | Lower `LIVE_PING_MS` |
| 4 | Whether auto-answered pings show up as Durable Object requests | After a day of real use, in the Cloudflare dashboard | Budget stays as §9 either way; if they don't count, the ceiling is ~5× higher |

The prerequisite spec has its own checks (which Google account owns the
triggers, and what its measurements show).

## 11. Acceptance checklist

**Prerequisite**

- [ ] The [Immediate sync](immediate-sync-spec.md) spec's acceptance
      checklist passed when it shipped, and §4's table above holds.

**Worker (Phase 1)**

- [ ] `smoke.mjs` passes against the deployed Worker.
- [ ] A connection from `https://example.com` is refused (`403`).
- [ ] Publish without the secret → `401`.

**Cloud Run (Phase 2)**

- [ ] `node scripts/verify-sync-merge.mjs` passes (the Immediate sync scenarios and Phase 2's).
- [ ] With `LIVE_PUSH_URL` unset, a real sync behaves exactly as before.
- [ ] With it set, a real sync returns `live.published: true` and a new
      GitHub commit (`archive.committed: true`).
- [ ] **Concurrent race:** two facilities' syncs fired at the same moment —
      the final snapshot holds both facilities' new data.
- [ ] Worker unreachable (wrong URL on a test revision) → sync still
      succeeds through GitHub with `live.published: false`.
- [ ] `POST /sync/:day/live` with `false` → the pushed snapshot has
      `isLive: false`; then `auto` restores it.

**Pages (Phase 3)**

- [ ] Public page, wall board and Control Center each receive an edit within
      ~5s of pressing Enter (stopwatch, three edits).
- [ ] **Force hidden** hides the public page within ~3s; `auto` restores it.
- [ ] Switching day on the public page and Control Center never shows the
      previous day's data.
- [ ] Hide the tab 2 minutes, show it → data is current within ~2s.
- [ ] Block the Worker's host (DevTools request blocking) → the page keeps
      updating from GitHub at the old speed; unblock → push resumes without
      a reload.
- [ ] Control Center on `localhost?fixture=…` makes no WebSocket connection.
- [ ] Mission Control shows **Live updates: push connected**.
- [ ] Wall board keeps its scroll position across pushed updates.
- [ ] Every copy of the live-channel block is byte-identical
      (compare them with `diff`).

## 12. Rollout order and rollback

1. The [Immediate sync](immediate-sync-spec.md) spec, shipped and measured
   at an event or rehearsal.
2. Phase 1: deploy the Worker; run `smoke.mjs`.
3. Phase 2: deploy with `LIVE_PUSH_URL` unset; then set it. GitHub still
   gets every snapshot, so pages don't notice yet.
4. Phase 3: ship the pages. Rehearse with the dry-run checklist.
5. Phase 4.

None of it on an event day, or within a few days before one.

**Rollback**, fastest first:

- Pages misbehave: set `LIVE_BASE_URL = ''` in the affected files and
  commit; they poll GitHub as before.
- Publishing misbehaves: clear `LIVE_PUSH_URL` on the Cloud Run service (a
  new revision, no code change); pages fall back to GitHub automatically
  through the safety poll and reconnect attempts.

Rolling this spec back never requires undoing the prerequisite.

## 13. Out of scope

- Retiring GitHub as archive or fallback.
- `tools/scoresheet-generator.html` (one fetch per day pick; stays on GitHub).
- Pages under `events/archives/`.
- Per-viewer authentication for the live socket (the data is public).
- A GitHub Action to deploy the Worker; it is deployed by hand.
- Sending diffs instead of whole snapshots (≤ 45 KB is fine).
