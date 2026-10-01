# Spec — Fast data delivery

> **Status: superseded, not going to be built.** [Live push delivery](../in-progress/durable-object-push-spec.md)
> was chosen and built instead; this spec is kept for its reasoning and its
> measurements. It was written as "not started, with one exception".
>
> **Status as written: not started**, with one exception. Step 16's debounce change is
> already made: `scripts/sheets-sync.gs` has `DEBOUNCE_MS = 3000`, and a
> workbook runs it once its copy of the file is the current one. Nothing
> else here is built.
>
> **This is one of two alternative designs.** The other is
> [Live push delivery](../in-progress/durable-object-push-spec.md), which pushes each
> snapshot to open pages over a WebSocket from a Cloudflare Durable Object
> instead of having them poll R2. Build one, not both.
>
> **Prerequisite:** the [Immediate sync](../implemented/immediate-sync-spec.md) spec — timing instrumentation, a retry for
> syncs that lose a GitHub commit race, and an Apps Script trigger that
> syncs straight away. Both delivery designs need it, and it is useful on
> its own. Build it first; §0 lists what to check for.
>
> Revised 2026-10-01 against the code as of `sage-tools-api` 2.2.0: the
> snapshot's new fields (§3, §4.1), the Live/Hide republish as a second
> writer (§5), the client files that exist now (§8), and the Pickle for
> Sight measurements (§1).

Replace the GitHub Pages build/deploy step in the live data path with Cloudflare
R2, and replace full-payload polling with pointer polling.

## 0. Read this first (implementer orientation)

Assume no knowledge of this project beyond this page and the prerequisite
spec.

**Repos.** `D:\Personal\SAGE` is a plain folder holding four git repos:

| Repo | Role here |
|---|---|
| `sage-tools-api/` | Node 22 / Express backend on Google Cloud Run (`us-central1`). ESM `.mjs`, classes with constructor injection, no TypeScript, no build step, no test framework (plain-Node `scripts/verify-*.mjs` checks). Deploys automatically on push to `main`. The R2 publisher goes in `src/sync/` |
| `sage-match-control.github.io/` | Static site on GitHub Pages. Every page is one self-contained HTML file (inline `<style>` and `<script>`), no shared JS files. Commit to deploy. The pages in §8 change |
| `event-data/` | Public repo served by GitHub Pages: `config/events.json` (the event/day/facility registry) and every day's snapshot at `<event-key>/data/<day>.json`. Stays as the archive and fallback |
| `sage-docs/` | Documentation (mkdocs). This spec lives here |

**Rules for every change** (from the root `CLAUDE.md`):

- A code change in `sage-tools-api` bumps `package.json`'s version (minor
  for a feature) and adds a matching Changelog entry in its `README.md`.
- Cite a spec from outside `sage-docs` as
  `sage-docs/docs/specs/.../fast-data-delivery-spec.md` — literally `...`
  where the status folder goes.
- Documentation is written in the present tense.
- **Never deploy any of this on an event day**, or within a few days
  before one.

**How the live data flows today:** a facility sheet's edit triggers
`scripts/sheets-sync.gs` (Google Apps Script, pasted into each workbook by
hand, not deployed with the service), which calls
`POST /sync/:day?facility=<name>` on Cloud Run. `SyncService.syncDay` fetches
that facility's tabs through the Sheets API, reads the published snapshot
back from GitHub, merges, and commits it to `event-data`. GitHub Pages
rebuilds, and every page polls
`https://sage-match-control.github.io/event-data/<event-key>/data/<day>.json`
every 10s. Control Center also calls `POST /sync/:day` for a full resync and
`POST /sync/:day/live` for the Live/Hide override (§5).

**Prerequisite check.** Confirm these exist before starting; if any is
missing, build the [Immediate sync](../implemented/immediate-sync-spec.md) spec first:

| What | Where to look |
|---|---|
| `facilities[].lastEditAt` and a `timing` object in the sync response | `src/sync/SyncService.mjs` `syncDay` |
| `GitHubPublisher.publish` throws errors carrying `status` | `src/sync/GitHubPublisher.mjs` |
| 3-attempt re-read-and-re-merge on `409` in `syncDay`, `setLiveOverride`, `SyncConfigStore.setIsLive`, and a private `#buildSnapshot(...)` | `src/sync/SyncService.mjs`, `src/sync/SyncConfigStore.mjs` |
| `scripts/verify-sync-merge.mjs` | `sage-tools-api/scripts/` — extend it for R2 the way §12's write-path checks describe |
| Lock-based `syncUntilSettled_` / `syncWithRetry_` | `scripts/sheets-sync.gs` |

This spec's R2 retry loop (§5) replaces the GitHub `409` loop as the
correctness guarantee on the live path; the GitHub loop stays for the
archive commit and for the fallback when R2 is unreachable.

| | Now | Target |
|---|---|---|
| Edit → visible | ~35–40s typical, 90s+ worst (§1 check 1) | ~5–7s, **only** with the Apps Script fix (§7) |
| Client poll transfer | 4–6 KB gzipped | ~100 bytes |
| Client poll interval | 10s | 3s |
| Apps Script debounce | 3s (already — see status) | Replaced by a lock (§7) |

**Store:** Cloudflare R2 behind a custom domain.
**Write path:** Cloud Run. Apps Script stays a thin trigger.
**Archive:** GitHub Pages retained as durability backup and aged-out fallback.

