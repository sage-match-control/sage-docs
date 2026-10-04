# Spec — Team workbook recalculation (StackCache and named-function cleanup)

> **Status: not started.** Nothing here has been applied. Written 2026-10-05
> against an `.xlsx` export of the PickleDrive workbook,
> **2026-10-03 PickleDrive Club One Year Celebration**
> (`1Bk3iAqB6Fdt6t-EiI8MrU0Or8UZc4MAZcnWcMUOCkOA`, owned by
> `sagematchcontrol@gmail.com`), as it stood after the event: last modified
> 2026-10-03 13:36 UTC.
>
> **Done by hand, in the workbook.** Named functions cannot be created,
> edited or deleted through the Sheets API, so every step in §4 is a person
> in the Sheets UI. Nothing in any repo changes until §7.

Make the team workbook fast to read, so a sync's Sheets API read stops
waiting 10–58 s for it to recalculate. Do it by building the workbook's five
stacked `SCHEDULE` columns **once**, in a hidden tab, and pointing every named
function at that tab instead of rebuilding the stacks thousands of times per
edit. Remove the named functions nothing calls.

The PickleDrive workbook is the prototype the team template is extracted from
([Team tournament](../in-progress/pickledrive-club-anniversary-team-tournament-spec.md)
§15), so this is worth doing in it before that extraction, not only for its
own sake: the event is over.

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

Every named function that reads `SCHEDULE` bottoms out in `STACKBLOCKS`,
which walks `SCHEDULE`'s court blocks (period 8 from column `E`; this
workbook has 83 columns, so 10 blocks) and stacks one column of each into a
single array, through `INDIRECT`. Two things make that expensive:

- **`INDIRECT` is volatile.** Every formula that reaches `STACKBLOCKS`
  recalculates after **any** edit anywhere in the workbook, including the
  edits that fire the sync and the attendance marks.
- **The columns are open-ended** (`INDIRECT("SCHEDULE!F6:F")`), so each one
  reads to the bottom of the grid. `SCHEDULE`'s content ends at row 46, and
  every row below it is read anyway.

Counted over every formula cell in the workbook, shared-formula copies
included, assuming a team plays 4 matches per matchup, all on one side:

| Function | Cells calling it | `STACKBLOCKS` per call | Per recalculation |
| --- | --- | --- | --- |
| `GETPAIRWINSBYMATCHUPANDTEAMCODE` | 91 | 32 | 2,912 |
| `GETMATCHSCORESBYMATCHUPANDTEAMCODE` | 130 | 8 | 1,040 |
| `GETMATCHRESULTSBYMATCHUPANDTEAMCODE` | 30 | 32 | 960 |
| `GETTEAM1CODEBYMATCH` / `GETTEAM2CODEBYMATCH` | 310 each | 2 | 1,240 |
| `GETTEAM1SCOREBYMATCH` / `GETTEAM2SCOREBYMATCH` | 292 each | 2 | 1,168 |
| `GETMATCHNUMBERS` | 584 | 1 | 584 |
| `GETOPPONENTSCORESBYMATCHUPANDTEAMCODE` | 62 | 8 | 496 |
| `GETPAIRLOSESBYMATCHUPANDTEAMCODE` | 15 | 32 | 480 |
| `MATCHTIME`, `MATCHCOURT` | 152 each | 1 | 304 |
| **Total** | | | **≈ 9,200** |

By tab: `Standings` ≈ 5,400, `CSV` ≈ 1,800, `MatchLookup` ≈ 1,400,
`STANDINGSCSV` ≈ 500, `Court Control` ≈ 70. `CSV` and `MatchLookup` compute
the same per-match columns (match number, matchup, codes, names, scores).

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
  `STACKBLOCKS`;
- `GETMATCHESBYMATCHUPANDTEAMCODE1`/`2` run
  `MAP(MatchLookup!$C$2:$C, LAMBDA(tc, INDEX(SPLIT(tc, "_", 1), 1)))` over
  the whole open-ended column on every call, about 1,300 passes per
  recalculation.

`GETMATCHSCORES…`/`GETOPPONENTSCORES…` `1`/`2` compute their match list
twice the same way.

### 2.3 Named functions nobody calls

