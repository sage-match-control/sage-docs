# Spec — Who a link was issued to, recorded on what it changes

An operator who issues a **desk link** or a **scorer link** can say who it is for
and add a short note, both optional. Whatever that link then changes in a
facility workbook carries that label:

- **Attendance:** a `markedBy` column in the `ATTENDANCE` tab records who last
  marked or unmarked each person, what they did and when:
  `In · Ana (Gate A) · 14:32`.
- **Scores:** both score cells of a match get a Google Sheets **note** (the
  corner triangle) listing the last five score-entry edits, newest first:
  `11–7 · Ana (Courts 3–4) · Oct 8 14:32 · was 9–11`.

Operator writes from Control Center are labelled `Control Center`.

> **Status: implemented.** Built in `sage-tools-api` 3.1.0, Control Center, the desk and
> scorer pages (`lib/v1`) and the docs, and merged into `main` on 2026-10-09. Written
> 2026-10-08 against `sage-tools-api` 3.0.1. It is tested against fakes; the real-Google
> checks in §10.2 (L1–L7) have not run, so the first event to issue a labelled link is
> also its first real-workbook use. **Divergence:** in fixture mode a desk link works like
> a scorer link (§5.3 gives it only a real payload): it opens the local `attendance.html`
> with the same `&fixture=` and lasts to the end of the event's day in Manila, or of today
> once that day has passed, so §6 step 3 can follow it. **Note:** the label shows on desk
> pages made from `_templates/attendance/`; the three desk pages of Pickle for Sight,
> PickleDrive and Piggleball are older inline copies, frozen with their events, and don't
> show it. The workbook records `markedBy` either way.
>
> Builds on [Score entry and scorer links](../implemented/control-center-score-entry-spec.md)
> and [Attendance for every event](../implemented/multi-event-attendance-spec.md).
> Read their §0 if a term here is unfamiliar.

---

## 0. Read this first

You are implementing this with no other context. Everything you need is in this
page and the files it names. Read §0 to §3 in full before writing code, then do §4
to §7 in order. §10 is what to report when you finish.

### 0.1 The system in one paragraph

SAGE runs pickleball tournaments. Each **event** has **days**, each day has
**facilities** (venues), and every facility-day has its own **Google Sheets
workbook**. `sage-tools-api` (Node/Express on Cloud Run) writes to those workbooks
as a Google service account in exactly two places: a person's attendance in the
`ATTENDANCE` tab, and a match's two score cells in `SCHEDULE`. Operators sign in to
**Control Center** (one HTML file) with one shared login, so the API cannot tell
operators apart. Staff who are not operators use **links**: a **desk link** opens the
event's attendance desk page and can only mark attendance for one day; a **scorer
link** opens the event's scorer page and can only enter scores for one day. A link is
a stateless HMAC-signed token (`<base64url JSON payload>.<signature>`). The API keeps
no record of issued links, so anything a link should carry must be in its payload.

### 0.2 Where things are

`D:\Personal\SAGE` is a plain folder holding four git repos:

| Path | What you change there |
|---|---|
| `sage-tools-api/` | §4: tokens, the two issue routes, attendance's `markedBy`, score notes, `SheetsClient`, tests, version, Changelog |
| `sage-match-control.github.io/` | §5: `tools/control-center.html` (the two issue forms), `lib/v1/data/tokens.js`, `lib/v1/apps/attendance-desk.js`, `lib/v1/apps/scorer.js`, `_tests/` |
| `sage-docs/` | §7: feature, technical and usage pages |
| `D:\Personal\SAGE\CLAUDE.md` | §7: two sentences (not in any repo) |

`event-data/` and every `apps-script/*.gs` file are **not** touched (§8).

### 0.3 Hard rules

1. **Leave every change uncommitted.** No `git add`, `commit`, `push`, branch,
   switch or stash. The owner reviews and commits.
2. **Never write to a real workbook, never call the production API**, and never
   touch the real `events.json`. Tests use fakes. Real-Google checks are the
   owner's (§10.2).
3. **Write each test first and watch it fail**, then make it pass
   (`sage-tools-api/CLAUDE.md` rule). **Never weaken or delete a test.** The existing
   assertions that change on purpose are listed in §4.11; nothing else existing
   should.
4. `sage-tools-api`: ESM `.mjs`, classes with constructor injection, **no new
   dependency**, no TypeScript. 4-space indent, double quotes, comments that say why.
   Obey the dependency rules in `D:\Personal\SAGE\CLAUDE.md` (enforced by
   `test/unit/guards/dependency-rules.test.mjs`).
5. Site: no new dependency, 2-space indent, single quotes, match the surrounding
   style. **`lib/v1` changes must be compatible** (`_templates/CLAUDE.md` §8): add,
   never remove or rename an export; and **no module may start importing a new export
   of another module in this change**. A browser can hold a 10-minute-old copy of one
   file beside a new copy of another. A new import of a missing export kills the page.
   §5 is written so nothing needs a new export.
6. Find functions **by name** (`grep -n "function name("`), not by line number.
7. Line endings: keep whatever each file has today (the files named here are LF).
8. Cite this spec from code comments as
   `sage-docs/docs/specs/.../link-attribution-spec.md`, with literally `...`.
9. Documentation is present tense: what the system does, not what changed.
10. If the code contradicts this spec in a way that blocks you, stop and report it
    (§10). Don't work around it.

### 0.4 Facts checked against the code (2026-10-08)

**`sage-tools-api`**

