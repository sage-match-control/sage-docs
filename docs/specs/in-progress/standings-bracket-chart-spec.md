# Spec — Playoff bracket chart in the standings

> **Status: in progress.** Built, tested and merged to `main` in `sage-tools-api`,
> `sage-match-control.github.io`, `event-data`, `sage-docs` and the workspace on 22 October 2026
> (the owner's go-ahead, two days before the Millennium Invitational). What is left is §10: the owner
> pasting generator 1.3.0 into the masters and the Sportsfest and Piggleball workbooks and running
> **Add bracket links** in each. §14 records where the build differs from this text.
>
> Written 2026-10-21 against `sage-match-control.github.io` at
> `75c3e42`, `sage-tools-api` at `93e090d`, `event-data` at `c3e0972` and `sage-docs` at
> `77bf767`. The owner set the three-part model in §3 the same day (D1–D3). Every
> worked example in §5 was produced by running the generator's own `planCategory` and
> `playoffBlocks_` at `93e090d`, not worked out by hand.
>
> **Deadlines.** The site repo takes no merge within 3 days of an event. The Millennium
> Invitational is on **Saturday 24 October 2026**, so nothing in this spec reaches the
> site's `main` before **Sunday 25 October**. The Angeles City Employees Sportsfest is on
> **Friday 13 November**, so the site work must reach `main` by **Monday 9 November**, or
> wait until after the event. After 9 November, push to the site only with the owner's
> go-ahead.

Show each category's playoffs as a **bracket chart** (rounds side by side, with lines from
each match to the one its winner plays next) on the Tournament Hub and in Control Center,
beside today's list of matchup cards. On a phone it shows one round at a time, with a
**Full tree** view that fits the whole ladder on screen and zooms.

This spec is written for an implementer with **no other context**. Read §1 first. Every
file, function, value and test is named. If something here disagrees with the code you
find, **stop and ask the owner**. Don't choose for yourself.

---

## 1. Orientation — read this first

### 1.1 The repos

All four live side by side under one workspace folder (`D:\Personal\SAGE` on the owner's
machine). Read the workspace's `CLAUDE.md` before starting.

| Repo | What you change here |
| --- | --- |
| `sage-tools-api/` | **Only `apps-script/standard-tournament-generator.gs`** and its verify script, `apps-script/verify-standard-tournament-generator.mjs` (§5). No change to the Cloud Run service. |
| `sage-match-control.github.io/` | The site engine `lib/v1/` (§7), Control Center `tools/control-center.html` (§7.9), tests under `_tests/` and a fixture under `_fixtures/` (§8) |
| `event-data/` | `config/README.md` documents the new optional setting (§6). No event's entry changes unless the owner asks |
| `sage-docs/` | The pages listed in §9 |

### 1.2 What exists today, in one paragraph

A tournament day runs from a Google Sheets workbook, one per venue per day, which
`sage-tools-api` syncs into a JSON **snapshot** holding two CSV strings per venue: the
`CSV` tab (every match) and the **`STANDINGSCSV`** tab (one row per pair and per playoff
slot). Every page reads that snapshot. The site's pages are plain ES modules with no build
step. The shared code is the **engine** in `lib/v1/`, which has four layers:

- `domain/`: pure rules.
- `data/`: fetching.
- `views/`: HTML-string builders and small components.
- `apps/`: one per page kind.

The Hub (`lib/v1/apps/hub.js`) and Control Center (`tools/control-center.html`) both draw
a standard event's standings with **`createStandings`** in `lib/v1/views/standings.js`.
Each category is one card: its round robin tables, then one block per playoff stage, each
a list of matchup cards. **This spec adds a second way to draw the playoff blocks.**

### 1.3 Rules you must follow

- **`lib/v1` takes compatible changes only.** You may add a module, an export, an option, a
  CSS class or a field. You may not remove or rename one, or change a parameter or return
  shape an existing caller would notice.
  - Within one push, a module may use another module's new export. A module may not start
    using an export that an *earlier* push did not publish, because browsers cache files
    for up to 10 minutes. This spec's site work is one push, so that's fine.
  - See `sage-docs/docs/technical/site-engine.md` § "The versioning rule".
- **Dependency rules**, enforced by `_tests/unit/guards.test.mjs`:
  - `domain/` imports only `domain/`.
  - `views/` imports `domain/` and `views/`.
  - Nothing under `views/` or `domain/` knows about operators: Control Center adds its
    behaviour through an **option that does nothing by default** (§7.9).
- **The theme contract** (`_tests/unit/theme-contract.test.mjs`): engine CSS takes its
  colours only from the shell's custom properties. Use the `--…` names listed in that test.
  The `rgba(…)` literals already used by `.matchup-side.win` and `.matchup-side.loss` in
  `lib/v1/css/standings.css` may be reused as they are.
- **`apps-script/` is not part of the Cloud Run service.**
  - Changing the generator bumps `STANDARD_TOURNAMENT_GENERATOR_VERSION` and its history
    comment, not `package.json`.
  - Run `node apps-script/verify-standard-tournament-generator.mjs` and `npm run verify`
    in `sage-tools-api` before pushing.
  - A push to `sage-tools-api`'s `main` triggers a Cloud Run redeploy of unchanged service
    code. That's harmless, but push only with the owner's go-ahead.
- **The site:**
  - Run `cd _tests && npm install && npm run verify` before every push to `main`.
  - Never merge within 3 days of an event (see the deadlines at the top).
  - Work on a branch. Don't push to `main` without the owner.
- **Docs** are written in the present tense (what the system does), never "now" or "used to".

### 1.4 Read these before writing code

1. `sage-match-control.github.io/lib/v1/views/standings.js`, all of it. The new code sits
   inside `stageTablesHTML`, `categoryCardHTML` and `createStandings`.
2. `lib/v1/domain/standings.js` (`rowsToStandings`, `isEmptyStanding`),
   `lib/v1/domain/codes.js` (`parseCode`, `standingsStageKey`, `STAGE_META`, `STAGE_ORDER`,
   `matchInstanceOf`), `lib/v1/domain/model.js` (`buildDayModel`, the `EventConfig` and
   `DayModel` typedefs) and `lib/v1/data/registry.js` (`eventConfigFrom`, and
   `seedLabelsFrom` as the pattern to copy).
3. `sage-tools-api/apps-script/standard-tournament-generator.gs`:
   - `buildLadder_` (§7.6.1 numbering) and `playoffBlocks_`;
   - `planCategory` and `writeVariablesTab_`;
   - `buildStandingsCsvTab_` and `REBUILT_IN_PLACE_TABS`;
   - `readPlanBlock_` and `onOpen`.
4. `sage-docs/docs/specs/implemented/standard-tournament-master-spec.md` §5.4 (the tiered
   ladder), §7.6 (the qualifier draw), §10.1 (`Variables`) and §10.4 (`STANDINGSCSV`).

---

## 2. What it looks like

### 2.1 A category card, Bracket view (desktop, or any screen the tree fits)

The round robin tables stay at the top of the card. Below them, the playoff stage blocks
are replaced by the chart. The switch sits under the category title.

