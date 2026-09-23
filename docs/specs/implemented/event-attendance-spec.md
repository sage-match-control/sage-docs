# Spec — Event attendance

A staff check-in page for an event's desk: open it on any phone, find a
player, flip their switch, and they are marked in with the time they arrived.
The marks are written to the event's scoring workbook. Built for **Pickle for
Sight, Sunday 27 September 2026**, first.

> **Status: built.** `attendance.gs`, `verify-attendance.mjs` and the
> Pickle for Sight page exist, and every check in §5 passes. Reaching a
> workbook is §6, a person's job: until it is done the page shows both
> venues as "not connected".
>
> **Revised after §6 was done** — see §11. The page's reads no longer go
> through `attendance.gs` at all; §5's code blocks below are the original
> build and are left as history, not the current file contents.

| | |
| --- | --- |
| Where the code goes | `sage-tools-api/scripts/attendance.gs` and `verify-attendance.mjs` (new); `sage-match-control.github.io/events/pickle-for-sight-2026/attendance.html` (new) |
| How it ships | `attendance.gs` is pasted into the two live workbooks and deployed there as a web app. The page is a static-site commit |
| New infra | two Apps Script web-app deployments, one per venue workbook |
| New credentials | none |
| `sage-tools-api` change | none to the service — no version bump, no Cloud Run deploy. `scripts/*.gs` is not part of it |
| Reads | The page reads each venue workbook's `ATTENDANCE` tab straight from its published CSV export — `attendance.gs` is not involved (§11). `attendance.gs` itself still reads `STANDINGSCSV` columns A:C, to resolve the roster on write and on resync |
| Writes | each venue workbook's `ATTENDANCE` tab, which the script creates — nothing else |

Context: [Pickle for Sight event spec](../in-progress/pickle-for-sight-spec.md).
The roster tab is described in the
[Standard Tournament Master spec](../implemented/standard-tournament-master-spec.md) §10.4.

---

## 0. Who does what

### Implementer (§5)

| Repo | File | Change |
| --- | --- | --- |
| `sage-tools-api` | `scripts/attendance.gs` | new, whole file (§5.1) |
| `sage-tools-api` | `scripts/verify-attendance.mjs` | new, whole file (§5.2) |
| `sage-match-control.github.io` | `events/pickle-for-sight-2026/attendance.html` | new, whole file (§5.3) |
| `sage-docs` | `docs/features/event-attendance.md`, `docs/technical/event-attendance.md` | new pages (§5.5) |
| `sage-docs` | both section READMEs, `mkdocs.yml` | index the new pages (§5.5) |
| `sage-docs` | this spec and the spec indexes | file it as implemented (§5.8) |
| none (plain folder) | `D:\Coding Projects\SAGE\CLAUDE.md` | three edits (§5.6) |

Copy the code blocks exactly. Do not reformat, rename or "improve" them —
every one was run as written.

Commit each repo once, on `main`: `sage-tools-api` and the site at the end of
§5.7, `sage-docs` at the end of §5.8. **Do not push.** `CLAUDE.md` is not in
any repo, so it is edited but not committed.

### Operator (§6) — a person, not the implementer

Pasting `attendance.gs` into the workbooks, deploying the two web apps,
putting their URLs into the page, pushing, and the on-the-day checks. These
need the SAGE Google account and a browser; the implementer cannot do them.

### Do not touch

- `sheets-sync.gs`, `standard-generator.gs`, `sheet-generator.gs`,
  `mock-apps-script.mjs`, anything in `sage-tools-api/src/`, and
  `package.json` (no version bump — see the table above).
- The masters. `attendance.gs` goes into the two **live** workbooks only.
- Any tab except `ATTENDANCE`. The script only ever writes that one.
- The event's `index.html` and `schedule.html`. Do not link to the attendance
  page from anywhere — it is for desk staff only.
- `event-data` and its `config/events.json`. Attendance does not go through
  the sync.

### Blocked — do not fake

- **The two web-app URLs.** They exist only after §6 is done. Leave the
  `PASTE_PCPH_MAIN_EXEC_URL` and `PASTE_PCPH_ANNEX_EXEC_URL` placeholders
  exactly as written; the page shows such a venue as "not connected".
- **A real deployment test.** §5's checks run against a mock and a stub.

### Stop and ask

Stop and ask rather than working around it if:

- any check in §5 fails and the fix would mean changing a file on the
  do-not-touch list, or changing a code block in this spec;
- `verify-attendance.mjs` reports a name shared with another `.gs` file —
  do **not** rename anything in the other file;
- `event-data/pickle-for-sight-2026/data/pickle-for-sight-day1.json` has no
  `teamCode`, `player1` or `player2` column in its `standingsCsv`;
- the `DIVISIONS` or `EVENTS` constants in
  `events/pickle-for-sight-2026/index.html` differ from the ones in §5.3;
- an anchor quoted in §5.5–§5.8 is not found exactly once.

---

## 1. What it is for

On the day, desk staff at each venue mark players in as they arrive. Staff
are the organizer's people, not SAGE's, and have no SAGE sign-in. They use
their own phones, several at once per venue.

Pickle for Sight has two venue workbooks, each with its own roster:

| Venue | Workbook (live, registered in `event-data`) | Pairs | Players |
| --- | --- | --- | --- |
| PCPH Main | `2026-09-27 Pickle for Sight Tournament - MAIN` (`1rNIlK2Zz3zTaLILlpq4rCGIQbtr0ECpG4A3OdP4v25A`) | 51 | 102 |
| PCPH Annex | `2026-09-27 Pickle for Sight Tournament - ANNEX` (`1ks84WK7vo5FAenpPbZocm9PLPpLoS6_SRPivVJqmS-0`) | 59 | 118 |

Counts are from the published snapshot of 2026-09-23. Attendance is marked
**per player**, with the time each one was marked in.

## 2. Data

### 2.1 The roster — `STANDINGSCSV`, read only

Columns A:C are `teamCode`, `player1`, `player2`, from row 2. `A2` is a spill
(`=UNIQUE(FILTER('Reference for Players'!$A$3:$A, …))`), so the tab lists
**playoff seats** as well as pairs — `HIMD_QF_1`, `HIMD_SF_2` and so on —
whose names repeat pairs already listed.

- A row is a pair only when its code matches `^[A-Z0-9]+_\d+$` (`HIMD_3`,
  `LIXD_12`). Everything else is skipped.