The dominant measured term today is the GitHub Pages rebuild (p50 25s, p90
52s), which R2 removes entirely. The poll-interval change is worth a further
~3.5s on average. The Apps Script trigger's own delay measured a median of ~74s (up to ~2 min) and is
larger than the Pages build — R2 does nothing about it, which is why §7's
prerequisite exists.

**The custom domain is not cosmetic — it is what makes this affordable.** See
§9.4: Cloudflare's CDN in front of R2 absorbs client polling, so billed reads are
flat regardless of audience size. Without it (on `r2.dev`, or on any store
without a CDN) cost scales with viewers × time.

**§1 is a go/no-go gate. Do not write code before completing it** — two of its
checks can invalidate the design.

---

## 1. Verify first — blocking

Complete all six before implementing anything. Record the answers in the PR or
commit message.

| # | Check | Why it blocks |
|---|---|---|
| 1 | **Measure the real latency split** — time an edit through debounce → sync → Pages deploy → client poll | **Partly done** (table below). The Apps Script leg measured a median of ~74s, so it dominates: §7 alone captures most of the win for a fraction of the work, and this project should be re-scoped or dropped |
| 2 | **Confirm R2 supports `If-Match` / `If-None-Match` on `PutObject`** for this account, against the live bucket | **Hard gate.** Without conditional writes, §5 causes silent data loss. If unavailable, do not implement §6.3 — keep the merge-forward read on GitHub, where its `sha` CAS still applies, and accept the slower read |
| 3 | **Confirm the Cache Rule actually caches the pointer** — fetch it twice through the custom domain and check for `cf-cache-status: HIT` | **Hard gate, and silent if wrong.** Cloudflare does not cache JSON by default. Without a working Cache Rule every client poll becomes a billed origin read and cost scales with viewers × time instead of staying flat (§9.4). Everything still *works*, which is why this must be checked explicitly rather than assumed |
| 4 | **Confirm a suitable domain is available** as a zone in the same Cloudflare account | R2 custom domains require it; `r2.dev` is not an acceptable substitute — it is rate-limited and cannot carry Cache Rules, which forfeits check 3 entirely (§9.1) |
| 5 | **Verify `@aws-sdk/client-s3` checksum behaviour** against the version actually installed | Recent versions send checksums R2 rejects (§6.1) |
| 6 | **Measure the SDK's cold-start import cost** on a sync-only request | If significant, lazy-import it or drop the SDK for plain `fetch` + SigV4 (§6) |

Check 1's measured legs, from CLSO Pickle for Sight (27 September 2026, 2
facilities). Cloud Run from its request logs; Pages from the public Actions
runs of `sage-match-control/event-data`, commit to completed deploy:

| Leg | p50 | p90 | max | Source |
|---|---|---|---|---|
| Apps Script: edit → `runIfSettled` fires | 74s | 119s | 119s | Piggleball Executions page, 1 Oct 2026, five bursts (22–119s). See `technical/sync-pipeline.md` § Baseline |
| Cloud Run `POST /sync/:day` (264 Apps Script calls) | 1.6s | 1.85s | 19s | `gcloud logging read`, `httpRequest.latency` |
| GitHub commit → Pages deploy done (287 builds, 46 cancelled by a newer push) | 25s | 52s | 82s | `GET /repos/sage-match-control/event-data/actions/runs` |
| Client poll | 5s | 10s | 10s | `POLL_INTERVAL_MS = 10000` |

Cancelled builds matter: a push while a build is running cancels it and
starts again, so during busy stretches nothing becomes visible until pushes
pause for ~25s.

Payload size is already measured and needs no further checking: the largest real
day (`bkl-cup-2026/day2`, 3 facilities) is **45.2 KB raw / 6.3 KB gzipped**;
`day3` is 21.9 KB / 4.0 KB. Pointer polling still cuts ~6 KB per poll to ~100
bytes, but **the case for this change is latency, not bandwidth** — do not
justify it on transfer volume.

---

## 2. Architecture

```
Facility Google Sheet  (one per venue per tournament day)
   |  installable onEdit trigger, lock-based  (scripts/sheets-sync.gs, §7)
   v
POST /sync/:day?facility=<name>   on Cloud Run   (X-Sync-Secret header)
   ...or POST /sync/:day/live from Control Center (operator token) — the
   Live/Hide override, which republishes the same snapshot with only
   isLive changed and must go through the same steps 2 and 5–7 (§5)
   |  1. resolve event/day via SyncConfigStore  (event-data/config/events.json)
   |  2. read R2 pointer (+ETag) and the payload it names
   |  3. fetch facility CSVs (Sheets API, or gviz via ?method=csv)
   |  4. merge-forward untouched/failed facilities
   |  5. hash content; unchanged -> refresh pointer's updatedAt only, stop
   v
   |--> PUT  <event>/data/<day>-<hash>.json   (R2, immutable)
   |--> PUT  <event>/pointer/<day>.json       (R2, max-age=2)  <-- LAST in R2
   |      If-Match: <etag>  -> on 412, re-read and redo the merge (§5)
   |--> commit <event>/data/<day>.json        (GitHub, archive)  <-- after pointer
   v
Client polls  <event>/pointer/<day>.json  every 3s  (~100 bytes)
   |  hash changed?
   v
Client fetches <event>/data/<day>-<hash>.json  (immutable, CDN-cached forever)
   |  on 404, network error, or 5xx -> GitHub Pages fallback (per-fetch)
   v
render
```

