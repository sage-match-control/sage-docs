# Spec — Score entry from Control Center and scorer links

Two ways to enter a match's score without opening the workbook:

- **Operators**, signed in to Control Center, click a match in **Match
  Finder**, type the two scores, check a review step that names the winner,
  and save.
- **Scorer staff** get a **scorer link** from Mission Control. It opens the
  event's **scorer page**, which can enter scores for that day and nothing
  else. They never see Control Center. Mission Control has a switch that stops
  (and resumes) scorer links at any time.

Either way, `sage-tools-api` writes the scores into that match's two score
cells in the facility workbook's `SCHEDULE` tab (the cells a scorer types into
by hand), then publishes the day at once.

> **Status: in progress.** Built and tested against fakes (`sage-tools-api` 2.8.0,
> Control Center, the scorer template, the docs); the real-Google checks in §11.2
> (C1–C8) have not run, and no event has `"scoreEntry"` set yet. Written 2026-10-05 against
> `sage-tools-api` 2.7.1, `sage-match-control.github.io`'s
> `tools/control-center.html` and templates, and `event-data`'s
> `config/events.json` as of that date. **Every decision is settled (§1).**
> There are no open questions: do not stop to ask about anything §1 covers.
>
> Builds on [Attendance for every event](../implemented/multi-event-attendance-spec.md),
> which already writes to facility workbooks as a Google service account and
> issues **desk links**. Score entry reuses that account, its token and its
> Sheets client, and scorer links are built the same way desk links are.

---

## 0. Read this first

You are implementing this with no other context. Everything you need is in
this page and the files it names. Read §0 to §3 in full before writing code,
then do §4 to §8 in order. §11 is what to report when you finish.

### 0.1 The system in one paragraph

