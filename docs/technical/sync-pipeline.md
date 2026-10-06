# Sync pipeline

How a score typed into a Google Sheet ends up on the public site, and how
`sage-tools-api` knows which spreadsheets belong to which event without a
redeploy. [Live push delivery](#live-push-delivery) covers the last hop, from
Cloud Run to open pages.

## The sheet → GitHub path

```
Facility Google Sheet
   |  installable onEdit trigger, document-locked, syncs from the edit itself
   |  (apps-script/sheets-sync.gs)
   v
POST /v3/days/{day}/facilities/{facility}/syncs   (X-Sync-Secret header, or an operator's
   |     bearer token; body { editedAt } carries the edit time. A workbook made before
   |     3.0.0 calls the legacy POST /sync/:day?facility= with an X-Edit-At header)
   |  SheetsCsvFetcher (default) or GvizCsvFetcher ({ method: "csv" } fallback)
   |  read the day's snapshot + version from the live Worker (GitHub only if it holds none),
   |  merge — never drop a facility on failure — and publish with expectedVersion;
   |  re-read and re-merge on a 409 version conflict (3 attempts)
   v
Cloudflare Worker "sage-live": one Durable Object per <event>/<day>
   |  stores {version, snapshot}; pushes every accepted publish to its WebSockets
   |     +--> every open page, over wss://.../live/<event>/<day>
   v
GitHub Contents API commit -> event-data/<event-key>/data/<day>.json   (the archive)
   |  GitHub Pages redeploys on push (no cache-purge step)
   v
A page with no open socket fetches https://sage-match-control.github.io/event-data/<event-key>/data/<day>.json
```

With `LIVE_PUSH_URL` unset the Worker steps are skipped: the merge reads the
file from GitHub and the commit is the publish. The same happens for a single
sync whose Worker calls fail.

Key pieces, in `sage-tools-api/src/`:

- **`clients/SheetsCsvFetcher`** — the default path, reads via the Sheets API.
- **`clients/GvizCsvFetcher`** — a fallback path (`{ "method": "csv" }` in a
  `/v3` sync's body, `?method=csv` on the legacy route) using Google's
  `gviz` CSV export, for when the Sheets API path has trouble. Both fetchers are
  `FacilityFetcher`s, and `SyncService` picks one from a `{ sheets, csv }` map by
  the request's method.
- **`clients/LivePublisher`** — the client for the live Worker: `read(event, day)`
  returns the day's `{ version, snapshot }`, and `publish(event, day,
  snapshot, expectedVersion)` answers `{ ok: true, version }` or, when
  another publish got there first, `{ ok: false, conflict: true }`. It is
  `enabled` only when both `LIVE_PUSH_URL` and `LIVE_PUSH_SECRET` are set.
- **`clients/GitHubPublisher`** — wraps the Contents API. Used both to publish a new
  snapshot and (shared with the config store below) to read
  `config/events.json`. A failed commit throws an error carrying the HTTP
  `status`.
- **`sync/SyncService`** — orchestrates one sync and nothing else: resolves the
  day's config, fetches each facility (in parallel), merges the results with
  whatever is already published (`sync/domain/mergeSnapshot`: a facility that
  fails to fetch keeps its last-known-good data rather than vanishing from the
  snapshot), and hands the merged snapshot to the `SnapshotPublisher`. It knows
  nothing about GitHub, the Worker, the archive or the fallback.
- **`sync/publishing/`** — how a snapshot is delivered; see
  [Delivering a snapshot](#delivering-a-snapshot).
- **`sync/DayVisibilityService`** — the Live/Hide override (below).
- **`sync/SyncSettingsService`** — the live-push switch and the
  `GET /v3/diagnostics/sync` payload (legacy `GET /sync/config`).

Both fetchers are pure functions of `(facility, sheets)` — neither imports
config directly; `SyncService` resolves the day's sheet names once and
passes them down.

### Delivering a snapshot

Both places a snapshot lives, GitHub and the live Worker, are optimistic stores:
you read a value and a token, write back with that token, and are told when
someone else wrote first. The token is the file's `sha` for GitHub and the
`version` for the Worker. One interface, `SnapshotStore` (`read`, `write`),
covers both, with one failure contract: a conflict is `{ ok: false, conflict:
true }`, and any other failure of a read or write is thrown as a
`StoreUnavailableError` that keeps the original message and `status` and names
the store that failed.

| Class (`src/sync/publishing/`) | One job |
| --- | --- |
| `GitHubSnapshotStore`, `LiveSnapshotStore` | adapt one client to `SnapshotStore` |
| `mergeIntoStore` | the one read, build, stamp `publishedAt`, write, retry-on-conflict loop, over any one store |
| `GitHubOnlyDelivery` | deliver through one store, with no fallback (live push off) |
| `LiveFirstDelivery` | deliver to a primary store, archive to a second, and hand over to a fallback when the primary fails |
| `SnapshotPublisher` | choose the delivery for this config; the only place that decides whether live push is used |

`SyncService` and `DayVisibilityService` call `SnapshotPublisher` and never learn
which delivery ran. In production the primary is the live store, the archive is
the GitHub store, and the fallback is `GitHubOnlyDelivery`. Live push is used when
the environment configures it (`LIVE_PUSH_URL` and `LIVE_PUSH_SECRET`) and the
operator's switch in `config/events.json` is on. A third delivery target is a
store adapter, a delivery (or a reuse of `LiveFirstDelivery` with different
stores), and one line in `SnapshotPublisher` and `src/app.mjs`.

### Commit conflicts

Every facility of a day writes into the same `<event>/data/<day>.json`, and
every commit of every event moves the one `main` branch of `event-data`. A
commit made against a stale sha is rejected with `409`, which means another
commit landed first — not necessarily that this file changed, since GitHub
moves the branch one commit at a time, so two syncs of different files (two
events on one day, or a sync and a Live/Hide override writing
`config/events.json`) can collide too.

A sync therefore reads, merges and commits in a loop of at most 3 attempts
(`COMMIT_ATTEMPTS`). On a `409` it re-reads the file and re-merges the facility
data it already fetched (the sheets are not read again), then commits again; any
other error is thrown, and so is a third `409`, as a `ConflictError`: the request
answers `409 { error, code: "conflict" }`, with the last GitHub error's text. The
Live/Hide republish and the registry's three switches (`SyncConfigStore.setIsLive`,
`setLivePush`, `setScoreEntry`) do the same, re-reading and re-applying their one
change. The loop is written once (`src/shared/conflictRetry.mjs`,
`withConflictRetry`) and every caller goes through it. The sync response carries
`attempts`; each retry logs `commit conflict (attempt n/3)`.

The live Worker has the same race: two syncs read version `n` and both publish
with `expectedVersion: n`. The Durable Object answers the loser `409`, and the
sync re-reads, re-merges and publishes again, up to 3 times, logging
`live version conflict (attempt n/3)`. The merge that follows is against the
snapshot that won, so neither facility's data is lost.

```bash
npm test
```

### Edit timing

Apps Script sends the edit's time as `editedAt` (ISO 8601) in the body of its
`/v3` sync; a workbook made before 3.0.0 sends it as `X-Edit-At` (epoch ms) to the
legacy route. Cloud Run ignores it unless it is within the last hour and not in the future. A
facility-scoped sync records it as that facility's `lastEditAt` in the
snapshot; a full resync carries the published value forward. Each sync logs

```
timing edit→request=<n>ms fetch=<n>ms publish=<n>ms live=<n>ms archive=<n>ms edit→published=<n>ms
```

(`edit→…` read `n/a` for a manual sync, which sends no edit time; `live` and
`archive` read `n/a` when that step did not run) and returns the same figures
as `timing`. `publish` runs from the first read to the end of the archive
commit; `live` is the reads and publishes to the Worker, all attempts.
`edit→published` is measured when the live publish succeeds, since that is
when open pages see the edit, not after the archive. To read them afterwards (PowerShell, `gcloud`
signed in; inner double quotes escaped as `\"`):

```
gcloud logging read 'resource.type=cloud_run_revision AND resource.labels.service_name=sage-tools-api AND textPayload:\"timing edit\"' --freshness=1d --format='value(timestamp,textPayload)' --project=sage-tools-api
```

#### Baseline: the time-based trigger

Before the lock-based sync, `sheets-sync.gs` scheduled a one-shot
`.timeBased().after(DEBOUNCE_MS)` trigger (`runIfSettled`) from every edit.
Measured on the Piggleball workbook on 1 October 2026 (five bursts, from the
Executions page: start of the last `onEditInstallable` to start of the
`runIfSettled` that followed):

| Burst | Delay |
| --- | --- |
| single edit | 119 s |
| single edit | 70 s |
| 3 edits | 74 s |
| 19 edits | 22 s |
| 6 edits | 108 s |

That is a median of about 74 s against the 3 s intended, and up to about two
minutes before the sync even started. Each `onEditInstallable` run took
1.3–3.3 s, mostly deleting and recreating triggers. Edits running in
parallel raced on that delete-and-create and left a stray trigger: the
19-edit burst produced a second `runIfSettled` about a minute after the
first, which is the same workbook syncing twice. The lock-based sync
replaces all of this.

#### Measured: the lock-based sync

The same workbook, 1 October 2026, after the lock-based script was pasted
in, before and after `SYNC_MIN_GAP_MS` (Cloud Run's `timing` lines plus the
Executions page). Without the gap: about 20 syncs across a single edit, two
edits 3 s apart and two bursts of rapid edits. With the 5 s gap: 10 syncs
across a single edit, a two-edit burst, a 22 s burst of about 14 edits and a
short burst of four.

| Leg | Without the gap | With the 5 s gap |
| --- | --- | --- |
| Edit → request reaches Cloud Run (`edit→request`) | 0.4–3.1 s, typically 1–2 s | 0.4–4.7 s |
| Cloud Run fetch | 0.2–1.4 s, ~0.3 s typical | 0.2–2.0 s |
| Cloud Run publish (read + commit) | 1.0–1.5 s, ~1.2 s typical | 1.1–1.3 s |
| Edit → published (`edit→published`) | 2.0–4.6 s, median ~3.3 s | 2.0–6.1 s, median ~3.5 s |
| Syncs during steady typing | one every ~3 s | one every ~5 s |
| Non-holder `onEditInstallable` run | 0.6–2.2 s, median ~1.1 s | 0.7–2.1 s, median ~1.2 s |

- A sync round takes about 3 s (the 1.5 s settle plus about 1.5 s of Cloud
  Run), so without the gap a steady stream of edits produced one sync every
  3 s: a 26 s burst of about 16 edits produced seven syncs. With the gap a
  22 s burst of about 14 edits produced five. Every edit was in the last
  sync either way. No `runIfSettled` ran, and no commit conflict occurred
  (one workbook cannot collide with itself).
- The gap costs latency only when an edit lands just after a sync started:
  it then waits for the next one, up to about 5 s (the 6.1 s figure above).
  The last edit of a burst published within 2.0 s when the gap had already
  elapsed. A lone edit after a quiet spell is unaffected.
- The lock holder's own run is longer with the gap (up to about 15 s for a
  22 s burst) because sleeping counts as trigger runtime. That cost follows
  the length of a burst, not the number of edits.