---

## 3. Data contract

Two objects per tournament day, both **event-scoped** — mirroring
`SyncConfigSnapshot.repoPathFor(event, day)`, which yields
`<event>/data/<day>.json`. `SyncConfigStore` already validates event and day
keys against `^[a-z0-9][a-z0-9-]*$`, so both are safe as key prefixes without
further sanitising.

### Pointer — `<event-key>/pointer/<day>.json`

~100 bytes. Polled every 3s. Short TTL.

```json
{
  "hash": "a3f91c7b20e4",
  "updatedAt": "2026-08-18T04:12:33.129Z"
}
```

```
Cache-Control: public, max-age=2
Content-Type:  application/json
```

`updatedAt` advances on **every** successful sync, including ones where `hash`
is unchanged — it is the pipeline-liveness signal the client's staleness
warnings key off (§4.2). `hash` changes only when the rendered data does.

### Payload — `<event-key>/data/<day>-<hash>.json`

The full day snapshot `SyncService.syncDay` already builds, unchanged in
shape:

```js
{
  day, label,
  isLive,               // true | false | "auto" — from events.json; the Live/Hide override
  generatedAt,          // ISO, every sync
  facilities: [{
    name, matchesCsv, standingsCsv,
    syncedAt,           // ISO, restamped every time this facility is fetched
    completedAt         // ISO or null — facilityCompletion.mjs; carried forward once set
  }],
  failedFacilities: [], // "name: error" strings from this attempt
  staleFacilities: []   // names carried forward because this attempt failed them
}
```

The filename is content-addressed, therefore immutable.

```
Cache-Control: public, max-age=31536000, immutable
Content-Type:  application/json
```

---

## 4. Content hashing

### 4.1 Hash content only, never timestamps

`SyncService` stamps two volatile fields into every snapshot:

```js
if (r.status === "fulfilled") freshByName.set(name, { ...r.value, syncedAt: now });
// ...
const snapshot = { day, label, generatedAt: now, facilities: finalFacilities, ... };
```

Hashing the snapshot object would produce a different hash on every sync even
when no cell changed. The skip in §2 step 5 would never fire, every no-op edit
would write a new immutable payload, and the lifecycle rule (§10) would fill R2
with identical content under different names.

Hash **only what the pages render**:

```js
// Stable, order-independent, excludes sync-attempt timestamps.
const hashInput = JSON.stringify({
    label: snapshot.label,
    isLive: snapshot.isLive,
    facilities: snapshot.facilities
        .map(f => [f.name, f.matchesCsv, f.standingsCsv, f.completedAt ?? null])
        .sort((a, b) => a[0].localeCompare(b[0])),
});
const hash = createHash("sha256").update(hashInput).digest("hex").slice(0, 12);
```

**`isLive` must be in the hash.** `setLiveOverride` republishes the snapshot
with nothing but `isLive` changed. Leave it out and the Live/Hide override
produces the same hash, the pointer never changes, no client refetches, and
**Force hidden silently does nothing** on the public site — the one control an
operator reaches for when a wrong score has gone public.

`completedAt` is in it because Control Center's Facility Progress shows it,
and it changes without either CSV changing (it is stamped against the
previously published value). `label` is in it because the pages print it.

Excluded: `generatedAt`, every `syncedAt`, `failedFacilities`,
`staleFacilities`, and `lastEditAt` (added by the prerequisite spec) — all
describe the sync attempt, not the data.

**Use 12 hex chars (48 bits), not 8.** A collision serves the wrong day's board
from an immutable, year-cached URL — the worst failure mode in this design, and
effectively unrecoverable without renaming keys. 8 chars is 32 bits, where
birthday collisions become plausible within a retention window; 12 makes it
negligible and costs four bytes in a filename.

Exclude `failedFacilities` and `staleFacilities` too: they describe the sync
attempt, not the data. A facility that fails and gets carried forward produces
byte-identical rendered output and must not churn the payload.

### 4.2 Required client change: staleness must key off the pointer

Skipping payload writes means `generatedAt` and per-facility `syncedAt` stop
advancing during quiet periods. They are read in these places (line numbers
as of this revision — search by name):

| File | Reads | For |
|---|---|---|
| `tools/control-center.html` `renderOrganizerStatus` area (~6634, ~6689) | `facilities[].syncedAt` | Mission Control's **Facility Sync Status**: amber after `STALE_WARNING_MS` (5 min) |
| `tools/control-center.html` Facility Progress (`facilityInfo`-style helper ~5463, `facilityActualEnd` ~5439) | `facilities[].syncedAt`, `completedAt` | "Data from X ago" line when stale; actual-end fallback for pre-`completedAt` snapshots |
| `tools/control-center.html` `loadLiveData` (~2861) | `generatedAt` | Top status line "last synced HH:MM:SS" |
| Both templates' and every event's `index.html` `loadLiveData` | `generatedAt` | The same "last synced" status line on the public page |

The Mission Control check:

```js
const STALE_WARNING_MS = 5 * 60 * 1000;
const ageMs = Date.now() - new Date(entry.syncedAt || 0).getTime();
const isWarn = failedLastAttempt || ageMs > STALE_WARNING_MS;
```

Left alone, **any gap longer than five minutes with no score change — a lunch
break, between rounds, warm-up — lights up every facility with an amber "aging
data" warning** and freezes the top-line `last synced HH:MM:SS`. Nothing is
actually wrong. This turns Mission Control's main health signal into a false
alarm during exactly the calm periods when someone is most likely to glance at
it.

Separate the two things the current UI conflates — *when data last changed* vs.
*when the pipeline last ran*:

- **Write the pointer on every successful sync**, even when the hash is
  unchanged: same `hash`, fresh `updatedAt`. Only the payload write is skipped.
  Cost is one ~100-byte PUT per sync; the client sees an unchanged hash and
  never refetches the payload, so the dedup benefit is fully preserved.
- Drive "last synced" and the stale warning from **`pointer.updatedAt`**.
  The pointer is per day, not per facility, so Mission Control loses
  per-facility staleness. Either accept that (the red "Last attempt failed"
  state still comes from `failedFacilities`), or add a
  `facilities: { <name>: <ISO> }` map of last-fetch times to the pointer —
  still well under 1 KB.
- Keep per-facility `syncedAt`, but relabel it in the UI to mean "this
  facility's data last *changed*" — useful, and no longer alarming when old.
  Facility Progress's "Data from X ago" line must move to the same source as
  the stale warning, or it reads late during every quiet stretch.
- `STALE_WARNING_MS` moves onto the pointer's age, where 5 minutes is the right
  threshold: a pointer that hasn't advanced in 5 minutes genuinely does mean the
  pipeline has stopped.

This is required, not optional polish. Shipping §4.1 without it makes Mission
Control cry wolf on every quiet stretch of a tournament.

---

## 5. Concurrency — conditional writes are mandatory

**Gated on §1 check 2.** Every facility spreadsheet for a day POSTs to the
**same** `/sync/:day`. Three facilities being scored simultaneously means three
concurrent read-modify-write cycles against one payload. Cloud Run runs
`--concurrency 4` and may have several instances live, so this is ordinary
operation, not an edge case — and a faster Apps Script trigger (§7) makes
overlapping fires *more* likely, not less.

The existing GitHub write is protected by accident: `GitHubPublisher.publish()`
passes the `sha` it read, and GitHub rejects a stale write with `409 Conflict`.
Today the losing sync fails loudly and its data lands only on that
facility's next edit — all four failed syncs at Pickle for Sight were this.
The [Immediate sync](../implemented/immediate-sync-spec.md) spec (this spec's step 0) makes the loser re-read, re-merge and
retry instead. The R2 path must keep
that guarantee, which is what the conditional pointer write below does.

**There are two writers, not one.** `SyncService.setLiveOverride` (behind
`POST /sync/:day/live`) reads the published snapshot, sets `isLive`, and
publishes it again. It must take the same path as a sync: read the pointer
with its ETag, write the payload, write the pointer with `If-Match`, and redo
the read on `412`. Otherwise a Live/Hide click racing a sync can either lose
that sync's scores or lose the override.

**Plain `PutObjectCommand` has no such check:**

> A and B both read state `S`. A writes `payload_A` (= S+A) and points at it.
> B writes `payload_B` (= S+B) and points at it. Final state is `S+B`.
> **A's scores are gone**, silently, and will not return until A's sheet is
> edited again — which may be never, since that score was already entered.

That is strictly worse than the current behaviour. Restore CAS on the pointer:

1. Read the pointer, keeping its **ETag**.
2. Do the merge and write the payload. This is safe unconditionally —
   content-addressing means a concurrent writer with different content writes a
   *different* key, and one with identical content writes identical bytes.
3. Write the pointer with `If-Match: <etag>`.
4. **On `412 Precondition Failed`** — someone published first. Re-read the
   pointer and payload and redo the merge from step 1. Cap at 3 attempts, then
   fail the request like any other upstream error.

For the first pointer of a day, use `If-None-Match: *` so two concurrent
first-syncs can't both believe they are creating it.

Retrying the **whole merge**, not just the pointer write, is the point: the
re-read picks up the other writer's facility data, so the retry publishes the
union rather than clobbering it.

---

## 6. Backend implementation

### 6.1 New file: `src/sync/R2Publisher.mjs`

Mirror `GitHubPublisher.mjs`'s shape (`publish` / `fetchExisting`) so
`SyncService` can treat the two interchangeably. Owns the CAS retry loop (§5).

```js
import { S3Client, PutObjectCommand } from "@aws-sdk/client-s3";

const r2 = new S3Client({
    region: "auto",
    endpoint: `https://${R2_ACCOUNT_ID}.r2.cloudflarestorage.com`,
    credentials: {
        accessKeyId: R2_ACCESS_KEY_ID,
        secretAccessKey: R2_SECRET_ACCESS_KEY,
    },
    // Required: recent aws-sdk-js-v3 sends checksums R2 rejects (§1 check 5).
    requestChecksumCalculation: "WHEN_REQUIRED",
});
```

Per §1 check 5, if the SDK's import cost is significant on a sync-only cold
start, either lazy-import it the way `index.mjs` already does for puppeteer, or
drop it entirely — R2's S3 API is reachable with plain `fetch` + SigV4, and the
write surface here is two `PUT`s.

### 6.2 Writes

```js
// 1. payload — immutable, unconditional
await r2.send(new PutObjectCommand({
    Bucket: R2_BUCKET,
    Key: `${event}/data/${day}-${hash}.json`,
    Body: payloadJson,
    ContentType: "application/json",
    CacheControl: "public, max-age=31536000, immutable",
}));

