# Spec — Immediate sync

> **Status: implemented.** Part 1 shipped in `sage-tools-api` 2.3.0 with
> the Control Center `edit→sync` figure; Part 2 is in `scripts/sheets-sync.gs`
> and runs in the Piggleball workbook, measured there on 1 October 2026 (see
> [Sync pipeline](../../technical/sync-pipeline.md#measured-the-lock-based-sync)).
> One divergence: a minimum gap between syncs, `SYNC_MIN_GAP_MS` (§4.3
> notes). Not yet run from §5: the paused-workbook check and the
> multi-workbook collision check, which needs more than one workbook in an
> event; `scripts/verify-sync-merge.mjs` covers the retry it exercises. Two
> events syncing at once was seen for real at Piggleball and PickleDrive on
> 3 October 2026, and §4.4's quota question has an answer from that day
> (see [Sync pipeline](../../technical/sync-pipeline.md#measured-at-two-events-3-october-2026)).
> Written 2026-10-01 against `sage-tools-api` 2.2.0.
>
> **Stands alone.** Build this on its own; it makes today's GitHub-based
> pipeline faster and stops it losing updates. It is also the prerequisite
> for both delivery designs, [Live push delivery](../in-progress/durable-object-push-spec.md)
> and [Fast data delivery](../archived/fast-data-delivery-spec.md), which build on the
> code this spec adds.

Make a facility sheet's edit reach Cloud Run within about two seconds instead
of whenever Google fires a delayed trigger, measure every step from the edit
onwards, and stop a sync that collides with another sync from being silently
lost.

---

## 0. Read this first (implementer orientation)

Assume no knowledge of this project beyond this page.

**Repos.** `D:\Personal\SAGE` is a plain folder holding four git repos. This
spec touches three:

| Repo | What it is | What changes here |
|---|---|---|
| `sage-tools-api/` | Node 22 / Express backend on Google Cloud Run (`us-central1`). ESM `.mjs` throughout, classes with constructor injection, no TypeScript, no build step, no test framework (a few plain-Node `scripts/verify-*.mjs` checks). Deploys automatically on push to `main` | `src/sync/*`, a new `scripts/verify-sync-merge.mjs`, and `scripts/sheets-sync.gs` |
| `sage-match-control.github.io/` | Static site on GitHub Pages. Each page is one self-contained HTML file (inline `<style>` and `<script>`). Commit to deploy | `tools/control-center.html` only (one diagnostic line) |
| `event-data/` | Public repo served by GitHub Pages. `config/events.json` is the event/day/facility registry; every day's live snapshot is `<event-key>/data/<day>.json` | Nothing (it is written to at runtime) |
| `sage-docs/` | Documentation (mkdocs). This spec lives here | Docs updates (§6) |

**`scripts/sheets-sync.gs` is not part of the Cloud Run service.** It is
Google Apps Script, kept in the repo for versioning, and ships by pasting the
whole file into each spreadsheet's script project (**Extensions → Apps
Script**). It is byte-identical in every workbook; each workbook's identity
(day key, facility name, watched tabs, shared secret) lives in that
project's Script Properties. In a live workbook it shares one project with a
generator file (`sheet-generator.gs` or `standard-generator.gs`) and often
`attendance.gs`, so **a top-level name declared twice silently replaces the
other file's** — every new top-level name must be unique across all four
files.

**Rules for every change** (from the root `CLAUDE.md`):

- A code change in `sage-tools-api` bumps `package.json`'s version (minor
  for a feature) **and** adds a matching entry to the Changelog section of
  `sage-tools-api/README.md`. A change to `scripts/*.gs` alone does not.
- Cite a spec from outside `sage-docs` as
  `sage-docs/docs/specs/.../immediate-sync-spec.md` — literally `...` where
  the status folder goes.
- Documentation is written in the present tense.
- **Never deploy or paste any of this on an event day**, or within a few
  days before one. Nothing here ships before 3 October 2026.

**How a sync works today:**

```
Facility Google Sheet (one per venue per tournament day)
  | installable onEdit trigger -> onEditInstallable(e)       (sheets-sync.gs)
  |   ignores edits outside the watched tabs (by sheet GID)
  |   records PROP_LAST_EDIT, deletes any pending runIfSettled trigger,
  |   creates a one-shot time-based trigger .after(DEBOUNCE_MS = 3000)
  v
runIfSettled()  — fires "about" 3s later; Google treats that as a minimum
  |   skips if a newer edit is pending; else triggerSync_()
  v
POST /sync/:day?facility=<name>   Cloud Run, header X-Sync-Secret
  |   src/sync/routes.mjs handleSync -> SyncService.syncDay
  |   1. fetch this facility's CSV + STANDINGSCSV tabs (Sheets API)
  |   2. GitHubPublisher.fetchExisting(<event>/data/<day>.json) -> { json, sha }
  |   3. merge: fresh data for this facility, the rest carried forward
  |   4. GitHubPublisher.publish(path, snapshot, message, sha)  (a commit)
  v
event-data commit -> GitHub Pages build -> pages poll every 10s
```

Control Center (the operator console) also calls `POST /sync/:day` without
`?facility=` (a full resync of every facility, with an operator bearer
token) and `POST /sync/:day/live` (the Live/Hide override:
`SyncConfigStore.setIsLive` commits `config/events.json`, then
`SyncService.setLiveOverride` re-commits the day's snapshot with only
`isLive` changed).

**The snapshot** (`SyncService.syncDay` builds it; this spec adds
`lastEditAt`):

```js
{
  day, label,
  isLive,               // true | false | "auto"
  generatedAt,          // ISO, set by each syncDay
  facilities: [{
    name, matchesCsv, standingsCsv,
    syncedAt,           // ISO, restamped whenever this facility is fetched
    completedAt,        // ISO or null — src/sync/facilityCompletion.mjs
    lastEditAt          // NEW: ISO or null — when the edit behind the latest sync was made
  }],
  failedFacilities: [], // "name: error" strings
  staleFacilities: []   // names carried forward because this attempt failed them
}
```

**Files you will read and change:**

| File | Why |
|---|---|
| `sage-tools-api/scripts/sheets-sync.gs` | `onEditInstallable`, `runIfSettled`, `triggerSync_`, `testSyncNow`, `readSyncConfig_`, `isSyncPaused_`, `DEBOUNCE_MS`, `SETTLE_HANDLER`, `PROP_LAST_EDIT` |
| `sage-tools-api/src/sync/routes.mjs` | `handleSync`, `handleSetLive`, and their `@openapi` JSDoc blocks |
| `sage-tools-api/src/sync/SyncService.mjs` | `syncDay`, `setLiveOverride` |
| `sage-tools-api/src/sync/SyncConfigStore.mjs` | `setIsLive` |
| `sage-tools-api/src/sync/GitHubPublisher.mjs` | `publish`, `fetchExisting` |
| `sage-tools-api/scripts/attendance.gs` | Read only: it holds `LockService.getScriptLock()`, which is why §4 uses the document lock |
| `sage-tools-api/scripts/verify-*.mjs`, `mock-apps-script.mjs` | Existing checks to keep passing; a pattern for the new one |
| `sage-match-control.github.io/tools/control-center.html` | `renderOrganizerStatus` (Facility Sync Status rows) |

## 1. Why, in numbers

Measured at CLSO Pickle for Sight, 27 September 2026 (2 facilities, one
event that day):

| Leg | p50 | p90 | max | Source |
|---|---|---|---|---|
| Apps Script: edit → `runIfSettled` fires | 74s | 119s | 119s | Piggleball Executions page, 1 Oct 2026, five bursts (22–119s). See `technical/sync-pipeline.md` § Baseline |
| Cloud Run `POST /sync/:day` (264 Apps Script calls) | 1.6s | 1.85s | 19s | Cloud Run request logs |
| GitHub commit → Pages deployed (287 builds, 46 cancelled by newer pushes) | 25s | 52s | 82s | `event-data` Actions runs |
| Page poll | 5s | 10s | 10s | `POLL_INTERVAL_MS = 10000` |

This spec shortens the first leg to ~0.5–2s plus a 1.5s settle. The Pages
leg is the delivery specs' job.

**Concurrent syncs already fail.** Every facility of a day writes into the
same `<event>/data/<day>.json`, and every commit of every event moves the
same `main` branch of `event-data`. A commit made with a stale sha is
rejected with `409 Conflict`, and today the losing sync returns `500`. All
four failed syncs at Pickle for Sight were this (Cloud Run logs, UTC):

| Time | Collision |
|---|---|
| 04:15:22, 11:07:56 | The same workbook syncing twice within a second — the time-based trigger firing twice for one burst |
| 06:44:41, 09:58:22 | A Control Center full resync against a facility's sync |

None lost data that day, but a collision **between two facilities**, or
**between two events on the same day** (3 October 2026 has two), loses the
loser's update until someone next edits that facility's sheet — minutes, if
it was a match's final score. Estimated at a handful a day with three
facilities, and more as syncs get faster. Mission Control only turns a
facility amber after 5 minutes without a sync, so nobody notices quickly.

## 2. The two parts

| Part | What | Ships as |
|---|---|---|
| 1 | Cloud Run: timing instrumentation; re-read-and-retry on a `409` in `syncDay`, `setLiveOverride` and `setIsLive`; `verify-sync-merge.mjs`. Apps Script: send the edit time. Control Center: show `edit→sync` | Minor `sage-tools-api` release + site commit + (optionally) a paste of `sheets-sync.gs` |
| 2 | Apps Script: sync from the edit itself under a document lock, and retry a failed sync once | A paste of `sheets-sync.gs` into every workbook and both masters |

**Order:** Part 1's Cloud Run release goes first — the retry must be live
before Part 2 makes syncs faster and collisions more likely. Part 1's small
`sheets-sync.gs` change (§3.1) and Part 2 can go out in the same paste.
To get a "before" measurement of the old trigger, paste §3.1 on its own
first (with the `runIfSettled` change it describes) and run one event or
rehearsal before Part 2.

---

## 3. Part 1 — Timing instrumentation and conflict retry

Goals: every sync records how long it took from the edit to Cloud Run, and
from the edit to publication; and a sync that loses a race with another
sync for the same day retries instead of failing (§3.3). The retry must be
live **before** Part 2 makes syncs faster.

### 3.1 `scripts/sheets-sync.gs`

`triggerSync_()` takes an optional argument and sends the edit time as a
header:

```js
function triggerSync_(opts) {
  const editAt = opts && opts.editAt ? Number(opts.editAt) : 0;
  // ... existing config/secret checks unchanged ...
  const headers = { 'X-Sync-Secret': secret };
  if (editAt > 0) headers['X-Edit-At'] = String(editAt);
  const response = UrlFetchApp.fetch(url, {
    method: 'post',
    headers: headers,
    fetchTimeoutSeconds: 30,
    muteHttpExceptions: true,
  });
  // ... rest unchanged ...
}
```

The edit path passes `{ editAt: <value of PROP_LAST_EDIT> }` (Part 2 shows
where). `testSyncNow()` (the **SAGE → Sync now** menu item) keeps calling
`triggerSync_()` with no argument, so manual syncs carry no edit time.

If Part 2 is not shipped in the same paste, change `runIfSettled` to read
`PROP_LAST_EDIT` and call `triggerSync_({ editAt: lastEdit })`.

### 3.2 `sage-tools-api`

**`src/sync/routes.mjs` → `handleSync`:** read the header and pass it on.

```js
const rawEditAt = Number(req.get("X-Edit-At"));
// Only a plausible recent edit time counts: within the last hour, not in
// the future by more than a few seconds of clock skew.
const editAt = Number.isFinite(rawEditAt)
    && rawEditAt > Date.now() - 3600_000
    && rawEditAt < Date.now() + 5_000 ? rawEditAt : null;
const result = await syncService.syncDay(req.params.day, { facilityName, method, editAt });
```

Document the header in the route's `@openapi` JSDoc block (the spec at
`GET /openapi.json` is generated from those blocks).

**`src/sync/SyncService.mjs` → `syncDay(day, { facilityName, method, editAt = null })`:**

- Record `const t0 = Date.now()` first thing, and time the fetch
  (`fetchMs`), the merge read + publish (`publishMs`), and the total.
- When a facility was fetched fresh **and** `editAt` is set **and** the sync
  is facility-scoped, set that facility's `lastEditAt` to
  `new Date(editAt).toISOString()`. Otherwise carry the prior value forward:
  `lastEditAt: existingByName.get(f.name)?.lastEditAt ?? null`. (A full
  resync must not wipe it.)
- Log one line per sync:

  ```
  timing edit→request=<t0-editAt|n/a>ms fetch=<fetchMs>ms publish=<publishMs>ms edit→published=<now-editAt|n/a>ms
  ```

- Add a `timing` object to the return value:
  `{ editToRequestMs, fetchMs, publishMs, editToPublishedMs }` (the two
  `edit…` fields `null` when there is no `editAt`).

Bump the minor version in `package.json` (2.2.0 → 2.3.0 if nothing else has
shipped in between) with a Changelog entry in `README.md`.

**Reading the numbers afterwards** (PowerShell, gcloud installed and signed
in; on PowerShell 5.1 inner double quotes must be escaped as `\"`):

```
gcloud logging read 'resource.type=cloud_run_revision AND resource.labels.service_name=sage-tools-api AND textPayload:\"timing edit\"' --freshness=1d --format='value(timestamp,textPayload)' --project=sage-tools-api
```

(`Logger` writes plain text, which Cloud Logging stores as `textPayload`,
e.g. `2026-09-27T11:07:57.569Z :: [scoresheet:sync:github] committed …`.)

### 3.3 Retry on a GitHub conflict

**`src/sync/GitHubPublisher.mjs` → `publish`:** when the PUT fails, attach
the HTTP status to the thrown error so callers can tell a conflict apart
from anything else. The message stays exactly as it is.

```js
if (!res.ok) {
    const detail = await res.text().catch(() => "");
    const err = new Error(`GitHub commit failed: HTTP ${res.status} ${detail}`.trim());
    err.status = res.status;
    throw err;
}
```

GitHub reports a stale sha as `409` (seen at Pickle for Sight with two
wordings: `is at <sha> but expected <sha>` and `<path> does not match
<sha>`). Match on the status, never the message.

**A `409` does not mean the file changed.** Every commit moves the one
`main` branch of `event-data`, and GitHub moves it one commit at a time.
The first wording above is a branch-level conflict: the `<sha>` it is "at"
was another request's commit a moment earlier. So two syncs that touch
**different files** — two events on the same day, or a sync and a Live/Hide
override writing `config/events.json` — can still collide. The retry below
handles that case too: the re-read returns the same file and sha, the merge
produces the same snapshot, and the second commit goes through.

**`src/sync/SyncService.mjs` → `syncDay`:** wrap the read-merge-publish
part — from `this.publisher.fetchExisting(path)` through
`this.publisher.publish(...)` — in a loop of at most **3 attempts**. On an
error with `status === 409`, go round again: re-read the file (new
`existing.json` and `existing.sha`) and redo the merge with the facility
data **already fetched** this request (`freshByName`, `failed`). Do not
fetch the sheets again; they were read moments ago. Any other error, or a
third `409`, throws as today.

Move the snapshot-building code (the `allFacilities.forEach` that fills
`finalFacilities` and `stale`, and the `snapshot` object) into a private
method, e.g. `#buildSnapshot({ day, label, isLive, now, allFacilities,
targetFacilities, freshByName, failed, existing })`, so the loop can call
it again against the re-read file. Its logic doesn't change —
`resolveCompletedAt` must see the re-read file's `completedAt`, which it
does because `existing` is the re-read one.

Log each retry at `info`: `commit conflict (attempt n/3) — re-reading and
re-merging`. Add `attempts` to the return value.

**`setLiveOverride`:** same loop around its `fetchExisting` → spread →
`publish`, re-reading on `409`.

**`src/sync/SyncConfigStore.mjs` → `setIsLive`:** the override's first
commit, to `config/events.json`, has the same exposure: it reads the file,
sets the day's `isLive`, and publishes with the sha it read, so any sync of
any event landing in between makes it fail and the operator sees an error.
Wrap its read → modify → `publish` in the same 3-attempt loop on
`status === 409`, re-reading `config/events.json` and re-applying the one
change each time. Keep invalidating the cache only after a successful
commit.

**`scripts/verify-sync-merge.mjs` (new):** a plain Node script in the style
of the other `verify-*` scripts (no framework, no network, exits non-zero
on failure). It constructs `SyncService` with in-memory fakes: a fetcher
returning fixed CSVs per facility, and a fake `GitHubPublisher` holding one
file and its sha that can be told to change the file (as if another sync
committed) and return `409` on the next `publish`. Scenarios:

1. Facility-scoped sync of A with no conflict → A fresh, B carried forward.
2. Conflict: before A's commit, the fake is changed to contain B's new
   data and made to return `409` once → A's retried commit contains
   **A's and B's** new data, `attempts: 2`.
3. Three `409`s in a row → the sync throws with `status` 409.
4. A non-409 error (`500`) → throws on the first attempt, no retry.
5. `setLiveOverride(false)` with one `409` → retried; the file ends with
   `isLive: false` and B's data intact.
6. `lastEditAt` from an `editAt` sync survives a later full resync.
7. Branch-level conflict: the fake returns `409` once **without changing
   the file** (another event's commit moved the branch) → the retry
   re-reads the same sha, commits the same snapshot, `attempts: 2`.
8. `SyncConfigStore.setIsLive` with one `409`, where the fake's
   `config/events.json` has meanwhile gained another day's `isLive` change
   → the retried commit keeps **both** changes.

Whichever delivery spec is built next extends this script (§8). Add it to
the root `CLAUDE.md`'s list of verify scripts with its run command:

```bash
node scripts/verify-sync-merge.mjs
```

### 3.4 `tools/control-center.html`

In Mission Control's **Facility Sync Status** rows (the code that computes
`STALE_WARNING_MS` / `isWarn` inside or near `renderOrganizerStatus`), when a
facility has both `syncedAt` and `lastEditAt`, append
` · edit→sync <n>s` where
`n = Math.round((Date.parse(syncedAt) - Date.parse(lastEditAt)) / 1000)`.
Show it only when `0 <= n < 600`. It is an operator diagnostic; no public
page shows it.

### 3.5 Done when

- A test edit in a configured workbook produces a Cloud Run log line with a
  numeric `edit→request`.
- The published snapshot's facility has `lastEditAt`.
- Mission Control shows the `edit→sync` figure.
- **SAGE → Sync now** still works and logs `edit→request=n/a`.
- `node scripts/verify-sync-merge.mjs` passes.
- On a scratch day with two facilities, firing **SAGE → Sync now** in both
  workbooks at the same moment (two people, or two browser windows) ends
  with both facilities' edits published and no `500` — the Cloud Run log
  shows a `commit conflict` retry if they collided.

---

## 4. Part 2 — Lock-based Apps Script sync

### 4.1 The problem

`onEditInstallable` today (read it in `scripts/sheets-sync.gs`) records
`PROP_LAST_EDIT`, deletes any pending `runIfSettled` trigger, and creates a
new one with `.timeBased().after(DEBOUNCE_MS)`. Google fires `.after()`
triggers on a best-effort schedule, so the sync can start well after the 3s
debounce. Nothing downstream can make up for that.

### 4.2 The design

Sync from the edit's own execution, and let a lock collapse bursts:

- Every edit records `PROP_LAST_EDIT`, then tries to take the workbook's
  **document lock** without waiting.
- The execution that gets the lock waits `SYNC_SETTLE_MS` (1.5s, so the
  second score of a match usually lands in the same sync), syncs, and then
  checks whether any edit was recorded after that sync started. If so it
  syncs again, and so on.
- Executions that don't get the lock return immediately: the lock holder
  will see their `PROP_LAST_EDIT` and cover them.
- After releasing the lock, the holder checks once more. An edit that
  arrived between its last check and the release could not take the lock
  and was not covered; the holder takes the lock again for it.
- A sync that fails (a `5xx` from Cloud Run, a network error or a timeout)
  is tried once more after 2s. The loop only re-syncs for *newer* edits, so
  without this a failed sync would stay unpublished until the next edit.

**Use `LockService.getDocumentLock()`, not `getScriptLock()`.**
`scripts/attendance.gs` runs in the same Apps Script project in live
workbooks and holds `getScriptLock()` for up to 10s while marking a player.
Sharing that lock would make edits made during an attendance mark lose their
sync.

### 4.3 Code

Replace `onEditInstallable` and `runIfSettled`, and add the helper and
constants. Keep `DEBOUNCE_MS` and `SETTLE_HANDLER` (the fallback below uses
them). All new top-level names must be unique across `sheets-sync.gs`,
`sheet-generator.gs`, `standard-generator.gs` and `attendance.gs`, because
they share one project in real workbooks.

```js
const SYNC_SETTLE_MS = 1500;          // let a two-cell score entry land in one sync
const SYNC_RETRY_MS = 2000;           // wait before retrying a failed sync once
const SYNC_BUDGET_MS = 4 * 60 * 1000; // stay well inside Apps Script's 6-minute execution cap

function onEditInstallable(e) {
  const config = readSyncConfig_();
  if (!config) return;
  if (isSyncPaused_()) return;

  const editedGid = e && e.range ? String(e.range.getSheet().getSheetId()) : null;
  if (!editedGid || !config.watchedGids.includes(editedGid)) return;

  PropertiesService.getScriptProperties().setProperty(PROP_LAST_EDIT, String(Date.now()));
  syncUntilSettled_();
}

/**
 * Fallback path only: fired by a time-based trigger that
 * syncUntilSettled_ schedules when it runs out of budget, and by any
 * trigger the previous version of this file left scheduled.
 */
function runIfSettled() {
  if (isSyncPaused_()) {
    console.log('Skipping sync — live sync is paused. Run SAGE -> Resume live sync to re-enable it.');
    return;
  }
  syncUntilSettled_();
}

/**
 * Syncs now, then again for as long as edits keep arriving. Only one
 * execution per workbook runs this at a time (document lock); every other
 * edit just records PROP_LAST_EDIT and returns, and whoever holds the lock
 * picks it up.
 */
function syncUntilSettled_() {
  const props = PropertiesService.getScriptProperties();
  const lock = LockService.getDocumentLock();
  const startedAt = Date.now();

  while (true) {
    if (!lock.tryLock(0)) return; // someone else is syncing and will cover this edit
    let lastSyncStart = 0;
    try {
      while (true) {
        Utilities.sleep(SYNC_SETTLE_MS);
        if (isSyncPaused_()) return;
        lastSyncStart = Date.now();
        const editAt = Number(props.getProperty(PROP_LAST_EDIT) || 0);
        syncWithRetry_(editAt);
        const lastEdit = Number(props.getProperty(PROP_LAST_EDIT) || 0);
        if (lastEdit < lastSyncStart) break;          // nothing new during that sync
        if (Date.now() - startedAt > SYNC_BUDGET_MS) { // pathological edit storm
          scheduleFallbackSync_();
          return;
        }
      }
    } finally {
      lock.releaseLock();
    }
    // An edit recorded after the last check but before releaseLock() found
    // the lock taken and returned. Go round again for it.
    const lastEdit = Number(props.getProperty(PROP_LAST_EDIT) || 0);
    if (lastEdit < lastSyncStart) return;
  }
}

/**
 * One sync, tried a second time after SYNC_RETRY_MS if it fails. Without
 * this, a failed sync stays unpublished until the next edit in this
 * workbook — the loop above only goes round again for NEWER edits.
 */
function syncWithRetry_(editAt) {
  for (let attempt = 1; attempt <= 2; attempt++) {
    try {
      const result = triggerSync_({ editAt: editAt });
      if (!result || result.ok) return;                         // done, or unconfigured
      if (result.status >= 400 && result.status < 500) return;  // bad secret/day/facility: a retry can't help
    } catch (err) {
      console.error(`Sync threw: ${err}`);                      // network error or timeout
    }
    if (attempt === 1) Utilities.sleep(SYNC_RETRY_MS);
  }
}

function scheduleFallbackSync_() {
  ScriptApp.getProjectTriggers().forEach(t => {
    if (t.getHandlerFunction() === SETTLE_HANDLER) ScriptApp.deleteTrigger(t);
  });
  ScriptApp.newTrigger(SETTLE_HANDLER).timeBased().after(DEBOUNCE_MS).create();
}
```

Notes:

- `return` inside `try` still runs `finally`, so the lock is always
  released.
- `syncWithRetry_` is the Apps Script half of the conflict fix; §3.3 is the
  server half. Cloud Run's own 3-attempt retry handles almost every
  collision, so this second try mostly covers a network error, a timeout,
  or a cold start that failed. `UrlFetchApp.fetch` throws on those even with
  `muteHttpExceptions: true` (that option only covers HTTP error statuses),
  which is why it is caught. Cloud Run answers every failure except bad
  auth and validation with a `5xx`, so `4xx` is not retried.
- `DEBOUNCE_MS`'s comment and the file header (which describe the old
  "once edits go quiet for DEBOUNCE_MS" behaviour) must be rewritten to
  describe this design.