```
┌──────────────────────── NOVICE WOMEN'S DOUBLES ────────────────────────┐
│                         [ List | Bracket ]                              │
│  Round Robin   (bracket tables, unchanged)                              │
│                                                                         │
│  Round of 16        Quarterfinals       Semifinals          Finals      │
│                                         ┌ SF-1 · Ct 1  ─┐               │
│                                         │ Cruz / Reyes  │─┐             │
│                                         │ Uy / Chua     │ │ ┌ Final ──┐ │
│                                         └───────────────┘ └─│ W SF-1  │ │
│  ┌ R16-3 · Ct 2 ┐   ┌ QF-2 · Ct 2 ┐     ┌ SF-2 ─────────┐ ┌─│ W SF-2  │ │
│  │ Lao / Dizon 11│───│ Lao / Dizon │     │ Dee / Co      │ │ └─────────┘ │
│  │ Go / Sy      8│   │ Ong / Ramos │─────│ Winner of QF-2│─┘             │
│  └──────────────┘   └─────────────┘     └───────────────┘   Bronze      │
│                                                           ┌ L SF-1 ──┐  │
│                                                           │ L SF-2   │  │
│                                                           └──────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

- **Each card:**
  - A head line: the match label (`R16-3`, `QF-2`, `SF-1`, `Final`, `Bronze`), the court
    when there is one, and a status (`Done`, `Live`, or the scheduled time).
  - Then two seat rows: the pair's names and its score.
  - The winning seat is shaded like today's `.matchup-side.win` and the losing seat like
    `.loss`. A match on court has a `--court` outline.
- **A line** joins a match to the seat its winner fills.
- **Bronze** sits under the Final with no lines. Its seats say where they come from.
- **On desktop** the category's column widens to fit the chart (§7.6). The board already
  scrolls sideways.

### 2.2 What an empty seat says

| The seat | Shows |
| --- | --- |
| Has names on the sheet | The names. A seat filled from the round robin also gets a tag such as `Br 2 · #1` (§7.3.6) |
| Empty, filled by a match | `Winner of QF-2`, `Loser of SF-1` |
| Empty, filled from the round robin, and the event names it (§6) | That label, e.g. `Best runner-up` |
| Empty, filled from the round robin, no label | `Qualifier` |
| Empty, and the workbook has no `match`/`from` columns | `TBD` (today's text) |

### 2.3 A phone, or any screen the tree doesn't fit

```
 Standings
 [ Novice women's doubles      ▾ ]
 [ List | Bracket ]
 ( Round of 16 ) ( Quarterfinals ) (•Semifinals•) ( Finals )
 ┌ SF-1 · Ct 1 ─── Live ┐ ┌ Fin…
 │ Cruz / Reyes      9  │ │ Win…
 │ Uy / Chua        10  │ │ Win…
 └──────────────────────┘ └─────
 ┌ SF-2 ────────────────┐
 │ Dee / Co          –  │
 │ Winner of QF-2    –  │
 └──────────────────────┘
                   [ ⤢ Full tree ]
```

- One round per column, in a horizontal scroll-snap track, with the next round peeking in
  at the right edge.
- The chips jump to a round, and the chip of the round in view is highlighted.
- There are no lines here; each seat's text says where it comes from.

### 2.4 Full tree (phones)

```
 ┌───────────────────────────────────────┐
 │ ✕   Novice women's doubles     [Fit]  │
 ├───────────────────────────────────────┤
 │ R16    QF     SF     Final            │
 │               ┌Cruz 9┐                │
 │               └Uy 10┘┐ ┌W SF-1┐       │
 │ ┌Lao 11┐ ┌Lao  ┐┌Dee ┐└─┤W SF-2│       │
 │ └Go  8 ┘─└Ong  ┘└W QF┘   └──────┘       │
 │                          Bronze        │
 └───────────────────────────────────────┘
```

The whole chart covers the screen, scaled to fit. Pinch zooms and dragging pans.
Double-tap zooms in on a point, or back out to fit. Below 0.6× the cards switch to
surnames in a larger type so they stay legible (§7.7).

### 2.5 The three states of one category (the owner's model, §3)

| | Before the round robin ends, no event labels | Same, with the event's labels (§6) | After the round robin, names on the sheet |
| --- | --- | --- | --- |
| R16-3 | `Qualifier` / `Qualifier` | `2nd-best runner-up` / `3rd-best runner-up` | `Ong / Ramos  Br 2 · #2` / `Lao / Dizon  Br 3 · #2` |
| QF-2 | `Winner of R16-3` / `Qualifier` | `Winner of R16-3` / `Best runner-up` | `Winner of R16-3` / `Go / Sy  Br 1 · #2` |
| SF-1 | `Qualifier` / `Qualifier` | `Bracket winner` / `Bracket winner` | `Cruz / Reyes  Br 1 · #1` / `Uy / Chua  Br 3 · #1` |
| SF-2 | `Qualifier` / `Winner of QF-2` | `Bracket winner` / `Winner of QF-2` | `Dee / Co  Br 2 · #1` / `Winner of QF-2` |

---

## 3. Decisions

| # | Decision | By |
| --- | --- | --- |
| D1 | Seating rules change from event to event. The workbook carries only the **shape** of the ladder: which matches exist and which match feeds which seat. It never says which round robin finisher takes a seat | Owner, 2026-10-21 |
| D2 | Where an event seats specific finishers in specific slots, that is **dictated per event**, as text labels in `events.json` (§6). The site displays the labels. It never computes who qualifies | Owner, 2026-10-21 |
| D3 | Once the round robin is done and the qualifiers' names are on the sheet, the site **infers** where each came from by matching names to the round robin (§7.3.6) | Owner, 2026-10-21 |
| D4 | The view defaults to **List**, today's display. One choice applies to every category on the page and is remembered in that browser (`localStorage` key `sage.standings.playoffView`, value `list` or `bracket`, every access in `try`/`catch`) | This spec |
| D5 | The switch appears only on a category with **two or more playoff rounds**, counting Bronze as no round. A category whose playoff is a single final or series keeps today's card with no switch | This spec |
| D6 | **Standard events only** (`event.type === 'standard'`). Dual meets and team events are unchanged | This spec |
| D7 | The whole tree shows in the card when it fits the card's width. Otherwise the card shows one round at a time, with **Full tree**. On the desktop layout it always fits, because the column widens | This spec |
| D8 | Full tree is a fixed-position layer inside the page, **not** the browser's Fullscreen API, which iPhone Safari does not offer for page elements. The phone's Back gesture closes it | This spec |
| D9 | No pair's name is ever shown before the sheet has it. Labels describe a seat; they never predict who will fill it. This keeps the rule in the Hub player experience spec: a playoff slot is never a projection | This spec |
| D10 | The Tournament Calculator is not changed | Owner, 2026-10-21 |
| D11 | Engine pages of finished standard events (they import the same `lib/v1`) get the switch too, in the fallback layout, since their workbooks have no `match`/`from` columns. That is a compatible change, accepted in the harness (§8.4) | This spec |

---

## 4. The data model in one table

| Part | Comes from | When there is none |
| --- | --- | --- |
| Rounds, matches, lines, `Winner of` / `Loser of` | `STANDINGSCSV` columns `match` and `from` (§5) | Fallback: rounds side by side, no lines, empty seats `TBD` |
| Labels on empty round robin seats | `events.json` `display.playoffSeats` (§6) | `Qualifier` (or `TBD` in the fallback) |
| Source tags after the round robin | Names already in `STANDINGSCSV`, matched to the round robin rows (§7.3.6) | No tag |

---

## 5. Workbook: two new `STANDINGSCSV` columns

### 5.1 The columns

`STANDINGSCSV` keeps its seven columns `A:G` exactly as they are. The generator adds two:

| Col | Header | Value on a playoff slot's row | On a round robin row |
| --- | --- | --- | --- |
| `H` | `match` | The **block label** of the match this slot plays in, exactly as the category tab's playoff block reads: `R16-3`, `QF-2`, `SF-1`, `F`, `B`, `F (1)`, `F (2)`, `F (3)` | empty |
| `I` | `from` | `W:<block label>` when the winner of that match fills this slot, `L:<block label>` when its loser does, and **empty** when the slot is filled from the round robin (the draw, a fixed seat or a bye) | empty |

**Why `match` and not seat numbers.** The generator's seat numbering (master spec §7.6.1)
doesn't always pair seats as `(2j-1, 2j)`. With 3 brackets, top 1, the semifinal is seats
`2` and `3`. So the site must never pair seats by arithmetic; it groups them by `match`.

Every reader of `STANDINGSCSV` finds its columns by header name, so the two extra columns
break nothing. These are the readers checked at the commits above:

- the site's `rowsToStandings`;
- `sage-tools-api`'s `SheetsCsvFetcher`/`GvizCsvFetcher`, which read the whole tab;
- `src/attendance/domain/roster.mjs` (`headerIndex`);
- `src/sync/domain/facilityCompletion.mjs`.

**No `sage-tools-api` service change and no `package.json` bump.**

### 5.2 Worked examples (generator output at `93e090d`)

Each row below is one playoff slot: its code (column `A`), then `match`, then `from`.

**3 brackets, top 2** (`HIMD`, 13 pairs 5-4-4: the tiered stepladder)

| A | match | from |
| --- | --- | --- |
| `HIMD_R16_5`, `HIMD_R16_6` | `R16-3` | |
| `HIMD_QF_3` | `QF-2` | `W:R16-3` |
| `HIMD_QF_4` | `QF-2` | |
| `HIMD_SF_1`, `HIMD_SF_2` | `SF-1` | |
| `HIMD_SF_3` | `SF-2` | |
| `HIMD_SF_4` | `SF-2` | `W:QF-2` |
| `HIMD_B_1` | `B` | `L:SF-1` |
| `HIMD_B_2` | `B` | `L:SF-2` |
| `HIMD_F_1` | `F` | `W:SF-1` |
| `HIMD_F_2` | `F` | `W:SF-2` |

**4 brackets, top 2** (`XD`, 20 pairs)

- `XD_QF_1`…`XD_QF_8`: `match` `QF-1` (seats 1, 2), `QF-2` (3, 4), `QF-3` (5, 6),
  `QF-4` (7, 8); `from` empty.
- `XD_SF_1` `SF-1` `W:QF-1`, `XD_SF_2` `SF-1` `W:QF-2`, `XD_SF_3` `SF-2` `W:QF-3`,
  `XD_SF_4` `SF-2` `W:QF-4`.
- `XD_B_1` `B` `L:SF-1`, `XD_B_2` `B` `L:SF-2`, `XD_F_1` `F` `W:SF-1`, `XD_F_2` `F` `W:SF-2`.

**2 brackets, top 2** (`MD`, 10 pairs)

- `MD_SF_1`, `MD_SF_2`: `SF-1`, `from` empty. `MD_SF_3`, `MD_SF_4`: `SF-2`, `from` empty.
- `MD_B_1` `B` `L:SF-1`, `MD_B_2` `B` `L:SF-2`, `MD_F_1` `F` `W:SF-1`, `MD_F_2` `F` `W:SF-2`.

**3 brackets, top 1** (`T3`, 12 pairs: a seat in the Final filled straight from the round robin)

- `T3_SF_2`, `T3_SF_3`: `SF-1`, `from` empty.
- `T3_F_1`: `F`, `from` empty. `T3_F_2`: `F`, `W:SF-1`.
- No bronze, because there are fewer than 4 qualifiers.

**1 bracket, twice-to-beat** (`ONE`, 6 pairs)

- `ONE_F_1_(1)`, `ONE_F_2_(1)`: `F (1)`. `ONE_F_1_(2)`, `ONE_F_2_(2)`: `F (2)`.
- `ONE_B_1`, `ONE_B_2`: `B`. `ONE_B_2` is the bye.
- Every `from` is empty.
- This category has one playoff round, so it gets no switch (D5). The columns are still
  written.

### 5.3 The generator change (`standard-tournament-generator.gs`)

1. **The playoff link table on `Variables`.**
   - In `writeVariablesTab_`, after the plan block, write a three-column table at
     **`Variables!AD:AF`**: header row 1 `playoff code`, `match`, `from`, then one row
     per playoff seat of every category in `plans`, from row 2.
   - Category by category in `plans` order, seats in `playoffBlocks_(p)` order, each
     block's seats in their order. For each block `b` and seat `s`, write `s.code`,
     `b.label`, then:
     - `'W:' + s.from.label` when `s.from`;
     - otherwise `'L:' + s.loserFrom.label` when `s.loserFrom`;
     - otherwise `''`.
   - First clear `AD2:AF` to the sheet's last row (a master may carry a previous run's
     table). Call `ensureColumnCount_(sheet, COL.AF)`, adding `AF: 32` to the `COL`
     constant.
   - Before writing, check that the master's `Variables` holds nothing in `AD:AF` (§10.1
     of the master spec lists `A:C`, `H:R` and `AB` only). If it does, stop and ask the
     owner.
