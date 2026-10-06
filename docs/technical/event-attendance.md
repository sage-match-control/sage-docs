# Event attendance

Staff check-in for any registered event, written by `sage-tools-api` straight
into each facility workbook's `ATTENDANCE` tab, as a dedicated service
account. One row per person, covering every category they play.

Spec: [attendance for every event](../specs/implemented/multi-event-attendance-spec.md).
The earlier, Pickle for Sight design is at the end of this page.

## Flow

```
Control Center "Attendance" tab          events/<key>/attendance.html (desk page, "desks" mode)
        |   the same ATTENDANCE CLIENT block in both
        | read:  ATTENDANCE tab CSV export (gviz), every 10 s while visible
        | write: PUT /v3/days/{day}/facilities/{facility}/people/{personKey}/attendance
        |        Authorization: Bearer <operator token | desk token>
        v
sage-tools-api   src/attendance/
        | reads:  Sheets API, existing API key (GOOGLE_SHEETS_API_KEY)
        | writes: Sheets API, the service account's token (metadata server)
        v
Facility workbook -- ATTENDANCE tab
        ^
        | after every sync of a facility, only when the roster changed
a sync (Apps Script or Control Center) -- SyncService -- onFacilitiesSynced hook
```

The live Worker is not used: its channel is public.

## The setting

`events[<event>].attendance` in `event-data/config/events.json`:
`"console"`, `"desks"`, or absent. Validated by `SyncConfigStore`;
`SyncConfigSnapshot.getEvent()` returns it and `getDay()` returns the day's
`date`. See [event registry schema](event-data-config.md).

## The ATTENDANCE tab

Row 1 is `key | player | teams | categories | present | timeIn | withdrawn`
(columns A to G). One row per person from row 2. `key` is the person key:
`personKey()` in `src/attendance/domain/personKey.mjs` (accents stripped, whitespace
collapsed, lower-cased), the only definition of identity. `teams` and
`categories` are comma-joined and parallel. `timeIn` is `yyyy-MM-dd HH:mm`
(Asia/Manila). Columns H onward belong to people; the API never reads or
writes them.