| Fact | Where |
|---|---|
| Desk token payload `{ exp, scope: "attendance-desk", day }`, scorer token `{ exp, scope: "score-desk", day }`, each base64url of UTF-8 JSON, signed by `#signScoped(scope, payload)`. `verifyDeskToken`/`verifyScorerToken` return `{ day, exp }` or `null` | `src/auth/AuthService.mjs` |
| `createAuthMiddleware`'s `requireOperatorOr(kind)` sets `req.actor = { kind, day }` for a scoped token, `requireOperator`/`isOperator` set `{ kind: "operator" }` | `src/auth/middleware.mjs` |
| Token port typedefs: `TokenVerifier`, `DeskTokenIssuer`, `ScorerTokenIssuer` | `src/shared/ports.mjs` |
| `/v1` and `/v3` desk-link routes share `handlers.deskLink`, which calls `attendanceService.issueDeskLink(req.params.day)` and answers 201 `{ token, expiresAt, day }`. Same for scorer links (`handlers.scorerLink`, `scoreService.issueScorerLink`). Neither route reads a body today | `src/attendance/routes.mjs`, `src/scores/routes.mjs` |
| The pattern for a `/v3` route with an optional JSON body is `requireJson({ optional: true })` before the handler, a separate `…V3` handler, and a `415` in the `@openapi` block. See `/v3/days/{day}/attendance/reconciliations` (`reconcileV3`) | `src/attendance/routes.mjs`, `src/server/requireJson.mjs` |
| The `ScopedToken` OpenAPI schema is `{ token, expiresAt, day }` | `src/docs/openapiSpec.mjs` |
| `AttendanceService.mark({ day, facilityName, key, present, actor })` reads `ATTENDANCE!A1:G`, `parseTab`s it, `planMark(row, present, now)` → `{ changed, present, timeIn }`, and when changed writes **one** `updateValues` range `ATTENDANCE!E{r}:F{r}` | `src/attendance/AttendanceService.mjs`, `src/attendance/domain/attendanceTab.mjs` |
| `parseTab` throws `AttendanceLayoutError` unless row 1 is **exactly** the 7 `HEADERS`; a test pins that an extra 8th header throws (`attendanceTab.test.mjs`, "early-extra"). That stays (§1 D8) | same, and `test/unit/attendance/domain/attendanceTab.test.mjs` |
| `formatTimeIn(date)` gives `yyyy-MM-dd HH:mm` in Asia/Manila through an `Intl.DateTimeFormat` with `hourCycle: "h23"` | `attendanceTab.mjs` |
| **Column H onward belongs to people**: an organizer may keep their own columns there (the desk page shows an optional `TShirt Size` column wherever it is). Column **J** holds the "Not yet in" `FILTER` formula (J1 header, J2 formula spilling down) in every tab the API or a generator creates | `SheetsClient.mjs` comment above `NOT_YET_IN_RANGE`; `lib/v1/domain/attendance.js` reads columns by header name |
| `SheetsClient.updateValues` refuses any range not matching `WRITABLE_RANGE` (`ATTENDANCE!` A–G). `writeScores`/`clearScores` refuse any range failing `assertScoreRange` (one row ≥ 6, team-1 score column `9 + 8k` and the next). Reads use the API key; writes the service account's token through `#send(url, init, { write })`; `#assertOk` turns a write 403 into "Share it with … as Editor" | `src/clients/SheetsClient.mjs` |
| `ScoreService.submit` reads `SCHEDULE`, finds the match, checks `expected`, writes or clears the two cells (only when not `unchanged`), calls `syncService.syncDay`, logs one line `score <day>/<facility> #<n> <c1>:<c2> <prev> -> <new> by=<kind> sync=<ok|failed>`, and returns the 200 body | `src/scores/ScoreService.mjs` |
| `at.scoreRange` is `SCHEDULE!<c1><row>:<c2><row>`; `previous` is `{ team1Score, team2Score }`, each an integer or `null` | same |
| `FakeSheets` stands in for `SheetsClient` in unit and integration tests, recording `reads`, `writes`, `scoreWrites`, `scoreClears` | `test/helpers/fakeSheets.mjs` |
| `ScoreService.test.mjs` pins the log line for operators (`… by=operator sync=ok`) and a scorer (`/ by=scorer sync=ok$/`) | `test/unit/scores/ScoreService.test.mjs` |
| Version 3.0.1 | `package.json` |

