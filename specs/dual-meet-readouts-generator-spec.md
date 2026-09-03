# Spec — Dual Meet Readouts Generator (Phase 3)

Phase 3 of the Dual Meet Sheet Generator: the four **readout** tabs —
`Court Control`, `Timeline`, `CSV` and `STANDINGSCSV` — plus the small
additions to `SCHEDULE` and `Variables` they depend on.

With these, a generated workbook runs an event end to end and publishes
through the live sync. Without them, a generated workbook has correct
category tabs and a correct schedule but cannot publish anything.

Delivery is the same as Phases 1 and 2: bound Apps Script in
`sage-tools-api/scripts/sheet-generator.gs`. No new infrastructure, no Cloud
Run deploy, no `package.json` bump — `scripts/*.gs` is not part of the
Cloud Run service.

| | |
| --- | --- |
| Scope | `Court Control`, `Timeline`, `CSV`, `STANDINGSCSV` |
| Also in scope | `SCHEDULE`'s end-time row (§3.2) and two `Variables` cells (§4) |
| Not in scope | Standard-format tournaments; player-name import |
| Prior phases | `dual-meet-sheet-generator-spec.md` (category tabs), `dual-meet-schedule-generator-spec.md` (`SCHEDULE`) |

---

## 0. Orientation — read this first if you have no context

**What this is.** SAGE runs pickleball tournaments. A "dual meet" is two
clubs playing each other across several categories (age/skill/gender
divisions). Each event runs off one Google Sheets workbook. The **Dual Meet
Sheet Generator** is bound Apps Script inside that workbook that builds the
event's tabs from a CSV the Tournament Time Calculator exports.

The operator copies the *SAGE Dual Meet Master* workbook, opens the copy,
and runs `SAGE → Generate event tabs` from the menu, pasting or dropping the
calculator's CSV into a sidebar.

**What the four tabs are for.**

| Tab | Read by | What it holds |
| --- | --- | --- |
| `Court Control` | the operator, live | Which match is on which court right now, plus pace-against-plan estimates |
| `Timeline` | the operator, planning | A pair × time-slot grid of match counts — used to spot a pair double-booked |
| `CSV` | **the live sync** | One flat row per match. This is what gets published |
| `STANDINGSCSV` | **the live sync** | One flat row per pair, with W/L and quotient |

The last two carry the most weight: the Cloud Run sync fetches them **by
name** and publishes them to the `event-data` repo, which the public event
pages read.

**Where everything lives.**

| Thing | Path |
| --- | --- |
| The generator | `sage-tools-api/scripts/sheet-generator.gs` |
| Phase 1 spec — category tabs | `sage-docs/specs/dual-meet-sheet-generator-spec.md` |
| Phase 2 spec — `SCHEDULE` | `sage-docs/specs/dual-meet-schedule-generator-spec.md` |
| The live sync | `sage-tools-api/scripts/sheets-sync.gs` |
| The Cloud Run fetchers | `sage-tools-api/src/sync/` |

**Vocabulary.**

| Term | Means |
| --- | --- |
| category | A division, keyed `LIWD`, `AMD`, … — one tab each |
| pair | Two players who play together; the atomic competitor |
| `n` | Pairs per club per bracket — `teams_a / brackets` |
| bracket | A group within a category; 1 or 2 of them |
| slot | One start time across all courts — two `SCHEDULE` rows |
| team code | `<CLUB>_<KEY>_<i>`, e.g. `PNF_LIWD_3` — the join key for everything |
| blank slot | A (court, slot) cell with no match — carries `-` |

**There is no test framework.** No linter, no build step, no TypeScript.
Keep derivations in pure functions so they can be checked by hand against
§10 before any Sheets code runs.

**Section-reference convention.** A bare `§n` means this document. A
reference into another spec always names it — "Phase 1 spec §11.1", "Phase 2
spec §8.5". All three documents have overlapping section numbers.

---

## 1. Sources

None of the workbook's own structure is in source control. Everything below
was read out of these two files:

| Workbook | File ID | What it holds |
| --- | --- | --- |
| Reference (PNF × BUP, played) | `1DH7BNVdvSkyaywavLzUlfCP4NEhT0SjnzFr_e0LVfFg` | All four tabs built at 9 courts / 15 slots / 126 matches |
| Master template | `1cBvQ5r6Nm29BJ7Hpsk_gonX31hWJHOrCK1WPLgQaqtg` | The stripped prototypes the generator grows |

### 1.1 How to read a workbook's formulas

The Drive **text** export returns values only — it will not show a single
formula. To read formulas, export as `.xlsx` and unzip it (an `.xlsx` is a
zip of XML):

