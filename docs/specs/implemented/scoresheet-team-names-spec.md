# Spec — team names on scoresheets

> **Status: implemented.** §5 and §6 are built and live: `sage-tools-api` 3.6.0 and 3.6.1, and the
> site's `94fa9c7`, all 6 November 2026. The owner used the named sheets at Ladies' Dink on
> 7 November 2026, which stands for §7's checks, C5 included; they were not run one by one as §7 lists
> them. §8's docs are written (docs 2.64.0). Written 2026-11-06 against `sage-tools-api` 3.5.0 (`f885c64`) and the
> site at `9c9b584`. All the code in §5 and §6 was run on a throwaway copy of both repos on that date
> (§10 says what that did and did not cover). All owner's decisions are made (§12).

Print each team's name on a team event's scoresheets, next to the pair code that is there today.
Every other event's sheets stay exactly as they are.

**This spec is written to be built with no other context.** Do §4 to §8 in order. The code is meant to
be used as written. If an anchor in the code has moved, or a check gives a different result, follow §9.

---

## 1. Why

A team event's sheet says only which pair is playing. Before lineups are in, a sheet reads:

```
Team 1: A1_1 -
```

Whoever holds that slip has to look up who `A1` is. The team name is what people at the venue know a
team by, and every published snapshot already has it: `standingsCsv` has `teamCode` and `teamName`
beside the `matchesCsv` the generator prints from.

The code stays, because its `_1` is the only thing on the sheet that says which of the team's pairs is
playing.

---

## 2. What it does

| Situation | Team 1's line on the sheet |
| --- | --- |
| Team event, group stage | **SPICE GIRLS** `B2_1 -` |
| Team event, lineup known | **SPICE GIRLS** `B2_1 - Ann / Cy` |
| Team event, playoff team seated | **SPICE GIRLS** `QF-3_1 -` |
| Team event, playoff team not seated | `QF-3_1 -`, and the team is written in by hand |
| Standard or dual-meet event | Unchanged |
| Uploaded CSV without the new columns | Unchanged |
| Uploaded CSV with the new columns | Prints them, whatever the event |

**Not in scope:** pair labels (*MD*, *Pair 1*), replacing the code with the name, names for standard or
dual-meet events, team colours or marks, a new route, any change to the workbooks or `event-data`.

---

## 3. The design in brief

Everything below is decided. It is here so the code in §5 and §6 makes sense.

**The site joins the names; the API prints them.** The generator page adds two columns, `teamName1` and
`teamName2`, to the matches CSV it posts. The join is in the browser because the rule for who a side
belongs to is already in `lib/v1/domain/teams.js`. Doing it on the server would make a second copy to
keep in step.

**No columns, no change.** The API prints names only when the columns exist, so other events, old cached
pages and the frozen routes behave as they do today.

**Layout, measured** at the service's real settings (Letter, scale 0.8), with a 24-character name and a
48-character stress test. The longest real name so far is 26 characters.

| Layout (`type`) | A separate name line | Chosen layout | Result |
| --- | --- | --- | --- |
| Standard (`scoresheet`) | Spills to 2 pages | Name inside the existing underline, before the code | 1 page |
| Best of 3 (`scoresheet-best-of-3`) | Spills to 2 pages | Name inside the existing underline, before the code | 1 page |
| No Referee (`scoresheet-no-ref`) | Fits | Its own underlined line above the code | 1 page; a 48-character name wraps and still fits |
| No Referee, Wide (`scoresheet-no-ref-wide`) | Spills to 2 pages | Its own line, score boxes 240 → 200 px, on named sheets only | 1 page; only just at 48 characters |

**An unseated playoff team** has an empty name. The two No Referee layouts still print the empty line to
write on, which is why they switch on `hasTeamNames` (the column exists) rather than on the name being
set. Standard and Best of 3 print nothing extra, and the team is written on the existing line.

**Spare blank sheets** get the same empty columns when the CSV has them, so they match the rest of the set.

**Commas in names.** The API's CSV parser runs with quoted fields off, so `"Dink, Inc"` would shift every
later column. Turning quoted fields on for everything would make a CSV with a stray `"` throw, on frozen
routes. So they are turned on only when the header has a team-name column. Such a CSV with a mismatched
quote fails with a clear message: a `400` on the PDF response, or a `{"phase":"error"}` line on the NDJSON
stream the page uses. The page's own CSV is always balanced, so only a hand-edited upload can reach that.

---

## 4. Before starting

The workspace is `D:\Personal\SAGE`. It contains these separate git repos, each with its own `main`:

| Folder | What it is | Pushing `main` |
| --- | --- | --- |
| `sage-tools-api/` | Node/Express API on Cloud Run (ESM, no build step) | **Deploys automatically** |
| `sage-match-control.github.io/` | The static site (no build step) | Publishes on GitHub Pages |
| `sage-docs/` | The documentation (this spec) | Publishes the docs |
| The workspace folder itself | Tracks `CLAUDE.md` | Nothing |

Read `D:\Personal\SAGE\CLAUDE.md` for the general rules. The ones that matter here are repeated in the steps.

**Check all of these first. If any fails, stop and tell the owner.**

1. **No event within 3 days.** Look at every day's `date` in `event-data/config/events.json`. Nothing is
   merged to the site within 3 days of an event, and the API and site must ship together.
2. **The repos are clean and current.** Run `git status` and `git pull` in `sage-tools-api`, the site and
   `sage-docs`. Unrelated untracked files can stay, but stage files by name so they aren't committed.
   Never commit a `*beta*` file.
3. **Baselines pass.** In `sage-tools-api`, `npm test`. In the site, `cd _tests`, `npm install`, `npm test`.
   A failure here isn't caused by this work, so report it instead of fixing it.
4. **The API version.** This spec assumes `package.json` says 3.5.0. If it is higher, use the next minor
   version instead of 3.6.0 throughout.

**How to work:**

- Tests first: add the tests, see them fail as described, then write the code. Never weaken or delete a
  test to get a change through.
- The working copies on this machine use CRLF, while git stores LF. Keep each file's existing line endings.
  A multi-line exact-match edit written with LF won't match a CRLF file.
- Commit one repo at a time. Push only when the owner has told you to, and in the order of §7.

---

## 5. Step 1: the API (`sage-tools-api`)

### 5.1 Add the tests

**`test/unit/scoresheets/PageChunkBuilder.test.mjs`**: add inside `describe("Match", …)`, before the
`"still formats a side with a code but no players"` test:

```js
    it("adds the team names, trimmed, when the row has the columns", () => {
        const m = new Match({ ...row(1, "A1_1", "Ann", "Bob", "A2_1", "Cy", "Di"), teamName1: " Spice Girls ", teamName2: "Wok Stars" }, "Cup");
        assert.deepEqual(m.toTemplateData(), {
            eventName: "Cup", matchNumber: "1", team1: "A1_1 - Ann / Bob", team2: "A2_1 - Cy / Di",
            team1Name: "Spice Girls", team2Name: "Wok Stars", hasTeamNames: true,
        });
    });

    it("sets hasTeamNames for empty name columns, so the sheet still has the line", () => {
        const data = new Match({ ...row(1, "QF-1_1", "", "", "QF-2_1", "", ""), teamName1: "", teamName2: "" }, "Cup").toTemplateData();
        assert.deepEqual([data.team1Name, data.team2Name, data.hasTeamNames], ["", "", true]);
    });
```

The existing tests in this file must pass **unedited**. They pin that a row without the columns gets exactly
the template data it gets today.

**`test/unit/scoresheets/CsvService.test.mjs`**: add this import under the `CsvService` one:

```js
import { ValidationError } from "../../../src/shared/errors.mjs";
```

and these tests before `"throws for a file that does not exist"`:

```js
    it("reads a quoted team name with a comma and a doubled quote", () => {
        const file = write("names.csv", 'matchNumber,teamCode1,teamName1,teamCode2,teamName2\n1,A_1,"Dink, Inc",B_1,"Say ""hi"""\n');
        assert.deepEqual(new CsvService().parseFile(file), [
            { matchNumber: "1", teamCode1: "A_1", teamName1: "Dink, Inc", teamCode2: "B_1", teamName2: 'Say "hi"' },
        ]);
    });

    it("still reads a stray quote in a CSV without team-name columns", () => {
        const file = write("stray.csv", 'matchNumber,team1Player1\n1,Jun "JR" Cruz\n');
        assert.deepEqual(new CsvService().parseFile(file), [{ matchNumber: "1", team1Player1: 'Jun "JR" Cruz' }]);
    });

    it("answers a validation error for a stray quote in a CSV with team-name columns", () => {
        const file = write("bad.csv", 'matchNumber,teamName1\n1,6" Team\n');
        assert.throws(() => new CsvService().parseFile(file), ValidationError);
    });

    it("does not leave quoted fields on for the next file", () => {
        const service = new CsvService();
        service.parseFile(write("first.csv", 'matchNumber,teamName1\n1,"A, B"\n'));
        const next = service.parseFile(write("second.csv", 'matchNumber,team1Player1\n1,Jun "JR" Cruz\n'));
        assert.equal(next[0].team1Player1, 'Jun "JR" Cruz');
    });
```

