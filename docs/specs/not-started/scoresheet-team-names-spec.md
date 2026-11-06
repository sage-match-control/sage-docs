# Spec — team names on scoresheets

> **Status: not started.** Written 2026-11-06 against `sage-tools-api` 3.5.0 (`f885c64`) and the
> site at `9c9b584`. Nothing here is built. The layouts in §5 were measured on throwaway copies of
> the templates, not in the repos. Build it **after** the Angeles City Employees Sportsfest
> (13 November 2026), as §10 explains. §11 records the owner's decisions.

Print each team's name on a team event's scoresheets, next to the code that is there today. Every
other event's sheets stay exactly as they are.

Three repos' worth of change, none of it a new endpoint:

| Repo | Change |
| --- | --- |
| `sage-tools-api` | Reads two optional CSV columns and prints them, on all four layouts (§4, §5) |
| `sage-match-control.github.io` | The generator page adds those columns for a team event (§3) |
| `sage-docs` | Three pages, plus this spec's status (§9) |

---

## 1. Why

A team event's sheet says only which pair is playing. At Ladies' Dink the player columns still hold
pair codes, so a sheet reads:

```
Team 1: A1_1 -
```

A scorer or player holding that slip has to look up who `A1` is. Once a lineup is in the workbook
the sheet gains two player names but still no team. The team name is what people at the venue know
a team by, and it is already in every published snapshot: `standingsCsv` has `teamCode` and
`teamName` beside the `matchesCsv` the generator prints from.

The code stays, because its `_1` is the only thing on the sheet that says which pair of the team is
playing.

---

## 2. What it does

| Situation | Team 1's line, on a sheet |
| --- | --- |
| Team event, group stage | **DINK ME BABY 1 MORE TIME** `A1_1 -` |
| Team event, lineup known | **DINK ME BABY 1 MORE TIME** `A1_1 - Yanna / Betty` |
| Team event, playoff side already seated | **KITCHEN CLUTCHERS** `QF-3_1 -` |
| Team event, playoff side not seated yet | `QF-3_1 -`, with room to write the team in (§6) |
| Standard or dual-meet event | Unchanged |
| Uploaded CSV without the new columns | Unchanged |
| Uploaded CSV with the new columns | Prints them, whatever the event |

### Not in scope

- **Pair labels** such as *Pair 1* or *MD* from `display.pairs`. The code already identifies the
  pair, and this keeps the change to one idea.
- **Replacing the code with the name.** See §1.
- **Names for standard and dual-meet events.** Their standings rows name pairs, and their sheets
  already carry the player names.
- **Team colours or marks.** The ask is the name, and a printed mark needs a colour printer.

---

## 3. The site: where the names are joined

The join happens in the browser, because the rule for who a side belongs to already lives in
`lib/v1/domain/teams.js`, with the rule for a playoff side whose slot is a team letter (`SF-A`).
Joining on the server would be another hand-kept copy of a rule, like the ones CLAUDE.md lists
under "must be kept in sync by hand".

A new file, `lib/v1/domain/scoresheet-names.js`, exports one pure function:

```js
withTeamNames(matchesCsv, standingsCsv) // -> CSV text
```

1. Parse both with `parseCSV`, finding columns by header name.
2. Build `rowByCode` from the standings rows.
3. For each match row, look up `teamNameOf(sideOf(code), rowByCode)` for `teamCode1` and `teamCode2`.
4. Append `teamName1` and `teamName2` to the header and to every row, empty when there is no name.
5. Write the CSV back, quoting any cell with a comma, quote or line break.

It returns `matchesCsv` unchanged if the header has no `teamCode1` and `teamCode2`, or the standings
have no `teamName` column.

A new file rather than a new export in `teams.js`, because of the engine rule that one module only
starts using another's new export in a later push: a browser may hold a cached `teams.js` for 10
minutes, and it cannot hold a cached copy of a file that did not exist. `parseCSV`, `sideOf` and
`teamNameOf` are all existing exports.

`tools/scoresheet-generator.html` calls it in its event mode, just before it makes the `Blob`, and
only when `RAW_EVENTS[CURRENT_EVENT_KEY].type === 'team'`:

```js
const { withTeamNames } = await import('/lib/v1/domain/scoresheet-names.js');
csvText = withTeamNames(fac.matchesCsv, fac.standingsCsv);
```

A dynamic import, because the page's script is a classic one and every other event's sheets should
not depend on this file. If the import fails (venue wifi), the page stops with *Couldn't load the
team names. Try again.* It does not silently print sheets without them. Upload mode never touches
any of this, so a hand-made CSV prints what it says.

---

## 4. The API: reading and printing the columns

No route changes, no new parameter. The `csv` part simply may carry two more columns.

