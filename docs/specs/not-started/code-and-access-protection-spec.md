# Spec — code and access protection

> **Status: not started.** Sections 2–8 are a to-do list, not a design: each
> item says what to change and where, and leaves the details to whoever builds
> it, ordered by value. §9 (facility workbooks closed to link access) and §10
> (two fixes to the played and series rules) are full designs, written to be
> built without other context. Written 2026-10-11; §9 and §10 added
> 2026-10-12. Everything here is free or close to it. Each section is
> independent and can be done on its own.
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
- The `sage-tools-api-runtime` service account is an Editor on facility
  workbooks (every event's, §9.3 D3), never on the masters.
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

Unlike §2–§8, this section is a full design with build steps, written so it can
be built by someone with no other context. §9.1–§9.3 say what and why; §9.4 is
what the owner does by hand; §9.5–§9.9 are the build, in the order it ships.

### 9.1 Goal

A facility workbook's General access is **Restricted**. It is shared by name
with the people who edit it and with the API's service account,
`sage-tools-api-runtime@sage-tools-api.iam.gserviceaccount.com`. Nobody else can
open it, so its `ATTENDANCE` tab (names, teams, shirt sizes, who is present)
and every tab the sync does not publish stay inside the workbook. What the sync
publishes to `event-data` stays public, as now.

This applies to events built after §9.5–§9.8 ship. It changes nothing for an
event already registered (§9.3, D4).

### 9.2 What reads a workbook today

Four readers need link access. An API key can read only a sheet that anyone
with the link can open, and so can a request with no credential, so all four
change:

| # | Reader | File | How it reads now |
| --- | --- | --- | --- |
| R1 | The sync's default fetcher | `sage-tools-api/src/clients/SheetsCsvFetcher.mjs` | Sheets API `values:batchGet`, `?key=` `GOOGLE_SHEETS_API_KEY` |
| R2 | The sync's fallback, `{ method: "csv" }` | `sage-tools-api/src/clients/GvizCsvFetcher.mjs` | the gviz CSV export, no credential |
| R3 | Attendance and score-entry reads | `sage-tools-api/src/clients/SheetsClient.mjs`: `readValues`, `readScoreNotes` | Sheets API, `?key=`. Its writes already send the service account's bearer token |
| R4 | The desk page and Control Center's Attendance tab | `sage-match-control.github.io/lib/v1/views/attendance-view.js`: `attendanceCsvUrl`, `fetchFacility` | the gviz CSV export of `ATTENDANCE`, from the browser, every `ATTENDANCE_POLL_MS` (10 s, `lib/v1/platform.js`) |

Not affected: the Apps Script (`sheets-sync.gs`, the two generators), which
runs inside the workbook; every page's reading of `event-data`; the masters.

The service account's token comes from `src/clients/GoogleAccessToken.mjs`:
`token()` fetches it from Cloud Run's metadata server with the `spreadsheets`
scope (reads included) and caches it; `GOOGLE_ACCESS_TOKEN` overrides it for
local runs; `email()` gives the account's address for messages. It is
constructed once in `src/app.mjs` (`googleAccessToken`).

### 9.3 Decisions

| | Decision | Consequence |
| --- | --- | --- |
| D1 | **`GOOGLE_SHEETS_API_KEY` stays, as a fallback.** | Reads use the token. The key is used when no token can be had (a local run with no metadata server and no `GOOGLE_ACCESS_TOKEN`), and once more after the token is refused with `403`, which lets a still link-shared workbook (a finished event) be read. |
| D2 | **The gviz fallback (R2) is kept.** If spike S1 (§9.4) shows gviz accepts a bearer token, R2 sends the service account's token; if not, R2 stays as it is and works only on a link-shared workbook. | Either way the code stays and `{ method: "csv" }` keeps its meaning. The docs say which case holds. |
| D3 | **The service account is Editor on every facility workbook**, whatever the event's `attendance` and `scoreEntry` settings. | One runbook step for every event, and turning attendance or score entry on later needs no re-share. §6 says the same. |
| D4 | **Workbooks of events already registered keep their sharing.** | BKL Cup, PNF x BUP, Pickle for Sight, Piggleball and PickleDrive are finished. Pages that read their workbooks straight from Google (the archived pages under `events/archives/`, and the frozen desk pages of Pickle for Sight, Piggleball and PickleDrive) keep working. |
| D5 | **The desk list is read through a new API route**, `GET /v3/days/{day}/facilities/{facility}/people` (§9.7). | Taken because R4 is the only reader in a browser, and a browser has no credential Google accepts. |
| D6 | **The list read is cached on the server for 10 s per workbook**, shared by every caller, and dropped on every write to that workbook's `ATTENDANCE` tab. | The number of desks open does not multiply reads from Google (§9.4, O2). A marker sees their own change at once; another desk within one poll and one cache period (≤ 20 s). |

### 9.4 Owner tasks (not code)

These need the Google account and the Cloud console, so the owner does them.

- **S1, the gviz spike: does the gviz export accept a bearer token?** Make a
  test spreadsheet, set it to Restricted, and type anything in `A1` of a tab
  named `T`. In it, open **Extensions → Apps Script**, paste and run:

  ```js
  function spikeGvizBearer() {
    const id = SpreadsheetApp.getActive().getId();
    const url = `https://docs.google.com/spreadsheets/d/${id}/gviz/tq?tqx=out:csv&sheet=T`;
    const res = UrlFetchApp.fetch(url, {
      headers: { Authorization: 'Bearer ' + ScriptApp.getOAuthToken() },
      muteHttpExceptions: true,
    });
    Logger.log(res.getResponseCode() + ' ' + res.getContentText().slice(0, 200));
  }
  ```

  CSV in the log (the `A1` text, quoted) means **yes**; an HTML page or a
  `401`/`403` means **no**. Record the answer here, under D2, before §9.6
  starts. The test spreadsheet can be deleted afterwards.
- **O1, the Sheets API quota.** In the Google Cloud console (project
  `sage-tools-api`), open **APIs & Services → Google Sheets API → Quotas** and
  note *Read requests per minute per user*. Every read moves to one identity,
  the service account, so this is the limit that counts. At the time of writing
  Google's documented default is 60 a minute per user (300 per project).
- **O2, compare it with an event's peak.** The reads a minute at one event's
  busiest moment, with §9.7's cache:
  - **syncs:** one `batchGet` each. On 3 October 2026 two events made 936 syncs
    in 3 h 15 min, about 5 a minute.
  - **desk lists:** at most 6 a minute per facility (one per 10 s cache period),
    however many desks are open.
  - **marks:** one read each (`ATTENDANCE!A1:Z`), at the rate people arrive.
  - **score entry:** two reads per save (`SCHEDULE` and the score notes).
  - **reconcile after a sync:** one or two reads per facility, when the roster
    changed.

  For three facilities that is roughly 5 + 18 + marks + scores. If the quota is
  60 or close to the estimate, request a raise (to 300) on the same page; it is
  free. Do it before the first Restricted event.
- **O3, after §9.8 ships: the first event built after it** has every facility
  workbook Restricted and shared with the service account as Editor, from the
  start. Never change a live event's sharing during its event week.

### 9.5 Before you build

Read this whole section, then the workspace `CLAUDE.md` (the root of
`D:\Personal\SAGE`), which holds every repo's rules. The rules that matter
here:

- **`sage-tools-api`:** no production change without a failing test first.
  Every change to `src/` changes `test/` in the same commit; run `npm test`
  before every commit and `npm run verify` before every push to `main`. A push
  to `main` deploys to Cloud Run, so never push in an event week. Bump
  `package.json` (patch for a fix, minor for a feature) and add a `README.md`
  Changelog entry ending with a **Tests:** line. Only `src/app.mjs` constructs
  objects. `src/clients/` may import `shared/*`, domain files and built-ins,
  never `registry/`, services, routes or `express`. Never weaken or delete a
  test to make a change pass; when a test pins behaviour this section changes on
  purpose, change it and say so in the commit message.
- **The site:** only `lib/v1/` is edited, and only compatibly: add, never
  remove or rename an export, and never change an export's parameters or
  return shape. `views/` may import only `domain/` and `views/` (so not
  `data/api.js`). Run `cd _tests && npm install && npm run verify` before every
  push to `main`; never merge within 3 days of an event.
- **sage-docs:** present tense; every change carries its version (see
  `docs/changelog.md`, "Releasing a version").

Ship §9.6, §9.7 and §9.8 as three separate releases, in that order. §9.6 and
§9.7 each work on today's link-shared workbooks, so neither has to wait for an
event to pass. §9.8 needs §9.7 deployed first.

### 9.6 Build 1: the server reads as the service account (R1, R2, R3)

`sage-tools-api`, a **minor** release.

**A read credential, in one place.** Add `src/clients/sheetsReadAuth.mjs`,
exporting `class SheetsReadAuth`:

```js
// constructor({ accessToken, apiKey = null, logger, now = () => Date.now(), tokenRetryMs = 60_000 })
//   accessToken: a GoogleAccessToken; apiKey: GOOGLE_SHEETS_API_KEY or null
//
// async primary() -> { kind: "token", headers: { Authorization: "Bearer …" } }
//                  | { kind: "key",   key: "<api key>" }
//   The token, unless token() failed less than tokenRetryMs ago (remembered,
//   so a local run without a metadata server does not wait 3 s on every read).
//   Falls back to the key when no token can be had. Throws UpstreamError
//   "Google Sheets: no credential (no service-account token and no
//   GOOGLE_SHEETS_API_KEY)" when neither exists.
//
// fallbackAfter(credential) -> the key credential, when `credential` was the
//   token and an API key is set; otherwise null.
//
// async refusedMessage() -> "The API can't read this workbook. Share it with
//   <email> as Editor." (email from accessToken.email(), or "the API's
//   service account" when it is unknown)
```

Every reader then follows the same steps:

1. `const cred = await auth.primary()`. A token goes in the `Authorization`
   header; a key goes in the `key` query parameter, as today.
2. If the answer is `403` and `auth.fallbackAfter(cred)` is not null, send the
   same request once more with that key credential.
3. If the answer is still `403`, throw `UpstreamError(await auth.refusedMessage())`.
   Any other status is handled exactly as today.

Changes:

- **`SheetsCsvFetcher`** (R1): the constructor becomes
  `(logger, auth, timeoutMs)`, where `auth` is a `SheetsReadAuth`.
  `#fetchBothOnce` uses it as above in place of `url.searchParams.set("key", …)`.
  The "throws before any request when there is no API key" check becomes
  `auth.primary()`'s throw. The roster-tab retry
  (`400 Unable to parse range`) is unchanged.
- **`SheetsClient`** (R3): the constructor takes `auth` (a `SheetsReadAuth`)
  in place of `apiKey`; keep `accessToken` for writes. `readValues` and
  `readScoreNotes` build their URL without `key=` and send through a private
  `#read(url)` that applies the steps above. `readValues`'s
  `400 Unable to parse range → null` stays. Writes are unchanged, including
  their own `403` message ("can't edit … as Editor").
- **`GvizCsvFetcher`** (R2): only if S1 answered **yes**. The constructor takes
  `auth`, and `#fetchTextOnce` sends the token's `Authorization` header when
  `primary()` gives a token, and nothing when it gives a key (gviz takes no
  key). If S1 answered **no**, leave this file unchanged.
- **`src/app.mjs`**: construct one `SheetsReadAuth({ accessToken: googleAccessToken, apiKey: config.sheets.apiKey, logger })`
  next to `googleAccessToken` and pass it to every `SheetsClient` and to
  `SheetsCsvFetcher` (and to `GvizCsvFetcher` under S1 = yes). Move the comment
  above `googleAccessToken` ("Attendance writes …") to say the account now
  reads for the sync, attendance and score entry, and writes for attendance and
  score entry.
- **Comments to correct:** the header of `SheetsClient.mjs` ("Reads use the API
  key (the workbooks are link-viewable …)"), the `readScoreNotes` comment ("A
  read, so the API key"), and `GoogleAccessToken.mjs`'s header, which names
  attendance only.
- `loadConfig.mjs` and `.env.example` need no change: `GOOGLE_SHEETS_API_KEY`
  stays optional (D1). Update its line in `.env.example` to say it is the
  fallback for reads.

Tests, written first:

- `test/unit/clients/sheetsReadAuth.test.mjs` (new, the mirrored file):
  `primary()` gives the token when `token()` resolves; gives the key when
  `token()` rejects and a key is set; throws `UpstreamError` when it rejects
  and no key is set; after a rejection, does not call `token()` again within
  `tokenRetryMs` (use the injected `now`) and does after it;
  `fallbackAfter` gives the key after a token and null after a key or with no
  key; `refusedMessage()` names the email, or "the API's service account" when
  `email()` gives null.
- `test/unit/clients/SheetsCsvFetcher.test.mjs`: sends `Authorization: Bearer …`
  and no `key` parameter; uses the key when no token can be had; retries once
  with the key after a `403` to the token; throws the refused message when the
  key also gets `403`; throws before any request with neither credential. Its
  existing test "asks for the matches tab then the standings tab, with the key
  and FORMATTED_VALUE" changes on purpose: say so in the commit.
- `test/unit/clients/SheetsClient.test.mjs`: the same five cases for
  `readValues` and for `readScoreNotes`. Its existing test "sends the API key
  and the render option, and no bearer token" changes on purpose.
- `test/unit/clients/GvizCsvFetcher.test.mjs`, under S1 = yes only: sends the
  bearer token; sends no `Authorization` when only a key exists.
- `test/unit/app.test.mjs`, if it asserts how the clients are constructed.

Docs in the same release: [Sync pipeline](../../technical/sync-pipeline.md)
(how the fetchers authenticate; what `{ method: "csv" }` needs, per S1),
[Event attendance](../../technical/event-attendance.md) and
[Deployment](../../technical/deployment.md) (`GOOGLE_SHEETS_API_KEY` is the
read fallback). In the workspace `CLAUDE.md`: the `src/clients/` and
`src/attendance/` bullets that say `SheetsClient` "reads with the API key".

**Done when:** `npm test` and `npm run verify` pass; with the token and no key
every read sends only `Authorization`; with neither, the error names both.

### 9.7 Build 2: the attendance list from the API (R4, server half)

`sage-tools-api`, a **minor** release.

**The route.** In `src/attendance/routes.mjs`, `attendanceV3Routes`:

```js
router.route("/days/:day/facilities/:facility/people")
    .get(auth.requireOperatorOr("desk"), asyncHandler(handlers.listPeople))
    .all(methodNotAllowed);
```

`handlers.listPeople` calls `attendanceService.listPeople({ day, facilityName: req.params.facility, actor: req.actor })`,
sets `Cache-Control: no-store`, and answers `200` with the array itself (no
envelope, per the [API](../../technical/api.md) conventions). There is no `/v1`
twin: new routes are `/v3` only.

**The body:** a JSON array of people, in sheet order, one per key:

```json
[
  { "key": "ana-santos", "player": "Ana Santos", "teams": ["PNF_LIWD_1"],
    "categories": ["LIWD"], "present": true, "timeIn": "2026-10-03 08:14",
    "withdrawn": false, "shirt": "M" }
]
```

This is exactly the `Person` shape the site's `parseAttendanceCsv`
(`lib/v1/domain/attendance.js`) builds today, so the page renders it
unchanged.

**The rules,** a new pure function `peopleFromTab(values)` in
`src/attendance/domain/attendanceTab.mjs`. `values` is what
`readValues(sheetId, "ATTENDANCE!A1:Z", { render: "FORMATTED_VALUE" })`
returned:

- Columns A–G go through the existing `parseTab` (so a wrong header row throws
  `AttendanceLayoutError`, a `409 attendance_layout`, as a mark does).
- One person per key: the first row with that key wins; rows with a blank key
  are skipped.
- `player` is the trimmed player text, or the key when it is blank.
- `teams` and `categories` split on commas, each part trimmed, empty parts
  dropped.
- `present` and `withdrawn` are true for `true` or the text `TRUE` (any case):
  `parseTab`'s `flag` already does this.
- `timeIn` is the trimmed text.
- `shirt` is the trimmed text of the first column in row 1 (A–Z) whose
  trimmed header is one of `SHIRT_HEADERS`, or `""` when there is none. Add
  `export const SHIRT_HEADERS = ["TShirt Size", "T-Shirt Size", "Shirt Size", "tshirtSize", "shirtSize"]`
  to `attendanceTab.mjs`: the same list as the site's
  `ATTENDANCE_SHIRT_HEADERS`, which stays where it is.

`FORMATTED_VALUE` (not the mark's `UNFORMATTED_VALUE`) because the page shows
what the sheet displays: a hand-typed time stays text instead of arriving as a
date serial number, as it did through gviz.

**The service method,** `AttendanceService.listPeople({ day, facilityName, actor })`,
checks exactly what `mark` checks, in the same order and with the same errors:

1. `config.getDay(day)` (an unknown day is a `404` on `/v3`), then the event's
   `attendance` setting: unset → `NotFoundError("Attendance is off for this event")`.
2. A desk actor: `attendance !== "desks"` → `ForbiddenError("Desk check-in is off for this event")`;
   `actor.day !== day` → `ForbiddenError("This desk link is for another day")`.
3. An unknown facility → `UnknownFacilityError` (a `404` on `/v3`).
4. Read and return `peopleFromTab(values)`. A missing tab (`readValues` gives
   `null`) is `[]`, not an error: the page shows "no roster yet", as it does
   today.

**The cache (D6):** `this.peopleCache`, a `Map` from `sheetId` to `{ at, promise }`:

- A call within `PEOPLE_CACHE_MS = 10_000` of the entry's `at` returns its
  promise, so concurrent and repeated calls share one read.
- A rejected promise is removed, so a failure is not cached.
- `mark` deletes the facility's entry after a write (`plan.changed`), and so
  does `#reconcileFacility` after it writes. Delete it after the write
  resolves, so a list read cannot start between the delete and the write.
- It is per instance and in memory, like `fingerprints` and `teamsCache`.

**OpenAPI and conventions:**

- An `@openapi` block above the route, in the style of the `PUT` beside it:
  `operationId: listFacilityPeople`, tag `attendance`, `security: bearerAuth`,
  the `day` and `facility` parameters, `200` (array of
  `#/components/schemas/AttendancePerson`), and `401`, `403`, `404`, `405`,
  `409`, `502`, `503` as problem details. Add `AttendancePerson` next to
  `AttendanceRow` in `src/docs/openapiSpec.mjs`: `AttendanceRow`'s fields plus
  `teams` and `categories` (arrays of strings) and `shirt` (string).
- `test/unit/guards/rest-conventions.test.mjs`: the `EXCEPTIONS` entry for
  `PUT /v3/days/:day/facilities/:facility/people/:personKey/attendance` keeps
  its rule; change its reason to "each person's state is read with
  GET …/facilities/{facility}/people, the list a desk polls; a per-person GET
  would have no reader". Make the same change to that row of the exceptions
  table on the [API](../../technical/api.md) page.
- `test/helpers/routeManifest.mjs`: add
  `{ route: "GET /v3/days/:day/facilities/:facility/people", test: "test/integration/v3.test.mjs" }`.
- Run `npm run test:snapshots`, read the diff of
  `test/unit/docs/openapiSpec.test.mjs.snapshot`, and commit it.

Tests, written first:

- `test/unit/attendance/domain/attendanceTab.test.mjs`, `peopleFromTab`: a
  wrong header throws `AttendanceLayoutError`; a duplicate key keeps the first
  row; a blank key is skipped; a blank player reads as the key; teams and
  categories split and trim; `TRUE`, `true`, `FALSE` and booleans read as
  flags; each of the five shirt spellings is found in any column H–Z; no shirt
  column gives `""`; other columns past G are ignored.
- `test/unit/attendance/AttendanceService.test.mjs`, `listPeople`, on
  `test/helpers/fakeSheets.mjs`: attendance off → `NotFoundError`; a desk
  token for another day, or attendance `"console"` with a desk actor →
  `ForbiddenError`; an unknown facility → `UnknownFacilityError`; no tab →
  `[]`; reads `ATTENDANCE!A1:Z` with `FORMATTED_VALUE`; two calls within 10 s
  make one read, and a call after it another (inject `now`, or give the
  constructor a `peopleCacheMs`); concurrent calls make one read; a failed
  read is not cached; a `mark` that writes makes the next list call read again.
- `test/integration/v3.test.mjs`: `200` and the array with an operator token;
  `200` with a desk token for that day; `403` with one for another day; `401`
  with none; `404` for an unknown facility; `405` with `Allow: GET, HEAD` for
  a `POST`; `Cache-Control: no-store`.

Docs in the same release: [API](../../technical/api.md) (the route in the `/v3`
list, the changed exception row), [Event attendance](../../technical/event-attendance.md)
(the desk reads through the API), the `README.md` endpoint list, and the
workspace `CLAUDE.md`'s `/v3` list ("Sixteen routes" becomes seventeen, plus
the new route's line).

**Done when:** the route answers as above against the fake, `npm test` and
`npm run verify` pass, and the release is deployed.

### 9.8 Build 3: the page reads through the API (R4, site half)

`sage-match-control.github.io`, `lib/v1/views/attendance-view.js` only. Ship
after §9.7 is deployed.

- **Add an export,** beside `markPerson` and in the same style:

  ```js
  // GET one facility's people. Resolves to Person[]. Rejects with
  // Error(body.error || "HTTP <status>") and err.status; a fetch that never
  // got an answer rejects with no status.
  export async function fetchPeople({ apiBase, token, day, facility })
  ```

  It requests `${apiBase}/v3/days/${encodeURIComponent(day)}/facilities/${encodeURIComponent(facility)}/people`
  with `Authorization: Bearer ${token}` and `cache: 'no-store'`.
- **`fetchFacility`** keeps its fixture branch exactly as it is (the fixtures
  stay CSV, read by `parseAttendanceCsv`). Otherwise it returns
  `fetchPeople({ apiBase: opts.apiBase, token: opts.getToken && opts.getToken(), day: opts.day, facility: f.name })`.
  The gviz request and its `400`/`404` → `[]` rule go: the API answers `[]` for
  a missing tab.
- **A failed load** keeps the list it had and shows `Could not load <facility>: <message>`,
  as now. Add the same hints `markFailure` gives: on `401`, "This desk link has
  expired." on the desk page or "Sign in again." in Control Center; on `403`,
  "Ask the operator for a new desk link."
- **Keep every export,** `attendanceCsvUrl` included (compatibility rule), and
  update the file's header comment: the list comes from sage-tools-api, not
  the CSV export.
- Control Center mounts the view only when signed in (`attConsoleSignedIn`),
  and the desk page always has its desk token, so `getToken()` always gives a
  token outside fixture mode. No change to `tools/control-center.html` or
  `lib/v1/apps/attendance-desk.js`.

Tests: in `_tests/unit/views/attendance-view.test.mjs`, beside the
`markPerson` tests: `fetchPeople` requests the encoded URL with `GET` and the
bearer token and resolves to the body; a refused read rejects with the
server's message and its status. Keep the `attendanceCsvUrl` test. The
harness's attendance cases run in fixture mode and should not change: if one
does, the fixture branch was touched.

Docs in the same push: [Event attendance](../../technical/event-attendance.md)
and [Control Center](../../technical/control-center.md), wherever they say the
list is read from the CSV export.

### 9.9 Build 4: sharing, the runbook and the docs

Ships after §9.8, before the first event that uses it (O3).

- **`_templates/CLAUDE.md` step 13** (site repo) covers every event (D3):
  share every facility workbook with the service account as **Editor**, then
  set General access to **Restricted**. Keep its note about Drive folders
  inheriting the share.
- **`_templates/dry-run-checklist-template.md`:** add a check that each
  facility workbook's Share dialog lists the service account and General
  access reads Restricted.
- **sage-docs**, wherever a facility workbook is told to be "Anyone with the
  link: Viewer": [Build the workbooks](../../usage/build-the-workbooks.md),
  [Connect the workbooks](../../usage/connect-the-workbooks.md),
  [Troubleshooting](../../usage/troubleshooting.md) (its first check becomes
  "is the service account still on the workbook, as Editor?") and
  [Adding a new event](../../technical/adding-a-new-event.md). Find every
  mention with `grep -rn -i -E "anyone with the link|link-viewable|link-shared|gviz" docs/`.
  The masters keep "Anyone with the link: Viewer", because the calculator's
  **Make a copy** handoff needs it; say so where the masters are described.
- **This page:** §2's facility-workbook bullet points here, and the spec moves
  to `implemented/` once §9.6–§9.9 ship (see `docs/specs/README.md` for the
  procedure).

## 10. Played and series rules: two small fixes

These are not about access. They were found in the same review and touch the
one rule still kept by hand in two places:

- `sage-tools-api/src/sync/domain/facilityCompletion.mjs` decides when a
  facility finished, and the sync publishes it as `facilities[].completedAt`.
- The site's `lib/v1/domain/` (`byes.js`, `series.js`, `matches.js`,
  `progress.js`) decides when the facility card reads "All matches done".
- `sage-match-control.github.io/_tests/unit/parity-server.test.mjs` runs both
  on the same inputs and fails if they disagree. It imports the server file
  from the sibling `sage-tools-api` checkout.

Both fixes go in one `sage-tools-api` **patch** release. The rules in §9.5
apply.

### 10.1 Reading a score's value

Both sides count a cell as a score when its trimmed text is not blank and is a
number (`!isNaN`). They read its value differently:

| | Reads with | `11.5` | `1e1` |
| --- | --- | --- | --- |
| The site: `rowsToMatches` in `lib/v1/domain/matches.js` | `parseInt(raw, 10)` | 11 | 1 |
| The server: `unneededSeriesGames` in `facilityCompletion.mjs` | `Number(raw)` | 11.5 | 10 |

Only the series-final walk compares values on the server, so that is where they
disagree. In a twice-to-beat final (two games, codes `X_F_1_(1)`/`X_F_2_(1)`
and `X_F_1_(2)`/`X_F_2_(2)`), game 1 typed `11.5`–`11` is a seat-1 win on the
server, which decides the series and makes game 2 unneeded, so the facility is
complete and `completedAt` is stamped. On the site it is a tie, game 2 is still
needed, and the card says one match left. Score entry writes whole numbers only
(`src/scores/domain/scoreRequest.mjs`), but a score typed into the sheet is not
checked.

**The fix:** in `unneededSeriesGames`, replace
`const s1 = Number(g.score1), s2 = Number(g.score2);` with
`const s1 = parseInt(String(g.score1).trim(), 10), s2 = parseInt(String(g.score2).trim(), 10);`.
The site's rule wins because it is the value every page shows. `hasScore` is
unchanged.

**Tests, written first:**

- `test/unit/sync/domain/facilityCompletion.test.mjs`, in the "series finals"
  `describe`: `unneededSeriesGames` with game 1 `"11.5"`–`"11"` and game 2
  unscored gives an empty set (it gives game 2's key before the fix); and
  `facilityIsComplete` on the same two rows as CSV is `false`.
- In the site repo, add to `CASES` in `_tests/unit/parity-server.test.mjs`:

  ```js
  'a decimal score in a series final': csvOf(
    row(1, 'X_F_1_(1)', 'a', 'b', '11.5', 'X_F_2_(1)', 'c', 'd', '11'),
    row(2, 'X_F_1_(2)', 'a', 'b', '', 'X_F_2_(2)', 'c', 'd', '')),
  ```

  Push it after the server fix is on `main`: until then the case fails,
  because it imports the server's file from the sibling checkout.

### 10.2 Stale comments

Fix these in the same release:

- **`facilityCompletion.mjs`'s header** says the rules "mirror
  control-center.html exactly (parseCSV, rowsToMatches, sideIsBye,
  seriesGameOf/unneededSeriesGames, and computeFacilityProgress's `left`)".
  Those functions live in the site's `lib/v1/domain/` now. Say so, naming
  `csv.js`, `matches.js`, `byes.js`, `series.js` and `progress.js`, and name
  `_tests/unit/parity-server.test.mjs` as the check that they agree.
- **`SheetsCsvFetcher.mjs`'s header** says parsing "stays client-side in the
  site's HTML as the single source of truth, so this service and the browser
  never have two copies". `facilityCompletion.mjs` parses the matches CSV and
  `teamRoster.mjs` the Teams tab. Say that this class only fetches and
  converts values to CSV, and that the sync's domain files parse what they
  need.

**Done when:** `npm test` and `npm run verify` pass in `sage-tools-api`, then
`npm run verify` passes in the site's `_tests` with the new case, and the
Changelog entry's **Tests:** line names `facilityCompletion.test.mjs`.

## 11. Deferred

- **Tournament Calculator logic on the server.** Its planning core (tiered
  byes, wildcard fill, slot-aware finish times) is the one piece of front-end
  logic with real originality. It could move behind an endpoint on the
  `sage-live` Worker (free, no cold start), at the cost of offline use and a
  network call on every change. Not worth it unless it is actually copied.
- **Private site repo.** Moving the site to Cloudflare Pages would let
  `sage-match-control.github.io` go private, but only hides its history,
  templates and tests: the served code stays readable.