- An `onEditInstallable` run that does not hold the lock still takes
  0.6–2.2 s, median about 1.1 s, against 1.3–3.3 s before. That is the
  figure trigger-runtime quota is spent at, so a busy three-facility day
  lands nearer 50–60 minutes than the 20–30 an estimate of under a second
  per edit gives.
- `lastEditAt` is recorded when the script reads the edit, 1–2 s after the
  trigger starts, so `edit→published` slightly understates the delay from
  the keystroke.
- The first **Sync now** after a quiet spell took about 11 s although Cloud
  Run's own `fetch` and `publish` were about 1.5 s: with `--min-instances 0`
  the first request after idle pays a cold start.

#### Measured at two events: 3 October 2026

Piggleball (`piggleball-day1`, one venue, 68 matches) and PickleDrive
(`pickledrive-anniversary-2026-day1`, one venue, 152 matches, a `"team"`
workbook) ran the same day, overlapping from 12:52 to 16:07 Manila time, with
live push on. Figures are from Cloud Run's 933 `timing` lines for the day and
the 936 archive commits in `event-data`.

| Leg | Piggleball (238 syncs) | PickleDrive (695 syncs) |
| --- | --- | --- |
| `edit→request` | p50 2.2 s, p90 3.9 s, max 6.4 s | p50 2.2 s, p90 7.9 s, max 42.9 s |
| `fetch` (Sheets API) | p50 0.3 s, p90 0.4 s, max 5.7 s | p50 0.3 s, p90 22.6 s, max 57.8 s |
| `live` (Worker reads + publish) | p50 0.5 s, max 1.2 s | p50 0.6 s, max 1.1 s |
| `archive` (GitHub commit) | p50 1.1 s, max 2.4 s | p50 1.2 s, max 2.8 s |
| `edit→published` | p50 3.2 s, p90 5.2 s, max 8.8 s | p50 3.6 s, **p90 27.3 s**, max 62.8 s |