**`Match` (`PageChunkBuilder.mjs`).** `#formatTeam` is untouched. A row with either column gets three
more template fields. A row with neither gets none, so `toTemplateData()` returns what it does today:

```js
{ team1Name, team2Name, hasTeamNames: true }  // names trimmed, "" when empty
```

**Blank sheets (`ScoresheetService.mjs`).** `#makeBlankItem` adds empty `teamName1` and `teamName2`
when the parsed CSV's first row has those columns, so a spare sheet at a team event has the same
layout as the rest and a line for the name.

**Commas (`CsvService.mjs`).** The parser runs with quoted fields off, which is why a name like
`Dink, Inc` shifts every later column:

| Input | Quoted off (today) | Quoted on |
| --- | --- | --- |
| `"Dink, Inc",A2_1` | `"Dink` then ` Inc"` | `Dink, Inc` then `A2_1` |
| `Jun "JR" Cruz` (a stray quote) | `Jun "JR" Cruz` | **Throws** *Row contains mismatched quotes!* |

Turning it on for everything would make a CSV that works today fail, on routes that are frozen. So
`CsvService` reads the header first and turns quoted fields on **only when the header has
`teamName1` or `teamName2`**. A CSV without them is parsed exactly as it is now. With them, a
mismatched quote answers `400 validation_error` naming the problem rather than a `500`. The page's
own CSV is always balanced, so that only reaches a hand-edited upload.

The library's settings are shared across calls, so `parseFile` sets `supportQuotedField` both ways
on every call. Parsing is synchronous, so two requests cannot interleave.

The frozen routes share this pipeline. They behave identically for anything an existing client
sends, since no existing client sends these columns.

---

## 5. The layouts

A stacked name line looked like the obvious design, but it does not fit on three of the four
layouts. Measured at the service's real settings (Letter, scale 0.8, 2 or 4 sheets per page) with
the real `PageChunkBuilder`, using a real 24-character name and a stress-test 48-character one:

| Layout | Stacked name line | Chosen | Result with the chosen layout |
| --- | --- | --- | --- |
| **Standard** | 2 PDF pages | Name on the existing `Team 1:` line | 1 page, at 24 and 48 characters |
| **Best of 3** | 2 PDF pages | Name on the existing `Team 1:` line | 1 page, at 24 and 48 characters |
| **No Referee** | 1 page | Stacked | 1 page, and a 48-character name wraps to two lines and still fits |
| **No Referee, Wide** | 2 PDF pages | Stacked, score boxes 240 → 200 px tall | 1 page at 24 and 48 characters |

Longest real names so far: 26 characters (PickleDrive), 24 (Ladies' Dink). The 48-character case on
**No Referee, Wide** only just fits, so it is the one to re-check if a longer name ever appears.

**Standard and Best of 3.** Their team line runs the full width of the sheet, so the name can share
it, inside the same underline as the code, and still leave about two-thirds of the line for the code
and handwriting at 24 characters (about a third at 48):

```html
<span>Team 1:</span>
<span class="underline">{{#if team1Name}}<span class="team-label">{{team1Name}}</span>{{/if}}{{team1}}</span>
```

`.team-label` takes its bold from the underline and adds a small right margin, to separate the name
from the code. Team 2's line is upside down like the rest of that half of the sheet, and needs nothing
extra.

**No Referee and No Referee, Wide.** Each team has a half-width column, too narrow to share, so the
name goes on its own underlined line above the code, in a `{{#if hasTeamNames}}` block.

**No Referee, Wide** also needs the height back. `class="main-container{{#if hasTeamNames}} named{{/if}}"`
and `.named .square { height: 200px; }` shorten the score boxes on a named sheet only. A 200 px box is
still plenty to write a score in.

A block is either inline, as on Standard and Best of 3, or its `{{#if}}` tags sit on lines of their
own, which Handlebars removes when the block is off. Either way an unnamed sheet's HTML stays
byte-for-byte the same. §8 pins that.

---

## 6. A playoff side with no team yet

Ladies' Dink's standings hold `QF-1` to `Fi-2` with an empty `teamName` until the side is seated, and
`QF-1`'s slot is a number, so there is no group team to fall back on. Such a side gets an empty
`teamName1`.

- **No Referee, Wide and No Referee:** the name line still prints, empty, for writing on. That is why
  the switch is `hasTeamNames` (the column exists) and not "this name is set", and it keeps every
  sheet in a set the same height.
- **Standard and Best of 3:** nothing extra prints, and the team goes on the existing line by hand.

The Hub's `Seed 1 · TBD` wording is deliberately not used. It would have to be crossed out, and an
empty line asks for what is wanted. Printing after seating gives named sheets.

A team whose name equals its code's side (Millennium's `Red`, code `Red_1`) prints both. That is
redundant and harmless, and a special case would be more code than it is worth.