// 2. pointer — LAST of the R2 writes, and conditional (§5)
await r2.send(new PutObjectCommand({
    Bucket: R2_BUCKET,
    Key: `${event}/pointer/${day}.json`,
    Body: JSON.stringify({ hash, updatedAt: new Date().toISOString() }),
    ContentType: "application/json",
    CacheControl: "public, max-age=2",
    IfMatch: pointerEtag,        // or IfNoneMatch: "*" when creating (§5)
}));

// 3. GitHub archive — after the pointer, off the visibility path.
// Pass null for the sha: publish() looks it up itself, costing one extra GET
// on a write that is already off the critical path.
await this.publisher.publish(config.repoPathFor(event, day), snapshot, msg, null);
```

**The pointer must be written only after the payload write confirms.** A pointer
naming a missing payload gives every client a 404.

**The GitHub commit goes after the pointer**, and stays inside the request. It is
the slowest write in the chain and nothing on the read path waits for it, so
placing it last means client-visible latency is R2-only and a slow or failed
archive commit degrades durability, not freshness. It must not be
fire-and-forget — Cloud Run's default CPU throttling makes post-response
background work unreliable.

Do not reintroduce a GitHub `fetchExisting()` before the R2 writes to
"optimise away" that `null`. That puts GitHub back on the critical path, which
is what §6.3 exists to remove.

### 6.3 Merge-forward reads from R2

`SyncService` currently reads published state back from GitHub before merging:

```js
const path = config.repoPathFor(event, day);
const existing = await this.publisher.fetchExisting(path);
```

Leaving that pointed at GitHub keeps it in the **correctness** path even after
it leaves the latency path, which makes a failed archive commit a data
regression rather than a durability gap:

> Facility A syncs (R2 ✓, GitHub ✗). Facility B syncs moments later, merges
> forward from GitHub — which never received A's update — and publishes
> `B-fresh + A-stale` to R2. A's scores visibly regress on every client.

Read merge-forward state from R2 instead, and treat GitHub as a write-only
archive sink. Read the pointer, then the payload it names:

```
read R2 pointer -> payload
  on 404 -> read GitHub <event>/data/<day>.json   (transition + disaster recovery)
  on both 404 -> treat as first sync of this day (existing behaviour)
```

The GitHub fallback can be removed once every active day has synced at least
once through R2.

`setLiveOverride` reads through the same R2-first path. Its current
`this.publisher.fetchExisting(path)` call against GitHub has to go, for the
same reason.

### 6.4 `SyncService` constructor

Currently positional and already five arguments:

```js
constructor(sheetsApiFetcher, gvizFetcher, publisher, configStore, logger)
```

A sixth positional argument is where this becomes error-prone. **Switch to a
destructured options object** as part of this change. `index.mjs` is the only
call site.

---

## 7. Apps Script

Stays a thin trigger — no sheet reading, no direct writes. `DEBOUNCE_MS` is
already `3000`. That is not the problem any more; the trigger mechanism is.

Today `onEditInstallable` schedules a one-shot **time-based trigger**
(`ScriptApp.newTrigger('runIfSettled').timeBased().after(DEBOUNCE_MS)`), and
that trigger calls Cloud Run. `.after(ms)` is best-effort: the script's own
comment says it can fire up to roughly a minute late. That delay sits in front
of everything R2 speeds up, so without fixing it this project cannot reach its
~5–7s target.

**The fix is its own spec, [Immediate sync](../implemented/immediate-sync-spec.md)** — timing instrumentation, a retry
for syncs that lose a GitHub commit race, and a lock-based trigger that
syncs straight from the edit. It is independent of the delivery store:
build it first, then this spec. Its `edit→request` numbers also settle §1
check 1.

There is no central push: changing `sheets-sync.gs` means pasting it into each
installed Apps Script project by hand — every live facility workbook, plus the
**SAGE Dual Meet Master** and **SAGE Standard Tournament Master** — across
every event registered in `event-data/config/events.json`. Budget for that.
See `_templates/CLAUDE.md` §2 step 8 in `sage-match-control.github.io` for the
install procedure.

---

## 8. Client implementation

Every file in `sage-match-control.github.io` that polls a day snapshot. There
is no shared script file — each page is self-contained — so each copy is
changed by hand, identically where the code is the same:

| File | Loader to change | Notes |
|---|---|---|
| `tools/control-center.html` | `snapshotUrlFor(eventKey, dayKey)`, `fetchDaySnapshot`, `loadLiveData`, poll at the bottom | Switches event **and** day; has `FIXTURE` mode on localhost, which must keep reading `/_fixtures/` and never touch R2. Owns every staleness display in §4.2 |
| `_templates/standard-tournament-template/index.html` | `snapshotUrlFor(dayKey)`, `fetchDaySnapshot`, `loadLiveData` | Public page; switches day |
| `_templates/dual-meet-template/index.html` | same | same |
| `_templates/standard-tournament-template/schedule.html` | `fetchDaySnapshot()` (URL built inline), `loadSchedule` | Wall board; one fixed `DAY_KEY`, no day switching |
| `_templates/dual-meet-template/schedule.html` | same | same |
| `events/<event-key>/index.html` and `schedule.html` for every event not yet archived | same as its template | Template changes do not reach instantiated events. As of this revision: `pickle-for-sight-2026`, `piggleball-2026`, `pnf-x-bup-dual-meet`, `pickledrive-anniversary-2026` (hand-built, not from a template; has its own `FIXTURE` mode). Update the ones still in use; leave finished ones on GitHub |

Not changed: `tools/scoresheet-generator.html` (one fetch when an operator
picks a day, not a poll — GitHub is fine), everything under
`events/archives/`, and any `beta.html`.

Per-event `match-control.html` pages no longer exist outside the BKL Cup 2026
archive; the central Control Center replaced them.

### 8.1 Polling

```
on load:
  fetch pointer  →  fetch <event>/data/<day>-<hash>.json  →  render
  currentHash = pointer.hash
  lastPointerUpdatedAt = pointer.updatedAt        // drives freshness (§4.2)