2. **`STANDINGSCSV!H:I`.** In `buildStandingsCsvTab_`:
   - Write the header `['teamCode', 'player1', 'player2', 'wins', 'loss', 'quotient',
     'bracket', 'match', 'from']`. Change `STANDINGSCSV_HEADERS_` and the
     `getRange(1, 1, 1, 7)` width to 9.
   - Extend each body row with two formulas:
     - `H<r>`: `=IFNA(VLOOKUP($A<r>, Variables!$AD:$AF, 2, FALSE), "")`
     - `I<r>`: `=IFNA(VLOOKUP($A<r>, Variables!$AD:$AF, 3, FALSE), "")`
   - So the body range becomes `getRange(2, 2, body.length, 8)`. Call
     `ensureColumnCount_(sheet, 9)` first.
   - `REBUILT_IN_PLACE_TABS`'s `maxCol: 7` for `STANDINGSCSV` **stays 7**: it describes
     the master's untouched prototype, which still has seven columns. A built tab has nine,
     so a rerun is still refused.
3. **Version.** `STANDARD_TOURNAMENT_GENERATOR_VERSION = '1.3.0'`. Add the history line:
   `1.3  STANDINGSCSV match and from columns (the bracket chart); SAGE -> Add bracket links`.
4. **`SAGE → Add bracket links`** (§5.4). Add it to the menu in `onOpen`, after
   **Import bracket draws**.

### 5.4 `SAGE → Add bracket links`, for a workbook generated before 1.3.0

The Angeles City Sportsfest workbook (`18UH-CHx4_qIdI5y89qrSKDQR0Cd0dzKWQRQfKunNFjg`) was
generated with 1.2.0 and can't be regenerated: the generator refuses to run twice. This
menu item adds the two columns to an existing generated workbook. It is **idempotent**:
running it again rewrites the same values.

1. Read the plan block like `readPlanBlock_`, but read **`Variables!I:N`** from row 11:
   key, value, teams, brackets, advance and fill. Keep only rows whose category tab exists
   (`hasTab`).
2. For each such category, call `planCategory({ key, value, teams, brackets, advance, fill,
   soloFormat })`.
   - `soloFormat` matters only for a one-bracket category. Use `'ttb'` when
     `STANDINGSCSV!A:A` contains `<key>_F_1_(1)`, else `'rr'`.
3. **Safety check.** Collect every playoff seat code `planCategory` yields, and every code
   in `STANDINGSCSV!A2:A` that starts with `<key>_` and is not a round robin code
   (`/^<key>_\d+$/`). Do the two sets differ for any category? Then write nothing at all.
   Show an alert naming each category and its first five missing or extra codes. That means
   the workbook was edited by hand, and the owner decides.
4. Otherwise:
   - Write `Variables!AD:AF` exactly as §5.3 step 1.
   - Write `STANDINGSCSV!H1:I1` = `match`, `from` and the two formulas of §5.3 step 2 on
     every row from 2 to the tab's last row.
   - Show an alert: `Bracket links added for <n> categories (<keys>).`
5. Log each category's seat lines with the existing `ladderTraceLines_`.

Don't touch any other cell. Don't run inside `generateEventTabs`. In a 1.3.0 workbook the
values are already there and the menu item just rewrites them.

### 5.5 A hand-built workbook (optional, documented only)

A workbook not made by the generator can get the same columns by hand:

1. Type the `match` and `from` values (§5.1) into a spare two-or-three-column range.
2. Look them up from `STANDINGSCSV!H:I` the same way.

Document this in the features page (§9). Nothing in code depends on it.

### 5.6 The verify script (`verify-standard-tournament-generator.mjs`)

Add checks to the Annex run (the `pfs-annex-plan.csv` fixture, which has `HIMD`):

- `STANDINGSCSV!H1:I1` = `match`, `from`.
- `STANDINGSCSV!H2` and `I2` hold the two formulas of §5.3, as text (the mock does not
  evaluate formulas).
- `Variables!AD1:AF1` = `playoff code`, `match`, `from`.
- The `Variables!AD:AF` rows for `HIMD` equal the 12 rows of §5.2's first table, in that
  order.

Add a pure check on a hand-made category for each §5.2 shape (`XD`, `MD`, `T3`, `ONE`). It
compares `planCategory` + `playoffBlocks_` against the tables above, through the same
function the writer uses: factor the row-building into `playoffLinkRows_(p)` →
`[[code, match, from], …]` and test that.

Add a test of **Add bracket links** on a mocked generated workbook:

- It writes the expected rows.
- It refuses (writes nothing) when one `STANDINGSCSV` code is renamed.
- A second run leaves identical values.

Run `node apps-script/verify-standard-tournament-generator.mjs`, then `npm run verify`.

---

## 6. `events.json`: `display.playoffSeats` (optional)

Shape, under an event's `display`:

```json
"display": {
  "divisions": { "": "" },
  "events": { "MD": "Men's Doubles" },
  "playoffSeats": {
    "HIMD": {
      "SF_1": "Bracket winner", "SF_2": "Bracket winner", "SF_3": "Bracket winner",
      "QF_4": "Best runner-up",
      "R16_5": "2nd-best runner-up", "R16_6": "3rd-best runner-up"
    },
    "2x2": {
      "SF_1": "Bracket 1 · 1st", "SF_2": "Bracket 2 · 2nd",
      "SF_3": "Bracket 2 · 1st", "SF_4": "Bracket 1 · 2nd"
    }
  }
}
```

