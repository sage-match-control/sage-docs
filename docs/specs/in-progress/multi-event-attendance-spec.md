# Spec — Attendance for every event

Staff check-in for any registered event, marked in Control Center or on a
desk page, written by `sage-tools-api` straight into each facility workbook's
`ATTENDANCE` tab. One check-in per person covers every category they play.

> **Status: in progress.** Phases 1 and 3 to 7 are built and tested on the
> `attendance` branches, not merged or deployed. Phase 2 (the owner's checks on
> Google's side, §8) and Phase 8 are open. Decisions settled with the owner on
> 2026-10-02 (§1). Written against `sage-tools-api` 2.5.0,
> `sage-match-control.github.io` at `89a7476` and `event-data`'s
> `config/events.json` as of that date.
>
> **Phase 2 (§9) is the owner's, and it gates everything after it.** It
> proves on Google's side what this design assumes. If a result in §8.4
> contradicts the spec, stop and report. Do not work around it.
>
> Builds on [Event attendance](../implemented/event-attendance-spec.md), the
> Pickle for Sight version (one event, an Apps Script web app per workbook).
> That event keeps its own page and script; nothing here changes it.

---

## 0. Read this first (implementer orientation)

Assume no knowledge of this project beyond this page. Read all of §0 to §7
before writing code.

### 0.1 The system in one paragraph

SAGE runs pickleball tournaments. Each **event** (e.g. `piggleball-2026`) has
one or more **days** (`piggleball-day1`), and each day has one or more
**facilities** (venues). Every facility-day has its own **Google Sheets
workbook**, where operators type scores. An Apps Script trigger in each
workbook calls `sage-tools-api` (Node/Express on Google Cloud Run) on every
edit. The API reads the workbook's `CSV` (matches) and `STANDINGSCSV` tabs and
publishes a JSON snapshot that the public pages and **Control Center** (the
operators' console, `tools/control-center.html`) display. The registry of
events, days, facilities and their sheet IDs is `config/events.json` in the
`event-data` GitHub repo. The API fetches it at runtime and caches it for about
a minute.

### 0.2 Repos

`D:\Personal\SAGE` is a plain folder holding four git repos:

| Repo | What changes in it |
|---|---|
| `sage-tools-api/` | the attendance module, auth, config validation, a sync hook, routes, tests, and a one-off check service (§8) |
| `sage-match-control.github.io/` | an `ATTENDANCE CLIENT` code block, the Control Center **Attendance** tab, an attendance page template, fixtures |
| `event-data/` | `config/README.md` documents the new field. **Never commit to `config/events.json`.** That is production config, and the owner edits it |
| `sage-docs/` | feature and technical pages, this spec's status, the runbooks |

### 0.3 Rules that apply to every change (from the root `CLAUDE.md`)

- **Never deploy, and never push to `main` on an event day or in the days
  before one.** Pushing `sage-tools-api`'s `main` deploys Cloud Run
  automatically, and pushing the site's `main` publishes the site. Work on a
  branch named `attendance` in each repo. The owner merges.
- **Never write to a real event workbook, and never call the production API
  from a script.** Your tests use fakes (§7). Real-Google checks are the
  owner's (§8).
- `sage-tools-api`: ESM `.mjs`, classes with constructor injection, no
  TypeScript, no build step. Bump `package.json`'s version for any change to
  what runs on Cloud Run (this spec is a minor bump: `2.5.0` → `2.6.0`), and
  add a matching entry to `README.md`'s **Changelog**.
- No new **runtime** dependency in `sage-tools-api`. Google APIs are called
  with the global `fetch`, as the sync's fetchers already do.
- Site pages are self-contained HTML: one inline `<style>`, one classic inline
  `<script>`, no bundler. The only external scripts allowed are Google Fonts
  and the one QR library named in §6.6.
- Working copies use CRLF line endings. Keep them.
- Cite a spec from outside `sage-docs` as
  `sage-docs/docs/specs/.../multi-event-attendance-spec.md`, with literally
  `...` where the status folder goes.
- Documentation is written in the present tense: what the system does, not
  what changed.
- A block marked `// ==== <NAME> — identical in …` must be byte-identical in
  every file that carries it. This spec adds one, `ATTENDANCE CLIENT` (§6.1).

### 0.4 Facts checked against the code (2026-10-02)

| Fact | Where |
|---|---|
| The sync fetches each facility as `{ name, matchesCsv, standingsCsv }` (CSV **text**) through `SheetsCsvFetcher` (API key, Sheets API `values:batchGet`) | `src/sync/SheetsCsvFetcher.mjs`, `SyncService.syncDay` |
| `SyncService.syncDay` publishes, then archives, then returns. Nothing runs after the response, because Cloud Run throttles CPU once a response is sent | `src/sync/SyncService.mjs` header comment and lines 121–185 |
| `facilityCompletion.mjs` exports a CSV parser, `parseCsv(text)` → `string[][]` | `src/sync/facilityCompletion.mjs:15` |
| `AuthService.verify(token)` accepts **any** validly signed, unexpired token. Its payload is only `{ exp }` | `src/auth/AuthService.mjs:46-62` |
| Auth checks live as closures inside the sync routes factory (`requireAuthToken`, `requireSyncSecretOrAuthToken`) | `src/sync/routes.mjs` |
| CORS allows only `GET, POST, OPTIONS`, so a browser `PUT` fails its preflight today | `src/server/Server.mjs`, `#registerMiddleware` |
| Config validation is the `validate(raw)` function in `SyncConfigStore.mjs`. `SyncConfigSnapshot.getDay(day)` returns `{ day, event, label, facilities, isLive }`. It doesn't return `date` or the event's `type` | `src/sync/SyncConfigStore.mjs:224`, `src/sync/SyncConfigSnapshot.mjs` |
| Events carry `type` (`"dual-meet"`, `"standard"`, `"team"`), optional `archived`, `display` labels. Days carry an optional `date` (`YYYY-MM-DD`). Every current event's day has one, but BKL Cup's (archived) do not | `event-data/config/events.json`, `event-data/config/README.md` |
| Standard team codes look like `NMD_1`, dual meet like `PNF_LIWD_1`. `STANDINGSCSV`'s header is `teamCode,player1,player2,wins,loss,quotient,bracket` for both. Names can carry trailing spaces | published snapshots in `event-data` |
| A team event's `STANDINGSCSV` is `teamCode,teamName,totalPoints,…` with **no player columns**. Its players are in a tab named **`Teams`**, one row per player. The name is in the column headed **`FINAL LEVEL ORDER`** (confirmed by the owner). The other headers include `Team Code`, `Team Name`, `LEVEL`, `Gender` | PickleDrive's workbook |
| Google's gviz CSV export can return a column **empty** when it guesses that column's type wrongly. PickleDrive's `NAMES` column comes back empty that way. The Sheets API (`values.get`) does not do this | observed 2026-10-02 |
| Pickle for Sight's `attendance.gs` skips standings rows whose code doesn't match `^[A-Z0-9]+_\d+$` (playoff-seat rows like `HIMD_QF_1` repeat names), and names that equal the team code | `scripts/attendance.gs:78-97` |
| Control Center: view tabs are `<button class="view-tab" data-view="…">` in `#viewTabsWrap`, in the order Mission Control, Awards, Live Matches, Match Finder, Standings. The console opens on Mission Control. **`showView(view)` is the one place that switches views**: it marks the tab active, shows that view's container and hides the rest, and renders it. Sign in is at the top of Mission Control. The operator token lives in `sessionStorage` under `sage.authToken`, read with `currentAuthToken()`. API calls use `CLOUD_RUN_BASE_URL`. The selected event's raw `events.json` entry is in `EVENTS_REGISTRY.get(CURRENT_EVENT_KEY)`, and its type is `CURRENT_TYPE` | `tools/control-center.html`; find each by name (`grep -n`), not by line number |
| Control Center shows outcomes two ways. **`showToast(kind, text, key)`** (`kind` is `ok`, `warn`, `error` or `loading`) for short outcomes, pinned to the top of the screen, a later toast with the same `key` replacing the last. **`showResultRows(box, kind, title, rows)`** for results worth reading, as labelled rows in a box right under the button, scrolled into view (`rows: [{ label, value, kind?, list? }]`). **`friendlyApiMessage(raw)`** turns an API error string into plain words, including any `HTTP <status> <json>` inside it. **No raw JSON is ever shown**. Every finished result box has a close button, and `clearResultBoxes()` hides every `.organizer-result` when the event, day or tab changes; a result that arrives after that (its box's `loading` message came first) becomes a toast. Attendance's boxes use the same helpers, so they get all of this for free: give each box the `organizer-result` class and an `id` | `tools/control-center.html`, the "Showing outcomes" and "Readable API messages" blocks |
| On `localhost`, Control Center's `?fixture=<name>` loads `/_fixtures/config.json` and `/_fixtures/<event>/<name>.json` instead of the published data (`const FIXTURE`) | `tools/control-center.html`, `const FIXTURE` |
| Cloud Run's URL `sage-tools-api-811926984834.us-central1.run.app` puts the project **number** at `811926984834`. The project **ID** is not recorded anywhere in the repos | `tools/control-center.html`, `const CLOUD_RUN_BASE_URL` |

### 0.5 Glossary

- **Person key:** a person's name, normalized (§3.3). Two roster entries with
  the same key are the same person.
- **Roster:** everyone who plays at one facility on one day, built from the
  workbook (§3.4).
- **Roster update (reconcile):** making the `ATTENDANCE` tab match the roster
  (§4.6).
- **Mark:** setting one person present or not present (§4.7).
- **Operator token:** what `POST /auth/login` returns to Control Center.
- **Desk token / desk link:** a token that allows marking attendance only, on
  one day (§4.3). The link is the desk page URL carrying it.

---

## 1. Decisions (settled by the owner, 2026-10-02)

| # | Decision |
|---|---|
| D1 | Attendance is a per-event setting in `events.json`: `"attendance": "console"` (operators mark in Control Center), `"attendance": "desks"` (also desk links and the event's attendance page), or absent (no attendance) |
| D2 | **One check-in per person** covers all their categories that day at that venue. `ATTENDANCE` holds one row per person |
| D3 | Writes go through `sage-tools-api` using the Google Sheets API as a **dedicated service account**. No per-workbook Apps Script, and `attendance.gs` is retired for new events |
| D4 | Reads come from the `ATTENDANCE` tab's CSV export, polled. The live Worker (`sage-live`) is **not** used: its channel is public |
| D5 | The roster refreshes itself after every sync: missing people are added, changes applied, leavers flagged |
| D6 | Besides signed-in operators, only **desk links** can mark: scoped to one day, expiring at the end of that day, and only when the event is in `"desks"` mode. Switching the event to `"console"` stops every outstanding desk link |
| D7 | A person no longer on the roster keeps their row and any check-in, is flagged `withdrawn`, and is hidden from the desk list |
| D8 | Standard and dual-meet rosters come from `STANDINGSCSV`; team-event rosters come from the `Teams` tab's `FINAL LEVEL ORDER` column |
| D9 | Same person = same normalized name. Near-identical spellings are **listed** in Control Center as possible duplicates, never merged automatically |
| D10 | Built **before** the `sage-tools-api` test suite and architecture Phase 2, with its own tests from day one, laid out the way those specs expect (§7.1, §11) |
| D11 | Both the desk page and Control Center can **filter** the list to one category (or team) and **jump** to a category without filtering (§6.1) |

---

## 2. Overview

```
Control Center "Attendance" tab          events/<key>/attendance  (desk page, "desks" mode only)
        │   the same ATTENDANCE CLIENT block in both (§6.1)
        │
        │ read:  ATTENDANCE tab CSV export, every 10 s while visible
        │ write: PUT /v1/days/{day}/facilities/{facility}/attendance/{key}
        │        Authorization: Bearer <operator token | desk token>
        ▼
sage-tools-api   src/attendance/
        │ reads:  Sheets API with the existing API key (GOOGLE_SHEETS_API_KEY)
        │ writes: Sheets API with the service account's token (metadata server)
        ▼
Facility workbook ── ATTENDANCE tab (one row per person)
        ▲
        │ after every sync of a facility: roster update (only when the roster changed)
POST /sync/:day (Apps Script, unchanged) ── SyncService ── onFacilitiesSynced hook
```

**Unchanged:** `sheets-sync.gs`, the generators, the sync's publishing (live
Worker and GitHub), the snapshot format, every existing route and response,
Pickle for Sight's page and `attendance.gs`.

---

## 3. Data model

### 3.1 The `events.json` setting

At the **event** level, beside `type` and `title`:

```jsonc
"piggleball-2026": {
  "type": "standard",
  "title": "Piggleball Chairman's Cup",
  "attendance": "desks",          // "console" | "desks"; omit for no attendance
  "days": { "piggleball-day1": { "label": "Oct 3", "date": "2026-10-03", ... } }
}
```

Validation (§4.2): if present, it must be exactly `"console"` or `"desks"`. A
day used with desk links must have a `date`.

### 3.2 The `ATTENDANCE` tab (one per facility workbook)

Row 1 is this exact header, columns A–G. Data starts at row 2, one row per
person.

| Col | Header | Type | Written by | Meaning |
|---|---|---|---|---|
| A | `key` | text | roster update | the person key (§3.3). Identifies the row |
| B | `player` | text | roster update, once | display name: the first spelling seen, trimmed, inner whitespace collapsed |
| C | `teams` | text | roster update | the person's team codes, joined with `", "`, in roster order: `NMD_1, MXD_3` |
| D | `categories` | text | roster update | the category of each team in C, **same length and order**: `NMD, MXD`. Blank for team events |
| E | `present` | boolean | mark | `TRUE` / `FALSE`. Never blank |
| F | `timeIn` | text | mark | first check-in time, `yyyy-MM-dd HH:mm` in Asia/Manila, or `""` |
| G | `withdrawn` | boolean | roster update | `TRUE` when the person is no longer on the roster. Never blank |
| H+ | anything | | **nobody** | owner-added columns (e.g. `TShirt Size`). The API never reads or writes past G |

Rules:

- **Types never mix within a column** (§0.4's gviz pitfall): E and G are always
  booleans, A–D and F always strings. Write with `valueInputOption: "RAW"`.
  A JSON string stays a string even when it looks like a number or a date, and
  a JSON boolean becomes a checkbox-style `TRUE`/`FALSE`.
- **The API refuses to touch a tab whose A1:G1 is not exactly this header**
  (error `AttendanceLayoutError`, §4.8). This protects Pickle for Sight's
  old-format tab and any tab someone has rearranged.
- Rows with a blank `key` are ignored. Rows are only ever added below the last row or updated
  in place, never inserted or deleted by the API.
- If the tab does not exist, the first roster update creates it: the
  `addSheet` request, the header row, and a frozen row 1 (§4.5).

### 3.3 Person key

One function, `src/attendance/personKey.mjs`. It is the only definition: the
client never recomputes keys and only reads column A.

```js
/** The identity of a person: their name with spacing, case and accents ignored. */
export function personKey(name) {
    return String(name ?? "")
        .normalize("NFD").replace(/[\u0300-\u036f]/g, "")   // strip accents: José -> Jose, Ñino -> Nino
        .replace(/\s+/g, " ")
        .trim()
        .toLowerCase();
}

/** The name as shown: trimmed, inner whitespace collapsed, case and accents kept. */
export function displayName(name) {
    return String(name ?? "").replace(/\s+/g, " ").trim();
}
```

| Input | `personKey` | `displayName` |
|---|---|---|
| `"Juan Dela Cruz "` | `juan dela cruz` | `Juan Dela Cruz` |
| `"JUAN  dela cruz"` | `juan dela cruz` | `JUAN dela cruz` |
| `"José Rizal"` | `jose rizal` | `José Rizal` |
| `"  "` | `""` (not a person) | `""` |

### 3.4 Roster, by event type

`src/attendance/roster.mjs`, pure functions. Each returns
`Person[] = { key, player, teams: string[], categories: string[] }[]`, one
entry per distinct key, in first-seen order. A person's `teams` and
`categories` keep first-seen order, with no duplicate team codes.

**Standard** (`type: "standard"`): from `STANDINGSCSV` rows (`string[][]`,
header first). The sync hook has CSV text and passes `parseCsv(standingsCsv)`,
using `parseCsv` from `src/sync/facilityCompletion.mjs`. A manual roster update
reads the tab with the Sheets API and passes its values.
- Find columns by header name (trimmed): `teamCode`, `player1`, `player2`.
  Missing any of them is an error naming the missing header.
- Skip a row unless its trimmed `teamCode` matches `^[A-Z0-9]+_\d+$`. That
  skips playoff-seat rows such as `HIMD_QF_1`, as `attendance.gs` does.
- For each of `player1` and `player2`: skip it if `displayName` is empty,
  equals the team code, or matches `/^bye$/i`.
- Category = the team code before the first `_` (`NMD_1` → `NMD`).

**Dual meet** (`type: "dual-meet"`): the same, except the code pattern is
`^[A-Z0-9]+_[A-Z0-9]+_\d+$` and the category is the **middle** segment
(`PNF_LIWD_1` → `LIWD`).

**Team** (`type: "team"`): from the `Teams` tab's values (a 2-D array from the
Sheets API, §4.5).
- The header row is the **first row among the first 5** that contains a cell
  equal (trimmed) to `Team Code` and a cell equal (trimmed) to
  `FINAL LEVEL ORDER` or `Player`. Prefer `Player` if both exist. It is
  accepted so a future template can rename the column. No such row is an
  error: `Teams tab has no "Team Code" and "FINAL LEVEL ORDER" header row`.
- Each later row with a non-empty `displayName` in the name column and a
  non-empty trimmed `Team Code` is a player. `teams = [teamCode]` and
  `categories = []`.

**Any other or missing type** is an error: `event has no attendance roster
rule for type "<type>"`.

**Same name twice in one category.** For standard and dual meet: if one key
ends up with two different team codes that share a category, keep both team
codes (they stay one row, by D9), and include the key in the result's
`warnings` list (`{ key, kind: "same-name-same-category", teams }`). The
function returns `{ people, warnings }`. Control Center lists these (§6.3).

### 3.5 Possible duplicates (Control Center only)

Computed in the browser from the `ATTENDANCE` rows (§6.1,
`possibleDuplicates`). Two keys are a possible duplicate pair when:
- they are equal after removing every character that isn't a letter or
  digit (`dela cruz` / `delacruz`), or
- both are at least 5 characters long and their Levenshtein distance is 1.

They are listed, never merged. The fix is to correct the spelling in the
workbook's category tab, and the next sync's roster update follows.

---

## 4. `sage-tools-api`

### 4.1 Files

| File | New/changed | Holds |
|---|---|---|
| `src/attendance/personKey.mjs` | new | §3.3 |
| `src/attendance/roster.mjs` | new | §3.4: `rosterFromStandings(type, rows)`, `rosterFromTeamsValues(values)` |
| `src/attendance/attendanceTab.mjs` | new | pure: `HEADERS`, `parseTab(values)`, `planReconcile(people, tab)`, `planMark(row, present, now)`, `formatTimeIn(date)` (§4.6–4.7) |
| `src/attendance/GoogleAccessToken.mjs` | new | the service account's access token and email from the metadata server (§4.4) |
| `src/attendance/SheetsClient.mjs` | new | Sheets API reads (API key) and writes (token), restricted to an allowlist (§4.5) |
| `src/attendance/AttendanceService.mjs` | new | `mark`, `reconcileDay`, `reconcileAfterSync`, `issueDeskLink` |
| `src/attendance/routes.mjs` | new | the `/v1` routes and their `@openapi` blocks (§4.8) |
| `src/shared/errors.mjs` | changed | new error classes (§4.8) |
| `src/auth/AuthService.mjs` | changed | desk tokens; operator `verify` rejects them (§4.3) |
| `src/sync/SyncConfigStore.mjs` | changed | validate `attendance` (§4.2) |
| `src/sync/SyncConfigSnapshot.mjs` | changed | `getEvent(eventKey)`; `getDay` also returns `date` (§4.2) |
| `src/sync/SyncService.mjs` | changed | the `onFacilitiesSynced` hook (§4.9) |
| `src/server/Server.mjs` | changed | CORS adds `PUT`; mount `/v1` (§4.10) |
| `src/docs/openapiSpec.mjs` | changed | add `src/attendance/routes.mjs` to `apis`, add tag `attendance` |
| `index.mjs` | changed | wiring (§4.10) |
| `.env.example` | changed | document `GOOGLE_ACCESS_TOKEN` (dev only) |
| `Dockerfile` | changed | the deploy comment gains `--service-account` (§8.3) |
| `package.json`, `README.md` | changed | version `2.6.0`, `test` scripts (§7.1), Changelog |
| `test/…` | new | §7 |
| `spikes/sheets-write-check/` | new | the owner's check service (§8). Not part of the Cloud Run image: the `Dockerfile` copies only `index.mjs`, `src` and `templates` |

### 4.2 Config

In `SyncConfigStore.mjs`'s `validate(raw)`, inside the per-event loop, add:

```js
if (eventEntry.attendance !== undefined && eventEntry.attendance !== "console" && eventEntry.attendance !== "desks") {
    throw new Error(`event "${eventKey}" has an invalid attendance value (must be "console" or "desks")`);
}
```

In `SyncConfigSnapshot.mjs`:

```js
// The event-level settings attendance needs. Throws UnknownEventError (404) for an unknown key.
getEvent(eventKey) {
    const entry = this.events[eventKey];
    if (!entry) throw new UnknownEventError(eventKey);
    return { event: eventKey, type: entry.type ?? null, attendance: entry.attendance ?? null };
}
```

and add `date: found.entry.date ?? null` to `getDay`'s returned object. It's
an additive field; existing callers are unaffected. Document the field in
`event-data/config/README.md` beside `type`.

### 4.3 Desk tokens (`AuthService`)

Operator tokens keep their exact format (`<base64url {exp}>.<hmac>`). Add:

```js
// A desk token allows exactly one thing: marking attendance on one day.
// Signed with a key derived from AUTH_TOKEN_SECRET, so a desk token's
// signature never verifies as an operator token, and the payload carries a
// scope that verify() rejects as a second guard.
issueDeskToken({ day, expiresAt }) -> { token, expiresAt }   // payload { exp, scope: "attendance-desk", day }
verifyDeskToken(token) -> { day, exp } | null                // null on bad signature, wrong scope, expired, malformed
```

- Derived key: `createHmac("sha256", this.tokenSecret).update("attendance-desk").digest()`,
  then sign with `createHmac("sha256", derivedKey).update(payload)`.
- `verify(token)` (operator) additionally returns `false` when the parsed
  payload has any `scope` property.
- Both return `null`/`false` when `tokenSecret` is unset (fail closed).

**Expiry** is computed by the service, not `AuthService`: the end of the day's
`date` in Manila time, `Date.parse(`${date}T23:59:59.999+08:00`)`. Manila has no
daylight saving, so the fixed offset is exact.

### 4.4 The service account's token (`GoogleAccessToken`)

```js
export class GoogleAccessToken {
    constructor({ fetchImpl = fetch, override = null, logger }) {}
    async token()   // -> string; cached until 60 s before expiry
    async email()   // -> string | null; for error messages ("share it with <email>"); cached
}
```

- `token()`: `GET http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token?scopes=https://www.googleapis.com/auth/spreadsheets`
  with header `Metadata-Flavor: Google`. The response is
  `{ access_token, expires_in }`. Cache it until
  `now + (expires_in - 60) * 1000`.
- `email()`: `GET …/service-accounts/default/email` with the same header,
  returning plain text. Return `null` on failure, never throw.
- `override`: when `GOOGLE_ACCESS_TOKEN` is set (local development only;
  §4.10), `token()` returns it and the metadata server is never called.
- A failure to get a token throws `UpstreamError("Could not get a Google
  access token: <detail>")` (502).

Whether the metadata server honours `?scopes=` on Cloud Run is **check C1**
(§8). If it does not, stop: the fallback (IAM Credentials
`generateAccessToken` with the account impersonating itself) needs an IAM
grant the owner has to decide on.

### 4.5 `SheetsClient`

```js
export class SheetsClient {
    constructor({ apiKey, accessToken /* GoogleAccessToken */, fetchImpl = fetch, timeoutMs = 5000, logger }) {}

    // Reads, with the API key (the workbooks are link-viewable; the sync already relies on it).
    async readValues(sheetId, range, { render = "UNFORMATTED_VALUE" } = {})  // -> any[][]; [] when the range is empty; null when the TAB doesn't exist

    // Writes, with the service-account token. Every range must pass the allowlist check first.
    async updateValues(sheetId, data /* [{ range, values }] */)              // one values:batchUpdate
    async createAttendanceTab(sheetId)                                        // addSheet + header + frozen row 1
}
```

**Endpoints** (base `https://sheets.googleapis.com/v4/spreadsheets`):

| Method | Request |
|---|---|
| `readValues` | `GET /{id}/values/{range}?key=…&valueRenderOption=…`. `200 { values? }` gives `values ?? []`. A `400` whose message contains `Unable to parse range` means the tab is missing, so return `null` |
| `updateValues` | `POST /{id}/values:batchUpdate`, body `{ valueInputOption: "RAW", data }` |
| `createAttendanceTab` | `POST /{id}:batchUpdate` with `{ requests: [{ addSheet: { properties: { title: "ATTENDANCE", gridProperties: { frozenRowCount: 1 } } } }] }`, then `updateValues` with `ATTENDANCE!A1:G1` = `[HEADERS]` |

Writes send `Authorization: Bearer <await accessToken.token()>`.

**The allowlist.** Before any write request, every range must parse as
`ATTENDANCE!<col><row>[:<col><row>]` with columns within A–G. Anything else
throws `Error("SheetsClient: write outside the allowlist: <range>")` before
the request is made. This is the guarantee that a bug in attendance can never
overwrite scores or formulas. `createAttendanceTab` is allowed because its
only title is `ATTENDANCE`.

**There is no append.** The Sheets API's `values:append` with `INSERT_ROWS`
finds "the table" by itself, and after a blank row it can insert rows in the
middle of the tab. That would shift row numbers under a mark that is in
flight. New rows are written to explicit row numbers with `updateValues`
instead (§4.6), so the API never inserts, deletes or moves a row.

**Errors** (all calls):
- A timeout (`AbortSignal.timeout(timeoutMs)`) or network error throws
  `UpstreamError("Google Sheets: <detail>")` (502).
- `429` waits 1000 ms and retries once. If it fails again, it throws
  `ServiceBusyError("Google Sheets is busy — try again in a moment")` (503).
- `403` on a **write** throws `UpstreamError("The API can't edit this workbook. Share it with <email> as Editor.")`,
  with `<email>` from `accessToken.email()`, or "the API's service account" when
  that is null.
- Any other non-2xx throws `UpstreamError("Google Sheets: HTTP <status> <message>")`.
- `<message>` is always words, never the raw body: Google's error body is
  JSON (`{ "error": { "code", "message", "status" } }`), so take
  `error.message`. If the body isn't JSON or has no message, use the HTTP
  status text alone. Every error the attendance routes return is a
  sentence a person can read in Control Center or on the desk page.

### 4.6 Roster update (reconcile)

**Pure planning,** `attendanceTab.mjs`:

```js
export const HEADERS = ["key", "player", "teams", "categories", "present", "timeIn", "withdrawn"];

// values: what readValues("ATTENDANCE!A1:G") returned (UNFORMATTED_VALUE: booleans arrive as true/false).
// Throws AttendanceLayoutError if row 1 is not exactly HEADERS.
// lastRow = values.length (the API trims trailing empty rows; an empty row in the middle arrives as []).
export function parseTab(values) -> { rows: [{ row /* 1-based sheet row */, key, player, teams, categories, present, timeIn, withdrawn }], lastRow }

export function planReconcile(people, tab) -> {
    data: [{ range, values }],      // every change, for ONE SheetsClient.updateValues call
    newKeys: string[],              // keys written to new rows, in order
    counts: { added, updated, withdrawn, restored, duplicatesCleared }
}
```

`planReconcile`, in this order:
1. **Duplicates.** The first row per key is canonical. For each later row
   with the same key: merge into the canonical row (`present` = either;
   `timeIn` = the earlier non-empty string). The canonical row's E:F are
   written if they changed. The duplicate row's A:G are blanked (`""` in A–D
   and F, `false` in E and G). Count `duplicatesCleared`.
2. **Every roster person:**
   - No row: write `[key, player, teams.join(", "), categories.join(", "), false, "", false]`
     to the next new row: `tab.lastRow + 1`, then `+ 2`, and so on, as
     `ATTENDANCE!A<n>:G<n>`. Count `added`.
   - A row whose C or D differs from the roster's joined values: update C:D.
     Count `updated`.
   - A row with `withdrawn` true: set G to `false`. Count `restored`.
   - Never touch B, E or F of an existing row.
3. **Every non-blank row whose key is not on the roster** and is not already
   withdrawn: set G to `true`. Count `withdrawn`. Leave C, D, E and F as they
   are.

Ranges are single rows (`ATTENDANCE!C7:D7`, `ATTENDANCE!G7`,
`ATTENDANCE!A9:G9`). Everything goes in **one** `updateValues` call, so a
reconcile is at most one write request, and usually none.

**Orchestration,** `AttendanceService`:

```js
// Called by the sync hook. Never throws: logs and returns.
async reconcileAfterSync({ day, event, facilities /* [{ name, sheetId, standingsCsv }] */ })

// Called by POST …/attendance/reconciliations. force = true skips the unchanged-roster shortcut.
async reconcileDay(day, { facilityName = null, force = true } = {}) -> { day, facilities: [ result ] }
// result: { name, ok: true, skipped, added, updated, withdrawn, restored, duplicatesCleared, warnings }
//       | { name, ok: false, error }
```

For each facility, in parallel (`Promise.allSettled`):
1. `getEvent(event)`. If `attendance` is null, do nothing. `reconcileAfterSync`
   returns at once without any read, so events without attendance pay nothing.
2. **Build the roster:**
   - Standard or dual meet: from the `standingsCsv` the sync already has.
     `reconcileDay` reads it with `readValues(sheetId, "<standingsSheetName>!A1:Z", { render: "FORMATTED_VALUE" })`,
     using `config.sheetsFor(day).standingsSheetName`, and turns the values into
     the same rows.
   - Team: `readValues(sheetId, "Teams!A1:Z", { render: "FORMATTED_VALUE" })`.
     Inside `reconcileAfterSync`, cache that per `sheetId` for **5 minutes**, so
     a busy scoring workbook doesn't re-read `Teams` on every sync.
     `reconcileDay` always re-reads it.
3. **Unchanged-roster shortcut** (`reconcileAfterSync` only): fingerprint the
   roster (`sha1` of `JSON.stringify(people)`). If it equals this instance's
   last **successful** reconcile for `day|facility`, skip the facility:
   `skipped: true`, no reads, no writes. Scores change standings rows but not
   names, so most syncs stop here.
4. Read `ATTENDANCE!A1:G` (API key, `UNFORMATTED_VALUE`). If it's `null` (no
   tab), call `createAttendanceTab` and treat the tab as empty.
5. `parseTab`, then `planReconcile`, then `updateValues(data)` if `data` is
   non-empty.
6. **Verify new rows.** If `newKeys` is non-empty, read
   `ATTENDANCE!A<first new row>:A<last new row>` and check that each holds its
   expected key. If any doesn't (another reconcile wrote the same rows at the
   same moment), log `attendance <day>/<facility>: new rows raced, retrying
   next sync` and **do not** record the fingerprint, so the next sync
   reconciles again and writes the missing people below.