**`test/unit/scoresheets/ScoresheetService.test.mjs`**: in `build()`, change the fake template's `render` so a
test can see the flag. Sheets without names render as before, so the existing tests are unaffected:

```js
render: d => `<${type}:${d.matchNumber}|${d.team1}|${d.team2}|${d.eventName}${d.hasTeamNames ? "|named" : ""}>`,
```

Then add these first inside `describe("ScoresheetService.generate", …)`:

```js
    it("gives blank sheets the team-name columns when the CSV has them", async () => {
        const { service, rec } = build({ rows: [{ ...row(1), teamName1: "Spice Girls", teamName2: "" }] });
        await service.generate({ ...base, blankCount: 1 });
        const html = rec.renders.map(r => r.html).join("");
        assert.equal(html.split("|named>").length - 1, 2);
    });

    it("gives blank sheets no team-name columns when the CSV has none", async () => {
        const { service, rec } = build();
        await service.generate({ ...base, blankCount: 1 });
        assert.ok(!rec.renders.map(r => r.html).join("").includes("|named"));
    });
```

**`test/unit/scoresheets/TemplateService.test.mjs`**: add this above `let dir, csvPath;`:

```js
// The same shape with team names: a real one, one with an ampersand and a comma, an unseated side.
const NAMED_CSV = [
    "matchNumber,teamCode1,team1Player1,team1Player2,teamCode2,team2Player1,team2Player2,teamName1,teamName2",
    "1,A1_1,A1_1 ,A1_1 ,A2_1,A2_1 ,A2_1 ,DINK ME BABY 1 MORE TIME,KITCHEN CLUTCHERS",
    '2,B1_1,Ann Cruz,Ben Diaz,B2_1,Cy Eng,Di Fox,"Dink & Drive, Inc",SPICE GIRLS',
    "3,QF-1_1,QF-1_1 ,QF-1_1 ,QF-2_1,QF-2_1 ,QF-2_1 ,,",
    "",
].join("\n");
```

and this as the first test inside the `for (const type of ScoresheetConfig.list())` loop:

```js
        it(`renders ${type} with team names to the HTML in its snapshot`, t => {
            const namedPath = path.join(dir, "named.csv");
            fs.writeFileSync(namedPath, NAMED_CSV);
            const items = new CsvService().parseFile(namedPath);
            const { render } = new TemplateService(TEMPLATES_DIR).load(type);
            const chunks = new PageChunkBuilder(ScoresheetConfig.get(type)).build(items, render, "Snapshot Cup 2026");
            const html = chunks.join("");
            assert.ok(html.includes("DINK ME BABY 1 MORE TIME") && html.includes("Dink &amp; Drive, Inc"));
            t.assert.snapshot({ type, pages: chunks.map(collapseImages) });
        });
```

**`test/e2e/pdf.e2e.test.mjs`**: add these imports above the `startApp` one:

```js
import { PDFDocument } from "pdf-lib";
import { ScoresheetConfig } from "../../src/scoresheets/ScoresheetConfig.mjs";
```

and this before the `"POST /scoresheets/generate/stream …"` test:

```js
    // The sheets have no spare height: this is what catches a layout spilling onto a second page.
    const LONG = "A VERY LONG TEAM NAME THAT GOES ON AND ON AND ON";
    const NAMED_CSV = [
        "matchNumber,teamCode1,team1Player1,team1Player2,teamCode2,team2Player1,team2Player2,teamName1,teamName2",
        ...[1, 2, 3, 4].map(n => `${n},A_1,Ann Cruz,Ben Diaz,B_1,Cy Eng,Di Fox,${LONG},${LONG}`),
        "",
    ].join("\n");
    for (const type of ScoresheetConfig.list()) {
        it(`${type} with a 48-character team name stays at one PDF page per chunk`, async () => {
            const res = await fetch(`${app.baseUrl}/v3/scoresheets`, {
                method: "POST",
                body: multipart({ evt: "E2E Cup", type }, { content: NAMED_CSV, name: "matches.csv" }),
            });
            assert.equal(res.status, 200);
            const pdf = await PDFDocument.load(Buffer.from(await res.arrayBuffer()));
            assert.equal(pdf.getPageCount(), Math.ceil(4 / ScoresheetConfig.get(type).perPage));
        });
    }
```