| Want | Read |
| --- | --- |
| Cell formulas | `xl/worksheets/sheet<N>.xml`, the `<f>` element inside each `<c>` |
| Sheet name → file | `xl/workbook.xml` `<sheet>` + `xl/_rels/workbook.xml.rels` |
| **The Named Function library** | `xl/workbook.xml`, `<definedNames>` |

Two traps when parsing that XML:

- **Empty cells are self-closing** (`<c r="D5"/>`). A regex expecting
  `<c …>…</c>` silently attributes the *next* cell's formula to the empty
  one, and every address after it is wrong. Match both forms.
- **Formulas Excel cannot represent come back wrapped** as
  `IFERROR(__xludf.DUMMYFUNCTION("<real formula>"), "<cached value>")`.
  `UNIQUE`, `FILTER` and `REGEXMATCH` are all in that class. Unwrap the
  shim to get the real formula.

---

## 2. The function library

`GETTOTALWINS`, `SORTBYWINS`, `GETSCOREAGAINSTPAIR` and the rest are Google
Sheets **Named Functions** — 29 `LAMBDA` definitions in the workbook's
`<definedNames>`. They are not Apps Script.

Two properties follow, and Phase 3 depends on both:

1. **They are ordinary formulas**, so Sheets tracks their dependencies and
   they recalculate whenever `SCHEDULE` changes. The readouts publish live
   scores, so this is what makes the sync correct rather than stale.
2. **They already read `SCHEDULE` dynamically**, at any court count.

### 2.1 `STACKBLOCKS` is the base layer

`SCHEDULE` lays courts out as a repeating 8-column block from column `D`
(Phase 2 spec §2.1). `STACKBLOCKS(first_col, step, first_row)` walks that
period and stacks one column from every court block into a single array:

```
STACKBLOCKS = LAMBDA(first_col, step, first_row, LET(
  n,   INT((COLUMNS(SCHEDULE!$1:$1) - first_col) / step) + 1,
  ref, LAMBDA(c, LET(L, SUBSTITUTE(ADDRESS(1, c, 4), "1", ""),
                     INDIRECT("SCHEDULE!" & L & first_row & ":" & L))),
  REDUCE(ref(first_col), SEQUENCE(n - 1, 1, first_col + step, step),
         LAMBDA(acc, c, VSTACK(acc, ref(c))))
))
```

It derives the court count from the sheet's own width, so nothing built on
it needs regenerating when the court count changes.

The five column primitives:

| Named function | Definition | Stacks |
| --- | --- | --- |
| `GETPLAYERSCOLUMN1` | `STACKBLOCKS(5, 8, 6)` | `E` — `teamCode1` |
| `GETMATCHNUMBERS` | `STACKBLOCKS(6, 8, 6)` | `F` — match number |
| `GETPLAYERSCOLUMN2` | `STACKBLOCKS(7, 8, 6)` | `G` — `teamCode2` |
| `GETSCORESCOLUMN1` | `STACKBLOCKS(9, 8, 6)` | `I` — `team1Score` |
| `GETSCORESCOLUMN2` | `STACKBLOCKS(10, 8, 6)` | `J` — `team2Score` |

Each stacked array is **court-major**: all of court 1's rows, then all of
court 2's, every block the same height. §6.1 exploits that.

Two more read the roster index, **open-ended from row 3**:

```
GETTEAMCODES   = LAMBDA({'Reference for Players'!$A$3:$A})
GETPLAYERNAMES = LAMBDA({'Reference for Players'!$B$3:$B & " " & 'Reference for Players'!$C$3:$C})
```

### 2.2 What composes on top

| Named function | Shape |
| --- | --- |
| `GETTEAM1CODEBYMATCH(m)` | `FILTER(GETPLAYERSCOLUMN1(), GETMATCHNUMBERS() = m)`, `"-"` if absent |
| `GETTEAM2CODEBYMATCH(m)` | same against `GETPLAYERSCOLUMN2()` |
| `GETPLAYERNAMESBYTEAMCODE(code)` | `FILTER(GETPLAYERNAMES(), GETTEAMCODES() = code)` — returns **two** names |
| `GETSCOREAGAINSTPAIR(p1, p2)` | `MATCH(CONCAT(p1,p2), GETMATCHES(), 0)` into the score columns |
| `GETTOTALWINS` / `GETTOTALLOSES` | `COUNTIF(GETMATCHRESULTSBYTEAMCODE(code), TRUE / FALSE)` |
| `GETSCOREQUOTIENT(code)` | `GETTOTALSCORE / GETTOTALOPPONENTSCORE` |
| `COUNTPAIRAT(time, code)` | a pair's match count at one slot time — `Timeline`'s body |

