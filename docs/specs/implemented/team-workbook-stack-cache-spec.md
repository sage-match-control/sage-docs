# Spec — Team workbook recalculation (StackCache and named-function cleanup)

> **Status: implemented.** Applied to the PickleDrive workbook on 2026-10-05,
> steps 0–7 and 9 (step 8, the optional cleanup, was skipped). Written the same
> day against an `.xlsx` export of the workbook,
> **2026-10-03 PickleDrive Club One Year Celebration**
> (`1Bk3iAqB6Fdt6t-EiI8MrU0Or8UZc4MAZcnWcMUOCkOA`, owned by
> `sagematchcontrol@gmail.com`), as it stood after the event: last modified
> 2026-10-03 13:36 UTC. Every figure in §0.3 and §2 was re-checked against a
> fresh export the same day; the workbook had not been modified since.
>
> **What was found when it ran.** `SCHEDULE` already ended at row 46, so step 1
> deleted nothing. All five `StackCache` checks read `410 rows, 0 differ`. The
> unplayed probe read the same values before and after (§6.2's "after clearing"
> column). The before/after export comparison (§6.1) printed `IDENTICAL` across
> `CSV`, `STANDINGSCSV`, `Standings`, `MatchLookup`, `Court Control`,
> `SCHEDULE`, `FINAL RANK` and `Awards`; the workbook went from 40 named
> functions to 25 and from 48 defined names to 33. Three **Sync now** syncs
> afterwards read `fetch=` 345, 391 and 441 ms, against the event day's
> p50 0.3 s and p90 22.6 s. They ran with no edit in between, so they are a
> spot check, not a distribution: the slow reads on the day followed edits.
> The spec needed no change.
>
> **Done by hand, in the workbook.** Named functions cannot be created,
> edited or deleted through the Sheets API or Apps Script, so every step in
> §4 was a person in the Sheets UI. Nothing in any repo changed until §7.

Make the team workbook fast to read, so a sync's Sheets API read stops
waiting 10–58 s for it to recalculate. Three changes, all inside the
workbook:

1. Build `SCHEDULE`'s five stacked columns **once**, in a hidden `StackCache`
   tab, from **direct** cell references (not `INDIRECT`), and point the five
   primitive named functions at it. The stacks stop being rebuilt about 9,200
   times per edit, and stop being volatile, so an edit only recalculates the
   formulas that depend on what it changed.
2. Give `MatchLookup` two helper columns holding each side's base team code,
   and rewrite the six matchup-family functions to read a matchup's scores
   straight from `MatchLookup`'s rows instead of looking each one up again.
3. Remove the 15 named functions nothing calls once 1 and 2 are in.

The PickleDrive workbook is the prototype the team template and the team
generator start from ([Team tournament event-site template](../not-started/team-tournament-template-spec.md),
[Team Tournament Master](../not-started/team-tournament-master-spec.md)), so this is worth doing in
it before either, not only for its own sake: the event is over.

---

## 0. For the implementer

### 0.1 What you are doing

You are guiding a person through edits in a Google Sheet. You cannot make
them yourself: named functions are reachable only through the Sheets UI. Your
job is to:

- hand the person **one step of §4 at a time**, with its click path and the
  exact text to paste, copied from this spec character for character;
- tell them what they should see when the step is done (each step's
  **Check**), and wait for them to confirm it before the next;
- run the before/after comparison (§6.1) yourself, from two `.xlsx` exports;
- after the work is verified, make the documentation changes in §7.

Read §3 before starting, so you can answer "why" questions, and §8 before
step 1, so you know the way back.

### 0.2 Rules

1. **Never improvise a formula.** Every formula and named-function definition
   to paste is in §4. If one is rejected by Sheets, stop and report the exact
   error message; do not adjust it.
2. **One step at a time, in order.** Each step depends on the previous ones
   (step 3 needs the `StackCache` tab from step 2; step 6 must come after
   step 5, or cells show `#NAME?`).
3. **A failed Check means stop.** Don't fix forward. Either the step can be
   undone with **Edit → Undo** (Ctrl+Z, Cmd+Z on a Mac) right away, or the
   workbook goes back to the named version (§8).
4. **Live sync stays paused** from step 0 until step 9. While it is paused,
   no edit publishes or commits to `event-data`.
5. **Touch nothing that §4 doesn't name.** In particular, no cell formula
   outside `StackCache` and `MatchLookup!K:L` changes.
6. **Wait for recalculation.** After a named function changes, every cell
   that uses it recalculates. A thin progress bar appears at the top right of
   the sheet while it runs. Have the person wait until it is gone before the
   next step or a check.
7. Write documentation in the present tense (§7).

### 0.3 Facts checked against the workbook (2026-10-05)