**The Sheets API (Google's v4 reference)**

| Fact |
|---|
| `GET https://sheets.googleapis.com/v4/spreadsheets/{id}?ranges=<A1>&fields=sheets(properties(sheetId),data(rowData(values(note))))&key=<API key>` returns `{ sheets: [ { properties: { sheetId }, data: [ { rowData: [ { values: [ { note? }, … ] } ] } ] } ] }` for just the range's tab. Empty parts are **omitted**, not empty: no `note` key on a cell without one, no `values` or `rowData` when nothing in the range has a note |
| `POST https://sheets.googleapis.com/v4/spreadsheets/{id}:batchUpdate` with `{ requests: [ { repeatCell: { range: { sheetId, startRowIndex, endRowIndex, startColumnIndex, endColumnIndex }, cell: { note: "<text>" }, fields: "note" } } ] }` sets that note on every cell of the range and changes nothing else. Indexes are 0-based, end-exclusive. `sheetId` here is the tab's numeric ID (its gid), not the spreadsheet ID |
| A note is plain text. API writes fire no onEdit trigger, and the sync reads CSV, which carries no notes: a note never reaches a snapshot |

**The site**

| Fact | Where |
|---|---|
| `tokens.js` decodes a payload with `JSON.parse(atob(…))`. `atob` yields one character per **byte**, so any non-ASCII name ("Niño") decodes as mojibake today | `lib/v1/data/tokens.js`, `decodePayload` |
| Control Center issues links in `issueScorerLink()` and `attConsoleIssueDeskLink()`, POSTing with only an `Authorization` header; renders them in `renderScorerLink(day, issued)` and `attConsoleRenderDeskLink(day, issued)`, whose result box has a `.result-title` div. In fixture mode it fakes the token: the scorer one with `btoa(JSON.stringify(…))`, the desk one as the literal `'fixture.desk-token'` | `tools/control-center.html` |
| Markup: `#scorerLinkBtn` + `#scorerLinkResult` in Mission Control's `#scorerLinksSection`; `#attendanceDeskBtn` + `#attendanceDeskResult` in the Attendance tab's `.attendance-actions`. Text inputs use `class="organizer-text-input"` (see `#orgUsernameInput`) | same |
| The desk page sets `#dayLabel` once to `` ` · ${dayEntry.label || info.day}` ``. The scorer page sets `dayLabelEl.textContent = ` · ${label}` + (facility ? ` · ${facility}` : '')` whenever the facility changes | `lib/v1/apps/attendance-desk.js`, `lib/v1/apps/scorer.js` |

---

## 1. Decisions (settled with the owner, 2026-10-08)

| # | Decision |
|---|---|
| D1 | Issuing a desk or scorer link takes two **optional** fields: **Issued to** (`issuedTo`) and **Note** (`note`). Each is at most **40 characters** after trimming. Only the `/v3` issue routes accept them; `/v1` is frozen and ignores any body |
| D2 | Both travel **inside the signed token** (payload keys `to` and `note`, present only when given). No server-side record of links. A link issued without them has a byte-identical payload to today's; links issued before this change keep working, unlabelled |
| D3 | **The label** of a write: operator → `Control Center`. Desk or scorer link → `Ana (Gate A)` with both, `Ana` with only a name, `Desk link (Gate A)` / `Scorer link (Courts 3–4)` with only a note, `Desk link` / `Scorer link` with neither. **No operator name field**: every operator write reads `Control Center` |
| D4 | It is **attribution, not authentication**: the label says who the link was issued to, not who typed. Anyone holding a forwarded link writes under its label. Docs say so |
| D5 | **Attendance:** a `markedBy` column holds the **last** change to that person: `In · <label> · HH:mm` or `Out · <label> · HH:mm` (Manila, 24-hour). Written in the same `updateValues` call as `present`/`timeIn`, only when the mark changes something (an unchanged mark writes nothing, as today) |
| D6 | **Where `markedBy` lives:** the column whose row-1 header is exactly `markedBy`, in H–Z except J. If none, the API claims the **first column from H to Z, skipping J, whose row 1 and every data row are blank**, and writes the `markedBy` header into it in the same call. If no column qualifies, the mark is written without `markedBy` and a warning is logged. It never overwrites a person's own column |
| D7 | **The generators and `apps-script/` are not touched.** A new workbook's `ATTENDANCE` tab has no `markedBy` header until the first mark claims one. `HEADERS` stays the 7 columns A–G |
| D8 | `parseTab` stays strict on A–G. `mark` reads `ATTENDANCE!A1:Z` and gives `parseTab` only the first 7 cells of each row |
| D9 | **Scores:** after a save that changed the cells (not `unchanged`), the API puts the **same note on both score cells**. The note is score entry's last **five** lines, newest first, followed by any other text that was in either cell's note (a person's own note is kept, never trimmed) |
| D10 | One line: `<t1>–<t2> · <label> · <Mon D HH:mm>` plus ` · was <p1>–<p2>` when the cells weren't both empty before; a clear is `cleared · <label> · <Mon D HH:mm> · was <p1>–<p2>`. A missing side of a half-filled previous score prints `?`. En dash `–`, middle dot `·`, Manila time |
| D11 | **The note is best effort.** It is written after the score write and the publish, before responding. If reading or writing the note fails, the save still succeeds: the failure is logged as a warning and the response is unchanged. The score is what matters |
| D12 | **No response body changes** on any route except the two `/v3` issue routes, which echo `issuedTo` and `note` (`null` when not given). The mark and score responses, and every `/v1` body, stay exactly as they are |
| D13 | The desk and scorer pages show the label after the day: ` · Day 1 · Ana (Gate A)`. Nothing when the link has neither field |
| D14 | Control Center's two issue buttons each get two optional text inputs above them, cleared after a successful issue so the next link isn't mislabelled. The result box's title carries the label |

---

## 2. Overview

```
Control Center ──POST /v3/days/{day}/…/desk-links  { issuedTo?, note? }──▶ API
                                  ◀── 201 { token, expiresAt, day, issuedTo, note }
token payload: { exp, scope, day, to?, note? }   (signed: can't be edited)

desk page  ──PUT …/attendance  (Bearer token)──▶ middleware: req.actor = { kind:"desk", day, issuedTo, note }
                                                  AttendanceService.mark
                                                    one updateValues: E:F + markedBy cell (+ header)

scorer page──PUT …/score (Bearer token)──▶ ScoreService.submit
                                             write/clear cells → syncDay → note on both cells (best effort)
```

---

## 3. The label and the two texts (pure functions)

### 3.1 `src/shared/issuedLink.mjs` (new)

`shared/` because both features' routes and services use it, and the dependency
rules let anything import `shared/*`. Imports only `./errors.mjs`.

```js
export const ISSUED_FIELD_MAX = 40;

// { issuedTo, note } from an issue request's body. Each is optional; a string is
// trimmed, inner whitespace (newlines, tabs included) collapsed to one space, and
// control characters (\p{Cc}) removed; "" after that means absent (null).
// Throws ValidationError: "issuedTo must be text" / "note must be text" for a
// non-string non-null value; "issuedTo must be at most 40 characters" (or note) when
// longer after cleaning. Length is counted in code points ([...s].length).
// body may be undefined (no body sent) -> { issuedTo: null, note: null }.
export function parseIssuedLink(body) { … }

// The label of whoever made a write (§1 D3).
// actor: { kind: "operator" } | { kind: "desk" | "scorer", day, issuedTo?, note? } | undefined
// undefined or operator -> "Control Center".
export function actorLabel(actor) { … }
```

`actorLabel` table (the unit test asserts every row):

| actor | label |
|---|---|
| `undefined`, `{ kind: "operator" }` | `Control Center` |
| desk, both | `Ana (Gate A)` |
| desk, `issuedTo` only | `Ana` |
| desk, `note` only | `Desk link (Gate A)` |
| desk, neither (`null`s or keys absent) | `Desk link` |
| scorer, neither / note only | `Scorer link` / `Scorer link (Courts 3–4)` |

### 3.2 `markedBy` (in `src/attendance/domain/attendanceTab.mjs`)

```js
export const MARKED_BY_HEADER = "markedBy";

// "In · Ana (Gate A) · 14:32" / "Out · Control Center · 09:05". Manila, 24-hour.
export function formatMarkedBy(present, label, now) { … }

// Where markedBy goes, given what readValues("ATTENDANCE!A1:Z") returned.
// -> { column: "H", header: false }   an existing markedBy header at H
//    { column: "K", header: true }    claim K: write the header too
//    null                              nowhere to put it (§1 D6)
// Candidates in order: H, I, K, L, … Z (J never). An existing header wins over a
// blank column. A column qualifies for claiming when every row's cell in it is blank
// ("" / undefined / null), row 1 included.
export function markedByColumn(values) { … }
```

Reuse the Manila formatter `formatTimeIn` already uses; `formatMarkedBy`'s time is
the `HH:mm` part.

### 3.3 `src/scores/domain/scoreNote.mjs` (new)

```js
export const NOTE_LINES_KEPT = 5;

// One line (§1 D10). scores and previous are { team1Score, team2Score }, each an
// integer or null; scores both null means a clear.
//   "11–7 · Ana (Courts 3–4) · Oct 8 14:32 · was 9–11"
//   "11–7 · Control Center · Oct 8 14:32"            (previous both null)
//   "cleared · Scorer link · Oct 8 14:40 · was 11–7"
//   "11–7 · Ana · Oct 8 14:32 · was 9–?"             (half-filled previous)
// Date: en-US short month, day without zero, then HH:mm (h23), Asia/Manila.
export function noteLine({ scores, previous, label, now }) { … }

// True for a line noteLine wrote. Anchored: starts with "<d>–<d> · " or
// "cleared · ", and contains " · <Mon> <D> <HH:mm>" after that.
export function isScoreEntryLine(line) { … }

// The note to write on both cells (§1 D9): line, then up to NOTE_LINES_KEPT - 1 of
// the score-entry lines from existing[0] (team 1 cell's note), then every other
// non-empty line from existing[0] and existing[1] in order, each distinct line once.
// existing: [string, string], "" for no note. Lines split on \n; trailing \r trimmed.
export function buildNote(line, existing) { … }
```

Score-entry lines are read from **team 1's** note only (both cells always get the
same text from the API, so team 2's adds nothing but a person's own lines). Domain
rules: no imports at all (`Intl` is a global, not an import).

---

## 4. `sage-tools-api`

### 4.1 Tokens (`src/auth/AuthService.mjs`, `src/shared/ports.mjs`)

- `issueDeskToken({ day, expiresAt, issuedTo = null, note = null })` and
  `issueScorerToken(…)` the same: the payload is
  `{ exp, scope, day }` plus `to: issuedTo` only when it is a non-empty string and
  `note` only when it is. Key order: `exp, scope, day, to, note`. With neither, the
  JSON is exactly today's.
- `verifyDeskToken` / `verifyScorerToken` return
  `{ day, exp, issuedTo, note }`: each `parsed.to` / `parsed.note` when it is a
  non-empty string, else `null`. Nothing else about verification changes.
- `ports.mjs`: the three typedefs gain the optional fields and the new return shape.

### 4.2 Middleware (`src/auth/middleware.mjs`)

In `requireOperatorOr`, set
`req.actor = { kind, day: scoped.day, issuedTo: scoped.issuedTo ?? null, note: scoped.note ?? null }`.
Operators stay `{ kind: "operator" }`.

### 4.3 The issue routes

**`src/attendance/routes.mjs`:** the `/v3` desk-links route becomes
`.post(auth.requireOperator, requireJson({ optional: true }), asyncHandler(handlers.deskLinkV3))`.
`deskLinkV3` calls `attendanceService.issueDeskLink(day, parseIssuedLink(req.body))`.
The `/v1` route keeps `handlers.deskLink`, which keeps calling
`issueDeskLink(req.params.day)` with no second argument, so `/v1` answers exactly as
today (its body, if any, is ignored).

**`src/scores/routes.mjs`:** the same with `scorerLinkV3` and `issueScorerLink`.

**Responses.** The services return `{ token, expiresAt, day, issuedTo, note }` (`null`
for absent). The `/v1` handlers must still answer exactly `{ token, expiresAt, day }`:
have them pick those three keys from the service's result.

**OpenAPI.** Both `/v3` `@openapi` blocks gain:

```yaml
requestBody:
  required: false
  content:
    application/json:
      schema:
        type: object
        properties:
          issuedTo: { type: string, maxLength: 40, description: Who the link is for. Shown on the page it opens and recorded on every change it makes. }
          note: { type: string, maxLength: 40, description: A short note, e.g. a gate or courts. Recorded beside issuedTo. }
      example: { issuedTo: Ana, note: Gate A }
```

and a `"415"` response like reconciliations'. The `400` description adds "issuedTo or
note is not text or longer than 40 characters". Add a schema `IssuedScopedToken`
(the `ScopedToken` properties plus `issuedTo` and `note`, each
`{ type: string, nullable: true }`) and point the two `/v3` `201`s at it.
`ScopedToken` stays as it is (the `/v1` blocks may reference it).

### 4.4 The services' issue methods

`AttendanceService.issueDeskLink(day, { issuedTo = null, note = null } = {})` passes both
to `issueDeskToken` and returns them. `ScoreService.issueScorerLink` the same; its log
line becomes
`scorer link issued <day> until <iso>` plus ` for "<label>"` when either field is
given (label per §3.1, as a scorer actor).

### 4.5 `SheetsClient` (`src/clients/SheetsClient.mjs`)

**Attendance allowlist.** Add

```js
// The one ATTENDANCE cell outside A:G the API writes: a person's markedBy (or its
// header), in H–Z but never J, which holds the "Not yet in" formula. A single cell
// only. See sage-docs/docs/specs/.../link-attribution-spec.md §1 D6.
const MARKED_BY_CELL = /^ATTENDANCE!([HIK-Z])([1-9]\d*)$/;
```

`updateValues` accepts a range matching `WRITABLE_RANGE` **or** `MARKED_BY_CELL`.
Update the comments that say the API never writes past G.

**Score notes.** Two methods, both behind `assertScoreRange(range)`:

```js
// -> { gid, notes: [team1Note, team2Note] } ("" for no note). A read: API key.
async readScoreNotes(sheetId, range)

// Sets the same note on both cells of a score range; nothing else changes.
async writeScoreNote(sheetId, range, gid, note)
```

- `readScoreNotes`: the `GET` in §0.4 with `ranges=<range>` (URL-encoded) and the
  `fields` mask. `gid` = `sheets[0].properties.sheetId`; notes from
  `sheets[0].data[0].rowData[0].values[0|1].note`, any missing part → `""`. Errors as
  `readValues` does through `#send`/`#assertOk` (`write: false`).
- `writeScoreNote`: the `repeatCell` `POST` in §0.4. Parse the range with
  `SCORE_RANGE`: `startRowIndex = row - 1`, `endRowIndex = row`,
  `startColumnIndex = columnNumber(c1) - 1`, `endColumnIndex = columnNumber(c2)`.
  `typeof note === "string"` or throw. `write: true`, so the 403 message applies.

### 4.6 `AttendanceService.mark`

1. Read `ATTENDANCE!A1:Z` instead of `A1:G`.
2. `parseTab(values.map(r => (r ?? []).slice(0, 7)))`. Everything that follows
   about rows is unchanged.
3. When `plan.changed`:
   - `where = markedByColumn(values)`
   - data = `[{ range: E{r}:F{r}, … }]` as today, plus, when `where`:
     `{ range: "ATTENDANCE!<col><r>", values: [[formatMarkedBy(plan.present, actorLabel(actor), this.now())]] }`,
     plus, when `where.header`: `{ range: "ATTENDANCE!<col>1", values: [[MARKED_BY_HEADER]] }`.
   - **One** `updateValues` call with all of them.
   - When `where` is `null`: `this.logger.warn(\`attendance ${day}/${facilityName}: no free column for markedBy (H–Z, not J)\`)`.
4. The return value is unchanged.

`reconcileDay` and everything else in the service keep reading `A1:G`.

### 4.7 `ScoreService.submit`

After the `syncDay` block and the log line, before returning, when `!unchanged`:

```js
await this.#annotate({ sheetId, range: at.scoreRange, scores: { team1Score, team2Score }, previous, actor, where: `${day}/${facilityName} #${n}` });
```

`#annotate` does `readScoreNotes`, `noteLine`, `buildNote`, `writeScoreNote`, inside one
`try`/`catch`. On error: `this.logger.warn(\`score note ${where} not written: ${err?.message ?? err}\`)`.
It never throws.

**The log line.** `by=<kind>` gains the label in brackets **only for a link with a
name or note**: `by=scorer[Ana (Courts 3–4)]`. Operators and unlabelled links log
exactly as today, so the two pinned log tests still pass unchanged.

### 4.8 Ports and wiring

- `ports.mjs`: `AttendanceSheets` is unchanged (still `updateValues`). The Sheets port
  the score service is typed against gains `readScoreNotes` and `writeScoreNote`.
- `src/app.mjs`: nothing new is constructed, so no change is expected. Check it.

### 4.9 Test helpers

`test/helpers/fakeSheets.mjs`:

- `this.notes = {}` keyed `` `${sheetId}!${cellA1}` `` (e.g. `"S1!AG14"`),
  `this.noteWrites = []` (`{ sheetId, range, gid, note }`), `this.noteFailOn = new Set()`
  (sheetIds whose note calls throw, to test D11).
- `readScoreNotes(sheetId, range)` → `{ gid: 1234, notes: [notes[c1] ?? "", notes[c2] ?? ""] }`.
- `writeScoreNote(sheetId, range, gid, note)` records, then sets both cells' notes.
- `updateValues` already writes any column into the `ATTENDANCE` array. Check that it
  does for H–Z; `readValues` already handles `A1:Z`.

### 4.10 New tests (write each first; watch it fail)

| File | Cases |
|---|---|
| `test/unit/shared/issuedLink.test.mjs` (new) | `parseIssuedLink`: no body; `{}`; both given; trimming and collapsing (`"  Ana \n Cruz "` → `"Ana Cruz"`); control chars removed; `""` and `"   "` → `null`; 40 code points OK, 41 → `ValidationError` naming the field; a 40-character name with an emoji counts code points; non-string (`5`, `{}`, `[]`) → `ValidationError`; `null` → `null`. `actorLabel`: every row of §3.1's table |
| `test/unit/auth/AuthService.test.mjs` | desk and scorer: a token issued without the fields has exactly today's payload JSON; with both, the payload has `to` and `note` and verify returns them; with one, the other is `null`; a non-ASCII name (`"Niño"`) round-trips; a token with a tampered `to` fails verification; an old-shape token verifies with `issuedTo: null, note: null` |
| `test/unit/auth/middleware.test.mjs` | `requireOperatorOr("desk")` and `("scorer")` put `issuedTo` and `note` on `req.actor`; `null`s for an unlabelled token |
| `test/unit/attendance/domain/attendanceTab.test.mjs` | `formatMarkedBy` In/Out with a fixed Manila time (pick a UTC instant that is a different date in UTC than in Manila). `markedByColumn`: existing header at H; existing header at L with H blank (existing wins); none → H when H is blank everywhere; H has a header (`TShirt Size`) → I; H and I taken → K (never J); a column with a blank header but a data cell → skipped; everything H–Z taken → `null`; values with ragged short rows |
| `test/unit/scores/domain/scoreNote.test.mjs` (new) | `noteLine`: each example in §3.3, plus a 0 score (`0–11`, not treated as empty). `isScoreEntryLine`: true for each `noteLine` output, false for `"Ask Ref Joy"` and `"11-7 by hand"`. `buildNote`: empty existing; 4 old lines → 5; 5 old lines → newest 5 (oldest dropped); a person's own line in team 1's note kept after score-entry lines; own lines in both notes kept, a duplicate once; `\r\n` notes |
| `test/unit/clients/SheetsClient.test.mjs` | `updateValues` accepts `ATTENDANCE!H5`, `ATTENDANCE!K1`, `ATTENDANCE!Z9`; refuses `ATTENDANCE!J5`, `ATTENDANCE!H5:H6`, `ATTENDANCE!AA5`, `SCHEDULE!H5`, before any fetch. `readScoreNotes`: the URL (path, `ranges`, `fields`, `key`), no `Authorization`; both notes; a response with no `data`/`rowData`/`values`/`note` → `""`s; refuses a non-score range before fetching. `writeScoreNote`: the URL, `Authorization`, the exact body for `SCHEDULE!AG14:AH14` (`startRowIndex 13, endRowIndex 14, startColumnIndex 32, endColumnIndex 34`); a 403 gives the share-with message; refuses a non-score range and a non-string note |
| `test/unit/attendance/AttendanceService.test.mjs` | a desk mark with a label writes E:F **and** the `markedBy` cell in one `updateValues`; on a tab with no `markedBy`, also the header, at H; on a tab whose H is `TShirt Size`, at I; on a second mark, no header write; unmark writes `Out · …`; an operator mark writes `Control Center`; an unchanged mark writes nothing; no free column → E:F only, and a warning; the mark's read is `ATTENDANCE!A1:Z`; a tab with a person's own H column still parses (no `AttendanceLayoutError`); `issueDeskLink(day, { issuedTo, note })` returns them and passes them to the issuer |
| `test/unit/scores/ScoreService.test.mjs` | a save writes the same note on both cells, after the score write; a correction's note lists the new line first and keeps the old; a clear writes a `cleared · … · was …` line; an `unchanged` save reads and writes no note; a note failure (`noteFailOn`) still returns the normal 200 body with `sync.ok`, logs a warning; the scorer log line with a labelled actor ends `by=scorer[Ana (Courts 3–4)] sync=ok`; `issueScorerLink(day, { issuedTo, note })` returns them, its log line says `for "Ana (Courts 3–4)"` |
| `test/integration/attendance-routes.test.mjs` | `POST /v3/…/desk-links` with `{ issuedTo, note }` → 201 echoing them; with no body → `issuedTo: null, note: null`; a 41-character `issuedTo` → 400 problem details; a `text/plain` body → 415; `POST /v1/…/desk-links` with the same body → exactly `{ token, expiresAt, day }` and a token whose payload has no `to`; a PUT with the labelled token writes `markedBy` |
| `test/integration/score-routes.test.mjs` | the same four issue cases for scorer links; a PUT with the labelled scorer token leaves a note on both cells |

### 4.11 Existing tests that change on purpose

- Any `AttendanceService` test asserting the mark's read range `ATTENDANCE!A1:G`
  becomes `A1:Z` (the reconcile reads stay `A1:G`).
- Any test asserting `mark`'s `updateValues` data is exactly one range: a **desk or
  operator mark** now writes two or three ranges (the actor label always exists, so
  `markedBy` is always written when a column is free).
- `test/unit/docs/openapiSpec.test.mjs.snapshot`: regenerate with
  `npm run test:snapshots`, then read the diff: only the two issue routes and the new
  schema may change.
- Tests asserting the services' issue return value exactly (`{ token, expiresAt, day }`)
  gain `issuedTo: null, note: null`.

The strict-header test in `attendanceTab.test.mjs` does **not** change.

### 4.12 Version and Changelog

`package.json` → **3.1.0**. `README.md` Changelog entry `### 3.1.0` at the top:

- Desk and scorer links can name who they're for, with a note (`/v3` issue routes,
  optional body `{ issuedTo, note }`, carried in the signed token).