- A player is listed only when their cell is non-blank and not the team code
  itself (an unnamed pair's cells show its code).
- A pair with no listed player is skipped.
- `category` is the code before the `_` (`HIMD`).

### 2.2 The marks — the `ATTENDANCE` tab

| A | B | C | D | E |
| --- | --- | --- | --- | --- |
| `teamCode` | `slot` | `name` | `present` | `timeIn` |
| `HIMD_1` | `1` | `Juan Dela Cruz` | `TRUE` | `2026-09-27 08:14` |

- Created by the script on the first **POST**, never on a GET. Columns C and
  E are formatted as plain text.
- One row per player ever marked, found by `teamCode` + `slot` — never by
  row position. Unmarking keeps the row and sets `present` to `FALSE`.
- `name` is the player's name when marked. A mark counts **only while that
  name still matches** the roster: after a re-draw gives the code and slot to
  someone else, the old mark does not carry over.
- `timeIn` is `yyyy-MM-dd HH:mm` in the spreadsheet's time zone. Marking a
  player who is already in keeps the **first** time; unmarking clears it.
- Since §11, the tab is also pre-filled with every player (not just those
  marked) by `attendanceResync`, run once at setup — the page's read no
  longer merges in `STANDINGSCSV` itself, so an unmarked player has to
  already be a row for the page to show them at all.

## 3. The web app

One per venue workbook, deployed with **Execute as: Me** and **Who has
access: Anyone**. As built here, `doGet` is no longer read by the page
(§11) — it's a diagnostic endpoint now, useful to eyeball from a browser,
but `attendanceResync` (§11) is what the page's reads actually depend on.

**GET** `…/exec` →

```json
{ "ok": true,
  "pairs": [ { "teamCode": "HIMD_1", "category": "HIMD",
               "players": [ { "slot": 1, "name": "Juan Dela Cruz", "present": true, "timeIn": "2026-09-27 08:14" },
                            { "slot": 2, "name": "Maria Santos", "present": false, "timeIn": "" } ] } ] }
```

**POST** `…/exec`, body (sent as text) `{"teamCode":"HIMD_1","slot":2,"present":true}` →

```json
{ "ok": true, "teamCode": "HIMD_1", "slot": 2, "name": "Maria Santos", "present": true, "timeIn": "2026-09-27 08:21" }
```

**Failure** → `{ "ok": false, "error": "…" }`. Always HTTP 200:
`ContentService` cannot set a status. A POST is refused, writing nothing,
when the body is not JSON, `present` is not a boolean, the code is not a
pair row (§2.1), or the slot has no player. Every POST runs under the script
lock.

## 4. The page

`events/pickle-for-sight-2026/attendance.html`, served by GitHub Pages at
`/events/pickle-for-sight-2026/attendance`. Self-contained like the event's
`schedule.html`, with the same colour and font tokens; always light, like the
staff tools. `noindex`, and linked from nowhere.

As built here, reads described below go straight to the workbook's
published CSV export of `ATTENDANCE`, not through the web app — see §11
for why and what that changes.

- A **venue switch** (PCPH Main / PCPH Annex), remembered per phone.
- A **count** of players in at that venue, e.g. `84 / 118 in`.
- **Categories** in division then event order (Novice → Low Intermediate →
  High Intermediate; Men's → Women's → Mixed), each with its own count.
- Each **pair** as a card: code, then one row per player with a switch. A pair
  with every player in shows a **Ready** badge and a green edge.
- **Search** over names and team codes.
- A switch updates **at once**, shows *Saving…* and locks until the reply.
  On success it shows *In HH:mm*; on failure it **snaps back** and the status
  line says why.
- **Refresh** button, a poll every 30 seconds while the page is visible, and
  a reload when it becomes visible again. A player with a save in flight
  keeps its local state through a reload.
- A venue whose URL is still a `PASTE_` placeholder shows "*Venue* is not
  connected yet." and is never fetched.

---

## 5. Implementation guide

Work from `D:\Coding Projects\SAGE`. Apply the steps in order.

### 5.1 `sage-tools-api/scripts/attendance.gs` — new file

Create it with exactly this content:

```js
/**
 * Event attendance (Google Apps Script web app)
 * ============================================================================
 * Implements sage-docs/docs/specs/.../event-attendance-spec.md.
 *
 * A web app bound to one live event workbook. The staff check-in page
 * (sage-match-control.github.io/events/<event>/attendance.html) reads the
 * roster with GET and marks one player with POST. Names come from
 * STANDINGSCSV, read-only. Marks go to an ATTENDANCE tab this script creates
 * on first use, keyed by team code and player slot. It writes nothing else.
 *
 * ----------------------------------------------------------------------------
 * INSTALL — once per live workbook, never in a master
 * ----------------------------------------------------------------------------
 *   1. Open the workbook -> Extensions -> Apps Script.
 *   2. File "+" -> Script, name it "attendance", paste this whole file.
 *      Leave sheets-sync.gs and the generator file untouched.
 *   3. Deploy -> New deployment -> type: Web app.
 *      Execute as: Me. Who has access: Anyone. Deploy, authorize.
 *   4. Copy the Web app URL (ends in /exec) into the VENUES list at the
 *      bottom of that event's attendance.html.
 *
 * After changing this file: Deploy -> Manage deployments -> edit -> Version:
 * New version. Saving alone does not change what the /exec URL runs.
 * ============================================================================
 */

var ATTENDANCE_STANDINGS_SHEET = 'STANDINGSCSV';
var ATTENDANCE_SHEET = 'ATTENDANCE';
var ATTENDANCE_HEADERS = ['teamCode', 'slot', 'name', 'present', 'timeIn'];
var ATTENDANCE_TEAM_CODE = /^[A-Z0-9]+_\d+$/;
var ATTENDANCE_TIME_FORMAT = 'yyyy-MM-dd HH:mm';

function doGet() {
  return attendanceRespond_(function () {
    return { pairs: attendanceRoster_() };
  });
}

function doPost(e) {
  return attendanceRespond_(function () {
    var body = JSON.parse((e && e.postData && e.postData.contents) || '{}');
    return attendanceMark_(body.teamCode, body.slot, body.present);
  });
}

// ContentService cannot set an HTTP status, so every reply is 200 and says
// whether it worked in `ok`.
function attendanceRespond_(work) {
  var out;
  try {
    out = work();
    out.ok = true;
  } catch (err) {
    out = { ok: false, error: String((err && err.message) || err) };
  }
  return ContentService.createTextOutput(JSON.stringify(out))
    .setMimeType(ContentService.MimeType.JSON);
}

// Every pair row of STANDINGSCSV that has a name. Playoff-seat rows
// (HIMD_QF_1) are skipped: they repeat the names of pairs already listed.
function attendanceTeams_(ss) {
  var sheet = ss.getSheetByName(ATTENDANCE_STANDINGS_SHEET);
  if (!sheet) throw new Error('This workbook has no "' + ATTENDANCE_STANDINGS_SHEET + '" tab.');
  var last = sheet.getLastRow();
  if (last < 2) return [];
  var out = [];
  sheet.getRange(2, 1, last - 1, 3).getValues().forEach(function (r) {
    var code = String(r[0]).trim();
    if (!ATTENDANCE_TEAM_CODE.test(code)) return;
    var players = [];
    [r[1], r[2]].forEach(function (v, i) {
      var name = String(v).trim();
      if (name && name !== code) players.push({ slot: i + 1, name: name });
    });
    if (players.length) out.push({ teamCode: code, category: code.split('_')[0], players: players });
  });
  return out;
}

function attendanceSheet_(ss, create) {
  var sheet = ss.getSheetByName(ATTENDANCE_SHEET);
  if (sheet || !create) return sheet;
  sheet = ss.insertSheet(ATTENDANCE_SHEET);
  sheet.getRange(1, 1, 1, ATTENDANCE_HEADERS.length).setValues([ATTENDANCE_HEADERS]);
  // Plain text, or Sheets turns a stamp into a date and a numeric name into a number.
  sheet.getRange(1, 3, sheet.getMaxRows(), 1).setNumberFormat('@');
  sheet.getRange(1, 5, sheet.getMaxRows(), 1).setNumberFormat('@');
  return sheet;
}

// "teamCode|slot" -> { row, name, present, timeIn }
function attendanceMarks_(sheet) {
  var marks = {};
  if (!sheet) return marks;
  var last = sheet.getLastRow();
  if (last < 2) return marks;
  sheet.getRange(2, 1, last - 1, 5).getValues().forEach(function (r, i) {
    marks[String(r[0]).trim() + '|' + Number(r[1])] = {
      row: i + 2,
      name: String(r[2]).trim(),
      present: r[3] === true,
      timeIn: String(r[4]),
    };
  });
  return marks;
}

function attendanceRoster_() {
  var ss = SpreadsheetApp.getActiveSpreadsheet();
  var marks = attendanceMarks_(attendanceSheet_(ss, false));
  var teams = attendanceTeams_(ss);
  teams.forEach(function (t) {
    t.players.forEach(function (p) {
      var m = marks[t.teamCode + '|' + p.slot];
      // A mark counts only for the player it was made for: after a re-draw
      // the same code and slot can belong to someone else.
      var mine = !!(m && m.name === p.name && m.present);
      p.present = mine;
      p.timeIn = mine ? m.timeIn : '';
    });
  });
  return teams;
}

function attendanceMark_(teamCode, slot, present) {
  teamCode = String(teamCode || '').trim();
  slot = Number(slot);
  if (typeof present !== 'boolean') throw new Error('"present" must be true or false.');
  var lock = LockService.getScriptLock();
  lock.waitLock(10000);
  try {
    var ss = SpreadsheetApp.getActiveSpreadsheet();
    var team = attendanceTeams_(ss).filter(function (t) { return t.teamCode === teamCode; })[0];
    var player = team && team.players.filter(function (p) { return p.slot === slot; })[0];
    if (!player) throw new Error('No player ' + slot + ' on ' + teamCode + '. Refresh the page.');
    var sheet = attendanceSheet_(ss, true);
    var mark = attendanceMarks_(sheet)[teamCode + '|' + slot];
    // Marking someone already in keeps their first time, so a double tap
    // or two phones marking the same player do not move it.
    var already = !!(mark && mark.name === player.name && mark.present);
    var timeIn = '';
    if (present) {
      timeIn = already ? mark.timeIn
        : Utilities.formatDate(new Date(), ss.getSpreadsheetTimeZone(), ATTENDANCE_TIME_FORMAT);
    }
    var row = mark ? mark.row : Math.max(sheet.getLastRow(), 1) + 1;
    sheet.getRange(row, 1, 1, 5).setValues([[teamCode, slot, player.name, present, timeIn]]);
    SpreadsheetApp.flush();
    return { teamCode: teamCode, slot: slot, name: player.name, present: present, timeIn: timeIn };
  } finally {
    lock.releaseLock();
  }
}
```

**Check:**

```bash
cd "D:/Coding Projects/SAGE/sage-tools-api" && node -e "new Function(require('fs').readFileSync('scripts/attendance.gs','utf8'))" && echo "GS OK"
```

→ `GS OK`.

### 5.2 `sage-tools-api/scripts/verify-attendance.mjs` — new file

The harness. It loads `attendance.gs` into the existing mock and supplies the
three services the mock lacks (`LockService`, `ContentService`, `Utilities`)
itself, so `mock-apps-script.mjs` is not touched.

```js
#!/usr/bin/env node
// Runs attendance.gs's real doGet/doPost against the mocked Sheets service in
// mock-apps-script.mjs, and checks the one rule a live workbook cannot afford
// to learn the hard way: attendance.gs shares a script project with
// sheets-sync.gs and a generator, where a repeated top-level name silently
// replaces the other file's.
//
//   node scripts/verify-attendance.mjs
import fs from "node:fs";
import vm from "node:vm";
import path from "node:path";
import { fileURLToPath } from "node:url";
import { MockSpreadsheet, buildSandbox } from "./mock-apps-script.mjs";

const here = path.dirname(fileURLToPath(import.meta.url));
const read = (name) => fs.readFileSync(path.join(here, name), "utf8");
let failures = 0;

function check(label, actual, expected) {
    const ok = String(actual) === String(expected);
    console.log(`  ${ok ? "OK  " : "FAIL"} ${label.padEnd(56)} = ${actual}` +
        (ok ? "" : `   (expected ${expected})`));
    if (!ok) failures++;
}

// ------------------------------------------------------------- name clashes
console.log("\n########## top-level names ##########");
const topLevel = (src) => new Set(
    [...src.matchAll(/^(?:var|let|const|function)\s+([A-Za-z_$][\w$]*)/gm)].map((m) => m[1]));
const mine = topLevel(read("attendance.gs"));
for (const other of ["sheets-sync.gs", "standard-generator.gs", "sheet-generator.gs"]) {
    const clash = [...topLevel(read(other))].filter((n) => mine.has(n));
    check(`no name shared with ${other}`, clash.join(", ") || "none", "none");
}

// ---------------------------------------------------------------- workbook
function workbook() {
    const ss = new MockSpreadsheet();
    ss.insertSheet = (name) => ss.addSheet(name, 1000, 26);
    ss.getSpreadsheetTimeZone = () => "Asia/Manila";
    const standings = ss.addSheet("STANDINGSCSV", 20, 7);
    const rows = [
        ["teamCode", "player1", "player2", "wins", "loss", "quotient", "bracket"],
        ["HIMD_1", "Juan Dela Cruz", "Maria Santos", 0, 0, 0, 1],
        ["HIMD_2", "Team Thunder", "", 0, 0, 0, 1],
        ["HIMD_3", "HIMD_3", "HIMD_3", 0, 0, 0, 1],     // not yet named
        ["HIMD_SF_1", "Juan Dela Cruz", "Maria Santos", 0, 0, 0, ""],
        ["LIXD_12", "Ana Reyes", "Ben Cruz", 0, 0, 0, 2],
        ["", "", "", "", "", "", ""],
    ];
    standings.getRange(1, 1, rows.length, 7).setValues(rows);
    return ss;
}

function load(ss) {
    const sandbox = buildSandbox(ss);
    const lock = { waits: 0, releases: 0 };
    let stamps = 0;
    Object.assign(sandbox, {
        LockService: {
            getScriptLock: () => ({
                waitLock: () => { lock.waits++; },
                releaseLock: () => { lock.releases++; },
            }),
        },
        ContentService: {
            MimeType: { JSON: "JSON" },
            createTextOutput: (text) => {
                const out = { text, mime: null, setMimeType: (m) => { out.mime = m; return out; } };
                return out;
            },
        },
        Utilities: { formatDate: () => `2026-09-27 08:${String(14 + stamps++).padStart(2, "0")}` },
    });
    vm.createContext(sandbox);
    vm.runInContext(read("attendance.gs"), sandbox, { filename: "attendance.gs" });
    const get = () => JSON.parse(sandbox.doGet().text);
    const post = (body) => JSON.parse(sandbox.doPost(
        { postData: { contents: typeof body === "string" ? body : JSON.stringify(body) } }).text);
    return { get, post, lock, mime: () => sandbox.doGet().mime };
}

const cells = (sheet) => JSON.stringify([...sheet.cells.entries()].sort());

// ----------------------------------------------------------------- roster
console.log("\n########## roster ##########");
const ss = workbook();
const standingsBefore = cells(ss.getSheetByName("STANDINGSCSV"));
const { get, post, lock, mime } = load(ss);

let res = get();
check("GET ok", res.ok, true);
check("replies are JSON", mime(), "JSON");
check("pairs listed", res.pairs.map((p) => p.teamCode).join(","), "HIMD_1,HIMD_2,LIXD_12");
check("unnamed pair and playoff seat skipped", res.pairs.some((p) => /HIMD_3|SF/.test(p.teamCode)), false);
check("HIMD_1 players", res.pairs[0].players.map((p) => `${p.slot}:${p.name}`).join("|"),
    "1:Juan Dela Cruz|2:Maria Santos");
check("a one-name pair has one player", res.pairs[1].players.length, 1);
check("category from the code", res.pairs[2].category, "LIXD");
check("nobody present yet", res.pairs.flatMap((p) => p.players).some((p) => p.present), false);
check("GET creates no ATTENDANCE tab", ss.getSheetByName("ATTENDANCE"), null);

// ------------------------------------------------------------------- marks
console.log("\n########## marking ##########");
res = post({ teamCode: "HIMD_1", slot: 1, present: true });
check("mark ok", res.ok, true);
check("stamped", res.timeIn, "2026-09-27 08:14");
const att = ss.getSheetByName("ATTENDANCE");
check("ATTENDANCE created", !!att, true);
check("header row", att.getRange(1, 1, 1, 5).getValues()[0].join(","), "teamCode,slot,name,present,timeIn");
check("first mark on row 2", att.getRange(2, 1, 1, 5).getValues()[0].join(","),
    "HIMD_1,1,Juan Dela Cruz,true,2026-09-27 08:14");

res = get();
const juan = () => get().pairs[0].players[0];
check("GET sees the mark", `${res.pairs[0].players[0].present} ${res.pairs[0].players[0].timeIn}`,
    "true 2026-09-27 08:14");
check("partner still out", res.pairs[0].players[1].present, false);

res = post({ teamCode: "HIMD_1", slot: 1, present: true });
check("marking again keeps the first time", res.timeIn, "2026-09-27 08:14");
check("and adds no row", att.getLastRow(), 2);

res = post({ teamCode: "HIMD_1", slot: 1, present: false });
check("unmark ok", `${res.ok} ${res.present} [${res.timeIn}]`, "true false []");
check("unmark clears the time", `${juan().present} [${juan().timeIn}]`, "false []");
check("unmark reuses the row", att.getLastRow(), 2);

res = post({ teamCode: "HIMD_1", slot: 1, present: true });
check("re-mark gets a new time", res.timeIn, "2026-09-27 08:15");
res = post({ teamCode: "LIXD_12", slot: 2, present: true });
check("second player appended", att.getLastRow(), 3);

// ---------------------------------------------------------------- refusals
console.log("\n########## refusals ##########");
const refused = (body) => { const r = post(body); return r.ok === false && typeof r.error === "string"; };
check("unknown team", refused({ teamCode: "HIMD_99", slot: 1, present: true }), true);
check("playoff seat", refused({ teamCode: "HIMD_SF_1", slot: 1, present: true }), true);
check("unnamed pair", refused({ teamCode: "HIMD_3", slot: 1, present: true }), true);
check("missing second player", refused({ teamCode: "HIMD_2", slot: 2, present: true }), true);
check("present not a boolean", refused({ teamCode: "HIMD_1", slot: 1, present: "yes" }), true);
check("body not JSON", refused("not json"), true);
check("empty body", refused(""), true);
check("refusals wrote nothing", att.getLastRow(), 3);

// ----------------------------------------------------------------- re-draw
console.log("\n########## re-draw ##########");
ss.getSheetByName("STANDINGSCSV").getRange(2, 2).setValue("New Person");
res = get();
check("a mark does not follow the code to a new player", res.pairs[0].players[0].present, false);
check("the untouched partner is unaffected", res.pairs[0].players[1].present, false);

// ------------------------------------------------------------------ safety
console.log("\n########## safety ##########");
check("every lock taken was released", lock.waits > 0 && lock.waits === lock.releases, true);
ss.getSheetByName("STANDINGSCSV").getRange(2, 2).setValue("Juan Dela Cruz");
check("STANDINGSCSV never written", cells(ss.getSheetByName("STANDINGSCSV")) === standingsBefore, true);

console.log(failures ? `\n*** ${failures} CHECK(S) FAILED` : "\nALL CHECKS PASSED");
process.exit(failures ? 1 : 0);
```

**Check:**

```bash
cd "D:/Coding Projects/SAGE/sage-tools-api" && node scripts/verify-attendance.mjs
```

→ 38 lines starting `OK`, none starting `FAIL`, and last line
`ALL CHECKS PASSED`.

### 5.3 `sage-match-control.github.io/events/pickle-for-sight-2026/attendance.html` — new file

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<meta name="robots" content="noindex, nofollow">
<title>Pickle for Sight — Attendance</title>
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Archivo+Black&family=Barlow+Condensed:wght@500;600;700&family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">
<style>
  /* Same tokens as schedule.html. Staff pages stay light, like the tools. */
  :root{
    --navy:#05133B;
    --navy-deep:#020B29;
    --green:#5E9106;
    --green-dark:#3C6B02;
    --paper:#F5F6F8;
    --paper-dim:#E9ECF1;
    --line:#D3D9E3;
    --ink:#05133B;
    --ink-soft:#4A5572;
    --white:#FFFFFF;
    --red:#B42318;
    --radius:14px;
  }
  *{box-sizing:border-box;}
  html,body{margin:0;padding:0;}
  body{
    background:var(--paper);
    color:var(--ink);
    font-family:'Inter',sans-serif;
    font-size:16px;
    line-height:1.4;
    -webkit-font-smoothing:antialiased;
  }
  .wrap{max-width:720px;margin:0 auto;padding:0 16px;}

  header.top{
    position:sticky;top:0;z-index:2;
    background:var(--navy);
    color:var(--white);
    padding:14px 0 12px;
    box-shadow:0 6px 18px -12px rgba(2,11,41,.6);
  }
  .kicker{
    font-family:'Barlow Condensed',sans-serif;
    font-weight:600;
    font-size:12.5px;
    letter-spacing:.22em;
    text-transform:uppercase;
    color:#9CCB4A;
  }
  h1{
    font-family:'Archivo Black',sans-serif;
    font-weight:400;
    text-transform:uppercase;
    font-size:26px;
    margin:2px 0 12px;
  }
  .venues{display:flex;gap:8px;margin-bottom:10px;}
  .venues button{
    flex:1;
    font:600 14px 'Inter',sans-serif;
    padding:9px 10px;
    border-radius:10px;
    border:1px solid rgba(255,255,255,.28);
    background:transparent;
    color:var(--white);
    cursor:pointer;
  }
  .venues button[aria-pressed="true"]{background:var(--white);color:var(--navy);border-color:var(--white);}
  .bar{display:flex;gap:8px;}
  .bar input{
    flex:1;min-width:0;
    font:16px 'Inter',sans-serif;
    padding:10px 12px;
    border-radius:10px;
    border:0;
    color:var(--ink);
  }
  .bar button{
    font:600 14px 'Inter',sans-serif;
    padding:0 14px;
    border-radius:10px;
    border:0;
    background:var(--green);
    color:var(--white);
    cursor:pointer;
  }
  .meta{
    display:flex;justify-content:space-between;gap:12px;
    margin-top:8px;
    font-size:13px;
    min-height:18px;
  }
  #count{font-weight:700;}
  #status{color:#D6DCE8;text-align:right;}
  #status.error{color:#FFB4AB;font-weight:600;}

  main{padding-top:14px;padding-bottom:60px;}
  .cat{margin:0 0 22px;}
  .cat h2{
    display:flex;justify-content:space-between;align-items:baseline;gap:10px;
    font-family:'Barlow Condensed',sans-serif;
    font-weight:700;
    text-transform:uppercase;
    letter-spacing:.05em;
    font-size:17px;
    margin:0 0 8px;
  }
  .cat h2 span{font-size:13px;color:var(--ink-soft);letter-spacing:.08em;}
  .pair{
    background:var(--white);
    border:1px solid var(--line);
    border-left:4px solid var(--line);
    border-radius:var(--radius);
    margin-bottom:8px;
    overflow:hidden;
  }
  .pair.ready{border-left-color:var(--green);}
  .pair-head{
    display:flex;justify-content:space-between;align-items:center;
    padding:8px 14px 0;
    font-family:'Barlow Condensed',sans-serif;
    font-size:13px;
    font-weight:600;
    letter-spacing:.08em;
    color:var(--ink-soft);
  }
  .badge{
    font-size:11px;
    letter-spacing:.12em;
    text-transform:uppercase;
    color:var(--white);
    background:var(--green);
    border-radius:20px;
    padding:2px 8px;
  }
  .player{
    display:flex;align-items:center;justify-content:space-between;gap:12px;
    padding:10px 14px;
    cursor:pointer;
  }
  .player + .player{border-top:1px solid var(--paper-dim);}
  .who{min-width:0;}
  .name{font-weight:600;overflow-wrap:anywhere;}
  .time{display:block;font-size:12.5px;color:var(--ink-soft);}
  .player.in .time{color:var(--green-dark);font-weight:600;}

  .switch{
    appearance:none;-webkit-appearance:none;
    flex:none;
    width:52px;height:30px;margin:0;
    border-radius:30px;
    background:var(--line);
    position:relative;
    cursor:pointer;
    transition:background .15s;
  }
  .switch::after{
    content:"";
    position:absolute;top:3px;left:3px;
    width:24px;height:24px;border-radius:50%;
    background:var(--white);
    box-shadow:0 1px 3px rgba(0,0,0,.25);
    transition:transform .15s;
  }
  .switch:checked{background:var(--green);}
  .switch:checked::after{transform:translateX(22px);}
  .switch:disabled{opacity:.55;cursor:progress;}
  .switch:focus-visible{outline:3px solid var(--green);outline-offset:2px;}
  .empty{color:var(--ink-soft);text-align:center;padding:40px 0;}
  @media (prefers-reduced-motion: reduce){ .switch,.switch::after{transition:none;} }
</style>
</head>
<body>
<header class="top">
  <div class="wrap">
    <div class="kicker">Pickle for Sight &middot; Staff</div>
    <h1>Attendance</h1>
    <div class="venues" id="venues"></div>
    <div class="bar">
      <input type="search" id="search" placeholder="Search a name or code" autocomplete="off" aria-label="Search a name or code">
      <button type="button" id="refresh">Refresh</button>
    </div>
    <div class="meta"><span id="count"></span><span id="status" role="status"></span></div>
  </div>
</header>
<main class="wrap" id="list"></main>

<script>
// ============================================================================
// CONFIGURATION
// ============================================================================
const EVENT_KEY = 'pickle-for-sight-2026';

// One entry per venue workbook. `url` is that workbook's attendance web app:
// Deploy -> Web app URL, ending in /exec. Setup is in the header of
// sage-tools-api/scripts/attendance.gs. A venue still holding a PASTE_ value
// shows as not connected instead of being fetched.
const VENUES = [
  { name: 'PCPH Main',  url: 'PASTE_PCPH_MAIN_EXEC_URL' },
  { name: 'PCPH Annex', url: 'PASTE_PCPH_ANNEX_EXEC_URL' },
];

// Category names, as in this event's index.html (DIVISIONS / EVENTS).
const DIVISIONS = { N: 'Novice', LI: 'Low Intermediate', HI: 'High Intermediate' };
const EVENTS = { MD: "Men's Doubles", WD: "Women's Doubles", XD: 'Mixed Doubles' };

// Other phones' marks show up within this long.
const POLL_MS = 30000;

// ============================================================================
// STATE
// ============================================================================
const STORAGE_VENUE_KEY = `${EVENT_KEY}.attendanceVenue`;
let venueIndex = loadVenue();
let pairs = [];
let loadSeq = 0;             // a reply for a venue no longer shown is dropped
const saving = new Set();    // "teamCode|slot" with a save in flight

const $ = (id) => document.getElementById(id);

function loadVenue(){
  try{ const i = Number(localStorage.getItem(STORAGE_VENUE_KEY)); return VENUES[i] ? i : 0; }
  catch(e){ return 0; }
}
function saveVenue(i){ try{ localStorage.setItem(STORAGE_VENUE_KEY, String(i)); }catch(e){} }

function isConnected(venue){
  return /^https:\/\/script\.google\.com\/macros\/s\/[^/]+\/exec$/.test(venue.url);
}

function categoryName(key){
  for(const d of Object.keys(DIVISIONS).sort((a, b) => b.length - a.length)){
    if(key.startsWith(d) && EVENTS[key.slice(d.length)]) return `${DIVISIONS[d]} ${EVENTS[key.slice(d.length)]}`;
  }
  return key;
}
function categoryRank(key){
  const divs = Object.keys(DIVISIONS), evs = Object.keys(EVENTS);
  for(const d of divs.slice().sort((a, b) => b.length - a.length)){
    const e = key.slice(d.length);
    if(key.startsWith(d) && EVENTS[e]) return divs.indexOf(d) * evs.length + evs.indexOf(e);
  }
  return Number.MAX_SAFE_INTEGER;
}

function setStatus(text, isError){
  $('status').textContent = text || '';
  $('status').classList.toggle('error', !!isError);
}

// ============================================================================
// NETWORK
// ============================================================================
// The POST body is a plain string on purpose: a string is sent as text/plain,
// which needs no CORS preflight. Do not add a Content-Type header — Apps
// Script cannot answer the preflight that header would trigger.
async function call(venue, body){
  const res = await fetch(venue.url, body === undefined
    ? { cache: 'no-store' }
    : { method: 'POST', body: JSON.stringify(body) });
  if(!res.ok) throw new Error(`The attendance sheet replied ${res.status}.`);
  const data = await res.json();
  if(!data.ok) throw new Error(data.error || 'The attendance sheet refused the change.');
  return data;
}

function findPlayer(teamCode, slot){
  const team = pairs.find((t) => t.teamCode === teamCode);
  return team && team.players.find((p) => p.slot === slot);
}

async function load(quiet){
  const venue = VENUES[venueIndex];
  const seq = ++loadSeq;
  if(!isConnected(venue)){
    pairs = [];
    render();
    setStatus('');
    return;
  }
  if(!quiet) setStatus('Loading…');
  try{
    const data = await call(venue);
    if(seq !== loadSeq) return;
    // A player this phone is still saving keeps its local state: the sheet
    // may not have the change yet.
    data.pairs.forEach((t) => t.players.forEach((p) => {
      const mine = saving.has(`${t.teamCode}|${p.slot}`) && findPlayer(t.teamCode, p.slot);
      if(mine){ p.present = mine.present; p.timeIn = mine.timeIn; }
    }));
    pairs = data.pairs;
    setStatus('');
    render();
  }catch(err){
    if(seq !== loadSeq) return;
    setStatus(`Could not load ${venue.name}: ${err.message}`, true);
  }
}

async function toggle(teamCode, slot, want){
  const venue = VENUES[venueIndex];
  const key = `${teamCode}|${slot}`;
  const player = findPlayer(teamCode, slot);
  if(!player) return;
  const before = { present: player.present, timeIn: player.timeIn };
  player.present = want;
  player.timeIn = want ? player.timeIn : '';
  saving.add(key);
  render();
  try{
    const res = await call(venue, { teamCode, slot, present: want });
    const p = findPlayer(teamCode, slot);
    if(p){ p.present = res.present; p.timeIn = res.timeIn; }
    setStatus('');
  }catch(err){
    const p = findPlayer(teamCode, slot);
    if(p){ p.present = before.present; p.timeIn = before.timeIn; }
    setStatus(`Not saved (${player.name}): ${err.message}`, true);
  }finally{
    saving.delete(key);
    render();
  }
}

// ============================================================================
// RENDER
// ============================================================================
function el(tag, cls, text){
  const e = document.createElement(tag);
  if(cls) e.className = cls;
  if(text !== undefined) e.textContent = text;
  return e;
}

function renderVenues(){
  const box = $('venues');
  box.textContent = '';
  VENUES.forEach((v, i) => {
    const b = el('button', '', v.name);
    b.type = 'button';
    b.setAttribute('aria-pressed', String(i === venueIndex));
    b.addEventListener('click', () => {
      if(i === venueIndex) return;
      venueIndex = i;
      saveVenue(i);
      pairs = [];
      render();
      load();
    });
    box.append(b);
  });
}

function playerRow(team, p){
  const key = `${team.teamCode}|${p.slot}`;
  const row = el('label', 'player' + (p.present ? ' in' : ''));
  const who = el('span', 'who');
  who.append(el('span', 'name', p.name));
  const time = saving.has(key) ? 'Saving…' : (p.present && p.timeIn ? `In ${p.timeIn.slice(-5)}` : '');
  if(time) who.append(el('span', 'time', time));
  const sw = el('input', 'switch');
  sw.type = 'checkbox';
  sw.setAttribute('role', 'switch');
  sw.checked = p.present;
  sw.disabled = saving.has(key);
  sw.setAttribute('aria-label', `${p.name} present`);
  sw.addEventListener('change', () => toggle(team.teamCode, p.slot, sw.checked));
  row.append(who, sw);
  return row;
}

function render(){
  renderVenues();
  const venue = VENUES[venueIndex];
  const list = $('list');
  list.textContent = '';

  let total = 0, present = 0;
  pairs.forEach((t) => t.players.forEach((p) => { total++; if(p.present) present++; }));
  $('count').textContent = total ? `${present} / ${total} in` : '';

  if(!isConnected(venue)){
    list.append(el('p', 'empty', `${venue.name} is not connected yet.`));
    return;
  }
  const q = $('search').value.trim().toLowerCase();
  const shown = pairs.filter((t) => !q || t.teamCode.toLowerCase().includes(q) ||
    t.players.some((p) => p.name.toLowerCase().includes(q)));
  if(!shown.length){
    if(pairs.length) list.append(el('p', 'empty', 'No one matches that search.'));
    return;
  }

  const byCat = new Map();
  shown.forEach((t) => {
    if(!byCat.has(t.category)) byCat.set(t.category, []);
    byCat.get(t.category).push(t);
  });
  [...byCat.keys()]
    .sort((a, b) => categoryRank(a) - categoryRank(b) || a.localeCompare(b))
    .forEach((cat) => {
      const teams = byCat.get(cat);
      const players = teams.flatMap((t) => t.players);
      const section = el('section', 'cat');
      const h = el('h2', '', categoryName(cat));
      h.append(el('span', '', `${players.filter((p) => p.present).length} / ${players.length}`));
      section.append(h);
      teams.forEach((t) => {
        const ready = t.players.every((p) => p.present);
        const card = el('div', 'pair' + (ready ? ' ready' : ''));
        const head = el('div', 'pair-head', t.teamCode);
        if(ready) head.append(el('span', 'badge', 'Ready'));
        card.append(head);
        t.players.forEach((p) => card.append(playerRow(t, p)));
        section.append(card);
      });
      list.append(section);
    });
}

// ============================================================================
// START
// ============================================================================
$('search').addEventListener('input', render);
$('refresh').addEventListener('click', () => load());
document.addEventListener('visibilitychange', () => { if(!document.hidden) load(true); });
setInterval(() => { if(!document.hidden) load(true); }, POLL_MS);
render();
load();
</script>
</body>
</html>
```

**Checks:**

```bash
cd "D:/Coding Projects/SAGE/sage-match-control.github.io/events/pickle-for-sight-2026"
python -c "import re;s=open('attendance.html',encoding='utf-8').read();open('attendance-inline-beta.js','w',encoding='utf-8').write(re.search(r'<script>([\s\S]*)</script>',s).group(1))"
node --check attendance-inline-beta.js && echo "PAGE JS OK"; rm attendance-inline-beta.js
grep -c "PASTE_PCPH_MAIN_EXEC_URL\|PASTE_PCPH_ANNEX_EXEC_URL" attendance.html
```

→ `PAGE JS OK`, then `2`.

Confirm the category names still match the event page:

```bash
grep -n "LI: { name: 'LI', full: 'Low Intermediate' }\|XD: \"Mixed Doubles\"" index.html
```

→ two lines. If not, stop and ask (§0).

### 5.4 Browser check of the page

The page has no test file (the site repo has none by convention). This check
serves it with `fetch` stubbed to behave like §3, fed the real roster. Both
files it uses end in `-beta`, which the site repo's `**/*beta*` rule keeps
out of git.

**Step 1.** Save this as
`D:\Coding Projects\SAGE\sage-match-control.github.io\make-attendance-harness-beta.mjs`:

```js
// Run from D:\Coding Projects\SAGE. Writes attendance-beta.html beside the
// real page: the same page with fake /exec URLs and fetch() stubbed to act
// like attendance.gs, fed the real roster from event-data. `**/*beta*` is
// gitignored in the site repo, so the file cannot be committed by accident.
import fs from "node:fs";

const SNAPSHOT = "event-data/pickle-for-sight-2026/data/pickle-for-sight-day1.json";
const PAGE = "sage-match-control.github.io/events/pickle-for-sight-2026/attendance.html";
const OUT = "sage-match-control.github.io/events/pickle-for-sight-2026/attendance-beta.html";
const MAIN = "https://script.google.com/macros/s/FAKE_MAIN/exec";
const ANNEX = "https://script.google.com/macros/s/FAKE_ANNEX/exec";
const unconnected = process.argv.includes("--unconnected");

function parseCsv(text) {
    const rows = []; let row = [], cell = "", q = false;
    for (let i = 0; i < text.length; i++) {
        const c = text[i];
        if (q) { if (c === '"') { if (text[i + 1] === '"') { cell += '"'; i++; } else q = false; } else cell += c; }
        else if (c === '"') q = true;
        else if (c === ",") { row.push(cell); cell = ""; }
        else if (c === "\n") { row.push(cell); rows.push(row); row = []; cell = ""; }
        else if (c !== "\r") cell += c;
    }
    if (cell || row.length) { row.push(cell); rows.push(row); }
    return rows;
}

// The same selection rule as attendanceTeams_ in attendance.gs.
function roster(csv) {
    const [head, ...rows] = parseCsv(csv);
    const col = (n) => head.indexOf(n);
    return rows.flatMap((r) => {
        const code = (r[col("teamCode")] || "").trim();
        if (!/^[A-Z0-9]+_\d+$/.test(code)) return [];
        const players = [r[col("player1")], r[col("player2")]]
            .map((v, k) => ({ slot: k + 1, name: (v || "").trim(), present: false, timeIn: "" }))
            .filter((p) => p.name && p.name !== code);
        return players.length ? [{ teamCode: code, category: code.split("_")[0], players }] : [];
    });
}

const snap = JSON.parse(fs.readFileSync(SNAPSHOT, "utf8"));
const data = {};
for (const f of snap.facilities) data[f.name] = roster(f.standingsCsv);
const failCode = data["PCPH Main"][1].teamCode;

const stub = `<script>
(() => {
  const DATA = ${JSON.stringify(data)};
  const URLS = { ${JSON.stringify(MAIN)}: "PCPH Main", ${JSON.stringify(ANNEX)}: "PCPH Annex" };
  const FAIL = ${JSON.stringify(failCode)};
  window.__calls = [];
  let minute = 0;
  const reply = (obj) => new Promise((ok) => setTimeout(() => ok({ ok: true, status: 200,
    json: async () => JSON.parse(JSON.stringify(obj)) }), 350));
  window.fetch = (url, init = {}) => {
    const venue = URLS[url];
    window.__calls.push({ url, method: init.method || "GET", headers: init.headers || null,
                          bodyType: typeof init.body, body: init.body || null });
    if (!venue) return Promise.reject(new TypeError("fetch to an unexpected URL: " + url));
    if (!init.method) return reply({ ok: true, pairs: DATA[venue] });
    const b = JSON.parse(init.body);
    if (b.teamCode === FAIL && b.slot === 2) return reply({ ok: false, error: "Simulated failure" });
    const team = DATA[venue].find((t) => t.teamCode === b.teamCode);
    const p = team && team.players.find((x) => x.slot === b.slot);
    if (!p) return reply({ ok: false, error: "No player" });
    const already = p.present;
    p.present = b.present;
    p.timeIn = b.present ? (already ? p.timeIn : "2026-09-27 08:" + String(10 + minute++).padStart(2, "0")) : "";
    return reply({ ok: true, teamCode: b.teamCode, slot: b.slot, name: p.name, present: p.present, timeIn: p.timeIn });
  };
})();
</script>
`;

let page = fs.readFileSync(PAGE, "utf8");
for (const [placeholder, url] of [["'PASTE_PCPH_MAIN_EXEC_URL'", MAIN], ["'PASTE_PCPH_ANNEX_EXEC_URL'", ANNEX]]) {
    if (!page.includes(placeholder)) throw new Error(`${placeholder} not found in ${PAGE}`);
    if (unconnected && url === ANNEX) continue;
    page = page.replace(placeholder, `'${url}'`);
}
fs.writeFileSync(OUT, page.replace("<script>", stub + "<script>"));
console.log(`wrote ${OUT}${unconnected ? " (Annex left unconnected)" : ""}`);
for (const [k, v] of Object.entries(data)) {
    console.log(`  ${k}: ${v.length} pairs, ${v.reduce((n, t) => n + t.players.length, 0)} players`);
}
console.log(`  a save for ${failCode} player 2 is made to fail`);
```

**Step 2.** From `D:\Coding Projects\SAGE`, run:

```bash
node sage-match-control.github.io/make-attendance-harness-beta.mjs
```

→ `wrote …/attendance-beta.html`, then `PCPH Main: 51 pairs, 102 players`
and `PCPH Annex: 59 pairs, 118 players`.

**Step 3.** Start the `static-site` preview (`D:\Coding Projects\SAGE\.claude\launch.json`,
port 8123) and open
`http://localhost:8123/events/pickle-for-sight-2026/attendance-beta.html`.
Confirm each line:

| Do | Expect |
| --- | --- |
| Load | `0 / 102 in`; categories Novice Men's, Novice Women's, Novice Mixed, Low Intermediate Women's; no console errors |
| Flip the first switch | *Saving…* and locked at once; then *In 08:10* and `1 / 102 in` |
| Flip its partner | the card shows **Ready** and a green left edge |
| Flip `NMD_2`'s second player | it snaps back off; status reads `Not saved (…): Simulated failure` |
| In the console, `__calls.filter(c => c.method === 'POST')` | every POST has `headers: null` and `bodyType: "string"` |
| Search `pasion`, then `zzzz`, then clear | one pair; "No one matches that search."; all 51 back |
| Switch to PCPH Annex, reload the page | `0 / 118 in`, five categories, Annex still selected |
| Viewport 375 px wide | no sideways scroll |

**Step 4.** Run step 2 again with ` --unconnected` on the end, reload, and
switch to PCPH Annex → "PCPH Annex is not connected yet.", no count, and no
request to a placeholder URL in `__calls`.

**Step 5.** Stop the preview, then from `D:\Coding Projects\SAGE`:

```bash
rm sage-match-control.github.io/make-attendance-harness-beta.mjs sage-match-control.github.io/events/pickle-for-sight-2026/attendance-beta.html
```

### 5.5 Docs

**a. New file `sage-docs/docs/features/event-attendance.md`:**

```markdown
# Event attendance

A check-in page for the staff at an event's desk. Open it on any phone, find a
player, and flip their switch: they're marked in, with the time they arrived.
No app, no sign-in.

Pickle for Sight is the first event with one, at
`/events/pickle-for-sight-2026/attendance`. It isn't linked from the public
event page — share the link with desk staff only.

## Using it

- **Pick your venue** at the top. The page remembers it on that phone.
- **Find the player** by scrolling to their category, or type part of a name
  or a team code into the search box.
- **Flip their switch.** It shows *Saving…* for a moment, then *In 08:14*.
  Each player has their own switch, so a pair with a partner still on the way
  shows exactly who is missing. Once both are in, the pair is marked
  **Ready**.
- **Marked the wrong person?** Flip it back. That clears the time.
- The count at the top shows how many players are in at that venue.

Several phones can mark at once. Each picks up the others' marks within about
30 seconds, straight away when you come back to the page, or when you press
**Refresh**.

## When something goes wrong

- **A switch flips back with a message at the top.** That mark was not saved
  — usually a dropped connection. Try again.
- **"… is not connected yet."** That venue's workbook hasn't been set up for
  attendance. Tell whoever runs the event's workbooks.
- **A player is missing or misspelled.** The list comes from the scoring
  workbook's roster. Fix it there; the page picks it up on the next refresh.

## What it records

Each mark lands in an **ATTENDANCE** tab of that venue's scoring workbook:
team code, which player, their name, whether they're in, and the time. None
of it appears on the public event page.

Anyone who has the link can change marks, so share it with desk staff only.

---
**Technical:** [event attendance](../technical/event-attendance.md)
```

**b. New file `sage-docs/docs/technical/event-attendance.md`:**

```markdown
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
```

**c. `sage-docs/docs/features/README.md`** — find:

```markdown
- **[Standard Tournament Generator](standard-tournament-generator.md)** — the
  same for a standard tournament, one venue-day workbook at a time.
```

Replace with:

```markdown
- **[Standard Tournament Generator](standard-tournament-generator.md)** — the
  same for a standard tournament, one venue-day workbook at a time.
- **[Event attendance](event-attendance.md)** — a staff-only check-in page
  that marks each player in, with their arrival time, from any phone.
```

**d. `sage-docs/docs/technical/README.md`** — find:

```markdown
All three live in `sage-tools-api/scripts/` for versioning, run inside a
```

Replace with:

```markdown
All four live in `sage-tools-api/scripts/` for versioning, run inside a
```

Then find:

```markdown
- **[Standard Tournament Generator](standard-tournament-generator.md)** —
  builds one facility-day's standard-tournament workbook from a calculator
  CSV.
```

Replace with:

```markdown
- **[Standard Tournament Generator](standard-tournament-generator.md)** —
  builds one facility-day's standard-tournament workbook from a calculator
  CSV.
- **[Event attendance](event-attendance.md)** — `attendance.gs`, the web app
  behind an event's staff check-in page, and the page itself.
```

**e. `sage-docs/mkdocs.yml`** — find:

```yaml
      - Standard Tournament Generator: features/standard-tournament-generator.md
```

Replace with:

```yaml
      - Standard Tournament Generator: features/standard-tournament-generator.md
      - Event attendance: features/event-attendance.md
```

Then find:

```yaml
      - Standard Tournament Generator: technical/standard-tournament-generator.md
```

Replace with:

```yaml
      - Standard Tournament Generator: technical/standard-tournament-generator.md
      - Event attendance: technical/event-attendance.md
```

**Check:** deferred to §5.8 — the technical page links to this spec at its
`implemented/` path, which exists only after §5.8.

### 5.6 `D:\Coding Projects\SAGE\CLAUDE.md`

**a.** Find:

```markdown
No TypeScript, no build step, no test framework, no linter. The exceptions
are `scripts/verify-sheet-generator.mjs` and
`scripts/verify-standard-generator.mjs`, which run the two Apps Script
generators against a mocked Sheets API — they cover those `.gs` files only,
nothing in `src/`, and are run by hand (see the `scripts/*.gs` notes below).
```

Replace with:

```markdown
No TypeScript, no build step, no test framework, no linter. The exceptions
are `scripts/verify-sheet-generator.mjs`,
`scripts/verify-standard-generator.mjs` and `scripts/verify-attendance.mjs`,
which run their Apps Script files against a mocked Sheets API — they cover
those `.gs` files only, nothing in `src/`, and are run by hand (see the
`scripts/*.gs` notes below).
```

**b.** Find:

```markdown
    workbook it was copied from (`secretMenuItem_`).
- `scripts/verify-standard-generator.mjs` — the same kind of harness for
```

Replace with:

````markdown
    workbook it was copied from (`secretMenuItem_`).
  - `attendance.gs` — the staff attendance web app, pasted into an event's
    **live** workbooks only (never a master), beside `sheets-sync.gs` and the
    generator, and deployed as a web app (Execute as: Me, access: Anyone).
    `doGet` lists the pairs in `STANDINGSCSV`; `doPost` marks one player in
    an `ATTENDANCE` tab it creates, keyed by team code and slot. It writes no
    other tab, and nothing it writes is published — the sync reads only the
    `CSV` and `STANDINGSCSV` tabs. Implements
    `sage-docs/docs/specs/.../event-attendance-spec.md`.
- `scripts/verify-attendance.mjs` — runs `attendance.gs`'s real `doGet` and
  `doPost` against the same mock, and fails if any top-level name in it is
  also declared by `sheets-sync.gs` or a generator: they share one script
  project, where a repeated name silently replaces the other file's.

  ```bash
  node scripts/verify-attendance.mjs
  ```

- `scripts/verify-standard-generator.mjs` — the same kind of harness for
````

**c.** Find:

```markdown
- `field-guide.html` (repo root) — client-facing, plain-language feature
  tour for players/spectators.
```

Replace with:

```markdown
- `field-guide.html` (repo root) — client-facing, plain-language feature
  tour for players/spectators.
- `events/pickle-for-sight-2026/attendance.html` — staff-only check-in page,
  deliberately linked from nowhere. It talks to one `attendance.gs` web app
  per venue workbook; their `/exec` URLs are its `VENUES` constant.
```

### 5.7 Final checks and commits

```bash
cd "D:/Coding Projects/SAGE/sage-tools-api"
node -e "new Function(require('fs').readFileSync('scripts/attendance.gs','utf8'))" && echo "GS OK"
node scripts/verify-attendance.mjs | tail -1
node scripts/verify-standard-generator.mjs | tail -1
node scripts/verify-sheet-generator.mjs | tail -1
```

→ `GS OK`, then `ALL CHECKS PASSED` three times. The last two prove the
shared mock is untouched.

Then `git status --porcelain` in each repo must list exactly:

| Repo | Expected |
| --- | --- |
| `sage-tools-api` | `?? scripts/attendance.gs`, `?? scripts/verify-attendance.mjs` |
| `sage-match-control.github.io` | `?? events/pickle-for-sight-2026/attendance.html` |
| `sage-docs` | after §5.8: the two new pages, the renamed spec, both section READMEs, `docs/specs/README.md`, both spec-folder READMEs, `mkdocs.yml` |

Anything else — in particular an `attendance-beta.html` — remove it before
committing. Commit `sage-tools-api` and `sage-match-control.github.io` now, on
`main`; `sage-docs` waits for §5.8. Do not push.

### 5.8 File this spec as implemented

Follow `sage-docs/docs/specs/README.md` → *Moving a spec between folders*:

1. ```bash
   cd "D:/Coding Projects/SAGE/sage-docs"
   git mv docs/specs/not-started/event-attendance-spec.md docs/specs/implemented/event-attendance-spec.md
   ```

2. In the moved spec, replace the whole status block — from
   `> **Status: not built.**` down to its last `>` line — with:

   ```markdown
   > **Status: built.** `attendance.gs`, `verify-attendance.mjs` and the
   > Pickle for Sight page exist, and every check in §5 passes. Reaching a
   > workbook is §6, a person's job: until it is done the page shows both
   > venues as "not connected".
   ```

   And in §10, replace `*(None — nothing here is built. Record departures
   when it is.)*` with `*(None.)*` — or, if you departed from anything in §5,
   with a short entry per departure saying what and why.

3. `docs/specs/README.md` — find:

   ```markdown
   - **[Pickle & Friends × 1Bataan United Picklers dual meet](implemented/pnf-x-bup-dual-meet-spec.md)**
     — the event that drove the dual-meet template's first real run.
   ```

   Replace with:

   ```markdown
   - **[Pickle & Friends × 1Bataan United Picklers dual meet](implemented/pnf-x-bup-dual-meet-spec.md)**
     — the event that drove the dual-meet template's first real run.
   - **[Event attendance](implemented/event-attendance-spec.md)** — a
     staff check-in page writing each player's arrival into the venue
     workbook through an Apps Script web app; Pickle for Sight first.
   ```

   Then delete this entry from the `## Not started` list (both lines):

   ```markdown
   - **[Event attendance](not-started/event-attendance-spec.md)** — a staff
     check-in page that marks each player in, via an Apps Script web app.
   ```

4. `docs/specs/not-started/README.md` — delete this table row:

   ```markdown
   | [Event attendance](event-attendance-spec.md) | A staff check-in page that marks each player in, with their arrival time, through an Apps Script web app writing to each venue workbook | Nothing |
   ```

5. `docs/specs/implemented/README.md` — find:

   ```markdown
   | [Facility progress](facility-progress-spec.md) | Live Matches' per-facility progress cards and Mission Control's progress line |
   ```

   Replace with:

   ```markdown
   | [Facility progress](facility-progress-spec.md) | Live Matches' per-facility progress cards and Mission Control's progress line |
   | [Event attendance](event-attendance-spec.md) | `attendance.gs` and `events/pickle-for-sight-2026/attendance.html` |
   ```

6. `mkdocs.yml` — find:

   ```yaml
             - Facility progress: specs/implemented/facility-progress-spec.md
   ```

   Replace with:

   ```yaml
             - Facility progress: specs/implemented/facility-progress-spec.md
             - Event attendance: specs/implemented/event-attendance-spec.md
   ```

   Then delete this line from the `Not started:` group:

   ```yaml
             - Event attendance: specs/not-started/event-attendance-spec.md
   ```

**Check:**

```bash
cd "D:/Coding Projects/SAGE/sage-docs"
grep -rn "event-attendance-spec.md" docs/ mkdocs.yml --exclude=event-attendance-spec.md | grep -c "not-started"
python -m mkdocs build --strict --site-dir "$TEMP/sage-docs-build" 2>&1 | grep -iE "warning|error" | grep -v "Material for MkDocs"; rm -rf "$TEMP/sage-docs-build"
```

→ `0`, then no output from the build filter. Then commit `sage-docs`, on
`main`. Do not push.

---

## 6. Operator — deploying it (a person, not the implementer)

Signed in as **sagematchcontrol@gmail.com**, which owns both workbooks. Do
§6.1 first; it touches nothing live.

### 6.1 Dry run on a copy

1. Open the live Main workbook,
   `https://docs.google.com/spreadsheets/d/1rNIlK2Zz3zTaLILlpq4rCGIQbtr0ECpG4A3OdP4v25A/edit`,
   then **File → Make a copy**, named `ATTENDANCE TEST - MAIN`. A copy does
   not inherit live sync, so it cannot publish anything.
2. In the copy: **Extensions → Apps Script → + (Add a file) → Script**, name
   it `attendance`, paste the whole of `sage-tools-api/scripts/attendance.gs`,
   **Save**.
3. **Deploy → New deployment →** gear → **Web app**. Description
   `attendance test`; **Execute as: Me**; **Who has access: Anyone**.
   **Deploy → Authorize access**, pick the account; on "Google hasn't verified
   this app" choose **Advanced → Go to … (unsafe) → Allow**. Copy the
   **Web app URL** (ends in `/exec`).
4. Open that URL in a browser → JSON starting `{"pairs":[` with Main's names
   and `"ok":true` at the end.
5. In Git Bash:

   ```bash
   curl -sL -d '{"teamCode":"NMD_1","slot":1,"present":true}' "PASTE_THE_TEST_URL"
   ```

   → `{"teamCode":"NMD_1","slot":1,…,"present":true,"timeIn":"2026-…","ok":true}`,
   and the copy now has an **ATTENDANCE** tab with that row.
6. **Deploy → Manage deployments → Archive**, then trash the copy.

### 6.2 The live workbooks

Repeat §6.1 steps 2–4 in each of these, and nowhere else — not the `[REF]`
copies, not the `ALL` workbook, not a master:

| Venue | Workbook |
| --- | --- |
| PCPH Main | `https://docs.google.com/spreadsheets/d/1rNIlK2Zz3zTaLILlpq4rCGIQbtr0ECpG4A3OdP4v25A/edit` |
| PCPH Annex | `https://docs.google.com/spreadsheets/d/1ks84WK7vo5FAenpPbZocm9PLPpLoS6_SRPivVJqmS-0/edit` |

Use description `attendance`. Keep the two `/exec` URLs. Do **not** POST to
the live ones by hand — §6.3 does that through the page.

### 6.3 Connect the page and push

1. In `events/pickle-for-sight-2026/attendance.html`, replace
   `PASTE_PCPH_MAIN_EXEC_URL` and `PASTE_PCPH_ANNEX_EXEC_URL` with the two
   URLs (keep the quotes). Commit `Attendance: connect the venue web apps`.
2. Push all three repos.
3. Once Pages has deployed, open
   `https://sage-match-control.github.io/events/pickle-for-sight-2026/attendance`
   on a phone, flip one player on at each venue, check the row in that
   workbook's **ATTENDANCE** tab, then flip them off again.

### 6.4 On the day, and after

- Send the link to desk staff only.
- If a venue shows "not connected", its URL was not pasted; if its switches
  all snap back, open its `/exec` URL — the error is in the JSON.
- After the event: **Deploy → Manage deployments → Archive** in both
  workbooks. That closes the public endpoint; the marks stay in the tab.

## 7. Acceptance

- `verify-attendance.mjs` passes, and the other two harnesses still pass.
- With the URLs in place, the page lists 102 players at PCPH Main and 118 at
  PCPH Annex, grouped into their categories, with no playoff seats.
- Flipping a player on writes one `ATTENDANCE` row with a time; flipping on
  again keeps that time; flipping off clears it; nothing else in the workbook
  changes.
- Two phones on the same venue see each other's marks within 30 seconds.
- A pair shows **Ready** once both players are in.
- No attendance data appears in `event-data`.

## 8. Decisions

**A separate tab, not a column in `STANDINGSCSV`.** Three reasons, all
checked against the code: the sync reads the whole `STANDINGSCSV` tab, so a
column there would be published to the public `event-data` repo; `A2` is a
spill, so a hand-kept column beside it drifts when the spill shifts; and
nearly half the rows are playoff seats that repeat real pairs.

**Keyed by team code and slot, guarded by name.** Row positions are not
stable; codes are. The name check stops a re-draw from handing one player's
mark to another.

**Per player, with the time.** A pair where one partner is late is the case
that matters at a desk. The time costs nothing extra to record.

**An Apps Script web app, not a Cloud Run endpoint.** The existing API reads
Sheets with an API key, which cannot write. Writing from Cloud Run would need
a service account, a new secret, sharing each workbook with it, and a deploy.
A bound web app needs none of that and touches only the workbooks it is
pasted into.

**No sign-in.** Staff are the organizer's, with no SAGE account. The cost is
that anyone who reads the page source can change marks — the repo is public.
The script limits that to the `ATTENDANCE` tab, and archiving the deployments
after the event closes it.

**Polling, not push.** Several phones mark the same venue. A 30-second poll
while visible, plus a reload on focus, keeps them close enough without a
server; a save in flight is protected from being overwritten by a reload.

**Marking again keeps the first time.** A double tap or a second phone must
not move someone's arrival time.

**Reads via CSV export, not the web app (§11).** Every read counts against
Apps Script's per-account simultaneous-execution quota, since the web app
runs **Execute as: Me** — all staff phones' polling shares one account's
budget with every write. The workbook's own published CSV export isn't
subject to that quota at all, and (measured against the live workbook) it
reflects a write in under a second, not the multi-second-to-minute lag
Google's caching could have produced. The cost is that reads no longer
self-heal a re-draw on every poll the way `attendanceRoster_` did — that
now needs `attendanceResync` run again by hand (§11).

## 9. Out of scope

- Other events. The page and constants are Pickle for Sight's; a second
  event copies the page and deploys the script into its own workbooks.
- Adding walk-ins from the page. Names come from the scoring workbook only.
- A PIN or sign-in.
- Showing attendance anywhere else — Control Center, the event page, exports.
- Multi-day events. One day, one set of marks per workbook.

## 10. Divergences

*(None.)*

## 11. Revision — CSV reads and manual resync

Applied after §6 was done and both venues were already live. Changes two
things in `attendance.gs`, and the page's read path — §5's code blocks
above are the original build's record and were **not** rewritten to match;
the real current content of both files is the source of truth.

**What changed:**

- `attendance.gs` gained one function, `attendanceResync` — not reachable
  through `doGet`/`doPost`, run by hand from the Apps Script editor's
  function dropdown. It pre-fills `ATTENDANCE` with every player at
  `present: FALSE`, and for any row whose stored name no longer matches the
  current roster (a re-draw), resets that one row to the new name and
  `present: FALSE`. A row whose name still matches is left completely
  alone, so running it twice — or a hundred times — is always safe.
- `attendance.html`'s reads (`load()`, on a timer and on refresh) no longer
  call the web app's `doGet` at all. They fetch the venue workbook's own
  published CSV export of `ATTENDANCE` directly —
  `https://docs.google.com/spreadsheets/d/<sheetId>/gviz/tq?tqx=out:csv&sheet=ATTENDANCE`
  — parse it client-side, and group by `teamCode` prefix for category, the
  same way `attendanceTeams_` always did server-side. `doPost` (marking) is
  unchanged; `VENUES` gained a `sheetId` per venue alongside the existing
  `url`.

**Why:** the web app runs **Execute as: Me**, so every phone's poll and
every mark share one Google account's simultaneous-execution quota. A
CSV export isn't part of that quota at all, which matters once several
staff phones are polling a venue at once. This was only viable because the
workbooks are already shared "Anyone with the link — Viewer" (so the CSV
export needs no sign-in) and because `gviz` accepts a tab **name**
(`sheet=ATTENDANCE`) instead of a numeric gid, so no gid needs discovering
or hardcoding per venue.

**What it costs:** `attendanceRoster_`'s per-read re-draw check — comparing
a stored mark's name against the live roster on every single `doGet` — no
longer runs on the page's read path at all. A swap (a team code and slot
reassigned to a different player in `STANDINGSCSV`) now leaves the old name
showing on the page, present or not, until someone runs `attendanceResync`
again. The new player doesn't appear until then either — they're simply not
a row in `ATTENDANCE` under a name anyone would search for. `doPost` still
resolves the current roster on every mark, so a stray tap on the stale row
would silently correct it, but nothing prompts staff to do that.

**Operationally, this means:** if a swap happens after `ATTENDANCE` has
been pre-filled, whoever manages the workbook re-runs `attendanceResync`
(Extensions → Apps Script → pick `attendanceResync` from the function
dropdown → Run). The page reflects it within about a second of the run
finishing, per the same measurement above. `attendanceResync` does not
prune a row for a pair that withdraws entirely — that row is left in
`ATTENDANCE` as an inactive leftover rather than removed, since nothing in
this revision reconciles the other direction (a code disappearing from
`STANDINGSCSV`).

Verified: `verify-attendance.mjs`'s resync section (pre-fill, a no-op
re-run, a re-run after a mark, and a re-run after a swap — 10 checks, none
touching `STANDINGSCSV`) against the mock; the CSV read path against the
live PCPH Main workbook — cross-origin fetch from the real GitHub Pages
origin, a mark round-tripped through a fresh page load, and the write
reflected in the CSV export in 407ms and 702ms across two runs.
