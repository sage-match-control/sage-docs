# Sync pipeline

How a score typed into a Google Sheet ends up on the public site, and how
`sage-tools-api` knows which spreadsheets belong to which event without a
redeploy.

## The sheet → GitHub path

```
Facility Google Sheet
   |  installable onEdit trigger, document-locked, syncs from the edit itself
   |  (scripts/sheets-sync.gs)
   v
POST /sync/:day?facility=<name>   (X-Sync-Secret header, or an operator's bearer token;
   |                               X-Edit-At header carries the edit time)
   |  SheetsCsvFetcher (default) or GvizCsvFetcher (?method=csv fallback)
   |  merge with the currently-published snapshot — never drop a facility on failure;
   |  re-read and re-merge on a 409 commit conflict (3 attempts)
   v
GitHub Contents API commit -> event-data/<event-key>/data/<day>.json
   |  GitHub Pages redeploys on push (no cache-purge step)
   v
Event page fetches https://sage-match-control.github.io/event-data/<event-key>/data/<day>.json
```

Key pieces, in `sage-tools-api/src/sync/`:

- **`SheetsCsvFetcher`** — the default path, reads via the Sheets API.
- **`GvizCsvFetcher`** — a fallback path (`?method=csv`) using Google's
  `gviz` CSV export, for when the Sheets API path has trouble.
- **`SyncService`** — orchestrates a sync: resolves the day's config,
  fetches each facility (in parallel, via `ConcurrencyPool`), merges results
  with whatever's already published (a facility that fails to fetch keeps
  its last-known-good data rather than vanishing from the snapshot), and
  hands the result to the publisher.
- **`GitHubPublisher`** — wraps the Contents API. Used both to publish a new
  snapshot and (shared with the config store below) to read
  `config/events.json`. A failed commit throws an error carrying the HTTP
  `status`.

Both fetchers are pure functions of `(facility, sheets)` — neither imports
config directly; `SyncService` resolves the day's sheet names once and
passes them down.

### Commit conflicts

Every facility of a day writes into the same `<event>/data/<day>.json`, and
every commit of every event moves the one `main` branch of `event-data`. A
commit made against a stale sha is rejected with `409`, which means another
commit landed first — not necessarily that this file changed, since GitHub
moves the branch one commit at a time, so two syncs of different files (two
events on one day, or a sync and a Live/Hide override writing
`config/events.json`) can collide too.

`SyncService.syncDay` therefore reads, merges and commits in a loop of at
most 3 attempts (`COMMIT_ATTEMPTS`). On a `409` it re-reads the file and
re-merges the facility data it already fetched (the sheets are not read
again), then commits again; any other error, or a third `409`, is thrown.
`SyncService.setLiveOverride` and `SyncConfigStore.setIsLive` do the same,
re-reading and re-applying their one change. The sync response carries
`attempts`; each retry logs `commit conflict (attempt n/3)`.

```bash
node scripts/verify-sync-merge.mjs
```

### Edit timing

Apps Script sends the edit's time as `X-Edit-At` (epoch ms). Cloud Run
ignores it unless it is within the last hour and not in the future. A
facility-scoped sync records it as that facility's `lastEditAt` in the
snapshot; a full resync carries the published value forward. Each sync logs

```
timing edit→request=<n>ms fetch=<n>ms publish=<n>ms edit→published=<n>ms
```