- Attendance marks record `markedBy` (`In · Ana (Gate A) · 14:32`) in a column the API
  finds or claims in H–Z, never J and never a person's own column; `SheetsClient`'s
  attendance allowlist adds that one cell.
- Score saves put a note on both score cells: the last five edits, newest first, best
  effort.
- `/v1` unchanged.
- **Tests:** every file in §4.10 and §4.11.

Run `npm test` after each step and `npm run verify` at the end. Both must pass.

---

## 5. The site

### 5.1 `lib/v1/data/tokens.js`

`decodePayload` decodes the base64 to **bytes** and the bytes as UTF-8:

```js
function decodePayload(token){
  const part = String(token).split('.')[0].replace(/-/g, '+').replace(/_/g, '/');
  const binary = atob(part + '='.repeat((4 - part.length % 4) % 4));
  return JSON.parse(new TextDecoder().decode(Uint8Array.from(binary, c => c.charCodeAt(0))));
}
```

No new export. `scorerDecode`/`deskDecode` return the whole payload as today, so
`info.to` and `info.note` are there when the token has them. Update the two JSDoc
comments to name `to?` and `note?`.

### 5.2 The desk and scorer pages (`lib/v1/apps/attendance-desk.js`, `scorer.js`)

Each file gets a **local** (not exported, not imported) helper:

```js
// "Ana (Gate A)", "Ana", "Gate A", or '' — who this link was issued to (link-attribution-spec.md §1 D13).
function issuedText(info){
  const to = typeof info.to === 'string' ? info.to : '';
  const note = typeof info.note === 'string' ? info.note : '';
  return to && note ? `${to} (${note})` : (to || note);
}
```

- Desk page: `#dayLabel` becomes `` ` · ${dayEntry.label || info.day}` `` plus
  `` ` · ${issuedText(info)}` `` when non-empty.
- Scorer page: the same, appended after the facility in the existing
  `dayLabelEl.textContent = …` line.

`textContent` only, never `innerHTML`: the text came from a person.

### 5.3 Control Center (`tools/control-center.html`)

**Markup.** Above `#scorerLinkBtn`, and above `#attendanceDeskBtn`, add the same kind
of block (ids differ):

```html
<div class="link-issue-fields" id="scorerLinkFields" style="display:none;">
  <input id="scorerLinkTo" type="text" class="organizer-text-input" maxlength="40" placeholder="Issued to (optional)" autocomplete="off" aria-label="Issued to, optional" />
  <input id="scorerLinkNote" type="text" class="organizer-text-input" maxlength="40" placeholder="Note, e.g. Courts 3–4 (optional)" autocomplete="off" aria-label="Note, optional" />
</div>
```