- **PickleDrive's slow syncs are its workbook's Sheets API reads.** 161 of
  its 695 reads took 10–58 s, spread across the afternoon (14:00–21:00); the
  rest took about 0.3 s. Piggleball's reads, against the same Cloud Run
  service in the same hours, never passed 5.7 s, so the service is not the
  cause. The two-way split (0.3 s or 10 s and up, nothing between) is
  consistent with the Sheets API waiting for the workbook to finish
  recalculating before it returns values. 59 of the 62 slow `edit→request`
  legs came straight after a slow read: the next sync queued behind it.
- **Production runs with `SHEETS_FETCH_TIMEOUT_MS=0`**, which turns the
  read's timeout off, so a slow read waits as long as Google takes rather
  than retrying after 8 s. Apps Script's `UrlFetchApp` call gives up at 30 s
  (`fetchTimeoutSeconds`) and `syncWithRetry_` sends the sync again 2 s
  later while the first is still running: 24 PickleDrive syncs ran past
  30 s.
- **Conflicts resolved on their first retry.** 2 live version conflicts
  (PickleDrive) and 8 archive commit conflicts (6 PickleDrive, 2 Piggleball),
  none reaching a second attempt. No snapshot ever carried a
  `failedFacilities` or `staleFacilities` entry.
- **The two events never interfered.** 5 of their publishes landed within
  1 s of each other's, and both carried on normally.
- **Trigger runtime.** Estimated as each sync's settle plus its whole
  server round trip (the lock holder blocks on `UrlFetchApp` throughout),
  the day spent about 105 minutes on PickleDrive's syncs and 16 on
  Piggleball's, before the non-holder `onEditInstallable` runs. That is over
  a consumer account's 90-minute daily allowance, and syncs never stopped,
  so the triggers' owners were either a Workspace account or more than one
  account.
- Every sync both published to the Worker and committed to GitHub, the
  commit about 0.2 s after `publishedAt` (p50; 1.6 s at most).