- **Built differently: `SYNC_MIN_GAP_MS` (5000).** Not in the code above.
  Measured on Piggleball, steady typing made a sync every ~3 s (seven
  commits for a 26 s burst of ~16 edits), each one a GitHub commit and a Pages
  build, which risks GitHub's commit rate limits. `syncUntilSettled_` waits
  `syncWaitMs_` before every sync: `SYNC_SETTLE_MS`, stretched so the sync
  starts at least `SYNC_MIN_GAP_MS` after the previous one began (kept in
  the `lastSyncStartTime` script property). A lone edit after a quiet spell
  still waits only the settle. The last edit of a burst can publish up to
  ~5 s later than without the gap.
- Nothing needs migrating: a `runIfSettled` trigger left over from the old
  version fires once, runs the new `runIfSettled`, and deletes itself.

### 4.4 Quota check (do this before shipping)

Installable triggers count against the **installing account's** daily
trigger runtime: 90 minutes a day on a consumer Google account, 6 hours on
Google Workspace, shared across every workbook that account installed
triggers in. Per sync burst this design spends ~1.5s of settle plus the
Cloud Run round trip (~2s now, ~3–4s once a delivery spec adds its own
publish); each non-holder edit spends well under a second. Pickle for
Sight's 266 syncs would be roughly 20–30 minutes; a three-facility day
(~400 syncs, ~1,200 watched-tab edits) about 30 minutes. The allowance is
per account, so two events on the same day whose triggers the same account
installed share it. That fits a consumer account,
but find out which account owns the triggers (SAGE → Set up live sync
installs them as whoever runs it) and whether it is consumer or Workspace.
If a larger event could approach 90 minutes, raise `SYNC_SETTLE_MS` less
than you might think — the settle is paid once per burst — and instead make
sure one Google account doesn't hold every workbook's triggers.

