# Event attendance

Staff check-in for any registered event, written by `sage-tools-api` straight
into each facility workbook's `ATTENDANCE` tab, as a dedicated service
account. One row per person, covering every category they play.

Spec: [attendance for every event](../specs/implemented/multi-event-attendance-spec.md).
The earlier, Pickle for Sight design is at the end of this page.

## Flow

```
Control Center "Attendance" tab          events/<key>/attendance.html (desk page, "desks" mode)
        |   the same view in both (lib/v1/views/attendance-view.js)
        | read:  GET /v3/days/{day}/facilities/{facility}/people, on open, on a nudge,
        |        on coming back into view, on Refresh (nothing polls)
        | write: PUT /v3/days/{day}/facilities/{facility}/people/{personKey}/attendance
        |        Authorization: Bearer <operator token | desk token>
        v
sage-tools-api   src/attendance/
        | reads:  Sheets API, the service account's token (GOOGLE_SHEETS_API_KEY is the fallback)
        | writes: Sheets API, the service account's token (metadata server)
        | nudge:  after a write that changed the tab, POST /nudge/{event}/{day} {facility}
        v                                                  |
Facility workbook -- ATTENDANCE tab                        v
        ^                            live Worker "sage-live": object attendance:<event>/<day>
        |                                    | {"type":"attendance","facility":"Gym"}
        |                                    v
        |                            every other open page, over wss://.../attendance/<event>/<day>,
        |                            which re-reads that facility's list from the API
        | after every sync of a facility, only when the roster changed
a sync (Apps Script or Control Center) -- SyncService -- onFacilitiesSynced hook
```