- **Outer key:** a **category code** (the `category` part of `parseCode(code, 'standard')`,
  e.g. `HIMD`, or `MD` in an event with no division), or a **shape key**
  `<brackets>x<advance>` (§7.3.5). A category's own entry wins over its shape's, per seat.
- **Inner key:** the slot code without its category: `parseCode(code, 'standard').rest`,
  e.g. `SF_1`, `QF_4`, `R16_5`. It must match `/^(R32|R16|QF|SF|F|B)_\d+$/`.
- **Value:** a non-empty label, shown as is on that seat while it is empty **and** its
  `from` is empty (or the workbook has no `from` column).
- Anything malformed is dropped with a `console.warn`, the way `seedLabelsFrom` does it.
  The rest of the setting still applies.
- No event needs this setting. Without it, open seats read `Qualifier`. `sage-tools-api`
  does not read `display` and does not validate it, so no service change is needed.

Document it in `event-data/config/README.md` beside `seedLabels`, and in
`sage-docs/docs/technical/event-data-config.md` (§9). **Don't add it to any event's entry**
unless the owner gives the labels.

---

## 7. Site engine

### 7.1 `domain/standings.js`: two new row fields

`rowsToStandings` reads two more headers, `match` and `from`, trimmed, `''` when the column
is absent. Every row gets `match` and `from`. Update the shape comment at the top:
`Standings row = { …, match, from }`. Nothing else changes.

### 7.2 `data/registry.js`: `playoffSeats`

Add `playoffSeatsFrom(raw, eventKey)`, written like `seedLabelsFrom`. It returns `{}`
without the setting, and `{ [outer]: { [inner]: label } }` holding only the valid entries
(§6). `eventConfigFrom` returns `playoffSeats: playoffSeatsFrom(raw.display &&
raw.display.playoffSeats, eventKey)`. Add the property to the `EventConfig` typedef in
`domain/model.js`:

```js
@property {Record<string, Record<string,string>>} [playoffSeats]  a standard event's labels for empty playoff seats, by category code or shape key, then slot (display.playoffSeats; {} without it)
```

### 7.3 New module `domain/bracket.js` (pure)

Imports only from `domain/`: `codes.js`, `standings.js`, `byes.js`, `awards.js`. Everything
below is exported, with a JSDoc block on each.

#### 7.3.1 `parseFrom(text)`

`'W:QF-2'` → `{ kind: 'W', match: 'QF-2' }`, and `'L:SF-1'` → `{ kind: 'L', match: 'SF-1' }`.
`''`, `null` or anything else → `null`. Trim, and accept only `/^([WL]):(.+)$/`.

#### 7.3.2 `matchDisplayLabel(id)`

`'F'` → `'Final'`, `'B'` → `'Bronze'`, `'F (2)'` → `'Final · Match 2'` (the wording
`roundLabel` already uses for a series game), and anything else as is (`'QF-2'`).

#### 7.3.3 `rankByWinsThenQuotient(rows)`

A stable copy sorted by `wins` descending, then `quotient` descending (`null` lowest), the
sheet's own ordering. It deliberately ignores head-to-head, which `rankStandings` uses: a
tag must say the finish the sheet seated the pair by.

#### 7.3.4 `buildBracket(rows, model)`

`rows` are **all** of one category's standings rows (what `categoryCardHTML` receives). It
returns:

```js
{
  linked: boolean,          // every playoff row has a non-empty `match`
  category: string,         // parseCode(firstPlayoffRow.teamCode, 'standard').category
  rounds: [                 // earliest → latest, BRONZE excluded, in sortStageKeys order
    { key: 'R16'|'QF'|'SF'|'FINAL'|…, label: STAGE_META[key].label, matches: [Node] }
  ],
  bronze: Node | null,
  eligible: boolean,        // rounds.length >= 2  (D5)
}
```

A `Node` is `{ id, stageKey, games: [Game] }`. It has more than one `Game` only for a series
final (`F (1)`, `F (2)`, …). A `Game` is:

```js
{
  id,        // the `match` value, or a made-up id in the fallback: `${stageKey}-${i+1}` (or `${stageKey}-${i+1} (${g})`)
  m,         // model.matchByCode.get(seat code of either seat) — the scheduled Match, or null
  notNeeded, // !!m && model.unneededGames.has(m)
  seats: [Seat, Seat],
}
```

A `Seat` is:

```js
{
  row,       // the standings row (may be undefined: a game with one row)
  code, rest,
  from,      // parseFrom(row.from), or null
  empty,     // !row || isEmptyStanding(row)
  result,    // 'win' | 'loss' | ''  (from m: played and unequal scores)
  score,     // m.t1Score / m.t2Score for this seat when m.played, else null
}
```

**Which rows count.** Rows whose `standingsStageKey(code, 'standard') !== 'RR'`, minus
`isByeStandingRow` rows. If `categoryBronzeIsByeDecided(model, category)`, drop the BRONZE
rows: that's the same rule as `stageTablesHTML`.

**Linked grouping.** Group rows by `row.match` in row order.

- In each group, order the seats by the seat number in the code: the first integer after
  the stage keyword, `/_(?:R32|R16|QF|SF|F|B)_(\d+)/`.
- A group with one row gets an undefined second seat. That happens with hand-edited data;
  the Seat then has `empty: true`.
- Groups whose id matches `/^(.+) \((\d+)\)$/` and share the first capture form one series
  `Node`, with games in game order. Every other group is a `Node` of one game, whose id is
  the group's `match`.
- A node's `stageKey` is `standingsStageKey` of its first seat's code.

**Fallback grouping** (not linked). Per stage, `pairUpMatchups(stageRows)`, the existing
function, which stays in `views/standings.js`.

- It is not importable from `domain/`. So **move its body** into `domain/bracket.js` as
  `pairPlayoffRows(rows)`, and keep `pairUpMatchups` in `views/standings.js` as a one-line
  wrapper calling it. The export stays, and its behaviour is identical.
- Each pair is a Game. Series games are pairs whose codes carry `matchInstanceOf`. In the
  FINAL stage they form one Node, in instance order.

**Node order within a round.**

- **Linked:** by the number after the last `-` in the id (`QF-2` → 2), then id text.
- **Fallback:** in the order the pairs arrived.

**Score and result.** For each game, `m` is the Match whose `t1` or `t2` equals a seat's
code. A seat's score is `m.t1Score` when `m.t1 === seat.code`, else `m.t2Score`, and only
when `m.played`. When played with unequal scores, the higher score's seat is `'win'` and
the other `'loss'`.

#### 7.3.5 `bracketShapeKey(bracket, rows)`

`null` when not `linked`. Otherwise:

- `groups` = the number of distinct non-empty `bracket` values among the category's round
  robin rows (at least 1).
- `fresh` = the number of seats, over all nodes and the bronze, whose `from` is `null`.
- Return `${groups}x${fresh / groups}` when that divides evenly, else `null`.

For §5.2: `HIMD` → `3x2`, `XD` → `4x2`, `MD` → `2x2`, `T3` → `3x1`.

#### 7.3.6 `seatText(seat, ctx)`

`ctx = { playoffSeats, category, shapeKey, rrRows, linked }`. It returns
`{ kind, names, label, tag }`, where `kind` is `'pair' | 'winner' | 'loser' | 'label' | 'qualifier' | 'tbd'`.
Apply the first case that fits:

1. **`!seat.empty`** → `kind 'pair'`, with:
   - `names: [player1, player2]`;
   - `tag: sourceTag(seat.row, rrRows)` **only when `seat.from === null`**, else `''`.
2. **`seat.from`** → `kind 'winner'` or `'loser'`, `label: 'Winner of ' +
   matchDisplayLabel(from.match)` (or `'Loser of …'`).
3. **A label exists**: `playoffSeats[category]?.[seat.rest] ?? playoffSeats[shapeKey]?.[seat.rest]`
   → `kind 'label'`, `label`.
4. **`linked`** → `kind 'qualifier'`, `label: 'Qualifier'`.
5. Otherwise → `kind 'tbd'`, `label: 'TBD'`.

**`sourceTag(row, rrRows)`.** `rrRows` are the category's round robin rows that are not
empty and not BYE.