7. Otherwise record the fingerprint, and log one line:
   `attendance <day>/<facility>: +<added> ~<updated> -<withdrawn> ^<restored> x<duplicatesCleared>`.
8. Roster `warnings` are logged and returned (§3.4).

**Budget.** The sync's own response waits for `reconcileAfterSync`. Bound the
attendance work with an overall 6-second timeout. If it's exceeded, log a
warning and return, and let the next sync try again. Viewers are never
affected, because publishing already happened (§4.9).

**Concurrency.** Two reconciles of one facility can run at once (a sync and
a manual update, on different instances). They read the same `lastRow`, so
both write their new people to the same row numbers, and the later write
wins. Step 6 notices the loser's missing people, and the next reconcile adds
them below. If both added the same person, the rows are identical and
nothing is lost. A duplicate key that slips through any other way is merged
by step 1 of the next reconcile, and readers always use the first row per key
(§6.1). The narrow case left: a mark aimed at a brand-new row in the
sub-second window before the other reconcile overwrites it. It's accepted;
the person shows as not present and is marked again.

### 4.7 Marking

```js
// actor: { kind: "operator" } | { kind: "desk", day }
async mark({ day, facilityName, key, present, actor }) -> { key, player, present, timeIn, withdrawn }
```

1. Resolve `getDay(day)` (an unknown day gives `UnknownSyncDayError`, 400) and
   `getEvent(event)`.