**Build on these; do not re-derive them.** Any new formula that needs to
walk the court blocks should call a primitive rather than rebuild the
period-8 walk.

### 2.3 Why generation starts from a copy of the master

Named Functions live only in the workbook, yield `#NAME?` in a blank
spreadsheet, and cannot be created through the Sheets API. Generation must
therefore start from a copy of the master and must run as bound script — the
same conclusion Phase 1 reaches, for this reason.

The library is readable (§1.1), so it can be inspected when a formula's
behaviour is in question.

---

## 3. What `SCHEDULE` provides

Phase 2 builds `SCHEDULE`. Phase 3 reads it, and needs one thing added.

### 3.1 The grid

Rows 1-4 blank, row 5 the header band, data from row 6. Courts are a
repeating 8-column block from `D`; a slot is two rows.

```
court c block starts at column  4 + 8(c - 1)
slot s occupies rows            6 + 2(s - 1)  and  7 + 2(s - 1)
grid width                      8 * courts + 2
```

Within a block: `D` names, `E` `teamCode1`, `F` match number, `G`
`teamCode2`, `H` names, `I` `team1Score`, `J` `team2Score`. Row 5 carries
`Court <c>` at the `F` offset. Column `B` carries each slot's time, merged
across the slot's two rows. An unused court in a slot carries `-` in `E`,
`F` and `G`.

### 3.2 `SCHEDULE` carries an end-time row

The grid runs **one row-pair past the last slot**. That pair holds the time
the event ends — the last slot's start plus one pitch — with `-` in `E`,
`F` and `G` of every court, and nothing else.

```
gridSlots = slots + 1
rows      = 5 + 2 * gridSlots
endTimeRow = rows - 1
```

At 15 slots the last slot's time sits at `B34` and the end time at `B36`,
with row 37 blank.

Three consumers read it, and each fails silently without it — no error, just
blanks and wrong arithmetic:

| Consumer | Reads |
| --- | --- |
| `Court Control!H9` | `=SCHEDULE!$B$<endTimeRow>` — "Ideal End Time" |
| `Court Control!J9`, `J10` | both Time Differences, derived from `H9` |
| `Timeline` row 2 | its last time column, which pairs with this row |

The trailing pair needs no special handling in the build: it has no matches,
so the existing "unused court carries `-`" path fills it, and the existing
time loop writes `start + slots * pitch` as its time. Tile, write and colour
over `gridSlots` rather than `slots`.

### 3.3 Blank slots

```
blankSlots = courts * slots - matches
```

**Real slots only** — the end-time row's `-` cells are the end marker, not
unplayed court-slots. At 9 courts × 15 slots with 126 matches that is 9.

---

## 4. `Variables`

Phase 1 writes `C` (category key), `D` (category name), `G2` (courts), `H2`
(start), `I2` (duration). Phase 2 writes `E` (category colour). `B` and `F`
are free.

### 4.1 `I2` holds the slot pitch

```js
sheet.getRange('I2').setValue((duration_min + buffer_min) / 1440);
```

`Timeline!C2 = B$2 + Variables!$I$2` chains the time headers off `I2`, and
`SCHEDULE`'s slot pitch is `duration_min + buffer_min`. If `I2` held the
duration alone, any non-zero buffer would drift the Timeline's headers away
from the real slot times — and because `COUNTPAIRAT` matches the slot time
**exactly**, every count on the tab would read `0` from the first buffered
slot onward.

`Court Control!H7` and `J7` also read `I2`, as the per-match cost in their
pace maths. Pitch is the right number there too: a match occupies its buffer
as surely as its duration.

The cell's label should read "Slot Pitch" rather than "Est Duration per
Match" (§12).

### 4.2 `F` holds `n`

The generator writes `n` — pairs per club per bracket, `cat.teamsA /
cat.brackets` — to column `F` on each category's own row, beside the key in
`C`. §6.3 reads it back with a `VLOOKUP` over `Variables!$C:$F`.

---

## 5. The four tabs

Column letters are literal. Row numbers are given as formulas of `courts`,
`slots` and `matches` wherever they move.

### 5.1 `Court Control`

Operator-facing, live. 13 columns (`A:M`). Two regions that size
independently: court blocks that grow downward, and a fixed stats block.

**Court blocks — 3 rows each, from row 5.**

```
court c occupies rows  5 + 3(c - 1) .. 7 + 3(c - 1)
last court row         3 * courts + 4          (31 at 9 courts)
```

Three rows because the block is one code row plus the two names that spill
from it.