- Normalise a name: lower-case it, collapse runs of whitespace to one space, and trim.
- A pair's key is its two normalised names, **sorted**, joined with `|`.
- Find the round robin rows with the slot's key. If there is not exactly one, return `''`.
- Otherwise take its `bracket` `b`. Its rank is its 1-based position in
  `rankByWinsThenQuotient` of the round robin rows sharing `b`.
- Return `` `Br ${b} · #${rank}` ``, or `` `#${rank}` `` when `b` is empty.

#### 7.3.7 `layoutBracket(bracket, dims)`

Pure geometry. `dims = { cardW, colGap, headH, seatH, rowGap, captionH, bronzeGap,
seriesGap }`. It returns:

```js
{
  width, height,
  captions: [{ text, x, y }],                 // one per round, at y = 0, plus 'Bronze Battle' above the bronze
  nodes: [{ node, x, y, h }],                 // h = games.length * cardH + (games.length - 1) * seriesGap
  lines: [{ d }],                             // SVG path data, linked only
}
```

Here `cardH = headH + 2 * seatH`. Round `i`'s `x = i * (cardW + colGap)`, and
`width = rounds.length * cardW + (rounds.length - 1) * colGap`.

**Linked:**

1. `feeders(N)`: for each seat index `k` (0 or 1) of the node's **first game**, whose
   `from.kind === 'W'` and whose `from.match` is the id of a game of some node `F` → `{ k,
   F }`, in `k` order.
2. `cursor = captionH`, `y` unset everywhere. `place(N)`:
   - Return if `N` is placed; mark it placed; `place(F)` for each feeder, in order.
   - **No feeders:** `N.y = cursor`.
   - **One `{k, F}`:** `N.y = F.y + F.h/2 - (headH + k*seatH + seatH/2)`, so `F`'s middle
     lines up with the seat it feeds.
   - **Two:** `N.y = mean(F.y + F.h/2) - N.h/2`.
   - Then `cursor = max(cursor, N.y + N.h + rowGap)`.
3. Call `place` on the last round's nodes in order. Then, for every round from first to
   last, on its unplaced nodes in order: matches the Final can't reach.
4. **Overlaps:** in each round, sort by `y`; any node closer than `prev.y + prev.h +
   rowGap` moves down to that.
5. **Normalise:** shift every `y` by `captionH - min(y)` when that is positive.
6. **Bronze:** at the last round's `x`, `y = max(bottom of every node) + bronzeGap`. Its
   caption `Bronze Battle` sits at `y - captionH + 4`.
7. **Lines:** for each node `N` and feeder `{k, F}`: `x1 = F.x + cardW`, `y1 = F.y + F.h/2`,
   `x2 = N.x`, `y2 = N.y + headH + k*seatH + seatH/2`, `xm = x1 + colGap/2`, and
   `d = M x1 y1 H xm V y2 H x2`. Round to whole pixels.
8. `height = max(bottom of every node, the bronze included)`.

**Fallback:** each round's nodes are stacked from `captionH` with `rowGap`. Each column is
then centred vertically on the tallest column. No lines. The bronze is placed as in linked
step 6.

### 7.4 New module `views/bracket.js`

Imports `domain/` and `views/html.js` only.

```js
export const BRACKET_DIMS = { cardW: 172, colGap: 36, headH: 20, seatH: 30, rowGap: 16,
                              captionH: 22, bronzeGap: 34, seriesGap: 8 };
```

#### `bracketChartHTML(bracket, model, { decorate } = {})`

It calls `layoutBracket(bracket, BRACKET_DIMS)` and returns `{ html, width, height }`.

```html
<div class="bk" style="width:Wpx;height:Hpx" data-bk-cat="<category>">
  <svg class="bk-lines" width="W" height="H" aria-hidden="true"><path d="…"/>…</svg>
  <div class="bk-caption" style="left:Xpx;top:Ypx">Quarterfinals</div>
  <div class="bk-node" style="left:Xpx;top:Ypx">            <!-- one per node; a series stacks its games -->
    <div class="bk-card[ bk-card--final][ bk-card--live][ not-needed][ <decorate className>]"
         data-bk-match="QF-2"<decorate attrs>>
      <div class="bk-head"><span class="bk-label">QF-2 · Court 3</span><span class="bk-status">Done</span></div>
      <div class="bk-seat[ win| loss][ bk-seat--open]" title="<full names>">
        <span class="bk-names"><span class="bk-full">Maria Cruz / Ana Reyes</span><span class="bk-short">Cruz / Reyes</span></span>
        <span class="bk-tag">Br 1 · #1</span>
        <span class="bk-score">11</span>
      </div>
      …second seat…
    </div>
  </div>
</div>
```

- **Head:** `matchDisplayLabel(game.id)`, then ` · <m.court>` when the match has a court
  assignment. In the fallback, the head is `roundLabel(rest)` from `domain/codes.js`
  (`Quarterfinal`, or `Final · Match 2` for a series game).
- **Status:**
  - `Not needed` when `game.notNeeded`;
  - else `Done` when `m.played`;
  - else `Live` when `m.liveCourt`;
  - else `m.time`;
  - else empty.
- **Classes:**
  - `bk-card--live` while `m.liveCourt` and not played;
  - `bk-card--final` on the FINAL node's cards;
  - `not-needed` mirrors `.matchup.not-needed`.
- **A seat:**
  - `kind 'pair'`: `bk-full` holds `player1 / player2`, and `bk-short` holds each name's
    last whitespace-separated word joined by ` / `. Add a `bk-tag` span only when the tag
    is non-empty.
  - Any other kind: one span with the label, and the seat gets `bk-seat--open`.
  - **Score:** the number, or `–`.
- Escape every text with `escapeHtml`.
- **`decorate(m)`** (optional): Control Center's hook, the same contract as the ticket's
  `decorate`. It returns `{ className?, attrs? }` for a game with a scheduled Match `m`, and
  is never called when `m` is null.

#### `roundStripHTML(bracket, model, { decorate } = {})`

```html
<div class="bk-rounds" data-bk-cat="<category>">
  <div class="bk-chips" role="tablist">
    <button type="button" class="bk-chip active" data-bk-round="0" role="tab" aria-selected="true">Round of 16</button>…
  </div>
  <div class="bk-track">
    <div class="bk-round" data-bk-round="0"> …the round's cards, same card markup, in normal flow… </div>…
    <div class="bk-round" data-bk-round="last"> …Final, then "Bronze Battle" caption and the bronze card… </div>
  </div>
  <button type="button" class="bk-full-open" data-bk-full="<category>">Full tree</button>
</div>
```

The first chip is active on render. **Which round to start on:** the earliest round with an
unplayed game whose two seats are both `kind 'pair'`. If there is none, the last round with
a played game; if there is none of either, round 0. `createStandings` scrolls the track to
it after rendering (§7.6).

### 7.5 CSS: append to `lib/v1/css/standings.css`

The new rules go in the file every standings page already links, so no shell changes. Add
them under a heading comment `/* ---- bracket chart (standings-bracket-chart-spec) ---- */`.
Use only theme properties:

- `.bk`: `position:relative`.
- `.bk-lines`: `position:absolute; inset:0`; paths `fill:none; stroke:var(--line);
  stroke-width:1.5`.
- `.bk-caption`: `position:absolute`, the `.stage-caption` type (Barlow Condensed 10.5px
  uppercase, `var(--muted)`).
- `.bk-node`: `position:absolute; width:172px`.
- `.bk-card`:
  - `box-sizing:border-box; height:80px; background:var(--white); border-radius:10px;
    box-shadow:var(--card-shadow); overflow:hidden`.
  - No border, so the geometry stays exact.
  - Inside a series node, `margin-bottom:8px` except the last.
- `.bk-head`: `height:20px`, the `.matchup-label` type, `display:flex;
  justify-content:space-between; padding:0 8px; align-items:center`.
- `.bk-card--final .bk-head`: `background:var(--navy); color:var(--white)`.
- `.bk-seat`:
  - `height:30px; display:flex; align-items:center; gap:6px; padding:0 8px;
    border-top:1px solid var(--line)`, plus the `.mu-pair` type (Inter 700 11.5px
    `var(--ink)`).
  - `.bk-seat.win` and `.bk-seat.loss` take the same backgrounds as `.matchup-side.win`
    and `.matchup-side.loss`.
- `.bk-names`: `flex:1; min-width:0; white-space:nowrap; overflow:hidden;
  text-overflow:ellipsis`.
- `.bk-short`: `display:none`.
- `.bk-seat--open .bk-names`: `color:var(--muted); font-style:italic; font-weight:600`.
- `.bk-tag`: 10px, `var(--muted)`, `background:var(--paper-dim)`, radius 4px, padding
  `0 4px`.