**Measured, 3 October 2026.** Piggleball's 238 syncs and PickleDrive's 695
(two events, one venue each, the same day) cost about two hours of trigger
runtime between them: about 105 minutes for PickleDrive, whose Sheets reads
were often 10–58 s slow and whose lock holder waits on `UrlFetchApp` for all
of it, and 16 for Piggleball, before the non-holder edits. That is over a
consumer account's 90 minutes, yet no sync stopped, so the two workbooks'
triggers were owned by a Workspace account or by more than one account.
Which, is still to be recorded. Either way, a slow workbook read costs
trigger runtime one-for-one: making the read fast (or bounding it, see
`SHEETS_FETCH_TIMEOUT_MS`) is what keeps a busy day inside the allowance,
not the settle. Figures in
[Sync pipeline](../../technical/sync-pipeline.md#measured-at-two-events-3-october-2026).

### 4.5 Verify and ship

- `node --check` cannot parse `.gs`; copy to a temp `.js` file and run
  `node --check` on that.
- Run `node scripts/verify-attendance.mjs` and
  `node scripts/verify-standard-generator.mjs` and
  `node scripts/verify-sheet-generator.mjs`. They load `sheets-sync.gs`
  into the same context as the other files and fail on a top-level name
  clash. (`scripts/mock-apps-script.mjs` has no `LockService`; that is fine
  as long as nothing in those scripts calls `onEditInstallable`.)
- Test in a **copy** of a live workbook before any real one. A copy starts
  unconfigured (the spreadsheet-ID guard in `readSyncConfig_`), so run
  **SAGE → Set up live sync** in it against a scratch day registered in
  `event-data/config/events.json` — add one if none exists (shape in
  `event-data/config/README.md`), and remove it afterwards. Then check: single edit, two edits 1s apart (one sync), ten
  rapid edits (two or three syncs, the last containing every edit), an edit
  while **SAGE → Pause live sync** is on (no sync).
- Ship by pasting the file into every live facility workbook **and** into
  the SAGE Dual Meet Master and SAGE Standard Tournament Master.


---

## 5. Acceptance checklist

**Part 1 — Cloud Run**

- [ ] Everything in §3.5.
- [ ] **Three facilities at once:** on a scratch day with three workbooks,
      edit all three within the same second, five times over. Every edit
      ends up in the published snapshot, and no sync returns `500`.
- [ ] **Two events at once:** with two scratch events registered, sync one
      of each within the same second, five times over; both publish.
      Click Live/Hide for one while the other syncs; the override applies
      without an error.

**Part 2 — Apps Script**

- [ ] One edit → one sync, with `edit→request` ≤ ~3s in the Cloud Run log.
- [ ] A failed sync is retried: point a copy's sync at a scratch day while
      Cloud Run is made to fail once (e.g. a temporary wrong
      `GOOGLE_SHEETS_API_KEY` on a test revision, restored after the first
      attempt), and the edit is published by the second try. If staging
      that is impractical, at least check the Executions log shows
      `Sync failed (5xx)` followed by `Sync OK` for a real failure.
- [ ] Two cells edited within 1s → one sync containing both.
- [ ] Ten rapid edits → every edit is in the final published snapshot.
- [ ] Paused workbook → no sync.
- [ ] An attendance mark (`attendance.gs`) during a burst of edits doesn't
      block or lose a sync.
- [ ] `verify-attendance`, `verify-standard-generator`,
      `verify-sheet-generator` pass.

**From real use, 3 October 2026** (not the scripted checks above, but the
same ground): Piggleball and PickleDrive synced side by side from 12:52 to
16:07, five of their publishes within a second of each other's, and both
published every sync (936 commits); the day's 8 archive conflicts (6
PickleDrive, 2 Piggleball) and 2 live version conflicts all resolved on the
first retry. Piggleball's
`edit→request` was p50 2.2 s, p90 3.9 s against this list's ~3 s.