| Cell | Content | Source |
| --- | --- | --- |
| `B<top>` | court number, merged across the 3 rows | generator |
| `C<top>` | **match number**, merged — typed during play | operator |
| `D<top>` | `=GETTEAM1CODEBYMATCH(C<top>)` | generator |
| `E<top>` | `=GETTEAM2CODEBYMATCH(C<top>)` | generator |
| `D<top+1>` | `=GETPLAYERNAMESBYTEAMCODE(D<top>)` — spills into `D<top+1>`, `D<top+2>` | generator |
| `E<top+1>` | `=GETPLAYERNAMESBYTEAMCODE(E<top>)` — spills likewise | generator |

Column `C` is the tab's entire live input surface: type a match number, and
the codes and names appear.

**Stats block — fixed at rows 5-10, columns `G:M`.** It does not move with
the court count.

| Cell | Content | Kind |
| --- | --- | --- |
| `H5` | `matches` | literal |
| `H6` | `=SUM(STANDINGSCSV!$D:$D)` | formula — matches done, since each finished match yields one win |
| `H7` | `=Variables!$I$2*H6/Variables!$G$2` | formula — ideal elapsed |
| `H8` | `=SCHEDULE!$B$6+H7` | formula — ideal current time |
| `H9` | `=SCHEDULE!$B$<endTimeRow>` | formula — ideal end time (§3.2) |
| `J5` | `=NOW()` | formula |
| `J6` | `=H5-H6` | formula — matches remaining |
| `J7` | `=Variables!$I$2*(J6+M7-M8)/Variables!$G$2` | formula — estimated remaining |
| `J8` | `=J5+J7` | formula — estimated end |
| `J9` | `=text(H9-J8,"hh:mm")` | formula — ahead of plan |
| `J10` | `=text(J8-H9,"hh:mm")` | formula — behind plan |
| `M7` | `blankSlots` (§3.3) | literal — total blank slots |
| `M8` | `0` | literal — blank slots *done*; operator increments during play |

`J7`'s `M7 - M8` term is why blank slots are tracked at all: an idle court
burns a slot as surely as a played one, so the work remaining is the matches
still to play **plus** the blank slots still to sit through.

`J9` and `J10` are the same gap in both directions; whichever is positive is
the one the operator reads.

**Prototype: 2 court blocks** (rows 5-10), because the stats block occupies
exactly those six rows. One court block would not make the tab shorter —
`getLastRow()` stays 10 either way.

### 5.2 `Timeline`

Operator-facing. A pair × slot grid of match counts, used to spot a pair
double-booked: **any body cell above 1 is a conflict.**

**Row 2 — time headers.**

| Cell | Content |
| --- | --- |
| `B2` | `=Variables!H2` — the start time |
| `C2` … | `=B$2+Variables!$I$2`, each chaining off the previous |
| last time column | `B` + `slots` → **`slots + 1`** time columns, the last being the end time |
| next column | `TOTAL` (label) |
| next column, row 2 | `=SUM(B3:<lastTimeCol><lastRow>)/2` |

The `/2` is not a fudge: every match appears twice in the body, once for
each of its two team codes.

At 15 slots the time headers are `B2:Q2` (`15:00` … `21:15`), `TOTAL` sits
at `R2`, and the grand total at `S2`.

**Column A — a spill.**

```
A3 = UNIQUE(FILTER('Reference for Players'!$A$3:$A,
                   'Reference for Players'!$A$3:$A > 0))
```

Open-ended from row 3, matching `GETTEAMCODES` (§2.1). Phase 1 places its
first `Reference for Players` anchor at row 3, so no bound is computed and
none can drift.

The predicate is `> 0` — any non-empty — **not** "contains an underscore".
The spill therefore includes each category's display name and each club's
name as well as the pair codes. Those header rows read `0` across the whole
row, which is correct.

**Body — one row per spilled label, filled down over all of them.**

| Cell | Content |
| --- | --- |
| `B3` … `<lastTimeCol>3` | `=COUNTPAIRAT(B$2,$A3)` |
| `<totalCol>3` | `=SUM(B3:<lastTimeCol>3)` |

A pair row's total is that pair's match count — 4 for a group pair at
`n = 4`, 1 for a bronze or final entrant.

### 5.3 `CSV`

Machine-facing; this is what the live sync publishes. 12 columns, header on
row 1, **one row per match** from row 2.

