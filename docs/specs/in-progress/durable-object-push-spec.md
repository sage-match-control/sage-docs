# Spec — Live push delivery (Durable Objects)

> **Status: deployed and switched on; the real-use checks are open.** All four
> phases are built and live (2026-10-01 to 02: `sage-tools-api` 2.5.0,
> the `sage-live` Worker on Cloudflare, `LIVE_PUSH_URL` and
> `LIVE_PUSH_SECRET` on Cloud Run, `LIVE_BASE_URL` set in all nine pages).
> Checked live: the Worker's `smoke.mjs`, a sync from each event publishing to
> the Worker and archiving to GitHub with the same `publishedAt`, and the three
> kinds of page (public, schedule board, Control Center) holding an open socket
> to the production Worker. Not done: the real-workbook checks in §11 (the
> two-workbook race, stopwatch timings, **Force hidden**, the blocked-Worker
> fallback, the 10-minute tab, the ping-count check, and the **Sync method**
> switch with a real sign-in). Both events ran on live push on 3 October
> 2026: every one of their 933 syncs published to the Worker and archived to
> GitHub, and the server side of the ~5 s target held at Piggleball
> (`edit→published` p90 5.2 s) but not at PickleDrive (p90 27.3 s), whose
> workbook's Sheets reads were slow (§11's notes). Piggleball and PickleDrive have since finished
> (2026-10-03), and their four pages now carry `LIVE_BASE_URL = ''`; Control
> Center and the two templates keep it set, so the open checks wait for the
> next event's workbooks. The spec moves to `implemented/` when those are
> done. [§14](#14-as-built-divergences) records
> where the build departs from the text below. Revised 2026-10-01 against
> `sage-tools-api` 2.3.0 (the prerequisite below, built) and the
> `sage-match-control.github.io` pages as of that date.
>
> **This is one of two alternative designs.** The other is
> [Fast data delivery](../archived/fast-data-delivery-spec.md) (Cloudflare R2
> behind a CDN, pointer polling), which this one supersedes. Build one, not
> both.
>
> **Prerequisite, already built:** the
> [Immediate sync](../implemented/immediate-sync-spec.md) spec — timing
> instrumentation, a retry for syncs that lose a GitHub commit race, and an
> Apps Script trigger that syncs straight from the edit. It shipped in
> `sage-tools-api` 2.3.0. §4 lists exactly what it left in the code for this
> spec to build on; confirm that list before starting.
>
> Plain-language version: [explainer](durable-object-push-explainer.md).

Make a score typed into a facility sheet appear on every open page within a
few seconds, for free, by pushing each new snapshot to open pages over a
WebSocket from a Cloudflare Durable Object, instead of waiting for GitHub
Pages to rebuild and for pages to poll. (The Apps Script side already syncs
straight from the edit: that was the prerequisite spec.)

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
- **Never deploy any part of this on an event day, or in the few days
  before one.** It replaces the live data path.
- Working copies use CRLF line endings (Git's `autocrlf`). Keep them: an
  editor that rewrites a whole file to LF turns a small diff into a
  whole-file one.

**How the live data flows today:**

```
Facility Google Sheet (one per venue per tournament day)
  | installable onEdit trigger -> onEditInstallable -> syncUntilSettled_
  |   (scripts/sheets-sync.gs): takes the workbook's document lock, waits
  |   SYNC_SETTLE_MS (1.5s) — stretched so syncs start at least
  |   SYNC_MIN_GAP_MS (5s) apart — syncs, and repeats while newer edits keep
  |   arriving. Edits that can't take the lock just record their time.
  v
POST /sync/:day?facility=<name>   Cloud Run, X-Sync-Secret + X-Edit-At headers
  | SyncService.syncDay: fetch that facility's CSV + STANDINGSCSV tabs
  | (Sheets API), then up to COMMIT_ATTEMPTS (3) times: read the published
  | snapshot back from GitHub, merge (#buildSnapshot), commit with its sha;
  | a 409 goes round again
  v
GitHub Contents API commit -> event-data/<event-key>/data/<day>.json
  | GitHub Pages build + deploy (25s p50, 52s p90)
  v
Pages poll https://sage-match-control.github.io/event-data/<event-key>/data/<day>.json
every 10s (POLL_INTERVAL_MS), paused while the tab is hidden
```

Control Center also calls `POST /sync/:day` (no `?facility=`, a full
resync, operator bearer token) and `POST /sync/:day/live` (the Live/Hide
override: `SyncConfigStore.setIsLive` commits `config/events.json`, then
`SyncService.setLiveOverride` republishes the day's snapshot with only
`isLive` changed). Both commits retry a `409` the same way `syncDay` does.

**Checks you can run** (from `sage-tools-api/`; plain Node, no framework,
each exits non-zero on failure):

```bash
node scripts/verify-sync-merge.mjs
```

```bash
node scripts/verify-facility-completion.mjs
```

The site has no tests. Serve `sage-match-control.github.io/` with any static
file server (for example `npx http-server` from that folder) and open the
page; Control Center and the PickleDrive pages also accept `?fixture=<name>`
on localhost to load `_fixtures/` instead of live data.

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

Measured at CLSO Pickle for Sight (27 September 2026, 2 facilities) and on
the Piggleball workbook after the prerequisite shipped (1 October 2026, ~30
syncs; `technical/sync-pipeline.md` § Measured: the lock-based sync):

| Leg | p50 | p90 | max | Source |
|---|---|---|---|---|
| Edit → request reaches Cloud Run (`edit→request`, includes the 1.5s settle) | ~1.5s | ~3s | 4.7s | Piggleball `timing` log lines. The old time-based trigger took 74s p50, 119s max |
| Cloud Run `POST /sync/:day` (264 calls) | 1.6s | 1.85s | 19s | Pickle for Sight request logs |
| Edit → committed to GitHub (`edit→published`) | ~3.5s | ~4.5s | 6.1s | Piggleball `timing` log lines |
| GitHub commit → Pages deployed (287 builds, 46 cancelled by newer pushes) | 25s | 52s | 82s | `event-data` Actions runs, Pickle for Sight |
| Page poll | 5s | 10s | 10s | `POLL_INTERVAL_MS = 10000` |

Edit → screen today: ~35s typical (3.5 + 25 + 5), ~65s at p90, 90s+ worst.
Nearly all of it is the Pages build and the poll, the two legs this spec
removes. Target after all phases: **~2–5s typical, ~8–10s worst** (a Cloud
Run cold start).

| Leg after this spec | Estimate |
|---|---|
| Edit → sync starts (built, measured) | 1–2s including the 1.5s settle; up to ~5s more when a sync from the same workbook started just before (`SYNC_MIN_GAP_MS`) |
| Cloud Run fetch + merge + push (Phase 2) | ~1.5–2.5s (Sheets fetch ~1s, plus two ~200ms round trips to the Durable Object) |
| Durable Object → every open page (Phase 1, 5) | < 0.5s |

**Concurrent writes.** Every facility of a day writes into the same
`<event>/data/<day>.json`, and every commit of every event moves the one
`main` branch of `event-data`, so syncs race. A commit made with a stale sha
is rejected with `409 Conflict`. Before the prerequisite the loser returned
`500` and its update was lost until the next edit to that sheet (all four
failed syncs at Pickle for Sight). The prerequisite made the GitHub path
re-read, re-merge and retry (§4). The Durable Object has the same race — two
syncs read version `n` and both publish with `expectedVersion: n` — and
answers the loser with its own `409`; Phase 2 must retry that the same way
(§6.5). A test that loses a facility's data here is a failed build.

**Commit volume.** Every sync is a GitHub commit and a Pages build. GitHub
allows 80 content-creating requests a minute and 500 an hour
([rate limits](https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api));
`SYNC_MIN_GAP_MS` holds one workbook to about 12 commits a minute. Over the
limit GitHub answers `403` or `429`. After Phase 2 those commits are only the
archive: a rate-limited archive commit is logged and dropped without
affecting what viewers see, and the next sync's archive catches GitHub up
(§6.5).

## 2. Target architecture

```
Facility Google Sheet
  | installable onEdit -> onEditInstallable -> syncUntilSettled_ (built)
  |   document lock, 1.5s settle, syncs at least 5s apart; catches up if more edits land
  v
POST /sync/:day?facility=<name>  + X-Edit-At header (built)
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
| — | **Prerequisite, built:** [Immediate sync](../implemented/immediate-sync-spec.md) — timing, commit-conflict retry, lock-based Apps Script sync | `sage-tools-api`, `sheets-sync.gs`, `control-center.html` | `sage-tools-api` 2.3.0 |
| 1 | Worker + Durable Object | `sage-tools-api/live-worker/` | `wrangler deploy` (manual) |
| 2 | Cloud Run publishes to the object | `sage-tools-api/src/` | Minor version bump |
| 3 | Pages subscribe | `sage-match-control.github.io` | Site commit |
| 4 | Documentation and runbooks | `sage-docs`, `CLAUDE.md`s, checklist template | Commits |

The prerequisite has shipped and been measured (§1), so Phase 1 can start.
Phase 2 is `sage-tools-api` 2.3.0 → **2.4.0**, unless another release has
shipped in between (check `package.json`).

---

## 4. Prerequisite — Immediate sync

Everything this spec builds on is in the
[Immediate sync](../implemented/immediate-sync-spec.md) spec, which is built.
Confirm each of these exists in the code before starting Phase 1. If one is
missing, stop and ask: it means the code has moved on since this revision.

| What | Where to look |
|---|---|
| `X-Edit-At` header sent by Apps Script and read by Cloud Run (accepted only within the last hour and at most 5s in the future) | `scripts/sheets-sync.gs` `triggerSync_(opts)`; `src/sync/routes.mjs` `handleSync` |
| `facilities[].lastEditAt`, and `attempts` plus `timing: { editToRequestMs, fetchMs, publishMs, editToPublishedMs }` in the sync response (the two `edit…` fields `null` without an edit time) | `src/sync/SyncService.mjs` `syncDay` |
| `GitHubPublisher.publish` throws errors carrying `status`, and the module exports `COMMIT_ATTEMPTS = 3` | `src/sync/GitHubPublisher.mjs` |
| The `COMMIT_ATTEMPTS` loop on `err.status === 409` in `syncDay`, `setLiveOverride` and `SyncConfigStore.setIsLive` | `src/sync/SyncService.mjs`, `src/sync/SyncConfigStore.mjs` |
| `#buildSnapshot({ day, label, isLive, now, allFacilities, targetFacilities, freshByName, failed, existing, lastEditAt })` returning `{ snapshot, stale }`. `existing` is `{ json, sha }` as `fetchExisting` returns it (only `.json` is read); `lastEditAt` is an ISO string or `null` | `src/sync/SyncService.mjs` |
| `scripts/verify-sync-merge.mjs`: eight scenarios built on its `FakePublisher` class (a `files` map, `set(path, json)`, `failNext(status, mutate)`, `publishCalls`), `makeService(publisher)` and `check(label, actual, expected)` | `sage-tools-api/scripts/` |
| `syncUntilSettled_`, `syncWithRetry_`, `syncWaitMs_`, `SYNC_MIN_GAP_MS` | `scripts/sheets-sync.gs` — nothing in this spec changes it |

`syncDay` today runs, in order: resolve the day from `configStore`, pick the
target facilities, fetch them (timed as `fetchMs`), build `freshByName` and
`failed`, compute `lastEditAt` (only for a facility-scoped sync that has an
`editAt`), run the read → `#buildSnapshot` → `publish` loop (timed as
`publishMs`), then build `timing`, log
`timing edit→request=… fetch=…ms publish=…ms edit→published=…` through a local
`ms()` helper that prints `n/a` for `null`, and return. Phase 2 changes
only the loop and what follows it.

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

Already done by the [Immediate sync](../implemented/immediate-sync-spec.md) spec (its §3.3): a failed PUT throws an error carrying
`status`, so `409 Conflict` can be told apart from other failures.

### 6.4 `SyncService` — constructor

Switch the constructor to one options object (it is already five positional
arguments):

```js
constructor({ sheetsApiFetcher, gvizFetcher, publisher, livePublisher, configStore, logger })
```

Store each as a same-named property (`this.livePublisher` and so on).
Treat a missing `livePublisher` as disabled (`this.livePublisher?.enabled`),
so a caller that passes none gets exactly today's behaviour.
There are two call sites:

- `index.mjs`, where `new SyncService(...)` is built after
  `syncConfigStore`. Construct `LivePublisher` there from the three env
  vars (`LIVE_PUSH_TIMEOUT_MS` parsed like `SHEETS_FETCH_TIMEOUT_MS` is, and
  left `undefined` when unset so the default applies) with
  `syncLogger.child("live")`, and pass it in.
- `scripts/verify-sync-merge.mjs`'s `makeService` (§6.7).

`SyncConfigStore` keeps its constructor. Add `live: { enabled, baseUrl }` to
`handleConfigDiagnostics` in `routes.mjs`, read from
`syncService.livePublisher` (the router already receives `syncService`);
never include the secret. Document the field in that route's `@openapi`
block.

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

**Mapping the pseudo-code onto the current code:**

- "today's GitHub path" is the existing read → `#buildSnapshot` → `publish`
  loop in `syncDay`, unchanged. Move it into a private method (for example
  `#publishViaGitHub(...)`, returning `{ snapshot, stale, publishResult,
  attempts }`) so the live-disabled branch and the fallback branch both call
  it.
- `merge(fresh, existingJson)` is
  `this.#buildSnapshot({ ..., existing: { json: existingJson }, lastEditAt })`
  — the same method, with the Durable Object's snapshot wrapped the way
  `fetchExisting` returns a file. It still throws `SyncUpstreamError` when
  there is nothing to publish; let that propagate as it does today.
- Set `snapshot.publishedAt = new Date().toISOString()` immediately before
  **every** publish, live or GitHub, and in `setLiveOverride` too. Pages use
  it to decide which of two copies is newer.
- `attempts` counts live publish attempts when the live publish succeeded,
  and the GitHub loop's attempts otherwise.
- `timing` keeps its four fields and adds `liveMs` and `archiveMs` (`null`
  when that step didn't run). `publishMs` stays the whole time from the first
  read to the end of the archive. The log line becomes
  `timing edit→request=… fetch=…ms publish=…ms live=…ms archive=…ms edit→published=…`,
  with `n/a` for any `null`.
- Archive failures include GitHub's rate-limit answers (`403` or `429`):
  log them and return `{ committed: false, error }`; the request still
  succeeds.

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
archiveToGitHub(path, snapshot, message)   # the §6.5 helper
return { day, label, isLive, republished: true, live, archive }
```

"Today's GitHub path" here is `setLiveOverride`'s existing
`COMMIT_ATTEMPTS` loop, unchanged, used when live push is disabled or a
live call throws. Three live version conflicts in a row also fall back to it.

Without this, a Live/Hide click would only reach GitHub, and every page
connected by WebSocket would keep showing the old state until the next
score edit. **Force hidden must work through the push path.**

### 6.7 Extend `scripts/verify-sync-merge.mjs`

The [Immediate sync](../implemented/immediate-sync-spec.md) spec created
the script (its §3.3). Read it first: its eight scenarios are numbered
blocks, each building a `FakePublisher` and a service with
`makeService(publisher)`.

- Change `makeService` to `makeService(publisher, livePublisher = null)`
  and construct `SyncService` with the options object, passing a disabled
  fake (`enabled: false`) when none is given, so the eight existing
  scenarios run unchanged on the GitHub path.
- Add a `FakeLivePublisher` in the same style: `enabled`, a stored
  `{ version, snapshot }`, `read()` and `publish()` with the
  §6.2 return shapes, a `failNext(kind, mutate)` queue where `kind` is
  `"conflict"` (return a `409`-shaped result after running `mutate`) or
  `"throw"`, and a `publishCalls` counter.

Keep the eight scenarios passing, and add:

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

Bump the minor version in `package.json` (2.3.0 → 2.4.0) with a Changelog
entry in `README.md`. Deploy by pushing to `main`, with
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

Update the pages of every event that has not finished when Phase 3 ships;
a finished event's pages stay on polling and keep working unchanged. As of
this revision, `events/` holds `pickle-for-sight-2026` (27 September 2026)
and `pnf-x-bup-dual-meet`, both finished, and `piggleball-2026` and
`pickledrive-anniversary-2026` (both 3 October 2026). Each event's spec in
`sage-docs/docs/specs/` gives its dates. `pickledrive-anniversary-2026` was
hand-built, not instantiated from a template, and both its pages define
`FIXTURE`. An event created after this revision is covered by the
templates.

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

Also correct the stale comments about the old Apps Script debounce in every
file this phase touches: the one above `POLL_INTERVAL_MS` ("edit debounce is
~10s" / "debounce is ~10s") and, in each `index.html` and Control Center, the
one inside `loadLiveData` that says "its debounce settled". `grep -n -i
debounce <file>` finds both. Apps Script now syncs within a few seconds of
the edit (`SYNC_SETTLE_MS`, `SYNC_MIN_GAP_MS`).

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
  the system". The line "Each edit triggers Apps Script's own debounced
  sync" (around line 95) is stale too: reword it to say each edit syncs
  within a few seconds and reaches open pages by push.
- Root `CLAUDE.md`'s "Things that must be kept in sync by hand": add the
  live-channel block, which must stay byte-identical in every page that
  carries it.
- `scripts/sheets-sync.gs`'s Help dialog (`showSyncHelp`) tells operators
  scores reach the website "about 10 seconds after you stop typing" and
  take "40 to 60 seconds to appear". Once Phase 3 ships, change both to
  "within a few seconds". A `.gs` change is not a deploy and bumps no
  version: it ships by pasting the file into every live workbook and both
  masters.
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

The prerequisite's own checks are done: the account that owns the Apps
Script triggers is a consumer Google account (90 minutes of trigger runtime
a day), and its measurements are in §1.

## 11. Acceptance checklist

**Prerequisite**

- [x] The [Immediate sync](../implemented/immediate-sync-spec.md) spec is
      built and measured (Piggleball workbook, 1 October 2026). Its
      multi-workbook and multi-event collision checks were never run on
      real workbooks; Phase 2's **Concurrent race** check below covers the
      same ground, so run that one with two real workbooks.
- [x] §4's table holds in the code.

**Worker (Phase 1)**

- [ ] `smoke.mjs` passes against the deployed Worker.
- [ ] A connection from `https://example.com` is refused (`403`).
- [ ] Publish without the secret → `401`.

**Cloud Run (Phase 2)**

- [x] `node scripts/verify-sync-merge.mjs` passes (the Immediate sync scenarios and Phase 2's).
- [ ] With `LIVE_PUSH_URL` unset, a real sync behaves exactly as before.
- [x] With it set, a real sync returns `live.published: true` and a new
      GitHub commit (`archive.committed: true`). All 933 syncs of
      3 October 2026 (Piggleball, PickleDrive) logged a `live` and an
      `archive` time, and each has its commit in `event-data`.
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
- [x] Control Center on `localhost?fixture=…` makes no WebSocket connection.
- [ ] Mission Control shows **Live updates: push connected**.
- [ ] Wall board keeps its scroll position across pushed updates.
- [x] Every copy of the live-channel block is byte-identical
      (compare them with `diff`).

**Notes from the 3 October 2026 events** (figures in
[Sync pipeline](../../technical/sync-pipeline.md#measured-at-two-events-3-october-2026)):

- The server side of the ~5 s check: `edit→published` (measured when the
  live publish succeeds) was p50 3.2 s, p90 5.2 s at Piggleball and p50
  3.6 s, p90 27.3 s at PickleDrive. The Worker leg was 0.5–0.6 s at both.
  PickleDrive's tail is its workbook: 161 of 695 Sheets API reads took
  10–58 s. Production runs with `SHEETS_FETCH_TIMEOUT_MS=0` (no read
  timeout), so those reads were waited out, and the 24 syncs that ran past
  Apps Script's 30 s `fetchTimeoutSeconds` were sent a second time while the
  first was still running. The page leg (push to screen) was not timed; the
  stopwatch check still needs a person.
- **Concurrent race:** still not run with two facilities, both events having
  one venue. The same code path ran for real, though: overlapping syncs of
  one facility (most likely a slow sync and Apps Script's resend of it) caused 2 live
  version conflicts, both re-read and re-merged on the first retry, and 8
  archive conflicts across the two events, all retried with the live
  snapshot on the first attempt.

## 12. Rollout order and rollback

1. Done: the [Immediate sync](../implemented/immediate-sync-spec.md) spec,
   shipped in 2.3.0 and measured on the Piggleball workbook.
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

## 14. As built: divergences

Where the built code departs from the text above. Everything else was built
as written.

- **`LIVE_BASE_URL` was set after the Worker existed.** §7.2's block has
  `wss://sage-live.<subdomain>.workers.dev`, which is not a valid WebSocket URL
  (`new WebSocket` would throw inside `selectDay`), so the pages shipped with
  `''`, which the block treats as "push disabled". Once the Worker was deployed
  the constant was set to `wss://sage-live.sagematchcontrol.workers.dev` in all
  nine pages in one pass, and the copies were confirmed identical (§11's `diff`
  check).
- **§7.3's "fetch failed but a push exists" case** uses
  `liveChannel.newer(event, day, null)`, which returns the pushed snapshot
  (or `null`), so the block itself is unchanged and stays identical across
  files.
- **`LIVE_PUSH_TIMEOUT_MS`** is read with a truthiness test, not
  `!== undefined` as `SHEETS_FETCH_TIMEOUT_MS` is. A variable that is set but
  empty, as `.env.example` leaves it, would otherwise parse to `0` and abort
  every Worker call immediately.
- **The sync response omits `live` and `archive` when live push is
  disabled**, rather than returning nulls, so a disabled service answers
  exactly as it did before. `archive` is also omitted when the sync fell back
  to GitHub (that commit is the publish, reported as `commitSha`).
  `setLiveOverride` likewise returns `live: { published: false, error }` when
  the Worker failed and the republish went through GitHub.
- **Archive retry when the Durable Object cannot be re-read.** §6.5 retries a
  `409` with the object's current snapshot. If that read itself fails, the
  retry commits the snapshot it already has; the next sync's archive catches
  GitHub up either way.
- **A failed GitHub seed read propagates.** When the object is empty, the merge
  seeds from GitHub as §6.5 says; if that read throws, the sync fails as it
  would without live push rather than falling back (the fallback would need the
  same read).
- **The Mission Control line** is an ordinary status row: green dot for
  **push connected**, grey for **polling GitHub**, below the facility rows. It
  is grey rather than a warning colour because polling is correct, only slower.
- **`timing.liveMs`** covers the whole live loop: the reads, the merge and the
  publishes of every attempt, including a GitHub seed read on the first sync
  of a day.
- **An operator switch (2.5.0).** Control Center's **Sync method** switch
  (`POST /sync/live-push`, `livePush` in `config/events.json`) turns live
  push off and on without clearing `LIVE_PUSH_URL` on Cloud Run. It lives in the
  config rather than the Worker so it works when the Worker is the problem.
- **The live merge builds on the newer of the object's and GitHub's snapshot
  (2.5.0), not the object's alone.** §6.5 reads GitHub only when the object is
  empty. After syncs that went through GitHub alone (the switch off, a Worker
  outage), the object holds an older copy; merging into it would republish, and
  archive over GitHub, facility data GitHub already has newer. The two are now
  read together, so it costs no extra time, and the newer by `publishedAt` wins.
  Live/Hide does the same.
- **Checked locally, not in production.** `smoke.mjs` passed against
  `wrangler dev`, which also confirmed §10 check 2 (the on-connect message
  arrives) and that a wrong `Origin` gets `403`, a missing secret `401` and a
  plain request `426`. Checks 1, 3 and 4 need the deployed Worker. In a
  browser, a pushed snapshot reached the public page, the schedule board and
  Control Center within about a second, and pushing `isLive: false` flipped the
  public page hidden at once. The two-workbook race and the stopwatch checks in
  §11 have not been run.