The desk one is `#deskLinkFields`, `#deskLinkTo`, `#deskLinkNote`, with the placeholder
`Note, e.g. Gate A (optional)`. Show and hide each block wherever its button is shown
and hidden (find every `scorerLinkBtn.style.display` / `attConsoleDeskBtn.style.display`
assignment and mirror it). A small `.link-issue-fields` rule: the two inputs side by
side with a gap, stacking under 480px, margin below. Use the existing tokens on
`:root`.

**Issuing.** In `issueScorerLink()` and `attConsoleIssueDeskLink()`:

- Read both inputs, `.trim()`. Build `fields` with only the non-empty ones.
- `fetch` with `headers: { 'Authorization': …, 'Content-Type': 'application/json' }`
  and `body: JSON.stringify(fields)`.
- Fixture mode: build the payload `{ exp, scope, day }` plus `to`/`note` when given,
  and encode it as UTF-8 base64url (`TextEncoder`, then the bytes to a binary string,
  then `btoa`, then the URL-safe replacements). The desk fixture token becomes a real
  payload too (scope `'attendance-desk'`), plus `'.fixture'`, so the desk page can show
  the label locally. `issued.issuedTo`/`issued.note` set from `fields` (or `null`).
- On success (after the epoch check passes and the box renders), clear both inputs.