every 3s, while tab visible:
  fetch pointer
  lastPointerUpdatedAt = pointer.updatedAt        // advances even when hash doesn't
  if pointer.hash !== currentHash:
      fetch <event>/data/<day>-<hash>.json
      render
      currentHash = pointer.hash

on 404 (payload aged out of R2, or pointer names a pruned key):
  fall back to GitHub Pages URL for this fetch, keep polling R2 next tick

on network error / 5xx (R2 or DNS unreachable):
  fall back to GitHub Pages URL for this fetch, keep polling R2 next tick

on any fetch error:
  keep last rendered data
  retry with backoff — never blank the screen
```

Two details that shorthand hides:

- **404 and "R2 unreachable" are different conditions.** A pruned payload
  returns 404; an outage returns a network error or 5xx. Both fall back, but
  code checking only `res.status === 404` silently fails to fall back during an
  actual outage — the case the fallback exists for.
- **The fallback is per-fetch, not a mode switch.** Do not latch to GitHub after
  one failure; resume polling R2 on the next tick, or a single blip parks every
  client on the slow path for the rest of the day.

Carry over unchanged: pausing polling while the tab is hidden and refreshing
immediately on becoming visible (`visibilitychange`), the `AbortController`
timeout (`FETCH_TIMEOUT_MS`), and the stale-response guard (`dayIndexAtStart`).

> **The stale-response guard must wrap both fetches.** `loadLiveData()` currently
> makes a single request and re-checks `currentDayIndex` after it.
> Pointer-then-payload is two sequential awaits, doubling the window in which the
> user can switch days — and the payload fetch is issued from data read *after*
> the first await. Re-check the guard after each, or a day switch landing between
> them renders the wrong day's board.

### 8.2 No cache-buster on the payload URL

The client currently builds:

```js
const snapshotUrlFor = dayKey =>
  `https://${GHPAGES_OWNER}.github.io/${GHPAGES_REPO}/${EVENT_KEY}/data/${dayKey}.json?t=${Date.now()}`;
```

**The payload URL must have no cache-buster.** Carrying `?t=` onto it would
defeat content-addressing entirely — every fetch would miss both the browser and
CDN cache, and immutability is exactly what makes the payload cheap to serve
(§9.4). The pointer needs no buster either: `max-age=2` bounds its staleness
below the 3s poll interval.

> **That `?t=` does not do what its comment claims, and never did.** Measured
> against both `sage-match-control.github.io` and `raw.githubusercontent.com`:
> three requests with *unique* query values all returned `X-Cache: HIT` with a
> monotonically increasing `Age`/`Source-Age`. Fastly strips the query string
> from the cache key on both hosts, so the buster has never defeated anything.
> The current system stays fresh because **GitHub Pages purges its CDN on
> deploy** — the build step this project removes is also what makes the data
> visible. Do not carry the buster forward expecting it to substitute for that
> purge.
>
> It is harmless on the GitHub fallback path, so leave it there rather than
> churning the fallback code; just don't rely on it.

### 8.3 Template constants

The `CONFIGURATION` block gains the R2 base URL and keeps
`GHPAGES_OWNER`/`GHPAGES_REPO` for the fallback path:

```js
// ---- Data source ----------------------------------------------------------
const R2_BASE_URL = 'https://data.example.com';   // R2 custom domain (§9.1)
const pointerUrlFor = dayKey => `${R2_BASE_URL}/${EVENT_KEY}/pointer/${dayKey}.json`;
const payloadUrlFor = (dayKey, hash) => `${R2_BASE_URL}/${EVENT_KEY}/data/${dayKey}-${hash}.json`;

// Fallback only (§10). Keeps the cache-buster; Pages still needs it.
const GHPAGES_OWNER = 'sage-match-control';
const GHPAGES_REPO  = 'event-data';
const snapshotUrlFor = dayKey =>
  `https://${GHPAGES_OWNER}.github.io/${GHPAGES_REPO}/${EVENT_KEY}/data/${dayKey}.json?t=${Date.now()}`;