2. If `attendance` is null, throw `NotFoundError("Attendance is off for this event")`
   (404).
3. If the actor is a desk: when `attendance !== "desks"`, throw
   `ForbiddenError("Desk check-in is off for this event")` (403); when
   `actor.day !== day`, throw `ForbiddenError("This desk link is for another day")` (403).
4. The facility must be in `getDay(day).facilities`, otherwise
   `ValidationError("Unknown facility …. Known facilities: …")` (400).
5. `key` must equal `personKey(key)` (already normalized), otherwise
   `ValidationError("Not a person key")`. `present` must be a boolean.
6. Read `ATTENDANCE!A1:G`. No tab gives
   `NotFoundError("No ATTENDANCE tab yet — run Update roster in Control Center")`.
   `parseTab` may throw `AttendanceLayoutError`.
7. Find the **first** row whose key equals `key`. If there is none, throw
   `NotFoundError("<key> is not on the roster — refresh the page")`.
8. `planMark(row, present, now)`:
   - `present === row.present`: no write, return the row as it is. This makes
     the call idempotent: a double tap or a retry keeps the first `timeIn`.
   - `present === true`: write `E:F = [true, formatTimeIn(now)]`.
   - `present === false`: write `E:F = [false, ""]`.
9. `updateValues(sheetId, [{ range: "ATTENDANCE!E<row>:F<row>", values: [[present, timeIn]] }])`.
10. Return `{ key, player, present, timeIn, withdrawn }` as stored.