`pdf-lib` comes with `pdf-merger-js`, so it needs no new dependency.

**See them fail.** Run `npm test`. These should fail, and nothing else:

- `adds the team names…` and `sets hasTeamNames…`
- `reads a quoted team name…` and `answers a validation error…`
- `gives blank sheets the team-name columns…`
- `renders <type> with team names…`, once for each of the four types

The new e2e cases aren't part of `npm test`. They pass before the change too, because no name prints yet, and
they guard against overflow once it does.

### 5.2 Write the code

**`src/scoresheets/CsvService.mjs`**: replace the whole file:

```js
import fs from "fs";
import csvToJson from "convert-csv-to-json";
import { ValidationError } from "../shared/errors.mjs";

export class CsvService {
    // Quoted fields stay off unless the header has a team-name column, so a CSV
    // from any existing client (a stray " in a name included) parses as it always did.
    // The library's settings are shared across calls, so this sets them on every call.
    parseFile(csvPath) {
        const header = fs.readFileSync(csvPath, "utf8").split(/\r?\n/, 1)[0];
        const hasTeamNames = /\bteamName[12]\b/.test(header);
        csvToJson.fieldDelimiter(",").supportQuotedField(hasTeamNames);
        try {
            return csvToJson.getJsonFromCsv(csvPath);
        } catch (err) {
            if (hasTeamNames && /mismatched quotes/.test(err.message)) {
                throw new ValidationError("The CSV has a mismatched quote in one of its rows.");
            }
            throw err;
        }
    }
}
```

**`src/scoresheets/PageChunkBuilder.mjs`**: in `Match`, add this at the end of the constructor:

```js
        // Only a team event's CSV has these columns; without them the template data is unchanged.
        this.teamNames = "teamName1" in row || "teamName2" in row
            ? { team1Name: (row.teamName1 ?? "").trim(), team2Name: (row.teamName2 ?? "").trim(), hasTeamNames: true }
            : {};
```

and in `toTemplateData()`, add this after `team2: this.team2,`:

```js
            ...this.teamNames,
```

`#formatTeam` doesn't change.

**`src/scoresheets/ScoresheetService.mjs`**: three edits:

```js
    static #makeBlankItem(withTeamNames) {
```

```js
            team2Player2: ScoresheetService.#BLANK_FIELD,
            ...(withTeamNames ? { teamName1: ScoresheetService.#BLANK_FIELD, teamName2: ScoresheetService.#BLANK_FIELD } : {}),
```

```js
            const items = this.csvService.parseFile(csvPath);
            const withTeamNames = items.length > 0 && "teamName1" in items[0];

            for (let i = 0; i < numBlanks; i++) {
                items.push(ScoresheetService.#makeBlankItem(withTeamNames));
            }
```

**`src/scoresheets/routes.mjs`**: the two lines reading `description: The matches CSV for this event.` become:

```
description: The matches CSV for this event. Optional teamName1 and teamName2 columns print each team's name on the sheet.
```

**`templates/scoresheet.html` and `templates/scoresheet-best-of-3.html`**: replace the two underline spans. Put
each on one line, because an inline block adds nothing to an unnamed sheet's HTML:

```html
<span class="underline">{{#if team1Name}}<span class="team-label">{{team1Name}}</span>{{/if}}{{team1}}</span>
```

```html
<span class="underline">{{#if team2Name}}<span class="team-label">{{team2Name}}</span>{{/if}}{{team2}}</span>
```

Add to the end of **`templates/scoresheet.css` and `templates/scoresheet-best-of-3.css`**:

```css
.team-label {
    margin-right: 0.5rem;
}
```

**`templates/scoresheet-no-ref.html` and `templates/scoresheet-no-ref-wide.html`**: insert this block immediately
above the existing `<div class="team-headers">` that holds `{{team1}}` and `{{team2}}`. The `{{#if}}` and `{{/if}}`
tags must be on lines of their own, because Handlebars then removes those lines when the block is off:

```html
      {{#if hasTeamNames}}
      <div class="team-headers team-name-line">
        <div class="field team-name">
          <span class="underline">{{team1Name}}</span>
        </div>
        <div class="field team-name">
          <span class="underline">{{team2Name}}</span>
        </div>
      </div>
      {{/if}}
```

Add to the end of **`templates/scoresheet-no-ref.css` and `templates/scoresheet-no-ref-wide.css`**. Without this
an empty name's underline collapses onto the line above it:

```css
.team-name-line .underline {
  min-height: 1.2em;
}
```

**`templates/scoresheet-no-ref-wide.html` only**: change the opening container to:

```html
<div class="main-container{{#if hasTeamNames}} named{{/if}}">
```

**`templates/scoresheet-no-ref-wide.css` only**: add to the end:

```css
.named .square {
  height: 200px;
}
```

**`package.json`**: `"version"` becomes `3.6.0`. Leave `package-lock.json` alone, since its version isn't kept in step.

**`README.md`**: add this at the top of the Changelog's versions, above `### 3.5.0`:

```markdown
### 3.6.0

- **A team event's scoresheets can print team names.** A matches CSV may carry optional
  `teamName1` and `teamName2` columns, which the Scoresheet Generator adds for a team event.
  Standard and Best of 3 print the name inside the team's underline, before the code. No
  Referee and No Referee, Wide print it on a line of its own above the code, and Wide's score
  boxes are 200 px tall on a named sheet so it stays on one page. A CSV with these columns is
  read with quoted fields on; a CSV without them prints and parses exactly as before.
- **Tests:** `test/unit/scoresheets/PageChunkBuilder.test.mjs`,
  `test/unit/scoresheets/CsvService.test.mjs`, `test/unit/scoresheets/ScoresheetService.test.mjs`,
  `test/unit/scoresheets/TemplateService.test.mjs` and its snapshot, `test/e2e/pdf.e2e.test.mjs`.
```

### 5.3 Regenerate the snapshots and check them

```bash
npm run test:snapshots
```

Then look at `git diff test/unit/scoresheets/TemplateService.test.mjs.snapshot`. **This is the check that other
events' sheets didn't change:**

- Four new entries appear, one `with team names` per type.
- In the four existing entries, only the `"styles"` line may change, by the CSS added above.
- If any existing entry's `"pages"` changed, an unnamed sheet's HTML changed. That's a bug in the template edit:
  fix the template, and don't accept the snapshot.

Generate snapshots only on this Windows checkout. They were made from its CRLF templates and contain `\r\n`, so
a checkout with LF templates would rewrite every line.

### 5.4 Run everything

```bash
npm run verify
```

```bash
RUN_PDF_E2E=1 PUPPETEER_EXECUTABLE_PATH="C:/Program Files/Google/Chrome/Application/chrome.exe" npm run test:e2e
```

That command is for Git Bash. On the 3.5.0 baseline these gave 1708 passing and 6 passing, up from 1696 and 2. All must
pass, and the second is the one that proves nothing spills onto a second page.

### 5.5 Commit

Stage only the files named in this section. Commit message:

```
3.6.0: a team event's scoresheets can print team names

Tests: existing template snapshots change only in "styles", for the CSS added;
their pages are unchanged.
```

---

## 6. Step 2: the site (`sage-match-control.github.io`)

### 6.1 Add the test

Create **`_tests/unit/domain/scoresheet-names.test.mjs`**:

```js
// domain/scoresheet-names.js
import test from 'node:test';
import assert from 'node:assert/strict';
import { parseCSV } from '../../../lib/v1/domain/csv.js';
import { withTeamNames } from '../../../lib/v1/domain/scoresheet-names.js';
import { loadSnapshot } from '../../helpers/fixtures.mjs';

const MATCHES = 'matchNumber,teamCode1,team1Player1,team1Score,teamCode2,team2Player1,team2Score\r\n';
const STANDINGS = 'teamCode,teamName,totalPoints,totalOpponentPoints,quotient,bracket\r\n'
  + 'A,Spice Girls,0,0,-,1\r\nB,Wok Stars,0,0,-,1\r\nC,"Dink, ""Inc""",0,0,-,1\r\n'
  + 'QF-1,,0,0,-,\r\nSF-A,,0,0,-,\r\nFi-Green,Green Team,0,0,-,\r\n';
const names = csv => {
  const [header, ...rows] = parseCSV(csv);
  const i1 = header.indexOf('teamName1'), i2 = header.indexOf('teamName2');
  return rows.map(r => [r[i1], r[i2]]);
};

test('a group side gets its name from its team, and the other cells are kept', () => {
  const csv = withTeamNames(`${MATCHES}1,A_1,"Cruz, Ann",11,B_1,Bob,9\r\n`, STANDINGS);
  const rows = parseCSV(csv);
  assert.deepEqual(rows[0].slice(-2), ['teamName1', 'teamName2']);
  assert.deepEqual(rows[1], ['1', 'A_1', 'Cruz, Ann', '11', 'B_1', 'Bob', '9', 'Spice Girls', 'Wok Stars']);
});

test('a seated playoff side gets the name on its own row', () => {
  assert.deepEqual(names(withTeamNames(`${MATCHES}1,Fi-Green_3,x,,QF-1_1,y,\r\n`, STANDINGS))[0], ['Green Team', '']);
});

test('a lettered playoff slot falls back to that team, and an unseated side is empty', () => {
  assert.deepEqual(names(withTeamNames(`${MATCHES}1,SF-A_1,x,,QF-1_1,y,\r\n`, STANDINGS))[0], ['Spice Girls', '']);
});

test('a comma and a quote in a name survive being written and read back', () => {
  assert.deepEqual(names(withTeamNames(`${MATCHES}1,C_1,x,,A_1,y,\r\n`, STANDINGS))[0], ['Dink, "Inc"', 'Spice Girls']);
});

test('the input comes back unchanged with no code columns or no names', () => {
  const noCodes = 'matchNumber,court\r\n1,2\r\n';
  assert.equal(withTeamNames(noCodes, STANDINGS), noCodes);
  const plain = `${MATCHES}1,A_1,x,,B_1,y,\r\n`;
  assert.equal(withTeamNames(plain, 'teamCode,player1,player2,wins,loss,quotient,bracket\r\nA_1,x,y,0,0,0,1\r\n'), plain);
  assert.equal(withTeamNames(plain, ''), plain);
  assert.equal(withTeamNames('', STANDINGS), '');
});

test('every match of a real team snapshot keeps its row and gains names', () => {
  const { matchesCsv, standingsCsv } = loadSnapshot('pickledrive-anniversary-2026').facilities[0];
  const before = parseCSV(matchesCsv), after = parseCSV(withTeamNames(matchesCsv, standingsCsv));
  assert.equal(after.length, before.length);
  after.slice(1).forEach((row, i) => {
    assert.deepEqual(row.slice(0, before[0].length), before[i + 1]);
    assert.ok(row.at(-2) && row.at(-1), `match ${row[0]} has both names`);
  });
});
```

From `_tests`, `npm test` should now fail because the module doesn't exist.

### 6.2 Write the code

Create **`lib/v1/domain/scoresheet-names.js`**. It's a new file, and not a new export in `teams.js`, because of the
engine rule that a module only starts using another's new export in a later push: a browser may hold a cached
`teams.js` for 10 minutes, but it can't hold a stale copy of a file that didn't exist.

```js
// A team event's scoresheets print each team's name (sage-docs/docs/specs/.../scoresheet-team-names-spec.md).
// The Scoresheet Generator sends this CSV in place of the snapshot's matchesCsv.

import { parseCSV } from './csv.js';
import { rowsToStandings } from './standings.js';
import { sideOf, teamNameOf } from './teams.js';

const csvCell = v => /[",\r\n]/.test(v) ? `"${v.replace(/"/g, '""')}"` : v;

/**
 * matchesCsv with teamName1 and teamName2 added to every row, '' for a side with no name yet.
 * Returned unchanged when it has no teamCode1 and teamCode2 columns, or the standings have no names.
 * @param {string} matchesCsv
 * @param {string} standingsCsv
 * @returns {string}
 */
export function withTeamNames(matchesCsv, standingsCsv){
  const rows = parseCSV(matchesCsv);
  if(rows.length === 0) return matchesCsv;
  const header = rows[0].map(h => h.trim());
  const i1 = header.indexOf('teamCode1'), i2 = header.indexOf('teamCode2');
  if(i1 < 0 || i2 < 0) return matchesCsv;

  const standings = rowsToStandings(parseCSV(standingsCsv));
  if(!standings.some(s => s.teamName)) return matchesCsv;
  const rowByCode = new Map(standings.map(s => [s.teamCode, s]));
  const nameOf = code => (code || '').trim() ? teamNameOf(sideOf(code.trim()), rowByCode) || '' : '';

  const out = rows.map((row, r) => r === 0
    ? [...row, 'teamName1', 'teamName2']
    : [...row, nameOf(row[i1]), nameOf(row[i2])]);
  return out.map(row => row.map(csvCell).join(',')).join('\r\n') + '\r\n';
}
```

**`tools/scoresheet-generator.html`**: in the form's `submit` handler, replace this one line:

```js
      csvPart = new Blob([fac.matchesCsv], { type: 'text/csv' });