The Worker's snapshot channel is public, so no check-in state travels over a socket:
the Worker relays only a [nudge](#push), and each page re-reads the list from the API with its
own token.

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
(Asia/Manila). Columns H onward belong to the people running the workbook; the
API reads them only to find a free column for [markedBy](#markedby), and writes one
cell there per mark.

Both generators (`dual-meet-generator.gs`, `standard-tournament-generator.gs`) create this tab
empty when they build a workbook: the header, a frozen first row, and the
column J list below. `verify-dual-meet-generator.mjs` and
`verify-standard-tournament-generator.mjs` import `HEADERS` from `attendanceTab.mjs`, so
the generated header cannot drift from the one the API requires. A tab of that
name already in the workbook is left alone. The API creates the tab itself only
when it is missing (a workbook not built by a generator, such as PickleDrive's).

When the API creates the tab it also writes a
**Not yet in** list in column J (`J1` the header, `J2` a `FILTER` formula listing
everyone neither present nor withdrawn). That write is fixed in
`SheetsClient.createAttendanceTab` and entered as a formula; `updateValues`
refuses column J. A tab that already exists is not changed.

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

### markedBy

A mark that changes something also writes who did it: `In · Ana (Gate A) · 14:32` or
`Out · Control Center · 09:05` (`formatMarkedBy`, Manila time, 24-hour). The label is
`actorLabel(actor)`: the desk link's `issuedTo` and `note`, or `Control Center` for an
operator ([auth](auth.md#who-a-scoped-token-was-issued-to)). It is one `updateValues` call with
`E:F`, so the mark and its label land together; a mark that changes nothing writes nothing.

`mark` reads `ATTENDANCE!A1:Z`, not `A1:G`, and gives `parseTab` only the first seven cells of
each row. `parseTab` is strict on A–G (a header that is not exactly the seven `HEADERS`
throws `AttendanceLayoutError`), and a person's own columns beyond G don't trip it.

`markedByColumn(values)` picks the column from that read. H onward belongs to the people running the workbook, and J
holds the **Not yet in** formula, so it never takes a column that is in use:

1. a column in H–Z (J excepted) whose row-1 header is exactly `markedBy` wins, and nothing more
   is written for the header;
2. otherwise the first column in H, I, K…Z whose row 1 **and every data row** are blank is claimed,
   and the same call writes the `markedBy` header into it;
3. otherwise `null`: the mark is written without it and a warning is logged.

`HEADERS` is the seven columns A–G, and neither generator writes a `markedBy` header, so a
new tab has no `markedBy` until the first mark claims a column.

## Routes

Mounted at `/v3` (`src/attendance/routes.mjs`), which every page calls:

| Route | Auth |
| --- | --- |
| `GET /v3/days/{day}/facilities/{facility}/people` | operator token, or a desk token |
| `PUT /v3/days/{day}/facilities/{facility}/people/{personKey}/attendance`, body `{ present }` | operator token, or a desk token |
| `POST /v3/days/{day}/attendance/desk-links` | operator token only |
| `POST /v3/days/{day}/attendance/reconciliations`, optional body `{ facility }` | operator token only |

The three writes each have a frozen `/v1` twin (`PUT /v1/days/:day/facilities/:facility/attendance/:key`,
`…/desk-links`, `…/reconciliations?facility=`) that copies of the pages from
before 3.0.0 call; the list has none, because nothing before it read through the API; it answers the same, with the plain `{ error, code }` body
instead of problem details and `400` instead of `404` for an unknown day or
facility ([API](api.md)).

CORS allows `PUT`. Errors carry `error` in words (inside a problem details body
on `/v3`), never raw Google JSON.

## Reading the list

`GET /v3/days/{day}/facilities/{facility}/people` answers one facility's people
in sheet order as a JSON array (no envelope) with `Cache-Control: no-store`. Each
person is `{ key, player, teams, categories, present, timeIn, withdrawn, shirt }`.
It makes `mark`'s checks in `mark`'s order: attendance off for the event is a
`404`, a desk token for another day, or while the event is `"console"`, is a
`403`, an unknown day or facility is a `404`. A workbook with no `ATTENDANCE` tab
answers `[]`, which the page shows as "no roster yet"; a wrong header row is a
`409 attendance_layout`, as a mark gets.

`AttendanceService.listPeople` reads `ATTENDANCE!A1:Z` with `FORMATTED_VALUE`, so
every cell is the text the sheet displays (a checkbox reads `TRUE`, and a
hand-typed time stays text), and hands it to `peopleFromTab`
(`src/attendance/domain/attendanceTab.mjs`). That function gives the `Person`
shape the site's `parseAttendanceCsv` builds, by the same rules, so the same sheet
gives the same list:

- the header check is `parseTab`'s, on the first seven columns, so columns past G
  do not trip it; the first row of a key wins, and a blank key is skipped;
- `player` is the trimmed name, or the key when it is blank; `teams` and
  `categories` split on commas, trimmed, with empty parts dropped;
- `present` and `withdrawn` are true only for the text `TRUE`;
- `shirt` is read from the column of the first name in `SHIRT_HEADERS`
  (`TShirt Size`, `T-Shirt Size`, `Shirt Size`, `tshirtSize`, `shirtSize`, in that
  order, the site's list) that some header equals, and is empty when none does.

**The read is cached for 10 s per workbook** (`peopleCacheMs`), shared by every
caller, so the number of desks open does not multiply reads from Google. Calls made
together share one read, a failed read is not cached, and any write to the
workbook's `ATTENDANCE` tab (a mark that changed something, a roster update that
created the tab or wrote rows) drops the entry. A person who marks sees their
change at once; another desk is told by a [nudge](#push) and sees it within a second or
two. The cache is per instance and in memory, so another Cloud Run instance can hold a
list up to 10 s old; that is why a page reads once more 11 s after a read of everything.

The reads a workbook sees at an event's busiest moment are the sync's (one
`batchGet` each, about 5 a minute across two events on 3 October 2026), the list's (a
page reads it on opening and once more 11 s later, then once per nudge, at most one per
2 s per facility, and when a phone wakes), one per mark, and two per score save. All of
them are the service account's, so the Sheets API's *Read requests per minute per
user* quota is the limit that counts
([code and access protection](../specs/in-progress/code-and-access-protection-spec.md)).

### Push

Other pages learn that a list changed from a **nudge**, a message that says only *this
facility's list changed*:
`{"type":"attendance","facility":"<name>"}`. It carries no person, key, name, state, time
or count, so it can travel on a public socket; a facility's name is already public in
`events.json`. Spec: [push for attendance](../specs/implemented/attendance-push-spec.md).

- **The Worker** (`live-worker/`) gives attendance its own object, `attendance:<event>/<day>`,
  of the same `DayChannel` class as the snapshot channel, so a public viewer never receives a
  nudge and a fault here cannot reach the public feed. It stores nothing.
  `GET /attendance/<event>/<day>` is the socket (a WebSocket upgrade, the same `Origin` rule as
  `/live`, no secret, nothing sent on connect); `POST /nudge/<event>/<day>` with `{ facility }`
  and `X-Publish-Secret` broadcasts the message to that object's sockets and answers
  `{ clients }`. See [sync pipeline](sync-pipeline.md#live-push-delivery).
- **The API** (`AttendanceService`, through the `AttendanceSignals` port that `LivePublisher`
  implements) nudges after every write that changes the tab: a mark that changed someone, a
  roster update that created the tab, and a roster update that wrote rows. One nudge per write,
  after the write and the cache drop. It is awaited before the response, because Cloud Run
  throttles CPU once a response is sent, with a 2-second timeout. A failure or timeout logs one
  `live nudge failed` warning and never changes the answer; after one, the instance sends none
  for 60 s. It nudges whenever `LIVE_PUSH_URL` and `LIVE_PUSH_SECRET` are set, whatever the
  **Sync method** switch says: that switch is about how snapshots are delivered.
- **The page** (`lib/v1/data/attendance-signal.js`, wired by the desk page and Control Center,
  never by the view) holds one socket to the day's attendance object, with a ping and the same
  reconnect backoff as the snapshot channel. A nudge for a facility the page shows re-reads that
  facility only, at most once every 2 s, and once more 11 s after the last nudge. Nudges for a
  facility the page does not show, or received while the page is hidden, are ignored.

**Nothing polls.** A page reads the list when it opens, when it comes back into view (a phone
that slept missed nudges), when its socket opens again after a drop, on a nudge, and on
**Refresh**. Each of those reads except a nudge's is followed by one more read 11 s later (the
cache of another instance may be 10 s old, and a page that was not listening missed the nudges
that would have corrected it); a newer such read restarts the 11 s, and the follow-up itself
schedules nothing. When the socket is not open, or the page has no Worker address, the view shows
*Live updates are off. Press Refresh to see marks from other devices.* The finished events'
inline desk pages keep their own polling.

## Reading the workbook

The API reads every facility workbook as its own service account: `SheetsReadAuth`
(`src/clients/`) gives each read the account's token, falls back to
`GOOGLE_SHEETS_API_KEY` when no token can be had, and on a `403` to the token
retries once with the key, so a finished event's link-shared workbook that the
account was never given still reads. A read still refused after that is a
`SheetsAccessError`, which answers like any upstream failure (`502`,
`upstream_failure`) with "The API can't read this workbook. Share it with
`<service account>` as Editor." So a facility workbook is **Restricted** and shared
with the account as Editor, never anyone-with-the-link. The same reader serves
[the sync](sync-pipeline.md) and [score entry](sync-pipeline.md#score-entry-writes).

## Writing to the workbook

`SheetsClient` reads through `SheetsReadAuth` (above) and writes with the service
account's token (`GoogleAccessToken`, from the metadata server; `GOOGLE_ACCESS_TOKEN`
overrides it for local development). **Every attendance write range must be
inside `ATTENDANCE!A:G`, or be one single cell in `H`–`Z` other than `J`** (the
`markedBy` cell or its header; `updateValues`); anything else throws before a request
is made, so a bug here cannot overwrite scores or formulas, or the Not yet in list. The client’s only other
writes are score entry’s, which have their own allowlist (one match’s two `SCHEDULE`
score cells and the note on them; see [sync pipeline](sync-pipeline.md#score-entry-writes)). There is no append call. Google
`429` is retried once after a second, then answers 503.

## Desk tokens

See [auth](auth.md). A desk token carries a day and expires at the end of that
day in Manila time; it is accepted only while the event is `"desks"`.

## The client

One module of the [site engine](site-engine.md), `lib/v1/views/attendance-view.js`,
imported by `tools/control-center.html` and by the desk page
(`lib/v1/apps/attendance-desk.js`, which the attendance template's shell
calls). Its stylesheet is `lib/v1/css/attendance.css`, which a page links after
its own styles. `createAttendanceView` renders the list, the category bar and
search, and marks optimistically; it reads on the events listed under [Push](#push)
(`changed(facility)`, `setLive(open)`, and its own opening, visibility and **Refresh**) and
sets an interval only when handed a positive `pollMs`, which the apps no longer pass. The
finished events' `attendance.html` files (Pickle for Sight, Piggleball,
PickleDrive) keep their own copy of the block they were made with and do not
use the engine.

Control Center loads every facility of the day so counts are complete; the
desk page loads only the venue being shown. With `?fixture=<name>` on
localhost both read `_fixtures/` CSVs and mark in memory; `?attfail` makes the
first mark fail.

## Tests

`npm test` in `sage-tools-api` (`node:test`): unit tests for each module
under `test/unit/attendance/` and an HTTP test of the routes. The site has
fixtures and manual checks (spec section 7.3).

---

## Pickle for Sight's attendance

The notes below describe Pickle for Sight's `attendance.gs` and
`events/pickle-for-sight-2026/attendance.html`. The script runs in that event's
workbooks and is not kept in `sage-tools-api`'s current tree: its source, with its
harness `verify-attendance.mjs`, is in `apps-script/` at commit `198f02c`.

Pickle for Sight's check-in page writes to Google Sheets through a web app in
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
  name, present, timeIn }] }] }`. The page does not call this; it is a
  diagnostic endpoint, useful to open directly in a browser.
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
the real roster from `event-data`; §11 of the spec checks the CSV read path
against the live PCPH Main workbook.

---
**Features:** [event attendance](../features/event-attendance.md)