## 6. Documentation to update

In the present tense, once each part ships:

- Root `CLAUDE.md` (in `D:\Personal\SAGE`): the data-flow diagram's first
  step ("installable onEdit trigger, debounced") and the `sheets-sync.gs`
  description (lock-based, not debounced); add
  `scripts/verify-sync-merge.mjs` to the list of verify scripts with its run
  command, and note it covers `SyncService`'s merge and conflict retry.
- `sage-tools-api/README.md`: the Changelog entry for Part 1's version.
- `sage-docs/docs/technical/sync-pipeline.md`: how a sync is triggered, the
  retry on `409`, the `X-Edit-At` header and `lastEditAt`, and the
  measured timings once available.
- `sage-docs/docs/features/control-center.md`: the `edit→sync` figure in
  Facility Sync Status.
- `sage-match-control.github.io/_templates/CLAUDE.md` §2 step 8 and the
  `sheets-sync.gs` header comment, wherever they describe the debounce.
- Move this spec to `implemented/` per `sage-docs/docs/specs/README.md`.

## 7. Rollout and rollback

1. Part 1's Cloud Run release: push to `main` (Cloud Run deploys on push);
   check `GET /ping`'s `X-App-Version` header shows the new version.
2. Control Center change: commit to `sage-match-control.github.io`.
3. Rehearse Part 2 in a copy of a workbook (§4.5).
4. Paste `sheets-sync.gs` into every live facility workbook of every
   upcoming event, and into the **SAGE Dual Meet Master** and **SAGE Standard
   Tournament Master** so new workbooks copied from them get it.