`formatTimeIn(date)` uses `Intl.DateTimeFormat("en-CA", { timeZone: "Asia/Manila", year: "numeric", month: "2-digit", day: "2-digit", hour: "2-digit", minute: "2-digit", hourCycle: "h23" })`
and reassembles the result as `yyyy-MM-dd HH:mm`.

There is no lock. Two people marking **different** people write different
rows. Two people marking the **same** person both write `present: true`; the
later `timeIn` wins, a second or two apart. Acceptable.

### 4.8 Routes (`src/attendance/routes.mjs`, mounted at `/v1`)

| Route | Auth | Body | Success |
|---|---|---|---|
| `PUT /v1/days/:day/facilities/:facility/attendance/:key` | operator, or a desk token (§4.7 step 3) | `{ "present": true }` | `200 { key, player, present, timeIn, withdrawn }` |
| `POST /v1/days/:day/attendance/desk-links` | operator only | none | `201 { token, expiresAt, day }` |
| `POST /v1/days/:day/attendance/reconciliations` | operator only | none; optional `?facility=` | `200 { day, facilities: [result] }` (§4.6) |

- Path params arrive URL-encoded (facility names and keys contain spaces);
  Express decodes them.
- **The PUT route validates its body itself:** `req.body?.present` must be
  `true` or `false`, otherwise `ValidationError('present must be true or false')`
  (400), before calling the service. This mirrors `handleSetLive` in
  `src/sync/routes.mjs`. The service checks it again; the route's check is
  what the route tests exercise.
- **Auth.** Read `Authorization: Bearer <token>`.
  `authService.verify(token)` → operator.
  Otherwise `authService.verifyDeskToken(token)` → desk `{ day }`.
  Otherwise `401 { error: "Unauthorized" }`. Operator-only routes answer `401`
  to a desk token. Write these checks as small local functions in this file,
  as `src/sync/routes.mjs` does. Architecture Phase 2 later moves all of them
  into `src/auth/middleware.mjs`.
- **Desk links:** the day's event must have `attendance === "desks"`
  (otherwise `ValidationError("Desk check-in is off for this event")`), the
  day must have a `date` (otherwise `ValidationError("This day has no date in events.json")`),
  and the expiry (§4.3) must be in the future (otherwise
  `ValidationError("This day is over")`).
- **Errors** use today's body, `{ error: message }`, with
  `err.statusCode ?? 500`, exactly as `src/sync/routes.mjs` does.
- Every route has an `@openapi` JSDoc block in the style of
  `src/sync/routes.mjs`, under the tag `attendance`, with security
  `bearerAuth`. The PUT's description states that a desk token is accepted.

New classes in `src/shared/errors.mjs` (`AppError` subclasses):

| Class | Status | Used for |
|---|---|---|
| `ForbiddenError(message)` | 403 | desk token for the wrong day, or desks off |
| `NotFoundError(message)` | 404 | attendance off, no tab, key not on the roster |
| `UnknownEventError(event)` | 404 | `getEvent` on an unknown key; message `Unknown event: <event>` |
| `AttendanceLayoutError()` | 409 | `ATTENDANCE` header row isn't §3.2's. Message: `The ATTENDANCE tab's header row is not key, player, teams, categories, present, timeIn, withdrawn — fix or rename the tab` |
| `UpstreamError(message)` | 502 | Google failures |
| `ServiceBusyError(message)` | 503 | Google `429` after the retry |

### 4.9 The sync hook

`SyncService`'s constructor gains an optional `onFacilitiesSynced` (default
`null`). In `syncDay`, **after** the publish and archive have finished and
before building the return value:

```js
if (this.onFacilitiesSynced && freshByName.size > 0) {
    const facilities = targetFacilities
        .filter(f => freshByName.has(f.name))
        .map(f => ({ name: f.name, sheetId: f.sheetId, standingsCsv: freshByName.get(f.name).standingsCsv }));
    try {
        await this.onFacilitiesSynced({ day, event, facilities });
    } catch (err) {
        log.error("onFacilitiesSynced failed (sync result unaffected)", err);
    }
}
```