SAGE runs pickleball tournaments. Each **event** (e.g. `piggleball-2026`) has
one or more **days** (`piggleball-day1`), and each day has one or more
**facilities** (venues). Every facility-day has its own **Google Sheets
workbook**. Scorers type each match's two scores into the workbook's
**`SCHEDULE`** tab; the workbook's formulas read them back from there to build
standings and two readout tabs, `CSV` (one row per match) and `STANDINGSCSV`.
An Apps Script onEdit trigger in each workbook calls `sage-tools-api`
(Node/Express on Google Cloud Run) on every edit; the API reads `CSV` and
`STANDINGSCSV` and publishes a JSON **snapshot** of the day. Public pages and
**Control Center** (the operators' console, one HTML file) display that
snapshot, receiving each new one over a WebSocket (the **live channel**)
within a second or two, or by polling GitHub Pages every 10 s. The registry
of events, days, facilities and their sheet IDs is `config/events.json` in
the `event-data` repo; the API fetches it at runtime, caches it for about a
minute, and can write it (Control Center's go-live and sync-method switches
do). Today Control Center only reads scores. This spec lets operators and
scorer staff write one match's score.

### 0.2 Where things are

`D:\Personal\SAGE` is a plain folder (not a repo) holding four git repos:

| Path | What you change there |
|---|---|
| `sage-tools-api/` | §4: a new `src/scores/` module, three new routes, scorer tokens in `AuthService`, two new `SheetsClient` methods, config validation and a config write, tests, version, Changelog |
| `sage-match-control.github.io/` | §5: `tools/control-center.html` (a shared `SCORE CLIENT` block, clickable matches, Mission Control's **Scorer links** section). §6: a new template, `_templates/scorer/scorer.html`. `_fixtures/config.json`. `_templates/CLAUDE.md` |
| `event-data/` | §8: `config/README.md` only. **Never edit `config/events.json`** |
| `sage-docs/` | §8: feature and technical pages |
| `D:\Personal\SAGE\CLAUDE.md` | §8: a few edits (not in any repo) |

### 0.3 Hard rules

1. **Leave every change uncommitted.** Do not `git add`, `git commit`,
   `git push`, create branches, switch branches or stash. The owner reviews the
   working trees and commits. `sage-docs` already holds this spec's own
   uncommitted files; leave them as they are.
2. **Never write to a real workbook, never call the production API**
   (`https://sage-tools-api-811926984834.us-central1.run.app`) and never touch
   the real `events.json` from a script or test. Tests use fakes. Real-Google
   checks are the owner's (§11.2).
3. **Never weaken or delete a test to make a change pass.** The existing
   assertions that change on purpose are listed in §4.13; nothing else
   existing should.
4. `sage-tools-api`: ESM `.mjs`, classes with constructor injection, **no new
   dependency** of any kind, no TypeScript, no build step. Match the
   surrounding style: 4-space indent, double quotes, comments that say why.
5. Site pages: one inline `<style>`, one inline classic `<script>`, no new
   external dependency (the QR library Control Center already loads is
   reused), 2-space indent, single quotes, match the surrounding style. Find
   functions **by name** (`grep -n "function name("`), not by line number:
   line numbers in this spec are approximate.
6. Working copies use **CRLF** line endings. Keep them. After editing a file,
   `file <path>` must still say "with CRLF line terminators" (where it did
   before). New files: CRLF too.
7. Cite this spec from code comments as
   `sage-docs/docs/specs/.../control-center-score-entry-spec.md`, with
   literally `...`.
8. Documentation is written in the present tense: what the system does, not
   what changed.
9. If something in the code contradicts this spec in a way that blocks you,
   stop and report it (§11). Don't work around it.

### 0.4 Facts checked against the code (2026-10-05)

**The workbook**

| Fact | Where |
|---|---|
| **`SCHEDULE`'s geometry is the same in dual-meet, standard and team workbooks.** Court blocks repeat every 8 columns. Data starts at row 6; one time slot is two rows. In 1-based column numbers, match numbers are in columns **6 + 8k** (`F`, `N`, `V`, `AD`, …), team 1 code at **−1**, team 2 code at **+1**, team 1 score at **+3**, team 2 score at **+4** | `sage-docs/docs/specs/implemented/dual-meet-schedule-generator-spec.md` §2.1–2.3; `…/implemented/standard-tournament-master-spec.md` §10.2.1; the team workbook's `STACKBLOCKS(6,8,6)` etc. in `…/not-started/team-workbook-stack-cache-spec.md` §3.1 |
| Codes and scores are merged vertically across the slot's two rows; the match number is on the slot's first row only. Writing a merged cell through the API means writing its top-left cell, on the match number's row | same |
| Grid widths differ (`8·courts + 2`, `8·courts + 3`, 83 columns) only by trailing separator columns. The 6 + 8k rule holds for all | same |
| Scores are plain numbers. Team workbooks test "unplayed" with `ISBLANK`, so a cleared score must be a truly empty cell, not `""` | `team-workbook-stack-cache-spec.md` §2.2 |
| Team workbooks' playoff team codes in `SCHEDULE` are formulas. Score cells are not | `team-workbook-stack-cache-spec.md` §2.4 |
| **Sheets API writes do not fire installable onEdit triggers** ("Script executions and API requests do not cause triggers to run", Google's Apps Script trigger docs). So the API must publish after it writes; nothing else will | — |
| The service account is `sage-tools-api-runtime@sage-tools-api.iam.gserviceaccount.com`; it must be an **Editor** on a workbook to write to it | `_templates/CLAUDE.md` step 12 |

**`sage-tools-api`**

| Fact | Where |
|---|---|
| `SheetsClient` reads with the API key (`readValues(sheetId, range, { render })`, `null` when the tab doesn't exist) and writes as the service account. `updateValues` refuses any range outside `ATTENDANCE!A:G` before sending | `src/attendance/SheetsClient.mjs` |
| `SyncService.syncDay(day, { facilityName, method, editAt })` reads the facility, publishes, and returns `{ facilitiesSynced, facilitiesFailed, … }`. `editAt` is **epoch milliseconds**. It throws on config errors and when nothing could be published. Cloud Run throttles CPU once a response is sent, so publishing must finish before responding | `src/sync/SyncService.mjs` |
| `config.getDay(day)` → `{ day, event, label, date, facilities: [{ name, sheetId }], isLive }`, throws `UnknownSyncDayError` (400). `config.getEvent(event)` → `{ event, type, attendance }`, throws `UnknownEventError` (404) | `src/sync/SyncConfigSnapshot.mjs` |
| `SyncConfigStore.setLivePush(enabled)` is the pattern for writing `events.json`: fetch with sha, `validate`, change, `publisher.publish(path, raw, message, sha)`, retry on a 409 up to `COMMIT_ATTEMPTS`, then clear `this.cached`/`this.cachedAt` | `src/sync/SyncConfigStore.mjs` |
| `AuthService`: `login` issues operator tokens; `verify(token)` accepts only unscoped ones. `issueDeskToken({ day, expiresAt })` / `verifyDeskToken(token)` make and check desk tokens: payload `{ exp, scope: "attendance-desk", day }`, signed with a key derived from `AUTH_TOKEN_SECRET` and the scope (`#signDesk`). A desk token never verifies as an operator token, and vice versa | `src/auth/AuthService.mjs` |
| `AttendanceService.issueDeskLink(day)` checks the event's setting, computes expiry, calls `issueDeskToken`, returns `{ token, expiresAt, day }`; errors are `ValidationError`s | `src/attendance/AttendanceService.mjs` |
| An unknown facility is a 400 `ValidationError` (`unknownFacility` in `AttendanceService.mjs`) | same |
| Route errors are sent as `res.status(err.statusCode ?? 500).json({ error: err.message })`. Auth checks are small local functions in each routes file (`bearerToken`, `requireOperator`, `requireOperatorOrDesk`) | `src/attendance/routes.mjs` |
| CORS already allows `PUT`. JSON bodies are parsed. Attendance's routes are mounted at `/v1` | `src/server/Server.mjs` |
| Two guard tests: every `src/**/*.mjs` needs a mirrored `test/unit/<path>.test.mjs` (or a `test/helpers/coveredElsewhere.mjs` entry); every route needs a `test/helpers/routeManifest.mjs` line | `test/unit/guards/` |
| `test/unit/helpers/routes.test.mjs` asserts **12** registered routes; `test/unit/docs/openapiSpec.test.mjs` asserts **11** documented paths and has a `.snapshot` | those files |

**Control Center (`tools/control-center.html`)**

| Fact | Where |
|---|---|
| A match is `{ num, time, court, liveCourt, matchUp, t1, t1p1, t1p2, t2, t2p1, t2p2, t1Score, t2Score, played, facility }` in the global `MATCHES`. A player name equal to its code (or blank) becomes `'TBD'`. Scores are integers or `null`. Match numbers are unique across a day's facilities | `rowsToMatches`, `loadLiveData` |
| Match Finder draws standard/dual-meet matches with `ticketHTML(m, teamCode, isNext)`, and team-event matches as lines (`teamMatchupRowHTML`) inside `teamMatchupCardHTML(mu, opts)` cards, in `#results` (`resultsEl`), fully redrawn on every poll and push (`refreshFinder()`) | same |
| `teamMatchupCardHTML` is called from Match Finder by `teamResultHTML`, `teamPlayerResultHTML`, `allMatchupsHTML` and `renderMatchByNumber`, and from **Standings** by two calls in the bracket/playoff renderers | same |
| Helpers: `currentAuthToken()`, `EVENTS_REGISTRY` (Map of raw `events.json` entries, archived events excluded), `CURRENT_EVENT_KEY`, `CURRENT_TYPE`, `DAYS` (each `{ key, label, date, … }`), `currentDayIndex`, `FACILITIES`, `CLOUD_RUN_BASE_URL`, `FIXTURE` (localhost `?fixture=` name or `null`), `showToast(kind, text, key)`, `friendlyApiMessage(raw)`, `unreachableMessage(err)`, `escapeHtml`, `matchByeSide(m)`, `unneededSeriesGames(matches)` (a `Set` of match objects), `roundLabel`, `parseCode`, `nonByeCode(m)`, `divisionEventLabel`, `pairLabel(code, short)`, `sideOf(code)`, `teamNameOf(side)`, `rebuildTeamData()`, `refreshFinder()`, `renderAuthStatus()`, `renderOrganizerStatus()`, `selectEvent()`, `selectDay()`, `addResultClose(box)`, `revealResultBox(box)`, `resultEpoch` | same |
| **Desk-link UI to mirror:** `attConsoleIssueDeskLink` (POSTs, then renders), `attConsoleRenderDeskLink(day, issued)` (a result box with a read-only link input, Copy, Share when `navigator.share` exists, Show QR, "Valid until …"), `attConsoleShowQr(url, holder, btn)` and `attConsoleLoadQr()` (loads `qrcode-generator` from cdnjs with an SRI hash, draws to a canvas) | "Desk link" section near the end |
| **Switch UI to mirror:** Mission Control's "Sync method": `<label class="search-label">`, a `.golive-current` state line, `.golive-buttons` with `.golive-btn` (`.golive-btn-danger` for the stopping side, `.active` on the current one), an `.organizer-hint`, `.organizer-divider` between sections. Rendered by `renderSyncMethod()` | markup in `#organizerResults`; `renderSyncMethod` |
| CSS tokens on `:root`: `--navy`, `--navy-deep`, `--green`, `--green-dark`, `--paper`, `--paper-dim`, `--line`, `--ink`, `--ink-soft`, `--white`, `--radius`, `--card-shadow`. No dark mode. `.toast.warn` `#FFF4E0`/`#7A4B00`, `.toast.error` `#FDEDEF`/`#9A2A3A` | `<style>` |
| Byte-identical shared blocks are marked `// ==== <NAME> — identical in … ====` … `// ==== END <NAME> ====` (see `ATTENDANCE CLIENT`, `LIVE CHANNEL`) | same |
| Control Center has no modal today | — |

**Event pages and templates**

| Fact | Where |
|---|---|
| The attendance desk page is a template, `_templates/attendance/attendance.html`, copied per event to `events/<key>/attendance.html` with two tokens replaced, `{{EVENT_KEY}}` and `{{EVENT_TITLE}}`. Its link is `…/events/<key>/attendance?desk=<token>`. On load it moves the token from the URL into `localStorage` and strips it from the address bar (`deskLoadToken`), decodes the payload for display only (`deskDecode`), reads `events.json` and checks the event's setting, then starts | `_templates/attendance/attendance.html`, "THE DESK PAGE" section |
| Event pages wire the `LIVE CHANNEL` block as `createLiveChannel({ enabled: !!LIVE_BASE_URL && !FIXTURE, onSnapshot: () => loadLiveData() })`, `fetchDaySnapshot(dayKey)` (pushed snapshot first, else GitHub Pages, then `liveChannel.newer`), `liveChannel.follow(EVENT_KEY, dayKey)`, and pause/resume with polling on `visibilitychange` | `_templates/standard-tournament-template/index.html`: `createLiveChannel(`, `fetchDaySnapshot`, `liveChannel.follow`, `liveChannel.pause` |
| `.gitignore` ignores `**/*beta*`: a `scorer-beta.html` is never committed | `sage-match-control.github.io/.gitignore` |
| Fixtures: `_fixtures/config.json` registers `pickledrive-anniversary-2026` (team; pages exist in `events/`) and `attendance-demo-2026` (standard, two facilities; no `events/` folder). Snapshots in `_fixtures/<event>/<name>.json` (`pre`, `edge`, `finished`, `qf-pre` for PickleDrive; `pre` for the demo) | `_fixtures/` |

---

## 1. Decisions (settled with the owner, 2026-10-05)

| # | Decision |
|---|---|
| D1 | The API writes **only** that match's two score cells in `SCHEDULE`. Never a code, a match number, a formula, or any other tab |
| D2 | **Per-event setting** `"scoreEntry"` in `events.json`: absent = off (nothing clickable, the API refuses with 403); `"console"` = operators only; `"links"` = operators **and** scorer links. The owner turns it on by hand; you never edit `events.json` |
| D3 | **Operators** use the bearer token from `POST /auth/login`. **Scorer staff** use a scorer token. Nothing accepts the sync secret |
| D4 | **The API publishes right after it writes, in the same request**, via `syncService.syncDay` for that facility. A publish failure doesn't undo the write; the 200 response reports it |
| D5 | **Optimistic check:** the client sends the match as it showed it (`expected`). If the sheet's codes differ, or its scores differ from both `expected` and the new scores, nothing is written: 409 with the sheet's current values |
| D6 | **Ties are allowed**, with a warning. Every "unusual score" check is a warning, never a refusal |
| D7 | **Corrections and clearing are allowed**, for operators and scorers alike. Clearing empties both cells (truly empty, via `values:batchClear`) |
| D8 | In Control Center, matches are **clickable only in Match Finder**: every `ticketHTML` ticket, and every match line of a team matchup card drawn by Match Finder. Not Live Matches, not Standings, not Awards |
| D9 | **One dialog, two steps:** Enter → Review → Save. Review names the winner in the largest text (it guards against swapped scores). No `window.confirm`. The same dialog, in a byte-identical `SCORE CLIENT` block, is used by Control Center and the scorer page |
| D10 | **Conflicts offer both:** "Keep the sheet's score" and "Replace with yours" (unless the codes changed, §5.6) |
| D11 | Warnings assume **games to 11**: none reached 11, won by 1, over 21. No config for it |
| D12 | Standard and dual-meet matches with undecided players (`TBD`) open **read-only**. Team-event `TBD` means "lineup not set": allowed, with a warning |
| D13 | `SheetsClient` stays in `src/attendance/` (not moved). The score module imports it from there |
| D14 | **A scorer link covers one day, every facility of that day.** The scorer page has a facility picker, remembered per device and always visible, so a scorer can switch venue at any time without a new link |
| D15 | **A scorer link is valid for 24 hours from when it's issued** (not until midnight). It can't be issued for a day that ended (§4.8) |
| D16 | **A scorer can do everything an operator can** in the dialog: enter, correct, clear, and replace on a conflict. A scorer token can do nothing else: no other route accepts it |
| D17 | **Mission Control has a switch** under a **Scorer links** heading: **Accepting** (`"links"`) / **Stopped** (`"console"`). It writes the event's `scoreEntry` in `events.json` through the API, like the sync-method switch, and takes effect within about a minute. Stopped refuses every scorer save (403) and every new scorer link; operators can still enter scores; links issued earlier work again if switched back to Accepting before they expire. The switch only moves between those two values: it never turns score entry on or off |
| D18 | **The scorer page is per event**, from a new template `_templates/scorer/scorer.html`, copied to `events/<key>/scorer.html` for an event with `"scoreEntry": "links"`. Link: `https://sage-match-control.github.io/events/<key>/scorer?scorer=<token>` |

---

## 2. Overview

```
Control Center, Match Finder (signed in)            events/<key>/scorer?scorer=<token>  (scorer staff)
        \                                                 /
         \  the same SCORE CLIENT dialog: Enter -> Review -> Save
          \                                             /
           PUT /v1/days/{day}/facilities/{facility}/matches/{matchNumber}/score
           Authorization: Bearer <operator token | scorer token>
           { team1Score, team2Score, expected: { teamCode1, teamCode2, team1Score, team2Score } }
                                   |
sage-tools-api  ScoreService.submit()
   1. config: day, facility, event's scoreEntry; a scorer token needs "links" and its own day
   2. read SCHEDULE (API key)                        -> find the match (§3)
   3. read its two score cells with render FORMULA   -> refuse a formula or text
   4. compare with `expected`                        -> 409, or unchanged, or go on
   5. write both (values:batchUpdate RAW) or clear both (values:batchClear), as the service account
   6. syncService.syncDay(day, { facilityName, editAt })  -> pages update within seconds
   7. log one audit line; respond 200
                                   |
Facility workbook  SCHEDULE!<team1Score><row>:<team2Score><row>

Mission Control, "Scorer links" (signed in)
   Accepting / Stopped   -> PUT  /v1/events/{event}/score-entry   { mode: "links" | "console" }  -> events.json
   Issue scorer link     -> POST /v1/days/{day}/scores/scorer-links  -> { token, expiresAt, day } -> link, Copy, Share, QR
```

Unchanged: `sheets-sync.gs` and typing in the sheet (both paths write the
same cells), the generators, the snapshot format, the live Worker, attendance,
every existing route.

---

## 3. Finding a match in `SCHEDULE`

**Worked example.** A standard workbook, match 42 in court block 4, slot 5:

```
block 4's match-number column = 6 + 8·3 = 30 (AD)
slot 5 = rows 14–15 -> the match number is in AD14
teamCode1 AC14 (29), teamCode2 AE14 (31), team1Score AG14 (33), team2Score AH14 (34)
scoreRange = "SCHEDULE!AG14:AH14"
```

`isTeam1ScoreColumn(33)` is true because `(33 − 9) % 8 === 0`.

Rules:

- The API reads the whole tab: `readValues(sheetId, "SCHEDULE", { render: "UNFORMATTED_VALUE" })`.
  A bare tab name is a valid A1 range and returns the used grid, row 1 at
  index 0. Rows can be short (trailing empties dropped): a missing cell is `""`.
- A cell in a match-number column at row ≥ 6 matches when it is the number
  `n`, or a string whose trim is the digits of `n` (`"42"`).
- Codes are compared **trimmed**. A playoff code that is a formula reads as
  its computed value, which is what the snapshot (and so the client) shows.
- **Zero hits:** 404, `Match #42 isn't in SCHEDULE. Its row may have been moved or cleared.`
- **Two or more hits:** 422, `Match #42 appears more than once in SCHEDULE (AD14, F30). Fix the sheet, then try again.`

---

## 4. `sage-tools-api`

Work in `D:\Personal\SAGE\sage-tools-api`. Run `npm test` at the end of each
numbered subsection that changes `src/` or `test/`; it must be green before
you move on.

### 4.1 Files

| File | New/changed |
|---|---|
| `src/scores/scheduleGrid.mjs` | new (§4.2) |
| `src/scores/scoreRequest.mjs` | new (§4.3) |
| `src/shared/errors.mjs` | changed (§4.4) |
| `src/attendance/SheetsClient.mjs` | changed (§4.5) |
| `src/sync/SyncConfigStore.mjs`, `src/sync/SyncConfigSnapshot.mjs` | changed (§4.6) |
| `src/auth/AuthService.mjs` | changed (§4.7) |
| `src/scores/ScoreService.mjs` | new (§4.8) |
| `src/scores/routes.mjs` | new (§4.9) |
| `src/server/Server.mjs`, `src/docs/openapiSpec.mjs`, `index.mjs` | changed (§4.10) |
| `test/…` | §4.11–4.13 |
| `package.json`, `README.md` | §4.14 |

### 4.2 `src/scores/scheduleGrid.mjs`

Pure functions, no imports. Write it as:

```js
// Where a match and its scores sit in a facility workbook's SCHEDULE tab.
// Every workbook type (dual meet, standard, team) lays SCHEDULE out in
// 8-column court blocks from column D, data from row 6, two rows per slot:
// team 1 code, match number, team 2 code, names, team 1 score, team 2 score.
// See sage-docs/docs/specs/.../control-center-score-entry-spec.md §3.
// Columns and rows here are 1-based, as in A1 notation.

export const FIRST_DATA_ROW = 6;
export const FIRST_MATCH_COL = 6;                       // F
export const BLOCK = 8;
export const FIRST_SCORE_COL = FIRST_MATCH_COL + 3;     // I

export function isMatchNumberColumn(col) {
    return Number.isInteger(col) && col >= FIRST_MATCH_COL && (col - FIRST_MATCH_COL) % BLOCK === 0;
}

// The team-1 score column of a court block: I, Q, Y, AG, … (9 + 8k).
export function isTeam1ScoreColumn(col) {
    return Number.isInteger(col) && col >= FIRST_SCORE_COL && (col - FIRST_SCORE_COL) % BLOCK === 0;
}

// 1 -> "A", 26 -> "Z", 27 -> "AA", 702 -> "ZZ", 703 -> "AAA".
export function columnLetter(col) {
    if (!Number.isInteger(col) || col < 1) throw new RangeError(`not a column number: ${col}`);
    let s = "";
    for (let n = col; n > 0; n = Math.floor((n - 1) / 26)) s = String.fromCharCode(65 + ((n - 1) % 26)) + s;
    return s;
}

// "A" -> 1, "AA" -> 27. Throws on anything but 1–3 capital letters.
export function columnNumber(letters) {
    if (!/^[A-Z]{1,3}$/.test(letters)) throw new RangeError(`not a column: ${letters}`);
    return [...letters].reduce((n, ch) => n * 26 + (ch.charCodeAt(0) - 64), 0);
}

const cellText = v => (v === null || v === undefined ? "" : String(v).trim());

function isMatchNumber(v, n) {
    if (typeof v === "number") return v === n;
    const t = cellText(v);
    return /^\d+$/.test(t) && Number(t) === n;
}

/** Every match-number cell at or below row 6 holding n. -> [{ row, col }] */
export function findMatchCells(values, n) {
    const hits = [];
    for (let row = FIRST_DATA_ROW; row <= values.length; row++) {
        const cells = values[row - 1] ?? [];
        for (let col = FIRST_MATCH_COL; col <= cells.length; col += BLOCK) {
            if (isMatchNumber(cells[col - 1], n)) hits.push({ row, col });
        }
    }
    return hits;
}

/** The cells around one match-number cell. */
export function readMatchAt(values, { row, col }) {
    const cells = values[row - 1] ?? [];
    const scoreCol = col + 3;
    return {
        row,
        cell: `${columnLetter(col)}${row}`,
        teamCode1: cellText(cells[col - 2]),
        teamCode2: cellText(cells[col]),
        scoreRange: `SCHEDULE!${columnLetter(scoreCol)}${row}:${columnLetter(scoreCol + 1)}${row}`,
    };
}
```

### 4.3 `src/scores/scoreRequest.mjs`

Validates the score route's input. Pure; imports only `ValidationError`.

```js
export const MAX_SCORE = 99;

/**
 * matchNumberParam: the path segment (a string). body: the parsed JSON body.
 * -> { matchNumber, team1Score, team2Score, expected: { teamCode1, teamCode2, team1Score, team2Score } }
 * Throws ValidationError naming the first bad field.
 */
export function parseScoreRequest(matchNumberParam, body) {}

/** body of PUT /v1/events/:event/score-entry -> "console" | "links". */
export function parseScoreEntryMode(body) {}
```

`parseScoreRequest` rules, in this order, each failure a `ValidationError`
with exactly this message:

| Check | Message |
|---|---|
| `matchNumberParam` matches `/^[1-9]\d{0,4}$/` | `matchNumber must be a positive whole number` |
| `body` is a non-null object | `Body must be a JSON object` |
| `team1Score` and `team2Score` are each `null` or an integer 0–99 (`Number.isInteger`) | `team1Score must be a whole number from 0 to 99, or null` (and the `team2Score` twin) |
| both `null` or neither | `team1Score and team2Score must both be numbers, or both null to clear the score` |
| `expected` is a non-null object | `expected is required` |
| `expected.teamCode1`, `expected.teamCode2` are strings non-empty after trim | `expected.teamCode1 must be a non-empty string` (and the twin) |
| `expected.team1Score`, `expected.team2Score` are each `null` or an integer 0–99 (independently: a sheet can hold one score) | `expected.team1Score must be a whole number from 0 to 99, or null` (and the twin) |

Return the codes trimmed and `matchNumber` as a number. A missing score key
counts as `undefined`, which fails the first score check.

`parseScoreEntryMode(body)`: `body?.mode` must be exactly `"console"` or
`"links"`, else `ValidationError('mode must be "console" or "links"')`.

### 4.4 Errors (`src/shared/errors.mjs`)

Append, after `ServiceBusyError`, under a `// Score entry (src/scores/).` comment:

```js
// The event's "scoreEntry" setting is absent (or the event is archived).
export class ScoreEntryOffError extends ForbiddenError {
    constructor() {
        super("Score entry is off for this event");
    }
}

// SCHEDULE is in a state the API won't write into: no tab, the match number
// twice, or a score cell holding a formula or text.
export class ScheduleLayoutError extends AppError {
    constructor(message) {
        super(message, 422);
    }
}

// The sheet no longer matches what the client showed. `current` is what it
// holds now: { teamCode1, teamCode2, team1Score, team2Score }.
export class ScoreConflictError extends AppError {
    constructor(current) {
        super("The sheet changed since this match was opened", 409);
        this.current = current;
    }
}
```

### 4.5 `SheetsClient`: two score methods with their own allowlist

In `src/attendance/SheetsClient.mjs`. **Do not change `updateValues` or its
`WRITABLE_RANGE`.** Add:

```js
import { FIRST_DATA_ROW, columnNumber, isTeam1ScoreColumn } from "../scores/scheduleGrid.mjs";

// The only SCHEDULE cells the API may ever change: one match's two score
// cells, side by side on one row at or below row 6, starting in a team-1
// score column (I, Q, Y, … = 9 + 8k). Checked before any request is made, so
// a bug in score entry can never overwrite a code, a name or a formula.
// See sage-docs/docs/specs/.../control-center-score-entry-spec.md §4.5.
const SCORE_RANGE = /^SCHEDULE!([A-Z]{1,3})([1-9]\d*):([A-Z]{1,3})([1-9]\d*)$/;

function assertScoreRange(range) {
    const m = SCORE_RANGE.exec(range);
    const ok = m
        && m[2] === m[4]
        && Number(m[2]) >= FIRST_DATA_ROW
        && isTeam1ScoreColumn(columnNumber(m[1]))
        && columnNumber(m[3]) === columnNumber(m[1]) + 1;
    if (!ok) throw new Error(`SheetsClient: write outside the score allowlist: ${range}`);
}
```

and two methods on the class:

```js
// One match's two scores, as numbers (RAW), exactly what typing them stores.
async writeScores(sheetId, range, [team1Score, team2Score]) {
    assertScoreRange(range);
    for (const s of [team1Score, team2Score]) {
        if (!Number.isInteger(s) || s < 0 || s > 99) throw new Error(`SheetsClient: not a score: ${s}`);
    }
    return this.#batchUpdateValues(sheetId, [{ range, values: [[team1Score, team2Score]] }], "RAW");
}

// Empties one match's two score cells. values:batchClear, not a write of "",
// so the cells are truly empty and ISBLANK reads them as unplayed.
async clearScores(sheetId, range) {
    assertScoreRange(range);
    const url = `${BASE}/${encodeURIComponent(sheetId)}/values:batchClear`;
    const res = await this.#send(url, { method: "POST", body: JSON.stringify({ ranges: [range] }) }, { write: true });
    await this.#assertOk(res, { write: true });
    return res.json();
}
```

Update the class's header comment: it now writes `ATTENDANCE!A:G` **and**
one match's two `SCHEDULE` score cells, each through its own allowlist.

### 4.6 Config

**Validation.** `src/sync/SyncConfigStore.mjs`, in `validate(raw)`, directly
after the `attendance` check:

```js
if (eventEntry?.scoreEntry !== undefined && eventEntry.scoreEntry !== "console" && eventEntry.scoreEntry !== "links") {
    throw new Error(`event "${eventKey}" has an invalid scoreEntry value (must be "console" or "links")`);
}
```

**Reading.** `src/sync/SyncConfigSnapshot.mjs`, `getEvent` returns two more
fields, and its comment becomes "The event-level settings attendance and
score entry need":

```js
return {
    event: eventKey,
    type: entry.type ?? null,
    attendance: entry.attendance ?? null,
    scoreEntry: entry.scoreEntry ?? null,      // null | "console" | "links"
    archived: entry.archived === true,
};
```

**Writing (the Mission Control switch).** Add to `SyncConfigStore`, beside
`setLivePush` and written the same way (fetch with sha, `validate`, change,
publish with the sha, retry a 409 up to `COMMIT_ATTEMPTS`, clear the cache):

```js
// Mission Control's "Scorer links" switch: moves an event between "links"
// (operators and scorer links) and "console" (operators only). Never turns
// score entry on or off: an event without a scoreEntry setting is refused.
// -> { event, scoreEntry, changed }
async setScoreEntry(eventKey, mode) {}
```

- `mode` not `"console"`/`"links"` → `ValidationError('mode must be "console" or "links"')`.
- The event isn't in the fetched config → `UnknownEventError(eventKey)`.
- Its `scoreEntry` is absent → `ValidationError("Score entry is off for this event. Turn it on in events.json first.")`.
- Already `mode` → `{ event, scoreEntry: mode, changed: false }`, nothing committed.
- Else set it, commit with the message `set ${eventKey} scoreEntry=${mode}`,
  clear the cache, return `changed: true`.

### 4.7 Scorer tokens (`src/auth/AuthService.mjs`)

Scorer tokens mirror desk tokens with their own scope, `"score-desk"`, and so
their own derived signing key: a scorer token never verifies as an operator
token or a desk token, and neither of those verifies as a scorer token.

- Replace `#signDesk(payload)` with `#signScoped(scope, payload)`, which
  derives the key from `scope` exactly as `#signDesk` does today from
  `DESK_SCOPE`. Desk tokens call it with `DESK_SCOPE`; every existing desk
  token test must still pass unchanged.
- Add `const SCORER_SCOPE = "score-desk";` and:

```js
// A scorer token allows exactly one thing: entering scores on one day, while
// the event's scoreEntry is "links". Same construction as a desk token, with
// its own scope and so its own signing key. See
// sage-docs/docs/specs/.../control-center-score-entry-spec.md §4.7.
// Returns { token, expiresAt }, or null when AUTH_TOKEN_SECRET is unset.
issueScorerToken({ day, expiresAt }) {}

// { day, exp } for a well-formed, correctly signed, unexpired scorer token;
// null for anything else.
verifyScorerToken(token) {}
```

Write them as copies of `issueDeskToken`/`verifyDeskToken` with
`SCORER_SCOPE`. Two small shared private helpers are fine if they keep the
desk behaviour identical.

### 4.8 `src/scores/ScoreService.mjs`

```js
export class ScoreService {
    // configStore: SyncConfigStore. sheets: SheetsClient. syncService: SyncService. authService: AuthService.
    constructor({ configStore, sheets, syncService, authService, logger, now = () => new Date() }) {}

    // actor: { kind: "operator" } | { kind: "scorer", day }. -> the 200 body of §4.9.
    async submit({ day, facilityName, matchNumber, team1Score, team2Score, expected, actor }) {}

    // -> { token, expiresAt, day }
    async issueScorerLink(day) {}

    // -> { event, scoreEntry, changed }
    async setScoreEntryMode(eventKey, mode) {}
}
```

**`submit`, step by step:**

1. `const config = await this.configStore.get();`
   `const { event, facilities } = config.getDay(day);` (unknown day → its own 400).
   `const { scoreEntry, archived } = config.getEvent(event);`
   - `!scoreEntry || archived` → `ScoreEntryOffError`.
   - `actor.kind === "scorer"`: `scoreEntry !== "links"` →
     `new ForbiddenError("Scorer links are stopped for this event. Ask the operator.")`;
     `actor.day !== day` → `new ForbiddenError("This scorer link is for another day. Ask the operator for today's.")`.
   - Find the facility by exact name; if absent throw
     `new ValidationError(\`Unknown facility "${facilityName}". Known facilities: ${names}\`)`
     (the same wording as attendance's `unknownFacility`).
2. `values = await this.sheets.readValues(sheetId, "SCHEDULE", { render: "UNFORMATTED_VALUE" })`.
   `null` → `ScheduleLayoutError("This workbook has no SCHEDULE tab")`.
3. `hits = findMatchCells(values, matchNumber)`. Zero → `NotFoundError` (§3
   message). More than one → `ScheduleLayoutError` (§3 message, cells via
   `columnLetter(col) + row`, joined `", "`). Then `at = readMatchAt(values, hits[0])`.
4. `cells = (await this.sheets.readValues(sheetId, at.scoreRange, { render: "FORMULA" })) ?? []`,
   `[raw1, raw2] = cells[0] ?? []`. Turn each into a score:
   - `undefined`, `null` or a string empty after trim → `null`;
   - an integer number → itself; a string of digits → `Number` of it;
   - a string starting with `=` → throw
     `ScheduleLayoutError(\`Match #${n}'s score cell ${cell} holds a formula. Enter this score in the sheet.\`)`;
   - anything else → throw
     `ScheduleLayoutError(\`Match #${n}'s score cell ${cell} holds "${text}", not a score. Enter this score in the sheet.\`)`.

   `cell` is that score cell's A1 (`AG14` or `AH14`). Call the results
   `previous = { team1Score, team2Score }`.
5. `current = { teamCode1: at.teamCode1, teamCode2: at.teamCode2, ...previous }`.
   - Codes differ from `expected`'s → throw `new ScoreConflictError(current)`.
   - `unchanged = previous.team1Score === team1Score && previous.team2Score === team2Score`.
   - `!unchanged` and `previous` differs from `expected`'s scores → throw `new ScoreConflictError(current)`.
6. If `!unchanged`: `team1Score === null` → `await this.sheets.clearScores(sheetId, at.scoreRange)`;
   otherwise `await this.sheets.writeScores(sheetId, at.scoreRange, [team1Score, team2Score])`.
7. Publish (even when `unchanged`: a retried save may have written without
   publishing):
   ```js
   let sync;
   try {
       const result = await this.syncService.syncDay(day, { facilityName, editAt: this.now().getTime() });
       sync = result.facilitiesFailed?.includes(facilityName)
           ? { ok: false, error: "The score is in the sheet, but the workbook couldn't be read back to publish it" }
           : { ok: true };
   } catch (err) {
       sync = { ok: false, error: err?.message ?? String(err) };
   }
   ```
8. Log one line with `this.logger.info` (the audit trail):
   `score ${day}/${facilityName} #${n} ${cellRange} ${fmt(previous)} -> ${fmt(new)}${unchanged ? " (unchanged)" : ""} by=${actor.kind} sync=${sync.ok ? "ok" : "failed"}`,
   where `fmt` gives `11:7`, or `–:–` for nulls, and `cellRange` is `AG14:AH14`.
9. Return:
   ```js
   {
       day, facility: facilityName, matchNumber,
       cell: at.scoreRange,
       teamCode1: at.teamCode1, teamCode2: at.teamCode2,
       team1Score, team2Score,              // as stored now (null, null after a clear)
       previous,
       unchanged,
       sync,
   }
   ```

**Known window, accepted.** Between step 4's read and step 6's write (a few
hundred milliseconds) a scorer typing the same match in the sheet would be
overwritten. Closing it needs the workbook's document lock, which only Apps
Script can take. Step 7 publishes whatever the sheet then holds, so the pages
always show the sheet.

**`issueScorerLink(day)`:**

1. `getDay(day)` (400 if unknown), `getEvent(event)`.
2. `!scoreEntry || archived` → `ScoreEntryOffError`.
   `scoreEntry !== "links"` →
   `ValidationError("Scorer links are stopped for this event. Set them to Accepting first.")`.
3. If the day has a `date`: the day is over from **06:00 Manila the morning
   after it** (`Date.parse(\`${date}T06:00:00+08:00\`) + 24 * 60 * 60 * 1000`),
   matching Control Center's event-day rollover. At or past that →
   `ValidationError("This day is over")`. A day without a `date` is allowed.
4. `expiresAt = this.now().getTime() + 24 * 60 * 60 * 1000`.
5. `issued = this.authService.issueScorerToken({ day, expiresAt })`; `null` →
   `ValidationError("Scorer links are not available: the server has no AUTH_TOKEN_SECRET")`.
6. Log `scorer link issued ${day} until ${new Date(expiresAt).toISOString()}`.
   Return `{ token: issued.token, expiresAt: issued.expiresAt, day }`.

**`setScoreEntryMode(eventKey, mode)`:** `config.getEvent(eventKey)` (404 if
unknown); archived → `ScoreEntryOffError`; then
`return this.configStore.setScoreEntry(eventKey, mode)` (which re-reads and
does its own checks, §4.6). Log `score entry ${eventKey} -> ${mode}` when
`changed`.

### 4.9 `src/scores/routes.mjs`

```js
export function scoreRoutes({ scoreService, authService, logger }) { … return router; }
```

Local auth helpers, in attendance's style:

- `bearerToken(req)`: as attendance's.
- `requireOperator`: operator token only, else `401 { error: "Unauthorized" }`.
- `requireOperatorOrScorer`: an operator token sets `req.actor = { kind: "operator" }`;
  else a valid scorer token (`authService.verifyScorerToken`) sets
  `req.actor = { kind: "scorer", day }`; else 401. A desk token is a 401.

Every handler: `try { … } catch (err) { logger.error(…, err); res.status(err.statusCode ?? 500).json({ error: err.message, ...(err.current ? { current: err.current } : {}) }); }`.

| Route | Auth | Handler |
|---|---|---|
| `PUT /days/:day/facilities/:facility/matches/:matchNumber/score` | `requireOperatorOrScorer` | `parseScoreRequest(req.params.matchNumber, req.body)`, then `scoreService.submit({ day, facilityName: req.params.facility, ...parsed, actor: req.actor })` → 200 |
| `POST /days/:day/scores/scorer-links` | `requireOperator` | `scoreService.issueScorerLink(req.params.day)` → **201** |
| `PUT /events/:event/score-entry` | `requireOperator` | `scoreService.setScoreEntryMode(req.params.event, parseScoreEntryMode(req.body))` → 200 |

Each route gets an `@openapi` block in the attendance route's format, tag
`scores`:

- score: `operationId: submitMatchScore`, summary `Enter, correct or clear one
  match's score`; description says it accepts an operator token or a scorer
  token (the latter only for its own day and only while the event's
  scoreEntry is "links"); the three path parameters; the request body of §2
  with an example; the 200 body of §4.8; responses
  `400 401 403 404 409 422 502 503`, 409's body
  `{ error, current: { teamCode1, teamCode2, team1Score, team2Score } }`.
- scorer link: `operationId: issueScorerLink`, summary `Issue a scorer link
  for one day`; 201 `{ token, expiresAt (Unix ms), day }`; `400 401 403`.
- switch: `operationId: setScoreEntryMode`, summary `Accept or stop scorer
  links for an event`; body `{ mode: "console" | "links" }`; 200
  `{ event, scoreEntry, changed }`; `400 401 403 404`.

### 4.10 Wiring

- `src/server/Server.mjs`: the constructor and `#registerRoutes` take
  `scoreService`; mount `this.app.use("/v1", scoreRoutes({ scoreService, authService, logger: this.logger }))`
  directly after the attendance mount.
- `src/docs/openapiSpec.mjs`: add `{ name: "scores" }` to `tags` (after
  `attendance`), `path.join(ROOT, "src/scores/routes.mjs")` to `apis` (after
  attendance's), `src/scores/routes.mjs` to the header comment's list, and
  extend `bearerAuth`'s description: "…The attendance PUT route also accepts
  a desk token, and the score PUT route a scorer token."
- `index.mjs`: import `ScoreService`. Hoist the attendance
  `new GoogleAccessToken(…)` into a `const googleAccessToken` used by both
  services. After `syncService` is constructed:
  ```js
  // Score entry writes one match's score into SCHEDULE as the same service
  // account, then publishes through syncService; it also issues scorer links
  // and flips the event's scorer-link switch. Its own SheetsClient for the
  // longer timeout: a team workbook can take up to a minute to answer a read
  // while it recalculates. See sage-docs/docs/specs/.../control-center-score-entry-spec.md.
  const scoresLogger = logger.child("scores");
  const scoreService = new ScoreService({
      configStore: syncConfigStore,
      sheets: new SheetsClient({ apiKey: GOOGLE_SHEETS_API_KEY, accessToken: googleAccessToken, timeoutMs: 60000, logger: scoresLogger }),
      syncService,
      authService,
      logger: scoresLogger,
  });
  ```
  and pass `scoreService` to `new Server(…)`.

### 4.11 Test helpers

- `test/helpers/http.mjs` `startApp`: add `scoreService: {},` beside
  `attendanceService: {},`.
- `test/helpers/buildApp.mjs`: mirror §4.10's `index.mjs` changes exactly (it
  is a copy of those lines).
- `test/helpers/routeManifest.mjs`: three lines, all pointing at
  `test/integration/score-routes.test.mjs`:
  `PUT /v1/days/:day/facilities/:facility/matches/:matchNumber/score`,
  `POST /v1/days/:day/scores/scorer-links`, `PUT /v1/events/:event/score-entry`.
- `test/helpers/coveredElsewhere.mjs`: add
  `"src/scores/routes.mjs": ["test/integration/score-routes.test.mjs", "route handlers"],`
- `test/helpers/fakeSheets.mjs` (`FakeSheets`, used by service unit tests):
  - `parseRef` and `colIndex` accept **multi-letter** columns (`AG14:AH14`).
  - `readValues` with a **bare tab name** (no `!`) returns the whole tab, trailing blank rows trimmed.
  - A cell may be `{ formula: "=…", value: v }`: `readValues` returns
    `formula` when `render === "FORMULA"`, otherwise `value`. Plain cells are
    returned as they are for every render (existing tests keep working).
  - `writeScores(sheetId, range, scores)` and `clearScores(sheetId, range)`
    apply to that workbook's `SCHEDULE` tab, push
    `{ sheetId, range, values }` / `{ sheetId, range }` onto new arrays
    `this.scoreWrites` / `this.scoreClears`, and honour `failOn`. Clearing
    sets both cells to `undefined`.

`FakeWorld` is **not** extended: no pipeline test drives `SheetsClient`, as
for attendance. `SheetsClient`'s new calls are pinned by its stubbed-fetch
unit tests. Say so in the Changelog's Tests line.

### 4.12 New tests

Use `node:test` and `node:assert/strict`, in the style of the neighbouring
files (read `test/unit/attendance/AttendanceService.test.mjs`,
`test/unit/auth/` and `test/integration/attendance-routes.test.mjs` first).

`test/unit/scores/scheduleGrid.test.mjs`:
- §3's worked example: a grid with 42 at row 14, column 30 gives
  `{ row: 14, cell: "AD14", teamCode1, teamCode2, scoreRange: "SCHEDULE!AG14:AH14" }`.
- `findMatchCells` on three grid widths (dual meet 9 courts = 74 columns,
  standard 4 courts = 35, team 83) finds a match in the last block.
- ignores the number in a non-match column (`E`, `G`, `I`) and above row 6;
  accepts `"42"` and `" 42 "`; ignores `"42a"`; finds two hits; tolerates short rows.
- `columnLetter`: 1 `A`, 26 `Z`, 27 `AA`, 52 `AZ`, 53 `BA`, 702 `ZZ`, 703 `AAA`;
  throws on 0. `columnNumber` is its inverse for those, throws on `"a"`, `""`, `"AAAA"`.
- `isTeam1ScoreColumn`: true for 9, 17, 33, 81; false for 10, 8, 1.

`test/unit/scores/scoreRequest.test.mjs`: one passing request; a clear; every
row of §4.3's table with its exact message; codes come back trimmed;
`parseScoreEntryMode` accepts both modes and refuses `true`, `"off"`, a missing body.

`test/unit/scores/ScoreService.test.mjs` (with `FakeSheets`, a fake
`configStore` whose `get()` returns a real `SyncConfigSnapshot` and whose
`setScoreEntry` records calls, a fake `syncService` that records calls, and a
real `AuthService` with a test secret):
- **submit, operator:** writes `[11, 7]` to the right range, then calls
  `syncDay(day, { facilityName, editAt })` with `editAt` from the injected
  `now`; returns the full §4.8 body. Works with `scoreEntry` `"console"` and `"links"`.
- clear calls `clearScores`, not `writeScores`, and returns `null, null`.
- a correction (`expected` = the sheet's old scores) writes.
- 409 when a code differs, and when the scores differ from `expected` (and
  from the new ones); `err.current` is right; nothing written, `syncDay` not called.
- `unchanged`: the sheet already holds the new scores: no write, `syncDay` still called.
- 422 for no tab, a duplicate match number, a formula score cell, a text score cell; nothing written.
- 404 for a missing match; 400 for an unknown facility; `UnknownSyncDayError` for an unknown day.
- 403 when `scoreEntry` is absent or the event is `archived`.
- a `syncDay` that throws → 200 body with `sync.ok === false` and its message;
  one that lists the facility in `facilitiesFailed` → `sync.ok === false`.
- a `SheetsClient` failure while reading propagates; nothing written.
- **submit, scorer:** works with `"links"` on its own day (any facility of the
  day); 403 with `"console"` (message as §4.8); 403 for another day's token;
  the log line says `by=scorer`.
- **issueScorerLink:** `expiresAt` is `now + 24 h`; the token verifies with
  `verifyScorerToken` for that day; 403 when off/archived; 400 when
  `"console"`; 400 at and after 06:00 Manila the morning after `date`, allowed
  one minute before; allowed for a day without a `date`; 400 when
  `AUTH_TOKEN_SECRET` is unset.
- **setScoreEntryMode:** passes through to `configStore.setScoreEntry`; 404
  unknown event; 403 archived.

`test/unit/sync/SyncConfigStore.test.mjs` (`setScoreEntry`, with the fake
publisher the file already uses for `setLivePush`): commits the new value
with the §4.6 message and clears the cache; `changed: false` and no commit
when unchanged; refuses an event without `scoreEntry`; `UnknownEventError`;
retries after a 409 and re-applies to the re-read config.

`test/unit/sync/scoreEntryConfig.test.mjs` (pattern:
`test/unit/sync/attendanceConfig.test.mjs`): `"console"`/`"links"` accepted;
absent → `scoreEntry: null`; `true`, `"yes"`, `1`, `null` rejected with
§4.6's message; `archived: true` → `archived: true`.

`test/unit/auth/AuthService.test.mjs`: a scorer token round-trips with its
`day` and `exp`; it is refused by `verify` and `verifyDeskToken`; a desk token
and an operator token are refused by `verifyScorerToken`; expired, tampered,
malformed and no-secret cases return `null`; `issueScorerToken` is `null`
without a secret. Every existing desk-token test unchanged and green.

`test/unit/attendance/SheetsClient.test.mjs` (a `describe` for each new
method, stubbing `fetch` as the file already does):
- `writeScores` sends `POST …/values:batchUpdate` with the bearer token and
  body `{ valueInputOption: "RAW", data: [{ range: "SCHEDULE!AG14:AH14", values: [[11, 7]] }] }`.
- `clearScores` sends `POST …/values:batchClear` with `{ ranges: ["SCHEDULE!AG14:AH14"] }`.
- both throw **without any fetch** for: `SCHEDULE!H14:I14` (not a score
  column), `SCHEDULE!AG5:AH5` (row 5), `SCHEDULE!AG14:AI14` (not adjacent),
  `SCHEDULE!AG14:AH15` (two rows), `ATTENDANCE!A2:B2`, `Schedule!I6:J6`.
- `writeScores` throws without fetch for `11.5`, `"11"`, `-1`, `100`.
- a 403 on either names the service account (as `updateValues` already does).
- `updateValues` still refuses `SCHEDULE!I6:J6`.

`test/unit/shared/errors.test.mjs`: the three new classes' status and name
(`ScoreEntryOffError` 403 and `instanceof ForbiddenError`, `ScheduleLayoutError`
422, `ScoreConflictError` 409 carrying `current`), and add them to the list
the file checks in bulk.

`test/integration/score-routes.test.mjs` (pattern:
`test/integration/attendance-routes.test.mjs`: a fake `scoreService` passed
to the real `Server`, a real `AuthService`):
- **score:** 200 with an operator token (actor `{ kind: "operator" }`) and
  with a scorer token (actor `{ kind: "scorer", day }`); the service receives
  the decoded path (`Court%201` → `Court 1`) and the parsed body. 401 with no
  token, a bad token, a desk token, and only `X-Sync-Secret`. 400 for a bad
  body and for `matchNumber` `0`, `abc` (service not called). Each service
  error maps to its status: 403, 404, 409 (body includes `current`), 422,
  502, 503; a plain `Error` → 500. `OPTIONS` preflight → 204 with `PUT`.
- **scorer-links:** 201 with an operator token; 401 with a scorer token, a
  desk token, none; service errors map to 400/403.
- **score-entry:** 200 with an operator token and `{ mode: "links" }`; 400
  for `{ mode: "off" }` (service not called); 401 with a scorer token; 404
  from the service passes through.

### 4.13 Existing tests that change on purpose

- `test/unit/sync/SyncConfigSnapshot.test.mjs` (two `getEvent` `deepEqual`s)
  and `test/unit/sync/attendanceConfig.test.mjs` (one): add
  `scoreEntry: null, archived: false` to the expected objects.
- `test/unit/helpers/routes.test.mjs`: 12 → 15 routes.
- `test/unit/docs/openapiSpec.test.mjs`: "11 paths" → "14 paths", adding
  `/v1/days/{day}/facilities/{facility}/matches/{matchNumber}/score`,
  `/v1/days/{day}/scores/scorer-links` and `/v1/events/{event}/score-entry`
  to the list, and the `bearerAuth` description if the file pins it. Then run
  `npm run test:snapshots` once, read the `.snapshot` diff (it should only add
  the three paths, the tag, their schemas and the description change) and
  leave it.

### 4.14 Version and Changelog

`package.json` `"version": "2.8.0"`. In `README.md`'s Changelog, above
`### 2.7.1`:

```markdown
### 2.8.0

- Score entry. `PUT /v1/days/{day}/facilities/{facility}/matches/{matchNumber}/score`
  writes one match's two score cells in the facility workbook's `SCHEDULE`
  tab as the attendance service account, or clears them, then publishes that
  facility through `syncDay`: an API write fires no onEdit trigger. It refuses
  with 409 and the sheet's current values when the sheet no longer matches
  what the client showed, and with 422 when the match number appears twice or
  a score cell holds a formula or text. It accepts an operator token, or a
  scorer token for its own day while the event's scoreEntry is "links".
- Scorer links. `POST /v1/days/{day}/scores/scorer-links` (operator only)
  issues a scorer token valid for 24 hours. `PUT /v1/events/{event}/score-entry`
  (operator only) moves the event's `scoreEntry` between "links" and
  "console" in `events.json`: Mission Control's switch.
- `events.json` gains an optional event-level `scoreEntry`: "console" or
  "links" (absent = off). `getEvent` also returns `scoreEntry` and `archived`.
  `SheetsClient` gains `writeScores` and `clearScores`, which accept only one
  match's two score cells at row 6 or below. See
  `sage-docs/docs/specs/.../control-center-score-entry-spec.md`.

  Tests: `scheduleGrid.test.mjs`, `scoreRequest.test.mjs`,
  `ScoreService.test.mjs`, `scoreEntryConfig.test.mjs`,
  `score-routes.test.mjs` (new); `SheetsClient.test.mjs`, `AuthService.test.mjs`
  (scorer tokens), `SyncConfigStore.test.mjs` (`setScoreEntry`),
  `errors.test.mjs`, `SyncConfigSnapshot.test.mjs` and
  `attendanceConfig.test.mjs` (`getEvent` returns `scoreEntry` and
  `archived`), `routes.test.mjs` (15 routes), `openapiSpec.test.mjs` and its
  snapshot (14 paths), `fakeSheets.mjs`, `http.mjs`, `buildApp.mjs`,
  `routeManifest.mjs`, `coveredElsewhere.mjs`. `FakeWorld` is unchanged: the
  new Sheets calls are covered by `SheetsClient`'s stubbed-fetch tests, as
  attendance's are.
```

**Finish §4 with `npm run verify`.** It must pass.

---

## 5. Control Center (`sage-match-control.github.io/tools/control-center.html`)

### 5.1 The `SCORE CLIENT` block

The dialog is shared with the scorer page (§6), so it lives in one
**byte-identical** block in both pages, everything page-specific passed in.
Place it in Control Center right after `ticketHTML`, before the top-level
`renderIntro();` call that runs at load (`let`/`const` in it must exist
before anything at load reaches them).

```js
// ==== SCORE CLIENT — identical in tools/control-center.html, ====
// ==== _templates/scorer/scorer.html and every events/<key>/scorer.html; ====
// ==== see sage-docs/docs/specs/.../control-center-score-entry-spec.md §5 ====
// The score dialog: Enter -> Review -> Save, conflicts, and the save request.
// Everything that differs between pages is passed in.

const SCORE_SAVE_TIMEOUT_MS = 120000;
const SCORE_KEY_GUARD_MS = 400;
const SCORE_SLOW_MS = 5000;

/**
 * opts:
 *   dialog            the <dialog> element (markup §5.2, the same in every page)
 *   apiBase           Cloud Run base URL
 *   getToken()        bearer token, or null
 *   fixture           string | null: when set, saves are simulated (onFixtureSave)
 *   describe(m)       -> { title, sub, sides: [side, side] },
 *                        side = { name, code, players: [p1, p2] | null, missing }  // missing: shown when players is null
 *   readOnlyReason(m) -> string | null   (non-null opens the dialog read-only with that text)
 *   extraWarnings(m)  -> string[]        (series, lineup: page-specific)
 *   findMatch(facility, num) -> the page's current copy of that match, or null
 *   notify(kind, text)                   kind: 'ok' | 'warn' | 'error'
 *   friendlyError(raw) -> string
 *   unreachable(err)  -> string
 *   expiredText       shown on a 401
 *   resyncHint        appended to the "publishing failed" message
 *   onFixtureSave(state, team1Score, team2Score)  page patches its data and redraws
 * -> { open(day, facility, m), close(), refresh(), isOpen() }
 */
function createScoreDialog(opts){ … }

// ==== END SCORE CLIENT ====
```

- `open(day, facility, m)` copies `m` and sets the state (§5.3); `close()`
  closes and forgets it; `refresh()` is §5.5, called by the page after each
  new snapshot; `isOpen()` is true while the dialog shows.
- The block uses no global of either page. Everything else it needs is
  inside it or in `opts`.
- `match` objects have the shape in §0.4 (Control Center's `rowsToMatches`
  output plus `facility`); the scorer page builds the same shape (§6.3).

### 5.2 Markup and style (same in both pages)

Markup, placed just before Control Center's toast stack (`id="toastStack"`),
and in the scorer page just before its `<script>`:

```html
<dialog id="scoreDialog" class="score-dialog" aria-labelledby="scoreDialogTitle">
  <div class="score-head">
    <div>
      <h2 id="scoreDialogTitle" class="score-title"></h2>
      <div id="scoreDialogSub" class="score-sub"></div>
    </div>
    <button type="button" class="score-x" id="scoreDialogClose" aria-label="Close">&times;</button>
  </div>
  <div id="scoreDialogBody"></div>
  <p id="scoreDialogNote" class="score-note" hidden></p>
  <div id="scoreDialogError" class="score-error" role="alert" hidden></div>
  <div id="scoreDialogActions" class="score-actions"></div>
</dialog>
```

The block finds its parts with `opts.dialog.querySelector('#scoreDialog…')`.

Styles (copy the same rules into both pages; the scorer page defines the
same `:root` token names, §6.1):

- `.score-dialog`: `width:min(520px, calc(100vw - 32px))`, `border:none`,
  `border-radius:var(--radius)`, `padding:20px`, `background:var(--white)`,
  `color:var(--ink)`, `box-shadow:var(--card-shadow)`.
  `::backdrop`: `background:rgba(11,24,38,.55)`.
- `@media (max-width:599px)`: a bottom sheet: `width:100%`, `max-width:100%`,
  `margin:auto 0 0`, `border-radius:var(--radius) var(--radius) 0 0`,
  `padding-bottom:calc(20px + env(safe-area-inset-bottom))`.
- `.score-sides`: two equal columns, 12 px gap. Each side: name (bold), code
  (small chip) when it differs from the name, players (`--ink-soft`), then the input.
- `.score-input`: `font-size:32px`, centred, `width:100%`, `height:64px`,
  `border:2px solid var(--line)`, `border-radius:10px`; `:focus` border `var(--green)`.
- `.score-winner`: 22 px, weight 800. `.score-big`: 44 px, weight 800,
  centred, `font-variant-numeric:tabular-nums`. `.score-was`: `--ink-soft`.
- `.score-warn`: the `.toast.warn` colours, a list. `.score-error`: the
  `.toast.error` colours. `.score-note`: `--ink-soft`, italic.
- `.score-actions`: right-aligned buttons, 10 px gap, wrapping. Primary:
  `background:var(--navy); color:var(--white)`. Secondary:
  `background:var(--paper-dim); color:var(--ink)`. Disabled `opacity:.5`.
- `.scoreable`: `cursor:pointer`; `:hover` and `:focus-visible` outline
  `2px solid var(--green)`, `outline-offset:2px`.
- `.score-hint`: 12 px, `--green-dark`, `margin-left:auto` in its flex row.

### 5.3 The steps (inside the block)

State, held inside the closure:

```js
s = {
  day, facility, num: m.num, m: { ...m },
  expected: { teamCode1: m.t1, teamCode2: m.t2, team1Score: m.t1Score, team2Score: m.t2Score },
  step: opts.readOnlyReason(m) ? 'readonly' : 'enter',
  t1: m.t1Score === null ? '' : String(m.t1Score),
  t2: m.t2Score === null ? '' : String(m.t2Score),
  clear: false, note: '', error: '', conflict: null, reviewShownAt: 0, saves: 0,
}
```

Every step draws the title and sub-title from `opts.describe(s.m)`. Sides are
**always in sheet order**: team 1 left, team 2 right.

**`readonly`.** Body: the two sides and `opts.readOnlyReason(m)`. Actions: **Close**.

**`enter`.** Body: the two sides, each with
`<input class="score-input" type="text" inputmode="numeric" pattern="[0-9]*" maxlength="2" autocomplete="off">`
labelled with the side's name. On `input`, strip non-digits and keep 2;
store into `s.t1`/`s.t2`; enable **Review** only when both are non-empty.
Focus team 1's input on open (text selected). Enter in team 1 focuses team 2;
Enter in team 2 is **Review** when enabled.
Actions: **Cancel**, **Clear score** (only when `m.played`; sets
`s.clear = true`, goes to `review`), **Review →** (primary; `s.clear = false`).

**`review`.** `s.reviewShownAt = performance.now()`. Body:
- not a clear, scores differ: `Winner: <higher side's name>` (`.score-winner`;
  add its players in the normal weight), then `t1 – t2` (`.score-big`), then
  each side's name under its number;
- a tie: `Tied — neither side wins this match` instead of the winner line;
- a clear: `Clear the score of match #${num}` and `Now ${old1} – ${old2}`;
- a correction (not a clear, `m.played`): `Was ${m.t1Score} – ${m.t2Score}` (`.score-was`);
- warnings (`.score-warn`) when any, from `scoreBasicWarnings` then `opts.extraWarnings(m)`:

```js
function scoreBasicWarnings(a, b){
  const hi = Math.max(a, b), out = [];
  if(a === b) out.push('Tied: neither side wins this match.');
  else if(hi < 11) out.push('Neither side reached 11.');
  else if(Math.abs(a - b) === 1) out.push('Won by 1 point.');
  if(hi > 21) out.push('Unusually high: over 21.');
  return out;
}
```

(no warnings for a clear). Actions: **← Back** (to `enter`, values kept)
and the primary **Save t1–t2 to sheet** (or **Clear score**), focused.

**Double-press guard:** a capture-phase `keydown` on the dialog: when
`s.step === 'review'` and `performance.now() - s.reviewShownAt < SCORE_KEY_GUARD_MS`
and the key is Enter or space, `preventDefault()` and `stopPropagation()`.

**`saving`.** Body unchanged; every button disabled; actions show "Saving
to the sheet…". After `SCORE_SLOW_MS`, if still saving, the note reads
"Still waiting on the workbook. Large workbooks can take up to a minute."
The dialog's `cancel` event (Esc) is `preventDefault()`ed while saving and
the × button is disabled. Otherwise Esc and × close.

**`conflict`.** §5.6.

### 5.4 Saving (inside the block)

```js
async function save(){
  const team1Score = s.clear ? null : Number(s.t1);
  const team2Score = s.clear ? null : Number(s.t2);
  const mine = s;
  s.step = 'saving'; s.error = ''; render();
  if(opts.fixture) return fixtureSave(mine, team1Score, team2Score);
  const token = opts.getToken();
  if(!token) return failed(mine, opts.expiredText);
  const url = `${opts.apiBase}/v1/days/${encodeURIComponent(s.day)}/facilities/` +
              `${encodeURIComponent(s.facility)}/matches/${s.num}/score`;
  let res, json = null;
  try {
    res = await fetch(url, {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${token}` },
      body: JSON.stringify({ team1Score, team2Score, expected: s.expected }),
      signal: AbortSignal.timeout(SCORE_SAVE_TIMEOUT_MS),
    });
    json = await res.json().catch(() => null);
  } catch(err){
    if(s !== mine) return;
    return failed(mine, err && err.name === 'TimeoutError'
      ? 'No answer after 2 minutes. The score may or may not be in the sheet: Save again is safe.'
      : opts.unreachable(err));
  }
  if(s !== mine) return;   // closed meanwhile
  handle(mine, res.status, json);
}
```

`failed(s, text)`: `s.step = 'review'`, `s.error = text`, re-render (the
error box shows; Save is enabled again).

`handle(s, status, json)`, with `a–b` the saved scores and the codes from `s.expected`:

| Response | Then |
|---|---|
| 200, `json.sync.ok`, not `json.unchanged` | close; `notify('ok', \`Match #${num} saved: ${code1} ${a}–${b} ${code2}.\`)`; a clear: `Match #${num}'s score cleared.` |
| 200, `json.unchanged`, `sync.ok` | close; `ok`: `Match #${num} already read ${a}–${b} in the sheet.` |
| 200, `!json.sync.ok` | close; `warn`: `Match #${num} is in the sheet, but publishing failed: ${opts.friendlyError(json.sync.error)}. ${opts.resyncHint}` |
| 409 with `json.current` | `s.conflict = json.current`, `s.step = 'conflict'`, re-render |
| 401 | `failed(s, opts.expiredText)` |
| anything else | `failed(s, opts.friendlyError(json && json.error || \`HTTP ${status}\`))` |

No local patching after a real save: the API published before answering, so
the push (or the next poll) brings the new score.

`fixtureSave(s, a, b)`: wait 700 ms. If the page URL has `scoreConflict=1`
and `s.saves === 0`: `s.saves++` and
`handle(s, 409, { error: 'conflict', current: { ...s.expected, team1Score: 11, team2Score: 9 } })`.
Otherwise call `opts.onFixtureSave(s, a, b)`, close, and
`notify('ok', 'Fixture: not sent. The next poll restores the fixture\u2019s scores.')`.

### 5.5 A new snapshot while the dialog is open (`refresh()`)

```js
function refresh(){
  if(!s || !['enter', 'review'].includes(s.step)) return;
  const now = opts.findMatch(s.facility, s.num);
  const e = s.expected;
  let note = '';
  if(!now) note = 'This match is no longer in the published schedule.';
  else if(now.t1 !== e.teamCode1 || now.t2 !== e.teamCode2) note = `This match number now shows ${now.t1} v ${now.t2}.`;
  else if(now.t1Score !== e.team1Score || now.t2Score !== e.team2Score)
    note = now.played ? `The sheet now reads ${now.t1Score}–${now.t2Score}.` : 'The sheet\u2019s score for this match was cleared.';
  if(note !== s.note){ s.note = note; /* update only the note element, never the inputs */ }
}
```

`expected` is never updated here: a save then gets a 409 and the person chooses.

### 5.6 Conflicts

Body:

```
The sheet changed since you opened this match.
It now reads:  NMD_1 11 – 9 NMD_4        (or "no score")
You entered:   NMD_1 11 – 7 NMD_4        (or "clear the score")
```

- Codes unchanged: **Keep the sheet's score** (closes, `notify('ok', \`Match #${num} left as the sheet has it.\`)`)
  and **Replace with yours** (primary): `s.expected = { ...s.conflict }`, then `save()`.
- Codes changed: the body says
  `This match number now belongs to ${c.teamCode1} v ${c.teamCode2}. Close it and open the match again.`
  and the only action is **Close**.

### 5.7 Wiring the dialog into Control Center

After the block (still before `renderIntro();` at load):

```js
// ---- Score entry in Match Finder (spec §5.7) ----
function scoreEntryAvailable(){
  const ev = EVENTS_REGISTRY.get(CURRENT_EVENT_KEY);
  if(!ev || (ev.scoreEntry !== 'console' && ev.scoreEntry !== 'links')) return false;
  return FIXTURE ? true : !!currentAuthToken();
}

function scoreAttrs(m){
  if(!scoreEntryAvailable() || matchByeSide(m) !== null) return '';
  return ` data-score-num="${m.num}" data-score-facility="${escapeHtml(m.facility)}" role="button" tabindex="0"` +
         ` aria-label="${m.played ? 'Edit' : 'Enter'} score for match ${m.num}"`;
}

function scoreHintHTML(m){
  return `<span class="score-hint" aria-hidden="true">\u270E ${m.played ? 'Edit score' : 'Enter score'}</span>`;
}

const scoreDialog = createScoreDialog({ … });   // below
```

`createScoreDialog` options for Control Center:

| Option | Value |
|---|---|
| `dialog` | `document.getElementById('scoreDialog')` |
| `apiBase`, `getToken`, `fixture` | `CLOUD_RUN_BASE_URL`, `currentAuthToken`, `FIXTURE` |
| `describe(m)` | **title** `Match #${m.num}` + ` · ${m.court}` if any + ` · ${m.time}` if any + ` · ${m.facility}` when `FACILITIES.length > 1`. **sub**: team events `${m.matchUp} · ${pairLabel(m.t1, true)}` (skip empty parts); others `${divisionEventLabel(nonByeCode(m))} · ${roundLabel(parseCode(nonByeCode(m)).rest) \|\| 'Round Robin'}`. **side name**: team events `teamNameOf(sideOf(code)) \|\| code`; others the code. **players** `[p1, p2]`, or `null` when `TBD`, with **missing** `'Lineup not set'` (team) or `'To be decided'` |
| `readOnlyReason(m)` | `CURRENT_TYPE !== 'team' && (m.t1p1 === 'TBD' \|\| m.t2p1 === 'TBD') ? 'Players for this match aren\u2019t decided yet.' : null` |
| `extraWarnings(m)` | `'This series game isn\u2019t needed: the series is already decided.'` when `[...unneededSeriesGames(MATCHES)].some(x => x.num === m.num && x.facility === m.facility)`; `'Lineup not set for this match.'` when `CURRENT_TYPE === 'team'` and either side is `TBD` |
| `findMatch(f, n)` | `MATCHES.find(x => x.num === n && x.facility === f) \|\| null` |
| `notify(kind, text)` | `showToast(kind, text, 'score')` |
| `friendlyError`, `unreachable` | `friendlyApiMessage`, `unreachableMessage` |
| `expiredText` | `'Your sign-in expired. Sign in on Mission Control, then save again.'` |
| `resyncHint` | `'Use \u201cResync this day now\u201d on Mission Control.'` |
| `onFixtureSave(s, a, b)` | find the match in `MATCHES` by `s.facility`/`s.num`, set `t1Score = a`, `t2Score = b`, `played = a !== null`; `if(CURRENT_TYPE === 'team') rebuildTeamData();` then `refreshFinder()` |

**Clickable matches (Match Finder only):**

- **`ticketHTML`**: put `${scoreAttrs(m)}` on the outer `<div class="ticket…">`,
  add the class `scoreable` when it's non-empty, and append
  `scoreHintHTML(m)` (when non-empty) at the end of the `meta-row`.
- **Team matchup cards**: give `teamMatchupCardHTML`'s options a
  `scoreable = false` option, passed to `teamMatchupRowHTML(m, swap, scoreable)`.
  There, when `scoreable` and `scoreAttrs(m)` is non-empty, put the
  attributes and `scoreable` class on the outer `<div class="tm-row…">` and
  add `scoreHintHTML(m)` at the end of `tm-row-meta`. Pass `scoreable: true`
  from the **four Match Finder callers only**: `teamResultHTML`,
  `teamPlayerResultHTML`, `allMatchupsHTML` and the team branch of
  `renderMatchByNumber`. The two Standings callers are unchanged.

**Opening:** two delegated listeners on `resultsEl` (beside the existing
team-chip `click` listener, separately):

```js
resultsEl.addEventListener('click', (e) => {
  const el = e.target.closest('[data-score-num]');
  if(!el || e.target.closest('button, a, input')) return;
  openScoreFromFinder(el.dataset.scoreFacility, Number(el.dataset.scoreNum));
});
resultsEl.addEventListener('keydown', (e) => {
  if((e.key === 'Enter' || e.key === ' ') && e.target.matches('[data-score-num]')){
    e.preventDefault();
    openScoreFromFinder(e.target.dataset.scoreFacility, Number(e.target.dataset.scoreNum));
  }
});
function openScoreFromFinder(facility, num){
  const m = MATCHES.find(x => x.num === num && x.facility === facility);
  if(!m){ showToast('error', `Match #${num} isn't loaded. Refresh and try again.`, 'score'); return; }
  scoreDialog.open(DAYS[currentDayIndex].key, facility, m);
}
```

**Other hooks:**
- end of `renderAuthStatus()`: `if(MATCHES.length) refreshFinder();` (the
  hint appears and disappears with sign-in);
- top of `selectEvent()` and `selectDay()`: `scoreDialog.close();`
- end of `loadLiveData()`'s successful branch, after `MATCHES` is reassigned:
  `scoreDialog.refresh();`

### 5.8 Mission Control: **Scorer links**

**Markup.** In `#organizerResults`, after the "Sync method" section and its
`.organizer-divider`, before "Public pages", add a section in the same
pattern:

```html
<!-- Scorer links (control-center-score-entry-spec.md §5.8): a link that can
     enter scores and nothing else, and a switch that stops every such link.
     The switch writes config/events.json (PUT /v1/events/:event/score-entry). -->
<div id="scorerLinksSection" style="display:none;">
  <label class="search-label">Scorer links</label>
  <div class="golive-current" id="scorerLinksCurrent"></div>
  <div class="golive-buttons" id="scorerLinksButtons" style="display:none;">
    <button type="button" class="golive-btn" data-scorer="links">Accepting</button>
    <button type="button" class="golive-btn golive-btn-danger" data-scorer="console">Stopped</button>
  </div>
  <button type="button" id="scorerLinkBtn" class="organizer-secondary-btn" style="display:none;">Issue scorer link</button>
  <div id="scorerLinkResult" class="organizer-result" style="display:none;"></div>
  <p class="organizer-hint">A scorer link opens this event's scorer page, where staff enter scores for any venue on that day and do nothing else. Each link works for 24 hours, and only while this is set to <b>Accepting</b>. <b>Stopped</b> blocks every scorer link at once; operators can still enter scores in Match Finder. Takes effect within about a minute.</p>
  <div class="organizer-divider"></div>
</div>
```

**`renderScorerLinks()`**, called at the end of `renderAuthStatus()` and of
`renderOrganizerStatus()`:

- `mode = EVENTS_REGISTRY.get(CURRENT_EVENT_KEY)?.scoreEntry`.
- Section hidden when `mode` is neither `"console"` nor `"links"`.
- State line: `"links"` → **Accepting**: anyone with a scorer link for the
  day can enter scores. `"console"` → **Stopped**: scorer links can't save
  scores or be issued. Operators still can, in Match Finder. Not signed in
  (and not a fixture): append "Sign in to change this or issue a link."
- Buttons shown only when signed in (or a fixture); `.active` on the one
  matching `mode` (as `renderSyncMethod` does).
- **Issue scorer link** shown only when signed in (or a fixture), `mode === "links"`
  and a day is selected; its label is `Issue scorer link for ${day.label}`.

**Switch click** (`scorerLinksButtons` delegated `click` on `[data-scorer]`):
no confirm. Disable both buttons; `PUT ${CLOUD_RUN_BASE_URL}/v1/events/${encodeURIComponent(CURRENT_EVENT_KEY)}/score-entry`
with the bearer token and `{ mode }`. On 200: set
`EVENTS_REGISTRY.get(CURRENT_EVENT_KEY).scoreEntry = body.scoreEntry` (the
page's copy of `events.json` would otherwise stay stale until reload), hide
`scorerLinkResult` when stopping, `renderScorerLinks()`, and
`showToast('ok', mode === 'links' ? 'Scorer links accepted again. Takes effect within about a minute.' : 'Scorer links stopped. Takes effect within about a minute.', 'scorerlinks')`.
On error: `showToast('error', \`Couldn't change scorer links: ${friendlyApiMessage(...)}\`, 'scorerlinks')`
(+ " Sign in again." on 401). Network error: `unreachableMessage(err)`.
Fixture: no request; set the registry value, re-render, toast `Fixture: not sent.`

**Issue click:** mirror `attConsoleIssueDeskLink`: guard on day and sign-in,
remember `resultEpoch`, disable the button, `POST ${CLOUD_RUN_BASE_URL}/v1/days/${encodeURIComponent(day.key)}/scores/scorer-links`
with the bearer token. Errors as above with "Couldn't issue a scorer link:".
If `resultEpoch` moved on meanwhile: `warn` toast, as attendance does. Then
`renderScorerLink(day, issued)`, a copy of `attConsoleRenderDeskLink` with:

- url `https://sage-match-control.github.io/events/${CURRENT_EVENT_KEY}/scorer?scorer=${issued.token}`
  (on a fixture: `${location.origin}/events/${CURRENT_EVENT_KEY}/scorer.html?scorer=${token}&fixture=${encodeURIComponent(FIXTURE)}`);
- title `Scorer link for ${day.label}`, input `aria-label` `Scorer link`,
  Share title `Scorer · ${day.label}`, toast key `'scorerlinks'`;
- the validity line `Valid until ${weekday, month day, h:mm AM/PM} (24 hours)`,
  formatted in `Asia/Manila`;
- QR through `attConsoleShowQr(url, holder, qrBtn, 'scorer link')`: give that
  function a fourth parameter `label = 'desk link'` used in its canvas
  `aria-label` (`QR code for the ${label}`). Attendance's calls are unchanged.

Fixture issue: no request; a token whose payload is real, so the scorer page
can decode it:

```js
const b64url = s => btoa(s).replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');
issued = { token: b64url(JSON.stringify({ exp: Date.now() + 864e5, scope: 'score-desk', day: day.key })) + '.fixture',
           expiresAt: Date.now() + 864e5, day: day.key };
```

**Fixtures:** in `_fixtures/config.json`, add `"scoreEntry": "links"` to both
fixture events (after `"attendance"`), keeping the file's 2-space formatting.

---

## 6. The scorer page (`_templates/scorer/scorer.html`)

A per-event page, made from this new template for an event with
`"scoreEntry": "links"`: copied to `events/<event-key>/scorer.html` with two
tokens replaced, `{{EVENT_KEY}}` and `{{EVENT_TITLE}}`, exactly like the
attendance desk page. Linked from nowhere public. **Don't instantiate it for
any existing event** (Piggleball and PickleDrive are finished); for testing,
§7 uses git-ignored `scorer-beta.html` copies.

### 6.1 What's in the file, top to bottom

1. **Shell:** start from `_templates/attendance/attendance.html`'s `<head>`,
   header and base styles (the `:root` tokens, fonts, the `.event` header with
   `{{EVENT_TITLE}}` and a `#dayLabel` span, the message line). Title
   `{{EVENT_TITLE}} — Scorer`. Leave out everything attendance-specific,
   including the `ATTENDANCE CLIENT` block. Make sure the `:root` tokens the
   §5.2 styles use exist (copy any missing ones from Control Center's `:root`).
2. **Body:** `#scorerMessage` (role `status`), `#facilityPicker`,
   `#scorerFilters`, `#scorerCount`, `#scorerList`, then the §5.2 dialog
   markup, and a toast area (a fixed `#scorerToast` element at the top; one
   message at a time, `ok` 4.5 s, `warn` 7 s, `error` 9 s, using the toast colours).
3. **Script**, in this order:
   - constants: `EVENT_KEY = '{{EVENT_KEY}}'`, `CLOUD_RUN_BASE_URL`,
     `GHPAGES_OWNER`, `GHPAGES_REPO`, `FIXTURE`, as the attendance desk page
     has them; `POLL_INTERVAL_MS = 10000`, `FETCH_TIMEOUT_MS = 8000`;
     `SCORER_STORAGE_KEY = \`sage.scorer.${EVENT_KEY}\``;
   - the **`LIVE CHANNEL`** block, copied byte for byte from `tools/control-center.html`;
   - copies (not shared blocks) of Control Center's `parseCSV`,
     `rowsToMatches`, `BYE_RE`, `sideIsBye`, `matchByeSide`, `SERIES_GAME_RE`,
     `seriesGameOf`, `seriesGroups`, `walkSeries`, `unneededSeriesGames`,
     `HTTP_WORDS`, `httpWords`, `messageFromJson`, `friendlyApiMessage` and
     `escapeHtml`, each unchanged, under a comment naming where they're copied from;
   - the **`SCORE CLIENT`** block, byte for byte;
   - the page code (§6.2–6.5).

### 6.2 Start-up

Mirror the desk page's `deskStart`:

1. **Token:** `?scorer=<token>` → `localStorage[SCORER_STORAGE_KEY] = JSON.stringify({ token })`,
   then remove `scorer` from the address bar with `history.replaceState`
   (keep other parameters, e.g. `fixture`). Read the token back from storage.
   A newer link replaces an older one.
2. **Decode** the payload for display only (like `deskDecode`), requiring
   `scope === 'score-desk'`, a string `day` and a number `exp`. No token or a
   bad one → "Ask the operator for a scorer link." and stop. Expired
   (`Date.now() >= exp`) → "This scorer link has expired. Ask the operator for
   a new one." and stop.
3. **Config:** fetch `events.json` (fixture: `/_fixtures/config.json`) as the
   desk page does. `entry = config.events[EVENT_KEY]`. No entry, or
   `entry.scoreEntry !== 'links'` → "Scorer links are stopped for this event.
   Ask the operator." and stop (re-checked on every reload).
   `dayEntry = entry.days[info.day]`; missing → "This scorer link doesn't
   match a day of this event. Ask the operator for a new one."
4. `#dayLabel` ← ` · ${dayEntry.label}`, plus ` · ${venue}` once a venue
   is chosen (updated on every switch). `facilities` = `dayEntry.facilities`
   with a non-empty `sheetId`.
5. Build the dialog with `createScoreDialog` (§6.5), start data loading (§6.3),
   draw the picker and list (§6.4).
6. A timer checks `exp` every minute; once past it, close the dialog, empty
   the list, and show the expired message.

### 6.3 Data

Wire the `LIVE CHANNEL` block the way `_templates/standard-tournament-template/index.html`
does: `createLiveChannel({ enabled: !!LIVE_BASE_URL && !FIXTURE, onSnapshot: () => loadData() })`,
a `fetchDaySnapshot(dayKey)` that prefers `liveChannel.cached`, else fetches
`https://${GHPAGES_OWNER}.github.io/${GHPAGES_REPO}/${EVENT_KEY}/data/${day}.json?t=…`
(fixture: `/_fixtures/${EVENT_KEY}/${FIXTURE}.json?t=…`) with
`FETCH_TIMEOUT_MS`, then `liveChannel.newer(...)`; `liveChannel.follow(EVENT_KEY, info.day)`;
poll every `POLL_INTERVAL_MS`; pause the channel and the poll while
`document.hidden`, resume and load at once when visible.

`loadData()` keeps the snapshot it got in `lastSnapshot` (so a venue switch
redraws from it at once, §6.4), then:
- for the **selected facility** only: `matches = rowsToMatches(parseCSV(f.matchesCsv)).map(m => ({ ...m, facility: f.name }))`,
  dropping byes (`matchByeSide(m) !== null`);
- team events: `teamNames[code] = name` from that facility's `standingsCsv`
  (`teamCode`,`teamName` columns, trimmed), as the desk page does;
- a 404 snapshot: "This day's schedule isn't published yet." (keep polling);
  other errors: "Couldn't load the schedule (…). Retrying." (keep polling);
- then redraw the list and call `scoreDialog.refresh()`.

### 6.4 Facility picker, filters, list

- **Facility picker** (`#facilityPicker`): one button per facility, the
  chosen one `aria-pressed="true"`, labelled "Venue". Remembered in
  `localStorage[\`${SCORER_STORAGE_KEY}.facility\`]`.
  - **One facility:** chosen automatically; the picker shows just that
    venue's name, with no buttons.
  - **Several, none remembered** (or the remembered one isn't on this day):
    "Which venue are you at?" above the buttons, and no list until one is picked.
  - **Several, one chosen:** the picker **stays visible above the filters
    at all times**, as a row of buttons (wrapping on a narrow screen), so the
    scorer can switch venue whenever they need to. Never collapse it into a
    menu or hide it behind a setting.
  - **Switching** needs no new link (the link covers every venue that day)
    and no confirmation. It saves the choice, restores that venue's own
    remembered filters (or the defaults), shows "Loading <venue>…", and
    redraws the list from the snapshot already loaded (one snapshot holds
    every facility, so no extra fetch is needed). The title line shows the
    current venue: `{{EVENT_TITLE}} · <day> · <venue>`.
  - The dialog is modal, so the picker can't be used while a score is
    being entered or saved.
- **Filters** (`#scorerFilters`), remembered per facility in localStorage:
  a search box (a match number, `#42`, a team code, a team name or a player
  name; case-insensitive substring), court chips ("All courts" plus each
  distinct `m.court`, sorted by its number), and a **Hide scored** checkbox,
  on by default.
- **Count** (`#scorerCount`): `${played} of ${total} scored` for the facility.
- **List** (`#scorerList`): the filtered matches by match number. Each is a
  card with `data-score-num`, the `scoreable` class, `role="button"`,
  `tabindex="0"`: first line `#${num} · ${time} · ${court}`; then the two
  sides in sheet order (team events: team name, code chip, players or
  "Lineup not set"; others: code and players or "To be decided"); on the
  right, `a–b` when played, else `–`. Nothing left: "No matches to score
  here." (or "Every match here is scored." when *Hide scored* hid them all).
  Use the dialog's colours and tokens; cards at least 56 px tall; no
  horizontal scroll at 375 px.
- Click or Enter/Space on a card → `scoreDialog.open(info.day, facility, m)`.

### 6.5 The dialog on the scorer page

`createScoreDialog` options:

| Option | Value |
|---|---|
| `dialog`, `apiBase`, `fixture` | `document.getElementById('scoreDialog')`, `CLOUD_RUN_BASE_URL`, `FIXTURE` |
| `getToken` | `() => token` (the stored scorer token) |
| `describe(m)` | **title** `Match #${m.num}` + court + time; **sub** team events `${m.matchUp}`, others the code's category (`m.t1` up to its first `_`; dual meet: the middle segment) and `Round Robin` or the stage; **side names** team events `teamNames[code.split('_')[0]] \|\| code`, others the code; **missing** as in §5.7 |
| `readOnlyReason(m)` | `entry.type !== 'team' && (m.t1p1 === 'TBD' \|\| m.t2p1 === 'TBD') ? 'Players for this match aren\u2019t decided yet.' : null` |
| `extraWarnings(m)` | the series and lineup warnings of §5.7, against this page's `matches` and `entry.type` |
| `findMatch(f, n)` | from this page's current `matches` |
| `notify` | the page toast |
| `friendlyError`, `unreachable` | the copied `friendlyApiMessage`; `err => \`Couldn't reach the server. Check this device's connection and try again. (${err && err.message \|\| err})\`` |
| `expiredText` | `'This scorer link has expired. Ask the operator for a new one.'` |
| `resyncHint` | `'Tell the operator so they can resync.'` |
| `onFixtureSave` | patch this page's match, redraw |

A **403** shows the API's message in the dialog's error box ("Scorer links
are stopped for this event. Ask the operator." / "This scorer link is for
another day…"): the generic "anything else" row of §5.4 already does that.

---

## 7. Checking it yourself

Python isn't installed on this machine. From `D:\Personal\SAGE` run
`npx --yes http-server sage-match-control.github.io -p 8123 -c-1` (a dev tool
fetched by npx, not a project dependency).

For the scorer page, make two git-ignored test copies of the template, with
the tokens replaced: `events/pickledrive-anniversary-2026/scorer-beta.html`
(`pickledrive-anniversary-2026`, its title) and
`events/attendance-demo-2026/scorer-beta.html` (`attendance-demo-2026`,
"Attendance Demo"). `**/*beta*` keeps them out of git; leave them in place.

**Control Center** — `http://localhost:8123/tools/control-center.html?fixture=pre`,
**Attendance Demo**, then repeat with **PickleDrive** and `?fixture=finished`.
Desktop window and 375 px wide:

- [ ] Match Finder tickets show the pencil hint; Live Matches and Standings rows don't.
- [ ] Team event: match lines in Match Finder's matchup cards are clickable; the same cards on Standings are not.
- [ ] Byes aren't clickable. A standard playoff match with `TBD` players opens read-only.
- [ ] Enter → Review → Save works with mouse and with keyboard only; a fast double Enter from team 2's input stops on Review.
- [ ] Winner line, tie wording, each warning, "Was" on a correction, Clear score.
- [ ] `&scoreConflict=1`: the first save shows the conflict panel; Replace saves.
- [ ] Esc and × close the dialog; switching day closes it.
- [ ] Mission Control shows **Scorer links** with Accepting active; Stopped flips it, hides **Issue scorer link**, and toasts; Accepting brings it back.
- [ ] **Issue scorer link** shows the link, Copy, Share (where supported), Show QR (a QR with the right aria-label), and "Valid until … (24 hours)".
- [ ] At 375 px the dialog is a bottom sheet and nothing scrolls sideways.
- [ ] No console errors.

**Scorer page** — take the fixture link from **Issue scorer link**, replace
`scorer.html` with `scorer-beta.html`, open it:

- [ ] The token leaves the address bar; a reload still works (from storage).
- [ ] Attendance Demo: the facility picker asks for a venue; the choice is remembered on reload.
- [ ] With a venue chosen, the picker is still visible; switching to the other venue shows that venue's matches and its own filters, and switching back restores the first venue's filters. The title line names the current venue.
- [ ] Court chips, search (number, `#42`, a name) and Hide scored filter the list; the count is right.
- [ ] A card opens the same dialog; save (fixture) updates the card; `&scoreConflict=1` works.
- [ ] PickleDrive: team names and "Lineup not set" show; the lineup warning appears.
- [ ] Set the fixture event's `"scoreEntry"` to `"console"` in `_fixtures/config.json` and reload: the stopped message. Put it back to `"links"`.
- [ ] A link whose `exp` is in the past (build one in the console with the §5.8 fixture recipe and `exp: Date.now() - 1000`) shows the expired message.
- [ ] 375 px wide: no sideways scroll; the dialog is a bottom sheet.
- [ ] No console errors.

Finally: `diff` the `SCORE CLIENT` block (from its first `// ==== SCORE CLIENT`
line to `// ==== END SCORE CLIENT ====`) between Control Center, the template
and both beta copies, and the `LIVE CHANNEL` block between Control Center and
the template. Both must be identical.

Do **not** test against the production API.

---

## 8. Documentation

All present tense; describe what exists.

| File | Add |
|---|---|
| `sage-docs/docs/features/control-center.md` | Under `## Match Finder`, `### Entering a score`: who can (signed-in operators, events with score entry on), clicking a match, the review step and why it names the winner, corrections and clearing, conflicts and the two choices, the publish warning and **Resync this day now**. Under `## Mission Control`, `### Scorer links`: the Accepting/Stopped switch, issuing a link (24 hours, one day, every venue), Copy/Share/QR |
| `sage-docs/docs/features/running-an-event-day.md` | Under `## Before doors open`: issue scorer links for the day if scorers use them. Under `## During play`: scores can be entered in the sheet, Control Center, or scorer links, all writing the same cells; stopping scorer links |
| `sage-docs/docs/features/scorer-page.md` (new; linked from `features/README.md` and `mkdocs.yml`) | The scorer page for scorer staff: opening the link, picking a venue and switching to another one later (no new link needed), filters, entering a score, conflicts, what "stopped" and "expired" mean |
| `sage-docs/docs/technical/control-center.md` | `## Score entry`: the `SCORE CLIENT` block and its options, `scoreEntryAvailable`, why only Match Finder, the dialog states, why it copies the match, `refresh`, fixtures and `scoreConflict=1`, the Scorer links section and its fixture behaviour |
| `sage-docs/docs/technical/scorer-page.md` (new; linked from `technical/README.md` and `mkdocs.yml`, cross-linked with the feature page) | The template and its tokens, start-up checks, token storage, data loading through the live channel, the copied helpers, the shared blocks |
| `sage-docs/docs/technical/sync-pipeline.md` | `## Score entry writes`: the API writes `SCHEDULE` and calls `syncDay` itself because API writes fire no onEdit trigger; the optimistic check and its accepted window; the 60 s timeout and why |
| `sage-docs/docs/technical/auth.md` | Scorer tokens beside desk tokens: scope, derived key, 24-hour expiry, what they allow, how the switch stops them |
| `sage-docs/docs/technical/event-data-config.md` and `event-data/config/README.md` | `scoreEntry`: optional, `"console"` (operators) or `"links"` (also scorer links); absent = off; anything else fails validation; Mission Control's switch moves it between the two; every facility workbook must be shared with the service account as Editor |
| `sage-docs/docs/technical/architecture.md` | `src/scores/` in its module list; the `SCORE CLIENT` block in its "kept in sync by hand" list |
| `sage-match-control.github.io/_templates/CLAUDE.md` | A new step after step 11: "**Add the scorer page, if the event uses scorer links.** For `"scoreEntry": "links"`, copy `_templates/scorer/scorer.html` to `events/<event-key>/scorer.html` and replace `{{EVENT_KEY}}` and `{{EVENT_TITLE}}`. Linked from nowhere public; Mission Control's **Issue scorer link** produces the link." Renumber the steps after it. The service-account sharing step covers "any `attendance` or `scoreEntry` setting". §5.1's table gains a `SCORE CLIENT` row (Control Center, the scorer template and every event's `scorer.html`: byte-identical) and the `LIVE CHANNEL` row names the scorer pages too |
| `D:\Personal\SAGE\CLAUDE.md` | (1) Layout: a `src/scores/` entry beside `src/attendance/`; the attendance entry's "writes … **only** inside `ATTENDANCE!A:G`" adds "and, for score entry, one match's two `SCHEDULE` score cells"; `src/auth/` mentions scorer tokens. (2) Endpoints: the three new routes, one line each. (3) "Things that must be kept in sync by hand": a `SCORE CLIENT` bullet (Control Center, `_templates/scorer/scorer.html`, every `events/<key>/scorer.html`); the `LIVE CHANNEL` bullet adds the scorer pages; the played/BYE and series bullets add the scorer pages' copies. (4) Site section: `_templates/scorer/` and `events/<key>/scorer.html` |

Do **not** move this spec to `implemented/` and do not edit its status line:
the owner does that after reviewing and running §11.2.

---

## 9. What doesn't change

`sheets-sync.gs` and every `.gs` file, the generators, typing scores in the
sheet, Live Matches, Standings, Awards, attendance and its desk links, the
snapshot format, the live Worker, the real `events.json`, every existing
route's behaviour.

---

## 10. Alternatives considered (for the record; don't build these)

| Alternative | Why not |
|---|---|
| An Apps Script web app per workbook doing the write | Could take the document lock, but means a script and a deployment URL per workbook; attendance moved away from that model |
| Write, and let `sheets-sync.gs` publish | API writes fire no trigger; nothing would publish until the next hand edit |
| Respond first, publish after | Cloud Run throttles CPU after the response |
| Inline score inputs inside the tickets | Every poll redraws Match Finder and would wipe them mid-typing |
| A `window.confirm` instead of a review step | Repeats nothing the person can check; swapped scores get through |
| Scorer links that open Control Center with fewer tabs | Scorers would hold a page built for operators; a separate page keeps them out of it entirely |
| Revoking one link at a time | Needs server-side state per link; the switch plus 24-hour expiry covers the need |

---

## 11. Finishing

### 11.1 Report back

When done, reply with:

1. `git status --short` from each of the four repos.
2. `npm run verify`'s summary line from `sage-tools-api`.
3. The §7 checklists with each item ticked or explained, and the two `diff` results.
4. Anything in this spec you found wrong or had to interpret, with what you did.

### 11.2 The owner's checks (not yours)

After review, the owner deploys the change to a **no-traffic** Cloud Run
revision (`--no-traffic --tag score-entry`), adds a scratch test event with
`"scoreEntry": "links"` whose facilities are **copies** of each master,
instantiates its `scorer.html`, and points a local Control Center and scorer
page at that revision's URL.

| # | Check | Pass |
|---|---|---|
| C1 | Save a score on a Dual Meet Master copy, a Standard Master copy and a team workbook copy | numbers appear in the right `SCHEDULE` cells; category tab and `STANDINGSCSV` update; the snapshot carries them; **the first save of a blank match does not 409** (the published codes and blanks equal the sheet's) |
| C2 | The copy's Apps Script **Executions** page after C1 | no `sheets-sync.gs` run caused by the API write |
| C3 | Clear a score in the team workbook copy | `=ISBLANK(<cell>)` is TRUE; the matchup goes back to unplayed |
| C4 | Type a different score in the sheet while the dialog is on Review, then Save | the conflict panel, showing the typed score |
| C5 | Remove the service account from one copy, then save | the error names the account to share with |
| C6 | Issue a scorer link, save from the scorer page on a phone | saved; the Cloud Run log line says `by=scorer` |
| C7 | Set **Stopped**, wait a minute, save from the scorer page; then **Accepting** and save again | 403 "Scorer links are stopped…", then saved |
| C8 | Time from Save to the public page updating, standard and team | recorded in `technical/sync-pipeline.md`; no pass mark |

Then the owner sets `"scoreEntry"` on a real event and instantiates its scorer page.