| Col | Header | Row 2 content |
| --- | --- | --- |
| `A` | `matchNumber` | `1` … `matches` — **literal** |
| `B` | `court` | `=IFNA(FILTER('Court Control'!$B$5:$B$<lastCourtRow>,'Court Control'!$C$5:$C$<lastCourtRow>=$A2),"")` |
| `C` | `Schedule` | `=MATCHTIME($A2)` (§6.1) |
| `D` | `CourtAssignment` | `=MATCHCOURT($A2)` (§6.2) |
| `E` | `teamCode1` | `=GETTEAM1CODEBYMATCH($A2)` |
| `F` | `team1Player1` | `=INDEX(GETPLAYERNAMESBYTEAMCODE($E2),1)` |
| `G` | `team1Player2` | `=INDEX(GETPLAYERNAMESBYTEAMCODE($E2),2)` |
| `H` | `team1Score` | `=GETSCOREAGAINSTPAIR($E2,$I2)` |
| `I` | `teamCode2` | `=GETTEAM2CODEBYMATCH($A2)` |
| `J` | `team2Player1` | `=INDEX(GETPLAYERNAMESBYTEAMCODE($I2),1)` |
| `K` | `team2Player2` | `=INDEX(GETPLAYERNAMESBYTEAMCODE($I2),2)` |
| `L` | `team2Score` | `=GETSCOREAGAINSTPAIR($I2,$E2)` |

`lastCourtRow = 3 * courts + 4` (§5.1). The range starts at row **5**, the
first court row — column `B` is the "is a match entered on this court"
probe, so row 4 must not be included.

`B` and `D` are not redundant: `B` is the court a match is running on *right
now* per Court Control, blank unless it is live; `D` is the court it is
*scheduled* on.

### 5.4 `STANDINGSCSV`

Machine-facing. 7 columns, header on row 1, one row per pair from row 2.

| Col | Header | Row 2 content |
| --- | --- | --- |
| `A` | `teamCode` | `=UNIQUE(FILTER('Reference for Players'!$A$3:$A, REGEXMATCH('Reference for Players'!$A$3:$A,"_")))` |
| `B` | `player1` | `=INDEX(GETPLAYERNAMESBYTEAMCODE($A2),1)` |
| `C` | `player2` | `=INDEX(GETPLAYERNAMESBYTEAMCODE($A2),2)` |
| `D` | `wins` | `=GETTOTALWINS($A2)` |
| `E` | `loss` | `=GETTOTALLOSES($A2)` |
| `F` | `quotient` | `=ROUND(GETSCOREQUOTIENT($A2),4)` |
| `G` | `bracket` | §6.3 |

Unlike `Timeline`, this spill filters on `REGEXMATCH(…,"_")` — team codes
only, no category or club headers.

Column `D` is load-bearing beyond publishing: `Court Control!H6` sums it for
"matches done".

Row count is `categories × (2 clubs × teams_a + 2 bronze + 2 final)` — 84
for the reference event.

---

## 6. Named functions to add to the master

Defined via **Data → Named functions** in the master; every copy inherits
them.

A `LET` name that reads as a cell reference is rejected — `row1` parses as
cell `ROW1` and fails with "argument N of LET is not a valid name". Use
letters-only names or include an underscore.

### 6.1 `MATCHTIME(match_number)`

```
=IFERROR(
  LET(
    h,   ROWS(SCHEDULE!$B$6:$B),
    pos, MATCH(match_number, GETMATCHNUMBERS(), 0),
    INDEX(SCHEDULE!$B$6:$B, MOD(pos - 1, h) + 1)
  ),
  "Not found"
)
```

`GETMATCHNUMBERS()` is court-major (§2.1): court 1's whole `F6:F`, then
court 2's `N6:N`, every block the same height. So a single `MATCH` locates
the match, and its offset *within* a block is the slot row:

```
h   = block height = ROWS(SCHEDULE!$B$6:$B)   (the same open-ended shape)
row = MOD(pos - 1, h) + 1                     → index into SCHEDULE!$B$6:$B
```

Searching only the match-number column is required, not an optimisation:
across a whole block a match number can collide with a *score* in `I`/`J`
(match 11 against a score of 11).

### 6.2 `MATCHCOURT(match_number)`

```
=IFERROR(
  LET(
    h,   ROWS(SCHEDULE!$B$6:$B),
    pos, MATCH(match_number, GETMATCHNUMBERS(), 0),
    "Court " & (INT((pos - 1) / h) + 1)
  ),
  "Not found"
)
```

Same `MATCH`; *which* block the position landed in is the court.

### 6.3 The bracket column

`STANDINGSCSV!G2`, filled down. Requires `Variables!F` (§4.2).