The sync's response body and status are unchanged. `setLiveOverride` does not
call the hook.

### 4.10 Wiring, CORS, environment

- `Server.mjs`: `"Access-Control-Allow-Methods": "GET, POST, PUT, OPTIONS"`.
  The constructor accepts `attendanceService` and mounts
  `this.app.use("/v1", attendanceRoutes({ attendanceService, authService, logger: this.logger }))`.
  Architecture Phase 5 later adds its own routes to this same `/v1` prefix.
- `index.mjs`: construct `GoogleAccessToken({ override: process.env.GOOGLE_ACCESS_TOKEN || null, logger })`,
  `SheetsClient({ apiKey: GOOGLE_SHEETS_API_KEY, accessToken, timeoutMs: 5000, logger })`
  and `AttendanceService({ configStore: syncConfigStore, sheets, authService, logger })`.
  Pass `onFacilitiesSynced: args => attendanceService.reconcileAfterSync(args)`
  to `SyncService`, and `attendanceService` to `Server`. Keep the scoresheet
  pipeline lazy, as it is.
- `.env.example`: a `GOOGLE_ACCESS_TOKEN=` entry under a new "Attendance"
  heading. Document it as local development only: paste the output of
  `gcloud auth print-access-token --impersonate-service-account=<SA email>`.
  On Cloud Run it stays unset and the metadata server is used.
- No other new environment variable. The service account is chosen by the
  Cloud Run service's own setting (§8.3), not by code.

---

## 5. Who can do what (summary)

| Action | Operator token | Desk token (its day, event in `"desks"`) | Desk token otherwise | No token |
|---|---|---|---|---|
| Mark a person | yes | yes | 403 | 401 |
| Issue a desk link | yes (event in `"desks"`) | 401 | 401 | 401 |
| Run a roster update | yes | 401 | 401 | 401 |
| Any `/sync/*` route | yes (unchanged) | **401** (new guard, §4.3) | 401 | 401 or secret |
| Read the `ATTENDANCE` CSV | anyone with the sheet ID (§10) | | | |

---

## 6. The site (`sage-match-control.github.io`)

### 6.1 The `ATTENDANCE CLIENT` block

One block of plain JavaScript, delimited exactly like the `LIVE CHANNEL` block:

```js
// ==== ATTENDANCE CLIENT — identical in control-center.html and every attendance.html; see ====
// ==== sage-docs/docs/specs/.../multi-event-attendance-spec.md §6.1                       ====
…
// ==== END ATTENDANCE CLIENT ====
```

It must be byte-identical in `tools/control-center.html`,
`_templates/attendance/attendance.html`, and every
`events/<key>/attendance.html` made from it. It references nothing outside
itself except what its host passes in. It contains:

| Name | Does |
|---|---|
| `ATTENDANCE_POLL_MS = 10000` | refresh interval while the page is visible |
| `attendanceCsvUrl(sheetId)` | `https://docs.google.com/spreadsheets/d/${sheetId}/gviz/tq?tqx=out:csv&headers=1&sheet=ATTENDANCE` |
| `parseAttendanceCsv(text)` | gviz CSV → `people[]`, one per **first** row of each non-blank key: `{ key, player, teams[], categories[], present, timeIn, withdrawn, shirt }`. `teams`/`categories` split on `", "`. `present`/`withdrawn` are `=== "TRUE"`. `shirt` comes from the first header among `TShirt Size`, `T-Shirt Size`, `Shirt Size`, `tshirtSize`, `shirtSize` (as Pickle for Sight's page does), else `""`. Rows after the first for a key are ignored |
| `groupForDesk(people, { type, teamName, categoryLabel, showWithdrawn })` | Standard and dual meet: sections by category (`categoryLabel(code)` for the heading, in first-seen order), each holding cards per team code, each card listing its players. A person in two categories appears in both. Team: sections per team code (`teamName(code) \|\| "Team " + code`), listing players. Withdrawn people are left out unless `showWithdrawn`. Returns `[{ id, label, cards: [{ teamCode, players: [person] }] }]`; for team events each section has a single card. `id` is the section's code, which the category bar (below) filters on |
| `defaultCategoryLabel(code, display)` | a division key from `display.divisions` that prefixes `code`, with the remainder a key of `display.events`: `"<division label> <event label>"`; otherwise `code` |
| `markPerson({ apiBase, token, day, facility, key, present })` | `PUT ${apiBase}/v1/days/${enc(day)}/facilities/${enc(facility)}/attendance/${enc(key)}` with `Authorization: Bearer`, `Content-Type: application/json`, body `{ present }`. Resolves to the response JSON. Rejects with `Error(body.error \|\| "HTTP <status>")` and `err.status` |
| `possibleDuplicates(people)` | §3.5. Returns `[[a, b], …]` |
| `sameNameSameCategory(people)` | people with two team codes sharing one category |
| `createAttendanceView(opts)` | the UI, below |
| `attParseCsv(text)`, `attClockTime(raw)` | private copies, inside the block, of Pickle for Sight's `parseCsv` and `clockTime` (`events/pickle-for-sight-2026/attendance.html`), renamed with the `att` prefix so they can't collide with the console's own `parseCSV`. The block depends on nothing outside itself |

Every top-level name the block declares starts with `att`, `ATTENDANCE_`,
`attendance`, or is one of the names in this table. Before adding it to
`control-center.html`, check that none of them is already declared there
(`grep -n "function <name>\|const <name>\|let <name>"`): a second top-level
declaration of the same name is a `SyntaxError` that takes the whole console
down.

`createAttendanceView({ root, mode /* "console" | "desk" */, apiBase, getToken, event, day, facilities /* [{ name, sheetId }] */, type, teamName /* code -> string|null */, categoryLabel /* code -> string */, notify /* optional (kind, text) */, fixture })`
renders into `root` and returns `{ refresh(), destroy() }`:

- **Header:** a venue picker when there's more than one facility (remembered
  in `localStorage` under `sage.attendance.venue.<day>`), a search box
  (matches name, team code or shirt size, case-insensitive), a
  `<present> / <total> in` count (withdrawn people are not counted), and a
  **Refresh** button.