- `.bk-score`: `font-family:'Archivo Black'; min-width:18px; text-align:right`.
- `.bk-card--live`: `outline:2px solid var(--court); outline-offset:-2px`.
- `.bk-card.not-needed`: the same treatment as `.matchup.not-needed` in `css/ticket.css`
  (`opacity:.4; filter:grayscale(1)`). Put the rule in `standings.css`; don't edit
  `ticket.css`.
- **Switch:**
  - `.standings-view-switch`: centred under the title, `margin-top:6px`.
  - `.standings-view-btn`: the `.bracket-cols-toggle` look, but in normal flow, not
    absolute.
  - `[aria-pressed="true"]`: `background:var(--navy); color:var(--white)`.
- **Desktop column:**

  ```css
  @media (min-width:900px) and (min-height:600px) {
    .standings-board-desktop .standings-col.standings-col--bracket{ flex:0 0 var(--bk-col-w); max-width:none; }
  }
  ```

  This beats the shells' own `.standings-board-desktop .standings-col` rule on specificity.
- **Round strip:**
  - `.bk-chips`: a horizontally scrolling flex row, `gap:6px`, chips with the toggle look.
  - `.bk-track`: `display:flex; gap:12px; overflow-x:auto; scroll-snap-type:x mandatory;
    -webkit-overflow-scrolling:touch`.
  - `.bk-round`: `flex:0 0 82%; scroll-snap-align:start`.
  - `.bk-rounds .bk-card`: `position:relative; margin-bottom:10px`.
  - `.bk-full-open`: right-aligned, the toggle look.
- **Full tree:** see §7.7.

### 7.6 `views/standings.js`: integration

All additions are optional. With `view: 'list'` everything renders exactly as today.

1. **`stageTablesHTML(rows, model, opts)`** gets two new options:
   - **`view` (`'list'` default):** when `'bracket'` and the event is standard and
     `buildBracket(rows, model).eligible`, it emits the round robin section as today, then
     the bracket section **instead of** the stage blocks.
   - **`availableWidth` (default `Infinity`):** the bracket section is
     `bracketChartHTML(...).html` when its `width <= availableWidth`, else
     `roundStripHTML(...)`.
   - It returns `{ html, hasMultiBracket, bracketWidth }`, where `bracketWidth` is the
     chart's width when a chart was emitted, else `0`. That's an added field, which is
     compatible.
2. **`categoryCardHTML(cat, rows, model, opts)`** gets new options `view`,
   `availableWidth` and `decorateBracketMatch`, passed through.
   - When the category is eligible, the head gets the switch after the title:

     ```html
     <div class="standings-view-switch" role="group" aria-label="Playoff view">
       <button type="button" class="standings-view-btn" data-standings-view="list" aria-pressed="true|false">List</button>
       <button type="button" class="standings-view-btn" data-standings-view="bracket" aria-pressed="false|true">Bracket</button>
     </div>
     ```

   - When a chart was emitted with `bracketWidth + 28 > 360`, the column gets
     `standings-col--bracket` and `style="--bk-col-w:<bracketWidth + 28>px"`.
3. **`createStandings(options)`:**
   - **New option `decorateBracketMatch`.** Control Center's hook (§7.9); none on the Hub.
   - **`state.view`:** read once from `localStorage['sage.standings.playoffView']`, in a
     `try`/`catch`; anything but `'bracket'` reads `'list'`.
   - **Clicks.** A click on `[data-standings-view]` in `bodyEl` sets `state.view`, stores
     it (`try`/`catch`) and calls `render()`.
   - **`cardOptions()`** adds `view`, `decorateBracketMatch` and `availableWidth`:
     - `Infinity` when `isDesktopStandings()`;
     - else the category body's width, measured as `bodyEl.clientWidth - 28` (the card's
       14px side padding).
   - **After every render:**
     - For each `.bk-rounds` in `bodyEl` whose category has **not** shown a round strip
       before (keep a `Set` of categories in `state`), scroll its `.bk-track` to the start
       round of §7.4 (`track.scrollLeft = roundEl.offsetLeft`). Every later render keeps
       where the viewer scrolled (next point). Then mark the chip of the round in view
       active.
     - Chips scroll their track on click (`scrollTo({ left, behavior: 'smooth' })`).
     - On `scroll`, debounced 80 ms, the chip of the round nearest the left edge becomes
       active.
   - **Resize.** Extend the existing resize handler. Outside the desktop layout, when
     `state.view === 'bracket'` and `bodyEl.clientWidth` changed since the last render,
     call `render()`: a rotated phone or tablet may now fit the tree, or stop fitting it.
   - **`[data-bk-full]`** opens Full tree (§7.7) for that category.
   - **Scroll positions.** `render()` already wraps the redraw in `captureScrollPositions`
     / `restoreScrollPositions` (`views/scroll.js`), which only restore nodes matching
     `SCROLLABLE_SELECTOR`. Add `, .bk-track` to the end of that exported string (an
     added selector is a compatible change), so a snapshot arriving doesn't jump the strip
     back.
     - `renderBody()` (a category picked on a phone, or a chip toggled) doesn't capture
       today. Leave that as it is.

### 7.7 New module `views/bracket-full.js`: Full tree

`createFullTree({ hostEl })` returns `{ open(cat, title), update(html, width, height),
close(), isOpen(), category() }`. `createStandings` owns one, with
`hostEl = options.panelEl`.

- It opens from `[data-bk-full]` with the category's `bracketChartHTML` output.
- On every `render()` while it's open, `createStandings` calls `update(...)` with the
  category's fresh chart. A snapshot arriving keeps the tree live.

Markup, appended to `hostEl` on open and removed on close:

```html
<div class="bk-full" role="dialog" aria-modal="true" aria-label="<title> bracket">
  <div class="bk-full-bar">
    <button type="button" class="bk-full-close" aria-label="Close">✕</button>
    <span class="bk-full-title"><title></span>
    <button type="button" class="bk-full-fit">Fit</button>
  </div>
  <div class="bk-full-stage"><div class="bk-full-canvas">…chart…</div></div>
</div>
```

CSS:

- `.bk-full`: `position:fixed; inset:0; height:100dvh; z-index:1000;
  background:var(--paper); display:flex; flex-direction:column`.
- `.bk-full-bar`: a 48px flex row with `var(--white)` background and a bottom
  `var(--line)` border.
- `.bk-full-stage`: `flex:1; position:relative; overflow:hidden; touch-action:none`.
- `.bk-full-canvas`: `position:absolute; left:0; top:0; transform-origin:0 0;
  padding:16px`.
- **Compact:**
  - `.bk-full--compact .bk-full` and `.bk-full--compact .bk-tag`: `display:none`.
  - `.bk-full--compact .bk-short`: `display:inline; font-size:20px`.
  - `.bk-full--compact .bk-head`: `font-size:16px`.

Behaviour:

- **Fit.** `fit = min(stageW / (W + 32), stageH / (H + 32), 1)`. Opening shows the tree at
  `fit`, centred.
- **Transform.** `translate(tx, ty) scale(s)`, with `s` clamped to `[fit, 2.5]`. Clamp
  `tx`/`ty` so at least 80px of the canvas stays on screen on each axis.