| Fact | Where |
| --- | --- |
| The workbook is unchanged since the event: Drive `modifiedTime` 2026-10-03T13:36:28Z | Drive file metadata |
| **40 named functions** are defined. **27** are reachable from a cell; **13** are not called by any cell formula, conditional format or data validation, and the workbook has no charts | `xl/workbook.xml` `<definedNames>`, every cell formula of all 20 tabs |
| `SCHEDULE`'s content is rows 1–46 and columns `A`–`CE` (83). Court blocks repeat every 8 columns; row 5 labels them `Court 1`…`Court 10`, so there are **10 blocks**. Times are in `B6:B`, one slot per two rows | `SCHEDULE` |
| Per block, 1-based column `5+8k` is team 1's code, `6+8k` the match number (with the matchup title, from `GETMATCHUPTITLE`, in the slot's second row), `7+8k` team 2's code, `9+8k` team 1's score, `10+8k` team 2's score | `STACKBLOCKS` calls in the primitives; `SCHEDULE` row 6–7 |
| `SCHEDULE`'s grid probably runs well past row 46: `Timeline`'s formulas reference `SCHEDULE!$B$6:$B652`, and `Standings` rows 59+ reference `SCHEDULE!…$6:…104`. The export carries only rows that hold a cell, so the real grid height is checked in step 1 | `Timeline!B3`, `Standings!J59` |
| The only volatile cell formula is `Court Control!M50` (`NOW()`). `INDIRECT` appears only inside `STACKBLOCKS` | every cell formula |
| **Nothing in `SCHEDULE` depends on the stacked columns.** Its formulas are player names (`GETPLAYERNAMESBYTEAMCODE`, which reads `Reference for Players`), matchup titles (`GETMATCHUPTITLE` of the same row's codes), times, and the playoff codes in rows 32+ (`=MatchUps!$E$604&"_4"` and similar), where `MatchUps!E604` is `B604&D604`, both typed values. So a direct reference from `StackCache` to `SCHEDULE` creates no circular dependency | `SCHEDULE`, `MatchUps` rows 600–674 |
| `MatchLookup` has headers `matchNumber`, `matchUp`, `teamCode1`, `team1Player1`, `team1Player2`, `team1Score`, `teamCode2`, `team2Player1`, `team2Player2`, `team2Score` in `A1:J1`. Rows 2–141 are formulas for matches 1–140 (`F2` is `=GETTEAM1SCOREBYMATCH(A2)`, `J2` is `=GETTEAM2SCOREBYMATCH(A2)`); rows 142–171 hold typed blank values; nothing is in column `K` or beyond | `MatchLookup` |
| The 101 distinct team codes in `MatchLookup!C` and `G` give the same result from `INDEX(SPLIT(code, "_", 1), 1)` and from `REGEXEXTRACT(code, "^_*([^_]+)")` | checked over every value |
| Score entry clears a score by clearing the cell (`SheetsClient.clearScores`), so an unplayed score is a truly empty cell | `sage-tools-api/src/scores/ScoreService.mjs` |
| The sync's log line is `timing edit→request=… fetch=…ms …`, scoped by the day key (`pickledrive-anniversary-2026-day1`) | `sage-tools-api/src/sync/SyncService.mjs` (`log.info` after publish; `this.logger.child(day)`) |
| `SAGE → Pause live sync` / `Resume live sync` and `SAGE → Sync now` are in the workbook's menu (the sync script is installed and configured there) | `sage-tools-api/scripts/sheets-sync.gs` (`onOpen` menu) |

### 0.4 The tabs this touches

| Tab | What it is | Read by the sync? |
| --- | --- | --- |
| `SCHEDULE` | the court grid scorers type into | no (watched for edits) |
| `CSV` | one row per match, 152 rows; published as the matches table | **yes** |
| `STANDINGSCSV` | one row per team and playoff slot; published as standings | **yes** |
| `MatchLookup` | hidden; the same per-match columns as `CSV`, for matches 1–140; the matchup family reads it | no |
| `Standings` | matchup and team standings; feeds `STANDINGSCSV`, `Awards`, `FINAL RANK`, `Court Control` | no |
| `Court Control` | which match is on which court; has the live clock | no (watched for edits) |
| `StackCache` | **new**, hidden: the five stacked columns | no |

---

## 1. Why: the event-day measurements

From Cloud Run's `timing` lines for 3 October 2026 (full figures in
[Sync pipeline](../../technical/sync-pipeline.md#measured-at-two-events-3-october-2026)):

| | Piggleball (standard workbook) | PickleDrive (this workbook) |
| --- | --- | --- |
| Syncs | 238 | 695 |
| `fetch` (Sheets API read) | p50 0.3 s, p90 0.4 s, max 5.7 s | p50 0.3 s, **p90 22.6 s, max 57.8 s** |
| `edit→published` | p50 3.2 s, p90 5.2 s | p50 3.6 s, **p90 27.3 s, max 62.8 s** |

161 of PickleDrive's 695 reads took 10–58 s; the rest took about 0.3 s. Same
Cloud Run service, same hours, so the service is not the cause. The two-way
split (fast, or very slow, nothing between) fits the Sheets API waiting for a
recalculation to finish before it returns values. The slow reads also cost
Apps Script trigger runtime one-for-one (the lock holder waits on
`UrlFetchApp` throughout): about 105 minutes of it that day, against a
consumer account's 90-minute daily allowance.

## 2. What makes it slow

### 2.1 `STACKBLOCKS` is rebuilt about 9,200 times per edit

Every named function that reads `SCHEDULE` bottoms out in one of five
primitives (`GETMATCHNUMBERS`, `GETPLAYERSCOLUMN1`/`2`, `GETSCORESCOLUMN1`/`2`),
and each primitive is one `STACKBLOCKS` call. `STACKBLOCKS` walks
`SCHEDULE`'s 10 court blocks and stacks one column of each into a single
array, through `INDIRECT` (definition in Appendix A.3). Two things make that
expensive:

- **`INDIRECT` is volatile.** Every formula that reaches `STACKBLOCKS`
  recalculates after **any** edit anywhere in the workbook, including the
  edits that fire the sync and the attendance marks.
- **The columns are open-ended** (`INDIRECT("SCHEDULE!F6:F")`), so each one
  reads to the bottom of the grid, which `Timeline`'s references suggest is
  row 652 or beyond. `SCHEDULE`'s content ends at row 46.

Counted over every formula cell in the workbook, assuming a team plays 4
matches per matchup, all on one side:

| Function | Cells calling it | `STACKBLOCKS` per call | Per recalculation |
| --- | --- | --- | --- |
| `GETPAIRWINSBYMATCHUPANDTEAMCODE` | 91 (`Standings`) | 32 | 2,912 |
| `GETMATCHSCORESBYMATCHUPANDTEAMCODE` | 130 (`Standings` 99, `STANDINGSCSV` 31) | 8 | 1,040 |
| `GETMATCHRESULTSBYMATCHUPANDTEAMCODE` | 30 (`Standings`) | 32 | 960 |
| `GETTEAM1CODEBYMATCH` / `GETTEAM2CODEBYMATCH` | 310 each (`CSV` 152, `MatchLookup` 140, `Court Control` 18) | 2 | 1,240 |
| `GETTEAM1SCOREBYMATCH` / `GETTEAM2SCOREBYMATCH` | 292 each (`CSV` 152, `MatchLookup` 140) | 2 | 1,168 |
| `GETMATCHNUMBERS` | 292 cells, twice each (`CSV!B`, `MatchLookup!B`) | 1 | 584 |
| `GETOPPONENTSCORESBYMATCHUPANDTEAMCODE` | 62 (`Standings` 31, `STANDINGSCSV` 31) | 8 | 496 |
| `GETPAIRLOSESBYMATCHUPANDTEAMCODE` | 15 (`Standings`) | 32 | 480 |
| `MATCHTIME`, `MATCHCOURT` | 152 each (`CSV`) | 1 | 304 |
| **Total** | | | **≈ 9,200** |

By tab: `Standings` ≈ 5,400, `CSV` ≈ 1,800, `MatchLookup` ≈ 1,400,
`STANDINGSCSV` ≈ 500, `Court Control` ≈ 70. `CSV` and `MatchLookup` compute
the same per-match columns (match number, matchup, codes, names, scores).

Volatility also drags in work that has nothing to do with the stacks: the
1,608 `GETPLAYERNAMESBYTEAMCODE` cells in `CSV` and `MatchLookup` take their
arguments from stack-derived cells, so they recalculate on every edit too,
each one a `FILTER` over `Reference for Players`' 2,600 rows.

### 2.2 The matchup family repeats its own work

`GETMATCHRESULTSBYMATCHUPANDTEAMCODE1` (and `…2`), behind every wins and
losses cell in `Standings`:

```
IF(ISNA(GETMATCHESBYMATCHUPANDTEAMCODE1(matchUp, teamCode)), "",
   MAP(GETMATCHESBYMATCHUPANDTEAMCODE1(matchUp, teamCode),
       LAMBDA(mn, IF(OR(ISBLANK(GETTEAM1SCOREBYMATCH(mn)), ISBLANK(GETTEAM2SCOREBYMATCH(mn))), "",
                     GETTEAM1SCOREBYMATCH(mn) > GETTEAM2SCOREBYMATCH(mn)))))
```

- the match list is computed twice (once for `ISNA`, once for `MAP`);
- each match looks up its two scores twice each: four lookups, eight
  `STACKBLOCKS`, although `MatchLookup` already holds both scores in the same
  row the match number came from (`F` and `J`);
- `GETMATCHESBYMATCHUPANDTEAMCODE1`/`2` run
  `MAP(MatchLookup!$C$2:$C, LAMBDA(tc, INDEX(SPLIT(tc, "_", 1), 1)))` over
  the whole open-ended column on every call, about 1,300 passes per
  recalculation.

`GETMATCHSCORES…`/`GETOPPONENTSCORES…` `1`/`2` compute their match list
twice the same way. "Unplayed" is tested with `ISBLANK`, which is why score
entry must leave a cleared score truly empty.

### 2.3 Named functions nobody calls

These 13 are not called by any cell formula, conditional format or data
validation, nor by any function that is:

`COUNTPAIRAT`, `GETMATCHES`, `GETMATCHRESULTSBYTEAMCODE`, `GETPLAYERNAMES`,
`GETSCOREAGAINSTPAIR`, `GETSCOREQUOTIENT`, `GETTEAMCODES`, `GETTOTALLOSES`,
`GETTOTALOPPONENTSCORE`, `GETTOTALSCORE`, `GETTOTALWINS`, `GETWINSCORE`,
`SORTBYWINS`

Most are the standard/dual-meet library's standings functions, inherited by
copying and replaced in a team workbook by the `…BYMATCHUPANDTEAMCODE`
family. After §4 step 5, `GETMATCHESBYMATCHUPANDTEAMCODE1` and `…2` join
them: nothing calls them any more. Step 6 removes all 15. Their definitions
are in Appendix A.2 and A.4.

### 2.4 Not named functions, but nearby

- **Playoff codes in `SCHEDULE` are formulas** (rows 32+,
  `=MatchUps!$E$604&"_4"` and similar). Score cells are typed values.
- **Five broken named ranges**, scoped to `Teams`: `Player`, `Bracket`,
  `Team`, `Level`, `Number`, all `#REF!`.
- **`Timeline (Individual}`** (the closing brace is in the tab's real name):
  8,999 formulas, 522 of them containing `#REF!`, and nothing reads the tab.
- **`Timeline`**: 15,910 cells of 20 `COUNTIFS` each (two code columns per
  court) over `SCHEDULE`'s code columns, to row 652. Not volatile, but
  recalculated whenever a `SCHEDULE` code cell changes. Only
  `Court Control!K50` (`=Timeline!A2`) reads it.
- **`Court Control!K54`** is the formula `=#REF!`; `M54` and `M55` are
  computed from it. Separately, the typed match numbers in
  `Court Control!C` read `#REF!` after the event. Neither is touched here.

## 3. The design

### 3.1 `StackCache`, from direct references

A hidden tab holding the five stacks, one spilled formula each in `A1:E1`.
Each formula is a `VSTACK` of the ten blocks' columns, written as direct,
open-ended ranges:

| Cell | Holds | Columns stacked (court 1 → court 10) | Replaces |
| --- | --- | --- | --- |
| `A1` | match numbers | `F N V AD AL AT BB BJ BR BZ` | `STACKBLOCKS(6, 8, 6)` in `GETMATCHNUMBERS` |
| `B1` | team 1 codes | `E M U AC AK AS BA BI BQ BY` | `STACKBLOCKS(5, 8, 6)` in `GETPLAYERSCOLUMN1` |
| `C1` | team 2 codes | `G O W AE AM AU BC BK BS CA` | `STACKBLOCKS(7, 8, 6)` in `GETPLAYERSCOLUMN2` |
| `D1` | team 1 scores | `I Q Y AG AO AW BE BM BU CC` | `STACKBLOCKS(9, 8, 6)` in `GETSCORESCOLUMN1` |
| `E1` | team 2 scores | `J R Z AH AP AX BF BN BV CD` | `STACKBLOCKS(10, 8, 6)` in `GETSCORESCOLUMN2` |

The five primitives then return `StackCache`'s columns. Every function built
on them keeps its name, arguments and callers; it just reads a range.

**Why direct references and not `=STACKBLOCKS(…)` in the cache.** A cache
built by `STACKBLOCKS` would cut the 9,200 rebuilds to 5, but it would still
be volatile: every edit, including an attendance mark, would recalculate the
five stacks and then everything downstream of them. A direct reference is not
volatile, so Sheets recalculates a cache column only when a cell it covers
changes, and a formula downstream only when a column it reads changed. A
score edit then reaches the score columns and what depends on them (§3.4),
not the codes, names, times or courts.

**Why it's safe here.** `STACKBLOCKS` used `INDIRECT` for two reasons, and
neither applies to one workbook's own cache:

- *The court count.* `STACKBLOCKS` derives it from `SCHEDULE`'s width, so one
  library serves every event. A cache cell belongs to one workbook, whose
  court count is fixed (10). The generator, when it exists, writes the cache
  for its event's court count.
- *Circularity.* A direct reference makes Sheets track the dependency
  statically. That would be a circular dependency if any `SCHEDULE` cell
  depended on the stacks; none does (§0.3).

**Why the results don't change.** `VSTACK(SCHEDULE!F6:F, SCHEDULE!N6:N, …)`
is the array `STACKBLOCKS(6, 8, 6)` builds: the same ranges, in the same
court-major order. Spilled from row 1, a value's position in the cache is its
position in the old array, so `MATCHTIME` and `MATCHCOURT`, which turn a
position into a time slot and a court with `ROWS(SCHEDULE!$B$6:$B)`, stay
correct: each stacked block is `ROWS(SCHEDULE!$B$6:$B)` tall, as before.
The cache columns are open-ended (`$A$1:$A`), so they run past the spill with
empty rows; all five are the same length, so the elementwise `FILTER`s that
combine them still line up, and an empty row never equals a match number or a
team code. Step 2's check compares each cache column with `STACKBLOCKS`'s
array value for value before anything reads the cache.

### 3.2 Base team codes, once

The matchup family matches a team by the part of its code before the first
`_` (`A_1` → `A`, `SF-J_2` → `SF-J`). Today every call re-derives that for
every row of `MatchLookup`. Two helper columns derive it once:

- `MatchLookup!K` (`teamBase1`): from `C` (`teamCode1`);
- `MatchLookup!L` (`teamBase2`): from `G` (`teamCode2`).

Each is one `ARRAYFORMULA` in row 2, using
`REGEXEXTRACT(TO_TEXT(code), "^_*([^_]+)")`, which returns what
`INDEX(SPLIT(code, "_", 1), 1)` did: the first non-empty part before an `_`
(checked over all 101 codes in the workbook, §0.3). A blank or unparseable
row gives `""` through `IFERROR`, which never equals a team code; the old
`SPLIT` gave an error there, which `FILTER` likewise never kept.

`MatchLookup!K2:K` is the same length as the `MatchLookup!$A$2:$A` and
`$B$2:$B` ranges it is filtered beside, as `FILTER` requires.

### 3.3 The matchup family reads `MatchLookup`'s rows

`MatchLookup` row *r* already holds, for match `A`*r*, its matchup (`B`), both
codes (`C`, `G`) and both scores (`F` = `GETTEAM1SCOREBYMATCH(A`*r*`)`,
`J` = `GETTEAM2SCOREBYMATCH(A`*r*`)`). So "the scores of team *t*'s matches in
matchup *m*" is one `FILTER` of `F` or `J` over the rows whose `B` contains
*m* and whose base code is *t*, instead of a match-number list followed by a
`MAP` of per-match lookups. The six functions become:

| Function | Side | Rows kept | Returns |
| --- | --- | --- | --- |
| `GETMATCHRESULTSBYMATCHUPANDTEAMCODE1` | team 1 | `K = teamcode` | `""` if either score is empty, else `F > J` |
| `GETMATCHRESULTSBYMATCHUPANDTEAMCODE2` | team 2 | `L = teamcode` | `""` if either score is empty, else `J > F` |
| `GETMATCHSCORESBYMATCHUPANDTEAMCODE1` | team 1 | `K = teamcode` | `F` |
| `GETMATCHSCORESBYMATCHUPANDTEAMCODE2` | team 2 | `L = teamcode` | `J` |
| `GETOPPONENTSCORESBYMATCHUPANDTEAMCODE1` | team 1 | `K = teamcode` | `J` |
| `GETOPPONENTSCORESBYMATCHUPANDTEAMCODE2` | team 2 | `L = teamcode` | `F` |

Each keeps its arguments (`matchup, teamcode`) and is wrapped in `IFNA(…, "")`,
which returns `""` when no row matches, as `IF(ISNA(list), "", …)` did.

**Unplayed is `= ""`, not `ISBLANK`.** `F` and `J` are formula cells, and
`ISBLANK` on a formula cell is unreliable: it is false whenever the formula
returns an empty string. `= ""` is true for an
empty value and for an empty string, and false for a score of `0` (`0 = ""`
is `FALSE` in Sheets), so played and unplayed are told apart as before.

**Why the results don't change.** The rows kept are the rows the old match
list came from (same `B` test, same base-code test), in the same order, and
each row's `F`/`J` is by definition the score the old `MAP` looked up for
that row's match number. Every cell that calls the family consumes it through
`SUM` or `COUNTIF` (the wrappers `GETMATCHRESULTS…`, `GETMATCHSCORES…`,
`GETOPPONENTSCORES…` stack both sides with `{…;…}`;
`GETPAIRWINS…`/`GETPAIRLOSES…` count `TRUE`/`FALSE`), so only the values
matter, not the array's shape. The wrappers and
`GETPAIRWINS…`/`GETPAIRLOSES…` don't change.

### 3.4 What recalculates afterwards

| Edit | Recalculates |
| --- | --- |
| A score in `SCHEDULE` (almost every edit on event day) | `StackCache!D` or `E`; the 584 `GETTEAM1SCOREBYMATCH`/`…2…` cells in `CSV` and `MatchLookup`; the matchup family's cells in `Standings` and `STANDINGSCSV`; what reads those (`FINAL RANK`, `Awards`, `Court Control`'s totals). Not codes, names, matchup titles, times or courts |
| A team code in `SCHEDULE` (playoff seeding: rare) | `StackCache!A`–`C` and most of `CSV` and `MatchLookup`, names included, plus `Timeline`. About what every edit costs today, without the 9,200 rebuilds |
| Anything outside `SCHEDULE` (attendance marks, `Court Control`'s on-court numbers) | only that cell's own dependents |

### 3.5 Considered and not done

- **`XLOOKUP` instead of `FILTER` in `GETTEAM1CODEBYMATCH` and the like.**
  Stops at the first hit instead of scanning, but over a ~410-row cache the
  gain is small, and how it returns an empty cell would change the published
  `CSV` for unplayed matches.
- **`Standings!Q`/`R` duplicate `O`/`P`.** `GETPAIRWINSBYMATCHUPANDTEAMCODE("", N5)`
  is `COUNTIF(GETMATCHRESULTSBYMATCHUPANDTEAMCODE("", N5), TRUE)`, exactly
  `O5`, and `R5` is `P5` the same way. After §3.3 each is a few cheap
  `FILTER`s; changing 30 cell formulas isn't worth the risk here. The template
  should compute each once.
- **`GETPLAYERNAMESBYTEAMCODE`** (1,608 cells in `CSV` and `MatchLookup`, 400
  in `SCHEDULE`, each a `FILTER` concatenating two columns of
  `Reference for Players`' 2,600 rows). After §3.1 it no longer recalculates
  on score edits. Trimming `Reference for Players` and precomputing the full
  name belongs to the template.
- **Merging `CSV` and `MatchLookup`** into one table: cell-formula work on a
  finished workbook; the template's job (§7).
- **`STACKBLOCKS` in the cache** (`=STACKBLOCKS(6, 8, 6)` in `A1`): cuts
  the rebuilds to 5 but stays volatile (§3.1).
- **Raising `SHEETS_FETCH_TIMEOUT_MS`** from `0` on Cloud Run: §9.

## 4. Steps

Do them in order. Each step says what to do, what to paste, and what the
person should see (**Check**). Keyboard shortcuts are Ctrl on Windows, Cmd on
a Mac.

**Editing a named function** (steps 3 and 5) always goes the same way:

1. **Data → Named functions**. A sidebar lists them.
2. Hover the function's name, click its **⋮** menu, **Edit**.
3. Leave **Function name** and **Argument placeholders** exactly as they are.
4. Select all of **Formula definition**, delete it, and paste the new
   definition from this spec (it starts with `=`).
5. **Next**, then **Update**.
6. Wait for the progress bar at the top right to disappear.

If Sheets rejects a definition, it says so under **Formula definition**:
stop, and report the message word for word.

### Step 0. Prepare

1. Open the workbook
   `https://docs.google.com/spreadsheets/d/1Bk3iAqB6Fdt6t-EiI8MrU0Or8UZc4MAZcnWcMUOCkOA/edit`
   (**2026-10-03 PickleDrive Club One Year Celebration**), signed in as an
   account that can edit it.
2. **SAGE → Pause live sync.** If the menu shows **Resume live sync**
   instead, it is already paused; leave it.
3. **File → Version history → Name current version**, name it
   `Before named-function cleanup`, **Save**. §6 and §8 depend on it.
4. **Take the "before" export** (§6.1): either you download it through a
   Google Drive connector, or the person does **File → Download → Microsoft
   Excel (.xlsx)** and tells you where the file is. Keep it until §7 is done.
5. **Run the unplayed probe** (§6.2) and record the values the person reads
   out. They must equal the "after clearing" column there. If they don't, stop
   and report: the workbook does not behave as this spec assumes.

**Check:** the SAGE menu shows **Resume live sync**; the named version
appears in version history; the before export exists; the probe matched.

### Step 1. Trim `SCHEDULE` to its content

1. Open the `SCHEDULE` tab. Click any cell in column `A`, then press
   Ctrl+↓ repeatedly until it stops: the last row number of the grid shows on
   the left.
2. If the last row is **46**, skip to step 2.
3. Otherwise click the row header **47** (the grey number on the left), then
   Shift+click the header of the last row. Right-click the selection,
   **Delete rows 47 – N**.

Every stack reads to the bottom of the grid, so this shrinks each one.
Formulas that measure the tab (`ROWS(SCHEDULE!$B$6:$B)` in `MATCHTIME` and
`MATCHCOURT`) adjust on their own, and Sheets shrinks other references to it
(`Timeline`'s `SCHEDULE!$B$6:$B652`, `Standings`'s `…$6:…104`) to row 46 as
the rows go. Nothing references a cell below row 46 by itself.

**Check:** `SCHEDULE` ends at row 46. `CSV!D2` (match 1's time) still
shows 2:00 PM and `CSV!E2` (its court) still shows `Court 1`.

### Step 2. Add the `StackCache` tab

1. Click **+** (Add sheet) at the bottom left. Double-click the new tab's
   name and rename it `StackCache` exactly (no spaces).
2. Paste each formula below into its cell. Each spills down column by
   itself.

`A1`:

```
=VSTACK(SCHEDULE!F6:F, SCHEDULE!N6:N, SCHEDULE!V6:V, SCHEDULE!AD6:AD, SCHEDULE!AL6:AL, SCHEDULE!AT6:AT, SCHEDULE!BB6:BB, SCHEDULE!BJ6:BJ, SCHEDULE!BR6:BR, SCHEDULE!BZ6:BZ)
```

`B1`:

```
=VSTACK(SCHEDULE!E6:E, SCHEDULE!M6:M, SCHEDULE!U6:U, SCHEDULE!AC6:AC, SCHEDULE!AK6:AK, SCHEDULE!AS6:AS, SCHEDULE!BA6:BA, SCHEDULE!BI6:BI, SCHEDULE!BQ6:BQ, SCHEDULE!BY6:BY)
```

`C1`:

```
=VSTACK(SCHEDULE!G6:G, SCHEDULE!O6:O, SCHEDULE!W6:W, SCHEDULE!AE6:AE, SCHEDULE!AM6:AM, SCHEDULE!AU6:AU, SCHEDULE!BC6:BC, SCHEDULE!BK6:BK, SCHEDULE!BS6:BS, SCHEDULE!CA6:CA)
```

`D1`:

```
=VSTACK(SCHEDULE!I6:I, SCHEDULE!Q6:Q, SCHEDULE!Y6:Y, SCHEDULE!AG6:AG, SCHEDULE!AO6:AO, SCHEDULE!AW6:AW, SCHEDULE!BE6:BE, SCHEDULE!BM6:BM, SCHEDULE!BU6:BU, SCHEDULE!CC6:CC)
```

`E1`:

```
=VSTACK(SCHEDULE!J6:J, SCHEDULE!R6:R, SCHEDULE!Z6:Z, SCHEDULE!AH6:AH, SCHEDULE!AP6:AP, SCHEDULE!AX6:AX, SCHEDULE!BF6:BF, SCHEDULE!BN6:BN, SCHEDULE!BV6:BV, SCHEDULE!CD6:CD)
```

3. If a cell shows `#REF!` with the message *Result was not automatically
   expanded, please insert more rows*, add that many rows at the bottom of the
   tab (the **Add [n] more rows at bottom** button after the last row).
4. Paste these five checks into `G1`:`K1`. Each compares one cache column
   with the array the old primitive built, value for value:

`G1`:

```
=LET(old, STACKBLOCKS(6, 8, 6), n, ROWS(old), n & " rows, " & SUMPRODUCT(ARRAYFORMULA(--(TO_TEXT(old) <> TO_TEXT(ARRAY_CONSTRAIN(A1:A, n, 1))))) & " differ")
```

`H1`:

```
=LET(old, STACKBLOCKS(5, 8, 6), n, ROWS(old), n & " rows, " & SUMPRODUCT(ARRAYFORMULA(--(TO_TEXT(old) <> TO_TEXT(ARRAY_CONSTRAIN(B1:B, n, 1))))) & " differ")
```

`I1`:

```
=LET(old, STACKBLOCKS(7, 8, 6), n, ROWS(old), n & " rows, " & SUMPRODUCT(ARRAYFORMULA(--(TO_TEXT(old) <> TO_TEXT(ARRAY_CONSTRAIN(C1:C, n, 1))))) & " differ")
```

`J1`:

```
=LET(old, STACKBLOCKS(9, 8, 6), n, ROWS(old), n & " rows, " & SUMPRODUCT(ARRAYFORMULA(--(TO_TEXT(old) <> TO_TEXT(ARRAY_CONSTRAIN(D1:D, n, 1))))) & " differ")
```

`K1`:

```
=LET(old, STACKBLOCKS(10, 8, 6), n, ROWS(old), n & " rows, " & SUMPRODUCT(ARRAYFORMULA(--(TO_TEXT(old) <> TO_TEXT(ARRAY_CONSTRAIN(E1:E, n, 1))))) & " differ")
```

**Check:** all five read `410 rows, 0 differ` (10 courts × 41 rows, rows 6–46).
A different row count with `0 differ` everywhere is fine: `SCHEDULE` is wider
or taller than §0.3 found, and the cache still matches value for value.
**Any non-zero "differ", or an error in a check cell, means stop**: delete
the tab (right-click it, **Delete**) and report the five readings.

5. Delete `G1:K1` (select them, press Delete). Then right-click the
   `StackCache` tab, **Hide sheet**.

### Step 3. Point the five primitives at the cache

Edit each (the steps at the top of §4). They have no argument placeholders.

| Function | New formula definition |
| --- | --- |
| `GETMATCHNUMBERS` | `=StackCache!$A$1:$A` |
| `GETPLAYERSCOLUMN1` | `=StackCache!$B$1:$B` |
| `GETPLAYERSCOLUMN2` | `=StackCache!$C$1:$C` |
| `GETSCORESCOLUMN1` | `=StackCache!$D$1:$D` |
| `GETSCORESCOLUMN2` | `=StackCache!$E$1:$E` |

Leave `STACKBLOCKS` defined: nothing calls it after this step, so it costs
nothing, and it is still the library's definition (Appendix A.3).

Between the first and the last of these five edits, `CSV` and `MatchLookup`
fill with errors: until all five read the cache, the columns they combine
are different lengths. That is expected; carry on to the last one.

**Check:** after the last one, `CSV` row 2 reads `1`, `A v B` (`A2:B2`),
2:00 PM, `Court 1` (`D2:E2`), `A_1`, `Jello Miranda`, `Errol Cunanan`, `2`
(`F2:I2`), `B_1`, `Jr Pineda`, `Jr Cancio`, `11` (`J2:M2`). `MatchLookup!F2`
is `2` and `J2` is `11`. No `#REF!`, `#NAME?`, `#VALUE!` or `#N/A` in `CSV`
or `MatchLookup`.

### Step 4. Add the base-code columns to `MatchLookup`

`MatchLookup` is hidden: **View → Hidden sheets → MatchLookup** shows it.

1. If the sheet has no column `K` (the last column is `J`), right-click the
   `J` column header, **Insert 1 column right**, twice.
2. `K1`: type `teamBase1`. `L1`: type `teamBase2`.
3. `K2`:

```
=ARRAYFORMULA(IFERROR(REGEXEXTRACT(TO_TEXT(C2:C), "^_*([^_]+)"), ""))
```

4. `L2`:

```
=ARRAYFORMULA(IFERROR(REGEXEXTRACT(TO_TEXT(G2:G), "^_*([^_]+)"), ""))
```

**Check:** `K2` reads `A` and `L2` reads `B` (match 1 is `A_1` v `B_1`).
`K141` reads `SF-J` and `L141` reads `SF-C`. `K142` and below are empty.
Leave `MatchLookup` visible until step 7; hide it again there.

### Step 5. Rewrite the six matchup-family functions

Edit each (the steps at the top of §4). Each keeps its argument placeholders,
`matchup` and `teamcode`.

`GETMATCHRESULTSBYMATCHUPANDTEAMCODE1`:

```
=IFNA(FILTER(ARRAYFORMULA(IF((MatchLookup!$F$2:$F = "") + (MatchLookup!$J$2:$J = ""), "", MatchLookup!$F$2:$F > MatchLookup!$J$2:$J)), ISNUMBER(SEARCH(matchup, MatchLookup!$B$2:$B)), MatchLookup!$K$2:$K = teamcode), "")
```

`GETMATCHRESULTSBYMATCHUPANDTEAMCODE2`:

```
=IFNA(FILTER(ARRAYFORMULA(IF((MatchLookup!$J$2:$J = "") + (MatchLookup!$F$2:$F = ""), "", MatchLookup!$J$2:$J > MatchLookup!$F$2:$F)), ISNUMBER(SEARCH(matchup, MatchLookup!$B$2:$B)), MatchLookup!$L$2:$L = teamcode), "")
```

`GETMATCHSCORESBYMATCHUPANDTEAMCODE1`:

```
=IFNA(FILTER(MatchLookup!$F$2:$F, ISNUMBER(SEARCH(matchup, MatchLookup!$B$2:$B)), MatchLookup!$K$2:$K = teamcode), "")
```

`GETMATCHSCORESBYMATCHUPANDTEAMCODE2`:

```
=IFNA(FILTER(MatchLookup!$J$2:$J, ISNUMBER(SEARCH(matchup, MatchLookup!$B$2:$B)), MatchLookup!$L$2:$L = teamcode), "")
```

`GETOPPONENTSCORESBYMATCHUPANDTEAMCODE1`:

```
=IFNA(FILTER(MatchLookup!$J$2:$J, ISNUMBER(SEARCH(matchup, MatchLookup!$B$2:$B)), MatchLookup!$K$2:$K = teamcode), "")
```

`GETOPPONENTSCORESBYMATCHUPANDTEAMCODE2`:

```
=IFNA(FILTER(MatchLookup!$F$2:$F, ISNUMBER(SEARCH(matchup, MatchLookup!$B$2:$B)), MatchLookup!$L$2:$L = teamcode), "")
```

Leave the wrappers (`GETMATCHRESULTSBYMATCHUPANDTEAMCODE`,
`GETMATCHSCORESBYMATCHUPANDTEAMCODE`, `GETOPPONENTSCORESBYMATCHUPANDTEAMCODE`,
`GETPAIRWINSBYMATCHUPANDTEAMCODE`, `GETPAIRLOSESBYMATCHUPANDTEAMCODE`) as
they are.

**Check:** `Standings` row 5 still reads `A v B`, `A`, `1`, `25`, `B`, `3`,
`39`, winner `B` (`C5:K5`), and team `A` (`N5`) reads `9`, `7`, `9`, `7`
(`O5:R5`), `141`, `126` (`U5:V5`). `STANDINGSCSV!C2:D2` is `141`, `126`.

### Step 6. Remove the 15 unused functions

**Data → Named functions**; for each name below, its **⋮** menu,
**Remove**, and confirm.

`COUNTPAIRAT`, `GETMATCHES`, `GETMATCHESBYMATCHUPANDTEAMCODE1`,
`GETMATCHESBYMATCHUPANDTEAMCODE2`, `GETMATCHRESULTSBYTEAMCODE`,
`GETPLAYERNAMES`, `GETSCOREAGAINSTPAIR`, `GETSCOREQUOTIENT`,
`GETTEAMCODES`, `GETTOTALLOSES`, `GETTOTALOPPONENTSCORE`, `GETTOTALSCORE`,
`GETTOTALWINS`, `GETWINSCORE`, `SORTBYWINS`

**Check:** the sidebar lists 25 functions. No cell anywhere shows `#NAME?`:
**Edit → Find and replace** (Ctrl+H), **Find** `#NAME?`, **Search** set to
**All sheets**, **Also search within formulas** unticked, **Find**: Sheets
reports nothing found. Close the dialog without replacing anything.

### Step 7. Verify

1. Hide `MatchLookup` again (right-click its tab, **Hide sheet**).
2. Run the unplayed probe again (§6.2). The values must equal step 0's.
3. Take the "after" export the same way as step 0's, and run the comparison
   (§6.1). It must print `IDENTICAL`.

**Check:** both pass. If either fails, stop: report the differences, and
roll back (§8) unless the person decides otherwise.

### Step 8. Optional cleanup (§2.4)

Ask the person whether they want any of these; none affects the published
tabs.

- **Broken named ranges:** **Data → Named ranges**; remove `Player`,
  `Bracket`, `Team`, `Level` and `Number` if listed. They are sheet-scoped
  and may not show there; if not, leave them.
- **`Timeline (Individual}`:** nothing reads it. Delete it (right-click the
  tab, **Delete**), or keep it as a record of the schedule check.
- **`Timeline`:** for a finished event it can go. First replace
  `Court Control!K50` with its value (select `K50`, Ctrl+C, then
  **Edit → Paste special → Values only**), then delete `Timeline`.

If anything was deleted, take another export and run §6.1 again against the
"before" export: it must still print `IDENTICAL`.

### Step 9. Resume and measure

1. **SAGE → Resume live sync.**
2. **SAGE → Sync now**, three times, about 10 s apart. Each one publishes the
   same data again and commits it to `event-data`.
3. Read the timings (§6.3).

**Check:** report the three `fetch=` figures. The target is Piggleball's
0.3–0.4 s. Figures still in the seconds mean the change didn't fix the reads:
report them, but don't roll back, since §6.1 has already shown the values are
unchanged.

## 5. Expected effect

`STACKBLOCKS` goes from about 9,200 rebuilds per edit to none: the cache is
five `VSTACK`s over 410 rows, recalculated only when a cell they cover
changes. What a score edit recalculates is §3.4's first row: plain `FILTER`s
over the cache and over `MatchLookup`'s 290 rows. How much of the 10–58 s
that removes is not known until §6.3; the Piggleball workbook's reads, with
no matchup family and the same `INDIRECT` stacks, were about 0.3 s.

## 6. Verification

### 6.1 Same values, before and after

Two `.xlsx` exports: "before" (step 0) and "after" (step 7). To export
through a Google Drive connector, download file
`1Bk3iAqB6Fdt6t-EiI8MrU0Or8UZc4MAZcnWcMUOCkOA` with export MIME type
`application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`; the
content comes back base64-encoded, so decode it to a `.xlsx` file. Otherwise
the person downloads it (**File → Download → Microsoft Excel (.xlsx)**).

Unzip each into its own folder (Bash):

```bash
unzip -q before.xlsx -d before && unzip -q after.xlsx -d after
```

Save the script below as `compare-exports.mjs` in your scratchpad (not in a
repo) and run `node compare-exports.mjs before after`. It compares the cached
value of every cell on the tabs that matter (the two published tabs, the tabs
that feed them, and `SCHEDULE`), ignoring `Court Control`'s live clock (`M50`
and the cells computed from it) and the new `MatchLookup!K:L`. It prints each
difference and ends with `IDENTICAL` or a count. Numbers are compared as
numbers, so `11` and `11.0` match.

```js
// Usage: node compare-exports.mjs <before-dir> <after-dir>
// Each dir is an unzipped .xlsx export of the workbook. Compares the cached
// value of every cell on the checked tabs and prints each difference.
import fs from "fs";
import path from "path";

const TABS = ["CSV", "STANDINGSCSV", "Standings", "MatchLookup", "Court Control", "SCHEDULE", "FINAL RANK", "Awards"];
// Court Control's live clock (NOW() in M50) and the cells computed from it.
const IGNORE = { "Court Control": new Set(["M50", "M51", "M52", "M53", "M54", "M55"]) };
// Columns the change adds; they don't exist in the before export.
const IGNORE_COLS = { MatchLookup: new Set(["K", "L"]) };

const un = s => s.replace(/&quot;/g, '"').replace(/&lt;/g, "<").replace(/&gt;/g, ">").replace(/&apos;/g, "'").replace(/&amp;/g, "&");

function load(dir) {
  const x = p => fs.readFileSync(path.join(dir, "xl", p), "utf8");
  const ssPath = path.join(dir, "xl", "sharedStrings.xml");
  const ss = fs.existsSync(ssPath)
    ? [...fs.readFileSync(ssPath, "utf8").matchAll(/<si>([\s\S]*?)<\/si>/g)].map(m => un(m[1].replace(/<rPh[\s\S]*?<\/rPh>/g, "").replace(/<[^>]+>/g, "")))
    : [];
  const rels = {};
  for (const m of x("_rels/workbook.xml.rels").matchAll(/<Relationship ([^>]*)\/>/g)) {
    rels[/Id="([^"]+)"/.exec(m[1])[1]] = /Target="([^"]+)"/.exec(m[1])[1].replace(/^\/?xl\//, "");
  }
  const tabs = {};
  for (const m of x("workbook.xml").matchAll(/<sheet ([^>]*)\/>/g)) {
    const name = un(/name="([^"]+)"/.exec(m[1])[1]);
    if (!TABS.includes(name)) continue;
    const cells = new Map();
    for (const c of x(rels[/r:id="([^"]+)"/.exec(m[1])[1]]).matchAll(/<c r="([A-Z]+)(\d+)"([^>]*?)(?:\/>|>([\s\S]*?)<\/c>)/g)) {
      const body = c[4] || "";
      let v = null;
      const is = /<is>([\s\S]*?)<\/is>/.exec(body);
      const vv = /<v>([\s\S]*?)<\/v>/.exec(body);
      if (is) v = un(is[1].replace(/<[^>]+>/g, ""));
      else if (vv) v = /t="s"/.test(c[3]) ? ss[+vv[1]] : un(vv[1]);
      if (v !== null && v !== "") cells.set(c[1] + c[2], v);
    }
    tabs[name] = cells;
  }
  return tabs;
}

const [a, b] = process.argv.slice(2).map(load);
let total = 0;
for (const tab of TABS) {
  if (!a[tab] || !b[tab]) { console.log(`MISSING TAB ${tab} (before: ${!!a[tab]}, after: ${!!b[tab]})`); total++; continue; }
  const keys = new Set([...a[tab].keys(), ...b[tab].keys()]);
  let n = 0;
  for (const k of [...keys].sort()) {
    if (IGNORE[tab]?.has(k) || IGNORE_COLS[tab]?.has(k.replace(/\d+/g, ""))) continue;
    const va = a[tab].get(k) ?? "", vb = b[tab].get(k) ?? "";
    if (va !== vb && !(Number(va) === Number(vb) && va !== "" && vb !== "" && !isNaN(Number(va)))) {
      if (n < 25) console.log(`${tab}!${k}: before=${JSON.stringify(va)} after=${JSON.stringify(vb)}`);
      n++;
    }
  }
  console.log(`${tab}: ${n} difference(s) across ${keys.size} cells`);
  total += n;
}
console.log(total === 0 ? "IDENTICAL" : `${total} DIFFERENCE(S)`);
process.exit(total === 0 ? 0 : 1);
```

The script was run on 2026-10-05 against two copies of the workbook's
export: identical copies print `IDENTICAL`; one altered cached value prints
that cell and `1 DIFFERENCE(S)`.

`CSV` and `STANDINGSCSV` are what the sync publishes, so a difference there
is a difference on the site. Any difference means a step changed behaviour:
roll back (§8) rather than fixing forward. Named functions can't be checked
any other way: the generator verify scripts assert formula text and never
evaluate it.

### 6.2 The unplayed probe

Every match in the finished workbook has both scores, so the export
comparison never exercises "unplayed", the one place §3.3 changes a test
(`ISBLANK` to `= ""`). The probe does, by clearing one score and undoing it.
Live sync must be paused (it is, from step 0 to step 9).

1. In `SCHEDULE`, click `I6` (match 1, court 1: team 1's score, `2`; team 2's
   in `J6` is `11`). Press Delete.
2. Wait for the progress bar to go, then have the person read out these
   cells:

| Cell | What it is | With `I6` = 2 | After clearing `I6` |
| --- | --- | --- | --- |
| `Standings!E5` | A's pair wins in `A v B` | 1 | **1** |
| `Standings!F5` | A's points in `A v B` | 25 | **23** |
| `Standings!I5` | B's pair wins in `A v B` | 3 | **2** |
| `Standings!J5` | B's points in `A v B` | 39 | **39** |
| `Standings!K5` | `A v B` winner | B | **(empty)** |
| `Standings!O5` / `P5` | team A wins / losses | 9 / 7 | **9 / 6** |
| `Standings!U5` / `V5` | team A points for / against | 141 / 126 | **139 / 126** |
| `Standings!O6` / `P6` | team B wins / losses | 11 / 5 | **10 / 5** |
| `Standings!U6` / `V6` | team B points for / against | 153 / 117 | **153 / 115** |
| `STANDINGSCSV!C2` / `D2` | team A points for / against | 141 / 126 | **139 / 126** |
| `STANDINGSCSV!C3` / `D3` | team B points for / against | 153 / 117 | **153 / 115** |
| `CSV!I2` | match 1's team 1 score | 2 | **(empty)** |

   An unplayed match counts as neither a win nor a loss and adds no points.
   If `P5` stays `7` or `O6` stays `11`, the workbook counts an empty score as
   a played match: stop and report.
3. **Edit → Undo** (Ctrl+Z) until `I6` shows `2` again, and confirm
   `Standings!P5` is back to `7`.

### 6.3 Faster

Read step 9's syncs from Cloud Run (PowerShell, `gcloud` signed in):

```
gcloud logging read 'resource.type=cloud_run_revision AND resource.labels.service_name=sage-tools-api AND textPayload:\"timing edit\" AND textPayload:\"pickledrive\"' --freshness=1h --format='value(timestamp,textPayload)' --project=sage-tools-api
```

**Sync now** sends no edit time, so `edit→request` and `edit→published` read
`n/a`; `fetch=` is the figure that matters. Compare it with the event day's
p50 0.3 s / p90 22.6 s. Three syncs are a spot check, not a distribution;
every one is a commit of unchanged data to `event-data`.

## 7. Afterwards

- **Docs.** Add a team-workbook section to
  [The Named Function library](../../technical/named-function-library.md):
  the `…BYMATCHUPANDTEAMCODE` family as rewritten here, the
  `GETTEAM1SCOREBYMATCH`/`…2…` and `GETMATCHUPTITLE` functions,
  `MatchLookup!K:L`, and `StackCache`, including that the five primitives
  read it, that it is built from direct references, and why that is safe in a
  team workbook (§3.1). Record the measured effect beside the event-day
  figures in
  [Sync pipeline](../../technical/sync-pipeline.md#measured-at-two-events-3-october-2026),
  and update the
  [Team tournament](pickledrive-club-anniversary-team-tournament-spec.md)
  spec's status note. In
  [Control Center score entry](../in-progress/control-center-score-entry-spec.md)
  §0.4, the row saying team workbooks test "unplayed" with `ISBLANK` becomes
  `= ""`; a cleared score still has to be a truly empty cell for the
  standard and dual-meet libraries. Then move this spec to `implemented/`
  by the procedure in [`docs/specs/README.md`](../README.md).
- **The team template ([its spec](../not-started/team-tournament-template-spec.md)) and the
  generator ([its spec](../not-started/team-tournament-master-spec.md))** start from the
  optimised workbook and ship with the cache from the start, written for the
  event's court count, `SCHEDULE` trimmed to its real height, and no unused
  functions. `CSV` and `MatchLookup` should become one table, not two
  computing the same columns; `Standings!Q:R` should not recompute `O:P`; the
  `Timeline` tabs belong to schedule planning, before the event, and in a live
  workbook they should not recalculate on code edits.
- **The other libraries.** The standard and dual-meet masters run the same
  `INDIRECT` design, and their workbooks were fast on the day (Piggleball
  p90 0.4 s), so they are not part of this spec. If one grows slow, §3.1
  applies to it with that workbook's own block columns.

## 8. Rollback

**File → Version history → See version history**, click
`Before named-function cleanup`, **Restore this version**. That restores the
functions, the rows and the tabs together. Live sync must be paused while
you do it (it is, until step 9; after step 9, pause it first and resume
after). To restore a single function by hand, its definition is in
Appendix A.

## 9. Out of scope

- Any change to `sage-tools-api`, the sync scripts or the site.
- Raising `SHEETS_FETCH_TIMEOUT_MS` from `0` on Cloud Run. Bounding the read
  helps whatever the workbook does, and is a separate decision.
- Any cell formula outside `StackCache` and `MatchLookup!K:L` (§3.5).
- The event's results: §6 exists to prove they don't change.

---

## Appendix A: every named function, as exported on 2026-10-05

All 40, from the export's `<definedNames>`. Each is
`LAMBDA(<arguments>, <body>)`; in **Data → Named functions**, the arguments
go in **Argument placeholders** and the body, with a leading `=`, in
**Formula definition**. `matchUp`/`teamCode`/`matchNumber` inside a body are
the `matchup`/`teamcode`/`matchnumber` arguments (Sheets names are
case-insensitive).

### A.1 Changed by step 3 or step 5 (11)

| Function | Definition before this spec |
| --- | --- |
| `GETMATCHNUMBERS` | `LAMBDA(STACKBLOCKS(6,8,6))` |
| `GETPLAYERSCOLUMN1` | `LAMBDA(STACKBLOCKS(5, 8, 6))` |
| `GETPLAYERSCOLUMN2` | `LAMBDA(STACKBLOCKS(7, 8, 6))` |
| `GETSCORESCOLUMN1` | `LAMBDA(STACKBLOCKS(9, 8, 6))` |
| `GETSCORESCOLUMN2` | `LAMBDA(STACKBLOCKS(10, 8, 6))` |
| `GETMATCHRESULTSBYMATCHUPANDTEAMCODE1` | `LAMBDA(matchup, teamcode, IF(ISNA(GETMATCHESBYMATCHUPANDTEAMCODE1(matchUp, teamCode)),"",MAP(GETMATCHESBYMATCHUPANDTEAMCODE1(matchUp, teamCode),LAMBDA(mn,IF(OR(ISBLANK(GETTEAM1SCOREBYMATCH(mn)),ISBLANK(GETTEAM2SCOREBYMATCH(mn))), "", GETTEAM1SCOREBYMATCH(mn) > GETTEAM2SCOREBYMATCH(mn))))))` |
| `GETMATCHRESULTSBYMATCHUPANDTEAMCODE2` | `LAMBDA(matchup, teamcode, IF(ISNA(GETMATCHESBYMATCHUPANDTEAMCODE2(matchUp, teamCode)),"",MAP(GETMATCHESBYMATCHUPANDTEAMCODE2(matchUp, teamCode),LAMBDA(mn,IF(OR(ISBLANK(GETTEAM1SCOREBYMATCH(mn)),ISBLANK(GETTEAM2SCOREBYMATCH(mn))), "", GETTEAM2SCOREBYMATCH(mn) > GETTEAM1SCOREBYMATCH(mn))))))` |
| `GETMATCHSCORESBYMATCHUPANDTEAMCODE1` | `LAMBDA(matchup, teamcode, IF(ISNA(GETMATCHESBYMATCHUPANDTEAMCODE1(matchUp, teamCode)),"",MAP(GETMATCHESBYMATCHUPANDTEAMCODE1(matchUp, teamCode),LAMBDA(mn,GETTEAM1SCOREBYMATCH(mn)))))` |
| `GETMATCHSCORESBYMATCHUPANDTEAMCODE2` | `LAMBDA(matchup, teamcode, IF(ISNA(GETMATCHESBYMATCHUPANDTEAMCODE2(matchUp, teamCode)),"",MAP(GETMATCHESBYMATCHUPANDTEAMCODE2(matchUp, teamCode),LAMBDA(mn,GETTEAM2SCOREBYMATCH(mn)))))` |
| `GETOPPONENTSCORESBYMATCHUPANDTEAMCODE1` | `LAMBDA(matchup, teamcode, IF(ISNA(GETMATCHESBYMATCHUPANDTEAMCODE1(matchUp, teamCode)),"",MAP(GETMATCHESBYMATCHUPANDTEAMCODE1(matchUp, teamCode),LAMBDA(mn,GETTEAM2SCOREBYMATCH(mn)))))` |
| `GETOPPONENTSCORESBYMATCHUPANDTEAMCODE2` | `LAMBDA(matchup, teamcode, IF(ISNA(GETMATCHESBYMATCHUPANDTEAMCODE2(matchUp, teamCode)),"",MAP(GETMATCHESBYMATCHUPANDTEAMCODE2(matchUp, teamCode),LAMBDA(mn,GETTEAM1SCOREBYMATCH(mn)))))` |

### A.2 Removed by step 6 because §4 retires them (2)

| Function | Definition |
| --- | --- |
| `GETMATCHESBYMATCHUPANDTEAMCODE1` | `LAMBDA(matchup, teamcode, FILTER(MatchLookup!$A$2:$A,ISNUMBER(SEARCH(matchUp,MatchLookup!$B$2:$B)), MAP(MatchLookup!$C$2:$C,LAMBDA(tc,INDEX(SPLIT(tc,"_",1),1)))=teamCode))` |
| `GETMATCHESBYMATCHUPANDTEAMCODE2` | `LAMBDA(matchup, teamcode, FILTER(MatchLookup!$A$2:$A,ISNUMBER(SEARCH(matchUp,MatchLookup!$B$2:$B)), MAP(MatchLookup!$G$2:$G,LAMBDA(tc,INDEX(SPLIT(tc,"_",1),1)))=teamCode))` |

### A.3 Unchanged (14)

`STACKBLOCKS`:

```
LAMBDA(first_col, step, first_row, LET(
  n,   INT((COLUMNS(SCHEDULE!$1:$1) - first_col) / step) + 1,
  ref, LAMBDA(c, LET(L, SUBSTITUTE(ADDRESS(1, c, 4), "1", ""),
                     INDIRECT("SCHEDULE!" & L & first_row & ":" & L))),
  REDUCE(ref(first_col), SEQUENCE(n - 1, 1, first_col + step, step),
         LAMBDA(acc, c, VSTACK(acc, ref(c))))
))
```

`MATCHTIME`:

```
LAMBDA(match_number, IFERROR(
  LET(
    h,   ROWS(SCHEDULE!$B$6:$B),
    pos, MATCH(match_number, GETMATCHNUMBERS(), 0),
    INDEX(SCHEDULE!$B$6:$B, MOD(pos - 1, h) + 1)
  ),
  "Not found"
))
```

`MATCHCOURT`:

```
LAMBDA(match_number, IFERROR(
  LET(
    h,     ROWS(SCHEDULE!$B$6:$B),
    pos,   MATCH(match_number, GETMATCHNUMBERS(), 0),
    blk,   INT((pos - 1) / h),
    label, INDEX(SCHEDULE!$5:$5, 1, 6 + 8 * blk),
    "Court " & REGEXEXTRACT(TO_TEXT(label), "\d+")
  ),
  "Not found"
))
```

| Function | Definition |
| --- | --- |
| `GETTEAM1CODEBYMATCH` | `LAMBDA(matchnumber, IFNA(FILTER(GETPLAYERSCOLUMN1(), GETMATCHNUMBERS()=matchNumber),"-"))` |
| `GETTEAM2CODEBYMATCH` | `LAMBDA(matchnumber, IFNA(FILTER(GETPLAYERSCOLUMN2(), GETMATCHNUMBERS()=matchNumber),"-"))` |
| `GETTEAM1SCOREBYMATCH` | `LAMBDA(matchnumber, IFNA(FILTER(GETSCORESCOLUMN1(), GETMATCHNUMBERS()=matchNumber),"-"))` |
| `GETTEAM2SCOREBYMATCH` | `LAMBDA(matchnumber, IFNA(FILTER(GETSCORESCOLUMN2(), GETMATCHNUMBERS()=matchNumber),"-"))` |
| `GETMATCHUPTITLE` | `LAMBDA(team1, team2, IFNA(SPLIT(team1,"_")&CHAR(10)&" v "&CHAR(10)&SPLIT(team2,"_"),"-"))` |
| `GETPLAYERNAMESBYTEAMCODE` | `LAMBDA(matchup, teamcode, IFNA(FILTER('Reference for Players'!$C$3:$C&" "&'Reference for Players'!$D$3:$D, 'Reference for Players'!$A$3:$A = matchup, 'Reference for Players'!$B$3:$B = teamcode)))` |
| `GETMATCHRESULTSBYMATCHUPANDTEAMCODE` | `LAMBDA(matchup, teamcode, {GETMATCHRESULTSBYMATCHUPANDTEAMCODE1(matchUp,teamCode);GETMATCHRESULTSBYMATCHUPANDTEAMCODE2(matchUp,teamCode)})` |
| `GETMATCHSCORESBYMATCHUPANDTEAMCODE` | `LAMBDA(matchup, teamcode, {GETMATCHSCORESBYMATCHUPANDTEAMCODE1(matchUp,teamCode);GETMATCHSCORESBYMATCHUPANDTEAMCODE2(matchUp,teamCode)})` |
| `GETOPPONENTSCORESBYMATCHUPANDTEAMCODE` | `LAMBDA(matchup, teamcode, {GETOPPONENTSCORESBYMATCHUPANDTEAMCODE1(matchUp,teamCode);GETOPPONENTSCORESBYMATCHUPANDTEAMCODE2(matchUp,teamCode)})` |
| `GETPAIRWINSBYMATCHUPANDTEAMCODE` | `LAMBDA(matchup, teamcode, COUNTIF(GETMATCHRESULTSBYMATCHUPANDTEAMCODE(matchUp,teamCode),TRUE))` |
| `GETPAIRLOSESBYMATCHUPANDTEAMCODE` | `LAMBDA(matchup, teamcode, COUNTIF(GETMATCHRESULTSBYMATCHUPANDTEAMCODE(matchUp,teamCode),FALSE))` |

### A.4 Removed by step 6 because nothing calls them (13)

| Function | Definition |
| --- | --- |
| `COUNTPAIRAT` | `LAMBDA(time_value, pair_code, LET(col_t1, GETPLAYERSCOLUMN1(), col_t2, GETPLAYERSCOLUMN2(), courts, INT((COLUMNS(SCHEDULE!$1:$1) - 5) / 8) + 1, col_time, REDUCE(SCHEDULE!$B$6:$B, SEQUENCE(courts - 1), LAMBDA(acc, unused, VSTACK(acc, SCHEDULE!$B$6:$B))), SUMPRODUCT((col_time = time_value) * ((col_t1 = pair_code) + (col_t2 = pair_code)))))` |
| `GETMATCHES` | `LAMBDA(ArrayFormula(GETPLAYERSCOLUMN1()&GETPLAYERSCOLUMN2()))` |
| `GETMATCHRESULTSBYTEAMCODE` | `LAMBDA(teamcode, LET(col_t1, GETPLAYERSCOLUMN1(), col_t2, GETPLAYERSCOLUMN2(), col_s1, GETSCORESCOLUMN1(), col_s2, GETSCORESCOLUMN2(), my_pts, ARRAYFORMULA(IF(col_t1 = teamcode, col_s1, col_s2)), opp_pts, ARRAYFORMULA(IF(col_t1 = teamcode, col_s2, col_s1)), is_mine, ARRAYFORMULA((col_t1 = teamcode) + (col_t2 = teamcode)), played, ARRAYFORMULA(ISNUMBER(col_s1) * ISNUMBER(col_s2)), IFERROR(FILTER(ARRAYFORMULA(my_pts > opp_pts), ARRAYFORMULA((is_mine > 0) * played)), "")))` |
| `GETPLAYERNAMES` | `LAMBDA({'Reference for Players'!$B$3:$B})` |
| `GETSCOREAGAINSTPAIR` | `LAMBDA(pair1, pair2, LET(m, GETMATCHES(), IFERROR(IFERROR(INDEX(GETSCORESCOLUMN1(), MATCH(CONCAT(pair1, pair2), m, 0)), INDEX(GETSCORESCOLUMN2(), MATCH(CONCAT(pair2, pair1), m, 0))), "No Match Found")))` |
| `GETSCOREQUOTIENT` | `LAMBDA(paircode, LET(col_t1, GETPLAYERSCOLUMN1(), col_t2, GETPLAYERSCOLUMN2(), col_s1, GETSCORESCOLUMN1(), col_s2, GETSCORESCOLUMN2(), pts_for, IFERROR(SUM(FILTER(col_s1, col_t1 = paircode)), 0) + IFERROR(SUM(FILTER(col_s2, col_t2 = paircode)), 0), pts_agst, IFERROR(SUM(FILTER(col_s2, col_t1 = paircode)), 0) + IFERROR(SUM(FILTER(col_s1, col_t2 = paircode)), 0), IFERROR(pts_for / pts_agst, 0)))` |
| `GETTEAMCODES` | `LAMBDA({'Reference for Players'!$A$3:$A})` |
| `GETTOTALLOSES` | `LAMBDA(paircode, COUNTIF(GETMATCHRESULTSBYTEAMCODE(paircode), FALSE))` |
| `GETTOTALOPPONENTSCORE` | `LAMBDA(paircode, IFERROR(SUM(FILTER(GETSCORESCOLUMN2(),GETPLAYERSCOLUMN1()=paircode)),0) + IFERROR(SUM(FILTER(GETSCORESCOLUMN1(),GETPLAYERSCOLUMN2()=paircode)),0))` |
| `GETTOTALSCORE` | `LAMBDA(paircode, IFERROR(SUM(FILTER(GETSCORESCOLUMN1(),GETPLAYERSCOLUMN1()=paircode)),0) + IFERROR(SUM(FILTER(GETSCORESCOLUMN2(),GETPLAYERSCOLUMN2()=paircode)),0))` |
| `GETTOTALWINS` | `LAMBDA(paircode, COUNTIF(GETMATCHRESULTSBYTEAMCODE(paircode), TRUE))` |
| `GETWINSCORE` | `LAMBDA(11)` |
| `SORTBYWINS` | `LAMBDA(range, INDEX(SORT(range, 4, FALSE, 8, FALSE), 0, 1))` |