```
=IF($A2 = "", "",
  IFERROR(
    LET(
      key, INDEX(SPLIT($A2, "_"), 2),
      idx, VALUE(REGEXEXTRACT($A2, "_(\d+)$")),
      n,   VLOOKUP(key, Variables!$C:$F, 4, FALSE),
      IF(n = "", "", ROUNDUP(idx / n, 0))
    ),
  "")
)
```

Group codes are `<CLUB>_<KEY>_<i>` with `i` running `1..teams_a`, the first
`n` in bracket 1 and the next `n` in bracket 2 (Phase 1 spec §4.6.3), so the
bracket is `ceil(i / n)`.

Playoff codes (`_B`, `_F`, `_SF_1`) have no trailing number, `REGEXEXTRACT`
fails, and `IFERROR` leaves them blank — correct, since a bronze entrant
belongs to no bracket. Single-bracket categories yield `1` for every pair.

---

## 7. Decisions

- **`SCHEDULE` carries an end-time row** (§3.2), and blank slots exclude it
  (§3.3).
- **`Variables!I2` is the slot pitch**, `duration_min + buffer_min` (§4.1).
  It needs no formula edits: `Timeline` and `Court Control` already read
  `I2`.
- **`Timeline` has `slots + 1` time columns** (§5.2), the last being the end
  time.
- **`Timeline`'s body is filled down over every spilled row**, category and
  club header rows included (§5.2). They read `0`.
- **Both spills read `'Reference for Players'!$A$3:$A` open-ended** (§5.2,
  §5.4), matching `GETTEAMCODES`. No bound is computed, so none can drift.
- **`CSV!C`/`D` are named functions** (§6.1, §6.2) rather than generated
  ranges, so they never need regenerating.
- **Ranges the generator writes are exact**, computed from the numbers
  `buildScheduleTab_` already returns. This applies only to `CSV!B2`.
- **`Court Control`'s prototype is 2 court blocks** (§5.1).
- **`CSV!B2` ranges from row 5**, the first court row (§5.3).
- **`STANDINGSCSV!G` is a formula, not generated literals** (§6.3), so it
  stays correct if a code is edited by hand.

---

## 8. Implementation notes

Phase 1 §11.1 covers general Apps Script mechanics. Phase 2 §8.10 and §8.11
add the two that matter most and are easy to miss:

- **`copyTo` carries values, formats, formulas and merges — but not column
  widths or row heights.** Those are sheet properties. Copy the prototype's
  dimensions explicitly, or tiled regions sit at Sheets' defaults while the
  prototype keeps the master's sizing.
- **Flush between stages.** `SpreadsheetApp.flush()` after each phase of
  work. A large un-flushed batch fails with `Service timed out:
  Spreadsheets` — not from too many calls, but from one batch too large to
  resolve.

### 8.1 Where the code goes

```js
// PURE — no Sheets API
function readoutGeometry_(summary, categories)   // -> row/column extents per tab

// SHEETS GLUE
function buildCourtControlTab_(ss, summary)
function buildTimelineTab_(ss, summary)
function buildCsvTab_(ss, summary)
function buildStandingsCsvTab_(ss)
```

`summary` is `buildScheduleTab_`'s return value, which already carries
`matches`, `slots`, `courts`, `width` and `rows`. Extend it with
`gridSlots`, `endTimeRow` and `blankSlots` (§3).

Hook all four into `generateEventTabs` after `buildScheduleTab_`, which runs
last. Order among the four does not matter — none reads another's output at
build time, only at recalculation time.

### 8.2 All four are rebuilt in place

`REBUILT_IN_PLACE_TABS` in `sheet-generator.gs` holds the registry;
`assertTabsPristine_(ss)` checks every entry during validation, before
anything is written. Add four entries:

| Tab | `maxRow` | `maxCol` | Prototype |
| --- | --- | --- | --- |
| `Court Control` | 10 | 13 | 2 court blocks (rows 5-10) + the stats block |
| `Timeline` | 3 | 19 | time header row + one body row |
| `CSV` | 2 | 12 | header row + one match row |
| `STANDINGSCSV` | 2 | 7 | header row + one pair row |