**Rendering.** In `renderScorerLink` and `attConsoleRenderDeskLink`, the title becomes
`Scorer link for <day> · <label>` / `Desk link for <day> · <label>` when the response
has `issuedTo` or `note` (label as `issuedText`, written inline), via `textContent`.

### 5.4 `_tests/`

- `_tests/unit/data/data-modules.test.mjs`: `scorerDecode` and `deskDecode` of a
  payload with `to: 'Niño'` and `note: 'Courts 3–4'` return them intact; an old-shape
  token still decodes.
- Run `cd sage-match-control.github.io/_tests && npm run verify`. The comparison
  harness must show no difference on any page (no fixture token carries a label).

---

## 6. Checking it yourself

1. `npm run verify` in `sage-tools-api`: passes.
2. `npm run verify` in `sage-match-control.github.io/_tests`: passes.
3. Serve the site locally. Open Control Center with `?fixture=pre` on an event with
   desks and one with scorer links in `_fixtures/config.json`. Issue each link with
   `Niño` and `Gate A`: the title reads `… · Niño (Gate A)`, the inputs clear. Open
   the link: the page's day line ends ` · Niño (Gate A)`. Issue without either:
   no label anywhere.
4. `grep -rn "A1:G" sage-tools-api/src`: only the reconcile read and the header write
   remain.

