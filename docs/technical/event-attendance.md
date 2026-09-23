# Event attendance

A staff check-in page that writes to Google Sheets — the only place a GitHub
Pages page writes into a scoring workbook. Two parts:

- `sage-tools-api/scripts/attendance.gs` — bound Apps Script, pasted into
  each of an event's **live** workbooks (never a master) beside
  `sheets-sync.gs` and the generator, and deployed there as a web app. Not
  part of the Cloud Run service: changing it is not a deploy and does not
  bump `package.json`.
- `events/<event>/attendance.html` — a self-contained static page, like the
  event's `schedule.html`. Pickle for Sight's is the first.

Spec: [event attendance](../specs/implemented/event-attendance-spec.md).

## The web app

Deployed per workbook with **Execute as: Me** and **Who has access: Anyone**,
so desk staff need no Google account. One `/exec` URL per venue; the page
lists them in its `VENUES` constant.

- `doGet` returns `{ ok, pairs: [{ teamCode, category, players: [{ slot,
  name, present, timeIn }] }] }`.
- `doPost` takes a JSON body `{ teamCode, slot, present }` and returns the
  saved player. The code must be a real pair row and the slot a named player.
- A failure comes back as `{ ok: false, error }`. `ContentService` cannot set
  an HTTP status, so every reply is a 200.

Editing the script changes nothing until **Deploy → Manage deployments →
edit → Version: New version**. An `/exec` URL keeps serving the version it
was given.

### Where the roster and the marks live

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

A stored mark counts only while its `name` still matches the roster, so a
re-draw that gives a code and slot to someone else doesn't inherit the mark.
`timeIn` is `yyyy-MM-dd HH:mm` in the spreadsheet's time zone, stored as
text; marking an already-present player keeps the first time.

### Concurrency

Every POST holds the script lock, so two phones marking at once cannot both
append a row for the same player. Script writes don't fire the installable
`onEdit`, so a mark never triggers a sync.

## The page

Reads with a plain `fetch(url)`. Writes with `fetch(url, { method: 'POST',
body: JSON.stringify(…) })` and **no headers**: a string body goes as
`text/plain`, which needs no CORS preflight, and Apps Script cannot answer
one. Both follow Apps Script's redirect to `script.googleusercontent.com`,
which is what makes the reply readable from the page.

A switch updates at once and locks until the save replies; a failed save
snaps back and says why. The page polls every 30 seconds while visible and
reloads when it becomes visible again. A player with a save in flight keeps
its local state through a reload, since the sheet may not have it yet.

## Verifying a change

    node scripts/verify-attendance.mjs

Runs the real `doGet`/`doPost` against `scripts/mock-apps-script.mjs`: roster
selection, marking and unmarking, every refusal, the re-draw rule, lock
release, and that `STANDINGSCSV` is never written. It also fails if a
top-level name in `attendance.gs` is declared by `sheets-sync.gs` or either
generator — they share one script project, where a repeated name silently
replaces the other file's.

The page has no test file. The spec's §5.4 serves it with `fetch` stubbed and
the real roster from `event-data`.

---
**Features:** [event attendance](../features/event-attendance.md)