`Court Control` shares `SCHEDULE`'s hard constraint: its tab GID is pinned
in `scripts/sheets-sync.gs` (`WATCHED_SHEET_GIDS`), so replacing the tab
kills the live sync trigger while the sheet still looks correct. `CSV` and
`STANDINGSCSV` are addressed by **name** by the Cloud Run fetcher
(`GvizCsvFetcher`'s `&sheet=`) and would survive replacement; they are
rebuilt in place anyway so one rule covers every readout tab.

Rebuilding in place is non-idempotent — a second run tiles an already-tiled
sheet — which is why the pristine check aborts rather than overwriting.

### 8.3 Growing each tab

| Tab | Grows | To |
| --- | --- | --- |
| `CSV` | rows | `1 + matches` |
| `STANDINGSCSV` | rows | `1 + pairs` |
| `Court Control` | rows | `3 * courts + 4` |
| `Timeline` | rows **and** columns | `2 + spillRows` × `slots + 4` |

`Timeline`'s columns are `A` + `(slots + 1)` time columns + `TOTAL` + the
grand total.

Use `copyTo` from the prototype row. It adjusts relative references, so a
filled-down `=COUNTPAIRAT(B$2,$A3)` becomes `=COUNTPAIRAT(B$2,$A4)` without
rewriting it. Write explicitly only the cells whose content the generator
computes: `CSV!A` (match numbers), `CSV!B2`'s range, `Court Control`'s
literals and `H9`, and `Timeline`'s row 2.

`Timeline`'s body height is the spill height, which no formula reports.
Fill to the pair count plus one row per category and one per club per
category: `pairs + categories * 3`.

### 8.4 `Court Control`'s two regions grow independently

Court blocks grow downward; the stats block stays at rows 5-10. Copying rows
5-10 downward would tile the stats block too. Copy **`B:F` only** — the
court columns — and leave `G:M` alone.

### 8.5 Tracing

Follow Phase 2 §8.9: `trace_` every computed extent. A readout tab one row
short looks entirely normal until the sync publishes short. At minimum log
each tab's final row and column count, `blankSlots` and its arithmetic, the
`endTimeRow` written into `Court Control!H9`, and `Timeline`'s spill height
and how it was derived.

---

## 9. Build order

1. `SCHEDULE`'s end-time row and `blankSlots` (§3) — everything else depends
   on it
2. `Variables` `I2` → pitch and `F` → `n` (§4)
3. Add `MATCHTIME` and `MATCHCOURT` to the master; fix its `CSV` prototype
   row to call them (§12)
4. Strip the four prototypes in the master, then add the
   `REBUILT_IN_PLACE_TABS` entries (§8.2)
5. `buildCsvTab_` and `buildStandingsCsvTab_` — simplest, and the two the
   live sync needs
6. `buildTimelineTab_` — the only two-axis grower
7. `buildCourtControlTab_` — the only one with two independently-sized
   regions
8. Update the completion toast: with these four the workbook is ready to
   run, given rosters
9. Update the documents in §13

---

## 10. Worked example

Input: the reference plan — 9 courts, 15 slots, 126 matches, 7 categories,
`n = 4`, 84 pairs, `start 15:00`, `duration_min 25`, `buffer_min 0`.

| Tab | Extent | Key values |
| --- | --- | --- |
| `SCHEDULE` | `A1:BV37` | end time `21:15` at `B36`; `blankSlots = 9` |
| `Court Control` | `A1:M31` | `H5 = 126`, `H9 = SCHEDULE!$B$36`, `M7 = 9`, `M8 = 0` |
| `Timeline` | `A1:S99` | 16 time columns `B2:Q2` (`15:00`…`21:15`); `S2 = SUM(B3:Q99)/2 = 126` |
| `CSV` | `A1:L127` | 126 match rows |
| `STANDINGSCSV` | `A1:G85` | 84 pair rows |

`Timeline`'s 97 body rows are 84 pair codes + 7 category names + 14 club
names (`PNF` and `BUP` once per category).

---

## 11. Acceptance

- `SCHEDULE` carries an end-time row: `B<endTimeRow>` is the last slot's
  time plus one pitch, with `-` in `E`/`F`/`G` of every court
- `Court Control!H9` resolves to a time, and `J9`/`J10` compute
- `Court Control`'s last court row is `3 * courts + 4`, with the stats block
  still at rows 5-10
- `Timeline` has `slots + 1` time columns, its last equal to `SCHEDULE`'s
  end time, and its grand total equals the match count exactly
- `Timeline`'s body covers every spilled label, header rows included, and no
  body cell exceeds 1
- `CSV` has one row per match, `C`/`D` resolve at any court count, and no
  row reads `Not found`
- `STANDINGSCSV` has one row per pair, and its `D` column sums to the match
  count
- A two-bracket category populates `STANDINGSCSV!G` with `1` and `2`
- Running the generator twice is refused for all four tabs, not
  half-applied
- The sync publishes a generated workbook with no hand-editing

---

## 12. Manual changes required in the master

None of these can be done by the generator; the master must be correct
before a copy is made.

1. Add `MATCHTIME` and `MATCHCOURT` as named functions (§6.1, §6.2).
2. Point `CSV!C2` and `CSV!D2` at them: `=MATCHTIME($A2)`,
   `=MATCHCOURT($A2)`.
3. Range `CSV!B2` from row 5, not row 4 (§5.3).
4. Add `STANDINGSCSV!G2`'s bracket formula (§6.3).
5. Relabel `Variables!I2` to "Slot Pitch" (§4.1).
6. Add a `Variables!F` header, e.g. "Pairs per Bracket" (§4.2).
7. Range both spills from `'Reference for Players'!$A$3`, not `$A$2`
   (§5.2, §5.4).
8. Delete `Timeline!B1`.
9. Confirm `Court Control!M7`/`M8` are labelled "Blank Slots" and "Blank
   Slots Done" (§5.1).
10. Strip each readout tab to the prototype shape in §8.2's table.

**Do #10 last, and generate once immediately afterwards.** The pristine
guard's `maxRow`/`maxCol` are exactly those shapes, so a master that does
not match refuses to generate — with a "start again from a fresh copy"
message that misleads, because the fault is in the master itself.

---

## 13. Documents to update

**This section is the traceable record.** It is where the discrepancies are
written down and where a later reader looks to see what moved and why.

The documents themselves are corrected to plain present tense — they state
what is true now and carry no note that anything changed. Duplicating the
record into them would put the same history in two places and leave it to
drift; git history and this section already carry it.

Each entry names the file, the section, what it currently says that is
wrong, and what it should say instead.

### 13.1 The Named Function correction

Both earlier specs describe the library as container-bound Apps Script
custom functions. It is Named Functions (§2). Their conclusion — generation
must start from a copy of the master, as bound script — is unaffected; the
mechanism and its consequences are not.

This matters beyond terminology. An Apps Script custom function
recalculates from its **arguments**, not from what it reads internally, so
`GETSCOREAGAINSTPAIR($E2,$I2)` — two codes that never change during play —
would never refresh when a score is typed into `SCHEDULE`, and the sync
would publish stale scores. Named Functions are ordinary formulas and carry
no such risk. A reader who believes the earlier description would go looking
for a staleness bug that cannot exist, or add defences against it.

| File | Section | Change |
| --- | --- | --- |
| `specs/dual-meet-sheet-generator-spec.md` | §1 | "container-bound Apps Script custom functions" → Named Functions; `#NAME?` reasoning stands |
| `specs/dual-meet-sheet-generator-spec.md` | §1.1 | The custom function contract — retitle and describe them as Named Functions |
| `specs/dual-meet-sheet-generator-spec.md` | §6 step 5, §9 | "Keep the bound script project, which carries the custom functions" — the library is workbook-level, not in the script project |
| `specs/dual-meet-sheet-generator-spec.md` | §11.1 | "Custom functions recalculate lazily … may show `Loading...`" — Named Functions do not; drop it |
| `specs/dual-meet-schedule-generator-spec.md` | §0 | "Why it must be bound script" — same correction |
| `specs/dual-meet-schedule-generator-spec.md` | Appendix A | `STACKBLOCKS` is presented as a suggestion to define; it exists and the whole library is built on it (§2.1) |
| `technical/dual-meet-sheet-generator.md` | "Why bound Apps Script and not a Cloud Run endpoint" | Same correction |

### 13.2 Phase 3 landing

| File | Section | Change |
| --- | --- | --- |
| `specs/dual-meet-sheet-generator-spec.md` | §10.2 | Mark Phase 3 built |
| `features/dual-meet-sheet-generator.md` | "What it does for you" | Add the four readout tabs |
| `features/dual-meet-sheet-generator.md` | "What it does *not* do" | Only player names remain; remove the readout-tabs entry |
| `features/dual-meet-sheet-generator.md` | "If it refuses to run" | The pristine refusal covers five tabs, not one |
| `technical/dual-meet-sheet-generator.md` | "What is not generated" | Rewrite: nothing is left unbuilt but rosters |
| `technical/dual-meet-sheet-generator.md` | body | Describes Phase 2 as unstarted and blocked on a fork that is settled |
| `CLAUDE.md` | `sheet-generator.gs` bullet | Lists tabs it does not build; after Phase 3 only rosters remain |
| `sheet-generator.gs` | file header | Master-workbook build instructions gain the four prototypes |
| `sheet-generator.gs` | completion toast | Stops saying the workbook is not ready to run |

### 13.3 Not affected

`sheets-sync.gs`, `src/sync/`, `event-data/config/events.json` and the event
pages need no change. Phase 3 changes what a generated workbook contains,
not the shape of what it publishes — `CSV` and `STANDINGSCSV` keep their
existing columns, which is what makes a generated workbook a drop-in for a
hand-built one.