- **Category bar** (jump and filter), below the header and sticky at the top
  of the view while the list scrolls. It shows one chip per section of
  `groupForDesk`'s output, in the same order (categories for standard and
  dual meet, teams for team events), plus an **All** chip first. Each chip reads
  `<label> <present>/<total>`, counted the same way as the header.
  - **Tapping a chip filters** the list to that section; **All** (the default)
    shows every section. The chosen filter is remembered in `localStorage`
    under `sage.attendance.filter.<day>` and cleared if that section no
    longer exists. It combines with the search box: search narrows within
    the filtered section, and a chip whose section has no search match is
    shown dimmed, not removed.
  - **Jumping:** while **All** is selected, a `Jump to…` `<select>` beside
    the chips lists the same sections. Choosing one scrolls its heading
    into view (`scrollIntoView({ block: "start" })`, offset by the sticky
    bar's height) without filtering. Each section heading carries an `id`
    (`att-sec-<n>`) for this.
  - The chip row scrolls sideways on narrow screens (no wrapping), and
    the selected chip has `aria-pressed="true"`.
  - A person in two categories counts in both chips.
- **List:** `groupForDesk` output. Each player row has the name, a shirt chip
  if any, `In <h:mm AM/PM>` when present (convert `timeIn` the way Pickle for
  Sight's `clockTime` does), and a switch (`role="switch"`). A team card whose
  players are all present gets a **Ready** badge.
- **Toggling** marks optimistically: the switch flips at once and shows
  `Saving…`, then calls `markPerson`. The person's key is added to a `saving`
  set, so a poll that lands meanwhile doesn't flip it back. On success the
  stored values are applied. On failure the switch reverts and the view
  reports `Not saved (<player>): <error>`. On `403` the message adds `Ask
  the operator for a new desk link.`; on `401` it adds `Sign in again.`
  (console) or `This desk link has expired.` (desk). Toggling someone who
  appears in two categories updates both places, because they are one person.
- **Where messages appear:** through `opts.notify(kind, text)` when the host
  passes one, otherwise in the view's own status line, which sits inside the
  sticky category bar so it is on screen wherever the list is scrolled.
  Control Center passes `notify: (kind, text) => showToast(kind, text, 'attendance')`;
  the desk page passes nothing. A network failure (`fetch` rejects) reads
  `Couldn't reach the server. Check this device's connection.`; an API error
  shows the response's `error` text as it is (§4.5 keeps it in words).
- **Polling:** load on start, every `ATTENDANCE_POLL_MS` while
  `!document.hidden`, on `visibilitychange` to visible, and right after the
  page's own successful mark. A load that finishes after a newer load started
  is discarded (a sequence counter, as Pickle for Sight's `load` does).
- **Empty states:** no `ATTENDANCE` tab yet (gviz answers an error page or no
  header row) shows `No roster yet — the operator runs Update roster in
  Control Center.` A fetch failure shows `Could not load <venue>: <message>`
  and keeps the last good list.
- **Fixture mode:** when `fixture` is a name (the host passes it only on
  `localhost`), read `/_fixtures/<event>/attendance-<facility slug>-<fixture>.csv`
  instead of gviz, and make `markPerson` resolve locally, changing the
  in-memory list without any network. This is how the UI is exercised without
  Google or the API (§7.3). The facility slug is the name lowercased with
  spaces replaced by `-`.

### 6.2 Control Center: the **Attendance** tab

- Add `<button class="view-tab" data-view="attendance" type="button" hidden>Attendance</button>`
  directly after **Awards**, so the tabs read Mission Control, Awards,
  Attendance, Live Matches, Match Finder, Standings. Add a results container
  `<div id="attendanceResults" style="display:none;">` beside `#awardsResults`.
- Wire it **in `showView(view)` only**: one line showing or hiding
  `#attendanceResults`, like the other containers, and an `attendance` branch
  that creates the view (below). Leaving the tab, which is any other
  `showView` call, destroys it. The console still opens on Mission Control;
  don't change `revealLiveTabsAfterLoad`.
- Show the tab only when `EVENTS_REGISTRY.get(CURRENT_EVENT_KEY).attendance`
  is `"console"` or `"desks"`, and a day is selected. Re-check on event and
  day change. Hide it, and destroy any view, otherwise.
- On entering the tab, `createAttendanceView({ root: <#attendanceResults list area>, mode: "console", apiBase: CLOUD_RUN_BASE_URL, getToken: currentAuthToken, event: CURRENT_EVENT_KEY, day: <selected day key>, facilities: <the day's facilities with sheetId from the registry entry>, type: CURRENT_TYPE, teamName: code => (CURRENT_TYPE === 'team' ? teamNameOf(code) : null), categoryLabel: code => categoryLabel(code), fixture: FIXTURE })`.
  `teamNameOf` and `categoryLabel` are the
  console's existing functions (find them by name); reuse them, don't copy
  them into the block. Pass `notify` as in §6.1.
- Not signed in: show the list read-only, with `Sign in at the top of
  Mission Control to mark attendance.`, and switches disabled.

### 6.3 Control Center: operator extras (above the list, console mode only)

- **Counts per facility:** `<venue>: <present> / <total>`.
- **Update roster:** `POST /v1/days/{day}/attendance/reconciliations`, then
  refresh. Show the result with `showResultRows` in a box directly under the
  button (`#attendanceRosterResult`), one row per facility: `ok` with
  `3 added · 1 updated · 2 withdrawn` (`No changes` when all are zero,
  `skipped` never applies to a manual update), or `error` with its message.
  The box's kind is `ok`, `warn` (some facilities failed) or `error` (all
  failed). Roster `warnings` (§3.4) add a `warn` row each. A request that
  fails outright (`401`, network) is a toast via `showToast`, with the text
  passed through `friendlyApiMessage`.
- **Desk link** (only when the event's `attendance` is `"desks"`): calls
  `POST …/attendance/desk-links`, then shows the URL
  `https://sage-match-control.github.io/events/<event>/attendance?desk=<token>`
  with **Copy**, **Share** (`navigator.share` when available) and **Show QR**
  (§6.6), plus `Valid until <date> 11:59 PM`, in a box directly under the
  button. It's content to use, not a message, so it stays until the tab is
  left. **Copy** confirms with a toast (`Link copied.`). A failure (desks
  off, no date, day over, `401`) is a toast with the API's message.
- **Needs attention** (collapsed when empty): `possibleDuplicates` pairs,
  `sameNameSameCategory` people, and a **Show withdrawn** toggle that passes
  `showWithdrawn` to the view. Each item says what to do: `Fix the spelling in
  the category tab; the roster follows on the next sync.`

### 6.4 The desk page template: `_templates/attendance/attendance.html`

New file. It isn't type-specific, so it serves all three types. Tokens:
`{{EVENT_KEY}}`, `{{EVENT_TITLE}}`. It uses the house palette and fonts from
`_templates/standard-tournament-template/index.html`'s `THEME` block, and
loads `/assets/favicons/…` like the other pages.

Behaviour:
1. If the URL has `?desk=<token>`: store `{ token }` in `localStorage` under
   `sage.attendance.desk.{{EVENT_KEY}}`, then `history.replaceState` to drop
   the query string, so the token doesn't show in screenshots or reshared
   links.
2. Read the stored token. Decode its payload (the base64url JSON before the
   `.`) for `day` and `exp`. This is for display only; the API decides. No
   token, or `exp` passed, shows `Ask the operator for a desk link.` and
   stops.
3. Fetch `https://sage-match-control.github.io/event-data/config/events.json`
   (on `localhost` with `?fixture=`, `/_fixtures/config.json`), take
   `events["{{EVENT_KEY}}"]` and the token's `day`. If the event's
   `attendance !== "desks"`, show `Check-in for this event is handled by
   staff.` and stop.
4. For team events, fetch the day's published snapshot
   (`…/event-data/{{EVENT_KEY}}/data/<day>.json`) once, and build a map from
   each facility's `standingsCsv` (`teamCode` → `teamName`). On failure use
   an empty map. Pass `teamName: code => map[code] || null`.
5. `createAttendanceView({ root: <the page main container>, mode: "desk", apiBase: <CLOUD_RUN_BASE_URL, the same literal as the console>, getToken: () => token, event: "{{EVENT_KEY}}", day, facilities, type, teamName, categoryLabel: code => defaultCategoryLabel(code, entry.display || {}), fixture })`, where `entry` is `events["{{EVENT_KEY}}"]` from step 3 and `facilities`/`type` come from it (`entry.days[day].facilities`, `entry.type`).

The page is linked from nowhere public (like Pickle for Sight's). Jekyll
publishes it, so it's reachable only by its URL.

### 6.5 Instantiating it for an event

In `_templates/CLAUDE.md` §2, add a step after step 10. For an event with
`"attendance": "desks"`, copy `_templates/attendance/attendance.html` to
`events/<event-key>/attendance.html` and replace its two tokens. For
`"console"` or no attendance, skip it. Also add a step to §0/§2 for **sharing
every facility workbook with the API's service account as Editor**, unless
the workbook lives in the shared Drive folder (check C5, §8.4). Name the
service account's email there once the owner has created it.

### 6.6 QR code

**Show QR** lazy-loads `qrcode-generator` from cdnjs on first click, then
draws the desk URL into a `<canvas>` in the browser. The token never leaves
the page; never use a QR web service. Pin an exact version that exists on
cdnjs (check the URL answers `200`), and add `integrity` with its SHA-384 SRI
hash, computed with:

```bash
curl -s <url> | openssl dgst -sha384 -binary | openssl base64 -A
```

and `crossorigin="anonymous"`. Add this one CDN script to root `CLAUDE.md`'s
list of the site's allowed external dependencies.

---

## 7. Tests

### 7.1 `sage-tools-api`: set up the test runner now

The [API test-suite spec](../not-started/sage-tools-api-test-suite-spec.md) has not been
built yet. Set up exactly the parts of its layout that attendance needs, so
that spec extends this work rather than redoing it:

- `package.json` `scripts`: `"test"`, `"test:unit"` and `"test:integration"`
  **exactly** as in that spec's §3.2. Nothing else from it yet.
- `test/helpers/logger.mjs`, exactly as that spec's §3.4.
- `test/unit/attendance/*.test.mjs`, `test/unit/auth/AuthService.test.mjs`
  (only the desk-token and scope cases below; the suite adds the rest), and
  `test/integration/attendance-routes.test.mjs`.
- Runner `node:test`, assertions `node:assert/strict`, Node 22.23.3 or later.
  `globalThis.fetch` is replaced in tests and restored in `afterEach`. No real
  network, no real waiting (use `mock.timers` for the `429` retry and the
  5-minute `Teams` cache).

### 7.2 Required cases

| Test file | Cases |
|---|---|
| `personKey.test.mjs` | every row of §3.3's table; tabs and newlines collapse; `null`/`undefined` give `""` |
| `roster.test.mjs` | standard: two pairs sharing a player give one person with both teams and categories in order; playoff-seat rows skipped; a name equal to the team code skipped; `bye`/`BYE` skipped; trailing spaces merged; a missing header throws naming it; two pairs in one category with the same name give a warning. Dual meet: `PNF_LIWD_1` → category `LIWD`; a standard-shaped code skipped. Team: header found on row 1 and on row 3; `Player` preferred over `FINAL LEVEL ORDER`; blank names skipped; categories empty; no header row throws. Unknown type throws |
| `attendanceTab.test.mjs` | `parseTab`: the exact header passes; a missing, extra-early or reordered header throws `AttendanceLayoutError`; blank-key rows ignored; booleans read. `planReconcile`: empty tab (every person written to rows 2, 3, … as `A<n>:G<n>`, `false` booleans, joined strings, `newKeys` in order); a tab with a blank row in the middle (new rows go after `lastRow`, never into the hole); nothing changed (empty `data`); a new category for an existing person (C:D update only; B, E, F untouched); a person gone (G true, E:F kept); a person back (G false); duplicates (merged into the first, the later row blanked, `present` OR-ed, earliest `timeIn` kept); every count correct. `planMark`: no write when unchanged; first check-in writes the time; unmark clears it. `formatTimeIn`: a fixed instant renders in Manila time (`2026-10-03T00:30:00Z` → `2026-10-03 08:30`) |
| `AuthService.test.mjs` | `issueDeskToken`/`verifyDeskToken` round trip; expired → null; tampered payload → null; another secret → null; **an operator token is rejected by `verifyDeskToken`, and a desk token is rejected by `verify`**; a hand-made operator-key token whose payload has a `scope` is rejected by `verify`; unset secret → null/false |
| `GoogleAccessToken.test.mjs` | sends `Metadata-Flavor: Google` and the `scopes` query; caches until 60 s before expiry and refetches after (mock timers); `override` never calls fetch; failure throws `UpstreamError`; `email()` returns `null` on failure |
| `SheetsClient.test.mjs` | read URL carries `key` and `valueRenderOption`; a missing tab gives `null`; writes carry the bearer token and `RAW`; **a range outside `ATTENDANCE!A–G` throws before any fetch** (`ATTENDANCE!H2`, `CSV!A1`, `ATTENDANCE!A:Z`); `429` then `200` succeeds after 1 s; `429` twice → `ServiceBusyError`; write `403` → message names the email; timeout → `UpstreamError` |
| `AttendanceService.test.mjs` (fake `SheetsClient` and config) | `reconcileAfterSync` on an event without attendance makes no Sheets call; the unchanged-roster shortcut skips on the second call and not after a name changes; a failure in one facility doesn't stop the others and never throws; the tab is created when missing; a raced new row (the verify read returns another key) leaves the fingerprint unrecorded, so the next call reconciles again; team events read `Teams`, cached for 5 minutes (mock timers) in `reconcileAfterSync` but always re-read by `reconcileDay`; the 6-second budget returns without throwing. `mark`: §4.7's every step, each error with its class; desk actor with `"console"` → 403; desk actor for another day → 403. `issueDeskLink`: no date → 400; past day → 400; `"console"` → 400; expiry is the day's 23:59:59.999 +08:00 |
| `SyncService` hook (`test/unit/sync/onFacilitiesSynced.test.mjs`, using the fakes in `scripts/verify-sync-merge.mjs` copied into the test file) | called after publish with the fresh facilities' `standingsCsv`; not called when every fetch failed; a throwing hook leaves the sync's response unchanged |
| `SyncConfigStore` validation (`test/unit/sync/attendanceConfig.test.mjs`) | `"console"`, `"desks"` and absent pass; `"yes"`, `true` fail with the message; `getEvent` returns `attendance`; `getDay` returns `date` |
| `attendance-routes.test.mjs` (the real `Server` on `http.createServer(server.app)` port 0, as the test-suite spec's `startApp` does; a fake `attendanceService`) | `PUT` with an operator token → 200 and the service gets `actor.kind === "operator"`; with a desk token → `actor.kind === "desk"` and its day; no token → 401; bad body (`present: "yes"`) → 400; service errors map to their status with body `{ error }`; desk token on `POST …/desk-links` and `…/reconciliations` → 401; `POST …/desk-links` → 201; **a desk token on `POST /sync/<day>` → 401**; `OPTIONS` preflight answers `204` with `Access-Control-Allow-Methods: GET, POST, PUT, OPTIONS`; `GET /openapi.json` lists the three new paths |

`npm test` must pass, and the five existing `node scripts/verify-*.mjs`
scripts must still pass, untouched.

### 7.3 The site

The site test suite ([site test suite spec](../not-started/site-test-suite-spec.md)) isn't
built yet. Until it is:

- Add fixtures: `_fixtures/config.json` gains `"attendance": "desks"` on the
  PickleDrive entry (fixtures only, never the real config), and
  `_fixtures/pickledrive-anniversary-2026/attendance-kingcourts-pre.csv` holds
  six people. One is in two teams, one is withdrawn, one is present with a
  `timeIn`, and one row duplicates another's key. Add a standard-event fixture
  the same way if you add a standard test event to `_fixtures/config.json`.
- Verify by hand on `localhost` with `?fixture=pre`, in the browser preview
  if you have one:
  - the console's tab appears only for an event with `attendance`
  - toggling a two-team person flips both places
  - a category chip filters the list, and the filter is restored after a
    reload; **All** with `Jump to…` scrolls to a section without filtering;
    the chip counts match the header's
  - search inside a filtered category narrows within it
  - the tabs read Mission Control, Awards, Attendance, …; picking a day still
    lands on Mission Control; switching away from Attendance and back
    recreates the view without errors
  - in the console, a failed mark (fixture mode: make `markPerson` reject
    once) shows a toast; on the desk page it shows in the status line
  - **Update roster**'s result appears as rows under the button, and no
    message anywhere contains `{` or `HTTP 4`/`HTTP 5` followed by JSON
  - withdrawn people are hidden until **Show withdrawn**
  - the duplicate row is ignored
  - the desk page shows the desk-off message when the fixture config says
    `"console"`
  - no request leaves `localhost` in fixture mode
- Check the block is byte-identical. This must print nothing:

  ```bash
  diff <(sed -n '/==== ATTENDANCE CLIENT/,/==== END ATTENDANCE CLIENT/p' tools/control-center.html) <(sed -n '/==== ATTENDANCE CLIENT/,/==== END ATTENDANCE CLIENT/p' _templates/attendance/attendance.html)
  ```

---

## 8. Checks on Google's side (the owner, with a one-off service)

### 8.1 What the check service is

`sage-tools-api/spikes/sheets-write-check/`: `package.json` (no dependencies,
`"type": "module"`, `"engines": { "node": "22.x" }`, `"scripts": { "start":
"node server.mjs" }`), `server.mjs` (plain `node:http`), and a `README.md` with
§8.2's steps. One endpoint, `GET /check?sheet=<spreadsheetId>`, that runs, in
order, and returns a JSON report with each step's result and milliseconds:

1. `token`: get a token from the metadata server with
   `?scopes=https://www.googleapis.com/auth/spreadsheets` (§4.4); report
   success and `expires_in`, **never the token**.
2. `email`: the service account's email.
3. `addTab`: add a tab named `SAGE_WRITE_CHECK` (if it already exists, report
   that and continue).
4. `formula`: write `B1` = `=A1*2` with `valueInputOption=USER_ENTERED`.
5. `write`: write `A1` = `21` with `RAW`, timing it.
6. `recalc`: read `B1` (`UNFORMATTED_VALUE`) **immediately**. Report the value;
   `42` means formulas recalculate before the next read.
7. `writeText`: write `A2` = `"08:42"` and `A3` = `"123"` with `RAW`, read them
   back with `UNFORMATTED_VALUE`, and report their types (they must stay
   strings).
8. `cleanup`: delete the `SAGE_WRITE_CHECK` tab.

It is deployed **private** (`--no-allow-unauthenticated`), so only the owner
can call it. It writes only to the `SAGE_WRITE_CHECK` tab it creates, and
only in the spreadsheet named in the request.

### 8.2 The owner's steps

Do these after the 3 October events, on a **copy** of a facility workbook.
Never use the live one.

1. **Find the project ID** (the number `811926984834` is the project
   number):

   ```bash
   gcloud projects describe 811926984834 --format="value(projectId)"
   ```

2. **Create the service account** (no project roles):

   ```bash
   gcloud iam service-accounts create sage-tools-api-runtime --project=<PROJECT_ID> --display-name="sage-tools-api runtime"
   ```

3. **Let deploys use it.** Find the Cloud Build trigger's service account
   (Cloud Build → Triggers → the `sage-tools-api` trigger → Service account),
   and grant it, and yourself, **Service Account User** on the new account:

   ```bash
   gcloud iam service-accounts add-iam-policy-binding sage-tools-api-runtime@<PROJECT_ID>.iam.gserviceaccount.com --project=<PROJECT_ID> --member="serviceAccount:<TRIGGER_SA_EMAIL>" --role="roles/iam.serviceAccountUser"
   ```

4. **Check that the Sheets API is enabled** (it almost certainly is, since the
   sync's API key uses it):

   ```bash
   gcloud services list --enabled --project=<PROJECT_ID> --filter="name:sheets.googleapis.com"
   ```

5. **Make the test copy.** In Drive, make a copy of any facility workbook
   (File → Make a copy). Share it with
   `sage-tools-api-runtime@<PROJECT_ID>.iam.gserviceaccount.com` as **Editor**.
   In the copy, add a trigger to observe `onEdit` (check C3): Extensions →
   Apps Script → a new file with

   ```js
   function logEdit(e) { console.log("onEdit fired: " + e.range.getA1Notation()); }
   ```

   then Triggers → Add trigger → `logEdit`, From spreadsheet, On edit. **Do
   not** run **SAGE → Set up live sync** in the copy: it would publish the
   copy's data to the real event.

6. **Deploy and run the check service**, from `sage-tools-api/`:

   ```bash
   gcloud run deploy sheets-write-check --source spikes/sheets-write-check --region us-central1 --project=<PROJECT_ID> --service-account=sage-tools-api-runtime@<PROJECT_ID>.iam.gserviceaccount.com --no-allow-unauthenticated
   ```

   ```bash
   curl -s -H "Authorization: Bearer $(gcloud auth print-identity-token)" "<SERVICE_URL>/check?sheet=<COPY_SPREADSHEET_ID>"
   ```

   Then in the copy's Apps Script → Executions: `logEdit` must **not** appear
   for the check's writes. Type into any cell by hand and confirm it does
   appear, which proves the trigger works.

7. **Delete the check service:**

   ```bash
   gcloud run services delete sheets-write-check --region us-central1 --project=<PROJECT_ID>
   ```

8. **Drive folder sharing (C5).** Create a Drive folder (e.g. `SAGE
   workbooks`), share it with the service account as Editor. Then:
   - make a copy of a workbook **into** that folder
   - move an existing workbook into it

   For each, open Share and see whether the service account is listed.

9. **Quotas (C6).** Cloud Console → APIs & Services → Google Sheets API →
   Quotas. Note the per-minute read and write limits per user and per
   project.

10. **The Teams tab (C7).** In PickleDrive's workbook, confirm the values in
    `Teams`' **Team Code** column are the same codes as `STANDINGSCSV`'s
    `teamCode` (`A`, `B`, …), and that the header row is row 1, or note which
    row.

11. Fill in §8.4 and commit it to `sage-docs`.

### 8.3 Switching the API to the service account (after the checks pass)

```bash
gcloud run services update sage-tools-api --region us-central1 --project=<PROJECT_ID> --service-account=sage-tools-api-runtime@<PROJECT_ID>.iam.gserviceaccount.com
```

Later deploys keep this setting. The implementer adds
`--service-account=sage-tools-api-runtime@<PROJECT_ID>.iam.gserviceaccount.com`
to the deploy command in the `Dockerfile`'s comment, and the owner fills in
the project ID. Afterwards, confirm `GET /ping` still answers and a sync from
Control Center's **Resync this day now** still succeeds. Logs keep working:
writing to stdout needs no role.

### 8.4 Results (the owner fills this in)

| # | Check | Expected | Result | Date |
|---|---|---|---|---|
| C1 | `token` with the Sheets scope works on Cloud Run | success | | |
| C2 | `write` and `addTab` succeed on a workbook shared with the account | success; note the write ms | | |
| C3 | the check's writes do **not** fire `onEdit`; a hand edit does | none / one | | |
| C4 | `recalc` reads `42` immediately | `42` (needed later for Court Control, not for attendance) | | |
| C4b | `writeText` values stay strings | strings | | |
| C5 | a copy made into, or moved into, the shared folder lists the service account | yes / no | | |
| C6 | Sheets API quotas: read/min/user, write/min/user, per project | ≥ 60 writes/min/user | | |
| C7 | Teams `Team Code` = standings `teamCode`; header row number | same; row 1 | | |

**What changes the plan:**
- C1 fails → stop; the token comes another way (owner decision).
- C2 fails → the sharing or API setup is wrong; fix it before Phase 3.
- C3 fires → the roster update must not write when nothing changed (it
  already doesn't). Note it, because a write would trigger a sync that
  triggers a roster update.
- C5 "no" → §6.5's sharing step is per workbook.
- C6 below 60 writes/min/user → tell the owner the busiest-desk limit before
  Phase 4.
- C7 differs → §3.4's team rule changes.

---

## 9. Build phases

Each phase ends with its acceptance met and one commit on the `attendance`
branch of each repo it touches. Stop after any phase if something in this
spec turns out wrong, and report it.

| Phase | Who | What | Acceptance |
|---|---|---|---|
| 1 | implementer | the check service (§8.1) | `node --check spikes/sheets-write-check/server.mjs` passes; the README carries §8.2 |
| 2 | **owner** | §8.2 and §8.4 | §8.4 filled in, with no "stop" outcome |
| 3 | implementer | config (§4.2), errors (§4.8 table), desk tokens (§4.3), test runner (§7.1) and their tests | `npm test` green; `verify-*` scripts green |
| 4 | implementer | `personKey`, `roster`, `attendanceTab`, `GoogleAccessToken`, `SheetsClient`, `AttendanceService`, the sync hook, routes, CORS, wiring, OpenAPI, and all §7.2 tests | `npm test` green; `node index.mjs` starts locally and `GET /openapi.json` shows the three paths; version `2.6.0` with its Changelog entry |
| 5 | implementer | the `ATTENDANCE CLIENT` block and Control Center tab and extras (§6.1–6.3, 6.6), fixtures (§7.3) | §7.3's manual checks pass in fixture mode |
| 6 | implementer | the desk page template (§6.4), `_templates/CLAUDE.md` steps (§6.5) | the `diff` in §7.3 prints nothing; the desk page passes §7.3's checks in fixture mode |
| 7 | implementer | docs (§9.1) | every page in §9.1 updated |
| 8 | **owner** | §8.3 (switch the service account), merge and deploy, then the first real run (§9.2) | §9.2 passes |

### 9.1 Documentation (Phase 7)

| File | Change |
|---|---|
| `sage-docs/docs/features/event-attendance.md` | rewrite in the present tense: the two modes, one check-in per person, desk links, Update roster, what withdrawn means, Pickle for Sight as the earlier version |
| `sage-docs/docs/technical/event-attendance.md` | rewrite: §2's flow, the tab layout, the roster rules, the routes, the service account, the allowlist, the sync hook |
| `sage-docs/docs/technical/auth.md` | desk tokens and the operator scope guard |
| `sage-docs/docs/technical/deployment.md` | the runtime service account and its deploy flag |
| `sage-docs/docs/technical/event-data-config.md` and `event-data/config/README.md` | the `attendance` field |
| `sage-docs/docs/features/preparing-an-event.md`, `running-an-event-day.md` | setting the mode, sharing workbooks, issuing a desk link, Update roster |
| root `D:\Personal\SAGE\CLAUDE.md` | `sage-tools-api`: `src/attendance/` in the layout, the `/v1` routes in the endpoints list, `test/` and `npm test`; the site: the `ATTENDANCE CLIENT` block in "kept in sync by hand", `_templates/attendance/`, the QR CDN script, and the attendance pages' copy of `CLOUD_RUN_BASE_URL` in the hard-coded constants list |
| this spec | status line → "in progress", and move it per `docs/specs/README.md` |

### 9.2 First real run (the owner, Phase 8)

1. Pick an event at least a week away. Set `"attendance": "console"` in
   `event-data/config/events.json` and commit.
2. Share its workbooks with the service account (or put them in the shared
   folder, per C5).
3. Control Center → the event → **Attendance** → **Update roster**. Every
   facility reports people added, and the workbook's new `ATTENDANCE` tab
   lists them, one row per person, with multi-category players listing every
   team.
4. Mark and unmark someone; the workbook updates and the tab shows it within
   10 seconds.
5. Switch the event to `"desks"`, issue a desk link, open it on a phone, mark
   someone. Switch back to `"console"`; within about a minute the phone's next
   mark answers `Ask the operator for a new desk link`.
6. Change a player's name in a category tab and let it sync. The new name
   appears, and the old one is withdrawn if they're in no other category.

---

## 10. Risks and limits (known, accepted)

- **The `ATTENDANCE` tab is unlisted, not private.** The CSV export is
  readable by anyone with the sheet ID, and sheet IDs are in the public
  `events.json`. Names, check-in times and T-shirt sizes are fine. Never add
  anything sensitive (phone numbers, emails) to that tab. If that's ever
  needed, reads must move behind the API.
- **Sorting the tab by hand** between a mark's read and its write could, in
  a sub-second window, write the wrong row, as with the old script.
  Operators use a filter view, not Sort. The runbook says so.
- **Name matching** merges people with identical names (rare). They appear
  as one row with both teams, and `sameNameSameCategory` flags the case where
  they share a category.
- **Quota:** a mark costs one read (API key, per-project quota) and at most
  one write (service account). The busiest-desk limit is C6's write figure.
- **Cold starts:** `--min-instances 0`, so the first mark after idle can take
  a few seconds. The switch shows `Saving…` meanwhile.

## 11. Relationship to other specs

- **[`sage-tools-api` test suite](../not-started/sage-tools-api-test-suite-spec.md):** this
  spec creates `test/`, the `test` scripts and `test/helpers/logger.mjs` in
  that spec's layout. That spec then pins attendance's routes too (its §2.1
  gains them). Its characterization B3 changes from `GET, POST, OPTIONS` to
  `GET, POST, PUT, OPTIONS`. The spec is updated with this one.
- **[Architecture hardening](../not-started/sage-tools-api-architecture-spec.md):** Phase 2
  moves attendance's auth checks into `src/auth/middleware.mjs` and adds
  `code` to its error bodies. Phase 5 adds the other `/v1` routes beside
  attendance's, on the same prefix. Phase 4's `SyncService` split keeps the
  `onFacilitiesSynced` hook.
- **[Site test suite](../not-started/site-test-suite-spec.md):** the `ATTENDANCE CLIENT`
  block joins its consistency layer, and the console tab and desk page join
  its page tests. The spec is updated with this one.
- **Next, on the same foundation (not in this spec):** Court Control and
  score entry from Control Center. `SheetsClient`'s allowlist gains the cells
  those features need, and check C4 is what makes write-then-sync work. It
  gets its own spec after attendance runs at a real event.

## 12. Out of scope

- Pickle for Sight's page and `attendance.gs` (unchanged; archived with the
  event).
- Walk-ins added from the page, check-out times, per-category check-in
  (decided against, D2), public attendance views.
- Pushing attendance through the live Worker (D4).
- Automatic merging of near-duplicate names (D9).

## 13. Divergences

Built on the `attendance` branches. Where it departs from the text above:

- **`createAttendanceView` returns `setShowWithdrawn(v)` and takes `onData`.**
  Control Center needs both for §6.3: the counts per facility, **Needs
  attention** and **Show withdrawn** are computed from the people the view has
  loaded. In console mode the view loads every facility of the day (so counts
  are complete); on the desk page it loads only the venue shown.
- **Messages through `notify`** skip "Loading…" and successes: only errors and
  warnings are toasted, because a loading toast would never be replaced.
- **Fixture mode** also covers Update roster and Issue desk link in Control
  Center (no API call; a canned result and a fake token), and `?attfail` makes
  the first mark fail, to exercise the failure paths. The QR code still loads
  from cdnjs when clicked.
- **`possibleDuplicates`** ignores withdrawn people and returns pairs of person
  objects, not keys.
- **A tab that exists but is empty** (no header row) is an
  `AttendanceLayoutError`, as the header rule reads. Only a missing tab is
  created.
- **The package lock** still says 2.0.0 and was left alone.
- **QR library:** cdnjs has `qrcode-generator` files only up to **1.4.4**, so that
  is the pinned version (SRI `sha384-mZT2gIty7ZDdOGkxfP6joZcYdMW1Jvj9dRlfpTmaJAKKXTqzygtB22k7FLe+KZC1`).
- Fixtures add a standard `console` demo event (`attendance-demo-2026`, in
  `_fixtures/config.json` only) beside the PickleDrive `desks` one.
- Not yet checked by hand: Control Center's read-only state when signed out, and
  the tab hiding for an event with no `attendance` setting.