40 named functions are defined. 27 are reachable from a cell. These 13 are
not called by any cell formula, conditional format, data validation or
chart; each name appears in the export only in its own definition:

`COUNTPAIRAT`, `GETMATCHES`, `GETMATCHRESULTSBYTEAMCODE`, `GETPLAYERNAMES`,
`GETSCOREAGAINSTPAIR`, `GETSCOREQUOTIENT`, `GETTEAMCODES`, `GETTOTALLOSES`,
`GETTOTALOPPONENTSCORE`, `GETTOTALSCORE`, `GETTOTALWINS`, `GETWINSCORE`,
`SORTBYWINS`

Most are the standard/dual-meet library's standings functions, inherited by
copying and replaced in a team workbook by the `…BYMATCHUPANDTEAMCODE`
family. Their current definitions are in [Appendix A](#appendix-a-the-13-unused-definitions).

### 2.4 Not named functions, but nearby

- **Five broken named ranges**, scoped to `Teams` in the export: `Player`,
  `Bracket`, `Team`, `Level`, `Number`, all `#REF!`.
- **`Timeline (Individual}`** (the closing brace is in the tab's real name):
  8,999 formulas, 519 of them `#REF!`, and nothing reads the tab.
- **`Timeline`**: 15,910 cells of 20 `COUNTIFS` each (two code columns per
  court) over `SCHEDULE`'s code columns. Not volatile, but recalculated whenever a `SCHEDULE` code cell
  does, and playoff codes there are formulas fed from `MatchUps`. Only
  `Court Control!K50` (`=Timeline!A2`) reads it.

## 3. The design

### 3.1 `StackCache`

A hidden tab holding the five stacks, one spilled formula each:

| Cell | Formula | Holds | Replaces |
| --- | --- | --- | --- |
| `A1` | `=STACKBLOCKS(6, 8, 6)` | match numbers | `GETMATCHNUMBERS`'s body |
| `B1` | `=STACKBLOCKS(5, 8, 6)` | team 1 codes | `GETPLAYERSCOLUMN1`'s |
| `C1` | `=STACKBLOCKS(7, 8, 6)` | team 2 codes | `GETPLAYERSCOLUMN2`'s |
| `D1` | `=STACKBLOCKS(9, 8, 6)` | team 1 scores | `GETSCORESCOLUMN1`'s |
| `E1` | `=STACKBLOCKS(10, 8, 6)` | team 2 scores | `GETSCORESCOLUMN2`'s |

The five primitives then return `StackCache`'s columns instead of calling
`STACKBLOCKS`. Every function built on them keeps its name, arguments and
callers; it just reads a range. `STACKBLOCKS` itself is unchanged and is now
called five times per recalculation, not about 9,200.

**Why the results don't change.** A spilled column holds exactly the array
`STACKBLOCKS` returns, in the same court-major order, starting at row 1, so a
value's position in the cache is its position in the old array. `MATCHTIME`
and `MATCHCOURT`, which turn a position into a time slot and a court with
`ROWS(SCHEDULE!$B$6:$B)`, stay correct. The cache columns are open-ended
(`$A$1:$A`), so they run past the spill with empty rows; all five are the same
length, so the elementwise `FILTER`s that combine them still line up, and an
empty row never equals a match number or a team code.

**Circularity.** The cache reads `SCHEDULE` through `INDIRECT`, as the
functions did, so its dependencies are as dynamic as before. Nothing new
reads back into `SCHEDULE`.

### 3.2 The matchup family

- **The match list once:** wrap it in `LET` so `ISNA` and `MAP` share it.
- **Each score once:** the results functions bind both scores in a `LET`.
- **Unplayed is `= ""`, not `ISBLANK`.** A blank that comes out of a spilled
  cache cell may not read as `ISBLANK` the way one read straight from
  `SCHEDULE` did. `= ""` is true for both a blank and an empty string, and
  false for a score of `0` (`0 = ""` is `FALSE` in Sheets), so played and
  unplayed are told apart exactly as before.
- **The team-code match without `MAP`:**
  `REGEXEXTRACT(TO_TEXT(code), "^_*([^_]+)")` returns what
  `INDEX(SPLIT(code, "_", 1), 1)` did (the first non-empty part before an
  `_`: `A_1` → `A`, `SF-1_2` → `SF-1`), over the whole column in one pass.
  A blank row gives `""` through `IFERROR`, which never equals a team code.

## 4. Steps

Do them in order, in the workbook.

**0. Prepare.**

- **File → Version history → Name current version**:
  `Before named-function cleanup`. §6 compares against it and §8 rolls
  back to it.
- **SAGE → Pause live sync**, so the edits below don't each start a sync
  and commit to `event-data`.

**1. Trim `SCHEDULE`.** Delete the empty rows below row 46, keeping a few
spare. Every stack reads to the bottom of the grid, so this shrinks each one
directly. Formulas that measure the tab (`ROWS(SCHEDULE!$B$6:$B)`) adjust on
their own, and Sheets shrinks other references to it (`Timeline`'s
`SCHEDULE!$B$6:$B652`) as rows go.

**2. Add the `StackCache` tab** (the name has no spaces or leading
underscore, so references to it need no quotes). Enter §3.1's five formulas
in `A1:E1`. If a cell reports that its result was not expanded and asks for
more rows, add that many rows to the tab. Then hide the tab.

**3. Point the primitives at the cache.** In **Data → Named functions**,
edit each of these. They take no arguments.

| Function | New definition |
| --- | --- |
| `GETMATCHNUMBERS` | `=StackCache!$A$1:$A` |
| `GETPLAYERSCOLUMN1` | `=StackCache!$B$1:$B` |
| `GETPLAYERSCOLUMN2` | `=StackCache!$C$1:$C` |
| `GETSCORESCOLUMN1` | `=StackCache!$D$1:$D` |
| `GETSCORESCOLUMN2` | `=StackCache!$E$1:$E` |

Leave `STACKBLOCKS` as it is: the cache calls it.

**4. Rewrite the matchup family.** Each keeps its arguments,
`matchup, teamcode`.

`GETMATCHESBYMATCHUPANDTEAMCODE1`:

```
=FILTER(MatchLookup!$A$2:$A, ISNUMBER(SEARCH(matchup, MatchLookup!$B$2:$B)), ARRAYFORMULA(IFERROR(REGEXEXTRACT(TO_TEXT(MatchLookup!$C$2:$C), "^_*([^_]+)"), "")) = teamcode)
```

`GETMATCHESBYMATCHUPANDTEAMCODE2`: the same, with `MatchLookup!$G$2:$G` in
place of `MatchLookup!$C$2:$C`.

`GETMATCHRESULTSBYMATCHUPANDTEAMCODE1`:

```
=LET(mns, GETMATCHESBYMATCHUPANDTEAMCODE1(matchup, teamcode), IF(ISNA(mns), "", MAP(mns, LAMBDA(mn, LET(mine, GETTEAM1SCOREBYMATCH(mn), theirs, GETTEAM2SCOREBYMATCH(mn), IF(OR(mine = "", theirs = ""), "", mine > theirs))))))
```

`GETMATCHRESULTSBYMATCHUPANDTEAMCODE2`:

```
=LET(mns, GETMATCHESBYMATCHUPANDTEAMCODE2(matchup, teamcode), IF(ISNA(mns), "", MAP(mns, LAMBDA(mn, LET(mine, GETTEAM2SCOREBYMATCH(mn), theirs, GETTEAM1SCOREBYMATCH(mn), IF(OR(mine = "", theirs = ""), "", mine > theirs))))))
```

The four score functions share one shape:

```
=LET(mns, <match list>(matchup, teamcode), IF(ISNA(mns), "", MAP(mns, LAMBDA(mn, <score>(mn)))))
```

| Function | `<match list>` | `<score>` |
| --- | --- | --- |
| `GETMATCHSCORESBYMATCHUPANDTEAMCODE1` | `GETMATCHESBYMATCHUPANDTEAMCODE1` | `GETTEAM1SCOREBYMATCH` |
| `GETMATCHSCORESBYMATCHUPANDTEAMCODE2` | `GETMATCHESBYMATCHUPANDTEAMCODE2` | `GETTEAM2SCOREBYMATCH` |
| `GETOPPONENTSCORESBYMATCHUPANDTEAMCODE1` | `GETMATCHESBYMATCHUPANDTEAMCODE1` | `GETTEAM2SCOREBYMATCH` |
| `GETOPPONENTSCORESBYMATCHUPANDTEAMCODE2` | `GETMATCHESBYMATCHUPANDTEAMCODE2` | `GETTEAM1SCOREBYMATCH` |

The wrappers that combine the two sides (`GETMATCHRESULTSBYMATCHUPANDTEAMCODE`,
`GETMATCHSCORESBYMATCHUPANDTEAMCODE`, `GETOPPONENTSCORESBYMATCHUPANDTEAMCODE`,
`GETPAIRWINSBYMATCHUPANDTEAMCODE`, `GETPAIRLOSESBYMATCHUPANDTEAMCODE`) need no
change.

LET names here are letters only (`mns`, `mine`, `theirs`, `mn`): a name like
`s1` or `col1` reads as a cell reference and Sheets rejects it.

**5. Remove the 13 unused functions** listed in §2.3: **Data → Named
functions**, then each one's menu, **Remove**.

**6. Optional cleanup** (§2.4):

- Delete the five `#REF!` named ranges if they appear under **Data → Named
  ranges**. They are sheet-scoped in the export and may not show there.
- `Timeline (Individual}`: nothing reads it. Delete it, or keep it as a
  record of the schedule check.
- `Timeline`: for a finished event, safe to delete after replacing
  `Court Control!K50` with a value. For the template, see §7.

**7. Resume and measure.** **SAGE → Resume live sync**, then retype one
score with its own value. That syncs once and republishes identical data.

## 5. Expected effect

`STACKBLOCKS` goes from about 9,200 rebuilds per edit to 5, each over a
shorter `SCHEDULE`. What remains per recalculation is ordinary `FILTER`s over
the cache's five columns and the single-pass regex over `MatchLookup`. How much
of the 10–58 s that removes is not known until §6's timing check; the
Piggleball workbook's reads, which had no matchup family, were about 0.3 s.

## 6. Verification

**Same results.** Export both versions as `.xlsx`: the current one, and
`Before named-function cleanup` from version history. Compare every cell's
cached value in `CSV`, `STANDINGSCSV`, `Standings`, `MatchLookup` and
`Court Control`. They must match exactly, except `Court Control`'s live clock
(`NOW()` in `M50` and the cells derived from it). `CSV` and `STANDINGSCSV` are
what the sync publishes, so a difference there is a difference on the site.
Any difference means a step changed behaviour: roll back (§8) rather than
fixing forward. Named functions can't be checked any other way: the
generator verify scripts assert formula text and never evaluate it.

**Faster.** Read §4 step 7's sync from Cloud Run (PowerShell, `gcloud`
signed in):

```
gcloud logging read 'resource.type=cloud_run_revision AND resource.labels.service_name=sage-tools-api AND textPayload:\"timing edit\" AND textPayload:\"pickledrive\"' --freshness=1h --format='value(timestamp,textPayload)' --project=sage-tools-api
```

Compare its `fetch=` with the event day's p50 0.3 s / p90 22.6 s. One sync
is a spot check, not a distribution; a handful of edits a few seconds apart
is better. Every one is a commit to `event-data` of unchanged data.

## 7. Afterwards

- **Docs.** Add a team-workbook section to
  [The Named Function library](../../technical/named-function-library.md):
  the `…BYMATCHUPANDTEAMCODE` family, the `GETTEAM1SCOREBYMATCH`/`…2…` and
  `GETMATCHUPTITLE` functions, and `StackCache`, including that the five
  primitives read it rather than calling `STACKBLOCKS`. Record the measured
  effect beside the event-day figures in
  [Sync pipeline](../../technical/sync-pipeline.md#measured-at-two-events-3-october-2026),
  and update the
  [Team tournament](../in-progress/pickledrive-club-anniversary-team-tournament-spec.md)
  spec's status. Move this spec to `implemented/`.
- **The team template (Team tournament §15)** starts from the optimised
  workbook, and ships with the cache from the start, `SCHEDULE` trimmed to
  its real height, and no unused functions. `CSV` and `MatchLookup` should
  become one table, not two computing the same columns. The `Timeline` tabs
  belong to schedule planning, before the event; in a live workbook they
  should not recalculate on score edits.
- **The other libraries.** The standard and dual-meet masters run the same
  `STACKBLOCKS` design, and their workbooks were fast on the day (Piggleball
  p90 0.4 s), so they are not part of this spec. If one grows slow, §3.1
  applies to it unchanged.

## 8. Rollback

**File → Version history → `Before named-function cleanup` → Restore this
version.** That restores the functions, the rows and the tabs together.
Pause live sync first if the workbook is in use, then resume. To restore one
removed function by hand, its definition is in Appendix A.

## 9. Out of scope

- Any change to `sage-tools-api`, the sync scripts or the site.
- Raising `SHEETS_FETCH_TIMEOUT_MS` from `0` on Cloud Run. Bounding the read
  helps whatever the workbook does, and is a separate decision.
- The event's results: §6 exists to prove they don't change.

---

## Appendix A: the 13 unused definitions

As exported on 2026-10-05, for restoring any one by hand. Each is
`LAMBDA(<arguments>, <body>)`; in **Data → Named functions** enter the
arguments in the argument fields and the body, with a leading `=`, as the
definition.

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

## Appendix B: the definitions §4 replaces

For reading alongside §4, and for undoing a single step. `matchUp`/`teamCode`
inside a body are the `matchup`/`teamcode` arguments (Sheets names are
case-insensitive).

| Function | Current definition |
| --- | --- |
| `GETMATCHNUMBERS` | `LAMBDA(STACKBLOCKS(6,8,6))` |
| `GETPLAYERSCOLUMN1` | `LAMBDA(STACKBLOCKS(5, 8, 6))` |
| `GETPLAYERSCOLUMN2` | `LAMBDA(STACKBLOCKS(7, 8, 6))` |
| `GETSCORESCOLUMN1` | `LAMBDA(STACKBLOCKS(9, 8, 6))` |
| `GETSCORESCOLUMN2` | `LAMBDA(STACKBLOCKS(10, 8, 6))` |
| `GETMATCHESBYMATCHUPANDTEAMCODE1` | `LAMBDA(matchup, teamcode, FILTER(MatchLookup!$A$2:$A,ISNUMBER(SEARCH(matchUp,MatchLookup!$B$2:$B)), MAP(MatchLookup!$C$2:$C,LAMBDA(tc,INDEX(SPLIT(tc,"_",1),1)))=teamCode))` |
| `GETMATCHESBYMATCHUPANDTEAMCODE2` | the same with `MatchLookup!$G$2:$G` |
| `GETMATCHRESULTSBYMATCHUPANDTEAMCODE1` | `LAMBDA(matchup, teamcode, IF(ISNA(GETMATCHESBYMATCHUPANDTEAMCODE1(matchUp, teamCode)),"",MAP(GETMATCHESBYMATCHUPANDTEAMCODE1(matchUp, teamCode),LAMBDA(mn,IF(OR(ISBLANK(GETTEAM1SCOREBYMATCH(mn)),ISBLANK(GETTEAM2SCOREBYMATCH(mn))), "", GETTEAM1SCOREBYMATCH(mn) > GETTEAM2SCOREBYMATCH(mn))))))` |
| `GETMATCHRESULTSBYMATCHUPANDTEAMCODE2` | the same over `…CODE2`, comparing `GETTEAM2SCOREBYMATCH(mn) > GETTEAM1SCOREBYMATCH(mn)` |
| `GETMATCHSCORESBYMATCHUPANDTEAMCODE1` | `LAMBDA(matchup, teamcode, IF(ISNA(GETMATCHESBYMATCHUPANDTEAMCODE1(matchUp, teamCode)),"",MAP(GETMATCHESBYMATCHUPANDTEAMCODE1(matchUp, teamCode),LAMBDA(mn,GETTEAM1SCOREBYMATCH(mn)))))` |
| `GETMATCHSCORESBYMATCHUPANDTEAMCODE2` | the same over `…CODE2`, returning `GETTEAM2SCOREBYMATCH(mn)` |
| `GETOPPONENTSCORESBYMATCHUPANDTEAMCODE1` | the same over `…CODE1`, returning `GETTEAM2SCOREBYMATCH(mn)` |
| `GETOPPONENTSCORESBYMATCHUPANDTEAMCODE2` | the same over `…CODE2`, returning `GETTEAM1SCOREBYMATCH(mn)` |