```

`R2_BASE_URL` is shared platform config, not per-event — treat it like
`GHPAGES_REPO`: a hardcoded constant in both templates, **not** a `{{TOKEN}}`.

Control Center's versions take the event key as an argument
(`pointerUrlFor(eventKey, dayKey)`), like its existing `snapshotUrlFor`, and
keep the `FIXTURE` branch in front of both.

---

## 9. Infrastructure

### 9.1 R2 bucket

| Item | Value |
|---|---|
| Access | Custom domain **only** — `r2.dev` disabled |
| Domain requirement | Must be a zone in the **same Cloudflare account** |
| Cache Rule | Cache **all file types** for the hostname (JSON is not cached by default) — **load-bearing, see §9.4** |
| Smart Tiered Cache | Enabled |
| CORS | Allow `https://sage-match-control.github.io` |
| Lifecycle rule | Delete `*/data/` objects older than **14–30 days** |
| Pointers | No lifecycle rule — retained indefinitely |

> The lifecycle rule must match the event-scoped prefix `*/data/`, not `data/`.
> A rule written against a flat layout matches nothing and silently never
> deletes.

> **The Cache Rule is not a nice-to-have.** Cloudflare does not cache
> `application/json` by default. Without the rule the CDN passes every client
> poll through to the bucket, which still works but changes the cost curve from
> flat to linear in viewers (§9.4). Verify it with `cf-cache-status` (§1 check 3)
> rather than assuming it took effect.

### 9.2 Credentials

R2 API token scoped to **this bucket only**, Object Read & Write. Not
account-wide.

### 9.3 Cloud Run env vars

Added alongside the existing set (`GITHUB_*`, `SYNC_SHARED_SECRET`,
`GOOGLE_SHEETS_API_KEY`, `SHEETS_FETCH_TIMEOUT_MS`, `SYNC_CONFIG_TTL_MS`,
`AUTH_PASSWORD_HASH`, `AUTH_TOKEN_SECRET`, `AUTH_TOKEN_TTL_MS`, `CORS_ORIGIN`):

```
R2_ACCOUNT_ID
R2_ACCESS_KEY_ID
R2_SECRET_ACCESS_KEY
R2_BUCKET
```

Document all four in `.env.example`, matching the commenting style already used
there. Credentials stay env vars — they must never go in
`event-data/config/events.json`, which is a public repo.

### 9.4 Cost model — why the CDN is the whole design

R2 bills Class B operations on **origin reads**, not on requests served from
Cloudflare's cache. With the Cache Rule in place and `max-age=2` on the pointer,
the CDN collapses all client polling into roughly *one origin fetch per 2
seconds per PoP* — **independent of how many people are watching**. Payloads are
`immutable` and cached indefinitely, so each one is fetched from origin about
once per PoP, ever.

That flattening is the entire economic argument. Modelled on an 8-hour event day
at a 3s poll:

| | Billed reads/month, no CDN | Origin reads/month, with CDN |
|---|---|---|
| 100 viewers, 20 event-days | 19,800,000 | ~882,000 |
| 200 viewers, 20 event-days | 39,600,000 | ~882,000 |
| 400 viewers, 20 event-days | 79,200,000 | ~882,000 |

R2's free tier is 10M Class B / 1M Class A / 10 GB per month. With the CDN
working, a heavy month (4 tournaments × 5 days) sits at roughly 9% of the read
allowance and stays free at any realistic audience size. Writes are ~18k Class A
and storage a few hundred MB — both far inside the free tier.

With the Cache Rule missing, the same month at 200 viewers is ~40M billed reads.
It still functions, and nothing in the logs looks wrong; the bill is the only
signal. Hence §1 check 3.

**Do not respond to unexpected cost by lengthening the pointer's `max-age`
beyond the poll interval** — that trades freshness, the entire point of this
project, for a saving the CDN should already be providing. Fix the Cache Rule
instead. If cost genuinely needs tuning after the rule is confirmed working, the
poll interval is the honest dial: 3s → 5s cuts polls ~40% and costs ~2s of
worst-case latency.

---

## 10. Retention model

| Tier | Holds | Retention |
|---|---|---|
| **R2** | Recent content-addressed versions | 14–30 days (lifecycle rule) |
| **GitHub Pages** | Each day's final state | Indefinite |

The lifecycle rule is only safe because the archive tier exists. Ship both
together — the client's 404 fallback covers aged-out objects and R2 outages
through one code path.

This preserves an existing, intentional quirk: the archived `bkl-cup-2026` pages
read from the pre-migration Pages location and are updated by neither path.

---

## 11. Implementation order

Step 0 comes first and ships on its own. Steps 1–11 are inert until 12–15
ship. Steps 12–15 work with either Apps Script trigger. Nothing here requires
a flag day.

0. The [Immediate sync](../implemented/immediate-sync-spec.md) spec, all of it (§7). Re-measure
   before continuing: if Pages is no longer the largest term, stop and
   re-scope
1. Add the domain to Cloudflare as a zone
2. Create the R2 bucket
3. Connect the custom domain (R2 → bucket → Settings → Custom Domains)
4. Add a Cache Rule for the hostname — cache all file types
5. Enable Smart Tiered Cache
6. Configure bucket CORS for `https://sage-match-control.github.io`
7. Disable the `r2.dev` managed subdomain
8. Create the bucket-scoped R2 API token
9. Add the lifecycle rule on `*/data/`
10. **Confirm conditional writes work against the live bucket** (§1 check 2) —
    everything after this depends on it