```

with:

```js
      let matchesCsv = fac.matchesCsv;
      if (RAW_EVENTS[CURRENT_EVENT_KEY]?.type === 'team') {
        // Only a team event's sheets print names (sage-docs/docs/specs/.../scoresheet-team-names-spec.md).
        try {
          const { withTeamNames } = await import('/lib/v1/domain/scoresheet-names.js');
          matchesCsv = withTeamNames(fac.matchesCsv, fac.standingsCsv || '');
        } catch (_) {
          setStatus("Couldn't load the team names. Try again.", 'err');
          return;
        }
      }
      csvPart = new Blob([matchesCsv], { type: 'text/csv' });
```

It's a dynamic import because the page's script is a classic one, and other events' sheets shouldn't depend on this
file. If the import fails on venue wifi, the page stops and says so rather than quietly printing sheets without names.
Upload mode isn't touched, so a hand-made CSV prints what it says.

### 6.3 Run and commit

From `_tests`:

```bash
npm run verify
```

This runs the unit tests and then the Playwright comparison harness. Only the site's unit tests were run for this spec.
The harness has no case for the generator page, and `lib/v1` only gains a new file, so it should report no difference.
If it does, follow §9.

Stage only `lib/v1/domain/scoresheet-names.js`, `tools/scoresheet-generator.html` and the new test. Commit message:

```
Scoresheet Generator: a team event's sheets print team names
```

---

## 7. Step 3: ship and check

Only when the owner has said to push. The order matters, because a page sending these columns to a 3.5.0 API would
misread a team name with a comma. A browser still holding the old page keeps working, since it sends no new columns.

**1. Push `sage-tools-api`,** then wait for the Cloud Run deploy to finish. This should show `x-app-version: 3.6.0`:

```bash
curl -sI https://sage-tools-api-811926984834.us-central1.run.app/ping
```

**2. Check the deployed API,** from a temporary folder, not a repo:

```bash
printf 'matchNumber,teamCode1,team1Player1,team1Player2,teamCode2,team2Player1,team2Player2,teamName1,teamName2\n1,A_1,A_1,A_1,B_1,B_1,B_1,"Dink, Inc",Spice Girls\n' > named.csv
```

```bash
curl -s -o named.pdf -H 'Accept: application/pdf' -F 'evt=API check' -F 'type=scoresheet' -F 'csv=@named.csv;type=text/csv' https://sage-tools-api-811926984834.us-central1.run.app/v3/scoresheets
```

Open `named.pdf`. Team 1's line should read `Dink, Inc A_1 -`, and Team 2's `Spice Girls B_1 -`.

**3. Push the site,** and allow up to 10 minutes for its cache.

**4. Check the live page.** It has to be the live page: the production API only accepts requests from
`https://sage-match-control.github.io`, so a copy on `localhost` can't post to it. Open each link, then generate:

| # | Open `https://sage-match-control.github.io/tools/scoresheet-generator.html` with | Check |
| --- | --- | --- |
| C1 | `?day=ladies-dink-day1`, each of the four layouts | Names beside the right codes, for both teams. An unseated playoff sheet has no name, and on the No Referee layouts an empty line. The PDF has 36 pages for Standard, Best of 3 and Wide, and 18 for No Referee, if the event still has 72 matches |
| C2 | `?day=ladies-dink-day1`, No Referee, Wide, 2 blank sheets | Spare sheets have the same layout, with an empty name line |
| C3 | `?day=piggleball-day1`, Standard | Looks as it did before, with no name line |
| C4 | Upload a matches CSV without the new columns | Prints as before |
| C5 | Print one Standard sheet at a team event | Enough room on the line to write two players' names by hand. **The owner's call** |

If a check fails, tell the owner before reverting anything. Both pushes can be reverted on their own, and reverting the
site alone stops names being sent.

---

## 8. Step 4: the docs

Do this after §7's checks pass. Docs describe what the system does now, in the present tense.

**`sage-docs/docs/features/scoresheet-generator.md`**: add this paragraph above `Every registered facility spreadsheet's`:

```markdown
At a team event, each sheet prints the team's name beside its pair code, from the day's last
published sync. A playoff team that isn't seated yet has no name to print, so print after seating
for fully named sheets. An uploaded CSV prints names if it has `teamName1` and `teamName2` columns.
```

**`sage-docs/docs/technical/scoresheet-pipeline.md`**: replace the sentences starting `That snapshot's CSV already has`
and ending `where its input came from.` with:

```markdown
That snapshot's CSV already has the header this pipeline's `CsvService` expects (matched by header
name, not position). For an event of type `team`, the page first adds `teamName1` and `teamName2`
columns with `withTeamNames` from `lib/v1/domain/scoresheet-names.js`, which reads the same
snapshot's `standingsCsv`. Otherwise the CSV is sent as it is. This pipeline has no awareness of
where its input came from.

`Match` passes the names to the templates only when the columns exist, along with `hasTeamNames`, so
a CSV without them renders the same HTML as before. `CsvService` turns quoted fields on only for a
CSV with these columns, since quoted parsing throws on a stray `"` that the plain parser accepts.
Every layout has no spare height: `test/e2e/pdf.e2e.test.mjs` checks each one stays at one PDF page
per chunk with a 48-character name.
```

**`sage-docs/docs/usage/printables.md`**: under *Where the formats differ*, replace the first Team bullet with:

```markdown
- **Team:** scoresheets carry the matchup's four matches, and each prints the team's name beside
  its pair code. Print once lineups are known, or print blanks. A playoff team's name prints once
  it is seated, so print those sheets after seating, or write the team in by hand.
```

**`sage-docs/docs/technical/site-engine.md`**: add this paragraph above the one starting `` `domain/matches.js` reads an optional ``:

```markdown
`domain/scoresheet-names.js` `withTeamNames(matchesCsv, standingsCsv)` adds `teamName1` and
`teamName2` columns to a team event's matches CSV, for the Scoresheet Generator to post.
```

**This spec:** move it to `implemented/` by the procedure in the [Specs index](../README.md#moving-a-spec-between-folders),
and set its status line to `**Status: implemented.**` with the API version and the site commit. Record in §11 anything built
differently from this page. If C5 is still waiting on the owner, move it to `in-progress/` instead and say so in the status line.

**`sage-docs` version:** add a MINOR entry at the top of `docs/changelog.md` and set the same version in `mkdocs.yml`'s
`copyright` line, as the changelog's *Releasing a version* describes. Commit message: `docs <version>: team names on scoresheets`.

**`D:\Personal\SAGE\CLAUDE.md`** (the workspace repo): add this to the end of the paragraph starting
`The scoresheet feature is independent of all of the above`:

```markdown
For a team event the page first adds `teamName1` and `teamName2` columns from the snapshot's
standings (`lib/v1/domain/scoresheet-names.js`), and the templates print them only when they exist.
```

Commit message: `CLAUDE.md: a team event's scoresheets print team names`.

---

## 9. When to stop and ask

Stop and tell the owner, rather than working around it, if:

- An event falls within 3 days, or any baseline in §4 fails.
- An anchor in §5 or §6 isn't where this page says, or the code has changed so the snippet no longer fits.
- A test other than those listed in §5.1 fails, or `npm run verify` fails, or the comparison harness shows a difference.
- An existing snapshot's `pages` changes (§5.3).
- A layout spills onto a second page. Don't shrink anything beyond what §3 says without asking: the layouts are the owner's decisions (§12).
- A real team name is longer than 48 characters. That length is the tested limit.

---

## 10. What was tested for this spec

On a throwaway copy of `sage-tools-api` and the site, with the code above:

| Check | Result |
| --- | --- |
| API `npm test` and `npm run verify` | Passed, 1708 tests (1696 before) |
| API `test:e2e` with real Chromium | Passed, 6 tests, each layout at the expected page count |
| Existing template snapshots | `pages` identical to the committed ones; only `styles` changes |
| Site `npm test` | Passed: 528 pass, 6 skipped, 0 fail (519, 6, 0 before) |
| The patched page in Chrome, posting stubbed | Ladies' Dink sends both columns with real names, and empty for `QF-1_1`. Piggleball sends neither. A blocked module shows the error and posts nothing |
| The page's real CSV through the real templates | Names on the right sides, with an empty line for an unseated team on Wide |

Not run: the site's Playwright comparison harness, anything against the deployed API, a full PDF from the live page, and a printed sheet. §6.3 and §7 cover those.

---

## 11. Divergences

None yet. Record here anything built differently from this page.

---

## 12. Owner's decisions

All answered on 2026-11-06.

| # | Decision | Answer |
| --- | --- | --- |
| D1 | Where the name goes on Standard and Best of 3 | The same line, inside the underline with the code |
| D2 | Score boxes on No Referee, Wide, named sheets only | 240 → 200 px |
| D3 | What an unseated playoff team prints | Empty, not `Seed 1 · TBD` |
| D4 | Whether the page has a switch for names | No switch: a team event always gets names |

---
**Related:** [Scoresheet Generator](../../features/scoresheet-generator.md) ·
[scoresheet pipeline](../../technical/scoresheet-pipeline.md) ·
[event picker spec](../implemented/scoresheet-event-picker-spec.md)