- **Gestures.** Use pointer events on `.bk-full-stage`:
  - **One pointer drags** to pan.
  - **Two pointers pinch,** zooming about their midpoint.
  - **`wheel` with `ctrlKey`** zooms about the cursor (desktop testing and trackpads).
  - **Double-tap** (two `pointerup`s within 300 ms and 24 px) or **`dblclick`**: from
    `s < 1`, zoom to `1` about the tap; else back to `fit`.
  - **A tap is a tap.** A `pointerup` that moved under 6px from its `pointerdown` is left
    alone, so a click reaches a card (Control Center's score click, §7.9).
- **Compact mode.** `bk-full--compact` is on the `.bk-full` root whenever `s < 0.6`.
- **The Fit button** returns to `fit`.
- **Resize** (`resize` and `orientationchange`): recompute `fit`. If the view was at `fit`,
  stay at the new `fit`; else keep `s` and clamp.
- **Back closes it.**
  - On open: `history.pushState({ sageBracketFull: true }, '')`.
  - A `popstate` while open closes it without touching history.
  - The close button and `Escape` call `history.back()` when this layer pushed the entry,
    so `popstate` closes it; otherwise they close directly.
- **Scroll lock.** On open, set `document.documentElement.style.overflow = 'hidden'`, and
  restore the previous value on close.
- **Focus.** On open, focus moves to the close button. On close, it goes back to the
  element that opened the layer.
- **On `update`:** replace the canvas's inner HTML and keep `s`/`tx`/`ty`. If the chart's
  size changed, recompute `fit` and clamp.

### 7.8 The Hub

`lib/v1/apps/hub.js` needs **no change**: it already builds standings with
`createStandings`, and the defaults give the Hub its behaviour. Check that a standard
event's Hub shows the switch with no change to any shell.

### 7.9 Control Center (`tools/control-center.html`)

1. **Pass the hook.** Where `createStandings({...})` is called (the block commented
   "Standings: lib/v1/views/standings.js draws…"), add:

   ```js
   decorateBracketMatch: m => { const attrs = scoreAttrs(m); return attrs ? { className: ' scoreable', attrs } : {}; },
   ```

2. **Listen for it.** Add click and keydown listeners on `standingsResultsEl`, identical to
   the two on `resultsEl` (they call `openScoreFromFinder(facility, num)`).
   - `standingsResultsEl` is the `panelEl`, so this also catches clicks inside Full tree.
   - Check that no other `[data-score-num]` element sits in the standings panel today. At
     `75c3e42` there is none.
3. **Show the hint.** The `.scoreable` hover and focus styling already in Control Center
   applies to cards too. Check it in a browser. If the card needs its own `cursor:pointer`,
   add a rule in Control Center's own `<style>`, not in the engine.

The Hub passes no hook, so its cards aren't clickable.

---

## 8. Tests and checks

### 8.1 `sage-tools-api`

See §5.6. Then run:

```bash
node apps-script/verify-standard-tournament-generator.mjs
```

```bash
npm run verify
```

### 8.2 Site unit tests (`_tests/unit/`)

Write each test first and watch it fail.

- **`domain/bracket.test.mjs` (new)**, on hand-made models built with `buildDayModel` like
  `views/standings.test.mjs` does. It has a `STAND_HEADER` that adds `,match,from`, and
  rows for the §5.2 shapes `HIMD`, `XD`, `MD`, `T3`, `ONE` copied from §5.2.
  - **`parseFrom`:** `W:`/`L:` parse, and empty or garbage give `null`.
  - **`matchDisplayLabel`:** `F`, `B`, `F (2)`, `QF-2`.
  - **`rankByWinsThenQuotient`:** sorts by wins then quotient, is stable, and puts `null`
    quotients last.
  - **`buildBracket`:**
    - **HIMD:** round keys `R16, QF, SF, FINAL`; node ids per round `[R16-3]`, `[QF-2]`,
      `[SF-1, SF-2]`, `[F]`; bronze `B`; eligible.
    - **T3:** `SF-1`'s seats are `T3_SF_2` and `T3_SF_3`, in that order.
    - **ONE:** one FINAL node with two games `F (1)` and `F (2)`; not eligible.
    - **Without the columns:** the PFS-style rows `NMD_QF_1..8`, `SF_1..4`, `B`, `F`
      without `match`/`from` give `linked:false`, 4/2/1 nodes, and an eligible bracket.
    - A bronze decided by a BYE is dropped.
  - **`bracketShapeKey`:** gives `3x2`, `4x2`, `2x2`, `3x1`, and `null` when not linked.
  - **`seatText`:** all five kinds, plus:
    - the precedence category > shape key > `Qualifier`;
    - no label is used on a seat whose `from` is set;
    - the `TBD` fallback.
  - **`sourceTag`:**
    - matches regardless of name order, case and extra spaces;
    - returns `''` on no match and on two matches;
    - is `Br 2 · #1` with the brackets of §2.5's example;
    - is `#n` when `bracket` is empty.
  - **`layoutBracket`:**
    - For HIMD: `R16-3`'s middle equals `QF-2`'s seat-0 line end; the `QF-2` → `SF-2` line
      ends at SF-2's seat-1 centre; no two nodes in a round overlap; the bronze sits below
      every node; the width is 4 rounds.
    - For XD: SF-1's middle is the mean of QF-1's and QF-2's middles.
    - Fallback: no lines, and the columns are centred.
- **`views/bracket.test.mjs` (new):**
  - The chart contains one `.bk-card` per game, with `data-bk-match` ids.
  - It has `Winner of QF-2`, `Loser of SF-1` and the `Final` head with `bk-card--final`.
  - `decorate` attrs and className land on the card, and `decorate` is never called
    without a Match.
  - The `bk-short` text is the surnames.
  - Names are escaped.
  - In the round strip, the start-round rule (§7.4) picks the expected round in three
    states: before the playoffs, mid, and finished.
- **`views/standings.test.mjs` (extend):**
  - The switch is present only for an eligible standard category (absent for `ONE`, a dual
    meet and a team event).
  - With `view: 'list'`, the output is **identical** to the output without the option. Use
    a deep string equality on the HIMD and PFS models.
  - `view: 'bracket'` swaps the stage blocks for `.bk`.
  - `availableWidth: 300` gives `.bk-rounds`.
  - The column class and `--bk-col-w` appear when the chart is wider than 332px.
  - `pairUpMatchups` still behaves as its existing test says.
- **`domain/matches-standings-codes.test.mjs` (extend):** `rowsToStandings` reads
  `match`/`from`, and gives `''` when the columns are absent.
- **`data/data-modules.test.mjs` (extend):** `eventConfigFrom` gives `playoffSeats: {}`
  without the setting. It keeps valid entries and drops bad inner keys, empty labels and
  non-object values, each with a warning (stub `console.warn` as the `seedLabels` tests do).

### 8.3 A fixture for fixture mode (`_fixtures/bracket-demo-2026/`)

Add an event `bracket-demo-2026` to `_fixtures/config.json`:

- `type: "standard"`, title `Bracket Demo (fixture only)`.
- One day, `bracket-demo-2026-day1`, dated `2026-11-01`, with facilities `Main` and
  `Annex`.
- `display.divisions` `{ "": "" }` and `display.events` `{ "TMD": "Three-bracket Doubles",
  "FXD": "Four-bracket Mixed", "CWD": "Crossover Doubles", "LXD": "Legacy Mixed" }`.
- `display.playoffSeats.TMD` = the HIMD labels of §6, with the key renamed to `TMD`.

Snapshots go in three files, `rr.json`, `drawn.json` and `final.json`, opened with
`?fixture=rr` and so on.

- **`Main`** has `TMD` (13 pairs, brackets 5-4-4, the §5.2 HIMD ladder with `TMD_` codes),
  `FXD` (20 pairs, 4×5, the XD ladder) and `CWD` (10 pairs, 2×5, the MD ladder). Its
  `STANDINGSCSV` carries `match` and `from`.
- **`Annex`** has `LXD` (16 pairs, 4×4, the XD ladder) **without** the two columns: it's
  the fallback.
- **The states:**
  - `rr`: half the round robin played; playoff slots' names equal their codes.
  - `drawn`: round robin done; round robin seats named on the sheet; the earliest playoff
    round half played.
  - `final`: everything played.
- **Pairs are fake** (two given names and a surname each). Don't copy a real roster.
- **`matchesCsv` header:** `matchNumber,court,Schedule,CourtAssignment,teamCode1,team1Player1,team1Player2,team1Score,teamCode2,team2Player1,team2Player2,team2Score`.
  An unfilled playoff side's player cells hold its slot code, as the sheet does.

Build the JSON with a throwaway Node script and commit **only the JSON**. Use it for the
browser checks in §8.5. The comparison harness doesn't run it: it isn't in `SNAPSHOTS` in
`_tests/helpers/fixtures.mjs`, because the baseline tree would have nothing to compare.

### 8.4 The comparison harness

`npm run verify` compares the branch with its starting commit on the Piggleball and Pickle
for Sight fixtures. In List view (the default, D4), the only expected difference is **the
switch under the title** of each eligible category on the standard Hub's and Control
Center's Standings views.

- Add one entry to `_tests/engine/accepted.mjs`, under a heading comment `// ---- playoff
  bracket chart (sage-docs/docs/specs/.../standings-bracket-chart-spec.md D4, D11) ----`:
  `{ case: <regex over the standings case ids of std-index and cc for piggleball-2026 and
  pickle-for-sight-2026>, kind: 'both', row: 'bracket-chart D11', reason: 'a category with
  two or more playoff rounds has a List / Bracket switch under its title' }`.
- Write the regex against the real case ids, which `npm run compare` prints. It must match
  **only** Standings views.
- Any other case that differs is a bug in this change. Don't widen the entry to hide it.

### 8.5 Browser checks

Open Control Center locally with `?fixture=rr`, `?fixture=drawn` and `?fixture=final`, and
pick **Bracket Demo**. For the Hub, make an untracked copy of an engine standard event's
shell as `beta.html` in that event's folder (the site's `.gitignore` excludes `**/*beta*`),
set its `EVENT_KEY` to `bracket-demo-2026`, and open it with the same `?fixture=`. Never
commit it.

