# Spec — Court Control from Control Center

> **Status: in progress.** Built and live on 2026-11-14 (`sage-tools-api` 3.8.0, the site's `lib/v1/` and
> Control Center, and the docs); the owner's checks K1–K6 (§10.2) on workbook copies are left, and it moves
> to `implemented/` once they pass. Written 2026-10-05 as an outline. Rewritten on 2026-11-14 as a
> build spec, against `sage-tools-api` 3.7.1, the site engine (`lib/v1/`), both workbook
> generators, and the `Court Control` tabs of five real workbooks (ACSO and UPAP, standard;
> PNF x BUP, dual meet; Ladies' Dink and Millennium, team). **Every decision is settled (§1).** There are no open questions: do
> not stop to ask about anything §1 covers. §11 (the score dialog's "next match" prompt)
> is a later phase, outlined only. Do not build it.
>
> Builds on [Score entry from Control Center](../implemented/control-center-score-entry-spec.md),
> whose service account, write path, conflict check and dialog this reuses. Read that
> spec's §3–§5 before starting. This one assumes them.

Let a signed-in operator put a match on a court, replace it, or clear the court, from
Control Center's **Live Matches** tab, without opening the facility workbook.

---

## 0. Read this first

This spec is written for an implementer with no other context. Do the sections in order:
the API (§4), then the site (§5 to §7), then the docs (§8). Check each part as §9
says. Report as §10 says.

### 0.1 The system in one paragraph

Each tournament day has one Google Sheet ("facility workbook") per venue. Operators type
into it. An installable `onEdit` trigger (`apps-script/sheets-sync.gs`) then asks the
Cloud Run service `sage-tools-api` to re-read the workbook's `CSV` and `STANDINGSCSV`
tabs and publish a JSON snapshot of the day. Every page reads that snapshot. The pages
are the public Tournament Hub, the schedule board, and the operator console **Control
Center** (`sage-match-control.github.io/tools/control-center.html`). A snapshot is
published through a Cloudflare Worker that pushes it to open pages over a WebSocket,
and it is archived to GitHub. One tab of the workbook, **Court Control**, says which
match is on which court right now. Its values become each match's `court` column in
`CSV`. On the site that column is `liveCourt`, and it is what makes a match show as
**live** everywhere. Today an operator types match numbers into `Court Control` by hand.
This spec lets Control Center write that cell instead, through a new API route.

### 0.2 Where things are

| Path (from `D:\Personal\SAGE`) | What |
| --- | --- |
| `CLAUDE.md` | the workspace rules. Read it all |
| `sage-tools-api/` | the API (Node 22, ESM, Express, `node:test`). Its `README.md` **Testing** section has the "change → test" table you must follow |
| `sage-tools-api/src/scores/` | score entry, the feature this copies: `ScoreService.mjs`, `routes.mjs`, `domain/scheduleGrid.mjs`, `domain/scoreRequest.mjs` |
| `sage-tools-api/src/clients/SheetsClient.mjs` | every Google Sheets read and write, each write behind an allowlist |
| `sage-tools-api/src/shared/errors.mjs` | the `AppError` classes. An error carries its status and `code` |
| `sage-tools-api/src/app.mjs` | the only place services and clients are constructed (`createApp`, `createRouters`) |
| `sage-match-control.github.io/tools/control-center.html` | Control Center, one file |
| `sage-match-control.github.io/lib/v1/` | the site engine: `domain/` (pure rules), `views/` (HTML builders, dialogs), `css/` |
| `sage-match-control.github.io/_templates/CLAUDE.md` §8 | the engine's compatibility rule, in brief |
| `sage-match-control.github.io/_tests/` | the site's unit tests and the comparison harness (`npm run verify`) |
| `sage-docs/docs/` | the docs. `technical/api.md` holds the REST conventions |

### 0.3 Hard rules

1. **Branches, never pushes.** Work on a branch named `court-control` in
   `sage-tools-api`, `sage-match-control.github.io` and `sage-docs`, and commit there.
   Commit the workspace `CLAUDE.md` change (§8) on a `court-control` branch of the
   workspace repo (`D:\Personal\SAGE`). Never push, merge, deploy or run
   `wrangler`. The owner does all of that (§10.2).
2. **Never call the production API**
   (`https://sage-tools-api-811926984834.us-central1.run.app`) and never touch a real
   workbook. Every test uses the fakes. The real-Google checks are the owner's (§10.2).
3. **API: test first.** For every production change, write the test, watch it fail,
   then make it pass (`sage-tools-api` rule). Never weaken or delete a test to make a
   change pass.
4. **`lib/v1` changes are compatible only.** You may add a file, an export, an option
   or a CSS class. You may never remove or rename an export, a CSS class a page uses, or
   a theme property. Without the new option, an existing function must return
   byte-for-byte what it returns today.
5. **Do not change:** any `apps-script/*.gs` file, the generators, `sheets-sync.gs`, the
   live Worker (`live-worker/`), the snapshot format, `events.json`'s shape, the score
   routes, the scorer page (`_templates/scorer/`, `events/*/scorer.html`), the
   attendance code, any `/v1` or legacy route, the Hub, or the schedule board.
6. **Write docs in the present tense** (workspace `CLAUDE.md`). Describe what the system
   does, never what it used to do.

### 0.4 Facts checked against the code and workbooks (2026-11-14)

These are the facts the design rests on. If one is no longer true when you start, stop
and report it (§10.1 item 4).

1. **Court Control's shape differs between workbooks.** Each court is a 3-row block.
   Column B holds the court's label, merged over the block's 3 rows. Column C, on the
   block's first row, holds the match number typed by hand. D and E are formulas that
   read the match, and with C blank they show `#REF!` or `-`. Where the blocks start
   is **not** the same everywhere:

   | Workbook | Header row (`Court`, `Match`) | First court row | B holds |
   | --- | --- | --- | --- |
   | Standard (ACSO, UPAP; `standard-tournament-generator.gs`) | 4 | 5 | `=Variables!R<k+1>`, the sidebar's court labels (`1`, `2`, … or `5,6,7…`) |
   | Dual meet (PNF x BUP) | 5 | 6 | the court number |
   | Team (Ladies' Dink, Millennium) | 5 | 6 | the court number |

   In the Ladies' Dink workbook, courts 7–12 hold the text `#` in C. Below the courts,
   B also holds non-court labels with input cells beside them: `Next Stand By` (row 50),
   `TBA Stand By` (53) and `Match to Record` (56, 59). The PNF workbook has
   counters in `B3:C3` (`0`, `0`). So **the API never assumes a row.** It finds a
   court's row by the number in B (§4.3), starting at row 4.
2. **The live court is B's value.** `CSV`'s `court` column is
   `FILTER('Court Control'!B:B, 'Court Control'!C:C = matchNumber)`. On the site,
   `lib/v1/domain/matches.js` reads that column as `liveCourt`. `CourtAssignment` is
   the scheduled court, a different field (`m.court`, such as `Court 3`).
3. **Live Matches keys a court by its number.** `buildFacilityCourtGroups` in
   `lib/v1/views/live-matches.js` makes one row per court that appears in the facility's
   scheduled matches, keyed by the digits of `CourtAssignment` (`courtNumberFrom`). It
   adds any court that has a `liveCourt`, keyed by that raw value. A court with no
   scheduled match gets no row. A court whose label has no digit gets a row only while
   occupied.
4. **Three builders draw a court cell, each as one `<td class="live-court-cell" …>`:**
   `liveTableRowsHTML` (occupied), `liveEmptyRowHTML` (idle, class `live-court-cell idle`)
   in `views/live-matches.js`, and `liveRowHTML` in `views/teams.js` (a team event's
   occupied row, which Control Center passes as `options.liveRow`). Each court's HTML holds
   exactly one such `<td>`.
5. **Control Center always shows Live Matches.** `consoleDayIsLive()` returns `true`,
   so there is no go-live gate. `drawLiveMatches()` (around line 2309) calls
   `Live.renderLiveMatches(MODEL, {…})` on every new snapshot. `scoreDialog.refresh()` is
   called right after it (around line 1595).
6. **`MATCHES` and `MODEL.matchesByFacility` hold different objects.** `MATCHES` (=
   `model.matches`) holds copies that carry `facility`. `matchesByFacility` holds the
   facility's own objects, which have no `facility`. A fixture-mode patch must update
   both (§7.4).
7. **An API write fires no `onEdit`.** So the API publishes in the same request through
   `syncService.syncDay`, as `ScoreService.submit` does (score entry's D4).
8. **Score entry gates on `scoreEntryAvailable()`** in Control Center: the event's
   `scoreEntry` is `"console"` or `"links"`, and the operator is signed in (or `FIXTURE`
   is set). It also draws a `✎ Enter score` hint (`.score-hint`).
9. **Speed.** A score save at ACSO (13 Nov 2026) took 7.0 s typically, most of it the
   publish. A court write takes the same path without the cell note: about 5 s from
   Save to the pages updating. A hand edit in the sheet takes about 2–4 s.

---

## 1. Decisions (settled with the owner, 2026-11-14)

| # | Decision |
| --- | --- |
| D1 | **Operators only.** Only a signed-in operator token may set a court. Scorer and desk tokens get `401`. The scorer page gets nothing |
| D2 | **No new switch.** Court control is on exactly when score entry is: the event's `scoreEntry` is `"console"` or `"links"` and the event is not archived. Otherwise the API answers `403 court_control_off` |
| D3 | **A court is found by the number in its label**, its first run of digits (`"Court 3"` and `3` are court 3). This is the rule Live Matches already keys by. A court whose label has no digit cannot be set from the console |
| D4 | **No `GET`.** The route is a `PUT` alone, with an exceptions-table row, as the score route has. The published snapshot already carries every live court |
| D5 | **The API refuses, the console warns.** The API refuses: an unreadable or unsafe cell (a formula, an error, text), a match not in `SCHEDULE`, a BYE match, a match already on another court, and a changed cell (`409`). Everything else is a warning in the dialog, and the operator may go ahead anyway |
| D6 | **Moving a match between courts is two actions:** clear the old court, then set the new one. The API never writes two cells in one request |
| D7 | **The button is the court cell.** In Live Matches, the court's own cell (the number at the left of the row) is the button, with a small `✎ Set match` (idle) or `✎ Change` (occupied) label under the number. There is no separate pencil icon. It appears only where score entry would (fact 8) |
| D8 | **Publish in the same request, even when unchanged**, as score entry does. A retried save may have written without publishing |
| D9 | **No cell note.** The cell is overwritten many times a day. The API logs one `court` line per request instead |
| D10 | **The score dialog's "Court 3 is free: put #57 on it?" prompt is a later phase** (§11). This build does not touch `views/score-dialog.js` |

---

## 2. Overview

```
Live Matches court cell ("✎ Set match")
  -> court dialog: Pick (suggestions, or a typed number) -> Review (warnings) -> Save
  -> PUT /v3/days/{day}/facilities/{facility}/courts/{court}/match
       { matchNumber: 57 | null, expected: { matchNumber: 42 | null } }
  -> CourtService.setMatch:
       read Court Control B4:C200 (values + formulas) and SCHEDULE, in parallel
       find the court's row by the number in B; check C; compare with expected (409)
       check the match: in SCHEDULE, not a BYE, not on another court
       write C (RAW number) or clear it; publish via syncService.syncDay; log
  -> 200 { …, sync: { ok } }  -> a toast; the snapshot arrives over the live socket
```

Files added or changed:

| Repo | Added | Changed |
| --- | --- | --- |
| `sage-tools-api` | `src/courts/CourtService.mjs`, `src/courts/routes.mjs`, `src/courts/domain/courtControlGrid.mjs`, `src/courts/domain/courtRequest.mjs`, their tests, `test/integration/court-routes.test.mjs` | `src/clients/SheetsClient.mjs`, `src/shared/errors.mjs`, `src/shared/ports.mjs`, `src/app.mjs`, `package.json`, `README.md`, test helpers and guards (§4.10) |
| site | `lib/v1/domain/courts.js`, `lib/v1/views/court-dialog.js`, `lib/v1/css/court-dialog.css`, `_tests/unit/domain/courts.test.mjs` | `lib/v1/views/live-matches.js`, `lib/v1/css/live-matches.css`, `tools/control-center.html`, `_tests/unit/views/live-matches.test.mjs`, `_tests/engine/cases.mjs`, `_tests/engine/accepted.mjs` |
| `sage-docs` | — | the pages in §8 |
| workspace | — | `CLAUDE.md` |

---

## 3. Finding a court in `Court Control`

The API reads `'Court Control'!B4:C200` twice, in parallel: once with
`UNFORMATTED_VALUE`, because B is a formula in standard workbooks and only its value is
the label, and once with `FORMULA`, so it can tell what C really holds. Row `i` of the
result (0-based) is sheet row `4 + i`. A merged B reads as its value on the block's
first row and blank on the other two, so each court has exactly one row.

- **A court row** is a row whose B value (as text, trimmed) contains a digit. Its
  **court number** is `Number` of the first run of digits. `Court`, `Next Stand By`,
  `Match to Record` and blanks are not court rows.
- **The court's row** is the one court row whose number equals the requested court. If
  there is none, the answer is `404`. If there are two or more, it is `422 court_layout`,
  naming the rows.
- **The match cell** is C on that row, read from the `FORMULA` result:

  | C holds (FORMULA render) | Means |
  | --- | --- |
  | nothing, `""`, or only spaces | empty: `null` |
  | a whole number ≥ 1, or text of digits only | that match number |
  | text starting `=` | a formula: refused, `422 court_layout`, *"Court 3's match cell C14 holds a formula. Set this court in the sheet."* |
  | anything else (`#`, `#REF!`, `12a`) | refused, `422 court_layout`, *"Court 3's match cell C14 holds "#", not a match number. Set this court in the sheet."* |

- **"On another court"**: every *other* court row whose C (by the same table) is the
  requested match number. Rows that are not court rows (`Next Stand By`) are never
  considered.

Worked example (team workbook, fact 1): court 3 is row 12. `'Court Control'!C12` is
written with `57`. In the standard workbook (ACSO), court 3 is row 11.

---

## 4. `sage-tools-api`

A minor version: **3.8.0**.

### 4.1 Files

```
src/courts/
  CourtService.mjs              setMatch
  routes.mjs                    courtV3Routes (one route)
  domain/courtControlGrid.mjs   the §3 rules, pure
  domain/courtRequest.mjs       input validation, pure
test/unit/courts/CourtService.test.mjs
test/unit/courts/domain/courtControlGrid.test.mjs
test/unit/courts/domain/courtRequest.test.mjs
test/integration/court-routes.test.mjs
```

`src/courts/` is a new feature folder. Under the dependency rules it may import
`src/scores/domain/scheduleGrid.mjs` (another feature's `domain/`) and `src/shared/*`,
and nothing else from another feature.

### 4.2 `src/courts/domain/courtRequest.mjs`

```js
import { ValidationError } from "../../shared/errors.mjs";

// "3" -> 3. A court number from the path: 1–999.
export function parseCourtParam(param) { … }
// throws ValidationError("court must be a court number, such as 3")

// The body of PUT …/courts/{court}/match ->
//   { matchNumber: number|null, expected: { matchNumber: number|null } }
export function parseCourtMatchRequest(body) { … }
```

Rules:

- `param` matches `/^[1-9]\d{0,2}$/`.
- `body` is a non-null object, or the error is *"Body must be a JSON object"*.
- `matchNumber` is an integer from 1 to 99999, or `null`. Otherwise:
  *"matchNumber must be a match number, or null to clear the court"*.
- `expected` is a non-null object, or the error is *"expected is required"*.
- `expected.matchNumber` follows the same rule as `matchNumber`:
  *"expected.matchNumber must be a match number, or null"*.
- `matchNumber` may be absent only if the body says so explicitly: `undefined` is an
  error. This means `{}` is a 400, not a clear.

### 4.3 `src/courts/domain/courtControlGrid.mjs`

No imports.

```js
export const COURT_TAB = "Court Control";
export const FIRST_COURT_ROW = 4;
export const LAST_COURT_ROW = 200;
export const COURT_READ_RANGE = `'${COURT_TAB}'!B${FIRST_COURT_ROW}:C${LAST_COURT_ROW}`;
// The one cell shape the API may write: a single C cell at or below row 4.
export const COURT_CELL = /^'Court Control'!C([1-9]\d*)$/;

/** "Court 3" -> 3, 3 -> 3, "12" -> 12, "Next Stand By" -> null, "" -> null. */
export function courtNumberOf(label) { … }

/** "'Court Control'!C12" for row 12. */
export function matchCellRange(row) { … }

/**
 * values: the B4:C200 read (UNFORMATTED_VALUE). -> [{ row, label, court }] for every court row,
 * row being the sheet row (4 + index), label B's value as trimmed text, court its number.
 */
export function courtRows(values) { … }

/**
 * One C cell as read with FORMULA rendering ->
 *   { kind: "empty" } | { kind: "match", matchNumber } | { kind: "formula" } | { kind: "other", text }
 */
export function readMatchCell(raw) { … }

/** Is this a range the API may write? (used by SheetsClient) */
export function isCourtCell(range) { … }   // COURT_CELL and row >= FIRST_COURT_ROW
```

`readMatchCell`:

- `undefined`, `null`, or text that trims to `""` is `empty`.
- An integer ≥ 1 (`typeof === "number"`), or text matching `/^\d+$/` with a value ≥ 1, is
  a `match`.
- Text starting with `=` (after trimming) is a `formula`.
- Anything else is `other`, with `text` set to `String(raw).trim()`.

### 4.4 Errors (`src/shared/errors.mjs`)

Add these classes under a `// Court control (src/courts/)` heading, after the score-entry
errors:

```js
// The event's scoreEntry setting is absent, or the event is archived (D2).
export class CourtControlOffError extends ForbiddenError {
    constructor() {
        super("Court control is off for this event. It follows the event's score entry setting.");
        this.code = "court_control_off";
    }
}

// Court Control is in a state the API won't write into: no tab, a court number on two
// rows, or a match cell holding a formula, an error or text.
export class CourtLayoutError extends AppError {
    constructor(message) { super(message, 422); this.code = "court_layout"; }
}

// The match can't be put on a court: it isn't in SCHEDULE, or it is a BYE.
export class UnplayableMatchError extends AppError {
    constructor(message) { super(message, 422); this.code = "unplayable_match"; }
}

// The court's cell no longer holds what the client showed. current: { matchNumber }.
export class CourtConflictError extends AppError {
    constructor(current) {
        super("The sheet changed since this court was opened", 409);
        this.code = "court_conflict";
        this.current = current;
        this.extra = { current };
    }
}

// The match is already on another court. onCourt: that court's label.
export class MatchOnCourtError extends AppError {
    constructor(matchNumber, onCourt) {
        super(`Match #${matchNumber} is on Court ${onCourt} now. Clear that court first.`, 409);
        this.code = "match_on_court";
        this.onCourt = onCourt;
        this.extra = { onCourt };
    }
}
```

An unknown court is a `NotFoundError` (404, `not_found`): *"Court Control has no court 3."*
`/v3`'s error handler already turns all of these into problem details that carry `extra`.
Check that `extra` reaches the body, as `ScoreConflictError`'s `current` does, and add
it if it doesn't.

### 4.5 `SheetsClient`: two court methods with their own allowlist

In `src/clients/SheetsClient.mjs`, import `isCourtCell` from
`../courts/domain/courtControlGrid.mjs`. A client may import domain files. Then add:

```js
// The only Court Control cell the API may ever change: one match cell in column C, at
// or below row 4 (sage-docs/docs/specs/.../control-center-court-control-spec.md §4.5).
function assertCourtCell(range) {
    if (!isCourtCell(range)) throw new Error(`SheetsClient: write outside the court allowlist: ${range}`);
}

    // One court's match number, as a number (RAW), exactly what typing it stores.
    async writeCourtMatch(sheetId, range, matchNumber) {
        assertCourtCell(range);
        if (!Number.isInteger(matchNumber) || matchNumber < 1) throw new Error(`SheetsClient: not a match number: ${matchNumber}`);
        return this.#batchUpdateValues(sheetId, [{ range, values: [[matchNumber]] }], "RAW");
    }

    // Empties one court's match cell (values:batchClear, as clearScores does).
    async clearCourtMatch(sheetId, range) { assertCourtCell(range); … same body as clearScores … }
```

Update the class's header comment to list this third allowlist. In
`test/unit/clients/SheetsClient.test.mjs`, add tests that:

- accept `'Court Control'!C12` for both methods, with the right request body
  (`valueInputOption: "RAW"`, `values: [[57]]`, or `ranges: [range]`);
- refuse, before any fetch, `'Court Control'!D12`, `'Court Control'!C3`,
  `'Court Control'!C12:C13`, `Court Control!C12` (unquoted), `SCHEDULE!C12` and a
  `matchNumber` of `0`.

### 4.6 Ports (`src/shared/ports.mjs`)

Add:

```js
 * @typedef {object} CourtSheets
 * @property {(...args: any[]) => Promise<any>} readValues
 * @property {(...args: any[]) => Promise<any>} writeCourtMatch
 * @property {(...args: any[]) => Promise<any>} clearCourtMatch
```

### 4.7 `src/courts/CourtService.mjs`

```js
export class CourtService {
    /** @param {{ configStore: RegistryReader, sheets: CourtSheets, syncService: DayPublisher,
     *            logger: Logger, now?: () => Date }} deps */
    constructor({ configStore, sheets, syncService, logger, now = () => new Date() }) { … }

    // actor: { kind: "operator" }. -> the 200 body (§4.8).
    async setMatch({ day, facilityName, court, matchNumber, expected, actor }) { … }
}
```

`setMatch` does these steps in order:

1. `config = await configStore.get()`, then `const { event, facilities } = config.getDay(day)`
   (an unknown day throws, which is a 404 on `/v3`), then
   `const { scoreEntry, archived } = config.getEvent(event)`. If `!scoreEntry || archived`,
   throw `CourtControlOffError`.
2. Find `facility` by exact name, or throw `UnknownFacilityError(facilityName, facilities)`,
   as `ScoreService` does. Use `sheetId = facility.sheetId`.
3. Read in parallel (`Promise.all`):
   `readValues(sheetId, COURT_READ_RANGE, { render: "UNFORMATTED_VALUE" })`,
   `readValues(sheetId, COURT_READ_RANGE, { render: "FORMULA" })`, and, only when
   `matchNumber !== null`, `readValues(sheetId, "SCHEDULE", { render: "UNFORMATTED_VALUE" })`.
   If the first read is `null` (no tab), throw
   `CourtLayoutError("This workbook has no Court Control tab")`.
4. `rows = courtRows(values).filter(r => r.court === court)`. With none, throw
   `NotFoundError(\`Court Control has no court ${court}.\`)`. With more than one, throw
   `CourtLayoutError(\`Court ${court} is on more than one row of Court Control (B${a}, B${b}). Fix the sheet, then try again.\`)`.
   Let `at` be the one row and `cell = \`C${at.row}\``.
5. `cur = readMatchCell(formulas?.[at.row - FIRST_COURT_ROW]?.[1])`. A `formula` or an
   `other` throws `CourtLayoutError` with §3's wording. `previous = cur.kind === "match" ? cur.matchNumber : null`.
6. `unchanged = previous === matchNumber`. If `!unchanged && previous !== expected.matchNumber`,
   throw `CourtConflictError({ matchNumber: previous })`.
7. If `matchNumber !== null && !unchanged`:
   - If SCHEDULE is `null`, throw `ScheduleLayoutError("This workbook has no SCHEDULE tab")`.
   - `hits = findMatchCells(schedule, matchNumber)`. With none, throw
     `UnplayableMatchError(\`Match #${n} isn't in SCHEDULE. Its row may have been moved or cleared.\`)`.
     With more than one, throw `ScheduleLayoutError` worded as `ScoreService`'s.
   - `const { teamCode1, teamCode2 } = readMatchAt(schedule, hits[0])`. If either code
     trims to `bye` (case-insensitive), throw
     `UnplayableMatchError(\`Match #${n} is a BYE: it is never played.\`)`.
   - For every *other* court row `o` (`courtRows(values)` minus `at`), check
     `readMatchCell(formulas?.[o.row - FIRST_COURT_ROW]?.[1])`. If it is a `match` with
     `matchNumber`, throw `MatchOnCourtError(matchNumber, o.label)`.
8. If `!unchanged`: when `matchNumber === null`, call
   `sheets.clearCourtMatch(sheetId, matchCellRange(at.row))`, otherwise
   `sheets.writeCourtMatch(sheetId, matchCellRange(at.row), matchNumber)`.
9. Publish, even when unchanged (D8), exactly as `ScoreService.submit` does:
   `await syncService.syncDay(day, { facilityName, editAt: now().getTime() })`. Map the
   result to `sync`: `{ ok: true }`, or `{ ok: false, error }`. For a facility in
   `facilitiesFailed`, the error is *"The court is set in the sheet, but the workbook
   couldn't be read back to publish it"*. A throw becomes its message.
10. Log one line (D9):
    `court ${day}/${facilityName} ${at.label} ${cell} ${previous ?? "–"} -> ${matchNumber ?? "–"}${unchanged ? " (unchanged)" : ""} by=${actor?.kind} sync=${ok|failed}`.
11. Return §4.8's body.

Never read or write anything else, and never retry a write.

### 4.8 `src/courts/routes.mjs`

```js
export function courtV3Routes({ courtService, auth }) {
    const router = express.Router();
    /** @openapi … (below) */
    router.route("/days/:day/facilities/:facility/courts/:court/match")
        .put(auth.requireOperator, requireJson(), asyncHandler(async (req, res) => {
            const court = parseCourtParam(req.params.court);
            const { matchNumber, expected } = parseCourtMatchRequest(req.body);
            const result = await courtService.setMatch({
                day: req.params.day, facilityName: req.params.facility, court, matchNumber, expected, actor: req.actor,
            });
            res.status(200).json(result);
        }))
        .all(methodNotAllowed);
    return router;
}
```

Copy the imports and the `@openapi` block's form from `src/scores/routes.mjs`'s `/v3`
score route: `tags: [courts]`, `operationId: setCourtMatch`, summary *"Put a match on a
court, replace it, or clear the court"*. The description says what §4.7 does, and that the
route has no `GET` because the published snapshot carries every live court. Responses:

| Status | When (code) |
| --- | --- |
| `200` | the result (below) |
| `400` | a bad court number or body (`validation_error`) |
| `401` | no token, an expired one, or a scorer or desk token |
| `403` | court control is off (`court_control_off`) |
| `404` | unknown day or facility, or no such court |
| `405` | another method, with `Allow: PUT` |
| `409` | `court_conflict` (with `current: { matchNumber }`) or `match_on_court` (with `onCourt`) |
| `415` | not JSON |
| `422` | `court_layout`, `schedule_layout` or `unplayable_match` |
| `502` / `503` | Google failed or is busy |

The `200` body:

```json
{
  "day": "upap-day1", "facility": "The Foundry",
  "court": "3", "cell": "'Court Control'!C11",
  "matchNumber": 57, "previous": { "matchNumber": 42 },
  "unchanged": false,
  "sync": { "ok": true }
}
```

`court` is B's label as text, `matchNumber` what the cell holds now (`null` after a
clear), and `sync` is `{ ok: false, error }` when publishing failed.

Add a `CourtResult` schema beside `ScoreResult` in `src/docs/openapiSpec.mjs`, and
reference it from the `200`.

### 4.9 Wiring (`src/app.mjs`)

After the score service:

```js
    // Court control writes one match number into Court Control as the same service account,
    // then publishes through syncService. Its own SheetsClient, with score entry's long timeout.
    // See sage-docs/docs/specs/.../control-center-court-control-spec.md.
    const courtsLogger = logger.child("courts");
    const courtService = new CourtService({
        configStore: syncConfigStore,
        sheets: new SheetsClient({ auth: sheetsReadAuth, accessToken: googleAccessToken, timeoutMs: 60000, logger: courtsLogger }),
        syncService,
        logger: courtsLogger,
    });
```

Add `courtService` to `services`, to `createApp`'s return value, and to `createRouters`
as `{ path: "/v3", router: courtV3Routes({ courtService: services.courtService, auth }) }`,
last. Update the JSDoc above `createRouters`.

### 4.10 Tests

Follow the README's **Testing** table. In full:

| File | What |
| --- | --- |
| `test/unit/courts/domain/courtRequest.test.mjs` | every rule of §4.2, each error message |
| `test/unit/courts/domain/courtControlGrid.test.mjs` | `courtNumberOf` (`3`, `"3"`, `"Court 12"`, `" 7 "`, `"Next Stand By"`, `""`, `null`, `0`); `courtRows` on **three fixtures copied from fact 1**: the standard layout (header row 4, courts from row 5), the team layout (header row 5, courts from row 6, `#` in C of courts 7–12, the `Next Stand By` / `Match to Record` rows at 50–59) and the PNF layout (`0` counters in row 3, outside the read); `readMatchCell` for every row of §3's table; `isCourtCell` and `matchCellRange` |
| `test/unit/courts/CourtService.test.mjs` | against `FakeSheets` and a fake `syncService`, one test per numbered step of §4.7 that can fail or branch: off / archived; unknown facility; no tab; no court; court on two rows; formula in C; `#` in C; conflict; unchanged (no write, still publishes); set; replace; clear (uses `clearCourtMatch`); match not in SCHEDULE; match twice in SCHEDULE; BYE; on another court (but **not** when only `Next Stand By`'s C holds it); publish failure and publish throw (written, `sync.ok` false); the log line; the team layout's court 3 writes `C12` and the standard layout's writes `C11`; the three reads are issued together, and SCHEDULE is not read for a clear |
| `test/unit/clients/SheetsClient.test.mjs` | §4.5 |
| `test/helpers/fakeSheets.mjs` | add `writeCourtMatch` / `clearCourtMatch`: they record into `courtWrites` / `courtClears` and set or clear the cell in the workbook's `Court Control` tab (strip the quotes from the range). `failOn` applies to them |
| `test/integration/court-routes.test.mjs` | over HTTP: 200 set, replace and clear; 401 with no token, a scorer token and a desk token; 403 off; 404 unknown day, facility and court; 405 on `GET`; 415; 400 bad court and bad body; 409 `court_conflict` with `current`; 409 `match_on_court` with `onCourt`; 422 `court_layout` and `unplayable_match`. Every non-2xx is `application/problem+json`. Copy the setup of `test/integration/v3.test.mjs`'s score cases (`withScoreEntry`, `isProblem`) |
| `test/helpers/routeManifest.mjs` | `{ route: "PUT /v3/days/:day/facilities/:facility/courts/:court/match", test: "test/integration/court-routes.test.mjs" }` |
| `test/integration/v3.test.mjs` | add the route to `V3_ROUTES`. Now *"exactly the eighteen routes, on fifteen paths"* |
| `test/unit/guards/rest-conventions.test.mjs` | an `EXCEPTIONS` row, rule `M2`: *"the live court is read from the published day snapshot, which pages already hold; a Sheets read per GET would spend Google quota for no reader"* |
| `test/unit/guards/dependency-rules.test.mjs` | add `courts` to `featureOf`'s list of feature folders |
| `test/unit/docs/openapiSpec.test.mjs` | the route count. Regenerate its snapshot with `npm run test:snapshots`, then read the diff: only the new route and schema may change |
| `test/unit/helpers/routes.test.mjs` | `registeredRoutes(…).length` goes from 32 to 33 |
| `test/helpers/http.mjs` | `startApp`'s stub services gain `courtService: {}` beside `scoreService: {}` |
| `package.json` | `"test:coverage:courts"` (90 %, same form as `test:coverage:scores`), chained into `test:coverage` |

### 4.11 Version and Changelog

Set `package.json` to `3.8.0`. Add a `README.md` Changelog entry at the top in the
house style:

```
### 3.8.0

- **Court control from Control Center.** `PUT /v3/days/{day}/facilities/{facility}/courts/{court}/match`
  puts a match on a court, replaces it, or clears the court: it finds the court's row in `Court Control`
  by the number in column B, writes that row's C as the service account, and publishes the facility.
  It refuses a cell holding a formula or text, a match not in `SCHEDULE` or a BYE, a match already on
  another court, and a cell that changed since the console showed it (409 with `current`). On when the
  event's `scoreEntry` is set; operator token only.
- Tests: `courtRequest.test.mjs`, `courtControlGrid.test.mjs`, `CourtService.test.mjs`,
  `SheetsClient.test.mjs`, `court-routes.test.mjs`, `v3.test.mjs`, `rest-conventions.test.mjs`,
  `dependency-rules.test.mjs`, `openapiSpec.test.mjs` (and its snapshot), `routes.test.mjs`,
  `fakeSheets.mjs`, `routeManifest.mjs`, `http.mjs`.
```

Also update `README.md`'s layout section (if it lists `src/` folders) and its coverage row.

---

## 5. Site engine (`lib/v1/`)

### 5.1 `lib/v1/views/live-matches.js`: the `courtCell` option

Add one export and one option. Do not change any existing function's output when the option is
absent.

```js
/**
 * A court's row HTML with its court cell decorated: `className` added to the cell's class,
 * `attrs` (a string starting with a space) added to its tag, `html` appended inside it. The
 * first <td class="live-court-cell…"> only; the row unchanged when deco is null.
 * @param {string} rowHTML
 * @param {{ className?: string, attrs?: string, html?: string } | null} deco
 */
export function withCourtCell(rowHTML, deco){
  if(!deco) return rowHTML;
  return rowHTML.replace(/<td class="live-court-cell([^"]*)"([^>]*)>([\s\S]*?)<\/td>/, (all, cls, rest, inner) =>
    `<td class="live-court-cell${cls}${deco.className ? ' ' + deco.className : ''}"${rest}${deco.attrs || ''}>${inner}${deco.html || ''}</td>`);
}
```

In `facilityTableHTML`, change only the `rows.map(…)` to:

```js
rows.map(([c, m]) => withCourtCell(liveTableRowsHTML(c, m, model, options),
  options.courtCell ? options.courtCell(facility, c, m) : null)).join('')
```

Document the option in the extension-point comment at the top of the file:
`options.courtCell(facility, court, match|null) → { className, attrs, html } | null`, which
decorates each court's cell (Control Center's **Set match**). Also add it to
`renderLiveMatches`'s JSDoc `options` type. This works for all three row builders (fact 4)
without touching `teams.js`.

Compatibility: Control Center passes the option to `renderLiveMatches`, an existing export.
A browser holding a cached old `live-matches.js` ignores it and shows no button. That
is harmless, so this needs no two-step push.

### 5.2 `lib/v1/css/live-matches.css`

Append:

```css
/* Control Center's Set match (control-center-court-control-spec.md §5.2): the court cell is the button. */
td.live-court-cell.court-settable{ cursor:pointer; }
td.live-court-cell.court-settable:hover,
td.live-court-cell.court-settable:focus-visible{ outline:2px solid var(--green); outline-offset:-2px; }
.court-hint{
  display:block; margin-top:4px;
  font-family:'Inter',sans-serif; font-size:11px; font-weight:600; letter-spacing:0;
  color:var(--green-dark); white-space:nowrap;
}
@media (max-width:480px){ .court-hint{ font-size:10px; margin-top:2px; } }
```

The idle cell's muted colour does not apply to `.court-hint`, which keeps `--green-dark`.

### 5.3 `lib/v1/domain/courts.js` (new, pure)

```js
import { courtNumberFrom, eventScheduleMinutes } from './progress.js';
import { matchByeSide } from './byes.js';

/** The match on court `court` ("3") among one facility's matches: the highest-numbered whose
 *  liveCourt's number is `court`, as buildFacilityCourtGroups picks. null when idle. */
export function matchOnCourt(facilityMatches, court){ … }

/** Up to `limit` matches to suggest for court `court`: not played, on no court, not a BYE;
 *  sorted by Schedule time (none last), then those scheduled on this court first, then number. */
export function courtSuggestions(facilityMatches, court, { limit = 3 } = {}){ … }

/** Why match number n can't be picked for court `court`, or null when it can. */
export function courtPickError(n, facilityMatches, court){ … }

/** The Review step's warnings for putting `m` on court `court` (D5). */
export function courtWarnings({ m, court, current, dayMatches, type }){ … }

/** The warning for clearing a court whose match `current` is unscored, or []. */
export function clearWarnings(current){ … }
```

The rules:

- **`courtPickError`.** It checks these in order and returns the first that applies:
  - `n` is not among `facilityMatches`: *`No match #${n} at this venue today.`*
  - `matchByeSide(m) !== null`: *`Match #${n} is a BYE: it is never played.`*
  - `courtNumberFrom(m.liveCourt) === court`: *`Match #${n} is already on this court.`*
  - `m.liveCourt` is set: *`Match #${n} is on Court ${courtNumberFrom(m.liveCourt) ?? m.liveCourt} now. Clear that court first.`*
- **`courtWarnings`.** It returns these in this order, skipping any that don't apply:
  1. `m.played`: *`Match #${n} already has a score (${t1Score}–${t2Score}).`*
  2. `s = courtNumberFrom(m.court)` is not null and differs from `court`:
     *`Scheduled for Court ${s}.`*
  3. **Players live elsewhere.** For each name among `t1p1, t1p2, t2p1, t2p2`, skip it if
     it is `TBD` or `bye` (any case) or a name already reported. Look in `dayMatches`
     (every facility, with `.facility`) for a match `x` with `x.liveCourt` set that holds
     the name (compare trimmed, lower-cased). Skip `x` when it is `m` itself, and skip it
     when it is `current`, the match being replaced. When there is one:
     *`${name} is on Court ${courtNumberFrom(x.liveCourt) ?? x.liveCourt} now (#${x.num}${x.facility !== m.facility ? ' at ' + x.facility : ''}).`*
  4. Players not decided (`m.t1p1 === 'TBD' || m.t2p1 === 'TBD'`): for `type === 'team'`,
     *`Lineup not set for this match.`*, otherwise
     *`Players for this match aren’t decided yet.`*
  5. `current && !current.played`:
     *`#${current.num} isn’t scored yet. It leaves the court and shows as not started.`*
- **`clearWarnings(current)`** returns rule 5 alone, or `[]`.

`courtNumberFrom` returns a string (`'3'`), so compare court numbers as strings throughout.
`court` is a string too.

### 5.4 `lib/v1/views/court-dialog.js` (new)

`createCourtDialog(opts)` → `{ open(day, facility, court), close(), refresh(), isOpen() }`.
Model it on `views/score-dialog.js`: the same structure, the same save, slow-timer,
conflict and plumbing code, adapted. Read that file first.

```
opts:
  dialog              the <dialog> (markup §6.1)
  apiBase, getToken(), fixture             as the score dialog
  matchesFor(facility) -> that facility's matches now (objects with .facility)
  dayMatches()        -> every match of the day now
  eventType()         -> 'standard' | 'dual-meet' | 'team'
  describe(m)         -> { sub, sides: [{ name, code, players }, …] }   (the score dialog's describe)
  multiFacility()     -> boolean
  notify(kind, text), friendlyError(raw), unreachable(err), expiredText, resyncHint
  onFixtureSave(state, matchNumber|null)
```

`open(day, facility, court)` sets up this state:
`{ day, facility, court, current: matchOnCourt(matchesFor(facility), court),
expected: current ? current.num : null, step: 'pick', choice: null, typed: '', clear:
false, note: '', error: '', conflict: null, saves: 0 }`. It then shows the dialog modal
and focuses the first suggestion, or the input when there is none.

A side's text is `side.players && side.name === side.code ? side.players.join(' / ') : side.name`
(a pair by its players, a team by its name), and `'To be decided'` when `players` is null
for a pair. A match's one-line text is `${sideText(a)} v ${sideText(b)}`. Its meta is
`#${m.num} · ${m.time || 'no time'} · ${m.court || 'no court'}`.

**The four steps, using the score dialog's classes:**

1. **`pick`.**
   - Title: `Court ${court}`, plus `· ${facility}` when `multiFacility()`.
   - Sub: `Now: #${current.num} · ${matchText(current)}`, or `Now: no match`.
   - Body:
     - A label `Next up` (`.court-suggest-label`), then one `<button type="button"
       class="court-option" data-court-pick="${n}" aria-pressed>` per suggestion
       (`courtSuggestions`). Each holds `<span class="court-option-meta">` (the meta),
       `<span class="court-option-sub">` (`describe(m).sub`) and
       `<span class="court-option-text">` (the match text). With no suggestion, show
       `<p class="score-note">No unplayed match is waiting at this venue.</p>`.
     - A `<label class="court-type-label" for="courtTypeInput">Or type a match number</label>`
       and `<input id="courtTypeInput" class="court-type-input" type="text" inputmode="numeric"
       pattern="[0-9]*" maxlength="5" autocomplete="off">`.
   - Clicking an option sets `choice` (and `aria-pressed="true"` on it alone) and clears
     the input. Typing keeps digits only, sets `typed`, sets `choice` to that number (or
     `null` when empty) and un-presses the options. While `choice` has a
     `courtPickError`, show it in `errorEl` and disable **Review**.
   - Actions: `Cancel`; `Clear court` (secondary, only when `current`), which sets
     `clear = true` and goes to `review`; `Review →` (primary, enabled when `choice !==
     null` and there is no pick error). Enter in the input presses Review when it is
     enabled.
2. **`review`.**
   - For a set: `<div class="score-winner">Put #${n} on Court ${court}</div>`, the meta and
     sub as `<div class="score-was">`, the match text as
     `<div class="court-review-match">`, `Replaces #${current.num}` as `score-was` when
     there is a current match, then `courtWarnings(…)` as `<ul class="score-warn">`.
   - For a clear: `<div class="score-winner">Clear Court ${court}</div>`,
     `<div class="score-was">Now #${current.num} · ${matchText(current)}</div>`, then
     `clearWarnings(current)`.
   - Actions: `← Back` (to `pick`, keeping the choice), and the primary button,
     `Put #${n} on Court ${court}` or `Clear Court ${court}`.
   - Guard against a double Enter for 400 ms, as the score dialog does.
3. **`saving`.** `Saving to the sheet…` and both buttons disabled. After 5 s, the note
   reads *"Still waiting on the workbook. Large workbooks can take up to a minute."*
   Esc and the close button do nothing while saving.
4. **`conflict`.**
   - Body: `<p class="score-conflict">The sheet changed since you opened this court.</p>`,
     then the row `It now reads: #${c}` (or `no match`), then the row `You chose: #${n}`
     (or `clear the court`).
   - Actions: `Keep the sheet’s` (close, then notify `ok`
     `Court ${court} left as the sheet has it.`) and `Replace with yours` (set
     `expected = c`, then save).

**Saving.** Send
`PUT ${apiBase}/v3/days/${enc(day)}/facilities/${enc(facility)}/courts/${court}/match`
with `{ matchNumber: clear ? null : choice, expected: { matchNumber: expected } }`, a
bearer token and a 120 s timeout. Handle the answer like this:

| Answer | Then |
| --- | --- |
| 200, `sync.ok` false | close; `warn` `Court ${court} is set in the sheet, but publishing failed: ${friendlyError(sync.error)}. ${resyncHint}` |
| 200, `unchanged` | close; `ok` `Court ${court} already had #${n}.` or `Court ${court} was already clear.` |
| 200 | close; `ok` `#${n} is on Court ${court}.` or `Court ${court} cleared.` |
| 409 with `json.code === 'court_conflict'` and `json.current` | `conflict` step |
| 401 | back to `review` with `expiredText` |
| anything else (409 `match_on_court`, 422, 5xx) | back to `review` with `friendlyError(json.error \|\| 'HTTP ' + status)` |
| no answer / timeout | back to `review` with `unreachable(err)`, or *"No answer after 2 minutes. The court may or may not be set in the sheet: Save again is safe."* |

**Fixture mode** (`opts.fixture`). Wait 700 ms. If the page URL has `courtConflict=1`
and this is the first save, answer 409 `{ code: 'court_conflict', current: { matchNumber: 99 } }`.
Otherwise call `onFixtureSave(state, matchNumber)`, close, and notify `ok`
*"Fixture: not sent. The next poll restores the fixture’s courts."*

**`refresh()`** runs when a new snapshot arrives. In `pick` or `review`, it recomputes
`matchOnCourt(matchesFor(facility), court)`. If its number differs from `expected`, it
sets the note *"The published data now shows #N on this court."* (or *"…shows no match
on this court."*). It changes only the note, never the choice.

### 5.5 `lib/v1/css/court-dialog.css` (new)

Make it short, built on the score dialog's tokens:

- `.court-suggest-label`: 13px, `--ink-soft`, margin 0 0 6px.
- `.court-option`: a full-width block button with left-aligned text, 10px 12px padding,
  `border:2px solid var(--line)`, `border-radius:10px`, `background:var(--white)`,
  margin-bottom 8px, `min-height:44px`. With `[aria-pressed="true"]`, the border is
  `var(--green)` and the background `color-mix(in srgb, var(--green) 10%, var(--white))`.
  `:focus-visible` gets a 2px `--green` outline.
- `.court-option-meta`: block, 12px, weight 600, `--ink-soft`.
- `.court-option-sub`: block, 13px, `--ink-soft`.
- `.court-option-text`: block, 15px, weight 700, `overflow-wrap:anywhere`.
- `.court-type-label`: block, 13px, `--ink-soft`, margin 12px 0 4px.
- `.court-type-input`: the score input's look at 20px and height 48px, width 100%.
- `.court-review-match`: 17px, weight 700, centred, margin 6px 0 10px.

---

## 6. `tools/control-center.html`

### 6.1 Markup and styles

Link `/lib/v1/css/court-dialog.css` after `score-dialog.css`. After `#linkDialog`, add:

```html
<!-- Court dialog (control-center-court-control-spec.md §5.4): Live Matches' Set match. -->
<dialog id="courtDialog" class="score-dialog court-dialog" aria-labelledby="courtDialogTitle">
  <div class="score-head">
    <div>
      <h2 id="courtDialogTitle" class="score-title"></h2>
      <div id="courtDialogSub" class="score-sub"></div>
    </div>
    <button type="button" class="score-x" id="courtDialogClose" aria-label="Close">&times;</button>
  </div>
  <div id="courtDialogBody"></div>
  <p id="courtDialogNote" class="score-note" hidden></p>
  <div id="courtDialogError" class="score-error" role="alert" hidden></div>
  <div id="courtDialogActions" class="score-actions"></div>
</dialog>
```

### 6.2 Script

1. Import `{ createCourtDialog }` from `/lib/v1/views/court-dialog.js`.
2. **Share the describe.** Move the score dialog's `describe(m)` body into a named function
   `describeMatchForDialog(m)` and pass it as `describe` to both dialogs. Its behaviour
   does not change.
3. **The court cell.** In `drawLiveMatches()`, pass
   `courtCell: scoreEntryAvailable() ? courtCellDeco : undefined`, where:

   ```js
   // Live Matches' Set match (court-control spec §5.1, D7): the court's own cell is the button.
   function courtCellDeco(facility, court, m){
     const n = Live.courtNumberFrom(court);
     if(n === null) return null;
     return {
       className: 'court-settable',
       attrs: ` data-court-facility="${escapeHtml(facility)}" data-court-number="${n}" role="button" tabindex="0"` +
              ` aria-label="${m ? `Change the match on court ${n}` : `Put a match on court ${n}`}"`,
       html: `<span class="court-hint" aria-hidden="true">✎ ${m ? 'Change' : 'Set match'}</span>`,
     };
   }
   ```

4. **Open it.** Add a delegated `click` listener on `liveBoardEl` for
   `[data-court-number]`, ignoring clicks inside `button, a, input`. Add a `keydown`
   listener for Enter and Space on a focused `[data-court-number]`. Both call
   `courtDialog.open(DAYS[currentDayIndex].key, el.dataset.courtFacility, el.dataset.courtNumber)`.
5. **Create it:**

   ```js
   const courtDialog = createCourtDialog({
     dialog: document.getElementById('courtDialog'),
     apiBase: CLOUD_RUN_BASE_URL, getToken: currentAuthToken, fixture: FIXTURE,
     matchesFor: f => MATCHES.filter(x => x.facility === f),
     dayMatches: () => MATCHES,
     eventType: () => CURRENT_TYPE,
     describe: describeMatchForDialog,
     multiFacility: () => FACILITIES.length > 1,
     notify: (kind, text) => showToast(kind, text, 'court'),
     friendlyError: friendlyApiMessage, unreachable: unreachableMessage,
     expiredText: 'Your sign-in expired. Sign in on Mission Control, then save again.',
     resyncHint: 'Use “Resync this day now” on Mission Control.',
     onFixtureSave: fixtureSetCourt,
   });
   ```

6. **`fixtureSetCourt(state, n)`** patches both `MATCHES` and
   `MODEL.matchesByFacility.get(state.facility)` (fact 6). Every match whose
   `courtNumberFrom(liveCourt) === state.court` gets `liveCourt = ''`. When `n !== null`,
   match `n` of that facility gets `liveCourt = state.court`. Then call
   `drawLiveMatches()` and `finder.refresh()`.
7. **Keep it current.** Call `courtDialog.refresh()` beside `scoreDialog.refresh()` (fact 5).
   In `renderAuthStatus()`, beside `if(MATCHES.length) finder.refresh();`, add
   `if(MATCHES.length && CURRENT_VIEW === 'live') drawLiveMatches();`, so the hints appear
   and disappear with sign-in.
8. **Update the comment** above `scoreEntryAvailable()`. Live Matches still opens no
   score dialog (score entry's D8), but its court cells open the court dialog.

---

## 7. Site tests

### 7.1 Unit tests

- `_tests/unit/views/live-matches.test.mjs`: add these tests.
  - **Without the option nothing changes:** `facilityTableHTML` and `renderLiveMatches`
    called with `courtCell: undefined` give exactly the HTML they give today. Pin it
    against a call with no `courtCell` key at all.
  - `withCourtCell` decorates the first court cell only, keeps its existing classes
    (`idle`), and leaves the row unchanged for `null`.
  - With `courtCell`, every court of the `MAIN`/`ANNEX` model gets exactly one
    `court-settable` cell. The option is called with `(facility, court, match|null)`, and
    an idle court receives `null`.
  - The same for a team model, whose occupied rows come from `options.liveRow`. Use
    `views/teams.js`'s `liveRowHTML` the way Control Center does.
- `_tests/unit/domain/courts.test.mjs` (new): `matchOnCourt` (two matches on one court:
  the higher number wins); `courtSuggestions` (played, live and BYE excluded; time order
  with no time last; this court first within a time; limit); every message of
  `courtPickError`; every warning of `courtWarnings` and `clearWarnings`, including a
  player on court at another facility, the replaced match not counting as a clash, `TBD`
  and `bye` never compared, and the team wording.

### 7.2 The comparison harness

In `_tests/engine/cases.mjs`, `controlCenterSignedInCases`:

1. Add `courtOk`:
   `{ method: 'PUT', path: /\/v3\/days\/[^/]+\/facilities\/[^/]+\/courts\/\d+\/match$/,
   reply: () => ({ status: 200, body: { unchanged: false, sync: { ok: true } } }) }`.
   Add it to `cloudRun`.
2. Add two views:
   - `['live', [tab('Live Matches')]]`
   - `['court-set', [tab('Live Matches'), ifPresent('.court-settable', click('.court-settable')),
     ifPresent('.court-option', click('.court-option')), ifPresent('.court-dialog .score-btn.primary:not(:disabled)',
     click('.court-dialog .score-btn.primary:not(:disabled)'))]]`. These open the first court, pick
     the first suggestion and go to Review. Stop on Review: do not save, so the screenshot shows
     the warnings.

   Check `steps.mjs` for the exact names of `tab`, `click` and `ifPresent`, and for how
   `tab` finds a tab by its label.

The baseline has no court cells, so these cases differ. Add to `_tests/engine/accepted.mjs`:

```js
  // ---- Court control (sage-docs/docs/specs/.../control-center-court-control-spec.md) ----
  {
    case: /^cc-signed-in\/[^/]+\/(live|court-set)\//, kind: 'both', row: 'owner 2026-11-14',
    reason: 'signed in, a Live Matches court cell is a button with “✎ Set match” / “✎ Change”, and opens the court dialog',
  },
```

No other case may differ. The public Hub and the not-signed-in Control Center cases must
compare equal.

---

## 8. Documentation

In `sage-docs` (present tense, no history in the pages):

| Page | Change |
| --- | --- |
| `docs/features/control-center.md` | Live Matches: the court cell's **Set match** / **Change**, the dialog's steps, what it refuses and warns, that it follows score entry's setting |
| `docs/technical/control-center.md` | the `courtCell` option, `courtDialog`, `fixtureSetCourt`, the `courtConflict=1` fixture flag |
| `docs/technical/site-engine.md` | `domain/courts.js`, `views/court-dialog.js`, `css/court-dialog.css`, `withCourtCell` and the `courtCell` option in the module list and the extension points |
| `docs/technical/api.md` | the route in the `/v3` table (no frozen twin); *"eighteen routes on fifteen paths"*; the exceptions-table row; `court_control_off`, `court_layout`, `unplayable_match`, `court_conflict`, `match_on_court` in the errors table |
| `docs/technical/sync-pipeline.md` | where score entry's write-then-publish is described, add that court control does the same |
| `docs/usage/run-the-day.md` | "The court rhythm": courts can be set from Live Matches as well as in the sheet, and the two mix. In step 5's note: a court set from the console does not prove the workbook's own sync; the first hand edit in the sheet does |
| `docs/usage/rehearse.md` | one rehearsal step: set, change and clear a court from Live Matches, and see it on the Hub |
| `docs/changelog.md`, `mkdocs.yml` | a MINOR entry, and the same version in `copyright` |
| this spec | status line to **implemented**, moved to `implemented/` as `docs/specs/README.md` describes. Only once the owner's checks in §10.2 pass. Until then, leave it in `not-started/` and say in §10.1 that it is built |

In the workspace `CLAUDE.md`:

- Add `src/courts/` to the `sage-tools-api` **Layout** list.
- In `src/attendance/`'s `SheetsClient` sentence, add *"and, for court control, one
  `Court Control` match cell in column C"* to what it may write.
- In **Endpoints**, change *"Seventeen routes; thirteen of them are the twin…"* to
  eighteen, and list `PUT /v3/days/{day}/facilities/{facility}/courts/{court}/match`
  (operator token; court found by the number in B; no frozen twin).

---

## 9. Checking it yourself

1. `cd sage-tools-api && npm run verify`. Everything passes, coverage included.
2. `cd sage-match-control.github.io/_tests && npm install && npm run verify`. The unit
   tests pass, and the harness's only differences are the accepted ones (§7.2).
3. **By hand, in the built-in browser**, serve the site locally. The harness's
   `_tests/helpers/server.mjs` does it, or add a `.claude/launch.json` entry. Fixture
   mode needs no sign-in. Open
   `/tools/control-center?fixture=pre` with the event `pickledrive-anniversary-2026`
   (team; its fixture registry entry has `scoreEntry`), then
   `/tools/control-center?fixture=rr` with `bracket-demo-2026` (standard). Pick each
   event and its day in the console, then open **Live Matches** and check:
   - Every court cell shows `✎ Set match` or `✎ Change`. The Hub page on the same
     fixture shows none.
   - Clicking a court opens the dialog with suggestions. Typing a BYE, an unknown number,
     the same match, or a match on another court shows the right error and keeps Review
     disabled.
   - Review shows the warnings. Save closes the dialog, and Live Matches shows the match
     on that court.
   - Clear works.
   - `&courtConflict=1` shows the conflict step first.
   - At 375 px wide, the hint fits and the dialog is a bottom sheet, as the score
     dialog is.
   - Sign out (in non-fixture mode) and the hints disappear.

   Take a desktop and a phone screenshot of Live Matches and of Review.

---

## 10. Finishing

### 10.1 Report back

1. `git status --short` and `git log --oneline -5` from each repo's `court-control` branch.
2. `npm run verify`'s summary line from `sage-tools-api`, and the site harness summary.
3. §9's checks, each ticked or explained, with the screenshots.
4. Anything in this spec you found wrong or had to interpret, and what you did about it.

### 10.2 The owner's checks (not yours)

The owner merges and pushes `sage-tools-api` first. Cloud Run deploys it, and `/ping`
must read 3.8.0. Only after that are the site and docs merged, and **never within 3 days
of an event**. Then the owner runs these checks, on **copies** of the Standard
Tournament Master, a team workbook and a dual-meet workbook, registered as a scratch
event with `"scoreEntry": "console"`:

| # | Check | Pass |
| --- | --- | --- |
| K1 | Set court 3 on each copy | the number lands in C of court 3's own row (row 11 standard, row 12 team), as a number; `CSV`'s `court` column shows the label; the Hub shows the match live |
| K2 | The copy's Apps Script **Executions** after K1 | no `sheets-sync.gs` run caused by the API write |
| K3 | Replace, then clear | the old match leaves the court; after a clear, C is truly empty (`=ISBLANK(C11)` is TRUE) |
| K4 | Type a number into C by hand while the dialog is on Review, then save | the conflict step, showing the typed number |
| K5 | Put a formula in an idle court's C, then set it | 422 *"…holds a formula. Set this court in the sheet."* |
| K6 | Time from Save to the Hub updating | recorded in `technical/sync-pipeline.md`; no pass mark |

---

## 11. Later phase: "Court 3 is free: put #57 on it?" (outline only; do not build)

After a score is saved for a match that is live on a court, the score dialog offers to put
the next match on that court in the same step. This is the day's rhythm, "score it, call the
next one", in one place. Sketch:

- A new optional `afterSave(state, json)` option of `createScoreDialog`, off by default, so
  the scorer page is unchanged. Control Center uses it to open the court dialog for
  that court with `courtSuggestions`' first match already chosen.
- No API change: it is the same `PUT …/courts/{court}/match`.
- Open questions for when it is specified: whether it opens the court dialog or asks a
  one-button question in the score dialog's success state, and whether a scored match's
  court is cleared when the operator declines.

---

## 12. Alternatives considered (do not build these)

- **A pencil icon beside the match.** It reads as "edit this score", and score entry
  deliberately keeps Live Matches read-only for scores. It is also a tiny tap target at
  phone sizes.
- **Exact-text court lookup.** Live Matches does not know B's exact text for an idle court
  (fact 3).
- **A fixed geometry (`5 + 3·(k − 1)`).** It is wrong for the team and dual-meet workbooks
  (fact 1).
- **`GET …/courts`.** It would cost a Sheets read for no reader the snapshot doesn't
  already serve (D4).
- **Moving a match in one request.** That is two cells, and a half-done write would leave
  the match on both courts or neither (D6).

---
**Related:** [Score entry from Control Center](../implemented/control-center-score-entry-spec.md) ·
[Run the day](../../usage/run-the-day.md#the-court-rhythm) ·
[Control Center technical](../../technical/control-center.md) ·
[Site engine](../../technical/site-engine.md) ·
[API](../../technical/api.md)
