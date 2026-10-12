# Spec — code and access protection

> **Status: not started.** A to-do list, not a design: each item says what to
> change and where, and leaves the details to whoever builds it. Written
> 2026-10-11; §9 and §10 added 2026-10-12. Everything here is free or close
> to it. Sections 2–8 are ordered by value; §9 is a larger piece of work with
> decisions of its own, and §10 is two small fixes found in the same review.
> Each section is independent and can be done on its own.
>
> This page is public. It states the rules to follow, not which of them are
> broken today: check each one when you pick it up.

## 1. What is protected, and what cannot be

The code worth protecting runs on the server: `sage-tools-api` (sync, auth,
score and attendance writes, the scoresheet pipeline and the Apps Script) is a
private repo. Keep it that way.

Anything a browser runs (`lib/v1/`, the tools, Control Center) is readable by
whoever opens the page, whatever repo it comes from. So:

- **No obfuscation or minification of the site.** It is reversed with free
  tools, costs a build step the site deliberately does not have, breaks the
  comparison harness and the `lib/v1` compatibility rule, slows pages on venue
  Wi-Fi and makes event-day debugging harder.
- **The Bracket Generator's draw stays in the browser on purpose.** Its
  verifiable draw ([spec](../implemented/bracket-generator-verifiable-draw-spec.md))
  depends on anyone being able to re-run it.
- **The pages and the engine stay in the browser.** They render live from
  snapshots, and the rules they hold are tournament rules players should see.
- **Control Center stays as it is.** It holds no secret and enforces nothing
  itself; the API decides every action.

Protection is therefore about **access**: who can edit the workbooks, who can
push, what a leaked credential can do, and what an attacker can cost us.

## 2. Workbook sharing (Google Drive)

Workbook IDs are public: the masters' IDs are in the Tournament Calculator and
every facility workbook's is in `event-data/config/events.json`.

- **Every link-shared workbook is "Anyone with the link: Viewer", never
  Editor.** That covers both masters (SAGE Dual Meet Master, SAGE Standard
  Tournament Master) and every event's facility workbooks, finished events
  included. Staff who edit are added by name. The calculator's handoff still
  works: **Make a copy** needs only view access.
- **Why it matters:** an editor can change scores (the onEdit sync publishes
  them), change the bound Apps Script that every future workbook is copied
  from, and read Script Properties, where `sheets-sync.gs` keeps the sync
  secret.
- **If a workbook holding the secret was ever editor-shared:** run **SAGE →
  Replace shared secret** in it, and if exposure can't be ruled out rotate
  `SYNC_SHARED_SECRET` in Cloud Run (every syncing workbook then needs the new
  one, so never close to an event).
- **Add it to the dry-run checklist**
  (`_templates/dry-run-checklist-template.md`): check each facility workbook's
  sharing before the event.

A facility workbook cannot go further than Viewer, to **Restricted**, until
the readers that rely on link access change: that is §9.

## 3. Cap what Cloud Run can cost

- Add `--max-instances` (e.g. `3`) to the deploy command in the `Dockerfile`
  comment and to the Cloud Run build trigger, and update
  [Deployment](../../technical/deployment.md). With `--concurrency 4`, check the
  cap still covers an event's peak sync load.
- Set a Google Cloud **budget alert** (e.g. $5–10/month) on the project. It
  only warns; it does not stop spending.
- Check the Cloudflare account for `sage-live` is on the free plan with no
  card-backed overage.

## 4. Rate-limit sign-in

- `POST /v3/sessions` and its frozen twin `POST /auth/login` throttle failed
  attempts, e.g. 5 per IP per 15 minutes, answering `429` (problem details on
  `/v3`, `{ error, code }` on the legacy route) with `Retry-After`.
- In-memory is enough: no new service, no new dependency. Per-instance counts
  are acceptable with the instance cap above.
- `sage-tools-api` rules apply: failing test first, a `package.json` bump
  (minor), a Changelog entry with its **Tests:** line, and the
  [API](../../technical/api.md) and [Auth](../../technical/auth.md) pages
  updated.

## 5. GitHub settings

Org `sage-match-control`, all repos:

- **Require two-factor authentication** for members.
- **Secret scanning with push protection** and **Dependabot alerts** on.
- **A ruleset on `main` of `sage-tools-api`**: no force-push, no deletion. A
  push there deploys to production, and the pre-push hook is skippable with
  `--no-verify`. Optionally the same for the site and `event-data`.
- Review who has write access to each repo.

## 6. Least-privilege credentials

- `GITHUB_TOKEN` (Cloud Run) is a **fine-grained token**: repository
  `event-data` only, Contents read/write, nothing else, with an expiry date and
  a calendar reminder to renew it outside event weeks.
- The `sage-tools-api-runtime` service account is an Editor only on facility
  workbooks it writes (attendance and score entry), not on the masters.
- `GOOGLE_SHEETS_API_KEY` is restricted to the Sheets API in the Google Cloud
  console.

## 7. Where the docs live

`sage-docs` is public and published to GitHub Pages, so the architecture, auth
design and API are readable by anyone. GitHub Pages on a private repo needs a
paid plan. Owner's decision, one of:

1. **Leave it public** (no cost, no change).
2. **Make the repo private and host the site on Cloudflare Pages** (free from a
   private repo; the account already exists for `sage-live`). The site URL
   changes, so every link to `sage-match-control.github.io/sage-docs/` has to
   follow.
3. **Split it:** publish `features/` and `usage/`, keep `technical/` and
   `specs/` in a private repo.

## 8. Licence notice

Add a `LICENSE` file (all rights reserved, no permission to copy or reuse) to
each public repo: `sage-match-control.github.io`, `event-data` and, if it stays
public, `sage-docs`. Unlicensed code is already all rights reserved; the file
makes that explicit.

## 9. Facility workbooks closed to link access

**Goal:** a facility workbook's General access is **Restricted**. People who
edit it are added by name, and the API's service account
(`sage-tools-api-runtime`) is added as well. Nobody else can open it, so the
`ATTENDANCE` tab (names, teams, shirt sizes, who is present) and every tab the
sync does not publish stay inside the workbook. What the sync publishes to
`event-data` stays public, as now.

### 9.1 Who reads a workbook by link

Four readers need link access. An API key, like no credential at all, can read
only a sheet anyone with the link can open, so all four change together:

| Reader | Where | How it reads |
| --- | --- | --- |
| The sync's default fetcher | `sage-tools-api` `src/clients/SheetsCsvFetcher.mjs` | Sheets API `values:batchGet` with `GOOGLE_SHEETS_API_KEY` |
| The sync's fallback, `{ method: "csv" }` | `src/clients/GvizCsvFetcher.mjs` | the gviz CSV export, no credential |
| Attendance and score-entry reads | `src/clients/SheetsClient.mjs` (`readValues`, `readScoreNotes`) | Sheets API with the API key. Writes already use the service account |
| The desk page and Control Center's Attendance view | the site's `lib/v1/views/attendance-view.js` (`attendanceCsvUrl`, `fetchFacility`) | the gviz CSV export of the `ATTENDANCE` tab, straight from the browser, every `ATTENDANCE_POLL_MS` (10 s) |

The Apps Script (`sheets-sync.gs` and the generators) runs inside the workbook
and is unaffected. So is the site's reading of `event-data`.

### 9.2 The server reads as the service account

- `SheetsCsvFetcher` and `SheetsClient`'s reads send the `GoogleAccessToken`
  bearer token in place of `?key=`. The token's scope (`spreadsheets`) already
  covers reading. `SheetsCsvFetcher` takes the token as a constructor argument,
  passed in by `src/app.mjs`.
- **A read the account cannot make** (`403` or `404`) answers with a message
  naming the account, as a failed write already does, so the operator knows
  to share the workbook with it.
- **Quota.** Every read now comes from one identity. Check the Sheets API's
  per-user read quota in the Google Cloud console against an event's peak:
  syncs (on 3 October two events made 936 syncs in 3 h 15 min, about 5 a
  minute), score-entry reads, and the desk reads in §9.4. Raising a quota is
  free.
- **D1: whether the API key stays.** Keeping it as a fallback lets a run
  without the metadata server (a local `npm start`) and a finished event's
  link-shared workbook still read. Removing it leaves one way to read. Either
  way `loadConfig.mjs`, `.env.example` and its test follow.
- `sage-tools-api` rules apply: failing tests first, a minor bump, a Changelog
  entry with its **Tests:** line, and the [Sync pipeline](../../technical/sync-pipeline.md)
  and [Event attendance](../../technical/event-attendance.md) pages updated.

### 9.3 The sync's gviz fallback

`{ method: "csv" }` exists for when the API key or its quota fails mid-event.
Without a credential the gviz export cannot read a Restricted workbook.

- **Spike first:** does the gviz export accept the service account's bearer
  token? If it does, `GvizCsvFetcher` sends it and the fallback keeps working.
- If it does not, the fallback works only on a link-shared workbook. The
  [Sync pipeline](../../technical/sync-pipeline.md) page and
  [Troubleshooting](../../usage/troubleshooting.md) say so, and the fallback
  matters less anyway once reads no longer depend on the API key.

### 9.4 The desk page reads through the API

- **A new route,** `GET /v3/days/{day}/facilities/{facility}/attendance` (its
  final name settled against the [API](../../technical/api.md) conventions).
  It is allowed for the operator token or a desk token for that day, the same
  as the mark it sits beside. There is no `/v1` twin: new routes are `/v3`
  only.
- **It answers the people list,** in the shape `parseAttendanceCsv` builds
  today (`key`, `player`, `teams`, `categories`, `present`, `timeIn`,
  `withdrawn`, `shirt`). It reads the tab through
  `src/attendance/domain/attendanceTab.mjs`, so the tab is parsed in one place.
  The shirt column's accepted spellings (`ATTENDANCE_SHIRT_HEADERS` in the
  site's `lib/v1/domain/attendance.js`) move with it.