- **After the PickleDrive workbook was trimmed (5 October 2026).** Its
  `StackCache` tab, rewritten matchup family and 15 removed named functions
  (see the
  [team workbook recalculation](../specs/implemented/team-workbook-stack-cache-spec.md)
  and [the Named Function library](named-function-library.md#the-team-workbooks-library))
  left every published value unchanged. Three **Sync now** syncs ten seconds
  apart read in `fetch=` 345, 391 and 441 ms, in Piggleball's range and against
  the event day's p50 0.3 s and p90 22.6 s. They ran with no edit in between, so
  they show the read is fast, not how it behaves under a run of score edits.

### Facility completion

Each facility in a published snapshot carries `syncedAt` (restamped on every
successful fetch) and `completedAt`, the moment it finished play, or `null`
while matches are left. Nothing in the sheet timestamps a match, so
`src/sync/domain/facilityCompletion.mjs` infers "finished": every non-BYE match has
both scores in. `resolveCompletedAt` then decides the value against what was
already published:

| Facility now | Previously stamped | Publishes |
| --- | --- | --- |
| not finished | anything | `null`. Clearing a score reopens the day |
| finished | yes | the original stamp, carried forward |
| finished | no | now |

That is what lets `syncedAt` stay a "last heard from" time while
`completedAt` stays a fixed end time. A manual full resync restamps every
facility's `syncedAt` but leaves `completedAt` alone. An empty or header-only
CSV is never "finished", and neither is a CSV whose only rows are BYEs.

The rules copy Control Center's `rowsToMatches`, `sideIsBye`,
`seriesGameOf`/`unneededSeriesGames` and `computeFacilityProgress` exactly:
a match is played only when **both** scores are numbers, a side is a BYE
when its team code **or** either player name is `bye` (case-insensitive),
and a series Final's games after the decider don't count. A series Final is
coded `<prefix>_F_<seat>_(<game>)`: two games is twice-to-beat (seat 1 needs
one win, seat 2 two), three is best-of-3 (two each), so a twice-to-beat
Final whose #1 won game 1 leaves game 2 unscored and the facility still
finishes. Change one copy and you have to change the other (see
[Control Center](control-center.md) § Actual end).

```bash
npm test
```

checks the completion rule (BYEs, zero scores, quoted names, missing
columns) and the resync story (first finish, repeated resyncs, a cleared and
re-entered score). Unlike the other `verify-*` scripts, it covers code in
`src/`, not Apps Script.

**Apps Script side:** `apps-script/sheets-sync.gs`, installed once per facility
spreadsheet, watches that workbook's configured tabs (SCHEDULE and Court
Control by default) via an *installable* `onEdit` trigger (a bare `onEdit(e)`
can't call `UrlFetchApp`, which is why it has to be installed rather than the
default simple trigger), and POSTs to Cloud Run straight from the edit. See
[How an edit becomes a sync](#how-an-edit-becomes-a-sync) below.

The file itself is **identical in every workbook** and holds no
spreadsheet-specific values. Everything that identifies a workbook — day key,
facility name, watched tabs — lives in that project's Script Properties, set
through **SAGE → Set up live sync**, a menu-driven dialog rather than edited
constants. Setup validates what it's given before saving anything:

- A `GET /v3/diagnostics/sync` call (`GET /sync/config` in a workbook made
  before 3.0.0), gated on the entered secret, checks the secret
  itself (401 blocks) and that the entered day key is registered (blocks with
  the real list of registered day keys otherwise).
- A real test sync (`POST /v3/days/{day}/facilities/{facility}/syncs`) checks
  the facility name — a `404` with code `unknown_facility` blocks and shows its
  detail, which names the valid alternatives —
  and, on success, publishes a real snapshot as end-to-end proof the workbook
  is wired up.
- A timeout, 5xx, or unreachable host at either step is **not** blocking: the
  configuration may be correct and Cloud Run simply unavailable, so setup
  saves anyway and reports a warning instead.

Identity values (day key, facility name, watched tabs) are only honoured for
the spreadsheet they were saved for, keyed off a stored spreadsheet ID — a
workbook copied from an already-configured one reads as unconfigured rather
than inheriting the source's day key and publishing over its snapshot. The
shared secret is **not** guarded this way, since it's the same one Cloud Run
env var for every workbook of every event.

### How an edit becomes a sync

Every edit to a watched tab records `PROP_LAST_EDIT`, then tries to take the
workbook's **document lock** without waiting (`syncUntilSettled_`). The
execution that gets the lock waits `SYNC_SETTLE_MS` (1.5 s, so the second
score of a match usually lands in the same sync), syncs, and checks whether
any edit was recorded after that sync started; if so it syncs again, waiting
first so that the new sync starts at least `SYNC_MIN_GAP_MS` (5 s) after the
previous one did. Every sync is a GitHub commit (the archive; a Pages build
follows each one), so the gap caps steady typing at about one commit every 5 s instead of one every
3 s; after a quiet spell a lone edit waits only the settle. (The start time
of the last sync is kept in a script property, `lastSyncStartTime`, because a
different execution may hold the lock next.) An edit
that cannot take the lock returns immediately — the lock holder covers it,
and after releasing the lock the holder checks once more for an edit that
arrived in between.

A sync that fails with a `5xx`, a network error or a timeout is tried once
more after `SYNC_RETRY_MS` (2 s); a `4xx` is not retried, since a bad secret,
day or facility will not fix itself. Cloud Run retries its own commit
conflicts (above), so this mostly covers a network error or a failed cold
start.

The execution log says what happened at each of those points: a sync that
finds the lock taken, a retry, why a `4xx` is not retried, giving up after the
second attempt, and an edit storm handing over to the follow-up trigger. A
successful sync logs `Sync OK (200):` followed by the result as one line of
text (`describeSyncResult_`): the venue and day synced, how it was published
(the live version and archive commit, or GitHub after a failed live push) and
edit-to-published time. **Sync now** and setup's test sync show the same lines
one per row. A failure reads `Sync failed (<status> <code>): <detail>` from a
problem-details body (`describeFailure_`), the `error` of a legacy body, or the
body as text, with an HTML page's tags dropped and a long body cut at 300
characters.

The file carries `SHEETS_SYNC_VERSION`, bumped with every change to it.
**SAGE → Help** lists it, and the version of each generator the workbook also
carries (`scriptVersions_`), so anyone can tell which copy a workbook runs.

The lock is the *document* lock, not the script lock, because Pickle for
Sight's live workbooks also carry `attendance.gs`, which holds the script lock
while marking a player; sharing it would make edits made during a mark lose
their sync.

An edit storm that outlasts `SYNC_BUDGET_MS` hands over to a one-shot
time-based trigger (`runIfSettled`, `DEBOUNCE_MS` later) instead of running
past Apps Script's execution cap. That trigger is otherwise unused; a
`runIfSettled` left scheduled by an older version fires once and runs the
same loop.

Installable triggers count against the installing account's daily trigger
runtime (90 minutes on a consumer Google account, 6 hours on Workspace,
shared by every workbook that account installed triggers in). A burst spends
about the settle plus the Cloud Run round trip, so a day of a few hundred
syncs lands around 20–30 minutes.

### How the secret reaches a workbook

The secret is the one value an operator should never have to type, and the one
that has to survive a workbook being *copied* — a generated dual-meet workbook
starts life as a copy of the Dual Meet Master.

Script Properties can't do that job. They belong to the Apps Script project,
and copying a spreadsheet creates a new, empty one, so a secret set in the
Master's Project Settings reaches no copy. The secret is therefore stored as
**developer metadata on the spreadsheet itself**, which Drive does copy, under
the key `SAGE_SYNC_SECRET`. It is entered once on the Master through **SAGE →
Set shared secret**, which validates it against Cloud Run before writing.

Every copy made afterwards carries it. Setting up a facility workbook is
therefore day, venue and tabs only — the dialog reads *Secret stored* and never
asks. At the copy's first successful setup the value is promoted into that
workbook's own Script Properties, so it stands alone from then on and rotating
the Master's secret can't unconfigure a venue mid-event.

Only the secret is carried. Day key and facility name are deliberately excluded:
metadata copying is the whole point of it, so putting identity there would hand
every copy an inherited venue and defeat the guard above.

The **Set shared secret** menu item appears only where it belongs — on a
master (labelled *Replace shared secret*, since that's the rotation path) and
on a hand-built workbook that carries no secret at all. Copies don't show it.
A master is recognised two ways: it is the spreadsheet the carried secret was
entered in, or it is named `SAGE … Master` and carries a secret entered
somewhere else. The second covers the Standard Tournament Master, which is a
copy of an event workbook and so inherits that workbook's secret; using
*Replace shared secret* there once makes the secret its own. A copy is told
apart from its master by name: `Copy of …` until it is generated, then the
event's own name. `secretMenuItem_` makes the decision and is checked by
`apps-script/verify-standard-tournament-generator.mjs`.
The secret now travels inside the spreadsheet file, so sharing a copy of the
Master shares the secret with it.

`sheets-sync.gs` also declares `onOpen`, contributing **Generate
Scoresheets** (deep-links into the Scoresheet Generator with this workbook's
day and venue preselected — a link only, no HTTP request and no new OAuth
scope), **Sync now**, **Pause live sync** / **Resume live sync**, **Live
sync settings** once configured (or **Set up live sync** otherwise),
**Set shared secret** where it applies (see above), **Fill match numbers**,
and **Help**. The last two are always present regardless of configuration
state, since this is the one file guaranteed to be in every workbook. Help
shows a workflow refresher plus this workbook's live status (configured/not,
paused/not) in one dialog, written for organizers rather than developers.

**Fill match numbers** numbers the matches on `SCHEDULE`. It asks for a base
and gives the first match base + 1. It reads the grid by its fixed shape:
`Court <n>` headings on row 5 mark each court's match-number column, with
`teamCode1` one column left and `teamCode2` one column right. Slots are
two-row pairs from row 6. A slot is a match when both codes are filled in
(`-` counts as empty). Matches are numbered left to right across courts,
then down the slots, and every other slot's number cell gets `-`, so no
leftover number collides with a new one. It writes nothing if any slot has a
code on one side only, or if a code sits on a slot's second row (a grid that
doesn't start at row 6). The planning step, `planMatchNumbers_`, is pure and
separate from the Sheets reads and writes.

A base exists so a multi-venue event can give each venue's workbook its own
range (1000 → 1001…, 2000 → 2001…). A day's venues merge into one snapshot,
and the site treats `matchNumber` as unique within a day. The numbers stay
plain integers because every page parses them with `parseInt` and drops a
row that doesn't parse.

It then writes the same numbers down the `CSV` tab's `matchNumber` column,
one per row from row 2. That column is what every other `CSV` column looks
its match up by, and the site only shows matches listed in it. A generated
workbook fills it with literal numbers, so it doesn't follow `SCHEDULE` by
itself. Rows past the last match are blanked, and the site skips them. If
there are more matches than rows, row 2 is copied down first, so the new
rows carry the same relative lookup formulas the generator tiles. The tab
is left alone when that column holds formulas (it already follows
`SCHEDULE`) or when it has only a header row (there are no formulas to
copy). A final check warns if any new number is still missing from the
column. Script writes don't fire the `onEdit` trigger, so nothing
publishes until the next sync.

Pausing is for editing a watched tab (rosters, a mid-event schedule fix) without
publishing every intermediate state — it leaves the saved configuration and
the installed trigger alone, it just makes `onEditInstallable` and the
fallback trigger no-op until resumed. It's a backstop, not the primary
safeguard: the trigger doesn't exist at all until `Set up live sync` has run
once, so the normal workflow — finish rosters and schedule fixes, wire up
sync last — never needs it. `Sync now` still works while paused, since
that's an explicit manual action rather than the automatic edit-triggered
path pausing is scoped to. A generated workbook carries both this file and
one generator — `dual-meet-generator.gs` or `standard-tournament-generator.gs` — and Apps
Script silently lets the last-loaded file's `onOpen` win when two are
declared. So all three files declare a **byte-identical** `onOpen` body that
delegates to feature-detected builders (`addSyncMenuItems_` in this file,
`addGeneratorMenuItems_` in each generator), each contributing only its own
items and its own leading separator. Which declaration wins can't matter,
since the bodies are the same text. Change one file's `onOpen`, change the
other two to match — see
`sage-docs/docs/specs/implemented/sync-script-configuration-spec.md` §7 for the full
contract, and the divergences/design notes in
`sage-docs/docs/specs/implemented/scoresheet-event-picker-spec.md` §8.3 for why a
shared-contract shape was chosen over the alternatives.

## The runtime-fetched event registry

The event/day/facility registry — which spreadsheets exist, their
labels, their `isLive` state — lives at `event-data/config/events.json`, not
in `sage-tools-api` source. This is deliberate: source-code config means
adding an event or fixing a wrong sheet ID requires a full Cloud Build image
rebuild (`gcloud run deploy --source .`, which reinstalls Chromium). A
JSON file in a data repo means the same change is a commit, live within
about a minute, with no redeploy.

**`SyncConfigStore`** (`src/registry/SyncConfigStore.mjs`) owns fetching,
caching, and validating it:

- **Cache hit within the TTL** (`SYNC_CONFIG_TTL_MS`, default 60s) — no
  network call.
- **Cache miss** — fetches via `GitHubPublisher.fetchExisting()` (the same
  Contents API path used for snapshots), validates, swaps in.
- **Concurrent callers on a cold instance** share one in-flight request; a
  rejection isn't cached.
- **Fetch fails, something already held** — logs a warning, keeps serving
  the held config. A sync during a GitHub outage still fails, but at the
  same point it always would have (fetching/publishing the snapshot itself)
  — config availability doesn't add a new failure mode.
- **Fetch succeeds but validation fails** — the bad config is *refused
  wholesale*, not partially applied; the previous good config keeps serving,
  and an error names the failed rule. A typo in a commit can't crash-loop
  the service or half-apply.
- **Nothing held, no usable fetch** — falls back to a bundled seed
  (`events.seed.json`, a snapshot of the config as of the last deploy),
  logged at error level. This only fires on a cold start during a GitHub
  outage, or a cold start right after someone commits a broken config —
  exactly the scenario the whole mechanism exists to make safer.

Validation (rejects the whole file if any rule fails): a schema `version`
check, event/day keys restricted to `^[a-z0-9][a-z0-9-]*$` (they become
path segments and filenames — this is what makes a `../` traversal
impossible), day keys globally unique *across every event* (two events can
never race to publish into each other's folder, since a day key is also the
path of every sync), each day needs a `label` and a `facilities` array, and
facility names unique within a day. Full shape and rules:
[event registry schema](event-data-config.md).

**Observability:**

- `GET /ping`'s `X-Sync-Config` header reports the cached config's short
  SHA + source (`a1b2c3d/remote` or `seed/fallback`) — but never *triggers*
  a load itself, since Cloud Run's health checks hit this endpoint and it
  must stay fast even when GitHub is unreachable.
- `GET /v3/diagnostics/sync` (secret- or token-gated; legacy `GET /sync/config`)
  returns the full resolved view for debugging: SHA, source, age, every
  registered event/day, and the live push state (`live: { enabled, baseUrl, switch, active }`).

## The go-live override (`PUT /v3/days/{day}/visibility`)

Separate from a data sync: this endpoint (operator-token-only, no shared
secret; legacy twin `POST /sync/:day/live`) is Mission Control's public-site
kill switch. It writes the day's
`isLive` value (`true` / `false` / `"auto"`) into `config/events.json`, then
immediately republishes that day's snapshot with the new value stamped in
— so a later real sync can never silently revert an operator's override,
since every sync re-reads `isLive` from the registry each time. See
[Auth](auth.md) for the token vs. shared-secret distinction.

`DayVisibilityService` writes the config first and then asks the
`SnapshotPublisher` to republish. That goes through the live Worker when live push
is on (then is archived to GitHub), so pages holding a WebSocket hide or show the
day within a second or two instead of waiting for the next score edit. If the
Worker cannot be reached it is a GitHub commit, as without live push. A day that
has never been published is still updated in the config (`republished: false`),
and the first real sync stamps it in.

## Score entry writes

Score entry is the one path where `sage-tools-api` writes scores into a facility
workbook instead of reading them. `PUT /v3/days/{day}/facilities/{facility}/matches/{matchNumber}/score`
(`src/scores/`, `ScoreService.submit`) writes one match's two score cells in the
workbook's `SCHEDULE` tab as the attendance service account, then publishes the
day itself, in the same request. Usage and the dialog:
[Entering a score](../features/control-center.md#entering-a-score).

**Why the API publishes.** The normal sync starts from an installable onEdit
trigger, and Google documents that script executions and Sheets API requests do
not fire triggers. A write by the API therefore starts nothing, and without a
publish the pages would show the old score until someone next typed in the sheet.
`ScoreService` calls `syncService.syncDay(day, { facilityName, editAt })` for that
facility, with `editAt` as the time of the save, so the snapshot's edit-to-sync
diagnostics read as they do for a typed edit. The publish finishes before the
response is sent, because Cloud Run throttles CPU once a response goes out. A
publish failure does not undo the write: the response is 200 with
`sync: { ok: false, error }`, and the client tells the person the score is in the
sheet but not published.

**Where it writes.** `scheduleGrid.mjs` finds the match: every workbook type lays
`SCHEDULE` out in 8-column court blocks from column D, with data from row 6 and two
rows per slot. The match number is in column 6 + 8k, the two codes either side of it
and the two scores 3 and 4 columns to its right. The API reads the whole tab, finds
the cell holding the number (none is a 404, more than one a 422 that names the cells),
then reads that row's two score cells with formulas visible: a formula or text in a
score cell is a 422, so the API never overwrites one. `SheetsClient.writeScores` and
`clearScores` accept only a range of the form `SCHEDULE!<team 1 score><row>:<team 2
score><row>` at row 6 or below, checked before any request is made. A clear uses
`values:batchClear`, so the cells are truly empty, which the team workbooks'
`ISBLANK` test needs. `updateValues`, attendance's write, keeps its own `ATTENDANCE!A:G`
allowlist.

**The optimistic check.** The client sends the match as it showed it (`expected`: the
two codes and two scores). If the sheet's codes differ, or its scores differ from both
`expected` and the new scores, nothing is written and the answer is a 409 carrying the
sheet's current values. A sheet that already holds the new scores is `unchanged`: nothing
is written but the publish still runs, because an earlier save may have written without
publishing. There is a window of a few hundred milliseconds between the API's read and
its write in which a person typing the same match in the sheet would be overwritten.
Closing it needs the workbook's document lock, which only Apps Script can take; the
publish that follows reads whatever the sheet then holds, so pages always show the sheet.

**The 60-second timeout.** `index.mjs` gives score entry its own `SheetsClient`, with a
60-second timeout instead of attendance's 5. A team workbook recalculates when it is
written and can take up to a minute to answer the read that follows. The client's
two-minute request timeout covers the read, the write and the publish.

**Audit.** Every save logs one line: the day, facility, match, cells, old and new
scores, who (`operator` or `scorer`) and whether the publish worked.

Who may call it, and the switch that stops scorer links: [Auth § Scorer tokens](auth.md#scorer-tokens).

## Sheet tabs are addressed by name, not GID

Both fetch paths address a facility's matches/standings tabs by name
(`matchesSheetName`/`standingsSheetName`, both overridable per day, default
`CSV`/`STANDINGSCSV`) — never by numeric tab GID, since a GID is assigned
per-workbook and doesn't carry over if a spreadsheet is ever duplicated
from another event's.

## Live push delivery

Without push, a page polls the published GitHub Pages snapshot every 10
seconds (`POLL_INTERVAL_MS`). Measured at Pickle for Sight (27 September
2026), Cloud Run's sync takes **1.6s** (p50) but GitHub Pages takes **25s**
(p50) / **52s** (p90) from the data commit to a finished deploy, longer in busy
stretches because each new commit cancels the build in progress. With the poll,
a score reached a viewer in roughly **35–40 seconds** once it was published,
although the edit itself reaches publication in about 2–6 s (see
[Measured: the lock-based sync](#measured-the-lock-based-sync)). Live push
removes both the Pages build and the poll from the path: an edit reaches an
open page in about **2–5 seconds**.

[Live push delivery](../specs/in-progress/durable-object-push-spec.md) is the
spec; [Immediate sync](../specs/implemented/immediate-sync-spec.md) is the
prerequisite it builds on.

### The Worker

`sage-tools-api/live-worker/` is a Cloudflare Worker, `sage-live`, plus a
Durable Object class, `DayChannel`: one object per `<event>/<day>`, holding
`{ version, snapshot }`. The Cloud Run image never contains it, and it does not
deploy on push — see [Deployment](deployment.md#live-worker-cloudflare).

| Request | Auth | Response |
| --- | --- | --- |
| `GET /health` | none | `200 ok` |
| `GET /live/<event>/<day>` (WebSocket upgrade) | `Origin` must be the site, localhost, or absent | `101`, then `{type:"snapshot"\|"empty"}` messages; `403` wrong origin, `426` no upgrade |
| `GET /snapshot/<event>/<day>` | `X-Publish-Secret` | `200 {version, snapshot}`, or `404 {version: 0}` |
| `POST /publish/<event>/<day>` `{expectedVersion, snapshot}` | `X-Publish-Secret` | `200 {version, clients}`; `409` if `expectedVersion` is not the current version; `400` bad body |

Event and day keys match `^[a-z0-9][a-z0-9-]{0,63}$`; anything else is `404`.
A wrong or missing secret is `401`. A socket receives the current snapshot
(or `empty`) on connect and another after every accepted publish. Clients
send only the text `ping`, which Cloudflare answers `pong` without waking the
object, so an idle object hibernates and is not billed for duration. The
version check and the write happen with no other request in between, which is
what makes `expectedVersion` a safe compare-and-set.

### How a sync publishes

After fetching the facilities, `LiveFirstDelivery` (live push on) does this:

1. Read the day's snapshot and version from the object, and GitHub's copy
   alongside it. Merge into whichever is newer by `publishedAt`. They differ
   after any stretch where syncs went through GitHub alone (the switch below, a
   Worker outage, live push unconfigured): merging into the object's older copy
   would republish, and then archive over GitHub, facility data GitHub already
   has newer. The object's version is still what the publish checks. With an
   empty object (the first sync since it was created) GitHub's copy is what is
   merged into. GitHub's read failing only matters when the object holds
   nothing.
2. Merge (the same `mergeSnapshot` as without live push), stamp
   `publishedAt`, and publish with the version just read. On `409`, go back to
   step 1, up to 3 attempts.
3. **Archive**: commit the same snapshot to GitHub. A `409` there is retried
   with the object's current snapshot, which is the newest there is. Any other
   failure — including GitHub's `403`/`429` rate limits — is logged and
   reported as `archive: { committed: false }`; the request still succeeds,
   because the live copy is correct and already visible. The next sync's
   archive commit catches GitHub up.

If a Worker call throws, or three publishes conflict, the delivery hands over to
its fallback and the sync publishes through GitHub alone and reports `live: { published: false, error }`. The
response otherwise gains `live`, `archive` and `timing.liveMs` / `archiveMs`;
`commitSha` is the archive commit's, or `null` if it failed. The live publish
comes before the GitHub commit and both finish before the response is sent:
Cloud Run throttles CPU after a response, so the archive is never
fire-and-forget.

Every snapshot carries `publishedAt`, restamped on every publish (live, GitHub
and Live/Hide). Pages use it to decide which of two copies is newer.

**Team rosters.** For a `"type": "team"` day, `sheetsFor(day)` also names a
roster tab (`Teams`, or the day's `rosterSheetName`). `SheetsCsvFetcher` adds
it as a third range to the same `batchGet`, and
`rosterCsvFromTeamsValues` (`src/sync/domain/teamRoster.mjs`) turns it into
`facilities[].rosterCsv`: `teamCode,player,level,gender`, and no other
column of the tab. A workbook without the tab is fetched again without it, so
a missing roster never fails a sync. A fetch that brings no roster (that case,
or the gviz fallback, which never reads it) keeps the last published
`rosterCsv` in the merge.

**After publishing**, `SyncService` awaits its optional `onFacilitiesSynced` hook
with the facilities fetched fresh this round. That is
[attendance's](event-attendance.md) roster update: nothing for an event without
attendance, and no Sheets call when a facility's roster is unchanged. It is
bounded to 6 seconds and never changes the sync's response; a failure there is
logged and the next sync tries again. Live/Hide does not call it.

### The sync-method switch

Control Center's Mission Control has a **Sync method** switch between **Live
push + GitHub** and **GitHub only**, for an emergency where live updates
misbehave and waiting to clear `LIVE_PUSH_URL` on Cloud Run is too slow. It
calls `PUT /v3/settings/live-push` (operator token only), which writes a top-level
`livePush` boolean into `event-data/config/events.json`. `false` makes every
sync and every Live/Hide publish to GitHub alone, exactly as with live push
unconfigured; absent or `true` leaves it to the environment. It is stored in
the config, not in the Worker, so it works when the Worker is the thing that is
broken. The instance that takes the request applies it at once; the others
within `SYNC_CONFIG_TTL_MS`. Pages with an open socket notice within a minute:
GitHub's copies carry newer `publishedAt` values, which the safety poll
prefers, and from then on they poll GitHub as before push existed. The switch
cannot turn on live push the environment does not configure.
`GET /v3/diagnostics/sync` reports it as `live.switch` (`on`/`off`) and `live.active`
(configured and switched on).

`LIVE_PUSH_URL` (the Worker's base URL), `LIVE_PUSH_SECRET` (its
`PUBLISH_SECRET`) and `LIVE_PUSH_TIMEOUT_MS` (default 4000) configure it.
Clearing `LIVE_PUSH_URL` on the Cloud Run service is the rollback switch: a
new revision, no code change. `GET /v3/diagnostics/sync` reports
`live: { enabled, baseUrl }`, never the secret.

### Pages

Each page that shows live data carries one identical block, marked
`LIVE CHANNEL`, which opens a WebSocket to `/live/<event>/<day>` for the day
it is showing and reconnects with backoff. The pages are Control Center, both
templates' `index.html` and `schedule.html`, and the same pair of every event
that has not finished, plus the [scorer page](scorer-page.md) of an event that uses scorer links. The block must stay byte-identical in every copy. A
finished event's pair keeps the block with `LIVE_BASE_URL` empty, the one
line that differs: nothing is published for it any more, so its pages read
GitHub and hold no socket against the Worker's request cap.

`fetchDaySnapshot` keeps its name and callers. While the socket is open it
returns the pushed snapshot without touching the network; every
`LIVE_SAFETY_POLL_MS` (60 s) it fetches GitHub once and keeps whichever copy
has the later `publishedAt`, which catches a push path that has quietly
stopped while the socket stays open. If GitHub fails (a `404` before a day's
first archive, say) but a pushed copy exists, the pushed copy is used. With
the socket down, the page polls GitHub every 10 seconds exactly as it did
before push existed. A hidden tab closes its socket and reopens it when shown.
Pages opened with `?fixture=` on localhost never connect.

`LIVE_BASE_URL` is one constant, inside the block, in every page. Empty
disables push for that page. Control Center's Mission Control reads **Live
updates: push connected** or **Live updates: polling GitHub (push not
connected)**.

### Limits

On Cloudflare's free plan a Worker and its Durable Objects are capped at
100,000 requests a day per account, counting one per connect, publish or read
and 1 per 20 client pings (every 50 s per open page). That is about 10k a day
for 200 viewers all day and about 93k for 2,000. At the cap, requests fail and
pages stay on GitHub polling: today's speed, not an outage. The limit resets
at 00:00 UTC, which is 08:00 in Manila. Some venue networks block WebSockets;
those pages stay on polling. Anyone can open a socket to a valid-looking key
(the data is already public on GitHub Pages); the `Origin` check stops other
websites embedding the feed, not scripts.

---
**Features:** [Control Center's Mission Control tab](../features/control-center.md#mission-control) uses this pipeline's resync/go-live actions.