---

## 7. Documentation

All present tense.

| File | Add |
|---|---|
| `sage-docs/docs/features/control-center.md` | Where desk and scorer links are issued: the optional **Issued to** and **Note**, what they're for, where they show up (the page, `markedBy`, the score notes), and that they record who the link was given to, not who typed (§1 D4) |
| `sage-docs/docs/features/event-attendance.md` | The `markedBy` column: what it holds, that it's the last change only, how the API picks its column and never overwrites one of yours, `Control Center` for operator marks |
| `sage-docs/docs/features/score-entry.md` | The note on both score cells: the line format, last five, a person's own note kept, `Control Center` for operator saves, notes never reach the public pages |
| `sage-docs/docs/features/scorer-page.md` | The label after the day |
| `sage-docs/docs/technical/auth.md` | The `to`/`note` payload keys, signed, 40 characters, absent when not given; links issued earlier verify unlabelled |
| `sage-docs/docs/technical/api.md` | The two `/v3` issue routes' optional body and the echoed fields; `/v1` unchanged |
| `sage-docs/docs/technical/event-attendance.md` | `markedByColumn`'s rule, the widened allowlist (one cell in H–Z, not J), why `parseTab` stays strict |
| `sage-docs/docs/technical/control-center.md` | The issue fields, fixture tokens with a real payload |
| `sage-docs/docs/technical/scorer-page.md` | The label in the day line; UTF-8 decoding |
| `sage-docs/docs/usage/run-the-day.md` | When issuing links, put the person's name and their gate or courts, so the sheet shows who did what |
| `sage-docs/docs/usage/desk-handout.md`, `scorer-handout.md` | One sentence: the page shows who the link was issued to; don't pass it on |
| `sage-docs/docs/changelog.md` + `mkdocs.yml` | A MINOR entry and version bump, per the changelog's "Releasing a version" |
| `D:\Personal\SAGE\CLAUDE.md` | In the `src/attendance/` layout entry, the write rule becomes "**only** inside `ATTENDANCE!A:G` plus one `markedBy` cell in H–Z (never J), and, for score entry, one match's two `SCHEDULE` score cells and their note". In `src/auth/`, scoped tokens may carry `to` and `note` |

Do **not** move this spec out of `not-started/` or edit its status line: the owner
does that after §10.2.

---

## 8. What doesn't change

Every `apps-script/*.gs` file and both masters; `HEADERS` and new tabs' layout
(`createAttendanceTab`); the reconcile; the "Not yet in" list; every `/v1` response
body; the mark and score response bodies; the snapshot format and the sync; the live
Worker; `events.json`; the operator login.

---

## 9. Alternatives considered (for the record; don't build these)

| Alternative | Why not |
|---|---|
| Store issued links (name, note) server-side and look them up by ID | Needs a store the API doesn't have; the signed payload carries the same data, and can't be edited |
| `markedBy` fixed in column H | H onward belongs to the organizer's own columns; a fixed H would overwrite one |
| Add `markedBy` to `HEADERS` and to both generators | Every existing tab would fail the strict header check, and both masters would need re-pasting. Claiming a column lazily needs neither |
| Write the score and its note in one `updateCells` call | Atomic, but replaces the tested `values:batchUpdate`/`batchClear` path that has yet to run against real workbooks. A note is attribution; it must never put a score at risk |
| Drive API comments instead of notes | Comments can't be reliably anchored to a cell through the API |
| An operator name field in Control Center | Decided against (§1 D3): every operator write is `Control Center` |
| Notes in the published snapshot | Who entered a score is for operators, not the public |

---

## 10. Finishing

### 10.1 Report back

1. `git status --short` from each of the four repos.
2. `npm run verify`'s summary line from `sage-tools-api` and from the site's `_tests/`.
3. §6's checks, each ticked or explained.
4. Anything in this spec you found wrong or had to interpret, with what you did.

### 10.2 The owner's checks (not yours)

On a no-traffic Cloud Run revision (`--no-traffic --tag link-attribution`) with a
scratch event whose facilities are **copies** of each master:

| # | Check | Pass |
|---|---|---|
| L1 | Issue a desk link with a name and note, mark someone in, then out | `markedBy` header claimed at H; the row reads `In · …`, then `Out · …`; times in Manila |
| L2 | The same on a copy whose `ATTENDANCE` has a `TShirt Size` column in H | `markedBy` lands in I; H untouched |
| L3 | Issue a scorer link with a name; save a score, correct it, clear it | both score cells show the same note, three lines, newest first; the published snapshot is unaffected |
| L4 | Add a note by hand to a score cell, then save from the scorer page | the hand-written text is still there, below the score-entry lines |
| L5 | Save from Control Center | the line reads `Control Center` |
| L6 | Save six times on one match | five score-entry lines remain |
| L7 | Open an old-shape (pre-3.1.0) scorer link | works; no label on the page; notes read `Scorer link` |