5. Measure at the next event with §3.2's `gcloud` command.

**Rollback:** the Cloud Run part is backwards compatible — old
`sheets-sync.gs` copies send no `X-Edit-At` and still work. To undo Part 2,
paste the previous `sheets-sync.gs` from git history back into the
workbooks. To undo Part 1, revert the commit and push.

## 8. What the delivery specs build on

[Live push delivery](../in-progress/durable-object-push-spec.md) and
[Fast data delivery](../archived/fast-data-delivery-spec.md) both assume this spec is
built. They rely on, by name:

- `facilities[].lastEditAt` and the `timing` object in the sync response;
- `GitHubPublisher.publish` errors carrying `status`;
- the 3-attempt `409` loop in `syncDay`, `setLiveOverride` and
  `setIsLive`, and `SyncService#buildSnapshot`;
- `scripts/verify-sync-merge.mjs`, which they extend;
- `syncUntilSettled_` / `syncWithRetry_` in `sheets-sync.gs`.

Keep those names if you change the design, or update both delivery specs.

## 9. Out of scope

- Anything about how pages receive data (GitHub Pages builds, polling,
  push) — the delivery specs.
- Retrying from Control Center's **Resync this day now** button: it is a
  manual action and already reports failure to the operator.
- Moving triggers between Google accounts to spread Apps Script quota
  (§4.4 says when that would be needed).