11. **Confirm the Cache Rule caches JSON** (§1 check 3) — upload a throwaway
    object with `max-age=2`, fetch it twice through the custom domain, require
    `cf-cache-status: HIT` on the second. Do this before writing client code, so
    a misconfigured rule surfaces now rather than on a bill
12. Build `R2Publisher` including the CAS retry loop (§5, §6.1, §6.2); refactor
    `SyncService`'s constructor to an options object (§6.4)
13. Move the merge-forward read to R2 with GitHub fallback (§6.3)
14. Update every file in §8's table: R2 fetch + fallback (§8), and move
    staleness onto `pointer.updatedAt` (§4.2)
15. Add env vars, bump `package.json` (minor) with a `README.md` Changelog
    entry, push to `main` (Cloud Run deploys on push), verify (§12)
16. ~~Set `DEBOUNCE_MS = 3000`~~ — done; superseded by step 0

---

## 12. Acceptance checklist

**Write path**

- [ ] Two syncs with no cell change: the second writes **no payload**, but does
      refresh the pointer's `updatedAt` with an unchanged `hash`.
- [ ] Changing one cell produces exactly one new payload key and one pointer
      update naming it.
- [ ] The pointer never names a key that does not exist — verify by checking
      pointer→payload resolves after every sync in a burst.
- [ ] **Concurrent race:** fire two facility syncs for the same day
      simultaneously; both facilities' data survives in the final payload. This
      is the failure mode that loses real scores — test it before an event.
- [ ] A forced `412` triggers a full re-merge and retry, not just a pointer
      rewrite, and gives up after 3 attempts with a normal upstream error.
- [ ] A GitHub archive failure leaves R2 correct and the client unaffected.
- [ ] `?method=csv` still works end to end.
- [ ] **Live/Hide:** `POST /sync/:day/live` with `false` produces a new hash
      and a pointer update; the public page hides within one poll. Then
      `auto` restores it the same way.
- [ ] A Live/Hide override fired at the same moment as a facility sync
      keeps both the override and the sync's scores.
- [ ] A change that only moves a facility's `completedAt` (clearing then
      re-entering the last score) produces a new hash.

**Client**

- [ ] All three tabs render from R2 against a real day.
- [ ] Payload requests carry **no** `?t=` query param; repeat loads hit CDN
      cache.
- [ ] Deleting the payload from R2 (or blocking the domain) falls back to GitHub
      Pages and keeps rendering; polling returns to R2 on the next tick without
      a reload.
- [ ] Switching days mid-fetch never renders the wrong day's board.
- [ ] Polling stops while the tab is backgrounded and resumes on focus.
- [ ] Standings and live board keep scroll position across a poll re-render.
- [ ] **Mission Control shows no stale warning after 10 minutes of no score
      changes**, while the sync pipeline is still running (§4.2).
- [ ] Mission Control *does* warn within ~5 minutes of the pipeline actually
      stopping.
- [ ] Every file in §8's table is identical in every changed region to the
      others of its kind (`index.html` pages, `schedule.html` pages).
- [ ] Control Center on `localhost` with `?fixture=…` still loads the fixture
      and makes no R2 request.
- [ ] Facility Progress's "Data from X ago" line does not appear during a
      10-minute quiet stretch while the pipeline runs.

**Infrastructure**

- [ ] `/ping` and `/sync/config` alone do **not** constitute verification —
      neither touches R2 and both stay green with a broken bucket. Verify R2
      explicitly: the pointer object exists, its `hash` names an existing
      payload, and the custom domain serves both with the expected
      `Cache-Control`.
- [ ] **Pointer fetched twice through the custom domain returns
      `cf-cache-status: HIT` on the second.** This is the check that keeps cost
      flat in audience size (§9.4); a `MISS`/`DYNAMIC` here means the Cache Rule
      is not applying to JSON and every viewer poll is billed.
- [ ] **Payload returns `cf-cache-status: HIT` on a repeat fetch** and is served
      with `max-age=31536000, immutable`.
- [ ] Payload requests carry no query string — confirm the URL the client
      actually issues, not just the constant that builds it.
- [ ] After a real sync, the pointer reflects the new hash within ~2s through
      the custom domain (the CDN TTL), not only from the bucket directly.
- [ ] `r2.dev` is disabled; the custom domain is the only access path.
- [ ] A payload older than the lifecycle window is actually deleted, and the
      client's 404 fallback covers it.
- [ ] After one event, check R2 metrics: Class B operations should be in the
      hundreds of thousands, not millions. Millions means the CDN is not
      absorbing polls (§9.4).

---

## 13. Out of scope

- Migrating existing published `event-data` snapshots into R2. The client's
  fallback and §6.3's GitHub read cover the transition.
- Retiring GitHub Pages as the archive tier.
- Any change to the scoresheet half of `sage-tools-api`, or to
  `tools/scoresheet-generator.html`'s one-shot snapshot fetch — it stays on
  GitHub Pages.
- Pages under `events/archives/`.
- Changing the `/sync/:day` route shape or its shared-secret auth.
- Extending `/sync/config` to report R2 health. Worth doing later — it is the
  natural home for it — but not required to ship this.

> **Do not deploy mid-event.** This replaces the live data path end to end.