(`edit→…` read `n/a` for a manual sync, which sends no edit time) and returns
the same figures as `timing`. To read them afterwards (PowerShell, `gcloud`
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

### Facility completion

Each facility in a published snapshot carries `syncedAt` (restamped on every
successful fetch) and `completedAt`, the moment it finished play, or `null`
while matches are left. Nothing in the sheet timestamps a match, so
`src/sync/facilityCompletion.mjs` infers "finished": every non-BYE match has
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

The rules copy Control Center's `rowsToMatches`, `sideIsBye` and
`computeFacilityProgress` exactly: a match is played only when **both**
scores are numbers, and a side is a BYE when its team code **or** either
player name is `bye` (case-insensitive). Change one copy and you have to
change the other (see [Control Center](control-center.md) § Actual end).

```bash
node scripts/verify-facility-completion.mjs
```

checks the completion rule (BYEs, zero scores, quoted names, missing
columns) and the resync story (first finish, repeated resyncs, a cleared and
re-entered score). Unlike the other `verify-*` scripts, it covers code in
`src/`, not Apps Script.

**Apps Script side:** `scripts/sheets-sync.gs`, installed once per facility
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

- A `GET /sync/config` call, gated on the entered secret, checks the secret
  itself (401 blocks) and that the entered day key is registered (blocks with
  the real list of registered day keys otherwise).
- A real test sync (`POST /sync/:day?facility=`) checks the facility name —
  an "unknown facility" rejection blocks and names the valid alternatives —
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
previous one did. Every sync is a GitHub commit and a Pages build, so the
gap caps steady typing at about one commit every 5 s instead of one every
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

The lock is the *document* lock, not the script lock, because `attendance.gs`
shares the project in live workbooks and holds the script lock while marking
a player; sharing it would make edits made during a mark lose their sync.

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
`scripts/verify-standard-generator.mjs`.
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
one generator — `sheet-generator.gs` or `standard-generator.gs` — and Apps
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

**`SyncConfigStore`** (`src/sync/SyncConfigStore.mjs`) owns fetching,
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
`/sync/:day` route), each day needs a `label` and a `facilities` array, and
facility names unique within a day. Full shape and rules:
[event registry schema](event-data-config.md).

**Observability:**

- `GET /ping`'s `X-Sync-Config` header reports the cached config's short
  SHA + source (`a1b2c3d/remote` or `seed/fallback`) — but never *triggers*
  a load itself, since Cloud Run's health checks hit this endpoint and it
  must stay fast even when GitHub is unreachable.
- `GET /sync/config` (secret- or token-gated) returns the full resolved
  view for debugging: SHA, source, age, every registered event/day.

## `/sync/:day/live` — the go-live override

Separate from a data sync: this endpoint (operator-token-only, no shared
secret) is Mission Control's public-site kill switch. It writes the day's
`isLive` value (`true` / `false` / `"auto"`) into `config/events.json`, then
immediately republishes that day's snapshot with the new value stamped in
— so a later real sync can never silently revert an operator's override,
since every sync re-reads `isLive` from the registry each time. See
[Auth](auth.md) for the token vs. shared-secret distinction.

## Sheet tabs are addressed by name, not GID

Both fetch paths address a facility's matches/standings tabs by name
(`matchesSheetName`/`standingsSheetName`, both overridable per day, default
`CSV`/`STANDINGSCSV`) — never by numeric tab GID, since a GID is assigned
per-workbook and doesn't carry over if a spreadsheet is ever duplicated
from another event's.

## Live delivery today, and a planned upgrade

Today, the client polls the published GitHub Pages snapshot directly on a
10-second interval (`POLL_INTERVAL_MS`). The URL carries a `?t=` query
param, but GitHub's CDN ignores query strings in its cache key; what keeps
the data fresh is that Pages purges its cache on every deploy.

Measured at Pickle for Sight (27 September 2026): Cloud Run's sync takes
**1.6s** (p50), and GitHub Pages takes **25s** (p50) / **52s** (p90) from the
data commit to a finished deploy — longer in busy stretches, because each
new commit cancels the build in progress. With the poll, a score typed into
a sheet reaches a viewer in roughly **35–40 seconds**, plus however long the
Apps Script time-based trigger takes to fire, which is not yet measured.

**Not yet built:** [Immediate sync](../specs/not-started/immediate-sync-spec.md)
replaces the delayed Apps Script trigger with a sync straight from the edit,
records how long each step takes, and retries a sync that loses a GitHub
commit race instead of dropping it. On top of that, two alternative specs
cut the GitHub Pages wait:
[Live push delivery](../specs/not-started/durable-object-push-spec.md) has a
Cloudflare Durable Object push each snapshot to open pages over WebSockets
(~2–5s), and [Fast data delivery](../specs/not-started/fast-data-delivery-spec.md)
puts Cloudflare R2 behind a CDN with pages polling a tiny pointer file every
3 seconds (~5–7s). Both keep GitHub as archive and fallback. None of this is
reflected in this doc site's architecture pages until it ships.

---
**Features:** [Control Center's Mission Control tab](../features/control-center.md#mission-control) uses this pipeline's resync/go-live actions.