| Size | Expect |
| --- | --- |
| 1280×800 and 1440×900 | List by default; Bracket widens each eligible column to its chart, with lines and labels as in §2.5; `LXD` in the fallback |
| 1180×820 (tablet sideways) | As desktop |
| 820×1180 (tablet upright) | Phone layout; `CWD` (2 rounds, 380px) and `FXD` (3 rounds, 588px) show the whole tree; `TMD` (4 rounds, 796px) shows the round strip |
| 375×812 (phone) | The round strip for every eligible category; Full tree opens, fits, pinches, pans, double-taps, goes compact under 0.6×; Back and Escape close it |
| Control Center, signed in (fixture mode counts as signed in for score entry) | Clicking a card with a scheduled match opens the score dialog, in the chart and in Full tree; saving updates the card |

Take a screenshot of each row for the owner.

---

## 9. Documentation (`sage-docs`)

Present tense, each page linking its counterpart as the others do.

- `features/tournament-hub.md`, Standings: the List / Bracket switch, what an empty seat
  says (§2.2), the round strip and Full tree on phones.
- `features/control-center.md`, Standings: the same, plus clicking a bracket card to enter
  a score.
- `features/standard-tournament-generator.md`: the two `STANDINGSCSV` columns, and
  **SAGE → Add bracket links** for workbooks made before 1.3.0. Also the hand-built
  workbook note (§5.5).
- `technical/site-engine.md`:
  - The model paragraph lists `playoffSeats`.
  - The Extension points table's Standings row adds `decorateBracketMatch(m)`, which
    Control Center uses for click-to-score.
  - The Layout section lists `domain/bracket.js`, `views/bracket.js` and
    `views/bracket-full.js`.
- `technical/control-center.md`: the hook and the listeners.
- `technical/event-data-config.md` and `event-data/config/README.md`: `display.playoffSeats`
  (§6).
- `specs/implemented/standard-tournament-master-spec.md`:
  - §10.1's table gains the `AD:AF` row.
  - §10.4's opening says "nine columns", with the `H`/`I` formulas.
  - §18 Divergences gains a row naming this spec as the source.
- The workspace `CLAUDE.md`, the `standard-tournament-generator.gs` bullet: one sentence
  that 1.3.0 writes `STANDINGSCSV`'s `match`/`from` from a `Variables!AD:AF` table, and
  that **SAGE → Add bracket links** adds them to an older workbook.
- This spec: move it to `in-progress/`, then `implemented/`, as the procedure in
  `docs/specs/README.md` says, with a changelog entry each time.

---

## 10. Rollout

1. **`sage-tools-api`:** the generator change and the verify script (§5). The owner pastes
   1.3.0 into the **SAGE Standard Tournament Master**'s script project.
2. **The Sportsfest workbook** (`18UH-CHx4_qIdI5y89qrSKDQR0Cd0dzKWQRQfKunNFjg`):
   - The owner pastes 1.3.0 into its script project and runs **SAGE → Add bracket links**.
   - Expect `MD` and `WD` in the 2-bracket shape and `XD` in the 4-bracket shape (§5.2).
   - Sync the day once and check that the published snapshot's `standingsCsv` has `match`
     and `from`.
3. **The site:** §7 and §8, on a branch, **after 24 October** and **by 9 November**.
4. **`event-data`:** `config/README.md` (§6). Add `display.playoffSeats` to the Sportsfest
   entry only if the owner gives labels.
5. **`sage-docs`:** §9.

The Claude steps are 1 (code), 3, 4 (README) and 5. The Google steps in 1 and 2 are the
owner's.

---

## 11. Acceptance

- [ ] The verify script passes, with the §5.6 checks.
- [ ] Add bracket links refuses when a code differs, and is idempotent.
- [ ] `_tests` `npm run verify` passes with exactly the one new accepted entry.
- [ ] In List view nothing changes (unit test of string equality, plus the harness).
- [ ] §8.5's table, with a screenshot of each row.
- [ ] A category with only a final or series has no switch.
- [ ] Dual-meet and team pages are unchanged (the harness covers them).
- [ ] The Sportsfest snapshot carries `match` and `from` after step 2 of §10.
- [ ] The docs in §9 are updated, and the changelog and version are bumped.

## 12. Out of scope

- A minimap in Full tree.
- Lines into the Bronze.
- Dual-meet and team events (D6). Their stage builders don't change.
- Changing the Tournament Calculator's chart (D10).
- Predicting who qualifies before the sheet says so (D9).
- The schedule board and scorer page. They could later show `seatText` labels for unfilled
  playoff sides; not here.

## 13. Open questions

None block the build. If the owner wants seat labels for the Sportsfest, they give the
text for `MD`/`WD` (2×2) and `XD` (4×2), and it goes into `events.json` (§6).

## 14. Divergences

The build follows the spec; these are the places it differs from the text, or fills in what it leaves
open.

- **The Full tree layer's CSS selector is `div.bk-full`** (§7.7). §7.4 gives the span that holds a
  pair's full names the class `bk-full`, and §7.7 gives the layer the same class, so a plain
  `.bk-full{position:fixed…}` would make every name span a fixed layer. The markup is as written; only the
  layer's rules are scoped to the `div`.
- **Pointer handling.** A mouse double-click uses `dblclick`; a touch or pen double-tap uses the two
  `pointerup`s (each would otherwise fire both). A drag takes the pointer only once it has moved 6px, so a
  tap still reaches the card under it. Plain wheel scrolling does nothing; `ctrl`+wheel zooms (§7.7).
- **Names the spec leaves open.** `seatContext(bracket, model)` (domain) reads the labels, the shape and the
  named round robin rows `seatText` needs; `startRound(bracket)` (views) is §7.4's start-round rule, also
  written as `data-bk-start` on `.bk-rounds`; the round strip's rounds are numbered 0…n-1 in
  `data-bk-round`, the last included (§7.4 writes `"last"`); `.bk-track` carries `data-group-key`, so
  `scroll.js` can tell one category's strip from another's; `stageTablesHTML` takes an optional
  pre-built `bracket`; `PLAYOFF_VIEW_KEY`, `CARD_PADDING` and `COLUMN_WIDTH` are exported.
  The strip and chips have no visible scrollbar.
- **Linked means every retained playoff row has a `match`** (§7.3.4): a BYE row, or a Bronze dropped as
  bye-decided, does not count against it.
- **`readPlanBlock_` itself now returns `advance` and `fill`** (§5.4 step 1 describes reading
  `Variables!I:N` like it); the draw import ignores them.
- **The fixture event has `"scoreEntry": "console"`** (§8.3), or its cards could not open the score dialog in
  the §8.5 check. `_tests/helpers/fixtures.mjs`'s `eventConfig` carries `playoffSeats` like `eventConfigFrom`.
- **Control Center redraws Standings after a fixture-mode score save** (§7.9), so a card shows the new score
  as §8.5 expects; a live save still waits for the snapshot like every other view.
- **Piggleball's and Pickle for Sight's own public pages are not engine pages.** Their `index.html` files are
  hand-built and import nothing from `lib/v1`, so D11's switch reaches them in Control Center (and in the
  harness, which renders the standard template over their fixtures) but not on their Hubs. The §8.5 Hub check
  used `events/acso-sportsfest-2026/`'s shell, the engine standard event.
- **Piggleball and Pickle for Sight are Control Center only** (owner, 22 October 2026). Piggleball's workbook is
  sheet-generated, so generator 1.3.0 and **Add bracket links** give it the full linked chart there. Pickle for
  Sight's workbooks are hand-built and stay without the `match` and `from` columns, so Control Center draws
  its chart without lines and with `TBD` seats. Neither event's public page changes.
- **§5.3's check that the master holds nothing in `Variables!AD:AF`** cannot be run from the repo. The master spec's
  §10.1 lists `A:C`, `H:R` and `AB` only, and the generator clears `AD2:AF` before writing, but the owner
  should look at the real master before pasting 1.3.0.