- **A short per-facility cache** (a few seconds) is shared by every caller, so
  the number of desks open does not multiply reads from Google. A mark (the
  `PUT`) clears that facility's entry, so the marker sees the change at once.
- **The site:** `attendance-view.js` reads through `data/api.js` with the
  token it already holds (`getToken`), for both the desk page and Control
  Center's console mode. This is a compatible `lib/v1` change:
  `attendanceCsvUrl` stays exported (an export is never removed), and
  `?fixture=` reads its files as before. It ships in a push after the API
  route is deployed.
- **The finished events' desk pages** (Pickle for Sight, Piggleball,
  PickleDrive) are frozen and keep reading gviz. Their workbooks keep link
  access (§9.5).

### 9.5 Sharing, and the runbook

- **The two masters stay "Anyone with the link: Viewer".** The calculator's
  handoff opens their `/copy` URL, which needs view access, and they hold no
  event data.
- **Facility workbooks of every event built after §9.2–§9.4 ship are
  Restricted** and shared with the service account: as Viewer when the event
  only syncs, as Editor when it has an `attendance` or `scoreEntry` setting
  (today's step 13 of `_templates/CLAUDE.md`).
- **Finished events' workbooks keep their sharing.** The pages under
  `events/archives/` (Kingcourts Cup, JR at 45, PPA x Club 2600, BOC Friendly
  Match) and the three finished desk pages read their workbooks straight from
  Google.
- **Never change a live event's sharing during its event week.**
- **Pages that change with it:** `_templates/CLAUDE.md` (step 13 covers every
  event, and the General access line), the dry-run checklist template, §2 of
  this page, [Build the workbooks](../../usage/build-the-workbooks.md),
  [Connect the workbooks](../../usage/connect-the-workbooks.md),
  [Troubleshooting](../../usage/troubleshooting.md) (its first check becomes
  "is the service account still on the workbook?") and
  [Adding a new event](../../technical/adding-a-new-event.md).

### 9.6 Order

1. The server reads with the token (§9.2) and the gviz spike (§9.3). This works
   on link-shared workbooks too, so it can ship in any week without an event.
2. The attendance route and its cache (§9.4, API half).
3. `lib/v1`'s desk view reads through the route (§9.4, site half).
4. The runbook and docs (§9.5). The next event's workbooks are Restricted
   from the start.

## 10. Played and series rules: two small fixes

These are not about access. They were found in the same review and touch the
one rule still kept by hand in two places: `sage-tools-api`'s
`src/sync/domain/facilityCompletion.mjs` (the published `completedAt`) and the
site's `lib/v1/domain/` (`byes.js`, `series.js`, `matches.js`, `progress.js`),
which `_tests/unit/parity-server.test.mjs` keeps in step. Both fixes go in one
`sage-tools-api` patch release.

### 10.1 Stale comments

- `facilityCompletion.mjs`'s header says it mirrors `control-center.html`
  (`parseCSV`, `rowsToMatches`, `sideIsBye`, `seriesGameOf`/
  `unneededSeriesGames`, `computeFacilityProgress`). Those functions now live
  in `lib/v1/domain/`. The comment names the files and the parity test.
- `SheetsCsvFetcher.mjs`'s header says parsing stays client-side, so the
  service and the browser never hold two copies. `facilityCompletion.mjs`
  parses the matches CSV and `teamRoster.mjs` the Teams tab. The comment says
  what is parsed where.

### 10.2 Reading a score's value

Both sides count a cell as a score when it is not blank and is a number
(`!isNaN`). They read its value differently:

| | Reads with | `11.5` | `1e1` |
| --- | --- | --- | --- |
| The site (`rowsToMatches` in `matches.js`) | `parseInt(raw, 10)` | 11 | 1 |
| The server (`unneededSeriesGames` in `facilityCompletion.mjs`) | `Number(raw)` | 11.5 | 10 |

Only the series-final walk compares values on the server, so that is the one
place they can disagree. A game typed `11.5`–`11` is a win on the server and a
tie on the site. The server can then decide the series, call the later games
unneeded and stamp `completedAt`, while the facility card still counts those
games as left. Score entry writes whole numbers only (`scoreRequest.mjs`), but
a score typed into the sheet is not checked.

- **The fix:** the server reads the value with `parseInt(raw, 10)`, the site's
  rule, since that is the value every page shows.
- **Tests:** a case in `test/unit/sync/domain/facilityCompletion.test.mjs`
  that fails before the fix, and the same series case added to `CASES` in the
  site's `_tests/unit/parity-server.test.mjs`.
- A patch bump and a Changelog entry with its **Tests:** line. §10.1's comments
  ride along in the same release.

## 11. Deferred

- **Tournament Calculator logic on the server.** Its planning core (tiered
  byes, wildcard fill, slot-aware finish times) is the one piece of front-end
  logic with real originality. It could move behind an endpoint on the
  `sage-live` Worker (free, no cold start), at the cost of offline use and a
  network call on every change. Not worth it unless it is actually copied.
- **Private site repo.** Moving the site to Cloudflare Pages would let
  `sage-match-control.github.io` go private, but only hides its history,
  templates and tests: the served code stays readable.