---

## 7. Rollout order

1. **API first** (3.6.0). It accepts the columns, and no client sends them yet.
2. **Site second.** A page sending the columns to a 3.5.0 API would misread a name with a comma.

A browser still holding the old page keeps working, since it sends no new columns and prints what it
prints today.

---

## 8. Tests

**`sage-tools-api`, written first, each seen to fail.** `npm test`, then `npm run verify` before the push.

| File | Pins |
| --- | --- |
| `test/unit/scoresheets/PageChunkBuilder.test.mjs` | The names and `hasTeamNames` with the columns, trimmed; empty columns still set `hasTeamNames`. The existing `deepEqual` with no columns passes **unedited** |
| `test/unit/scoresheets/CsvService.test.mjs` | A comma in a quoted name; a stray quote still parses without the columns; a stray quote with them is a `ValidationError`; a named parse does not leave quoted mode on for the next |
| `test/unit/scoresheets/ScoresheetService.test.mjs` | Blank sheets get the empty columns only when the CSV has them |
| `test/unit/scoresheets/TemplateService.test.mjs` | A new named fixture, one snapshot per layout. The existing snapshots' `pages` **do not change**; only `styles` does, for the CSS added, and the commit message says so |
| `test/integration/v3.test.mjs` | `POST /v3/scoresheets` accepts a CSV with the columns |
| `test/e2e/pdf.e2e.test.mjs` (opt-in, `RUN_PDF_E2E=1`) | Every layout at a 48-character name renders one PDF page per chunk. Run by hand before the push; this is what catches a sheet overflowing |

No `routeManifest` line, as there is no new route. The `csv` field's description in `routes.mjs`
gains the two optional columns, so `/openapi.json` documents them. `package.json` goes to 3.6.0, with a
Changelog entry and its **Tests:** line.

**The site.** A new `_tests/unit/domain/scoresheet-names.test.mjs`, then `npm run verify`:

- A group side (`A1_1`) gets its name, from a real snapshot's two CSVs.
- A seated playoff side (`Fi-Green_3`) gets its name from its own row.
- A lettered slot whose own row is empty (`SF-A_1`) falls back to team `A`'s name.
- An unseated side (`QF-1_1`) is empty.
- A comma and a quote in a name survive, read back through `parseCSV`.
- Existing cells are preserved, and a missing column returns the input unchanged.

The comparison harness does not cover this page, so the generator's checks are by hand:

| # | Check |
| --- | --- |
| C1 | Ladies' Dink, each of the four layouts: the right names on the right sides, one page per chunk |
| C2 | Ladies' Dink with 2 blank sheets: spare sheets match, with empty name space |
| C3 | Piggleball or the Sportsfest (a standard event): the PDF looks exactly as before |
| C4 | Upload a plain matches CSV: as before |
| C5 | Print one sheet on paper and write a pair's names on the line: is there room? |

---

## 9. Docs to update when it is built

- `features/scoresheet-generator.md`: a team event's sheets print team names, and an uploaded CSV can
  add `teamName1` and `teamName2`.
- `technical/scoresheet-pipeline.md`: its *Where the CSV comes from* says no transformation happens,
  which stops being true for a team event.
- `usage/printables.md`: the Team line under *Where the formats differ*, and printing after seating
  for named playoff sheets.
- This spec's status, by the procedure in the [Specs index](../README.md#moving-a-spec-between-folders).
- `CLAUDE.md`'s scoresheet paragraph in the workspace repo, and the `sage-tools-api` README Changelog.

---

## 10. When to build

No event needs it soon. Ladies' Dink is on 7 November, inside the site's 3-day merge rule, and it is
the last of the three team events registered in `events.json`. The Sportsfest is a
standard event, which this should not affect, but it closes the merge window from 10 to 13 November
and this touches its print path. So: build after 13 November, or before a new team event's 3-day
window if one is registered sooner.

---

## 11. Owner's decisions

All four were answered on 2026-11-06.

| # | Decision | Answer |
| --- | --- | --- |
| D1 | Where the name goes on Standard and Best of 3 (§5). A stacked line would mean shrinking the logo or the scoring area on two layouts that have no space to spare | The same line, inside the underline with the code |
| D2 | Score boxes on **No Referee, Wide**, named sheets only (§5) | 240 → 200 px |
| D3 | What an unseated playoff side prints (§6) | Empty, not `Seed 1 · TBD` |
| D4 | Whether the page has a switch for names. A switch would be one more thing to get wrong while printing | No switch: a team event always gets names |

---
**Related:** [Scoresheet Generator](../../features/scoresheet-generator.md) ·
[scoresheet pipeline](../../technical/scoresheet-pipeline.md) ·
[event picker spec](../implemented/scoresheet-event-picker-spec.md)