Both generators (`sheet-generator.gs`, `standard-generator.gs`) create this tab
empty when they build a workbook: the header, a frozen first row, and the
column J list below. `verify-sheet-generator.mjs` and
`verify-standard-generator.mjs` import `HEADERS` from `attendanceTab.mjs`, so
the generated header cannot drift from the one the API requires. A tab of that
name already in the workbook is left alone. The API creates the tab itself only
when it is missing (a workbook made before this, or a hand-built one such as
PickleDrive's).

One exception to "never past G": when the API creates the tab it also writes a
**Not yet in** list in column J (`J1` the header, `J2` a `FILTER` formula listing
everyone neither present nor withdrawn). That write is fixed in
`SheetsClient.createAttendanceTab` and entered as a formula; `updateValues`
still refuses anything outside A to G. A tab that already exists is not changed.

Cells are written with `valueInputOption: RAW`, and `present` and `withdrawn`
are always booleans, so a column never mixes types (gviz's CSV export empties
a column whose type it guesses wrongly). The API refuses to touch a tab whose
first row is not exactly this header (`AttendanceLayoutError`, 409).

## The roster

`src/attendance/domain/roster.mjs` (pure): standard from `STANDINGSCSV` (team code
`^[A-Z0-9]+_\d+$`, category before the first `_`), dual meet likewise
(`^[A-Z0-9]+_[A-Z0-9]+_\d+$`, the middle segment), team events from the
`Teams` tab (`Team Code`, and `Player` or `FINAL LEVEL ORDER`). Playoff-seat
rows, names equal to the team code and `bye` are skipped. The same name twice
in one category is one person with a warning.

`attendanceTab.mjs` plans a **roster update** (`planReconcile`): rows are only
updated in place or added below the last row, never inserted or deleted; every
change goes in one `values:batchUpdate`. Duplicate keys are merged into the
first row and the later one blanked. A person not on the roster gets
`withdrawn = TRUE` and keeps `present` and `timeIn`.

`AttendanceService.reconcileAfterSync` is the sync hook. It does nothing for an
event without attendance, skips a facility whose roster fingerprint (sha1) is
unchanged since this instance's last successful update, caches the `Teams` tab
for 5 minutes, and is bounded to 6 seconds. It never throws. After writing
new rows it re-reads them, and leaves the fingerprint unrecorded if another
instance overwrote them, so the next sync adds the missing people.
`reconcileDay` is the manual one: forced, never cached.

## Marking

`AttendanceService.mark` finds the first row with the key and writes `E:F`.
It is idempotent, so a repeat keeps the first `timeIn`. No lock: two people
marking different people write different rows.

## Routes

Mounted at `/v3` (`src/attendance/routes.mjs`), which every page calls:

| Route | Auth |
| --- | --- |
| `PUT /v3/days/{day}/facilities/{facility}/people/{personKey}/attendance`, body `{ present }` | operator token, or a desk token |
| `POST /v3/days/{day}/attendance/desk-links` | operator token only |
| `POST /v3/days/{day}/attendance/reconciliations`, optional body `{ facility }` | operator token only |

Each has a frozen `/v1` twin (`PUT /v1/days/:day/facilities/:facility/attendance/:key`,
`…/desk-links`, `…/reconciliations?facility=`) that copies of the pages from
before 3.0.0 call; it answers the same, with the plain `{ error, code }` body
instead of problem details and `400` instead of `404` for an unknown day or
facility ([API](api.md)).

CORS allows `PUT`. Errors carry `error` in words (inside a problem details body
on `/v3`), never raw Google JSON.

## Writing to the workbook

`SheetsClient` reads with the API key and writes with the service account's
token (`GoogleAccessToken`, from the metadata server; `GOOGLE_ACCESS_TOKEN`
overrides it for local development). **Every attendance write range must be
inside `ATTENDANCE!A:G`** (`updateValues`); anything else throws before a request
is made, so a bug here cannot overwrite scores or formulas. The client’s only other
writes are score entry’s, which have their own allowlist (one match’s two `SCHEDULE`
score cells; see [sync pipeline](sync-pipeline.md#score-entry-writes)). There is no append call. Google
`429` is retried once after a second, then answers 503.

## Desk tokens

See [auth](auth.md). A desk token carries a day and expires at the end of that
day in Manila time; it is accepted only while the event is `"desks"`.

## The client

One block of plain JavaScript, `ATTENDANCE CLIENT`, byte-identical in
`tools/control-center.html` and `_templates/attendance/attendance.html` (and
each event's `attendance.html` made from it). It injects its own styles, so a
host needs no CSS for the list. `createAttendanceView` renders the list, the
category bar and search, polls, and marks optimistically.

Control Center loads every facility of the day so counts are complete; the
desk page loads only the venue being shown. With `?fixture=<name>` on
localhost both read `_fixtures/` CSVs and mark in memory; `?attfail` makes the
first mark fail.

## Tests

`npm test` in `sage-tools-api` (`node:test`): unit tests for each module
under `test/unit/attendance/` and an HTTP test of the routes. The site has
fixtures and manual checks (spec section 7.3).

---

# Earlier version: Pickle for Sight

The notes below describe Pickle for Sight's `attendance.gs` and
`events/pickle-for-sight-2026/attendance.html`. The script runs in that event's
workbooks but is no longer kept in `sage-tools-api`: its last version, with its
harness `verify-attendance.mjs`, is in `apps-script/` at commit `198f02c`.


Pickle for Sight's check-in page wrote to Google Sheets through a web app in
each workbook. Two parts:

- `attendance.gs` — bound Apps Script, pasted into
  each of an event's **live** workbooks (never a master) beside
  `sheets-sync.gs` and the generator, and deployed there as a web app. Not
  part of the Cloud Run service: changing it is not a deploy and does not
  bump `package.json`. Also holds `attendanceResync`, run by hand from the
  Apps Script editor once at setup and again after any player swap.
- `events/pickle-for-sight-2026/attendance.html` — a self-contained static
  page, like the event's `schedule.html`.

Spec: [event attendance](../specs/implemented/event-attendance-spec.md).

### The web app

Deployed per workbook with **Execute as: Me** and **Who has access: Anyone**,
so desk staff need no Google account. One `/exec` URL per venue; the page
lists them in its `VENUES` constant, for marking only — the page's reads go
elsewhere (below).

- `doGet` returns `{ ok, pairs: [{ teamCode, category, players: [{ slot,
  name, present, timeIn }] }] }`. The page no longer calls this; it's kept as
  a diagnostic endpoint, useful to open directly in a browser.
- `doPost` takes a JSON body `{ teamCode, slot, present }` and returns the
  saved player. The code must be a real pair row and the slot a named player.
- `attendanceResync` pre-fills `ATTENDANCE` with every player at
  `present: FALSE`, and corrects any row whose stored name no longer matches
  the roster (a re-draw), resetting it to the new name and `present: FALSE`.
  A row whose name still matches is left alone. Not reachable through
  `doGet`/`doPost` — run it by hand from the Apps Script editor's function
  dropdown: once at setup (the page's CSV read depends on every player
  already being a row), and again any time a player is swapped.
- A failure comes back as `{ ok: false, error }`. `ContentService` cannot set
  an HTTP status, so every reply is a 200.

Editing `doGet`/`doPost` changes nothing until **Deploy → Manage
deployments → edit → Version: New version**. An `/exec` URL keeps serving
the version it was given. `attendanceResync` runs from the editor, not
through `/exec`, so it needs no redeploy — only the file's latest content
pasted in.

#### Where the roster and the marks live

Names are read from `STANDINGSCSV` columns A:C. Only pair rows count — codes
like `HIMD_3`. The tab also lists playoff seats (`HIMD_QF_1`) that repeat the
names of pairs already listed; those are skipped, as is a pair whose names
are still its code.

Marks go to an `ATTENDANCE` tab the script creates on the first POST:
`teamCode | slot | name | present | timeIn`, one row per player ever marked,
found by team code and slot rather than by row. It is a separate tab rather
than a column beside `STANDINGSCSV` because:

- The sync reads the whole `STANDINGSCSV` tab (`SheetsCsvFetcher` asks for it
  by tab name), so anything added there is published to `event-data`.
  `ATTENDANCE` is never read by the sync.
- `STANDINGSCSV!A2` is a spill. A hand-kept column beside it is tied to row
  position and silently shifts when the spill does.

A stored mark counts only while its `name` still matches the roster — but
only at the moment something writes the row (a mark, or `attendanceResync`).
Nothing re-checks it on every read any more (see below), so a re-draw needs
`attendanceResync` run again before the swap is reflected.
`timeIn` is `yyyy-MM-dd HH:mm` in the spreadsheet's time zone, stored as
text; marking an already-present player keeps the first time.

#### Concurrency

Every POST holds the script lock, so two phones marking at once cannot both
append a row for the same player. Script writes don't fire the installable
`onEdit`, so a mark never triggers a sync.

### The page

Reads and writes go to two different places. **Reads** fetch the venue
workbook's own published CSV export of `ATTENDANCE` directly —
`https://docs.google.com/spreadsheets/d/<sheetId>/gviz/tq?tqx=out:csv&sheet=ATTENDANCE`,
parsed client-side and grouped by the `teamCode` prefix for category, the
same rule `attendance.gs` uses server-side. `sheetId` lives in `VENUES`
alongside the write `url`; `gviz` takes the tab by **name**, so no numeric
gid needs finding or hardcoding. This needs the workbook shared "Anyone with
the link — Viewer" — both Pickle for Sight workbooks already are.

Reads bypass `attendance.gs` on purpose: the web app runs **Execute as:
Me**, so every phone's `doGet` would share one Google account's
simultaneous-execution quota with every `doPost`. The CSV export isn't part
of that quota, and measured against the live workbook it reflects a write in
under a second — no meaningful caching lag. The cost is that the roster
seen by the page is exactly whatever's in `ATTENDANCE`: an unmarked player
has to already be a pre-filled row to show up at all, and a swap needs
`attendanceResync` run again (above) before the page notices.

**Writes** still go through the web app: `fetch(url, { method: 'POST',
body: JSON.stringify(…) })` with **no headers** — a string body goes as
`text/plain`, which needs no CORS preflight, and Apps Script cannot answer
one. The reply follows Apps Script's redirect to
`script.googleusercontent.com`, which is what makes it readable from the
page.

**Shirt size** is an optional, hand-added column of `ATTENDANCE`. The page
looks for the first header matching `TShirt Size`, `T-Shirt Size`,
`Shirt Size`, `tshirtSize` or `shirtSize` and shows the value as a chip
beside the name. A search equal to a size (case-insensitive, exact) matches
it. `attendance.gs` neither reads nor writes the column. It finds rows by
team code and slot and writes only its own five columns, so the column
survives marks and `attendanceResync`. A swap rewrites the row's name in
place, so the size there still belongs to the old player until someone
edits it by hand.

`timeIn` is displayed through `clockTime`, which takes the clock off the end
of whatever the cell holds (24-hour or already AM/PM) and shows it as
`h:mm AM/PM`. A shape it doesn't recognise falls back to the last five
characters.

A switch updates at once and locks until the save replies; a failed save
snaps back and says why. The page polls every 30 seconds while visible and
reloads when it becomes visible again. A player with a save in flight keeps
its local state through a reload, since the sheet may not have it yet.

### Verifying a change

The harness, `verify-attendance.mjs`, is at the same commit as the script.
It runs the real `doGet`/`doPost`/`attendanceResync` against
`apps-script/mock-apps-script.mjs`: roster selection, marking and unmarking,
every refusal, the re-draw rule, lock release, resync's pre-fill/no-op/swap
behavior, and that `STANDINGSCSV` is never written. It also fails if a
top-level name in `attendance.gs` is declared by `sheets-sync.gs` or either
generator — they share one script project, where a repeated name silently
replaces the other file's.

The page has no test file. The spec's §5.4 serves it with `fetch` stubbed and
the real roster from `event-data`; the CSV read path was additionally
verified against the live PCPH Main workbook (§11 of the spec).

---
**Features:** [event attendance](../features/event-attendance.md)
